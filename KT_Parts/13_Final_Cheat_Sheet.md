# 13 - Final Cheat Sheet (opus-documentprintdata-service: Template12, ASP, RejectRisk)
> **AUTHORSHIP UPDATE (confirmed by the developer): git author `GS <extern.gs_nisarga@allianz.com>` is the developer. Wherever this file says commits cannot be attributed to the developer, read it as: the 11 GS commits listed in `KT_Parts/14_My_Commits_And_Ownership.md` ARE theirs; commits by other authors still must not be claimed. No GS commit touches `src/test`, so test authorship is still `[CONFIRM]`.**

How to use this file: read sections 1-12 the night before, the three one-pagers (13-15) one hour before, and the Q&A banks (16-19) as practice. Section 20 gives the spoken explanations. Section 21 lists what you MUST confirm with the team.

Tags used: (evidence) = read in repo, pom, Jasper dump or earlier KT files. (inferred) = reasoning, not proven. [CONFIRM: ...] = a personal fact only you know. I did not run Maven, the app, or any test. Nothing here is copied from secrets; connection strings are `<redacted>`.

Source files behind this sheet: `OPUS_DocumentPrintData_KT_Guide.md` and `KT_Parts/01..06`. Versions were re-checked against `pom.xml` and the Spring Boot 4.1.1 BOM file in the local Maven cache.

Rules I must respect when I speak:
- One live bean for Template12, one live bean for ASP. The "PolicyDocument2/Landscape" and "EndtDocument2/AddMemberLandscape/DeleteMemberLandscape" are logical parts of those two beans.
- XSD v0.1.6 is the only XSD found. Real Java output does NOT validate against it.
- "ASP" has no written expansion in the code or Jasper files. Do not invent one.
- The API has no authentication. An unknown report name returns HTTP 200 with status SUCCESS.
- Git does not show which commits are mine. Never say "I wrote commit X" unless I know it. [CONFIRM: which commits/PRs are mine]

---

## 1. Project Summary

- Service: `opus-documentprintdata-service` (Maven artifactId `gi-documentprintdata`, Spring app name `Documentprintdata`). A report DATA provider for Allianz OPUS (Malaysia general insurance). (evidence)
- It does NOT make PDFs. It takes a report name plus filters, reads MongoDB and a few OPUS REST APIs, and returns XML inside a JSON response. A print/Jasper engine makes the PDF. (evidence)
- Why: OPUS is being retrofitted. Old Jasper reports ran Oracle SQL. Oracle data is mirrored to MongoDB (same names as Oracle tables and views), so each SQL was rewritten as a Mongo aggregation in Java. (evidence)
- My three Personal-Lines (Plines) reports:
  - Template12 (PolicyDocument, PolicyDocument2, Landscape): a policy schedule template.
  - ASP (EndtDocument, EndtDocument2, AddMemberLandscape, DeleteMemberLandscape): endorsement documents for the ASP product (config code 070412).
  - RejectRisk: a listing of rejected risks (customer, reject type, reason, revoke info).
- My Jira scope: analysis, code, unit tests, XSD changes for all three. [CONFIRM: ticket numbers that are mine; OPUSM-41763 and OPUSM-44159 appear in the repo history but the repo does not say who owned them]
- My window: 12 May 2026 to 30 Dec 2026. Repo history I can see ends 2026-10-08. [CONFIRM: what I did after that date]

## 2. Architecture Summary

```
Caller (print orchestrator, not in repo)
  POST /gi/documentprintdata/api/opus/GenerateReport   JSON {metadata, filterCondition{reportName,...}}
  -> CustomRequestFilter (turns pBranch arrays into "a,b,c")
  -> CommonController: strip leading "/" and "reports/"; reportStrategies.get(reportName)
        found  -> ReportStrategy bean .execute()      (new style, Plines)
        not    -> legacy switch(reportName) (~50 cases; Motor/Fire/Bond...)
  -> Service (orchestrates + business gates)
        -> Handlers (@Component) -> MongoTemplate aggregation -> MongoDB (read only)
        -> Clients (PlineUtilsClient etc.) -> OPUS REST APIs (/ext/utils, /ext/policy, /ext/customer)
  -> DTO -> Pline* payload classes (XSD shape) -> Transactions
  -> Jackson XmlMapper -> XML string -> JSON {status, data, timestamp}  (HTTP 200)
```
- Layers: Controller, Service (`@Service("<report path>")` implements `ReportStrategy`), Handler (one per Jasper sub-report), DTO, request payload (`Pline*`, mirrors XSD), clients, shared utils (`PlineReportNames`, `PlineReportBlocks`, `DateUtils`). (evidence)
- Pattern: Strategy by bean name. Spring injects `Map<String, ReportStrategy>`. (evidence)
- Rule "1 sub-report = 1 reportData". Each reportData has `prntDetails > transDetList > transDet`. (evidence)
- Size: plines has about 422 handlers, 377 DTOs, 302 payload classes, 140 services (113 are `ReportStrategy`). Big classes: `PlinePolicyDocumentBaseService` 4,083 lines, `Template12PolicyDocumentBaseService` 3,732, `Template12PolicyDocumentService` 2,235, `AspEndtDocumentService` 1,701. (evidence)
- No frontend, no messaging, no batch jobs in the app, no database writes found. (evidence)
- Security: `SecurityConfig` ends with `anyRequest().permitAll()`. No authentication. Stateless, CSRF off. (evidence)

## 3. Technology Summary

- Java 17, Spring Boot 4.1.1 (servlet MVC; WebFlux also on classpath, used by one WebClient client).
- Spring Data MongoDB `MongoTemplate` with raw `Document` pipeline stages (no repositories, no `@Document` entities for my modules).
- Jackson 2 (`com.fasterxml.jackson`) for JSON and `jackson-dataformat-xml` `XmlMapper` for XML. Jackson 3 is on the classpath but forced off for HTTP with `spring.http.converters.preferred-json-mapper=jackson2`.
- Apache HttpClient 5 with a custom PKCS12 truststore for outbound calls; `RestTemplate` beans with bounded timeouts.
- springdoc-openapi (Swagger UI), Spring Boot Actuator (health, info only), Lombok.
- Tests: JUnit 5 style (Jupiter), Mockito (mock-maker-inline), AssertJ; JaCoCo 0.8.12 to Sonar.
- CI: GitHub Actions (Sonar quality gate on PRs, CodeQL, secret scan with gitleaks, Jira key check, changelog bot, Claude PR review). Deploy build is manual (`workflow_dispatch`). Helm charts are in another repo.
- Not used: JPA/Hibernate, Gradle, Oracle driver, Kafka, Jenkins, Docker files in this repo. (evidence)

## 4. Version Summary (verified against pom.xml and the local Boot 4.1.1 BOM)

Source column: EXPLICIT = written in this `pom.xml`. BOM = managed by `spring-boot-dependencies 4.1.1`. "BOM value" shows what Boot would give without the pin.

| Item | Version | Source | Note |
|---|---|---|---|
| Java | 17 | EXPLICIT (`java.version`) | |
| Spring Boot parent | 4.1.1 | EXPLICIT (parent) | Moved up from 3.5.14 on 2026-09-10 (commit 65f1a8711) for CVE-2026-59270 (per pom/guide) |
| Spring Framework | 7.0.9 | BOM | verified in BOM file |
| Spring Security | 7.1.1 | BOM | verified in BOM file |
| Spring Data BOM / Spring Data MongoDB | 2026.0.1 / 5.1.1 | BOM | 2026.0.1 verified in BOM; 5.1.1 is quoted in a pom comment |
| MongoDB Java driver | 5.8.1 | BOM | verified |
| Tomcat embedded | 11.0.25 | EXPLICIT (`tomcat.version`) | BOM value 11.0.24; pin for 3 Wiz critical CVEs (pom comment) |
| Jackson 2 | 2.21.5 | EXPLICIT (`jackson-2-bom.version`) | BOM value also 2.21.5 in the local BOM file; must use the `jackson-2-bom` property name, not `jackson-bom` |
| Jackson 3 (`tools.jackson`) | 3.1.5 | BOM (`jackson-bom.version`) | not used for HTTP here |
| springdoc-openapi | 3.1.1 | EXPLICIT | 2.8.15 broke on Boot 4 (guide) |
| HttpClient 5 | 5.6.4 | BOM (pin removed) | pom comment says old 5.6.3 pin was a downgrade |
| HttpCore 5 | 5.4.3 | EXPLICIT | |
| Lombok | 1.18.46 | BOM | |
| commons-lang3 | 3.18.0 | EXPLICIT | BOM value 3.20.0 (pin is BELOW the BOM) |
| logback | 1.5.34 | EXPLICIT | BOM value 1.5.38 (pin is BELOW the BOM) |
| log4j2 | 2.25.5 | EXPLICIT | BOM value also 2.25.5 |
| Micrometer | 1.17.1 | BOM | old 1.15.12 pin removed because it broke context startup (pom comment) |
| Netty | 4.2.17.Final | BOM | |
| org.json | 20231013 | EXPLICIT | old; used by `CustomRequestFilter` |
| junit-jupiter (test) | 5.9.0 declared | EXPLICIT | BOM value 6.0.3. Which one runs is NOT verified (needs `mvn dependency:tree`) |
| Mockito | 5.23.0 | BOM | |
| JaCoCo plugin | 0.8.12 | EXPLICIT | |
| rewrite-maven-plugin | 6.46.1 | EXPLICIT | no active recipes (guide) |
| sonar-maven-plugin | 5.0.0.4389 | EXPLICIT in pom text | guide says the plugin entry is commented out; Sonar runs from the workflow |
| Maven / wrapper | 3.9.11 / 3.3.4 | `.mvn/wrapper` (from KT part 06, not re-read today) | |
| MongoDB server | unknown | not in repo | `$setWindowFields` needs 5.0+ (inferred) |
| Project artifact version | 0.0.1-SNAPSHOT | EXPLICIT | image tag, not pom version, identifies a release (inferred) |

Not verified: transitive versions. Nobody ran `./mvnw dependency:tree`. Stale docs: `ARCHITECTURE.md` says Boot 3.4.11 and `README.md` says 3.5.14; the pom is the truth.

## 5. API Summary

- Inbound (evidence): `POST /gi/documentprintdata/api/opus/GenerateReport` is the only real report endpoint (all LOBs). `GET .../GenerateReport/getJSON` returns an empty DTO. Actuator `health` and `info` are open. Swagger UI `/swagger-ui/index.html` and `/v3/api-docs` are open. `DocumentController` and `ExtendedController` are empty.
- Request: `{ "metadata": {...}, "filterCondition": {...string values...}, "mergeStaticPdfFiles": [...] }`. `filterCondition` is a `Map<String,String>`; `reportName` inside it picks the service. Only `pBranch`/`pBranchName` arrays are flattened by `CustomRequestFilter`. Missing `metadata` probably gives a NullPointerException and HTTP 500 (inferred).
- Response: `{ "status": "SUCCESS", "data": "<XML string>", "timestamp": "..." }`. The field is `data` in code (README says `xmlData`; check a live response). Always HTTP 200 for success and for "no data".
- Errors: `NoDataFoundException` is mapped to 404 but never thrown (dead path). `IllegalArgumentException` gives 400. Any other exception gives 500 with a generic message and the handler does not log the cause. Unknown report name gives 200 SUCCESS with an empty default transaction (as stated in the guide; the legacy switch default was not fully traced).
- My three bean names: `AGIC/UW/Plines/RejectRisk/RejectRisk`, `AGIC/UW/Plines/ASPEndtSchedule/EndtDocument`, `AGIC/UW/Plines/PolicySchedule/Template12/PolicyDocument`.
- Merged-away names with NO bean: `.../Template12/PolicyDocument2`, `.../Template12/Landscape`, `.../ASPEndtSchedule/EndtDocument2`, `.../AddMemberLandscape`, `.../DeletionMemberLandscape`.
- Outbound (evidence): `invoice-print-indicator`, `invoice-print-endt-indicator`, `stax-inv`, `comp-brn-number`, `fn-rppm-get-signature`, `sst-rollout-date`, `einvoice/qr-indicator`, `get-policy-ref` (two variants: pg-uwge-utils and pme-api). Failures are caught and defaulted, so the report still returns.
- Auth: none. SSRF protection exists in the clients (`sanitize`, `assertTrustedHost`, commit 40d3fc1f7).

## 6. Database Summary

- MongoDB (Atlas style multi-host URI, `ssl=true`). Database name in the working copy: `OPUSAGICPRODR1`. URI is `<redacted>` and sits only in an uncommitted change to `application.properties`; never commit it.
- Read only: handlers use `MongoTemplate.aggregate(...)` on collection-name strings. No inserts, updates or deletes found. No indexes defined in the repo (`auto-index-creation` key is in the wrong place and does nothing).
- Collection names equal Oracle table/view names (`UWGE_*`, `UWPL_*`, `*_VI_*`, `CMGE_CODE`, `SAPM_SYS_CONSTANTS`, `CPGE_*`, `DMAG_*`). Whether `*_VI_*` are real Mongo views or copied collections is unknown.
- Prefix `WIP_` = work in progress copy (used for checklist copy / not yet issued). No prefix = posted/live. `SB_` and `OCP_` are not explained in the code.
- Key types: `CONTRACT_ID` and `VERSION_NO` are numbers. A string match returns nothing silently. Handlers convert with `Long.parseLong`/`safeNum`.
- Oracle to Mongo patterns: scalar sub-select to `$lookup` + `$limit 1` + `$arrayElemAt`; UNION to `$unionWith` or two lookups with `$ifNull`; DISTINCT to `$group`; ROWNUM to `$setWindowFields` with `$documentNumber`; Oracle function to REST call; ORA-01427 emulated by throwing `IllegalStateException`.
- Perf notes: `allowDiskUse` and `maxTime` 600 s in many Template12/ASP handlers, NOT in `RejectRiskHandler`. MongoTemplate DEBUG logging prints pipelines (PII risk). Index suggestions in KT part 06 are recommendations only.

## 7. Configuration Summary

- Files: `application.properties` (251 lines), `application-env.properties` (SSL keys only, loaded with `@PropertySource`, not a Boot profile), `mytruststore.p12` (committed). `spring.profiles.active=dev` activates nothing because no `application-dev*.properties` exists.
- Server: port 8080, context path `/gi/documentprintdata`.
- Mongo under Boot 4: use `spring.mongodb.uri` and `spring.mongodb.database`. Old `spring.data.mongodb.*` keys are silently ignored and the app goes to `localhost:27017`.
- `spring.http.converters.preferred-json-mapper=jackson2` is needed because DTOs use `com.fasterxml` annotations.
- HTTP client (`app.http.*`): connect 5 s, read and socket 20 s, connection-request 3 s, pool 100 total / 20 per route, fast-lookup read 8 s. `PlineUtilsClient` uses the fast-lookup bean; `PlinePolicyRefClient`, `PlineSSTRolloutDateClient`, `PlinePgRpgeUtilsClient` use the primary RestTemplate (20 s).
- Base URLs: `API_BASE-URL` (about 45 `@Value` uses) and a separate `plines.api.base-url`. Both point to a dev2 host in the committed file; to move environment you must change both plus Mongo keys. Several older keys point to dev1 and are unused.
- Dead or ineffective keys: `auto-index-creation`, `option.*` pool keys, `spring.messages.basename`, `management.endpoint.loggers.enabled`, several logging package names that do not exist.
- Logging: Mongo DEBUG on, `System.out` request body printing in `CustomRequestFilter`. No logback XML.
- Truststore password is plaintext in `application-env.properties` (`<redacted>`). `gitleaks.toml` allowlists both properties files (OPUSM-50404, review-by 2026-10-23).
- Deployment overrides (env vars, Helm values) are not in this repo. [CONFIRM: how pods get Mongo URI and base URLs]

## 8. XML Summary

- Output is a no-namespace XML tree: `transactions > transaction > (reportParameters, mailingDetails, templateDetails, transactionDetails > headerTransaction > reportTransaction > reportData[] > (dataClass, mainReportName, subReportName, endorsementDetails, prntDetails > transDetList > transDet))`. (evidence)
- Written by a plain Jackson `XmlMapper` in `CommonController`, then put inside the JSON `data` string.
- `PlineTransDet` is a very wide "bag of leaves" (about 639 scalar fields plus lists) because the XSD flattened many Jasper fields into one type.
- Dates are written as `dd/MM/yyyy`. Null fields are written as empty elements (`@JsonInclude(ALWAYS)`).
- `PlineReportNames.tag()` fills `mainReportName` on every reportData; services or `PlineReportBlocks` set `subReportName`. These two elements are not in XSD v0.1.6.
- Template12 puts band rows into one reportData with a reflective merge (`mergeSingleTransDet`) and 16 landscape bands each in their own reportData. ASP gives a master row first, then one reportData per sub-report. RejectRisk gives one reportData per DB row plus one trailing header block.
- Template/mailing details are added by the controller from `metadata`.

## 9. XSD Summary

- Files: `Pline.xsd` (1,400 lines) and `Common.xsd` (748 lines), header version v0.1.6 (2026-07-28). Repo test copy (`src/test/resources/xsd/plines/`) is identical to the reference copy (line endings aside). No newer XSD exists on this machine. (evidence)
- No target namespace, `elementFormDefault="qualified"`, `xs:include` of Common, only `xs:sequence` (order matters), 1,062 of 1,065 Pline elements are `minOccurs=0`, no attributes, no enumerations.
- Changes by version: v0.1.3 moved `prntDetails` inside `reportData` (breaking). v0.1.4 moved `permanentDisablement`, removed `showCoverNote`/`showTaxInvoice` in favour of `pshowCoverNote`/`pshowTaxInvoice`, added 12 parameters (breaking for moved/removed). v0.1.5 additive. v0.1.6 added `pAgentChannel` and changed `pBeforeEffDate` to `xs:dateTime`.
- Java comments mention v0.1.7 to v0.1.10 but those files were not found. Java emits things v0.1.6 does not know: `mainReportName`, `subReportName`, `pCreatedDateFrom`, `pCreatedDateTo`, `pReportTitle`, and 4 extra transDet fields (`leasingBranch`, `policystatus`, `printingDate`, `state`).
- Real output does NOT validate against v0.1.6: dates are `dd/MM/yyyy` not `xs:dateTime`; empty elements for decimal/date fields are invalid; sibling order differs (for example `reportParameters` is last in Java, first in XSD; `address` is last in Java agent/branch/comp types, first in XSD). An older saved RejectRisk response gave hundreds of errors (929 and 1766 in two samples). A hand-ordered XML gave 0 errors. (evidence from KT part 04; ran by a JDK validator on a scratch copy)
- `PlineXsdValidationTest` only checks a hand-written minimal shell. Its four "service generates valid XML" tests validate the same shell and never call a service. No ASP or RejectRisk XSD test exists.
- Mapping workbooks (requirement side): `Pline_MASTER_MAPPING_SHEET_v0.1 (final1) 1 5.xlsx` and `RejectRisk_XSD_Mapping_sheet.xlsx`. Only the RejectRisk sheet was read fully.
- Honest statement to use: "We aligned the payload classes and structure to the XSD. Full validation of real service output was still open."

## 10. Deployment Summary

- CI/CD (evidence): `cicd-workflow.yml` runs only by `workflow_dispatch` (optional `imageTag`), calls a shared action `java-microservice` (Java 17, Sonar, Wiz) in `opus-github-cicd-workflows`. Steps inside the shared action are not visible.
- PR checks: Sonar `verify` with quality gate wait on PRs to `retrofit-develop-R1` and `R2`; secret scan on every PR; nightly full-history secret scan; Jira key check (`OPUSM-`); Claude PR review; CodeQL weekly plus some pushes (its branch filters miss the R1/R2 branches, inferred).
- Branches: default `retrofit-develop-R1`; long-lived integration `retrofit-develop-R2`; feature naming `OPUSM-<n>/retrofit-develop-R2-pline-<topic>`.
- Not in repo: Helm chart (separate repo `opus-helm-gi-documentprintdata-service`), env values, rollout, rollback, monitoring alerts. Rollback is probably "redeploy previous image tag via Helm" (inferred). The service owns no DB state, so a code rollback is stateless.
- Health: actuator `/gi/documentprintdata/actuator/health` (probes presumably use it, inferred).
- [CONFIRM: how I deployed to dev/SIT/UAT, who triggers the workflow, how rollback is done]

## 11. Production Support Summary

| Symptom | Likely cause | Look at |
|---|---|---|
| Template12 returns 500 | empty or non-numeric `pContractId` (NumberFormatException in landscape dispatch); WIP vs live conflict (`IllegalStateException`, mimics ORA-01427) | `Template12LandscapeDispatchHandler`, `fetchTemplate12PolicyDocumentMain` |
| Page or sub-report missing | a gate is closed: show flags, `checklistCopy`, product code, `ASP_CUTOFF_IND`, e-invoice flags; or an external API failed and defaulted to N | service `execute()` gates; `PlineUtilsClient` |
| Tax flags null, tax invoice never prints | `CONTRACT_ID` string vs number mismatch | `AspEndtDocument2MainHandler.buildCommonPipeline` (fix is on another branch, check merge status) |
| Empty XML with SUCCESS | unknown report name or old merged-away name | `CommonController` routing; use the merged bean name |
| RejectRisk missing rows on end date / no rows | HEAD uses `$lte` on midnight and null bounds | `RejectRiskHandler` date filter (fix not committed on my branch) |
| App talks to localhost:27017 | old `spring.data.mongodb.*` keys | `application.properties` |
| JSON conversion errors on start or POST | Jackson 3 default | `preferred-json-mapper=jackson2` |
| Slow or hanging | external API latency; Mongo `maxTime` is 600 s | `app.http.*`, fast-lookup template |
| Cause of a 500 not in logs | `GlobalExceptionHandler` does not log | add logging; look at thrower |

First triage steps (inferred, standard): get the exact request body, call again in dev, read the app log for the report name, check which gate or handler returned empty, check external API reachability, check Mongo for the contract in WIP and live collections.

## 12. (spare) Numbers worth remembering

- Test files in scope (name filter Template12/Asp/RejectRisk): 91 files, 865 `@Test` annotations (annotation counts, not run results). `RejectRiskHandlerTest` 15, `RejectRiskServiceTest` 4, `AspEndtDocumentServiceTest` 28, `Template12PolicyDocumentServiceTest` 34, `Template12PolicyDocumentBaseServiceTest` 135.
- About 39 percent of the in-scope tests (337 of 865) are DTO/payload getter-setter tests.
- Test base was built 2026-09-25 to 2026-09-30 under OPUSM-44159 by several authors. [CONFIRM: which tests I wrote]
- Commits on plines since 2026-05-12: 134 (105 non-merge). Latest 2026-10-08.
- No coverage numbers exist in the repo (no `target/site/jacoco`).

---

## 13. PLine Template12 - one page

**Purpose.** Policy schedule document (`PolicySchedule/Template12/PolicyDocument`) used by several Personal-Lines products. One call returns policy pages, accounting pages (debit note, tax invoices) and landscape member/benefit tables as XSD-shaped data.

**Bean.** `@Service("AGIC/UW/Plines/PolicySchedule/Template12/PolicyDocument")`, class `Template12PolicyDocumentService` (line 103). It is the ONE live bean. Commit c0c6d449a (2026-08-03, "Added Template12 dynamic merging") merged PolicyDocument2 and Landscape into it (`executeMergedPolicyDocument2`, `executeMergedLandscape`). The old classes `Template12PolicyDocument2Service` and `Template12LandscapeService` keep tests but have `@Service` commented out; re-enabling would render every band twice.

**Inputs (filterCondition defaults).** `pContractId ""`, `pPolicyNo ""`, `versionNo "1"`, `companyParamType`, `checklistCopy "N"`, `showCoverNote "N"`, `showComplaint "Y"`, `showScheduleStd "Y"`, `showDebitNote "N"`, `showTaxInvoice "N"`, `showPaymentInfo "N"`, `printSeries`, `copyType`, `pCertInd`, `ShowOutPrintCovLetter "N"` (capital S), `pType`, `pFirstPageNumber`, `pMstPolicyNo`, `pContractIdList`, `pCertNoFrom/To`, `pTaxInvNo`, `pGstTaxInvoiceNo`. Some (`pMstPolicyNo`, `pContractIdList`, `ShowOutPrintCovLetter`) are read but not written to `reportParameters` (no home in v0.1.6). Merged PD2 hardcodes `certNo=""`.

**Flow.**
1. Controller finds the bean by name; service reads filters and defaults.
2. Create a shared header reportData.
3. Primary fetch `fetchTemplate12PolicyDocumentMain` gives product, scheme, `ASP_CUTOFF_IND`, issue date, dependants. It throws if WIP and live data disagree (copies ORA-01427).
4. Set product flags: 070504, 070412 (ASP), 070406, 070414, 100908 and schemes ASTRO, BMW, POSCOVER, AZP, POSMESRA, FWPA, Elite.
5. Part A (PolicyDocument): footer or outsourced cover letter; then by product the CIPrint variant with nested bands (risk details, coverage, installments, note, schedule of benefits, perils, PIDM, ASP portrait pages, complaint redress). BMW prints one row per certificate.
6. Part B (PolicyDocument2): second primary fetch `PlineT12PolicyDocumentHandler` (posted first, WIP fallback, calls REST for e-invoice and service-tax flags); nominations, installments, domestic/overseas ATC, debit note, GST and service-tax invoices, QR section, transitional invoices, PDPA/PDS.
7. Part C (Landscape): `Template12LandscapeDispatchHandler` decides; if the scheme is excluded in `CMGE_CODEREL_DET` (`CI_LANDSCAPE_PRT`) nothing prints; else up to 16 bands, each its own reportData.
8. Sub-band failures are caught and logged (partial report). Primary fetch and landscape dispatch are not caught (request fails).
9. `buildTransactions` assembles `Transactions > Transaction > ... > reportData[]`; report-level values chosen PD, then Landscape, then PD2; date strings reformatted by reflection to `dd/MM/yyyy`.
10. `mainReportName = Template12PolicyDocument`; the Template12 service does not set `subReportName` per section (guide statement).

**Collections.** `WIP_/SB_UWGE_POLICY_CONTRACTS`, `WIP_UWPL_POLICY_BASES`/`UWPL_POLICY_BASES`, `UWPL_RISK_PERSON` (+WIP), `UWGE_VI_POL_REPORT_PARAM`, `SAPM_SYS_CONSTANTS` (`ASP_CUTOFF_DATE`), `UWPL_VI_POLICY_SCHEDULE_COVER`, `UWGE_VI_POLICY_INSTALLMENT`, `UWPL_VI_POLICY_SCHEDULE_RISK`, `UWPL_VI_POLICY_SCHEDULE_TPL4/_BEN/_PREM/_NOM/_DIAG/_TPL20`, `UWPL_VI_PREMIUM_DEBIT`, `UWPL_VI_TAX_INVOICE`, `CPGE_VI_PARTNERS_ALL`, `CMGE_CODEREL_DET`, `CMGE_CODE`, `SAPM_COMPANY`.

**APIs.** e-invoice print indicator (`invoice-print-indicator`), `stax-inv`, `comp-brn-number`, `fn-rppm-get-signature`, `sst-rollout-date`, co-inward indicator. Failures are defaulted.

**XSD notes.** No Template12-specific elements; it uses the generic PolicyDocument family. Dates `dd/MM/yyyy` vs `xs:dateTime`. `mainReportName` is not in v0.1.6. Commits for XSD work: d4575fa5b (2026-08-03), 6961abbf9 (2026-08-08), edbef120c, e74bc0d5c. [CONFIRM: which of these are mine]

**Known gaps.** Page-number conditions (blank pages for duplex, PrintSeriesFooter call 2) cannot be done in Java. Java prints two PDS bands Jasper marks `false &&`. Domestic/Overseas ATC page prints for every product. Three divergent product/scheme fetches. `Long.parseLong("")` crash. Landscape promo amount lacks version filters. Many bands are dispatch-only stubs (ComplaintRedress, ScheduleOfBenefits_ASP/AMC, ImportantNotes, SSTNote...). Handler `PlineT12ParticularOfInsuredPosHandler` is never called. QR section uses the request `pTaxInvNo`.

**5 facts to remember.**
1. One bean does three Jasper reports; the old classes are dead reference code.
2. Two primary fetches on purpose: PD fails on conflict, PD2 prefers posted data and continues.
3. Template12 has its own 3,732-line base class so it cannot break the other ~24 templates.
4. Landscape is switched off by `CMGE_CODEREL_DET` (`GPA_SCHEME`, `CI_LANDSCAPE_PRT`).
5. `mergeSingleTransDet` merges non-null fields onto every existing transDet (reflection); last writer wins.

---

## 14. ASP - one page

**Purpose.** Endorsement documents for the ASP product (config code 070412): endorsement cover, schedule, debit/credit note, service-tax invoice, and landscape pages for added members, deleted members, sponsor and nominee. ASP has no written expansion anywhere. Internal Jasper names are `ASPlusSTI` and `ASPlusLLB`; benefit codes are `ASP-A1..D5`; the benefits look like personal accident for groups (inferred). Say: "I did not find a written expansion." [CONFIRM: the real expansion from the business]

**Bean.** `@Service("AGIC/UW/Plines/ASPEndtSchedule/EndtDocument")`, class `AspEndtDocumentService` (line 165), the ONE live bean. It also does what was `EndtDocument2`, `AddMemberLandscape`, `DeletionMemberLandscape`. `AspEndtDocument2Service` (`@Service` commented) and `AspEndtDocumentLandscapeService` (both `@Bean` methods commented) remain as reference. Other ASP pieces live inside `AutoRenewalDocumentService` and `Template12PolicyDocumentService`.

**Inputs.** `pEndtNo, pRefNo, pContractId, companyParamType, versionNo, checklistCopy, printSeries, copyType, pUserID, pPolicyNo` default `""`; `pFirstPageNumber "1"`; `showCoverNote/showEndtSchedule/showComplaint/showDebitNote/showTaxInvoice` default `"N"`. `showComplaint` is only echoed (ComplaintRedress is decided by product 070412). `pPolicyNo` unused. `pSchemeName` is not read.

**Flow.**
1. `AspEndtDocumentMainHandler` gives product code, labels, issue date, `ENDT_CODE`; `AspEndtDocument2MainHandler` gives tax flags, e-invoice flag, premium diff, tax invoice.
2. Always: print-series footer and blank page; master row `reportData[0]` with `mainReportName=ASPEndtScheduleEndtDocument`, `subReportName=PrintSeriesFooter`.
3. `showCoverNote=Y`: `AspEndtCoverHandler` and company name with GST reg no.
4. `showEndtSchedule=Y`: `AspEndtScheduleHandler` (840 lines) and signature.
5. `ENDT_CODE` 72 or 168: schedule of benefits, LLB, STI handlers are stubs and emit nothing.
6. Product 070412: ComplaintRedress is called but the result is discarded.
7. Debit note: `showDebitNote=Y` and premium diff not zero; `D` or `C` + `ENDT_NO` by sign of `PREM_DUE`; master customer details if `premByMstInd=Y`.
8. Service-tax invoice: `showTaxInvoice=Y` and `showStaxInvInd` and `checklistCopy != Y` and `showEinvInd`; TIN masked except last 4; EndtMT rows when tax count above 1; QR section when consent, status VALID and QR indicator Y. Transitional invoices if `showTranInvInd`.
9. Landscape dispatch runs on EVERY call and returns counts (addition, deletion, nominee, sponsor); then insured, premium, sponsor, nomination and delete-member sections are added, each with its own `subReportName`.
10. Wrap into Transactions and tag names; external failures are swallowed with defaults.

**Collections.** `UWMT_VI_ENDT_TAX_INV_ALT_UPV` (anchor of main handler), `UWGE_VI_ENDT_REPORT_PARAM`, `WIP_/SB_UWGE_POLICY_CONTRACTS`, `UWGE_/WIP_UWGE_POLICY_VERSIONS`, `UWPL_VI_ENDT_SCHEDULE_SUMMARY`, `UWGE_VI_ENDT_DEBIT`, `UWPL_VI_ENDT_TAX_INV`, `UWPL_VI_TAX_INVOICE`, `UWGE_POLICY_BASES_STAX`, `UWGE_COMPLAINT_REDRESS`, landscape: `UWPL_VI_ENDT_ADD_DEL_RISK`, `UWPL_VI_ENDT_SCHEDULE_TPL4/_NOM/_SPONSOR`, `UWGE_RISK`, `SAPM_SYS_CONSTANTS` (`ASP_CUTOFF_DATE`, `TPA_CUTOFF_IND`, ...). The list may not be complete.

**APIs.** `invoice-print-endt-indicator`, `get-policy-ref` (pme-api variant), `sst-rollout-date`, `comp-brn-number`, `einvoice/qr-indicator`. `safeCall` returns null; QR defaults to N.

**XSD notes.** `mainReportName`/`subReportName` (not in v0.1.6). Six delete-member fields are fetched but have no XSD leaf and are not output: `subcoverPrem, endtDoubleIndemnity, endtImaFee, endtStaxAmt, endtLlbPrem, endtStiPrem`. `contractId` is decimal in XSD, `mstContractId` integer. No ASP XSD test.

**Known gaps.** Stubs for ScheduleOfBenefits_ASP, ASPLLB, ASPSTI. Landscape dispatch always runs (5-6 extra queries). Main handler anchors on `UWMT_VI_ENDT_TAX_INV_ALT_UPV`; if no row, the DTO is all null and gates close (risk, not a confirmed bug; fix `cc6a1b84d` is on another branch). INST_IND and SHOW_GST_LABEL cut-off constants not computed. ORA-01427 emulation only in the service-tax handler. A near-copy of `AspServiceTaxInvoiceHandler` sits in the TEST tree and may shadow the real class (inferred). 1,701-line service. No test for two ASP portrait handlers.

**5 facts to remember.**
1. One bean replaces four Jasper reports; each Jasper print condition is kept as a gate.
2. The CONTRACT_ID string-vs-number bug silently hid tax sub-reports; fix is `safeNum` and an ASP-only override (commits 578778080, ba9e29586, OPUSM-41763; on other branches, merge status unknown).
3. Service-tax invoice needs four conditions, and `checklistCopy=Y` closes it.
4. Landscape sections are chosen by dispatch counts, not by request flags.
5. ASP product config code is 070412; "ASP" is never expanded in code.

---

## 15. RejectRisk - one page

**Purpose.** A listing of rejected (or blacklisted) risks: customer name and IDs, reject type, reason, remarks, created by/date, revoke flag/duration/by/date, updated by/date. Filtered by creation date range and optional filters.

**Bean.** `@Service("AGIC/UW/Plines/RejectRisk/RejectRisk")`, class `RejectRiskService` implements `ReportStrategy`. Handler: `RejectRiskHandler`. Workmen has a different `WorkmenRejectRiskReportHandler`; do not confuse.

**Inputs (all optional).** `pCreatedDateFrom`, `pCreatedDateTo`, `pCustomerName` (exact, case sensitive), `pIdNo` (matches ID_VALUE1 or ID_VALUE2), `pRejType`, `pReasonCode`, `pReportTitle`, `reportName`. Date formats accepted by `plines.utils.DateUtils` include `yyyy-MM-dd`, `dd-MM-yyyy`, `dd/MM/yyyy`; a datetime without offset is not accepted and gives 500.

**Flow.**
1. Controller routes by bean name to `RejectRiskService`.
2. Service reads filters and calls `RejectRiskHandler.fetchRejectRisks`.
3. Pipeline: `$match` on `CREATED_DATE`; optional `$match` on `REJ_TYPE`, `REASON_CODE`.
4. `$lookup CPGE_VI_PARTNERS_ALL` on `CP_PART_ID`/`CP_PART_VERSION` = `PART_ID`/`VERSION`, `$limit 1` (left join).
5. Optional `$match` on `CUST.NAME_EXT` and on `ID_VALUE1`/`ID_VALUE2` (after the lookup).
6. Three `CMGE_CODE` lookups: `PLREJ_RISK_TYPE`, `PLREJ_REASON`, `PLREJ_REVOKE_DUR`.
7. `$project` to DTO names; `$setWindowFields` + `$documentNumber` sorted by `customerName` gives `rowNumber` (replaces Oracle ROWNUM); `$sort` by `customerName`.
8. Service looks up `dataClass` from `RPGE_RPT_TEMPLATE` via `PlineReportHeaderHandler` (title + report name).
9. One `reportData` per row: `prntDetails > transDetList > transDet` (rownumber, customerName, idType1/idNo1, idType2/idNo2, riskList.risk, reasoncode, createdBy/Date, revokeInd, revokedDuration, revokeBy/Date, updatedby/date) and `companyType.companyDesc.remarks`.
10. If rows exist, add a trailing `ReportHeader` block (`reportHeader("RejectRisk", dataClass)`); `reportParameters` echo title, report name, from/to dates. Zero rows gives no header block and an empty list.

**Collections.** `CMUW_PL_REFER_RISK` (driver), `CPGE_VI_PARTNERS_ALL`, `CMGE_CODE`, `RPGE_RPT_TEMPLATE`.

**APIs.** None.

**XSD notes.** `REMARKS` goes to `transDetList/companyType/companyDesc/remarks` (a sibling of transDet, which is why each row gets its own transDetList). Irregular XSD names: `reasoncode`, `updatedby`, `updateddate`, `idType1`, `rownumber`. `pCreatedDateFrom/To` and `pReportTitle` and `mainReportName` are not in v0.1.6. Date leaves are `xs:dateTime`; Java writes `dd/MM/yyyy`. Mapping sheet: `RejectRisk_XSD_Mapping_sheet.xlsx`. No XSD test for RejectRisk.

**Known gaps.**
- HEAD has two bugs: `$lte` on midnight cuts off the last day (from=to returns only exact-midnight rows) and null date bounds match nothing.
- My uncommitted working-tree fix: `dateTo + 1 day` with `$lt`, only supplied bounds added, no date stage if none given, removed the fake `ROW_NUMBER=1` stage, and service uses the Plines `DateUtils`. It is the same as commit `cc6a1b84d` (OPUSM-41763, 2026-10-05) on another branch. [CONFIRM: merge status]
- That fix likely breaks 7 of 15 `RejectRiskHandlerTest` tests (they assert `$lte`, null bounds and stage indexes). Not run.
- "No dates" now returns ALL rows (Oracle returned none): decide with business.
- Possible time-zone shift (dates built as UTC midnight). Null customer names sort first in Mongo (Oracle puts them last). No pagination; sort without `allowDiskUse`; customer filters run after the lookup.
- Disclaimer in service: renderer treatment of `reportData[]` as separate print units was not confirmed.
- `application.properties` has an uncommitted Mongo URI with credentials: do not commit.

**5 facts to remember.**
1. Row number = `$setWindowFields` + `$documentNumber`, not a counter in Java.
2. `$lt (dateTo + 1 day)` keeps an index usable and includes the whole last day.
3. A null bound in Mongo matches only null dates, so it returned nothing.
4. One reportData per row because `companyDesc.remarks` is outside transDet in the XSD.
5. The fix equals `cc6a1b84d` on another branch, and the existing tests were not updated.

---

## 16. Top 50 Most Likely Questions

Q1. Tell me about your project.
A: It is a Spring Boot report data service for Allianz OPUS. It receives a report name and filters, reads MongoDB and a few OPUS APIs, and returns XML inside JSON for the print engine. I owned three Personal-Lines reports: Template12, ASP and RejectRisk.

Q2. What was your role?
A: I did analysis, code, unit tests and XSD changes for Template12, ASP and RejectRisk. [CONFIRM: exact tickets, reviewers, how many people on the team]

Q3. Does the service produce the PDF?
A: No. It produces the data as XML. A downstream print/Jasper engine renders the PDF. The service has no PDF code.

Q4. Why was the service built?
A: OPUS is being retrofitted. Old reports were Jasper files with Oracle SQL. Data is now mirrored in MongoDB, so the SQL had to become Java aggregation pipelines in a new service.

Q5. What is the single endpoint?
A: `POST /gi/documentprintdata/api/opus/GenerateReport`. The report is chosen by `filterCondition.reportName`. It serves all lines of business.

Q6. How does it pick the right report?
A: Each new-style service is a Spring bean named like the report path. `CommonController` looks it up in a `Map<String, ReportStrategy>`. If no bean matches, it falls to an old `switch`.

Q7. What is Template12?
A: A policy schedule template for several Personal-Lines products. One bean returns policy pages, accounting pages and landscape tables. Product code and scheme decide which pages are included.

Q8. What is ASP?
A: The endorsement documents for product config code 070412. Cover, schedule, debit/credit note, service-tax invoice, and landscape pages for added or deleted members. The code never expands "ASP", so I do not claim an expansion. [CONFIRM: real expansion]

Q9. What is RejectRisk?
A: A listing report of rejected risks with customer, reason, remarks and revoke details for a date range. One service, one handler, mainly one collection plus lookups.

Q10. Why is there one bean for three Template12 reports?
A: The print engine needs one response, and having three beans would render pages twice. Commit c0c6d449a merged PolicyDocument2 and Landscape into the PolicyDocument bean. The old classes are kept unregistered for reference.

Q11. Same question for ASP.
A: EndtDocument2, AddMemberLandscape and DeleteMemberLandscape are folded into the EndtDocument bean. Each original Jasper print condition is kept as a gate inside the service. Old names no longer have a bean.

Q12. How did you convert SQL to MongoDB?
A: By patterns. Scalar sub-select became `$lookup` + `$limit 1`. UNION became `$unionWith` or two lookups with `$ifNull`. DISTINCT became `$group`. ROWNUM became `$setWindowFields`. Oracle functions became REST calls.

Q13. What is WIP data?
A: Work-in-progress data, not yet issued. Collections have a `WIP_` copy next to the live one. The code checks both, and `checklistCopy=Y` usually means preview from WIP.

Q14. What did you do for XSD?
A: I aligned the payload classes to Pline.xsd (names, types, structure) and added the report and sub-report names. Honest point: real output was not fully validated, and the XSD copy I have is v0.1.6. [CONFIRM: exactly which XSD changes were mine]

Q15. Is your output valid against the XSD?
A: Not fully. Dates are `dd/MM/yyyy` while the XSD needs `xs:dateTime`, empty numeric elements are invalid, and some element order differs. The existing XSD test only checks a minimal shell.

Q16. What is the hardest bug you handled?
A: A good candidate is the CONTRACT_ID type mismatch: Mongo compares types strictly, so a string id matched nothing and tax flags came back null, so tax sub-reports never printed. [CONFIRM: whether this was my fix; commits are on other branches]

Q17. Tell me about a RejectRisk bug.
A: The date filter used `$lte` on midnight, so rows later on the end date were lost, and null bounds matched nothing. The fix adds one day and uses `$lt`, adds only supplied bounds. My fix is in my working tree, not committed. [CONFIRM]

Q18. How did you test your work?
A: Unit tests with Mockito, mocking `MongoTemplate` and handlers. Service tests check gates and field mapping per product. There is no integration test against real Mongo. [CONFIRM: which tests I wrote]

Q19. What are the limits of those tests?
A: They prove the code does what the author meant. They do not prove it matches the old Jasper output, they do not hit real Mongo, and they do not validate real XML against the XSD.

Q20. Which technologies did you use?
A: Java 17, Spring Boot 4.1.1, Spring MVC, Spring Data MongoDB `MongoTemplate`, Jackson (JSON and XML), HttpClient 5, Lombok, JUnit/Mockito, Maven, GitHub Actions, Sonar.

Q21. Which Spring Boot version and why?
A: 4.1.1. The team moved from 3.5.14 on 2026-09-10 mainly for a Spring Security CVE fix that was only free in Security 7.1.1. Boot 4 brings Spring 7 and Jackson 3.

Q22. What changed in the Boot 4 upgrade?
A: Mongo properties moved to `spring.mongodb.*`, `@MockBean` became `@MockitoBean`, `fromHttpUrl` became `fromUriString`, an explicit `WebClient.Builder` bean was needed, and Jackson 2 was kept with `preferred-json-mapper=jackson2`. [CONFIRM: my part in the upgrade]

Q23. How does the response look?
A: JSON with `status`, `data` (the XML string) and `timestamp`. Always HTTP 200 for success and for no data. The caller must read the XML.

Q24. How is the XML built?
A: DTOs from handlers are mapped into `Pline*` payload classes that mirror the XSD. `XmlMapper` writes them as XML. The controller wraps the string in JSON.

Q25. What is "one sub-report = one reportData"?
A: Each Jasper sub-report becomes one `reportData` element with its own `prntDetails/transDetList/transDet`. `mainReportName` and `subReportName` say which jrxml it belongs to.

Q26. What are the business gates in ASP?
A: Cover note, schedule, debit note and tax invoice depend on `show*` flags. Debit note also needs a non-zero premium difference. The service-tax invoice needs `showTaxInvoice`, the tax flag, the e-invoice flag, and `checklistCopy` not Y.

Q27. How does product code change output?
A: Flags such as 070504, 070412, 070406, 070414 and 100908 switch which bands and nested pages print. Template12 also checks scheme names like ASTRO, BMW, POSCOVER.

Q28. What external APIs are called?
A: OPUS utility APIs for e-invoice print indicator, service-tax indicator, company BRN, signature, SST rollout date, QR indicator, policy reference. They replace Oracle functions. Failures are caught and defaulted.

Q29. What happens if an external API fails?
A: The call is caught, logged, and a default is used (for example QR indicator N). The report still returns, possibly with a page missing. Timeouts are bounded (5 s connect, 20 s read, 8 s fast lookup).

Q30. How is the database accessed?
A: Read only through `MongoTemplate.aggregate` with raw `Document` stages on collection-name strings. No repositories, no writes.

Q31. What is the biggest risk in your modules?
A: Contract drift between XSD and Java (types, order, new elements), and unconfirmed assumptions about the print engine (for example treating each reportData as an independent print unit).

Q32. What would you improve?
A: Add a real XSD validation test on service output, make dates ISO or change the XSD, replace giant `execute()` methods with a table of bands, add indexes after checking `explain`, add authentication, return proper errors for unknown reports.

Q33. How big are the main classes?
A: `Template12PolicyDocumentService` about 2,235 lines, its base class 3,732, `AspEndtDocumentService` 1,701 with an `execute` of about 1,300 lines. They are hard to read and test.

Q34. How do you handle Oracle errors like ORA-01427?
A: Some handlers copy the behaviour. If WIP and live data disagree where Oracle would return two rows, they throw `IllegalStateException`. Template12 PolicyDocument and the ASP service-tax handler do it; PD2 and others do not.

Q35. What is the Landscape report in Template12?
A: Wide tables (insured, benefits, premium, nomination, diagnosis) in landscape orientation. It has up to 16 bands. It is skipped when the scheme is marked `CI_LANDSCAPE_PRT` in `CMGE_CODEREL_DET`.

Q36. How did you deal with the Jasper originals?
A: I read the jrxml files from the Jasper dump to find sub-reports, print conditions and SQL, then matched them to handlers. Conditions based on page number cannot be done in Java and are left to the renderer.

Q37. How do you know you matched the old report?
A: I read both sides and wrote tests per gate. I did not compare real output on real contracts, so parity is by design review, not data comparison. [CONFIRM: any SIT/UAT comparison I did]

Q38. What does `checklistCopy` mean?
A: A draft copy printed before issue. It makes the code read WIP data and closes the service-tax invoice gate in ASP.

Q39. How does this service get deployed?
A: Through a GitHub Actions workflow that calls a shared `java-microservice` action; Helm charts are in a separate repo. [CONFIRM: my deploy steps and environments]

Q40. How is code quality enforced?
A: Sonar quality gate on pull requests, CodeQL, secret scanning, a Jira key check on commits, and an AI review workflow. Branch rules are in the shared workflows.

Q41. What did you learn?
A: Mongo type strictness, translating SQL semantics (nulls, ordering, joins), reading old Jasper reports, and that tests must be tied to behaviour, not stage numbers. [CONFIRM: personal lessons]

Q42. What Jira work was in your scope?
A: Analysis, coding, unit tests and XSD changes for the three reports. [CONFIRM: ticket numbers]

Q43. How did you break down the work?
A: Per report: read Jasper, list sub-reports and conditions, map fields to the XSD with the mapping sheet, write handlers and service, write tests, review with the team. [CONFIRM]

Q44. How did you check field mappings?
A: Against the XSD and the mapping workbook (`RejectRisk_XSD_Mapping_sheet.xlsx` for RejectRisk). I grepped XSD names such as `reasoncode`, `updatedby`, `rownumber`. [CONFIRM: workbook use for Template12/ASP]

Q45. Which collections matter in RejectRisk?
A: `CMUW_PL_REFER_RISK` is the driver; `CPGE_VI_PARTNERS_ALL` for customer; `CMGE_CODE` for three descriptions; `RPGE_RPT_TEMPLATE` for `dataClass`.

Q46. How do you get row numbers?
A: In the pipeline with `$setWindowFields` and `$documentNumber`, sorted by `customerName`. This copies Oracle ROWNUM after ORDER BY.

Q47. What configuration did you touch?
A: Mostly none for the reports. The one local change in my tree is a Mongo URI in `application.properties`, which must not be committed. [CONFIRM]

Q48. How do you handle errors?
A: Sub-bands catch and log, giving a partial report. Primary fetches and dispatch throw. Global handler returns 400 or 500 with a generic message. Unknown report is 200 SUCCESS with an empty body.

Q49. How would a support engineer debug a missing page?
A: Find which gate controls the page, check the request flags and product code, check WIP vs live data, check if the external API defaulted, then read the handler pipeline in DEBUG logs.

Q50. Where do you want to take this?
A: Make the contract testable: one shared XmlMapper bean, an XSD validation test with a ratchet baseline, and agreement with the print team on dates and new elements. [CONFIRM: my own goals]

---

## 17. Top 50 Scenario Questions

S1. Production: Template12 returns HTTP 500 for one policy. What do you do?
A: Check the request for an empty or non-numeric `pContractId` (landscape dispatch uses `Long.parseLong`). Then check for WIP and live data conflict, which throws `IllegalStateException`. Read the log for the thrower, because the global handler does not log.

S2. ASP endorsement prints without the service-tax invoice.
A: Check all four conditions: `showTaxInvoice=Y`, tax flag, e-invoice flag, and `checklistCopy` not Y. If the flags are null, suspect the CONTRACT_ID type mismatch. If an API failed, the flag defaulted to N.

S3. ASP debit note missing.
A: It needs `showDebitNote=Y` and a non-zero premium difference. Check `DIFF_PREM_DUE` for the endorsement and `UWGE_VI_ENDT_DEBIT` data. Note the doc may also depend on the `EndtDocument2` main handler result.

S4. RejectRisk returns zero rows for a single day.
A: On HEAD, from=to means `<= midnight`, so only exact-midnight rows match. The fix is `$lt (to + 1 day)`. Confirm which build is deployed. [CONFIRM: merge status of the fix]

S5. RejectRisk with no dates after the fix returns a huge response.
A: With the fix, no date means no date filter, so all rows return. Oracle returned none. Decide with business: keep parity (return empty) or require at least one date or cap rows.

S6. Business says a customer in RejectRisk is shown first, in Oracle he was last.
A: Mongo sorts null and missing first; Oracle puts nulls last. Rows with no partner match or null name appear at the top and get row numbers 1..k. Fix with a null-flag sort key or sort in Java.

S7. A caller sends the name `.../Template12/Landscape`.
A: There is no bean now. The request falls through to the legacy switch and returns HTTP 200 SUCCESS with an empty transaction (guide statement). The caller must use the PolicyDocument name.

S8. After deploy the app connects to localhost:27017.
A: Boot 4 reads `spring.mongodb.*`. Old `spring.data.mongodb.*` keys are ignored silently. Fix the property names or the environment variable names.

S9. Every POST fails with a message conversion error after a Boot upgrade.
A: Boot 4 defaults to Jackson 3, which ignores `com.fasterxml` annotations on our DTOs. We set `spring.http.converters.preferred-json-mapper=jackson2`.

S10. The print team says XML fails XSD validation.
A: Expected for some things: date format, empty elements, element order, and the two new report-name elements. Ask which XSD version they use, collect all errors with an error-collecting validator, then fix by priority (order and types first). [CONFIRM: what the print team actually does]

S11. A new field is needed in the output. Steps?
A: Add the leaf in the XSD at the right position and type, add the field in the same place in `Pline*` payload, populate it in the service, update the mapping sheet and changelog header, add a test.

S12. The XSD owner changes a type from string to dateTime.
A: That is a breaking change for senders. The Java sends `dd/MM/yyyy`, so a strict validator rejects it. Options: emit ISO, use a pattern type in the XSD, or add a parallel string field. Agree with the print team first.

S13. A slow external API makes reports hang.
A: Check the timeouts: fast-lookup 8 s, others 20 s. Which client uses which template matters. Failures default and continue. Also check Mongo `maxTime` (600 s in many handlers, none in RejectRisk).

S14. Someone re-enables `Template12LandscapeService` by uncommenting `@Service`.
A: Landscape bands would render twice, because the merged bean already runs them. Keep the old classes unregistered.

S15. A reviewer asks you to unify the two Template12 primary fetches.
A: I would explain it was deliberate (class javadoc "CRITICAL FACT #1"): PD throws on conflict, PD2 prefers posted data. Unifying changes PD2 behaviour and needs product sign-off.

S16. Unit tests start failing after your RejectRisk fix.
A: Seven handler tests assert `$lte`, null bounds and fixed stage indexes. Update them to `$lt (to+1 day)`, no date stage when no bounds, and use stage lookup by content, not index. Add tests for the new cases. [CONFIRM: whether I already did this]

S17. You find a password in `application.properties` in your working tree.
A: Do not commit it. Restore or exclude that file, use an environment variable or secret store, and tell the owner. The repo has secret scanning; gitleaks allowlists these files, so the scan may not catch it.

S18. Landscape page missing for a Template12 policy.
A: Check the dispatch: the scheme may be excluded in `CMGE_CODEREL_DET` (`CI_LANDSCAPE_PRT=Y`), the band conditions (product, `ASP_CUTOFF_IND`), or nominee/diagnosis presence. An empty dispatch returns nothing.

S19. ASP added-member landscape missing.
A: Dispatch needs `ENDT_CODE` 72 or 168 and a risk with `ACTION_CODE=A`. For deletion: `ENDT_CODE` 98, `ACTION_CODE=C`, and `TERMINATE_DATE` not null. Check `UWPL_VI_ENDT_ADD_DEL_RISK`.

S20. Report is correct in dev and wrong in UAT.
A: Check which data differs (WIP vs live), which base URLs and DB the pod uses (there are two base URL keys), and whether the same build is deployed. Configuration drift in Helm values is a known risk. [CONFIRM: environments]

S21. The business wants case-insensitive customer name search.
A: Today it is exact and case sensitive, like Oracle. Options: a collation, a case-insensitive regex with escape, or a normalized field. Regex after the lookup will not use an index; I would raise that cost.

S22. RejectRisk is slow for a big date range.
A: Each matched row triggers four sub-pipeline lookups. Check indexes on `CREATED_DATE`, `CPGE_VI_PARTNERS_ALL(PART_ID, VERSION)`, `CMGE_CODE(CAT_CODE, CODE_CD)`. Memory limit of sort (100 MB) can be hit because no `allowDiskUse` is set. Add pagination.

S23. The dates in RejectRisk are one day off.
A: The window is built at UTC midnight. If data holds real instants of Malaysian time, rows near midnight shift. Check a known record and consider `Asia/Kuala_Lumpur` for the window. (inferred; needs data check)

S24. The renderer prints only one RejectRisk row.
A: The service builds one reportData per row, but a code comment says the renderer was not confirmed to treat each as a print unit. Ask the report team; do not assume.

S25. A client sends `filterCondition` with a number value.
A: `filterCondition` is `Map<String,String>`. A JSON number or array fails binding (400/500), except `pBranch`/`pBranchName` arrays, which the filter flattens. Ask the caller to send strings.

S26. A request has no `metadata`.
A: The controller reads metadata without a null check, so it likely throws NullPointerException and returns 500 (inferred from code). Ask the caller to always send metadata.

S27. You must add authentication.
A: Add a filter or Spring Security rule that checks a gateway token or API key, keep health open, and close Swagger in prod. Today `anyRequest().permitAll()` and protection is assumed at network level.

S28. Security asks about personal data in logs.
A: `CustomRequestFilter` prints bodies with `System.out`, Mongo DEBUG logs queries, and legacy path logs XML at INFO. RejectRisk has names and ID numbers. Remove prints, use DEBUG only in non-prod, mask data.

S29. A Wiz scan flags a new CVE.
A: Follow the pom practice: pin the property to a fixed version with a comment, or upgrade Boot if the BOM has it. Remove old pins that fall below the BOM. Watch property names (`jackson-2-bom.version`).

S30. A pinned version is lower than the BOM.
A: `commons-lang3` 3.18.0 (BOM 3.20.0) and `logback` 1.5.34 (BOM 1.5.38) are pins below the BOM. Check `dependency:tree` and the CVE floor, then drop the pin if the BOM is safe.

S31. `@SpringBootTest` fails at startup after a version pin.
A: This happened with the Micrometer 1.15.12 pin: a Boot 4.1.1 class was missing. The fix was to remove the pin and use the BOM version 1.17.1.

S32. Swagger fails with NoClassDefFoundError.
A: springdoc 2.8.15 was not compatible with Boot 4 (relocated `WebMvcProperties`). The pom now uses springdoc 3.1.1.

S33. A product owner asks for a new scheme in Template12.
A: Add the scheme flag in `execute()`, call the right bands, check the dispatch rules for landscape, add tests for the gate, update the mapping and XSD if new fields are needed. Remember all three primary fetches derive scheme.

S34. A PR touches both tests and 200 production files.
A: That makes review and bisect hard (this happened in commit 3f2157b28). I would ask to split test-only and production changes.

S35. The test classpath uses a copy of `AspServiceTaxInvoiceHandler`.
A: The test tree has a same-name copy, so tests may run the copy and not the real class, and JaCoCo may be misleading (inferred). Delete the copy and re-run the tests.

S36. Sonar quality gate fails on a PR.
A: Open the Sonar report, fix new issues or coverage on new code. Do not add empty tests just to raise coverage; the repo already has coverage-driven tests like `pipelineLambdas_fullyForced`.

S37. How would you roll back a bad release?
A: Redeploy the previous image tag through the Helm repo. The service has no database state, so rollback is stateless; config drift in values is the risk. (inferred) [CONFIRM: real rollback process]

S38. A report works for one product but fails for BMW.
A: BMW prints one row per certificate between `pCertNoFrom` and `pCertNoTo`, and debit note is skipped for BMW. Check the cert range and `pCertInd`. Also `certNo` is hardcoded empty in merged PD2, so BMW nomination gets no cert.

S39. Business asks "why does the PDF show a blank page here?"
A: Blank pages depend on page numbers (duplex printing). Java cannot compute page numbers, so some blank logic is left to the renderer. Ask the print team how they handle it.

S40. A user says the QR code is missing on the invoice.
A: In ASP it needs consent Y, status VALID and QR indicator Y. In Template12 the QR section uses the request `pTaxInvNo`, which is empty if the caller does not send it (inferred). Check both.

S41. Two callers get different data for the same contract.
A: Check `versionNo`, `checklistCopy` (WIP vs live), and whether data changed between calls. The service is read-only so the same inputs should give the same output except timestamp.

S42. You must prove a change did not break other templates.
A: Template12 has its own base class for this reason. For shared classes I would run the full plines test suite, and check callers of any shared helper. Note motor classes are shared by other LOBs.

S43. The team asks for sub-report names in the listing report.
A: Done in commits e74bc0d5c and 861f0666c (OPUSM-44159): `PlineReportNames.tag` fills `mainReportName`, `PlineReportBlocks` sets `subReportName`. These elements are newer than v0.1.6. [CONFIRM: whether I made these commits]

S44. A reviewer says "the XSD test passes so output is valid".
A: Correct them politely: that test validates a hand-written shell and never calls a service. I would show the plan for a real test: run the service with mocked handlers, serialise with the production mapper, collect all errors.

S45. How would you validate a RejectRisk response yourself?
A: Save the XML, use a JDK `SchemaFactory` validator with the Pline and Common XSD files in one folder (so the include resolves), and print all errors with an error handler. Compare the count before and after changes.

S46. The Mongo `$sort` memory error appears in RejectRisk.
A: Add `allowDiskUse(true)` to the aggregation options, narrow the date range, or paginate. The handler sets no options today.

S47. A key-based lookup returns nothing for a number-looking id.
A: Check the BSON type. `CONTRACT_ID` is numeric; a string id will not match. Convert with `Long.parseLong` and keep a fallback for non-numeric values.

S48. Your branch is behind the long-lived branch.
A: The repo pattern is to merge `retrofit-develop-R2` into the feature branch before a PR. Fixes like `cc6a1b84d` are on a diverged branch (194 ahead, 1363 behind), so a cherry-pick may be needed. [CONFIRM: team process]

S49. A new person must support Template12 tomorrow.
A: Give them the product-code table, the gate list, the three fetch methods, the dispatch rules for landscape, the unregistered classes warning, and the production-support table from section 11.

S50. Business asks for an ETA for full XSD validation.
A: I would first agree on the contract (which XSD version, date format, new elements), then add the ratchet test, then fix order and types by priority. I would not give an ETA before the contract is agreed. [CONFIRM: team decision]

---

## 18. Top 50 Trap Questions

T1. "Your service generates the PDF, right?"
A: No. It returns XML in JSON. A print engine makes the PDF. Saying otherwise is wrong.

T2. "You have three Template12 reports, so three beans?"
A: No. One bean, `Template12PolicyDocumentService`. PolicyDocument2 and Landscape are merged parts. The old classes have `@Service` commented out.

T3. "ASP means Accident Student Plan, correct?"
A: I cannot confirm any expansion. Code and Jasper never write it. Internal names are `ASPlus...`, and product config code is 070412. [CONFIRM: real meaning]

T4. "Your output is XSD valid, right?"
A: No. XSD v0.1.6 is the only XSD I found, and real output fails it on date format, empty elements, element order and new elements. Say what was aligned and what was still open.

T5. "Which XSD version do you use?"
A: v0.1.6 is what exists on my machine and in the repo test folder. Java comments mention v0.1.7 to v0.1.10 which I never saw. [CONFIRM: newest version]

T6. "Did you write commit c0c6d449a?"
A: I cannot claim that from git. The repo shows other author names. I can explain what it does. [CONFIRM: which commits are mine]

T7. "Is the API secured?"
A: No. `anyRequest().permitAll()`. Security is assumed upstream (gateway or network), which is not visible in the repo.

T8. "Unknown report name returns 404, right?"
A: No. It returns 200 with status SUCCESS and an empty default transaction. The 404 handler exists but is never thrown.

T9. "Is `xmlData` the response field?"
A: README says that, but the class `SuccessResponse` has `data`. Check a live response before quoting.

T10. "Your RejectRisk date fix is merged, right?"
A: It is uncommitted in my working tree. It equals commit `cc6a1b84d` on another branch. I do not know its merge status. [CONFIRM]

T11. "Do all the tests pass?"
A: I do not know. Tests were never run in this analysis, and no surefire reports exist. After the RejectRisk fix, 7 of 15 handler tests probably fail.

T12. "What is your test coverage?"
A: No coverage numbers exist in the repo (no JaCoCo output). JaCoCo feeds Sonar and the gate is on the server. Do not quote a figure. [CONFIRM: Sonar number]

T13. "Do your tests validate against Mongo?"
A: No. They mock `MongoTemplate`. There is no Testcontainers or embedded Mongo in these tests.

T14. "The XSD test proves service output is valid?"
A: No. `PlineXsdValidationTest` validates a hand-written minimal shell. Four tests named "generates valid XML" never call a service.

T15. "You said 'ASP landscape runs only when needed'."
A: Wrong. Landscape dispatch runs on every ASP call and always appends two dispatch rows, even with zero counts. It costs 5-6 extra queries.

T16. "Does showComplaint control ComplaintRedress in ASP?"
A: No. It is only echoed. Product code 070412 decides, and the ComplaintRedress result is discarded anyway.

T17. "Debit note prints whenever showDebitNote is Y?"
A: No. It also needs a non-zero premium difference (and in Template12, not BMW and `copyType` not `BC`).

T18. "Service-tax invoice prints when showTaxInvoice is Y?"
A: No. Four conditions in ASP: `showTaxInvoice`, tax flag, e-invoice flag, and `checklistCopy != Y`.

T19. "Spring Boot properties for Mongo are `spring.data.mongodb`?"
A: Not in Boot 4. Use `spring.mongodb.*`. The old keys are ignored and the app uses localhost.

T20. "Boot 4 uses Jackson 2 by default?"
A: No. It defaults to Jackson 3. We force Jackson 2 for HTTP with a property, because our annotations are `com.fasterxml.jackson`.

T21. "`jackson-bom.version` pins Jackson 2.21.5?"
A: In Boot 4.1.1 that property controls Jackson 3. The Jackson 2 property is `jackson-2-bom.version`.

T22. "Tomcat version is the Boot managed one?"
A: No. The pom pins 11.0.25; the BOM gives 11.0.24. The pin fixes three critical Wiz CVEs per the pom comment.

T23. "All version pins are above the BOM?"
A: No. `commons-lang3` 3.18.0 and `logback` 1.5.34 are below BOM values 3.20.0 and 1.5.38. Check if intended.

T24. "JUnit version is 5.9.0?"
A: Declared 5.9.0 in the pom, BOM manages 6.0.3. The effective version is not verified; run `dependency:tree`.

T25. "Mongo credentials are in the repo, so it is fine to commit the properties file?"
A: No. The URI with credentials in my working tree is uncommitted and must stay out. Use env or a secret store. The gitleaks allowlist for this file means CI may not catch it.

T26. "Template12 has one primary fetch?"
A: It has three that derive product/scheme (PD main, PD2 handler, landscape dispatch), with different rules. Two are deliberate; the divergence is a risk.

T27. "Page-number blanks are implemented?"
A: No. Conditions using `$V{PAGE_NUMBER}` cannot run in Java. They are left to the renderer.

T28. "Java matches Jasper exactly?"
A: No. Examples: two PDS bands Jasper disables are wired in Java, ATC page prints for every product, `RELATIONSHIP_DESC` is not cut with SUBSTR, six ASP fields are not mapped.

T29. "The schedule of benefits in ASP is produced?"
A: Not by this service. The handlers are stubs returning a dummy row; nothing is mapped. Whether the renderer prints it from the master row is unknown.

T30. "Mongo `$lte` with a null bound works like Oracle?"
A: No. A null bound matches only null dates in BSON order, so it returned zero real rows. Oracle also returned none by SQL logic, but by a different route.

T31. "Mongo sorts nulls last like Oracle?"
A: No. Ascending sort puts null and missing first. Oracle puts nulls last by default.

T32. "Index exists on CREATED_DATE?"
A: Unknown. No indexes are defined in the repo and `auto-index-creation` is in the wrong place. My index list is a recommendation.

T33. "`spring.profiles.active=dev` means dev config?"
A: No. No profile files exist, so it is only a label. Environment is decided by base URL and Mongo keys.

T34. "One base URL key switches the environment?"
A: No. `API_BASE-URL` and `plines.api.base-url` are separate, plus Mongo keys. Several old keys point to dev1.

T35. "Are `option.*` pool settings applied?"
A: No. They are not bound to anything. Driver defaults apply.

T36. "The 404 `NoDataFoundException` path is used?"
A: It is never thrown today. It is a dead path.

T37. "Rollback is a documented procedure?"
A: Not in this repo. I think it is a Helm redeploy of the previous image (inferred). I must confirm with the team.

T38. "The deploy runs automatically on merge?"
A: No. `cicd-workflow.yml` is `workflow_dispatch` only (manual). Release trigger is commented out.

T39. "CodeQL runs on every PR to R1/R2?"
A: Its branch filters do not include the retrofit R1/R2 branches, so probably not (inferred).

T40. "Spring Data Mongo is 5.1.1 from the pom?"
A: The version is not a direct pom entry. It comes via spring-data-bom 2026.0.1, and 5.1.1 is quoted in a pom comment.

T41. "The test with `AspServiceTaxInvoiceHandler` tests the real class?"
A: Probably not. A same-name copy in the test tree may win on the classpath (inferred, not run).

T42. "You did all the 865 tests?"
A: No. The test base was written 2026-09-25 to 09-30 by several authors, about 39 percent are getter/setter tests. I describe only what I wrote. [CONFIRM]

T43. "`mainReportName` is in the XSD?"
A: Not in v0.1.6. Java adds it, citing a later version. Validation against v0.1.6 fails on it ("Invalid content ... mainReportName").

T44. "Does the print engine validate against the XSD?"
A: Unknown. Nothing in the repo shows it. Treat it as an open question for the print team.

T45. "RejectRisk with empty result still returns a title block?"
A: No. Zero rows gives an empty reportData list with no header block, though Jasper printed a title and "No Data Found". Confirm with the report team.

T46. "ROW_NUMBER = 1 added by the pipeline gives the row number?"
A: No. It was a dead constant stage dropped by `$project`. The row number comes from `$setWindowFields`.

T47. "Did you finish all your work by 30 Dec 2026?"
A: The repo history I can see ends 2026-10-08, and today is before the end of the window. I describe only work I can stand behind. [CONFIRM: final state at handover]

T48. "Does the service retry external calls?"
A: I did not find retry for Plines clients; failures are caught and defaulted. Retry keys (`ext.pme.retry.*`) exist for another client. (inferred)

T49. "Mapping workbooks are all reviewed?"
A: Only the RejectRisk sheet was read fully; the master workbook was only skimmed. Read them before the interview. [CONFIRM]

T50. "CSRF is disabled, so it is insecure?"
A: It is disabled with a justification: stateless, no cookies, machine clients. That is acceptable only while there is no browser session. The bigger issue is no authentication.

---

## 19. Top 50 Technical Questions

X1. What is `ReportStrategy`?
A: An interface with `execute(request)`. Each report service implements it and is registered as a Spring bean with the report path as name. The controller picks it from a map.

X2. How does Spring inject all strategies?
A: By injecting `Map<String, ReportStrategy>`. Spring fills it with bean name as key and bean as value.

X3. What does `CustomRequestFilter` do?
A: It wraps the request, reads JSON, and if `pBranch` or `pBranchName` is an array, joins it with commas so `Map<String,String>` binding works. It also prints bodies with `System.out`, which is a logging risk.

X4. What is a handler?
A: A `@Component` that runs one or more Mongo aggregations for one Jasper sub-report and returns a DTO. Services call many handlers.

X5. What is `$lookup` with `let` and `pipeline`?
A: A correlated join. `let` passes fields from the outer document; the inner `$match` with `$expr` uses them. We add `$limit 1` to act like a scalar sub-select.

X6. Why `$arrayElemAt` instead of `$unwind`?
A: `$unwind` drops documents with an empty array. `$arrayElemAt` keeps the row with null, like a left join.

X7. What does `$setWindowFields` with `$documentNumber` do?
A: It numbers documents 1..N in the sorted order. We use it for Oracle ROWNUM after ORDER BY. It needs MongoDB 5.0 or later.

X8. How do you emulate `SELECT DISTINCT`?
A: With `$group` on the selected fields. Template12 coverage uses it and sorts with `$toDouble(SORT_SEQ)`.

X9. How do you emulate Oracle UNION of WIP and live?
A: `$unionWith` of the WIP collection, or two lookups merged with `$ifNull`. Then de-duplicate if needed.

X10. Why is the date filter `$gte from` and `$lt to+1day`?
A: Oracle used `TRUNC(CREATED_DATE) BETWEEN`. With a timestamp field, the same whole-day result is `>= from` and `< to+1`. It keeps the field bare so an index can be used.

X11. Why must null bounds be skipped?
A: In BSON comparison `{ $gte: null }` matches only null/missing values. So a null bound would give zero real rows. We only add non-null bounds.

X12. What is type strictness in Mongo?
A: `{CONTRACT_ID: "123"}` does not match a numeric 123. There is no implicit cast. We convert ids with `Long.parseLong` / `safeNum`.

X13. How is `MongoTemplate` mocked in tests?
A: As `@Mock`. Tests stub `aggregate(any(Aggregation.class), any(String.class), any(Class.class))`. To check pipelines they capture the aggregation and call `toPipeline(...)`.

X14. What is `allowDiskUse` and `maxTime`?
A: `allowDiskUse` lets blocking stages (sort, group) use disk beyond 100 MB. `maxTime` limits query time. Many Template12/ASP handlers set 600 s; RejectRisk sets none.

X15. How is the XML produced?
A: Payload objects with Jackson XML annotations go to `XmlMapper.writeValueAsString`. Names with odd case use `@JacksonXmlProperty(localName=...)`. Lists use `useWrapping=false`.

X16. Why do some elements appear as `<x/>`?
A: Payload classes use `@JsonInclude(ALWAYS)` and a plain `XmlMapper`, so null becomes an empty element. Empty is invalid for decimal or date in the XSD.

X17. Why `Policyletterind` with a capital P?
A: The XSD element has that case. Lombok getters would give `policyletterind`, so the class uses field visibility or `@JacksonXmlProperty` to emit the exact name.

X18. What is `@JsonIncludeProperties` in `PlineReportParameters`?
A: A whitelist of property names. It stops base-class motor fields leaking into XML. Side effect: a setter for a name not in the list is silently dropped (for example `pContractId`).

X19. Why shadow fields in `PlineReportParameters`?
A: Plines subclasses shared motor classes. Shadow fields and `@JsonIgnoreProperties` hide old names so only XSD names are written, without changing motor behaviour.

X20. What is `PlineReportNames.tag`?
A: A helper that sets `mainReportName` (from the bean path) on each reportData and fills blank names. `PlineReportBlocks` builds header blocks with `subReportName`.

X21. What is `mergeSingleTransDet`?
A: A reflection helper that copies non-null fields onto every existing transDet to avoid multiplying rows across single-row bands. Multi-row bands use `addSingleTransDet`.

X22. Why a separate Template12 base class?
A: `PlinePolicyDocumentBaseService` (4,083 lines) serves about 24 templates with the older flat shape. A sibling base keeps Template12 changes from breaking them and uses the new XSD-typed envelope. Cost: duplicated code.

X23. How does Template12 decide scheme-specific output?
A: `execute()` computes booleans from product code and scheme name, then calls band methods in if/else blocks. Landscape uses its own dispatch handler and `CMGE_CODEREL_DET`.

X24. What is the ORA-01427 emulation?
A: Oracle error "single-row subquery returns more than one row". Helpers like `requireNoConflict` throw `IllegalStateException` when WIP and live values differ, to mimic it.

X25. What does `safeCall` do in `AspServiceTaxInvoiceHandler`?
A: It wraps an external call and returns null on failure, so the report continues. Defaults then apply (QR indicator N).

X26. How are external URLs protected against SSRF?
A: `PlineUtilsClient.sanitize` allows only `[A-Za-z0-9_\-./#]` in values, `assertTrustedHost` checks the URL starts with the configured base, and `PlinePolicyRefClient` uses a stricter regex. The host is never from the request.

X27. What RestTemplates exist and how do timeouts differ?
A: A primary `restTemplate` (5 s connect, 20 s read) and `plineRestTemplate`, plus `fastLookupRestTemplate` (8 s) used by `PlineUtilsClient`. Some old motor clients use `new RestTemplate()` with no timeout.

X28. How is the custom truststore used?
A: `SSLContexts.custom().loadTrustMaterial(...)` from `mytruststore.p12` with PKCS12. It replaces the default cacerts for outbound calls, so a new upstream CA must be in the file.

X29. How does Spring Security config look?
A: Stateless sessions, CSRF ignored for all, health and info permitted, other actuator paths denied, everything else permitted, and an empty in-memory user manager to avoid a generated password.

X30. What does `GlobalExceptionHandler` do?
A: Maps `NoDataFoundException` to 404 (never thrown) and any `Exception` to 500 ProblemDetail with a generic message, without logging the cause. `GenericExceptionHandler` maps `IllegalArgumentException` to 400.

X31. Where is `dataClass` from?
A: In RejectRisk, `PlineReportHeaderHandler` reads `RPGE_RPT_TEMPLATE` by template name (report name + "/" + title). It is set on each row and on the header block.

X32. Why one `transDetList` per row in RejectRisk?
A: In the XSD, `companyType/companyDesc/remarks` is a sibling of `transDet` inside `transDetList`. One list per row keeps remarks tied to its own row.

X33. What do `prntDetails` and `transDet` look like in the XSD?
A: `prntDetails` is the last child of `reportData` (since v0.1.3). It holds info blocks and `transDetList`; `transDetList/transDet` repeats and has about 634 scalar leaves plus 12 lists.

X34. Why xs:sequence matters?
A: The XSD uses only sequence, so sibling order is enforced. Jackson writes fields in declaration order, so Java field order must match the XSD or validation fails.

X35. Why is `elementFormDefault="qualified"` irrelevant?
A: There is no target namespace, so no element is qualified by any namespace. It matters only if a namespace is added.

X36. What is `xs:include` vs `xs:import`?
A: Include merges a schema of the same (or no) namespace; import brings another namespace. Pline.xsd includes Common.xsd because both have no namespace.

X37. How would you write a real XSD test?
A: Mock handlers, call `service.execute`, serialise with the same `XmlMapper` as the controller (extract it into a shared bean), validate with an error-collecting handler, and assert zero errors or a ratchet baseline.

X38. Why must the XSD test use a `File` source?
A: So the relative `Common.xsd` include resolves. An `InputStream` source has no base URI.

X39. What was `benefitSi` mismatch?
A: Java `String` but XSD `xs:decimal`; the only primitive type mismatch found in compared classes.

X40. How are dates handled in payloads?
A: `LocalDateTime` fields use `@JsonFormat(pattern="dd/MM/yyyy")`. Template12 also reformats date-named string fields by reflection at the end. XSD wants `xs:dateTime`, so they differ.

X41. What does `DateUtils.toLocalDate` accept?
A: `yyyy-MM-dd` fast path, `dd-MM-yyyy`, `dd/MM/yyyy`, `MM/dd/yyyy` (after dd/MM fails), `dd-MMM-yyyy`, `dd-MMM-yy`, a `Date.toString()` style, and ISO offset forms. A local datetime without offset fails.

X42. Why is `toLocalDateTime` in RejectRisk risky?
A: It swallows every exception and returns null, so a parse failure silently drops the date from the output.

X43. What is the master row in ASP?
A: `reportData[0]` with `mainReportName=ASPEndtScheduleEndtDocument`, `subReportName=PrintSeriesFooter`, carrying shared fields such as product, issue date, endorsement code and labels.

X44. How is the debit note number built?
A: `D` + `ENDT_NO` if `PREM_DUE >= 0`, else `C` + `ENDT_NO`. For some product codes extra fee differences are added to gross premium.

X45. How is TIN masked?
A: In `AspServiceTaxInvoiceHandler` all but the last 4 characters are masked.

X46. What is the CONTRACT_ID bug fix in code?
A: `AspEndtDocument2MainHandler` overrides `buildCommonPipeline` and uses `safeNum` (`Long.parseLong`, fallback to the string). It is ASP-only so other templates are not affected.

X47. How do tests name and structure?
A: `methodUnderTest_condition_expectedResult`. Mocks every handler, one big `setUp`, helpers like `reportDataList(t)`. Lenient Mockito strictness is used in 36 files, which can hide dead stubs.

X48. What are the weak patterns in the test base?
A: Getter/setter tests for Lombok classes, tests tied to pipeline stage index, lenient mocks, `forcePipelines` just to run lambdas, placeholders named as XSD tests, and a production class copy in the test tree.

X49. How does CI run tests?
A: `sonar-code-review.yml` runs `mvn -B verify ... sonar:sonar -Dsonar.qualitygate.wait=true`. JaCoCo is bound to the test phase with no minimum check in the pom; the gate threshold is on the Sonar server.

X50. What does `spring-boot-devtools` and `org.json` mean for risk?
A: `org.json:json:20231013` is old and parses request bodies in the filter without a size limit (DoS risk, inferred). Devtools is listed in the pom; normally excluded from the fat jar, verify. These are cleanup items, not report logic.

---

## 20. Project explanations (first person)

### 20.1 One minute

I worked on the OPUS document print data service at Allianz from May to December 2026. It is a Spring Boot microservice. It does not create PDFs. It gets a report name and some filters, reads data from MongoDB and a few OPUS APIs, and returns XML inside a JSON response for the print engine. The data in Mongo is a mirror of the old Oracle database, so the old Jasper SQL had to be rewritten as Mongo aggregations. I owned three Personal-Lines reports: Template12, ASP and RejectRisk. My Jira scope was analysis, code, unit tests and XSD changes. [CONFIRM: ticket numbers and my exact share of the code]

### 20.2 Three minutes

[Start with the one-minute version.] The design is a strategy pattern. One endpoint, `POST /api/opus/GenerateReport`. Spring finds a service bean whose name is the report path. The service calls handlers; each handler is one Jasper sub-report and runs a Mongo aggregation. The service maps the result into payload classes that follow an XSD called Pline.xsd, and the rule is one sub-report equals one `reportData` element.

Template12 is a policy schedule template. It used to be three Jasper reports: PolicyDocument, PolicyDocument2 and Landscape. In Java there is one bean that does all three, so the print engine gets one response and nothing is rendered twice. ASP is the endorsement document for product 070412, and again four Jasper reports sit in one bean. RejectRisk is a listing of rejected risks, one reportData per row.

On the XSD side, I aligned the payload fields and structure, and added report and sub-report names. I will be honest: the only XSD I have is v0.1.6, and real output does not fully validate against it. Dates are `dd/MM/yyyy`, some elements are empty, and some order differs. [CONFIRM: what exactly I changed in the XSD]

### 20.3 Five minutes

[Start with the three-minute version.] Let me give examples from each module.

For ASP, the service-tax invoice only prints if four things are true: `showTaxInvoice` is Y, the tax flag is Y, the e-invoice flag is Y, and `checklistCopy` is not Y. I remember a bug where the tax flags came back null, because Mongo compares types strictly and the contract id was a string while the data was numeric. The fix converts the id to a number in an ASP-only handler. That fix lives on another branch, so I would check its merge status. [CONFIRM: whether this was my work]

For Template12, the interesting parts are that there are two different primary fetches on purpose, one that throws on conflicting WIP and live data like Oracle did, and one that prefers posted data. Landscape pages are switched off for some schemes through a code relation table. There is a reflective merge helper that puts single-row bands into the same transDet so rows do not multiply.

For RejectRisk, the original SQL used `TRUNC(CREATED_DATE) BETWEEN`. A direct Mongo translation lost the last day, and a null date bound matched nothing. The fix adds one day to the end date and uses `$lt`, and only adds the bounds that are present. I have this fix in my working tree; it matches a commit on another branch, and it will break about seven handler tests that I need to update. The row number comes from `$setWindowFields`.

Testing was unit tests with mocked Mongo and mocked handlers. I want to be clear that they do not compare against real Jasper output or real Mongo data. [CONFIRM: the tests I wrote]

### 20.4 Ten minutes

[Use the five-minute version, then add these parts.]

Technology and platform. It is Java 17 and Spring Boot 4.1.1. The team upgraded from 3.5.14 in September for a Spring Security CVE. Boot 4 brings Jackson 3, but our DTOs use Jackson 2 annotations, so we force Jackson 2 for HTTP. Mongo properties moved to `spring.mongodb.*`; the old keys are silently ignored and the app falls back to localhost. Some versions are pinned in the pom for scanner findings, such as Tomcat 11.0.25 and Jackson 2.21.5, and two pins are below the BOM, which I would review. [CONFIRM: my part in the upgrade, if any]

API and errors. The API has no authentication and returns HTTP 200 even for an unknown report name, so the caller has to look at the XML. I would add auth at the gateway or in the app and return a clear error for unknown reports. The global handler does not log the cause of a 500, which makes support harder.

Configuration. There are two base URL keys, several dead keys and plaintext secrets in properties files, protected only by a gitleaks allowlist. I would move secrets to the environment and clean the dead keys.

XML and XSD. The XSD is all `xs:sequence`, almost everything optional, no namespace. That means the contract is about order and types, not about required fields. The Java output differs in date format, empty elements, sibling order and new elements like `mainReportName`. The existing XSD test only checks a minimal shell. If I had more time, I would build a real validation test that runs each service with mocked handlers, serialises with the production mapper and counts all errors, with a baseline that goes down over time. I would also agree with the print team on dates and the newer XSD.

Known gaps in my modules. ASP stubs for the schedule of benefits, LLB and STI pages emit nothing. Template12 cannot do page-number blank logic. RejectRisk returns all rows when no dates are given, which differs from Oracle. Sorting differs for null names. The test base is large but much of it is getter/setter tests.

Delivery. Work followed the repo process: feature branch, merge from the integration branch, PR with Jira key, Sonar gate, secret scan. Deployment is a manual GitHub Actions run with a shared action, and Helm values are in another repo. [CONFIRM: my real deployment and rollback steps]

What I would do next: real XSD test, null-safe sort, `allowDiskUse` and pagination for RejectRisk, remove the test-tree copy of the tax invoice handler, split the 1,300-line `execute()` methods into a table of bands.

---

## 21. Honest gaps list (verify with the team BEFORE the interview)

1. ASP expansion. Not in code, Jasper or any file. Ask the business. Until then say "I did not find a written expansion".
2. Newest XSD version. Only v0.1.6 exists on my machine, but Java comments cite v0.1.7 to v0.1.10 (`mainReportName`, `subReportName`, `pCreatedDateFrom/To`). Ask the schema owner for the real latest file and whether it matches what Java emits.
3. Merge status of commit `cc6a1b84d` (OPUSM-41763, 2026-10-05) and of the commits that fix CONTRACT_ID (578778080, ba9e29586) and rewrite `AspEndtDocumentMainHandler`. They are on a diverged branch (194 ahead, 1363 behind HEAD). My RejectRisk working-tree fix equals it. Decide: commit, cherry-pick, or drop.
4. Whether output was ever validated against the XSD, by me, the print team, or the report team. Ask whether the print engine validates at all and whether it tolerates unknown elements.
5. Rollback and deploy process: who runs `cicd-workflow.yml`, which image tag, Helm values, how pods get the Mongo URI and base URLs, health probe path, monitoring and alerts.
6. Mapping workbooks: read `Pline_MASTER_MAPPING_SHEET_v0.1 (final1) 1 5.xlsx` (only sheet names were checked) and `RejectRisk_XSD_Mapping_sheet.xlsx`. They are the requirement side of my XSD work.
7. Which commits and tests are mine. Git shows other author names for the Plines commits and the test batches. Prepare a list from the Jira tickets and PRs.
8. Test status. Nobody ran Maven. Run `mvn -B verify`, record pass/fail and coverage. Fix the 7 RejectRisk handler tests if the date fix is kept. Check the effective JUnit version with `dependency:tree`.
9. Whether the renderer treats each `reportData` as an independent print unit (RejectRisk disclaimer, Template12 band merge, ASP sections). Ask the report team for sign-off.
10. Whether the renderer prints ASP schedule of benefits, LLB and STI pages from the master row `actionCode`, since Java emits no row for them.
11. RejectRisk rules for "no dates given" (all rows vs empty), null-name sort order, and the time zone of `CREATED_DATE` (UTC instants vs local time). Check one known record.
12. Unknown report name behaviour. Confirm with one test call that it returns HTTP 200 SUCCESS, and what a missing `metadata` or null `reportName` does.
13. Whether the live response field is `data` or `xmlData`.
14. Meaning of `SB_` and `OCP_` collection prefixes, and whether `*_VI_*` are Mongo views or collections; which indexes exist in Atlas.
15. My exact role and dates. The project window runs to 30 Dec 2026; repo history I saw ends 2026-10-08. Be ready to say what was done by handover. Also remove any uncommitted credentials from `application.properties` and rotate if they were ever pushed.

---

## 22. Short summary

- This sheet covers the service, architecture, versions (checked against `pom.xml` and the Boot 4.1.1 BOM), API, database, configuration, XML, XSD, deployment and support, then one page each for Template12, ASP and RejectRisk, 200 practice questions (50 most likely, 50 scenario, 50 trap, 50 technical), four spoken explanations, and a list of 15 things to verify.
- Core truths to say out loud: one live bean each for Template12 and ASP; XSD v0.1.6 is the only XSD and real output does not validate against it; ASP has no known expansion; RejectRisk's fix is uncommitted, equals `cc6a1b84d` on another branch, and likely breaks 7 of 15 handler tests; the API has no authentication; unknown report names return 200 SUCCESS.
- Never claim commits you cannot prove, never quote coverage or test pass counts, and use the [CONFIRM] items to fill in your personal story.
