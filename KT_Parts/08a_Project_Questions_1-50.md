# 08a - Project Questions 1-50 (opus-documentprintdata-service)
> **AUTHORSHIP UPDATE (confirmed by the developer): git author `GS <extern.gs_nisarga@allianz.com>` is the developer. Wherever this file says commits cannot be attributed to the developer, read it as: the 11 GS commits listed in `KT_Parts/14_My_Commits_And_Ownership.md` ARE theirs; commits by other authors still must not be claimed. No GS commit touches `src/test`, so test authorship is still `[CONFIRM]`.**

How to use this file
- These are PROJECT questions: they ask about this system, your three modules (Template12, ASP, RejectRisk) and your decisions.
- Every fact comes from the KT guide and KT_Parts 01-06, or from the repo. "(inferred)" means my reasoning, not proven. "[CONFIRM: ...]" means only you know the answer. Do not invent it.
- Git does not prove who wrote which commit. Never say "I wrote commit X" unless you know it.
- Short names: T12 = Template12, PD2 = PolicyDocument2, S12 = Template12PolicyDocumentService, B12 = Template12PolicyDocumentBaseService.

---

### Q1. Explain the project in one minute.
- **Why the interviewer asks:** To see if you understand the system, not only your own class.
- **Ideal answer:** `opus-documentprintdata-service` is a Spring Boot microservice in Allianz OPUS. It does not make PDFs. A caller sends `POST /gi/documentprintdata/api/opus/GenerateReport` with a `reportName` and filters. The service reads MongoDB (a mirror of the old Oracle data), calls a few OPUS REST APIs, and returns XML inside a JSON wrapper (`status`, `data`, `timestamp`). A print engine makes the PDF. The old reports were Jasper `.jrxml` files with Oracle SQL inside; we rewrote the data side as Mongo aggregation pipelines. I worked on three Personal Lines reports: Template12, ASP and RejectRisk.
- **Common mistakes:** Saying "it creates PDFs". Forgetting the data comes from Mongo, not Oracle. Not saying who renders the PDF.
- **Follow-up questions:** Who calls this endpoint? (Not visible in the repo.) What is the response field name for the XML? (`data` in the class `SuccessResponse`; the README says `xmlData`.)

### Q2. What exactly was your part in the project?
- **Why the interviewer asks:** To separate your real work from team work.
- **Ideal answer:** Jira scope was analysis, code, unit tests and XSD changes for Template12 (PolicyDocument, PolicyDocument2, Landscape), ASP (EndtDocument, EndtDocument2, AddMemberLandscape, DeleteMemberLandscape) and RejectRisk. For each report I read the Jasper jrxml tree, wrote down print conditions, mapped SQL to Mongo, mapped fields to Pline.xsd leaves, and wrote tests. [CONFIRM: which parts you personally coded vs reviewed, and which commits are yours - git authors in the repo are other names, so do not claim commits without checking.]
- **Common mistakes:** Claiming the whole service. Claiming commit hashes you cannot prove. Mixing up "analysed" and "coded".
- **Follow-up questions:** Which of the three was hardest and why? Which part would you not touch again?

### Q3. Why does this service exist? Why not keep Jasper with Oracle?
- **Why the interviewer asks:** To check you know the business reason (retrofit).
- **Ideal answer:** OPUS is being retrofitted. Oracle data is now mirrored into MongoDB, with the same upper-case names as the Oracle tables and views (for example `UWGE_POLICY_VERSIONS`, `UWPL_VI_POLICY_SCHEDULE_TPL4`). Jasper reports ran SQL directly on Oracle, so that SQL had to be rewritten. `plines/README.md` has an "Oracle source to MongoDB strategy" table. Oracle functions such as `FN_DECIDE_MT_CERT` are now REST calls through `PlineUtilsClient`.
- **Common mistakes:** Saying "to replace Jasper". The Jasper template still renders the PDF; only the data source changed (inferred from the design).
- **Follow-up questions:** What did not translate easily from SQL? (ROWNUM, UNION, DISTINCT, scalar sub-selects, NULL ordering.)

### Q4. Walk me through one request from HTTP to XML.
- **Why the interviewer asks:** To test the main flow.
- **Ideal answer:** `CustomRequestFilter` wraps the body (turns array `pBranch` into a comma string). `CommonController` normalises `reportName` (strips leading `/` and `reports/`), looks it up in `Map<String, ReportStrategy>`, else falls to a legacy `switch`. The service `execute()` calls handlers (Mongo aggregation) and REST clients, maps DTOs into `Pline*` payload classes, returns `Transactions`. The controller adds `templateDetails` and `mailingDetails` from `metadata`, writes XML with `XmlMapper`, wraps it in `SuccessResponse`. Always HTTP 200 on the normal path.
- **Common mistakes:** Forgetting the filter, or saying the controller has one method per report.
- **Follow-up questions:** What if `metadata` is missing? (NPE in `populateMailingDetails`, so 500 - inferred from code.)

### Q5. How does the controller know which report class to run?
- **Why the interviewer asks:** The Strategy pattern is the base of every module.
- **Ideal answer:** Each report is `@Service("AGIC/UW/Plines/RejectRisk/RejectRisk")` style and implements `ReportStrategy` (`Transactions execute(PayLoadRequest)`). Spring injects all of them as a map keyed by bean name. The controller does `reportStrategies.get(reportName)`. To add a report you only add a service with the right bean name.
- **Common mistakes:** Thinking the name is a URL path. It is a bean name with slashes.
- **Follow-up questions:** What happens when the name has no bean? (Falls to the legacy switch; unknown name still gives HTTP 200 SUCCESS with an empty default transaction.)

### Q6. Explain the layers: handler, DTO, payload, service.
- **Why the interviewer asks:** To check clean-layer thinking.
- **Ideal answer:** A handler (`@Component`) runs one Mongo aggregation, equal to one Jasper query or sub-report, and returns a DTO. The service decides which handlers to call (the Jasper print conditions) and maps DTOs into `Pline*` payload classes that copy `Pline.xsd`. Example: `RejectRiskHandler` -> `PlineRejectRiskDto` -> `RejectRiskService` sets `PlineTransDet` fields.
- **Common mistakes:** Putting mapping in the handler. Putting business gates in the DTO.
- **Follow-up questions:** Why one handler per sub-report? (Each Jasper sub-report is one query; easy to test and trace.)

### Q7. What is Template12 in business terms?
- **Why the interviewer asks:** To see domain understanding.
- **Ideal answer:** It is the Personal Accident policy schedule family. Three Jasper mains: PolicyDocument (CI print variants by product and scheme), PolicyDocument2 (debit note, GST/SST/transitional invoices, PDPA, ATC page) and Landscape (16 wide "particulars" bands). Each main runs a one-row `dual` query that gives product, scheme and flags, and each band has a `printWhenExpression`.
- **Common mistakes:** Treating it as one flat report. Not knowing the product codes (070504, 070412, 070406, 070414, 100908).
- **Follow-up questions:** How many sub-reports? (About 87 `.jrxml.data` files plus 3 mains in the Template12 folder.)

### Q8. Why is there only one live bean for Template12?
- **Why the interviewer asks:** This is a key design fact and a trap question.
- **Ideal answer:** Only `@Service("AGIC/UW/Plines/PolicySchedule/Template12/PolicyDocument")` on `Template12PolicyDocumentService` is live. Commit `c0c6d449a` ("Added Template12 dynamic merging", 2026-08-03) copied PD2 and Landscape bodies into private methods `executeMergedPolicyDocument2` and `executeMergedLandscape`, and commented out `@Service` on `Template12PolicyDocument2Service` and `Template12LandscapeService`. Reason: one response with all pages and no double rendering.
- **Common mistakes:** Saying the old classes were deleted. They are kept for reference and still have tests.
- **Follow-up questions:** What if someone re-adds `@Service`? (Every band would render twice.) What happens to a request for the old name? (No bean; legacy fall-through, likely an empty SUCCESS - inferred.)

### Q9. Why does Template12 have three different primary fetches?
- **Why the interviewer asks:** Checks you understand deliberate inconsistency.
- **Ideal answer:** Class javadoc "CRITICAL FACT #1". PolicyDocument uses `fetchTemplate12PolicyDocumentMain`, which throws `IllegalStateException` if WIP and posted data disagree (copies Oracle ORA-01427). PD2 uses `PlineT12PolicyDocumentHandler`, which prefers posted data and falls back to WIP with `$ifNull`. Landscape uses `Template12LandscapeDispatchHandler` with its own chain. They were not unified because that would change PD2 behaviour without product sign-off. Cost: three places derive product and scheme and can disagree.
- **Common mistakes:** Calling it a bug or calling it perfect.
- **Follow-up questions:** How would you unify safely? (One shared product-context object after sign-off.)

### Q10. How does Java choose the right CIPrint variant?
- **Why the interviewer asks:** Tests knowledge of the product/scheme tree.
- **Ideal answer:** Same rules as `PolicyDocument_jrxml.data`. Booleans in S12 (`is070504`, `is070412`, `is070406`, `is070414`, BMW, POSMESRA, FWPA). Default products -> `CIPrint`; 070406 -> `CIPrint_ATC`; 070414 -> `CIPrint_PA4A` (one set of bands per row); 070504 by scheme: ASTRO, BMW/BMW1, POSCOVER, AZP1/AZP2, POSMESRA, FWPA, else Elite. ASTRO and POSCOVER share one handler.
- **Common mistakes:** Forgetting the aliases BMW1 and AZP2. Forgetting that Elite is the "none of the others" fallback.
- **Follow-up questions:** What changes when a new 070504 scheme is launched? (New boolean, new `if`, update the Elite exclusion list, handlers, DTO, XSD leaves, tests.)

### Q11. What is the Landscape suppression rule?
- **Why the interviewer asks:** A concrete business rule with a data source.
- **Ideal answer:** `Template12LandscapeDispatchHandler.fetchDispatch` looks in `CMGE_CODEREL_DET` for `CAT_CODE=GPA_SCHEME`, `DETAIL_CODE=CI_LANDSCAPE_PRT`, `CD_VALUE=Y`, `HEADER_CODE=scheme`. If it matches, the dispatch returns `null` and S12 prints no landscape band at all. The Jasper SQL had the same rule as `where not exists`.
- **Common mistakes:** Putting the check inside each band. Treating `null` as an error.
- **Follow-up questions:** Which test covers it? (`execute_landscapeDispatchNull_excludedByCiLandscapePrt_noExtraReportData`.)

### Q12. Why does Template12 have its own base class?
- **Why the interviewer asks:** Trade-off thinking.
- **Ideal answer:** `Template12PolicyDocumentBaseService` (3,732 lines) does not extend `PlinePolicyDocumentBaseService` (4,083 lines, used by about 24 templates). Its javadoc says it is a sibling toolkit, so Template12 changes cannot break other templates, and Template12 uses the new XSD-typed envelope. Cost is duplication (for example `fetchCIPrintFwpaResilient`, a copy of `PlineCIPrintFwpaPrimaryHandler`).
- **Common mistakes:** Saying inheritance is always wrong. The reason is risk isolation.
- **Follow-up questions:** How would you remove the duplication later? (Extract shared helpers with tests first.)

### Q13. What is `mergeSingleTransDet` and what is the risk?
- **Why the interviewer asks:** Tests understanding of a tricky helper.
- **Ideal answer:** `addSingleTransDet` appends a new `transDet` (real multi-row bands). `mergeSingleTransDet` copies non-null fields of a one-row result onto every existing `transDet` using reflection (`mergeNonNullFields`), so many one-row bands do not create many empty rows. Risk: last non-null writer wins, so one band can overwrite a field from an earlier band. Commits `7399d2fc1` and `c0c6d449a` relate to this.
- **Common mistakes:** Not knowing it uses reflection. Not naming the overwrite risk.
- **Follow-up questions:** How would you avoid reflection? (Explicit mapper per band, or a typed merge.)

### Q14. What is a "dispatch-only" sub-report and why do you still emit a row?
- **Why the interviewer asks:** Checks if you understand static Jasper pages.
- **Ideal answer:** Many sub-reports are `SELECT 1 FROM DUAL` with static text (ComplaintRedress, Domestic/Overseas_ATC, ScheduleOfBenefits_ASP/AMC, ImportantNotes). The handler returns a constant and S12 merges an empty row only to show "this band printed". The wording belongs to the template, not the data service.
- **Common mistakes:** Thinking the handler queries Mongo for them.
- **Follow-up questions:** Why merge, not append? (Appending multiplies rows.)

### Q15. Some Jasper conditions use `$V{PAGE_NUMBER}`. What did you do?
- **Why the interviewer asks:** Real-world gap handling.
- **Ideal answer:** Page numbers exist only at render time. Those conditions mostly insert blank filler pages so a document starts on an odd page (duplex). Java implements the data part only and drops the page-parity part: `Blank_NoMargin` is added whenever `070412 && cutoff=Y`, the PD2 blank before the debit note is not implemented, POSMESRA blanks are partial, PrintSeriesFooter call 2 is skipped. Comments in S12 document it. The renderer must supply parity logic.
- **Common mistakes:** Saying it was forgotten. It was a conscious, documented gap.
- **Follow-up questions:** Risk? (Extra or missing blank pages; unconfirmed with the renderer.)

### Q16. PD2 has `false &&` bands in Jasper. What does Java do?
- **Why the interviewer asks:** Detects whether you read print conditions carefully.
- **Ideal answer:** Five PD2 bands start with `false &&` (PDS_Elite, PDS_Income, CommonReport PDPA, NominationForm, a Blank after GST), so Jasper never prints them. Java does not implement PDPA(CommonReport), NominationForm and the blank. But PDS_Income and PDS_Elite are wired in S12 and tested, so Java emits a PDS marker row for INCOMESHIELD and ELITESHIELD that Jasper never produced. The code comment says "per explicit instruction". It is a known deviation to confirm with the product owner.
- **Common mistakes:** Reading only the inner element condition.
- **Follow-up questions:** How did you find these? (Reading `printWhenExpression` of each band.)

### Q17. Name two other PD2 differences between Jasper and Java.
- **Why the interviewer asks:** Depth check.
- **Ideal answer:** (1) Domestic/Overseas ATC: Jasper prints only for product 070406; Java fires one of them for every product (no product test), so non-ATC policies get an Overseas_ATC marker. (2) Java always calls ParticularOfInsuredTpl4/Base, NominationPA4A/BMW and InstallmentDetails in the PD2 block, which have no band in the PD2 jrxml; and the merged method hard-codes `certNo = ""`, while the old PD2 class read `certNo` from the request. Tests encode some of these as expected behaviour.
- **Common mistakes:** Not knowing tests lock in deviations.
- **Follow-up questions:** How would you fix? (Product gate for ATC; confirm the extra bands with the business; read `certNo`.)

### Q18. Why does each Landscape row become its own `reportData`?
- **Why the interviewer asks:** Checks the "1 sub-report = 1 reportData" rule.
- **Ideal answer:** `toLandscapeReportData` wraps one `transDet` in a fresh `PlineReportData`, sharing branch/company info and one `endorsementDetails` by reference. The idea is that each row is an independent print unit. A comment in S12 warns that the renderer has not confirmed this, and that `endorsementDetails` is "last call wins".
- **Common mistakes:** Presenting it as final and confirmed.
- **Follow-up questions:** What would you ask the report team? (Does one `reportData` equal one page group?)

### Q19. What does "ASP" mean?
- **Why the interviewer asks:** Trap: honesty test.
- **Ideal answer:** I did not find a written expansion in code or jrxml. Evidence: internal names `ASPlusSTI` and `ASPlusLLB`, benefit codes `ASP-A1..ASP-D5`, personal-accident style benefits, sponsor and nominee tables, and Java ties it to product code `070412`. I treat it as a group personal-accident style product. [CONFIRM: if you know the real expansion from the business, say it.]
- **Common mistakes:** Inventing an expansion.
- **Follow-up questions:** How do you know 070412? (Java comments and the EndtDocument2 ComplaintRedress gate.)

### Q20. Four Jasper ASP reports, one bean. Explain.
- **Why the interviewer asks:** Same merge pattern as Template12; tests if you can repeat it precisely.
- **Ideal answer:** `AspEndtDocumentService` is `@Service("AGIC/UW/Plines/ASPEndtSchedule/EndtDocument")`. It also does EndtDocument2, AddMemberLandscape and DeletionMemberLandscape. `AspEndtDocument2Service` has `@Service` commented; `AspEndtDocumentLandscapeService` has both `@Bean` lines commented. Re-enabling them would clash bean keys and render twice. Each original `printWhenExpression` became a gate in `execute`. Note: the class javadoc still says "dispatched via reportName" but the code never reads `reportName` (stale text).
- **Common mistakes:** Believing the landscape reports are called from EndtDocument in Jasper. They are separate top-level reports.
- **Follow-up questions:** Why is it a risk that landscape dispatch runs on every call? (Extra queries; always two dispatch rows.)

### Q21. What is the shape of the ASP response?
- **Why the interviewer asks:** Output contract knowledge.
- **Ideal answer:** `reportData[0]` is a master row (`mainReportName=ASPEndtScheduleEndtDocument`, `subReportName=PrintSeriesFooter`, with product code, SST label, issue date, `actionCode`=ENDT_CODE, `prnCnt`). Then one `reportData` per sub-report that fired: EndtCover, ComplaintRedress, EndtSchedule, DebitNote, EndtTaxDoc, ServiceTaxInvoice (+EndtMT rows, QRSection), Transitional invoices, then landscape dispatch and landscape rows. Test `allGatesOpen_oneReportDataPerSubreportInSubreportOrder` locks the order.
- **Common mistakes:** Saying one big `transDet`. Assuming the order equals print order (it is a convention - inferred).
- **Follow-up questions:** Why is ComplaintRedress before EndtSchedule? (Not Jasper print order; layout is the renderer's job - inferred.)

### Q22. When is the ASP service-tax invoice printed?
- **Why the interviewer asks:** Classic business rule question.
- **Ideal answer:** `showTaxInvoice=Y` and `SHOW_STAX_INV_IND=Y` and `checklistCopy != Y` and `SHOW_EINV_IND=Y` (from the REST call `fnEinvoicePrintEndtInd`). Then `AspServiceTaxInvoiceHandler` runs. EndtMT/EndtMTSub rows are added when `STAX_COUNT > 1`. QRSection only if `EINV_CONSENT_IND=Y`, `EINV_STATUS=VALID` and `QR_IND=Y`. Tests: `showServiceTax_checklistCopyY_closesGate`, `showServiceTax_staxCountNotAboveOne_skipsEndtMTAndSub`.
- **Common mistakes:** Forgetting the checklist copy exclusion or that the REST call can fail to "N".
- **Follow-up questions:** What if the e-invoice API is down? (Swallowed, defaults, the page silently disappears.)

### Q23. How do debit and credit notes work in ASP?
- **Why the interviewer asks:** Domain rule with a subtle difference.
- **Ideal answer:** Gate: `showDebitNote=Y` and `PREM != 0` (PREM = `DIFF_PREM_DUE != 0`). `AspDebitNoteHandler` reads `UWGE_VI_ENDT_DEBIT`. Note number is `D`+`ENDT_NO` if `PREM_DUE >= 0`, else `C`+`ENDT_NO`; flag DEBIT/CREDIT. For products 089332, 089432, 089329, 089429 fee differences are added to gross premium. The service-tax invoice uses `GROSS_PREM >= 0` instead (same as Jasper). If `PREM_BY_MST_IND=Y`, master customer details are added.
- **Common mistakes:** Using the same field for both documents.
- **Follow-up questions:** Rounding? (`ROUNDED_NETT_PREM`: Oracle ROUND vs Mongo `$round` half-even - inferred edge case on .5.)

### Q24. Tell me about the CONTRACT_ID type mismatch bug.
- **Why the interviewer asks:** A good bug story with root cause.
- **Ideal answer:** Some Jasper sub-reports declare `pContractId` as String, but the column is numeric; Oracle converts silently. In Mongo `CONTRACT_ID` is stored as a number and matching is type-strict, so a String filter matches nothing. The shared base handler therefore returned null tax flags, and tax sub-reports silently never printed. Fix: `safeNum` (`Long.parseLong`, fallback to raw string) in ASP handlers, and an ASP-only override of `buildCommonPipeline` in `AspEndtDocument2MainHandler`. The fix commits (578778080, ba9e29586) are on other branches, not HEAD.
- **Common mistakes:** "Just call toString on both sides."
- **Follow-up questions:** How do you detect this class of bug? (Zero rows with no error; compare with a known contract in Compass.)

### Q25. Which ASP parts are stubs?
- **Why the interviewer asks:** Honesty about limits.
- **Ideal answer:** `PlineEndtScheduleOfBenefitsAspHandler`, `PlineAspLlbHandler`, `PlineAspStiHandler` return `dualValue=1` and the service maps nothing (Jasper pages are static text, shown when ENDT_CODE is 72 or 168). ComplaintRedress and AntiRasuahBanner are called but their results are discarded. Open question: does the renderer print the schedule-of-benefits pages from the master row's `actionCode`? Unknown.
- **Common mistakes:** Saying data is missing. The pages are static.
- **Follow-up questions:** Would you add a marker `reportData`? (Probably, after confirming with the renderer team.)

### Q26. What are INST_IND and SHOW_GST_LABEL, and what does Java do?
- **Why the interviewer asks:** Knows the gaps.
- **Ideal answer:** In Jasper, `INST_IND` = 'Y' when ISSUE_DATE >= the `SAPM_SYS_CONSTANTS` value (`CUT_OFF_RPT`, `<product>_ENDT`) and `PMT_FREQUENCY <> 'AN'`. `SHOW_GST_LABEL` depends on the SST rollout date function. In Java `AspEndtScheduleHandler` reads `PMT_FREQUENCY` but never uses it; `INST_IND` and `SHOW_GST_LABEL` come from a base document and default to "N". So the cut-off rules are not evaluated in the schedule and debit handlers. The master row reads the label from `UWGE_VI_ENDT_REPORT_PARAM`.
- **Common mistakes:** Claiming parity.
- **Follow-up questions:** Which ASP handler does fetch the rollout date? (`AspServiceTaxInvoiceHandler` via `PlineSSTRolloutDateClient`.)

### Q27. EndtDocument2 prints nothing in Jasper when no premium and no tax difference. Does Java match?
- **Why the interviewer asks:** Edge-case analysis.
- **Ideal answer:** No. Jasper's driver query is a UNION of WIP and posted bases with `DIFF_PREM_DUE != 0 OR DIFF_GST_AMT != 0`; no row means no output. Java's `buildCommonPipeline` always returns a DTO. Also `$ne` of a missing field against 0 is true in Mongo, so PREM can become 1 when `PB_ROW` is null (inferred from Mongo semantics; no test covers it). The debit-note gate could then open when Jasper would not.
- **Common mistakes:** Thinking it is `from dual` like the others.
- **Follow-up questions:** How would you test it? (Handler test with a null `PB_ROW` and assert PREM = 0.)

### Q28. What is the anchor-collection risk in `AspEndtDocumentMainHandler`?
- **Why the interviewer asks:** Knows deep handler risk and branch state.
- **Ideal answer:** On HEAD the pipeline does `$match {ENDT_NO, CONTRACT_ID}` on `UWMT_VI_ENDT_TAX_INV_ALT_UPV`. The shared base class says that collection is not related to the contract and the base fixed it with `$limit 1` only; the ASP override re-introduced the match. A checklist copy has no `ENDT_NO`, so the DTO can be all null and the 72/168 and 070412 gates close. Commit `cc6a1b84d` (OPUSM-41763) on another branch rewrites it to per-field lookups. It is a risk with evidence, not a confirmed production bug.
- **Common mistakes:** Stating it as confirmed production defect.
- **Follow-up questions:** What would you check first for an empty checklist copy? (This anchor, then WIP vs live branch.)

### Q29. What are the six unmapped delete-member fields?
- **Why the interviewer asks:** Detail check on XSD mapping.
- **Ideal answer:** `subcoverPrem`, `endtDoubleIndemnity`, `endtImaFee`, `endtStaxAmt`, `endtLlbPrem`, `endtStiPrem` are fetched but not mapped, with a comment in the service. The XSD does have `endtDoubleIndemnity`, `endtImaFee`, `endtLlbPrem`, `endtStaxAmt`, `endtStiPrem` under `endorsementDetails`, but not as `transDet` leaves. So a mapping decision is pending (put at endorsement level, or add leaves).
- **Common mistakes:** Saying no leaf exists anywhere.
- **Follow-up questions:** Why not just add leaves? (Schema change needs the XSD owner and version bump.)

### Q30. Explain the master policy (MST) logic.
- **Why the interviewer asks:** Multi-step data chain.
- **Ideal answer:** A member policy can point to a master policy by `MST_POL_NO`. `AspDebitNoteHandler.fetchMstContractInfo` reads `UWGE_POLICY_BASES` and `WIP_UWGE_POLICY_BASES` for `MST_POL_NO`, finds the master contract in `OCP_POLICY_BASES.POLICY_REF` (VERSION_NO=1), then reads `PREM_BY_MST_IND`. If Y, the document shows the master policy holder and master TIN/BRN/SST (`CustomerDtlsOfMasterPolicy`). It is 4-5 Mongo round trips, mirroring nested Oracle sub-selects. Failures are swallowed to `premByMstInd="N"`.
- **Common mistakes:** Not knowing the failure becomes "N" silently.
- **Follow-up questions:** Why not one pipeline? (Simpler to debug; each step index-friendly; but more round trips.)

### Q31. How did you handle Oracle's ORA-01427 in ASP?
- **Why the interviewer asks:** Checks honesty on parity.
- **Ideal answer:** ORA-01427 means a single-row sub-select returned many rows. Only `AspServiceTaxInvoiceHandler` emulates it (`requireSingleValue`, `requireNoConflict`, throwing `IllegalStateException`). Other ASP handlers do not, even where Jasper SQL has the same pattern (ERP sub-selects, UNION ALL in the add/delete dispatch, MST sub-select). Template12's PolicyDocument primary does emulate it. So parity is partial.
- **Common mistakes:** Claiming full emulation.
- **Follow-up questions:** Why emulate an error at all? (To avoid hiding bad data and silently choosing a value.)

### Q32. What is RejectRisk?
- **Why the interviewer asks:** Simple module; expect precision.
- **Ideal answer:** A listing of rejected (blacklisted) customers for a created-date range, with reject type, reason, remarks, creator, revoke info and revoke duration. Optional filters: `pCustomerName`, `pIdNo`, `pRejType`, `pReasonCode`. One handler on `CMUW_PL_REFER_RISK`, joined to `CPGE_VI_PARTNERS_ALL` and three `CMGE_CODE` lookups (`PLREJ_RISK_TYPE`, `PLREJ_REASON`, `PLREJ_REVOKE_DUR`). No external REST call.
- **Common mistakes:** Mixing it with the Workmen RejectRisk (different module and DTO).
- **Follow-up questions:** Where does `dataClass` come from? (`RPGE_RPT_TEMPLATE` via `PlineReportHeaderHandler`.)

### Q33. How did you translate the RejectRisk Oracle SQL to Mongo?
- **Why the interviewer asks:** Core skill.
- **Ideal answer:** `(+)` outer join -> `$lookup` with `let`/`pipeline`/`$limit 1` and `$arrayElemAt` (keeps rows without a partner). Three scalar sub-selects -> three `CMGE_CODE` lookups. `ROWNUM` after `ORDER BY NAME_EXT` -> `$setWindowFields` with `$documentNumber` sorted by `customerName`, then `$sort`. `TRUNC(CREATED_DATE) BETWEEN` -> `$gte from` and `$lt (to + 1 day)`. Empty-string filters count as null in both Oracle and Java.
- **Common mistakes:** Using `$unwind` without preserve (drops rows).
- **Follow-up questions:** Which parts are not equal? (NULL ordering, no-dates case, tie order.)

### Q34. Tell me about the date bug in RejectRisk.
- **Why the interviewer asks:** Strong story if you can tell it clearly.
- **Ideal answer:** HEAD built `{$gte: dateFrom, $lte: dateTo}`. `CREATED_DATE` has a time, `dateTo` is midnight, so almost the whole last day was lost, and from = to returned only exact-midnight rows. Fix: add one day to `dateTo` and use `$lt`. It also keeps `CREATED_DATE` as a bare field so an index can be used (no `$dateTrunc`). Also a null bound became `{$gte: null}`, which matches only null dates, so nothing returned; now only supplied bounds are added.
- **Common mistakes:** Saying "null is ignored by Mongo".
- **Follow-up questions:** How to test boundaries? (Seed D 00:00:00, D 23:59:59.999, D+1 00:00:00 with from = to = D.)

### Q35. After your fix, no dates returns all rows. Is that right?
- **Why the interviewer asks:** Spots behaviour change vs Oracle.
- **Ideal answer:** Not necessarily. Oracle's `BETWEEN NULL AND NULL` gives zero rows, and one missing date also gives zero rows. HEAD also returned nothing (by accident). The working tree returns all rows: unbounded, a full scan, and a big response. It needs a business decision: keep parity (empty), require at least one date, or cap rows.
- **Common mistakes:** Calling it simply "fixed".
- **Follow-up questions:** What would you choose? (Require both dates, or keep parity - after asking the business.)

### Q36. What is the time zone risk in RejectRisk?
- **Why the interviewer asks:** Real-data thinking.
- **Ideal answer:** `toDate` builds UTC midnight (`ZoneId.of("UTC")`). If `CREATED_DATE` is a true instant of Malaysian time (UTC+8), a record at 2026-01-05 00:30 MYT is stored as 2026-01-04T16:30Z and falls in the previous window. Other handlers use `timezone: Asia/Kuala_Lumpur`, which suggests true instants (inferred). Check one known record. Also the DTO date is a String made from `Date.toString()` (inferred), which depends on the JVM zone.
- **Common mistakes:** Not checking real data before changing the zone.
- **Follow-up questions:** Better output format? (`$dateToString` with timezone, to `yyyy-MM-dd`.)

### Q37. What is the NULL-ordering difference?
- **Why the interviewer asks:** Classic Oracle-vs-Mongo difference.
- **Ideal answer:** Oracle `ORDER BY NAME_EXT ASC` puts NULLs last. Mongo puts null and missing first. So customers with no partner match or null name appear at the top and take row numbers 1..k, while Oracle shows them at the bottom. The unit test `fetchRejectRisks_finalSortStage_sortsByCustomerNameAscending` locks the plain sort. Fix: sort with a null flag first, or sort in Java with `nullsLast` and number rows there.
- **Common mistakes:** Saying both are the same.
- **Follow-up questions:** Ties? (Add a stable tie-breaker such as `createdDate`, `_id`.)

### Q38. Your uncommitted RejectRisk change equals another commit. What is that?
- **Why the interviewer asks:** Branch awareness and honesty.
- **Ideal answer:** The handler and service changes equal commit `cc6a1b84d` (2026-10-05, "OPUSM-41763 ASP RejectRisk/EndtSchedule/EndtDocument fixes") on branch `origin/OPUSM-41763/retrofit-develop-R2-pline-Template12-ASP-RejectRisk`, which is not in HEAD. The diff to that commit is one comment. So I am re-applying it on `feature/retrofit-develop-updated` and removing a dead `ROW_NUMBER = 1` stage. Also `RejectRiskService` now uses the Plines `DateUtils`, not the motor one. [CONFIRM: how and why you chose to re-apply by hand instead of cherry-pick.]
- **Common mistakes:** Presenting it as original work with no mention of the other branch.
- **Follow-up questions:** Which tests break? (7 of 15 `RejectRiskHandlerTest`.)

### Q39. Which RejectRisk handler tests break and why?
- **Why the interviewer asks:** Do you know the effect of your change on tests?
- **Ideal answer:** From reading code (not run): `dateInstances_usedDirectlyInMatch`, `dateStrings_parsedViaDateUtils`, `nullDates_matchStageHasNullBounds`, `rejTypeOnly_addsSecondMatchStage`, `reasonCodeOnly_addsSecondMatchStage`, `rejTypeAndReasonCode_bothInSameMatchStage`, `customerNameAndIdNoAbsent_noExtraMatchStages`. They assert `$lte`, null bounds and fixed stage indexes. Lesson: find the stage by content, not by index.
- **Common mistakes:** Saying "all tests pass" without running.
- **Follow-up questions:** Which tests pass by accident? (`noOptionalFilters_noSecondMatchStageAdded`.)

### Q40. Why one `reportData` per RejectRisk row?
- **Why the interviewer asks:** Design reason and the open risk.
- **Ideal answer:** In the XSD, `companyDesc` (which holds `remarks`) is a sibling of `transDet` inside `transDetList`. One shared list would give one remarks slot for many rows. One `transDetList` per row ties remarks to the row. After the rows, one extra `reportData` is added (`mainReportName=RejectRisk`, `subReportName=ReportHeader`, `dataClass`). A disclaimer in the service says the renderer has not confirmed it treats each entry as an independent print unit.
- **Common mistakes:** Hiding the disclaimer.
- **Follow-up questions:** Why is the header block missing for zero rows? (Test `execute_emptyRows_returnsEmptyReportDataList` locks it; Jasper still printed title and "No Data Found".)

### Q41. How did you map RejectRisk fields to XSD leaves?
- **Why the interviewer asks:** Checks your XSD work for this module.
- **Ideal answer:** All 17 `transDet` leaves exist in v0.1.6 with matching types: `rownumber` (decimal), `customerName`, `idType1/idNo1`, `idType2/idNo2`, `reasoncode` (lower-case c), `reasonCodeDescription`, `createdBy`, `createdDate` (dateTime), `revokeInd`, `revokedDuration`, `revokedDurationDescription`, `revokeBy`, `revokeDate`, `updatedby`, `updateddate`. Risk type goes to `riskList/risk/rejectRiskType` and `rejectRiskTypeCode`. `REMARKS` goes to `transDetList/companyType/companyDesc/remarks`. Reference: `RejectRisk_XSD_Mapping_sheet.xlsx`.
- **Common mistakes:** Putting remarks in `transDet`. Wrong casing of `reasoncode` / `updatedby`.
- **Follow-up questions:** Which RejectRisk report parameters are not in v0.1.6? (`pReportTitle`, `pCreatedDateFrom`, `pCreatedDateTo`.)

### Q42. Can you say your service output is XSD-valid?
- **Why the interviewer asks:** The most dangerous claim in the project.
- **Ideal answer:** No. Real Java output does not validate against v0.1.6. Reasons found: dates written as `dd/MM/yyyy` while the XSD needs `xs:dateTime`; empty elements for decimal/date/integer/boolean (null with `Include.ALWAYS`); sibling order differences (for example `reportParameters` last, `address` last in agent/branch/comp types); `mainReportName`/`subReportName` and some `reportParameters` not in the XSD. An older saved response gave 929 errors. A hand-ordered XML validated with 0 errors, so the schema is workable. I say: "We aligned payload types and structure to the XSD; full end-to-end validation was still open."
- **Common mistakes:** Claiming "valid" because a test is named `..._generatesValidXml`.
- **Follow-up questions:** What would you fix first? (Order and empty numeric/date elements.)

### Q43. What are `mainReportName` and `subReportName`?
- **Why the interviewer asks:** Recent work (Oct commits) and a known gap.
- **Ideal answer:** `PlineReportNames.tag(...)` fills `mainReportName` on every `reportData` from the bean path (for example `Template12PolicyDocument`, `RejectRisk`); services and `PlineReportBlocks` set `subReportName`. Commits `e74bc0d5c` and `861f0666c` (Oct 2026). They exist in the Java class, but not in the v0.1.6 XSD; code comments cite v0.1.10, and I found no newer XSD on this machine. Any strict validation fails on them.
- **Common mistakes:** Saying the repo XSD defines them.
- **Follow-up questions:** Who owns the newest XSD? [CONFIRM: ask your team/schema owner.]

### Q44. Dates: Java writes `dd/MM/yyyy`, XSD says `xs:dateTime`. What are the options?
- **Why the interviewer asks:** Design decision.
- **Ideal answer:** Every `LocalDateTime` has `@JsonFormat(pattern="dd/MM/yyyy")`, so `06/06/2017` is on the wire, but the XSD wants `2017-06-06T00:00:00`. Options: (a) emit ISO and format in the report (may break Jasper expressions); (b) retype the XSD leaf to a string/pattern type (loses typing); (c) add a parallel string leaf (schema grows). Nobody picked yet; code and XSD disagree. v0.1.6 even retyped `pBeforeEffDate` to dateTime while Java still sends dd/MM/yyyy.
- **Common mistakes:** Saying the XSD controls display format.
- **Follow-up questions:** Which option would you propose? (Decide the contract with the print-engine team first.)

### Q45. What XSD work did you do?
- **Why the interviewer asks:** It is in your Jira scope.
- **Ideal answer:** I mapped each handler DTO field to an existing leaf in `Pline.xsd` / `Common.xsd` (v0.1.6): `transDet`, `endorsementDetails`, `prntDetails/clientInfo|compInfo|branchInfo|agentInfo|taxClientInfo`, `reportParameters`, structured lists (`provisoList`, `nomineeList`, `diagnosisList`, `coverList`, `sobList`, `staxList`). When no leaf existed I left the field unmapped with a comment rather than guess. Related commits: `d4575fa5b` (Template12 + Template4 landscape XSD), `6961abbf9` (align types, XML-legal names, validation test). [CONFIRM: which XSD edits were yours; git authors are other names.]
- **Common mistakes:** Saying you "changed the XSD" when the v0.1.6 on this machine equals the repo copy.
- **Follow-up questions:** Example of an illegal name renamed? (`COLUMN_COUNT` -> `Columncount`, `EMAIL_BCC` -> `Emailbcc`.)

### Q46. What does `PlineXsdValidationTest` really do?
- **Why the interviewer asks:** Tests honesty about test quality.
- **Ideal answer:** It loads `Pline.xsd` (with the relative include `Common.xsd`) and validates a hand-written minimal XML. `minimalShellXml_isValid` and `knownInvalidXml_failsValidation` are real. The four "service" tests (`template12PolicyDocumentService_generatesValidXml`, etc.) validate the same constant string and never call a service. No ASP or RejectRisk XSD test exists. It proves only that the schema loads.
- **Common mistakes:** Quoting the test names as coverage.
- **Follow-up questions:** How would you write a real one? (Mock handlers, run service, serialise with the same `XmlMapper`, collect all errors, assert zero or a ratchet baseline.)

### Q47. What did the tests for your modules cover?
- **Why the interviewer asks:** Evidence of test discipline.
- **Ideal answer:** Roughly 865 `@Test` in files named Template12/Asp/RejectRisk: 282 service, 246 handler, 224 DTO, 113 payload. Service tests mock every handler and assert which branch fired and how fields map (for example `execute_band9_excludedForPoscoverAzp_doesNotFire`, `showDebitNote_withZeroPrem_doesNotFetchDebitNotes`). Handler tests capture the `Aggregation` from a mocked `MongoTemplate` and assert stage shapes. About 39% are Lombok getter/setter tests (inferred as low value). No test run results exist on this machine.
- **Common mistakes:** Quoting coverage percentages (not measured).
- **Follow-up questions:** What do these tests not prove? (Real data behaviour, parity with Jasper, XSD validity.)

### Q48. What is missing in the tests?
- **Why the interviewer asks:** Self-critique.
- **Ideal answer:** No Mongo integration tests (no Testcontainers/embedded Mongo), so null sort, `$lookup` semantics, date boundaries and type mismatch are never checked. No golden comparison with Jasper output. No direct test of `fetchTemplate12PolicyDocumentMain` and its ORA-01427 emulation. No test of the zero-row EndtDocument2 case. Tests for the disabled `Template12PolicyDocument2Service`, `Template12LandscapeService`, `AspEndtDocument2Service`, `AspEndtDocumentLandscapeService` do not exercise the live bean. Lenient Mockito hides dead stubs.
- **Common mistakes:** Saying "coverage is high so it is fine".
- **Follow-up questions:** What would you add first? (A seeded-Mongo test for RejectRisk boundaries.)

### Q49. What is the odd file `AspServiceTaxInvoiceHandler.java` in `src/test`?
- **Why the interviewer asks:** Spots a real repo smell.
- **Ideal answer:** A near-copy (483 lines vs 476) of the production class sits in the test tree with the same package and name, without `@Test`, next to `AspServiceTaxInvoiceHandlerTest` (27 tests). Added by commits `3f2157b28` and `24e50ab9b`. Because Maven puts test classes before main classes on the test classpath (standard expectation, not run), the tests may exercise the copy, and JaCoCo shows the real class as less covered than it looks. Fix: delete the copy and re-run the tests.
- **Common mistakes:** Calling it a harmless helper.
- **Follow-up questions:** How to prove which class is loaded? (Print `getProtectionDomain().getCodeSource()` in a test.)

### Q50. What are the biggest risks in your three modules today?
- **Why the interviewer asks:** Judgement and prioritisation.
- **Ideal answer:** (1) XSD drift: output does not validate; v0.1.6 vs comments for v0.1.10. (2) Renderer contract for multiple `reportData` is unconfirmed. (3) Parity gaps: page parity, ASP `INST_IND`, EndtDocument2 zero row, stubs, three divergent T12 fetches. (4) RejectRisk: time zone, NULL sort, no-dates rule, 7 broken tests, plaintext Mongo URI in the working copy. (5) Platform: no authentication, unknown report name returns SUCCESS, swallowed failures. I would rank (4) secret and tests first because they are cheap and blocking.
- **Common mistakes:** A flat list with no priority.
- **Follow-up questions:** Which one would you fix in a week? (The secret, the tests, then a real XSD validation test.)
