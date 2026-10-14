# 09c - Debugging Questions 1-50 (symptom -> how you investigate)

Service: `opus-documentprintdata-service` (Spring Boot 4.1.1, Java 17, MongoDB, XML-in-JSON). Modules: Template12, ASP (EndtDocument family), RejectRisk.

Rules for reading this file
- Each question is a symptom. The ideal answer is an investigation path with real class, collection, field and property names from the KT files and the repo.
- "(inferred)" = my reasoning, not proven by a run. Nothing here was executed against real data; nobody has run Maven on this working copy (no `target/surefire-reports`).
- No real incident tickets exist in the repo. Answers say "this is how I would investigate", never "this happened to me". Use [CONFIRM: ...] only if you truly debugged the case.
- Git authorship is not attributable to you. Secrets are <redacted>.
- Operations/process questions are in 09a/09b; XML structure questions are in 09e/09f. Q51-Q100 of debugging are in 09d.

General tool kit used below
- Reproduce with the same JSON body (Swagger UI or curl) and read the XML inside `data`.
- `MongoTemplate` DEBUG logging prints pipelines; paste the pipeline in mongosh and run it, with `explain("executionStats")` if slow.
- Unit tests capture the `Aggregation` and call `toPipeline(Aggregation.DEFAULT_CONTEXT)`; use the same trick in a scratch test.
- Compare with the original jrxml `printWhenExpression` (folder `...\AGIC\UW\Plines`) for the same gate.

---

## A. Null gates and silent "does not print"

**Q1. Symptom: ASP request has `showTaxInvoice=Y`, but the XML has no `ServiceTaxInvoice` reportData. How do you investigate?**
- Why the interviewer asks: The service-tax gate has four inputs and some are external. Tests structured elimination.
- Ideal answer: Open `AspEndtDocumentService` around line 335. The gate is `showTaxInvoice=Y && showStaxInvInd=Y && checklistCopy != Y && showEinvInd=Y`. Step 1: check the request value of `checklistCopy` (Y closes the gate by design; test `showServiceTax_checklistCopyY_closesGate`). Step 2: read `showStaxInvInd` and `showEinvInd` from `mainHandler2` (`AspEndtDocument2MainHandler`); `showEinvInd` comes from `PlineUtilsClient.fnEinvoicePrintEndtInd`, which returns "" on failure. Step 3: grep logs for `fnEinvoicePrintEndtInd ... failed`. Step 4: check the main DTO is not all-null (see Q4). Step 5: if the gate opens but the row is empty, look at `AspServiceTaxInvoiceHandler` (it can throw `IllegalStateException` for ambiguous e-invoice cutoff or live/WIP version conflict).
- Common mistakes: Reading only the request flags; ignoring the REST-derived flag.
- Follow-up questions: Which of the four inputs is not under the caller's control? How would a debugger breakpoint help here?

**Q2. Symptom: all `showXxxInd` flags come back null for an ASP endorsement although Oracle has `Y`. Investigate.**
- Why the interviewer asks: Classic type-mismatch bug (CONTRACT_ID).
- Ideal answer: Suspect a type mismatch. Mongo `$match` is type strict: a String `"123"` never matches numeric `CONTRACT_ID`, and no error is raised. The shared base pipeline compared String to number; `AspEndtDocument2MainHandler` overrides `buildCommonPipeline` and uses `safeNum` (`Long.parseLong`). Steps: (1) print the pipeline from DEBUG logs; (2) look at the literal type in `$match` (`NumberLong` vs string); (3) in mongosh run `db.UWGE_VI_ENDT_REPORT_PARAM.find({CONTRACT_ID: <both types>})` and check `typeof`/`$type`; (4) confirm the build contains the override (the fix commits 578778080, ba9e29586 are on other branches per the KT; not in HEAD as commit ids, but the override exists in code - verify in your build).
- Common mistakes: Looking at the REST client first.
- Follow-up questions: What else breaks quietly with this bug? (Every gated sub-report never fires.) Which other handlers use `safeNum`?

**Q3. Symptom: Template12 PD2 service-tax invoice missing, `showTaxInvoice=Y`. Which flags and where?**
- Why the interviewer asks: Template12 uses a different primary fetch than ASP.
- Ideal answer: PD2 uses `PlineT12PolicyDocumentHandler`: it prefers posted data, falls back to WIP with `$ifNull`, and calls REST for `EINV_PRINT_IND` and `SHOW_STAX_INV_IND` (property `plines.fn.show.stax.inv.ind.path`, mode `WIP` when `checklistCopy=Y` else `OCP`). The gate in `executeMergedPolicyDocument2` needs `showTaxInvoice=Y`, `showStaxInvInd`, and `einvPrintInd`. Investigate: DTO values (breakpoint or unit test with real DTO), the REST response in dev, log lines for failures. For product 070414 a different invoice (`ServiceTaxInvoice_PA4A`) is used. [CONFIRM: if you traced this on real data].
- Common mistakes: Using the ASP gate for Template12.
- Follow-up questions: Which test names exist for the PD2 gates? What differs for 070406?

**Q4. Symptom: for one ASP endorsement, product code and ENDT_CODE are null in the master row, so the 070412 and 72/168 gates close.**
- Why the interviewer asks: Anchor collection risk found in KT 03.
- Ideal answer: `AspEndtDocumentMainHandler` (HEAD) starts from `UWMT_VI_ENDT_TAX_INV_ALT_UPV` with `$match {ENDT_NO, CONTRACT_ID}`. If that collection has no row for the endorsement, the pipeline returns an empty/default DTO and every lookup after it is lost. The base class comment in `PlineEndtDocument2BaseHandler` says that collection is not related to the contract and the base seeds with `$limit 1` instead. The commit `cc6a1b84d` (another branch) rewrote the handler to resolve each field independently. Investigate: query the anchor collection for the ENDT_NO + CONTRACT_ID pair; check the main DTO fields in a test. This is reasoning from code (KT 03 4.1 #3), not a confirmed production bug.
- Common mistakes: Concluding Mongo has no policy data.
- Follow-up questions: What would an all-null DTO do to the response order? How would you test the fix?

**Q5. Symptom: a debit note prints for an endorsement with no premium change.**
- Why the interviewer asks: Missing-field semantics in Mongo vs SQL.
- Ideal answer: `AspEndtDocument2MainHandler` computes PREM as `$cond [{$ne: [$PB_ROW.DIFF_PREM_DUE, 0]}, 1, 0]`. If `PB_ROW` is null, the field is missing and `$ne` of a missing value and 0 is true in Mongo, so PREM becomes 1 and `prem2 != 0` opens the gate (inferred from Mongo semantics; no test covers it). Jasper prints nothing when its driver is empty. Investigate: log `prem2`; run the pipeline and see `PB_ROW`; compare the Jasper condition `showDebitNote=Y && PREM != 0`.
- Common mistakes: Assuming a missing value equals 0.
- Follow-up questions: How would you write the guard? (`$ifNull` or check `$PB_ROW` exists.)

**Q6. Symptom: ComplaintRedress appears twice, or not at all, for a Template12 policy.**
- Why the interviewer asks: Multiple rules for the same page.
- Ideal answer: Rules: plain `ComplaintRedress` when `showComplaint=Y` and not BMW/070504/070406 and (not ASP or `ASP_CUTOFF_IND=N`); `ComplaintRedress_SCB` for 070504 (POSMESRA and FWPA get it from PD2 instead); `ComplaintRedress_NoMargin` inside CIPrint for 070412 with cutoff Y and for BMW. To debug: log the booleans `is070504`, `is070412`, `isBmwScheme`, `isPosMesraScheme`, `isFwpaScheme` and `aspCutoffInd`; check `showComplaint` default ("Y"). Check for double rendering by confirming only the merged bean is live (the old PD2/Landscape classes have `@Service` commented out).
- Common mistakes: Not considering the default `showComplaint="Y"`.
- Follow-up questions: Which unit tests cover scheme branches? (`execute_astroScheme070504_mapsAstroFields`, `execute_bmwScheme070504_mapsFirstCertRowAndRunsNestedSubreportsPerRow`, others.)

**Q7. Symptom: ASP add-member pages not in the XML although members were added.**
- Why the interviewer asks: Dispatch counts.
- Ideal answer: Look at the `AddMemberLandscapeDispatch` reportData: `additionPrint`, `deletionPrint`, `sponsorPrint`, `nomineePrint`. If `additionPrint` is 0, check `AspAddMemberLandscapeDispatchHandler.additionExists`: needs `ENDT_CODE` in (72,168) in `UWGE_POLICY_VERSIONS`/`WIP_UWGE_POLICY_VERSIONS` and a risk row with `ACTION_CODE=A` in `UWGE_RISK`/`WIP_UWGE_RISK`. If `CONTRACT_ID` was passed as String the counts are 0 (type mismatch, `safeNum`). If the count is positive but rows are empty, check `UWPL_VI_ENDT_SCHEDULE_TPL4` in `AspAddMemberLandscapeInsuredHandler`.
- Common mistakes: Looking for an error; the dispatch row is always present, even with zero counts.
- Follow-up questions: Why 5-6 queries on every call? What if both WIP and live rows exist? (Oracle would raise ORA-01427; Java returns true.)

**Q8. Symptom: Template12 returns the policy pages but no landscape bands for a policy that should have them.**
- Why the interviewer asks: Suppression by code table.
- Ideal answer: `Template12LandscapeDispatchHandler.fetchDispatch` returns null when `CMGE_CODEREL_DET` has (`GPA_SCHEME`, `CI_LANDSCAPE_PRT`, `Y`) for the scheme; in that case `executeMergedLandscape` prints nothing. Check: the scheme name used by the dispatch (product: WIP, else SB, else `UWGE_POLICY_CONTRACTS`; scheme from `UWPL_POLICY_BASES` VERSION 1 or WIP-not-in-OCP), the `CMGE_CODEREL_DET` entry, then the per-band gates (`BENEFIT_PRINT`, `NOMINEE_PRINT`, `DIAGNOSIS_PRINT`, product/scheme list in the decision matrix).
- Common mistakes: Debugging a single band when the whole dispatch is null.
- Follow-up questions: Which band flags look only at the first joined version row? (`NOMINEE_PRINT`/`DIAGNOSIS_PRINT` - inferred.)

**Q9. Symptom: the QR section is missing from an ASP service tax invoice.**
- Why the interviewer asks: Three-part gate plus API default.
- Ideal answer: Needs `einvConsentInd=Y`, `einvStatus=VALID`, `qrInd=Y`. `qrInd` comes from `PlineEinvoiceQrIndicatorClient.getQrIndicator`, and `safeCall` returns null on failure, which the service defaults to `N`. Investigate: the consent/status values in the `ServiceTaxInvoice` DTO; call the QR indicator in dev; look at the property key (`plines.fn.einvoice.qr.ind.path` or `einvoice.qr.path`; the client's exact key was not verified in the KT); look for ERROR logs; confirm that `QRSection` handler (`PlineGenericQRSectionHandler`) reads `UWGE_GST_TAX_INV` / `WIP_UWGE_GST_TAX_INV`.
- Common mistakes: Only checking consent.
- Follow-up questions: And for Template12? (The request `pTaxInvNo` is used - probably empty unless the caller sends it, inferred.)

**Q10. Symptom: preview (`checklistCopy=Y`) prints a page that final (`N`) does not, or the reverse.**
- Why the interviewer asks: WIP vs live logic.
- Ideal answer: Compare the collections used. Template12 picks `WIP_x` or `x` by flag in Java and the live base query uses `VERSION_NO = 1`; ASP main handler uses `$$chk` in `$lookup` sub-pipelines (WIP lookup only if `Y`, live only if `N`). The service-tax invoice is closed for checklist copy by design. REST functions use mode `WIP` vs `OCP`. For each page, find the gate and the collection pair and compare the two documents. A data difference between WIP and posted is not a bug.
- Common mistakes: Comparing outputs without the same `versionNo`.
- Follow-up questions: What does `SB_` mean? (Not documented - say so.)

---

## B. Type mismatch and conversion

**Q11. Symptom: Template12 returns 500 with `NumberFormatException`. Investigate.**
- Why the interviewer asks: The documented hard failure.
- Ideal answer: The stack trace points to `Long.parseLong` in `Template12LandscapeDispatchHandler.fetchDispatch` (test `fetchDispatch_nonNumericContractId_throwsNumberFormatException`). Cause: `pContractId` empty (default `""`) or non-numeric text. Other handlers use `asNum` (Integer if numeric else the raw string) and would silently match nothing; this one throws. The exception is not caught in `executeMergedLandscape` (test `execute_landscapeDispatchHandlerThrows_notWrappedInTryCatch_propagates`). Fix at the caller; optionally validate and return 400 (`IllegalArgumentException`). If there is no stack trace in the log, remember that `GlobalExceptionHandler` has no logger (see 09a Q7).
- Common mistakes: Catching the exception to hide it.
- Follow-up questions: Which exception types map to 400? Why is `GlobalExceptionHandler` silent?

**Q12. Symptom: a query returns zero rows for a contract that clearly exists. How do you prove a type mismatch?**
- Why the interviewer asks: Mongo's strict typing.
- Ideal answer: (1) Get the pipeline from `MongoTemplate` DEBUG. (2) Run the first `$match` in mongosh with the value as Long, then as String. (3) Check `typeof`/`$type` of `CONTRACT_ID`, `VERSION_NO` in that collection. (4) Check the Java conversion: `safeNum`, `asNum`, `Long.parseLong`. (5) For code tables, check `CODE_CD` type vs the value. Mongo does not raise an error; a mismatch just returns nothing.
- Common mistakes: Re-running the same string filter in mongosh and concluding "no data".
- Follow-up questions: Does Int32 vs Long matter? (Numeric comparison treats them equal; a string does not.)

**Q13. Symptom: `AspServiceTaxInvoiceHandler` throws "conflict" although the live and WIP version numbers look equal.**
- Why the interviewer asks: Subtle Java equality.
- Ideal answer: `requireNoConflict` compares live and WIP `VERSION_NO` with `liveValue.equals(wipValue)`. If one is `Integer` and the other `Long`, `equals` is false even for equal numbers (inferred; KT 03 4.3). Investigate: log the classes of both values; compare with `Number.longValue()`. The existing tests `fetchServiceTaxInvoice_conflictingLiveAndWipVersionNo_throwsIllegalStateException` cover a real conflict only.
- Common mistakes: Assuming a data conflict.
- Follow-up questions: How would you write a test for it? What does `safeNum` return for each type?

**Q14. Symptom: a landscape page throws a 500 only for some policies, mentioning conversion.**
- Why the interviewer asks: `$toDouble` on non-numeric data.
- Ideal answer: Several handlers sort with `$toDouble(COV_SEQ_REF)` (for example `PlineT12ParticularOfInsuredLandscapeAspHandler`, `AspAddMemberLandscapeInsuredHandler`). A non-numeric string makes the aggregation fail (like Oracle's cast error ORA-01722). Investigate: run the aggregation in mongosh and see the error; query the collection for `COV_SEQ_REF` values that do not look numeric; check null/missing as well. Fix in data or with `$convert` with `onError` (inferred suggestion).
- Common mistakes: Catching the exception around the handler only.
- Follow-up questions: Which handlers sort the raw field? (AMC insured and the two diagnosis handlers - the sort can then be lexical.)

**Q15. Symptom: dates in RejectRisk output are empty, although the row has a date.**
- Why the interviewer asks: Swallowed conversion.
- Ideal answer: `RejectRiskService.toLocalDateTime` calls `DateUtils.toLocalDateTime`/`toLocalDate` and catches everything, returning null. DTO dates are `String` (BSON `Date` converted by Spring, inferred `Date.toString()` style such as "Mon Jan 05 08:30:00 UTC 2026"). If the zone abbreviation cannot be parsed by pattern `EEE MMM dd HH:mm:ss z yyyy`, the date silently becomes null. Investigate: log the raw DTO string; test `DateUtils` with it; try a different JVM zone. Safer: project `$dateToString` with `timezone: Asia/Kuala_Lumpur` so the DTO gets `yyyy-MM-dd`.
- Common mistakes: Searching the renderer.
- Follow-up questions: Which service change moved to the Plines `DateUtils`? (Uncommitted hunk 3; also `com.agcs.fo.opus.plines.utils.DateUtils`.)

**Q16. Symptom: `ROUNDED_NETT_PREM` is 1 cent different from the Jasper/Oracle value.**
- Why the interviewer asks: Rounding semantics.
- Ideal answer: `AspDebitNoteHandler` uses `$round` (half to even in Mongo); Oracle `ROUND` is half away from zero. Only exact .5 boundaries differ. Check the unrounded `NETT_PREM`; compute with both rules; if confirmed, round in Java with `RoundingMode.HALF_UP`. Not tested in the repo (inferred).
- Common mistakes: Changing the scale of the BigDecimal in the payload.
- Follow-up questions: Where else is rounding used? (GPA refund = round(match - gross - stax - stamp) in the schedule - see KT.)

---

## C. Date handling

**Q17. Symptom: RejectRisk returns 500 for `pCreatedDateFrom="2026-01-05T10:00:00"`. Investigate.**
- Why the interviewer asks: Accepted formats.
- Ideal answer: `DateUtils.toLocalDate` accepts `yyyy-MM-dd`, `dd-MM-yyyy`, `dd/MM/yyyy`, `MM/dd/yyyy` (only reached if dd/MM fails), `dd-MMM-yyyy`, `dd-MMM-yy`, `EEE MMM dd HH:mm:ss z yyyy`, and ISO offset/zoned. A local date-time without offset is not accepted, so `toDate` in the handler throws and the request is 500. Check by calling `DateUtils.toLocalDate` with the string in a test. Fix at the caller, or extend `DateUtils` after discussion.
- Common mistakes: Assuming ISO local date-time works.
- Follow-up questions: What does `2026-13-45` throw? (`DateTimeParseException`, not `IllegalArgumentException`: 500, not 400.)

**Q18. Symptom: a RejectRisk row created on the last day of the range is missing. Which build is running?**
- Why the interviewer asks: Version confusion between branches.
- Ideal answer: Open `RejectRiskHandler` of the deployed build. If the `$match` has `$lte` with `dateTo` at start of day, it is the old logic (HEAD) and the fix is missing. The fixed logic adds one day and uses `$lt`. Confirm by the pipeline in the DEBUG log. The fix is uncommitted here and equals `cc6a1b84d` on the OPUSM-41763 branch; check whether your build includes it.
- Common mistakes: Writing a second fix without checking.
- Follow-up questions: What tests assert the old shape? (`fetchRejectRisks_dateInstances_usedDirectlyInMatch`.)

**Q19. Symptom: after the fix, a record at 00:30 Malaysia time on day D shows up in the report for D-1.**
- Why the interviewer asks: Time zone boundary.
- Ideal answer: `toDate` creates UTC midnight (`ZoneId.of("UTC")`). If `CREATED_DATE` holds real instants (stored as UTC), 00:30 MYT = 16:30Z the previous day, so it falls in D-1's UTC window. Other handlers in the repo format with `timezone: Asia/Kuala_Lumpur` (for example `GenericProjectCodeReportHandler`), which suggests instants (inferred). Investigate with one known record: compare the stored BSON instant with the Oracle value. If instants, use `Asia/Kuala_Lumpur` in `toDate`. Not verified on data.
- Common mistakes: "Fixing" by adding hours without testing.
- Follow-up questions: How would you seed a test? (A record at 2026-01-04T16:30:00Z and query day 2026-01-05.)

**Q20. Symptom: the print engine's validator complains about `dd/MM/yyyy` dates.**
- Why the interviewer asks: Date contract mismatch.
- Ideal answer: Every `LocalDateTime` in the payload classes has `@JsonFormat(pattern="dd/MM/yyyy")` with `LocalDateTimeSerializer`, so the wire value is `06/06/2017`. `Pline.xsd` types the 64 date leaves as `xs:dateTime` (`2017-06-06T00:00:00`). Investigate: find the offending leaf; check the Java field annotation. The options: emit ISO, change the XSD to a pattern type, or add parallel string leaves. Decision is a contract decision with the report team; Jasper formats `dd/MM/yyyy` itself (for example `pBeforeEffDate` comes from `SimpleDateFormat("dd/MM/yyyy")`).
- Common mistakes: Changing all annotations without asking the consumer.
- Follow-up questions: Which date leaves does RejectRisk populate? (`createdDate`, `revokeDate`, `updateddate`, plus `pCreatedDateFrom/To`.)

**Q21. Symptom: in Template12 some date strings in `transDet` have a different format than in `reportParameters`.**
- Why the interviewer asks: Reflective date reformat.
- Ideal answer: The service reformats every `String` field whose name contains `date` or `dob` to `dd/MM/yyyy` at the end (`S12:2105-2230`, applied `S12:549-554`) because `toLocalDateTime` strips the time. `LocalDateTime` fields are written by Jackson with the annotation. So two paths exist. Jasper passes `issueDate` as `dd-MMM-yyyy` text to sub-reports. Investigate with a field-by-field list of the type and the code path for the failing leaf.
- Common mistakes: Searching for one date formatter.
- Follow-up questions: What is the XSD type for those leaves? (`xs:dateTime`.)

**Q22. Symptom: `issueDate` is blank for ASP, and the company BRN is missing in the same report.**
- Why the interviewer asks: Linking two symptoms to one cause.
- Ideal answer: `fnGetCompBrnNumber` normalises `issueDate` to ISO and, if it is blank or unparseable, skips the call and returns null (KT 06 table row 5). So a blank issue date from the main handler leads to a missing BRN. Investigate the main DTO's `issueDate` first (the handler date format `yyyy-MM-dd` from `formatDate`, lines ~1659), then the API.
- Common mistakes: Treating BRN as an API problem.
- Follow-up questions: Which handlers pass the issue date? (`CompanyNameWithGSTRegNo`, `Signature`.)

---

## D. reportParameters, whitelist and dropped values

**Q23. Symptom: the service sets `rp.setPcontractId(...)` but the XML has no `pContractId`.**
- Why the interviewer asks: The `@JsonIncludeProperties` whitelist.
- Ideal answer: `PlineReportParameters` has a whitelist of 73 names. Properties set via the base class that are not in the whitelist are silently dropped: `pContractId`, `pPolicyNo`, `pversionNo`, `pendtNo`, `prefNo`, `puserId`, `pcompanyParamType`, `pissueDate`, `psignIssueDate`, `reportName`, `pshowGst/Sst/Tran`, `showPaymentInfo`. Investigate: compare the setter's JSON name with the whitelist; serialise the object in a test and read the output. This is by design to stop the Motor/Fire/Bond fields leaking into Pline XML.
- Common mistakes: Adding a field to the Java class and expecting it in the XML.
- Follow-up questions: Which six whitelisted names are not in v0.1.6? (`pCreatedDateFrom`, `pCreatedDateTo`, `pReportTitle`, `pshowEndtSchedule`, `pShowOutPrintCovLetter`, `pShowPDPANotice`.)

**Q24. Symptom: `ShowOutPrintCovLetter=Y` in the request has no effect on the XML parameters.**
- Why the interviewer asks: Case and setter spelling.
- Ideal answer: The service reads the key `ShowOutPrintCovLetter` (capital S) to choose `OutSourcePrintCoverLetter` vs `PrintSeriesFooter` (test `execute_showOutPrintCovLetterY_usesOutSourceCoverLetterBranch`). But the report parameter setter is commented out, and the whitelisted name is `pShowOutPrintCovLetter`, so the echo is not emitted (KT 02 D15). The behaviour is in the pages, not in `reportParameters`. Check the request key spelling too: `showOutPrintCovLetter` with a small s is a different key and defaults to "N".
- Common mistakes: Looking for the value in the XML parameters.
- Follow-up questions: What other request keys are read but not emitted? (`pMstPolicyNo`, `pContractIdList`, `pCertNoFrom/To`, `pTaxInvNo`.)

**Q25. Symptom: `pCreatedDateFrom` appears in the XML but the XSD validator says it is an unknown element.**
- Why the interviewer asks: Java ahead of XSD.
- Ideal answer: The Java whitelist and `PlineReportParameters` have `pCreatedDateFrom/To` (comment "XSD v0.1.9"), but v0.1.6 (the only XSD found) has no such leaf. Same for `pReportTitle`. Grep the XSD to prove it. The mapping is as designed; the contract has to catch up. Report as a contract gap, not a code bug.
- Common mistakes: Deleting the parameter to pass validation.
- Follow-up questions: Where is v0.1.9? (Not found on the machine.)

**Q26. Symptom: ASP GST/SST label flags are not visible in `reportParameters`.**
- Why the interviewer asks: Setter vs property naming.
- Ideal answer: ASP calls `setPshowGstLabel/setPshowSstLabel` (base setters). The base properties `pshowGstLabel/pshowSstLabel` are in `@JsonIgnoreProperties`, while Template12 additionally calls the upper-case `setPShowGstLabel/SstLabel` that map to whitelisted `pShowGstLabel/pShowSstLabel`. So ASP's flags probably do not reach the XML (inferred, not run). Investigate: serialise `PlineReportParameters` after the ASP setters in a test and see. The master row still carries `showSstLabel` in `transDet`.
- Common mistakes: Reporting it as confirmed.
- Follow-up questions: Where else can a renderer get the label? (`transDet.showSstLabel` on the master row.)

**Q27. Symptom: ASP landscape parameters (`pReportTitle`) are missing or overwritten.**
- Why the interviewer asks: One `reportParameters` per transaction.
- Ideal answer: The XSD allows one `reportParameters` per `transaction`. The old landscape service created two parameter sets; in the merged bean only one set exists, and `pReportTitle` is not in v0.1.6. Investigate: find who calls `setPReportTitle` and what the final `Transaction.reportParameters` holds; per-reportData values must go on `reportData` or `paramGroup` instead.
- Common mistakes: Expecting per-reportData parameters.
- Follow-up questions: What is `paramGroup`? (Generic `dataType/paramCode/paramValue` carrier.)

**Q28. Symptom: `parentName` in `reportParameters` has the wrong value for a policy that has Landscape and PD2 data.**
- Why the interviewer asks: Precedence rule.
- Ideal answer: Report-level leaves (`parentName`, `planDescp`, `psi`, `pmtDueDate`...) are set with precedence PolicyDocument > Landscape > PD2 (test `execute_parentNamePrecedence_policyDocumentWinsOverLandscapeAndPolicyDocument2`). Find which band sets the first non-null value; PD2 uses the first row of `ParticularOfInsuredTpl4`/`Base`/`InstallmentDetails`. If PolicyDocument has null, Landscape wins, then PD2.
- Common mistakes: Blaming PD2.
- Follow-up questions: Where are the merged values set? (`mergeSingleTransDet` and parameter code in `Template12PolicyDocumentService`.)

---

## E. External API calls swallowed

**Q29. Symptom: a blank company BRN on the invoice only in one environment.**
- Why the interviewer asks: Silent failure path.
- Ideal answer: `PlineUtilsClient.fnGetCompBrnNumber` catches exceptions, logs `fnGetCompBrnNumber url={} failed: {}` at ERROR and returns null. So HTTP 200 comes out with a blank. Steps: grep that log line; read the URL (base + `plines.fn.comp.brn.number.path`); call it with curl from the pod network; check truststore and timeouts (8 s `fastLookupRestTemplate`); verify `API_BASE-URL` points to the right host.
- Common mistakes: Looking at the response status.
- Follow-up questions: Which other key has the same path? (There are 8 keys for comp-brn-number.)

**Q30. Symptom: a request parameter reaches the external API as an empty value.**
- Why the interviewer asks: `sanitize()` side effect.
- Ideal answer: `PlineUtilsClient.sanitize` allows only `[A-Za-z0-9_\-./#]`; anything else is replaced with "" without error. A contract/endt/ref value with a space or other character reaches the API blank, and the API answers with a default. Investigate: log the sanitised query; compare with the input. `PlinePolicyRefClient` is stricter (`[A-Za-z0-9_\-]+`, returns null otherwise).
- Common mistakes: Debugging the API.
- Follow-up questions: What does `assertTrustedHost` do? What happens to the `SecurityException`?

**Q31. Symptom: 500 on every request that needs an external call after a base URL change.**
- Why the interviewer asks: SSRF guard interaction.
- Ideal answer: `assertTrustedHost(url)` requires the URL to start with the configured base. In `safeGet` the `SecurityException` is thrown outside the try, so it propagates as 500 (KT 06 2.8). A mismatch (trailing slash handling, a path built from another base) triggers it. Investigate: print base and built URL; check `API_BASE-URL` trailing slash (the file has one; `url()` strips it) and `plines.api.base-url` (a different class, `Pline*Client`).
- Common mistakes: Assuming network errors.
- Follow-up questions: Which clients use which base key?

**Q32. Symptom: external calls hang for a long time and the report returns after about 20 s with missing values.**
- Why the interviewer asks: Timeout reasoning.
- Ideal answer: The primary `restTemplate` used by `PlinePolicyRefClient`, `PlineSSTRolloutDateClient`, `PlinePgRpgeUtilsClient` has 5 s connect and 20 s read; `PlineUtilsClient` uses `fastLookupRestTemplate` with 8 s read. Pool is 100 total/20 per route, connection-request timeout 3 s (Pline beans only; motor `restTemplate` has default pool - inferred). Investigate which client is slow from the log call sequence; check pool exhaustion (3 s wait errors); there is no retry or circuit breaker.
- Common mistakes: Raising all timeouts.
- Follow-up questions: Which motor clients create `new RestTemplate()` with no timeout? (`PolicyRefClient`, `PMEGetContractIdClient` - not used by the three modules.)

**Q33. Symptom: startup fails with `Could not resolve placeholder`.**
- Why the interviewer asks: Property keys without defaults.
- Ideal answer: Required keys with no default include `opus.fn-decide-mt-cert-path` and `opus.fn-uwge-get-endtschd-query-path` (read by `PlineUtilsClient`), `plines.api.base-url`, `app.ssl.trust-store` and password. Check the effective config; also note `API_BASE-URL` with a hyphen needs a matching override (`API_BASE_URL` expected by relaxed binding - inferred, untested). Compare with a known-good file; look at the first failing bean in the startup log.
- Common mistakes: Editing random properties.
- Follow-up questions: Which key has a code default but no definition? (`plines.api.customer.signature.type` defaults to `UW`.)

**Q34. Symptom: the signature or date-constants lookup returns nothing after a backend move.**
- Why the interviewer asks: Path keys.
- Ideal answer: Check the path keys: `plines.fn.rppm.get.signature.path`, `plines.fn.date.value.constants.path` (the properties comment says the path was derived/guessed: "verify if it starts 404'ing"). Failures are swallowed and logged as ERROR; the value is null. Call the URL in dev; if 404, update the key; look for duplicates of the same path in other keys (many exist), so edit the one the code reads (search `@Value`).
- Common mistakes: Editing a duplicate key that nobody reads (for example `external.pme.base-url`).
- Follow-up questions: Which keys are unread? (`pg-rpge-utils.base-url` and the dev1 group.)

---

## F. Unknown report name and routing

**Q35. Symptom: HTTP 200 SUCCESS, XML has only `templateDetails`/`mailingDetails`. How do you confirm it is the unknown-name case?**
- Why the interviewer asks: Distinguish from "no data".
- Ideal answer: Check `transactionDetails`: a default transaction built by `createDefaultTransaction` has no `transactionDetails`/`reportData`. Then compare `reportName` exactly with the bean names; after normalisation (leading `/` and `reports/` removed, case-insensitive per the KT). If no bean and no switch case match, the `default:` branch (CommonController ~393) creates the default transaction. A real strategy result with zero rows instead gives an envelope with an empty `reportData` structure (the payload defaults add one empty element, see 09e).
- Common mistakes: Assuming a 404.
- Follow-up questions: What if `strategy.execute` returns a Transactions with null `Transaction`? (The controller also replaces it with the default.)

**Q36. Symptom: the same request works with `AGIC/UW/...` but not with `reports/AGIC/UW/...`.**
- Why the interviewer asks: Name normalisation.
- Ideal answer: The controller normalises by stripping a leading `/` and `reports/` and compares case-insensitively (KT 06 2.2). If a caller adds other prefixes, spaces or different casing in the path segments, bean lookup `reportStrategies.get(reportName)` fails (exact key). Also check the `metadata.reportName` vs `filterCondition.reportName` - routing uses `filterCondition.reportName`. Test by sending the exact bean name.
- Common mistakes: Editing the controller.
- Follow-up questions: Where is the key read? (`filterCondition`.)

**Q37. Symptom: after re-enabling an old service class the app fails at startup or every band prints twice.**
- Why the interviewer asks: Merge side effects.
- Ideal answer: The old classes are kept as reference with `@Service` commented out (`Template12PolicyDocument2Service`, `Template12LandscapeService`, `AspEndtDocument2Service`; landscape class has its `@Bean` methods commented). Re-enabling would register extra strategies (ASP: clash of bean keys per the javadoc; Template12: duplicate bands because the merged bean already includes them). Check `git diff` for `@Service`/`@Bean` and the startup log for `ConflictingBeanDefinition`/override errors (inferred wording). Revert the annotation.
- Common mistakes: Calling it a Spring bug.
- Follow-up questions: Why do tests of these dead classes still pass? (They instantiate the classes directly.)

**Q38. Symptom: `reportName` missing from the body produces 500.**
- Why the interviewer asks: Null routing.
- Ideal answer: `switch(reportName)` on null throws `NullPointerException` (inferred, not run), and the global handler returns 500. Also `filterCondition` missing can cause NPE in services before `getOrDefault`. Investigate by sending the same JSON in dev; check `CommonController` normalisation lines. Do not "fix" with a default report name.
- Common mistakes: Defaulting to some report.
- Follow-up questions: Which exception would be 400? (`IllegalArgumentException`.)

---

## G. Merge side effects in the response

**Q39. Symptom: one scalar value in `transDet` is overwritten by a later band.**
- Why the interviewer asks: Reflection merge.
- Ideal answer: `mergeSingleTransDet` copies non-null fields onto every existing `transDet` (`B12:189-258`, last non-null wins), by reflection. A band that sets the same leaf later replaces the earlier value. Multi-row bands use `addSingleTransDet` and do not merge. Investigate: find all bands that set the leaf; the log line `mergeNonNullFields: failed to copy field '{}': {}` shows reflection failures; use a unit test that runs two bands. Commits `7399d2fc1` and `c0c6d449a` relate.
- Common mistakes: Searching for a single setter.
- Follow-up questions: What happens to `endorsementDetails`? ("last call wins".)

**Q40. Symptom: in ASP, landscape rows show the same `endorsementDetails` values as the master row, not their own.**
- Why the interviewer asks: Shared object by reference.
- Ideal answer: `prntDetails`, `endorsementDetails` and `transDetList` are built once; `landscapeRow` reuses master `branchInfo`/`compInfo` and the same `endorsementDetails` object (lines 1566-1574), and `mergeEndorsement` is last-non-null-wins. So each row sees the final merged values (inferred from code; check before claiming). Investigate by comparing object identity in a test and mutating one row.
- Common mistakes: Believing each reportData has its own object.
- Follow-up questions: How would you fix? (Copy per row.)

**Q41. Symptom: Template12 PolicyDocument returns 500 with "Ambiguous ... ORA-01427" and PD2 for the same contract works.**
- Why the interviewer asks: Three primary fetches.
- Ideal answer: The PolicyDocument primary (`PlinePolicyDocumentMainHandler`: `requireNoConflict`, `distinctValueOrConflict`) throws `IllegalStateException` when WIP and posted product/scheme differ (mimics ORA-01427). PD2 uses `PlineT12PolicyDocumentHandler` with `$ifNull` precedence, no conflict check. Investigate: read the exception message for the field; query the WIP and posted collections for the contract: `WIP_UWGE_POLICY_CONTRACTS`, `SB_UWGE_POLICY_CONTRACTS`, `WIP_UWPL_POLICY_BASES`, `UWPL_POLICY_BASES (TOP_INDICATOR='Y')`. The fix is data. Do not change the exception to silence it without sign-off (design note CRITICAL FACT #1).
- Common mistakes: Making fetch 1 tolerant.
- Follow-up questions: Why would Oracle itself have failed? (Scalar sub-select with UNION returning 2 rows.)

**Q42. Symptom: product 070412 policies show the Blank_NoMargin page on even pages (duplex).**
- Why the interviewer asks: Page-number-dependent logic.
- Ideal answer: Jasper uses `$V{PAGE_NUMBER}` parity in the print condition (`(PAGE_NUMBER-1)%2 != 0 && ASP_CUTOFF_IND=="Y" && 070412`). Java has no page numbers; it adds the blank whenever `070412 && cutoff=Y`, with a code comment "may over-include" (`S12:1669-1676`). Same for PD2 Q1/Q5 and `PrintSeriesFooter` call 2. So the renderer must do parity. Investigate by checking the rows emitted in `Blank_NoMargin` and what the renderer does.
- Common mistakes: Trying to compute page parity in Java.
- Follow-up questions: Which bands are NOT IMPLEMENTED for this reason?

---

## H. Aggregation behaviour

**Q43. Symptom: an aggregation returns 0 rows with no error. How do you take the pipeline out of the app?**
- Why the interviewer asks: Practical Mongo debugging.
- Ideal answer: Turn on `logging.level.org.springframework.data.mongodb.core.MongoTemplate=DEBUG` (already set in `application.properties`) and read the pipeline; or in a scratch unit test capture the `Aggregation` and call `toPipeline(Aggregation.DEFAULT_CONTEXT)`. Run it in mongosh stage by stage (use `$limit` or run the first N stages) to find where rows disappear: `$match`, `$lookup` result array empty, post-lookup `$match`. Keep PII out of shared tickets.
- Common mistakes: Running the final pipeline only.
- Follow-up questions: Which stages drop rows in RejectRisk? (date `$match`, optional `$match`, name/ID `$match` after lookup.)

**Q44. Symptom: RejectRisk with only `pIdNo` is very slow.**
- Why the interviewer asks: Filter order and index use.
- Ideal answer: The ID filter is applied after the `$lookup` into `CPGE_VI_PARTNERS_ALL`, so without dates every `CMUW_PL_REFER_RISK` row is looked up first. With the new behaviour, no dates means no date filter. Investigate: `explain` on the first stage; count rows before the lookup; check indexes on `CPGE_VI_PARTNERS_ALL (PART_ID, VERSION)`. Possible improvements: filter partners first when name/ID is given (inferred), or require dates.
- Common mistakes: Adding `$limit` and hiding the problem.
- Follow-up questions: Does `$lookup` with `let` + `$expr` use an index? (Can, in recent MongoDB; verify with explain.)

**Q45. Symptom: "Sort exceeded memory limit" for a large RejectRisk range.**
- Why the interviewer asks: Memory limits.
- Ideal answer: `$setWindowFields` with sortBy and the final `$sort` are blocking stages with a 100 MB limit each unless `allowDiskUse`; `RejectRiskHandler` sets no `AggregationOptions` (unlike `PlinePolicyDocumentMainHandler`, which sets `allowDiskUse` and `maxTime`). Not seen in the repo as a real error (inferred). Investigate with the date range and row count; options: `allowDiskUse(true)`, narrower range, sort in Java.
- Common mistakes: Raising JVM heap.
- Follow-up questions: Should `maxTime` also be set? (The 600 s used elsewhere is long compared with caller timeouts.)

**Q46. Symptom: RejectRisk `rowNumber` starts with blank-name customers.**
- Why the interviewer asks: Null-first sort.
- Ideal answer: Mongo ascending sort puts null/missing first, `$documentNumber` follows that order. Oracle puts them last. Investigate: filter output where `customerName` is null and see their `rownumber`; compare with the Jasper report for the same range. Fix with a null-flag sort key in both `sortBy` and `$sort`, or sort in Java with `Comparator.nullsLast` and assign row numbers there. Add a tie-breaker (for example `createdDate`, `_id`).
- Common mistakes: Assuming a deterministic order for equal names.
- Follow-up questions: Which test locks the current sort? (`..._finalSortStage_sortsByCustomerNameAscending`.)

**Q47. Symptom: the same Template12 request returns different version-based data between WIP and live although `versionNo` is the same.**
- Why the interviewer asks: Hard-coded version rule.
- Ideal answer: For non-WIP, the base query sets `VERSION_NO = 1` (`PlinePolicyDocumentMainHandler`: `if (!isWip) baseCriteria.and("VERSION_NO").is(1)`), while the request's `versionNo` is used for the print series footer and QR section. Landscape queries lack some version filters (`WIP_UWGE_COVER`/`UWGE_COVER` have none; `UWPL_RISK_PERSON` has no `VERSION_NO=1`), so older version covers can be included (KT 02 D7). Investigate: print the version fields on returned documents.
- Common mistakes: Assuming `versionNo` filters everything.
- Follow-up questions: Which leaf is affected? (`promoAmt`, the 070406 band L16.)

---

## I. Tests as a debugging tool

**Q48. Symptom: after the RejectRisk change, the handler tests fail in odd ways (NPE in test code). Investigate.**
- Why the interviewer asks: Test brittleness.
- Ideal answer: Tests read `pipeline.get(1).get("$match")` by stage index and assert `$lte`/null bounds. After the change the first stage is the optional-filter `$match`, or there is no date stage. Static reading says 7 of 15 fail (not run). Fix: find the stage by content (the one containing `CREATED_DATE`), assert `$gte`/`$lt` and that a missing bound is absent. Keep the tests that pass by accident under review (`..._noOptionalFilters_noSecondMatchStageAdded`).
- Common mistakes: Updating the expected indices only.
- Follow-up questions: How would you check the test actually fails when `$lt` is changed back to `$lte`? (Manual mutation.)

**Q49. Symptom: tests for `AspServiceTaxInvoiceHandler` pass, but production behaviour differs.**
- Why the interviewer asks: Class shadowing.
- Ideal answer: A near-copy `AspServiceTaxInvoiceHandler.java` (483 lines) exists in `src/test/java/.../documentHandler` with the same package and name as the production class. On the usual Maven test classpath test classes come first, so the 27 tests probably test the copy (inferred, not verified). Investigate: in a test print `AspServiceTaxInvoiceHandler.class.getProtectionDomain().getCodeSource()`; diff the two files (17 changed lines, for example `catch (Exception e)` vs `catch (RuntimeException e)`). Delete the copy and re-run.
- Common mistakes: Trusting coverage numbers.
- Follow-up questions: Why does JaCoCo still show the production class partly covered or not?

**Q50. Symptom: a Mockito test passes but a stub is never used. How do you find dead stubs here?**
- Why the interviewer asks: Lenient mocks.
- Ideal answer: `@MockitoSettings(strictness = Strictness.LENIENT)` is used in 36 of the 91 in-scope test files, so unused stubs are never reported. Temporarily switch the class to strict stubs (default `MockitoExtension` strictness) and run; unnecessary stubbing errors point to dead stubs and changed behaviour. Also watch `forcePipelines(captured)` helpers, which only run lambda bodies for coverage and assert nothing (for example `fetchRejectRisks_pipelineLambdas_fullyForced`).
- Common mistakes: Removing LENIENT in all tests at once.
- Follow-up questions: What does `verify(` count tell you? (116 uses in 9 files: most tests check outputs only.)
