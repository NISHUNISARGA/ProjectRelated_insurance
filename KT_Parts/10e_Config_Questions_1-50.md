# 10e - Configuration Questions 1-50 (opus-documentprintdata-service)
> **AUTHORSHIP UPDATE (confirmed by the developer): git author `GS <extern.gs_nisarga@allianz.com>` is the developer. Wherever this file says commits cannot be attributed to the developer, read it as: the 11 GS commits listed in `KT_Parts/14_My_Commits_And_Ownership.md` ARE theirs; commits by other authors still must not be claimed. No GS commit touches `src/test`, so test authorship is still `[CONFIRM]`.**

Evidence base: `06_Config_API_DB_DevOps.md` Part 1 (config catalog with line numbers), `application.properties` (251 lines), `application-env.properties` (3 lines), `pom.xml`, `RestTemplateConfig`, `PlineRestTemplateConfig`, `PlineUtilsClient`. "(inferred)" = reasoning, not proven. Where I cannot see the deployment (Helm values, environment variables), I say so. Personal facts are [CONFIRM: ...]. Secrets (Mongo URI, user, password, truststore password) are never printed: `<redacted>`.

---

### Q1. Which configuration files exist, and where does each value come from at runtime?
- **Why the interviewer asks:** Basic ownership of config; many candidates cannot name the files.
- **Ideal answer:** In `src/main/resources`: `application.properties` (251 lines), `application-env.properties` (3 lines, SSL only), and the binary `mytruststore.p12` on the classpath. `src/test/resources` has only `mockito-extensions` and `xsd` (no test properties). No config server, `bootstrap`, or `spring.config.import`. Everything is baked into the jar, so the deployed values are either those or whatever the platform overrides (Helm is outside this repo) [CONFIRM: how overrides are supplied].
- **Common mistakes:** Assuming a config server or Vault exists.
- **Follow-up questions:** Where does Mongo URI come from in a deployed pod? (Not visible in repo.)

### Q2. `spring.profiles.active=dev` is set. What does it do?
- **Why the interviewer asks:** A trap: looks like environment separation but is not.
- **Ideal answer:** Nothing profile-specific. No `application-dev*.properties` exists and no `@Profile` or `@ConditionalOn*` was found in `src/main/java`. It is only a label. If a deployment sets `SPRING_PROFILES_ACTIVE=uat`, that also activates nothing, giving a false sense of environment separation.
- **Common mistakes:** Believing dev profile switches DB or URLs.
- **Follow-up questions:** How would you introduce real profiles? (Create `application-uat.properties` etc. with only non-secret differences; secrets via env.)

### Q3. Is `application-env.properties` a profile file?
- **Why the interviewer asks:** Naming convention misunderstanding.
- **Ideal answer:** No. A profile file would be `application-<profile>.properties` loaded only if profile `env` is active. This one is loaded explicitly with `@PropertySource("classpath:/application-env.properties")` on `RestTemplateConfig` (motor) and `PlineRestTemplateConfig` (plines). It is therefore always loaded for those beans, and `@PropertySource` has lower precedence than `application.properties`, environment variables and system properties.
- **Common mistakes:** Thinking it activates with `spring.profiles.active=env`.
- **Follow-up questions:** Could an env var override `app.ssl.trust-store-password`? (Yes, higher precedence.)

### Q4. List ways to override a property for one environment, in order of precedence.
- **Why the interviewer asks:** Practical deployment control.
- **Ideal answer:** Highest first (Spring Boot standard): command-line args (`--spring.mongodb.uri=...`), JVM system properties (`-D...`), OS environment variables (`SPRING_MONGODB_URI`), external config files (`spring.config.additional-location`), then packaged `application.properties`, then `@PropertySource`. Which one the real deployment uses is not in the repo [CONFIRM]. I would recommend env vars from a secret store for URI/credentials.
- **Common mistakes:** Editing the packaged file per environment.
- **Follow-up questions:** Why not rebuild per environment? (Immutable image principle.)

### Q5. What does `server.servlet.context-path=/gi/documentprintdata` change?
- **Why the interviewer asks:** Real operational impact.
- **Ideal answer:** All URLs get the prefix: report API `/gi/documentprintdata/api/opus/GenerateReport`, health `/gi/documentprintdata/actuator/health`, Swagger under the same prefix. Callers, ingress rules and probes must include it. Removing it moves everything to `/` and breaks every caller. Risk rating: very high.
- **Common mistakes:** Probing `/actuator/health`.
- **Follow-up questions:** Does `SecurityConfig` matcher use the prefix? (Matchers are relative to the context path.)

### Q6. `server.port=8080`: where else must 8080 be right?
- **Why the interviewer asks:** Config consistency across layers.
- **Ideal answer:** Container port, Kubernetes service `targetPort`, and probes (Helm, outside the repo). Changing only `server.port` breaks readiness: the pod stays not ready. The Boot default is also 8080, so removing the property has no effect.
- **Common mistakes:** Changing one place only.
- **Follow-up questions:** Can you run two instances locally? (Use `--server.port=`.)

### Q7. Why is the Mongo key `spring.mongodb.uri` and not `spring.data.mongodb.uri`?
- **Why the interviewer asks:** The Boot 4 upgrade trap your team hit.
- **Ideal answer:** In Boot 4, `MongoProperties` moved to `org.springframework.boot.mongodb.autoconfigure` with prefix `spring.mongodb`. Old keys are silently ignored and the app falls back to `localhost:27017`. Commit `9248b12ff` (18 Sep 2026) added the new keys; comments in lines ~26-31 explain it.
- **Common mistakes:** Debugging network access to the cluster.
- **Follow-up questions:** How to prove which URI is active safely? (Log host names only, never credentials.)

### Q8. What if `spring.mongodb.database` points to the wrong database?
- **Why the interviewer asks:** Silent failure.
- **Ideal answer:** The connection works, but all aggregations return zero rows with no error, so reports come out empty with HTTP 200. The working copy sets a name that looks like a production clone (`OPUSAGICPRODR1`, inferred from the string); the committed history removed the UAT lines (`7dd2f2f67`), so the deployed DB is chosen outside this repo.
- **Common mistakes:** Looking for exceptions.
- **Follow-up questions:** How to add a startup sanity check? (Count documents in `CMGE_CODE` and log the DB name.)

### Q9. Are `option.min-connection-pool-size`, `option.max-connection-pool-size` and `option.max-wait-time` effective?
- **Why the interviewer asks:** Detecting dead config.
- **Ideal answer:** No. They are not Boot keys and no `@Value` reads them (introduced in the "Performance issue fixes." commit `62b3c94a7`, 2026-01-28). Driver defaults apply (maxPoolSize 100, minPoolSize 0, maxWait 2 min). To set pool values, add `maxPoolSize`, `minPoolSize`, `waitQueueTimeoutMS` to the connection string (inferred; verify against Boot 4 docs).
- **Common mistakes:** Quoting 20/100/2000 as the live pool.
- **Follow-up questions:** Why did the author expect it to work? (Copied from another project style or earlier Boot setup; inferred.)

### Q10. Is `auto-index-creation=false` effective?
- **Why the interviewer asks:** Same class of question as Q9 with index ownership.
- **Ideal answer:** No: wrong prefix (the real key would be `spring.data.mongodb.auto-index-creation`) and there are no `@Document`/`@Indexed` entities anyway. Indexes are managed in the cluster. Nothing in the repo creates indexes.
- **Common mistakes:** Believing index creation is disabled by it.
- **Follow-up questions:** Who creates indexes? [CONFIRM: DBA/Atlas.]

### Q11. Why `spring.http.converters.preferred-json-mapper=jackson2`?
- **Why the interviewer asks:** Upgrade knowledge with real consequence.
- **Ideal answer:** Boot 4.1.1 prefers Jackson 3 (`tools.jackson.*`), which ignores `com.fasterxml.jackson` annotations used by `PayLoadRequest` and DTOs. Without the property POSTs fail with `HttpMessageConversionException`. Commit `9248b12ff`. The property is on line 17 with explanation lines 7-16.
- **Common mistakes:** Removing it as redundant.
- **Follow-up questions:** What is the long-term plan? (Migrate annotations to Jackson 3 imports, with XML output regression tests.)

### Q12. Why does `pom.xml` set `jackson-2-bom.version` and not `jackson-bom.version`?
- **Why the interviewer asks:** CVE pin mechanics under Boot 4.
- **Ideal answer:** Under Boot 4.1.1 the property `jackson-bom.version` controls Jackson 3; Jackson 2 uses `jackson-2-bom.version`. The pom pins `2.21.5` with CVE comments (CVE-2026-54512/54513 as quoted in the pom, not independently verified). Using the wrong property would silently not apply the pin.
- **Common mistakes:** Pinning the old property name.
- **Follow-up questions:** How to verify the resolved version? (`mvn dependency:tree`.)

### Q13. `JacksonConfig` defines a plain `ObjectMapper`. Does `spring.jackson.*` still work?
- **Why the interviewer asks:** Bean override effects.
- **Ideal answer:** `motor/config/JacksonConfig` defines an `ObjectMapper` with `JavaTimeModule`. Properties such as `spring.jackson.default-property-inclusion` configure the auto-configured mapper builder, not a hand-built mapper, so they would not affect this bean. None are set today. The controller's `XmlMapper` is a separate `new XmlMapper()` that ignores all Spring Jackson properties, which is why the XML null handling cannot be changed by config.
- **Common mistakes:** Trying `spring.jackson` to fix empty XML elements.
- **Follow-up questions:** Where would you fix XML inclusion? (Shared `XmlMapper` bean with `setDefaultPropertyInclusion`.)

### Q14. What does `spring.messages.basename=messages` do?
- **Why the interviewer asks:** Detecting harmless stale keys.
- **Ideal answer:** Points the MessageSource to `messages*.properties`, but none exist in resources. Boot tolerates a missing bundle, so it is harmless and stale. Risk none; clean-up candidate.
- **Common mistakes:** Claiming i18n is supported.
- **Follow-up questions:** Does the service localise text? (Texts come from data/Jasper side; not from message bundles.)

### Q15. Which logging keys are stale and why?
- **Why the interviewer asks:** Reading config critically.
- **Ideal answer:** `logging.level.com.agcs.fo.opusMotorGateway.service=DEBUG` and `...GrouptaxinvoiceService=INFO` target a package that does not exist (real root is `com.agcs.fo.opus`, main class `motor/OpusMotorGatewayApplication`). `logging.level.service.com.agcs.fo.opus.motor.DataDownloadingService=DEBUG` has wrongly ordered segments (real: `com.agcs.fo.opus.motor.service.DataDownloadingService`). They do nothing.
- **Common mistakes:** Thinking they enable useful logs.
- **Follow-up questions:** How to log my handlers at DEBUG? (`logging.level.com.agcs.fo.opus.plines.documentHandler=DEBUG`.)

### Q16. Which logging keys are active and what is the risk?
- **Why the interviewer asks:** PII and performance.
- **Ideal answer:** Active: `org.springframework.data.mongodb.core.MongoTemplate=DEBUG` and `...core.aggregation.Aggregation=DEBUG`. They log each Mongo operation including values (contract ids, names, ID numbers). In prod they slow requests and leak identifiers. Recommend INFO in prod and a controlled way to raise it.
- **Common mistakes:** Leaving DEBUG "just in case".
- **Follow-up questions:** Why can't you change it at runtime? (`loggers` endpoint not exposed.)

### Q17. `management.endpoint.loggers.enabled=true` is set. Can you change log levels at runtime?
- **Why the interviewer asks:** Exposure versus enablement.
- **Ideal answer:** No. Enabled is not exposed: `management.endpoints.web.exposure.include=health,info` leaves out `loggers`, and `SecurityConfig` denies other `/actuator/**`. So the setting has no effect (inferred mismatch: someone wanted it). Exposing it would need adding `loggers` to the include list and authenticating it.
- **Common mistakes:** Thinking enabled means reachable.
- **Follow-up questions:** Security implication of exposing it? (Anyone could raise DEBUG and flood logs; since API has no auth, protect it.)

### Q18. What does health expose and what are the probe implications?
- **Why the interviewer asks:** Deployment readiness.
- **Ideal answer:** `/gi/documentprintdata/actuator/health` shows overall status only (`show-details` not set = never). `management.endpoint.health.probes.enabled` is not set, so separate liveness/readiness groups are not enabled by Boot; Helm probably uses the main health URL (inferred, Helm not in repo). A Mongo outage can make health DOWN if the Mongo indicator is active (inferred default).
- **Common mistakes:** Using health as a deep dependency check without knowing what it includes.
- **Follow-up questions:** Should liveness depend on Mongo? (No; use readiness for dependencies.)

### Q19. How are springdoc and Swagger configured, and should they be on in prod?
- **Why the interviewer asks:** Exposure control.
- **Ideal answer:** Keys: `springdoc.api-docs.path=/v3/api-docs`, `springdoc.swagger-ui.path=/swagger-ui/index.html`, `operationsSorter=method`, `tagsSorter=alpha`; `OpenApiConfig` adds group `public` for `/api/**`. `anyRequest().permitAll()` makes them open. In prod I would disable (`springdoc.api-docs.enabled=false`, `springdoc.swagger-ui.enabled=false`) or protect them.
- **Common mistakes:** Believing Swagger requires auth by default.
- **Follow-up questions:** Any data exposed? (Only the endpoint shape.)

### Q20. Walk through `app.http.*` keys and their values.
- **Why the interviewer asks:** Timeout configuration is a core piece of your outbound design.
- **Ideal answer:** `connect-timeout-ms=5000`, `read-timeout-ms=20000` (HttpClient5 `responseTimeout`), `socket-timeout-ms=20000`, `connection-request-timeout-ms=3000` (pline only), `max-conn-total=100` and `max-conn-per-route=20` (pline only), `fast-lookup.read-timeout-ms=8000` (read+socket for the `fastLookupRestTemplate`). Introduced by `57c573b74` (2025-12-06) and `fd3d73383`/`4cd1cad2e` for the pline pieces.
- **Common mistakes:** Quoting them without knowing which bean uses which.
- **Follow-up questions:** What are the code defaults? (Motor 10000/120000/120000; pline 5000/20000/20000.)

### Q21. Why does the code have defaults different from the properties file?
- **Why the interviewer asks:** Behaviour if a key is removed.
- **Ideal answer:** `@Value("${app.http.read-timeout-ms:120000}")`-style defaults in `RestTemplateConfig` are old legacy values (120 s). The properties file overrides them with 20 s. If someone removes the properties line, the motor bean silently returns to 120 s. The pline config documents a "Phase 1 incident fix - bounded timeouts".
- **Common mistakes:** Treating the code default as dead.
- **Follow-up questions:** What incident? [CONFIRM: details of the incident; the repo only says "Phase 1 incident fix".]

### Q22. Which RestTemplate does each Pline client use and what does the config imply?
- **Why the interviewer asks:** Linking config to injection.
- **Ideal answer:** `PlineUtilsClient` -> `fastLookupRestTemplate` (8 s). `PlinePolicyRefClient`, `PlineSSTRolloutDateClient`, `PlinePgRpgeUtilsClient` have an unqualified `RestTemplate` parameter, so they get the `@Primary` motor `restTemplate` (5 s connect, 20 s read, no connection-request timeout). Fix with `@Qualifier("plineRestTemplate")`.
- **Common mistakes:** Assuming qualifier-by-name.
- **Follow-up questions:** Effect on pooling? (Motor bean uses library-default pool.)

### Q23. Explain `app.ssl.*` keys and the startup failure modes.
- **Why the interviewer asks:** Secrets and startup behaviour.
- **Ideal answer:** `app.ssl.trust-store=mytruststore.p12` (classpath), `app.ssl.trust-store-password=<redacted>` (plaintext in VCS since `57c573b74`), `app.ssl.trust-store-type=PKCS12` (default PKCS12). Read via `@Value` without defaults for the first two. Wrong password -> `IOException: keystore password was incorrect` at bean creation = startup failure. Removed key -> startup failure.
- **Common mistakes:** Believing password failure shows up at first outbound call.
- **Follow-up questions:** How to rotate the password? (Rebuild p12, update properties/secret, rebuild jar.)

### Q24. What does the custom truststore do to the JVM default trust?
- **Why the interviewer asks:** TLS reasoning.
- **Ideal answer:** `SSLContexts.custom().loadTrustMaterial(trustStore, null)` builds trust from only this file. For the two RestTemplate families it replaces the JVM default cacerts, so every upstream CA must be inside the p12 or the handshake fails. Mongo uses the JVM default trust (`ssl=true`), not this file.
- **Common mistakes:** Saying the p12 adds to default trust.
- **Follow-up questions:** Symptom of a missing CA? (`SSLHandshakeException`; fail-soft clients then return blanks.)

### Q25. What are the operational problems of a classpath truststore?
- **Why the interviewer asks:** Config-as-code consequences.
- **Ideal answer:** Any CA rotation requires a code rebuild and a new image; the binary and password are committed; different environments with different CAs need different builds or an override path (for example a file path property pointing outside the jar; the code uses `ClassPathResource`, so this is not supported today). Better: mount truststore as a secret volume and use a file path.
- **Common mistakes:** Treating it as only a security issue and not an operations one.
- **Follow-up questions:** Could JVM `-Djavax.net.ssl.trustStore` replace it? (Not with this code, which builds its own context.)

### Q26. What exactly is `API_BASE-URL` and who reads it?
- **Why the interviewer asks:** The central environment key.
- **Ideal answer:** Line 76: `https://dev2-apiservice.opusm.srv.allianz/` (note leading space after `=` and trailing `/`). Read by about 45 `@Value("${API_BASE-URL}")` sites, including `PlineUtilsClient` (line ~31), Fire/Bond/Motor handlers, `SharedReportUtilsClient`, `WorkmenUtilsClient`. Added in commit `c4407d66c` (2026-03-08). `PlineUtilsClient.url()` strips the trailing slash before appending the path.
- **Common mistakes:** Calling it the only base URL (it is not; see Q28).
- **Follow-up questions:** Why does a leading space not matter? (`Properties` trims leading whitespace.)

### Q27. Is the hyphen in `API_BASE-URL` a problem for env vars?
- **Why the interviewer asks:** Name-binding detail.
- **Ideal answer:** POSIX shells cannot export names with `-`; Kubernetes `env:` can carry arbitrary names. Spring's `SystemEnvironmentPropertySource` maps names by replacing `.` and `-` with `_` and trying upper-case, so `API_BASE_URL` should resolve `${API_BASE-URL}` (inferred, not tested in this repo). Lowest-risk override: JVM arg `-DAPI_BASE-URL=https://...` or a mounted config file. Verify by starting with the override and watching the outbound URL.
- **Common mistakes:** Stating relaxed binding as proven.
- **Follow-up questions:** How would you test it? (Start jar with env var and call an endpoint that logs the URL.)

### Q28. Why must `plines.api.base-url` be changed separately?
- **Why the interviewer asks:** Multi-key environment switch.
- **Ideal answer:** `PlinePolicyRefClient`, `PlineSSTRolloutDateClient` and `PlinePgRpgeUtilsClient` read `plines.api.base-url` (line 227, added `d4c069135`, 2026-07-16, no default). Same host as `API_BASE-URL` today, by coincidence. If you flip only `API_BASE-URL` to UAT, those three still call dev2. Minimum set for promotion: `API_BASE-URL`, `plines.api.base-url`, `workmen.api.base-url`, `spring.mongodb.uri`, `spring.mongodb.database`.
- **Common mistakes:** Changing one URL and assuming all calls moved.
- **Follow-up questions:** How to unify? (Single key or a derived default `${plines.api.base-url:${API_BASE-URL}}`.)

### Q29. dev1 hosts vs dev2 hosts: which are live?
- **Why the interviewer asks:** Stale config reading.
- **Ideal answer:** Live (read by code): dev2 in lines 76, 187, 227. dev1 keys (113 `external.pme.base-url`, 121 `pg-rpge-utils.base-url`, 125 `ext.sst-rollout-date.base-url`, 128 `ext.prnt-ptv-trans.base-url`, 139 `pg-uwge-utils.base-url`) have no `@Value` reader found (grep over `src/main/java`; `Environment.getProperty` calls searched, none). They mislead but are inert.
- **Common mistakes:** Deleting the active ones.
- **Follow-up questions:** How to prove unused? (Search `${...}`, `@ConfigurationProperties`, `getProperty`, reflection.)

### Q30. Which keys are defined but read by nobody?
- **Why the interviewer asks:** Dead-config awareness.
- **Ideal answer:** `external.pme.base-url`, `pg-rpge-utils.base-url`, `pg-rpge-utils.timeout-ms`, `pg-uwge-utils.base-url`, `ext.sst-rollout-date.base-url`, `ext.prnt-ptv-trans.base-url`, `sharedreport.fn.email.not.polstatus.path` (singular; code reads the plural), `auto-index-creation`, `option.*`, `spring.messages.basename`, `management.endpoint.loggers.enabled`. Based on grep of `@Value` placeholders; reflection or non-grep access is not excluded.
- **Common mistakes:** Claiming certainty beyond the grep.
- **Follow-up questions:** Safe clean-up method? (Remove, run tests and start-up, deploy to dev first.)

### Q31. Which path keys do the three modules actually use?
- **Why the interviewer asks:** Focus on your scope.
- **Ideal answer:** `plines.fn.comp.brn.number.path`, `plines.fn.rppm.get.signature.path` (+ signature type default `UW`), `plines.fn.show.stax.inv.ind.path`, `plines.sst.rollout.date.path`, `plines.policy.ref.pme.path`, `opus.fn-einvoice-print-endt-ind-path`, `opus.fn-einvoice-print-pol-ind-path`, `opus.fn-get-co-inward-ind-path`; e-invoice QR via its own client. RejectRisk uses none (no outbound).
- **Common mistakes:** Listing all 40 path keys.
- **Follow-up questions:** Which has no default and fails startup? (`opus.fn-decide-mt-cert-path`, `opus.fn-uwge-get-endtschd-query-path` required by `PlineUtilsClient`, though used by other modules.)

### Q32. `plines.api.customer.signature.type` is not defined. What happens?
- **Why the interviewer asks:** Defaults in code.
- **Ideal answer:** `PlineUtilsClient` (line ~57) reads `${plines.api.customer.signature.type:UW}` and the key is absent from the properties, so the default `UW` applies. It is different from `api.customer.signatureType=UW` (line 111), which motor code reads. Someone changing line 111 expecting Plines to follow would see no change.
- **Common mistakes:** Merging the two keys.
- **Follow-up questions:** How would you make the default visible? (Add the key to properties with the value.)

### Q33. Which path was derived rather than confirmed?
- **Why the interviewer asks:** Risk awareness on config provenance.
- **Ideal answer:** `plines.fn.date.value.constants.path=/ext/utils/api/v1/pg-util-general/get-date-value-constants` (line 237); comment lines 234-236 say the path is derived/guessed and to verify if it starts 404'ing. It is the highest-risk path in the file. Whether the three modules call it is not shown in my notes [CONFIRM usage].
- **Common mistakes:** Presenting every path as confirmed.
- **Follow-up questions:** How to verify? (Compare with the API team's catalog; call it in dev.)

### Q34. Compare the three stax-invoice indicator keys.
- **Why the interviewer asks:** Duplicates with variations.
- **Ideal answer:** `plines.fn.show.stax.inv.ind.path=/ext/policy/api/v1/stax-inv` (247, used by Template12 handlers), `opus.fn-show-stax-inv-ind-path=/ext/policy/api/v1/pg-uwge-utils/stax-inv-indicator` (138, older) and `workmen.fn.show.stax.inv.ind.path=/ext/policy/api/v1/stax-inv` (202). Same business question, different keys and one different path. Which is "right" depends on the backend; the Plines handlers read the plines key.
- **Common mistakes:** Changing the plines key because another key looks more official.
- **Follow-up questions:** How would you reduce duplicates? (Alias properties to one source.)

### Q35. Explain the two policy-ref configurations and their conflict.
- **Why the interviewer asks:** Careful config analysis.
- **Ideal answer:** `plines.policy.ref.path` (228) -> `/ext/policy/api/v1/pg-uwge-utils/get-policy-ref`; `plines.policy.ref.pme.path` (223) -> `/ext/policy/api/v1/pme-api/get-policy-ref` (placed under the Marine heading). ASP ServiceTaxInvoice uses the PME one. Another key, `pg-uwge-utils.path.get-policy-ref` (163), has no `/ext/policy` prefix and an underscore, read by motor `PolicyRefClient`; likely a 404 (inferred).
- **Common mistakes:** Using the wrong one in a new feature.
- **Follow-up questions:** Who decides the right backend? [CONFIRM: API team.]

### Q36. `pme-api.path.get-contract-id` has the value `.../get-policy-ref`. Problem?
- **Why the interviewer asks:** Name vs value mismatch.
- **Ideal answer:** The key says contract-id, the value points to policy-ref (line 164). Anyone reading code with that key believes they call a contract-id API. Could also be intentional if the same endpoint returns both; not documented. Rename or fix after confirming with the owner.
- **Common mistakes:** "Fixing" the value to a guessed URL.
- **Follow-up questions:** Which class reads it? (Motor `PMEGetContractIdClient`, with retry keys `ext.pme.retry.*`.)

### Q37. Why is `opus.fnGetCodeDesc.path=ext/utils/...` risky?
- **Why the interviewer asks:** Slash handling.
- **Ideal answer:** No leading slash (line 174). `PlineUtilsClient.url()` concatenates `base (without trailing slash) + path`, so a path without a slash gives `...allianzext/utils`. That key is read by motor `FnGetCodeDescClient`, which may add its own slash (not verified). It only matters if a Plines client uses it.
- **Common mistakes:** Assuming all concatenation inserts slashes.
- **Follow-up questions:** Safe URL building? (`UriComponentsBuilder.fromUriString(base).path(path)` normalises.)

### Q38. Why are there two co-inward-ind keys with different spellings?
- **Why the interviewer asks:** Copy-paste drift.
- **Ideal answer:** `opus.fn-co-inward-ind-path` (137, read by Fire/Bond/Misc/Marine with a code default) versus `opus.fn-get-co-inward-ind-path` (249, read by Plines `PlineUtilsClient`), plus `opus.fn-get-co-inward-ind-api-path` (219) and `api.pgRpgeUtils.coInwardIndPath` (171). A path change needs edits in each, and a near-identical spelling means a typo is easy. The Plines block lines 249-251 contain non-plines keys appended later.
- **Common mistakes:** Editing the first key found by search.
- **Follow-up questions:** Detection of mismatch? (A test that loads all aliases and asserts equality.)

### Q39. `sharedreport.fn...` versus `sharedreports.fn...`: which is live?
- **Why the interviewer asks:** Plural/singular trap.
- **Ideal answer:** The singular `sharedreport.fn.email.not.polstatus.path` (142, "placeholder path, TBD from platform team") is dead. The plural `sharedreports.fn.email.not.polstatus.path` (251) with `.../get-email-not-polstatus` is read by `SharedReportUtilsClient` (commit `130eea57c`, OPUSM-41733). Outside the three modules but shows how stale keys pile up.
- **Common mistakes:** Updating the singular.
- **Follow-up questions:** What would you do? (Delete the dead key.)

### Q40. Which `@Value` keys have no default and would fail startup if missing?
- **Why the interviewer asks:** Fail-fast knowledge.
- **Ideal answer:** `app.ssl.trust-store` and `app.ssl.trust-store-password` (RestTemplate configs), `plines.api.base-url` (constructor `@Value`, no default), `API_BASE-URL` at its ~45 sites, and `opus.fn-decide-mt-cert-path` / `opus.fn-uwge-get-endtschd-query-path` in `PlineUtilsClient`. Spring throws `IllegalArgumentException: Could not resolve placeholder` during bean creation, so the pod crash-loops. `api.fn.decide-mt-cert.url` and similar are in a fully commented-out class, so harmless.
- **Common mistakes:** Assuming missing keys become null.
- **Follow-up questions:** Pros/cons of fail-fast? (Early detection vs deployment friction.)

### Q41. What is `ReportSettings` and what happens if its property is missing?
- **Why the interviewer asks:** `@ConfigurationProperties` null behaviour.
- **Ideal answer:** `motor/config/ReportSettings` is bound with `@ConfigurationProperties(prefix = "report.settings")` for `featureToggles`. No `report.settings.*` key exists in `application.properties`, so `getFeatureToggles()` returns null (inferred); callers must null-check. Not part of my three modules per notes; verify usages before relying on it.
- **Common mistakes:** Assuming an empty map.
- **Follow-up questions:** How to default? (Initialise the field `= new HashMap<>()`.)

### Q42. Why use `@Value` in 45 places instead of `@ConfigurationProperties`?
- **Why the interviewer asks:** Design critique.
- **Ideal answer:** Organic growth; each client added its own `@Value`. `@ConfigurationProperties` gives one typed class, validation (`@Validated`), IDE metadata and easier tests. With `@Value`, a typo in one placeholder is found only at startup, and refactoring needs text search. Migration should be incremental, starting with the Pline clients (`plines.*`).
- **Common mistakes:** Proposing a big-bang rewrite.
- **Follow-up questions:** How would you validate URLs? (`@Validated` with `@NotBlank` and a URL pattern.)

### Q43. How would you add startup validation for critical config?
- **Why the interviewer asks:** Quality improvement.
- **Ideal answer:** A `@ConfigurationProperties` class annotated `@Validated` for base URLs and paths (non-blank, path starts with `/`, base starts with `https://`), plus an `ApplicationRunner` that logs resolved hosts (not secrets) and checks Mongo with a ping on `CMGE_CODE`. Fail the startup if the DB name is blank or equals localhost. This catches Q7/Q8 class errors.
- **Common mistakes:** Logging the full URI.
- **Follow-up questions:** Will a failing startup hurt rollouts? (It prevents bad pods entering service.)

### Q44. Explain the dependency pins in `pom.xml` for security.
- **Why the interviewer asks:** Supply-chain config.
- **Ideal answer:** Boot parent 4.1.1 (CVE-2026-59270 per pom comment), `tomcat.version` 11.0.25, `commons-lang3` 3.18.0, `logback.version` 1.5.34, `log4j2.version` 2.25.5, `jackson-2-bom.version` 2.21.5, httpcore5 5.4.3. CVE ids are quoted from pom comments, not independently verified. Old pins for Netty/Spring/mongodb were removed because the Boot BOM is higher. Remaining old items: `org.json:json:20231013` and `junit-jupiter 5.9.0`.
- **Common mistakes:** Pinning below the BOM (a downgrade).
- **Follow-up questions:** How to find drift? (`mvn versions:display-dependency-updates`, Wiz/Sonar reports.)

### Q45. Java 17, Boot 4.1.1, Maven wrapper: what do you say about the toolchain?
- **Why the interviewer asks:** Reproducible builds.
- **Ideal answer:** `pom.xml` line ~39 Java 17; parent `spring-boot-starter-parent` 4.1.1 (line 12); `.mvn/wrapper/maven-wrapper.properties` wrapper 3.3.4, Maven 3.9.11 `only-script`. Plugins: `rewrite-maven-plugin` 6.46.1, compiler with Lombok processor, `spring-boot-maven-plugin`, `jacoco-maven-plugin` 0.8.12; the sonar plugin is commented out (CI calls it by coordinates). Version is `0.0.1-SNAPSHOT`, so the image tag identifies the release.
- **Common mistakes:** Mentioning Java 21 features.
- **Follow-up questions:** Why OpenRewrite? (Used for the Boot upgrade; inferred from branch names around Spring Boot 4 upgrade.)

### Q46. `spring-boot-devtools` is in the pom with runtime scope. Concern?
- **Why the interviewer asks:** Packaging hygiene.
- **Ideal answer:** Devtools is normally excluded from the repackaged fat jar by the Boot plugin, so it should not run in prod; the KT notes say verify. It can cause restarts and expose remote features if present. Check the built jar (`BOOT-INF/lib`) for `spring-boot-devtools`.
- **Common mistakes:** Assuming scope alone guarantees exclusion.
- **Follow-up questions:** How to mark? (`<optional>true</optional>`.)

### Q47. Is there a logback or log4j2 file, and what logging format is used?
- **Why the interviewer asks:** Observability config.
- **Ideal answer:** No `logback*.xml`/`log4j2*.xml` in repo. Only `logging.level.*` keys are set; Boot's default console pattern is used (plain text, no MDC, no JSON). The pom pins logback 1.5.34 and log4j2 2.25.5 only for CVEs. To add a request id or JSON format you must add a logback config or Boot structured logging properties (a newer Boot feature; availability in 4.1.1 is inferred, verify).
- **Common mistakes:** Assuming JSON logs in prod.
- **Follow-up questions:** What pattern would you add? (`%X{requestId}`.)

### Q48. Which Tomcat/server settings are NOT configured, and why does it matter?
- **Why the interviewer asks:** Defaults awareness.
- **Ideal answer:** No `server.tomcat.threads.max`, `server.tomcat.connection-timeout`, `server.tomcat.max-http-form-post-size`, `server.shutdown`, or `spring.mvc.async.request-timeout`. Defaults apply (for example 200 request threads, inferred from Boot defaults). With a heavy report (10-minute `maxTime`), 200 stuck threads would block the pod. Whether graceful shutdown is on by default for Boot 4.1.1 is not something I verified; the repo does not set `server.shutdown` [CONFIRM: check the Boot 4.1.1 reference].
- **Common mistakes:** Claiming specific defaults with certainty for the exact Boot version.
- **Follow-up questions:** What would you tune first? (Thread cap versus pod memory; request timeouts at the gateway.)

### Q49. What config would you add for graceful shutdown during deployment?
- **Why the interviewer asks:** Release safety.
- **Ideal answer:** `server.shutdown=graceful` and `spring.lifecycle.timeout-per-shutdown-phase=30s` so in-flight report requests finish when a pod is replaced; plus Kubernetes `terminationGracePeriodSeconds` longer than that and a readiness drop before SIGTERM (Helm, outside repo). Not set in this repo; whether default graceful applies depends on the Boot version [CONFIRM].
- **Common mistakes:** Setting only the Spring property.
- **Follow-up questions:** Long reports exceeding the grace period? (They fail; caller retries.)

### Q50. Which parts of configuration did you personally change for your three modules?
- **Why the interviewer asks:** Ownership honesty.
- **Ideal answer:** Be truthful and specific: git history cannot attribute commits to you (git user `NISHUNISARGA` has no commits in the repo). The current working copy shows uncommitted changes in `application.properties` (a local Mongo `#newDev` block with credentials, which must not be committed), `RejectRiskHandler` and `RejectRiskService`. [CONFIRM: which config keys you added or edited, for example any `plines.*` path keys for Template12/ASP.] Do not claim commits you cannot verify.
- **Common mistakes:** Claiming ownership of base-URL or CI changes by others.
- **Follow-up questions:** What would you never commit? (The Mongo URI block.)
