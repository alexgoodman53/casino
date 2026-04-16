# Пример Слоистой Архитектуры Интеграций

Этот документ показывает **упрощенный, но уже похожий на реальный код** пример архитектуры из `README`.

Цель:

- показать слои;
- показать минимальные классы и интерфейсы;
- показать таблицы;
- показать, как идут вызовы во время игры;
- не усложнять модель лишними деталями.

---

## 1. Слои

Используем 5 слоев:

1. `Provider Adapter`
2. `Provider Policy`
3. `Protocol Engine`
4. `Integration Platform`
5. `Casino Core`

Коротко:

- `Provider Adapter` знает transport, подпись, формат запроса и ответа.
- `Provider Policy` знает provider-specific правила.
- `Protocol Engine` знает общую логику семейства протоколов.
- `Integration Platform` хранит внешнее состояние, inbox, pending, links.
- `Casino Core` двигает деньги и ведет ledger.

---

## 2. Общая схема

```mermaid
flowchart TD
    A["Provider HTTP Request"] --> B["Provider Adapter"]
    B --> C["Provider Policy"]
    C --> D["Protocol Engine"]
    D --> E["Integration Repositories"]
    D --> F["Casino Core Gateway"]
    F --> G["Wallets / Ledger / Rounds"]
    D --> H["Provider Response"]
    H --> B
```

---

## 3. Пример классов

### 3.1. Provider Adapter

```ts
export interface ProviderAdapter {
  providerCode: string;

  verify(request: HttpRequest): void;
  parse(request: HttpRequest): ProviderCommand;
  formatSuccess(result: EngineResult): HttpResponse;
  formatError(error: EngineError): HttpResponse;
}

export class SlotegratorAdapter implements ProviderAdapter {
  providerCode = "slotegrator";

  verify(request: HttpRequest): void {
    // check X-Merchant-Id, X-Timestamp, X-Nonce, X-Sign
  }

  parse(request: HttpRequest): ProviderCommand {
    // action=balance|bet|win|refund|rollback -> internal command
    return {
      provider: "slotegrator",
      type: "bet",
      externalTransactionId: request.body.transaction_id,
      externalRoundId: request.body.round_id,
      externalSessionId: request.body.session_id,
      amount: request.body.amount,
      currency: request.body.currency,
      raw: request.body,
    };
  }

  formatSuccess(result: EngineResult): HttpResponse {
    return { status: 200, body: result.responseBody };
  }

  formatError(error: EngineError): HttpResponse {
    return { status: 200, body: error.responseBody };
  }
}

export class TreasurePruneAdapter implements ProviderAdapter {
  providerCode = "treasure-prune";

  verify(request: HttpRequest): void {
    // verify signature for incoming JSON request
  }

  parse(request: HttpRequest): ProviderCommand {
    return {
      provider: "treasure-prune",
      type: request.body.cmd,
      externalTransactionId: request.body.transaction_id,
      externalRoundId: request.body.round_id,
      externalSessionId: request.body.session_id,
      amount: request.body.amount,
      currency: request.body.currency,
      raw: request.body,
    };
  }

  formatSuccess(result: EngineResult): HttpResponse {
    return { status: 200, body: result.responseBody };
  }

  formatError(error: EngineError): HttpResponse {
    return { status: 200, body: error.responseBody };
  }
}
```

### 3.2. Provider Policy

```ts
export interface ProviderPolicy {
  providerCode: string;

  resolveEngine(command: ProviderCommand): EngineKind;
  buildContext(command: ProviderCommand): ProviderContext;
}

export class SlotegratorPolicy implements ProviderPolicy {
  providerCode = "slotegrator";

  resolveEngine(command: ProviderCommand): EngineKind {
    return "seamless-wallet";
  }

  buildContext(command: ProviderCommand): ProviderContext {
    return {
      dedupeKey: `${command.provider}:${command.type}:${command.externalTransactionId}`,
      roundMatchStrategy: "round_id-first",
      canArriveBeforeSource: command.type === "refund",
      callbackTimeoutSeconds: 3,
    };
  }
}

export class TreasurePrunePolicy implements ProviderPolicy {
  providerCode = "treasure-prune";

  resolveEngine(command: ProviderCommand): EngineKind {
    return "stateful-transaction";
  }

  buildContext(command: ProviderCommand): ProviderContext {
    return {
      dedupeKey: `${command.provider}:${command.type}:${command.externalTransactionId}`,
      recoveryMode:
        command.type === "withdraw.bet"
          ? "cancel-on-undefined"
          : command.type === "deposit.win"
            ? "complete-on-fail"
            : "none",
      callbackTimeoutSeconds: 3,
    };
  }
}
```

### 3.3. Protocol Engine

```ts
export interface ProtocolEngine {
  kind: EngineKind;
  handle(command: ProviderCommand, context: ProviderContext): Promise<EngineResult>;
}

export class SeamlessWalletEngine implements ProtocolEngine {
  kind: EngineKind = "seamless-wallet";

  constructor(
    private inboxRepo: IntegrationInboxRepository,
    private txRepo: ExternalTransactionRepository,
    private roundRepo: ExternalRoundRepository,
    private pendingRepo: PendingOperationRepository,
    private core: CasinoCoreGateway,
  ) {}

  async handle(command: ProviderCommand, context: ProviderContext): Promise<EngineResult> {
    const duplicate = await this.inboxRepo.findProcessed(context.dedupeKey);
    if (duplicate) return duplicate.savedResult;

    if (command.type === "bet") {
      const round = await this.roundRepo.openOrAttach(command);
      const money = await this.core.applyBetDebit({
        playerId: command.playerId!,
        amount: command.amount!,
        currency: command.currency!,
        roundId: round.internalRoundId,
      });

      const result = { ok: true, balance: money.balance };
      await this.txRepo.saveApplied(command, round, result);
      await this.inboxRepo.markProcessed(context.dedupeKey, result);
      return result;
    }

    if (command.type === "refund") {
      const sourceBet = await this.txRepo.findSourceBet(command);
      if (!sourceBet) {
        const result = { ok: true, accepted: true };
        await this.pendingRepo.save(command);
        await this.inboxRepo.markProcessed(context.dedupeKey, result);
        return result;
      }

      const money = await this.core.applyRefund({
        playerId: sourceBet.playerId,
        amount: sourceBet.amount,
        currency: sourceBet.currency,
        roundId: sourceBet.internalRoundId,
      });

      const result = { ok: true, balance: money.balance };
      await this.txRepo.saveRefund(command, sourceBet, result);
      await this.inboxRepo.markProcessed(context.dedupeKey, result);
      return result;
    }

    throw new Error(`Unsupported seamless command: ${command.type}`);
  }
}

export class StatefulTransactionEngine implements ProtocolEngine {
  kind: EngineKind = "stateful-transaction";

  constructor(
    private inboxRepo: IntegrationInboxRepository,
    private txRepo: ExternalTransactionRepository,
    private roundRepo: ExternalRoundRepository,
    private core: CasinoCoreGateway,
  ) {}

  async handle(command: ProviderCommand, context: ProviderContext): Promise<EngineResult> {
    const duplicate = await this.inboxRepo.findProcessed(context.dedupeKey);
    if (duplicate) return duplicate.savedResult;

    if (command.type === "withdraw.bet") {
      const round = await this.roundRepo.openOrAttach(command);
      const money = await this.core.applyBetDebit({
        playerId: command.playerId!,
        amount: command.amount!,
        currency: command.currency!,
        roundId: round.internalRoundId,
      });

      const result = { ok: true, balance: money.balance };
      await this.txRepo.saveApplied(command, round, result);
      await this.inboxRepo.markProcessed(context.dedupeKey, result);
      return result;
    }

    if (command.type === "trx.complete") {
      const source = await this.txRepo.findSourceTransaction(command);
      const result = { ok: true, completed: true, sourceId: source?.externalTransactionId };
      await this.txRepo.markCompleted(command, source, result);
      await this.inboxRepo.markProcessed(context.dedupeKey, result);
      return result;
    }

    throw new Error(`Unsupported stateful command: ${command.type}`);
  }
}
```

### 3.4. Integration Platform

```ts
export interface IntegrationInboxRepository {
  findProcessed(dedupeKey: string): Promise<{ savedResult: EngineResult } | null>;
  markProcessed(dedupeKey: string, result: EngineResult): Promise<void>;
}

export interface ExternalTransactionRepository {
  saveApplied(
    command: ProviderCommand,
    round: ExternalRoundRecord,
    result: EngineResult,
  ): Promise<void>;

  saveRefund(
    command: ProviderCommand,
    sourceBet: ExternalTransactionRecord,
    result: EngineResult,
  ): Promise<void>;

  findSourceBet(command: ProviderCommand): Promise<ExternalTransactionRecord | null>;
  findSourceTransaction(command: ProviderCommand): Promise<ExternalTransactionRecord | null>;
  markCompleted(
    command: ProviderCommand,
    source: ExternalTransactionRecord | null,
    result: EngineResult,
  ): Promise<void>;
}

export interface ExternalRoundRepository {
  openOrAttach(command: ProviderCommand): Promise<ExternalRoundRecord>;
}

export interface PendingOperationRepository {
  save(command: ProviderCommand): Promise<void>;
}
```

### 3.5. Casino Core

```ts
export interface CasinoCoreGateway {
  applyBetDebit(input: ApplyBetDebitInput): Promise<MoneyResult>;
  applyWinCredit(input: ApplyWinCreditInput): Promise<MoneyResult>;
  applyRefund(input: ApplyRefundInput): Promise<MoneyResult>;
}

export class CasinoWalletService implements CasinoCoreGateway {
  constructor(
    private walletRepo: WalletRepository,
    private ledgerRepo: LedgerRepository,
    private roundRepo: CoreRoundRepository,
  ) {}

  async applyBetDebit(input: ApplyBetDebitInput): Promise<MoneyResult> {
    await this.ledgerRepo.insertDebit(input);
    const balance = await this.walletRepo.decrease(input.playerId, input.amount);
    await this.roundRepo.attachDebit(input.roundId, input.amount);
    return { balance };
  }

  async applyWinCredit(input: ApplyWinCreditInput): Promise<MoneyResult> {
    await this.ledgerRepo.insertCredit(input);
    const balance = await this.walletRepo.increase(input.playerId, input.amount);
    await this.roundRepo.attachCredit(input.roundId, input.amount);
    return { balance };
  }

  async applyRefund(input: ApplyRefundInput): Promise<MoneyResult> {
    await this.ledgerRepo.insertRefund(input);
    const balance = await this.walletRepo.increase(input.playerId, input.amount);
    await this.roundRepo.attachRefund(input.roundId, input.amount);
    return { balance };
  }
}
```

### 3.6. Application Facade

```ts
export class ProviderRequestHandler {
  constructor(
    private adapters: Map<string, ProviderAdapter>,
    private policies: Map<string, ProviderPolicy>,
    private engines: Map<EngineKind, ProtocolEngine>,
  ) {}

  async handle(providerCode: string, request: HttpRequest): Promise<HttpResponse> {
    const adapter = this.adapters.get(providerCode)!;
    const policy = this.policies.get(providerCode)!;

    adapter.verify(request);
    const command = adapter.parse(request);
    const context = policy.buildContext(command);
    const engine = this.engines.get(policy.resolveEngine(command))!;

    const result = await engine.handle(command, context);
    return adapter.formatSuccess(result);
  }
}
```

---

## 4. Минимальная диаграмма классов

```mermaid
classDiagram
    class ProviderRequestHandler {
      +handle(providerCode, request) HttpResponse
    }

    class ProviderAdapter {
      <<interface>>
      +verify(request)
      +parse(request) ProviderCommand
      +formatSuccess(result) HttpResponse
    }

    class ProviderPolicy {
      <<interface>>
      +resolveEngine(command) EngineKind
      +buildContext(command) ProviderContext
    }

    class ProtocolEngine {
      <<interface>>
      +handle(command, context) EngineResult
    }

    class SlotegratorAdapter
    class TreasurePruneAdapter
    class SlotegratorPolicy
    class TreasurePrunePolicy
    class SeamlessWalletEngine
    class StatefulTransactionEngine
    class CasinoCoreGateway {
      <<interface>>
      +applyBetDebit(input)
      +applyWinCredit(input)
      +applyRefund(input)
    }

    class IntegrationInboxRepository {
      <<interface>>
    }
    class ExternalTransactionRepository {
      <<interface>>
    }
    class ExternalRoundRepository {
      <<interface>>
    }
    class PendingOperationRepository {
      <<interface>>
    }

    ProviderRequestHandler --> ProviderAdapter
    ProviderRequestHandler --> ProviderPolicy
    ProviderRequestHandler --> ProtocolEngine

    SlotegratorAdapter ..|> ProviderAdapter
    TreasurePruneAdapter ..|> ProviderAdapter

    SlotegratorPolicy ..|> ProviderPolicy
    TreasurePrunePolicy ..|> ProviderPolicy

    SeamlessWalletEngine ..|> ProtocolEngine
    StatefulTransactionEngine ..|> ProtocolEngine

    SeamlessWalletEngine --> IntegrationInboxRepository
    SeamlessWalletEngine --> ExternalTransactionRepository
    SeamlessWalletEngine --> ExternalRoundRepository
    SeamlessWalletEngine --> PendingOperationRepository
    SeamlessWalletEngine --> CasinoCoreGateway

    StatefulTransactionEngine --> IntegrationInboxRepository
    StatefulTransactionEngine --> ExternalTransactionRepository
    StatefulTransactionEngine --> ExternalRoundRepository
    StatefulTransactionEngine --> CasinoCoreGateway
```

---

## 5. Минимальные таблицы

### 5.1. Integration Platform

```sql
create table providers (
  code varchar(50) primary key,
  name varchar(100) not null
);

create table provider_policies (
  provider_code varchar(50) primary key,
  engine_kind varchar(50) not null,
  round_match_strategy varchar(50) not null,
  callback_timeout_seconds int not null,
  supports_cancel boolean not null default false,
  supports_complete boolean not null default false,
  supports_promo boolean not null default false
);

create table integration_inbox (
  id bigint primary key generated always as identity,
  provider_code varchar(50) not null,
  dedupe_key varchar(255) not null unique,
  status varchar(30) not null,
  response_json json not null,
  created_at timestamp not null
);

create table external_rounds (
  id bigint primary key generated always as identity,
  provider_code varchar(50) not null,
  external_round_id varchar(100) not null,
  external_session_id varchar(100),
  internal_round_id bigint not null,
  unique (provider_code, external_round_id)
);

create table external_transactions (
  id bigint primary key generated always as identity,
  provider_code varchar(50) not null,
  external_transaction_id varchar(100) not null,
  operation_kind varchar(50) not null,
  external_round_id varchar(100),
  internal_round_id bigint,
  state varchar(30) not null,
  amount numeric(18,2),
  currency varchar(10),
  response_json json,
  unique (provider_code, operation_kind, external_transaction_id)
);

create table external_transaction_links (
  id bigint primary key generated always as identity,
  provider_code varchar(50) not null,
  source_transaction_id bigint not null,
  related_transaction_id bigint not null,
  relation_kind varchar(30) not null
);

create table pending_operations (
  id bigint primary key generated always as identity,
  provider_code varchar(50) not null,
  operation_kind varchar(50) not null,
  external_transaction_id varchar(100) not null,
  payload_json json not null,
  status varchar(30) not null
);
```

### 5.2. Casino Core

```sql
create table wallets (
  player_id bigint primary key,
  currency varchar(10) not null,
  balance numeric(18,2) not null
);

create table ledger_entries (
  id bigint primary key generated always as identity,
  player_id bigint not null,
  round_id bigint,
  entry_kind varchar(30) not null,
  amount numeric(18,2) not null,
  currency varchar(10) not null,
  created_at timestamp not null
);

create table rounds (
  id bigint primary key generated always as identity,
  player_id bigint not null,
  game_id varchar(100) not null,
  status varchar(30) not null
);
```

---

## 6. Какие сущности реально нужны на старте

Если не усложнять, то для первого рабочего варианта достаточно:

- `ProviderRequestHandler`
- `SlotegratorAdapter`
- `TreasurePruneAdapter`
- `SlotegratorPolicy`
- `TreasurePrunePolicy`
- `SeamlessWalletEngine`
- `StatefulTransactionEngine`
- `CasinoWalletService`
- `IntegrationInboxRepository`
- `ExternalTransactionRepository`
- `ExternalRoundRepository`
- `PendingOperationRepository`

Этого достаточно, чтобы:

- принять запрос;
- защититься от duplicate;
- связать внешний round с внутренним;
- применить money-effect;
- корректно обработать `refund before bet` и `trx.complete`.

---

## 7. Как идут вызовы во время игры

### 7.1. Slotegrator `action=bet`

```mermaid
sequenceDiagram
    participant P as Slotegrator
    participant A as SlotegratorAdapter
    participant PP as SlotegratorPolicy
    participant E as SeamlessWalletEngine
    participant IR as Integration Repos
    participant C as CasinoCore

    P->>A: POST action=bet
    A->>A: verify signature
    A->>PP: buildContext(command)
    PP->>E: engine=seamless-wallet
    E->>IR: findProcessed(dedupeKey)
    E->>IR: openOrAttach external_round
    E->>C: applyBetDebit(...)
    C-->>E: new balance
    E->>IR: save external_transaction
    E->>IR: mark inbox processed
    E-->>A: EngineResult
    A-->>P: success response
```

Коротко:

- адаптер только проверяет и парсит;
- policy говорит, как строить dedupe key и как матчить round;
- engine делает идемпотентность и orchestration;
- core реально списывает деньги.

### 7.2. Slotegrator `action=refund`, если исходной ставки еще нет

```mermaid
sequenceDiagram
    participant P as Slotegrator
    participant A as SlotegratorAdapter
    participant PP as SlotegratorPolicy
    participant E as SeamlessWalletEngine
    participant IR as Integration Repos

    P->>A: POST action=refund
    A->>PP: buildContext(command)
    PP->>E: refund may be orphan
    E->>IR: find source bet
    IR-->>E: not found
    E->>IR: save pending refund
    E->>IR: mark inbox processed
    E-->>A: accepted
    A-->>P: success response
```

Коротко:

- деньги пока не двигаются;
- операция сохраняется в `pending_operations`;
- наружу все равно отдается success.

### 7.3. Treasure-prune `withdraw.bet`

```mermaid
sequenceDiagram
    participant P as Treasure-prune
    participant A as TreasurePruneAdapter
    participant PP as TreasurePrunePolicy
    participant E as StatefulTransactionEngine
    participant IR as Integration Repos
    participant C as CasinoCore

    P->>A: POST withdraw.bet
    A->>A: verify signature
    A->>PP: buildContext(command)
    PP->>E: recovery=cancel-on-undefined
    E->>IR: findProcessed(dedupeKey)
    E->>IR: openOrAttach external_round
    E->>C: applyBetDebit(...)
    C-->>E: new balance
    E->>IR: save applied transaction
    E->>IR: mark inbox processed
    E-->>A: success
    A-->>P: success response
```

### 7.4. Treasure-prune `trx.complete`

```mermaid
sequenceDiagram
    participant P as Treasure-prune
    participant A as TreasurePruneAdapter
    participant PP as TreasurePrunePolicy
    participant E as StatefulTransactionEngine
    participant IR as Integration Repos

    P->>A: POST trx.complete
    A->>PP: buildContext(command)
    PP->>E: engine=stateful-transaction
    E->>IR: find source transaction
    E->>IR: mark source as completed
    E->>IR: save link(source -> complete)
    E->>IR: mark inbox processed
    E-->>A: success
    A-->>P: success response
```

Коротко:

- `trx.complete` не должен второй раз двигать деньги;
- он должен обновить состояние внешней транзакции;
- при необходимости сохранить связь между исходной транзакцией и recovery-вызовом.

---

## 8. Как это выглядело бы в runtime

Минимальный runtime wiring:

```ts
const handler = new ProviderRequestHandler(
  new Map([
    ["slotegrator", new SlotegratorAdapter()],
    ["treasure-prune", new TreasurePruneAdapter()],
  ]),
  new Map([
    ["slotegrator", new SlotegratorPolicy()],
    ["treasure-prune", new TreasurePrunePolicy()],
  ]),
  new Map<EngineKind, ProtocolEngine>([
    [
      "seamless-wallet",
      new SeamlessWalletEngine(
        inboxRepo,
        externalTransactionRepo,
        externalRoundRepo,
        pendingOperationRepo,
        casinoCore,
      ),
    ],
    [
      "stateful-transaction",
      new StatefulTransactionEngine(
        inboxRepo,
        externalTransactionRepo,
        externalRoundRepo,
        casinoCore,
      ),
    ],
  ]),
);
```

И точка входа:

```ts
app.post("/callbacks/slotegrator", async (req, res) => {
  const response = await handler.handle("slotegrator", req);
  res.status(response.status).send(response.body);
});

app.post("/callbacks/treasure-prune", async (req, res) => {
  const response = await handler.handle("treasure-prune", req);
  res.status(response.status).send(response.body);
});
```

---

## 9. Что важно не забыть

Даже в простом варианте нельзя убирать:

- `integration_inbox`;
- dedupe key;
- `external_transactions`;
- `external_rounds`;
- `pending_operations`;
- единый `ledger_entries`.

Именно эти части не дают архитектуре скатиться в "толстые адаптеры с собственной денежной логикой".

---

## 10. Практический минимум для первого релиза

Если делать MVP этой архитектуры, я бы начал так:

1. Поднять `SlotegratorAdapter`, `TreasurePruneAdapter`.
2. Поднять `SlotegratorPolicy`, `TreasurePrunePolicy`.
3. Поднять `SeamlessWalletEngine`, `StatefulTransactionEngine`.
4. Сделать 5 таблиц:
   - `integration_inbox`
   - `external_rounds`
   - `external_transactions`
   - `external_transaction_links`
   - `pending_operations`
5. Подключить `wallets`, `ledger_entries`, `rounds`.

Этого уже достаточно, чтобы схема была:

- не игрушечной;
- не перегруженной;
- расширяемой под новых провайдеров.
