# 10g - XSD Questions 1-50 (Pline.xsd + Common.xsd v0.1.6)

Evidence base: `04_XSD_XML_Deep_Dive.md` (all counts and audit results come from there: XSD reading by grep/awk, a scratch JDK validator run, Java source reading), `01_RejectRisk_Deep_Dive.md`, repo test copies `src/test/resources/xsd/plines/Pline.xsd` and `Common.xsd`, and the reference folder `Plines LOB v0.1.6`. Labels: [XSD] read from the schema, [JAVA] read from the repo, [RAN] executed in the earlier analysis with a JDK validator, (inferred) = reasoning only.

Known truths used in every answer: v0.1.6 is the ONLY XSD found on the machine (Java comments mention v0.1.7-v0.1.10, which were not found); the real Java output does NOT validate against v0.1.6; the XSD test only validates a hand-written shell. Personal facts are [CONFIRM: ...].

---

### Q1. Walk me from the root element to a `transDet`, including which elements are siblings.
- **Why the interviewer asks:** Your XSD work was on this tree; you must be able to draw it.
- **Ideal answer:** [XSD] `transactions` (global root) > `transaction` (1..unbounded, referenced with `ref`) > children `reportParameters`, `mailingDetails`, `templateDetails`, `transactionDetails` (all 0..1). Under `transactionDetails`: `headerTransaction` (0..unbounded) > `reportTransaction` > `reportData` (0..unbounded) > `prntDetails` (last child of `reportData`) > `transDetList` > `transDet`. `riskList`, `coverList`, `sobList`, `nomineeList` and other lists are children of `transDet`. `companyType` appears both under `prntDetails` and under `transDetList`.
- **Common mistakes:** Putting `prntDetails` beside `reportData` (the pre-v0.1.3 shape).
- **Follow-up questions:** Which tail leaves follow `headerTransaction` in `transactionDetails`? (`clientType`, `documentNumber`, `isfirstTransactionDtl`, `islastTransactionDtl`, `printType`, `user`, `checklistCopy`.)

### Q2. Only three declarations are not `minOccurs="0"`. Which, and what does that imply?
- **Why the interviewer asks:** Tests whether you understand "XSD-valid" as a weak statement.
- **Ideal answer:** [XSD] Of 1065 `xs:element` declarations in Pline.xsd, 1062 have `minOccurs="0"`. The exceptions are global `transactions`, global `transaction`, and the `ref="transaction" minOccurs="1"`. There are zero required leaves. So an almost empty document such as `<transactions><transaction/></transactions>` is schema-valid. The schema controls order and type, not presence of business data.
- **Common mistakes:** Assuming "valid" means "complete".
- **Follow-up questions:** How would you enforce a mandatory field? (Change `minOccurs` for that leaf, or validate in the service.) What is the risk of making a leaf required? (Breaks senders and Jasper templates that omit it.)

### Q3. There is no `targetNamespace`. What does that mean for the XML we emit?
- **Why the interviewer asks:** Namespaces are a common source of invisible validation errors.
- **Ideal answer:** [XSD] Both files are no-namespace schemas. XML must have no `xmlns` on `<transactions>` and no prefixes; the service emits plain tags via `XmlMapper`. If an `xmlns` were added, every element would fall into a namespace the schema does not declare and fail. `elementFormDefault="qualified"` has no practical effect without a namespace.
- **Common mistakes:** Saying "qualified adds a namespace".
- **Follow-up questions:** What would change if a `targetNamespace` were introduced? (Prefixes/`xmlns` in XML, `xs:import` instead of include, Jackson namespace settings, and every consumer XPath.)

### Q4. Why does `Pline.xsd` use `xs:include` for Common.xsd and not `xs:import`?
- **Why the interviewer asks:** Basic XSD mechanics grounded in the files.
- **Ideal answer:** `xs:include` merges a schema of the same namespace (here none) and requires the same `targetNamespace`; `xs:import` is for a different namespace. Both files are no-namespace, so include works. In tests the relative `schemaLocation="Common.xsd"` only resolves if the base URI is known, so `PlineXsdValidationTest` converts the resource URL to a `File`.
- **Common mistakes:** Using `xs:import` without a namespace attribute and expecting it to merge types.
- **Follow-up questions:** What if the two files had different namespaces? (Use `import` with `namespace`.) What about `xs:redefine`? (Deprecated; avoid.)

### Q5. What is the practical meaning of "everything is `xs:sequence`"?
- **Why the interviewer asks:** The ordering rule explains most real validation errors.
- **Ideal answer:** [XSD] There are zero `xs:all` and zero `xs:choice`. In a sequence, present children must appear in the declared order; optional ones can be skipped; elements of an unbounded list must be contiguous. Name-based readers (XPath) ignore order, but a validating parser does not. Order is the single largest structural cause of failures in the audit.
- **Common mistakes:** Saying order never matters in XML.
- **Follow-up questions:** How does Jackson decide the order? (Field declaration order unless `@JsonPropertyOrder` is present.)

### Q6. Give the type mix in Pline.xsd and say what it tells you about the data.
- **Why the interviewer asks:** Quantified understanding.
- **Ideal answer:** [XSD] 546 `xs:string`, 387 `xs:decimal`, 64 `xs:dateTime`, 8 `xs:integer`, 2 `xs:boolean` (nomineeInd, printClauseWordingInd); 0 `xs:date`, 0 `xs:int`. Decimals dominate numeric data (premiums, taxes, sums), so empty decimals are the biggest failure source. Only 2 booleans: flags are mostly strings such as "Y"/"N".
- **Common mistakes:** Quoting counts from memory without saying they come from grep.
- **Follow-up questions:** How did you count? (`grep -c` on `type="xs:..."`; pitfalls: comments and multi-line declarations.)

### Q7. Why is there no `xs:date`, and what lexical form do the 64 dateTime leaves require?
- **Why the interviewer asks:** Date conflict is central to your XSD story.
- **Ideal answer:** [XSD] Every date leaf is `xs:dateTime`, whose lexical form is `YYYY-MM-DDThh:mm:ss` (optional fraction/zone), for example `2017-06-06T00:00:00`. A date-only or `dd/MM/yyyy` value is invalid. The schema was generated from Jasper types (`java.sql.Timestamp`) and "consistent with every other date leaf".
- **Common mistakes:** Calling it a date type; sending `06/06/2017`.
- **Follow-up questions:** What would `xs:date` change? (Lexical `YYYY-MM-DD`, still not `dd/MM/yyyy`.)

### Q8. Is `<slNo/>` valid? Is `<remarks/>`?
- **Why the interviewer asks:** Empty-element rule, the highest-volume failure.
- **Ideal answer:** `<remarks/>` (xs:string) is valid; `<slNo/>` (xs:decimal) is not. Empty content is valid only for string types (and for types that allow an empty lexical form). For decimal, dateTime, integer and boolean, it is a datatype error. Nothing in the schema is `nillable`. With `@JsonInclude(ALWAYS)` and a plain `new XmlMapper()`, Java nulls become `<x/>`. In the sample output audit this was 456 of 929 errors (387 decimal, 56 dateTime, 10 integer, 3 boolean) [RAN, older output].
- **Common mistakes:** Treating empty as absent.
- **Follow-up questions:** Fix options? (Omit nulls with `NON_NULL`, or add `nillable="true"` and write `xsi:nil`.)

### Q9. What is the difference between omitting an element and sending it empty, for the print engine and for the schema?
- **Why the interviewer asks:** Practical consequences of inclusion settings.
- **Ideal answer:** For the schema: omitted is always valid (minOccurs 0); empty is valid only for strings. For the print engine (Jasper side): unknown. A template that reads an element by name may treat missing and empty differently (null vs empty string / zero). So changing inclusion from ALWAYS to NON_NULL is a contract change that needs consumer testing [CONFIRM: how the print engine reads the XML].
- **Common mistakes:** Changing inclusion without telling the consumer.
- **Follow-up questions:** How would you test? (Render a sample document before and after.)

### Q10. Explain case-sensitivity traps: `Policyletterind`, `LOADING`/`loading`, `PREM`, `Emailbcc`, `Columncount`.
- **Why the interviewer asks:** Real quirks in the schema.
- **Ideal answer:** [XSD] XML is case-sensitive. `reportData` has `Policyletterind` with capital P; `transDet` has both `LOADING` (Jasper BigDecimal field) and `loading` (Bond-derived envelope field); `PREM` (Jasper field) coexists with `premium`; v0.1.5 header names `COLUMN_COUNT`, `TableHeader_COUNT`, `EMAIL_BCC`, `sumTotalPremium_MEDIX` but the real element names are `Columncount`, `TableHeaderCOUNT`, `Emailbcc`, `sumTotalPremiumMEDIX`. Java maps with `@JacksonXmlProperty(localName=...)` (`loading` and `loadingUpper`) and field visibility instead of Lombok getters.
- **Common mistakes:** Trusting the header spellings.
- **Follow-up questions:** How would Lombok getters break this? (They would emit `policyletterind`, as the old Sep sample showed.)

### Q11. The header says `permanentDisablement` and `dataClass` are "attributes". Are they?
- **Why the interviewer asks:** Catching a documentation error.
- **Ideal answer:** No. [XSD] There are zero `xs:attribute` declarations. They are elements of `reportData` (`slNo`, `dataClass`, `permanentDisablement`, `Policyletterind`, `policystatus`, `requestedby`, `status`, `tat`...). The header text is wrong; trust the declarations.
- **Common mistakes:** Writing them as XML attributes.
- **Follow-up questions:** How would Jackson write an attribute? (`@JacksonXmlProperty(isAttribute=true)`.)

### Q12. Explain the v0.1.3 change and who is affected.
- **Why the interviewer asks:** Changelog fluency.
- **Ideal answer:** v0.1.3 (2026-07-14) moved `prntDetails` from a sibling of `reportData` under `reportTransaction` to the LAST child inside `reportData`, to match the Bond reference. Path changes from `reportTransaction/prntDetails` to `reportTransaction/reportData/prntDetails`. It is a breaking structural change for any XPath consumer. Java: `PlineReportData` owns `prntDetails`.
- **Common mistakes:** Calling it a refactor.
- **Follow-up questions:** Why last? (Declared order in `reportData` in the schema.)

### Q13. List the breaking and the additive parts of v0.1.4.
- **Why the interviewer asks:** Precise change classification.
- **Ideal answer:** Breaking: `permanentDisablement` moved from `reportParameters` to `reportData`; `showCoverNote` and `showTaxInvoice` removed in favour of `pshowCoverNote`/`pshowTaxInvoice`; `companionName` kept only in `transDet`; `totalpremium` kept only in `transDet`; `PREM` only in `transDet` (reportParameters `prem` removed). Additive: new string leaves pMstPolHolder, pCertInd, pcopyType, pshowPaymentInfo, pshowEinvEmail, pchecklistCopy, ppidmprint, pshowCoverNote, pshowTaxInvoice, pPrintSeries, pFirstPageNumber, pProductCode; `class` under taxClientInfo. The header claims "no structural changes" - inaccurate.
- **Common mistakes:** Calling v0.1.4 entirely additive.
- **Follow-up questions:** How did Java survive the removal? (Shadow fields and `@JsonIgnoreProperties` in `PlineReportParameters`.)

### Q14. What did v0.1.5 add, and why was `EMAIL_BCC` included?
- **Why the interviewer asks:** Detail plus scope reasoning.
- **Ideal answer:** reportParameters strings: pAgentEmailAdd, pAgentName, pAgentPhoneCode, pAgentPhoneNo, pAtceInd, pBeforeEffDate, pBeforeOneMnth, pPymtMode. transDet: `LOADING` (decimal), sum variables (sumInMCO, sumInPremium, sumLoading, sumMCOCover, sumOptDMA, sumOptIMA, sumOutMCO, sumOutPremium, sumPremiumCover, sumTotalPremium, sumTotalPremiumMEDIX), `Columncount`, `TableHeaderCOUNT`, `Emailbcc`. Traced to AutoRenewal, EndtSchedule, ServiceTaxInvoice, DebitNote. `EMAIL_BCC` exists only in SharedReport `eMailNotificationPolicyReport` and was added on "explicit request" although outside Plines scope. Additive, backward compatible.
- **Common mistakes:** Using the underscore names.
- **Follow-up questions:** Why decimal for sums? (Money.)

### Q15. What happened in v0.1.6 and why is the `pBeforeEffDate` change a problem?
- **Why the interviewer asks:** The central contradiction in the XSD story.
- **Ideal answer:** v0.1.6 (2026-07-28) added `pAgentChannel` (string) and changed `pBeforeEffDate` from `xs:string` to `xs:dateTime`. The Jasper side populates it as `new SimpleDateFormat("dd/MM/yyyy").format($F{BR_EFF_DATE})`, so the value is `dd/MM/yyyy`, which is not a valid dateTime. Java still sends `dd/MM/yyyy`. So the retype is semantically neat but wire-incompatible until the service emits ISO-8601 or the XSD uses a string/pattern type. pAtceInd and llb25Ind were deliberately not re-added (duplicates).
- **Common mistakes:** Saying the retype was harmless.
- **Follow-up questions:** Compatible with senders of free-form strings? (No, narrows validity.)

### Q16. Name inaccuracies in the XSD header you should not repeat in an interview.
- **Why the interviewer asks:** Critical reading.
- **Ideal answer:** (1) "attribute inside reportData" - they are elements. (2) v0.1.4 says "no structural changes" but contains removals/moves. (3) Underscore names in the header differ from real element names. (4) An unrelated branch edit (commit `ea782121d`, 2026-09-28, branch `OPUSM-44159-retrofit-develop-pline-jrxml-comparison-fixes`) added `leasingBranch`, `premiumPrint`, `benefitPrint`, `nomineePrint` to the repo copy without bumping the header version; it is not in v0.1.6 and not in HEAD of the current branch.
- **Common mistakes:** Quoting the header as truth.
- **Follow-up questions:** Why does the extra edit matter? (Drift between copies.)

### Q17. Why does `companyType` exist at two levels and which does RejectRisk use?
- **Why the interviewer asks:** Precise path knowledge for your module.
- **Ideal answer:** [XSD] `prntDetails/companyType` and `transDetList/companyType` (sibling of `transDet`, last child of `transDetList`). RejectRisk maps `REMARKS` to `transDetList/companyType/companyDesc/remarks`, the second place, once per row's `transDetList`. The `companyDesc` type (10 leaves: addr1-3, companyName, companyRegNo, phone, remarks, showGstPct, signature, tinNo) is reused for company header/footer text in Template12 and ASP.
- **Common mistakes:** Putting `remarks` into `transDet`.
- **Follow-up questions:** Why a company type for a remark? (The Jasper layout used that block; inferred from the mapping.)

### Q18. Describe `reportParameters`: size, types and ordering.
- **Why the interviewer asks:** A core container for all three reports.
- **Ideal answer:** [XSD] 67 leaves: 43 string, 17 decimal, 6 dateTime (dateValue, pBeforeEffDate, pLaunchDate, pmtDueDate, policyGeneratedDate, printingDate), 1 integer (perilscount). Roughly alphabetical (case-insensitive) with Pline additions appended. It is 0..1 per `transaction`. Java `PlineReportParameters` extends the shared motor `ReportParameters` and uses a `@JsonIncludeProperties` whitelist of 73 names.
- **Common mistakes:** Assuming reportParameters is per `reportData`.
- **Follow-up questions:** Why a whitelist? (To stop base-class fields of Motor/Fire/Bond leaking into Plines XML.)

### Q19. Which whitelisted names are not in the v0.1.6 XSD?
- **Why the interviewer asks:** Contract drift awareness.
- **Ideal answer:** [RAN via comm] Six: `pCreatedDateFrom`, `pCreatedDateTo`, `pReportTitle`, `pshowEndtSchedule`, `pShowOutPrintCovLetter`, `pShowPDPANotice`. Source comments tag `pCreatedDateFrom/To` as "XSD v0.1.9", a version I could not find. XSD leaves missing from the whitelist: 0. RejectRisk sets `pReportTitle`, `pCreatedDateFrom`, `pCreatedDateTo`, so its reportParameters are schema-invalid against v0.1.6.
- **Common mistakes:** Saying all 73 are in the XSD.
- **Follow-up questions:** What would fix it? (Newer XSD or remove from the whitelist.)

### Q20. Which setters on `ReportParameters` never reach the XML, and why?
- **Why the interviewer asks:** Silent-drop knowledge.
- **Ideal answer:** (inferred from setter names and the whitelist, not executed) Setters writing base-class properties not whitelisted are silently dropped: `pContractId`, `pPolicyNo`, `pversionNo`, `pcompanyParamType`, `pendtNo`, `pRefNo`, `pUserId`, `pissueDate`, `psignIssueDate`, `pshowGst/Sst/Tran`, `reportName`, `showPaymentInfo`, `copyType`, `PMstPolicyNo`, `PContractIdList`, `PCertNoFrom/To`, `PTaxInvNo`, and Template12's `ShowOutPrintCovLetter` (setter name differs from whitelisted `pShowOutPrintCovLetter`). A request parameter being set in Java is not proof it appears in the XML.
- **Common mistakes:** Assuming every setter is serialised.
- **Follow-up questions:** How to verify one? (Serialise and grep, or unit test the JSON property names.)

### Q21. What is the "shadow field" technique in `PlineReportParameters`?
- **Why the interviewer asks:** Java-side XSD alignment technique.
- **Ideal answer:** The base class has properties with names that the XSD removed or spelled differently (`showCoverNote`, `showTaxInvoice`, casing of `ppidmprint`, `pPrintSeries`, `pshowEinvEmail`, `showComplaint`, `showDebitNote`, `showScheduleStd`). The Pline subclass declares new fields with the XSD-correct names, overrides setters (for example `setPshowComplaint`) and hides base names with `@JsonIgnoreProperties`. Only XSD names are emitted. It isolates Plines without editing shared Motor classes.
- **Common mistakes:** Editing the shared base class.
- **Follow-up questions:** Downside? (Duplicated fields and setter confusion, e.g. `setPShowGstLabel` vs `setPshowGstLabel`.)

### Q22. ASP calls `setPshowGstLabel`/`setPshowSstLabel`. Do they reach the XML?
- **Why the interviewer asks:** A specific suspected gap.
- **Ideal answer:** (inferred, not executed) Template12 also calls the upper-case `setPShowGstLabel/SstLabel` which map to the XSD names `pShowGstLabel`/`pShowSstLabel`; ASP calls only the base setters whose properties are in `@JsonIgnoreProperties`, so ASP's GST/SST label flags likely do not reach the XML. I would verify by serialising an ASP result. [CONFIRM: whether this is a known issue.]
- **Common mistakes:** Stating it as proven.
- **Follow-up questions:** Fix? (Call the same override as Template12 or unify the setters.)

### Q23. `mainReportName` and `subReportName`: where do they exist?
- **Why the interviewer asks:** The most recent XSD-visible change in your branch.
- **Ideal answer:** [XSD] Not in v0.1.6 (zero hits; the only `reportName` is `templateDetails/reportName`). [JAVA] `PlineReportData` has both fields (comment cites Pline.xsd v0.1.10); `PlineReportNames.tag()` writes `mainReportName` on every `reportData` of Plines services, and `PlineReportBlocks` writes `subReportName` (commits `e74bc0d5c`, `861f0666c`). [RAN] a document with those elements fails: "Invalid content was found starting with element 'mainReportName'...". The newer XSD was not found.
- **Common mistakes:** Saying they are in the XSD.
- **Follow-up questions:** Who should resolve? (Schema owner; either publish the newer XSD or remove the elements.)

### Q24. What is in `templateDetailsType` and how does Java deviate?
- **Why the interviewer asks:** Common.xsd awareness.
- **Ideal answer:** [XSD] 14 leaves, alphabetical: batchConfigId, boResId, channel, copyTypeList, dataClass, disallowCopy, docType, linkLocation, outputFormat, outputPath, reportName, sequence, templateId, templateName. Java `TemplateDetails` (shared motor class) uses a different order (channel, outputFormat, templateName, copyTypeList, reportName, outputPath...), which a code comment calls "XSD order" (it is the Motor XSD order, not Common.xsd), and adds `mergeStaticPdfFiles` when non-empty (not in the XSD).
- **Common mistakes:** Treating the code comment as authoritative.
- **Follow-up questions:** Blast radius of reordering? (Shared with Motor/Fire/Bond.)

### Q25. The root `Transaction` has the wrong child order. Describe it.
- **Why the interviewer asks:** Concrete audit finding.
- **Ideal answer:** XSD: reportParameters, mailingDetails, templateDetails, transactionDetails. Java `motor.Transaction`: templateDetails, mailingDetails, transactionDetails, reportParameters. Result: invalid ordering for every Plines response. Fixing in the shared class affects other LOBs; a Plines-specific subclass with `@JsonPropertyOrder` is safer.
- **Common mistakes:** "Order is irrelevant."
- **Follow-up questions:** How do you check other LOBs' schemas? (They have their own XSD; unknown to me.)

### Q26. Which types have `address` in the wrong position?
- **Why the interviewer asks:** The pattern behind many ordering errors.
- **Ideal answer:** [XSD] `plineAgentInfoType`, `plineBranchInfoType`, `plineCompInfoType`, `plineTaxClientInfoType` (and the Common client info) list `address` FIRST. [JAVA] `PlineAgentInfo`, `PlineBranchInfo`, `PlineCompInfo`, `PlineTaxClientInfo` hold `address` LAST. Reason: the XSD is generated alphabetically, then Pline additions appended; Java fields were grouped differently. Fix with `@JsonPropertyOrder` or moving fields.
- **Common mistakes:** Fixing XSD order to match Java without consumer agreement.
- **Follow-up questions:** Which direction should the fix go? (Match the published XSD.)

### Q27. What are the order problems in `clientInfo`, `buyerInfo`, `supplierInfo`?
- **Why the interviewer asks:** Completeness of the audit.
- **Ideal answer:** [XSD vs JAVA] `clientInfoType` order starts with address, age, clientName, clientType, dob...; Java starts with longName, clientType, idType... and puts address and mstpoladdress last. `buyerInfo` (11 leaves) order: address1-3, buyerRegNo, buyerSstRegNo, buyerTin, city...; Java starts with name. `supplierInfo` (11 leaves): city, country, name, postCode, state, supplierAdd1...; Java order differs. These are co-inward tax-invoice party blocks.
- **Common mistakes:** Assuming alphabetical equals Java order.
- **Follow-up questions:** Are buyerInfo/supplierInfo populated by your modules? (Not constructed by name in the services; empty defaults from `PlinePrntDetails`.)

### Q28. Which order issues exist in `cover`, `risk` and `transDet`?
- **Why the interviewer asks:** Deep-structure knowledge.
- **Ideal answer:** `cover`: XSD puts `excessList`, `provisoList` before the Pline additions; Java puts additions first and the two lists last. `risk`: `occpDescp` is placed among scalars in XSD (between additionRisk and rejectRiskType), Java moves it to the end of scalars; invalid when present. `transDet`: XSD has `LOADING` before `loading`, `PREM` before `premiumExcludeGST`, `companionName` earlier; Java has `loading` before `LOADING`, `prem` after `polsi`, `companionName` later, plus four extra fields.
- **Common mistakes:** Only checking the top-level containers.
- **Follow-up questions:** Does RejectRisk use `risk`? (Yes, `rejectRiskTypeCode` and `rejectRiskType`.)

### Q29. Which Java classes have `@JsonPropertyOrder` and which do not?
- **Why the interviewer asks:** Mechanism for ordering control.
- **Ideal answer:** [JAVA] Only `PlinePrntDetails`, `PlineTransDetList`, `PlineTransactionDetails` and `PlinePaymentsType` have it, and those match the XSD. The rest rely on field declaration order (Jackson default). Adding `@JsonPropertyOrder({...})` listing XSD order to the other classes, or generating the order list from the XSD, would fix most structure errors.
- **Common mistakes:** Believing alphabetical default.
- **Follow-up questions:** Risk of a long order list for transDet (634+ names)? (Maintenance; generate it.)

### Q30. How many Java fields have no XSD leaf?
- **Why the interviewer asks:** Audit result recall.
- **Ideal answer:** 12 plus one conditional: `PlineTransDet` 4 (`leasingBranch`, `policystatus`, `printingDate`, `state`; `postcode` and `payments` are `@JsonIgnore`), `PlineReportParameters` 6, `PlineReportData` 2 (`mainReportName`, `subReportName`), and `TemplateDetails.mergeStaticPdfFiles` when non-empty. XSD leaves without a Java field: 0 (a naive diff shows 6 casing-only hits: Columncount, Emailbcc, LOADING, PREM, sumTotalPremiumMEDIX, TableHeaderCOUNT, mapped by `@JacksonXmlProperty`).
- **Common mistakes:** Giving the naive-diff number.
- **Follow-up questions:** Which of the 4 transDet extras are also in the branch ea782121d XSD edit? (`leasingBranch`; the others are `reportData` ones.)

### Q31. Which Java classes matched the XSD exactly in field presence?
- **Why the interviewer asks:** Balanced picture.
- **Ideal answer:** `PlineEndorsementDetails` (50 = 50 leaves, order and types identical), `PlineRisk`, `PlineClientInfo`, and the list/type classes: AgentInfo, BranchInfo, CompInfo, TaxClientInfo, CompanyDesc, Buyer/SupplierInfo, Cover, Nominee, Diagnosis, StaxPcts, Benefit(s), Proviso, Memo, Excess, CsLimit, Specification, PremiumProjection, AgencyInfo, Address, ParamGroup (no Java-only fields and no XSD-only leaves). So field presence is aligned (commit `6961abbf9` "Align Pline requestpayload types with XSD"); order and value formatting are the gaps.
- **Common mistakes:** Saying the whole payload is misaligned.
- **Follow-up questions:** What does that commit show? (137 lines changed in `PlineReportParameters`, 82 in `PlineCover`, 74 in `PlineEndorsementDetails`.) [CONFIRM: whether you contributed.]

### Q32. Map the RejectRisk fields onto the XSD (give the main ones).
- **Why the interviewer asks:** Your module's mapping.
- **Ideal answer:** Under `transDetList/transDet`: `rownumber` (decimal) from ROWNUMBER; `customerName`; `idType1`/`idNo1`; `idType2`/`idNo2`; `reasoncode` (lower-case c); `reasonCodeDescription`; `createdBy`; `createdDate` (dateTime); `revokeInd`; `revokedDuration`; `revokedDurationDescription`; `revokeBy`; `revokeDate` (dateTime); `updatedby` (lower-case b); `updateddate` (lower-case d, dateTime). Under `transDet/riskList/risk`: `rejectRiskTypeCode`, `rejectRiskType`. `REMARKS` under `transDetList/companyType/companyDesc/remarks`. `dataClass` under `reportData`. All 17 transDet leaves exist in the XSD with those types.
- **Common mistakes:** Using camelCase `updatedBy`.
- **Follow-up questions:** Where is the mapping workbook? (`RejectRisk_XSD_Mapping_sheet.xlsx`.)

### Q33. Why are RejectRisk's reject type leaves inside a `riskList`?
- **Why the interviewer asks:** Odd schema fit.
- **Ideal answer:** The Pline XSD (Bond-style) has no flat `rejectRiskType` leaf in `transDet`; `risk` has `rejectRiskType` and `rejectRiskTypeCode`, so the mapping used `riskList/risk` with one risk per row (inferred from the mapping table). The Jasper original treated them as plain fields. The mapping sheet records the decision.
- **Common mistakes:** Adding new leaves without schema owner agreement.
- **Follow-up questions:** Alternative? (Request a flat leaf in a new XSD version.)

### Q34. What does RejectRisk emit after the data rows?
- **Why the interviewer asks:** Response shape detail.
- **Ideal answer:** If there is at least one row, one extra `reportData` created by `PlineReportBlocks.reportHeader` with `mainReportName=RejectRisk`, `subReportName=ReportHeader`, and `dataClass` from `PlineReportHeaderHandler` (lookup in `RPGE_RPT_TEMPLATE` by report title and name). It represents the generic ReportHeader sub-report. With zero rows, it is omitted.
- **Common mistakes:** Thinking the header is first.
- **Follow-up questions:** What is `dataClass` for data rows? (Set per row from the same lookup; subReportName empty.)

### Q35. How is `ROWNUMBER` converted into `rownumber`?
- **Why the interviewer asks:** Type conversion in the service.
- **Ideal answer:** Mongo `$documentNumber` yields a number; the DTO field is converted by `toBigDecimal` and set via `setRownumber`, matching `xs:decimal`. A null becomes an empty `<rownumber/>`, invalid, but rows always have a number.
- **Common mistakes:** Mapping to integer when the XSD says decimal.
- **Follow-up questions:** Why is it decimal? (Jasper `BigDecimal`.)

### Q36. How do the REJECT dates reach the XSD, and what is wrong today?
- **Why the interviewer asks:** Date issue applied to your module.
- **Ideal answer:** `CREATED_DATE`, `REVOKE_DATE`, `UPDATED_DATE` are BSON dates, mapped to `LocalDateTime` (`toLocalDateTime`, `atStartOfDay` for request dates) and written with `@JsonFormat(pattern="dd/MM/yyyy")`, e.g. `06/06/2017`. XSD wants `2017-06-06T00:00:00`. The same applies to `pCreatedDateFrom/To` in reportParameters. Jasper displayed `dd/MM/yyyy`, blank when null.
- **Common mistakes:** Saying dates are valid because they are `LocalDateTime`.
- **Follow-up questions:** How to fix without breaking Jasper? (Q51 in the next file.)

### Q37. What does the `dataClass` element do and who sets it?
- **Why the interviewer asks:** Report routing field.
- **Ideal answer:** `reportData/dataClass` (string). In RejectRisk it comes from `PlineReportHeaderHandler` (`RPGE_RPT_TEMPLATE.DATA_CLASS` by `pReportTitle` + `reportName`). Template details also carry `dataClass` from request metadata. The print engine likely uses it to pick the template/data class (inferred). The sample value `RR01` in the KT sample is made up.
- **Common mistakes:** Quoting RR01 as real.
- **Follow-up questions:** What if the lookup returns nothing? (Null dataClass; behaviour downstream unknown.)

### Q38. What is `paramGroup` and when would you use it?
- **Why the interviewer asks:** Schema extensibility.
- **Ideal answer:** [XSD] A `prntDetails` child, 0..unbounded, type `paramGroupType` (dataType, paramCode, paramValue). A generic name/value carrier. Useful for per-block parameters (the schema allows one `reportParameters` per transaction) or for fields that do not yet have a leaf, avoiding a schema change. Trade-off: loses typing and discoverability.
- **Common mistakes:** Overusing it and hiding the contract.
- **Follow-up questions:** Does your code use it? [CONFIRM: usage in modules.]

### Q39. The schema allows one `reportParameters` per transaction. How does Landscape cope?
- **Why the interviewer asks:** Cardinality constraints in real code.
- **Ideal answer:** [JAVA, inferred] ASP landscape builds two `reportData` and two parameter objects; the transaction can serialise only one `reportParameters`, so the second set is not representable validly. Values needed per block would have to live in `reportData` fields or `paramGroup`. The AhiEndtDocumentService comment states the cardinality limit.
- **Common mistakes:** Believing each reportData has parameters.
- **Follow-up questions:** Which parameters get dropped? (The second set; verify by code.)

### Q40. `PlineBenefit.benefitSi` is a Java `String` but the XSD says decimal. What happens?
- **Why the interviewer asks:** The one primitive type mismatch found.
- **Ideal answer:** Serialised text like "100,000.00" or "N/A" would be invalid decimal; plain digits would pass. It is the only primitive mismatch found among compared classes. Fix: change to `BigDecimal` (Template12 benefit mapping would need conversion) or change the XSD type to string if text is truly needed.
- **Common mistakes:** Assuming String-to-decimal conversion happens in Jackson.
- **Follow-up questions:** Why a String? (Probably the Jasper field was a formatted text; inferred.)

### Q41. What is the `benefits` element order and the sample issue?
- **Why the interviewer asks:** Another order detail.
- **Ideal answer:** [XSD] `benefits`: benefit, benefitHdr, benefitType, plan, planPrint. Current `PlineBenefits` puts `benefit` first (matches); the saved 1 Sep sample shows `benefitHdr` first (older code). So current code is fine; old samples are not evidence for current behaviour.
- **Common mistakes:** Using old samples to claim current errors.
- **Follow-up questions:** Where does `sobList` sit? (`transDet/sobList/benefits/benefit`.)

### Q42. How many lists does `transDet` contain and what are they?
- **Why the interviewer asks:** Structure detail.
- **Ideal answer:** [XSD] 634 scalar leaves plus 12 structured lists: coverList, provisoList, excessList, staxList, sobList, nomineeList, diagnosisList, csLimitList, memoList, specificationList, premiumProjectionList, riskList (riskList > provisoList and `risk` 0..unbounded; each risk has 28 scalar leaves plus coverList, provisoList, excessList). Java `PlineTransDet`: 639 scalars + 13 lists.
- **Common mistakes:** Remembering only coverList and riskList.
- **Follow-up questions:** Which list does Template12 PolicyDocument fill? (nominee, benefits, diagnosis, stax, memo, cover, risk.)

### Q43. How does the Java side avoid `<x><x>..</x></x>` wrappers for lists?
- **Why the interviewer asks:** Jackson XML specifics.
- **Ideal answer:** The 04 audit notes that the lists which are unbounded in the XSD are `List` fields with `useWrapping=false` (repeated elements, no extra wrapper). How wrapper elements such as `coverList` > `cover` are modelled in each class is (inferred) a wrapper class or a wrapped list; check the class before answering in detail. `reportTransaction` is 0..1 in the XSD but a `List` in Java (always size one): harmless.
- **Common mistakes:** Forgetting `useWrapping=false` and getting double wrappers.
- **Follow-up questions:** Effect on ordering? (Order inside the wrapper is by field order.)

### Q44. Why does an "empty" response still contain an empty `reportData` with many empty elements?
- **Why the interviewer asks:** Default objects.
- **Ideal answer:** [JAVA] `PlineHeaderTransaction.reportTransaction` defaults to a list with one empty `PlineReportTransaction`, and `PlineReportTransaction.reportData` defaults to one empty `PlineReportData`. With `ALWAYS` inclusion, serialisation prints many empty tags (and empty decimals invalid). The default transaction in the controller (no strategy data) avoids this by not including `transactionDetails`.
- **Common mistakes:** Believing empty means no output.
- **Follow-up questions:** Fix? (Do not initialise lists, or use `NON_EMPTY`.)

### Q45. What does `@JsonInclude(ALWAYS)` plus `new XmlMapper()` do to the payload size?
- **Why the interviewer asks:** Size/performance link.
- **Ideal answer:** Every null scalar becomes an empty element, so each `transDet` can carry hundreds of empty elements (634 scalar leaves possible). XML becomes large and invalid for non-string types. Reducing to `NON_NULL` shrinks the payload and removes most datatype errors, but changes what the print engine sees (Q9 above).
- **Common mistakes:** Not measuring XML size.
- **Follow-up questions:** How to measure? (Log length at DEBUG for a sample.)

### Q46. Why are there Common.xsd types that Plines never uses?
- **Why the interviewer asks:** Reuse model.
- **Ideal answer:** [XSD] Common.xsd (66 named types, 404 elements, no root) is shared across Motor, Fire, Marine and Bond; many types (extraCover*, floatingRisk, ncd, questionnaire, recommendedPerils...) are not used by Plines. Plines includes the file unchanged "byte-for-byte" and defines `pline*` types in Pline.xsd.
- **Common mistakes:** Editing Common.xsd for a Plines change.
- **Follow-up questions:** Why does it matter? (A Common change affects every LOB.)

### Q47. Why do the `pline*` types copy the Common body rather than use `xs:extension`?
- **Why the interviewer asks:** Schema design rationale.
- **Ideal answer:** `xs:extension` appends new elements after the base sequence, so Pline additions would always come last and the alphabetical merged order of the Bond-style "extension types" could not be expressed; copying the body plus additions keeps Common unchanged and allows the intended order. It costs duplication and risk of divergence. (Rationale partly inferred from the header "Common body + Pline additions in the Bond style".)
- **Common mistakes:** Claiming extension is always better.
- **Follow-up questions:** Downside of copying? (Common fixes do not propagate.)

### Q48. What is `plineStaxListType`/`PlineStaxPctsType`?
- **Why the interviewer asks:** Tax block knowledge.
- **Ideal answer:** [XSD] Service-tax lines: Common `staxPcts` body plus 12 Pline fields, used by `transDet/staxList`. Template12 base service builds 5 `PlineStaxPcts`, ASP 1. Used for GST/SST percentage and amount lines on invoices.
- **Common mistakes:** Mixing with `taxClientInfo`.
- **Follow-up questions:** Which handler builds it? [CONFIRM by code search.]

### Q49. How many `endorsementDetails` leaves are there and who uses it?
- **Why the interviewer asks:** ASP relevance.
- **Ideal answer:** [XSD] 50 leaves: 32 string, 8 decimal, 9 dateTime, 1 integer, inline under `reportData`. Java `PlineEndorsementDetails` matches exactly. ASP builds `PlineEndorsementDetails`; Template12 base service also does. Dates inside it have the same dd/MM/yyyy conflict.
- **Common mistakes:** Forgetting it is only in `reportData`.
- **Follow-up questions:** Example leaf? (`endtNo`, `endtEffDate` as in the KT skeleton.)

### Q50. Describe `nominee` naming irregularity and the sample issue.
- **Why the interviewer asks:** Details in case/shape.
- **Ideal answer:** The nominee type has `IDNo` in upper case (Java sample showed an error with `IDNo` in nominee order in the old sample). There are two nominee types: Common `nomineeType` and `plineNomineeType` (13 leaves). `nomineeInd` is one of only two booleans in Pline.xsd. Template12 builds 4 `PlineNominee` objects in PolicyDocumentService, ASP 1.
- **Common mistakes:** Using `idNo`.
- **Follow-up questions:** What does `nomineeInd` hold? (xs:boolean: `true/false/1/0` only; "Y"/"N" is invalid.)
