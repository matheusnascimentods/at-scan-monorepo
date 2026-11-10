# 🎯 ATScan

![Node.js](https://img.shields.io/badge/Node.js-20.19%2B-339933?style=for-the-badge&logo=nodedotjs&logoColor=white)
![TypeScript](https://img.shields.io/badge/TypeScript-5-007ACC?style=for-the-badge&logo=typescript&logoColor=white)
![NestJS](https://img.shields.io/badge/NestJS-11-E0234E?style=for-the-badge&logo=nestjs&logoColor=white)
![Angular](https://img.shields.io/badge/Angular-21-DD0031?style=for-the-badge&logo=angular&logoColor=white)
![Tailwind CSS](https://img.shields.io/badge/Tailwind_CSS-4-06B6D4?style=for-the-badge&logo=tailwindcss&logoColor=white)
![Prisma](https://img.shields.io/badge/Prisma-6-2D3748?style=for-the-badge&logo=prisma&logoColor=white)
![PostgreSQL](https://img.shields.io/badge/PostgreSQL-16-4169E1?style=for-the-badge&logo=postgresql&logoColor=white)
![Google Gemini](https://img.shields.io/badge/Google_Gemini-ADK_1.3-8E75B2?style=for-the-badge&logo=googlegemini&logoColor=white)
![Docker](https://img.shields.io/badge/Docker-2496ED?style=for-the-badge&logo=docker&logoColor=white)
![Projeto pessoal](https://img.shields.io/badge/Projeto-Pessoal-003566?style=for-the-badge)
![License](https://img.shields.io/badge/License-MIT-yellow.svg?style=for-the-badge)
![PRs Welcome](https://img.shields.io/badge/PRs-welcome-brightgreen.svg?style=for-the-badge)

> Analise e otimize seu currículo para uma vaga com IA — sem cadastro, sem fricção.

O **ATScan** é um projeto pessoal: uma aplicação web que mede a compatibilidade entre um **currículo em PDF** e a **descrição de uma vaga**, do jeito que um sistema ATS (Applicant Tracking System) faria. Uma arquitetura de multi-agentes (Google Gemini + [Agent Development Kit](https://www.npmjs.com/package/@google/adk)) converte o PDF em Markdown, calcula um score ATS em quatro dimensões, aponta keywords ausentes e problemas de formatação, gera perguntas sobre as lacunas e, com as respostas do candidato, reescreve o currículo. Não há login nem conta: cada currículo, análise e otimização é identificado apenas pelo seu UUID.

Este repositório é um **monorepo npm workspaces** com duas aplicações:

| App | Caminho | Papel |
| :--- | :--- | :--- |
| **API** | [`apps/api`](apps/api) | Backend NestJS: upload do currículo, análise ATS, otimização, persistência (Prisma) e orquestração dos agentes de IA |
| **Web** | [`apps/web`](apps/web) | Frontend Angular: landing page e o analisador em 4 etapas (upload → vaga → resultado → otimização) |

```mermaid
sequenceDiagram
    autonumber
    actor User as Usuário
    participant Web as Frontend (Angular)
    participant API as Backend (NestJS)
    box rgb(240, 240, 240) AI Intelligence Layer (ADK)
        participant RP as ResumeParserAgent
        participant Orc as OrchestratorAgent
        participant Opt as OptimizerOrchestratorAgent
    end
    participant Gemini as Google Gemini
    participant DB as PostgreSQL

    User->>Web: Upload do PDF
    Web->>API: POST /resumes (multipart, campo file)
    API->>RP: extractMarkdownFromPdf (PDF em base64)
    RP->>Gemini: PDF como inlineData
    Gemini-->>RP: Markdown
    API->>DB: INSERT resumes
    API-->>Web: 201 { id, content, fileName, createdAt }

    User->>Web: Cola a descrição da vaga
    Web->>API: POST /ats/analyze { resumeId, resumeContent, jobDescription }
    API->>Orc: analyze()
    Orc->>Gemini: parsers → match/formato → consolidação → perguntas
    Orc-->>API: score, breakdown, keywords, recomendações, perguntas
    API->>API: Validação (Zod)
    API->>DB: INSERT analyses
    API-->>Web: 201 AnalyzeResponse (com id)

    User->>Web: Responde as perguntas
    Web->>API: POST /ats/optimize { analysisId, answers[] }
    API->>DB: SELECT analyses (currículo + vaga)
    API->>Opt: optimize()
    Opt->>Gemini: reanálise → reescrita → novo score
    Opt-->>API: newScore, gain, optimizedContent, changes
    API->>DB: INSERT optimizations
    API-->>Web: 201 OptimizeResponse (com id)
    Web-->>User: Comparativo de score + download .md
```

---

## ✨ O Problema
Candidatos perdem oportunidades não por falta de qualificação, mas porque o currículo não passa pelos filtros automáticos dos sistemas ATS. Esses sistemas fazem casamento de palavras-chave e penalizam formatações que seus parsers não leem, rejeitando currículos bons que simplesmente não usam os termos da vaga. O candidato nunca sabe o motivo da rejeição.

---

## 🚀 A Solução
O ATScan recebe o currículo em PDF e a descrição da vaga, e uma cadeia de agentes especializados estima o score ATS por categoria, lista exatamente quais keywords estão faltando, quais problemas de formatação existem e o que fazer primeiro. Se o candidato quiser, a IA faz perguntas sobre as lacunas e reescreve o currículo **usando apenas o que ele confirmou nas respostas**. Tudo sem cadastro, e cada resultado é persistido no PostgreSQL antes de voltar ao cliente.

---

## 🎯 Diferenciais
- **Zero fricção**: sem cadastro, login ou conta.
- **Score em 4 dimensões**: Keywords (35%), Semântica (25%), Formatação (20%) e Peso por Seção (20%), conforme o prompt do orquestrador.
- **Match semântico**: o `SemanticMatchAgent` reconhece equivalências entre termos da vaga e do currículo, não só a string exata.
- **Recomendações priorizadas**: 4 a 6 recomendações com prioridade (`Alta`, `Média`, `Baixa`) e impacto estimado em pontos.
- **Otimização interativa**: de 3 a 5 perguntas geradas a partir das keywords ausentes; respostas em branco são ignoradas.
- **Multi-agentes com prompts versionados**: cada agente tem seu `index.prompt.md` na própria pasta, o que torna o comportamento da IA auditável.
- **Saídas validadas**: a resposta de cada agente passa por um schema Zod antes de seguir no pipeline.
- **Download do resultado**: o currículo otimizado é baixado como `curriculo_otimizado.md`.

---

## 🛠️ Stack Tecnológica

Versões conforme declaradas em [`package.json`](package.json), [`apps/api/package.json`](apps/api/package.json), [`apps/web/package.json`](apps/web/package.json) e no [`docker-compose.yml`](docker-compose.yml).

### Backend (`apps/api`)

| Camada | Tecnologia |
|---|---|
| **Runtime** | Node.js (não há `engines` declarado; o Angular 21 do frontend exige `^20.19`, `^22.12` ou `>=24`) |
| **Linguagem** | TypeScript ^5.7.3 |
| **Framework** | NestJS ^11.0.1 (`@nestjs/platform-express`) |
| **IA / Agentes** | `@google/adk` ^1.3.0 com Google Gemini (modelo padrão `gemini-2.0-flash`) |
| **ORM & Migrations** | Prisma ^6.10.1 (`@prisma/client` ^6.10.1) |
| **Banco de Dados** | PostgreSQL 16 (imagem `postgres:16-alpine`) |
| **Upload** | Multer (via `@nestjs/platform-express`, em memória, limite de 5 MB) |
| **Validação** | Zod ^4.4.3 (declarado no `package.json` da raiz) |
| **Testes** | Jest ^30.0.0, ts-jest e Supertest ^7.0.0 |
| **Qualidade** | ESLint ^9.18.0, Prettier ^3.4.2 |

### Frontend (`apps/web`)

| Camada | Tecnologia |
|---|---|
| **Framework** | Angular 21.2.17 (componentes standalone, `@angular/router`, `@angular/forms`) |
| **Build / Dev server** | Angular CLI 21.2.9 + `@angular-devkit/build-angular` 21.2.3 (builder `application`) |
| **Linguagem** | TypeScript ~5.9.3 |
| **Estilos** | Tailwind CSS ^4.3.1 via `@tailwindcss/postcss` |
| **HTTP** | `HttpClient` do Angular |
| **Tipos** | Zod (schemas em `index.schema.ts`, usados para inferir os tipos) |
| **Testes** | Karma ~6.4.0 + Jasmine ~5.6.0 (`ng test`) |

---

## 🏛️ Arquitetura

### Visão geral do monorepo

```mermaid
flowchart LR
    Browser[Navegador] --> Web["apps/web<br/>Angular<br/>:4200 (ng serve)"]
    Web -->|"HTTP/JSON e multipart<br/>(environment.apiUrl)"| API["apps/api<br/>NestJS<br/>:3000"]
    API -->|Prisma| DB[("PostgreSQL 16<br/>:5433 no host")]
    API -->|"@google/adk"| Gemini[Google Gemini API]
```

- O [`package.json`](package.json) da raiz declara o workspace `apps/*`, a dependência compartilhada `zod` e os scripts `db:*` do banco.
- Não há pacote compartilhado entre API e Web: cada app tem seu próprio `index.schema.ts` e o contrato é o JSON trocado via HTTP.
- O Docker Compose sobe **apenas o PostgreSQL**; API e Web rodam localmente com Node.js.

### Backend (`apps/api`)

Cada domínio é um módulo NestJS em [`apps/api/src`](apps/api/src), com arquivos prefixados por `index.` e o teste ao lado:

```
index.module.ts      ← Declaração do módulo NestJS
index.controller.ts  ← Rotas HTTP
index.service.ts     ← Regras de negócio, chamada dos agentes e persistência
index.schema.ts      ← Schemas Zod e tipos (DTOs)
*.spec.ts            ← Testes Jest ao lado do arquivo testado
```

| Módulo | Prefixo | Responsabilidade |
| :--- | :--- | :--- |
| `resumes` | `/resumes` | Upload do PDF, conversão para Markdown via IA e consulta por ID |
| `ats` | `/ats` | Análise ATS, otimização do currículo e consulta dos resultados persistidos |
| `agents` | — | Provê os 8 agentes ADK para os outros módulos |
| `prisma` | — | `PrismaService` global (conexão com o PostgreSQL) |

```mermaid
flowchart TD
    Req[Requisição HTTP] --> Ctrl[Controller]
    Ctrl -->|"/ats: safeParse Zod<br/>falha → 400"| Svc[Service]
    Ctrl -->|"/resumes: arquivo obrigatório,<br/>PDF, até 5 MB"| Svc
    Svc --> Agents["Agentes ADK<br/>(erro → 502)"]
    Agents --> Gemini[Google Gemini]
    Svc --> Prisma[PrismaService]
    Prisma --> DB[(PostgreSQL)]
    Svc -->|"Response schema (Zod)"| Res[Resposta JSON]
```

### 🤖 Arquitetura de Agentes

O módulo [`agents`](apps/api/src/agents) usa o **@google/adk**: cada agente é um `LlmAgent` executado por um `Runner` com `InMemorySessionService`, todos sobre o mesmo modelo (`GEMINI_MODEL`). O prompt de cada agente fica em `index.prompt.md` na própria pasta e é carregado com `fs.readFileSync` na inicialização. A resposta do modelo tem as cercas de código Markdown removidas, é convertida com `JSON.parse` e validada com o schema Zod correspondente de [`ats/index.schema.ts`](apps/api/src/ats/index.schema.ts).

```
OrchestratorAgent
├── ResumeParserAgent       → seções, peso por seção, keywords e sectionScore
├── JobParserAgent          → requisitos obrigatórios vs desejáveis, senioridade e responsabilidades
├── SemanticMatchAgent      → keywords encontradas/ausentes, keywordsScore e semanticScore
├── FormatCheckerAgent      → problemas de formatação ATS (critical/moderate/minor) e formatScore
└── QuestionGeneratorAgent  → 3 a 5 perguntas sobre as keywords ausentes

OptimizerOrchestratorAgent
├── OrchestratorAgent       → reexecuta a análise completa
└── ResumeOptimizerAgent    → reescreve o currículo com base na análise + respostas
```

**Análise (`POST /ats/analyze`)**

```mermaid
flowchart LR
    In["resumeContent<br/>jobDescription"] --> P1[ResumeParserAgent]
    In --> P2[JobParserAgent]
    P1 --> M[SemanticMatchAgent]
    P2 --> M
    In --> F[FormatCheckerAgent]
    M --> C["OrchestratorAgent<br/>consolida score,<br/>breakdown e recomendações"]
    F --> C
    P1 --> C
    P2 --> C
    M --> Q[QuestionGeneratorAgent]
    P2 --> Q
    C --> Out[AnalyzeResponse]
    Q --> Out
```

1. `ResumeParserAgent` e `JobParserAgent` rodam **em paralelo** (`Promise.all`).
2. Em seguida, `SemanticMatchAgent` e `FormatCheckerAgent` rodam **em paralelo**.
3. O `OrchestratorAgent` (também um LLM) consolida as quatro saídas: calcula o score ponderado, mapeia o breakdown, filtra os problemas de formatação `critical`/`moderate` e gera as recomendações.
4. O `QuestionGeneratorAgent` recebe as `missingKeywords`, o cargo e as responsabilidades da vaga e gera as perguntas.

**Otimização (`POST /ats/optimize`)**

1. O service busca a análise persistida (currículo e vaga) e descarta as respostas em branco.
2. O `OptimizerOrchestratorAgent` reexecuta o `OrchestratorAgent.analyze` sobre o mesmo currículo e vaga.
3. O `ResumeOptimizerAgent` reescreve o currículo usando as keywords ausentes, as recomendações, os problemas de formatação e as respostas — o prompt proíbe inventar informações que o candidato não confirmou.
4. O `OptimizerOrchestratorAgent` estima o novo score e o ganho e repassa as mudanças por seção.

**Conversão do PDF (`POST /resumes`)**: o `ResumeParserAgent` tem um segundo modo, `extractMarkdownFromPdf`, com um prompt embutido no código, que envia o PDF ao Gemini como `inlineData` (base64) e pede o conteúdo em Markdown sem reescrever nada.

### Frontend (`apps/web`)

O código é organizado em **core / features**:

```
app.routes.ts   ← Rotas: '' (landing) e 'analyzer'
app.config.ts   ← provideRouter, provideHttpClient e zone change detection
core            ← interceptors/index.interceptor.ts (arquivo vazio, ainda não registrado)
features
   landing      ← Landing page com mock estático de score e CTA para /analyzer
   analyzer     ← Componente pai (stepper), service HTTP, schemas e os 4 steps
      upload           ← Step 1: dropzone de PDF
      job-description  ← Step 2: textarea da vaga (mínimo de 50 caracteres)
      result           ← Step 3: score, breakdown, keywords, formatação e recomendações
      optimize         ← Step 4: perguntas, comparativo de score e download
environments    ← apiUrl da API
```

```mermaid
flowchart LR
    Browser[Navegador] --> Router["Angular Router"]
    Router -->|"/"| Landing[LandingComponent]
    Router -->|"/analyzer"| Analyzer["AnalyzerComponent<br/>dono do estado e do activeStep"]
    Analyzer --> Steps["Upload / JobDescription /<br/>Result / Optimize<br/>(@Input / @Output)"]
    Analyzer --> Service[AnalyzerService]
    Service -->|HttpClient| API[ATScan API]
```

- O `AnalyzerComponent` guarda `resumeId`, `resumeContent`, `jobDescription`, `analyzeResult` e `optimizeResult`; os steps recebem dados por `@Input()` e emitem eventos por `@Output()`.
- Erros da API aparecem num banner no topo do analisador, usando a `message` retornada pela API quando existe.
- O estado vive só em memória: recarregar a página reinicia o fluxo.
- A interface está em português.

---

## 🗄️ Modelo de Dados

Schema Prisma em [`apps/api/prisma/schema.prisma`](apps/api/prisma/schema.prisma), com migrations em [`apps/api/prisma/migrations`](apps/api/prisma/migrations) (`init`, `add_resume_content_to_analyses` e `start`). Não há tabela de usuários. Os IDs são UUID gerados pelo Prisma (`@default(uuid())`).

```mermaid
erDiagram
    resumes ||--o{ analyses : "é analisado em"
    analyses ||--o| optimizations : "gera no máximo uma"

    resumes {
        uuid id PK
        text content
        string fileName
        int fileSize
        string language
        timestamp createdAt
    }
    analyses {
        uuid id PK
        uuid resumeId FK
        text jobDescription
        text resumeContent
        int score
        json breakdown
        string_array matchedKeywords
        string_array missingKeywords
        string_array formatIssues
        json recommendations
        json questions
        timestamp createdAt
    }
    optimizations {
        uuid id PK
        uuid analysisId FK, UK
        int previousScore
        int newScore
        int gain
        text optimizedContent
        json changes
        json answers
        timestamp createdAt
    }
```

| Entidade (tabela) | Descrição |
| :--- | :--- |
| `Resume` (`resumes`) | Currículo convertido em Markdown, com nome e tamanho do PDF original; `language` é sempre gravado como `pt` |
| `Analysis` (`analyses`) | Resultado de uma análise: cópia do currículo e da vaga, score, breakdown (4 sub-scores), keywords, problemas de formatação, recomendações e perguntas. Índices em `resumeId`, `score` e `createdAt` |
| `Optimization` (`optimizations`) | Resultado da otimização de uma análise (`analysisId` único): scores anterior/novo, ganho, currículo otimizado, mudanças e as respostas usadas. Índices em `gain` e `createdAt` |

As relações usam `onDelete: Cascade`: apagar um currículo apaga suas análises, e apagar uma análise apaga sua otimização.

---

## 🗺️ Telas & Rotas

O frontend tem **duas rotas de URL**: `/` (landing) e `/analyzer`. As quatro etapas do analisador são controladas pelo estado do `AnalyzerComponent`, sem mudar a URL. Não há rota curinga para URLs desconhecidas.

```mermaid
flowchart TD
    Landing["/ — Landing"] -->|"CTA"| S1["/analyzer — Step 1: Upload<br/>POST /resumes"]
    S1 -->|"Próximo"| S2["Step 2: Descrição da vaga<br/>POST /ats/analyze"]
    S2 -->|"← Voltar"| S1
    S2 --> S3["Step 3: Resultado"]
    S3 -->|"Otimizar com IA"| S4["Step 4: Perguntas<br/>POST /ats/optimize"]
    S4 --> S5["Step 4: Resultado da otimização<br/>download .md"]
    S5 -->|"Analisar outra vaga"| S1
```

| Tela / visão | Como se chega | O que entrega |
| :--- | :--- | :--- |
| Landing | `/` | Hero, mock estático de score ATS e cards "Análise de Keywords", "Score por Seção" e "Otimização com IA" |
| Step 1 — Upload | CTA da landing → `/analyzer` | Dropzone ou seletor de arquivo; aceita só PDF; "Próximo" envia o arquivo e avança quando a conversão termina |
| Step 2 — Descrição da vaga | Após o upload | Textarea da vaga; "Analisar agora" exige 50+ caracteres |
| Step 3 — Resultado | Após a análise | Anel de score (vermelho < 50, amarelo 50–74, verde ≥ 75), 4 barras do breakdown, keywords encontradas/ausentes, problemas de formatação e recomendações |
| Step 4 — Otimização | "Otimizar meu currículo com IA" | Todas as perguntas de uma vez, cada uma com textarea; respostas em branco são ignoradas |
| Step 4 — Resultado da otimização | "Gerar currículo otimizado" | Score anterior vs novo, lista do que foi alterado, download do `.md` e "Analisar outra vaga" |

| | |
| :---: | :---: |
| ![Landing page com mock de score ATS](docs/screens/01-landing.png) <br/> *Landing page* | ![Step 1: upload do currículo em PDF](docs/screens/02-step1-upload.png) <br/> *Step 1: upload do currículo* |
| ![Step 2: descrição da vaga](docs/screens/03-step2-job-description.png) <br/> *Step 2: descrição da vaga* | ![Step 3: resultado com score e recomendações](docs/screens/04-step3-resultado.png) <br/> *Step 3: resultado* |
| ![Step 4: perguntas de otimização](docs/screens/05-step4-otimizacao.png) <br/> *Step 4: perguntas de otimização* | ![Step 4: comparativo de score](docs/screens/06-step4-resultado-otimizacao.png) <br/> *Step 4: resultado da otimização* |

---

## 📁 Estrutura do Projeto

```text
at-scan-monorepo/
├── apps/
│   ├── api/                              # Backend NestJS (workspace "api")
│   │   ├── prisma/
│   │   │   ├── schema.prisma             # Modelos Resume, Analysis e Optimization
│   │   │   └── migrations/               # init, add_resume_content_to_analyses, start
│   │   ├── src/
│   │   │   ├── main.ts                   # Bootstrap, CORS e porta (PORT ou 3000)
│   │   │   ├── app.module.ts             # Importa Prisma, Resumes, Ats e Agents
│   │   │   ├── prisma/                   # PrismaModule global e PrismaService
│   │   │   ├── resumes/                  # POST /resumes e GET /resumes/:id
│   │   │   ├── ats/                      # /ats/analyze e /ats/optimize; schemas de todos os agentes
│   │   │   └── agents/
│   │   │       ├── index.module.ts       # Provê e exporta os 8 agentes
│   │   │       ├── orchestrator/         # index.agent.ts, index.agent.spec.ts, index.prompt.md
│   │   │       ├── resume-parser/
│   │   │       ├── job-parser/
│   │   │       ├── semantic-match/
│   │   │       ├── format-checker/
│   │   │       ├── question-generator/
│   │   │       ├── optimizer-orchestrator/
│   │   │       └── resume-optimizer/
│   │   ├── .env.example                  # Modelo de variáveis da API
│   │   ├── eslint.config.mjs
│   │   └── package.json                  # Scripts e configuração do Jest
│   └── web/                              # Frontend Angular (workspace "web")
│       ├── public/favicon.ico
│       ├── src/
│       │   ├── main.ts                   # bootstrapApplication
│       │   ├── styles.css                # Tailwind
│       │   ├── environments/             # environment.ts e environment.prod.ts (apiUrl)
│       │   └── app/
│       │       ├── app.routes.ts         # '' e 'analyzer'
│       │       ├── app.config.ts
│       │       ├── core/interceptors/    # index.interceptor.ts (vazio)
│       │       └── features/
│       │           ├── landing/
│       │           └── analyzer/         # Stepper, service, schema e os steps upload, job-description, result, optimize
│       ├── .env.example                  # Referência (o Angular não lê .env neste projeto)
│       ├── angular.json
│       └── package.json
├── docs/screens/                         # Screenshots usados neste README
├── docker-compose.yml                    # Serviço db (PostgreSQL 16) no profile "db"
├── package.json                          # Workspaces, zod e scripts db:*
├── CLAUDE.MD                             # Guia de padrões para agentes de IA
└── LICENSE
```

---

## ⚡ Quick Start

### Pré-requisitos
- Node.js `^20.19`, `^22.12` ou `>=24` (exigência do Angular 21) e npm
- Docker e Docker Compose v2+ (para o PostgreSQL)
- Uma chave da API do Google Gemini (Google AI Studio)

### 1. Clone e instale
```bash
git clone https://github.com/matheusnascimentods/at-scan-monorepo.git
cd at-scan-monorepo
npm install
```

### 2. Configure a API
```bash
cp apps/api/.env.example apps/api/.env
```
Edite `apps/api/.env` e preencha `GEMINI_API_KEY`. O `DATABASE_URL` do exemplo já aponta para o Postgres do Docker em `localhost:5433`.

> A API não carrega o `.env` por conta própria (não usa `dotenv` nem `@nestjs/config`): ele é lido pelo Prisma. Para garantir que `GEMINI_API_KEY` e `GEMINI_MODEL` cheguem aos agentes, exporte as variáveis no shell antes de subir a API (passo 4).

### 3. Suba o banco e aplique as migrations
```bash
npm run db:up                  # docker compose --profile db up -d
docker compose --profile db ps # aguarde o status "healthy"
cd apps/api
npx prisma migrate deploy
npx prisma generate
cd ../..
```

### 4. Inicie os serviços
```bash
# Terminal 1 → http://localhost:3000
cd apps/api
set -a && source .env && set +a
npm run start:dev

# Terminal 2 → http://localhost:4200
npm run start --prefix apps/web
```

| Serviço | URL | Porta |
| :--- | :--- | :--- |
| **Frontend** (`ng serve`) | http://localhost:4200 | 4200 (padrão do Angular CLI) |
| **Backend API** | http://localhost:3000 | 3000 (`PORT`) |
| **PostgreSQL** (`db`) | localhost:5433 | 5433 → 5432 no container |

O frontend chama a API em `http://localhost:3000`, definido em [`apps/web/src/environments/environment.ts`](apps/web/src/environments/environment.ts).

### 5. Parando o banco
```bash
npm run db:down        # mantém os dados (volume atscan_pg_data)
npm run db:down:clean  # remove o volume e apaga todos os dados
```

---

## 💻 Comandos Úteis

### Raiz do monorepo
```bash
npm run db:up          # docker compose --profile db up -d
npm run db:up:logs     # docker compose --profile db up (logs no terminal)
npm run db:down        # docker compose --profile db down
npm run db:down:clean  # docker compose --profile db down -v
```

### API (`apps/api`)
```bash
npm run --prefix apps/api build            # nest build
npm run --prefix apps/api start            # nest start
npm run --prefix apps/api start:dev        # nest start --watch
npm run --prefix apps/api start:debug      # nest start --debug --watch
npm run --prefix apps/api start:prod       # node dist/main
npm run --prefix apps/api lint             # eslint --fix
npm run --prefix apps/api format           # prettier --write
npm run --prefix apps/api test             # jest
npm run --prefix apps/api test:watch       # jest --watch
npm run --prefix apps/api test:cov         # jest --coverage
npm run --prefix apps/api prisma:generate  # prisma generate
npm run --prefix apps/api prisma:migrate   # prisma migrate dev (cria/aplica migrations)
npm run --prefix apps/api prisma:studio    # prisma studio

# Dentro de apps/api
npx prisma migrate deploy                  # aplica as migrations existentes
```

### Web (`apps/web`)
```bash
npm run --prefix apps/web start   # ng serve (porta 4200)
npm run --prefix apps/web build   # ng build (dist/web)
npm run --prefix apps/web watch   # ng build --watch --configuration development
npm run --prefix apps/web test    # ng test (Karma + Jasmine)
```

---

## 🔌 API Reference

Base URL: `http://localhost:3000`. **Nenhuma rota exige autenticação** e não há prefixo global. Mensagens de erro são retornadas em português no formato padrão do NestJS (`{ statusCode, message }`).

### 📄 Resumes (`/resumes`)
| Método | Rota | Descrição | Respostas |
| :--- | :--- | :--- | :--- |
| `POST` | `/resumes` | Recebe o PDF (`multipart/form-data`, campo `file`, até 5 MB), converte para Markdown com IA e salva | 201; 400 sem arquivo ou não PDF; 502 se a IA falhar ou não extrair conteúdo |
| `GET` | `/resumes/:id` | Retorna um currículo salvo | 200; 400 se o ID não for UUID; 404 |

**Exemplo de Response (`POST /resumes` e `GET /resumes/:id`):**
```json
{
  "id": "uuid-v4",
  "content": "# Jane Doe\n\n## Experiência ...",
  "fileName": "curriculo.pdf",
  "createdAt": "2026-06-25T22:59:30.000Z"
}
```

---

### 📊 ATS (`/ats`)
| Método | Rota | Descrição | Respostas |
| :--- | :--- | :--- | :--- |
| `POST` | `/ats/analyze` | Analisa o currículo contra a vaga, persiste e retorna score, keywords, recomendações e perguntas | 201; 400 se o body for inválido; 502 se a IA falhar |
| `GET` | `/ats/analyze/:id` | Retorna uma análise persistida | 200; 404 |
| `POST` | `/ats/optimize` | Otimiza o currículo da análise com base nas respostas, persiste e retorna o resultado | 201; 400 se o body for inválido; 404 se a análise não existir; 502 se a IA falhar |
| `GET` | `/ats/optimize/:id` | Retorna uma otimização persistida | 200; 404 |

**Request (`POST /ats/analyze`):**
```json
{
  "resumeId": "uuid — id retornado pelo POST /resumes",
  "resumeContent": "string — Markdown do currículo (mínimo 100 caracteres)",
  "jobDescription": "string — descrição da vaga (mínimo 50 caracteres)"
}
```

**Response (`POST /ats/analyze` e `GET /ats/analyze/:id`):**
```json
{
  "id": "uuid — analysisId",
  "score": 73,
  "breakdown": {
    "keywordsScore": 80,
    "semanticScore": 68,
    "formatScore": 90,
    "sectionScore": 55
  },
  "matchedKeywords": ["Python", "Docker", "PostgreSQL"],
  "missingKeywords": ["Kubernetes", "Redis", "CI/CD"],
  "formatIssues": ["Evite tabelas — parsers ATS não leem corretamente"],
  "recommendations": [
    { "priority": "Alta", "text": "Adicione experiência com Kubernetes", "impact": "+6 pontos" }
  ],
  "questions": [
    { "tag": "Orquestração de Containers", "text": "Você tem experiência com Kubernetes?" }
  ]
}
```
`score` e os sub-scores são inteiros de 0 a 100; `priority` é `Alta`, `Média` ou `Baixa`.

**Request (`POST /ats/optimize`):**
```json
{
  "analysisId": "uuid — id retornado pelo POST /ats/analyze",
  "answers": [
    { "tag": "Orquestração de Containers", "question": "string", "answer": "string" }
  ]
}
```
O currículo e a vaga são lidos da análise persistida. Respostas com `answer` vazio são aceitas e descartadas antes de chegar aos agentes.

**Response (`POST /ats/optimize` e `GET /ats/optimize/:id`):**
```json
{
  "id": "uuid — optimizationId",
  "previousScore": 73,
  "newScore": 91,
  "gain": 18,
  "optimizedContent": "string — currículo otimizado em Markdown",
  "changes": [
    { "section": "Habilidades Técnicas", "description": "Adicionados: Redis, Kubernetes (familiar)" }
  ]
}
```

---

## 🗂️ Variáveis de Ambiente

### API (`apps/api/.env`)
Modelo em [`apps/api/.env.example`](apps/api/.env.example). Lidas via `process.env` (veja a observação sobre o carregamento do `.env` no [Quick Start](#-quick-start)).

| Variável | Descrição | Obrigatória | Default |
| :--- | :--- | :--- | :--- |
| `DATABASE_URL` | String de conexão PostgreSQL usada pelo Prisma | Sim | `.env.example`: `postgresql://postgres:postgres@localhost:5433/atscan` |
| `GEMINI_API_KEY` | Chave da API do Google Gemini, lida pelo `@google/adk` (que também aceita `GOOGLE_GENAI_API_KEY`, com precedência) | Sim | - |
| `GEMINI_MODEL` | Modelo usado por todos os agentes | Não | `gemini-2.0-flash` |
| `PORT` | Porta HTTP da API | Não | `3000` |
| `NODE_ENV` | Presente no `.env.example`, mas **não é lida pelo código** | Não | - |

### Web (`apps/web/src/environments`)
O Angular **não** lê `.env` neste projeto; o [`apps/web/.env.example`](apps/web/.env.example) é apenas referência. A URL da API fica em [`environment.ts`](apps/web/src/environments/environment.ts):

| Chave | Descrição | Valor |
| :--- | :--- | :--- |
| `apiUrl` | URL base da API usada pelo `AnalyzerService` | `http://localhost:3000` |
| `production` | Flag de ambiente | `false` |

> O `environment.prod.ts` existe, mas o `angular.json` não tem `fileReplacements`: todos os builds usam o `environment.ts`.

### Docker Compose
O serviço `db` usa credenciais fixas de desenvolvimento: usuário `postgres`, senha `postgres`, banco `atscan`, porta `5433` no host e volume `atscan_pg_data`.

---

## 🔒 Segurança & Boas Práticas
- **Sem dados de conta**: não há cadastro, senha ou sessão; nada além do currículo, da vaga e das respostas é armazenado.
- **Validação de entrada**: os bodies de `/ats/analyze` e `/ats/optimize` são validados com Zod (UUIDs, tamanhos mínimos e formato das respostas); `GET /resumes/:id` valida o formato do UUID.
- **Upload restrito**: só PDF, até 5 MB, mantido em memória e nunca gravado em disco.
- **Validação da IA**: a saída de cada agente e as respostas finais passam por schemas Zod antes de seguir no pipeline ou chegar ao cliente.
- **Falhas da IA isoladas**: erros dos agentes são registrados no log do servidor e devolvidos como 502 com mensagem genérica.
- **Auditabilidade da IA**: os prompts ficam em arquivos Markdown versionados ao lado de cada agente.
- **Segredos fora do Git**: `.env` está no `.gitignore`; só os `.env.example` são versionados.

### Limitações conhecidas
- **Sem CI**: não há workflow de integração contínua; os testes (Jest na API, Karma/Jasmine na Web) rodam só localmente, e os specs dos agentes cobrem apenas o básico.
- **Sem Docker para as aplicações**: o compose só tem o PostgreSQL; não há Dockerfiles para API ou Web.
- **`.env` não carregado explicitamente**: a API não usa `dotenv` nem `@nestjs/config`.
- **CORS aberto**: `origin: true` com `credentials: true` aceita qualquer origem.
- **Segredos padrão**: o Postgres do compose usa senha fixa de desenvolvimento.
- **Validação parcial**: `GET /ats/analyze/:id` e `GET /ats/optimize/:id` não validam o formato do UUID, e erros do Prisma (ID malformado, `resumeId` inexistente) voltam como 500. O PDF é validado por mimetype/extensão, não pelo conteúdo.
- **Uma otimização por análise**: `analysisId` é único em `optimizations`; otimizar a mesma análise de novo falha com 500.
- **Score estimado por LLM**: o score, o novo score e o ganho são calculados pelos próprios agentes seguindo os pesos do prompt, então podem variar entre execuções. Na otimização, a análise é refeita pelo agente, enquanto o `previousScore` salvo é o da análise original — `gain` pode não bater exatamente com `newScore - previousScore`.
- **Custo de IA**: uma análise faz 6 chamadas ao Gemini e uma otimização faz 8 (inclui a reanálise); não há retry, timeout nem cache.
- **Idioma fixo**: `language` é sempre gravado como `pt`.
- **Frontend**: o estado do analisador não sobrevive a um refresh; os schemas Zod do front só geram tipos (as respostas não são validadas em runtime); o interceptor HTTP está vazio; a URL da API é fixa em `environment.ts`.

---

## 📚 Documentação do Projeto

| Documento | Conteúdo |
| :--- | :--- |
| [`CLAUDE.MD`](CLAUDE.MD) | Guia de padrões para agentes de IA: convenção `index.*`, regras dos agentes, schemas, banco e casos de teste. Alguns itens descrevem o plano do projeto e ainda não existem no código (ex.: testes E2E com Cypress) |
| [`docs/screens`](docs/screens) | Screenshots das telas usados neste README |
| [`apps/api/src/agents`](apps/api/src/agents) | Prompts de cada agente (`*/index.prompt.md`) |

---

## 🧑‍💻 Contexto

Projeto pessoal desenvolvido por [@matheusnascimentods](https://github.com/matheusnascimentods) para explorar arquiteturas multi-agente com o Google ADK num problema real de quem procura emprego.

---

## 🤝 Contribuindo
1. Faça um Fork do projeto
2. Crie uma branch para a sua funcionalidade (`git checkout -b feature/nome-da-feature`)
3. Siga os padrões do [`CLAUDE.MD`](CLAUDE.MD) (arquivos `index.*`, um `index.schema.ts` por módulo, teste `.spec.ts` ao lado)
4. Rode `npm run --prefix apps/api test`, `npm run --prefix apps/api build` e `npm run --prefix apps/web build`
5. Faça o commit das suas alterações (`git commit -m 'feat: implementa nova funcionalidade'`)
6. Envie para o repositório remoto (`git push origin feature/nome-da-feature`)
7. Abra um **Pull Request** detalhando as alterações

---

## 📄 Licença

Distribuído sob a licença MIT. Veja [`LICENSE`](LICENSE) para mais detalhes.

---
Feito com ❤️ por [@matheusnascimentods](https://github.com/matheusnascimentods)
