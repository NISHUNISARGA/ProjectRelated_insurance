# 10h - XSD Questions 51-100 (Pline.xsd + Common.xsd v0.1.6)

Evidence base: `04_XSD_XML_Deep_Dive.md`, `05_Testing_Deep_Dive.md`, `01_RejectRisk_Deep_Dive.md`, repo test XSD copies and `PlineXsdValidationTest`. Labels: [XSD] read from schema, [JAVA] read from repo, [RAN] executed earlier with a JDK validator on scratch copies, (inferred) = reasoning only. Known truths: v0.1.6 is the only XSD found; real Java output does NOT validate against it; the existing XSD test validates only a hand-written shell. Personal facts: [CONFIRM: ...].

---

### Q51. A date leaf must stay `dd/MM/yyyy` for Jasper but the XSD says `xs:dateTime`. What are your options?
- **Why the interviewer asks:** The key design decision in your XSD scope.
- **Ideal answer:** (a) Emit ISO-8601 and let the report format it; cleanest for the schema, but Jasper expressions that read the string must change. (b) Retype the leaf to `xs:string` or to a custom `simpleType` with a `xs:pattern` for `dd/MM/yyyy`; keeps Jasper working but loses date typing. (c) Add a parallel string leaf for display and keep the dateTime one; schema grows. In the repo neither side moved: code and XSD disagree (v0.1.6 even retyped `pBeforeEffDate` to dateTime). Whichever is chosen, it must be agreed with the schema owner and the report side [CONFIRM: owner].
- **Common mistakes:** Picking one without naming the consumer impact.
- **Follow-up questions:** Which affects fewest places? ((b) for the XSD side; (a) for correctness.)

### Q52. How is the `dd/MM/yyyy` format produced in Java and where would you change it?
- **Why the interviewer asks:** Concrete code-level knowledge.
- **Ideal answer:** [JAVA] Every `LocalDateTime` field in the Pline payload classes has `@JsonFormat(pattern="dd/MM/yyyy")` together with a `LocalDateTimeSerializer`. Changing the pattern to `yyyy-MM-dd'T'HH:mm:ss` on each field (or a shared serializer/mixin) would give ISO output. It applies to all dateTime leaves the code populates (createdDate, revokeDate, updateddate, pBeforeEffDate, pLaunchDate, dateValue, pmtDueDate, policyGeneratedDate, printingDate, header effectivedatefrom/to, endtdatefrom/to, riskDob, riskEffDate, riskExpDate...). A central mixin reduces edits.
- **Common mistakes:** Changing a global ObjectMapper and affecting Motor/Fire.
- **Follow-up questions:** Does the test file mention this? (Yes, a comment in `PlineXsdValidationTest` says real XML is not used until date formatting is reconciled.)

### Q53. How many errors did the saved RejectRisk samples produce and why do the numbers need care?
- **Why the interviewer asks:** Evidence quality.
- **Ideal answer:** [RAN] `1357766.xml`: 929 errors = 456 datatype pairs (empty decimal/dateTime/integer/boolean, each reported twice) + 17 structure errors. `2439275.xml`: 1766 errors. These files are dated 1 Sep 2026 and come from the older `T12_final_code` copy, not the current branch (they lack `mainReportName`, show `policyletterind` in lower case). So they are historical, a lower-bound style estimate, and not proof of current counts.
- **Common mistakes:** Quoting 929 as the current error count.
- **Follow-up questions:** What did the experiment on a hand-written file show? (0 errors for a correctly ordered file; 5 errors when mainReportName/subReportName, `06/06/2017` and reportParameters last were added.)

### Q54. Why does Xerces report each datatype error twice?
- **Why the interviewer asks:** Reading validator output.
- **Ideal answer:** For a bad simple-type value it emits `cvc-datatype-valid.1.2.1` (value not valid for the type) and `cvc-type.3.1.3` (element content not valid), so counts double. 456 datatype "pairs" in the audit means about 228 bad values (inferred from the pairing statement in the KT notes; verify with a deduplicated count).
- **Common mistakes:** Treating total error lines as the number of problems.
- **Follow-up questions:** How to deduplicate? (Group by element path.)

### Q55. Why are the 17 structure errors a lower bound?
- **Why the interviewer asks:** Validator behaviour.
- **Ideal answer:** Xerces stops reporting structural problems after the first error in each content model (it cannot continue matching a sequence once it is off the rails), so each container yields at most one structure error. After you fix the first, the next appears. This is standard behaviour, not tested for every container in the audit.
- **Common mistakes:** Claiming "17 problems in total".
- **Follow-up questions:** How to find all? (Fix iteratively in a ratchet test.)

### Q56. What exactly does `PlineXsdValidationTest` contain?
- **Why the interviewer asks:** Know your test and its limits.
- **Ideal answer:** [JAVA] Commit `6961abbf9`. It loads `/xsd/plines/Pline.xsd`, converts the URL to a `File` so the relative include resolves, builds a `Schema` with `SchemaFactory.newInstance(W3C_XML_SCHEMA_NS_URI)`, then: `minimalShellXml_isValid` (`<transactions><transaction><reportParameters/><transactionDetails/></transaction></transactions>`), `knownInvalidXml_failsValidation` (`<bogusUnknownElement>` expects `SAXException`), and four "service" tests (template4Landscape, template12Landscape, template12PolicyDocument, template12PolicyDocument2) that validate the same hand-written shell. It never calls a service or serializer.
- **Common mistakes:** Presenting it as proof that Template12 output is valid.
- **Follow-up questions:** Did you write it? [CONFIRM: authorship; git cannot attribute to you.]

### Q57. Why must the test use a `File`-backed source for the XSD?
- **Why the interviewer asks:** Include resolution.
- **Ideal answer:** `Pline.xsd` contains `<xs:include schemaLocation="Common.xsd"/>`, a relative reference. With an `InputStream` source, the parser has no base URI, so it cannot find `Common.xsd` (or looks relative to the working directory). A `File` or URL-based `StreamSource` with a system id lets the relative path resolve next to `Pline.xsd`.
- **Common mistakes:** Loading as a string and wondering about "unable to resolve".
- **Follow-up questions:** Alternative? (Set `LSResourceResolver` on the `SchemaFactory`.)

### Q58. How would an `LSResourceResolver` improve the test and is it safe?
- **Why the interviewer asks:** Security-aware XML tooling.
- **Ideal answer:** A resolver maps `Common.xsd` to a classpath resource explicitly, so tests do not rely on file paths (works inside jars). Combine with `XMLConstants.ACCESS_EXTERNAL_SCHEMA` set to empty/`file` to prevent remote schema fetches. For validation of our own generated XML there is no XXE risk since inbound is JSON, but secure processing is good practice.
- **Common mistakes:** Allowing network access for schemaLocation.
- **Follow-up questions:** Which properties? (`FEATURE_SECURE_PROCESSING`, `ACCESS_EXTERNAL_SCHEMA`, `ACCESS_EXTERNAL_DTD`.)

### Q59. The test method names say "generates valid XML". Is that accurate?
- **Why the interviewer asks:** Honesty about coverage.
- **Ideal answer:** No: the four service tests assert nothing about generated XML; they re-validate the same shell. A reviewer could believe coverage exists. Rename to `...shellIsValid` or replace with real tests. Be direct about it in an interview: "the existing test proves the schema loads, nothing more".
- **Common mistakes:** Defending the names.
- **Follow-up questions:** What is the minimal real test? (RejectRisk service output validation with mocked handlers.)

### Q60. Design the real RejectRisk XSD test.
- **Why the interviewer asks:** Core test design for your module.
- **Ideal answer:** Mock `RejectRiskHandler` and `PlineReportHeaderHandler`; build a `PayLoadRequest` with `pReportTitle`, `reportName`, `pCreatedDateFrom/To`; call `new RejectRiskService(...).execute(req)`; serialise with the same mapper as `CommonController` (extract a shared bean); validate with an error-collecting `ErrorHandler` and assert the list. Use fully populated sample DTO rows (dates included) plus one null-name row. The 04 guide contains a sketch; nothing was written to the repo.
- **Common mistakes:** Using `fail on first error` handler.
- **Follow-up questions:** Why populate all fields? (Nulls hide type errors in other leaves, and empty strings pass.)

### Q61. Why collect all errors instead of failing on the first?
- **Why the interviewer asks:** Practical validation approach.
- **Ideal answer:** The default JDK handler throws on the first fatal error, hiding the rest. A custom `ErrorHandler` that records `SAXParseException`s lets you see categories (order, type, unknown element) and counts. That enables a ratchet: assert `errors.size() <= baseline` and lower the baseline as fixes land, so the build is not blocked on day one.
- **Common mistakes:** Making the test blocking before any fixes exist.
- **Follow-up questions:** Where to store the baseline? (A constant or a file in `src/test/resources`.)

### Q62. What is a ratchet test and why use it here?
- **Why the interviewer asks:** Practical approach to improving a failing contract.
- **Ideal answer:** A test that fails if the error count rises above a recorded baseline and requires lowering the baseline when it falls. Given hundreds of known violations (order, dates, unknown elements), a hard "zero errors" test is not possible yet; the ratchet prevents regressions and shows progress. Tag as `@Tag("xsd")`.
- **Common mistakes:** Letting the baseline drift upward.
- **Follow-up questions:** Alternative? (Category-specific zero targets, e.g. "no unknown elements".)

### Q63. Why should test and production share one `XmlMapper` bean?
- **Why the interviewer asks:** Avoiding drift.
- **Ideal answer:** `CommonController` uses `new XmlMapper()` inline. A test that builds its own mapper with different settings (inclusion, modules) may pass while production fails. A shared `@Bean XmlMapper` configured once (inclusion, JavaTimeModule, date format) makes the test faithful and allows central fixes (NON_NULL, ISO dates). Changing it affects all LOBs, so scope carefully (Plines-specific mapper bean).
- **Common mistakes:** Changing the shared mapper behaviour for all LOBs without regression tests.
- **Follow-up questions:** How to scope? (A qualifier-based Plines mapper selected by the strategy.)

### Q64. How would a reflection-based drift test work?
- **Why the interviewer asks:** Automation of the audit you did manually.
- **Ideal answer:** Parse `Pline.xsd` (DOM) to get ordered child element names and types per complex type; reflect on the Java class: names from `@JacksonXmlProperty`/`@JsonProperty`/field names, order from `@JsonPropertyOrder` or declaration, types from field classes. Assert: no Java-only names unless in an allow-list (the 12 known), no XSD-only leaves, same order, compatible types (String-string, BigDecimal-decimal, LocalDateTime-dateTime). The allow-list documents accepted drift.
- **Common mistakes:** Comparing only names.
- **Follow-up questions:** Why not codegen? (Q65.)

### Q65. Why not generate Java classes from the XSD with `xjc`?
- **Why the interviewer asks:** Tooling trade-off.
- **Ideal answer:** Payload classes are Jackson XML POJOs shared with Motor/Fire/Bond (subclassing, whitelists, shadow fields), with Lombok and custom serializers. xjc generates JAXB classes with different annotations and no support for those reuse patterns; the project has an existing serialisation pipeline. Generated code would also inherit the schema's drift. A generator that only produces `@JsonPropertyOrder` lists could be a middle ground (inferred).
- **Common mistakes:** Saying generation is impossible.
- **Follow-up questions:** Jackson 3 plans? (Annotations would move; not now.)

### Q66. Where would Schematron (or service validation) belong for business rules?
- **Why the interviewer asks:** Limits of XSD.
- **Ideal answer:** XSD cannot express "if `checklistCopy` is Y then some leaf is required" or cross-field rules. Options: Schematron rules (extra tool), or Java validation in the service before serialisation. Since every element is optional, business completeness is only in service code and tests. I would start with targeted unit tests of the service for required business leaves rather than adding Schematron.
- **Common mistakes:** Trying to encode business rules via minOccurs.
- **Follow-up questions:** Example rule? (For tax invoice, GST registration number must be present when the indicator is Y; hypothetical.)

### Q67. Which leaves would be good candidates for `minOccurs="1"` and what would break?
- **Why the interviewer asks:** Contract tightening judgement.
- **Ideal answer:** Candidates: `reportData/dataClass`, `templateDetails/reportName` (routing) - but I cannot assert that from evidence. Making a leaf required breaks any sender or sample that omits it, including the default transaction (no `transactionDetails`) and the existing minimal shell test. It requires coordination and a version bump; I would keep all optional until the consumer states required fields [CONFIRM: with the report team].
- **Common mistakes:** Making many leaves required in one release.
- **Follow-up questions:** How to roll out? (Warn first, then enforce.)

### Q68. Should Y/N flags become enumerations?
- **Why the interviewer asks:** Schema quality.
- **Ideal answer:** Many flags are strings ("Y"/"N"). An `xs:simpleType` restriction with `xs:enumeration` would catch typos, but the schema has zero simpleTypes/enumerations today, and some flags may hold other values (for example "" or blank). Also pFirstPageNumber is a string because Jasper types vary. Adding enumerations is breaking for senders with different values; check data first.
- **Common mistakes:** Assuming flags are only Y/N.
- **Follow-up questions:** Where do the `show*` flags originate? (Request filter values and Mongo view columns.)

### Q69. How would you write a pattern type for `dd/MM/yyyy`?
- **Why the interviewer asks:** Concrete XSD authoring.
- **Ideal answer:** `<xs:simpleType name="ddMMyyyyType"><xs:restriction base="xs:string"><xs:pattern value="(0[1-9]|[12][0-9]|3[01])/(0[1-9]|1[0-2])/[0-9]{4}"/></xs:restriction></xs:simpleType>` and use it for date leaves where Jasper wants display format; allow empty with `|` or `minLength`. A regex checks shape not calendar validity (31/02/2026 passes). It would be an XSD change plus a version bump and the consumer must agree.
- **Common mistakes:** Believing patterns validate real dates.
- **Follow-up questions:** Alternative? (Union of dateTime and the pattern type, `xs:union`.)

### Q70. `xs:date` versus `xs:dateTime`: which would you choose for report dates?
- **Why the interviewer asks:** Semantics.
- **Ideal answer:** Most report dates are calendar dates (policy effective date) but sources are Timestamps with time 00:00:00; `xs:dateTime` preserves what Jasper had. `xs:date` communicates intent better and removes fake times, but is a breaking type change. It does not solve the `dd/MM/yyyy` format issue either. Keep dateTime unless the consumer requests a change.
- **Common mistakes:** Claiming `xs:date` accepts `dd/MM/yyyy`.
- **Follow-up questions:** Time zone? (dateTime without zone is local-ambiguous.)

### Q71. How would you use `nillable="true"` and `xsi:nil` here?
- **Why the interviewer asks:** Alternative to omission.
- **Ideal answer:** Declare decimal/dateTime/integer leaves `nillable="true"`; the XML writes `<slNo xsi:nil="true"/>`. Requires the `xsi` namespace on the root, which conflicts with the "no namespace" model (you can add `xmlns:xsi` without a target namespace). Consumers must understand nil. It adds noise; omission with `NON_NULL` is simpler. The schema currently has no `nillable`.
- **Common mistakes:** Forgetting to declare the `xsi` namespace.
- **Follow-up questions:** Impact on Jackson? (A custom serializer to write the attribute.)

### Q72. How is the XSD versioned and what is missing?
- **Why the interviewer asks:** Governance.
- **Ideal answer:** A version in the file header comments (v0.1.3 ... v0.1.6, dates, change lists) and the folder name `Plines LOB v0.1.6`. No `version` attribute on `xs:schema`, no tags in the repo, and the repo copy sits under `src/test/resources`. Java comments refer to v0.1.7-v0.1.10 that were not found. Recommendation: publish each version in one controlled place, keep Java and XSD versions in one table, and make the copy used in tests the same file as production.
- **Common mistakes:** Assuming the test copy is the latest.
- **Follow-up questions:** How did the commit history treat the XSD? (`git log --all -- '*.xsd'` shows only 3 commits: d4575fa5b 2026-08-03, 6961abbf9 2026-08-08, ea782121d 2026-09-28.)

### Q73. What is the governance gap between Java and XSD?
- **Why the interviewer asks:** Process maturity.
- **Ideal answer:** The repo added elements ahead of the published XSD (`mainReportName`, `subReportName`, `pCreatedDateFrom/To`), citing versions v0.1.9/v0.1.10, while the only available schema is v0.1.6; another branch edited the repo XSD without bumping the header. So there is no single source of truth. Fix: a named schema owner, a change request path, an agreed release of XSD and Java together, and a CI check. [CONFIRM: who owns the schema.]
- **Common mistakes:** Blaming one team.
- **Follow-up questions:** What would you ask first? (Where is the latest XSD and who approved the new elements?)

### Q74. Java comments cite v0.1.7, v0.1.9, v0.1.10. How do you speak about these?
- **Why the interviewer asks:** Handling unknowns.
- **Ideal answer:** Say: comments in the Java code refer to schema versions (counts in HEAD: "Pline.xsd v0.1.6" about 228 times, v0.1.7 twice, v0.1.10 twice, v0.1.4 once, "XSD v0.1.9" in `PlineReportParameters`), but I only have v0.1.6 files, so I cannot confirm the content of later versions. I would ask the schema owner. Never describe v0.1.7-v0.1.10 contents as fact.
- **Common mistakes:** Inventing the changes.
- **Follow-up questions:** Where did you search? (The reference folder, repo history, two remote branches, Documents folder.)

### Q75. What is the Pline master mapping workbook?
- **Why the interviewer asks:** Requirements trace.
- **Ideal answer:** `Pline_MASTER_MAPPING_SHEET_v0.1 (final1) 1 5.xlsx` ("Generated 2026-07-06", v0.1) with tabs for report families (EndtDocument, RejectRisk, PolicyDocument, AutoRenewal, CardPrinting, etc.) and a `_ChangeLog`; plus the per-report `RejectRisk_XSD_Mapping_sheet.xlsx`. The RejectRisk sheet was read fully and agrees with Jasper; the master workbook was only skimmed (sheet list and intro), so I cannot claim full cross-checks for the other tabs.
- **Common mistakes:** Claiming all tabs were verified.
- **Follow-up questions:** Which workbook is for your RejectRisk? (The RejectRisk one.)

### Q76. Describe, end to end, adding a new `transDet` leaf.
- **Why the interviewer asks:** Practical, repeatable process.
- **Ideal answer:** (1) Confirm the Jasper field and the mapping sheet row. (2) Add the leaf to `Pline.xsd` in the correct alphabetical slot with the right type, optional. (3) Add the field to `PlineTransDet` in the same relative position with the same type and exact case (`@JacksonXmlProperty` if needed). (4) Populate it in the service (DTO alias -> setter). (5) Update the header changelog and version. (6) Add unit tests (setter mapping; serialised element name/order; type). (7) Update the mapping sheet. (8) Notify the consumer, since strict old schemas reject unknown elements.
- **Common mistakes:** Forgetting position and case.
- **Follow-up questions:** For a rename? (Treat as remove + add; breaking.)

### Q77. How do you rename an element safely?
- **Why the interviewer asks:** Compatibility.
- **Ideal answer:** A rename is breaking for readers of the old name. Safe path: add the new element (additive), emit both for a transition period, update the consumer, then remove the old one in a later version. v0.1.4 handled renames with shadow fields to emit only the new names, which was a hard cut. Announce in the changelog.
- **Common mistakes:** Renaming in one step.
- **Follow-up questions:** Cost of dual emit? (Larger XML; duplicate fields.)

### Q78. How do you remove an element safely?
- **Why the interviewer asks:** Change control.
- **Ideal answer:** Stop populating it first (it stays in the XSD optional), confirm no consumer reads it (print engine templates), then remove from the XSD and Java in a version bump. v0.1.4 removed `showCoverNote`, `showTaxInvoice`, `prem`, `totalpremium` from reportParameters/reportData in favour of other locations - breaking. Check Jasper templates for references.
- **Common mistakes:** Removing because "unused in Java".
- **Follow-up questions:** How to find consumer references? (Search JRXML for the field name.)

### Q79. How do you move an element safely?
- **Why the interviewer asks:** Structural change handling.
- **Ideal answer:** Moves (v0.1.3 `prntDetails`, v0.1.4 `permanentDisablement`) change the path, which breaks XPath-based consumers and strict validators. Safe path: emit at both locations temporarily (if the schema allows both), migrate consumers, then remove the old location. Order within sequences also matters: moves need the sequence position updated in Java.
- **Common mistakes:** Updating the XSD only.
- **Follow-up questions:** What does "move only" in the header mean? (No content change.)

### Q80. Classify these changes as safe or breaking: add optional leaf at end; add in the middle; change type; reorder; case change.
- **Why the interviewer asks:** Summarised rule set.
- **Ideal answer:** Add optional leaf at end/alphabetical slot: safe for name-based readers; a strict validator on the OLD schema would reject the new element, a validator on the NEW schema accepts old files. Add in the middle: fine only if Java and XSD agree on the position. Change type (string to dateTime in v0.1.6): breaking for senders. Reorder: breaking for validators, harmless for name-based readers. Case change: breaking, XML is case-sensitive. Make optional required: breaking. Remove/rename/move: breaking.
- **Common mistakes:** "Adding is always safe."
- **Follow-up questions:** Which of these have happened here? (All of them across v0.1.3-v0.1.6 except making required.)

### Q81. Backward versus forward compatibility in schema terms.
- **Why the interviewer asks:** Terminology precision.
- **Ideal answer:** Backward compatible: new schema accepts documents valid under the old schema (adding optional elements). Forward compatible: old schema accepts documents produced under the new schema (needs wildcards `xs:any`, which Pline.xsd does not use). So additive changes are backward but not forward compatible with strict validators. Deploy consumers first when adding fields.
- **Common mistakes:** Swapping the terms.
- **Follow-up questions:** How could the schema be made forward compatible? (`xs:any processContents="lax"` extension points; costs strictness.)

### Q82. What would you say about the print engine's relationship to the XSD?
- **Why the interviewer asks:** Honesty about unknowns.
- **Ideal answer:** It is not evidenced anywhere I read whether the print engine validates against `Pline.xsd` or merely reads elements by name. If it only reads by name, order and empty elements might be harmless while path/name/format changes are not; if it validates, the current output is broken. Ask the report-side owners; do not guess [CONFIRM: engine behaviour].
- **Common mistakes:** Asserting either way.
- **Follow-up questions:** How to find out? (Ask for the engine's validation settings or test by sending an invalid document to dev.)

### Q83. What is the effect of `elementFormDefault="qualified"` once a namespace is added?
- **Why the interviewer asks:** Forward-looking schema knowledge.
- **Ideal answer:** All locally declared elements must be namespace-qualified in instances. Today there is no namespace, so it has no effect. If a `targetNamespace` is added, the XML needs `xmlns` and every element is in it (or prefixed). Jackson would need `@JacksonXmlRootElement(namespace=...)` and property namespaces; Common.xsd would need the same namespace or an `import`.
- **Common mistakes:** Saying "qualified" already puts elements in a namespace.
- **Follow-up questions:** Is it likely? (Not planned in evidence.)

### Q84. What does the XSD say about `mailingDetails`?
- **Why the interviewer asks:** Common type knowledge.
- **Ideal answer:** [XSD] `mailingDetails` (0..1) of `mailingDetailsType` with `mailingAddressType` (address with `addressline1/2`, `email`...). The controller fills `name`, `email`, `senderMail` from metadata (`populateMailingDetails`). In the KT sample the `address` has `addressline1`, `addressline2` and `email`. Note: the Java placement is before `transactionDetails` but after `templateDetails`, which conflicts with the XSD order (reportParameters, mailingDetails, templateDetails, transactionDetails).
- **Common mistakes:** Treating mailing as part of Plines types.
- **Follow-up questions:** Does `senderMail` exist in the XSD? [CONFIRM by reading `mailingDetailsType`.]

### Q85. Explain the counts "Template12 351 distinct setter names, 213 equal a transDet field".
- **Why the interviewer asks:** Reasoning about a rough metric.
- **Ideal answer:** The 04 guide matched setter names in Template12 service files against the 634 transDet leaf names: 351 distinct setter names, 213 equal a leaf (over-estimate because other classes such as cover or risk share names). For ASP: 274 / 161; 173 setter names are common to both. It is a name-matching heuristic, not execution. Use it as "scale", not as a coverage claim.
- **Common mistakes:** Quoting 213 as the number of populated leaves.
- **Follow-up questions:** Better method? (Serialise a populated result and count non-empty leaves.)

### Q86. How would you measure which `transDet` leaves a report actually fills?
- **Why the interviewer asks:** Practical coverage method.
- **Ideal answer:** Create a populated sample through the service with mocked handler data, serialise, then parse and list non-empty leaves; compare with the mapping sheet's expected list for that report. Missing ones show gaps against Jasper. Do for Template12, ASP and RejectRisk. This replaces the setter-name heuristic.
- **Common mistakes:** Counting setters.
- **Follow-up questions:** Where does the expected list come from? (Mapping workbook / JRXML fields.)

### Q87. Why does Template12 build `PlineTransDet` 57 times?
- **Why the interviewer asks:** Structure of the service.
- **Ideal answer:** Per the constructor count in the audit, T12 base service builds 57 `PlineTransDet`, 15 `PlineRisk`, 15 `PlineCover`, 5 `PlineStaxPcts`, 4 `PlineCompanyDesc`; each sub-report block (schedule, benefit, nominee, debit note, tax invoice...) creates its own `transDet`/`reportData`. ASP builds `PlineTransDet` 20 times. These are counts of constructor calls in code, not runtime instances.
- **Common mistakes:** Treating the numbers as runtime counts.
- **Follow-up questions:** Why not one transDet? (Different sub-reports map to different Jasper templates.)

### Q88. How do `mainReportName`/`subReportName` map to the original Jasper reports?
- **Why the interviewer asks:** Purpose of the newest fields.
- **Ideal answer:** The Jasper dump has a main report and sub-reports per module (RejectRisk hierarchy: main `RejectRisk`, sub `ReportHeader`). The names tie each `reportData` to the right JRXML so the print engine can pick the template per block. For Template12/ASP, one request produces many blocks (cover note, schedule, complaint, debit note, tax invoice...), so the tag is needed. The exact naming convention for the other modules is [CONFIRM: with the code and report team].
- **Common mistakes:** Assuming they are already agreed in an XSD.
- **Follow-up questions:** Where is `PlineReportNames` used? (Every Plines service via `tag(transactions, getClass())`.)

### Q89. How do you check XSD counts with tools and what are the pitfalls?
- **Why the interviewer asks:** Verification method.
- **Ideal answer:** `grep -c '<xs:element'`, `grep -c 'type="xs:decimal"'`, `grep -c 'minOccurs="0"'`, awk ranges between type anchors; confirm with a JDK validator or xmllint. Pitfalls: counts include comments, multi-line declarations, `ref=` elements and inline types, and one shared element name appears in many containers. The earlier figures (1065 declarations, 1062 optional, 22 + 66 named types) come from such greps.
- **Common mistakes:** Mixing counts of declarations and counts of distinct names.
- **Follow-up questions:** Better tool? (Parse with an XML library and walk the schema.)

### Q90. What did the hand-written RejectRisk XML experiment prove?
- **Why the interviewer asks:** Evidence interpretation.
- **Ideal answer:** [RAN] A correctly ordered RejectRisk document with ISO dates and no empty numeric elements validated with 0 errors, proving the schema itself is workable for this report. A variant with `mainReportName`/`subReportName`, `06/06/2017` for createdDate and `reportParameters` after `transactionDetails` gave 5 errors, one per category. It does not prove the real service output, which was not generated by running the application.
- **Common mistakes:** Saying "RejectRisk validates".
- **Follow-up questions:** What would prove it? (The real service test in Q60.)

### Q91. What do you say when asked "does your output validate against the XSD?"
- **Why the interviewer asks:** The most likely trap question.
- **Ideal answer:** "No, not end to end, and I can say why. We aligned payload types and structure to the XSD (field names/types match; 12 Java-only fields), but validation of real service output shows order, date format, empty numeric elements and unknown elements (`mainReportName`, `subReportName`, `pCreatedDateFrom/To`, `pReportTitle`) as violations against v0.1.6. The existing test validates a minimal hand-written shell. The next step is a real output test and agreeing the newer schema." Say what you personally did [CONFIRM].
- **Common mistakes:** Overclaiming.
- **Follow-up questions:** Which fix first? (Ordering and null handling; they remove most errors.)

### Q92. What are the highest-volume and highest-effort fixes to make real output validate?
- **Why the interviewer asks:** Prioritisation.
- **Ideal answer:** Highest volume: empty numeric/date elements (use NON_NULL for the Plines mapper). Medium effort: ISO date serialisation (central serializer). Structure: `@JsonPropertyOrder` on `Transaction`/Plines subclasses, `TemplateDetails` order, address-first types, cover/risk/transDet order. Contract: publish the newer XSD or drop unknown elements. Each can be done and tested separately with the ratchet.
- **Common mistakes:** Editing shared classes first.
- **Follow-up questions:** Which breaks other LOBs? (Shared `Transaction`, `TemplateDetails`.)

### Q93. How do `reportData` leaves like `slNo`, `status`, `tat`, `requestedby` relate to the three modules?
- **Why the interviewer asks:** Knowing which fields are in use.
- **Ideal answer:** [XSD] `reportData` has slNo (decimal), dataClass, permanentDisablement (decimal), Policyletterind, policystatus, requestedby, status, tat before `endorsementDetails` and `prntDetails`. They come from listing-style Plines reports (status/TAT reports). RejectRisk fills `dataClass`; Template12/ASP fill `dataClass`, `policystatus`, `status` (per the audit notes). Others stay empty (e.g. `<slNo/>`, invalid when empty).
- **Common mistakes:** Assuming all are used.
- **Follow-up questions:** What is `permanentDisablement` doing in `reportData`? (v0.1.4 requirement.)

### Q94. Why is `reportParameters` first in the XSD but last in Java, and who decides?
- **Why the interviewer asks:** Tying a design reason to the mismatch.
- **Ideal answer:** The XSD (generated from the Bond-style envelope) puts `reportParameters` first; the shared motor `Transaction` class declares `templateDetails`, `mailingDetails`, `transactionDetails`, `reportParameters` (historic order from earlier LOBs). Nobody validated order before. The decision belongs to the schema owner; the safe Java fix is a Plines subclass with `@JsonPropertyOrder`.
- **Common mistakes:** Claiming Jackson sorts alphabetically.
- **Follow-up questions:** Does Motor's own XSD agree with its Java? (Unknown.)

### Q95. How would you teach a new joiner to read `Pline.xsd` quickly?
- **Why the interviewer asks:** Communication.
- **Ideal answer:** (1) Start at `transactions` and walk down to `transDet` (Q1). (2) Learn the three rules: everything optional, everything ordered, dates are dateTime. (3) Find a leaf in the mapping sheet, then in the XSD, then in `PlineTransDet`, then in the service setter. (4) Learn the case traps. (5) Use the Jasper JRXML to see the intent. (6) Run a validator on a sample. Provide the one-page tree.
- **Common mistakes:** Starting with the 1400-line file top to bottom.
- **Follow-up questions:** Tools? (IDE XSD view, xmllint.)

### Q96. Explain `Common.xsd` `addressType` (19 leaves) and why it matters.
- **Why the interviewer asks:** Reuse detail.
- **Ideal answer:** [XSD] `addressType` has addr, address1-4, city, country, mobileNo, mstPol* (8 leaves), postCode, state. Used by agent/branch/comp/taxClient/client address children and by `agencyInfoType` (wrapper). Java `PlineClientInfo` keeps mstPol* fields in a nested class. Any change here affects all LOBs, so Plines avoids editing it.
- **Common mistakes:** Putting Pline-only address fields in Common.
- **Follow-up questions:** Where would Pline-only address fields go? (In `pline*` types.)

### Q97. What risks does the `@JsonIncludeProperties` whitelist introduce?
- **Why the interviewer asks:** Maintenance risk.
- **Ideal answer:** A new XSD leaf requires adding its name to the whitelist; forgetting it silently drops the value. A setter for a non-whitelisted property is silently ignored (see the dropped list). The whitelist has 73 names vs 67 XSD leaves (the 6 extras). A test that compares whitelist names with XSD leaves (plus a documented allow-list) would catch both directions.
- **Common mistakes:** Debugging the service when the whitelist is the cause.
- **Follow-up questions:** Alternative? (Separate DTO class without base inheritance.)

### Q98. How would you present the XSD risks to your team lead?
- **Why the interviewer asks:** Communication and prioritisation.
- **Ideal answer:** Three bullets with evidence: (1) the published schema v0.1.6 and Java output disagree on dates, order, null elements and four groups of unknown elements; (2) the only test validates a shell, so CI cannot see this; (3) the newest schema is unknown. Ask for: schema owner decision, a ratchet test, and an agreed ISO date policy. Offer a small first PR: shared Plines mapper with NON_NULL plus a RejectRisk validation test.
- **Common mistakes:** Presenting only problems with no plan.
- **Follow-up questions:** What if the team says the engine does not validate? (Then document that and downgrade priority but keep path/name/format checks.)

### Q99. Which XSD facts are certain and which are uncertain? Give both lists.
- **Why the interviewer asks:** Intellectual honesty test.
- **Ideal answer:** Certain: v0.1.6 content and counts; Java classes' field lists; that real output sample (older) failed validation; that the minimal shell and a hand-ordered document validate. Uncertain: the content of v0.1.7-v0.1.10; whether the print engine validates; current-branch error counts (not executed); ASP GST/SST label drop; the Template12/ASP leaf coverage (heuristic); behaviour of untested containers under Xerces; whether `buyerInfo`/`supplierInfo` are ever populated.
- **Common mistakes:** Listing only certainties.
- **Follow-up questions:** How to reduce uncertainty? (Run the service with sample data and validate; ask the schema owner.)

### Q100. If you had two days, how would you make progress on XSD alignment?
- **Why the interviewer asks:** Execution planning.
- **Ideal answer:** Day 1: build the RejectRisk service-output validation test with an error-collecting handler and a baseline; list errors by category. Introduce a Plines mapper with NON_NULL and ISO dates behind a flag for the test only, to measure the effect. Day 2: add `@JsonPropertyOrder` to the Plines subclasses and the shared `Transaction` alternative; produce a short decision document for the schema owner on `mainReportName`/`subReportName`, `pCreatedDate*`, `pReportTitle`. Do not change shared Motor classes without a regression run. Keep every change small and reversible.
- **Common mistakes:** Trying to fix all 1000+ errors in one PR.
- **Follow-up questions:** What would you not do? (Rewrite the XSD or generate classes.)
