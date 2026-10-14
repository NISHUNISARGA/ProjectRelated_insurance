# 10d - Database Questions 51-100 (MongoDB mirrored from Oracle)
> **AUTHORSHIP UPDATE (confirmed by the developer): git author `GS <extern.gs_nisarga@allianz.com>` is the developer. Wherever this file says commits cannot be attributed to the developer, read it as: the 11 GS commits listed in `KT_Parts/14_My_Commits_And_Ownership.md` ARE theirs; commits by other authors still must not be claimed. No GS commit touches `src/test`, so test authorship is still `[CONFIRM]`.**

Evidence base: same as 10c. Oracle-to-Mongo statements about the original SQL come from handler javadocs, the RejectRisk JRXML query and the KT deep dives. General MongoDB behaviour statements are standard documented behaviour but are not tested in this repo; they are marked (inferred) where they depend on server version or cluster settings. Personal facts are [CONFIRM: ...]. No credentials are printed.

---

### Q51. How do you translate an Oracle INNER JOIN into a pipeline in this codebase?
- **Why the interviewer asks:** Join semantics are the core of the migration.
- **Ideal answer:** `$lookup` into the second collection, then `$unwind` WITHOUT `preserveNullAndEmptyArrays` (rows with no match are dropped), or `$match {"ARR.0": {$exists:true}}` / `$match {$expr:{$gt:[{$size:"$ARR"},0]}}`. The auto-renewal handler's second branch joins `PSUW_BATCH_AUTOREN_STATUS` to `WIP_UWGE_POLICY_BASES` in this style (`buildJoinAndProjectStages(..., true, ...)`). In the shared-lookup stage of ASP schedule the code instead keeps rows (LEFT) with `preserveNullAndEmptyArrays: true`. Choose per JRXML SQL, not by habit.
- **Common mistakes:** Using LEFT semantics for everything and returning extra rows.
- **Follow-up questions:** How do you tell from the JRXML? (`INNER JOIN` vs `LEFT JOIN`/`(+)` in the SQL.)

### Q52. Translate a correlated scalar sub-select that picks the latest version.
- **Why the interviewer asks:** Common Oracle pattern with ordering inside.
- **Ideal answer:** In `AspEndtScheduleHandler` the annual premium is found with a `$lookup` whose sub-pipeline does `$match` with `$expr` `$lte ["$VERSION_NO","$$pv"]` and `$gt ["$ANNUAL_PREM", 0]`, then `$sort {VERSION_NO:-1}`, `$limit 1`, `$project ANNUAL_PREM`. Then `$unwind` with `preserveNullAndEmptyArrays: true`. This mirrors an Oracle sub-select ordered by version descending with ROWNUM = 1. `$sort` + `$limit` inside the sub-pipeline is optimised as top-1.
- **Common mistakes:** Forgetting the `$sort` (arbitrary row) or using `$last` without sorting.
- **Follow-up questions:** Index to support it? (`{CONTRACT_ID:1, VERSION_NO:-1}` recommended.)

### Q53. Translate `DECODE` or `CASE WHEN` to Mongo.
- **Why the interviewer asks:** Expression translation.
- **Ideal answer:** `$cond` for two branches and `$switch {branches:[{case, then}], default}` for several. Remember that comparison with a missing field behaves differently from SQL NULL logic: `$eq: ["$f", null]` is true for missing. The checklist branching `$eq ["$$chk","N"]` in ASP is this type. Always set `default` in `$switch` or it throws when no branch matches.
- **Common mistakes:** Omitting `default`. Using `$cond` with `$f == "Y"` string comparisons on mixed types.
- **Follow-up questions:** Oracle `DECODE(a,b,c,d)` NULL-equals-NULL semantics? (DECODE treats two NULLs as equal; `$eq` of two missing values is also true, but code should state it.)

### Q54. Translate `NVL(x, y)` and discuss Oracle's empty-string-equals-NULL rule.
- **Why the interviewer asks:** A subtle data-semantics gap in migrated data.
- **Ideal answer:** `NVL` -> `$ifNull: ["$x", y]`. `$ifNull` replaces null or missing, but NOT the empty string `""`. In Oracle an empty varchar IS null, so `NVL('', 'N')` returns 'N'. If replicated data holds `""` in Mongo (loader-dependent, inferred), `$ifNull` would keep `""` and the Jasper-expected default would not apply. Where a default must apply to blanks, use `$cond` with `$in: ["$x", [null, ""]]`.
- **Common mistakes:** Assuming `$ifNull` is identical to `NVL` for strings.
- **Follow-up questions:** How to check the data? (`$match {f: ""}` count.)

### Q55. Translate `TRUNC(date)` and date arithmetic.
- **Why the interviewer asks:** Date functions appear in many JRXML queries.
- **Ideal answer:** On filters, prefer range comparisons (`$gte` start of day, `$lt` next day) so indexes stay usable, as RejectRisk does. In projections, `$dateTrunc` (5.0+) or `$dateToString`/`$dateFromString`. Oracle `date + 1` -> `$dateAdd` or a Java-computed bound. Avoid wrapping an indexed field in a function inside `$match`.
- **Common mistakes:** `$dateTrunc` in the `$match` field side.
- **Follow-up questions:** Time zone parameter? (Specify `timezone` explicitly; default UTC.)

### Q56. Translate `TO_CHAR(date, 'DD-MON-YYYY')`.
- **Why the interviewer asks:** Formatting responsibilities.
- **Ideal answer:** `$dateToString {format: "%d/%m/%Y", date: "$D", timezone: ...}` for numeric formats (the KT doc mentions a `%d-%b-%Y` usage around `PlineUtilsClient` line 548; confirm it before citing). To my knowledge `$dateToString` has no month-name specifier like Oracle's `MON` (inferred from the documented specifier list; verify on your server version), so month names are usually formatted in Java. Also XSD `xs:dateTime` fields should carry ISO dateTime, not formatted text. Where the Jasper report formatted a date for display, decide whether the XML should carry the typed date or the formatted string (XSD files discuss this conflict).
- **Common mistakes:** Formatting dates in Mongo and then putting the string into a dateTime leaf.
- **Follow-up questions:** Which approach did RejectRisk take? (Typed `LocalDateTime` in the service, formatted `dd/MM/yyyy` by annotation, which is itself a conflict with the XSD.)

### Q57. Translate `WHERE ROWNUM <= N` and top-N queries.
- **Why the interviewer asks:** ROWNUM variants.
- **Ideal answer:** Two different Oracle meanings: (a) `ROWNUM` as a numbering column over an ordered inner query (RejectRisk) -> `$setWindowFields` + `$documentNumber`; (b) `WHERE ROWNUM = 1` or `<= N` to limit -> `$sort` (if the SQL orders) then `$limit`. Important: Oracle applies `ROWNUM` BEFORE `ORDER BY` in the same query level; the correct Mongo translation depends on whether the SQL ordered in an inner query.
- **Common mistakes:** `$limit` before `$sort`.
- **Follow-up questions:** Example in your code? (RejectRisk uses (a); ASP/Template12 point queries use `$limit 1` after `$match`.)

### Q58. Translate `WHERE x IN (SELECT ...)`.
- **Why the interviewer asks:** Sub-query patterns.
- **Ideal answer:** Either a `$lookup` with a `$match` on the result, or two steps: query the sub-select first into a Java list and use `$in`. ASP debit note does the latter for master policies (`MST_POL_NO` list then `OCP_POLICY_BASES.POLICY_REF $in [...]`). Trade-off: simple and indexable, but N round trips and size limits for the list; an empty list makes `$in` match nothing, which equals Oracle's empty IN.
- **Common mistakes:** Passing huge lists (stage size/16 MB limits).
- **Follow-up questions:** What if the first query returns duplicates? (`$in` is set-like; fine.)

### Q59. Translate `EXISTS` / `NOT EXISTS`.
- **Why the interviewer asks:** Semi-join patterns.
- **Ideal answer:** `$lookup` with a `$limit 1` sub-pipeline and `$project {_id:1}`, then `$match {"arr.0": {$exists:true}}` for EXISTS or `{arr: {$size: 0}}` for NOT EXISTS. The `$limit 1` keeps it cheap. Do not `$unwind` since you only need existence and duplicates would multiply rows.
- **Common mistakes:** Using `$unwind` (row multiplication).
- **Follow-up questions:** Alternative? (Two queries and a Java set.)

### Q60. Oracle `UNION` versus `UNION ALL`: what does `$unionWith` do?
- **Why the interviewer asks:** A classic correctness trap with `$unionWith`.
- **Ideal answer:** `$unionWith` appends the results of another collection/pipeline without removing duplicates, i.e. `UNION ALL`. If the SQL used `UNION` (dedupe), add a `$group` on the full row key afterwards. In `AspEndtScheduleHandler` live and WIP are disjoint by design (each side is a different source), so no dedupe is needed; verify per handler by reading the SQL.
- **Common mistakes:** Assuming `$unionWith` dedupes.
- **Follow-up questions:** What if the same row exists in both live and WIP? (Duplicate output; business rule must prevent or dedupe.)

### Q61. Translate `GROUP BY` with `SUM` and `HAVING`.
- **Why the interviewer asks:** Aggregation breadth.
- **Ideal answer:** `$group {_id: "$KEY", total: {$sum: "$AMT"}}` then `$match {total: {$gt: 0}}` for HAVING. In Template12 the sums use `_id: null` (single total). For decimals, `$sum` over Double is floating; if the source is Decimal128 the sum stays decimal. Money should be cross-checked against Oracle totals in testing.
- **Common mistakes:** Applying `$match` on the aggregated field before `$group`.
- **Follow-up questions:** How does `$group` interact with memory? (Q19 of 10c; `allowDiskUse`.)

### Q62. Oracle `NUMBER` versus Mongo numeric types for money and ids: what is the mapping risk?
- **Why the interviewer asks:** Precision awareness.
- **Ideal answer:** Oracle `NUMBER(p,s)` is exact decimal. Replication may write Double (binary floating point) or Decimal128 (exact). Doubles can show `0.1+0.2` effects; Decimal128 maps to Java `Decimal128`/`BigDecimal`. DTOs use `BigDecimal` for XSD `xs:decimal` leaves (387 decimal leaves in `Pline.xsd`). Large ids above 2^53 would lose precision as Double. Which types the loader uses is not in the repo [CONFIRM: field types in `UWGE_POLICY_BASES`].
- **Common mistakes:** Using `double` fields in DTOs for money.
- **Follow-up questions:** How to check field types? (`$type` queries or `$bsonType` in aggregation.)

### Q63. Oracle sorts strings with NLS rules, Mongo with binary comparison by default. Impact?
- **Why the interviewer asks:** Ordering parity.
- **Ideal answer:** Default Mongo string sort is binary on UTF-8 bytes (upper case before lower, no accent folding) unless a collation is set. Oracle's default `NLS_SORT=BINARY` is similar, but some environments use linguistic sort (inferred; unknown). For `ORDER BY NAME_EXT` in RejectRisk mixed-case names could be ordered differently from the original report if NLS differs [CONFIRM: NLS_SORT]. A collation (`locale`, `strength`) can be specified on the aggregate.
- **Common mistakes:** Stating Oracle and Mongo always match.
- **Follow-up questions:** Does collation affect index use? (The index must use the same collation.)

### Q64. Translate `LIKE '%x%'`.
- **Why the interviewer asks:** Search semantics and performance.
- **Ideal answer:** `$regex` with escaped input. A leading wildcard cannot use an index efficiently (scan of the index or collection). With user text, always escape metacharacters (`Pattern.quote`) to avoid ReDoS/operator-like input. RejectRisk avoids this by using equality on `NAME_EXT`, matching the JRXML (`=`). Introducing contains-search would change report behaviour and performance.
- **Common mistakes:** Passing raw user input to `$regex`.
- **Follow-up questions:** Case-insensitive options? (`i` flag disables index use for most cases; use collation.)

### Q65. Oracle views were turned into `*_VI_*` collections. What can go wrong with the view logic?
- **Why the interviewer asks:** Logic duplication risk.
- **Ideal answer:** A view contains joins/filters/formatting. If replicated as a flat collection, logic is frozen at load time: when the Oracle view definition changes, the Mongo copy changes only if the loader is updated. If implemented as Mongo views, the pipeline lives in Mongo. Handlers query these names as if they were Oracle (`UWGE_VI_ENDT_REPORT_PARAM`, `UWPL_VI_POLICY_SCHEDULE_BASE`). Debugging a wrong value requires knowing the view definition [CONFIRM: owner of replication].
- **Common mistakes:** Debugging the handler when the replicated data is wrong.
- **Follow-up questions:** Check? (Compare one row between Oracle view and Mongo.)

### Q66. How fresh is the Mongo data compared with Oracle, and how do you handle a "data not found" complaint right after policy issue?
- **Why the interviewer asks:** Replication lag awareness.
- **Ideal answer:** The repo says nothing about the replication mechanism or lag. If a policy was just issued, the Mongo copy may not yet have it, so the report is empty/incomplete and the API returns 200. Triage: check the document in Mongo by `CONTRACT_ID`; if missing, wait/ask the replication owner; do not touch handler code. Document the expected lag with the data team [CONFIRM: lag SLA].
- **Common mistakes:** "Fixing" a handler for a missing-data issue.
- **Follow-up questions:** Does `checklistCopy=Y` (WIP) have the same lag? (Unknown.)

### Q67. Sequential queries have no common snapshot. Is that a risk and how would you reduce it?
- **Why the interviewer asks:** Consistency reasoning.
- **Ideal answer:** Template12 and ASP debit-note logic issue several independent reads. Between steps, data could change (for example an endorsement posts), so the document could mix versions. Probability is low for read-mostly policy data (inferred), but real for WIP data being edited. Options: read concern "snapshot" in a multi-document transaction (heavy, rarely used for reporting), a single pipeline, or compare version numbers at the end and retry. I would document the risk and not over-engineer.
- **Common mistakes:** Claiming Mongo gives snapshot isolation by default across calls.
- **Follow-up questions:** Does the code set read concern? (Not seen.)

### Q68. Read preference and replica sets: does the code choose?
- **Why the interviewer asks:** Replica set behaviour.
- **Ideal answer:** The URI is Atlas-style (multi-host, replicaSet, ssl) and the code sets no read preference per query, so the URI default (primary) applies unless the URI includes `readPreference` (the URI is `<redacted>`, not inspected for options [CONFIRM]). Reading from secondaries would lower load on the primary but can return slightly stale data, which matters for fresh WIP data.
- **Common mistakes:** Assuming secondaries are used because it is a replica set.
- **Follow-up questions:** Which collections are safe to read from secondaries? (Static code tables such as `CMGE_CODE`.)

### Q69. `option.min-connection-pool-size`, `option.max-connection-pool-size`, `option.max-wait-time` are in `application.properties`. Are they applied?
- **Why the interviewer asks:** Detecting ineffective config.
- **Ideal answer:** No. Nobody reads these keys (no `@Value`, not a Boot property). So the driver defaults apply: `maxPoolSize` 100, `minPoolSize` 0, `maxWaitTime` 2 minutes (MongoDB Java driver defaults). To really set them put `maxPoolSize=100&minPoolSize=20&waitQueueTimeoutMS=2000` into the connection string options (inferred; Boot 4 has no dedicated pool properties; verify with the Boot reference). They came from the commit "Performance issue fixes." (`62b3c94a7`, 2026-01-28).
- **Common mistakes:** Believing the pool is 20-100.
- **Follow-up questions:** Symptoms of pool exhaustion? (Threads waiting on `MongoWaitQueueFullException`/timeouts.)

### Q70. Which connection-string options would you consider for this service?
- **Why the interviewer asks:** Practical production readiness.
- **Ideal answer:** `maxPoolSize`, `minPoolSize`, `waitQueueTimeoutMS`, `serverSelectionTimeoutMS` (fail fast at startup/outage; default 30 s), `socketTimeoutMS` and `connectTimeoutMS`, `readPreference`, `retryReads` (default true) and `appName` (for server logs). These go in the URI, which is a secret-bearing value, so it should come from a secret store or environment variable, not the committed file.
- **Common mistakes:** Putting options in unused `option.*` properties.
- **Follow-up questions:** How would `socketTimeoutMS` relate to `maxTime`? (Socket timeout kills the client wait; `maxTime` stops server work.)

### Q71. Why does the Boot 4 upgrade matter for the Mongo URI key?
- **Why the interviewer asks:** Real upgrade bug class (commit `9248b12ff`).
- **Ideal answer:** In Boot 4 `MongoProperties` moved to `org.springframework.boot.mongodb.autoconfigure` with prefix `spring.mongodb.*`. Old `spring.data.mongodb.uri` is silently ignored, and the app falls back to `localhost:27017`. Symptom: connection errors to localhost or empty results. The properties file has a comment block (lines ~26-31) warning about this.
- **Common mistakes:** Debugging the network when the key is wrong.
- **Follow-up questions:** How to verify the effective URI? (Startup logs; careful not to print credentials.)

### Q72. Is `auto-index-creation=false` doing anything?
- **Why the interviewer asks:** Ineffective configs and index ownership.
- **Ideal answer:** No. It is not under `spring.mongodb.*`/`spring.data.mongodb.*`, and there are no `@Document`/`@Indexed` entities anyway. Indexes are therefore managed outside the app (Atlas/DBA). Because there is no index code in the repo, one cannot see from the repo whether the indexes in Q38 exist.
- **Common mistakes:** Claiming the app creates indexes at startup.
- **Follow-up questions:** Where would index definitions live? (A migration script or infrastructure repo [CONFIRM].)

### Q73. How should Mongo credentials be handled for this service?
- **Why the interviewer asks:** Security hygiene directly visible in your working copy.
- **Ideal answer:** The working copy of `application.properties` has an uncommitted block containing a cleartext Mongo URI (`<redacted>`). Never commit it. Use environment variables (`SPRING_MONGODB_URI`, `SPRING_MONGODB_DATABASE`) or `--spring.mongodb.uri=` at run, or a mounted secret. History: commit `7dd2f2f67` removed the UAT URI lines from the committed file; `gitleaks.toml` allowlists the two properties files wholesale (OPUSM-50404), so scanners would not flag new secrets there. Rotate any credential that was ever committed.
- **Common mistakes:** Printing the URI in a chat or ticket.
- **Follow-up questions:** What is the gitleaks risk? (File-wide allowlist hides new secrets; review-by date 2026-10-23.)

### Q74. A `$lookup` returns an array. Is there a size limit?
- **Why the interviewer asks:** Hidden MongoDB limits.
- **Ideal answer:** The result document of the stage must stay within the 16 MB BSON document limit, so a `$lookup` that returns many matches per row into an array field can fail ("document constructed by $lookup is too large"). That is why handlers use `$limit:1` for scalar lookups and `$project` to keep only needed fields. For child lists (covers, risks) the code fetches them with separate queries or per-contract limits, not a giant array.
- **Common mistakes:** Looking up whole child collections into one array.
- **Follow-up questions:** What is the pipeline stage memory limit? (Separate: 100 MB for blocking stages unless disk use.)

### Q75. What is the effect of the 16 MB document limit on your modules?
- **Why the interviewer asks:** Capacity thinking.
- **Ideal answer:** Source documents are small relational rows. The risk is only in intermediate documents with embedded arrays (`$lookup` results, `CARD_PERSONS` arrays in the card printing handler) for contracts with very many members (group policies with large member lists). ASP "AddMemberLandscape" and "DeleteMemberLandscape" deal with member lists, so a large endorsement could stress array-shaped pipelines [CONFIRM: typical member counts].
- **Common mistakes:** Dismissing it as theoretical.
- **Follow-up questions:** Mitigation? (`$unwind` early, process per member, or paginate.)

### Q76. Which MongoDB server versions do these pipelines require?
- **Why the interviewer asks:** Compatibility awareness.
- **Ideal answer:** `$setWindowFields` and `$documentNumber`: 5.0+. `$unionWith`: 4.4+. `$lookup` with both `localField`/`foreignField` and a `pipeline`: 5.0+ (concise correlated form). `$dateTrunc`/`$dateAdd`: 5.0+. `$toDouble`: 4.0+. The cluster is Atlas-style, version not in the repo [CONFIRM: cluster version]. If the cluster were below 5.0, RejectRisk's window stage would fail at runtime, not at startup.
- **Common mistakes:** Not knowing that feature gaps fail at runtime.
- **Follow-up questions:** How to find version? (`db.version()` / Atlas UI.)

### Q77. How large can an `$in` list be in the master-policy chain?
- **Why the interviewer asks:** Practical limits.
- **Ideal answer:** A BSON command is limited to 16 MB plus overhead; a list of strings of this kind is far below that unless thousands of values. The real concern is performance: a huge `$in` multiplies index probes. For the master policy chain the list is small (a few `MST_POL_NO` values per contract; inferred). Validate with a real big master policy [CONFIRM].
- **Common mistakes:** Claiming `$in` has a hard low limit like Oracle's 1000-item limit; Mongo has no such limit.
- **Follow-up questions:** Oracle limit? (1000 items in an IN list; one reason Oracle SQL used sub-selects.)

### Q78. Why project early and keep documents narrow?
- **Why the interviewer asks:** Memory and speed.
- **Ideal answer:** Wide documents (some `UWPL_VI_*` views have many columns; transDet has up to ~650 output elements) cost memory in later stages and in the driver. Using `$project` in `$lookup` sub-pipelines (as `AspEndtScheduleHandler` does for product name, agent) transfers only needed fields. Early `$project` after the first `$match` reduces memory in `$sort`/`$group`. Do not project away fields later stages need.
- **Common mistakes:** Projecting `_id: 0` and then needing it for `$group`.
- **Follow-up questions:** Does the optimizer auto-prune? (It can detect dependencies and project automatically in some cases; do not rely on it.)

### Q79. Describe index recommendations for contract-keyed reads in Template12.
- **Why the interviewer asks:** Concrete indexing plan for your module.
- **Ideal answer:** `{CONTRACT_ID:1, VERSION_NO:1}` on `UWGE_POLICY_BASES`, `WIP_UWGE_POLICY_BASES`, `UWGE_POLICY_VERSIONS`, `UWPL_POLICY_BASES`, `UWPL_POLICY_JOURNEY`, `UWPL_RISK_PERSON`, `UWPL_COVER` and their WIP twins, plus `UWPL_VI_POLICY_SCHEDULE_BASE` and `UWGE_VI_POL_REPORT_PARAM` if they are real collections. `{CONTRACT_ID:1}` on the `*_UWGE_POLICY_CONTRACTS` family. These are recommendations from query shapes, not repo facts.
- **Common mistakes:** Presenting them as existing.
- **Follow-up questions:** Priority? (The collections hit by every request first: bases and contracts.)

### Q80. Why do WIP twin collections double the index and replication work?
- **Why the interviewer asks:** Operational cost of the WIP pattern.
- **Ideal answer:** Every `X` has `WIP_X`, so each needs indexes and data loads. Writes/replication to both and storage cost. For reads, the WIP collections are probably small (only in-progress work; inferred), so a missing index hurts less, but `checklistCopy` previews are likely used by underwriters during peak hours.
- **Common mistakes:** Assuming WIP collections are tiny without checking.
- **Follow-up questions:** How to size them? [CONFIRM: counts].

### Q81. How do you read an `explain("executionStats")` output for a slow handler?
- **Why the interviewer asks:** Practical DB debugging.
- **Ideal answer:** Check `winningPlan` stages: `IXSCAN` good, `COLLSCAN` bad on big collections; compare `nReturned`, `totalKeysExamined`, `totalDocsExamined` (ratio near 1 is ideal), `executionTimeMillis`; for aggregation, look at each stage (`$lookup` strategy, `$sort` uses memory or index). Capture the pipeline from `MongoTemplate` DEBUG logs (Q43) and run it in mongosh with the same literal values.
- **Common mistakes:** Judging only by execution time on small data.
- **Follow-up questions:** What does `hasSortStage` / in-memory sort indicate? (Missing sort-supporting index.)

### Q82. Handler tests mock `MongoTemplate`. What do they prove and not prove?
- **Why the interviewer asks:** Testing honesty for the DB layer (testing guide).
- **Ideal answer:** They prove the handler builds the expected stages (tests capture the `Aggregation` with `doAnswer` and call `toPipeline(Aggregation.DEFAULT_CONTEXT)`; 116 uses in 23 files) and maps mocked results to DTOs. They do not prove the pipeline runs on a real server, returns correct rows, uses an index or respects server version limits. No Testcontainers or embedded Mongo exist in `src/test`. A green test never validates query semantics.
- **Common mistakes:** Calling these integration tests.
- **Follow-up questions:** What would you add? (Testcontainers Mongo with seeded documents for RejectRisk, with a null-name row and a boundary-date row.)

### Q83. How would you test the RejectRisk date boundary and null sort against a real Mongo?
- **Why the interviewer asks:** Test design for the exact DB behaviours discussed.
- **Ideal answer:** Seed `CMUW_PL_REFER_RISK` with rows at `2026-01-31T23:59:59Z`, `2026-02-01T00:00:00Z` and a row with no matching partner. Assert: `pCreatedDateTo=2026-01-31` includes the first and excludes the second; missing `pCreatedDateFrom` still returns rows; null-name row position is documented (currently first; Oracle would put it last); row numbers are 1..n. Use Testcontainers (not present today) or an in-memory Mongo-compatible engine only if it supports `$setWindowFields` (many do not).
- **Common mistakes:** Embedded fakes that lack window functions.
- **Follow-up questions:** How to keep it fast? (Single container reused across tests.)

### Q84. How does `toPipeline` help in unit tests?
- **Why the interviewer asks:** Practical testing technique used in the repo.
- **Ideal answer:** `Aggregation.toPipeline(Aggregation.DEFAULT_CONTEXT)` converts the built operations into a `List<Document>` so a test can assert stage order and key values (for example `$match` contains `$gte` and `$lt` but no `$lte`; no `$match` with null). It is cheap and catches regressions in structure, such as the removed `ROW_NUMBER` stage. It cannot catch runtime errors.
- **Common mistakes:** Asserting on the whole pipeline string (brittle).
- **Follow-up questions:** What to assert instead? (Specific stages by operator name.)

### Q85. Customer data (`CPGE_VI_PARTNERS_ALL`) includes ID numbers. What DB-level privacy controls exist?
- **Why the interviewer asks:** Data governance awareness.
- **Ideal answer:** Nothing in this repo: the service reads with whatever user the URI contains, with no field-level masking. Controls would be at the DB (role-based access, field-level encryption) and the API (who may call). The RejectRisk report intentionally shows ID type/value. Logging of pipelines leaks values (Q43). The compliance requirement (PDPA in Malaysia, inferred) belongs to the data owner [CONFIRM].
- **Common mistakes:** Saying Mongo encryption at rest solves leakage via logs.
- **Follow-up questions:** Least-privilege? (A read-only DB user restricted to needed collections.)

### Q86. Why join on both `PART_ID` and `VERSION`?
- **Why the interviewer asks:** Data model understanding.
- **Ideal answer:** `CMUW_PL_REFER_RISK` stores `CP_PART_ID` and `CP_PART_VERSION`, referencing the exact partner (customer) version recorded at reject time. `CPGE_VI_PARTNERS_ALL` has rows per (PART_ID, VERSION); joining only on id could return the wrong version's name or multiple rows. The pair gives the historical name at the time. This is why the lookup matches both fields.
- **Common mistakes:** Joining on `PART_ID` only.
- **Follow-up questions:** What if the version row is missing? (LEFT semantics: customer fields null.)

### Q87. Leading zeros and text ids: what is the risk when ids are numeric in Mongo?
- **Why the interviewer asks:** Type conversion pitfalls.
- **Ideal answer:** Fields that are numeric in Mongo lose leading zeros; fields that are text keep them. Policy numbers like `070412/2026/0001` and code values like `CODE_CD` are strings; matching a numeric request value to a text code fails. `REASON_CODE` e.g. "09" in the sample output is text. Converting such codes with `safeNum` would turn "09" into 9 and not match (inferred risk). Only convert fields known to be numeric: `CONTRACT_ID`, `VERSION_NO`.
- **Common mistakes:** Converting every id to Long.
- **Follow-up questions:** How to find a field's type? (Sample document in mongosh.)

### Q88. How are dates stored and displayed across layers?
- **Why the interviewer asks:** Cross-layer date flow, linking DB with XSD.
- **Ideal answer:** DB: BSON Date (for example `CREATED_DATE`). Service: `LocalDateTime` (`toLocalDateTime` makes `atStartOfDay()` for request dates). XML: `@JsonFormat(pattern="dd/MM/yyyy")` writes `06/06/2017`, while XSD v0.1.6 demands `xs:dateTime` (`2017-06-06T00:00:00`). Jasper original displayed `dd/MM/yyyy`. So the conflict is between display format and schema type, not a DB matter, but the DB value is fine.
- **Common mistakes:** Changing the DB to fix the XML.
- **Follow-up questions:** What would a clean fix look like? (ISO in XML and formatting in the template, or an XSD pattern type.)

### Q89. What is the cost of many round trips per request on Atlas?
- **Why the interviewer asks:** Latency model.
- **Ideal answer:** Each aggregate is a network round trip (single-digit milliseconds in-region, more cross-region; unknown here). Template12 may run tens of small queries, ASP debit note 4-5 chained. At 100 round trips x 5 ms, roughly half a second (an illustration, not a measurement). Plus external calls. Also each request borrows connections from the driver pool for the duration. Measure before optimising [CONFIRM: latency figures].
- **Common mistakes:** Quoting numbers as facts.
- **Follow-up questions:** Optimisation? (Combine independent lookups into one pipeline, cache static code tables.)

### Q90. Should `CMGE_CODE` lookups be cached in the application?
- **Why the interviewer asks:** Cache design judgement.
- **Ideal answer:** Code tables change rarely, are small, and are used on every request, so caching `(CAT_CODE, CODE_CD) -> CODE_DESC` is attractive. Risks: stale descriptions after a code change, memory, and the existing design (per-request `$lookup`) which avoids cache invalidation. If added, use a short TTL (minutes) and a cache size bound, and keep an escape hatch to bypass. Not currently implemented; decision needs business agreement.
- **Common mistakes:** Caching without TTL.
- **Follow-up questions:** Where would you put it? (A small component used by handlers; Caffeine with TTL.)

### Q91. Duplicate keys in a code table would break `$limit:1`. How would you detect them?
- **Why the interviewer asks:** Data validation technique.
- **Ideal answer:** `db.CMGE_CODE.aggregate([{ $group: {_id: {c:"$CAT_CODE", d:"$CODE_CD"}, n:{$sum:1}} }, { $match: {n:{$gt:1}} }])`. If duplicates exist, the lookup returns an arbitrary row (Q10). Run it for the categories used by my modules: `PLREJ_RISK_TYPE`, `PLREJ_REASON`, `PLREJ_REVOKE_DUR`, `PL_PRODUCT`. Result belongs to the data team [CONFIRM].
- **Common mistakes:** Assuming Oracle's constraint carried over in replication.
- **Follow-up questions:** Add a unique index? (Only if data is clean; it protects future loads.)

### Q92. How would you reconcile the Mongo-based output with the original Oracle/JasperServer report?
- **Why the interviewer asks:** Migration validation.
- **Ideal answer:** Choose sample contracts/date ranges; run the original JRXML on Oracle and this service for the same parameters; compare row counts, totals, ordering and key fields (per the mapping sheet). Differences usually come from: NULL ordering (Q16), date boundaries (Q6), rounding (Q34), WIP vs live, lookup version choice. Record variance in a sheet. The repo has no automated comparison [CONFIRM: if the team did parity testing].
- **Common mistakes:** Comparing only counts.
- **Follow-up questions:** Who signs off? (Business/UAT owner.)

### Q93. What does `retryReads` do and does it matter here?
- **Why the interviewer asks:** Driver resilience.
- **Ideal answer:** The Java driver retries a read once on transient network errors or primary elections by default (`retryReads=true`). For read-only aggregations this is safe and helps during replica set failovers. It does not retry application-level errors or timeouts at `maxTime`. The repo does not change the default (URI options not inspected).
- **Common mistakes:** Adding application retry loops around reads.
- **Follow-up questions:** Does `$out`/`$merge` retry? (Not used here.)

### Q94. How do you design the `transDet` data assembly when several collections feed one XML node?
- **Why the interviewer asks:** Combining multiple DB results into one XSD structure.
- **Ideal answer:** The service calls several handlers, receives DTOs, and sets leaves onto one `PlineTransDet` (T12 base service builds ~57 `PlineTransDet`, ASP ~20 per the XSD guide counts). Each leaf has a DB origin (view column), a DTO alias and an XSD leaf name; the mapping sheet documents it. Missing source rows produce null leaves which serialise as empty elements. A full trace for one leaf: collection field -> `$project` alias -> DTO field -> service setter -> XSD leaf.
- **Common mistakes:** Skipping the DTO naming step when debugging.
- **Follow-up questions:** How would you find the source of `grossPrem`? (Search the setter then the handler `$project`.)

### Q95. Is it safe for handlers to be singletons using shared static `AGG_OPTIONS`?
- **Why the interviewer asks:** Thread safety.
- **Ideal answer:** Yes if the object is immutable: `AggregationOptions` is built once and only read. Handlers hold injected `MongoTemplate` (thread-safe) and no mutable state. Per-request lists (`ops`) are local variables. The pattern is safe; mutable static collections or `SimpleDateFormat` fields would not be [CONFIRM by code review that none exist].
- **Common mistakes:** Making `ops` a field.
- **Follow-up questions:** What about `SimpleDateFormat`? (Not thread-safe; use `DateTimeFormatter`.)

### Q96. What happens during a Mongo failover while a request is running?
- **Why the interviewer asks:** Failure-mode reasoning.
- **Ideal answer:** In-flight aggregates may fail with a network or not-primary error; `retryReads` may recover single reads. Multi-step handlers may fail mid-way, leading to an exception and a 500 response (the generic handler hides the cause). No circuit breaker or degraded mode exists. The caller can retry the whole POST safely (read-only). Server selection timeout (default 30 s) delays failure.
- **Common mistakes:** Saying the app queues and recovers transparently.
- **Follow-up questions:** Should the API return 503? (Better signal for retry; needs a handler for Mongo exceptions.)

### Q97. How would you check data quality for a sample policy before blaming code?
- **Why the interviewer asks:** Practical triage routine.
- **Ideal answer:** For a `CONTRACT_ID`: confirm it exists in `UWGE_POLICY_BASES` (live) or `WIP_UWGE_POLICY_BASES`; check `VERSION_NO` values; check the report-param view (`UWGE_VI_POL_REPORT_PARAM` or `UWGE_VI_ENDT_REPORT_PARAM`) has the row; check child rows (`UWPL_RISK_PERSON`, `UWPL_COVER`); check types with `$type`. Compare with Oracle for the same contract if available. Only then investigate the handler.
- **Common mistakes:** Starting with the debugger.
- **Follow-up questions:** Which view drives show flags? (`UWGE_VI_POL_REPORT_PARAM` for policy, `UWGE_VI_ENDT_REPORT_PARAM` for endorsement.)

### Q98. What are the risks of depending on `CMGE_CODEREL_DET` and relationship tables in Template12?
- **Why the interviewer asks:** Reference-data dependency.
- **Ideal answer:** `CMGE_CODEREL_DET` (code relationships) is queried by the Template12 main and landscape dispatch handlers to map related codes (for example plan/scheme relationships; exact use not documented in my notes). Missing or changed relationship rows silently change which scheme or text is chosen. Treat it as business configuration: any change in reference data can alter output without code changes. Add tests that seed typical relationship rows.
- **Common mistakes:** Treating reference tables as static.
- **Follow-up questions:** How would you detect a changed relationship? (Golden-output regression for sample policies.)

### Q99. Which DB-level improvements would you make first, in order?
- **Why the interviewer asks:** Prioritisation with justification.
- **Ideal answer:** (1) Confirm and add indexes on `CMGE_CODE {CAT_CODE, CODE_CD}`, `CMUW_PL_REFER_RISK {CREATED_DATE}` and contract-keyed collections; check with `explain`. (2) Add `AggregationOptions` (`maxTime`, `allowDiskUse`) to RejectRisk and `AspEndtDocumentMainHandler`. (3) Fix RejectRisk NULL ordering parity and tie-breaking. (4) Apply the pool/timeouts through the URI. (5) Remove DEBUG Mongo logging in prod. Each is small and measurable.
- **Common mistakes:** Proposing sharding or a different database.
- **Follow-up questions:** How would you measure improvement? (Before/after `explain` and handler timing.)

### Q100. Tell me about one DB bug or subtle behaviour you handled in this project.
- **Why the interviewer asks:** Evidence of real ownership; the answer must be truthful.
- **Ideal answer:** A safe, evidence-based story: the RejectRisk date filter. The earlier code always appended both `$gte` and `$lte` bounds; with a missing bound, `{$gte: null}` matched only null dates and returned no rows, and `$lte` on the last day excluded most timestamps from that day. The working-copy fix appends only supplied bounds and shifts the upper bound by one day with `$lt`, keeping index use. Say clearly which parts you did yourself [CONFIRM: your actual contribution; git authorship cannot be attributed to you]. Do not invent incidents.
- **Common mistakes:** Claiming credit for commits you cannot confirm, or inventing numbers.
- **Follow-up questions:** How did you test it? (Handler test capturing the pipeline; note the KT guide says the fix may break about 7 of the 15 handler tests that expect the old shape.)
