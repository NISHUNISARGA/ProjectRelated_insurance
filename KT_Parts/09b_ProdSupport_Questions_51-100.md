# 09b - Production Support Questions 51-100 (opus-documentprintdata-service)

Same rules as 09a. "(inferred)" = reasoning, not proof. No real incident ticket exists in the repo, so answers describe how you would work, not what "happened". [CONFIRM: ...] marks a personal fact you must fill from memory or remove. No git authorship is attributed to you. Secrets appear as <redacted>.

---

## F. Release, rollback and change control

**Q51. How is this service released, and how would you roll back?**
- Why the interviewer asks: Basic production ownership.
- Ideal answer: The repo shows `cicd-workflow.yml` triggered only by `workflow_dispatch` (optional `imageTag`), which calls the shared action `opus-github-cicd-workflows/actions/java-microservice` (Java 17, Sonar, Wiz). The pom version is a static `0.0.1-SNAPSHOT`, so the image tag identifies a release. Helm charts and environment values are in another repo (`opus-helm-gi-documentprintdata-service`) that I did not see. Rollback is therefore presumably "deploy the previous image tag via the Helm repo" (inferred). The service owns no schema or state, so code rollback is stateless; the risk is configuration drift in the Helm values. [CONFIRM: your real release steps.]
- Common mistakes: Describing a pipeline you have not seen.
- Follow-up questions: What would you check after a rollback? (A known contract per module, plus `health`.)

**Q52. Which branch do you fix a production bug on?**
- Why the interviewer asks: Branch hygiene in a long-lived retrofit.
- Ideal answer: Facts from the repo: default branch `retrofit-develop-R1`, long-lived integration `retrofit-develop-R2`, work branch names like `OPUSM-<n>/retrofit-develop-R2-pline-<topic>`. PRs target R1 or R2 and Sonar runs on them. Commit messages need an `OPUSM-` Jira key (`jira-commit-check.yml`). I would confirm with the team which branch feeds the environment that has the bug before branching. The KT warns that `cc6a1b84d` (RejectRisk/ASP fixes) lives on another branch, 194 commits ahead and 1363 behind HEAD, so "fixed in the repo" does not mean "fixed in this branch".
- Common mistakes: Cherry-picking blindly across diverged branches.
- Follow-up questions: How do you check whether a fix is in your build? (`git branch --contains <sha>` - general git; then compare the handler code.)

**Q53. Your fix for RejectRisk equals an existing commit on another branch. How do you handle that?**
- Why the interviewer asks: Judgement about duplicate work and merge conflicts.
- Ideal answer: The working-tree handler/service diff equals `cc6a1b84d` (OPUSM-41763) except one comment. I would tell the team, then bring the change in once by the agreed path (cherry-pick or merge) so history does not hold two versions, and keep `application.properties` out. The `PlineReportBlocks`/`PlineReportNames` lines in the service exist only on HEAD, so a plain merge of that service file needs care. Also update the 7 tests that will break.
- Common mistakes: Re-typing the change and creating a conflict later.
- Follow-up questions: Which Jira keys relate? (OPUSM-41698 mapping fix, OPUSM-41763 branch, OPUSM-44159 umbrella.)

**Q54. A change to the shared `Transaction` class is proposed to fix XSD ordering. What is your concern?**
- Why the interviewer asks: Blast radius awareness.
- Ideal answer: `com.agcs.fo.opus.motor.requestpayload.Transaction` is used by Motor, Fire, Bond and others. Its field order is `templateDetails, mailingDetails, transactionDetails, reportParameters`, which differs from the Pline.xsd order. Adding `@JsonPropertyOrder` there changes every LOB's XML. Plines isolates itself with subclasses (`PlineTransactionDetails`, `PlineReportParameters`). I would first agree with the Motor/Fire/Bond owners and the print engine team, or create a Plines-only wrapper. Name-based consumers may not care about order, but that is not proven (inferred).
- Common mistakes: Editing the shared class without asking.
- Follow-up questions: How would you test the effect on other LOBs? (Existing LOB tests plus a golden XML comparison.)

**Q55. A PR fails the Sonar quality gate. What do you do?**
- Why the interviewer asks: CI ownership.
- Ideal answer: `sonar-code-review.yml` runs `mvn -B verify ... -Dsonar.qualitygate.wait=true` on PRs to R1/R2, so tests, JaCoCo and the gate run together. Thresholds live on the Sonar server, not in the repo. I open the Sonar report for new-code issues (the repo has history of "Sonar build and error fixes", for example `24e50ab9b`) and fix those; I do not lower the gate. Coverage counts DTO and payload tests too, because the pom has no exclusions and no `check` goal.
- Common mistakes: Adding Lombok getter tests just to pass the gate.
- Follow-up questions: Why is line coverage weak here? (Some tests only run lambdas, for example `fetchRejectRisks_pipelineLambdas_fullyForced`.)

**Q56. The Jira commit-check workflow rejects your PR. Why?**
- Why the interviewer asks: Process awareness.
- Ideal answer: `jira-commit-check.yml` verifies commits reference an `OPUSM` Jira key (allowed project `OPUSM`). Commit messages like "OPUSM-44159 Added report, sub report name in resposne" follow it. Fix by amending the message or adding the key to the PR. It runs on opened/synchronize/reopened/edited.
- Common mistakes: Using a ticket from another project key.
- Follow-up questions: Which ticket covers your three reports? [CONFIRM: your Jira numbers; the repo only shows 44159, 41763, 42246.]

**Q57. A vulnerability scan (Wiz) blocks a release. How do you respond?**
- Why the interviewer asks: Dependency management.
- Ideal answer: The pom has CVE-driven pins with comments: Boot 4.1.1 (CVE-2026-59270; the Spring Security 6.5.x fix was paywalled), Tomcat 11.0.25, Jackson 2.21.5 via `jackson-2-bom.version`, `commons-lang3` 3.18.0, `logback` 1.5.34, `log4j` 2.25.5. Remember: in Boot 4 the property `jackson-bom.version` means Jackson 3, so the old name silently does nothing. Some pins are below the BOM (commons-lang3 3.18.0 vs 3.20.0; logback 1.5.34 vs 1.5.38) so verify with `mvn dependency:tree` that you are not downgrading. CVE ids are quoted from pom comments, not independently checked.
- Common mistakes: Adding a pin without checking the BOM version.
- Follow-up questions: `org.json:json:20231013` is old and used by `CustomRequestFilter`. What would you do? (Upgrade, and cap body size.)

**Q58. After the Boot 4 upgrade several tests and the app broke. Name the traps you know.**
- Why the interviewer asks: Upgrade history is in the repo; shows learning.
- Ideal answer: `spring.mongodb.*` instead of `spring.data.mongodb.*`; `preferred-json-mapper=jackson2`; springdoc 2.8.15 gave `NoClassDefFoundError` on `WebMvcProperties`, fixed by 3.1.1; `@MockBean` replaced by `@MockitoBean`; `UriComponentsBuilder.fromHttpUrl` replaced by `fromUriString`; `getRawStatusCode()` replaced by `getStatusCode().value()`; explicit `WebClient.Builder` bean; Micrometer old pin 1.15.12 broke every `@SpringBootTest` (removed, now BOM 1.17.1). Commits `65f1a8711`, `c6b6803f3`, `9248b12ff`. I should say these are from the KT, not my own memory. [CONFIRM: which of these you personally handled.]
- Common mistakes: Claiming all of it as your own work.
- Follow-up questions: Why not downgrade to Boot 3.5? (It reopens CVE-2026-59270 and needs the property renames reverted.)

**Q59. Product code literals like `070412`, `72`, `168`, `98`, `999` are hard-coded. How does this affect support?**
- Why the interviewer asks: Maintainability and change risk.
- Ideal answer: A new product or endorsement code needs a code change and a release; there is no config table for it. The literals are in `AspEndtDocumentService` and many handlers (`.in("72","168")`, `ENDT_CODE != "999"`). Some product lists are in Jasper conditions too (for example PIDM Set A/B product lists in KT 02). When a business user says "product X should behave like ASP", the answer is a code change and tests, not a data change.
- Common mistakes: Promising a config-only fix.
- Follow-up questions: How would you design a config-driven approach? (Table-driven band registry - suggested in the KT.)

**Q60. UAT found an issue that unit tests missed. What does the repo say about that?**
- Why the interviewer asks: Test/UAT gap awareness.
- Ideal answer: `CHANGELOG.md` line 13: "2026-09-21 OPUSM-52530 Reverted aggregation and lookup code changes unable to reproduce on BAU UAT (PR #864)". It shows a UAT exists and that an aggregation change was reverted after UAT. Our unit tests mock `MongoTemplate`, so they cannot catch query correctness (null order, `$lookup` semantics, date boundaries, type mismatches). No Testcontainers or embedded Mongo is used in the in-scope tests. I would add seeded-Mongo tests for boundary cases.
- Common mistakes: Saying "we have 19,665 tests so it is safe".
- Follow-up questions: What does a green test suite prove here? (That code does what the author intended, not that it matches Jasper/Oracle.)

---

## G. Monitoring and observability

**Q61. What monitoring exists for this service, and what would you add?**
- Why the interviewer asks: Operations maturity.
- Ideal answer: In the repo: stdout logs with Boot defaults, actuator `health` and `info` only, no Prometheus or metrics exposed, no alerts, no custom health indicator, no JSON logs. I would add: metrics (request count, latency, empty-`reportData` count, external call failures), a correlation ID, and an exposed `loggers` endpoint behind security. These are proposals; whether the platform adds sidecar monitoring is unknown (inferred).
- Common mistakes: Claiming dashboards exist.
- Follow-up questions: Which single metric would catch silent failures best? (Count of responses with no `reportData` or with fallback defaults.)

**Q62. How do you trace one request across logs?**
- Why the interviewer asks: Practical debugging in a service with no request ID.
- Ideal answer: There is no MDC or request ID. I use time and the contract/endorsement ID that handlers and `CustomRequestFilter` print. `CustomRequestFilter` prints the original and modified body with `System.out`; `MongoTemplate` DEBUG prints pipelines with the ID values. I would search by `pContractId` in those lines and by handler class names. A proper fix is a filter that sets an MDC ID and returns it in a header (inferred suggestion).
- Common mistakes: Searching for a request ID that does not exist.
- Follow-up questions: What is wrong with logging full bodies? (PII for RejectRisk: `pCustomerName`, `pIdNo`.)

**Q63. A report is partly empty but returns 200. How do you find which band failed?**
- Why the interviewer asks: Fail-soft design.
- Ideal answer: Sub-bands in Template12 (`B12.add*`) catch exceptions and `log.warn` (for example `PlineCommonBlankHandler failed: {}`, `mergeNonNullFields: failed to copy field '{}': {}`); in ASP, safe calls return null; external clients log `<method> failed: {}` at ERROR. The primary fetch and the landscape dispatch propagate. So I grep WARN/ERROR with the handler name around the request time. Output is partial instead of failing, which is why the user only sees missing pages.
- Common mistakes: Looking only at ERROR level.
- Follow-up questions: Which calls are not wrapped? (`fetchDispatch`, primary document fetch.)

**Q64. Would you alert on empty responses? How?**
- Why the interviewer asks: Design thinking.
- Ideal answer: Yes, but with care: RejectRisk with no rows is legitimate. Alert on "default transaction returned" (unknown report name) and on strategy results with zero `reportData` for Template12/ASP, since a policy document always has at least the master row. Implementation (inferred): a counter in `CommonController` where it calls `createDefaultTransaction`.
- Common mistakes: Alerting on every empty listing.
- Follow-up questions: What status code would you return for an unknown report name? (404/400 - but callers may depend on 200; ask them first.)

**Q65. How do you check whether a slow report is Mongo or external API?**
- Why the interviewer asks: Isolation of latency.
- Ideal answer: Time the Mongo pipeline in mongosh (copy from the `MongoTemplate` DEBUG log) with `explain("executionStats")` and look for COLLSCAN and `totalDocsExamined`. For external calls, look for `... failed` logs and the call timing; the 8 s fast-lookup and 20 s timeouts bound them. Template12 can fire about 5 distinct utility calls (inferred), and ASP master policy resolution can run up to 3 times. I compare both before suggesting indexes.
- Common mistakes: Adding an index before the explain.
- Follow-up questions: Which indexes would you propose first? (`CMGE_CODE {CAT_CODE, CODE_CD}`, `{CONTRACT_ID, VERSION_NO}` on policy collections - recommendations only; `*_VI_*` may be views that cannot have their own indexes.)

**Q66. Many parallel heavy RejectRisk calls arrive. What is the risk?**
- Why the interviewer asks: Capacity thinking with no auth and no limits.
- Ideal answer: The API has no authentication, no rate limit and no inbound timeout. RejectRisk has no row cap, loads all rows into the heap, and sorts in memory (`$setWindowFields` plus `$sort`, 100 MB limit per blocking stage; no `allowDiskUse` in this handler). Combined with the new "no dates = everything" behaviour this is a DoS shape. Mitigation: gateway rate limits, mandatory date range, row cap (inferred suggestions).
- Common mistakes: Only talking about CPU.
- Follow-up questions: What Mongo-side limit exists? (`maxTime` 600 s in some handlers, not here.)

**Q67. What log lines would you want to grep during a Template12 incident?**
- Why the interviewer asks: Familiarity with actual log content.
- Ideal answer: `PlineUtilsClient ... failed` lines (ERROR), `log.warn` from `Template12PolicyDocumentBaseService` helpers, `IllegalStateException` with "Ambiguous ... ORA-01427", and `NumberFormatException` from the landscape dispatch. Mongo pipelines from `MongoTemplate` DEBUG. Exact strings can change, so I check the code before relying on them.
- Common mistakes: Quoting log text from memory.
- Follow-up questions: Where is the 500 root cause if the handler class swallows it?

**Q68. A customer says their ID number appears in logs. What do you check?**
- Why the interviewer asks: PDPA awareness.
- Ideal answer: RejectRisk carries `pCustomerName` and `pIdNo` in the request body. Places where it can land: `CustomRequestFilter` `System.out`, `MongoTemplate` DEBUG pipelines, legacy path `log.info("xml response data ...")` (the strategy path logs the XML only at DEBUG). The response XML contains names and IDs. I report it to the owner, reduce logging levels, and remove the `System.out` prints. Domain is Malaysian (inferred from the PDPA notice flag and company data).
- Common mistakes: Fixing only the RejectRisk module.
- Follow-up questions: Which logging key is enabled at DEBUG now? (`MongoTemplate` and `Aggregation`.)

---

## H. Stakeholders and communication

**Q69. Business says "the tax invoice did not print". Which questions do you ask before you open the code?**
- Why the interviewer asks: Triage with non-technical users.
- Ideal answer: Which module (Template12 or ASP)? Was it a checklist (preview) copy? Which `showTaxInvoice`? Was the policy/endorsement issued after the SST rollout date? Which endorsement code? Was it a master-policy certificate? These map to gates: `checklistCopy`, `showTaxInvoice`, `SHOW_STAX_INV_IND`, `SHOW_EINV_IND`, `SHOW_TRAN_INV_IND`, `PREM_BY_MST_IND`. Then I ask for the contract ID and endorsement number.
- Common mistakes: Asking for screenshots only.
- Follow-up questions: Which question rules out the code fastest? (Checklist copy - the SST invoice is by design not printed.)

**Q70. The report team says our XML does not match the old PDF. How do you compare?**
- Why the interviewer asks: Parity testing method.
- Ideal answer: Run both systems for the same filter. Compare counts, order, each column, and the display rules ("NA" for null in Jasper; Java sends null or empty). For RejectRisk compare null-name customers (sort order), boundary days, and empty-string filters. For Template12 compare gates and page blocks using the traceability tables in KT 02. The repo has no golden files or sample data, so this is manual today. Differences known in advance: page-number-dependent bands cannot be done in Java (for example `PrintSeriesFooter` call 2, `Blank_NoMargin` parity), and PD2 hard-codes `certNo=""`.
- Common mistakes: Comparing only one happy-path record.
- Follow-up questions: How would you automate it? (Golden files per scenario; seeded Mongo.)

**Q71. The print engine rejects our XML. What do you ask them?**
- Why the interviewer asks: Contract thinking.
- Ideal answer: Whether they validate against Pline.xsd at all (not evidenced anywhere I read), which XSD version, and the exact error. Our output does not validate against v0.1.6: dates are `dd/MM/yyyy` instead of `xs:dateTime`, empty elements for decimal/date/integer/boolean, sibling order differences, and unknown elements (`mainReportName`, `subReportName`, `pCreatedDateFrom`, `pCreatedDateTo`, `pReportTitle`). A hand-ordered RejectRisk XML validated with 0 errors, so the schema itself is workable. Then I agree whether the contract is "as the XSD says" or "as Jasper reads".
- Common mistakes: Saying "our output is XSD valid".
- Follow-up questions: Who owns the newest XSD? (Not found; only v0.1.6 exists on the machine.)

**Q72. Who owns the XSD and where is the latest version?**
- Why the interviewer asks: Governance honesty.
- Ideal answer: I do not know the owner. The reference copy `Plines LOB v0.1.6` equals the repo test copy. Java comments mention v0.1.7 to v0.1.10 and v0.1.9 for `pCreatedDateFrom/To`, but no such file exists locally or in the branches checked. Branch `OPUSM-44159-retrofit-develop-pline-jrxml-comparison-fixes` (commit `ea782121d`) edited the repo XSD without a version bump (adds `leasingBranch`, `premiumPrint`, `benefitPrint`, `nomineePrint`); it is not in HEAD. [CONFIRM: who the schema owner is and where later versions live.]
- Common mistakes: Making up a version history.
- Follow-up questions: What risk does this create? (Java drifts ahead of the contract.)

**Q73. A business analyst asks what "ASP" means. How do you answer?**
- Why the interviewer asks: Honesty about domain knowledge.
- Ideal answer: The expansion is not written in the code or the jrxml. Evidence: the nested pages are named `ASPlusSTI` and `ASPlusLLB`, benefit codes `ASP-A1` to `ASP-D5`, benefits look like personal accident (principal sum, hospital income, weekly benefit), landscape pages are group-style (members, sponsors, nominees), and Java ties it to product code `070412`. I would say "I did not find a written expansion" and ask the analyst. [CONFIRM: if you know the business expansion, use it.]
- Common mistakes: Inventing "Accident Student Plan".
- Follow-up questions: How do you identify an ASP policy in data? (Product config code 070412.)

**Q74. A user reports an issue in a report owned by another team (Template4 landscape, AutoRenewal). How do you handle it?**
- Why the interviewer asks: Scope boundaries.
- Ideal answer: Template12 calls Template4 for 070406 only through the dispatch row (`Template4LandscapeDispatchHandler`, band L16: `070406 && promoAmt>0`), partial by design (inferred: owned by the Template4 team). AutoRenewal has its own ASP letters (product 070412 and `tpaCutoffNum > 0`). I confirm the report name in the request, route the ticket to the owning module, and share evidence (request, XML part).
- Common mistakes: Fixing other teams' handlers in an emergency.
- Follow-up questions: How can you see which module a reportData belongs to? (`mainReportName` and `subReportName`.)

**Q75. What would you hand to a new support engineer on day one?**
- Why the interviewer asks: Knowledge transfer ability.
- Ideal answer: (1) The sample request bodies per module (KT 06 section 2.7). (2) The gate tables: Template12 product/scheme matrix, ASP service-tax gate. (3) The list of known gaps: stubs, unmapped fields, page-number-dependent bands, always-200. (4) Where logs and config live and what is not in the repo (Helm values, Jira, Confluence, DB scripts). (5) The three live bean names. (6) The XSD status. I would not hand over secrets.
- Common mistakes: A tool list without the gaps.
- Follow-up questions: What is still unknown? (ASP expansion, newest XSD, print engine behaviour, index state, deployment values.)

**Q76. Tell me about a production issue you handled.**
- Why the interviewer asks: Standard behavioural question; this repo has no incident tickets.
- Ideal answer: Be honest: "There is no incident ticket I can show from this repo. The closest real evidence of fixes is the commit history: the CONTRACT_ID type mismatch fix for ASP and the RejectRisk date-bound fix (which is not merged into my branch). I can explain each in situation-task-action-result form." Then give the CONTRACT_ID story: tax flags came back null because a String was compared with numeric data, so tax sub-reports silently never printed; fix `safeNum` in `AspEndtDocument2MainHandler.buildCommonPipeline`. Mark your own role with [CONFIRM: what you personally did]. Do not invent numbers, dates or people.
- Common mistakes: Making up an outage.
- Follow-up questions: How did you verify the fix? (Unit tests mock Mongo, so say what you would run against real data.)

**Q77. How do you prioritise: one user with a missing page vs a wrong value on all RejectRisk reports?**
- Why the interviewer asks: Judgement.
- Ideal answer: Wrong values affecting everyone (for example last-day rows missing) has a wider impact and may be a compliance listing; a single missing page may be a data or gate issue. I would check impact (how many callers), whether there is a workaround (the caller can widen the date range by one day, inferred), and risk of the fix. Severity definitions are not in the repo. [CONFIRM: your team's priority scheme.]
- Common mistakes: Prioritising by who shouts loudest.
- Follow-up questions: How do you communicate a workaround?

**Q78. When do you say "this is a data issue, not a code issue"?**
- Why the interviewer asks: Boundary judgement.
- Ideal answer: When the pipeline faithfully mirrors the Oracle SQL and the data is missing or inconsistent: missing `CMGE_CODE` row, duplicate partner view row, WIP vs live conflict, missing `CMGE_CODEREL_DET` entry, or an indicator that is `N` in the mirror. Evidence needed: the Mongo document and the matching JRXML condition. Yes if the service throws the emulated ORA-01427 error. No if the code differs from Jasper (for example PD2 `certNo`, the dropped parameters).
- Common mistakes: Using "data issue" to avoid analysing code.
- Follow-up questions: Who fixes Mongo data? (The mirror is read-only here; the owner is outside the repo.)

**Q79. What do you tell a caller whose request was fine but the output has no `mainReportName`?**
- Why the interviewer asks: Version awareness.
- Ideal answer: HEAD adds `mainReportName` through `PlineReportNames.tag(...)` on every Plines `reportData` (commits `e74bc0d5c` 2026-10-07 and `861f0666c` 2026-10-08). It only fills blanks. If the caller sees none, they may be on an older build or on a path that does not use `tag`; the legacy switch path is not tagged. Check the deployed image against the commits.
- Common mistakes: Telling them it is in the XSD (it is not in v0.1.6).
- Follow-up questions: How is `mainReportName` derived? (`PlineReportNames.mainNameOf`: last path segment, with the previous segment prepended unless it already starts with it.)

---

## I. Data refresh, mirror and master data

**Q80. A policy was issued a minute ago and the report still shows WIP data. Possible reasons?**
- Why the interviewer asks: Eventual consistency.
- Ideal answer: Collections are a Mongo mirror of Oracle objects; the sync method and delay are not in the repo. So posted collections may lag behind (inferred). Other reasons: `checklistCopy=Y` was sent, or Template12's `VERSION_NO = 1` rule for non-WIP is used. PD2 prefers posted data and falls back to WIP with `$ifNull`. [CONFIRM: sync mechanism and lag.]
- Common mistakes: Restarting the service.
- Follow-up questions: How would you prove lag? (Compare the document in Oracle and in Mongo.)

**Q81. A new endorsement code is introduced for add-member. What breaks?**
- Why the interviewer asks: Hard-coded domain knowledge.
- Ideal answer: Addition detection uses `ENDT_CODE in ("72","168")` in `AspAddMemberLandscapeDispatchHandler.additionExists` and the service. The new code will not trigger insured/premium landscape pages or the (stub) schedule-of-benefits. A code change is needed in both places, plus tests (`endtCode72_fetchesScheduleOfBenefitsAndLlbAndSti`, `endtCodeOther_skipsScheduleOfBenefits`).
- Common mistakes: Updating only the service.
- Follow-up questions: What other numbers are hard-coded? (98 deletion, 999 excluded from EndtMT.)

**Q82. `CMGE_CODEREL_DET` is wrong for a scheme and the landscape disappears. Who can fix it?**
- Why the interviewer asks: Data-driven behaviour.
- Ideal answer: The rule is data: the Template12 landscape is suppressed when `CMGE_CODEREL_DET` has `GPA_SCHEME`, `CI_LANDSCAPE_PRT`, `Y` for the scheme. A data owner changes the code table (outside this service). Support only proves the lookup returns a row. No code change is needed.
- Common mistakes: Changing the handler to ignore the table.
- Follow-up questions: How would you prove the lookup caused the suppression? (Query the collection by the scheme.)

**Q83. What is `SAPM_SYS_CONSTANTS` used for in these modules?**
- Why the interviewer asks: Knowing configuration in data.
- Ideal answer: ASP behaviour is controlled by constants such as `ASP_CUTOFF_DATE`, `ASP_CONFIG_DATE`, `ASP_LAUNCH_DATE`, `ASP_POLICYHOLDERLETTER_DATE` and flag `TPA_CUTOFF_IND`; the service-tax handler reads the e-invoice cutoff (`EINV_CUT_OFF`, under `UW_VALIDATION`) and the occupation cutoff (`OCCUP_CUT_OFF_DATE`). Changing a constant changes which pages and fields print, without a deploy. Not implemented in ASP schedule/debit handlers: `CUT_OFF_RPT` for `INST_IND` (defaults to "N").
- Common mistakes: Thinking cutoffs are in application properties.
- Follow-up questions: What does an ambiguous cutoff produce? (`AspServiceTaxInvoiceHandler` throws `IllegalStateException`, test `fetchServiceTaxInvoice_ambiguousEinvCutoff_throwsIllegalStateException`.)

**Q84. The partner (customer) view has two rows for one `PART_ID` and `VERSION`. What does RejectRisk do?**
- Why the interviewer asks: Join semantic risk.
- Ideal answer: The `$lookup` has `$limit 1` and no sort, so one arbitrary row is used. Oracle's outer join would duplicate the risk row. The name/ID filters also test only that one row. This is hidden, so I would check the view's uniqueness when numbers differ from Jasper (uniqueness not verified).
- Common mistakes: Believing `$limit 1` picks the latest row.
- Follow-up questions: How would you make it deterministic? (Add a sort inside the lookup pipeline.)

**Q85. How do you check data type problems in Mongo without changing code?**
- Why the interviewer asks: Practical skill.
- Ideal answer: In mongosh, run `find` on the collection and check `$type` of `CONTRACT_ID`, `VERSION_NO`, `COV_SEQ_REF`, `CREATED_DATE`. Numeric values stored as Int32/Long/Decimal compare equal with numbers, but a stored string never matches a numeric `$match`. Compare with the Java side (`safeNum`/`asNum` produce Long or Integer). Mongo does not raise errors for type-mismatch matches; it returns zero rows.
- Common mistakes: Using `find({CONTRACT_ID: "123"})` and concluding "not found".
- Follow-up questions: Which handlers convert the ID? (Every ASP handler `safeNum`; Template12 `asNum`.)

**Q86. `pFirstPageNumber` shows differently between callers. Why?**
- Why the interviewer asks: Parameter type drift.
- Ideal answer: Jasper declares it as String in PolicyDocument and Integer in PD2/Landscape. The XSD models it as `xs:string` (v0.1.4). Java defaults: Template12 `""` (the old PD2 service defaulted to `"1"`), ASP `"1"`. It is only echoed in `reportParameters`; the renderer does page numbering. Page-number-dependent bands cannot be computed in Java.
- Common mistakes: Expecting the service to number pages.
- Follow-up questions: Which bands depend on page parity? (`Blank_NoMargin`, PD2 Q1/Q5.)

---

## J. Security, risk and judgement

**Q87. Someone asks to add basic authentication quickly. How?**
- Why the interviewer asks: Practical security change.
- Ideal answer: Add a `OncePerRequestFilter` that checks a shared secret or JWT from the gateway, and change `authorizeHttpRequests` to `.anyRequest().authenticated()` while keeping `health` and `info` open. Roll out with the callers first sending the credential, then enforce. Credentials must come from a secret store, not from properties. Also decide Swagger. The caller list is unknown to me, so a phased rollout is needed.
- Common mistakes: Turning it on in production first.
- Follow-up questions: How do you keep CSRF decision consistent? (Stateless, CSRF stays off.)

**Q88. What is the SSRF protection in `PlineUtilsClient` and what are its limits?**
- Why the interviewer asks: Security understanding of your client code.
- Ideal answer: The base URL is configured, the path comes from config, `sanitize()` limits query values to `[A-Za-z0-9_\-./#]`, and `assertTrustedHost(url)` requires the URL to start with the configured base (commit `40d3fc1f7`, 2026-07-16). `PlinePolicyRefClient` checks `contractId` with `[A-Za-z0-9_\-]+`. Limits: `startsWith` semantics on a base without trailing slash (inferred), non-matching characters silently become "" (an operations effect: a value silently disappears), and `SecurityException` thrown outside the try propagates as 500. Not all Pline*Client classes were audited.
- Common mistakes: Saying user input can control the host.
- Follow-up questions: What symptom does `sanitize` cause in support? (A param becomes blank, the API returns a default.)

**Q89. Is the NoSQL injection risk real here?**
- Why the interviewer asks: Risk reasoning.
- Ideal answer: Low. `filterCondition` is `Map<String,String>`, so a client cannot send `{"$ne": ...}` objects; values go into `Document` equality filters. No `$where`. The RejectRisk name/ID filters are equality. A numeric ID goes through `Long.parseLong` and falls back to the raw string. Risk remains for heavy queries and information exposure, not injection.
- Common mistakes: Claiming "Mongo is immune".
- Follow-up questions: What if the map type changed to Object? (Operator injection becomes possible.)

**Q90. Credentials from `application.properties` were pushed once. What is the cleanup?**
- Why the interviewer asks: Incident response for secrets.
- Ideal answer: Rotate the credential first, then clean the history only if policy demands it. Add detection without file-wide allowlist (the existing allowlist covers two whole files). Check logs of the DB for unknown access (inferred). Document the ticket. The repo history already had Mongo URIs (gitleaks comment lists 18 occurrences in `application.properties`, OPUSM-50404), and `7dd2f2f67` removed the UAT URI from the tracked file.
- Common mistakes: Removing the line only.
- Follow-up questions: Where should it live afterwards? (Secret store via environment.)

**Q91. The `AspServiceTaxInvoiceHandler` copy in `src/test` - why should support care?**
- Why the interviewer asks: Hidden quality issue.
- Ideal answer: A 483-line near-copy (17 changed lines, for example `catch (Exception e)` vs `catch (RuntimeException e)`) with the same package and class name lives in the test tree. On the test classpath, test classes normally come first (inferred, not verified by running), so the 27 tests of `AspServiceTaxInvoiceHandlerTest` probably exercise the copy, not production, and JaCoCo looks at `target/classes`. So a green test and good coverage may not prove production behaviour. The fix is to delete the copy and re-run.
- Common mistakes: Saying it is harmless.
- Follow-up questions: How would you prove which class ran? (Print the class location in a test or check JaCoCo coverage on the production class.)

**Q92. What are the dangers of the "always 200" design when something is wrong?**
- Why the interviewer asks: Strategic view.
- Ideal answer: Callers cannot distinguish "no data", "unknown report" and "failed external call". Retries do not trigger, alerts do not fire, and bad output may be printed and sent to a customer. Examples from the code: unknown name returns `SUCCESS` with a default transaction, external failures return blanks, sub-band failures give a partial report. A better contract: non-200 for unknown names and a warning flag or header for degraded output. Needs agreement with callers.
- Common mistakes: Changing codes without telling callers.
- Follow-up questions: What would you return for a degraded report? (200 plus a field the caller can check, inferred.)

**Q93. Is the `ReportSettings` feature toggles object safe to use?**
- Why the interviewer asks: Null configuration trap from the config review.
- Ideal answer: `@ConfigurationProperties(prefix = "report.settings")` in `ReportSettings` is not defined anywhere in the properties, so `getFeatureToggles()` returns null (KT 06, inferred). Callers must null-check. It is not part of my three modules; I would mention it only as a config smell.
- Common mistakes: Using it without a null check.
- Follow-up questions: How do you find unread or undefined keys? (Search `@Value` and `@ConfigurationProperties`; KT 06 did this.)

**Q94. What changes if the `fnGetCodeDesc.path` key without a leading slash is used?**
- Why the interviewer asks: Config detail awareness.
- Ideal answer: `opus.fnGetCodeDesc.path=ext/utils/...` (line 174) has no leading slash. `PlineUtilsClient.url()` joins base (without trailing slash) and path, so it could produce `...allianzext/utils` if the reader does not add a slash. It is read by motor `FnGetCodeDescClient`, not by the three modules, and whether that client adds its own slash was not verified. I would test the call before changing anything.
- Common mistakes: Fixing config without a test call.
- Follow-up questions: Which key is the highest-risk path in the file? (`plines.fn.date.value.constants.path` - the comment admits it was guessed.)

**Q95. Two policy-ref paths exist (`pg-uwge-utils` and `pme-api`). Which does ASP use, and why does it matter?**
- Why the interviewer asks: Integration detail.
- Ideal answer: ASP ServiceTaxInvoice uses `PlinePolicyRefClient.getPolicyReference` with `plines.policy.ref.pme.path` (`/ext/policy/api/v1/pme-api/get-policy-ref`). The other method uses `plines.policy.ref.path` (`/pg-uwge-utils/get-policy-ref`). Two backend services answer the same business question. A third key `pg-uwge-utils.path.get-policy-ref` has no `/ext/policy` prefix and an underscore, likely stale. If the policy reference is blank on the invoice, check the exact path in use and the base URL.
- Common mistakes: Editing the wrong key (many duplicates exist).
- Follow-up questions: Where is the key misplaced? (`plines.policy.ref.pme.path` sits under the Marine heading.)

**Q96. A deploy changes `server.port`. What breaks?**
- Why the interviewer asks: Config coupling.
- Ideal answer: Port 8080 must match the container port in Helm and probes; context path `/gi/documentprintdata` must match ingress and probes. Helm values are outside the repo. Changing either without the chart gives not-ready pods (inferred).
- Common mistakes: Changing only one.
- Follow-up questions: What is the default if the key is removed? (8080.)

**Q97. How do you prove a hotfix did not change other reports?**
- Why the interviewer asks: Regression approach.
- Ideal answer: Run the unit tests of the touched module, then run saved requests for the other two modules in dev and compare the XML (before/after) with a diff. Check `git diff --stat` for shared files (`Transaction`, `PlineReportParameters`, `Template12PolicyDocumentBaseService`). There are no golden files in the repo, so I would create them for the incident. Note the class-level risk: `Template12PolicyDocumentBaseService` is separate from `PlinePolicyDocumentBaseService` (about 24 other templates) on purpose.
- Common mistakes: Running only the changed test class.
- Follow-up questions: Which base class would you avoid touching? (`PlinePolicyDocumentBaseService`, 4,083 lines, shared by about 24 templates.)

**Q98. What is the first thing you would improve to cut support effort?**
- Why the interviewer asks: Prioritisation and ownership.
- Ideal answer: (1) Log the exception in `GlobalExceptionHandler`. (2) Replace "unknown report returns SUCCESS" with a visible signal. (3) Add a correlation ID and remove `System.out` body printing. (4) Add real XSD validation of service output with a ratchet baseline. (5) Remove secrets from properties. I rank by how much each removes investigation time.
- Common mistakes: Proposing a rewrite.
- Follow-up questions: Which one is cheapest? (The logger line.)

**Q99. How would you explain to a manager that two parts of the code disagree (Java vs XSD) and nobody noticed?**
- Why the interviewer asks: Communication of a hard fact.
- Ideal answer: Plainly: the only machine check is `PlineXsdValidationTest`, which validates a hand-written minimal XML, and four tests named "generatesValidXml" re-use the same string. Real output was validated only by hand (older RejectRisk samples gave 929 and 1766 errors, mostly empty typed elements and `dd/MM/yyyy` dates). The structure and types are aligned in many classes, but not enforced for all. Propose a ratchet test. Do not hide or oversell. The older samples came from `T12_final_code` (1 Sep 2026), so counts are indicative only.
- Common mistakes: Blaming the previous team.
- Follow-up questions: What would the ratchet assert? (Errors <= baseline, then reduce.)

**Q100. Summarise in one minute how you would support this service.**
- Why the interviewer asks: Closing check.
- Ideal answer: "One endpoint, one bean per report. I start from the request and the XML in `data`, not the HTTP code. I check routing (exact bean name, merged names), then data (database name, WIP vs live, type of `CONTRACT_ID`), then gates (show flags, product and scheme, `checklistCopy`), then external APIs (silent blanks). I know the known gaps: stubs, unmapped fields, always-200, no auth, XML does not validate against v0.1.6. I keep secrets out of commits, and I say what I do not know: the expansion of ASP, the latest XSD and the deployment values." [CONFIRM: add one real case from your own work if you have one.]
- Common mistakes: Reading a tool list.
- Follow-up questions: Which unknown worries you most? (Print engine behaviour with the XML.)
