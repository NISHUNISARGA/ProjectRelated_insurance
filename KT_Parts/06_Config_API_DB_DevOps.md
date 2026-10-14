# KT Part 06 - Configuration, API, Database, DevOps and Security

Repo: `C:\Users\EPGRVFY\Documents\Plines_Base_code\New_branch\2\opus-documentprintdata-service`
Branch at snapshot: `feature/retrofit-develop-updated` (main/default branch: `retrofit-develop-R1`; git status shows `application.properties`, `RejectRiskHandler.java`, `RejectRiskService.java` modified).
Scope: the developer's three modules - **Template12 PolicyDocument** (`Template12PolicyDocumentService`), **ASP EndtDocument** (`AspEndtDocumentService`), **RejectRisk** (`RejectRiskService`), all in `src/main/java/com/agcs/fo/opus/plines/service/`.

Conventions: "(evidence)" = read directly in a file, path:line given. "(inference)" = my reasoning, not proven by the repo. Secrets are shown as `<redacted>`. Paths below are relative to the repo root unless absolute.

---------------------------------------------------------------------------------------------------

# PART 1 - CONFIGURATION CATALOG (Phase 7)

## 1.0 How configuration is loaded (profiles, files, overrides)

* Files in `src/main/resources/` (evidence): `application.properties` (251 lines), `application-env.properties` (3 lines, SSL only), `mytruststore.p12` (PKCS12 truststore on the classpath). `src/test/resources/` has only `mockito-extensions` and `xsd` (no test properties file).
* `spring.profiles.active=dev` (application.properties:72). **No `application-dev*.properties` exists** (only the two files above), so the "dev" profile activates nothing profile-specific; it is a label only. Also grep found no `@Profile`/`@ConditionalOn*` usage in `src/main/java` (searched: none returned). (evidence)
* `application-env.properties` is NOT a Spring Boot profile file (a profile file would be `application-<profile>.properties` and only load when profile `env` is active). It is pulled in explicitly with `@PropertySource("classpath:/application-env.properties")` on `motor/config/RestTemplateConfig.java:~52` and `plines/config/PlineRestTemplateConfig.java:~52`. Consequence: it is always loaded, and `@PropertySource` has lower precedence than `application.properties`, env vars and system properties. (evidence + Spring precedence rules)
* Everything is baked into one jar: base URLs, Mongo URI and truststore are in the repo resources. There is no `bootstrap`, config-server, or `spring.config.import` (evidence: none in properties).
* **How a deployment would override** (inference, based on standard Spring relaxed binding; the Helm repo is not in this repo so the actual practice is unverified):
  * Env var `SPRING_MONGODB_URI`, `SPRING_MONGODB_DATABASE` overrides `spring.mongodb.uri/database`.
  * `APP_HTTP_READ_TIMEOUT_MS` overrides `app.http.read-timeout-ms`.
  * **The key `API_BASE-URL` contains a hyphen and is upper-case.** Spring's relaxed binding for `@Value("${API_BASE-URL}")` does an exact property-name lookup against each PropertySource; for the OS-environment source Spring's `SystemEnvironmentPropertySource` maps the name by replacing `.` and `-` with `_` and tries upper-case variants, so an env var named `API_BASE_URL` (underscore) is expected to resolve it. Treat as inference; verify by starting the jar with `API_BASE_URL=...` and watching the logged outbound URL. Lowest-risk override: JVM arg `-DAPI_BASE-URL=https://...` or a Helm-supplied `application.properties` mounted via `spring.config.additional-location`.
  * Keys with other prefixes (`plines.api.base-url`, `workmen.api.base-url`, `pg-rpge-utils.base-url`, `external.pme.base-url`, ...) each need their own override. There is NOT one single switch for "which environment do external calls go to" - see 1.12.
* Build tooling: `.mvn/wrapper/maven-wrapper.properties` - wrapper 3.3.4, Maven 3.9.11, `distributionType=only-script`. Java 17 (`pom.xml:39`), Spring Boot parent 4.1.1 (`pom.xml:12`). No `logback*.xml`/`log4j2*.xml` file in repo (evidence: `git ls-files` filter returned nothing) - logging is configured only through `logging.level.*` keys (Boot defaults: console, pattern default). No Dockerfile or Helm chart in repo (evidence: `git ls-files` filter returned nothing).

## 1.1 App / server

| Key (line) | Value type / example | Read by | Purpose | Introduced / why | Business impact | Technical impact | Risk if changed | Risk if removed |
|---|---|---|---|---|---|---|---|---|
| `spring.application.name` (3) | string `Documentprintdata` | Boot (actuator/logging) | App identity | original | none | metrics/log tag | low | low (defaults to none) |
| `server.port` (4) | int `8080` | Tomcat | Listen port | original | none | must match container/Helm probe port | high: probes + service port mismatch = pod not ready | default is also 8080, so low |
| `server.servlet.context-path` (5) | `/gi/documentprintdata` | Tomcat / Spring MVC / Security matchers | Prefix for ALL URLs | original (GI = General Insurance, inference) | Caller (JasperServer/OPUS) URL contains it | All endpoints become `/gi/documentprintdata/api/opus/GenerateReport`, actuator `/gi/documentprintdata/actuator/health` | very high: every caller + ingress path + k8s probe breaks | very high (URLs move to `/`) |
| `spring.http.converters.preferred-json-mapper=jackson2` (17) | enum | Boot 4 HTTP message converters | Force Jackson 2 converter | commit `9248b12ff` (2026-09-18, "OPUSM-48579 Fixes after merging spring boot version upgradation changes"); explanatory comment lines 7-16 | Without it every POST to the report API fails with HttpMessageConversionException | Boot 4.1.1 defaults to Jackson 3 (`tools.jackson.*`) which ignores `com.fasterxml.jackson` annotations used by `PayLoadRequest` (`@JsonProperty`) and DTOs | high: removing breaks `@RequestBody PayLoadRequest` (400/500) | high |
| `spring.messages.basename=messages` (64) | string | Boot MessageSource | i18n bundle | original | none | **No `messages*.properties` exists in resources** (evidence: dir listing) -> stale but harmless (Boot tolerates missing bundle) | none | none |
| `spring.profiles.active=dev` (72) | string | Boot | Activates profile `dev` | original | none | no profile files exist -> inert label. Risk: a deployment that sets `SPRING_PROFILES_ACTIVE=uat` also activates nothing, giving a false sense of environment separation | low | low |

## 1.2 MongoDB (Boot 4 prefix change)

| Key (line) | Value | Read by | Purpose | Why/when | Impact | Notes |
|---|---|---|---|---|---|---|
| `spring.mongodb.uri` (67) | `mongodb://<redacted>` (Atlas-style multi-host + replicaSet + ssl=true + authSource=admin; user/password embedded) | Boot 4 `org.springframework.boot.mongodb.autoconfigure.MongoProperties` -> `MongoTemplate` (injected in all handlers e.g. `RejectRiskHandler.java:~30`) | Connection string | Commit `9248b12ff` (2026-09-18) added `spring.mongodb.*` after Boot 4 upgrade; commit `7dd2f2f67` (2026-09-26, "OPUSM-52675 Claude PR review fix") **removed** the UAT URI/database lines from the committed file; the current working copy re-adds a `#newDev` block (uncommitted diff, +3 lines at line 66-68) | If wrong/missing: app starts but can't query; with the old prefix it silently falls back to `localhost:27017` (comment lines 26-31) | **Credentials are in cleartext in the working-copy file. Do not commit it** (see Security). Under Boot 4 the prefix is `spring.mongodb.*`; `spring.data.mongodb.*` is silently ignored (evidence: comment lines 26-31). |
| `spring.mongodb.database` (68) | `OPUSAGICPRODR1` | same | Database name (name suggests an AGIC production-clone R1 DB - inference from the string) | working copy only (uncommitted) | Wrong DB = empty reports (not errors: aggregates just return 0 rows) | The committed history previously used a different DB for UAT (removed in `7dd2f2f67`) - which DB a deployed pod uses is decided outside this repo |
| `auto-index-creation` (34) | bool `false` | **Nobody** - it is not under `spring.mongodb.*`/`spring.data.mongodb.*` so Boot does not bind it (evidence: no `@Value("${auto-index-creation}")`; correct Boot key would be `spring.data.mongodb.auto-index-creation`) | Intended: stop Spring Data creating indexes | Commit `62b3c94a7` (2026-01-28 "Performance issue fixes.") | None today: the handlers use raw `MongoTemplate.aggregate` on collection-name strings, no `@Document`/`@Indexed` entities, so no auto-index would happen anyway | Stale/ineffective key (inference, confidence high) |
| `option.min-connection-pool-size` (35), `option.max-connection-pool-size` (36), `option.max-wait-time` (37) | ints 20 / 100 / 2000 | **Nobody** (no `@Value`, not a Boot key) | Intended: driver pool sizing | `62b3c94a7` (same perf commit) | Pool sizing NOT applied. Real driver defaults apply: maxPoolSize 100, minPoolSize 0, maxWaitTime 2 min (MongoDB driver defaults) | To actually set pool: put `maxPoolSize=100&minPoolSize=20&waitQueueTimeoutMS=2000` in the URI options (inference: Boot 4 has no dedicated pool properties; verify) |

## 1.3 Jackson converter
Only `spring.http.converters.preferred-json-mapper=jackson2` (see 1.1). Supporting code: `motor/config/JacksonConfig.java` defines a plain `ObjectMapper` with `JavaTimeModule` (so it ignores `spring.jackson.*` settings - those keys would not take effect on this bean; none are set). pom pins `jackson-2-bom.version=2.21.5` (`pom.xml:~52`) - note comment: under Boot 4.1.1 the property is `jackson-2-bom.version`, NOT `jackson-bom.version`.

## 1.4 Logging
| Key (line) | Value | Effect | Status |
|---|---|---|---|
| `logging.level.org.springframework.data.mongodb.core.MongoTemplate=DEBUG` (20) | DEBUG | Logs every Mongo operation incl. query documents (can contain contract IDs, customer ids/names in `$match`) | Active; performance + PII-in-log risk in prod |
| `logging.level.org.springframework.data.mongodb.core.aggregation.Aggregation=DEBUG` (23) | DEBUG | Logs aggregations | Active (note `MongoTemplate` logs the pipeline when DEBUG) |
| `logging.level.com.agcs.fo.opusMotorGateway.service=DEBUG` (21) | DEBUG | **Package `com.agcs.fo.opusMotorGateway.*` does not exist** (real root is `com.agcs.fo.opus`, main class `motor/OpusMotorGatewayApplication.java`) | Stale |
| `logging.level.service.com.agcs.fo.opus.motor.DataDownloadingService=DEBUG` (22) | DEBUG | Logger name is wrongly ordered (`service.com...`); real class is `com.agcs.fo.opus.motor.service.DataDownloadingService` | Stale/no effect |
| `logging.level.com.agcs.fo.opusMotorGateway.service.GrouptaxinvoiceService=INFO` (102) | INFO | Same non-existent package | Stale |

No logback/log4j2 XML. pom pins `logback.version=1.5.34`, `log4j2.version=2.25.5` for Wiz CVEs (`pom.xml:65,75`). Also `CustomRequestFilter` uses `System.out.println` for full request bodies (bypasses logging levels) - see Security.

## 1.5 Actuator
| Key (line) | Value | Effect |
|---|---|---|
| `management.endpoints.web.exposure.include=health,info` (49) | list | Only these two exposed over web. Introduced by commit `cc7f5d101` (2025-12-29 "Enabled Spring Actuator.") |
| `management.endpoint.info.enabled`, `management.endpoint.health.enabled` (50-51) | true | Explicit enable (redundant; default true) |
| `management.endpoint.loggers.enabled=true` (52) | true | Enabled but NOT exposed (exposure list excludes `loggers`) and `SecurityConfig` denyAll for `/actuator/**` other than health/info - so no runtime log-level change possible. Mismatch (inference: someone wanted /loggers, never exposed it). |
Health has no `show-details` set -> default `never`; no custom probes (`management.endpoint.health.probes.enabled`) set -> Helm liveness/readiness likely use `/gi/documentprintdata/actuator/health` (inference).

## 1.6 springdoc (Swagger)
`springdoc.api-docs.path=/v3/api-docs` (41), `springdoc.swagger-ui.path=/swagger-ui/index.html` (42), `operationsSorter=method`, `tagsSorter=alpha` (44-45). Read by springdoc 3.1.1 (`pom.xml:~155`). `OpenApiConfig.java` registers GroupedOpenApi `public` with `pathsToMatch("/api/**")` -> grouped docs at `/v3/api-docs/public`. Security: `anyRequest().permitAll()` means Swagger UI and api-docs are **open without auth** (evidence `SecurityConfig.java:~42`). Risk: documentation disclosure; removal of springdoc keys only changes paths.

## 1.7 HTTP timeouts and pool (`app.http.*`)
Read in `motor/config/RestTemplateConfig.java` (connect/read/socket only) and `plines/config/PlineRestTemplateConfig.java` (all keys). Introduced by `57c573b74` (2025-12-06, added with "DataDownloadingService") and moved to properties by `fd3d73383` (2025-12-29 "Removed Hardcoded URL and moved it to properties."); `fast-lookup.read-timeout-ms` + pool keys by `4cd1cad2e` (2026-08-31 "OPUSM-42246 Working code for pline"), documented in `PlineRestTemplateConfig` header as "Phase 1 incident fix - bounded timeouts ... Phase 2 - dedicated fast-lookup RestTemplate".

| Key (line) | Value | Code default | Meaning | Beans affected |
|---|---|---|---|---|
| `app.http.connect-timeout-ms` (55) | 5000 | motor 10000 / pline 5000 | TCP connect | all three RestTemplates |
| `app.http.read-timeout-ms` (56) | 20000 | motor 120000 / pline 20000 | HttpClient5 `responseTimeout` | `restTemplate` (@Primary), `plineRestTemplate` |
| `app.http.socket-timeout-ms` (57) | 20000 | 120000 / 20000 | SO_TIMEOUT | same |
| `app.http.connection-request-timeout-ms` (58) | 3000 | 3000 | Wait for pooled connection | **pline beans only** |
| `app.http.max-conn-total` (59) | 100 | 100 | Pool size | **pline beans only**; motor `restTemplate` uses HttpClient5 defaults (25 total / 5 per route, library defaults; inference) |
| `app.http.max-conn-per-route` (60) | 20 | 20 | Per-host pool | pline only |
| `app.http.fast-lookup.read-timeout-ms` (61) | 8000 | 8000 | Read+socket timeout for the `fastLookupRestTemplate` used by `PlineUtilsClient` | pline fast-lookup bean |

Which bean each of the developer's classes gets (evidence): `PlineUtilsClient` -> `@Qualifier("fastLookupRestTemplate")` (8 s). `PlinePolicyRefClient`, `PlineSSTRolloutDateClient`, `PlinePgRpgeUtilsClient` take an **unqualified** `RestTemplate` -> the `@Primary` motor `restTemplate` (5 s connect / 20 s read, no connection-request timeout). Motor `PolicyRefClient`/`PMEGetContractIdClient` create `new RestTemplate()` privately (no timeouts, no truststore) - a risk for hung calls (not used by the three modules).
Worst-case latency for one request (inference): Template12 can fire up to ~5 distinct utility calls plus several Mongo aggregates each with `maxTime 600 s`; each external failure returns null/"" quickly, but a hung one costs up to 8 s (fast-lookup) or 20 s.

## 1.8 SSL truststore (`application-env.properties`)
| Key (line) | Value | Read by | Notes |
|---|---|---|---|
| `app.ssl.trust-store` (2) | `mytruststore.p12` (classpath resource) | `RestTemplateConfig`, `PlineRestTemplateConfig` (`@Value("${app.ssl.trust-store}")`, no default) | Loaded via `ClassPathResource`; file is in `src/main/resources` (committed binary) |
| `app.ssl.trust-store-password` (3) | `<redacted>` (plaintext in the file; added in `57c573b74`, 2025-12-06) | same, no default | Plaintext secret in VCS. Gitleaks allowlist covers this file (see 4.4) |
| `app.ssl.trust-store-type` (4) | `PKCS12` | same, default `PKCS12` | Changing to JKS requires the file to be JKS |
Risk if changed: wrong password -> `IOException: keystore password was incorrect` at bean creation = app fails to start (these are eagerly created @Beans). Risk if removed: startup failure (`@Value` has no default). Effect of truststore: custom truststore **replaces** the JVM default cacerts for outbound calls (`SSLContexts.custom().loadTrustMaterial(trustStore, null)`), so any new upstream whose CA is not in `mytruststore.p12` fails the TLS handshake (inference from code).
Note: the Mongo connection (`ssl=true`) uses the JVM's default trust, not this truststore.

## 1.9 External API - base URLs (the "which environment" keys)
| Key (line) | Value (host, not secret) | Read by | Note |
|---|---|---|---|
| `API_BASE-URL` (76) | `https://dev2-apiservice.opusm.srv.allianz/` (note: leading space after `=` in the file and trailing `/`; `Properties` trims leading whitespace) | ~45 `@Value("${API_BASE-URL}")` sites, incl. `PlineUtilsClient.java:31`, Fire/Bond/Motor handlers, `SharedReportUtilsClient`, `WorkmenUtilsClient` | The "global" base. Commit `c4407d66c` (2026-03-08, "URL configuration in application.properties"). Trailing slash handled by `PlineUtilsClient.url()` (strips it). Commented older values on lines 75, 77 marked "can be removed" (dev1 host and an `opus-core` EKS host). |
| `plines.api.base-url` (227) | `https://dev2-apiservice.opusm.srv.allianz` | `PlinePolicyRefClient`, `PlineSSTRolloutDateClient`, `PlinePgRpgeUtilsClient` (constructor `@Value`, no default) | Added `d4c069135` (2026-07-16, "AutoRenewalDocument with XSD changes"). **Separate from API_BASE-URL** - same host today by coincidence, must be changed twice. |
| `workmen.api.base-url` (187) | dev2 host | Workmen clients | `b928bc28d` (2026-06-04). Not used by the three modules. |
| `external.pme.base-url` (113) | **dev1** host + path | no `@Value` hit found in `src/main/java` | Unreferenced (inference) |
| `pg-rpge-utils.base-url` (121) | **dev1** host + `/ext/utils` | no `@Value` hit | Unreferenced; `pg-rpge-utils.timeout-ms=5000` (122) also unreferenced. Commit `252be1658` (2026-02-12). |
| `pg-uwge-utils.base-url` (139) | **dev1** host + `/ext/policy` | no `@Value` hit | Unreferenced |
| `ext.sst-rollout-date.base-url` (125), `ext.prnt-ptv-trans.base-url` (128) | **dev1** host | no `@Value` hit | Unreferenced |
**Conflict:** dev1 hosts (lines 113,121,125,128,139) vs dev2 hosts (76, 187, 227). If an environment is flipped by editing only `API_BASE-URL`, the Plines three-module calls through `PlinePolicyRefClient`/`PlineSSTRolloutDateClient` still follow `plines.api.base-url`.

## 1.10 External API - paths
Pattern: `<key>=/ext/<utils|policy|customer>/api/v<n>/<package-kebab>/<function-kebab>`; the Oracle function it wraps is usually named in code comments (e.g. `PG_RPGE_UTILS.FN_GET_COMP_BRN_NUMBER`).

### 1.10.1 Plines keys actually used by the three modules (via `PlineUtilsClient`, `Pline*Client`)
| Key (line) | Path value | Used by |
|---|---|---|
| `plines.fn.comp.brn.number.path` (229) | `/ext/utils/api/v1/pg-rpge-utils/comp-brn-number` | `PlineUtilsClient.fnGetCompBrnNumber`, `PlinePgRpgeUtilsClient` (company BRN for Template12 base `Template12PolicyDocumentBaseService.java:2017`, ASP CompanyNameWithGSTRegNo/Signature handlers) |
| `plines.fn.rppm.get.signature.path` (230) | `/ext/customer/api/v1/fn-rppm-get-signature` | `fnRppmGetSignature*` (ASP `PlineCommonSignatureHandler`) |
| `plines.api.customer.signature.type` (not defined!) | code default `UW` | `PlineUtilsClient.java:57` `${plines.api.customer.signature.type:UW}` - key absent from properties, default used (note `api.customer.signatureType=UW` line 111 is a different key read by motor) |
| `plines.fn.einvoice.qr.ind.path` (231) | `/ext/utils/api/v1/einvoice/qr-indicator` | `fnGetEinvoiceQrInd` (ASP ServiceTaxInvoice uses `PlineEinvoiceQrIndicatorClient`, own property, see note) |
| `plines.fn.gen.cp.table.path` (232), `plines.fn.gen.cp.add.table.path` (233) | `/ext/customer/api/v2/pgcpgentable/FN_GEN_CP_TABLE`, `/ext/customer/api/v1/addresses` | `fnGenCpTable` / `fnGenCpAddTable` (not directly by the 3 modules) |
| `plines.fn.date.value.constants.path` (237) | `/ext/utils/api/v1/pg-util-general/get-date-value-constants` | comment lines 234-236: path **derived/guessed** ("verify ... if it starts 404'ing") - highest-risk path in file |
| `plines.fn.benefit.hdr.path` (238) / `det` (239) / `hdr1` (240) / `det1` (241) | `/ext/utils/.../pg-rpge-uw-listing/fn-get-benefit-hdr|det` and `/ext/policy/...` | benefit functions; `hdr`/`hdr1` differ only by `/utils` vs `/policy` segment (duplicate-with-variation) |
| `plines.fn.auto.rn.*.path` (242-245), `plines.fn.accounting.rec.path` (246) | auto-renewal functions `/ext/policy/api/v1/pg-rpuw-auto-rn-rptng/...` | duplicates of `fire.autorenewal.*` (lines 178-182); not used by the three modules |
| `plines.fn.show.stax.inv.ind.path` (247) | `/ext/policy/api/v1/stax-inv` | `fnShowStaxInvInd` (Template12 `PlinePolicyDocumentMainHandler.java:193,821,967,1219,1775`; `PlineT12PolicyDocumentHandler.java:264`). Note the *older* equivalents differ: `opus.fn-show-stax-inv-ind-path=/ext/policy/api/v1/pg-uwge-utils/stax-inv-indicator` (138) and `workmen.fn.show.stax.inv.ind.path=/ext/policy/api/v1/stax-inv` (202) |
| `plines.sst.rollout.date.path` (248) | `/ext/policy/api/v1/pg-uwge-utils/sst-rollout-date` | `PlineSSTRolloutDateClient` (ASP ServiceTaxInvoice `AspServiceTaxInvoiceHandler.java:86`; Template12 `PlineT12PremiumDebitNoteHandler.java:49`), also `PlineUtilsClient.fnGetSstRolloutDate` |
| `plines.policy.ref.path` (228) | `/ext/policy/api/v1/pg-uwge-utils/get-policy-ref` | `PlinePolicyRefClient.getPolicyRef` |
| `plines.policy.ref.pme.path` (223, in the "Marine" block!) | `/ext/policy/api/v1/pme-api/get-policy-ref` | `PlinePolicyRefClient.getPolicyReference` - used by `AspServiceTaxInvoiceHandler.java:83`. Added by `4cd1cad2e` (2026-08-31). Misplaced under the Marine heading. |
| `opus.fn-einvoice-print-endt-ind-path` (134) | `/ext/utils/api/v1/pg-rpge-utils/invoice-print-endt-indicator` | `PlineUtilsClient.fnEinvoicePrintEndtInd` (ASP `AspEndtDocument2MainHandler.java:122`) |
| `opus.fn-einvoice-print-pol-ind-path` (136) | `/ext/utils/api/v1/pg-rpge-utils/invoice-print-indicator` (code default identical) | `fnGetEinvoicePrintPolInd` (Template12 main/T12 handlers) |
| `opus.fn-get-co-inward-ind-path` (249, "Plines" block tail) | `/ext/utils/api/v1/pg-rpge-utils/co-inward-ind` | `fnGetCoInwardInd` (Template12 `PlinePolicyDocumentMainHandler.java:185,832,1210`). NB near-duplicates exist: `opus.fn-co-inward-ind-path` (137, **different key spelling**; read by Fire/Bond/Misc/Marine with code default), `opus.fn-get-co-inward-ind-api-path` (219), `api.pgRpgeUtils.coInwardIndPath` (171). |
| `opus.fn-decide-mt-cert-path` (132), `opus.fn-uwge-get-endtschd-query-path` (133) | `/ext/utils/api/v1/fn-decide-mt-cert`, `.../fn-uwge-get-endtschd-query` | required (no default) by `PlineUtilsClient` -> app will not start without them; used by endorsement schedule modules, not the three |

### 1.10.2 Other lines in the file
* Motor/global: `opus.indicator.path`(80), `contract.id`(82), `gst.rate.endpoint`(83), `opus.customer.path`(85), `opus.address.path`(86), `endorsement.report.path`(87), `policy.ref.path`(90), `company.reg.path`(91), `sst.rollout.date.path`(92), `customer.fn.prnt.ptv.trans.path`(93), `grouptaxinvoice.constants.path/connect-timeout-ms/read-timeout-ms`(98-100, the only per-client timeouts, read by `motor/client/GrouptaxinvoiceConstantsClient.java`), `api.customer.signatureType`(111), `api.pgRpgeUtils.compBrnNumberPath`/`coInwardIndPath`/`api.customer.fnRppmGetSignaturePath`/`...NamePath`(170-173), `opus.fnGetCodeDesc.path`(174; **no leading slash**), `opus.qr.path`(147), `einvoice.qr.path`(153; same value as 147), `opus.func-getdesc.path`(148), `pg-rpge-utils.path`(155), `pg-util-general.path`(157), `ext.pme.get-contract-id-path`(159), `ext.pme.retry.max-attempts/backoff-ms`(160-161; **also duplicated as comments at 116-117**), `pg-uwge-utils.path.get-policy-ref`(163), `pme-api.path.get-contract-id`(164), `pme-api.path.get-policy-ref`(217), `ext.prnt-ptv-trans.path.fn-prnt-ptv-trans`(166 = duplicate of 93), `ext.sst-rollout-date.path.get`(168 = duplicate of 92 and 248).
* Fire AutoRenewal: `fire.autorenewal.*`(178-182) with a `TODO: confirm exact paths with the external API team before deployment` (177); `outstanding.survey.path`(184).
* Workmen block (187-202), Misc block (205-209), Bond block (212-213), Marine block (216-223) - used only by those LOB modules.
* Shared report: `sharedreport.fn.email.not.polstatus.path`(142, "placeholder path, TBD from platform team") vs `sharedreports.fn.email.not.polstatus.path`(251, **plural `sharedreports`**, path `.../get-email-not-polstatus`). Code reads the plural (`SharedReportUtilsClient`); the singular is dead. Commit `130eea57c` (2026-08-14, OPUSM-41733).

## 1.11 Defaults in code vs not in properties; stale and conflicting keys
* **Read with a default in code, NOT defined in properties:** `plines.api.customer.signature.type:UW` (PlineUtilsClient.java:57); `app.ssl.trust-store-type` has a default but is defined; `ext.pme.retry.*` defined; `bond.spell-number-*` (defined); `api.fn.get-einvoice-print-endt-ind.url:` (inside a fully commented-out class `EndorsementFunctionsExternalService`).
* **Read WITHOUT default and NOT defined (would fail startup if the bean were live):** `api.fn.decide-mt-cert.url`, `api.fn.get-endtschd-query.url` - but the class is entirely commented out (`motor/service/EndorsementFunctionsExternalService.java`, `//@Service`), so harmless. `report.settings.feature-toggles` bound by `@ConfigurationProperties(prefix = "report.settings")` (`motor/config/ReportSettings.java`) - not defined anywhere -> `getFeatureToggles()` returns null; callers must null-check (inference: check before use).
* **Defined but unread by any `@Value` found:** `external.pme.base-url`, `pg-rpge-utils.base-url`, `pg-rpge-utils.timeout-ms`, `pg-uwge-utils.base-url`, `ext.sst-rollout-date.base-url`, `ext.prnt-ptv-trans.base-url`, `sharedreport.fn.email.not.polstatus.path`, `auto-index-creation`, `option.*`, `spring.messages.basename`, `management.endpoint.loggers.enabled`. (My grep covered `src/main/java` for `${...}` placeholders; `Environment.getProperty` calls were searched and none found.)
* **Same path, many keys (maintenance trap):** comp-brn-number is in 8 keys (91, 155, 170, 190, 206, 220, 229, plus `-th` variants at 205, 221). invoice-indicator in 80, 135, 218. qr-indicator in 147, 153, 198, 231. co-inward-ind in 137, 171, 201, 219, 249. sst-rollout-date in 92, 168, 189, 248. get-policy-ref in 90, 163 (wrong), 164 (named contract-id), 188, 209, 217, 223, 228.
* **Policy-ref conflict (important):** 
  * `plines.policy.ref.path` = `/ext/policy/api/v1/pg-uwge-utils/get-policy-ref` (228)
  * `plines.policy.ref.pme.path` / `policy.ref.path` / `workmen.policy.ref.path` / `pg-uwge-utils.path.get-policy-miscref` / `pme-api.path.get-policy-ref` = `/ext/policy/api/v1/pme-api/get-policy-ref`
  * `pg-uwge-utils.path.get-policy-ref` (163) = `/api/v1/pg-uwge-utils/get_policy_ref` - **no `/ext/policy` prefix and an underscore** instead of hyphen -> read by motor `PolicyRefClient`; will 404 unless the base URL already includes `/ext/policy` (it does not: `API_BASE-URL` is host only). Likely stale/broken (inference).
  * `pme-api.path.get-contract-id` (164) has the value `.../get-policy-ref` - the key name says contract-id; value is policy-ref (mismatch).
  Two Plines methods therefore hit two different backend services (pg-uwge-utils vs pme-api) for the same business concept; ASP uses the `pme-api` one via `getPolicyReference`.
* **Commented "can be removed" lines:** 75, 77, 96-97, 106, 108, 115-117, 145-146, 151-152 (old dev1/opus-core hosts). They still contain real hostnames; removing them is safe.
* **Trailing/leading slash inconsistency:** `opus.fnGetCodeDesc.path=ext/utils/...` (174) has no leading slash; `API_BASE-URL` has trailing `/`. `PlineUtilsClient.url()` concatenates `base(without trailing '/') + path`, so a path without a leading slash would produce `...allianzext/utils` (bug) - only matters for the key at line 174 (read by `motor/client/FnGetCodeDescClient`, which may add its own slash; not verified).

## 1.12 Environment variable naming and promotion across environments
`API_BASE-URL` is the only key styled like an env var (upper-case with a hyphen). Source: commit `c4407d66c` / `37efa7f17` (2026-03/04). Because of the hyphen, shell-level `export API_BASE-URL=...` is invalid in POSIX shells (identifiers cannot contain `-`); container runtimes (k8s `env:` name) can carry it, but Spring's relaxed mapping is the only reliable path (inference - verify). To move dev2 -> UAT -> prod a release engineer must change at minimum: `API_BASE-URL`, `plines.api.base-url`, `workmen.api.base-url`, `spring.mongodb.uri`, `spring.mongodb.database` - and likely the truststore if the CA differs. Hosts follow `devN-apiservice.opusm.srv.allianz` (evidence).

## 1.13 LOB-specific blocks (heading comments in the file)
* Group tax invoice (95-100), Policy certificate printing (89-93), Signature (105-111), Fire AutoRenewal (176-182), Workmen (186-202), Misc (204-209), Bond (211-213), Marine (215-223), Plines (225-251). The Plines block lines 249-251 (after `plines.sst.rollout.date.path`) contain *non-plines* keys appended later (`opus.fn-get-co-inward-ind-path`, `opus.fn-getapplonitem-path`, `sharedreports.fn...`) = organic growth, no owner per block.

## 1.14 Build/pom facts relevant to config
* `spring-boot-starter-parent 4.1.1`, Java 17, `tomcat.version 11.0.25` (CVE pin), `commons-lang3 3.18.0`, `logback 1.5.34`, `log4j2 2.25.5`, `jackson-2-bom 2.21.5`, springdoc 3.1.1, httpclient5/httpcore5 (httpcore5 pinned 5.4.3 `pom.xml:168`), `org.json:json:20231013` (`pom.xml:128`, old - used by `CustomRequestFilter`; potential CVE-2023-5072 era library, inference), `spring-boot-devtools` runtime scope (`pom.xml:~132` - devtools is excluded from fat jar by `spring-boot-maven-plugin` normally; verify).
* Plugins: `rewrite-maven-plugin 6.46.1`, `maven-compiler-plugin` (Lombok annotation processor), `spring-boot-maven-plugin`, `jacoco-maven-plugin 0.8.12`; sonar plugin commented out (invoked from CI by full coordinates).
* `<artifactId> gi-documentprintdata </artifactId>` has leading/trailing spaces (`pom.xml:19`) - cosmetic.

## 1.15 Why-introduced summary (commit evidence)
`57c573b74` DataDownloadingService + truststore + app.http (2025-12-06); `fd3d73383` hardcoded URL -> properties (2025-12-29); `cc7f5d101` actuator (2025-12-29); `62b3c94a7` perf fixes: `auto-index-creation`, `option.*` (2026-01-28); `252be1658` pg-rpge-utils.* (2026-02-12); `c4407d66c` API_BASE-URL (2026-03-08); `b928bc28d` workmen (2026-06-04); `d4c069135` `plines.api.base-url`, date-value-constants (2026-07-16); `130eea57c` shared reports (2026-08-14); `4cd1cad2e` `plines.policy.ref.pme.path`, fast-lookup timeout (2026-08-31); `9248b12ff` Boot 4 converter + `spring.mongodb.*` (2026-09-18); `7dd2f2f67` Mongo UAT URI removed from committed file (2026-09-26).

---------------------------------------------------------------------------------------------------

# PART 2 - API CATALOG (Phase 10)

## 2.1 Inbound endpoints (complete list found)
| Method + path (with context path `/gi/documentprintdata`) | Source | Notes |
|---|---|---|
| `POST /api/opus/GenerateReport` (JSON in/out) | `motor/controller/CommonController.java:45,180` (`@RestController @RequestMapping("/api/opus/GenerateReport")`, `@PostMapping(produces=application/json)`) | **Single entry point for ALL reports, all LOBs.** The report is chosen by `filterCondition.reportName`. |
| `GET /api/opus/GenerateReport/getJSON` (XML out) | `CommonController.java:174` | Returns empty `ResponseDTO` (utility/legacy) |
| `GET /actuator/health`, `GET /actuator/info` | Boot actuator (`application.properties:49`) | Permitted without auth (`SecurityConfig`) |
| `/actuator/**` (other) | not exposed + `denyAll` | |
| `GET /swagger-ui/index.html`, `GET /v3/api-docs`, `/v3/api-docs/public` | springdoc | open (permitAll) |
Other controllers: `motor/controller/DocumentController.java` is an empty class (4 lines); `ExtendedController.java` is empty (0 bytes). `@RestControllerAdvice`: `motor/exception/GlobalExceptionHandler`, `generic/advice/GenericExceptionHandler`; `fire/advice/FireRequestBodyAdvice` and `miscellaneous/advice/MiscRequestBodyAdvice` are commented out (file starts with `/*`). No other `@RequestMapping` found in `src/main/java`.

## 2.2 Request schema
`motor/payload/PayLoadRequest.java`:
```
{
  "metadata":        { "<string>": "<string|number>", ..., "mergeStaticPdfFiles": ["a.pdf", ...] },   // Map<String,String> built by a custom setter
  "filterCondition": { "<string>": "<string>" },                                                        // Map<String,String>; DRIVES everything
  "mergeStaticPdfFiles": [ ... ]                                                                       // optional top-level too (List<String>, default empty)
}
```
Behaviours (evidence, `PayLoadRequest.java:~30-58`): `setMetadata(Map raw)` keeps String and Number values (numbers converted with `toString()`); a `mergeStaticPdfFiles` List found *inside* metadata is lifted into the field; **booleans, nulls, nested objects are silently dropped**. `filterCondition` values must be strings: a JSON number or array there makes Jackson fail (400/500) - except `pBranch`/`pBranchName` arrays, which `CustomRequestFilter` flattens (below). Field is declared `FilterCondition` (capital F) but Lombok getter `getFilterCondition()` makes Jackson name it `filterCondition`.

**Metadata keys consumed by the controller** (`CommonController.java:~442-470`): `name`, `email`, `senderMail` (-> MailingDetails); `copyType`, `batchConfigId`, `boResId`, `channel`, `dataClass`, `disallowCopy`, `docType`, `linkLocation`, `outputFormat`, `outputPath`, `reportName`, `templateId`, `templateName` (-> TemplateDetails; the integer ones go through `parseIntSafe`, returning null on bad input). `requestBody.getMetadata().get(...)` is called with no null check - **a request without `metadata` throws NPE -> 500** (inference from code: `populateMailingDetails` dereferences `getMetadata()`).

**`CustomRequestFilter`** (`motor/interceptor/CustomRequestFilter.java`, `@Component Filter`, applies to every request): wraps request, reads body, if `Content-Type: application/json` and body is a JSON object with `filterCondition.pBranch` or `pBranchName` as an array, joins the array with commas into a string (so `Map<String,String>` binding works); empty/invalid body becomes `{}`; any exception -> pass original through. **Prints content type, original and modified body with `System.out.println`** (lines ~25-36) - PII/log-bypass concern.

**Routing in the controller** (`CommonController.java:180-232`): `reportName` normalised (strip leading `/`, strip `reports/` prefix, case-insensitive). First a Spring bean named exactly the reportName (map `Map<String, ReportStrategy>`) -> `strategy.execute(requestBody)`; else legacy `switch(reportName)`. The three modules are `@Service("...")` beans (strategy path):
* RejectRisk: `AGIC/UW/Plines/RejectRisk/RejectRisk` (`RejectRiskService.java:34`) - (a Workmen variant `AGIC/UW/Workmen/RejectRisk/RejectRiskReport` is separate)
* ASP: `AGIC/UW/Plines/ASPEndtSchedule/EndtDocument` (`AspEndtDocumentService.java:165`)
* Template12: `AGIC/UW/Plines/PolicySchedule/Template12/PolicyDocument` (`Template12PolicyDocumentService.java:103`)
Merged-away names (not registered, per class javadoc): `.../Template12/PolicyDocument2`, `.../Template12/Landscape`, `.../ASPEndtSchedule/EndtDocument2`, `.../AddMemberLandscape`, `.../DeletionMemberLandscape`. A caller still sending those would fall to the legacy `switch`; behaviour for an unknown name: `switch` has no default case visible in the excerpt read, `transactions` stays an empty `Transactions` -> the "default transaction" empty XML is returned with status SUCCESS (inference; verify with a test call). `reportName == null` -> `switch(null)` NPE -> 500.

## 2.3 Response schema
`motor/requestpayload/SuccessResponse.java` (verified): 
```
{ "status": "SUCCESS", "data": "<XML string of Transactions>", "timestamp": "2026-10-13T10:15:30.123" }
```
Fields exactly: `status`, `data`, `timestamp` (`LocalDateTime.now().toString()`). `data` is the XML (`XmlMapper.writeValueAsString(transactions)`), escaped inside JSON. Always HTTP 200 with `Content-Type: application/json`, even when no rows are found (a default empty `Transaction` with `mailingDetails` + `templateDetails` is produced, `CommonController.createDefaultTransaction`). The legacy path logs the whole XML at INFO (`log.info("xml response data ...")`, TODO to change to DEBUG); the strategy path at DEBUG.
Inside the XML for the three modules: `Transaction -> transactionDetails -> reportDataList[] -> reportData(dataClass, subReportName?) -> prntDetails -> transDetList -> transDet`, plus `reportParameters`; the `PlineReportNames.tag(transactions, getClass())` call stamps report/sub-report names (commits `e74bc0d5c`, `861f0666c`, 2026-10-07/08, "Added report, sub report name in response/listing report").

## 2.4 Status codes, errors
| Situation | Handler | Result |
|---|---|---|
| Success or no data | controller | 200, SuccessResponse (empty transaction when nothing found) |
| `NoDataFoundException` | `GlobalExceptionHandler.handleNoData` | 404 `ProblemDetail` {title "No data found", detail=message, errorCode `NO_DATA`, timestamp} |
| `IllegalArgumentException` (anywhere) | `generic/advice/GenericExceptionHandler` | 400 {title "Bad request", detail "Invalid request parameters", errorCode `BAD_REQUEST`}; message only logged (WARN) |
| any other `Exception` | `GlobalExceptionHandler.handleGeneric` | 500 {title "Internal error", detail "An unexpected error occurred", errorCode `INTERNAL_ERROR`}; **the exception is not logged in this handler** (no `log.error`) -> root cause invisible unless logged by the thrower (evidence: file has no logger) |
| Malformed JSON body | Spring `HttpMessageNotReadableException` -> falls into `handleGeneric` -> 500 rather than 400 (inference: handler catches `Exception` and Spring's DefaultHandlerExceptionResolver is not reached because `@ExceptionHandler(Exception.class)` matches first) |
| Unsupported method / media type | same catch-all -> 500 (inference) |
Note comment in `GenericExceptionHandler` header: handler selection is by exception specificity across all `@RestControllerAdvice`.

## 2.5 Authentication / authorization (SecurityConfig)
`motor/config/SecurityConfig.java`: STATELESS sessions; CSRF ignored for `/**`; `/actuator/health`, `/actuator/info` permitAll; other `/actuator/**` denyAll; `anyRequest().permitAll()`; an empty `InMemoryUserDetailsManager` bean (avoids the generated default password). **There is no authentication or authorization on the report endpoint** - anyone who can reach the pod network can call it. Presumed compensation: cluster-internal service + upstream gateway (inference; not in repo). No rate limiting, no API key header, no JWT.

## 2.6 Validation, idempotency, timeouts
* Validation: `spring-boot-starter-validation` is on the classpath but `PayLoadRequest` has no `@Valid`/constraints; validation is by convention inside services (`getOrDefault`). The controller's `@RequestBody` has no `@Valid`. Outbound params are sanitised (`PlineUtilsClient.sanitize`, regex `[A-Za-z0-9_\-./#]*`).
* Idempotency: report generation is a read-only pipeline (only SELECT-like aggregations and GETs), so repeated POSTs return the same data apart from `timestamp` and any data changes - effectively idempotent but modelled as POST. No idempotency key. (inference)
* Timeouts: inbound - none configured (`server.tomcat.*`/`spring.mvc.async` not set; Tomcat default connection timeout 20 s applies to idle connections, not processing). Mongo aggregates: `maxTime 600 s` (10 min) only in handlers that set `AggregationOptions` (see 3.5); `RejectRiskHandler` sets none. Outbound: see 1.7.

## 2.7 Sample requests (realistic; values are illustrative, keys are the ones the services read)
Common metadata block (all three):
```json
"metadata": {
  "name": "Jane Doe", "email": "jane@example.com", "senderMail": "noreply@example.com",
  "copyType": "ORIGINAL", "batchConfigId": "1001", "boResId": "2002", "channel": "PRINT",
  "dataClass": "1", "disallowCopy": "N", "docType": "PDF", "linkLocation": "",
  "outputFormat": "PDF", "outputPath": "/out", "templateId": "55", "templateName": "RejectRisk",
  "reportName": "AGIC/UW/Plines/RejectRisk/RejectRisk",
  "mergeStaticPdfFiles": []
}
```
**RejectRisk** (keys from `RejectRiskService.java:~44-52`: `pCreatedDateFrom`, `pCreatedDateTo`, `pCustomerName`, `pIdNo`, `pRejType`, `pReasonCode`, `pReportTitle`, `reportName`; all optional - missing ones just don't filter; the date format accepted is whatever `plines/utils/DateUtils.toLocalDate` parses (not verified - check the class)):
```json
{
  "metadata": { "reportName": "AGIC/UW/Plines/RejectRisk/RejectRisk", "outputFormat": "PDF", "channel": "PRINT", "dataClass": "1" },
  "filterCondition": {
    "reportName": "AGIC/UW/Plines/RejectRisk/RejectRisk",
    "pReportTitle": "Reject Risk Listing",
    "pCreatedDateFrom": "2026-01-01", "pCreatedDateTo": "2026-01-31",
    "pCustomerName": "", "pIdNo": "", "pRejType": "", "pReasonCode": ""
  }
}
```
**ASP EndtDocument** (keys `AspEndtDocumentService.java:215-230`; defaults: `showCoverNote/showEndtSchedule/showComplaint/showDebitNote/showTaxInvoice` = "N", `pFirstPageNumber` = "1", others ""; `pContractId`/`versionNo` must be numeric text (handlers parse to Long via `safeNum`)):
```json
{
  "metadata": { "reportName": "AGIC/UW/Plines/ASPEndtSchedule/EndtDocument", "outputFormat": "PDF", "channel": "PRINT", "dataClass": "1" },
  "filterCondition": {
    "reportName": "AGIC/UW/Plines/ASPEndtSchedule/EndtDocument",
    "pContractId": "123456", "pEndtNo": "E/2026/000123", "pRefNo": "REF0001",
    "pPolicyNo": "070412/2026/0001", "versionNo": "2", "companyParamType": "AGIC",
    "pFirstPageNumber": "1", "checklistCopy": "N", "printSeries": "", "copyType": "ORIGINAL", "pUserID": "USER01",
    "showCoverNote": "Y", "showEndtSchedule": "Y", "showComplaint": "Y", "showDebitNote": "Y", "showTaxInvoice": "Y"
  }
}
```
(`companyParamType` value "AGIC" and the number formats are illustrative; confirm real values from a captured production request.)
**Template12 PolicyDocument** (keys `Template12PolicyDocumentService.java:~148-173`; defaults: `versionNo` "1", `checklistCopy` "N", `showComplaint` "Y", `showScheduleStd` "Y", other `show*` "N", `ShowOutPrintCovLetter` "N" - note the capital S key):
```json
{
  "metadata": { "reportName": "AGIC/UW/Plines/PolicySchedule/Template12/PolicyDocument", "outputFormat": "PDF", "channel": "PRINT", "dataClass": "1" },
  "filterCondition": {
    "reportName": "AGIC/UW/Plines/PolicySchedule/Template12/PolicyDocument",
    "pContractId": "123456", "pPolicyNo": "070504/2026/0001", "versionNo": "1", "companyParamType": "AGIC",
    "checklistCopy": "N", "showCoverNote": "Y", "showComplaint": "Y", "showScheduleStd": "Y",
    "showDebitNote": "Y", "showTaxInvoice": "Y", "showPaymentInfo": "N",
    "printSeries": "", "copyType": "ORIGINAL", "pCertInd": "", "ShowOutPrintCovLetter": "N",
    "pType": "", "pFirstPageNumber": "1", "pMstPolicyNo": "", "pContractIdList": "",
    "pCertNoFrom": "", "pCertNoTo": "", "pTaxInvNo": "", "pGstTaxInvoiceNo": ""
  }
}
```
Branching driven by data not request: product code 070504/070412/070406/070414 and scheme names (BMW, BMW1, POSMESRA, FWPA, POSCOVER) are looked up from Mongo (`Template12PolicyDocumentService.java:~190-205`).

## 2.8 Outbound API catalog for the three modules
All are HTTP GET unless stated; all failures are swallowed (log + null/"" return) - **none throws to the controller**, so an upstream outage produces a *valid-looking report with blank fields*, not an error. (evidence per method in `PlineUtilsClient.java`).
| # | Client.method | Property key(s) | Path | Params | Used by (module) | Failure | Timeout bean |
|---|---|---|---|---|---|---|---|
| 1 | `PlineUtilsClient.fnGetEinvoicePrintPolInd` | `opus.fn-einvoice-print-pol-ind-path` | `/ext/utils/api/v1/pg-rpge-utils/invoice-print-indicator` | contractId, checklistCopy (sanitised) | Template12 (`PlinePolicyDocumentMainHandler`, `PlineT12PolicyDocumentHandler`) | `safeGet` -> "" | fastLookup 8 s |
| 2 | `fnGetCoInwardInd` | `opus.fn-get-co-inward-ind-path` | `/ext/utils/api/v1/pg-rpge-utils/co-inward-ind` | contractId, versionNo | Template12 | "" | fastLookup |
| 3 | `fnShowStaxInvInd` | `plines.fn.show.stax.inv.ind.path` | `/ext/policy/api/v1/stax-inv` | contractId, checklistCopy | Template12 (main + T12 handlers) | `catch` -> default | fastLookup |
| 4 | `fnEinvoicePrintEndtInd` | `opus.fn-einvoice-print-endt-ind-path` | `/ext/utils/api/v1/pg-rpge-utils/invoice-print-endt-indicator` | contractId, endtNo, refNo, versionNo, checklistInd | ASP (`AspEndtDocument2MainHandler.java:122`) | "" | fastLookup |
| 5 | `fnGetCompBrnNumber` | `plines.fn.comp.brn.number.path` | `/ext/utils/api/v1/pg-rpge-utils/comp-brn-number` | compCode (= companyParamType), issueDate (normalised to ISO; blank/unparseable date skips the call and returns null) | Template12 (`PlineT12TransitionalInvoiceHandler.java:43`, base service), ASP (CompanyNameWithGSTRegNo, Signature) | null | fastLookup |
| 6 | `fnRppmGetSignature`, `fnRppmGetSignatureAndRemarks`, `fnRppmGetSignatureName` | `plines.fn.rppm.get.signature.path` (+ type default `UW`) | `/ext/customer/api/v1/fn-rppm-get-signature` | asOfDate, type | ASP (`PlineCommonSignatureHandler.java:74,131`) | null/"" | fastLookup |
| 7 | `PlineSSTRolloutDateClient.getSSTRolloutDate` | `plines.api.base-url`, `plines.sst.rollout.date.path` | `/ext/policy/api/v1/pg-uwge-utils/sst-rollout-date` | none | ASP ServiceTaxInvoice (`AspServiceTaxInvoiceHandler.java:86`), Template12 PremiumDebitNote (`PlineT12PremiumDebitNoteHandler.java:49`) | `RestClientException`/`Exception` -> null | **primary** (5 s/20 s) |
| 8 | `PlinePolicyRefClient.getPolicyReference` | `plines.api.base-url`, `plines.policy.ref.pme.path` | `/ext/policy/api/v1/pme-api/get-policy-ref` | `contractId` (validated `[A-Za-z0-9_\-]+`; others return null) | ASP ServiceTaxInvoice (`:83`) | null | primary |
| 9 | `PlineEinvoiceQrIndicatorClient.getQrIndicator` | own client (`generic/PlineEinvoiceQrIndicatorClient.java`, not read in detail) | `.../einvoice/qr-indicator` (key `plines.fn.einvoice.qr.ind.path` or `einvoice.qr.path`: not verified) | contract, endt, ref | ASP ServiceTaxInvoice (`:296`) | `safeCall` -> null | not verified |
| 10 | `PlinePgRpgeUtilsClient.getCompBrnNumber` | `plines.api.base-url`, `plines.fn.comp.brn.number.path` | comp-brn-number | compCode, issueDate (ISO) | company-name subreport via `fwpaResilientUtilsClient.getCompBrnNumber` (note: field name says "PlineUtilsClient", the call is on `PlineUtilsClient.getCompBrnNumber(String,String)` at `PlineUtilsClient.java:429`, uses `restTemplate.exchange`) | null | fastLookup |
RejectRisk makes **no external call**: Mongo only (`CMUW_PL_REFER_RISK` + lookups + `RPGE_RPT_TEMPLATE` via `PlineReportHeaderHandler`).
SSRF guards (evidence `PlineUtilsClient.java`): (a) base host fixed from config, request URL built from base + configured path - callers cannot supply a host; (b) `assertTrustedHost(url)` requires `url.startsWith(baseUrl)` before every call (`safeGet`, `safeGetEncoded`, `postForFirstDataEntry`, `fnGetCompBrnNumber`); throws `SecurityException` which, in `safeGet`, is thrown *outside* the try so it propagates (500). Weakness (inference): `startsWith` on a base without trailing slash can match `https://host.evil`, but `baseUrl` here is configured by operators, not users; (c) query values restricted by `sanitize()` regex (anything else becomes "" - silently); (d) `PlinePolicyRefClient` validates `contractId` with `[A-Za-z0-9_\-]+` and uses `build(true)` (already-encoded assumption); (e) `PlineEinvoiceQrIndicatorClient`/`Pline*Client` not audited for `startsWith` host check. The `restTemplate.getForObject(String url...)` template-expansion pitfall (`#`/`%`) is documented at `PlineUtilsClient.java:~597-606`.
Retries: none in the Pline clients (only motor `PMEGetContractIdClient` has `ext.pme.retry.*`). No circuit breaker. 

---------------------------------------------------------------------------------------------------

# PART 3 - DATABASE CATALOG (Phase 9)

## 3.1 Conventions and caveats
* The DB is MongoDB; handlers use `MongoTemplate.aggregate(Aggregation.newAggregation(ops), "<COLLECTION>", Dto.class)` with raw `Document` stages (`ctx -> new Document("$match", ...)`). No `@Document`, no repositories for these modules.
* Collection names mirror Oracle objects (`UWGE_*` general UW tables, `UWPL_*` Plines tables, `*_VI_*` Oracle **views**, `CMGE_CODE` code table, `SAPM_SYS_CONSTANTS`, `DMAG_*` agents, `CPGE_*` customer/partner). Javadocs quote the original JRXML SQL (e.g. `AspDebitNoteHandler.java:~55-63`: "LEFT JOIN CMGE_CODE STATE ON STATE.CODE_CD = VI.STATE AND STATE.CAT_CODE = 'STATE'"). So the Oracle equivalent is the same-named table/view (evidence in javadocs; the Oracle schema owner is `CUSTOMER` per comments such as `CUSTOMER.PG_RPGE_UW_LISTING...` in properties line 141).
* WIP vs live: a prefix `WIP_` marks the work-in-progress copy (used when `checklistCopy = Y`, i.e. a checklist/preview copy before issue), unprefixed = posted/live. `SB_UWGE_POLICY_CONTRACTS` exists alongside (`AspEndtDocumentMainHandler.java:~21`, comment "WIP ∪ SB contracts"); `OCP_POLICY_BASES` appears in `AspDebitNoteHandler.java`. Meanings of `SB_` and `OCP_` are **not documented in the files I read** (uncertainty; SB possibly "standby/submitted", OCP possibly an older/original contract policy base).
* No foreign keys (Mongo has none). No index or schema definition exists in this repo (`auto-index-creation` ineffective; no `createIndex` calls). The indexes below are **recommendations** inferred from the query shapes; the real cluster may already have them (Atlas).

## 3.2 Collections used by the three modules
Key: P = primary (driver of the pipeline), L = lookup/secondary read. Join keys in parentheses.
### RejectRisk (`RejectRiskHandler.java`)
| Collection | Oracle equiv | Role | Fields used | Join/filter keys | WIP/live |
|---|---|---|---|---|---|
| `CMUW_PL_REFER_RISK` (line 30 const `REFER_RISK`) | table `CMUW_PL_REFER_RISK` | **P** | CREATED_DATE, REJ_TYPE, REASON_CODE, CP_PART_ID, CP_PART_VERSION, REMARKS, CREATED_BY, REVOKE_IND, REVOKED_DURATION, REVOKE_BY, REVOKE_DATE, UPDATED_BY, UPDATED_DATE | filter: CREATED_DATE range, REJ_TYPE, REASON_CODE; join: (CP_PART_ID, CP_PART_VERSION) | live only |
| `CPGE_VI_PARTNERS_ALL` | view | L | PART_ID, VERSION, NAME_EXT, ID_TYPE1, ID_VALUE1, ID_TYPE2, ID_VALUE2 | PART_ID=CP_PART_ID, VERSION=CP_PART_VERSION; also filters by NAME_EXT, ID_VALUE1/2 after join | - |
| `CMGE_CODE` (x3 lookups) | code table | L | CAT_CODE, CODE_CD, CODE_DESC | CAT_CODE in (`PLREJ_RISK_TYPE`->REJ_TYPE, `PLREJ_REASON`->REASON_CODE, `PLREJ_REVOKE_DUR`->REVOKED_DURATION) | - |
| `RPGE_RPT_TEMPLATE` (`PlineReportHeaderHandler.java:31`) | table | L | DATA_CLASS by (reportTitle, reportName) | | - |
### ASP EndtDocument (`AspEndtDocumentService`)
Main V1 `AspEndtDocumentMainHandler`: `UWGE_VI_ENDT_REPORT_PARAM` (ERP, view; show flags, ISSUE_DATE, keyed CONTRACT_ID + ENDT_NO/REF_NO), `WIP_UWGE_POLICY_CONTRACTS` + `SB_UWGE_POLICY_CONTRACTS` (PRODUCT_CONFIG_CODE by CONTRACT_ID), `UWGE_POLICY_VERSIONS` + `WIP_UWGE_POLICY_VERSIONS` (ENDT_CODE by CONTRACT_ID + VERSION_NO), primary scan on `UWMT_VI_ENDT_TAX_INV_ALT_UPV` (ANC, matched ENDT_NO + CONTRACT_ID). V2 `AspEndtDocument2MainHandler`: `UWGE_POLICY_CONTRACTS`, `UWGE_POLICY_VERSIONS` (+ utils call #4). Others: `UWGE_POLICY_VERSIONS_CTRL` (print count, `AcpPrintSeriesFooterHandler`), `UWPL_VI_ENDT_SCHEDULE_SUMMARY` + `DMT_AGENTS`/`DMAG_AGENTS`/`CMDM_BRANCH`/`CMGE_CODE` (EndtCover), `UWGE_POLICY_BASES`/`UWGE_POLICY_CONTRACTS`/`UWGE_POLICY_VERSIONS`/`UWPL_POLICY_BASES`/`DMAG_VI_AGENT`/`CPGE_VI_PARTNERS_ALL`/`CPGE_VI_ADDRESS_ALL` (`AspEndtScheduleHandler`), `UWGE_VI_ENDT_DEBIT` + `UWGE_POLICY_BASES`/`WIP_UWGE_POLICY_BASES`/`OCP_POLICY_BASES`/`UWPL_POLICY_BASES`/`DMAG_VI_AGENT`/`CMUW_BANKBRANCH`/`CMGE_CODE` (debit note), `UWPL_VI_ENDT_TAX_INV`, `UWPL_VI_TAX_INVOICE`, `CMUW_ENDT`, `SAPM_SYS_CONSTANTS`, `UWGE_POLICY_BASES_STAX` (service tax invoice), `UWGE_COMPLAINT_REDRESS`, landscape: `UWPL_VI_ENDT_ADD_DEL_RISK`, `UWPL_VI_ENDT_SCHEDULE_TPL4`, `UWPL_VI_ENDT_SCHEDULE_NOM`, `UWPL_VI_ENDT_SCHEDULE_SPONSOR`, `UWGE_RISK`, `UWGE_VI_ENDT_REPORT_PARAM`. (evidence: constants lists via grep of each handler; the ones from classes not opened — `PlineEndtTaxDocHandler`, `PlineTransitionalInvoice*Handler`, `PlineGenericQRSectionHandler`, `AcpServiceTaxInvoiceEndtMT*` — were not grepped, so this list is not guaranteed exhaustive.)
### Template12 PolicyDocument
Main `PlinePolicyDocumentMainHandler.java:~26-52` constants: `UWGE_VI_POL_REPORT_PARAM`, `UWGE_POLICY_BASES`/`WIP_UWGE_POLICY_BASES`, `UWPL_VI_POLICY_SCHEDULE_BASE`, `SAPM_SYS_CONSTANTS`, `UWPL_POLICY_BASES`/`WIP_UWPL_POLICY_BASES`, `UWGE_GST_TAX_INV`, `UWPL_VI_FWIG_LETTER`, `UWGE_VI_POL_COVERNOTE`, `UWGE_POLICY_CONTRACTS`/`WIP_`/`SB_`, `UWPL_POLICY_JOURNEY`/`WIP_`, `UWPL_RISK_PERSON`/`WIP_`, `UWPL_COVER`/`WIP_`, `UWPL_VI_POLICY_SCHEDULE_PROV`, `UWGE_VI_POLICY_SCHEDULE_BASE`, `DMAG_VI_AGENT`, `UWPL_COVER_BENEFIT`/`WIP_`, `UWGE_RISK`/`WIP_`, `CMGE_CODEREL_DET`. `PlineT12PolicyDocumentHandler`: `UWGE_VI_POL_REPORT_PARAM`, `UWPL_POLICY_BASES`, `UWPL_POLICY_JOURNEY`, `UWGE_GST_TAX_INV`, `UWGE_POLICY_BASES`, `UWPL_RISK_PERSON`, `UWGE_COVER`. Landscape dispatch (`Template12LandscapeDispatchHandler`, `Template4LandscapeDispatchHandler`): `UWPL_RISK_NOMINEE`, `UWPL_COVER_DIAGNOSIS`, `UWPL_RISK_COMPANION`, `UWPL_POLICY_JOURNEY`, `UWGE_POLICY_VERSIONS`, `UWGE_POLICY_CONTRACTS`, `UWPL_RISK_PERSON`, `UWGE_COVER`, `UWGE_COVER_SPONSOR`, `SAPM_SYS_CONSTANTS`, `UWGE_POLICY_BASES`, `UWPL_POLICY_BASES`, `CMGE_CODEREL_DET`. T12 sub-reports: `UWPL_VI_POLICY_SCHEDULE_NOM/DIAG/BEN/PREM/TPL4/TPL20` (views per schedule band), `UWPL_VI_PREMIUM_DEBIT`, `UWPL_VI_TAX_INVOICE`, `CPGE_VI_PARTNERS_ALL`, `CMGE_CODE`, `MST_POL_NO` (a field used for master-policy joins).
Core key throughout: `CONTRACT_ID` (numeric) + `VERSION_NO` (numeric); endorsement: `ENDT_NO`, `REF_NO`; policy: `POLICY_REF`/`MST_POL_NO`; code tables: `CAT_CODE` + `CODE_CD`; customer: `PART_ID` + `VERSION`.

## 3.3 Indexes (RECOMMENDATIONS - none are defined in this repo)
Query shapes -> suggested indexes (verify with `explain("executionStats")` first; an index may already exist in Atlas):
* RejectRisk: `CMUW_PL_REFER_RISK`: `{CREATED_DATE:1}` (the date range is the main selective filter; the code deliberately avoids `$dateTrunc` so an index can be used - comment `RejectRiskHandler.java:~37-41`); optionally compound `{REJ_TYPE:1, REASON_CODE:1, CREATED_DATE:1}` if those filters are common. `CPGE_VI_PARTNERS_ALL`: `{PART_ID:1, VERSION:1}`. `CMGE_CODE`: `{CAT_CODE:1, CODE_CD:1}` (used by ~every handler - highest value).
* Contract-keyed reads (Template12/ASP): `{CONTRACT_ID:1, VERSION_NO:1}` on `UWGE_POLICY_BASES`, `WIP_UWGE_POLICY_BASES`, `UWGE_POLICY_VERSIONS`, `WIP_UWGE_POLICY_VERSIONS`, `UWPL_POLICY_BASES`, `UWPL_POLICY_JOURNEY` (+WIP), `UWPL_RISK_PERSON` (+WIP), `UWPL_COVER` (+WIP), `UWPL_VI_POLICY_SCHEDULE_BASE`, `UWGE_VI_POL_REPORT_PARAM`; `{CONTRACT_ID:1}` on `*_UWGE_POLICY_CONTRACTS`.
* Endorsement-keyed: `{CONTRACT_ID:1, ENDT_NO:1}` and `{REF_NO:1}` on `UWMT_VI_ENDT_TAX_INV_ALT_UPV`, `UWGE_VI_ENDT_REPORT_PARAM`, `UWGE_VI_ENDT_DEBIT`, `UWPL_VI_ENDT_SCHEDULE_SUMMARY`, `UWPL_VI_ENDT_TAX_INV` (the `$or` on ENDT_NO/REF_NO in `AspDebitNoteHandler` means each branch needs its own index to avoid a COLLSCAN; the class javadoc itself says the real Oracle query had no CONTRACT_ID).
* Master-policy chain: `{POLICY_REF:1, VERSION_NO:1}` on `OCP_POLICY_BASES`.
* Agents: `DMAG_VI_AGENT {AGENTCODE:1}`.
Caveat: `*_VI_*` "views" in Mongo are either materialised collections or real Mongo views; **a real Mongo view cannot have its own indexes** (it uses the source collection's) - which one applies is not known from the repo.

## 3.4 Aggregation patterns (5 annotated examples from real handlers)
**Example 1 - date range without functions on the field (RejectRiskHandler.java:46-61).**
```java
if (dateFrom != null) dateFilter.append("$gte", dateFrom);
if (dateTo != null)   dateFilter.append("$lt", dateTo);   // dateTo = pCreatedDateTo + 1 day
```
Business: Oracle `TRUNC(CREATED_DATE) BETWEEN from AND to` is inclusive of the whole last day; since CREATED_DATE is a full timestamp, the upper bound is shifted +1 day and compared exclusively - index-friendly. Only supplied bounds are added, because a `null` bound would match only null dates (BSON ordering) and return zero rows. Gotcha: `dateFrom` is built with `ZoneId.of("UTC")`; if CREATED_DATE was stored in local time (+08:00), boundary rows near midnight may shift (inference).
**Example 2 - correlated `$lookup` with `let`/`pipeline` and `$limit:1` = LEFT JOIN scalar (RejectRiskHandler.java:~71-80, 91-102).**
```java
new Document("$lookup", new Document("from","CMGE_CODE")
  .append("let", new Document("rtc","$REJ_TYPE"))
  .append("pipeline", List.of(new Document("$match", new Document("$expr", new Document("$and", List.of(
        new Document("$eq", List.of("$CAT_CODE","PLREJ_RISK_TYPE")),
        new Document("$eq", List.of("$CODE_CD","$$rtc")))))), new Document("$limit",1)))
  .append("as","RRT_ROW"))
```
followed by `$addFields {RRT: {$arrayElemAt:["$RRT_ROW",0]}}` (instead of `$unwind`, so a missing code keeps the row = LEFT JOIN semantic; AspDebitNoteHandler uses `$unwind ... preserveNullAndEmptyArrays:true` for the same effect, lines 215-216). Performance: one sub-pipeline per input row; efficient only if `CMGE_CODE` has an index on CODE_CD/CAT_CODE (the `$expr`+`$eq` form can use an index in modern MongoDB, but verify with explain).
**Example 3 - post-join filter ordering (RejectRiskHandler.java:~82-90).** `customerName`/`idNo` matches (`CUST.NAME_EXT`, `$or` on `CUST.ID_VALUE1/2`) are applied AFTER `$lookup` to CPGE_VI_PARTNERS_ALL because the fields live in the joined collection. Therefore the date/type/reason `$match` stages before the lookup are what limit work; a request with only name/ID filters and no dates would `$lookup` every refer-risk row (potential full scan). Also exact-equality on `NAME_EXT` (case-sensitive) - Oracle used the parameter the same way (inference).
**Example 4 - WIP ∪ live fallback with `$ifNull` (AspEndtDocumentMainHandler.java:~72-118).**
```java
.append("PRODUCT_CODE", new Document("$ifNull", listOf(
        firstElem("$PC_WIP.PRODUCT_CONFIG_CODE"),
        firstElem("$PC_SB.PRODUCT_CONFIG_CODE"))))
```
Two `$lookup`s (one into `WIP_UWGE_POLICY_CONTRACTS`, one into `SB_UWGE_POLICY_CONTRACTS`), each with `$project` + `$limit:1`, then `$ifNull` picks the first non-null. `ENDT_CODE` uses the checklist switch: the WIP lookup runs only when `$$chk == "Y"`, the live lookup (`UWGE_POLICY_VERSIONS` filtered by `VERSION_NO`) only when `"N"` (the `$eq: ["$$chk","N"]` inside the sub-pipeline's `$expr` makes the other branch return empty). Contrast with Template12's `isWip ? WIP_x : x` collection switching in Java (`PlinePolicyDocumentMainHandler.java:95,141`). Note the base pipeline starts at `UWMT_VI_ENDT_TAX_INV_ALT_UPV` with `$match ENDT_NO + CONTRACT_ID` then `$limit 1` (lines 46-49) - the driver collection is a tax-invoice view, a quirk inherited from the JRXML (inference).
**Example 5 - separate sequential point-queries instead of one big pipeline (PlinePolicyDocumentMainHandler.java:~76-135; AspDebitNoteHandler.fetchMstContractInfo ~130-195).** Template12 runs "Step A..Step G" as many small aggregations (`match` -> `project` -> `limit 1`) each with `AGG_OPTIONS` (maxTime 600 s + allowDiskUse), e.g.
```java
Criteria baseCriteria = Criteria.where("CONTRACT_ID").is(asNum(contractId));
if (!isWip) baseCriteria = baseCriteria.and("VERSION_NO").is(1);
```
The ASP debit note resolves the master-policy chain (`UWGE_POLICY_BASES` + `WIP_UWGE_POLICY_BASES` -> `MST_POL_NO` list -> `OCP_POLICY_BASES.POLICY_REF $in [...]` -> `CONTRACT_ID` -> `PREM_BY_MST_IND`) with 4-5 round trips mirroring nested Oracle scalar sub-queries. Trade-off: simple, debuggable, each step index-friendly, but N network round trips per request (latency + driver pool use) and no atomic snapshot between steps. `VERSION_NO = 1` is hard-coded when not WIP (the Oracle JRXML read the original version; inference).
**Bonus pattern - `$setWindowFields` `$documentNumber` (RejectRiskHandler.java:~150-153)** reproduces Oracle `ROWNUM` after `ORDER BY NAME_EXT` (requires MongoDB 5.0+; `$sort` follows for output order). Using `$setWindowFields` with `sortBy` on a computed field forces an in-memory sort of the full result (100 MB limit per stage unless `allowDiskUse`) - RejectRisk sets no `AggregationOptions`, so a very large date range could hit "Sort exceeded memory limit" (inference; real limit is 100 MB per blocking stage).

## 3.5 Performance notes
* `allowDiskUse(true)` + `maxTime(600 s)`: set in `PlinePolicyDocumentMainHandler` (static `AGG_OPTIONS`, ~lines 54-57), `AspDebitNoteHandler`, `AspEndtScheduleHandler`/`AspEndtCoverHandler` (per the debit-note comment), `AcpEndtInstallmentDetailsHandler`, `AcpCommonCompanyNameBRNUpperAndLowercaseHandler`, `AcpComplaintsAndGrievancesHandler` (allowDiskUse only). **Not set** in `RejectRiskHandler` and `AspEndtDocumentMainHandler`. 10-minute maxTime is far above the 20 s HTTP read timeout of the callers (JasperServer side unknown) -> orphan queries can keep running after the client gave up.
* `$limit` placement: `$limit:1` is placed inside `$lookup` sub-pipelines and after `$match` in point-queries - good. In RejectRisk the final result is unbounded (a report listing), no `$limit` after `$sort`.
* `$lookup` without index on the foreign field = per-row collection scan (O(n*m)). With `let` + `$expr` the planner can use an index on the foreign field for equality in recent MongoDB versions; check with explain.
* Type strictness: `CONTRACT_ID`/`VERSION_NO` are numeric in Mongo; `$match {CONTRACT_ID: "123"}` (string) silently returns nothing. Handlers convert via `safeNum`/`asNum` (`AspEndtDocumentMainHandler.java:154-157`: `Long.parseLong`, fallback to the raw string). `Long` vs stored Int32 compare equal in MongoDB numeric comparison, but a stored string would not. Date fields are stored as BSON dates (RejectRisk) while some outputs are formatted strings (`$dateToString "%d-%b-%Y"` mentioned in `PlineUtilsClient.java:~548`).
* `MongoTemplate`/`Aggregation` DEBUG logging prints pipelines: expensive and leaks identifiers.
* Result mapping: `getMappedResults()` loads all rows into heap; large listings = memory pressure (RejectRisk).

---------------------------------------------------------------------------------------------------

# PART 4 - DEVOPS, RELEASE AND SECURITY (Phases 16-17)

## 4.1 Git branching evidence
* Remote branches: 236 (`git branch -r | wc -l`). Default `origin/HEAD -> origin/retrofit-develop-R1`; also `origin/main`, `origin/develop`, `origin/retrofit-develop-R2` (the long-lived integration branch per `secret-scan.yml`: `fallback_base_ref_pattern: '^retrofit-develop-R2$'`), `retrofit-develop-R2_Spring_version_update`, `retrofit-develop-R1-changes-new3/new6`, `retrofit-develop-R1_29TH_SEPT`.
* Naming patterns: `OPUSM-<ticket>/retrofit-develop-R2-pline-<topic>` (e.g. `OPUSM-41763/retrofit-develop-R2-pline-Template12-ASP-RejectRisk` - the branch for the three modules), `OPUSM-44159/...` (Spring Boot 4 upgrade family: `pline-spring-boot-4-upgrade-R2-Template1-2`, `...-poc-Template8`), `R2/wave1/feature/OPUSM-Bond_DemoDefectFixes...`, `feature/retrofit-develop-R2-<LOB>` (Liability, Misc, marine, plines...), `OPUSM-xxxxx-backup`. Naming is inconsistent (the secret-scan comment, `secret-scan.yml`, states that "essentially zero actual R2 feature branches follow the org's enforced r2/type/JIRA-slug convention").
* Merges: 1251 merge commits among 2845 commits (`git log --merges`). Pattern = GitHub "Merge pull request #NNN from OPUSCCC/<branch>" (seen #878, #906, #908, #910, #914, #916, #923, #929, #931, #932) plus frequent "Merge branch 'retrofit-develop-R2' into <feature>" (sync-from-integration merges), i.e. feature branches are merged back from R2 before PR (evidence from `git log --merges`). PRs target `retrofit-develop-R1`/`R2`; trunk merges to `main` run the changelog bot.
* Last commits on the current branch: `861f0666c`, `e74bc0d5c`, `74ecf94f3`, `edbef120c`, `bca355525` (all `OPUSM-44159 ...`, Oct 2026).
* Commit message rule: JIRA key `OPUSM-<n>` required (`jira-commit-check.yml`, allowed_projects "OPUSM").

## 4.2 Workflows (`.github/workflows`)
| File | Trigger | What it does |
|---|---|---|
| `cicd-workflow.yml` | `workflow_dispatch` only (input `imageTag` optional); `release: created` is commented out | Checks out repo + shared repo `OPUSCCC/opus-github-cicd-workflows` (token `secrets.CICD_READ_REPO`), runs local action `actions/java-microservice` with `java_version 17`, `app_name opus-gi-documentprintdata-service`, Sonar, Wiz secrets. Builds/scans/publishes an image (steps are inside the shared action; **not visible here**). Runner label `adp-runner-medium`. Actions pinned by SHA. |
| `sonar-code-review.yml` | push to `retrofit-develop-R2`/`R3`; PR to `R1`/`R2` | JDK 17 temurin, `mvn -B verify sonar:sonar -Dsonar.qualitygate.wait=true` -> quality gate blocks |
| `codeql.yml` | push `develop`/`main`; PR `develop`/`retrofit-develop`/`main`; weekly cron `0 2 * * 6`; manual | shared SAST action, language java. Note branch filters do not include `retrofit-develop-R1/R2`, the branches PRs actually target (inference: CodeQL rarely runs on PRs here) |
| `secret-scan.yml` | every PR and push | calls shared `secret-scan-template.yml@main`, `mode: diff`, fallback base `retrofit-develop-R2` (OPUSM-57911) |
| `secret-scan-nightly.yml` | cron `0 2 * * *` | same template, `mode: full-history` |
| `jira-commit-check.yml` | PR opened/synchronize/reopened/edited | shared action verifies commits reference an `OPUSM` Jira (uses `JIRA_USER`/`JIRA_PASSWORD` secrets) |
| `claude-pr-review.yml` | PR (non-draft, same-repo) to `retrofit-develop-R1/R2/main`; `@claude` comments by OWNER/MEMBER/COLLABORATOR | AI PR review, 20 min timeout, `GH_HOST=github.developer.allianz.io` |
| `changelog-update.yml` | PR closed/merged to `main`, push `main` | shared changelog workflow pinned by SHA; writes `CHANGELOG.md` (entries PR #626..#864 seen) |
Gaps: the CI/CD build is **manual** (workflow_dispatch) - no automatic deploy on merge; no test-and-deploy per environment visible.

## 4.3 Release and rollback
* Evidenced: image built by the shared action with an optional `imageTag` input; versioning in the pom is a static `0.0.1-SNAPSHOT` (`pom.xml:20`), so the image tag (not the pom version) identifies a release. Changelog bot records JIRA/PR per merge to `main`.
* **NOT in this repo:** Helm chart / Kubernetes manifests, environment values, deployment pipeline (Argo/Flux/Jenkins), canary/blue-green config, DB migration (none - no schema), rollback procedure. Rollback is therefore presumably "redeploy the previous image tag via the Helm repo" (inference; must be confirmed with the platform team). Because there is no schema/DB state owned by this service, a code rollback is stateless; the risk is configuration drift in the (external) Helm values (e.g. the removed Mongo URI).
* Config-as-code risk: the truststore and base URLs live in the jar; a URL change needs a rebuild unless Helm overrides them.

## 4.4 Logging, monitoring
* Logging: stdout via Boot defaults; SLF4J `@Slf4j` in services; `System.out.println` in `CustomRequestFilter` and `CommonController` (e.g. "Transaction object is null"); DEBUG for Mongo. No MDC/correlation id, no JSON log format, no request id propagation to outbound calls.
* Monitoring: actuator `health` and `info` only (no `metrics`/`prometheus`, even though `micrometer` comes with actuator; nothing exposed). No custom health indicators, no alerts in repo.
* Error visibility: `GlobalExceptionHandler.handleGeneric` does not log the exception (see 2.4).

## 4.5 Security review
* **Authn/Authz:** none at app level (`anyRequest().permitAll()`); network-level protection assumed (inference).
* **TLS:** outbound uses a custom truststore (`mytruststore.p12`, committed in `src/main/resources`, password in plaintext in `application-env.properties`). The same keystore password literal is in VCS history since `57c573b74`. Inbound is plain HTTP on 8080 (TLS presumably terminated at ingress).
* **Mongo credentials:** committed history had Mongo URIs (comment in `gitleaks.toml`: "application.properties (18 occurrences, live + commented-out copies ...)", ticket OPUSM-50404). Commit `7dd2f2f67` removed the UAT URI from tracked file, but the working copy currently contains a `#newDev` block with credentials (uncommitted). The URI/user/password are `<redacted>` in this document; **never commit or paste them**. Recommend rotating credentials that appeared in history (inference: history is not rewritten).
* **Gitleaks allowlist risk:** `gitleaks.toml` extends the org ruleset and sets a single `[allowlist]` of two **entire files** (`application.properties`, `application-env.properties`), description "OPUSM-50404 - tracked, owner: opus-documentprintdata-service team, review-by 2026-10-23". Consequence (the file's own comment says so): ANY secret type added to these two files is silently suppressed, including the truststore password and any new Mongo password. The review-by date (2026-10-23) is ten days after today (2026-10-13). Mitigation: move secrets to env/secret store and remove the allowlist.
* **SSRF fixes:** `PlineUtilsClient.assertTrustedHost` + `sanitize` (see 2.8); host never user-controlled. No outbound call uses a URL from the request.
* **CSRF:** disabled for all paths, justified in code as stateless/no cookies (SonarQube S4502 acknowledged). Acceptable only because there is no browser session and no auth.
* **Request body logging:** `CustomRequestFilter` prints full request bodies (`filterCondition` carries contract/policy numbers; RejectRisk carries `pCustomerName`, `pIdNo` = personal data) to stdout; `MongoTemplate` DEBUG logs query documents. PDPA exposure (Malaysia; the business domain is MY - inference from company names/GST/SST). Response XML (customer names, IDs) is logged at INFO on the legacy path (`CommonController`, `log.info("xml response data...")`).
* **CVE pins (pom.xml comments):** Boot 4.1.1 for CVE-2026-59270; Tomcat 11.0.25 (CVE-2026-65905/65182/68525); Jackson 2.21.5 (CVE-2026-54512/54513); commons-lang3 3.18.0 (CVE-2025-48924); logback 1.5.34; log4j-api 2.25.5 (CVE-2026-49844); old pins for Netty/Spring/mongodb removed (BOM higher). `org.json:json:20231013` and `junit-jupiter 5.9.0` are old and unpinned by a Wiz comment (inference). Scanners: Wiz (secrets WIZ_CLIENT_ID/SECRET in the CI action), Sonar, CodeQL, gitleaks.
* **Open docs:** Swagger/OpenAPI exposed without auth.
* **Untrusted deserialisation / injection:** no raw `@Query` string concatenation; Mongo filters built from Java `Document` objects with parameter values (no `$where`/JS) - but `pCustomerName` goes into an equality `Document` (no operator injection because the value is a String, and `Map<String,String>` prevents `{"$ne":...}` objects). Good point to cite.
* XML: `XmlMapper` writes only (no XXE surface); inbound is JSON.

## 4.6 Twenty-five security interview questions with strong answers
1. **Does the report API authenticate callers?** No - `SecurityConfig` is `anyRequest().permitAll()`; protection must come from network policy/gateway. I would state that clearly and propose an API key/JWT filter plus NetworkPolicy. *Mistake:* claiming Spring Security means it is secured.
2. **Why is CSRF disabled and is that acceptable?** Stateless, no cookies, no browser sessions, only machine clients (code comment, Sonar S4502). Acceptable here; unacceptable if cookies/basic-auth sessions were later added.
3. **What is the biggest secret risk in the repo?** Plaintext Mongo URI/credentials and truststore password in `application*.properties`, plus the committed `.p12`. Fix: externalise via secret manager/env, rotate, scrub allowlist.
4. **How does the gitleaks allowlist weaken scanning?** It exempts two whole files (`gitleaks.toml`), so any new secret there passes CI; review-by date 2026-10-23.
5. **Why was `[[allowlist]]` changed to `[allowlist]`?** gitleaks 8.30.1 rejects arrays-of-tables (commit `796191c51`); a single table with a `paths` array is valid; paths-only so it does not mask regexes elsewhere.
6. **How is SSRF prevented in `PlineUtilsClient`?** Fixed configured base URL, path from config, `assertTrustedHost` startsWith check, `sanitize()` on every query value. Residual: startsWith prefix semantics.
7. **What if `contractId` contains `../` or `?`?** `sanitize` allows `.`, `/`, `#` and `-`, letters/digits/underscore - so `../` passes the regex in `PlineUtilsClient` (it's within query values, which `UriComponentsBuilder` encodes, so path traversal on the server is not possible via query param). In `PlinePolicyRefClient` the stricter `[A-Za-z0-9_\-]+` applies.
8. **Why is a custom truststore a risk and a benefit?** Benefit: trust only internal CA. Risk: replaces default cacerts, stale CA after rotation -> outages; password in VCS.
9. **Is data logged that shouldn't be?** Yes: request bodies via `System.out`, Mongo DEBUG, response XML at INFO. Fix: remove prints, mask, DEBUG only in non-prod.
10. **What does `denyAll` on `/actuator/**` give?** Only health/info reachable; `/actuator/loggers` enabled but not exposed, so it can't be abused remotely.
11. **Are errors leaking internals?** No - ProblemDetail returns generic text; BUT swallowed exceptions make triage hard. Balance.
12. **How would you add authentication quickly?** Add a `OncePerRequestFilter` validating a shared secret/JWT from the gateway, `authorizeHttpRequests(...).anyRequest().authenticated()`, keep health permitAll.
13. **NoSQL injection risk?** Filters are built as `Document` with String values from `Map<String,String>`; no operators can be injected; avoid `$where`/`$expr` with user strings.
14. **ReDoS/regex?** Only fixed regexes (`SAFE_PARAM`, `[A-Za-z0-9_\-]+`) - linear.
15. **Dependency CVE handling?** pom properties with CVE comments, Wiz scan, CodeQL; Boot 4.1.1 BOM; remove obsolete pins to avoid downgrades (as comments explain).
16. **Why was Jackson pinned via `jackson-2-bom.version`?** Boot 4 repurposed `jackson-bom.version` for Jackson 3; the wrong property would silently not apply.
17. **Why keep Jackson 2?** DTOs use `com.fasterxml` annotations; Jackson 3 ignores them -> breaks payload; keep until migration.
18. **Is Swagger exposure a problem?** It enumerates `/api/**` (the one endpoint) - low but should be disabled in prod (`springdoc.api-docs.enabled=false`).
19. **Who can read the JSON `mergeStaticPdfFiles` paths?** The service only echoes them into XML for the renderer; confirm the renderer restricts to a static folder (path traversal there is outside this repo).
20. **DoS exposure?** Unauthenticated POST triggering heavy aggregations with 10-min `maxTime` and unbounded rows; mitigate with gateway rate limits, tighter maxTime, pagination.
21. **PII handling?** RejectRisk returns customer name and ID numbers; ensure only internal systems call; don't log; consider masking in listing.
22. **Where are secrets for CI?** GitHub secrets (`CICD_READ_REPO`, `SONAR_PROD_TOKEN`, `WIZ_*`, `JIRA_*`); actions pinned by SHA to avoid supply-chain drift (`cicd-workflow.yml`).
23. **Why does `claude-pr-review.yml` restrict to same-repo PRs and author associations?** To prevent forks/untrusted users from triggering an LLM with repo write tokens (prompt-injection/abuse).
24. **What would you do if a credential was committed?** Revoke/rotate immediately, purge from history if policy requires, add rule to gitleaks without file-wide allowlist, notify security (OPUSM-50404 is the existing tracker).
25. **Is `org.json` use safe?** It parses attacker-controlled JSON in `CustomRequestFilter` with an old version (20231013) and no size limit; potential DoS via deep nesting; upgrade and cap body size (`server.tomcat.max-swallow-size`/`spring.servlet.multipart` don't apply; use a size filter).

---------------------------------------------------------------------------------------------------

# PART 5 - INTERVIEW Q&A (1.6-year level, project-specific)

Format: **Q** / A (strong answer) / Mistake (common) / Follow-up.

## 5.A Thirty configuration questions

1. **Q: What is the context path and why does it matter?** A: `/gi/documentprintdata` (`application.properties:5`); every URL, ingress rule and probe includes it. Mistake: calling `/api/opus/GenerateReport` directly. Follow-up: what is the actuator health URL? (`/gi/documentprintdata/actuator/health`).
2. **Q: What does `spring.profiles.active=dev` do here?** A: Nothing profile-specific - no `application-dev.properties` exists. Mistake: assuming env separation. Follow-up: how would you introduce real profiles?
3. **Q: Is `application-env.properties` a profile file?** A: No; it is loaded by `@PropertySource` in two config classes. Mistake: thinking `env` profile loads it. Follow-up: precedence vs `application.properties`? (PropertySource is lower).
4. **Q: Why `spring.mongodb.*` and not `spring.data.mongodb.*`?** A: Boot 4 moved MongoProperties to `org.springframework.boot.mongodb.autoconfigure` with prefix `spring.mongodb`; old keys are silently ignored and fall to localhost:27017 (comment lines 26-31). Mistake: debugging "connection refused localhost". Follow-up: how to confirm which URI is used? (startup log / actuator not exposed).
5. **Q: Are `option.max-connection-pool-size` etc. applied?** A: No - non-Boot keys read by nobody. Mistake: believing pool is 100/20. Follow-up: how would you set pool size? (URI options).
6. **Q: Is `auto-index-creation=false` effective?** A: No (wrong prefix) and irrelevant (no `@Document` entities). Follow-up: where should indexes be created? (DBA/migration).
7. **Q: Why `spring.http.converters.preferred-json-mapper=jackson2`?** A: Boot 4.1.1 prefers Jackson 3; DTOs use Jackson 2 annotations; commit `9248b12ff`. Mistake: removing it as "unused". Follow-up: what breaks if removed? (HttpMessageConversionException on PayLoadRequest).
8. **Q: How is `API_BASE-URL` read and overridden?** A: `@Value("${API_BASE-URL}")` in ~45 places; override with env var/JVM property/Helm; relaxed binding from `API_BASE_URL` is expected but should be verified. Mistake: only changing `plines.api.base-url`. Follow-up: why is a hyphen risky in env vars?
9. **Q: Which keys determine where Template12/ASP external calls go?** A: `API_BASE-URL` (PlineUtilsClient) and `plines.api.base-url` (Pline*Client) plus path keys. Mistake: forgetting the second. Follow-up: how would you unify?
10. **Q: dev1 vs dev2 hosts in the file - which matter?** A: Active: dev2 (lines 76, 187, 227). dev1 keys (113,121,125,128,139) are unread. Mistake: deleting the active one.
11. **Q: What is the effect of the `//can be removed` comments?** A: None at runtime; they are commented keys containing old hostnames. Follow-up: any risk deleting? (none).
12. **Q: Which timeout applies to `PlineUtilsClient`?** A: 8 s (`app.http.fast-lookup.read-timeout-ms`), connect 5 s, connection-request 3 s. Mistake: saying 20 s. Follow-up: which clients use the 20 s bean?
13. **Q: Which RestTemplate does `PlinePolicyRefClient` get?** A: Unqualified -> `@Primary` motor `restTemplate`. Mistake: assuming `plineRestTemplate`. Follow-up: consequence for pool limits?
14. **Q: What happens if the truststore password is wrong?** A: Bean creation throws at startup; app does not start. Follow-up: how would you rotate?
15. **Q: Why two RestTemplate config classes?** A: Motor legacy vs Pline (Phase 1/2 incident fix: bounded timeouts, pool sizing, dedicated fast-lookup client). Follow-up: why `destroyMethod="close"`?
16. **Q: Key read without default and not defined?** A: Required ones like `opus.fn-decide-mt-cert-path` are defined; `api.fn.*` only in commented class. Mistake: assuming defaults. Follow-up: what does `${x:default}` do?
17. **Q: Where is `plines.api.customer.signature.type` defined?** A: Not defined; code default `UW`. Follow-up: how is it different from `api.customer.signatureType`?
18. **Q: Why are there three keys for the same comp-brn-number path?** A: Organic growth - each LOB added its own (`plines.*`, `workmen.*`, `misc.*`, legacy `api.pgRpgeUtils.*`). Risk: a backend path change needs N edits. Follow-up: how to refactor? (single property + per-LOB alias).
19. **Q: What's wrong with `pg-uwge-utils.path.get-policy-ref`?** A: Missing `/ext/policy` prefix and underscore; likely 404. Follow-up: which client reads it? (motor `PolicyRefClient`).
20. **Q: `pme-api.path.get-contract-id` value?** A: points to `get-policy-ref` - naming mismatch trap.
21. **Q: Which logging keys are stale?** A: Those targeting `com.agcs.fo.opusMotorGateway.*` and `service.com.agcs...`. Follow-up: how to set DEBUG for your handlers? (`logging.level.com.agcs.fo.opus.plines.documentHandler=DEBUG`).
22. **Q: Is MongoTemplate DEBUG safe in prod?** A: No - verbose, includes values. Follow-up: how to flip at runtime? (loggers endpoint is enabled but not exposed).
23. **Q: What does `management.endpoints.web.exposure.include` expose?** A: health, info; security denies others. Follow-up: how to add prometheus?
24. **Q: How are Swagger paths configured?** A: springdoc keys (41-45) + `GroupedOpenApi` `/api/**`.
25. **Q: Why is `spring.messages.basename=messages` harmless?** A: No bundle exists; Boot ignores a missing basename.
26. **Q: Difference between `server.port` and the K8s service port?** A: Container listens 8080; Helm maps. Mistake: changing only one.
27. **Q: Where is the Java version/boot version set?** A: pom: Java 17, Boot parent 4.1.1; wrapper Maven 3.9.11.
28. **Q: Why was Mongo URI removed from tracked config in 7dd2f2f67?** A: Secret hygiene after a Claude PR review/ gitleaks gating (commit title "Claude PR review fix"). Follow-up: where does the URI come from now? (external; working copy local only - unknown for deployed env).
29. **Q: How would you run locally against another DB?** A: Provide `SPRING_MONGODB_URI`/`SPRING_MONGODB_DATABASE` env vars or `--spring.mongodb.uri=...` rather than editing the file. Mistake: committing edited properties.
30. **Q: What is the risk of `@Value` constructor injection of base URL with leading space?** A: `Properties` trims leading whitespace so it works; trailing whitespace would not be trimmed (line 76 has none). Follow-up: how does `url()` treat trailing slash?

## 5.B Thirty API questions

1. **Q: How many REST endpoints does the service really have?** A: One business endpoint, POST `/api/opus/GenerateReport`, plus a trivial GET `/getJSON`. Mistake: assuming one controller per report.
2. **Q: How does the controller choose the report?** A: `filterCondition.reportName` normalised, then bean-name lookup in `Map<String,ReportStrategy>` and legacy switch fallback. Follow-up: how do you add a new report? (a `@Service("<reportName>")` implementing `ReportStrategy`).
3. **Q: What are the three modules' bean names?** A: `AGIC/UW/Plines/RejectRisk/RejectRisk`, `AGIC/UW/Plines/ASPEndtSchedule/EndtDocument`, `AGIC/UW/Plines/PolicySchedule/Template12/PolicyDocument`.
4. **Q: What does the response look like?** A: `{status, data(XML string), timestamp}`. Mistake: expecting JSON rows or a PDF.
5. **Q: Does the service produce the PDF?** A: No - it returns XML data; JasperServer (outside this repo) renders. Follow-up: who consumes? (reporting layer; inference).
6. **Q: What happens when no rows are found?** A: HTTP 200 with a default empty transaction (or empty reportDataList). Mistake: expecting 404.
7. **Q: When do you get 404?** A: Only when a thrower raises `NoDataFoundException`.
8. **Q: When 400 vs 500?** A: `IllegalArgumentException` -> 400; everything else (including malformed JSON, null pointers) -> 500 via catch-all.
9. **Q: What if `metadata` is missing?** A: NPE in `populateMailingDetails` -> 500.
10. **Q: Why is `FilterCondition` capitalised in the DTO?** A: Field name; Lombok creates `getFilterCondition`; Jackson uses `filterCondition`. Mistake: sending `FilterCondition`.
11. **Q: Can `filterCondition` hold arrays?** A: Not generally (Map<String,String>); `pBranch`/`pBranchName` arrays are flattened by `CustomRequestFilter`.
12. **Q: What does `CustomRequestFilter` do on invalid JSON?** A: Passes it through; empty body becomes `{}`.
13. **Q: Where does `mergeStaticPdfFiles` live?** A: In metadata (custom setter lifts it) or top-level; ends up in `TemplateDetails`.
14. **Q: How are numbers in metadata handled?** A: Converted to strings; booleans/nulls dropped.
15. **Q: How is authentication done?** A: It isn't (permitAll).
16. **Q: Why is CSRF disabled?** A: stateless API.
17. **Q: What are the key request fields for RejectRisk?** A: `pCreatedDateFrom/To`, `pCustomerName`, `pIdNo`, `pRejType`, `pReasonCode`, `pReportTitle`, `reportName`.
18. **Q: Key fields for ASP EndtDocument?** A: `pContractId`, `pEndtNo`, `pRefNo`, `versionNo`, `companyParamType`, `checklistCopy`, `show*` flags.
19. **Q: What do the `show*` flags do?** A: Gate sub-reports (cover note, schedule, complaint, debit note, tax invoice) - combined with data-driven gates (product code, `SHOW_GST_INV_IND` etc.).
20. **Q: What is `checklistCopy`?** A: Y = preview/checklist copy from WIP collections; N = live. 
21. **Q: Is the API idempotent?** A: Read-only; effectively yes.
22. **Q: What is the inbound timeout?** A: None configured; Mongo maxTime 600 s in some handlers.
23. **Q: What external APIs does Template12 call?** A: einvoice print indicator, co-inward indicator, show-stax-inv indicator, comp-brn-number, SST rollout date (via premium debit note) - all through `PlineUtilsClient`/clients.
24. **Q: What happens if an external API is down?** A: Swallowed; blank/default values; request still 200. Mistake: expecting 502. Follow-up: how to detect? (logs "PlineUtilsClient GET failed").
25. **Q: SSRF protection?** A: fixed base URL + `assertTrustedHost` + `sanitize`.
26. **Q: What is `PlineReportNames.tag` for?** A: Stamps report/sub-report names in the response (commits e74bc0d5c, 861f0666c).
27. **Q: Why one `PlineReportData` in Template12 but one per row in RejectRisk?** A: Template12 mirrors the multi-subreport single-document shape, RejectRisk lists per row (each with own transDetList), plus a final ReportHeader entry; a disclaimer warns the report side must treat each entry as a print unit.
28. **Q: What is `PlineReportBlocks.reportHeader`?** A: The generic ReportHeader jrxml block with `dataClass` appended after data rows when rows exist.
29. **Q: How do you test the endpoint locally?** A: POST to `http://localhost:8080/gi/documentprintdata/api/opus/GenerateReport` with the sample JSON; ensure Mongo reachable.
30. **Q: How would you version the API?** A: Not versioned (URL has no version). Introduce `/v2` or new reportName; payload backwards-compatible because Maps are lenient.

## 5.C Thirty database questions

1. **Q: Which database does the service use?** A: MongoDB (Atlas-style replica set), DB name configured via `spring.mongodb.database`.
2. **Q: Why are collection names uppercase Oracle-like?** A: They mirror the Oracle tables/views the JRXML used; the data was replicated.
3. **Q: What does the `WIP_` prefix mean?** A: Work-in-progress (not yet posted) copy used for checklistCopy=Y.
4. **Q: What is `SB_` / `OCP_`?** A: Not documented in the code I read - say so; don't guess in an interview.
5. **Q: Primary collection for RejectRisk?** A: `CMUW_PL_REFER_RISK`.
6. **Q: How is the customer name obtained in RejectRisk?** A: `$lookup` into `CPGE_VI_PARTNERS_ALL` on (PART_ID, VERSION) = (CP_PART_ID, CP_PART_VERSION).
7. **Q: How are code descriptions fetched?** A: `$lookup` into `CMGE_CODE` filtered by `CAT_CODE` and `CODE_CD`.
8. **Q: Why `$arrayElemAt` instead of `$unwind`?** A: Keeps rows with no match (LEFT JOIN).
9. **Q: Why `$limit:1` in lookup pipelines?** A: Scalar sub-select semantics and early stop.
10. **Q: How is ROWNUM emulated?** A: `$setWindowFields` with `$documentNumber` sorted by customerName.
11. **Q: Why is the date upper bound +1 day?** A: Inclusive whole-day semantic without `$dateTrunc` so an index on CREATED_DATE is usable.
12. **Q: Why only add non-null date bounds?** A: null bound matches only null dates in BSON.
13. **Q: Are there foreign keys?** A: No (Mongo); joins are application-level via `$lookup`/sequential queries.
14. **Q: Are indexes defined in the repo?** A: No; `auto-index-creation` doesn't apply. They are managed in the cluster.
15. **Q: Which indexes would you recommend?** A: See 3.3 (CMGE_CODE {CAT_CODE, CODE_CD}, CONTRACT_ID+VERSION_NO, ENDT_NO/REF_NO, CREATED_DATE) - labelled recommendations.
16. **Q: What is `allowDiskUse`?** A: Lets blocking stages (sort, group) spill beyond 100 MB RAM.
17. **Q: What is `maxTime` here?** A: 600 s server-side limit, in some handlers.
18. **Q: Why numeric conversion of contractId?** A: Stored as numbers; `$eq` is type-strict - string would match nothing (`safeNum`).
19. **Q: What if the stored type is string?** A: `Long.parseLong` fallback returns raw string only when not numeric; a string-stored numeric value would not match - check data.
20. **Q: Live vs WIP in Template12?** A: Java chooses `isWip ? WIP_x : x`; live base query uses `VERSION_NO = 1`.
21. **Q: How does ASP decide product code?** A: `$ifNull(WIP contracts, SB contracts)`.
22. **Q: How is ENDT_CODE decided?** A: WIP versions if checklistCopy=Y else live versions by VERSION_NO.
23. **Q: What is the debit note master-policy chain?** A: MST_POL_NO from policy bases (live + WIP) -> `OCP_POLICY_BASES.POLICY_REF $in` -> mst CONTRACT_ID -> `PREM_BY_MST_IND`.
24. **Q: Why multiple queries rather than one aggregation?** A: Mirrors Oracle sub-selects, simpler/indexable; costs round trips.
25. **Q: Impact of `$lookup` without an index?** A: Per-row scan; slow at scale.
26. **Q: How do you diagnose a slow handler?** A: Enable `MongoTemplate` DEBUG to get the pipeline, run `explain("executionStats")` in Atlas/mongosh, check COLLSCAN, `totalDocsExamined`.
27. **Q: Why might a report be empty with no error?** A: Wrong DB name, type mismatch, WIP/live mix, or filters on wrong case; Mongo never errors on a 0-row match.
28. **Q: How are DTOs mapped?** A: `aggregate(..., Dto.class).getMappedResults()` with `$project` aliases matching DTO fields.
29. **Q: Mongo views vs collections for `*_VI_*`?** A: Unknown; views cannot have own indexes; verify.
30. **Q: What is the database risk of RejectRisk without a date filter?** A: Full scan + lookups for every row + in-memory sort; mitigate with mandatory date range.

---------------------------------------------------------------------------------------------------

# UNCERTAINTIES AND ITEMS TO VERIFY
1. How deployed pods receive Mongo URI/base URLs (Helm repo not available). The deployed DB name and whether the committed config is overridden are unknown.
2. Whether `API_BASE_URL` (underscore) env var resolves `${API_BASE-URL}` in this Boot 4.1.1 build (expected by relaxed binding; not tested).
3. Behaviour of an unknown/merged-away `reportName` (legacy switch default branch not fully read) and of null `reportName`/`metadata` (NPE inferred).
4. Meaning of `SB_` and `OCP_` collection prefixes; whether `*_VI_*` are Mongo views or materialised collections.
5. Date formats accepted by `plines/utils/DateUtils.toLocalDate` for `pCreatedDateFrom/To`.
6. Whether `PlineEinvoiceQrIndicatorClient` applies the host check and which property it reads.
7. Whether stale `plines.fn.date.value.constants.path` is correct (comment admits it is guessed).
8. Exhaustiveness of the ASP/Template12 collection list: some handlers (tax doc, transitional invoice, QR, ServiceTaxInvoice EndtMT) were not individually grepped.
9. `git log -S` introduction commits were traced for ~14 keys only; others are marked as not traced. Dates shown are commit dates in this repo's history (the history may include rebased/squashed imports).
10. Whether Mongo pool limits are set in the URI (redacted, not inspected for options).
11. The "dev" commit-date year (2026) comes from git history; no claim about calendar plausibility.
12. CVE identifiers are quoted from pom comments, not independently verified.
