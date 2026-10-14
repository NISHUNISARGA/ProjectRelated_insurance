# OPUS Document Print Data Service — Knowledge Transfer Guide

Focus modules: **PLine Template12**, **ASP**, **RejectRisk**
Repo analysed: `opus-documentprintdata-service` (branch `feature/retrofit-develop-updated`, main branch `retrofit-develop-R1`)
Analysis date: 2026-10-13. History in the repo runs up to 2026-10-08.

---

## 0. Read this first (honesty notes)

This guide is built only from what is in the repository: source code, `pom.xml`, `application.properties`, `README.md`, `ARCHITECTURE.md`, `CHANGELOG.md`, `plines/README.md`, the test XSDs, and git history. I did not have Jira, Confluence, functional design documents, DB scripts, logs or deployment scripts. Where something is not in the repo, I say "not in repo" instead of guessing.

Things you must know before an interview:

1. **Authorship (updated):** your git author is `GS <extern.gs_nisarga@allianz.com>` (you confirmed). Your 11 commits (8 code + 3 merge) are mapped to modules in `KT_Parts/14_My_Commits_And_Ownership.md`. Claim those; do not claim commits by Shinge, Karunakar, Amarnath Yadav, Shaw, Neethu and others. No GS commit touches `src/test`, so your unit-test work must be confirmed from your own memory or another branch.
2. **"ASP" is not expanded anywhere in the code.** The code calls it "ASP product" / "ASP plan" with product config code `070412`. If you know the real expansion from the business, use it. I will not invent one.
3. **Several things in your brief do not match the code.** Template12 PolicyDocument2 and Landscape, and ASP EndtDocument2 / AddMemberLandscape / DeletionMemberLandscape, are *no longer separate reports*. They were merged into one report each (details below). This is a great interview talking point.
4. **There is no real Template12 or ASP XSD-conformance test.** The XSD validation test uses a hand-written minimal XML. Be honest about this if asked.
5. **Security hygiene:** `src/main/resources/application.properties` currently has an *uncommitted* change that adds a plaintext MongoDB username and password (`spring.mongodb.uri=...`). I did not copy it here. Do not commit it. Use an environment variable or secret store. (`gitleaks.toml` exists in the repo for this reason.)
6. The project window you gave (12 May – 30 Dec 2026) extends past today's date and past the last commit (2026-10-08). I only see history up to that point.

7. **Update after reading the Jasper dumps, your XSD folder and the deep-dive agents (read these facts carefully):**
   - Original reports: `C:\Users\EPGRVFY\Documents\jasper_dumps\jasper_02072026\resources\reports\AGIC\UW\Plines\` (Template12 under `PolicySchedule\Template12`, ASP under `ASPEndtSchedule`, RejectRisk under `RejectRisk`). The print conditions in the jrxml match the Java gates described below.
   - The XSD you gave (`Plines LOB v0.1.6`) is **identical** to the repo test copy (line endings aside). No XSD newer than v0.1.6 exists anywhere on this machine, but Java comments talk about v0.1.7–v0.1.10. So the Java output has elements (`mainReportName`, `subReportName`, `pCreatedDateFrom`, `pCreatedDateTo`, ...) that the v0.1.6 XSD does not know.
   - **Real Java output does not validate against v0.1.6.** An agent validated two saved RejectRisk responses and got hundreds of errors (dates written as `dd/MM/yyyy` while the XSD needs `xs:dateTime`; empty elements for numeric/date fields; sibling order differences such as `reportParameters` being last). A hand-ordered XML validated with 0 errors. So do **not** claim "our output is XSD-valid". Say: "We aligned the payload types and structure to the XSD; full end-to-end validation of service output was still open; the existing XSD test only validates a minimal shell." Details: `KT_Parts\04_XSD_XML_Deep_Dive.md`.
   - Your RejectRisk uncommitted fix equals commit `cc6a1b84d` on another branch (OPUSM-41763) and will likely break 7 of the 15 handler tests. Details: `KT_Parts\01_RejectRisk_Deep_Dive.md`.
   - Mapping workbooks exist: `C:\Users\EPGRVFY\Documents\Pline_MASTER_MAPPING_SHEET_v0.1 (final1) 1 5.xlsx` and `...\Plines_Base_code\T12_final_code\RejectRisk\RejectRisk\RejectRisk_XSD_Mapping_sheet.xlsx`. Read them before the interview; they are the "requirement" side of your XSD work.
   - Separate part files with the deep detail: `KT_Parts\01_RejectRisk_Deep_Dive.md`, `02_Template12_Deep_Dive.md`, `03_ASP_Deep_Dive.md`, `04_XSD_XML_Deep_Dive.md`, `05_Testing_Deep_Dive.md`, `06_Config_API_DB_DevOps.md`.

---

# PHASE 1 — PROJECT OVERVIEW

## 1.1 What the project is

`opus-documentprintdata-service` (Maven artifact `gi-documentprintdata`, Spring app name `Documentprintdata`) is a **report data provider microservice** for Allianz's **OPUS** general-insurance platform (Malaysia market: SST/GST tax, BRN company numbers, e-Invoice).

It does **not** make PDFs. It receives a request like "give me the data for report X for policy Y", collects data from MongoDB and a few OPUS REST APIs, and returns **XML inside a JSON response**. A downstream print/Jasper engine turns that XML into the PDF.

Evidence: `ARCHITECTURE.md` section 1: "receives a report name and filter parameters, fetches data from MongoDB (mirrored from Oracle), enriches it via external API calls, and returns the assembled data as an XML string inside a JSON response."

## 1.2 Why it was built (the business problem)

The old OPUS system produced printed documents with **JasperReports `.jrxml` files that ran Oracle SQL directly**. The company is *retrofitting* OPUS (branch names: `retrofit-develop-R1`, `...R2-pline...`). The Oracle data is now mirrored into **MongoDB** (collections have the same uppercase names as the Oracle views, e.g. `UWGE_POLICY_VERSIONS`, `UWPL_VI_POLICY_SCHEDULE_TPL4`). So the SQL inside each jrxml had to be rewritten as MongoDB aggregation pipelines inside a new Java service.

Proof: `plines/README.md` has a table "Oracle source → MongoDB strategy" (e.g. `UWMT_VI_ENDT_TAX_INV_ALT_UPV` → primary collection; Oracle functions like `FN_DECIDE_MT_CERT` → REST calls through `UtilsClient`). Many handler javadocs quote the original Oracle SQL they replace.

## 1.3 Who uses it

- **Direct caller:** an OPUS print/report orchestrator (not identifiable from this repo — README says so explicitly).
- **End business users (indirect):** underwriters, policy servicing and operations staff who print policy schedules, endorsements, debit/credit notes, tax invoices, reject-risk listings; customers who receive those documents.

## 1.4 Major modules (by LOB package under `com.agcs.fo.opus`)

| Package | Meaning | Notes |
|---|---|---|
| `motor` | Motor insurance | Original module. Contains `CommonController` and shared infrastructure |
| `fire`, `marine`, `bond`, `workmen`, `miscellaneous` | Other lines of business | Same pattern |
| `plines` | **Personal Lines** (PL) — your area | ~422 handlers, ~377 DTOs, ~302 request-payload classes, 140 services |
| `SharedReport`, `generic`, `dmt` | Shared listings/reports | |

"Plines" = Personal Lines (the `LOB=PL` filter appears in `AspEndtScheduleHandler`).

## 1.5 Main workflow

```
Print system  --POST /gi/documentprintdata/api/opus/GenerateReport-->  CommonController
   body: { metadata, filterCondition: { reportName, pContractId, ... } }
CommonController -> picks a ReportStrategy bean whose name == reportName
Service.execute() -> calls Handlers (MongoDB aggregations) + REST clients
Service maps DTOs -> XSD-shaped payload objects (Transactions/Transaction/reportData...)
XmlConverter -> XML string -> wrapped in SuccessResponse { status, xmlData, timestamp }
```

## 1.6 Key business rules (seen in your three modules)

- Print a sub-report **only if** certain flags are true (e.g. service-tax invoice prints only if `showTaxInvoice=Y` AND `SHOW_STAX_INV_IND=Y` AND `checklistCopy != Y` AND `SHOW_EINV_IND=Y`).
- Behaviour depends on **product code** (`070504`, `070412`, `070406`, `070414`, `100908`, `070521`).
- Data may be in **WIP** (work-in-progress, not yet issued) *or* in the **posted/live** collections. The code checks both.
- Dates before/after a **cut-off** (`ASP_CUTOFF_DATE`) change what is printed.

## 1.7 Interview-ready explanations

**30 seconds**
"It's a Spring Boot microservice in Allianz's OPUS insurance platform. It gives report data for printed documents like policy schedules, endorsements and tax invoices. It reads MongoDB, which mirrors the old Oracle DB, calls a few OPUS APIs, and returns XML for the print engine. I worked on the Personal Lines reports: Template12, ASP and RejectRisk."

**1 minute**
"Allianz is retrofitting OPUS. Printed documents used to be Jasper reports with Oracle SQL inside. We re-implemented the data side as a Java service. One endpoint, `POST /api/opus/GenerateReport`, takes a `reportName` and filters. Spring finds a service bean whose name equals the report name. That service calls handlers, which run MongoDB aggregation pipelines equivalent to the old SQL, plus REST calls for things Oracle computed with functions, such as e-invoice indicators. The result is mapped to an XSD-defined structure and returned as XML in JSON. My modules were Template12 (a policy schedule template), ASP (a group personal-accident-style product's endorsement documents) and RejectRisk (a listing of rejected risks)."

**3 minutes** — add: the layered design (Controller → Service → Handler → DTO → RequestPayload), why handlers are per sub-report (each Jasper subreport = one handler), the WIP-vs-live data rule, the product-code branching, the XSD alignment work (one subreport = one `reportData`; `mainReportName`/`subReportName`), and the consolidation of several reports into one bean to avoid double rendering.

**5 minutes** — add: concrete examples. ASP: service-tax gate and the CONTRACT_ID type-mismatch bug. Template12: three logical reports merged, 16 landscape bands, why Template12 has its own base toolkit. RejectRisk: date-range semantics and the `$setWindowFields` row number.

**10 minutes** — add: tech stack and the Boot 3.5 → 4.1.1 upgrade with CVE-driven pins (Phase 5), testing/coverage push (OPUSM-44159), known limitations (God-method services, no auth, always-200 responses), and what you would improve.

---

# PHASE 2 — BUSINESS DOMAIN KNOWLEDGE (from zero)

## 2.1 Insurance vocabulary used in the code

| Term | Plain meaning | Where you see it |
|---|---|---|
| **Policy** | The insurance contract | `UWGE_POLICY_CONTRACTS`, `pPolicyNo` |
| **Contract ID** | Internal numeric id of the policy | `pContractId` filter |
| **Version** | Each change creates a new version of the policy | `UWGE_POLICY_VERSIONS`, `versionNo` |
| **Endorsement (Endt)** | A mid-term change to a policy (add a member, delete a member, change premium) | `pEndtNo`, `ENDT_CODE` (e.g. 72, 168 = add; 98 = delete) |
| **Schedule** | The document listing the details/benefits of the policy | "PolicySchedule", "EndtSchedule" |
| **Risk** | The thing/person insured | `UWGE_RISK`, `RejectRisk` |
| **Cover / Benefit** | What is paid for which event | `UWGE_COVER`, `sobList/benefits` |
| **Premium** | Price paid | `PREM_DUE`, `GROSS_PREM` |
| **Debit note / Credit note** | Extra amount to pay (D) / refund (C) after a change | `AspDebitNoteHandler`: D if `PREM_DUE >= 0`, else C |
| **Tax invoice** | Official invoice. GST era earlier, **SST** (Sales & Service Tax) later | `SHOW_GST_INV_IND`, `SHOW_STAX_INV_IND`, `sst-rollout-date` |
| **e-Invoice / QR** | Malaysian electronic invoice with QR code | `SHOW_EINV_IND`, `EINV_CONSENT_IND`, `QRSection` |
| **BRN** | Company business registration number | `compBRN`, `comp-brn-number` API |
| **WIP** | Work in progress — not yet issued/posted | `WIP_UWGE_POLICY_VERSIONS` etc. |
| **Checklist copy** | A draft copy printed for checking before issue | `checklistCopy=Y/N` |
| **Master policy (MST)** | Group policy that individual certificates belong to | `MST_POL_NO`, `PREM_BY_MST_IND` |
| **Landscape / Portrait** | Page orientation of a sub-document. Member lists are wide, so landscape | `AddMemberLandscape`, `...Potrait_ASP` |
| **Scheme** | A named plan variant under a product | `SCHEME_NAME` (ASTRO, BMW, POSCOVER...) |
| **PIDM** | Malaysian deposit/insurance protection notice printed on some policies | `PIDM_PRINT` flag in Template12 |
| **PDPA** | Personal data protection notice | `SHOW_PDPA_NOTICE_IND` |
| **Complaint Redress** | A standard section telling the customer how to complain | `ComplaintRedress` subreport |
| **Anti-Rasuah banner** | Anti-corruption notice on invoices | `PlineGenericAntiRasuahBannerHandler` |
| **TPA** | Third-party administrator (claims handler) | `TPA_CUTOFF_IND`, `TPA_NAME` |

## 2.2 Stakeholders (what the repo shows)

- Report developers / retrofit team (this team).
- Report/Jasper team — the consumer. Code comments say sign-off from the report team is needed (see RejectRisk disclaimer).
- Platform/API team who own `/ext/utils`, `/ext/policy`, `/ext/customer` endpoints.
- DevOps (`.github/CODEOWNERS` → `@OPUSCCC/cloud_engineers_devops` for workflows).
- Business/UAT testers (Changelog mentions "unable to reproduce on BAU UAT", OPUSM-52530).

## 2.3 A user journey, end to end (example: ASP endorsement)

1. An underwriter adds two members to an ASP group policy. OPUS creates endorsement `ENDT_NO` with `ENDT_CODE` 72 and a premium difference.
2. The user clicks "Print endorsement". The print system calls our service with `reportName=AGIC/UW/Plines/ASPEndtSchedule/EndtDocument`, `pEndtNo`, `pContractId`, `showEndtSchedule=Y`, `showDebitNote=Y`, ...
3. Our service decides which pages exist: cover note, schedule, debit note, service-tax invoice, add-member landscape page listing the new members.
4. It returns one XML with many `reportData` blocks, one per page type.
5. The print engine renders the PDF.

---

# PHASE 3 — SPECIAL MODULE ANALYSIS

## Common design used by all three modules

- **Strategy pattern.** `ReportStrategy` interface (`motor/strategy`). Each service is `@Service("<jasper-like path>")`. Spring injects `Map<String, ReportStrategy> reportStrategies`. `CommonController` does `reportStrategies.get(reportName)` (CommonController.java:171, 203). Old reports still use the big legacy `switch`.
- **Handler per sub-report.** `@Component` handler classes with `MongoTemplate` raw `Document` aggregation stages.
- **DTO → request payload.** Handler returns a DTO; service maps to `Pline*` payload classes that mirror `Pline.xsd`.
- **XSD rule:** "one subreport = one `reportData`". Each `reportData` has `prntDetails/transDetList/transDet`. `PlineReportNames.tag(...)` fills `mainReportName`; services set `subReportName` per section (ASP does; Template12 does not).

---

## MODULE A — PLine Template12

### A.1 Purpose
- **Business:** Template12 is a *policy schedule* document template ("PolicySchedule/Template12/PolicyDocument") used by several PL products. It covers the policy schedule with product-specific pages, plus invoices, plus landscape member/benefit lists.
- **Technical:** assemble all those pages' data into one XSD-shaped response.

### A.2 Important correction
Only **one** bean is live:
`@Service("AGIC/UW/Plines/PolicySchedule/Template12/PolicyDocument")` — `Template12PolicyDocumentService.java:103`.

It does three logical reports in one call:
1. **PolicyDocument** itself (CIPrint variants, footer, complaint redress) — `execute()`.
2. **PolicyDocument2** (nominations, installments, ATC, debit note, tax invoices, PDPA, PDS) — private `executeMergedPolicyDocument2` (lines ~578–779).
3. **Landscape** (16 bands) — private `executeMergedLandscape` (lines ~801–1497).

`Template12PolicyDocument2Service` and `Template12LandscapeService` still exist but their `@Service` is commented out ("kept for reference"). Re-enabling would render every band twice. Commit **c0c6d449a** (2026-08-03, Amarnath Yadav, "Added Template12 dynamic merging") did the merge. Old report names now have no bean — a request to them would fall into the legacy fallback.

### A.3 Inputs (`filterCondition`, defaults) — `Svc:131-154`
`pContractId ""`, `pPolicyNo ""`, `versionNo "1"`, `companyParamType ""`, `checklistCopy "N"`, `showCoverNote "N"`, `showComplaint "Y"`, `showScheduleStd "Y"`, `showDebitNote "N"`, `showTaxInvoice "N"`, `showPaymentInfo "N"`, `printSeries`, `copyType`, `pCertInd`, `ShowOutPrintCovLetter "N"` (capital S), `pType`, `pFirstPageNumber`, `pMstPolicyNo`, `pContractIdList`, `pCertNoFrom`, `pCertNoTo`, `pTaxInvNo`, `pGstTaxInvoiceNo`.

Quirks: `pMstPolicyNo`, `pContractIdList`, `ShowOutPrintCovLetter` are read but not emitted to `reportParameters` (no home in Pline.xsd v0.1.6). The merged PD2 hardcodes `certNo=""` (Svc:583) — the old standalone read a `certNo` filter. That is a behaviour change.

### A.4 Flow (Svc:130-556)
1. Create one shared header `reportData` (`newSharedReportData`).
2. Primary fetch: `PlinePolicyDocumentMainHandler.fetchTemplate12PolicyDocumentMain` → product, schemeName, aspCutoffInd, issueDate, signIssueDate, dependantCnt.
3. Flags: `is070504`, `is070412` (ASP!), `is070406`, `is070414`, BMW, POSMESRA, FWPA, POS-cover-with-dependants.
4. **Part A — PolicyDocument bands:** PrintSeriesFooter (or OutSourcePrintCoverLetter if `ShowOutPrintCovLetter=Y`), then by product:
   - **070406** → `CIPrintAtc` + nested (Note, ComplaintRedress_NoMargin, PerilsClauses, Signature, PIDM, BNMeMarketing).
   - **Default** → `CIPrint` primary + nested bands (RiskDetails, Coverage, InstallmentDetails for 070521, Note for 070412, ScheduleOfBenefits_ASP for 070412, ScheduleOfBenefits_AMC for 100908, PerilsClauses, PIDM, ASP portrait pages, ImportantNotes...).
   - **070504** by scheme: ASTRO, BMW (one row *per certificate* between `pCertNoFrom..pCertNoTo`), POSCOVER, AZP1/AZP2, POSMESRA, FWPA, Elite fallback.
   - **070414** (PA4A): one set of bands per row.
   - ComplaintRedress rules: plain version when `showComplaint=Y` and not BMW/070504/070406 and (not ASP or `ASP_CUTOFF_IND=N`); SCB version for 070504.
5. **Part B — PolicyDocument2:** own primary fetch `PlineT12PolicyDocumentHandler`, then ParticularOfInsured, Nomination, InstallmentDetails, Domestic/Overseas ATC (by `JOURNEY_AREA`), Complaint (POSMESRA/FWPA), debit note (if `showDebitNote=Y`, not BMW, copyType != `BC`), GST tax invoice, service-tax invoice (needs `showTaxInvoice=Y`, `showStaxInvInd`, `einvPrintInd`), QR section, transitional invoices, PDPA, PDS_Income/Elite.
6. **Part C — Landscape:** `Template12LandscapeDispatchHandler.fetchDispatch`. If null (excluded by `CMGE_CODEREL_DET` `CI_LANDSCAPE_PRT`), nothing prints. Otherwise 16 bands, each its own `reportData`: ASP insured/premium/sponsor (070412 and `ASP_CUTOFF_IND=N`), Benefits, Benefits_Elite/Income, Premium_Elite/Income, Nomination variants (Elite, POS, generic, AZP), Diagnosis, Insured_AMC (100908), Diagnosis_AMC, and a cross-template call to Template4 dispatch for 070406 with promo amount.
7. Finish: `buildTransactions` → `Transactions > Transaction > PlineTransactionDetails > PlineHeaderTransaction(dataclassH="PlinePolicyDocument") > ReportTransaction > reportData[]`. Report-level leaves (`parentName`, `planDescp`, `psi`, `pmtDueDate`...) are set with precedence PD > Landscape > PD2. Dates are reformatted reflectively to `dd/MM/yyyy` (Svc:549-554). `mainReportName` = `"Template12PolicyDocument"`.

### A.5 Why two primary fetches? ("CRITICAL FACT #1", Svc:77-87)
- Fetch 1 throws if WIP and live data disagree (copies Oracle error ORA-01427 "single-row subquery returns more than one row"). Does not use `checklistCopy`.
- Fetch 2 prefers posted data, falls back to WIP with `$ifNull`, no conflict detection, calls REST for `EINV_PRINT_IND` and `SHOW_STAX_INV_IND`.
- They were deliberately not unified because that would change PD2's behaviour without product sign-off. Trade-off: 3 fetches that derive product/scheme (`fetchTemplate12PolicyDocumentMain`, `PlineT12PolicyDocumentHandler`, `Template12LandscapeDispatchHandler`) and can disagree.

### A.6 Design decision: own base class
`Template12PolicyDocumentBaseService` (3,732 lines) does **not** extend `PlinePolicyDocumentBaseService` (4,083 lines, shared by ~24 templates). Its javadoc (Base:42-60) says it is a sibling toolkit so Template12 changes cannot break other templates, and because Template12 uses the new XSD-typed envelope. Cost: duplication (e.g. `fetchCIPrintFwpaResilient` duplicates `PlineCIPrintFwpaPrimaryHandler`, added because the Mongo view `UWPL_VI_POLICY_SCHEDULE_TPL4` throws when a rebate amount is null).

`mergeSingleTransDet` (Base:189) merges non-null fields onto every existing `transDet` using reflection, to avoid multiplying rows across bands. Multi-row bands use `addSingleTransDet`.

### A.7 Data sources (collections)
- Main: `WIP_UWGE_POLICY_CONTRACTS`, `SB_UWGE_POLICY_CONTRACTS`, `WIP_UWPL_POLICY_BASES`, `UWPL_POLICY_BASES`, `UWPL_RISK_PERSON`, `WIP_UWPL_RISK_PERSON`, `UWGE_VI_POL_REPORT_PARAM`, `SAPM_SYS_CONSTANTS` (`ASP_CUTOFF_DATE`).
- Coverage: `UWPL_VI_POLICY_SCHEDULE_COVER` ($group emulates SELECT DISTINCT; numeric sort via `$toDouble(SORT_SEQ)`).
- Installments: `UWGE_VI_POLICY_INSTALLMENT` (excludes `PMT_FREQUENCY` ANNUALLY — annual payers have no instalment table).
- Risk details: `UWPL_VI_POLICY_SCHEDULE_RISK`.
- Note: `UWGE_VI_POLICY_SCHEDULE_BASE`, `OCP_POLICY_BASES`, `UWGE_POLICY_BASES`, `CPGE_VI_PARTNERS_ALL` (master policy holder; throws if more than one combination exists).
- Signature/company: `SAPM_COMPANY` + REST.
- Landscape dispatch: `WIP_UWPL_..NOMINEE`, `..COVER_DIAGNOSIS`, `CMGE_CODEREL_DET`. Bands read `UWPL_VI_POLICY_SCHEDULE_TPL4 / _BEN / _PREM / _NOM / _DIAG / _TPL20`.
- Debit note: `UWPL_VI_PREMIUM_DEBIT`; master customer: `UWPL_VI_TAX_INVOICE`.

### A.8 External APIs (via `PlineUtilsClient` etc.)
`invoice-print-indicator` (`opus.fn-einvoice-print-pol-ind-path`), `stax-inv` (`plines.fn.show.stax.inv.ind.path`), `comp-brn-number`, `fn-rppm-get-signature`, `sst-rollout-date`. Failures are caught and defaulted, so the report still returns.

### A.9 XSD
No Template12-specific elements; it uses the generic PolicyDocument family in `Pline.xsd` (1,400 lines) and `Common.xsd` (748 lines) in `src/test/resources/xsd/plines/`. Mapping rules: cover/benefit/nominee/diagnosis go into nested lists; company reg no → `compInfo.compBRN`; `source`, `partnerType`, `contractId` → `endorsementDetails`.
XSD version drift: code comments cite Pline.xsd v0.1.10 for `mainReportName/subReportName`; the test copy ends at v0.1.6 and does not contain those elements.
Relevant commits: `d4575fa5b` (2026-08-03 XSD changes for Template 12 and Template4 landscape), `6961abbf9` (2026-08-08 align payload types with XSD + schema validation test), `edbef120c` (multiple reportData changes), `e74bc0d5c` (report/sub-report name).

### A.10 Validations
- Business: product/scheme gates, `ASP_CUTOFF_IND`, `CI_LANDSCAPE_PRT` exclusion, e-invoice and tax flags.
- Technical: `Long.parseLong(contractId)` in landscape dispatch — empty `pContractId` throws `NumberFormatException` (not caught; test `execute_landscapeDispatchHandlerThrows_notWrappedInTryCatch_propagates`). WIP-vs-live conflict throws `IllegalStateException` (unguarded at Svc:166). Band failures elsewhere are swallowed with `log.warn`, giving a partial report.

### A.11 Alternatives, advantages, limitations
- Alternative: one class per band/report (as before the merge); or a config-driven band table instead of if/else; or unify the three product/scheme fetches.
- Advantages: one response with all pages; isolation from other templates; each Oracle behaviour preserved.
- Limitations: `execute` + helpers are huge; reflection-based merging; duplicated landscape mapping (~700 lines); unregistered duplicate classes still tested; orphaned `Template12PolicyDocumentHandler`; leftover hardcoded test ids in comments ("6063333", "2885273"); XSD date format mismatch (`dd/MM/yyyy` strings vs `xs:dateTime`).

### A.12 Interview Q&A
1. *Why one bean for three reports?* To avoid duplicate rendering and give the print engine one response. Merge commit c0c6d449a; old classes kept unregistered.
2. *Why two primary fetches?* Different Oracle semantics (conflict-throwing vs posted-first). Unifying would change PD2's behaviour.
3. *How is Landscape suppressed?* `CMGE_CODEREL_DET` lookup (`GPA_SCHEME`, `CI_LANDSCAPE_PRT`, `Y`, header = scheme); if matched, dispatch returns null.
4. *How do you choose pages per product?* Flags from product code/scheme in `execute()`.
5. *What is `mergeSingleTransDet`?* Reflection merge of non-null fields into existing rows to avoid row explosion.
6. *Why not extend the common base?* ~24 other templates depend on it and use the legacy flat shape.
7. *Instalments?* `UWGE_VI_POLICY_INSTALLMENT`, excludes ANNUALLY, sorted by `SEQ_NO`, first due date becomes `pmtDueDate`.
8. *Failure handling?* Sub-bands swallow; primary and landscape dispatch propagate.
9. *Biggest risk?* Date format vs XSD, three divergent product fetches, no true XSD test.
10. *What would you refactor?* Table-driven band registry, shared product-context object, real XSD validation of service output.

---

## MODULE B — ASP

### B.1 Purpose
- **Business:** documents for the ASP product (config code `070412`): endorsement document, cover note, schedule, debit/credit note, service-tax invoice, and landscape pages for added/deleted members, sponsor and nominees. Also ASP pieces of the policy schedule (via Template12) and the auto-renewal letters.
- **Meaning of ASP:** not stated in code (unknown). Data hints (inference only): a group personal-accident-style product with benefits AD/PD/PSI, sponsor, nominee, parent/subsidiary/department, occupation groups `PA_CP`, `PA_CP_EVO`, relationship category `PA_NOMREL`. `STI` and `LLB` appear as plan codes, also unexplained.
- ASP is controlled by `SAPM_SYS_CONSTANTS`: `ASP_CUTOFF_DATE`, `ASP_CONFIG_DATE`, `ASP_LAUNCH_DATE`, `ASP_POLICYHOLDERLETTER_DATE`, and flag `TPA_CUTOFF_IND`.

### B.2 Report names
Live: `@Service("AGIC/UW/Plines/ASPEndtSchedule/EndtDocument")` — `AspEndtDocumentService.java:165`.
It is a **merged bean** that also produces what used to be `.../EndtDocument2`, `.../AddMemberLandscape`, `.../DeletionMemberLandscape` (annotations commented out; classes `AspEndtDocument2Service`, `AspEndtDocumentLandscapeService` kept for reference — re-enabling would clash bean keys and render twice). `reportName`-based dispatch in the landscape class (`contains("DeletionMemberLandscape")`) is dead code now.
Other ASP pieces are sub-flows of: `AutoRenewalDocumentService` (`AGIC/UW/Plines/AutoRenewal/AutoRenewalDocument`, ASPInsuredDetails and letters ASP_EG / ASP_MY when product `070412` and `tpaCutoffNum > 0`), and `Template12PolicyDocumentService` (portrait ASP pages, landscape ASP bands).

### B.3 Inputs (`AspEndtDocumentService.execute`, lines 215-230)
`pEndtNo, pRefNo, pContractId, companyParamType, versionNo, checklistCopy, printSeries, copyType, pUserID, pPolicyNo` default `""`; `pFirstPageNumber "1"`; `showCoverNote/showEndtSchedule/showComplaint/showDebitNote/showTaxInvoice` default `"N"`.
`showComplaint` is only echoed (ComplaintRedress is decided by product `070412`). `pPolicyNo` is unused.

### B.4 Flow
1. Main handlers: `AspEndtDocumentMainHandler` (productCode, labels, issueDate, endtCode) and `AspEndtDocument2MainHandler` (tax flags, `showEinvInd`, `prem`, `taxInv`).
2. Always: print-series footer count, Blank.
3. `showCoverNote=Y`: `AspEndtCoverHandler` + company name with GST reg no.
4. `showEndtSchedule=Y`: `AspEndtScheduleHandler` (840 lines) + signature.
5. `endtCode` 72 or 168: schedule-of-benefits/LLB/STI handlers — currently **stubs** returning `SELECT 1 FROM DUAL`-style dualValue; nothing mapped.
6. Product `070412`: ComplaintRedress (result discarded) + company name BRN.
7. `showDebitNote=Y && prem != 0`: `AspDebitNoteHandler`; if `premByMstInd=Y` also master customer details.
8. `showTaxInvoice=Y && showGstInvInd=Y`: EndtTaxDoc + AntiRasuah banner.
9. **Service tax invoice** if `showTaxInvoice=Y && showStaxInvInd=Y && checklistCopy != Y && showEinvInd=Y` (line ~335): `AspServiceTaxInvoiceHandler`, company contact, EndtMT/EndtMTSub rows when `staxCount > 1`, QR section when `einvConsentInd=Y && einvStatus=VALID && qrInd=Y`.
10. Transitional invoices if `showTaxInvoice=Y && showTranInvInd=Y`.
11. **Landscape flows run unconditionally**: `AspAddMemberLandscapeDispatchHandler` returns counts (`additionPrint`, `deletionPrint`, `nomineePrint`, `sponsorPrint`). Then insured + premium (if additions), sponsor, nomination, and delete-member landscape.
12. Response: `reportData[0]` is master row (`mainReportName="ASPEndtScheduleEndtDocument"`, `subReportName="PrintSeriesFooter"`), then each section in order with its own `subReportName` (e.g. `AddMemberLandscape_ASPInsured`, `AddMemberLandscape_ASPPremium`, `EndtParticularOfSponsorLandscape_ASP`, `EndtParticularOfNominationLandscape_ASP`, `DeleteMemberLandscapeDispatch`, `DeleteMemberLandscape_ASP`).

### B.5 Data sources and rules (highlights)
- Main: `UWMT_VI_ENDT_TAX_INV_ALT_UPV`, `UWGE_VI_ENDT_REPORT_PARAM`, `WIP_/SB_UWGE_POLICY_CONTRACTS`, `UWGE_/WIP_UWGE_POLICY_VERSIONS`.
- Cover: `UWPL_VI_ENDT_SCHEDULE_SUMMARY`, agents/branch chain `DMT_AGENTS → DMAG_AGENTS → CMDM_BRANCH`, `CMGE_CODE`.
- Schedule: queries `UWGE_POLICY_VERSIONS` directly + `$unionWith WIP_UWGE_POLICY_VERSIONS`, bypassing the summary view because the view applies a VERSION_NO ceiling. Rules: `CI_IND="Y"` if `MST_POL_NO` not null; GPA refund = round(match − gross − stax − stamp).
- Debit note: `UWGE_VI_ENDT_DEBIT`; note number `D`/`C` + `ENDT_NO` by sign of `PREM_DUE`; extra fee diffs added to gross premium for product codes 089332, 089432, 089329, 089429.
- Service-tax invoice: `UWPL_VI_ENDT_TAX_INV`, master TIN masked by keeping all but the last 4 characters and replacing those 4 with `****` (the opposite of "show last 4"), `computeStaxCount` from WIP and live tax tables, `requireNoConflict` mimics ORA-01427.
- Landscape: `UWPL_VI_ENDT_SCHEDULE_TPL4`, `_NOM`, `_SPONSOR`, `UWPL_VI_ENDT_ADD_DEL_RISK`. Addition needs `ENDT_CODE` 72/168 + risk with `ACTION_CODE=A`; deletion `ENDT_CODE` 98 + `ACTION_CODE=C` and `TERMINATE_DATE` not null. Aggregations use `allowDiskUse` and `maxTime` 600 s.

### B.6 External APIs
`PlineUtilsClient.fnEinvoicePrintEndtInd` → `/ext/utils/api/v1/pg-rpge-utils/invoice-print-endt-indicator`; policy-ref (`plines.policy.ref.path`); SST rollout date; company BRN; QR indicator. `safeCall` returns null on failure; QR defaults to `N`.

### B.7 Notable bug fix to quote
**CONTRACT_ID type mismatch** (commits 578778080, ba9e29586, OPUSM-41763 — on other branches, not HEAD): the shared base handler compared a String `CONTRACT_ID` with numeric data in Mongo (which is type-strict), so tax flags came back null and tax sub-reports silently never printed. `AspEndtDocument2MainHandler` overrides `buildCommonPipeline` and uses `safeNum` (`Long.parseLong`). ASP-only fix.

### B.8 Other commits (as reported by git)
`4cd1cad2e` (OPUSM-42246, 2026-08-31) working Plines code; `7a925920c` (OPUSM-36054/40884) wire ASP EndtDocument sub-reports and fold Landscape into one response; `94ea23d81` (2026-09-21) EndtCover/EndtSchedule mapping gaps and `AspServiceTaxInvoiceHandler`; `cc6a1b84d` (OPUSM-41763, 2026-10-05) RejectRisk/EndtSchedule/EndtDocument fixes on branch `OPUSM-41763/retrofit-develop-R2-pline-Template12-ASP-RejectRisk` — **this branch is not in HEAD**; `c336f76fa` sonar fixes; `e74bc0d5c`/`861f0666c` report and sub-report names. Check with your team which of these are merged to your target branch.

### B.9 Limitations
`AspEndtDocumentService` is 1,701 lines; `execute` ~1,300 lines. Landscape dispatch queries run on every call. MST resolution can run up to 3 times. Hardcoded literals (`070412`, `72`, `168`, `98`, `999`). Six fields (the ASP deep-dive found `endt*` leaves under `endorsementDetails`, so check whether they are really unmapped: see KT_Parts/03 and 11a T036) (`subcoverPrem, endtDoubleIndemnity, endtImaFee, endtStaxAmt, endtLlbPrem, endtStiPrem`) are fetched but have no XSD leaf, so are not output. A near-copy `AspServiceTaxInvoiceHandler.java` sits in the **test** source tree (added in 3f2157b28) — it can shadow the real class on the test classpath. Tests for the disabled `AspEndtDocument2Service` / `AspEndtDocumentLandscapeService` do not exercise the live bean. README has no ASP section.

### B.10 Tests
`AspEndtDocumentServiceTest` (977 lines, 25+ tests: gates open/closed, endtCode 72, product 070412, checklistCopy=Y closes service-tax gate, one reportData per subreport in order), ~24 ASP handler tests, DTO/payload tests. No test for `PlineParticularOfPremiumPotraitAspHandler` or `PlineParticularOfSponsorPotraitAspHandler`.

### B.11 Interview Q&A
1. *One bean for four Jasper reports?* Merged to give one response; each original `printWhenExpression` kept as a gate.
2. *Which params control sub-reports?* `showCoverNote, showEndtSchedule, showDebitNote, showTaxInvoice`, `checklistCopy`; ComplaintRedress by product code.
3. *When is service-tax invoice printed?* See B.4 step 9.
4. *When is debit note printed?* `showDebitNote=Y` and `PREM != 0` (from `DIFF_PREM_DUE`).
5. *CONTRACT_ID bug?* See B.7.
6. *Oracle UNION in Mongo?* `$unionWith`; or two `$lookup`s with `$ifNull`; de-dup with `LinkedHashSet`.
7. *How are landscape sections chosen?* Dispatch counts.
8. *Known gap?* The six unmapped fields.
9. *How are `mainReportName`/`subReportName` set?* `PlineReportNames.mainNameOf` from the bean path; service sets `subReportName`; `tag()` fills blanks.
10. *External call failure?* Swallowed and defaulted; the report still returns.

---

## MODULE C — RejectRisk

I read this module completely, including your uncommitted changes.

### C.1 Purpose
- **Business:** a **listing report of rejected risks** — customers/IDs whose risk was rejected (or "blacklisted"), with reject type, reason, remarks, who created it, and whether it was later revoked (and for how long).
- **Technical:** one service, one handler, one collection (plus lookups).

### C.2 Classes
- `RejectRiskService` — `@Service("AGIC/UW/Plines/RejectRisk/RejectRisk")`, implements `ReportStrategy`.
- `RejectRiskHandler` — queries `CMUW_PL_REFER_RISK`, joins `CPGE_VI_PARTNERS_ALL` (customer) and `CMGE_CODE` (three code lookups).
- `PlineRejectRiskDto`, `PlineRejectRiskDetails`; header via `PlineReportHeaderHandler`; `PlineReportBlocks.reportHeader(...)`; `PlineReportNames.tag(...)`.

### C.3 Inputs (service lines 47-55)
`pCreatedDateFrom`, `pCreatedDateTo`, `pCustomerName`, `pIdNo`, `pRejType`, `pReasonCode`, `pReportTitle`, `reportName`. All optional.

### C.4 Processing
1. Handler builds an aggregation:
   - `$match` on `CREATED_DATE`; optional `$match` on `REJ_TYPE` and `REASON_CODE`.
   - `$lookup CPGE_VI_PARTNERS_ALL` on `CP_PART_ID`/`CP_PART_VERSION` = `PART_ID`/`VERSION` (limit 1) → customer.
   - Optional `$match` on `CUST.NAME_EXT` (exact) and on `ID_VALUE1` or `ID_VALUE2` = `pIdNo`.
   - `$lookup CMGE_CODE` for descriptions: `PLREJ_RISK_TYPE` (on `REJ_TYPE`), `PLREJ_REASON` (on `REASON_CODE`), `PLREJ_REVOKE_DUR` (on `REVOKED_DURATION`).
   - `$project` to DTO names; `$setWindowFields` with `$documentNumber` sorted by `customerName` gives a 1-based `rowNumber`; then `$sort` by `customerName`.
2. Service looks up `dataClass` from the report-header template (`RPGE_RPT_TEMPLATE`, same as `Generic/ReportHeader.jrxml`) by `pReportTitle + reportName`.
3. For **each row** builds a separate `reportData` with `prntDetails > transDetList > transDet` (rownumber, customerName, idType1/idNo1, idType2/idNo2, `riskList.risk` with type code and name, reasoncode and description, createdBy/Date, revokeInd, revokedDuration + description, revokeBy/Date, updatedby/date) and `companyType.companyDesc.remarks` (from `REMARKS`).
4. If there are rows, a final `reportHeader("RejectRisk", dataClass)` block is added.
5. `reportParameters`: `pReportTitle`, `reportName`, `pCreatedDateFrom`, `pCreatedDateTo`.

### C.5 Your uncommitted changes (git diff) — know these cold
`RejectRiskHandler` (these are clearly bug fixes you are making):
1. **Upper date bound.** `CREATED_DATE` is a full timestamp. Old code used `$lte pCreatedDateTo` (midnight), so rows created later on the "to" day were lost. Now `dateTo` is shifted +1 day and compared with `$lt`. This matches the JRXML's `TRUNC(CREATED_DATE) BETWEEN` (inclusive whole day) without using `$dateTrunc`, so an index on `CREATED_DATE` is still usable.
2. **Null bounds.** Previously a null `dateFrom`/`dateTo` produced `{$gte: null, $lte: null}`, which matches only documents with null `CREATED_DATE` (BSON ordering) — zero real rows. Now only supplied bounds are added, and if neither is supplied no date filter is added.
3. **Removed the fake `ROW_NUMBER = 1`** `$addFields`; the real row number comes from `$setWindowFields`.

`RejectRiskService`: `toLocalDateTime` now uses `com.agcs.fo.opus.plines.utils.DateUtils` instead of the **motor** `DateUtils` (correct package for Plines; removes a cross-LOB dependency).

`application.properties`: adds a Mongo URI with credentials — **do not commit** (see section 0).

### C.6 Business and technical validations
- Business: all filters optional; customer-name match is exact; ID matches either of two ID columns; code descriptions come from `CMGE_CODE` categories.
- Technical: invalid dates are parsed through `DateUtils.toLocalDate`; parse failures return null (swallowed) in the service's `toLocalDateTime`; empty result gives no header block.

### C.7 Why designed this way
The Jasper report was a SQL with a join and a `ROWNUM` assigned after `ORDER BY NAME_EXT`. In Mongo, `$setWindowFields + $documentNumber` is the equivalent. One `reportData` per row follows the XSD rule "one print unit per entry". A disclaimer comment in the service says the consuming JRXML side has not been confirmed to treat `reportData[]` entries as independent print units — get report-team sign-off.

### C.8 Limitations
- `$lookup` per row set; customer filters are applied *after* the lookup (no index help), so name/ID filters scan all date-matched rows.
- Exact (case-sensitive) name match; no pagination.
- Date parsing failures silently become null.
- `report` is built per row with a nested single-element list structure, which bloats XML.
- Header block uses the same `dataClass` for all rows.

### C.9 Tests
`RejectRiskServiceTest` (4 tests) and `RejectRiskHandlerTest` (15 tests) exist (counted, not read in detail). Your date fixes (+1 day, null bounds) are uncommitted; confirm the tests still pass and add cases for them.

### C.10 Interview Q&A
1. *What is RejectRisk?* A list of rejected risks with reasons and revocation info for a date range.
2. *How did you map `ROWNUM`?* `$setWindowFields` with `$documentNumber`, sorted by `customerName`.
3. *Why `$lt dateTo+1day` instead of `$lte dateTo`?* Because `CREATED_DATE` has a time part; this includes the whole "to" day and keeps an index usable.
4. *What bug did null date bounds cause?* Mongo matches nulls only; result would be empty.
5. *Why one `reportData` per row?* XSD convention; note the disclaimer.
6. *How are descriptions resolved?* Three `CMGE_CODE` lookups by category: `PLREJ_RISK_TYPE`, `PLREJ_REASON`, `PLREJ_REVOKE_DUR`.
7. *What would you improve?* Index on `CREATED_DATE`, move customer filters earlier, case-insensitive name search, pagination.

---

# PHASE 4 — ARCHITECTURE ANALYSIS

## 4.1 High-level architecture

```
 Print/Report orchestrator (caller; not in repo)
        │  HTTPS  POST /gi/documentprintdata/api/opus/GenerateReport   (JSON)
        ▼
 CustomRequestFilter  (wraps body; turns array pBranch/pBranchName into "a,b,c")
        ▼
 CommonController.forwardRequest
   1. normalise reportName (strip leading "/" and "reports/")
   2. reportStrategies.get(reportName)  ──► ReportStrategy bean (new architecture)
   3. else legacy switch(reportName) ~50 cases (Motor/Fire/Bond etc.)
        ▼
 Service (e.g. Template12PolicyDocumentService)  ── orchestration + business gates
   ├─► Handlers (@Component) ──► MongoTemplate aggregation ──► MongoDB Atlas (DB OPUSAGICPRODR1)
   └─► Clients (PlineUtilsClient, PlinePgRpgeUtilsClient, ...) ──► OPUS core REST APIs (/ext/utils, /ext/policy, /ext/customer)
        ▼
 DTO → Pline* payload objects (XSD shape) → Transactions
        ▼
 Jackson XmlMapper → XML string → SuccessResponse{status, data, timestamp} (JSON, always HTTP 200)
        ▼
 Print engine renders PDF
```

Note: README says the response field is `xmlData`; the actual class `SuccessResponse` has `data` (SuccessResponse.java:16-21). Check against a live response before quoting the field name.

## 4.2 Component list (what exists / what does not)

| Item | Finding |
|---|---|
| Frontend | None. Pure backend. Swagger UI at `/gi/documentprintdata/swagger-ui/index.html` |
| Backend | Spring Boot 4.1.1, Java 17, servlet stack (MVC); WebFlux also on classpath (only `ClauseDescriptionClient`/WebClient uses it) |
| APIs | One real endpoint `POST /api/opus/GenerateReport`; `GET /getJSON` returns empty DTO; `DocumentController`, `ExtendedController` empty |
| Database | MongoDB Atlas, DB `OPUSAGICPRODR1`, read-only usage (no insert/update/delete found), collections mirror Oracle views |
| Third-party / external systems | OPUS core APIs at `devN-apiservice.opusm.srv.allianz` (PME, PG_RPGE_UTILS, PG_UWGE_UTILS, PG_UTIL_GENERAL, customer engine, e-invoice QR) |
| Batch / scheduled jobs | None in the app. CI has cron: CodeQL weekly (Sat 02:00), nightly secret scan (02:00) |
| Messaging | None |
| Deployment | Shared GitHub Actions in `opus-github-cicd-workflows`; Helm charts in separate repo `opus-helm-gi-documentprintdata-service` |

## 4.3 Low-level: layers

1. **Controller** — `motor/controller/CommonController.java` (502 lines). Request body class `PayLoadRequest` (`metadata`, `filterCondition`, `mergeStaticPdfFiles`).
2. **Service** — `@Service("<report path>")` implementing `ReportStrategy` (113 Plines services; 63 active `@Service`, 48 commented out).
3. **Handler** — `@Component`, raw `Document` aggregation stages.
4. **DTO** — one per sub-report (`documentdto`).
5. **Request payload** — `Pline*` classes mirroring Pline.xsd (`requestpayload`).
6. **Clients** — `PlineUtilsClient` (659 lines, one method per Oracle function exposed as REST), `PlinePgRpgeUtilsClient`, `PlinePolicyRefClient`, `PlineSSTRolloutDateClient`.
7. **Shared utils** — `PlineReportNames`, `PlineReportBlocks`, `DateUtils` (Plines copy), `PlineCommonMappingService`, `PlineReportHeaderHandler`.

## 4.4 Response (XML) structure

```
transactions
 └ transaction
    ├ templateDetails   (added by controller from metadata)
    ├ mailingDetails    (added by controller from metadata)
    ├ transactionDetails
    │   └ headerTransaction (repeated; dataclassH ...)
    │       └ reportTransaction
    │           └ reportData (repeated)  ← "1 subreport = 1 reportData"
    │               dataClass, mainReportName, subReportName, endorsementDetails
    │               prntDetails
    │                 branchInfo, clientInfo, compInfo, companyType, paramGroup
    │                 transDetList → transDet (≈652 leaf fields), companyType
    └ reportParameters  (echo of request params + report-level values)
```
`PlineTransDet` is a very wide "bag of leaves" (809 lines) because the XSD flattened many Jasper fields into one type.

## 4.5 Request / response / data flow (RejectRisk as the simple example)

1. Caller sends `filterCondition` with `reportName=AGIC/UW/Plines/RejectRisk/RejectRisk`, dates, optional filters.
2. Controller picks `RejectRiskService`.
3. Service → `RejectRiskHandler.fetchRejectRisks` → Mongo aggregation on `CMUW_PL_REFER_RISK`.
4. Service → `PlineReportHeaderHandler` → `RPGE_RPT_TEMPLATE` for `dataClass`.
5. Rows → `reportData` list → `Transactions` → `PlineReportNames.tag`.
6. Controller adds template/mailing details, serialises XML, wraps in JSON.

## 4.6 Security flow
- `SecurityConfig`: stateless sessions, CSRF ignored for `/**` (justified in javadoc since stateless), `/actuator/health` and `/info` open, other actuator paths denied, **`anyRequest().permitAll()`** — no authentication in this service. Auth must be enforced upstream (gateway/network); not visible in this repo.
- Outbound: custom truststore `mytruststore.p12` (PKCS12), configured by `app.ssl.*` in `application-env.properties`. No client credentials are attached to outbound calls.
- SSRF protections added in `40d3fc1f7` (2026-07-16): `PlineUtilsClient.sanitize` (only `[A-Za-z0-9_\-./#]`), `assertTrustedHost` (URL must start with configured base), `PlinePgRpgeUtilsClient` host check, `PlinePolicyRefClient` regex `[A-Za-z0-9_\-]+`.
- Secrets: `gitleaks.toml` allowlists `application.properties` and `application-env.properties` (OPUSM-50404, review-by 2026-10-23) — this hides findings in exactly the files where secrets are most likely. Mention as a known risk.
- Logging risks: `CustomRequestFilter` prints full request bodies via `System.out`; the legacy path logs full XML at INFO (TODO at CommonController:428).

## 4.7 Integration flow (external APIs)
All use base URL `API_BASE-URL` (dev2 host in committed file) plus a path property. Important ones for your modules:

| Purpose | Path (property) |
|---|---|
| E-invoice print indicator, endorsement | `/ext/utils/api/v1/pg-rpge-utils/invoice-print-endt-indicator` |
| E-invoice print indicator, policy | `/ext/utils/api/v1/pg-rpge-utils/invoice-print-indicator` |
| Service-tax indicator | `/ext/policy/api/v1/stax-inv` (`plines.fn.show.stax.inv.ind.path`); mode `WIP` when `checklistCopy=Y` else `OCP` |
| Company BRN | `/ext/utils/api/v1/pg-rpge-utils/comp-brn-number` |
| Signature | `/ext/customer/api/v1/fn-rppm-get-signature` (type `UW`) |
| SST rollout date | `/ext/policy/api/v1/pg-uwge-utils/sst-rollout-date` |
| QR indicator | `/ext/utils/api/v1/einvoice/qr-indicator` |
| Policy ref | `/ext/policy/api/v1/pg-uwge-utils/get-policy-ref` |

Timeouts (Plines config): connect 5 s, read/socket 20 s, connection-request 3 s, max 100 connections / 20 per route; "fast lookup" RestTemplate read 8 s. These came from a "Phase 1 incident fix" (bounded timeouts, pool sizing); Phase 2 added `fastLookupRestTemplate` for `PlineUtilsClient`. Good story: "we bounded external calls so one slow API could not hang report generation".

## 4.8 Error handling contract
- `GlobalExceptionHandler`: `NoDataFoundException` → 404 ProblemDetail (never thrown today — dead path); any other `Exception` → 500 ProblemDetail `INTERNAL_ERROR`. A second advice, `generic/advice/GenericExceptionHandler`, maps `IllegalArgumentException` (which includes `NumberFormatException`) to 400 and logs a warning; it is unscoped, so it applies app-wide.
- Unknown report name or null transaction → **HTTP 200**, `status: "SUCCESS"`, default empty transaction. The caller must inspect the XML. This is a design weakness worth naming in interviews.
- A `null` `reportName` in the legacy switch would NPE → 500 (inferred, not run).

---

# PHASE 5 — TECHNOLOGY STACK AND VERSIONS

Versions are from `pom.xml` and the Boot BOM as reported by the build analysis. "Explicit" = written in pom; "BOM" = managed by Spring Boot 4.1.1. I did not run `mvn dependency:tree`; verify before quoting exact transitive versions.

| Tool | Version | Source | Purpose / why |
|---|---|---|---|
| Java | 17 | explicit (`java.version`) | Records, pattern matching (`instanceof Number n` used in code). Boot 4 requires 17+. Alternative 21 — not required |
| Spring Boot | 4.1.1 | explicit parent | Upgraded from 3.5.14 on 2026-09-10 (`65f1a8711`) for CVE-2026-59270 |
| Spring Framework | 7.0.9 | BOM | |
| Spring Security | 7.1.1 | BOM | The 6.5.x fix was paywalled, so upgrade to Boot 4 was the route |
| Spring MVC + WebFlux | BOM | starters | Both present; only one WebClient client |
| Spring Data MongoDB | 5.1.1 (spring-data-bom 2026.0.1) | BOM | `MongoTemplate` aggregation |
| MongoDB driver | 5.8.1 | BOM | |
| MongoDB | Atlas, DB `OPUSAGICPRODR1` | config | Mirror of Oracle views. `$setWindowFields` (used in RejectRisk) needs MongoDB 5.0+ |
| Jackson 2 | 2.21.5 | explicit property `jackson-2-bom.version` | CVE patches. Code uses `com.fasterxml.jackson` annotations |
| Jackson 3 (`tools.jackson`) | 3.1.5 | BOM | On classpath; forced off for HTTP via `spring.http.converters.preferred-json-mapper=jackson2` |
| jackson-dataformat-xml | via Jackson 2 BOM | | XML generation by `XmlMapper` |
| Tomcat (embedded) | 11.0.25 | explicit; BOM 11.0.24 | 3 critical Wiz CVEs fixed in 11.0.25 |
| springdoc-openapi | 3.1.1 | explicit | Swagger UI. 2.8.15 broke on Boot 4 (`NoClassDefFoundError` on relocated `WebMvcProperties`) |
| Apache HttpClient 5 / HttpCore 5 | 5.6.4 / 5.4.3 | BOM / explicit | Pooled client with truststore |
| Lombok | 1.18.46 | BOM | `@Slf4j`, `@RequiredArgsConstructor` |
| commons-lang3 | 3.18.0 | explicit (BOM is 3.20.0) | CVE-2025-48924 |
| logback | 1.5.34 | explicit (BOM is 1.5.38) | Wiz CVEs |
| log4j2 | 2.25.5 | explicit | Wiz CVE-2026-49844 |
| Micrometer | 1.17.1 | BOM (pin removed) | Old 1.15.12 pin broke startup of every `@SpringBootTest` |
| org.json | 20231013 | explicit | Old; candidate for review |
| JUnit | declared junit-jupiter 5.9.0 (explicit), BOM 6.0.3 | | Effective engine probably 6.0.3 (inferred from `~/.m2`) |
| Mockito | 5.23.0 | BOM | `mock-maker-inline` in test resources |
| JaCoCo | 0.8.12 | explicit | Coverage into Sonar |
| OpenRewrite plugin | 6.46.1 | explicit | Scaffolding, no active recipes (6.48.0 not resolvable on Nexus mirror) |
| Maven | 3.9.11 (wrapper 3.3.4) | `.mvn/wrapper` | |
| GitHub Actions | | `.github/workflows` | CI/CD, Sonar, CodeQL, secret scan, Jira check, changelog bot, Claude PR review |
| SonarQube | project `opus-gi-documentprintdata-service` | workflow | Quality gate wait on |
| Wiz | | CICD secrets | Vulnerability scan; drives CVE pins |
| Gitleaks | 8.30.1 mentioned | `gitleaks.toml` | Secret scanning |

Not used (checked): Hibernate/JPA, Gradle, Oracle/SQL Server/PostgreSQL drivers, Docker/Kubernetes files in this repo (Helm is in another repo), Jenkins, Kafka.

## 5.1 Why these versions / constraints / risks
- **Boot 4.1.1:** chosen because the CVE fix for Spring Security was only freely available in 7.1.1. Constraint: Boot 4 brings Framework 7, Jackson 3, and moved properties.
- **Upgrade fixes recorded** (`65f1a8711`, `c6b6803f3`): `UriComponentsBuilder.fromHttpUrl` → `fromUriString`; `getRawStatusCode()` → `getStatusCode().value()`; `@MockBean` → `@MockitoBean`; explicit `MediaType` imports (JUnit 6 collision); explicit `WebClient.Builder` bean (no longer auto-configured); `spring.mongodb.*` instead of `spring.data.mongodb.*` (old keys are silently ignored and the app falls back to `localhost:27017` — a classic trap); `jackson-2-bom.version` instead of `jackson-bom.version`.
- **Upgrade risk:** moving off Jackson 2 later will need annotation migration. Pins below the BOM (`logback`, `commons-lang3`) may be accidental downgrades — verify with `dependency:tree`.
- **Downgrade risk:** going back to Boot 3.5 re-opens CVE-2026-59270 and requires reverting the property renames.
- **Version-conflict stories:** micrometer pin vs `JvmMemoryMeterConventions`; springdoc vs Boot 4; `jackson-bom.version` meaning change.
- **Stale docs:** `ARCHITECTURE.md` says Boot 3.4.11 and `README.md` says 3.5.14 / Security 6.5.9. `pom.xml` is the truth (4.1.1).

---

# PHASE 6 — CODE ANALYSIS (what is covered so far, and the gap)

Your message ended in the middle of the Phase 6 heading, so I did not get its instructions. Below is what I could ground in the code. Tell me what you want Phase 6 to contain and I will extend this file.

## 6.1 Where the real work is

| Area | Size | Notes |
|---|---|---|
| `plines/documentHandler` | ~422 main files | One per sub-report |
| `plines/documentdto` | ~377 | |
| `plines/requestpayload` | ~302 | XSD mirror |
| `plines/service` | 140 | 113 `ReportStrategy` |
| Biggest classes | `PlinePolicyDocumentBaseService` 4,083; `Template12PolicyDocumentBaseService` 3,732; `Template12PolicyDocumentService` 2,235; `AspEndtDocumentService` 1,701 | God-class risk |

## 6.2 Recurring code patterns (learn these, they appear everywhere)
1. **Oracle → Mongo translation.** Scalar subselects → `$lookup` + `$limit 1` + `$arrayElemAt`. `UNION ALL` of WIP and live → two lookups with `$ifNull`, or `$unionWith`. `SELECT DISTINCT` → `$group`. `ROWNUM` → `$setWindowFields`/`$documentNumber`. `FROM DUAL` (always one row) → fallback DTO. Oracle functions → REST calls.
2. **Oracle error emulation.** `requireNoConflict` / `distinctValueOrConflict` throw when data conflicts, mimicking ORA-01427.
3. **Type strictness in Mongo.** `CONTRACT_ID` string vs number mismatch silently returns no match. Always convert (`Long.parseLong`/`safeNum`).
4. **Fail-soft vs fail-hard.** Sub-bands catch and log; primary fetch and dispatch propagate.
5. **Reflection-based merging** of transDet fields (Template12).
6. **Naming:** `PlineReportNames.mainNameOf` builds `mainReportName` from the bean path.

## 6.3 Bugs / smells you can speak about honestly
- Template12: `Long.parseLong("")` NPE-class failure in landscape dispatch; hardcoded `certNo=""`; unused locals; duplicated `det.setJoinDate`; three divergent product fetches; commented-out hardcoded contract ids.
- ASP: unmapped six fields; stubs for LLB/STI/ScheduleOfBenefits; landscape queries run every time; test-tree copy of `AspServiceTaxInvoiceHandler`.
- RejectRisk: date bound bug and null-bound bug (your uncommitted fixes); unindexed customer filters.
- Platform: always-200 responses; no auth; dead `XmlConverter`; stray `@Service` on the controller; `System.out` logging; committed `.scratch/` folder and a file literally named `null` in the repo root; `pom.xml` artifactId has leading and trailing spaces.
- Testing: ~1,929 test files (1,384 under plines), but no true XSD conformance test of real output; the test XSD (v0.1.6) lacks `mainReportName/subReportName` that the code emits (v0.1.10 per comments).

## 6.4 Delivery / process facts
- 134 commits touched `plines` since 2026-05-12 (105 non-merge), earliest non-merge 2026-07-16, latest 2026-10-08.
- Themes: XSD alignment (July–Aug), "dynamic merging" of Document/Document2/Landscape (Aug), Sonar/CodeQL fixes, SSRF fix (07-16), Boot 4 upgrade (09-10 to 09-24), test-coverage push OPUSM-44159 (09-25 to 10-01), report/sub-report names (10-06 to 10-08).
- Top tickets: OPUSM-44159 (7 commits), OPUSM-42246 (5), OPUSM-42048 (4).
- CI: Sonar on PRs to R1/R2, CodeQL, secret scanning, Jira commit-link check (project `OPUSM`), Claude PR review, changelog bot. `cicd-workflow.yml` is manual (`workflow_dispatch`), runs the shared `java-microservice` action with Java 17.

---

# PRODUCTION SUPPORT CHEAT SHEET (from code evidence)

| Symptom | Likely cause | Where to look |
|---|---|---|
| Template12 fails: 400 for empty or non-numeric `pContractId` (`NumberFormatException` is an `IllegalArgumentException`, handled by `GenericExceptionHandler`; inferred, not run) or 500 for WIP-vs-live conflict | Empty/non-numeric `pContractId` → `NumberFormatException` in landscape dispatch; or WIP/live conflict → `IllegalStateException` | `Template12LandscapeDispatchHandler`, `fetchTemplate12PolicyDocumentMain` |
| ASP tax invoice missing | Gate not met: `showTaxInvoice`, `SHOW_STAX_INV_IND`, `checklistCopy=Y`, `SHOW_EINV_IND`; or external API failed (defaults to N) | `AspEndtDocumentService` ~line 335; `PlineUtilsClient` |
| Flags come back null | ID type mismatch in Mongo match | `AspEndtDocument2MainHandler.buildCommonPipeline` |
| RejectRisk missing rows on the end date | Old `$lte` bound | `RejectRiskHandler` date filter |
| RejectRisk returns nothing with no dates | Null bounds | Same |
| Empty XML with "SUCCESS" | Unknown report name or no strategy bean | `CommonController` STEP 1/2; check `@Service` name |
| Old report name returns nothing | Merged beans (Template12 PD2/Landscape; ASP EndtDocument2/Landscape) | Use the merged name |
| App connects to localhost:27017 | Old `spring.data.mongodb.*` keys under Boot 4 | `application.properties` |
| Startup errors with JSON conversion | Jackson 3 default | `spring.http.converters.preferred-json-mapper=jackson2` |
| Slow/hanging report | External API latency | `app.http.*` timeouts, `fastLookupRestTemplate` |

---

# NEXT STEPS I RECOMMEND

1. Tell me what Phase 6 should include (your message was cut off).
2. Get from your team: Jira text for OPUSM-44159, 41763, 42246, the functional design / mapping sheet `Pline_MASTER_MAPPING_SHEET_v0.1.xlsx`, and the real meaning of ASP. I can then fill in business detail and tie your own tickets to code.
3. Run `./mvnw dependency:tree` so Phase 5 versions can be confirmed.
4. Do not commit the `application.properties` change.
5. I can deepen any module line by line (for example, the full Template12 band table or the ASP service-tax handler).

