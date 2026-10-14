# 08d - Technical Questions 51-100 (Jackson / XML / XSD / Spring Boot / Maven / testing as used in THIS code)

Conventions: facts come from the KT guide and KT_Parts 01-06 and the repo; "(inferred)" = reasoning, not run; "[CONFIRM: ...]" = personal fact.

---

### Q51. How is the XML produced, and why do null values become `<x/>`?
- **Why the interviewer asks:** This explains hundreds of XSD errors.
- **Ideal answer:** `CommonController` uses a plain `new XmlMapper()` and `writeValueAsString(transactions)`. The `Pline*` payload classes use `@JsonInclude(JsonInclude.Include.ALWAYS)` (for example `PlineReportData`), so null fields are written as empty elements. For `xs:string` that is valid; for `xs:decimal`, `xs:dateTime`, `xs:integer`, `xs:boolean` an empty element is invalid, and nothing is `nillable`. In the old saved RejectRisk sample, 456 of 929 errors were empty numeric/date elements.
- **Common mistakes:** Thinking empty equals absent.
- **Follow-up questions:** Fix options? (`NON_NULL`/`NON_EMPTY` on those classes, or `xsi:nil` plus `nillable`.)

### Q52. What do `@JacksonXmlProperty` and `useWrapping=false` do in the payload classes?
- **Why the interviewer asks:** XML mapping with Jackson.
- **Ideal answer:** `@JacksonXmlProperty(localName="Columncount")` sets the exact element name (needed for XSD names like `Columncount`, `Emailbcc`, `LOADING`, `PREM`, `sumTotalPremiumMEDIX`, `TableHeaderCOUNT`). For lists that the XSD declares as repeated siblings (`transDet`, `reportData`, `risk`, `cover`), `@JacksonXmlElementWrapper(useWrapping=false)` avoids an extra wrapper element. Without it you get `<transDets><transDet>..` which breaks the XSD. Lombok getters alone would emit lower-case names.
- **Common mistakes:** Forgetting the wrapper setting and getting double nesting.
- **Follow-up questions:** Example of two names differing only by case? (`LOADING` vs `loading`; Java uses `loading` and `loadingUpper`.)

### Q53. How do you make a `LocalDateTime` serialise as ISO-8601 for the XSD?
- **Why the interviewer asks:** Technical fix for the known type mismatch.
- **Ideal answer:** Today fields use `@JsonFormat(pattern="dd/MM/yyyy")` with `LocalDateTimeSerializer`, giving `06/06/2017`. XSD needs `2017-06-06T00:00:00`, i.e. `DateTimeFormatter.ISO_LOCAL_DATE_TIME`. Change the pattern (or remove it with `JavaTimeModule` and `WRITE_DATES_AS_TIMESTAMPS` off), but then the Jasper report, which reads `dd/MM/yyyy` text, must parse ISO. Do it only after the contract is agreed. Affects 64 dateTime leaves in Pline.xsd.
- **Common mistakes:** Changing the format and breaking the renderer.
- **Follow-up questions:** Per-class or global? (Per-class to limit blast radius; the Motor classes share the base.)

### Q54. What does `@JsonIncludeProperties` do in `PlineReportParameters`?
- **Why the interviewer asks:** A silent filter that drops data.
- **Ideal answer:** It is a whitelist of 73 JSON property names. Only those names are serialised. It prevents the shared Motor `ReportParameters` fields (many) from leaking into Plines XML. Side effect: setters for non-listed properties are silently ignored in the output (`pContractId`, `pPolicyNo`, `pCertNoFrom/To`, `pTaxInvNo`, `reportName`, `ShowOutPrintCovLetter`). Six whitelisted names are not in v0.1.6 (`pCreatedDateFrom`, `pCreatedDateTo`, `pReportTitle`, `pshowEndtSchedule`, `pShowOutPrintCovLetter`, `pShowPDPANotice`).
- **Common mistakes:** Adding a setter call and assuming it reaches the XML.
- **Follow-up questions:** Test to catch drops? (Serialise and assert the element is present.)

### Q55. Why is element order a Jackson problem for the XSD?
- **Why the interviewer asks:** Links Jackson behaviour to `xs:sequence`.
- **Ideal answer:** Pline.xsd is entirely `xs:sequence`, so order is enforced. Jackson writes properties in field declaration order unless `@JsonPropertyOrder` is present. Only `PlinePrntDetails`, `PlineTransDetList`, `PlineTransactionDetails`, `PlinePaymentsType` have it. Known mismatches: `transaction` (Java writes templateDetails, mailingDetails, transactionDetails, reportParameters; XSD wants reportParameters first), `address` last in agent/branch/comp/taxClient types (XSD first), `clientInfo`, `buyerInfo`, `supplierInfo`, `cover`, `risk`, `transDet`, `reportData` (unknown `mainReportName`).
- **Common mistakes:** Believing XML order does not matter for validation.
- **Follow-up questions:** Risk of fixing the shared `Transaction` class? (Motor/Fire/Bond share it; check blast radius.)

### Q56. Explain the `Policyletterind` and `LOADING` naming tricks.
- **Why the interviewer asks:** Field visibility and naming control in Jackson.
- **Ideal answer:** The XSD element is `Policyletterind` (capital P), but a Lombok getter `getPolicyletterind` becomes `policyletterind`. `PlineReportData` uses field visibility so the exact field name is emitted. Where names differ more (`Columncount`, `LOADING`), `@JacksonXmlProperty(localName=...)` is used. XML is case sensitive, so `LOADING` and `loading` are different elements.
- **Common mistakes:** Relying on getter names.
- **Follow-up questions:** What is `@JsonAutoDetect`? (Controls which members Jackson sees; field visibility ANY and getters NONE.)

### Q57. What are "shadow fields" with `@JsonIgnoreProperties` in `PlineReportParameters`?
- **Why the interviewer asks:** Working around a shared base class.
- **Ideal answer:** The base class has properties with names that the Plines XSD does not use (for example `showCoverNote`, `showTaxInvoice`, `pshowGstLabel`). The subclass declares its own fields with the XSD names (`pshowCoverNote`, `pshowTaxInvoice`, `pShowGstLabel`) and uses `@JsonIgnoreProperties` to hide the base ones, so only XSD-correct names are emitted. It also overrides setters (`setPshowComplaint`). Gap: ASP calls base setters `setPshowGstLabel/SstLabel`, which are ignored, so ASP label flags may not reach the XML (unverified).
- **Common mistakes:** Calling the wrong setter casing (`setPShow...` vs `setPshow...`).
- **Follow-up questions:** Cleaner design? (Plines-only parameters class, not extending the base.)

### Q58. What changed with Jackson 3 in Boot 4, and why does this code still compile?
- **Why the interviewer asks:** Migration awareness.
- **Ideal answer:** Jackson 3 uses `tools.jackson` packages (annotations stay in `com.fasterxml.jackson.annotation`, but the databind classes move; the repo uses Jackson 2 databind and XML). Boot 4.1.1 has both on the classpath and prefers Jackson 3 for HTTP conversion; `spring.http.converters.preferred-json-mapper=jackson2` forces Jackson 2. `jackson-2-bom.version=2.21.5` pins Jackson 2, while `jackson-bom.version` now refers to Jackson 3. `JacksonConfig` defines a plain `ObjectMapper` with `JavaTimeModule`, so `spring.jackson.*` keys do not apply to it. [Caveat: the exact annotation/package detail of Jackson 3 is general knowledge, not read from the repo.]
- **Common mistakes:** Setting `jackson-bom.version` to pin Jackson 2.
- **Follow-up questions:** What would the migration cost? (Move databind/XML usage and annotations in 300+ payload classes - inferred.)

### Q59. How does `PayLoadRequest.setMetadata` work and what does it drop?
- **Why the interviewer asks:** Custom binding.
- **Ideal answer:** `metadata` is a `Map<String,String>` built from a raw `Map`: String and Number values are kept (numbers become strings), a list `mergeStaticPdfFiles` inside metadata is lifted to a field, and booleans, nulls and nested objects are silently dropped. `filterCondition` is `Map<String,String>` too. Jackson names the field `filterCondition` because of Lombok `getFilterCondition()`, although the field is declared `FilterCondition`.
- **Common mistakes:** Sending booleans in metadata and expecting them.
- **Follow-up questions:** What if `metadata` is missing? (`populateMailingDetails` dereferences it, NPE -> 500; inferred.)

### Q60. How do `SuccessResponse` and the XML string interact?
- **Why the interviewer asks:** Response format detail.
- **Ideal answer:** `SuccessResponse` has `status`, `data` (XML text) and `timestamp` (`LocalDateTime.now().toString()`). The XML is a JSON string, so quotes and angle brackets are escaped by the JSON layer, and the consumer must unescape (normal JSON parse) before XML parsing. The README says `xmlData` but the class says `data`; check a live response. A large report means a large single string in memory.
- **Common mistakes:** Treating `data` as nested JSON.
- **Follow-up questions:** Would you stream it? (The wrapper makes that hard; could return `application/xml` directly in a new version.)

### Q61. How does `SecurityConfig` work, and how would you add a minimal API-key check?
- **Why the interviewer asks:** Security coding on the real class.
- **Ideal answer:** `SecurityFilterChain` bean: stateless session, CSRF ignoring `/**`, `/actuator/health` and `/info` `permitAll`, other `/actuator/**` `denyAll`, `anyRequest().permitAll()`, and an empty `InMemoryUserDetailsManager` to avoid the generated password. To add a key check: write a `OncePerRequestFilter` that compares a header with a secret from env/secret store (constant-time compare), register it with `http.addFilterBefore(...)`, change to `anyRequest().authenticated()`, keep health open. Spring Security 7.1.1 uses the lambda DSL.
- **Common mistakes:** Putting the secret in `application.properties` (the repo already has that problem).
- **Follow-up questions:** JWT instead? (Resource server config with issuer from the gateway.)

### Q62. How are exceptions mapped to HTTP responses?
- **Why the interviewer asks:** Exception advice details.
- **Ideal answer:** `GlobalExceptionHandler` (`@RestControllerAdvice`): `NoDataFoundException` -> 404 ProblemDetail (never thrown today); any `Exception` -> 500 ProblemDetail `INTERNAL_ERROR`; it does not log the exception. `GenericExceptionHandler`: `IllegalArgumentException` -> 400 `BAD_REQUEST`. Handler selection is by specificity. Because `@ExceptionHandler(Exception.class)` matches broadly, malformed JSON, unsupported media type or method likely become 500 instead of 400/415 (inferred). Fix: handle `HttpMessageNotReadableException` and add `log.error`.
- **Common mistakes:** Assuming Spring's default 400 still applies.
- **Follow-up questions:** Why does the 500 hide the root cause? (No logging in the handler.)

### Q63. Springdoc 3.1.1 and `GroupedOpenApi`: what is configured?
- **Why the interviewer asks:** Boot 4 compatibility detail.
- **Ideal answer:** `OpenApiConfig` registers `GroupedOpenApi` named `public` with `pathsToMatch("/api/**")`, docs at `/v3/api-docs` and `/v3/api-docs/public`, UI at `/swagger-ui/index.html` (under the `/gi/documentprintdata` context path). springdoc 2.8.15 failed on Boot 4 with `NoClassDefFoundError` for a relocated `WebMvcProperties`, so 3.1.1 is pinned. Security `permitAll` leaves docs open; consider disabling in prod (`springdoc.api-docs.enabled=false`).
- **Common mistakes:** Forgetting the context path in the URL.
- **Follow-up questions:** How do you disable only in prod? (Profile-specific property - but no real profiles exist yet.)

### Q64. What does actuator expose and why is `loggers` pointless?
- **Why the interviewer asks:** Reading config against security rules.
- **Ideal answer:** `management.endpoints.web.exposure.include=health,info`. `management.endpoint.loggers.enabled=true` is set but `loggers` is not exposed and `SecurityConfig` denies other actuator paths, so you cannot change log levels at runtime. Health has no `show-details`, so it shows only the status. Probes probably use `/gi/documentprintdata/actuator/health` (inferred; Helm repo not seen).
- **Common mistakes:** Calling `/actuator/health` without the context path.
- **Follow-up questions:** How to add Prometheus? (Add the registry dependency, expose `prometheus`, and open it only to the scraper.)

### Q65. How does Maven pick the version of a library here?
- **Why the interviewer asks:** Build knowledge.
- **Ideal answer:** `spring-boot-starter-parent 4.1.1` imports the BOM. A dependency without version uses the BOM version. Properties override BOM versions: `tomcat.version=11.0.25`, `jackson-2-bom.version=2.21.5`, `logback.version=1.5.34`, `log4j2.version=2.25.5`. Explicit `<version>` on a dependency overrides too (`commons-lang3 3.18.0`, `springdoc 3.1.1`, `org.json 20231013`, `httpcore5 5.4.3`, `junit-jupiter 5.9.0`). Danger: a pin lower than the BOM is a downgrade (commons-lang3, logback, junit). Check with `mvn dependency:tree`.
- **Common mistakes:** Thinking property names are the same across Boot majors (`jackson-bom.version` changed meaning).
- **Follow-up questions:** How to find unused pins? (Compare `dependency:tree` with the BOM and remove pins at or below the BOM.)

### Q66. How can a class in `src/test` hide a class in `src/main`?
- **Why the interviewer asks:** Real defect in the repo.
- **Ideal answer:** Maven puts `target/test-classes` before `target/classes` on the test classpath. `src/test/.../documentHandler/AspServiceTaxInvoiceHandler.java` has the same fully qualified name as the main class, so tests (27 in `AspServiceTaxInvoiceHandlerTest`) can load the copy. It compiles because test compilation sees main classes as a dependency. Surefire's default includes `*Test.java`, so the copy itself is not run. JaCoCo reads `target/classes`, so production coverage can look lower than the tests suggest (inferred). Delete the copy.
- **Common mistakes:** Believing duplicate classes always fail the build.
- **Follow-up questions:** How to prove which class loaded? (Log the class's code source in a test.)

### Q67. How do JaCoCo and Sonar connect in the pom and workflow?
- **Why the interviewer asks:** CI/test tooling detail.
- **Ideal answer:** `jacoco-maven-plugin 0.8.12`: `prepare-agent` before tests and `report` bound to the `test` phase, writing `target/site/jacoco/jacoco.xml`. Properties: `sonar.coverage.jacoco.xmlReportPaths`, `sonar.junit.reportPaths=target/surefire-reports`. The workflow runs `mvn -B verify org.sonarsource.scanner.maven:sonar-maven-plugin:sonar -Dsonar.qualitygate.wait=true`. No JaCoCo `check` goal, no exclusions, so DTO and payload tests count toward coverage. No `target/site/jacoco` exists locally, so coverage for the three modules is unknown.
- **Common mistakes:** Quoting a coverage percentage.
- **Follow-up questions:** How to exclude Lombok code? (`lombok.config` with `addLombokGeneratedAnnotation = true`.)

### Q68. Why does the Maven wrapper matter here?
- **Why the interviewer asks:** Reproducible builds.
- **Ideal answer:** `.mvn/wrapper/maven-wrapper.properties`: wrapper 3.3.4, Maven 3.9.11, `distributionType=only-script`. It pins the Maven version for all developers and CI without installing Maven. The OpenRewrite plugin 6.46.1 is present but has no active recipes (6.48.0 was not resolvable on the Nexus mirror). The pom artifactId `gi-documentprintdata` has stray spaces (cosmetic).
- **Common mistakes:** Mixing a local Maven with another version and getting different results.
- **Follow-up questions:** Why a Nexus mirror issue? (Corporate repository only has approved versions - inferred.)

### Q69. Why did the micrometer pin break `@SpringBootTest`?
- **Why the interviewer asks:** Version-conflict experience.
- **Ideal answer:** The pom once pinned micrometer to 1.15.12 (below the Boot 4 BOM 1.17.1). Boot 4 classes such as `JvmMemoryMeterConventions` expected the newer API, so `ApplicationContext` startup failed for every `@SpringBootTest`. The pin was removed (pom comment). Lesson: do not pin libraries managed by the BOM unless a CVE needs it, and re-check pins after every Boot upgrade.
- **Common mistakes:** Debugging the test instead of the dependency tree.
- **Follow-up questions:** Other conflict story? (springdoc 2.8.15 vs Boot 4; `jackson-bom.version` meaning change.)

### Q70. Which JUnit version really runs, and why does it matter?
- **Why the interviewer asks:** Attention to build detail.
- **Ideal answer:** The pom declares `junit-jupiter 5.9.0` explicitly while the Boot 4.1.1 BOM manages 6.0.3; the effective engine is probably 6.0.3 (inferred from `~/.m2`). A mixed Jupiter API/engine/platform set can cause `NoSuchMethodError` or discovery failures. Verify with `mvn dependency:tree -Dincludes=org.junit.jupiter,org.junit.platform`. No test run exists on this machine, so compile and pass status are unknown.
- **Common mistakes:** Assuming the explicit version wins everywhere.
- **Follow-up questions:** Fix? (Remove the explicit version and use the BOM.)

### Q71. How do you validate XML against Pline.xsd in Java?
- **Why the interviewer asks:** Direct XSD technical skill.
- **Ideal answer:** `SchemaFactory.newInstance(XMLConstants.W3C_XML_SCHEMA_NS_URI)`, `newSchema(new StreamSource(file))`, `schema.newValidator()`, then `validate(new StreamSource(new StringReader(xml)))`. The schema source must be a File or URL with a system id so the relative `xs:include schemaLocation="Common.xsd"` resolves; an `InputStream` has no base URI (the test converts the classpath URL to a `File`). Use `validator.setErrorHandler` to collect all errors; the default stops at the first fatal error.
- **Common mistakes:** Loading the schema from a stream and failing to resolve the include.
- **Follow-up questions:** Alternative to fix the base URI? (`LSResourceResolver` or `setSystemId`.)

### Q72. `xs:include` versus `xs:import` in this schema pair.
- **Why the interviewer asks:** XSD structure.
- **Ideal answer:** `Pline.xsd` uses `xs:include` of `Common.xsd`. Both have no `targetNamespace`, so include works (chameleon include is not needed). `xs:import` is for a different namespace. `elementFormDefault="qualified"` has no effect without a namespace. So the XML must have no `xmlns`. If a namespace is added later, the output needs a prefix and `import`, and Jackson root needs `@JacksonXmlRootElement(namespace=...)`.
- **Common mistakes:** Adding `xmlns` in the output.
- **Follow-up questions:** Why avoid `xs:redefine`? (Deprecated.)

### Q73. What does a Xerces error like `cvc-complex-type.2.4.a` mean here?
- **Why the interviewer asks:** Reading validator output.
- **Ideal answer:** "Invalid content was found starting with element X. One of '{...}' is expected" means a sibling is out of sequence or unknown. Real example: `mainReportName` in `reportData` where the expected list starts with `permanentDisablement, Policyletterind, ...`. `cvc-datatype-valid.1.2.1` plus `cvc-type.3.1.3` appear as a pair for an invalid value, such as `<createdDate>06/06/2017</createdDate>` or an empty decimal. Xerces reports roughly one structure error per container, so the structure count is a lower bound.
- **Common mistakes:** Fixing the first error only and not re-running.
- **Follow-up questions:** How to count errors usefully? (Group by message pattern.)

### Q74. How is a schema change rolled out safely?
- **Why the interviewer asks:** Contract evolution.
- **Ideal answer:** Additive optional leaf: old consumers keyed by name ignore it; strict old validators reject it (forward-incompatible), newer schemas accept old files. Move, rename, retype, remove: breaking (v0.1.3 moved `prntDetails`; v0.1.4 removed `showCoverNote`, `showTaxInvoice`, `prem`, `totalpremium` at old places; v0.1.6 retyped `pBeforeEffDate`). Process: update XSD in alphabetical slot, same position and type in `PlineTransDet`, populate in the service, update the mapping sheet and header changelog, bump version, add tests, tell the consumer first. Note the repo added elements (`mainReportName`) ahead of the XSD - a governance gap.
- **Common mistakes:** "Adding is always safe".
- **Follow-up questions:** Who owns the version number? [CONFIRM: your team.]

### Q75. Why were `COLUMN_COUNT` and `EMAIL_BCC` renamed?
- **Why the interviewer asks:** XML naming rules.
- **Ideal answer:** Commit `6961abbf9` renamed to XML-legal leaf names: `COLUMN_COUNT` -> `Columncount`, `EMAIL_BCC` -> `Emailbcc`, `sumTotalPremium_MEDIX` -> `sumTotalPremiumMEDIX`, `TableHeader_COUNT` -> `TableHeaderCOUNT`, `policyLetterInd` -> `Policyletterind`. The header comment still shows the Jasper spellings, so do not repeat those. XML names allow underscores, so the reason was naming consistency with the generated schema style (inferred); the key rule is that Java must emit the exact XSD name, including case.
- **Common mistakes:** Quoting the header spelling as the real element name.
- **Follow-up questions:** How do you check names? (grep the XSD and compare with `@JacksonXmlProperty`.)

### Q76. How would you write a drift guard between XSD and Java classes?
- **Why the interviewer asks:** Automation of the audit.
- **Ideal answer:** Parse the XSD (DOM or `XSOM`) to get element names, types and order per complex type; use reflection on `Pline*` classes, mapping `@JacksonXmlProperty` names and `@JsonIgnore`, then compare name sets and order, and type pairs (String-xs:string, BigDecimal-xs:decimal, Long/Integer-xs:integer, LocalDateTime-xs:dateTime, Boolean-xs:boolean). This replaces the manual awk audit that found 12 Java-only fields and 0 XSD-only leaves. Also known type mismatch: `PlineBenefit.benefitSi` is `String` while the XSD says decimal.
- **Common mistakes:** Comparing only names.
- **Follow-up questions:** Why not generate classes from the XSD (`xjc`)? (Existing Jackson-XML classes and shared Motor types.)

### Q77. Are the service classes thread-safe?
- **Why the interviewer asks:** Concurrency on singletons.
- **Ideal answer:** Services and handlers are Spring singletons. They should be stateless: all request data lives in local variables (`filter`, `PlineReportData`, `Transactions`). The risk is any instance field set per request, or a static non-thread-safe helper such as a shared `SimpleDateFormat`. Within one request, shared objects (`prntDetails`, `endorsementDetails`, `transDetList`) are built once and shared by reference across several `reportData` (ASP landscape rows, Template12 landscape rows), so they must not be mutated by other threads (they are not shared across requests). [CONFIRM: you checked fields in your classes.]
- **Common mistakes:** Saying singleton means unsafe.
- **Follow-up questions:** What would break under concurrency? (A static mutable cache.)

### Q78. What goes wrong when rows share `endorsementDetails` by reference?
- **Why the interviewer asks:** Mutable object sharing.
- **Ideal answer:** `toLandscapeReportData` (Template12) and `landscapeRow` (ASP) reuse the master `branchInfo`, `compInfo` and the same `endorsementDetails` object. `mergeEndorsement` is "last non-null wins", so every row sees the final merged values, not the value at the time of the row. If a later call changes a field, earlier rows change as well. Fix: copy per row (copy constructor or builder), or freeze after the merge. The code comment says the same.
- **Common mistakes:** Assuming each `PlineReportData` has independent children.
- **Follow-up questions:** How would you test it? (Mutate after creating row 1 and assert row 1 is unchanged.)

### Q79. How would you build a Mongo integration test for RejectRisk?
- **Why the interviewer asks:** Fill the biggest test gap.
- **Ideal answer:** Use Testcontainers with a real MongoDB 6/7 image (embedded fakes may not support `$setWindowFields` or `$lookup` with `let` - inferred). Seed `CMUW_PL_REFER_RISK` rows at `D 00:00:00`, `D 23:59:59.999`, `D+1 00:00:00`; `CPGE_VI_PARTNERS_ALL` with one null-name partner; `CMGE_CODE` rows for the three categories. Cases: from = to = D; only from; only to; neither; null-name ordering; `pIdNo` matching `ID_VALUE2`; type mismatch. Run the real `RejectRiskHandler` with a real `MongoTemplate`. Mark with a tag so it can run separately from unit tests. The repo has no Testcontainers today.
- **Common mistakes:** Mocking `MongoTemplate` again.
- **Follow-up questions:** Time zone test? (Seed 2026-01-04T16:30:00Z and decide the expected day with the business.)

### Q80. How would you add a new `transDet` leaf end to end?
- **Why the interviewer asks:** The full vertical change.
- **Ideal answer:** (1) Confirm with the schema owner; add the element in `Pline.xsd` at the alphabetical slot with the right type; bump version/changelog. (2) Add the field in `PlineTransDet` in the same position and type (`@JacksonXmlProperty` if case differs). (3) Add it to the DTO and `$project` stage. (4) Map it in the service/base `add*`. (5) If it is a merge field, check `mergeNonNullFields`. (6) Update the mapping sheet. (7) Tests: handler pipeline, mapping, and a serialise-and-validate test. Mistakes to avoid: wrong position, wrong case, forgetting the Motor base class is shared.
- **Common mistakes:** Updating Java but not the XSD (or reverse).
- **Follow-up questions:** For a new `reportParameters` leaf? (Also add to the `@JsonIncludeProperties` whitelist.)

### Q81. Can handlers run in parallel inside one request?
- **Why the interviewer asks:** Performance design.
- **Ideal answer:** Many independent handlers (Template12 default path can call 10+; ASP landscape dispatches) could run in parallel with `CompletableFuture` on a bounded executor. `MongoTemplate` is thread-safe. Constraints: the shared `PlineReportData`/`transDet` merges are not thread-safe and order matters (last non-null wins), so collect results first and merge sequentially. Pool limits matter (Mongo driver pool 100 default; HTTP pool 100/20). Use only for REST calls and read-only fetches. I have no timing data (not measured).
- **Common mistakes:** Parallelising the merge steps.
- **Follow-up questions:** Which calls first? (The five external REST calls with 8 s timeouts.)

### Q82. How would you cache static lookups like `SAPM_SYS_CONSTANTS` and `CMGE_CODE`?
- **Why the interviewer asks:** Optimisation with correctness.
- **Ideal answer:** These tables change rarely: `ASP_CUTOFF_DATE`, `PIDM_CUTOFF_POL`, `OCCUP_CUT_OFF_DATE`, `EINV_CUT_OFF`, `CMGE_CODE` descriptions. Use Spring `@Cacheable` (Caffeine) with a short TTL (minutes) and a key of `(SYS_TYPE, SYS_CODE)` or `(CAT_CODE, CODE_CD)`. For RejectRisk, load the three code categories into a map once per request instead of three lookups per row. Risks: stale values after a data fix; multi-pod inconsistency within the TTL; null caching. Measure before doing it.
- **Common mistakes:** Caching per-contract data.
- **Follow-up questions:** How to invalidate? (TTL or a manual evict endpoint behind auth.)

### Q83. How would you add retry and a circuit breaker to the REST clients?
- **Why the interviewer asks:** Resilience design.
- **Ideal answer:** Now: no retry, failures swallowed to default values (for example e-invoice "N"). Using Resilience4j: a time limiter around the call (already ~8 s), a small retry (1-2, with jitter) for GETs, and a circuit breaker that opens after repeated failures. The key decision is what the default should be: for gate-controlling calls (`fnGetEinvoicePrintPolInd`, `fnShowStaxInvInd`) a silent "N" removes invoice pages, so prefer failing the request or marking the response as degraded. Add a metric and alert.
- **Common mistakes:** Retrying on 4xx.
- **Follow-up questions:** Idempotent? (GET calls are, so retrying is safe.)

### Q84. Why is `Include.ALWAYS` plus default placeholder objects a payload-size problem?
- **Why the interviewer asks:** Payload analysis.
- **Ideal answer:** `transDet` has about 650 leaves and `ALWAYS` writes all of them for each row, mostly empty. `PlineHeaderTransaction.reportTransaction` and `PlineReportTransaction.reportData` default to one empty object each, so even empty responses contain an empty `reportData` with many empty elements. RejectRisk builds one `transDetList` per row plus `companyType` nesting, which bloats XML further. Options: `NON_NULL` plus schema-valid optionality (all elements are optional, so omission is legal), and remove default placeholders.
- **Common mistakes:** Thinking the consumer needs the empty elements (unknown - ask).
- **Follow-up questions:** Risk of omitting empties? (The renderer might expect the element to exist.)

### Q85. What is `$setWindowFields` on a large result doing to memory, and how would you protect it?
- **Why the interviewer asks:** Aggregation memory model.
- **Ideal answer:** `$sort` and `$setWindowFields` are blocking stages with a 100 MB limit each unless `allowDiskUse` is true. `RejectRiskHandler` has no `AggregationOptions`. With no-dates-returns-all, a large collection can raise "Sort exceeded memory limit" (inferred). Protect: require a date range, add `allowDiskUse(true)` and a sane `maxTime`, push selective `$match` stages first (already done for date/type/reason), cap rows, or assign row numbers in Java after the sort.
- **Common mistakes:** Adding `$limit` after the window stage (too late).
- **Follow-up questions:** Index help for `$sort`? (Only if the sort key is the leading index field before blocking; here the key is a joined field, so no.)

### Q86. How should `NumberFormatException` on `pContractId` be turned into a clean 400?
- **Why the interviewer asks:** API validation design.
- **Ideal answer:** Validate once at the start of `execute`: if `pContractId` is blank or not numeric, throw `IllegalArgumentException("pContractId must be numeric")`; `GenericExceptionHandler` maps it to 400 `BAD_REQUEST` with a generic detail. Alternatively add Bean Validation (`spring-boot-starter-validation` is on the classpath but `PayLoadRequest` has no constraints and the controller has no `@Valid`). Also log the message at WARN, because the handler logs only the message. Today the empty value gives a 500 from the landscape dispatch.
- **Common mistakes:** Catching `NumberFormatException` deep in a handler and returning a partial report.
- **Follow-up questions:** Where do you keep the rule? (One shared validator for all three modules.)

### Q87. What does `getOrDefault` hide when reading `filterCondition`?
- **Why the interviewer asks:** Edge-case in request handling.
- **Ideal answer:** `getOrDefault(key, default)` only applies when the key is absent. A key present with an empty string stays empty (for example `"showComplaint": ""` is not "Y"). Keys are case-sensitive (`ShowOutPrintCovLetter`, not `showOutPrintCovLetter`). A JSON null cannot exist in a `Map<String,String>` after custom binding (nulls dropped in metadata; for filterCondition behaviour may differ - inferred). Defaults differ across modules (`showComplaint` "Y" in Template12; ASP default "N" and only echoed).
- **Common mistakes:** Assuming an empty value falls back to the default.
- **Follow-up questions:** Safer helper? (`valueOrDefault` that treats blank as missing.)

### Q88. How does `PlineReportNames.tag` work?
- **Why the interviewer asks:** Newest feature in the code.
- **Ideal answer:** `tag(transactions, getClass())` walks all `reportData` and fills a blank `mainReportName` derived by `mainNameOf` from the `@Service` bean path: parent folder segment + last segment unless the last already starts with the parent (for example `Template12PolicyDocument`, `ASPEndtScheduleEndtDocument`, `RejectRisk`). Only blanks are filled, so services that set names themselves (ASP sets four constants) are kept. `subReportName` is set by services/`PlineReportBlocks`. Test: RejectRisk row `mainReportName=RejectRisk`, `subReportName=null`.
- **Common mistakes:** Believing it overwrites existing names.
- **Follow-up questions:** Why not in the XSD? (The v0.1.6 on this machine has no such elements.)

### Q89. Why is the `RestTemplate` truststore a deployment risk?
- **Why the interviewer asks:** TLS config consequences.
- **Ideal answer:** The service loads `mytruststore.p12` from the classpath (committed binary) with the password in `application-env.properties` in plain text (`<redacted>`). `loadTrustMaterial(trustStore, null)` replaces JVM default cacerts, so a new upstream with a CA not in the file fails the handshake. CA rotation needs a rebuild unless overridden. The Mongo connection (`ssl=true`) uses the JVM default trust, not this file. Fix: mount the truststore and password from a secret at deploy time.
- **Common mistakes:** Thinking the file is used for Mongo.
- **Follow-up questions:** Symptoms of a missing CA? (`SSLHandshakeException`, swallowed to null by `safeGet`.)

### Q90. How do Sonar rules show up in this code?
- **Why the interviewer asks:** Static-analysis literacy.
- **Ideal answer:** Examples in the repo: S4502 (CSRF disabled) is justified in `SecurityConfig` javadoc; "cognitive complexity" and long methods apply to `execute` (about 1,300 lines in ASP, 400+ in Template12); `catch (Exception)` narrowing (the test-copy of `AspServiceTaxInvoiceHandler`); unused locals; duplicated code (about 700 lines landscape mapping); commits `660044b26`, `c336f76fa` "fixed sonar issues". The quality gate waits in CI. Real fix for complexity is extraction by sub-report or a registry, not suppression.
- **Common mistakes:** Fixing the warning with `@SuppressWarnings`.
- **Follow-up questions:** Which rule would you ignore? (Only with a written reason, like S4502 here.)

### Q91. Why do some Mongo expressions need `$ifNull` before arithmetic?
- **Why the interviewer asks:** Null semantics in pipelines.
- **Ideal answer:** `$add`, `$subtract`, `$multiply` return null if any operand is null or missing. Oracle `NVL(x,0)` becomes `{$ifNull: ["$x", 0]}`. Examples: GPA refund = round(match - gross - stax - stamp) in the ASP schedule; fee differences added to gross premium for products 089332, 089432, 089329, 089429; `NVL(STI_PREM,0)` in the delete-member handler. Missing `$ifNull` yields an empty decimal element that fails the XSD.
- **Common mistakes:** Using `$ifNull` on empty strings.
- **Follow-up questions:** `$sum` of an empty group? (0 for `$sum`; null for `$avg`.)

### Q92. What is the difference between `$project` and `$addFields` in these pipelines?
- **Why the interviewer asks:** Basic stage use, tied to DTO mapping.
- **Ideal answer:** `$project` selects and renames the output fields to the DTO names; fields not listed disappear (so a leftover `ROW_NUMBER` from `$addFields` never reached the DTO). `$addFields` adds computed fields and keeps the rest (`CUST = $arrayElemAt(CUST_ROW, 0)`). In `RejectRiskHandler`, `$addFields` builds `CUST`, `RRT`, `RSN`, `RVK`, then one `$project` produces `customerName`, `idType`, `rejectRiskType`, etc.
- **Common mistakes:** Projecting before the lookup result is added.
- **Follow-up questions:** `$set` vs `$addFields`? (Aliases.)

### Q93. How do you read `Date.toString()`-style values safely?
- **Why the interviewer asks:** Real parsing pitfall in `DateUtils`.
- **Ideal answer:** Pattern `EEE MMM dd HH:mm:ss z yyyy` (for example `Mon Jan 05 08:30:00 UTC 2026`) needs a `Locale` (use `Locale.ENGLISH`), and the `z` zone abbreviations (IST, UTC, MYT) are not all parsed by `DateTimeFormatter`; other handlers use `SimpleDateFormat` for that reason (comment in `MarinePolScheduleHandler`). The service wraps parsing in try/catch and returns null. Better to avoid this format entirely with `$dateToString` in the pipeline.
- **Common mistakes:** Running tests on a machine with a different default locale/zone and getting different results.
- **Follow-up questions:** How to make the test stable? (Fix zone and locale in the test.)

### Q94. What logging style is used, and what would you change?
- **Why the interviewer asks:** Observability basics.
- **Ideal answer:** `@Slf4j` with `log.warn`/`log.info` in services and handlers (for example `Template12LandscapeDispatchHandler contractId={}`), plus `System.out.println` in `CustomRequestFilter` and `CommonController`. No MDC/correlation id, no JSON format, no `logback.xml`. Use parameterised logging, pass the exception object to `log.warn("...", e)` so the stack is kept, never log request bodies with personal data, and add a request id per call. `GlobalExceptionHandler` should log the exception.
- **Common mistakes:** String concatenation in log calls; swallowing the stack trace.
- **Follow-up questions:** Where would you put the request id? (A filter adding it to MDC and to outbound headers.)

### Q95. Why is `POST /GenerateReport` idempotent in practice?
- **Why the interviewer asks:** HTTP semantics for a read-only action.
- **Ideal answer:** The service only reads (aggregations and GET calls), so repeating the same request returns the same data apart from `timestamp` and data changes. It is POST because filters travel in a JSON body. No idempotency key is needed. Caches and proxies do not cache POST, so each print recalculates everything. Consider a short result cache only after confirming the data freshness rules.
- **Common mistakes:** Assuming POST means a state change.
- **Follow-up questions:** HTTP status for no data? (Today 200; 404 exists in code but is never thrown.)

### Q96. How does the controller treat an unknown `reportName` in code terms?
- **Why the interviewer asks:** Code-level reading of the error contract.
- **Ideal answer:** After normalising the name, the controller calls `reportStrategies.get(name)`; when it returns null it goes to the legacy `switch(reportName)` of about 50 cases. No visible default branch, so `transactions` stays empty; the controller builds `createDefaultTransaction` with `mailingDetails` and `templateDetails` and returns `SuccessResponse` SUCCESS. `switch(null)` throws NPE. Improvement: check `strategies.containsKey` and legacy case list, and throw `IllegalArgumentException` (400) for unknown names. [Caveat: the full switch was not re-read; confirm in code.]
- **Common mistakes:** Not distinguishing a null name from an unknown name.
- **Follow-up questions:** Do the merged-away names behave this way? (Yes, they now have no bean.)

### Q97. What is the thread/time risk of 600 s `maxTime` with Tomcat threads?
- **Why the interviewer asks:** Capacity thinking.
- **Ideal answer:** Each request holds a Tomcat worker (default 200 threads) while handlers run. A slow aggregation with `maxTime` 600 s can occupy a thread for 10 minutes, even if the caller timed out at ~20 s. Several such calls (an unauthenticated endpoint) can exhaust the pool. Mitigate: lower `maxTime` close to the caller timeout, add inbound timeouts, rate limit at the gateway, and set the Mongo `socketTimeoutMS`/`timeoutMS` in the URI. No inbound timeout is configured.
- **Common mistakes:** Treating the HTTP read timeout as a server-side limit.
- **Follow-up questions:** How to see thread use? (Expose metrics via actuator/Micrometer; currently only health/info.)

### Q98. How would you unit-test the gate logic in `AspEndtDocumentService` without 1,300 lines of mocks?
- **Why the interviewer asks:** Testing a god method.
- **Ideal answer:** Today `AspEndtDocumentServiceTest` (28 tests) mocks all handlers, with one big `setUp` and tests per gate (`showServiceTax_checklistCopyY_closesGate`, `endtCode72_fetchesScheduleOfBenefitsAndLlbAndSti`, `showDebitNote_withZeroPrem_doesNotFetchDebitNotes`). Cleaner: extract the gates into small pure methods or a `AspGates` record computed from the two main DTOs and the filter, then test the matrix with `@ParameterizedTest` (`showTaxInvoice`, `SHOW_STAX_INV_IND`, `checklistCopy`, `SHOW_EINV_IND`). The service then only wires handlers.
- **Common mistakes:** Refactoring without first locking behaviour with the existing tests.
- **Follow-up questions:** How many gate combinations for the service-tax invoice? (Four boolean inputs = 16; only one combination opens it.)

### Q99. What does the `rewrite-maven-plugin` do here?
- **Why the interviewer asks:** Build-tool awareness (small but real).
- **Ideal answer:** OpenRewrite 6.46.1 is declared as scaffolding for automated migrations (for example Boot upgrades) but no active recipes are configured; version 6.48.0 could not be resolved from the Nexus mirror. It does not change runtime behaviour. The Boot 4 code changes in this project were done by hand (listed in the upgrade commits). Candidate recipes later: Jackson 2 to 3, JUnit 5 to 6, Spring Boot 4 property migration.
- **Common mistakes:** Saying the upgrade was automated.
- **Follow-up questions:** Risk of running recipes blindly? (They rewrite many files; review the diff and run tests.)

### Q100. If you had two days, which three technical improvements would you ship and why?
- **Why the interviewer asks:** Prioritisation with technical depth.
- **Ideal answer:** (1) Real XSD validation test: serialise `RejectRiskService` output with the production `XmlMapper` and validate with an error-collecting validator, ratchet baseline; it makes the biggest risk visible. (2) Fix RejectRisk tests and ordering: assert pipeline stages by content, add the null-name flag to `$setWindowFields.sortBy` and `$sort`, decide the no-dates rule. (3) Remove the test-tree copy of `AspServiceTaxInvoiceHandler`, move the Mongo URI to env vars, and log in the catch-all exception handler. Reason: cheap, high risk reduction, no behaviour change for other teams.
- **Common mistakes:** Picking a rewrite or a new feature.
- **Follow-up questions:** What would you leave out and why? (Big refactors without golden tests.)
