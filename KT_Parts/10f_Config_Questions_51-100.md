# 10f - Configuration Questions 51-100 (opus-documentprintdata-service)

Evidence base: `06_Config_API_DB_DevOps.md` (Parts 1, 4), `05_Testing_Deep_Dive.md`, `.github/workflows/*` as summarised in the KT notes, `gitleaks.toml`, `pom.xml`. The Helm chart, deployment pipeline and environment variables are NOT in the repo, so anything about the live environment is marked [CONFIRM: ...] or (inferred). Secrets are never printed (`<redacted>`).

---

### Q51. How would you promote this service from dev2 to UAT? List every config item that changes.
- **Why the interviewer asks:** Real release runbook knowledge.
- **Ideal answer:** At minimum: `API_BASE-URL`, `plines.api.base-url`, `workmen.api.base-url`, `spring.mongodb.uri`, `spring.mongodb.database`; the truststore only if the UAT CA is not in `mytruststore.p12`. Hosts follow `devN-apiservice.opusm.srv.allianz` in the file. There is no single environment switch and `spring.profiles.active` does nothing. How the platform injects these is outside the repo [CONFIRM: Helm values].
- **Common mistakes:** Changing the profile and calling it done.
- **Follow-up questions:** How would you reduce the number of keys? (One base URL property with aliases.)

### Q52. What can you NOT claim about deployment from this repo?
- **Why the interviewer asks:** Honesty and boundary knowledge.
- **Ideal answer:** No Dockerfile, Helm chart, Kubernetes manifests, environment values, canary or rollback procedure are in the repo (`git ls-files` filter found none). I can say: the image is built by the shared action `java-microservice` from `cicd-workflow.yml`, run manually; rollback is presumably redeploying a previous image tag (inferred); there is no DB migration because the service owns no schema. Everything else needs the platform team.
- **Common mistakes:** Describing a Helm-based canary as fact.
- **Follow-up questions:** Who owns the Helm repo? [CONFIRM.]

### Q53. Describe `cicd-workflow.yml`.
- **Why the interviewer asks:** CI/CD awareness at the right level.
- **Ideal answer:** Trigger: `workflow_dispatch` only, optional input `imageTag` (the `release: created` trigger is commented out). It checks out this repo and the shared repo `OPUSCCC/opus-github-cicd-workflows` (token secret `CICD_READ_REPO`), runs the local action `actions/java-microservice` with `java_version 17` and `app_name opus-gi-documentprintdata-service`, with Sonar and Wiz secrets. Runner label `adp-runner-medium`. Actions are pinned by SHA. The build/scan/publish steps live in the shared action and are not visible.
- **Common mistakes:** Saying it deploys automatically on merge.
- **Follow-up questions:** Gap? (Manual trigger; no visible per-environment deploy.)

### Q54. How does the Sonar workflow gate PRs?
- **Why the interviewer asks:** Quality gate understanding.
- **Ideal answer:** `sonar-code-review.yml` runs on push to `retrofit-develop-R2`/`R3` and PRs to `R1`/`R2`: JDK 17 Temurin, `mvn -B verify sonar:sonar -Dsonar.qualitygate.wait=true`. A failing quality gate fails the check and blocks the merge if the check is required (branch protection not visible). JaCoCo 0.8.12 produces the coverage Sonar reads.
- **Common mistakes:** Assuming gate failure reasons are in the repo (they are in Sonar).
- **Follow-up questions:** What affects the gate for your changes? (New-code coverage and issues; project thresholds [CONFIRM].)

### Q55. What does the CodeQL workflow cover and what gap exists?
- **Why the interviewer asks:** Reading CI critically.
- **Ideal answer:** `codeql.yml` runs on push to `develop`/`main`, PRs to `develop`/`retrofit-develop`/`main`, a weekly cron `0 2 * * 6` and manually, using a shared SAST action for Java. The branch filters do not include `retrofit-develop-R1`/`R2`, the branches PRs actually target here, so CodeQL rarely runs on PRs (inferred from filters). Suggest adding the R-branches.
- **Common mistakes:** Claiming every PR is CodeQL-scanned.
- **Follow-up questions:** Why cron? (Catches new rules on old code.)

### Q56. Explain the secret-scan workflows.
- **Why the interviewer asks:** Security automation.
- **Ideal answer:** `secret-scan.yml` runs on every PR and push, calls the shared `secret-scan-template.yml@main` in `diff` mode with fallback base `retrofit-develop-R2` (OPUSM-57911). `secret-scan-nightly.yml` runs cron `0 2 * * *` in `full-history` mode. The repo's `gitleaks.toml` extends the org ruleset. Both are only as good as the allowlist (Q57).
- **Common mistakes:** Believing history is scrubbed (scans only report).
- **Follow-up questions:** What if a secret is found in history? (Rotate; rewrite only if policy requires.)

### Q57. What is wrong with the gitleaks allowlist?
- **Why the interviewer asks:** A real governance risk next to your config file.
- **Ideal answer:** `gitleaks.toml` has one `[allowlist]` exempting two entire files: `application.properties` and `application-env.properties` (ticket OPUSM-50404, owner: documentprintdata team, review-by 2026-10-23). Any new secret of any type added to those files is silently suppressed, including a new Mongo password or the truststore password. Today is 2026-10-13, so review-by is ten days away. Better: move secrets out and remove the file-wide allowlist, or allowlist specific fingerprints.
- **Common mistakes:** Calling the allowlist harmless because it is "tracked".
- **Follow-up questions:** Why did it change from `[[allowlist]]` to `[allowlist]`? (gitleaks 8.30.1 rejects arrays of tables; commit `796191c51`.)

### Q58. Your working copy has an uncommitted Mongo block. What do you do?
- **Why the interviewer asks:** Practical security discipline.
- **Ideal answer:** Do not commit it, do not paste it anywhere. `git status` shows `application.properties` modified (a `#newDev` block with URI and DB name). Options: move to an untracked local file such as `application-local.properties` (ignored via `.gitignore` if added) or use env vars (`SPRING_MONGODB_URI`); before each commit run `git diff --staged` for properties files; stage hunks with `git add -p`. If it was ever committed, rotate the credential.
- **Common mistakes:** Relying on the allowlist-protected file ("scanner will catch it" - it will not).
- **Follow-up questions:** How to prevent accidents? (Pre-commit hook; a local override file.)

### Q59. Why did commit `7dd2f2f67` remove the Mongo UAT URI from the tracked file?
- **Why the interviewer asks:** Context for the current state.
- **Ideal answer:** Commit `7dd2f2f67` (2026-09-26, titled "OPUSM-52675 Claude PR review fix") removed the UAT URI and database lines from the committed `application.properties` after automated review flagged credentials (the title suggests the AI PR review; inferred). Result: the committed file no longer carries a URI, so a deployed pod must receive it from outside; the old URI remains in git history, so rotation is advised.
- **Common mistakes:** Thinking removal from HEAD removes exposure.
- **Follow-up questions:** Who rotates? [CONFIRM: process owner.]

### Q60. Why is the `claude-pr-review.yml` restricted to same-repo PRs and certain author associations?
- **Why the interviewer asks:** Security of automation.
- **Ideal answer:** It runs on non-draft, same-repo PRs to `retrofit-develop-R1/R2/main`, and on `@claude` comments by OWNER/MEMBER/COLLABORATOR, with a 20-minute timeout and `GH_HOST=github.developer.allianz.io`. Restricting avoids forks and untrusted users triggering an LLM job that holds repository tokens (prompt injection and abuse). It is a control, not just convenience.
- **Common mistakes:** Saying it auto-approves.
- **Follow-up questions:** What can go wrong? (Secrets exposure via prompt injection if permissions are broad.)

### Q61. What does `jira-commit-check.yml` enforce and how does it affect you?
- **Why the interviewer asks:** Workflow compliance.
- **Ideal answer:** On PR open/sync/reopen/edit, a shared action verifies commits reference an `OPUSM` Jira key (allowed project "OPUSM"; uses `JIRA_USER`/`JIRA_PASSWORD` secrets). Commit messages like "OPUSM-44159 Added report, sub report name in response" satisfy it. A message without the key blocks the PR.
- **Common mistakes:** Putting the key only in the branch name.
- **Follow-up questions:** What if you used the wrong key? (Fix with rebase/amend before merge.)

### Q62. How do branches and PRs flow in this repo?
- **Why the interviewer asks:** Release process awareness.
- **Ideal answer:** Default branch `retrofit-develop-R1`; also `origin/main`, `develop`, `retrofit-develop-R2` (long-lived integration branch per `secret-scan.yml`). Feature branches such as `OPUSM-41763/retrofit-develop-R2-pline-Template12-ASP-RejectRisk` target R1/R2 through GitHub PRs ("Merge pull request #NNN from OPUSCCC/<branch>"). Feature branches are frequently synced from R2 by merge. Naming is inconsistent. The current branch is `feature/retrofit-develop-updated`.
- **Common mistakes:** Naming `main` as the working branch.
- **Follow-up questions:** Rebase or merge? (Merge commits dominate: 1251 merges of 2845 commits.)

### Q63. `changelog-update.yml`: what triggers it and what does it write?
- **Why the interviewer asks:** Release documentation.
- **Ideal answer:** On PR closed/merged into `main` and pushes to `main`, a shared changelog workflow (pinned by SHA) updates `CHANGELOG.md` (entries PR #626..#864 seen). Since your work lands on R1/R2 first, entries appear only when R-branches merge to `main`.
- **Common mistakes:** Expecting a changelog update on each feature PR.
- **Follow-up questions:** Who merges to main? [CONFIRM.]

### Q64. What is the role of `@PropertySource` ordering and could it silently ignore your override?
- **Why the interviewer asks:** Precedence mechanics.
- **Ideal answer:** `@PropertySource` entries have the lowest precedence among the standard sources. A value in `application-env.properties` is overridden by the same key in `application.properties`, env vars or system props. So if someone sets `app.ssl.trust-store` in `application.properties`, it wins silently over the "env" file. Conversely if an override is not applied, check for a higher source defining the same key.
- **Common mistakes:** Believing `@PropertySource` is highest priority because it is explicit.
- **Follow-up questions:** How to print the effective source? (Actuator `env` is not exposed; use a debug startup listener.)

### Q65. How would you test configuration without starting the whole app?
- **Why the interviewer asks:** Testing config in a codebase with few context tests.
- **Ideal answer:** Use `ApplicationContextRunner` or `@SpringBootTest(classes = {...})` slices with `@TestPropertySource` to bind a `@ConfigurationProperties` class and assert values; a plain `Properties` test that loads `application.properties` and asserts alias keys equal (for example all comp-brn-number keys). The repo has no test properties file and only one context test in motor, and the pom comment says a micrometer version once broke `@SpringBootTest` startup, so keep context tests narrow.
- **Common mistakes:** Full `@SpringBootTest` for a property check.
- **Follow-up questions:** What would you assert for URLs? (Starts with `https://`, path starts with `/`.)

### Q66. Why are the tests unable to catch the `spring.mongodb.*` mistake?
- **Why the interviewer asks:** Test limits tied to config.
- **Ideal answer:** Handler tests use mocked `MongoTemplate`; no test starts the Mongo auto-configuration with a real or fake URI. So a wrong prefix compiles, tests pass, and the failure appears in an environment. A minimal `ApplicationContextRunner` test could bind `spring.mongodb.uri` and assert the client settings host, with no real server.
- **Common mistakes:** Trusting 19,665 `@Test` annotations (testing guide figure for the whole repo) as config coverage.
- **Follow-up questions:** Is that count about the Plines modules? (No, it is repo-wide.)

### Q67. How do you document configuration so new developers do not repeat these mistakes?
- **Why the interviewer asks:** Knowledge-transfer thinking.
- **Ideal answer:** A short table in the README for each key: purpose, readers (class), default, environment-specific (yes/no), secret (yes/no); mark deprecated/dead keys; keep a promotion checklist. Add Spring's `additional-spring-configuration-metadata.json` for typed properties if migrating to `@ConfigurationProperties`. Flag the three trap groups: base URLs, Mongo prefix, ineffective pool keys.
- **Common mistakes:** Documenting values instead of ownership.
- **Follow-up questions:** Where is today's doc? (`README.md`, `ARCHITECTURE.md`, `plines/README.md`; content for config not verified.)

### Q68. How would you add a brand-new external API for Template12 in config and code?
- **Why the interviewer asks:** End-to-end change checklist.
- **Ideal answer:** (1) Get base path and parameters from the API team [CONFIRM]. (2) Add a path key under the Plines block (leading slash, `/ext/...`), reusing `plines.api.base-url` or `API_BASE-URL` consistently. (3) Add a method in `PlineUtilsClient` using `safeGet`, `sanitize`, `assertTrustedHost`, and the fast-lookup RestTemplate. (4) Decide fail-soft default. (5) Unit test with a mocked RestTemplate for success, failure and invalid input. (6) Record the key in the config table. (7) Smoke-call in dev.
- **Common mistakes:** Creating a new `RestTemplate` or hard-coding a URL.
- **Follow-up questions:** Where do timeouts come from? (`app.http.fast-lookup.read-timeout-ms`.)

### Q69. How would you add a per-client timeout (for example 3 s for one client) properly?
- **Why the interviewer asks:** Config extension.
- **Ideal answer:** Define a new `@Bean("xyzRestTemplate")` in `PlineRestTemplateConfig` using the same HttpClient5 builder with a property such as `app.http.xyz.read-timeout-ms` with a code default, then inject it with `@Qualifier`. Do not change the `@Primary` bean's global timeouts. Existing precedent: `fastLookupRestTemplate` (`4cd1cad2e`).
- **Common mistakes:** Mutating the shared template per call.
- **Follow-up questions:** Why `destroyMethod="close"` on these beans? (To close the pooled HttpClient on shutdown.)

### Q70. Describe a safe plan to unify the many base-URL and path keys.
- **Why the interviewer asks:** Refactoring under risk.
- **Ideal answer:** Step 1: inventory keys and readers (the KT catalog gives it). Step 2: introduce a typed `@ConfigurationProperties("plines.api")` with `baseUrl` defaulting to `${API_BASE-URL}`. Step 3: change Pline clients to use it, leaving old keys in place. Step 4: add an equality test between aliases. Step 5: remove unread keys in a separate PR after dev verification. Step 6: tell ops the new key. Never change values and structure in one PR.
- **Common mistakes:** Deleting 40 keys at once.
- **Follow-up questions:** How to roll back? (Revert the PR; old keys remain until step 5.)

### Q71. What is a safe procedure to delete a dead property?
- **Why the interviewer asks:** Change safety.
- **Ideal answer:** Search for `${key}`, `@Value("key")`, `@ConfigurationProperties` prefixes, `Environment.getProperty`, YAML/Helm values referencing it (outside repo [CONFIRM]); delete in a separate PR; run the full test suite and start the app in dev; watch for `Could not resolve placeholder`. Keys containing real hostnames should be removed first (commented dev1/opus-core lines 75, 77, 96-97, etc.).
- **Common mistakes:** Trusting grep in this repo only (Helm may still set the env var).
- **Follow-up questions:** Risk of leaving it? (Confusion, not outages.)

### Q72. Why is `org.json:json:20231013` a configuration/dependency concern?
- **Why the interviewer asks:** Dependency hygiene specific to `CustomRequestFilter`.
- **Ideal answer:** The filter parses attacker-controlled request bodies with `org.json` 20231013 (pom line ~128); the KT notes flag it as old and note a CVE-2023-5072-era issue (inferred; verify against advisories). No body-size cap. Mitigation: upgrade, or replace with Jackson `ObjectMapper` already present, and add a size guard.
- **Common mistakes:** Quoting CVE specifics as certain.
- **Follow-up questions:** Why not switch to Jackson? (Possible; need to preserve the array-flatten behaviour.)

### Q73. What else in the repo configures security besides `SecurityConfig`?
- **Why the interviewer asks:** Full-picture security.
- **Ideal answer:** `gitleaks.toml`, CodeQL, Wiz secrets in CI, Sonar quality gate, the pom CVE pins and the custom truststore. At runtime only `SecurityConfig` applies: stateless, CSRF off, `/actuator/health|info` open, other actuator denied, everything else permitted. No CORS config found in my notes; browsers are not callers (inferred).
- **Common mistakes:** Treating CI scanners as runtime protection.
- **Follow-up questions:** Which control is missing? (Authentication, rate limiting.)

### Q74. Is TLS terminated in this service for inbound traffic?
- **Why the interviewer asks:** Deployment security.
- **Ideal answer:** No: inbound is plain HTTP on 8080 (no `server.ssl.*` keys). TLS is presumably terminated at ingress (inferred; Helm not in repo). Outbound TLS uses the custom truststore. Mongo uses `ssl=true`.
- **Common mistakes:** Saying the p12 secures inbound.
- **Follow-up questions:** In-cluster mTLS? (Not in this repo.)

### Q75. Why does `CustomRequestFilter` printing bodies count as a configuration problem?
- **Why the interviewer asks:** Config cannot control it.
- **Ideal answer:** `System.out.println` bypasses `logging.level.*`, so no property can silence it. Prod log volume and PII exposure depend on code, not config. Compare: Mongo DEBUG logs can be reduced with a property. Fix in code (use SLF4J at DEBUG with masking).
- **Common mistakes:** Trying to silence it with `logging.level.root=OFF`.
- **Follow-up questions:** Could stdout be filtered at the platform? (Possible in log pipeline; not a fix.)

### Q76. What would a minimal `application-uat.properties` contain?
- **Why the interviewer asks:** Practical profile design.
- **Ideal answer:** Only non-secret differences: `logging.level.org.springframework.data.mongodb.core.MongoTemplate=INFO`, `API_BASE-URL`, `plines.api.base-url`, `workmen.api.base-url`, and maybe springdoc disabled. Secrets (`spring.mongodb.uri`, truststore password) come from env or mounted secret. Activated by `spring.profiles.active=uat` from the platform, replacing the useless `dev` in the base file.
- **Common mistakes:** Copying the whole base file.
- **Follow-up questions:** Which host name for UAT? [CONFIRM: environment naming.]

### Q77. How do you handle the leading-space URL line safely?
- **Why the interviewer asks:** Detail awareness.
- **Ideal answer:** Line 76 `API_BASE-URL= https://...` has a leading space after `=`; the `Properties` loader trims leading whitespace of values, so it works; trailing whitespace is not trimmed (none present). Clean it anyway. When copying to Helm values, leading/trailing spaces can break `startsWith(baseUrl)` in `assertTrustedHost`.
- **Common mistakes:** Believing whitespace is always ignored.
- **Follow-up questions:** Effect of a trailing slash? (`url()` strips it.)

### Q78. Where would an incorrect trailing slash in `plines.api.base-url` bite?
- **Why the interviewer asks:** URL concatenation bugs.
- **Ideal answer:** `plines.api.base-url` (no trailing slash in the file) is concatenated by the `Pline*Client`s with a path starting with `/`. If someone adds a trailing slash, the result could contain `//ext/...`; some servers tolerate it, others 404. `PlineUtilsClient.url()` strips a trailing slash for `API_BASE-URL`, but I did not confirm every `Pline*Client` does the same [CONFIRM by reading them].
- **Common mistakes:** Assuming all clients normalise.
- **Follow-up questions:** Safer approach? (`UriComponentsBuilder`.)

### Q79. What is `workmen.api.base-url` and do you need it?
- **Why the interviewer asks:** Scope control.
- **Ideal answer:** Line 187, dev2 host, added by `b928bc28d` (2026-06-04), for the Workmen LOB clients. The three modules do not use it, but a promotion checklist for the whole service includes it because it is in the same jar. Do not change it for a Plines-only fix.
- **Common mistakes:** Editing unrelated LOB config.
- **Follow-up questions:** Does RejectRisk have a Workmen variant? (Yes, separate bean `AGIC/UW/Workmen/RejectRisk/RejectRiskReport`.)

### Q80. How can configuration drift between the repo and the platform cause an incident?
- **Why the interviewer asks:** Operations mindset.
- **Ideal answer:** The repo has defaults baked into the jar; Helm overrides some. After `7dd2f2f67` removed the URI from the committed file, a pod without an external override starts with no Mongo URI and falls back to localhost (or fails), while old images had one. Also a changed key name in the repo (for example `spring.mongodb.*`) needs the matching key in Helm. Mitigation: a config contract document, startup validation (Q43), and deploy checklists.
- **Common mistakes:** Assuming repo equals production.
- **Follow-up questions:** How to detect drift? (Log resolved non-secret values at startup; compare with expected.)

### Q81. What changes in rollback when config is part of the image?
- **Why the interviewer asks:** Release safety.
- **Ideal answer:** Redeploying the previous image restores its baked-in defaults, but externally supplied values (Helm) persist, which can be incompatible (for example the Mongo key prefix changed between versions: old image reads `spring.data.mongodb.*`, new reads `spring.mongodb.*`). So rollbacks across the Boot 4 upgrade need the Helm values matched to the old image. Stateless service means no data rollback.
- **Common mistakes:** Assuming rollback is always safe.
- **Follow-up questions:** How to prepare? (Support both keys during a transition window, if feasible.)

### Q82. What does `spring.application.name=Documentprintdata` influence?
- **Why the interviewer asks:** Small but real knowledge.
- **Ideal answer:** Application name in logs/metrics tags and actuator info if exposed; it can feed service registries and tracing if present (not here). Changing it affects dashboards that group by name [CONFIRM]. Low risk.
- **Common mistakes:** Mixing it with `artifactId` (`gi-documentprintdata` in the pom, with stray spaces).
- **Follow-up questions:** What is the image/app name in CI? (`opus-gi-documentprintdata-service`.)

### Q83. Why might `management.endpoint.info.enabled` matter?
- **Why the interviewer asks:** Info endpoint content.
- **Ideal answer:** `/actuator/info` is open and returns whatever info contributors exist (build info/git info only if configured; none set in my notes). By default it returns an empty object. If someone adds git properties or env info, it could leak version and branch details to anyone on the network. Review before enabling contributors.
- **Common mistakes:** Assuming info is always empty.
- **Follow-up questions:** Use for deployment verification? (Enable `build-info` goal to show image version.)

### Q84. How can you verify the effective configuration of a running pod without actuator `env`?
- **Why the interviewer asks:** Operational debugging.
- **Ideal answer:** Options: startup log lines of resolved non-secret values (add), `kubectl exec` and inspect env vars / mounted files, a temporary DEBUG for `org.springframework.boot.context.config`, or run the same image locally with the same env. Exposing `env` is dangerous (values) unless sanitised and authenticated. In this repo none of these helpers exist [CONFIRM: platform tooling].
- **Common mistakes:** Exposing `/actuator/env` in prod.
- **Follow-up questions:** Spring's sanitising of keys? (Masks keys containing password/secret by default in older versions; behaviour changed in newer versions; verify.)

### Q85. How would you manage the truststore across environments?
- **Why the interviewer asks:** Cert lifecycle.
- **Ideal answer:** Move from classpath to a file path property (for example `app.ssl.trust-store-location` accepting `file:`), mount from a secret/ConfigMap, load via `DefaultResourceLoader` so it supports both `classpath:` and `file:`; password via env. Add a startup log of certificate aliases and expiry dates. Plan rotation before expiry.
- **Common mistakes:** Rebuilding the image for every CA change.
- **Follow-up questions:** Backward compatibility? (Keep the classpath default.)

### Q86. A new upstream returns an `SSLHandshakeException`. What do you check?
- **Why the interviewer asks:** Using config knowledge in an incident.
- **Ideal answer:** (1) Which RestTemplate the client uses (all use the custom truststore). (2) Whether the upstream certificate chain's issuing CA exists in `mytruststore.p12` (`keytool -list -keystore ... -storetype PKCS12`, entering the password from the secret store, never printing it in chat). (3) Hostname vs certificate SAN. (4) Add the CA to a new p12, rebuild/override. (5) Remember fail-soft returns blanks, so users see missing data rather than errors.
- **Common mistakes:** Disabling certificate validation.
- **Follow-up questions:** How to test in dev quickly? (`openssl s_client -connect host:443 -showcerts`.)

### Q87. How do `ext.pme.retry.*` keys appear twice?
- **Why the interviewer asks:** Reading duplicates.
- **Ideal answer:** `ext.pme.retry.max-attempts` and `backoff-ms` are defined at lines 160-161 and also appear as comments at 116-117. Only the active ones matter; they are read by motor `PMEGetContractIdClient`, not Pline clients. The pline clients have no retry config.
- **Common mistakes:** Assuming retry applies to all external calls.
- **Follow-up questions:** Would you add retry to Pline clients? (Maybe 1 retry with short backoff for idempotent GET after measuring.)

### Q88. How do you protect configuration from accidental logging?
- **Why the interviewer asks:** Secret leakage via logs.
- **Ideal answer:** Never log `Environment` dumps or full URIs; log only hosts. Avoid `toString()` of config beans with secrets; mark fields with `@ToString.Exclude` (Lombok). Keep Boot's key sanitising for actuator `env`. In this repo `MongoTemplate` DEBUG logs queries (data), not the URI; the URI may still appear in driver connection logs at DEBUG/INFO without credentials (verify).
- **Common mistakes:** Printing config in a ticket for help.
- **Follow-up questions:** Review step? (grep logs for "mongodb://" before sharing.)

### Q89. What would you add to `.gitignore` or tooling to prevent committing local config?
- **Why the interviewer asks:** Prevention.
- **Ideal answer:** Ignore `application-local.properties` and `*.local.properties`; keep local overrides there; document in README. Add a pre-commit hook running gitleaks locally on staged files. Since the CI allowlist exempts the main properties files, the local hook is the real safeguard for those files.
- **Common mistakes:** Relying only on CI.
- **Follow-up questions:** Can the hook be bypassed? (`--no-verify`; policy and CI back it up.)

### Q90. Explain `logging.level.org.springframework.data.mongodb.core.aggregation.Aggregation=DEBUG`.
- **Why the interviewer asks:** Knowing what logs what.
- **Ideal answer:** Aggregation helper class logging, in addition to the `MongoTemplate` logger which prints the pipeline when executing. Both at DEBUG produce duplicates of the same pipeline text in some cases (inferred). Useful to copy a pipeline into mongosh; heavy in prod.
- **Common mistakes:** Enabling only one and wondering why the pipeline is missing.
- **Follow-up questions:** Which logger prints the pipeline? (`MongoTemplate` per the KT note.)

### Q91. Why does the pom have `<artifactId> gi-documentprintdata </artifactId>` with spaces and does it matter?
- **Why the interviewer asks:** Attention to detail.
- **Ideal answer:** Maven trims the whitespace in most places, so the build works; it is cosmetic. It can cause confusion in tools that read raw XML. Not a risk, but a clean-up candidate. The jar name and final name should be checked in CI if scripts use them [CONFIRM].
- **Common mistakes:** Treating it as a build bug.
- **Follow-up questions:** Where else is the app name defined? (`spring.application.name`, CI `app_name`.)

### Q92. Which configuration items are risky to change at runtime on a single pod for testing?
- **Why the interviewer asks:** Safe experimentation.
- **Ideal answer:** `server.servlet.context-path` (breaks probes/callers), `spring.mongodb.database` (empty reports), `spring.http.converters.preferred-json-mapper` (breaks the API), the truststore keys (startup failure), `plines.api.base-url` (wrong environment). Low-risk: logging levels (when changed through a redeploy), springdoc paths. Always test in dev first.
- **Common mistakes:** Experimenting on a shared UAT pod.
- **Follow-up questions:** How do you isolate? (Local run with env overrides.)

### Q93. What is the config story for the XSD files in tests?
- **Why the interviewer asks:** Link to XSD testing.
- **Ideal answer:** `src/test/resources/xsd/plines/Pline.xsd` and `Common.xsd` are the only schema copies in the repo (identical to the v0.1.6 reference apart from line endings). `PlineXsdValidationTest` loads them from the classpath and resolves the `xs:include` by converting the URL to a `File`. There is no property that selects a schema version; a schema upgrade is a file replacement plus a test update.
- **Common mistakes:** Assuming production code validates against these files (it does not).
- **Follow-up questions:** Where would a runtime validation toggle live? (A feature flag property such as `report.validate-xsd`; does not exist today.)

### Q94. Would you add a feature toggle for validating output against the XSD?
- **Why the interviewer asks:** Design judgement tying XSD with config.
- **Ideal answer:** Possibly in non-prod only: `plines.xsd.validate=true` runs validation after serialisation and logs errors (no failure), giving visibility without blocking production. Because real output fails v0.1.6 today (dates, empty elements, order), enforcing would block everything. Start with log-only in dev/UAT, then ratchet. It needs the XSD on the main classpath, not only `src/test`.
- **Common mistakes:** Making it blocking from day one.
- **Follow-up questions:** Performance cost? (Validation of large XML per request; sample only.)

### Q95. How do you keep local runs from hitting real UAT/prod data?
- **Why the interviewer asks:** Safety.
- **Ideal answer:** Run with a local or dev Mongo URI via env, not the committed values; keep outbound base URLs pointing to dev2 (the committed default). Because the service is read-only, the worst case is data exposure and load, not corruption. Verify the DB name printed at startup before posting requests. The working copy's production-clone-looking DB name (`OPUSAGICPRODR1`, inferred) is a reminder to double-check.
- **Common mistakes:** Pointing a debug session at a production-like DB with real PII.
- **Follow-up questions:** Any write path? (None found in these modules.)

### Q96. What is your view on keeping legacy `motor` and new `plines` RestTemplate configs side by side?
- **Why the interviewer asks:** Architecture trade-off.
- **Ideal answer:** `RestTemplateConfig` (motor, `@Primary`, only connect/read/socket) and `PlineRestTemplateConfig` (pool limits, connection-request timeout, fast-lookup) exist side by side because the Pline fix was added without touching shared LOB behaviour (Phase 1/2 in its header). It limits blast radius but creates the unqualified-injection trap (Q22). Longer-term unify with one config and named beans per use case.
- **Common mistakes:** Merging them in one PR with other changes.
- **Follow-up questions:** How to test timeouts? (MockWebServer with delayed responses.)

### Q97. Which of the config issues would you fix first and why?
- **Why the interviewer asks:** Prioritisation.
- **Ideal answer:** (1) Remove/relocate secrets and avoid committing the local Mongo block (security). (2) Qualify the three Pline clients to use the pline RestTemplate (reliability). (3) Set Mongo pool/timeouts in the URI instead of dead `option.*` (performance). (4) Reduce prod logging (Mongo DEBUG, `System.out`). (5) Clean dead keys and unify base URL (maintainability). Order by risk times effort.
- **Common mistakes:** Starting with clean-up before security.
- **Follow-up questions:** What needs another team? (Helm values, secrets store.)

### Q98. What do you tell a new joiner about "which config key do I change?" for a URL problem?
- **Why the interviewer asks:** Communication of the key trap.
- **Ideal answer:** First find which class makes the call, then which key it reads (`@Value`). For `PlineUtilsClient`: `API_BASE-URL` + the specific path key. For `PlinePolicyRefClient`, `PlineSSTRolloutDateClient`, `PlinePgRpgeUtilsClient`: `plines.api.base-url` + path key. Search duplicates, but change only the key that the class reads. Test in dev by calling the report with a known contract and checking logs.
- **Common mistakes:** Changing every similar key.
- **Follow-up questions:** How do you find the reader? (IDE "find usages" of the property string.)

### Q99. How do you explain `spring.profiles.active=dev` to a manager who thinks it separates environments?
- **Why the interviewer asks:** Communication and risk.
- **Ideal answer:** "It is only a label. There are no profile-specific files, so setting it to UAT or prod would change nothing. Environment differences are carried by a handful of individual settings: two API base URLs and the database address. We should either create real profile files or remove the label so nobody relies on it." Add the proposal for startup logging of resolved hosts.
- **Common mistakes:** Over-technical wording.
- **Follow-up questions:** What would you ask the manager? (Time to introduce profiles and a config checklist.)

### Q100. Give a 60-second summary of the service's configuration risks.
- **Why the interviewer asks:** Closing question; tests synthesis.
- **Ideal answer:** "One jar carries all LOBs and its config. Secrets live in tracked property files and a committed truststore; the gitleaks allowlist exempts those files. Environment selection is spread across at least five keys and the profile does nothing. Several keys are dead (option.*, auto-index-creation, many dev1 URLs), and the Mongo key prefix changed in Boot 4. Timeouts exist but two Pline clients get a different RestTemplate than intended. Logging prints PII through Mongo DEBUG and System.out. Fixes are small: externalise secrets, qualify beans, put pool options in the URI, trim logs, and add startup validation." Mention that deployment details need the platform team [CONFIRM].
- **Common mistakes:** Listing 20 keys without a message.
- **Follow-up questions:** Which one did you personally fix? [CONFIRM: your actual contribution.]
