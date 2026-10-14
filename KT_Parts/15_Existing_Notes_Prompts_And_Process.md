# 15 - Existing Notes, Prompts and Process Evidence

Written 2026-10-14. Source: the developer's own local artifacts (outside the repo) that the earlier pack called "not in repo". Everything below was read directly; nothing was executed, no Maven run, no Mongo query. Tokens, connection strings, hostnames and URLs are shown as `<redacted>`. Statements that are my reading rather than a quote are labelled **[inference]**. Facts only the developer knows are marked **[CONFIRM]**.

Companion files: `00_INDEX_Read_Me_First.md`, `14_My_Commits_And_Ownership.md` (GS commits), `02_Template12_Deep_Dive.md`, `03_ASP_Deep_Dive.md`, `07a`/`07b` (defects and STAR stories).

IMPORTANT attribution caution (read first). Several artifacts carry other people's or other machines' fingerprints:
- `.scratch/*.ps1` and `*_rows.txt` inside the repo copy are tracked in git and were committed by **Karunakar** (`bb82a9c4`, 2026-08-10, "My local changes - t12") and **D / Gokulakrishnan** (`cbb89978`, 2026-08-03, "Add GHS handler/DTO/payload chain and scratch JRXML fixtures"), not by GS. The scripts hard-code a different Windows profile (`C:\Users\YDVFYES\Documents\Code\Plines_03_08_26\...`).
- `xsd_mapping_prompt.txt` Prompt 3 and `prompt1.txt` use paths under `C:\Users\FDAP9F4\...` (AutoRenewal, ScrutinyRejectedChecklist, Auto_Policy_*), which are other reports/another profile.
- So: describe the scratch scripts as "team scratch tooling in the repo that I read/used", never as "scripts I wrote", unless you can confirm you ran/wrote them **[CONFIRM]**.

---------------------------------------------------------------------

## 1. Inventory

Sizes are bytes. "Mine?" is my assessment of ownership from evidence only.

### 1.1 Earlier prep notes (the developer's own, 2026-10-07)

| Path | Type | Size | What it is | Mine? |
|---|---|---|---|---|
| `Documents\Plines_Interview_Prep\01_Project_Business_Architecture_Tech.md` | md | 13,543 | Project pitch (30s/2/5/10 min), business rules, Jira ticket decode, architecture, design-decision table, tech table | Yes (developer's own prep) |
| `...\02_My_Role_and_Code_Walkthrough.md` | md | 10,188 | Role statement, day-to-day, RejectRisk/ASP/Template12 walkthrough, patterns | Yes |
| `...\03_Data_API_Issues_Support_Perf_Test_DevOps_Security.md` | md | 10,438 | Mongo, API, 13-row issue table, support playbook, perf, testing, DevOps, security | Yes |
| `...\04_Stories_Questions_CheatSheet.md` | md | 8,618 | 7 STAR seeds, ~45 Q&A, scenario seeds, cheat sheet | Yes |

### 1.2 Prompts (AI-assistant prompts the developer wrote or pasted)

| Path | Type | Size | What it is |
|---|---|---|---|
| `Documents\code generation prompt.txt` | txt | 7,832 | Earliest prompt set (modified 2026-07-29). RiskUploadUnsuccessful JRXML to Java conversion; then T12 PolicyDocument2 and Template4 Landscape hierarchy CSV, "check your code", "add condition" prompts; last block asks to restructure nested subreport calls under their gates (ServiceTaxInvoice example). Also an older copy at `ASPEndtSchedule\base_code\ASP\prompt.txt` (7,942). |
| `Documents\xsd_changes_prompt.txt` | txt | 7,891 | "Restructure Plines LOB Report XML Responses to New XSD Format" (XSD v0.1.6). Impact-analysis-first requirement, Fire LOB as reference, Plines-only classes `ReportParameters`/`ReportData`, no change outside `plines` package. Targets `AspEndtDocumentService` and `AspEndtDocument2Service`. |
| `Documents\xsd_mapping_prompt.txt` | txt | 5,531 | Three chained prompts: (1) JRXML to Oracle table/column CSV + call-hierarchy sheet; (2) join that sheet to the master mapping sheet (columns D vs H) to fill XSD attribute and XSD group; (3) derive "Code XSD Mapping Path" from a response XML. |
| `Documents\report_subreport_checking_prompt.txt` | txt | 40,734 (2,105 lines) | Read-only "QA/UAT readiness" audit prompt, reused several times (EndtDocument tree; EndtDocument2 + EndtDocument; Add/Delete-MemberLandscape trees, twice). Modified 2026-10-13. |
| `Plines_Base_code\new_springboot_upgraded\prompt1.txt` | txt | 1,435 | Compare Auto_Policy Successful/Unsuccessful JRXMLs with their Java services; "fan out subagents". |
| `...\prompt2.txt` | txt | 2,410 | Compare `EndtDocument` JRXML tree (11 files) with `AspEndtDocumentService.java`; fixed "Issue / Expected / Actual / Affected / Impact" output; "fan out subagents". |
| `...\prompt3.txt` | txt | 3,450 | Same for `EndtDocument2` tree (~20 files) vs `AspEndtDocument2Service.java`; "fan out subagents". Modified 2026-10-13. |
| `ASPEndtSchedule\base_code\ASP\Finding Values_QueryString_Prompt 1.txt` | txt | 19,287 | Prompts to turn a JRXML `<queryString>` into a Mongo aggregation and find a dev contract id whose data satisfies the parent `printWhenExpression` ("Dev environment only, we can insert/update column values"). Per-report blocks for ServiceTaxInvoice, ServiceTaxInvoice_PA4A, Template4 ServiceTaxInvoice, ComplaintRedress_SCB, ServiceTaxInvoicePL... |

### 1.3 Process / command notes

| Path | Size | What it is |
|---|---|---|
| `Plines_Base_code\pull_cmt_push_cmds.txt` | 297 | 7-line git routine cheat sheet (see 2.6) |
| `ASPEndtSchedule\base_code\ASP\report_path.txt` | 6,718 | ASPEndtDocument hierarchy as a tree with JRXML file paths and line numbers of the call sites |
| `plines_testing\Template_12\PolicyDocuement\subreport_path.txt`, `...\PolicyDocument2\report_path.txt` | 1.7 KB / 1.6 KB | T12 subreport trees with call-site line numbers and owning template (T12, T4, CR, G). The PolicyDocument2 file also holds a pasted "HARDCODED FOR TESTING" snippet (see 3.8). |
| `plines_testing\RejectRisk\{payload,mongo_compass_aggregate,postman_result}.txt`, `REJECTRISK_IMAGES.docx` | 2.6 KB / 2.6 KB / 38 KB / docx | RejectRisk verification kit: test payload (with picked Oracle rows), the logged aggregation from Compass, the Postman XML result; the docx is a template with headings "SQL RESULT / POSTMAN / CONSOLE / MongoDB Compass" (images only) |
| `Documents\jrxml_list1-3.txt`, `*_Hierarchy.csv`, `build_landscape_hierarchy.ps1` | small | Hierarchy extraction outputs (Template12 Landscape, PolicyDocument2, Template4 Landscape) and the PowerShell that builds the CSV from the JRXML dumps |
| `Documents\jr-tools\{ReportTree,XlsxDump,UpdateStatus,MongoProbe}.java`, `mp.sh`, `b1-3.txt`, `out1.txt`, `ids100909.txt` | small | Java single-file utilities: JRXML subreport tree builder; xlsx dumper; tracker status updater; **read-only** Mongo probe (find/count/distinct/aggregate only; reads the URI from application.properties, which is not echoed). Dated 2026-10-07. |
| `New folder\AHI\contract_id_details.txt` (13 KB) and `Plines_Base_code\new_springboot_upgraded\ASP\contract_id _list.txt` (4 KB) | txt | Test contract lists mapped to which ASP sub-reports they fire (e.g. 24080537 for AddMember/Delete landscape; 27275348 covers all EndtDocument subreports) |
| `New folder\Nisarga_req_res\{ASP,RejectRisk}\*Request/ResponsePayload.txt` | up to 603 KB | Saved request/response pairs for ASP and RejectRisk (folder name has the developer's first name) |

### 1.4 Defect / data-mismatch write-ups

| Path | Size | What it is |
|---|---|---|
| `Documents\Data_mismatch_issue.docx` | 301,532 (2026-06-24) | 1 page + 6 screenshots: Oracle vs Mongo comparison for `CMGE_UPLOAD_FILE` joined to `WIP_UWGE_POLICY_VERSIONS`; conclusion "data issue, not a code defect" |
| `Plines_Base_code\T12_final_code\Tempalte12_details\KNOWN_BUG_staxCount_eInvStatus.txt` | 7,282 (2026-08-17) | Two code bugs in Template12 service-tax-invoice family |
| `Documents\New folder\Template_12_MAIN\new_responses\CODE_CHANGES_LOG.txt` | 8,647 | Follow-up log (2026-08-19/20): 3 numbered code changes with before/after, verification, and 2 known-not-fixed items |
| `...\Tempalte12_details\test_write.txt` | 14 | A UTF-16 "test" file; no content value |
| `Plines_Base_code\New_branch\opus-documentprintdata-service\CHANGELOG.md` | 14,280 | Auto-generated team changelog (2026-09-21), ~80 OPUSM tickets and PR numbers. Not specific to the developer; mostly other LOBs. Useful only to show the team's release/PR numbering (#855-#864) |
| `T12_merge_code\...\plines_t12_diff.txt` | 137,637 | Raw `git diff` of the July T12 merge (new handlers). Code, not a note |

### 1.5 Repo-local Claude config and scratch (T12_final_code copy)

| Path | What it is |
|---|---|
| `T12_final_code\opus-documentprintdata-service\.claude\settings.local.json` (450 B) | Allow-list of Bash permissions: `git fetch/remote/status/log/rev-parse/rev-list/checkout/branch/push`, `cd`, one `xargs ... grep -l "staxPcts|STAX_PCTS|StaxList"` pattern. No secrets. Shows an AI coding assistant was run in this folder and was pre-approved for git read/branch/push and nothing else (no `mvn`, no file-edit allowance). |
| `.scratch\` (about 100 files) | `check_*.ps1`, `parse_*.ps1`, `*_rows.txt`, two mapping workbook copies; see 2.5 |
| `ARCHITECTURE.md` (10.8 KB) | Team architecture doc, committed by Biswal 2026-06-18, not the developer's |
| `null` (842 B) | Accidental shell redirect file (PowerShell error "assembly DocumentFormat.OpenXml could not be found"); proves a script tried to use OpenXml and fell back to raw zip parsing **[inference]** |
| `_jrxml_audit_tmp`, `_jrxml_audit_tmp2` | Copies of JRXMLs (ComplaintRedress_NoBlankPage, EndtSchedule, TransitionalInvoice, SSTNote, STCompanyContact, TransitionalInvoice2): untracked, i.e. used during the ASP EndtDocument2 audit **[inference]** |

### 1.6 Spreadsheets (names only, not parsed in depth)
`Pline_MASTER_MAPPING_SHEET_v0.1 (final1) 1 5.xlsx` (160 KB, the XSD mapping source of truth), `xsd_check.xlsx`, `Reports_check_list.xlsx`, `reports_list.xlsx`, `MongoAPI_analysis*.xlsx`, `PolicySchedule(T-1,2,3).xlsx`, `PolicySchedule(T-25).xlsx`, `ASP_contract_id_list.xlsx`; in `T12_final_code\Tempalte12_details\`: `contract_id_list.xlsx` (+ 12 `.bak` versions, 2026-08-10 to 08-19), `contract_reports_mapping.xlsx`, `Template12_XSD_Mapping_Final.xlsx`. Opened `contract_id_list.xlsx`: three sheets (PolicyDocument, PolicyDocument2, Landscape); columns "Subreport | Path/Notes | Completed/FiredWithRealValue | Contract ID". It is the **per-subreport test tracker** (gate condition + contract id that fires it + Done/Blocked).

---------------------------------------------------------------------

## 2. The developer's working method (as evidenced)

Phrase everything as "I used", only for steps you actually did **[CONFIRM each]**.

### 2.1 Treat JRXML as the contract, Java as the implementation
Evidence: `report_subreport_checking_prompt.txt` says "Treat JRXML as a CONTRACT", "THIS IS NOT A JASPER REVIEW... THIS IS A JAVA CODE IMPLEMENTATION REVIEW", and demands a trace per field: `Database Column -> Query -> Alias -> DTO -> Handler -> Service -> Payload -> XSD -> Setter -> Getter -> Serialization -> JRXML`. Each field is marked VERIFIED / NOT VERIFIED, and "If ANY field cannot be fully traced... mark NOT READY FOR TESTING." It also asks for params (declare -> pass -> consume), PrintExpression routing, report registration, and "Can this report actually execute successfully?" Output format: summary counts, Top 5 issues, `ISSUE #n` blocks (Severity, Report, Field, Problem, Expected, Actual, Files, Impact, Status), then 13 fixed sections.

### 2.2 AI-assisted comparison prompts ("fan out subagents")
Factual statement you can make: **"I used an AI coding assistant (Claude) with structured comparison prompts to cross-check each JRXML tree against the Java service, then I verified its findings against the code, Postman and Mongo."**
- `prompt1.txt`: JRXML `queryString`, `printWhenExpression`, `field`, `subreportExpression` hierarchy vs Mongo aggregations for **Auto_Policy_Successful / Unsuccessful** (`AutoPolicySuccessfulService`, `AutoPolicyUnsuccessfulService`). Asks for analysis only ("Please Dont make any Code implementation changes"), "do deep analysis check multiple times", ends with "fan out subagents", then a second paragraph "implement ... check once again 2 times ... finally build clean maven".
- `prompt2.txt`: **ASP EndtDocument** tree (EndtDocument, PrintSeriesFooter, EndtCover, CompanyNameWithGSTRegNo, Blank, EndtSchedule, EndtNarration, Signature, EndtScheduleOfBenefits_ASP, ASPLLB, ASPSTI) vs `AspEndtDocumentService.java`. Compares: query vs Mongo logic, fields, parameters, PrintWhenExpression, subreport hierarchy and calling conditions, datasource wiring, DTO/Payload/XSD mapping, getter/setter population, routing/registration. "Do NOT make code changes." Output format fixed: `Issue / Expected / Actual / Affected Report/Subreport / Affected Java File(s) / Impact`; "Reverify all findings and provide only confirmed mismatches. fan out subagents".
- `prompt3.txt`: **ASP EndtDocument2** tree (ComplaintRedress, CompanyNameBRNUpperAndLowercase, DebitNote, CustomerDtlsOfMasterPolicy_Debit, CompanyContactJr, AntiRasuahBanner, ServiceTaxInvoice, STCompanyContact, CustomerDtlsOfMasterPolicy, ServiceTaxInvoiceEndtMT/MTSub, QRSection, TransitionalInvoice/2, SSTNote) vs `AspEndtDocument2Service.java`. Same analysis-only rule and "fan out subagents".
- `report_subreport_checking_prompt.txt` targets, in order: (a) EndtDocument tree; (b) EndtDocument2 tree + EndtDocument tree together, with separate findings and separate readiness verdicts, final question "Would you be comfortable sending BOTH ... to QA/UAT today?"; (c) AddMemberLandscape + DeletionMemberLandscape (with landscape-specific checks: every visible column traced, flag `CRITICAL: LANDSCAPE IMPLEMENTATION GAP`); block (c) appears twice. The phrase "fan out subagents" is **not** in this file; it appears only in prompt1-3.
- Pattern: **audit first, read-only, then separately authorize a fix.** Same discipline appears in `xsd_changes_prompt.txt` ("Do not start implementation until the full impact analysis and proposed change list have been presented") and in CODE_CHANGES_LOG ("explicitly authorized by the user, one at a time, as one-off overrides of the standing 'don't touch code' rule").
- Honest limit: the prompts show what was asked, not what was returned. No AI output files are in the folders I read. Do not claim a number of defects found by these audits **[CONFIRM]**.

### 2.3 Code generation, hierarchy and "add the gate" prompts (earliest phase)
`code generation prompt.txt` (July): (1) convert `RiskUploadUnsuccessful` JRXML `queryString` to a Java API following the Motor controller pattern, create a README, put code in a new `plines` package like `motor`/`fire`; (2) build a main/sub/sub-sub report CSV (`Template12_PolicyDocument2_Subreport_Hierarchy.csv`, `Template12_Landscape_Hierarchy.csv`, `Template4_Landscape_Subreport_Hierarchy.csv` exist at Documents level); (3) "Add condition": copy how `FireCommonRepLandscapeService` guards a subreport (`if ("Y".equalsIgnoreCase(pShowRAListingReport))`) onto the Plines report; (4) restructure handlers so a nested subreport is only called inside its parent's gate, with the example `showTaxInvoice=Y && showStaxInvInd=Y && einvPrintInd=Y -> 070414 ? ServiceTaxInvoicePA4A : ServiceTaxInvoice -> STCompanyContact, STaxInvoiceProductInfo, ServiceTaxInvoicePL, PLSub, QRSection (+ MST, CustomerDtlsOfMasterPolicy)`. Each step repeats the guard question: "While fixing have you made any change in common files? is that will affect to other classes?" (blast-radius control, same concern as CODE_CHANGES_LOG Change 3).

### 2.4 How XSD mapping was driven
- `xsd_mapping_prompt.txt` Prompt 1: input = main JRXML path; output = CSV/XLSX named after the main report in `...\ASP_MAPPING`, columns `Jrxml Name | Referenced From (Parent JRXML) | Oracle Table or View Name | Column Name | Code XSD Mapping Path | XSD Attribute Name | XSD Group (Identified)`, plus a hierarchy sheet. Real outputs exist: `RejectRisk_Jrxml_Table_Column_Mapping.csv/.xlsx`, `RejectRisk_Jrxml_Hierarchy.csv`, and `Apply_MasterMapping.ps1` / `Build_RejectRisk_Xlsx.ps1` in `T12_final_code\RejectRisk\Mapping_sheet\`.
- Prompt 2: join the generated sheet to `Pline_MASTER_MAPPING_SHEET_v0.1 (final1) 1 5.xlsx` (only the sheet named after the report, e.g. `EndtDocument`): where generated column D equals mapping column H, copy mapping columns I (XSD Attribute Name) and J (XSD Group) into generated F and G.
- Prompt 3: from a full response XML, infer the hierarchical path to each column and write it in column E ("Code XSD Mapping Path"); if blank use the literal "Template metadata (report engine configuration) - not a Java XSD payload element".
- `xsd_changes_prompt.txt`: requirement text for OPUSM-44149/44159 (XSD v0.1.6); asks for a Delta analysis, current-vs-target structure, file-level impact table, class creation plan, mapping gap report (XSD field, current source, transformation, gap Y/N), sample XML. "Fire LOB as Reference Architecture". It names `ReportParameters.java` and `ReportData.java`; **[inference]** these became `PlineReportData` and the report-parameter payload class (the repo uses `PlineReportData`).
- The `.scratch` scripts are the same mapping idea at smaller scale (see 2.5).

### 2.5 Scratch scripts and what they validated (read 12 representative ones)
All are PowerShell that opens `mapping_copy4/5.xlsx` (copy of the master mapping workbook) with `System.IO.Compression.ZipFile`, reads `xl/sharedStrings.xml` and `xl/worksheets/sheet12.xml`, and filters rows. Mapping sheet columns used: **B** list of JRXMLs using the column (semicolon list), **H** DB column, **I** XSD attribute, **J** XSD group, **L** flag `Y`, **M** group. Output goes to `*_rows.txt`.
- `parse_ciprint.ps1` (and `_astro/_azp/_bmw/_elite/_fwpa/_pa4a/_posm`): all mapping rows whose B list contains `CIPrint.jrxml` (or the scheme variant) -> `ciprint_rows.txt`. Purpose: which XSD leaf and group each CIPrint column must populate. Sample output: `CITY -> city -> address(clientInfo)`, `AGENT_CODE -> agentCode -> agentInfo`, `PRODUCT_CONFIG_CODE -> productConfigCode -> taxClientInfo`.
- `parse_blank.ps1`: rows for `Blank.jrxml;Blank_NoMargin.jrxml` -> `BLANK -> blank -> transDet`.
- `parse_by_column_name.ps1`: lookup by DB column (PD_RB, RISK_PARENT_ID, DESIGNATION, LLB_PREM, STI_PLAN_CODE...) showing every JRXML that uses it.
- `check_policyno.ps1`, `check_all_policyref.ps1`, `check_policyref*.ps1`, `check_totprem.ps1`, `check_product*.ps1`: for a given column (`POLICY_NO`, `POLICY_REF`...) in a given JRXML (`ParticularOfInsuredPotrait_ASP`...) print H/I/J/M; used to settle which XSD leaf an ambiguous column maps to.
- `check_astro_missing*.ps1`, `check_azp_missing.ps1`: check whether specific columns (BRANCH_NAME, ISSUE_BY, IP_PH_NAME, PREM_DUE_ROUNDED, POLICY_REF) are present for the CIPrint_ASTRO/AZP list, to find mapping gaps (`InAstroList=true/false`).
- `parse_premium_asp.ps1`: 40+ target columns (CONTRACT_ID, INSURED_NAME, PREMIUM, STAX_AMT, STI_PREM, LLB_PREM, COV_DISC_PCT...) for `ParticularOfPremiumPotrait_ASP.jrxml`.
- `check_sheetname.ps1`: lists workbook sheets and relationship ids (to find which `sheetN.xml` is the right mapping sheet; scripts use `sheet12.xml`).
- `extract_partB.ps1`: unpacks a saved AI tool-result JSON (path under `C:\Users\YDVFYES\.claude\projects\...`) into a text file. Shows the author of the scripts also used Claude Code on that profile.
- What they do **not** do: none of these run the service or compare XML; they are mapping lookups. **[inference]** Their value in a story is "I removed guesswork about which XSD leaf a DB column feeds".
- Ownership: committed by Karunakar/D (see caution box). **[CONFIRM]** whether you used/ran them.

### 2.6 Git routine
`pull_cmt_push_cmds.txt` (verbatim):
`git status` ; `git diff <file> -> to see changes` ; `git pull origin <branch name>` ; `git add <file name> -> individually` ; `git add . -> for all` ; `git commit -m "any message"` ; `git push origin <future branch name>`.
Facts: the example file in `git diff` is `Template12PolicyDocumentBaseService.java`. Pull before commit; stage individually when you want to avoid accidental files (the repo shows untracked `.postman/`, `postman/`, `_jrxml_audit_tmp*` and a modified `application.properties` that must not be committed, because it holds the Mongo URI). The project's `jira-commit-check.yml` requires an OPUSM key in the message, but two of the GS commits (`b4e588bd4`, `94ea23d81`) have none (see file 14) - an honest improvement point. The `.claude/settings.local.json` allow-list also lets the AI run `git push *`, which is a risk to mention only if asked about AI safeguards.

### 2.7 Test-data and verification method
- Dev contract discovery: Prompt in `Finding Values_QueryString_Prompt 1.txt` (convert query to aggregation, find a contract whose data satisfies the parent `printWhenExpression`; dev DB allows inserts/updates). `jr-tools\MongoProbe.java` is a read-only probe; `b1.txt` shows commands such as `find UWGE_POLICY_VERSIONS {"CONTRACT_ID":23913536}` and `find UWGE_POLICY_CONTRACTS {"PRODUCT_CONFIG_CODE":"100909"}`.
- Tracker: `contract_id_list.xlsx` per subreport: gate, contract id, "confirmed real" vs "gate fired only", Done/Blocked; 12 `.bak` versions on 2026-08-17 show incremental updates; `contract_reports_mapping.xlsx` maps contract to reports.
- Responses saved: `Template_12\new_responses\<contractId>.xml`; counts of "blocks correct" were computed (e.g. 6 of 17).
- Standard accepted for tracker: a subreport is "Done" if a real value appears in at least one block ("not mandatory in all blocks", KNOWN_BUG text).
- RejectRisk: pick an Oracle row (`PART_ID 1357766`, `NAM_EXT`, `Created_DATE 06-JUL-17`) then choose a date window that contains it (`06-06-17 to 31-08-17`), call the API, compare with the Compass aggregation logged by Spring.

---------------------------------------------------------------------

## 3. Defects, known bugs and data mismatches, with STAR stories

Provenance: items 3.1-3.3 are covered by documents. 3.1 is **corroborated by git**: the code described in CODE_CHANGES_LOG Change 1 + Change 3 is in commit `f79ecc811` (GS, 2026-08-20, "OPUSM-41763 modified changes for T12", `Template12PolicyDocumentBaseService.java` +24/-4), and is present in the current `Template12PolicyDocumentBaseService.mergeNonNullFields` (staxList block and eInvStatus guard). Note the commit message does not mention it. Writing style of KNOWN_BUG and CODE_CHANGES_LOG reads like an AI-assistant log **[inference]**; be ready to say you directed the investigation and reviewed the change.

### 3.1 DEFECT A - staxCount and eInvStatus lost in the Template12 merge (KNOWN_BUG_staxCount_eInvStatus.txt, 2026-08-17; CODE_CHANGES_LOG 2026-08-19/20)

Symptom (quotes): "Two fields that are computed correctly by the handler code never survive into the final print-data response". `staxCount` blank; `eInvStatus` blank where it should be `VALID`. Blocked verification of: `ServiceTaxInvoicePL` (gate `staxCount > 1`), `ServiceTaxInvoicePLSub` (same), `QRSection` (`eInvConsentInd=Y AND eInvStatus=VALID AND (qrCnt>0 or qrInd=Y)`).
Scope: "Template12 ServiceTaxInvoice family (PA4A, T12(635), and likely T4(692))".

Evidence of code-not-data: "confirmed by simulating the exact Java query logic directly against MongoDB and getting the correct result every time, while the live Postman response still shows the field blank."

Root cause 1 (staxCount): `staxCount` lives only inside `staxList.staxPcts` (no top-level field on `PlineTransDet`). `mergeSingleTransDet()` -> `mergeNonNullFields()` "explicitly SKIPS any field whose type name ends in 'List', then only re-merges riskList / sobList / nomineeList". `staxList` was "never added to that exception list". Because many earlier bands have already populated `transDetList`, the new row is merged, not appended, and the stax data is dropped.
Root cause 2 (eInvStatus): two handlers write the same DTO field with different null defaults: `PlineCommonServiceTaxInvoiceHandler` uses `$ifNull(..., "X")`; `PlineCommonSTaxInvoiceProductInfoHandler` (line 93) uses `""`. The second runs right after (`addSTaxInvoiceProductInfo/MST` after `addServiceTaxInvoiceRows`), and the merge "overwrites unconditionally whenever the new value is not literally null"; `""` is non-null so it "silently clobbers the correct 'VALID'".

Fix (documented): 
- Change 1 (2026-08-19): in `mergeNonNullFields`, after the nomineeList re-merge, add stax rows unless the source is a placeholder (`size()==1 && staxAmt==null`; the committed code also checks `mstSstRegNo==null`). Doc comment updated "three lists" to "four lists".
- Change 2 (2026-08-19): change the shared handler default from `""` to `"X"`. **Then reverted** on 2026-08-20 (Change 3).
- Change 3 (2026-08-20): "found PlineCommonSTaxInvoiceProductInfoHandler.fetchSTaxInvoiceProductInfo() is a genuinely shared/common handler... called from 11+ files covering Template5, 5.2, 12, 13, 14, 15, 20, 21, 22 plus PlineXsdTemplate1BaseService..." so changing its null fallback "would have silently altered eInvStatus behavior for every one of those templates". Also a sibling `PlineCommonSTaxInvoiceProductInfoMSTHandler:86` writes `""` too, so Change 2 alone could not fully fix it. New fix is Template12-scoped: guard in `mergeNonNullFields` so a blank `eInvStatus` is copied only if the target's is also blank; "verified all 4 setEInvStatus call sites in this file each set a fresh local PlineTransDet then immediately call mergeSingleTransDet" (repo shows `det.setEInvStatus(row.getEinvStatus())` at 4 places).

Verification (quotes): contract 35774076 re-tested after inserting dev rows in `UWGE_POLICY_BASES_STAX` (6% 01/06/2024-31/08/2024; 8% 01/09/2024-30/06/2025): "staxCount=2 correctly in 4 of the STAX-bearing transDet blocks, with staxAmt2/staxPct2 matching both inserted rows exactly (1.20/6.00 and 2.40/8.00)". Tracker rows for ServiceTaxInvoicePL/PLSub "Blocked -> Done". Found along the way: wrong `pTaxInvNo` in the test payload (`STAX24076000001854` vs real `STAX24062000000072` in `UWGE_GST_TAX_INV`).

Measurements before fix: 39821415 (PA4A, 070414): staxCount right in 6 of 17 blocks (35%), eInvStatus 8 of 17 (47%); 35774076 (090202, 9 blocks, 555 KB): staxCount 0 of 9, eInvStatus 0 of 9, eInvConsentInd 3 of 9, premByMstInd 6 of 9 (unaffected, plain scalar); 35112689 (070202): 26 endorsement versions, 102 MB, about 2,095 blocks, not saved. 40799613: eInvStatus 0 of 6.

Status: Change 1 verified by fresh Postman. Change 3 "code change applied, not yet rebuilt/redeployed or re-verified" as of the log **[CONFIRM later re-test result]**. T4(692) branch "assumed affected until proven otherwise" (not separately tested).
Still open in the log: (1) `qrInd` blank in every block on 40799613 - "not yet explained" (external `FN_GET_EINVOICE_QR_IND` or a second merge-drop). (2) Latent param gap: Java passes the request `pTaxInvNo` to QRSection (`Template12PolicyDocument2Service.java:238`) where JRXML wires `$F{STAX_INV_NO}`; only matters if the caller's number differs from the contract's real one.
Also a process lesson in the log: "the service must actually be restarted, not just rebuilt (see stale-deploy notes)", i.e. earlier false negatives came from a stale deployment.

STAR story (use only these facts):
- **Situation:** While verifying the Template12 service-tax-invoice subreports (ServiceTaxInvoicePL, PLSub, QRSection), their gating fields `staxCount` and `eInvStatus` were blank in the Postman response on almost every contract, so those subreports could not be signed off.
- **Task:** Find out whether it was bad test data or a code defect, and fix it without disturbing other templates.
- **Action:** Re-ran the exact Java query logic directly in Mongo and got correct values every time, so the defect was after the query. Traced it to `mergeNonNullFields` in `Template12PolicyDocumentBaseService`: it skipped every `*List` field, and `staxList` was not in the exception list; and two handlers wrote `eInvStatus` with different null defaults ("X" vs ""), the later one overwriting a good "VALID". My first fix changed the shared handler default; I then found that handler is used by 11+ templates, reverted it, and made the fix inside the Template12 merge step (re-merge `staxList`, and never let a blank `eInvStatus` overwrite a real one). To test, I inserted two dev `STAX` rows (6% and 8%) for contract 35774076 and compared the response to those values. I recorded each change, its before/after and verification in a change log and updated the contract tracker.
- **Result:** `staxCount=2` with `staxAmt2/staxPct2` = 1.20/6.00 and 2.40/8.00 now appeared in 4 stax-bearing blocks; PL and PLSub moved from Blocked to Done. The fix is in commit `f79ecc811` and is present on current code. [CONFIRM: eInvStatus re-test outcome, qrInd resolution, QA outcome.]
- Follow-ups an interviewer may ask: why reflection-based merge is fragile (a new List field is silently dropped until added to the exception list); safer designs (typed merge per field or explicit "merge strategy" per band; stop using non-null as proxy for meaningful).

### 3.2 DEFECT B / data mismatch - RiskUploadUnsuccessful returns 0 rows in Mongo and Oracle (Data_mismatch_issue.docx, 2026-06-24)

Symptom: report built from `CMGE_UPLOAD_FILE` joined to `WIP_UWGE_POLICY_VERSIONS` returned 0 rows (docx heading: "Tables: CMGE_UPLOAD_FILE and WIP_UWGE_POLICY_VERSIONS"; "Issue:", "Oracle:", "MongoDB Compass:").
Evidence in screenshots (read): Oracle `SELECT UPLOAD_REF_NO FROM CMGE_UPLOAD_FILE ... WHERE FILE_TYPE = 'PL_RISKUPL' AND PROCESS_IND = 'E'` returned 7 rows (`UWPL170031560`, `UWPL170036473`, `UWPL170042747`, `UWPL170036473`, `UWPL170031560`, null, `UWPL190167683`). Oracle `SELECT REF_NO FROM WIP_UWGE_POLICY_VERSIONS WHERE REF_NO LIKE 'UWPL%'` returned 50 rows beginning `UWPL240616683`... (24xxxx, 25xxxx). Mongo Compass shows the same 7 UPLOAD_REF_NO values (collection about 51K docs) and the same WIP REF_NO series (collection about 23K docs). A cell reads `<redacted>` for the Atlas host.
Root cause as documented (quote): "UPLOAD_REF_NO values in CMGE_UPLOAD_FILE (2017-2019) do not exist in WIP_UWGE_POLICY_VERSIONS table, WIP_UWGE_POLICY_VERSION table contains the results of 2024-2025 (Latest details), causing the JOIN to return 0 rows. This is a data issue, not a code defect. Both Oracle and MongoDB return 0 rows for the same reason."
Fix: none in code, by design. Status: closed as data issue **[CONFIRM who accepted it and whether a test row was inserted]**.
Code cross-check (I verified): `RiskUploadUnsuccessfulHandler` (plines/documentHandler) javadoc says "Collections: CMGE_UPLOAD_FILE, CMGE_UPLOAD_FILE_ERR, WIP_UWGE_POLICY_VERSIONS; Filter: FILE_TYPE=PL_RISKUPL, PROCESS_IND=E" and it does a `$lookup` on `WIP_UWGE_POLICY_VERSIONS` as an inner-join filter "only keep records with a matching policy version". So the docx is about RiskUploadUnsuccessful **[inference, high confidence]** (the report is named in `code generation prompt.txt`). Handler first committed by Shinge in `d4c069135` (2026-07-16), after the docx date, so GS's analysis preceded it or lives on another branch **[CONFIRM]**.

STAR story:
- **Situation:** The RiskUploadUnsuccessful conversion returned no rows in Mongo and the team had to know if the Mongo port was wrong.
- **Task:** Prove whether the 0 rows came from code or data.
- **Action:** Ran the original Oracle query and the equivalent Mongo query side by side (Oracle SQL Developer vs MongoDB Compass), pulled the join keys from both sides (`UPLOAD_REF_NO` 2017-2019 vs `REF_NO` 2024-2025), and compared. Wrote it up in a one-page note with screenshots.
- **Result:** Showed Oracle and Mongo agree (both 0 on the join), so the code is faithful and the data has no overlap; no code change, avoided a wrong fix. [CONFIRM: whether QA accepted; whether test data was added.]
- Lesson: when a port returns empty, run the legacy query on the same data before touching code; check that join keys overlap in time.

### 3.3 Known gaps documented but NOT fixed (stories only as "I identified and documented")
1. `qrInd` blank in all tested blocks (40799613). Say: "I isolated that eInvStatus clobbering was one cause but qrInd remained unexplained; I recorded two hypotheses (external FN_GET_EINVOICE_QR_IND vs a second merge drop) and left it as an open item." [CONFIRM later outcome.]
2. QRSection `pTaxInvNo` wiring: JRXML uses the row's own `STAX_INV_NO`, Java passes the request parameter. Impact only when caller passes a mismatching number.
3. T4(692) branch not retested.

### 3.4 Other defect-type evidence in these artifacts (smaller)
- **Gate fidelity / order**: `code generation prompt.txt` step "re arrange the service handlers" - nested subreports were called outside their parent gate. Fix direction: put children inside the parent's `if`. Outcome not in these files **[CONFIRM]**.
- **Unmapped column mapping lookups**: `.scratch` shows CIPrint ASTRO/AZP columns checked for presence in the mapping list (`InAstroList` flag). Fix = add mapping or setter **[inference]**.
- **Contract coverage**: `contract_id details` files document that some subreports could only be fired by specific contracts (e.g. EndtDocument2: ServiceTaxInvoice family by 3756869 endt `*003`, EndtTaxDoc/CompanyContactJr/AntiRasuahBanner by endt `*005`, ComplaintRedress by 25190981), meaning single-contract testing hides branches.
- **Hardcoded test values**: `plines_testing\Template_12\PolicyDocument2\report_path.txt` contains pasted text "HARDCODED FOR TESTING - dummy values to force the ComplaintRedress_SCB condition true; remove and restore the lines below once verified" with `schemeName = "POSMESRA"; productCode = "070504"`, and a line "The if block itself is back to its original, untouched real condition". Lesson: a temporary override was used to force a gate and then restored. Verify none remained in committed code (grep `HARDCODED FOR TESTING`) **[CONFIRM]**.
- **Cross-check from `UpdateStatus.java`**: statuses like "Fired (inferred) - prnCnt populated", "Fired - Landscape returned add-member data (ctr 24080537, endt code 72)" and "Not confirmed - no data dependency, not visible in response": shows the tracker distinguishes confirmed vs inferred firing. Good honesty point.

---------------------------------------------------------------------

## 4. Earlier prep notes (Plines_Interview_Prep) summarised and reconciled

### 4.1 What the earlier notes claim (summary)
1. (01) OPUS Plines doc service on Spring Boot, Mongo mirror of Oracle, one endpoint, strategy-per-report, handler-per-subreport; Boot **4.1.1** (CVE-2026-59270), Jackson 2 pinned; Template12 PolicyDocument + PolicyDocument2 + Landscape merged to one bean; ASP EndtDocument + EndtDocument2 + AddMember/Delete landscape merged to one bean; business rules table (tax invoice gating, debit note, QR, STAX_COUNT>1, PDPA, MST).
2. (02) Role: owned Java migration of RejectRisk, ASP, Template12; "most time coding... 70/20/10"; RejectRisk pipeline (6 lookups, window function); T12 "CRITICAL FACT #1: two primary fetches kept on purpose"; single shared `PlineReportData`; "don't modify XSD-backed payload classes"; evidence list of commits `d4c06913`, `d4575fa5`, `c0c6d449`, `bc42674a`, `6961abbf`, `bb82a9c4`, `8d4ecc99`, `b4e588bd`, `4cd1cad2`, `da95d245`, `245c44a4` with the instruction "Check git log --author for your own name".
3. (03) 13 issues (XSD cvc errors, element order, double printing after merge, product/scheme mismatch, date format, Mongo prefix after Boot 4, Jackson 3, ORA-01427 emulation, swallowed exceptions, tie order, null dates...), support playbook, perf ideas, "Plines has just PlineXsdValidationTest", DevOps, security (permitAll; plaintext Mongo URI; SSRF guard "host must equal configured host").
4. (04) STAR seeds: (1) merging T12 into one bean as major contribution, (2) product/scheme mismatch, (3) Boot 4.1.1 upgrade, (4) performance only if measured, (5) collaboration, (6) tight deadline Aug 3-11, (7) requirement change XSD v0.1.6; ~45 Q&A; cheat-sheet numbers.

### 4.2 Agreements with the new pack (safe to keep)
- One endpoint; strategy via bean-name map; handler per subreport; Mongo mirror; XML inside JSON (matches 06, 04, 02).
- ASP EndtDocument is a merged bean; old `AspEndtDocument2Service` has `@Service` commented out (verified in code this session: `AspEndtDocument2Service.java:44`).
- Template12 PolicyDocument2 and Landscape services have `@Service` commented out ("disabled: merged into Template12PolicyDocumentService") - verified.
- Boot 4.1.1, Java 17, Jackson 2 pin, Mongo property prefix change.
- "Gates copied from JRXML"; WIP vs live; ORA-01427 emulation via `requireNoConflict`; shared mutable `rd` (new pack: last-writer-wins, `mergeNonNullFields`).
- Plain-text Mongo URI in `application.properties` is a risk (new pack repeats it; the working tree currently shows `application.properties` modified, do not commit).
- Debit note gate `showDebitNote=Y && prem != 0` for ASP; ASP product 070412.

### 4.3 CONFLICTS and which to trust (the code-supported side)

| # | Topic | Earlier notes say | New pack says | Which is better supported | Action |
|---|---|---|---|---|---|
| 1 | Who built the T12 merge | Story 1 "merging T12 into one bean" as the developer's major contribution; "ported the old execute() bodies verbatim" | `14`: merge commit `c0c6d449a` is by Amarnath Yadav, **not GS**; GS T12 commits are `b4e588bd4`, `f79ecc811` | **New pack** (git author). Also confirmed here: `4cd1cad2e` is Shinge | Do not claim the merge. Claim instead: the `mergeNonNullFields` stax/eInvStatus fix (`f79ecc811`, section 3.1) |
| 2 | Number of T12 primary fetches ("CRITICAL FACT #1") | "Two primary fetches" | Three (PolicyDocument main, PD2 `fetchT12PolicyDocument`, Landscape `fetchDispatch`) - `02`, `07a`, `08a` | **New pack** (later, line-referenced, matches the class javadoc "S12:62-72") | Say "three derivations of product/scheme" |
| 3 | Plines tests | "Plines has just `PlineXsdValidationTest`... mostly manual" | `05_Testing_Deep_Dive` lists many Plines service/handler tests (e.g. AspEndtDocumentService tests); commits by Karunakar/Shaw/Neethu; GS has no `src/test` commit | **New pack** (the test tree exists; commit `db7588897` "Add test coverage for plines.service/client/config/utils") | Do not say "only XSD test". Say "unit tests were written by teammates under OPUSM-44159; my verification was Postman vs JRXML vs Mongo and the XSD test" **[CONFIRM your test role]** |
| 4 | SSRF guard | "host must equal configured host" in `PlinePgRpgeUtilsClient`; client package has 3 classes | `06`: `PlineUtilsClient.assertTrustedHost` using `startsWith(baseUrl)` + `sanitize()`; weakness noted | **New pack** | Use "fixed base URL + assertTrustedHost (prefix check) + sanitize" |
| 5 | Boot upgrade, CVE, Sonar fixes as the developer's stories | Story 3 lists Boot 4.1.1 upgrade as "production-style issue" he lived | `14`: Boot upgrade, CVE pins, Sonar fixes are Shinge's | **New pack** (git) | Tell it only as a team change you reviewed/ran; own your RejectRisk/ASP commits |
| 6 | XSD alignment | "Re-typed payload classes... added XSD test" in the developer's role | `14`: `6961abbf9` and `d4575fa5b` are Shinge; GS did mapping alignment in `0d715fbad`/ASP commits | **New pack** | Say "I aligned my reports' mapping to Pline.xsd v0.1.6 using the master mapping sheet" |
| 7 | Role in tickets | Jira decode lists OPUSM-40850/41763 "PolicyDocument2", 36061/42227 "Multiple CI printing", 40849 "Landscape", 36074/41698-700 Reject Risk, ASP 36054/40884/46666 | `14`: GS commits cite OPUSM-41763, 41698, 36054, 40884 | Mixed: earlier notes come from the developer's Jira list (not visible to the pack); commits corroborate 41763, 41698, 36054, 40884 only | Keep the Jira list but mark the others "from my Jira board" **[CONFIRM]** |
| 8 | Effort split "70% code / 20% test / 10% analysis" | stated as adjustable placeholder | The artifacts show far more analysis/verification (audit prompts, tracker, probes) than "10%" | **Artifacts** | Replace with your real split **[CONFIRM]** |
| 9 | Issue table items marked "[code-evidence]" | e.g. #1, #2, #5 attributed as fixed by him | New pack's list of GS-fixes: CONTRACT_ID String vs Long mismatch, AspDebitNoteHandler timeout, RejectRisk null-bound date, stax shared-mutable bug, ~42 unmapped EndtSchedule fields | **New pack** (these are GS commits) | Lead with the GS-commit defects plus 3.1/3.2 here |
| 10 | Branch name | "feature/pline-spring-boot-4-upgrade-R2-poc" | Current branch `feature/retrofit-develop-updated` (index, git status) | **New pack / git status** | Use the current branch |

Rule of thumb: when two notes disagree, trust (1) `git log` authorship and (2) code lines the new pack cites, over the October-7 prep notes, which were generated before authorship was confirmed (file 02 itself says "Check git log --author for your own name").

---------------------------------------------------------------------

## 5. What these artifacts fill, and what is still missing

### 5.1 Now filled (previously "not in repo")

| Earlier gap | Now covered by | Strength |
|---|---|---|
| Defect notes / known bugs | KNOWN_BUG_staxCount_eInvStatus.txt, CODE_CHANGES_LOG.txt, Data_mismatch_issue.docx | Strong for stax/eInv (also matched in git commit `f79ecc811`); medium for data mismatch (document only) |
| Design / requirement docs | `xsd_changes_prompt.txt` (requirement text for XSD v0.1.6, acceptance criteria), `Pline_MASTER_MAPPING_SHEET`, `ARCHITECTURE.md` | Medium: this is a requirement written by the developer for the AI, not the official Jira text **[CONFIRM relation to OPUSM-44149]** |
| Test evidence / test plan | `contract_id_list.xlsx` tracker, saved `new_responses/*.xml`, RejectRisk kit (`payload.txt`, `mongo_compass_aggregate.txt`, `postman_result.txt`), contract lists | Strong |
| Release / change history | `CHANGELOG.md` (team auto-generated, PRs #855-#864, 2026-09-21), `CODE_CHANGES_LOG.txt` (personal) | Team-level only |
| Working method / tooling | Prompts, `.claude/settings.local.json`, jr-tools | Strong |
| DB "scripts" | Not SQL scripts: `MongoProbe` batch files (`b1-3.txt`), `mongo_compass_aggregate.txt`, SQL snippets inside the docx screenshots, dev inserts into `UWGE_POLICY_BASES_STAX` (described, script not present) | Partial |
| Logs | Only quoted snippets (Spring "Executing aggregation: [...]" lines in `mongo_compass_aggregate.txt`); no server logs | Partial |
| Deployment | `.claude/settings.local.json` (git push allowed); notes "service must actually be restarted, not just rebuilt (stale-deploy)"; jar name `target/gi-documentprintdata-0.0.1-SNAPSHOT.jar` rebuilt 2026-08-19 16:21 (local run) | Local only |

### 5.2 Still missing
- Real Jira text (acceptance criteria, comments, QA/UAT defect IDs); who raised OPUSM-41763/41698/36054/40884; sprint dates; story points.
- AI outputs (the audit reports/findings produced by the prompts), so no count of defects found. Prompt files only.
- The result of the post-fix eInvStatus re-test, the qrInd root cause, T4(692) retest, and the QA/UAT outcome of any of the three defects.
- PR numbers/reviews for GS commits; review comments; Sonar results; deployment/environment promotion dates; rollback evidence.
- Server logs and monitoring; DB change scripts for the dev data inserts (stax rows).
- Who owns `.scratch` scripts (Karunakar/D by git) and whether the developer ran them.
- Evidence of the GS RejectRisk end-to-end tests beyond the saved Postman result (a docx of images exists but is image-only; I did not open its images).
- Which prompt outputs led to actual commits (e.g., `94ea23d81` "~42 fields computed but never copied" looks like the product of an audit prompt **[inference]**; confirm).

---------------------------------------------------------------------

## 6. Twenty interview questions these artifacts make answerable

Answers use only evidence above. `[CONFIRM]` marks gaps.

1. **How did you find a bug where a correctly computed field never reached the response?**
   I separated "data" from "code": I reran the exact Java aggregation logic directly in Mongo and got the right value every time, while Postman showed it blank. That put the fault after the query, in the merge step, `mergeNonNullFields` in `Template12PolicyDocumentBaseService`.
2. **What was the root cause of the `staxCount` loss?**
   The reflection-based merge skipped every field whose type name ends in `List` and only re-merged riskList, sobList and nomineeList. `staxCount` lives inside `staxList.staxPcts`, so once earlier bands had created a `transDet`, the stax row was merged and its list silently dropped.
3. **And `eInvStatus`?**
   Two handlers wrote the same DTO field with different null defaults ("X" vs ""). The merge treats any non-null as meaningful, so the empty string from the second handler overwrote "VALID".
4. **Why did you revert your first fix?**
   My first fix changed the shared handler's default to "X". I then found the handler is called from 11+ templates (Template5 through 22 and several base services), so it would have changed their behaviour, and a sibling MST handler would still write "". I moved the fix into the Template12-only merge step with a guard: a blank status never overwrites a real one.
5. **How did you test the stax fix without suitable production data?**
   In dev I inserted two `UWGE_POLICY_BASES_STAX` rows for contract 35774076 (6% for 01/06/2024-31/08/2024, 8% for 01/09/2024-30/06/2025), called the API and checked `staxCount=2` and `staxAmt2/staxPct2` = 1.20/6.00 and 2.40/8.00 in four blocks. I also fixed a wrong `pTaxInvNo` in my test payload.
6. **How did you record and control code changes?**
   A numbered change log: file, method, before/after, reason, verification, status; and a contract tracker spreadsheet with Done/Blocked per subreport. Each code edit was an explicit, separate decision rather than a bulk change (the log says each change was authorised one at a time).
7. **What remained unresolved?**
   `qrInd` still blank on contract 40799613 (two hypotheses recorded), the QRSection `pTaxInvNo` wiring difference (JRXML uses the row's STAX_INV_NO, Java uses the request parameter), and the T4(692) branch not retested. [CONFIRM later status.]
8. **Describe a case where the data, not the code, was wrong.**
   RiskUploadUnsuccessful returned 0 rows. Oracle and Mongo returned identical results for both tables: upload refs `UWPL17...` (2017-2019) vs policy version refs `UWPL24...`/`25...`, so the join was legitimately empty. I wrote a one-page comparison with screenshots and made no code change.
9. **How do you approach migrating a Jasper report to Java?**
   Treat the JRXML as the contract: list `queryString`, fields, parameters, `printWhenExpression` and `subreportExpression`; build a main/sub/sub-sub hierarchy CSV; implement a handler per subreport and put each subreport call inside its parent's gate; map each column to its XSD leaf through the master mapping sheet.
10. **How did you use AI tools, and how did you control them?**
    I used Claude with structured prompts: comparison prompts ("fan out subagents") to cross-check each JRXML tree against `AspEndtDocumentService` / `AspEndtDocument2Service` / Auto_Policy services; a read-only audit prompt with a field-by-field trace and a fixed issue format; and an impact-analysis-first prompt for the XSD v0.1.6 alignment. I told it not to change code during audits, asked to "reverify and provide only confirmed mismatches", and authorised fixes separately. I verified results myself in code, Postman and Mongo. [CONFIRM what each audit found.]
11. **Which reports did the audit prompts target?**
    ASP EndtDocument tree (11 JRXMLs), EndtDocument2 tree (about 20), AddMemberLandscape and DeletionMemberLandscape trees, and Auto_Policy Successful/Unsuccessful.
12. **How did you drive the XSD mapping?**
    Three steps: generate a JRXML-table-column sheet and hierarchy; match the column to the master mapping sheet (sheet named after the report) to pull XSD attribute and group; derive the code path from a sample response XML. For ambiguous columns, a small PowerShell lookup against the mapping workbook showed which JRXML used the column and which XSD leaf and group it feeds.
13. **How do you limit blast radius when fixing shared code?**
    Every prompt asks "have you changed any common file, will it affect other classes?" and the stax/eInvStatus case shows it: I measured who calls the shared handler before changing it and chose a Template12-scoped fix.
14. **How did you decide a subreport was "done" in testing?**
    The tracker records the gate, a contract that fires it, and whether a real value appeared; "Done" required a real value in at least one block, "gate fired only" was recorded as a weaker state, and subreports with no data dependency were marked "Not confirmed".
15. **How do you find test contracts when conditions are complex?**
    Convert the JRXML query to a Mongo aggregation, add the parent `printWhenExpression` as a filter, find contracts that satisfy both, record them (e.g. 24080537 for add/delete member landscape with endt codes 72/98; 27275348 for all EndtDocument subreports), and, in dev, insert missing column values when no real record exists.
16. **Why did a single contract hide defects?**
    Some branches only fire for specific endorsements: e.g. EndtDocument2's service-tax family fired on contract 3756869 endt `*003`, while EndtTaxDoc/CompanyContactJr/AntiRasuahBanner fired on endt `*005`; and contract 35112689 has 26 versions producing a 102 MB response. I picked a simple single-version contract (35774076, 9 blocks, 555 KB) to rule out size/timeout.
17. **What did you learn about deployments while debugging?**
    The log notes a re-test only counts after a real rebuild and restart ("not just rebuilt"), because stale deployments produced false negatives. Also I should have noted whether the restart happened in each test.
18. **What is your git routine?**
    `git status`, `git diff <file>`, `git pull origin <branch>`, add files individually (or `git add .` when safe), commit with a Jira key (CI check), push to the feature branch. I keep `application.properties` (credentials) and untracked tool folders out of commits. Honest gap: two of my commits lacked a Jira key.
19. **What would you do differently about the reflection merge?**
    Replace "copy all non-null scalars, skip Lists" with a typed merge per field or an explicit strategy per band, and treat empty strings as absent. Add a test that every `List` field in `PlineTransDet` is either handled or intentionally ignored, so the next added list does not vanish.
20. **How did you verify RejectRisk end to end?**
    I picked an Oracle row (e.g. PART_ID 1357766, created 06-Jul-17), built a payload with a date window that contains it (06-06-17 to 31-08-17 / 01-07-17 to 30-09-17), called the API, captured the aggregation Spring logged (Compass), and compared the Postman XML with the Oracle result. Customer name and ID filters used values taken from the same row (`TAN T357766`, `LEE KEN439275`, ID `671207082757`, type C, reason 09).

---------------------------------------------------------------------

## 7. Most important findings (short)

1. The strongest real story is the **staxCount / eInvStatus merge bug**: documented in two files, a numbered change log, and confirmed in git (commit `f79ecc811`, GS, 2026-08-20, and in current code). It includes a good judgment moment (revert the shared-handler change, scope the fix to Template12).
2. The Data_mismatch_issue.docx is a clean "data, not code" proof (Oracle vs Mongo); the code it concerns is `RiskUploadUnsuccessfulHandler`.
3. The earlier prep notes **conflict** with the new pack on who merged T12, how many primary fetches exist (two vs three), the SSRF guard, and whether Plines has tests. Trust git authorship and the new pack's line-referenced claims.
4. The prompts prove an honest AI-assisted method (JRXML-as-contract audits, "fan out subagents", read-only first, fix separately). They do not prove outputs; keep claims to the method.
5. Do not present `.scratch` scripts or the FDAP9F4/YDVFYES path prompts as your own work without confirming; git shows teammates committed the scratch scripts.
6. Still missing: real Jira/QA/UAT outcomes, deployments, logs, DB change scripts, and the post-fix re-test results.
