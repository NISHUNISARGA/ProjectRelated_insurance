# 05 - Testing Deep Dive (Template12, ASP, RejectRisk in opus-documentprintdata-service)

Conventions:
- EVIDENCE = read in a file / git output (path given).
- INFERENCE = my reasoning, labelled.
- I did NOT run Maven or any test. All statements about pass/fail are from reading code and files.
- `REPO` = `C:\Users\EPGRVFY\Documents\Plines_Base_code\New_branch\2\opus-documentprintdata-service`
- Test root: `REPO\src\test\java\com\agcs\fo\opus\plines\` (sub-folders `documentHandler`, `documentdto`, `requestpayload`, `service`, `client`, `config`, `utils`, plus `PlineXsdValidationTest.java`).
- Counting method: `git grep -c "@Test"` on the working tree for file names matching `Template12`, `Asp` or `RejectRisk`. `@ParameterizedTest` methods are counted separately (14 hits in 6 of these files). Counts can include a commented `@Test`; I did not check for that.

---

## a) Test inventory

### a.1 Totals
| Scope | Files | `@Test` count |
|---|---|---|
| Whole repo `src/test` | 1916 files with `@Test` (1932 files under src/test incl. 2 XSDs, 1 MockMaker file, 1 misplaced class) | 19,665 |
| Plines tests (all) | 1386 files under `src/test` match `plines` | not summed |
| Template12 + ASP + RejectRisk (name filter) | 91 | 865 |

Breakdown of the 91 files (EVIDENCE: `git grep -c`):
| Layer | Files | `@Test` | What they test |
|---|---|---|---|
| `documentHandler` | 32 | 246 | Mongo aggregation handlers (mocked `MongoTemplate`) |
| `documentdto` | 32 | 224 | Lombok DTO classes |
| `requestpayload` | 19 | 113 | Lombok payload classes (`*Details`) |
| `service` | 8 | 282 | `ReportStrategy` services (orchestrate handlers, build payload) |

Share of the low-value kind: DTO + payload = 337 of 865 = about 39 percent of the tests in scope are tests of Lombok-generated getters/setters/constructors (INFERENCE on "low value", the count is evidence).

### a.2 Service tests (the most valuable ones)
| Path (under `...\plines\service\`) | `@Test` | Focus |
|---|---|---|
| `Template12PolicyDocumentBaseServiceTest.java` | 135 | shared mapping helpers used by the Template12 services (`mergeSingleTransDet_*`, `addTemplate12Coverage_mapsRow`, `toStr_*`, `toBigDecimal_*`) |
| `Template12PolicyDocumentServiceTest.java` | 34 | `execute_*` per product/scheme branch |
| `Template12PolicyDocument2ServiceTest.java` | 33 | second Template12 policy document service |
| `Template12LandscapeServiceTest.java` | 28 | landscape "bands 1-16" gates |
| `AspEndtDocumentServiceTest.java` | 28 | ASP endorsement document, many gates |
| `AspEndtDocument2ServiceTest.java` | 12 | ASP endorsement document 2 |
| `AspEndtDocumentLandscapeServiceTest.java` | 8 | ASP landscape rows |
| `RejectRiskServiceTest.java` | 4 | RejectRisk payload mapping |

Real method names (EVIDENCE, grep of `void ...(`):
- RejectRiskService: `execute_emptyRows_returnsEmptyReportDataList`, `execute_populatedRow_mapsFieldsAndReturnsOneReportDataPerRow`, `execute_reportParameters_nullTitlesDefaultToEmptyString`, `execute_createdDateFromTo_parsedAsLocalDateTime`.
- Template12PolicyDocumentService: `execute_nullFilterCondition_defaultsAndReturnsValidTransactions`, `execute_primaryDocumentNull_defaultsAppliedNoCrash`, `execute_atcProduct070406_mapsAtcFieldsOntoReportParameters`, `execute_astroScheme070504_mapsAstroFields`, `execute_bmwScheme070504_mapsFirstCertRowAndRunsNestedSubreportsPerRow`, `execute_poscoverSchemeWithDependants_addsLandscapePosReportDataRows`, `execute_pa4aProduct070414_mapsPolicyNoAndRunsPerRowNestedSubreports`, `execute_showOutPrintCovLetterY_usesOutSourceCoverLetterBranch`, `execute_parentNamePrecedence_policyDocumentWinsOverLandscapeAndPolicyDocument2`, `execute_pd2DebitNoteT4Product070406_usesT4Handler`, `execute_pd2DebitNoteSkippedForBmwScheme`, `execute_ciPrintPrimaryHandlerThrows_swallowedAndTransactionsStillReturned`, `execute_landscapeDispatchHandlerThrows_notWrappedInTryCatch_propagates`.
- Template12LandscapeService: `execute_bands1to3_aspProductCutoffN_populatesInsuredPremiumSponsorRows`, `execute_band4_excludedFor070504_doesNotFire`, `execute_band9_excludedForPoscoverAzp_doesNotFire`, `execute_band12_excludedFor100908_doesNotFire`, `execute_band16_promoAmtZero_doesNotFire`, `toLocalDateTime_string_nullBlankValidAndInvalid`, `toBigDecimal_variousInputs`.
- AspEndtDocumentService: `minimalDefaults_skipsAllOptionalSubReportsButRunsLandscapeDispatch`, `endtCode72_fetchesScheduleOfBenefitsAndLlbAndSti`, `endtCodeOther_skipsScheduleOfBenefits`, `showDebitNote_withZeroPrem_doesNotFetchDebitNotes`, `showServiceTax_checklistCopyY_closesGate`, `showServiceTax_staxCountNotAboveOne_skipsEndtMTAndSub`, `landscapeDeletionGate_closed_skipsDeleteMemberHandler`, `allGatesOpen_oneReportDataPerSubreportInSubreportOrder`, `execute_allGatesOpen_fullyPopulatedRows_mapsEveryField`.
- AspEndtDocument2Service: `issueDateAsUtilDate_isFormattedForTransitionalInvoice2`, `issueDateAsLocalDate_isFormattedIntoTransDetIssuedate`, `issueDateNull_yieldsEmptyFormattedString`, `reportParameters_capturesAllFilterEchoes`.

### a.3 Handler tests (examples)
| Path (under `...\plines\documentHandler\`) | `@Test` |
|---|---|
| `AspServiceTaxInvoiceHandlerTest.java` | 27 (e.g. `fetchServiceTaxInvoice_ambiguousEinvCutoff_throwsIllegalStateException`, `..._conflictingLiveAndWipVersionNo_throwsIllegalStateException`, `..._mstContractIdProvided_idValue1Blank_fallsBackToIdValue2`) |
| `AspDebitNoteHandlerTest.java` | 18 |
| `PlineTemplate12NoteHandlerTest.java` | 17 (e.g. `fetchNote_multipleUpbRowsSameOpb_throwsIllegalStateException_evenIfNameExtValuesAgree`, `fetchNote_numericContractId_matchedAsLong`) |
| `RejectRiskHandlerTest.java` | 15 |
| `PlineT12ParticularOfInsuredLandscapeAspHandlerTest.java` | 14 |
| `PlineTemplate12RiskDetailsHandlerTest.java` | 11 |
| `AspAddMemberLandscapeDispatchHandlerTest.java` | 11 |
| `AspEndtScheduleHandlerTest.java` | 9, `PlineTemplate12CoverageHandlerTest` 9, `PlineTemplate12InstallmentDetailsHandlerTest` 9 |
| `AspEndtCoverHandlerTest` 8, `AspParticularOfSponsorLandscapeHandlerTest` 8, `PlineTemplate12SignatureHandlerTest` 8, `AutoRenewalLetterToPolicyholderAspEgHandlerTest` 8 | |
| smaller | `PlineAspLlbHandlerTest` 1, `PlineAspStiHandlerTest` 1, `PlineEndtScheduleOfBenefitsAspHandlerTest` 1 ... |

RejectRiskHandlerTest method names (EVIDENCE): `fetchRejectRisks_dateInstances_usedDirectlyInMatch`, `..._dateStrings_parsedViaDateUtils`, `..._nullDates_matchStageHasNullBounds`, `..._noOptionalFilters_noSecondMatchStageAdded`, `..._rejTypeOnly_addsSecondMatchStage`, `..._reasonCodeOnly_addsSecondMatchStage`, `..._rejTypeAndReasonCode_bothInSameMatchStage`, `..._customerNameProvided_addsMatchOnCustName`, `..._idNoProvided_addsOrMatchOnIdValues`, `..._customerNameAndIdNoAbsent_noExtraMatchStages`, `..._cmgeCodeLookups_targetExpectedCatCodes`, `..._rowNumber_usesSetWindowFieldsSortedByCustomerName`, `..._finalSortStage_sortsByCustomerNameAscending`, `..._pipelineLambdas_fullyForced`, `..._returnsMappedResults`.

### a.4 DTO / payload tests
Same recipe in every file (EVIDENCE: `PlineRejectRiskDtoTest.java`, `PlineRejectRiskDetailsTest.java`): `noArgsConstructor_createsInstanceWithNullFields`, `allArgsConstructor_setsAllFields`, `settersAndGetters_workCorrectly`, `equalsAndHashCode_sameValues_areEqual`, `equalsAndHashCode_differentValue_notEqual`, `toString_containsFieldValues`. Six tests per class, about 100-140 lines each.

### a.5 Other related tests
- `PlineXsdValidationTest.java` (119 lines): 6 tests. `minimalShellXml_isValid` and `knownInvalidXml_failsValidation` really prove the harness. The four `template4LandscapeService_generatesValidXml`, `template12LandscapeService_generatesValidXml`, `template12PolicyDocumentService_generatesValidXml`, `template12PolicyDocument2Service_generatesValidXml` are PLACEHOLDERS: they validate the same hand-written `MINIMAL_VALID_XML` string and never call a service (file lines 91-118, with a comment saying to replace them). They give a false sense of XSD coverage. There is no ASP or RejectRisk XSD test.
- Resource: `src/test/resources/mockito-extensions/org.mockito.plugins.MockMaker` = `mock-maker-inline` (lets Mockito mock final classes/static-like patterns).
- Defect: `REPO\src\test\java\...\documentHandler\AspServiceTaxInvoiceHandler.java` (483 lines, NO `@Test`) is a near-copy of the production class `src/main/.../AspServiceTaxInvoiceHandler.java` (476 lines) living in the TEST source tree with the SAME package and class name. Differences (after ignoring line endings): 17 changed lines, such as `catch (Exception e)` vs `catch (RuntimeException e)` in `safeCall`, `SafeCallable.call() throws Exception`, and comment edits (matches the Sonar fix style of commit `24e50ab9b`). INFERENCE: the test classpath puts `test-classes` before `classes`, so `AspServiceTaxInvoiceHandlerTest` (27 tests) probably exercises the copy, not the real production class; and JaCoCo (which reads `target/classes`) would show that production class as less covered than the tests suggest. Fix: delete the copy and run the tests against the real class. Not verified by running.

### a.6 Frameworks and mocking style (EVIDENCE, counts over the 91 files)
| Item | Result |
|---|---|
| JUnit 5 (Jupiter) | all. `pom.xml` l.171-176 declares `junit-jupiter` 5.9.0 explicitly, plus `spring-boot-starter-test` (l.81) and `mockito-core` (l.184) |
| Mockito | `@ExtendWith(MockitoExtension.class)`; `@Mock` 352 uses; `@InjectMocks` in 35 files |
| Lenient | `@MockitoSettings(strictness = Strictness.LENIENT)` in 36 files (so unused or wrong stubs are never reported) |
| AssertJ | `org.assertj` imported in 40 files; `assertThat(` 847 uses |
| JUnit assertions | `org.junit.jupiter.api.Assertions` in 51 files; about 2,365 `assertEquals/Null/True/False/NotNull` calls (mostly DTO/payload tests) |
| `ArgumentCaptor` | 0 uses |
| `MockedStatic` / `@SpringBootTest` / Testcontainers / embedded Mongo | 0 in these files. Across the whole `src/test` only 3 files reference such things (`OpusMotorGatewayApplicationTests`, two motor service tests) |
| `verify(` | 116 uses in only 9 files, so most tests check outputs, not interactions |
| Exception checks | `assertThatThrownBy/assertThrows`: 7 uses in 4 files |
| Parameterized | `@ParameterizedTest`: 14 hits in 6 of the 91 files (100 hits over all plines tests) |
| `@Nested`, `@DisplayName` | none in these files |
| Mongo mocking style | `mongoTemplate` is a `@Mock`; stubs on `aggregate(any(Aggregation.class), any(String.class), any(Class.class))`, `find`, `count(any(Query.class), eq("COLLECTION"))`. For pipeline checks the tests capture the `Aggregation` with `doAnswer` and call `toPipeline(Aggregation.DEFAULT_CONTEXT)` (116 uses in 23 files) |
| Helper | `forcePipelines(captured)` exists in about 110 plines test files; its comment says "only need lambda bodies to execute for coverage" (RejectRiskHandlerTest l.49-57) |

### a.7 Naming convention
`methodUnderTest_condition_expectedResult`, e.g. `fetchServiceTaxInvoice_staxCount_refNoOnly_usesWipCount`, `execute_band9_excludedForPoscoverAzp_doesNotFire`. Service tests are named after the gate or product code they cover (`070412`, `070504`, `100908`). Good for readers; they act as a spec of the business rules.

### a.8 What is covered and what is not
Covered (EVIDENCE from names):
- Payload field mapping for each service, per branch (product codes, schemes, "bands").
- Gate logic: sub-report fetched or skipped (`showDebitNote_withZeroPrem_doesNotFetchDebitNotes`).
- Null / empty / exception-swallowing behaviour (`..._handlerThrows_isSwallowed`, `..._lookupThrows_warnsAndContinues`).
- Date/number helpers (`toLocalDateTime_*`, `toBigDecimal_*`).
- Pipeline structure of Mongo aggregations (stage counts, presence of `$lookup`, `$match`).

Not covered (INFERENCE from absence; I searched the test names/imports):
- Real MongoDB behaviour. Nobody checks that a pipeline returns the right rows (null sort order, `$lookup` semantics, date boundaries, type mismatches). The RejectRisk NULL-ordering and time-zone issues (see file 01) could not be caught.
- Comparison with the original Jasper/Oracle output (no golden files, no sample data in the repo).
- XSD validity of real service output (placeholders only; RejectRisk's `pCreatedDateFrom/To` are not in the test-side XSD).
- The renderer contract (`reportData[]` as separate print units: the disclaimer in `RejectRiskService` l.27-32).
- Concurrency, performance, large-data behaviour.
- Controllers / HTTP layer for these reports (not looked at in this task).
- After the developer's uncommitted RejectRisk fix: 7 of 15 `RejectRiskHandlerTest` tests are expected to fail (details in file 01, section 4.1).

---

## b) How the tests were generated (commits and quality)

| Commit | Date | Author (as in git) | Message (short) | Size / content (EVIDENCE: `git show --shortstat`) |
|---|---|---|---|---|
| `0ab953cf8` | 2026-09-25 | Kumari Soni | OPUSM-44159 Add JUnit5 tests for documentdto batch (Asp/Auto/Coverage/Endt/Fw/Ghs/...) | 135 files, +22,089; all DTO tests (about 163 lines per file) |
| `df70d6b5c` | 2026-09-25 | extern.neethunivas-subrahmannian_neethu | ... Alp/Asp documentHandler batch B | 13 files, +2,097 (about 160 lines per file) |
| `8fa89231e` | 2026-09-25 | same | ... Asp/AutoRenewal documentHandler batch C | 13 files, +2,051 |
| `3a3bba346` | 2026-09-25 | same | ... AutoRenewal letters/documents documentHandler batch D | 13 files, +1,416 |
| `3f2157b28` | 2026-09-29 | Shaw | OPUSM-44159 Test cases added for plines | 1,486 files, +203,271 / -4,913. About 1,237 are `src/test` files (documentHandler 527 + documentdto 413 + requestpayload 307 + service 84 path-hits include main too). The remaining files are production changes (e.g. `AcpDebitNoteHandler`, `AhiEndtDocumentMainHandler`, `RejectRiskHandler`, `PlineRestTemplateConfig`) |
| `db7588897` | 2026-09-30 | Karunakar | OPUSM-44159 Add test coverage for plines.service/client/config/utils and SharedReport | 156 files, +59,697; all under `src/test` (141 service tests, 4 requestpayload, 4 client, 3 documentdto, others) |
| `72d1bdf32` | 2026-09-30 | extern-karunakar-shravya | OPUSCCC/test-coverage/plines-service-and-misc-latest ... | same shortstat as `db7588897`, no file list with `--name-only` -> looks like a merge commit bringing the same work in (INFERENCE) |

Related: `24e50ab9b` (2026-09-29, Shaw) "Sonar build and error fixes"; `0b43bc07a` "Document handler test cases part3" (2026-09-28). So the test base was built in about 6 days (09-25 to 09-30) by at least 5 different authors, in "batches" by package. Ticket for all: OPUSM-44159.

Patterns:
- DTO tests: fixed 6-test template (a.4). Payload tests: same template. Values are the field name as a string ("customerName") or small constants.
- Handler tests: `@Mock MongoTemplate`, `@InjectMocks` handler, lenient strictness, one test per branch, assertions on resulting DTO or on the captured pipeline (`toPipeline`).
- Service tests: mock every handler, one big `setUp`, `execute_*` tests per product branch, helper methods like `reportDataList(t)` to dig into the payload.

Generated-code smells (EVIDENCE for each; whether a tool wrote them is INFERENCE):
1. Getter/setter/equals/hashCode/toString tests for Lombok classes (337 tests in scope). They only prove Lombok works and inflate line coverage.
2. "Coverage-driven" tests: `fetchRejectRisks_pipelineLambdas_fullyForced` and `forcePipelines` run the lambda bodies and assert nothing (comment says so).
3. Tests tied to stage positions: `pipeline.get(1).get("$match")` breaks when a stage is added or removed (as the RejectRisk fix will show).
4. Lenient mocks everywhere hide dead stubs.
5. Batch commit messages with the package list ("batch B/C/D") and identical structure across files. The repo does not say whether AI tools were used. Do not claim it; say "consistent with template or tool-assisted generation".
6. Placeholders presented as tests (`PlineXsdValidationTest` four service tests).
7. A production class pasted into the test tree (`AspServiceTaxInvoiceHandler.java`).
8. Mixed commit: `3f2157b28` mixes test additions with 200+ production file edits, which makes review and bisect hard.

Quality assessment (my opinion, labelled INFERENCE): good for protecting against accidental refactor breaks in the service layer (branch/gate logic is described by real scenario names), weak for proving correctness against the original Jasper reports. Good-faith unit tests, not a regression suite.

---

## c) Coverage tooling

EVIDENCE from `REPO\pom.xml`:
- Properties (l.44-47): `sonar.junit.reportPaths=target/surefire-reports`, `sonar.java.coveragePlugin=jacoco`, `sonar.coverage.jacoco.xmlReportPaths=target/site/jacoco/jacoco.xml` (also a leftover `sonar.javascript.lcov.reportPaths`).
- Plugin (l.227-246): `jacoco-maven-plugin` 0.8.12; goals `prepare-agent` and `report` bound to the `test` phase. No `check` goal, no minimum-coverage rule, no exclusions (so DTO/payload tests count toward coverage).
- The `sonar-maven-plugin` entry is commented out in the pom (l.222-226); Sonar is invoked from the workflow instead.
- `maven-surefire-plugin` has no custom config in the pom (default includes `*Test.java`, so the misplaced `AspServiceTaxInvoiceHandler.java` is compiled but not run).

CI (EVIDENCE: `REPO\.github\workflows\`):
| Workflow | What it shows |
|---|---|
| `sonar-code-review.yml` ("Code Review(Sonar)") | on push to `retrofit-develop-R2`, `retrofit-develop-R3`; on pull_request to `retrofit-develop-R1`, `retrofit-develop-R2`. Java 17 Temurin. Runs `mvn -B verify org.sonarsource.scanner.maven:sonar-maven-plugin:sonar -Dsonar.projectKey=... -Dsonar.qualitygate.wait=true`. So tests run, JaCoCo XML is produced, Sonar reads it, and the job waits for the quality gate. The gate thresholds live on the Sonar server, not in the repo. |
| `cicd-workflow.yml` | manual (`workflow_dispatch`); calls a shared action `opus-github-cicd-workflows/actions/java-microservice` with Sonar and Wiz secrets. The shared action is not in this repo, so I cannot say if it runs tests. |
| `codeql.yml`, `secret-scan.yml`, `secret-scan-nightly.yml` | security scanning (names only; I did not read inside) |
| `jira-commit-check.yml` | PR commit must link a JIRA ticket (matches `OPUSM-xxxxx` prefixes) |
| `changelog-update.yml` | CHANGELOG automation |
| `claude-pr-review.yml` | AI-assisted PR review (name only) |

How coverage is consumed: the Sonar quality gate on PRs. The 5-day test push (`09-25` to `09-30`) and the "Sonar build and error fixes" commit suggest the tests were written to satisfy that gate (INFERENCE).

Coverage numbers present in repo: NOT AVAILABLE. `REPO\target` contains only `classes` and `generated-sources`. There is no `target\site\jacoco`, no `jacoco.csv/xml/exec`, no `surefire-reports`. So I cannot give coverage for Template12, ASP or RejectRisk.

---

## d) Do tests compile and pass today?

- No `target\surefire-reports` and no `target\test-classes`. So there is no recorded test run in this working copy. `target\classes` has compiled main classes (`RejectRiskHandler.class`, `RejectRiskService.class`, last written Oct 8, 11:51), so main code compiled at least once there.
- Status: UNKNOWN for tests. Reasoned expectations (INFERENCE):
  1. Compile: a duplicate class (`AspServiceTaxInvoiceHandler`) exists in main and test with the same name. The Maven compiler normally allows this (test compile sees main classes as a dependency), so it should still compile.
  2. `pom.xml` pins `junit-jupiter` to 5.9.0 while the parent is Spring Boot 4.1.x (pom comments l.40-76). I believe Boot 4 manages a newer JUnit platform; a manual old version could cause a version mismatch warning/failure. Not verified.
  3. RejectRisk: with the uncommitted handler change 7 of 15 handler tests should fail (file 01, 4.1). `RejectRiskServiceTest` should pass.
  4. `AspServiceTaxInvoiceHandlerTest` result depends on which class wins on the classpath.
- To know for sure, someone must run `mvn -B verify` (I was told not to).

---

## e) Testing strategy: evidence vs assumption

| Level | Evidence in repo | Assumption (not proven) |
|---|---|---|
| Unit | 19,665 `@Test` annotations; plines unit tests with mocked `MongoTemplate`/handlers; JaCoCo + Sonar | Developers run `mvn test` locally before PR |
| Component / integration | None for Mongo (no Testcontainers, no embedded Mongo). Only 1 Spring context test in motor (`OpusMotorGatewayApplicationTests`). Pom comment l.70: a micrometer version once "broke ApplicationContext startup for every @SpringBootTest", so Spring context tests existed or were tried | Integration is done by calling the deployed service in a dev environment |
| Contract / schema | `PlineXsdValidationTest` (harness real, service checks are placeholders); XSD kept in `src/test/resources/xsd/plines` (header says v0.1.6) | The XSD is the contract with the report/render team; sign-off is manual (the disclaimer in `RejectRiskService`) |
| SIT | Not in repo | Likely a shared environment where OPUS calls this service (INFERENCE only) |
| UAT | `CHANGELOG.md` line 13: "2026-09-21 OPUSM-52530 Reverted aggregation and lookup code changes unable to reproduce on BAU UAT (PR #864)" - proves a UAT environment exists and that UAT found issues that unit tests missed. Line 20: OPUSM-48783 "Code refactor and tests for getPolicyScheduleTrSpJrDetailsAsync (PR #863)" | UAT is business-owned |
| Regression | Unit suite acts as a regression net for payload mapping. `ARCHITECTURE.md`/README not read for this task | A Jasper-vs-new-service comparison happens manually; branch names like `OPUSM-44159-retrofit-develop-pline-jrxml-comparison-fixes` suggest JRXML comparison work (INFERENCE from branch names only) |
| End to end | Not in repo | Would be: OPUS request -> this service -> XML -> report renderer -> PDF |
| Static analysis / security | Sonar, CodeQL, secret scan, Wiz (CVE comments in `pom.xml` mention "Wiz scan") | n/a |
| Review | `claude-pr-review.yml`, CODEOWNERS, JIRA link check | n/a |

Key message: unit tests prove the code does what the author intended. They do not prove it matches the old Jasper/Oracle report. That proof needs data-level comparison (same filter, same rows, same order) and is not in the repo.

---

## f) 30 testing interview questions (about 1.6 years experience level)

Format: Q, strong answer, common mistake, follow-up.

1. **What is a unit test vs an integration test? Which are these?** Unit = one class, dependencies mocked, fast. Integration = real collaborators (e.g. real Mongo). Here almost all are unit tests with a mocked `MongoTemplate`. *Mistake:* calling them integration tests because they use Spring classes. *Follow-up:* what would an integration test for RejectRisk need? (seeded Mongo).
2. **What does `@InjectMocks` do?** Creates the class under test and injects `@Mock` fields (constructor first). Used in 35 files. *Mistake:* thinking it creates mocks. *Follow-up:* what if there are two constructors?
3. **What does `Strictness.LENIENT` change?** Mockito will not fail on unused stubs. Used in 36 files. Pro: fewer false failures with shared `setUp`. Con: dead stubs stay hidden. *Follow-up:* how would you find dead stubs?
4. **Why mock `MongoTemplate`?** Speed, no DB needed. Cost: you do not test query correctness. *Mistake:* believing a green test means the query works.
5. **How do you test an aggregation pipeline without a DB?** Capture the `Aggregation`, call `toPipeline(Aggregation.DEFAULT_CONTEXT)`, assert on the `Document` stages. *Follow-up:* limitation? Tests depend on stage order.
6. **Why did RejectRisk handler tests break after the date fix?** They assert `$lte`, null bounds and fixed stage indexes. After the fix the date `$match` uses `$lt` and may be absent, so indexes shift. *Lesson:* assert by content (find the stage that has `CREATED_DATE`) not by index.
7. **What is `ArgumentCaptor` and is it used?** It captures arguments passed to a mock for assertion. 0 uses in these files; tests use `doAnswer` to capture instead. *Follow-up:* how would you rewrite with a captor?
8. **`verify` vs state assertions?** `verify` checks interactions, `assertThat` checks results. Here `verify(` appears in 9 of 91 files. Prefer result checks, use `verify` for "must not call" cases like `showDebitNote_withZeroPrem_doesNotFetchDebitNotes`.
9. **What is AssertJ and why use it over JUnit asserts?** Fluent, readable messages (`assertThat(list).hasSize(2)`). Both are used here (AssertJ 847 calls, JUnit about 2,365).
10. **Explain the test naming convention in this repo.** `method_condition_expected`. Example: `execute_band9_excludedForPoscoverAzp_doesNotFire`. *Follow-up:* benefit? Reads like a rule.
11. **Is testing Lombok getters/setters useful?** Little. It proves the library works and raises coverage numbers. Better to test logic. 337 of 865 tests in scope are of this type.
12. **What does line coverage not tell you?** Lines executed, not behaviour verified. `fetchRejectRisks_pipelineLambdas_fullyForced` executes lambdas with no assertion.
13. **How is JaCoCo wired here?** `prepare-agent` before tests, `report` at `test` phase; Sonar reads `target/site/jacoco/jacoco.xml` (pom l.44-47, 227-246). No minimum rule in the pom.
14. **What is a quality gate?** Sonar rule set (coverage, bugs, smells) that can fail the PR. Workflow uses `-Dsonar.qualitygate.wait=true`. Thresholds are on the Sonar server (not in repo).
15. **How would you test the "no dates" case for RejectRisk?** Unit: pipeline has no `CREATED_DATE` `$match`. Integration: seed 3 records and check results. Also ask what Oracle did (0 rows).
16. **How would you test the whole-day boundary?** Seed `D 00:00:00`, `D 23:59:59.999`, `D+1 00:00:00` and query from=to=D; expect first two only.
17. **How would you test the time zone issue?** Seed a record at `2026-01-04T16:30:00Z` (= 2026-01-05 00:30 Malaysia) and query day 2026-01-05; decide which answer is correct from the business and assert it.
18. **How do you test exception paths?** Stub with `thenThrow`, assert behaviour: swallow (`execute_ciPrintPrimaryHandlerThrows_swallowedAndTransactionsStillReturned`) or propagate (`execute_landscapeDispatchHandlerThrows_notWrappedInTryCatch_propagates`). 26 throw-stubs in scope.
19. **What is a parameterized test and where would it help?** One method, many inputs (`@ParameterizedTest` with `@CsvSource`). Good for `DateUtils.toLocalDate` formats and product-code gates. 14 uses in scope.
20. **What is `mock-maker-inline`?** Mockito setting that allows mocking final classes and more. File in `src/test/resources/mockito-extensions`.
21. **How do you validate XML against an XSD in a test?** `SchemaFactory` + `Validator` on a real service output. The repo's helper exists but four of its tests validate a fixed string. Fix: build a `Transactions` via the service and marshal it first.
22. **What is wrong with a test class named like production code in `src/test`?** `AspServiceTaxInvoiceHandler.java` shadows the real class on the test classpath; tests may not test production code and coverage can mislead.
23. **Why is mixing tests and production changes in one commit risky?** `3f2157b28` has +203k lines; reviewers cannot see which production changes are intended. Prefer separate commits.
24. **How do you know a test is good?** It fails when behaviour breaks. Try mutation: change `$lt` to `$lte` and see if any test fails.
25. **What is a flaky test and a risk here?** Test that sometimes fails. Risks: time zone/date-dependent tests (`new Date()`), JVM default zone in `Date.toString()` parsing. Use fixed clocks/zones.
26. **How do you decide what to mock in a service test?** Mock handlers (I/O), keep payload classes real. This repo does exactly that.
27. **What is regression testing for a report migration?** Run old and new for the same inputs, compare rows, order, formatting, null display ("NA"). Not present in the repo; would be added as golden-file or DB-comparison tests.
28. **SIT vs UAT?** SIT = system integration by the delivery team across systems. UAT = business users validate. `CHANGELOG` line 13 shows a UAT-found issue was reverted. (Definition is general knowledge; the environment names here are unknown.)
29. **A test passes locally but fails in the Sonar workflow. What do you check?** JDK (Java 17 in workflow), time zone/locale, test order dependence, memory (`-Xmx6g` in workflow), file paths (Windows vs Linux; the XSD test loads from disk), secrets or profiles missing.
30. **What would you improve first in this test suite?** (1) Delete the duplicated class in `src/test`. (2) Fix RejectRisk tests and assert by content. (3) Add Mongo integration tests with seed data for boundary/null cases. (4) Replace XSD placeholders with real output validation. (5) Reduce LENIENT usage. (6) Add golden comparisons vs Jasper.

---

## Uncertainties
1. No test run evidence (no surefire reports, no JaCoCo files, no `target\test-classes`), so compile/pass/coverage are unknown. All "will fail" statements are static reading.
2. Counts are `@Test` text occurrences by name filter (`Template12|Asp|RejectRisk`); some ASP-related or Template12-related tests with other names (for example `AutoRenewal*Asp*` are included, but anything not containing those words is not) may be missing.
3. Whether the tests were AI-generated is not stated anywhere; I only describe patterns.
4. The classpath ordering claim for the duplicated `AspServiceTaxInvoiceHandler` is a standard Maven expectation, not verified here.
5. The shared CI action (`opus-github-cicd-workflows`) is outside this repo; I do not know if it runs tests or what coverage threshold the Sonar server enforces.
6. JUnit version compatibility (5.9.0 pin with Spring Boot 4.1.x) is a suspicion, not verified.
7. The SIT/E2E layers are not visible in this repo; the table marks them as assumptions.
8. Author names are as printed by git (some look like external/contractor accounts); no inference about roles is made.
