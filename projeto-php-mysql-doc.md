# Projeto PHP + MySQL — Galeria de Imagens

> Versão: 1.0.0
> Data: 2026-03-09
> Servidor: Container LXD `servidor` (192.168.1.9)

---

## 1. Descrição

Aplicação web com cadastro/login de usuários e galeria de imagens. O usuário se cadastra com e-mail e senha, faz login e acessa uma lista de imagens PNG carregadas a partir de registros no MySQL.

---

## 2. Stack

| Tecnologia | Versão | Função |
|---|---|---|
| PHP-FPM | 8.3.6 | Backend e lógica do servidor |
| MySQL | 8.0.45 | Banco de dados |
| Nginx | 1.24.0 | Servidor web |
| HTML5 + CSS3 | — | Interface (dark theme) |

---

## 3. Estrutura de Arquivos

```
/var/www/html/projeto/
├── config.php       # Conexão com MySQL e início de sessão
├── index.php        # Página de boas-vindas
├── cadastro.php     # Cadastro e login de usuários
├── painel.php       # Galeria de imagens (área protegida)
├── logout.php       # Encerrar sessão
├── style.css        # Estilos CSS (dark theme)
└── img/
    ├── php-logo.png
    ├── mysql-logo.png
    └── linux-logo.png
```

---

## 4. Banco de Dados

### Conexão

| Parâmetro | Valor |
|---|---|
| **Host** | 127.0.0.1 |
| **Banco** | teste_projeto |
| **Usuário** | projeto |
| **Senha** | projeto123 |
| **Charset** | utf8mb4 |

### Tabela `usuarios`

| Coluna | Tipo | Descrição |
|---|---|---|
| id | INT AUTO_INCREMENT | Chave primária |
| email | VARCHAR(255) UNIQUE | E-mail do usuário |
| senha | VARCHAR(255) | Hash da senha (password_hash) |
| criado_em | DATETIME | Data de criação |

### Tabela `imagens`

| Coluna | Tipo | Descrição |
|---|---|---|
| id | INT AUTO_INCREMENT | Chave primária |
| nome | VARCHAR(255) | Nome de exibição |
| arquivo | VARCHAR(255) | Nome do arquivo PNG em `img/` |
| criado_em | DATETIME | Data de criação |

### Dados iniciais (imagens)

| id | nome | arquivo |
|---|---|---|
| 1 | Logo PHP | php-logo.png |
| 2 | Logo MySQL | mysql-logo.png |
| 3 | Logo Linux | linux-logo.png |

---

## 5. Fluxo de Funcionamento

```
┌─────────────────┐
│   index.php     │  Página de boas-vindas
│  "Bem-vindo!"   │  Botão: "Cadastre-se aqui"
└────────┬────────┘
         │
         ▼
┌─────────────────┐
│  cadastro.php   │  Modo CADASTRO (padrão)
│  E-mail + Senha │──── Valida campos
│  [Cadastrar]    │──── Gera hash com password_hash()
│                 │──── INSERT na tabela usuarios
│                 │──── Se e-mail existe → erro
│                 │──── Se OK → mensagem + muda para login
└────────┬────────┘
         │ toggle: "Já tem conta? Faça login"
         ▼
┌─────────────────┐
│  cadastro.php   │  Modo LOGIN
│  E-mail + Senha │──── SELECT na tabela usuarios
│  [Entrar]       │──── Verifica com password_verify()
│                 │──── Se OK → cria sessão → redirect
│                 │──── Se erro → mensagem
└────────┬────────┘
         │ redirect (302)
         ▼
┌─────────────────┐
│   painel.php    │  Área protegida (verifica sessão)
│  Galeria de     │──── SELECT * FROM imagens
│  Imagens        │──── Exibe grid de cards com img/
│  [Sair]         │──── Clique abre imagem em nova aba
└────────┬────────┘
         │ logout
         ▼
┌─────────────────┐
│  logout.php     │  Destroi sessão
│                 │──── Redirect para index.php
└─────────────────┘
```

---

## 6. Detalhamento dos Arquivos

### config.php
- Inicia a sessão (`session_start()`)
- Cria conexão MySQLi com o banco `teste_projeto`
- Define charset utf8mb4
- Incluído por todos os outros arquivos PHP

### index.php
- Página estática de boas-vindas
- Texto: "Bem-vindo ao Teste!"
- Link para `cadastro.php`

### cadastro.php
- Dois modos: **cadastro** (padrão) e **login** (`?modo=login`)
- **Cadastro**: valida campos, gera hash com `password_hash(PASSWORD_DEFAULT)`, insere no banco
- **Login**: busca usuário por e-mail, verifica senha com `password_verify()`, cria sessão
- Trata erro de e-mail duplicado (MySQL errno 1062)
- Toggle entre cadastro e login na mesma página
- Mensagens de feedback com classes CSS `.msg.erro` e `.msg.ok`

### painel.php
- Protegido por sessão — redireciona para login se não autenticado
- Consulta tabela `imagens` e exibe em grid responsivo
- Cada card mostra: imagem PNG + nome
- Clique abre a imagem em nova aba (`target="_blank"`)
- Exibe e-mail do usuário logado + link de logout

### logout.php
- Destroi a sessão com `session_destroy()`
- Redireciona para `index.php`

### style.css
- Dark theme (fundo #1a1a2e, cards #16213e)
- Cor de destaque: #e94560 (vermelho)
- Layout responsivo com CSS Grid (`grid-template-columns: repeat(auto-fill, minmax(200px, 1fr))`)
- Efeito hover nos cards (translateY)
- Formulários estilizados com transição de foco

---

## 7. Segurança

| Aspecto | Implementação |
|---|---|
| Senhas | Hash com `password_hash()` / `password_verify()` |
| SQL Injection | Prepared statements (`$conn->prepare()` + `bind_param()`) |
| XSS | `htmlspecialchars()` em todas as saídas |
| Sessão | `session_start()` + verificação em páginas protegidas |
| E-mail duplicado | Constraint UNIQUE + tratamento de errno 1062 |

---

## 8. Acesso

| Recurso | URL |
|---|---|
| Página inicial | http://192.168.1.9/projeto/ |
| Cadastro | http://192.168.1.9/projeto/cadastro.php |
| Login | http://192.168.1.9/projeto/cadastro.php?modo=login |
| Painel | http://192.168.1.9/projeto/painel.php |

---

## 9. Comandos Úteis

```bash
# Acessar o container
lxc exec servidor -- bash

# Ver usuários cadastrados
lxc exec servidor -- mysql -u root -e "SELECT id, email, criado_em FROM teste_projeto.usuarios;"

# Ver imagens cadastradas
lxc exec servidor -- mysql -u root -e "SELECT * FROM teste_projeto.imagens;"

# Adicionar nova imagem (colocar o .png em img/ antes)
lxc exec servidor -- mysql -u root -e "INSERT INTO teste_projeto.imagens (nome, arquivo) VALUES ('Nova Imagem', 'nova.png');"

# Upload de imagem para o servidor
lxc file push imagem.png servidor/var/www/html/projeto/img/

# Reiniciar serviços
lxc exec servidor -- systemctl restart nginx php8.3-fpm mysql
```

---

## 10. Controle de Versão

### v1.0.0 — 2026-03-09 (versão atual)

**Lançamento inicial**

- Página de boas-vindas com link para cadastro
- Sistema de cadastro com e-mail e senha (hash seguro)
- Sistema de login com verificação de senha
- Painel com galeria de imagens em grid responsivo
- 3 imagens de exemplo (PHP, MySQL, Linux)
- Registros de imagens no MySQL
- Interface dark theme responsiva
- Proteção de páginas por sessão
- Prepared statements contra SQL injection
- Escape de saída contra XSS

### Melhorias futuras (planejado)

- [ ] Upload de imagens pelo painel (formulário + `move_uploaded_file`)
- [ ] Página de perfil do usuário
- [ ] Paginação na galeria
- [ ] Categorias/tags para imagens
- [ ] Confirmação de e-mail no cadastro
- [ ] Recuperação de senha
- [ ] Painel administrativo (CRUD de imagens e usuários)
- [ ] HTTPS com certificado autoassinado
