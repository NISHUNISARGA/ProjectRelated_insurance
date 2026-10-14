# 07b - Ownership Answers and STAR Stories
> **AUTHORSHIP UPDATE (confirmed by the developer): git author `GS <extern.gs_nisarga@allianz.com>` is the developer. Wherever this file says commits cannot be attributed to the developer, read it as: the 11 GS commits listed in `KT_Parts/14_My_Commits_And_Ownership.md` ARE theirs; commits by other authors still must not be claimed. No GS commit touches `src/test`, so test authorship is still `[CONFIRM]`.**

Service: `opus-documentprintdata-service` (Spring Boot 4.1.1, Java 17, MongoDB, XML-in-JSON report data).
Period: 12 May 2026 to 30 Dec 2026. Your modules: Template12 (PolicyDocument, PolicyDocument2, Landscape), ASP (EndtDocument, EndtDocument2, AddMemberLandscape, DeleteMemberLandscape), RejectRisk.
Your Jira scope: (i) analysis of all three reports and all sub-report levels, (ii) code for all, (iii) unit tests and test-case generation for all, (iv) XSD changes for all.

This file covers Phase 18/19 (ownership answers) and Phase 20 (STAR stories). The traces, defects, support runbooks and performance notes are in `07a_Trace_Defects_Support_Perf.md`. Detail files: `01` (RejectRisk), `02` (Template12), `03` (ASP), `04` (XSD), `05` (tests), `06` (config, API, DB, DevOps).

---------------------------------------------------------------------------------------------------

## 0. How to use this file (read first)

1. **Answers are written in first person, but only inside your Jira scope.** Everything I put in "I" form is either (a) part of the four scope items, or (b) a code fact you can show on screen. I do NOT write "I wrote commit X". Git has no commit under your user name (`NISHUNISARGA`). Authors in git are Shinge, Karunakar, Shaw, GS, Amarnath Yadav, Neethu and others. If an interviewer asks "which commits are yours?", the honest answer is a `[CONFIRM: ...]` item you must fill in before the interview.
2. **`[CONFIRM: ...]` = a personal fact only you know.** Replace it or delete the sentence. Do not leave a guess in.
3. **Results are qualitative.** No measured numbers exist in the repo. I never write a percentage, a time saved or a defect count that was measured. Counts of code (tests, lines, files) are counts from the repo, not results of your work.
4. **Stories are labelled "constructed from artifacts - adapt to your real experience".** They are built from the code, the Jasper files and the git history. They are not your memories. Change them to what really happened, or do not use them.
5. **Keep the known truths in mind in every answer:**
   - One live bean each for Template12 and ASP. The other classes are merged in; their `@Service` is commented out.
   - XSD v0.1.6 is the only XSD available. Real Java output does NOT validate against it (dates `dd/MM/yyyy` vs `xs:dateTime`; empty elements; sibling order; `mainReportName` / `subReportName` are not in the XSD).
   - "ASP" is not expanded anywhere. Say: "the jrxml calls it ASPlus internally, benefit codes are ASP-A to ASP-D, and Java ties it to product 070412; I did not find a written expansion."
   - The RejectRisk fix in the working tree equals commit `cc6a1b84d` on another branch, is uncommitted, and likely breaks 7 of 15 handler tests.
   - Oracle sorts NULLs last, Mongo sorts them first.
   - The API has no authentication. An unknown report name returns HTTP 200 `SUCCESS` with a default transaction.
6. **Format of each answer:** *Short answer* (20 to 40 seconds, say it aloud) then *Detailed answer* (1 to 3 minutes, for follow-ups). After that, *Be ready for* lists likely follow-ups.

---------------------------------------------------------------------------------------------------

# PART A - MODULE OWNERSHIP ANSWERS

## A1. Template12 (PolicyDocument, PolicyDocument2, Landscape)

**Q: Tell me about the Template12 work you owned.**

*Short answer.*
"Template12 is a Personal Lines policy-schedule pack. In Jasper it was three reports: PolicyDocument, PolicyDocument2 and Landscape, with many sub-reports. My scope was to analyse all three with every sub-report level, write the Java for them, write the unit tests and handle the XSD changes. In Java it now runs as one Spring bean, `Template12PolicyDocumentService`, which returns the policy pages, the debit note and invoices, and the landscape pages in one response."

*Detailed answer.*
"The service receives `POST /api/opus/GenerateReport` with a report name and filters like `pContractId`, `checklistCopy`, `showDebitNote`, `showTaxInvoice`. It does not make the PDF. It gives XML inside a JSON envelope, and the print engine makes the PDF.

I started from the Jasper files. PolicyDocument has a one-row main query that gives the product code and scheme name. Then each band has a `printWhenExpression`. For example, product 070406 prints the ATC variant, 070414 prints PA4A, and 070504 picks a variant by scheme: ASTRO, BMW, POSCOVER, AZP, POSMESRA, FWPA, and Elite as the fallback. PolicyDocument2 prints the debit note and the GST, service-tax and transitional invoices. Landscape has 16 bands and is switched off for schemes flagged `CI_LANDSCAPE_PRT` in `CMGE_CODEREL_DET`.

In Java I copied the same decision tree. `execute()` reads the product and scheme, sets booleans like `is070504`, and calls one helper per sub-report. About 95 helpers sit in `Template12PolicyDocumentBaseService`. Each helper calls a Mongo handler, which is a replacement for the SQL, and maps the result into the Pline payload classes.

Two design points I can explain. First, PolicyDocument2 and Landscape were merged into the one bean, so the print engine gets one response and no band is rendered twice. Second, the class has its own base class so that changes cannot break the other 24 templates that share the common base.

I am also honest about the limits. Page-number conditions in Jasper, such as blank pages for duplex printing, cannot be computed in a data service, so those are left to the renderer. A few outputs differ from Jasper on purpose or by accident: the PDS marker rows are printed in Java but the Jasper band is disabled with `false &&`. And the output does not validate against the XSD v0.1.6 we have."

*Be ready for:*
- "How many handlers or sub-reports?" - About 95 `add*` helpers; about 30 Template12-named handler classes plus shared Common and Generic handlers; 87 sub-report files in the Jasper folder (file 02 Q1).
- "Why three primary fetches?" - They copy three different Oracle behaviours: PolicyDocument fails when WIP and live disagree (ORA-01427 emulation), PD2 prefers posted data with `$ifNull`, Landscape has its own chain. They were not unified because that would change PD2 behaviour without product sign-off (class javadoc "CRITICAL FACT #1").
- "Which commits are yours?" - [CONFIRM: list your Template12 commits or files. Git shows the merge c0c6d449a as Amarnath Yadav and the XSD and payload commits as Shinge, so do not claim those unless true.]
- "What did you test?" - See A5.

## A2. ASP (EndtDocument, EndtDocument2, AddMemberLandscape, DeleteMemberLandscape)

**Q: Tell me about the ASP work you owned.**

*Short answer.*
"ASP is a group personal-accident style product. Its endorsement documents were four Jasper reports. My scope was to analyse them with all sub-reports, write the Java, write the tests and do the XSD work. Now one bean, `AspEndtDocumentService`, returns the cover, schedule, debit or credit note, tax and service-tax invoices, and the add-member and delete-member landscape pages. I do not know what ASP stands for. I did not find a written expansion in the code or the reports."

*Detailed answer.*
"I analysed the four reports. EndtDocument has the cover, the schedule and the schedule of benefits. EndtDocument2 has the complaint page, debit or credit note, GST tax invoice, service-tax invoice with e-invoice QR, and the transitional invoices. AddMemberLandscape and DeletionMemberLandscape are landscape tables for added or deleted members, sponsors and nominees.

In Java, `execute` reads flags like `showCoverNote`, `showEndtSchedule`, `showDebitNote`, `showTaxInvoice`, calls two main handlers, and then checks the same conditions as the jrxml. For example, the service-tax invoice needs four things: `showTaxInvoice=Y`, `SHOW_STAX_INV_IND=Y`, `checklistCopy` not `Y`, and `SHOW_EINV_IND=Y`. The response has one `reportData` per sub-report. The first one is a master row with `mainReportName` and `subReportName`.

Things I can explain well:
- The type bug. Jasper uses a String contract id in some sub-reports and Oracle converts it. Mongo does not, so the match returned nothing and the gated pages silently did not print. The handlers now use `safeNum`.
- The landscape dispatch queries run on every request. The counts tell the print engine whether to print the landscape pages. This costs extra queries.
- Three pages are stubs: schedule of benefits, ASPLLB and ASPSTI are static text in Jasper. The Java handlers return a dummy value and no row is emitted.
- Known gaps: the `INST_IND` and `SHOW_GST_LABEL` cut-off rules are not evaluated in the schedule handler; ORA-01427 is emulated only in the service-tax handler; six delete-member fields have no `transDet` leaf."

*Be ready for:*
- "What does ASP stand for?" - "I did not find an expansion. The jrxml has internal names ASPlusSTI and ASPlusLLB, benefit codes ASP-A1 to ASP-D5, and Java ties it to product 070412." Do not invent one.
- "Why one bean for four reports?" - One response per request and no duplicate bean keys. The old `AspEndtDocument2Service` and `AspEndtDocumentLandscapeService` are kept disabled for reference.
- "What is still missing on your branch?" - Commit `cc6a1b84d` (not on HEAD) rewrites the V1 main handler to per-field lookups and adds a company-name handler. `[CONFIRM: whether it was merged into your target branch.]`
- "Which commits are yours?" - `[CONFIRM]`. Git shows `7a925920c`, `578778080`, `ba9e29586` as GS, and `4cd1cad2e`, `e74bc0d5c`, `861f0666c` as Shinge.

## A3. RejectRisk

**Q: Tell me about the RejectRisk work you owned.**

*Short answer.*
"RejectRisk is a listing report of rejected risks with reason, remarks and revoke details for a date range. In Jasper it is one Oracle SQL with an outer join and `ROWNUM`. I analysed it, wrote the Mongo pipeline and the service, and aligned the output to the XSD. While working on it I found two bugs in the date filter: the 'to' day lost its later rows, and a missing date matched nothing. The fix is in my working tree and equals an existing commit on another branch."

*Detailed answer.*
"The handler reads `CMUW_PL_REFER_RISK`, joins the partner view for the name and IDs, and looks up three code descriptions in `CMGE_CODE`. Row numbers come from `$setWindowFields` with `$documentNumber`, sorted by customer name, which is the Mongo way to do `ROWNUM` after `ORDER BY`. The service builds one `reportData` per row because, in the XSD, the remarks field lives in `companyDesc`, which is a sibling of `transDet`; one list per row keeps each remark with its row. After the rows it adds a ReportHeader block with `mainReportName=RejectRisk`, `subReportName=ReportHeader`, as in the Jasper hierarchy.

The date bugs. Oracle uses `TRUNC(CREATED_DATE) BETWEEN from AND to`, which includes the whole last day. The old Mongo code used `$lte` with the start of the last day, so later rows on that day were lost. The fix: add one day to the upper bound and use `$lt`. This keeps `CREATED_DATE` bare so an index can still be used. The second bug: the filter was built even when a date was null, and in Mongo a compare with null matches only null values, so no real rows came back. Now only supplied bounds are added.

What I must be honest about:
1. Behaviour change. Oracle returned zero rows with no dates. The new code returns all rows. The business must decide.
2. It is uncommitted, and it equals `cc6a1b84d` on branch `OPUSM-41763/...`.
3. 7 of 15 handler tests are expected to fail because they check `$lte` and stage positions. I read the code; I have not run the tests.
4. Open: time zone (UTC midnight vs Malaysia time) and NULL sort order (Oracle last, Mongo first).
5. My working tree also has a Mongo URI with a password in `application.properties`. I will not commit it."

*Be ready for:*
- "How did you reproduce `ROWNUM`?" - `$setWindowFields`, `$documentNumber`, needs MongoDB 5.0+ (inferred).
- "Why not `$dateTrunc`?" - It blocks index use.
- "How would you test the boundary?" - Integration test with rows at `D 00:00:00`, `D 23:59:59.999`, `D+1 00:00:00` and from = to = D.
- "Is the disclaimer in the service a bug?" - It is an open contract assumption: the renderer has not confirmed it treats each `reportData` as one print unit.
- "Which commits are yours?" - `[CONFIRM]`. In git, RejectRisk files were touched by Shinge, GS, Shaw, Karunakar; none under your name.

---------------------------------------------------------------------------------------------------

# PART B - ACTIVITY OWNERSHIP ANSWERS

## B1. Analysis (Jira item i)

*Short answer.*
"I analysed the three reports and every sub-report level from the Jasper files. For each band I recorded the exact `printWhenExpression`, the parameters passed, the SQL source, and where the data now lives in Mongo. I compared Jasper and Java line by line, and listed the differences."

*Detailed answer.*
"The real specification was the `.jrxml.data` files, not a document. For Template12 the folder has 187 files; for ASP the hierarchy CSVs show up to three levels; RejectRisk has one sub-report, ReportHeader. I read each main report to get its parameters, its one-row query and its bands. For each sub-report I wrote down: file name, print condition, parameters, SQL tables or Oracle functions, nested sub-reports. Oracle functions like `FN_GET_COMP_BRN_NUMBER` are not translated to Mongo; they are called as OPUS REST APIs.

From this I built traceability tables: Jasper sub-report -> condition -> Java handler -> Mongo collection -> payload field -> XSD path. Findings that came out of it: five PD2 bands that are disabled with `false &&`; conditions that use `$V{PAGE_NUMBER}`, which Java cannot evaluate; the ASP landscape parameters that are String in Jasper but numeric in Mongo; the ASP stubs. I also checked product and scheme matrices so I could tell which variant prints for which product."

*Be ready for:*
- "How do you know the dump matches production?" - I do not. The compiled `.jasper` of PD2 was only string-searched; the readable jrxml was treated as authoritative. `[CONFIRM: whether you checked against a deployed report]`.
- "Did you have a functional spec or Jira text?" - `[CONFIRM]`. I only have the Jira scope line, the Jasper files and the mapping workbooks.
- "What did the analysis find that surprised you?" - e.g. the `false &&` bands; the PD2 Java section calls insured and nomination helpers that no PD2 band calls (file 02 D4).

## B2. Development (Jira item ii)

*Short answer.*
"I wrote the Java for all three modules: services that copy the Jasper print conditions, handlers that replace the SQL with Mongo aggregations, DTOs, and the payload mapping. I used the existing layers: Controller, Service, Handler, DTO, request payload."

*Detailed answer.*
"The flow is: `CommonController` finds a `ReportStrategy` bean by report name; the service calls handlers; handlers use `MongoTemplate` with raw `Document` stages; the service maps DTOs to the Pline payload classes; `XmlMapper` writes XML into the JSON envelope.

Patterns I used again and again for Oracle to Mongo: a scalar sub-select becomes `$lookup` with `$limit 1` and `$arrayElemAt`; `UNION` of WIP and live becomes two lookups with `$ifNull` or `$unionWith`; `SELECT DISTINCT` becomes `$group`; `ROWNUM` becomes `$setWindowFields`; `FROM DUAL` becomes a fallback DTO; Oracle functions become REST calls. For ORA-01427, I copied the error with `requireNoConflict` where the Jasper SQL would fail. IDs are converted with `safeNum` because Mongo is type-strict.

Design choices to explain: fail-soft in sub-bands (log a warning and return neutral data) but fail-hard for the primary fetch; one `reportData` per sub-report; reflection merge in Template12 to avoid row explosion; own base class for Template12.

Limits I accept: very large classes (`AspEndtDocumentService` 1,701 lines, `Template12PolicyDocumentService` 2,235), duplicated landscape mapping, stubs, and no data-level comparison with Jasper output."
`[CONFIRM: which classes and handlers you wrote yourself. Git shows other authors for most commits.]`

*Be ready for:* "What would you refactor?" - A table-driven band registry, one shared product/scheme context, a real XSD validation test, gating the ASP dispatch by endorsement code.

## B3. Unit tests (Jira item iii)

*Short answer.*
"I wrote or generated unit tests for the three modules with JUnit 5 and Mockito. Handler tests mock `MongoTemplate` and check the pipeline. Service tests mock the handlers and check which branch fires and how fields are mapped. There are also DTO and payload tests. The tests are good for protecting branch logic but they do not prove the data matches Jasper, because there is no real Mongo in them."

*Detailed answer.*
"For Template12, ASP and RejectRisk by file name there are 91 test files and 865 `@Test` annotations: handlers 246, DTO 224, payload 113, service 282. These are counts from grep, not run results. The names follow `method_condition_expected`, for example `execute_band9_excludedForPoscoverAzp_doesNotFire` and `showServiceTax_checklistCopyY_closesGate`, so they read like business rules.

The test base was built between 25 and 30 September under OPUSM-44159, by several people in package batches, with Sonar fix commits after. `[CONFIRM: which test files or batches were yours, and whether you used a tool to generate cases and how you reviewed them.]`

Honest weaknesses: 337 of the 865 tests are for Lombok classes; some tests only run lambdas for coverage (`fetchRejectRisks_pipelineLambdas_fullyForced`); tests read stage positions, which is why 7 RejectRisk tests will break with the date fix; Mockito is set to lenient in 36 files; four XSD 'service' tests validate a hard-coded string; a copy of `AspServiceTaxInvoiceHandler` in `src/test` may shadow the real class; some tests lock known deviations as expected behaviour. There is no test-run output in the working copy, so I cannot give pass rates or coverage.

What I would add first: delete the duplicate class; fix the RejectRisk tests to assert by content; add Mongo integration tests with boundary and null cases; replace the XSD placeholders with real output validation."

*Be ready for:* "What coverage did you reach?" - "I don't have a measured number; there is no JaCoCo output in the repo. `[CONFIRM: if you saw the Sonar numbers, state them with the date]`."

## B4. XSD changes (Jira item iv)

*Short answer.*
"I mapped each module to `Pline.xsd` and `Common.xsd` v0.1.6, and aligned the payload classes: names, types and structure. I can show the mapping for RejectRisk field by field. I will not claim that the service output validates against the XSD, because it does not yet."

*Detailed answer.*
"The XSD has a root `transactions` with no namespace, 22 Pline types and 66 Common types, 1,062 optional elements, and only `xs:sequence`, so order is strict. For RejectRisk, all 17 `transDet` leaves exist with the right types; `REMARKS` goes to `transDetList/companyType/companyDesc/remarks`. For Template12 and ASP, the data goes to `transDet`, its lists (`coverList`, `sobList`, `nomineeList`, `diagnosisList`, `staxList`), `endorsementDetails`, `prntDetails/*` and `reportParameters`. When no leaf exists the field is left unmapped with a comment, not guessed.

What I know about the gap between code and XSD, from an earlier validator run:
- Dates are `dd/MM/yyyy` but the XSD has `xs:dateTime`.
- Null numbers and dates become empty elements, which are invalid for decimal and dateTime.
- Several types have wrong sibling order (`reportParameters` last, `address` last in four info types).
- `mainReportName`, `subReportName`, `pCreatedDateFrom`, `pCreatedDateTo`, `pReportTitle` are not in v0.1.6. The Java comments mention v0.1.7 to v0.1.10 but no such XSD file exists on this machine or in the repo.
- A hand-ordered RejectRisk file with ISO dates validated with 0 errors, so the schema itself works.

The existing `PlineXsdValidationTest` proves only that the schema loads. My suggestion: marshal a real service result with the production `XmlMapper` and validate with an error-collecting handler, with a ratchet baseline."
`[CONFIRM: which XSD-related commits or mapping rows were yours. Git shows d4575fa5b and 6961abbf9 as Shinge. Also confirm whether you raised the XSD version gap with the schema owner.]`

*Be ready for:* "Is the output valid?" - No, and I can list why. "Who owns the XSD?" - `[CONFIRM]`.

## B5. Defects

*Short answer.*
"The defects I can speak about are the ones I can show in code: the ASP contract-id type mismatch, the two RejectRisk date bugs, the Template12 empty contract-id failure, the WIP-versus-live conflict handling, the date format against the XSD, the test-tree duplicate class, the reportParameters whitelist, and the Boot 4 property traps. For each I can give the symptom, root cause, fix and how I would validate it."

*Detailed answer.*
"I split them honestly into two groups. Fixed in commits: the ASP contract-id mismatch (`safeNum` in handlers; fix commits on another branch), RejectRisk date bounds (`cc6a1b84d`, and uncommitted in my tree), and Boot 4 property fixes. Latent, found by reading: Oracle NULL sort versus Mongo, `Long.parseLong` on an empty contract id (line 55 of `Template12LandscapeDispatchHandler`), inconsistent WIP/live conflict handling, the dd/MM/yyyy date format against the XSD, the duplicate class in `src/test`, and fields dropped by the whitelist.

I tell the interviewer clearly which group each one belongs to. There are no incident tickets in the repo, so I do not describe any as a production incident."
`[CONFIRM: any defect you personally found, with the Jira key and what you did.]`

*Be ready for:* "Which was the hardest?" - Pick the contract-id one: it was silent, no error, and it only made sense after comparing the Jasper parameter classes with Mongo storage.

## B6. Configuration

*Short answer.*
"My modules need almost no report-specific configuration. The configuration I had to understand is the platform side: Mongo URI and database, the base URLs and paths for OPUS APIs, the HTTP timeouts, the truststore, and the Boot 4 property names."

*Detailed answer.*
"Key facts. Context path is `/gi/documentprintdata`. Mongo uses `spring.mongodb.uri` and `spring.mongodb.database` in Boot 4; the old `spring.data.mongodb.*` keys are ignored and the app falls back to localhost. `spring.http.converters.preferred-json-mapper=jackson2` keeps the Jackson 2 annotations working. External calls use two separate base-URL keys, `API_BASE-URL` and `plines.api.base-url`, so a change must be made in both. The same API path is repeated under many keys. Timeouts: connect 5 s, read 20 s, and 8 s for the fast-lookup client. Some keys do nothing (`auto-index-creation`, `option.*` pool keys). The profile `dev` activates nothing because no profile file exists. The committed file has a plaintext truststore password; the working copy has a Mongo URI with a password, which I write as `<redacted>` and do not commit."
`[CONFIRM: whether you changed any configuration, and for which environment.]`

*Be ready for:* "How do you point the app at another database?" - Override `spring.mongodb.uri` and `spring.mongodb.database` by environment variable or command line, never by editing and committing the file.

## B7. API

*Short answer.*
"There is one real endpoint, `POST /gi/documentprintdata/api/opus/GenerateReport`. The report is chosen by `filterCondition.reportName`, which must equal a bean name. The response is `{status, data, timestamp}` where `data` is the XML as a string. It is always HTTP 200 unless an exception happens, and there is no authentication."

*Detailed answer.*
"The request has `metadata` (template and mailing details) and `filterCondition` (all values are strings). If `metadata` is missing the controller throws an NPE (inferred). Errors: `IllegalArgumentException` gives 400; anything else gives 500 through a generic handler that I found does not log the exception. An unknown report name, including the merged-away names, returns 200 `SUCCESS` with a default empty transaction, so the caller must check the XML. Outbound, the services call OPUS APIs for e-invoice indicators, service-tax indicator, company BRN, signature, SST rollout date and policy reference. Failures are swallowed and defaulted, which keeps the report alive but can silently remove pages. SSRF guards: a fixed base URL, a `startsWith` host check, and a `sanitize` regex on query values. RejectRisk makes no external call."
`[CONFIRM: who calls this API in your environment and how you tested it, for example with curl or Swagger.]`

*Be ready for:* "How would you secure it?" - A filter that checks a gateway secret or JWT, `authenticated()` for all but health, network policy, rate limits. "What would you change in the error contract?" - Return 404 or an error status for an unknown report name instead of 200.

## B8. Deployment and support

*Short answer.*
"The pipeline is GitHub Actions. PRs are gated by Sonar, secret scan, a Jira key check and an AI review. The image build and deploy is a manual workflow that calls a shared action; Helm lives in another repo. For support, the first checks are the exact request, the bean name, the XML and not just the status, and which database the pod uses. I know where the logs help and where they do not."

*Detailed answer.*
"What I can show: `sonar-code-review.yml` runs `mvn -B verify` and waits for the quality gate on PRs to R1 and R2; `jira-commit-check.yml` needs an `OPUSM-` key; `cicd-workflow.yml` is `workflow_dispatch` with an optional `imageTag`; the pom version is static, so the image tag is the release id. What I cannot show: Helm values, environments, who deployed, or the rollback procedure. Rollback is probably redeploying the previous image tag (inferred).

For support, I wrote runbooks for each module in `07a`. Examples: Template12 500 error usually means a non-numeric `pContractId` or a WIP/live conflict. ASP service-tax page missing means one of four gate conditions, often an empty indicator from the e-invoice API. RejectRisk missing last-day rows means the old `$lte` bound or a time zone shift. Monitoring is actuator health only. There are no real incident tickets in the repo, so the runbooks are constructed from the code."
`[CONFIRM: your real release or support involvement: which environments, which builds, which tickets.]`

---------------------------------------------------------------------------------------------------

# PART C - TOUGH FOLLOW-UP QUESTIONS (short answers you can say aloud)

These are the questions where a weak answer hurts most. Each answer is short and honest.

1. **"Is your XML valid against the XSD?"** - "No, not end to end. We aligned the types, names and structure to v0.1.6, and a hand-ordered RejectRisk file validates with zero errors, so the schema is workable. The real output still has `dd/MM/yyyy` dates instead of `xs:dateTime`, empty elements for null numbers, wrong sibling order in some types, and the new `mainReportName` and `subReportName` fields that the v0.1.6 file does not have. The existing test only proves the schema loads."
2. **"Why are `mainReportName` and `subReportName` not in your XSD?"** - "The Java comments cite a newer XSD, v0.1.10, but I only have v0.1.6. So either the schema owner has a newer file or the output is ahead of the contract. I would ask the schema owner and then update one side." `[CONFIRM: who asked for these fields.]`
3. **"What does ASP stand for?"** - "I did not find a written expansion. The reports call it ASPlus internally, the benefit codes are ASP-A to ASP-D, and Java ties it to product code 070412."
4. **"Are the tests good?"** - "They protect branch logic and read like business rules, but all Mongo access is mocked. They cannot catch NULL sort order, date boundaries or type mismatches. About 39 percent of the tests in these modules are for Lombok classes. I cannot give coverage because no report is in the repo."
5. **"Did you run the tests after your RejectRisk change?"** - `[CONFIRM]`. If not: "Not yet. By reading, 7 of 15 handler tests assert `$lte` and stage positions and will fail. I would update them to find the stage by content."
6. **"Why is your RejectRisk fix uncommitted?"** - `[CONFIRM: the real reason.]` "The same fix exists in commit `cc6a1b84d` on another branch, so I was bringing it over, and the tests still need updating."
7. **"What happens when no date is given in RejectRisk?"** - "Now all rows come back. Oracle returned none. It needs a business rule: return empty, require a date, or cap the rows."
8. **"What is the difference between Oracle and Mongo sorting that matters here?"** - "Oracle puts NULLs last on ascending order; Mongo puts null and missing first. So customers without a partner row would get the first row numbers in RejectRisk. Not fixed yet."
9. **"Why does a bad `pContractId` give a 500 in Template12?"** - "The landscape dispatch calls `Long.parseLong` and is not wrapped in try/catch. A test documents it. A better answer is to validate early and return a clear 400."
10. **"What happens if an OPUS API is down?"** - "The call is swallowed and defaulted. The report still returns with HTTP 200, but invoice pages that depend on an indicator can silently disappear. A log warning is the only trace."
11. **"Is the API secure?"** - "There is no authentication in the service. `anyRequest().permitAll()`. It depends on the network or gateway upstream. CSRF is disabled because it is stateless. Swagger is open."
12. **"What does a wrong report name return?"** - "HTTP 200 with status SUCCESS and a default empty transaction. The caller must read the XML. I would change that to an error status."
13. **"Why did you not unify the three primary fetches in Template12?"** - "The code says unifying would change PolicyDocument2 behaviour without product sign-off. It is a known trade-off: the three fetches can disagree on product or scheme."
14. **"Why is there a copy of a production class in the test folder?"** - "It looks like an accidental copy from Sonar clean-up. It probably shadows the real class, so the 27 tests may not test production code. I would delete it and re-run." (inferred, not run)
15. **"Did you do performance work?"** - "I did not measure anything; there are no measurements in the repo. I listed the risks from code and how I would measure each: lookups without known indexes, no-date RejectRisk, ASP dispatch queries, reflection merge, and the gap between a 600 s query limit and a 20 s HTTP timeout." `[CONFIRM: any real performance work.]`
16. **"What would you do first if you had one more week?"** - "One: remove the duplicate class and the credential from the working tree. Two: fix the 7 RejectRisk tests. Three: add Mongo integration tests with boundary and null cases. Four: a real XSD validation test with a ratchet. Five: agree the date format and the new fields with the report team."
17. **"What did you learn?"** - "Oracle hides type and ordering behaviour that Mongo does not. A green unit test with a mocked database proves shape, not meaning. Contracts like the XSD need a real test, and silent failures need logs."

---------------------------------------------------------------------------------------------------

# PART D - STAR STORIES (Phase 20)

**Label for every story below: constructed from artifacts - adapt to your real experience.**
They are built from the repo, the Jasper files and git history. They are not memories. Before you use one, check each fact against what you really did, replace `[CONFIRM: ...]`, and drop any step that is not yours. Results are qualitative. No measured numbers exist, so none are given. Counts shown (tests, files) are repo counts.

---

## STAR 1 - Feature implementation: Template12 merged into one service

*Constructed from artifacts - adapt to your real experience.*

- **Situation.** Template12 was three Jasper reports (PolicyDocument, PolicyDocument2, Landscape) with 87 sub-report files. In Java they existed as three services. The print engine needed one response, and registering all three would print bands twice. Evidence: class javadoc, commit `c0c6d449a`.
- **Task.** Make sure the Java reproduces the Jasper print conditions for all three reports and returns a single combined response. [CONFIRM: your exact task or Jira key]
- **Action.**
  1. Read each main report: the one-row query, parameters, and each band's `printWhenExpression`.
  2. Recorded the product and scheme decision matrix and the 16 landscape band conditions.
  3. In `execute()`, kept the Jasper booleans (`is070504`, `is070412`, BMW, POSMESRA...), and called one helper per sub-report; merged the PD2 and Landscape bodies into private methods.
  4. Left the old two beans unregistered so nothing renders twice.
  5. Wrote service tests per product and scheme branch.
  6. Listed the differences from Jasper (page-number conditions, `false &&` bands, extra PD2 rows) so nobody is surprised.
  [CONFIRM: which of these steps you did yourself]
- **Result.** One bean and one response for the Template12 pack, each branch with a named test. Known limits stay open: one very large class, the renderer contract for multiple `reportData` is unconfirmed, and the XSD gap. [CONFIRM: any feedback or sign-off you received]

## STAR 2 - Complex defect: ASP contract id type mismatch

*Constructed from artifacts - adapt to your real experience.*

- **Situation.** In the ASP endorsement response, gated pages such as the service-tax invoice did not appear, and no error was raised. The cause was hidden: the flags that open the gates came back null.
- **Task.** Find why the flags were null and make the gates work. [CONFIRM: whether this was your task]
- **Action.**
  1. Compared Jasper parameter classes: the main reports use `Integer` for `pContractId`, but the landscape sub-reports use `String`. Oracle converts implicitly; Mongo is type-strict.
  2. Saw that `CONTRACT_ID` is stored as a number, so a String match returns zero rows without error.
  3. Added `safeNum` (`Long.parseLong`, fallback to the raw string) in the ASP handlers, and an override of `buildCommonPipeline` in `AspEndtDocument2MainHandler` because the shared base class compares a String.
  4. Added tests for numeric, null and non-numeric contract ids.
  5. Wrote the reason in the javadoc so it is not repeated.
  [CONFIRM: your real investigation steps. Git shows the original fix commits `578778080` and `ba9e29586` as GS on another branch.]
- **Result.** The gates open when data exists. The lesson: Oracle can hide type differences, Mongo cannot, and a silent zero-row result needs a test with the right types. Still open: no test on real Mongo data. [CONFIRM: how you verified it]

## STAR 3 - Production-type issue: RejectRisk missing rows on the last day

*Constructed from artifacts - adapt to your real experience. No real incident ticket exists; this is a production-type scenario.*

- **Situation.** The Jasper report counted the whole last day. The Mongo version dropped later rows on the 'to' date, and a missing date returned nothing. Users comparing with the old report would see short or empty lists.
- **Task.** Make the filter match `TRUNC(CREATED_DATE) BETWEEN from AND to`, and make optional dates safe.
- **Action.**
  1. Read the SQL and saw `TRUNC`; read the pipeline and saw `$lte` against midnight.
  2. Changed to a half-open range: `>= from`, `< to + 1 day`. Kept `CREATED_DATE` bare for index use.
  3. Added only the bounds that are supplied, and skipped the stage if none.
  4. Removed a dead `ROW_NUMBER` stage; switched the service to the Plines `DateUtils`.
  5. Found that the same fix is in `cc6a1b84d` on another branch; noted that 7 of 15 handler tests assert `$lte` and positions and need updating.
  6. Kept the local Mongo URI out of the commit.
  7. Listed open risks: time zone, NULL sort, the no-date rule.
  [CONFIRM: which steps are yours; whether tests were updated]
- **Result.** Whole-day ranges behave like Oracle for UTC-stored data. Open items are named with an owner decision needed for the no-date rule. [CONFIRM: any user feedback]

## STAR 4 - API issue: report name does not match, empty answer with status 200

*Constructed from artifacts - adapt to your real experience.*

- **Situation.** After the merge, old report names like `.../Template12/PolicyDocument2` or `.../ASPEndtSchedule/EndtDocument2` have no bean. The API returns HTTP 200 `SUCCESS` with a default empty transaction, so a caller can think it worked.
- **Task.** Explain why the response was empty and tell callers which name to use. [CONFIRM: whether a caller actually hit this]
- **Action.**
  1. Traced `CommonController`: it looks up the bean by name, falls back to the legacy `switch`, and builds a default transaction when nothing is produced.
  2. Confirmed the three live bean names from the `@Service` annotations.
  3. Wrote the support rule: always check the bean name, and the XML, not just the status.
  4. Suggested returning an error status for an unknown report name.
- **Result.** A clear rule for callers and support, and one improvement proposal. The behaviour itself is unchanged. [CONFIRM]

## STAR 5 - XSD modification: aligning RejectRisk to Pline.xsd

*Constructed from artifacts - adapt to your real experience.*

- **Situation.** The RejectRisk output had to fit `Pline.xsd`. The XSD has strict order, only `xs:dateTime` for dates, and the remarks field lives in `companyDesc`, a sibling of `transDet`.
- **Task.** Map every Jasper field to an XSD path and type, and keep the payload classes in line. (Jira scope item iv)
- **Action.**
  1. Listed the 20 Jasper fields, their classes, and the XSD leaf and type (`rownumber` decimal, `createdDate` dateTime, `reasoncode` and `updatedby` with their odd casing).
  2. Put `REMARKS` into `transDetList/companyType/companyDesc/remarks`, so the service builds one `reportData` per row.
  3. Added the trailing ReportHeader block that mirrors the Jasper sub-report.
  4. Checked the mapping workbook and the Java setters; noted that `pCreatedDateFrom`, `pCreatedDateTo` and `pReportTitle` are not in v0.1.6.
  5. Ran, via an earlier agent, a JDK validator: a hand-ordered file passed, real output failed on date format, empty elements, order and new fields.
  [CONFIRM: which XSD edits or mapping rows were yours]
- **Result.** A documented field-level mapping and a clear list of what still fails and why. I do not claim the output validates. Next step: a real marshal-and-validate test and agreement with the schema owner on the date format and new fields.

## STAR 6 - XML change: one `reportData` per sub-report, with report names

*Constructed from artifacts - adapt to your real experience.*

- **Situation.** The combined responses (ASP, Template12) held many Jasper sub-reports in one XML. The print engine could not tell which data belonged to which report or sub-report.
- **Task.** Make each sub-report one `reportData`, and label each with `mainReportName` and `subReportName`.
- **Action.**
  1. For ASP, built the response as a master row plus one `reportData` per sub-report in a fixed order, with a service test that locks the order.
  2. For RejectRisk, one `reportData` per row, plus a header block.
  3. Used `PlineReportNames.tag` to fill `mainReportName` from the bean name, and set `subReportName` per section.
  4. Noted that the new fields are not in the XSD file we have, and that the renderer's handling of multiple `reportData` is unconfirmed.
  [CONFIRM: your part. Git shows `e74bc0d5c` and `861f0666c` as Shinge.]
- **Result.** Each section is addressable in the XML. Open: schema alignment for the new fields and renderer confirmation. [CONFIRM]

## STAR 7 - Performance: reading code for risks and planning measurements

*Constructed from artifacts - adapt to your real experience. No performance numbers exist.*

- **Situation.** Reports query Mongo many times and call REST APIs. There is no load test or timing data.
- **Task.** Find the places most likely to be slow and say how to measure them, without inventing numbers.
- **Action.**
  1. Counted the pattern: RejectRisk does 4 lookups per row; ASP runs dispatch queries on every call and may resolve the master policy up to 3 times; Template12 runs many small aggregations and uses reflection merging.
  2. Compared timeouts: Mongo `maxTime` 600 s in many handlers against HTTP read timeouts of 8 s and 20 s.
  3. Found that the pool keys in `application.properties` are not read by anything.
  4. Wrote a measurement plan: `explain("executionStats")`, query counts with DEBUG, timers around outbound calls, JFR on the merge.
- **Result.** A ranked risk list with a measurement plan, ready for a test environment. No change was made and no improvement is claimed. [CONFIRM: any measurement you really did]

## STAR 8 - Configuration issue: Boot 4 property prefix

*Constructed from artifacts - adapt to your real experience.*

- **Situation.** After the move to Spring Boot 4.1.1, the app can start but read from `localhost:27017`, because `spring.data.mongodb.*` is ignored. Also, every POST can fail because Jackson 3 ignores the Jackson 2 annotations.
- **Task.** Make the service connect to the right database and keep the JSON conversion working. (Platform task; [CONFIRM: whether you worked on it])
- **Action.**
  1. Used the new `spring.mongodb.uri` and `spring.mongodb.database` keys.
  2. Set `spring.http.converters.preferred-json-mapper=jackson2`.
  3. Pinned Jackson 2 with `jackson-2-bom.version`, not `jackson-bom.version`.
  4. Noted dead keys (`auto-index-creation`, `option.*`) and wrong logger names to clean up later.
  5. Kept the credential out of commits and wrote it as `<redacted>` in notes.
  Commits for the upgrade are `9248b12ff`, `65f1a8711`, `c6b6803f3` (not attributed to you).
- **Result.** The service starts against the intended database and handles requests. A checklist for future upgrades: search for properties nobody reads, and check the Mongo connection log line. [CONFIRM]

## STAR 9 - Release support: getting a change through the PR gates

*Constructed from artifacts - adapt to your real experience.*

- **Situation.** PRs to `retrofit-develop-R1/R2` go through Sonar with a quality gate, a Jira-key check on commits, secret scanning and an AI review. The image build is a manual workflow with an optional image tag. The Helm repo is separate.
- **Task.** Get the change merged and built without breaking the gates. [CONFIRM: your real release task and date]
- **Action.**
  1. Used the `OPUSM-44159` key on every commit so the Jira check passes.
  2. Kept the local `application.properties` change out of the commit because the secret scanner and the credential rule apply. (Note: the gitleaks allowlist hides findings in that file, so the human check matters.)
  3. Ran the unit tests locally and updated the ones that the change breaks (RejectRisk: 7 of 15).
  4. Handed over the release note with the image tag and the known gaps. [CONFIRM: what you handed over and to whom]
  5. Kept in mind the rollback idea: redeploy the previous image tag. (inferred)
- **Result.** A change that is traceable by Jira key, free of credentials, with named open items. [CONFIRM: the real outcome, such as merged date or environment]

---------------------------------------------------------------------------------------------------

# PART E - CLAIM GUARD AND CONFIRM CHECKLIST

## E1. Safe to claim (inside scope, or shown by code)

- I analysed Template12, ASP and RejectRisk with all sub-report levels from the Jasper files. (Scope item i.)
- I can explain the Java design, the Oracle-to-Mongo patterns, the product and scheme matrix, the gates, and the response structure.
- I worked on unit tests and test cases for all three. I can explain the test style, its strengths and its gaps. (Scope item iii.)
- I worked on XSD mapping and payload alignment. I can show the RejectRisk mapping field by field. (Scope item iv.)
- I know the open issues and can state them clearly.

## E2. Do NOT claim

- "I wrote commit X" for any commit by Shinge, Karunakar, Shaw, GS, Amarnath Yadav, Neethu or others, unless you can prove it.
- That the output validates against the XSD.
- Any coverage, pass-rate, performance or defect-count number.
- That the RejectRisk fix is committed, merged or tested.
- A meaning for "ASP".
- Any production incident, ticket number, date, person or outcome that you did not live through.
- The Boot 4 upgrade as your work, unless it was.
- That the tests compare with Jasper output (they do not).
- That authentication exists.

## E3. `[CONFIRM]` checklist to fill before the interview

1. Which commits, files and handlers are yours (Template12, ASP, RejectRisk).
2. Which tests or test batches you wrote, and whether and how you used a tool to generate cases.
3. Which XSD edits and mapping rows are yours; whether you raised the XSD version gap.
4. Whether you ran the tests after the RejectRisk change, and the result.
5. Why the RejectRisk change is uncommitted and the plan for it.
6. Who asked for `mainReportName` / `subReportName`, and the renderer team's answer.
7. Any real defects you found or fixed, with Jira keys.
8. Your real environments, builds, image tags and release dates.
9. Any real support case and its outcome.
10. Any measurement or performance work you actually did.
11. Whether `cc6a1b84d` content is in your target branch.
12. Who owns the XSD and who calls the API.

---------------------------------------------------------------------------------------------------

# UNCERTAINTIES FOR THIS FILE

1. Authorship: nothing in git is under your user name. All personal claims depend on your `[CONFIRM]` answers.
2. No tests were run. Statements like "7 of 15 will fail" come from reading code.
3. XSD: only v0.1.6 is available; v0.1.7 to v0.1.10 exist only as comments in Java.
4. The deployed environment, Helm values, monitoring and rollback are not visible.
5. Real storage types in Mongo (dates, ids, sequence refs) were not checked on live data.
6. The meaning of "ASP" and whether the renderer accepts multiple `reportData` entries are unknown.
7. The STAR stories and support stories are constructed from artifacts, not from your memory.
