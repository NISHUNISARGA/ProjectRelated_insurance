# 08c - Technical Questions 1-50 (Java / Spring / MongoDB aggregation as used in THIS code)

Conventions: facts come from the KT guide and KT_Parts 01-06 and the repo; "(inferred)" = reasoning, not run; "[CONFIRM: ...]" = personal fact. No textbook definitions: each answer is tied to a class, collection or field in this service.

---

### Q1. How does Spring give `CommonController` all report services?
- **Why the interviewer asks:** The whole routing depends on this mechanism.
- **Ideal answer:** `ReportStrategy` is an interface (`Transactions execute(PayLoadRequest request)`). Each report is a `@Service("<name>")` bean. When a class injects `Map<String, ReportStrategy>`, Spring fills the map with all beans of that type, keyed by bean name. `CommonController` does `reportStrategies.get(reportName)` (lines ~171, 203). A new report only needs a service with the right bean name; no controller change.
- **Common mistakes:** Thinking the key is the class name, or that you need `@Qualifier`.
- **Follow-up questions:** What if two beans have the same name? (Startup failure or override problem; this is why merged-away classes must keep `@Service` commented.)

### Q2. Why can a bean name contain slashes, and what breaks if you re-enable a disabled class?
- **Why the interviewer asks:** Checks bean-registration understanding.
- **Ideal answer:** `@Service` takes any string as bean name, so `AGIC/UW/Plines/ASPEndtSchedule/EndtDocument` is legal and matches the Jasper repository path. `AspEndtDocument2Service` has `//@Service` and the landscape class has two `//@Bean` lines commented. If re-enabled, the bean keys for `.../EndtDocument2`, `.../AddMemberLandscape`, `.../DeletionMemberLandscape` appear again, and the same work would also run inside the merged `EndtDocument` bean, so bands render twice (class javadoc warns). For Template12, re-enabling `Template12PolicyDocument2Service` does the same.
- **Common mistakes:** Believing slashes make it a URL mapping.
- **Follow-up questions:** How would you make the merge visible in tests? (A test that asserts the map keys.)

### Q3. How are dependencies injected in handlers and clients here?
- **Why the interviewer asks:** Spring basics applied to real code.
- **Ideal answer:** Handlers are `@Component` with `MongoTemplate` injected (Lombok `@RequiredArgsConstructor` style, constructor injection). `PlineUtilsClient` takes `@Qualifier("fastLookupRestTemplate") RestTemplate` and `@Value("${API_BASE-URL}")`. `PlinePolicyRefClient`, `PlineSSTRolloutDateClient`, `PlinePgRpgeUtilsClient` take `@Value("${plines.api.base-url}")` in the constructor. Constructor injection makes tests simple with `@InjectMocks`.
- **Common mistakes:** Field injection with `@Autowired` in tests, or forgetting `@Qualifier` and getting the `@Primary` motor `RestTemplate`.
- **Follow-up questions:** Which bean does an unqualified `RestTemplate` get? (The `@Primary` motor `restTemplate`: 5 s connect, 20 s read.)

### Q4. How does a handler run an aggregation with `MongoTemplate`?
- **Why the interviewer asks:** Core data access in this service.
- **Ideal answer:** Pattern: `mongoTemplate.aggregate(Aggregation.newAggregation(ops), "COLLECTION", Dto.class).getMappedResults()`. `ops` is a list of `AggregationOperation` lambdas like `ctx -> new Document("$match", ...)`. The `$project` stage names fields so they map by name to the DTO. In Template12's main handler, `.withOptions(AGG_OPTIONS)` adds `allowDiskUse` and `maxTime`.
- **Common mistakes:** Forgetting that unlisted fields vanish after `$project`.
- **Follow-up questions:** What if the DTO field type differs from the BSON type? (Conversion error or null; for example date `String` fields.)

### Q5. Why use raw `Document` stages instead of the typed `Aggregation.match(...)` API?
- **Why the interviewer asks:** Design trade-off.
- **Ideal answer:** The pipelines come from Oracle SQL translations and use `$setWindowFields`, `$lookup` with `let/pipeline`, `$unionWith`, `$toDouble` and `$ifNull` expressions; raw `Document` is exact and copy-paste from mongosh. Cost: no compile-time checks, and tests must inspect `Document` stages (`aggregation.toPipeline(Aggregation.DEFAULT_CONTEXT)`). Typed Criteria is used in simple point queries (`Criteria.where("CONTRACT_ID").is(asNum(contractId))`).
- **Common mistakes:** Mixing both styles in one handler without reason.
- **Follow-up questions:** How do you debug a pipeline quickly? (Log with `Aggregation` DEBUG, paste into Compass.)

### Q6. Explain the `$lookup` with `let` and `pipeline` used for `CMGE_CODE`.
- **Why the interviewer asks:** Most-used pattern in RejectRisk.
- **Ideal answer:** `from: CMGE_CODE, let: {rtc: "$REJ_TYPE"}, pipeline: [{$match: {$expr: {$and: [{$eq: ["$CAT_CODE","PLREJ_RISK_TYPE"]}, {$eq: ["$CODE_CD","$$rtc"]}]}}}, {$limit: 1}], as: "RRT_ROW"`. Then `$addFields {RRT: {$arrayElemAt: ["$RRT_ROW", 0]}}`. It replaces the Oracle scalar sub-select `(SELECT CODE_DESC FROM CMGE_CODE WHERE CAT_CODE=... AND CODE_CD=...)`. Same for `PLREJ_REASON` and `PLREJ_REVOKE_DUR`. It is one sub-pipeline per input row.
- **Common mistakes:** Using `localField/foreignField` when two conditions are needed, or forgetting `$$` for the `let` variable.
- **Follow-up questions:** Cost? (Per-row sub-pipeline; needs an index on `{CAT_CODE, CODE_CD}`.)

### Q7. `$unwind` vs `$arrayElemAt` after a lookup: which keeps LEFT JOIN behaviour?
- **Why the interviewer asks:** Join semantics.
- **Ideal answer:** A plain `$unwind` removes documents whose array is empty (inner join). RejectRisk uses `$addFields {X: {$arrayElemAt: [rows, 0]}}`, which gives null when there is no match, keeping the row. `AspDebitNoteHandler` uses `$unwind` with `preserveNullAndEmptyArrays: true` for the same result. This reproduces `CpgeViPartnersAll.PART_ID(+)`.
- **Common mistakes:** Using plain `$unwind` and losing rows with no partner.
- **Follow-up questions:** What if the lookup returns several rows? (`$arrayElemAt 0` picks the first; `$limit 1` makes it explicit, but without a sort the "first" is arbitrary.)

### Q8. Why is `$limit: 1` inside the `$lookup` pipeline?
- **Why the interviewer asks:** Performance and semantics.
- **Ideal answer:** It stops the sub-pipeline after the first match and models a scalar sub-select. Downside: it hides duplicates; Oracle would raise ORA-01427 on two rows, Mongo silently takes one. Only `AspServiceTaxInvoiceHandler` and the Template12 PolicyDocument primary emulate the error.
- **Common mistakes:** Thinking `$limit` sorts.
- **Follow-up questions:** How would you detect duplicate codes? (A `$group` by key with count > 1 in a data-quality check.)

### Q9. How is Oracle `ROWNUM` reproduced in `RejectRiskHandler`?
- **Why the interviewer asks:** A classic translation.
- **Ideal answer:** Oracle: inner query `ORDER BY NAME_EXT`, outer `ROWNUM`. Mongo: `$setWindowFields: {sortBy: {customerName: 1}, output: {rowNumber: {$documentNumber: {}}}}`, then a final `$sort {customerName: 1}` to fix the output order. `$setWindowFields` needs MongoDB 5.0+ (inferred; other handlers like `CardPrintingHandler` use it). It is a blocking stage; with no `allowDiskUse` it can hit the 100 MB per-stage limit on a huge result (inferred). The earlier fake `ROW_NUMBER = 1` `$addFields` was dead code and was removed in the uncommitted change.
- **Common mistakes:** Using `$addFields: {n: 1}` as a row number.
- **Follow-up questions:** What changes if two names are equal? (Tie order is undefined; add a tie-breaker.)

### Q10. How does BSON ordering affect `$sort` here?
- **Why the interviewer asks:** Real parity bug.
- **Ideal answer:** BSON type order puts null and missing before numbers and strings, so ascending sort puts them first. Oracle ascending puts NULL last. In RejectRisk, rows with no partner (null `customerName`) are first and take row numbers 1..k. Fix in pipeline: add `_nullName` flag (`$cond` on `$ifNull`) and sort by `{_nullName: 1, customerName: 1, ...}`, using the same keys in `$setWindowFields.sortBy`; or sort in Java with `Comparator.nullsLast`. The same applies to `COV_SEQ_REF` ordering in landscape handlers.
- **Common mistakes:** Fixing only the final `$sort` and not the window `sortBy`.
- **Follow-up questions:** Collation? (Mongo default is binary; Oracle NLS_SORT default is binary too - inferred, unverified for the real DB.)

### Q11. How do you emulate `SELECT DISTINCT` and `UNION` in these handlers?
- **Why the interviewer asks:** Several handlers need it.
- **Ideal answer:** DISTINCT -> `$group` on the selected fields (`UWPL_VI_POLICY_SCHEDULE_COVER`). `UNION ALL` of WIP and posted: `$unionWith` (ASP schedule: `UWGE_POLICY_VERSIONS` + `$unionWith WIP_UWGE_POLICY_VERSIONS`), or two `$lookup`s with `$ifNull`, or two Java queries merged and de-duplicated with `LinkedHashSet`. Note SQL `UNION` removes duplicates while `UNION ALL` does not.
- **Common mistakes:** Treating UNION and UNION ALL the same.
- **Follow-up questions:** Why did ASP query the version collections directly? (The summary view applies a `VERSION_NO` ceiling.)

### Q12. How are WIP-vs-posted defaults written with `$ifNull`?
- **Why the interviewer asks:** Used in PD2 and ASP main handlers.
- **Ideal answer:** Two `$lookup`s (for example into `WIP_UWGE_POLICY_CONTRACTS` and `SB_UWGE_POLICY_CONTRACTS`), each with `$project` + `$limit 1`, then `{$ifNull: [firstElem("$PC_WIP.PRODUCT_CONFIG_CODE"), firstElem("$PC_SB.PRODUCT_CONFIG_CODE")]}`. For `ENDT_CODE`, `$$chk` (checklist copy) decides which branch returns rows: the WIP lookup runs only when `$$chk == "Y"`, the live one when "N". `$ifNull` only checks null or missing, not empty string.
- **Common mistakes:** Expecting `$ifNull` to skip empty strings.
- **Follow-up questions:** What if both branches return data? (First non-null wins: no conflict detection except in the emulated-ORA-01427 handlers.)

### Q13. `$toDouble` vs `$convert` for sorting `COV_SEQ_REF` and `SORT_SEQ`.
- **Why the interviewer asks:** Data type robustness.
- **Ideal answer:** Jasper sorts by `cast(COV_SEQ_REF as decimal(15,2))`. Handlers add `$toDouble` first, then `$sort`. `$toDouble` throws on a non-numeric string, like Oracle's cast error. `$convert` with `onError`/`onNull` is safer when empty strings exist. Three handlers still sort the raw field (`...InsuredLandscapeAmc`, `...DiagnosisLandscape`, `...DiagnosisLandscapeAmc`). Whether the field is a string in Mongo is unverified.
- **Common mistakes:** Sorting after `$project` renamed the field.
- **Follow-up questions:** Test? (Two rows "2" and "10"; assert order 2, 10.)

### Q14. Why does `$ne: [missing, 0]` matter in `AspEndtDocument2MainHandler`?
- **Why the interviewer asks:** Mongo expression semantics.
- **Ideal answer:** PREM is computed as `$cond [{$ne: ["$PB_ROW.DIFF_PREM_DUE", 0]}, 1, 0]`. If `PB_ROW` is null, the path is missing, and in the aggregation expression language a missing value compares as not equal to 0, so `$ne` is true and PREM becomes 1 (inferred from documented Mongo behaviour; no test). Fix: guard with `$ifNull(..., 0)` or check `$gt` / existence. Compare Oracle: no driver row, no output at all.
- **Common mistakes:** Assuming missing equals 0 or null equals 0.
- **Follow-up questions:** What does `$add` do with null? (Returns null; wrap each operand in `$ifNull`.)

### Q15. `$round` in Mongo vs `ROUND` in Oracle.
- **Why the interviewer asks:** Money rounding.
- **Ideal answer:** `AspDebitNoteHandler` computes `ROUNDED_NETT_PREM` as `ABS(round(NETT_PREM*20)/20)` (nearest 5 sen). Oracle `ROUND` rounds half away from zero; Mongo `$round` rounds half to even (documented behaviour; inferred risk at .5 boundaries). In Java you would use `BigDecimal.setScale(..., RoundingMode.HALF_UP)`. Add a test on exact .5 amounts.
- **Common mistakes:** Using double for money.
- **Follow-up questions:** How would you emulate half-up in Mongo? (`$ceil`/`$floor` on sign-adjusted values.)

### Q16. Why is `$lt dateTo+1day` better than `$lte dateTo` or `$dateTrunc`?
- **Why the interviewer asks:** Index-aware query design.
- **Ideal answer:** `TRUNC(x) <= to` equals `x < to + 1 day`. The half-open range `[from 00:00, to+1d 00:00)` uses the bare field `CREATED_DATE`, so an index on it can be used. Wrapping the field in `$dateTrunc` inside `$expr` would block index use. `$lte` with a midnight value loses the whole last day. `RejectRiskHandler` builds `dateFilter` with `$gte` and `$lt` only for non-null bounds.
- **Common mistakes:** Using `$expr` with date functions in `$match` and expecting index use.
- **Follow-up questions:** Risk if `dateTo` already has a time? (+1 day is then not the next midnight.)

### Q17. How are dates converted in `RejectRiskHandler.toDate`?
- **Why the interviewer asks:** Java time API in context.
- **Ideal answer:** Input may be a `Date`, a `LocalDate` or a string. Strings go to `plines.utils.DateUtils.toLocalDate` (formats: `yyyy-MM-dd` fast path, `dd-MM-yyyy`, `dd/MM/yyyy`, `MM/dd/yyyy`, `dd-MMM-yyyy`, `dd-MMM-yy`, `EEE MMM dd HH:mm:ss z yyyy`, ISO offset/zoned). Then `Date.from(localDate.atStartOfDay(ZoneId.of("UTC")).toInstant())`. `dd/MM/yyyy` is tried before `MM/dd/yyyy`, so `03/04/2026` is 3 April. A local date-time without offset throws. Invalid values throw and the request fails (500).
- **Common mistakes:** Using the JVM default zone without noticing.
- **Follow-up questions:** Why UTC? (Matches UTC-stored data; check a real record, KL time may be needed.)

### Q18. How would you return dates from Mongo as stable strings?
- **Why the interviewer asks:** DTO date fields are `String`.
- **Ideal answer:** Today BSON `Date` is converted into a DTO `String` field, probably via `Date.toString()` (inferred), which depends on the JVM zone and needs the `EEE MMM dd HH:mm:ss z yyyy` pattern to parse. Better: in `$project` use `{$dateToString: {format: "%Y-%m-%d", date: "$CREATED_DATE", timezone: "Asia/Kuala_Lumpur"}}` (pattern already used in other handlers), so `DateUtils` hits the fast path. Keep the zone choice explicit and tested.
- **Common mistakes:** Parsing `Date.toString()` text.
- **Follow-up questions:** Why can the `z` pattern fail? (Zone abbreviations are not all parsed; the service swallows it and returns null.)

### Q19. When does `$lookup` with `$expr` use an index?
- **Why the interviewer asks:** Performance understanding.
- **Ideal answer:** In recent MongoDB versions an equality `$eq` between a foreign field and a `let` variable inside `$expr` can use an index on the foreign collection; older versions cannot. Without an index it scans the foreign collection per input row (O(n*m)). Recommended indexes: `CMGE_CODE {CAT_CODE, CODE_CD}`, `CPGE_VI_PARTNERS_ALL {PART_ID, VERSION}`. Verify with `explain("executionStats")` (look at `totalDocsExamined` and COLLSCAN). `*_VI_*` names may be Mongo views, which cannot have their own indexes - unknown here.
- **Common mistakes:** Assuming indexes exist because the Oracle tables had them.
- **Follow-up questions:** `auto-index-creation=false`: does it matter? (No: wrong prefix, and no `@Document` entities.)

### Q20. Explain type strictness and `safeNum`.
- **Why the interviewer asks:** Root of the CONTRACT_ID bug.
- **Ideal answer:** Mongo equality is type-aware across string vs number: `{CONTRACT_ID: "123"}` does not match numeric 123. Int32 vs Int64 vs double compare equal numerically, so `Long` is fine against a stored Int32. Handlers convert request strings: `safeNum` / `asNum` = `Long.parseLong` with fallback to the raw string when not numeric (`AspEndtDocumentMainHandler:154`, `AspDebitNoteHandler:393`, `AspServiceTaxInvoiceHandler.safeNum` tries Integer then Long). If the data is stored as a string, a numeric match would silently fail - check data.
- **Common mistakes:** Calling `toString` on the numeric side.
- **Follow-up questions:** What happens with empty `pContractId`? (Template12 landscape dispatch does `Long.parseLong("")` and throws `NumberFormatException`.)

### Q21. Why does `Integer.equals(Long)` matter in `requireNoConflict`?
- **Why the interviewer asks:** A Java equality trap found in the code review notes.
- **Ideal answer:** `AspServiceTaxInvoiceHandler.requireNoConflict` compares live vs WIP `VERSION_NO` with `equals`. If one comes as `Integer` and the other as `Long`, `equals` returns false even for equal numbers, giving a false conflict and an `IllegalStateException` (inferred). Fix: normalise both to `Long` (or compare `Number.longValue()`).
- **Common mistakes:** Using `==` on boxed numbers.
- **Follow-up questions:** Which test would catch it? (Live Integer 1, WIP Long 1, expect no exception.)

### Q22. Why are Template12 and ASP built from many small queries instead of one big pipeline?
- **Why the interviewer asks:** Architecture choice with trade-offs.
- **Ideal answer:** Oracle SQL used nested scalar sub-selects and `dual`. The Java code runs "Step A..Step G": `match -> project -> limit 1` per source, each with `AGG_OPTIONS`. Good: each step is simple, indexable and unit-testable. Bad: many round trips per request (the ASP MST chain is 4-5 queries), no atomic snapshot between steps, more load on the pool. Alternative: one pipeline with several `$lookup`s, harder to debug.
- **Common mistakes:** Saying "one pipeline is always faster".
- **Follow-up questions:** How would you reduce round trips safely? (Cache `SAPM_SYS_CONSTANTS`/`CMGE_CODE`; parallel independent steps.)

### Q23. What do `allowDiskUse` and `maxTime` mean in `AGG_OPTIONS`?
- **Why the interviewer asks:** Operational settings.
- **Ideal answer:** `allowDiskUse(true)` lets blocking stages (`$sort`, `$group`) spill past the 100 MB per-stage memory limit. `maxTime` is 600 s on the server. It is set in `PlinePolicyDocumentMainHandler`, `AspDebitNoteHandler`, `AspEndtScheduleHandler`, etc., but not in `RejectRiskHandler` or `AspEndtDocumentMainHandler`. 10 minutes is far above the 20 s HTTP read timeout of callers, so orphan queries can run after the caller gave up.
- **Common mistakes:** Believing `maxTime` aborts on the client side.
- **Follow-up questions:** What is a sensible value? (Seconds, aligned with caller timeouts.)

### Q24. How do you avoid operator injection in Mongo filters here?
- **Why the interviewer asks:** Security reasoning on real code.
- **Ideal answer:** `filterCondition` is `Map<String,String>`, so a JSON object like `{"$ne": ""}` cannot bind (Jackson fails). Filters are built as `Document` with String values (`CUST.NAME_EXT` equals `pCustomerName`), no `$where`, no JS, no string concatenation into queries. Outbound REST values are separately sanitised (`PlineUtilsClient.sanitize`).
- **Common mistakes:** Saying Mongo is immune to injection.
- **Follow-up questions:** What if the filter map became `Map<String,Object>`? (You must validate types.)

### Q25. Why `Pattern.quote` plus option `i` in `PlineReportHeaderHandler`?
- **Why the interviewer asks:** Regex use in a Mongo query.
- **Ideal answer:** Jasper compares `UPPER(TEMPLATE_NAME) = UPPER(reportName || '/' || pReportTitle)`. Java builds a case-insensitive regex `^\Q...\E$` with option `i` so characters like `.` or `/` in the title are literal. A regex with `i` cannot use an index efficiently (inferred), but `RPGE_RPT_TEMPLATE` is small. It returns `DATA_CLASS`, or `new PlineReportHeaderDto(null)` when none is found.
- **Common mistakes:** Concatenating raw user text into a regex.
- **Follow-up questions:** Alternative? (Store an upper-case key field and match equality.)

### Q26. `getMappedResults()` loads everything. When is that a problem?
- **Why the interviewer asks:** Memory awareness.
- **Ideal answer:** RejectRisk returns all rows in memory, then creates about eight payload objects per row and serialises them to XML and a JSON string. With the working-tree "no dates = all rows" it can be very large, and `$setWindowFields`/`$sort` can also exceed 100 MB without `allowDiskUse`. Mitigations: require dates, cap rows, add paging or a streaming approach, set `allowDiskUse`. Template12 BMW/PA4A loops are per row without a cap.
- **Common mistakes:** Ignoring the payload object explosion.
- **Follow-up questions:** Streaming option? (`aggregateStream`, but the XML is a single string, so you still buffer.)

### Q27. How do you test the shape of an aggregation without a database?
- **Why the interviewer asks:** Your test method and its limit.
- **Ideal answer:** Mock `MongoTemplate`. Stub `aggregate(any(Aggregation.class), anyString(), any(Class.class))` with `doAnswer` that captures the `Aggregation`, then `captured.toPipeline(Aggregation.DEFAULT_CONTEXT)` and assert on the `Document` stages. This is used in 116 places over 23 files. A `forcePipelines` helper only runs lambdas for coverage. Limit: proves stage shape, not data semantics (null order, `$lookup`, dates).
- **Common mistakes:** Asserting `pipeline.get(1)` by index.
- **Follow-up questions:** Better assertion? (Find the stage containing `CREATED_DATE`.)

### Q28. `Strictness.LENIENT` and `@InjectMocks`: what do they change?
- **Why the interviewer asks:** Mockito knowledge as used in 36 files.
- **Ideal answer:** `@MockitoSettings(strictness = LENIENT)` stops Mockito failing on unused or mismatched stubs, which suits large shared `setUp` in service tests. Cost: dead stubs hide. `@InjectMocks` builds the class under test and injects `@Mock` fields (constructor first). `mock-maker-inline` (in `src/test/resources/mockito-extensions`) allows mocking final classes.
- **Common mistakes:** Thinking `@InjectMocks` creates mocks.
- **Follow-up questions:** How would you find dead stubs? (Run once with STRICT_STUBS.)

### Q29. Which Boot 4 test API changes hit this repo?
- **Why the interviewer asks:** Migration detail.
- **Ideal answer:** `@MockBean` -> `@MockitoBean`; explicit `MediaType` imports (collision with JUnit 6); `getRawStatusCode()` -> `getStatusCode().value()`; `UriComponentsBuilder.fromHttpUrl` -> `fromUriString`; explicit `WebClient.Builder` bean (no longer auto-configured). The pom still declares `junit-jupiter 5.9.0` while the BOM is 6.0.3 (inferred effective engine); a mismatch risk to verify with `dependency:tree`.
- **Common mistakes:** Believing the explicit 5.9.0 wins over the BOM.
- **Follow-up questions:** How do you see the effective version? (`mvn dependency:tree -Dincludes=org.junit.jupiter`.)

### Q30. How do you write a test that proves the ORA-01427 emulation?
- **Why the interviewer asks:** Exception testing.
- **Ideal answer:** Stub the Mongo result so two different values come back (live and WIP), call the handler, and assert `IllegalStateException` with message containing "ORA-01427". Existing: `AspServiceTaxInvoiceHandlerTest.fetchServiceTaxInvoice_ambiguousEinvCutoff_throwsIllegalStateException`. Gap: no direct test of `PlinePolicyDocumentMainHandler.fetchTemplate12PolicyDocumentMain` (`requireNoConflict`, `distinctValueOrConflict`). Use `assertThatThrownBy(...).isInstanceOf(IllegalStateException.class)` or `assertThrows`.
- **Common mistakes:** Testing only the happy path of the handler.
- **Follow-up questions:** Why `IllegalStateException`? (The state of the data is invalid; not a bad argument.)

### Q31. Which Lombok features does the code rely on, and what do the DTO tests prove?
- **Why the interviewer asks:** Practical Lombok knowledge.
- **Ideal answer:** `@Slf4j`, `@RequiredArgsConstructor`, `@Data`-style getters/setters/equals/hashCode/toString on DTOs and `*Details` payloads, `@NoArgsConstructor/@AllArgsConstructor`. The generated tests (six per class) prove Lombok works: no-arg, all-args, setters/getters, equals/hashCode, toString. They add coverage but little protection (337 of 865 tests in scope). Lombok 1.18.46 via BOM; the annotation processor is configured in `maven-compiler-plugin`.
- **Common mistakes:** Counting them as business tests.
- **Follow-up questions:** Pitfall: Lombok getter vs Jackson name? (`FilterCondition` field with `getFilterCondition()` serialises as `filterCondition`.)

### Q32. Why do DTO date fields use `String`, and what is the cost?
- **Why the interviewer asks:** Data modelling choice.
- **Ideal answer:** Mixed formats arrive from Mongo (BSON dates, `T00:00:00` strings, formatted strings). Strings avoid conversion errors in `aggregate(..., Dto.class)`. Cost: parsing later in the service (`toLocalDateTime`, `DateUtils`), silent null on failure (`toLocalDateTime` swallows all exceptions), and dependence on `Date.toString()` (inferred). Better: project ISO strings in the pipeline.
- **Common mistakes:** Keeping `Date` in the DTO and then formatting in different ways per handler.
- **Follow-up questions:** Where would a parse failure show up? (A missing `createdDate` leaf in the XML.)

### Q33. `SimpleDateFormat` or `DateTimeFormatter`? Which is used and why does it matter?
- **Why the interviewer asks:** Java time API judgement.
- **Ideal answer:** `DateUtils` uses `DateTimeFormatter` lists. Other handlers use `SimpleDateFormat` for `Date.toString()` text (comments mention timezone abbreviations like IST/UTC). `SimpleDateFormat` is not thread-safe; in singletons it must be created per call. `DateTimeFormatter` is immutable and safe. `SimpleDateFormat("dd-MMM-yyyy").format(date)` is also used in Jasper expressions.
- **Common mistakes:** Sharing a static `SimpleDateFormat` in a `@Component`.
- **Follow-up questions:** Why does `dd/MM/yyyy` precede `MM/dd/yyyy`? (Ambiguous dates are read as day first.)

### Q34. `catch (Exception)` or `catch (RuntimeException)` in `safeCall`?
- **Why the interviewer asks:** Exception handling style and Sonar.
- **Ideal answer:** The production `AspServiceTaxInvoiceHandler.safeCall` and the stray test-tree copy differ here (`catch (Exception e)` vs `catch (RuntimeException e)` with `SafeCallable.call() throws Exception`), consistent with a Sonar-driven edit. Catching broad `Exception` hides checked exceptions and interrupts; narrow catches are cleaner but need the callable not to throw checked ones. The same class returns null on client failure, which the service treats as default "N".
- **Common mistakes:** Swallowing `InterruptedException` without restoring the flag.
- **Follow-up questions:** Which failures should not be swallowed? (SecurityException from `assertTrustedHost` propagates outside the try in `safeGet`.)

### Q35. What is the Java 17 feature used for numbers, and how does `toBigDecimal` work?
- **Why the interviewer asks:** Language features and money types.
- **Ideal answer:** Pattern matching for `instanceof` (`instanceof Number n`) is used when converting mixed Mongo values (Integer, Long, Double, String). `toBigDecimal` maps a value to `BigDecimal` for XSD decimals (null for blank or invalid). Use `new BigDecimal(String)` or `BigDecimal.valueOf(double)`, not `new BigDecimal(double)`. XSD `xs:decimal` has no fixed scale, so scale is a formatting choice for the renderer.
- **Common mistakes:** Using `double` for premium.
- **Follow-up questions:** Why `rownumber` is `BigDecimal`? (Jasper `ROWNUMBER` is `BigDecimal` and the XSD leaf is decimal.)

### Q36. How does `mergeNonNullFields` work and what are its risks?
- **Why the interviewer asks:** Reflection in a hot path.
- **Ideal answer:** It walks the fields of `PlineTransDet` reflectively and copies non-null values from the new row to each existing `transDet` (`B12:212-258`), with special cases for lists (risk, benefit print, nominee print, stax) and a guard that a blank `eInvStatus` never overwrites a real one. Risks: last non-null writer wins; reflection hides field renames from the compiler; slower than direct setters; `setAccessible` issues under strict module rules (not a problem on the classpath). The 650-leaf class is why reflection was chosen.
- **Common mistakes:** Forgetting default-populated placeholder lists.
- **Follow-up questions:** Replace with? (Generated or explicit mappers per band, or MapStruct - not used here.)

### Q37. The reflective date pass at the end of S12: how does it work?
- **Why the interviewer asks:** Another reflection use.
- **Ideal answer:** It walks String fields whose name contains "date" or "dob" and rewrites them to `dd/MM/yyyy` (`S12:2105-2230`, applied `S12:549-554`), plus `reportParameters` dates. It exists because the base class cannot be touched and `toLocalDateTime` removes the time. Weakness: name-based matching can hit unintended fields; no test for the pass (inferred from test names).
- **Common mistakes:** Treating it as a general formatter.
- **Follow-up questions:** Cleaner design? (Typed date fields with `@JsonFormat`, or one date policy in the mapper.)

### Q38. Where do collections like `LinkedHashSet` and `Collections.singletonList(1)` appear?
- **Why the interviewer asks:** Reading real code accurately.
- **Ideal answer:** `LinkedHashSet` removes duplicates while keeping order after merging WIP and live results (Oracle UNION). `Collections.singletonList(1)` is the stub result of "dual" handlers such as `PlineT12DomesticAtcHandler` and `PlineT12OverseasAtcHandler` (no DB call). `Document` and `List.of(...)` build stages; note `List.of` rejects nulls.
- **Common mistakes:** `List.of` with a null element.
- **Follow-up questions:** Why keep order? (Output order is part of the report.)

### Q39. How is an `Aggregation` built with variable stages (optional filters)?
- **Why the interviewer asks:** Dynamic query construction.
- **Ideal answer:** `RejectRiskHandler` builds `List<AggregationOperation> ops` and appends stages only when a filter is non-null and non-empty (`!= null && !isEmpty()`, same as Oracle `IS NULL` semantics for empty strings). Order matters: date/type/reason `$match` before the lookups to cut work; customer name/ID `$match` after the partner lookup because the fields live there. With no filters the stage list is shorter, which is why index-based tests break.
- **Common mistakes:** Putting the partner filters first (the field does not exist yet).
- **Follow-up questions:** How would you filter partners first? (Query `CPGE_VI_PARTNERS_ALL` for matching ids, then `$in` on `CP_PART_ID`.)

### Q40. What is the Java behaviour of `Long.parseLong` with blank input, and where does it bite?
- **Why the interviewer asks:** Edge case with a documented test.
- **Ideal answer:** `Long.parseLong("")` throws `NumberFormatException`. In `Template12LandscapeDispatchHandler.fetchDispatch` an empty `pContractId` throws and the exception is not caught, so the whole Template12 request fails with 500 (tests `fetchDispatch_nonNumericContractId_throwsNumberFormatException` and `execute_landscapeDispatchHandlerThrows_notWrappedInTryCatch_propagates`). Other handlers use `asNum`/`safeNum` with fallback. Fix: validate input early and return 400.
- **Common mistakes:** Catching the exception and returning a partial report silently.
- **Follow-up questions:** Which HTTP status does it give now? (500 via the catch-all handler.)

### Q41. How do `"72".equalsIgnoreCase(x)` and `x.equalsIgnoreCase("72")` differ in this code?
- **Why the interviewer asks:** Null-safety reading of code vs Jasper.
- **Ideal answer:** Jasper expressions like `$F{ENDT_CODE}.equalsIgnoreCase("72")` throw NPE on null. Java services write `"72".equalsIgnoreCase(x)`, null-safe, with defaults "N" for flags. So where Jasper would fail, Java quietly closes the gate and prints nothing. That is more tolerant but can hide missing data (for example all-null main DTO).
- **Common mistakes:** Assuming identical behaviour.
- **Follow-up questions:** Where could null hide a bug? (`AspEndtDocumentMainHandler` all-null DTO closes 72/168 and 070412 gates.)

### Q42. Explain the HttpClient5-backed `RestTemplate` configuration.
- **Why the interviewer asks:** Outbound HTTP resilience.
- **Ideal answer:** `PlineRestTemplateConfig` builds a pooled HttpClient 5 (max 100 total, 20 per route, connection-request timeout 3 s), connect 5 s, response/socket 20 s, and a custom truststore (`mytruststore.p12`, PKCS12) loaded via `SSLContexts.custom().loadTrustMaterial(trustStore, null)`, which replaces the JVM default cacerts. A separate `fastLookupRestTemplate` has an 8 s read. Wrong truststore password causes a startup failure. Motor `PolicyRefClient`/`PMEGetContractIdClient` create `new RestTemplate()` with no timeouts (not used by these modules).
- **Common mistakes:** Not knowing the truststore replaces default CAs.
- **Follow-up questions:** Retries? (None in Plines clients; add with care for non-idempotent calls - these are GETs.)

### Q43. `@Primary` vs `@Qualifier`: which RestTemplate does each Plines client get?
- **Why the interviewer asks:** Bean selection trap.
- **Ideal answer:** `PlineUtilsClient` uses `@Qualifier("fastLookupRestTemplate")` (8 s). The three `Pline*Client` classes inject an unqualified `RestTemplate`, so Spring picks the single `@Primary` bean, the motor `restTemplate` (5 s connect, 20 s read, no connection-request timeout, library default pool). The `plineRestTemplate` is not what they get. Fix by adding `@Qualifier("plineRestTemplate")` if that was intended [CONFIRM: with the team].
- **Common mistakes:** Assuming the bean with the matching name is chosen.
- **Follow-up questions:** What happens with two `@Primary` beans? (Startup error.)

### Q44. Why did `UriComponentsBuilder` and `getForObject(String url)` cause trouble?
- **Why the interviewer asks:** URL encoding details.
- **Ideal answer:** `restTemplate.getForObject(String url, ...)` treats `{}` as URI template variables and may re-encode `%` and `#`; the code comments near `PlineUtilsClient:~597-606` document this. `PlineUtilsClient` builds URLs with `UriComponentsBuilder.fromUriString(...).queryParam(...)`. `PlinePolicyRefClient` uses `build(true)`, which assumes the parts are already encoded. Boot 4 removed `fromHttpUrl`. `sanitize()` allows `. / # -` in values, and builder encoding keeps them in the query only.
- **Common mistakes:** Double encoding (`%2520`).
- **Follow-up questions:** Safer API? (`RestClient` with URI variables, or a `URI` object.)

### Q45. How is `${API_BASE-URL}` read, and what is the override story?
- **Why the interviewer asks:** Config binding details.
- **Ideal answer:** `@Value("${API_BASE-URL}")` in about 45 classes looks up the exact property name `API_BASE-URL`. The OS environment source maps names by replacing `.`/`-` with `_` and upper-casing, so env var `API_BASE_URL` should resolve it (inferred; verify by logging the outbound URL). `export API_BASE-URL=x` is invalid in POSIX shells; Kubernetes `env:` can carry it. `${key:default}` gives a default (for example `plines.api.customer.signature.type:UW`, not defined in the file). `plines.api.base-url` is a separate key.
- **Common mistakes:** Changing only one of the two base-URL keys.
- **Follow-up questions:** How to test the override? (JVM arg `-DAPI_BASE-URL=...` and a log line.)

### Q46. `@PropertySource` vs `application.properties` vs profile files.
- **Why the interviewer asks:** Spring config precedence.
- **Ideal answer:** `application-env.properties` is not a profile file; two config classes load it with `@PropertySource("classpath:/application-env.properties")`. `@PropertySource` has lower precedence than `application.properties`, env vars and system properties. `spring.profiles.active=dev` activates nothing because no `application-dev.properties` exists and there is no `@Profile` use. So "profiles" give no environment separation here.
- **Common mistakes:** Believing an `env` profile loads that file.
- **Follow-up questions:** How would you introduce real profiles? (`application-uat.properties` + env override, secrets from a store.)

### Q47. How does the Mongo URI get into the app, and what are the rules for local runs?
- **Why the interviewer asks:** Secret handling in code and config.
- **Ideal answer:** Boot 4 binds `spring.mongodb.uri` and `spring.mongodb.database` to `MongoProperties` and creates `MongoTemplate`. For local runs use `SPRING_MONGODB_URI` / `SPRING_MONGODB_DATABASE` env vars or `--spring.mongodb.uri=...` instead of editing the file, so nothing sensitive is committed. Pool options (maxPoolSize, waitQueueTimeoutMS) go in the URI options. The driver is 5.8.1; Spring Data MongoDB 5.1.1.
- **Common mistakes:** Editing the tracked file and committing it.
- **Follow-up questions:** What if the old key is used? (Silent fall back to localhost:27017.)

### Q48. How does `CommonController` handle `Map<String,String> filterCondition` when the JSON has an array?
- **Why the interviewer asks:** Binding behaviour.
- **Ideal answer:** A JSON array or number inside `filterCondition` fails binding because the map type is `Map<String,String>`. `CustomRequestFilter` wraps the request, reads the body and converts only `pBranch` and `pBranchName` arrays to comma-joined strings, so they bind. `PayLoadRequest.setMetadata(Map)` keeps String and Number (numbers `toString()`), lifts `mergeStaticPdfFiles`, and drops booleans, nulls and nested objects.
- **Common mistakes:** Sending `"pContractId": 123` and expecting it to bind.
- **Follow-up questions:** What status does a bad body give? (500 via the catch-all handler - inferred.)

### Q49. Why is `org.json` in `CustomRequestFilter` a concern?
- **Why the interviewer asks:** Library risk reading.
- **Ideal answer:** `org.json:json:20231013` parses attacker-controlled JSON with no size limit, may be exposed to old CVEs (inferred), and the filter prints bodies with `System.out.println`. The filter wraps `HttpServletRequest` and re-serves the body (the usual way is an `HttpServletRequestWrapper` with a new `ServletInputStream`; exact code not re-read, inferred). Upgrade the library, cap body size, and use Jackson instead.
- **Common mistakes:** Forgetting the body stream can be read only once without a wrapper.
- **Follow-up questions:** Does it apply to every request? (Yes, it is a `@Component Filter`.)

### Q50. What is the `ReportStrategy` contract for a new report in this style, as code?
- **Why the interviewer asks:** Can you write the skeleton?
- **Ideal answer:** A `@Service("AGIC/UW/Plines/<Area>/<Report>")` class implementing `ReportStrategy`: read `filterCondition` with `getOrDefault`; call the handler(s); build `PlineReportData` objects (one per row or sub-report) with `prntDetails/transDetList/transDet`; build `Transactions` through the shared helpers; call `PlineReportNames.tag(transactions, getClass())` to fill `mainReportName`; return. Add handler + DTO + payload classes, XSD mapping, and tests (service gate tests and handler pipeline tests). See `RejectRiskService` as the smallest example. [CONFIRM: you can write this from memory in an interview.]
- **Common mistakes:** Returning null `Transactions` (the controller then builds a default empty transaction).
- **Follow-up questions:** Where is `mainReportName` derived? (`PlineReportNames.mainNameOf` from the bean path.)
