# Cloudflare Tunnel — Acesso Externo ao Servidor LXD

> Versão: 1.0.0
> Data: 2026-03-09
> Servidor: Container LXD `servidor` (192.168.1.9)

---

## 1. Visão Geral

O Cloudflare Tunnel permite expor serviços locais do container LXD para a internet, sem abrir portas no roteador, com HTTPS automático e proteção DDoS gratuita.

| Item | Detalhe |
|---|---|
| **Ferramenta** | cloudflared |
| **Versão** | 2026.2.0 |
| **Modo** | Quick Tunnel (sem conta/domínio) |
| **Instalado em** | Container LXD `servidor` |
| **URLs geradas** | `*.trycloudflare.com` (temporárias) |

---

## 2. Como Funciona

```
Usuário externo (internet)
        │
        ▼
┌──────────────────────┐
│  Cloudflare Edge     │  HTTPS automático
│  *.trycloudflare.com │  Proteção DDoS
└──────────┬───────────┘
           │ tunnel criptografado
           ▼
┌──────────────────────┐
│  cloudflared         │  Dentro do container
│  (cliente tunnel)    │  Conexão de saída (sem abrir portas)
└──────────┬───────────┘
           │
           ▼
┌──────────────────────┐
│  Serviço local       │  Nginx (:80), Spring Boot (:8081), etc.
│  localhost:porta     │
└──────────────────────┘
```

- O `cloudflared` inicia uma conexão **de saída** para a rede Cloudflare
- Não precisa abrir nenhuma porta no roteador
- O tráfego externo chega via Cloudflare e é encaminhado ao serviço local
- HTTPS é fornecido automaticamente pelo Cloudflare

---

## 3. Instalação

O `cloudflared` já está instalado no container. Caso precise reinstalar:

```bash
lxc exec servidor -- bash -c '
    curl -fsSL https://github.com/cloudflare/cloudflared/releases/latest/download/cloudflared-linux-amd64.deb -o /tmp/cloudflared.deb
    dpkg -i /tmp/cloudflared.deb
    rm /tmp/cloudflared.deb
'

# Verificar versão
lxc exec servidor -- cloudflared --version
```

---

## 4. Tunnels Configurados

### 4.1 Projeto PHP (porta 80)

```bash
# Iniciar
lxc exec servidor -- sh -c 'cloudflared tunnel --url http://localhost:80 > /tmp/tunnel-php.log 2>&1 &'

# Ver URL gerada
lxc exec servidor -- grep -o "https://[^ ]*trycloudflare.com" /tmp/tunnel-php.log | head -1

# Acessar (exemplo — URL muda a cada execução)
# https://XXXXX.trycloudflare.com/projeto/
```

| Serviço | Porta local | URL externa |
|---|---|---|
| Nginx (PHP) | 80 | `https://XXXXX.trycloudflare.com` |
| Projeto PHP | 80 | `https://XXXXX.trycloudflare.com/projeto/` |
| phpMyAdmin | 8080 | (precisa tunnel separado) |

### 4.2 Projeto Java Spring Boot (porta 8081)

```bash
# Iniciar
lxc exec servidor -- sh -c 'cloudflared tunnel --url http://localhost:8081 > /tmp/tunnel-java.log 2>&1 &'

# Ver URL gerada
lxc exec servidor -- grep -o "https://[^ ]*trycloudflare.com" /tmp/tunnel-java.log | head -1

# Acessar (exemplo — URL muda a cada execução)
# https://YYYYY.trycloudflare.com/
```

| Serviço | Porta local | URL externa |
|---|---|---|
| Spring Boot | 8081 | `https://YYYYY.trycloudflare.com` |

---

## 5. Comandos de Gerenciamento

### Iniciar todos os tunnels

```bash
# PHP (porta 80)
lxc exec servidor -- sh -c 'cloudflared tunnel --url http://localhost:80 > /tmp/tunnel-php.log 2>&1 &'

# Java (porta 8081)
lxc exec servidor -- sh -c 'cloudflared tunnel --url http://localhost:8081 > /tmp/tunnel-java.log 2>&1 &'
```

### Ver URLs ativas

```bash
# PHP
lxc exec servidor -- grep -o "https://[^ ]*trycloudflare.com" /tmp/tunnel-php.log | head -1

# Java
lxc exec servidor -- grep -o "https://[^ ]*trycloudflare.com" /tmp/tunnel-java.log | head -1
```

### Parar todos os tunnels

```bash
lxc exec servidor -- pkill cloudflared
```

### Ver processos cloudflared rodando

```bash
lxc exec servidor -- pgrep -a cloudflared
```

### Ver logs

```bash
# Tempo real
lxc exec servidor -- tail -f /tmp/tunnel-php.log
lxc exec servidor -- tail -f /tmp/tunnel-java.log

# Últimas linhas
lxc exec servidor -- tail -20 /tmp/tunnel-php.log
```

---

## 6. Expor Outros Serviços

Para expor qualquer outro serviço, basta criar um novo tunnel apontando para a porta local:

```bash
# phpMyAdmin (porta 8080)
lxc exec servidor -- sh -c 'cloudflared tunnel --url http://localhost:8080 > /tmp/tunnel-phpmyadmin.log 2>&1 &'

# SSH (porta 22) — expõe via TCP
lxc exec servidor -- sh -c 'cloudflared tunnel --url tcp://localhost:22 > /tmp/tunnel-ssh.log 2>&1 &'

# Qualquer serviço futuro
lxc exec servidor -- sh -c 'cloudflared tunnel --url http://localhost:PORTA > /tmp/tunnel-NOME.log 2>&1 &'
```

---

## 7. Limitações do Quick Tunnel (sem conta)

| Aspecto | Limitação |
|---|---|
| **URL** | Aleatória, muda a cada reinício do tunnel |
| **Persistência** | Tunnel morre se o processo for encerrado |
| **Domínio** | Não suporta domínio próprio |
| **Dashboard** | Sem painel de controle ou métricas |
| **Rate limit** | Pode ter limites de tráfego não documentados |

---

## 8. Evolução — Tunnel Permanente com Domínio (futuro)

Para ter URLs fixas com domínio próprio:

### Pré-requisitos
1. Conta no Cloudflare (grátis)
2. Domínio registrado e apontando para o Cloudflare (nameservers)

### Passos

```bash
# 1. Autenticar
cloudflared tunnel login

# 2. Criar tunnel nomeado
cloudflared tunnel create servidor

# 3. Configurar rotas DNS
cloudflared tunnel route dns servidor meusite.meudominio.com
cloudflared tunnel route dns servidor java.meudominio.com

# 4. Criar arquivo de configuração
cat > ~/.cloudflared/config.yml << EOF
tunnel: <TUNNEL_ID>
credentials-file: /root/.cloudflared/<TUNNEL_ID>.json

ingress:
  - hostname: meusite.meudominio.com
    service: http://localhost:80
  - hostname: java.meudominio.com
    service: http://localhost:8081
  - service: http_status:404
EOF

# 5. Rodar como serviço
cloudflared service install
systemctl enable cloudflared
systemctl start cloudflared
```

### Resultado

| Subdomínio | Serviço | Porta |
|---|---|---|
| `meusite.meudominio.com` | Nginx / PHP | 80 |
| `java.meudominio.com` | Spring Boot | 8081 |
| `db.meudominio.com` | phpMyAdmin | 8080 |

**Vantagens**: URL fixa, HTTPS, autostart com systemd, dashboard Cloudflare com métricas.

---

## 9. Segurança

### Recomendações ao expor serviços na internet

- **Nunca expor phpMyAdmin** sem autenticação adicional (Cloudflare Access ou IP whitelist)
- **Nunca expor SSH** via Quick Tunnel sem autenticação por chave
- **Usar senhas fortes** em todos os serviços expostos
- **Manter pacotes atualizados** (`apt update && apt upgrade`)
- **Monitorar logs** para acessos suspeitos

### Cloudflare Access (futuro, com conta)

Com conta Cloudflare, é possível adicionar autenticação (e-mail, Google, GitHub) antes de acessar os serviços — uma camada extra de proteção gratuita.

---

## 10. Referência Rápida

| Ação | Comando |
|---|---|
| Iniciar tunnel PHP | `lxc exec servidor -- sh -c 'cloudflared tunnel --url http://localhost:80 > /tmp/tunnel-php.log 2>&1 &'` |
| Iniciar tunnel Java | `lxc exec servidor -- sh -c 'cloudflared tunnel --url http://localhost:8081 > /tmp/tunnel-java.log 2>&1 &'` |
| Ver URL PHP | `lxc exec servidor -- grep -o "https://[^ ]*trycloudflare.com" /tmp/tunnel-php.log \| head -1` |
| Ver URL Java | `lxc exec servidor -- grep -o "https://[^ ]*trycloudflare.com" /tmp/tunnel-java.log \| head -1` |
| Parar tunnels | `lxc exec servidor -- pkill cloudflared` |
| Ver processos | `lxc exec servidor -- pgrep -a cloudflared` |
| Ver log PHP | `lxc exec servidor -- tail -f /tmp/tunnel-php.log` |
| Ver log Java | `lxc exec servidor -- tail -f /tmp/tunnel-java.log` |
| Versão | `lxc exec servidor -- cloudflared --version` |
