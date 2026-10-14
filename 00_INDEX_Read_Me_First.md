# OPUS Document Print Data — KT and Interview Pack (INDEX)

Project: `opus-documentprintdata-service` (Spring Boot 4.1.1, Java 17, MongoDB, XML-in-JSON report data provider).
Your modules: **PLine Template12**, **ASP**, **RejectRisk**.
Built from: the repo (branch `feature/retrofit-develop-updated`, history up to 2026-10-08), the Jasper dumps, the Pline XSD v0.1.6, the READMEs, and git history. Nothing was executed (no Maven run, no Mongo queries, no tests).

## Read in this order
1. `OPUS_DocumentPrintData_KT_Guide.md` — main guide: Phases 1-5 (overview, domain, the three modules, architecture, tech/versions) and the support cheat sheet. Start with its Section 0 "honesty notes".
2. `KT_Parts/13_Final_Cheat_Sheet.md` — one-stop revision sheet, incl. 1/3/5/10-minute explanations and the honest gaps list.
3. Module deep dives (jrxml -> Java -> Mongo -> XSD traceability):
   - `KT_Parts/02_Template12_Deep_Dive.md`
   - `KT_Parts/03_ASP_Deep_Dive.md`
   - `KT_Parts/01_RejectRisk_Deep_Dive.md`
4. `KT_Parts/04_XSD_XML_Deep_Dive.md` — XSD anatomy, v0.1.3-v0.1.6 changelog, Java-vs-XSD audit.
5. `KT_Parts/06_Config_API_DB_DevOps.md` — configuration, API, database, DevOps, security.
6. `KT_Parts/05_Testing_Deep_Dive.md` — tests, coverage tooling, quality assessment.
7. `KT_Parts/07a_Trace_Defects_Support_Perf.md` — requirement-to-deployment traces, defects/RCA, support runbooks, performance risks.
8. `KT_Parts/07b_Ownership_Answers_And_STAR_Stories.md` — first-person answers and STAR stories (fill in every `[CONFIRM: ...]`).
9. `KT_Parts/14_My_Commits_And_Ownership.md` — YOUR 11 commits mapped to modules, safe first-person claims, and gaps. Read this before 07b.

## Question banks (each question: why asked / ideal answer / common mistakes / follow-ups)
| Files | Topic | Count |
|---|---|---|
| 08a-08b | Project | 100 |
| 08c-08d | Technical | 100 |
| 08e-08f | Scenario | 100 |
| 09a-09b | Production support | 100 |
| 09c-09d | Debugging | 100 |
| 09e-09f | XML | 100 |
| 10a-10b | API | 100 |
| 10c-10d | Database | 100 |
| 10e-10f | Configuration | 100 |
| 10g-10h | XSD | 100 |
| 11a-11d | Interviewer trap questions (strong/weak answer, red flags, follow-ups) | 200 |
| 12 | Mock interview script (60 questions, 3 rounds, scoring rubric) | 60 |

## Read these warnings before any interview
- **Authorship (updated):** you confirmed git author `GS <extern.gs_nisarga@allianz.com>` is you. See `KT_Parts/14_My_Commits_And_Ownership.md` for your 11 commits mapped to modules. Do not claim other authors' commits. No GS commit touches `src/test`; every remaining `[CONFIRM: ...]` needs a fact from you.
- **Not run:** statements such as "7 of 15 RejectRisk handler tests fail" and "the test-tree copy of `AspServiceTaxInvoiceHandler` shadows the real class" come from reading code. Verify by running `./mvnw test`.
- **XSD:** only v0.1.6 exists locally. Real Java output does not validate against it (dates `dd/MM/yyyy` vs `xs:dateTime`, empty elements, sibling order, `mainReportName`/`subReportName`). Java comments mention v0.1.7-v0.1.10, which were not found.
- **ASP:** the code never says what it stands for. Do not invent an expansion.
- **Credentials:** your working copy of `src/main/resources/application.properties` has an uncommitted block with a MongoDB URI containing a password. Do not commit it. The pack never prints it.
- **Open questions for your team:** merge status of `cc6a1b84d` (OPUSM-41763 branch), where the deployed Mongo URI comes from, the Helm/rollback process, whether the print engine validates against the XSD, and the mapping workbooks:
  - `C:\Users\EPGRVFY\Documents\Pline_MASTER_MAPPING_SHEET_v0.1 (final1) 1 5.xlsx`
  - `C:\Users\EPGRVFY\Documents\Plines_Base_code\T12_final_code\RejectRisk\RejectRisk\RejectRisk_XSD_Mapping_sheet.xlsx`

## Phase coverage
| Phase | Where |
|---|---|
| 1-5 | Main guide + cheat sheet |
| 3, 18 | Guide Phase 3, deep dives 01-03, 07b |
| 6 | Deep dives 02/03/01, 07a (code-level), 06 |
| 7 | 06 (configuration catalog), 10e-10f |
| 8 | 04, 09e-09f, 10g-10h |
| 9 | 06 (database catalog), 10c-10d |
| 10 | 06 (API catalog), 10a-10b |
| 11-14 | 07a |
| 15 | 05 |
| 16-17 | 06 |
| 19-20 | 07b |
| 21 | 08, 09, 10 |
| 22 | 11a-11d |
| 23 | 12 (live mock interview: ask me to start) |
| 24 | 13 |

## Added after you pointed me to your other folders (Oct 14)
| File | What it adds |
|---|---|
| `KT_Parts/15_Existing_Notes_Prompts_And_Process.md` | Your earlier prep notes vs this pack (conflicts), your AI-assisted comparison method (prompt files), the staxCount/eInvStatus bug (best real story), the data-mismatch (Oracle and Mongo both 0 rows) case, STAR stories from your own notes. |
| `KT_Parts/16_Real_Testing_Evidence.md` | Your manual data-parity testing (Oracle SQL vs Mongo Compass vs Postman), RejectRisk LEE KEN439275 example, Template12 defect list, hardcoded-test-data warning. Replaces the earlier "no SIT evidence" line in 05. |
| `KT_Parts/17_XSD_Mapping_And_Real_Payloads.md` | Mapping workbooks explained, per-module request keys, real responses validated against XSD v0.1.6 (still thousands of errors, mostly empty numeric/date elements), what your XSD work really was. |
| `KT_Parts/18_Clones_Branches_Reconciliation.md` | Your final clone has only 2 of your 11 commits; where the rest live; port checklist; git identity warning. |

### Five things to act on
1. The open clone lacks your newest ASP/RejectRisk work (`cc6a1b84d` and earlier). Port it before relying on that clone.
2. No GS commit touches `src/test` in any clone. Find your tests or state your testing contribution as manual parity testing.
3. Do not claim the Template12 merge (`c0c6d449a`, Amarnath Yadav), the XSD files, the Boot 4.1.1 upgrade or the OPUSM-44159 test generation.
4. Do not say the output is XSD-valid. Newest samples still fail v0.1.6 (empty decimals/dates, dd/MM/yyyy, element order, mainReportName/subReportName).
5. Do not commit the Mongo URI in `application.properties`. Your commits in the final clone carry the gmail identity, not GS.
