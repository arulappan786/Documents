# Rate Engine — Microservice Architecture Assessment

**Central question: *What is keeping the Rate Engine from being a true microservice?***

The service being a separate deployable is a settled decision — two upstream systems will consume it, which
*strengthens* the case. This assessment is therefore a **blocker analysis**: against the defining properties
of a microservice (independently deployable, owns its data, coupled only through versioned contracts,
consumer-agnostic, independently scalable, correct under failure), what is standing in the way?

**Sources analyzed**
- Rate Engine service repo: `C:\Users\jcastillo\source\repos\VertexOne Retail Rate Engine Service`
- Package-owned DDL: `…\.nuget\packages\vertexone.retail.rateengine.infrastructure\3.563.0-beta0081\contentFiles\any\any\migrations\RateEngineDbContext.sql`
- Legacy monolith (VXretail): `C:\Local Data\Work\Code Analysis\Utilibill30`

**Note on precision.** Table-level conclusions are pinned to package `3.563.0-beta0081` (the version both
`…App.csproj` and `…Data.csproj` reference); the schema is in flux (`3.565.0` differs), so re-verify before
acting. Data-ownership verdicts are inferred from names + code-usage and should be validated with the team.
Idempotency and consumer-coupling findings are from direct code reads (file:line cited in the appendix).

---

## Executive summary — the blocker scorecard

| # | Blocker question (what a "true microservice" needs) | Verdict | Severity |
|---|---|---|---|
| 1 | **Deploy independently** — no coordinated release with VXretail, the API, or a shared package | ❌ Blocked | High |
| 2 | **Own its datastore** — every table used by it; nothing else reads/writes it directly | ❌ Blocked | High |
| 3 | **Couple only through a versioned contract** — consumers never bind to our internal types/enums/tables | ❌ Blocked | High |
| 4 | **Be consumer-agnostic** — a 2nd upstream can onboard without VXretail-specific rework | ⚠️ Partial | Medium |
| 5 | **Scale out** — multiple instances/threads under load | ⚠️ At risk | Medium |
| 6 | **Stay correct under redelivery/failure** — no double-billing on a replayed message | ⚠️ **Partial** — business-key idempotency covers single-account, pool, band, and datasync paths; no unit-of-work transaction or transport inbox yet | Medium |
| 7 | **Evolve its schema** without a multi-repo coordinated release | ❌ Blocked | High |

**The story.** Internally the service is well-built — SOLID seams, message-driven, fail-loud with an audit
trail. But it is not yet a microservice, for two distinct reasons:

- **A structural knot (blockers 1, 2, 3, 7).** One shared **NuGet package** owns the schema/entities/enums and
  one shared **database** is used by this worker *and* a separate REST API, and the engine is **system of
  record for commercial data it doesn't own** (e.g. `Contract` → `payment terms` live in the calc DB). These
  four blockers are the same root, addressed in *The structural knot* below.
- **Runtime correctness (blocker 6).** Redelivery idempotency is enforced at the business-key level across
  the single-account, pool, band, and datasync paths, but two durable layers are missing — a unit-of-work
  transaction and a transport-level inbox — so narrow reprocessing windows remain.

The structural knot is the larger architectural program, led by the data-ownership fix; blocker 6's remaining
work (the transaction + inbox) is a contained correctness follow-up.

---

# Part I — The structural knot (blockers 1, 2, 3, 7)

These four are one problem wearing four hats: a shared package + shared database + wrong system-of-record
bind three deployables (VXretail UI, the REST API, this worker) into a distributed monolith. The root is
data ownership.

## The root — data-ownership inversion

The problem isn't *unused columns*; it's **the wrong service acting as system of record (SoR)** for concepts
that live beyond calculation.

### The SoR test
For any concept (Contract, Account, Product…), the system of record is the service that (1) **governs its
lifecycle**, (2) holds the **widest, authoritative model**, and (3) **wins in a conflict**. The SoR masters
the **full** record; every other service keeps only the **subset it interprets**, as a **read-only projection**
synced one-directionally (owner → consumer), keyed by a shared business key.

### Deliberate projection vs. accidental master
Duplication is not the anti-pattern — **wrong ownership + reach-in is.**

| | Accidental master (today) | Deliberate projection (target) |
|---|---|---|
| Authority | A downstream service masters another context's concept | The concept's true owner masters it |
| Columns | Stores fields it never uses (e.g. `payment terms`) | Stores only the fields it interprets |
| Direction | The real owner **reaches into** the downstream store | Owner **pushes** changes; consumer subscribes |
| Result | God table shared across contexts | Fit-for-purpose local read model |

### Worked example — Contract
A Contract is a **commercial** concept (parties, payment terms, billing config, dates, customer linkage);
pricing is one consumer of it. Today the Rate Engine is SoR for the whole Contract — **VXretail has no
contract table and reads the engine's** — so `payment terms` sit in the calc DB and the billing platform
depends on a peripheral calculation service for its own master data. **That is the inversion.** Target: the
concept **splits** — VXretail masters the header (incl. `payment terms`); the engine keeps a projection
(contract key + priced-on attributes) plus the pricing structure it genuinely owns (`ContractDetail`,
components, TOU). VXretail publishes `ContractChanged`; the engine updates its projection and **stops being
read from**. For a batch engine the projection is a **local subset table refreshed by events**, not a
per-calculation call.

### Table-by-table ownership call-out (all 30 tables)
Grouped by *who should be SoR*. Rows marked *boundary* need a domain call.

**A — Rate Engine correctly owns; keep in full** (pricing structure it authors + calc outputs it produces):
`Component`, `ComponentDeterminant`, `ContractDetail`, `TOUSchedule`, `TOUTier`, `InvoiceLineItem`,
`BackingSheet`, `EphemeralBackingSheet`, `BandDetail`, `BillingException`, `CalculationClaim`.
`InvoiceLineItem` flowing *back* to VXretail's billing register is correct — RE is SoR for the calculation.

**B — VXretail should master; RE keeps a needs-only projection:**

| Table | Why VXretail should be SoR | RE should keep |
|---|---|---|
| `Contract` | Commercial agreement; payment terms/billing span beyond calc | contract key + priced-on attributes (`ContractDetail` stays) |
| `Account` | Customer account is a billing/CRM concept | account key + service characteristics the calc reads |
| `AccountAttribute` | Attributes of a VXretail-owned entity | only attributes used as determinants (`@AccountAttribute[...]`) |
| `Client` | Marketer/tenant identity is a platform concept | reference/key only |
| `Usage` | Raw meter reads originate in VXretail/MDM | the interval subset needed to price (bulk also in ClickHouse) |
| `Product` *(boundary)* | Retail offer = commercial identity **and** pricing composition | pricing composition (`ProductComponent`); commercial identity → VXretail |
| `AccountMapping` *(boundary)* | Enrollment association is a VXretail lifecycle fact | the mapping + dates the calc needs, as a projection |

**C — Shared reference/lookup; own upstream, don't warehouse in RE:** `ChargeType`, `ChargeTypes` (VXretail
already has its own `ChargeTypes` with a `RateEngineCharge` flag — duplicated lookup the engine never
queries), `BillingArgument`, `NRGProfile`. All 0 code refs — clearest "remove from RE" candidates.

**D — Outside the calc bounded context (identity/audit):** `AppUser` (0 refs), `AuditLog` (0 refs),
`AppProfile` (read once; likely tenant/config profile).

**E — Infrastructure; leave as-is:** `InboxState`, `OutboxMessage`, `OutboxState`, `__EFMigrationsHistory`
(`BlockCalculationClaim` orphaned, 0 refs — retire).

**Net:** the datastore should shrink toward **A + E**; **B/C/D** are mastered elsewhere and should be held (if
at all) only as projections keyed by a shared identifier. This one reframing resolves blockers 1, 2, 3, and 7.

## Blocker 2 — Own its datastore  ❌
The DB serves **five concerns at once**: calc inputs, calc outputs/audit, configuration SoR, identity/auth,
messaging outbox. Beyond the wrong-SoR tables above, it is **physically shared by two deployables**: this
worker (writes config via the `datasync` consumer + writes calc outputs) and a **separate Rate Engine REST
API** (serves the CRUD/query endpoints the monolith calls, on the same tables). Two services on one database
= the **shared-database anti-pattern**, and it **splits the writer**: config reaches the tables via both the
REST API and the worker's `datasync` path, with no single owner. *(VXretail itself does not touch the DB —
its integration is clean REST + RabbitMQ; the sharing is internal to the Rate Engine.)*

## Blocker 3 — Couple only through a versioned contract  ❌
Consumers bind to the engine's **internals**: the message DTOs and enums are defined in the shared
`Infrastructure` package (`SubmitCalculationOrderConsumer.cs:18-21`, `EnvelopMessageConsumer.cs:14-19`), so a
change to an internal type is a change to everyone's contract. A true microservice exposes only a **thin,
versioned, published contract** (message/API schema) and keeps its `DbContext`/entities private.

## Blocker 1 — Deploy independently  ❌
The `Infrastructure` package owns the `DbContext`, entities, enums, repositories, **and** migrations, consumed
by the worker, the API, and (DTOs) the monolith. Per `CLAUDE.md`, an enum/entity change means publishing a
package and bumping it **in lockstep** across `…App.csproj` + `…Data.csproj` (+ the API). No independent
deploy, no schema encapsulation, change amplification — the distributed-monolith signature.

## Blocker 7 — Evolve its schema independently  ❌
There are **no migrations in this repo** (no `Migrations/` folder, no `Migrate()`/DbUp/FluentMigrator). The
DDL ships as a **content file inside the NuGet package** (`…\migrations\RateEngineDbContext.sql`, EF Core
`InitialMigration`; nuspec `repository commit="4ffd9e35…"` → a separate Infrastructure git repo). A schema
change is therefore a cross-repo package release, not a local migration. *(The REST API the monolith calls is
likewise a separate hosted service — `RateEngineApiBaseUrl` — in neither analyzed repo.)*

---

# Part II — The runtime blockers (4, 5, 6)

## Blocker 6 — Stay correct under redelivery  ⚠️ **Partial**

RabbitMQ is at-least-once (redelivery on retry, consumer restart, or manual replay), so every path must
tolerate reprocessing the same message. Redelivery idempotency today is enforced at the **business-key**
level, per path:

- **Single-account (non-pool, non-band):** in `Product.ComputeInvoiceLineItemsAsync`, before billing an
  `Original` reading the run checks — via a single `EXISTS` — whether prior `Pending` output already exists
  for `(account, TransactionReferenceNumber, ClientId)`. If so it invalidates and replaces it, so a
  redelivery/replay of the same TxnRef does not duplicate; a first delivery skips the sweep. Transport
  counters (`GetRedeliveryCount`/`GetRetryAttempt`) are **not** the signal — they read `0` after a consumer
  restart or manual republish — so the persisted business key is used instead.
- **Multi-account Block pool:** the `CalculationClaim` — a pool-level concurrency lock keyed by
  `(ClientId, ContractNumber, ServicePeriod, ClaimType)`, not by TxnRef/account — makes a `Completed` pool a
  no-op on redelivery.
- **Band pool:** the durable `BandDetail` existence check (`ProcessBandDetailResult`) prevents re-pricing.
  (Band-bearing calcs are deliberately excluded from the single-account sweep above, since
  `ProcessBandChargesIfNecessary()` writes this run's band ephemeral backing sheets under the same TxnRef.)
- **Datasync:** `Contract` insert is existence-guarded by `(ClientId, ContractNumber)`; `Account` is upserted
  on `(AccountNumber, ClientId)`; `AccountMapping` extends the existing row instead of inserting a duplicate
  (`ProcessContractMessage`, `UpsertAccountMapping`).

**Two gaps remain:**
- **No unit of work.** `GenericRepositoryHelper` calls `SaveAsync` per entity (`:48-58, 166-174`), so a throw
  mid-run can leave committed partial rows, and the business-key sweep and re-insert commit separately (a
  failure between them can leave an account's prior bill invalidated with no replacement — a recoverable
  *missing* bill). A single transaction around calc-and-persist would remove both windows.
- **No transport inbox.** `DependencyInjectionExtensions.cs` configures retry + concurrency but no
  `AddEntityFrameworkOutbox`/`InboxState` dedup, so a redelivery arriving *after* items leave `Pending`, or a
  re-published message with a fresh `MessageId`, is not caught by the business-key sweep. A MassTransit inbox
  (dedup by `MessageId`) is the durable close — gated on the Infrastructure package registering the inbox
  entity (draft `SPEC_Invoice_Message_Publishing_Outbox.md`).

## Blocker 4 — Be consumer-agnostic  ⚠️ Partial

**Inbound/calculation is substantially agnostic; the VXretail shape is concentrated in the *outbound* path and
config.** Good news first — the engine has a **real multi-tenant `Client` model**: messages carry
`ClientNumber` + `AppProfileId` resolved to a `Client` (`SubmitCalculationOrderConsumer.cs:189, 388-392`),
every calc query is client-scoped, inbound queues are **neutral and shared**
(`taskservice.rateengine.meterreading` / `.datasync`), and there is **no account-number-format assumption**
(free-form string; tests use both long numerics and `ACC-…` alphanumerics). **A second upstream onboards as a
new `Client` row publishing to the same neutral queues.** That's the core reason it's mostly agnostic.

Couplings that must be addressed to onboard a second upstream:

| Coupling | Where | Rating |
|---|---|---|
| Outbound `InvoiceLineItemMessage` stamps `DatabaseName="Utilibill24DB"` + `DatabaseServer` — tells the downstream *which VXretail SQL DB to write into* | `InvoiceLineItemMessageService.cs:216-227`, `appsettings.json:31-34` | **Hard blocker** (non-Utilibill downstream has no such DB); also global, not per-client |
| Outbound queue bases `taskservice.utilibill.*`, single global (per-client *suffix* only) | `appsettings.json:44-45`, `InvoiceLineItemMessageService.cs:233-251` | Soft (config + per-client support) |
| Market-profile mappings are **global singletons**, one ISO/zone/utility convention for all clients | `appsettings.json:67-136`, `MarketProfileResolverService.cs:32-48,186-189` | Soft (only if the 2nd upstream names ISO/utility differently; the promised `MarketProfileMap` DB table would fix it) |
| Shared, VXretail-named `vxretail_dw` ClickHouse warehouse sink | `EnvelopMessageConsumer.cs:87,746`, `CalculationLedger*.sql` | Soft |
| Stray `VertexOne.VXretail.Messages.Models` pkg ref (unused in code), enum/context names, elastic host, NuGet feed | `App.csproj:38`, `ClickhouseContextFactory.cs:16-17`, `appsettings.json:11` | Cosmetic |

**Verdict:** inbound-agnostic, outbound-VXretail-shaped. To onboard upstream #2: (1) give it a real
result-delivery target instead of `Utilibill24DB` (the one hard blocker), (2) make the outbound queue base and
DW per-client or neutral, (3) confirm its ISO/utility/zone naming fits the global market mappings (or finish
the `MarketProfileMap` table). None require touching calculation logic.

## Blocker 5 — Scale out  ⚠️ At risk (needs load verification)

Per `CLAUDE.md`, **NoStringEvaluator keeps process-global state and serializes facade creation through
`NoStringEvaluatorLock.FacadeLock`** — the formula engine every calculation depends on has a process-wide
lock. Combined with `UseConcurrencyLimit(10)`, this raises a real question about horizontal/vertical scaling
under load that the structural findings don't touch. **Not investigated in depth here** — flagged as an open
item: load-test concurrent calculation throughput and confirm the global lock isn't the ceiling before
assuming the worker scales by adding replicas/threads.

---

## What's already working (preserve these)

The internal design is genuinely good and should not regress during any refactor:
- **Single responsibility / clean seams** — `BlockDispatchService`, `BlockResidualChainService`,
  `BillingExceptionService`, `MarketProfileResolverService`; decision tables extracted (`BlockComponentTypeMap`).
- **DIP / testability** — everything behind `I…` interfaces + DI; Testcontainers integration tests.
- **Open/closed, fail-loud** — exhaustive switches that throw on unmapped cases; `BillingException` +
  `BackingSheet` audit on every path.
- **Message-driven integration & a real multi-tenant `Client` model** — the foundation the fixes build on.

---

## Recommendations (prioritized)

**Tier 0 — Correctness (urgent, independent of the architecture work)**

1. **Close the two remaining idempotency gaps (blocker 6).** Business-key idempotency already covers the
   single-account, pool, band, and datasync paths. Add the durable layers:
   - **Unit of work:** wrap calc-and-persist in a single transaction (removes partial commits and the
     invalidate-then-fail "missing bill" window).
   - **Transport inbox:** MassTransit EF `AddEntityFrameworkOutbox` **inbox** (dedup by `MessageId`) — closes
     redelivery uniformly across calculate **and** datasync, including the after-`Pending` window the
     business-key sweep can't cover. Gated on the Infrastructure package registering the inbox entity (see
     `SPEC_Invoice_Message_Publishing_Outbox.md` §5.1).

**Tier 1 — The structural knot (the "true microservice" program)**

2. **Fix the data-ownership inversion (blockers 1, 2, 7).** Make VXretail SoR for `Contract` (own contract
   table incl. `payment terms`), then `Account`, `AccountAttribute`, `Client`, `Usage`, and the boundary
   entities. The engine keeps needs-only projections synced VXretail→RE; VXretail stops reading contracts from
   the engine.
3. **Break the shared-package coupling / own the schema in-solution (blockers 1, 3, 7).** Move
   `DbContext`/entities/migrations into an in-solution `Infrastructure` project; publish only a **thin
   contracts package** for consumers (blocker 3). See target architecture.
4. **Split remaining non-calc concerns out of the DB (blocker 2).** Move identity (`AppUser`, `AppProfile`),
   audit (`AuditLog`), and lookups (`ChargeType(s)`, `BillingArgument`, `NRGProfile`) to their owners; retire
   orphan tables (`ChargeTypes`, `BlockCalculationClaim`).
5. **Define one writer per table (blocker 2).** Decide REST API **or** worker `datasync` — not both.

**Tier 2 — Consumer-agnostic & scale (blockers 4, 5)**

6. **Neutralize the outbound path (blocker 4).** Replace the hardcoded `Utilibill24DB` result target with a
   per-client delivery target; make outbound queue base + DW per-client or neutral; complete the
   `MarketProfileMap` DB table so ISO/utility naming isn't a single global set.
7. **Load-test concurrency (blocker 5).** Verify the NoStringEvaluator global lock isn't the throughput ceiling
   before relying on replica/thread scaling.

**Tier 3 — Hygiene**

8. **Get secrets out of `appsettings.json`** (live staging/QA RMQ/SQL/ClickHouse/Elastic creds in plaintext);
   rotate them. **Document** where the schema/API live (Infrastructure repo URL + commit) until they move
   in-house. **Add schema governance** (changelog per release; new tables must be referenced before merge).

---

## Target architecture — two hosts + owned schema

The REST API and the message consumer are both **inbound adapters** to the *same* bounded context. Share the
domain/application logic (one library, never duplicated); host as **two thin hosts**:

```
RateEngine.sln
├─ RateEngine.Domain          (entities, calc rules — the core)
├─ RateEngine.Application     (use-cases; both hosts call these)
├─ RateEngine.Infrastructure  (DbContext, repos, migrations, ClickHouse)  ← owns the schema
├─ RateEngine.Contracts       (DTOs/messages — thin, publishable to consumers)  ← the versioned contract (blocker 3)
├─ RateEngine.Api    (Sdk=Web)     — thin: controllers → Application
└─ RateEngine.Worker (Sdk=Worker)  — thin: MassTransit consumers → Application
```

**Why two hosts, not one process.** API is synchronous/latency-bound (UI users); worker is async/CPU-bound
(calc volume). Separate hosts so a heavy calc batch can't starve the API and an API hotfix doesn't restart
consumers mid-batch. A single combined host is acceptable only at low/steady volume; because the domain is in
shared libraries, splitting later is mechanical.

**Independently deployable — yes, at the artifact level.** Each host is its own build artifact/pipeline/scale
unit; shared libraries are compiled *into* each host (no shared running component).

**The residual coupling is the shared database.** Keep deploys independent with:
1. **Expand → migrate → contract** (backward-compatible migrations): add additively → deploy code that uses it
   → drop the old shape only after nothing references it. Every step leaves the schema understood by whatever
   host version is running, so hosts deploy in any order.
2. **Exactly one migration owner — never auto-migrate on startup** (both hosts racing `Migrate()` can
   deadlock/half-apply). Prefer a **gated pipeline step** (`dotnet ef database update` / idempotent script) run
   before rollout; `__EFMigrationsHistory` is the single source of applied state.
3. **Only breaking changes force ordered deploys** — the flagged exception.

This also closes blockers 1/3/7: schema owned in-solution, consumers depend only on the thin contracts package.

**Deployment checklist**
- [ ] Domain/application logic in shared libraries; API and Worker hosts are thin adapters.
- [ ] Each host = its own artifact + pipeline + scaling policy.
- [ ] Schema + migrations owned in-solution (not a cross-repo NuGet package).
- [ ] Consumers depend only on the thin `Contracts` package, never on `DbContext`/entities.
- [ ] All migrations additive (expand-contract); one migration owner (pipeline step); no host auto-migrates.
- [ ] Idempotency barrier (inbox/dedup) in place before relying on at-least-once delivery.

---

## Evidence appendix (key file references)

- **Idempotency (blocker 6):** business-key invalidation + `EXISTS` redelivery detector (single-account) —
  `Product.cs` `ComputeInvoiceLineItemsAsync`; pool claim (multi-account, pool-level, keyed by
  contract+period+type) — `CalculationClaimService.cs`; band `BandDetail` existence check —
  `Product.cs` `ProcessBandDetailResult`; datasync existence guards — `EnvelopMessageConsumer.cs`
  (`ProcessContractMessage`, `UpsertAccountMapping`); regression —
  `SubmitCalculationOrderConsumerTests.ProcessMessage_RedeliveredSingleAccountMeterReading_DoesNotDoubleBill`.
  Gaps: no unit of work — `GenericRepositoryHelper.cs:48-58,166-174`; no outbox/inbox —
  `DependencyInjectionExtensions.cs`.
- **Consumer-agnostic (blocker 4):** multi-tenant client model — `SubmitCalculationOrderConsumer.cs:189,388-392`;
  neutral inbound queues — `appsettings.json:42-43`; hard outbound coupling `Utilibill24DB` —
  `InvoiceLineItemMessageService.cs:216-227`, `appsettings.json:31-34`; global market mappings —
  `MarketProfileResolverService.cs:32-48,186-189`; stray VXretail pkg ref — `App.csproj:38`.
- **Scalability (blocker 5):** NoStringEvaluator process-global state + `FacadeLock` (`CLAUDE.md`);
  `UseConcurrencyLimit(10)` — `DependencyInjectionExtensions.cs`.
- **Structural (blockers 1,2,3,7):** worker not web — `…App.csproj:1`, `Program.cs`; package-owned migration —
  `…\migrations\RateEngineDbContext.sql`, nuspec `commit="4ffd9e3579dcaba455c544eb0416c27df934561f"`; lockstep
  bump — `CLAUDE.md`; VXretail integrates remotely (no DB) — `Utilibill30\CommonClasses\Services\RateEngineApiService.vb`,
  `RabbitMQSingleton.vb`; unused-table DDL lines — AppUser 37, AuditLog 59, BillingArgument 87, NRGProfile 181,
  ChargeType 4104, ChargeTypes 4148, BlockCalculationClaim 4715.
