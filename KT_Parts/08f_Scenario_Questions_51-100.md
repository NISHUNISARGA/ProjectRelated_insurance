# 08f - Scenario Questions 51-100 (XSD, platform, operations, process, interview situations)
> **AUTHORSHIP UPDATE (confirmed by the developer): git author `GS <extern.gs_nisarga@allianz.com>` is the developer. Wherever this file says commits cannot be attributed to the developer, read it as: the 11 GS commits listed in `KT_Parts/14_My_Commits_And_Ownership.md` ARE theirs; commits by other authors still must not be claimed. No GS commit touches `src/test`, so test authorship is still `[CONFIRM]`.**

"What would you do if ..." questions for THIS system. Facts come from the KT guide and KT_Parts 01-06; "(inferred)" = reasoning, not run; "[CONFIRM: ...]" = personal fact. Answers are action plans.

Order: Q51-54 XSD/contract, Q55-57 git/CI, Q58-70 operations and security, Q71-77 change and parity, Q78-85 data and performance, Q86-96 team/process, Q97-100 interview situations.

---

### Q51. The schema owner sends you a newer Pline.xsd (say v0.1.10). You validate the real output and get hundreds of errors. What do you do?
- **Why the interviewer asks:** Structured triage of a large failure.
- **Ideal answer:** Do not fix errors one by one. Collect all errors with an `ErrorHandler`, group by message type: (a) empty decimal/dateTime/integer/boolean elements (biggest volume; null with `Include.ALWAYS`), (b) `dd/MM/yyyy` vs `xs:dateTime`, (c) sibling order (`transaction`, templateDetails, `address`-first types, cover, risk, transDet), (d) unknown elements (`mainReportName`, `pReportTitle`). Check what the newer XSD already fixes (for example `mainReportName`). Fix by group in order of volume, starting with a service-level test (RejectRisk first) using a ratchet baseline that only goes down. Report remaining items to the schema owner.
- **Common mistakes:** Editing the XSD to match the output without a decision.
- **Follow-up questions:** Which fix has the largest blast radius? (Changing the shared `Transaction` order affects Motor/Fire/Bond.)

### Q52. The print engine rejects the XML because of empty numeric elements. What do you do?
- **Why the interviewer asks:** Practical serialisation fix.
- **Ideal answer:** Empty `<x/>` is only valid for `xs:string`. Short fix: use `@JsonInclude(NON_NULL)` (or `NON_EMPTY`) on the Plines payload classes so nulls are omitted (all elements are optional, so omission is legal). Check the renderer first: if it expects every element present, agree on a rule. Because `PlineHeaderTransaction` and `PlineReportTransaction` default to one empty object, remove those placeholders too. Verify with a validation test and compare XML size before and after. Make the change per class, not on the shared Motor base.
- **Common mistakes:** Replacing empties with `0`, which changes meaning.
- **Follow-up questions:** Other option? (`xsi:nil` with `nillable` in the XSD.)

### Q53. You must send ISO dates to satisfy the XSD, but Jasper reads `dd/MM/yyyy`. What is your plan?
- **Why the interviewer asks:** Cross-team compatibility.
- **Ideal answer:** List every date leaf Java populates (64 `dateTime` leaves in Pline.xsd; examples `createdDate`, `revokeDate`, `updateddate`, `pBeforeEffDate`, `dateValue`, `pmtDueDate`). Ask the report team how each is used in expressions. Options: (a) ISO on the wire and the report formats it (needs report change), (b) XSD retyped to string/pattern (needs schema change; loses typing), (c) both: ISO leaf plus a formatted string leaf (schema grows). Pick one with both teams, apply by class, support both in a transition (feature flag), and write a validation test. Do not change the format alone.
- **Common mistakes:** Changing the `@JsonFormat` and shipping.
- **Follow-up questions:** The reflective date pass in S12? (It rewrites strings to `dd/MM/yyyy`; remove or update it together.)

### Q54. Fixing sibling order needs a change in the shared `Transaction` class. What do you do?
- **Why the interviewer asks:** Blast-radius thinking.
- **Ideal answer:** Java writes templateDetails, mailingDetails, transactionDetails, reportParameters; the XSD wants reportParameters, mailingDetails, templateDetails, transactionDetails. The class is shared with Motor/Fire/Bond and their XSDs may differ (the Motor XSD order is what the code comment calls "XSD order"). Do not edit it in place. Options: a Plines-specific subclass or wrapper with `@JsonPropertyOrder`; or an order-fixing serialiser used only for Plines. Test with existing Motor tests plus new Plines validation. Check how other LOBs depend on the order (name-based consumers may not care).
- **Common mistakes:** Adding `@JsonPropertyOrder` to the shared class and breaking other LOB validations.
- **Follow-up questions:** Same pattern elsewhere? (`PlineTransactionDetails`, `PlinePrntDetails` already have it.)

### Q55. The Sonar quality gate fails on your PR. What do you do?
- **Why the interviewer asks:** Daily CI behaviour.
- **Ideal answer:** Open the Sonar report for the PR; separate new-code issues from legacy. Fix real issues (narrow catches, unused code, duplicates, complexity) rather than suppress. If the gate is coverage on new code, add tests that check behaviour (gates, mapping), not Lombok getters. Re-run `mvn -B verify` locally. If a rule is a false positive (like CSRF S4502 for a stateless API), document it, as `SecurityConfig` does. Do not mix test additions with big production edits in one commit (the `3f2157b28` problem).
- **Common mistakes:** `@SuppressWarnings` or `@Generated` hacks.
- **Follow-up questions:** Who sets the thresholds? (The Sonar server, not the repo.)

### Q56. Your feature branch conflicts with `retrofit-develop-R2` in `Template12PolicyDocumentService`. What do you do?
- **Why the interviewer asks:** Real merge handling on a huge file.
- **Ideal answer:** Pull R2 into the branch (the team pattern is "Merge branch 'retrofit-develop-R2' into <feature>" before PR). Resolve by intent, not by taking one side: look at `git log -p` for both sides, especially merge helpers (`mergeSingleTransDet`) and the `if` blocks per product. After resolving, build, run the Template12 service tests (34 + 135 base tests) and handler tests, and diff the XML for one sample contract before and after if possible. Commit the merge separately with a clear message and a Jira key (`OPUSM-`). Ask the other author when unsure.
- **Common mistakes:** Resolving with "accept mine" in a 2,235-line class.
- **Follow-up questions:** How to avoid? (Smaller classes; frequent syncs.)

### Q57. You must bring `cc6a1b84d` (on the OPUSM-41763 branch) into your branch. How?
- **Why the interviewer asks:** Branch strategy knowledge.
- **Ideal answer:** The commit touches RejectRisk plus 13 other files (ASP Endt* handlers, a new `AspCompanyNameWithGSTRegNoHandler`, `AspEndtScheduleHandler` +333 lines). Options: `git cherry-pick cc6a1b84d` (or `-n` to review first) and resolve conflicts, since the other branch is 194 commits ahead and 1,363 behind; or cherry-pick only the files you need with `git checkout <commit> -- <path>`. Check that the RejectRisk tests do not exist on that branch (367 test files vs 1,932). After picking, update tests, run the suite, and keep the Jira key `OPUSM-41763` in the message. Tell the original author and the reviewers.
- **Common mistakes:** Copying code by hand and losing the history link.
- **Follow-up questions:** How do you check content equality? (`git diff <branch> -- file`.)

### Q58. After an external API outage, users report reports with blank fields but no errors. What do you do?
- **Why the interviewer asks:** Silent-failure incident handling.
- **Ideal answer:** Cause: `PlineUtilsClient` and `Pline*Client` swallow failures and return null/"" (for example e-invoice "N"). Steps: confirm the outage window from logs (search "PlineUtilsClient GET failed" and `log.warn` lines); list affected requests (contracts, endorsements) from logs or the caller; ask the caller to reprint after recovery; check which gates depend on the failed API (e-invoice, service-tax indicator, QR, company BRN, signature, SST rollout date). Later: add metrics/alerts on failures, a degraded flag in the response, retries for GETs, and fail fast for gate-controlling calls. Write a short incident note.
- **Common mistakes:** Assuming HTTP 200 means correct output.
- **Follow-up questions:** Why is it dangerous for invoices? (Legal documents can be missing pages.)

### Q59. After deployment the app connects to `localhost:27017`. What do you check?
- **Why the interviewer asks:** Boot 4 trap.
- **Ideal answer:** Likely old keys: `spring.data.mongodb.*` are ignored in Boot 4 (the prefix is `spring.mongodb.*`), so the default URI is used. Check the effective config (env vars, Helm values, mounted properties), look for `SPRING_DATA_MONGODB_URI` vs `SPRING_MONGODB_URI`. Fix the key names. Verify with a startup log or a simple health query. Remember the committed `application.properties` has no Mongo URI after `7dd2f2f67`; deployment must supply it (Helm repo not seen).
- **Common mistakes:** Debugging network policy first.
- **Follow-up questions:** How to prevent? (A startup check that logs the Mongo host without secrets.)

### Q60. After a dependency bump, every POST fails with `HttpMessageConversionException`. What do you do?
- **Why the interviewer asks:** Jackson 2 vs 3 knowledge.
- **Ideal answer:** Check that `spring.http.converters.preferred-json-mapper=jackson2` is still in `application.properties`; without it Boot 4.1.1 uses Jackson 3, which ignores `com.fasterxml.jackson` annotations used by `PayLoadRequest` and the payload classes. Also check the Jackson pin property (`jackson-2-bom.version`, not `jackson-bom.version`) and `mvn dependency:tree` for mixed versions. Run one controller test. Roll back the bump if needed and open a task for a planned Jackson 3 migration.
- **Common mistakes:** Removing the property thinking it is a leftover.
- **Follow-up questions:** Where else is Jackson 2 required? (XML output via `XmlMapper`.)

### Q61. Wiz reports a new critical CVE in a library. What is your process?
- **Why the interviewer asks:** Vulnerability handling.
- **Ideal answer:** Identify the artifact and the fixed version; check `mvn dependency:tree` for the path (BOM-managed or explicit). If the Boot BOM already has a fixed version, bump Boot or remove an old pin. Otherwise pin with a property and a comment with the CVE id (the repo style: Tomcat 11.0.25, Jackson 2.21.5, log4j 2.25.5, commons-lang3 3.18.0). Run the tests and one sample report. Check that the pin is not lower than the BOM (commons-lang3 3.18.0 vs BOM 3.20.0 is a downgrade - review). Record the ticket and remove the pin when the BOM catches up.
- **Common mistakes:** Pinning an older version than the BOM.
- **Follow-up questions:** When Boot itself is the cause? (Upgrade, like 3.5 to 4.1.1 for CVE-2026-59270.)

### Q62. After upgrading springdoc, the app fails at startup with `NoClassDefFoundError`. What do you do?
- **Why the interviewer asks:** Compatibility debugging.
- **Ideal answer:** This happened with springdoc 2.8.15 on Boot 4 (a relocated `WebMvcProperties`). Use the springdoc line that supports Boot 4 (3.1.1 here). Confirm the compatibility matrix, run `dependency:tree`, and check the Swagger UI under `/gi/documentprintdata/swagger-ui/index.html`. Add a smoke `@SpringBootTest` or at least a startup check in CI. Consider disabling springdoc in production.
- **Common mistakes:** Adding the missing class manually.
- **Follow-up questions:** Another upgrade conflict? (Micrometer 1.15.12 pin broke `ApplicationContext` tests.)

### Q63. A caller complains that an unknown report name returns SUCCESS with an empty result. What do you do?
- **Why the interviewer asks:** API design improvement.
- **Ideal answer:** Agree it is a design weakness: HTTP 200, `status: SUCCESS`, default transaction. Plan: in `CommonController`, check that the name has a strategy bean or a legacy case; otherwise throw `IllegalArgumentException` (400) or a not-found exception (404, the handler exists) with a clear message; log it at WARN with the name. Check callers first: some may rely on the lenient behaviour, so announce the change and roll it out with a flag. Add tests: known name, unknown name, null name (currently NPE -> 500, inferred), merged-away names.
- **Common mistakes:** Changing it without telling callers.
- **Follow-up questions:** What about `/getJSON`? (Returns an empty DTO; legacy.)

### Q64. A request without `metadata` returns 500. How do you handle it?
- **Why the interviewer asks:** Input validation.
- **Ideal answer:** `populateMailingDetails` dereferences `getMetadata()` without a null check (NPE, inferred from code). Fix in the controller: validate that `metadata` and `filterCondition` exist and that `filterCondition.reportName` is present; return 400 with a message. Add a controller test. Consider Bean Validation (`@Valid`, `@NotNull`) since `spring-boot-starter-validation` is already a dependency. Check the catch-all handler also logs.
- **Common mistakes:** Making `metadata` optional silently and producing an incomplete `templateDetails`.
- **Follow-up questions:** Which metadata keys are used? (`name`, `email`, `senderMail`, `copyType`, `batchConfigId`, ..., `reportName`.)

### Q65. The Mongo password must be rotated in all environments. What do you do for this service?
- **Why the interviewer asks:** Operations with secrets.
- **Ideal answer:** The URI comes from outside the repo in deployed environments (Helm/secret; not visible here). Steps: coordinate with the DB owner for the new credential; update the secret store/Helm values per environment; roll pods one by one; watch `health` and a sample report call; remove the old credential. Check nothing else uses it, including developers' local env vars. Make sure the old one is not in the tracked files or history (`gitleaks.toml` allowlist hides findings in `application*.properties`). Keep a rollback value for a short window only.
- **Common mistakes:** Updating the properties file in the repo.
- **Follow-up questions:** How to avoid downtime? (Rolling update with both credentials valid for a while.)

### Q66. The security team says "this API must authenticate callers". How do you approach it?
- **Why the interviewer asks:** Security change plan.
- **Ideal answer:** Facts: `anyRequest().permitAll()`, no key, no JWT, open Swagger, request body printed to stdout. Plan: (1) learn who the callers are (OPUS print orchestrator; not visible in the repo) and what the gateway already does; (2) choose a mechanism (gateway-validated JWT with a resource-server config, or a shared secret header); (3) implement with a `OncePerRequestFilter` or Spring Security resource server, keep `/actuator/health` open; (4) secrets from env/secret store; (5) restrict Swagger; (6) NetworkPolicy; (7) rate limits; (8) test 401/403; (9) release with callers ready. Estimate with the caller team.
- **Common mistakes:** Breaking the callers with no coordination.
- **Follow-up questions:** Interim step? (Network-level restriction and removing public docs.)

### Q67. Compliance finds personal data (names, IDs) in application logs. What do you do?
- **Why the interviewer asks:** PDPA awareness with code evidence.
- **Ideal answer:** Sources: `CustomRequestFilter` `System.out.println` of full bodies (`pCustomerName`, `pIdNo`, contract/policy numbers); `MongoTemplate`/`Aggregation` DEBUG logs with values; the legacy path logs the response XML at INFO. Actions: remove the prints; set those loggers to INFO/WARN in non-dev; mask identifiers; keep correlation id only; ask ops to purge or restrict log storage; add a check (grep) in CI for `System.out.println`. Record the change in a ticket and inform the data protection contact.
- **Common mistakes:** Only lowering the level and leaving the prints.
- **Follow-up questions:** How to debug without values? (Log counts, ids hashed, and pipeline shape.)

### Q68. In a new environment all reports return empty but with SUCCESS. What do you check first?
- **Why the interviewer asks:** Mongo "silent empty" diagnosis.
- **Ideal answer:** Mongo never errors on a zero-row match. Check: `spring.mongodb.database` (wrong DB name gives empty collections), the URI key prefix (localhost fallback would fail to connect, so not this), data present for the contract in that DB, `CONTRACT_ID` type, WIP vs live prefix and `checklistCopy`, collection names (view vs collection), report name resolution (a missing bean gives an empty default transaction), and the request values. Test with one known contract using Compass against the same DB.
- **Common mistakes:** Blaming the code first.
- **Follow-up questions:** Which settings differ per environment? (See Q69.)

### Q69. You must promote the service from dev2 to UAT. Which configuration items do you change?
- **Why the interviewer asks:** Environment configuration awareness.
- **Ideal answer:** At least: `API_BASE-URL`, `plines.api.base-url`, `workmen.api.base-url`, `spring.mongodb.uri`, `spring.mongodb.database`, and the truststore if the CA differs. There is no single environment switch; `spring.profiles.active=dev` activates nothing; dev1 host keys are unread. Use env var overrides or a mounted `application.properties` rather than editing the jar's file. Verify with a startup log of the base URLs (not secrets) and a smoke call per module. Update the Helm values (other repo). [CONFIRM: how your team actually deploys.]
- **Common mistakes:** Changing only `API_BASE-URL` so `Pline*Client` calls still go to dev2.
- **Follow-up questions:** How to unify? (One base-URL property.)

### Q70. The review-by date 2026-10-23 for the gitleaks allowlist is near. What do you do?
- **Why the interviewer asks:** Security debt management.
- **Ideal answer:** The allowlist exempts the two whole files `application.properties` and `application-env.properties` (OPUSM-50404), hiding any new secret in them (the truststore password and past Mongo URIs). Plan: remove secrets from those files (externalise to env/secret store), then delete the allowlist or narrow it to specific rule ids and false-positive lines; run the full-history scan (nightly job exists); rotate credentials that appeared in history; update the ticket. Do it before the date or request an extension with a written reason.
- **Common mistakes:** Extending the date without a plan.
- **Follow-up questions:** Why is a whole-file allowlist weak? (Any future secret there passes.)

### Q71. Business asks to add a new product with its own CIPrint layout to Template12. How do you plan it?
- **Why the interviewer asks:** Change management.
- **Ideal answer:** Get the new Jasper variant and its print conditions (`printWhenExpression`) and nested sub-reports; map each to Mongo (collections, joins, WIP vs posted); decide product-level vs scheme-level branch in `S12.execute` and `addCIPrintDefaultNestedSubreports`; check Landscape bands and PD2 gates for the new code; update Elite exclusion if scheme-based; add handlers/DTOs/payloads; XSD leaves via the schema owner; tests (gate, mapping, handler pipeline, plus an XSD validation case); a compare with a Jasper PDF for two contracts. Keep PRs small (handlers first, then wiring). Estimate after reading the jrxml.
- **Common mistakes:** Copying the closest variant and forgetting gates.
- **Follow-up questions:** Where could it silently fall to the default? (Product lists with `!=` conditions.)

### Q72. Your manager asks for an estimate to prove parity for all products in Template12. What do you say?
- **Why the interviewer asks:** Estimation under uncertainty.
- **Ideal answer:** I would not quote a number blindly. Break it down: product/scheme matrix (070504 with 7+ schemes, 070412 with cutoff Y/N, 070406, 070414, 100908, default), three reports (PD, PD2, Landscape), flags (`showDebitNote`, `showTaxInvoice`, `checklistCopy`, `copyType`). For each cell: choose a contract, produce Jasper PDF and Java XML, compare sub-reports present, row counts, key amounts. Needs test data access, the Jasper environment, and the renderer team. Give a range after a pilot of 2-3 cells, list assumptions and risks, and state what is out of scope (page parity). [CONFIRM: your team's estimation practice.]
- **Common mistakes:** Quoting days with no basis.
- **Follow-up questions:** Can parts be automated? (Yes, with a gate-matrix test and golden files.)

### Q73. The report team says your output is wrong, the data team says the Mongo data is right. How do you settle it?
- **Why the interviewer asks:** Cross-team conflict.
- **Ideal answer:** Use evidence, not opinion. Take one contract. Show: the Oracle/Jasper result (SQL output and PDF), the Mongo data, the pipeline output, and the XML field. Locate the first step where they differ (data copy, pipeline logic, mapping, serialisation, rendering). Common root causes in this system: type mismatch, null ordering, WIP vs posted, swallowed external API, `reportParameters` whitelist, date format. Document the finding with a table and agree on the owner for the fix.
- **Common mistakes:** Arguing from assumptions.
- **Follow-up questions:** What artefact do you keep? (A comparison note for the contract with the steps.)

### Q74. You discover the deployed compiled report differs from the Jasper dump you analysed. What do you do?
- **Why the interviewer asks:** Source-of-truth risk.
- **Ideal answer:** For PD2 only the readable jrxml was treated as authoritative; the compiled `.jasper` was string-searched. If the deployed report differs, deviations D2-D4 (dead bands, extra bands) may change. Steps: get the deployed jrxml from the report server, diff print conditions and sub-report lists with the dump, update the traceability tables and Java gates, add tests for changed gates, and notify the reviewers. Record the version/date of each source you used.
- **Common mistakes:** Silently updating only the code.
- **Follow-up questions:** How to avoid? (Pin the dump version in the analysis note.)

### Q75. You find a bug in a colleague's handler outside your modules. What do you do?
- **Why the interviewer asks:** Teamwork and scope.
- **Ideal answer:** Verify it first with a minimal reproduction (a test or a Compass query). Write a clear ticket with evidence (file, line, input, expected, actual) and tag the owner. If it blocks you, propose a small fix in a separate PR with its own Jira key and tests, and ask the owner to review. Do not change shared code (for example a shared base handler used by ~24 templates) inside your feature PR. Example type: the shared base `buildCommonPipeline` anchor issue, which ASP overrides locally.
- **Common mistakes:** Silent fix mixed into your feature.
- **Follow-up questions:** If the owner is unavailable? (Ask the lead; use an ASP-only override like the CONTRACT_ID fix.)

### Q76. You are pressured to raise coverage quickly for the Sonar gate. What do you do?
- **Why the interviewer asks:** Integrity vs metrics.
- **Ideal answer:** Say what coverage proves and what it does not. Prioritise tests that protect behaviour: gates (`showServiceTax_checklistCopyY_closesGate`), mapping, date bounds, null handling, exception paths. Avoid padding with Lombok getter/setter tests (337 of 865 tests in scope) and lambda-forcing tests with no assertions. If time is short, add a few high-value tests and agree with the lead on a plan for the rest. Exclude generated Lombok code via `lombok.config` where policy allows. Be open about what remains untested (Mongo semantics, Jasper parity, XSD).
- **Common mistakes:** Gaming the number and reporting it as quality.
- **Follow-up questions:** Better metric? (Mutation testing on gates.)

### Q77. You are asked to prove Java output equals Jasper for five contracts. Describe the plan.
- **Why the interviewer asks:** Verification design.
- **Ideal answer:** Choose contracts to cover branches: ASP 72/168 with sponsor and nominee; ASP deletion (98); ASP checklist copy; Template12 070504 BMW; Template12 070412 pre/post cutoff. For each: request parameters, Jasper PDF text, Java XML. Compare: which sub-reports exist, row counts per sub-report, order, key amounts (gross, GST/SST, premium due, rounding), debit/credit flag, null display ("NA" in Jasper vs empty), dates. Record differences as defects vs known gaps (page parity, stubs, `INST_IND`). Automate later as a golden-file test with a Mongo test dataset. Needs the renderer team to apply the XML to the template; otherwise compare data-level only.
- **Common mistakes:** Comparing totals only.
- **Follow-up questions:** What do you do with known gaps? (List them in the report, do not hide.)

### Q78. The DBA says `*_VI_*` objects are Mongo views and cannot be indexed. What do you do?
- **Why the interviewer asks:** Performance with data constraints.
- **Ideal answer:** A Mongo view uses the indexes of its source collection and runs its pipeline on every read. Confirm which `*_VI_*` names are views vs materialised collections (unknown from the repo). If views, run `explain` through the view; push selective `$match` stages into queries, keep the keys on the source collection indexed, or ask for materialised collections for hot views (`UWPL_VI_POLICY_SCHEDULE_TPL4`, `UWPL_VI_ENDT_*`). Avoid repeated reads of the same view in one request (merge handlers or cache per request). Record findings for the DB team.
- **Common mistakes:** Asking for an index on a view.
- **Follow-up questions:** Which handlers read the same view several times? (Template12 landscape bands on `TPL4`.)

### Q79. The Mongo cluster in an environment is version 4.4. What breaks?
- **Why the interviewer asks:** Dependency on server version.
- **Ideal answer:** `$setWindowFields` (RejectRisk row number) needs 5.0+, so RejectRisk fails there (inferred; other handlers also use it). Also check `$unionWith` (4.4+), `$lookup` with `let`/`pipeline` (3.6+), `$dateToString` timezone and `$convert` (4.0+). Steps: confirm the version with the DBA, make a fallback (sort and number rows in Java), and fail clearly with a message. Add the minimum Mongo version to the README. Ask which environments have which version.
- **Common mistakes:** Learning it from a production error.
- **Follow-up questions:** Java-side numbering code? (After sort with `Comparator.nullsLast`, assign `i + 1`.)

### Q80. After the Boot 4 upgrade, requests feel slower. How do you investigate?
- **Why the interviewer asks:** Performance triage method.
- **Ideal answer:** First, compare like for like: same request, same data, old vs new build. Check changes that could matter: `MongoTemplate`/`Aggregation` DEBUG logging (verbose and expensive; enabled in properties), Jackson conversion changes, HTTP client timeouts/pool (different beans for different clients), Tomcat version. Measure phases with logs or timing around handlers (no timing data exists now). Run `explain` for the heaviest pipelines. Fix the largest contributor first, usually the number of Mongo and REST calls per request. Do not blame the upgrade without data.
- **Common mistakes:** Changing several things at once.
- **Follow-up questions:** Tools? (Actuator metrics are not exposed now; add `metrics` carefully.)

### Q81. The service runs out of memory when generating a big XML. What do you do?
- **Why the interviewer asks:** Capacity.
- **Ideal answer:** Evidence for causes: `getMappedResults()` loads all rows; about 650 leaves per `transDet` with `Include.ALWAYS`; default placeholders; per-row `transDetList` in RejectRisk; the XML is held as one String, then embedded as a JSON string. Steps: take a heap dump and a request sample; reduce payload (`NON_NULL`), cap or split requests, remove duplicated lists, avoid copying the XML string more than needed, and set container memory limits with headroom. Check whether a few giant requests drive it.
- **Common mistakes:** Only raising `-Xmx`.
- **Follow-up questions:** Which module is most at risk? (RejectRisk with no date range; BMW with large cert range.)

### Q82. A load test shows request threads stuck and timeouts. What do you suspect?
- **Why the interviewer asks:** Concurrency and timeouts.
- **Ideal answer:** Many small Mongo calls and up to ~5 REST calls per Template12 request, each holding a Tomcat thread; Mongo `maxTime` is 600 s in some handlers; no inbound timeout; the connection pools (HTTP 100 total/20 per route; Mongo driver default 100, `option.*` keys are ignored). A slow upstream (8 s fast-lookup, 20 s others) multiplies. Steps: thread dump to see where threads wait (HttpClient pool, Mongo, application), metrics on pool use, lower `maxTime`, add `connectionRequestTimeout` to the clients that lack it (the motor primary bean has none), and consider parallelising independent calls and caching constants.
- **Common mistakes:** Increasing thread count first.
- **Follow-up questions:** Which clients lack the pool timeout? (`PlinePolicyRefClient`, `PlineSSTRolloutDateClient`, `PlinePgRpgeUtilsClient` via the primary bean.)

### Q83. A release causes wrong reports in UAT. How do you roll back?
- **Why the interviewer asks:** Release management.
- **Ideal answer:** The service has no schema or DB state, so rollback is a redeploy of the previous image tag through the Helm repo (inferred; `cicd-workflow.yml` is manual with an optional `imageTag`, pom version is static `0.0.1-SNAPSHOT`). Steps: stop the rollout, redeploy the previous tag, verify one report per module, keep the bad build for analysis, check config drift in Helm values (for example the removed Mongo URI), and write the cause. CHANGELOG shows a similar case: OPUSM-52530 "Reverted aggregation and lookup code changes unable to reproduce on BAU UAT". [CONFIRM: your team's rollback procedure.]
- **Common mistakes:** Hot-patching in UAT without a ticket.
- **Follow-up questions:** How to prevent? (Seeded-data tests for aggregation changes.)

### Q84. A production bug in RejectRisk needs a hotfix, but your branch has unrelated changes. What do you do?
- **Why the interviewer asks:** Git and release discipline.
- **Ideal answer:** Branch from the production line (the branch/tag deployed; confirm with the team, since targets are `retrofit-develop-R1/R2`), apply only the fix (cherry-pick or a small commit with the Jira key), add or update the exact test, build, run the module tests, and open a PR with a short risk note. Do not include `application.properties` changes or the "no dates" behaviour change unless approved. After release, merge the fix back into other branches (R2, feature branches) to avoid regressions. Keep the fix minimal.
- **Common mistakes:** Shipping a feature branch as a hotfix.
- **Follow-up questions:** Which two RejectRisk fixes are safest? (Date `$lt` + 1 day and null-bound handling, after tests; the no-date rule needs a decision.)

### Q85. You want to ship the RejectRisk "no dates" change safely. What do you do?
- **Why the interviewer asks:** Risk-controlled change.
- **Ideal answer:** Treat it as a behaviour change. Options: a config flag (`rejectrisk.allow-open-date-range`, default off = parity: empty result or 400), a max-rows cap, logging when it happens, and a release note to callers. Add tests for each mode. Roll out with the flag off, ask the business whether any user needs it, then decide. Monitor request size and time. Keep the default safe (parity with Oracle's zero rows) until sign-off.
- **Common mistakes:** Shipping default-on "all rows".
- **Follow-up questions:** Where do flags live? (Properties with env override; no real profiles.)

### Q86. A security scan flags the committed truststore and its plaintext password. What do you do?
- **Why the interviewer asks:** Secret remediation plan.
- **Ideal answer:** Facts: `mytruststore.p12` is in `src/main/resources`; `app.ssl.trust-store-password` is plain text in `application-env.properties` (added in `57c573b74`, 2025-12-06); a truststore password protects integrity, not a private key (a truststore holds public CAs - general knowledge), so risk is lower than a keystore, but it is still a policy finding. Plan: mount the truststore and password from a secret at deploy time, read them via env (`app.ssl.*` already uses `@Value`), remove the file and the password from the repo, keep a default PKCS12 type, rotate the password, update the allowlist. Test startup with a wrong password (the app fails to start by design).
- **Common mistakes:** Removing the file and breaking TLS to upstreams.
- **Follow-up questions:** Why does a custom truststore matter? (It replaces JVM default CAs.)

### Q87. You need to change one product branch in the 2,235-line `Template12PolicyDocumentService`, and a teammate edits the same file. What do you do?
- **Why the interviewer asks:** Working in a god class.
- **Ideal answer:** Coordinate: agree who edits which region, merge frequently, keep changes small and local (one `if` block plus a new helper in the base class or a new class). Put new logic in a new method or class instead of the middle of `execute`. Run the existing 34 service tests after every merge. Suggest a refactor ticket (registry of bands) but do not mix it with the feature. Add tests for your branch before changing code (characterisation).
- **Common mistakes:** Reformatting the file, which makes the diff unreadable.
- **Follow-up questions:** Tools? (`git diff -w`, small commits.)

### Q88. A reviewer asks why `AspEndtDocumentService.execute` is about 1,300 lines. How do you respond?
- **Why the interviewer asks:** Defending and improving design.
- **Ideal answer:** Be honest: the method ports four Jasper reports with every `printWhenExpression` as inline code, to keep behaviour traceable to the jrxml; it grew by merges (commits `7a925920c`, `4cd1cad2e`). It is a God method, hard to test and review (Sonar complexity). Propose steps: first characterisation tests (28 exist), then extract per-section methods (cover, schedule, debit, tax, service tax, transitional, landscape), compute the gates in one small object, and build `reportData` through a small list of section builders. Keep behaviour identical and refactor in separate PRs.
- **Common mistakes:** Getting defensive, or promising a full rewrite.
- **Follow-up questions:** What would you extract first? (The gate computation; it is pure.)

### Q89. A new teammate cannot run the service locally. What do you do?
- **Why the interviewer asks:** Onboarding.
- **Ideal answer:** Checklist: Java 17, Maven wrapper (3.9.11, `./mvnw`), a Mongo URI via `SPRING_MONGODB_URI` and `SPRING_MONGODB_DATABASE` env vars (never edit tracked files), reachable dev API base URLs (`API_BASE-URL`, `plines.api.base-url`; VPN/corporate network), the truststore on the classpath, context path `/gi/documentprintdata`. Start the app, call `GET /gi/documentprintdata/actuator/health`, then POST a sample RejectRisk request. Common failures: old Mongo keys (localhost fallback), missing network, wrong URL path. Add a README section with this (no ASP section exists today). Do not share real credentials in chat.
- **Common mistakes:** Sending your own properties file.
- **Follow-up questions:** Which sample first? (RejectRisk: one handler, no external calls.)

### Q90. UAT cannot reproduce a defect found in production. What do you do?
- **Why the interviewer asks:** Data-dependent bugs.
- **Ideal answer:** Think data and config differences: Mongo data copy date, WIP vs posted rows, type of `CONTRACT_ID`, missing rows in lookup collections, REST API behaviour (UAT may return "N"), different `API_BASE-URL`, time zone. Get the exact production request (without secrets), the contract id, and the response. Reproduce the pipelines on a read-only copy or with the same document shapes (anonymised). Add temporary targeted logging if allowed. Add a regression test with seeded documents shaped like the production data. Remember OPUSM-52530 (reverted because not reproducible on UAT).
- **Common mistakes:** Closing as "cannot reproduce".
- **Follow-up questions:** What data do you need from prod? (Only the specific documents, masked.)

### Q91. You are blocked waiting for the report team and the business asks for an ETA. What do you say?
- **Why the interviewer asks:** Communication under dependency.
- **Ideal answer:** State what is done, what is blocked and why, and what you can do in parallel. Example: the contract for multiple `reportData` is unconfirmed (disclaimer in `RejectRiskService`); meanwhile I can add a real XSD validation test, fix tests, and prepare options with trade-offs. Give a dated ask: "I need an answer by X to keep the Y date", name the owner, and propose a fallback decision if no reply. Write it in the ticket. Avoid promising dates that depend on others.
- **Common mistakes:** Silent waiting, or guessing the answer.
- **Follow-up questions:** Escalation path? [CONFIRM: your lead.]

### Q92. The mapping sheet, the XSD and the Java class disagree on a field name. Which wins?
- **Why the interviewer asks:** Source of truth.
- **Ideal answer:** None wins automatically. Compare: the mapping workbook (generated 2026-07-06, v0.1), `Pline.xsd` v0.1.6, and the Java `@JacksonXmlProperty` names. For RejectRisk the sheet and Java agree; irregular names (`reasoncode`, `updatedby`, `updateddate`) are in the XSD. If they disagree, find the Jasper field (the origin), then ask the schema owner which name is the contract; fix the other two and the changelog. Add the drift-guard reflection test so it cannot happen silently. Mapping sheet tabs other than RejectRisk were only skimmed, so read the relevant tab.
- **Common mistakes:** Renaming the Java field to match a guess.
- **Follow-up questions:** Case issues? (`Policyletterind`, `LOADING` vs `loading`.)

### Q93. Someone asks you to split PolicyDocument2 and Landscape back into separate reports. What do you answer?
- **Why the interviewer asks:** Architecture reasoning and impact.
- **Ideal answer:** The merge was done on purpose (commit `c0c6d449a`) to give one response and avoid duplicate rendering. Splitting again affects callers (new report names), the single `PlineReportData` header design, merge helpers, and 230 tests. First ask for the reason (for example separate printing, performance, ownership). Alternatives that keep one bean: filter parameters to emit only a subset of bands (add a `bands` filter), or a router that maps the old names to the merged service with a mode. If the split is still needed, re-enabling old classes is not enough (they are stale vs the merged logic); plan a refactor with shared band methods.
- **Common mistakes:** Re-adding `@Service` to the old classes.
- **Follow-up questions:** Risk of stale old classes? (They miss later fixes such as `toLandscapeReportData`, date passes and tags.)

### Q94. After a data migration, `CONTRACT_ID` is stored as a string in some collections. What happens and what do you do?
- **Why the interviewer asks:** Type-strictness edge.
- **Ideal answer:** Handlers match with `Long`/`Integer` (`safeNum`), so a string-stored id never matches and the lookups return nothing, silently: tax flags null, sub-reports missing. Detect with a Compass query on both types (`{CONTRACT_ID: "123"}` vs `{CONTRACT_ID: 123}`) and `$type`. Fix options: correct the data (best), or make handlers match both (`$in: [123, "123"]` or `$or`) at a performance cost and index awareness. Add a test for both types. Tell the data team so migrations keep types stable.
- **Common mistakes:** Converting only in some handlers.
- **Follow-up questions:** Other typed keys? (`VERSION_NO`, `PART_ID`, `VERSION`, `CAT_CODE`/`CODE_CD` strings.)

### Q95. A Mongo collection used by a handler is renamed (or the name is wrong). How do you find and handle it?
- **Why the interviewer asks:** Naming mismatch risk (U9).
- **Ideal answer:** Aggregations on a non-existent collection return empty results, not errors. Signs: a sub-report disappears. Example open question: the Java collection `UWGE_VI_ENDT_DEBIT` vs the Oracle view `CUSTOMER.UWPL_VI_ENDT_DEBIT` in the jrxml; which exists in Mongo is unverified. Steps: list collections in the target DB; compare with the constants in the handler (`REFER_RISK`, etc.); add a startup or smoke check that logs missing collections; fix the constant and the test; document the Oracle-to-Mongo name map.
- **Common mistakes:** Assuming names equal Oracle names everywhere.
- **Follow-up questions:** A cheap guard? (A health contributor that checks collection existence for key names.)

### Q96. You are asked to fix a Template4 landscape problem during your Template12 work. What do you do?
- **Why the interviewer asks:** Scope control.
- **Ideal answer:** Check the boundary: Template12 Landscape band L16 only emits the dispatch fields from `Template4LandscapeDispatchHandler` (`productcode`, `aspCutoffInd`, `boatPrint`, `sponsorPrint`, `gossPremB4Promo`); Template4's own bands belong to another scope (inferred). If the problem is in the dispatch call, fix in the dispatch handler with the Template4 owner's agreement. If it needs Template4 changes, raise a separate Jira with the owner and effort estimate, and keep your PRs separate. Tell the lead about the extra scope and its effect on dates.
- **Common mistakes:** Quietly absorbing scope.
- **Follow-up questions:** How to size it? (Read the Template4 jrxml first.)

### Q97. An interviewer says: "You told me your output is XSD-valid. Show me." What do you do?
- **Why the interviewer asks:** Tests integrity under pressure.
- **Ideal answer:** Correct it immediately: "I should be precise. Real service output does not validate against v0.1.6; I aligned payload types and structure to the XSD, and the existing XSD test validates only a minimal shell." Explain the known reasons (dates `dd/MM/yyyy` vs dateTime, empty elements, sibling order, `mainReportName`/`subReportName` and some parameters not in the XSD) and what you would do (service-level validation test with error collection and a ratchet). A hand-ordered RejectRisk XML validated with 0 errors, so the schema works. Stay calm and factual.
- **Common mistakes:** Defending the earlier claim.
- **Follow-up questions:** How would you prove progress? (Error counts by category over time.)

### Q98. In the interview you are asked to write a `$lookup` for the `CMGE_CODE` description live. What do you do?
- **Why the interviewer asks:** Live coding on a known pattern.
- **Ideal answer:** Write calmly in Java `Document` form: `new Document("$lookup", new Document("from","CMGE_CODE").append("let", new Document("rtc","$REJ_TYPE")).append("pipeline", List.of(new Document("$match", new Document("$expr", new Document("$and", List.of(new Document("$eq", List.of("$CAT_CODE","PLREJ_RISK_TYPE")), new Document("$eq", List.of("$CODE_CD","$$rtc"))))))), new Document("$limit",1))).append("as","RRT_ROW"))`, then `$addFields {RRT: {$arrayElemAt: ["$RRT_ROW",0]}}`. Explain: left-join semantics, `$limit 1`, and the index `{CAT_CODE, CODE_CD}`. State assumptions aloud (types are strings). If you forget a detail, say what you would verify in mongosh.
- **Common mistakes:** Forgetting `$$` for let variables or `$expr`.
- **Follow-up questions:** How would you test it? (Capture the pipeline and assert the `$lookup` stage content, plus a seeded-Mongo test.)

### Q99. A tester reports that BMW nomination data is empty in the merged Template12 PD2 block. What do you check?
- **Why the interviewer asks:** Template12-specific deviation (D4) in a scenario.
- **Ideal answer:** The merged method hard-codes `String certNo = ""` (`S12:583`) and calls `addNominationBMW(contractId, certNo, rd)`, so the nomination query never gets a certificate, while the old `Template12PolicyDocument2Service` read `certNo` from the request. Also, the Jasper PD2 jrxml has no band for `NominationBMW`; the BMW nominations belong to `CIPrint_BMW` (`Nomination_BMW` per certificate row, using `UWPL_RISK_NOMINEE`). Steps: check which block the tester looks at; compare with the PolicyDocument per-cert path; decide whether the extra PD2 call is needed at all; if yes, read `certNo` or use the cert range; add a test; confirm with the product owner.
- **Common mistakes:** Fixing only the empty cert without asking if the call should exist.
- **Follow-up questions:** Why is the call there? (Unknown; not in the PD2 jrxml or compiled `.jasper`.)

### Q100. You realise a mapping you delivered earlier was wrong, after it was reviewed and merged. What do you do?
- **Why the interviewer asks:** Ownership and honesty.
- **Ideal answer:** Say it quickly to the lead and the reviewers with facts: which field, what is wrong, impact (for example a value written to the wrong `transDet` leaf or dropped by the whitelist), who consumes it. Create a ticket, fix with a test that would have caught it, and ask whether output already went to UAT/production and needs correction or reprint. Update the mapping sheet and traceability note. Share the lesson (for example add a serialise-and-check test for the leaf). Do not hide it or blame the review. [CONFIRM: a real example from your work if you have one.]
- **Common mistakes:** Fixing silently in a later commit.
- **Follow-up questions:** How do you reduce the chance next time? (Reflection drift guard, serialisation tests, second reviewer on mapping.)
