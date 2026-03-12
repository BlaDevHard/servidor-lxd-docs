# Projeto Java Spring Boot + MySQL — Galeria de Imagens

> Versão: 1.0.0
> Data: 2026-03-09
> Servidor: Container LXD `servidor` (192.168.1.9:8081)

---

## 1. Descrição

Aplicação web com cadastro/login de usuários e galeria de imagens, construída com Spring Boot + Thymeleaf + JPA/Hibernate + MySQL. Projeto similar ao equivalente em PHP, porém com stack Java completa e tema visual verde.

---

## 2. Stack

| Tecnologia | Versão | Função |
|---|---|---|
| Spring Boot | 3.2.5 | Framework web + servidor embutido (Tomcat) |
| Thymeleaf | 3.1.2 | Engine de templates HTML |
| Spring Data JPA / Hibernate | — | ORM e acesso ao banco |
| MySQL | 8.0.45 | Banco de dados |
| Maven | 3.8.7 | Build e gerenciamento de dependências |
| Java (OpenJDK) | 21 | Runtime |

---

## 3. Estrutura de Arquivos

```
/var/www/galeria-java/
├── pom.xml                                          # Dependências Maven
├── target/
│   └── galeria-java-1.0.0.jar                       # JAR executável
└── src/main/
    ├── java/com/galeria/
    │   ├── GaleriaApplication.java                  # Classe principal (@SpringBootApplication)
    │   ├── DataLoader.java                          # Carga inicial de imagens (CommandLineRunner)
    │   ├── controller/
    │   │   └── GaleriaController.java               # Rotas e lógica de negócio
    │   ├── model/
    │   │   ├── Usuario.java                         # Entidade JPA (tabela usuarios)
    │   │   └── Imagem.java                          # Entidade JPA (tabela imagens)
    │   └── repository/
    │       ├── UsuarioRepository.java               # Repository JPA (usuarios)
    │       └── ImagemRepository.java                # Repository JPA (imagens)
    └── resources/
        ├── application.properties                   # Configuração (porta, banco, JPA)
        ├── templates/
        │   ├── index.html                           # Página de boas-vindas
        │   ├── cadastro.html                        # Formulário de cadastro
        │   ├── login.html                           # Formulário de login
        │   └── painel.html                          # Galeria de imagens
        └── static/
            ├── css/
            │   └── style.css                        # Estilos (dark theme verde)
            └── img/
                ├── java-logo.png
                ├── spring-logo.png
                └── mysql-logo.png
```

---

## 4. Banco de Dados

### Conexão

| Parâmetro | Valor |
|---|---|
| **Host** | 127.0.0.1 |
| **Porta** | 3306 |
| **Banco** | galeria_java |
| **Usuário** | galeria |
| **Senha** | galeria123 |
| **DDL** | hibernate ddl-auto=update (cria/atualiza tabelas automaticamente) |

### Tabela `usuarios`

| Coluna | Tipo | Descrição |
|---|---|---|
| id | BIGINT AUTO_INCREMENT | Chave primária |
| email | VARCHAR(255) UNIQUE | E-mail do usuário |
| senha | VARCHAR(255) | Hash SHA-256 com salt (Base64) |
| criado_em | DATETIME | Data de criação |

### Tabela `imagens`

| Coluna | Tipo | Descrição |
|---|---|---|
| id | BIGINT AUTO_INCREMENT | Chave primária |
| nome | VARCHAR(255) | Nome de exibição |
| arquivo | VARCHAR(255) | Nome do arquivo PNG em `static/img/` |
| criado_em | DATETIME | Data de criação |

### Dados iniciais (inseridos pelo DataLoader)

| id | nome | arquivo |
|---|---|---|
| 1 | Logo Java | java-logo.png |
| 2 | Logo Spring | spring-logo.png |
| 3 | Logo MySQL | mysql-logo.png |

---

## 5. Fluxo de Funcionamento

```
┌─────────────────────┐
│   GET /              │  Página de boas-vindas
│   index.html         │  Botão: "Cadastre-se aqui"
└──────────┬──────────┘
           │
           ▼
┌─────────────────────┐
│   GET /cadastro      │  Formulário de cadastro
│   cadastro.html      │  E-mail + Senha
│                      │
│   POST /cadastro     │──── Valida campos (mín. 6 chars)
│                      │──── Verifica e-mail duplicado (existsByEmail)
│                      │──── Hash SHA-256 + salt aleatório
│                      │──── Salva via UsuarioRepository
│                      │──── Redirect → /login com flash msg
└──────────┬──────────┘
           │ link: "Já tem conta? Faça login"
           ▼
┌─────────────────────┐
│   GET /login         │  Formulário de login
│   login.html         │  E-mail + Senha
│                      │
│   POST /login        │──── Busca por e-mail (findByEmail)
│                      │──── Verifica senha (SHA-256 + salt)
│                      │──── Cria sessão HTTP (usuarioId + email)
│                      │──── Redirect → /painel
└──────────┬──────────┘
           │ redirect (302)
           ▼
┌─────────────────────┐
│   GET /painel        │  Área protegida (verifica sessão)
│   painel.html        │──── findAllByOrderByNomeAsc()
│                      │──── Exibe grid de cards com imagens
│   Galeria de Imagens │──── Clique abre imagem em nova aba
│   [Sair]             │
└──────────┬──────────┘
           │ GET /logout
           ▼
┌─────────────────────┐
│   logout             │  session.invalidate()
│                      │──── Redirect → /
└─────────────────────┘
```

---

## 6. Detalhamento dos Componentes

### GaleriaApplication.java
- Classe principal com `@SpringBootApplication`
- Ponto de entrada da aplicação (`main`)

### DataLoader.java
- Implementa `CommandLineRunner` — executa ao iniciar a aplicação
- Verifica se a tabela `imagens` está vazia (`imagemRepo.count() == 0`)
- Insere 3 imagens de exemplo na primeira execução

### GaleriaController.java
- Controller único com todas as rotas:
  - `GET /` → página inicial
  - `GET /cadastro` + `POST /cadastro` → cadastro de usuário
  - `GET /login` + `POST /login` → autenticação
  - `GET /painel` → galeria (protegida)
  - `GET /logout` → encerrar sessão
- Usa `RedirectAttributes` para flash messages entre redirects
- Usa `HttpSession` para controle de sessão

### Usuario.java / Imagem.java
- Entidades JPA mapeadas com `@Entity` e `@Table`
- Hibernate cria/atualiza as tabelas automaticamente (`ddl-auto=update`)

### UsuarioRepository.java / ImagemRepository.java
- Interfaces que estendem `JpaRepository`
- Métodos customizados:
  - `findByEmail(String email)` → busca por e-mail
  - `existsByEmail(String email)` → verifica duplicidade
  - `findAllByOrderByNomeAsc()` → lista imagens ordenadas

### Templates Thymeleaf
- Usam namespace `th:` para expressões dinâmicas
- `th:if`, `th:text`, `th:each` para lógica no HTML
- `th:classappend` para classes CSS condicionais
- `@{}` para URLs relativas ao context path

### style.css
- Dark theme (fundo #1a1a2e, cards #16213e)
- Cor de destaque: #0f9b58 (verde — diferencia do projeto PHP que usa vermelho)
- Layout responsivo com CSS Grid
- Efeito hover nos cards

---

## 7. Segurança

| Aspecto | Implementação |
|---|---|
| Senhas | SHA-256 + salt aleatório de 16 bytes (Base64) |
| SQL Injection | Spring Data JPA (prepared statements automáticos) |
| XSS | Thymeleaf escapa HTML automaticamente (`th:text`) |
| Sessão | `HttpSession` do Servlet + verificação em rotas protegidas |
| E-mail duplicado | Constraint UNIQUE + verificação prévia (`existsByEmail`) |
| Validação | Campos required no HTML + validação no controller |

---

## 8. Rotas

| Método | Rota | Descrição | Autenticação |
|---|---|---|---|
| GET | `/` | Página inicial | Não |
| GET | `/cadastro` | Formulário de cadastro | Não |
| POST | `/cadastro` | Processar cadastro | Não |
| GET | `/login` | Formulário de login | Não |
| POST | `/login` | Processar login | Não |
| GET | `/painel` | Galeria de imagens | Sim |
| GET | `/logout` | Encerrar sessão | Sim |

---

## 9. Configuração (application.properties)

```properties
server.port=8081
spring.datasource.url=jdbc:mysql://127.0.0.1:3306/galeria_java?useSSL=false&allowPublicKeyRetrieval=true
spring.datasource.username=galeria
spring.datasource.password=galeria123
spring.jpa.hibernate.ddl-auto=update
spring.jpa.show-sql=false
spring.thymeleaf.cache=false
```

---

## 10. Acesso

| Recurso | URL |
|---|---|
| Página inicial | http://192.168.1.9:8081/ |
| Cadastro | http://192.168.1.9:8081/cadastro |
| Login | http://192.168.1.9:8081/login |
| Painel | http://192.168.1.9:8081/painel |

---

## 11. Comandos Úteis

```bash
# Compilar o projeto
lxc exec servidor -- sh -c 'cd /var/www/galeria-java && mvn package -DskipTests'

# Iniciar a aplicação
lxc exec servidor -- sh -c 'cd /var/www/galeria-java && java -jar target/galeria-java-1.0.0.jar &'

# Parar a aplicação
lxc exec servidor -- sh -c 'kill $(lsof -t -i:8081)'

# Reiniciar a aplicação
lxc exec servidor -- sh -c 'kill $(lsof -t -i:8081) 2>/dev/null; sleep 2; cd /var/www/galeria-java && java -jar target/galeria-java-1.0.0.jar &'

# Ver log da aplicação
lxc exec servidor -- tail -f /tmp/galeria.log

# Ver usuários cadastrados
lxc exec servidor -- mysql -u root -e "SELECT id, email, criado_em FROM galeria_java.usuarios;"

# Ver imagens cadastradas
lxc exec servidor -- mysql -u root -e "SELECT * FROM galeria_java.imagens;"

# Adicionar nova imagem (colocar o .png em static/img/ e recompilar, ou inserir direto no banco)
lxc exec servidor -- mysql -u root -e "INSERT INTO galeria_java.imagens (nome, arquivo) VALUES ('Nova Imagem', 'nova.png');"

# Abrir projeto no VS Code
code --remote ssh-remote+servidor /var/www/galeria-java/
```

---

## 12. Port Forwarding

| Serviço | Desktop (rede local) | Container (interno) |
|---|---|---|
| Spring Boot | 192.168.1.9:8081 | 127.0.0.1:8081 |

Configurado via:
```bash
lxc config device add servidor proxy-8081 proxy listen=tcp:0.0.0.0:8081 connect=tcp:127.0.0.1:8081
```

---

## 13. Comparação com Projeto PHP

| Aspecto | Projeto PHP | Projeto Java |
|---|---|---|
| **Framework** | PHP puro + MySQLi | Spring Boot + JPA |
| **Templates** | PHP inline | Thymeleaf |
| **Servidor** | Nginx + PHP-FPM | Tomcat embutido |
| **Porta** | 80 (via Nginx) | 8081 |
| **Hash de senha** | password_hash (bcrypt) | SHA-256 + salt |
| **ORM** | Consultas manuais | Hibernate/JPA |
| **Tema visual** | Vermelho (#e94560) | Verde (#0f9b58) |
| **Banco** | teste_projeto | galeria_java |
| **URL** | /projeto/ | / (raiz na 8081) |

---

## 14. Controle de Versão

### v1.0.0 — 2026-03-09 (versão atual)

**Lançamento inicial**

- Projeto Spring Boot 3.2.5 com Thymeleaf e JPA
- Página de boas-vindas com link para cadastro
- Sistema de cadastro com e-mail e senha (hash SHA-256 + salt)
- Sistema de login com verificação de senha e sessão HTTP
- Painel com galeria de imagens em grid responsivo
- 3 imagens de exemplo (Java, Spring, MySQL) via DataLoader
- Registros de imagens no MySQL com JPA/Hibernate
- Interface dark theme verde responsiva
- Proteção de rotas por sessão
- Port forwarding na porta 8081

### Melhorias futuras (planejado)

- [ ] Migrar hash de senha para BCrypt (Spring Security)
- [ ] Adicionar Spring Security para autenticação completa
- [ ] Upload de imagens pelo painel
- [ ] Página de perfil do usuário
- [ ] Paginação na galeria
- [ ] Categorias/tags para imagens
- [ ] Validação com Bean Validation (@Valid)
- [ ] Testes unitários e de integração
- [ ] Dockerfile para deploy containerizado
- [ ] Serviço systemd para autostart da aplicação
