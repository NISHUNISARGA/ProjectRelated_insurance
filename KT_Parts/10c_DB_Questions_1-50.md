# 10c - Database Questions 1-50 (MongoDB mirrored from Oracle)

Evidence base: `06_Config_API_DB_DevOps.md` Part 3, `01_RejectRisk_Deep_Dive.md`, `02_Template12_Deep_Dive.md`, `03_ASP_Deep_Dive.md`, and repo handlers (`RejectRiskHandler`, `AspEndtScheduleHandler`, `AutoRenewalReportAutoRenewalPolicyHandler`, `CardPrintingHandler`, `PlineT12PolicyDocumentHandler`, `Template12LandscapeDispatchHandler`). "(inferred)" = reasoning, not proven. Personal facts are [CONFIRM: ...]. Connection strings and credentials are never printed (`<redacted>`). No index definitions or DB scripts exist in the repo, so all index statements are recommendations.

---

### Q1. How does this service reach data, and why do collection names look like Oracle names?
- **Why the interviewer asks:** Opening question for the whole DB area; checks you understand the migration context.
- **Ideal answer:** The service uses `MongoTemplate.aggregate(Aggregation.newAggregation(ops), "<COLLECTION>", Dto.class)` with raw `org.bson.Document` stages. Collections mirror the Oracle tables and views the original JasperServer JRXML queries used (`UWGE_*`, `UWPL_*`, `CMGE_CODE`, `CPGE_VI_PARTNERS_ALL`, `*_VI_*` views). Handler javadocs quote the original SQL. How the data is replicated from Oracle (tool, frequency) is not in the repo [CONFIRM: replication method and lag].
- **Common mistakes:** Saying the app owns the schema or has `@Document` entities. It has neither.
- **Follow-up questions:** How do you know the Oracle equivalent of a collection? (Handler javadoc and JRXML.) What is the Oracle schema owner? (`CUSTOMER` appears in comments.)

### Q2. What do the prefixes `WIP_`, `SB_` and `OCP_` mean?
- **Why the interviewer asks:** Tests honesty about gaps versus invention.
- **Ideal answer:** `WIP_` marks the work-in-progress copy (not yet posted) used when `checklistCopy = Y`; unprefixed collections are live/posted. `SB_UWGE_POLICY_CONTRACTS` and `OCP_POLICY_BASES` appear in ASP handlers but their meaning is not documented in the code I read, so I say that and do not guess (the comment on `SB_` reads "WIP union SB contracts"). I would ask the data owner [CONFIRM: SB and OCP meaning].
- **Common mistakes:** Inventing expansions such as "standby" as fact.
- **Follow-up questions:** How does Template12 choose WIP versus live? (Java `isWip ? WIP_x : x`.) How does ASP? (Both through `$ifNull` and union patterns.)

### Q3. Are the `*_VI_*` names Mongo views or ordinary collections, and why does it matter?
- **Why the interviewer asks:** Index and performance reasoning.
- **Ideal answer:** Unknown from the repo. In Oracle they are views. In Mongo they may be materialised collections (loaded by replication) or real Mongo views. A real Mongo view has no indexes of its own; queries use the source collection's indexes. A materialised collection needs its own indexes. This decides whether `{CONTRACT_ID:1, ENDT_NO:1}` can be created on `UWMT_VI_ENDT_TAX_INV_ALT_UPV` [CONFIRM: with DBA].
- **Common mistakes:** Recommending `createIndex` on a view.
- **Follow-up questions:** How to check? (`db.getCollectionInfos({name:...})` shows `type: "view"`.)

### Q4. Why does the code use raw `Document` stages instead of Spring Data `@Document` entities and repositories?
- **Why the interviewer asks:** Design rationale of the data layer.
- **Ideal answer:** Pipelines are translated from Oracle SQL with `$lookup` sub-pipelines, `$setWindowFields`, `$unionWith`, computed fields. Typed `Aggregation` helper classes do not support all of these stages comfortably, and the data is read-only reporting data, so entities and repositories give little value. Cost: no compile-time checks of field names, tests must inspect the generated pipeline, and no automatic index creation (so `auto-index-creation` is irrelevant).
- **Common mistakes:** Saying raw documents are faster at runtime (they are not).
- **Follow-up questions:** How do you protect against field-name typos? (Tests that capture the pipeline with `toPipeline`, constants for names.)

### Q5. Describe the RejectRisk aggregation from first stage to last.
- **Why the interviewer asks:** Your own module; expects precise recall.
- **Ideal answer:** On `CMUW_PL_REFER_RISK`: `$match` on `CREATED_DATE` range (only supplied bounds); `$match` on `REJ_TYPE` / `REASON_CODE` if given; `$lookup` `CPGE_VI_PARTNERS_ALL` on (`PART_ID`,`VERSION`) = (`CP_PART_ID`,`CP_PART_VERSION`) with `$limit 1`, then `$addFields` CUST via `$arrayElemAt`; optional `$match` on `CUST.NAME_EXT` and `$or` on `CUST.ID_VALUE1/2`; three `$lookup`s to `CMGE_CODE` for categories `PLREJ_RISK_TYPE`, `PLREJ_REASON`, `PLREJ_REVOKE_DUR`; `$project` DTO aliases; `$setWindowFields` `$documentNumber` sorted by `customerName`; final `$sort customerName:1`. The working-copy change removed an earlier `$addFields ROW_NUMBER:1` constant.
- **Common mistakes:** Forgetting that the name/ID filters run after the customer lookup.
- **Follow-up questions:** Which stages can use an index? (Only the leading `$match`.) Why is the customer filter late? (Fields live in the joined collection.)

### Q6. Why is the upper date bound shifted by one day and compared with `$lt`?
- **Why the interviewer asks:** Date-range semantics are a classic source of off-by-one bugs.
- **Ideal answer:** The Oracle JRXML used `TRUNC(CREATED_DATE) BETWEEN from AND to`, inclusive of the whole last day. `CREATED_DATE` is a full timestamp. In Mongo the handler computes `to + 1 day` at start of day and uses `{$gte: from, $lt: to+1}`. This gives the same whole-day inclusion without a `$dateTrunc` on the field, so an index on `CREATED_DATE` stays usable. Comment in `RejectRiskHandler` explains it.
- **Common mistakes:** Using `$lte: to` (misses most of the last day) or wrapping the field in a function (blocks the index).
- **Follow-up questions:** What if `pCreatedDateTo` already has a time part? (Handler treats the parsed value as start of day; behaviour with time component is not tested.)

### Q7. What goes wrong if a date bound is null and you still write `{$gte: null}`?
- **Why the interviewer asks:** BSON comparison semantics from a real bug.
- **Ideal answer:** In MongoDB a comparison with `null` matches only null (or missing) values of that field, because of BSON type bracketing. So the earlier code, which always appended both bounds, returned zero real rows when a bound was missing. The fixed code appends a bound only if supplied: `if (dateFrom != null) append("$gte")`, same for `$lt`, and skips the `$match` entirely if the filter is empty.
- **Common mistakes:** Expecting `{$gte:null}` to mean "no lower bound".
- **Follow-up questions:** What is the new risk? (No date filter at all means a full scan; see Q18.) How is a bad date string handled? (Parse failure gives null = no bound.)

### Q8. `toDate` uses UTC start of day. Where could this cause a boundary error?
- **Why the interviewer asks:** Time zone awareness.
- **Ideal answer:** `LocalDate.atStartOfDay(ZoneId.of("UTC"))`. If Oracle stored local time (Malaysia is UTC+8, inferred from the business) and the replication wrote it as if it were UTC, boundaries are consistent. If the replication converted real local times to true UTC instants, rows between 00:00 and 08:00 local would fall on the previous UTC date. Which one applies is not in the repo [CONFIRM: how timestamps are stored].
- **Common mistakes:** Asserting "it is correct" without knowing the stored zone.
- **Follow-up questions:** How to check? (Compare a known Oracle row's `CREATED_DATE` with the Mongo value.)

### Q9. Why use `$lookup` with `let` and `pipeline` instead of `localField`/`foreignField`?
- **Why the interviewer asks:** Understanding of the two lookup forms.
- **Ideal answer:** RejectRisk joins on two fields (`PART_ID` + `VERSION`) and on constants (`CAT_CODE = 'PLREJ_RISK_TYPE'`), which needs a sub-pipeline with `$expr`. `let` passes the outer fields (`$$pid`, `$$pver`). `localField`/`foreignField` handles one equality. Some other handlers (e.g. `AspEndtScheduleHandler`) combine `localField`/`foreignField` with a `pipeline` for extra conditions, which needs a newer server version (inferred: 5.0+).
- **Common mistakes:** Using `$match` with `$eq` outside `$expr` (cannot reference `$$var`).
- **Follow-up questions:** Does `$expr` use an index? (Equality via `$expr` can use an index in recent versions; verify with `explain`.) What does `$limit:1` inside do? (Q10.)

### Q10. `$limit: 1` is inside each `$lookup` pipeline with no `$sort`. Any problem?
- **Why the interviewer asks:** Deterministic results.
- **Ideal answer:** `$limit:1` mimics a scalar sub-select and stops early. Without `$sort`, if multiple rows match, which one is returned is not defined. For `CMGE_CODE` with `CAT_CODE`+`CODE_CD` it is unique by design. For `CPGE_VI_PARTNERS_ALL` with `PART_ID`+`VERSION` it should be unique too; if duplicates existed, the customer name could vary between runs (inferred risk). The Oracle original used a join and would have returned duplicates, so behaviour could differ.
- **Common mistakes:** Believing `$limit` returns the first by `_id`.
- **Follow-up questions:** How would you prove uniqueness? (`$group` by key with count > 1.) Add a `$sort` for determinism? (Cheap with an index.)

### Q11. Why `$addFields` with `$arrayElemAt` rather than `$unwind`?
- **Why the interviewer asks:** LEFT JOIN semantics in Mongo.
- **Ideal answer:** `$unwind` on an empty array drops the document (INNER JOIN behaviour) unless `preserveNullAndEmptyArrays: true`. `$arrayElemAt [arr, 0]` yields missing/null for an empty array, so the row stays with null customer fields. That matches Oracle LEFT JOIN. Other handlers (ASP debit note, schedule) use `$unwind` with `preserveNullAndEmptyArrays: true` for the same effect.
- **Common mistakes:** Plain `$unwind` for optional joins, silently losing rows.
- **Follow-up questions:** What would you use to emulate INNER JOIN? (`$unwind` without preserve, or `$match` on array size.)

### Q12. The `CUST.NAME_EXT` and ID filters are applied after the `$lookup`. What does that cost?
- **Why the interviewer asks:** Performance reasoning about stage ordering.
- **Ideal answer:** They need joined fields, so they cannot move before the lookup. If a request has only name/ID filters and no dates, every `CMUW_PL_REFER_RISK` row is looked up and then filtered. A rewrite would first query `CPGE_VI_PARTNERS_ALL` for matching `PART_ID`s (indexed on name/ID) and then `$match CP_PART_ID $in [...]` before the lookup. That is two queries, but selective.
- **Common mistakes:** Saying the DB optimiser will reorder (Mongo does limited reordering and cannot push a filter before a `$lookup` it depends on).
- **Follow-up questions:** Index needed for the rewrite? (`NAME_EXT`, `ID_VALUE1`, `ID_VALUE2` on the partner collection.)

### Q13. Is the customer name filter case-sensitive, and does it match Oracle?
- **Why the interviewer asks:** Behavioural equivalence.
- **Ideal answer:** Mongo string equality is exact, case-sensitive and without collation here. Oracle `=` is also case-sensitive by default unless NLS settings say otherwise (inferred; the KT deep dive labels it as inference). The handler does no trim or upper-case. So "tan" would not match "TAN" in either system.
- **Common mistakes:** Assuming partial or case-insensitive matching.
- **Follow-up questions:** How would you add case-insensitive search? (Collation on query/index or a normalised field; avoid `$regex` with `i` for index use.)

### Q14. How is Oracle `ROWNUM` after `ORDER BY` reproduced?
- **Why the interviewer asks:** A signature Oracle-to-Mongo translation.
- **Ideal answer:** The JRXML was `SELECT ROWNUM ROWNUMBER, REJECT_RISK.* FROM (SELECT ... ORDER BY NAME_EXT)`. In Mongo: `$setWindowFields` with `sortBy {customerName:1}` and `output.rowNumber = {$documentNumber: {}}`, then `$sort customerName:1`. `$documentNumber` gives 1,2,3... in the window order. Requires MongoDB 5.0+. Other handlers use the same pattern (`CardPrintingHandler`, `MyegStatusHandler`, `ScrutinyRejectedChecklistHandler`).
- **Common mistakes:** Using a constant like the removed `ROW_NUMBER: 1` stage.
- **Follow-up questions:** Is order stable for equal names? (No tie-breaker; Q15.) What if there is no partition? (One window for all rows, in memory.)

### Q15. What happens to row numbers when two customers have the same name?
- **Why the interviewer asks:** Determinism detail.
- **Ideal answer:** `sortBy` only has `customerName`. For ties the order is unspecified, so the numbering among equal names can differ between runs, and the final `$sort` on the same key does not fix that. Oracle's inner `ORDER BY NAME_EXT` has the same ambiguity (inferred). Add a secondary key such as `createdDate` and `_id` to be stable if reproducibility is required.
- **Common mistakes:** Assuming stable insertion order.
- **Follow-up questions:** Does `$sort` in Mongo guarantee stability? (No, not for equal keys.)

### Q16. Oracle sorts NULLs last on ASC, MongoDB sorts them first. Where does this bite RejectRisk?
- **Why the interviewer asks:** Known difference in your module.
- **Ideal answer:** Customers with no partner match have `customerName` null/missing. Oracle `ORDER BY NAME_EXT ASC` puts them at the end; Mongo `$sort {customerName:1}` puts them first, and `$documentNumber` numbers them first. So row numbers and order differ from the original report. Fix options: add a flag field (`$cond` null -> 1 else 0) and sort by `[flag, name]`, or sort in Java with `Comparator.nullsLast` and assign row numbers there (also removes the need for MongoDB 5.0 window fields). It is documented but not fixed in the working copy.
- **Common mistakes:** Saying both databases sort nulls the same way.
- **Follow-up questions:** How to test? (A handler test with a null customer row and an assertion on order.)

### Q17. Does `$setWindowFields` followed by `$sort` do two sorts?
- **Why the interviewer asks:** Performance reasoning.
- **Ideal answer:** Yes in principle: `$setWindowFields` sorts by `customerName` to number rows, then the explicit `$sort` repeats the same sort (the planner may or may not reuse the order; do not assume). Both are blocking stages operating over the whole result in memory. For large listings that is wasteful; sorting once in Java after fetching (or before the window stage only) avoids duplication.
- **Common mistakes:** Believing the second sort is free.
- **Follow-up questions:** Can `allowDiskUse` help? (Yes if set; see Q19.)

### Q18. What is the DB risk of RejectRisk when no date filter is sent?
- **Why the interviewer asks:** Real operational risk.
- **Ideal answer:** With no `$match`, the pipeline scans all of `CMUW_PL_REFER_RISK`, performs up to four `$lookup`s per row, projects, window-numbers and sorts everything in memory, then maps all rows into Java heap (`getMappedResults`) and into XML. No `$limit`, no `maxTime`, no `allowDiskUse`. It is a self-inflicted DoS vector. Mitigation: require a date range or cap span at API level; set `AggregationOptions` with `maxTime`.
- **Common mistakes:** Saying the date is mandatory (it is optional in the JRXML input controls too).
- **Follow-up questions:** How large is the collection? [CONFIRM: row count.]

### Q19. RejectRisk sets no `AggregationOptions`. What are the consequences?
- **Why the interviewer asks:** Knows `allowDiskUse` and `maxTime`.
- **Ideal answer:** Without `allowDiskUse(true)`, blocking stages (`$sort`, `$setWindowFields` on a computed key) are limited to 100 MB of RAM per stage on versions where disk use is not default (in MongoDB 6.0+ the server default can allow disk use; check cluster version [CONFIRM]) and fail with "exceeded memory limit" for very large results. Without `maxTime`, a bad query can run indefinitely, unlike Template12's handlers that set 600 s.
- **Common mistakes:** Stating the 100 MB limit as universal for all versions.
- **Follow-up questions:** How to add? (`Aggregation.withOptions(AggregationOptions.builder().allowDiskUse(true).maxTime(Duration.ofSeconds(..)).build())`.)

### Q20. Where is `$limit` placed in this codebase and is it right?
- **Why the interviewer asks:** Understanding of `$limit` placement and effects.
- **Ideal answer:** `$limit:1` inside `$lookup` sub-pipelines (scalar lookup) and after `$match` in point queries (for example the ASP base pipeline starts at `UWMT_VI_ENDT_TAX_INV_ALT_UPV`, `$match` ENDT_NO + CONTRACT_ID, then `$limit 1`). Both are right. In RejectRisk the final result is intentionally unbounded (listing). A `$limit` placed before `$sort` would cut arbitrary rows; after `$sort` it gives top-N. `$sort` followed by `$limit` is optimised into a top-k sort.
- **Common mistakes:** Placing `$limit` before `$sort` and expecting "first N by name".
- **Follow-up questions:** What would you add to bound RejectRisk? (A cap after `$sort`, plus an API-level signal.)

### Q21. `CONTRACT_ID` is numeric in Mongo. What happens if you match it with a string?
- **Why the interviewer asks:** Type strictness, a top cause of empty reports.
- **Ideal answer:** MongoDB equality is type-aware across type brackets: number vs string never match. `{CONTRACT_ID: "123"}` against a stored number returns nothing and raises no error. Handlers convert request text to numbers (`safeNum`/`asNum`, e.g. `AspEndtDocumentMainHandler` uses `Long.parseLong` with raw-string fallback). If the stored value is itself a string in some replicated collection, the numeric query would miss it (inferred risk).
- **Common mistakes:** Passing request strings straight into `Criteria.is`.
- **Follow-up questions:** How to detect type drift? (`db.coll.find({CONTRACT_ID:{$type:"string"}}).limit(1)`.)

### Q22. `Long` in Java versus Int32/Int64/Double/Decimal128 in Mongo: do comparisons work?
- **Why the interviewer asks:** Numeric type behaviour.
- **Ideal answer:** MongoDB compares numeric types by value across Int32, Int64, Double and Decimal128, so a Java `Long` matches an Int32 stored 123. Precision issues arise when Double or Decimal128 is mixed with large values or when mapping money to Java: use `BigDecimal` DTO fields for decimals (XSD `xs:decimal`). Strings never compare equal to numbers (Q21).
- **Common mistakes:** Believing Int32 vs Int64 mismatch fails.
- **Follow-up questions:** Why `$toDouble` for `COV_SEQ_REF`? (Q26.)

### Q23. Why is `VERSION_NO = 1` hard-coded for the live base query in Template12?
- **Why the interviewer asks:** Business rule hidden in a filter.
- **Ideal answer:** In `PlinePolicyDocumentMainHandler` the base criteria is `CONTRACT_ID is asNum(contractId)` and, when not WIP, `VERSION_NO is 1`. The Oracle JRXML read the original policy version for the base document (inferred); later versions are endorsements shown by ASP. If a policy had no version 1 row (data quality issue), the report would be empty with no error.
- **Common mistakes:** Replacing it with the request `versionNo` without understanding.
- **Follow-up questions:** What does the request `versionNo` default to? ("1".) How to test? (Handler test capturing the pipeline criteria.)

### Q24. How does `safeNum` fail, and what does the fallback do?
- **Why the interviewer asks:** Edge-case handling.
- **Ideal answer:** `Long.parseLong` on text; if not numeric, the code returns the raw string (per the KT note), which then compares against a number field and matches nothing. So bad input gives an empty result, not an exception. That is silent but safe; it does not throw `IllegalArgumentException`.
- **Common mistakes:** Expecting `NumberFormatException` to reach the caller.
- **Follow-up questions:** Should it 400? (Arguably yes for key fields; changing it alters the empty-result contract.)

### Q25. Why are dates sometimes `$dateToString` in a pipeline and sometimes passed as BSON dates?
- **Why the interviewer asks:** Data type decisions at the DB/DTO boundary.
- **Ideal answer:** RejectRisk passes `CREATED_DATE` as a BSON Date to the DTO; the service converts it to `LocalDateTime` for the XSD `xs:dateTime`. Some handlers format in the pipeline (`$dateToString` with `format` and `timezone: UTC`, e.g. `CardPrintingHandler` `%Y-%m-%d`) because the Jasper expression formatted dates in SQL/Java. Formatting in Mongo returns strings, which the XML then carries as text; types must match XSD leaves, which are `xs:dateTime`, not `xs:date`.
- **Common mistakes:** Formatting dates in the DB then treating the string as a date later.
- **Follow-up questions:** Which timezone does `$dateToString` use by default? (UTC unless specified.)

### Q26. Explain the `COV_SEQ_REF` sort with `$toDouble`.
- **Why the interviewer asks:** A concrete type-conversion translation in ASP.
- **Ideal answer:** Oracle: `ORDER BY cast(COV_SEQ_REF as decimal(15,2)) asc`. Sorting the stored text would put "10" before "2". The Java adds `$toDouble` in an `$addFields` and sorts ascending (`AspAddMemberLandscapeInsuredHandler` ~line 59-60; the same in Premium and Nomination handlers). `$toDouble` on a non-numeric string throws, like Oracle's cast would. Null/missing sorts first in Mongo, last in Oracle (inferred difference).
- **Common mistakes:** Sorting as strings. Not knowing `$toDouble` can throw (versus `$convert` with `onError`).
- **Follow-up questions:** Safer alternative? (`$convert` with `onError`/`onNull`.)

### Q27. Explain the `$unionWith` pattern in `AspEndtScheduleHandler`.
- **Why the interviewer asks:** Live + WIP union, a key pattern for your ASP module.
- **Ideal answer:** The main pipeline reads the OCP source (`OCP_...` versions), normalises fields with `$project`, then `$unionWith {coll: WIP_VERSIONS, pipeline: buildWipPipeline(...)}` appends WIP rows (each side filtered by contract/endorsement), then shared lookups (product name, agent name) run once on the merged set, and `$unwind` with `preserveNullAndEmptyArrays` keeps rows without matches. It mirrors an Oracle `UNION ALL` of live and WIP queries.
- **Common mistakes:** Running shared lookups before the union (twice the work).
- **Follow-up questions:** Does `$unionWith` remove duplicates? (No; it is UNION ALL; Q60 in the next file.)

### Q28. Two ways to read live or WIP data: Java switch (`isWip ? WIP_x : x`) versus `$unionWith`. Compare.
- **Why the interviewer asks:** Design trade-off.
- **Ideal answer:** Template12 picks the collection in Java based on `checklistCopy`; one collection per query, simple, index-friendly. ASP schedule uses `$unionWith` because the Oracle query combined both sets; one request returns rows from either source. Use Java switch when the business rule is "either/or", `$unionWith` when it is "both". Union costs two scans and a larger merged set; Java switch costs nothing extra but depends on a correct flag.
- **Common mistakes:** Using union where either/or is the rule (returns stale live rows next to WIP).
- **Follow-up questions:** What happens if `checklistCopy` is wrong? (Reads the wrong source, silently.)

### Q29. In `AutoRenewalReportAutoRenewalPolicyHandler`, what sets the shape of the sort after `$unionWith`?
- **Why the interviewer asks:** Ordering after a union (this handler is outside your three modules but shows the pattern used in the same package).
- **Ideal answer:** After the union it adds `$sort {issueDate:1}` to mirror `ORDER BY ISSUE_DATE` of the SQL, and uses `AggregationOptions` with `allowDiskUse(true)` and `maxTime(Duration.ofMinutes(5))`. A union has no inherent order, so the sort must come after the union stage. This handler is not part of the three modules; I cite it only as a pattern in the same package.
- **Common mistakes:** Sorting inside each branch and expecting global order.
- **Follow-up questions:** Why `maxTime` 5 min here but 10 min in Template12 handlers? (Chosen per handler; no central policy.)

### Q30. How does ASP choose `PRODUCT_CODE` and `ENDT_CODE` across WIP and live data?
- **Why the interviewer asks:** Detailed ASP knowledge.
- **Ideal answer:** `PRODUCT_CODE` = `$ifNull(first of WIP contracts' PRODUCT_CONFIG_CODE, first of SB contracts' PRODUCT_CONFIG_CODE)` via two `$lookup`s, each with `$project` + `$limit:1`. `ENDT_CODE`: the WIP lookup runs only when `$$chk == "Y"`, the live lookup on `UWGE_POLICY_VERSIONS` (by `VERSION_NO`) only when `"N"`; the `$expr` with `$eq ["$$chk","N"]` makes the other branch return empty. The pipeline starts at `UWMT_VI_ENDT_TAX_INV_ALT_UPV` (ANC) matched on ENDT_NO + CONTRACT_ID with `$limit 1`.
- **Common mistakes:** Thinking `$ifNull` handles empty arrays (it checks null/missing; first-element extraction produces missing).
- **Follow-up questions:** Why `$ifNull` over `$cond`? (Concise "first non-null".)

### Q31. Where is `$group` used in the Template12 and ASP code, and for what?
- **Why the interviewer asks:** Aggregation breadth with project-specific grounding.
- **Ideal answer:** Template12 handlers `PlineT12PolicyDocumentHandler` (lines ~213, 222) and `Template12LandscapeDispatchHandler` (lines ~200, 208) use `$group {_id: null, ...}` to compute totals (sums) in one document, mirroring Oracle `SUM(...)` without `GROUP BY`. Shared coverage handlers (`PlineCoverageHandler`) use `$group` to de-duplicate rows (`_id` as a set of columns) so Jasper-like DISTINCT behaviour holds. I did not list every use [CONFIRM which ones you personally touched].
- **Common mistakes:** Using `$group` for ordering. Forgetting that `$group` output order is undefined.
- **Follow-up questions:** What does `$group _id:null` return on zero input rows? (No document; Q33.)

### Q32. How do you use `$group` to emulate Oracle `DISTINCT`?
- **Why the interviewer asks:** Translation skill.
- **Ideal answer:** `$group {_id: {colA:"$colA", colB:"$colB", ...}}` then project from `_id`, or `$group {_id:"$$ROOT"}` to dedupe whole documents (used in `AutoPolicyHandler` ~line 270). Order is not preserved, so add `$sort` afterward. For big sets `$group` on `$$ROOT` is memory heavy (100 MB limit per stage unless disk use allowed).
- **Common mistakes:** Assuming `$group` keeps input order.
- **Follow-up questions:** Alternative? (`$setWindowFields` with `$rank` or distinct-by-key after a sort.)

### Q33. Oracle `SUM` over zero rows returns NULL; Mongo `$group` over zero rows returns nothing. What breaks?
- **Why the interviewer asks:** Edge-case semantics in totals.
- **Ideal answer:** Oracle returns one row with NULL; Mongo `$group {_id:null}` emits no document when the input is empty. Java code that does `results.get(0)` without a size check would fail with IndexOutOfBounds, or a total that should print blank/0 is missing. The handlers must check for empty results and default to `BigDecimal.ZERO` or null as the JRXML expression did (verify per handler).
- **Common mistakes:** Assuming a total of 0 is returned.
- **Follow-up questions:** How to force one row? (Add a `$unionWith` of a dummy or handle in Java.)

### Q34. How does rounding in `$round` compare with Oracle `ROUND`/`TRUNC`?
- **Why the interviewer asks:** Money correctness.
- **Ideal answer:** Oracle `ROUND` rounds half away from zero. MongoDB `$round` rounds half to even (banker's rounding) per documentation, so 2.5 and 0.125 cases can differ (the ASP deep dive lists this as a documented behaviour not tested in the repo). For premium/tax values on a legal document, rounding must be done where the Jasper original did it, usually Java `BigDecimal` with `RoundingMode.HALF_UP`. Do not rely on `$round` for money.
- **Common mistakes:** Rounding doubles.
- **Follow-up questions:** Which DTO types carry money? (`BigDecimal` -> `xs:decimal`.)

### Q35. Template12 uses many small sequential queries ("Step A..G") instead of one big pipeline. Why, and what is the cost?
- **Why the interviewer asks:** Architecture trade-off in your module.
- **Ideal answer:** Each step is a `$match` -> `$project` -> `$limit 1` point query with `AGG_OPTIONS` (`maxTime` 600 s, `allowDiskUse`). It mirrors nested Oracle scalar sub-selects, is easy to debug and each step can use a simple index. Costs: N round trips per request (latency, driver pool use), no atomic snapshot between steps (data can change), and logic scattered across Java.
- **Common mistakes:** Calling it "inefficient" without noting debuggability, or "efficient" without counting round trips.
- **Follow-up questions:** When would you merge into one pipeline? (When steps are strictly dependent and each is small, using `$lookup`.)

### Q36. Describe the master-policy chain in the ASP debit note.
- **Why the interviewer asks:** A multi-step DB logic from your module.
- **Ideal answer:** `UWGE_POLICY_BASES` + `WIP_UWGE_POLICY_BASES` give `MST_POL_NO` values; those go into `OCP_POLICY_BASES` with `POLICY_REF $in [...]` to find the master `CONTRACT_ID`; then `PREM_BY_MST_IND` is read. 4-5 round trips mirror nested Oracle scalar sub-queries. The class javadoc says the real Oracle query had no `CONTRACT_ID` filter in places, so index-friendly access is not guaranteed.
- **Common mistakes:** Skipping the WIP leg.
- **Follow-up questions:** Index for `$in`? (`{POLICY_REF:1, VERSION_NO:1}` on `OCP_POLICY_BASES`, recommended.)

### Q37. `AspDebitNoteHandler` uses `$or` on `ENDT_NO` and `REF_NO`. How does that affect indexes?
- **Why the interviewer asks:** Query planner knowledge.
- **Ideal answer:** A top-level `$or` can use indexes only if every branch has a usable index, then the planner runs an OR of index scans. If one branch lacks an index, the whole `$or` falls back to a collection scan. So `UWGE_VI_ENDT_DEBIT` etc. need an index on `ENDT_NO` and another on `REF_NO` (or compound including each). The index set I list is a recommendation; none are defined in the repo.
- **Common mistakes:** One compound index `{ENDT_NO, REF_NO}` expecting it to serve both branches (it serves only the ENDT_NO prefix branch).
- **Follow-up questions:** How to confirm? (`explain` showing `OR` stage with two `IXSCAN`s.)

### Q38. Which indexes would you propose for RejectRisk collections, and in what field order?
- **Why the interviewer asks:** Index design with ESR rule.
- **Ideal answer:** `CMUW_PL_REFER_RISK`: `{CREATED_DATE:1}` (range, main selective filter). If `REJ_TYPE`/`REASON_CODE` filters are common: equality fields first, then the range: `{REJ_TYPE:1, REASON_CODE:1, CREATED_DATE:1}` (ESR: equality, sort, range). `CPGE_VI_PARTNERS_ALL`: `{PART_ID:1, VERSION:1}`. `CMGE_CODE`: `{CAT_CODE:1, CODE_CD:1}` (used by nearly every handler, highest value). The repo has no index definitions, so these are suggestions to verify with `explain` [CONFIRM: existing Atlas indexes].
- **Common mistakes:** Putting the range field first in a compound index.
- **Follow-up questions:** Does one compound index serve the case where `REJ_TYPE` is absent? (No; prefix rule.)

### Q39. Why is `CMGE_CODE {CAT_CODE, CODE_CD}` the highest-value index?
- **Why the interviewer asks:** Impact-based prioritisation.
- **Ideal answer:** Nearly every handler looks up descriptions in `CMGE_CODE` by category and code; RejectRisk does it three times per row, ASP and Template12 do it for product, state, relationship, etc. A missing index makes each lookup a scan of a code table, multiplied by rows. A small table scans fast, but a compound index makes it near constant time. It is one index shared by all LOBs.
- **Common mistakes:** Indexing only by `CODE_CD`.
- **Follow-up questions:** Is the pair unique? (Expected by `$limit:1`; verify.)

### Q40. How do you know whether a `$lookup` uses an index?
- **Why the interviewer asks:** Tooling skill.
- **Ideal answer:** Run `db.<coll>.explain("executionStats").aggregate([...])` with the pipeline captured from `MongoTemplate` DEBUG logs. Look for `$lookup` stage details (`indexesUsed`, `totalDocsExamined`, `COLLSCAN` vs `IXSCAN`) and `nReturned`. Equality in `let/$expr` form can use an index on the foreign field in recent server versions; verify per version rather than assume.
- **Common mistakes:** Judging by timing on a tiny dev dataset.
- **Follow-up questions:** What if the cluster is Atlas? (Performance Advisor, profiler.)

### Q41. `maxTime(600 s)` is set in many Template12/ASP handlers. Is that sensible?
- **Why the interviewer asks:** Timeout alignment.
- **Ideal answer:** `AGG_OPTIONS` (`maxTime` 600 s, `allowDiskUse true`) is a safety net, not a performance goal. The caller side is likely much shorter (unknown), so a 10-minute server limit lets orphan queries continue after the HTTP caller has given up. I would set `maxTime` slightly below the caller's timeout per handler and fail fast with an `IllegalArgumentException`/custom error instead of hanging a thread.
- **Common mistakes:** Treating 600 s as "the timeout of the API".
- **Follow-up questions:** What exception does the driver throw? (A MongoExecutionTimeout-type exception wrapped by Spring; caught by the 500 handler.)

### Q42. Which of your modules' handlers set `AggregationOptions` and which do not?
- **Why the interviewer asks:** Detail knowledge.
- **Ideal answer:** Set: `PlinePolicyDocumentMainHandler` (static `AGG_OPTIONS`), `AspDebitNoteHandler`, and several Asp/Acp handlers (end schedule, cover, installment details, complaints with `allowDiskUse` only). Not set: `RejectRiskHandler` and `AspEndtDocumentMainHandler`. This inconsistency is a risk for the unbounded RejectRisk pipeline.
- **Common mistakes:** Assuming a global default.
- **Follow-up questions:** How to make it consistent? (A shared helper returning default options.)

### Q43. Why is DEBUG logging of `MongoTemplate` a problem for the database layer?
- **Why the interviewer asks:** Observability versus cost.
- **Ideal answer:** It logs each operation including query documents and parameter values: slower and noisy, and values include contract ids, names and ID numbers. Useful to capture a pipeline for `explain`, harmful as a prod default. Prefer targeted debug via a toggle. Keys `...MongoTemplate=DEBUG` and `...aggregation.Aggregation=DEBUG` are active in `application.properties`.
- **Common mistakes:** Leaving DEBUG on because "it helps support".
- **Follow-up questions:** How to capture pipelines safely in prod? (Mongo profiler or Atlas query insights.)

### Q44. A report comes back empty with no error. List DB-side causes.
- **Why the interviewer asks:** Troubleshooting checklist; real incidents are mostly this.
- **Ideal answer:** (1) Wrong DB name (`spring.mongodb.database`) or URI fell back to localhost due to old `spring.data.mongodb.*` prefix. (2) Type mismatch (string vs number). (3) WIP vs live confusion (`checklistCopy`). (4) Filters too narrow or case-sensitive text. (5) Date boundary or time zone. (6) A lookup joined on the wrong version (`VERSION_NO`). (7) Data not yet replicated from Oracle (lag) [CONFIRM]. Mongo never raises an error for a zero-row match.
- **Common mistakes:** Suspecting the XML layer first.
- **Follow-up questions:** Fastest first check? (Run the first `$match` in mongosh against the same DB.)

### Q45. How does a DTO get filled from an aggregation?
- **Why the interviewer asks:** Mapping mechanics.
- **Ideal answer:** `mongoTemplate.aggregate(agg, COLLECTION, Dto.class).getMappedResults()` maps result documents to the DTO by field name. The `$project` stage creates aliases that equal DTO field names (`customerName`, `rejectRiskType`, `rowNumber`...). A name mismatch gives null silently (no error). Types convert per Spring Data rules (BSON Date to `Date`/`LocalDateTime` depending on the DTO; `PlineRejectRiskDto` holds strings/dates as designed, see 01 deep dive).
- **Common mistakes:** Expecting an error on unmapped fields.
- **Follow-up questions:** How to catch mismatches? (Unit test with a real `Document` -> DTO conversion, not only mocks.)

### Q46. `$project` stage uses `updatedBy` and `updatedDate` in the DTO while the XSD leaves are `updatedby` and `updateddate`. Where is the translation?
- **Why the interviewer asks:** Naming layers across DB, DTO and XSD.
- **Ideal answer:** Three naming layers: Mongo field `UPDATED_BY`/`UPDATED_DATE`; DTO alias `updatedBy`/`updatedDate` (from `$project`); XSD leaf `updatedby`/`updateddate` set by `transDet.setUpdatedby(...)`/`setUpdateddate(...)` in `RejectRiskService`. The mapping is manual in the service. Same for `reasoncode`. Mistakes show as empty elements, not errors.
- **Common mistakes:** Using the DB field name in the XSD.
- **Follow-up questions:** Where is the mapping documented? (`RejectRisk_XSD_Mapping_sheet.xlsx`.)

### Q47. Missing field versus null versus empty string: how do they behave in matches?
- **Why the interviewer asks:** Data-quality reasoning for replicated Oracle data.
- **Ideal answer:** `{f: null}` matches both null and missing. `{f: {$exists:false}}` matches only missing. `{f: {$ne: "x"}}` also matches documents where `f` is missing (this is why an Oracle `<>` translation can return more rows than Oracle: Oracle excludes NULL). An empty string is a distinct value. Oracle treats `''` as NULL, so replicated data may hold null in Oracle and `""` or null in Mongo depending on the loader (inferred) [CONFIRM: loader behaviour].
- **Common mistakes:** Translating `col <> 'X'` to `$ne` without adding an exists check.
- **Follow-up questions:** How to write Oracle-faithful `<>`? (`$and: [{f:{$exists:true}}, {f:{$ne:null}}, {f:{$ne:"X"}}]`.)

### Q48. Does `$expr` with `$eq` in a first-stage `$match` use an index?
- **Why the interviewer asks:** Planner subtleties.
- **Ideal answer:** In current MongoDB versions `$expr` with simple equality on a field and a constant can use an index, but range operators in `$expr` and comparisons between two fields of the same document cannot use indexes the same way. Prefer plain `{field: value}` or `{field: {$gte, $lt}}` in the leading `$match` (as RejectRisk does for `CREATED_DATE`). Inside `$lookup` pipelines `$expr` is needed for `let` variables; verify with `explain`.
- **Common mistakes:** Using `$expr` for plain filters.
- **Follow-up questions:** Where is a field-to-field comparison used? (`$lte: ["$VERSION_NO","$$pv"]` in lookups, e.g. annual premium by version.)

### Q49. What risks come from the unbounded `getMappedResults()`?
- **Why the interviewer asks:** Memory behaviour.
- **Ideal answer:** The driver cursor is fully drained into a Java `List`, then DTOs are converted into XML objects, then serialised: three copies of the data in heap. For RejectRisk with a large date range this can cause GC pressure or OOM on a small pod. Mitigation: cap, stream per batch (not supported by the one-document response), or reduce columns in `$project`. Memory sizing of pods is unknown [CONFIRM: Helm resources].
- **Common mistakes:** Believing Mongo cursor batching protects heap.
- **Follow-up questions:** How to measure? (Heap dump or GC logs under a large request.)

### Q50. If the DB name in the URI is wrong, what does the service do?
- **Why the interviewer asks:** Config meets DB.
- **Ideal answer:** Connection to the cluster succeeds, queries run against a database with no such collections, and every aggregation returns zero rows silently (Mongo does not error on a missing collection). The working copy sets `spring.mongodb.database` to a production-clone-looking name (`OPUSAGICPRODR1`, inferred from the string). The committed history removed the UAT URI lines in `7dd2f2f67`, so which DB a deployed pod uses is decided outside this repo.
- **Common mistakes:** Expecting `MongoCommandException` or a startup failure.
- **Follow-up questions:** How to detect at startup? (A startup check that counts documents in a known collection and logs the DB name, not the URI.)
