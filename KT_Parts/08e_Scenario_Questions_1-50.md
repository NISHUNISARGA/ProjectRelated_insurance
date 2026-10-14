# 08e - Scenario Questions 1-50 (RejectRisk, ASP, Template12)
> **AUTHORSHIP UPDATE (confirmed by the developer): git author `GS <extern.gs_nisarga@allianz.com>` is the developer. Wherever this file says commits cannot be attributed to the developer, read it as: the 11 GS commits listed in `KT_Parts/14_My_Commits_And_Ownership.md` ARE theirs; commits by other authors still must not be claimed. No GS commit touches `src/test`, so test authorship is still `[CONFIRM]`.**

"What would you do if ..." questions for THIS system. Facts come from the KT guide and KT_Parts 01-06; "(inferred)" = reasoning, not run; "[CONFIRM: ...]" = personal fact. Answers are action plans: check, decide, fix, test, communicate.

Order in this file: Q1-17 RejectRisk, Q18-34 ASP, Q35-50 Template12.

---

### Q1. The business says RejectRisk is missing rows created on the "to" date. What do you do?
- **Why the interviewer asks:** Checks that you can trace a symptom to the known date bug.
- **Ideal answer:** First check which code is deployed: the old `RejectRiskHandler` used `$lte dateTo` with a midnight value, so rows created later on the last day (a full timestamp) were lost; from = to returned only exact-midnight rows. Reproduce in Compass with one known record at 15:30 on the end date. The fix is `[from 00:00, to+1 day 00:00)` with `$gte` and `$lt` (equal to Oracle `TRUNC(CREATED_DATE) BETWEEN`). Write boundary tests (D 00:00:00, D 23:59:59.999, D+1 00:00:00), check the time zone (Q5), and tell the business which build has the fix.
- **Common mistakes:** Changing `$lte` to `$lte endOfDay` in a local zone without checking the data zone.
- **Follow-up questions:** Why not `$dateTrunc`? (Blocks index use.)

### Q2. A caller sends no dates to RejectRisk. What should happen and what do you do?
- **Why the interviewer asks:** Behaviour change vs Oracle.
- **Ideal answer:** Facts: Oracle `BETWEEN NULL AND NULL` returns zero rows; HEAD returned zero (by accident, `$gte: null`); the working tree returns ALL rows (unbounded scan, big response). I would not decide alone. Ask the business: keep parity (return empty), require at least one or both dates (400), or cap rows. Until answered, I would keep parity or require dates because "all rows" with no auth and no paging is a risk. Implement with a flag or validation in `RejectRiskService`, and add tests for none/only-from/only-to.
- **Common mistakes:** Shipping "all rows" silently.
- **Follow-up questions:** How do you protect memory if all rows are allowed? (`allowDiskUse`, row cap, paging.)

### Q3. Users complain that unknown customers appear first with number 1. What is the cause and the fix?
- **Why the interviewer asks:** NULL ordering parity.
- **Ideal answer:** Oracle sorts NULL last; Mongo sorts null/missing first. Rows with no partner match or null `NAME_EXT` become row 1..k. Fix: add a null flag (`_nullName`) and sort by `{_nullName:1, customerName:1, createdDate:1, _id:1}` in both `$setWindowFields.sortBy` and `$sort`; or sort in Java with `Comparator.nullsLast` and assign `rowNumber`. Update the test that locks the plain sort (`fetchRejectRisks_finalSortStage_sortsByCustomerNameAscending`), add a test with a null-name row, and confirm with the report team how they use `rownumber` (Jasper's "NO" column used its own counter).
- **Common mistakes:** Fixing only the final `$sort`.
- **Follow-up questions:** Ties? (Add a stable tie-breaker.)

### Q4. The report team says they cannot treat `reportData[]` entries as independent print rows. What now?
- **Why the interviewer asks:** The disclaimer risk becomes real.
- **Ideal answer:** The service has a disclaimer comment for exactly this. Options: (a) put all rows in one `reportData` and one `transDetList` with many `transDet`, but then `remarks` (sibling `companyDesc`) has only one slot per list - ask for an XSD change (remarks in `transDet`); (b) keep per-row `reportData` and ask them to iterate; (c) aggregate by a key they define. Get the exact consumer rule in writing, check the Jasper layout (it printed rows from the detail band, "NO" from `$V{GroupByBranch_COUNT}`), update the mapping sheet and tests, and bump the schema if needed.
- **Common mistakes:** Changing the payload shape without the XSD owner.
- **Follow-up questions:** What else depends on this shape? (Template12 landscape rows, ASP landscape rows.)

### Q5. A user in Kuala Lumpur says records created on 5 Jan early morning appear in the 4 Jan report. What do you check?
- **Why the interviewer asks:** Time zone thinking.
- **Ideal answer:** `toDate` makes UTC midnight. If `CREATED_DATE` is a true instant, 2026-01-05 00:30 MYT is 2026-01-04T16:30Z, so it falls in the 4 Jan UTC window (inferred; other handlers use `timezone: Asia/Kuala_Lumpur`, suggesting true instants). Steps: fetch one record in Compass and compare with the Oracle value; decide the business day (Malaysia); change `toDate` to `ZoneId.of("Asia/Kuala_Lumpur")` for both bounds; project dates with `$dateToString` and timezone; add a test with `2026-01-04T16:30:00Z`.
- **Common mistakes:** Changing the zone without confirming how the data is stored.
- **Follow-up questions:** DST? (Malaysia has none.)

### Q6. You run the tests after the RejectRisk fix and 7 of 15 handler tests fail. What do you do?
- **Why the interviewer asks:** Maintenance discipline.
- **Ideal answer:** Expected (static reading): they assert `$lte`, null bounds and stage indexes. I would not delete them. Rewrite by content: find the stage that has `CREATED_DATE` and assert `$gte` and `$lt` (+1 day); a test for "no date stage" when no bounds; for type/reason filters find the `$match` that has `REJ_TYPE`/`REASON_CODE`; the "no extra `$match`" test counts `$match` stages without the date one. Add the new cases (only-from, only-to, +1 day, from = to). Keep `RejectRiskServiceTest` (4 tests) as is. Run the full class, then the suite.
- **Common mistakes:** Changing assertions to pass without understanding why they failed.
- **Follow-up questions:** Which tests pass by accident? (`noOptionalFilters_noSecondMatchStageAdded`.)

### Q7. A colleague asks you to commit `application.properties` with your local Mongo URI so "it works for everyone". What do you do?
- **Why the interviewer asks:** Secret handling under pressure.
- **Ideal answer:** Refuse politely and explain. The uncommitted diff has `spring.mongodb.uri=<redacted>` with user and password; `gitleaks.toml` allowlists both properties files, so CI would not stop it. Use `SPRING_MONGODB_URI`/`SPRING_MONGODB_DATABASE` or `--spring.mongodb.uri` locally; `git restore src/main/resources/application.properties` or `git add -p` the code files only. Share a template (placeholder) via a doc if needed.
- **Common mistakes:** "It is only dev data": the DB name `OPUSAGICPRODR1` suggests production-like data (inferred).
- **Follow-up questions:** What if it is already pushed? (See Q8.)

### Q8. You find that the Mongo credential was pushed to a remote branch. What are the steps?
- **Why the interviewer asks:** Incident response basics.
- **Ideal answer:** (1) Tell the security contact and the DB owner immediately (existing tracker OPUSM-50404). (2) Rotate the credential first; history rewrite alone is not enough. (3) Check where else it was used. (4) Remove it from the branch and, if policy requires, purge history. (5) Remove the whole-file gitleaks allowlist entries (review-by 2026-10-23) and move secrets to env/secret store. (6) Add a pre-commit scan. Do not paste the secret in chat or tickets.
- **Common mistakes:** Deleting the line and calling it done.
- **Follow-up questions:** How to find exposure time? (Commit and push times; remote logs.)

### Q9. RejectRisk is slow when only `pCustomerName` is sent. What do you do?
- **Why the interviewer asks:** Pipeline performance.
- **Ideal answer:** Cause: the name/ID `$match` runs after the partner `$lookup`, so with no date range every refer-risk row is looked up (4 lookups per row) before filtering. Steps: run `explain("executionStats")`, check COLLSCAN and `totalDocsExamined`; add indexes (`CMUW_PL_REFER_RISK {CREATED_DATE}`, `CPGE_VI_PARTNERS_ALL {PART_ID, VERSION}`, `CMGE_CODE {CAT_CODE, CODE_CD}`); if name/ID is given without dates, first query `CPGE_VI_PARTNERS_ALL` for matching ids and `$match` `CP_PART_ID $in` early; require a date range; consider an index on `NAME_EXT`. Confirm whether `*_VI_*` is a view (no own index).
- **Common mistakes:** Adding indexes in the repo (none are managed there).
- **Follow-up questions:** Who creates the indexes? (DBA/Atlas; recommendations only.)

### Q10. Production logs show "Sort exceeded memory limit" for RejectRisk. What do you do?
- **Why the interviewer asks:** Aggregation limits.
- **Ideal answer:** `$setWindowFields` and `$sort` are blocking with a 100 MB per-stage limit; `RejectRiskHandler` sets no `allowDiskUse`. Short-term: add `AggregationOptions` with `allowDiskUse(true)` and a sensible `maxTime`. Better: require/limit the date range, cap the number of rows, and sort and number in Java after reducing the data. Ask why the request is so big (probably the no-dates case). Add a test that the options are set.
- **Common mistakes:** Only raising memory.
- **Follow-up questions:** Does `allowDiskUse` fix performance? (It prevents the failure, not the slowness.)

### Q11. The partner view has two rows for one (PART_ID, VERSION). What happens and what do you do?
- **Why the interviewer asks:** Duplicate data semantics.
- **Ideal answer:** Oracle's outer join would return two rows (duplicate report lines). Our `$lookup` uses `$limit 1`, so we show one row with an arbitrary partner and hide the data problem. Check with a `$group` count in Compass whether duplicates exist. If they exist, ask the data owner; decide which partner is correct (add a deterministic `$sort`) or replicate Oracle (remove `$limit` and `$unwind`). Add a test with two partner rows.
- **Common mistakes:** Assuming the view is unique without checking.
- **Follow-up questions:** Same question for `CMGE_CODE` duplicates? (Oracle raises ORA-01427; Mongo hides it.)

### Q12. The business asks for case-insensitive customer name search in RejectRisk. How do you approach it?
- **Why the interviewer asks:** Requirement change vs parity.
- **Ideal answer:** Today the match is exact and case-sensitive, same as the Oracle `=` (unless NLS settings differ - unverified). Changing it is a behaviour change from the Jasper report. Confirm it with the business. Options: regex with `i` and `Pattern.quote` (no index use), or a collation on the query (`collation` strength 2, can use a collation-matching index), or a normalised upper-case field. Keep the exact-match mode for parity or add a new parameter. Add tests with mixed-case data.
- **Common mistakes:** Unescaped user text in a regex.
- **Follow-up questions:** Impact on performance? (Regex scans.)

### Q13. A request has `pCreatedDateFrom: "31.12.2026"`. What happens and what do you change?
- **Why the interviewer asks:** Input validation.
- **Ideal answer:** `DateUtils.toLocalDate` does not accept `dd.MM.yyyy`, so `toDate` throws and the request ends in 500 (the catch-all handler; if `IllegalArgumentException` then 400). Fix: validate dates at the start of `execute`, return 400 with a clear message and a list of accepted formats (`yyyy-MM-dd`, `dd-MM-yyyy`, `dd/MM/yyyy`, ...). Add a parameterised test over formats. Remember `dd/MM/yyyy` wins over `MM/dd/yyyy`.
- **Common mistakes:** Adding every format; ambiguity grows.
- **Follow-up questions:** What does the caller send today? [CONFIRM: check a captured request.]

### Q14. Business wants the RejectRisk header and "No Data Found" even when there are no rows. What do you do?
- **Why the interviewer asks:** Parity gap.
- **Ideal answer:** Jasper `whenNoDataType="AllSectionsNoDetail"` prints the title, `DATA_CLASS`, group header and "*** No Data Found ***". Java returns an empty `reportData` list when no rows (test `execute_emptyRows_returnsEmptyReportDataList`), so no header block. Change the service to add the `PlineReportBlocks.reportHeader("RejectRisk", dataClass)` block also for zero rows, keep the rows empty, and let the renderer print the no-data text (confirm who owns that text). Update the test and the mapping note.
- **Common mistakes:** Adding a fake data row to carry the message.
- **Follow-up questions:** What if `dataClass` is not found? (null; the header lookup returns `new PlineReportHeaderDto(null)`.)

### Q15. After a data fix, a code description is missing in RejectRisk. How do you debug?
- **Why the interviewer asks:** Lookup debugging.
- **Ideal answer:** Descriptions come from `CMGE_CODE` with `CAT_CODE` in `PLREJ_RISK_TYPE` (by `REJ_TYPE`), `PLREJ_REASON` (by `REASON_CODE`), `PLREJ_REVOKE_DUR` (by `REVOKED_DURATION`). Check the code value's type (string vs number) and spaces, check the category exists, check duplicates. The lookup returns null silently (Oracle's subquery would also give null for none, error for two). Compare with the raw `CMUW_PL_REFER_RISK` value. Remember Jasper printed "NA" for null text fields; the Java side leaves them empty (rendering question for the report team).
- **Common mistakes:** Assuming a code issue before checking data types.
- **Follow-up questions:** Jasper "R1 / null" case? (Reason column concatenates code and description.)

### Q16. The XSD validation of RejectRisk fails on `pCreatedDateFrom`. What do you do?
- **Why the interviewer asks:** Contract drift handling.
- **Ideal answer:** `pCreatedDateFrom/To` and `pReportTitle` are in the Java whitelist (comment "XSD v0.1.9") but not in the v0.1.6 XSD on this machine; `reportName` is dropped by the whitelist. Steps: ask the schema owner for the current version (not found locally); if v0.1.9+ has them, update the test copy of the XSD; if not, agree to add them (additive leaf) or stop emitting them. Also dates go out as `dd/MM/yyyy` against `xs:dateTime`. Do not edit only one side. Document it in the changelog header and mapping sheet.
- **Common mistakes:** Removing the fields from Java to make a local test pass.
- **Follow-up questions:** Who owns the XSD? [CONFIRM: your team.]

### Q17. A user wants all 500,000 reject-risk rows in one report. What do you do?
- **Why the interviewer asks:** Scale thinking.
- **Ideal answer:** Per row the service creates about eight payload objects and one `reportData` with its own `transDetList`, then one big XML string inside JSON; memory and time grow linearly, and the renderer may also struggle. Steps: estimate size with a count and a sample XML; ask the business for the real need (date ranges); enforce a max range or max rows; split by month with multiple calls; if needed, design a paged or file-based export outside this endpoint. Add `allowDiskUse` and limits. No numbers claimed: none are measured.
- **Common mistakes:** Saying "increase heap".
- **Follow-up questions:** Where is the first bottleneck? (Probably the lookups and the XML size - inferred.)

### Q18. ASP: a user says the service-tax invoice is missing on an endorsement print. How do you investigate?
- **Why the interviewer asks:** Gate analysis.
- **Ideal answer:** Check the four gate conditions in `AspEndtDocumentService`: `showTaxInvoice=Y`, `SHOW_STAX_INV_IND=Y`, `checklistCopy != Y`, `SHOW_EINV_IND=Y`. Then check the sources: `UWGE_VI_ENDT_REPORT_PARAM` flags (type issue: CONTRACT_ID as string returns null flags), the REST call `fnEinvoicePrintEndtInd` (failure defaults to "N" silently - search logs for "PlineUtilsClient GET failed"), and the handler returning a row (`UWPL_VI_ENDT_TAX_INV`). Confirm the request values the caller sent. Reproduce with the same contract and endorsement in dev.
- **Common mistakes:** Starting with the XSD or the renderer.
- **Follow-up questions:** Checklist copy? (Closes the gate by design.)

### Q19. ASP: the cover and schedule are empty for a checklist copy. What do you check?
- **Why the interviewer asks:** Known handler risk.
- **Ideal answer:** A checklist copy has no `ENDT_NO` yet. On HEAD, `AspEndtDocumentMainHandler` matches `{ENDT_NO, CONTRACT_ID}` on `UWMT_VI_ENDT_TAX_INV_ALT_UPV`; the shared base class says that collection is not related and dropped the match for this reason, but the ASP override re-introduced it, so the DTO can be all null (product code, ENDT_CODE) and gates close. Commit `cc6a1b84d` (OPUSM-41763, other branch) rewrites it to per-field lookups. Confirm with one real checklist copy, then bring that change in properly (cherry-pick with tests). Treat it as a risk with evidence until reproduced.
- **Common mistakes:** Declaring it a confirmed production bug without a repro.
- **Follow-up questions:** WIP vs live branch selection? (`checklistCopy` drives `$$chk`.)

### Q20. ASP: a debit note prints although premium did not change. What do you check?
- **Why the interviewer asks:** Edge-case reasoning.
- **Ideal answer:** The gate is `showDebitNote=Y && PREM != 0`, where PREM is derived from `DIFF_PREM_DUE`. In `AspEndtDocument2MainHandler`, `$ne` of a missing `PB_ROW.DIFF_PREM_DUE` against 0 is true in Mongo, so PREM becomes 1 when no base row exists (inferred, no test). Jasper would have no driver row at all. Check the handler with a null `PB_ROW`, fix with `$ifNull(..., 0)` or an explicit existence check, add a test, and compare with Oracle for the same endorsement.
- **Common mistakes:** Changing the service gate instead of the handler.
- **Follow-up questions:** Expected output when both diffs are zero? (No EndtDocument2 output in Jasper.)

### Q21. ASP: the schedule-of-benefits page is missing in the PDF for an add-member endorsement. What now?
- **Why the interviewer asks:** Stub awareness.
- **Ideal answer:** Jasper prints `EndtScheduleOfBenefits_ASP` (+ASPLLB, ASPSTI) when `ENDT_CODE` is 72 or 168. Java calls three stub handlers and emits no `reportData` for them. Check whether the renderer derives the page from the master row's `actionCode` (unknown; open question). Check the master row has `actionCode` = ENDT_CODE (main handler anchor risk). If the renderer needs a marker, add a `reportData` with `subReportName` set (for example `EndtScheduleOfBenefits_ASP`) after agreeing with the report team. Add a test for 72 and 168.
- **Common mistakes:** Looking for benefit data in Mongo: the page is static text.
- **Follow-up questions:** How did you learn the pages are static? (Jasper `SELECT 1 FROM DUAL`.)

### Q22. ASP: the deletion landscape page prints an empty table for an endorsement with code 72. Why?
- **Why the interviewer asks:** Quirk in Jasper itself.
- **Ideal answer:** The dispatch counts deleted risks of ANY endorsement code (`UWPL_VI_ENDT_ADD_DEL_RISK`, `ACTION_CODE='C'`, `TERMINATE_DATE` not null), but the sub-report shows only `ENDT_CODE IN ('98','168')`. So a count > 0 for another code gives a landscape background with an empty table (inferred from reading both SQLs). Java copies the dispatch logic. Fix only after business sign-off: add the same `ENDT_CODE` filter to the dispatch (a deviation from Jasper), or leave parity. Add a test for code 72 with a terminated risk.
- **Common mistakes:** "Fixing" silently.
- **Follow-up questions:** Which tests exist? (`landscapeDeletionGate_closed_skipsDeleteMemberHandler`.)

### Q23. Someone wants to re-enable `AspEndtDocument2Service` "to test it separately". What do you say?
- **Why the interviewer asks:** Understanding of the merge risk.
- **Ideal answer:** Do not re-enable. The merged `AspEndtDocumentService` already runs those bands; a second bean means duplicate bean keys and double rendering (class javadoc warns). The legacy classes' tests (12 and 8) do not exercise the live bean. If they need isolation, test the live bean with mocks (28 tests exist), or write a test-only subclass or call the private logic through the service. If they want old report names, handle it in a routing layer, not by a second implementation.
- **Common mistakes:** Re-adding `@Service` temporarily and forgetting to remove it.
- **Follow-up questions:** How to prevent regressions? (A test asserting the strategy map keys.)

### Q24. A caller still sends `.../ASPEndtSchedule/EndtDocument2`. What happens and what do you do?
- **Why the interviewer asks:** Backward-compatibility.
- **Ideal answer:** No bean has that name now, so it goes to the legacy switch and likely returns HTTP 200 SUCCESS with an empty default transaction (inferred; the full switch was not re-read). So the caller gets a blank report with no error. Steps: confirm in dev; tell the caller to use the merged `.../EndtDocument` name (it includes EndtDocument2, AddMemberLandscape, DeletionMemberLandscape output); optionally map old names to the merged service in the controller with a log warning; return 400/404 for unknown names. Update the docs and README (no ASP section exists).
- **Common mistakes:** Re-creating beans with the old names.
- **Follow-up questions:** Same for Template12 old names? (`.../PolicyDocument2` and `.../Landscape` are also merged away.)

### Q25. The SST rollout date API is down. What do you expect, and what do you do?
- **Why the interviewer asks:** Dependency failure handling.
- **Ideal answer:** `PlineSSTRolloutDateClient` returns null on `RestClientException`; `AspServiceTaxInvoiceHandler` and `PlineT12PremiumDebitNoteHandler` continue with a null date. The response is still 200, with blank or default values (for example label logic). Steps: check logs for the warning, check the platform API status, see how the rollout date affects the output for this contract (label wording, tax lines), and reprint after recovery. Longer term: metric/alert on the warning, a degraded flag in the response, a cached value with TTL (the date changes rarely - inferred), and retry on GET.
- **Common mistakes:** Treating the printed document as correct.
- **Follow-up questions:** Which timeout applies? (Unqualified client: 5 s connect / 20 s read.)

### Q26. ASP: the QR code is missing on the service-tax invoice. What do you check?
- **Why the interviewer asks:** Multi-condition gate.
- **Ideal answer:** QRSection needs `EINV_CONSENT_IND=Y`, `EINV_STATUS=VALID` and `QR_IND=Y`, after the service-tax gate itself. Check consent in `UWGE_POLICY_BASES`/`WIP_UWGE_POLICY_BASES` (NVL 'N'), the status in the tax invoice row, the QR indicator REST call (`PlineEinvoiceQrIndicatorClient`, `safeCall` returns null, defaults to "N"), and the QR data in `UWGE_GST_TAX_INV`/`WIP_UWGE_GST_TAX_INV`. For Template12, also remember QR is looked up with the request `pTaxInvNo` (default ""), so QR can be empty unless the caller sends it.
- **Common mistakes:** Assuming QR depends only on the QR indicator.
- **Follow-up questions:** Which test covers the gate? (QR gate test in `AspEndtDocumentServiceTest`, line ~903.)

### Q27. Master policy details are wrong on an ASP service-tax invoice. What do you check?
- **Why the interviewer asks:** Multi-step data chain debugging.
- **Ideal answer:** Walk the chain in Compass: `MST_POL_NO` from `UWGE_POLICY_BASES` and `WIP_UWGE_POLICY_BASES` for the contract (VERSION_NO=1) -> `OCP_POLICY_BASES` where `POLICY_REF` equals it (VERSION_NO=1) -> master `CONTRACT_ID` -> `PREM_BY_MST_IND` (default 'N'). If any step fails, `AspDebitNoteHandler.fetchMstContractInfo` swallows the exception and sets "N", so the normal buyer block prints instead of master details. Then check `UWPL_VI_TAX_INVOICE` for `MASTR_TIN_NO`, BRN and SST; TIN is masked to the last 4 characters. Add logging for each step if needed.
- **Common mistakes:** Not noticing the swallowed exception.
- **Follow-up questions:** Why is it 4-5 queries? (Mirrors nested Oracle sub-selects.)

### Q28. A service-tax invoice should show multiple tax rates but shows one. What do you check?
- **Why the interviewer asks:** `STAX_COUNT` logic.
- **Ideal answer:** EndtMT and EndtMTSub rows are added only when `STAX_COUNT > 1` (and EndtMT also `ENDT_CODE != "999"`). `computeStaxCount` sums non-zero `DIFF_STAX_AMT` rows from WIP and committed tax tables (`WIP_UWGE_POLICY_BASES_STAX`, `UWGE_POLICY_BASES_STAX`) and swallows errors to 0. Check both tables for the endorsement, the `ENDT_NO`/`REF_NO` filter, errors in the log (count 0 on failure), and that the test `showServiceTax_staxCountNotAboveOne_skipsEndtMTAndSub` matches your case. The Jasper banner gate (`STAX_COUNT>1 && ENDT_CODE != 999`) differs slightly from Java (banner whenever the invoice exists; static anyway).
- **Common mistakes:** Forgetting the 999 exception.
- **Follow-up questions:** Which tables for the live side? (`UWGE_POLICY_BASES_STAX`.)

### Q29. A new ASP-related product code needs the fee-difference formula. What do you change?
- **Why the interviewer asks:** Change impact.
- **Ideal answer:** Today products 089332, 089432, 089329, 089429 add ASST/CRF/OCR fee diffs to gross premium (`AspDebitNoteHandler` lines ~273-283, `AspServiceTaxInvoiceHandler.grossPremiumExpr`). Jasper also does it in the EndtSchedule query, but `AspEndtScheduleHandler` does not. Steps: confirm with the business which documents need the formula; change the list in all places (preferably one shared constant), update the Jasper-equivalent for EndtSchedule if required, add tests for the new code and for others (no change), and check layout rules that exclude those codes (fee lines). Avoid hard-coded literals scattered in handlers.
- **Common mistakes:** Updating one handler only.
- **Follow-up questions:** Where else are product codes hard-coded? (`070412`, `72`, `168`, `98`, `999`.)

### Q30. The schedule shows wrong instalment text for a policy. How do you investigate `INST_IND`?
- **Why the interviewer asks:** Known gap.
- **Ideal answer:** Jasper: `INST_IND='Y'` when `ISSUE_DATE >= SAPM_SYS_CONSTANTS(CUT_OFF_RPT, <PRODUCT>_ENDT)` and `PMT_FREQUENCY <> 'AN'`. Java reads `PMT_FREQUENCY` but never uses it, and takes `INST_IND` from a base document with default "N". So the cut-off is not evaluated. Steps: confirm the case in Jasper vs Java for the contract; implement the rule in `AspEndtScheduleHandler` (lookup the constant, compare dates, check frequency), add tests for before/after cut-off and 'AN', and confirm with the business that this rule is still correct.
- **Common mistakes:** Setting `INST_IND` in the service.
- **Follow-up questions:** Same for `SHOW_GST_LABEL`? (Needs the SST rollout date; handlers do not call the function.)

### Q31. Credit-note amounts differ by 0.05 from Oracle on some endorsements. Where do you look?
- **Why the interviewer asks:** Rounding parity.
- **Ideal answer:** `ROUNDED_NETT_PREM = ABS(round(NETT_PREM*20)/20)`. Oracle `ROUND` is half away from zero; Mongo `$round` is half to even (documented behaviour; inferred risk at exact .5 boundaries after scaling). Check amounts where `NETT_PREM*20` ends in .5. Fix: emulate half-up in the pipeline (`$cond` on sign with `$floor`/`$ceil`) or round in Java with `BigDecimal` `RoundingMode.HALF_UP`. Add a test with exact boundary values. Also check the `GROSS_PREM` formula and the `D`/`C` sign rule (`PREM_DUE >= 0`).
- **Common mistakes:** Using double arithmetic in Java.
- **Follow-up questions:** How to prove the cause? (Compare 10 boundary cases with Oracle.)

### Q32. `AspServiceTaxInvoiceHandlerTest` passes but production behaves differently. What do you suspect?
- **Why the interviewer asks:** The duplicate-class defect.
- **Ideal answer:** A copy of `AspServiceTaxInvoiceHandler` sits in `src/test` with the same package and name (483 vs 476 lines, small differences like `catch (Exception)` vs `catch (RuntimeException)`). Test classes come first on the test classpath, so the test may run the copy. Steps: print the class's code source in a test, delete the copy, re-run the 27 tests, and fix any failures against the real class. Also review whether JaCoCo coverage for the real class changes.
- **Common mistakes:** Updating both copies to keep them in sync.
- **Follow-up questions:** How did it get there? (Commits `3f2157b28`, `24e50ab9b`: a Sonar-style edit applied to the wrong tree - inferred.)

### Q33. Your lead asks you to put the meaning of "ASP" in the documentation. What do you write?
- **Why the interviewer asks:** Honesty and documentation practice.
- **Ideal answer:** Write what is proven: no expansion in code or jrxml; internal names `ASPlusSTI`, `ASPlusLLB`; benefit codes `ASP-A1..D5`; personal-accident style benefits; Java binds it to product code `070412`. Ask the product owner for the official name and add it with the source. Until then, do not guess, and mark it "to be confirmed". [CONFIRM: if you know the real name from the business.]
- **Common mistakes:** Writing a plausible expansion.
- **Follow-up questions:** Who could confirm? [CONFIRM: your BA/product owner.]

### Q34. The sponsor landscape rows come in a different order than the Jasper PDF. What do you do?
- **Why the interviewer asks:** Ordering parity.
- **Ideal answer:** The Jasper sponsor and delete-member queries have no `ORDER BY`, and the Java ones add none, so the order is natural storage order in both (Oracle order is also unspecified). Insured, premium and nomination landscape use `cast(COV_SEQ_REF as decimal(15,2))`; Java uses `$toDouble` then `$sort`. If the business wants stable order, agree on a key (for example name or sequence) and add `$sort` plus a test, noting it differs from Jasper. If it is a numeric-order problem, check the raw sorts in AMC/Diagnosis handlers.
- **Common mistakes:** Promising the same order as Oracle without a sort key.
- **Follow-up questions:** Null `COV_SEQ_REF`? (Mongo first, Oracle last.)

### Q35. Template12: a Landscape page is missing for a contract. What is your checklist?
- **Why the interviewer asks:** Structured debugging.
- **Ideal answer:** (1) Did `fetchDispatch` return null (`CI_LANDSCAPE_PRT` row for the scheme in `CMGE_CODEREL_DET`)? (2) Product and scheme as the dispatch derives them (can differ from PolicyDocument's: three fetches). (3) The flag for that band: `benefitPrint`, `nomineePrint`, `diagnosisPrint` (first version row only), `aspCutoffInd`. (4) The Java `if` in `executeMergedLandscape` (lines ~842-1479) vs the Jasper condition. (5) Run the band handler's pipeline in Compass (collections `UWPL_VI_POLICY_SCHEDULE_NOM/DIAG/BEN/PREM/TPL4/TPL20`). (6) Check the renderer handles multiple `reportData`.
- **Common mistakes:** Debugging the renderer first.
- **Follow-up questions:** Log lines? (`Template12LandscapeDispatchHandler contractId=`.)

### Q36. Template12 returns HTTP 500 for a request. What are the two most likely causes?
- **Why the interviewer asks:** Known failure modes.
- **Ideal answer:** (1) Empty or non-numeric `pContractId`: `Long.parseLong` in `Template12LandscapeDispatchHandler` throws `NumberFormatException`, not caught. (2) WIP vs posted conflict in the PolicyDocument primary: `IllegalStateException` ("Ambiguous ... ORA-01427"), unguarded. Check the request body first, then the data for that contract in `WIP_UWGE_POLICY_CONTRACTS`/`SB_UWGE_POLICY_CONTRACTS`/`WIP_UWPL_POLICY_BASES`/`UWPL_POLICY_BASES`. The catch-all handler does not log the exception, so find the stack in the service logs. Longer-term: validate input early (400), and log in the advice.
- **Common mistakes:** Catching and hiding the data conflict.
- **Follow-up questions:** Is the conflict a code bug? (No, it mirrors Oracle; it is a data issue.)

### Q37. The PolicyDocument fetch throws "Ambiguous ... ORA-01427" for a contract. What do you do?
- **Why the interviewer asks:** Data conflict handling.
- **Ideal answer:** It means WIP and posted data disagree on product or scheme (or one side has several distinct values). Oracle would fail the same way, so first confirm with Oracle if possible. Look at `WIP_UWGE_POLICY_CONTRACTS` and `SB_UWGE_POLICY_CONTRACTS` (`PRODUCT_CONFIG_CODE`), `WIP_UWPL_POLICY_BASES` and `UWPL_POLICY_BASES` (`TOP_INDICATOR='Y'`). Ask the data owner to correct the data. Do not change `requireNoConflict` to "prefer WIP": it would change behaviour silently and hide bad data (class javadoc "CRITICAL FACT #1"). If the business wants tolerance, get sign-off, and treat PD2's `$ifNull` precedence as the model.
- **Common mistakes:** Removing the exception to make the report print.
- **Follow-up questions:** Why do PD2 and Landscape not throw? (Different primary fetch semantics.)

### Q38. A new scheme is launched under product 070504. How do you add it?
- **Why the interviewer asks:** Extension procedure.
- **Ideal answer:** (1) Check whether Jasper has a new CIPrint variant and the exact print conditions. (2) In `S12.execute` add a boolean and an `if (is070504 && isNewScheme)` block; update the Elite fallback (the "none of the others" list at ~374-375). (3) Create the primary handler, nested handlers, DTO and payload classes; add `addTemplate12CIPrintX` in `B12`. (4) Check the Landscape bands (L5-L10, L15) and PD2 complaint/SCB rules for the scheme. (5) XSD leaves if new fields (version bump, mapping sheet). (6) Service branch test plus handler tests. The danger is silent fall-through to Elite.
- **Common mistakes:** Forgetting the Elite exclusion list or the Landscape nomination conditions.
- **Follow-up questions:** Table-driven alternative? (A band registry with conditions.)

### Q39. A BMW policy has a very large certificate range. What do you do?
- **Why the interviewer asks:** Scalability of per-row loops.
- **Ideal answer:** `CIPrint_BMW` returns one row per certificate between `pCertNoFrom` and `pCertNoTo`, and for each row S12 calls six nested handlers (Coverage_BMW, Nomination_BMW, Note_BMW, CoverageBenefit_BMW, PerilsClauses_BMW, ComplaintRedress_NoMargin) - many Mongo queries per row. Steps: measure with a sample (not measured now); consider bulk handlers keyed by a list of certs; batch requests by range; cap the range; run independent calls in parallel. Also note `pCertNoFrom/To` are not serialised to the XML.
- **Common mistakes:** Guessing performance numbers.
- **Follow-up questions:** What makes it safe to batch? (Read-only queries.)

### Q40. The PDF has missing or extra blank pages for ASP-cutoff policies. What do you do?
- **Why the interviewer asks:** The `PAGE_NUMBER` gap.
- **Ideal answer:** Jasper adds `Blank_NoMargin` only when `(PAGE_NUMBER-1)%2 != 0 && ASP_CUTOFF_IND=="Y" && 070412`; Java adds the row whenever `070412 && cutoff=Y` (comment: "may over-include"). Page parity exists only at render time. Steps: confirm with the report team whether the renderer re-evaluates parity for `Blank_NoMargin`; if not, the template should hide it on even pages. Agree on a contract (marker row with a condition, or the renderer rule). Similar gaps: PD2 blank before the debit note (not implemented), POSMESRA blanks, PrintSeriesFooter call 2.
- **Common mistakes:** Trying to compute parity from row counts.
- **Follow-up questions:** Why does parity exist? (Duplex printing: a document starts on a front page.)

### Q41. Non-travel policies show an Overseas_ATC marker in the PD2 section. What is the cause?
- **Why the interviewer asks:** Known deviation (D3).
- **Ideal answer:** Jasper's band is gated by `PRODUCT_CODE=="070406"` and picks Domestic or Overseas by `JOURNEY_AREA`. Java does `if DOM -> addT12DomesticAtc else addT12OverseasAtc` with no product test (comment "always fires one or the other"), in S12 and in the old PD2 class; tests `execute_pd2JourneyAreaDom_usesDomesticAtcBranch` and `..._defaultsToOverseas` encode it. Fix: wrap in `is070406`, update both tests, and check that the renderer ignores or uses the marker. Confirm with the product owner first; it is a behaviour change.
- **Common mistakes:** Editing the test to match the bug.
- **Follow-up questions:** Impact on the XML? (Extra dispatch-only merged row; minor.)

### Q42. Some dates in the Template12 XML show the wrong format or are missing. How do you debug?
- **Why the interviewer asks:** Date pipeline understanding.
- **Ideal answer:** The chain: DTO date String (maybe with `T00:00:00`) -> `toLocalDateTime` (strips time, parses; unparseable becomes null silently) -> `LocalDateTime` payload -> reflective pass to `dd/MM/yyyy` on String fields named like `*date*`/`*dob*` (`S12:2105-2230`) -> `@JsonFormat dd/MM/yyyy`. Missing date = a parse failure or null from the DTO. Check the raw value in the handler result, add a temporary log, and unit-test the converter with that string. Remember XSD wants dateTime and the REST calls get `yyyy-MM-dd`.
- **Common mistakes:** Changing a format in one place and breaking the reflective pass.
- **Follow-up questions:** Is the reformat pass tested? (No test found, inferred from names.)

### Q43. The promo amount or the ATC Landscape page differs from what Jasper printed. What do you check?
- **Why the interviewer asks:** Known SQL semantic difference (D7).
- **Ideal answer:** Jasper sums `COV_PROMO_AMT` over a join of `wip_UWPL_RISK_PERSON` and `wip_uwge_cover` on contract id only, so covers repeat per risk person (inferred: multiplied by number of insured). Java checks that a risk person exists and sums covers once; PD2 applies version filters, the Landscape dispatch does not. Steps: pick a contract with 3 insureds; run the Oracle query and the Java pipelines; compare; decide with the business whether to replicate the multiplication or treat it as a Jasper defect; align PD2 and Landscape (same version filters); add a test with several persons.
- **Common mistakes:** Assuming the values are equal because column names match.
- **Follow-up questions:** Which band depends on promo? (L16: `070406 && PROMO_AMT > 0`.)

### Q44. QR section is empty in the Template12 service-tax invoice. What do you suspect?
- **Why the interviewer asks:** Request-vs-row value bug (D14).
- **Ideal answer:** `addGenericQRSection(contractId, versionNo, pTaxInvNo, rd)` uses the request `pTaxInvNo` (default ""), but Jasper passes the row's `STAX_INV_NO` (or `TAX_INVOICE` for PA4A). `PlineGenericQRSectionHandler` filters on `STAX_INV_NO`, so without the caller sending the number the result is empty (inferred). Fix: use the invoice number from the service-tax row (`staxInvNo`/`taxInvoice`), as TaxInvoince and Transitional already do; add a test that the QR handler receives the row's number. Check the consent/VALID/QR_IND conditions too.
- **Common mistakes:** Asking callers to send the number as the permanent fix.
- **Follow-up questions:** `pGstTaxInvoiceNo`? (Ignored by handlers that match on contract id only.)

### Q45. Product says the PDS_Income marker must not print for INCOMESHIELD (Jasper does not). What do you do?
- **Why the interviewer asks:** Deviation resolution.
- **Ideal answer:** In Jasper the band is `false && ...`, so it never prints. Java wires `addPDSIncome`/`addPDSElite` "per explicit instruction" and tests assert it (`execute_070504IncomeShield_firesPdsIncomeOnly`). Ask who gave the instruction and what the renderer does with the marker. If the decision is to match Jasper, remove the two calls, update the tests (PD2 and merged service tests), and note the change in the traceability table. If product wants it later, keep a feature flag.
- **Common mistakes:** Deleting code without checking the original instruction.
- **Follow-up questions:** How do you record the decision? (Mapping sheet and a short note in the class javadoc.)

### Q46. Someone adds `@Service` back to `Template12LandscapeService` and every landscape band appears twice. What do you do?
- **Why the interviewer asks:** Merge regression handling.
- **Ideal answer:** Revert the annotation. The merged bean already calls `executeMergedLandscape`; the old class kept `//@Service` with a warning. Check the strategy map (the old report name would register again), confirm via a response with duplicate `reportData`, and add a test that fails if more than one bean claims the Template12 names or if the old classes have `@Service`. Explain the history: commit `c0c6d449a`. Add a code-review note.
- **Common mistakes:** Removing the merged call instead.
- **Follow-up questions:** How to make accidental re-enable harder? (Move old classes to a `legacy` package without stereotype annotations or delete them with their tests.)

### Q47. Product asks you to add a field to the output but the XSD has no leaf. What do you do?
- **Why the interviewer asks:** Governance.
- **Ideal answer:** Do not guess a place. Check for an existing leaf by name in `transDet` (634 leaves) and `endorsementDetails` (50). If none: raise a change with the schema owner, agree on name, type, position (alphabetical slot) and version; add to the XSD, `PlineTransDet` (same order, `@JacksonXmlProperty` if case differs), the DTO/`$project`, the mapper, the mapping sheet, and tests including serialise-and-validate. Meanwhile leave the field unmapped with a comment (the existing practice). Do not repurpose an unrelated leaf.
- **Common mistakes:** Emitting the element first and updating the XSD later (what happened with `mainReportName`).
- **Follow-up questions:** Alternative without schema change? (`paramGroup`: name/value pairs; trade-offs.)

### Q48. FWPA policies fail because a rebate amount is null in the Mongo view. What do you do?
- **Why the interviewer asks:** Knowledge of a real workaround.
- **Ideal answer:** The view `UWPL_VI_POLICY_SCHEDULE_TPL4` throws when a rebate amount is null, which is why `Template12PolicyDocumentBaseService` has `fetchCIPrintFwpaResilient` (duplicating `PlineCIPrintFwpaPrimaryHandler`) and `fetchParticularOfInsuredPotraitFwpaResilient`. A null-safe path with `$ifNull` is used. For a new failure, reproduce with the contract in Compass, find the null field, wrap it in `$ifNull`/`$convert onError`, and add a test with null input. Longer term fix the data/view and remove the duplicate.
- **Common mistakes:** Catching the exception and returning an empty row.
- **Follow-up questions:** Why not fix the shared handler? (Template12 is isolated by design; other templates use it.)

### Q49. The coverage list prints "10" before "2". How do you find and fix it?
- **Why the interviewer asks:** Sort type issue.
- **Ideal answer:** Jasper sorts `SORT_SEQ` / `COV_SEQ_REF` numerically (`cast ... decimal`). The Coverage handler sorts by plan code then `$toDouble(SORT_SEQ)`; three handlers sort `COV_SEQ_REF` raw (`InsuredLandscapeAmc`, `DiagnosisLandscape`, `DiagnosisLandscapeAmc`). Check the field's BSON type in Compass. If string, add `$addFields` with `$toDouble` (or `$convert` with `onError`) and sort on it; add a test with "2" and "10". Check that the handler's `$project` keeps the sort field until after the sort.
- **Common mistakes:** Sorting after the field was dropped.
- **Follow-up questions:** Null values? (Mongo first, Oracle last.)

### Q50. A `reportParameters` value you set in the service does not appear in the XML. What is wrong?
- **Why the interviewer asks:** The whitelist trap.
- **Ideal answer:** `PlineReportParameters` has a 73-name `@JsonIncludeProperties` whitelist. If the setter writes a base-class property not in the list (for example `setPcontractId` -> `pContractId`, `pCertNoFrom`, `pTaxInvNo`, `reportName`, or `ShowOutPrintCovLetter` vs whitelisted `pShowOutPrintCovLetter`) it is dropped silently. Also ASP's `setPshowGstLabel` base setter is ignored, unlike `setPShowGstLabel`. Steps: serialise and look; check the exact JSON property name; if the XSD has the leaf, add the name to the whitelist; if not, it is a schema discussion. Add a serialisation test for the property.
- **Common mistakes:** Debugging the service logic.
- **Follow-up questions:** Which six whitelisted names are not in v0.1.6? (`pCreatedDateFrom/To`, `pReportTitle`, `pshowEndtSchedule`, `pShowOutPrintCovLetter`, `pShowPDPANotice`.)
