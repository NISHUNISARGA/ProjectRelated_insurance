# 07a - Traces, Defects, Production Support, Performance
> **AUTHORSHIP UPDATE (confirmed by the developer): git author `GS <extern.gs_nisarga@allianz.com>` is the developer. Wherever this file says commits cannot be attributed to the developer, read it as: the 11 GS commits listed in `KT_Parts/14_My_Commits_And_Ownership.md` ARE theirs; commits by other authors still must not be claimed. No GS commit touches `src/test`, so test authorship is still `[CONFIRM]`.**

Service: `opus-documentprintdata-service` (Spring Boot 4.1.1, Java 17, MongoDB, XML-in-JSON report data).
Your modules: Template12 (PolicyDocument, PolicyDocument2, Landscape), ASP (EndtDocument, EndtDocument2, AddMemberLandscape, DeleteMemberLandscape), RejectRisk.
Your Jira scope: (i) analysis of the three reports and all sub-report levels, (ii) code, (iii) unit tests, (iv) XSD changes.

This file covers Phase 11 (traces), Phase 12 (defects), Phase 13 (production support) and Phase 14 (performance).
The interview answers and STAR stories are in `07b_Ownership_Answers_And_STAR_Stories.md`.

---------------------------------------------------------------------------------------------------

## 0. Rules for reading this file

1. Every fact comes from the earlier KT files (`OPUS_DocumentPrintData_KT_Guide.md`, `01` to `06`) or from a check I did in the repo. Where I am guessing, I write **(inferred)**.
2. Personal facts (what YOU did, when, who asked) are not in the repo. Git has no commit by your user name (`NISHUNISARGA`). I write `[CONFIRM: ...]` where you must fill in the truth. Do not say "I wrote commit X" unless you know it is yours.
3. There is **no Jira text, no Confluence, no incident ticket, no log file and no Helm chart** in what I could read. So "requirement" below means: the original Jasper `.jrxml` (the real specification), the mapping workbooks, and the Jira scope line you gave me.
4. Known truths I keep repeating on purpose:
   - Only ONE live bean each for Template12 and ASP. The other classes are merged in and their `@Service` is commented out.
   - XSD v0.1.6 is the only XSD on this machine. Real Java output does NOT validate against it (see 3.7 below).
   - "ASP" is not expanded anywhere. Java ties it to product code `070412`.
   - The RejectRisk uncommitted fix equals commit `cc6a1b84d` on another branch, and it likely breaks 7 of 15 handler tests.
   - The API has no authentication. An unknown report name returns HTTP 200 `SUCCESS` with a default transaction.

---------------------------------------------------------------------------------------------------

# PHASE 11 - REQUIREMENT TO DEPLOYMENT TRACES (10 features)

How to read a trace. Each feature is one chain:
Requirement -> Design -> Configuration -> DB -> API -> XML -> XSD -> Code -> Testing -> Deployment, followed by "Why this decision?".

Deployment is the same for all ten features, so I write it once here and refer to it.

**Common deployment facts (evidence: guide section 4, file 06 part 4):**
- The pipeline is GitHub Actions. `cicd-workflow.yml` is manual (`workflow_dispatch`, optional `imageTag`). It calls a shared action `opus-github-cicd-workflows/actions/java-microservice`. That shared action is not in this repo, so I cannot say what it does inside.
- PR gate: `sonar-code-review.yml` runs `mvn -B verify` plus Sonar with `-Dsonar.qualitygate.wait=true` on PRs to `retrofit-develop-R1` / `R2`.
- Other gates: `jira-commit-check.yml` (commit must carry an `OPUSM-` key), `secret-scan.yml`, `codeql.yml`, `claude-pr-review.yml`.
- Helm charts and Kubernetes manifests are in another repo (`opus-helm-gi-documentprintdata-service`). Not visible to me. Rollback is probably "redeploy the previous image tag" **(inferred)**.
- The pom version is a static `0.0.1-SNAPSHOT`, so the image tag identifies a release.
- Who actually ran the deployment and in which environment: `[CONFIRM: who triggered cicd-workflow, which environments (dev/SIT/UAT), which image tags]`.

---------------------------------------------------------------------------------------------------

## Trace 1 - Template12: three Jasper reports merged into one bean

| Step | What the evidence shows |
|---|---|
| **Requirement** | Jasper has three mains in `PolicySchedule/Template12`: `PolicyDocument` (648 lines), `PolicyDocument2` (1240 lines), `Landscape` (529 lines). Together they print one policy pack. Jira scope item (i) and (ii) cover all three (file 02, section 1). |
| **Design** | One Spring bean `@Service("AGIC/UW/Plines/PolicySchedule/Template12/PolicyDocument")` (`Template12PolicyDocumentService.java:103`). `execute()` runs PolicyDocument bands, then private `executeMergedPolicyDocument2` (lines ~578-779), then private `executeMergedLandscape` (lines ~801-1497). Response: one shared header `reportData` plus one `reportData` per Landscape row. |
| **Configuration** | No report-specific property. Routing depends only on the bean name. The caller sends `filterCondition.reportName` equal to the bean name. Old names `.../PolicyDocument2` and `.../Landscape` have no bean any more. |
| **DB** | Three different primary fetches (see Trace 2): `fetchTemplate12PolicyDocumentMain`, `PlineT12PolicyDocumentHandler`, `Template12LandscapeDispatchHandler`. Collections include `WIP_UWGE_POLICY_CONTRACTS`, `SB_UWGE_POLICY_CONTRACTS`, `UWPL_POLICY_BASES`, `UWPL_VI_POLICY_SCHEDULE_*`, `CMGE_CODEREL_DET` and many more (file 02 section 3). |
| **API** | `POST /gi/documentprintdata/api/opus/GenerateReport`. `CommonController` looks up `reportStrategies.get(reportName)` (`CommonController.java:171, 203`). |
| **XML** | `Transactions > Transaction > PlineTransactionDetails > PlineHeaderTransaction(dataclassH="PlinePolicyDocument") > ReportTransaction > reportData[]`. `mainReportName = "Template12PolicyDocument"`. |
| **XSD** | `Pline.xsd` v0.1.6 (`reportData` has no `mainReportName`/`subReportName`). See Trace 7 and 8. |
| **Code** | Commit `c0c6d449a` (2026-08-03, "Added Template12 dynamic merging") did the merge: it commented out `@Service` on `Template12PolicyDocument2Service` and `Template12LandscapeService`. `Template12PolicyDocumentBaseService` (3,732 lines) holds about 95 `add*` helpers. Git author of this commit is Amarnath Yadav, not you (git shows names exactly as recorded). |
| **Testing** | `Template12PolicyDocumentServiceTest` 34 tests, `Template12PolicyDocumentBaseServiceTest` 135. `Template12PolicyDocument2ServiceTest` (33) and `Template12LandscapeServiceTest` (28) test the DISABLED classes directly (file 05). Counts are `@Test` text counts, not run results. |
| **Deployment** | Common facts above. |

**Why this decision? (Evidence and reasoning)**
1. The class javadoc says re-enabling the old `@Service` would render every band twice. So merge = no duplicate output and one response for the print engine (guide A.2, A.12 Q1).
2. The old classes were kept "for reference" instead of being deleted. Cost: dead code with its own tests (file 02 D17). Benefit: easy diff against the merged logic **(inferred)**.
3. Trade-off the code itself admits: one 2,235-line service, and the renderer has not confirmed it treats each Landscape `reportData` as an independent print unit (comment at `Template12PolicyDocumentService` lines 159-165).
4. What I would say if asked "was it a good idea?": it gives one response and fewer calls, but it hides the report boundary. The new `mainReportName`/`subReportName` fields (Trace 8) were added later to give the print engine that boundary back.

---------------------------------------------------------------------------------------------------

## Trace 2 - Template12: product and scheme dispatch

| Step | Evidence |
|---|---|
| **Requirement** | In Jasper `PolicyDocument`, band `P2` to `P4` choose a CI print variant by `PRODUCT_CODE`, then by `SCHEME_NAME` for product `070504` (jrxml lines 221-336). Decision matrix in file 02 section 5.1. |
| **Design** | `execute()` computes booleans (`is070504`, `is070412`, `is070406`, `is070414`, BMW, POSMESRA, FWPA, POS-cover-with-dependants) at `Template12PolicyDocumentService` lines 179-187 and runs one `if` block per variant. Elite is the fallback "none of the other schemes". |
| **Configuration** | None. Product and scheme come from DATA, not from the request (`API sample`, file 06 section 2.7 note). Cut-off dates come from `SAPM_SYS_CONSTANTS` (`ASP_CUTOFF_DATE`, `PIDM_CUTOFF_POL`). |
| **DB** | Primary fetch returns product, scheme, `ASP_CUTOFF_IND`, issue date, dependants. Landscape dispatch also reads `CMGE_CODEREL_DET` for `GPA_SCHEME / CI_LANDSCAPE_PRT`. |
| **API** | Same endpoint. Flags that affect output from the request: `showComplaint` (default `Y`), `showScheduleStd` (default `Y`), `ShowOutPrintCovLetter` (capital S key), `pCertNoFrom/To` for BMW. |
| **XML** | Each variant writes its own rows into `transDetList/transDet`. BMW writes one `transDet` per certificate; PA4A one per row. |
| **XSD** | No Template12-specific element. It uses the generic PolicyDocument family in `Pline.xsd`. |
| **Code** | Variant handlers: `PlineCIPrintPrimaryHandler`, `...AtcHandler`, `...AstroPosHandler`, `...BmwPrimaryHandler`, `...AzpHandler`, `...PosmPrimaryHandler`, `...FwpaHandler`, `...ElitePrimaryHandler`, `...Pa4aPrimaryHandler`. Helper methods `addTemplate12CIPrint*` in `Template12PolicyDocumentBaseService`. |
| **Testing** | Service tests per branch, for example `execute_astroScheme070504_mapsAstroFields`, `execute_bmwScheme070504_mapsFirstCertRowAndRunsNestedSubreportsPerRow`, `execute_pa4aProduct070414_mapsPolicyNoAndRunsPerRowNestedSubreports`. Handler tests use a mocked `MongoTemplate`. |
| **Deployment** | Common facts. |

**Why this decision?**
1. The Jasper main query is a one-row `dual` select. In Mongo there is no `dual`. So the Java reads the same facts first (product, scheme) and then repeats the same `printWhenExpression` logic in `if` statements. That is the closest port, so behaviour can be compared line by line with the jrxml.
2. Three primary fetches were deliberately NOT unified (class javadoc "CRITICAL FACT #1", lines 77-87). Reason given in code: unifying would change PD2 behaviour without product sign-off. Cost: product and scheme can disagree between the three fetches (file 02 D6, D19).
3. Template12 has its own base class (`Template12PolicyDocumentBaseService`) and does not extend `PlinePolicyDocumentBaseService` (4,083 lines, used by about 24 templates). Reason (javadoc Base:42-60): changes must not break other templates, and Template12 uses the new XSD-typed envelope. Cost: duplicated code, for example `fetchCIPrintFwpaResilient`.
4. Known deviations from Jasper that I should be honest about (file 02 section 4): page-number conditions (`$V{PAGE_NUMBER}`) cannot be computed in a data service; Java prints the `PDS_Elite`/`PDS_Income` marker rows that Jasper never prints (`false &&` bands); the Domestic/Overseas ATC row is not product-gated; PD2 has extra insured/nomination rows that no PD2 band calls.

---------------------------------------------------------------------------------------------------

## Trace 3 - ASP: four Jasper reports merged into `EndtDocument`

| Step | Evidence |
|---|---|
| **Requirement** | Four Jasper mains in `ASPEndtSchedule`: `EndtDocument`, `EndtDocument2`, `AddMemberLandscape`, `DeletionMemberLandscape` (file 03 section 1). Hierarchy CSVs exist for the first two only. |
| **Design** | One bean `@Service("AGIC/UW/Plines/ASPEndtSchedule/EndtDocument")` (`AspEndtDocumentService.java:165`, 1,701 lines, `execute` about 1,300 lines). It calls two main handlers (V1 and V2), evaluates the gates, then runs both landscape dispatchers with no gate. `AspEndtDocument2Service` has `//@Service`; `AspEndtDocumentLandscapeService` has both `@Bean` methods commented out. |
| **Configuration** | None specific. Request flags: `showCoverNote`, `showEndtSchedule`, `showComplaint` (only echoed), `showDebitNote`, `showTaxInvoice`, `checklistCopy`. Defaults `"N"` for the show flags (`AspEndtDocumentService` lines 215-230). |
| **DB** | Main: `UWMT_VI_ENDT_TAX_INV_ALT_UPV`, `UWGE_VI_ENDT_REPORT_PARAM`, `WIP_/SB_UWGE_POLICY_CONTRACTS`, `UWGE_/WIP_UWGE_POLICY_VERSIONS`. Many more per sub-report (file 03 section 3.3). |
| **API** | Same endpoint. `pContractId` and `versionNo` must be numeric text. Outbound: `fnEinvoicePrintEndtInd`, policy ref, SST rollout date, QR indicator, company BRN, signature. |
| **XML** | "1 subreport = 1 reportData". `reportData[0]` is a master row (`mainReportName = ASPEndtScheduleEndtDocument`, `subReportName = PrintSeriesFooter`). Then EndtCover, ComplaintRedress, EndtSchedule, DebitNote rows, EndtTaxDoc, ServiceTaxInvoice(+EndtMT, QR), Transitional invoices, then landscape rows (file 03 section 3.2). |
| **XSD** | Uses `transDet`, `endorsementDetails`, `prntDetails/*`. Six delete-member fields have no `transDet` leaf, so they are not output (`subcoverPrem`, `endtDoubleIndemnity`, `endtImaFee`, `endtStaxAmt`, `endtLlbPrem`, `endtStiPrem`). File 03 notes that some of them DO exist under `endorsementDetails` (XSD lines 607-617). |
| **Code** | Handlers: `AspEndtDocumentMainHandler`, `AspEndtDocument2MainHandler`, `AspEndtCoverHandler`, `AspEndtScheduleHandler` (840 lines), `AspDebitNoteHandler`, `AspServiceTaxInvoiceHandler`, `AspAddMemberLandscape*Handler`, `AspDeleteMemberLandscapeHandler` and others. Commit `7a925920c` (2026-08-28, git author GS) "Wire ASP EndtDocument sub-reports and fold Landscape reports into one response" is not on HEAD; `4cd1cad2e` "Working code for pline" is on HEAD. |
| **Testing** | `AspEndtDocumentServiceTest` 28 tests (example: `allGatesOpen_oneReportDataPerSubreportInSubreportOrder`). ASP handler tests about 126. 277 tests for the EndtDocument family by file 03's own arithmetic. |
| **Deployment** | Common facts. |

**Why this decision?**
1. Same reason as Template12: one response per request, and no duplicate bean keys (class javadoc lines 87-96).
2. The merged order in `reportData[]` is a CONVENTION, not a print order. For example ComplaintRedress is placed before EndtSchedule, which is not the Jasper print order (file 03 section 3.2). The service test locks this order.
3. The javadoc sentence "Dispatched via the incoming reportName filter value" is stale: `execute` never reads `reportName` (file 03 section 3.1).
4. Gaps I must name honestly: `EndtScheduleOfBenefits_ASP`, `ASPLLB`, `ASPSTI` are stubs (no `reportData` emitted); `INST_IND` and `SHOW_GST_LABEL` cut-off rules are not evaluated in the schedule handler; ORA-01427 emulation is only in `AspServiceTaxInvoiceHandler`; `pSchemeName` is never read.

---------------------------------------------------------------------------------------------------

## Trace 4 - ASP: service-tax invoice gate

| Step | Evidence |
|---|---|
| **Requirement** | `EndtDocument2.jrxml` band 6: `showTaxInvoice=Y && SHOW_STAX_INV_IND=Y && !checklistCopy=Y && SHOW_EINV_IND=Y` (file 03 section 1.3). The inner element condition is weaker, but a false band condition hides the whole band. |
| **Design** | Boolean `showSt` at `AspEndtDocumentService` lines 335-338 (four conditions). If it is true: fetch `AspServiceTaxInvoiceHandler`, then company contact, AntiRasuah banner, EndtMT / EndtMTSub rows when `STAX_COUNT > 1 && ENDT_CODE != 999`, QR section when `EINV_CONSENT_IND=Y && EINV_STATUS=VALID && QR_IND=Y`, master-customer block when `PREM_BY_MST_IND=Y`. |
| **Configuration** | Paths: `opus.fn-einvoice-print-endt-ind-path` (`/ext/utils/api/v1/pg-rpge-utils/invoice-print-endt-indicator`), `plines.sst.rollout.date.path`, `plines.policy.ref.pme.path`, QR indicator path. Base URLs: `API_BASE-URL` and `plines.api.base-url` (two separate keys). Timeouts: fast-lookup 8 s for `PlineUtilsClient`; the Pline*Client classes use the primary RestTemplate (5 s connect / 20 s read). |
| **DB** | `UWPL_VI_ENDT_TAX_INV`, `CMGE_CODE`, `CMUW_ENDT`, `SAPM_SYS_CONSTANTS` (`EINV_CUT_OFF`), `UWGE_POLICY_VERSIONS`, `WIP_UWGE_POLICY_VERSIONS`, `UWGE_POLICY_BASES_STAX`, `WIP_UWGE_POLICY_BASES_STAX`, `UWPL_VI_TAX_INVOICE`. |
| **API** | Request needs `showTaxInvoice=Y`. `checklistCopy=Y` closes the gate. `SHOW_EINV_IND` comes from a REST call in the V2 main handler, not from Mongo. |
| **XML** | One `ServiceTaxInvoice` `reportData` plus `endorsementDetails` fields (`staxInvNo`, `staxInvDate`, `staxInvType`, `sstDate`, `maskedTin`, `sstRegisterNo` ...). TIN is masked except the last 4 characters. |
| **XSD** | `transDet/staxInvDate`, `endorsementDetails/staxInvNo`, `staxInvType`, `sstDate`, `maskedTin` exist in v0.1.6 (file 03 section 3.3). |
| **Code** | `AspServiceTaxInvoiceHandler` (476 lines): `requireSingleValue` and `requireNoConflict` throw `IllegalStateException` to copy ORA-01427; `computeStaxCount` sums WIP and live counts; `safeCall` returns null on REST failure; QR defaults to `N`. |
| **Testing** | `AspServiceTaxInvoiceHandlerTest` 27 tests (for example `fetchServiceTaxInvoice_ambiguousEinvCutoff_throwsIllegalStateException`). Service tests: `showServiceTax_checklistCopyY_closesGate`, `showServiceTax_staxCountNotAboveOne_skipsEndtMTAndSub`. **Caution:** a near-copy of the handler sits in `src/test` and may shadow the real class (see Defect 8). |
| **Deployment** | Common facts. |

**Why this decision?**
1. Keep the four-part Jasper condition exactly. The guide warns not to add a product-code condition that the jrxml does not have.
2. External API failures are swallowed (`safeCall` returns null, QR defaults to `N`). The report still returns. The price: if the e-invoice API is down, the indicator becomes empty and the service-tax page silently does not print. That is a production support point (Phase 13).
3. ORA-01427 emulation here is on purpose: Oracle would fail the query when the cut-off or version is ambiguous, so Java fails too instead of choosing one value silently.

---------------------------------------------------------------------------------------------------

## Trace 5 - ASP: landscape dispatch (add member, sponsor, nominee, delete member)

| Step | Evidence |
|---|---|
| **Requirement** | `AddMemberLandscape` main query computes `ADDITION_PRINT`, `NOMINEE_PRINT`, `SPONSOR_PRINT`; `DeletionMemberLandscape` computes `DELETION_PRINT`. Addition needs `ENDT_CODE` 72 or 168 and a risk with `ACTION_CODE=A`. Deletion needs `ACTION_CODE=C` and `TERMINATE_DATE` not null (file 03 sections 1.4, 1.5). |
| **Design** | `AspAddMemberLandscapeDispatchHandler` and `AspDeletionMemberLandscapeDispatchHandler` run on EVERY call (no gate), and two dispatch rows are always added (`AddMemberLandscapeDispatch`, `DeleteMemberLandscapeDispatch`) even when counts are 0. The count tells the print engine whether to print. |
| **Configuration** | Aggregations use `allowDiskUse` and `maxTime` 600 s in many ASP handlers. |
| **DB** | `UWGE_POLICY_VERSIONS`, `WIP_UWGE_POLICY_VERSIONS`, `UWGE_RISK`, `WIP_UWGE_RISK`, `UWPL_VI_ENDT_SCHEDULE_NOM`, `_SPONSOR`, `_TPL4`, `UWPL_VI_ENDT_ADD_DEL_RISK`. |
| **API** | Same endpoint. The old `reportName` values `AddMemberLandscape` / `DeletionMemberLandscape` no longer resolve (inferred from the `@Service`/`@Bean` annotations, file 03 U2). |
| **XML** | Dispatch rows plus `AddMemberLandscape_ASPInsured`, `_ASPPremium`, `EndtParticularOfSponsorLandscape_ASP`, `EndtParticularOfNominationLandscape_ASP`, `DeleteMemberLandscape_ASP` rows. `subReportName` distinguishes them. |
| **XSD** | `additionPrint` (line 685), `deletionPrint` (842), `sponsorPrint` (1226), `nomineeList/nominee/nomineePrint` (337). |
| **Code** | `safeNum` (`Long.parseLong`) in every handler because the Jasper landscape sub-reports declare `pContractId` as String but Mongo stores a number (file 03 section 2.3, 4.4). Sort uses `$toDouble(COV_SEQ_REF)` to copy Oracle `cast(... as decimal(15,2))`. |
| **Testing** | `AspAddMemberLandscapeDispatchHandlerTest` 11 tests (WIP/live branches, null and non-numeric contract id). `minimalDefaults_skipsAllOptionalSubReportsButRunsLandscapeDispatch` documents the "always runs" rule. |
| **Deployment** | Common facts. |

**Why this decision?**
1. In Jasper these were separate top-level reports, each with its own dispatch SQL. After the merge, the print engine gets all counts in one response and decides what to print.
2. Honest cost: 5 to 6 extra Mongo queries on every ASP request, even for endorsements that are not 72/168/98 (file 03 section 4.1 #1). An easy improvement: gate the dispatch by `ENDT_CODE` from the main DTO. I have not tested this idea.
3. Known Jasper quirk preserved or not: Jasper counts deleted risks of ANY endorsement code but shows only 98/168 rows; Java follows the 98/168 match in the row handler (file 03 section 1.5).
4. Known gaps: ORA-01427 is not emulated in the dispatch handlers; `RELATIONSHIP_DESC` in Jasper uses `SUBSTR(CODE_DESC,'4')`, Java returns the full description; `$round` is half-to-even in Mongo but Oracle `ROUND` is half-away-from-zero (inferred from documented DB behaviour, not tested).

---------------------------------------------------------------------------------------------------

## Trace 6 - RejectRisk listing (date range and row number)

| Step | Evidence |
|---|---|
| **Requirement** | `RejectRisk_jrxml.data` (780 lines). Oracle SQL lines 77-114: `ROWNUM` after `ORDER BY NAME_EXT`; `TRUNC(CREATED_DATE) BETWEEN from AND to`; optional filters on name, ID, reject type, reason code; three `CMGE_CODE` scalar sub-selects. One sub-report: `ReportHeader` (reads `DATA_CLASS`). Input controls exist only for 4 of the 8 parameters (file 01 section a.1). |
| **Design** | `RejectRiskService` (`@Service("AGIC/UW/Plines/RejectRisk/RejectRisk")`) calls `RejectRiskHandler.fetchRejectRisks`, then `PlineReportHeaderHandler` for `dataClass`. One `reportData` per DB row, plus one trailing `ReportHeader` block if there are rows. |
| **Configuration** | None. All 8 parameters are optional. No `AggregationOptions` set (no `allowDiskUse`, no `maxTime`). |
| **DB** | `CMUW_PL_REFER_RISK` (driver), `CPGE_VI_PARTNERS_ALL` (`$lookup`, `$limit 1`), `CMGE_CODE` x3 (`PLREJ_RISK_TYPE`, `PLREJ_REASON`, `PLREJ_REVOKE_DUR`), `RPGE_RPT_TEMPLATE` (header). |
| **API** | Keys: `pCreatedDateFrom`, `pCreatedDateTo`, `pCustomerName`, `pIdNo`, `pRejType`, `pReasonCode`, `pReportTitle`, `reportName`. No external REST call at all. |
| **XML** | Per row: `rownumber`, `customerName`, `idType1/idNo1`, `idType2/idNo2`, `riskList/risk` (type code and name), `reasoncode`, `reasonCodeDescription`, `createdBy/Date`, `revoke*`, `updatedby/date`; `REMARKS` goes to `transDetList/companyType/companyDesc/remarks`, a sibling of `transDet`. |
| **XSD** | All 17 `transDet` leaves exist in v0.1.6 with the listed types. `pCreatedDateFrom/To` and `pReportTitle` do NOT exist in v0.1.6 (file 04 section 4.1). |
| **Code** | Pipeline order (working tree): date `$match` -> optional type/reason `$match` -> `$lookup` partner -> optional name/ID `$match` -> 3 `$lookup` -> `$project` -> `$setWindowFields` (`$documentNumber` sorted by `customerName`) -> `$sort`. Working-tree changes: upper bound is next day with `$lt`, only supplied bounds are added, fake `ROW_NUMBER` stage removed, plines `DateUtils` import in the service. |
| **Testing** | `RejectRiskHandlerTest` 15 tests, `RejectRiskServiceTest` 4 tests. With the uncommitted fix, 7 of 15 handler tests are expected to fail (static reading, not run). |
| **Deployment** | Common facts. Do NOT commit the `application.properties` change (it holds a plaintext Mongo URI; I write it as `<redacted>`). |

**Why this decision?**
1. `x < to + 1 day` is the same as `TRUNC(x) <= to`, and it keeps `CREATED_DATE` as a bare field, so an index can be used. A `$dateTrunc` would block index use (code comment, file 01 d.1).
2. `$setWindowFields` + `$documentNumber` is the Mongo way to number rows after a sort, like Oracle `ROWNUM` over an ordered subquery. It needs MongoDB 5.0+ (inferred; other handlers already use it).
3. One `reportData` per row because in the XSD `companyDesc` (holds `remarks`) is a sibling of `transDet` inside `transDetList`. One list per row keeps each remark with its row (service comment lines 91-93).
4. Behaviour change to be honest about: Oracle returned ZERO rows when a date was null (`BETWEEN NULL AND NULL`); the new code returns ALL rows when no date is given. That differs from the original and can be unbounded. Needs a business decision (file 01 d.1 Risk 1).
5. Two open risks: UTC vs Malaysia time zone for the day window, and null customer names sort FIRST in Mongo but LAST in Oracle (Defect 4).

---------------------------------------------------------------------------------------------------

## Trace 7 - Payload alignment to `Pline.xsd`

| Step | Evidence |
|---|---|
| **Requirement** | Jira scope item (iv): XSD changes for all three reports. Reference files: `Pline.xsd` (1,400 lines) and `Common.xsd` (748 lines), v0.1.6. Mapping workbooks: `Pline_MASTER_MAPPING_SHEET_v0.1 (final1) 1 5.xlsx` (generated 2026-07-06) and `RejectRisk_XSD_Mapping_sheet.xlsx`. I only skimmed the master workbook (file 04 U5). |
| **Design** | Java `Pline*` payload classes mirror XSD complex types. `PlineReportParameters` has a `@JsonIncludeProperties` whitelist (73 names) so base-class fields from Motor/Fire/Bond do not leak. Shadow fields and `@JsonIgnoreProperties` give the XSD spelling (`ppidmprint`, `pPrintSeries`, `showComplaint`). `@JacksonXmlProperty(localName=...)` handles odd case (`LOADING`, `Columncount`, `Emailbcc`, `Policyletterind`). |
| **Configuration** | `spring.http.converters.preferred-json-mapper=jackson2` is needed so Jackson 2 annotations on the payload classes still work under Boot 4. |
| **DB / API** | Not changed by this work. |
| **XML** | Plain `new XmlMapper()` in `CommonController` writes null as `<x/>`. No namespace. |
| **XSD** | 22 named types in `Pline.xsd`, 66 in `Common.xsd`, 1,062 optional elements, only `xs:sequence` (order is strict). 64 `xs:dateTime`, no `xs:date`. |
| **Code** | Commit `6961abbf9` (2026-08-08, "Align Pline requestpayload types with XSD and add schema validation test"): types changed (BigDecimal / LocalDateTime instead of strings), illegal names renamed (`COLUMN_COUNT` -> `Columncount` and so on), `PlineXsdValidationTest` added. Commit `d4575fa5b` (2026-08-03) introduced the XSD files into the repo. Git author for both is Shinge. [CONFIRM: which XSD changes were yours] |
| **Testing** | `PlineXsdValidationTest` (119 lines, 6 tests). Only 2 are real (`minimalShellXml_isValid`, `knownInvalidXml_failsValidation`). The 4 "service generatesValidXml" tests validate one hand-written string and never call a service. No ASP or RejectRisk XSD test. |
| **Deployment** | Common facts. |

**Why this decision?**
1. The XSD is the contract with the print engine, so Java types and names follow it. Whitelisting `reportParameters` is the only way to stop shared Motor fields from appearing in Plines XML.
2. **The truth about validation (say this clearly):** real Java output does NOT validate against v0.1.6. Evidence from file 04 section 5 and 6.4:
   - Dates are written as `dd/MM/yyyy`; the XSD needs `xs:dateTime` (`2017-06-06T00:00:00`).
   - Null numbers/dates become empty elements; empty is invalid for decimal/dateTime/integer/boolean. In the old saved RejectRisk response this was 456 of 929 errors.
   - Sibling order is wrong in several types: `reportParameters` is emitted last (XSD wants first), `address` is last in agent/branch/comp/taxClient info (XSD wants first), and others.
   - `mainReportName`, `subReportName`, `pCreatedDateFrom`, `pCreatedDateTo`, `pReportTitle` are not in v0.1.6.
   - A hand-ordered RejectRisk XML with ISO dates and no empty numeric elements validated with 0 errors, so the XSD itself is workable.
3. The right sentence for an interview: "We aligned the payload types and structure to the XSD. End-to-end validation of real service output was still open. The existing test only proves the schema loads."

---------------------------------------------------------------------------------------------------

## Trace 8 - `mainReportName` and `subReportName`

| Step | Evidence |
|---|---|
| **Requirement** | One response now carries many Jasper reports and sub-reports. The print engine needs to know which `reportData` belongs to which main report and sub-report. No written requirement found in the repo. The Java comment cites "Pline.xsd v0.1.10" (`PlineReportData.java:20`, `PlineReportNames.java:13`). [CONFIRM: who asked for it and where it is documented] |
| **Design** | `PlineReportNames.tag(transactions, class)` fills `mainReportName` on every `reportData` where it is blank. `mainNameOf` builds it from the bean path (for Template12: `Template12PolicyDocument`). Services set `subReportName` per section (ASP does; Template12 does not). `PlineReportBlocks.reportHeader(...)` writes the trailing RejectRisk header block (`mainReportName=RejectRisk`, `subReportName=ReportHeader`). |
| **Configuration / DB / API** | None. |
| **XML** | Two extra elements inside every `reportData`, after `dataClass`. |
| **XSD** | **Not in v0.1.6.** A document with them fails: "Invalid content was found starting with element 'mainReportName'" (RAN by the earlier agent, file 04 section 5.4). No newer XSD (v0.1.7 to v0.1.10) exists on this machine or in the repo history. |
| **Code** | Commits on HEAD: `e74bc0d5c` (2026-10-07, "Added report, sub report name in resposne", 3 Template12 services) and `861f0666c` (2026-10-08, "Added report, sub report name in listing report"; RejectRisk service +9 lines). Both are git author Shinge. [CONFIRM: your part, if any] |
| **Testing** | RejectRisk: unit test asserts row 0 has `mainReportName="RejectRisk"` and `subReportName=null`. ASP: `allGatesOpen_oneReportDataPerSubreportInSubreportOrder` checks order. No test for `PlineReportNames.tag` on Template12 (file 02 section 8.3 point 5, inferred from names). |
| **Deployment** | Common facts. |

**Why this decision?**
1. "1 subreport = 1 reportData" only helps if each entry is labelled. These two fields make the label explicit and let the print engine bind data to the right jrxml.
2. Governance gap to state honestly: the Java emits the fields ahead of the XSD file we have. Either the real XSD is newer and lives elsewhere, or the output is schema-invalid. I could not tell which. Ask the schema owner.

---------------------------------------------------------------------------------------------------

## Trace 9 - Unit-test generation (OPUSM-44159)

| Step | Evidence |
|---|---|
| **Requirement** | Jira scope item (iii): unit tests / test-case generation for all three reports. The Sonar quality gate on PRs (`-Dsonar.qualitygate.wait=true`) is the visible pressure. Thresholds live on the Sonar server, not in the repo (file 05 section c). |
| **Design** | Layered unit tests. Handlers: `@Mock MongoTemplate`, `@InjectMocks`, capture `Aggregation` with `doAnswer` and call `toPipeline(Aggregation.DEFAULT_CONTEXT)`. Services: mock every handler, assert which branch fired. DTO/payload: fixed 6-test template. Naming: `method_condition_expected`. |
| **Configuration** | `mock-maker-inline` in `src/test/resources/mockito-extensions`. JaCoCo 0.8.12 `prepare-agent` + `report` at `test` phase; no `check` goal. |
| **Code (tests)** | Test base built 2026-09-25 to 2026-09-30 by several authors in package batches: `0ab953cf8` (135 DTO test files), `df70d6b5c`, `8fa89231e`, `3a3bba346` (handler batches B/C/D), `3f2157b28` (1,486 files, +203k lines, mixes tests with production edits), `db7588897` (service/client/config/utils tests). Sonar fixes: `24e50ab9b`, `c336f76fa`. [CONFIRM: which test files you wrote or reviewed] |
| **Numbers** | Template12 + ASP + RejectRisk name filter: 91 files, 865 `@Test`: handlers 246, DTO 224, payload 113, service 282. DTO + payload = 337 of 865 (about 39 percent) test Lombok code (counts, not run results). |
| **Deployment** | Tests run in `mvn verify` inside the Sonar workflow on PRs. |

**Why this decision? (and the honest quality view)**
1. Reason: protect service-layer branch logic from accidental refactor breaks, and satisfy the quality gate. Scenario names read like business rules (for example `execute_band9_excludedForPoscoverAzp_doesNotFire`).
2. Weak points: all Mongo access is mocked, so the tests prove pipeline SHAPE, not data meaning (null sort, date boundary, type mismatch cannot be caught). `fetchRejectRisks_pipelineLambdas_fullyForced` runs lambdas with no assertion (coverage-driven). Tests read `pipeline.get(1)` by position, which is why 7 of 15 RejectRisk tests break when a stage moves. Four XSD "tests" are placeholders. Some service tests encode known deviations as expected behaviour (PDS rows, ATC row).
3. No test run evidence exists in the working copy (`target` has only `classes` and `generated-sources`; no surefire or JaCoCo output). So pass/fail and coverage numbers are UNKNOWN. Do not quote coverage.
4. Do not claim the tests were AI-generated. The repo does not say. Say "consistent with template-style or tool-assisted generation".

---------------------------------------------------------------------------------------------------

## Trace 10 - Indicators from REST instead of Mongo (e-invoice and service-tax flags)

| Step | Evidence |
|---|---|
| **Requirement** | Jasper SQL calls Oracle functions: `FN_GET_EINVOICE_IND`, `FN_SHOW_STAX_INV_IND`, `FN_GET_COMP_BRN_NUMBER`, `FN_RPPM_GET_SIGNATURE`, `FN_GET_EINVOICE_PRINT_ENDT_IND`. Mongo cannot run Oracle functions. |
| **Design** | Each function is exposed as an OPUS REST API and called through `PlineUtilsClient` (fast-lookup RestTemplate, 8 s read) or `Pline*Client` (primary RestTemplate, 5 s connect / 20 s read). Failures are caught and defaulted (`"N"` or null). |
| **Configuration** | Base URL keys `API_BASE-URL` and `plines.api.base-url` are separate; path keys such as `plines.fn.show.stax.inv.ind.path`, `opus.fn-einvoice-print-pol-ind-path`, `plines.fn.comp.brn.number.path`. The same path appears under many keys (comp-brn-number in about 8 keys) - a maintenance trap (file 06 section 1.11). Truststore `mytruststore.p12` replaces JVM cacerts for outbound calls. |
| **DB** | Not used for these values. |
| **API** | Outbound GETs. SSRF guards: fixed base URL from config, `assertTrustedHost` (`startsWith(baseUrl)`), `sanitize()` regex on query values (commit `40d3fc1f7`, 2026-07-16). No retries, no circuit breaker. |
| **XML** | Values appear as flags such as `einvPrintInd`, and decide whether pages exist. |
| **XSD** | Flags exist as `transDet` leaves; some (`showGstInvInd`, `showStaxInvInd`) were left unmapped on purpose (comment `addPrimaryDocumentT12:536-538`: "no matching leaf found - left unmapped rather than guessing"). |
| **Code** | `PlineUtilsClient` (659 lines), `PlinePgRpgeUtilsClient`, `PlinePolicyRefClient`, `PlineSSTRolloutDateClient`. |
| **Testing** | Handler tests stub client exceptions and assert the default (`fnGetEinvoicePrintPolInd` throws -> "N"). No test calls a real endpoint. |
| **Deployment** | Environment is chosen by base URL keys. Moving dev2 to UAT to prod needs at least `API_BASE-URL`, `plines.api.base-url`, `spring.mongodb.uri`, `spring.mongodb.database` (file 06 section 1.12). How the real pods get these values is not visible (Helm repo not available). |

**Why this decision?**
1. Reproducing the Oracle logic in Mongo would copy business rules that other systems own. Calling the owner API keeps one source of truth.
2. Fail-soft keeps the report alive, but an outage turns invoice pages OFF with only a `log.warn` (file 02 Q13). Detection needs log search (Phase 13).

---------------------------------------------------------------------------------------------------

# PHASE 12 - DEFECTS AND ISSUES

Label meaning:
- **actual (fixed in commits)** = a real bug that the repo shows was found and fixed in a commit (I give the hash and whether the commit is on HEAD).
- **actual (not fixed)** = real and visible in the repo today, still open.
- **likely/latent** = I found it by reading code and Jasper. I did not see it fail in production. It may never have been reported.

Important: there are NO incident tickets or bug reports in the repo. "Symptom" below is what the code would do, not what a user reported, unless I say so. [CONFIRM: any real defects you personally found or fixed, with Jira keys]

Summary table:

| # | Defect | Label | Where fixed |
|---|---|---|---|
| 1 | ASP `CONTRACT_ID` type mismatch | actual (fixed in commits) | `578778080`, `ba9e29586` (other branch); `safeNum` is in HEAD handlers |
| 2 | RejectRisk date upper bound | actual (fixed in commits) | `cc6a1b84d` (other branch); same change uncommitted in your working tree |
| 3 | RejectRisk null date bounds | actual (fixed in commits) | same as 2 |
| 4 | Oracle NULL sort vs Mongo | likely/latent | not fixed |
| 5 | Template12 `Long.parseLong` on empty contractId | likely/latent (a test documents it) | not fixed |
| 6 | WIP vs live conflict, ORA-01427 emulation | actual (design, in commit) + latent inconsistency | `d4575fa5b` for Template12; ASP `AspServiceTaxInvoiceHandler` |
| 7 | Date format vs XSD | actual (not fixed) | open |
| 8 | Test-tree copy of `AspServiceTaxInvoiceHandler` | actual (not fixed) | open |
| 9 | `reportParameters` whitelist drops fields | actual design; dropped fields are latent | partly handled by shadow fields |
| 10 | Boot 4 property prefix traps | actual (fixed in commits) + stale keys latent | `9248b12ff`, `65f1a8711`, `c6b6803f3` |

---

## Defect 1 - ASP `CONTRACT_ID` type mismatch

**Label:** actual (fixed in commits).

- **Symptom:** the ASP tax and e-invoice sub-reports "silently never fire". No error, no 500. The report returns, but the service-tax or other gated pages are missing. In the V2 main handler all `showXxxInd` flags came back null.
- **Root cause:** Jasper declares `pContractId` as `Integer` in the main reports but as `String` in the landscape sub-reports. Oracle converts implicitly when it compares to a numeric column. MongoDB does not. `CONTRACT_ID` is stored as a number, so a String match returns nothing (file 03 sections 2.3 and 4.4; javadoc of `AspEndtDocument2MainHandler.buildCommonPipeline`, lines 102-115).
- **Investigation (what the evidence supports):** compare the Jasper parameter classes with the Java filter type; see that gated flags are null while the data exists; notice the match is by String. The steps you really took: [CONFIRM].
- **Fix:** convert before matching. `safeNum` = `Long.parseLong` with fallback to the raw string, in `AspAddMemberLandscapeDispatchHandler:123`, `AspDebitNoteHandler:393`, `AspEndtScheduleHandler:836`, `AspEndtDocumentMainHandler:154`. `AspServiceTaxInvoiceHandler.safeNum` tries Integer then Long (lines 461-465). `AspEndtDocument2MainHandler` overrides `buildCommonPipeline` to apply it, because the shared base class compares a String. Commits `578778080` (2026-08-25) and `ba9e29586` (2026-08-26), git author GS, are on `origin/OPUSM-41763-backup`, not on HEAD. The code with `safeNum` IS present in HEAD handlers; whether HEAD got it from a different commit I did not check. [CONFIRM]
- **Validation:** unit tests `fetchServiceTaxInvoice_contractIdNonNumeric_keptAsStringInMatch`, and dispatch handler tests for null and non-numeric contract id (`AspAddMemberLandscapeDispatchHandlerTest`). These test the conversion, not real data. No test on a real Mongo document.
- **Prevention:** always convert IDs before a Mongo `$match`; add an integration test with a numeric `CONTRACT_ID`; add a code review check "Mongo is type-strict".
- **Interview line:** "Oracle hid a type mismatch with an implicit cast. Mongo does not, and it fails silently with zero rows, not with an error."

## Defect 2 - RejectRisk date upper bound lost the last day

**Label:** actual (fixed in commits). Fixed in `cc6a1b84d` (2026-10-05, git author GS, branch `origin/OPUSM-41763/retrofit-develop-R2-pline-Template12-ASP-RejectRisk`, NOT in HEAD). Your working tree has the same change **uncommitted**.

- **Symptom:** rows created later on the "to" day are missing. With from = to = D, only rows at exactly midnight come back. (HEAD behaviour.)
- **Root cause:** Oracle used `TRUNC(CREATED_DATE) BETWEEN from AND to`, which includes the whole last day. The Mongo version compared the raw timestamp with `$lte` against the start of the "to" day.
- **Investigation:** read the SQL (JRXML lines 77-114) next to the pipeline and notice the `TRUNC`. [CONFIRM: how you found it]
- **Fix:** shift `dateTo` forward one day and use `$lt`: range `[from 00:00, to+1 day 00:00)`. No `$dateTrunc`, so an index on `CREATED_DATE` stays usable (handler lines 40-58; I checked that `$gte` and `$lt` are in the file).
- **Validation:** none that runs. The proper test is an integration test with rows at `D 00:00:00`, `D 23:59:59.999`, `D+1 00:00:00` and from = to = D (file 01 d.1). The old unit tests still assert `$lte` and will fail (see below).
- **Open risks you must mention:**
  1. Time zone. `toDate` builds UTC midnight. If stored values are true instants of Malaysian time (UTC+8), a row at 00:30 MYT falls in the previous UTC day. Other handlers format dates with `Asia/Kuala_Lumpur`, which hints that values are real instants (inferred). Check with one known record.
  2. 7 of 15 `RejectRiskHandlerTest` tests are expected to fail: `dateInstances_usedDirectlyInMatch`, `dateStrings_parsedViaDateUtils`, `nullDates_matchStageHasNullBounds`, `rejTypeOnly_addsSecondMatchStage`, `reasonCodeOnly_addsSecondMatchStage`, `rejTypeAndReasonCode_bothInSameMatchStage`, `customerNameAndIdNoAbsent_noExtraMatchStages` (static reading, not run).
  3. Your working tree also has an `application.properties` change that contains a Mongo URI with a password: `<redacted>`. Do not commit it.
- **Prevention:** assert tests by content (find the stage that has `CREATED_DATE`), not by position; add boundary-time integration tests.

## Defect 3 - RejectRisk null date bounds

**Label:** actual (fixed in commits). Same commit and same uncommitted change as Defect 2.

- **Symptom:** with no dates, or with only one date, zero rows come back, with no error.
- **Root cause:** the filter was always `{$gte: from, $lte: to}`. In MongoDB a comparison with `null` matches only null or missing values. So a null bound matches no real row.
- **Fix:** add a bound only if it is not null; add the date `$match` stage only if at least one bound exists.
- **Investigation / validation:** as in Defect 2. Unit test `fetchRejectRisks_nullDates_matchStageHasNullBounds` documented the OLD behaviour and will now fail.
- **Behaviour change (be honest):** Oracle returned zero rows for a null date (`BETWEEN NULL AND NULL` is unknown). The new code returns all rows when no date is given. That is NOT parity with Oracle and it is unbounded (full scan, in-memory sort, large XML). Business must decide: return empty to match Oracle, require at least one date, or cap the rows.
- **Prevention:** build optional filters from non-null values only, and decide the "no filter" rule explicitly with the business.

## Defect 4 - Oracle NULL sort versus Mongo

**Label:** likely/latent. Not fixed. Locked in by a test.

- **Symptom (expected, not seen):** in RejectRisk, rows with no matching partner or a null `NAME_EXT` appear at the TOP and take row numbers 1..k. In Oracle they appear at the bottom.
- **Root cause:** Oracle `ORDER BY x ASC` puts NULLs last. MongoDB sorts null and missing first.
- **Where else:** the same risk exists in `COV_SEQ_REF` sorts for ASP and Template12 landscape lists. Three Template12 handlers also sort the raw string field (`PlineT12ParticularOfInsuredLandscapeAmcHandler:38`, `PlineT12ParticularOfDiagnosisLandscapeHandler:41`, `...DiagnosisLandscapeAmcHandler:38`), so "10" sorts before "2" if the field is stored as a string (inferred; storage type not checked).
- **Investigation:** read the Oracle `ORDER BY` and the Mongo `$sort`. No data comparison was done.
- **Proposed fix (not done):** add a null flag to the sort key (`{_nullName:1, customerName:1, createdDate:1, _id:1}`) and use the same key in `$setWindowFields.sortBy`; or sort in Java with `Comparator.nullsLast` and assign the row number there. Add a tie-breaker.
- **Validation needed:** compare Oracle and Mongo output for the same date range, including null customers.
- **Prevention:** put "check NULL ordering" on the Oracle-to-Mongo translation checklist; extend `fetchRejectRisks_finalSortStage_sortsByCustomerNameAscending` to cover nulls.

## Defect 5 - Template12 `Long.parseLong` on an empty `contractId`

**Label:** likely/latent. I verified the line in the repo: `Template12LandscapeDispatchHandler.java:55  long cidLong = Long.parseLong(contractId);`.

- **Symptom (expected):** if `pContractId` is empty or not numeric, the whole Template12 request fails with `NumberFormatException`, because the landscape dispatch is not wrapped in try/catch. The caller gets HTTP 500 with the generic text. `pContractId` defaults to `""` in `Template12PolicyDocumentService`.
- **Root cause:** Landscape dispatch parses the id with `Long.parseLong`; other handlers use `asNum` (number if numeric, else raw string).
- **Evidence it is known:** unit tests `fetchDispatch_nonNumericContractId_throwsNumberFormatException` and `execute_landscapeDispatchHandlerThrows_notWrappedInTryCatch_propagates` lock the behaviour.
- **Fix (not done, idea only):** validate `pContractId` at the start of `execute` and return a clear 400; or treat it as "no dispatch". Decide with the caller owner.
- **Prevention:** validate required keys early; the controller has no `@Valid` on `PayLoadRequest`.

## Defect 6 - WIP versus live conflict, and the ORA-01427 emulation

**Label:** actual design (added in a commit as a deliberate emulation) plus latent inconsistency.

- **Background:** Oracle scalar sub-selects over `WIP_*` UNION live tables raise ORA-01427 ("single-row subquery returns more than one row") when WIP and live data disagree. Java copies this on purpose in the Template12 PolicyDocument primary fetch: `requireNoConflict` and `distinctValueOrConflict` throw `IllegalStateException` (`PlinePolicyDocumentMainHandler:1463, 1474, 1543-1585`; arrived with `d4575fa5b`). `addTemplate12PDPrimaryDocument` has no try/catch, so the request fails. In ASP the same idea is in `AspServiceTaxInvoiceHandler` (`requireSingleValue`, `requireNoConflict`).
- **Why it matters (symptom):** a bad WIP-vs-live data state shows up as a failed request, not as silent wrong output. That is intended. The problem is that the root cause is hard to see, because the generic handler does not log (Issue 11 below).
- **Latent inconsistencies:**
  1. PolicyDocument fails fast, but PolicyDocument2 and Landscape use `$ifNull` precedence and continue silently (file 02 D6, D19). The same contract can behave differently per section.
  2. In ASP, only the service-tax handler emulates it. The EndtDocument/EndtDocument2 `ERP` sub-selects, the landscape dispatch `UNION ALL` and the debit-note `MST_CONTRACT_ID` sub-select do not (file 03 section 4.3). The debit-note chain swallows the exception and sets `premByMstInd="N"`.
  3. `liveValue.equals(wipValue)` on Integer vs Long returns false for equal numbers, so a false conflict is possible (inferred).
  4. No direct test of `fetchTemplate12PolicyDocumentMain` and its conflict logic (file 02 section 8.3 point 1).
- **Validation:** `AspServiceTaxInvoiceHandlerTest.fetchServiceTaxInvoice_ambiguousEinvCutoff_throwsIllegalStateException` and `..._conflictingLiveAndWipVersionNo_throwsIllegalStateException`. Caution: these may run against the test-tree copy (Defect 8).
- **Prevention:** a shared helper and a documented rule "which handlers fail, which fall back"; tests with a WIP+live conflict fixture.

## Defect 7 - Date format versus XSD

**Label:** actual (not fixed). Known, written in the test file comment and the XSD header.

- **Symptom:** a strict XSD validator rejects dates such as `06/06/2017` where `xs:dateTime` is required. Also empty `<x/>` for null numbers or dates, and wrong sibling order.
- **Root cause:** every `LocalDateTime` has `@JsonFormat(pattern="dd/MM/yyyy")`. The Jasper side expects the `dd/MM/yyyy` text (the v0.1.6 header says `pBeforeEffDate` is populated with `new SimpleDateFormat("dd/MM/yyyy")`). The XSD says `xs:dateTime`. The XSD type was tightened to dateTime in v0.1.6 but Java still sends `dd/MM/yyyy`.
- **Template12 specific:** `Template12PolicyDocumentService` rewrites every String field with "date" or "dob" in the name to `dd/MM/yyyy` at the end (lines 2105-2230, applied at 549-554). Three formats exist at once: XSD dateTime, Java `dd/MM/yyyy`, Jasper `dd-MMM-yyyy`.
- **RejectRisk specific:** `createdDate` comes from Mongo as a BSON Date but the DTO field is a String; it is probably turned into text with `Date.toString()` (inferred) and parsed back by `DateUtils`. If parsing fails, `toLocalDateTime` returns null silently and the date disappears (I saw no log call in the catch blocks at service lines 141 and 156).
- **Investigation done earlier:** an earlier agent ran a JDK validator on saved RejectRisk responses: 929 and 1,766 errors on older output. A hand-ordered file validated with 0 errors. These counts describe older output (1 Sep 2026), not current HEAD.
- **Options (file 04 Q37):** (a) emit ISO and format in the report, (b) retype the XSD leaf to string or a pattern type, (c) add a parallel string leaf. None was chosen. The team must agree with the report side.
- **Prevention:** a real marshal-and-validate test with a ratchet baseline (errors must not grow); one shared `XmlMapper` bean so test and production cannot drift.

## Defect 8 - Test-tree copy of `AspServiceTaxInvoiceHandler`

**Label:** actual (not fixed).

- **Symptom:** a 483-line file `src/test/java/com/agcs/fo/opus/plines/documentHandler/AspServiceTaxInvoiceHandler.java` has the SAME package and class name as the 476-line production class. It has no `@Test`.
- **Root cause:** looks like an accidental copy during Sonar clean-up. It was added or edited by `3f2157b28` (on HEAD) and `24e50ab9b` (not on HEAD). Differences are about 17 lines (`catch (Exception e)` vs `catch (RuntimeException e)` in `safeCall`, `SafeCallable.call() throws Exception`, comment edits).
- **Consequence (inferred, not run):** Maven puts `test-classes` before `classes`, so `AspServiceTaxInvoiceHandlerTest` (27 tests) probably tests the copy, not the real class. JaCoCo reads `target/classes`, so the production class would show less coverage than the tests suggest, and a bug in production could pass.
- **Fix (not done, idea only):** delete the copy and re-run the 27 tests. To prove which class loads, print `AspServiceTaxInvoiceHandler.class.getProtectionDomain().getCodeSource()` in a test.
- **Prevention:** a CI check that no class in `src/test` has the same fully qualified name as a class in `src/main`.

## Defect 9 - `reportParameters` whitelist drops fields

**Label:** actual design (the whitelist is real). The dropped fields are latent (found by reading; not executed).

- **What it is:** `PlineReportParameters` has `@JsonIncludeProperties` with 73 names. Any setter that writes a property not in the list is silently dropped from the XML. No error.
- **Dropped in Template12 (file 04 section 3.1):** `pContractId`, `pPolicyNo`, `versionNo`, `companyParamType`, `showPaymentInfo`, `copyType`, `pCertNoFrom/To`, `pTaxInvNo`, `pissueDate`, `psignIssueDate`, `pMstPolicyNo`, `pContractIdList`, and `ShowOutPrintCovLetter` (the setter name differs from the whitelisted `pShowOutPrintCovLetter`).
- **Dropped in ASP:** `pEndtNo`, `pRefNo`, `pContractId`, `pVersionNo`, `pCompanyParamType`, `pUserID`, `pissueDate`. Also likely `pshowGstLabel/pshowSstLabel` (ASP calls the base setters; Template12 also calls the upper-case ones) - not verified by running.
- **RejectRisk:** `reportName` is set but not whitelisted, so it is dropped. `pReportTitle`, `pCreatedDateFrom`, `pCreatedDateTo` are whitelisted but not in the v0.1.6 XSD, so they are schema-invalid.
- **Why the whitelist exists:** the base class is shared with Motor/Fire/Bond and has many more fields. Without it they would leak into Plines XML.
- **Prevention:** a reflection test that compares the setters used by each service with the whitelist and lists what is dropped; log a warning when a set value is dropped.

## Defect 10 - Spring Boot 4 property and library traps

**Label:** actual (fixed in commits) for the upgrade fixes; stale keys are latent.

- **Scope note:** the Boot 3.5 to 4.1.1 upgrade (commits `65f1a8711`, `c6b6803f3`, `9248b12ff`, branch family `OPUSM-44159/...spring-boot-4-upgrade...`) is a platform task. It is not in your Jira scope as you gave it. Use it only as "problems I had to understand because my modules run on it". [CONFIRM whether you worked on it]
- **Trap 1 - Mongo prefix.** Boot 4 uses `spring.mongodb.*`. The old `spring.data.mongodb.*` keys are silently ignored and the app falls back to `localhost:27017`. Symptom: connection to localhost, or empty reports against the wrong database. Fixed in `9248b12ff` (2026-09-18). Comment lines 26-31 of `application.properties` explain it.
- **Trap 2 - Jackson 3 default.** Boot 4.1.1 prefers Jackson 3 (`tools.jackson`), which ignores the `com.fasterxml.jackson` annotations used by `PayLoadRequest` and the payload classes. Fix: `spring.http.converters.preferred-json-mapper=jackson2`. Without it every POST fails with `HttpMessageConversionException`.
- **Trap 3 - BOM property name.** Under Boot 4, `jackson-2-bom.version` pins Jackson 2. `jackson-bom.version` now means Jackson 3, so the old pin would not apply.
- **Trap 4 - other library moves:** `springdoc` 2.8.15 broke on Boot 4 (`NoClassDefFoundError`), so 3.1.1; the Micrometer 1.15.12 pin broke every `@SpringBootTest`, so the pin was removed; `@MockBean` became `@MockitoBean`; `WebClient.Builder` needs an explicit bean; `UriComponentsBuilder.fromHttpUrl` became `fromUriString`.
- **Latent leftovers (file 06 section 1.2):** `auto-index-creation=false` and `option.min-connection-pool-size / max-connection-pool-size / max-wait-time` are not Boot keys and nothing reads them. The Mongo pool is NOT sized by them; real driver defaults apply. Logging keys that target `com.agcs.fo.opusMotorGateway.*` point to a package that does not exist. `spring.profiles.active=dev` activates nothing because no `application-dev.properties` exists.
- **Prevention:** after a major upgrade, search for properties nobody reads; check the Mongo connection log line; keep a CVE-pin note per property.

## Other issues I saw while reading (latent, not in the ten above)

| # | Issue | Evidence |
|---|---|---|
| 11 | `GlobalExceptionHandler.handleGeneric` returns 500 but I found no logger call in the file, so the root cause is invisible unless the thrower logged it. | I grepped the file for `log.` and found nothing. |
| 12 | Unknown or merged-away `reportName` returns HTTP 200 `SUCCESS` with a default transaction. Old names `.../PolicyDocument2`, `.../Landscape`, `.../EndtDocument2`, `.../AddMemberLandscape`, `.../DeletionMemberLandscape` fall in this path (inferred from the `@Service` annotations; the legacy `switch` default was not fully read). | file 06 section 2.2 |
| 13 | No authentication or authorization: `anyRequest().permitAll()`. | `SecurityConfig.java` |
| 14 | ASP main handler anchors on `UWMT_VI_ENDT_TAX_INV_ALT_UPV` with a `$match` on ENDT_NO + CONTRACT_ID. The shared base class comment says that collection is not related and gave empty DTOs. An endorsement or checklist copy with no row there gives an all-null DTO and closes the `070412` / 72 / 168 gates. Branch commit `cc6a1b84d` rewrites it to per-field lookups; not on HEAD. | file 03 section 4.1 #3. A risk, not a confirmed production bug. |
| 15 | EndtDocument2 zero-row rule is not reproduced; PREM could become 1 when the row is missing (`$ne` of a missing field and 0 is true in Mongo, inferred). | file 03 section 4.1 #2 |
| 16 | Request body printed with `System.out` (personal data such as `pIdNo`, `pCustomerName`); `MongoTemplate` DEBUG logs pipelines with values. | file 06 sections 1.4, 4.5 |

---------------------------------------------------------------------------------------------------

# PHASE 13 - PRODUCTION SUPPORT

## 13.1 What exists and what does not

- **No real incident tickets exist in the repo.** I found no incident log, no Service-Now key, no post-mortem, no alert definition.
- The only support-like evidence: `CHANGELOG.md` line 13: "2026-09-21 OPUSM-52530 Reverted aggregation and lookup code changes unable to reproduce on BAU UAT (PR #864)". It proves a UAT environment exists and that UAT can find problems unit tests miss. It is not stated to be about your three modules.
- The stories and runbooks below are **constructed from artifacts**: they are what a support person would do given this code. They are not memories. Adapt them to what really happened. [CONFIRM: any real support cases you handled, ticket keys, environment, outcome]

## 13.2 Logging facts you must know before you debug

| Fact | Evidence |
|---|---|
| Logs go to stdout with Boot defaults. No `logback.xml`. No correlation id. No JSON log format. | file 06 section 4.4 |
| `MongoTemplate` and `Aggregation` loggers are at DEBUG in the committed file. This prints pipelines with values (contract ids, names, IDs). | `application.properties` lines 20, 23 |
| Several logging keys point to wrong package names, so they do nothing. To see your handlers use `logging.level.com.agcs.fo.opus.plines.documentHandler=DEBUG` (suggested in file 06 Q21). | file 06 section 1.4 |
| `CustomRequestFilter` prints the full request body with `System.out.println`. | file 06 section 2.2 |
| The legacy path logs the full response XML at INFO. The strategy path (your three services) logs it at DEBUG. | file 06 section 2.3 |
| `GlobalExceptionHandler` returns 500 and I found no logger call there. `IllegalArgumentException` returns 400, message only logged at WARN. | file 06 section 2.4; my grep |
| Template12: `log.info("Template12LandscapeDispatchHandler contractId={}")`; 88 `log.warn` calls in `Template12PolicyDocumentBaseService` (my grep count) for swallowed sub-report failures. | file 02 Q25; grep |
| ASP: `log.info("AspEndtDocumentService endtNo={} contractId={}")` at line 232. `AspDebitNoteHandler.fetchMstContractInfo` logs a WARN when the master-contract chain fails. REST failures are swallowed by `safeCall` (null). | grep; file 03 |
| RejectRisk: the handler logs only `createdDateFrom` and `createdDateTo` at INFO (line 36). I saw no log call in the catch blocks of `RejectRiskService` (lines 141, 156), so a date parse failure is silent. | grep |
| `GET /actuator/health` and `/info` are open. Nothing else is exposed (no metrics, no prometheus, no loggers endpoint). The `loggers` endpoint is enabled but not exposed, so log levels cannot be changed at runtime. | file 06 section 1.5 |

**Monitoring today = actuator health only.** No custom health indicator, no alert in the repo. Anything beyond that is outside this repo (Atlas monitoring, pod monitoring, gateway). [CONFIRM: what the platform team really monitors]

## 13.3 Common first steps for any "report is wrong or missing" ticket

1. Get the exact request: `reportName`, `pContractId` / `pEndtNo` / `pRefNo`, `versionNo`, `checklistCopy`, `show*` flags, and the environment.
2. Check the report name against the bean name. Template12: `AGIC/UW/Plines/PolicySchedule/Template12/PolicyDocument`. ASP: `AGIC/UW/Plines/ASPEndtSchedule/EndtDocument`. RejectRisk: `AGIC/UW/Plines/RejectRisk/RejectRisk`. An old or wrong name gives HTTP 200 and an empty default transaction.
3. Check the response `status` AND the XML. HTTP 200 does not mean data. The envelope fields are `status`, `data` (XML string), `timestamp`. (The README says `xmlData`; the class has `data`. Check a real response.)
4. Which database is the pod using? A wrong DB name gives empty reports, not errors. The URI is decided outside the repo. [CONFIRM how to see it in your environment]
5. Reproduce with the same payload in a lower environment. Use the Swagger UI path `/gi/documentprintdata/swagger-ui/index.html` or `curl` to `POST /gi/documentprintdata/api/opus/GenerateReport`.
6. If you can deploy a config change, raise the log level for `com.agcs.fo.opus.plines`.

## 13.4 Runbook - Template12 (PolicyDocument, PolicyDocument2, Landscape)

**Detection.** Print team or QA says: a page is missing (landscape, debit note, service-tax invoice, complaint page); the whole call fails with 500; or a page is extra (PDS marker, ATC page).

**Quick causes.**
- 500 for the whole report: empty or non-numeric `pContractId` gives `NumberFormatException` at `Template12LandscapeDispatchHandler:55`; or a WIP-vs-live conflict gives `IllegalStateException` "Ambiguous ... ORA-01427" from `fetchTemplate12PolicyDocumentMain`.
- Landscape page missing: dispatch returned null because the scheme has `GPA_SCHEME / CI_LANDSCAPE_PRT = Y` in `CMGE_CODEREL_DET` (this is intended); or the band flag (`benefitPrint`, `nomineePrint`, `diagnosisPrint`, `aspCutoffInd`) is 0.
- Tax or service-tax invoice missing: `showTaxInvoice` is not `Y`; `SHOW_STAX_INV_IND` or `EINV_PRINT_IND` came back `N` because the REST call failed and defaulted; or `TAX = 0`.
- Debit note missing: `showDebitNote` not `Y`, BMW scheme, or `copyType = BC`.
- Wrong page for a product: product and scheme differ between the three primary fetches.
- A field is missing from `reportParameters`: the whitelist dropped it (Defect 9).

**Log analysis.** Search for `Template12LandscapeDispatchHandler contractId=<id>` to confirm the request arrived. Search the `log.warn` lines from `Template12PolicyDocumentBaseService` helpers for the failing sub-report (they name the handler). Search `PlineUtilsClient` GET-failed lines for REST problems. **Not logged:** which gates were true or false; which band fired; the final list of `reportData` names; the dropped `reportParameters`. For these you need DEBUG on Mongo or a debugger.

**RCA steps.**
1. Confirm `pContractId` is numeric text.
2. Read the product and scheme that the three fetches derive (run the three handler pipelines in Compass for that contract; collection names are in file 02 section 3).
3. Compare with the Jasper condition in file 02 sections 1.1 to 1.3.
4. For a missing invoice, call the REST endpoint by hand (`stax-inv`, `invoice-print-indicator`) and compare.
5. Check WIP and live collections for the contract. WIP rows have no `VERSION_NO`; live rows use `VERSION_NO = 1`.
6. Ask whether the print engine treats multiple `reportData` as separate pages (open question).

**Resolution.** Data fix by the data owners (WIP/live conflict, missing scheme code); fix the request (numeric `pContractId`); fix the REST API or its config (base URL, path, truststore); code fix only after deciding the Jasper rule. Never re-enable `Template12PolicyDocument2Service` or `Template12LandscapeService` (every band would print twice).

**Monitoring.** Actuator health only. Suggested but not present: count of HTTP 500, count of base-service `log.warn`, latency per report name.

## 13.5 Runbook - ASP (EndtDocument and the merged reports)

**Detection.** Service-tax invoice or QR missing; debit note missing; add-member or delete-member landscape missing; cover or schedule empty; master-customer block wrong.

**Quick causes.**
- Service-tax invoice missing: the four-part gate (`showTaxInvoice=Y`, `SHOW_STAX_INV_IND=Y`, `checklistCopy != Y`, `SHOW_EINV_IND=Y`). `SHOW_EINV_IND` comes from REST `invoice-print-endt-indicator`; on failure it is empty, which closes the gate silently.
- All gated flags null: `CONTRACT_ID` type mismatch (Defect 1).
- Cover or schedule empty on a checklist copy, or on an endorsement with no row in the anchor collection: Issue 14 (all-null DTO).
- Debit note missing: `showDebitNote` not `Y`, or `DIFF_PREM_DUE = 0` (prem 0).
- Landscape missing: dispatch counts are 0 because `ENDT_CODE` is not 72/168 (add) or 98 (delete), or there is no risk with `ACTION_CODE=A` / `C` with `TERMINATE_DATE`.
- Master-customer block missing: `fetchMstContractInfo` failed and fell back to `premByMstInd="N"` (logged as WARN).
- Old report name used: empty default transaction.

**Log analysis.** Search `AspEndtDocumentService endtNo=... contractId=...`. Search the WARN `AspDebitNoteHandler.fetchMstContractInfo: mstContractId chain query failed`. REST failures in `safeCall` return null and may log little; check the `PlineUtilsClient` failure lines. **Not logged:** the value of each gate (`showSt`, `showGst`, `showTran`), the dispatch counts, and which `reportData` were emitted.

**RCA steps.**
1. Get `pEndtNo`, `pRefNo`, `pContractId`, `versionNo`, `checklistCopy`.
2. Check `UWGE_VI_ENDT_REPORT_PARAM` for the flags (`SHOW_GST_INV_IND`, `SHOW_STAX_INV_IND`, `SHOW_TRAN_INV_IND`) for that contract and endorsement.
3. Call the e-invoice endt indicator API by hand with the same ids.
4. For landscape, run the dispatch queries for that endorsement: version row with `ENDT_CODE`, risk with `ACTION_CODE`, tax and nominee views.
5. For a checklist copy, remember `checklistCopy=Y` closes the SST gate by rule (Jasper has the same rule).
6. Check types: `CONTRACT_ID` must be numeric.

**Resolution.** Fix upstream data or the REST service; fix the request; for code, follow the Jasper condition exactly (do not add a product gate that the jrxml does not have). Known gaps are not incidents: stubs (`ASPLLB`, `ASPSTI`, schedule of benefits), `INST_IND` and `SHOW_GST_LABEL` cut-off rules, six unmapped delete-member fields.

**Monitoring.** Same as above. Suggested: a counter of "SST gate closed because the indicator was empty".

## 13.6 Runbook - RejectRisk

**Detection.** A listing has no rows, is missing rows for the last day, has rows in a strange order or numbering, or is very slow or huge.

**Quick causes.**
- No rows with no dates: old behaviour (null bounds), or Oracle parity (zero rows).
- Missing last-day rows: old `$lte` bug (Defect 2), or a time zone shift.
- Blank customers at the top: Defect 4.
- Empty although rows exist: wrong DB, or a type mismatch on `CP_PART_ID` / `VERSION` (strings vs numbers; not verifiable here).
- Dates missing in some rows: `toLocalDateTime` swallowed a parse failure.
- No header block: by design when there are zero rows.
- Slow: no dates given (full scan).

**Log analysis.** Search `RejectRiskHandler createdDateFrom=... createdDateTo=...`. That is all the service logs. **Not logged:** the other filters (name, ID, type, reason), the row count, parse failures. With `MongoTemplate` DEBUG you see the pipeline and values (careful: personal data).

**RCA steps.**
1. Take the date range and filters.
2. Run the aggregation by hand in Compass against `CMUW_PL_REFER_RISK` with the same range.
3. Check one boundary record's `CREATED_DATE` as stored (UTC instant vs wall clock).
4. Check `CPGE_VI_PARTNERS_ALL` for the `PART_ID` and `VERSION`.
5. Check `CMGE_CODE` categories `PLREJ_RISK_TYPE`, `PLREJ_REASON`, `PLREJ_REVOKE_DUR`.
6. Compare with the Jasper query (file 01 section a.4).

**Resolution.** Data fix; request fix (always send both dates); code fix for null sort and time zone (not done); decide the no-date rule with the business. Do not commit the local `application.properties` change.

**Monitoring.** Same as above. Suggested: row count and elapsed time per call.

## 13.7 Runbook - platform issues that hit all three

| Symptom | Likely cause | What to check |
|---|---|---|
| Every call returns 500 after a deploy | Jackson 3 default (missing `preferred-json-mapper=jackson2`), or bean creation failure (wrong truststore password gives a startup failure) | startup log; `application.properties` |
| Reports empty with no error | pod points to the wrong Mongo DB, or old `spring.data.mongodb.*` keys under Boot 4 (silent fallback to localhost) | Mongo connection log line; env overrides |
| Pages that need an indicator vanish | upstream OPUS API down or slow; calls time out at 8 s or 20 s and default | `PlineUtilsClient` failure logs; API health |
| Calls hang then fail | no inbound timeout configured; Mongo `maxTime` is 600 s in many handlers | thread dump; Atlas profiler |
| TLS error to an API | custom truststore replaces JVM cacerts; new CA missing | `mytruststore.p12` |
| Environment change had no effect | two base URL keys (`API_BASE-URL` and `plines.api.base-url`) | change both |
| Unknown caller using the API | no authentication | gateway and network logs (outside repo) |

## 13.8 Support stories - constructed from artifacts, adapt to your real experience

Each story is built from code and docs. None is a real ticket.

**Story S1 - "Service-tax invoice page is missing for an ASP endorsement"**
- Situation: a tester says the service-tax invoice and QR are missing for an ASP endorsement; other pages are fine.
- Task: find which condition closed the gate.
- Action: read the four-part gate (guide B.4 step 9); check `showTaxInvoice`; check the flags in `UWGE_VI_ENDT_REPORT_PARAM`; call `invoice-print-endt-indicator` by hand; check `checklistCopy`.
- Result: qualitative only. The cause is one of: request flag, data flag, checklist copy rule, or empty indicator from the API. [CONFIRM: your real finding]

**Story S2 - "Template12 call returns 500 for a PolicyDocument print"**
- Situation: the whole Template12 request fails.
- Task: find out whether it is a request problem or a data problem.
- Action: check `pContractId` is numeric (`Long.parseLong` at `Template12LandscapeDispatchHandler:55`); look for "Ambiguous ... ORA-01427" in the log; if present, compare WIP and live product and scheme for the contract.
- Result: either a caller fix or a data fix. [CONFIRM]

**Story S3 - "RejectRisk shows fewer rows than Jasper for the last day"**
- Situation: business compares a listing with the old report; the last day is short.
- Task: explain and fix.
- Action: compare the Oracle `TRUNC(CREATED_DATE) BETWEEN` with the Mongo `$lte`; change to `[from, to+1 day)`; note the tests that must be updated; note the time zone question.
- Result: whole-day parity for UTC-stored data; time zone and test updates still open. [CONFIRM]

---------------------------------------------------------------------------------------------------

# PHASE 14 - PERFORMANCE

## 14.1 Honest statement

- **There are no measurements in the repo.** No load test, no APM trace, no slow-query log, no timing, no surefire or JaCoCo output. I will not invent latency, throughput or "x percent faster" numbers.
- Everything below is a **risk read from code**. Each one has a way to measure it.
- Only one performance change is visible in history: commit `62b3c94a7` (2026-01-28, "Performance issue fixes.") which added `auto-index-creation` and `option.*` keys. These keys are not read by anything (file 06 section 1.2), so that change had no effect on the pool (inferred). It is not in your scope.

## 14.2 Risks from code

**P1 - `$lookup` without indexes.** RejectRisk does 4 `$lookup` per matched row (partner plus 3 `CMGE_CODE`). Without an index on the foreign fields this is a scan per row. The repo creates no indexes (no `createIndex`, no `@Document`). Real indexes may exist in Atlas; unknown. Evidence: file 01 g.3, file 06 section 3.3. Measure: `explain("executionStats")` in Compass or mongosh; look for `COLLSCAN` and `totalDocsExamined`; list the indexes on `CMGE_CODE`, `CPGE_VI_PARTNERS_ALL`, `CMUW_PL_REFER_RISK`. Direction (idea): indexes on `CMGE_CODE{CAT_CODE,CODE_CD}`, `CPGE_VI_PARTNERS_ALL{PART_ID,VERSION}`, `CMUW_PL_REFER_RISK{CREATED_DATE}`; or load the three code maps once per request.

**P2 - Customer name and ID filters run after the lookup.** With name or ID but no dates, every refer-risk row is looked up first. Evidence: file 06 section 3.4 example 3. Measure: the same `explain`, run with a name only. Direction: require a date range; filter partners first when a name or ID is given.

**P3 - "No dates returns all rows" (new behaviour).** Unbounded scan; in-memory `$sort` and `$setWindowFields` (100 MB per blocking stage, and RejectRisk sets no `allowDiskUse`); all rows mapped into heap and then into XML. Evidence: working tree; file 01 g.3. Measure: call with no dates against a copy of the data and watch heap and response size. Direction: business decision, require a date or cap rows.

**P4 - Unconditional dispatch queries in ASP.** The add and delete dispatch handlers run on every request and add 5 to 6 extra Mongo queries, even for endorsements that are not member changes. Evidence: file 03 section 4.1 #1. Measure: turn on `MongoTemplate` DEBUG and count queries per call; time the handlers. Direction: gate the dispatch by `ENDT_CODE` from the main DTO.

**P5 - Many small aggregations per Template12 request.** The default CIPrint path alone can call 10 or more handlers; BMW and PA4A loop per row; FWPA does several lookups. Three primary fetches derive product and scheme separately. Evidence: file 02 Q22 and section 3. Measure: count `MongoTemplate.aggregate` calls per request (DEBUG log or a counter) and trace one request. Direction: one shared product/scheme context; run independent handlers in parallel; cache `SAPM_SYS_CONSTANTS` and `CMGE_CODE`.

**P6 - Reflection-based merging.** `mergeSingleTransDet` copies non-null fields over every existing `transDet` with reflection; `PlineTransDet` has about 650 leaves; the date reformat pass walks every String field. Cost grows with (single-row bands) x (`transDet` count) x (fields). Evidence: `Template12PolicyDocumentBaseService:189-258`, `Template12PolicyDocumentService:2105-2230`. Measure: profile one large request (JFR or a simple timer around `buildTransactions` and the reformat). Direction: merge once at the end, or use a typed mapper.

**P7 - Repeated master-policy (MST) resolution.** The MST chain (`UWGE_POLICY_BASES` + `WIP_UWGE_POLICY_BASES`, then `OCP_POLICY_BASES`, then `PREM_BY_MST_IND`) can run up to 3 times per ASP request; the debit note alone needs 4 to 5 round trips. Evidence: guide B.9, file 06 example 5. Measure: count calls to `fetchMstContractInfo` per request. Direction: resolve once and pass the result.

**P8 - `allowDiskUse` and `maxTime` 600 s.** Many handlers allow 10-minute server-side queries while the HTTP read timeout of callers may be about 20 s (caller side unknown). A query can keep running after the client gave up. RejectRisk and `AspEndtDocumentMainHandler` set no options. Evidence: file 06 section 3.5. Measure: Atlas profiler or `currentOp` for long queries; compare with the caller timeout. Direction: lower `maxTime` to match the caller timeout; set options for RejectRisk.

**P9 - HTTP timeouts and pools.** Two RestTemplate stacks.
- Pline beans: connect 5 s, read 20 s, connection-request 3 s, max 100 total and 20 per route.
- `fastLookupRestTemplate` (used by `PlineUtilsClient`): read 8 s.
- `PlinePolicyRefClient`, `PlineSSTRolloutDateClient` and `PlinePgRpgeUtilsClient` take an unqualified `RestTemplate`, so they get the `@Primary` motor `restTemplate` (5 s / 20 s; the library default pool is 25 total / 5 per route, inferred).
- No retries, no circuit breaker. Calls run on the request thread.
- Worst case for one Template12 request: up to about 5 distinct utility calls, each up to 8 s if the API hangs (file 06 section 1.7).
Measure: time each outbound call (add timers, or use gateway logs); inject a slow stub and measure end-to-end time. Direction: parallel calls; shorter timeouts for non-critical flags; cache stable values (company BRN, SST rollout date).

**P10 - Mongo pool is not configured the way the file suggests.** The `option.*` keys are dead. Driver defaults apply (max pool size 100). Measure: connection count in Atlas under load. Direction: put pool options in the URI.

**P11 - Logging cost.** `MongoTemplate` and `Aggregation` DEBUG on, `System.out` request body print, legacy path logs full XML at INFO. Evidence: file 06 section 1.4. Measure: compare latency with DEBUG on and off in a test environment. Direction: DEBUG only in non-prod.

**P12 - Response size.** Payload classes use `Include.ALWAYS` and a plain `new XmlMapper()`, so a `transDet` with about 650 leaves writes mostly empty elements. RejectRisk repeats the full structure per row (about 8 objects per row). Evidence: file 04 section 5.3, file 01 g.3. Measure: bytes of the XML for a known request. Direction: `NON_EMPTY` inclusion (empty numeric elements are invalid for the XSD anyway, so this also helps validity).

## 14.3 How to measure, in order (a plan you can state in an interview)

1. Pick 3 to 5 real requests per report (small, typical, large). [CONFIRM: sample ids from the data owners]
2. Turn on `MongoTemplate` DEBUG in a test environment. Count queries and read the pipelines.
3. For each slow-looking pipeline, run `explain("executionStats")`. Record `COLLSCAN` vs `IXSCAN`, `totalDocsExamined`, `nReturned`.
4. Wrap each outbound call with a timer and log the duration.
5. Run JFR, or simple timers, around the pure-Java parts: `buildTransactions`, the reflective merge, the date reformat, `XmlMapper.writeValueAsString`.
6. Repeat with a slow stub for one REST API and check that the 8 s and 20 s timeouts really cut the call.
7. Only then change one thing at a time and measure again.

## 14.4 What to say if asked "did you optimise performance?"

"I did not run performance tests, and there are none in the repo. I read the code for risks: lookups with no known indexes in RejectRisk, the unbounded result when no date is given, ASP dispatch queries that always run, reflection merging in Template12, and the gap between `maxTime` 600 s and the 20 s HTTP timeout. I know how I would measure each one: `explain`, query counts with DEBUG, timers around outbound calls." Then name only what you really did. [CONFIRM: any real performance work]

---------------------------------------------------------------------------------------------------

# UNCERTAINTIES FOR THIS FILE

1. Which commits are yours: git has none under `NISHUNISARGA`. Every `[CONFIRM]` above needs your answer.
2. Whether the CONTRACT_ID fix on HEAD came from `578778080` / `ba9e29586` or from another commit.
3. The real storage form of `CREATED_DATE` (UTC instant vs wall clock) and of `COV_SEQ_REF`, `CONTRACT_ID`, `CP_PART_ID` (string vs number) was not checked on live data.
4. Whether indexes exist in Atlas; whether `*_VI_*` names are real Mongo views or collections.
5. Test results: no test was run. All "will fail" statements come from reading code.
6. XSD newer than v0.1.6 (v0.1.7 to v0.1.10 are only code comments).
7. How the real environment is deployed and monitored (Helm repo and platform tooling not available).
8. The meaning of "ASP", of the collection prefixes `SB_` and `OCP_`, and whether the renderer treats multiple `reportData` entries as separate print units.
