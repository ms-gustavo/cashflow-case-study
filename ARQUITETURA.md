# Arquitetura — CashFlow Backend

## Visão geral

O sistema é uma aplicação NestJS organizada em módulos independentes, com duas portas de entrada (API REST e bot do Telegram) que compartilham a mesma camada de serviços. A persistência usa PostgreSQL via Prisma, e a auditoria é processada de forma assíncrona por uma fila Redis/BullMQ.

---

## Diagrama

```
                  ┌──────────────────────────────┐
                  │          Clientes             │
                  │   (Frontend, Mobile, cURL)    │
                  └──────────────┬───────────────┘
                                 │ HTTP
                  ┌──────────────▼───────────────┐
                  │       Express + CORS          │
                  │     ValidationPipe global     │
                  │   (whitelist, transform)       │
                  └──────────────┬───────────────┘
                                 │
          ┌──────────────────────┼────────────────────────┐
          │                      │                        │
          │         ┌────────────▼────────────┐           │
          │         │   CLS Middleware        │           │
          │         │  (correlation ID por    │           │
          │         │   request, gera UUID    │           │
          │         │   ou aceita header)     │           │
          │         └────────────┬────────────┘           │
          │                      │                        │
          │         ┌────────────▼────────────┐           │
          │         │   Pino Logger           │           │
          │         │  (log estruturado com   │           │
          │         │   correlationId)        │           │
          │         └────────────┬────────────┘           │
          │                      │                        │
┌─────────▼──────────┐          │          ┌──────────────▼─────────────┐
│    Auth Module      │          │          │      Telegram Module       │
│                     │          │          │                            │
│  POST /auth/login   │          │          │  Grammy Bot ◄── Telegram   │
│  POST /auth/register│          │          │       │                    │
│                     │          │          │  ┌────▼─────────────────┐  │
│  JwtStrategy        │          │          │  │ TelegramCommands     │  │
│  JwtAuthGuard       │          │          │  │  /novo /saldo /hoje  │  │
│  Passport           │          │          │  │  /mes /ultimos       │  │
└─────────┬──────────┘          │          │  └────┬────────────────┘  │
          │ JWT                  │          │       │                    │
          │ token                │          │  ┌────▼─────────────────┐  │
          │                      │          │  │ Handlers             │  │
          │                      │          │  │  Text, Callback,     │  │
          │                      │          │  │  Contact             │  │
          │                      │          │  └────┬────────────────┘  │
          │                      │          │       │                    │
          │                      │          │  ┌────▼─────────────────┐  │
          │                      │          │  │ WizardService        │  │
          │                      │          │  │  (fluxo guiado       │  │
          │                      │          │  │   passo a passo)     │  │
          │                      │          │  └────┬────────────────┘  │
          │                      │          └───────┼──────────────────┘
          │                      │                  │
  ┌───────▼──────────────────────▼──────────────────▼────────┐
  │                   Módulos HTTP (protegidos por JWT)       │
  │                                                          │
  │  ┌──────────────┐ ┌────────────────┐ ┌───────────────┐   │
  │  │ Categories   │ │ Transactions   │ │ Reports       │   │
  │  │  Controller  │ │  Controller    │ │  Controller   │   │
  │  │  Service     │ │  Service       │ │  Service      │   │
  │  │  DTOs        │ │  DTOs          │ │  DTOs         │   │
  │  └──────┬───────┘ └───────┬────────┘ └──────┬────────┘   │
  │         │                 │                  │            │
  │         │    ┌────────────▼─────────┐        │            │
  │         │    │   Users Module       │        │            │
  │         │    │   Controller+Service │        │            │
  │         │    └────────────┬─────────┘        │            │
  └─────────┼────────────────┼──────────────────┼────────────┘
            │                │                  │
            │                │                  │
  ┌─────────▼────────────────▼──────────────────▼────────────┐
  │                Audit Interceptor (global)                │
  │  Ignora GET/HEAD/OPTIONS                                 │
  │  Captura: method, path, IP, user-agent, userId           │
  │  Armazena contexto no CLS para o AuditService ler        │
  └──────────────────────────┬───────────────────────────────┘
                             │
               ┌─────────────┴─────────────┐
               │                           │
  ┌────────────▼──────────┐   ┌────────────▼──────────┐
  │    Prisma ORM         │   │    AuditService       │
  │    (Database Module)  │   │    enfileira job       │
  │                       │   │    no BullMQ           │
  │  PrismaService        │   └────────────┬──────────┘
  │  onModuleInit()       │                │
  │  $connect()           │   ┌────────────▼──────────┐
  │                       │   │    Redis (BullMQ)     │
  └────────────┬──────────┘   │    Fila: audit-queue  │
               │              └────────────┬──────────┘
               │                           │
               │              ┌────────────▼──────────┐
               │              │  AuditProcessor       │
               │              │  consome a fila e     │
               │              │  persiste AuditLog    │
               │              │  via Prisma           │
               │              └────────────┬──────────┘
               │                           │
  ┌────────────▼───────────────────────────▼──────────┐
  │                  PostgreSQL                       │
  │                                                   │
  │  User ──< Category ──< Transaction                │
  │                                                   │
  │  AuditLog (correlationId, before/after, diff)     │
  └───────────────────────────────────────────────────┘
```

---

## Decisões arquiteturais e trade-offs

### 1. Auditoria assíncrona via fila (BullMQ/Redis)

**Decisão**: toda operação de escrita gera um job na fila Redis em vez de gravar o AuditLog na mesma transação do banco.

**Por quê**: a auditoria não deve adicionar latência às operações do usuário. Separar o processamento garante que o response volta rápido, mesmo que o Redis ou o processor estejam sob carga.

**Trade-off**: em caso de falha do Redis ou do processor, existe uma janela onde o evento pode se perder antes de ser persistido. A constraint de unicidade `[correlationId, entity, entityId, action]` mitiga duplicações no reprocessamento, mas não resolve perda total do job.

### 2. Correlation ID via CLS (Context Local Storage)

**Decisão**: o correlation ID é gerado (ou propagado do header `x-correlation-id`) no middleware CLS e fica disponível automaticamente pra todos os serviços, logs e jobs de auditoria daquela requisição.

**Por quê**: evita passar o correlation ID manualmente por parâmetro em cada chamada de serviço. Todo log do Pino e todo registro de auditoria já saem com o ID correto, sem boilerplate.

**Trade-off**: o CLS depende do contexto assíncrono do Node.js (AsyncLocalStorage). Se algum código escapar desse contexto (ex: callbacks mal gerenciados ou bibliotecas que criam seu próprio event loop), o correlation ID pode se perder. Na prática, com NestJS e as bibliotecas usadas, isso não tem sido problema.

### 3. Interceptor global registrado no módulo de auditoria

**Decisão**: o `AuditInterceptor` é registrado como `APP_INTERCEPTOR` global dentro do `AuditModule`, em vez de ser aplicado manualmente em cada controller.

**Por quê**: garante que toda rota de escrita é auditada automaticamente. Não depende do desenvolvedor lembrar de adicionar o interceptor ao criar um novo controller.

**Trade-off**: o interceptor roda em todas as rotas (mas ignora GET/HEAD/OPTIONS por conta própria). Se algum endpoint de escrita não devesse ser auditado, seria preciso criar um mecanismo de exclusão.

### 4. Bot e API compartilhando services

**Decisão**: o módulo do Telegram importa os mesmos módulos de negócio (Transactions, Categories, Reports, Users) e usa os mesmos services que os controllers HTTP.

**Por quê**: evita duplicação de lógica. Uma correção ou feature nova nos services já vale pras duas interfaces.

**Trade-off**: o bot do Telegram não passa pelo pipeline HTTP (interceptors, guards). A auditoria de operações feitas pelo bot precisa ser tratada separadamente, já que o `AuditInterceptor` só atua no contexto HTTP.

### 5. Prisma como ORM único com migrations versionadas

**Decisão**: usar Prisma como única interface com o banco, com schema declarativo e migrations commitadas.

**Por quê**: o schema funciona como documentação viva do banco. As migrations garantem reprodutibilidade — qualquer ambiente novo aplica o histórico completo e chega no mesmo estado. O Prisma Client gera tipos TypeScript a partir do schema, eliminando descompasso entre código e banco.

**Trade-off**: queries complexas ou muito otimizadas podem ser mais difíceis de expressar no Prisma comparado a SQL puro. Para os cenários atuais (CRUD + agregações de relatórios), não tem sido limitante.
