# 03 - ASP Reports Deep Dive (PLine ASPEndtSchedule: EndtDocument, EndtDocument2, AddMemberLandscape, DeletionMemberLandscape)

Purpose: interview-ready knowledge transfer for the PLine ASP endorsement reports, built only from evidence in the repo and the Jasper dumps.

Conventions used in this file
- "J:" = Jasper dump file. Folder root `J:\...\resources\reports\AGIC\UW\Plines\ASPEndtSchedule` is written as `ASP/`. The real jrxml source sits in the `*.jrxml.data` files (the `*.jrxml.xml` files are only 500-byte descriptors). Main reports: `ASP/EndtDocument_files/EndtDocument_jrxml.data`, `ASP/EndtDocument2_files/EndtDocument2_jrxml.data`, `ASP/AddMemberLandscape_files/AddMemberLandscape_jrxml.data`, `ASP/DeletionMemberLandscape_files/DeletionMemberLandscape_jrxml.data`.
- "Java:" = `src/main/java/com/agcs/fo/opus/plines/...` in `opus-documentprintdata-service`. Line numbers are from the working tree at branch `feature/retrofit-develop-updated` (HEAD 861f0666c).
- "(inference)" = my reading, not stated anywhere in the code or reports. "(verified)" = I read both sides.
- No secrets are copied. (I did not open any Mongo URI or truststore config.)

---------------------------------------------------------------------------------------------------

## 0. One-page summary (what to say first in an interview)

1. Four Jasper top-level reports exist for ASP endorsements: `EndtDocument` (cover + schedule + schedule of benefits), `EndtDocument2` (complaint page, debit/credit note, tax invoices, service tax invoice, transitional invoices), `AddMemberLandscape` (landscape tables for added members, sponsors, nominees) and `DeletionMemberLandscape` (landscape table for deleted members).
2. In Java they are ONE Spring bean: `AspEndtDocumentService`, bean name `AGIC/UW/Plines/ASPEndtSchedule/EndtDocument` (Java: service/AspEndtDocumentService.java:165). The class Javadoc (lines 81-163) says it is a MERGED bean. The older `AspEndtDocument2Service` has its `@Service` commented out (AspEndtDocument2Service.java:44) and `AspEndtDocumentLandscapeService` has both `@Bean` methods commented out (AspEndtDocumentLandscapeService.java:620 and :625).
3. The response is "1 subreport = 1 reportData": element 0 is a master row (`mainReportName=ASPEndtScheduleEndtDocument`, `subReportName=PrintSeriesFooter`), then one reportData per subreport that fired, then landscape rows (AspEndtDocumentService.java:1386-1498).
4. Biggest gaps vs Jasper (details in section 4): ASPLLB / ASPSTI / EndtScheduleOfBenefits_ASP are stubs (no reportData emitted); landscape dispatch always runs; INST_IND cut-off constant is not computed; ORA-01427 emulation exists only in `AspServiceTaxInvoiceHandler`; main handler anchor-collection risk; EndtCover SQL simplification.
5. "ASP" is never expanded in the jrxml. The only evidence is internal report names `ASPlusSTI` and `ASPlusLLB` and benefit codes `ASP-A1` ... `ASP-D5`, plus product code 070412 in Java comments (section 5).

---------------------------------------------------------------------------------------------------

## 1. Report trees at every level

### 1.1 Hierarchy files (verbatim)

`ASP/EndtDocument_files/EndtDocument_ReportHierarchy.csv`
```
EndtDocument,PrintSeriesFooter,
EndtDocument,EndtCover,CompanyNameWithGSTRegNo
EndtDocument,Blank,
EndtDocument,EndtSchedule,EndtNarration
EndtDocument,EndtSchedule,Signature
EndtDocument,EndtScheduleOfBenefits_ASP,ASPLLB
EndtDocument,EndtScheduleOfBenefits_ASP,ASPSTI
```
`ASP/EndtDocument2_files/EndtDocument2_ReportHierarchy.csv`
```
EndtDocument2,ComplaintRedress,CompanyNameBRNUpperAndLowercase
EndtDocument2,DebitNote,CustomerDtlsOfMasterPolicy_Debit
EndtDocument2,Blank,
EndtDocument2,EndtTaxDoc,CompanyContactJr
EndtDocument2,EndtTaxDoc,AntiRasuahBanner
EndtDocument2,ServiceTaxInvoice,STCompanyContact
EndtDocument2,ServiceTaxInvoice,CustomerDtlsOfMasterPolicy
EndtDocument2,ServiceTaxInvoice,AntiRasuahBanner
EndtDocument2,ServiceTaxInvoice,ServiceTaxInvoiceEndtMT
EndtDocument2,ServiceTaxInvoice,ServiceTaxInvoiceEndtMTSub
EndtDocument2,ServiceTaxInvoice,QRSection
EndtDocument2,TransitionalInvoice,CompanyContactJr
EndtDocument2,TransitionalInvoice,SSTNote
EndtDocument2,TransitionalInvoice2,STCompanyContact
EndtDocument2,TransitionalInvoice2,SSTNote
```
`AddMemberLandscape_files` and `DeletionMemberLandscape_files` have NO hierarchy csv (only `*_jrxml.data`). Their trees below come from the jrxml `subreportExpression` entries.

### 1.2 EndtDocument (page 595x852, `ASP/EndtDocument_files/EndtDocument_jrxml.data`)

Main report parameters (name : class): pEndtNo String (default empty), pRefNo String, pContractId **Integer**, pPolicyNo String, companyParamType String, showCoverNote String, showComplaint String, showEndtSchedule String, showDebitNote String, showTaxInvoice String, SUBREPORT_DIR / SUBREPORT_DIR2 (= `repo:/reports/AGIC/UW/Plines/ASPEndtSchedule/`), pCertInd, checklistCopy, versionNo Integer, printSeries, copyType, pActionCode, pFirstPageNumber Integer, ShowGstLabel, ShowSstLabel, SUBREPORT_DIR3 (= `.../Plines/PolicySchedule/Template4/`), pAtceInd, issueDate String, SUBREPORT_CR (= `repo:/reports/AGIC/UW/Generic/`), pSchemeName.

Main query ("select 1 ... from dual"): one row with these scalar sub-selects (verified):
- PRODUCT_CODE = DISTINCT product_config_code from (WIP_UWGE_POLICY_CONTRACTS where contract_id and rownum=1 UNION ALL **SB_UWGE_POLICY_CONTRACTS** where contract_id and rownum=1).
- SHOW_GST_LABEL, SHOW_SST_LABEL, ISSUE_DATE = from `UWGE_VI_ENDT_REPORT_PARAM ERP` where `ERP.CONTRACT_ID = pContractId AND (ERP.ENDT_NO = pEndtNo OR ERP.REF_NO = pRefNo)` (no ROWNUM, so more than one row would raise ORA-01427).
- ENDT_CODE = `WIP_UWGE_POLICY_VERSIONS` (when checklistCopy='Y') UNION ALL `UWGE_POLICY_VERSIONS` (version_no = versionNo, when checklistCopy='N').
Fields: "1" BigDecimal, PRODUCT_CODE, SHOW_GST_LABEL, SHOW_SST_LABEL, ISSUE_DATE (Timestamp), ENDT_CODE.

Background band: sub-report `PrintSeriesFooter` (`repo:/reports/AGIC/UW/Generic/PrintSeriesFooter.jrxml`), params pVersionNo, pContractId, pPrintSeries; its query is `select PRN_CNT from UWGE_POLICY_VERSIONS_CTRL where CONTRACT_ID and VERSION_NO` and it prints `printSeries-PRN_CNT` only if printSeries is not null/"null" (J: Generic/PrintSeriesFooter.jrxml.data:15,25). A text field prints `"PAGE: " + (PAGE_NUMBER - 2)` when PAGE_NUMBER > 2.

Detail bands, in order (each band has the same printWhenExpression on the band and on the subreport element):

| # | Sub report (file) | Exact print condition | Parameters passed (names) | Own SQL / data | Nested sub-sub reports |
|---|---|---|---|---|---|
| 1 | EndtCover.jrxml | `$P{showCoverNote}.equalsIgnoreCase("Y")` | SUBREPORT_DIR, pPolicyNo, companyParamType, pContractId, issueDate (=`new SimpleDateFormat("dd-MMM-yyyy").format($F{ISSUE_DATE})`), pEndtNo, checklistCopy, pRefNo | `SELECT UPS.* ... FROM UWPL_VI_ENDT_SCHEDULE_SUMMARY UPS, DMT_AGENTS DMA, DMAG_AGENTS DMAG, CMDM_BRANCH BR, DMAG_RI DMR WHERE UPS.AGENT_CODE = DMA.REFERENCE_CODE(+) AND DMA.INT_ID = DMAG.INT_ID(+) AND BR.BRANCH_CODE = NVL(DMAG.BRANCH_CODE, BR.BRANCH_CODE) AND DMA.INT_ID = DMR.INT_ID(+) AND BR.BRANCH_CODE = NVL(DMR.BRANCH, BR.BRANCH_CODE) AND (UPS.ENDT_NO=pEndtNo OR UPS.REF_NO=pRefNo)`; BR_ADDRESS built with DECODE concatenation + two CMGE_CODE (CITY, STATE) sub-selects; BR_PHONE = PHONE_CODE\|\|PHONE_NO; BR_FAX likewise (J: EndtCover.jrxml.data:11-69). | CompanyNameWithGSTRegNo.jrxml (call at EndtCover.jrxml.data:215): `SELECT COMPANY_DESCP AS COMPANY_NAME, (select PG_RPGE_UTILS.FN_GET_COMP_BRN_NUMBER(companyParamType, to_date(issueDate,'dd/MM/yyyy')) from DUAL) AS COMPANY_REG_NO FROM SAPM_COMPANY WHERE TYPE=companyParamType`. Unconditional once EndtCover runs. |
| 2 | Blank.jrxml | `$P{showCoverNote}.equalsIgnoreCase("Y")` (same as cover) | long list incl. ShowSstLabel, showComplaint, SUBREPORT_DIR2/3, pSchemeName ... | `SELECT 'This Page Is Intentionally Left Blank' AS BLANK FROM DUAL` | none |
| 3 | EndtSchedule.jrxml (runToBottom=true) | `$P{showEndtSchedule} != null && $P{showEndtSchedule}.equalsIgnoreCase("Y")` | pPolicyNo, printSeries, pCertInd, ShowSstLabel (=`$F{SHOW_SST_LABEL}`), ShowGstLabel (=`$F{SHOW_GST_LABEL}`), issueDate (dd-MMM-yyyy string), pEndtNo, pRefNo, pContractId, versionNo, SUBREPORT_DIR2 (**overridden** to `repo:/reports/AGIC/UW/Plines/EndtSchedule/`), companyParamType ... | Big query over `UWPL_VI_ENDT_SCHEDULE_SUMMARY VI` where `VI.ENDT_NO = pEndtNo or VI.REF_NO = pRefNo` (J: EndtSchedule.jrxml.data:125-733). See 2.1. | EndtNarration.jrxml (`$P{SUBREPORT_DIR}+"EndtNarration.jrxml"`, line 2603); Signature.jrxml (`$P{SUBREPORT_DIR2}+"Signature.jrxml"`, line 2661, in the last page footer) |
| 4 | Blank.jrxml | `$F{ENDT_CODE}.equalsIgnoreCase("72") \|\| $F{ENDT_CODE}.equalsIgnoreCase("168")` | same big list | same Blank query | none |
| 5 | EndtScheduleOfBenefits_ASP.jrxml (runToBottom=true) | `$F{ENDT_CODE}.equalsIgnoreCase("72") \|\| $F{ENDT_CODE}.equalsIgnoreCase("168")` | ShowGstLabel (=`$F{SHOW_GST_LABEL}`), SUBREPORT_DIR, SUBREPORT_DIR3, SUBREPORT_CR ... | `SELECT 1 FROM DUAL` - the whole page is static text ("SCHEDULE OF BENEFITS", benefit rows coded ASP-A1 ... ASP-D5, "Optional Benefits", "Note :") | ASPLLB.jrxml (line 1823, `$P{SUBREPORT_DIR2}+"ASPLLB.jrxml"`) and ASPSTI.jrxml (line 1833); both are `SELECT 1 FROM DUAL`, static text ("Optional Benefit - Lifestyle and Living Benefits/Plans", "Optional Benefit - Study Interruption (up to)/Plans", "Study Interruption Benefit is only applicable for the Insured Person who is a registered full time student ...") |

Small facts: Signature prints "SEAN" vs non-"SEAN" variants: `!"SEAN".equalsIgnoreCase($F{REMARKS})` and `"SEAN".equalsIgnoreCase($F{REMARKS})` (J: Signature.jrxml.data:64,70); its query is `SELECT COMPANY_DESCP, FN_GET_COMP_BRN_NUMBER(...) AS COMPANY_REG_NO, CUSTOMER.FN_RPPM_GET_SIGNATURE(issueDate) AS SIGNATURE, UPPER(CUSTOMER.FN_RPPM_GET_SIGNATURE_NAME(issueDate)) AS REMARKS FROM SAPM_COMPANY WHERE TYPE=companyParamType`. EndtNarration query: `case when source='WIP' then (select ENDT_NARR_PREVIEW from WIP_UWGE_POLICY_VERSIONS where ref_no=pRefNo) else (select ENDT_NARR from UWGE_POLICY_VERSIONS where endt_no=pEndtNo) end as ENDT_NARR from UWPL_VI_ENDT_SCHEDULE_SUMMARY where ENDT_NO=pEndtNo or REF_NO=pRefNo`.

### 1.3 EndtDocument2 (`ASP/EndtDocument2_files/EndtDocument2_jrxml.data`)

Parameters: same family as EndtDocument but the cover flag is called **`showCover`** (not `showCoverNote`), no pSchemeName, no ShowGstLabel/ShowSstLabel, no SUBREPORT_DIR3. pContractId Integer, versionNo Integer, issueDate String.

Main query (verified, lines ~46-140): `SELECT tbl.*, (select CUSTOMER.PG_RPGE_UTILS.FN_GET_EINVOICE_PRINT_ENDT_IND(pContractId,pEndtNo,pRefNo,versionNo,checklistCopy) from dual) AS SHOW_EINV_IND FROM (SELECT PRODUCT_CODE, (CASE WHEN DIFF_PREM_DUE <> 0 THEN 1 ELSE 0 END) prem, (CASE WHEN DIFF_GST_AMT <> 0 THEN 1 ELSE 0 END) tax_inv, SHOW_GST_INV_IND, SHOW_TRAN_INV_IND, SHOW_STAX_INV_IND (each from UWGE_VI_ENDT_REPORT_PARAM with `AND ERP.CHECK_LIST_COPY = checklistCopy`), SHOW_GST_LABEL, SHOW_SST_LABEL, ISSUE_DATE, ENDT_CODE, EINV_CONSENT_IND (NVL(...,'N') from UWGE_POLICY_BASES+UWGE_POLICY_VERSIONS UNION WIP_UWGE_POLICY_BASES+WIP_UWGE_POLICY_VERSIONS) FROM (SELECT PB.DIFF_PREM_DUE, PB.DIFF_GST_AMT FROM WIP_UWGE_POLICY_BASES PB, WIP_UWGE_POLICY_VERSIONS PV WHERE PB.CONTRACT_ID=PV.CONTRACT_ID AND PV.REF_NO=pRefNo AND (PB.DIFF_PREM_DUE!=0 OR PB.DIFF_GST_AMT!=0) UNION ALL SELECT ... FROM UWGE_POLICY_BASES PB, UWGE_POLICY_VERSIONS PV WHERE ... PV.ENDT_NO=pEndtNo AND PB.VERSION_NO=PV.VERSION_NO AND (...!=0 OR ...!=0))) tbl`.
Important consequence (inference from reading the SQL): the driving table is the inner UNION. If neither DIFF_PREM_DUE nor DIFF_GST_AMT is non-zero, the main query returns **zero rows** and EndtDocument2 prints nothing (even the ComplaintRedress band). If it returns more than one row, the detail bands repeat per row.
Fields: PRODUCT_CODE, PREM (BigDecimal), TAX_INV, SHOW_GST_INV_IND, SHOW_TRAN_INV_IND, SHOW_STAX_INV_IND, SHOW_GST_LABEL, SHOW_SST_LABEL, ISSUE_DATE, ENDT_CODE, EINV_CONSENT_IND, SHOW_EINV_IND.

Background band printWhen: `($F{PREM}!=null && $F{PREM}!=0 && $P{showDebitNote}.equalsIgnoreCase("Y")) || ($F{TAX_INV}!=null && $F{TAX_INV}!=0 && $P{showTaxInvoice}.equalsIgnoreCase("Y"))` -> prints `"PAGE: " + ((PAGE_NUMBER + pFirstPageNumber) - 1)`.

| # | Sub report | Exact print condition | Parameters passed | Own SQL | Nested |
|---|---|---|---|---|---|
| 1 | `$P{SUBREPORT_CR}+"ComplaintRedress.jrxml"` (Generic folder; line 244) | `$F{PRODUCT_CODE} != null && $F{PRODUCT_CODE}.equalsIgnoreCase("070412")` | pPolicyNo, printSeries, pCertInd, issueDate, pEndtNo, showCover, pFirstPageNumber, SUBREPORT_DIR, showComplaint, SUBREPORT_CR, SUBREPORT_DIR2, showEndtSchedule, companyParamType, pContractId, versionNo, showTaxInvoice, copyType, pActionCode, checklistCopy, showDebitNote, pRefNo | static text page ("Allianz General Insurance Company (Malaysia) Berhad", contact details) | CompanyNameBRNUpperAndLowercase (Generic) |
| 2 | DebitNote.jrxml (line 275) | `$P{showDebitNote}!=null && $P{showDebitNote}.equalsIgnoreCase("Y") && ($F{PREM}!=null && $F{PREM}!=0)` | ShowSstLabel, copyType, checklistCopy, pEndtNo, showDebitNote, pRefNo, ShowGstLabel (**pContractId is not passed**; DebitNote declares it Integer so it is null inside) | `FROM CUSTOMER.UWPL_VI_ENDT_DEBIT VI LEFT JOIN CMGE_CODE CITY ... LEFT JOIN CMGE_CODE STATE ... WHERE ENDT_NO=pEndtNo OR REF_NO=pRefNo` (J: DebitNote.jrxml.data:36-125); see 2.3 | CustomerDtlsOfMasterPolicy_Debit (line 1932), printWhen `$F{PREM_BY_MST_IND}.equalsIgnoreCase("Y")` (line 1923), param = `$F{MST_CONTRACT_ID}` (row-correlated) |
| 3 | Blank.jrxml (line 290) | `($P{showTaxInvoice}=="Y") && ($P{showEndtSchedule}=="Y")` (null-safe, equalsIgnoreCase) | SUBREPORT_DIR, copyType, checklistCopy | Blank query | none |
| 4 | EndtTaxDoc.jrxml (line 324) | `$P{showTaxInvoice}.equalsIgnoreCase("Y") && "Y".equalsIgnoreCase($F{SHOW_GST_INV_IND})` | companyParamType, showTaxInvoice, copyType, issueDate, checklistCopy, pEndtNo, pRefNo, pContractId | UWPL_VI_ENDT_TAX_INV-style GST tax invoice query | CompanyContactJr, AntiRasuahBanner (per hierarchy csv) |
| 5 | Blank.jrxml (line 385) | `showTaxInvoice=Y && SHOW_GST_INV_IND=Y` | long list | Blank | none |
| 6 | ServiceTaxInvoice.jrxml (line 455) | band: `$P{showTaxInvoice}.equalsIgnoreCase("Y") && "Y".equalsIgnoreCase($F{SHOW_STAX_INV_IND}) && !$P{checklistCopy}.equalsIgnoreCase("Y") && $F{SHOW_EINV_IND}.equalsIgnoreCase("Y")`; the inner element condition is weaker (no checklist/e-invoice test) | pPolicyNo, printSeries, pCertInd, issueDate, pEndtNo, showCover, pFirstPageNumber, SUBREPORT_DIR, showComplaint, SUBREPORT_DIR2, showEndtSchedule, companyParamType, pContractId, versionNo, copyType, showTaxInvoice, pActionCode, checklistCopy, pRefNo, showDebitNote (**pType is NOT passed**) | `FROM CUSTOMER.UWPL_VI_ENDT_TAX_INV VI` plus STAX_COUNT, QR_IND, EINV_CUTOFF, VERSION_NO, MST_CONTRACT_ID, PREM_BY_MST_IND scalar sub-selects (J: ServiceTaxInvoice.jrxml.data:38-166) | STCompanyContact (line 609, no extra gate), CustomerDtlsOfMasterPolicy (line 1075), AntiRasuahBanner (line 1710, cond `STAX_COUNT>1 && !ENDT_CODE.equalsIgnoreCase("999")` at 1716), ServiceTaxInvoiceEndtMT (line 1758), ServiceTaxInvoiceEndtMTSub (line 1806; cond `STAX_COUNT>1` at 1764), QRSection (line 1827; cond at 1812 `EINV_CONSENT_IND=Y && EINV_STATUS=VALID && QR_IND=Y`) |
| 7 | Blank.jrxml (line 516) | same as ServiceTaxInvoice band | list | Blank | none |
| 8 | TransitionalInvoice.jrxml (line 586) | `$P{showTaxInvoice}.equalsIgnoreCase("Y") && "Y".equalsIgnoreCase($F{SHOW_TRAN_INV_IND})` | list | tax-invoice style query | CompanyContactJr, SSTNote |
| 9 | Blank.jrxml (line 647) | same as 8 | list | Blank | none |
| 10 | TransitionalInvoice2.jrxml (line 717) | same as 8 | list | tax-invoice style query | STCompanyContact, SSTNote |

Note: in ServiceTaxInvoice, the AntiRasuahBanner subreport (Jasper line 1716) is gated by `STAX_COUNT>1 && ENDT_CODE != "999"` (the banner is printed with the multi-tax-rate block), while Java fires the banner whenever `serviceTaxDto != null` (AspEndtDocumentService.java:365). The banner is static so no data differs; the only difference is the Pline signal (none is emitted).

### 1.4 AddMemberLandscape (page 862x595, `ASP/AddMemberLandscape_files/AddMemberLandscape_jrxml.data`)

Parameters: pContractId Integer, pActionCode, pEndtNo, checklistCopy, pFirstPageNumber Integer, pRefNo, SUBREPORT_DIR, versionNo **String**, issueDate java.util.Date.
Main query (`from dual`, lines 25-52): 
- ADDITION_PRINT = `select 1 from wip_uwge_policy_versions where ref_no=pRefNo and endt_code in ('72','168') AND CONTRACT_ID IN (select UR.CONTRACT_ID from WIP_UWGE_RISK UR where ... UR.ACTION_CODE='A' AND UR.VERSION_NO IS NULL) UNION ALL select 1 from uwge_policy_versions where endt_no=pEndtNo and endt_code in ('72','168') AND CONTRACT_ID IN (SELECT UR.CONTRACT_ID from UWGE_RISK UR where ... ACTION_CODE='A' AND VERSION_NO=uwge_policy_versions.VERSION_NO)`.
- DELETION_PRINT = WIP_UWGE_POLICY_VERSIONS (ref_no, endt_code in ('98')) UNION ALL UWGE_POLICY_VERSIONS (endt_no, endt_code in ('98')) - fetched but **not used by any band** in this report.
- NOMINEE_PRINT = `count(1) FROM UWPL_VI_ENDT_SCHEDULE_NOM WHERE (ENDT_NO=pEndtNo OR REF_NO=pRefNo) AND CONTRACT_ID=pContractId and NOMINEE_NAME is not null`.
- SPONSOR_PRINT = same with `UWPL_VI_ENDT_SCHEDULE_SPONSOR` and `SPONSOR_NAME is not null`.
- ISSUE_DATE = from UWGE_VI_ENDT_REPORT_PARAM (no ROWNUM).
Background (frame, watermark, logo, `PAGE: PAGE_NUMBER + pFirstPageNumber - 2`) prints when `$F{ADDITION_PRINT}>0 || $F{NOMINEE_PRINT}>0 || $F{SPONSOR_PRINT}>0`.

| # | Sub report | Exact condition | Params | SQL |
|---|---|---|---|---|
| 1 | AddMemberLandscape_ASPInsured.jrxml | `$F{ADDITION_PRINT}>0` | SUBREPORT_DIR, pContractId, versionNo, issueDate (=`$F{ISSUE_DATE}`), pActionCode, pEndtNo, checklistCopy, pRefNo, pFirstPageNumber | `select ... FROM customer.UWPL_VI_ENDT_SCHEDULE_TPL4 WHERE (ENDT_NO=pEndtNo OR REF_NO=pRefNo) AND CONTRACT_ID=pContractId ORDER BY cast(COV_SEQ_REF as decimal(15,2)) asc` (J: AddMemberLandscape_ASPInsured.jrxml.data:20-72). INSURED_TYPE via CMGE_CODE CAT_CODE='INSURED_TYPE'. `replace(RB_AMT,'.00','')`. SHOW_OCCP_DESC = `TO_DATE(issueDate,'dd-mm-yyyy') <= TO_DATE(SAPM_SYS_CONSTANTS.DATE_VALUE where SYS_TYPE='OCCUP_CUT_OFF_DATE' and SYS_CODE='OCCUP_DATE','dd-mm-yyyy')` -> 'Y' else 'N'. Title text "ADDITION OF MEMBER(S)"; header "Attaching and forming part of Endorsement No. :". Note: pContractId is declared **String** here. |
| 2 | AddMemberLandscape_ASPPremium.jrxml | `$F{ADDITION_PRINT}>0` | same | same view `UWPL_VI_ENDT_SCHEDULE_TPL4`, same WHERE/ORDER BY (lines 57-71), plus the same SHOW_OCCP_DESC |
| 3 | EndtParticularOfSponsorLandscape_ASP.jrxml | `$F{SPONSOR_PRINT}>0` | SUBREPORT_DIR, versionNo, pContractId, pActionCode, issueDate, checklistCopy, pEndtNo, pFirstPageNumber, pRefNo | `select SP.CONTRACT_ID, sp.insured_name as risk_name, sp.sponsor_name, sp.SPONSOR_IDVALUE, sp.SPONSOR_DOB, sp.SPONSOR_AGE, sp.ref_no, sp.endt_no from UWPL_VI_ENDT_SCHEDULE_SPONSOR sp where (sp.ref_no=pRefNo OR sp.endt_no=pEndtNo) and sp.contract_id=pContractId and sponsor_name is not null` (no ORDER BY). Title "PARTICULAR OF SPONSOR(S)". |
| 4 | EndtParticularOfNominationLandscape_ASP.jrxml | `$F{NOMINEE_PRINT}>0` | same as 3 | `select CONTRACT_ID, POLICY_REF AS POLICY_NO, ENDT_NO, REF_NO, INSURED_NAME, NOMINEE_NAME, RELATIONSHIP (CMGE_CODE 'RELATIONSHIP'), SHARE_PCT, ID_TYPE, ID_VALUE1, NOMINEE_ID_VALUE1, NOMINEE_RELATIONSHIP FROM UWPL_VI_ENDT_SCHEDULE_NOM WHERE (...) AND CONTRACT_ID=pContractId AND NOMINEE_NAME IS NOT NULL ORDER BY cast(COV_SEQ_REF as decimal(15,2)) asc` (lines 23-40). Title "PARTICULAR OF NOMINEE(S)". |

### 1.5 DeletionMemberLandscape (`ASP/DeletionMemberLandscape_files/DeletionMemberLandscape_jrxml.data`)

Parameters: pContractId Integer, pActionCode, pEndtNo, checklistCopy, pFirstPageNumber, pRefNo, SUBREPORT_DIR, issueDate Date.
Main query: `select (select count(CONTRACT_ID) from UWPL_VI_ENDT_ADD_DEL_RISK where (endt_no=pEndtNo or ref_no=pRefNo) and ACTION_CODE='C' and TERMINATE_DATE is not null) as deletion_print from dual`.
Background and band condition: `$F{DELETION_PRINT} > 0`. One subreport:

| # | Sub report | Condition | SQL |
|---|---|---|---|
| 1 | DeleteMemberLandscape_ASP.jrxml | `$F{DELETION_PRINT} > 0` | `select ... from UWPL_VI_ENDT_ADD_DEL_RISK where (ENDT_NO=pEndtNo OR REF_NO=pRefNo) AND ENDT_CODE IN ('98','168') and ACTION_CODE='C' and TERMINATE_DATE is not null` (J: DeleteMemberLandscape_ASP.jrxml.data:21-72). No ORDER BY, no CONTRACT_ID filter. RELATIONSHIP_DESC = `SUBSTR(CODE_DESC,'4')` from CMGE_CODE 'RELATIONSHIP' (drops first 3 characters). Columns include NVL(STI_PREM,0), NVL(LLB_PREM,0), DIFF_AGIC_FEE, DIFF_FEE_SST, NVL(LOADING,0), NVL(ABS(MCO_FEE),0), COV_STAX_AMT, COV_DISC_AMT/PCT, COV_REBATE_AMT/PCT, DIFF_COV_REBATE_AMT, RISK_LOADING, INSURED_TYPE (CMGE_CODE), ENDT_DOUBLE_INDEMNITY, ENDT_IMA_FEE, INPATIENT_PREM, ENDT_STAX_AMT, ENDT_LLB_PREM, ENDT_STI_PREM. Column headers include "Lifestyle and Living Benefit Plan", "Study Interruption (RM)". |

Quirk inside Jasper itself: the dispatch counts deleted risks of ANY endorsement code, but the subreport only shows ENDT_CODE 98 or 168. So a dispatch count > 0 for another code (e.g. 72) gives a landscape background page with an empty table (inference from reading the two SQLs).

Also present in the folder but NOT in any hierarchy: `ComplaintRedress_NoBlankPage.jrxml`, `TransitionalInvoice` variants are used; `Test.xml` / `Test_files/Main jrxml.data` is a scratch copy of the delete-member report (same column labels). `ComplaintRedress_NoBlankPage` is unused by EndtDocument2, which calls the Generic `ComplaintRedress.jrxml` (EndtDocument2 line 244).

---------------------------------------------------------------------------------------------------

## 2. Main queries and parameters, summarised

### 2.1 EndtSchedule main query (the heaviest ASP query, J: EndtSchedule.jrxml.data:125-733)
- Source: `UWPL_VI_ENDT_SCHEDULE_SUMMARY VI` (`where VI.ENDT_NO = pEndtNo or VI.REF_NO = pRefNo`).
- Derived columns: NAME_EXT = NVL(LONG_NAME, NAME_EXT); ACC_COMM_PCT / ACC_COMM2_PCT via CMGE_CODE 'COMM_RATE'; GROSS_PREMIUM = ABS(CASE WHEN PRODUCT_CODE IN ('089332','089432','089329','089429') THEN ABS(GROSS_PREM)+ASST_FEE_DIFF+CRF_FEE_DIFF+OCR_FEE_DIFF ELSE GROSS_PREM END); fee diffs DMA, FW (MCOI/SPPA), MCOI, MCOO, IMA, OCR, CRF, ASST, MCO as SUM(DIFF_FEE_AMT) from WIP_UWGE_COVER_FEES (source WIP) or SB_UWGE_POLICY_FEES (committed); FW_POL_SI from WIP_UWPL_COVER / SB_UWPL_COVER; ENDT_NARR from WIP_UWGE_POLICY_VERSIONS.ENDT_NARR_PREVIEW or UWGE_POLICY_VERSIONS.ENDT_NARR; **SHOW_GST_LABEL** = CASE (source='OCP' and ISSUE_DATE < pg_uwge_utils.Fn_Get_SST_Rollout_Date) or (source='WIP' and SYSDATE < Fn_Get_SST_Rollout_Date) then 'Y' else 'N' (lines 624-626); GPA_ANNUAL_PREM, GPA_MATCH_AMT (ACGC_KO_INST + ACRC_KO), GPA_GROSS_PREM, GPA_STAX_AMT, GPA_STAMP, REFUND_AMT, TOT_REFUND_AMT; **INST_IND** (line 727) = `CASE WHEN VI.ISSUE_DATE >= (SELECT DATE_VALUE FROM SAPM_SYS_CONSTANTS WHERE SYS_TYPE='CUT_OFF_RPT' AND SYS_CODE = VI.PRODUCT_CODE||'_ENDT') AND VI.PMT_FREQUENCY <> 'AN' THEN 'Y' ELSE 'N'`; DIFF_AGIC_FEE, DIFF_FEE_SST; TPA_NAME from UWPL_POLICY_BASES (comment `AGIC-41641`).
- Layout conditions in the jrxml (product-specific print rules): hides some lines for products 100908 / 070406; installment/GST lines for 100906, 070521, 070504 (scheme names INCOMESHIELD, ELITESHIELD, SNATCHPA-A, SNATCHPA-M with INST_IND='N'); IMA fee only for 070412, 070202, 100905, 100906; ASST/CRF/OCR fee lines excluded for 089332/089432/089329/089429; MEDIX TPA gets a separate variant of the page (`TPA_NAME != null && "MEDIX".equalsIgnoreCase(TPA_NAME)`); SST lines driven by ShowSstLabel + ST_REVISE_FEE_IND + STAX_AMT / GPA_STAX_AMT; GST label lines by ShowGstLabel.

### 2.2 Other main queries (short)
- EndtCover: see table row 1 above (summary view + agent/branch chain + DMAG_RI).
- DebitNote: `UWPL_VI_ENDT_DEBIT` + CMGE_CODE joins; DEBIT_NOTE = `case when PREM_DUE>=0 then 'D' else 'C' end || ENDT_NO`; DEBITCREDIT_FLAG = `case when PREM_DUE>=0 then 'DEBIT' else 'CREDIT' end`; GROSS_PREMIUM special-product formula (089332, 089432, 089329, 089429); ROUNDED_NETT_PREM = `ABS(round(NETT_PREM*20)/20)`; BANCA_AGENT via CMUW_BANKBRANCH; POLICYNO = PME_API.GET_POLICY_REF(CONTRACT_ID); SHOW_GST_LABEL via SST rollout date; MST_CONTRACT_ID = `(SELECT contract_id FROM OCP_POLICY_BASES WHERE POLICY_REF IN (SELECT MST_POL_NO FROM UWGE_POLICY_BASES WHERE contract_id=pContractId AND VERSION_NO=1 UNION SELECT MST_POL_NO FROM WIP_UWGE_POLICY_BASES WHERE contract_id=pContractId AND VERSION_NO=1 AND MST_POL_NO IS NOT NULL) AND VERSION_NO=1)`; PREM_BY_MST_IND = NVL((select PREM_BY_MST_IND from UWGE_POLICY_BASES where CONTRACT_ID = that master contract and VERSION_NO=1),'N'). The title text switches `PREMIUM CREDIT NOTE` / `PREMIUM DEBIT NOTE` (J: DebitNote.jrxml.data:572) and `Credit Note` / `Debit Note` (line 741). Odd Groovy: `$F{PRODUCT}!="070406" || $F{PRODUCT}!="070411"` (lines 1118, 1153, 1686, 1731) is always true (reference comparison on strings), so only the second half of those conditions has effect.
- ServiceTaxInvoice: `UWPL_VI_ENDT_TAX_INV`; `case when VI.GROSS_PREM>=0 then 'DEBIT' else 'CREDIT'` (line 78: note it uses GROSS_PREM, DebitNote uses PREM_DUE); EINV_CUTOFF = SAPM_SYS_CONSTANTS where SYS_TYPE='UW_VALIDATION' and SYS_CODE='EINV_CUT_OFF' (line 147); VERSION_NO = UWGE_POLICY_VERSIONS UNION WIP_UWGE_POLICY_VERSIONS (line 148-150); QR_IND = FN_GET_EINVOICE_QR_IND(...) (line 40); STAX_COUNT sub-select (line 136). Print rules: `ISSUE_DATE >= EINV_CUTOFF && !PREM_BY_MST_IND.equalsIgnoreCase("Y")` for e-invoice header blocks; the master-policy block (Tax Identification No., etc.) when PREM_BY_MST_IND=Y; "STRN" invoice type switch via `STAX_INV_NO.substring(0,4).equals("STRN")` or `STAX_INV_TYPE.equals("STRN")` (lines 808, 903); agent commission lines only if `COMM_AMT>0 && (pType = "AO" or "AD")` (lines 1301-1500); "not valid" text if `!EINV_STATUS.equalsIgnoreCase("VALID")` (line 1662).
- QRSection: `SELECT * FROM (UWGE_GST_TAX_INV VERSION_NO=1 ... UNION ... VERSION_NO=versionNo ... UNION WIP_UWGE_GST_TAX_INV ...) WHERE ROWNUM = 1` returning ISSUER_DIGITAL_SIGN, TO_CHAR(EINV_TIME,'DD/MM/YYYY HH24:MI:SS'), EINV_QR (J: Generic/QRSection.jrxml.data:21-33).
- PrintSeriesFooter: see 1.2.
- ServiceTaxInvoiceEndtMT: per-tax-rate rows from WIP/UWGE_POLICY_..._STAX (UNION), prints when `STAX_COUNT>1` (J: Generic/ServiceTaxInvoiceEndtMT.jrxml.data:115).

### 2.3 Parameter class mismatches that matter in Java
- Main reports: pContractId is **Integer**; landscape sub reports `AddMemberLandscape_ASPInsured / _ASPPremium / EndtParticularOfSponsorLandscape_ASP / EndtParticularOfNominationLandscape_ASP` declare it **String** while the view column CONTRACT_ID is BigDecimal (Oracle converts implicitly). In Mongo, CONTRACT_ID is stored numerically so a String filter never matches. All ASP Java handlers therefore convert with a local `safeNum(...)` before querying (e.g. AspAddMemberLandscapeDispatchHandler.java:123-126).
- AddMemberLandscape `versionNo` is String, `issueDate` is java.util.Date; EndtDocument `issueDate` is a String (dd-MMM-yyyy).

---------------------------------------------------------------------------------------------------

## 3. Traceability table and the merge

### 3.1 How the four reports were merged into one bean

- Evidence: class Javadoc `AspEndtDocumentService.java:81-163`; service annotation `:164-167`; disabled `//@Service` at `AspEndtDocument2Service.java:44`; disabled `//@Bean("AGIC/UW/Plines/ASPEndtSchedule/AddMemberLandscape")` at `AspEndtDocumentLandscapeService.java:620` and `.../DeletionMemberLandscape` at `:625`.
- `AspEndtDocumentService.execute` (line 212) reads filters (pEndtNo, pRefNo, pContractId, companyParamType, versionNo, checklistCopy, printSeries, copyType, pUserID, showCoverNote, showEndtSchedule, showComplaint, showDebitNote, showTaxInvoice, pPolicyNo, pFirstPageNumber), then calls BOTH main handlers: `mainHandler` (V1 = EndtDocument) at line 236 and `mainHandler2` (V2 = EndtDocument2) at line 238. It then evaluates V1 gates, V2 gates, then runs both landscape dispatchers with no gate (lines 1161 and 1323).
- Flag name difference: EndtDocument2 calls the cover flag `showCover`; the merged bean uses `showCoverNote` only (line 225).
- The landscape service used to be dispatched by a `reportName` filter value (`AspEndtDocumentLandscapeService.java:102-105`). The merged bean ignores `reportName`; the Javadoc (lines 153-154) still says "Dispatched via the incoming reportName filter value" but the code of `AspEndtDocumentService.execute` never reads it (verified by reading the full method). That sentence is stale documentation.
- Callers: since `EndtDocument2`, `AddMemberLandscape`, `DeletionMemberLandscape` bean keys no longer exist, only the key `.../ASPEndtSchedule/EndtDocument` resolves (inference: bean lookup by `@Service` name; I did not read the dispatcher).

### 3.2 Effect on reportData ordering and subReportName

Final order built at `AspEndtDocumentService.java:1386-1498`:
1. `reportData[0]` master row: `mainReportName="ASPEndtScheduleEndtDocument"`, `subReportName="PrintSeriesFooter"` (lines 1389-1390). Carries productcode, showSstLabel, issuedate, actionCode (=ENDT_CODE), prnCnt (print series count), blank text, einvConsentInd, endorsementDetails, shared prntDetails.
2. `EndtCover` (main name `ASPEndtScheduleEndtDocument`) - if cover row exists (line 1393).
3. `ComplaintRedress` (main name `ASPEndtScheduleEndtDocument2`) - if product 070412 (line 1414). **Note this is placed before EndtSchedule**, which is not the order Jasper would print the two reports in (EndtDocument fully, then EndtDocument2). Because the Java output has no page layout, the order is a convention, not a print order (inference).
4. `EndtSchedule` (`...EndtDocument`) - line 1423.
5. one `DebitNote` per debit/credit row (`...EndtDocument2`) - line 1444; then `CustomerDtlsOfMasterPolicy_Debit` (line 1445).
6. one `EndtTaxDoc` per row (line 1459).
7. `ServiceTaxInvoice` (+ one per EndtMT / EndtMTSub row), then `QRSection` (lines 1460-1495).
8. `TransitionalInvoice` and `TransitionalInvoice2` rows (line 1496).
9. landscape rows (line 1497): `AddMemberLandscapeDispatch`, then `AddMemberLandscape_ASPInsured` rows, `AddMemberLandscape_ASPPremium` rows, `EndtParticularOfSponsorLandscape_ASP` rows, `EndtParticularOfNominationLandscape_ASP` rows (main name `ASPEndtScheduleAddMemberLandscape`), then `DeleteMemberLandscapeDispatch` and `DeleteMemberLandscape_ASP` rows (main name `ASPEndtScheduleDeletionMemberLandscape`).
- `mainReportName` is "ASPEndtSchedule" + report name via `PlineReportNames.mainNameOf` (utils/PlineReportNames.java:56-63; only blanks are filled by `tag`, line 33-36 of that class). The service sets four main names itself as constants (lines 168-171).
- `subReportName` values are plain strings set in `sectionReportData` (line 1550). The names used in Java differ slightly from the jrxml: dispatch rows use `AddMemberLandscapeDispatch` and `DeleteMemberLandscapeDispatch` (no jrxml of those names; they are synthetic), and `DeleteMemberLandscape_ASP` matches the jrxml name.
- Test that locks the order: `AspEndtDocumentServiceTest.allGatesOpen_oneReportDataPerSubreportInSubreportOrder` (service test line 424-454) expects: PrintSeriesFooter, EndtCover, ComplaintRedress, EndtSchedule, DebitNote, ServiceTaxInvoice, AddMemberLandscapeDispatch, DeleteMemberLandscapeDispatch.
- Shared state caveat: `prntDetails`, `endorsementDetails`, `transDetList` are built once and shared; landscape rows reuse the master `branchInfo` and `compInfo` by reference and the same `endorsementDetails` object (`landscapeRow`, lines 1566-1574). Because `mergeEndorsement` is "last non-null wins", every landscape row sees the final merged values (inference from the code; check before claiming per-row accuracy).

### 3.3 Traceability table

Legend: Mongo collection names are those used in the Java source (the Oracle table/view name where the collection is only known by the Java constant is the same string). "Stub" = handler returns `dualValue=1` and nothing is mapped. XSD path root: `transaction/transactionDetails/headerTransaction/reportTransaction/reportData` (Pline.xsd lines 472, 552, 555, 572, 579), so `endorsementDetails` = reportData/endorsementDetails (line 593) and `transDet` = reportData/prntDetails/transDetList/transDet (lines 652, 668, 671). `mainReportName` / `subReportName` exist in the Java class `requestpayload/PlineReportData.java:21-22` but are **not present in the copy of Pline.xsd at `src/test/resources/xsd/plines/Pline.xsd`** (searched case-insensitively). PlineReportNames' Javadoc points to "Pline.xsd v0.1.10" for them (the authoritative XSD is therefore outside this repo copy; inference).

| Jasper (sub)report | Print condition | Java handler.method (class) | Mongo collection(s) | Pline payload field(s) (Java) | Pline.xsd path |
|---|---|---|---|---|---|
| EndtDocument main query | always (1 row) | `AspEndtDocumentMainHandler.fetchMainDocument` (documentHandler/AspEndtDocumentMainHandler.java:36) | anchor UWMT_VI_ENDT_TAX_INV_ALT_UPV, lookups UWGE_VI_ENDT_REPORT_PARAM, WIP_UWGE_POLICY_CONTRACTS, SB_UWGE_POLICY_CONTRACTS, WIP_UWGE_POLICY_VERSIONS, UWGE_POLICY_VERSIONS | master transDet: productcode, showSstLabel, issuedate, actionCode (service lines 440-443); reportParameters pShowGstLabel, pShowSstLabel, pissueDate (1508-1509) | .../transDet/productcode etc.; reportParameters/pShowGstLabel (line 532), pShowSstLabel (536) |
| EndtDocument2 main query | always (see 4.1 on zero-row difference) | `AspEndtDocument2MainHandler.fetchMainDocument` (AspEndtDocument2MainHandler.java:27) + overridden `buildCommonPipeline` (:117) | anchor UWMT_VI_ENDT_TAX_INV_ALT_UPV; UWGE_VI_ENDT_REPORT_PARAM, WIP_UWGE_POLICY_BASES, UWGE_POLICY_BASES, UWGE_POLICY_VERSIONS, WIP_UWGE_POLICY_VERSIONS, WIP/UWGE_POLICY_CONTRACTS; SHOW_EINV_IND via REST client `PlineUtilsClient.fnEinvoicePrintEndtInd` | gates held in local variables; `transDet.eInvConsentInd` (line 511) | transDet/eInvConsentInd |
| PrintSeriesFooter | always (printSeries test is a layout rule) | `AcpPrintSeriesFooterHandler.fetchPrintCount` (shared ACP handler) | UWGE_POLICY_VERSIONS_CTRL (from the jrxml query; handler not opened) | master transDet.prnCnt (line 444) | transDet/prnCnt (inference on leaf name) |
| EndtCover | showCoverNote=Y | `AspEndtCoverHandler.fetchEndtCover` (AspEndtCoverHandler.java:69) | UWPL_VI_ENDT_SCHEDULE_SUMMARY, DMT_AGENTS, DMAG_AGENTS, CMDM_BRANCH, CMGE_CODE | reportData "EndtCover": transDet.mstPolNo, agentCode, agentName, effDate, expDate, idNo, insuredName, cpPartId, cpVersion, cpAddrId, cpAddrVersion, productName, stateDescp; prntDetails.branchInfo.brAddress/faxNo/phoneNo; clientInfo.name/address; agentInfo.agentCatType (service 1393-1413) | .../prntDetails/branchInfo, clientInfo, agentInfo; transDet/... |
| CompanyNameWithGSTRegNo (under EndtCover) | unconditional once EndtCover fires | `PlineCommonCompanyNameWithGSTRegNoHandler.fetchCompanyNameWithGST` | SAPM_COMPANY + FN_GET_COMP_BRN_NUMBER (via client) | prntDetails.compInfo.compName / compBRN (lines 640-643, 1408-1411) | prntDetails/compInfo/compName, compBRN |
| Blank | cover: showCoverNote=Y; 72/168 (EndtDocument); tax+schedule combos (EndtDocument2) | `PlineCommonBlankHandler.fetchBlank` (always called, line 246) | none (constant text) | master transDet.blank (line 445) | transDet/blank |
| EndtSchedule | showEndtSchedule=Y | `AspEndtScheduleHandler.fetchEndtSchedule` (AspEndtScheduleHandler.java:76) | UWGE_POLICY_VERSIONS (anchor), UWGE_POLICY_CONTRACTS, UWGE_POLICY_BASES, OCP_POLICY_BASES, UWPL_VI_ENDT_SCHEDULE_SUMMARY, SB_UWPL_POLICY_BASES, WIP_UWGE_POLICY_VERSIONS/CONTRACTS/BASES, WIP_POLICY_BASES, UWPL_POLICY_BASES, ACGC_KO_INST, ACRC_KO, CMGE_CODE, DMT_AGENTS, DMAG_AGENTS, CMDM_BRANCH | reportData "EndtSchedule": transDet polno, productcode, grossPrem, gstAmt, premDue, totalpremium, rebateAmount, polsi, fee diffs, staxList/staxPcts.asstFeeDiff & gpaStaxAmt, instInd, tpaName, premPayableRounded, cityDescp, premium (service 448-491); branchInfo.branchName; clientInfo.address; taxClientInfo.stampDuty; endorsementDetails endtEffDate, source, endtCnoteNo, accCommPct, accComm2Pct, endtNarr (690-701) | transDet/*; prntDetails/branchInfo/branchName; reportData/endorsementDetails/endtNarr (line 613) |
| EndtNarration (under EndtSchedule) | with EndtSchedule | no separate handler; ENDT_NARR comes from `AspEndtScheduleHandler` (WIP: ENDT_NARR_PREVIEW) | WIP/UWGE_POLICY_VERSIONS | endorsementDetails.endtNarr (line 700) | endorsementDetails/endtNarr |
| Signature (under EndtSchedule) | with EndtSchedule; prints "SEAN"/non-"SEAN" variant | `PlineCommonSignatureHandler.fetchSignature` (service 264-265) | SAPM_COMPANY + FN_RPPM_GET_SIGNATURE(_NAME) (via client) | companyType/companyDesc.companyName, companyRegNo, signature, remarks (703-710; remarks upper-cased, "NULL" skipped) | prntDetails/companyType/companyDesc |
| EndtScheduleOfBenefits_ASP | ENDT_CODE 72 or 168 | `PlineEndtScheduleOfBenefitsAspHandler.fetchEndtScheduleOfBenefitsAsp` (**stub**) | none | **nothing mapped** (service 273-278 calls and discards) | none |
| ASPLLB | with the above | `PlineAspLlbHandler.fetchAspLlb` (**stub**, returns dualValue=1) | none | nothing | none |
| ASPSTI | with the above | `PlineAspStiHandler.fetchAspSti` (**stub**) | none | nothing | none |
| ComplaintRedress | product 070412 | `AcpComplaintRedressHandler.fetchComplaintRedress` (result discarded) | none (static page) | reportData "ComplaintRedress" with empty transDet (line 1416) | (empty transDet) |
| CompanyNameBRNUpperAndLowercase | nested in ComplaintRedress | `PlineEndtScheduleCompanyNameBRNUpperAndLowercaseHandler.fetchCompanyNameBRN` | SAPM_COMPANY | compInfo.compName / compBRN on the Complaint row (1417-1420) | prntDetails/compInfo |
| DebitNote | showDebitNote=Y and PREM != 0 | `AspDebitNoteHandler.fetchDebitNotes` (AspDebitNoteHandler.java:187) | UWGE_VI_ENDT_DEBIT, CMGE_CODE, DMAG_VI_AGENT, CMUW_BANKBRANCH, OCP_POLICY_BASES, UWPL_POLICY_BASES (+ MST chain in `fetchMstContractInfo`:131) | one reportData per row: transDet no1, polno, insuredName, agentName, debitNoteNo, debitCreditFlag, gstPct, gstAmt, totalpremium, premDueRounded, rebateAmount, bankCommission, commPct, nettPrem, roundedNettPrem, staxPct/Amt, fee diffs, premByMstInd, mstContractId; clientInfo.longName; taxClientInfo.stampDuty/clazz; agentInfo.bancaAgent (service 531-611) | transDet/debitNoteNo (line 839), debitCreditFlag (838), premByMstInd (1133), mstContractId (1050) |
| CustomerDtlsOfMasterPolicy_Debit | PREM_BY_MST_IND=Y (+ master contract id present) | `PlineCommonCustomerDtlsOfMasterPolicyHandler.fetchCustomerDtls` (uses first debit row's mstContractId, service 302-307) | via common handler | clientInfo.name, mstPolName, address.* (1445-1457) | prntDetails/clientInfo |
| EndtTaxDoc (+ AntiRasuahBanner, CompanyContactJr) | showTaxInvoice=Y and SHOW_GST_INV_IND=Y | `PlineEndtTaxDocHandler.fetchEndtTaxDoc` (5-arg overload), `PlineGenericAntiRasuahBannerHandler.fetchAntiRasuahBanner`, `PlineCommonCompanyContactHandler.fetchCompanyContact` | UWPL tax-invoice view (via shared handler) | one reportData per row "EndtTaxDoc" (968-978) | transDet/* ; prntDetails/compInfo |
| ServiceTaxInvoice | showTaxInvoice=Y and SHOW_STAX_INV_IND=Y and checklistCopy != Y and SHOW_EINV_IND=Y | `AspServiceTaxInvoiceHandler.fetchServiceTaxInvoice` (AspServiceTaxInvoiceHandler.java:75) | UWPL_VI_ENDT_TAX_INV, CMGE_CODE, CMUW_ENDT, SAPM_SYS_CONSTANTS, UWGE_POLICY_VERSIONS, WIP_UWGE_POLICY_VERSIONS, UWGE_POLICY_BASES_STAX, WIP_UWGE_POLICY_BASES_STAX, UWPL_VI_TAX_INVOICE; REST clients for policy reference, SST rollout date, QR indicator | one "ServiceTaxInvoice" reportData (stDet, 1460-1492) + endorsementDetails.staxInvNo, staxInvDate, oriStaxInvNo, staxInvType, invNo, sstDate, maskedTin, partnerType, sstRegisterNo ... (714-736) | transDet/staxInvDate; endorsementDetails/staxInvNo (641), staxInvType (642), sstDate (639), maskedTin (624) |
| STCompanyContact | once ServiceTaxInvoice row exists, or TransitionalInvoice2 rows exist | `PlineCommonSTCompanyContactHandler.fetchSTCompanyContact` | via common handler | compInfo.companyGstRegNo, companySstRegNo, fax, phoneNo, address.*; companyDesc.companyName/companyRegNo (843-858, 1475-1489) | prntDetails/compInfo |
| CustomerDtlsOfMasterPolicy (under ServiceTaxInvoice) | PREM_BY_MST_IND=Y | same handler; reuses `AspDebitNoteHandler.fetchMstContractInfo` (service 352, 392-395) | UWGE_POLICY_BASES, WIP_UWGE_POLICY_BASES, OCP_POLICY_BASES | clientInfo.* (same variable as debit note) | prntDetails/clientInfo |
| ServiceTaxInvoiceEndtMT | STAX_COUNT > 1 and ENDT_CODE != "999" | `AcpServiceTaxInvoiceEndtMTHandler.fetchServiceTaxInvoiceEndtMT` (service 369-372) | via shared handler | one reportData "ServiceTaxInvoiceEndtMT" per row: transDet.staxAmt2, staxPct2, effDate2, expDate2, diffAgicFee2; staxPcts.staxPrem2 (887-897) | transDet/staxAmt2 etc. |
| ServiceTaxInvoiceEndtMTSub | STAX_COUNT > 1 | `AcpServiceTaxInvoiceEndtMTSubHandler.fetchServiceTaxInvoiceEndtMTSub` | via shared handler | "ServiceTaxInvoiceEndtMTSub" rows: staxAmt2, staxPct2, effDate2, expDate2, diffFeeSst2 (899-909) | transDet/diffFeeSst2 |
| QRSection | EINV_CONSENT_IND=Y and EINV_STATUS=VALID and QR_IND=Y | `PlineGenericQRSectionHandler.fetchQRSection` (service 379-383) | UWGE_GST_TAX_INV / WIP_UWGE_GST_TAX_INV (per jrxml) | "QRSection" reportData: transDet.digitalSign, einvTime, einvQr (879-883, 1493-1495) | transDet/digitalSign (line 871), einvQr (890) |
| AntiRasuahBanner | static | `PlineGenericAntiRasuahBannerHandler.fetchAntiRasuahBanner` (called, no result mapped) | none | none | none |
| TransitionalInvoice (+CompanyContactJr, SSTNote) | showTaxInvoice=Y and SHOW_TRAN_INV_IND=Y | `PlineTransitionalInvoiceHandler.fetchTransitionalInvoices` (5-arg) | via shared handler | "TransitionalInvoice" rows (982-1055); SSTNote is not mapped | transDet/* |
| TransitionalInvoice2 (+STCompanyContact, SSTNote) | same gate | `PlineTransitionalInvoice2Handler.fetchTransitionalInvoice2` | via shared handler | "TransitionalInvoice2" rows (1057-1145) | transDet/* |
| AddMemberLandscape (dispatch) | always evaluated (Jasper: it is a top-level report) | `AspAddMemberLandscapeDispatchHandler.fetchDispatch` (:34) | UWGE_POLICY_VERSIONS, WIP_UWGE_POLICY_VERSIONS, UWGE_RISK, WIP_UWGE_RISK, UWPL_VI_ENDT_SCHEDULE_NOM, UWPL_VI_ENDT_SCHEDULE_SPONSOR, UWGE_VI_ENDT_REPORT_PARAM | reportData "AddMemberLandscapeDispatch": transDet.additionPrint, deletionPrint, sponsorPrint, issuedate; nomineeList/nominee.nomineePrint (1168-1174) | transDet/additionPrint (line 685), deletionPrint (842), sponsorPrint (1226); nomineeList/nominee/nomineePrint (337) |
| AddMemberLandscape_ASPInsured | ADDITION_PRINT>0 | `AspAddMemberLandscapeInsuredHandler.fetch` (:46) | UWPL_VI_ENDT_SCHEDULE_TPL4, CMGE_CODE, SAPM_SYS_CONSTANTS | reportData "AddMemberLandscape_ASPInsured": transDet policyNo, productcode, subsidiaryName, department, idType, idValue1, age, dateOfBirth, insuredName, insuredType, occupDesc, plancode, effDate, relationship, ad, pd, rbAmt, personalLiability, medicalExpenses, hospitalIncome, bereavement, bloodTransfusion, infertility, covNettPrem, optImaFee, doubleIndemnity, mcoFee, loading, designation, occupClass, stiPlanCode, llbPlanCode, llbSi, stiSi, agicFee, feeSst, showOccpDesc; sobList/benefits/benefit.weeklyBenefit; coverList/cover.covGrossPrem, covAnnualPrem (1176-1230); reportParameters parentName, planDescp, psi (1511-1513) | transDet/*; transDet/sobList/benefits/benefit; coverList/cover; reportParameters/parentName (484), planDescp (497), psi (538) |
| AddMemberLandscape_ASPPremium | ADDITION_PRINT>0 | `AspAddMemberLandscapePremiumHandler.fetch` | UWPL_VI_ENDT_SCHEDULE_TPL4 ... | "AddMemberLandscape_ASPPremium" rows: llbPrem, stiPrem, riskList/risk.riskLoading, coverList/cover.covGrossPrem, covAnnualPrem, covDiscAmt/Pct, covRebateAmt/Pct, covStaxAmt, covDiscChdPct/Amt, covBasicPrem (1232-1283) | transDet/riskList/risk; coverList/cover |
| EndtParticularOfSponsorLandscape_ASP | SPONSOR_PRINT>0 | `AspParticularOfSponsorLandscapeHandler.fetch` | UWPL_VI_ENDT_SCHEDULE_SPONSOR | rows: transDet.contractId, riskName, sponsorName, sponsorIdvalue, sponsorDob, sponsorAge (1286-1300) | transDet/sponsorName (1225), sponsorIdvalue (1224) |
| EndtParticularOfNominationLandscape_ASP | NOMINEE_PRINT>0 | `AspParticularOfNominationLandscapeHandler.fetch` | UWPL_VI_ENDT_SCHEDULE_NOM, CMGE_CODE | rows: policyNo, insuredName, nomineeName, relationship, sharePct, idType, idValue1; nominee.nomineeIdValue1, nomineeRelationship (1302-1321) | transDet/nomineeName (1086); nomineeList/nominee |
| DeletionMemberLandscape (dispatch) | always evaluated | `AspDeletionMemberLandscapeDispatchHandler.fetchDispatch` (:152) | UWPL_VI_ENDT_ADD_DEL_RISK | reportData "DeleteMemberLandscapeDispatch": transDet.deletionPrint (1323-1328) | transDet/deletionPrint (842) |
| DeleteMemberLandscape_ASP | DELETION_PRINT>0 | `AspDeleteMemberLandscapeHandler.fetch` | UWPL_VI_ENDT_ADD_DEL_RISK, CMGE_CODE | rows: contractId, nameExt, gender, dateOfBirth, idNo, joinDate, terminateDate, memberType, occupCode/Descp, plancode, prem, actionCode, llb/sti plan code & prem, relationship(+Desc), diffAgicFee, diffFeeSst, loading, mcoFee, imaFee, doubleIndemnity, insuredType, inpatientPrem; risk.riskLoading; cover.* (1330-1380). **Unmapped (no leaf)**: subcoverPrem, endtDoubleIndemnity, endtImaFee, endtStaxAmt, endtLlbPrem, endtStiPrem (comment at lines 1372-1374). Note XSD does contain `endtDoubleIndemnity`, `endtImaFee`, `endtLlbPrem`, `endtStaxAmt`, `endtStiPrem` under endorsementDetails (lines 607-617) - so the leaves exist at endorsement level, not transDet (verified in XSD); the Java comment says "no verified leaf" for transDet/PlineCover only. | transDet/*; endorsementDetails/endtStiPrem (617) |

Items marked stub/unimplemented/merged: EndtScheduleOfBenefits_ASP, ASPLLB, ASPSTI (stub); AntiRasuahBanner, SSTNote, ComplaintRedress body (no data, static); AspEndtDocument2Service and AspEndtDocumentLandscapeService (merged and disabled); `aspa*` leaves in the XSD (e.g. `aspaAdditionPrint` line 715, `aspaNomineePrint` line 334) belong to a different report family (ASPA), not used by the ASP Java (inference from names).

---------------------------------------------------------------------------------------------------

## 4. Jasper versus Java differences (each verified in both sides unless marked)

### 4.1 Conditions and gating
1. **Landscape dispatch is unconditional.** Jasper AddMemberLandscape/DeletionMemberLandscape are independent top-level reports. Java runs `addDispatchHandler.fetchDispatch` (line 1161) and `deletionDispatchHandler.fetchDispatch` (line 1323) on every EndtDocument call and always appends two dispatch rows (`AddMemberLandscapeDispatch`, `DeleteMemberLandscapeDispatch`) even when all counts are 0 (lines 1174, 1328). Test `minimalDefaults_skipsAllOptionalSubReportsButRunsLandscapeDispatch` (service test line 192) documents it as intended. Cost: 5-6 extra Mongo queries on every request.
2. **EndtDocument2 zero-row behavior is not reproduced.** Jasper prints nothing when both DIFF_PREM_DUE and DIFF_GST_AMT are zero (empty driver). Java's `buildCommonPipeline` always returns a DTO (`runAndMap` returns `new PlineEndtDocumentMainDto()` when empty; and PREM is computed as `$cond [{$ne: [$PB_ROW.DIFF_PREM_DUE, 0]}, 1, 0]` (AspEndtDocument2MainHandler.java:192). When `PB_ROW` is null, `$PB_ROW.DIFF_PREM_DUE` is missing and `{$ne: [missing, 0]}` is true in MongoDB, so PREM becomes 1 (inference from MongoDB semantics; no test covers it). Effect: debit note gate (`prem2 != 0`, service line 298) could open when Jasper's gate would not.
3. **Master-row anchor risk.** `AspEndtDocumentMainHandler` (HEAD) does `$match {ENDT_NO, CONTRACT_ID}` on collection `UWMT_VI_ENDT_TAX_INV_ALT_UPV` (lines 47-49, 128). The shared base class comment (documentHandler/PlineEndtDocument2BaseHandler.java:43-49) says that collection is "NOT related to this contract/endorsement" and that matching on it produced empty/default DTOs, and the base class fixed this by seeding with `$limit 1` only. The ASP override `buildCommonPipeline` (AspEndtDocument2MainHandler.java:126-128) re-introduced the `$match`. A not-on-HEAD commit `cc6a1b84d` (origin/OPUSM-41763/retrofit-develop-R2-pline-Template12-ASP-RejectRisk) rewrote `AspEndtDocumentMainHandler` to resolve each field independently ("The real query is select ... from dual ... a missing row in one lookup only leaves that one field null"). So on HEAD, an endorsement with no row in that anchor collection yields an all-null DTO: product code null, ENDT_CODE null, so 72/168 gate and 070412 gate close. Present as a risk with evidence, not as a confirmed production bug.
4. **Null safety.** Jasper `$F{ENDT_CODE}.equalsIgnoreCase("72")` and `$P{showCoverNote}.equalsIgnoreCase("Y")` throw NullPointerException on null; Java uses `"72".equalsIgnoreCase(x)` and defaults for missing filters ("N" for show flags, line 225-229) so Java is more tolerant (and prints "nothing" where Jasper would fail).
5. **Blank page.** Jasper prints "This Page Is Intentionally Left Blank" only when its band condition holds; Java sets `transDet.blank` on the master row always (line 445), leaving the renderer to decide (inference).
6. **AntiRasuahBanner gate.** See 1.3 note.
7. **EndtScheduleOfBenefits_ASP / ASPLLB / ASPSTI.** Jasper prints a full static page when ENDT_CODE is 72 or 168 (no product gate in this jrxml; the sibling report `EndtScheduleEndtDocumentService` has a stricter 070412+72-only gate for a different template, Javadoc lines 112-116). Java calls three stub handlers and emits no reportData, so the downstream renderer gets no row to trigger those pages except the master row's `actionCode` (ENDT_CODE). Whether the renderer derives it from actionCode is unknown (uncertainty U3).

### 4.2 Null and ordering
1. `COV_SEQ_REF` ordering: Oracle `ORDER BY cast(COV_SEQ_REF as decimal(15,2)) asc` puts NULLs last; Java uses `$toDouble` then `$sort` ascending (AspAddMemberLandscapeInsuredHandler.java:59-60; same in Premium and Nomination handlers). MongoDB sorts null/missing first, so rows with null COV_SEQ_REF would come first (inference). `$toDouble` of a non-numeric string throws, as Oracle's cast would (ORA-01722 equivalent).
2. Sponsor and DeleteMember have no ORDER BY in Jasper and none in Java (verified). Row order therefore equals Mongo natural order.
3. `RELATIONSHIP_DESC`: Jasper `SUBSTR(CODE_DESC,'4')` (strip first 3 characters); Java returns the full `CODE_DESC` (AspDeleteMemberLandscapeHandler.java:65-79). Unmapped difference.
4. `ROUNDED_NETT_PREM`: Oracle `ROUND(x)` rounds half away from zero; MongoDB `$round` rounds half to even (AspDebitNoteHandler.java:286-289). Edge case on .5 boundaries (inference from documented MongoDB behavior).
5. SHOW_OCCP_DESC: Jasper evaluates it per row from the subreport's issueDate (Date) vs system constant; Java evaluates it once per handler call (document-wide) and returns "N" if issueDate or the constant is null (AspAddMemberLandscapeInsuredHandler.java:145-153). The Jasper SQL does `TO_DATE($P{issueDate},'dd-mm-yyyy')` on a java.util.Date parameter (relies on NLS conversion) - fragile, not replicated.
6. INST_IND / SHOW_GST_LABEL on EndtSchedule: **Jasper computes** INST_IND from `SAPM_SYS_CONSTANTS` (`CUT_OFF_RPT`, `<product>_ENDT`) and PMT_FREQUENCY, and SHOW_GST_LABEL from the SST rollout date. **Java** fetches `PMT_FREQUENCY` from the schedule summary (AspEndtScheduleHandler.java:161, 661, 753) but never uses it, and sets `INST_IND` from `$_wupb.INST_IND` defaulting to "N" (line 794) and `SHOW_GST_LABEL` from `$_wupb.SHOW_GST_LABEL` defaulting to "N" (line 786). So the cut-off constants are not evaluated in the schedule handler. (Grep: no `CUT_OFF_RPT` or `Rollout` in Asp*Handler files apart from AspServiceTaxInvoiceHandler's `EINV_CUT_OFF`.) The same applies to DebitNote's SHOW_GST_LABEL (Javadoc comment "intentionally not projected", AspDebitNoteHandler.java:52-53, 364). In the master row the label comes from UWGE_VI_ENDT_REPORT_PARAM, which does exist in the main query.
7. Several EndtSchedule fee-diff fields in Java default to `0` when `_wupb` lacks them (AspEndtScheduleHandler.java:760-770), whereas Jasper sums `DIFF_FEE_AMT` by fee code from the cover-fee tables (WIP_UWGE_COVER_FEES / SB_UWGE_POLICY_FEES, J EndtSchedule.jrxml.data:259-441). This is a data-source difference (inference: the author chose the base-table path in the Javadoc "Bypasses UWPL_VI_ENDT_SCHEDULE_SUMMARY", lines 22-24).
8. EndtCover SQL: Jasper uses outer joins to DMAG and DMAG_RI and `BR.BRANCH_CODE = NVL(DMAG.BRANCH_CODE, BR.BRANCH_CODE)`; Java's Javadoc (AspEndtCoverHandler.java:36-46) describes a simpler chain with an inner join to DMAG and BR and no DMAG_RI. The Java pipeline keeps the row when no agent matches (nested `$lookup`, left-outer style) and applies `$limit 1`. In Jasper, if DMA is missing, the `NVL(...)` predicates become true for every BR row, which would return one row per branch (inference). So behavior in the "agent not found" case differs. Also `buildMatch` returns an empty `Document` when both endtNo and refNo are blank (lines 188-201), so Java would return an arbitrary first row in that case; Jasper would return no rows (`ENDT_NO = null` is never true). Both are inference-level edge cases.

### 4.3 Oracle error emulation (ORA-01427)
- Implemented only in `AspServiceTaxInvoiceHandler`: `requireSingleValue` (lines 388-398) for EINV_CUTOFF and each VERSION_NO subquery, and `requireNoConflict` (lines 404-411) for the live-vs-WIP VERSION_NO UNION; both throw `IllegalStateException("... Oracle's scalar subquery would raise ORA-01427 here")`. Tests: `AspServiceTaxInvoiceHandlerTest.fetchServiceTaxInvoice_ambiguousEinvCutoff_throwsIllegalStateException` (line 373) and `..._conflictingLiveAndWipVersionNo_throwsIllegalStateException` (line 387). Potential false conflict: `liveValue.equals(wipValue)` on Integer vs Long returns false for equal numbers (inference).
- NOT implemented elsewhere in ASP code even though the Jasper SQL has the same patterns: (a) the master EndtDocument/EndtDocument2 ERP sub-selects (no ROWNUM) - Java takes `$arrayElemAt ... 0`; (b) AddMemberLandscape `addition_print` and `deletion_print` `UNION ALL` scalar subqueries return two rows if both WIP and committed rows exist, Oracle raises ORA-01427 - Java's `additionExists` / `deletionExists` simply return true (AspAddMemberLandscapeDispatchHandler.java:85-121); (c) `ISSUE_DATE` sub-select in the dispatch query - Java uses `findOne`; (d) DebitNote `MST_CONTRACT_ID` scalar sub-select - Java takes `$limit 1` (AspDebitNoteHandler.java:158) and swallows any exception to `premByMstInd="N"` (lines 181-183). Other plines handlers (e.g. EndtScheduleEndtDocumentMainHandler) do emulate ORA-01427 in the same situations; the ASP ones do not.

### 4.4 Type mismatch CONTRACT_ID (String versus numeric)
- Jasper: main pContractId Integer; landscape subreports String (section 2.3). Mongo stores CONTRACT_ID numerically, so any raw String match returns 0 rows silently.
- Java fixes: `safeNum()` in every ASP handler: Long.parseLong (AspAddMemberLandscapeDispatchHandler.java:123, AspDebitNoteHandler.java:393, AspEndtScheduleHandler.java:836, AspEndtDocumentMainHandler.java:154), and Integer-then-Long in `AspServiceTaxInvoiceHandler.safeNum` (lines 461-465). `AspEndtDocument2MainHandler.buildCommonPipeline` Javadoc (lines 102-115) explains the exact bug (all `showXxxInd` null and the gated subreports "silently never fire") and the ASP-only override. History: commits 578778080 and ba9e29586 ("CONTRACT_ID type-mismatch fixes") on the OPUSM-41763-backup branch.
- Reverse conversion in the response: `PlineServiceTaxInvoiceDto.contractId` is a String (`toStringOrNull`, handler line 281), later `toBigDecimal(...)` for `transDet.contractId` (service 746-749). The XSD types `endorsementDetails/contractId` as decimal (line 600), `mstContractId` as integer (line 1050). Test `fetchServiceTaxInvoice_contractIdNonNumeric_keptAsStringInMatch` (line 523) covers the non-numeric fallback.

### 4.5 Stubs and unmapped fields
- Stubs: `PlineEndtScheduleOfBenefitsAspHandler`, `PlineAspLlbHandler`, `PlineAspStiHandler` (each returns `dualValue=1`; tests have 1 test each). `AspEndtDocumentService` lines 268-278 call them fire-and-forget.
- Fire-and-forget (called, nothing mapped): `AcpComplaintRedressHandler.fetchComplaintRedress`, `PlineGenericAntiRasuahBannerHandler.fetchAntiRasuahBanner`, SSTNote (not called at all).
- Unmapped fields in the delete-member rows: subcoverPrem, endtDoubleIndemnity, endtImaFee, endtStaxAmt, endtLlbPrem, endtStiPrem (service comment 1372-1374).
- The `AspDeleteMemberLandscapeDto` comment says the unmapped list is "same as AspEndtDocumentLandscapeService's own gap" (legacy gap carried over).
- The delete-member row also does not map `INSURED_TYPE` description beyond `det.setInsuredType` (it does map), and `ISSUE_DATE` sub report parameter is not used.

### 4.6 Other
- `showComplaint` is accepted but never used to gate ComplaintRedress; the Jasper gate is the product code 070412 only (verified in both sides: service line 288, EndtDocument2 band 1). `showComplaint` is echoed into reportParameters (line 1506).
- `pSchemeName` (used by EndtSchedule layout for product 070504 schemes INCOMESHIELD, ELITESHIELD, SNATCHPA-A, SNATCHPA-M) is not read from the filter map in Java at all (not in the list at lines 215-230). So scheme-specific layout choices cannot be made from the Java output unless the renderer gets it elsewhere (inference; uncertainty U6).
- Date formatting: Jasper passes `issueDate` as `dd-MMM-yyyy` to EndtCover/EndtSchedule subreports but the sub-sub queries parse with `to_date(issueDate,'dd/MM/yyyy')`; Java passes `yyyy-MM-dd` strings (`formatDate`, lines 1659-1664) to the common handlers. I did not open the common handlers to confirm how they parse, so this is an unverified edge (U7).

---------------------------------------------------------------------------------------------------

## 5. What does "ASP" stand for? (evidence only)

Direct expansion: **not found.** I searched every `*.jrxml.data` in `ASP/` for titles, static text, captions and clause text (grep for "ASP", "Accident", "Student", "Allianz", etc.).

What the evidence shows:
- Internal jasperReport names of the two nested pages are `ASPlusSTI` (ASPSTI.jrxml.data line 2, `name="ASPlusSTI"`) and `ASPlusLLB` (ASPLLB.jrxml.data line 2, `name="ASPlusLLB"`). This suggests a plan or product called "ASPlus" or "A.S.Plus" (inference; the "ASP" prefix on the files may be shorthand for it).
- Benefit codes on the schedule-of-benefits page: `ASP-A1`..`ASP-A10`, `ASP-B1`..`ASP-B10`, `ASP-C1`..`ASP-C5`, `ASP-D1`..`ASP-D5` (EndtScheduleOfBenefits_ASP.jrxml.data lines 116-1722), with a "PLAN" header and "Optional Benefits".
- The benefits are personal-accident style: "Principal Sum Insured Accidental Death & Permanent Disablement (up to)", "Medical Expenses (up to)", "Weekly Benefit for Occupation Class 1 and 2 / Class 3", "Hospital Income (per day/max. 180 days)", "Double Indemnity in the event of Motor Vehicle Accident (Only applicable Occupation Class 1 and 2)", "Permanent Impotency or Infertility", "Miscarriage (due to Accident)", "Blood Transfusion", "Bereavement Allowance", "Funeral Expenses", "Repatriation Expenses", "Ambulance Fee", "Mobility Expenses", "Compassionate Care", "Snatch Theft or Attempted Snatch Theft", "International and Domestic Medical Assistance and Evacuation", "Renewal Bonus". Optional pages: "Lifestyle and Living Benefit Plan" (LLB) and "Study Interruption" (STI: "applicable for the Insured Person who is a registered full time student").
- Landscape tables use group-insurance language: "Name of Insured Person(s)/ID No./DOB (Age)/Designation (Class)", "Category/Relationship to Principal", "ADDITION OF MEMBER(S)", "PARTICULAR OF SPONSOR(S)", "PARTICULAR OF NOMINEE(S)". So it is a group (scheme) personal-accident product with sponsors (the sponsor is likely a person paying for a student insured; inference).
- Java ties these ASP pages to **product code 070412**: `Template4PolicyDocumentService.java:404-431` (comment "ScheduleOfBenefits_ASP.jrxml (+ nested ASPLLB/ASPSTI): product=070412", "PerilsClauses_ASP.jrxml: product=070412 && aspCutoff=Y", "ParticularOf*Potrait_ASP.jrxml ... 070412 && aspCutoff=Y") and the EndtDocument2 ComplaintRedress gate is also 070412. So 070412 is ASP's product config code in the policy-document family (inference; the product name itself is never printed).
- Do not claim "ASP = Accident Student Plan" or any expansion in an interview; say "the jrxml calls it ASPlus internally, benefit codes ASP-A..D, and Java ties it to product 070412; I did not find a written expansion".

---------------------------------------------------------------------------------------------------

## 6. Business rules and where Java implements them

| Rule | Where in jrxml / SQL | Where in Java |
|---|---|---|
| Product 070412 gates the Complaint/Redress page | EndtDocument2 band 1 `PRODUCT_CODE.equalsIgnoreCase("070412")` (line ~175) | `AspEndtDocumentService.java:288` (fetch) and `:1414` (reportData); `mainDto2.getProductCode()` |
| Endorsement code 72 or 168 = add members (also schedule-of-benefits + blank) | EndtDocument bands 4 and 5; AddMemberLandscape `endt_code in ('72','168')` | `AspEndtDocumentService.java:274`; `AspAddMemberLandscapeDispatchHandler.additionExists` (lines 85-110, `.in("72","168")`) |
| Endorsement code 98 = delete members; 168 also in the delete-member rows | AddMemberLandscape `deletion_print endt_code in ('98')`; DeleteMemberLandscape_ASP `ENDT_CODE IN ('98','168')` | `deletionExists` (lines 113-121); `AspDeleteMemberLandscapeHandler` match on `ENDT_CODE in (98,168)`, `ACTION_CODE=C`, `TERMINATE_DATE != null` (lines 50-52) |
| Endorsement code 999 excluded from EndtMT block | ServiceTaxInvoice line 1716 `!ENDT_CODE.equalsIgnoreCase("999")` | `AspEndtDocumentService.java:370` |
| Debit versus credit note | `case when PREM_DUE>=0 then 'D' else 'C' end \|\| ENDT_NO` (note number), `'DEBIT'/'CREDIT'` (flag); titles PREMIUM DEBIT NOTE / PREMIUM CREDIT NOTE | `AspDebitNoteHandler.java:291-297` (`$gte PREM_DUE 0`); mapped to `dnDet.setDebitNoteNo`, `setDebitCreditFlag` (service 563-564). ServiceTaxInvoice uses GROSS_PREM >= 0 instead (handler lines 220-221) - same as Jasper line 78 |
| Debit note only when premium changed | band `showDebitNote=Y && PREM != 0` where PREM = (DIFF_PREM_DUE != 0) | service line 298 `"Y".equalsIgnoreCase(showDebitNote) && prem2 != 0` |
| Tax invoice (GST) gate | `showTaxInvoice=Y && SHOW_GST_INV_IND=Y` | service line 314 |
| Service tax (SST) e-invoice gate | `showTaxInvoice=Y && SHOW_STAX_INV_IND=Y && !checklistCopy=Y && SHOW_EINV_IND=Y` | service lines 335-338 (`showSt`) |
| Transitional invoice gate | `showTaxInvoice=Y && SHOW_TRAN_INV_IND=Y` | service line 406 |
| E-invoice cut-off | `ISSUE_DATE >= EINV_CUTOFF` (SAPM_SYS_CONSTANTS UW_VALIDATION / EINV_CUT_OFF) controls the e-invoice header blocks in ServiceTaxInvoice (lines 922-1181) | read in handler (`einvCutoff` literal, handler lines 145-151, 279) and written to `stDet.eInvCutOff` (service line 792); the comparison itself is a rendering rule left to the renderer (inference) |
| QR section | `EINV_CONSENT_IND=Y && EINV_STATUS=VALID && QR_IND=Y` | service lines 379-383 |
| Multi-rate service tax block | `STAX_COUNT>1` (count of non-zero DIFF_STAX_AMT rows) | `AspServiceTaxInvoiceHandler.computeStaxCount` (lines 413-455) summing WIP and committed counts; swallowed errors give 0 |
| Master policy (MST) handling | `MST_CONTRACT_ID` = master policy's contract via MST_POL_NO -> OCP_POLICY_BASES.POLICY_REF at VERSION_NO=1; `PREM_BY_MST_IND` from master's UWGE_POLICY_BASES; when Y, the invoice shows the master policy holder (CustomerDtlsOfMasterPolicy(_Debit)) and "Tax Identification No." block; when not Y, e-invoice buyer block | `AspDebitNoteHandler.fetchMstContractInfo` (lines 131-185); used in service lines 302-307 (debit) and 352, 392-395 (service tax); MASTR_TIN_NO / BRN / SST in `AspServiceTaxInvoiceHandler` lines 304-334 (TIN masked to last 4 -> `****`) |
| Sponsor and nominee | landscape sub reports gated by `SPONSOR_PRINT>0` / `NOMINEE_PRINT>0`; count of rows with `SPONSOR_NAME` / `NOMINEE_NAME` not null | dispatch handler lines 42-56; service lines 1286-1321 |
| Cut-off constants | `CUT_OFF_RPT / <PRODUCT>_ENDT` -> INST_IND; `OCCUP_CUT_OFF_DATE / OCCUP_DATE` -> SHOW_OCCP_DESC; `UW_VALIDATION / EINV_CUT_OFF`; SST rollout date function | OCCUP_DATE and EINV_CUT_OFF implemented (Insured/Premium handler; ServiceTax handler); `CUT_OFF_RPT` and rollout function not implemented in ASP schedule/debit handlers (4.2 #6); rollout date is fetched via `PlineSSTRolloutDateClient` in ServiceTax only (handler lines 85-90) |
| Special product fee formula | products 089332, 089432, 089329, 089429 add ASST/CRF/OCR fee diffs into gross premium | `AspDebitNoteHandler` (lines 273-283), `AspServiceTaxInvoiceHandler.grossPremiumExpr` (lines 339-349); not in `AspEndtScheduleHandler` (Jasper does it in EndtSchedule query line 164) |
| Banca agent | `DECODE((SELECT ... FROM CMUW_BANKBRANCH ...),NULL,'N','Y')` | `AspDebitNoteHandler` lines 239-243, 311-312 -> `agentInfo.bancaAgent` |
| TPA = MEDIX special layout | EndtSchedule `TPA_NAME` checks | `tpaName` mapped to transDet (service 485); layout decision left to renderer |
| Checklist copy (WIP) versus committed | `source='WIP'` branches, `checklistCopy` Y/N | main handlers' `chk` lookups; `showSt` excludes checklist copies |
| Print series footer | PRN_CNT from UWGE_POLICY_VERSIONS_CTRL | `AcpPrintSeriesFooterHandler.fetchPrintCount`, `transDet.prnCnt` |
| QR / e-invoice consent | EINV_CONSENT_IND NVL 'N' from UWGE_POLICY_BASES | `AspEndtDocument2MainHandler.java:239-244`; `transDet.eInvConsentInd` |

---------------------------------------------------------------------------------------------------

## 7. Git history of ASP code

Command used: `git log --all --format='%h|%ad|%an|%s' --date=short -- <Asp* / PlineAsp* / PlineEndtScheduleOfBenefitsAsp* in main and test>` plus `git merge-base --is-ancestor <hash> HEAD` for each hash. I do not attribute any commit to the developer who asked for this file; names below are exactly what git prints.

| Hash | Date | Author (as in git) | Subject | On HEAD? |
|---|---|---|---|---|
| 861f0666c | 2026-10-08 | Shinge | OPUSM-44159 Added report, sub report name in listing report | yes |
| e74bc0d5c | 2026-10-07 | Shinge | OPUSM-44159 Added report, sub report name in resposne | yes |
| cc6a1b84d | 2026-10-05 | GS | OPUSM-41763 ASP RejectRisk/EndtSchedule/EndtDocument fixes | **no** (origin/OPUSM-41763/retrofit-develop-R2-pline-Template12-ASP-RejectRisk) |
| c336f76fa | 2026-10-01 | Shinge | OPUSM-44159 fixed sonar issues | yes |
| db7588897 | 2026-09-30 | Karunakar | OPUSM-44159 Add test coverage for plines.service/client/config/utils and SharedReport | yes |
| 3f2157b28 | 2026-09-29 | Shaw | OPUSM-44159 Test cases added for plines | yes |
| 24e50ab9b | 2026-09-29 | Shaw | OPUSM-44159 Sonar build and error fixes | no (origin/OPUSM-44159-retrofit-develop-pline-jrxml-comparison-fixes, origin/feature/retrofit-develop-pline) |
| 660044b26 | 2026-09-29 | Shaw | OPUSM-44159 Sonar fixes | no (same branches) |
| a0c3a8939 | 2026-09-26 | extern.neethunivas-subrahmannian_neethu | OPUSM-44159 Fix test failures surfaced by full-suite verification across documentHandler batch | no |
| 680f55bc5 | 2026-09-25 | same | Merge branch test-coverage/plines-documentHandler-1-batchC | no |
| df70d6b5c | 2026-09-25 | same | OPUSM-44159 Add test coverage for Alp/Asp documentHandler batch B | no |
| 8fa89231e | 2026-09-25 | same | OPUSM-44159 Add test coverage for Asp/AutoRenewal documentHandler batch C | no |
| 0e2afadab | 2026-09-25 | extern-kumari-soni-manisha | Merge pull request #917 from OPUSCCC/test-coverage/plines-documentdto-1 | no |
| 8048d95bf | 2026-09-25 | Shinge | OPUSM-44159 Add test coverage for plines.requestpayload (352 classes) | no |
| 0ab953cf8 | 2026-09-25 | Kumari Soni | OPUSM-44159 Add JUnit5 tests for documentdto batch (Asp/Auto/Coverage/...) | no |
| 755c8083f | 2026-09-24 | Shinge | OPUSM-44159 XSD fixes | no |
| a6b15c418 | 2026-09-22 | Varsha | OPUSM-41697: fix ASP EndtDocument Landscape/Service report data handling | no (origin/OPUSM-41697/retrofit-develop-R2-pline-underWritingErrors) |
| da95d245c | 2026-09-22 | Shinge | OPUSM-44159 spring boot update and code changes | no (spring-boot-4-upgrade branches) |
| de2cf813e | 2026-09-21 | GS | Merge branch feature/retrofit-develop-R2-pline ... | no |
| 94ea23d81 | 2026-09-21 | GS | ASP EndtDocument: fix EndtCover/EndtSchedule field mapping gaps, restore Template12PolicyDocument2Service (adds AspServiceTaxInvoiceHandler, +472 lines) | no |
| fc2303da0 | 2026-09-16 | D | OPUSM-40978 - AutorenewalDocument | no |
| 532d1688e | 2026-09-11 | Shinge | OPUSM-42246 Multiple Report data for landscape | no |
| ba9e29586 | 2026-08-26 | GS | OPUSM-41763 Fix ASP EndtDocument/EndtDocument2 data mapping and CONTRACT_ID type bugs | no (origin/OPUSM-41763-backup) |
| 578778080 | 2026-08-25 | GS | OPUSM-41763 ASP EndtDocument field-mapping and CONTRACT_ID type-mismatch fixes | no (origin/OPUSM-41763-backup) |
| 4cd1cad2e | 2026-08-31 | Shinge | OPUSM-42246 Working code for pline | yes |
| 7a925920c | 2026-08-28 | GS | OPUSM-36054 OPUSM-40884 Wire ASP EndtDocument sub-reports and fold Landscape reports into one response (+2657 lines: AspDebitNoteHandler, AspEndtCoverHandler, AspEndtScheduleHandler, DTOs, service) | no |
| 4babbaa1e | 2026-08-26 | extern-yadav-amarnath | OPUSM-40884 Added dynamic merging for ACPEndtDoc | no |
| 005cf0779 | 2026-08-26 | Varsha | OPUSM-41697 AspLandscapeImplementation-14files (7 handlers, 7 DTOs, AspEndtDocumentLandscapeService 626 lines) | no |
| 4630687db | 2026-08-19 | Karunakar | OPUSM-42267, OPUSM-41535 and OPUSM-41791 EndtSchedule XSD changes | yes |
| b4e588bd4 | 2026-08-11 | GS | final_T12_code | yes |
| 6961abbf9 | 2026-08-08 | Shinge | Align Pline requestpayload types with XSD and add schema validation test | yes |
| d4575fa5b | 2026-08-03 | Shinge | Code merged with XSD changes for Template 12 and Template4 landscape | yes |
| d4c069135 | 2026-07-16 | Shinge | AutoRenewalDocument with XSD changes | yes |
| d90cb9c6b | 2026-07-08 | D | AutoRenewalDocument | no |
| 8d1b5374c | 2026-07-06 | Shinge | Add Plines module implementation with handlers, services, DTOs and bug fixes | no (initial module commit on feature/plines-auto-policy-successful) |

Themes (my summary of the subjects and `--stat` output):
1. Jul-Aug 2026: initial Plines module, XSD alignment (Template12/Template4 landscape), AutoRenewal.
2. Aug 2026: ASP landscape implemented as a separate service + 7 handlers + 7 DTOs (005cf0779), then folded into the EndtDocument response (7a925920c).
3. Aug-Sep 2026: field-mapping and CONTRACT_ID type-mismatch fixes (578778080, ba9e29586, 94ea23d81), new `AspServiceTaxInvoiceHandler`.
4. Sep 2026: "Multiple report data" (1 subreport = 1 reportData) conversion (532d1688e, a6b15c418), XSD fixes (755c8083f, ea782121d for Pline.xsd).
5. Sep-Oct 2026: unit-test batches (0ab953cf8, 8048d95bf, 8fa89231e, df70d6b5c, a0c3a8939, 3f2157b28, db7588897), Sonar fixes (660044b26, 24e50ab9b, c336f76fa).
6. Oct 2026: reportName/subReportName in response (e74bc0d5c, 861f0666c).
Not on HEAD but worth knowing: `cc6a1b84d` (2026-10-05, ASP RejectRisk/EndtSchedule/EndtDocument fixes, 751 insertions) adds `AspCompanyNameWithGSTRegNoHandler` (not present on HEAD), rewrites `AspEndtDocumentMainHandler` to per-field lookups, and extends `AspEndtScheduleHandler` (+333 lines). If the interviewer asks "what is still missing on your branch", this is the answer (to the extent the commit content is complete; I only read the main handler diff).
Another not-on-HEAD item: `ea782121d` (Kumari Soni, 2026-09-28, Pline.xsd, "Multiple ReportData changes for Template25, 26, 6") appears in the XSD log; the XSD on HEAD predates the mainReportName/subReportName fields.
Working tree: `git status` shows only RejectRisk files and application.properties modified; no ASP changes uncommitted.

---------------------------------------------------------------------------------------------------

## 8. Tests

Counts are `@Test` / `@ParameterizedTest` annotations per file (Grep count), so a parameterized test counts once.

### 8.1 ASP EndtDocument-related test classes (src/test/java/com/agcs/fo/opus/plines/...)
- service/AspEndtDocumentServiceTest.java - **28** tests, tests the live merged bean (Mockito mocks for all handlers).
- service/AspEndtDocument2ServiceTest.java - 12; service/AspEndtDocumentLandscapeServiceTest.java - 8 (these two test the disabled legacy classes, kept for reference).
- documentHandler (13 ASP handler tests): AspServiceTaxInvoiceHandlerTest 27, AspDebitNoteHandlerTest 18, AspAddMemberLandscapeDispatchHandlerTest 11, AspEndtScheduleHandlerTest 9, AspEndtDocument2MainHandlerTest 8, AspEndtCoverHandlerTest 8, AspParticularOfSponsorLandscapeHandlerTest 8, AspAddMemberLandscapeInsuredHandlerTest 7, AspAddMemberLandscapePremiumHandlerTest 7, AspEndtDocumentMainHandlerTest 6, AspParticularOfNominationLandscapeHandlerTest 6, AspDeleteMemberLandscapeHandlerTest 5, AspDeletionMemberLandscapeDispatchHandlerTest 3; stub handler tests PlineAspLlbHandlerTest 1, PlineAspStiHandlerTest 1, PlineEndtScheduleOfBenefitsAspHandlerTest 1.
- documentdto (POJO getter/setter style): AspDebitNoteDtoTest 15, AspEndtScheduleDtoTest 10, AspEndtCoverDtoTest 9, AspDeleteMemberLandscapeDtoTest 9, AspAddMemberLandscapeInsuredDtoTest 9, AspAddMemberLandscapePremiumDtoTest 9, AspAddMemberLandscapeDispatchDtoTest 5, AspDeletionMemberLandscapeDispatchDtoTest 5, AspParticularOfSponsorLandscapeDtoTest 5, AspParticularOfNominationLandscapeDtoTest 5, PlineAspLlbDtoTest 5, PlineAspStiDtoTest 5.
- requestpayload: PlineAspLlbDetailsTest 6, PlineAspStiDetailsTest 6.
- Sum for the EndtDocument family: 48 (service) + 126 (handlers incl. 3 stubs) + 91 (DTOs) + 12 (payload) = **277** (my own arithmetic from the counts above; the full `*Asp*` file glob gives 458 across 60 files because it also includes Template12 portrait/landscape, AutoRenewal and Letter-to-policyholder ASP classes).
- XSD check: `PlineXsdValidationTest` (src/test/java/com/agcs/fo/opus/plines/PlineXsdValidationTest.java) loads `/xsd/plines/Pline.xsd` (line 50); I did not read it to see whether an ASP response is among the validated samples (U8).

### 8.2 Scenarios covered (examples)
Service test (AspEndtDocumentServiceTest): minimal defaults (only dispatch handlers called, line 192); cover note fetch (207); schedule + signature (222); endt code 72 fetches the three stubs, 99 skips (234, 247); 070412 triggers ComplaintRedress and company-name handler (258); debit note with prem != 0 and with MST flag; zero prem skips (270, 290, 831, 848); GST tax doc with banner (297); full SST gate and EndtMT rows when STAX_COUNT>1 (310, 865, 883); checklistCopy=Y closes SST (340); QR gate (903); master-policy via service-tax path (921); transitional invoices (353, 951); landscape addition/sponsor/nominee/deletion gates (365-424, 457, 966); order of reportData (424); report parameters echo (464); a "fully populated" row mapping test (656). 
Handler tests: dispatch handler WIP/committed branches, fall-through, nominee/sponsor pass-through, issue date, non-numeric and null contract ids (AspAddMemberLandscapeDispatchHandlerTest lines 60-180); ServiceTax handler: null rows, scalar literals, client exceptions swallowed, MST TIN/BRN/SST mapping with fallbacks, QR default "N", ORA-01427 style exceptions, stax count branches, non-numeric contractId (lines 139-523); main handlers: empty results -> default DTO, WIP vs live branches, numeric coercion of contractId, e-inv indicator normalisation; schedule handler: match shape (OR / only endt / only ref / neither).

### 8.3 Gaps (my assessment)
1. Mongo pipelines are mostly tested with a mocked `MongoTemplate` returning canned results and by inspecting generated stages (e.g. "forces WIP pipeline branch"), so they prove the stage shape, not the data semantics (no embedded-Mongo or integration test seen).
2. No test of the zero-row EndtDocument2 behavior or the `$ne missing 0` PREM edge (4.1 #2).
3. No test that INST_IND / SHOW_GST_LABEL follow the Oracle cut-off rules (they are not implemented).
4. No negative test for ORA-01427 in the dispatch handlers (not implemented).
5. No test for `RELATIONSHIP_DESC` SUBSTR parity, COV_SEQ_REF null ordering, ROUND half-even.
6. No golden-file comparison between Jasper output (PDF text) and Java response for a real ASP endorsement.
7. Stub tests only assert `dualValue=1`.
8. Tests for `AspEndtDocument2Service` and `AspEndtDocumentLandscapeService` cover disabled code (maintenance cost without runtime value).
9. Service test covers `reportParameters_echoedFromFilters`, but `pSchemeName` is not part of it because the service does not read it.

### 8.4 The odd copy of AspServiceTaxInvoiceHandler.java in src/test
- File: `src/test/java/com/agcs/fo/opus/plines/documentHandler/AspServiceTaxInvoiceHandler.java` (483 lines, `package com.agcs.fo.opus.plines.documentHandler`, `@Slf4j @Component`, **no `@Test`**) next to the real test `AspServiceTaxInvoiceHandlerTest.java`.
- The main copy has 476 lines; `diff` shows only comment text and try/catch formatting differences (lines 428, 450, 463-467, 471-473), consistent with a Sonar-driven edit applied to the test copy.
- Git: added/edited by commits `3f2157b28` (Shaw, 2026-09-29, "Test cases added for plines", on HEAD) and `24e50ab9b` (Shaw, 2026-09-29, "Sonar build and error fixes", not on HEAD).
- Consequence (inference): a class with the same fully qualified name exists in both source sets. At test time the test-classes directory normally precedes main classes on the classpath, so `AspServiceTaxInvoiceHandlerTest` may be exercising the test copy, not the production class; changes to the production file might not be detected by the tests. It is also dead weight in the repository. Recommended action: delete the test copy and re-run the test class (not done; out of scope for this read-only task).

---------------------------------------------------------------------------------------------------

## 9. Twenty-five interview questions (1.6 years of experience level)

Format: Q, strong answer, follow-ups, common mistakes.

**Q1. In one minute, what are the ASP reports and what did you work on?**
A: ASP is a group personal-accident style product family in PLine. There are four Jasper top-level reports for endorsements: EndtDocument (cover, schedule, schedule of benefits), EndtDocument2 (complaint page, debit/credit note, tax and e-invoices, transitional invoices), AddMemberLandscape and DeletionMemberLandscape (landscape tables). My work covered analysis of each report with all sub levels, Java code in the merged service `AspEndtDocumentService` and Asp* handlers, unit tests, and the Pline XSD mapping. Follow-up: how many subreports? Roughly 12 in EndtDocument tree and 15+ in EndtDocument2 per the hierarchy csv. Mistake: saying ASP stands for something specific without evidence.

**Q2. What does ASP stand for?**
A: I did not find a written expansion. The jrxml uses internal names `ASPlusSTI` and `ASPlusLLB`, benefit codes `ASP-A1..D5`, and Java comments tie it to product 070412. Follow-up: how do you know it is 070412? Template4PolicyDocumentService comments and the EndtDocument2 complaint gate. Mistake: inventing "Accident Shield Plan".

**Q3. Why one Spring bean for four Jasper reports?**
A: The new Pline response is one JSON/XML envelope per request; separate beans meant separate calls and duplicated main queries. The merge ported every band with its original print condition into `AspEndtDocumentService` and disabled the old `@Service`/`@Bean` registrations to avoid duplicate bean keys. Follow-ups: what is the risk of re-enabling the old beans (duplicate rendering/bean collision, class Javadoc lines 87-96). Mistake: claiming the old classes were deleted (they are kept for reference).

**Q4. How is the output structured?**
A: `transaction/transactionDetails/headerTransaction/reportTransaction/reportData[]`. reportData[0] is the master (mainReportName `ASPEndtScheduleEndtDocument`, subReportName `PrintSeriesFooter`); each subreport that fired adds its own reportData with its own prntDetails/transDetList/transDet. Follow-up: why one per subreport? So the renderer can bind each Jasper page to its own data. Mistake: saying one big transDet.

**Q5. Where do mainReportName and subReportName come from and are they in the XSD?**
A: Set in `sectionReportData` and constants lines 168-171, tagged by `PlineReportNames.tag`. The XSD copy under src/test/resources does not contain them; PlineReportNames mentions Pline.xsd v0.1.10, so the authoritative XSD is elsewhere. Follow-up: what does `mainNameOf` return? Parent segment + last segment unless last already starts with parent. Mistake: claiming the repo XSD defines them.

**Q6. Explain the print conditions of EndtDocument.**
A: Cover and a blank page when showCoverNote=Y; EndtSchedule when showEndtSchedule=Y; another blank and the schedule of benefits (with ASPLLB and ASPSTI) when ENDT_CODE is 72 or 168. PrintSeriesFooter is in the background band always. Follow-up: what is the code for? 72/168 = member addition endorsements per AddMemberLandscape SQL. Mistake: adding a product-code condition that the jrxml does not have (the Javadoc explicitly warns about it).

**Q7. Explain EndtDocument2's driving query and a surprising consequence.**
A: The main query is built on a UNION of WIP and committed policy bases where DIFF_PREM_DUE or DIFF_GST_AMT is non-zero, so PREM and TAX_INV flags are derived from it. If both diffs are zero, there is no driving row and nothing prints. Follow-up: does Java reproduce this? No, flagged as a gap. Mistake: thinking `from dual`.

**Q8. Which conditions control the invoice family?**
A: showTaxInvoice=Y plus flags from UWGE_VI_ENDT_REPORT_PARAM: SHOW_GST_INV_IND for EndtTaxDoc, SHOW_STAX_INV_IND with not checklist copy and SHOW_EINV_IND for ServiceTaxInvoice, SHOW_TRAN_INV_IND for the transitional pair. Follow-up: what about QR? EINV_CONSENT_IND=Y, EINV_STATUS=VALID, QR_IND=Y. Mistake: forgetting checklistCopy excludes SST e-invoice.

**Q9. How do debit and credit notes work?**
A: DebitNote view `UWPL_VI_ENDT_DEBIT`; `PREM_DUE >= 0` gives 'D' prefix and flag DEBIT, otherwise 'C' and CREDIT; titles switch accordingly. Java: `AspDebitNoteHandler` uses `$gte PREM_DUE 0`, and the service sets debitNoteNo and debitCreditFlag per row. Follow-up: ServiceTaxInvoice uses GROSS_PREM instead — explain. Mistake: assuming both use the same field.

**Q10. What is the MST / PREM_BY_MST_IND logic?**
A: A member policy can point to a master policy (MST_POL_NO). The query finds the master's contract id through OCP_POLICY_BASES (version 1), then reads its PREM_BY_MST_IND. When Y the document shows the master policyholder details and master TIN/BRN/SST; when not Y it shows the normal buyer block. Java: `AspDebitNoteHandler.fetchMstContractInfo`, reused for service tax. Follow-up: what if the lookup fails? It logs a warning and returns "N" (silent). Mistake: not noting the swallowed exception.

**Q11. What was the CONTRACT_ID type mismatch?**
A: Jasper subreports declared pContractId as String while Oracle compares with a numeric column (implicit cast). Mongo stores CONTRACT_ID as a number, so a String filter matches nothing and the gated subreports silently disappear. We convert with `safeNum` (Long.parseLong, fallback to the raw string). Follow-up: where is the override for EndtDocument2? `AspEndtDocument2MainHandler.buildCommonPipeline`. Mistake: "just use toString on both sides".

**Q12. What is ORA-01427 and how did you handle it?**
A: "single-row subquery returns more than one row": Oracle fails the query when a scalar subquery without ROWNUM/DISTINCT finds several rows. In Java we emulate it only in `AspServiceTaxInvoiceHandler` by throwing IllegalStateException when EINV_CUTOFF or VERSION_NO is ambiguous. Honest follow-up: other ASP handlers do not emulate it (dispatch UNION ALL, ERP sub-selects). Mistake: claiming full parity.

**Q13. Which Jasper features are stubs in Java and why?**
A: EndtScheduleOfBenefits_ASP, ASPLLB and ASPSTI are `SELECT 1 FROM DUAL` with static text, so the Java handlers return dualValue=1 and the service does not emit rows. Risk: the renderer has only the master row's actionCode to know to print them. Follow-up: would you add a marker reportData? Probably yes with subReportName set; confirm with the renderer team. Mistake: saying data is missing — the pages are static.

**Q14. Why is the landscape dispatch unconditional and is that a problem?**
A: Original landscape reports were separate top-level reports with their own dispatch SQL; in the merge both dispatch handlers run on every call and always add two dispatch rows. It costs extra queries but keeps the contract simple; counts of 0 tell the renderer not to print. Follow-up: how would you gate it? Only when ENDT_CODE is in (72,168,98) from the main DTO. Mistake: saying Jasper EndtDocument calls the landscape reports (it does not; verified in both jrxml).

**Q15. How does the addition_print dispatch query work?**
A: For WIP: versions row by REF_NO with endt_code in (72,168) and the contract has a risk with ACTION_CODE 'A' and VERSION_NO null. For committed: versions row by ENDT_NO and a risk with ACTION_CODE 'A' at the same VERSION_NO. Java loops rows and checks risk existence (`additionExists`). Follow-up: why `VERSION_NO is null` for WIP? WIP risks have no version yet. Mistake: reversing WIP and committed.

**Q16. What about ordering of the insured and premium landscape rows?**
A: Oracle orders by `cast(COV_SEQ_REF as decimal(15,2))`; Java adds `$toDouble` and sorts ascending. Nulls would sort differently (Mongo first, Oracle last). Follow-up: how would you test that? A handler test with a null COV_SEQ_REF row and assert order. Mistake: sorting as strings ("10" before "2").

**Q17. INST_IND — what is it and what does Java do?**
A: In the EndtSchedule SQL it is 'Y' when ISSUE_DATE is on/after the product's cut-off constant (`SAPM_SYS_CONSTANTS`, `CUT_OFF_RPT`, `<product>_ENDT`) and PMT_FREQUENCY is not 'AN'; the layout uses it to decide instalment text. In Java the handler fetches PMT_FREQUENCY but takes INST_IND from the base document (default "N"), so the cut-off rule is not evaluated — a known gap. Mistake: claiming it is implemented.

**Q18. What does SHOW_GST_LABEL mean?**
A: A flag that shows GST wording for documents issued before the SST rollout date (function `pg_uwge_utils.Fn_Get_SST_Rollout_Date`); otherwise SST wording. The master query reads it from UWGE_VI_ENDT_REPORT_PARAM; the EndtSchedule and DebitNote handlers cannot call the Oracle function, so they default or leave it unprojected. Follow-up: which service does fetch the rollout date? `PlineSSTRolloutDateClient` in the ServiceTax handler. Mistake: mixing GST and SST labels.

**Q19. How do you handle the `DebitNote` subreport not receiving pContractId?**
A: In the Jasper call the contract id is not passed; the query filters by ENDT_NO/REF_NO only, and the MST sub-selects use the main parameter. In Java we still AND CONTRACT_ID in Mongo as a selectivity aid (documented in Javadoc lines 64-70) since an endorsement number belongs to one contract. Follow-up: risk? None if endorsement numbers are unique across contracts; I assumed so. Mistake: removing the contract filter and causing a collection scan.

**Q20. Walk through ServiceTaxInvoice data flow in Java.**
A: Gate `showSt` (4 conditions) -> resolve mstContractId -> `AspServiceTaxInvoiceHandler.fetchServiceTaxInvoice` (aggregation on UWPL_VI_ENDT_TAX_INV, REST scalars: policy ref, SST date, company reg no; EINV_CUTOFF and VERSION_NO resolved up front; STAX_COUNT, QR_IND post-pipeline; MASTR_TIN/BRN/SST from UWPL_VI_TAX_INVOICE) -> if a row: STCompanyContact, banner, EndtMT/EndtMTSub when STAX_COUNT>1, QRSection when consent/valid/QR, CustomerDtls when PREM_BY_MST_IND=Y. Follow-up: where is TIN masked? Both in handler `maskedTinExpr` and master TIN logic (last 4 chars -> `****`). Mistake: forgetting the REST clients can return null (swallowed by safeCall).

**Q21. Why is there a copy of AspServiceTaxInvoiceHandler in the test folder?**
A: It appears to be an accidental duplicate introduced during Sonar clean-ups (commits 3f2157b28/24e50ab9b); same package and name as the production class, so the tests may be hitting the copy. I would delete it and re-run. Follow-up: how to verify? Print `AspServiceTaxInvoiceHandler.class.getProtectionDomain().getCodeSource()` in a test. Mistake: saying it is a harmless test helper.

**Q22. What did you do about XSD changes?**
A: Mapped each handler DTO field to an existing Pline.xsd leaf (transDet, endorsementDetails, prntDetails/compInfo/clientInfo, reportParameters) and avoided editing requestpayload classes when no leaf existed (e.g., delete-member `subcoverPrem`, endt* fees at transDet level). XSD validation test `PlineXsdValidationTest` checks generated XML. Follow-up: example of a leaf only at endorsement level? `endtStiPrem`, `endtLlbPrem` live under endorsementDetails (lines 607-617). Mistake: claiming mainReportName is in the repo XSD.

**Q23. What is `runToBottom` and why do the bands have 10px height?**
A: Subreports are placed in 10px bands with runToBottom so the subreport stretches over as many pages as needed; band printWhenExpression decides whether the subreport appears at all. Not relevant to Java but shows how I read the jrxml. Follow-up: what about Blank pages? Inserted to keep duplex alignment between sections. Mistake: describing runToBottom as a SQL feature.

**Q24. If production returns an empty cover/schedule for a checklist copy, where do you look first?**
A: The main handler anchor (`UWMT_VI_ENDT_TAX_INV_ALT_UPV` match on ENDT_NO + CONTRACT_ID) because checklist copies have no ENDT_NO; the shared base class dropped that match for this exact reason, ASP did not; branch commit `cc6a1b84d` fixes the V1 handler. Then WIP vs committed branch selection via checklistCopy. Mistake: starting with the XSD.

**Q25. How would you prove the Java output matches Jasper for one endorsement?**
A: Pick a 72/168 endorsement with sponsor and nominee, generate Jasper PDF and the Java response; compare gating (which subreports appear), row counts per subreport and key values (premiums, GST/SST amounts, debit/credit flag); log differences by section; automate as a golden-file test. I would explicitly list known gaps (stubs, INST_IND, ORA emulation). Follow-up: tooling? Mongo test data export plus the dump of Jasper output. Mistake: comparing only totals.

---------------------------------------------------------------------------------------------------

## 10. Uncertainties and things I could not confirm

- U1. "ASP" expansion: no text in the jrxml; only `ASPlus` internal names and product 070412 tie-in (section 5).
- U2. Which dispatcher resolves the bean key and whether callers still send `reportName` for landscape; I did not read the controller/dispatcher, so "only the EndtDocument key resolves" is inferred from the `@Service` annotations.
- U3. Whether the downstream renderer prints EndtScheduleOfBenefits_ASP / ASPLLB / ASPSTI from the master row's actionCode (72/168) since Java emits no row for them.
- U4. Whether `AspEndtDocumentMainHandler` anchor collection `UWMT_VI_ENDT_TAX_INV_ALT_UPV` has rows for all production endorsements. The base-class comment says it does not; the not-on-HEAD commit `cc6a1b84d` fixes it. I did not run anything against Mongo.
- U5. MongoDB behaviors I cite (`$ne` on missing field, null sort order, `$round` half-even) are standard documented behavior but are not tested in this repo.
- U6. `pSchemeName` is a Jasper EndtDocument parameter used in EndtSchedule layout (product 070504 schemes) but is not read by `AspEndtDocumentService`; I do not know if the renderer gets it from another place.
- U7. Date-format handoff to `PlineCommonCompanyNameWithGSTRegNoHandler`/`PlineCommonSignatureHandler` (Jasper passes dd-MMM-yyyy text to a `to_date(...,'dd/MM/yyyy')` function; Java passes yyyy-MM-dd); common handlers not opened.
- U8. Whether `PlineXsdValidationTest` includes an ASP sample; and whether the real v0.1.10 XSD (with mainReportName/subReportName) lives in another repo.
- U9. Mongo collection name `UWGE_VI_ENDT_DEBIT` (Java) vs Oracle view `CUSTOMER.UWPL_VI_ENDT_DEBIT` (jrxml): names differ in the middle token; I could not verify which one exists in Mongo (could be a deliberate migration naming).
- U10. Hierarchy csv files exist only for EndtDocument and EndtDocument2; the landscape trees come from my reading of `subreportExpression`, and the nested trees of EndtTaxDoc, TransitionalInvoice(2), CompanyContactJr, SSTNote, ComplaintRedress were taken from the csv, not re-verified in each jrxml.
- U11. The DebitNote and ServiceTaxInvoice layouts have many more product/amount-specific print conditions (ASST fee for non-089xxx, instalment lines, STRN invoice type switch, pType AO/AD commission). I summarised them; I did not verify field-by-field rendering rules, so do not claim full layout parity.
- U12. Git "not on HEAD" means "not an ancestor of HEAD"; a commit may still be content-equivalent via squash or cherry-pick (HEAD's `4cd1cad2e` "Working code for pline" probably consolidates some of them). I did not diff each.
- U13. The statement that the test-copy of AspServiceTaxInvoiceHandler shadows the main class depends on build tool classpath ordering (Maven/Gradle default puts test-classes first); not executed.
- U14. Test counts are annotation counts, not executed pass/fail results; I did not run the test suite.
