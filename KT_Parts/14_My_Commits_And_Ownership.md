# My Commits and Ownership Map (author `GS <extern.gs_nisarga@allianz.com>`)

Status: you confirmed that git author **GS** is you. This file replaces every earlier statement in this pack that said "git cannot attribute commits to you". Earlier files were written before that confirmation. Where they say "do not claim commits", read it as "claim only the GS commits listed here, and not other authors' commits".

Source: `git log --all --author="extern.gs_nisarga"` and `git show --stat` on each commit (read-only; nothing was run). Dates are author dates. Branch names come from `git branch -a --contains`.

## 1. Your commits at a glance (11 in total: 8 code, 3 merges)

| # | Commit | Date | Ticket | Module | One-line meaning |
|---|---|---|---|---|---|
| 1 | `b4e588bd4` | 2026-08-11 | none in message | Template12 | `final_T12_code`: renames/cleanups of handlers, PIDM SetB handler change, Template12 base service tweaks. 12 files, +37/-36. On HEAD. |
| 2 | `f79ecc811` | 2026-08-20 | OPUSM-41763 | Template12 | "modified changes for T12": `Template12PolicyDocumentBaseService` +24/-4. |
| 3 | `578778080` | 2026-08-25 | OPUSM-41763 | ASP | First big ASP EndtDocument work: new `PlineASP*` handlers and DTOs (24 files, +5,222). Maps previously unmapped subreport results into the XSD response. Fixes CONTRACT_ID String vs BSON Long mismatch. |
| 4 | `ba9e29586` | 2026-08-26 | OPUSM-41763 | ASP | Fixes ASP EndtDocument/EndtDocument2 mapping and CONTRACT_ID bugs; fixes a wrong query parameter name (500 error) and a null-literal `$project` crash; adds `.mvn/jvm.config` for local build OOM. |
| 5 | `7a925920c` | 2026-08-28 | OPUSM-36054, OPUSM-40884 | ASP | Wires the ASP EndtDocument subreports; **folds AddMemberLandscape/DeletionMemberLandscape into one response**; fixes CONTRACT_ID/VERSION_NO type mismatches; fixes an `AspDebitNoteHandler` Mongo timeout (missing `AggregationOptions`); adds `AspEndtCoverHandler`, `AspEndtScheduleHandler` and DTOs. 9 files, +2,657/-225. |
| 6 | `0d715fbad` | 2026-09-01 | OPUSM-41698 | RejectRisk | Fixes RejectRisk data mapping and aligns with Pline.xsd v0.1.6: correct `CPGE_VI_PARTNERS_ALL` join (`CP_PART_ID`/`CP_PART_VERSION`), `CMGE_CODE` lookups by `CAT_CODE`/`CODE_DESC`, field names, date parsing; nests `rejectRiskType` under `transDet.riskList`; `idType1/idNo1`; ReportHeader `dataClass`; `REMARKS` into `companyDesc`. |
| 7 | `94ea23d81` | 2026-09-21 | none in message | ASP, Template12, RejectRisk | Fixes EndtCover/EndtSchedule mapping gaps (~42 fields computed but never copied to the payload); re-stages Template12 changes; adds `AspServiceTaxInvoiceHandler` (472 lines); restores `Template12PolicyDocument2Service`. 8 files, +1,402/-84. |
| 8 | `cc6a1b84d` | 2026-10-05 | OPUSM-41763 | ASP, RejectRisk | Fixes branch-lookup join depth; **RejectRisk date range silently matching 0 rows when a bound was null**; `invDate` Date vs String mismatch; wrong `DateUtils` import in RejectRiskService; adds REFUND_AMT / TOT_REFUND_AM / INST_IND computation; adds `AspCompanyNameWithGSTRegNoHandler`; **restructures Landscape rows into separate `PlineReportData` entries**; fixes shared-mutable-state bug in `staxPcts()`. 14 files, +771/-241. |
| - | `359d10f9a`, `29de1c85d`, `de2cf813e` | Aug-Sep | - | - | Merge commits (sync with `feature/retrofit-develop-R2-pline`). |

Branches that hold the work:
- On your current branch `feature/retrofit-develop-updated`: `b4e588bd4` only (of the code commits).
- `origin/OPUSM-41763-backup`: `578778080`, `ba9e29586`.
- `origin/OPUSM-41697/...underWritingErrors`, `origin/OPUSM-36035/...template12MemoListChanges`, `origin/OPSUM-44159/...`, `origin/Autorenewal_Document_Updated`: `f79ecc811`, `7a925920c`, `0d715fbad`.
- `origin/OPUSM-41763/retrofit-develop-R2-pline-Template12-ASP-RejectRisk` and `origin/feature/retrofit-develop-R2-pline`: `94ea23d81`; only the first branch holds `cc6a1b84d` (the 2026-10-05 fix).
This matters: your current local branch does **not** contain the final ASP/RejectRisk fixes. That is why your working tree shows the `cc6a1b84d` RejectRisk change as uncommitted edits.

## 2. What this proves for each module (safe interview claims)

### ASP (most of your work)
- You built the ASP EndtDocument subreport wiring: handlers, DTOs and mapping into the XSD-shaped response (`578778080`, `7a925920c`, `94ea23d81`, `cc6a1b84d`).
- You found and fixed the **CONTRACT_ID type mismatch** (String against BSON Long) that silently returned zero rows (`578778080`, `ba9e29586`, `7a925920c`). Proof: commit messages say "silently returning zero rows" and "silent empty results".
- You merged AddMemberLandscape and DeletionMemberLandscape into the EndtDocument response because no separate bean is registered for those report names (`7a925920c`). In `cc6a1b84d` you restructured landscape rows into separate `PlineReportData` entries (one reportData per subreport, per the XSD convention).
- You fixed a Mongo aggregation timeout in `AspDebitNoteHandler` (missing `AggregationOptions`; added `CONTRACT_ID` to the match for selectivity) (`7a925920c`).
- You fixed a wrong query parameter name that caused a 500 on the company BRN call, and a null-literal `$project` crash (`ba9e29586`).
- You noticed ~42 EndtSchedule fields that the handler computed but the service never copied onto the payload, and fixed that (`94ea23d81`).
- You wrote `AspServiceTaxInvoiceHandler` (service-tax invoice, master TIN masking, STAX count) and `AspCompanyNameWithGSTRegNoHandler` (ASP-specific, because the shared handler concatenated GST incorrectly) (`94ea23d81`, `cc6a1b84d`).
- You fixed a shared-mutable-state bug in `staxPcts()` across `transDet` rows (`cc6a1b84d`).

### RejectRisk
- `0d715fbad` (OPUSM-41698): rebuilt the data mapping after finding that the join key (`CUSTOMER_ID`) did not exist, that `CMGE_CODE` lookups used the wrong field names (`MODULE_CODE`/`DESCRIPTION` instead of `CAT_CODE`/`CODE_DESC`), and aligned the output to Pline.xsd v0.1.6 using the mapping sheet.
- `cc6a1b84d`: fixed the null-bound date filter bug and the wrong `DateUtils` package. The `RejectRiskHandler` +1-day `$lt` upper bound and `$setWindowFields` row number in your working tree match this commit.

### Template12
- `b4e588bd4` (on HEAD) and `f79ecc811`: Template12 base service changes and handler renames. `94ea23d81` restaged Template12 changes and restored `Template12PolicyDocument2Service` to match origin after a broken `buildTransactions(List.of(rd), ...)` call.
- Evidence of your Template12 work is thinner than for ASP in this git history. The merge of PolicyDocument2/Landscape into one bean is commit `c0c6d449a` by Amarnath Yadav ("Added Template12 dynamic merging"), **not yours**. Do not claim it. Many Template12 handlers came from other authors (`d4575fa5b`, `7399d2fc1`, `bb82a9c47`, `8d4ecc991` by Shinge/Karunakar). Check with your own memory what you did beyond the three commits above, and fill in `[CONFIRM]`.

### Not yours (do not claim)
- The XSD authoring itself: `Pline.xsd`/`Common.xsd` were supplied (the XSD header cites a client-supplied report list and the master mapping sheet). Your XSD work, per your Jira scope, was **aligning the Java payload and mapping to the XSD**; `6961abbf9` (payload-to-XSD alignment, 119 files) and `d4575fa5b` are by Shinge, not you. Your XSD-related commits are `0d715fbad` ("align with Pline.xsd v0.1.6") and the mapping work in the ASP commits.
- The test-generation commits for OPUSM-44159 (`3f2157b28` by Shaw, `db7588897` by Karunakar, `0ab953cf8`, Neethu's handler tests). Your Jira scope says you did unit tests and test-case generation for all three; if so, find your own test commits. None of the 11 GS commits is a pure test commit, so confirm where your tests are (perhaps under other commit messages or on a branch not scanned) and fill in `[CONFIRM]`.
- The Spring Boot 4.1.1 upgrade, CVE pins, Sonar fixes, `PlineReportNames`, `mainReportName/subReportName` (Shinge, `e74bc0d5c`).

## 3. Honest gaps to resolve with your own memory
1. **Unit tests:** the Jira scope lists test-case generation for all three reports, but no GS commit is a test commit. Where are your tests? (`git log --all --author=gs_nisarga -- src/test`: run it; it may show nothing.)
2. **XSD changes:** your scope lists XSD changes. Be precise: you changed Java payload mapping to match Pline.xsd; the XSD file itself came from the architects/client. Did you propose any XSD edits (e.g. new leaves)? Check the mapping workbooks: `C:\Users\EPGRVFY\Documents\Pline_MASTER_MAPPING_SHEET_v0.1 (final1) 1 5.xlsx` and `...\T12_final_code\RejectRisk\RejectRisk\RejectRisk_XSD_Mapping_sheet.xlsx` (the second folder name suggests you worked on it).
3. **Template12 analysis depth:** you analysed PolicyDocument, PolicyDocument2 and Landscape subreports at all levels. The Jasper dump and the `02_Template12_Deep_Dive.md` traceability table are your analysis cross-check.
4. **Current branch:** `feature/retrofit-develop-updated` lacks `cc6a1b84d`. Decide: cherry-pick `cc6a1b84d` instead of re-typing the RejectRisk fix by hand (run `git cherry-pick cc6a1b84d` on a clean tree; expect conflicts in the ASP files). Do not commit the Mongo credentials in `application.properties`.
5. **Process facts to fill in:** review feedback, UAT defects, who raised OPUSM-41763/41698/36054/40884, release dates. The pack cannot know these.

## 4. Ready-to-use first-person claims (all backed by the commits above)
- "On ASP, I wired the EndtDocument subreports end to end: handlers, DTOs and mapping into the XSD-shaped response, and I folded the add- and delete-member landscape reports into that response, because there was no separate bean for those report names."
- "A key bug I found was a type mismatch: CONTRACT_ID is stored as a number in Mongo but we matched it as a string, so several lookups silently returned zero rows and the tax-invoice and company sections never appeared. I converted the value to a number in the handlers and the shared base handler."
- "I fixed a Mongo timeout in the ASP debit-note handler by adding aggregation options and adding the contract id to the match, following the pattern already used in the ACP debit-note handler."
- "I found about 42 fields on the endorsement schedule that the handler computed correctly but the service never copied to the payload, and mapped them."
- "On RejectRisk, I rebuilt the mapping: the join key and code lookups were wrong, I aligned it to the XSD per the mapping sheet, and later fixed a date-range bug where a null bound matched zero rows."
- "I wrote ASP-specific service-tax and company-name handlers because the shared handlers produced incorrect GST values for ASP."
For each, be ready for the follow-ups in `07b_Ownership_Answers_And_STAR_Stories.md` and the trap questions `11*` (rewrite any answer there that says "I cannot claim commits" to use this table).
