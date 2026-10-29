Port the Billing Group Display Name feature (VXR-10280 / dev sub-task VXR-11542) onto releases/09.10.2026 in three repos. Work under C:\Arul\Source\UBCollection. For each repo: create a new branch from the latest origin/releases/09.10.2026, port the change, build it, commit, and stop before pushing. I'll review it and raise the PRs (target: releases/09.10.2026) myself.

## What the feature is (already in QA)
- Billing Group Name stays required and unique. A new optional, non-unique Display Name is printed on invoices and correspondence when the System Setting "Enable Billing Group Display Name" is on.
- VXretail: a Display Name field on the Billing Group page, the System Settings toggle, update history ("Display Name (Billing Group)"), and the {@CustomerName} email placeholder.
- DB: new function dbo.UbGetBillingGroupDisplayName. UbInvoiceEngieMain.BillGroupName uses it, and UbUpdateHistoryExport includes the Display Name. A post-deploy script turns the setting on for ENGIE 950/956/960 only when it isn't already set.
- Web API: DisplayName on BillingGroups Add/Update/Get/Find, 422 when the setting is off, and a new validation rule (ruleset 45, position 26).
- Demo feedback: a Display Name equal to the Billing Group Name is kept as entered.

## Source → target
| Repo | Source (what QA has) | Target |
|---|---|---|
| UtiliBill Web Application | origin/release, PR 17769 = merge 55e1a09c15 (commits 11c2ea7801, 12e810f326) | releases/09.10.2026 |
| UtiliBillDatabaseSchema | origin/release, PR 17770 = merge 63cc0f936 (commits 1a1a87475, f54e6f4ac) | releases/09.10.2026 |
| Utilibill Restful WebAPI | origin/qa, PR 17771 = merge 11762a92d (commits 0f3e6e691, 0dfc1007c, 301394d9f, 1b4bfa00e) **plus the follow-up PR 18102 = merge dc65d0ad8 (VXR-12017, PUT/PATCH/POST in sync with the UI)** | releases/09.10.2026 |

Use `git diff <merge>^1 <merge>` to get each PR's net change. Don't cherry-pick merge commits blindly, because the branches have diverged.

## Step 0: check for an existing port (do this first, then STOP and report)
Commits labelled ENGI-548 already port the WebApp and DB parts to 09.10 on `origin/feature/ENGI-P1-batch2-10232026` (WebApp 3488efd1f3, DB 5e871d5aa, by another developer, not yet in releases/09.10.2026). After `git fetch`, report:
1. Whether those commits, or their content (look for dbo/Functions/UbGetBillingGroupDisplayName.sql and the "Enable Billing Group Display Name" setting), are now in origin/releases/09.10.2026.
2. Whether there's any API-side port (search all remote branches for "11542", "12017", "ENGI-548").
Wait for my go-ahead before continuing. If the ENGI branch hasn't merged, use its commits as a reference for how the 09.10 context differs, but port from the original VXR PRs.

## Known differences on releases/09.10.2026 (check each one, don't assume)
**WebApp**
- On 09.10 the Billing Group page is `webroot/Config/BillingGroupD.aspx(.vb/.css)`, not `webroot/Customer/...`. Apply the changes there.
- EmailTemplateFormatter.vb, SystemSettings.aspx(.vb) and BillingGroupD.aspx.vb hunks have different surrounding context on 09.10 (no VXR-9908 aggregation block, no Historical Usage/Shaping Curve rows). Place them by hand.
- Some .vb files are not UTF-8. Keep each file's original encoding and line endings, and edit with perl or the Edit tool, not sed. Check `git diff` afterwards to confirm that only the intended lines changed.

**DB**
- `dbo/Stored Procedures/UbInvoiceEngieMain.sql` shows as binary in git (UTF-16 or mixed encoding). Keep its encoding. 09.10's version is quite different from release's (76 KB vs 64 KB), so port only the BillGroupName change, which calls dbo.UbGetBillingGroupDisplayName, and check that it sits in the equivalent place.
- Add the function to UtiliBillDatabaseSchema.sqlproj. Make the post-deploy block in Script.PostDeployment/MainPostDeploymentScript.sql idempotent: insert the setting only if it doesn't already exist.
- UbUpdateHistoryExport.sql: port the Display Name column/union into 09.10's version of the proc.

**Web API (the riskiest part)**
- Rules migrations on 09.10 live in `DataAccessLayer/RulesEngine/Migrations/`, not `DataAccessLayer/Migrations/`. The latest one is 20261028033446_VXR-11675.
- **RuleId 175107 is already used on 09.10** by a different rule. Don't copy the qa migration's RuleId. Find the highest RuleId on 09.10 (in the model snapshot plus all migrations), take the next free one, and confirm that ruleset 45 position 26 is free on 09.10.
- Don't copy the 26k-line Designer.cs or snapshot from qa. Make the RulesEngineDBContext seed change on 09.10, then generate a new migration with `dotnet ef migrations add "VXR-11542 BillingGroup DisplayName rule"` (use the correct project/startup project and context for RulesEngineDBContext). The migration file it creates should contain only the one InsertData/DeleteData for the DisplayName rule. If it contains anything else, stop and show me.
- 09.10 already has DisplayName logic in BillingGroupService.cs: `CommonFunctions.GetDisplayName(...)` from VXR-6062, at around lines 93 and 278. Work out how the qa version (after 17771 and 18102) replaced or wrapped that logic, and reproduce the final qa behaviour. Don't end up with both versions.
- Port the CommonFunctionsTests additions too.

## Branches, build, commit
- Branch name in each repo: `feature/VXR-11542-releases-09.10.2026`, created from a freshly fetched `origin/releases/09.10.2026`.
- Build each solution in Release (MSBuild for WebApp/DB; `dotnet build` and `dotnet test --filter CommonFunctionsTests` for the API). Report any failures with the output. Don't skip them.
- One commit per repo. The message should start with `VXR-11542 - Port Billing Group Display Name (non unique) to releases/09.10.2026` and the body should list the source PR numbers and every place where the port departs from the source (paths, RuleId, migration name, hand-placed hunks).
- Don't push or create PRs until I approve.

## Report back
For each repo: the branch, the commit hash, files changed, the differences from the source PR and why, build/test results, and anything you weren't sure about. Also confirm the release order: DB → Web API (rule migration) → VXretail.
