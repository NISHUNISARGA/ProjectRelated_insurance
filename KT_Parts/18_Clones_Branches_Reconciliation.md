# 18 - Clones, Branches and Reconciliation (evidence-based)

Scope: opus-documentprintdata-service, modules Template12, ASP (AspEndtDocument*), RejectRisk.
Method: read-only git (`log/show/branch/status/diff/rev-parse/merge-base/ls-tree/cat-file/cherry/rev-list`), `sha1sum`, `unzip -l/-p` (no extraction into clones). Evidence date: 2026-10-14.
Labels: **[FACT]** = observed command output; **[INFERENCE]** = my reading of the facts. Secrets are shown as `<redacted>`.

Short names used below:
- **FINAL** = `Plines_Base_code\New_branch\2\opus-documentprintdata-service`
- **T12FC** = `Plines_Base_code\T12_final_code\opus-documentprintdata-service`
- **SBU** = `Plines_Base_code\new_springboot_upgraded\opus-documentprintdata-service`
- **NB** = `Plines_Base_code\New_branch\opus-documentprintdata-service`

---

## 1. Per-clone state

| Clone | Branch | HEAD | HEAD author | Boot parent | `git status --short` | Remote branches |
|---|---|---|---|---|---|---|
| FINAL | feature/retrofit-develop-updated | 861f0666c 2026-10-08 "OPUSM-44159 Added report, sub report name in listing report" | Shinge | **4.1.1** | 3 entries, all ` M`: `RejectRiskHandler.java`, `RejectRiskService.java`, `application.properties` | ~55 incl. origin/feature/retrofit-develop-updated, OPUSM-41763/..., OPUSM-41763-backup, OPUSM-36035/..., OPUSM-41697/..., many OPUSM-44159/*. Local HEAD is **0 ahead / 3 behind** origin/feature/retrofit-develop-updated (d44d0f9fa, ecf62bdd4, 4ee0722ea, all 2026-10-08, not by GS). |
| T12FC | feature/retrofit-develop-R2-pline | de2cf813e 2026-09-21 "Merge branch 'feature/retrofit-develop-R2-pline' of ... into feature/retrofit-develop-R2-pline" | **GS** | **3.4.11** | 5 entries: ` M AspEndtDocumentService.java` (+11/-3) and 4 untracked dirs `.postman/`, `_jrxml_audit_tmp/`, `_jrxml_audit_tmp2/` (jrxml scratch), `postman/` | ~45; local branches: OPUSM-41763-backup, OPUSM-41763/retrofit-develop-R2-pline-Template12-ASP-RejectRisk, feature/retrofit-develop-R2-pline, retrofit-develop-R1 |
| SBU | feature/pline-spring-boot-4-upgrade-R2-poc | 245c44a4 2026-09-22 "OPUSM-44159 multiple report data changes" | Shinge | **4.1.1** | **clean (0)** | ~45; has a **local branch OPUSM-41763/retrofit-develop-R2-pline-Template12-ASP-RejectRisk = cc6a1b84 (GS, 2026-10-05)**. `git config user.name` here is `GS <extern.gs_nisarga@allianz.com>` (the only clone with the corporate identity). Extra folders: `ASP\` (sample XML contracts), `RejectRisk\`, `Template12\` (empty), `prompt1-3.txt` (prompt3 dated 2026-10-13). |
| NB | feature/retrofit-develop-updated | e74bc0d5c 2026-10-07 "OPUSM-44159 Added report, sub report name in resposne" | Shinge | **4.1.1** | 1 entry: ` M application.properties` (same Mongo lines as FINAL, see 8.3) | ~48. NB is exactly **one commit behind FINAL** (861f0666c is its child). |
| BC1 | feature/retrofit-develop-R2-plines | 5f9c930 2026-06-09 "Merge pull request #184 from OPUSCCC/feature-riskaccgrp" | extern.parihar_aditya | 3.4.11 | 250: 188 `A ` staged, 54 `AM`, 3 `MM`, 1 `AD`, 1 ` M`, 3 `??` (plines package staged but never committed; `.claude/settings.local.json` modified) | 21 |
| BC2_local_work | feature/retrofit-develop-R2-plines | 5f9c930 2026-06-09 (same as BC1) | extern.parihar_aditya | 3.4.11 | 408: 246 `A `, 153 `??`, 9 ` M` | 40 |
| For_G_and_CR | feature/retrofit-develop-R2-plines | 8d1b537 2026-07-06 "Add Plines module implementation with handlers, services, DTOs and bug fixes" | Shinge | 3.4.11 | 127: 98 ` M`, 9 `AM`, 9 `??`, 6 `RM`, 3 `D `, 2 `A ` | 35 |
| Git_pulled | feature/retrofit-develop-R2-plines | 8d1b537 2026-07-06 (same) | Shinge | 3.4.11 | 208: 79 ` M`, 126 `??`, 3 ` D` | 37; also `opus-documentprintdata-service.zip` (2026-07-29) |
| T12_merge_code (not a git repo at top) | contains `Plines_T12_Merged_code\opus-documentprintdata-service` and `original_code\Plines_T12_Merged_code\opus-documentprintdata-service`, both git repos | both on feature/retrofit-develop-R2-plines @ 8d1b537 2026-07-06 | Shinge | 3.4.11 | 435 / 434 entries | n/a; zip `Plines_T12_Merged_code.zip` 2026-07-30 |

Last commit date on ANY ref per clone: FINAL 2026-10-08; BC1 2026-06-09; BC2 2026-07-06; For_G_and_CR 2026-07-08; Git_pulled 2026-07-06.
Remote URL of all clones: the same Allianz GitHub Enterprise `OPUSCCC/opus-documentprintdata-service` (credentials not shown).

[FACT] Local git identity on FINAL, T12FC, NB, BC1, BC2 is `NISHUNISARGA <nisarga.g.s568@gmail.com>` (also global). Only SBU is configured as GS/corporate. No commit in any clone is authored by the NISHUNISARGA/gmail identity.

---

## 2. GS commit inventory

### 2.1 Union of commits by author `extern.gs_nisarga@allianz.com` (name `GS`), all refs, all 4 working clones = **11 commits**
BC1, BC2, For_G_and_CR, Git_pulled have **0** GS commits (they stop in June/July before GS started pushing).

| # | Hash | Date (IST) | Ticket | Subject | In FINAL HEAD? | Which refs carry it (FINAL view) |
|---|---|---|---|---|---|---|
| 1 | b4e588bd4 | 2026-08-11 21:57 | - | final_T12_code | **yes** | feature/retrofit-develop-updated + many |
| 2 | 359d10f9a | 2026-08-11 22:00 | - | Merge remote-tracking branch 'origin/feature/retrofit-develop-R2-pline' into feature/retrofit-develop-R2-pline | **yes** | same |
| 3 | f79ecc811 | 2026-08-20 17:43 | OPUSM-41763 | modified changes for T12 | **no** | feature/retrofit-develop-R2-pline, OPUSM-36035/..., OPUSM-41697/..., Autorenewal_Document_Updated, OPUSM-41763/... |
| 4 | 29de1c85d | 2026-08-20 17:55 | - | Merge branch 'feature/retrofit-develop-R2-pline' of https://github.developer.allianz.io/OPUSCCC/... into feature/retrofit-develop-R2-pline | no | same as #3 |
| 5 | 578778080 | 2026-08-25 14:47 | OPUSM-41763 | ASP EndtDocument field-mapping and CONTRACT_ID type-mismatch fixes | no | **only origin/OPUSM-41763-backup** |
| 6 | ba9e29586 | 2026-08-26 09:56 | OPUSM-41763 | Fix ASP EndtDocument/EndtDocument2 data mapping and CONTRACT_ID type bugs | no | **only origin/OPUSM-41763-backup** (tip of that branch) |
| 7 | 7a925920c | 2026-08-28 00:10 | OPUSM-36054 + OPUSM-40884 | Wire ASP EndtDocument sub-reports and fold Landscape reports into one response | no | feature/retrofit-develop-R2-pline and downstream |
| 8 | 0d715fbad | 2026-09-01 21:42 | OPUSM-41698 | Fix RejectRisk data mapping and align with Pline.xsd v0.1.6 | no | feature/retrofit-develop-R2-pline and downstream |
| 9 | 94ea23d81 | 2026-09-21 12:31 | (no id in subject) | ASP EndtDocument: fix EndtCover/EndtSchedule field mapping gaps, restore Template12PolicyDocument2Service | no | feature/retrofit-develop-R2-pline, OPUSM-41763/... |
| 10 | de2cf813e | 2026-09-21 12:33 | - | Merge branch 'feature/retrofit-develop-R2-pline' of ... (pull merge) | no | tip of origin/feature/retrofit-develop-R2-pline and T12FC HEAD |
| 11 | cc6a1b84d | 2026-10-05 22:59 | OPUSM-41763 | ASP RejectRisk/EndtSchedule/EndtDocument fixes | **no** | **only origin/OPUSM-41763/retrofit-develop-R2-pline-Template12-ASP-RejectRisk** (tip), present in SBU as local branch; **absent from T12FC** |

Per clone counts: FINAL 11, T12FC 10 (lacks #11), SBU 11, NB 11. Ancestry check (`merge-base --is-ancestor <h> HEAD` in FINAL): only #1 and #2 are ancestors of FINAL HEAD.

Observations:
- [FACT] Earliest GS commit in any clone is 2026-08-11, not 12 May 2026. [INFERENCE] The May-Aug 2026 work (if any) was either never committed under this identity, was done under a different identity (e.g. the gmail identity configured locally, or committed by someone else via squash/PR), or lives in repos not examined.
- [FACT] Only 4 merge commits (#2, #4, #10) are "pull merges" created by `git pull` (message pattern "Merge branch ... of https://..."); GS has no PR-merge commits (no "Merge pull request #n" authored by GS).
- [FACT] No commit in the repo carries any ticket id OPUSM-36035 or OPUSM-41697 by GS (those are other authors: Karunakar, Varsha).

### 2.2 Other identities of the same person
Searched authors/committers (case-insensitive) for `nisarga`, `gs_`, `EPGRVFY`, `^GS`, `shinge` across all refs in all 8 clones:
- Only identity: `GS <extern.gs_nisarga@allianz.com>` (11 author, 11 committer).
- `Shinge <extern.shinge_pravin@allianz.com>` / `extern-shinge-pravin` is a **different** person's identity (HEAD of FINAL/NB/SBU is authored by Shinge). I matched it only because the search term `shinge` was broad. [INFERENCE] Shinge is a teammate who owns the OPUSM-44159 Boot-4 line; do not claim Shinge's commits as GS's.
- No `EPGRVFY`, `NISHUNISARGA` or gmail-address commits exist. [INFERENCE] If you ever commit from FINAL with current local config, the author will be `NISHUNISARGA <nisarga.g.s568@gmail.com>`, which will not look like your earlier GS commits. Fix with `git config user.name GS` / `user.email extern.gs_nisarga@allianz.com` in that clone before the next commit.

### 2.3 GS commits touching src/test
[FACT] **None.** `git log --all --author=gs_nisarga -- src/test` returned nothing in all 8 clones, all refs. (src/test commit totals: FINAL 1140, NB 1079, SBU 760, T12FC 697, BC1 116, BC2 187, For_G_and_CR 197, Git_pulled 187; none by GS.)

---

## 3. Which clone holds the NEWEST code for each module

Key principle [FACT]: FINAL (Boot-4 line, Shinge's OPUSM-44159) and the GS line (Boot-3.4 line: feature/retrofit-develop-R2-pline -> OPUSM-41763/...) **diverged**. Merge base of FINAL HEAD and cc6a1b84 is 4b003f899; `rev-list --left-right --count HEAD...cc6a1b84` = **1363 / 194**. `git cherry HEAD cc6a1b84` marks GS commits #3-#11 as `+` (not present in FINAL by patch).

### 3.1 Hash/line comparison of the 13 module files (normalised, CRLF stripped; sha1 first 8)
| File | FINAL worktree (lines) | cc6a1b84 (GS 10/05, lines) | T12FC worktree | SBU zip (09/27) |
|---|---|---|---|---|
| RejectRiskHandler | 91029345 (169) | 856f44d3 (171) | 89a1485c | = cc6 |
| RejectRiskService | 84120bb2 (160) | 79292dbd (150) | ac8d116a | = cc6 |
| AspEndtDocumentService | 3e6fbad9 (1701 in HEAD) | 503c9ad0 (1654) | 5ff600ad (1500) | 93930139 (1611) |
| AspEndtDocument2Service | ad79712f | 1285bd1a | = cc6 | = cc6 |
| AspEndtDocumentLandscapeService | 82eadfeb | e6e466c7 | 0da0014a | = cc6 |
| AspEndtScheduleHandler | 94fb0d5c (840) | 4ecf9653 (**1115**) | = FINAL | b1655961 |
| AspEndtCoverHandler | e547995e (213) | a7ef9e4c (236) | = FINAL | 2a6ce9e0 |
| AspEndtDocumentMainHandler | a2dca088 (158) | 7560c917 (114) | = FINAL | = cc6 |
| **AspCompanyNameWithGSTRegNoHandler** | **file does not exist** | b299ec2e (new file, 83 lines) | absent | present (= cc6) |
| Template12PolicyDocumentBaseService | ebf6c58b (3732) | 7b3ab147 (3732) | = cc6 | = cc6 |
| Template12PolicyDocument2Service | c16735fe (300) | fe3532c0 (299) | = cc6 | = cc6 |
| Template12PolicyDocumentService | fe7a7047 | f73919ad | = cc6 | = cc6 |
| Template12LandscapeService | 9c207bf6 | 4cf2cc6f | = cc6 | = cc6 |

Tracked-file hash diff between FINAL and NB over 59 module files: NB differs only in 10 files (RejectRisk*, AspEndtDocument*, Template12*, AspServiceTaxInvoiceHandler) because NB is one commit behind (Shinge's 10/08 change).

### 3.2 Function-level differences (public/private method names; HEAD vs cc6a1b84)
**RejectRisk**
- Method names identical (service: 3, handler: 2). Differences are behavioural:
  - FINAL HEAD (Shinge) has what cc6 lacks: `PlineReportBlocks.reportHeader(...)` global ReportHeader data block, `PlineReportNames.tag(...)`, `MAIN_REPORT_NAME`. These are the report/sub-report-name features of OPUSM-44159 (newest, 10/06-10/08).
  - cc6 (GS) has what FINAL HEAD lacks: date-range fix in `RejectRiskHandler` (inclusive-day upper bound via `$lt` day+1, only add bounds actually supplied, remove the hard-coded `ROW_NUMBER` `$addFields`).
  - [FACT] FINAL's **uncommitted** change in RejectRiskHandler is that same GS date-filter fix (diff vs cc6 is comments only), and in RejectRiskService it swaps `motor.utils.DateUtils` -> `plines.utils.DateUtils`. [INFERENCE] You re-applied your RejectRisk fix by hand in FINAL; it is not committed or pushed anywhere on the Boot-4 line.
  - T12FC and SBU-zip RejectRisk lack Shinge's ReportHeader blocks.
- Winner: **FINAL worktree** (HEAD + uncommitted date fix) is the superset for RejectRisk. Nothing GS-authored is lost provided the uncommitted diff is committed.

**ASP (AspEndt*)**
- AspEndtScheduleHandler: cc6 has 5 methods FINAL lacks: `computeShowGstLabel`, `enrichFeeDiffs`, `fetchCoverLimitSum`, `fetchFeeRows`, `sumByCode` (1115 vs 840 lines). **cc6 ahead.**
- AspEndtDocumentMainHandler: cc6 has `fetchEndtCode`, `fetchProductCode`; FINAL has `filterArray`, `firstElem`, `listOf` that cc6 dropped (158 vs 114 lines). Divergent.
- AspEndtDocumentService: cc6 has `snapshotEndorsement`, `toLandscapeReportData`; FINAL has `landscapeRow`, `sectionReportData`. cc6 sets endorsement fields `setEndtImaFee/EndtDoubleIndemnity/EndtStaxAmt/EndtLlbPrem/EndtStiPrem` and `det.setContractId(dto.getContractId())` in 5 places; FINAL HEAD has **0** `setEndtImaFee` and only 2 `setContractId(dto...)`. **cc6 ahead on field mapping**, FINAL ahead on Shinge's report-name/section structure (last touch 2026-10-08).
- **AspCompanyNameWithGSTRegNoHandler** (new in cc6, referenced in 3 files of cc6: its own file, AspEndtDocumentService, and one more): **does not exist in FINAL, T12FC, or SBU working tree.** It exists only in commit cc6a1b84 (branch origin/OPUSM-41763/...) and inside `new_springboot_upgraded\opus-documentprintdata-service.zip` (09/27) plus a compiled `.class` in SBU `target\classes`. [FACT]
- AspEndtCoverHandler: same method names; cc6 23 lines longer (236 vs 213), FINAL HEAD last touched 09-29 by Shaw (tests/coverage).
- AspEndtDocument2Service, AspEndtDocumentLandscapeService: same method names, content differs (hash).
- T12FC's uncommitted AspEndtDocumentService (+11/-3): adds `det.setContractId(dto.getContractId())` x3 and `endorsementDetails.setEndt{DoubleIndemnity,ImaFee,StaxAmt,LlbPrem,StiPrem}`. This equals the content of `T12_final_code\opus-documentprintdata-service_final_work.zip` (09/27 12:14, sha1 5ff600ad). cc6a1b84 (10/05) already includes these same field-mapping ideas in a later form (cc6 5 hits vs T12FC worktree 5 hits for setContractId), so T12FC worktree is an **intermediate** version, superseded by cc6.

**Template12**
- Method names in all 5 Template12 services identical (BaseService 124 methods in both).
- Content: FINAL HEAD is **newer** (Boot-4 / OPUSM-44159 XSD restructuring): `PlineProviso/ProvisoList` instead of `PlineMemo/MemoList` for clause rows, and `prntDetails.companyType.companyDesc` instead of `prntDetails.transDetList.companyType.companyDesc`. Last touched 2026-10-06/07 (Shinge). T12FC, SBU-zip and cc6 carry the older Memo-based layout (Template12PolicyDocumentBaseService last GS-line change 2026-09-21).
- Winner: **FINAL**. Do not overwrite FINAL's Template12 files with any other clone's.
- GS's T12 work (commits #1, #2 on Aug 11, "final_T12_code") is already in FINAL HEAD. Later T12 changes on the Boot-3 line were by Karunakar (OPUSM-36035) and Neethu (OPUSM-44159 Template2-12), not GS.

### 3.3 Summary: where is each clone ahead?
| Item | Newest at | Status in FINAL |
|---|---|---|
| RejectRisk report header / report-name / subReportName | FINAL HEAD (Shinge) | has it |
| RejectRisk date-range filter (GS) | FINAL worktree (uncommitted) and cc6a1b84 | **uncommitted - commit it** |
| ASP EndtSchedule extra fee/cover-limit logic (5 methods) | cc6a1b84 (origin/OPUSM-41763/...) | **missing** |
| ASP EndtDocument snapshotEndorsement/toLandscapeReportData + Endt* field mapping + contractId | cc6a1b84 | **missing** |
| AspCompanyNameWithGSTRegNoHandler (new file) | cc6a1b84, SBU zip | **missing** |
| AspEndtDocumentMainHandler fetchEndtCode/fetchProductCode | cc6a1b84 | **missing** |
| ASP sections/report names (landscapeRow, sectionReportData) | FINAL | has it (not in cc6) |
| Template12 Proviso + companyType restructure | FINAL | has it |
| 7 other GS commits (#3-#10) | feature/retrofit-develop-R2-pline, OPUSM-41763-backup | not in FINAL ancestry (content partly superseded) |
| Test code | FINAL (1932 test files) | has it |

[INFERENCE] The Boot-4 line was cut from feature/retrofit-develop-R2-pline right after GS's Aug 11 "final_T12_code" and never received your later ASP/RejectRisk work; Shinge's OPUSM-44159 work then re-shaped many files, so GS's later commits do not apply cleanly (expect conflicts in the ASP files).

---

## 4. Zip files

| Zip | Size | Zip date | Entries | Entry date range | Contains .git? | Notes |
|---|---|---|---|---|---|---|
| T12_final_code\opus-documentprintdata-service.zip | 47.3 MB | 2026-09-10 11:10 | 6800 files | 2026-08-11 .. 2026-09-10 | yes (199 entries) | snapshot before the Sep-21 commits; its RejectRisk/AspEndtDocumentService/Template12BaseService are older than all clones (hashes 41532f00, 3a18d0d3, e4a50949) |
| T12_final_code\opus-documentprintdata-service_final_work.zip | 48.3 MB | 2026-09-27 12:14 | 7085 | 2026-08-11 .. 2026-09-22 | yes (427) | **identical for the module files to T12FC working tree** (RejectRiskHandler 89a1485c, RejectRiskService ac8d116a, AspEndtDocumentService 5ff600ad, Template12 = cc6). Older than cc6a1b84 for ASP |
| T12_final_code\Template_12.zip | 6.4 MB | 2026-08-24 | 58 | 2026-08-19 .. 2026-08-24 | no | working docs only (not code of the three modules) |
| new_springboot_upgraded\opus-documentprintdata-service.zip | 54.6 MB | 2026-09-27 23:02 | 8168 | 2026-09-22 .. 2026-09-27 | yes (80) | **pre-commit snapshot of GS's cc6a1b84 work**: RejectRiskHandler/Service, AspEndtDocumentMainHandler, AspEndtDocumentLandscapeService, **AspCompanyNameWithGSTRegNoHandler** are byte-identical (normalised) to cc6a1b84; AspEndtDocumentService (93930139, 1611 lines), AspEndtScheduleHandler, AspEndtCoverHandler are earlier iterations (cc6 = 1654 lines, +87 changed lines) |
| Git_pulled\opus-documentprintdata-service.zip | 15.6 MB | 2026-07-29 | 3165 | 2026-07-06 .. 2026-07-29 | yes (46) | June/July base, pre-GS; most of the module files absent (empty-hash) |
| T12_merge_code\Plines_T12_Merged_code.zip | 17.9 MB | 2026-07-30 | 4102 | 2026-07-06 .. 2026-07-30 | yes (578) | pre-GS merge workspace; Template12 files older than every other source |

[FACT] No zip contains a version of RejectRisk, ASP or Template12 files newer than FINAL HEAD + cc6a1b84. The only unique content in a zip is the 09/27 intermediate ASP files (superseded by cc6a1b84, which is in git) - low value but cheap to keep.

---

## 5. Tests

- [FACT] FINAL tracks **1932** src/test files = NB's 1932 (19 files differ between them, among them `AspEndtDocumentServiceTest.java` and `RejectRiskServiceTest.java`, i.e. Shinge's 10/08 updates). SBU 528, T12FC 367, BC1/BC2/For_G_and_CR/Git_pulled 238 each.
- No src/test modifications or untracked files in any clone's working tree (status filtered on src/test is empty everywhere). Only unique test name outside FINAL: `motor/service/ManualDummyServiceTest.java` (in T12FC and the June/July clones; not in FINAL).
- Module tests that exist in FINAL (all by others): AspEndtCoverHandlerTest, AspEndtDocument2MainHandlerTest, AspEndtDocumentMainHandlerTest, AspEndtScheduleHandlerTest, RejectRiskHandlerTest, PlineTemplate12*HandlerTest (6), Template12LandscapeDispatchHandlerTest, Template12PolicyDocumentHandlerTest, DTO/details tests, AspEndtDocument(2|Landscape)?ServiceTest, RejectRiskServiceTest, Template12*ServiceTest (4).
- Last commits on those tests: Shinge 2026-10-08, 10-06, 10-01; Karunakar 2026-09-30 ("test coverage for plines.service/..."); Shaw 2026-09-29 ("Test cases added for plines"); E2FQZCS 2026-09-04.
- Commits touching src/test after 2026-09-25: Shinge (5: 10-08, 10-07, 10-06 x2, 10-01), Karunakar (09-30), Shaw (09-29), plus unrelated Bond/Fire/Spring-version commits. None by GS.
- [FACT] Conclusion: **there are no tests written by you in any clone or ref.** No newer uncommitted tests exist elsewhere. If the interview pack claims tests, verify manually or write them (see 8).

---

## 6. XSD versions

- Found Pline*.xsd files (Documents tree search): 8 files. Three distinct sources:
  - `Documents\PLINE LOB v0.1.6_New 1\...\Plines LOB v0.1.6\Pline.xsd`, 100409 bytes, 2026-08-25, md5 `55255bec56cebde5e17d26627fea52aa` (CRLF original).
  - `src/test/resources/xsd/plines/Pline.xsd` in FINAL (2026-10-08), NB, SBU, T12FC (2026-08-30) and `Documents\ASPEndtSchedule\base_code\...` (2026-08-22): 101809 bytes, md5 `fdfe9d05a0022c4c7399ecfe509986bc`.
  - After normalising line endings the delivered v0.1.6 and the repo copy are **identical** (diff = 0 lines); the md5 difference is only CRLF vs LF. `Common.xsd` md5 `1f47b156a1841eabc1f62e83061b5e32` is the same in both.
- Header version history in both files lists v0.1.3, v0.1.4, v0.1.5, v0.1.6; **no version above 0.1.6 anywhere** (also `git log --all -S'v0.1.7' -- '*.xsd'` is empty).
- Last commits to the repo copy: Kumari Soni 2026-09-28 (OPUSM-44159 Template25/26/6), Shinge 2026-08-08, Shinge 2026-08-03. None by GS.

---

## 7. Branch / merge story vs Jira ids

| Jira | Commits (subject grep) | Carried by | Merge direction / PR |
|---|---|---|---|
| OPUSM-41763 | 4 by GS: f79ecc811 (08-20), 578778080 (08-25), ba9e29586 (08-26), cc6a1b84d (10-05) | `origin/OPUSM-41763/retrofit-develop-R2-pline-Template12-ASP-RejectRisk` (tip cc6a1b84), `origin/OPUSM-41763-backup` (tip ba9e29586; holds #5, #6 only reachable here), `origin/feature/retrofit-develop-R2-pline` (has f79ecc8 only) | Branch was kept in step with feature/retrofit-develop-R2-pline by `git pull` merges (Merge branch '...' of https://... into ...). **No "Merge pull request #n" for this ticket exists in any clone** - [INFERENCE] never PR-merged into retrofit-develop-R1/R2 yet. cc6a1b84 is not in FINAL, not in origin/retrofit-develop-R1 |
| OPUSM-41698 | 1: 0d715fbad (09-01, RejectRisk + Pline.xsd v0.1.6) | feature/retrofit-develop-R2-pline, OPUSM-36035/..., OPUSM-41697/..., Autorenewal_Document_Updated, OPUSM-41763/... | not in FINAL, not in R1. No PR number |
| OPUSM-36054 + OPUSM-40884 | 7a925920c (08-28, GS, both ids). OPUSM-40884 also 4babbaa1e (08-26, extern-yadav-amarnath "dynamic merging for ACPEndtDoc") | same set as 41698 | not in FINAL; no PR number |
| OPUSM-36035 | 4 by Karunakar (09-10 .. 09-15): e.g. e27bec154 "Merge branch 'feature/retrofit-develop-R2-pline' into OPUSM-36035/retrofit-develop-R2-pline-template12MemoListChanges" | origin/OPUSM-36035/... | direction feature/retrofit-develop-R2-pline -> OPUSM-36035 (sync merge). Not yours; includes GS commits #3-#8 via that sync. No PR number |
| OPUSM-41697 | 4 by Varsha (08-26 .. 09-22) | origin/OPUSM-41697/retrofit-develop-R2-pline-underWritingErrors | a1a8808bb 2026-08-26 "Merge pull request **#654** from OPUSCCC/OPUSM-41697/retrofit-develop-R2-pline-underWritingErrors" (extern-varsha-alla). Contains GS commits #3-#8 as ancestors. Not yours |
| OPUSM-44159 | 77 commits, mainly Shinge/Neethu/Kumari/Shaw/Karunakar | feature/retrofit-develop-updated (FINAL), SBU branch, NB, many OPUSM-44159/* | PRs seen: #766 (Template1-2), #791 (Template2), #812 (Template2Landscape), #837 (Template2-12), #888 (xsd-changes), #893 (Template8), #915/#917/#918 (test-coverage / plines-temp-branch). Merge direction in the Boot-4 line: `feature/retrofit-develop-updated` <- sonar fixes (25d7f5d9b), and 10-08 "Resolved merge conflicts" (d44d0f9fa) on origin. **None of GS's commits (other than #1, #2) are on this line** |

General: origin/retrofit-develop-R1 (default branch, latest 2026-10-08 PR #1024) contains **none** of the 11 GS commits. PR merges onto R2 line are visible in FINAL first-parent history (#878, #906, #916, #923, #929, #931, #932 ... from other developers).

---

## 8. Recommendations (checklist)

### 8.1 Source of truth
- [ ] Treat **FINAL / origin/feature/retrofit-develop-updated** (Boot 4.1.1, 1932 tests, newest Template12 + RejectRisk report-name work) as the code base going forward. Run `git fetch` (allowed for you, it is not a read-only run) and fast-forward over the 3 remote commits of 10-08 first.
- [ ] Treat **origin/OPUSM-41763/retrofit-develop-R2-pline-Template12-ASP-RejectRisk (cc6a1b84)** as the source of truth for your ASP and RejectRisk logic - it is the newest GS work and is pushed (also keep origin/OPUSM-41763-backup untouched: it is the only ref holding commits 578778080 and ba9e29586).
- [ ] Keep SBU (it holds the only local copy-with-config of that branch plus the 09/27 zip) until the cherry-pick is pushed.

### 8.2 What to cherry-pick / merge
- [ ] First: commit the 2 RejectRisk files in FINAL (not application.properties) on a topic branch, e.g. `OPUSM-41698/rejectrisk-date-filter`, with a ticket id in the message. Use `git add -p` or `git add <two files>`.
- [ ] Port cc6a1b84 to the Boot-4 line **by file group, not blind cherry-pick**: `git cherry-pick -n cc6a1b84` on a new branch off feature/retrofit-develop-updated, then resolve per file:
  - Take cc6 side for: `AspCompanyNameWithGSTRegNoHandler.java` (new), AspEndtScheduleHandler's 5 new methods, AspEndtDocumentMainHandler `fetchEndtCode/fetchProductCode`, `AspDeleteMemberLandscapeHandler/Dto`, `AspAddMemberLandscapePremiumHandler`, `AspServiceTaxInvoiceHandler`.
  - Merge by hand in AspEndtDocumentService: keep FINAL's `sectionReportData/landscapeRow` + report-name tagging, add cc6's `snapshotEndorsement/toLandscapeReportData`, `setEndt*` and `setContractId`.
  - Keep FINAL's version of RejectRiskService (ReportHeader/names) + cc6's handler date filter (already in your worktree).
  - Drop the `application.properties` hunk of cc6 (it removes 1 line; review).
- [ ] Then optionally port #3-#9 (Aug-Sep) only if cc6a1b84 does not already contain their effect (cc6 is a descendant of all of them on the Boot-3 line, so cc6 content supersedes #3-#9; #5/#6 are on the backup branch only, and their content also flows into cc6).
- [ ] Re-run the module tests (`RejectRiskHandlerTest`, `RejectRiskServiceTest`, `AspEndt*Test`, `Template12*Test`, `PlineXsdValidationTest`) after the port, and add your own tests (none exist from you).
- [ ] Open a PR from the topic branch into feature/retrofit-develop-updated (or the team's agreed target) and record the PR number against OPUSM-41763/41698/36054/40884.

### 8.3 What NOT to lose / delete
- [ ] cc6a1b84 and ba9e29586/578778080: pushed on remote branches - do not delete those branches; also keep a local tag/branch (`git branch keep/gs-41763 cc6a1b84`) in FINAL (allowed since it changes only refs when YOU run it; I did not).
- [ ] T12FC uncommitted `AspEndtDocumentService` diff (+11/-3; also in `_final_work.zip`) and untracked `.postman/`, `postman/`, `_jrxml_audit_tmp*`: scratch/API collections; copy them out before deleting T12FC.
- [ ] SBU zip (09/27) - only place where the pre-commit version of AspCompanyNameWithGSTRegNoHandler exists outside git.
- [ ] SBU `ASP\` sample contract XMLs, `prompt1-3.txt`, `Template12_*.csv`, `contract_id_list.xlsx` etc. (test data / prompts - not in git).
- [ ] BC1/BC2/For_G_and_CR/Git_pulled/T12_merge_code hold large uncommitted staged/untracked plines packages (BC1 188 staged adds, BC2 246 staged + 153 untracked, For_G_and_CR 98 modified, Git_pulled 126 untracked, T12_merge 435 entries). [INFERENCE] Most is the early plines module that Shinge later committed (8d1b537, 07-06) and is now in git, but diff before deleting; any file not in FINAL could be your early work.

### 8.4 Avoid committing the Mongo credentials in application.properties
[FACT] FINAL and NB both have an uncommitted change adding, under a "#newDev" comment, `spring.mongodb.uri=<redacted>` and `spring.mongodb.database=<redacted>` (the database name looks like a production-style name - treat it as sensitive). The repo has a gitleaks gate (PR #878, OPUSM-57911) so a push would likely be blocked or flagged, but the commit would still be in your local history.
- [ ] Never `git add .` / `git add -A` / `git commit -a` in these clones; stage by path (`git add <file>` or `git add -p`). The `pull_cmt_push_cmds.txt` in Plines_Base_code teaches `git add .` - change it.
- [ ] Keep local overrides out of the tracked file: use a profile file `application-local.properties` (add it to `.git/info/exclude`) and run with `--spring.profiles.active=local`, or set `SPRING_MONGODB_URI` / `SPRING_MONGODB_DATABASE` as environment variables (Spring relaxed binding), or an IDE run configuration.
- [ ] If you must edit the tracked file: `git update-index --skip-worktree src/main/resources/application.properties` (undo with `--no-skip-worktree`), or `git stash push -p`.
- [ ] Add a pre-commit hook or run gitleaks locally; check `git diff --cached` before every commit.
- [ ] If a credential was ever committed or pushed: rotate it, do not just revert (history keeps it).
- [ ] Prior commits: confirm none contain it: `git log --all -S'spring.mongodb.uri' -- src/main/resources/application.properties` (read-only).

### 8.5 Why several clones exist (inferred)
[INFERENCE] Reconstructed timeline:
1. Early June-July 2026: BC1 / BC2_local_work / For_G_and_CR / Git_pulled / T12_merge_code - copies of the plines base (June 9 PR #184 base, then Shinge's 07-06 bulk "Add Plines module" commit). You kept copies as "before/after" safety copies while working with large uncommitted trees (hundreds of staged/untracked files) and zipped them (07-29, 07-30).
2. Aug 2026: T12_final_code on feature/retrofit-develop-R2-pline (Boot 3.4.11) for Template12 -> ASP -> RejectRisk work (GS commits Aug 11 - Sep 21), with zips on 09-10 and 09-27 as manual backups and local branch `OPUSM-41763-backup`.
3. Late Sep: team moved to Spring Boot 4 upgrade (OPUSM-44159, Shinge). You cloned new_springboot_upgraded (Boot 4.1.1) and pushed your newest ASP/RejectRisk work (cc6a1b84, 10-05) from there on the OPUSM-41763 branch.
4. Oct: New_branch and New_branch\2 are fresh clones of the new `feature/retrofit-develop-updated` line (NB at 10-07, FINAL at 10-08) used for the Boot-4 retrofit; FINAL carries the manual re-application of your RejectRisk fix.
Cause of sprawl: two parallel lines (Boot 3.4 vs Boot 4.1) with no merge between them, plus no commits of local WIP (so safety was achieved by copy/zip).

---

## 9. Interview questions (git / branching / merge conflicts) with strong answers

1. You have several local clones of one repo. How do you decide which is the source of truth?
   Answer: Not by folder name or timestamp. I compare `git log --all`, `rev-list --left-right --count`, `merge-base --is-ancestor` and `git cherry` against the remote, then check uncommitted work with `git status`/`git diff`. In my case the Boot-4 clone had the newest team code, but my last pushed commit (cc6a1b84) lived on a different branch and was not an ancestor of it.

2. How do you find out if your commit is in a branch?
   Answer: `git merge-base --is-ancestor <hash> <branch>` (exit code 0 = yes), or `git branch -r --contains <hash>`. `git cherry` additionally detects patch-equivalent commits that were rebased or cherry-picked.

3. Two branches diverged by 1363 vs 194 commits. What do you do?
   Answer: Do not merge blindly. Identify the merge base (`git merge-base`), list the touched files on each side, port the smaller side (mine) onto the bigger line with `cherry-pick -n` on a topic branch, resolve conflicts per file with tests, and open a PR.

4. What was the actual conflict in your ASP files?
   Answer: Both sides changed `AspEndtDocumentService` and `AspEndtScheduleHandler`: mine added field mapping (`setEndt*`, contractId, `snapshotEndorsement`), the Boot-4 line added report/sub-report-name tagging and section helpers. Resolution: keep both sets, because the method names are mostly disjoint, then run the tests.

5. Merge vs rebase vs cherry-pick - when did you use each?
   Answer: I used `git pull` merges to sync my branch with feature/retrofit-develop-R2-pline (hence "Merge branch ... of https://..." commits). Rebase only on private, unpushed work. Cherry-pick (`-n`) to port my fixes across the Boot-3/Boot-4 split, because merging two lines with 1363/194 divergence would drag unrelated history.

6. Why did you get "Merge branch ... of https://..." commits and how do you avoid noise?
   Answer: `git pull` on a branch that moved remotely creates a merge commit. Use `git fetch` then `git rebase origin/<branch>` for private commits, or `git pull --rebase`; keep merge commits for real integrations.

7. You have uncommitted changes and need to switch branches. What do you do?
   Answer: Commit WIP on a topic branch, or `git stash push -p`, or use a second worktree (`git worktree add`) instead of copying the repo to another folder.

8. How do you avoid committing secrets like a Mongo URI in application.properties?
   Answer: Stage by path / `git add -p`, never `git add .`; keep overrides in a git-excluded `application-local.properties` or env vars; `git update-index --skip-worktree` on the tracked file; gitleaks pre-commit and the repo's CI gate; rotate if exposed.

9. A secret was committed locally but not pushed. What now?
   Answer: Amend/reset the local commit (`git commit --amend`, or `git reset --soft HEAD~1`) before pushing, verify with `git log -p -S`, and rotate the secret if there is any doubt. If pushed, rotate first and rewrite history only with team agreement.

10. Why are there pull-merge commits authored by you but no "Merge pull request" commits?
    Answer: Pull-request merges are created by the platform/reviewer under their own name (e.g. #654 by the reviewer). My own merges were local syncs. PR merge commits tie a ticket to a PR number, which is why I record PR numbers against Jira ids.

11. How would you prove which of your commits made it to the default branch?
    Answer: `git branch -r --contains <hash>` and `git merge-base --is-ancestor <hash> origin/retrofit-develop-R1`. In my repo none of my 11 commits are on R1 yet; only 2 are on the Boot-4 line.

12. What is a backup branch for and what risk does it carry (OPUSM-41763-backup)?
    Answer: It preserves commits that could otherwise be lost; here it is the only ref with two of my commits. Risk: it drifts and nobody knows its purpose. I protect it and name/tag it, then retire after the real branch contains the content.

13. How do you review what changed in a file between two clones?
    Answer: `git hash-object` on the files (or sha1 after normalising CRLF), `git diff --stat` between refs, and compare method inventories. Line-ending differences show different md5 even when content is the same - I saw that with Pline.xsd.

14. What do you do about `git status` showing hundreds of untracked/staged files in an old clone?
    Answer: Do not delete. Diff it against the up-to-date branch (`git diff --no-index` or compare to the later commit), save anything unique to a branch or a patch (`git diff > x.patch`), then archive or remove the clone.

15. How do you keep one author identity across machines/clones?
    Answer: Set `user.name`/`user.email` per repo (`git config --local`). I found one clone with the gmail identity and one with the corporate GS identity; commits from the former would not group with my earlier work.

16. How do you track tickets in commits?
    Answer: Start subject lines with the Jira id (OPUSM-41763 ...) and put the same id in the branch name; then `git log --all --grep=OPUSM-xxxxx` gives the full story. I note where I did not (commit 94ea23d81 has no id).

17. What does it mean that two sets of commits show `+` in `git cherry`?
    Answer: The patch is not present upstream. All my commits #3-#11 showed `+` against the Boot-4 branch, confirming they were never ported.

18. Describe the safest way to bring your branch into a team branch with conflicts.
    Answer: Create a new branch off the target, `git cherry-pick -n <commit>`, resolve file by file, build and run the module tests, commit, push, PR. Use `git rerere` for repeated conflicts and keep the original branch untouched as the fallback.

19. What would you do differently to avoid having 8 clones?
    Answer: One clone plus `git worktree` per branch, commit WIP daily on feature branches (push as backup), use tags instead of zips, and keep scratch files (postman, jrxml audits) in a git-excluded folder outside the repo.

20. You discover your teammate's branch overwrote your RejectRisk behaviour. How do you handle it?
    Answer: Check history (`git log -p --follow <file>`), identify the divergence point, explain with commit ids, ask for the fix to be re-applied via PR or push it myself on a topic branch with tests (RejectRiskHandlerTest for the date-range filter), and communicate in the ticket - no force pushes.

---

## 10. Key findings (short summary)

1. Your identity is `GS <extern.gs_nisarga@allianz.com>`; union of GS commits = **11** (earliest 2026-08-11, latest 2026-10-05); 0 in BC1/BC2/For_G_and_CR/Git_pulled; **none touch src/test in any clone/ref**; no other identity of yours exists in history (Shinge is a different person).
2. FINAL (Boot 4.1.1, HEAD 861f0666c, 10-08) contains only 2 of your 11 commits (Aug 11 T12). Your newest work, **cc6a1b84 (10-05, ASP + RejectRisk fixes)**, is only on `origin/OPUSM-41763/retrofit-develop-R2-pline-Template12-ASP-RejectRisk` and in SBU; two earlier commits live only on `origin/OPUSM-41763-backup`. None are on origin/retrofit-develop-R1. The lines have diverged (1363 vs 194).
3. FINAL lacks: `AspCompanyNameWithGSTRegNoHandler` (file absent), AspEndtScheduleHandler 5 methods, `snapshotEndorsement/toLandscapeReportData`, `setEndt*`/contractId mapping, `fetchEndtCode/fetchProductCode`. FINAL is ahead on Template12 (Proviso/companyType) and RejectRisk report headers/names, plus all tests.
4. FINAL's 3 uncommitted files: RejectRiskHandler and RejectRiskService (your date-filter fix, ~= cc6) and application.properties with Mongo URI/db (`<redacted>`) - do not commit that file. T12FC has an intermediate uncommitted AspEndtDocumentService (superseded by cc6).
5. Zips contain nothing newer than git (SBU zip 09/27 = pre-commit cc6 snapshot, T12 `_final_work.zip` = T12FC worktree).
6. XSD: latest is v0.1.6 everywhere; repo copy equals delivered copy except CRLF (md5 fdfe9d05... vs 55255bec...).
7. Immediate actions: fetch/fast-forward FINAL, commit the RejectRisk fix on a topic branch, port cc6a1b84 file-by-file, add own tests, fix local git identity, keep application.properties out of commits, then retire clones only after diffing their uncommitted trees.
