# CashFlow — Case Study

Aplicação para controle de fluxo de caixa, com API REST, bot do Telegram integrado e aplicação web (CashFlow Web). O sistema permite registrar receitas e despesas, categorizá-las, gerar relatórios financeiros e acompanhar tudo pelo celular, direto pelo chat do Telegram. Suporta múltiplos usuários com autenticação, auditoria completa de operações e rastreamento de requisições ponta a ponta.

---

## O problema

Controlar finanças pessoais ou de um pequeno negócio parece simples, mas na prática as pessoas acabam esquecendo de anotar gastos, abrindo planilhas que ficam desatualizadas, ou usando apps que exigem esforço demais pra registrar cada centavo.

O que eu queria era algo que funcionasse de duas formas: uma API robusta pra quem quiser um frontend dedicado, e um bot no Telegram pra quem quer registrar um gasto em segundos, seguindo um fluxo guiado direto pelo chat.

Além disso, precisava de isolamento de dados entre usuários, auditoria de tudo que acontece no sistema e uma arquitetura que fosse fácil de manter e evoluir.

---

## O que foi construído

- **API REST completa** com autenticação JWT, gestão de transações, categorias e relatórios financeiros
- **Bot do Telegram** com fluxo wizard — guia o usuário passo a passo pra registrar transações de forma rápida e estruturada
- **Aplicação web (CashFlow Web)**: dashboard com gráficos, CRUD de transações (com filtros/paginação/parcelamentos), gestão de categorias, relatórios com gráficos interativos, perfil e responsividade.
- **Relatórios**: saldo atual, resumo diário/mensal, consulta por intervalo de datas e resumo geral
- **Auditoria assíncrona**: toda operação de escrita (criar, atualizar, deletar) é registrada com estado antes/depois, campos alterados, IP, user-agent e correlation ID — processada via fila pra não impactar a latência
- **Multi-tenant por usuário**: cada usuário só vê e manipula seus próprios dados
- **Categorias personalizadas** com categorias padrão pré-configuradas
- **Desfazer lançamentos** tanto pela API quanto pelo bot

---

## Stack

| Camada                 | Tecnologia                          |
| ---------------------- | ----------------------------------- |
| Runtime / Framework    | Node.js + NestJS (TypeScript)       |
| Banco de dados         | PostgreSQL                          |
| ORM                    | Prisma                              |
| Fila / Cache           | Redis + BullMQ                      |
| Autenticação           | Passport + JWT                      |
| Bot Telegram           | Grammy                              |
| Logs / Observabilidade | Pino + nestjs-cls (correlation ID)  |
| Validação              | class-validator + class-transformer |
| Infra                  | Docker + Docker Compose             |
| Testes                 | Jest + Supertest                    |

---

### Frontend (Web)

| Camada              | Tecnologia                              |
| ------------------- | --------------------------------------- |
| Framework           | Next.js 16 (App Router)                 |
| UI                  | React 19                                |
| Linguagem           | TypeScript 5                            |
| Componentes         | Shadcn/UI + Radix UI                    |
| Estilização         | Tailwind CSS 4                          |
| Estado do Servidor  | TanStack Query v5                       |
| Estado Global       | Zustand                                 |
| Formulários         | React Hook Form + Zod                   |
| HTTP Client         | Axios (interceptors para auth)          |
| Gráficos            | Recharts                                |
| Ícones              | Lucide React                            |
| Testes Unitários    | Vitest + Testing Library                |
| Testes E2E          | Playwright                              |
| Mock de API         | MSW (Mock Service Worker)               |
| Qualidade de Código | ESLint + Prettier + Husky + lint-staged |

---

## Arquitetura

Para mais detalhes sobre a arquitetura, consulte [ARQUITETURA.md](ARQUITETURA.md).

```
┌───────────────────────────────────────────┐
│      Clientes (HTTP / Telegram / Web)     │
└─────────────────────┬─────────────────────┘
                      │
        ┌─────────────┴─────────────┐
        │                           │
  ┌─────▼──────┐            ┌──────▼──────┐
  │  API REST  │            │  Bot Telegram│
  │  (JWT)     │            │  (Grammy)    │
  └─────┬──────┘            └──────┬──────┘
        │                          │
        └─────────────┬────────────┘
                      │
           ┌──────────▼──────────┐
           │    Services Layer   │
           │  (lógica de negócio)│
           └──────────┬──────────┘
                      │
        ┌─────────────┼─────────────┐
        │             │             │
  ┌─────▼─────┐ ┌────▼─────┐ ┌────▼──────┐
  │ Prisma ORM│ │ BullMQ   │ │ Audit     │
  │           │ │ (Redis)  │ │ Processor │
  └─────┬─────┘ └──────────┘ └───────────┘
        │
  ┌─────▼─────┐
  │ PostgreSQL│
  └───────────┘
```
**CashFlow Web (Next.js)** consome a API via HTTP e compartilha o mesmo fluxo de autenticação (JWT). O frontend também envia `x-correlation-id` para rastreamento ponta a ponta.

### Web (CashFlow Web)

### Autenticação

O token JWT é armazenado em cookie (`cashflow_access_token`) e localStorage. O middleware do Next.js redireciona:

- Usuários **não autenticados** em rotas protegidas → `/login`
- Usuários **autenticados** em rotas públicas → `/dashboard`

### Camada de Dados

- **Axios** com interceptors para injetar `Authorization` header e `x-correlation-id`
- **TanStack Query** para cache, revalidação automática e sincronização do estado do servidor (`staleTime: 30s`, `refetchOnWindowFocus`)
- **Zustand** para estado global leve (auth, preferências de UI)

### Formulários e Validação

- **React Hook Form** para performance e controle de formulários
- **Zod** com schemas espelhando as validações `class-validator` do backend

---


O sistema é organizado em módulos NestJS independentes: Auth, Users, Transactions, Categories, Reports, Telegram, Audit e Database. Cada módulo encapsula controller, service e DTOs.

---

## Fluxos importantes

### 1. Lançamento via bot do Telegram

1. Usuário envia o comando `/novo` no chat
2. O bot inicia o wizard guiado e pergunta o tipo (entrada ou saída)
3. Em seguida pede o valor, a data, a categoria e uma descrição (opcional)
4. O usuário revisa e confirma o lançamento
5. A transação é persistida no banco vinculada ao usuário
6. Um job de auditoria é enfileirado no Redis com os detalhes da operação
7. O bot confirma o registro pro usuário

### 2. Auditoria assíncrona

1. Um interceptor HTTP captura a requisição e a resposta
2. Compara o estado da entidade antes e depois da operação
3. Monta um payload com: ação, entidade, campos alterados, contexto HTTP (IP, user-agent, path) e correlation ID
4. Enfileira o payload no BullMQ (Redis)
5. Um processor consome a fila e persiste o registro de auditoria no PostgreSQL

Esse fluxo assíncrono garante que a auditoria não adicione latência às operações do usuário.

### 3. Relatórios financeiros

O módulo de relatórios agrega dados de transações e retorna:

- **Saldo**: soma de todas as entradas menos todas as saídas do usuário
- **Diário**: entradas e saídas de uma data específica
- **Mensal**: resumo completo de um mês (totais, contagem de transações)
- **Por intervalo**: análise entre duas datas informadas
- **Resumo geral**: estatísticas configuráveis

Todos os relatórios são filtrados pelo usuário autenticado — não há como acessar dados de outro usuário.

---
### 4. Uso via aplicação web (CashFlow Web)

1. Usuário faz login/registro e recebe o JWT
2. Middleware do Next.js protege rotas e redireciona conforme autenticação
3. A aplicação consulta dashboard/transações/relatórios consumindo a API
4. Operações de escrita (CRUD) seguem o mesmo fluxo do backend e disparam auditoria assíncrona

## API

### Endpoints

| Grupo        | Método | Rota                | Descrição                |
| ------------ | ------ | ------------------- | ------------------------ |
| Health       | GET    | `/`                 | Status da aplicação      |
| Auth         | POST   | `/auth/register`    | Registro de usuário      |
| Auth         | POST   | `/auth/login`       | Login (retorna JWT)      |
| Users        | GET    | `/users/me`         | Perfil do usuário logado |
| Users        | PATCH  | `/users/me`         | Atualizar perfil         |
| Categories   | GET    | `/categories`       | Listar categorias        |
| Categories   | POST   | `/categories`       | Criar categoria          |
| Categories   | GET    | `/categories/:id`   | Buscar categoria         |
| Categories   | PATCH  | `/categories/:id`   | Atualizar categoria      |
| Categories   | DELETE | `/categories/:id`   | Remover categoria        |
| Transactions | GET    | `/transactions`     | Listar transações        |
| Transactions | POST   | `/transactions`     | Criar transação          |
| Transactions | GET    | `/transactions/:id` | Buscar transação         |
| Transactions | PATCH  | `/transactions/:id` | Atualizar transação      |
| Transactions | DELETE | `/transactions/:id` | Remover transação        |
| Reports      | GET    | `/reports/balance`  | Saldo atual              |
| Reports      | GET    | `/reports/daily`    | Relatório diário         |
| Reports      | GET    | `/reports/monthly`  | Relatório mensal         |
| Reports      | GET    | `/reports/range`    | Por intervalo            |
| Reports      | GET    | `/reports/summary`  | Resumo geral             |

Todas as rotas (exceto Auth) exigem header `Authorization: Bearer <token>`.

### Exemplo: Criar transação

**Request**

```http
POST /transactions
Authorization: Bearer eyJhbG...exemplo
Content-Type: application/json

{
  "type": "OUT",
  "amount": 89.90,
  "description": "Supermercado semanal",
  "date": "2026-02-10T10:00:00Z",
  "categoryId": "a1b2c3d4-e5f6-7890-abcd-ef1234567890"
}
```

**Response (201)**

```json
{
  "id": "f9e8d7c6-b5a4-3210-fedc-ba0987654321",
  "type": "OUT",
  "amount": "89.90",
  "description": "Supermercado semanal",
  "date": "2026-02-10T10:00:00.000Z",
  "categoryId": "a1b2c3d4-e5f6-7890-abcd-ef1234567890",
  "userId": "11223344-5566-7788-9900-aabbccddeeff",
  "createdAt": "2026-02-10T10:01:00.000Z",
  "updatedAt": "2026-02-10T10:01:00.000Z"
}
```

### Exemplo: Consultar saldo

**Request**

```http
GET /reports/balance
Authorization: Bearer eyJhbG...exemplo
```

**Response (200)**

```json
{
  "balance": 4521.33
}
```

---

## Modelagem de dados

O banco tem quatro entidades principais:

- **User**: representa um usuário do sistema. Possui nome, email (opcional), senha (hash), telefone e pode estar vinculado a um chat do Telegram. Cada usuário tem suas próprias categorias e transações.

- **Category**: categoria financeira (ex: Alimentação, Transporte, Salário). Pertence a um usuário e tem nome único por usuário. Pode ter transações associadas.

- **Transaction**: o coração do sistema. Representa uma entrada ou saída de dinheiro, com valor (decimal 12,2), data, descrição e categoria. Sempre pertence a um usuário e a uma categoria.

- **AuditLog**: registro de auditoria de cada operação de escrita. Guarda o correlation ID da requisição, a entidade afetada, a ação (criação/atualização/remoção), o estado antes e depois, os campos que mudaram, e contexto HTTP. Possui constraint de unicidade por correlation + entidade + ação para garantir idempotência.

As entidades usam UUID como chave primária e têm índices estratégicos nos campos mais consultados (data, userId, categoryId, correlationId).

---

## Boas práticas adotadas

**Modularização** — O projeto segue a arquitetura modular do NestJS, com cada domínio (auth, users, transactions, categories, reports, telegram, audit) isolado em seu próprio módulo. Isso facilita manutenção, testes e evolução independente de cada parte.

**Observabilidade** — Toda requisição HTTP recebe um correlation ID (via nestjs-cls) que acompanha o log estruturado (Pino) e o registro de auditoria. Isso permite rastrear uma operação de ponta a ponta, do request ao job na fila.

**Fila assíncrona** — A auditoria é processada fora da thread principal via BullMQ/Redis. O interceptor captura o diff, enfileira e a resposta volta pro cliente sem esperar a escrita no banco de auditoria.

**Idempotência na auditoria** — A constraint unique `[correlationId, entity, entityId, action]` no AuditLog evita registros duplicados caso um job seja reprocessado.

**Migrations versionadas** — O schema do banco evolui por migrations do Prisma, com histórico commitado no repositório. Isso garante que qualquer ambiente pode ser reproduzido do zero.

**Validação rigorosa** — Todas as entradas da API passam por DTOs com `class-validator`, com whitelist ativada (propriedades não esperadas são rejeitadas).

**Separação de interfaces** — A lógica de negócio é compartilhada entre a API REST e o bot do Telegram via services, sem duplicação. Ambas as interfaces são apenas "portas de entrada" para os mesmos services.

---

## Demo

📹 **Vídeo demonstrativo**

https://github.com/user-attachments/assets/01faa57e-b623-4566-b3d9-8164458fd670

---

## Próximos passos

- Exportação de relatórios em PDF/CSV
- Transações recorrentes (agendamento automático)
- Notificações no Telegram (alertas de saldo baixo, resumo semanal)
- Testes de integração mais abrangentes com banco de testes dedicado
- Documentação via Swagger/OpenAPI
