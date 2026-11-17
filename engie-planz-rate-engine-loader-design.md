# Engie Plan Z: Rate Engine Staging Loader (draft design)

Owner: Arulappan Antonysamy, guided by Dilip Reddy. Status: draft for review with Dilip.
Sources:
- Group call "Engie | Future-Dated Enrollments", 16 Nov 2026.
- 1:1 call between Arulappan and Dilip, 17 Nov 2026, 00:47 IST.
- RateEngine code in UBCollection.

## 1. Problem
From go-live (~21 Dec) until March, Therm and EDI are not live, and Engie will not call our APIs. Engie already loads rate-engine data by SQL into the data-migration DB (`VXRAZVWMINISQL01.Vxretail_Data_Migration_Engie`). They will keep doing that for five tables: **Account, AccountAttribute, Contract, ContractDetail, AccountMapping**. We need an **automated** job that moves those rows into the production Rate Engine DB, through our validations, and tells Engie what failed.

Enrollments (the VXR inbound customer, service-location and meter tables) still come through the existing enrollment import (`InboundFiles/CustomerDataImport.vb`). This loader **ignores the VXR inbound tables**. The Rate Engine and VXR are tied only by utility account number + utility ID (1:1 call, 00:03:40 to 00:03:59).

## 2. Key technical finding: don't write Rate Engine tables with raw SQL
`RateEngineDbContext` has EF Core interceptors that run on every save:
- `ReplicationSaveChangesInterceptor` sends Component, Product, Contract, ContractDetail, AccountMapping and Account changes to VXR Retail. The message goes over RabbitMQ to the queue `taskservice.utilibill.rateenginereplication.<clientNumber>`.
- `AuditSaveChangesInterceptor` → `UnbilledUsageRecalculationService` re-sends unbilled usage for the affected accounts.

A linked-server or T-SQL insert into the Rate Engine DB **bypasses both**. VXR Retail would never receive the contracts, details, mappings or accounts, and unbilled usage would not be recalculated.

**Decision proposed:** the loader writes through the existing Rate Engine write path. The recommended way is the Rate Engine API's Add endpoints, called internally with a service account. That gives us:
- the same validations as production (they sit inline in the controllers' `Add` methods), and
- the replication and recalculation side effects, for free.

Engie still does no API work. Only our loader calls the API.

## 3. Architecture
```
Engie (SQL inserts)
   │
   ▼
Staging DB (Vxretail_Data_Migration_Engie, or a dedicated one with the 5 tables)
   ├─ Account / AccountAttribute / Contract / ContractDetail / AccountMapping   (+ control columns)
   └─ <Table>_Error  (one parallel error table per staging table, read-only for Engie)
   │
   ▼  RE Staging Loader (.NET worker / console app run on a schedule; SQL Agent or Windows Task)
   │   1. claim rows  2. validate  3. upsert via Rate Engine API  4. write status / errors
   ▼
Rate Engine API ──EF + interceptors──► Rate Engine DB ──RabbitMQ──► VXR Retail (replication)
```
A linked server is not needed. The loader holds two connections: the staging DB and the Rate Engine API.

## 4. Staging table changes (all 5 tables)
| Column | Purpose |
|---|---|
| `StagingRowId BIGINT IDENTITY` | stable row key |
| `BatchId` (Engie supplies, e.g. `ENGIE-20261218-001`) | group a load; Engie can re-query or re-feed a whole batch (Dilip, 1:1 call 00:10:04 to 00:11:15) |
| `CreatedOn DATETIME2 DEFAULT SYSUTCDATETIME()` | load time |
| `LoadStatus` (`New`, `Processing`, `Loaded`, `Skipped`, `Error`) | loader state; Engie inserts as `New` |
| `ProcessedOn`, `RateEngineId UNIQUEIDENTIFIER NULL` | audit: the RE key that was created or matched |

## 5. Error tables (agreed on the 1:1 call, 00:09:28 to 00:13:49)
Create `<Table>_Error` for each staging table. Each holds the same data columns, plus:
- `ErrorId`, `StagingRowId`, `BatchId`
- `ErrorCode`, `ErrorMessage` (all reasons, not just the first)
- `ErrorOn`
- `IsResolved`, `ResolvedOn`, `ResolvedByStagingRowId`

Rules:
- Error tables are **read-only for Engie**, so the audit history stays intact (Dilip).
- **Fix flow:** Engie inserts a corrected row with a new `BatchId`. When it loads, the loader marks earlier errors for the same natural key `IsResolved = 1` and links them to the corrected row.
  - This reconciles the two ideas from the call: Arulappan's "mark resolved, reprocess" and Dilip's "read-only log".
  - If you prefer editing in place, give Engie a single updatable `RetryRequested` flag on the staging row instead.
- **Reporting:** a view `vw_EngieLoadResult` per `BatchId`, with counts (loaded, skipped, error) and the error details.

## 6. Processing order and natural keys
The loader processes in parent → child order. If a parent row fails, its children go to their error tables as `PARENT_FAILED`.

| # | Table | Natural key used for upsert / lookup |
|---|---|---|
| 1 | Contract | ClientId + ContractNumber |
| 2 | ContractDetail | ContractNumber + SubAgreementNumber |
| 3 | Account | ClientId + AccountNumber (+ Utility) |
| 4 | AccountAttribute | AccountNumber + AttributeName + EffectiveStartDate |
| 5 | AccountMapping | AccountNumber + SubAgreementNumber/ProductId + EffectiveStartDate |

The loader resolves GUIDs by **natural key**, not by Engie's staging GUIDs. Production GUIDs are generated by us. The Therm GUID travels in Product **Name** (group call, 00:19:00 to 00:22:55).

**Idempotency:**
- A row that already exists and is identical → `Skipped`.
- A row that exists but differs → handled per the rule we choose (update or error). **Open question for Dilip.**
- Accounts and mappings may already exist, because VXR autocomplete creates Account, AccountAttribute and AccountMapping when an enrollment moves to master (Rambabu, group call). The loader must check before inserting, or we get **duplicate AccountMapping** rows (Rambabu's warning, 01:04:30 to 01:05:06).

## 7. Validations
Phase 1 is what Dilip asked for first (1:1 call, 00:04:18 and 00:15:05):
- **Product:** `ProductId` exists for the client.
- **Components:** every component in `ComponentAggregatorJson` exists. There is no direct ComponentId column, so the loader parses the JSON. No duplicate ComponentIds.
- **Attributes:** values follow our formula and attribute rules.

The production rules the API already enforces come along automatically if we go through the API (from the RateEngine controllers):
- **Contract:** ContractNumber must be unique ("A Contract with this number already exists.").
- **ContractDetail:**
  - the Contract must exist;
  - SubAgreementNumber must be unique;
  - no other detail with the same effective start/end;
  - the dates must fall within the contract's dates, unless the detail is a Rollover;
  - the components must exist and must not repeat;
  - the formula must be valid.
- **Account:** AccountNumber must be unique; state, ISO and utility must be valid.
- **AccountAttribute:** the account must exist; no attribute with the same name in an overlapping date range; the formula must be valid.
- **AccountMapping:** the account, the contract detail and the product must all exist; contract-detail validation runs.

Pre-validation in the loader (cheap, before any API call): required fields, data types, enum values, start ≤ end, and parsing of the JSON columns. Dilip suggested using Claude to pull the full rule list out of the code (1:1 call, 00:15:10).

## 8. Loader algorithm (per run)
1. Claim the rows: `UPDATE TOP (n) … SET LoadStatus = 'Processing' OUTPUT … WHERE LoadStatus = 'New'`, ordered by BatchId and then table order. This makes the loader safe if two runs overlap.
2. For each row: pre-validate, resolve the natural keys to RE GUIDs, then call the API's Add (or Update).
3. On success: set `Loaded`, store `RateEngineId`, and resolve older errors for the same key.
4. On failure: set `Error` and insert into `<Table>_Error` with the API's `ErrorMessage` list.
5. Log a run summary per batch. Optionally email or Teams-notify on errors (later).

## 9. Build and test plan
1. **Local DB:** copy about the top 100 rows of each table from `Vxretail_Data_Migration_Engie`. Don't test against the client DB, because Engie inserts and deletes there (Dilip, 1:1 call 00:05:58 to 00:06:32). Point at a QA Rate Engine.
2. **DDL:** add the control columns, the 5 error tables and the result view.
3. **Loader skeleton:** config (connection strings, API base URL, service account, batch size, client), table order, claim/process/log.
4. **Validators:** Phase 1 rules, then wire in the API errors.
5. **Test cases:**
   - happy path;
   - missing product or component;
   - duplicate contract number;
   - overlapping dates;
   - an account that already exists from enrollment autocomplete;
   - a parent that failed;
   - fix-and-resubmit.
6. **Check after a QA load:** the replication messages reach VXR Retail, and no duplicate AccountMapping rows exist.

## 9b. Alternative: SQL Agent job + stored procedure (Arulappan's proposal, 17 Nov)
The VXR side keeps replica tables in each marketer DB: dbo.Component, Product, Contract, ContractDetail and AccountMapping. They are filled by `dbo.UbRateEngineReplicaUpsert`, which takes JSON, through `RateEngineReplicaWriter.vb`. That procedure is called by the `RateEngineReplicationConsumer`, which reads from RabbitMQ.

The stored procedure therefore works **if it does three extra things**:
1. **Set-based validation in T-SQL.** Flag the bad rows, write them to `<Table>_Error`, then MERGE only the clean rows into the Rate Engine DB (linked server), in parent → child order.
2. **Feed the VXR replica as well.** Build the replica JSON for the rows it loaded (`FOR JSON PATH`, same shape as `RateEngineReplicationMessage`) and call `dbo.UbRateEngineReplicaUpsert` in the marketer's VXR DB. Without this step, VXR never sees the contracts and mappings.
3. **Accept the gaps, or close them:**
   - Rules that live in C# and are hard to repeat in T-SQL (formula validation) need to be repeated or skipped.
   - Unbilled-usage recalculation does not run. That is fine for brand-new accounts with no usage; it is a problem for changes to accounts that already have usage.
   - Two linked servers. Without a distributed transaction, the Rate Engine and the replica can drift, so track a per-row `ReplicaStatus`.

## 9c. The enrollment bulk import (CustomerDataImport.vb) and what it means for Plan Z
- Correction, verified 17 Nov. IsRateEngineAccount is set by:
  - the **API enrollment** (`Integrations/.../CustomerService.cs:785`), as "ProductId is not blank";
  - **API meter add/update** (`MeterService.SyncRateEngineAccountFlag`), for both inbound and master records;
  - the **Service Location page** checkbox (`CusSvcLocationsD.aspx.vb:4414`); turning it on there publishes the account to the rate engine (around line 5399);
  - the **New Enrollment page** checkbox (`NewCusEnroll.aspx.vb:6572`).
  `AutoEnroll.vb:1596` copies the flag from inbound to master.
- The **file import (CustomerDataImport.vb) is the only enrollment path that never sets it**, even when the product validates. Suggested fix: mirror the API rule (flag = product present and validated). Autocomplete publishes the account to the rate engine only when the flag is on (`AutoEnroll.vb:1061`).
  - So imported Engie enrollments create **no** Rate Engine Account, AccountAttribute or AccountMapping.
  - The same flag also drives rate-engine billing (`Stage2MeterReadings`, `StandardEnergyCharge`, `BillRegEdit`).
- **Decision needed: who owns Account and AccountMapping in the Rate Engine?**
  - **(a) The loader owns them.** The import sets IsRateEngineAccount without publishing, or the publish is suppressed for these rows.
  - **(b) Autocomplete owns them.** The flag is set and autocomplete publishes; the loader then loads Contract and ContractDetail only, and maps them by sub-agreement number. This avoids duplicate mappings.
- Import errors don't block a row: it is still written as Pending with "Import Error: …" in StatusMessage. There is no error file and no per-row batch ID.

## 10. Open questions (take to Dilip / AJ)
- Do we use the existing DM DB or a dedicated staging DB with only the 5 tables? Who creates it, and on which server?
- If a natural key already exists with different values: update, or error?
- Does Engie supply its own GUIDs in staging? If so, we keep a staging → RE key map, or resolve by natural key only (recommended).
- The schedule (every 15 minutes?) and the service account for the API.
- The long-term ask (Dilip, 1:1 call 00:16:28): should this become a permanent bulk-import feature for any client, like the enrollment import, not just an interim one?
- The future-dated enrollments are still not designed. That is a separate call.
