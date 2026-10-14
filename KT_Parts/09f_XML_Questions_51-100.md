# 09f - XML Questions 51-100 (XSD versions, mapping, validation, governance)

Same rules as 09e. Sources: KT 04 (XSD/XML), KT 01-03, KT Guide, the XSD v0.1.6 folder, the repo. "(inferred)" = reasoning, not run. Real Java output does NOT validate against v0.1.6. No invented incidents; [CONFIRM: ...] marks a personal fact. Git authorship is not attributable to you. Secrets are <redacted>.

---

## G. Schema versions and change types

**Q51. What changed in v0.1.3 and why was it breaking?**
- Why the interviewer asks: Version history knowledge.
- Ideal answer: v0.1.3 (2026-07-14) moved `prntDetails` from a sibling of `reportData` (under `reportTransaction`) to the last child inside `reportData`. "Move only", to match the Bond reference structure. The path changed from `reportTransaction/prntDetails` to `reportTransaction/reportData/prntDetails`, so any XPath on the old path stops matching. Java mirrors it: `PlineReportData` owns `prntDetails`.
- Common mistakes: Calling it a refactor.
- Follow-up questions: How would a name-based consumer notice? (Missing nodes, no error.)

**Q52. What changed in v0.1.4 and which parts are breaking?**
- Why the interviewer asks: The biggest change set.
- Ideal answer: (a) `permanentDisablement` (decimal) moved from `reportParameters` to `reportData`; (b) new string parameters added (`pMstPolHolder`, `pCertInd`, `pcopyType`, `pshowPaymentInfo`, `pshowEinvEmail`, `pchecklistCopy`, `ppidmprint`, `pshowCoverNote`, `pshowTaxInvoice`, `pPrintSeries`, `pFirstPageNumber`, `pProductCode`); (c) removed duplicates `showCoverNote`, `showTaxInvoice`, `companionName` in `compInfo`, `totalpremium` in `reportData`, `prem` in `reportParameters`; (d) added `class` under `taxClientInfo`. Breaking: moves and removals (a, c). Additive: (b), (d). The header says "no structural changes" but contains structural removals.
- Common mistakes: Calling all of v0.1.4 additive.
- Follow-up questions: Why `PREM` and not `premium`? (Jasper field is literally `PREM`; `premium` is the Bond envelope field; both exist in `transDet`.)

**Q53. What changed in v0.1.5 and what is the case trap?**
- Why the interviewer asks: Details at leaf level.
- Ideal answer: Additive: `reportParameters` strings (`pAgentEmailAdd`, `pAgentName`, `pAgentPhoneCode`, `pAgentPhoneNo`, `pAtceInd`, `pBeforeEffDate`, `pBeforeOneMnth`, `pPymtMode`); `transDet` `LOADING` plus premium-particulars sums, `COLUMN_COUNT`, `TableHeader_COUNT`, `EMAIL_BCC`. Real names are `Columncount`, `TableHeaderCOUNT`, `Emailbcc`, `sumTotalPremiumMEDIX`. `EMAIL_BCC` comes from SharedReport and was added on explicit request, outside Plines scope. Traps: `LOADING` vs existing `loading`.
- Common mistakes: Using the underscore spellings from the header.
- Follow-up questions: Where did they trace the leaves? (AutoRenewal sub reports, EndtSchedule, ServiceTaxInvoice, DebitNote.)

**Q54. What changed in v0.1.6 and is it compatible with what Java sends?**
- Why the interviewer asks: The version you work with.
- Ideal answer: (a) `pAgentChannel` (string) added; (b) `pBeforeEffDate` retyped from string to `xs:dateTime`; (c) date sweep kept `llb25Ind`, `pBeforeOneMnth`, `pAgentChannel` as string. (a) is compatible. (b) is not: Jasper populates it with `SimpleDateFormat("dd/MM/yyyy").format(BR_EFF_DATE)` and Java sends `dd/MM/yyyy`; a strict `dateTime` validator rejects `28/07/2026` and an empty string. So the retype is tidy but wire-incompatible until Java emits ISO or the XSD changes back.
- Common mistakes: Saying the retype is harmless.
- Follow-up questions: Which entry point sets `pBeforeEffDate`? (AutoRenewal/RenewalNotificationLetter feeds - not the three modules.)

**Q55. Java comments mention XSD v0.1.7 to v0.1.10. Where are they?**
- Why the interviewer asks: Honesty about missing artefacts.
- Ideal answer: I could not find them. Only v0.1.6 exists on the machine (five copies, same md5 after stripping CR) and in the repo test resources. `git log --all -- '*.xsd'` shows only three commits touching the test copy (`d4575fa5b`, `6961abbf9`, `ea782121d`). Comment counts: "Pline.xsd v0.1.6" about 228, v0.1.7 x2, v0.1.10 x2, "XSD v0.1.9" in `PlineReportParameters`. So `mainReportName`/`subReportName` and `pCreatedDateFrom/To` are defined by comment only. [CONFIRM: ask the schema owner for the newest file.]
- Common mistakes: Quoting a version number as fact.
- Follow-up questions: What risk follows? (Java drifts ahead of the contract.)

**Q56. Classify these changes as safe or breaking: add a leaf, reorder siblings, rename an element, retype string to dateTime, add an unknown element.**
- Why the interviewer asks: Schema evolution judgement.
- Ideal answer: Add optional leaf: safe for name-based readers; old strict schemas reject it. Reorder siblings: breaking for validators, harmless for name-based readers (inferred). Rename or remove: breaking (v0.1.4). Retype string to dateTime: breaking for senders (v0.1.6). Add elements the XSD does not know (`mainReportName`): invalid against the XSD; harmless to name-based readers. Making an optional element required: breaking (never done here). Moves: breaking (v0.1.3, v0.1.4).
- Common mistakes: "Adding is always safe".
- Follow-up questions: How do you roll out an additive change? (Consumer tolerant first, then producer.)

**Q57. Walk through adding a new leaf end to end.**
- Why the interviewer asks: Your Jira scope included XSD changes.
- Ideal answer: (1) Confirm the Jasper source (field/parameter) and the mapping sheet row. (2) Add the element to the XSD in the alphabetical slot with the right type, bump the header version and note the reason. (3) Add the field to the Java class in the same position, with `@JacksonXmlProperty` if the case differs. (4) If it is a report parameter, add it to the `@JsonIncludeProperties` whitelist. (5) Populate it in the service/handler. (6) Add tests including serialisation and validation. (7) Update the mapping sheet. [CONFIRM: which leaves you added yourself.]
- Common mistakes: Forgetting the whitelist for `reportParameters`, then seeing no output.
- Follow-up questions: And for a rename? (Shadow fields and ignore the base name.)

**Q58. Why was Common.xsd left "byte-for-byte unchanged" and how do the `pline*` types relate to it?**
- Why the interviewer asks: Reuse model.
- Ideal answer: Common.xsd is the shared type library for all LOBs. Pline.xsd defines 22 types, most "extension types" such as `plineAgentInfoType` (17 leaves) = the Common body copied plus Pline additions (`agentCatType`, `agentCodeName`, `bancaAgent`, `totalAgentCode`). They copy the body instead of using `xs:extension`, because extension would append children after the base and change order, and Common must stay unchanged. Many Common types (motor, fire, marine) are unused by Plines.
- Common mistakes: Saying they use `xs:extension`.
- Follow-up questions: Why include and not import? (No namespace.)

**Q59. How big is the schema and what does that imply?**
- Why the interviewer asks: Scale awareness.
- Ideal answer: Pline.xsd: 1400 lines, 22 named complexTypes, 1065 elements (546 string, 387 decimal, 64 dateTime, 8 integer, 2 boolean), 1062 `minOccurs="0"`, 15 `maxOccurs="unbounded"`, no simpleTypes, no enumerations, no patterns, no attributes. Common.xsd: 748 lines, 66 types, 404 elements. `transDet` has 634 scalar leaves plus 12 lists; `reportParameters` 67; `endorsementDetails` 50; `risk` 28. Implication: a "bag of leaves" design (`PlineTransDet` is 809 lines) and no value validation (no enumerations).
- Common mistakes: Thinking the XSD enforces allowed values.
- Follow-up questions: Where would enumeration help? (Flags "Y"/"N".)

**Q60. What are the 67 `reportParameters` types in v0.1.6?**
- Why the interviewer asks: Quick fact check.
- Ideal answer: 43 string, 17 decimal, 6 dateTime (`dateValue`, `pBeforeEffDate`, `pLaunchDate`, `pmtDueDate`, `policyGeneratedDate`, `printingDate`), 1 integer (`perilscount`). All optional, in the listed order. Java whitelists 73 names (the 67 plus 6 extras).
- Common mistakes: Remembering only the totals.
- Follow-up questions: Which are dates that Template12 populates? (`pmtDueDate`, `dateValue`.)

---

## H. Mapping: RejectRisk, Template12, ASP

**Q61. Map the RejectRisk Jasper fields to XSD paths from memory (main ones).**
- Why the interviewer asks: Evidence of your mapping work.
- Ideal answer: ROWNUMBER to `transDet/rownumber` (decimal); CUSTOMER_NAME to `customerName`; ID_TYPE_1/ID_NO_1 to `idType1`/`idNo1`; ID_TYPE_2/ID_NO_2 to `idType2`/`idNo2`; REJECT_RISK_TYPE_CODE and REJECT_RISK_TYPE to `riskList/risk/rejectRiskTypeCode` and `rejectRiskType`; REASON_CODE to `reasoncode`; description to `reasonCodeDescription`; REMARKS to `transDetList/companyType/companyDesc/remarks`; CREATED_BY/DATE to `createdBy`/`createdDate`; REVOKE_IND, REVOKED_DURATION (+ description), REVOKE_BY, REVOKE_DATE; UPDATED_BY/DATE to `updatedby`/`updateddate`; DATA_CLASS to `reportData/dataClass`. [CONFIRM: which rows of RejectRisk_XSD_Mapping_sheet.xlsx you drew yourself.]
- Common mistakes: Putting REMARKS in `transDet`.
- Follow-up questions: Which Jasper parameters are only query filters? (`pCustomerName`, `pIdNo`, `pRejType`, `pReasonCode`.)

**Q62. What does RejectRisk put into `reportParameters`?**
- Why the interviewer asks: Parameter handling.
- Ideal answer: `pReportTitle`, `reportName`, `pCreatedDateFrom`, `pCreatedDateTo`. `reportName` is a base-class property that is not in the whitelist, so it is dropped. `pReportTitle`, `pCreatedDateFrom`, `pCreatedDateTo` are whitelisted but not in v0.1.6, so they are schema-invalid. The dates are `LocalDateTime` written `dd/MM/yyyy`. Null titles become `""` (test `execute_reportParameters_nullTitlesDefaultToEmptyString`).
- Common mistakes: Expecting `reportName` in the XML parameters.
- Follow-up questions: What does the Jasper report do with `pCreatedDateFrom`? (Header echo "dd/MM/yyyy", "NA" if null, plus the filter.)

**Q63. Where does RejectRisk `dataClass` come from and where does it appear?**
- Why the interviewer asks: Subreport mapping.
- Ideal answer: `PlineReportHeaderHandler` reads `RPGE_RPT_TEMPLATE` with `DOC_TYPE='OPUS_RPT'`, `UPPER(TEMPLATE_NAME) = UPPER(reportName + '/' + pReportTitle)` and `UPPER(DESCRIPTION) = UPPER(pReportTitle)` (the same as the Jasper `ReportHeader` subreport). The value is set on every row `reportData` and on the trailing ReportHeader `reportData`. Null if no template row matches. It is a classification label (0 Public ... 3 Strictly Confidential).
- Common mistakes: Thinking it is per row data.
- Follow-up questions: Why is it case-insensitive? (Regex with option `i` and `Pattern.quote`.)

**Q64. How does Template12 build its `transDet` rows?**
- Why the interviewer asks: Single-envelope design.
- Ideal answer: One shared header `reportData` (`newSharedReportData`) with `prntDetails/transDetList`. Single-row bands call `mergeSingleTransDet`, which merges non-null fields onto every existing `transDet` by reflection (last non-null wins). Multi-row bands use `addSingleTransDet`. Landscape/POS bands become separate `reportData` entries. `endorsementDetails` is "last call wins". The service warns that consumer behaviour is unconfirmed.
- Common mistakes: Assuming a row per band.
- Follow-up questions: What did commits `7399d2fc1` and `c0c6d449a` change? (Merge of non-null values and dynamic merging.)

**Q65. Which Template12 leaves go to which XSD structures (cover, benefit, nominee, diagnosis)?**
- Why the interviewer asks: Nested list mapping.
- Ideal answer: Cover rows to `transDet/coverList/cover` and `riskList/risk/coverList/cover`; benefits to `transDet/sobList/benefits/benefit`; nominees to `nomineeList/nominee`; diagnosis to `diagnosisList/diagnosis`; service tax lines to `staxList/staxPcts`; memo to `memoList/memo`; company registration number to `compInfo.compBRN`; `source`, `partnerType`, `contractId` to `endorsementDetails`. Constructor counts in KT 04: the T12 base service builds 57 `PlineTransDet`, 15 `PlineRisk`, 15 `PlineCover`, 5 `PlineStaxPcts`, 4 `PlineCompanyDesc`.
- Common mistakes: Putting lists in `transDetList`.
- Follow-up questions: In `benefits`, what is the XSD order? (`benefit, benefitHdr, benefitType, plan, planPrint`.)

**Q66. Which Template12 report-level values go to `reportParameters`, and which are dropped?**
- Why the interviewer asks: Whitelist effect.
- Ideal answer: Reaching the XML: `pCertInd`, `pType`, `parentName`, `planDescp`, `planType`, `promoAmt`, `premiumPrint`, `policyholderName`, `premDueAftPtv`, `premDueRoundedAftPtv`, `prlInd`, `provInd`, `polRef`, `printPoip`, `dateValue`, `pmtDueDate`, `psi`, `pcopyType`, `pchecklistCopy`, `pshowCoverNote`, `showComplaint`, `showScheduleStd`, `showDebitNote`, `pshowTaxInvoice`, `pPrintSeries`, `pShowGstLabel/SstLabel`, `pAtceInd`, `pProductCode`, `pFirstPageNumber`. Dropped: `pContractId`, `pPolicyNo`, `pversionNo`, `pcompanyParamType`, `showPaymentInfo`, `copyType`, `pissueDate`, `psignIssueDate`, `pshowGst/Sst/Tran`, `PMstPolicyNo`, `PContractIdList`, `PCertNoFrom/To`, `PTaxInvNo`, `ShowOutPrintCovLetter` (setter name differs from the whitelisted `pShowOutPrintCovLetter`).
- Common mistakes: Expecting request echoes.
- Follow-up questions: Which comes from the first row of which band? (`parentName`, `planDescp`, `pmtDueDate` from PD2 bands.)

**Q67. Describe the ASP master row.**
- Why the interviewer asks: ASP-specific structure.
- Ideal answer: `reportData[0]`: `mainReportName="ASPEndtScheduleEndtDocument"`, `subReportName="PrintSeriesFooter"`; carries `productcode`, `showSstLabel`, `issuedate`, `actionCode` (= ENDT_CODE), `prnCnt` (print count), `blank` text, `eInvConsentInd`, `endorsementDetails` and the shared `prntDetails`. Because the stub handlers emit nothing for ScheduleOfBenefits, `actionCode` is the only hint left for those pages (whether the renderer uses it is unknown).
- Common mistakes: Ignoring the master row.
- Follow-up questions: Which fields come from which main handler? (`AspEndtDocumentMainHandler` for product/labels/issue date/endt code; `AspEndtDocument2MainHandler` for tax flags and prem.)

**Q68. Which `endorsementDetails` leaves does ASP service tax fill?**
- Why the interviewer asks: Detail knowledge.
- Ideal answer: `staxInvNo`, `staxInvDate`, `oriStaxInvNo`, `staxInvType`, `invNo`, `sstDate`, `maskedTin` (TIN masked to the last 4 characters), `partnerType`, `sstRegisterNo`, from `AspServiceTaxInvoiceHandler`. `endorsementDetails/contractId` is decimal and `mstContractId` (transDet) integer; Java stores `PlineServiceTaxInvoiceDto.contractId` as a String and converts with `toBigDecimal`. Test: `fetchServiceTaxInvoice_contractIdNonNumeric_keptAsStringInMatch`.
- Common mistakes: Showing the full TIN.
- Follow-up questions: How is the TIN masked? (All but the last 4 characters.)

**Q69. How are ASP debit note rows mapped?**
- Why the interviewer asks: Typical per-row structure.
- Ideal answer: One `reportData` per debit/credit row: `transDet` fields `no1`, `polno`, `insuredName`, `agentName`, `debitNoteNo`, `debitCreditFlag`, `gstPct`, `gstAmt`, `totalpremium`, `premDueRounded`, `rebateAmount`, `bankCommission`, `commPct`, `nettPrem`, `roundedNettPrem`, `staxPct/Amt`, fee diffs, `premByMstInd`, `mstContractId`; `clientInfo.longName`; `taxClientInfo.stampDuty/clazz`; `agentInfo.bancaAgent`. Note number is `D`/`C` + `ENDT_NO` by sign of `PREM_DUE`; flag `DEBIT`/`CREDIT`.
- Common mistakes: Mixing `GROSS_PREM` sign (service tax invoice) with `PREM_DUE` sign (debit note).
- Follow-up questions: Which products add fee diffs to gross premium? (089332, 089432, 089329, 089429.)

**Q70. The six ASP delete-member fields have no leaf. Where do the XSD and Java disagree?**
- Why the interviewer asks: Careful reading.
- Ideal answer: The KT Guide says "no XSD leaf" for `subcoverPrem`, `endtDoubleIndemnity`, `endtImaFee`, `endtStaxAmt`, `endtLlbPrem`, `endtStiPrem`. KT 03 says leaves exist under `endorsementDetails` (XSD lines ~607-617) and the Java comment only claims no verified leaf on `transDet`/`PlineCover`. Both can be true: there is no per-member leaf, but contract-level ones exist. Per-member values do not fit per-contract `endorsementDetails` (inferred). I would check the XSD and class myself and ask for per-member leaves.
- Common mistakes: Repeating the guide line without checking.
- Follow-up questions: What does the delete-member row map? (`contractId`, `nameExt`, `gender`, `dateOfBirth`, `idNo`, `joinDate`, `terminateDate`, `memberType`, `occupCode/Descp`, `plancode`, `prem`, `actionCode`, ...)

**Q71. How does ASP share objects between `reportData` entries and what is the risk?**
- Why the interviewer asks: Reference sharing in a mapping layer.
- Ideal answer: `prntDetails`, `endorsementDetails`, `transDetList` are built once; landscape rows reuse the master `branchInfo`/`compInfo` and the same `endorsementDetails` object by reference (`landscapeRow`, lines ~1566-1574). `mergeEndorsement` is last-non-null-wins, so every row sees the final merged state (inferred). If a later step mutates a shared object, earlier rows change in the XML. Safe only if all uses want the same values.
- Common mistakes: Believing each `reportData` is isolated.
- Follow-up questions: How would you test? (Mutate after building and assert on the other rows.)

---

## I. Validation and testing

**Q72. What does `PlineXsdValidationTest` actually test?**
- Why the interviewer asks: Honest description of test value.
- Ideal answer: It loads `/xsd/plines/Pline.xsd` from test resources through a file-backed source (so `Common.xsd` resolves) and builds a `Schema`. `minimalShellXml_isValid` and `knownInvalidXml_failsValidation` prove the harness. Four tests named `template4LandscapeService_generatesValidXml`, `template12LandscapeService_generatesValidXml`, `template12PolicyDocumentService_generatesValidXml`, `template12PolicyDocument2Service_generatesValidXml` validate the same hand-written string and never call a service. There is no ASP or RejectRisk test. A comment says real XML is not used until date format and `transDet/sobList` ordering are reconciled.
- Common mistakes: Claiming XSD coverage.
- Follow-up questions: What would you write instead? (Run the service, serialise, collect all errors.)

**Q73. Design the proper validation test for RejectRisk.**
- Why the interviewer asks: Practical design.
- Ideal answer: Mock `RejectRiskHandler.fetchRejectRisks` and `PlineReportHeaderHandler`, build a request with title and dates, call `new RejectRiskService(handler, header).execute(req)`, serialise with the same `XmlMapper` as `CommonController`, validate with a collecting `ErrorHandler` and assert errors. Use a ratchet (errors <= baseline) first; goal zero. Add a golden XML for one case. The KT contains a sketch; it was not written to the repo.
- Common mistakes: Stopping at the first error.
- Follow-up questions: Why extract a shared mapper bean? (Test and production cannot drift.)

**Q74. How do you read Xerces output? Why are counts doubled?**
- Why the interviewer asks: Interpreting validator results.
- Ideal answer: For a datatype error Xerces reports two messages, `cvc-datatype-valid` and `cvc-type.3.1.3`, so 456 pairs mean 456 problems, not 912. For structure errors it reports one per container (it stops after the first order error there), so the structure count is a lower bound (KT 04, "not tested for every container"). Group errors by element name to see patterns.
- Common mistakes: Reporting the raw total.
- Follow-up questions: What were the samples' numbers? (929 errors = 456 datatype pairs + 17 structure; 1766 for the second file, from older `T12_final_code` output.)

**Q75. What failed in the old real RejectRisk samples and what fails now?**
- Why the interviewer asks: Evidence about conformance.
- Ideal answer: Old samples (`1357766.xml`, `2439275.xml`, 1 Sep 2026): empty decimal/dateTime/integer/boolean elements (456 of 929), and structure errors: `copyTypeList` order, `mailingDetails` before `transactionDetails`, `address` after fields, `clientType` position, `buyerInfo`/`supplierInfo` order, `companionName` in `transDet`, `provisoList` in `cover`, `benefit` in `benefits`, `IDNo` in `nominee`, `occpDescp` in `risk`, `policyletterind` casing. Current code additionally has `mainReportName`, `subReportName`, `pReportTitle`, `pCreatedDateFrom/To`, `dd/MM/yyyy` dates and `reportParameters` last. Some old problems were fixed since (casing, `indicationList`, `pendtNo`), so the old counts are indicative only (derived, not run on HEAD).
- Common mistakes: Quoting old counts as current.
- Follow-up questions: What proves the schema is workable? (A hand-ordered RejectRisk XML validated with 0 errors.)

**Q76. What made the hand-written RejectRisk sample validate with 0 errors?**
- Why the interviewer asks: Positive evidence and what a fix target looks like.
- Ideal answer: Correct order (`reportParameters` first, `mailingDetails`, `templateDetails`, then `transactionDetails`), ISO dates (`2017-06-06T00:00:00`), no empty typed elements, no `mainReportName`/`subReportName`, no unknown parameters, `companyType` inside `transDetList`, `riskList/risk` inside `transDet`. A variant with `mainReportName`, a `06/06/2017` date and `reportParameters` after `transactionDetails` produced 5 errors, all of those categories. This is the target for the real output (validated by a JDK validator in the KT, not by me).
- Common mistakes: Thinking the sample represents real output.
- Follow-up questions: Which of those differences are cheapest to fix in Java? (Remove nulls for typed leaves; order annotations.)

**Q77. Propose a ratchet strategy for XSD conformance in CI.**
- Why the interviewer asks: Practical engineering.
- Ideal answer: Count validation errors for each service's sample output and store a baseline. The test fails if errors exceed the baseline; as fixes land, lower the baseline until zero. Tag with `@Tag("xsd")`. Group errors by category (order, type, unknown element) in the report. Keep a separate list of "accepted deviations" (for example `mainReportName`) until the XSD catches up. This avoids blocking the build for known open items while preventing regression.
- Common mistakes: Making the build red for all existing errors on day one.
- Follow-up questions: What is the first category to fix? (Empty typed elements and order.)

**Q78. Why must the validator load the schema from a file, not a stream?**
- Why the interviewer asks: Mechanics.
- Ideal answer: `Pline.xsd` has `xs:include schemaLocation="Common.xsd"`, a relative reference. A `StreamSource` from an `InputStream` has no base URI, so the include cannot resolve. A file URL (or a custom `LSResourceResolver` / systemId) fixes it. The existing test converts the classpath URL to a `File`.
- Common mistakes: Copying `Common.xsd` into the same stream.
- Follow-up questions: What if the test runs from inside a jar? (A resolver is needed - inferred.)

**Q79. How would you prove that `transDet` ordering in Java matches the XSD?**
- Why the interviewer asks: Verification.
- Ideal answer: Compare declared field order in `PlineTransDet` with the 634 leaves plus 12 lists in the XSD sequence, accounting for `@JacksonXmlProperty` renames. Known differences: `loading` before `LOADING`, `prem` position, `companionName` position, Java-only `leasingBranch`, `policystatus`, `printingDate`, `state`. `PlineTransDetList` declares `transDet` then `companyType`. A reflection test that parses the XSD is the maintainable way.
- Common mistakes: Only comparing the set of names.
- Follow-up questions: What would the Java fix be? (Reorder fields or add `@JsonPropertyOrder`.)

---

## J. Fixes, options and governance

**Q80. How would you fix the empty-element problem without breaking Jasper?**
- Why the interviewer asks: Trade-off thinking.
- Ideal answer: Options: (a) `NON_NULL` or `NON_EMPTY` on typed leaves only; (b) keep elements and make the XSD `nillable` with `xsi:nil`; (c) change the XSD types to string (loses typing). Question for the report team: does the Jasper XML data source need the element to exist (for example to print "NA")? Unknown. Test with one report and the print engine in dev first. The class-level `ALWAYS` is deliberate (the XSD shape was mirrored), so a change is a design decision.
- Common mistakes: Changing all classes to NON_NULL in one commit.
- Follow-up questions: Which option changes the XSD? (b and c.)

**Q81. How would you make the sibling order correct, and what is the blast radius?**
- Why the interviewer asks: Safe change planning.
- Ideal answer: For Plines-only classes, add `@JsonPropertyOrder` (or reorder fields) in `PlineAgentInfo`, `PlineBranchInfo`, `PlineCompInfo`, `PlineTaxClientInfo`, client/buyer/supplier info, `PlineCover`, `PlineRisk`, `PlineTransDet`, `PlineReportData`. For the shared `Transaction` and `TemplateDetails` (used by Motor, Fire, Bond, and others) agree first or create Plines wrappers. Test with unit serialisation plus the XSD ratchet. Name-based consumers should not notice (inferred), but confirm with the print engine.
- Common mistakes: Touching `Transaction` first.
- Follow-up questions: Which classes already have order? (`PlinePrntDetails`, `PlineTransDetList`, `PlineTransactionDetails`, `PlinePaymentsType`.)

**Q82. Contract drift: the Java emits elements the XSD does not know. What do you recommend?**
- Why the interviewer asks: Governance.
- Ideal answer: Decide the contract: "as the XSD says" or "as Jasper reads". Then either update the XSD to what Java emits (add `mainReportName`, `subReportName`, `pCreatedDateFrom/To`, `pReportTitle`, date pattern type) or change Java to the XSD. Avoid both drifting. Bump the version and record the change in the header. The newest XSD is unknown to me, so the first step is to obtain it.
- Common mistakes: Updating only the test copy.
- Follow-up questions: Who signs off? [CONFIRM: schema owner.]

**Q83. Does the print engine validate the XML? Does it care about order?**
- Why the interviewer asks: Do not claim what you do not know.
- Ideal answer: Not evidenced in the repo or my documents. The only machine check is the test above. The reference XSD is described as the contract. If the engine reads by element name, order and extra elements probably do not matter, but names, paths and text formats do (inferred). I would ask the report team for their parser type and a failing example before acting.
- Common mistakes: Stating that the engine "uses the XSD".
- Follow-up questions: What would convince you? (A rejected message or a sample failing request.)

**Q84. The XSD has no enumerations. Does the service enforce values like "Y"/"N"?**
- Why the interviewer asks: Validation boundaries.
- Ideal answer: No schema enforcement: no simpleTypes, enumerations or patterns in Pline.xsd. Flag values come from Mongo/REST data or defaults in the services. Validation of business values is by convention and tests (for example gates use `"Y".equalsIgnoreCase(x)`). Validation annotations exist on the classpath (`spring-boot-starter-validation`) but `PayLoadRequest` has no `@Valid`.
- Common mistakes: Expecting invalid flags to fail.
- Follow-up questions: Where would enumerations help? (Flags and `copyType`.)

**Q85. Why is `permanentDisablement` an element and not an attribute, given the header?**
- Why the interviewer asks: Catching inaccuracies.
- Ideal answer: The v0.1.4 header text says "attribute inside reportData", but the schema has zero `xs:attribute`; it is a decimal element after `dataClass`. Other header inaccuracies: v0.1.4 says "no structural changes" though it removes/moves elements; underscore leaf names in the header differ from real names. Rule: trust the XSD body and grep it.
- Common mistakes: Repeating the header in an interview.
- Follow-up questions: How did you verify? (grep for `xs:attribute` returns nothing.)

**Q86. What is `Emailbcc` doing in a Plines schema?**
- Why the interviewer asks: Scope edge.
- Ideal answer: `EMAIL_BCC` comes from SharedReport `eMailNotificationPolicyReport`, added to Pline.xsd on explicit request in v0.1.5 although outside Plines scope. The XSD name is `Emailbcc`. It shows the schema was used as a shared bag. For my modules it is irrelevant (inferred).
- Common mistakes: Using `EMAIL_BCC` in Java.
- Follow-up questions: What other leaves are outside the three modules? (AutoRenewal parameters such as `pAgentChannel`.)

**Q87. What are the mapping workbooks and how do you use them?**
- Why the interviewer asks: Requirement side of your XSD work.
- Ideal answer: `Pline_MASTER_MAPPING_SHEET_v0.1 (final1) 1 5.xlsx` (tabs include EndtDocument, RejectRisk, PolicyDocument, AutoRenewal ...; generated 2026-07-06, v0.1) and `RejectRisk_XSD_Mapping_sheet.xlsx`. The RejectRisk sheet was read fully by the KT author; the master workbook only skimmed, so I should read the tabs for my modules before an interview. They list Jasper field -> XSD path -> type. I use them to check each leaf and type when adding or fixing mapping. [CONFIRM: which tabs you contributed to.]
- Common mistakes: Saying you know the whole workbook.
- Follow-up questions: Which tab applies to ASP? (`EndtDocument`; PolicyDocument for Template12.)

**Q88. How do Jasper field classes (BigDecimal, Timestamp, String) map to XSD types in this project?**
- Why the interviewer asks: Type mapping rule.
- Ideal answer: String to `xs:string`; BigDecimal to `xs:decimal`; `java.sql.Timestamp` to `xs:dateTime` (Java `LocalDateTime`); Integer to `xs:integer`; Boolean to `xs:boolean`. Example RejectRisk: ROWNUMBER (BigDecimal) to decimal; CREATED_DATE, REVOKE_DATE, UPDATED_DATE (Timestamp) to dateTime. Jasper parameters `pCreatedDateFrom/To` are `java.util.Date`. `pFirstPageNumber` is String in some templates and Integer in others, so the XSD chose string.
- Common mistakes: Mapping every Oracle number to integer.
- Follow-up questions: What is the exception? (`PlineBenefit.benefitSi` String vs decimal.)

**Q89. Why do `LOADING` and `loading` both exist and which does ASP use?**
- Why the interviewer asks: Case collision.
- Ideal answer: `loading` is the older Bond-derived camelCase envelope leaf; `LOADING` was added in v0.1.5 for the Jasper field (decimal). Java has two fields (`loading`, and `loadingUpper` with `@JacksonXmlProperty(localName="LOADING")`). The ASP add-member and delete-member rows set `loading` from the member data (the ASP map lists `loading`); which exact Java setter is used per band should be checked in the code before claiming. XML names are case sensitive so both can appear.
- Common mistakes: Merging them.
- Follow-up questions: Which order does the XSD use? (`LOADING` before `loading`.)

**Q90. How does the repo mark that the XSD used by tests might be older than the code?**
- Why the interviewer asks: Reading code comments critically.
- Ideal answer: `PlineReportData` and `PlineReportNames` say "Pline.xsd v0.1.10"; `PlineReportParameters` says "XSD v0.1.9" for `pCreatedDateFrom/To`; the test XSD header ends at v0.1.6. Comments are the only evidence for newer versions. In an interview I say: "the comments point to a newer schema I could not locate; the validated reference is v0.1.6".
- Common mistakes: Treating comments as the schema.
- Follow-up questions: How would you find the truth? (Ask the schema owner; search the shared drive; compare with the print engine's copy.)

**Q91. Explain why a hand-ordered XML passes but real output fails, in terms of the three kinds of errors.**
- Why the interviewer asks: Synthesis.
- Ideal answer: (1) Order: Java field order differs from the alphabetical-plus-additions XSD order. (2) Type: `dd/MM/yyyy` dates and empty typed elements. (3) Unknown elements: Java-only fields. The schema is workable, as the hand file proves. All three are fixable in Java or the XSD; the work is choosing which side moves.
- Common mistakes: Calling the schema broken.
- Follow-up questions: Which is the highest volume? (Empty typed elements.)

**Q92. How do you handle `transDet` size - about 650 leaves - when only a few are used?**
- Why the interviewer asks: Design critique.
- Ideal answer: The XSD flattened many Jasper fields into one `transDet` type, so every report shares the same bag. With `ALWAYS` inclusion, each `transDet` writes all leaves including empties. Costs: payload size, parsing time, a larger chance of accidental overwrite when merging by reflection. Options: `NON_EMPTY` for the listing family, per-report types in a new XSD version. No measurements exist in the repo.
- Common mistakes: Quoting performance numbers.
- Follow-up questions: Where does the reflection merge touch all fields? (`mergeNonNullFields`.)

**Q93. How are Jasper date/number formats relevant to what Java writes?**
- Why the interviewer asks: Linking presentation to data.
- Ideal answer: Jasper prints dates as `dd/MM/yyyy` (RejectRisk: CREATED_DATE, REVOKE_DATE, UPDATED_DATE; header dates "NA" if null). Java writes dates in the same format into the XML so the report can print them directly, which is why the XML differs from `xs:dateTime`. Sub-reports in Template12 get `issueDate` as `dd-MMM-yyyy`. So format knowledge is spread across three places (inferred reason for the Java format: matching the report's pattern).
- Common mistakes: Assuming Jasper formats an ISO input automatically.
- Follow-up questions: What would an ISO-based design need? (Report-side formatting.)

**Q94. How do you document a deviation so the next person does not repeat the analysis?**
- Why the interviewer asks: Knowledge transfer discipline.
- Ideal answer: Put it where the work is: a row in the mapping workbook, a note in the XSD header change log, a comment citing the evidence (Jasper file and line), and a test that locks the behaviour. Examples already in the repo: the disclaimer in `RejectRiskService`, "CRITICAL FACT #1" in the Template12 service javadoc, the PD2 comment on the over-inclusive blank. Use "(inferred)" for assumptions.
- Common mistakes: Documenting only in chat.
- Follow-up questions: Which deviation list is in KT 02? (D1-D19.)

**Q95. What does `Policyletterind` tell you about the Bond heritage of the schema?**
- Why the interviewer asks: Understanding origins.
- Ideal answer: Pline.xsd "follows the BOND structure" and Common.xsd is the shared type library for Motor, Fire, Marine and Bond. Odd casing and leaves such as `Policyletterind` come from that heritage and from Jasper field names. Hence Plines classes extend Motor classes (`PlineReportParameters extends ReportParameters`) and use shadow fields.
- Common mistakes: Cleaning the names.
- Follow-up questions: What other Bond-style rule exists? (`prntDetails` inside `reportData`.)

**Q96. If you had to change the date rule, which classes would you touch first?**
- Why the interviewer asks: Scope estimation.
- Ideal answer: The `LocalDateTime` fields with `@JsonFormat(pattern="dd/MM/yyyy")` in `PlineTransDet`, `PlineEndorsementDetails`, `PlineHeaderTransaction`, `PlineReportParameters`, `PlineRisk` and others (the 64 date leaves). A central approach: one shared serializer/format constant, or a mapper-level change, so there is a single place. Then fix the String dates in Template12 reflection (`S12:2105-2230`) and `toLocalDateTime` helpers. Tests: serialise and validate. Contract decision first (Q32).
- Common mistakes: Editing leaf by leaf.
- Follow-up questions: Which date strings bypass annotations? (String fields set as text by helper code.)

**Q97. A reviewer asks: why do you trust the XML shape when the validation is weak? Answer honestly.**
- Why the interviewer asks: Probing for overclaiming.
- Ideal answer: I trust it at the unit level only: payload classes mirror the XSD names and types (12 Java-only fields, 0 missing leaves), service tests check mapping per branch, and I traced mapping tables from Jasper. I do not trust end-to-end: no real output is validated in CI, ordering and date format deviate, and the report side has not confirmed the multi-`reportData` assumption. The next step is the ratchet test plus a comparison with the real print engine in dev.
- Common mistakes: Overselling.
- Follow-up questions: What is your evidence for the 12 and 0? (KT 04 section 5.1 audit.)

**Q98. Which XML facts about ASP are still unknown?**
- Why the interviewer asks: Intellectual honesty on the ASP module.
- Ideal answer: The expansion of "ASP" is not in the code (jrxml names `ASPlusSTI`, `ASPlusLLB`; benefit codes `ASP-A1`..`ASP-D5`; Java uses product 070412). Whether the renderer uses `actionCode` for the stub pages (ScheduleOfBenefits, ASPLLB, ASPSTI). Whether `aspa*` leaves in the XSD (a different report family, inferred) matter. Which exact property key the QR client reads. How `common handlers` parse the `issueDate` strings (U7 in KT 03).
- Common mistakes: Inventing an expansion.
- Follow-up questions: Where would you ask? (Business analyst, report team.)

**Q99. In RejectRisk, how do empty-string filters and the XML relate?**
- Why the interviewer asks: Connecting input to output.
- Ideal answer: Empty-string `pCustomerName`, `pIdNo`, `pRejType`, `pReasonCode` mean "no filter" in both Oracle and Java and do not appear in the XML. The XML carries only `pReportTitle`, `reportName` (dropped), `pCreatedDateFrom`, `pCreatedDateTo`. So a reader of the XML cannot see which filters produced it. The listing rows show `idNo1/idNo2`, which come from the partner view (first partner row only).
- Common mistakes: Expecting filter echoes.
- Follow-up questions: Would you add them to `reportParameters`? (Needs XSD change.)

**Q100. Give a 60-second honest summary of your XML/XSD work, with placeholders for personal facts.**
- Why the interviewer asks: Closing question for the XML area.
- Ideal answer: "My scope covered XSD changes for Template12, ASP and RejectRisk. The response is XML in a JSON string. I mapped Jasper fields to `Pline.xsd` v0.1.6 paths, for example RejectRisk REMARKS to `transDetList/companyType/companyDesc/remarks`, one `reportData` per subreport or row, and `mainReportName`/`subReportName` per `reportData`. I know where Java and the XSD still differ: `dd/MM/yyyy` vs `xs:dateTime`, empty typed elements, sibling order, and Java-only elements. I would add a ratchet validation test and agree the date and null strategy with the report team. I did not find a newer XSD than v0.1.6." [CONFIRM: add the specific leaves, sheets or classes you personally changed.]
- Common mistakes: Claiming full conformance or the repo commits as yours.
- Follow-up questions: Which gap would you close first and why?
