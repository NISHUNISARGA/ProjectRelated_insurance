# 09e - XML Questions 1-50 (response structure and Jackson XmlMapper behaviour)

Scope: the XML string that `CommonController` puts inside `SuccessResponse.data` for Template12, ASP and RejectRisk. Reference schema: `Pline.xsd` + `Common.xsd` v0.1.6 (the only XSD found; identical to `src/test/resources/xsd/plines/`).

Rules for reading this file
- Facts come from KT 04 (XSD/XML deep dive), KT 01-03, the KT Guide and the repo classes (`CommonController`, `Transactions`, `Transaction`, `PlineHeaderTransaction`, `PlinePrntDetails`, `PlineTransDetList`, `PlineReportNames`, `PlineReportData`). "(inferred)" = general Jackson/XSD behaviour or reasoning that I did not run on this repo.
- Real Java output does NOT validate against v0.1.6 (dates `dd/MM/yyyy` vs `xs:dateTime`, empty typed elements, sibling order, `mainReportName`/`subReportName` absent from the XSD). Never say "our output is XSD valid".
- No real incident tickets, no invented numbers. [CONFIRM: ...] marks a personal fact. Git authorship is not attributable to you. Secrets are <redacted>.
- Q1-Q50 here; Q51-Q100 in 09f.

Skeleton used in many answers (path facts from KT 04 section 1.3):
```
transactions
 transaction
  reportParameters | mailingDetails | templateDetails      (siblings, 0..1 each)
  transactionDetails
   headerTransaction (0..n)
    reportTransaction
     reportData (0..n)
      dataClass, [mainReportName, subReportName in Java only], ..., endorsementDetails
      prntDetails
       agentInfo, branchInfo, clientInfo, compInfo, companyType, taxClientInfo, ... paramGroup
       transDetList
        transDet (0..n), companyType
```

---

## A. Structure of the response

**Q1. Draw the path from the root to a `transDet` and name the siblings at each level that matter.**
- Why the interviewer asks: Basic fluency with the contract you worked on.
- Ideal answer: `transactions > transaction > transactionDetails > headerTransaction > reportTransaction > reportData > prntDetails > transDetList > transDet`. Siblings of `transactionDetails` inside `transaction` are `reportParameters`, `mailingDetails`, `templateDetails`. Inside `reportData`: `slNo`, `dataClass`, `permanentDisablement`, `Policyletterind`, ..., `endorsementDetails`, and last `prntDetails`. Inside `prntDetails`: `agentInfo`, `branchInfo`, `clientInfo`, `compInfo`, `companyType`, `taxClientInfo`, `buyerInfo`, `supplierInfo`, `issueBy`, `paramGroup`, `transDetList`. `riskList` sits inside `transDet`. In Java the chain is `Transactions > Transaction > PlineTransactionDetails > PlineHeaderTransaction > PlineReportTransaction > PlineReportData > PlinePrntDetails > PlineTransDetList > PlineTransDet`.
- Common mistakes: Putting `prntDetails` beside `reportData` (that was the pre-v0.1.3 shape).
- Follow-up questions: Where are the tail leaves of `transactionDetails`? (After `headerTransaction`: `clientType`, `documentNumber`, `isfirstTransactionDtl`, `islastTransactionDtl`, `printType`, `user`, `checklistCopy`.)

**Q2. What is the XSD order of the children of `transaction`, and what does Java emit?**
- Why the interviewer asks: Sibling ordering is a known failing point.
- Ideal answer: XSD: `reportParameters, mailingDetails, templateDetails, transactionDetails`. Java (`motor.requestpayload.Transaction`): `templateDetails, mailingDetails, transactionDetails, reportParameters`. So `reportParameters` comes last in the XML. All containers are `xs:sequence`, so a strict validator rejects it. The class is shared with Motor, Fire and Bond, which is why it was not changed in Plines.
- Common mistakes: Believing order never matters (it matters for validators; for name-based readers it probably does not - inferred).
- Follow-up questions: How would you fix it without breaking other LOBs? (Plines-only subclass or wrapper, or agree a `@JsonPropertyOrder` change with all owners.)

**Q3. Explain the rule "one subreport = one `reportData`" and how it shows in Template12, ASP and RejectRisk.**
- Why the interviewer asks: Core design rule.
- Ideal answer: Each Jasper subreport (a printable unit) becomes one `reportData` with its own `prntDetails/transDetList/transDet`. ASP: `reportData[0]` is a master row (`mainReportName="ASPEndtScheduleEndtDocument"`, `subReportName="PrintSeriesFooter"`), followed by one `reportData` per section that fired (`EndtCover`, `ComplaintRedress`, `EndtSchedule`, one `DebitNote` per row, `EndtTaxDoc`, `ServiceTaxInvoice`, `QRSection`, transitional rows, landscape rows). Template12: the PolicyDocument and PD2 bands are merged into the shared header `reportData` (single-row bands merge onto existing `transDet`), while landscape/POS bands are separate `reportData` entries. RejectRisk: one `reportData` per DB row plus a trailing ReportHeader block. The consuming side's behaviour is not confirmed (disclaimer in `RejectRiskService`).
- Common mistakes: Saying Template12 follows the rule the same way as ASP.
- Follow-up questions: Which test fixes the ASP order? (`allGatesOpen_oneReportDataPerSubreportInSubreportOrder`.)

**Q4. Why does RejectRisk create one `reportData` per row?**
- Why the interviewer asks: Shows you understand the XSD constraint behind a design choice.
- Ideal answer: In the XSD, `remarks` lives at `transDetList/companyType/companyDesc/remarks`, a sibling of `transDet`, not inside it. If many rows shared one `transDetList` there would be a single remarks slot. So each row gets its own `reportData` and `transDetList` with one `transDet` and one `companyType/companyDesc`. After the rows, one extra `reportData` (`mainReportName=RejectRisk`, `subReportName=ReportHeader`, `dataClass`) mirrors the Jasper subreport, only if at least one row exists. Supporting hint: the XSD comment on `reportData` mentions "one row per listing line".
- Common mistakes: Putting `REMARKS` on `transDet`.
- Follow-up questions: What does the disclaimer in the service say? (Report team has not confirmed that each `reportData` is an independent print unit.)

**Q5. Where can `companyType` appear and which one does RejectRisk use?**
- Why the interviewer asks: Easy to get wrong.
- Ideal answer: Two places in the XSD: `prntDetails/companyType` and `transDetList/companyType` (last child of `transDetList`, after `transDet`). RejectRisk fills only the second (`REMARKS` into `companyDesc/remarks`). Java `PlineTransDetList` has `@JsonPropertyOrder({"transDet","companyType"})` to honour that.
- Common mistakes: Putting it under `transDet`.
- Follow-up questions: Why is the placeholder `companyDesc` cleared first? (The constructor adds a default one.)

**Q6. How are `mainReportName` and `subReportName` produced?**
- Why the interviewer asks: Recent work (commits `e74bc0d5c`, `861f0666c`).
- Ideal answer: `PlineReportNames.tag(transactions, serviceClass)` walks `headerTransaction > reportTransaction > reportData` and fills `mainReportName` where it is blank. The name comes from `mainNameOf`: the last segment of the `@Service` bean name, prefixed with the previous segment unless it already starts with it (ignoring underscores). Examples: `.../Template12/PolicyDocument` gives `Template12PolicyDocument`; `.../RejectRisk/RejectRisk` gives `RejectRisk`. Services set `subReportName` themselves (ASP: `PrintSeriesFooter`, `AddMemberLandscape_ASPInsured`, ...; RejectRisk header block: `ReportHeader`); `PlineReportBlocks` writes it for shared blocks. ASP sets four main names itself as constants, for example `ASPEndtScheduleEndtDocument`.
- Common mistakes: Saying they are in the XSD (not in v0.1.6; Java comments cite v0.1.10).
- Follow-up questions: What if the service already set a main name? (Kept; only blanks are filled.)

**Q7. Which `subReportName` values does the merged ASP bean produce, in order?**
- Why the interviewer asks: Tests the merged-bean detail.
- Ideal answer: `PrintSeriesFooter` (master), `EndtCover`, `ComplaintRedress` (product 070412), `EndtSchedule`, `DebitNote` (per row), `CustomerDtlsOfMasterPolicy_Debit`, `EndtTaxDoc`, `ServiceTaxInvoice` (+ `ServiceTaxInvoiceEndtMT`/`...Sub` per row), `QRSection`, `TransitionalInvoice`/`TransitionalInvoice2`, then landscape: `AddMemberLandscapeDispatch`, `AddMemberLandscape_ASPInsured`, `AddMemberLandscape_ASPPremium`, `EndtParticularOfSponsorLandscape_ASP`, `EndtParticularOfNominationLandscape_ASP`, `DeleteMemberLandscapeDispatch`, `DeleteMemberLandscape_ASP`. The two dispatch names are synthetic (no jrxml of that name). The order is a convention, not a print order (inferred; ComplaintRedress comes before EndtSchedule).
- Common mistakes: Assuming the list matches the jrxml names exactly.
- Follow-up questions: What do the main names for landscape rows look like? (`ASPEndtScheduleAddMemberLandscape`, `ASPEndtScheduleDeletionMemberLandscape`.)

**Q8. How does Template12 differ from ASP in `subReportName`?**
- Why the interviewer asks: Not every service does the same.
- Ideal answer: The KT Guide states ASP sets `subReportName` per section, while Template12 does not. Template12 relies on the merged shared header `reportData` with marker rows and `mainReportName="Template12PolicyDocument"` from `tag(...)`. Landscape/POS bands become separate `reportData` entries via `toLandscapeReportData`. So a consumer of Template12 cannot rely on `subReportName` to find a band (verify in the current code before relying on it).
- Common mistakes: Expecting the same naming for all three modules.
- Follow-up questions: Where is the report-level precedence set? (PolicyDocument > Landscape > PD2.)

**Q9. Does the output have a namespace? Why does the schema work with `xs:include`?**
- Why the interviewer asks: Schema basics tied to the project.
- Ideal answer: No. `Pline.xsd` and `Common.xsd` have no `targetNamespace`; `elementFormDefault="qualified"` has no effect without a namespace. `Pline.xsd` uses `xs:include` of `Common.xsd`, which is only possible because both are in the same (no) namespace. The XML root is plain `<transactions>`; the class `Transactions` has `@JacksonXmlRootElement(localName = "transactions")`. Do not add `xmlns`.
- Common mistakes: Mixing up include and import.
- Follow-up questions: What would change if a namespace were added? (Import, prefixes, root element namespace.)

**Q10. Which mapper writes the XML and what does that imply?**
- Why the interviewer asks: Jackson behaviour awareness.
- Ideal answer: `CommonController` creates `new XmlMapper()` per request (lines ~218, 409, 426), from `com.fasterxml.jackson.dataformat.xml`. Plain mapper, no customisation, so behaviour comes from annotations on the payload classes (`@JsonInclude`, `@JsonFormat`, `@JacksonXmlProperty`, `@JacksonXmlElementWrapper`, `@JsonPropertyOrder`). Date format is set per field with `@JsonFormat(pattern="dd/MM/yyyy")` plus `LocalDateTimeSerializer` (so the Java time module is not needed as a registered module for those fields, inferred). The KT suggests sharing one configured mapper bean so tests and production cannot drift. A new mapper per request is also wasteful (inferred).
- Common mistakes: Assuming the Spring `ObjectMapper` bean is used.
- Follow-up questions: Which mapper does the HTTP layer use for JSON? (Jackson 2, forced by `preferred-json-mapper=jackson2`.)

**Q11. Does the response start with an XML declaration `<?xml ...?>`?**
- Why the interviewer asks: Small detail with consumer impact.
- Ideal answer: A plain `XmlMapper.writeValueAsString` does not write an XML declaration unless `ToXmlGenerator.Feature.WRITE_XML_DECLARATION` is enabled (standard Jackson behaviour; inferred, not run here). The sample in KT 04 7.1 shows a declaration because it was hand-written for validation. Check a real response before telling a consumer. A parser usually does not need it.
- Common mistakes: Quoting the sample as real output.
- Follow-up questions: How would you add it? (Enable the feature on the mapper.)

---

## B. Nulls, empty elements, repeated elements

**Q12. What does a `null` field look like in the output and why?**
- Why the interviewer asks: Largest source of validation errors.
- Ideal answer: The Pline payload classes use `@JsonInclude(JsonInclude.Include.ALWAYS)` (for example `PlineReportData`, `PlineTransDetList`, `PlinePrntDetails`, `PlineHeaderTransaction`), so nulls are written, and `XmlMapper` writes a null as an empty element such as `<slNo/>`. `Transaction` itself is `NON_NULL`, so its null children are skipped. In the old real sample, 456 of 929 validation errors came from empty typed elements.
- Common mistakes: Thinking nulls are omitted by default.
- Follow-up questions: Is `<x/>` valid for `xs:string`? (Yes; for decimal, dateTime, integer, boolean no.)

**Q13. Distinguish null, empty string and absent for a consumer.**
- Why the interviewer asks: Semantic care.
- Ideal answer: In Java three states exist: null, `""`, value. In the XML a null and an empty string both look like an empty element to a consumer (inferred; Jackson may write `<x/>` or `<x></x>`, which are the same in XML). An absent element appears only when the class is `NON_NULL`/`NON_EMPTY` or the field is not whitelisted. For `xs:string` all three are schema-valid. For typed leaves only a value or absence is valid. In Jasper, null often means "NA" on the page, so changing null to absent could change what is printed if the report reads the element.
- Common mistakes: Equating empty and absent for the report side.
- Follow-up questions: Which option would you test first with the report team? (`NON_NULL` for typed leaves only.)

**Q14. How are repeated elements written, and what does `useWrapping=false` do?**
- Why the interviewer asks: Core Jackson XML annotation.
- Ideal answer: By default Jackson XML writes a `List` property inside a wrapper element named after the property (`<transDet><transDet>..</transDet></transDet>` shape). `@JacksonXmlElementWrapper(useWrapping=false)` removes the extra wrapper so each item is written as a repeated sibling (`PlineTransDetList.transDet`, `PlinePrntDetails.transDetList`...). The XSD says `transDet` is `0..unbounded` directly under `transDetList`, so no wrapper is wanted.
- Common mistakes: Forgetting the annotation on a new list.
- Follow-up questions: Which XSD "list" elements are real wrappers? (See Q15.)

**Q15. `coverList`, `riskList`, `nomineeList`, `sobList` end in "List". Are they wrappers or elements?**
- Why the interviewer asks: Naming trap.
- Ideal answer: They are real XSD elements inside `transDet`, each holding repeated children: `coverList/cover`, `riskList/risk` (with `coverList`, `provisoList`, `excessList` inside `risk`), `nomineeList/nominee`, `diagnosisList/diagnosis`, `sobList/benefits/benefit`, `staxList/staxPcts`, `memoList/memo`. In Java each is a class with its own list property (not wrapped), so the "List" element exists once and the children repeat. A new list must be modelled the same way.
- Common mistakes: Turning on wrapping and getting double elements.
- Follow-up questions: In which order are the 12 lists inside `transDet`? (After the scalar leaves, XSD order.)

**Q16. A repeated element must be contiguous in `xs:sequence`. Why does it matter?**
- Why the interviewer asks: Order rule for lists.
- Ideal answer: With `xs:sequence`, each declared particle must appear in order; repetitions of the same element must be adjacent. If Java emits `transDet, companyType, transDet` the second `transDet` is invalid. `PlineTransDetList` uses a `@JsonPropertyOrder` of `transDet`, then `companyType`, and builds all `transDet` first. Merged responses must keep all repeated children together.
- Common mistakes: Adding rows to a list in a different object and re-ordering.
- Follow-up questions: Same for `reportData` inside `reportTransaction`? (Yes, `0..unbounded`.)

**Q17. What do the default lists in the payload constructors do to an "empty" response?**
- Why the interviewer asks: Hidden objects.
- Ideal answer: `PlineHeaderTransaction.reportTransaction` defaults to a list with one empty `PlineReportTransaction`, and `PlineReportTransaction.reportData` defaults to one empty `PlineReportData` (KT 04 5.5). So untouched structures still emit an empty `reportData` with many empty children. Also `companyType` adds a default `companyDesc` placeholder. A consumer that counts `reportData` to detect data would be misled.
- Common mistakes: Using the count of `reportData` as a data flag.
- Follow-up questions: What does RejectRisk do for zero rows? (An empty `reportData` list, test `execute_emptyRows_returnsEmptyReportDataList`.)

**Q18. What does `@JsonInclude(NON_NULL)` on `Transaction` change compared with the Pline classes?**
- Why the interviewer asks: Mixed inclusion rules.
- Ideal answer: The motor `Transaction` skips null children (for example, no `reportParameters` element if null), while Pline classes write nulls as empty elements. So the XML mixes two behaviours: sparse at the top, dense below. The default transaction (`createDefaultTransaction`) contains only `mailingDetails` and `templateDetails`.
- Common mistakes: Generalising one rule to the whole tree.
- Follow-up questions: Would you unify them? (Only with the consumer's agreement.)

---

## C. Sibling ordering

**Q19. How does Jackson decide the element order in your payload classes?**
- Why the interviewer asks: Source of the largest structure mismatch.
- Ideal answer: Without `@JsonPropertyOrder`, Jackson normally follows field declaration order for these classes (inferred standard behaviour; no alphabetical sorting is enabled). Only four Plines classes declare an explicit order: `PlinePrntDetails`, `PlineTransDetList`, `PlineTransactionDetails`, `PlinePaymentsType`, and they match the XSD. Others rely on where the author put the field, which differs from the XSD's alphabetical-plus-additions layout.
- Common mistakes: Assuming Lombok or Jackson sorts alphabetically.
- Follow-up questions: What is the XSD ordering rule? (Alphabetical, case-insensitive within a block, "Pline additions" after the Common body.)

**Q20. Give three concrete order mismatches between Java and the XSD.**
- Why the interviewer asks: Shows you did the audit.
- Ideal answer: (1) `transaction`: XSD `reportParameters` first, Java last. (2) `agentInfo`, `branchInfo`, `compInfo`, `taxClientInfo`: XSD `address` first, Java `address` last. (3) `reportData`: Java adds `mainReportName`/`subReportName` after `dataClass`, which the XSD does not know. Others: `templateDetails` order, `clientInfo`, `buyerInfo`, `supplierInfo`, `cover` (`excessList`/`provisoList` last), `risk` (`occpDescp`), `transDet` (`loading` vs `LOADING`, `prem` position, `companionName`).
- Common mistakes: Listing only the first one.
- Follow-up questions: How many Java-only fields exist? (12: 4 in transDet, 6 in reportParameters, 2 in reportData; +1 optional template detail.)

**Q21. `reportData` has `prntDetails` last. What else is in that order?**
- Why the interviewer asks: Details of v0.1.3.
- Ideal answer: `slNo, dataClass, permanentDisablement, Policyletterind, policystatus, requestedby, status, tat, endorsementDetails, prntDetails`. Note `permanentDisablement` and `dataClass` are elements even though the header text says "attribute"; the schema has no `xs:attribute` at all. Java adds `mainReportName` and `subReportName` after `dataClass`.
- Common mistakes: Following the header comment.
- Follow-up questions: When did `permanentDisablement` move here? (v0.1.4, from `reportParameters`.)

**Q22. What is the order inside `prntDetails` and how is it enforced?**
- Why the interviewer asks: One of the classes with explicit order.
- Ideal answer: `@JsonPropertyOrder({"agentInfo","agencyInfo","branchInfo","clientInfo","compInfo","customerServiceInfo","companyType","taxClientInfo","buyerInfo","supplierInfo","issueBy","paramGroup","transDetList"})`. It matches the XSD at this level. The children themselves (`agentInfo` and others) still have `address` in the wrong place.
- Common mistakes: Assuming the child classes are ordered the same way.
- Follow-up questions: Where does `paramGroup` repeat? (`0..unbounded`.)

**Q23. `transactionDetails` has leaves after `headerTransaction`. How is that kept right?**
- Why the interviewer asks: Detail of ordering in `PlineTransactionDetails`.
- Ideal answer: `PlineTransactionDetails` has `@JsonPropertyOrder({"headerTransaction","clientType","documentNumber","isfirstTransactionDtl", ...})`, so the tail leaves (`clientType`, `documentNumber`, `isfirstTransactionDtl`, `islastTransactionDtl`, `printType`, `user`, `checklistCopy`) come after the repeated `headerTransaction`. The Java field for the list is named `plineHeaderTransactions`; the XML name `headerTransaction` is set by annotation (inferred from the `@JsonPropertyOrder` use of `headerTransaction`).
- Common mistakes: Placing tail leaves before the list.
- Follow-up questions: Which headerTransaction leaves are dates? (`effectivedatefrom/to`, `endtdatefrom/to`.)

**Q24. Why is `templateDetails` order wrong even though a code comment says "XSD order"?**
- Why the interviewer asks: Trust the schema, not the comment.
- Ideal answer: `templateDetailsType` in Common.xsd is alphabetical: `batchConfigId, boResId, channel, copyTypeList, dataClass, disallowCopy, docType, linkLocation, outputFormat, outputPath, reportName, sequence, templateId, templateName`. Java writes `channel, outputFormat, templateName, copyTypeList, reportName, outputPath, ...` - the Motor XSD order that the comment refers to. Also `mergeStaticPdfFiles` is an extra element, written only when non-empty.
- Common mistakes: Trusting the comment.
- Follow-up questions: Where do `templateDetails` values come from? (`metadata` in the request.)

---

## D. Names and casing

**Q25. Name the leaves in Pline.xsd that differ only by case or spelling from the Jasper names.**
- Why the interviewer asks: XML is case sensitive.
- Ideal answer: `Policyletterind` (capital P) on `reportData`; `LOADING` and `loading` both in `transDet`; `Columncount` (Jasper `COLUMN_COUNT`), `Emailbcc` (`EMAIL_BCC`), `TableHeaderCOUNT` (`TableHeader_COUNT`), `sumTotalPremiumMEDIX` (`sumTotalPremium_MEDIX`); `PREM` vs `premium`; RejectRisk leaves `reasoncode`, `updatedby`, `updateddate`, `rownumber`, `idType1`/`idNo1`. The v0.1.5 header uses the Jasper spellings, not the real XSD names.
- Common mistakes: Using the header spelling.
- Follow-up questions: How does Java force the right name? (`@JacksonXmlProperty(localName=...)` or field visibility.)

**Q26. Why would Lombok produce the wrong element name, and how is it fixed?**
- Why the interviewer asks: Jackson bean naming.
- Ideal answer: Jackson derives property names from getters by default. Lombok `getPolicyletterind()` gives `policyletterind`. For names with capital first letters or all-caps parts the lower-camel rule changes them. Fix: annotate fields with `@JacksonXmlProperty(localName="...")` and use field visibility (`@JsonAutoDetect`) so the field name wins. This is why `PlineReportData` emits `Policyletterind`, and `PlineTransDet` has `loadingUpper` mapped to `LOADING`.
- Common mistakes: Renaming the getter.
- Follow-up questions: What does `@JsonAutoDetect` change? (Which members Jackson uses: fields instead of getters.)

**Q27. List the irregular transDet names used by RejectRisk and why they matter.**
- Why the interviewer asks: Mapping accuracy.
- Ideal answer: `rownumber` (decimal), `customerName`, `idType1`, `idNo1`, `idType2`, `idNo2`, `riskList/risk/rejectRiskTypeCode`, `riskList/risk/rejectRiskType`, `reasoncode` (lower-case c), `reasonCodeDescription`, `createdBy`, `createdDate` (dateTime), `revokeInd`, `revokedDuration`, `revokedDurationDescription`, `revokeBy`, `revokeDate` (dateTime), `updatedby` (lower-case b), `updateddate` (lower-case d, dateTime). All 17 exist in the XSD (grep-checked). `REMARKS` goes to `companyDesc/remarks`.
- Common mistakes: Using camelCase for `updatedby`.
- Follow-up questions: Which Jasper parameters are not in the XML? (`pCustomerName`, `pIdNo`, `pRejType`, `pReasonCode`: query filters only.)

**Q28. What is the `@JsonIncludeProperties` whitelist on `PlineReportParameters` and why does it exist?**
- Why the interviewer asks: Reason for silently missing parameters.
- Ideal answer: `PlineReportParameters extends` the Motor `ReportParameters`, which has many LOB fields. A whitelist of 73 names stops them leaking into Plines XML. It covers all 67 XSD leaves plus 6 extras (`pCreatedDateFrom`, `pCreatedDateTo`, `pReportTitle`, `pshowEndtSchedule`, `pShowOutPrintCovLetter`, `pShowPDPANotice`). Any setter whose JSON name is not in the list is silently dropped (for example `pContractId`, `pPolicyNo`, `pendtNo`).
- Common mistakes: Adding a field to the class and expecting output.
- Follow-up questions: How do shadow fields help? (They emit the XSD names and suppress base names with `@JsonIgnoreProperties`.)

**Q29. Why were `showCoverNote` and `showTaxInvoice` removed from `reportParameters`, and how did Java cope?**
- Why the interviewer asks: Breaking change handling.
- Ideal answer: v0.1.4 removed them as duplicates of `pshowCoverNote` and `pshowTaxInvoice` (Jasper parameter names). It is a breaking removal. Java uses shadow fields in `PlineReportParameters` plus `@JsonIgnoreProperties` to suppress the base class names and emit only the XSD ones. Template12 still sets `showComplaint`, `showDebitNote`, `showScheduleStd` through override setters.
- Common mistakes: Calling it backwards compatible.
- Follow-up questions: Which parameter changed type in v0.1.6? (`pBeforeEffDate` string to dateTime.)

---

## E. Dates, decimals and types

**Q30. How are dates written and what does the XSD want?**
- Why the interviewer asks: Known hard mismatch.
- Ideal answer: Every `LocalDateTime` field has `@JsonFormat(pattern="dd/MM/yyyy")` and `LocalDateTimeSerializer`, so the wire value is `06/06/2017`. All 64 date leaves in Pline.xsd are `xs:dateTime` (`2017-06-06T00:00:00`); there is no `xs:date`. So every date fails strict validation. The test file even says so in a comment. Jasper reports use `dd/MM/yyyy` for display, so the format was probably chosen for the report side (inferred).
- Common mistakes: Thinking `xs:dateTime` accepts any string.
- Follow-up questions: Which date leaves does RejectRisk fill? (`createdDate`, `revokeDate`, `updateddate`; params `pCreatedDateFrom/To`.)

**Q31. Is an empty `<createdDate/>` valid? Is `<createdDate>06/06/2017</createdDate>`?**
- Why the interviewer asks: Precision.
- Ideal answer: Neither is valid for `xs:dateTime`. Valid: `2017-06-06T00:00:00` (optionally with fractional seconds and a time zone). An empty element is valid only for `xs:string` (nothing is `nillable`). If the date is absent, drop the element or use a nillable declaration.
- Common mistakes: Believing empty is "no value" for all types.
- Follow-up questions: What is the lexical form of a negative or zone date? (Out of scope; know the basic form.)

**Q32. Offer options to fix the date mismatch without breaking Jasper.**
- Why the interviewer asks: Design trade-offs.
- Ideal answer: (a) Emit ISO and let the report format it - breaks Jasper expressions that expect `dd/MM/yyyy`. (b) Change the XSD to a string or a pattern `simpleType` - loses semantic typing but matches reality. (c) Add a parallel string leaf - grows the schema. None has been chosen: code and XSD disagree today. The decision belongs to the schema owner and the report team. I would test (b) first because it changes the contract, not the data (inferred preference).
- Common mistakes: Changing annotations in all classes first.
- Follow-up questions: What did v0.1.6 do to `pBeforeEffDate`? (Retyped to dateTime although Java still sends `dd/MM/yyyy`.)

**Q33. Why does Template12 reformat date strings by reflection at the end?**
- Why the interviewer asks: Detail from `Template12PolicyDocumentService`.
- Ideal answer: Some `transDet` fields are `String` dates built by helper methods; `toLocalDateTime` strips the time. The service reformats every `String` field whose name contains `date` or `dob` to `dd/MM/yyyy` (`S12:2105-2230`, applied `S12:549-554`). So there are three date forms in the system: XSD `dateTime`, Java `LocalDateTime` printed as `dd/MM/yyyy`, and Jasper `dd-MMM-yyyy` passed to subreports.
- Common mistakes: Searching for one date formatter.
- Follow-up questions: What risk does reflection by name have? (A field with "date" in the name that is not a date gets reformatted - inferred.)

**Q34. How are decimals written and can they be invalid?**
- Why the interviewer asks: `xs:decimal` has 387 leaves in Pline.xsd.
- Ideal answer: 387 of Pline.xsd's leaves are `xs:decimal`, backed by `BigDecimal` in Java. Jackson XML writes `BigDecimal` with `toString()` unless plain writing is enabled, which can give exponent form such as `1E+3`; `xs:decimal` does not allow exponents (inferred standard behaviour; not run here). Scale (`6` vs `6.00`) is fine for decimals. An empty element is not valid for decimal (see Q12).
- Common mistakes: Thinking `double` values are safer.
- Follow-up questions: How would you check? (Serialise a `BigDecimal("1E+3")` in a test.)

**Q35. Which leaves have a primitive type mismatch with the XSD?**
- Why the interviewer asks: Result of the audit.
- Ideal answer: One found: `PlineBenefit.benefitSi` is a `String` in Java but `xs:decimal` in the XSD. Others match by the mapping used in the audit (String-xs:string, BigDecimal-xs:decimal, Long/Integer-xs:integer, LocalDateTime-xs:dateTime, Boolean-xs:boolean). `rownumber` is decimal, filled by `toBigDecimal`. `endorsementDetails/contractId` is decimal; `mstContractId` is integer.
- Common mistakes: Reporting date format as a "type mismatch" in the same list (it is a format mismatch).
- Follow-up questions: What breaks if `benefitSi` holds "Up to 10,000"? (Invalid decimal.)

**Q36. Booleans and integers: which leaves exist and how does Java write them?**
- Why the interviewer asks: Complete type coverage.
- Ideal answer: Pline.xsd has 2 `xs:boolean` (`nomineeInd`, `printClauseWordingInd`) and 8 `xs:integer` (for example `perilscount`); Common.xsd has 4 and 15. A Java `Boolean` writes `true`/`false` (the schema's lexical form allows `true/false/1/0`); a null writes an empty element, which is invalid. In the old sample 10 integer and 3 boolean empties were counted.
- Common mistakes: Writing "Y"/"N" into boolean leaves.
- Follow-up questions: Most flags are strings "Y"/"N". Why? (The Jasper flags are strings.)

**Q37. `pFirstPageNumber` is a string in the XSD although it is a number. Why?**
- Why the interviewer asks: Type decision reasoning.
- Ideal answer: Jasper declares it String in PolicyDocument and Integer in PD2/Landscape, so v0.1.4 chose string as the lowest common denominator. The defaults in Java differ per service (Template12 `""`, ASP `"1"`). It is only echoed; the renderer numbers pages.
- Common mistakes: Making it integer.
- Follow-up questions: Other parameters that stayed string after the date sweep? (`llb25Ind`, `pBeforeOneMnth`, `pAgentChannel`.)

---

## F. Mapper features and wire details

**Q38. How does `SuccessResponse` carry the XML?**
- Why the interviewer asks: Wire format.
- Ideal answer: `{ "status": "SUCCESS", "data": "<XML string>", "timestamp": "..." }`. `data` is the output of `xmlMapper.writeValueAsString(transactions)`, so inside JSON it is an escaped string. README says `xmlData`, but the class field is `data` (verify on a live response). Always HTTP 200 with content type JSON.
- Common mistakes: Parsing the whole body as XML.
- Follow-up questions: What escapes happen? (JSON escapes quotes and backslashes; XML escapes `&`, `<`, `>`.)

**Q39. A customer name contains `&`. What does the XML contain?**
- Why the interviewer asks: Escaping responsibility.
- Ideal answer: The XML writer escapes it as `&amp;`; JSON then carries that string unchanged (only quotes, backslashes and control characters are JSON-escaped). A correct consumer unescapes both layers once. If it shows `&amp;amp;` somewhere, someone escapes twice. Behaviour with illegal XML control characters was not tested (inferred).
- Common mistakes: Escaping again in the service.
- Follow-up questions: How would you test? (Serialise a payload and parse it back.)

**Q40. What decides which controller path writes the XML?**
- Why the interviewer asks: Two code paths.
- Ideal answer: New strategy path (Step 1): `reportStrategies.get(reportName)` then `execute`, then `populateMailingDetails`/`populateTemplateDetails`, then `XmlMapper`, logged at DEBUG. Legacy path: a large `switch`, with `default:` creating a default transaction; the legacy path logs the XML at INFO. All three modules are on the strategy path. `PlineReportNames.tag` is called inside the Plines services, not in the controller, so legacy reports are not tagged (inferred).
- Common mistakes: Thinking `tag` is global.
- Follow-up questions: What does `createDefaultTransaction` contain? (`mailingDetails` and `templateDetails`.)

**Q41. What do `mailingDetails` and `templateDetails` contain and where do they come from?**
- Why the interviewer asks: Elements not produced by the service.
- Ideal answer: Added by the controller from `metadata`: `name`, `email`, `senderMail` for `mailingDetails`; `copyType`, `batchConfigId`, `boResId`, `channel`, `dataClass`, `disallowCopy`, `docType`, `linkLocation`, `outputFormat`, `outputPath`, `reportName`, `templateId`, `templateName` for `templateDetails` (integers via `parseIntSafe`, null on bad input). `mergeStaticPdfFiles` goes in too. A request without `metadata` throws NPE (inferred from code).
- Common mistakes: Looking in the service for these elements.
- Follow-up questions: Which metadata types are dropped? (Booleans, nulls, nested objects.)

**Q42. How would you print the real XML in a unit test?**
- Why the interviewer asks: Practical test skill.
- Ideal answer: Mock the handlers, call `service.execute(request)`, then `new XmlMapper().writeValueAsString(transactions)`, the same call as `CommonController` (ideally via a shared mapper bean so test and production cannot drift). Assert specific paths with XPath or parse into DOM; validate with the XSD using a collecting `ErrorHandler`. The KT sketch uses `RejectRisk` because it has one handler plus `PlineReportHeaderHandler`.
- Common mistakes: Asserting on the full string.
- Follow-up questions: What does the current XSD test do instead? (Validates a hand-written minimal string.)

**Q43. What does "XSD valid" prove about a report, given that everything is optional?**
- Why the interviewer asks: Limits of validation.
- Ideal answer: Almost nothing about completeness: 1062 of 1065 local elements in Pline.xsd are `minOccurs="0"` and required leaves are 0. A minimal shell `<transactions><transaction><reportParameters/><transactionDetails/></transaction></transactions>` is valid. Validity checks order, type and unknown elements only. Completeness has to be tested with business assertions.
- Common mistakes: Using a green validator as a go-live signal.
- Follow-up questions: How would you enforce a required field? (Change `minOccurs` or add a service check.)

**Q44. What is the practical meaning of `xs:sequence` for adding one new leaf?**
- Why the interviewer asks: Schema evolution.
- Ideal answer: The new leaf must be inserted at the same position in the XSD and in the Java class (alphabetical slot for the Pline body or at the end of the additions). Wrong position makes validators reject output even if the name and type are right. Old schemas reject the new element (forward-incompatible); new schemas still accept old files (backward-compatible). A change also needs the mapping sheet, header version and a test.
- Common mistakes: Appending in Java only.
- Follow-up questions: What is the case trap? (`LOADING` vs `loading`.)

**Q45. Which branch edited the repo XSD without bumping the version?**
- Why the interviewer asks: Governance detail from KT 04.
- Ideal answer: Branch `origin/OPUSM-44159-retrofit-develop-pline-jrxml-comparison-fixes`, commit `ea782121d` (2026-09-28), added `leasingBranch` (transDet) and `premiumPrint`, `benefitPrint`, `nomineePrint` (reportData) in the test copy without changing the header version. Not in v0.1.6 reference and not in HEAD of the current branch. This is a version-control smell: schema edits should bump the version and be recorded in the header.
- Common mistakes: Assuming those leaves exist in HEAD.
- Follow-up questions: Which Java-only transDet fields exist already? (`leasingBranch`, `policystatus`, `printingDate`, `state`.)

**Q46. What does `paramGroup` do and when would you use it?**
- Why the interviewer asks: Extensibility.
- Ideal answer: A Common type with `dataType`, `paramCode`, `paramValue`, repeated (`0..unbounded`) in `prntDetails`. A generic name/value carrier. It could carry per-reportData parameters without changing the schema, because `reportParameters` is allowed once per `transaction`. It is not used by the three modules in the evidence I read (inferred).
- Common mistakes: Using it for everything and losing typing.
- Follow-up questions: Where would ASP landscape parameters go? (Per-`reportData` leaves or `paramGroup`.)

**Q47. Why can there be only one `reportParameters` in a multi-`reportData` response, and what is the consequence?**
- Why the interviewer asks: Real design constraint.
- Ideal answer: The schema allows one `reportParameters` (0..1) per `transaction`. The old ASP landscape service built two parameter sets; in a merged response only one survives. Template12 sets report-level leaves with precedence PolicyDocument > Landscape > PD2. Anything that varies per `reportData` must live in that `reportData` or in `paramGroup`.
- Common mistakes: Adding a second `reportParameters` element.
- Follow-up questions: Which values are report-level in Template12? (`parentName`, `planDescp`, `psi`, `pmtDueDate`, `promoAmt` ...)

**Q48. What are the Java-only elements in the output compared with v0.1.6?**
- Why the interviewer asks: Knowledge of the exact gaps.
- Ideal answer: 12 fields: transDet `leasingBranch`, `policystatus`, `printingDate`, `state`; reportParameters `pCreatedDateFrom`, `pCreatedDateTo`, `pReportTitle`, `pshowEndtSchedule`, `pShowOutPrintCovLetter`, `pShowPDPANotice`; reportData `mainReportName`, `subReportName`. Plus the optional `mergeStaticPdfFiles` in `templateDetails`. XSD leaves without a Java field: 0.
- Common mistakes: Quoting 6 and forgetting the others.
- Follow-up questions: Which of these does RejectRisk emit? (`pReportTitle`, `pCreatedDateFrom`, `pCreatedDateTo`, `mainReportName`, `subReportName`.)

**Q49. How can you check, without running the service, whether a Java class matches an XSD type?**
- Why the interviewer asks: Verification technique.
- Ideal answer: Compare field names, order and types by script or reflection (the KT audit did it with awk/comm), or generate classes with `xjc` (not chosen here because Jackson XML and shared Motor classes are used). A reflection test could parse the XSD and compare leaf names to Java fields including `@JacksonXmlProperty` renames; it should also compare order. Counts in the KT: PlineEndorsementDetails 50=50, PlineRisk equal, PlineTransDet 639 scalar + 13 lists vs XSD 634 + 12.
- Common mistakes: Comparing names only.
- Follow-up questions: What would the drift guard catch next time? (Java adding leaves ahead of the XSD.)

**Q50. In one minute, describe the response XML to a new teammate.**
- Why the interviewer asks: Communication.
- Ideal answer: "The service returns a JSON with a `data` string. Inside is `transactions/transaction` with `templateDetails` and `mailingDetails` added from the request metadata, `reportParameters`, and `transactionDetails > headerTransaction > reportTransaction > reportData[]`. Each `reportData` is one printable unit with a name (`mainReportName`, `subReportName`), `prntDetails` and `transDetList > transDet` where about 650 leaves are available. Nulls are written as empty elements. Dates use `dd/MM/yyyy`. It is designed from Pline.xsd v0.1.6 but does not validate against it today: dates, empty typed elements, order, and new elements." [CONFIRM: add which parts you personally mapped.]
- Common mistakes: Claiming validity.
- Follow-up questions: Which gap would you fix first? (Typed empty elements and order, then dates.)
