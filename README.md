# HomeServer — Ubuntu Server + Docker + Portainer (Usando GitOps)

Documentação técnica de implantação e operação de um HomeServer voltado para centralização de mídias e acesso remoto seguro, utilizando Ubuntu Server 26.04 LTS, Docker Engine, Portainer (com GitOps), Jellyfin e Tailscale.

---

## 1. Visão geral da arquitetura

O ambiente foi desenhado focando em baixa manutenção, alta reprodutibilidade e acesso remoto seguro sem abertura de portas no roteador (sem Port Forwarding / CGNAT).

### Diagrama de fluxo de rede

```text
[ REDE LOCAL: 192.168.1.0/24 ]
  │
  ├── Dispositivos Domésticos / Smart TVs / PCs
  │    │
  │    └── Acesso via mDNS: http://homeserver.local
  │         │
  │         ▼
  │   ┌─────────────────────────────────────────┐
  │   │ Ubuntu Server (hostname: homeserver)   │
  │   └────────────────────┬────────────────────┘
  │                        │
  │        ┌───────────────┴───────────────┐
  │        │        DOCKER ENGINE          │
  │        │                               │
  │        │   ┌───────────────────────┐   │
  │        │   │ Portainer (Porta 9000)│   │
  │        │   └───────────┬───────────┘   │
  │        │               │ GitOps        │
  │        │               ▼               │
  │        │   ┌───────────────────────┐   │
  │        │   │ Jellyfin  (Porta 8096)│   │
  │        │   └───────────────────────┘   │
  │        │   ┌───────────────────────┐   │
  │        │   │ Tailscale (host net)  │   │
  │        │   └───────────┬───────────┘   │
  │        └───────────────┼───────────────┘
  │                        │
  └────────────────────────┼────────────────
                           │
                           ▼
   [ TAILNET PRIVADA: Faixa 100.x.y.z ]
     │
     └── Dispositivos Android/iOS / Clientes Remotos
```

---

## 2. Especificações do servidor e rede

### 2.1 Interfaces de rede e mapeamento de IPs (Netplan)

| Interface | MAC | IP fixo | Função | Métrica |
|---|---|---|---|---:|
| `enp0s25` | `78:...:80` | `192.168.1.201/24` | Link principal | 100 |
|  `wls1`   | `ac:...:ca` | `192.168.1.202/24` | Link de backup | 600 |

### 2.2 Resolução de nome local (mDNS)

O serviço Avahi Daemon resolve o nome `homeserver.local` na rede local, dispensando configuração de registros DNS no roteador.

---

## 3. Estrutura de diretórios no servidor

```text
/
├── opt/
│   └── dockers/                     # Volume persistente dos serviços Docker
│       ├── jellyfin/
│       │   ├── config/              # Banco de dados e configurações do Jellyfin
│       │   └── cache/               # Transcodificação e cache de mídias
│       └── tailscale/
│           └── state/               # Estado do daemon e chaves do Tailscale
│
└── midia/                           # Diretório de armazenamento de mídias
    ├── filmes/
    └── series/
```

---

## 4. Arquivos de configuração do sistema

### 4.1 Configuração Netplan

Arquivo:

```text
/etc/netplan/50-cloud-init.yaml
```

Conteúdo:

```yaml
network:
  version: 2
  renderer: networkd
  ethernets:
    enp0s25:
      dhcp4: no
      addresses:
        - 192.168.1.201/24
      routes:
        - to: default
          via: 192.168.1.1
          metric: 100
      nameservers:
        addresses:
          - 192.168.1.1
          - 1.1.1.1

    wls1:
      dhcp4: no
      addresses:
        - 192.168.1.202/24
      routes:
        - to: default
          via: 192.168.1.1
          metric: 600
      nameservers:
        addresses:
          - 192.168.1.1
          - 1.1.1.1
      access-points:
        "SUA_REDE_WIFI":
          password: "SUA_SENHA_WIFI"
```

---

## 5. Estrutura do repositório Git (GitOps)

Estrutura de pastas do repositório remoto:

```text
homeserver-config/
├── jellyfin/
│   └── docker-compose.yml
└── tailscale/
    └── docker-compose.yml
... Seus outros serviços.
```

> **Nota técnica:** a inclusão do `TS_EXTRA_ARGS` é obrigatória para evitar falha de reboot (`exit status 1`) quando o estado do Tailscale já contiver flags ativas salvas.

---

## 6. Guia de implantação passo a passo

### Passo 1 — Preparação do sistema operacional

#### 1.1 Aplicar Netplan

```bash
sudo chmod 600 /etc/netplan/*.yaml
sudo netplan apply
```

#### 1.2 Configurar mDNS (Avahi)

```bash
sudo apt update && sudo apt install -y avahi-daemon
sudo hostnamectl set-hostname homeserver
sudo systemctl enable --now avahi-daemon
```

#### 1.3 Criar estrutura de pastas no servidor

```bash
sudo mkdir -p /opt/dockers/jellyfin/config /opt/dockers/jellyfin/cache /opt/dockers/tailscale/state
sudo mkdir -p /midia/filmes /midia/series
sudo chown -R $USER:$USER /opt/dockers /midia
```

### Passo 2 — Instalação da Engine Docker e Portainer

#### 2.1 Instalar Docker Engine e Plugin Compose

```bash
sudo apt update && sudo apt install -y curl ca-certificates gnupg
sudo install -m 0755 -d /etc/apt/keyrings
sudo curl -fsSL https://download.docker.com/linux/ubuntu/gpg -o /etc/apt/keyrings/docker.asc
sudo chmod a+r /etc/apt/keyrings/docker.asc

echo "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.asc] https://download.docker.com/linux/ubuntu $(. /etc/os-release && echo "$VERSION_CODENAME") stable" | sudo tee /etc/apt/sources.list.d/docker.list > /dev/null

sudo apt update
sudo apt install -y docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin
sudo usermod -aG docker $USER
```

#### 2.2 Instalar o Portainer CE

```bash
docker volume create portainer_data

docker run -d \
  -p 9000:9000 \
  -p 9443:9443 \
  --name portainer \
  --restart=always \
  -v /var/run/docker.sock:/var/run/docker.sock \
  -v portainer_data:/data \
  portainer/portainer-ce:latest
```

### Passo 3 — Deploy das Stacks via GitOps no Portainer

1. Acesse `http://homeserver.local:9000`.
2. Vá em **Stacks → Add stack → Build method: Repository**.
3. Crie a Stack `jellyfin` apontando para:
   `jellyfin/docker-compose.yml`.
4. Crie a Stack `tailscale` apontando para:
   `tailscale/docker-compose.yml`.

---

## 7. Configuração de acesso remoto no Android/iOS

1. Baixe o app Tailscale na Google Play ou App Store e conecte-se com a mesma conta vinculada ao servidor.
2. Baixe o app Jellyfin (ou Swiftfin) no dispositivo.
3. No campo de endereço do servidor dentro do app, utilize uma das opções:

   - **Pelo IP Interno:**
     `http://192.168.1.201:8096`
   - **Pelo IP do Tailscale:**
     `http://100.x.y.z:8096`

---

## 8. Comandos úteis e manutenção

### Logs do Tailscale

```bash
docker logs -f tailscale
```

### Logs do Jellyfin

```bash
docker logs -f jellyfin
```

### Verificar status dos contêineres

```bash
docker ps
```

### Atualização via GitOps

Após efetuar o commit das alterações no repositório Git, vá no Portainer, acesse a Stack correspondente e clique em **Pull and update**.

---

## 9. Resumo dos principais componentes

| Componente | Função | Acesso |
|---|---|---|
| Ubuntu Server 26.04 LTS | Sistema operacional | — |
| Docker Engine | Contêinerização | — |
| Portainer CE | Gerenciamento e GitOps | `http://homeserver.local:9000` |
| Jellyfin | Servidor de mídia | `http://homeserver.local:8096` |
| Tailscale | Acesso remoto/VPN | Tailnet `100.x.y.z` |
| Avahi | Resolução mDNS local | `homeserver.local` |

---

## 10. Considerações operacionais

- O servidor utiliza a interface cabeada `enp0s25` como rota principal.
- A interface Wi-Fi `wls1` permanece configurada como rota de menor prioridade para contingência.
- O Jellyfin mantém configuração e cache em volumes persistentes fora do ciclo de vida do contêiner.
- O estado do Tailscale também é persistido em `/opt/dockers/tailscale/state`.
- O acesso remoto é realizado através da Tailnet, evitando a necessidade de abrir portas no roteador.
- As aplicações são gerenciadas pelo Portainer e podem ser atualizadas a partir do repositório Git configurado no fluxo GitOps.
