# 17 - XSD Mapping Workbooks and Real Payloads (RejectRisk, ASP, Template12)

Companion to `04_XSD_XML_Deep_Dive.md` (read that first). This file adds three things that file did not have:
(a) what the mapping workbooks really contain and how the mapping process worked,
(b) the real request payload shape per module,
(c) validation of the NEWEST real responses against Pline.xsd v0.1.6, compared with the old finding (929 / 1766 errors).

Evidence tags used everywhere:
- [WB] read from a workbook (xlsx unzipped and parsed by me, nothing opened in Excel, nothing modified).
- [RAN] I executed it (JDK 26.0.1 `javax.xml.validation`, my own small Java classes; scratch only in `C:\Users\EPGRVFY\AppData\Local\Temp\xsdcheck2`).
- [GIT] read from `git log` / `git show` of `opus-documentprintdata-service`.
- [CODE] read from the Java source at HEAD 861f0666c (2026-10-08).
- [INFER] my inference, not proven. [CONFIRM] = ask the developer; the files do not settle it.

Reference schema used: `Pline.xsd` + `Common.xsd` v0.1.6 from the reference folder (md5 of Pline.xsd `55255bec56ce...`, same as every other copy on disk). Sensitive content: hostnames, tokens and cookies are written as `<redacted>`; no token/cookie/password exists in any curl file I read (the T12 curl had a host name, redacted). Names, IDs and amounts in skeletons are placeholders.

---------------------------------------------------------------------

## 1. Workbook anatomy

All files were parsed straight from the xlsx zip (sharedStrings + sheet XML). Row counts below exclude header rows unless stated.

### 1.1 Master mapping sheet - `Documents\Pline_MASTER_MAPPING_SHEET_v0.1 (final1) 1 5.xlsx` [WB]
Last modified 2026-08-25. Change log tab says "Generated 2026-07-06", v0.1, "Base pattern = Bond (Bond.xsd + Common.xsd)", "Content driven by the Plines report set (784 jrxml, 20 report families)", Common.xsd "included byte-for-byte (unmodified)".

Tabs: `_ChangeLog` + 21 report-family tabs. Data rows per tab (header is row 5, data from row 6):
EndtDocument 474, PolicyDocument 551, AutoRenewal 135, CharityReport 49, ScrutinyRejectedChecklist 39, Auto_Policy 36, EndorsementStatusReport 34, HealthInsurancePremiumStatement 32, CIOutsourcePrinting 25, CancellationPolicy 24, ScrutinyRejection 23, MyegStatus 21, **RejectRisk 21**, CardPrinting 20, ForeignWorker 17, CIUploadPolicyGenerated 16, CIUploadPendingConfirmed 14, PolicyEndtEncodedNotPrinted 12, UnderwritingErrors 11, CIUnderWritingErrors 10, ScrutinyRejectionSummary 7. Total 1571 rows.
Mapping of our modules: **Template12 = `PolicyDocument` tab**, **ASP = `EndtDocument` tab**, **RejectRisk = `RejectRisk` tab** [INFER from names; the code/bean names match].

The 16 columns (identical on every tab), what each means:
| Col | Header | Meaning / observed content |
|---|---|---|
| A | Source Subreport (Jrxml) | the jrxml where the field was found |
| B | Jrxml File Name | for PolicyDocument/EndtDocument this is a semicolon list of every jrxml that uses the field (e.g. 35 files for CITY); for RejectRisk it is just `RejectRisk.jrxml` |
| C | Oracle Table / View Name | almost always EMPTY (only 66 of 551 rows on PolicyDocument) - the DB link is NOT carried in this sheet |
| D | Existing in Report Y/N | `Y` on 100% of rows |
| E | Attributes | the Jasper field/variable name (UPPER_SNAKE, e.g. `REVOKED_DURATION`) |
| F | SmartComm | a straight copy of E (generated; carries no extra information) |
| G | Mongo DB collection Name | empty |
| H | Attributes | again a copy of E (this is the lookup key the RejectRisk script uses) |
| I | XSD Attribute Name | the XSD element (camelCase, e.g. `revokedDuration`) = the decision |
| J | XSD Group (Identified) | the XSD container where I lives, e.g. `transDet`, `risk`, `address(clientInfo)`, `companyDesc(transDet)` |
| K | XSD Group (To be added) | same as J on every row I checked (never different) |
| L | Existing in XSD Y/N | `Y` on 100% of rows of every tab I counted (0 rows with N) |
| M | JAVA DTO | the Java payload class name implied by J (transDet, risk, address...) |
| N | Datatype | `xs:string`, `xs:decimal`, `xs:dateTime`, `xs:integer` (RejectRisk: 17 string, 1 decimal, 3 dateTime) |
| O | Format | empty (only the header cell) |
| P | Comment / Derivation | one of three boilerplate sentences (see below) |

Comment (P) categories - this is the closest thing to a "status" column [WB]:
| Tab | rows | "New Pline field (no Bond/Common equivalent)" | "Reuses existing Bond/Common leaf" | "Mirrors Bond: field maps to ..." |
|---|---|---|---|---|
| PolicyDocument (T12) | 551 | 292 | 79 | 180 |
| EndtDocument (ASP) | 474 | 263 | 59 | 152 |
| RejectRisk | 21 | 16 | 4 (createdBy, reasoncode, updatedby, updateddate) | 1 (REMARKS -> remarks) |

Quality notes found while reading [WB]:
- Column L ("Existing in XSD") is `Y` everywhere, but the RejectRisk row for `DATA_CLASS` has a comment that says "Existing-in-XSD=N". The column was not kept in sync with the comment text. The sheet was generated together with the XSD, so "Y" mostly means "we put it there".
- Name drift against the final XSD v0.1.6 (exact-name check of column I against all 1072 distinct element names of the XSD, case sensitive): RejectRisk 0 misses; PolicyDocument 3 misses (`perilsCount` vs XSD `perilscount`, `policyHolderName` vs `policyholderName`, `prem` vs `PREM` - `prem` was removed in v0.1.4); EndtDocument 1 miss (`policyHolderName`); AutoRenewal 1 miss (`policyLetterInd` vs `Policyletterind`). So the workbook is v0.1 (6 Jul) and the XSD moved on to v0.1.6 (28 Jul) without the sheet being re-synced. [RAN name check]
- One Jasper name often maps to the same XSD leaf in different groups: e.g. `CITY` -> `city` in `address(clientInfo)`, `address(compInfo)`, `address(agentInfo)`, buyerInfo. About 30 rows per big tab are pure renames (not just snake_case to camelCase), e.g. `ACCOUNT_NO` -> `agentCode`, `ADDR1` -> `address1` (compInfo), `ADDRESS_LINE1` -> `address1` (clientInfo), `BR_PHONE` -> `phoneNo`, `BRANCH` -> `branchName`, `COMPANY_REG_NO` -> `compBRN`, `GROSS_PREMIUM` -> `grossPrem`, `GST_PAYABLE` -> `gstPct`, `DEBIT_NOTE` -> `debitNoteNo`, `AMOUNT_PAYABLE_ROUNDED` -> `premDueRounded`, `CLASS` -> `productDesc` (ASP) / taxClientInfo class. RejectRisk has zero renames except the REMARKS grouping.

### 1.2 RejectRisk code mapping workbook - `T12_final_code\RejectRisk\RejectRisk\RejectRisk_XSD_Mapping_sheet.xlsx` (and identical `Mapping_sheet\RejectRisk_Jrxml_Table_Column_Mapping.xlsx`, 12,651 bytes, modified 2026-09-01 11:34) [WB]
Two sheets:
1. `Jrxml-Table-Column Mapping` - 21 data rows, 7 columns: A Jrxml Name, B Referenced From (Parent JRXML), C Oracle Table or View Name, D Column Name, E Code XSD Mapping Path, F XSD Attribute Name, G XSD Group (Identified).
   - Unlike the master tab this one DOES carry the DB side: tables `CMUW_PL_REFER_RISK` (REJ_TYPE, REASON_CODE, REMARKS, CREATED_BY/DATE, REVOKE_*, REVOKED_DURATION, UPDATED_*), `CPGE_VI_PARTNERS_ALL` (NAME_EXT, ID_TYPE1, ID_VALUE1, ID_TYPE2, ID_VALUE2), `CMGE_CODE` (CODE_DESC looked up by CAT_CODE `PLREJ_RISK_TYPE` / `PLREJ_REASON` / `PLREJ_REVOKE_DUR`), pseudo-column ROWNUM, and `CUSTOMER.RPGE_RPT_TEMPLATE.DATA_CLASS` from the ReportHeader subreport.
   - No status column (no done/pending). Status is implicit: every row has a path.
2. `Jrxml Call Hierarchy` - 2 rows: level 1 `RejectRisk.jrxml` (/reports/AGIC/UW/Plines/RejectRisk/RejectRisk) and level 2 `ReportHeader.jrxml` (/reports/AGIC/UW/Generic/ReportHeader.jrxml, title band, expression `$P{SUBREPORT_DC} + "ReportHeader.jrxml"`); parameters passed: SUBREPORT_DIR, pReportTitle, pCreatedDateFrom, SUBREPORT_DC, pRejType, pCreatedDateTo, pIdNo, reportName, pCustomerName, pReasonCode.
- Verified [RAN path check]: all 21 "Code XSD Mapping Path + attribute" values exist as leaf paths in v0.1.6.
- Difference between sheet and output [WB+RAN]: the sheet says REMARKS goes to `...reportData/prntDetails/companyType/companyDesc/remarks`; the real response (and the code, KT 04 4.1) writes it at `...prntDetails/transDetList/companyType/companyDesc/remarks`. Both paths are legal in the XSD; the sheet and the output disagree. [CONFIRM which one the print engine reads]

### 1.3 The three PowerShell scripts in `RejectRisk\Mapping_sheet` [read as text]
- `Apply_MasterMapping.ps1` (2,355 B): opens the master workbook through Excel COM (read-only), finds tab `RejectRisk`, loops rows from 6, builds a dictionary: key = column H (Attributes), value = (I XSD Attribute Name, J XSD Group). Then loads `RejectRisk_Jrxml_Table_Column_Mapping.csv`, and for each row looks up its `Column Name`; on a hit it OVERWRITES the CSV columns `XSD Attribute Name` and `XSD Group (Identified)` and prints `MATCH ... (was: old)`; otherwise `NO MATCH ... (kept existing F/G)`. It prints Matched/Unmatched counts and writes the CSV back. (The comment says "Alias1's CSV (source of truth for the xlsx)" - [INFER] there were two Claude/AI "aliases" or two working sessions.)
- `Build_RejectRisk_Xlsx.ps1` (9,885 B): builds the xlsx by hand (raw zip + SpreadsheetML, forward-slash entry names so Excel opens it) from the mapping CSV and the hierarchy CSV; frozen header, autofilter, column widths by content.
- `RejectRisk_Jrxml_Hierarchy.csv`, `RejectRisk_Jrxml_Table_Column_Mapping.csv` (7,336 B): the sources.
So the sheet was produced in two steps: (1) jrxml analysis gave Jasper field -> DB table.column -> first-guess XSD name; (2) the script replaced the XSD name/group with the decision already recorded in the master sheet. The timeline in file times: curl files 31 Aug 18:02, scripts 31 Aug 21:24-21:30, xlsx 1 Sep 11:34, JSON samples 1 Sep 10:37-10:42, git commit "Fix RejectRisk data mapping and align with Pline.xsd v0.1.6" 1 Sep 21:42 (commit 0d715fbad by GS) [GIT].

### 1.4 `RejectRisk 2.xlsx` and `ASPEndtSchedule.xlsx` - `Documents\plines\Plines\` (both 2026-06-15) [WB]
Not XSD maps: jrxml INVENTORY sheets. Columns: No, Primary Report, Sub Report Count, Sub Report, Sub SQL Report Count, SQL Queries Count, Table Name Referred, Common Report (+count), Stored Procedures & Function Name (+count), Type. `RejectRisk 2`: 1 report, `ReportHeader (Not Found)` in the dump, 1 main query, tables CMUW_PL_REFER_RISK, CPGE_VI_PARTNERS_ALL, CMGE_CODE. `ASPEndtSchedule`: 10 report rows (AddMemberLandscape, DeletionMemberLandscape, EndtDocument, EndtCover, EndtSchedule, EndtDocument2, EndtTaxDoc, ServiceTaxInvoice, TransitionalInvoice, TransitionalInvoice2) with the table names and DB functions, plus a block "Stored Procedures And Functions" with Found / Not Found flags (PG_RPGE_UTILS.FN_GET_COMP_BRN_NUMBER and FN_GET_EINVOICE_QR_IND "Not Found"; Fn_Get_SST_Rollout_Date, FN_GET_EINVOICE_PRINT_ENDT_IND, PME_API.GET_POLICY_REF, FN_RPPM_GET_SIGNATURE(_NAME) "Found"). These were the scoping step before mapping (June, before the July master sheet).

### 1.5 `Template12_XSD_Mapping_Final.xlsx` (T12_final_code\Tempalte12_details and \Template_12_MAIN, identical 74,269 B, modified 2026-08-19 10:46) [WB]
One sheet, 1944 data rows, 6 columns: A JRXML Name, B Referenced From (Parent JRXML), C Oracle Table / View Name, D Column Referenced / Fetched, E Code XSD Mapping Path, F Code XSD attributes. It has an Excel table object; NO status/done column.
- Meaning: A = the jrxml (names end in `.jrxml.data`, i.e. read from the `*_jrxml.data` dump files); B = parent (661 rows PolicyDocument_jrxml.data, 396 Landscape_jrxml.data, 318 PolicyDocument2_jrxml.data, 251 "(Main Report - no parent)", rest sub-levels such as CIPrint*.jrxml.data); C/D = DB table + column the jrxml selects; E = the slash-form XSD path the CODE writes (`transactions/transaction/.../transDet/<leaf>`); F = the Java-side attribute name.
- Size: 75 distinct jrxml, ~67 distinct tables (86 spellings; `customer.` prefix variants), 241 distinct DB column names, 228 distinct XSD paths, 231 distinct XSD path+attribute pairs. 42 rows have a multi-value B cell (a shared subreport used under several parents). No duplicate rows after accounting for that.
- [RAN] 228 of 228 distinct paths exist as leaf paths in v0.1.6 (0 invalid). Rows by XSD container: transDet 1226, endorsementDetails 328, clientInfo/address 123, reportParameters 80, cover (under riskList/risk) 52, agentInfo 34, benefit 21, taxClientInfo 20, staxPcts 18, branchInfo 11, nominee 10, companyDesc (transDetList/companyType) 7, compInfo/address 6, compInfo 5, diagnosis 3.
- Coverage of the XSD [RAN]: 228 of 1332 leaf paths; transDet 164 of 634; reportParameters 10 of 67; endorsementDetails 7 of 50. That is the "reached" part; the master tab names 474 distinct XSD attributes for PolicyDocument, so this code sheet is a subset (what was actually coded and tested), not the whole of Template12 [INFER].
- F differs from the path's last element in 10 distinct cases = deliberate Java-name to XSD-name translation: `refNo -> refno`, `addressLine1/2/3 -> address1/2/3`, `postcode -> postCode`, `addr1/2/3 -> address1/2/3` (compInfo), `phone -> phoneNo`, `grossPremium -> grossPrem`; plus `COV_LOAD_AMT -> sumLoading` (DB column to sum variable).

### 1.6 `xsd_check.xlsx` (Documents, 2026-08-04, 25,945 B) [WB]
One sheet, header on row 5 (rows 1-4 empty), 551 data rows, 3 columns: A Attributes, B XSD Attribute Name, C XSD Group (Identified). It is the extraction of columns H/I/J of the master tab. Row count (551) and distinct XSD names (474) both equal the master `PolicyDocument` tab, so [INFER] it is a snapshot of that lookup used to check/seed Template12 (same idea as the RejectRisk `Apply_MasterMapping.ps1` lookup). Three names are not in v0.1.6: `perilsCount`, `policyHolderName`, `prem` (see 1.1).

### 1.7 Tracking workbooks (test/coverage status, not XSD) [WB]
These hold the real "status" columns.
| File | Sheets / rows | Columns | Status values |
|---|---|---|---|
| `Documents\ASP_contract_id_list.xlsx` (2026-08-26) | EndtDocument 9 rows, EndtDocument2 20 rows | Subreport, Path/Notes (the jrxml gate, e.g. `showCoverNote=Y`), Completed/FiredWithRealValue, Contract ID | EndtDocument: 6 "Done", 1 "Done (dispatch-only, static)", 1 "Done (dispatch-only, SELECT 1 FROM DUAL...)", 1 "Done (static content only)". EndtDocument2: 4 Done, 1 "Done (dispatch-only)", 1 "Gate fired (not confirmed real value)", 3 "Not fired / gate not met", 11 blank (not yet tried) |
| `T12_final_code\Tempalte12_details\contract_id_list.xlsx` (2026-08-19; 12 `.bak` versions from 10-17 Aug) | PolicyDocument 70, PolicyDocument2 51, Landscape 17 | Subreport, Path/Notes, Completed/FiredWithRealValue, Contract ID | PD: 67 Done + Done(Dual) x2 + "never execute (deadcode)"; PD2: 49 Done + 2 "Not found in Jasper"; Landscape 17 Done. (Blocked->Done was updated by hand per CODE_CHANGES_LOG) |
| `contract_reports_mapping.xlsx` (two versions: Tempalte12_details 2026-08-21, 50 rows; Template_12_MAIN 2026-09-07, 47 rows) | 1 sheet | Contract ID, Reports Fired (subreports tagged [PD]/[PD2]/[LS]), StaxNo | none; it is "which contract fires which subreports" |
| `Documents\New folder\Template12\Template12_070521_ContractId_Mapping.xlsx` (2026-10-08) | 42 rows | Main Report, Sub Report Lv1, Lv2, Conditions, Input, Values, Status, Set, Report Type, Aggregation/Query | Status text: 20 "Not satisfiable..." (e.g. no product-070521 contract has a nominee), 19 "Satisfied", 2 "Partially" |
| `Documents\New folder\Template12\Template12_Report_Structure.xlsx` (2026-10-07) | Report Structure 150 rows; Summary; Distinct Reports 127; Legend | Main Report, Sub Lv1-Lv4, Testing Status, Query Creation, Response Status | The three status columns exist but are EMPTY (template). Summary: 3 main reports, 127 distinct reports (74 Template12-specific, 14 generic, 36 common), deepest level 4, 3 files not found in the dump |
| `Documents\Template12_*Hierarchy*.csv` (2 files, July) | Landscape 1,613 B, PolicyDocument2_Subreport 5,097 B | the jrxml call tree | - |
Driver for the T12 tracker generation: `contract_id_and_responses\_build.ps1` / `_build_xlsx.ps1` (10 Aug) turn `_PD.csv`, `_PD2.csv`, `_LS.csv` into `contract_id_list.xlsx` (COM version and a raw-zip version when Excel was not usable). `contract_id_and_responses\responses` is empty.
[CONFIRM] There is no ASP counterpart of `Template12_XSD_Mapping_Final.xlsx`; ASP's XSD decisions live only in the master `EndtDocument` tab and in code.

---------------------------------------------------------------------

## 2. Reconstructed mapping PROCESS (from Jasper field to XSD leaf)

Reconstruction from the workbooks, scripts, git, and file dates. [INFER] where marked.

1. Scope the jrxml tree. Inventory sheets (`RejectRisk 2.xlsx`, `ASPEndtSchedule.xlsx`, June) and call-hierarchy CSVs (T12, July; RejectRisk 31 Aug) list main report -> subreports (levels 1-4) with tables and DB functions.
2. Harvest the Jasper fields. For each jrxml (taken from the Jasper dump `*_jrxml.data`) collect fields/variables and the table.column they read. Output: `Template12_XSD_Mapping_Final.xlsx` columns A-D (and RejectRisk csv columns A-D).
3. Decide the XSD home. The Pline XSD (v0.1 of 6 Jul, built "in Bond style") already lists every field in the master sheet: columns E/H (Jasper attribute) -> I (XSD element) -> J (XSD group). Rule recorded in the comments: reuse a Bond/Common leaf when the name matches ("Reuses existing ... leaf"), otherwise add a Pline leaf placed in the group Bond would use ("New Pline field ... Placed in X per Bond conventions"), otherwise mirror Bond's known mapping ("Mirrors Bond: field maps to city in address, buyerInfo, companyDesc"). Types: DB NUMBER -> xs:decimal, DATE/Timestamp -> xs:dateTime, text -> xs:string, counts -> xs:integer.
4. Carry the decision into the code sheet. For RejectRisk, `Apply_MasterMapping.ps1` copies I/J from the master tab into the per-report csv keyed on the Jasper column name. For Template12 the per-report sheet (E/F) was built with the slash-form full path.
5. Implement. Java payload classes mirror the XSD field names (`PlineTransDet`, `PlineRisk`, `PlineCompanyDesc`...), the service sets the value (e.g. `risk.setRejectRiskTypeCode`), dates go `Timestamp -> LocalDateTime`, numbers `-> BigDecimal` (commit 6961abbf9 "Align Pline requestpayload types with XSD", 2026-08-08, by Shinge, not GS) [GIT].
6. Call the API with a curl/Postman request (section 4), save the JSON-wrapped response as `<contractId>.xml` and check the filled leaves against the sheet. For Template12/ASP, record per subreport whether it fired "with real value" in the tracker xlsx (section 1.7).
7. Status tracking is therefore: (a) rows exist in the code sheet = mapped; (b) tracker xlsx "Done / Gate fired / Not fired / Blocked"; (c) `KNOWN_BUG_staxCount_eInvStatus.txt` and `CODE_CHANGES_LOG.txt` (19-20 Aug) record the two real mapping/merge defects found while testing (staxList dropped by `mergeNonNullFields`; `eInvStatus` clobbered by a blank default; fix scoped to the Template12 merge, shared handler left untouched because 11+ templates use it).

Limits: no sheet has an "unmapped/not in XSD/extra" column. Those four counts in section 3 are computed by me by comparing sheets, XSD paths and real responses.

---------------------------------------------------------------------

## 3. Counts: mapped / unmapped / not in XSD / extra

Method [RAN]: I enumerated all element paths of Pline.xsd+Common.xsd from `transactions` (1410 paths: 78 containers, 1332 leaves, 1072 distinct element names), then (a) matched sheet paths/names against them, (b) extracted every element name from the newest real response of each module and split into "present with a value" vs "present but empty".

### 3.1 RejectRisk
| Measure | Count |
|---|---|
| Master tab rows (Jasper -> XSD decisions) | 21 (17 transDet, 2 risk, 1 reportData/dataClass, 1 companyDesc/remarks) |
| Code mapping rows | 21, all 21 paths exist in v0.1.6 |
| "Existing in XSD = N" | 0 |
| New leaf vs reused leaf vs mirrored | 16 / 4 / 1 |
| Mapped leaves actually non-empty in newest sample (15 Sep) | 19 of 21; empty: idNo2, idType2 (data, not mapping) |
| Unmapped Jasper items | `pCustomerName`, `pIdNo`, `pRejType`, `pReasonCode` (query filters only, not in XML), `SUBREPORT_DIR`, `SUBREPORT_DC` |
| "Not in XSD" but emitted by Java | reportParameters `pReportTitle` (has a value), `pCreatedDateFrom`, `pCreatedDateTo` (empty in the sample), `pShowOutPrintCovLetter`; transDet `leasingBranch`; and in current code `mainReportName`/`subReportName` |
| "Extra" element names in the 15 Sep response that the XSD has no element of that name | 165 distinct names, 148 of them `p...` reportParameters leaked from the shared Motor `ReportParameters` base class (+ `indicationList`, `indicationData`, `payments`, `payment`, `acodecd`, ... ). Pre-dates the whitelist (see 5.4) |
| XSD element names never emitted in that response | 50 |

Notable decisions (from the sheets):
- `REMARKS` -> `companyDesc.remarks` (not a transDet leaf). Reason in the comment: "Mirrors Bond: field maps to remarks in companyDesc, needBasedAssessmentDet, questionnaireDet". One companyDesc per data row, index-matched to the transDet (commit message of 0d715fbad) [GIT].
- `REJECT_RISK_TYPE_CODE`, `REJECT_RISK_TYPE` -> `transDet/riskList/risk/rejectRiskTypeCode` and `rejectRiskType` (nested, not flat; before the 1 Sep fix the code had flat "type/description") [GIT].
- `ID_TYPE_1/ID_NO_1/ID_TYPE_2/ID_NO_2` -> `idType1/idNo1/idType2/idNo2` (before: `idType/idNo`) [GIT].
- Irregular casing kept exactly as the XSD: `reasoncode`, `updatedby`, `updateddate` (the three "Reuses existing Bond/Common leaf" lookalikes with lower-case tails) plus `createdBy` [WB].
- `ROWNUM` pseudo-column -> `rownumber` (xs:decimal) [WB].
- `CODE_DESC` is read three times from `CMGE_CODE` with three `CAT_CODE` values -> `rejectRiskType`, `reasonCodeDescription`, `revokedDurationDescription` [WB].
- `DATA_CLASS` is not in the RejectRisk query at all; it comes from the ReportHeader subreport/`RPGE_RPT_TEMPLATE` and goes to `reportData/dataClass` [WB].

### 3.2 ASP (master tab `EndtDocument`; no code sheet exists)
| Measure | Count |
|---|---|
| Master rows | 474 -> 433 distinct XSD attribute names, 432 found by exact name (the miss is `policyHolderName` vs `policyholderName`) |
| Container split of the 474 rows (column J) | transDet 291, endorsementDetails 42, cover(riskList) 23, staxPcts 17, supplierInfo 11, buyerInfo 11, reportParameters 11, companyDesc(transDet) 10, compInfo 9, risk 8, address(compInfo) 6, nominee 6, taxClientInfo 5, agentInfo 5, branchInfo 4, address(clientInfo) 4, clientInfo 3, diagnosis 3, benefit 3, reportData 2 |
| New / reused / mirrored | 263 / 59 / 152 |
| Types | 237 decimal, 209 string, 25 dateTime, 3 integer |
| Names with a non-empty value in newest ASP sample with most data (`35112689_1.xml`, 24 Sep) | 181 of 433 (41.8%); 432 of 433 appear as elements |
| Populated names not in either ASP/T12 sheet | 34, almost all request-driven (templateDetails fields, `p...` flags) or spelling variants (`cityDescp`, `compName`, `mstPolName`, `polno`) [CONFIRM] |
| "Extra" element names emitted that XSD lacks | 220 distinct, 200 are `p...` reportParameters from the base class; the rest include `indicationList/Data`, `payments`, `acodecd`, `rcodedesc`, `leasingBranch`, `policyletterind`, `stamp`... |
| Emitted with a VALUE but not in XSD | `pcompanyParamType`, `pContractId`, `pendtNo`, `pissueDate`, `prefNo`, `pshowEndtSchedule`, `pUserID`, `pversionNo` |
Notable ASP decisions: `CLASS` -> `productDesc` (not the taxClientInfo `class` leaf); `NAME` -> `nameExt`; reports with the same field map to the same leaf (e.g. `ACCOUNT_NO` -> `agentCode`). Landscape reports (AddMember/DeletionMember) are separate reportData blocks (commit cc6a1b84d "Restructure AspEndtDocumentService Landscape rows into separate PlineReportData entries") [GIT]. ASP EndtDocument2 and Landscape services were merged into the one `EndtDocument` bean (`AspEndtDocument2Service` @Service commented out) [CODE].

### 3.3 Template12 (master tab `PolicyDocument` + code sheet `Template12_XSD_Mapping_Final.xlsx`)
| Measure | Count |
|---|---|
| Master rows | 551 -> 474 distinct XSD names (3 not in v0.1.6: perilsCount, policyHolderName, prem) |
| Container split (col J) | transDet 364, reportParameters 31, endorsementDetails 22, cover(risk) 21, staxPcts 18, compInfo 16, buyerInfo 11, supplierInfo 11, companyDesc(transDet) 10, benefit 9, address(clientInfo) 7, taxClientInfo 5, risk 5, agentInfo 5, branchInfo 4, diagnosis 3, address(compInfo) 3, reportData 2, proviso(risk) 2, nominee 2 |
| New / reused / mirrored | 292 / 79 / 180 |
| Types | 289 string, 228 decimal, 31 dateTime, 3 integer |
| Code sheet rows | 1944 (228 distinct paths, all valid in v0.1.6; 164 of 634 transDet leaves; 10 of 67 reportParameters) |
| Names with a value in newest sample (`29441234.xml`, 8 Oct) | 159 distinct; 130 of the 474 master names; 94 of the 225 code-sheet leaf names |
| Populated names in neither sheet | 28 (request/template fields, `addr1-3`, `phone`, `companyRegNo`, `asstFeeDiff`, `gossPremB4Promo`, `IDNo`, `provisoCode/Desc/Narr`, `mainReportName`) [CONFIRM] |
| Emitted but not in XSD | 9 distinct names only: `leasingBranch`, `mainReportName`, `subReportName`, and the 6 whitelisted parameters `pCreatedDateFrom`, `pCreatedDateTo`, `pReportTitle`, `pshowEndtSchedule`, `pShowOutPrintCovLetter`, `pShowPDPANotice` (matches KT 04 section 3: whitelist works) |
| XSD element names never emitted | 1 |
Notable T12 decisions: `CP_VERSION`, `ACTION_CODE` etc. are mapped the same under several subreports (CIPrint*, Coverage*, Nomination*) to one transDet leaf; the BMW/Elite/POSM/ATC CIPrint variants all write into the same transDet leaves (merged with `mergeNonNullFields`); `staxList` carried as `staxPcts` rows (staxCount lives only inside staxList.staxPcts; bug fixed 19 Aug); `COV_LOAD_AMT` -> `sumLoading`; REMARKS-like text for company blocks goes to `transDetList/companyType/companyDesc` (7 rows) [WB/CODE].

---------------------------------------------------------------------

## 4. Real request payload shape (from the curl / request files)

Endpoint (all three): `POST <redacted-host>/gi/documentprintdata/api/opus/GenerateReport`, header `Content-Type: application/json`, no Authorization/cookie header in any file I read (host is `localhost:8080` in the RejectRisk curl, otherwise `<redacted>`). Body = `{ "metadata": {...}, "filterCondition": {...} }`. `metadata.reportName` and `filterCondition.reportName` both carry the report path = the `@Service` bean name that selects the Java service. Response = `{"status":"SUCCESS","data":"<transactions>...XML...</transactions>","timestamp":"..."}` (the XML is NOT JSON-escaped, so the saved `.xml` files are not valid JSON; I extract by string position).

### 4.1 RejectRisk (`API_Crul_1357766.txt`, `API_Crul_2439275.txt`, `Nisarga_req_res\RejectRisk\RejectRisk_RequestPayload.txt`)
- metadata keys (19): addressline1, addressline2, email, name, senderMail, mailSubject, copyType, batchConfigId, boResId, channel, dataClass, disallowCopy, docType, linkLocation, outputFormat, outputPath, reportName, templateId, templateName. Values are placeholders ("Address Line 1", "recipient@example.com", copyType ORIGINAL, channel WEB, docType PDF, templateName RejectRisk, templateId "1"). These echo into `templateDetails`/`mailingDetails` of the response.
- filterCondition keys (8): `reportName`, `pReportTitle` ("PLINES : REJECT RISK REPORT"), `pCreatedDateFrom` (dd/MM/yyyy), `pCreatedDateTo` (dd/MM/yyyy), `pCustomerName`, `pIdNo`, `pRejType` (code, e.g. `C`), `pReasonCode` (e.g. `09`).
- The two curl files differ only in the 4 values (dates, customer name, id number) - test data chosen from the DB to return rows. The 15 Sep request reuses the second one's filter.

### 4.2 ASP (`Nisarga_req_res\ASP\ASPEndtSchedule_RequestPayload.txt`, 18 Sep; `new_springboot_upgraded\ASP\contract_id _list.txt` 27 Sep)
- metadata: same keys plus `mergeStaticPdfFiles: []`; reportName `AGIC/UW/Plines/ASPEndtSchedule/EndtDocument`, templateId 23, templateName `ASPEndtSchedule_EndtDocument`, copyType `O`, channel EMAIL, docType PDF.
- filterCondition keys (21): `reportName`, `pContractId`, `pEndtNo`, `pRefNo`, `versionNo`, `checklistCopy`, `printSeries`, `copyType`, `companyParamType`, `pPolicyNo`, `pType` (e.g. IO), `pAtceInd`, `pUserID`, `showCoverNote`, `showCover`, `showEndtSchedule`, `showComplaint`, `showDebitNote`, `showTaxInvoice`, `pFirstPageNumber`, `pReportTitle`.
- The text file `contract_id _list.txt` is the "which contract exercises which subreports" note; each entry lists ContractId, EndtNo, RefNo, versionNo, checklistCopy, companyParamType, PolicyNo (12 numbered cases; one contract (27275348) covers all EndtDocument subreports and AddMember landscape).

### 4.3 Template12 (`Template_12_MAIN\API_CURL.txt`, 24 Aug; `Nisarga_req_res\Template12\Template12_req.json`, 15 Sep)
- metadata: reportName `AGIC/UW/Plines/PolicySchedule/Template12/PolicyDocument`, templateName `Template12PolicyDocument`, copyType `IO`; everything else empty strings, templateId/batchConfigId/boResId numeric strings (the 15 Sep file uses large ids, the 24 Aug file 12/0/0).
- filterCondition keys (20): `reportName`, `pContractId`, `pPolicyNo`, `versionNo`, `companyParamType` ("08"), `checklistCopy`, `showCoverNote`, `showComplaint`, `showScheduleStd`, `showDebitNote`, `showTaxInvoice`, `showPaymentInfo`, `printSeries`, `copyType`, `ShowOutPrintCovLetter` (capital S, as written), `pCertNoFrom`, `pCertNoTo`, `pCertInd`, `pTaxInvNo`, `pGstTaxInvoiceNo`. The two T12 files differ in which of pCertNo*/pTaxInvNo/pGstTaxInvoiceNo are filled (certificate vs tax-invoice scenarios).
- Case traps visible here: `ShowOutPrintCovLetter` (request) vs `pShowOutPrintCovLetter` (whitelisted XML name) - KT 04 3.1 notes the request key's setter name is not the whitelisted one.

---------------------------------------------------------------------

## 5. Validation of the newest real responses against Pline.xsd v0.1.6 [RAN]

### 5.1 Method
`V.java` (scratch): `SchemaFactory.newInstance(W3C_XML_SCHEMA_NS_URI).newSchema(File Pline.xsd)` (Common.xsd resolves via `xs:include` because both are file-based), a `Validator` with an `ErrorHandler` that COLLECTS every error (does not stop at the first), then classifies messages. Compiled and run from a directory containing only the `.class` files (`xsdcheck2\run`), schema copies in `xsdcheck2\xsd`, sample file paths passed as arguments. The `data` value is extracted from the JSON wrapper. JDK 26.0.1, default JDK Xerces.
Error counting, so the numbers are comparable with KT 04: Xerces reports each bad empty value TWICE (`cvc-datatype-valid.1.2.1` + `cvc-type.3.1.3`). "Total" = both; "Logical" = total minus the `cvc-type.3.1.3` duplicates (one per bad value or structure problem).
Important limitation [RAN, known Xerces behaviour]: after the first unexpected element inside one parent, Xerces stops validating that parent's remaining children, so "unexpected element / no child expected" counts are LOWER BOUNDS (hidden errors appear only after the first is fixed - see `pchecklistCopy` in 5.4). Empty-typed-value counts are exact.

### 5.2 Results (file dates are file-modified dates; the response `timestamp` inside the file agrees)
| Module | File | Modified | Size (B) | Total errors | Logical | Empty-typed values | dd/MM/yyyy dateTime | Structure/unknown |
|---|---|---|---|---|---|---|---|---|
| RejectRisk (old, KT 04) | `1357766.xml` | 2026-09-01 10:42 | 71,707 | **929** | 473 | 454 | 2 | 17 |
| RejectRisk (old, KT 04) | `2439275.xml` | 2026-09-01 10:37 | 121,003 | **1766** | 894 | 867 | 5 | 22 |
| **RejectRisk NEWEST** | `Nisarga_req_res\RejectRisk\RejectRisk_ResponsePayload.txt` | 2026-09-15 15:26 | 121,294 | **1754** | 888 | 861 | 5 | 22 |
| ASP | `5490878_delete.xml` | 2026-09-27 11:57 | 466,957 | 7,458 | 3,771 | 3,650 | 37 | 84 |
| ASP | `27275348.xml` | 2026-09-23 21:05 | 603,250 | 9,150 | 4,637 | 4,471 | 42 | 124 |
| ASP | `35112689_1.xml` | 2026-09-24 19:40 | 1,780,875 | 30,763 | 15,508 | 15,129 | 125 | 254 |
| Template12 | `New folder\Template12\29441234.xml` (NEWEST of all) | 2026-10-08 09:58 | 2,646,112 | 41,061 | 20,692 | 20,174 | 195 | 323 |
| Template12 | `Nisarga_req_res\Template12\6549578.xml` | 2026-09-10 16:51 | 1,884,208 | 28,772 | 14,505 | 14,167 | 100 | 238 |
| Template12 | `new_responses\40795067_checklist_N_STI_T4.xml` | 2026-09-07 17:57 | 563,923 | 7,672 | 3,867 | 3,770 | 35 | 62 |
The old RejectRisk numbers (929 / 1766) were reproduced exactly - the method matches KT 04.
All 12 ASP files: total errors 3,204 - 30,763 (sum 107,188). All 53 T12 `new_responses` files: 1,710 - 47,928 (mean about 15,360). None of the 66 ASP/T12 files I ran in bulk (12 ASP, 53 T12 `new_responses`, 1 ASP copy under Nisarga_req_res; full list in `xsdcheck2\all_results.txt`) validates; error count scales with the number of transDet blocks (each block has ~620 empty decimal elements). 96-98% of logical errors in every file are "empty element for a decimal/dateTime/integer/boolean".

### 5.3 Top 10 distinct error types, by module (counts per file; the duplicate type `cvc-type.3.1.3` is listed once at the end of each block)
RejectRisk NEWEST (15 Sep): decimal empty 741; dateTime empty 96; integer empty 18; boolean empty 6; dateTime dd/MM/yyyy-style 5; `no child expected: address` 4; `no child expected: IDNo` 2; `no child expected: benefit` 2; `no child expected: provisoList` 2; `unexpected element companionName` 2 (also 2 `occpDescp`; singles: `copyTypeList`, `indicationList`, `address1`, `city`, `clientType`, `mailingDetails`, `pendtNo`, `policyletterind`). Duplicates cvc-type.3.1.3: 866. 20 distinct types.

ASP (3 files, same ranking everywhere): decimal empty (3,137 / 3,823 / 13,129); dateTime empty (413 / 532 / 1,648); integer empty (73 / 83 / 235); dateTime dd/MM/yyyy-style (37 / 42 / 125); boolean empty (27 / 33 / 117); `no child expected: address` (20 / 36 / 44); then `IDNo`, `benefit`, `provisoList`, `unexpected companionName` (9 / 11 / 39 each), `address1`, `city`, `clientType` (5 / 9 / 11), plus 1 each of `policyletterind`(x5/9/11 by file), `occpDescp`, `copyTypeList`, `indicationList`, `mailingDetails`, `pendtNo`, `pproduct`, `xshowStaxInvInd` in some. 20-21 distinct types.

Template12 (3 files; 8 Oct file in front): decimal empty 17,151 (8 Oct) / 12,259 (10 Sep) / 3,274 (7 Sep); dateTime empty 2,291 / 1,533 / 398; integer empty 598 / 273 / 71; dd/MM/yyyy-style 195 / 100 / 35; boolean empty 134 / 102 / 27; `no child expected: provisoList` 116 / 63 / 16; `unexpected occpDescp` 76 / 29 / 7; `no child expected: IDNo` 40 / 34 / 9; `no child expected: benefit` 40 / 34 / 9; `unexpected companionName` 40 / 34 / 9; then `address`, `copyTypeList`, `address1`, `city`, `clientType`, `mailingDetails`, plus (8 Oct only) `mainReportName` and `pchecklistCopy`. 19-20 distinct types.
Reading the structure types (cross-ref KT 04 5.2) [INFER on which class each belongs to]: `address` after other fields (agentInfo/branchInfo/compInfo/taxClientInfo: address must be first), `IDNo` (nominee order), `benefit` (benefits order), `provisoList` (cover order), `companionName`/`occpDescp` (transDet/risk order), `copyTypeList` (templateDetails order), `mailingDetails` (transaction order: XSD wants reportParameters, mailingDetails, templateDetails, transactionDetails; Java writes templateDetails first and reportParameters last), `address1/city/clientType` (buyerInfo/supplierInfo/clientInfo order).

### 5.4 What improved and what regressed
RejectRisk (15 Sep vs 1 Sep, same filter as `2439275.xml`): total 1766 -> 1754, logical 894 -> 888. The whole change is -6 empty decimal elements because `staxPrem`, `staxCount` and `asstFee` now have values in the two placeholder `staxPcts` blocks (a text diff of the two files shows only that, plus new empty `leasingBranch` in each transDet and new empty `pCreatedDateFrom`, `pCreatedDateTo`, `pShowOutPrintCovLetter` in reportParameters). Same 20 error types; no structural error was fixed. So: effectively NO improvement. A value such as `staxPrem` = 7497.1 inside a RejectRisk response looks like leaked default/test data [CONFIRM; I could not find "7497" in src/main]. `leasingBranch` is a Java-only leaf (KT 04 5.1).
The 15 Sep file has no `mainReportName/subReportName`, no trailing ReportHeader reportData, no whitelisted reportParameters (219 leaves: 148 `p...` names the XSD does not know). Commits after it: GS `cc6a1b84d` 2026-10-05 (RejectRisk date-range filter fix), Shinge `bca355525` 2026-10-06 (reportParameters `@JsonIncludeProperties` whitelist + `Policyletterind` field), `e74bc0d5c` 2026-10-07 and `861f0666c` 2026-10-08 (mainReportName/subReportName). The git status also lists UNCOMMITTED edits in RejectRiskHandler/RejectRiskService/application.properties. So no real RejectRisk response from the current code exists: [CONFIRM] re-capture it. What to expect from it [INFER, from the T12 8 Oct result]: reportParameters shrinks to the 73-name whitelist, `Policyletterind` no longer an error, `pendtNo`/`indicationList` gone, `mainReportName`/`subReportName` now NEW unexpected elements, and the ~860 empty-typed errors unchanged.

Template12 (the only module with a post-whitelist sample; compare 10 Sep -> 8 Oct, different contracts so compare error TYPES not totals):
- Improved (types that disappeared): `policyletterind` (now `Policyletterind`), `pendtNo`, `indicationList`, `xshowStaxInvInd` (case fixed by 10 Sep: present in the Aug 20 file, absent 10 Sep), reportParameters cut from hundreds of leaves to exactly 73 (the whitelist), `address1/city/clientType` repeated 5 -> 1.
- Regressed or newly visible: `mainReportName` (new, not in v0.1.6) and `pchecklistCopy` (an ORDER error in reportParameters that was hidden behind `pendtNo` before). Six whitelisted names are not in the XSD (`pCreatedDateFrom/To`, `pReportTitle`, `pshowEndtSchedule`, `pShowOutPrintCovLetter`, `pShowPDPANotice`) but are masked by the first unexpected element in reportParameters.
- Unchanged: all empty-typed errors, dd/MM/yyyy dates, the order errors in cover, nominee, benefits, risk, transDet, templateDetails, transaction.
- Also a size effect: the 8 Oct file collapses everything into ONE reportData with 40 transDet; the 10 Sep file had 5 reportData (1 + 4 landscape blocks). So the multi-reportData layout was changed back and forth.

ASP: all 12 ASP samples (23-27 Sep) pre-date `bca355525`, so they still show `policyletterind`, `pendtNo`, `pproduct`, 263 empty reportParameters leaves and have NO `mainReportName`. They are the evidence for "before"; there is no ASP "after" sample [CONFIRM re-capture].

Bottom line vs earlier finding: the old statement "real output does not validate" is still true for every module, and the dominant cause (empty decimal/dateTime/integer/boolean elements, ~97% of errors) is unchanged in the newest files. The only measurable structural improvement is in the 8 Oct Template12 payload.

---------------------------------------------------------------------

## 6. reportData names and element counts in the real responses

### 6.1 What the real files contain [RAN, StAX parse]
Only ONE real response contains `mainReportName`/`subReportName`: Template12 `29441234.xml` (8 Oct) with `mainReportName` = `Template12PolicyDocument` and an EMPTY `subReportName`, in its single reportData. None of the 12 ASP files and none of the 54 T12 `new_responses`/10 Sep files have either element (they pre-date commit e74bc0d5c). The only other responses in Documents with `mainReportName` are AHI/Template13/Template16 (other modules, out of scope).

Element counts per response (elements = all XML elements; "non-empty" = leaf elements with text):
| File | reportData | transDet | elements | leaf elements | non-empty leaves |
|---|---|---|---|---|---|
| RR `1357766.xml` (1 Sep) | 1 | 1 | 1,421 | 1,354 | 31 |
| RR 15 Sep response | 1 | 2 | 2,309 | 2,211 | 55 |
| ASP `5490878_delete.xml` | 5 | 9 | 9,677 | 9,270 | 477 |
| ASP `27275348.xml` | 9 | 11 | 12,675 | 12,106 | 736 |
| ASP `35112689_1.xml` | 11 | 39 | 36,913 | 35,479 | 1,835 |
| ASP `35112689.xml` | 15 | 18 | 20,632 | 19,701 | 1,096 |
| ASP `2898648.xml` | 10 | 13 | 14,654 | 14,002 | 715 |
| ASP `37568698.xml` | 7 | 13 | 13,667 | 13,093 | 664 |
| ASP `35427695.xml` | 4 | 6 | 6,729 | 6,438 | 334 |
| ASP `2825280.xml` | 2 | 5 | 5,230 | 5,020 | 256 |
| ASP single-block files (`10892721`, `35063101`, `27275348_1`, `37568698_1`) | 1 | 4-5 | 4,028-4,869 | 3,874-4,686 | 141-234 |
| T12 `29441234.xml` (8 Oct) | 1 | 40 | 43,911 | 42,350 | 3,760 |
| T12 10 Sep `6549578.xml` | 5 | 34 | 32,709 | 31,494 | 1,890 |
| T12 `new_responses\6549578_id.xml` (20 Aug) | 1 | 33 | 30,010 | 28,994 | 1,754 |
| T12 `40795067_checklist_N_STI_T4.xml` | 1 | 9 | 8,699 | 8,379 | 452 |
About 95%+ of leaf elements are empty in every file (e.g. 8 Oct T12: 38,590 of 42,350). Each ASP landscape reportData is about 1,170-1,200 elements (1 transDet); the first ASP reportData (master) is 2.9k-24.7k elements.
ASP sample structure (e.g. `5490878_delete.xml`): reportData #1 = master (5 transDet, 4,533 elements), #2-#5 = one landscape row each (1,201 elements). `27275348.xml`: #1 master (3 transDet), #2-#9 section/landscape rows. `dataClass` is empty on every ASP/T12 reportData; in RejectRisk `dataClass` = `1`.

### 6.2 What the CURRENT code is meant to emit [CODE, HEAD 861f0666c] - not seen in any real sample
`PlineReportNames.tag()` fills `mainReportName` on every reportData that lacks one, from the `@Service` bean name: last segment, prefixed by the previous segment unless it already starts with it.
- RejectRisk: `RejectRisk`; data rows keep an empty `subReportName`; one extra trailing reportData `mainReportName=RejectRisk`, `subReportName=ReportHeader`, with `dataClass` (`PlineReportBlocks.reportHeader`).
- Template12 (`.../Template12/PolicyDocument`): `Template12PolicyDocument`; the T12 services do not call `setSubReportName` (grep at HEAD), consistent with the one real sample.
- ASP (`.../ASPEndtSchedule/EndtDocument`): four main names `ASPEndtScheduleEndtDocument`, `ASPEndtScheduleEndtDocument2`, `ASPEndtScheduleAddMemberLandscape`, `ASPEndtScheduleDeletionMemberLandscape`, each with sub names. Order in which `AspEndtDocumentService` adds them (code order): (1) master: main EndtDocument / sub `PrintSeriesFooter`; (2) `EndtCover` (EndtDocument); (3) `ComplaintRedress` (EndtDocument2, only for product 070412); (4) `EndtSchedule` (EndtDocument); (5) `DebitNote` rows (EndtDocument2); (6) `CustomerDtlsOfMasterPolicy_Debit`; (7) `EndtTaxDoc` rows; (8) `ServiceTaxInvoice`, then `ServiceTaxInvoiceEndtMT`, `ServiceTaxInvoiceEndtMTSub`; (9) `QRSection`; (10) `TransitionalInvoice`, `TransitionalInvoice2`; (11) landscape rows: `AddMemberLandscapeDispatch` (main AddMemberLandscape), `AddMemberLandscape_ASPInsured`, `AddMemberLandscape_ASPPremium`, `EndtParticularOfSponsorLandscape_ASP`, `EndtParticularOfNominationLandscape_ASP`, then `DeleteMemberLandscapeDispatch`, `DeleteMemberLandscape_ASP` (main DeletionMemberLandscape). Each row is added only when the matching subreport fires. [DERIVED from source order; not executed]

---------------------------------------------------------------------

## 7. Trimmed, anonymised real response skeletons
Produced by a small tool (`S.java`) from the real files: containers kept, first occurrence only, non-empty leaves shown with placeholders, empty leaves collapsed to a count. `9999` = number, `dd/MM/yyyy` = date as the service writes it (NOT valid xs:dateTime), `<anonymised>` = name/id/address/amount text.

### 7.1 RejectRisk - 15 Sep response (`RejectRisk_ResponsePayload.txt`)
```xml
{"status":"SUCCESS","data":"
<transactions>
  <transaction>
    <templateDetails>
      <channel>WEB</channel><outputFormat>PDF</outputFormat><templateName>RejectRisk</templateName>
      <copyTypeList><copyType>ORIGINAL</copyType></copyTypeList>
      <reportName>AGIC/UW/Plines/RejectRisk/RejectRisk</reportName>
      <outputPath>/output</outputPath><templateId>9999</templateId> <!-- + boResId, batchConfigId, docType, disallowCopy -->
    </templateDetails>
    <mailingDetails><address><!-- 2 empty --></address><email>x@example.com</email><name>x</name><senderEmail>x@example.com</senderEmail></mailingDetails>
    <transactionDetails>
      <headerTransaction>
        <reportTransaction>
          <reportData>
            <dataClass>9999</dataClass>
            <prntDetails>
              <agentInfo><address><!-- 19 empty --></address><!-- 16 empty --></agentInfo>
              <agencyInfo/> <branchInfo/> <clientInfo/> <compInfo/> <customerServiceInfo/>   <!-- all empty leaves -->
              <companyType><!-- empty companyDesc --></companyType>
              <taxClientInfo/> <buyerInfo/> <supplierInfo/> <paramGroup/>
              <transDetList>
                <transDet>   <!-- row 1 of 2 -->
                  <createdBy><anonymised></createdBy>
                  <createdDate>dd/MM/yyyy</createdDate>
                  <customerName><anonymised></customerName>
                  <idNo1>9999</idNo1><idType1><anonymised></idType1>
                  <reasoncode>9999</reasoncode><reasonCodeDescription><anonymised></reasonCodeDescription>
                  <revokeInd>N</revokeInd><rownumber>9999</rownumber>
                  <updatedby><anonymised></updatedby><updateddate>dd/MM/yyyy</updateddate>
                  <!-- ~620 empty leaves incl. leasingBranch -->
                  <coverList/> <provisoList/> <excessList/>   <!-- one empty child each -->
                  <staxList><staxPcts><staxPrem>9999</staxPrem><staxCount>9999</staxCount><asstFee>9999</asstFee></staxPcts></staxList>
                  <sobList/> <nomineeList/> <diagnosisList/> <csLimitList/> <memoList/> <specificationList/> <premiumProjectionList/>
                  <riskList><risk><rejectRiskType><anonymised></rejectRiskType><rejectRiskTypeCode>C</rejectRiskTypeCode></risk></riskList>
                  <payments/>
                </transDet>
                <transDet> ...row 2... </transDet>
                <companyType>
                  <companyDesc><remarks><anonymised></remarks></companyDesc>   <!-- one per row, REMARKS here -->
                  <companyDesc><remarks><anonymised></remarks></companyDesc>
                </companyType>
              </transDetList>
            </prntDetails>
          </reportData>
        </reportTransaction>
      </headerTransaction>
      <indicationList><indicationData/></indicationList>   <!-- not in v0.1.6 -->
    </transactionDetails>
    <reportParameters>
      <pReportTitle>PLINES : REJECT RISK REPORT</pReportTitle>   <!-- not in v0.1.6 -->
      <reportName>AGIC/UW/Plines/RejectRisk/RejectRisk</reportName>
      <!-- 217 empty leaves, incl. pCreatedDateFrom, pCreatedDateTo, many base-class p* names -->
    </reportParameters>
  </transaction>
</transactions>","timestamp":"2026-09-15T15:24:09"}
```
Facts visible: single reportData holding ALL rows (2 transDet), not one reportData per row (KT 04 4.1 describes the current code, this sample is older); `reportParameters` LAST; `templateDetails` before `mailingDetails`.

### 7.2 ASP - `5490878_delete.xml` (27 Sep), first reportData and a landscape reportData
```xml
<transactions><transaction>
  <templateDetails> channel EMAIL, outputFormat PDF, templateName ASPEndtSchedule_EndtDocument, copyType O,
                    reportName AGIC/UW/Plines/ASPEndtSchedule/EndtDocument, outputPath /output/asp/EndtDocument.pdf,
                    linkLocation /output/asp/, templateId 9999, boResId 9999, batchConfigId 9999, docType PDF, dataClass 9999 </templateDetails>
  <mailingDetails> ... </mailingDetails>
  <transactionDetails><headerTransaction><reportTransaction>
    <reportData>                                   <!-- #1 master: 5 transDet, 4,533 elements, no mainReportName -->
      <endorsementDetails>
        <accComm2Pct>A</accComm2Pct><contractId>9999</contractId><endtCode>9999</endtCode><endtEffDate>dd/MM/yyyy</endtEffDate>
        <endtNo><anonymised></endtNo><endtStiPrem>9999</endtStiPrem><gstRateType>..</gstRateType> <!-- + ~23 more -->
      </endorsementDetails>
      <prntDetails>
        <agentInfo><agentCode>..</agentCode><agentName><anonymised></agentName><agentCatType>PW</agentCatType><bancaAgent>N</bancaAgent><address/></agentInfo>
        <branchInfo><branchName><anonymised></branchName><phoneNo/><brAddress/><address/></branchInfo>
        <clientInfo><name><anonymised></name><address><address1/><address2/><address3/><city/><postCode/><state/></address></clientInfo>
        <compInfo><compName/><compBRN/><companyGstRegNo/><phoneNo/><fax/><address>...</address></compInfo>
        <companyType><companyDesc/></companyType>  <taxClientInfo><stampDuty>9999</stampDuty></taxClientInfo>
        <transDetList>
          <transDet> actionCode, agentCode, ciInd, cityDescp, cpAddrId, effDate, ... (non-empty sample)
                     <staxList><staxPcts><asstFee/><asstFeeDiff/><gpaStaxAmt/></staxPcts></staxList> ... </transDet>
          <!-- +4 transDet -->
        </transDetList>
      </prntDetails>
    </reportData>
    <reportData> ... landscape row, 1 transDet, 1,201 elements ... </reportData>   <!-- x4 more -->
  </reportTransaction></headerTransaction></transactionDetails>
  <reportParameters>
    <pendtNo/><pcompanyParamType/><pcopyType>O</pcopyType><pchecklistCopy>N</pchecklistCopy><pshowTaxInvoice>Y</pshowTaxInvoice>
    <prefNo/><pUserID/><pshowEndtSchedule>Y</pshowEndtSchedule><pContractId/><pversionNo/><pissueDate/>
    <pshowCoverNote/><pFirstPageNumber>9999</pFirstPageNumber><pPrintSeries>9999</pPrintSeries>
    <!-- 263 empty leaves; pendtNo, pcompanyParamType, prefNo, pUserID, pshowEndtSchedule, pContractId, pversionNo, pissueDate are NOT in v0.1.6 -->
  </reportParameters>
</transaction></transactions>
```
`dataClass` empty; no `slNo`/`mainReportName` text. The endorsementDetails is the FIRST child written in reportData here.

### 7.3 Template12 - `29441234.xml` (8 Oct, newest) - 1 reportData, 40 transDet
```xml
<transactions><transaction>
  <templateDetails> templateName Template12PolicyDocument, copyType IO, templateId 1790828239, boResId .., batchConfigId .. </templateDetails>
  <mailingDetails> ... </mailingDetails>
  <transactionDetails><headerTransaction><reportTransaction>
    <reportData>
      <mainReportName>Template12PolicyDocument</mainReportName>     <!-- NOT in v0.1.6 -->
      <subReportName/>                                              <!-- empty -->
      <endorsementDetails>
        <accComm2Pct>A</accComm2Pct><accCommPct>S</accCommPct><contractId>9999</contractId><partnerType>P</partnerType>
        <proposalRecvDate>dd/MM/yyyy</proposalRecvDate><showGstLabel>N</showGstLabel><source>OCP</source>
        <sstDate>dd/MM/yyyy</sstDate><staxInvNo><anonymised></staxInvNo> <!-- 36 empty leaves -->
      </endorsementDetails>
      <prntDetails>
        <agentInfo/> <branchInfo/> <clientInfo/> <compInfo/> <companyType/> <taxClientInfo/> ...
        <transDetList>
          <transDet>   <!-- 40 of these, one per fired subreport/band, merged by mergeNonNullFields -->
            <agentCode>..</agentCode><agentName><anonymised></agentName><agicFee>9999</agicFee><aspCutoffInd>Y</aspCutoffInd>
            <basicPrem>9999</basicPrem><commAmt>9999</commAmt><contractId>9999</contractId><debitNoteNo><anonymised></debitNoteNo>
            <effDate>dd/MM/yyyy</effDate><eInvConsentInd>N</eInvConsentInd> ...
            <staxList><staxPcts><staxAmt>9999</staxAmt><staxPrem>9999</staxPrem><staxCount>9999</staxCount><xShowStaxInvInd>N</xShowStaxInvInd></staxPcts> <!-- +9 --></staxList>
            <sobList><benefits><benefit><benefitPrint>9999</benefitPrint></benefit></benefits></sobList>
            <nomineeList><nominee><nomineePrint>9999</nomineePrint></nominee></nomineeList>
            <riskList>...116 cover blocks and 76 risk blocks across the response...</riskList>
          </transDet>
        </transDetList>
      </prntDetails>
    </reportData>
  </reportTransaction></headerTransaction></transactionDetails>
  <reportParameters>   <!-- exactly 73 leaves = whitelist -->
    <pcopyType>IO</pcopyType><pchecklistCopy>N</pchecklistCopy><pshowTaxInvoice>Y</pshowTaxInvoice><pshowCoverNote>..</pshowCoverNote>
    <pProductCode>9999</pProductCode><pCertInd>N</pCertInd><showComplaint>Y</showComplaint><showDebitNote>..</showDebitNote>
    <showScheduleStd>Y</showScheduleStd><parentName/><pmtDueDate>dd/MM/yyyy</pmtDueDate><policyholderName/><premDueAftPtv>9999</premDueAftPtv>
    <premDueRoundedAftPtv>9999</premDueRoundedAftPtv> <!-- 56 empty + 3 more non-empty -->
  </reportParameters>
</transaction></transactions>
```

---------------------------------------------------------------------

## 8. "What XSD changes did I make and why?" - in simple English (evidence only)

First an honest point. The Git history shows that the edits to the Pline.xsd file in the repo (3 commits: d4575fa5b 3 Aug, 6961abbf9 8 Aug, ea782121d 28 Sep) were made by other people (Shinge, Kumari Soni). Commits by GS (11 found) never touch a `.xsd` file. The XSD itself (v0.1 -> v0.1.6, "Added new XSD attributes" 2026-07-28) was supplied as a reference package. So "XSD changes" for you = changes to make the Java output and the mapping match the XSD, not edits of the schema. [CONFIRM with the developer whether they also edited the XSD or the master mapping sheet themselves; the files do not say who edited the master workbook.]

What you did, in plain words:
1. RejectRisk (commit 0d715fbad, 1 Sep 2026, OPUSM-41698, "Fix RejectRisk data mapping and align with Pline.xsd v0.1.6") [GIT]:
   - The old code wrote reject type and description as flat fields. The XSD has no such flat fields; it has `riskList/risk/rejectRiskType` and `rejectRiskTypeCode`. You moved them there. Why: the mapping sheet puts them in `risk`.
   - The old code used `idType`/`idNo`. The XSD names are `idType1/idNo1/idType2/idNo2`. You renamed them. Why: the XSD leaf names (from the sheet).
   - You moved REMARKS to `companyDesc.remarks` with one companyDesc per row, in the same order as the transDet. Why: "Mirrors Bond" - the master sheet puts remarks in companyDesc, and the XSD has no `remarks` under transDet.
   - You wired the ReportHeader `dataClass` lookup so `reportData/dataClass` is filled. Why: the sheet maps `DATA_CLASS` to reportData.
   - In the handler you fixed the SQL side that feeds those fields (join on CP_PART_ID/CP_PART_VERSION, code lookups by CAT_CODE/CODE_DESC, correct column names, parse the date range before querying). Why: wrong joins gave empty fields, and an empty field is an empty XML element.
   - Later (cc6a1b84d, 5 Oct) you fixed the date-range filter that matched 0 rows when one bound was null.
2. Template12 (commits b4e588bd4 "final_T12_code" 11 Aug, f79ecc811 20 Aug) and the T12 mapping sheet (19 Aug): you built the 1944-row jrxml -> table.column -> XSD path map for Template12 and implemented the services against it. The sheet shows every path was checked to be a real XSD leaf (all 228 distinct paths are valid in v0.1.6). Why: to make sure each Jasper field lands in a leaf that exists, and to know where each value comes from. You also found and fixed two merge defects (staxList lost, `eInvStatus` overwritten by blank) with the fix limited to Template12 so nine other templates were not affected [CODE_CHANGES_LOG, WB].
3. ASP (commits 578778080, ba9e29586 25-26 Aug, 7a925920c 28 Aug, 94ea23d81 21 Sep, cc6a1b84d 5 Oct) [GIT]: wired the EndtDocument subreports, fixed the CONTRACT_ID type mismatches (a data-type problem that makes a leaf empty), folded the Landscape reports into one response and then split each Landscape row into its own reportData, fixed the branch join, and the shared-mutable-state bug in `staxPcts()` (the same object was reused in several transDet rows, so values bled across rows). Why: so each subreport fills the leaves the master `EndtDocument` tab says it should.
4. The 06 Jul master sheet and the v0.1.6 XSD set the rules you followed: reuse the Bond/Common leaf if the name matches, otherwise add a Pline leaf in the Bond group; types are decimal/dateTime/string/integer; Common.xsd untouched.
5. What is NOT done, and the interviewer may find (be upfront): the XML still fails validation on the newest samples. The biggest causes are empty decimal/dateTime elements (null written as an empty element), dd/MM/yyyy dates where the XSD wants ISO dateTime, element ORDER inside many types, and non-XSD elements (`pReportTitle`, `pCreatedDateFrom/To`, `mainReportName`, `subReportName`, `leasingBranch`). Some of these were added by the team ahead of the XSD (code comments cite v0.1.9/v0.1.10, which I never found). Say it as: "I aligned the mapping and names; strict XSD validation of the runtime XML is a known open gap and I have measured it."
[CONFIRM] items for you: (a) whether the 1 Sep samples (10:37-10:42) came from the code later committed at 21:42 or from your working copy; (b) whether you chose `prntDetails/companyType` (sheet) or `transDetList/companyType` (code) for REMARKS and why; (c) who produced the master sheet comments; (d) any ASP sheet outside this repo.

---------------------------------------------------------------------

## 9. Interview questions (25) - XSD / mapping / sample payloads
Format: Q; strong answer; common mistake; follow-up.

1. Q: How did you decide where a Jasper field goes in the XSD?
   A: Start from the jrxml field and its DB table.column. Look it up in the master mapping tab (Attributes -> XSD Attribute Name -> XSD Group). Rule: reuse a Bond/Common leaf if the name matches, otherwise a Pline leaf in the group Bond would use, otherwise mirror Bond's existing mapping. Then confirm the full path exists in Pline.xsd. I checked 228 of 228 Template12 paths and 21 of 21 RejectRisk paths.
   Mistake: "I just matched names". Follow-up: Show a case where the name differs (`ACCOUNT_NO` -> `agentCode`, `ROWNUM` -> `rownumber`).
2. Q: Walk me through the RejectRisk mapping.
   A: 21 fields: 17 transDet leaves, 2 in `riskList/risk`, `REMARKS` in `companyDesc.remarks`, `DATA_CLASS` in `reportData/dataClass`. Dates are xs:dateTime, ROWNUM xs:decimal, the rest strings.
   Mistake: forgetting remarks and dataClass are not in transDet. Follow-up: where do `pCustomerName`, `pIdNo`, `pRejType`, `pReasonCode` go? (Nowhere - they are query filters.)
3. Q: Why is REMARKS in companyDesc?
   A: "Mirrors Bond": the master sheet says Bond maps remarks into companyDesc; v0.1.6 has no `remarks` in transDet. One companyDesc per row, index-matched. Caveat: the workbook path says `prntDetails/companyType`, the code and the 15 Sep response use `transDetList/companyType`; both are legal.
   Mistake: claiming the XSD forced it. Follow-up: how does the print engine find the right remarks for a row? (index match [CONFIRM].)
4. Q: How is the mapping sheet produced and kept in sync?
   A: For RejectRisk, a PowerShell script reads the master tab through Excel COM, builds a lookup keyed on the Jasper attribute and rewrites the XSD name/group columns of the per-report CSV, then a second script builds the xlsx. No automatic re-sync after XSD version changes - the master sheet is v0.1 (6 Jul) while the XSD is v0.1.6, and 4 names already drift.
   Mistake: saying "it is auto-generated and always correct". Follow-up: how would you detect drift? (Script: compare sheet names with XSD element names - I did it in minutes.)
5. Q: What does "Existing in XSD = Y" mean in the master sheet?
   A: In practice nothing useful: it is Y on all 1571 rows, even where the comment says "Existing-in-XSD=N". The real signal is the comment: new Pline field / reuses / mirrors Bond.
   Mistake: treating it as validated. Follow-up: how many new Pline fields in Template12? (292 of 551.)
6. Q: What payload do you send to generate a RejectRisk report?
   A: POST /gi/documentprintdata/api/opus/GenerateReport with `metadata` (19 template/mail keys) and `filterCondition` with 8 keys: reportName, pReportTitle, pCreatedDateFrom/To (dd/MM/yyyy), pCustomerName, pIdNo, pRejType, pReasonCode. The reportName selects the Spring bean.
   Mistake: thinking metadata drives the query. Follow-up: what happens if a date bound is null? (It matched 0 rows until the 5 Oct fix.)
7. Q: How do ASP and Template12 requests differ?
   A: ASP needs contract + endorsement identity (pContractId, pEndtNo, pRefNo, versionNo, pPolicyNo, pType, pAtceInd, pUserID) and show flags (showCover, showEndtSchedule...). T12 needs pContractId, pPolicyNo, versionNo, companyParamType, show flags, pCertNoFrom/To, pCertInd, pTaxInvNo, pGstTaxInvoiceNo, ShowOutPrintCovLetter.
   Mistake: using the same keys. Follow-up: which keys end up in the XML? (Only whitelisted reportParameters names.)
8. Q: Does the newest RejectRisk response validate?
   A: No. 15 Sep response: 1754 total / 888 logical errors; 97% are empty decimal/dateTime/integer/boolean elements. Old figures reproduced: 929 and 1766. The newest response changed by only -12 total.
   Mistake: reading "fewer errors" as "fixed". Follow-up: what caused the -12? (staxPrem/staxCount/asstFee got values.)
9. Q: Why does Xerces report every empty decimal twice?
   A: `cvc-datatype-valid` for the lexical value plus `cvc-type.3.1.3` for the element. Count "logical" errors by dropping the second.
   Mistake: quoting 1754 as 1754 problems. Follow-up: what is the real number? (888 logical.)
10. Q: Which errors are structural and why can't you count them exactly?
    A: Order or unknown elements (address after other fields, IDNo, benefit, provisoList, companionName, occpDescp, copyTypeList, mailingDetails...). Xerces stops validating a parent after its first unexpected child, so counts are a floor; fixing one reveals the next (`pchecklistCopy` appeared only after `pendtNo` went away).
    Mistake: stating the structure error count as complete. Follow-up: how would you get the complete list? (Iterative fix loop, or a path-based diff of element names against the enumerated XSD.)
11. Q: What improved between the September and the 8 Oct Template12 samples?
    A: reportParameters whitelisted to 73 leaves, `Policyletterind` casing, no `pendtNo`, no `indicationList`, fewer buyer/supplier order errors. Commit bca355525 (6 Oct).
    Mistake: attributing it all to your commits; that commit is by another developer. Follow-up: what got worse? (`mainReportName` unknown, `pchecklistCopy` order now visible.)
12. Q: What are mainReportName and subReportName and are they in the XSD?
    A: Added to each reportData by `PlineReportNames` (main = bean-derived, e.g. `Template12PolicyDocument`) and by ASP code (sub = subreport name, e.g. `PrintSeriesFooter`). Not in v0.1.6 (validated: "unexpected element mainReportName"); code comments cite v0.1.10 which I did not find.
    Mistake: saying they exist in the XSD. Follow-up: only one real sample has them; which? (T12 `29441234.xml`, sub is empty.)
13. Q: Give the order of reportData blocks in an ASP response.
    A: Master (`PrintSeriesFooter`), EndtCover, ComplaintRedress, EndtSchedule, DebitNote, CustomerDtlsOfMasterPolicy_Debit, EndtTaxDoc, ServiceTaxInvoice (+MT, MTSub), QRSection, TransitionalInvoice(2), then landscape rows. In the Sep samples there is no name element; 5-15 reportData per response.
    Mistake: presenting this as observed in samples (it is from code). Follow-up: how do you know a subreport fired? (Tracker xlsx: Done / Gate fired / Not fired.)
14. Q: How big are the real responses and why?
    A: ASP 4k-37k elements (1-15 reportData); T12 8 Oct 43.9k elements with 40 transDet; RejectRisk 1.4k-2.3k. 95%+ of leaves are empty because every transDet class carries ~634 scalar leaves and null is serialised as `<x/>`.
    Mistake: saying the payload is small. Follow-up: fix? (`NON_NULL` inclusion, but then empty-vs-absent semantics for the print engine need agreeing.)
15. Q: Why are dates a problem?
    A: XSD uses xs:dateTime (64 leaves, no xs:date) = `2017-06-06T00:00:00`; the service writes `06/06/2017` via `@JsonFormat(dd/MM/yyyy)`. Present in every sample (5 in RR, 195 in the 8 Oct T12).
    Mistake: blaming the database. Follow-up: options (ISO in XML + format in Jasper; or XSD pattern type).
16. Q: What is the whitelist on reportParameters and what did the samples show?
    A: `@JsonIncludeProperties` with 73 names. Before it (Sep ASP/RR) responses carried 219-265 reportParameters leaves, 148-200 unknown to the XSD; after it (8 Oct T12) exactly 73, six of which (pCreatedDateFrom/To, pReportTitle, pshowEndtSchedule, pShowOutPrintCovLetter, pShowPDPANotice) are still not in v0.1.6.
    Mistake: thinking whitelisting makes it XSD-valid. Follow-up: what silent drop does it cause? (setter names that map to non-whitelisted properties.)
17. Q: How did you test that a sheet path is real?
    A: Enumerated all 1410 paths of the XSD (78 containers, 1332 leaves) and compared; 228/228 T12 paths and 21/21 RejectRisk paths exist; names drift only 3 (PolicyDocument) / 1 (EndtDocument).
    Mistake: grepping only the leaf name (misses wrong container). Follow-up: why path not name? (`address` exists in many groups.)
18. Q: What does Template12's code sheet cover versus the XSD?
    A: 228 of 1332 leaf paths; 164 of 634 transDet leaves; 10 of 67 reportParameters; 7 of 50 endorsementDetails. The master tab names 474 attributes, so the code sheet is the coded subset.
    Mistake: calling the 1944 rows "1944 fields" (they repeat across subreports: 231 distinct path+attribute pairs). Follow-up: what are the 1944 rows? (jrxml x column.)
19. Q: Why does one Jasper field appear under many parents?
    A: Shared subreports/variants (CIPrint, CIPrint_BMW, Coverage_BMW...) read the same column and all write one transDet leaf; 42 rows have a multi-parent cell.
    Mistake: thinking it creates duplicates in XML. Follow-up: how are variants merged? (`mergeNonNullFields`.)
20. Q: Describe a mapping bug you found while checking sample responses.
    A: staxCount missing: it exists only inside `staxList.staxPcts`; the merge skipped any List-typed field except risk/sob/nominee. Added staxList to the re-merge. Second: `eInvStatus` overwritten with "" by another handler; fixed in the T12 merge step because the shared handler is used by 11+ templates.
    Mistake: changing the shared handler. Follow-up: how verified? (Postman response with staxCount=2 for the test contract, 19 Aug.)
21. Q: What is the status of ASP subreports, how do you know?
    A: Tracker sheets `ASP_contract_id_list.xlsx`: EndtDocument all Done (9 rows), EndtDocument2: 4 Done, 3 "Not fired / gate not met", 1 "gate fired, value not confirmed", 11 untested.
    Mistake: claiming full ASP coverage. Follow-up: which contracts cover what? (27275348 covers all EndtDocument subreports.)
22. Q: What is wrong with `<staxPrem>` having a value in a RejectRisk response?
    A: RejectRisk has no tax data; a constant-looking value (7497.1) in the placeholder staxPcts in the 15 Sep response indicates default/test data leaking from a shared class. It also lowers the error count for the wrong reason.
    Mistake: celebrating the lower error count. Follow-up: where would you look? (PlineStaxPcts defaults / service init) [CONFIRM].
23. Q: Would you change the XSD or the Java to fix the failures?
    A: Decide the contract first. Cheap XSD-side fixes: add mainReportName/subReportName and the six parameters; allow ISO or a pattern for dates. Cheap Java-side: `@JsonPropertyOrder` on the ~12 types with order errors, NON_NULL for numeric/date leaves, ISO dates. The empty numeric elements (97%) are Java-side or an XSD-nillable decision.
    Mistake: "change both". Follow-up: who owns the XSD? (Schema owner / Plines requirements team.)
24. Q: How would you prevent regression?
    A: A ratcheting test: run the services with mocked handlers, serialise with the production mapper, collect all errors, assert count <= stored baseline (RejectRisk 888 logical). The existing PlineXsdValidationTest only validates a hand-written shell.
    Mistake: validating a static sample. Follow-up: where do you store the baseline?
25. Q: A reviewer says "your responses fail the XSD, so the feature is broken". Reply?
    A: Separate the facts: the mapping (field -> leaf) is complete and path-valid (21/21, 228/228); the runtime XML has serialisation issues (empty typed elements, date format, element order) that I measured and categorised; I do not know whether the print engine validates (not evidenced) - so I would confirm with the consumer, then fix by priority: empties and order first.
    Mistake: defensive "the XSD is wrong". Follow-up: what evidence would change the priority? (Print engine rejecting a payload.)

---------------------------------------------------------------------

## 10. Key findings (short)
1. Workbooks: the master v0.1 sheet (21 tabs, 1571 rows; T12=PolicyDocument 551, ASP=EndtDocument 474, RejectRisk 21) is a generated, never re-synced decision table (Existing-in-XSD = Y everywhere; comments are the only status). RejectRisk and Template12 have separate code sheets (21 and 1944 rows) with DB table/column and slash path; all their paths are valid in v0.1.6. Real status columns exist only in tracker workbooks (ASP/T12 contract lists; the T12 Report_Structure status columns are empty).
2. Process: jrxml analysis -> DB column -> master-sheet decision (reuse/new/mirror Bond) -> per-report sheet (RejectRisk via Apply_MasterMapping.ps1 + Build_RejectRisk_Xlsx.ps1) -> Java payload -> curl -> check -> tracker.
3. Newest samples still fail: RejectRisk 15 Sep 1754 total/888 logical (old 929/1766 reproduced exactly; -12 only because staxPrem/staxCount/asstFee got values); ASP 7.5k-30.8k; T12 8 Oct 41,061 (20,692 logical). 95-98% of errors are empty decimal/dateTime/integer/boolean elements, the rest are dd/MM/yyyy dates and element ORDER/unknown elements.
4. Improvement visible only in T12 8 Oct (after another developer's commit bca355525): whitelist (73 parameters), `Policyletterind`, no `pendtNo`/`indicationList`. Regression: `mainReportName` (not in XSD) and a newly exposed `pchecklistCopy` order error. No post-change RejectRisk or ASP response exists.
5. `mainReportName/subReportName` appear in exactly one real response (T12 8 Oct: main `Template12PolicyDocument`, sub empty); the ASP sub names exist only in code.
6. Requests: all three use one endpoint, `metadata` (19 keys) + `filterCondition` (RejectRisk 8 keys, ASP 21, T12 20); the report path is the bean selector.
7. Git shows the XSD file edits were by other developers; GS's evidenced work is the mapping/alignment code (RejectRisk 0d715fbad 1 Sep, ASP and T12 commits) and the workbooks.

Scratch files (safe to delete): `C:\Users\EPGRVFY\AppData\Local\Temp\xsdcheck2` (dump of all workbooks as text, `V.java` validator, path enumerator, results `all_results.txt` covering all 66 ASP/T12 samples plus the three RejectRisk files).
