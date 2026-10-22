# Ca&Ca — Visão Geral do Projeto

Documento de entrada para agentes/orquestradores. Descreve a arquitetura, os
comandos de build/teste, as convenções e os limites conhecidos de cada repositório.
**Leia isto antes de tocar em qualquer arquivo.**

Idiomas de domínio, comentários, commits e nomes de arquivos são em **português**.
Manter esse padrão ao editar.

---

## 1. O que é

Sistema web de gestão de **estoque esemijoias** (joias) com controle de
movimentações, maletas, revendedoras, fornecedores e dashboards.

Fluxo funcional principal: **entrada → banho/categorização → maleta → saída →
reentrada/venda**, com histórico de movimentações e métricas (giro, top 5, peças frias).

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

## 3. BackEnd — API REST de estoque

Spring Boot **4.0.5** / Java **21**, Maven único em `BackEnd/estoque/`.

### Comandos (sempre a partir de `BackEnd/estoque/`)

| Ação | Comando |
| --- | --- |
| Testes | `mvn test` |
| Suíte + gate de cobertura | `mvn verify` |
| Um teste | `mvn test -Dtest=JoiaServiceTest` |
| Teste com `@Nested` | `mvn test -Dtest='JoiaServiceTest*'` |
| Pular JaCoCo | `mvn test -Djacoco.skip=true` |
| Subir a API | `mvn spring-boot:run` |

- `mvn verify` **falha se branch coverage < 70%** (JaCoCo). Sempre adicione testes junto com código.
- Não existe `mvnw` funcional; use `mvn` do sistema (3.8+).
- Suíte: ~200 testes em 28 classes, ~1,5 min. Roda offline com `-o` após o 1º build.

### Configuração

Fonte: `BackEnd/estoque/src/main/resources/application.properties`

| Variável | Default | Obrigatória |
| --- | --- | --- |
| `JWT_SECRET` | — | **sim** (Base64, `openssl rand -base64 48`) |
| `BREVO_API_KEY` | — | **sim** |
| `JWT_VALIDITY` | `1800` | não |
| `APP_STORAGE_TYPE` | `s3` | não |
| `APP_STORAGE_S3_BUCKET` | `bucket-fotos-caeca` | não |
| `APP_STORAGE_S3_REGION` | `us-east-1` | não |

Banco local: `jdbc:mysql://localhost:3306/caeca` · `root` / `root`.
Upload multipart limitado a **10 MB**.

⚠️ O `.env.example` **não é lido** por nada (não há dotenv). Exporte as variáveis no shell
ou configure no run config da IDE. Ao criar uma property obrigatória sem default,
adicione **também** em `src/test/resources/application.properties`, senão
`EstoqueApplicationTests.contextLoads` quebra.

⚠️ **Sem seed de dados.** `ddl-auto=update` cria o schema a partir das entidades e não há
`schema.sql`/`data.sql`. Para obter um token local é preciso inserir um usuário direto no MySQL.

### Arquitetura

Fluxo fixo: `controller` → `service` → `database/{model,repository}`.

```
com.caeca.estoque
├── controller/<dominio>   # @RestController por domínio
├── service/<dominio>      # regra de negócio
├── database/model         # entidades JPA
├── database/repository    # Spring Data
├── dto/<dominio>          # DTOs de entrada/saída
├── strategy/movimentacao  # Entrada | Saida | Reentrada
├── strategy/arquivo       # impl S3
├── config/                # SecurityConfiguracao, CORS, Swagger
├── consumer/              # CrmNotificadorConsumer (async → Brevo)
├── client/                # BrevoClient (Feign)
├── exception/, handler/   # exceções + GlobalExceptionHandler
```

**Endpoints por domínio** (todos sob `localhost:8080`):

| Base | Operações |
| --- | --- |
| `/usuarios` | `POST` criar, `POST /login`, `POST /logout`, `GET`, `GET /me` |
| `/joias` | `GET` (paginado), `GET /{id}`, `POST`, `PATCH /{id}` |
| `/movimentacoes` | `GET`, `GET /{id}`, `POST` (multipart/form-data) |
| `/vendas` | `GET`, `GET /{id}`, `PATCH /{id}`, `POST` |
| `/revendedoras` | `GET`, `GET /{id}`, `POST`, `PUT /{id}`, `PATCH /{id}` |
| `/fornecedores` | `GET`, `GET /{id}`, `POST`, `PUT /{id}`, `PATCH /{id}/status` |
| `/enderecos` | CRUD + `GET /cep/{cep}` |
| `/arquivos` | `PUT /joia/{joiaId}`, `DELETE /joia/{joiaId}`, `DELETE /{id}` |
| `/categorias`, `/materiais`, `/banhos`, `/motivos-movimentacao` | `GET`, `GET /{id}`, `POST` |
| `/tipos-movimentacao`, `/status-maletas` | `GET` |
| `/dashboard` | `/top-cinco`, `/giro-estoque`, `/graficos/motivos-saida`, `/graficos/motivos-reentrada`, `/top-cinco-mais-frios`, `/indice-pecas-frias` |

Listagens usam `Pageable` com **`@PageableDefault(size = 10)`** — o padrão do projeto é 10.

### Autenticação

JWT stateless. Token chega pelo **cookie HttpOnly `authToken`** (prioridade) ou header
`Authorization: Bearer`. Rotas públicas ficam em `URLS_PERMITIDAS`
(`config/SecurityConfiguracao.java`): `/h2-console/**`, `/swagger-ui/**`,
`/usuarios/login`, `/usuarios/logout`. Admin via `@PreAuthorize("hasAuthority('ROLE_ADMIN')")`.

⚠️ CORS está **fixo** em `localhost:5173` e `localhost:3000`. Ajuste em `SecurityConfiguracao`,
nunca no controller.

### Convenções críticas (backend)

- Mapeamento **manual** via classes `XMapper` com métodos estáticos `toEntity`,
  `toResponse`, `of`. **Sem MapStruct.**
- DTOs: predominam `XRequestDto`/`XResponseDto`; domínios novos (`joia`, `revendedora`)
  usam `EntradaDto`/`SaidaDto`/`AtualizacaoDto`. **Copie o padrão do domínio vizinho.**
- Erros: lance `NotFoundException`, `BadRequestException`, `ConflictException`,
  `QtdInsuficienteException` e deixe o `GlobalExceptionHandler` mapear. **Não** faça
  try/catch de domínio nos controllers.
- `MovimentacaoService.cadastrar` escolhe a `MovimentacaoStrategy` via `suporta(motivo)`
  e delega ao `MovimentacaoContext`. Para novo tipo de movimentação, crie uma strategy.
- POJOs com getters/setters manuais; Lombok é exceção (9 arquivos). Siga o estilo do
  arquivo que está editando.
- Alguns pacotes têm **maiúsculas** (`service/Material`, `service/NotificacaoBrevo`,
  `controller/DashEntradaXSaida`). Ao criar pasta, respeite o nome já usado pelo domínio.
- Ainda se usa `jakarta.transaction.Transactional` em vários services — não padronize
  sem necessidade.
- CRM/e-mail: `RevendedoraService` publica `NotificacaoEventoDto` →
  `CrmNotificadorConsumer` (`@Async @EventListener`) → `BrevoClient` (Feign).

### Armadilhas de Spring Boot 4 / Spring 7

Verifique `BackEnd/AGENTS.md` antes de escrever imports:

- Jackson 3 é `tools.jackson.databind.ObjectMapper`. Não introduza mais Jackson 2.
- Slices de teste: `org.springframework.boot.webmvc.test.autoconfigure.WebMvcTest`
  (não `org.springframework.boot.test.autoconfigure.*`).
- Contexto de teste: `@MockitoBean` (não `@MockBean`).
- `AntPathRequestMatcher` foi removido no Security 7 — use `requestMatchers(...)`.
- Testes: services com `MockitoExtension` + `@Mock`/`@InjectMocks`; controllers com
  `@WebMvcTest` + `@AutoConfigureMockMvc(addFilters = false)` + `@MockitoBean` para **cada**
  dependência. H2 (`MODE=MySQL`) só para o teste de context load. Convenção forte:
  `@Nested` por método e `@DisplayName` em português começando com "Deve ...".

---

## 4. FrontEnd — React 19 + Vite

Local: `FrontEnd/caeca/`. Deps: `axios`, `react-router-dom` 7, `lucide-react`.
Sem TypeScript, sem testes, sem Storybook.

```bash
cd FrontEnd/caeca
npm install
npm run dev       # Vite dev server
npm run build     # production build
npm run lint      # eslint .
npm run preview
```

### Cliente HTTP

`src/service/api.js`: axios com `baseURL: 'http://localhost:8080'` **hardcoded** e
`withCredentials: true` (envia o cookie de auth). Interceptor global trata `401`
 limpando `localStorage` e redirecionando para `/login`.

⚠️ O baseURL está fixo no código — não há `.env` / `VITE_*`. Para apontar para outro
ambiente é preciso editar `api.js`.

Camada de API por domínio em `src/service/*Api.js`; mappers de payload em `src/utils/*Mapper.js`;
`src/utils/fetchAllPages.js` pagina até concatenar todas as páginas da API.

### Rotas (`src/main.jsx`)

Rotas sob `<ProtectedRoute>` + `<Layout>`:

```
/login
/                                Home (dashboard)
/contribuintes                  CadastroContribuintes
/movimentacoes                   MovimentacaoEscolhas
/movimentacoes/entrada           EntradaJoia
/movimentacoes/entradaexistente  EntradaJoiaExistente
/movimentacoes/reentrada         ReentradaJoia
/movimentacoes/saida             SaidaJoia
/joias                           InventarioProdutos
/joias/:id                       JoiaDetalhes
/joias/movimentacoes             MovimentacaoEstoque
/joias/maletas                   Maletas
/joias/maletas/:id               MaletaDetalhes
```

⚠️ **Nav do `Header.jsx` aponta para rotas não registradas**: `/perfil`,
`/contribuintes/revendedoras`, `/contribuintes/fornecedores`, `/contribuintes/funcionarios`,
`/historico/gerais`, `/metricas`. Clicar nelas não renderiza nada. Ao adicionar página,
registre a rota em `main.jsx` **e** o link no `Header.jsx`.

⚠️ `src/App.jsx` está vazio (renderiza `<></>`) — o roteamento real vive em `main.jsx`.
Não use `App.jsx` como ponto de entrada.

Estrutura: `components/{layout,ui,modals}`, `components/ProtectedRoute.jsx`,
`hooks/useFetch.jsx`, `pages/*`, `styles/`, CSS Modules em `pages/home/Home.module.css`.

---

## 5. Infra — AWS

Terraform para rede/compute/storage/ALB; **CloudFormation para os buckets S3**.

### Comandos

```bash
cd Infra

# 1. Buckets S3 (CloudFormation) — PRIMEIRO
aws cloudformation create-stack --stack-name caeca-s3 \
  --template-body file://s3-buckets.yaml --region us-east-1
aws cloudformation wait stack-create-complete --stack-name caeca-s3 --region us-east-1

# 2. Resto da infra (Terraform)
terraform init && terraform plan && terraform apply
terraform destroy
```

⚠️ Os buckets usam `DeletionPolicy: Retain`: `delete-stack` **não** os remove. Para
remover de verdade, esvazie com `aws s3 rb s3://<bucket> --force` antes.

⚠️ **Terraform não gerencia os buckets S3** (SCP da conta bloqueia leitura de object lock,
o que impede o TF de gerenciá-los). É por isso que existem duas ferramentas.

⚠️ `terraform.tfstate` e `.backup` estão **versionados no git** — podem conter dados
sensíveis. Não adicione mais segredos lá.

### Recursos

| Camada | Itens |
| --- | --- |
| Rede | VPC, subnets públicas/privadas em 2 AZs, IGW, NAT, route tables, 3 NACLs |
| Segurança | SGs separados para front / back / db |
| Compute | 2× EC2 front (`t3.small`, nginx + node:22-alpine), 2× EC2 back (`t2.small`, `eclipse-temurin:21-jre-jammy`), 1× EC2 db (`t3.small`, `mysql:8.0`) |
| Storage | EFS + mount targets, VPC endpoint S3 |
| Balanceamento | ALB + target group do frontend |
| Lambda | `funcao-python-bronze` (python3.11, **dummy/stub**), disparada por EventBridge `cron(0 9 1 * ? *)` |

Bootstrap das instâncias via `user_data` → `cloud_init_{front,back,db}.yaml` (instala
Docker/Compose, nginx, JRE 21, MySQL e faz `docker pull` da imagem base).

`Infra/local-stack/` contém scripts bash (`bootstrap.sh` + camadas `01-network.sh` …
`04-validation.sh`) para subir uma infraestrutura **local/distribuída em camadas** para
desenvolvimento.

### Buckets

| Bucket | Acesso |
| --- | --- |
| `bucket-fotos-caeca` | **público** (`PublicReadGetObject`) |
| `bucket-bronze-caeca` | privado |
| `bucket-silver-caeca` | privado |
| `bucket-gold-caeca` | privado |

Buckets `bronze/silver/gold` são a camada de data lake — casa com `analiseDados/`.

---

## 6. analiseDados — pipeline de dados (não iniciado)

Repositório existe, `bronze/` está **vazia**. Não há código, nem Glue, nem Athena,
nem Step Functions. Os buckets `bronze/silver/gold` e a Lambda `funcao-python-bronze`
(hoje stub) são os pontos de apoio previstos na arquitetura.

---

## 7. Regras de operação para agentes

1. **Trabalhe por repositório.** Um `git status` na raiz não existe. Sempre `cd` no
   subrepo correto e confira a branch antes de commitar.
2. **Não commite `.env`, segredos ou `.idea/`.** `Infra/terraform.tfstate` já está no
   git — não faça pior que isso.
3. **BackEnd:** rode `mvn verify` (não só `mvn test`) antes de propor PR — o gate de
   branch coverage em 70% reprova builds incompletos.
4. **Ao criar property obrigatória sem default**, replique em
   `src/test/resources/application.properties`.
5. **Ao criar rota no front**, registre em `main.jsx` e adicione o link no `Header.jsx`.
6. **Novo tipo de movimentação** vai como `MovimentacaoStrategy` nova, não como `if` no service.
7. **Novo endpoint** precisa ser adicionado explicitamente a `URLS_PERMITIDAS` se for público.
8. **Mapeamento de entidade↔DTO** é manual; não introduza MapStruct.
9. **Commits:** Conventional Commits em português. Branches `feat/...`, `refactor/...`,
   `chore/...` a partir de `main`, com PR para `main`.
   Exemplos: `feat: adiciona paginação com tamanho 10`, `refactor: aplicando solids`.
10. **Idioma:** domínio, comentários, `@DisplayName`, commits e PRs em português.
