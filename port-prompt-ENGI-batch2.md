Port 8 ENGI tickets from `feature/ENGI-P1-batch2-10232026` to `releases/09.10.2026` in one dedicated PR.

Repo: C:\Arul\Source\UBCollection\UtiliBill Web Application (on-prem TFS/Azure DevOps Git, remote `origin`).

## Scope: these 7 commits only (7 commits cover 8 tickets)

Cherry-pick them in this order (oldest first). Use `git cherry-pick -x` so each commit keeps its message and records where it came from.

| # | Commit | Ticket(s) | Files |
|---|--------|-----------|-------|
| 1 | b5f0e10a3 | ENGI-647 | Invoices/StandardEnergyCharge.vb |
| 2 | 80c58d428 | ENGI-816 | webroot/Billing/PreBillWorkQueue.aspx.vb |
| 3 | da00ef85c | ENGI-805 | CommonClasses/Services/RateEngineApiService.vb, webroot/Billing/PreBillWorkQueue.aspx.vb |
| 4 | f987612d9 | ENGI-587 | webroot/Billing/PreBillWorkQueue.aspx.vb |
| 5 | 2103eef0e | ENGI-573 (its message also mentions ENGI-647) | Invoices/StandardEnergyCharge.vb |
| 6 | 384921d65 | ENGI-680 + ENGI-681 (one fix for both; ENGI-681 has no commit of its own, see the ticket's 10/26 internal note) | Invoices/Invoice.vb |
| 7 | 55c2e5972 | ENGI-797 | Invoices/Invoice.vb |

DO NOT port the other commits on the source branch: ENGI-794/781/556, 717, 790, 814, 548, 572, 792, 493, 819. Do not merge the source branch; cherry-pick only.

## Steps

1. Check that `git status` is clean, then run `git fetch origin`.
2. Create the branch from the latest target:
   `git switch -c feature/ENGI-P1-batch2-port-releases/09.10.2026 origin/releases/09.10.2026`
3. Cherry-pick the 7 commits in the order in the table.
4. Expected conflict, on #6 (384921d65) in `Invoices/Invoice.vb` only. A dry run with `git merge-tree` showed #1-5 and #7 apply cleanly.
   - Cause: the target already has PR 18064 "Meter reading id null check", which made the same `Not d.IsMeterReadingIdNull() AndAlso ...` change to the `For Each invoiceDetail ... .Where(...)` line (around line 4540).
   - Resolution: keep the target's `.Where(...)` line (it is the same as the source's). Apply the rest of the ENGI-680 change:
     a. Remove `Dim billCharges As New List(Of BillCharge)` from before the `For Each meterReadingRow` loop (around line 4524).
     b. Add it as the first statement inside `For Each meterReadingRow As MeterReadingsRow In invDetailsMeterReadings`, with the comment `' bill charges for this meter reading only`.
     c. Optionally take the source's updated comment above the `For Each invoiceDetail` line.
   - Check: `git diff origin/feature/ENGI-P1-batch2-10232026 -- Invoices/Invoice.vb` should be empty or show only lines from target-only PRs (18006/18017/18064/18090/18092/18093). Run `git show 384921d65` to compare with the original.
   - Encoding: Invoice.vb is not UTF-8. Resolve the conflict with the Edit tool or perl. Do not use sed, and do not re-save the file in a different encoding or with different line endings. After resolving, `git diff --stat` must show only a few lines changed in Invoice.vb, not the whole file.
   - Then `git add Invoices/Invoice.vb` and `git cherry-pick --continue`.
   - If any other commit conflicts, stop and show me the conflict. Don't guess.
5. Verify:
   - `git log --oneline origin/releases/09.10.2026..HEAD` shows exactly 7 commits.
   - `git diff origin/releases/09.10.2026 --stat` touches only these 5 files: StandardEnergyCharge.vb, Invoice.vb, PreBillWorkQueue.aspx.vb, RateEngineApiService.vb, and no others.
   - For each touched file, compare the ported hunks with the source branch. Only the 7 commits' changes should differ.
   - Build the solution (MSBuild/VS) and report the result. If a build isn't possible here, say so; don't claim it passed.
6. Before pushing, show me the 7-commit log and the diff stat, and wait for my OK.
7. After my OK, push with `git push -u origin feature/ENGI-P1-batch2-port-releases/09.10.2026` and open a PR into `releases/09.10.2026`:
   - Title: `ENGI-573, 587, 647, 680, 681, 797, 805, 816: Rate Engine / Pre-Bill Work Queue fixes (releases/09.10.2026)`
   - Description: one line per ticket (what changed and which file), a note that ENGI-681 is fixed by the ENGI-680 commit, the Invoice.vb conflict and how it was resolved, and a list of the source commits.
   - Try `az repos pr create` against the TFS remote first. If az isn't authorized for this on-prem server, give me the ready-to-paste title and description plus the create-PR URL:
     http://ecltfssvp.domain.local:8080/tfs/UBCollection/UtiliBill%20Web%20Application/_git/UtiliBill%20Web%20Application/pullrequestcreate?sourceRef=feature/ENGI-P1-batch2-port-releases/09.10.2026&targetRef=releases/09.10.2026

## Rules
- Don't touch my current branch or working tree. Don't force-push. Don't rebase or modify releases/09.10.2026 directly.
- Don't comment on or transition the Jira tickets.
- If a long-path error appears ("Filename too long"), don't create a git worktree. Work in the main checkout on the new branch.
