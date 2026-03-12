# Manual — Servidor Ubuntu LXD no SSD NVMe USB

> Atualizado em: 2026-03-12 (v3 — procedimentos SSD USB, montagem automática desabilitada)

---

## 1. Visão Geral

| Item | Detalhe |
|---|---|
| **Tipo** | Container LXD |
| **Nome** | `servidor` |
| **OS** | Ubuntu Server 24.04 LTS |
| **Storage** | SSD NVMe UMIS 128GB (btrfs) via USB-C |
| **Adaptador USB** | Realtek RTL9210 M.2 NVMe |
| **Velocidade do SSD** | ~378 MB/s (USB 3.2 Gen 2) |
| **UUID do disco** | `ef542798-f3f2-4319-b512-565218534b1e` |
| **Device** | `/dev/sdc` ou `/dev/sdd` (varia conforme ordem de detecção) |
| **IP interno (LXD)** | 10.86.109.153 |
| **IP acesso rede local** | 192.168.1.9 (via desktop) |

---

## 2. Serviços Instalados

| Serviço | Versão | Porta | Status |
|---|---|---|---|
| Nginx | 1.24.0 | 80 (HTTP), 443 (HTTPS) | Ativo |
| MySQL | 8.0.45 | 3306 | Ativo |
| PHP-FPM | 8.3.6 | socket unix | Ativo |
| Docker | 28.2.2 | — | Ativo |
| Composer | 2.9.5 | — | Instalado |
| OpenSSH | 9.6p1 | 22 (interno) / 2222 (externo) | Ativo |
| Python | 3.12.3 | — | Instalado |
| Java (OpenJDK) | 21.0.10 | — | Instalado |
| Maven | 3.8.7 | — | Instalado |
| Spring Boot | 3.2.5 | 8081 | Ativo (manual) |
| phpMyAdmin | — | 8080 | Ativo |
| Cloudflare Tunnel | 2026.2.0 | — | Instalado (manual) |

---

## 3. Conectar e Desconectar o SSD NVMe USB

### 3.1 Montagem automática desabilitada

O disco btrfs do LXD foi adicionado ao `/etc/fstab` com `noauto` para impedir que o Ubuntu monte automaticamente ao plugar:

```
UUID=ef542798-f3f2-4319-b512-565218534b1e none btrfs noauto,nofail 0 0
```

### 3.2 Procedimento para DESCONECTAR o SSD

```bash
# 1. Parar o container
lxc stop servidor

# 2. Parar o LXD
sudo snap stop lxd

# 3. Agora pode desplugar o SSD com segurança
```

### 3.3 Procedimento para RECONECTAR o SSD

```bash
# 1. Plugar o SSD na USB

# 2. Iniciar o LXD
sudo snap start lxd

# 3. Verificar se o storage voltou (deve mostrar CREATED)
lxc storage list

# 4. Iniciar o container
lxc start servidor

# 5. Verificar se pegou IP
lxc list servidor
```

### 3.4 Se esqueceu de parar antes de desplugar

```bash
# Plugar o SSD e reiniciar o LXD
sudo snap restart lxd

# Iniciar o container
lxc start servidor
```

### 3.5 Se o disco foi montado automaticamente (storage UNAVAILABLE)

Se `lxc storage list` mostrar **UNAVAILABLE**, o sistema montou o disco antes do LXD:

```bash
# Verificar onde foi montado
mount | grep btrfs

# Desmontar
sudo umount /dev/sdX    # substituir X pela letra correta

# Reiniciar o LXD
sudo snap restart lxd

# Verificar (deve mostrar CREATED)
lxc storage list
```

### 3.6 Descobrir qual /dev/sdX é o SSD

O device pode mudar (`sdc`, `sdd`, etc.) dependendo da ordem de detecção:

```bash
# Verificar pelo UUID
sudo blkid | grep ef542798

# Ou verificar o tipo btrfs
sudo file -s /dev/sdc
sudo file -s /dev/sdd
```

---

## 4. Acesso ao Container

### 4.1 Via LXC (do desktop)

```bash
# Entrar como root
lxc exec servidor -- bash

# Executar comando direto
lxc exec servidor -- nginx -v
```

### 4.2 Via SSH (de qualquer máquina na rede)

```bash
# Do desktop
ssh ubuntu@192.168.1.9 -p 2222

# Do ESP32 ou outra máquina na rede 192.168.1.x
ssh ubuntu@192.168.1.9 -p 2222
```

### 4.3 Via navegador (HTTP)

```
http://192.168.1.9
```

---

## 5. Upload e Download de Arquivos

### 5.1 Via `lxc file` (do desktop, mais direto)

```bash
# Upload de arquivo
lxc file push /caminho/local/arquivo.txt servidor/var/www/html/

# Download de arquivo
lxc file pull servidor/var/www/html/arquivo.txt /caminho/local/

# Upload de pasta inteira
lxc file push -r /caminho/local/pasta/ servidor/var/www/html/
```

### 5.2 Via SCP (de qualquer máquina na rede)

```bash
# Upload
scp -P 2222 arquivo.txt ubuntu@192.168.1.9:/tmp/

# Download
scp -P 2222 ubuntu@192.168.1.9:/tmp/arquivo.txt .

# Upload de pasta
scp -P 2222 -r pasta/ ubuntu@192.168.1.9:/var/www/html/
```

### 5.3 Via SFTP (interativo)

```bash
sftp -P 2222 ubuntu@192.168.1.9
# Comandos dentro do sftp:
#   put arquivo.txt /var/www/html/
#   get /tmp/arquivo.txt
#   ls, cd, pwd, etc.
```

### 5.4 Via gerenciador de arquivos gráfico (Nautilus/Files)

No Nautilus, pressione `Ctrl+L` e digite:

```
sftp://ubuntu@192.168.1.9:2222
```

Navega como se fosse uma pasta local — arrasta e solta arquivos.

---

## 6. Port Forwarding (Rede Local → Container)

| Serviço | Desktop (192.168.1.9) | Container (interno) |
|---|---|---|
| HTTP | `192.168.1.9:80` | `127.0.0.1:80` |
| HTTPS | `192.168.1.9:443` | `127.0.0.1:443` |
| MySQL | `192.168.1.9:3306` | `127.0.0.1:3306` |
| SSH | `192.168.1.9:2222` | `127.0.0.1:22` |
| phpMyAdmin | `192.168.1.9:8080` | `127.0.0.1:8080` |
| Spring Boot | `192.168.1.9:8081` | `127.0.0.1:8081` |

### Dispositivos IoT (ESP32/ESP01)

Para o ESP32 ou ESP01 acessar o MySQL do container:

```
Host: 192.168.1.9
Porta: 3306
Usuário: (criar no MySQL)
Banco: (criar no MySQL)
```

---

## 7. Gerenciamento do Container

### 7.1 Iniciar / Parar / Reiniciar

```bash
lxc start servidor
lxc stop servidor
lxc restart servidor
```

### 7.2 Ver status

```bash
lxc list
lxc info servidor
```

### 7.3 Ver uso de recursos

```bash
# CPU e memória do container
lxc info servidor

# Espaço no SSD NVMe
lxc storage info default
```

### 7.4 Backup / Snapshot

```bash
# Criar snapshot (com nome descritivo)
lxc snapshot servidor snap-$(date +%Y%m%d)-descricao

# Listar snapshots
lxc info servidor | grep -A10 Snapshots

# Restaurar snapshot (reverte o container inteiro para o estado salvo)
lxc restore servidor snap-20260308-completo

# Deletar snapshot (quando não precisar mais)
lxc delete servidor/snap-20260308-completo

# Exportar container completo para arquivo (backup externo)
lxc export servidor /caminho/servidor-backup.tar.gz

# Importar backup
lxc import /caminho/servidor-backup.tar.gz servidor-restaurado
```

#### Snapshots existentes

| Nome | Data | Descrição |
|---|---|---|
| `snap-20260308-completo` | 2026/03/08 | Stack completa: Nginx, MySQL, PHP, Docker, Python, Java, phpMyAdmin |
| `snap-20260309-php-java-python-testados` | 2026/03/09 | Nginx com PHP-FPM ativado no site default + arquivos de teste PHP, Java, Python |
| `snap-20260309-java-cloudflare-vscode` | 2026/03/09 | Projetos PHP e Java, Maven, Spring Boot, Cloudflare Tunnel, VS Code Remote SSH |
| `snap-20260309-site-documentacao` | 2026/03/09 | + Site de documentação HTML educacional (meuservidor.html) |

### 7.5 Autostart (iniciar com o desktop)

```bash
lxc config set servidor boot.autostart true
```

---

## 8. Nginx

### Arquivos de configuração

```
/etc/nginx/nginx.conf              # Configuração principal
/etc/nginx/sites-available/        # Sites disponíveis
/etc/nginx/sites-enabled/          # Sites ativos (symlinks)
/var/www/html/                     # Document root padrão
```

### Site default — PHP habilitado

O site default (`/etc/nginx/sites-available/default`) foi configurado com suporte a PHP:

- `index.php` adicionado à diretiva `index`
- Bloco `location ~ \.php$` ativado com `fastcgi_pass unix:/run/php/php8.3-fpm.sock`

Arquivos `.php` em `/var/www/html/` são processados normalmente via `http://192.168.1.9/arquivo.php`.

### Comandos

```bash
# Testar configuração
nginx -t

# Recarregar configuração
systemctl reload nginx

# Reiniciar
systemctl restart nginx

# Ver logs
tail -f /var/log/nginx/access.log
tail -f /var/log/nginx/error.log
```

### Criar novo virtual host (exemplo Laravel)

```bash
cat > /etc/nginx/sites-available/meuapp << 'EOF'
server {
    listen 80;
    server_name meuapp.local;
    root /var/www/meuapp/public;
    index index.php;

    location / {
        try_files $uri $uri/ /index.php?$query_string;
    }

    location ~ \.php$ {
        fastcgi_pass unix:/run/php/php8.3-fpm.sock;
        fastcgi_param SCRIPT_FILENAME $realpath_root$fastcgi_script_name;
        include fastcgi_params;
    }
}
EOF

ln -s /etc/nginx/sites-available/meuapp /etc/nginx/sites-enabled/
nginx -t && systemctl reload nginx
```

---

## 9. MySQL

### Acesso

```bash
# Acesso local (dentro do container)
mysql -u root

# Acesso remoto (do desktop ou rede)
mysql -h 192.168.1.9 -P 3306 -u usuario -p
```

### Criar banco e usuário para projeto

```sql
-- Dentro do mysql
CREATE DATABASE lanchonete CHARACTER SET utf8mb4 COLLATE utf8mb4_unicode_ci;
CREATE USER 'laravel'@'%' IDENTIFIED BY 'senha_segura';
GRANT ALL PRIVILEGES ON lanchonete.* TO 'laravel'@'%';
FLUSH PRIVILEGES;
```

### Acesso remoto (já configurado)

Arquivo: `/etc/mysql/mysql.conf.d/mysqld.cnf`

```ini
bind-address = 0.0.0.0
```

Usuário remoto criado:

```
Usuário: admin
Host: % (qualquer IP)
Privilégios: ALL (com GRANT OPTION)
```

Acesso do desktop ou rede local:

```bash
mysql -h 192.168.1.9 -P 3306 -u admin -p
```

### Comandos úteis

```bash
# Ver bancos
mysql -u root -e "SHOW DATABASES;"

# Backup
mysqldump -u root lanchonete > /tmp/backup_lanchonete.sql

# Restaurar
mysql -u root lanchonete < /tmp/backup_lanchonete.sql

# Ver logs
tail -f /var/log/mysql/error.log
```

---

## 10. PHP-FPM

### Arquivos de configuração

```
/etc/php/8.3/fpm/php.ini           # Config principal do PHP
/etc/php/8.3/fpm/pool.d/www.conf   # Config do pool FPM
```

### Comandos

```bash
# Reiniciar
systemctl restart php8.3-fpm

# Ver módulos instalados
php -m

# Ver configuração
php -i | grep -i memory_limit
```

### Extensões instaladas

- php8.3-mysql
- php8.3-mbstring
- php8.3-xml
- php8.3-curl
- php8.3-zip

---

## 11. Docker (dentro do container)

```bash
# Ver containers rodando
docker ps

# Docker Compose
docker compose up -d
docker compose down

# Exemplo: subir Redis
docker run -d --name redis -p 6379:6379 redis:alpine
```

---

## 12. Composer / Laravel

### Criar novo projeto Laravel

```bash
cd /var/www
composer create-project laravel/laravel meuapp
chown -R www-data:www-data meuapp/storage meuapp/bootstrap/cache
```

### Configurar .env do Laravel

```env
DB_CONNECTION=mysql
DB_HOST=127.0.0.1
DB_PORT=3306
DB_DATABASE=lanchonete
DB_USERNAME=laravel
DB_PASSWORD=senha_segura
```

### Comandos Laravel

```bash
cd /var/www/meuapp
php artisan migrate
php artisan serve --host=0.0.0.0
php artisan make:controller NomeController
php artisan make:model Nome -mcr
```

---

## 13. Acesso pelo ESP32/ESP01

### Exemplo conexão MySQL (ESP32 com biblioteca MySQL_Connector)

```cpp
#include <MySQL_Connection.h>
#include <MySQL_Cursor.h>

IPAddress server_addr(192, 168, 1, 9);
int server_port = 3306;
char user[] = "esp_user";
char password[] = "esp_senha";

MySQL_Connection conn((Client *)&client);
conn.connect(server_addr, server_port, user, password);
```

### Exemplo HTTP POST (ESP8266/ESP32)

```cpp
#include <ESP8266HTTPClient.h>

HTTPClient http;
http.begin("http://192.168.1.9/api/dados");
http.addHeader("Content-Type", "application/json");
int httpCode = http.POST("{\"temperatura\": 25.5}");
http.end();
```

---

## 14. Estrutura de Diretórios Importantes

```
/var/www/                    # Projetos web
/var/www/html/               # Document root padrão do Nginx
/var/www/html/projeto/       # Projeto PHP (Galeria)
/var/www/galeria-java/       # Projeto Java Spring Boot (Galeria)
/etc/nginx/sites-available/  # Virtual hosts Nginx
/etc/mysql/                  # Configuração MySQL
/etc/php/8.3/                # Configuração PHP
/var/log/nginx/              # Logs Nginx
/var/log/mysql/              # Logs MySQL
/tmp/galeria.log             # Log do Spring Boot
/tmp/tunnel-php.log          # Log do tunnel Cloudflare (PHP)
/tmp/tunnel-java.log         # Log do tunnel Cloudflare (Java)
```

---

## 15. Solução de Problemas

### Storage pool UNAVAILABLE

O disco USB pode mudar de letra (`/dev/sdc`, `/dev/sdd`) entre conexões. Se `lxc storage list` mostrar **UNAVAILABLE**:

1. Verificar se o disco está conectado: `lsblk`
2. Se montado pelo sistema: `sudo umount /dev/sdX`
3. Reiniciar LXD: `sudo snap restart lxd`
4. Verificar: `lxc storage list` (deve mostrar **CREATED**)

### Container não inicia

```bash
# Verificar se o SSD USB está conectado
lsblk

# Verificar storage pool
lxc storage list

# Ver logs do LXD
lxc info servidor --show-log
```

### Container sem IPv4

Se `lxc list servidor` não mostra IPv4:

```bash
# Restaurar de um snapshot funcional
lxc stop servidor
lxc restore servidor snap-20260309-site-documentacao
lxc start servidor
```

### Serviço não responde pela rede

```bash
# Verificar port forwarding
lxc config device show servidor

# Verificar se serviço está escutando
lxc exec servidor -- ss -tlnp
```

### SSH recusa senha (Permission denied publickey)

O arquivo `/etc/ssh/sshd_config.d/60-cloudimg-settings.conf` pode sobrescrever a config principal.

```bash
# Dentro do container
echo "PasswordAuthentication yes" > /etc/ssh/sshd_config.d/60-cloudimg-settings.conf
systemctl restart ssh
```

### MySQL não aceita conexão remota

```bash
# Dentro do container, verificar bind-address
lxc exec servidor -- grep bind-address /etc/mysql/mysql.conf.d/mysqld.cnf

# Deve ser: bind-address = 0.0.0.0
```

### Verificar uso de espaço no SSD

```bash
lxc storage info default
lxc exec servidor -- df -h
```

### Reiniciar todos os serviços

```bash
lxc exec servidor -- systemctl restart nginx mysql php8.3-fpm docker
```

---

## 16. Referência Rápida de Comandos

| Ação | Comando |
|---|---|
| Entrar no container | `lxc exec servidor -- bash` |
| SSH para o container | `ssh servidor` |
| Iniciar container | `lxc start servidor` |
| Parar container | `lxc stop servidor` |
| Status do container | `lxc list` |
| Criar snapshot | `lxc snapshot servidor nome` |
| Ver logs Nginx | `lxc exec servidor -- tail -f /var/log/nginx/error.log` |
| Acessar MySQL | `mysql -h 192.168.1.9 -P 3306 -u admin -p` |
| Testar HTTP (PHP) | `curl http://192.168.1.9/projeto/` |
| Testar HTTP (Java) | `curl http://192.168.1.9:8081/` |
| phpMyAdmin | `http://192.168.1.9:8080` |
| Upload arquivo (lxc) | `lxc file push arquivo servidor/caminho/` |
| Download arquivo (lxc) | `lxc file pull servidor/caminho/arquivo .` |
| Upload arquivo (scp) | `scp -P 2222 arquivo ubuntu@192.168.1.9:/tmp/` |
| SFTP gráfico (Nautilus) | `sftp://ubuntu@192.168.1.9:2222` |
| Desconectar SSD | `lxc stop servidor && sudo snap stop lxd` |
| Reconectar SSD | `sudo snap start lxd && lxc start servidor` |
| Storage pool status | `lxc storage list` |
| VS Code (PHP) | `code --remote ssh-remote+servidor /var/www/html/projeto/` |
| VS Code (Java) | `code --remote ssh-remote+servidor /var/www/galeria-java/` |
| Iniciar Spring Boot | `lxc exec servidor -- sh -c 'cd /var/www/galeria-java && java -jar target/galeria-java-1.0.0.jar &'` |
| Parar Spring Boot | `lxc exec servidor -- sh -c 'kill $(lsof -t -i:8081)'` |
| Iniciar tunnel PHP | `lxc exec servidor -- sh -c 'cloudflared tunnel --url http://localhost:80 > /tmp/tunnel-php.log 2>&1 &'` |
| Iniciar tunnel Java | `lxc exec servidor -- sh -c 'cloudflared tunnel --url http://localhost:8081 > /tmp/tunnel-java.log 2>&1 &'` |
| Ver URL tunnel PHP | `lxc exec servidor -- grep -o "https://[^ ]*trycloudflare.com" /tmp/tunnel-php.log \| head -1` |
| Ver URL tunnel Java | `lxc exec servidor -- grep -o "https://[^ ]*trycloudflare.com" /tmp/tunnel-java.log \| head -1` |
| Parar tunnels | `lxc exec servidor -- pkill cloudflared` |

---

## 17. Python

```bash
# Verificar versão
python3 --version

# Criar ambiente virtual
python3 -m venv meuambiente
source meuambiente/bin/activate

# Instalar pacotes
pip install flask django requests
```

---

## 18. Java (OpenJDK 21) + Maven + Spring Boot

```bash
# Verificar versão
java --version
javac --version
mvn --version
```

### Projeto Galeria Java (Spring Boot)

| Item | Detalhe |
|---|---|
| **Local** | `/var/www/galeria-java/` |
| **Framework** | Spring Boot 3.2.5 + Thymeleaf + JPA |
| **Porta** | 8081 |
| **Banco** | `galeria_java` (usuário: `galeria` / `galeria123`) |
| **URL local** | http://192.168.1.9:8081/ |

```bash
# Compilar
lxc exec servidor -- sh -c 'cd /var/www/galeria-java && mvn package -DskipTests'

# Iniciar
lxc exec servidor -- sh -c 'cd /var/www/galeria-java && java -jar target/galeria-java-1.0.0.jar > /tmp/galeria.log 2>&1 &'

# Parar
lxc exec servidor -- sh -c 'kill $(lsof -t -i:8081)'

# Ver log
lxc exec servidor -- tail -f /tmp/galeria.log

# Abrir no VS Code
code --remote ssh-remote+servidor /var/www/galeria-java/
```

Documentação completa: ver `~/projeto-java-mysql-doc.md`

---

## 19. phpMyAdmin

- **URL**: http://192.168.1.9:8080
- **Usuário**: `admin`
- **Senha**: (a mesma do MySQL)
- **Config Nginx**: `/etc/nginx/sites-available/phpmyadmin`
- **Arquivos**: `/usr/share/phpmyadmin/`

---

## 20. Cloudflare Tunnel (Acesso Externo)

Permite expor serviços locais para a internet sem abrir portas no roteador, com HTTPS automático.

| Item | Detalhe |
|---|---|
| **Ferramenta** | cloudflared 2026.2.0 |
| **Modo atual** | Quick Tunnel (sem conta/domínio) |
| **URLs** | `*.trycloudflare.com` (temporárias, mudam a cada execução) |

### Iniciar tunnels

```bash
# Projeto PHP (porta 80)
lxc exec servidor -- sh -c 'cloudflared tunnel --url http://localhost:80 > /tmp/tunnel-php.log 2>&1 &'

# Projeto Java (porta 8081)
lxc exec servidor -- sh -c 'cloudflared tunnel --url http://localhost:8081 > /tmp/tunnel-java.log 2>&1 &'
```

### Ver URLs geradas

```bash
# PHP
lxc exec servidor -- grep -o "https://[^ ]*trycloudflare.com" /tmp/tunnel-php.log | head -1

# Java
lxc exec servidor -- grep -o "https://[^ ]*trycloudflare.com" /tmp/tunnel-java.log | head -1
```

### Parar tunnels

```bash
lxc exec servidor -- pkill cloudflared
```

### Expor outros serviços

```bash
# phpMyAdmin (porta 8080)
lxc exec servidor -- sh -c 'cloudflared tunnel --url http://localhost:8080 > /tmp/tunnel-phpmyadmin.log 2>&1 &'
```

Documentação completa: ver `~/servidor-cloudflare-tunnel-doc.md`

---

## 21. Projetos Hospedados

| Projeto | Stack | Local | Porta | URL local | Banco |
|---|---|---|---|---|---|
| Galeria PHP | PHP 8.3 + MySQLi | `/var/www/html/projeto/` | 80 | http://192.168.1.9/projeto/ | `teste_projeto` |
| Galeria Java | Spring Boot 3.2.5 + JPA | `/var/www/galeria-java/` | 8081 | http://192.168.1.9:8081/ | `galeria_java` |

### Bancos de dados e usuários MySQL

| Banco | Usuário | Senha | Usado por |
|---|---|---|---|
| `teste_projeto` | `projeto` | `projeto123` | Projeto PHP |
| `galeria_java` | `galeria` | `galeria123` | Projeto Java |

---

## 22. VS Code Remote SSH

Configurado para editar arquivos diretamente no container via VS Code.

| Item | Detalhe |
|---|---|
| **Extensão** | Remote - SSH v0.122.0 |
| **Host SSH** | `servidor` (alias em `~/.ssh/config`) |
| **Chave** | `~/.ssh/id_ed25519` (login sem senha) |

```bash
# Conectar via terminal
ssh servidor

# Abrir VS Code no projeto PHP
code --remote ssh-remote+servidor /var/www/html/projeto/

# Abrir VS Code no projeto Java
code --remote ssh-remote+servidor /var/www/galeria-java/
```

Extensões instaladas no VS Code: Remote - SSH, Extension Pack for Java.

---

## 23. Futuras Instalações (planejado)

- [ ] Adminer (alternativa leve ao phpMyAdmin)
- [ ] Cloudflare Tunnel permanente com domínio próprio
- [ ] Serviço systemd para autostart do Spring Boot

---

## 24. Documentação Relacionada

| Arquivo | Conteúdo |
|---|---|
| `~/servidor-lxd-manual.md` | Este manual (visão geral do servidor) |
| `~/projeto-php-mysql-doc.md` | Documentação completa do projeto PHP |
| `~/projeto-java-mysql-doc.md` | Documentação completa do projeto Java |
| `~/servidor-cloudflare-tunnel-doc.md` | Documentação do Cloudflare Tunnel |
| `~/servidor-containers-explicacao.md` | Guia conceitual: Containers, Docker e LXD |

---

> **IP Desktop na rede local:** 192.168.1.9
> **SSD NVMe:** UMIS RPJTJ128RDE1MWY (119.2 GB) — UUID: ef542798-f3f2-4319-b512-565218534b1e
> **Storage Pool LXD:** default (btrfs) — device varia: /dev/sdc ou /dev/sdd
> **fstab:** montagem automática desabilitada via `noauto,nofail`
