# 08b - Project Questions 51-100 (opus-documentprintdata-service)
> **AUTHORSHIP UPDATE (confirmed by the developer): git author `GS <extern.gs_nisarga@allianz.com>` is the developer. Wherever this file says commits cannot be attributed to the developer, read it as: the 11 GS commits listed in `KT_Parts/14_My_Commits_And_Ownership.md` ARE theirs; commits by other authors still must not be claimed. No GS commit touches `src/test`, so test authorship is still `[CONFIRM]`.**

Conventions as in 08a: facts come from the KT guide and KT_Parts 01-06; "(inferred)" = my reasoning; "[CONFIRM: ...]" = only you know. Do not claim git commits without checking the author.

---

### Q51. How was the test base built, and what do you think of its quality?
- **Why the interviewer asks:** To see if you can judge test quality, not only count tests.
- **Ideal answer:** Under ticket OPUSM-44159 the tests came in batches between 2026-09-25 and 2026-09-30 from at least five git authors (for example `0ab953cf8` DTO tests, `df70d6b5c`/`8fa89231e` handler batches, `3f2157b28` with +203k lines, `db7588897` service tests). Patterns: a fixed six-test template for DTO/payload classes, `@Mock MongoTemplate` plus `@InjectMocks` for handlers, lenient strictness, a `forcePipelines` helper "only to execute lambda bodies for coverage". The suite protects against accidental refactors in service branch logic, but does not prove parity with Jasper. Whether tools generated them is not stated; I say "consistent with template or tool-assisted generation".
- **Common mistakes:** Claiming they are AI-generated, or claiming they are a regression suite against Jasper.
- **Follow-up questions:** Why is `3f2157b28` a review problem? (It mixes tests with 200+ production edits.)

### Q52. Tell me about the Spring Boot 3.5 to 4.1.1 upgrade.
- **Why the interviewer asks:** Real migration experience.
- **Ideal answer:** Boot went from 3.5.14 to 4.1.1 on 2026-09-10 (`65f1a8711`) for CVE-2026-59270; the Spring Security fix was only free in 7.1.1. Code fixes recorded: `UriComponentsBuilder.fromHttpUrl` -> `fromUriString`; `getRawStatusCode()` -> `getStatusCode().value()`; `@MockBean` -> `@MockitoBean`; explicit `MediaType` imports; explicit `WebClient.Builder` bean; `spring.mongodb.*` instead of `spring.data.mongodb.*`; `jackson-2-bom.version` instead of `jackson-bom.version`; springdoc 3.1.1 (2.8.15 failed with `NoClassDefFoundError`); micrometer pin removed. [CONFIRM: which of these you did yourself.]
- **Common mistakes:** Saying "it was just a version change".
- **Follow-up questions:** What would a rollback to 3.5 reopen? (The CVE, and you must revert property renames.)

### Q53. How did you decide about the dependency pins (CVE work)?
- **Why the interviewer asks:** Security hygiene under real scanners.
- **Ideal answer:** `pom.xml` pins versions with CVE comments after Wiz scans: Tomcat 11.0.25, Jackson 2.21.5 (`jackson-2-bom.version`), commons-lang3 3.18.0, logback 1.5.34, log4j 2.25.5. Risk: a pin below the BOM is an accidental downgrade (BOM has commons-lang3 3.20.0, logback 1.5.38). Check with `mvn dependency:tree`. `org.json:20231013` and `junit-jupiter 5.9.0` are old and worth review. CVE ids come from pom comments, not verified by me.
- **Common mistakes:** Pinning and forgetting to remove the pin when the BOM catches up.
- **Follow-up questions:** Why did the micrometer pin break tests? (It broke `ApplicationContext` startup of `@SpringBootTest`; pin removed.)

### Q54. Why keep Jackson 2 when Boot 4 uses Jackson 3?
- **Why the interviewer asks:** Understanding of a hidden framework change.
- **Ideal answer:** Boot 4.1.1 prefers Jackson 3 (`tools.jackson`), which ignores `com.fasterxml.jackson` annotations used by `PayLoadRequest` and all `Pline*` payload classes. So `spring.http.converters.preferred-json-mapper=jackson2` is set (commit `9248b12ff`). XML output is built with the Jackson 2 `XmlMapper`. Future work: migrate annotations to Jackson 3.
- **Common mistakes:** Removing the property as "unused": then `@RequestBody PayLoadRequest` fails.
- **Follow-up questions:** What else must stay on Jackson 2? (All the `@JacksonXmlProperty` payload mapping.)

### Q55. What is the `spring.mongodb` prefix trap?
- **Why the interviewer asks:** Classic upgrade bug.
- **Ideal answer:** In Boot 4 the Mongo properties moved to `spring.mongodb.*`. Old `spring.data.mongodb.*` keys are silently ignored and the app falls back to `localhost:27017`. Symptom: connection refused to localhost with a "correct" config. Also `option.max-connection-pool-size` and `auto-index-creation` in `application.properties` are read by nobody (pool sizing is not applied; set pool options in the URI instead - inferred).
- **Common mistakes:** Debugging the network instead of the key names.
- **Follow-up questions:** How to confirm the real URI? (Startup log; actuator does not expose config here.)

### Q56. Does the API authenticate callers?
- **Why the interviewer asks:** A known weak point; tests honesty and security sense.
- **Ideal answer:** No. `SecurityConfig` is stateless, CSRF off, `/actuator/health` and `/info` open, other actuator paths denied, and `anyRequest().permitAll()`. Swagger UI and api-docs are open too. Protection must come from the network or gateway (not visible in the repo; inferred). The endpoint returns customer names and ID numbers (RejectRisk). Proposal: gateway auth plus a simple filter checking a shared secret/JWT, NetworkPolicy, rate limits.
- **Common mistakes:** "It has Spring Security, so it is secured."
- **Follow-up questions:** Why is CSRF off acceptable? (Stateless, no cookies, machine clients.)

### Q57. What does the service return for an unknown `reportName`?
- **Why the interviewer asks:** Error-contract awareness.
- **Ideal answer:** HTTP 200 with `status: SUCCESS` and a default empty transaction (with `mailingDetails` and `templateDetails`). A `null` reportName in the legacy `switch` would NPE -> 500 (inferred, not run). `NoDataFoundException` -> 404 exists but is never thrown (dead path). `IllegalArgumentException` -> 400; any other exception -> 500, and `GlobalExceptionHandler` does not log the exception. Callers must inspect the XML. The merged-away names (`.../PolicyDocument2`, `.../Landscape`, `.../EndtDocument2`, ...) fall into this behaviour.
- **Common mistakes:** Expecting 404.
- **Follow-up questions:** How would you improve it? (404/400 for unknown names, and log in the catch-all handler.)

### Q58. What is wrong with `application.properties` in your working tree?
- **Why the interviewer asks:** Secret handling.
- **Ideal answer:** The uncommitted diff adds a `#newDev` block with `spring.mongodb.uri=<redacted>` (user and password in clear text) and `spring.mongodb.database=OPUSAGICPRODR1`. Do not commit it. Use an env var (`SPRING_MONGODB_URI`) or `--spring.mongodb.uri`. Commit `7dd2f2f67` had already removed a UAT URI from the tracked file. `gitleaks.toml` allowlists both properties files wholesale (OPUSM-50404, review-by 2026-10-23), so a leaked secret there would pass CI. If it was ever pushed, rotate it.
- **Common mistakes:** Thinking the scanner will catch it (the file is allowlisted).
- **Follow-up questions:** How to keep it out of a commit? (`git restore` or `git add -p`.)

### Q59. What logging risks does the service have?
- **Why the interviewer asks:** PDPA/PII awareness.
- **Ideal answer:** `CustomRequestFilter` prints full request bodies with `System.out.println` (bypasses log levels; contains contract numbers, and for RejectRisk `pCustomerName` and `pIdNo`). `MongoTemplate` and `Aggregation` loggers are DEBUG, which logs pipelines with values. The legacy controller path logs the full response XML at INFO. Several logging keys target a non-existent package (`com.agcs.fo.opusMotorGateway.*`). No correlation id. Fix: remove prints, mask, DEBUG only in non-prod.
- **Common mistakes:** Only mentioning log volume.
- **Follow-up questions:** How to change log level at runtime? (`loggers` is enabled but not exposed, so you cannot.)

### Q60. What happens when an external API is down?
- **Why the interviewer asks:** Failure design.
- **Ideal answer:** Failures in `PlineUtilsClient`, `PlinePolicyRefClient`, `PlineSSTRolloutDateClient` are swallowed (log, return null or ""). The request stays HTTP 200 and the report looks valid but with blank fields. Worse: e-invoice and service-tax indicators default to "N", so those invoice pages vanish silently. Mitigation: alert on the `log.warn`, add a warnings section or metric, consider failing fast for gate-controlling calls.
- **Common mistakes:** Expecting a 502.
- **Follow-up questions:** Which calls control print gates? (`fnGetEinvoicePrintPolInd`, `fnShowStaxInvInd`, `fnEinvoicePrintEndtInd`, QR indicator.)

### Q61. What timeouts and pools are used?
- **Why the interviewer asks:** Operational knowledge.
- **Ideal answer:** `app.http.*`: connect 5 s, read/socket 20 s, connection-request 3 s, max 100 total and 20 per route, and a `fastLookupRestTemplate` with 8 s read. `PlineUtilsClient` uses the fast-lookup bean; `PlinePolicyRefClient`, `PlineSSTRolloutDateClient`, `PlinePgRpgeUtilsClient` take an unqualified `RestTemplate`, so they get the `@Primary` motor one. These came from a "Phase 1 incident fix". No retries or circuit breaker in Plines clients. Mongo `maxTime` is 600 s in some handlers, far above the HTTP timeouts.
- **Common mistakes:** Saying all clients have 8 s.
- **Follow-up questions:** Which handlers set no `AggregationOptions`? (`RejectRiskHandler`, `AspEndtDocumentMainHandler`.)

### Q62. How is SSRF prevented in the clients?
- **Why the interviewer asks:** Security review of your area.
- **Ideal answer:** Commit `40d3fc1f7` (2026-07-16). `PlineUtilsClient`: base URL from config only; `assertTrustedHost(url)` requires `url.startsWith(baseUrl)`; `sanitize()` allows only `[A-Za-z0-9_\-./#]` in query values (others become "" silently). `PlinePolicyRefClient` validates `contractId` with `[A-Za-z0-9_\-]+`. No call uses a URL from the request. Weak spot: `startsWith` on a base without trailing slash; but the base is set by operators.
- **Common mistakes:** Saying inputs are validated everywhere; `PlineEinvoiceQrIndicatorClient` was not audited.
- **Follow-up questions:** What happens when `sanitize` blanks a value? (A wrong-but-valid call; failure hidden.)

### Q63. How do you move this service between environments?
- **Why the interviewer asks:** Release awareness.
- **Ideal answer:** There is no single switch. You must change at least `API_BASE-URL`, `plines.api.base-url`, `workmen.api.base-url`, `spring.mongodb.uri`, `spring.mongodb.database`, and maybe the truststore. `spring.profiles.active=dev` is a label only - no `application-dev.properties` exists. `application-env.properties` is not a profile file; it is loaded by `@PropertySource`. Env var `API_BASE_URL` should map to `API_BASE-URL` by relaxed binding (inferred; verify). Several dev1 host keys are unread. Helm values live in another repo.
- **Common mistakes:** Changing only `API_BASE-URL`.
- **Follow-up questions:** How would you unify base URLs? (One property with per-client paths.)

### Q64. How does CI/CD work?
- **Why the interviewer asks:** DevOps basics for your project.
- **Ideal answer:** `sonar-code-review.yml` runs `mvn -B verify sonar:sonar -Dsonar.qualitygate.wait=true` on PRs to `retrofit-develop-R1/R2` (Java 17). `cicd-workflow.yml` is manual (`workflow_dispatch`) and calls a shared action. Also CodeQL (weekly), secret scan (PR diff and nightly full history), Jira commit check (`OPUSM-` key required), a Claude PR review, and a changelog bot. Helm charts are in `opus-helm-gi-documentprintdata-service` (not in this repo). Quality-gate thresholds are on the Sonar server.
- **Common mistakes:** Saying deploy is automatic on merge.
- **Follow-up questions:** Rollback? (Redeploy the previous image tag via Helm - inferred.)

### Q65. Which branches hold which work?
- **Why the interviewer asks:** Real repo awareness; branch confusion is common.
- **Ideal answer:** Current branch `feature/retrofit-develop-updated`; default `retrofit-develop-R1`; integration `retrofit-develop-R2`. `cc6a1b84d` (RejectRisk/EndtSchedule/EndtDocument fixes) and `578778080`/`ba9e29586` (CONTRACT_ID fixes) are on `OPUSM-41763` branches, not HEAD. `ea782121d` (XSD edit adding `leasingBranch`, `premiumPrint` etc.) is on another branch. Content may still be equal through squash or cherry-pick (HEAD's `4cd1cad2e` probably consolidates some). Check with the team what is merged to the target branch.
- **Common mistakes:** Assuming a commit on the project is in HEAD.
- **Follow-up questions:** How do you check? (`git merge-base --is-ancestor <hash> HEAD`.)

### Q66. Which Jira tickets relate to your modules?
- **Why the interviewer asks:** Traceability.
- **Ideal answer:** From commit messages: OPUSM-44159 (umbrella: Boot 4 upgrade, XSD, tests, report names), OPUSM-41763 (Template12/ASP/RejectRisk fixes branch), OPUSM-42246 (working code for Pline, multiple report data), OPUSM-41698 (RejectRisk mapping aligned with Pline.xsd v0.1.6), OPUSM-42045 (merge non-null values), OPUSM-40884/36054 (ASP wiring). I did not see Jira text. [CONFIRM: the exact Jira ticket text and which keys are yours.]
- **Common mistakes:** Inventing ticket content.
- **Follow-up questions:** How does CI enforce ticket keys? (`jira-commit-check.yml`, project `OPUSM`.)

### Q67. What was the hardest problem you solved? 
- **Why the interviewer asks:** Real experience and problem solving.
- **Ideal answer:** Pick one that is true. Candidates that match the code: the RejectRisk date and null-bound bug (`$lte` vs `$lt` + 1 day), the CONTRACT_ID type mismatch where sub-reports silently vanish, or reproducing Jasper print conditions (including `false &&` bands and page-parity rules). Structure: symptom, how you found the root cause, fix, how you tested, what remains open. [CONFIRM: which one you really worked on; do not use a story you cannot explain at the code level.]
- **Common mistakes:** A vague story, or a story from the KT file that is not yours.
- **Follow-up questions:** What would you do differently? How did you prove the fix?

### Q68. How did you check that Java output matches Jasper?
- **Why the interviewer asks:** Verification mindset.
- **Ideal answer:** Honest answer: by reading the jrxml and the Java side by side (print conditions, SQL, parameters, fields) and writing traceability tables; unit tests assert branches and mapping. There is no automated Jasper-vs-Java comparison and no golden file in the repo. A proper check would be: pick contracts for each product/scheme, generate the Jasper PDF and the Java response, compare which sub-reports appear, row counts, order, key amounts, and "NA" null display. [CONFIRM: whether you did any manual data comparison in UAT.]
- **Common mistakes:** Claiming tests prove parity.
- **Follow-up questions:** What sample would you choose first? (An ASP 72/168 endorsement with sponsor and nominee.)

### Q69. How did you read the Jasper dump?
- **Why the interviewer asks:** Practical analysis method.
- **Ideal answer:** The real jrxml is in `*_files/*_jrxml.data` (the `.jrxml.xml` files are small descriptors). I followed `ReportHierarchy.csv` (exists only for EndtDocument and EndtDocument2), then every `subreportExpression`, `printWhenExpression`, `queryString`, `field` and `parameter`. Rule used: a band with false `printWhenExpression` hides all its elements. For nested levels I read each sub-report. Gaps: some sub-reports are missing in the dump (`PerilsClausesDetails_POSM`), and landscape trees come from my reading, not a csv.
- **Common mistakes:** Reading only the main report.
- **Follow-up questions:** How did you find orphans? (Grep of all `subreportExpression`; some Template12 files are not reachable from the three mains.)

### Q70. What is the role of the mapping workbook?
- **Why the interviewer asks:** The requirement side of your XSD work.
- **Ideal answer:** `Pline_MASTER_MAPPING_SHEET_v0.1 (final1) 1 5.xlsx` (generated 2026-07-06, tabs per report incl. RejectRisk, PolicyDocument, EndtDocument) and `RejectRisk_XSD_Mapping_sheet.xlsx` list Jasper field -> XSD path. The RejectRisk sheet was read fully and agrees with the Java setters. The master workbook was only skimmed, so I should not claim I cross-checked every tab. Read both again before an interview.
- **Common mistakes:** Quoting tabs you never opened.
- **Follow-up questions:** If sheet and XSD disagree? (Raise with the schema owner; do not choose silently.)

### Q71. How did you handle things nobody could tell you (ASP meaning, `SB_`, `OCP_`)?
- **Why the interviewer asks:** Behaviour under ambiguity.
- **Ideal answer:** I wrote down what evidence shows and what it does not. ASP: no expansion; evidence is `ASPlus` names and product 070412. `SB_` and `OCP_` collection prefixes are not documented in files I read. Whether `*_VI_*` are Mongo views or materialised collections is unknown (views cannot have their own indexes). I ask the owner, and in interviews I say "not documented" instead of guessing.
- **Common mistakes:** Making up an expansion.
- **Follow-up questions:** Who would you ask? [CONFIRM: your lead / report team / DBA names - do not invent.]

### Q72. How did you work with the consumer (report/print team)?
- **Why the interviewer asks:** Cross-team contract awareness.
- **Ideal answer:** The shared contract is the XML shape. Open items that need their sign-off: does each `reportData` equal one print unit (RejectRisk disclaimer, Landscape comment), how page parity and blank pages are handled, whether the renderer prints the ASP schedule-of-benefits from `actionCode`, whether it tolerates unknown elements (`mainReportName`), and the date format. None of this is confirmed in the repo. [CONFIRM: any real discussion or sign-off you had; do not claim one.]
- **Common mistakes:** Treating unit tests as consumer sign-off.
- **Follow-up questions:** How do you avoid breaking them? (Additive changes only; version the XSD; contract test.)

### Q73. What feedback did you get in code review?
- **Why the interviewer asks:** Growth mindset.
- **Ideal answer:** [CONFIRM: your real review comments.] Evidence-based topics that appear in the repo: Sonar issues (`c336f76fa` "fixed sonar issues"), a Claude PR review fix (`7dd2f2f67`), CodeQL and secret-scan findings, and Jira key checks. Say what you changed in response, for example moving a secret out of a tracked file or using the Plines `DateUtils` instead of the motor one.
- **Common mistakes:** Saying "no feedback".
- **Follow-up questions:** A comment you disagreed with and why?

### Q74. How does the Sonar quality gate affect your work?
- **Why the interviewer asks:** Process awareness.
- **Ideal answer:** PRs to R1/R2 run `verify` plus Sonar with `qualitygate.wait=true`, so the PR fails if the gate fails. JaCoCo 0.8.12 `prepare-agent` + `report` produce `target/site/jacoco/jacoco.xml`, which Sonar reads. The pom has no minimum-coverage `check`. The 25-30 Sep test push and "Sonar build and error fixes" commits suggest the tests were written to satisfy the gate (inferred). Danger: coverage targets reward low-value tests like Lombok getters.
- **Common mistakes:** Quoting a coverage number (none exists locally).
- **Follow-up questions:** What would be a better metric? (Mutation testing, or tests on gates and mapping.)

### Q75. What would you check about Mongo performance in this service?
- **Why the interviewer asks:** Practical DB thinking.
- **Ideal answer:** No index definitions are in the repo (`auto-index-creation` is ineffective and there are no `@Document` entities). Recommended (labelled recommendations): `CMUW_PL_REFER_RISK {CREATED_DATE:1}`, `CPGE_VI_PARTNERS_ALL {PART_ID,VERSION}`, `CMGE_CODE {CAT_CODE,CODE_CD}` (used by almost every handler), `{CONTRACT_ID, VERSION_NO}` on contract tables, `{CONTRACT_ID, ENDT_NO}` and `{REF_NO}` on endorsement views. Verify with `explain("executionStats")` first. RejectRisk issues: four lookups per row, customer filters applied after the lookup, no `allowDiskUse`, unbounded result. Template12/ASP issue: many small aggregations per request.
- **Common mistakes:** Stating the indexes exist.
- **Follow-up questions:** How do you see the pipeline? (`MongoTemplate` DEBUG log.)

### Q76. Explain WIP vs posted data and `checklistCopy`.
- **Why the interviewer asks:** Core domain/data concept.
- **Ideal answer:** `WIP_*` collections hold work-in-progress (not yet issued) data; unprefixed ones are posted/live. A checklist copy is a draft printed before issue, so it lives in WIP. Jasper SQL queries both with UNION; Java repeats it per handler with a precedence rule (posted VERSION_NO=1 first, else WIP not in `OCP_POLICY_CONTRACTS`; or `isWip ? WIP_x : x`). Note: Template12's primary does not branch on `checklistCopy` itself; it passes it to REST functions and the PDPA test. In ASP, `checklistCopy=Y` closes the service-tax gate.
- **Common mistakes:** Assuming `checklistCopy=Y` switches every query to WIP.
- **Follow-up questions:** Meaning of `SB_`? (Not documented; do not guess.)

### Q77. What is `ASP_CUTOFF_IND`?
- **Why the interviewer asks:** Specific business switch.
- **Ideal answer:** 'Y' when the issue date is on or after the `SAPM_SYS_CONSTANTS` `ASP_CONFIG_DATE`/`ASP_CUTOFF_DATE` value; 'N' otherwise or if the row is missing. For 070412: before the cutoff the old landscape ASP pages print (Landscape L1-L3, Note, PerilsClauses, ComplaintRedress); on or after it the new portrait ASP pages print inside CIPrint (`ParticularOfInsured/Premium/Sponsor/NominationPotrait_ASP`, `PerilsClauses_ASP`, `Blank_NoMargin`, `ComplaintRedress_NoMargin`). Java computes it in each of the three Template12 primaries.
- **Common mistakes:** Confusing it with `PIDM_CUTOFF_POL`.
- **Follow-up questions:** Boundary? (`!issueDate.before(cutoff)`, so `>=`.)

### Q78. How do BMW and PA4A differ in code?
- **Why the interviewer asks:** Specific branch knowledge.
- **Ideal answer:** BMW/BMW1 (070504): `CIPrint_BMW` returns one row per certificate between `pCertNoFrom` and `pCertNoTo`; for each row S12 calls `addCIPrintBmwNestedSubreports` (Coverage_BMW, Nomination_BMW, Note_BMW, CoverageBenefit_BMW, PerilsClauses_BMW, ComplaintRedress_NoMargin) using `addSingleTransDet`. BMW also has `PREM=0`, so no PD2 debit note. PA4A (070414): `CIPrint_PA4A` also loops per row with nested SOB variants (`SHOW_NEW_SOB`, `SHOW_SOB_P3_IND`), perils (`CLAUSE_PRINT>0`) and nomination (`NOM_PRINT>0`). Cert-range values are set on report parameters but not serialised.
- **Common mistakes:** Thinking they are single-row bands.
- **Follow-up questions:** Risk of a large cert range? (One set of handler calls per row; no cap.)

### Q79. Explain the tax invoice family in Template12 PD2.
- **Why the interviewer asks:** Business-critical documents.
- **Ideal answer:** GST invoice (`TaxInvoince`): `showTaxInvoice=Y`, TAX>0, `SHOW_GST_INV_IND=Y`. SST service tax invoice: `showTaxInvoice=Y`, `SHOW_STAX_INV_IND=Y`, `EINV_PRINT_IND=Y` (PA4A has its own variant; 070406 uses Template4's). Nested: STCompanyContact, product info (MST variant if `PREM_BY_MST_IND=Y`), PL/PLSub if `STAX_COUNT>1`, QRSection (consent, VALID, QR_IND). Transitional invoices if `SHOW_TRAN_INV_IND=Y`. Known gap: the QR section uses the request `pTaxInvNo` (default "") instead of the row's invoice number, so QR can be empty unless the caller sends it; and Java adds the transitional Blank before invoice 1 instead of between the two.
- **Common mistakes:** Forgetting `EINV_PRINT_IND`.
- **Follow-up questions:** Where do the indicators come from? (REST: `fnGetEinvoicePrintPolInd`, `fnShowStaxInvInd`.)

### Q80. What is the `PlineReportParameters` whitelist problem?
- **Why the interviewer asks:** A silent-drop trap that real code has.
- **Ideal answer:** `PlineReportParameters` extends the motor `ReportParameters` and has `@JsonIncludeProperties` with 73 names so base-class fields do not leak into Plines XML. A setter that writes a non-whitelisted property is silently dropped. Examples dropped: `pContractId`, `pPolicyNo`, `pCertNoFrom/To`, `pTaxInvNo`, `pissueDate`, `reportName`; `ShowOutPrintCovLetter` (setter name differs from whitelisted `pShowOutPrintCovLetter`). Six whitelisted names are not in v0.1.6: `pCreatedDateFrom/To`, `pReportTitle`, `pshowEndtSchedule`, `pShowOutPrintCovLetter`, `pShowPDPANotice`. ASP GST/SST label flags may not reach the XML (unverified).
- **Common mistakes:** Believing every `rp.setX` call reaches the output.
- **Follow-up questions:** How would you detect it? (A test that serialises and checks the property.)

### Q81. Why do request defaults matter?
- **Why the interviewer asks:** Detail awareness that causes real behaviour differences.
- **Ideal answer:** Jasper parameters have no defaults and would NPE on null; Java uses `getOrDefault`. Template12: `showComplaint="Y"`, `showScheduleStd="Y"`, `showDebitNote="N"`, `showTaxInvoice="N"`, `checklistCopy="N"`, `versionNo="1"`, `ShowOutPrintCovLetter="N"` (capital S key). ASP: show flags default "N", `pFirstPageNumber="1"`. So a minimal Template12 request prints the complaint page by default. ASP `showComplaint` is only echoed, `pPolicyNo` is unused, and `pSchemeName` is not read at all.
- **Common mistakes:** Assuming the default for `showComplaint` is N.
- **Follow-up questions:** Which keys are case-sensitive? (`ShowOutPrintCovLetter`.)

### Q82. What did UAT teach the team?
- **Why the interviewer asks:** Real-world testing gap.
- **Ideal answer:** `CHANGELOG.md` line: "2026-09-21 OPUSM-52530 Reverted aggregation and lookup code changes unable to reproduce on BAU UAT (PR #864)". This shows a UAT environment exists and found a problem unit tests missed. Lesson: mocked-Mongo unit tests cannot catch aggregation semantic changes; you need data-level checks. [CONFIRM: any UAT defect you personally handled.]
- **Common mistakes:** Claiming that ticket as yours.
- **Follow-up questions:** How would you prevent it? (Seeded Mongo tests and a before/after comparison on UAT data.)

### Q83. How far does the Template12 Landscape cover Template4?
- **Why the interviewer asks:** Scope boundaries.
- **Ideal answer:** Band L16 (`070406 && PROMO_AMT > 0`) calls `Template4LandscapeDispatchHandler.fetchDispatch` and emits only the dispatch fields (`productcode`, `aspCutoffInd`, `boatPrint`, `sponsorPrint`, `gossPremB4Promo`). Template4's own landscape bands are not emitted here (inferred: owned by the Template4 work). Also PROMO_AMT can differ from Jasper (see Q92).
- **Common mistakes:** Claiming full Template4 coverage.
- **Follow-up questions:** Who owns the rest? [CONFIRM: your team.]

### Q84. What dead or orphan code did you find?
- **Why the interviewer asks:** Code hygiene awareness.
- **Ideal answer:** `Template12PolicyDocument2Service`, `Template12LandscapeService`, `AspEndtDocument2Service`, `AspEndtDocumentLandscapeService` (disabled but tested); `PlineT12ParticularOfInsuredPosHandler` (no caller, 6 tests); `Template12PolicyDocumentHandler` (only wired in `PlineXsdTemplate2BaseService`); the `DeletionMemberLandscape` dispatch-by-`reportName` branch; hard-coded test ids left in comments ("6063333", "2885273"); unused locals; duplicated `det.setJoinDate`; a file named `null` and a committed `.scratch/` folder in the repo; `pom.xml` artifactId with spaces.
- **Common mistakes:** Deleting without checking callers.
- **Follow-up questions:** Safe removal plan? (Grep callers, remove tests and class together, run build.)

### Q85. How would you refactor the big classes?
- **Why the interviewer asks:** Design maturity.
- **Ideal answer:** Facts: `Template12PolicyDocumentService` 2,235 lines, `Template12PolicyDocumentBaseService` 3,732, `AspEndtDocumentService` 1,701 with `execute` about 1,300 lines. Plan: first freeze behaviour with golden tests; then a table-driven band registry (condition + handler + mapper), a shared product-context object instead of three fetches, replace reflective merge with explicit mappers, extract landscape mapping (about 700 duplicated lines), remove dead classes. Do it in small PRs, never mixed with behaviour changes.
- **Common mistakes:** "Rewrite everything."
- **Follow-up questions:** How do you keep risk low? (Characterisation tests on current output first.)

### Q86. What went well in the project?
- **Why the interviewer asks:** Reflection and self-awareness.
- **Ideal answer:** [CONFIRM: your real answer.] Evidence-based candidates: a clear traceability method (Jasper condition -> Java -> Mongo -> XSD), merged beans that give one response, tests that name business gates, and spotting issues like the date bound, null bounds and type mismatch. Do not claim metrics (none exist in the repo).
- **Common mistakes:** Inventing numbers such as "reduced time by 40%".
- **Follow-up questions:** How did you measure success?

### Q87. What would you do differently?
- **Why the interviewer asks:** Honest self-review.
- **Ideal answer:** [CONFIRM: personal view.] Reasonable evidence-based points: agree the XSD contract and version with the owner before changing Java; add a real XSD validation test early; add a seeded-Mongo test for each date or sort rule; confirm the renderer contract before the "reportData per row" design; keep tests and production changes in separate commits; never let secrets sit in a tracked properties file.
- **Common mistakes:** "Nothing" or blaming the team.
- **Follow-up questions:** Which of these is cheapest to start now?

### Q88. Describe the timeline of your work.
- **Why the interviewer asks:** Ownership over time.
- **Ideal answer:** Project window 12 May - 30 Dec 2026 [CONFIRM: your real milestones]. Repo history only runs to 2026-10-08. Themes from git: foundation and first XSD (16 Jul - 3 Aug), Template12 merge (3 Aug), schema alignment (8 Aug), stabilisation (10-11 Aug, 31 Aug), ASP wiring and landscape fold-in (26-28 Aug), multiple reportData (Sep), Boot 4 upgrade (10-24 Sep), test drive (25 Sep - 1 Oct), report/sub-report names (6-8 Oct). The part after 8 Oct is not visible to me.
- **Common mistakes:** Presenting the whole timeline as personal work.
- **Follow-up questions:** What was left at the end? (The open items in Q50.)

### Q89. How would you hand over your modules to a new developer?
- **Why the interviewer asks:** Communication and ownership.
- **Ideal answer:** Give: the three bean names; one-page flow per module; the traceability tables (Jasper -> Java -> Mongo -> XSD); the list of known deviations and gaps (D1-D19 style); the test map and what it does not prove; how to run locally with an env-var Mongo URI; sample requests; production support cheat sheet; open questions with owners. Walk through one real request per module with a debugger.
- **Common mistakes:** Handing over only code.
- **Follow-up questions:** What would you pair on first? (A Template12 branch addition.)

### Q90. A user says "the report is wrong". How do you triage in this system?
- **Why the interviewer asks:** Support readiness.
- **Ideal answer:** Use the cheat sheet. Whole Template12 report 500: empty/non-numeric `pContractId` (`NumberFormatException` in landscape dispatch) or WIP/live conflict (`IllegalStateException`). ASP tax invoice missing: check the four gate conditions and the external API. Flags null: ID type mismatch. RejectRisk missing last-day rows: old `$lte`. Empty XML with SUCCESS: wrong report name/no bean. Old report name returns nothing: use the merged name. Connects to localhost: old Mongo keys. Then enable pipeline DEBUG and run the pipeline in Compass.
- **Common mistakes:** Starting with the renderer.
- **Follow-up questions:** Where are logs? (stdout; no correlation id.)

### Q91. What was the `COV_SEQ_REF` sorting issue?
- **Why the interviewer asks:** Concrete Oracle-vs-Mongo detail.
- **Ideal answer:** Jasper orders by `cast(COV_SEQ_REF as decimal(15,2))`. If stored as string, a raw sort gives "10" before "2". Most handlers use `$toDouble` before `$sort`. Three still sort raw: `PlineT12ParticularOfInsuredLandscapeAmcHandler`, `PlineT12ParticularOfDiagnosisLandscapeHandler`, `PlineT12ParticularOfDiagnosisLandscapeAmcHandler`. Whether the field is string is unverified. Fix: `$addFields` numeric then `$sort`; use `$convert` with `onError` if empty strings exist; add a test with "10" and "2". Null rows sort first in Mongo (Oracle last).
- **Common mistakes:** Sorting after a `$project` that renamed the field.
- **Follow-up questions:** What if the value is "ABC"? (`$toDouble` throws; like Oracle's cast error.)

### Q92. Why can the Template12 promo amount differ from Jasper?
- **Why the interviewer asks:** Deep SQL semantics.
- **Ideal answer:** Jasper joins `wip_UWPL_RISK_PERSON` to `wip_uwge_cover` on contract id only, so cover amounts repeat once per insured person (inferred: promo is multiplied by the number of persons). Java checks that a risk person exists, then sums covers once. The PD2 handler applies version filters; the Landscape dispatch has none. So `RP/promoAmt` and the `070406 && promo>0` band can differ. Decision needed: replicate the quirk or treat it as a Jasper bug; verify on a contract with 3 insureds.
- **Common mistakes:** Assuming `$sum` equals Oracle `sum`.
- **Follow-up questions:** How would you prove it? (Run the Oracle query and the pipeline for one contract.)

### Q93. What is the "first version row" issue in Landscape flags?
- **Why the interviewer asks:** Another subtle parity gap.
- **Ideal answer:** Jasper computes `NOMINEE_PRINT` / `DIAGNOSIS_PRINT` with `exists (...)` over WIP and posted rows. Java joins nominee/diagnosis rows to versions and tests `$arrayElemAt(_pv.VERSION_NO, 0)`, so only the first joined version row decides. If several version rows exist the result can differ from "any row matches" (inferred). Test: seed two versions and check.
- **Common mistakes:** Not reading how `pipelineWithPv` works.
- **Follow-up questions:** Better pipeline? (`$match` with `$expr` and `$in`, or `$anyElementTrue`.)

### Q94. How are dates handled in Template12?
- **Why the interviewer asks:** Cross-cutting concern with three formats.
- **Ideal answer:** DTO dates arrive as strings (some with `T00:00:00`); `toLocalDateTime` strips the time and parses; payload fields are `LocalDateTime` with `@JsonFormat dd/MM/yyyy`. At the end S12 reflectively rewrites any String field named like `*date*`/`*dob*` to `dd/MM/yyyy` (written locally because the base class is under a no-touch constraint). Jasper passes `dd-MMM-yyyy` text to sub-reports; REST helpers get `yyyy-MM-dd`. So there are three or four formats. Failure mode: an unparseable value becomes null silently. XSD wants dateTime.
- **Common mistakes:** Saying "XSD fixes display".
- **Follow-up questions:** Time zones? (Not handled; mostly date-only.)

### Q95. Fail-soft or fail-hard? What is the pattern here?
- **Why the interviewer asks:** Design reasoning.
- **Ideal answer:** Most `add*` helpers catch exceptions, `log.warn`, and return neutral data, so one broken sub-report does not lose the whole policy document. Hard failures are deliberate: the PolicyDocument primary on WIP/live conflict, and the Landscape dispatch (test `execute_landscapeDispatchHandlerThrows_notWrappedInTryCatch_propagates`). Cost: silent partial output where Jasper would fail the report. Better: surface a warnings list or a metric.
- **Common mistakes:** Defending swallowing as always right.
- **Follow-up questions:** Which failures should be hard? (Those that control print gates.)

### Q96. Why did you leave some fields unmapped instead of guessing?
- **Why the interviewer asks:** Judgement.
- **Ideal answer:** When no matching `PlineTransDet` leaf exists, an invented mapping would put data in a wrong element and break the renderer silently. Example comment in `addPrimaryDocumentT12`: `showGstInvInd` etc. "no matching leaf ... left unmapped rather than guessing". Also `pMstPolicyNo` and `pContractIdList` are read but not emitted (no home in v0.1.6). Unmapped items are listed so the schema owner can add leaves.
- **Common mistakes:** Silently dropping without a comment.
- **Follow-up questions:** How do you track the list? (Mapping sheet and code comments.)

### Q97. RejectRisk exposes customer name and ID. How do you treat that?
- **Why the interviewer asks:** Data protection awareness.
- **Ideal answer:** The listing returns `NAME_EXT`, ID types and values (PDPA relevance in Malaysia - inferred). Risks here: no authentication, request body printed by `System.out`, response XML logged at INFO on the legacy path, `MongoTemplate` DEBUG logs values. The `Map<String,String>` filter type prevents Mongo operator injection. Actions: authenticate callers, remove prints, consider masking IDs, limit date range and rows. Also `dataClass` (0-3 classification from `RPGE_RPT_TEMPLATE`) is printed on the report header.
- **Common mistakes:** Ignoring logs as a leak path.
- **Follow-up questions:** What is the data classification label? (Public, Internal, Confidential-Named, Strictly Confidential.)

### Q98. What did you learn from this project?
- **Why the interviewer asks:** Growth and depth.
- **Ideal answer:** [CONFIRM: your real list.] Evidence-linked learning: translating SQL semantics to Mongo (outer join, ROWNUM, DISTINCT, UNION, NULL order, type strictness); reading Jasper print conditions; XSD sequence, types and namespace behaviour; Spring Boot 4 / Jackson 3 changes; why mocked-Mongo tests are not enough; and writing honest gap lists. Say the lesson, not just the tool.
- **Common mistakes:** A list of buzzwords.
- **Follow-up questions:** Which lesson changed how you code?

### Q99. How do you answer "did you write this commit?" when the author is someone else?
- **Why the interviewer asks:** Integrity check; resumes often overclaim.
- **Ideal answer:** Be direct: git authors for Plines commits are names such as Shinge, Karunakar, Shaw, GS, Amarnath Yadav, Neethu. My git user (`NISHUNISARGA`) does not appear as author in the history I checked. So I describe my work by task (analysis, mapping, code, tests, XSD) and by what I can explain line by line, and I do not quote hashes I cannot prove. [CONFIRM: whether your work was committed under another account or merged by someone else.]
- **Common mistakes:** Claiming a hash to look stronger.
- **Follow-up questions:** Explain this method for me now, line by line.

### Q100. Why should we pick you, based on this project?
- **Why the interviewer asks:** Closing question; ties project to fit.
- **Ideal answer:** Short and true: I worked on three non-trivial Personal Lines reports in a retrofit; I can read legacy Jasper and Oracle SQL, reproduce its rules in Mongo aggregations and Java, map to an XSD, and test the branch logic. I am honest about gaps (XSD not validated end to end, renderer contract unconfirmed, ASP expansion unknown) and I know the fixes. I keep secrets out of commits and write tests that name business rules. [CONFIRM: add one personal strength with a real example.]
- **Common mistakes:** Claiming to be an expert in Mongo or Spring after 1.6 years; claiming production metrics.
- **Follow-up questions:** What is your weakest area in this project? What do you want to learn next?
