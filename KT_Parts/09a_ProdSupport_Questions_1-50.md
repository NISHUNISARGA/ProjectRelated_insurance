# 09a - Production Support Questions 1-50 (opus-documentprintdata-service)

Scope: Template12 (PolicyDocument, PolicyDocument2, Landscape), ASP (EndtDocument family) and RejectRisk. Service: Spring Boot 4.1.1, Java 17, MongoDB, XML-in-JSON.

How to use this file
- Every answer is built from the KT files (01 to 06 and the KT Guide) and the repo. "(inferred)" means my reasoning, not something proven by a file or a run.
- No real incident ticket exists in the repo. So never say "last month we had incident X". Use "the code shows this failure mode, and this is how I would check it". Where a personal fact is needed, the text says [CONFIRM: ...] - fill it from your own memory or drop the sentence.
- Git authorship cannot be tied to you. Do not say "I wrote commit X".
- Secrets are never written here (<redacted>).
- Numbering: Q1-Q50 here, Q51-Q100 in 09b. Debugging questions (investigation steps) are in 09c/09d. XML questions are in 09e/09f.

Quick map used in many answers
- Entry: `POST /gi/documentprintdata/api/opus/GenerateReport` -> `CommonController.forwardRequest` -> `reportStrategies.get(reportName)` -> `ReportStrategy` bean.
- Live beans: `AGIC/UW/Plines/PolicySchedule/Template12/PolicyDocument` (`Template12PolicyDocumentService`), `AGIC/UW/Plines/ASPEndtSchedule/EndtDocument` (`AspEndtDocumentService`), `AGIC/UW/Plines/RejectRisk/RejectRisk` (`RejectRiskService`).
- Response: `{status, data (XML string), timestamp}`, HTTP 200 on almost everything.

---

## A. First response and triage

**Q1. A user says "the report is blank". What do you do in the first five minutes?**
- Why the interviewer asks: To see if you have a repeatable triage path for a service that returns HTTP 200 even for bad input.
- Ideal answer: First get the exact request: `reportName`, `pContractId`, `versionNo`, `checklistCopy`, `pEndtNo`/`pRefNo` (ASP), dates (RejectRisk). Then split the problem in three. (1) Routing: did `CommonController` find a bean named exactly `reportName`? An unknown name gives 200, `status: SUCCESS` and a default transaction that only has `templateDetails`/`mailingDetails`. (2) Data: did the handlers return rows? Mongo never errors on a zero-row match. (3) Gates: did a business gate close (for example `showTaxInvoice`, `checklistCopy=Y`, `SHOW_STAX_INV_IND`)? I would look at the XML inside `data`, not at the HTTP code. [CONFIRM: if you have used a real request in dev, say which one.]
- Common mistakes: Trusting HTTP 200. Opening the code before reading the actual XML. Blaming Mongo before checking the report name.
- Follow-up questions: How do you tell a default transaction from a real one? Which gate is the most common reason for a missing ASP tax invoice?

**Q2. The response is 200 and `SUCCESS` but the XML has no `reportData` rows. List the possible causes in order of cheapness to check.**
- Why the interviewer asks: Tests whether you know the "always 200" contract (documented weakness in the KT).
- Ideal answer: (1) Report name wrong or a merged-away name, so no strategy bean matched and `CommonController` fell to the legacy `switch` default, which sets `createDefaultTransaction`. (2) `strategy.execute` returned a `Transactions` with a null `Transaction`, which the controller also replaces by the default one. (3) Zero rows from the primary handler (wrong database, wrong filter, type mismatch). (4) For RejectRisk, no rows means an empty `reportData` list and no ReportHeader block (test `execute_emptyRows_returnsEmptyReportDataList`). (5) All gates closed. Cheap checks first: name spelling, then Swagger or curl with the same body, then logs.
- Common mistakes: Saying "404 means no data" - `NoDataFoundException` is never thrown today.
- Follow-up questions: Where does the controller build the default transaction? What would you change in the contract?

**Q3. A caller still sends `AGIC/UW/Plines/PolicySchedule/Template12/Landscape` (or `.../ASPEndtSchedule/AddMemberLandscape`). What happens and what do you tell them?**
- Why the interviewer asks: Tests knowledge of the merge: one live bean each for Template12 and ASP.
- Ideal answer: Those names have no bean. `reportStrategies.get(reportName)` is null, so the legacy `switch` runs. The KT says the switch has no matching case and ends in the `default:` branch that sets a default transaction (CommonController line ~393). So the caller gets 200 `SUCCESS` and an empty envelope. The data for Landscape and PolicyDocument2 is now inside the `.../Template12/PolicyDocument` response (private methods `executeMergedPolicyDocument2` and `executeMergedLandscape`). For ASP, EndtDocument2, AddMemberLandscape and DeletionMemberLandscape are inside `.../ASPEndtSchedule/EndtDocument`. Tell them to call only the merged name and read the `subReportName` values.
- Common mistakes: Saying "re-enable the old class". Re-enabling `@Service` on the old classes would render each band twice (Template12) or clash bean keys (ASP).
- Follow-up questions: Which `subReportName` values does the ASP response carry? How would you warn callers instead of silently returning 200?

**Q4. What is the minimum a caller must give you so you can reproduce a Template12 issue?**
- Why the interviewer asks: Checks that you know which inputs drive branches.
- Ideal answer: `reportName`, `pContractId` (numeric text), `versionNo`, `companyParamType`, `checklistCopy`, and the show flags `showCoverNote`, `showComplaint`, `showScheduleStd`, `showDebitNote`, `showTaxInvoice`. Also `copyType` (BC suppresses the debit note), `pCertInd`, `pCertNoFrom/To` for BMW, and the key `ShowOutPrintCovLetter` with a capital S. Branching on product code (070504, 070412, 070406, 070414) and scheme (ASTRO, BMW, POSCOVER, AZP1/2, POSMESRA, FWPA) comes from Mongo data, not from the request. So I also need the contract ID, to look up its product and scheme.
- Common mistakes: Asking for "the policy number" only - `pPolicyNo` is just echoed.
- Follow-up questions: Which defaults apply when a key is missing? (`getOrDefault`: `showComplaint` "Y", `showScheduleStd` "Y", others "N", `versionNo` "1".)

**Q5. What do you need from the caller for an ASP endorsement ticket?**
- Why the interviewer asks: Checks the ASP-specific keys.
- Ideal answer: `pContractId`, `pEndtNo`, `pRefNo`, `versionNo`, `companyParamType`, `checklistCopy`, `printSeries`, `copyType`, `pUserID`, and the flags `showCoverNote`, `showEndtSchedule`, `showDebitNote`, `showTaxInvoice` (all default "N"). `showComplaint` is only echoed: ComplaintRedress is decided by product code 070412. `pPolicyNo` is unused. I also need `ENDT_CODE` of the endorsement (72/168 add member, 98 delete), because landscape pages and schedule-of-benefits depend on it.
- Common mistakes: Thinking `showComplaint=Y` forces the complaint page.
- Follow-up questions: Why is `pContractId` required as numeric text? What is `pRefNo` for?

**Q6. Someone calls the API from an unknown machine. Can you find out who it was?**
- Why the interviewer asks: Tests honesty about missing authentication.
- Ideal answer: Not from the application. `SecurityConfig` uses `anyRequest().permitAll()`, CSRF is off, there is no API key or JWT, and no correlation or caller ID in logs. The only partial trace is `CustomRequestFilter`, which prints the request body with `System.out`. Caller identity has to come from the network layer (gateway, ingress logs), which is not in this repo (inferred). I would say so clearly and propose a gateway rule or a simple filter as an improvement.
- Common mistakes: Claiming "Spring Security protects it".
- Follow-up questions: What would a minimal fix look like? What PII does the body print contain for RejectRisk?

**Q7. A request returns HTTP 500 and the application log has no stack trace. Why, and what next?**
- Why the interviewer asks: Tests knowledge of the error handler.
- Ideal answer: `GlobalExceptionHandler.handleGeneric` catches `Exception` and returns a `ProblemDetail` (`INTERNAL_ERROR`) but the class has no logger, so it does not log the exception (confirmed: no `@Slf4j`, no `log.` calls). The root cause is visible only if the thrower logged it. So I search for logs from the likely class (for example a `log.warn` or `log.error` in the handler), and if nothing, I reproduce with the same body in a dev environment and read the exception there. A fix to propose: add `log.error("...", ex)` in the handler.
- Common mistakes: Searching for the stack trace in logs forever.
- Follow-up questions: Which exceptions would become 400? (`IllegalArgumentException` via `GenericExceptionHandler`.) Why does malformed JSON also become 500? (inferred: the catch-all matches first).

**Q8. How do you tell a Template12 500 caused by empty `pContractId` from one caused by WIP/live data conflict?**
- Why the interviewer asks: These are the two documented hard failures for Template12.
- Ideal answer: Empty or non-numeric `pContractId`: `Template12LandscapeDispatchHandler` calls `Long.parseLong`, so a `NumberFormatException` escapes (test `execute_landscapeDispatchHandlerThrows_notWrappedInTryCatch_propagates`; handler test `fetchDispatch_nonNumericContractId_throwsNumberFormatException`). Data conflict: `PlinePolicyDocumentMainHandler` (`requireNoConflict`, `distinctValueOrConflict`) throws `IllegalStateException("Ambiguous ... (Oracle's UNION would raise ORA-01427 here)")`, and `addTemplate12PDPrimaryDocument` has no try/catch. The exception message tells them apart. The first is a caller error. The second is a data problem: the same contract has different product or scheme in WIP and posted tables.
- Common mistakes: Treating both as "Mongo is down".
- Follow-up questions: Who owns the fix for the data conflict? Why did the team copy an Oracle error into Java?

**Q9. What does `/gi/documentprintdata/actuator/health` tell you, and what does it not?**
- Why the interviewer asks: Checks understanding of what is exposed.
- Ideal answer: Only `health` and `info` are exposed (`management.endpoints.web.exposure.include=health,info`); other actuator paths are `denyAll`. Health has no `show-details`, and there are no custom indicators. So "UP" means the process runs; it does not prove Mongo has data, that `API_BASE-URL` points to the right host, or that the truststore is valid (inferred, since no custom checks are in the repo). Note the context path `/gi/documentprintdata` must be in the URL.
- Common mistakes: Calling `/actuator/health` without the context path and concluding the app is down.
- Follow-up questions: How would you check the Mongo link? (A real request or `MongoTemplate` ping - inferred.)

**Q10. Can support use Swagger UI to test?**
- Why the interviewer asks: Tests awareness of exposed docs and risk.
- Ideal answer: Yes, `/gi/documentprintdata/swagger-ui/index.html` is open (`permitAll`), with docs at `/v3/api-docs` and group `public` on `/api/**`. It lists one real endpoint. It is handy for a reproduction, but it is also open to anyone who can reach the pod. Also `PayLoadRequest.filterCondition` is `Map<String,String>`, so Swagger's example is generic; use a saved body instead.
- Common mistakes: Sending numbers or arrays inside `filterCondition` (only `pBranch`/`pBranchName` arrays are flattened by `CustomRequestFilter`).
- Follow-up questions: Should Swagger be disabled in production? (`springdoc.api-docs.enabled=false` is the suggestion in KT 06.)

**Q11. A request without `metadata` crashes. Why?**
- Why the interviewer asks: Checks knowledge of the controller's null handling.
- Ideal answer: `populateMailingDetails` and `populateTemplateDetails` dereference `requestBody.getMetadata()` without a null check (KT 06, inferred from code), so a missing `metadata` gives NPE and then 500. Also `PayLoadRequest.setMetadata` keeps only String and Number values; booleans, nulls and nested objects are dropped, and numbers become strings. So `templateId` as a string like "55" is fine, but a bad value goes through `parseIntSafe` and becomes null.
- Common mistakes: Sending only `filterCondition` in a quick test.
- Follow-up questions: Where do the metadata values end up in the XML? (`mailingDetails` and `templateDetails`.)

**Q12. What is the difference between a 400 and a 500 here, in practice?**
- Why the interviewer asks: Checks the exception mapping.
- Ideal answer: `IllegalArgumentException` anywhere maps to 400 (`GenericExceptionHandler`, "Invalid request parameters"; message only logged at WARN). `NoDataFoundException` maps to 404 but is never thrown. Everything else is 500, including `NumberFormatException`, `IllegalStateException`, `DateTimeParseException` and NPE. Note `DateUtils.toLocalDate` throws `IllegalArgumentException` for unknown formats, but a bad ISO-shaped date like `2026-13-45` throws `DateTimeParseException`, which is 500 (01 deep dive).
- Common mistakes: Expecting 400 for every bad input.
- Follow-up questions: Which input errors would you convert to 400? Why does the caller see the same body for many causes?

---

## B. Configuration and environment

**Q13. After the Boot 4 upgrade the app starts but "cannot reach the database" or connects to localhost. What happened?**
- Why the interviewer asks: This is a classic trap listed in the KT.
- Ideal answer: Boot 4 uses the prefix `spring.mongodb.*` (class `org.springframework.boot.mongodb.autoconfigure.MongoProperties`). Old `spring.data.mongodb.*` keys are silently ignored and the driver falls back to `localhost:27017`. Commit `9248b12ff` added the new keys; `application.properties` has a comment about it. I check which keys are in the effective configuration (file, env var, Helm) and look at the startup log for the connection host.
- Common mistakes: Debugging the network first.
- Follow-up questions: What do `option.max-connection-pool-size` and `auto-index-creation` do? (Nothing - they are not bound to anything.)

**Q14. All reports return empty with no error after a deploy. First suspicion?**
- Why the interviewer asks: Tests thinking about silent failures.
- Ideal answer: A wrong database name or a wrong cluster. A pipeline on a database without the collection returns zero rows, not an error (KT 06, 5.C Q27). The committed file does not hold the Mongo URI for deployed environments (commit `7dd2f2f67` removed it), so the pod receives it from outside the repo (Helm values - not in repo). I compare `spring.mongodb.database` (the working copy shows `OPUSAGICPRODR1`) with what the environment should use. [CONFIRM: which database names the real environments use.]
- Common mistakes: Re-deploying the previous image without checking config.
- Follow-up questions: How can you prove the connected database without reading the secret? (Run one known contract in the dev environment; check `MongoTemplate` DEBUG output - inferred.)

**Q15. Which two properties decide where Template12 and ASP external calls go, and why does that matter in a promotion?**
- Why the interviewer asks: Tests the "two switches" finding.
- Ideal answer: `API_BASE-URL` (read by `PlineUtilsClient` and about 45 other places) and `plines.api.base-url` (read by `PlinePolicyRefClient`, `PlineSSTRolloutDateClient`, `PlinePgRpgeUtilsClient`). Both point at the dev2 host in the committed file, but they are separate keys. Changing only one sends part of the calls to the old host. Also path keys per function. A release engineer must change at least `API_BASE-URL`, `plines.api.base-url`, `spring.mongodb.uri`, `spring.mongodb.database`.
- Common mistakes: Updating only `API_BASE-URL`.
- Follow-up questions: Why is the hyphen in `API_BASE-URL` risky for env vars? (Relaxed binding from `API_BASE_URL` is expected but not tested - inferred.)

**Q16. Reports print but company BRN, signature or QR fields are blank only in one environment. What do you check?**
- Why the interviewer asks: Linking blank fields to external APIs.
- Ideal answer: These come from `PlineUtilsClient` and related clients (`comp-brn-number`, `fn-rppm-get-signature`, `qr-indicator`). Failures are caught and the method returns null or empty, so the report is still 200. I search logs for the `... failed: <message>` lines (for example `fnGetCompBrnNumber url={} failed: {}` at ERROR). Then check base URL, truststore (a new CA missing from `mytruststore.p12` breaks the TLS handshake), and the timeouts (8 s fast-lookup).
- Common mistakes: Looking only for HTTP errors in the response.
- Follow-up questions: Which clients use the 8 s bean and which use the 20 s bean? (`PlineUtilsClient` uses `fastLookupRestTemplate`; `PlinePolicyRefClient`, `PlineSSTRolloutDateClient`, `PlinePgRpgeUtilsClient` take the primary `restTemplate`.)

**Q17. The truststore. What breaks if its password or contents are wrong?**
- Why the interviewer asks: Operations impact of a committed `.p12`.
- Ideal answer: A wrong password gives an exception at bean creation (`RestTemplateConfig`, `PlineRestTemplateConfig`) and the app does not start. A CA missing from the truststore does not stop startup but breaks outbound TLS calls: the custom truststore replaces the JVM default `cacerts` (inferred from `SSLContexts.custom().loadTrustMaterial(trustStore, null)`). The password sits in plaintext in `application-env.properties` (<redacted>), and gitleaks allowlists that file.
- Common mistakes: Assuming Mongo uses this truststore (it uses the JVM default trust).
- Follow-up questions: How would you rotate the truststore without a rebuild? (It is in the jar; a rebuild is needed unless overridden - inferred.)

**Q18. What happens if someone removes `spring.http.converters.preferred-json-mapper=jackson2`?**
- Why the interviewer asks: Checks the Jackson 2 vs 3 knowledge.
- Ideal answer: Boot 4.1.1 prefers Jackson 3 (`tools.jackson`), which ignores the `com.fasterxml.jackson` annotations used by `PayLoadRequest` and the DTOs. POST requests would fail with message-conversion errors (commit `9248b12ff`, comments in lines 7-16 of `application.properties`). So the property must stay until the annotations are migrated.
- Common mistakes: Removing "unused" properties during cleanup.
- Follow-up questions: Does the XML side use Jackson 2? (Yes, `com.fasterxml.jackson.dataformat.xml.XmlMapper` in `CommonController`.)

**Q19. A tester calls `/api/opus/GenerateReport` and gets 404. Why?**
- Why the interviewer asks: Context path knowledge.
- Ideal answer: `server.servlet.context-path=/gi/documentprintdata`. The full URL is `/gi/documentprintdata/api/opus/GenerateReport`. The Helm probes and the ingress also need the prefix. I check the URL first, then the HTTP method (POST only).
- Common mistakes: Thinking the controller mapping is wrong.
- Follow-up questions: What other paths carry the prefix? (actuator, swagger, api-docs.)

**Q20. Reports are slow. Which timeouts exist and which do not?**
- Why the interviewer asks: Tests whether you know the real limits.
- Ideal answer: Outbound: connect 5 s, read/socket 20 s, connection-request 3 s, pool 100 total and 20 per route; fast-lookup read 8 s (`app.http.*`). Inbound: none configured. Mongo: `maxTime` 600 s and `allowDiskUse` only in handlers that set `AggregationOptions` (for example `PlinePolicyDocumentMainHandler`, `AspDebitNoteHandler`); not in `RejectRiskHandler` or `AspEndtDocumentMainHandler`. So a 20 s caller timeout can leave a 10 minute query running (inferred).
- Common mistakes: Saying "all calls time out at 20 s".
- Follow-up questions: How many external calls can one Template12 request make? (About 5 distinct utility calls - inference.)

**Q21. The `option.min-connection-pool-size` etc. keys exist. Do they tune the pool?**
- Why the interviewer asks: Catches "config that looks active".
- Ideal answer: No. They are not under `spring.mongodb.*`, nobody reads them with `@Value` (KT 06). The driver defaults apply (maxPoolSize 100, minPoolSize 0). To change the pool, add URI options, for example `maxPoolSize` (inferred; verify in Boot 4).
- Common mistakes: Reporting "pool is 100/20 as configured".
- Follow-up questions: Same question for `auto-index-creation`. (Also ineffective; no `@Document` entities.)

**Q22. What is risky about `MongoTemplate` DEBUG logging in production?**
- Why the interviewer asks: Operational hygiene and PDPA.
- Ideal answer: `logging.level.org.springframework.data.mongodb.core.MongoTemplate=DEBUG` logs pipelines including values such as contract IDs and, for RejectRisk, customer names and ID numbers. It also costs CPU and disk. `CustomRequestFilter` prints whole request bodies with `System.out`, which bypasses log levels. The `loggers` actuator endpoint is enabled but not exposed, so levels cannot be changed at runtime.
- Common mistakes: Saying "turn it off with the loggers endpoint".
- Follow-up questions: Which stale logging keys target non-existent packages? (`com.agcs.fo.opusMotorGateway.*`.)

**Q23. A developer's `application.properties` contains a Mongo URI with credentials. What do you do?**
- Why the interviewer asks: Secret handling judgement.
- Ideal answer: Never commit it. The current working copy has an uncommitted block with a plaintext user and password (<redacted>). Use env vars (`SPRING_MONGODB_URI`) or `--spring.mongodb.uri` locally. Check `git status` before every commit. If it was ever pushed, rotate the password and tell security (the existing tracker for related findings is OPUSM-50404). The repo runs `secret-scan.yml` on every PR and a nightly full-history scan.
- Common mistakes: Deleting the line in a new commit and thinking history is clean.
- Follow-up questions: Why is the gitleaks allowlist a risk? (It exempts two whole files; review-by date 2026-10-23.)

**Q24. The gitleaks allowlist has a review-by date of 2026-10-23. What does that mean for support?**
- Why the interviewer asks: Awareness of security debt.
- Ideal answer: `gitleaks.toml` allowlists whole files `application.properties` and `application-env.properties` (ticket OPUSM-50404). Any new secret in those files passes the scan. The date is a reminder to review. The right fix is to move secrets out of those files and remove the allowlist. I would raise it, not remove it alone, because removing it will make the scan fail on existing content (inferred).
- Common mistakes: Treating the date as an expiry that auto-fixes itself.
- Follow-up questions: Where should secrets live? (Secret store or environment of the deployment.)

---

## C. Business-flag tickets (missing pages)

**Q25. "Service tax invoice is not printing for an ASP endorsement." Walk through the gate.**
- Why the interviewer asks: The most typical ASP ticket.
- Ideal answer: In `AspEndtDocumentService` (~line 335) the ServiceTaxInvoice block needs all four: `showTaxInvoice=Y`, `showStaxInvInd=Y` (`SHOW_STAX_INV_IND`), `checklistCopy != Y`, and `showEinvInd=Y` (`SHOW_EINV_IND`, from the REST call `fnEinvoicePrintEndtInd`). Test `showServiceTax_checklistCopyY_closesGate` locks the checklist rule. If the REST call failed it returns "" and the gate closes silently. The QR section needs another three: `einvConsentInd=Y`, `einvStatus=VALID`, `qrInd=Y`. EndtMT/EndtMTSub rows need `staxCount > 1` and `ENDT_CODE != 999`.
- Common mistakes: Checking only `showTaxInvoice`.
- Follow-up questions: Where do the flags come from? (`AspEndtDocument2MainHandler`: `UWGE_VI_ENDT_REPORT_PARAM`, plus REST.) What was the CONTRACT_ID type bug?

**Q26. "Debit note is missing for an ASP endorsement."**
- Why the interviewer asks: Another standard gate.
- Ideal answer: Needs `showDebitNote=Y` and `prem != 0`, where PREM comes from `DIFF_PREM_DUE != 0` (test `showDebitNote_withZeroPrem_doesNotFetchDebitNotes`). `AspDebitNoteHandler` reads `UWGE_VI_ENDT_DEBIT`, matches on `ENDT_NO` or `REF_NO` with `$or`. Note number is `D`/`C` plus `ENDT_NO` by sign of `PREM_DUE`. If `PREM_BY_MST_IND=Y` the master customer block is added (a chain through `MST_POL_NO` and `OCP_POLICY_BASES`). A zero-premium endorsement correctly has no debit note.
- Common mistakes: Calling zero-premium "a bug".
- Follow-up questions: How does the Java gate differ from Jasper for a missing PB row? (inferred: `$ne` on a missing field is true, so PREM may become 1.)

**Q27. "Template12 debit note is missing."**
- Why the interviewer asks: Tests Template12 PD2 rules.
- Ideal answer: In the merged PD2 block the debit note needs `showDebitNote=Y`, scheme not BMW/BMW1 (their `PREM` is 0), and `copyType != "BC"` (`S12:633`). For product 070406 it uses the Template4 handler. Check `copyType` first; callers sometimes send BC for copies.
- Common mistakes: Looking in ASP code.
- Follow-up questions: Which test covers the BMW skip? (`execute_pd2DebitNoteSkippedForBmwScheme`.)

**Q28. "Template12 landscape pages are missing for a policy."**
- Why the interviewer asks: Landscape dispatch is a common source of "nothing prints".
- Ideal answer: `Template12LandscapeDispatchHandler.fetchDispatch` returns null if the scheme is excluded by `CMGE_CODEREL_DET` (`GPA_SCHEME`, `CI_LANDSCAPE_PRT`, `Y`), and then nothing prints. Otherwise each of the 16 bands has its own gate (product/scheme; `BENEFIT_PRINT`, `NOMINEE_PRINT`, `DIAGNOSIS_PRINT` counts; ASP bands need 070412 and `ASP_CUTOFF_IND=N`). For ASP with cutoff `Y` the ASP pages are portrait pages inside CIPrint instead.
- Common mistakes: Assuming landscape is a separate report call.
- Follow-up questions: Which flags inspect only the first joined version row? (`NOMINEE_PRINT`, `DIAGNOSIS_PRINT` - inferred risk.)

**Q29. "ASP landscape pages for added members are missing."**
- Why the interviewer asks: Needs the dispatch counts logic.
- Ideal answer: `AspAddMemberLandscapeDispatchHandler` returns `additionPrint`, `deletionPrint`, `nomineePrint`, `sponsorPrint`. Addition needs `ENDT_CODE` in (72, 168) and at least one risk with `ACTION_CODE=A`. Deletion needs `ENDT_CODE` 98 (and delete rows `ACTION_CODE=C`, `TERMINATE_DATE` not null). If count is 0 the insured/premium reportData rows are not added. The dispatch row itself is always added, even with zeros.
- Common mistakes: Expecting a page when the endorsement code is something else.
- Follow-up questions: Why do dispatch queries run on every call? (Landscape flows are unconditional - a known cost.)

**Q30. "ASP schedule of benefits page is missing for endorsement 72."**
- Why the interviewer asks: Tests knowledge of stubs.
- Ideal answer: `PlineEndtScheduleOfBenefitsAspHandler`, `PlineAspLlbHandler`, `PlineAspStiHandler` are stubs (they return a dual value, nothing is mapped). `AspEndtDocumentService` calls them and discards the result. So no `reportData` is emitted for those pages. Only the master row's `actionCode` (ENDT_CODE) is available to the renderer; whether the renderer uses it is not known. So this is a known gap, not a data problem.
- Common mistakes: Searching Mongo for the missing data.
- Follow-up questions: How would you confirm with the report team?

**Q31. "ComplaintRedress is missing."**
- Why the interviewer asks: Differences between Template12 and ASP rules.
- Ideal answer: ASP: printed only for product `070412` (not by `showComplaint`); the handler result is discarded and the row has an empty transDet plus company name/BRN. Template12: plain version when `showComplaint=Y` and not BMW/070504/070406 and (not ASP or `ASP_CUTOFF_IND=N`); SCB version for 070504; for POSMESRA/FWPA the SCB is printed from PD2. Always check product, scheme and `ASP_CUTOFF_IND` first.
- Common mistakes: Sending `showComplaint=Y` and expecting a page for every product.
- Follow-up questions: Where does `ASP_CUTOFF_IND` come from? (`SAPM_SYS_CONSTANTS` `ASP_CUTOFF_DATE` vs issue date - see 02.)

**Q32. "QR code section missing on the e-invoice."**
- Why the interviewer asks: Combination of data and API flags.
- Ideal answer: ASP: `einvConsentInd=Y` and `einvStatus=VALID` and `qrInd=Y`; `qrInd` from `PlineEinvoiceQrIndicatorClient` which defaults to `N` on failure (`safeCall`). Template12: `addGenericQRSection(contractId, versionNo, pTaxInvNo, rd)` uses the request `pTaxInvNo` (default ""), but `PlineGenericQRSectionHandler` filters on `STAX_INV_NO`, so unless the caller sends the invoice number the section is probably empty (inferred, KT 02 D14).
- Common mistakes: Only checking the e-invoice flag.
- Follow-up questions: How would you fix the Template12 case? (Use the number from the parent row.)

**Q33. "Preview (checklist) copy shows different numbers than the final copy."**
- Why the interviewer asks: WIP vs live understanding.
- Ideal answer: `checklistCopy=Y` makes handlers read `WIP_*` collections; `N` reads posted/live. Template12 switches collections in Java (`isWip ? WIP_x : x`, live base query uses `VERSION_NO = 1`). ASP main handler uses a `$$chk` switch inside the `$lookup`. The service-tax invoice is excluded for `checklistCopy=Y`. The REST function `stax-inv` is called with mode `WIP` for checklist and `OCP` otherwise. Differences are expected; the question is whether the WIP data changed before posting.
- Common mistakes: Treating differences as a bug without checking the flag.
- Follow-up questions: What is `SB_`? (Not documented in code - say so.)

**Q34. What is the difference in the support answer for "GST invoice" vs "Service tax invoice" vs "Transitional invoice" in ASP?**
- Why the interviewer asks: Three similar gates.
- Ideal answer: GST: `showTaxInvoice=Y` and `SHOW_GST_INV_IND=Y` (EndtTaxDoc plus AntiRasuah banner). Service tax (SST e-invoice): the four-part gate in Q25. Transitional: `showTaxInvoice=Y` and `SHOW_TRAN_INV_IND=Y` (TransitionalInvoice and TransitionalInvoice2). All also need `showTaxInvoice=Y` from the caller. The indicator values come from the report-param view and REST functions.
- Common mistakes: Mixing them in one rule.
- Follow-up questions: Which one depends on the SST rollout date? (Service-tax handler via `PlineSSTRolloutDateClient`.)

**Q35. "The tax invoice should print, indicators say `Y` in Oracle, but the service says `N`."**
- Why the interviewer asks: Cross-system consistency.
- Ideal answer: The service does not read Oracle directly. It reads the Mongo mirror and REST functions. So I compare three things: the mirrored `UWGE_VI_ENDT_REPORT_PARAM` / `UWGE_VI_POL_REPORT_PARAM` values, the REST function result (call it directly in dev), and the type of `CONTRACT_ID` in the query. If the mirror is behind Oracle the answer differs (inferred; mirror lag is not documented in the repo). [CONFIRM: how the Oracle-to-Mongo sync works and its delay.]
- Common mistakes: Assuming Mongo and Oracle are always identical.
- Follow-up questions: How do you prove a type mismatch? (Compare `$type` of `CONTRACT_ID` in Mongo with the Java literal.)

---

## D. RejectRisk tickets

**Q36. "RejectRisk misses rows created on the last day of the range."**
- Why the interviewer asks: Core RejectRisk bug.
- Ideal answer: HEAD used `CREATED_DATE $gte from, $lte to` with `to` at midnight, while `CREATED_DATE` has a time part, so most of the last day was lost; `from = to` returned only exact-midnight rows. The uncommitted fix shifts `dateTo` by +1 day and uses `$lt`, equal to `TRUNC(CREATED_DATE) BETWEEN`. It equals commit `cc6a1b84d` on the other branch. Check which build runs. The time-zone caveat: the code builds UTC midnight; if data is stored as real instants, a Malaysia (UTC+8) record near midnight can fall in the previous UTC day (inferred, not verified).
- Common mistakes: Saying `$lte` with end-of-day is the same thing.
- Follow-up questions: Why not `$dateTrunc`? (It blocks index use.)

**Q37. "RejectRisk now returns everything when no dates are sent."**
- Why the interviewer asks: Behaviour change risk.
- Ideal answer: After the fix, only supplied bounds are added and if neither is given no date filter is added. Oracle returned zero rows for a null `BETWEEN`; HEAD also returned zero (by accident - null bounds match only null dates). So the fix changes behaviour: possible huge unbounded result, full scan and `$setWindowFields` sort in memory (100 MB limit, no `allowDiskUse` set). It needs a business decision: keep parity (empty) or require a date. I would escalate before release.
- Common mistakes: Calling it "obviously correct".
- Follow-up questions: What guard would you add? (Mandatory date range or row cap.)

**Q38. The RejectRisk fix is uncommitted in the working tree. Is it safe to ship?**
- Why the interviewer asks: Judgement about tests.
- Ideal answer: Not yet. Static reading says 7 of the 15 `RejectRiskHandlerTest` tests will fail: `..._dateInstances_usedDirectlyInMatch`, `..._dateStrings_parsedViaDateUtils`, `..._nullDates_matchStageHasNullBounds`, `..._rejTypeOnly_addsSecondMatchStage`, `..._reasonCodeOnly_addsSecondMatchStage`, `..._rejTypeAndReasonCode_bothInSameMatchStage`, `..._customerNameAndIdNoAbsent_noExtraMatchStages` (not run - nobody has run Maven here). They assert `$lte`, null bounds and fixed stage indexes. Also the uncommitted `application.properties` must not be included. And the time zone and the "no dates" rule need sign-off.
- Common mistakes: Saying "tests are green" without running them.
- Follow-up questions: How should the tests be rewritten? (Find the stage by content, not index.)

**Q39. "The order of customers differs from the old Jasper report; rows without a name come first."**
- Why the interviewer asks: Oracle vs Mongo semantic difference.
- Ideal answer: Oracle `ORDER BY NAME_EXT ASC` puts NULLs last; Mongo ascending puts null and missing first. Customers without a partner match or with null `NAME_EXT` go to the top and get `rowNumber` 1..k (`$setWindowFields` with `$documentNumber`). The test `fetchRejectRisks_finalSortStage_sortsByCustomerNameAscending` locks the plain sort, so it is not caught. Fix: add a null-flag sort key, or sort in Java with `nullsLast`. Ties are also not deterministic.
- Common mistakes: Saying "sort is the same in both".
- Follow-up questions: Which column does Jasper's NO column actually print? (`GroupByBranch_COUNT`, not `ROWNUMBER`.)

**Q40. "Searching RejectRisk by customer name returns nothing, but the name exists."**
- Why the interviewer asks: Exact-match and filter-order behaviour.
- Ideal answer: `CUST.NAME_EXT` is matched exactly and is case sensitive (same as Oracle `=`; inferred default NLS). No trim. Also the filter runs after the `$lookup` into `CPGE_VI_PARTNERS_ALL` (`$limit 1`, no sort), so it matches only the first partner row. If the view has more than one row for (PART_ID, VERSION), the first may differ. Empty string is treated as no filter, same as Oracle.
- Common mistakes: Adding a regex without discussing the index and semantics change.
- Follow-up questions: Why is a name-only search slow? (The `$lookup` runs for every date-matched row.)

**Q41. "RejectRisk is slow or times out for a big date range."**
- Why the interviewer asks: Performance reasoning.
- Ideal answer: Per matched document there are four sub-pipeline lookups: one `CPGE_VI_PARTNERS_ALL` and three `CMGE_CODE` (`PLREJ_RISK_TYPE`, `PLREJ_REASON`, `PLREJ_REVOKE_DUR`). Without indexes on `CMGE_CODE (CAT_CODE, CODE_CD)` and `CPGE_VI_PARTNERS_ALL (PART_ID, VERSION)` it scales badly. Whether indexes exist is unknown (`auto-index-creation` has no effect). No pagination, full result is read into the heap, no `allowDiskUse`. Check with the `MongoTemplate` DEBUG pipeline and `explain("executionStats")`. [CONFIRM: real cluster indexes.]
- Common mistakes: Adding indexes in Java code.
- Follow-up questions: How would you cut the three code lookups? (Preload a map.)

**Q42. "The RejectRisk date shown in the PDF is one day off."**
- Why the interviewer asks: Time zone and date conversion.
- Ideal answer: Two places. Filter: UTC midnight window (see Q36). Display: DTO dates are `String`; Spring converts the BSON `Date` (inferred `Date.toString()` style, JVM default zone) and `DateUtils` parses it; the service's `toLocalDateTime` swallows errors and returns null. So the shown date depends on the JVM zone, and a parse failure silently drops the date. Better: project `$dateToString` with `timezone: Asia/Kuala_Lumpur` (pattern used in other handlers).
- Common mistakes: Blaming the renderer first.
- Follow-up questions: What formats does `DateUtils.toLocalDate` accept?

**Q43. "RejectRisk shows no header (title and data classification) when there are no rows."**
- Why the interviewer asks: Empty-result behaviour is a known difference.
- Ideal answer: The ReportHeader `reportData` is added only if there is at least one row (unit test `execute_emptyRows_returnsEmptyReportDataList`). Jasper still printed the title, DATA_CLASS line and "*** No Data Found ***" (`whenNoDataType=AllSectionsNoDetail`). So an empty list is a difference to confirm with the rendering team. Also DATA_CLASS is null if `RPGE_RPT_TEMPLATE` has no row for (`pReportTitle`, `reportName`).
- Common mistakes: Thinking it is a Mongo failure.
- Follow-up questions: How is `dataClass` looked up? (`PlineReportHeaderHandler`, case-insensitive regex with `Pattern.quote`.)

**Q44. "The date filter returns 500 when the caller sends 31.12.2026."**
- Why the interviewer asks: Input format handling.
- Ideal answer: `DateUtils.toLocalDate` accepts `yyyy-MM-dd`, `dd-MM-yyyy`, `dd/MM/yyyy`, `MM/dd/yyyy` (only if dd/MM fails), `dd-MMM-yyyy`, `dd-MMM-yy`, `EEE MMM dd HH:mm:ss z yyyy`, and ISO offset/zoned. Dots are not accepted, and a local date-time without offset (`2026-01-05T10:00:00`) is also not accepted. The handler's `toDate` then throws and the request is 500 (an `IllegalArgumentException` would be 400, but ISO-shaped bad values throw `DateTimeParseException`).
- Common mistakes: Assuming any date string works.
- Follow-up questions: What does "03/04/2026" mean? (3 April, because dd/MM is tried first.)

---

## E. Data-quality and mirror issues

**Q45. A code description is blank for one risk type. Where do you look?**
- Why the interviewer asks: Data-vs-code split.
- Ideal answer: Descriptions come from `CMGE_CODE` with `CAT_CODE` `PLREJ_RISK_TYPE`, `PLREJ_REASON`, `PLREJ_REVOKE_DUR` and `CODE_CD` equal to the stored code. `$lookup` + `$limit 1` + `$arrayElemAt` keeps the row with a null description if the code is missing (LEFT JOIN). So it is usually missing master data, or a type mismatch between `REJ_TYPE` and `CODE_CD`. Note: Oracle would raise ORA-01427 for duplicate codes; Mongo silently takes the first.
- Common mistakes: Changing the Java code to hide nulls.
- Follow-up questions: In Jasper, what prints when the description is null? ("R1 / null" for reason - inferred from Groovy concatenation.)

**Q46. Why can the same contract behave differently across Template12 PolicyDocument, PD2 and Landscape?**
- Why the interviewer asks: Tests the "three primary fetches" fact.
- Ideal answer: The PolicyDocument primary fetch emulates Oracle: it throws on WIP/live conflict. PD2 (`PlineT12PolicyDocumentHandler`) prefers posted data, falls back to WIP using `$ifNull`, no conflict check, and calls REST for `EINV_PRINT_IND` and `SHOW_STAX_INV_IND`. Landscape dispatch uses product = WIP, else SB, else `UWGE_POLICY_CONTRACTS`. They were not unified on purpose (CRITICAL FACT #1 in the service javadoc) because unifying would change PD2 behaviour without product sign-off. So a contract with inconsistent data fails PolicyDocument but still prints PD2 and Landscape.
- Common mistakes: Calling it a copy-paste bug.
- Follow-up questions: What are the trade-offs? (Three fetches can disagree.)

**Q47. A report shows "PRODUCT" or scheme-driven pages wrong for a contract. What is your data check?**
- Why the interviewer asks: Data-driven branching.
- Ideal answer: Find the product code and scheme in all places the three fetches read: `WIP_UWGE_POLICY_CONTRACTS`, `SB_UWGE_POLICY_CONTRACTS`, `UWGE_POLICY_CONTRACTS`; scheme in `WIP_UWPL_POLICY_BASES` and `UWPL_POLICY_BASES` (`TOP_INDICATOR='Y'` for PolicyDocument; `VERSION_NO=1` plus WIP-not-in-OCP for PD2/Landscape). If they disagree, the pages disagree. Also check the types of `CONTRACT_ID`.
- Common mistakes: Reading only one collection.
- Follow-up questions: What is `OCP_*`? (Not documented in the code I read.)

**Q48. "Master policy holder details are missing on the ASP debit note."**
- Why the interviewer asks: MST chain knowledge.
- Ideal answer: `AspDebitNoteHandler.fetchMstContractInfo` runs 4-5 round trips: `MST_POL_NO` from `UWGE_POLICY_BASES`/`WIP_UWGE_POLICY_BASES`, then `OCP_POLICY_BASES.POLICY_REF $in`, then master `CONTRACT_ID`, then `PREM_BY_MST_IND`. Any exception is swallowed and `premByMstInd` becomes "N" (handler ~lines 181-183), so the master block silently disappears. Resolution may run up to 3 times per request.
- Common mistakes: Not seeing the swallowed exception.
- Follow-up questions: Why is it multiple queries? (Mirrors Oracle nested sub-selects; simple but more round trips.)

**Q49. "Rounded amounts are off by one cent versus Oracle."**
- Why the interviewer asks: Numeric semantics.
- Ideal answer: The ROUNDED_NETT_PREM in `AspDebitNoteHandler` uses Mongo `$round`, which rounds half to even; Oracle `ROUND` rounds half away from zero. It only matters on exact .5 boundaries (inferred from documented behaviour, not tested here). Check whether the value is exactly at a half before concluding.
- Common mistakes: Blaming BigDecimal scale in the XML.
- Follow-up questions: How would you fix it? (Compute in Java with `RoundingMode.HALF_UP`, or an equivalent expression.)

**Q50. "Insured members appear in the wrong order on the landscape page."**
- Why the interviewer asks: Sort and type details.
- Ideal answer: Jasper orders by `cast(COV_SEQ_REF as decimal(15,2))`. Java uses `$toDouble(COV_SEQ_REF)` in several handlers, but `PlineT12ParticularOfInsuredLandscapeAmcHandler`, `PlineT12ParticularOfDiagnosisLandscapeHandler` and the diagnosis AMC handler sort the raw field. If stored as a string, "10" sorts before "2" (inferred). Null sequence sorts first in Mongo, last in Oracle. Sponsor, nominee and delete-member have no ORDER BY in either system.
- Common mistakes: Assuming Mongo sorts numerically.
- Follow-up questions: What happens if `$toDouble` meets a non-numeric string? (It throws, like the Oracle cast would.)
