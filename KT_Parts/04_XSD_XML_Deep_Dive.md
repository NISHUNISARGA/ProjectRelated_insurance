# 04 - XSD / XML Deep Dive (Pline.xsd + Common.xsd v0.1.6)

Scope of the developer's Jira item: "XSD changes for all three reports (PLine Template12, ASP, RejectRisk)".
Everything below is evidence-based. Each fact is tagged by how it was obtained:

- [XSD] read directly from the schema files, counts by grep/awk.
- [JAVA] read from the repo (`opus-documentprintdata-service`, branch `feature/retrofit-develop-updated`, HEAD 861f0666c).
- [RAN] I actually ran a JDK `javax.xml.validation` validator (scratch copy in the Windows temp folder, nothing written to the repo) against the XSD.
- [DERIVED] inferred from code, not executed.
- [UNVERIFIED] stated as an open point.

Source locations
- Reference XSDs: `C:\Users\EPGRVFY\Documents\PLINE LOB v0.1.6_New 1\PLINE LOB v0.1.6_New\Plines LOB v0.1.6\Plines LOB v0.1.6\Pline.xsd` (1400 lines) and `Common.xsd` (748 lines).
- Repo copies: `src/test/resources/xsd/plines/Pline.xsd` and `Common.xsd`. Byte-compare ignoring CRLF: identical to the reference (re-confirmed; md5 of CR-stripped Pline.xsd `55255bec56...` is the same in 5 working copies under Documents).
- Mapping workbooks found: `C:\Users\EPGRVFY\Documents\Pline_MASTER_MAPPING_SHEET_v0.1 (final1) 1 5.xlsx` (tabs: _ChangeLog, EndtDocument, ScrutinyRejection, Auto_Policy, CardPrinting, CIUnderWritingErrors, MyegStatus, AutoRenewal, ForeignWorker, CIUploadPendingConfirmed, RejectRisk, PolicyDocument, CharityReport, EndorsementStatusReport, PolicyEndtEncodedNotPrinted, CIOutsourcePrinting, HealthInsurancePremiumStatement, ScrutinyRejectionSummary, CancellationPolicy, ScrutinyRejectedChecklist, UnderwritingErrors, CIUploadPolicyGenerated; "Generated 2026-07-06", v0.1) and `C:\Users\EPGRVFY\Documents\Plines_Base_code\T12_final_code\RejectRisk\Mapping_sheet` / `...\RejectRisk\RejectRisk\RejectRisk_XSD_Mapping_sheet.xlsx`.
- Real RejectRisk service responses saved by the developer: `...\T12_final_code\RejectRisk\RejectRisk\1357766.xml`, `2439275.xml` (+ `API_Crul_*.txt` curl requests). These are dated 1 Sep 2026 and come from an OLDER working copy (T12_final_code), not the current branch (they lack mainReportName, show `policyletterind` lower-case, etc.). Treat as "historical real output".

---------------------------------------------------------------------

## 1. Anatomy of Pline.xsd and Common.xsd

### 1.1 Basics [XSD]
| Item | Pline.xsd | Common.xsd |
|---|---|---|
| Root element | `transactions` (global), which contains 1..unbounded `transaction` (global element, referenced with `ref`) | NONE (types only; header says "No root element") |
| targetNamespace | NONE (no-namespace schema) | NONE |
| elementFormDefault | `qualified` | `qualified` |
| Includes | `<xs:include schemaLocation="Common.xsd"/>` (include, not import: possible only because both are no-namespace) | - |
| Header comment says | "follows the BOND structure"; Common included "UNCHANGED (byte-for-byte)" | "Common_Motor.xsd - Shared types common to all LOBs (Motor, Fire, Marine, Bond)" |
| Named complexTypes | 22 | 66 |
| Total `xs:element` declarations | 1065 | 404 |
| `minOccurs="0"` | 1062 | 404 |
| `maxOccurs="unbounded"` | 15 | 27 |
| simpleTypes / enumerations / patterns / restrictions / attributes / xs:all / xs:choice / key / extension | 0 of each | 0 of each |

Consequences
- No namespace means the XML sent to the print engine must have NO `xmlns`; `<transactions>` is plain. That is what the service emits.
- Because `elementFormDefault="qualified"` with no target namespace, there is nothing to qualify; it is harmless.
- Every one of the 1062 local elements in Pline.xsd is optional. The only things that are not optional are the three non-`minOccurs=0` declarations: global `transactions`, global `transaction`, and the `ref="transaction" minOccurs="1"`. So: leaves with `minOccurs=0` = 1062, required leaves = 0 (grep: leaves with `type="xs:..."` and no `minOccurs="0"` = 0 in both files). In practice the schema constrains ORDER and TYPE, not presence.
- It is entirely `xs:sequence`; there is no `xs:all` or `xs:choice` anywhere. Hence ORDER of sibling elements is strictly enforced (this is the single biggest source of validation failure, see section 5).

### 1.2 Scalar type usage [XSD, grep counts]
| Type | Pline.xsd | Common.xsd |
|---|---|---|
| xs:string | 546 | 250 |
| xs:decimal | 387 | 79 |
| xs:dateTime | 64 | 11 |
| xs:integer | 8 | 15 |
| xs:boolean | 2 (nomineeInd, printClauseWordingInd) | 4 |
| xs:date / xs:int | 0 | 0 |

Per container (leaf counts, from awk):
- `reportParameters`: 67 leaves = 43 string, 17 decimal, 6 dateTime, 1 integer.
- `endorsementDetails`: 50 leaves = 32 string, 8 decimal, 9 dateTime, 1 integer.
- `transDet`: 634 scalar leaves = 311 string, 283 decimal, 38 dateTime, 2 integer, plus 12 structured list children (coverList, provisoList, excessList, staxList, sobList, nomineeList, diagnosisList, csLimitList, memoList, specificationList, premiumProjectionList, riskList).
- `risk`: 28 scalar leaves (15 string, 8 decimal, 3 dateTime, 2 integer) + coverList, provisoList, excessList.

Important: there is no `xs:date`. Every date is `xs:dateTime`, whose lexical form is `YYYY-MM-DDThh:mm:ss`. The Java services write dates as `dd/MM/yyyy` (see 5.3). Also an empty element (`<x/>`) is NOT a valid decimal/dateTime/integer/boolean (it is valid only for xs:string); nothing in the XSD is `nillable`.

### 1.3 Element hierarchy down to transDet [XSD, lines 464-1400]
```
transactions                                   (1)
  transaction                                  (1..unbounded)
    reportParameters            0..1   inline type, 67 leaves, alphabetical-ish (see 3)
    mailingDetails              0..1   mailingDetailsType   (Common)
    templateDetails             0..1   templateDetailsType  (Common)
    transactionDetails          0..1
      headerTransaction         0..unbounded
        branchcodeH, branchnameH, agentcodeH, agentnameH            (string)
        effectivedatefrom, effectivedateto, endtdatefrom, endtdateto (dateTime)
        description, distchndescp, subchndescp, regdescp, dataclassH (string)
        reportTransaction       0..1
          reportData            0..unbounded
            slNo (decimal), dataClass, permanentDisablement (decimal),
            Policyletterind, policystatus, requestedby, status, tat
            endorsementDetails  0..1  inline, 50 leaves
            prntDetails         0..unbounded
              agentInfo (plineAgentInfoType), agencyInfo (agencyInfoType),
              branchInfo (plineBranchInfoType), clientInfo (clientInfoType - Common),
              compInfo (plineCompInfoType), customerServiceInfo (Common),
              companyType (companyTypeType), taxClientInfo (plineTaxClientInfoType),
              buyerInfo, supplierInfo, issueBy (string), paramGroup (0..unbounded)
              transDetList      0..1
                transDet        0..unbounded   634 scalar leaves + 12 lists (below)
                  coverList, provisoList, excessList, staxList, sobList, nomineeList,
                  diagnosisList, csLimitList, memoList, specificationList,
                  premiumProjectionList,
                  riskList > provisoList, risk(0..unbounded) > 28 leaves, coverList, provisoList, excessList
                companyType     0..1   (sibling of transDet, last child of transDetList)
      clientType, documentNumber, isfirstTransactionDtl, islastTransactionDtl,
      printType, user, checklistCopy                                  (all string; AFTER headerTransaction)
```
Notes
- `companyType` exists at TWO places: `prntDetails/companyType` and `transDetList/companyType`. RejectRisk uses the second one (REMARKS -> companyDesc/remarks).
- `prntDetails` is the LAST child of `reportData` (v0.1.3 fix). `permanentDisablement` and `dataClass` are ELEMENTS of reportData even though the header text says "attribute"; there are zero `xs:attribute` in the schema.
- `reportData` has NO `mainReportName` or `subReportName` in v0.1.6 (grep: zero hits for either; the only `reportName` is `templateDetails/reportName` in Common.xsd).
- `transactionDetails` tail leaves (`clientType ... checklistCopy`) come AFTER `headerTransaction`. The Java `PlineTransactionDetails` honours this via `@JsonPropertyOrder`.

### 1.4 Named complexTypes
Pline.xsd (22) - pline-specific "extension types" = Common body + Pline additions, in the Bond style:
| Type | Purpose |
|---|---|
| PlineStaxPctsType / PlineStaxListType | service-tax lines (Common staxPcts body + 12 Pline fields) used by `staxList` |
| plineAgentInfoType (17 leaves) | agentInfo: address + agent fields + agentCatType, agentCodeName, bancaAgent, totalAgentCode |
| plineBranchInfoType (8) | branchInfo + brAddress, branchPrefix, issuedBranch |
| plineCompInfoType (12) | compInfo (company GST/SST reg no, companyCode, companyDescp...) |
| plineTaxClientInfoType (12) | taxClientInfo + class, commnAmt, stampDuty, taxAmtPrint... |
| plineProvisoType / plineProvisoListType (15) | provisos / clauses |
| plineBenefitType (22) / plineBenefitsType (5) / plineSobListType | Schedule Of Benefits: sobList > benefits > benefit |
| plineDiagnosisType (6) / plineDiagnosisListType | diagnosis rows |
| plineNomineeType (13) / plineNomineeListType | nominee rows |
| plineCoverType (44) / plineCoverListType | covers; contain excessList and provisoList |
| buyerInfoType (11) / supplierInfoType (11) | co-inward tax-invoice parties |
| agencyInfoType (1) | wrapper around addressType |
| companyTypeType / companyDescType (10) | company header/footer text blocks (addr1-3, companyName, companyRegNo, phone, remarks, showGstPct, signature, tinNo) |

Common.xsd (66) key ones: mailingDetailsType, mailingAddressType, templateDetailsType (14 leaves: batchConfigId, boResId, channel, copyTypeList, dataClass, disallowCopy, docType, linkLocation, outputFormat, outputPath, reportName, sequence, templateId, templateName), copyTypeListType, agentInfoType, addressType (19 leaves: addr, address1-4, city, country, mobileNo, mstPol* x8, postCode, state), branchInfoType, clientInfoType (23), mstpoladdressType, compInfoType, customerServiceInfoType (address, email, tollFreeNo), customerAddressType, taxClientInfoType, paramGroupType (dataType, paramCode, paramValue), memoType (memoCode, memoDesc, memoNarr), excessType, csLimitType, specificationType, premiumProjectionType, diagnosisType, nomineeType, provisoType, coverType, benefitType, sponsorType, staxListType, etc. Many Common types (motor/fire/marine: extraCover*, floatingRisk, ncd, questionnaire, recommendedPerils...) are NOT used by Plines at all.

### 1.5 Sequence-vs-all rules
- All containers are `xs:sequence`; children must appear in the declared order; optional ones can be skipped, but those present must keep relative order. A list element (maxOccurs unbounded) must have all its repetitions contiguous.
- The XSD is generated alphabetically (case-insensitive) within a block, then "Pline additions" appended after the Common body. That is why `address` is first in `plineAgentInfoType` but the Java field is last.
- Case matters: `Policyletterind` (capital P), `Columncount`, `Emailbcc`, `LOADING` and `loading` coexist, `PREM`, `sumTotalPremiumMEDIX`, `TableHeaderCOUNT`.

---------------------------------------------------------------------

## 2. Changelog v0.1.3 to v0.1.6 (from the Pline.xsd header)

| Version / date | What changed | Stated reason | Impact on a consumer | Backward compatibility |
|---|---|---|---|---|
| v0.1.3 (2026-07-14) | `prntDetails` moved from sibling of `reportData` (under reportTransaction) to LAST child inside `reportData`, sibling of `endorsementDetails`. "Move only". | Did not match the Bond reference structure. | Path changes: `reportTransaction/prntDetails` becomes `reportTransaction/reportData/prntDetails`. | BREAKING (structural). Any consumer/XPath on old path stops matching. |
| v0.1.4 (2026-07-20) | (a) `permanentDisablement` (decimal) moved from reportParameters to reportData (after dataClass). (b) Added to reportParameters (xs:string): pMstPolHolder, pCertInd, pcopyType, pshowPaymentInfo, pshowEinvEmail, pchecklistCopy, ppidmprint, pshowCoverNote, pshowTaxInvoice, pPrintSeries, pFirstPageNumber, pProductCode. (c) De-duplicated: originals `showCoverNote`, `showTaxInvoice` REMOVED in favour of `pshowCoverNote`, `pshowTaxInvoice`. (d) `companionName` kept only as transDet leaf (compInfo/companionName removed). (e) `totalpremium` only as transDet leaf (reportData/totalpremium removed). (f) `class` added under taxClientInfo (maps report CLASS). (g) `PREM` only as transDet leaf (reportParameters/prem removed); premium is literally PREM in Jasper. | Each addition "traced to a real Jasper report parameter/variable"; "per updated Plines requirement"; remove duplicates so each Jasper variable appears once. pFirstPageNumber is string because Jasper type varies Integer/String. | Consumers reading `permanentDisablement`, `showCoverNote`, `showTaxInvoice`, `totalpremium`, `prem` at the old location get nothing. | BREAKING for moved/removed elements (a, c, d, e, g); additive (safe) for b, f. |
| v0.1.5 (2026-07-24) | reportParameters (all string): pAgentEmailAdd, pAgentName, pAgentPhoneCode, pAgentPhoneNo, pAtceInd, pBeforeEffDate, pBeforeOneMnth, pPymtMode. transDet: LOADING (decimal), Sum variables (sumInMCO, sumInPremium, sumLoading, sumMCOCover, sumOptDMA, sumOptIMA, sumOutMCO, sumOutPremium, sumPremiumCover, sumTotalPremium, sumTotalPremium_MEDIX), COLUMN_COUNT, TableHeader_COUNT (Jasper built-in count variables), EMAIL_BCC (string, from SharedReport, added on "explicit request" although outside Plines scope). | Traced to AutoRenewal sub reports, *EndtSchedule, ServiceTaxInvoice, DebitNote; premium-particulars group sums. | New leaves only. NOTE: element names in the XSD are `Columncount`, `TableHeaderCOUNT`, `Emailbcc`, `sumTotalPremiumMEDIX` (not the header spellings). | Backward compatible (additive, no structural change). Case trap: `LOADING` vs existing `loading`; XML is case sensitive. |
| v0.1.6 (2026-07-28) | (a) Added `pAgentChannel` (string) to reportParameters in alphabetical position. (b) TYPE CHANGE `pBeforeEffDate` xs:string -> xs:dateTime. (c) Date sweep: llb25Ind, pBeforeOneMnth, pAgentChannel stay string. (d) NOT added (duplicates): pAtceInd (already v0.1.5), llb25Ind (already a transDet leaf). | AutoRenewal/RenewalNotificationLetter feeds AGCHANNEL; `pBeforeEffDate` is populated as `new SimpleDateFormat("dd/MM/yyyy").format($F{BR_EFF_DATE})` (BR_EFF_DATE is java.sql.Timestamp), "consistent with every other date leaf". | (a) new optional leaf. (b) Narrows what is valid: `28/07/2026` and empty string are no longer valid for that leaf. | (a) compatible. (b) NOT compatible with senders of free-form strings; the Java still sends dd/MM/yyyy, so it does not satisfy the new type (see 5.3). |

Header inaccuracies worth knowing (do not repeat them in an interview):
1. "attribute inside reportData" (v0.1.4) - they are elements.
2. v0.1.6 says "no structural changes", and v0.1.5 "no structural change" - correct for those, but v0.1.4 contains structural removals even though it says "no structural changes (elements added/removed only as sequence leaves)".
3. Header leaf names with underscores (COLUMN_COUNT, sumTotalPremium_MEDIX, EMAIL_BCC) are Jasper names; the real XSD element names differ (above).
4. Branch `origin/OPUSM-44159-retrofit-develop-pline-jrxml-comparison-fixes` (commit ea782121d, 2026-09-28) edited the repo Pline.xsd to add transDet `leasingBranch` and reportData `premiumPrint`, `benefitPrint`, `nomineePrint` WITHOUT bumping the header version. That edit is NOT in the v0.1.6 reference and NOT in HEAD of the current branch.

### 2.1 Q&A on the changelog
- Q: Why was `prntDetails` moved inside `reportData` (v0.1.3)? A: The shared Bond/Common envelope puts `prntDetails` inside `reportData` (reportTransaction > reportData > prntDetails). The first Pline draft put it beside reportData by mistake. The Java was changed accordingly: `PlineReportData` owns `prntDetails`.
- Q: Why is `permanentDisablement` now on reportData and not reportParameters? A: Updated Plines requirement (v0.1.4) - it is per-row data, not a single report-level parameter. Java: field `permanentDisablement` is in `PlineReportData`.
- Q: Why were `showCoverNote` and `showTaxInvoice` removed? A: Duplicates of `pshowCoverNote` / `pshowTaxInvoice`; the Jasper parameter names are the p-prefixed ones. Cost: a breaking removal for anyone reading the old names. In Java the shadow-field approach in `PlineReportParameters` suppresses base-class names (`@JsonIgnoreProperties`) to emit only the XSD-correct names.
- Q: Why is `pFirstPageNumber` a string? A: Jasper declares it Integer in some templates and String in others; string is the lowest common denominator.
- Q: Why was `pBeforeEffDate` retyped to xs:dateTime? A: Because it holds a date (BR_EFF_DATE Timestamp) and every other date leaf is dateTime. Honest caveat: the Jasper side wants the dd/MM/yyyy string, and the Java sends dd/MM/yyyy; a strict dateTime validator will reject that. So the retype is semantically tidy but wire-incompatible until the service emits ISO-8601 (or the XSD goes back to string / a custom pattern type).
- Q: Why was `EMAIL_BCC` added though it is outside Plines? A: Explicit request, present only in SharedReport eMailNotificationPolicyReport. It is in the XSD as `Emailbcc`.
- Q: Why is there both `LOADING` and `loading`? A: Different Jasper sources: `LOADING` is the Jasper field name (BigDecimal), `loading` is the older Bond-derived camelCase envelope field. Java mirrors this with `loading` and `loadingUpper` (`@JacksonXmlProperty(localName="LOADING")`).
- Q: Why `PREM` and not `premium`? A: Plines Jasper field literally named PREM; `premium` is a Bond-derived envelope field and both exist in transDet.

---------------------------------------------------------------------

## 3. reportParameters element (67 leaves) [XSD + JAVA]

All optional, order is the order below.

| # | Leaf | Type | | # | Leaf | Type |
|---|---|---|---|---|---|---|
| 1 | dateValue | dateTime | | 35 | polIdNo | string |
| 2 | pAgentChannel | string | | 36 | polRef | string |
| 3 | pAgentEmailAdd | string | | 37 | postcodeDescp | string |
| 4 | pAgentName | string | | 38 | pPrintSeries | string |
| 5 | pAgentPhoneCode | string | | 39 | premDueAftPtv | decimal |
| 6 | pAgentPhoneNo | string | | 40 | premDueRoundedAftPtv | decimal |
| 7 | parentName | string | | 41 | premDueValue | decimal |
| 8 | pAtceInd | string | | 42 | premDueWthGst | decimal |
| 9 | paymentFreq | string | | 43 | premiumPrint | decimal |
| 10 | pBeforeEffDate | dateTime | | 44 | printingDate | dateTime |
| 11 | pBeforeOneMnth | string | | 45 | printPoip | decimal |
| 12 | pCertInd | string | | 46 | printTitle | string |
| 13 | pchecklistCopy | string | | 47 | prlInd | decimal |
| 14 | pcopyType | string | | 48 | pProductCode | string |
| 15 | perilsClausesCnt | decimal | | 49 | pPymtMode | string |
| 16 | perilscount | integer | | 50 | productCodeName | string |
| 17 | pFirstPageNumber | string | | 51 | promoAmt | decimal |
| 18 | phoneCode | string | | 52 | proposerName | string |
| 19 | planCount | decimal | | 53 | provInd | decimal |
| 20 | planDescp | string | | 54 | pShowExpiryLetter | string |
| 21 | planDescpMal | string | | 55 | pShowGstLabel | string |
| 22 | planType | string | | 56 | pshowCoverNote | string |
| 23 | pLaunchDate | dateTime | | 57 | pshowEinvEmail | string |
| 24 | pmtDueDate | dateTime | | 58 | pshowPaymentInfo | string |
| 25 | pMstPolHolder | string | | 59 | pShowSstLabel | string |
| 26 | ppidmprint | string | | 60 | pshowTaxInvoice | string |
| 27 | polContractId | decimal | | 61 | psi | decimal |
| 28 | polHolder | string | | 62 | pType | string |
| 29 | policyCount | decimal | | 63 | pwdInd | decimal |
| 30 | policyGeneratedDate | dateTime | | 64 | pymtFrq | string |
| 31 | policyHolder | string | | 65 | showComplaint | string |
| 32 | policyholderName | string | | 66 | showDebitNote | string |
| 33 | policyPromoAmt | decimal | | 67 | showScheduleStd | string |
| 34 | policySumInsured | decimal | | | | |

Totals: 43 string, 17 decimal, 6 dateTime (dateValue, pBeforeEffDate, pLaunchDate, pmtDueDate, policyGeneratedDate, printingDate), 1 integer (perilscount).

Java side: `PlineReportParameters extends motor ReportParameters`, with a `@JsonIncludeProperties` whitelist of 73 names [JAVA]. Comparison [RAN via comm]:
- XSD leaves not in whitelist: 0.
- Whitelist names NOT in the v0.1.6 XSD (6): `pCreatedDateFrom`, `pCreatedDateTo`, `pReportTitle`, `pshowEndtSchedule`, `pShowOutPrintCovLetter`, `pShowPDPANotice`. Source comments tag `pCreatedDateFrom/To` as "XSD v0.1.9" - a version that is not in the files I could find.
- The whitelist is how the shared Motor/Fire/Bond base class (many more fields) is prevented from leaking into Plines XML. Shadow fields + `@JsonIgnoreProperties` fix casing (ppidmprint, pPrintSeries, pshowEinvEmail, showComplaint, showDebitNote, showScheduleStd).

### 3.1 Who uses what (from the `rp.set...` calls) [DERIVED from setters + whitelist; not executed]
Setter-name to JSON-name mapping matters: a setter that writes a base-class property which is not in the whitelist is silently dropped from the XML (e.g. `setPcontractId` -> `pContractId`, `setPpolicyNo`, `setPversionNo`, `setPendtNo`, `setPrefNo`, `setPuserId`, `setPcompanyParamType`, `setPissueDate`, `setPsignIssueDate`, `setPshowGst/Sst/Tran`, `setReportName`, `setShowPaymentInfo` are all NOT in the Plines XML).

Template12 (`Template12PolicyDocumentService`, `Template12PolicyDocumentBaseService`, `Template12LandscapeService`):
- reaches XSD: pCertInd, pType, parentName, planDescp, planType, promoAmt, premiumPrint, policyholderName, premDueAftPtv, premDueRoundedAftPtv, prlInd, provInd, polRef, printPoip, dateValue, pmtDueDate, psi, pcopyType, pchecklistCopy, pshowCoverNote, showComplaint (via setPshowComplaint override), showScheduleStd, showDebitNote, pshowTaxInvoice, pPrintSeries, pShowGstLabel/pShowSstLabel (via `setPShowGstLabel/SstLabel`), pAtceInd, pProductCode, pFirstPageNumber (Landscape).
- set but NOT serialised (dropped by whitelist): pContractId, pPolicyNo, pversionNo, pcompanyParamType, showPaymentInfo, copyType, pissueDate, psignIssueDate, signIssueDate, pshowGst/Sst/Tran, PMstPolicyNo, PContractIdList, PCertNoFrom/To, PTaxInvNo, ShowOutPrintCovLetter (setter name differs from whitelisted `pShowOutPrintCovLetter`).

ASP (`AspEndtDocumentService`, `AspEndtDocument2Service`, `AspEndtDocumentLandscapeService`):
- reaches XSD: pcopyType, pchecklistCopy, pPrintSeries, pshowCoverNote, showComplaint, showDebitNote, pshowTaxInvoice, pFirstPageNumber, parentName, planDescp, psi.
- emitted but NOT in the v0.1.6 XSD: `pReportTitle` (Landscape), `pshowEndtSchedule` (whitelisted, so it is emitted).
- gap worth checking: ASP calls `setPshowGstLabel/setPshowSstLabel` (base setters); base properties `pshowGstLabel/pshowSstLabel` are in `@JsonIgnoreProperties`, so ASP's GST/SST label flags likely do NOT reach the XML, unlike Template12 which also calls the upper-case `setPShowGstLabel/SstLabel`. [UNVERIFIED by execution]
- dropped: pEndtNo, pRefNo, pContractId, pVersionNo, pCompanyParamType, pUserID, pissueDate.

RejectRisk (`RejectRiskService`): `pReportTitle`, `reportName`, `pCreatedDateFrom`, `pCreatedDateTo`.
- `pReportTitle`, `pCreatedDateFrom`, `pCreatedDateTo` are whitelisted but ABSENT from v0.1.6 -> schema-invalid. `reportName` is in the base class but not in the whitelist -> dropped. The Jasper RejectRisk declares parameters pCreatedDateFrom/To (java.util.Date), pCustomerName, pIdNo, pRejType, pReasonCode, pReportTitle, reportName, SUBREPORT_DIR, SUBREPORT_DC; only the first two plus title are modelled.

---------------------------------------------------------------------

## 4. Mapping of the three modules onto the XSD

### 4.1 RejectRisk (fully verified)
Java: `RejectRiskService` bean `AGIC/UW/Plines/RejectRisk/RejectRisk`; Jasper: `...\jasper_dumps\jasper_02072026\resources\reports\AGIC\UW\Plines\RejectRisk\RejectRisk\RejectRisk_files\RejectRisk_jrxml.data` (hierarchy csv: main `RejectRisk`, sub `ReportHeader`). Workbook rows agree (RejectRisk_XSD_Mapping_sheet.xlsx).

One `reportData` per DB row, each with one `prntDetails/transDetList` holding one `transDet` plus one `companyType/companyDesc`. After the rows, ONE extra header reportData (`PlineReportBlocks.reportHeader`: mainReportName=RejectRisk, subReportName=ReportHeader, dataClass) if there is at least one row.

| Jasper field | Jasper class | XSD path (under transDetList) | XSD type | Java setter |
|---|---|---|---|---|
| ROWNUMBER | BigDecimal | transDet/rownumber | decimal | setRownumber(toBigDecimal) |
| CUSTOMER_NAME | String | transDet/customerName | string | setCustomerName |
| ID_TYPE_1 | String | transDet/idType1 | string | setIdType1 |
| ID_NO_1 | String | transDet/idNo1 | string | setIdNo1 |
| ID_TYPE_2 | String | transDet/idType2 | string | setIdType2 |
| ID_NO_2 | String | transDet/idNo2 | string | setIdNo2 |
| REJECT_RISK_TYPE_CODE | String | transDet/riskList/risk/rejectRiskTypeCode | string | risk.setRejectRiskTypeCode |
| REJECT_RISK_TYPE | String | transDet/riskList/risk/rejectRiskType | string | risk.setRejectRiskType |
| REASON_CODE | String | transDet/reasoncode (lower-case c) | string | setReasoncode |
| REASON_CODE_DESCRIPTION | String | transDet/reasonCodeDescription | string | setReasonCodeDescription |
| REMARKS | String | transDetList/companyType/companyDesc/remarks | string | companyDesc.setRemarks (sibling of transDet) |
| CREATED_BY | String | transDet/createdBy | string | setCreatedBy |
| CREATED_DATE | sql.Timestamp | transDet/createdDate | dateTime | setCreatedDate(LocalDateTime) |
| REVOKE_IND | String | transDet/revokeInd | string | setRevokeInd |
| REVOKED_DURATION | String | transDet/revokedDuration | string | setRevokedDuration |
| REVOKED_DURATION_DESCRIPTION | String | transDet/revokedDurationDescription | string | setRevokedDurationDescription |
| REVOKE_BY | String | transDet/revokeBy | string | setRevokeBy |
| REVOKE_DATE | sql.Timestamp | transDet/revokeDate | dateTime | setRevokeDate |
| UPDATED_BY | String | transDet/updatedby (lower-case b) | string | setUpdatedby |
| UPDATED_DATE | sql.Timestamp | transDet/updateddate (lower-case d) | dateTime | setUpdateddate |
| (DATA_CLASS lookup via PlineReportHeaderHandler, ReportHeader.jrxml) | - | reportData/dataClass | string | reportData.setDataClass |
| (params) | Date/String | reportParameters/pReportTitle, pCreatedDateFrom, pCreatedDateTo | NOT IN v0.1.6 | rp.setP... |

All 17 transDet leaves above exist in the XSD with the listed types (checked by grep). Jasper parameters pCustomerName, pIdNo, pRejType, pReasonCode are query filters only; they are not in the XML.

### 4.2 Template12 and ASP - main XSD paths used [DERIVED]
Approximate usage by setter name matching against the 634 transDet field names (a setter name hit is an over-estimate because other classes share names): Template12 files: 351 distinct setter names, 213 equal a transDet field; ASP files: 274 / 161; 173 setter names in common. Constructor counts (`new Pline...()`): T12 base service builds 57 PlineTransDet, 15 PlineRisk, 15 PlineCover, 5 PlineStaxPcts, 4 PlineCompanyDesc; T12 PolicyDocumentService adds PlineNominee x4, PlineBenefits x4, PlineBenefit x4, PlineDiagnosis x2; ASP builds PlineTransDet x20, PlineStaxPcts, PlineRisk, PlineNominee, PlineCover, PlineBenefits, PlineBenefit, PlineCompanyDesc, PlineEndorsementDetails.

Paths used (both modules unless noted):
- `reportParameters` (section 3.1).
- `transactionDetails/headerTransaction/reportTransaction/reportData` (via `PlineTransactionDetails.setReportDataList/setPrntDetails`), `reportData/dataClass`, `policystatus`, `status`, `endorsementDetails` (T12 base, ASP: yes; both build `PlineEndorsementDetails`).
- `prntDetails/agentInfo`, `branchInfo`, `clientInfo`, `compInfo`, `taxClientInfo`, `companyType/companyDesc` (T12 builds 4+ PlineCompanyDesc; ASP 1), `buyerInfo`/`supplierInfo` (not constructed by name in these services; come only as default empty objects from `PlinePrntDetails`).
- `transDetList/transDet` leaves (policyNo, effDate, expDate, polsi, gross/nett premium, agent and fee fields: e.g. agentCode, agentName, grossPrem, gstAmt, staxAmt, stiPrem, stiSi, llbPrem, llbSi, polsi, prncnt, issueDate, insuredName, idValue1, ...).
- `transDet/riskList/risk/coverList/cover` (T12 and ASP), `transDet/coverList/cover`, `transDet/sobList/benefits/benefit` (T12 PolicyDocumentService 4 benefits; ASP 1), `transDet/nomineeList/nominee` (T12 x4, ASP x1), `transDet/diagnosisList/diagnosis` (T12 x2; ASP none), `transDet/staxList/staxPcts` (T12 base x5, ASP x1), `transDet/memoList/memo` (Template12 memo list changes; branch OPUSM-36035), `transDetList/companyType`.

Landscape/AspEndtDocumentLandscapeService builds two reportData (two PlineTransDetList, two PlineReportParameters objects) - the second parameter set matters because the schema only allows ONE `reportParameters` per `transaction`. [DERIVED]

---------------------------------------------------------------------

## 5. XSD vs Java audit (payload classes)

Method: awk/comm over the Java private fields (with `@JacksonXmlProperty(localName)` renames and `@JsonIgnore`) versus the XSD sequence for each class. Types compared by mapping String-xs:string, BigDecimal-xs:decimal, Long/Integer-xs:integer, LocalDateTime-xs:dateTime, Boolean-xs:boolean.

### 5.1 Field presence
| Java class | XSD type | Java-only fields | XSD-only leaves |
|---|---|---|---|
| PlineTransDet (639 scalar + 13 lists, 1 ignored) | transDet (634 scalar + 12 lists) | `leasingBranch`, `policystatus`, `printingDate`, `state` (4; `postcode` and `payments` are `@JsonIgnore`) | 0 |
| PlineReportParameters | reportParameters (67) | 6: pCreatedDateFrom, pCreatedDateTo, pReportTitle, pshowEndtSchedule, pShowOutPrintCovLetter, pShowPDPANotice | 0 |
| PlineReportData | reportData | `mainReportName`, `subReportName` (comment says Pline.xsd v0.1.10) | 0 |
| PlineEndorsementDetails | endorsementDetails | 0 | 0 (50 = 50, order and types identical) |
| PlineAgentInfo, BranchInfo, CompInfo, TaxClientInfo, CompanyDesc, Buyer/SupplierInfo, Cover, Nominee, Diagnosis, StaxPcts, Benefit(s), Proviso, Memo, Excess, CsLimit, Specification, PremiumProjection, AgencyInfo, Address, ParamGroup | matching types | 0 | 0 |
| PlineRisk | risk | 0 | 0 |
| PlineClientInfo | clientInfoType | 0 | 0 (mstPol* fields are in the nested class) |
| TemplateDetails (motor, shared) | templateDetailsType | `mergeStaticPdfFiles` (only when non-empty) | 0 |

Summary: Java fields with no XSD leaf = 4 (transDet) + 6 (reportParameters) + 2 (reportData) (+1 optional template detail) = 12; XSD leaves with no Java field = 0 (the 6 transDet "XSD-only" hits in a naive diff are just casing: Columncount, Emailbcc, LOADING, PREM, sumTotalPremiumMEDIX, TableHeaderCOUNT, which Java maps by `@JacksonXmlProperty`).

### 5.2 Element order (xs:sequence) mismatches [RAN on the real sample + static diff]
Java relies on declaration order (Jackson default, no `@JsonPropertyOrder` on these classes; only PlinePrntDetails, PlineTransDetList, PlineTransactionDetails, PlinePaymentsType have it, and those match the XSD).
| Where | XSD order | Java order | Effect |
|---|---|---|---|
| transaction | reportParameters, mailingDetails, templateDetails, transactionDetails | templateDetails, mailingDetails, transactionDetails, reportParameters (class `motor Transaction`) | invalid ordering |
| templateDetails | alphabetical: batchConfigId, boResId, channel, copyTypeList, dataClass, disallowCopy, docType, linkLocation, outputFormat, outputPath, reportName, sequence, templateId, templateName | channel, outputFormat, templateName, copyTypeList, reportName, outputPath, ... (a code comment calls this "XSD order" - it is the Motor XSD order, not Common.xsd) | invalid ordering |
| agentInfo, branchInfo, compInfo, taxClientInfo | `address` FIRST (taxClientInfo: address first, `class` among the Pline additions at the end) | `address` LAST | invalid ordering (4 types) |
| clientInfo | address, age, clientName, clientType, dob, ... | longName, clientType, idType, ... address and mstpoladdress last | invalid ordering |
| buyerInfo | address1-3, buyerRegNo, buyerSstRegNo, buyerTin, city, ... | name first | invalid ordering |
| supplierInfo | city, country, name, postCode, state, supplierAdd1... | different order | invalid ordering |
| cover | ..., excessList, provisoList (before Pline additions) | additions first, excessList/provisoList last | invalid ordering |
| risk | ..., additionRisk, occpDescp, rejectRiskType, ... | occpDescp moved to the end of the scalar fields | invalid when occpDescp is present |
| transDet | LOADING before loading; PREM before premiumExcludeGST; companionName earlier | `loading` before `LOADING`; `prem` after polsi; companionName later; extra leasingBranch/policystatus/printingDate/state | invalid ordering |
| reportData | slNo, dataClass, permanentDisablement, Policyletterind, ... | slNo, dataClass, **mainReportName, subReportName**, permanentDisablement, ... | invalid (unknown elements) |
| benefits | benefit, benefitHdr, benefitType, plan, planPrint | current PlineBenefits: benefit first (match). The old Sep-1 sample shows benefitHdr first | current code OK, old sample not |

### 5.3 Type mismatches
- Dates: every `LocalDateTime` has `@JsonFormat(pattern="dd/MM/yyyy")` + `LocalDateTimeSerializer`, so the wire value is `06/06/2017`; the XSD requires `2017-06-06T00:00:00`. Applies to all 64 dateTime leaves of Pline.xsd that Java populates, incl. `createdDate`, `revokeDate`, `updateddate`, `pBeforeEffDate`, `pLaunchDate`, `dateValue`, `pmtDueDate`, `policyGeneratedDate`, `printingDate`, header `effectivedatefrom/to`, `endtdatefrom/to`, `riskDob`, `riskEffDate`, `riskExpDate`. The test file itself says this in a comment.
- `PlineBenefit.benefitSi`: Java `String`, XSD `xs:decimal` (the only primitive type mismatch I found in the classes compared).
- Empty elements: payload classes use `@JsonInclude(ALWAYS)`, and plain `new XmlMapper()` (CommonController) writes null as `<x/>`. For xs:decimal/dateTime/integer/boolean this is invalid. In the real 1357766.xml sample this is 456 of 929 errors (387 decimal, 56 dateTime, 10 integer, 3 boolean). This is the biggest-volume failure.
- Case: sample showed `policyletterind`; current `PlineReportData` fixes it with field visibility so `Policyletterind` is emitted.

### 5.4 mainReportName / subReportName / dataClass vs v0.1.6 and newer XSD
- v0.1.6: `reportData` has dataClass (string) but NOT mainReportName/subReportName. Java `PlineReportData` and `PlineReportNames.tag()` write `mainReportName` on EVERY reportData of every Plines service (commit e74bc0d5c, 2026-10-07, and 861f0666c for listing reports), and `PlineReportBlocks` writes subReportName. [RAN] A document with those two elements fails validation: "Invalid content was found starting with element 'mainReportName'. One of '{permanentDisablement, Policyletterind, ...}' is expected."
- Where could a "v0.1.10" XSD be? Searched: `git log --all -- '*.xsd'` returns only 3 commits (d4575fa5b 2026-08-03, 6961abbf9 2026-08-08, ea782121d 2026-09-28), all touching the same test-resource copy; none of the branches I checked (origin/OPUSM-44159-retrofit-develop-pline-jrxml-comparison-fixes, origin/feature/retrofit-develop-pline) contain mainReportName, subReportName or pCreatedDateFrom; `git worktree list` shows only the main worktree. `find` over `C:\Users\EPGRVFY\Documents` found Pline.xsd only in: the v0.1.6 reference folder, `ASPEndtSchedule\base_code\...`, `Plines_Base_code\New_branch\...` (two), `new_springboot_upgraded\...` (+ target), `T12_final_code\...` (+ target) - all 55255bec56 (v0.1.6) - and one mapping workbook `Pline_MASTER_MAPPING_SHEET_v0.1 (final1) 1 5.xlsx` (v0.1, 2026-07-06) plus the RejectRisk sheet. So v0.1.7 - v0.1.10 exist only as comments in Java (`grep` counts in HEAD: "Pline.xsd v0.1.6" ~228, v0.1.7 x2, v0.1.10 x2, v0.1.4 x1; "XSD v0.1.9" in PlineReportParameters). The newer XSD files are not available to me.

### 5.5 Cardinality
- `Transaction.templateDetails/mailingDetails/transactionDetails/reportParameters` are 0..1 in XSD and single objects in Java: OK.
- `headerTransaction`, `reportTransaction`, `reportData`, `prntDetails`, `transDet`, `risk`, `cover`, `paramGroup`, `companyDesc`, `benefit(s)`, `nominee`, `staxPcts` are lists in Java with `useWrapping=false` where the XSD is unbounded: OK. `reportTransaction` is 0..1 in XSD but `List` in Java (always size 1): harmless.
- `PlineHeaderTransaction.reportTransaction` default list holds one empty `PlineReportTransaction` and `PlineReportTransaction.reportData` defaults to one empty `PlineReportData`; this makes "empty" responses still contain an empty reportData with many empty elements.

---------------------------------------------------------------------

## 6. PlineXsdValidationTest: what it does, limits, and a proper test

### 6.1 What it does [JAVA]
`src/test/java/com/agcs/fo/opus/plines/PlineXsdValidationTest.java` (commit 6961abbf9). It loads `/xsd/plines/Pline.xsd` from the test classpath, converts the URL to a real `File` so that the relative `xs:include Common.xsd` resolves, builds a `Schema` with `SchemaFactory.newInstance(W3C_XML_SCHEMA_NS_URI)`, and validates a string.
- `minimalShellXml_isValid`: `<transactions><transaction><reportParameters/><transactionDetails/></transaction></transactions>` must validate.
- `knownInvalidXml_failsValidation`: `<bogusUnknownElement>` must throw `SAXException`.
- 4 "service" tests (template4Landscape, template12Landscape, template12PolicyDocument, template12PolicyDocument2) each re-validate the SAME hand-written shell: placeholders. The in-file comment says real XML is not used until date formatting and transDet/sobList ordering are reconciled.

### 6.2 Limits
1. Never calls any service or serialiser, so it proves the XSD loads and nothing about the application.
2. Names say "generates valid XML" but assert nothing about generated XML (misleading; a reviewer could believe coverage exists).
3. Valid shell is trivially valid: every element is optional, so empty containers pass.
4. Only the first fatal error is thrown; no error collection, so you cannot see the whole picture.
5. Not run in this analysis (I did not run the Maven test); the equivalent behaviour was reproduced with my own JDK validator: my hand-built minimal XML passes (0 errors) and a document with an unknown element fails.
6. Does not cover Jasper-side expectations (nothing proves the print engine accepts the XML).

### 6.3 Design for a proper test (prose + sketch; NOT written to the repo)
Approach: for each service, build a `PayLoadRequest` with a mocked handler (RejectRisk is easiest: one handler `RejectRiskHandler.fetchRejectRisks` and `PlineReportHeaderHandler.fetchReportHeader`), call `service.execute(...)`, serialise with the SAME mapper used in `CommonController` (a plain `new XmlMapper()`; extract a shared bean so test and production cannot drift), then validate with an error-collecting `ErrorHandler` and assert zero errors. Use a `@ParameterizedTest` over services, and keep a second, hand-maintained "golden" XML for one case. Use `@Tag("xsd")` and fail the build only for rules the team decides are mandatory, otherwise report counts (ratchet: assert errors <= baseline and lower the baseline as fixes land).

```java
// sketch only
class RejectRiskXmlMatchesXsdTest {
  @Mock RejectRiskHandler handler; @Mock PlineReportHeaderHandler header;

  @Test void rejectRiskOutputValidatesAgainstPlineXsd() throws Exception {
    when(handler.fetchRejectRisks(any(), any(), any(), any(), any(), any()))
        .thenReturn(List.of(sampleDto()));          // all fields populated incl. dates
    when(header.fetchReportHeader(any(), any())).thenReturn(new PlineReportHeaderDto("RR01"));
    PayLoadRequest req = request(Map.of("pReportTitle","PLINES : REJECT RISK REPORT",
        "reportName","AGIC/UW/Plines/RejectRisk/RejectRisk","pCreatedDateFrom","06/06/2017",
        "pCreatedDateTo","31/08/2017"));
    Transactions tx = new RejectRiskService(handler, header).execute(req);
    String xml = new XmlMapper().writeValueAsString(tx);          // same as CommonController

    List<String> errors = new ArrayList<>();
    Validator v = loadPlineSchema().newValidator();
    v.setErrorHandler(collecting(errors));                         // do not stop at first
    try { v.validate(new StreamSource(new StringReader(xml))); } catch (SAXException ignore) {}
    assertThat(errors).as(String.join("\n", errors)).isEmpty();
  }
}
```
Add: a test that `Transaction`/`TemplateDetails` ordering is right; a unit test per leaf type (a date leaf must serialise to ISO-8601); and a drift guard that parses the XSD and compares leaf names to Java fields by reflection (my awk audit is the manual version of this).

### 6.4 What would fail today and why (RejectRisk)
[RAN against the real 1357766.xml / 2439275.xml, converted from the JSON-wrapped response; the code has moved on since Sep 1, so treat counts as a lower-bound style estimate]:
- 1357766.xml: 929 errors = 456 datatype pairs (empty decimal/dateTime/integer/boolean, each reported twice by Xerces: `cvc-datatype-valid` + `cvc-type.3.1.3`) + 17 structure errors. 2439275.xml: 1766 errors.
- Structure errors seen (order/unknown element, one per container because Xerces stops after the first in each): `copyTypeList` out of order (templateDetails), `mailingDetails` before `transactionDetails`, `address` after fields in agentInfo/branchInfo/compInfo/taxClientInfo, `clientType` ahead of required position in clientInfo, buyerInfo `address1` before `postCode/state`, supplierInfo `city` ahead, `companionName` in transDet, `provisoList` in cover, `benefit` in benefits, `IDNo` in nominee, `occpDescp` in risk, `policyletterind` (old casing), `indicationList` (since removed by `@JsonIgnoreProperties`), `pendtNo` (not whitelisted any more).
- With today's code additionally expect: mainReportName/subReportName in every reportData (unknown element), reportParameters `pReportTitle`, `pCreatedDateFrom`, `pCreatedDateTo` (unknown), dd/MM/yyyy dates in createdDate/revokeDate/updateddate and in pCreatedDateFrom, empty `<slNo/>`, `<permanentDisablement/>`, `<effectivedatefrom/>` etc. (null BigDecimal/LocalDateTime written as empty elements), reportParameters emitted after transactionDetails, and the whole transDet carrying hundreds of empty decimal elements.
- Proof that the schema itself is workable: a hand-written RejectRisk XML in the correct order, ISO dates and no empty numeric elements validated with 0 errors [RAN] (it is the sample in 7.1). A variant with the same content but with mainReportName/subReportName, `06/06/2017` for createdDate, and reportParameters after transactionDetails gave 5 errors, all of those categories.

---------------------------------------------------------------------

## 7. Sample XML

### 7.1 RejectRisk - DERIVED from RejectRiskService + payload classes, trimmed (1 data row)
Labelled DERIVED: this is what the code intends, shown in the XSD-correct form that I validated with a JDK validator (0 errors). The REAL current output differs: (a) element order per 5.2, (b) hundreds of empty elements (`<slNo/>`, `<agentInfo>...`), (c) `mainReportName`/`subReportName` in each reportData, (d) dd/MM/yyyy dates, (e) reportParameters (pReportTitle, pCreatedDateFrom, pCreatedDateTo) emitted last.
```xml
<?xml version="1.0" encoding="UTF-8"?>
<transactions>
  <transaction>
    <reportParameters/>                      <!-- real code: pReportTitle, pCreatedDateFrom, pCreatedDateTo (not in v0.1.6) -->
    <mailingDetails>
      <address><addressline1/><addressline2/></address>
      <email>recipient@example.com</email>
    </mailingDetails>
    <templateDetails>
      <batchConfigId>1</batchConfigId><boResId>1</boResId><channel>WEB</channel>
      <copyTypeList><copyType>ORIGINAL</copyType></copyTypeList>
      <docType>PDF</docType><outputFormat>PDF</outputFormat>
      <reportName>AGIC/UW/Plines/RejectRisk/RejectRisk</reportName>
      <templateId>1</templateId><templateName>RejectRisk</templateName>
    </templateDetails>
    <transactionDetails>
      <headerTransaction>
        <reportTransaction>
          <!-- one reportData per DB row -->
          <reportData>
            <dataClass>RR01</dataClass>      <!-- from PlineReportHeaderHandler (RPGE_RPT_TEMPLATE); value is a made-up example -->
            <prntDetails>
              <companyType><companyDesc><remarks>Duplicate ID</remarks></companyDesc></companyType>
              <transDetList>
                <transDet>
                  <createdBy>USER1</createdBy>
                  <createdDate>2017-06-06T00:00:00</createdDate>   <!-- code writes 06/06/2017 -->
                  <customerName>TAN T357766</customerName>
                  <idNo1>841004017669</idNo1>
                  <idType1>NRIC</idType1>
                  <reasoncode>09</reasoncode>
                  <revokeInd>N</revokeInd>
                  <rownumber>1</rownumber>
                  <riskList>
                    <risk><rejectRiskType>Customer</rejectRiskType><rejectRiskTypeCode>C</rejectRiskTypeCode></risk>
                  </riskList>
                </transDet>
                <companyType><companyDesc><remarks>Duplicate ID</remarks></companyDesc></companyType>
              </transDetList>
            </prntDetails>
          </reportData>
          <!-- after the rows: ReportHeader block (code adds mainReportName=RejectRisk, subReportName=ReportHeader) -->
          <reportData><dataClass>RR01</dataClass></reportData>
        </reportTransaction>
      </headerTransaction>
    </transactionDetails>
  </transaction>
</transactions>
```
(The first `companyType` inside prntDetails is optional and the service actually fills only the one inside `transDetList`; shown twice only to illustrate the two legal places - drop the first for fidelity. The actual trailing ReportHeader reportData was not part of the validated file.)

### 7.2 Minimal ASP / Template12 skeleton - DERIVED (schema-valid shape; values illustrative)
```xml
<transactions>
  <transaction>
    <reportParameters>
      <parentName>...</parentName><pchecklistCopy>N</pchecklistCopy><pcopyType>ORIGINAL</pcopyType>
      <pFirstPageNumber>1</pFirstPageNumber><planDescp>...</planDescp><psi>100000</psi>
      <pshowCoverNote>Y</pshowCoverNote><pshowTaxInvoice>N</pshowTaxInvoice><showDebitNote>N</showDebitNote>
    </reportParameters>
    <templateDetails>...</templateDetails>
    <transactionDetails>
      <headerTransaction>
        <reportTransaction>
          <reportData>
            <dataClass>..</dataClass>
            <policystatus>..</policystatus>
            <endorsementDetails><endtNo>..</endtNo><endtEffDate>2026-01-01T00:00:00</endtEffDate></endorsementDetails>
            <prntDetails>
              <agentInfo><address/><agentCode>..</agentCode><agentName>..</agentName></agentInfo>
              <branchInfo><address/><branchCode>..</branchCode></branchInfo>
              <clientInfo><address/><clientName>..</clientName></clientInfo>
              <compInfo><address/></compInfo>
              <companyType><companyDesc><companyName>..</companyName></companyDesc></companyType>
              <taxClientInfo><address/></taxClientInfo>
              <transDetList>
                <transDet>
                  <agentCode>..</agentCode> ... <polsi>100000</polsi>
                  <coverList><cover><coverCode>..</coverCode></cover></coverList>
                  <staxList><staxPcts><staxPct>6</staxPct></staxPcts></staxList>
                  <sobList><benefits><benefit><benefitDescp>..</benefitDescp></benefit><benefitHdr>..</benefitHdr></benefits></sobList>
                  <nomineeList><nominee><IDNo>..</IDNo><name>..</name></nominee></nomineeList>
                  <diagnosisList><diagnosis><icdNarration>..</icdNarration></diagnosis></diagnosisList>
                  <memoList><memo><memoCode>..</memoCode></memo></memoList>
                  <riskList><risk><riskName>..</riskName><coverList><cover/></coverList></risk></riskList>
                </transDet>
                <companyType>...</companyType>
              </transDetList>
            </prntDetails>
          </reportData>
        </reportTransaction>
      </headerTransaction>
    </transactionDetails>
  </transaction>
</transactions>
```
This skeleton was NOT validated by me (only the RejectRisk document was); it follows the sequences read in the XSD.

---------------------------------------------------------------------

## 8. Namespace and compatibility discussion

What the downstream consumer (print engine / Jasper report generation) sees [DERIVED]
- A no-namespace XML tree rooted at `transactions`, with up to ~650 child elements per transDet and many empty elements (`ALWAYS` inclusion). Optionality: the XSD declares everything optional, so a consumer written to the XSD must treat all of it as nullable; in practice the consumer picks by element name (XPath-like), where ORDER normally does not matter but names, paths and text formats do.
- Whether the print engine actually validates against Pline.xsd is not evidenced anywhere in the repo or documents I read. The only machine check is the test above; the developer-supplied reference XSD is described as the contract. [UNVERIFIED]

Safe vs breaking changes
| Change | Safe? | Why / how this repo handled it |
|---|---|---|
| Add new optional leaf at the END of a sequence or at its alphabetical slot | Safe for readers keyed by name; for a strict validator older schemas reject the new element (forward-incompatible), newer schemas accept old files (backward-compatible) | v0.1.5, v0.1.6 (pAgentChannel) did this; Java adds field in the same relative position. Placing mid-sequence is fine only if Java and XSD agree on the position. |
| Add new optional complex node | Safe if optional | e.g. nested lists |
| Make an optional element required | Breaking | Never done; nothing is required. |
| Remove or rename element (v0.1.4: showCoverNote, showTaxInvoice, prem, totalpremium) | Breaking | Handled with "shadow fields" + `@JsonIgnoreProperties` in `PlineReportParameters` to emit only the new names. |
| Move element (v0.1.3 prntDetails, v0.1.4 permanentDisablement) | Breaking | Java moved the fields accordingly. |
| Reorder siblings in a sequence | Breaking for validators; harmless for name-based readers | Not controlled in Java for most classes (no `@JsonPropertyOrder`), the root cause of 5.2. |
| Change type (string -> dateTime, v0.1.6) | Breaking for senders | Java did not follow it (still dd/MM/yyyy). |
| Case change | Breaking (XML is case sensitive) | `Policyletterind`, `Emailbcc` etc. handled with `@JacksonXmlProperty`/field visibility. |
| Add elements the XSD does not know (mainReportName, pCreatedDateFrom) | Invalid against the reference XSD; harmless to name-based consumers | The repo did this ahead of the XSD, citing v0.1.9/v0.1.10. This is the key governance gap. |

How the repo coped overall: Java payload classes mirror XSD field names and (mostly) order (commit 6961abbf9 "Align Pline requestpayload types with XSD" - 137 lines changed in PlineReportParameters, 82 in PlineCover, 74 in PlineEndorsementDetails, GHS-specific classes removed), a whitelist stops base-class leakage, and a shared `PlineCommonMappingService` is used for T1/T2 (commit 6af525686). What it did not do: enforce order for every class, ISO dates, or CI validation of real output.

Recommendation: decide the contract is "name/path/format as the Jasper side reads it", then update the XSD to what Java really emits (dates as a pattern type, add mainReportName/subReportName/pCreatedDate*), or change Java to the XSD - not both drifting.

---------------------------------------------------------------------

## 9. 40 interview questions (about 1.6 years experience level)

Format: Q, strong answer, common mistake, follow-up.

1. Q: Walk me through the Pline.xsd root and the path to a transDet.
   A: `transactions > transaction > transactionDetails > headerTransaction > reportTransaction > reportData > prntDetails > transDetList > transDet`; reportParameters, mailingDetails, templateDetails are siblings of transactionDetails; riskList sits inside transDet.
   Mistake: placing prntDetails beside reportData (v0.1.3 fixed that). Follow-up: where does companyType live? (two places).
2. Q: Does the schema have a namespace? A: No targetNamespace; elementFormDefault is qualified but irrelevant; `xs:include` of Common.xsd works because both have no namespace. Mistake: adding xmlns in output. Follow-up: what would change if it had a namespace? (use import, prefix in XML, Jackson `@JacksonXmlRootElement(namespace=...)`).
3. Q: Which elements are mandatory? A: Effectively only `transaction` (1..unbounded); all 1062 other local elements are `minOccurs=0`. Mistake: assuming "XSD valid" means data complete. Follow-up: how would you enforce a mandatory field? (change minOccurs, or business validation in service).
4. Q: xs:sequence vs xs:all vs xs:choice? A: Pline.xsd uses only sequence: order is enforced. Mistake: thinking the order is irrelevant for XML. Follow-up: how does Jackson decide order? (declaration order, `@JsonPropertyOrder`).
5. Q: Which scalar types are used and how many dates? A: string, decimal, dateTime, integer, boolean; 64 dateTime in Pline.xsd, no xs:date. Mistake: calling them Date. Follow-up: valid lexical format for dateTime?
6. Q: What is wrong with `<createdDate>06/06/2017</createdDate>`? A: Not ISO 8601; xs:dateTime needs `2017-06-06T00:00:00`. Mistake: believing the schema accepts any string. Follow-up: how do you fix it without breaking Jasper? (separate formatted field, or XSD pattern type, or convert in print engine).
7. Q: Is `<x/>` valid for xs:decimal? A: No; only for xs:string. Null Java values with `Include.ALWAYS` produce `<x/>`. Mistake: thinking empty = absent. Follow-up: options? (`NON_NULL`/`NON_EMPTY`, or `xsi:nil` + `nillable`).
8. Q: Explain the v0.1.3 change. A: prntDetails moved inside reportData as last child; breaking path change. Mistake: calling it "just a refactor". Follow-up: how does Java mirror it? (`PlineReportData.prntDetails`).
9. Q: Why was permanentDisablement moved (v0.1.4)? A: Requirement: it's row-level data; now reportData child (element, not attribute despite header text). Follow-up: impact for consumer reading reportParameters.
10. Q: Why remove showCoverNote and showTaxInvoice? A: duplicates of pshowCoverNote / pshowTaxInvoice (Jasper parameter names). Mistake: calling it backward-compatible. Follow-up: how did Java keep old callers compiling? (shadow fields and overridden setters).
11. Q: What changed in v0.1.5? A: Additive: 8 reportParameters strings, LOADING + premium-particulars sum decimals, COLUMN_COUNT/TableHeader_COUNT, EMAIL_BCC. Follow-up: why `LOADING` and `loading` both? (case sensitivity, different origin).
12. Q: Why was pBeforeEffDate retyped? A: date value from BR_EFF_DATE Timestamp, consistent with other date leaves. Mistake: not mentioning that Java still sends dd/MM/yyyy. Follow-up: is that change compatible?
13. Q: Difference between additive and breaking schema change? A: see section 8 table. Mistake: "adding a leaf is always safe" (not for strict old validators). Follow-up: how to roll out an additive change safely? (deploy consumer first / tolerate unknown).
14. Q: What are the pline* extension types? A: Common type body + Pline-only additions (e.g. plineAgentInfoType = agentInfoType + agentCatType...). Follow-up: why not `xs:extension`? (they copy the body; Common stays byte-for-byte unchanged; extension would append and change ordering).
15. Q: Where is `address` in plineAgentInfoType vs in Java? A: First in XSD, last in `PlineAgentInfo`. Fix: `@JsonPropertyOrder` or move field. Follow-up: how many types have this issue? (agent, branch, comp, taxClient, client).
16. Q: How are renamed/odd-cased elements produced in Java? A: `@JacksonXmlProperty(localName=...)`, field visibility (`Policyletterind`, `LOADING`, `Columncount`). Mistake: relying on Lombok getters (would emit `policyletterind`). Follow-up: what does `@JsonAutoDetect` do?
17. Q: What does `@JsonIncludeProperties` in PlineReportParameters do? A: Whitelist of 73 property names so base-class (Motor) fields don't leak. Mistake: forgetting that a setter to a non-whitelisted property is silently dropped (e.g. pContractId). Follow-up: which 6 whitelisted names are not in v0.1.6?
18. Q: How does RejectRisk map REMARKS? A: `transDetList/companyType/companyDesc/remarks`, a sibling of transDet. Mistake: putting it in transDet. Follow-up: why one companyType per row? (each reportData has its own transDetList).
19. Q: Which transDet leaf names in RejectRisk are irregular? A: `reasoncode`, `updatedby`, `updateddate`, `idType1/idNo1`, `rownumber`. Follow-up: how did you confirm? (grep XSD + mapping sheet).
20. Q: What's the XSD type of createdDate/revokeDate/updateddate and ROWNUMBER? A: dateTime x3 and decimal. Follow-up: how are they converted in the service? (`toLocalDateTime`, `toBigDecimal`).
21. Q: Where does RejectRisk get dataClass? A: PlineReportHeaderHandler lookup (RPGE_RPT_TEMPLATE) using pReportTitle + reportName; set on each reportData and on the trailing ReportHeader block. Follow-up: what is subReportName for the data rows? (empty).
22. Q: What are mainReportName and subReportName? A: Added by `PlineReportNames.tag()` and `PlineReportBlocks` so the print engine can tie a reportData to the main/sub jrxml. Not in v0.1.6; Java comment cites v0.1.10. Mistake: claiming they are in the XSD. Follow-up: consequence for validation?
23. Q: Where is the newest XSD? A: Not found; only v0.1.6 copies exist; Java comments refer to v0.1.7-0.1.10. Good answer states uncertainty and asks the schema owner. Follow-up: how would you reconcile?
24. Q: What does PlineXsdValidationTest verify? A: Only that the schema loads, a minimal shell is valid, an unknown element is invalid; placeholder service tests reuse the same shell. Follow-up: what's missing? (real service output).
25. Q: Why must the test use a File-backed StreamSource? A: So the relative include `Common.xsd` resolves; an InputStream has no base URI. Follow-up: alternative? (LSResourceResolver / systemId).
26. Q: How would you validate real output? A: mock handlers, run service, serialise with the production mapper, collect all errors with an ErrorHandler, assert zero or a ratchet baseline. Follow-up: why collect all?
27. Q: What do you expect to fail today? A: element order (transaction, templateDetails, address-first types, cover, risk, transDet), empty numeric/date elements, dd/MM/yyyy dates, mainReportName/subReportName, pCreatedDateFrom/To, pReportTitle. Follow-up: priority? (order + types first).
28. Q: Why does the real transaction order fail? A: Java `Transaction` field order is templateDetails, mailingDetails, transactionDetails, reportParameters; XSD wants reportParameters, mailingDetails, templateDetails, transactionDetails. Follow-up: where would you fix it? (`@JsonPropertyOrder` on the shared Transaction class - affects Motor too, check blast radius).
29. Q: Why can't you just edit the shared motor classes? A: Motor/Fire/Bond extend them, so Plines uses subclasses/shadow fields. Follow-up: how do PlineTransactionDetails and PlineReportParameters isolate Plines? 
30. Q: How many leaves does reportParameters have and what types? A: 67: 43 string, 17 decimal, 6 dateTime, 1 integer. Follow-up: which are dates?
31. Q: What's the difference between `rp` for Template12 and ASP? A: Template12 sets many p* flags and plan/premium values (pCertInd, pType, premDue*, psi, prlInd, provInd...); ASP sets copy/checklist/print-series/show flags and plan values; ASP landscape also sets pReportTitle (not in XSD). Follow-up: which setters are dropped?
32. Q: Why is only one reportParameters allowed even for multi-reportData responses? A: Schema is 0..1 per transaction (comment in AhiEndtDocumentService). Follow-up: how to represent per-reportData parameters? (fields on reportData or paramGroup).
33. Q: What is paramGroup? A: Common type with dataType, paramCode, paramValue, repeated in prntDetails; generic name/value carrier. Follow-up: when would you use it instead of a new leaf? (avoid schema change).
34. Q: What's the effect of `elementFormDefault="qualified"` when there is no namespace? A: None in practice; matters when a targetNamespace is added. Mistake: saying it "adds namespaces".
35. Q: How do you check counts in the XSD without tools? A: grep -c on `<xs:element`, `type="xs:decimal"`, `minOccurs="0"` and awk ranges between known anchors; verify with xmllint or a JDK validator. Follow-up: pitfalls? (comments, multi-line declarations).
36. Q: How would you add a new transDet leaf end to end? A: add leaf in XSD in alphabetical slot, same position and type in `PlineTransDet`, populate in the service, update mapping sheet and changelog header, bump version, add a test. Mistake: forgetting position/case. Follow-up: and for a rename?
37. Q: A date leaf must stay dd/MM/yyyy for Jasper but the XSD says dateTime. Options? A: (a) emit ISO and format in the report, (b) retype to string / pattern simpleType, (c) add a parallel string leaf. Trade-offs: (a) breaks Jasper expressions, (b) loses semantic typing, (c) schema growth. Follow-up: which did the team pick? (none; code and XSD disagree).
38. Q: How would you prove a Java payload class matches an XSD type? A: reflection test that compares field names/order/types against parsed XSD; or generate classes with JAXB `xjc` from the XSD. Follow-up: why not generate? (Jackson XML + existing classes, many shared Motor types).
39. Q: What is `xs:include` vs `xs:import`? A: include merges a same-namespace (or no-namespace) schema; import brings in a different namespace. Follow-up: what about `xs:redefine`? (deprecated; avoid).
40. Q: What risks do you see in the current state? A: Contract drift (XSD v0.1.6 vs Java v0.1.10 comments), order and type violations, unvalidated claims in test names, silent drops by whitelist, no CI check. Good answers propose the ratchet test and a schema-owner agreement. Follow-up: how would you communicate this to the print-engine team?

---------------------------------------------------------------------

## 10. Uncertainties and things not verified

1. The newest XSD (v0.1.7 - v0.1.10) referenced by Java comments was not found anywhere under `C:\Users\EPGRVFY\Documents`, in the repo history, or in the two remote branches I checked. Whether mainReportName/subReportName/pCreatedDateFrom/To are in an agreed newer schema is unknown.
2. The RejectRisk "real output" files are from 1 Sep 2026 and from the `T12_final_code` copy. The error counts (929 / 1766) describe that older output, not the current branch. I did not run the application or the Maven test suite, so the "what would fail today" list for current code is [DERIVED], with the 5-error experiment as the only executed evidence for the new elements.
3. Whether the print engine validates against the XSD, and whether it tolerates unknown elements, is not documented anywhere I read.
4. The Template12/ASP mapping (4.2) is based on setter-name matching and constructor counts, not on executing the services; some setter names belong to other classes. The ASP pShowGstLabel/pShowSstLabel drop and Template12's ShowOutPrintCovLetter drop are inferred from base-class property names.
5. Mapping workbooks were only skimmed via shared strings (RejectRisk sheet read fully; master workbook only the sheet list and intro). I did not cross-check all tabs against the XSD.
6. I did not execute the 40-item interview content against a reviewer; the answers reflect the code as read.
7. The statement "Xerces reports one structure error per container" and hence that the 17 structure errors are a lower bound is standard Xerces behaviour but I did not test every container.
8. Scratch files used for validation are in `C:\Users\EPGRVFY\AppData\Local\Temp\xsdcheck` (not in the repo, safe to delete). The Java validator `V.java` is there too.
