# 12 - Mock Interview Script (60 questions, 3 rounds)
> **AUTHORSHIP UPDATE (confirmed by the developer): git author `GS <extern.gs_nisarga@allianz.com>` is the developer. Wherever this file says commits cannot be attributed to the developer, read it as: the 11 GS commits listed in `KT_Parts/14_My_Commits_And_Ownership.md` ARE theirs; commits by other authors still must not be claimed. No GS commit touches `src/test`, so test authorship is still `[CONFIRM]`.**

For `opus-documentprintdata-service` (Template12, ASP, RejectRisk). A later session can read this file and run a live mock interview with the candidate.

Related files: `11a_Trap_1-50.md`, `11b_Trap_51-100.md`, `11c_Trap_101-150.md`, `11d_Trap_151-200.md` (trap questions with full answers). Tags like `[T027]` point to the matching trap question for the long answer.

## 1. How to run (protocol for the interviewer session)

1. Pick a round (R1 Senior Developer, R2 Technical Lead, R3 Engineering Manager). Each round has 20 questions. A full run is 60 questions. A short run is 8-10 questions per round.
2. Ask ONE question. Say only the question text. Do not give hints.
3. WAIT for the full answer. Do not interrupt. If the candidate asks to repeat, repeat once.
4. Score 1 to 5 using the rubric of that question (section 2 scale below). Be strict. A 5 needs every "must contain" item.
5. Give short feedback: one thing that was good, one missing fact (say the real fact), one wording tip. Keep it under 60 words.
6. Optional: ask ONE follow-up from the "weak signals" list if the score is 3 or lower, or from the trap file if the score is 4 or 5. Score it as a bonus note, not a new question.
7. Go to the next question. Keep a running table: question id, score, missing facts.
8. At the end of the round: average score, the three weakest questions, and the three facts to revise. Then say which `T` files to re-read.
9. Never accept an invented fact. If the candidate invents an incident, a number, a ticket or a person, score 1 for that point and say: "that is not in the evidence; say [CONFIRM] or say you do not know".
10. Never accept "I wrote commit X". Git has no commit by the user NISHUNISARGA. Accept "the team did X; I worked on Y" with [CONFIRM] for the personal part.
11. Never read or show secrets (`<redacted>`).

### Score scale

| Score | Meaning |
|---|---|
| 5 | All "must contain" items, correct names and numbers, honest about limits. |
| 4 | Correct and clear, misses one detail or one limit. |
| 3 | Right idea, vague, misses 2-3 real facts. |
| 2 | Partly wrong or only generic Spring/Mongo talk, no project facts. |
| 1 | Wrong, invented, or "I don't know" with no attempt. |

Bonus honesty rule: if the candidate says "I did not verify this" about something that really is unverified (for example the ASP expansion or the deployed Mongo URI), do not reduce the score; add +0 but note it as a strength.

### Round feedback template

```
Q id | score | what was good | missing fact | trap file ref
```

## 2. Known truths (the interviewer must check answers against these)

- One live bean each: Template12 `AGIC/UW/Plines/PolicySchedule/Template12/PolicyDocument`; ASP `AGIC/UW/Plines/ASPEndtSchedule/EndtDocument`. PolicyDocument2, Landscape, EndtDocument2, AddMemberLandscape, DeletionMemberLandscape are merged in; the old classes have `@Service` or `@Bean` commented out.
- XSD v0.1.6 is the only XSD found. Real Java output does NOT validate against it.
- "ASP" has no written expansion; product config code 070412.
- RejectRisk working-tree fix equals commit `cc6a1b84d` on another branch; it likely breaks 7 of 15 `RejectRiskHandlerTest` tests (static reading).
- No authentication (`anyRequest().permitAll()`). Unknown report name returns HTTP 200 `SUCCESS`.
- `spring.profiles.active=dev` activates nothing (no profile files).
- Boot 4.1.1, Java 17, Tomcat 11.0.25 override, Jackson 2.21.5 via `jackson-2-bom.version`, preferred mapper jackson2, springdoc 3.1.1.
- Personal facts (what the candidate did) are always [CONFIRM] items.

---

## ROUND 1 - SENIOR DEVELOPER (20 questions)

Focus: code-level knowledge. Can the candidate explain the real behaviour of the three reports?

### R1-01. "Explain in two minutes what this service does and where your three reports fit."
**5/5 must contain:** report data provider, not a PDF maker; POST `/gi/documentprintdata/api/opus/GenerateReport`; `reportName` picks a `ReportStrategy` bean; MongoDB mirrors Oracle; OPUS REST APIs for Oracle functions; XML inside JSON `{status, data, timestamp}`; print engine renders; the three reports named (Template12, ASP, RejectRisk) and that Jasper SQL was rewritten as aggregations.
**Weak signals:** says "it makes PDFs"; mentions only Spring Boot; cannot name the endpoint or response shape.

### R1-02. "Which bean do I call for Template12 Landscape?" [T001]
**5/5 must contain:** none separately; only `.../Template12/PolicyDocument`; PD2 and Landscape merged (`executeMergedPolicyDocument2`, `executeMergedLandscape`); old classes `@Service` commented out; old names return HTTP 200 with an empty default transaction.
**Weak signals:** "there is a Landscape service"; thinks old classes were deleted.

### R1-03. "State the ASP service-tax invoice gate exactly." [T027]
**5/5 must contain:** `showTaxInvoice=Y` AND `SHOW_STAX_INV_IND=Y` AND `checklistCopy != Y` AND `SHOW_EINV_IND=Y`; handler `AspServiceTaxInvoiceHandler`; QR needs consent Y, status VALID, QR_IND Y; EndtMT needs `STAX_COUNT > 1` and `ENDT_CODE != 999`.
**Weak signals:** three conditions only; forgets checklist copy; mixes with the GST gate.

### R1-04. "How is the Template12 service-tax gate different from ASP?" [T005]
**5/5 must contain:** Template12 PD2: `showTaxInvoice=Y`, `showStaxInvInd=Y`, `einvPrintInd=Y`; `checklistCopy` goes into the REST calls (mode WIP or OCP), not into the condition; ASP has the direct `!checklistCopy` test; flags come from different handlers.
**Weak signals:** "same gate".

### R1-05. "Why does RejectRisk use `$lt` with dateTo + 1 day?" [T051]
**5/5 must contain:** Oracle `TRUNC(CREATED_DATE) BETWEEN`; `CREATED_DATE` has time; `$lte` at midnight loses the last day; half-open `[from, to+1d)`; no `$dateTrunc` so index usable; null bounds add nothing.
**Weak signals:** "it fixes a bug" with no reason; cannot explain TRUNC.

### R1-06. "What happened with null date bounds, and what does the fix change?" [T052, T053]
**5/5 must contain:** `{$gte: null, $lte: null}` matches only null values; zero real rows, same as Oracle by accident; fix adds only supplied bounds; neither bound means no date stage and so ALL rows; that is a behaviour change needing a decision.
**Weak signals:** "Mongo ignores nulls".

### R1-07. "Which 7 handler tests break and why?" [T056]
**5/5 must contain:** at least five names from: `dateInstances_usedDirectlyInMatch`, `dateStrings_parsedViaDateUtils`, `nullDates_matchStageHasNullBounds`, `rejTypeOnly_addsSecondMatchStage`, `reasonCodeOnly_addsSecondMatchStage`, `rejTypeAndReasonCode_bothInSameMatchStage`, `customerNameAndIdNoAbsent_noExtraMatchStages`; reason: `$lte` assertions and stage index; says it is static reading, not run; lesson: assert by content.
**Weak signals:** "some tests fail"; claims to have run them.

### R1-08. "How do you get ROWNUM in Mongo, and what is the sort trap?" [T057, T058]
**5/5 must contain:** `$setWindowFields` with `$documentNumber`, sortBy `customerName`, then `$sort`; MongoDB 5.0+; Mongo null-first vs Oracle null-last; unmatched customers first; fix with null flag or Java sort; in-memory 100 MB, no `allowDiskUse`.
**Weak signals:** "a counter"; no null-order knowledge.

### R1-09. "Walk me through the RejectRisk pipeline." [T073]
**5/5 must contain:** the order: date `$match`, type/reason `$match`, partner `$lookup` (`let`, `$limit 1`), `$arrayElemAt`, name/ID `$match` after lookup, three `CMGE_CODE` lookups (`PLREJ_RISK_TYPE`, `PLREJ_REASON`, `PLREJ_REVOKE_DUR`), `$project`, window, sort; collection `CMUW_PL_REFER_RISK`; 4 lookups per row.
**Weak signals:** misses categories; puts name filter first.

### R1-10. "Why one `reportData` per RejectRisk row? What is the disclaimer?" [T060]
**5/5 must contain:** `companyDesc/remarks` is a sibling of `transDet`; one `transDetList` per row; extra `ReportHeader` block (`mainReportName=RejectRisk`, `subReportName=ReportHeader`) only when rows exist; disclaimer says renderer treatment is unconfirmed and needs report-team sign-off.
**Weak signals:** "for performance"; has not read the disclaimer.

### R1-11. "What is the CONTRACT_ID bug?" [T031]
**5/5 must contain:** Jasper String parameter vs numeric column; Mongo type-strict; flags null so subreports silently do not print; `safeNum` with `Long.parseLong`; ASP-only override of `buildCommonPipeline` in `AspEndtDocument2MainHandler`; commits on other branches (not HEAD); personal part marked [CONFIRM].
**Weak signals:** "null pointer"; claims commit ownership.

### R1-12. "How do Template12's two primary fetches differ?" [T003, T025]
**5/5 must contain:** fetch 1 throws `IllegalStateException` on WIP/live conflict (ORA-01427 copy), ignores `checklistCopy`; fetch 2 posted-first with `$ifNull`, no conflict check, REST calls; Landscape adds third fallback `UWGE_POLICY_CONTRACTS`; deliberately not unified; "CRITICAL FACT #1".
**Weak signals:** "copy-paste".

### R1-13. "What is `mergeSingleTransDet` and its danger?" [T006]
**5/5 must contain:** reflection merge of non-null fields onto every existing `transDet` (B12 ~189); avoids row multiplication; last non-null wins; boundaries lost; `addSingleTransDet` for multi-row; history fixes (`7399d2fc1`, `c0c6d449a`).
**Weak signals:** "merges lists".

### R1-14. "Name the ASP stubs and the unmapped fields." [T035, T036]
**5/5 must contain:** `PlineEndtScheduleOfBenefitsAspHandler`, `PlineAspLlbHandler`, `PlineAspStiHandler` return `dualValue=1`, no `reportData`, renderer must use `actionCode` (unknown); six unmapped: `subcoverPrem`, `endtDoubleIndemnity`, `endtImaFee`, `endtStaxAmt`, `endtLlbPrem`, `endtStiPrem`; note some leaves exist under `endorsementDetails` (check).
**Weak signals:** "all complete".

### R1-15. "What does `PlineReportNames` do?" [T118, T024]
**5/5 must contain:** `mainNameOf` from `@Service` name (last segment, prefixed by parent unless it starts with it); `tag` fills blanks only; `Template12PolicyDocument` example; ASP sets four own names; `subReportName` from services or `PlineReportBlocks`; elements not in XSD v0.1.6.
**Weak signals:** "names the report"; says it overwrites.

### R1-16. "Does the output validate against the XSD?" [T116]
**5/5 must contain:** No; v0.1.6 only; `dd/MM/yyyy` vs `xs:dateTime`; empty numeric elements; element order; unknown elements (`mainReportName`, `subReportName`, `pCreatedDateFrom/To`); old sample counts 929 and 1766 are dated; `PlineXsdValidationTest` uses placeholders; honest sentence about alignment.
**Weak signals:** "yes, there is an XSD test".

### R1-17. "What does the duplicate class `AspServiceTaxInvoiceHandler` in `src/test` do?" [T044]
**5/5 must contain:** near-copy of production (483 vs 476 lines, no `@Test`); same package; may shadow production on the test classpath; JaCoCo reads `target/classes`; fix: delete and rerun; inferred, not run.
**Weak signals:** "a helper".

### R1-18. "How does the TIN get masked?" [T049]
**5/5 must contain:** first `length-4` characters kept, last 4 replaced by `****`; the KT guide wording "except last 4" is wrong; short TIN becomes `****`; null stays null.
**Weak signals:** repeats "shows last 4".

### R1-19. "Which Template12 parts fail hard and which fail soft?" [T020]
**5/5 must contain:** hard: PolicyDocument primary (WIP/live conflict) and landscape dispatch (`NumberFormatException`, maybe 400 via `IllegalArgumentException` handler, inferred); soft: `B12.add*` log WARN and continue; external REST swallowed; test names.
**Weak signals:** "all caught".

### R1-20. "Tell me one thing in the code you would fix today and how you would test it."
**5/5 must contain:** a concrete item from evidence (for example RejectRisk tests by content; delete the duplicate class; log in `GlobalExceptionHandler`; hard-coded `certNo`; QR uses request `pTaxInvNo`), the fix, and the test (seeded Mongo boundary test or unit test); no invented incident.
**Weak signals:** generic "improve code quality"; invented story.

---

## ROUND 2 - TECHNICAL LEAD (20 questions)

Focus: design, risk, security, versions, operations.

### R2-01. "Why merge PD2 and Landscape into one bean? Cost and benefit?" [T171]
**5/5 must contain:** one response, no double rendering, gates kept; cost: giant classes (ASP 1,701 lines, `execute` ~1,300; Template12 service 2,235; base 3,732), reflection merge, breaking old names, dead classes with tests; alternative band registry; merge commit `c0c6d449a` not claimed; who decided is [CONFIRM].
**Weak signals:** "easier"; no cost named.

### R2-02. "Is the API secured? Defend or attack the `SecurityConfig`." [T080, T097]
**5/5 must contain:** `anyRequest().permitAll()`; no auth; comment claiming header auth is wrong; CSRF off justified only because stateless; Swagger open; personal data in RejectRisk; proposal: gateway/NetworkPolicy plus API key or JWT filter, health open; roll-out risk.
**Weak signals:** "Spring Security protects it".

### R2-03. "Unknown report name: what happens, and would you change it?" [T078, T180]
**5/5 must contain:** HTTP 200 `SUCCESS`, default transaction; legacy switch `default: break`; merged names affected; null name NPE (inferred); change needs caller agreement; cheap first step: WARN log.
**Weak signals:** "404".

### R2-04. "Explain the error contract." [T081, T161]
**5/5 must contain:** IAE to 400 `BAD_REQUEST`; other `Exception` to 500 `INTERNAL_ERROR`; catch-all has no logger; `NoDataFoundException` 404 never thrown; malformed JSON likely 500 (inferred); `NumberFormatException` is an IAE.
**Weak signals:** "we return proper codes".

### R2-05. "Which outbound timeouts and clients exist, and what is wrong?" [T085, T084]
**5/5 must contain:** `PlineUtilsClient` fast-lookup 8 s; the other three clients get the `@Primary` template 5/20 s with no connection-request timeout; swallow policy gives valid-looking blank reports; no retry/circuit breaker; motor clients with `new RestTemplate()`; two base URL keys.
**Weak signals:** "20 s everywhere".

### R2-06. "Describe the SSRF controls." [T086]
**5/5 must contain:** fixed base; `assertTrustedHost` startsWith; `sanitize` regex; policy-ref regex; residual risks (prefix semantics, QR client not audited); commit `40d3fc1f7` not claimed.
**Weak signals:** "HTTPS".

### R2-07. "What do you do with a secret in `application.properties`?" [T068, T145]
**5/5 must contain:** do not commit; env var override; scans (PR, nightly) but whole-file allowlist (OPUSM-50404, review-by 2026-10-23); rotate if pushed; truststore password in VCS; never paste the value.
**Weak signals:** would commit; trusts gitleaks.

### R2-08. "Config traps after the Boot 4 upgrade?" [T138, T139, T136]
**5/5 must contain:** `spring.mongodb.*` vs silent localhost fallback; `preferred-json-mapper=jackson2`; `jackson-2-bom.version`; inert profile `dev`; ineffective `auto-index-creation` and `option.*`; `application-env.properties` via `@PropertySource`.
**Weak signals:** only one trap.

### R2-09. "Why Boot 4.1.1, Tomcat 11.0.25, Jackson 2.21.5, springdoc 3.1.1?" [T186-T190]
**5/5 must contain:** CVE-2026-59270 and Security 7.1.1; Wiz-flagged Tomcat CVEs (BOM 11.0.24); Jackson 2 vs 3 reason; springdoc 2.8.15 `NoClassDefFoundError`; CVE ids from pom comments, not verified; rollback reopens CVE.
**Weak signals:** "latest versions".

### R2-10. "Jackson 2 or 3 in this service?" [T188, T189]
**5/5 must contain:** Jackson 3 default `tools.jackson`; code uses Jackson 2 annotations and `XmlMapper`; property to force Jackson 2; Jackson 2 BOM property name; migration cost; keep until the XSD test exists.
**Weak signals:** "both same".

### R2-11. "Pins below the BOM: your view?" [T191-T193]
**5/5 must contain:** commons-lang3 3.18.0 vs 3.20.0; logback 1.5.34 vs 1.5.38; log4j2 2.25.5; micrometer pin break (`JvmMemoryMeterConventions`); removed pins; verify with `dependency:tree` (not run); `httpcore5` 5.4.3 compatibility check.
**Weak signals:** "pins are safer".

### R2-12. "Design a proper XSD conformance test." [T132, T122]
**5/5 must contain:** mock handlers, call service, same `XmlMapper`, error-collecting validator, ratchet baseline, drift guard by reflection, placeholder tests are misleading; expected failure categories.
**Weak signals:** "add asserts".

### R2-13. "Which schema changes are safe or breaking? How did drift happen?" [T133, T117]
**5/5 must contain:** additive vs breaking list (v0.1.3 move, v0.1.4 removal, v0.1.6 type); Java comments v0.1.7-v0.1.10 while only v0.1.6 exists; governance gap; recommend one agreed XSD and decide contract.
**Weak signals:** "adding is always safe".

### R2-14. "What do the tests prove, and how would you improve them?" [T177, T178]
**5/5 must contain:** 865 tests in scope, 337 DTO/payload (about 39 percent), mocks and LENIENT, no real Mongo/no golden files/no XSD of real output, stage-index coupling, forced-lambda coverage tests, duplicate class, mixed commit `3f2157b28`; improvements list; no coverage numbers available.
**Weak signals:** quotes test counts as quality.

### R2-15. "Triage: ASP tax invoice page missing in production." [T152, T153, T163]
**5/5 must contain:** four-part gate, external indicator default, type mismatch, anchor collection, renderer check by `subReportName`; ordered steps; honest [CONFIRM] if no real case.
**Weak signals:** random guesses; invented case.

### R2-16. "Triage: RejectRisk issues (missing last-day rows, empty with no dates, order)." [T154, T155, T165]
**5/5 must contain:** `$lte` bug, fix not on HEAD, null bound semantics, undecided no-date rule, null-first ordering, time zone check; tests to update.
**Weak signals:** blames users.

### R2-17. "A 500 with no logs." [T161]
**5/5 must contain:** `GlobalExceptionHandler` has no logger; WARN in 400 handler; fail-soft WARN lines; add `log.error`, correlation id; no metrics exposed.
**Weak signals:** "enable debug".

### R2-18. "How would you refactor Template12/ASP safely?" [T171, T183]
**5/5 must contain:** characterization tests first; band registry with gate/handler/mapper/subReportName; shared product-context object (replace 3 fetches only with sign-off); explicit mapping instead of reflection; remove dead classes after confirming callers; stepwise.
**Weak signals:** "rewrite".

### R2-19. "Deviations from Jasper: how do you manage them?" [T179, T010, T011]
**5/5 must contain:** a register with owner and sign-off; examples (PAGE_NUMBER, ATC always fires, PDS wired, `INST_IND`, null order, dispatch always runs, empty RejectRisk header); golden-file comparison; do not silently fix.
**Weak signals:** "Jasper is truth".

### R2-20. "Rollback and release process?" [T169, T170, T149]
**5/5 must contain:** stateless service; image tag; Helm repo separate; manual `workflow_dispatch`; Sonar gate on PRs; CodeQL branch filter gap (inferred); config drift risk; [CONFIRM] real steps.
**Weak signals:** "git revert"; invented pipeline.

---

## ROUND 3 - ENGINEERING MANAGER (20 questions)

Focus: ownership, honesty, communication, risk and delivery. Personal facts must be [CONFIRM] items; the candidate must not invent.

### R3-01. "What was your scope and what did you deliver?"
**5/5 must contain:** the three reports (Template12 PolicyDocument/PolicyDocument2/Landscape; ASP EndtDocument/EndtDocument2/AddMemberLandscape/DeleteMemberLandscape; RejectRisk); analysis, code, unit tests, XSD changes; period 12 May to 30 Dec 2026; specifics from memory marked [CONFIRM]; no commit claimed from git.
**Weak signals:** vague; claims the whole platform.

### R3-02. "Which parts of the code are yours and how do I verify that?" [T184]
**5/5 must contain:** admits git does not prove authorship (no commit by NISHUNISARGA); points to Jira, PRs, review comments [CONFIRM]; separates "team did" from "I did".
**Weak signals:** claims commit hashes; defensive.

### R3-03. "Tell me about a hard bug you handled."
**5/5 must contain:** a real story from [CONFIRM] material OR a clearly labelled study of the CONTRACT_ID mismatch or RejectRisk date bug (cause, detection, fix, test, what remains); STAR structure; no invented numbers.
**Weak signals:** invented drama; no cause; no result.

### R3-04. "What is the biggest risk in your modules right now?"
**5/5 must contain:** a ranked list: no auth with personal data; secret in working tree; XSD drift and no real validation; RejectRisk open items; stubs and unconfirmed renderer contract; silent failures; honest about unknowns.
**Weak signals:** "none".

### R3-05. "You find a credential in your working copy. What do you do and who do you tell?" [T068]
**5/5 must contain:** do not commit; restore; env var; notify security if pushed; rotate; know scans and allowlist gap; no pasting the value.
**Weak signals:** shrugs it off.

### R3-06. "Your fix duplicates a commit on another branch. How do you handle that with the team?" [T055]
**5/5 must contain:** acknowledge `cc6a1b84d` (OPUSM-41763, other branch, 194 ahead / 1363 behind); coordinate with its owner; avoid duplicate work; decide merge vs cherry-pick; mention tests on that branch missing; no claiming credit.
**Weak signals:** claims originality; ignores the other branch.

### R3-07. "A stakeholder asks: is the Template12 output correct versus the old Jasper? What do you answer?" [T179]
**5/5 must contain:** honest: unit tests prove intent only; no golden comparison in repo; known deviations list; propose a comparison plan on sample policies; ask for UAT; no "100 percent".
**Weak signals:** "yes, tested".

### R3-08. "We want to ship Friday. RejectRisk has 7 failing tests and an open time-zone question. Advice?" [T056, T054]
**5/5 must contain:** do not ship blindly; fix tests by content (small effort); verify one known record for the zone; decide no-date rule with business; ship with documented risk only if approved; remove secret from commit; track items in Jira.
**Weak signals:** "just ship"; or "never ship".

### R3-09. "How do you estimate and plan a new Plines report in this service?" [T093]
**5/5 must contain:** read the jrxml tree (parameters, SQL, `printWhenExpression`); traceability table; handlers per sub-report; mapping sheet and XSD leaves; services and gates; tests; compare with Jasper; known unknowns; buffer for XSD and renderer sign-off.
**Weak signals:** only "coding".

### R3-10. "What would you tell a new joiner about this codebase's dangers?" 
**5/5 must contain:** at least five: merged beans, old classes dead; type strictness; WIP vs live; swallowed failures; stale config keys; XSD v0.1.6; no auth; test limitations; duplicate class; two base-URL keys.
**Weak signals:** generic advice.

### R3-11. "Where did you disagree with the team or a requirement, and how did you resolve it?"
**5/5 must contain:** a real [CONFIRM] example; or an evidence-based example such as raising the RejectRisk renderer contract or the PAGE_NUMBER gap; respectful, data-based, outcome; no blaming.
**Weak signals:** none; blames; invented.

### R3-12. "How do you communicate a breaking change like merged report names?" [T094, T157]
**5/5 must contain:** announce to orchestrator/report team, list old to new names, response structure change, alias period, release notes/CHANGELOG, test with callers; admits repo shows no migration evidence.
**Weak signals:** "they will see".

### R3-13. "Production incident: users get blank reports but no errors. First 30 minutes?" [T162, T156, T160]
**5/5 must contain:** triage order: exact request, report name, logs for WARN, upstream API health and URL keys, truststore, Mongo DB name/data, recent deploy/config change; communicate status; rollback via image tag; postmortem items (metrics, warning flag); no invented prior incident.
**Weak signals:** panic; no ordering.

### R3-14. "What did you learn about the Boot 4 upgrade, and what would you do differently?" [T186, T191, T197]
**5/5 must contain:** silent-ignore traps; boot-only failures (springdoc); micrometer pin break; Jackson 2/3; add a boot smoke test; verify with `dependency:tree`; stale docs; CVE-driven pins reviewed each bump; not claiming authorship.
**Weak signals:** "smooth".

### R3-15. "How do you work with the report/Jasper team?" [T185, T060]
**5/5 must contain:** agree the contract (reportData as unit, `mainReportName`/`subReportName`, dates, empty elements); sign-off items such as the RejectRisk disclaimer; share XSD and mapping sheet; example render checks.
**Weak signals:** "they will adapt".

### R3-16. "How do you balance fixing Jasper parity gaps with new features?"
**5/5 must contain:** a deviation register; severity by business impact (missing invoice page vs wording); fix wrong-page issues first; owner and sign-off; bundle test improvement; negotiate scope.
**Weak signals:** "fix everything".

### R3-17. "How do you review someone else's PR in this repo?" 
**5/5 must contain:** small diffs; separate tests from production; check gates vs jrxml; type strictness; secrets; log of PII; unit tests assert content not index; Jira key, Sonar gate, secret scan; note the mixed commit `3f2157b28` as a bad example; AI review is only an aid.
**Weak signals:** "LGTM culture".

### R3-18. "What would you do in your first week if you owned this service?" [T183]
**5/5 must contain:** security quick wins; run `mvn verify` and `dependency:tree`; confirm deployed config source; fix tests; deviation register; stakeholder map (report team, platform, DevOps, UAT); priorities with reasons.
**Weak signals:** "rewrite it".

### R3-19. "What do you not know about this project?" [T150, T032]
**5/5 must contain:** at least four honest unknowns: ASP expansion; deployed Mongo URI source; meaning of `SB_`/`OCP_`; newer XSD v0.1.7-v0.1.10; renderer behaviour; test run status; collection types/indexes; plus how to find out.
**Weak signals:** "nothing"; invents.

### R3-20. "Why should we hire you for this kind of work? Where are you still growing?"
**5/5 must contain:** about 1.6 years experience stated honestly; strengths shown with evidence from this project (reading Jasper, translating SQL to aggregations, careful about unverified claims); growth areas (performance measurement, integration tests, design of large refactors); concrete learning plan; [CONFIRM] personal facts.
**Weak signals:** arrogance or false modesty; no evidence.

---

## 3. After the mock: summary template

```
Round: R1 / R2 / R3      Date: ____      Questions asked: __
Average score: __        Lowest 3: __ , __ , __
Facts to revise: 1) ... 2) ... 3) ...
Honesty check: any invented fact? Y/N   Any commit-ownership claim? Y/N
Next step: re-read 11a / 11b / 11c / 11d sections: ...
```

## 4. Files and open uncertainties (also see the final report)

- Files written: `11a_Trap_1-50.md`, `11b_Trap_51-100.md`, `11c_Trap_101-150.md`, `11d_Trap_151-200.md`, `12_Mock_Interview_Script.md` (all in `C:\Users\EPGRVFY\Documents\SELF\KT_Parts\`).
- Open uncertainties that affect answers: HTTP status of `NumberFormatException` (my reading says 400 through `GenericExceptionHandler`; the KT guide says 500; not run); `RejectRiskHandlerTest` failures are from reading, not running; deployed Mongo URI source and Helm values; whether the renderer treats `reportData[]` as units; ASP expansion; the newer XSD versions; the real meaning of `SB_`/`OCP_`; whether `mvn dependency:tree` matches the BOM-reported versions.
