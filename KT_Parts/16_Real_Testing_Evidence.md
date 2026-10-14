# 16. Real Testing Evidence (manual data-parity testing, from your own folders)

Purpose: this file replaces the "no integration / SIT evidence" statement in `05_Testing_Deep_Dive.md` with what your working folders actually prove. It is built only from files I read. Labels used: **EVIDENCE** = read in a named file or screenshot; **INFERENCE** = my reading, not proven; **[CONFIRM]** = a fact only you can supply.

Redaction: Authorization headers, tokens, the Mongo connection string (read by `mp.sh` from `application.properties`), the SmartComm URL and Jasper server address are deliberately not reproduced (shown as `<redacted>`).

Folders read:
- `C:\Users\EPGRVFY\Documents\plines_testing\` (296 files)
- `C:\Users\EPGRVFY\Documents\jr-tools\` (10 files)
- `C:\Users\EPGRVFY\Documents\pdf_generation\PDF\`
- `C:\Users\EPGRVFY\Documents\Plines_Base_code\T12_final_code\RejectRisk\`
- Small extras outside the requested list, used only where flagged: `Documents\ASP_contract_id_list.xlsx`, `Documents\ASPEndtSchedule\base_code\ASP\ASP_contract_reports_mapping.xlsx`, `Documents\Data_mismatch_issue.docx`.

---

## 1. Inventory

### 1.1 `plines_testing\` (296 files)

By type (EVIDENCE, file listing): 121 PNG, 66 TXT, 57 SQL, 37 XML, 7 XLSX, 2 PS1, 2 `.data` (jrxml dumps), 1 DOCX, 1 ZIP, 1 INI, and 1 extension-less file (`hardcoing_details`).

| Folder | What it contains |
|---|---|
| `RejectRisk\` (16 files) | `RejectRisk_main_Q.sql` (jrxml query with `$P{}` placeholders), `rejectrisk_run_q1.sql` (same query with literals substituted, runnable), `Reportheader.sql` (ReportHeader sub-report query), `RejectRisk_jrxml.data` (jrxml dump), `mongo_compass_aggregate.txt` (the Mongo aggregation, plus one console log line), `payload.txt` (test data notes plus request payloads), `postman_result.txt` (saved Postman response, 2 `reportData` blocks), `REJECTRISK_IMAGES.docx` (8 embedded screenshots under headings "SQL RESULT: Main Report / Sub Report / POSTMAN / CONSOLE / MongoDB Compass"), and 8 PNG (`REJECTRISK_SQL1-3`, `REJECTRISK_SUB_REPORT_SQL`, `Mongo_compass_aggre_result_main_report`, `Mongo_compass_aggre_result_sub_report`, `REJECTRISK_POSTMAN`, `REJECTRISK_CONSOLE`). |
| `Template_12\` top level | `AGICPROD~3.sql` and `AGICPROD~4.sql`: Oracle worksheet files used to look up gate values and to hunt for test contracts (see 2.5). |
| `Template_12\PolicyDocuement\` (typo is in the real folder name; 13 files) | Oracle SQL for PolicyDocument (PD) main report and sub-reports (`PD_main_report.sql`, `PD_main_report_Run_Q.sql`, `CIPrint_ATC.sql`, `PerilsClauses_ATC_SR_CIP_ATC.sql`, `PIDMSetA_CIP_ATC.sql`, `signature_SR_CIP_ATC.sql`, `PrintSeriesFooter.sql`, `OutSourcePrintCoverLetter.sql`, `practice.sql` and others), plus `subreport_path.txt` (the sub-report tree). No screenshots and no checklist for PD itself. |
| `Template_12\PolicyDocument2\` (about 100 files) | 44 PNG (SQL, Postman, Compass screenshots per sub-report), about 25 `.sql` (one per sub-report, for example `TransitionalInvoice2.sql`, `PremiumDebitNote_T12.sql`, `STI_QRSection.sql`), 2 response folders `old_response\` (14 TXT) and `New_response\` (17 TXT), checklist workbooks (`contract_id_report_path_payload\reports_list.xlsx` dated 2026-08-08, `reports_and_contract_ids_and_path.xlsx`, `reports_list.backup.xlsx`), mapping workbooks (`Template12_PolicyDocument2_Subreport_Mapping.xlsx` and `_v2`), `payload.txt`, `report_path.txt`, `hardcode_details\` (3 notes), and `opus-documentprintdata-service.zip` (a code snapshot). |
| `Template_12\Landscape\` (about 120 files) | 69 PNG, 15 `.sql` (one per Landscape band) plus `landscape_main_report_Q.sql` / `_run_Q.sql` and `T4_Landscape_sql.sql`, `T4_Landscape_jrxml.data`, `old_responses\` (8 TXT) and `new_responses\` (15 TXT), `contract_id_and_report_path_payload\` (`t12_LS_payload_MR.txt`, `t12_ls_report_path.txt`, `t12_ls_Reports_check_list.xlsx` dated 2026-08-07), and two PowerShell helpers `fill_excel.ps1`, `verify_excel.ps1` that write and read the checklist workbook through Excel COM. |
| `Template_12\dynamic_merge\` (40 files) | After the PD2 + Landscape merge: `contract_id_and_responses\contract_id_details_.txt` (verified coverage matrix), 11 saved full responses named `<contractId>_id.xml` (up to 1.8 MB each), `T12_PD2_response_dynamic_merge\` (16 XML responses, up to 2.7 MB), `t12_landscape_xsd_response_dynamic_merge\` (10 band XML plus `changes i made today.txt`), and `Template12_XSD_Mapping_Final.xlsx`. |
| ASP | **No ASP folder inside `plines_testing`.** ASP evidence exists only indirectly (see 3.3). |

Date span of the files (EVIDENCE, file times): RejectRisk screenshots 2026-06-19 to 2026-07-08; Landscape screenshots 2026-07-13 to 2026-07-29; PD2 screenshots 2026-07-22 to 2026-08-11; dynamic merge 2026-08-05 to 2026-08-08.

### 1.2 `jr-tools\` (10 files)
`MongoProbe.java`, `mp.sh`, `ReportTree.java`, `UpdateStatus.java`, `XlsxDump.java`, `b1.txt`, `b2.txt`, `b3.txt` (batch command files for MongoProbe), `ids100909.txt` (103 contract ids), `out1.txt` (a probe output). Details in section 6.

### 1.3 `pdf_generation\PDF\`
`PDF_Generation_Procedure.txt` (6-step recipe), `curlurl.txt` (curl to the SmartComm render endpoint; contains a Basic Authorization header, redacted here), `AMOS_EndPoint_Latest.postman_collection.json` (one POST request, `generateDocument?failOnError=false&includeDocumentData=true`), `encode 1\encode\` (`input.xml`, `encode_amit.ps1`, `base64.txt`, `request_body.json`) and `encode 1.zip`, and `sample.pdf` (278,950 bytes, 8 pages, PDF producer string "Smart Communications - SC33"; its source `input.xml` is a `Template12PolicyDocument` payload).

### 1.4 `Plines_Base_code\T12_final_code\RejectRisk\`
`API_Crul_1357766.txt`, `API_Crul_2439275.txt` (curl commands, no token in them), `1357766.xml` and `2439275.xml` (saved XSD-shaped responses), `RejectRisk_API_Crul_2439275_Response.txt` (curl plus response in one file), `Mapping_sheet\` (jrxml hierarchy CSV, jrxml-table-column-to-XSD mapping CSV/XLSX, two PowerShell builders), `RejectRisk\RejectRisk_XSD_Mapping_sheet.xlsx`, `RejectRisk.zip`. There is a file called `postman_result.txt` only in `plines_testing\RejectRisk\`, not here.

---

## 2. The testing method actually used

### 2.1 Tools and environments that appear (EVIDENCE unless marked)

| Tool | Proof |
|---|---|
| Spring Boot service run **locally on port 8080**, endpoint `/gi/documentprintdata/api/opus/GenerateReport` | `API_Crul_*.txt` (`http://localhost:8080/...`); console screenshots show thread `nio-8080-exec-1`, app name `Documentprintdata`. Log time stamps carry `+05:30` (your machine's zone). |
| IDE console (IntelliJ style) with DEBUG logging of `MongoTemplate` | `REJECTRISK_CONSOLE.png`, `Main_report_console.png`: "Executing aggregation: [...] in collection ...". This is how the pipeline was copied into Compass. |
| **Postman** | `REJECTRISK_POSTMAN.png` (Status 200 OK, 965 ms, 19.39 KB, JSON body `{"status":"SUCCESS","data":"<transactions>..."}`); Landscape and PD2 screenshots show Postman XML view; the checklist workbooks have a "Postman" column. |
| **curl** (as saved text, to be imported into Postman or run in a shell) | `API_Crul_*.txt`, `curlurl.txt`. |
| **MongoDB Compass** (Aggregations tab, TEXT view of the pipeline, and Documents tab filters) | `Mongo_compass_aggre_result_*.png`, `REJECTRISK_IMAGES.docx` image 7 (collection tab `CMUW_PL_REFER_RISK`, "Documents 228", "Aggregations", database name starts `OPUSAGICPR...`, a hosted cluster in the EU-central region). Other tabs visible: `CMGE_UPLOAD_FILE`, `RPGE_RPT_TEMPLATE`, `WIP_UWGE_POLICY_VERSIONS`. |
| **Oracle SQL Developer** (grid output, "All Rows Fetched: 2 in 0.649 seconds") | `REJECTRISK_SQL1-3.png`, all Landscape `*_sql*.png`. The default worksheet names `AGICPROD~3.sql` / `AGICPROD~4.sql` suggest a saved connection called AGICPROD (INFERENCE from the file name; you told me the database is AGICPROD). Queries use the `customer.` schema prefix (for example `customer.UWPL_VI_POLICY_SCHEDULE_TPL4`, `CUSTOMER.RPGE_RPT_TEMPLATE`). |
| Excel via PowerShell COM | `fill_excel.ps1`, `verify_excel.ps1` (Landscape checklist); jr-tools `XlsxDump.java`, `UpdateStatus.java`. |
| Jasper side | Not run as a report in the folders. The Jasper "expected" data is represented by the **jrxml SQL** (`*_main_Q.sql`, `*_jrxml.data`). INFERENCE: you treated the jrxml query result as the legacy truth. |
| SmartComm rendering (AMOS endpoint) | `pdf_generation` folder: used once to render a Template12 PolicyDocument PDF (2.6). |

### 2.2 The three-way comparison, step by step (this is the core method)

EVIDENCE for the columns: the checklists `t12_ls_Reports_check_list.xlsx` and `reports_list.xlsx` have the columns **Postman | SQL | Compass | Notes / Issues Found | Verification**. The RejectRisk folder shows the same trio.

1. **Choose a driving record or contract.** RejectRisk: look up a record in Oracle first (`payload.txt` lists `PART_ID => 1357766`, `NRIC 841004017669`, name `TAN T357766`, created `06-JUL-17`, "so taking between 06-06-17 to 31-08-17"; and `CP_PART_ID => 2439275`, name `LEE KEN439275`, id `671207082757`, `REJ_TYPE C`, `REASON_CODE 09`). T12: pick a contract that satisfies the jrxml gate (`PRODUCT_CODE`, `SCHEME_NAME`, `ASP_CUTOFF_IND`, `NOMINEE_PRINT`, `BENEFIT_PRINT`, `DIAGNOSIS_PRINT`), found with helper SQL (2.5).
2. **Call the Java API.** Postman POST with `metadata` and `filterCondition`. RejectRisk filters: `pReportTitle`, `pDateFrom/pDateTo` (later renamed `pCreatedDateFrom/pCreatedDateTo`), `pCustomerName`, `pIdNo`, `pRejType`, `pReasonCode`. T12 filters: `pContractId`, `versionNo`, `companyParamType`, `showComplaint`, `showDebitNote`, `showTaxInvoice`, `pTaxInvNo` and so on (`PolicyDocument2\...\payload.txt`; `t12_LS_payload_MR.txt`).
3. **Read the response** (XML inside JSON for RejectRisk; XML view in Postman for T12). Look for the `*Details` element of the sub-report being tested (the Postman screenshots have a find-bar on names such as `T12PremiumDebitNote`, `particularOfInsuredLandscapeAmcDetails`).
4. **Copy the aggregation from the console log** into Compass, run it, and look at "Pipeline output preview". This isolates whether a wrong value comes from the Mongo pipeline or from DTO/payload mapping.
5. **Run the jrxml SQL in Oracle SQL Developer** with the same ids (`rejectrisk_run_q1.sql`, `landscape_main_report_run_Q.sql`, per-band `.sql` files, with `$P{}` replaced by literals or `:pContractId`).
6. **Compare** row counts and field values across Oracle, Compass and Postman. Record `Yes/No` in the checklist and put discrepancies and fixes in "Notes / Issues Found" (workbook text, for example "Fixes: POLICY_NO field name (was POLICY_NO, fixed to POLICY_REF); SHOW_OCCP_DESC BSON type bug; RB_AMT null guard").
7. **Fix the Java handler/DTO, restart, re-run** and keep the before and after responses: `old_response\` and `New_response\`, `old_responses\` and `new_responses\` (EVIDENCE: for T4 landscape 1 the new response is 138,213 bytes against 110,055 bytes old with the same 6 `reportData` blocks, and the AZP/AZP2 nomination file is 50,741 against 41,352 bytes: the new responses carry more populated fields; INFERENCE on cause: the added `$project` mappings).
8. **Screenshot** each comparison (`SS taken` in the Verification column, 121 PNG).
9. **Gate testing for hard-to-find data**: temporary "Hardcode for Testing" overrides in Java (2.4).
10. **Whole-report regression** after the PD2 + Landscape dynamic merge: 11 contracts run end-to-end, results classified (2.7).

Important difference to what the earlier pack guessed: there is **no evidence of comparing the Java XML against a Jasper-generated PDF**, nor of an XML-vs-XSD validation run, in these folders. The comparison is **Oracle query result (jrxml SQL) versus Mongo pipeline result versus API XML**, plus a one-off PDF render through SmartComm (2.6). The XSD side is a mapping exercise (`*_XSD_Mapping*.xlsx`, `RejectRisk_Jrxml_Table_Column_Mapping.csv`), not a validator run.

### 2.3 Parameters and test data per module (EVIDENCE)

RejectRisk
- Run A (June to 8 July): dates `01/07/2017` to `30/09/2017`, name `LEE KEN439275`, id `671207082757`, type `C`, reason `09` (`payload.txt`, `REJECTRISK_POSTMAN.png`).
- Run B: name `TAN T357766`, id `841004017669`, type `C`, reason `09`, dates `01/06/2017` to `30/08/2017` in `payload.txt`, `06/06/2017` to `31/08/2017` in `API_Crul_1357766.txt`.
- Aug/Sep curl files use `pCreatedDateFrom/pCreatedDateTo` (the renamed parameters) for both people.
- Header check: `RPGE_RPT_TEMPLATE` where `DOC_TYPE='OPUS_RPT'` and template name `AGIC/UW/Plines/RejectRisk/RejectRisk/PLINES : REJECT RISK REPORT` returns `DATA_CLASS = 1`.

Template12 Landscape (checklist `t12_ls_Reports_check_list.xlsx`)
| Band | Sub-report | Contract(s) in sheet | Gate |
|---|---|---|---|
| Main | Template12LandscapeDispatch | 24994547 / 6235610 / 39765306 | always |
| 1 | ParticularOfInsuredLandscape_ASP | 17085732, 17139251 ("Good") | product 070412 and ASP_CUTOFF_IND=N |
| 2 | ParticularOfPremiumLandscape_ASP | 17139251 | same |
| 3 | ParticularOfSponsorLandscape | 23481292 | same |
| 4 | ParticularOfBenefitsLandscape | 17139251 | BENEFIT_PRINT>0 and not 070504/070521/070406 |
| 5 | ..._Benefits_Elite | 6623256 | 070504, ELITESHIELD |
| 6 | ..._Benefits_Income | 6858001 | 070504, INCOMESHIELD |
| 7 | ParticularOfPremiumLandscape_Elite | 6623256 | 070504, ELITESHIELD |
| 8 | ParticularOfPremiumLandscape_Income | 6623314 | 070504, INCOMESHIELD |
| 9 | ParticularOfNominationLandscape_Elite | 21042351 "(not working)", 36022275 "(hardcoded)" | 070504, nominee, not POSCOVER/AZP |
| 10 | ParticularOfNominationLandscape_POS | 19477065 "(not working)", 36022275 | 070504, POSCOVER |
| 11 | ParticularOfNominationLandscape | 17139251 "(notworked)", 36774231 | nominee, not 070504 |
| 12 | ParticularOfDiagnosisLandscape | 2541599, 40265574 "(hardcode)" | diagnosis, not 070504/100908 |
| 13 | ParticularOfInsuredLandscape_AMC | 25482538, 24631551 "(hardcode)" | product 100908 |
| 14 | ParticularOfDiagnosisLandscape_AMC | 25482538 "(hardcoded)" | 100908 |
| 15 | ParticularOfNominationLandscape_AZP | 25144644 (AZP1, hardcoded) / 25061730 (AZP2) | 070504, AZP1/AZP2 |
| 16 | Landscape (Template 4) | 39765306 and six more (31661260, 38759788, 5806920, 31431268, 14039711, 17512590, 23881104) | 070406 and PROMO_AMT>0 |
The sheet's second block (T4-1 to T4-14: Benefits_ATC, Benefits_GPA, Insured_ASP, Premium_ATC, Premium_GPA, Premium_ATC_Promo, Companion, Boat and so on) is **empty**: not tested.
All 16 rows have Postman = Yes, SQL = Yes, Compass = Yes and "SS taken" except row 16 (Verification empty). EVIDENCE caveat: an earlier version generated by `fill_excel.ps1` had rows 7 to 12 as "No / No / No"; the later workbook (2026-08-07) has them as Yes. So those six were done after the script ran.

Template12 PolicyDocument2 (`reports_list.xlsx`, 2026-08-08): main report contracts 2561619 and 7043083 ("two contracts used to cover both JOURNEY_AREA branches"); Domestic_ATC 2575144; Overseas_ATC 2561619; ComplaintRedress_SCB / CompanyRegNo / NoMargin 2885273 (hardcoded); PremiumDebitNote T12 29554526 / 11436963 (hardcode) and T4 33154885 / 2574275; TaxInvoince + CompanyContact + TaxInvoiceProductInfo 2754289; ServiceTaxInvoice 37693177 with `pTaxInvNo STAX24076000001854`; QRSection / PA4A 29735791 (hardcoded); STaxInvoiceProductInfoMST 36880167 (hardcode); TransitionalInvoice and TransitionalInvoice2 12721508; PDS_Elite 6623256; PDS_Income 6858001; PDPA 2561619. Status column: "Done" for Blank, ComplaintRedress group, Domestic_ATC, Overseas_ATC, TaxInvoince group, TransitionalInvoice2 group; "Not started" for PremiumDebitNote, Signature and others; "Pending / Not tested" for ServiceTaxInvoice (T12 and T4), the MST customer-details sub-reports, PDPA (1107 path) and NominationForm ("file Doesn't exist").

### 2.4 "Hardcode for Testing" blocks (EVIDENCE: `PolicyDocument2\hardcode_details\hardcoing_details`, `changes_made_on_merged_code.txt`)
Where no real contract fired a gate, temporary Java overrides keyed on a real contract id were written between `//Hardcode for Testing Starts / Ends`. Inventory: 13 blocks in 10 files. Examples: force `productCode="070406"` for contract 37693177 (route to T4 branch); force `productCode="070504"` for 2885273 (fire ComplaintRedress_SCB); force `mstContractId=99999901/99999902, premByMstInd="Y"` (fire CustomerDtlsOfMasterPolicy); force `finalStaxCount=2`, `einvConsentInd="Y"`, `qrCnt=1` for 29735791; and **fabricated fallback DTOs** (for example an "INTEL ELECTRONICS ... MST TEST" row) when the real query returns empty.
Later note ("merged code ->"): 8 files had these blocks commented out again; 3 were already commented. So the final code should not contain them (EVIDENCE of intent only; verify in the repo with `git grep "Hardcode for Testing"` [CONFIRM]).
Honest reading: a gate firing with hardcoded or fabricated data proves the **dispatch/wiring and XSD placement**, not data correctness. The checklist itself marks those with "(hardcoded)" and "(not working)".

### 2.5 Oracle helper SQL to find test data (EVIDENCE)
- `AGICPROD~3.sql`: for contract `27533722` computes `PRODUCT_CODE` (from `wip_uwge_policy_contracts` UNION `sb_uwge_policy_contracts`), `SCHEME_NAME` (WIP vs `uwpl_policy_bases` with `top_indicator='Y'`), `DEPENDANT` (insured type `D` in `UWPL_RISK_PERSON`), `ISSUE_DATE`, `SIGN_ISSUE_DATE` (from `UWGE_VI_POL_REPORT_PARAM`) and `ASP_CUTOFF_IND` by comparing issue date to `SAPM_SYS_CONSTANTS` (`ASP_CONFIG_DATE` / `ASP_CUTOFF_DATE`). It re-implements the Java/jrxml gate in SQL.
- `AGICPROD~4.sql`: finds up to 4 POSCOVER contracts (product `070504`) with a dependant and issue date after the ASP cut-off.
- `PolicyDocuement\practice.sql`: "to check when new ASP cutoff created", "checking any contract comes under issue_date greater than 7/3/22 to get the ASP cutoff response", "to get the right companyParamType" (`SELECT DISTINCT COMPANY_TYPE FROM UWGE_COMPLAINT_REDRESS`).
Note: here "ASP" in ASP_CUTOFF means a Template12 product (code `070412`) in Landscape bands 1 to 3; do not mix it up with your ASP EndtDocument module until you confirm what ASP stands for [CONFIRM].

### 2.6 Producing a PDF (EVIDENCE: `PDF_Generation_Procedure.txt`)
1. Prepare `encode_amit.ps1` (input `input.xml`, output `base64.txt`).
2. Call `GenerateReport` with the right `templateId`, `boResId`, `batchConfigId` (taken from the SC team; the note lists the T25 template id and a Plines `boResId` and `batchConfigId`).
3. Keep only `<transactions>...</transactions>` from the response, run the PowerShell script to Base64.
4. POST to the SmartComm `generateDocument` endpoint (`<redacted>`) with `{"batchConfigResId":..., "transactionData":"<base64>", "tags":{"BU_GROUP":"AGIC_OPUSM","APP_COMPONENT":"UW"}}`.
5. Base64-decode the `data` part (Notepad++ MIME Tools).
6. Save as `.pdf` and open it. Outcome evidenced: `sample.pdf`, 8 pages. What was checked on the PDF is not recorded [CONFIRM]. No Jasper-vs-SmartComm side by side file exists in these folders.

### 2.7 End-to-end coverage matrix after the merge (EVIDENCE: `dynamic_merge\contract_id_and_responses\contract_id_details_.txt`)
11 contracts (12661510, 17139251, 19353150, 19733243, 22659357, 2866088, 33882322, 60284301, 60285301, 6235612, 6623314) were run through PD, PD2 and Landscape with a legend: fired with real data / gate met but inconclusive / not fired or not evidenced. Examples:
- 17139251 (ASP product 070412, `aspCutoffInd=N`): PD default CIPrint + PerilsClauses + ComplaintRedress fire, PD2 `debitNoteNo` real, Landscape bands 1, 2 and 11 all fire together. Ranked best single contract with 19353150 (ATC 070406, real nominee) and 6235612 (ELITESHIELD, Band 7 real `weeklyBenefit`/`casualtyCashBenefit`).
- 60284301 (`aspCutoffInd=Y`, "opposite of 17139251"): Landscape's own dispatch fetch returned a different `aspCutoffInd` than PD's value (a finding).
- 22659357: Landscape gate met but no unique-field evidence, "cannot confirm". 12661510 (ASTRO): no Landscape evidence. 19733243 (POSCOVER with dependant): PD2 and Landscape "not evidenced this call". 60285301 (PA4A): "ServiceTaxInvoicePA4A confirmed impossible (Gap 1)".
This is an honest coverage map with explicit gaps. Use it to say what was and was not confirmed.

---

## 3. Per module: test cases, defects, mapping to your commits

Commit map reference: `14_My_Commits_And_Ownership.md`. Dates below come from file times; commit dates are from that file.

### 3.1 RejectRisk

Test cases (EVIDENCE)

| TC | Input | Expected (Oracle, jrxml SQL) | Actual |
|---|---|---|---|
| RR-1 | name `LEE KEN439275`, id `671207082757`, type C, reason 09, 01/07/2017 to 30/09/2017 (`rejectrisk_run_q1.sql`) | `REJECTRISK_SQL1-3.png`: **2 rows fetched**. Row A: created by `RAIHANAWATI_DAHARI`, 03-AUG-17 10:39:31, `REVOKE_IND=Y`, duration `P` = PERMANENT, revoke date 10:45:00.338, updated 10:45:00.539. Row B: created by `MD_AMRAN`, 03-AUG-17 10:22:04, `REVOKE_IND=N`, nulls for revoke fields, updated 10:24:20.229. Both: NRIC, type `C` = CUSTOMER PARTNER DETAILS, reason 09 = REFERRED POLICYHOLDER. | Compass: 2 documents (first shown is `MD_AMRAN`, `createdDate 2017-08-03T10:22:04.000+00:00`). Service log: "RejectRiskService: 2 rows fetched". Postman (Jul 8): 200 OK, 2 `plineRejectRiskDetails`, same names, ids, remarks, flags. **Match on count and values.** |
| RR-2 | name `TAN T357766`, id `841004017669`, window 06/06/2017 to 31/08/2017 | only `payload.txt` notes (created 06-JUL-17) | `1357766.xml` has 1 row (`MD_AMRAN`, created 06/07/2017, `revokeInd N`). No SQL or Compass screenshot for this case. Partially evidenced. |
| RR-3 | ReportHeader lookup | `REJECTRISK_SUB_REPORT_SQL.png`: `DATA_CLASS = 1`, 1 row in 0.056 s | Compass doc in `RPGE_RPT_TEMPLATE`: `DATA_CLASS "1"`, `BATCH_SIZE -1`, `DISALLOW_COPY_IND "N"`; Postman response header has `<dataClass>1</dataClass>`. **Three-way match.** |

Observations (what comparison showed; EVIDENCE for the numbers, INFERENCE for the causes)
1. **Timestamps differ by 5:30 in the first Postman run.** Oracle/Compass show 10:39:31 and 10:22:04 (Compass with `+00:00`); the Jul 8 Postman response shows `Thu Aug 03 16:09:31 IST 2017` and `15:52:04 IST`. INFERENCE: Oracle values were loaded into Mongo as UTC instants and Java `Date.toString()` printed them in the JVM zone (IST). In the later Aug/Sep XML (`2439275.xml`) `createdDate` is `03/08/2017` (date only, `dd/MM/yyyy`), which also hides the issue. This links to the `dd/MM/yyyy`-versus-`xs:dateTime` caveat in the pack index. Whether a time zone fix was applied is not evidenced.
2. **`rowNumber` is 1 for both rows** in the Postman response and in the Compass output, whereas Oracle `ROWNUM` gives 1 and 2. The console log shows `"ROW_NUMBER" : 1` as a constant `$addFields`; `mongo_compass_aggregate.txt` (Jun 25) has no row-number stage at all. The `2439275.xml` (Sep 1) still shows `<rownumber>1</rownumber>` twice. The `$setWindowFields` fix mentioned in `14_My_Commits...` (commit `cc6a1b84d`, and in your uncommitted working tree) is not visible in the saved responses. [CONFIRM whether the final run shows 1 and 2].
3. **Row order differs.** Oracle returned the `RAIHANAWATI_DAHARI` row first, Compass and Postman returned `MD_AMRAN` first. Both rows have the same `NAME_EXT`, and the SQL sorts only by `NAME_EXT`, so the tie order is undefined on both sides. Parity on content, not on order.
4. **Parameter and shape evolution.** June/July: `pDateFrom/pDateTo`, flat `plineRejectRiskDetails` (one per `reportData`). Aug 31/Sep 1: `pCreatedDateFrom/pCreatedDateTo`, XSD-shaped output: `prntDetails > transDetList > transDet` with `rejectRiskType` and `rejectRiskTypeCode` nested under `riskList > risk`, `reasoncode`, `reasonCodeDescription`, `rownumber`, `companyDesc` blocks (remarks appear to be mapped into `companyDesc`), dates as `dd/MM/yyyy`. Mapped in `RejectRisk_Jrxml_Table_Column_Mapping.csv` (for example `REJECT_RISK_TYPE_CODE` to `rejectRiskTypeCode`, `ID_VALUE1` to `idNo1`, `CP_PART_ID` and `CP_PART_VERSION` "join predicate only, not mapped").
5. The Mongo pipeline in the evidence mirrors the Oracle joins: `$lookup` into `CPGE_VI_PARTNERS_ALL` on `PART_ID = CP_PART_ID` and `VERSION = CP_PART_VERSION`, three `$lookup`s into `CMGE_CODE` by `CAT_CODE` (`PLREJ_RISK_TYPE`, `PLREJ_REASON`, `PLREJ_REVOKE_DUR`) and `CODE_CD`/`CODE_DESC`, collection `CMUW_PL_REFER_RISK`, date `$gte` 2017-07-01 and `$lte` 2017-09-30.

Mapping to commits (INFERENCE from dates and content)
- `0d715fbad` (2026-09-01, OPUSM-41698; join key `CP_PART_ID`/`CP_PART_VERSION`, `CMGE_CODE` `CAT_CODE`/`CODE_DESC`, XSD nesting, date parsing): consistent with the Oracle SQL used as the reference (`CMUW_PL_REFER_RISK LEFT JOIN CPGE_VI_PARTNERS_ALL ON CP_PART_ID/CP_PART_VERSION`), with the Sept XML shape, and with `API_Crul_*` / `Mapping_sheet` dated Aug 31 to Sep 1. The folder gives no proof of the "join key did not exist" failure itself (no failing run saved).
- `cc6a1b84d` (2026-10-05, null date bound matching 0 rows): **not evidenced** in these folders (no 0-row run saved, no null-bound request). Do not claim test evidence for it.

Related example of judging "data issue, not code defect": `Documents\Data_mismatch_issue.docx` (outside the requested list). It says `UPLOAD_REF_NO` values in `CMGE_UPLOAD_FILE` (2017 to 2019) do not exist in `WIP_UWGE_POLICY_VERSIONS` (2024 to 2025 data), so the join returns 0 rows, and **both Oracle and MongoDB return 0 rows**, so it is a data issue. The report name is not given in the text I read [CONFIRM which report].

### 3.2 Template12 (PolicyDocument, PolicyDocument2, Landscape)

Test cases: sections 2.3 and 2.7 are the case list (inputs = contract and flags; expected = Oracle sub-report SQL; actual = Compass pipeline and Postman XML).
Representative screenshots read: `ParticularOfInsuredLandscape_AMC_postman_mongo_compass.png` (contract 25482538: Postman `particularOfInsuredLandscapeAmcDetails` and Compass both show `LIM AH KOW @ LIM CHE110171`, NRIC `550420011719`, Obesity ICD loading, `loadPct 25.00`, `covSeqRef 1.01`), `ParticularOfDiagnosisLandscape_AMC_sql1.png` (Oracle 2 rows for 25482538, two insured persons, "Obesity"), `ParticularOfInsuredLandscapeAsp_SQL_1.png` (Oracle for 17085732: 1 row, 070412, `ASP-B5`, age 43, `ACCOUNTANT (01)`), `T12_premium_debit_note_mongo_postman.png`, `TransitionalInvoice_postman_mongo.sql.png` (contract 12721508 data: policy `12AML0000001-00#000043`, Johor).

Defects and findings recorded in the workbooks and notes (EVIDENCE: `Notes / Issues Found` and `changes i made today.txt`)

| Area | Finding | Resolution recorded |
|---|---|---|
| Landscape band 1 (Insured ASP) | `POLICY_NO` field name wrong; `SHOW_OCCP_DESC` BSON type bug; `RB_AMT` null; `DATE_OF_BIRTH` / `JOIN_DATE` came as Object not String | Renamed to `POLICY_REF`, type fix, null guard, `dateToString` |
| Landscape band 4 (Benefits) | same date problem; `AGE` BigDecimal vs Integer; `RELATIONSHIP` lookup on `CMGE_CODE` | `dateToString`; Integer; numeric sort via `toDouble` |
| Landscape band 13 (Insured AMC) | `PARENT_NAME` mismatch because Oracle `RELATIONSHIP` was null (data quality) | Judged Mongo `$PARENT_NAME` correct; `parentName` uses `INSURED_NAME` when `RELATIONSHIP=SELF`; `insured` resolved by lookup on `CMGE_CODE` `INSURED_TYPE` |
| Landscape band 7 (Premium Elite) | fields missing from output | 12 DTO fields and `$project` mappings added to `PlineT12ParticularOfPremiumLandscapeEliteDto` and handler (`occupClass`, `riskId`, `designation`, `stiPlanCode`, `llbPlanCode`, `llbPrem`, `stiPrem`, `stiSi`, `llbSi`, `membershipNo`, `riskParentId`); plus per-row mirrors `planDescp`, `psi`, `pd`, `pdRb`, `personalLiability` in `PlineTransDet` so several bands do not overwrite one shared `reportParameters` slot ("last-band-wins") |
| PD2 TaxInvoince and TransitionalInvoice2 | 12 fields in the DTO were never populated by the query (`premDueRounded`, `loading`, `discount`, `mcoInpatientFee`, `mcoOutpatientFee`, `inpatientPrem`, `outpatientPrem`, `gstRateType`, `rebateAmt`, `gstPremCalcTemp`, `taxInvoiceNo`, `staxRounded` and so on) | Added `$project` mappings in `PlineTaxInvoinceHandler` and `PlineTransitionalInvoice2Handler` (note: "verified safe for the other 13 templates sharing this handler") |
| PD2 CompanyRegNo | external call failing | "Required 3 bug fixes in `PgRpgeUtilsClient` (SSL truststore, URL encoding, date format)" |
| PD2 PremiumDebitNote | `PlinePremiumDebitNote` handler did not join `UWPL_VI_POLICY_SCHEDULE_BASE` (PLAN, OCCUP_DESC, LOADING missing); T12 and T4 field sets differ (MCO/PTV vs PROMO) | New `PlineT12PremiumDebitNoteHandler` and `PlineT4PremiumDebitNoteHandler`; status still "Not started / work paused mid-investigation" in the sheet |
| PD2 PremiumDebitNote | `SHOW_SST_LABEL` is `N` in Mongo for many contracts but `Y` in Postman and Oracle | Open item (no fix recorded); candidate cause is data load, not proven |
| PD2 STaxInvoiceProductInfo | `TIN_NO`, `MASKED_TIN` empty in SQL dump but populated in Mongo/Postman | "need view team to confirm"; status "Not fully done" |
| PD2 ServiceTaxInvoice and others | sources moved from `UWGE_VI_POL_SCHEDULE_STI` / `UWPL_VI_POLICY_SCHEDULE_STI` to `UWPL_VI_TAX_INVOICE`; PLSub rewritten as `$unionWith` over WIP and posted `*_POLICY_BASES_STAX`; QR variant for EndtDocument2 | recorded in `changes_made_on_merged_code.txt` (file list of 50+ modified files) |
| Speculative change | a `UWGE_POLICY_CONTRACTS` fallback for `PRODUCT_CODE` "didn't fix the actual issue ... reverted fully" | reverted, `git diff` zero |

Mapping to your commits (INFERENCE): `b4e588bd4` (2026-08-11, `final_T12_code`, base service tweaks and handler renames) and `f79ecc811` (2026-08-20, `Template12PolicyDocumentBaseService`) are the only GS Template12 commits, and they are much smaller than the handler/DTO/service change list above. The big handler/DTO work in the notes (new PD2 handlers, Elite premium fields, TaxInvoince fields) is therefore not visible as GS commits in `14_My_Commits_And_Ownership.md`; it may be in uncommitted work, in the shared branch by others, or in `Template12PolicyDocument2Service` rewrites described there. The screenshot `PolicyDocument2\Screenshot 2026-08-11 220152.png` shows an AI-assistant terminal message: "Push succeeded - feature/retrofit-develop-R2-pline is now at 359d10f9 on origin, merging your final_T12_code commit with the 9 new remote commits (no conflicts)". That lines up with your merge commit `359d10f9a` and the first GS commit `b4e588bd4` (same date). Also note that `changes i made today.txt`, `changes_made_on_merged_code.txt` and `hardcoing_details` read like an AI coding assistant's summaries (for example "Cooked for 30s" at the end of one). Be ready to say how you used AI assistance and that you verified the results yourself [CONFIRM]. Do not present those summaries as your own authorship if you did not write them.

### 3.3 ASP

In `plines_testing` there is no ASP folder and no ASP screenshots. What exists:
- `jr-tools\UpdateStatus.java` hard-codes the status text of a checklist for an endorsement report tree (EVIDENCE of results, with the caveat that rows marked "(inferred)" were not directly confirmed): contract `24080537`, endorsement code `72`: Landscape returned add-member data; endorsement code `98`: Landscape returned delete-member data; EndtDocument returned data for code 72; company block empty, company name/RegNo blank; diagnosis fields empty (not fired); rows 17 to 28: "Not fired - EndtDocument2 call returned empty reportTransaction (2.xml 17:52)". Each line also appends " | Data check: <old text>" so the older value is kept.
- `jr-tools\b1-b3.txt`, `ids100909.txt`, `out1.txt`: Mongo probes (3 below) on `UWGE_POLICY_VERSIONS` for 103 contract ids of product `100909`, grouped by `ENDT_CODE`, to find a contract per endorsement code, and `count` of 16 views/collections (`UWPL_VI_ENDT_ADD_DEL_RISK`, `UWPL_VI_ENDT_SCHEDULE_TPL4`, `UWPL_VI_ENDT_DEBIT`, `UWPL_VI_ENDT_TAX_INV`, `UWGE_GST_TAX_INV`, `SAPM_COMPANY`, `CMGE_CODE` and others) to check that the source data exists in Mongo.
- Outside the requested folders: `ASP_contract_reports_mapping.xlsx` lists real endorsement test records, for example contract `27275348` endt `22PKA0000048-00*002`, version 3, with "Reports Fired" `PrintSeriesFooter [ED]; EndtCover [ED]; ... EndtScheduleOfBenefits_ASP [ED]; ASPLLB; ASPSTI; ComplaintRedress; DebitNote [ED2]; ServiceTaxInvoice [ED2]; STCompanyContact [ED2]` and `StaxNo STDN22036000005718`; and `ASP_contract_id_list.xlsx` marks 9 EndtDocument sub-reports "Done" for 27275348. Postman/SQL/Compass screenshots for ASP were not in the folders I was told to read [CONFIRM where they are].
- The CONTRACT_ID type mismatch (String versus BSON Long): the `MongoProbe` batch files query with a **numeric** literal (`{"CONTRACT_ID":23913536}`), while Oracle helper SQL always compares quoted strings (`contract_id = '27533722'`, which Oracle converts silently). This shows the exact trap and is consistent with your fix in `578778080`, `ba9e29586`, `7a925920c`. No saved before/after run of that bug exists in these folders (INFERENCE only for the link).

### 3.4 Defect-to-commit table (summary)

| Defect (source file) | Likely commit | Strength |
|---|---|---|
| RejectRisk join/CMGE_CODE/XSD shape, parameter rename (`payload.txt` vs `API_Crul_*`) | `0d715fbad` | medium (dates and content align; no failing run saved) |
| RejectRisk null date bound, +1 day `$lt`, rowNumber window | `cc6a1b84d` | not evidenced here |
| CONTRACT_ID String vs Long | `578778080`, `ba9e29586`, `7a925920c` | indirect (MongoProbe numeric ids) |
| Landscape date `Object -> String`, `POLICY_REF`, `AGE` type | none of the GS commits is Landscape-specific; `cc6a1b84d` restructured Landscape rows | weak |
| PD2 handler/DTO field gaps | no GS commit lists them | do not claim by commit |

---

## 4. What level of testing this is (say it exactly like this)

- It is **manual, developer-driven data-parity testing (SIT-style, on a developer machine)**: the service ran locally, called by Postman, reading a real Mongo database, compared against Oracle query results of the original jrxml, tracked in an Excel checklist with screenshots.
- It is **not** automated unit testing, **not** an independent QA cycle, **not** UAT, **not** regression automation, and **not** formal SIT. There is no test ID scheme, no signed defect log, no environment promotion, no tester other than you, and no performance or negative testing in these folders.
- Oracle SQL result = the expected result. That is a **proxy oracle** (the jrxml query with literals). It does not prove the final PDF is identical to the legacy Jasper PDF.
- Coverage is purposive: one or two chosen records per branch, not all data. Sections 2.3 and 2.7 show known gaps (T4 landscape bands, several PD2 bands, PDPA, NominationForm).
- Hardcoded and fabricated data proved wiring only.
- Say the sentence: "I did manual data-parity testing at report-section level: for each sub-report, same record in Oracle, in Mongo Compass, and through the API in Postman, and I tracked it in a checklist with screenshots. I did not do automated integration testing; the unit tests are separate."
- Do not say "SIT passed", "UAT", "100 percent coverage", "all bands verified".

---

## 5. What the PNG screenshots show (important ones)

121 PNG in total (EVIDENCE: I opened 20; others follow naming).

RejectRisk
- `REJECTRISK_SQL1.png` (2026-06-19): SQL Developer, last lines of the query (`REASON_CODE = '09' OR '09' IS NULL ... ORDER BY NAME_EXT ) REJECT_RISK;`), grid: **All Rows Fetched: 2 in 0.649 seconds**, columns `ROWNUMBER` to `REMARKS`; both rows `LEE KEN439275`, NRIC, `671207082757`, type `C` CUSTOMER PARTNER DETAILS, reason 09 REFERRED POLICYHOLDER.
- `REJECTRISK_SQL2.png`: scrolled right, shows `REMARKS` (one row ends "...AT SCENE OF BREAKDOWN VEHICLE REQUESTED TOWING WAS A PROTON WIRA.") and `CREATED_BY` (`RAIHANAWATI_DAHARI`, `MD_AMRAN`), `CREATED_DATE`.
- `REJECTRISK_SQL3.png`: right-most columns: `REVOKE_IND` Y/N, `REVOKED_DURATION P`, description `PERMANENT`, revoke by/date, `UPDATED_BY`, `UPDATED_DATE`.
- `REJECTRISK_SUB_REPORT_SQL.png`: ReportHeader query on `CUSTOMER.RPGE_RPT_TEMPLATE`, 1 row, `DATA_CLASS 1`.
- `Mongo_compass_aggre_result_main_report.png`: Compass aggregation (stages `$addFields $match $match $lookup ...+5`), TEXT view, "Sample of 2 documents": first doc `rowNumber 1`, `LEE KEN439275`, `MD_AMRAN`, `createdDate 2017-08-03T10:22:04.000+00:00`, `revokeInd N`, `updatedDate 10:24:20.229`.
- `Mongo_compass_aggre_result_sub_report.png`: Compass find on `RPGE_RPT_TEMPLATE` (filter `DOC_TYPE`, `TEMPLATE_NAME`, `DESCRIPTION`) returning 1 document, `DATA_CLASS "1"`.
- `REJECTRISK_POSTMAN.png`: Postman raw JSON request (`pDateFrom 01/07/2017`, `pDateTo 30/09/2017`, `pCustomerName`, `pIdNo`, `pRejType C`), response **200 OK, 965 ms, 19.39 KB**, `"status":"SUCCESS"`, `<preportTitle>PLINES : REJECT RISK REPORT</preportTitle>`, echoed parameters.
- `REJECTRISK_CONSOLE.png` (2026-07-07 18:55): log lines `RejectRiskService: dateFrom=01/07/2017 dateTo=30/09/2017 customerName=LEE KEN439275 idNo=671207082757 rejType=C reasonCode=09`, `RejectRiskHandler createdDateFrom=...`, the full `MongoTemplate` aggregation, and `RejectRiskService: 2 rows fetched` (about 0.5 s after the call).
- Inside the docx, three extra images: a second Postman response (200 OK, 836 ms, 18.11 KB), the Jun 25 console log with `ReportHeaderHandler reportName=AGIC/UW/Plines/RejectRisk/RejectRisk pReportTitle=PLINES : REJECT RISK REPORT`, and a Compass view of `CMUW_PL_REFER_RISK` (228 documents).

Template12
- `Landscape\Main_report_console.png` (2026-07-13): DEBUG `MongoTemplate` aggregation for contract `39765306`: `$addFields CID`, `$lookup` into `WIP_UWPL_RISK_NOMINEE` and `UWPL_RISK_NOMINEE` (with version checks), diagnosis lookups, `PC_WIP / PC_SB / PC_LIVE` product code lookups to compute `NOMINEE_PRINT`, `DIAGNOSIS_PRINT`. This is the dispatch row: the Mongo pipeline mirrors the Oracle `landscape_main_report_Q.sql` printing flags.
- `Landscape\ParticularOfInsuredLandscape_AMC_postman_mongo_compass.png`: split screen, Postman XML (200 OK, 307 ms, 44.57 KB) next to Compass results (2 documents), contract 25482538.
- `Landscape\ParticularOfInsuredLandscapeAsp_SQL_1.png`: Oracle `customer.UWPL_VI_POLICY_SCHEDULE_TPL4`, contract 17085732, 1 row in 1.615 s.
- `Landscape\ParticularOfDiagnosisLandscape_AMC_sql1.png`: Oracle, 2 rows in 0.338 s, one per insured person.
- `PolicyDocument2\TransitionalInvoice_postman_mongo.sql.png`: Postman 200 OK, **3.52 s, 130.05 KB** (search "52 of 115" for TransitionalInvoice), with Compass preview (policyNo `12AML0000001-00#000043`, `accountNo JB08483`, `effDate 2014-01-01`, `taxInvoice TAX18126000000166`). Postman shows `EFF_DATE` as `Wed Jan 01 05:30:00 GMT+05:30 2014`, a Java `Date.toString()` form, another sign that dates were not yet XSD format at that time.
- `PolicyDocument2\T12_premium_debit_note_mongo_postman.png`: Postman `plineT12PremiumDebitNoteDetails` (`THAM MAY LING`, PETALING JAYA, `PERLINDUNGANKU ALLIANZ4ALL`) beside the Compass preview with the same fields, `debitNote D22PPN5001941-00`, `grossPremium 75.00`.
- `PolicyDocument2\Screenshot 2026-08-11 220152.png`: terminal message about a successful push at `359d10f9` (see 3.2).
- Remaining Landscape names show per-band triples (`*_sql*.png`, `*_mongo_compass*.png`, `*_postman*.png`) for Benefits, Benefits Elite, Benefits Income, Premium ASP, Premium Elite, Premium Income, Sponsor, Nomination (generic, Elite, POS, AZP1, AZP2), Diagnosis (generic and AMC). Remaining PD2 names cover CompanyContact, signature, STaxInvoiceProductInfo (and PA4A version), STCompanyContact, QRSection, ServiceTaxInvoicePL / PLSub, TaxInvoince, TaxInvoiceProductInfo, TransitionalInvoice(2) and ProductInfo, T4 and T12 premium debit note, T4 and T12 signature. I did not open each one.

---

## 6. `jr-tools` and how they supported the work

| Tool | What it does | How it helped |
|---|---|---|
| `MongoProbe.java` | Read-only batch client (commands `collections`, `find`, `count`, `distinct`, `agg`; tab-separated lines, `#` comments; 120 s max time) using the Mongo sync driver 5.6.0; reads the Mongo URI and database name from `application.properties`; prints `ERR ...` per failing line | Quick queries on live Mongo without opening Compass: find sample ids, confirm collections exist and have data, check types. |
| `mp.sh` | Wrapper: builds a classpath from `~/.m2` jars (`mongodb-driver-sync`, `-core`, `bson` 5.6.0), runs `java MongoProbe.java <props> <batchfile>`, filters SLF4J noise | One-line way to run a batch (`mp.sh b3.txt`). |
| `b1.txt`, `b2.txt`, `b3.txt` | Batch files: one `find` on `UWGE_POLICY_VERSIONS` for a contract; one `find` of 400 `UWGE_POLICY_CONTRACTS` with `PRODUCT_CONFIG_CODE "100909"`; an aggregation over `UWGE_POLICY_VERSIONS` with `ENDT_NO != null` grouped by `ENDT_CODE`, collecting contracts; 16 `count` checks on endorsement views and code tables | Choose test contracts per endorsement code for the endorsement/landscape reports; sanity-check that Mongo has the replicated views. |
| `ids100909.txt`, `out1.txt` | 103 contract ids (product 100909) and the probe output | Candidate pool and evidence of the numeric `CONTRACT_ID`. |
| `ReportTree.java` | Walks jrxml files (`<reportsRoot> <mainFolder> <outDir>`), parses `subreport` elements and expression strings, resolves `$P{}` defaults and passed parameters, detects NOT FOUND and unresolved sub-reports, classifies Main / Generic / Common / Specific, and writes `AHI_Report_Structure.csv` and `.xlsx` with columns including "Testing Status", "Query Creation", "Response Status" | Built the sub-report hierarchy and a ready-made tracking sheet for each main report (default folder in the source is `AHIEndtSchedule`). The same idea produced your `*_Hierarchy.csv` files. |
| `UpdateStatus.java` | Rewrites only column G (Status) of sheet1 inside an xlsx, copies other parts byte for byte, appends old text as "| Data check: ..." | Records per-sub-report outcomes (Fired / Fired (inferred) / Not fired / Not confirmed) into the endorsement checklist, preserving earlier "data check" notes. |
| `XlsxDump.java` | Prints cells of an xlsx (no Excel or Python needed) | Reading checklists and mapping sheets in a locked-down environment. |

Fair statement: these are **support utilities for test-data discovery and bookkeeping**, not automated assertions. No tool compares outputs automatically.

---

## 7. How this changes the earlier statement in `05_Testing_Deep_Dive.md`

Earlier statements (from reading code and the repo only): "SIT: Not in repo"; "Integration is done by calling the deployed service in a dev environment" (assumption); "A Jasper-vs-new-service comparison happens manually" (inference from branch names); "Evidence of no test run".

Updated position:
- Replace "no integration/SIT evidence" with: "**No automated integration or SIT evidence in the repo, but there is substantial manual integration evidence outside the repo**: Postman calls to a locally running service against real Mongo data, compared with the original jrxml Oracle SQL and with Mongo Compass, tracked in checklists with 121 screenshots."
- Confirmed (was assumption): comparison against the legacy side exists, and it uses the Oracle jrxml SQL, not Jasper PDF output.
- Still true: no Testcontainers or embedded Mongo tests, no coverage evidence, no automated parity test, no recorded unit test run, no QA/UAT sign-off.
- Environment correction: the evidence is a **local service** (`localhost:8080`) on your machine, not the shared dev environment [CONFIRM which Mongo/Oracle environment, shared dev or a copy].
- The "Regression" row in file 05: the old/new response folders are a manual before/after regression of payloads, not a regression suite.

---

## 8. Interview questions grounded in this evidence (25)

1. **How did you test your reports?** Strong: "I compared three sources for the same record: the Oracle query from the original jrxml, the Mongo aggregation in Compass, and the API response in Postman, then fixed handler/DTO mismatches and kept a checklist with screenshots." Mistakes: saying "unit tests" or "SIT". Follow-ups: How many bands? What was the expected-result source? (Oracle SQL.)
2. **Why is Oracle SQL a valid expected result?** Strong: it is the query inside the jrxml, so it is the legacy data logic; but it is a proxy, not the legacy PDF. Mistake: claiming PDF parity. Follow-up: how would you prove PDF parity? (side-by-side render, text diff.)
3. **Walk me through RejectRisk testing.** Strong: chose `LEE KEN439275` (id 671207082757, type C, reason 09, 01/07/2017 to 30/09/2017); Oracle gave 2 rows (one revoked permanently, one not); Compass 2; log "2 rows fetched"; Postman 200 OK with 2 detail blocks; values matched. Mistake: citing numbers without the parameters. Follow-up: why two rows?
4. **What differences did you see between Oracle and Postman for RejectRisk?** Strong: timestamps shifted by 5:30 (UTC vs IST via Java `Date.toString`), `rowNumber` 1/1 against ROWNUM 1/2, and tie order. Mistake: saying "no differences". Follow-up: what did you do? (Be honest: [CONFIRM] fix status; later XML used `dd/MM/yyyy`.)
5. **How do you explain the 5:30 difference?** Strong: Oracle value 10:39:31 loaded as an instant in UTC; Java printed it in IST; it is a presentation issue, content equal. Label it as inference. Mistake: calling it a data bug. Follow-up: what should the XSD say? (`xs:dateTime` vs `dd/MM/yyyy` mismatch noted in the pack.)
6. **Why copy the aggregation from the console log into Compass?** Strong: isolates whether the issue is the pipeline or the DTO/mapping; same stages as the app. Mistake: retyping the pipeline by hand. Follow-up: how did you log it? (DEBUG on `MongoTemplate`.)
7. **What was the join you validated in RejectRisk?** Strong: `CMUW_PL_REFER_RISK` to `CPGE_VI_PARTNERS_ALL` on `CP_PART_ID`/`CP_PART_VERSION`; `CMGE_CODE` lookups by `CAT_CODE` (`PLREJ_RISK_TYPE`, `PLREJ_REASON`, `PLREJ_REVOKE_DUR`). Mistake: mixing up with `CUSTOMER_ID`. Follow-up: LEFT JOIN vs `$lookup` semantic (note that the Mongo pipeline then matches on partner fields, so unmatched rows drop).
8. **What does a Landscape "band" mean and how did you test each?** Strong: a sub-report dispatched by gates (product, scheme, print flags); for each band one contract fulfilling the gate, Postman/SQL/Compass Yes, screenshots. Mistake: "I tested all". Follow-up: which bands are untested? (T4 block T4-1 to T4-14 empty.)
9. **How did you find contracts for gates?** Strong: helper SQL such as `AGICPROD~3/4.sql` and `practice.sql` re-implementing `ASP_CUTOFF_IND` with `SAPM_SYS_CONSTANTS`, product code from posted and WIP tables, dependants from `UWPL_RISK_PERSON`; Mongo probes for endorsement codes. Mistake: "random contracts". Follow-up: why both WIP and posted tables?
10. **What is "hardcode for testing" and is it acceptable?** Strong: temporary forced values keyed on a contract id to fire a gate when no real data existed; marked with start/end comments; commented out afterwards; proves wiring, not correctness. Mistake: hiding it or saying it proves data. Follow-up: how do you ensure none reached production? (`git grep`, review [CONFIRM].)
11. **Which result came from fabricated data?** Strong: be specific: MST customer-details DTOs "INTEL ELECTRONICS ... MST TEST" and QR/PLSub fallbacks for 29735791 and 36880167. Mistake: counting them as Yes in coverage. Follow-up: how did you label them in the sheet? ("hardcoded").
12. **What bugs did testing find in Landscape?** Strong: `POLICY_NO` vs `POLICY_REF`, `SHOW_OCCP_DESC` BSON type, `RB_AMT` null, dates as Object not String (`dateToString`), `AGE` BigDecimal vs Integer, `RELATIONSHIP` sort via `toDouble`, AMC parent name. Mistake: claiming they were your commits without checking `git log`. Follow-up: how would you stop date-type issues recurring? (typed projections + test.)
13. **Oracle and Mongo disagreed on AMC `PARENT_NAME`. Who was right?** Strong: Oracle `RELATIONSHIP` was null (data quality), so Oracle's derived value was wrong; Mongo's was right; documented in the sheet. Mistake: "Mongo is always right". Follow-up: how did you decide? (source data inspection.)
14. **What does `SHOW_SST_LABEL` N in Mongo and Y in Postman/Oracle tell you?** Strong: data mismatch between the replicated Mongo copy and Oracle for many contracts; open item; I would compare record counts and last refresh. Mistake: changing code to force Y. Follow-up: who owns the data load?
15. **How did you check that TaxInvoince / TransitionalInvoice2 were complete?** Strong: DTO had 12 fields never populated by the query; found by comparing Oracle columns to the XML; added `$project` mappings; checked the shared handler used by 13 other templates. Mistake: not mentioning shared-handler risk. Follow-up: how did you prove no regression? (old vs new responses; limited, manual.)
16. **What is the old/new response folder for?** Strong: manual before-and-after regression of payloads (larger new files because more fields populated; same number of `reportData`). Mistake: calling it an automated regression suite. Follow-up: how would you automate it? (golden-file diff with ignore rules.)
17. **How did you handle the 11-contract end-to-end matrix?** Strong: ran PD, PD2, Landscape for each, recorded fired/inconclusive/not evidenced; best contracts 17139251, 19353150, 6235612; gaps stated. Mistake: hiding the gaps. Follow-up: what is Gap 1? (`ServiceTaxInvoicePA4A` impossible with the data found.)
18. **How did you produce a PDF?** Strong: GenerateReport XML to Base64 PowerShell script to SmartComm `generateDocument` to decode `data` to PDF; one sample of 8 pages. Mistake: claiming Jasper-vs-PDF diffs. Follow-up: how would you compare PDFs?
19. **What did the jr-tools do?** Strong: MongoProbe for read-only probes, ReportTree to map sub-report trees and make tracking sheets, UpdateStatus/XlsxDump to maintain xlsx. Mistake: "automated test framework". Follow-up: why read-only? (safety on shared Mongo.)
20. **How does CONTRACT_ID typing show up in your test method?** Strong: Oracle accepts `'27533722'`, Mongo has a number, so string filters return 0 rows silently; MongoProbe uses numeric literals; fix in handlers. Mistake: no mention of silent zero rows. Follow-up: which commits? (`578778080`, `ba9e29586`, `7a925920c`.)
21. **How do you know a zero-row result is a code bug or data issue?** Strong: run the same filters on Oracle and Mongo; if both are zero it is data (the 2017 to 2019 upload refs not in 2024 to 2025 WIP versions example); if only Mongo is zero, suspect type/gate. Mistake: assuming code is wrong. Follow-up: next step if only Mongo is zero?
22. **What testing did you not do?** Strong: automated integration, Jasper PDF diff, XSD validation run, negative/performance tests, QA/UAT; T4 landscape bands; several PD2 bands. Mistake: overclaiming. Follow-up: what would you do first? (golden XML plus XSD validation in CI.)
23. **Postman showed 3.52 s and 130 KB for TransitionalInvoice. Does it concern you?** Strong: PD2 responses are large (up to 2.7 MB merged); this is a local single-call timing, not a load test; watch heavy `$lookup`/`$unionWith`; the earlier timeout fix in `AspDebitNoteHandler` is the precedent. Mistake: quoting it as performance testing. Follow-up: how would you profile? (`explain`, indexes.)
24. **How do unit tests relate to this evidence?** Strong: unit tests (mocked `MongoTemplate`) test pipeline stages; this manual work tests real data and mapping; they complement. Mistake: mixing the two. Follow-up: how would you turn a screenshot case into a test? (seeded Mongo, expected XML.)
25. **If I asked you to repeat RejectRisk tomorrow, what would you change?** Strong: fix and assert `rowNumber` and ordering, add a null-date and no-filter case, save the Oracle result as CSV and diff automatically, run XSD validation. Mistake: "nothing". Follow-up: what is the expected rowNumber for ties?

---

## 9. 90-second spoken answer: "How did you test your reports?"

"I tested at report-section level with manual data-parity checks. For each sub-report I picked a real record that satisfied the report's conditions, using helper queries in Oracle to find them. Then I did three checks on the same record. First I ran the original jrxml query in Oracle SQL Developer. Second I took the Mongo aggregation from my service's debug log and ran it in MongoDB Compass. Third I called the Java API from Postman on my local machine and looked at the XML for that section. I compared row counts and field values, and I kept a checklist with Postman, SQL and Compass columns, plus screenshots.

For RejectRisk, for example, one customer in July to September 2017 gave two rows in all three places, one permanently revoked and one not, and the service log also said two rows fetched. The differences I noticed were presentation: a five-and-a-half-hour time shift on timestamps, and row numbers and ordering. For Template 12 Landscape, this found real mapping issues like wrong field names, date and number types, and missing fields, and I fixed them in the handlers. [CONFIRM which of those fixes were yours.]

Where no real contract could trigger a section, I used temporary hard-coded test values, and I treat those as wiring checks only. This is manual testing, not automated integration tests or UAT, and I did not compare the final PDF with the Jasper PDF, apart from rendering one sample through SmartComm. [CONFIRM: ASP testing details, since my folder has no ASP screenshots.]"

---

## Summary of the most important findings

1. Real manual evidence exists: Postman + Oracle SQL Developer + MongoDB Compass triples, 121 screenshots, checklists with Yes/No per column (RejectRisk 8 screenshots, Landscape 69, PD2 44). It is local, developer-run data parity, not automated SIT.
2. RejectRisk is the best-evidenced module: 2 rows in Oracle, Compass, service log and Postman for `LEE KEN439275`; values match; observed differences are the +5:30 time shift, `rowNumber` always 1, and undefined tie order. The null-bound date fix (`cc6a1b84d`) is not evidenced.
3. Template12 findings are documented in workbook notes: field-name, BSON type, null and date-type bugs in Landscape; unpopulated DTO fields in TaxInvoince and TransitionalInvoice2; `PgRpgeUtilsClient` fixes; open items (`SHOW_SST_LABEL`, `TIN_NO`). The big PD2 handler/DTO work does not appear in your two Template12 GS commits, so check ownership before claiming.
4. Hard-coded "testing" blocks and fabricated fallback DTOs were used to fire gates; they were later commented out. State them honestly as wiring checks.
5. No ASP folder in `plines_testing`; ASP evidence is indirect (jr-tools status text for contract 24080537, MongoProbe numeric `CONTRACT_ID`, and extra workbooks outside the requested folders).
6. There is no Jasper-PDF-versus-new comparison and no XSD validation run in these folders; one 8-page SmartComm PDF exists.
7. Several of the notes read like AI-assistant summaries; be ready to explain how you used AI and what you verified yourself.
