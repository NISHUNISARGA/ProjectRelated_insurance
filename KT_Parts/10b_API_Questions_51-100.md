# 10b - API Questions 51-100 (opus-documentprintdata-service)

Evidence base: `06_Config_API_DB_DevOps.md`, `04_XSD_XML_Deep_Dive.md`, `05_Testing_Deep_Dive.md`, `01/02/03` deep dives, and repo files named in each answer. "(inferred)" = reasoning, not proven. Personal facts are [CONFIRM: ...]. No secrets are printed.

---

### Q51. List the outbound calls Template12 PolicyDocument can make and what each one decides.
- **Why the interviewer asks:** Shows you know the external dependencies of your own module.
- **Ideal answer:** Through `PlineUtilsClient` (fast-lookup bean, 8 s): `fnGetEinvoicePrintPolInd` (e-invoice print indicator, path `/ext/utils/api/v1/pg-rpge-utils/invoice-print-indicator`), `fnGetCoInwardInd` (`.../co-inward-ind`), `fnShowStaxInvInd` (`/ext/policy/api/v1/stax-inv`), `fnGetCompBrnNumber` (company BRN, `.../comp-brn-number`). Plus `PlineSSTRolloutDateClient.getSSTRolloutDate` (SST rollout date, used by the Premium Debit Note handler `PlineT12PremiumDebitNoteHandler`), which uses the primary 5 s/20 s RestTemplate. Calls are made from `PlinePolicyDocumentMainHandler`, `PlineT12PolicyDocumentHandler` and the base service. Results gate or fill parts of the document.
- **Common mistakes:** Listing all ~45 external paths in the properties file.
- **Follow-up questions:** Which are GET? (All are GET in the catalog.) Which can change legal content? (Tax invoice indicators.)

### Q52. Same question for ASP EndtDocument.
- **Why the interviewer asks:** Compare module dependencies.
- **Ideal answer:** `fnEinvoicePrintEndtInd` (`/ext/utils/api/v1/pg-rpge-utils/invoice-print-endt-indicator`, params contractId, endtNo, refNo, versionNo, checklistInd; from `AspEndtDocument2MainHandler`), `fnGetCompBrnNumber` (company name with GST reg no and signature handlers), `fnRppmGetSignature*` (`/ext/customer/api/v1/fn-rppm-get-signature`, signature type default `UW`), and for the service tax invoice: `PlineSSTRolloutDateClient`, `PlinePolicyRefClient.getPolicyReference` (`/ext/policy/api/v1/pme-api/get-policy-ref`) and `PlineEinvoiceQrIndicatorClient` (not audited in detail).
- **Common mistakes:** Mixing the two policy-ref paths (see Q75).
- **Follow-up questions:** What happens when signature lookup returns null? (Fail-soft null/"" - the signature block prints blank.)

### Q53. `app.http.connection-request-timeout-ms` only applies to some beans. Which, and what is the risk for the others?
- **Why the interviewer asks:** Connection-pool exhaustion understanding.
- **Ideal answer:** `connection-request-timeout-ms=3000`, `max-conn-total=100`, `max-conn-per-route=20` are read only by `PlineRestTemplateConfig` (pline beans). The motor `restTemplate` reads only connect/read/socket timeouts, so it uses HttpClient5 library defaults for pool size (about 25 total / 5 per route, inferred) and no wait limit. Clients that got the `@Primary` bean (Q31) can therefore queue for a connection without a bound if the pool is exhausted. Since every call goes to the same host, per-route 5 matters (inferred).
- **Common mistakes:** Believing the 100/20 pool applies to all clients.
- **Follow-up questions:** How to prove pool starvation? (Thread dump; connection-leasing metrics.) Fix? (Qualify the clients onto the pline bean.)

### Q54. Estimate worst-case latency for one Template12 request caused by external calls.
- **Why the interviewer asks:** Reasoning with real numbers rather than guesses.
- **Ideal answer:** Up to about five distinct utility calls. If all hang: fast-lookup calls cost up to 8 s each, the SST call up to 20 s (primary bean read timeout), plus connect 5 s on a dead host. Sequential worst case is therefore well above a minute (inferred: calls are made sequentially in handlers; verify). Plus Mongo aggregates with `maxTime` 600 s. In normal operation each call is milliseconds. Honest answer: I have not measured it; [CONFIRM: any production latency numbers].
- **Common mistakes:** Inventing p95 numbers.
- **Follow-up questions:** How to reduce? (Parallel calls with a bound, cache indicators per contract, circuit breaker.)

### Q55. Explain the effect of the custom truststore on outbound API calls.
- **Why the interviewer asks:** TLS knowledge tied to repo specifics.
- **Ideal answer:** `RestTemplateConfig` and `PlineRestTemplateConfig` build an SSL context with `SSLContexts.custom().loadTrustMaterial(trustStore, null)` from `mytruststore.p12` (classpath, type PKCS12, password in `application-env.properties` = `<redacted>`). This replaces the JVM default cacerts for those RestTemplates, so an upstream whose CA is not in the file fails the handshake. Fail-soft clients would then return blanks. Mongo (`ssl=true`) uses the JVM default trust, not this file.
- **Common mistakes:** Saying the truststore also secures Mongo or inbound traffic.
- **Follow-up questions:** What if the CA rotates? (Rebuild the jar with a new p12 or externalise it.) What exception at startup if the password is wrong? (`IOException: keystore password was incorrect`.)

### Q56. How would you diagnose "the e-invoice indicator is blank for one policy only"?
- **Why the interviewer asks:** End-to-end API debugging with fail-soft clients.
- **Ideal answer:** (1) Reproduce with the same `pContractId`, `checklistCopy`. (2) Search logs for "PlineUtilsClient GET failed" with that contract. (3) Check `sanitize()` did not blank an input (values outside `[A-Za-z0-9_\-./#]`). (4) Call the upstream path directly with the same params (dev2 host from `API_BASE-URL`). (5) Check whether the handler skips the call because of a data gate (for example blank issue date skips the BRN call). (6) Compare with a working policy.
- **Common mistakes:** Assuming the upstream is down without checking inputs.
- **Follow-up questions:** If upstream returns 404? (Likely a path key issue; see stale-key notes in config files.)

### Q57. What is the `ReportStrategy` contract and why does it return `Transactions`?
- **Why the interviewer asks:** Knows the SPI.
- **Ideal answer:** `motor/strategy/ReportStrategy` has one method `execute(PayLoadRequest)` returning `Transactions` (the XML root). Each module builds the whole tree (`Transaction` > `transactionDetails` > `headerTransaction` > `reportTransaction` > `reportData`...). The controller then adds mailing/template details and serialises. This keeps the controller report-agnostic.
- **Common mistakes:** Thinking the strategy writes the HTTP response.
- **Follow-up questions:** What if `execute` returns a `Transactions` with null `Transaction`? (Default transaction, Q19.)

### Q58. Why does RejectRisk return one `reportData` per row while Template12 returns a few?
- **Why the interviewer asks:** Response design tied to Jasper shape.
- **Ideal answer:** RejectRisk is a listing: each DB row becomes a `reportData` with its own `prntDetails/transDetList` holding one `transDet` (plus a trailing ReportHeader `reportData` if rows exist). Template12 is a document with sub-reports, so one `reportData` per printed block. The KT guide notes a disclaimer: the report side must treat each entry as a print unit. Whether the print engine handles thousands of `reportData` efficiently is unknown.
- **Common mistakes:** Calling it a bug.
- **Follow-up questions:** Alternative? (One `reportData` with many `transDet`, as the XSD allows unbounded `transDet`.)

### Q59. How would you test RejectRisk end to end locally?
- **Why the interviewer asks:** Practical hands-on knowledge.
- **Ideal answer:** Start the app with Mongo reachable (URI from env, not a committed file). POST to `http://localhost:8080/gi/documentprintdata/api/opus/GenerateReport` with the sample JSON (`reportName` set in `filterCondition`, dates `pCreatedDateFrom/To`). Check: status SUCCESS, `data` parses as XML, one `reportData` per row, `rownumber` starts at 1, last `reportData` is the ReportHeader. Then change filters (`pRejType`, `pIdNo`) and compare counts with a mongosh query.
- **Common mistakes:** Calling `/api/opus/GenerateReport` without the context path.
- **Follow-up questions:** What if DB name is wrong? (Zero rows, no error.) Which tool? (curl/Postman; saved `API_Crul_*.txt` files exist in `T12_final_code\RejectRisk`.)

### Q60. What defaults does the ASP service apply to request keys?
- **Why the interviewer asks:** Knowing defaults explains behaviour of "missing key" requests.
- **Ideal answer:** From `AspEndtDocumentService`: `showCoverNote`, `showEndtSchedule`, `showComplaint`, `showDebitNote`, `showTaxInvoice` default "N"; `pFirstPageNumber` default "1"; others default "". `pContractId` and `versionNo` must be numeric text because handlers parse them to Long (`safeNum`).
- **Common mistakes:** Assuming "Y" defaults.
- **Follow-up questions:** What if `pContractId` is "ABC"? (`Long.parseLong` fails; fallback to the raw string, which then matches nothing in Mongo; empty result.)

### Q61. And the Template12 defaults?
- **Why the interviewer asks:** Compare to Q60.
- **Ideal answer:** `versionNo` "1", `checklistCopy` "N", `showComplaint` "Y", `showScheduleStd` "Y", other `show*` "N", `ShowOutPrintCovLetter` "N". Branching is also driven by data, not the request: product codes 070504/070412/070406/070414 and scheme names (BMW, BMW1, POSMESRA, FWPA, POSCOVER) are looked up from Mongo.
- **Common mistakes:** Saying the request chooses the scheme.
- **Follow-up questions:** Why is data-driven branching hard to test? (Needs Mongo data per scheme; tests mock handlers.)

### Q62. Request-driven versus data-driven gates: how do `show*` flags combine with data?
- **Why the interviewer asks:** Understand when a sub-report prints.
- **Ideal answer:** A sub-report prints when the request flag allows it AND the data-driven condition holds (for example `SHOW_GST_INV_IND` from Mongo or an indicator from an external API). The Jasper original had the same print conditions; the Java mirrors them. So "flag = Y but nothing printed" often means the data condition is false, not a bug in request parsing.
- **Common mistakes:** Debugging only the request.
- **Follow-up questions:** How to confirm? (Log the gate values; compare with the JRXML printWhenExpression.)

### Q63. Which date formats does RejectRisk accept for `pCreatedDateFrom`/`pCreatedDateTo`?
- **Why the interviewer asks:** Input contract details.
- **Ideal answer:** Whatever `plines.utils.DateUtils.toLocalDate` parses; the exact list is not documented in the KT notes and should be read from that class before answering with certainty. The working-copy change switched `RejectRiskService` from `motor.utils.DateUtils` to `plines.utils.DateUtils`. Both the handler (`toDate`) and the service (`toLocalDateTime`) parse the same raw string; an unparseable value yields null, meaning "no bound" on the handler side.
- **Common mistakes:** Stating a format as fact. Not noticing that a bad date silently removes the filter.
- **Follow-up questions:** Should a bad date be a 400? (Yes, better contract.) Time zone? (Handler uses UTC start of day.)

### Q64. A bad date silently removes the bound. What is the business risk?
- **Why the interviewer asks:** Silent failure in a listing report.
- **Ideal answer:** `toDate` returns null on parse failure, the handler then appends no `$gte`/`$lt`, and the query returns every reject-risk row. The report header would echo a null date (the Jasper original prints "NA"). A user could think the report is for January while it contains all history, and PII volume rises. Fix: throw `IllegalArgumentException` for non-blank unparseable dates so `GenericExceptionHandler` returns 400.
- **Common mistakes:** Saying null would return zero rows (that was the old code before the working-copy fix).
- **Follow-up questions:** What did the old code do with a null bound? (`$gte: null` matches only null-date rows, giving empty output.)

### Q65. How is GET `/actuator/health` reached by Kubernetes probes, given the context path?
- **Why the interviewer asks:** Real deployment detail.
- **Ideal answer:** Path is `/gi/documentprintdata/actuator/health` because `server.servlet.context-path=/gi/documentprintdata` prefixes everything. Port 8080. Probes configuration is in Helm, not in this repo, so my statement about the probe URL is inferred [CONFIRM: Helm values]. Health shows only status (no `show-details`).
- **Common mistakes:** Probing `/actuator/health` at root.
- **Follow-up questions:** Does health include Mongo? (Boot's Mongo health indicator is on by default when actuator and Mongo are present; inferred.)

### Q66. What are the content-negotiation rules of the POST?
- **Why the interviewer asks:** HTTP basics, applied.
- **Ideal answer:** `@PostMapping(produces = application/json)`; request is read as JSON by Jackson 2 converter. A client sending `Accept: application/xml` only would get 406 (inferred; and the catch-all might turn it into 500, see Q11). `CustomRequestFilter` only rewrites when `Content-Type` is `application/json`.
- **Common mistakes:** Expecting XML back when asking for XML.
- **Follow-up questions:** Why is XML inside JSON? (Existing contract.)

### Q67. Why don't you validate request fields with Bean Validation?
- **Why the interviewer asks:** Design critique with awareness of the current state.
- **Ideal answer:** `spring-boot-starter-validation` is on the classpath but `PayLoadRequest` has no constraints and the controller has no `@Valid`. The model is a generic map shared by all LOBs; per-report rules live in services via `getOrDefault`. Option: a per-report validator interface called by the controller before `execute`, throwing `IllegalArgumentException` (-> 400). It must not break callers that rely on lenient defaults, so introduce it in warn-only mode first.
- **Common mistakes:** Adding `@NotNull` on a map value without thinking of the 14 LOBs.
- **Follow-up questions:** Which checks first? (reportName present, contract id numeric, dates parseable.)

### Q68. How do you trace one request across logs?
- **Why the interviewer asks:** Observability maturity.
- **Ideal answer:** You cannot well today: no MDC/correlation id, no JSON logs, no request id sent to outbound calls (KT: logging notes). You can grep by contract id or report name where handlers log them. Improvement: a filter that sets an MDC `requestId` (from a gateway header or UUID), logging pattern with `%X{requestId}`, and an outbound interceptor adding the same header.
- **Common mistakes:** Saying Sleuth/Micrometer tracing is already set (not in evidence).
- **Follow-up questions:** Where would the outbound interceptor go? (On the RestTemplate beans in the two config classes.)

### Q69. What metrics would you add for the API?
- **Why the interviewer asks:** Operability thinking.
- **Ideal answer:** Actuator exposes only health/info now. Add: request count and latency tagged by `reportName` (custom timer in the controller since the URL is one), count of default (empty) transactions, count of fail-soft upstream failures per client, Mongo aggregation duration per handler. Expose via `prometheus` and alert on empty-transaction spikes and upstream failure ratio.
- **Common mistakes:** Only HTTP status metrics (always 200).
- **Follow-up questions:** Cardinality risk of tagging by `reportName`? (Low; only a few dozen names.)

### Q70. Why are HTTP status metrics weak for this API?
- **Why the interviewer asks:** Ties the error contract to monitoring.
- **Ideal answer:** Most problems return 200: unknown report, no data, upstream failure blanks, wrong DB. Only exceptions give 4xx/5xx. So a dashboard of 5xx rates would look healthy while documents print blank. You need business-level signals (Q69) and synthetic checks with known contracts.
- **Common mistakes:** Trusting "200 = success".
- **Follow-up questions:** What synthetic check? (A canary POST for a fixed test policy per report and assert on key elements.)

### Q71. How would you design pagination or limits for RejectRisk?
- **Why the interviewer asks:** Practical API design for a listing.
- **Ideal answer:** The report contract is one document, so classic paging is hard. Options: require `pCreatedDateFrom/To` and cap the span (for example 31 days); cap the row count and return 400 when exceeded; or let the handler add `$limit` after `$sort` plus a flag in `reportParameters`. Currently the pipeline has no final `$limit`. Any cap changes report completeness, so it needs a business decision [CONFIRM: acceptable maximum rows].
- **Common mistakes:** Silently truncating.
- **Follow-up questions:** How would the caller learn about truncation? (Explicit error or a parameter echo.)

### Q72. Sync request versus async job for heavy reports?
- **Why the interviewer asks:** Architecture trade-off.
- **Ideal answer:** Today fully synchronous: Tomcat thread blocked while aggregates and external calls run, response returned in the same call. For long reports an async design (202 + job id + polling) would protect threads but requires the caller to change. Given the unknown caller timeout (20 s guess from `app.http.read-timeout-ms` is outbound, not inbound) I would first measure durations, then decide. Not recommended to change without caller agreement.
- **Common mistakes:** Proposing WebFlux for a Mongo-heavy blocking pipeline.
- **Follow-up questions:** What thread pool limit applies? (Tomcat default max threads 200, inferred, not configured here.)

### Q73. What if two sub-reports need different `reportParameters`?
- **Why the interviewer asks:** XSD cardinality leaking into API design.
- **Ideal answer:** XSD allows only one `reportParameters` per `transaction` (0..1). Landscape builds two `reportData` and two parameter objects, but only one can be serialised in a valid document (inferred, see XSD guide). Per-block values must travel in `reportData` fields or `paramGroup` entries instead.
- **Common mistakes:** Assuming each reportData has its own parameters.
- **Follow-up questions:** What is `paramGroup`? (Common type: dataType, paramCode, paramValue.)

### Q74. The properties file has `fire.autorenewal.*` paths marked TODO "confirm with API team". Does this affect your modules?
- **Why the interviewer asks:** Scoping and risk awareness.
- **Ideal answer:** No: those keys (lines 178-182) belong to Fire AutoRenewal, not to the three modules. But it shows the same risk class: external paths guessed or copied. For my modules the highest-risk guessed path is `plines.fn.date.value.constants.path`, whose comment says it was derived and may 404. I would check each path used by my modules against the API team's catalog [CONFIRM: catalog access].
- **Common mistakes:** Claiming all paths are verified.
- **Follow-up questions:** How to detect a 404 path? (Startup smoke test calling each path once.)

### Q75. Why are there two policy-ref endpoints, and which one does ASP use?
- **Why the interviewer asks:** Detailed config/API reading.
- **Ideal answer:** `plines.policy.ref.path` = `/ext/policy/api/v1/pg-uwge-utils/get-policy-ref` (`PlinePolicyRefClient.getPolicyRef`) and `plines.policy.ref.pme.path` = `/ext/policy/api/v1/pme-api/get-policy-ref` (`getPolicyReference`). ASP ServiceTaxInvoice uses the PME one (`AspServiceTaxInvoiceHandler`, line near 83). Both answer the same business question from different backend services. Which is authoritative is not documented [CONFIRM with API team].
- **Common mistakes:** Treating them as duplicates to delete.
- **Follow-up questions:** Why did the PME key get added under the Marine heading? (Organic config growth; commit `4cd1cad2e`.)

### Q76. `PlinePolicyRefClient` validates `contractId` with `[A-Za-z0-9_\-]+` while `sanitize` is looser. Why two rules?
- **Why the interviewer asks:** Security consistency.
- **Ideal answer:** `PlinePolicyRefClient` builds a URI with `build(true)` (assumes already-encoded input), so it must reject anything that could change the path or query: strict allow-list, returns null for invalid. `PlineUtilsClient` puts values in query params that `UriComponentsBuilder` encodes, so a looser allow-list is acceptable. The inconsistency is a maintenance risk; one shared validator would be cleaner.
- **Common mistakes:** Saying `build(true)` encodes the input (it does not).
- **Follow-up questions:** What is the template-expansion pitfall with `getForObject(String url...)`? (`#`/`%`/braces handled as URI variables; documented near `PlineUtilsClient` lines 597-606.)

### Q77. How do you tell dev from UAT from prod for the outbound calls?
- **Why the interviewer asks:** Environment management.
- **Ideal answer:** There is no single switch. Base URLs: `API_BASE-URL` (PlineUtilsClient and many others), `plines.api.base-url` (the three `Pline*Client`), `workmen.api.base-url`, plus Mongo URI/database. `spring.profiles.active=dev` does nothing because no `application-dev*.properties` exists. Promotion requires changing at least those keys, probably via Helm values outside this repo [CONFIRM: how deployments override].
- **Common mistakes:** Believing the Spring profile switches environment.
- **Follow-up questions:** How to prevent a prod pod calling dev2? (Startup check that logs resolved hosts; fail if host contains "dev" under a prod flag.)

### Q78. Unreferenced base URLs (dev1 hosts) still exist. Is there a risk?
- **Why the interviewer asks:** Config hygiene versus runtime impact.
- **Ideal answer:** Keys such as `external.pme.base-url`, `pg-rpge-utils.base-url`, `pg-uwge-utils.base-url`, `ext.sst-rollout-date.base-url` point to dev1 and are read by no `@Value` found in `src/main/java`. Runtime risk is none today, but they mislead operators ("I changed it, nothing changed") and carry real hostnames. Remove in a clean-up commit after a repo-wide check including `Environment.getProperty` (none found).
- **Common mistakes:** Deleting active dev2 keys by mistake.
- **Follow-up questions:** How to prove a key is unused? (Search all `${...}`, `@ConfigurationProperties` and `getProperty` usage.)

### Q79. What does the API reveal about OPUS integration ownership?
- **Why the interviewer asks:** Ownership clarity.
- **Ideal answer:** The service is a data provider: JasperServer or OPUS (callers, outside this repo) call it, receive XML, and render. The Oracle functions it wraps are exposed by other teams' REST gateways (`/ext/utils`, `/ext/policy`, `/ext/customer`). My team owns the three report strategies and their mapping. Contract changes (XSD) need the print/report side [CONFIRM: team names].
- **Common mistakes:** Claiming this service renders PDFs.
- **Follow-up questions:** Who validates XML against the XSD? (Not evidenced.)

### Q80. How would you introduce a breaking change to the XML but keep old callers working?
- **Why the interviewer asks:** Compatibility strategy.
- **Ideal answer:** Keep the old behaviour behind the existing `reportName` and add a new strategy bean (new report name) or an opt-in metadata flag. Run both until the consumer migrates, then retire. Examples of breaking XML changes in history: v0.1.3 `prntDetails` move; v0.1.4 removal of `showCoverNote`/`showTaxInvoice`. Java handled renames with shadow fields rather than versioned endpoints.
- **Common mistakes:** Changing silently because "only one consumer".
- **Follow-up questions:** Cost of two beans? (Duplicate maintenance; use a shared base class.)

### Q81. Explain how `Content-Length`, big bodies and body-size limits relate to this API.
- **Why the interviewer asks:** Edge protection.
- **Ideal answer:** Request bodies are small JSON. No explicit limit is set; Spring Boot/Tomcat defaults apply (for example a 2 MB form/post-parse limit applies only to form posts, JSON size is unbounded by default, inferred). `CustomRequestFilter` reads the full body into memory and parses with `org.json` without a cap, which opens a DoS path for huge or deeply nested JSON. Add a size check in the filter or at the gateway.
- **Common mistakes:** Citing multipart settings, which do not apply.
- **Follow-up questions:** Where do you enforce? (Gateway first, filter as backup.)

### Q82. Is the response deterministic enough to compare in regression tests?
- **Why the interviewer asks:** Testing the contract.
- **Ideal answer:** Except `timestamp`, yes if data is stable. For regression use XML comparison ignoring order of unordered parts and the timestamp, or compare selected XPath values. Element order is also stable (Jackson declaration order). I would store golden XML for a few contracts and compare after masking timestamp. Currently no such tests exist; handler tests mock `MongoTemplate` and capture pipelines with `toPipeline`.
- **Common mistakes:** Comparing raw strings including timestamp.
- **Follow-up questions:** What changes would break the golden? (New fields like `mainReportName`.)

### Q83. What PII does the API return, and who should be allowed to call it?
- **Why the interviewer asks:** Data protection responsibility.
- **Ideal answer:** RejectRisk returns customer name and ID types/numbers (`CPGE_VI_PARTNERS_ALL` fields NAME_EXT, ID_VALUE1/2) for rejected-risk persons, plus remarks. Template12/ASP return policyholder, address and policy data. Only internal reporting callers should reach it; today that relies on network controls (Q21). Logging of bodies and responses (Q26) is the biggest accidental exposure.
- **Common mistakes:** Treating the listing as non-sensitive.
- **Follow-up questions:** Masking? (Business may need full ID for the underwriter; decision belongs to data owner.)

### Q84. Why might the same path appear under many property keys, and what is the API consequence?
- **Why the interviewer asks:** Maintenance/consistency.
- **Ideal answer:** Each LOB added its own key (plines, workmen, misc, legacy `api.pgRpgeUtils.*`), so comp-brn-number is in about eight keys and co-inward-ind in five with different spellings (`opus.fn-co-inward-ind-path` vs `opus.fn-get-co-inward-ind-path`). If the upstream path changes, N edits are needed and one may be missed, so one LOB breaks while others work. Refactor: a single base key per backend function with per-LOB aliases or just one key.
- **Common mistakes:** Merging keys blindly (some differ: `/utils` vs `/policy` segments).
- **Follow-up questions:** How to test? (A test that loads properties and asserts all aliases resolve to the same value.)

### Q85. `pg-uwge-utils.path.get-policy-ref` lacks `/ext/policy` and uses an underscore. Does it affect your modules?
- **Why the interviewer asks:** Careful reading plus scoping.
- **Ideal answer:** That key (line 163) is read by motor `PolicyRefClient`, not by the Pline clients, so it does not affect Template12/ASP/RejectRisk. It is likely stale or broken because `API_BASE-URL` is host only (inferred 404). Motor `PolicyRefClient` also builds `new RestTemplate()` privately (no timeouts, no truststore).
- **Common mistakes:** Fixing it without knowing who still uses motor `PolicyRefClient`.
- **Follow-up questions:** How would you find out? (Search usages and run a smoke call.)

### Q86. What would a good smoke-test suite for this API look like?
- **Why the interviewer asks:** Practical release confidence.
- **Ideal answer:** For each of the three report names a known-good request and checks: HTTP 200, `status` SUCCESS, `data` is well-formed XML, root `transactions`, at least one `reportData`, key elements present (RejectRisk: `rownumber`, `customerName`; Template12/ASP: `policyNo` or `endtNo`). Plus negative tests: unknown name (document the 200 behaviour), missing metadata. Run after each deployment [CONFIRM: environment and test data]. There is no such suite in the repo.
- **Common mistakes:** Only checking status code.
- **Follow-up questions:** Where to run? (CI manual dispatch exists; deploy pipeline is outside the repo.)

### Q87. How is the CI/CD pipeline related to API releases?
- **Why the interviewer asks:** DevOps awareness limited to evidence.
- **Ideal answer:** `cicd-workflow.yml` is `workflow_dispatch` only (optional `imageTag`), runs a shared action `java-microservice` (Java 17, app name `opus-gi-documentprintdata-service`, Sonar, Wiz). `sonar-code-review.yml` gates PRs with `mvn verify sonar:sonar -Dsonar.qualitygate.wait=true`. Deployment (Helm/Argo) is not in the repo. So an API release equals "build image manually, deploy by platform process"; rollback is redeploying the previous tag (inferred).
- **Common mistakes:** Describing a full CD pipeline.
- **Follow-up questions:** What identifies a release? (Image tag; pom stays `0.0.1-SNAPSHOT`.)

### Q88. The same endpoint serves Motor, Fire, Bond, Marine and Plines. What are the blast-radius risks of editing shared classes?
- **Why the interviewer asks:** Shared-code discipline.
- **Ideal answer:** Shared classes (controller, `PayLoadRequest`, `Transaction`, `ReportParameters`, `TemplateDetails`, `XmlMapper` use) affect every LOB. Plines isolates itself via subclasses and whitelists (`PlineReportParameters` with `@JsonIncludeProperties`, `PlineTransactionDetails`). Changing something like `Transaction` field order to fix XSD ordering would also change Motor/Fire/Bond XML. Check all consumers and tests first.
- **Common mistakes:** Editing the shared class "because it is the root cause".
- **Follow-up questions:** Safer approach? (Plines-specific subclass with `@JsonPropertyOrder`.)

### Q89. What does "stateless" mean for this service operationally?
- **Why the interviewer asks:** Scaling and rollback.
- **Ideal answer:** No sessions (`SessionCreationPolicy.STATELESS`), no local DB, no caches of business data, and no schema owned by this service. Any pod can serve any request, scale-out is trivial, and a code rollback has no data migration. State lives in Mongo (read-only for this service) and the external APIs. Risk is configuration drift in Helm, not data.
- **Common mistakes:** Forgetting that the truststore and base URLs are baked into the jar.
- **Follow-up questions:** What would break statelessness? (In-memory caching of indicators across requests without invalidation.)

### Q90. A consumer reports the XML has `mainReportName` and their validator fails. What do you say?
- **Why the interviewer asks:** Honest handling of contract drift.
- **Ideal answer:** Be direct: the reference XSD v0.1.6 does not define `mainReportName`/`subReportName` in `reportData`; the Java adds them (comments cite v0.1.10, a schema I could not find). Also the real output fails v0.1.6 for other reasons (dd/MM/yyyy dates, empty numeric elements, sibling order). Propose: agree the contract owner, publish the newer XSD or remove the elements, and add a CI validation test. Do not claim the output is XSD-valid.
- **Common mistakes:** Saying "it validates" or blaming the consumer.
- **Follow-up questions:** Short-term mitigation? (Consumer validates with a lax schema or ignores unknown elements, if their parser allows.)

### Q91. What happens if two requests for the same contract arrive at once?
- **Why the interviewer asks:** Concurrency reasoning.
- **Ideal answer:** They run independently on separate threads; handlers keep no shared mutable state (they are singletons with injected `MongoTemplate`; static `AGG_OPTIONS` is immutable). Each does its own reads. Different multi-step reads may see different data if an underlying record changes between steps (no snapshot across the sequential queries, noted in the DB files). Outbound clients are shared RestTemplates, thread-safe by design.
- **Common mistakes:** Worrying about locks; nothing is written.
- **Follow-up questions:** Any static mutable fields? [CONFIRM by code review before claiming none.]

### Q92. How do you change timeouts for outbound calls without a code change?
- **Why the interviewer asks:** Config operations.
- **Ideal answer:** Properties `app.http.connect-timeout-ms` (5000), `app.http.read-timeout-ms` (20000), `app.http.socket-timeout-ms` (20000), `app.http.connection-request-timeout-ms` (3000), `app.http.fast-lookup.read-timeout-ms` (8000). Override via env var `APP_HTTP_READ_TIMEOUT_MS` (relaxed binding; inferred) or an external config file. Beans are built at startup, so a restart is needed. Remember the code defaults differ for motor beans (10000/120000/120000).
- **Common mistakes:** Expecting a refresh without restart.
- **Follow-up questions:** Which property would you lower first for a flaky upstream? (`fast-lookup.read-timeout-ms`.)

### Q93. What are the risks of `System.out.println` in a controller and filter?
- **Why the interviewer asks:** Code quality plus compliance.
- **Ideal answer:** It bypasses logging levels and formats, cannot be turned off by config, may flood stdout, and prints PII. Sonar would flag it. Replace with `@Slf4j` and a DEBUG log that excludes bodies. Examples: `CustomRequestFilter` prints bodies; `CommonController` prints "Transaction object is null".
- **Common mistakes:** Saying it is harmless because it is "just stdout".
- **Follow-up questions:** How to catch it in CI? (Sonar rule S106, checkstyle.)

### Q94. How do you verify that the HTTP API behaves the same after the Spring Boot 4 upgrade?
- **Why the interviewer asks:** Upgrade regression thinking (commit `9248b12ff`, OPUSM-48579).
- **Ideal answer:** Capture requests/responses for the three reports before the upgrade (golden), run after, and compare XML ignoring `timestamp`. Check the specific upgrade risks: Jackson 2 vs 3 converter (`preferred-json-mapper=jackson2`), `spring.mongodb.*` prefix (old keys fall back to localhost), Jackson pins (`jackson-2-bom.version`), springdoc 3.x paths, and actuator exposure. Unit tests alone mock Mongo and would not catch the prefix issue.
- **Common mistakes:** Trusting a green build.
- **Follow-up questions:** What symptom shows the Mongo prefix problem? (Connection to localhost:27017 / empty data.)

### Q95. What is the role of `GroupedOpenApi` and why only `/api/**`?
- **Why the interviewer asks:** Knows what is documented.
- **Ideal answer:** `OpenApiConfig` registers group "public" with `pathsToMatch("/api/**")`, so docs list only the `GenerateReport` endpoints and exclude actuator. The grouped document is at `/v3/api-docs/public`. springdoc keys: `api-docs.path=/v3/api-docs`, `swagger-ui.path=/swagger-ui/index.html`, operations sorted by method.
- **Common mistakes:** Expecting per-report schemas in Swagger.
- **Follow-up questions:** How could you document report-specific keys? (`@Schema` examples or per-report OpenAPI descriptions.)

### Q96. What do you do if an upstream path starts returning 404 after an API-team release?
- **Why the interviewer asks:** Incident handling.
- **Ideal answer:** Symptoms: blank indicators/BRN with log lines "GET failed" and status 404. Steps: confirm in logs, call the URL by hand against the same host, compare with the API team's current catalog, update the property key used by the client (check the right alias among duplicates), redeploy or override via config. Add a smoke check so the next time it is caught earlier. Mention that fail-soft hid the failure.
- **Common mistakes:** Changing the first key found instead of the one the class reads.
- **Follow-up questions:** Which keys does `PlineUtilsClient` require at startup? (`opus.fn-decide-mt-cert-path`, `opus.fn-uwge-get-endtschd-query-path`, no defaults.)

### Q97. Can a caller reach other LOB reports through the same endpoint, and is that a concern for the Plines team?
- **Why the interviewer asks:** Boundary thinking.
- **Ideal answer:** Yes: any `reportName` bean or switch case is reachable (Motor, Fire, Bond...). Without authorisation by report, any caller who can reach the pod can request any LOB report. For Plines it means a RejectRisk request is not restricted to Plines users. Real fix is gateway-level authorisation per report or a per-report permission check in the controller.
- **Common mistakes:** Believing LOBs are separated by deployment (they are one jar).
- **Follow-up questions:** Is it one deployment? (One Maven artifact `gi-documentprintdata`; deployments elsewhere unknown.)

### Q98. Explain `createDefaultTransaction` side effects with `templateDetails`/`mailingDetails`.
- **Why the interviewer asks:** Detail on the response in the no-data path.
- **Ideal answer:** It copies metadata into the `MailingDetails` and `TemplateDetails` so the print engine still receives routing information (channel, output format, template ids, report name) even when no data exists. That lets the downstream system decide to print a "no data found" page (the JRXML `whenNoDataType="AllSectionsNoDetail"` for RejectRisk prints title and headers only). The XML thus carries routing but no `transactionDetails`.
- **Common mistakes:** Believing the response body is empty.
- **Follow-up questions:** What does the print engine show? (Not evidenced; Jasper original shows "No Data Found".)

### Q99. How would you explain this API's biggest weakness to a non-technical manager?
- **Why the interviewer asks:** Communication skill.
- **Ideal answer:** "The service says everything is fine even when it could not fetch all the data. If a partner system is slow or a report name is mistyped, the document can come out blank or incomplete and nobody gets an error. Also anyone on the network can ask for any report. The fixes are small: clearer errors, alerts on blank results, and a gateway check." Keep it plain and specific.
- **Common mistakes:** Jargon (SSRF, ProblemDetail) without impact.
- **Follow-up questions:** What would you ask for in budget/time? Which fix gives most risk reduction per day of work?

### Q100. Walk through one full request for ASP EndtDocument from HTTP to XML.
- **Why the interviewer asks:** Integration of everything above; common closing question.
- **Ideal answer:** (1) POST with `reportName=AGIC/UW/Plines/ASPEndtSchedule/EndtDocument`, `pContractId`, `pEndtNo`, `pRefNo`, `versionNo`, `show*` flags. (2) `CustomRequestFilter` rewrites arrays. (3) Controller normalises name, finds the `AspEndtDocumentService` bean. (4) Service reads keys with defaults, calls main handler(s) that run Mongo aggregates (for example `UWGE_VI_ENDT_REPORT_PARAM`, `UWMT_VI_ENDT_TAX_INV_ALT_UPV`, WIP/SB contracts) and fail-soft utility calls (e-invoice endt indicator, BRN, signature). (5) Data-driven gates decide which sub-reports (cover note, endorsement schedule, complaint, debit note, tax invoice) are built; `PlineReportNames.tag` stamps names. (6) Controller adds mailing/template details, writes XML with `XmlMapper`, wraps it into `SuccessResponse`, returns 200. ASP's expansion is not stated in the code [CONFIRM: business meaning of ASP].
- **Common mistakes:** Skipping the data-driven gates or the fail-soft behaviour.
- **Follow-up questions:** Where would you add a new sub-report? Where does `checklistCopy=Y` change the data source? (WIP collections.)
