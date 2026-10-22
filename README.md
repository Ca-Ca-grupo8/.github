# Projeto de Extensão: Gestão de Inventário para Semijoias
Bem-vindo à organização dedicada ao desenvolvimento do sistema de gestão para vendedoras autônomas de semijoias. Este projeto faz parte das atividades de extensão acadêmica do 3º semestre.

# Contexto
Vendedoras de semijoias frequentemente lidam com um inventário composto por itens pequenos, variados e de alto valor agregado. A falta de um controle sistemático resulta em perda de mercadorias, dificuldade em calcular o lucro real, estoque parado e dificuldade de manejar os itens enviadas às clientes.

Este repositório foca em criar uma solução técnica que permita:
  Categorização por tipo de banho (ouro e prata), peça, etc.
  Controle de entradas e saídas de peças.
  Geração de alertas para reposição de estoque.
  Controle de clientes.

# Alinhamento com as ODS (Agenda 2030)
O projeto está fundamentado nos Objetivos de Desenvolvimento Sustentável da ONU:
  ODS 5 (Igualdade de Genêro): Foco na meta 5.b, visando a gestão eficiente e o uso estratégico de recursos tecnológicos para ter controle sobre o próprio dinheiro e estoque.
  ODS 8 (Trabalho Decente e Crescimento Econômico): Foco na meta 8.3, apoiando o empreendedorismo e a profissionalização de microempresas através da inovação tecnológica.

# Regras de Contribuição
Para manter a organização do código e o histórico do projeto limpo, todos os membros devem seguir as regras abaixo:
## Branches
  main: Versão estável (entrega).
  feature/nome ou nomeContribuidor: Para desenvolver tarefas específicas.
## Commits (Padrão)
Use sempre um prefixo antes da mensagem:
  feat: Nova funcionalidade.
  fix: Correção de erros.
  docs: Alteração em documentos/README.
  refactor: Melhoria no código existente.
Emplo: feat: adiciona botão de login


## 2. Topologia dos repositórios

A pasta raiz **não é um repositório git**. São **4 repositórios independentes**:

| Pasta | Repo remoto | Branch atual | Stack |
| --- | --- | --- | --- |
| `BackEnd/` | `Ca-Ca-grupo8/BackEnd.git` | `feat/integracao-brevo` | Spring Boot 4.0.5 · Java 21 · MySQL |
| `FrontEnd/caeca/` | `Ca-Ca-grupo8/FrontEnd.git` | `feat/movimentacoesMaletas` | React 19 · Vite 8 · React Router 7 |
| `Infra/` | `Ca-Ca-grupo8/Infra.git` | `main` | Terraform + CloudFormation (AWS) |
| `analiseDados/` | `Ca-Ca-grupo8/analiseDados.git` | `main` | Python / data lake (**vazio**, só `bronze/`) |

> Branches `main` defasadas em BackEnd e FrontEnd: Features relevantes só existem em
> branches de trabalho. **Não assuma que `main` está atualizado.**

### Documentos complementares

| Arquivo | Escopo |
| --- | --- |
| `BackEnd/AGENTS.md` | Detalhe do backend: convenções, testes, armadilhas de Spring Boot 4 |
| `Infra/README.md` | Detalhe da infra: buckets via CloudFormation, resto via Terraform |
| `FrontEnd/caeca/README.md` | Boilerplate do Vite (não descreve o app) |
| `C4_MODEL.md` | Modelo de arquitetura C4 (contexto, contêineres, componentes, implantação e fluxos dinâmicos) |

### Modelo C4

A arquitetura está documentada no padrão [C4](https://c4model.com/) em
[`C4_MODEL.md`](C4_MODEL.md), com diagramas em Mermaid:

| Nível | Diagrama |
| --- | --- |
| 1 — Contexto | Usuários e sistemas externos (Brevo, ViaCEP, AWS) |
| 2 — Contêineres | Frontend, API, banco e serviços de apoio |
| 3 — Componentes | Backend (`BackEnd/estoque`) e frontend (`FrontEnd/caeca`) |
| 4 — Implantação | Infraestrutura AWS (`Infra/`) |
| Dinâmicos | Autenticação, movimentação, notificação assíncrona e rotinas agendadas |

---

## Nível 1 — Diagrama de contexto

Quem usa o sistema, com quem ele interage e o que existe fora dos nossos limites.

```mermaid
flowchart TD
    admin["Administrador<br/><i>ROLE_ADMIN</i>"]
    func["Funcionária / operadora<br/><i>ROLE_USER</i>"]
    rev["Revendedora<br/><i>não acessa o sistema — só recebe e-mails</i>"]

    caeca["Sistema Ca&Cá<br/>Gestão de estoque de semijoias"]

    brevo["Brevo<br/>CRM / e-mails transacionais"]
    viacep["ViaCEP<br/>consulta de CEP"]
    aws["AWS Cloud<br/>S3 · EventBridge · Lambda"]

    admin -->|"gerencia cadastros, convites e<br/>configurações (categorias, banhos, materiais, fornecedores)"| caeca
    func -->|"registra movimentações, opera inventário,<br/>maletas e consulta dashboards"| caeca
    rev -->|"lê e-mails de consignação, alertas<br/>e convites de primeiro acesso"| brevo
    caeca -->|"publica eventos de notificação<br/>(e-mail transacional)"| brevo
    caeca -->|"consulta CEP (SPA chama direto)"| viacep
    caeca -->|"armazena fotos de joias, data lake<br/>e rotinas agendadas"| aws

    classDef pessoa fill:#08427B,stroke:#052E56,color:#fff
    classDef sistema fill:#1168BD,stroke:#0B4884,color:#fff
    classDef externo fill:#999999,stroke:#6B6B6B,color:#fff
    class admin,func,rev pessoa
    class caeca sistema
    class brevo,viacep,aws externo
```

**Atos do sistema:** autenticar usuários (JWT), cadastrar manutenção de dados
(joias, categorias, banhos, materiais, motivos), registrar movimentações de estoque,
gerenciar maletas/vendas de revendedoras, exibir dashboards e disparar notificações
assíncronas por e-mail.

---

## Nível 2 — Diagrama de contêineres

Contêineres = unidades de código executando em seu próprio processo (SPA, API,
banco, fila, buckets, functions).

```mermaid
flowchart TD
    navegador["Navegador\n[Container: Navegador Web]\nInterface utilizada pelos usuários para acessar o sistema"]

    subgraph aws["AWS — conta do projeto"]
        subgraph app["Compute"]
            spa["Aplicação Front-end\n[Container: React 19 + Vite 8 + Nginx Alpine]\nInterface web responsável pela interação com o sistema"]
            api["Aplicação Back-end\n[Container: Java 21 + Spring Boot 4]\nAplicação principal que gerencia a conexão com banco de dados, API e mensageria"]
        end
        subgraph dados["Persistência"]
            mysql[("Banco de Dados\n[Container: MySQL 8.0]\nArmazena usuários, joias, movimentações, maletas e demais dados transacionais")]
            fotos[("Armazenamento de Imagens\n[Container: Amazon S3]\nArmazena as fotos das joias cadastradas no sistema")]
            lake[("Data Lake\n[Container: Amazon S3]\nArmazena os dados em camadas Bronze / Silver / Gold para análises futuras")]
            lambda["Função de Ingestão Bronze\n[Container: AWS Lambda (Python 3.11)]\nExecuta a ingestão mensal de dados para a camada Bronze do Data Lake — Stub"]
        end
        subgraph mensag["Mensageria"]
            rabbit[("Broker de Mensagens\n[Container: RabbitMQ 3]\nGerencia a fila assíncrona para envio de notificações por e-mail")]
        end
    end

    brevo["Brevo\n[Container: Brevo API]\nServiço externo responsável pelo envio de e-mails transacionais"]
    viacep["ViaCEP\n[Container: ViaCEP API]\nServiço externo responsável pela consulta e autocompletar de CEPs"]

    navegador -->|"HTML/JS/CSS (SPA)"| spa
    navegador -->|"JSON sobre /api/* + cookie authToken<br/>(proxy nginx/vite → :8080)"| api
    spa -->|"axios com withCredentials"| api
    api -->|"JPA / JDBC — entidades e saldos"| mysql
    api -->|"SDK v2 — PUT/GET/DELETE de imagens"| fotos
    api -->|"publica NotificacaoEventoDto"| rabbit
    rabbit -->|"consumo (CrmNotificadorConsumer)"| api
    api -->|"OpenFeign HTTPS — e-mail transacional"| brevo
    spa -->|"fetch direto — consulta CEP<br/>(viacepApi.js)"| viacep
    lambda -->|"grava camada bronze"| lake

    classDef sistema fill:#1168BD,stroke:#0B4884,color:#fff
    classDef cont fill:#438DD5,stroke:#1168BD,color:#fff
    classDef dado fill:#DDDDDD,stroke:#6B6B6B,color:#000
    classDef ext fill:#999999,stroke:#6B6B6B,color:#fff
    classDef stub fill:#EEEEEE,stroke:#999999,color:#666,stroke-dasharray: 4 3
    class spa,api sistema
    class brevo,viacep,rabbit ext
    class mysql,fotos,lake dado
    class lambda stub
```

### Tabela de contêineres

| Contêiner | Tecnologia | Responsabilidade |
| --- | --- | --- |
| **SPA** | React 19, Vite 8, React Router 7, axios, lucide-react (JS puro, sem TS) | UI: login, dashboard, movimentações, inventário, maletas, contribuintes (revendedoras/fornecedores/funcionários). Servida por nginx; `nginx.conf` faz `location /api/ → api.backend:8080` |
| **API** | Spring Boot 4.0.5 (MVC, Security 7, Data JPA), Maven único `estoque/` | REST (`controller` → `service` → `database`), JWT stateless, strategies de movimentação, rotinas `@Scheduled`, notificações via fila |
| **MySQL 8** | `mysql:8.0` em EC2, `ddl-auto=update` + Flyway (migrações Java) | Fonte única de dados transacionais (banco `caeca`) |
| **RabbitMQ** | `rabbitmq:3-management`, porta 5672 só para o SG do backend | Desacopla eventos de negócio do envio de e-mails (fila durável `caecaFila`) |
| **S3 fotos** | `bucket-fotos-caeca` (CloudFormation) | Imagens das joias; acesso público de leitura |
| **Data lake** | `bucket-bronze/silver/gold-caeca` (CloudFormation) | Camadas do lake — suporte ao `analiseDados/` |
| **Lambda bronze** | `funcao-python-bronze` + EventBridge `cron(0 9 1 * ? *)` | Ingestão mensal p/ bronze — **stub** (código placeholder) |
| **Brevo** | API externa (Feign) | E-mails transacionais/CRM (templates: boas-vindas, aniversário, estoque baixo, peça parada, consignação, vencimento de maleta, convite) |
| **ViaCEP** | API externa | Autocompletar endereços — chamado **direto pelo SPA** (`service/viacepApi.js` → `fetch https://viacep.com.br/ws/{cep}/json/`). O backend mantém endpoint legado `GET /enderecos/cep/{cep}` (`ViaCepClient`/Feign), não usado pelo front |

---

## Nível 3 — Componentes da API (BackEnd)

Fluxo fixo: `controller` → `service` → `database/{model,repository}`.

```mermaid
flowchart TD
    subgraph api["Aplicação Back-end\n[Container: Java 21 + Spring Boot 4]\nAplicação principal que gerencia a conexão com banco de dados, API e mensageria"]
        ctrl["Controllers\n[Container: Spring MVC @RestController]\nCamada de entrada, validação, tratamento de requisições e autorização"]
        svc["Services\n[Container: Camada de Negócio]\nRegras de domínio, orquestração e controle de transações"]
        rot["Scheduled Jobs\n[Container: Spring @Scheduled]\nRotinas automatizadas diárias (08:00) e mensais (dia 1 às 09:00)"]
        strat["Movimentação Strategy\n[Container: Strategy Pattern]\nImplementações: EntradaStrategy, SaidaStrategy e ReentradaStrategy"]
        arq["Arquivo Strategy\n[Container: Strategy Pattern]\nImplementação: ArquivoS3Strategy para abstração do armazenamento de arquivos"]
        dto["DTOs + Mappers\n[Container: Camada de Transporte]\nResponsável pelo mapeamento entre DTOs e Entidades JPA"]
        repo["Repositories\n[Container: Spring Data JPA]\nCamada de acesso a dados com suporte à paginação"]
        model["Entities\n[Container: JPA Entities]\nModelagem das entidades de domínio do sistema"]
        prod["Notificação Producer\n[Container: RabbitMQ Publisher]\nPublica eventos de notificação na fila caecaFila"]
        cons["Notificação Consumer\n[Container: RabbitMQ Listener]\nConsome eventos da fila e encaminha para o serviço de e-mail"]
        feign["Feign Clients\n[Container: Spring Cloud OpenFeign]\nClientes HTTP para integração com Brevo e ViaCEP (legado)"]
        sec["Configurações\n[Container: Spring Security + Config]\nConfiguração de JWT, CORS, S3, Swagger e OpenAPI"]
        exc["Exception Handler\n[Container: Spring @ControllerAdvice]\nTratamento global de exceções de domínio e retorno de erros padronizados"]
    end

    mysql[("Banco de Dados\n[Container: MySQL 8.0]\nArmazena dados transacionais do sistema")]
    rabbit[("Broker de Mensagens\n[Container: RabbitMQ 3]\nFila assíncrona para envio de notificações por e-mail")]
    s3[("Armazenamento de Imagens\n[Container: Amazon S3]\nArmazena fotos das joias cadastradas")]
    brevo["Brevo\n[Container: Brevo API]\nServiço externo para envio de e-mails transacionais"]
    viacep["ViaCEP\n[Container: ViaCEP API]\nServiço externo para consulta de CEPs"]

    ctrl -->|"chama"| svc
    svc -->|"delega"| strat
    svc -->|"persiste imagens"| arq
    svc -->|"DTO ↔ entidade"| dto
    svc -->|"consultas"| repo
    strat -->|"grava movimentação + ajusta saldo"| repo
    rot -->|"gera alertas (maleta, peça parada, estoque)"| svc
    repo --> model
    model --> mysql
    arq --> s3
    svc -->|"publica evento"| prod
    prod --> rabbit
    rabbit -->|"entrega"| cons
    cons -->|"HTTPS (fire-and-forget)"| brevo
    feign -.->|"ViaCepClient — legado:<br/>rota não consumida pelo front"| viacep
    sec -.->|"filtra requisições"| ctrl
    exc -.->|"mapeia exceções de domínio"| ctrl

    classDef comp fill:#85BBF0,stroke:#0B4884,color:#000
    classDef dado fill:#DDDDDD,stroke:#6B6B6B,color:#000
    classDef ext fill:#999999,stroke:#6B6B6B,color:#fff
    class ctrl,svc,rot,strat,arq,dto,repo,model,prod,cons,feign,sec,exc comp
    class mysql,rabbit,s3 dado
    class brevo,viacep ext
```

### Observações por componente

- **`strategy/movimentacao`** — `MovimentacaoService.cadastrar` escolhe a strategy
  via `suporta(motivo)`; para um novo tipo de movimentação cria-se uma strategy nova
  (nunca `if` no service). Os tipos atuais são `ENTRADA`, `SAIDA`, `REENTRADA`
  (`TipoMovimentacaoEnum`).
- **Maletas** — modeladas por `MovimentacaoRevendedora` (nome, `dataLimite`,
  `StatusMaleta`: `ABERTO → EM_CONSIGNACAO → EM_ACERTO → FINALIZADA | ERRO`,
  relação 1:N com `Movimentacao`); expostas sob `/vendas`.
- **Notificações** — `NotificacaoBrevoService` / `MaletaNotificacaoService` /
  `UsuarioNotificacaoService` → `NotificacaoProducer` → fila →
  `CrmNotificadorConsumer` → `BrevoClient`. O registro de envio (evita reenvio)
  fica em `NotificacaoEnvio`. O efeito é *fire-and-forget*: falha só loga.
- **Segurança** — JWT stateless (`JJWT 0.12.6`); token no **cookie HttpOnly
  `authToken`** (ou header `Authorization: Bearer`); rotas públicas em
  `URLS_PERMITIDAS` (login, convites, swagger, actuator); admin via
  `@PreAuthorize("hasAuthority('ROLE_ADMIN')")`; CORS fixo em
  `localhost:5173` / `localhost:3000`.
- **Dashboards** — `DashboardController` (`/dashboard`: top-cinco, giro-estoque,
  motivos de saída/reentrada, top-5 mais frios, índice de peças frias) +
  `DashboardComparacaoController` (`/api/dashboard/kpis-e-grafico`).

### Mapeamento de endpoints (resumo)

| Base | Operações |
| --- | --- |
| `/usuarios` | `POST` criar, `POST /login`, `POST /logout`, `GET`, `GET /me` |
| `/usuarios/convites` | criar, reenviar, validar token, definir senha (primeiro acesso) |
| `/joias` | `GET` paginado, `GET/{id}`, `POST`, `PATCH/{id}` |
| `/movimentacoes` | `GET`, `GET/{id}`, `POST` (multipart/form-data) |
| `/vendas` (maletas) | `GET`, `GET/{id}`, `PATCH/{id}`, `POST` |
| `/revendedoras`, `/fornecedores` | CRUD + status |
| `/enderecos` | CRUD + `GET /cep/{cep}` (ViaCEP — **legado**, o front consulta direto) |
| `/arquivos` | upload/delete de foto por joia (S3) |
| `/categorias`, `/materiais`, `/banhos`, `/motivos-movimentacao` | leitura + criação |
| `/tipos-movimentacao`, `/status-maletas` | `GET` |
| `/dashboard`, `/api/dashboard` | métricas e gráficos |

---

## Nível 3 — Componentes do frontend (FrontEnd/caeca)

```mermaid
flowchart TD
    subgraph spa["Aplicação Front-end\n[Container: React 19 + Vite 8 + Nginx Alpine]\nInterface web responsável pela interação com o sistema"]
        routes["Rotas\n[Container: React Router 7]\nGerencia rotas, proteção de acesso, ErrorBoundary e navegação"]
        layout["Layout\n[Container: Componentes de Estrutura]\nHeader, Footer, Layout e CardForm"]
        ui["UI Components\n[Container: Componentes Reutilizáveis]\nBotões, Inputs, Selects, Modais e Toasts"]
        pages["Páginas\n[Container: Páginas da Aplicação]\nLogin, Primeiro Acesso, Dashboard, Movimentações, Joias, Maletas e Contribuintes"]
        hooks["Hooks & Utils\n[Container: Lógica Compartilhada]\nuseFetch, apiError, toastBus, fetchAllPages e Mappers"]
        client["HTTP Client\n[Container: Axios]\nGerencia requisições com baseURL '/api', withCredentials e interceptor de autenticação"]
        apis["API Services\n[Container: Camada de Integração]\nAbstrações por domínio para comunicação com o Back-end"]
        viacepapi["ViaCEP Service\n[Container: Fetch API]\nConsulta direta à API ViaCEP sem uso do Axios"]
    end

    api["Aplicação Back-end\n[Container: Java 21 + Spring Boot 4]\nAplicação principal que gerencia a conexão com banco de dados, API e mensageria"]
    viacep["ViaCEP\n[Container: ViaCEP API]\nServiço externo responsável pela consulta de endereços por CEP"]

    navegador["Navegador\n[Container: Navegador Web]\nInterface utilizada pelos usuários para acessar o sistema"]
    navegador --> routes
    routes --> pages
    pages --> layout
    pages --> ui
    pages --> hooks
    hooks --> apis
    apis --> client
    client -->|"JSON + Cookie authToken (HttpOnly)\n401 → limpa storage e redireciona para /login"| api
    pages -->|"Autocompletar endereço por CEP"| viacepapi
    viacepapi -->|"GET https://viacep.com.br/ws/{cep}/json/"| viacep

    classDef comp fill:#85BBF0,stroke:#0B4884,color:#000
    classDef ext fill:#999999,stroke:#6B6B6B,color:#fff
    class routes,layout,ui,pages,hooks,client,apis,viacepapi comp
    class api,viacep ext
```

- **Dev:** proxy do Vite (`/api → http://localhost:8080`, removendo o prefixo).
  **Prod:** mesmo repasse no `nginx.conf` do container.
- Rotas protegidas por `ProtectedRoute` (sessão derivada do cookie + perfil no
  `localStorage`); `/login` e `/primeiro-acesso` são públicas.

---

## Nível 4 — Diagrama de implantação

Infra gerida por **Terraform** (rede, compute, ALB, EFS) + **CloudFormation**
(buckets S3 — SCP da conta impede o Terraform de gerenciá-los).

```mermaid
flowchart TD
    usuario["Usuário\n[Container: Navegador Web]\nAcessa o sistema Ca&Cá via navegador"] -->|"HTTPS"| internet["Internet\n[Container: Rede Pública]\nTráfego HTTPS entre usuário e infraestrutura"]

    subgraph vpc["VPC — us-east-1 (2 AZs)"]
        alb["Application Load Balancer\n[Container: AWS ALB]\nBalanceia o tráfego entre as instâncias Front-end"]

        subgraph fronts["EC2 - Front-end (front-1a / front-1b) — t3.small"]
            nginx["Aplicação Front-end\n[Container: Nginx Alpine + Build React]\nServe arquivos estáticos e realiza proxy para a Aplicação Back-end"]
        end

        subgraph backs["EC2 - Back-end (back-1a / back-1b) — t2.small"]
            boot["Aplicação Back-end\n[Container: Docker (eclipse-temurin:21-jre)]\nExecuta a API Spring Boot na porta 8080"]
        end

        subgraph dbs["EC2 - Banco de Dados (db-1a) — t3.small"]
            mysqldb["Banco de Dados\n[Container: Docker (mysql:8.0)]\nArmazena dados transacionais do sistema"]
        end

        subgraph rmq["EC2 - Mensageria (rabbitmq-1a) — dedicada"]
            rabbithost["Broker de Mensagens\n[Container: Docker (rabbitmq:3-management)]\nGerencia fila AMQP restrita ao Back-end"]
        end

        efs["Armazenamento Compartilhado\n[Container: Amazon EFS]\nCompartilha arquivos entre instâncias quando necessário"]
    end

    subgraph cfn["CloudFormation — caeca-s3"]
        bktfotos[("Armazenamento de Imagens\n[Container: Amazon S3]\nArmazena fotos das joias — acesso público")]
        bktlake[("Data Lake\n[Container: Amazon S3]\nArmazena dados em camadas Bronze / Silver / Gold")]
    end

    subgraph event["EventBridge + Lambda"]
        cron["Agendador EventBridge\n[Container: Amazon EventBridge]\nDispara a função mensalmente: cron(0 9 1 * ? *)"]
        lambdafn["Função de Ingestão Bronze\n[Container: AWS Lambda (Python 3.11)]\nExecuta ingestão para camada Bronze do Data Lake — Stub"]
        cron --> lambdafn
        lambdafn --> bktlake
    end

    usuario --> internet --> alb
    alb --> nginx
    nginx -->|"Proxy /api → backend:8080"| boot
    boot -->|"Conexão JDBC (3306)"| mysqldb
    boot -->|"AMQP (5672)"| rabbithost
    boot -->|"SDK AWS v2 (S3)"| bktfotos
    boot -.->|"Montagem EFS"| efs

    classDef infra fill:#E9E9E9,stroke:#6B6B6B,color:#000
    classDef dado fill:#DDDDDD,stroke:#6B6B6B,color:#000
    classDef ext fill:#999999,stroke:#6B6B6B,color:#fff
    classDef stub fill:#EEEEEE,stroke:#999999,color:#666,stroke-dasharray: 4 3
    class alb,nginx,boot,mysqldb,rabbithost,efs,cron infra
    class bktfotos,bktlake dado
    class lambdafn stub
    class usuario,internet ext
```

> Bootstrap por `user_data` (`cloud_init_{front,back,db,rabbitmq}.yaml`) — roda
> apenas na criação da instância; mudanças exigem `terraform apply -replace`.

---

## Fluxos dinâmicos

### 1. Autenticação (login → cookie HttpOnly)

```mermaid
sequenceDiagram
    autonumber
    participant U as Usuário (navegador)
    participant S as SPA React
    participant A as API Spring Boot
    participant D as MySQL

    U->>S: informa e-mail e senha
    S->>A: POST /usuarios/login
    A->>D: valida credenciais
    D-->>A: usuário + perfil (ROLE_ADMIN / ROLE_USER)
    A-->>S: 200 + Set-Cookie: authToken (HttpOnly, SameSite=Strict)
    S->>S: grava nome/perfil no localStorage e navega para /
    Note over S,A: demais chamadas enviam o cookie automaticamente (withCredentials)
    A-->>S: 401 (token ausente/inválido/expirado — JWT_VALIDITY 1800s)
    S->>S: interceptor limpa storage e redireciona para /login
```

### 2. Movimentação de estoque (ex.: saída)

```mermaid
sequenceDiagram
    autonumber
    participant O as Operadora
    participant S as SPA (/movimentacoes/saida)
    participant A as API
    participant X as MovimentacaoContext
    participant D as MySQL

    O->>S: preenche formulário (joia, motivo, quantidade, fotos…)
    S->>A: POST /movimentacoes (multipart/form-data)
    A->>A: valida DTO + autentica (cookie JWT)
    A->>X: executar(dto)
    X->>X: seleciona strategy por suporta(motivo): SaidaStrategy
    X->>D: grava Movimentacao e ajusta saldo da Joia
    D-->>X: confirmado
    X-->>A: Movimentacao
    A-->>S: 201 + payload
    S-->>O: toast de sucesso
    Note over A,D: se quantidade insuficiente → QtdInsuficienteException<br/>GlobalExceptionHandler devolve 4xx
```

### 3. Notificação assíncrona (fila → Brevo)

```mermaid
sequenceDiagram
    autonumber
    participant Ctx as Serviço de negócio<br/>(Revendedora/Maleta/Estoque/Convite)
    participant P as RabbitMqNotificacaoProducer
    participant R as RabbitMQ<br/>caecaExchange → caecaFila
    participant Cons as CrmNotificadorConsumer
    participant F as BrevoClient (Feign)
    participant B as Brevo
    participant Rev as Revendedora

    Ctx->>P: enviarMensagem(email, nome, templateId, params)
    P->>R: publica NotificacaoEventoDto (routing key caecaKey)
    R->>Cons: entrega na caecaFila (@RabbitListener)
    Cons->>F: enviarEmailTransacional(apiKey, request)
    F->>B: POST /v3/smtp/email
    B-->>Rev: e-mail (boas-vindas, aniversário, estoque baixo,<br/>peça parada, consignação, vencimento de maleta, convite)
    Note over Cons,F: fire-and-forget: falha só registra log,<br/>nunca derruba o request que originou o evento
```

### 4. Rotinas agendadas

```mermaid
sequenceDiagram
    autonumber
    participant CR as Spring @Scheduled<br/>(America/Sao_Paulo)
    participant RS as RotinaScheduler<br/>cron 0 0 8 * * *
    participant ES as EstoqueRotinaMensalService<br/>cron 0 0 9 1 * *
    participant MN as MaletaNotificacaoService /<br/>EstoqueNotificacaoService
    participant P as Producer → RabbitMQ → Brevo
    participant EB as EventBridge
    participant L as Lambda bronze

    CR->>RS: diário 08:00 — executa RotinaDiaria[]
    RS->>MN: alertas de maleta (vencimento, consignação)
    MN->>P: evento de notificação
    CR->>ES: mensal dia 1 09:00 — joias paradas há N dias
    ES->>P: notificação de peça parada / estoque baixo
    EB->>L: cron(0 9 1 * ? *) — pipeline de dados
    L->>L: grava data lake bronze (stub — ainda não implementado)
```

---

## Legenda da notação

| Símbolo | Significado |
| --- | --- |
| Retângulo azul escuro | Pessoa / ator |
| Retângulo azul | Sistema ou contêiner de software em execução |
| Retângulo cinza | Sistema externo (fora do nosso controle) |
| Elipse cinza | Dados / armazenamento (banco, bucket, fila) |
| Retângulo tracejado | Componente previsto / stub (não implementado) |
| Seta contínua | Comunicação síncrona (requisição) |
| Seta pontilhada | Influência / dependência opcional |
| Diagrama de sequência | Fluxo dinâmico (interação temporal) |

Modelo alinhado ao C4 de Simon Brown: **Contexto (1)** → **Contêineres (2)** →
**Componentes (3)** → **Código/Implantação (4)**, mais vistas dinâmicas
(sequência) para os fluxos críticos.
