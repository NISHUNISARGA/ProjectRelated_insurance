# 09d - Debugging Questions 51-100 (symptom -> how you investigate)
> **AUTHORSHIP UPDATE (confirmed by the developer): git author `GS <extern.gs_nisarga@allianz.com>` is the developer. Wherever this file says commits cannot be attributed to the developer, read it as: the 11 GS commits listed in `KT_Parts/14_My_Commits_And_Ownership.md` ARE theirs; commits by other authors still must not be claimed. No GS commit touches `src/test`, so test authorship is still `[CONFIRM]`.**

Same rules as 09c. "(inferred)" = my reasoning or standard behaviour that was not run on this repo. No real incident tickets exist in the repo; answers describe a method, not a personal story. Use [CONFIRM: ...] only for a case you really debugged. No git authorship is attributed to you. Secrets are <redacted>.

---

## J. Build, upgrade and test-run symptoms

**Q51. Symptom: after updating Swagger/springdoc the app fails at startup with `NoClassDefFoundError` for `WebMvcProperties`. Investigate.**
- Why the interviewer asks: Boot 4 relocation of classes; the repo has this exact history.
- Ideal answer: The pom comment records that springdoc 2.8.15 broke on Boot 4 because `WebMvcProperties` was relocated, and 3.1.1 is the fixed version. Steps: read the first cause in the startup stack trace; run `./mvnw dependency:tree` and check the springdoc version against the Boot BOM; check that no other module pins an older springdoc. The fix is the version bump, not an exclusion. Commit family `65f1a8711`/`c6b6803f3` (Boot 4 fixes). I should present this as repo history, not my memory. [CONFIRM: if you were part of it.]
- Common mistakes: Adding the missing class manually.
- Follow-up questions: What else is tied to Boot 4 packages? (`MongoProperties` moved to `org.springframework.boot.mongodb.autoconfigure`.)

**Q52. Symptom: every `@SpringBootTest` fails at context start with an error mentioning `JvmMemoryMeterConventions`. Investigate.**
- Why the interviewer asks: Version pin conflict.
- Ideal answer: The pom comment says an old Micrometer pin `1.15.12` broke the `ApplicationContext` for every `@SpringBootTest`; the pin was removed and the BOM's `1.17.1` is used. Steps: `mvn dependency:tree | grep micrometer`; look for explicit versions in the pom or a parent; remove or align the pin. Rule: when a Boot BOM manages a library, a hand pin must be newer or equal, otherwise you downgrade it.
- Common mistakes: Re-pinning the older version to "fix" another CVE.
- Follow-up questions: Which other pins sit below the BOM? (`commons-lang3` 3.18.0 vs 3.20.0; `logback` 1.5.34 vs 1.5.38.)

**Q53. Symptom: test compile fails with "cannot find symbol MockBean" or `fromHttpUrl` after upgrade.**
- Why the interviewer asks: Known API removals.
- Ideal answer: Known replacements recorded in the KT: `@MockBean` -> `@MockitoBean`; `UriComponentsBuilder.fromHttpUrl` -> `fromUriString`; `getRawStatusCode()` -> `getStatusCode().value()`; explicit `MediaType` imports because of a JUnit 6 collision; an explicit `WebClient.Builder` bean. Steps: read the compiler message, find the removed API in the Spring 7 migration notes, replace. Do not mix tests and main changes in one huge commit (`3f2157b28` mixed them, which makes bisecting hard).
- Common mistakes: Suppressing errors with older dependencies.
- Follow-up questions: Why is `junit-jupiter 5.9.0` suspicious next to a Boot 4.1.x parent? (Effective engine probably 6.0.3; not verified.)

**Q54. Symptom: the request body arrives but `filterCondition` is empty or `@JsonProperty` names are ignored.**
- Why the interviewer asks: Jackson 2 vs 3.
- Ideal answer: Boot 4.1.1 defaults to Jackson 3 (`tools.jackson`), which ignores `com.fasterxml.jackson` annotations used by `PayLoadRequest` and the DTOs. The project forces `spring.http.converters.preferred-json-mapper=jackson2`. Steps: check that the property is present in the effective configuration; check which converter handles the request (debug logs of `HttpMessageConverter` selection - inferred); check that no new dependency brought a Jackson 3 converter first. Also `JacksonConfig` defines a plain `ObjectMapper` with `JavaTimeModule` which ignores `spring.jackson.*`.
- Common mistakes: Changing DTO annotations first.
- Follow-up questions: Which mapper writes the XML? (`new XmlMapper()` from `com.fasterxml.jackson.dataformat.xml` inside `CommonController`.)

**Q55. Symptom: a CVE fix for Jackson "does not take effect" - `dependency:tree` still shows the old version.**
- Why the interviewer asks: Property meaning change.
- Ideal answer: In Boot 4 `jackson-bom.version` refers to Jackson 3. For Jackson 2 the property is `jackson-2-bom.version` (the pom uses `2.21.5`). Setting the old name silently changes nothing. Steps: run `mvn dependency:tree -Dincludes=com.fasterxml.jackson.core`; compare with the pom property; fix the name; re-scan.
- Common mistakes: Adding explicit dependency versions for each module.
- Follow-up questions: What other relocated property did the upgrade have? (`spring.mongodb.*`.)

**Q56. Symptom: tests pass on the developer machine but fail in the Sonar workflow. Investigate.**
- Why the interviewer asks: Environment differences.
- Ideal answer: The workflow (`sonar-code-review.yml`) uses JDK 17 Temurin and `mvn -B verify ... -Dsonar.qualitygate.wait=true`. Compare: Java version, time zone and locale (tests using `Date.toString()` parsing or `new Date()` depend on the JVM zone), heap (the workflow mentions a large `-Xmx`), file paths (the XSD test loads a file; Windows vs Linux), missing secrets/profiles, test order. Reproduce with `mvn -B verify` on Java 17 with `-Duser.timezone=UTC`.
- Common mistakes: Re-running CI until green.
- Follow-up questions: Which test classes use dates? (RejectRisk service `execute_createdDateFromTo_parsedAsLocalDateTime`, `Template12LandscapeServiceTest.toLocalDateTime_string_nullBlankValidAndInvalid`.)

**Q57. Symptom: coverage is high but a change to production code breaks nothing in tests. How do you check the quality of the tests?**
- Why the interviewer asks: Mutation mindset.
- Ideal answer: Manually mutate: change `$lt` to `$lte` in `RejectRiskHandler`, `"72"` to `"73"` in the ASP gate, or remove the `checklistCopy` check, and see whether any test fails. 337 of 865 in-scope tests are Lombok getter/setter tests and some tests only run lambdas for coverage (`fetchRejectRisks_pipelineLambdas_fullyForced`). Useful tests are named after rules (`showServiceTax_checklistCopyY_closesGate`, `execute_band9_excludedForPoscoverAzp_doesNotFire`). Add seeded-Mongo tests for boundaries.
- Common mistakes: Quoting the total test count as proof.
- Follow-up questions: Why no `ArgumentCaptor`? (0 uses; tests use `doAnswer`.)

**Q58. Symptom: a date test is flaky across machines.**
- Why the interviewer asks: Zone dependence.
- Ideal answer: DTO dates are `String` from a BSON `Date`, likely in `Date.toString()` form with a zone abbreviation (for example IST, UTC), parsed by pattern `EEE MMM dd HH:mm:ss z yyyy` (other handlers comment on the same trap, for example `MarinePolScheduleHandler`). The result depends on JVM default zone. Steps: run the test with `-Duser.timezone=UTC` and another zone; use fixed zones in tests; prefer ISO strings from the pipeline.
- Common mistakes: Using `new Date()` in expectations.
- Follow-up questions: How does `toLocalDate` treat `03/04/2026`? (dd/MM first: 3 April.)

---

## K. XML output symptoms

**Q59. Symptom: the XML has `policyletterind` (lower case) but the XSD wants `Policyletterind`.**
- Why the interviewer asks: Jackson naming with Lombok.
- Ideal answer: Lombok generates `getPolicyletterind()`; Jackson bean naming turns it into `policyletterind`. The current `PlineReportData` fixes it with field visibility and `@JacksonXmlProperty`, so `Policyletterind` is emitted (the old Sep-1 sample still showed lower case). Steps: serialise the class in a test and print; check annotations (`@JsonAutoDetect` on fields, `localName`); compare with the XSD element in the `reportData` sequence. XML is case sensitive.
- Common mistakes: Renaming the field only.
- Follow-up questions: Which other leaves have unusual casing? (`Columncount`, `Emailbcc`, `TableHeaderCOUNT`, `sumTotalPremiumMEDIX`, `LOADING`.)

**Q60. Symptom: `<LOADING>` is present but `<loading>` is missing (or reverse).**
- Why the interviewer asks: Two leaves differing only by case.
- Ideal answer: The XSD has both: `LOADING` (Jasper field, decimal, added v0.1.5) and `loading` (older Bond-derived camelCase). Java has two fields: `loading` and `loadingUpper` with `@JacksonXmlProperty(localName="LOADING")`. Check which one the service populates for the band in question; `PREM` vs `premium` is the same pattern. In the XSD sequence, `LOADING` comes before `loading`; Java declares `loading` first (a known order mismatch).
- Common mistakes: Treating it as a typo and merging them.
- Follow-up questions: How did you find the order difference? (Static diff of field order vs XSD sequence.)

**Q61. Symptom: a consumer cannot find `prntDetails` under `reportTransaction`.**
- Why the interviewer asks: v0.1.3 path change.
- Ideal answer: Since v0.1.3 the path is `reportTransaction/reportData/prntDetails` (last child inside `reportData`). The first Pline draft had it beside `reportData`. Check the consumer's XPath and which XSD version they use; confirm in Java that `PlineReportData` owns `prntDetails`. This was a breaking move, so an old consumer breaks silently on missing nodes.
- Common mistakes: Changing Java to put it back.
- Follow-up questions: Which similar move happened in v0.1.4? (`permanentDisablement` from `reportParameters` to `reportData`.)

**Q62. Symptom: the XML is full of empty elements such as `<slNo/>`, `<permanentDisablement/>`, `<effectivedatefrom/>`.**
- Why the interviewer asks: Null serialisation.
- Ideal answer: Pline payload classes are annotated `@JsonInclude(Include.ALWAYS)` and the controller uses a plain `new XmlMapper()`, which writes a null as an empty element. An empty element is valid only for `xs:string`; for decimal, dateTime, integer and boolean it is invalid. In the older real RejectRisk sample, 456 of 929 errors were this (387 decimal, 56 dateTime, 10 integer, 3 boolean; each reported twice by Xerces). Steps: identify the leaf type in the XSD; choose between `NON_NULL`/`NON_EMPTY` or keeping elements for Jasper. Whether Jasper relies on the empty elements is unknown. Do not change globally without the consumer's answer.
- Common mistakes: Setting NON_NULL on one class only and mixing styles.
- Follow-up questions: What is `xsi:nil`? Why does the XSD not use `nillable`?

**Q63. Symptom: response time and size are large for a simple RejectRisk listing.**
- Why the interviewer asks: Payload bloat.
- Ideal answer: Each DB row becomes a full `reportData` with `prntDetails` (containing default empty `agentInfo`, `branchInfo`, `clientInfo` ... because of `ALWAYS`), a `transDetList` with one `transDet` (about 650 possible leaves, most empty) and a `companyType/companyDesc`. The XML is then escaped into a JSON string. Investigate by measuring the `data` length per row; see which defaults are created in the constructors; consider `NON_EMPTY` for the listing family after consumer agreement. Memory: all rows are read into a list first.
- Common mistakes: Tuning Mongo when the cost is serialisation.
- Follow-up questions: Why per-row `reportData`? (`companyDesc` sits beside `transDet`; one remarks slot per row.)

**Q64. Symptom: rows are multiplied - a value that should appear once appears on every `transDet`.**
- Why the interviewer asks: Merge vs add.
- Ideal answer: `mergeSingleTransDet` merges non-null fields onto every existing `transDet`; if a band is single-row but called via a multi-row path (`addSingleTransDet`), the number of rows multiplies. Steps: list the `transDet` count after each band in a unit test (`Template12PolicyDocumentBaseServiceTest` has `mergeSingleTransDet_*` tests); find the band that creates extra rows; check whether the band is meant to be a separate `reportData` (landscape bands are, via `toLandscapeReportData`).
- Common mistakes: De-duplicating in the renderer.
- Follow-up questions: What does last-writer-wins imply? (An earlier value can be replaced.)

**Q65. Symptom: a default company description (blank) appears before the real one in RejectRisk or Template12.**
- Why the interviewer asks: Constructor defaults.
- Ideal answer: The `companyType` constructor adds a default placeholder `companyDesc`. The service calls `companyType.getCompanyDesc().clear()` before adding the real one (RejectRisk) and the same pattern exists in `Template12PolicyDocumentBaseService.newSharedReportData` (test `newSharedReportData_clearsDefaultCompanyDescPlaceholder`). Steps: look for the creation site that lacks the `clear()`; check the test for the base service.
- Common mistakes: Filtering blanks in the consumer.
- Follow-up questions: Where is `remarks` in the XSD? (`transDetList/companyType/companyDesc/remarks`.)

**Q66. Symptom: a "no data" strategy response still contains an empty `reportData` with dozens of empty tags.**
- Why the interviewer asks: Default lists.
- Ideal answer: `PlineHeaderTransaction.reportTransaction` defaults to a list with one empty `PlineReportTransaction`, and `PlineReportTransaction.reportData` defaults to one empty `PlineReportData` (KT 04 5.5). RejectRisk replaces the list with real rows, but when it has zero rows the report data list is empty (test `execute_emptyRows_returnsEmptyReportDataList`); other services may keep the defaults. Steps: read the payload class constructors; check how the service resets the list. Be careful in the detection of "empty" responses by a consumer.
- Common mistakes: Treating any `reportData` as proof of data.
- Follow-up questions: How would you detect real data? (A populated `transDet` field such as `rownumber` or `polno`.)

**Q67. Symptom: the downstream parser fails on the `data` field - "unexpected character" or double-escaped tags.**
- Why the interviewer asks: XML-in-JSON handling.
- Ideal answer: `SuccessResponse.data` is a JSON string containing the XML (`xmlMapper.writeValueAsString(transactions)`), so quotes, `<` and `>` are escaped as JSON. The consumer must read `data` as a string value first and not parse the whole response as XML. The README says `xmlData` but the class field is `data` (verify with a live response). Steps: curl and print raw; run a JSON parser, then an XML parser on the extracted value.
- Common mistakes: Applying the XML parser to the whole body.
- Follow-up questions: What is the `timestamp` format? (`LocalDateTime.now().toString()`.)

**Q68. Symptom: a customer name with `&` or `<` breaks a consumer.**
- Why the interviewer asks: Escaping responsibilities.
- Ideal answer: Jackson's XML writer escapes `&`, `<` and `>` in text, so a well-formed consumer sees normal text. Check the raw `data` string for the escaped form; if the name shows `&amp;amp;` the consumer is escaping twice. Illegal control characters in a name (rare, from copy-paste) are a different problem; the behaviour of the XML writer with them was not tested here (inferred). Steps: reproduce with a test name in a unit test and parse with a standard XML parser.
- Common mistakes: Replacing characters in the service.
- Follow-up questions: How do you test it? (Serialise a payload with such a name and parse it back.)

**Q69. Symptom: XSD validation fails on `benefitSi` or a decimal leaf with text like `1E+3`.**
- Why the interviewer asks: Decimal formatting.
- Ideal answer: Two cases. (1) `PlineBenefit.benefitSi` is a Java `String` but the XSD type is `xs:decimal`: the only primitive type mismatch the KT found; text such as "Up to 10,000" is invalid. (2) For `BigDecimal` fields Jackson XML writes `BigDecimal.toString()` unless plain-string writing is enabled, and `toString()` can yield scientific notation for some scales; `xs:decimal` does not allow exponent form (inferred from library behaviour; not tested here). Steps: locate the offending value in the XML; check the Java type; format with `toPlainString()` or set the Jackson feature, after checking the consumer.
- Common mistakes: Stripping characters in the string field.
- Follow-up questions: Where is the decimal for ROWNUM created? (`toBigDecimal(Integer)`.)

**Q70. Symptom: your XSD test gives "cannot resolve Common.xsd" or stops at the first error.**
- Why the interviewer asks: Test harness mechanics.
- Ideal answer: `Pline.xsd` includes `Common.xsd` by relative path, so the schema must be loaded from a file URL (a `File`-backed source), not an `InputStream` without a base URI (as `PlineXsdValidationTest` does). To see all errors, set an `ErrorHandler` that collects them; Xerces throws the first fatal error otherwise. Also Xerces reports each datatype error twice (`cvc-datatype-valid` and `cvc-type.3.1.3`), so halve the count when reasoning. Add a ratchet: assert `errors <= baseline`.
- Common mistakes: Reading "929 errors" as 929 distinct problems.
- Follow-up questions: Why are the sample numbers only indicative? (The samples came from older `T12_final_code`, dated 1 Sep 2026.)

**Q71. Symptom: validator message "Invalid content was found starting with element 'mainReportName'". Meaning and action.**
- Why the interviewer asks: Reading XSD errors.
- Ideal answer: `reportData` in v0.1.6 has no `mainReportName`/`subReportName` (grep returns nothing), but Java writes them on every `reportData` (via `PlineReportNames.tag` and `PlineReportBlocks`; commits `e74bc0d5c`, `861f0666c`). Java comments cite XSD v0.1.10 which I could not find. Action: do not remove the fields; raise the contract gap with the schema owner and request an updated XSD, or agree to ignore. Same for `pCreatedDateFrom/To` and `pReportTitle`.
- Common mistakes: Editing the repo XSD copy alone (it must match the real contract).
- Follow-up questions: What did the 5-error experiment show? (Only these categories failed when a correctly ordered file had them added.)

**Q72. Symptom: validation errors say "address" is out of order in agentInfo, branchInfo, compInfo and taxClientInfo.**
- Why the interviewer asks: Sequence order.
- Ideal answer: The XSD puts `address` first in these types (they are generated alphabetically with Pline additions at the end); Java declares `address` last. Fix with `@JsonPropertyOrder` or by moving the field in `PlineAgentInfo`, `PlineBranchInfo`, `PlineCompInfo`, `PlineTaxClientInfo`. Similar mismatches: `clientInfo`, `buyerInfo`, `supplierInfo`, `cover`, `risk` (`occpDescp`), `transDet`. Only `PlinePrntDetails`, `PlineTransDetList`, `PlineTransactionDetails`, `PlinePaymentsType` have `@JsonPropertyOrder`. Because order matters only to validators (name-based readers do not care - inferred), agree priority first.
- Common mistakes: Fixing one class and assuming all are done.
- Follow-up questions: How would you detect all of them automatically? (A reflection test comparing Java field order to the XSD sequence.)

**Q73. Symptom: validation fails at the top: `mailingDetails` appears before `transactionDetails`, `reportParameters` last.**
- Why the interviewer asks: Shared class order.
- Ideal answer: XSD sequence in `transaction`: `reportParameters, mailingDetails, templateDetails, transactionDetails`. The shared Motor class `Transaction` declares `templateDetails, mailingDetails, transactionDetails, reportParameters`. Changing it affects Motor, Fire and Bond XML. Options: `@JsonPropertyOrder` on the shared class after agreement; or a Plines subclass. Check the blast radius with the other LOB tests.
- Common mistakes: Editing the shared class and not telling others.
- Follow-up questions: What does `@JsonInclude(NON_NULL)` on `Transaction` do? (Skips null children.)

**Q74. Symptom: `templateDetails` children are not alphabetical though a code comment says "XSD order".**
- Why the interviewer asks: Documentation vs truth.
- Ideal answer: The Common.xsd `templateDetailsType` is alphabetical (`batchConfigId, boResId, channel, copyTypeList, dataClass, ...`); Java emits `channel, outputFormat, templateName, copyTypeList, reportName, outputPath, ...`. The comment refers to the Motor XSD order, not Common.xsd. Verify with the XSD, not the comment. Also `mergeStaticPdfFiles` is an extra element in Java (only written when non-empty).
- Common mistakes: Trusting code comments.
- Follow-up questions: Where does `TemplateDetails` get its values? (`populateTemplateDetails` from `metadata`.)

**Q75. Symptom: RejectRisk remarks are missing in the PDF but present in Mongo.**
- Why the interviewer asks: Mapping location.
- Ideal answer: `REMARKS` is not on `transDet`; it maps to `transDetList/companyType/companyDesc/remarks`, a sibling of `transDet`. Steps: check the XML has `companyType` inside `transDetList` of that `reportData`; check `companyType.getCompanyDesc().clear()` ordering; check the renderer reads that path. XSD has `companyType` in two places (`prntDetails/companyType` and `transDetList/companyType`); RejectRisk uses the second.
- Common mistakes: Checking `transDet`.
- Follow-up questions: What is the Jasper field? (`REMARKS`, 9th column.)

**Q76. Symptom: the "NO" column in the PDF is not the same as `rownumber` in our XML.**
- Why the interviewer asks: Jasper detail.
- Ideal answer: In the original jrxml the NO column prints `$V{GroupByBranch_COUNT}`, not the SQL `ROWNUMBER`; `ROWNUMBER` is only used in `printWhenExpression ROWNUMBER != null`. So the renderer decides what to print; our `rownumber` follows the `$setWindowFields` order (Mongo null-first). Compare with the report team's template. Differences in order also change the counter.
- Common mistakes: Fixing the number in Java.
- Follow-up questions: Which fields are not printed at all? (`REJECT_RISK_TYPE_CODE`, raw `REVOKED_DURATION`.)

**Q77. Symptom: the reason column shows "R1 / null".**
- Why the interviewer asks: Null concatenation.
- Ideal answer: The jrxml prints `REASON_CODE + " / " + REASON_CODE_DESCRIPTION` (Groovy), so a null description prints "null" (inferred). Our XML sends an empty/null `reasonCodeDescription` if `CMGE_CODE` has no row for category `PLREJ_REASON` and that code. Steps: query `CMGE_CODE` by `CAT_CODE` and `CODE_CD`; check type of `REASON_CODE`; fix the data, not the Java.
- Common mistakes: Adding a default text in Java without a decision.
- Follow-up questions: What does "NA" cover in Jasper? (Null/empty for most text columns; dates are blank when null.)

---

## L. Logic differences between Jasper and Java

**Q78. Symptom: BMW nomination page has no certificate data in PD2.**
- Why the interviewer asks: Hardcoded variable.
- Ideal answer: The merged PD2 block hard-codes `String certNo = "";` (`S12:583`), so `addNominationBMW(contractId, certNo, rd)` never receives a certificate, while the old standalone PD2 service read `certNo` from the request. This is a behaviour change from the merge. Investigate by logging the call arguments; check the Jasper PD2 jrxml: it has no `certNo` parameter (`certNo` is not a Jasper parameter in PD2 per KT 02). Confirm with the business before restoring.
- Common mistakes: Assuming a data gap.
- Follow-up questions: Which unregistered class still reads `certNo`? (`Template12PolicyDocument2Service`.)

**Q79. Symptom: every Template12 policy, including non-ATC ones, has an Overseas ATC marker row.**
- Why the interviewer asks: Gate difference.
- Ideal answer: Jasper gates Domestic/Overseas ATC by product 070406. Java does `if ("DOM".equalsIgnoreCase(journeyArea)) addT12DomesticAtc(rd); else addT12OverseasAtc(rd);` with no product test (`S12:616-621`); tests `execute_pd2JourneyAreaDom_usesDomesticAtcBranch` and `execute_pd2PrimaryDtoNull_journeyAreaDefaultsToOverseas` lock it. The row is dispatch-only (marker). Investigate by listing `subReportName`/marker rows; decide whether the renderer ignores them.
- Common mistakes: Removing the code without checking the tests and renderer.
- Follow-up questions: What other PD2 calls are unconditional? (`addParticularOfInsuredTpl4`, `...Base`, `addNominationPA4A`, `addNominationBMW`, `addInstallmentDetails`.)

**Q80. Symptom: PDS_Income or PDS_Elite marker row appears but Jasper never printed those pages.**
- Why the interviewer asks: Dead upstream bands.
- Ideal answer: The jrxml band conditions start with `false && ...` (dead upstream), but Java wires them (`S12:770-775`, comment "wired here per explicit instruction"); tests `execute_070504IncomeShield_firesPdsIncomeOnly` and `...EliteShield...` assert it. The row carries a `companyDesc.companyRegNo` marker. It is a known behavioural difference. PDPA (CommonReport), NominationForm and the post-GST blank are correctly not implemented.
- Common mistakes: Treating it as a regression to be reverted.
- Follow-up questions: Who gave the instruction? (Not provable from the repo.)

**Q81. Symptom: `promoAmt` differs from Jasper, and the Template4 landscape band (070406) fires or not unexpectedly.**
- Why the interviewer asks: Join multiplication difference.
- Ideal answer: Jasper sums `COV_PROMO_AMT` by joining risk persons and covers only on contract, so each cover repeats per insured (inferred: N insured gives N times the sum). Java uses an existence lookup then a plain `$group $sum` (not multiplied); the Landscape dispatch also lacks version filters on `WIP_UWGE_COVER`/`UWGE_COVER` and `VERSION_NO=1` on `UWPL_RISK_PERSON` (KT 02 D7). Band L16 runs when `070406 && promoAmt > 0`. Steps: compute the three values for one contract (Jasper-style join, Java sum, with and without version filters).
- Common mistakes: Matching the multiplication without the business asking for it.
- Follow-up questions: Which handler? (`PlineT12PolicyDocumentHandler` promo block; `Template12LandscapeDispatchHandler:196-211`.)

**Q82. Symptom: ASP schedule shows `INST_IND` = N and GST label N though Jasper shows Y after a cutoff.**
- Why the interviewer asks: Missing constants.
- Ideal answer: Jasper computes `INST_IND` from `SAPM_SYS_CONSTANTS` (`CUT_OFF_RPT`, `<product>_ENDT`) and `PMT_FREQUENCY`, and `SHOW_GST_LABEL` from the SST rollout date. `AspEndtScheduleHandler` fetches `PMT_FREQUENCY` but never uses it, and takes `INST_IND`/`SHOW_GST_LABEL` from `$_wupb` defaulting to "N" (lines ~786, 794). Same for the debit note label. The master row label comes from `UWGE_VI_ENDT_REPORT_PARAM`. Steps: log the values; compare to constants; mark as known gap.
- Common mistakes: Changing the default to "Y".
- Follow-up questions: Where is the SST rollout date used in ASP? (Only in the service-tax handler.)

**Q83. Symptom: `showComplaint=N` but the ASP complaint page prints (or `Y` and it does not).**
- Why the interviewer asks: Unused flag.
- Ideal answer: In `AspEndtDocumentService`, `showComplaint` is only echoed into `reportParameters`; the page is controlled by product 070412 (as in the EndtDocument2 band). Also `pSchemeName` (used by Jasper layout for 070504 schemes) is not read from the request at all. Steps: read the gate at line ~288 and ~1414. Tell the caller the real rule.
- Common mistakes: Adding a `showComplaint` check without agreement.
- Follow-up questions: What else is accepted but unused? (`pPolicyNo`.)

**Q84. Symptom: ASP EndtCover returns data of a different endorsement when both `pEndtNo` and `pRefNo` are blank.**
- Why the interviewer asks: Edge case in the match builder.
- Ideal answer: In `AspEndtCoverHandler.buildMatch`, when both are blank an empty `Document` is returned, so the pipeline takes an arbitrary first row; Jasper's `ENDT_NO = null` never matches (inferred, KT 03 4.2 #8). Investigate by calling with blanks in dev; add validation of the request or a guard that returns nothing.
- Common mistakes: Treating this as a data problem.
- Follow-up questions: How would you protect all handlers? (Validate required keys in the service.)

**Q85. Symptom: relationship text on the delete-member page has an extra prefix compared with Jasper.**
- Why the interviewer asks: Function replication.
- Ideal answer: Jasper uses `SUBSTR(CODE_DESC,'4')` (drops the first three characters) for `RELATIONSHIP_DESC`; `AspDeleteMemberLandscapeHandler` returns the full `CODE_DESC`. Steps: compare for one record; confirm the code descriptions start with a numeric prefix; implement `$substrCP` if the business wants parity. It is an unmapped difference listed in KT 03.
- Common mistakes: Trimming in the renderer.
- Follow-up questions: Where else do codes carry a prefix? (Not known.)

**Q86. Symptom: six delete-member fields (`subcoverPrem`, `endtDoubleIndemnity`, `endtImaFee`, `endtStaxAmt`, `endtLlbPrem`, `endtStiPrem`) never appear. Investigate.**
- Why the interviewer asks: Known unmapped fields with a nuance.
- Ideal answer: The service comment (lines ~1372-1374) says they are fetched but have no verified leaf. The KT guide says "no XSD leaf". The ASP deep dive adds that `endtDoubleIndemnity`, `endtImaFee`, `endtLlbPrem`, `endtStaxAmt`, `endtStiPrem` exist under `endorsementDetails` in the XSD (lines ~607-617), and `PlineEndorsementDetails` has such fields; the missing piece is a leaf on `transDet`/`PlineCover`. The two KT sources differ; I would check the XSD and class myself before saying either. Per-row values do not fit the per-contract `endorsementDetails`, which explains the gap (inferred).
- Common mistakes: Putting per-member values into `endorsementDetails`.
- Follow-up questions: Which fix needs the XSD owner? (Adding transDet leaves.)

**Q87. Symptom: occupation description shows for all members or for none on the ASP add-member page.**
- Why the interviewer asks: Per-row vs per-call evaluation.
- Ideal answer: Jasper evaluates `SHOW_OCCP_DESC` per row from the subreport's issue date versus the system constant (`OCCUP_CUT_OFF_DATE`/`OCCUP_DATE`); Java evaluates it once per handler call and returns "N" if issue date or constant is null (`AspAddMemberLandscapeInsuredHandler`, lines ~145-153). Steps: log issue date and constant; check the constant type in `SAPM_SYS_CONSTANTS`. The Jasper SQL also depends on a date-to-string conversion of a `java.util.Date`, which is fragile and not replicated.
- Common mistakes: Making it per member without a business rule.
- Follow-up questions: Where does the flag appear? (`transDet.showOccpDesc`.)

**Q88. Symptom: deleted members exist in the endorsement but the delete page is absent.**
- Why the interviewer asks: Different code lists in two places.
- Ideal answer: The dispatch's `deletionExists` uses `ENDT_CODE in (98)`, while `AspDeleteMemberLandscapeHandler` matches `ENDT_CODE in (98,168)` with `ACTION_CODE=C` and `TERMINATE_DATE` not null (KT 03 section 6). An endorsement with code 168 and deleted members yields `deletionPrint = 0` and the rows are never fetched (inferred from the two code lists). Investigate: print the dispatch counts; run both queries. Raise with the business whether 168 should count.
- Common mistakes: Changing both lists without confirming with the Jasper source.
- Follow-up questions: Which Jasper condition holds the list? (AddMemberLandscape `deletion_print` and `DeleteMemberLandscape_ASP`.)

**Q89. Symptom: BMW policy produces fewer certificate rows than expected.**
- Why the interviewer asks: Range inputs.
- Ideal answer: The BMW branch builds one row per certificate between `pCertNoFrom` and `pCertNoTo`. Both are request keys with default `""`; they are used for the range but not serialised into `reportParameters` (dropped by the whitelist). Steps: check the request values; query `UWPL_RISK_PERSON`/`WIP_UWPL_RISK_PERSON` for `cert_no`; the Jasper ordering is by `det.cert_no`. Test: `execute_bmwScheme070504_mapsFirstCertRowAndRunsNestedSubreportsPerRow`.
- Common mistakes: Expecting all certificates when the range is empty.
- Follow-up questions: What does an empty range mean in Jasper? (Not documented in the KT.)

**Q90. Symptom: for product 070414 (PA4A) some pages repeat per row.**
- Why the interviewer asks: Per-row nested bands.
- Ideal answer: For 070414 the service creates one set of nested bands per primary row (`CIPrint_PA4A`: nomination by `NOM_PRINT`, perils by `CLAUSE_PRINT`, schedule of benefits by `SHOW_NEW_SOB`/`SHOW_SOB_P3_IND`) and uses `ServiceTaxInvoice_PA4A` instead of `ServiceTaxInvoice` in PD2. Test `execute_pa4aProduct070414_mapsPolicyNoAndRunsPerRowNestedSubreports`. Steps: count the primary rows; confirm the repetition is by design.
- Common mistakes: Filtering rows to get one set.
- Follow-up questions: Which flags select the SOB variant? (`SHOW_NEW_SOB`, `SHOW_SOB_P3_IND`.)

---

## M. Method questions

**Q91. How do you compare a Java gate with the original Jasper condition?**
- Why the interviewer asks: Core retrofit skill.
- Ideal answer: Open the jrxml (`...\jasper_dumps\jasper_02072026\resources\reports\AGIC\UW\Plines\...\*_jrxml.data`) and read the band `printWhenExpression`, the SQL and parameters. Write the condition on paper in terms of flags. Then find the Java gate (for example `AspEndtDocumentService` ~335, or the booleans `is070504` etc. at `S12:179-187`) and compare each term, including null behaviour (Jasper `.equalsIgnoreCase` on null throws NPE, Java uses `"Y".equalsIgnoreCase(x)` and defaults). Note `false && ...` bands, page-number terms and `printWhenExpression` of the band vs the inner element.
- Common mistakes: Comparing only names of the sub-reports.
- Follow-up questions: Where are the tables that map Jasper to Java? (KT 02 section 3 and KT 03 section 3.3.)

**Q92. How do you check that the Java output equals the Jasper output for one record?**
- Why the interviewer asks: Parity method; no golden files exist.
- Ideal answer: Choose a record with known edge features (null names, boundary dates, empty-string filters). Run the Oracle SQL from the jrxml and the Java service with the same filters. Compare counts, order, each column and null display ("NA" in Jasper). For RejectRisk the SQL is in the jrxml lines 77-114. Record the differences (null sort order, `ROWNUM`, time zone). No sample data or golden files are in the repo; create them for the case.
- Common mistakes: Trusting a single happy-path row.
- Follow-up questions: What would an automated version need? (A seeded Mongo and expected XML.)

**Q93. How do you check whether a fix is in the build you are running?**
- Why the interviewer asks: Branch divergence (`cc6a1b84d`).
- Ideal answer: (1) Read the handler in the deployed image's source revision, if the revision is known. (2) `git branch --contains <sha>` and `git log -S"<snippet>"` on the branch. (3) Compare the pipeline in the DEBUG log. Remember `target/classes` can be stale (the working copy has compiled classes from Oct 8 and no `target/test-classes`), so rebuild before testing. The OPUSM-41763 branch is 194 commits ahead and 1363 behind HEAD, so it cannot be merged blindly.
- Common mistakes: Trusting the commit message.
- Follow-up questions: What differs in the service on HEAD vs that branch? (`PlineReportBlocks`/`PlineReportNames` lines exist only on HEAD.)

**Q94. How do you find when a behaviour was introduced without blaming a person?**
- Why the interviewer asks: Safe use of git history.
- Ideal answer: `git log -S"<snippet>" -- <path>` or `git log --follow -- <file>` then read the diff. Authors in git are names such as Shinge, GS, Shaw, Karunakar, Amarnath Yadav; none matches your git user (`NISHUNISARGA`), so do not claim authorship. Talk about the ticket and the change, for example `c0c6d449a` (2026-08-03, dynamic merging) or `0d715fbad` (2026-09-01, RejectRisk mapping fix). [CONFIRM: which tasks you actually worked on, from your own records.]
- Common mistakes: Claiming a commit you cannot prove.
- Follow-up questions: What does commit `3f2157b28` tell you about review? (It mixes tests with 200+ production files.)

**Q95. You need to debug locally against a real database. How do you avoid leaking credentials?**
- Why the interviewer asks: Secret handling during debugging.
- Ideal answer: Do not put the URI into `application.properties`. Use `SPRING_MONGODB_URI`, `SPRING_MONGODB_DATABASE` env vars or command-line args. Before committing, run `git status` and `git diff` and use `git add -p`; the working copy here has an uncommitted block with credentials (<redacted>) which must stay out. Do not paste pipelines containing personal data into tickets. The nightly secret scan and PR scan exist but the allowlist covers two whole files.
- Common mistakes: Committing "temporary" config.
- Follow-up questions: What if it was pushed? (Rotate, tell security.)

**Q96. How do you add temporary logging without creating a PII problem?**
- Why the interviewer asks: Practical judgement.
- Ideal answer: Log identifiers and counts (contract ID, number of rows, gate booleans, product, scheme), not names or ID numbers. Use `log.debug` and remove before merge. Note the existing risks: `CustomRequestFilter` prints the body with `System.out`; `MongoTemplate` DEBUG logs pipelines. For RejectRisk avoid logging `pCustomerName`, `pIdNo`.
- Common mistakes: Logging whole DTOs.
- Follow-up questions: Where would you print the gates for ASP? (Just before the service-tax `if` in `AspEndtDocumentService`.)

**Q97. Your reproduction behaves differently from the caller's. What do you check?**
- Why the interviewer asks: Defaults.
- Ideal answer: Defaults from `getOrDefault`: Template12 `showComplaint` "Y", `showScheduleStd` "Y", `versionNo` "1", `checklistCopy` "N", others "N"; ASP `showXxx` "N", `pFirstPageNumber` "1"; the key `ShowOutPrintCovLetter` has a capital S. A missing key behaves as the default, a mistyped key is ignored silently. Also metadata fields (`copyType`, `templateId`) and `pBranch` arrays. Steps: ask for the exact raw body; diff against yours; check whether `CustomRequestFilter` changed it (`pBranch`).
- Common mistakes: Retyping the request.
- Follow-up questions: What does `PayLoadRequest.setMetadata` drop? (Booleans, nulls, nested objects.)

**Q98. Give a layered checklist to decide where a wrong value comes from.**
- Why the interviewer asks: Structured bisecting.
- Ideal answer: Five layers: (1) request: keys, defaults, spelling; (2) routing: exact bean name, merged names; (3) handler: pipeline output (DEBUG log, mongosh), types, WIP vs live; (4) service mapping: DTO to payload, merge, precedence, swallowed exceptions; (5) serialisation: whitelist, `@JsonFormat`, element names and order, empty elements. For each layer there is one cheap test: print the input, the bean, the pipeline, the payload object, the XML. I stop at the first layer where the value is already wrong.
- Common mistakes: Starting from the PDF.
- Follow-up questions: Which layer causes most silent problems? (Handler type mismatches and swallowed exceptions.)

**Q99. Symptom: `filterCondition` binding fails with an array or number value. How do you investigate?**
- Why the interviewer asks: Request binding limits.
- Ideal answer: `filterCondition` is `Map<String,String>`, so a JSON number, array or object fails (400/500; malformed JSON also lands in the catch-all 500 - inferred). Only `pBranch` and `pBranchName` arrays are converted to comma strings by `CustomRequestFilter`. Steps: validate the JSON, send values as strings, check the printed body in stdout. In `metadata`, numbers are kept as strings, booleans and nulls are dropped.
- Common mistakes: Sending `"pContractId": 123456` as a number.
- Follow-up questions: Why is this good for security? (Operators cannot be injected as objects.)

**Q100. Symptom: `templateDetails` has empty or null `templateId`/`batchConfigId` although metadata had them.**
- Why the interviewer asks: Safe parse.
- Ideal answer: The controller reads integer-valued metadata through `parseIntSafe`, which returns null on bad input, so `"55 "` or `"abc"` silently becomes null. Also if `metadata` has the value as boolean/nested it is dropped by `PayLoadRequest.setMetadata`. Steps: print the raw metadata; check the values are numeric strings; check `populateTemplateDetails`. Elements with null values are skipped by `@JsonInclude(NON_NULL)` on `Transaction`, but inner classes may write empty elements.
- Common mistakes: Searching for a serializer bug.
- Follow-up questions: Which metadata keys feed `mailingDetails`? (`name`, `email`, `senderMail`.)
