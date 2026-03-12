# Containers, Docker e LXD — Guia Conceitual

> Data: 2026-03-09
> Referência para o servidor LXD `servidor`

---

## 1. O Conceito Básico

Um container é um **computador dentro do computador**, sem o peso de uma máquina virtual completa.

```
┌─────────────────────────────────────────────┐
│              Seu Desktop (Host)              │
│                                             │
│  ┌───────────┐  ┌───────────┐  ┌─────────┐ │
│  │ Container │  │ Container │  │ Container│ │
│  │ "servidor"│  │  Redis    │  │  Node.js │ │
│  │ Nginx,PHP │  │           │  │          │ │
│  │ MySQL,Java│  │           │  │          │ │
│  └───────────┘  └───────────┘  └─────────┘ │
│                                             │
│         Linux Kernel (compartilhado)        │
└─────────────────────────────────────────────┘
```

Cada container é **isolado** — tem seus próprios arquivos, processos, rede — mas todos compartilham o **mesmo kernel** do host. Por isso são leves e rápidos.

---

## 2. Máquina Virtual vs Container

| Aspecto | Máquina Virtual (VM) | Container |
|---|---|---|
| **O que virtualiza** | Hardware inteiro + SO completo | Apenas o espaço do usuário |
| **Kernel** | Próprio (separado) | Compartilha com o host |
| **Tamanho** | GBs (SO completo) | MBs (só o necessário) |
| **Tempo de boot** | Minutos | Segundos |
| **RAM** | Reserva fixa (ex: 4GB) | Usa sob demanda |
| **Exemplo** | VirtualBox, VMware | Docker, LXD |

```
Máquina Virtual:                    Container:

┌──────────────┐                   ┌──────────────┐
│   Seu App    │                   │   Seu App    │
├──────────────┤                   ├──────────────┤
│   Libs/Bins  │                   │   Libs/Bins  │
├──────────────┤                   └──────┬───────┘
│  SO Completo │                          │
│  (kernel+tudo)│                         │ (compartilha)
├──────────────┤                          │
│  Hypervisor  │                   ┌──────┴───────┐
├──────────────┤                   │ Kernel Host  │
│  SO Host     │                   ├──────────────┤
│  Hardware    │                   │  Hardware    │
└──────────────┘                   └──────────────┘
```

---

## 3. Docker vs LXD — Duas Abordagens Diferentes

### Docker — Container de aplicação

Empacota **uma aplicação** com tudo que ela precisa para rodar.

```
┌─────────────────┐    ┌─────────────────┐    ┌─────────────────┐
│  Container 1    │    │  Container 2    │    │  Container 3    │
│  Nginx          │    │  MySQL          │    │  PHP-FPM        │
│  (só nginx)     │    │  (só mysql)     │    │  (só php)       │
└─────────────────┘    └─────────────────┘    └─────────────────┘
```

- **1 serviço por container** (filosofia Docker)
- Efêmero — destrói e recria facilmente
- Usa `Dockerfile` para definir a imagem
- Orquestra vários com `docker-compose`
- Ideal para: deploy de apps, microserviços, CI/CD

### LXD — Container de sistema

Roda um **SO completo** como se fosse uma VM, mas com a leveza de container.

```
┌──────────────────────────────────┐
│  Container "servidor"            │
│                                  │
│  Nginx + MySQL + PHP + Docker    │
│  Java + Python + SSH + tudo      │
│                                  │
│  (parece um servidor de verdade) │
└──────────────────────────────────┘
```

- **Vários serviços** no mesmo container (como um servidor real)
- Persistente — roda como uma máquina
- Tem `systemd`, serviços, usuários, cron
- Snapshots, backup, migração
- Ideal para: ambientes de desenvolvimento, servidores, labs

### Resumo comparativo

| | Docker | LXD |
|---|---|---|
| **Analogia** | Um **pacote** com uma app | Uma **máquina** virtual leve |
| **Foco** | Aplicação isolada | Sistema operacional completo |
| **Serviços** | 1 por container | Vários por container |
| **Init** | Não tem (roda 1 processo) | Tem systemd (como servidor real) |
| **Uso típico** | Deploy, microserviços | Dev server, lab, VPS local |

---

## 4. O Que Está Rodando no Nosso Caso

```
Desktop (Ubuntu)
  └── LXD
       └── Container "servidor" (Ubuntu 24.04)
            ├── Nginx (porta 80)
            ├── MySQL (porta 3306)
            ├── PHP-FPM
            ├── Java + Spring Boot (porta 8081)
            ├── Docker (sim, Docker DENTRO do LXD!)
            │    └── (pode rodar containers Docker aqui)
            ├── SSH (porta 22)
            ├── cloudflared
            └── tudo mais...
```

O container `servidor` se comporta como um **servidor real** rodando no SSD NVMe. E dentro dele, o **Docker** está disponível para rodar apps isoladas (Redis, PostgreSQL, etc.) no estilo Docker.

É como ter uma **VPS gratuita e local**, com a vantagem de snapshots instantâneos e sem pagar nada.

---

## 5. Conceitos Importantes

### Imagem
O "molde" usado para criar um container. Exemplo: `ubuntu:24.04` é a imagem base do servidor.

### Snapshot
Uma foto instantânea do estado completo do container. Permite voltar no tempo se algo der errado. Funciona como um **"Ctrl+Z" do container inteiro**.

### Volume / Storage Pool
Onde os dados do container ficam armazenados. No nosso caso, o storage pool `default` usa o SSD NVMe com filesystem btrfs.

### Port Forwarding
Regra que redireciona tráfego de uma porta do desktop para dentro do container. Exemplo: `192.168.1.9:80` → `container:80`.

### Bridge de Rede
O LXD cria uma rede virtual interna (10.86.109.x) para os containers. O port forwarding conecta essa rede à rede local (192.168.1.x).

---

## 6. Comandos Essenciais Comparados

### LXD (containers de sistema)

```bash
lxc launch ubuntu:24.04 meucontainer    # Criar container
lxc list                                  # Listar containers
lxc exec meucontainer -- bash            # Entrar no container
lxc stop meucontainer                    # Parar
lxc start meucontainer                   # Iniciar
lxc snapshot meucontainer nome           # Snapshot
lxc restore meucontainer nome            # Restaurar snapshot
lxc delete meucontainer                  # Deletar
```

### Docker (containers de aplicação)

```bash
docker run -d --name redis redis:alpine  # Criar e rodar
docker ps                                 # Listar rodando
docker exec -it redis bash               # Entrar no container
docker stop redis                        # Parar
docker start redis                       # Iniciar
docker rm redis                          # Deletar
docker compose up -d                     # Subir vários (compose)
docker compose down                      # Derrubar vários
```
