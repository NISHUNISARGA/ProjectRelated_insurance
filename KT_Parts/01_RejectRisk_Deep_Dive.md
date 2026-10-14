# 01 - RejectRisk Deep Dive (Jasper to Java/MongoDB migration)

Scope: the Plines "RejectRisk" listing report, migrated from JasperReports + Oracle to a Spring Boot service that reads MongoDB and returns an XSD-shaped payload.

Conventions used in this file:
- EVIDENCE = I read it in a file or git output. The path/line is given.
- INFERENCE = my reasoning from evidence. It is labelled.
- Nothing here was run. No Maven, no DB. All "test results" are from reading code.
- Secrets: the working tree contains a hard-coded MongoDB URI with a password in `application.properties`. It is NOT copied here (see 4.4).

Short names used for paths:
- `REPO` = `C:\Users\EPGRVFY\Documents\Plines_Base_code\New_branch\2\opus-documentprintdata-service`
- `JRXML` = `C:\Users\EPGRVFY\Documents\jasper_dumps\jasper_02072026\resources\reports\AGIC\UW\Plines\RejectRisk\RejectRisk_files\RejectRisk_jrxml.data`
- Java: `REPO\src\main\java\com\agcs\fo\opus\plines\...` (handler = `documentHandler\RejectRiskHandler.java`, service = `service\RejectRiskService.java`, DTO = `documentdto\PlineRejectRiskDto.java`)
- XSD: `REPO\src\test\resources\xsd\plines\Pline.xsd`

---

## a) Jasper report anatomy

### a.1 Files in the dump
| File | What it is |
|---|---|
| `RejectRisk\RejectRisk.xml` | JasperServer report-unit descriptor. Data source `/datasources/AGICDEV`. Defines 4 input controls only: `pCreatedDateFrom`, `pCreatedDateTo`, `pReportTitle`, `reportName` (all optional, `mandatory=false`). |
| `RejectRisk_files\RejectRisk_jrxml.data` | The real JRXML (780 lines, plain XML). |
| `RejectRisk_report_hierarchy.csv` | `Main Report,Sub Report` / `RejectRisk,ReportHeader`. So there is exactly one subreport. |
| `...\AGIC\UW\Generic\ReportHeader.jrxml.data` (+ `.xml`) | The generic ReportHeader subreport. |

Note (EVIDENCE): the JRXML declares 8 filter parameters, but only 4 have input controls. `pCustomerName`, `pIdNo`, `pRejType`, `pReasonCode` have no UI control in the dump. INFERENCE: they are passed by the caller (OPUS) and not by a user in the JasperServer prompt screen.

### a.2 Page setup (JRXML line 2)
- name `rejectRisk`, language `groovy`
- pageWidth 900, pageHeight 850, columnWidth 860, margins 20 on all sides
- `whenNoDataType="AllSectionsNoDetail"` - with no rows, title, group header and group footer still print, only the detail band is skipped. This is why "*** No Data Found ***" can be shown.
- `whenResourceMissingType="Key"`

### a.3 Parameters (JRXML lines 51-76)
| Parameter | Class | Used for |
|---|---|---|
| `SUBREPORT_DIR` | String | default `repo:/reports/AGIC/UW/Plines/RejectRisk/` (passed to subreport, not otherwise used) |
| `pCreatedDateFrom` | `java.util.Date` | SQL date filter + header echo |
| `pCreatedDateTo` | `java.util.Date` | SQL date filter + header echo |
| `pCustomerName` | String | SQL filter |
| `pIdNo` | String | SQL filter |
| `pRejType` | String | SQL filter |
| `pReasonCode` | String | SQL filter |
| `SUBREPORT_DC` | String | default `repo:/reports/AGIC/UW/Generic/` (folder of ReportHeader) |
| `pReportTitle` | String | passed to ReportHeader |
| `reportName` | String | passed to ReportHeader |

### a.4 The full Oracle query (JRXML lines 77-114; no secrets in it)
```sql
SELECT ROWNUM ROWNUMBER, REJECT_RISK.*
FROM (
  SELECT
    CpgeViPartnersAll.NAME_EXT   CUSTOMER_NAME,
    CpgeViPartnersAll.ID_TYPE1   ID_TYPE_1,
    CpgeViPartnersAll.ID_VALUE1  ID_NO_1,
    CpgeViPartnersAll.ID_TYPE2   ID_TYPE_2,
    CpgeViPartnersAll.ID_VALUE2  ID_NO_2,
    CmuwPlReferRisk.REJ_TYPE     REJECT_RISK_TYPE_CODE,
    (SELECT CODE_DESC FROM CMGE_CODE WHERE CAT_CODE='PLREJ_RISK_TYPE'  AND CODE_CD = CmuwPlReferRisk.REJ_TYPE)       REJECT_RISK_TYPE,
    CmuwPlReferRisk.REASON_CODE  REASON_CODE,
    (SELECT CODE_DESC FROM CMGE_CODE WHERE CAT_CODE='PLREJ_REASON'     AND CODE_CD = CmuwPlReferRisk.REASON_CODE)    REASON_CODE_DESCRIPTION,
    CmuwPlReferRisk.REMARKS      REMARKS,
    CmuwPlReferRisk.CREATED_BY   CREATED_BY,
    CmuwPlReferRisk.CREATED_DATE CREATED_DATE,
    CmuwPlReferRisk.REVOKE_IND   REVOKE_IND,
    CmuwPlReferRisk.REVOKED_DURATION REVOKED_DURATION,
    (SELECT CODE_DESC FROM CMGE_CODE WHERE CAT_CODE='PLREJ_REVOKE_DUR' AND CODE_CD = CmuwPlReferRisk.REVOKED_DURATION) REVOKED_DURATION_DESCRIPTION,
    CmuwPlReferRisk.REVOKE_BY    REVOKE_BY,
    CmuwPlReferRisk.REVOKE_DATE  REVOKE_DATE,
    CmuwPlReferRisk.UPDATED_BY   UPDATED_BY,
    CmuwPlReferRisk.UPDATED_DATE UPDATED_DATE
  FROM CMUW_PL_REFER_RISK CmuwPlReferRisk, CPGE_VI_PARTNERS_ALL CpgeViPartnersAll
  WHERE (CmuwPlReferRisk.CP_PART_ID      = CpgeViPartnersAll.PART_ID(+))
    AND (CmuwPlReferRisk.CP_PART_VERSION = CpgeViPartnersAll.VERSION(+))
    AND TRUNC(CmuwPlReferRisk.CREATED_DATE) BETWEEN $P{pCreatedDateFrom} AND $P{pCreatedDateTo}
    AND (CpgeViPartnersAll.NAME_EXT = $P{pCustomerName} OR $P{pCustomerName} IS NULL)
    AND ((CpgeViPartnersAll.ID_VALUE1 = $P{pIdNo} OR CpgeViPartnersAll.ID_VALUE2 = $P{pIdNo}) OR $P{pIdNo} IS NULL)
    AND (CmuwPlReferRisk.REJ_TYPE     = $P{pRejType}     OR $P{pRejType}     IS NULL)
    AND (CmuwPlReferRisk.REASON_CODE  = $P{pReasonCode}  OR $P{pReasonCode}  IS NULL)
  ORDER BY CpgeViPartnersAll.NAME_EXT
) REJECT_RISK
```

Key points about this SQL (EVIDENCE for the text, INFERENCE for the consequences):
1. The date predicate is not optional. `x BETWEEN NULL AND NULL` is UNKNOWN in Oracle, so zero rows. Only-from or only-to also gives zero rows. So the original report effectively needs BOTH dates, even though the input controls say `mandatory=false`.
2. `ROWNUM` is applied by the outer SELECT after the inner `ORDER BY`, so it is a clean 1..N in name order.
3. The three `(SELECT CODE_DESC ...)` scalar subqueries do not filter any rows. If CMGE_CODE returned two rows for one key, Oracle would raise ORA-01427 (INFERENCE: standard Oracle behaviour).
4. Partner filters are in WHERE, not in the join ON. So when `pCustomerName` or `pIdNo` is given, a row with no partner is dropped (the outer join becomes effectively inner). When they are null, no-partner rows stay.

### a.5 Fields (JRXML lines 115-134)
| Field | Class |
|---|---|
| ROWNUMBER | `java.math.BigDecimal` |
| CUSTOMER_NAME, ID_TYPE_1, ID_NO_1, ID_TYPE_2, ID_NO_2 | String |
| REJECT_RISK_TYPE_CODE, REJECT_RISK_TYPE | String |
| REASON_CODE, REASON_CODE_DESCRIPTION, REMARKS, CREATED_BY | String |
| CREATED_DATE | `java.sql.Timestamp` |
| REVOKE_IND, REVOKED_DURATION, REVOKED_DURATION_DESCRIPTION, REVOKE_BY | String |
| REVOKE_DATE, UPDATED_DATE | `java.sql.Timestamp` |
| UPDATED_BY | String |

### a.6 Variables, group, styles
- Variable `SlNo` (Integer, reset/increment on group `GroupByBranch`, expression `$V{REPORT_COUNT}`). It is declared but I found no element that prints it.
- Group `GroupByBranch`, `isStartNewPage="true"`, no group expression (so it is one group for the whole report). It carries the column header (as the group header) and the "No Data" message (as the group footer).
- Detail "NO" column prints `$V{GroupByBranch_COUNT}`, NOT the SQL ROWNUMBER. So in the original, ROWNUMBER is only used as a "row exists" guard.
- Styles `table`, `table_TH/CH/TD`, `table 1...` are template leftovers. Only `autoPolicyColumnHeaderStyle` (Arial 8) is used (by the "ID NO 1" header cell, line 188).
- No conditional styles, no page header, no page footer, no column header/footer band, no summary band.

### a.7 Bands
| Band | Height | Content |
|---|---|---|
| title | 60 | Logo image `repo:/images/AGIC/logo.gif` at x=715,y=0,145x30; `ReportHeader.jrxml` subreport at x=1,y=0, 555x60, using `REPORT_CONNECTION` (its own SQL) |
| groupHeader (GroupByBranch) | 70 | y=20: label "Created Date From", from-date (dd/MM/yyyy, "NA" if null), "to", to-date (dd/MM/yyyy, "NA" if null). y=50: 17 bold 6pt column header boxes |
| detail | 20 | 17 cells, Arial 5pt, each with `printWhenExpression ROWNUMBER != null` |
| groupFooter | 80 | "*** No Data Found ***" printed when `REPORT_COUNT==0 || ROWNUMBER==null` |

Column order on the page (left to right, by x position; EVIDENCE from reportElement x/width):

| # | Header text | x | w | Detail expression | Format / null rule |
|---|---|---|---|---|---|
| 1 | NO | 0 | 20 | `$V{GroupByBranch_COUNT}` | right aligned |
| 2 | CUSTOMER NAME | 20 | 60 | CUSTOMER_NAME | "NA" if null/empty |
| 3 | ID TYPE 1 | 80 | 30 | ID_TYPE_1 | "NA" |
| 4 | ID NO 1 | 110 | 40 | ID_NO_1 | "NA" |
| 5 | ID TYPE 2 | 150 | 30 | ID_TYPE_2 | "NA" |
| 6 | ID NO 2 | 180 | 40 | ID_NO_2 | "NA" |
| 7 | REJECT RISK TYPE | 220 | 100 | REJECT_RISK_TYPE (description, not the code) | "NA" |
| 8 | REASON CODE / DESCRIPTION | 320 | 120 | `REASON_CODE + " / " + REASON_CODE_DESCRIPTION` | "NA" if code null/empty. If description is null you get "R1 / null" (INFERENCE: Groovy string concat) |
| 9 | REMARKS | 440 | 80 | REMARKS | "NA" |
| 10 | CREATED BY | 520 | 50 | CREATED_BY | "NA" |
| 11 | CREATED DATE | 570 | 35 | CREATED_DATE | pattern dd/MM/yyyy, blank when null |
| 12 | REVOKE INDICATOR | 605 | 38 | REVOKE_IND | "NA" |
| 13 | REVOKE DURATION | 643 | 40 | REVOKED_DURATION_DESCRIPTION | "NA" |
| 14 | REVOKED BY | 683 | 50 | REVOKE_BY | "NA" |
| 15 | REVOKED DATE | 733 | 35 | REVOKE_DATE | dd/MM/yyyy, blank when null (no "NA") |
| 16 | UPDATED BY | 768 | 50 | UPDATED_BY | "NA" |
| 17 | UPDATED DATE | 818 | 35 | UPDATED_DATE | dd/MM/yyyy, blank when null (no "NA") |

Not printed anywhere: REJECT_RISK_TYPE_CODE, REVOKED_DURATION (the raw code). The Java side still sends them because the XSD has leaves for them.

### a.8 The ReportHeader subreport and DATA_CLASS
`Generic\ReportHeader.jrxml.data` (EVIDENCE):
```sql
SELECT DATA_CLASS FROM CUSTOMER.RPGE_RPT_TEMPLATE
WHERE DOC_TYPE = 'OPUS_RPT'
  AND UPPER(TEMPLATE_NAME) = UPPER($P{reportName} || '/' || $P{pReportTitle})
  AND UPPER(DESCRIPTION)   = UPPER($P{pReportTitle})
```
- Parameters: `pReportTitle`, `reportName`. One field `DATA_CLASS` (String).
- Detail band (height 60) prints three lines: bold "ALLIANZ GENERAL INSURANCE COMPANY (MALAYSIA) BERHAD"; "Report Name : " + pReportTitle (html markup, `<` escaped); "Data Classification Level : " + DATA_CLASS + " ( 0 - Public, 1 - Internal, 2 - Confidential-Named, 3 - Strictly Confidential )".
- If the SELECT returns no row, this subreport prints nothing (INFERENCE: no whenNoData setting, detail band only).
- DATA_CLASS is a per-report setting, not per-row. It is a security classification label.

---

## b) Line-by-line translation: Oracle SQL -> Mongo pipeline -> DTO -> payload -> XSD

Pipeline order in the current working tree (`RejectRiskHandler.fetchRejectRisks`, includes the uncommitted edit):

| Step | Stage | Handler lines (working tree) |
|---|---|---|
| 1 | `$match` CREATED_DATE (only if a bound was supplied) | 52-58 |
| 2 | `$match` REJ_TYPE / REASON_CODE (only if supplied) | 61-67 |
| 3 | `$lookup` CPGE_VI_PARTNERS_ALL (let pid, pver; `$expr $and $eq`; `$limit 1`) as CUST_ROW | 70-78 |
| 4 | `$addFields CUST = $arrayElemAt(CUST_ROW, 0)` | 79-80 |
| 5 | `$match CUST.NAME_EXT == customerName` (if supplied) | 83-85 |
| 6 | `$match $or CUST.ID_VALUE1 / CUST.ID_VALUE2 == idNo` (if supplied) | 86-90 |
| 7 | `$lookup` CMGE_CODE PLREJ_RISK_TYPE on REJ_TYPE -> RRT | 93-102 |
| 8 | `$lookup` CMGE_CODE PLREJ_REASON on REASON_CODE -> RSN | 105-114 |
| 9 | `$lookup` CMGE_CODE PLREJ_REVOKE_DUR on REVOKED_DURATION -> RVK | 117-126 |
| 10 | `$project` to DTO field names | 128-147 |
| 11 | `$setWindowFields` sortBy customerName, `rowNumber = $documentNumber` | 151-154 |
| 12 | `$sort` customerName asc | 156 |

### b.1 Column-level map
| Oracle expression | Mongo stage / source | DTO field (`PlineRejectRiskDto`) | Service setter (`RejectRiskService`) | XSD leaf and type |
|---|---|---|---|---|
| `ROWNUM ROWNUMBER` | `$setWindowFields ... $documentNumber` -> `rowNumber` (stage 11) | `Integer rowNumber` | `transDet.setRownumber(toBigDecimal(...))` (l.67) | `transDet/rownumber` xs:decimal (Pline.xsd l.1197) |
| `NAME_EXT CUSTOMER_NAME` | `$project customerName = $CUST.NAME_EXT` | `String customerName` | `setCustomerName` (l.68) | `transDet/customerName` xs:string (l.833) |
| `ID_TYPE1` | `idType = $CUST.ID_TYPE1` | `idType` | `setIdType1` (l.69) | `idType1` xs:string (l.958) |
| `ID_VALUE1` | `idNo = $CUST.ID_VALUE1` | `idNo` | `setIdNo1` (l.70) | `idNo1` xs:string (l.955) |
| `ID_TYPE2` | `idType2 = $CUST.ID_TYPE2` | `idType2` | `setIdType2` (l.71) | `idType2` xs:string (l.959) |
| `ID_VALUE2` | `idNo2 = $CUST.ID_VALUE2` | `idNo2` | `setIdNo2` (l.72) | `idNo2` xs:string (l.956) |
| `REJ_TYPE REJECT_RISK_TYPE_CODE` | `rejectRiskTypeCode = $REJ_TYPE` | `rejectRiskTypeCode` | `risk.setRejectRiskTypeCode` (l.75) | `riskList/risk/rejectRiskTypeCode` xs:string (l.1346) |
| scalar subquery PLREJ_RISK_TYPE | stage 7, `rejectRiskType = $RRT.CODE_DESC` | `rejectRiskType` | `risk.setRejectRiskType` (l.76) | `risk/rejectRiskType` xs:string (l.1345) |
| `REASON_CODE` | `reasonCode = $REASON_CODE` | `reasonCode` | `transDet.setReasoncode` (l.79; note lower-case "c") | `reasoncode` xs:string (l.1154) |
| scalar subquery PLREJ_REASON | stage 8, `reasonCodeDescription = $RSN.CODE_DESC` | `reasonCodeDescription` | `setReasonCodeDescription` (l.80) | `reasonCodeDescription` xs:string (l.1155) |
| `REMARKS` | `remarks = $REMARKS` | `remarks` | `companyDesc.setRemarks` (l.95) - not on transDet | `transDetList/companyType/companyDesc/remarks` xs:string (l.454) |
| `CREATED_BY` | `createdBy` | `createdBy` | `setCreatedBy` (l.81) | `createdBy` xs:string (l.826) |
| `CREATED_DATE` | `createdDate = $CREATED_DATE` (BSON Date) | `String createdDate` | `setCreatedDate(toLocalDateTime(...))` (l.82) | `createdDate` xs:dateTime (l.827) |
| `REVOKE_IND` | `revokeInd` | `revokeInd` | `setRevokeInd` (l.83) | `revokeInd` xs:string (l.1188) |
| `REVOKED_DURATION` | `revokedDuration` | `revokedDuration` | `setRevokedDuration` (l.84) | `revokedDuration` xs:string (l.1186) |
| scalar subquery PLREJ_REVOKE_DUR | stage 9, `revokedDurationDescription = $RVK.CODE_DESC` | same | `setRevokedDurationDescription` (l.85) | `revokedDurationDescription` xs:string (l.1187) |
| `REVOKE_BY` | `revokeBy` | `revokeBy` | `setRevokeBy` (l.86) | `revokeBy` xs:string (l.1184) |
| `REVOKE_DATE` | `revokeDate` | `String revokeDate` | `setRevokeDate(toLocalDateTime)` (l.87) | `revokeDate` xs:dateTime (l.1185) |
| `UPDATED_BY` | `updatedBy` | `updatedBy` | `setUpdatedby` (l.88; lower-case "b") | `updatedby` xs:string (l.1285) |
| `UPDATED_DATE` | `updatedDate` | `String updatedDate` | `setUpdateddate(toLocalDateTime)` (l.89) | `updateddate` xs:dateTime (l.1286) |
| (subreport) DATA_CLASS | `PlineReportHeaderHandler` on RPGE_RPT_TEMPLATE | n/a (`PlineReportHeaderDto.dataClass`) | `reportData.setDataClass` on each row (l.108) + extra header block (l.116) | `reportData/dataClass` xs:string (l.583) |
| params `pCreatedDateFrom/To` | echoed, not queried | n/a | `rp.setPCreatedDateFrom/To(toLocalDateTime(...))` (l.125-126) | NOT found in the XSD copy under src/test (see 4.5) |
| params `pReportTitle`, `reportName` | echoed | n/a | `rp.setPReportTitle/ReportName` (l.123-124, "" if null) | reportParameters children |

Payload shape per DB row (service lines 64-111): `reportData{dataClass, prntDetails{transDetList{transDet{...fields..., riskList{risk}}, companyType{companyDesc{remarks}}}}}`. After the loop, one extra reportData: `mainReportName=RejectRisk, subReportName=ReportHeader, dataClass` (via `PlineReportBlocks.reportHeader`, `PlineReportBlocks.java` l.18). `PlineReportNames.tag(...)` fills `mainReportName` on the row blocks (the unit test asserts row[0].mainReportName = "RejectRisk" and subReportName = null).

### b.2 Semantics, one by one
| Topic | Oracle | Mongo implementation | Match? |
|---|---|---|---|
| Outer join `(+)` | LEFT OUTER JOIN, all partner matches (duplicates rows if several) | `$lookup` with sub-pipeline + `$limit 1` + `$arrayElemAt 0`: left join, first match only, no sort so "first" is arbitrary | Same if the partner view is unique on (PART_ID, VERSION). INFERENCE: probably unique, not verified. |
| Join keys | `CP_PART_ID = PART_ID` and `CP_PART_VERSION = VERSION` | `$expr $and [$eq PART_ID pid, $eq VERSION pver]` | Same. Types must match in BSON (number vs string) - not verifiable here. |
| Filter on partner columns | in WHERE (kills no-partner rows when a value is given) | `$match` after the lookup | Same. |
| `TRUNC(CREATED_DATE) BETWEEN from AND to` | whole days, inclusive both ends | `CREATED_DATE >= fromStartOfDay(UTC)` and `< (toStartOfDay(UTC) + 1 day)` | Same for UTC-stored data. See 4.2 for the time zone risk. |
| NULL date bound | zero rows (UNKNOWN) | bound ignored (open range) after the fix | DIFFERENT (see c). |
| `= $P{x} OR $P{x} IS NULL` | empty string counts as NULL in Oracle | `!= null && !isEmpty()` | Same (nice parity). A string of spaces counts as a value in both. |
| Exact match / case | `=` is exact and case sensitive by default (INFERENCE: unless NLS_COMP/NLS_SORT is changed) | Mongo string equality is exact and case sensitive | Same. No trim, no collation. |
| `ID_VALUE1 = x OR ID_VALUE2 = x` | yes | `$or` on both | Same. |
| `ORDER BY NAME_EXT` | ASC, NULLs LAST | `$sort customerName:1`, null/missing FIRST | DIFFERENT (see b.3). |
| `ROWNUM` after ORDER BY | 1..N in sorted order | `$setWindowFields` with `$documentNumber` sorted by customerName | Same numbers, but follows the Mongo null ordering, and ties are not deterministic. |
| Scalar subquery for descriptions | error if 2+ rows, null if none | `$lookup` + `$limit 1`, null if none | Mostly same. Mongo hides duplicate-code data problems. |
| `UPPER()` matching in ReportHeader | `UPPER(a)=UPPER(b)` | regex `^\Q..\E$` with option `i` (`PlineReportHeaderHandler`) | Same. `Pattern.quote` makes it a literal match. |
| `ROWNUM` type | BigDecimal | Integer then BigDecimal | Same value. |

### b.3 NULL ordering (verified by reasoning from known behaviours)
- Oracle: `ORDER BY x ASC` puts NULLs last by default.
- MongoDB: BSON sort order is MinKey < Null/missing < Numbers < Strings < ... So ascending puts null AND missing fields FIRST.
- Consequence here: customers that did not join to a partner (no CP_PART_ID match) or have a null NAME_EXT appear at the TOP of the Mongo output and take ROWNUM 1..k. In Oracle they are at the bottom. Reports for date ranges with such rows will look reordered and numbered differently. The "NO" column in Jasper uses its own counter, so the visible number depends on how the renderer uses `rownumber` (INFERENCE).
- Unit test `fetchRejectRisks_finalSortStage_sortsByCustomerNameAscending` locks the plain `{customerName:1}` sort, so the bug is not caught.
- Recommended fix (two options):
  1. In the pipeline: add `{ $addFields: { _nullName: { $cond: [ { $eq: [ { $ifNull: ["$customerName", null] }, null ] }, 1, 0 ] } } }`, then `sortBy`/`$sort` = `{ _nullName: 1, customerName: 1, createdDate: 1, _id: 1 }`. Use the same key set in `$setWindowFields.sortBy`.
  2. In Java (simpler): sort the list in the service with `Comparator.comparing(PlineRejectRiskDto::getCustomerName, Comparator.nullsLast(Comparator.naturalOrder()))` and assign rowNumber `i+1` there. This also removes the dependency on `$setWindowFields` (needs MongoDB 5.0+; INFERENCE on version, other handlers like `CardPrintingHandler` already use it).
- Also add a tie-breaker. Oracle ties are in arbitrary order too, but a stable order helps testing.
- Binary vs linguistic order: Mongo default is binary UTF-8, Oracle default `NLS_SORT` is binary. For plain ASCII names they match (uppercase before lowercase in both). INFERENCE, not verified for the actual DB settings.

---

## c) Behaviour per filter combination

Assume rows exist. "Oracle" is the original JRXML behaviour. "Java (HEAD)" is the last commit (before the uncommitted edit). "Java (working tree)" is with the developer's edit.

| # | Scenario | Oracle (original) | Java HEAD | Java working tree |
|---|---|---|---|---|
| 1 | No dates at all | 0 rows, "NA to NA", "No Data Found" | `$match CREATED_DATE {$gte:null,$lte:null}` -> in Mongo, comparison with null only matches null/missing CREATED_DATE, so 0 real rows (same visible result by accident). `toDate(null)` is null and `Date.from` is not called. | NO date filter -> ALL rows returned (differs from Oracle; unbounded) |
| 2 | Only `pCreatedDateFrom` | 0 rows | 0 rows (the `$lte:null` half fails) | rows with CREATED_DATE >= from-day 00:00 UTC (open ended) |
| 3 | Only `pCreatedDateTo` | 0 rows | 0 rows | rows with CREATED_DATE < (to-day + 1) 00:00 UTC |
| 4 | from = to (same day D) | all rows created on day D | `>= D 00:00` and `<= D 00:00` -> only rows exactly at midnight (BUG: whole-day missed) | `>= D 00:00Z` and `< D+1 00:00Z` -> whole UTC day D (matches Oracle if data is stored as UTC wall-clock) |
| 5 | from < to range | whole days inclusive | last day cut at 00:00 (BUG) | whole days inclusive |
| 6 | Dates + `pCustomerName` only | exact name match; no-partner rows dropped | same filter stage | same |
| 7 | Dates + name that does not exist | 0 rows | 0 rows | 0 rows |
| 8 | Dates + `pIdNo` matching ID_VALUE1 | match | match via `$or` | match |
| 9 | Dates + `pIdNo` matching ID_VALUE2 | match | match (`$or` second branch) | match |
| 10 | `pIdNo` + `pCustomerName` both | AND of both | two separate `$match` stages = AND | AND |
| 11 | `pRejType` only (+dates) | filter on REJ_TYPE | `$match REJ_TYPE` | same |
| 12 | `pReasonCode` only (+dates) | filter on REASON_CODE | `$match REASON_CODE` | same |
| 13 | Both rejType and reasonCode | AND | same `$match` document (AND) | same |
| 14 | Empty strings for optional filters | treated as NULL (ignored) | `isEmpty()` -> ignored | ignored |
| 15 | Unparseable date string (e.g. "31.12.2026") | n/a (UI typed) | `DateUtils.toLocalDate` throws `IllegalArgumentException` from `toDate` -> request fails (500) | same |
| 16 | No rows after filters | title + headers + "No Data Found" | `reportData` list is EMPTY (no ReportHeader block either) | same |

Service-level extras (all versions): `pReportTitle`/`reportName` null become `""` in report parameters (l.123-124). `dataClass` is null if RPGE_RPT_TEMPLATE has no matching row (`PlineReportHeaderHandler` returns `new PlineReportHeaderDto(null)`).

---

## d) The developer's uncommitted diff, hunk by hunk

`git status` shows 3 modified files: `RejectRiskHandler.java`, `RejectRiskService.java`, `application.properties`. I treat them as in-progress fixes.

### d.0 Important finding: this is the same fix as an existing commit
Commit `cc6a1b84d` (2026-10-05, author GS, "OPUSM-41763 ASP RejectRisk/EndtSchedule/EndtDocument fixes") on branch `origin/OPUSM-41763/retrofit-develop-R2-pline-Template12-ASP-RejectRisk` contains the same handler and service changes. Its message lists: "Fix RejectRiskHandler date-range filter silently matching 0 rows when a bound was null" and "Fix RejectRiskService wrong DateUtils import package". `git diff origin/OPUSM-41763/... -- RejectRiskHandler.java` shows the working tree differs from that commit only in the wording of one comment (the service diff only shows newer lines that HEAD has and the branch does not, such as `PlineReportBlocks`). INFERENCE: the developer is re-applying (cherry-picking by hand) that fix onto `feature/retrofit-develop-updated`, plus removing a dead stage.

### d.1 Hunk 1 - date range (handler, around lines 40-58)
- Problem: HEAD built `$match {CREATED_DATE: {$gte: dateFrom, $lte: dateTo}}` always. (a) If a date is missing the value is null, and Mongo compares against null, so no real rows come back. (b) `dateTo` is the start of the day (00:00), and CREATED_DATE has a time, so `$lte dateTo` excluded almost the whole last day, and from = to returned only exact-midnight rows.
- Root cause: translating `TRUNC(x) BETWEEN a AND b` as `x BETWEEN a AND b`, and building the filter without a null check (unlike the other optional filters).
- Fix: add 1 day to `dateTo` and use `$lt` (half-open interval `[from 00:00, to+1d 00:00)`); build the document only from non-null bounds; add the stage only if at least one bound exists.
- Why it is correct: `TRUNC(x) <= to` is the same as `x < to + 1 day` and `TRUNC(x) >= from` is the same as `x >= from`. It also keeps `CREATED_DATE` as a bare field so an index can be used (the comment explains this; a `$dateTrunc` would block index use).
- Risk 1 (behaviour change): no dates now means ALL rows. Oracle returned none. This can produce a huge, unbounded response and a full collection scan. Decide with the business: either keep parity (return empty if either date is missing) or require at least one date.
- Risk 2 (time zone): `toDate` makes UTC midnight (`ZoneId.of("UTC")`, handler l.167). If CREATED_DATE values are real instants of Malaysian time (UTC+8), a record at 2026-01-05 00:30 MYT is stored as 2026-01-04T16:30Z and lands in the previous UTC day window. INFERENCE for the storage form: other handlers in this repo format CREATED_DATE with `timezone: Asia/Kuala_Lumpur` (e.g. `GenericProjectCodeReportHandler.java` l.80, `BondPolStatusReportHandler.java` l.393), which suggests values are true UTC instants, not wall-clock-as-UTC. Check with one known record before sign-off. If instants, use `ZoneId.of("Asia/Kuala_Lumpur")` in `toDate`.
- Risk 3: if `createdDateTo` arrives as a `java.util.Date` that already has a time (raw `instanceof Date`, l.165), +1 day does not give "next midnight".
- How to test: unit tests assert `$gte`/`$lt`; integration tests on a seeded collection: records at `D 00:00:00`, `D 23:59:59.999`, `D+1 00:00:00`, with from=to=D; only-from; only-to; neither.

### d.2 Hunk 2 - removed `ROW_NUMBER` `$addFields` (handler)
- Problem: HEAD added `ROW_NUMBER = 1` for every row early in the pipeline. This is a leftover; the real row number comes later from `$setWindowFields`. The `$project` stage drops every field not listed, so `ROW_NUMBER` never reached the DTO.
- Fix: delete the dead stage. Risk: very low. The same constant-1 pattern still exists in other handlers (`AutoPolicyHandler.java` l.106, `CIUploadSuccessfulHandler.java` l.104), not in scope here.
- Test: unit test `fetchRejectRisks_noOptionalFilters_noSecondMatchStageAdded` indexes stage numbers (see 4.1).

### d.3 Hunk 3 - `DateUtils` import (service, `toLocalDateTime`, around line 139)
- Problem: the service called `com.agcs.fo.opus.motor.utils.DateUtils` (the motor module) while the handler uses `com.agcs.fo.opus.plines.utils.DateUtils`. The plines copy says it is a "Plines-owned copy ... so Plines code doesn't depend on the motor package" (DateUtils.java header).
- Fix: use the plines copy in the service. Risk: low. I compared the format lists: both have the same `FORMATTERS` for ISO, dd-MM-yyyy, dd/MM/yyyy, MM/dd/yyyy, and the `EEE MMM dd HH:mm:ss z yyyy` pattern (motor file lines 39-58; plines `DateUtils.java` FORMATTERS). I did not diff the motor copy line by line.
- Note: `toLocalDateTime` swallows every exception and returns null. A parse failure will silently drop createdDate/revokeDate/updatedDate from the output.

### d.4 Hunk 4 - `application.properties` (do not commit)
- The diff adds a "#newDev" block with `spring.mongodb.uri=` containing a user name and password in clear text, and `spring.mongodb.database=OPUSAGICPRODR1`. I did not copy it.
- Problem: it is a developer's local connection setting, not a fix for RejectRisk. Committing it leaks a credential, and the repo has secret scanners (`.github/workflows/secret-scan.yml`, `secret-scan-nightly.yml`, `gitleaks.toml`).
- Action: exclude this file from the commit (`git restore` or `git add -p`), use an env var / profile file, and consider rotating that password if it was ever pushed. The database name suggests a production-like data set (INFERENCE from the name only).

### d.5 Correctness assessment of the whole diff
| Check | Result |
|---|---|
| Inclusive whole-day logic | Correct (UTC) |
| Null-safe | Correct |
| `toDate` accepted strings | `DateUtils.toLocalDate`: `yyyy-MM-dd` fast path; `dd-MM-yyyy`; `dd/MM/yyyy`; `MM/dd/yyyy` (only reached if dd/MM fails, so "03/04/2026" = 3 April); `dd-MMM-yyyy`; `dd-MMM-yy`; `EEE MMM dd HH:mm:ss z yyyy`; then ISO offset/zoned date-time. A local date-time without offset (e.g. `2026-01-05T10:00:00`) is NOT accepted -> exception. |
| Invalid but well-shaped value, e.g. `2026-13-45` | fast path throws `DateTimeParseException` (a `DateTimeException`), not `IllegalArgumentException`. Both bubble up as 500. |
| ISO with offset, e.g. `2026-01-05T00:00:00+08:00` | `OffsetDateTime.toLocalDate()` keeps the local date 2026-01-05, then treated as UTC midnight. OK only if UTC-wall-clock storage. |
| Existing unit tests | NOT updated -> many fail (see 4.1) |
| Secret in properties | Must be removed |

---

## 4) Remaining bugs and edge cases (summary list)

### 4.1 Existing tests will break with the diff (static reading, not executed)
`RejectRiskHandlerTest.java` (15 tests) was written for the old pipeline. With the working tree:
- FAIL (7): `fetchRejectRisks_dateInstances_usedDirectlyInMatch` (asserts `$lte` equals `to`; now `$lt` = to+1 day), `fetchRejectRisks_dateStrings_parsedViaDateUtils` (`$lte` is null now), `fetchRejectRisks_nullDates_matchStageHasNullBounds` (no date stage, so stage 0 is not a `$match`: NullPointerException), `fetchRejectRisks_rejTypeOnly_addsSecondMatchStage`, `fetchRejectRisks_reasonCodeOnly_addsSecondMatchStage`, `fetchRejectRisks_rejTypeAndReasonCode_bothInSameMatchStage` (they read stage index 1 as the second `$match`; now the opt-filter `$match` is stage 0), `fetchRejectRisks_customerNameAndIdNoAbsent_noExtraMatchStages` (expects exactly 1 `$match`, now 0).
- PASS (8): `..._noOptionalFilters_noSecondMatchStageAdded` (passes by accident after the stage shift), customerName, idNo, cmge lookups = 3, setWindowFields, final sort, forced lambdas, returns mapped results.
- `RejectRiskServiceTest` (4 tests) should still pass (it mocks the handler).
- On the OPUSM-41763 branch the RejectRisk tests do not exist at all (that branch has 367 test files vs 1932 on HEAD; INFERENCE: it diverged before the test-generation commits), so the fix was never run against them.

### 4.2 Time zone
See d.1 Risk 2. Also the display date: `createdDate` comes from Mongo as a `Date` and the DTO field is a `String`. INFERENCE: Spring converts it with `Date.toString()` (JVM default zone, e.g. "Mon Jan 05 08:30:00 UTC 2026") and `DateUtils` parses the last pattern. Other handlers in the repo comment on this exact format ("Handle Java Date.toString() format first - requires SimpleDateFormat due to timezone abbreviations (IST, UTC etc.)", `MarinePolScheduleHandler.java` l.447). Risks: (a) the date shown depends on the server JVM zone, (b) `DateTimeFormatter` `z` may not parse every zone abbreviation, then `toLocalDateTime` returns null silently. Better: project with `$dateToString {format:'%Y-%m-%d', date:'$CREATED_DATE', timezone:'Asia/Kuala_Lumpur'}` (pattern already used elsewhere) so the DTO gets `yyyy-MM-dd`, which hits the fast path.

### 4.3 Null-first sort and window numbering
See b.3.

### 4.4 Credentials in the working tree
See d.4.

### 4.5 Report parameters not in the XSD copy
`PlineReportParameters` has `pCreatedDateFrom/To` with a comment "XSD v0.1.9 addition (RejectRisk date-range filter)". The `Pline.xsd` under `src/test/resources` only documents versions up to v0.1.6 in its header and has no `pCreatedDate*` element in `reportParameters` (I searched case-insensitively in both XSDs). `PlineXsdValidationTest` has no RejectRisk test at all; its Template4/Template12 "generatesValidXml" tests are placeholders that validate one hand-written minimal XML string (file lines 91-118), so no real service output is validated. INFERENCE: either the test copy of the XSD is older than the production XSD, or the output of RejectRisk would fail validation. Nobody checks it today.

### 4.6 Empty result has no header block
For zero rows the payload has no `ReportHeader` reportData and no rows, while Jasper still printed the title, DATA_CLASS and the "No Data Found" footer. The unit test `execute_emptyRows_returnsEmptyReportDataList` locks this in. Confirm with the rendering team.

### 4.7 Other edge cases
- Only the first partner row is used (`$limit 1`, no sort) for name/id filters and values.
- `PlineRejectRiskDetails` (requestpayload) is a legacy class used in `PlineSubReportData`; the service does not use it. It has its own generated test.
- `dataClass` is copied onto every row block AND onto the header block (redundant).
- Types are assumed to be strings in Mongo (REJ_TYPE, REASON_CODE, CP_PART_ID, VERSION). A numeric vs string mismatch makes lookups return nothing. Not verifiable here (no sample data).
- No pagination or cap on result size.

---

## e) Data and collections

| Collection | Fields used (EVIDENCE: handler) | Purpose |
|---|---|---|
| `CMUW_PL_REFER_RISK` (constant `REFER_RISK`, handler l.30) | CREATED_DATE, REJ_TYPE, REASON_CODE, REMARKS, CREATED_BY, REVOKE_IND, REVOKED_DURATION, REVOKE_BY, REVOKE_DATE, UPDATED_BY, UPDATED_DATE, CP_PART_ID, CP_PART_VERSION | driving (base) collection |
| `CPGE_VI_PARTNERS_ALL` (a view in Oracle; a collection here) | PART_ID, VERSION (join keys), NAME_EXT, ID_TYPE1, ID_VALUE1, ID_TYPE2, ID_VALUE2 | customer name and IDs |
| `CMGE_CODE` | CAT_CODE, CODE_CD, CODE_DESC | descriptions. Cat codes: `PLREJ_RISK_TYPE` (by REJ_TYPE), `PLREJ_REASON` (by REASON_CODE), `PLREJ_REVOKE_DUR` (by REVOKED_DURATION) |
| `RPGE_RPT_TEMPLATE` (via `PlineReportHeaderHandler`, l.28) | DOC_TYPE='OPUS_RPT', TEMPLATE_NAME (= reportName + "/" + pReportTitle), DESCRIPTION (= pReportTitle), DATA_CLASS | report classification label |

Search results (EVIDENCE): in the repo, `CMUW_PL_REFER_RISK` and the `PLREJ_*` category codes appear only in `RejectRiskHandler.java`. There is no sample data, no JSON fixture, and no integration test with data. The only data-like content is in unit tests with made-up strings.

Do not confuse:
- Workmen has its own `WorkmenRejectRiskReportHandler` (different module `opus/workmen`, different DTO). Not the same report.
- The XSD comment on `reportData` mentions "ScrutinyRejection" among listing reports (Pline.xsd l.~570). I found no ScrutinyRejection class under `plines/`. Only `CaseIdScrutinyHandler` / `FormIdScrutinyHandler` exist under `dmt/`. They are unrelated to CMUW_PL_REFER_RISK.

---

## f) Git history of the RejectRisk files

`git log --all --follow` on the handler, and `git log --all` on service/DTO (EVIDENCE; dates are commit dates):

| Commit | Date | Author | Message | RejectRisk effect |
|---|---|---|---|---|
| `8d1b5374c` | 2026-07-06 | Shinge | Add Plines module implementation with handlers, services, DTOs and bug fixes | first version: handler (137 lines), DTO, `PlineRejectRiskDetails`, service (79) |
| `d4c069135` | 2026-07-16 | Shinge | AutoRenewalDocument with XSD changes | small service change (83 lines) |
| `6961abbf9` | 2026-08-08 | (n/a) | Align Pline requestpayload types with XSD and add schema validation test | touches the service |
| `0d715fbad` | 2026-09-01 | GS | OPUSM-41698 Fix RejectRisk data mapping and align with Pline.xsd v0.1.6 | handler +49/-, service +39/- |
| `94ea23d81` | 2026-09-21 | GS | ASP EndtDocument: fix ... | handler 11 lines, service 37 lines |
| `da95d245c` | 2026-09-22 | Shinge | OPUSM-44159 spring boot update and code changes | handler/service about 90 lines |
| `755c8083f` | 2026-09-24 | Shinge | OPUSM-44159 XSD fixes | handler/service about 90 lines |
| `3f2157b28` | 2026-09-29 | Shaw | OPUSM-44159 Test cases added for plines | handler -57, service +62 and the 3 test files (handler test 264 lines, DTO test 137, Details test 97) |
| `db7588897` | 2026-09-30 | Karunakar | OPUSM-44159 Add test coverage for plines.service/... | adds `RejectRiskServiceTest` |
| `cc6a1b84d` | 2026-10-05 | GS | OPUSM-41763 ASP RejectRisk/EndtSchedule/EndtDocument fixes | date filter + DateUtils import fix (only on the OPUSM-41763 branch) |
| `e74bc0d5c` | 2026-10-07 | Shinge | OPUSM-44159 Added report, sub report name in resposne | service 3 lines |
| `861f0666c` (HEAD) | 2026-10-08 | Shinge | OPUSM-44159 Added report, sub report name in listing report | service +9 (header block, `PlineReportNames.tag`), service test updated |

About the branch `origin/OPUSM-41763/retrofit-develop-R2-pline-Template12-ASP-RejectRisk`:
- Tip is `cc6a1b84d`. It is 194 commits ahead of HEAD and 1363 behind (diverged lines). The merge base starts with `4b003f899`.
- vs HEAD for RejectRisk: the handler on the branch already has the date fix and the `$setWindowFields` row number, but has NO `PlineReportBlocks`/`PlineReportNames` lines in the service (those are newer on HEAD). It has no RejectRisk tests (`git diff` shows the 4 RejectRisk test files as removed there).
- `cc6a1b84d` also changed 13 other files (ASP Endt* handlers, a new `AspCompanyNameWithGSTRegNoHandler`, etc.) - not part of this KT.
- Fix numbering mix: `OPUSM-41698` (RejectRisk mapping), `OPUSM-41763` (Template12/ASP/RejectRisk branch), `OPUSM-44159` (Spring Boot 4 upgrade + XSD + tests umbrella ticket). INFERENCE: 44159 is the main programme ticket; the others are sub-work.

---

## g) reportData-per-row design, disclaimer, performance, ideas

### g.1 Design
- Each DB row becomes one `reportData` with its own `transDetList` containing exactly one `transDet` and its own `companyType/companyDesc/remarks` (service l.64-111).
- Why (EVIDENCE from code comment l.91-93): in the XSD, `companyDesc` (which holds `remarks`) is a sibling of `transDet` inside `transDetList`, not a child of `transDet`. If all rows shared one `transDetList`, there would be one remarks slot for many rows. One list per row keeps remarks tied to its row.
- Supporting evidence: the XSD comment on `reportData` says "one row per monitoring / listing line (ScrutinyRejection, Auto_Policy, CardPrinting ...)" (Pline.xsd l.~575), and a branch named `OPUSM-44159/retrofit-develop-updated-Multitransdet-to-singletransdet` exists. INFERENCE: single transDet per list is a programme-wide decision.
- The header block is a separate `reportData` with `subReportName=ReportHeader`, mirroring the Jasper subreport (hierarchy CSV).

### g.2 The disclaimer comment (service l.27-32)
"the consuming JRXML/report-rendering side has not been confirmed to treat reportData[] as one independent print unit per entry; obtain report-team sign-off before relying on this in production." Meaning: the Java team assumed the renderer iterates `reportData[]` as separate rows. This has NOT been confirmed. It is an open risk, not a bug. Talk about it as "contract assumption that needs sign-off".

### g.3 Performance analysis (all INFERENCE, no measurements)
- Query shape: 1 scan of CMUW_PL_REFER_RISK (range on CREATED_DATE), then per row 1 partner lookup + 3 CMGE_CODE lookups = 4 sub-pipeline lookups per document. Cost scales with the number of matched documents, and with indexes on `CPGE_VI_PARTNERS_ALL(PART_ID, VERSION)` and `CMGE_CODE(CAT_CODE, CODE_CD)`. Whether these indexes exist is unknown (the properties file sets `auto-index-creation=false`).
- A `$lookup` with `let` + `$expr` can use an index on the foreign collection in recent MongoDB versions for equality, but you must check with `explain`.
- Memory: `$sort` and `$setWindowFields` are in-memory limited (100 MB per stage by default) because the code does not set `allowDiskUse`. Small data is fine.
- Java side: the full result is read into a list, then ~8 objects are created per row and serialised to XML/JSON. With "no dates = all rows" (working tree) this can be large.
- Duplicate work: `dataClass` fetched once (good), but copied into every row.

### g.4 Improvement ideas
1. Fix NULL ordering and tie-breaking (b.3).
2. Decide the "no dates" rule; if parity is wanted, return empty; otherwise cap rows.
3. Convert dates in Mongo with an explicit time zone and send ISO strings to the DTO.
4. Replace three CMGE_CODE `$lookup`s with one small in-memory map loaded once per request (3 tiny queries), or a single `$lookup` using `$in` on CAT_CODE.
5. Move the optional name/id filters before the CMGE lookups (already done) and, where possible, filter partners first when name/id is given.
6. Return the ReportHeader block even when there are no rows.
7. Update and extend tests: pipeline-content tests with the new stages, plus an embedded/containerised Mongo test with seed data (see 05 file).
8. Add RejectRisk to `PlineXsdValidationTest` and sync the XSD copy.
9. Remove credentials from `application.properties`.

---

## h) 30 interview questions (about 1.6 years experience level)

Format: Q, strong answer, common mistake, follow-up. Everything refers to real code in this repo.

1. **What does the RejectRisk report do?** Lists customers rejected for risk with reasons and revoke info, filtered by created date range, name, ID, type, reason. Original was Jasper + Oracle; now Spring Boot + MongoDB returning an XSD payload. *Mistake:* saying "it creates a PDF" (the service returns data, the renderer makes the PDF). *Follow-up:* who renders it?
2. **How are the layers split?** Handler (Mongo aggregation) -> DTO -> Service (maps to payload) -> `ReportStrategy` bean named `AGIC/UW/Plines/RejectRisk/RejectRisk`. *Mistake:* putting mapping logic in the handler. *Follow-up:* why a bean name with slashes?
3. **Explain `(+)` in the Oracle query.** Old-style left outer join: keep risk rows even when no partner. *Mistake:* calling it an inner join. *Follow-up:* how did you do it in Mongo? (`$lookup` + `$arrayElemAt`).
4. **Why `$limit 1` inside the lookup?** To take one partner row, like a single join result; hides duplicates. *Mistake:* thinking it sorts. *Follow-up:* what if the view has 2 rows for the key?
5. **What does `TRUNC(CREATED_DATE) BETWEEN a AND b` mean?** Compare dates ignoring time, inclusive. *Mistake:* using `$lte` with the day start. *Follow-up:* how to write it in Mongo? `>= from` and `< to + 1 day`.
6. **Why `$lt` and +1 day instead of `$lte`?** The stored value has a time. `$lte day-start` loses the whole last day. *Follow-up:* why not `$dateTrunc`? It prevents index use.
7. **What happened when a date was null in HEAD?** `$gte: null` matches only null values, so 0 rows; the fix adds only supplied bounds. *Mistake:* "null is ignored by Mongo". *Follow-up:* what did Oracle do with null dates? (0 rows too).
8. **Is the "no dates returns everything" change safe?** It differs from Oracle and can be huge. Needs a business decision. *Follow-up:* options?
9. **What is the NULL ordering issue?** Oracle ASC nulls last; Mongo nulls first. Unmatched customers appear first. Fix: null flag in sort or sort in Java. *Mistake:* "both are the same".
10. **How is ROWNUM reproduced?** `$setWindowFields` with `$documentNumber` after sorting. *Follow-up:* what Mongo version? (5.0+, INFERENCE) Why not `$addFields:1`? (constant).
11. **Why is the sort stage repeated (window sortBy and `$sort`)?** `$setWindowFields` needs sortBy for numbering; the final `$sort` makes output order explicit. *Mistake:* thinking one guarantees the other's output order.
12. **Case sensitivity of the filters?** Exact and case sensitive in both Oracle and Mongo here; the header lookup is case-insensitive via `UPPER` and regex `i`. *Follow-up:* risk of `Pattern.quote`?
13. **How does `pIdNo` match ID_VALUE2?** `$or` over ID_VALUE1 and ID_VALUE2. *Follow-up:* test case? (`fetchRejectRisks_idNoProvided_addsOrMatchOnIdValues`).
14. **Why is `companyDesc.remarks` created per row?** XSD puts `companyDesc` beside `transDet` in `transDetList`. *Mistake:* putting remarks on `transDet`.
15. **Why `companyType.getCompanyDesc().clear()`?** The constructor adds a default placeholder; clear then add the real one (also tested for shared data in `Template12PolicyDocumentBaseServiceTest.newSharedReportData_clearsDefaultCompanyDescPlaceholder`).
16. **What is DATA_CLASS and where does it come from?** Classification label from `RPGE_RPT_TEMPLATE` found by title and report name (case-insensitive). *Follow-up:* what if it is not found? null.
17. **What is the extra ReportHeader reportData?** Mirrors the Jasper subreport. `mainReportName=RejectRisk`, `subReportName=ReportHeader`. Added only when rows exist.
18. **What does the disclaimer comment mean?** Unconfirmed assumption that each `reportData` is one print unit; needs report-team sign-off.
19. **Why are DTO dates `String`?** Mixed formats arrive; converted later with `DateUtils`. Weakness: relies on `Date.toString()`. *Follow-up:* better way? ISO string from `$dateToString` with time zone.
20. **What date formats does `DateUtils.toLocalDate` accept?** yyyy-MM-dd, dd-MM-yyyy, dd/MM/yyyy, MM/dd/yyyy, dd-MMM-yyyy, dd-MMM-yy, Date.toString style, ISO offset/zoned. *Mistake:* assuming local date-time works (it does not).
21. **What is the time zone risk?** UTC midnight vs Malaysia time (UTC+8). Wrong day windows if data are true instants. Check a sample record.
22. **Why remove the `ROW_NUMBER` addFields?** Dead code (project drops it, real number comes from window).
23. **Why change the DateUtils import?** Plines has its own copy; avoid dependency on the motor package.
24. **What is wrong with the `application.properties` change?** A hard-coded Mongo URI with a password; must not be committed; secret scanning exists; rotate if pushed.
25. **How would you test the date logic?** Unit: assert `$gte`/`$lt`; integration: seeded Mongo with boundary timestamps (D 00:00, D 23:59:59.999, D+1 00:00), plus from=to. *Mistake:* only mocking `MongoTemplate`.
26. **Which existing tests break now and why?** 7 of 15 handler tests (they assert `$lte`, null bounds and stage indexes). Because tests are coupled to stage positions.
27. **How would you reduce 4 lookups per row?** Preload CMGE_CODE map; one lookup with `$in`; indexes.
28. **How would you find if the report matches Jasper?** Run both on the same date range, compare counts, order and each column, including null customers, empty strings and boundary days.
29. **What is the difference between this and the Workmen reject risk report?** Different module, handler, DTO and collections. Always check the package.
30. **What would you change first and why?** (1) Remove the secret, (2) fix the 7 tests, (3) null-ordering, (4) confirm time zone, (5) decide the no-date rule.

### h.1 STAR story (constructed from artifacts, not a real recollection)
Label: CONSTRUCTED FROM ARTIFACTS. Check each fact against the commit log before you use it in an interview.
- Situation: The Plines RejectRisk report was being moved from Jasper/Oracle to a Mongo-based service (HEAD history: `8d1b5374c` first version, `0d715fbad` mapping fix, `cc6a1b84d` date fix).
- Task: Make the Mongo version return the same rows as `TRUNC(CREATED_DATE) BETWEEN` and the same optional filters, and match the XSD (Pline.xsd).
- Action: Compared the JRXML SQL line by line with the aggregation. Found that the `$lte` on a day-start value dropped the last day, and that a missing date produced zero rows. Changed the filter to `[from, to+1day)` and built only supplied bounds. Removed a dead `ROW_NUMBER` stage, fixed the DateUtils import. Noticed the unit tests assert stage indexes and need updating; noticed a credential in `application.properties` and kept it out of the commit.
- Result: Whole-day ranges and single-day ranges behave like Oracle (for UTC-stored data). Open items: time zone check, NULL-first ordering, rule for missing dates, updating 7 tests. (Do not claim production numbers; none are in the repo.)

---

## Uncertainties (what I could not verify)
1. How CREATED_DATE is stored in Mongo (true UTC instants vs wall-clock-as-UTC). No sample data in repo.
2. Whether the partner view is unique on (PART_ID, VERSION), and the BSON types of join keys and codes.
3. Whether indexes exist on the lookup keys, and the real data volume.
4. The Oracle NLS settings (case and sort rules) - assumed defaults.
5. How the renderer uses `rownumber`, `dataClass` per row, and an empty `reportData` list.
6. The exact way Spring converts BSON `Date` to the DTO `String` (I assume `Date.toString()`; other handlers' comments support this).
7. Whether production XSD v0.1.9 has `pCreatedDateFrom/To`; the copy under `src/test` does not.
8. Test results above come from reading code only; no test was run.
9. Whether the `#newDev` credential was ever pushed anywhere.
10. Commit dates are as printed by git (they are in 2026 in this repo); I did not cross-check authorship against JIRA.
