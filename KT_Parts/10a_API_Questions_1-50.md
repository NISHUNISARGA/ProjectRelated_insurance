# 10a - API Questions 1-50 (opus-documentprintdata-service)
> **AUTHORSHIP UPDATE (confirmed by the developer): git author `GS <extern.gs_nisarga@allianz.com>` is the developer. Wherever this file says commits cannot be attributed to the developer, read it as: the 11 GS commits listed in `KT_Parts/14_My_Commits_And_Ownership.md` ARE theirs; commits by other authors still must not be claimed. No GS commit touches `src/test`, so test authorship is still `[CONFIRM]`.**

Evidence base: `06_Config_API_DB_DevOps.md` (Part 2 and 5.B), `04_XSD_XML_Deep_Dive.md`, and repo files `CommonController.java`, `GlobalExceptionHandler.java`, `GenericExceptionHandler.java`, `SecurityConfig.java`, `PlineUtilsClient.java`. "(inferred)" = my reasoning, not proven in the repo. Personal facts are written as [CONFIRM: ...]. Secrets are never printed.

---

### Q1. The whole service has one business endpoint. What is it, and why might the team choose one POST for every report?
- **Why the interviewer asks:** To see if you understand the real shape of the service and not a generic REST picture.
- **Ideal answer:** The business endpoint is `POST /gi/documentprintdata/api/opus/GenerateReport` (context path `/gi/documentprintdata` from `application.properties`, mapping from `CommonController`). The report is chosen by `filterCondition.reportName`, not by URL. The caller is a reporting layer that already has one "call data provider, get XML" contract for every report across LOBs (inferred). One URL means one ingress rule, one probe path and one contract; adding a report is adding a bean, not a route. The cost: no per-report URL metrics, no per-report auth, and a weak contract (everything is a `Map<String,String>`).
- **Common mistakes:** Saying "one controller per report". Forgetting the context path. Saying it returns the PDF.
- **Follow-up questions:** What is the other GET on the controller (`/getJSON`)? How would you get per-report metrics with one URL?

### Q2. Write the exact URL and body shape a caller sends for RejectRisk.
- **Why the interviewer asks:** Checks you have actually used the API, not only read about it.
- **Ideal answer:** `POST http://<host>:8080/gi/documentprintdata/api/opus/GenerateReport`, `Content-Type: application/json`. Body has `metadata` (name, email, outputFormat, channel, dataClass, templateId, reportName, mergeStaticPdfFiles...) and `filterCondition` (`reportName` = `AGIC/UW/Plines/RejectRisk/RejectRisk`, `pReportTitle`, `pCreatedDateFrom`, `pCreatedDateTo`, `pCustomerName`, `pIdNo`, `pRejType`, `pReasonCode`). All RejectRisk filters are optional; a missing one simply does not filter.
- **Common mistakes:** Putting `reportName` only in `metadata` (the controller reads it from `filterCondition`). Sending numbers or arrays inside `filterCondition`.
- **Follow-up questions:** What happens if `pCreatedDateFrom` is missing? (No lower bound; see DB question Q7 in 10c.) Which date formats are accepted? (Whatever `plines.utils.DateUtils.toLocalDate` parses; check the class.)

### Q3. How does the controller pick the code that serves a given report?
- **Why the interviewer asks:** Strategy lookup is the heart of the design; interviewers want the exact mechanics.
- **Ideal answer:** The controller injects a `Map<String, ReportStrategy>` (Spring fills it with bean name -> bean). It normalises `reportName` and calls `reportStrategies.get(reportName)`. If a strategy exists it runs `strategy.execute(requestBody)` and returns. Otherwise it falls to a legacy `switch(reportName)` that calls older per-LOB services. My modules are `@Service("AGIC/UW/Plines/RejectRisk/RejectRisk")`, `@Service("AGIC/UW/Plines/ASPEndtSchedule/EndtDocument")` and `@Service("AGIC/UW/Plines/PolicySchedule/Template12/PolicyDocument")`.
- **Common mistakes:** Thinking there is `@RequestMapping` per report. Not knowing that the bean name IS the report name.
- **Follow-up questions:** What if two beans have the same name? (Startup failure or override depending on Boot settings; do not guess in the interview.) Where is the `ReportStrategy` interface? (`motor/strategy/ReportStrategy.java`)

### Q4. What exactly does the controller do to `reportName` before the lookup?
- **Why the interviewer asks:** Small normalisation rules cause real "report not found" bugs.
- **Ideal answer:** It strips one leading `/`, then strips a leading `reports/` prefix (case-insensitive on the prefix check). So `/reports/AGIC/...`, `reports/AGIC/...` and `AGIC/...` map to the same key. The remaining string is matched exactly (inferred from `reportStrategies.get(reportName)`; there is no `toLowerCase` on the lookup key), so `agic/uw/...` would not match a bean named `AGIC/UW/...`.
- **Common mistakes:** Assuming the whole name is case-insensitive. Assuming a trailing slash or spaces are trimmed.
- **Follow-up questions:** How would you test the normalisation? Why would JasperServer paths carry a `/reports/` prefix? (JasperServer repository paths; inferred.)

### Q5. What happens when a caller sends a report name that no bean or switch case knows?
- **Why the interviewer asks:** This is a silent-failure trap; seniors must know it.
- **Ideal answer:** It does not fail. The legacy `switch` has no matching case, `transactions` stays empty, the controller builds a default transaction (mailing + template details only) and returns HTTP 200 with `status: SUCCESS` and an XML with no report rows. So a typo in `reportName` looks like "no data". Only a `null` reportName fails (`switch(null)` NPE -> 500).
- **Common mistakes:** Expecting 404 or 400. Saying "Spring returns 404 for unknown routes" (the route exists; the dispatch is by data).
- **Follow-up questions:** How would you make it fail loudly? (Throw an `IllegalArgumentException` in a switch `default` -> 400 via `GenericExceptionHandler`.) What would the print engine do with an empty XML?

### Q6. The brief mentions PolicyDocument2, Landscape, EndtDocument2, AddMemberLandscape and DeleteMemberLandscape. Can I call them directly?
- **Why the interviewer asks:** Tests whether you know the actual deployed structure versus the Jira wording.
- **Ideal answer:** Per class javadoc and the KT guide, each LOB module has ONE live bean: Template12 `PolicyDocument` and ASP `EndtDocument`. The other names were merged into them (they are sub-reports / branches decided by data and `show*` flags). They are not registered strategy names, so a caller sending those names falls to the legacy switch and gets the default empty transaction (see Q5).
- **Common mistakes:** Claiming five or seven separate endpoints/beans.
- **Follow-up questions:** Why merge them? (One request produces one document with several sub-reports, like the original JasperServer report with sub-reports; inferred.) How does the print engine know which sub-report a block is? (`mainReportName` / `subReportName` tags.)

### Q7. How do you add a brand-new report to this API?
- **Why the interviewer asks:** Shows you understand the extension point and the full change list.
- **Ideal answer:** Create a `@Service("<exact JasperServer-style report name>")` class implementing `ReportStrategy`, with `execute(PayLoadRequest)` returning `Transactions`. Handlers (Mongo) and DTOs sit beside it. No controller change. Then: XSD element mapping (if new fields), unit tests with mocked `MongoTemplate`, and give the caller the exact bean name. Do not add to the legacy switch ("All NEW reports added going forward must follow this path" - controller comment).
- **Common mistakes:** Adding another `case` to the switch. Forgetting that the bean name must equal the caller's `reportName` byte for byte.
- **Follow-up questions:** How would you protect against duplicate names? How would you list all registered report names at runtime? (Not exposed today; actuator exposes only health/info.)

### Q8. Describe the success response contract.
- **Why the interviewer asks:** The consumer depends on this exact envelope.
- **Ideal answer:** `{"status":"SUCCESS","data":"<XML string>","timestamp":"<LocalDateTime.toString()>"}` (class `SuccessResponse`). `data` is the XML of `Transactions` written with `new XmlMapper().writeValueAsString(transactions)`, escaped inside the JSON string. Always HTTP 200 and `application/json`. `timestamp` has no time-zone (LocalDateTime).
- **Common mistakes:** Saying the body is XML or a PDF. Expecting JSON rows.
- **Follow-up questions:** Why wrap XML in JSON? (Existing OPUS caller contract; inferred.) What is the downside? (Double escaping, large strings, no streaming.)

### Q9. When does the API return 404, 400 and 500?
- **Why the interviewer asks:** Error contract knowledge; also tests whether you know what is NOT mapped.
- **Ideal answer:** 404: only when code throws `NoDataFoundException` (`GlobalExceptionHandler.handleNoData`, errorCode `NO_DATA`). My three modules normally return an empty but valid result instead. 400: any `IllegalArgumentException` anywhere in the app (`GenericExceptionHandler`, errorCode `BAD_REQUEST`, generic detail "Invalid request parameters"). 500: any other `Exception` (`handleGeneric`, errorCode `INTERNAL_ERROR`, detail "An unexpected error occurred"). All three return Spring `ProblemDetail` plus `errorCode` and `timestamp`.
- **Common mistakes:** Saying "no rows = 404". Saying validation errors give 422.
- **Follow-up questions:** Which handler wins for `IllegalArgumentException` when a catch-all `Exception` handler exists? (The more specific exception type; the javadoc in `GenericExceptionHandler` says so.)

### Q10. The 500 handler hides the message. Is that good or bad?
- **Why the interviewer asks:** Balanced judgement on security versus operability.
- **Ideal answer:** Good for security: the client never sees stack traces or Mongo text. Bad for operations: `GlobalExceptionHandler` has no logger and does not log the exception, so the root cause is invisible unless the thrower logged it. I would add `log.error("...", ex)` with a correlation id and keep the generic client message.
- **Common mistakes:** Saying "Spring logs it automatically" (not for exceptions handled by an `@ExceptionHandler`).
- **Follow-up questions:** What would you add to the ProblemDetail? (`errorCode`, a request id.) Where would the request id come from? (None exists today; add a filter/MDC.)

### Q11. What happens with malformed JSON or an unsupported HTTP method?
- **Why the interviewer asks:** Tests understanding of how a catch-all `@ExceptionHandler(Exception.class)` interacts with Spring's built-in resolvers.
- **Ideal answer:** (inferred, not tested) Because `handleGeneric` catches `Exception`, an `HttpMessageNotReadableException` or `HttpRequestMethodNotSupportedException` probably ends up as HTTP 500 with `INTERNAL_ERROR`, not 400/405. Spring's `ExceptionHandlerExceptionResolver` runs before `DefaultHandlerExceptionResolver`. I would verify with a curl test and then add specific handlers (400 for unreadable body, 405 for method).
- **Common mistakes:** Stating it as fact without saying it is untested. Saying Spring always returns 400 for bad JSON.
- **Follow-up questions:** How would you write that test? (`MockMvc` post with `{`.) What status should a client get?

### Q12. What if the request has no `metadata` block?
- **Why the interviewer asks:** Null-safety in the controller.
- **Ideal answer:** The controller calls `requestBody.getMetadata().get(...)` in `populateMailingDetails` / `populateTemplateDetails` without a null check, so a missing `metadata` gives an NPE -> 500 (inferred from code reading, not executed). The same shape applies to a missing `filterCondition`: the first line reads `requestBody.getFilterCondition().get("reportName")`.
- **Common mistakes:** Saying defaults are applied. Saying Bean Validation rejects it (there is no `@Valid`).
- **Follow-up questions:** Where would you put validation? (`@Valid` + `@NotNull` on `PayLoadRequest`, or a request pre-check throwing `IllegalArgumentException`.) Does `spring-boot-starter-validation` exist on the classpath? (Yes, unused for this DTO.)

### Q13. How does `PayLoadRequest` treat metadata values?
- **Why the interviewer asks:** Hidden type coercion rules cause "my field disappeared" bugs.
- **Ideal answer:** `setMetadata(Map)` is custom. It keeps String and Number values (numbers converted with `toString()`), lifts a `mergeStaticPdfFiles` list found inside metadata into the dedicated field, and silently drops booleans, nulls and nested objects. So `"disallowCopy": true` (boolean) vanishes; `"disallowCopy": "N"` stays.
- **Common mistakes:** Sending booleans. Expecting an error for dropped fields.
- **Follow-up questions:** Why not fail on unknown types? What is `parseIntSafe` used for? (Integer metadata like `batchConfigId`, `boResId`, `templateId` return null on bad input.)

### Q14. `filterCondition` is `Map<String,String>`. What does that imply for callers and for security?
- **Why the interviewer asks:** Links API typing to injection safety and to contract fragility.
- **Ideal answer:** Every filter value must be a JSON string. A number or array makes Jackson fail (500 given Q11), except `pBranch`/`pBranchName` arrays, which `CustomRequestFilter` joins into a comma string before binding. Security side effect: because values are always Strings, a caller cannot inject a Mongo operator object such as `{"$ne":""}` into a `Document` filter built by the handlers.
- **Common mistakes:** Saying the map is validated. Missing the `CustomRequestFilter` special case.
- **Follow-up questions:** What if a value is too long or contains regex characters? (Equality match only, so harmless for Mongo, but sanitised in outbound calls.) How would a typed DTO per report help?

### Q15. What does `CustomRequestFilter` do and what is wrong with it?
- **Why the interviewer asks:** Cross-cutting concerns: filters, PII logging, robustness.
- **Ideal answer:** It is a servlet `Filter` on every request. It wraps the request, reads the body, and for `application/json` objects it flattens `filterCondition.pBranch`/`pBranchName` arrays into comma-joined strings; empty or invalid body becomes `{}`; any exception passes the original request through. Problems: it prints content type, original body and modified body with `System.out.println` (bypasses log levels; contract numbers, `pCustomerName`, `pIdNo` can be printed), and it parses with `org.json` 20231013 with no size cap.
- **Common mistakes:** Calling it "just a wrapper". Not mentioning PII.
- **Follow-up questions:** How would you fix the logging? (Remove prints; log only report name and an id at DEBUG.) Why must the body be cached? (`HttpServletRequest` input stream is read once.)

### Q16. Why is the DTO field `FilterCondition` capitalised but the JSON key is `filterCondition`?
- **Why the interviewer asks:** A real contract gotcha in `PayLoadRequest`.
- **Ideal answer:** The Java field is declared `FilterCondition`; Lombok generates `getFilterCondition()`; Jackson derives the property name `filterCondition` from the getter. Callers must send lower-case `filterCondition`. Renaming the field would be safe for JSON but would touch every `getFilterCondition()` usage only if the getter name changed (it would not).
- **Common mistakes:** Sending `FilterCondition`. Claiming Jackson uses the field name.
- **Follow-up questions:** How could you accept both? (`@JsonAlias`.) What Jackson version is used? (Jackson 2 forced by `spring.http.converters.preferred-json-mapper=jackson2`.)

### Q17. What is `mergeStaticPdfFiles` and how does it reach the XML?
- **Why the interviewer asks:** Checks knowledge of a metadata path that is not a filter.
- **Ideal answer:** A list of static PDF names the print engine should merge. It can arrive inside `metadata` (lifted by the custom setter) or at top level of the request. It ends up in `TemplateDetails` and is written to the XML only when non-empty. The service just echoes it; whether the renderer restricts the file location is outside this repo.
- **Common mistakes:** Believing this service merges PDFs.
- **Follow-up questions:** Path-traversal risk? (Renderer side, not verifiable here.) Is the element in XSD `templateDetailsType`? (No; Common.xsd template details has 14 leaves and no merge list - see XSD files.)

### Q18. Which metadata keys populate MailingDetails and TemplateDetails?
- **Why the interviewer asks:** Verifies you read the controller, not only the services.
- **Ideal answer:** MailingDetails: `name`, `email`, `senderMail`. TemplateDetails: `copyType`, `batchConfigId`, `boResId`, `channel`, `dataClass`, `disallowCopy`, `docType`, `linkLocation`, `outputFormat`, `outputPath`, `reportName`, `templateId`, `templateName`. They are populated only when the strategy returned a non-null `Transaction`; otherwise the default transaction builder fills them.
- **Common mistakes:** Thinking the services build these.
- **Follow-up questions:** Why does `Transaction` order matter for the XSD? (Java writes templateDetails before mailingDetails; XSD wants mailingDetails before templateDetails - see XSD files.)

### Q19. What is the "default transaction" and when do you see it?
- **Why the interviewer asks:** Edge-case behaviour that decides what the caller prints.
- **Ideal answer:** `createDefaultTransaction(requestBody)` creates a `Transaction` with only mailing and template details. It is used when the strategy returns `Transactions` whose `getTransaction()` is null, and also on the unknown-report path (Q5). Response is still 200 SUCCESS. The XML has no report data, so the report side must treat it as "no data".
- **Common mistakes:** Assuming it signals an error.
- **Follow-up questions:** How does RejectRisk behave with zero rows? (Valid empty result; the ReportHeader block is added only when at least one row exists.)

### Q20. Why is a plain `new XmlMapper()` created in the controller, and why does that matter?
- **Why the interviewer asks:** Links API code to the XSD problem.
- **Ideal answer:** Serialisation is done by a fresh `XmlMapper` per request with default settings. Payload classes use `@JsonInclude(ALWAYS)`, so null decimals and dates become empty elements such as `<slNo/>`; `xs:decimal` / `xs:dateTime` do not accept empty content. Creating the mapper each call is also needless cost (it is thread-safe and reusable). A shared bean would let tests use the same mapper as production.
- **Common mistakes:** Blaming the XSD for empty elements.
- **Follow-up questions:** Options to fix? (`NON_NULL` inclusion, `xsi:nil` with `nillable`, or per-field handling.) What changes for the print engine? (Depends on whether it needs the empty tags; unknown.)

### Q21. Does the endpoint authenticate or authorise callers?
- **Why the interviewer asks:** Security basics with a project-specific twist: Spring Security is present but not protecting.
- **Ideal answer:** No. `SecurityConfig` sets stateless sessions, CSRF disabled, permits `/actuator/health` and `/actuator/info`, `denyAll` for other `/actuator/**`, and `anyRequest().permitAll()`. An empty `InMemoryUserDetailsManager` bean only stops the generated password. Protection must come from network policy or an upstream gateway (inferred; not in repo).
- **Common mistakes:** "It uses Spring Security so it is secure."
- **Follow-up questions:** How would you add a shared-secret or JWT filter quickly? Which endpoints must stay open? (health probes.)

### Q22. Why is CSRF disabled and when would that be wrong?
- **Why the interviewer asks:** Judgement, not memorisation.
- **Ideal answer:** The code comment justifies it as stateless, no cookies, no browser session (Sonar S4502 acknowledged). It is acceptable only because there is no session or cookie auth. If basic auth with browser caching or cookies were added, CSRF would have to be reconsidered.
- **Common mistakes:** "CSRF is not needed for POST APIs" as a blanket rule.
- **Follow-up questions:** What attack does CSRF need? (A browser carrying ambient credentials.)

### Q23. Is Swagger exposed, and does it matter?
- **Why the interviewer asks:** Information disclosure awareness.
- **Ideal answer:** Yes. `/swagger-ui/index.html`, `/v3/api-docs` and `/v3/api-docs/public` are reachable because of `anyRequest().permitAll()`. `OpenApiConfig` registers `GroupedOpenApi` "public" for `/api/**`. Risk is low (one endpoint) but it documents the shape for anyone on the network. In prod set `springdoc.api-docs.enabled=false` and `springdoc.swagger-ui.enabled=false`.
- **Common mistakes:** Saying it is hidden behind auth.
- **Follow-up questions:** Does the doc describe `filterCondition` keys per report? (No, it is a generic map.)

### Q24. Which actuator endpoints are reachable?
- **Why the interviewer asks:** Probes and exposure.
- **Ideal answer:** Only `health` and `info` (`management.endpoints.web.exposure.include=health,info`), at `/gi/documentprintdata/actuator/health` (context path applies). `management.endpoint.loggers.enabled=true` is set but `loggers` is not exposed, and security also denies other `/actuator/**`, so no runtime log level change. `show-details` is not set (default never).
- **Common mistakes:** Dropping the context path from probe URLs. Thinking loggers is usable.
- **Follow-up questions:** How would you expose `prometheus`? (Add micrometer registry, include in exposure list, permit in security.)

### Q25. Could someone inject a MongoDB operator through the API?
- **Why the interviewer asks:** NoSQL injection awareness grounded in the code.
- **Ideal answer:** Unlikely. Handlers build `org.bson.Document` filters with values coming from a `Map<String,String>`, so values are always strings. `pCustomerName` goes into an equality match (`CUST.NAME_EXT`), not into `$where`, `$regex` or `$expr` with user text. The remaining risks are not injection but heavy queries (unbounded date range) and PII exposure.
- **Common mistakes:** Saying NoSQL injection is impossible because it is "not SQL".
- **Follow-up questions:** What if the value became an object? (It cannot; Jackson would fail for `Map<String,String>`.) What if a handler used `$regex` with user input? (ReDoS and operator-like patterns.)

### Q26. Where does response XML get logged, and is that a problem?
- **Why the interviewer asks:** PII and log hygiene.
- **Ideal answer:** On the strategy path `log.debug("xml response data ...")`; on the legacy path `log.info(...)` of the full XML (with a TODO to lower it). RejectRisk output carries customer name and ID values. Mongo logging is also at DEBUG (`MongoTemplate`), and `CustomRequestFilter` prints bodies with `System.out`. Together these put personal data in logs (the business is Malaysian; PDPA concern, inferred).
- **Common mistakes:** Only mentioning one of the three log sources.
- **Follow-up questions:** How to mask? (Do not log bodies; log ids/lengths.) Who can read prod logs? [CONFIRM: log access rules]

### Q27. Is the API idempotent?
- **Why the interviewer asks:** REST semantics applied to a "POST that only reads".
- **Ideal answer:** Functionally yes: the pipeline only runs read aggregations and GET calls, so the same request returns the same data except `timestamp` and any data changes. It is modelled as POST because the body is large and structured. There is no idempotency key and none is needed since nothing is written (inferred).
- **Common mistakes:** Saying POST is never idempotent. Saying it is cacheable by HTTP caches.
- **Follow-up questions:** Could you use GET? (Filter map is too large and has PII; URLs get logged.) Could you cache? (Key = full body + data version; invalidation unknown.)

### Q28. What timeouts protect the inbound call?
- **Why the interviewer asks:** Resource protection and the gap between layers.
- **Ideal answer:** None are configured on the server side (`server.tomcat.*` and `spring.mvc.async` not set). Mongo `maxTime` is 600 s only in handlers that set `AggregationOptions` (for example `PlinePolicyDocumentMainHandler`); `RejectRiskHandler` sets none. Outbound HTTP has timeouts (5 s connect, 20 s read, 8 s fast-lookup). So a caller that gives up at 20 s (caller timeout unknown) can leave a query running up to 10 minutes (orphan work).
- **Common mistakes:** Claiming Tomcat's 20 s connection timeout limits processing.
- **Follow-up questions:** How to align them? (Set `maxTime` below the caller timeout; use `maxTimeMS` per pipeline.) What happens to the thread? (Stays busy until Mongo returns.)

### Q29. How would you version this API?
- **Why the interviewer asks:** Evolution strategy in a one-URL design.
- **Ideal answer:** The URL has no version. Because `metadata` and `filterCondition` are lenient maps, adding optional keys is backward compatible. For breaking changes: a new `reportName` bean (for example a `...V2` name) or a new path `/api/opus/v2/GenerateReport`. Response changes need care because the XML shape is consumed by Jasper templates (XSD v0.1.6 vs newer Java output shows the drift risk).
- **Common mistakes:** Saying "Accept header versioning" without checking the caller can send it.
- **Follow-up questions:** How did new fields like `mainReportName` roll out? (Added in the Java payload ahead of the XSD; see XSD files.)

### Q30. How do you protect the endpoint from expensive requests?
- **Why the interviewer asks:** DoS and cost thinking.
- **Ideal answer:** Today nothing in-app: no rate limit, no auth, no body-size cap, no mandatory filters. RejectRisk with no date range scans the whole `CMUW_PL_REFER_RISK`, does `$lookup`s per row and an in-memory sort. Mitigations: gateway rate limit and auth, mandatory date range (or maximum span), tighter `maxTime`, `allowDiskUse` plus a row cap, a request size limit filter.
- **Common mistakes:** Proposing only "scale the pods".
- **Follow-up questions:** What would you measure first? (Row count and `explain` for the biggest filter combos.)

### Q31. Which RestTemplate does each outbound client use, and why does it matter?
- **Why the interviewer asks:** A subtle injection trap with real timeout impact.
- **Ideal answer:** `PlineUtilsClient` uses `@Qualifier("fastLookupRestTemplate")` (8 s read). `PlinePolicyRefClient`, `PlineSSTRolloutDateClient`, `PlinePgRpgeUtilsClient` take an unqualified `RestTemplate`, so they get the `@Primary` motor `restTemplate` (5 s connect, 20 s read, no connection-request timeout, library-default pool). Developers often assume the "pline" bean.
- **Common mistakes:** Saying all Pline clients use `plineRestTemplate`.
- **Follow-up questions:** Consequence? (A slow policy-ref call can hold a request thread 20 s.) How to enforce the right bean? (Qualifiers on every constructor.)

### Q32. External calls fail in this service. Walk me through what the caller sees.
- **Why the interviewer asks:** Fail-soft design with business consequences.
- **Ideal answer:** The Pline clients catch failures, log, and return `null` or `""` (for example `safeGet` -> ""). The request still returns 200 and a valid-looking report whose indicator fields (for example e-invoice print indicator, company BRN) are blank. That keeps reports flowing but can print a wrong document silently. Detection is only by logs ("PlineUtilsClient GET failed"); no metric or alert exists in repo.
- **Common mistakes:** Expecting a 502/503.
- **Follow-up questions:** When should it fail hard instead? (When the field changes legal content, e.g. tax invoice indicator.) How would you add a health signal? (Counter per upstream, expose via actuator metrics.)

### Q33. Explain the SSRF protections in `PlineUtilsClient`.
- **Why the interviewer asks:** A flagship security answer for this project.
- **Ideal answer:** (a) Host comes from configuration (`API_BASE-URL`) and the path from configured keys; callers never supply a URL. (b) `assertTrustedHost(url)` requires `url.startsWith(baseUrl)` before every call (`safeGet`, `safeGetEncoded`, `postForFirstDataEntry`, `fnGetCompBrnNumber`). (c) Query values pass `sanitize()` with regex `[A-Za-z0-9_\-./#]*`; other values become "". (d) `PlinePolicyRefClient` additionally validates `contractId` with `[A-Za-z0-9_\-]+`.
- **Common mistakes:** Claiming full URL allow-listing. Not saying the host is never user-controlled.
- **Follow-up questions:** Weakness of `startsWith`? (Prefix match; with a base lacking a trailing slash `https://host.evil` would match, but the base is operator-set.) Is `PlineEinvoiceQrIndicatorClient` audited? (Not verified.)

### Q34. What happens when `sanitize()` rejects a value?
- **Why the interviewer asks:** Silent data change under a security control.
- **Ideal answer:** The value becomes an empty string, not an error. The outbound call then runs with a blank parameter, usually returning nothing, and the field prints blank. It is safe but hides bad input (for example a `checklistCopy` value with a space). I would log at WARN with the param name, not the value.
- **Common mistakes:** Believing it throws `IllegalArgumentException`.
- **Follow-up questions:** Does the regex allow `../`? (Yes `.` and `/` are allowed, but values go in the query string and `UriComponentsBuilder` encodes them, so no server-path traversal.)

### Q35. `assertTrustedHost` throws `SecurityException` outside the try block in `safeGet`. What is the effect?
- **Why the interviewer asks:** Reading code for exception flow, not only intention.
- **Ideal answer:** Everything else in `safeGet` is fail-soft, but the host check is outside the try, so a mismatch propagates. `SecurityException` is not `IllegalArgumentException`, so the catch-all handler returns HTTP 500. It is intentional fail-closed for a security condition, though in practice it can only trigger on bad configuration.
- **Common mistakes:** Saying all external failures are swallowed.
- **Follow-up questions:** Would you keep it fail-closed? (Yes, but log at ERROR.) How would a base-URL typo present? (500 on every call that needs the util.)

### Q36. There are no retries or circuit breakers on the Pline clients. Is that a problem?
- **Why the interviewer asks:** Resilience design judgement.
- **Ideal answer:** Evidence: only motor `PMEGetContractIdClient` uses `ext.pme.retry.*`; Pline clients have none, and no Resilience4j. With 8 s/20 s timeouts and up to about five utility calls per Template12 request, an unhealthy upstream can add tens of seconds per request. Because the clients are fail-soft, a retry could mask blips, but a retry on a non-idempotent call would be wrong; these are GETs so retry is safe. A circuit breaker would cut wasted waiting when the upstream is down.
- **Common mistakes:** Adding retries with no bound or jitter.
- **Follow-up questions:** Where would you put the breaker? (Around `PlineUtilsClient.safeGet`.) What would the fallback be? (Same null/"" as now, but fast.)

### Q37. RejectRisk makes no external call. Why does that matter for design?
- **Why the interviewer asks:** Differentiates the modules' failure surface.
- **Ideal answer:** RejectRisk only reads Mongo (`CMUW_PL_REFER_RISK` plus lookups, and `RPGE_RPT_TEMPLATE` via `PlineReportHeaderHandler`). So it has no fail-soft blank-field risk from upstream APIs; its risks are data volume and Mongo performance. Template12 and ASP add the outbound calls on top of Mongo work.
- **Common mistakes:** Saying all three call the util APIs.
- **Follow-up questions:** Which module is the latency risk? (Template12: many aggregates plus up to five utility calls.)

### Q38. How would you test the error contract of this endpoint?
- **Why the interviewer asks:** Links testing to API behaviour; the repo has little Spring-context testing.
- **Ideal answer:** The repo's tests are unit tests with mocked `MongoTemplate`; there is one Spring context test in motor and no Testcontainers (testing guide). For the contract I would use `@WebMvcTest(CommonController.class)` or standalone `MockMvc` with `@Import` of the two advice classes: post unknown report (expect 200 and empty), post `{` (observe status), post with a strategy that throws `IllegalArgumentException` (expect 400 + `BAD_REQUEST`), `NoDataFoundException` (404), `RuntimeException` (500). Assert the `errorCode` field.
- **Common mistakes:** Testing only the service classes and calling it API testing.
- **Follow-up questions:** Why might `@SpringBootTest` be fragile here? (The pom comment says a micrometer version once broke context startup.)

### Q39. A caller says "I get SUCCESS but the document is blank". How do you debug?
- **Why the interviewer asks:** Realistic triage using the project's known blind spots.
- **Ideal answer:** Checklist: (1) exact `reportName` spelling and case (Q4/Q5) - unknown names return 200 empty; (2) filter keys spelled as the service reads them (`pContractId`, `versionNo`, `checklistCopy`); (3) types: contract id must be numeric text; (4) Mongo database name in `spring.mongodb.database` (wrong DB = zero rows, no error); (5) WIP vs live (`checklistCopy`); (6) enable MongoTemplate DEBUG to see the pipeline and run it in Atlas; (7) look for "PlineUtilsClient GET failed" lines for blank indicators.
- **Common mistakes:** Jumping to the XSD or the print engine first.
- **Follow-up questions:** What do you do if logs show nothing? (Reproduce with curl from the sample request; compare with a known-good contract.)

### Q40. What is `PlineReportNames.tag()` and why did the API response change in October?
- **Why the interviewer asks:** Recent change you may have touched; contract awareness.
- **Ideal answer:** `PlineReportNames.tag(transactions, getClass())` stamps `mainReportName` (and sub-report name through `PlineReportBlocks`) on every `reportData` so the print engine can tie a block to a main/sub JRXML. Commits `e74bc0d5c` and `861f0666c` (7-8 Oct 2026, "Added report, sub report name in response/listing report") added it. These elements are not in XSD v0.1.6, so the response is additive for name-based readers but invalid against v0.1.6. [CONFIRM: whether you made these commits; git authorship cannot be attributed to you.]
- **Common mistakes:** Saying the XSD already has them.
- **Follow-up questions:** Who asked for it? [CONFIRM: requirement source.] How would you notify the consumer? (Release note + schema bump.)

### Q41. Why was `spring.http.converters.preferred-json-mapper=jackson2` needed in the Boot 4.1.1 upgrade?
- **Why the interviewer asks:** The upgrade was a real piece of the project history.
- **Ideal answer:** Boot 4.1.1 defaults to Jackson 3 (`tools.jackson.*`), which ignores `com.fasterxml.jackson` annotations used by `PayLoadRequest` (for example `@JsonProperty`) and the DTOs. Without the property, POSTs fail with `HttpMessageConversionException`. Commit `9248b12ff` (18 Sep 2026) added it, with comments in `application.properties`. The pom also pins `jackson-2-bom.version` (not `jackson-bom.version`, which Boot 4 repurposed).
- **Common mistakes:** Removing it as "unused".
- **Follow-up questions:** How would you migrate to Jackson 3 later? (Change annotation imports, mapper builders; test XML output carefully because the XML mapper is part of the contract.)

### Q42. What does `GET /api/opus/GenerateReport/getJSON` do?
- **Why the interviewer asks:** Knowing the dead corners of an API.
- **Ideal answer:** It returns an empty `ResponseDTO` (utility/legacy). `DocumentController` is an empty class and `ExtendedController` is an empty file. It is safe to leave but should be removed or documented, as it widens the surface with no purpose.
- **Common mistakes:** Saying it returns the report JSON.
- **Follow-up questions:** How do you confirm an endpoint is unused before removing? (Gateway/access logs [CONFIRM: available?].)

### Q43. Two `@RestControllerAdvice` classes exist plus two commented-out advices. What is the risk?
- **Why the interviewer asks:** Cross-cutting behaviour scattered across packages.
- **Ideal answer:** `motor.exception.GlobalExceptionHandler` (404 and catch-all 500) and `generic.advice.GenericExceptionHandler` (400) are both active and unscoped, so they apply to every controller. `FireRequestBodyAdvice` and `MiscRequestBodyAdvice` are commented out. Risk: `IllegalArgumentException` thrown anywhere now becomes a generic 400 and its message is only logged (WARN), and vetting every throw site was never done (class javadoc admits it).
- **Common mistakes:** Assuming advice is scoped by package.
- **Follow-up questions:** How to scope? (`@RestControllerAdvice(assignableTypes=...)` or basePackages.)

### Q44. How would you add authentication without breaking the existing callers?
- **Why the interviewer asks:** Practical security change management.
- **Ideal answer:** Step 1: confirm who calls (reporting layer, OPUS) [CONFIRM: callers]. Step 2: add a `OncePerRequestFilter` that validates a gateway-supplied token or shared secret, keep health/info open, switch to `authorizeHttpRequests(...).anyRequest().authenticated()`. Step 3: roll out in "log-only" mode first, then enforce. Step 4: put secrets in a secret store, not in `application.properties`. Network policy remains as defence in depth.
- **Common mistakes:** Turning on `httpBasic()` without coordinating callers.
- **Follow-up questions:** How do you handle probes? (Permit `/actuator/health`.) mTLS instead?

### Q45. The service logs `MongoTemplate` at DEBUG. Any API-level risk?
- **Why the interviewer asks:** Ties config to behaviour.
- **Ideal answer:** `logging.level.org.springframework.data.mongodb.core.MongoTemplate=DEBUG` logs every operation with query documents; for RejectRisk that includes names/IDs in `$match` values. It slows the request path and leaks identifiers. Useful in dev; must be INFO in prod. Since the loggers endpoint is not exposed, changing it needs a redeploy.
- **Common mistakes:** Believing DEBUG only prints queries without values.
- **Follow-up questions:** How to enable temporarily in prod? (Expose `loggers` behind auth.)

### Q46. How does the response get big, and what is the risk?
- **Why the interviewer asks:** Memory and payload thinking.
- **Ideal answer:** The whole response is built in memory: `getMappedResults()` loads all rows, objects are serialised to one XML string, then wrapped in JSON (double escaping). Template12 `transDet` has up to about 650 elements per node, many empty (`ALWAYS` inclusion), so XML is large. RejectRisk produces one `reportData` per row, so thousands of rows mean large heap use. Streaming and pagination do not exist.
- **Common mistakes:** Assuming Tomcat compresses or streams it.
- **Follow-up questions:** Mitigation? (Row cap, `NON_NULL`, gzip at gateway, paging by date.)

### Q47. Why is `API_BASE-URL` a risky key name for operations?
- **Why the interviewer asks:** Config meets API deployment.
- **Ideal answer:** It is upper-case with a hyphen, read by about 45 `@Value("${API_BASE-URL}")` sites. POSIX shells cannot `export` a name with `-`; Kubernetes `env:` can carry it, but whether `API_BASE_URL` resolves it by relaxed binding is expected, not proven here. Safer: JVM arg `-DAPI_BASE-URL=...` or an external config file via `spring.config.additional-location`. Also `plines.api.base-url` is a second, independent key for the other Pline clients.
- **Common mistakes:** Changing only one of the two base URLs.
- **Follow-up questions:** How to verify at startup? (Log the resolved base URL on boot; hit a downstream and check the logged URL.)

### Q48. What does the caller need to know about `checklistCopy`?
- **Why the interviewer asks:** A business-meaning request field in API terms.
- **Ideal answer:** `checklistCopy = Y` means a preview/checklist copy produced from WIP (work-in-progress) collections before issue; `N` means the posted live document. Services default it to "N". It also flows to outbound calls (for example `fnGetEinvoicePrintPolInd(contractId, checklistCopy)`) and is echoed as `pchecklistCopy` in reportParameters.
- **Common mistakes:** Treating it as a boolean type (it is a string "Y"/"N").
- **Follow-up questions:** What if a caller sends "y"? (Behaviour depends on the comparison used; verify the handler before claiming.)

### Q49. The request key `ShowOutPrintCovLetter` starts with a capital S. Why does that matter?
- **Why the interviewer asks:** Exact-key contracts in a stringly-typed API.
- **Ideal answer:** `Template12PolicyDocumentService` reads the key with the capital S (per the KT doc). Maps are case-sensitive, so `showOutPrintCovLetter` would be ignored and default "N" used. It also differs from the XSD-whitelisted `pShowOutPrintCovLetter`, so the value never appears in the XML.
- **Common mistakes:** Normalising case in your head.
- **Follow-up questions:** How could the API be made tolerant? (Case-insensitive map, `TreeMap(String.CASE_INSENSITIVE_ORDER)`.) Is that a good idea? (It hides caller bugs, but reduces support tickets.)

### Q50. If you could change three things about this API first, what would they be?
- **Why the interviewer asks:** Prioritisation and ownership.
- **Ideal answer:** (1) Fail loudly on unknown `reportName` and missing `metadata`/`filterCondition` (400), and log exceptions in the catch-all handler. (2) Add authentication/gateway enforcement and remove body printing (`System.out`) and DEBUG Mongo logging in prod. (3) Make the XML contract testable: shared `XmlMapper` bean, ISO dates and non-null inclusion, and a schema-validation test of real service output. Each is small, low-risk and fixes a documented gap.
- **Common mistakes:** Proposing a rewrite or a new framework.
- **Follow-up questions:** Which first and why? How would you convince the caller team?
