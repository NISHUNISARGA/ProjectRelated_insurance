# 02 - PLine Template12 Deep Dive (PolicyDocument, PolicyDocument2, Landscape)

Purpose: a knowledge-transfer reference so the developer can explain, in interviews, how the Jasper
Template12 reports were analysed and re-implemented in Java (Mongo-backed, XSD-shaped output), with
evidence. Everything below was read from the two source trees; anything that is my interpretation is
tagged **(inferred)**. No secrets (Mongo URI, passwords, truststore) are reproduced here.

Scope of the JIRA work this document supports: (i) analysis of Template12 PolicyDocument + all-level
subreports, PolicyDocument2 + all-level subreports, Landscape + subreports; (ii) code for all three;
(iii) unit tests; (iv) XSD changes.

---------------------------------------------------------------------------------------------------
## 0. Sources, naming and abbreviations

### 0.1 Source locations
| Item | Path |
|---|---|
| Java repo (module `opus-documentprintdata-service`, package `com.agcs.fo.opus.plines`) | `C:\Users\EPGRVFY\Documents\Plines_Base_code\New_branch\2\opus-documentprintdata-service` |
| Jasper dump, Template12 folder | `C:\Users\EPGRVFY\Documents\jasper_dumps\jasper_02072026\resources\reports\AGIC\UW\Plines\PolicySchedule\Template12` |
| Jasper dump, shared "CommonReport" folder (SUBREPORT_DIR default of most Template12 reports) | `...\AGIC\UW\Plines\PolicySchedule\CommonReport` |
| Jasper dump, shared "Generic" folder (SUBREPORT_CR / SUBREPORT_GEN / SUBREPORT_BRN default) | `...\AGIC\UW\Generic` |
| XSD in repo | `src/test/resources/xsd/plines/Pline.xsd` (+ `Common.xsd`), header says **v0.1.6** |

How the Jasper dump is laid out (important to read it correctly):
* `PolicyDocument.xml`, `PolicyDocument2.xml`, `Landscape.xml` are the JasperServer *report unit* descriptors (input controls, datasource `/datasources/AGICDEV`).
* The actual main jrxml of each is the `*.data` file under the `*_files` folder:
  * PolicyDocument -> `PolicyDocument_files/PolicyDocument_jrxml.data` (648 lines, JasperReports 6.20.3, report name `PolicyDocument_12`)
  * PolicyDocument2 -> `PolicyDocument2_files/PolicyDocument2.jrxml.data` (1240 lines, name `PolicyDocument_2`) + `PolicyDocument2_files/PolicyDocument2_Main_Report.sql` (the main query exported as a standalone .sql) + a compiled `PolicyDocument2.jasper`
  * Landscape -> `Landscape_files/Landscape_jrxml.data` (529 lines, name `Landscape`, 842x595 landscape page)
* Every subreport is `<Name>.jrxml.data` (the jrxml text) + `<Name>.jrxml.xml` (just resource metadata). In this document "line N" for a Jasper file means the line in the `.data` file.
* Folder has 187 files in total; all `.jrxml.data` files were scanned for `subreportExpression`, `printWhenExpression`, `queryString`, `field`, `parameter`.

### 0.2 Java classes (all under `src/main/java/com/agcs/fo/opus/plines/`)
| Class | Role | Evidence |
|---|---|---|
| `service/Template12PolicyDocumentService` | The ONLY registered Template12 report bean: `@Service("AGIC/UW/Plines/PolicySchedule/Template12/PolicyDocument")` (line 103). Its `execute()` (lines 130-556) runs PolicyDocument bands, then a merged PolicyDocument2 block, then a merged Landscape block. 2235 lines. | `Template12PolicyDocumentService.java:103,130-556` |
| `service/Template12PolicyDocumentBaseService` | Abstract "toolkit" base: about 95 `add*()` helpers (94 counted by grep) (one per Jasper subreport/band), envelope helpers (`mergeSingleTransDet`, `setEndorsementDetails`...), `buildTransactions`. 3732 lines. | `Template12PolicyDocumentBaseService.java` |
| `service/Template12PolicyDocument2Service` | **Unregistered** (`// @Service(...)` commented, line 42). Kept for reference. Old stand-alone PolicyDocument2 body. | `Template12PolicyDocument2Service.java:24-44` |
| `service/Template12LandscapeService` | **Unregistered** (`// @Service(...)` line 110). Old stand-alone Landscape body. | `Template12LandscapeService.java:100-112` |
| `documentHandler/PlinePolicyDocumentMainHandler#fetchTemplate12PolicyDocumentMain` | PolicyDocument's own primary ("select ... from dual") fetch (lines 1452-1518). | `PlinePolicyDocumentMainHandler.java:1452-1518` |
| `documentHandler/PlineT12PolicyDocumentHandler#fetchT12PolicyDocument` | PolicyDocument2's own primary fetch. | `PlineT12PolicyDocumentHandler.java:72` |
| `documentHandler/Template12LandscapeDispatchHandler#fetchDispatch` | Landscape's own primary (dispatch) fetch incl. `CI_LANDSCAPE_PRT` exclusion. | `Template12LandscapeDispatchHandler.java:52-258` |
| `documentHandler/PlineT12*Handler`, `PlineTemplate12*Handler`, `PlineCIPrint*Handler`, shared `PlineCommon*/PlineGeneric*` handlers | One handler per Jasper query. | see section 3 |
| `documentHandler/Template12PolicyDocumentHandler` | Small legacy helper (delegates to `fetchMainDocumentSchemeProductCovernoteWithExtras`); autowired only in `PlineXsdTemplate2BaseService:78`, **not** used by the Template12 service. | `Template12PolicyDocumentHandler.java:20-42` |
| `utils/PlineReportNames#tag` | Fills `reportData/mainReportName` from the `@Service` bean name (`Template12PolicyDocument`). | `PlineReportNames.java:21-60` |

Report-name resolution: `CommonController` looks the report name up in a `Map<String, ReportStrategy>` keyed by bean name
(`CommonController.java:171,203`). Because PolicyDocument2 and Landscape beans are un-annotated, only
`.../Template12/PolicyDocument` finds a Plines strategy. **(inferred)** A caller still asking for
`.../Template12/PolicyDocument2` or `.../Template12/Landscape` would no longer hit these Plines beans; I did not trace the
legacy fall-through steps in `CommonController`.

### 0.3 Abbreviations used in the tables
| Abbrev | Meaning |
|---|---|
| `S12` | `Template12PolicyDocumentService` |
| `B12` | `Template12PolicyDocumentBaseService` |
| `RD` | the one shared `PlineReportData` (header) created by `B12.newSharedReportData()` (`B12:166`) |
| `TD` | XSD path `transactions/transaction/transactionDetails/headerTransaction/reportTransaction/reportData/prntDetails/transDetList/transDet` (Pline.xsd lines 464-672) |
| `CD` | `.../reportData/prntDetails/transDetList/companyType/companyDesc` (Pline.xsd 1369, 441-459) |
| `PRN` | `.../reportData/prntDetails` (children `agentInfo`, `branchInfo`, `clientInfo`, `compInfo`, `taxClientInfo`...; Pline.xsd 652-666) |
| `ED` | `.../reportData/endorsementDetails` (Pline.xsd 593-648) |
| `RP` | `transactions/transaction/reportParameters` (Pline.xsd 475-547) |
| WIP / posted | Oracle `WIP_*` tables (work-in-progress, not yet issued) vs posted tables (`UW*`, `SB_*`). Mongo collections carry the same upper-case names as the Oracle tables/views (constants in the handlers). |
| "dispatch-only" | Subreport whose query is `SELECT 1 FROM DUAL` (static text report); Java emits an empty/merged `transDet` row only to mark that the band printed. |

### 0.4 The three Java design facts you must be able to state in one breath
1. **One bean, three reports.** Since commit `c0c6d449a` ("Added Template12 dynamic merging", 2026-08-03) `S12.execute()` serves PolicyDocument, PolicyDocument2 and Landscape in one response (`S12:437` calls `executeMergedPolicyDocument2`, `S12:445` calls `executeMergedLandscape`).
2. **One header `PlineReportData` + one `reportData` per Landscape row.** Everything except Landscape rows is appended to one shared `RD.prntDetails.transDetList`; each multi-row Landscape band row becomes its own `PlineReportData` via `toLandscapeReportData` (`S12:2077-2091`), and a code comment says the consumer side "has not been confirmed" to treat them as independent print units (`S12:159-165`).
3. **Three different primary fetches, deliberately not unified** (class javadoc "CRITICAL FACT #1", `S12:62-72`): PolicyDocument (`fetchTemplate12PolicyDocumentMain`, throws on WIP/posted conflict), PolicyDocument2 (`fetchT12PolicyDocument`, posted-first `$ifNull`), Landscape (`fetchDispatch`, its own `$ifNull` chain).

---------------------------------------------------------------------------------------------------
## 1. Report family map (main -> every subreport at every level)

Notation per node: `file (level)`, **print** = exact `printWhenExpression` (band-level unless marked "element"), **params** = subreport
parameters passed, **data** = SQL tables/views/functions, **key output** = main fields.
Level 0 = main report; Level 1 = called from the main; Level 2 = called from a Level-1 report; etc.
JasperReports rule used throughout: a band whose `printWhenExpression` is false is not rendered at all, so
for "band condition AND element condition" the subreport prints only when both are true.

### 1.1 PolicyDocument (Level 0) - `PolicyDocument_files/PolicyDocument_jrxml.data`

Level 0 data: one row from `dual` (query lines 45-74). Fields: `1`, `PRODUCT_CODE`, `SCHEME_NAME`, `DEPENDANT`, `ISSUE_DATE`, `SIGN_ISSUE_DATE`, `ASP_CUTOFF_IND` (lines 75-102).

| # | Band / node | Level | Subreport file | Exact print condition | Params passed (main -> sub) | Data source of the sub |
|---|---|---|---|---|---|---|
| P0a | background | 1 | `repo:/reports/AGIC/UW/Generic/PrintSeriesFooter.jrxml` (line 128) | element: `!"Y".equalsIgnoreCase($P{ShowOutPrintCovLetter})` (line 116) | `pVersionNo=$P{versionNo}`, `pContractId`, `pPrintSeries=$P{printSeries}` (118-127) | `UWGE_POLICY_VERSIONS_CTRL` (`select PRN_CNT ... where CONTRACT_ID and VERSION_NO`); output label = `pPrintSeries-PRN_CNT` |
| P0b | background | 1 | same PrintSeriesFooter (line 147) | element: `"Y".equalsIgnoreCase($P{ShowOutPrintCovLetter}) && $V{PAGE_NUMBER}!=1 && $V{PAGE_NUMBER}!=2` (141) | `pPrintSeries` only (143) | no `pContractId`/`pVersionNo` is passed, so its query has no inputs **(inferred: the second call can only render the label side, no PRN_CNT row)** |
| P1 | detail band 1 | 1 | `$P{SUBREPORT_DIR}+"OutSourcePrintCoverLetter.jrxml"` (217) -> `CommonReport/OutSourcePrintCoverLetter` | band: `"Y".equalsIgnoreCase($P{ShowOutPrintCovLetter})` (153) | 18 params incl. `pPolicyNo, printSeries, showScheduleStd, issueDate, pCertNoTo/From, pFirstPageNumber, SUBREPORT_DIR/2, pMstPolicyNo, showComplaint, companyParamType, versionNo, pContractId(List), showTaxInvoice, checklistCopy, showCoverNote, pType, showDebitNote` (156-215) | `UWGE_POLICY_BASES`, `CMGE_CODE`, `DUAL` (+ `uwge_policy_bases`); fields `POLICYNO, CONTRACT_ID, ACCOUNTNO, PRODUCT, INSUREDNAME, ADDRESS_LINE1-3, POSTCODE, PRODUCT_DESC, STATE, CITY, COUNTRY, SCHEME_NAME, COMPANY_REG_NO` |
| P2 | detail band 2 | 1 | `$F{PRODUCT_CODE}.equalsIgnoreCase("070406") ? Template12/CIPrint_ATC.jrxml : Template12/CIPrint.jrxml` (254) | band + element: `!($F{PRODUCT_CODE}.equalsIgnoreCase("070504") \|\| $F{PRODUCT_CODE}.equalsIgnoreCase("070414"))` (221,224) | `pMstPolicyNo, pPolicyNo, SUBREPORT_DIR2, companyParamType, pContractId, pContractIdList, issueDate=dd-MMM-yyyy(ISSUE_DATE), checklistCopy, pCertInd` (226-252) | see 1.1.A / 1.1.B |
| P3 | detail band 3 | 1 | 070504 variant chosen by `SCHEME_NAME` (330-336): `ASTRO`->`CIPrint_ASTRO`; `BMW`/`BMW1`->`CIPrint_BMW`; `POSCOVER`->`CIPrint_POS`; `AZP1`/`AZP2`->`CIPrint_AZP`; `POSMESRA`->`CIPrint_POSM`; `FWPA`->`CIPrint_FWPA`; else `CIPrint_Elite` | band + element: `$F{PRODUCT_CODE}.equalsIgnoreCase("070504")` (258,261) | as P2 plus `printSeries, showScheduleStd, pCertNoTo/From, pFirstPageNumber, SUBREPORT_DIR, showComplaint, versionNo, showTaxInvoice, showCoverNote, showDebitNote, pType, pCertInd, signIssueDate=dd-MMM-yyyy(SIGN_ISSUE_DATE)` (263-328) | see 1.1.C-1.1.I |
| P4 | detail band 4 | 1 | `$P{SUBREPORT_DIR2}+"CIPrint_PA4A.jrxml"` (411) | band + element: `$F{PRODUCT_CODE}.equalsIgnoreCase("070414")` (340,345) | as P3 w/o signIssueDate | see 1.1.J |
| P5 | detail band 5 | 1 | `$P{SUBREPORT_CR}+"ComplaintRedress.jrxml"` (488) -> `Generic/ComplaintRedress` | band: `$P{showComplaint}.equalsIgnoreCase("Y") && !("BMW".equalsIgnoreCase($F{SCHEME_NAME}) \|\| "BMW1".equalsIgnoreCase($F{SCHEME_NAME})) && !$F{PRODUCT_CODE}.equalsIgnoreCase("070504") && (!$F{PRODUCT_CODE}.equalsIgnoreCase("070412") \|\| ($F{PRODUCT_CODE}.equalsIgnoreCase("070412") && $F{ASP_CUTOFF_IND}.equalsIgnoreCase("N"))) && !$F{PRODUCT_CODE}.equalsIgnoreCase("070406")` (415-416); element: `$P{showComplaint}.equalsIgnoreCase("Y")` (419) | 20 params (421-486) | `Generic/ComplaintRedress`: `SELECT 1 FROM DUAL` + static text; nested pageHeader `CompanyNameBRNUpperAndLowercase` (`SAPM_COMPANY`) |
| P6 | detail band 6 | 1 | `$P{SUBREPORT_DIR2}+"ComplaintRedress_SCB.jrxml"` (565) | band: `$F{PRODUCT_CODE}.equalsIgnoreCase("070504") && !(BMW\|\|BMW1) && !("POSMESRA") && !("FWPA")` (492-493); element: `$P{showComplaint}.equalsIgnoreCase("Y")` (496) | 19 params (498-563) | `ComplaintRedress_SCB`: `SELECT 1 FROM DUAL`; Level 2: `CompanyRegNo` (pageHeader, `SUBREPORT_BRN`=Generic, line 64), `ComplaintRedress_NoMargin` (detail, line 86) |
| P7 | detail band 7 | 1 | `$P{SUBREPORT_DIR}+"Blank.jrxml"` (576) -> `CommonReport/Blank` | band: `$F{PRODUCT_CODE}.equalsIgnoreCase("070504") && $F{SCHEME_NAME}.equalsIgnoreCase("POSCOVER") && ($F{DEPENDANT}!=null && $F{DEPENDANT}.doubleValue()>0.00)` (569) | `pType` (572) | page-break filler (`field BLANK`) |
| P8 | detail band 8 | 1 | `$P{SUBREPORT_DIR2}+"ParticularOfInsuredLandscape_POS.jrxml"` (644) | band: same as P7 (580) | 18 params (583-642) incl. `pType` | `UWPL_VI_POLICY_SCHEDULE_TPL4`, `ORDER BY cast(COV_SEQ_REF as decimal(15,2))` |

Level-0 query (lines 45-74), summarised: `PRODUCT_CODE` = DISTINCT `product_config_code` of (`wip_uwge_policy_contracts` rownum=1 UNION ALL `sb_uwge_policy_contracts` rownum=1);
`SCHEME_NAME` = `wip_uwpl_policy_bases` (by contract) UNION `uwpl_policy_bases` (by contract AND `top_indicator='Y'`); `DEPENDANT` = `SELECT 1` from `UWPL_RISK_PERSON` / `WIP_UWPL_RISK_PERSON` where `INSURED_TYPE='D'` (UNION); `ISSUE_DATE`, `SIGN_ISSUE_DATE` from `UWGE_VI_POL_REPORT_PARAM`; `ASP_CUTOFF_IND` = 'Y' if `ISSUE_DATE >= SAPM_SYS_CONSTANTS(ASP_CONFIG_DATE/ASP_CUTOFF_DATE).DATE_VALUE` else 'N'.

#### 1.1.A `CIPrint.jrxml` (default variant; Level 1 from P2 when product is not 070406/070504/070414) - `CIPrint.jrxml.data` (1351 lines)
Main query (135-231): `UWGE_VI_POLICY_SCHEDULE_BASE UGE LEFT JOIN UWPL_VI_POLICY_SCHEDULE_BASE UPL` on `CONTRACT_ID`; computed columns: `CITY/STATE/COUNTRY` via `CMGE_CODE` lookups, `MST_POL_HOLDER` (join `OCP_POLICY_BASES`/`UWGE_POLICY_BASES`/`CPGE_VI_PARTNERS_ALL`), `COMPANY_REG_NO` = `PG_RPGE_UTILS.FN_GET_COMP_BRN_NUMBER(companyParamType, to_date(issueDate,'dd/MM/yyyy'))`, `PROV_IND` = count in `UWPL_VI_POLICY_SCHEDULE_PROV`, `SHOW_OCCP_DESC` (cutoff `OCCUP_CUT_OFF_DATE`), `ASP_CUTOFF_IND`, hard-coded `1 as benefit_print, 1 as premium_print`, `NOMINEE_PRINT` (exists in `wip_uwpl_risk_nominee`/`uwpl_risk_nominee` joined to policy versions), `SPONSOR_PRINT` (`wip_uwge_cover_sponsor`/`uwge_cover_sponsor`), `PIDM_PRINT` ('Y' if `EFF_DATE > SAPM_SYS_CONSTANTS(PIDM_CUTOFF_POL).DATE_VALUE`). Key output: policy header block (policy no, plan, insured, address, agent, premium block).

Level 2 nested subreports (all with `SUBREPORT_DIR2`=Template12, `SUBREPORT_DIR`=CommonReport, `SUBREPORT_CR`=Generic):
| Sub | Exact print condition | Data |
|---|---|---|
| `Generic/BNM_eMarketing` (924) | element: `PRODUCT_CONFIG_CODE=="100908" \|\| =="070412"` (917-918) | `UWGE_POLICY_BASES`/`WIP_UWGE_POLICY_BASES` -> `EMARKETING_CONSENT_IND` |
| `RiskDetails` (935) | band: `!070412 && !070521 && !100908` (928) | `UWPL_VI_POLICY_SCHEDULE_RISK` -> veh/insured fields |
| `Coverage` (955) + `Generic/PIDMSetBEnglish` (976) + `PIDMSetBBahasa` (997) + `BNM_eMarketing` (1009) | band: `!070412 && !100908 && !089426 && !089434 && !070302 && !070510 && !089432 && !089437` (940-945); PIDMSetB elements: `PIDM_PRINT=="Y" && product in {070202,070203,070204,070207,070233,090102,090202,070413}` (962-970, 983-991) | `UWPL_VI_POLICY_SCHEDULE_COVER` (`order by PLAN_CODE,SORT_SEQ`); `SAPM_SYS_CONSTANTS` for PIDMSetB |
| `Coverage` (1028) + `BNM_eMarketing` (1040) | band: `product in {089426,089434,070302,070510,089432,089437}` (1014-1018) | same |
| `InstallmentDetails` (1075) | band: `PRODUCT_CONFIG_CODE=="070521"` (1044) | `UWGE_VI_POLICY_INSTALLMENT ... UPPER(PMT_FREQUENCY) <> 'ANNUALLY' ORDER BY SEQ_NO` |
| `Note` (1087) | band: `!070521 && !100908 && (070412 && !ASP_CUTOFF_IND=="Y")` (1079-1080) | `OCP_POLICY_BASES`, `UWGE_VI_POLICY_SCHEDULE_BASE`, `UWPL_VI_POLICY_SCHEDULE_BASE` -> `MST_POL_HOLDER` |
| `ScheduleOfBenefits_ASP` (1101) | band: `070412` (1091) | static (`DUAL`); Level 3: `ASPLLB` (1957), `ASPSTI` (1967), `Generic/PIDMSetA` (1982, element `$P{PIDMPrint}=="Y"`) |
| `ScheduleOfBenefits_AMC` (1115) | band: `100908` (1105) | static; Level 3: `PIDMSetA` (2356, element `PIDMPrint=="Y"`) |
| `CIPerilsClauses` or `CommonReport/PerilsClauses` (1163) | band: `$F{PROV_IND}.intValue() > 0` (1119); element: `!070412 \|\| (070412 && ASP_CUTOFF_IND=="N")` (1122-1123); file chosen by `070521 \|\| 100908 ? CIPerilsClauses : PerilsClauses` (1162-1163) | `UWPL_VI_POLICY_SCHEDULE_PROV order by CLAUSE_CODE`; `PerilsClauses` Level 3 nests `Generic/PIDMSetA` with its own `PIDMPrint/pCertInd/pProduct` gate |
| `PerilsClauses_ASP` (1174) | same band as above; element: `070412 && ASP_CUTOFF_IND=="Y"` (1167-1168) | `UWPL_VI_POLICY_SCHEDULE_PROV` |
| `Generic/PIDMSetA` (1201) + `BNM_eMarketing` (1213) | band: `PROV_IND==0 && PIDM_PRINT=="Y" && pCertInd=="Y" && product in {070501,070503,105002,070510,089426,089429,089434,089437,070302,070504}` (1179-1191) | `SAPM_SYS_CONSTANTS` -> `DATE_VALUE` |
| `ParticularOfInsuredPotrait_ASP` (1226) | band: `BENEFIT_PRINT>0 && 070412 && ASP_CUTOFF_IND=="Y"` (1217); element `070412` (1220) | `UWPL_VI_POLICY_SCHEDULE_TPL4`, `ORDER BY cast(COV_SEQ_REF as decimal(15,2))` |
| `ParticularOfPremiumPotrait_ASP` (1239) | band: `PREMIUM_PRINT>0 && 070412 && Y` (1230) | `UWPL_VI_POLICY_SCHEDULE_PREM` |
| `ParticularOfSponsorPotrait_ASP` (1252) | band: `SPONSOR_PRINT>0 && 070412 && Y` (1243) | `UWPL_RISK_PERSON`/`WIP_UWPL_RISK_PERSON`, `CPGE_VI_PARTNERS_ALL` |
| `ParticularOfNominationPotrait_ASP` (1265) | band: `NOMINEE_PRINT>0 && 070412 && Y` (1256) | `UWPL_VI_POLICY_SCHEDULE_NOM` |
| `CommonReport/Blank_NoMargin` (1278) | band: `((($V{PAGE_NUMBER})-1)%2 != 0) && ASP_CUTOFF_IND=="Y" && 070412` (1269); element `(PAGE_NUMBER-1)%2 != 0` (1272) | static filler |
| `Generic/ComplaintRedress_NoMargin` (1291) | band: `ASP_CUTOFF_IND=="Y" && 070412` (1282); element: `showComplaint=="Y" && showScheduleStd=="Y"` (1285) | static |
| `ImportantNotes` (1347) | lastPageFooter band: `PRODUCT_CONFIG_CODE=="070521"` (1296) | static |

#### 1.1.B `CIPrint_ATC.jrxml` (Level 1 from P2 when product 070406) - `CIPrint_ATC.jrxml.data`
Query: same base views as CIPrint plus `CNOTE_NO`, `SIGN_ISSUE_DATE`, `UWGE_VI_POL_REPORT_PARAM`, `UWGE_POLICY_VERSIONS`. Nested: `Note` (850, band `PRODUCT_CONFIG_CODE=="070406"` at 841), `Generic/ComplaintRedress_NoMargin` (864, unconditional), `PerilsClauses_ATC` (876; `UWPL_VI_POLICY_SCHEDULE_PROV order by CLAUSE_CODE`), lastPageFooter: `CommonReport/Signature` (994, `SAPM_COMPANY`), `Generic/PIDMSetA` (1062, element `PIDM_PRINT=="Y"` at 1050), `Generic/BNM_eMarketing` (1073).

#### 1.1.C `CIPrint_ASTRO` (592-624): `Coverage` (592), `BNM_eMarketing` (604), `Note` (614), `CommonReport/PerilsClauses` (624, nests PIDMSetA). Query tables: `UWGE_VI_POLICY_SCHEDULE_BASE`, `UWPL_VI_POLICY_SCHEDULE_BASE`, `OCP_POLICY_BASES`; extra field `EXT_ACC_NO`.
#### 1.1.D `CIPrint_BMW` (653-712): per certificate row (`order by det.cert_no`, line 243; tables `UWGE_RISK`/`UWGE_RISK_DET`/`UWGE_RISK_VEH`/`UWPL_RISK_PERSON`/`UWGE_COVER`/`UWPL_COVER`/`CPGE_VI_PARTNERS_ALL`/`CPGE_VI_ADDRESS`/`CMGE_CODEREL_DET`; params `pCertNoFrom/pCertNoTo`): `Coverage_BMW` (+ nested `BNM_eMarketing` line 204), `Nomination_BMW` (`UWPL_RISK_NOMINEE`), `Note_BMW` (param `pMstPolHolder`), `CoverageBenefit_BMW` (`WIP_UWPL_POLICY_BASES`/`UWPL_POLICY_BASES`), `PerilsClauses_BMW` (`UWGE_RISK`, `UWGE_COVER_CLAUSE_DET`, `order by cch.CLAUSE_CODE`), `Generic/ComplaintRedress_NoMargin`. All unconditional bands.
#### 1.1.E `CIPrint_POS` (589-611): `Coverage_POS` (589, static), `BNM_eMarketing` (601), `PerilsClauses` (611).
#### 1.1.F `CIPrint_AZP` (708-720): `ScheduleOfBenefits_AZP` (`UWPL_COVER`/`WIP_UWPL_COVER`/`CMUW_SST`; sub-bands `PLAN>0`, `AZPA_PLAN>0`, `AZPB_PLAN>0`; nested `BNM_eMarketing` line 428), `PerilsClauses_AZP` (band `PROV_IND.intValue() > 0` at 713).
#### 1.1.G `CIPrint_POSM` (834-872): `Coverage_POSM` (calls function `PG_RPGE_UW_LISTING.FN_GET_BENEFIT_HDR`; nested `CoverageHeader_POSM` 107, `CoverageDetails_POSM` 137 = `FN_GET_BENEFIT_DET`), `PerilsClauses_POSM` (band `PRL_IND > 0` at 40; nested `PerilsClausesDetails_POSM` 60 - **file not present in the dump**, `SignatureFooter` 86, `BNM_eMarketing` 105), `ParticularOfInsuredPotrait_POSM` (band `$F{FAM_IND} != null` at 862; `UWPL_VI_POLICY_SCHEDULE_TPL4`).
#### 1.1.H `CIPrint_FWPA` (907-941): `Coverage_FWPA` (`UWPL_VI_POLICY_SCHEDULE_COVER`), `PerilsClauses_FWPA` (title: `PerilsClausesDetails_FWPA` 62 (`UWPL_VI_POLICY_SCHEDULE_PROV order by CLAUSE_CODE`); lastPageFooter: `SignatureFooter` 88 -> `CommonReport/Signature`, `BNM_eMarketing` 107), `ParticularOfInsuredPotrait_FWPA` (band `PRINT_POIP>1` at 934; `UWPL_VI_POLICY_SCHEDULE_TPL4`).
#### 1.1.I `CIPrint_Elite` (854-897): `Coverage` (854), `BNM_eMarketing` (866), `Note` (878, band `!070504` at 871 = dead in this context), `CommonReport/PerilsClauses` (897). Query adds `UWPL_VI_POLICY_SCHEDULE_RISK` (veh fields).
#### 1.1.J `CIPrint_PA4A` (988-1059): `ScheduleOfBenefits_PA4A` (band `!SHOW_NEW_SOB=="Y" && !SHOW_SOB_P3_IND=="Y"`, lines 975-977; nests PIDMSetA at 208), `Template23/ScheduleOfBenefitsNew` (1007; element `SHOW_NEW_SOB=="Y" && !SHOW_SOB_P3_IND=="Y"`), `Template23/ScheduleOfBenefitsNewP3` (1025: `SHOW_SOB_P3_IND=="Y"`), `BNM_eMarketing` (1037), `CIPerilsClauses_PA4A` (band `CLAUSE_PRINT>0`, 1041), `ParticularOfNomination_PA4A` (band `NOM_PRINT>0`, 1052; `UWPL_VI_POLICY_SCHEDULE_NOM`). Query tables: `UWGE_VI_POLICY_SCHEDULE_BASE`, `UWPL_VI_POLICY_SCHEDULE_NOM`/`_PROV`, `WIP_UWGE_RISK`/`UWGE_RISK`, `SAPM_SYS_CONSTANTS`; extra output `SHOW_NEW_SOB, SHOW_SOB_P3_IND, CHARITY_DESC, SHOW_PTV, STAX_PCT/AMT, FW_FEE, MCO_FEE, OPT_IMA_FEE`.

Leaf reports (no nested subreports): `Note`, `RiskDetails`, `Coverage*`, `Nomination_BMW`, `Note_BMW`, `CoverageBenefit_BMW`, `InstallmentDetails`, `ImportantNotes`, `ASPLLB`, `ASPSTI`, `ScheduleOfBenefits_AMC` (except PIDMSetA), all `ParticularOf*Landscape*`, `ParticularOf*Potrait*`, `PerilsClauses_*`, `CIPerilsClauses*`.

### 1.2 PolicyDocument2 (Level 0) - `PolicyDocument2_files/PolicyDocument2.jrxml.data`

Level 0 query (lines 59-170, identical in `PolicyDocument2_Main_Report.sql`, summarised): `PREM` (0 when scheme in BMW/BMW1 else 1), `PROMO_AMT`, `PRODUCT_CODE`, `TAX`, `SCHEME_NAME`, `JOURNEY_AREA` (from `UWPL_POLICY_JOURNEY` VERSION_NO=1 rownum=1), `SHOW_GST_INV_IND/X_SHOW_STAX_INV_IND/SHOW_TRAN_INV_IND/SHOW_GST_LABEL/SHOW_SST_LABEL/SHOW_PDPA_NOTICE_IND/ISSUE_DATE/SIGN_ISSUE_DATE` (all from `UWGE_VI_POL_REPORT_PARAM`), `EINV_CONSENT_IND` (`UWGE_POLICY_BASES` VERSION_NO=1 UNION `WIP_UWGE_POLICY_BASES`), `EINV_PRINT_IND` = `CUSTOMER.PG_RPGE_UTILS.FN_GET_EINVOICE_IND(contract, checklistCopy)`, `SHOW_STAX_INV_IND` = `PG_UWGE_UTILS.FN_SHOW_STAX_INV_IND(contract, checklistCopy='Y' ? 'WIP' : 'OCP')`. Fields `EINV_STATUS`, `EINV_CUTOFF`, `SHOW_EINV_IND` are declared (lines 195-201) but are **not** selected by the query (**inferred**: leftovers).
Note: `PolicyDocument2_Main_Report.sql` contains the same statement twice; the first copy is truncated/garbled (`... in ('BMW','BMW1') then 0 SELECT ...`, lines 1-4) and the second copy omits `sp.VERSION_NO=1` in the posted promo branch (differs from the jrxml at lines 64-72). Treat the jrxml as authoritative.

Level 1 bands (detail, 215-1239):
| # | Subreport (line of subreportExpression) | Exact band print condition | Element condition(s) | Notes / data |
|---|---|---|---|---|
| Q1 | `CommonReport/Blank` (228) | `((($V{PAGE_NUMBER} + $P{pFirstPageNumber}) - 1)%2 != 0) && ($P{showDebitNote}!=null && showDebitNote=="Y") && (showScheduleStd=="Y") && $F{PREM}>0) && !SCHEME_NAME=="SNATCHPA-M" && !=="SNATCHPA-A" && !=="POSMESRA"` (216) | `(PAGE_NUMBER + pFirstPageNumber - 1)%2 != 0` (219) | blank page so debit note starts on an odd page; params `checklistCopy`, `pType=copyType` |
| Q2 | `Template12/ComplaintRedress_SCB` (251) | `PRODUCT_CODE=="070504" && ("POSMESRA" \|\| "FWPA")` (233) | `showComplaint=="Y"` (236) | params `issueDate(dd-MMM-yyyy)`, `companyParamType`, `pSchemeName=$F{SCHEME_NAME}`, `pFirstPageNumber` (239-248). Level 2: `CompanyRegNo` (64), `ComplaintRedress_NoMargin` (86); element `ComplaintRedress_SCB@70`: `((PAGE_NUMBER + pFirstPageNumber) - 1)%2 != 0 && pSchemeName=="POSMESRA"` (POSMESRA blank filler) |
| Q3 | `Domestic_ATC` or `Overseas_ATC` (336): `"DOM".equalsIgnoreCase($F{JOURNEY_AREA}) ? Domestic_ATC : Overseas_ATC` | `PRODUCT_CODE.equalsIgnoreCase("070406")` (255) | same (258) | both are `SELECT 1 FROM DUAL` static text |
| Q4 | `Template12/PremiumDebitNote` (404) **or** `Template4/PremiumDebitNote` (482, SUBREPORT_DIR4) | band: `(showDebitNote != null && =="Y") && $F{PREM}>0 && !$P{copyType}.equalsIgnoreCase("BC")` (341) | T12: `... && PREM>0 && !PRODUCT_CODE=="070406"` (344); T4: `... && PRODUCT_CODE=="070406"` (410) | Level 2: `CommonReport/Signature` (PremiumDebitNote@623), `CustomerDtlsOfMasterPolicy` (PremiumDebitNote@1449, element `PREM_BY_MST_IND=="Y"` @1436). Data `UWPL_VI_PREMIUM_DEBIT`, `UWPL_VI_POLICY_SCHEDULE_BASE`, `OCP_POLICY_BASES`, `UWGE_POLICY_BASES`, `WIP_UWGE_POLICY_BASES`; fields include `PTV_AMT, PREM_DUE_AFT_PTV, PREM_DUE_ROUNDED_AFT_PTV, MCO_*_FEE, AGIC_FEE, FEE_SST, PREM_BY_MST_IND, MST_CONTRACT_ID` |
| Q5 | `CommonReport/Blank` (496) | `SCHEME_NAME=="POSMESRA" ? (((PAGE_NUMBER+pFirstPageNumber)-1)%2 != 0) && ((showTaxInvoice=="Y" && showScheduleStd=="Y") && TAX>0) : (showTaxInvoice=="Y" && showScheduleStd=="Y") && TAX>0` (486) | - | GST-era page break |
| Q6 | `Template12/TaxInvoince` (521) | `showTaxInvoice=="Y" && $F{TAX}>0 && "Y".equalsIgnoreCase($F{SHOW_GST_INV_IND})` (500) | `showTaxInvoice=="Y" && SHOW_GST_INV_IND=="Y"` (503) | Level 2: `CommonReport/CompanyContact` (TaxInvoince@158), `TaxInvoiceProductInfo` (@445); data `UWPL_VI_TAX_INVOICE` |
| Q7 | `CommonReport/Blank` (576) | **`false && showTaxInvoice=="Y" && SHOW_GST_INV_IND=="Y"`** (525) -> never prints | - | dead band |
| Q8 | `ServiceTaxInvoice` (635: `!"070414".equalsIgnoreCase(PRODUCT_CODE) ? Template12/ServiceTaxInvoice : Template12/ServiceTaxInvoice_PA4A`) and `Template4/ServiceTaxInvoice` (692) | band: `showTaxInvoice=="Y" && "Y".equalsIgnoreCase(SHOW_STAX_INV_IND) && EINV_PRINT_IND=="Y"` (581) | non-070406 element adds `!PRODUCT_CODE=="070406"` (584); T4 element adds `PRODUCT_CODE=="070406"` (641) | Level 2 of `ServiceTaxInvoice`: `STCompanyContact` (383), `STaxInvoiceProductInfo` (686), `Generic/ServiceTaxInvoicePL` (906, element `STAX_COUNT>1`), `ServiceTaxInvoicePLSub` (921, `STAX_COUNT>1`), `Generic/QRSection` (939, `EINV_CONSENT_IND=="Y" && EINV_STATUS=="VALID" && QR_IND=="Y"`), `STaxInvoiceProductInfoMST` (959, `PREM_BY_MST_IND=="Y"`), `CustomerDtlsOfMasterPolicy` (972, `PREM_BY_MST_IND=="Y"`). `ServiceTaxInvoice_PA4A` has the same but QR uses `QR_CNT > 0` (835 region) and no MST branch. Data `UWPL_VI_TAX_INVOICE`, `WIP_UWGE_POLICY_BASES_STAX/UWGE_POLICY_BASES_STAX`, `UWGE_GST_TAX_INV/WIP_UWGE_GST_TAX_INV`, `OCP_POLICY_BASES`, `UWGE_POLICY_BASES` |
| Q9 | `CommonReport/Blank` (750) | `showTaxInvoice=="Y" && "Y".equalsIgnoreCase(SHOW_STAX_INV_IND)` (696) | same (699) | SST-era page break |
| Q10 | `Template12/TransitionalInvoice` (817) | `showTaxInvoice=="Y" && SHOW_TRAN_INV_IND=="Y"` (754) | same (757) | Level 2: `CompanyContact` (165), `TransitionalInvoiceProductInfo` (429), `Generic/SSTNote` (684) |
| Q11 | `CommonReport/Blank` (875) | same as Q10 (821) | same | page break between the two transitional invoices |
| Q12 | `Template12/TransitionalInvoice2` (942) | same as Q10 (879) | same | Level 2: `STCompanyContact` (172), `STaxInvoiceProductInfo` (436), `SSTNote` (698) |
| Q13 | `Template12/PDS_Elite` (997) | **`false && PRODUCT_CODE=="070504" && SCHEME_NAME=="ELITESHIELD"`** (946) -> never prints | element: `070504 && ELITESHIELD` (949) | dead band; sub query `DUAL`, field `COMPANY_REG_NO` |
| Q14 | `Template12/PDS_Income` (1052) | **`false && 070504 && INCOMESHIELD`** (1001) -> never prints | element (1004) | dead band |
| Q15 | `CommonReport/PDPA` (1107) | **`false && PRODUCT_CODE=="070504"`** (1056) -> never prints | - | dead band (note: `SUBREPORT_DIR`=CommonReport, which has no `PDPA.jrxml`; the live PDPA is Q17 from `Generic`) |
| Q16 | `CommonReport/NominationForm` (1162) | **`false && PRODUCT_CODE=="070504"`** (1111) -> never prints | - | dead band |
| Q17 | `Generic/PDPA` (1236: `$P{SUBREPORT_PDPA}+"PDPA.jrxml"`) | `$F{SHOW_PDPA_NOTICE_IND}.equalsIgnoreCase("Y") && $P{checklistCopy}.equalsIgnoreCase("N") && ($P{copyType}.equalsIgnoreCase("IO") \|\| $P{copyType}.equalsIgnoreCase("ID"))` (1166) | - | `DUAL`, fn `COMPANY_REG_NO` |

Parameters of PolicyDocument2 not used for any visible band: `SUBREPORT_DIR3` (Template9 ParticularOfPremiumLandscape), `SUBREPORT_DIR5` (Template23).

**No band in the PolicyDocument2 jrxml (or in the compiled `.jasper`, string-searched) calls `ParticularOfInsuredTpl4`, `ParticularOfInsuredBase`, `NominationPA4A`, `NominationBMW` or `InstallmentDetails`.** The compiled file references only: Blank, ComplaintRedress_SCB, Domestic_ATC, Overseas_ATC, NominationForm, PDPA, PDS_Elite, PDS_Income, PremiumDebitNote, ServiceTaxInvoice(_PA4A), TaxInvoince, TransitionalInvoice(2). See section 4, D4.

### 1.3 Landscape (Level 0) - `Landscape_files/Landscape_jrxml.data`
Level 0 query (lines 28-107): outer `select * from ( ... from dual ) aa where not exists (select 1 from cmge_coderel_det where cat_code='GPA_SCHEME' and detail_code='CI_LANDSCAPE_PRT' and cd_value='Y' and header_code = aa.scheme_name)`. Inner columns: `BENEFIT_PRINT=1`, `PREMIUM_PRINT=1`, `NOMINEE_PRINT`/`DIAGNOSIS_PRINT` (exists in `wip_uwpl_risk_nominee`/`uwpl_risk_nominee` resp. `wip_uwpl_cover_diagnosis`/`uwpl_cover_diagnosis` joined to policy versions: WIP `version_no IS NULL`, posted `version_no=1`), `PRODUCT_CODE` (`wip_uwge_policy_contracts` UNION ALL `sb_uwge_policy_contracts`), `SCHEME_NAME` (`uwpl_policy_bases` VERSION_NO=1 UNION ALL `wip_uwpl_policy_bases` NOT EXISTS `OCP_POLICY_CONTRACTS`), `ISSUE_DATE` (`WIP_UWGE_POLICY_CONTRACTS` UNION `UWGE_POLICY_CONTRACTS`), `ASP_CUTOFF_IND`, `PROMO_AMT` (sum of `COV_PROMO_AMT`, join `wip_UWPL_RISK_PERSON sp, wip_uwge_cover upv` on contract_id only). Parameters: `pContractId`, `SUBREPORT_DIR`(Template12), `pFirstPageNumber` (Integer), `checklistCopy`, `issueDate` (Date), `SUBREPORT_DIR2` (Template4/Landscape_files). Background: page number "PAGE: n", watermark chosen by `checklistCopy` (checklist watermark vs normal), logo, frame lines.

Level 1 bands (detail; 16 bands, all `runToBottom=true`; params always `SUBREPORT_DIR, pContractId, checklistCopy, pFirstPageNumber`, plus `issueDate=$F{ISSUE_DATE}` for bands 1 and 4):
| Band | Subreport (line) | Exact band print condition | Element condition (if different) | Data (sub) |
|---|---|---|---|---|
| L1 | `ParticularOfInsuredLandscape_ASP` (187) | `PRODUCT_CODE=="070412" && ASP_CUTOFF_IND=="N"` (165-166) | `070412` (169) | `UWPL_VI_POLICY_SCHEDULE_TPL4`, `order by cast(COV_SEQ_REF as decimal(15,2))` |
| L2 | `ParticularOfPremiumLandscape_ASP` (210) | same as L1 (191-192) | `070412` | `UWPL_VI_POLICY_SCHEDULE_PREM`, same ordering; param `pProductCode` |
| L3 | `ParticularOfSponsorLandscape` (233) | same as L1 (214-215) | `070412` | `UWPL_RISK_PERSON`, `UWGE_POLICY_VERSIONS`, `WIP_UWGE_POLICY_VERSIONS`, `CPGE_VI_PARTNERS_ALL` |
| L4 | `ParticularOfBenefitsLandscape` (258) | `BENEFIT_PRINT>0 && !(070504 \|\| 070521 \|\| 070406)` (237) | `BENEFIT_PRINT>0 && !(070504 \|\| 070521 \|\| 100908)` (240) -> effective exclusion of 070504, 070521, 070406 and 100908 | `UWPL_VI_POLICY_SCHEDULE_BEN`, `SAPM_SYS_CONSTANTS` |
| L5 | `ParticularOfBenefitsLandscape_Elite` (280) | `070504 && BENEFIT_PRINT>0 && SCHEME_NAME=="ELITESHIELD"` (262) | same | `UWPL_VI_POLICY_SCHEDULE_TPL4` |
| L6 | `ParticularOfBenefitsLandscape_Income` (302) | `070504 && BENEFIT_PRINT>0 && "INCOMESHIELD"` (284) | same | `UWPL_VI_POLICY_SCHEDULE_TPL4` |
| L7 | `ParticularOfPremiumLandscape_Elite` (324) | `070504 && PREMIUM_PRINT>0 && "ELITESHIELD"` (306) | same | `UWPL_VI_POLICY_SCHEDULE_TPL4` |
| L8 | `ParticularOfPremiumLandscape_Income` (346) | `070504 && PREMIUM_PRINT>0 && "INCOMESHIELD"` (328) | same | `UWPL_VI_POLICY_SCHEDULE_TPL4` |
| L9 | `ParticularOfNominationLandscape_Elite` (368) | `070504 && NOMINEE_PRINT>0 && !(POSCOVER \|\| AZP1 \|\| AZP2)` (350) | same | `UWPL_VI_POLICY_SCHEDULE_NOM` |
| L10 | `ParticularOfNominationLandscape_POS` (390) | `070504 && NOMINEE_PRINT>0 && "POSCOVER"` (372) | same | `UWPL_VI_POLICY_SCHEDULE_NOM` |
| L11 | `ParticularOfNominationLandscape` (412) | `NOMINEE_PRINT>0 && !070504` (394) | same | `UWPL_VI_POLICY_SCHEDULE_NOM` |
| L12 | `ParticularOfDiagnosisLandscape` (434) | `DIAGNOSIS_PRINT>0 && !070504 && !100908` (416) | same | `UWPL_VI_POLICY_SCHEDULE_DIAG`, `order by cast(COV_SEQ_REF...)` |
| L13 | `ParticularOfInsuredLandscape_AMC` (456) | `100908` (438) | same | `UWPL_VI_POLICY_SCHEDULE_TPL20`, `order by cast(COV_SEQ_REF...)` |
| L14 | `ParticularOfDiagnosisLandscape_AMC` (478) | `DIAGNOSIS_PRINT>0 && 100908` (460) | same | `UWPL_VI_POLICY_SCHEDULE_DIAG` |
| L15 | `ParticularOfNominationLandscape_AZP` (501) | `070504 && NOMINEE_PRINT>0 && (AZP1 \|\| AZP2)` (483) | same | `UWPL_VI_POLICY_SCHEDULE_NOM` |
| L16 | `$P{SUBREPORT_DIR2} + "Landscape_jrxml"` (525) = Template4's own Landscape main report | `PRODUCT_CODE=="070406" && ($F{PROMO_AMT}!=null && $F{PROMO_AMT}.doubleValue()>0.00)` (506) | - | params `pContractId, pFirstPageNumber, checklistCopy, issueDate`. Template4 `Landscape_files/Landscape_jrxml.data` has its own ~12 bands (ParticularOfBenefitsLandscape(_ATC/_GPA), ParticularOfPremiumLandscape(_ATC/_ATC_Promo/_GPA), ParticularOfSponsor/Nomination/Companion/BoatLandscape ...) - that is Template4's scope. |

All Template12 `ParticularOf*Landscape*` leaf reports have only `pContractId` (+ `issueDate`/`pType`/`pProductCode` in a few) as parameters and no nested subreports.

### 1.4 Family members found in the Template12 folder but not reachable from the three mains (inferred orphans)
I searched every `subreportExpression` in the folder (and the three mains): no call to
`ComplaintsAndGrievances_BMW`, `ComplaintsAndGrievances_SCB`, `ParticularOfBenefits`, `ParticularOfNomination`,
`ParticularOfNominationLandscape_POSM`, `Domestic_ATC_NEW`, `Overseas_ATC_NEW`, `Coverage_BMW2`,
`ComplaintRedress_PA4A`, `PremiumDebitNote_PA4A` (and therefore `Signature2`, only used by it), and `Template12/Signature.jrxml`
(every `$P{SUBREPORT_DIR}+"Signature.jrxml"` resolves to `CommonReport/Signature`). Other folders were not searched, so these may be used elsewhere.
These are the "legacy / alternate" versions and explain why Java has handlers for some of them but never calls them (e.g. `PlineT12ParticularOfInsuredPosHandler`, section 4 D12).

---------------------------------------------------------------------------------------------------
## 2. Main reports: parameters, queries, fields, business decision

### 2.1 PolicyDocument (`PolicyDocument_12`)
**Parameters (lines 15-44)** - name : class
`pContractId:Integer`, `pPolicyNo:String`, `companyParamType`, `showCoverNote`, `showComplaint`, `showScheduleStd`, `showDebitNote`, `showTaxInvoice`, `SUBREPORT_DIR2` (default `repo:/.../Template12/`), `SUBREPORT_DIR` (default `.../CommonReport/`), `checklistCopy`, `pType`, `versionNo:Integer`, `printSeries`, `pFirstPageNumber:String`, `pMstPolicyNo`, `pContractIdList`, `pCertNoFrom`, `pCertNoTo`, `ShowOutPrintCovLetter`, `issueDate:String` (not for prompting), `SUBREPORT_CR` (default `repo:/reports/AGIC/UW/Generic/`), `pCertInd`, `signIssueDate:String`. (All `String` unless stated.) JasperServer input controls in `PolicyDocument.xml` also expose `pTemplate` (line 343) which is not a jrxml parameter.
**Query**: section 1.1 (dual row; product, scheme, dependants, dates, ASP cutoff).
**Business decision it drives**: *which policy-schedule layout prints for a Personal Accident certificate* - product-first (`070504` group schemes, `070414` PA4A, `070406` ATC travel, everything else default), then scheme-first inside 070504; plus whether the complaints/redress page and the POSCOVER dependants landscape page print.

### 2.2 PolicyDocument2 (`PolicyDocument_2`)
**Parameters (lines 15-60)**: `pContractId:Integer, pPolicyNo, companyParamType, showCoverNote, showComplaint, showScheduleStd, showDebitNote, showTaxInvoice, SUBREPORT_DIR2, SUBREPORT_DIR, pCertInd, SUBREPORT_DIR3, checklistCopy, pType, pFirstPageNumber:Integer, copyType, pShowGst:BigDecimal, pShowSst:BigDecimal, pShowTran:BigDecimal, pShowGstLabel, pShowSstLabel, SUBREPORT_PDPA (Generic), SUBREPORT_DIR4 (Template4), SUBREPORT_DIR5 (Template23), issueDate, signIssueDate`.
Input controls exposed in `PolicyDocument2.xml`: `pPolicyNo, companyParamType, showDebitNote, showTaxInvoice, checklistCopy, pType, pFirstPageNumber, pContractId, pShowPDPANotice, copyType, showPaymentInfo, showEinvEmail` (the last three and `pShowPDPANotice` are not jrxml parameters; `showComplaint`/`showScheduleStd` are jrxml parameters without an input control).
**Query**: `PolicyDocument2_Main_Report.sql` / jrxml 59-170 (section 1.2).
**Business decision**: *which accounting documents are appended after the schedule*: debit note (and which template: T12 vs T4 for ATC), GST tax invoice, SST service tax invoice (with e-invoice QR), transitional invoices, PDPA notice, and the ATC (070406) domestic/overseas terms page; page-break blanks keep each document starting on an odd (front) page.

### 2.3 Landscape (`Landscape`)
**Parameters**: section 1.3. **Query**: section 1.3.
**Business decision**: *which landscape (wide) particulars pages print* - insured, premium, benefits, nominee, diagnosis, sponsor - by product/scheme, subject to the `CI_LANDSCAPE_PRT` code-relation switch: if the scheme is flagged `GPA_SCHEME / CI_LANDSCAPE_PRT = Y` in `CMGE_CODEREL_DET`, the whole report returns no row and nothing prints (the schedule is carried in the main CI print only).

---------------------------------------------------------------------------------------------------
## 3. TRACEABILITY TABLE (Jasper subreport -> condition -> Java -> Mongo -> payload -> XSD)

How to read:
* **Java** = `class.method:line`. `S12.execute` rows give the *call site that implements the print condition*; `B12.addXxx` is the method that fetches and maps.
* **Mongo** = collection names exactly as constants in the handler class. "REST" = the Oracle function is not run in Mongo; it is called through `PlineUtilsClient` (HTTP) - `getCompBrnNumber`, `fnGetEinvoicePrintPolInd`, `fnShowStaxInvInd`, `fnRppmGetSignatureAndRemarks`, `fnGetBenefitHdr/Det`. "none" = static handler (equivalent of `SELECT 1 FROM DUAL`, no DB call).
* **Payload** = Java object/field written (`PlineTransDet` = one `transDet` row). **XSD** = path suffix after the prefixes in section 0.3. Leaf names are the exact Pline.xsd element names.
* **Status**: OK = implemented as per condition; PARTIAL; NOT IMPLEMENTED; MERGED (done inside another method); STUB (dispatch-only row).

### 3.1 PolicyDocument - Level 1

| Jasper subreport | Print condition (exact) | Java (call site -> method) | Mongo / REST | Payload target | XSD path | Status |
|---|---|---|---|---|---|---|
| `Generic/PrintSeriesFooter` (call 1) | `!"Y".equalsIgnoreCase($P{ShowOutPrintCovLetter})` | `S12.execute:238-239` -> `B12.addTemplate12PDPrintSeriesFooter:2890` -> `PlineGenericPrintSeriesFooterHandler.fetchPrnCnt` | `UWGE_POLICY_VERSIONS_CTRL` | `det.prnCnt` (merged into existing transDet via `mergeSingleTransDet`) | `TD/prnCnt` | OK. The request `printSeries` is only echoed to `RP/pPrintSeries`; the label `printSeries-PRN_CNT` (Jasper textField at PrintSeriesFooter line 25) is left to the renderer |
| `Generic/PrintSeriesFooter` (call 2) | `"Y".equalsIgnoreCase($P{ShowOutPrintCovLetter}) && $V{PAGE_NUMBER}!=1 && $V{PAGE_NUMBER}!=2` | none | - | - | - | **NOT IMPLEMENTED** (page-number based; comment `S12:235-237`) |
| `CommonReport/OutSourcePrintCoverLetter` | `"Y".equalsIgnoreCase($P{ShowOutPrintCovLetter})` | `S12.execute:241-242` -> `B12.addTemplate12PDOutSourcePrintCoverLetter:2901` -> `PlineCommonOutSourcePrintCoverLetterHandler.fetchOutSourcePrintCoverLetter` | `UWGE_POLICY_BASES, OCP_POLICY_BASES, UWGE_POLICY_CONTRACTS, UWPL_POLICY_BASES, CMGE_CODE` | `det.policyNo, contractId, insuredName, schemeName, productDesc`; `clientInfo.address.address1-3/postCode/city/state/country`; `agentInfo.agentCode`=ACCOUNTNO; `compInfo.compBRN`; `endorsementDetails.contractId/product` | `TD/{policyNo,contractId,insuredName,schemeName,productDesc}`; `PRN/clientInfo/address/*`; `PRN/agentInfo/agentCode`; `PRN/compInfo/compBRN`; `ED/{contractId,product}` | OK |
| `Template12/CIPrint` (default) | `!(PRODUCT=070504 \|\| 070414)` and product != 070406 | `S12.execute:247-279` -> `B12.addTemplate12CIPrintPrimary:1360` -> `PlineCIPrintPrimaryHandler.fetchCIPrintPrimary` | `UWGE_VI_POLICY_SCHEDULE_BASE, UWPL_VI_POLICY_SCHEDULE_BASE, UWPL_VI_POLICY_SCHEDULE_PROV, WIP_UWPL_RISK_NOMINEE, UWPL_RISK_NOMINEE, WIP_UWGE_COVER_SPONSOR, UWGE_COVER_SPONSOR, WIP_UWGE_POLICY_VERSIONS, UWGE_POLICY_VERSIONS, UWGE_VI_POL_REPORT_PARAM, SAPM_SYS_CONSTANTS, OCP_POLICY_BASES, UWGE_POLICY_BASES, CMGE_CODE` + REST `getCompBrnNumber` | `PlineTransDet` (contractId, policyNo, plan, occupDesc, idType, idValue1, loading, discount, assistanceFee, carReplFee, optCarReplFee, agentCode/Name, mstPolNo/Holder, schemeName, designation, phonenoHp, email, prevPolNo, showOccpDesc, aspCutoffInd, sponsorPrint, pidmPrint, grossPrem, basicPrem, rebateAmt, gstAmt, stampAmt, premDue, polsi, totprem, issueby, issuedate, effDate, expDate, insuredName, loanAcNo); `riskList.risk.coverList.cover.repCoverNote`; `sobList.benefits.benefit.benefitPrint`; `nomineeList.nominee.nomineePrint`; `clientInfo.address`, `branchInfo.branchName`, `compInfo.compBRN`, `taxClientInfo.productConfigCode`, `agentInfo.*`; `endorsementDetails.refno/source/proposalRecvDate/partnerType/contractId/product`; `reportParameters.provInd/premiumPrint` | `TD/*` same leaf names; `TD/riskList/risk/coverList/cover/repCoverNote`; `TD/sobList/benefits/benefit/benefitPrint`; `TD/nomineeList/nominee/nomineePrint`; `PRN/clientInfo/address/*`; `PRN/branchInfo/branchName`; `PRN/compInfo/compBRN`; `PRN/taxClientInfo/productConfigCode`; `PRN/agentInfo/*`; `ED/{refno,source,proposalRecvDate,partnerType,contractId,product}`; `RP/{provInd,premiumPrint}` | OK |
| `Template12/CIPrint_ATC` | band `!(504\|\|414)` and `PRODUCT_CODE=="070406"` (file chosen by ternary at PolicyDocument line 254) | `S12.execute:248-257` -> `B12.addTemplate12CIPrintAtc:1435` -> `PlineCIPrintAtcHandler.fetchTemplate12CIPrintAtc` | as CIPrint | as CIPrint plus `det.cnoteNo`, `det.signIssueDate`; `RP/provInd, premiumPrint` | as CIPrint plus `TD/{cnoteNo,signIssueDate}` | OK |
| `CIPrint_ASTRO` | `PRODUCT_CODE=="070504"` and `SCHEME_NAME=="ASTRO"` | `S12.execute:285-290` -> `B12.addTemplate12CIPrintAstroPos:1516` -> `PlineCIPrintAstroPosHandler.fetchCIPrintAstroPos` | `UWGE_VI_POLICY_SCHEDULE_BASE, UWPL_VI_POLICY_SCHEDULE_BASE, OCP_POLICY_BASES, UWGE_POLICY_BASES, SAPM_SYS_CONSTANTS, CMGE_CODE` | as CIPrint + `det.extAccNo` | `TD/extAccNo` + CIPrint leaves | OK |
| `CIPrint_BMW` | `070504` and `SCHEME_NAME in (BMW, BMW1)` | `S12.execute:300-307` -> `B12.addTemplate12CIPrintBmwPrimary:2456` (list; loop calls `addCIPrintBmwNestedSubreports:1758` per row) -> `PlineCIPrintBmwPrimaryHandler` | `UWGE_RISK_DET, UWGE_POLICY_BASES, UWPL_POLICY_BASES, UWGE_RISK_VEH, UWPL_RISK_PERSON, UWGE_POLICY_VERSIONS, UWGE_COVER, UWPL_COVER, UWGE_POLICY_CONTRACTS, OCP_POLICY_BASES, UWGE_RISK, CMGE_CODE, CMGE_CODEREL_DET` | one `transDet` per certificate row (`vehNo`, `vehModel`, `vehmodeldesc`, plan code... via `addSingleTransDet`) | `TD/{vehNo,vehModel,vehmodeldesc,...}` | OK (per-row loop) |
| `CIPrint_POS` | `070504` and `POSCOVER` | `S12.execute:314-319` -> `B12.addTemplate12CIPrintAstroPos` (shared with ASTRO) | as ASTRO | as ASTRO | as ASTRO | OK (shared handler) |
| `CIPrint_AZP` | `070504` and `AZP1`/`AZP2` | `S12.execute:324-329` -> `B12.addTemplate12CIPrintAzp:1587` -> `PlineCIPrintAzpHandler.fetchCIPrintAzp` | `UWGE_VI_POLICY_SCHEDULE_BASE, UWPL_VI_POLICY_SCHEDULE_PROV, OCP_POLICY_BASES, UWGE_POLICY_BASES, SAPM_SYS_CONSTANTS, CMGE_CODE` | CIPrint-style leaves (AZP DTO); `RP/provInd` | as CIPrint | OK |
| `CIPrint_POSM` | `070504` and `POSMESRA` | `S12.execute:340-349` -> `B12.addTemplate12CIPrintPosm:1660` -> `PlineCIPrintPosmPrimaryHandler` | `UWGE_VI_POLICY_SCHEDULE_BASE, UWGE_COVER, UWPL_COVER, WIP_UWGE_COVER, WIP_UWPL_COVER, OCP_POLICY_BASES, UWGE_POLICY_BASES, CMGE_CODE` | CIPrint-style leaves + `famInd`; `RP/polRef` | `TD/famInd`; `RP/polRef` | OK |
| `CIPrint_FWPA` | `070504` and `FWPA` | `S12.execute:356-365` -> `B12.addTemplate12CIPrintFwpa:2206` (uses `fetchCIPrintFwpaResilient:1745`, a Mongo-resilient copy kept inside the base class) | `UWGE_VI_POLICY_SCHEDULE_BASE, UWPL_VI_POLICY_SCHEDULE_PROV, UWPL_VI_POLICY_SCHEDULE_TPL4, WIP_/UWPL_RISK_NOMINEE, WIP_/UWGE_COVER_SPONSOR, WIP_/UWGE_POLICY_VERSIONS, UWGE_VI_POL_REPORT_PARAM, SAPM_SYS_CONSTANTS, OCP_POLICY_BASES, UWGE_POLICY_BASES, CMGE_CODE` | as CIPrint; `RP/{provInd,premiumPrint,printPoip}` | as CIPrint; `RP/printPoip` | OK |
| `CIPrint_Elite` | `070504` and scheme not in {ASTRO,BMW,BMW1,POSCOVER,AZP1,AZP2,POSMESRA,FWPA} | `S12.execute:374-381` -> `B12.addTemplate12CIPrintElite:2283` -> `PlineCIPrintElitePrimaryHandler` | `UWGE_VI_POLICY_SCHEDULE_BASE, UWPL_VI_POLICY_SCHEDULE_RISK, SAPM_SYS_CONSTANTS, OCP_POLICY_BASES, UWGE_POLICY_BASES, CMGE_CODE` | CIPrint-style + veh fields | as CIPrint | OK |
| `CIPrint_PA4A` | `PRODUCT_CODE=="070414"` | `S12.execute:393-404` -> `B12.addTemplate12CIPrintPa4a:2357` (list; one `transDet` per row via `addSingleTransDet`) -> `PlineCIPrintPa4aPrimaryHandler` | `UWGE_VI_POLICY_SCHEDULE_BASE, UWPL_VI_POLICY_SCHEDULE_NOM, UWPL_VI_POLICY_SCHEDULE_PROV, WIP_UWGE_RISK, UWGE_RISK, SAPM_SYS_CONSTANTS, OCP_POLICY_BASES, UWGE_POLICY_BASES, CMGE_CODE` | `det.{optImaFee,mcoFee,fwcovtype,fwFee,fwPolSi,staxPct,staxAmt,showSstPct,stReviseFeeInd,charityDesc,nomPrint,voucherNo,showPtv,showNewSob,showSobP3Ind,pidmPrint,...}`; `ED.showGstLabel` | `TD/*`; `ED/showGstLabel` | OK |
| `Generic/ComplaintRedress` (+ nested `CompanyNameBRNUpperAndLowercase`) | band P5 (section 1.1) | `S12.execute:413-419` computes `showComplaintRedress` = `showComplaint=="Y" && !isBmwScheme && !is070504 && (!is070412 \|\| (is070412 && aspCutoffInd=="N")) && !is070406` -> `S12.addComplaintRedressWithCompanyNameBRN:1930` | `PlineGenericComplaintRedressHandler` = none; `PlineGenericCompanyNameBRNUpperAndLowercaseHandler` -> `SAPM_COMPANY` | empty `PlineTransDet` row (dispatch); `companyType.companyDesc.companyName`; `compInfo.compBRN` | `TD` (empty row); `CD/companyName`; `PRN/compInfo/compBRN` | OK (STUB for the static page) |
| `Template12/ComplaintRedress_SCB` (+ `CompanyRegNo`, `ComplaintRedress_NoMargin`) | `070504 && !BMW && !POSMESRA && !FWPA` and `showComplaint=="Y"` | `S12.execute:415-416,420-422` -> `S12.addComplaintRedressScbWithSubreports:1961` -> `B12.addTemplate12ComplaintRedressScb:3546` + `addTemplate12CompanyRegNo:3535` + `addTemplate12ComplaintRedressNoMargin:3177` | none / REST `getCompBrnNumber` / none | dispatch rows; `compInfo.compBRN` | `PRN/compInfo/compBRN` | OK (STUB + compBRN) |
| `CommonReport/Blank` (POSCOVER) | `070504 && POSCOVER && DEPENDANT>0` | `S12.execute:425-426` -> `B12.addBlank:544` -> `PlineCommonBlankHandler.fetchBlank` | none | `det.blank` | `TD/blank` | OK (`isPosCoverWithDependants`, `S12:186-187`) |
| `Template12/ParticularOfInsuredLandscape_POS` | same band | `S12.execute:427-431` -> `S12.addTemplate12ParticularOfInsuredLandscapePos:1980` -> `PlineParticularOfInsuredHandler.fetchParticularOfInsuredLandscapePos` | `UWPL_VI_POLICY_SCHEDULE_TPL4` (sorted by `$toDouble(COV_SEQ_REF)`, `PlineParticularOfInsuredHandler:145-146`) | one `PlineReportData` per row via `toLandscapeReportData`; `det.*` (`S12:1994-2051`), `riskList.risk.coverList.cover.covSeqRef`, `risk.riskParentId`, benefit leaves; first row gives `parentName/planDescp/psi` | `TD/*`; `TD/riskList/risk/{riskParentId,coverList/cover/covSeqRef}`; `TD/sobList/benefits/benefit/{weeklyBenefit,casualtyCashBenefit,dailyCareCashBenefit,hospitalReliefCashBenefit,snatchTheftBenefit}`; `RP/{parentName,planDescp,psi}` | OK |

### 3.2 PolicyDocument - nested levels (Level 2/3, inside CIPrint variants)

Default `CIPrint` nested calls are all in `S12.addCIPrintDefaultNestedSubreports:1507-1690` (arguments carry the CIPrint row's `PROV_IND, PIDM_PRINT, NOMINEE_PRINT, SPONSOR_PRINT...`).

| Jasper nested subreport | Condition (exact; section 1.1.A) | Java | Mongo / REST | Payload / XSD | Status |
|---|---|---|---|---|---|
| `BNM_eMarketing` | `100908 \|\| 070412` | `S12:1538-1540` -> `B12.addGenericBNMeMarketingConsent:2783` -> `PlineGenericBNMeMarketingHandler.fetchBNMeMarketing` | `UWGE_POLICY_BASES, WIP_UWGE_POLICY_BASES` | `det.eMarketingConsentInd` -> `TD/eMarketingConsentInd` | OK |
| `RiskDetails` | `!070412 && !070521 && !100908` | `S12:1543-1546` -> `B12.addTemplate12RiskDetails:2709` -> `PlineTemplate12RiskDetailsHandler` | `UWPL_VI_POLICY_SCHEDULE_RISK` | `det.{idType,idValue1,insuredName,vehNo,vehSeatCap,vehChassis,vehModel,...}`, `risk.coverList.cover.covSeqRef`; `ED.{refno,source,contractId}`; parentName returned | `TD/{vehNo,vehSeatCap,vehChassis,vehModel,vehmodeldesc,...}`; `TD/riskList/risk/coverList/cover/covSeqRef` | OK |
| `Coverage` | `!070412 && !100908 && !089426 && !089434 && !070302 && !070510 && !089432 && !089437` | `S12:1555-1563` -> `B12.addTemplate12Coverage:2505` -> `PlineTemplate12CoverageHandler.fetchCoverage` | `UWPL_VI_POLICY_SCHEDULE_COVER` (two-key `$sort`: plan code then numeric sort seq, `PlineTemplate12CoverageHandler:75-76`) | one transDet per benefit row: `plancode, contractId, benefitCode, benefitDescp, benefitSi, benNarr, sortSeq`; `ED.source/contractId` | `TD/{plancode,contractId,benefitCode,benefitDescp,benefitSi,benNarr,sortSeq}`; `ED/{source,contractId}` | OK |
| `Coverage` (group A products) | `089426 \|\| 089434 \|\| 070302 \|\| 070510 \|\| 089432 \|\| 089437` | `S12:1566-1569` (same methods) | same | same | OK |
| `PIDMSetBEnglish` / `PIDMSetBBahasa` | `PIDM_PRINT=="Y" && product in {070202,070203,070204,070207,070233,090102,090202,070413}` | `S12:1557-1561` -> `B12.addGenericPIDMSetB:3120` (called twice; "English + Bahasa share the same underlying query") -> `PlineGenericPIDMSetBHandler.fetchPIDMSetB` | `SAPM_SYS_CONSTANTS` | dispatch merged row; returns `DATE_VALUE` -> `RP/dateValue` | `RP/dateValue` | OK |
| `InstallmentDetails` | `070521` | `S12:1572-1574` -> `B12.addTemplate12InstallmentDetails:2683` -> `PlineTemplate12InstallmentDetailsHandler` | `UWGE_VI_POLICY_INSTALLMENT` (`ANNUALLY` excluded by regex `^(?i)ANNUALLY$`, `$sort SEQ_NO`; handler lines 50-51) | `det.{contractId,seqNo,staxAmt,stampAmt,instGrossPrem,balAmtPayable,amtPayable,grossPrem,pmtFrequency,totalAmtPay}`; `RP/pmtDueDate` | `TD/{seqNo,instGrossPrem,balAmtPayable,amtPayable,...}`; `RP/pmtDueDate` | OK |
| `Note` | `!070521 && !100908 && 070412 && !ASP_CUTOFF_IND=="Y"` | `S12:1577-1579` -> `B12.addTemplate12Note:2929` -> `PlineTemplate12NoteHandler.fetchNote` | `UWGE_VI_POLICY_SCHEDULE_BASE, OCP_POLICY_BASES, UWGE_POLICY_BASES` | `det.mstPolHolder` | `TD/mstPolHolder` | OK |
| `ScheduleOfBenefits_ASP` (+`ASPLLB`,`ASPSTI`,`PIDMSetA`) | `070412` (PIDMSetA only if `PIDMPrint=="Y"`) | `S12:1582-1585` -> `B12.addTemplate12EndtScheduleOfBenefitsAsp:3131` (+ `addTemplate12AspLlb:3146`, `addTemplate12AspSti:3155`, `addGenericPIDMSetACutoffDate:2803`) | none / `SAPM_SYS_CONSTANTS` | dispatch merged rows; `RP/dateValue` | `TD` (merged), `RP/dateValue` | STUB + dateValue |
| `ScheduleOfBenefits_AMC` | `100908` | `S12:1588-1591` -> `B12.addTemplate12EndtScheduleOfBenefitsAmc:3164` | none / `SAPM_SYS_CONSTANTS` | dispatch; `RP/dateValue` | `RP/dateValue` | STUB + dateValue |
| `CIPerilsClauses` / `CommonReport/PerilsClauses` (+ nested `PIDMSetA`) | band `PROV_IND>0`; element `!070412 \|\| (070412 && ASP_CUTOFF_IND=="N")`; `070521\|\|100908` -> `CIPerilsClauses` else `PerilsClauses` | `S12:1611-1618` -> `B12.addPerilsClausesVariant:2525` (CIPerilsClauses) or `S12.addPerilsClausesWithPidmSetA:1905` (PerilsClauses) -> `PlinePerilsClausesVariantHandler.fetchPerilsClauses` | `UWPL_VI_POLICY_SCHEDULE_PROV` (`$sort CLAUSE_CODE`, handler line 39) | ONE shared transDet; each clause -> `provisoList.proviso{provisoCode,provisoDesc,provisoNarr}`; `det.{contractId,policyref,narr}`; `ED.{source,clauseCode,descp,contractId}` | `TD/provisoList/proviso/{provisoCode,provisoDesc,provisoNarr}`; `TD/{contractId,policyref,narr}`; `ED/{clauseCode,descp,source,contractId}` | OK. (Source comments still say "memo"; the code writes `proviso`, `B12:2529-2543`) |
| `PerilsClauses_ASP` | band `PROV_IND>0`, element `070412 && ASP_CUTOFF_IND=="Y"` | `S12:1620-1622` -> `B12.addPerilsClausesVariant` | `UWPL_VI_POLICY_SCHEDULE_PROV` | as above | as above | OK (shared query) |
| `Generic/PIDMSetA` (+ `BNM_eMarketing`) | `PROV_IND==0 && PIDM_PRINT=="Y" && pCertInd=="Y" && product in {070501,070503,105002,070510,089426,089429,089434,089437,070302,070504}` | `S12:1595-1631` -> `B12.addGenericPIDMSetACutoffDate:2803`; `addGenericBNMeMarketingConsent` | `SAPM_SYS_CONSTANTS` | `RP/dateValue` | `RP/dateValue` | OK |
| `ParticularOfInsuredPotrait_ASP` | `BENEFIT_PRINT>0 && 070412 && ASP_CUTOFF_IND=="Y"` (+ nested `070412`) | `S12:1635-1643` -> `B12.addParticularOfInsuredPotraitAsp:2942` -> `PlineParticularOfInsuredPotraitAspHandler` | `UWPL_VI_POLICY_SCHEDULE_TPL4, CMGE_CODE, SAPM_SYS_CONSTANTS` (sort `_covSeqRefNum`, line 78) | one transDet per insured (`policyNo, subsidiaryName, department, idType, idValue1, age, dateOfBirth, insuredName, insuredType, occupDesc, plancode, joinDate, relationship, ad, pd, pdRb, rbAmt, personalLiability, medicalExpenses, hospitalIncome, bereavement, bloodTransfusion, infertility, designation, occupClass, stiPlanCode, stiSi, llbPlanCode, llbSi, showOccpDesc, productcode`), `sobList...weeklyBenefit`; `RP/{parentName,planDescp,psi}` | `TD/*`; `TD/sobList/benefits/benefit/weeklyBenefit`; `RP/{parentName,planDescp,psi}` | OK |
| `ParticularOfPremiumPotrait_ASP` | `PREMIUM_PRINT>0 && 070412 && Y` | `S12:1646-1653` -> `B12.addParticularOfPremiumPotraitAsp:3000` -> `PlineParticularOfPremiumPotraitAspHandler` | `UWPL_VI_POLICY_SCHEDULE_PREM, CMGE_CODE` | premium leaves + cover leaves | `TD/*`; `TD/coverList/cover/*` | OK |
| `ParticularOfSponsorPotrait_ASP` | `SPONSOR_PRINT>0 && 070412 && Y` | `S12:1656-1658` -> `B12.addParticularOfSponsorPotraitAsp:3066` -> `PlineParticularOfSponsorPotraitAspHandler` | `UWPL_RISK_PERSON, WIP_UWPL_RISK_PERSON, UWGE_COVER, WIP_UWGE_COVER, UWGE_COVER_SPONSOR, WIP_UWGE_COVER_SPONSOR` | `det.{contractId,riskName,sponsorName,sponsorIdvalue,sponsorAge,sponsorDob}` | `TD/{riskName,sponsorName,sponsorIdvalue,sponsorAge,sponsorDob}` | OK |
| `ParticularOfNominationPotrait_ASP` | `NOMINEE_PRINT>0 && 070412 && Y` | `S12:1661-1667` -> `B12.addParticularOfNominationPotraitAsp:3085` -> `PlineParticularOfNominationPotraitAspHandler` | `UWPL_VI_POLICY_SCHEDULE_NOM` | `det.{policyNo,subsidiaryName,department,insuredName,nomineeName,relationship,sharePct,idType,idValue1}`; `RP/parentName` | `TD/{nomineeName,sharePct,...}`; `RP/parentName` | OK |
| `Blank_NoMargin` | `((PAGE_NUMBER-1)%2 != 0) && ASP_CUTOFF_IND=="Y" && 070412` | `S12:1674-1676` -> `B12.addBlank` | none | `det.blank` | `TD/blank` | **PARTIAL** (page parity dropped; always adds when `070412 && cutoff=Y`) |
| `Generic/ComplaintRedress_NoMargin` | `070412 && ASP_CUTOFF_IND=="Y"`; element `showComplaint=="Y" && showScheduleStd=="Y"` | `S12:1680-1682` -> `B12.addTemplate12ComplaintRedressNoMargin:3177` -> `PlineGenericComplaintRedressNoMarginHandler` | none | dispatch merged row | `TD` | STUB |
| `ImportantNotes` | lastPageFooter `070521` | `S12:1685-1687` -> `B12.addImportantNotes:3111` | none | dispatch merged row | `TD` | STUB |

Variant-specific nested calls:
| Parent | Nested subreports | Condition | Java | Status |
|---|---|---|---|---|
| `CIPrint_ATC` | `Note`, `ComplaintRedress_NoMargin`, `PerilsClauses_ATC`, `Signature`, `PIDMSetA`, `BNM_eMarketing` | unconditional except `PIDMSetA` (`PIDM_PRINT=="Y"`) | `S12.addCIPrintAtcNestedSubreports:1702` -> `addTemplate12Note`, `addTemplate12ComplaintRedressNoMargin`, `addPerilsClausesVariant`, `B12.addTemplate12Signature:882` (`PlineTemplate12SignatureHandler`: `SAPM_COMPANY` + REST `getCompBrnNumber`, `fnRppmGetSignatureAndRemarks`) -> `CD/{companyName,signature,remarks}` + `PRN/compInfo/compBRN`, PIDMSetA, BNM | OK |
| `CIPrint_ASTRO` | `Coverage`, `BNM_eMarketing`, `Note`, `PerilsClauses`(+PIDMSetA) | unconditional | `S12.addCIPrintAstroNestedSubreports:1726` | OK |
| `CIPrint_POS` | `Coverage_POS`, `BNM_eMarketing`, `PerilsClauses` | unconditional | `S12.addCIPrintPosNestedSubreports:1742` -> `B12.addCoveragePos:3287` (none) | STUB for Coverage_POS |
| `CIPrint_BMW` (per certificate row) | `Coverage_BMW`(+BNM), `Nomination_BMW`, `Note_BMW`, `CoverageBenefit_BMW`, `PerilsClauses_BMW`, `ComplaintRedress_NoMargin` | unconditional | `S12.addCIPrintBmwNestedSubreports:1758` -> `B12.addCoverageBmw:3186` (`PlineCoverageBmwHandler`: `UWPL_COVER_BENEFIT, UWPL_COVER, UWGE_COVER, UWGE_RISK_DET, UWGE_POLICY_VERSIONS, UWGE_RISK, UWPL_POLICY_BASES`), `addNominationBmwDetail:3208` (`UWPL_RISK_NOMINEE, UWGE_RISK_DET, UWGE_RISK, CMGE_CODE` -> `TD/{nomineeName,relationship,sharePct,idValue1,idType1}`), `addNoteBmw:3226`, `addCoverageBenefitBmw:3237` (`WIP_UWPL_POLICY_BASES, UWPL_POLICY_BASES` -> `TD/schemeName`), `addPerilsClausesBmw:3248` (`UWGE_RISK, UWGE_RISK_DET, UWGE_COVER_CLAUSE_HDR, UWGE_COVER_CLAUSE_DET, UWGE_COVER` -> proviso list), `addTemplate12ComplaintRedressNoMargin` | OK |
| `CIPrint_AZP` | `ScheduleOfBenefits_AZP`(+BNM), `PerilsClauses_AZP` (`PROV_IND>0`) | as left | `S12.addCIPrintAzpNestedSubreports:1776` -> `B12.addScheduleOfBenefitsAzp:3324` (`PlineScheduleOfBenefitsAzpHandler`: `UWGE_VI_POLICY_SCHEDULE_BASE, UWPL_COVER, WIP_UWPL_COVER` -> `TD/{plan,azpaPlan,azpbPlan,azSobInd}`) | OK |
| `CIPrint_POSM` | `Coverage_POSM`(+Header/Details), `PerilsClauses_POSM` (+SignatureFooter, BNM), `ParticularOfInsuredPotrait_POSM` (`FAM_IND != null`) | `PRL_IND > 0` gates only the missing `PerilsClausesDetails_POSM`; footer subreports are unconditional | `S12.addCIPrintPosmNestedSubreports:1796` -> `B12.addCoveragePosm:3339` (`PlineCoveragePosmHandler`, REST `fnGetBenefitHdr/Det`; `TD/{dependentAdultInd,childInd,adultPlanTitle,childPlanTitle,benefitCode,benefitDescp,adultPlanSi,childPlanSi,narration}`; `RP/planType`), `addPerilsClausesPosm:3407` (`PlinePerilsClausesPosmHandler`: `UWPL_VI_POLICY_SCHEDULE_PROV`; `RP/prlInd`; then `addSignatureFooter:3424` + BNM), `addParticularOfInsuredPotraitPosm:3440` (`UWPL_VI_POLICY_SCHEDULE_TPL4, CMGE_CODE`) | OK. `PerilsClausesDetails_POSM` has no jrxml in the dump so it is not mapped (comment `S12:332-335`) |
| `CIPrint_FWPA` | `Coverage_FWPA`, `PerilsClauses_FWPA`(+Details, SignatureFooter, BNM), `ParticularOfInsuredPotrait_FWPA` (`PRINT_POIP>1`) | as left | `S12.addCIPrintFwpaNestedSubreports:1817` -> `addTemplate12Coverage`, `B12.addPerilsClausesFwpa:3460`, `B12.addParticularOfInsuredPotraitFwpa:3473` (`fetchParticularOfInsuredPotraitFwpaResilient:2153`) over `UWPL_VI_POLICY_SCHEDULE_COVER`, `UWPL_VI_POLICY_SCHEDULE_PROV`, `UWPL_VI_POLICY_SCHEDULE_TPL4, CMGE_CODE` | OK |
| `CIPrint_Elite` | `Coverage`, `BNM_eMarketing`, `Note`(dead), `PerilsClauses`(+PIDMSetA) | `Note` gated `!070504` (always false here) | `S12.addCIPrintEliteNestedSubreports:1840` (guard at line 1844) | OK |
| `CIPrint_PA4A` (per row) | `ScheduleOfBenefits_PA4A`/`New`/`NewP3`, `BNM_eMarketing`, `CIPerilsClauses_PA4A`, `ParticularOfNomination_PA4A` | see 1.1.J | `S12.addCIPrintPa4aNestedSubreports:1860` -> `B12.addScheduleOfBenefitsPa4a:3502`, `addScheduleOfBenefitsNew:3513`, `addScheduleOfBenefitsNewP3:3524` (none; PIDMSetA when `PIDM_PRINT=="Y" && pCertInd=="Y"`), `addGenericBNMeMarketingConsent`, `addPerilsClausesVariant` (`CLAUSE_PRINT>0`), `addParticularOfNominationPotraitAsp` (`NOM_PRINT>0`; reuses the ASP nomination handler because the query is identical, comment `S12:1850-1858`) | OK |

### 3.3 PolicyDocument2 (merged: `S12.executeMergedPolicyDocument2:578-779`)

Primary: `B12.addPrimaryDocumentT12:525` -> `PlineT12PolicyDocumentHandler.fetchT12PolicyDocument` (single aggregation anchored on `WIP_UWGE_POLICY_CONTRACTS` with `$limit 1`): `UWGE_VI_POL_REPORT_PARAM, WIP_UWGE_POLICY_CONTRACTS, SB_UWGE_POLICY_CONTRACTS, UWPL_POLICY_BASES, WIP_UWPL_POLICY_BASES, OCP_POLICY_CONTRACTS, UWPL_POLICY_JOURNEY, UWGE_GST_TAX_INV, WIP_UWGE_POLICY_BASES, UWGE_POLICY_BASES, WIP_UWPL_RISK_PERSON, UWPL_RISK_PERSON, WIP_UWGE_COVER, UWGE_COVER`; REST `fnGetEinvoicePrintPolInd`, `fnShowStaxInvInd` (errors default to "N", handler lines 256-270). Payload: `det.{productcode,schemeName,journeyArea,showPdpaNoticeInd,eInvConsentInd,einvPrintInd}` -> `TD/*` (`B12:525-542`); `promoAmt` -> `RP/promoAmt`. Other fields (`showGstInvInd`, `showStaxInvInd`, `showTranInvInd`, `taxInd`) stay in the DTO and drive the `if`s only.

| Jasper node | Print condition (exact) | Java | Mongo / REST | Payload / XSD | Status |
|---|---|---|---|---|---|
| Q1 `Blank` before debit note | `... %2 != 0 && showDebitNote=="Y" && showScheduleStd=="Y" && PREM>0 && !SNATCHPA-M && !SNATCHPA-A && !POSMESRA` | none | - | - | **NOT IMPLEMENTED** (page parity + SNATCHPA test; no `SNATCHPA` string exists in the `plines` package) |
| Q2 `ComplaintRedress_SCB` | `070504 && (POSMESRA \|\| FWPA)` + `showComplaint=="Y"` | `S12:624-630` -> `B12.addGenericComplaintRedress:3275`, `addGenericCompanyRegNo:2878` (`PlineGenericCompanyRegNoHandler`, REST), `addTemplate12ComplaintRedressNoMargin` | none / REST | dispatch rows; `CD/companyRegNo` | OK (the POSMESRA parity filler inside SCB is NOT implemented) |
| Q3 `Domestic_ATC`/`Overseas_ATC` | **`PRODUCT_CODE=="070406"`** | `S12:616-621` -> `B12.addT12DomesticAtc:3296` / `addT12OverseasAtc:3310` (`PlineT12DomesticAtcHandler`/`Overseas...` = static `Collections.singletonList(1)`) | none | dispatch merged row -> `TD` | **DEVIATION**: Java does not test the product (D3) |
| Q4 `PremiumDebitNote` (T12 vs T4) | `showDebitNote=="Y" && PREM>0 && copyType != "BC"`; T4 template when product 070406 | `S12:633-659` (`!isBmw` replaces `PREM>0`) -> `B12.addT12PremiumDebitNote:1148` (`PlineT12PremiumDebitNoteHandler`) or `addT4PremiumDebitNote:1231` (`PlineT4PremiumDebitNoteHandler`); then `addSignature:867` (`PlineCommonSignatureHandler`: `SAPM_COMPANY`); then, when `PREM_BY_MST_IND=="Y"`, `addT12CustomerDtlsOfMasterPolicy:1299` (`UWPL_VI_TAX_INVOICE, CMGE_CODE`) or `addT4CustomerDtlsOfMasterPolicyDebit:1316` | `UWPL_VI_PREMIUM_DEBIT, UWPL_VI_POLICY_SCHEDULE_BASE, UWGE_POLICY_BASES, WIP_UWGE_POLICY_BASES, OCP_POLICY_BASES, CMGE_CODE` | `det.{debitNoteNo,grossPrem,rebateAmt,gstAmt,gstPct,commAmt,commPct,nettPrem,...,ptvAmt,premByMstInd,productName,totalpremium,premPayableRounded}`; `ED.{gstRateType,accCommPct,accComm2Pct,showGstLabel,product}`; `clientInfo.address`; `agentInfo`; `RP/{premDueAftPtv,premDueRoundedAftPtv}` | `TD/{debitNoteNo,grossPrem,...,ptvAmt}`; `ED/*`; `PRN/clientInfo/address/*`; `RP/{premDueAftPtv,premDueRoundedAftPtv}`; `CD/{companyName,signature,remarks}` | OK |
| Q5 `Blank` GST era | `(POSMESRA ? parity && ... : showTaxInvoice=="Y" && showScheduleStd=="Y" && TAX>0)` | `S12:661-663` (`showTaxInvoice=="Y" && showScheduleStd=="Y" && hasTax`) | none | `TD/blank` | PARTIAL (POSMESRA parity dropped) |
| Q6 `TaxInvoince` (+`CompanyContact`, `TaxInvoiceProductInfo`) | `showTaxInvoice=="Y" && TAX>0 && SHOW_GST_INV_IND=="Y"` | `S12:664-671` -> `B12.addTaxInvoince:951` (`PlineTaxInvoinceHandler`: `UWPL_VI_TAX_INVOICE, CMGE_CODE`), `addCompanyContact:556` (`PlineCommonCompanyContactHandler`: `UWGE_VI_POL_COVERNOTE, SAPM_COMPANY, CMGE_CODE`), `addTaxInvoiceProductInfo:898` (`PlineCommonTaxInvoiceProductInfoHandler`: `UWPL_VI_TAX_INVOICE, UWGE_POLICY_BASES`) | as left | `TD/*` tax invoice leaves; `CD/*`, `PRN/compInfo/{companyGstRegNo,fax}`; `ED/taxInvoice`; `RP/{policyholderName,parentName}` | OK |
| Q7 `Blank` after GST (dead) | `false && ...` | none | - | - | correctly absent |
| Q8 `ServiceTaxInvoice` (T12/T4) / `ServiceTaxInvoice_PA4A` | `showTaxInvoice=="Y" && SHOW_STAX_INV_IND=="Y" && EINV_PRINT_IND=="Y"`; 070414 -> PA4A; 070406 -> Template4 | `S12:673-737` -> `B12.addServiceTaxInvoicePA4A:784` (`PlineServiceTaxInvoicePA4AHandler`) or `addServiceTaxInvoiceRows:687` (`PlineCommonServiceTaxInvoiceHandler`, flag `useGstTaxInvNoAsTaxInvoice` = 070406) | `UWPL_VI_TAX_INVOICE, WIP_UWGE_POLICY_BASES_STAX, UWGE_POLICY_BASES_STAX, UWGE_GST_TAX_INV, WIP_UWGE_GST_TAX_INV, SAPM_SYS_CONSTANTS, UW_VALIDATION, CMGE_CODE, UWPL_POLICY_BASES, WIP_UWPL_POLICY_BASES, UWGE_POLICY_BASES, WIP_UWGE_POLICY_BASES, OCP_POLICY_BASES` | `det.*` (gross/stamp/gst/mco/fees...), `eInvStatus`, `staxList.staxPcts`; `ED.*` | `TD/*`, `TD/staxList/staxPcts/*`, `ED/{staxInvNo,sstRegisterNo,maskedTin,...}` | OK |
| Q8 nested `STCompanyContact` | unconditional once rows exist | `B12.addSTCompanyContact:577` (`PlineCommonSTCompanyContactHandler`: `UWGE_VI_POL_COVERNOTE, SAPM_COMPANY, CMGE_CODE`) | as left | `CD/*`, `PRN/compInfo/{companyGstRegNo,companySstRegNo}` | OK |
| Q8 nested `STaxInvoiceProductInfo` / `...MST` | `PREM_BY_MST_IND` != Y / == Y | `B12.addSTaxInvoiceProductInfo:599` / `addSTaxInvoiceProductInfoMST:636` (+ `addT12CustomerDtlsOfMasterPolicy`) | `UWPL_VI_TAX_INVOICE, SAPM_SYS_CONSTANTS, UW_VALIDATION` | `TD/{policyNo,subsidiaryName,productDescp,productcode,staxInvDate,contractId,idNo,issuedate,eInvConsentInd,eInvStatus,eInvCutOff}`, `ED/{staxInvNo,sstRegisterNo,maskedTin,taxInvoice}`, `CD/tinNo`; MST: `TD/{mstTinNo,mstBusRegNo}`, `TD/staxList/staxPcts/mstSstRegNo` | OK |
| Q8 nested `ServiceTaxInvoicePL` / `PLSub` | `STAX_COUNT>1` | `B12.addGenericServiceTaxInvoicePL:2814` / `PLSub:2846` | `WIP_/UWGE_POLICY_BASES, ..._BASES_STAX, ..._FEES_STAX` | `TD/{rebateAmt,nettPrem,premDue,premDueRounded,staxAmt,staxPct,staxAmt2,staxPct2,grossPrem,agicFee2,feeSst2,effDate,effDate2}`, `TD/staxList/staxPcts/{staxPrem,staxCount,staxPrem2}` | OK |
| Q8 nested `QRSection` | non-PA4A: `EINV_CONSENT_IND=="Y" && EINV_STATUS=="VALID" && QR_IND=="Y"`; PA4A: `... && QR_CNT>0` | `S12:694-698`, `730-734` -> `B12.addGenericQRSection:2755` (`PlineGenericQRSectionHandler`) | `UWGE_GST_TAX_INV, WIP_UWGE_GST_TAX_INV` | `TD/{digitalSign,einvTime,einvQr}` | OK |
| Q9 `Blank` SST | `showTaxInvoice=="Y" && SHOW_STAX_INV_IND=="Y"` | `S12:739-741` | none | `TD/blank` | OK |
| Q10-Q12 `TransitionalInvoice`, `Blank`, `TransitionalInvoice2` (+ nested) | `showTaxInvoice=="Y" && SHOW_TRAN_INV_IND=="Y"` | `S12:742-763` -> `addBlank`, `B12.addT12TransitionalInvoice:1079` (`PlineT12TransitionalInvoiceHandler`), `addCompanyContact`, `addTransitionalInvoiceProductInfo:926`, `addGenericSSTNote:2772` (static), `addTransitionalInvoice2:1021` (`PlineTransitionalInvoice2Handler`: `UWPL_VI_TAX_INVOICE, SB_UWGE_POLICY_BASES, WIP_UWGE_POLICY_BASES, CMGE_CODE`), `addSTCompanyContact`, `addSTaxInvoiceProductInfo`, `addGenericSSTNote` | as left | `TD/*` | PARTIAL: Jasper order is invoice1, Blank, invoice2; Java adds the Blank FIRST (`S12:744`, before `addT12TransitionalInvoice` at 747), although its comment says "between" (D13) |
| Q13 `PDS_Elite` | **dead** `false && ...` | `S12:773-775` -> `B12.addPDSElite:1351` | REST `getCompBrnNumber` | `CD/companyRegNo` | **DEVIATION** (Java prints, Jasper never does; D2) |
| Q14 `PDS_Income` | **dead** | `S12:770-772` -> `B12.addPDSIncome:1342` | REST | `CD/companyRegNo` | **DEVIATION** |
| Q15 `PDPA` (CommonReport, dead) | dead | none | - | - | correctly absent |
| Q16 `NominationForm` (dead) | dead | none | - | - | correctly absent |
| Q17 `Generic/PDPA` | `SHOW_PDPA_NOTICE_IND=="Y" && checklistCopy=="N" && copyType in (IO, ID)` | `S12:764-767` -> `B12.addPDPA:1333` (`PlineGenericPDPAHandler`, REST) | REST | `CD/companyRegNo` | OK |
| (no jrxml band) ParticularOfInsured Tpl4/Base, Nomination PA4A/BMW, InstallmentDetails | - | `S12:603-614` -> `B12.addParticularOfInsuredTpl4:2553`, `addParticularOfInsuredBase:2569`, `addNominationPA4A:2609`, `addNominationBMW:2622`, `addInstallmentDetails:2661` (`PlineParticularOfInsuredHandler`: `UWPL_VI_POLICY_SCHEDULE_TPL4, UWPL_VI_POLICY_SCHEDULE_BASE`; `PlineNominationHandler`: `UWPL_VI_POLICY_SCHEDULE_BASE, UWPL_RISK_NOMINEE, UWPL_VI_POLICY_SCHEDULE_NOM, UWPL_VI_ENDT_SCHEDULE_NOM, CMGE_CODE`; `PlineInstallmentDetailsHandler`: `UWGE_VI_POLICY_INSTALLMENT`) | as left | rows appended unconditionally; `RP/{parentName,planDescp,pmtDueDate}`; `TD/*`, `TD/nomineeList/nominee/*` | **EXTRA** (no such band in PD2 jrxml; D4) |

### 3.4 Landscape (merged: `S12.executeMergedLandscape:801-1497`)

Primary: `Template12LandscapeDispatchHandler.fetchDispatch:52-258`. Collections: `WIP_UWPL_RISK_NOMINEE, UWPL_RISK_NOMINEE, WIP_UWPL_COVER_DIAGNOSIS, UWPL_COVER_DIAGNOSIS, WIP_UWGE_POLICY_VERSIONS, UWGE_POLICY_VERSIONS, WIP_UWGE_POLICY_CONTRACTS, SB_UWGE_POLICY_CONTRACTS, UWGE_POLICY_CONTRACTS, UWPL_POLICY_BASES, WIP_UWPL_POLICY_BASES, OCP_POLICY_CONTRACTS, SAPM_SYS_CONSTANTS, UWGE_VI_POL_REPORT_PARAM, WIP_UWGE_COVER, UWGE_COVER, WIP_UWPL_RISK_PERSON, UWPL_RISK_PERSON, CMGE_CODEREL_DET`. Returns `null` when the `CI_LANDSCAPE_PRT` row matches (`$match` on `$size EXCL_ROW == 0`, lines 223-236), and `S12:805-808` then returns without emitting any band. Dispatch row: `S12:833-839` writes `productcode, schemeName, aspCutoffInd, issuedate, gossPremB4Promo` as its own `transDet`; `RP/{promoAmt,premiumPrint}` are set at `S12:490-498`.

| Jasper band | Condition (Java `if` at) | Java handler (`.fetch`) | Mongo | Payload (each row becomes its own `reportData`) | XSD | Status |
|---|---|---|---|---|---|---|
| L1 `ParticularOfInsuredLandscape_ASP` | `070412 && aspCutoffInd=="N"` (`S12:842`) | `PlineT12ParticularOfInsuredLandscapeAspHandler` (sort `COV_SEQ_NUM` = `$toDouble`, lines 72-73) | `UWPL_VI_POLICY_SCHEDULE_TPL4, SAPM_SYS_CONSTANTS` | `det.*` (`S12:844-884`); `ED.{source,partnerType,contractId}`; `RP/{parentName,planDescp,psi}` | `TD/*`, `ED/*` | OK |
| L2 `ParticularOfPremiumLandscape_ASP` | same (`S12:893`) | `PlineT12ParticularOfPremiumLandscapeAspHandler` | `UWPL_VI_POLICY_SCHEDULE_PREM, CMGE_CODE` | `det.{premium,loading,grossPrem,discount,mcoFee,optImaFee,gstAmount,totprem,totalpremium,joinDate,policyNo,referenceNo,addSports,addSports1,doubleIndemnity,staxAmt,stiPrem,llbPrem,agicFee,feeSst,...}` | `TD/*` | OK |
| L3 `ParticularOfSponsorLandscape` | same (`S12:933`) | `PlineT12ParticularOfSponsorLandscapeHandler` | `UWPL_RISK_PERSON, UWGE_COVER, UWGE_COVER_SPONSOR, OCP_POLICY_BASES, UWGE_POLICY_VERSIONS, WIP_UWGE_POLICY_VERSIONS` | `det.{contractId,policyref,referenceNo,riskName,sponsorName,sponsorIdvalue,sponsorDob,sponsorAge}` | `TD/{sponsorName,sponsorIdvalue,sponsorDob,sponsorAge,...}` | OK |
| L4 `ParticularOfBenefitsLandscape` | `benefitPrint>0 && !070504 && !070521 && !070406 && !100908` (`S12:951-955`) = intersection of band and element conditions | `PlineT12ParticularOfBenefitsLandscapeHandler` | `UWPL_VI_POLICY_SCHEDULE_BEN, SAPM_SYS_CONSTANTS, CMGE_CODE` | `det.{...gender,nationalityDescp,dateOfBirth,age,rbAmt,...}` | `TD/*` | OK |
| L5 `..._Elite` | `070504 && benefitPrint>0 && ELITESHIELD` (`S12:988`) | `PlineT12ParticularOfBenefitsLandscapeEliteHandler` | `UWPL_VI_POLICY_SCHEDULE_TPL4` | `det.*` + `sobList.benefits.benefit.{weeklyBenefit,casualtyCashBenefit,dailyCareCashBenefit,hospitalReliefCashBenefit}` | `TD/sobList/benefits/benefit/*` | OK |
| L6 `..._Income` | `070504 && benefitPrint>0 && INCOMESHIELD` (`S12:1041`) | `PlineT12ParticularOfBenefitsLandscapeIncomeHandler` | same | `det.{ttd,tpd,funExp,adRb,pdRb,...}` + benefit `{weeklyBenefit,snatchTheftBenefit}` | `TD/{ttd,tpd,funExp,adRb,pdRb}` | OK |
| L7 `ParticularOfPremiumLandscape_Elite` | `070504 && premiumPrint>0 && ELITESHIELD` (`S12:1095`) | `PlineT12ParticularOfPremiumLandscapeEliteHandler` | same | `det.*`, `coverList.cover.covSeqRef`, benefit leaves | `TD/coverList/cover/covSeqRef` | OK |
| L8 `..._Income` | `070504 && premiumPrint>0 && INCOMESHIELD` (`S12:1162`) | `PlineT12ParticularOfPremiumLandscapeIncomeHandler` | same | as L7 | as L7 | OK |
| L9 `ParticularOfNominationLandscape_Elite` | `070504 && nomineePrint>0 && !POSCOVER && !AZP1 && !AZP2` (`S12:1228`) | `PlineT12ParticularOfNominationLandscapeEliteHandler` | `UWPL_VI_POLICY_SCHEDULE_NOM` | `det.{policyNo,referenceNo,insuredName,nomineeName,relationship,sharePct,idType,idValue1,subsidiaryName,department}` + `nomineeList.nominee.{nomineeName,nomineeRelation,nomineeShare,idNo,nomineePrint}`; `RP/parentName` | `TD/nomineeList/nominee/*` | OK |
| L10 `..._POS` | `070504 && nomineePrint>0 && POSCOVER` (`S12:1262`) | `PlineT12ParticularOfNominationLandscapePosHandler` | same | as L9 | as L9 | OK |
| L11 `ParticularOfNominationLandscape` | `nomineePrint>0 && !070504` (`S12:1291`) | `PlineT12ParticularOfNominationLandscapeHandler` | same | as L9 | as L9 | OK |
| L12 `ParticularOfDiagnosisLandscape` | `diagnosisPrint>0 && !070504 && !100908` (`S12:1319`) | `PlineT12ParticularOfDiagnosisLandscapeHandler` (sort raw `COV_SEQ_REF`, line 41) | `UWPL_VI_POLICY_SCHEDULE_DIAG` | `det.{loadingCode,typeOfLoading,exclusionInd,sublimitAmt,loadPct,covId}`, `coverList.cover.covSeqRef`, `diagnosisList.diagnosis.{insuredName,idNo,icdDescp,diagnosisPrint}` | `TD/diagnosisList/diagnosis/*` | OK (sort deviation D9) |
| L13 `ParticularOfInsuredLandscape_AMC` | `100908` (`S12:1356`) | `PlineT12ParticularOfInsuredLandscapeAmcHandler` (sort raw `COV_SEQ_REF`, line 38) | `UWPL_VI_POLICY_SCHEDULE_TPL20, CMGE_CODE` | `det.*`, `coverList.cover.{covSeqRef,covBasicPrem,covLoadAmt,stiCovBasicPrem,covStaxAmt,covGrossPrem,covDiscAmt,covDiscPct,covRebateAmt,covRebatePct}`, `det.sumLoading` | `TD/coverList/cover/*`, `TD/sumLoading` | OK (sort deviation D9) |
| L14 `ParticularOfDiagnosisLandscape_AMC` | `diagnosisPrint>0 && 100908` (`S12:1408`) | `PlineT12ParticularOfDiagnosisLandscapeAmcHandler` (raw sort, line 38) | `UWPL_VI_POLICY_SCHEDULE_DIAG` | as L12 + `diag.icdNarr` | as L12 | OK |
| L15 `ParticularOfNominationLandscape_AZP` | `070504 && nomineePrint>0 && (AZP1 \|\| AZP2)` (`S12:1445`) | `PlineT12ParticularOfNominationLandscapeAzpHandler` | `UWPL_VI_POLICY_SCHEDULE_NOM` | as L9 | as L9 | OK |
| L16 Template4 `Landscape_jrxml` | `070406 && promoAmt>0` (`S12:1478-1479`) | `Template4LandscapeDispatchHandler.fetchDispatch` (Template4's handler) | (Template4) | own transDet `{productcode, aspCutoffInd, boatPrint, sponsorPrint, gossPremB4Promo}` | `TD/{boatPrint,sponsorPrint,gossPremB4Promo,...}` | **PARTIAL**: only dispatch fields; Template4 landscape's own sub-bands (ATC benefits/premium/promo/companion/boat) are not emitted here (**inferred**: owned by the Template4 team) |

### 3.5 Summary list of NOT IMPLEMENTED / stub / merged items
* NOT IMPLEMENTED (cannot, runtime page numbers): PrintSeriesFooter call 2; PD2 Blank Q1 (parity + SNATCHPA exclusion); parity half of `Blank_NoMargin`, PD2 Q5 (POSMESRA), `ComplaintRedress_SCB@70` (POSMESRA filler); `PerilsClausesDetails_POSM` (no jrxml in the dump).
* NOT IMPLEMENTED (correct, dead upstream): PD2 Blank after GST invoice (Q7), CommonReport PDPA (Q15), NominationForm (Q16).
* STUB (dispatch-only rows): ComplaintRedress, ComplaintRedress_SCB, ComplaintRedress_NoMargin, Coverage_POS, ScheduleOfBenefits_ASP/AMC/PA4A/New/NewP3/ASPLLB/ASPSTI, ImportantNotes, SSTNote, Domestic/Overseas_ATC, PerilsClauses_FWPA header.
* MERGED: PolicyDocument2 and Landscape merged into `Template12PolicyDocumentService` (commit `c0c6d449a`); `PerilsClausesDetails_FWPA` merged into `addPerilsClausesFwpa:3465`; `CoverageHeader_POSM`/`CoverageDetails_POSM` merged into `addCoveragePosm:3339`; `SignatureFooter`/`BNM_eMarketing` merged into `addPerilsClausesPosm/Fwpa`.
* Un-reached Java: `Template12PolicyDocument2Service`, `Template12LandscapeService` (unregistered); `PlineT12ParticularOfInsuredPosHandler` (no caller in `src/main`; the POS landscape uses `PlineParticularOfInsuredHandler.fetchParticularOfInsuredLandscapePos`).

---------------------------------------------------------------------------------------------------
## 4. Differences between Jasper behaviour and the Java implementation

Each item: **Jasper** evidence, **Java** evidence, impact. "(inferred)" = my reasoning from the code, not something I could execute.

### D1. Conditions that depend on `$V{PAGE_NUMBER}` cannot be evaluated in Java
* Jasper: PolicyDocument background `PrintSeriesFooter` call 2 (`...$V{PAGE_NUMBER}!=1 && $V{PAGE_NUMBER}!=2`, PolicyDocument.jrxml.data:141); CIPrint `Blank_NoMargin` (`(($V{PAGE_NUMBER})-1)%2 != 0 && ASP_CUTOFF_IND=="Y" && 070412`, CIPrint:1269,1272); PolicyDocument2 Q1 blank before debit note (`...%2 != 0 && ...`, PD2:216,219), Q5 blank for POSMESRA (PD2:486); `ComplaintRedress_SCB` POSMESRA filler (ComplaintRedress_SCB:70); also the page footer texts `"PAGE: " + (PAGE_NUMBER - 1)` etc. (PolicyDocument:112,134; PD2:210; Landscape:130).
* Java: call 2 skipped (`S12:235-237`); Blank_NoMargin added whenever `070412 && cutoff=Y` and the code comment says "may over-include Blank_NoMargin rows on even pages" (`S12:1669-1676`); Q1 absent; Q5 reduced to the non-POSMESRA branch (`S12:661-663`); SCB filler absent. Page numbers are left to the renderer; only `pFirstPageNumber` is echoed (`S12:450`).
* Impact: the odd/even page logic (duplex printing: each document must start on a front page) has to be handled downstream. (inferred) For POSMESRA and ASP-cutoff policies the blank pages may be wrong unless the renderer re-implements it.

### D2. `false &&` dead bands in PolicyDocument2 - Java prints two of them
* Jasper: PDS_Elite band `false && 070504 && ELITESHIELD` (PD2:946), PDS_Income band `false && 070504 && INCOMESHIELD` (PD2:1001), CommonReport PDPA `false && 070504` (PD2:1056), NominationForm `false && 070504` (PD2:1111), Blank after GST invoice `false && showTaxInvoice...` (PD2:525). A band whose print-when is false never renders, even though the inner element condition is true.
* Java: PDS_Income / PDS_Elite are wired and print when scheme = INCOMESHIELD / ELITESHIELD (`S12:770-775`, comment "JRXML band printWhenExpression is false && ... (dead code upstream) but wired here per explicit instruction"); unit tests `execute_070504IncomeShield_firesPdsIncomeOnly` / `...EliteShield...` assert it. PDPA(CommonReport), NominationForm and the post-GST blank are correctly not implemented.
* Impact: Java output contains a `PDS_*` marker row (a `companyDesc.companyRegNo`) for those two schemes that the Jasper report never produced. If the product team enables the band in Jasper later it will already work; today it is a behavioural difference.

### D3. Domestic/Overseas ATC page is product-gated in Jasper, unconditional in Java
* Jasper: band `PRODUCT_CODE.equalsIgnoreCase("070406")` (PD2:255) selecting `"DOM".equalsIgnoreCase(JOURNEY_AREA) ? Domestic_ATC : Overseas_ATC` (PD2:336).
* Java: `if ("DOM".equalsIgnoreCase(journeyArea)) addT12DomesticAtc(rd); else addT12OverseasAtc(rd);` with the comment "always fires one or the other" and no product test (`S12:616-621`); the same in the unregistered class (`Template12PolicyDocument2Service:94-100`).
* Impact: every non-ATC policy also gets a (dispatch-only, merged) Overseas_ATC marker row. Tests `execute_pd2JourneyAreaDom_usesDomesticAtcBranch` and `execute_pd2PrimaryDtoNull_journeyAreaDefaultsToOverseas` encode the current behaviour.

### D4. Extra bands in the Java PolicyDocument2 block + dropped `certNo`
* Jasper: no band for `ParticularOfInsuredTpl4/Base`, `NominationPA4A/BMW`, `InstallmentDetails` in PD2 (jrxml and compiled `.jasper`; section 1.2).
* Java: `S12:603-614` always calls `addParticularOfInsuredTpl4`, `addParticularOfInsuredBase`, `addNominationPA4A`, `addNominationBMW`, `addInstallmentDetails` and uses their first-row `parentName/planDescp/pmtDueDate` for `reportParameters`. The merged method hard-codes `String certNo = "";` (`S12:583`) so `addNominationBMW(contractId, certNo, rd)` never receives a certificate, whereas the unregistered class read `certNo` from the request (`Template12PolicyDocument2Service:67,91`).
* Impact: unconditional insured/nomination rows for every product in the PD2 section; BMW nomination always for cert "". (inferred) These calls probably come from an older PD2 layout or from the mapping sheet - I could not find them in the dump.

### D5. PD2 "Blank before debit note" and SNATCHPA exclusion absent
* Jasper PD2 Q1 excludes `SNATCHPA-M`, `SNATCHPA-A`, `POSMESRA` and requires `PREM>0` (PD2:216).
* Java never mentions SNATCHPA (no match in the package) and has no such Blank. Debit-note gating is `showDebitNote=="Y" && !BMW && copyType != "BC"` (`S12:633`), i.e. `PREM>0` is approximated by "not BMW/BMW1" (the SQL sets `prem=0` only for those two schemes, PD2:59-62, so equivalent).

### D6. Oracle error emulation vs silent precedence (three different primary fetches)
* Jasper: scalar sub-selects with `UNION`/`DISTINCT` return more than one row -> Oracle raises ORA-01427 when WIP and posted disagree (e.g. PolicyDocument `PRODUCT_CODE`/`SCHEME_NAME`).
* Java PolicyDocument primary: **emulates the error** - `requireNoConflict` and `distinctValueOrConflict` throw `IllegalStateException("Ambiguous ... (Oracle's UNION would raise ORA-01427 here)")` (`PlinePolicyDocumentMainHandler:1463,1474,1543-1585`); `addTemplate12PDPrimaryDocument` has no try/catch so the request fails (`B12:508-523`).
* Java PolicyDocument2 primary: no conflict detection; posted/WIP precedence by `$ifNull` (`PlineT12PolicyDocumentHandler`: product = WIP then SB; scheme = posted then WIP-not-in-OCP; EINV consent posted(v1) else WIP else "N"; TAX = posted OR WIP). Jasper `tax` is `select 1 ... union all select 1 ...` (PD2:79-88) which would itself raise ORA-01427 if both exist; `EINV_CONSENT_IND` is a UNION of two `NVL` selects (PD2:113-115), same risk.
* Java Landscape primary: product = WIP, else SB, else **UWGE_POLICY_CONTRACTS** (extra third fallback, `Template12LandscapeDispatchHandler:98-110`; Jasper has only WIP UNION ALL SB, Landscape:56-64); issue date = WIP else posted (Jasper `UNION`).
* Impact: the same contract can behave differently per report: PolicyDocument fails fast on data conflict, PD2/Landscape continue silently. Class javadoc "CRITICAL FACT #1" documents the deliberate non-unification (`S12:62-72`).

### D7. `PROMO_AMT` (Jasper join multiplication vs Java existence gate; Landscape missing version filters)
* Jasper (PD2:64-71, Landscape:82-89): `sum(NVL(COV_PROMO_AMT,0))` from `wip_UWPL_RISK_PERSON sp, wip_uwge_cover upv where sp.contract_id = upv.contract_id ...` - a join on contract only, so every cover row is repeated once per risk-person row. (inferred) With N insured persons the printed promo amount is N times the cover sum.
* Java PD2: `$limit 1` existence lookup of risk persons then a plain `$group $sum` over covers with version filters `VERSION_NO` missing (WIP) / `=1` (posted) (`PlineT12PolicyDocumentHandler`, promo block) - **not multiplied**.
* Java Landscape: same existence gate but **no version filter** on `WIP_UWGE_COVER`/`UWGE_COVER` (`Template12LandscapeDispatchHandler:196-211`) and no `VERSION_NO=1` on `UWPL_RISK_PERSON` (line 190-195); Jasper has `version_no IS NULL` / `version_no = 1` / `sp.VERSION_NO=1`.
* Impact: `RP/promoAmt` and the `070406 && promo>0` Landscape band (L16) can differ from what Jasper computed; Landscape can include covers of older versions.

### D8. Landscape flags `NOMINEE_PRINT` / `DIAGNOSIS_PRINT` inspect only the first joined version row
* Jasper: `exists (nominee join wip policy versions where version_no IS NULL ...) UNION ALL (nominee join posted versions where version_no = 1)` (Landscape:34-54).
* Java: `pipelineWithPv` joins nominee/diagnosis rows to the version collection and then tests `$arrayElemAt(_pv.VERSION_NO, 0)` (null for WIP, 1 for posted) (`Template12LandscapeDispatchHandler:260-273`). (inferred) If a contract has several version rows the first array element decides, which can differ from Oracle's "any row matches".

### D9. Ordering and null-sort
* Jasper orders every landscape/potrait list by `cast(COV_SEQ_REF as decimal(15,2)) asc` (e.g. ParticularOfInsuredLandscape_ASP:53, _AMC:58, DiagnosisLandscape:30, _AMC:32); `Coverage` by `PLAN_CODE, SORT_SEQ`; clauses by `CLAUSE_CODE`; installment by `SEQ_NO`; BMW by `det.cert_no`.
* Java converts to number where it noticed the string-storage problem (`$toDouble(COV_SEQ_REF)` in `PlineT12ParticularOfInsuredLandscapeAspHandler:72-73`, `...BenefitsLandscape*`, `...PremiumLandscape*`, `PlineParticularOfInsured*Potrait*`, `PlineParticularOfInsuredHandler:145-146`; Coverage handler comment lines 66-76). But `PlineT12ParticularOfInsuredLandscapeAmcHandler:38`, `PlineT12ParticularOfDiagnosisLandscapeHandler:41` and `...DiagnosisLandscapeAmcHandler:38` sort the **raw** `COV_SEQ_REF`. (inferred) If the field is stored as a string, "10" sorts before "2"; if numeric it is equal.
* `PlineT12ParticularOfNominationLandscape*`, `...SponsorLandscape` have no sort (Jasper queries have no ORDER BY either, so Oracle order is unspecified).
* Null sort (inferred, DB semantics, not verified on data): Oracle `ORDER BY ... ASC` puts NULLs last; MongoDB `$sort` ascending puts null/missing first. Any row with a missing sequence field would move to the top in Java.
* ROWNUM: Jasper `rownum = 1` (product code, nominee/diagnosis flags) -> Java `$limit 1` without a deterministic sort, same non-determinism as Oracle when more than one row exists.

### D10. Single-envelope output vs band-by-band rendering
* Jasper renders each band as its own layout with page breaks; Java appends rows to one `RD.prntDetails.transDetList` (`mergeSingleTransDet` merges single-row bands onto *every existing transDet*, `B12:189-258`) and Landscape/POS rows to separate `reportData` entries (`toLandscapeReportData`, `S12:2077-2091`). The code itself warns the consumer behaviour is unconfirmed (`S12:159-165`) and that `endorsementDetails` is "last call wins" (`B12:296-331`).
* Impact: band boundaries are lost; a merged single-row band overwrites scalar fields set by an earlier band (non-null last-writer-wins, `mergeNonNullFields` `B12:212-258`). That is why commit `7399d2fc1` ("mergenonnullvalues changes") and `c0c6d449a` ("dynamic merging") exist.

### D11. Reflective merge and date handling
* Java reformats every String field named `*date*`/`*dob*` to `dd/MM/yyyy` at the end (`S12:2105-2230`, applied `S12:549-554`), because `toLocalDateTime` strips the time (`B12:366-381`). Jasper passes `issueDate`/`signIssueDate` to subreports as `dd-MMM-yyyy` text (PolicyDocument:245,327) and each report formats with its own pattern. XSD declares these as `xs:dateTime`, so the three representations differ (XSD `dateTime`, Java `LocalDateTime` serialised `dd/MM/yyyy` for `reportParameters`, Jasper `dd-MMM-yyyy`).

### D12. Handler written but never called
* `PlineT12ParticularOfInsuredPosHandler` (collection `UWPL_VI_POLICY_SCHEDULE_TPL4`, sorts the projected `covSeqRef` raw, line 87) has no caller in `src/main` (grep); the service calls `PlineParticularOfInsuredHandler.fetchParticularOfInsuredLandscapePos` instead (numeric sort). Tests exist for the unused one (`PlineT12ParticularOfInsuredPosHandlerTest`, 6 tests).

### D13. Blank placement in the transitional-invoice block
* Jasper order: TransitionalInvoice (754) -> Blank (821) -> TransitionalInvoice2 (879).
* Java order: Blank first (`S12:744`), then TransitionalInvoice (747) ... TransitionalInvoice2 (756), although the comment says "between". Because rows are appended in call order, the `blank` row precedes both invoices.

### D14. Request values vs row-derived values for the QR section
* Jasper: `ServiceTaxInvoice` passes `pTaxInvNo = $F{STAX_INV_NO}` to its nested subreports (ServiceTaxInvoice:935-937; PA4A passes `$F{TAX_INVOICE}`, ServiceTaxInvoice_PA4A:831-833); `QRSection` filters `STAX_INV_NO = $P{pTaxInvNo}` (Generic/QRSection query).
* Java: `addGenericQRSection(contractId, versionNo, pTaxInvNo, rd)` uses the **request** `pTaxInvNo` (default "", `S12:153,697,733`) even though `PlineCommonServiceTaxInvoiceDto.staxInvNo` / `taxInvoice` are available; `PlineGenericQRSectionHandler` filters on `STAX_INV_NO` (lines 55-58). (inferred) Unless the caller sends the invoice number, the QR section is empty. `pGstTaxInvoiceNo` is likewise taken from the request for STaxInvoiceProductInfo, but that handler ignores it (`PlineCommonSTaxInvoiceProductInfoHandler` javadoc: "matched on CONTRACT_ID only"), so no effect. For TaxInvoince/Transitional Java correctly takes the number from the parent row (`S12:667,749,758`).

### D15. `reportParameters` values set in Java but not serialised
* `PlineReportParameters` has `@JsonIncludeProperties` (whitelist, lines 25-38). `pCertNoFrom`, `pCertNoTo`, `pTaxInvNo`, `pissueDate`, `psignIssueDate`, `signIssueDate`, `pContractId`, `pPolicyNo`, `versionNo`, `companyParamType` are set by `S12:456-469,549-551` / `B12:3570-3598` but are not in the whitelist, so they never reach the XML (inferred from the annotation semantics). `ShowOutPrintCovLetter`, `pMstPolicyNo`, `pContractIdList` setters are commented out (`S12:454-461`), although `PlineReportParameters` has `pShowOutPrintCovLetter` in the whitelist - and the repo's XSD (v0.1.6) has no such element. See D16.

### D16. XSD copy in the repo lags the code
* Repo XSD header: v0.1.6 (Pline.xsd:102-120). Code comments refer to **v0.1.10** (`PlineReportData.java:20`, `PlineReportNames.java:13`: `mainReportName`/`subReportName`). `grep mainReportName Pline.xsd` finds nothing, and the whitelist contains `pShowOutPrintCovLetter`, `pShowPDPANotice`, `pReportTitle`, `pshowEndtSchedule`, `pCreatedDateFrom/To` which are not declared in `reportParameters` of the repo copy (Pline.xsd:478-544). `PlineXsdValidationTest` Template12 tests validate a hard-coded minimal XML only (section 8). (inferred) The authoritative XSD lives outside this repo.

### D17. Registration / routing
* Only `.../Template12/PolicyDocument` is a Spring bean. PD2 and Landscape report names no longer resolve to a Plines strategy (`CommonController:203`), and their work is done inside the PolicyDocument call. The old standalone classes remain as dead reference code with their own tests (`Template12PolicyDocument2ServiceTest` 33, `Template12LandscapeServiceTest` 28 - they instantiate the classes directly).

### D18. Error handling style
* Every `B12.add*` swallows exceptions (`log.warn`) and returns an empty/neutral value, whereas Jasper would fail the whole report on a bad subquery. Exceptions: the PolicyDocument primary fetch (throws, D6) and `landscapeDispatchHandler.fetchDispatch` (not wrapped; test `execute_landscapeDispatchHandlerThrows_notWrappedInTryCatch_propagates`). Result: partial output is possible in Java where Jasper fails.

### D19. Mixed definitions of "scheme" across the three primaries
| Report | Scheme source in Jasper | Scheme source in Java |
|---|---|---|
| PolicyDocument | `wip_uwpl_policy_bases` UNION `uwpl_policy_bases` (`top_indicator='Y'`) (PolicyDocument:57-63) | `distinctValueOrConflict(WIP_UWPL_POLICY_BASES)` + `(UWPL_POLICY_BASES, TOP_INDICATOR='Y')` (`MainHandler:1470-1474`) |
| PolicyDocument2 | `uwpl_policy_bases` (VERSION_NO=1) UNION ALL `wip_uwpl_policy_bases` NOT EXISTS `OCP_POLICY_CONTRACTS` (PD2:90-96) | same logic via `$ifNull(posted, WIP-not-in-OCP)` |
| Landscape | same as PD2 (Landscape:65-72) | same (`Template12LandscapeDispatchHandler:112-135`) |

---------------------------------------------------------------------------------------------------
## 5. Product / scheme decision matrix

Product codes below are exactly as they appear in the jrxml conditions. Names in parentheses are taken from file names/comments (inferred meaning).

### 5.1 PolicyDocument (main) - which CI print variant and extras
| PRODUCT_CODE | SCHEME_NAME | Variant printed (PolicyDocument) | Complaint page (if `showComplaint=Y`) | Extra | Java place |
|---|---|---|---|---|---|
| any except 070504/070414/070406 (e.g. 070412, 070521, 100908, 0702xx, 089xxx, 070302, 070510, 070413) | any | `CIPrint.jrxml` (default) with its own nested rules (1.1.A) | `ComplaintRedress` unless BMW/BMW1, or 070412 with `ASP_CUTOFF_IND=Y`. 070412 + cutoff Y gets `ComplaintRedress_NoMargin` inside CIPrint | - | `S12:247-279`, `addCIPrintDefaultNestedSubreports` |
| 070406 (ATC) | any | `CIPrint_ATC` | none (excluded) | ATC Note/Perils/Signature/PIDMSetA/BNM inside | `S12:248-257`, `addCIPrintAtcNestedSubreports` |
| 070414 (PA4A) | any | `CIPrint_PA4A` | `ComplaintRedress` (not excluded) | SOB variants by `SHOW_NEW_SOB`/`SHOW_SOB_P3_IND`, perils by `CLAUSE_PRINT`, nomination by `NOM_PRINT` | `S12:393-404` |
| 070504 | ASTRO | `CIPrint_ASTRO` | `ComplaintRedress_SCB` | Coverage, Note, Perils | `S12:285-290` |
| 070504 | BMW, BMW1 | `CIPrint_BMW` (per cert in `pCertNoFrom..pCertNoTo`) | none (SCB excluded; NoMargin inside BMW) | Coverage_BMW, Nomination_BMW, Note_BMW, CoverageBenefit_BMW, PerilsClauses_BMW | `S12:300-307` |
| 070504 | POSCOVER | `CIPrint_POS` | `ComplaintRedress_SCB` | with dependants: `Blank` + `ParticularOfInsuredLandscape_POS` | `S12:314-319`, `425-431` |
| 070504 | AZP1, AZP2 | `CIPrint_AZP` | `ComplaintRedress_SCB` | ScheduleOfBenefits_AZP, Perils if `PROV_IND>0` | `S12:324-329` |
| 070504 | POSMESRA | `CIPrint_POSM` | none in PolicyDocument (SCB excluded); **PD2 prints `ComplaintRedress_SCB` instead** | Coverage_POSM, Perils_POSM, Insured potrait if `FAM_IND` | `S12:340-349` |
| 070504 | FWPA | `CIPrint_FWPA` | none in PolicyDocument; **PD2 prints SCB** | Coverage_FWPA, Perils_FWPA, Insured potrait if `PRINT_POIP>1` | `S12:356-365` |
| 070504 | other, incl. ELITESHIELD, INCOMESHIELD | `CIPrint_Elite` | `ComplaintRedress_SCB` | Coverage, Perils (Note dead) | `S12:374-381` |

### 5.2 Landscape - which landscape pages print (whole report suppressed when scheme has `GPA_SCHEME/CI_LANDSCAPE_PRT=Y`)
| Product / scheme | Pages |
|---|---|
| 070412 with `ASP_CUTOFF_IND=N` | Insured_ASP, Premium_ASP, Sponsor (L1-L3); plus Benefits (L4) if `BENEFIT_PRINT>0`; Nomination (L11) if `NOMINEE_PRINT>0`; Diagnosis (L12) if `DIAGNOSIS_PRINT>0` |
| 070412 with `ASP_CUTOFF_IND=Y` | none from Landscape (the ASP potrait pages are inside CIPrint instead) except L4/L11/L12 as above |
| 070504 ELITESHIELD | Benefits_Elite (L5), Premium_Elite (L7), Nomination_Elite (L9) |
| 070504 INCOMESHIELD | Benefits_Income (L6), Premium_Income (L8), Nomination_Elite (L9) |
| 070504 POSCOVER | Nomination_POS (L10) |
| 070504 AZP1/AZP2 | Nomination_AZP (L15) |
| 070504 other schemes (ASTRO, BMW, POSMESRA, FWPA, others) | Nomination_Elite (L9) if `NOMINEE_PRINT>0`; no benefits/premium pages |
| 070521 | Nomination (L11)/Diagnosis (L12) only (Benefits excluded) |
| 070406 | Benefits excluded; Nomination (L11)/Diagnosis (L12) if flags; plus Template4 landscape (L16) when `PROMO_AMT>0` |
| 100908 (AMC) | Insured_AMC (L13), Diagnosis_AMC (L14) if `DIAGNOSIS_PRINT>0`; Benefits excluded; Nomination (L11) if flag |
| all other products | Benefits (L4) if `BENEFIT_PRINT>0` (always 1), Nomination (L11), Diagnosis (L12) |

### 5.3 PolicyDocument2 - which accounting pages print
| Condition | Page |
|---|---|
| 070406 | Domestic_ATC / Overseas_ATC by `JOURNEY_AREA`; debit note from Template4; service tax invoice from Template4 (Java: ATC page for all products, D3) |
| 070504 + POSMESRA or FWPA (+`showComplaint=Y`) | `ComplaintRedress_SCB` |
| BMW / BMW1 | no debit note (`PREM=0`) |
| SNATCHPA-M / SNATCHPA-A / POSMESRA | no blank before debit note (Q1) |
| 070414 | `ServiceTaxInvoice_PA4A` instead of `ServiceTaxInvoice` |
| POSMESRA | tax-invoice blank only on odd pages (Q5) |
| 070504 + ELITESHIELD / INCOMESHIELD | `PDS_Elite` / `PDS_Income` bands are dead in Jasper (Java prints them) |
| `copyType` = BC | debit note suppressed; `copyType` IO/ID + `SHOW_PDPA_NOTICE_IND=Y` + `checklistCopy=N` -> PDPA |

### 5.4 Complete code lists used in conditions
* PIDMSetB products: 070202, 070203, 070204, 070207, 070233, 090102, 090202, 070413 (+ `PIDM_PRINT=Y`).
* PIDMSetA products (needs `PIDM_PRINT=Y` and `pCertInd=Y`): 070501, 070503, 105002, 070510, 089426, 089429, 089434, 089437, 070302, 070504.
* "Group A" Coverage-only products: 089426, 089434, 070302, 070510, 089432, 089437.
* Schemes appearing in conditions: ASTRO, BMW, BMW1, POSCOVER, AZP1, AZP2, POSMESRA, FWPA, ELITESHIELD, INCOMESHIELD, SNATCHPA-M, SNATCHPA-A.
* Where Java encodes them: booleans at `S12:179-187` (`is070504`, `is070412`, `is070406`, `is070414`, `isBmwScheme`, `isPosMesraScheme`, `isFwpaScheme`, `isPosCoverWithDependants`), `S12:1524-1534`, `1551-1554`, `1595-1599`, and the literals in `executeMergedLandscape`.

---------------------------------------------------------------------------------------------------
## 6. Request parameters: Jasper parameters vs Java `filterCondition` keys

Java reads the request map `requestBody.getFilterCondition()` (`B12.filterOf:154`) and uses `getOrDefault`. All values are `String`. Keys and defaults (`S12:132-154`):

| Jasper parameter (class) | Present in | Java key | Java default | Difference / note |
|---|---|---|---|---|
| `pContractId` (Integer) | PD, PD2, Landscape | `pContractId` | `""` | Java passes a String, handlers convert with `asNum` (Integer if numeric else raw string). `Template12LandscapeDispatchHandler` does `Long.parseLong` (NumberFormatException on non-numeric, test `fetchDispatch_nonNumericContractId_throwsNumberFormatException`) |
| `pPolicyNo` | PD, PD2 | `pPolicyNo` | `""` | echoed only |
| `versionNo` (Integer) | PD (not PD2/Landscape) | `versionNo` | `"1"` | used for PrintSeriesFooter and QRSection |
| `companyParamType` | PD, PD2 | `companyParamType` | `""` | used for company reg no / signature lookups |
| `checklistCopy` | PD, PD2, Landscape | `checklistCopy` | `"N"` | Jasper has no default (`.equalsIgnoreCase` on null would NPE). Jasper uses it for the watermark and for `FN_SHOW_STAX_INV_IND(...,'WIP' vs 'OCP')`; Java passes it to the REST functions (`fnShowStaxInvInd`, `fnGetEinvoicePrintPolInd`) and PDPA test |
| `showCoverNote` | PD, PD2 | `showCoverNote` | `"N"` | echoed (`RP/pshowCoverNote`); no band uses it in these three reports |
| `showComplaint` | PD, PD2 | `showComplaint` | `"Y"` | Jasper null would NPE; Java defaults to Y -> complaint page prints by default |
| `showScheduleStd` | PD, PD2 | `showScheduleStd` | `"Y"` | same |
| `showDebitNote` | PD, PD2 | `showDebitNote` | `"N"` | |
| `showTaxInvoice` | PD, PD2 | `showTaxInvoice` | `"N"` | |
| `showPaymentInfo` | PD2 input control only | `showPaymentInfo` | `"N"` | echoed (`RP/pshowPaymentInfo`); not a jrxml parameter |
| `printSeries` | PD | `printSeries` | `""` | echoed `RP/pPrintSeries` |
| `copyType` | PD2 (CIPrint too) | `copyType` | `""` | Jasper reads `$P{copyType}.equalsIgnoreCase("BC")` (NPE if null); Java `""` |
| `pCertInd` | PD, PD2 | `pCertInd` | `""` | gates PIDMSetA (needs "Y") |
| `ShowOutPrintCovLetter` | PD | `ShowOutPrintCovLetter` (capital S) | `"N"` | exact key spelling matters; not echoed (D15) |
| `pType` | PD, PD2 | `pType` | `""` | echoed `RP/pType` |
| `pFirstPageNumber` (String in PD, Integer in PD2/Landscape) | PD, PD2, Landscape | `pFirstPageNumber` | `""` (the unregistered PD2 service defaulted to `"1"`) | echoed as string `RP/pFirstPageNumber` (XSD models it as string, Pline.xsd header v0.1.4) |
| `pMstPolicyNo`, `pContractIdList` | PD | same names | `""` | read but dropped (setters commented out, D15) |
| `pCertNoFrom`, `pCertNoTo` | PD | same | `""` | used for BMW cert range; set on rp but not serialised (D15) |
| `pTaxInvNo` | ServiceTaxInvoice(PA4A) only (derived from row in Jasper) | `pTaxInvNo` | `""` | request value used instead of row value (D14) |
| `pGstTaxInvoiceNo` | Tax/ProductInfo subreports (derived from row) | `pGstTaxInvoiceNo` | `""` | request value; ignored by handlers that match on contract only |
| `issueDate`, `signIssueDate` | PD/PD2/Landscape (computed from `$F{ISSUE_DATE}` in the main) | **not a request key** | derived from `UWGE_VI_POL_REPORT_PARAM` | Java derives them itself (`S12:177`) |
| `SUBREPORT_DIR*`, `SUBREPORT_CR/GEN/BRN/PDPA/NOTE` | all | dropped | - | repository paths are irrelevant in Java |
| `pShowGst`, `pShowSst`, `pShowTran` (BigDecimal), `pShowGstLabel`, `pShowSstLabel` | PD2 | not request keys | - | Java sets `rp.pshowGst/pshowSst/pshowTran/pShowGstLabel/pShowSstLabel` from the primary DTO (`B12:3586-3597`); only the label pair is whitelisted |
| `pTemplate` | PolicyDocument.xml input control | dropped | - | not a jrxml parameter |
| `showEinvEmail`, `pShowPDPANotice` | PD2 input controls | dropped | - | PDPA shown by `SHOW_PDPA_NOTICE_IND` from DB |
| `certNo` | none in Jasper PD2 | read only by the unregistered `Template12PolicyDocument2Service:67` | `""` | the merged service hard-codes `""` (D4) |

Parameters that reach the response (`PlineReportParameters` whitelist): `pchecklistCopy, pcopyType, pCertInd, pType, pFirstPageNumber, pshowCoverNote, pshowTaxInvoice, pshowPaymentInfo, showComplaint, showDebitNote, showScheduleStd, pPrintSeries, pShowGstLabel, pShowSstLabel, pProductCode, pAtceInd` plus the computed `dateValue, parentName, planDescp, planType, pmtDueDate, polRef, policyholderName, premDueAftPtv, premDueRoundedAftPtv, premiumPrint, printPoip, prlInd, promoAmt, provInd, psi` (section 3).

---------------------------------------------------------------------------------------------------
## 7. Git history of the Template12 module

Command used: `git log --format='%h|%ad|%an|%s' --date=short -- <Template12 / PlineT12 / PlineTemplate12 / T12 DTO, payload, handler, service and test paths>` on branch `feature/retrofit-develop-updated` (the working tree has unrelated uncommitted edits in `RejectRiskHandler`, `RejectRiskService`, `application.properties`; nothing Template12-specific is uncommitted). Authors are shown exactly as git records them; this document does **not** assign any of these commits to the person using it.

### 7.1 Commits touching Template12 files (19, newest first)
| Hash | Date | Author (git) | Subject | Template12 files touched (shortstat on those paths) |
|---|---|---|---|---|
| `e74bc0d5c` | 2026-10-07 | Shinge | OPUSM-44159 Added report, sub report name in resposne | 3 services: `Template12PolicyDocumentService`, `...PolicyDocument2Service`, `...LandscapeService` (+7/-4): `PlineReportNames.tag(...)` call |
| `74ecf94f3` | 2026-10-06 | extern.neethunivas-subrahmannian_neethu | OPUSM-44159 companyType and issuedate changes | `Template12PolicyDocumentBaseService` (+1/-1) |
| `edbef120c` | 2026-10-06 | Shinge | OPUSM-44159 multiple reportData changes | `Template12PolicyDocumentBaseService` + its test (+13/-13) |
| `c336f76fa` | 2026-10-01 | Shinge | OPUSM-44159 fixed sonar issues | 28 files, mostly handler tests (+428/-201) |
| `db7588897` | 2026-09-30 | Karunakar | OPUSM-44159 Add test coverage for plines.service/client/config/utils and SharedReport | 4 test files: `Template12{PolicyDocument,PolicyDocument2,PolicyDocumentBase,Landscape}ServiceTest` (+4165) |
| `3f2157b28` | 2026-09-29 | Shaw | OPUSM-44159 Test cases added for plines | 89 files (+11945/-57): all `PlineT12*`/`PlineTemplate12*`/`Template12*` handler, DTO and payload tests; also source edits to `Template12PolicyDocumentBaseService`/`...Service` (first appearance of `toLandscapeReportData`, `fwpaResilient*`, per `git log -S`) |
| `4cd1cad2e` | 2026-08-31 | Shinge | OPUSM-42246 Working code for pline | `Template12PolicyDocumentBaseService` (+24/-4) |
| `b4e588bd4` | 2026-08-11 | GS | final_T12_code | `...BaseService`, `...Service` (+13/-5) |
| `7399d2fc1` | 2026-08-10 | Karunakar | OPUSM-42045 mergenonnullvalues changes | `...BaseService` (+17/-11) |
| `8d4ecc991` | 2026-08-10 | Karunakar | T12 conflicts resolved commit | `...BaseService`, `...Service` (3 files, +1019/-875) |
| `bb82a9c47` | 2026-08-10 | Karunakar | My local changes - t12 | `PlineTemplate12NoteHandler`, PD2 service, base, service (+2112/-679) |
| `6961abbf9` | 2026-08-08 | Shinge | Align Pline requestpayload types with XSD and add schema validation test | `...LandscapeService`, `...BaseService`, `...Service` (+692/-634); also `Pline.xsd` (+13/-5) and `PlineXsdValidationTest` (+119) |
| `bc42674a4` | 2026-08-03 | Amarnath Yadav | Template12 date fixed | `Template12PolicyDocumentService` (+27/-18) |
| `c0c6d449a` | 2026-08-03 | Amarnath Yadav | Added Template12 dynamic merging | the 3 services (+791/-2): PD2 and Landscape merged into PolicyDocument (`executeMergedLandscape` first appears here), `@Service` commented on the two old beans |
| `2edd97981` | 2026-08-03 | Karunakar | ChecklistCopy update | `Template12PolicyDocumentBaseService` |
| `d4575fa5b` | 2026-08-03 | Shinge | Code merged with XSD changes for Template 12 and Template4 landscape | 88 files (+8977/-133): all Template12 handlers/DTOs/payload classes, `Template12PolicyDocumentBaseService` (first appearance); adds `Pline.xsd` (1397 lines) and `Common.xsd` (748 lines) |
| `ea6b0f1e8` | 2026-07-30 | Shinge | Added new XSD Changes in AutoRenewalDocument and crutinyRejectedChecklist | `Template12PolicyDocumentHandler`, `Template12PolicyDocumentDto`, `Template12PolicyDocumentDetails` |
| `604000915` | 2026-07-22 | Shinge | ScrutinyRejectedChecklist API with new XSD changes | `Template12PolicyDocument2Service` |
| `d4c069135` | 2026-07-16 | Shinge | AutoRenewalDocument with XSD changes | first appearance of `Template12PolicyDocumentService`, `...PolicyDocument2Service`, `...LandscapeService`, `Template12LandscapeDispatchHandler/Dto/Details` (bundled with an AutoRenewal change) |

Pline.xsd history in this repo (`src/test/resources/xsd/plines`): two commits - `d4575fa5b` (files created for Template12 + Template4 landscape) and `6961abbf9` (renames to XML-legal leaf names: `COLUMN_COUNT`->`Columncount`, `EMAIL_BCC`->`Emailbcc`, `sumTotalPremium_MEDIX`->`sumTotalPremiumMEDIX`, `TableHeader_COUNT`->`TableHeaderCOUNT`, `policyLetterInd`->`Policyletterind`; adds `pShowExpiryLetter`, `pLaunchDate`).
`requireNoConflict` was added to `PlinePolicyDocumentMainHandler` in `d4575fa5b` (177 inserted lines in that file, 3 occurrences of the helper in that diff) - i.e. the ORA-01427 emulation arrived with the first Template12 drop. (`git log -S` also lists `c0c6d449a`, `3f2157b28` and the EndtSchedule commit `50cf5fb55`, but their diffs of this file contain no `requireNoConflict`; the hits are in other files.)

### 7.2 Themes derived from the history (inferred)
1. **Foundation (16 Jul - 3 Aug):** Template12 services created, bundled into an AutoRenewal commit; then one large drop (88 files) adds every `PlineT12*`/`PlineTemplate12*` handler, DTO and payload class together with the first Pline.xsd.
2. **Merge into one bean (3 Aug):** "dynamic merging" collapses PD2 and Landscape into PolicyDocument and adds the merge helpers; date fixes (`bc42674a4`); checklistCopy plumbing (`2edd97981`).
3. **Schema alignment (8 Aug):** payload types aligned to XSD (BigDecimal/LocalDateTime instead of strings), XSD-validation test introduced.
4. **Stabilisation (10 - 11 Aug, 31 Aug):** conflict resolution, `mergeNonNullFields` (OPUSM-42045), "final_T12_code", OPUSM-42246 working code.
5. **Test and quality drive (29 Sep - 1 Oct, OPUSM-44159):** 89-file test drop, service-level tests (4165 lines), Sonar fixes; the 29 Sep commit also brings `toLandscapeReportData` (one `reportData` per Landscape row) and the resilient FWPA fetch into the source.
6. **Response-shape changes (6 - 7 Oct, OPUSM-44159):** small follow-ups on multiple `reportData`, `companyType`/issue date, and `mainReportName`/`subReportName` tagging (`PlineReportNames`).

---------------------------------------------------------------------------------------------------
## 8. Unit tests for Template12

Framework: JUnit 5 + Mockito (`@ExtendWith(MockitoExtension.class)`, `@MockitoSettings(strictness = LENIENT)`, e.g. `Template12PolicyDocumentServiceTest:46-47`). All handlers/`MongoTemplate` are mocks; there is no embedded Mongo and no Jasper comparison.

### 8.1 Test classes and counts (`@Test`/`@ParameterizedTest` annotations counted with grep; a parameterised test counts once)
| Layer | Classes | Tests | Notes |
|---|---|---|---|
| Service | `Template12PolicyDocumentServiceTest` (34), `Template12PolicyDocumentBaseServiceTest` (135), `Template12PolicyDocument2ServiceTest` (33), `Template12LandscapeServiceTest` (28) | **230** | the last two test the **unregistered** classes directly |
| Handlers (Template12-named) | 30 classes: `PlineT12*HandlerTest`, `PlineTemplate12*HandlerTest` (6), `Template12LandscapeDispatchHandlerTest` (6 tests), `Template12PolicyDocumentHandlerTest` (3 tests) | **231** | e.g. `PlineT12PolicyDocumentHandlerTest` 13, `PlineT12PremiumDebitNoteHandlerTest` 14, `PlineT12ParticularOfBenefitsLandscapeHandlerTest` 14, `PlineT12ParticularOfInsuredLandscapeAspHandlerTest` 14, `PlineTemplate12NoteHandlerTest` 17 |
| CIPrint handlers used by Template12 | `PlineCIPrint{Primary,Atc,AstroPos,Azp,BmwPrimary,Elite,Fwpa,Pa4a,Posm}HandlerTest` + `...DispatchHandlerTest` + `PlineCIPrintHandlerTest` | **117** (11 classes) | the last two are shared with other templates |
| DTO tests | 28 Template12/T12 DTO tests; 10 CIPrint DTO tests | **183 + 98** | bean-style tests |
| Payload (`*Details`) tests | 29 T12 + 10 CIPrint | **168 + 65** | |
| XSD | `PlineXsdValidationTest` (4 Template12-named tests, lines 105-120) | 4 | each calls `assertValidAgainstPlineXsd(MINIMAL_VALID_XML)` on a constant `<transactions><transaction><reportParameters/><transactionDetails/></transaction></transactions>`; **it does not run any service**, so it only proves the schema loads |

Total about 1,090 tests across these groups (a raw grep over `Template12*`, `PlineT12*`, `PlineTemplate12*`, `PlineCIPrint*` test files reports 1,128 occurrences, which includes two Template18/Apm handler classes that are not Template12).

### 8.2 What the service tests cover (from test method names)
* `Template12PolicyDocumentServiceTest` (34): null filter defaults; primary document null/populated (dependants, issue date); default CIPrint, ATC (070406), ASTRO, BMW (per-certificate loop), POSCOVER with/without dependants (landscape POS rows), AZP, POSM (`polRef`), FWPA (graceful null from the resilient fetch), Elite fallback, PA4A per-row nested calls; `ShowOutPrintCovLetter` N/Y; complaint redress (non-504 product), SCB (504 non-excluded), `showComplaint=N`; landscape dispatch null (CI_LANDSCAPE_PRT) and ASP band 1; `parentName` precedence PolicyDocument > Landscape > PolicyDocument2; PD2 journey area DOM/OVERSEAS, T12 debit note with master policy, T4 debit note (070406), BMW skip, GST tax invoice, PA4A service tax invoice with QR, transitional invoice, PDPA, PDS income/elite; error handling (a swallowed primary-handler exception; a landscape-dispatch exception that propagates).
* `Template12PolicyDocument2ServiceTest` (33): each PD2 branch separately (DOM/non-DOM, POSMESRA/FWPA SCB, debit note T4/T12 x master policy, BMW skip, copyType BC skip, Blank 2/4, GST, PA4A STAX with/without rows, T4 STAX, MST branch, PL/PLSub, transitional on/off, PDPA IO/ID/other/checklist=Y, PDS income/elite only).
* `Template12LandscapeServiceTest` (28): dispatch null, all-null defaults, bands 1-3 (ASP, cutoff N vs Y), band 4 (and exclusion for 070504), bands 5-15 including exclusions (band 9 for POSCOVER/AZP, band 12 for 100908), band 16 (Template4 non-null/null/promo zero), `toLocalDateTime`/`toBigDecimal` helpers.
* `Template12PolicyDocumentBaseServiceTest` (135): one or more tests per `add*` helper (DTO -> `PlineTransDet` mapping, null/empty rows, swallowed handler exceptions), merge helpers (`mergeSingleTransDet`, `setEndorsementDetails`), and the FWPA resilient aggregation chain (`addTemplate12CIPrintFwpa_*`).
* Handler tests (example `PlineT12PolicyDocumentHandlerTest`): empty vs non-empty aggregation result; `fnGetEinvoicePrintPolInd` throws -> "N"; `fnShowStaxInvInd` variants; product WIP-then-SB, scheme posted-then-WIP, journey area posted-only, tax OR, einv consent posted-then-WIP-then-N, promo sums - all by capturing the `Aggregation` pipeline and asserting stage shapes against a mocked `MongoTemplate`.

### 8.3 Gaps (evidence-based unless marked inferred)
1. **No direct test of `PlinePolicyDocumentMainHandler.fetchTemplate12PolicyDocumentMain`.** `grep fetchTemplate12PolicyDocumentMain` in tests hits only the two service tests (as a mock). The ORA-01427 emulation (`requireNoConflict`, `distinctValueOrConflict`) has no test of its own.
2. **Handler tests never touch real data**; they assert pipeline structure through a mocked `MongoTemplate`, so semantic differences such as D7 (promo join), D8 (first version row) and D9 (raw COV_SEQ_REF sort) cannot be caught.
3. **XSD validation is a stub** (constant XML) - no test marshals the `Transactions` returned by `Template12PolicyDocumentService.execute` and validates it against `Pline.xsd`. Together with D16 (repo XSD v0.1.6 vs code v0.1.10) this is the biggest risk for the "XSD changes" deliverable.
4. **Tests encode known deviations as expected behaviour**: `execute_pd2JourneyAreaDom_usesDomesticAtcBranch` / `...DefaultsToOverseas` (D3), `execute_pd2PdsIncomeAndEliteBranches_noCrash` and the PD2 PDS tests (D2), `execute_particularOfInsuredAndNominationRowsPopulated_...` (D4).
5. (inferred from method names) no test for the date reformat pass (`reformatDatesToDdMmYyyy`, `toDdMmYyyy`), for `PlineReportNames.tag` on Template12, for the PAGE_NUMBER gaps (D1), for the D13 ordering or for D14 (QR filter).
6. (inferred) no contract/regression test compares Java output with a Jasper-rendered reference; none exists in the repo.
7. `PlineT12ParticularOfInsuredPosHandler` is tested (6 tests) but unused in production (D12).
8. I did not run JaCoCo, so line-coverage percentages are unknown.

---------------------------------------------------------------------------------------------------
## 9. Interview questions (25) with grounded answers

Level: about 1.6 years of experience - expect "why" and "what would you do" questions, not only "what". Each answer cites where to look so you can back it up. Use the exact class/method names; interviewers like precise nouns.

**Q1. In two minutes, explain what the Template12 reports are and how they are built.**
Answer: Template12 is the Personal Accident policy-schedule family. In Jasper there are three mains: PolicyDocument (CI print variants chosen by product/scheme), PolicyDocument2 (debit note, GST/SST/transitional invoices, PDPA, ATC page) and Landscape (16 landscape "particulars" bands). Each main runs a one-row `dual` query (product, scheme, flags) and each band calls a subreport guarded by a `printWhenExpression`. In Java, the single bean `Template12PolicyDocumentService` (`@Service(".../Template12/PolicyDocument")`) replays that decision tree: fetch the primary row, evaluate the same booleans, call one `add*` helper per subreport (about 95 in `Template12PolicyDocumentBaseService`), each helper calls a Mongo-backed handler and maps fields into `PlineTransDet`/`reportParameters`/`endorsementDetails`, then `buildTransactions` wraps it in the Pline.xsd shape.
Follow-ups: How many subreports? (87 subreport `.jrxml.data` files plus the three mains in the folder, ~30 Template12-named handler classes plus shared Common/Generic handlers.) Where is the decision tree? (`S12.execute:130-556`, `addCIPrintDefaultNestedSubreports:1507`, `executeMergedLandscape:801`.)
Common mistakes: saying "Java renders the PDF" (it only produces data XML); forgetting PD2 and Landscape are merged into the PolicyDocument bean.

**Q2. Why are PolicyDocument2 and Landscape no longer separate beans?**
Answer: Commit `c0c6d449a` ("Added Template12 dynamic merging", 3 Aug) commented out `@Service` on `Template12PolicyDocument2Service` and `Template12LandscapeService` and copied their bodies into `executeMergedPolicyDocument2` / `executeMergedLandscape`. The class javadoc explains the single `PlineReportData` envelope ("one report-data-per-document shape"). The old classes are kept for reference with a warning not to re-add `@Service` without removing the call or every band would run twice (`Template12PolicyDocument2Service:24-44`).
Follow-ups: Consequence for callers? (only the PolicyDocument report name resolves in `CommonController`'s bean map; D17.) Trade-off? (one big 2235-line class, but one response.)
Common mistakes: claiming the merge changed the Jasper reports - it did not; it is a Java-side design.

**Q3. How does Java pick between CIPrint, CIPrint_ATC, CIPrint_PA4A and the seven 070504 variants?**
Answer: Same ternaries as `PolicyDocument_jrxml.data`: bands P2-P4. Booleans at `S12:179-187`. Non-070504/070414 products -> `070406 ? CIPrint_ATC : CIPrint` (`S12:247-280`); 070504 -> the scheme chain ASTRO, BMW/BMW1, POSCOVER, AZP1/AZP2, POSMESRA, FWPA, otherwise Elite (jrxml lines 330-336; Java lines 285-381, each guarded by `is070504 && schemeX`); 070414 -> PA4A (`S12:393`). Each variant has its own primary handler and its own nested subreports.
Follow-ups: Why are the seven 070504 branches independent `if`s and not `else if`? (mutually exclusive by construction; Elite is the fallback computed as "none of the others", `S12:374-375`.)
Common mistakes: forgetting `BMW1` and `AZP2` aliases; forgetting that POSCOVER and ASTRO share one handler (`addTemplate12CIPrintAstroPos`).

**Q4. What is a "dispatch-only" or stub subreport, and why do you still emit a row?**
Answer: Many subreports (ComplaintRedress, Domestic/Overseas_ATC, ScheduleOfBenefits_ASP/AMC/PA4A, ImportantNotes, SSTNote...) are `SELECT 1 FROM DUAL` plus static text; Jasper's only data decision is "does the band print". The handlers return a constant (`Collections.singletonList(1)` etc., no DB call) and `addMergedDispatchOnlyRow` merges an empty transDet so downstream still sees that the band fired (`B12:494-506`). Static wording is the template's job, not the data service's.
Follow-ups: Why merge instead of append? (appending would multiply rows - see Q5.)
Common mistakes: thinking these handlers query Mongo; thinking the empty row carries meaning beyond "band printed".

**Q5. Explain `addSingleTransDet` versus `mergeSingleTransDet`.**
Answer: `addSingleTransDet` appends a new `transDet` (used for genuine multi-row queries: coverage lines, insured rows, clauses). `mergeSingleTransDet` copies the non-null scalar fields of a one-row result onto every existing `transDet` (reflection in `mergeNonNullFields`, `B12:212-258`), special-casing lists (risk, benefit print, nominee print, stax) and a guard so a blank `eInvStatus` never overwrites a real one. Reason: dozens of single-row bands would otherwise create dozens of near-empty transDets (commit `c0c6d449a`, `7399d2fc1`).
Follow-ups: Risk? (last non-null writer wins; fields set by one band can be overwritten by another - D10.) Why reflection? (hundreds of leaves on `PlineTransDet`.)
Common mistakes: not mentioning the placeholder lists (`riskList` empty, `sobList`/`nomineeList` default-populated) that the merge code relies on.

**Q6. Why does each Landscape row get its own `reportData`, and what is the risk?**
Answer: `toLandscapeReportData` (`S12:2077-2091`) wraps one `transDet` in a fresh `PlineReportData` that shares branch/company info by reference and the one `endorsementDetails`; the idea is that each landscape row is an independent print unit. The code comment itself warns that the consuming renderer "has not been confirmed" to treat `reportData[]` entries that way (`S12:159-165`) and that `endorsementDetails` is shared (so "last call wins").
Follow-ups: How did you validate? (unit tests check row counts, e.g. `execute_poscoverSchemeWithDependants_addsLandscapePosReportDataRows`; no consumer-side test exists.) What would you ask the report team? (whether one `reportData` = one page group.)
Common mistakes: calling it a final design; ignoring the shared `endorsementDetails` object.

**Q7. Explain the ORA-01427 emulation.**
Answer: In Oracle the PolicyDocument query uses scalar sub-selects like `select distinct product_config_code from (wip ... union all sb ...)`; if WIP and posted tables hold different values the query errors (ORA-01427, single-row subquery returns more than one row). The Java primary fetch preserves that: it reads both sides independently and `requireNoConflict` throws `IllegalStateException` when they differ; `distinctValueOrConflict` also checks inside one collection (`PlinePolicyDocumentMainHandler:1463,1474,1543-1585`). `addTemplate12PDPrimaryDocument` is not wrapped in try/catch, so the request fails fast.
Follow-ups: Why not just prefer WIP? (silently choosing would hide bad data and change product behaviour without sign-off; the class javadoc "CRITICAL FACT #1".) Is it consistent? (No: PD2 and Landscape use `$ifNull` precedence - D6.)
Common mistakes: believing the other two primaries also throw; not knowing it is untested (section 8.3).

**Q8. Some Jasper conditions use `$V{PAGE_NUMBER}`. What did you do?**
Answer: Those are render-time and cannot be known in a data service. They exist to start a document on an odd page for duplex printing (blank filler pages) and to print footers only after page 2. Java implements the data half of the condition and drops the page-parity half: `Blank_NoMargin` (`S12:1669-1676`), PD2 blank before debit note (omitted entirely), POSMESRA blanks, PrintSeriesFooter call 2. These are documented in comments and listed in D1; the renderer must supply page-parity logic.
Follow-ups: What is the risk? (extra or missing blank pages on even pages.) Alternative? (return a `blank` marker only and let the renderer decide.)
Common mistakes: saying they were "ignored by mistake" - they were consciously documented; or claiming parity can be computed from row counts.

**Q9. What are the `false &&` bands in PolicyDocument2 and how did you treat them?**
Answer: Five PD2 bands have `printWhenExpression` starting with `false &&` (PDS_Elite, PDS_Income, PDPA from CommonReport, NominationForm, a Blank after GST) - disabled upstream, so Jasper never prints them. I did not implement PDPA/NominationForm/Blank, but PDS_Elite/PDS_Income are wired in Java (`S12:770-775`) "per explicit instruction" and tested, so Java prints a PDS marker for ELITESHIELD/INCOMESHIELD that Jasper does not (D2). If asked, say it is a documented deviation and should be confirmed with the product owner.
Follow-ups: How did you find them? (grep of `printWhenExpression` per band in the .data file.) Why does the element-level condition not matter? (a false band condition suppresses the whole band.)
Common mistakes: reading only the inner element condition (which is true) and concluding the report prints.

**Q10. How did you map Jasper fields to XSD elements? What if the XSD has no matching leaf?**
Answer: Fields were mapped by name and role to `transDet` leaves, structured lists, or other groups: list-like content goes to `provisoList/proviso`, `nomineeList/nominee`, `diagnosisList/diagnosis`, `coverList/cover`, `sobList/benefits/benefit`, `staxList/staxPcts`; party/address data to `prntDetails/clientInfo|compInfo|branchInfo|agentInfo|taxClientInfo`; "report-wide" values (PARENT_NAME, PLAN_DESCP, PSI, DATE_VALUE, PROV_IND, PREMIUM_PRINT...) to `reportParameters`; refno/source/partner type/contract id to `endorsementDetails`. If no leaf exists the field is left unmapped with a comment (e.g. `addPrimaryDocumentT12:536-538`: "showGstInvInd... no matching PlineTransDet leaf found - left unmapped rather than guessing") - or the XSD is extended (Pline.xsd version notes v0.1.3-0.1.6).
Follow-ups: Example of an XML-illegal name? (`COLUMN_COUNT` -> `Columncount`, commit `6961abbf9`.) What happens if you emit an element not in the XSD? (validation error `cvc-complex-type.2.4.a`, see comment in `Template12PolicyDocument2Service:292-296` about `pSchemeName`.)
Common mistakes: putting everything in `transDet`; forgetting `@JsonIncludeProperties` silently drops unlisted `reportParameters` (D15).

**Q11. What is `CI_LANDSCAPE_PRT`?**
Answer: A row in `CMGE_CODEREL_DET` (`CAT_CODE='GPA_SCHEME'`, `DETAIL_CODE='CI_LANDSCAPE_PRT'`, `CD_VALUE='Y'`, `HEADER_CODE=scheme`). If the policy's scheme is flagged, the Landscape main query returns no row (outer `where not exists`) so no landscape page prints. `Template12LandscapeDispatchHandler.fetchDispatch` reproduces it with a `$lookup` + `$match` on `$size == 0` and returns `null`; `S12:805-808` then skips all 16 bands.
Follow-ups: Test? (`execute_landscapeDispatchNull_excludedByCiLandscapePrt_noExtraReportData`.) Why `null` rather than empty? (a null dispatch means "report suppressed".)
Common mistakes: placing the exclusion in each band instead of once.

**Q12. What is `ASP_CUTOFF_IND` and why does 070412 behave differently?**
Answer: It is 'Y' when the report-param issue date is on/after `SAPM_SYS_CONSTANTS(ASP_CONFIG_DATE, ASP_CUTOFF_DATE)`. For 070412, policies issued before the cutoff use the old landscape ASP pages (Landscape L1-L3, `Note`, `PerilsClauses` + `ComplaintRedress` from the main); policies on/after the cutoff print the new "potrait" ASP pages inside CIPrint (`ParticularOfInsured/Premium/Sponsor/NominationPotrait_ASP`, `PerilsClauses_ASP`, `Blank_NoMargin`, `ComplaintRedress_NoMargin`). Java computes it in each of the three primaries (date comparison on real `Date`s, `MainHandler:1498-1508`).
Follow-ups: Where is the boundary condition? (`>=` : `!issueDate.before(cutoff)`.) What if cutoff row missing? ('N'.)
Common mistakes: confusing it with the PIDM cutoff (`PIDM_CUTOFF_POL`, `EFF_DATE > ...`).

**Q13. Oracle functions like `FN_GET_COMP_BRN_NUMBER` - how are they handled without Oracle?**
Answer: They are called through `PlineUtilsClient` (REST) from the handler, not translated to Mongo: `getCompBrnNumber`, `fnGetEinvoicePrintPolInd`, `fnShowStaxInvInd`, `fnRppmGetSignatureAndRemarks`, `fnGetBenefitHdr/Det`. Failures are caught and defaulted ("N" for the two indicators in `PlineT12PolicyDocumentHandler`).
Follow-ups: Consequence? (an outage turns e-invoice/service-tax pages OFF silently, since "N" disables the band.) Mitigation? (alerting on the `log.warn`.)
Common mistakes: saying the functions were re-implemented.

**Q14. How do you deal with WIP vs posted tables and `checklistCopy`?**
Answer: Jasper SQL queries both `WIP_*` and posted tables (union), because a draft (checklist copy) lives in WIP while an issued policy lives in posted tables. Java repeats the pattern per handler: query both collections and apply a precedence rule (posted VERSION_NO=1 first, else WIP not present in `OCP_POLICY_CONTRACTS`; WIP rows have no `VERSION_NO`). `checklistCopy` itself is only used for the watermark (renderer) and for the REST functions (`fnShowStaxInvInd(contractId, checklistCopy)`) and the PDPA test; the PolicyDocument primary accepts it but never branches on it (class javadoc).
Follow-ups: Where does it differ across handlers? (D6, D19.)
Common mistakes: assuming `checklistCopy=Y` switches every query to WIP.

**Q15. What was the `COV_SEQ_REF` sort issue?**
Answer: Jasper sorts by `cast(COV_SEQ_REF as decimal(15,2))` so "2" comes before "10". In Mongo the field may be stored as string; sorting it raw would put "10" first. Most handlers convert with `$toDouble` before `$sort` (`..LandscapeAsp:72-73`, `PlineParticularOfInsuredHandler:145-146`, Coverage handler comment lines 66-76). Three handlers still sort the raw field (`...InsuredLandscapeAmc:38`, `...DiagnosisLandscape:41`, `...DiagnosisLandscapeAmc:38`) - a known gap to fix (D9).
Follow-ups: How would you fix? (add `$addFields {COV_SEQ_NUM: {$toDouble ...}}` then `$sort`; add a unit test with "10" and "2".) Null handling? (Oracle puts NULLs last on ASC, Mongo first - inferred.)
Common mistakes: sorting after `$project` on a renamed field; `$toDouble` failing on empty strings (use `$convert` with `onError`).

**Q16. Why can PROMO_AMT differ from Jasper?**
Answer: Jasper joins `wip_UWPL_RISK_PERSON` to `wip_uwge_cover` on contract id only, so each cover amount repeats per insured person; Java checks that at least one risk person exists and sums covers once (PD2 handler applies version filters; the Landscape handler does not). Hence the amounts (and the `070406 && promo>0` band) can differ for multi-person policies (D7). It is an example where reproducing a Jasper SQL quirk needs a business decision: replicate the multiplication or treat it as a Jasper bug.
Follow-ups: How would you prove it? (run the Oracle query for a contract with 3 insureds and compare.) 
Common mistakes: assuming the Mongo `$sum` matches Oracle because the column names match.

**Q17. How did you test, and what is missing?**
Answer: About 1,090 unit tests: 230 service tests (the four Template12 service classes), 231 Template12-named handler tests, 117 CIPrint handler tests, plus DTO and payload tests. Service tests mock every handler (lenient Mockito) and assert which branch fired and how fields were mapped; handler tests capture the aggregation pipeline from a mocked `MongoTemplate`. Gaps: no test for `fetchTemplate12PolicyDocumentMain`/ORA-01427 emulation, no test against real data or Jasper output, and the XSD test validates a constant XML, not the service output (section 8.3).
Follow-ups: What would you add first? (a marshal-and-validate test of `S12.execute` against Pline.xsd; an embedded-Mongo test for the primary fetch.) Do tests assert wrong behaviour? (a few encode deviations D2/D3/D4.)
Common mistakes: quoting coverage percentages (not measured); saying the tests compare to Jasper.

**Q18. What were the XSD changes and how are they checked?**
Answer: Pline.xsd (+ Common.xsd) was introduced in `d4575fa5b` for Template12 and Template4 landscape: root `transactions/transaction/{reportParameters, transactionDetails/headerTransaction/reportTransaction/reportData/{endorsementDetails, prntDetails/{...,transDetList/transDet}}}`; Pline-specific extension types (`plineCoverType`, `plineBenefitType`, `plineProvisoType`, `plineNomineeType`, `plineDiagnosisType`...); later versions v0.1.3 (prntDetails nesting fix), v0.1.4 (report parameters added, `permanentDisablement` moved), v0.1.5, v0.1.6 (see the header comment). `6961abbf9` renamed illegal names and added `PlineXsdValidationTest`. Runtime output classes use `@JacksonXmlProperty` names matching the XSD.
Follow-ups: What is the current gap? (repo copy v0.1.6 vs code references v0.1.10 `mainReportName`, D16; the XSD test does not run services.) How would you enforce compliance? (CI test that serialises a sample response and validates.)
Common mistakes: believing the XSD in `src/test/resources` is the production schema.

**Q19. Why is the PolicyDocument2 handler an aggregation anchored on `WIP_UWGE_POLICY_CONTRACTS` with `$limit 1`?**
Answer: Oracle's `select ... from dual` returns exactly one row regardless of data. In Mongo a pipeline needs a collection; the code starts from any document (`$limit 1`), then `$lookup`s each source table by contract id, so one aggregation produces the whole row (`PlineT12PolicyDocumentHandler:72-110`). Side effect: if the WIP collection is empty the aggregation yields nothing and the DTO is empty (`results.isEmpty() ? new PlineT12PolicyDocumentDto()`).
Follow-ups: Alternatives? (several small `findOne` calls or a dedicated one-document "dual" collection.) Performance? (`allowDiskUse`, `maxTime` 600 s are set.)
Common mistakes: saying `$limit 1` filters by contract (it does not; the contract id is injected with `$addFields`).

**Q20. A new scheme for product 070504 is launched. What do you change?**
Answer: First decide whether Jasper has a new CIPrint variant. If yes: add the scheme to the `subreportExpression` ternary in Jasper, then in Java add a boolean and an `if (is070504 && isNewScheme)` block in `S12.execute`, update the Elite fallback exclusion list (`S12:374-375`), create primary + nested handlers, mapper `addTemplate12CIPrintX` in `B12`, DTO/payload classes, XSD leaves if new fields, and tests (service branch test + handler tests). Also check Landscape bands (L5-L10, L15) and PD2 complaint/SCB rules for the scheme.
Follow-ups: Where could the scheme silently fall to Elite? (the fallback is "none of the others".) 
Common mistakes: forgetting the Elite exclusion list or the Landscape nomination band conditions.

**Q21. Why do the Java request defaults differ from Jasper?**
Answer: Jasper parameters have no defaults, and expressions like `$P{showComplaint}.equalsIgnoreCase("Y")` would NPE on null; the Java service uses `getOrDefault` with business-neutral defaults (`showComplaint="Y"`, `showScheduleStd="Y"`, `showDebitNote="N"`, `showTaxInvoice="N"`, `checklistCopy="N"`, `ShowOutPrintCovLetter="N"`, `versionNo="1"`) at `S12:132-154`. The unregistered PD2 service used `pFirstPageNumber="1"` while PolicyDocument uses `""` (section 6).
Follow-ups: Which keys are case-sensitive? (`ShowOutPrintCovLetter` with a capital S.) Which values are dropped? (D15.)
Common mistakes: assuming the default for `showComplaint` is N.

**Q22. Performance - what would worry you?**
Answer: Many small aggregations per request (the default CIPrint path alone can call 10+ handlers; BMW and PA4A loop per row; FWPA does several lookups); each aggregation uses `allowDiskUse` and a 600-second `maxTime`; REST calls to `PlineUtilsClient` sit on the request thread. Improvements: parallelise independent handlers, cache static lookups (`SAPM_SYS_CONSTANTS`, `CMGE_CODE`), index by `CONTRACT_ID`/`VERSION_NO`, avoid fetching the same table twice (the default CIPrint already reuses `PIDM_PRINT`, `PROV_IND` from its primary row instead of re-querying - see javadocs "reused here instead of re-fetching").
Follow-ups: Evidence? (comments on reuse, e.g. `S12:1692-1700`.) I have no timing data - say so.
Common mistakes: quoting numbers you did not measure.

**Q23. Why does every `add*` swallow exceptions?**
Answer: To keep one failing subreport from losing the whole policy document, each helper logs a `warn` and returns neutral data. The cost: silent partial output, different from Jasper which fails the report. Exceptions to the rule are intentional: the PolicyDocument primary fetch (conflict) and the Landscape dispatch fetch (test `execute_landscapeDispatchHandlerThrows_notWrappedInTryCatch_propagates`).
Follow-ups: How would you surface partial failures? (add a `warnings` section or metrics counter.) 
Common mistakes: defending it as always correct; it hides data problems (D18).

**Q24. Dates: why `toLocalDateTime` and the reflective reformat at the end?**
Answer: DTO dates arrive as strings (some with `T00:00:00`), the payload now holds `LocalDateTime` (XSD `xs:dateTime`). `toLocalDateTime` strips a trailing time and parses (`B12:366-381`). Then `S12` walks every String field whose name contains "date" or "dob" and rewrites it to `dd/MM/yyyy` (`S12:2105-2230`), plus `reportParameters` dates (`S12:549-551`) - written locally because the base class is under a no-touch constraint (comment `S12:544-548`). Jasper passes `dd-MMM-yyyy` strings to children, so there are three representations (D11).
Follow-ups: Failure mode? (unparseable string becomes null silently.) Time zones? (not handled; dates are date-only.)
Common mistakes: saying the XSD fixes the display format - `xs:dateTime` has no display pattern.

**Q25. QA says "Landscape page X is missing for contract N". How do you debug?**
Answer: (1) Check the dispatch: log `Template12LandscapeDispatchHandler contractId=` and whether it returned null (CI_LANDSCAPE_PRT for the scheme). (2) Check product code and scheme as the dispatch derives them (they may differ from PolicyDocument's - D19). (3) Check the flag for that band (`benefitPrint`/`nomineePrint`/`diagnosisPrint`, `aspCutoffInd`) and the exact Java `if` (`S12:842-1479`) against the Jasper condition (section 1.3). (4) Run the band handler's pipeline in Compass for the contract (collection names in section 3.4). (5) Remember empty handler results still return gracefully. (6) Confirm the consumer renders multiple `reportData` entries.
Follow-ups: Where are logs? (`log.info("Template12LandscapeDispatchHandler contractId={}")`, `log.warn` per failing handler.) 
Common mistakes: debugging the renderer first; forgetting the nomination/diagnosis flags look only at the first version row (D8).

---------------------------------------------------------------------------------------------------
## 10. Uncertainties and open points

1. **Authoritative Pline.xsd.** The repo copy is v0.1.6 but the code mentions v0.1.10 (`mainReportName`, `subReportName`). I could not see the newer XSD, so XSD paths in section 3 are validated only against v0.1.6 plus the Java `@JacksonXmlProperty` names I did not exhaustively read.
2. **JasperServer behaviour of band vs element `printWhenExpression`.** I applied the standard JasperReports rule (a band that is not printed hides its elements). I did not render a report.
3. **Compiled `PolicyDocument2.jasper`.** I only string-searched it for `*.jrxml` names; the readable jrxml was treated as authoritative. If the deployed report differs from the dump, D2-D4 may change.
4. **Whether PD2's `ParticularOfInsured*/Nomination*/InstallmentDetails` calls come from an older layout.** No evidence in the dump either way (D4).
5. **Consumer expectations.** Unknown whether the downstream renderer treats multiple `reportData` entries as independent print units and how it implements page parity (D1, D10).
6. **Mongo field types.** Whether `COV_SEQ_REF`/`SORT_SEQ` are stored as strings (drives D9) was not checked against a live database; the Coverage handler comment says "some collections store SORT_SEQ as a string".
7. **Orphan list in 1.4** is based on subreport references inside the Template12 folder only; other folders (Template4, Template23, Generic, CommonReport) may reference some of these files.
8. **Template4 Landscape (L16).** Only the dispatch fields are emitted in this service; how Template4's own bands are produced for a Template12 request was not traced.
9. **Test counts** come from grep of `@Test`/`@ParameterizedTest` annotations; I did not run the suite, so pass/fail status and coverage are unverified.
10. **Original Oracle data semantics** (D7 multiplication, D8 first-row version check, D9 null ordering) are inferred from SQL/pipeline text, not from comparing outputs on real contracts.
11. **Commit authorship.** The log lists authors under their git names; I deliberately did not map them to the reader.
12. **`CommonController` fall-through** for report names that no longer have a Plines bean (`.../PolicyDocument2`, `.../Landscape`) was not traced.
13. **CIPrint `issueDate` format:** Jasper passes `dd-MMM-yyyy` but the CIPrint SQL calls `to_date($P{issueDate},'dd/MM/yyyy')`; I did not verify how Oracle copes, nor how the Java REST call formats the date it sends (Java passes `yyyy-MM-dd` after `formatDate`, `S12:177`).


---------------------------------------------------------------------------------------------------
