# DHI / FIPS PR Testing — Repo Applicability

Cross-repo summary of the Docker Hardened Images (DHI) / FIPS-compliance migration PRs and whether the manual test scenario from the reference PR (`sa-tools-vc-gen#296`) applies to each other repo.

## What the initiative is doing

All 8 PRs are part of the same "Migrate to Docker Hardened Images" (STUD-104xx / STUD-107xx) initiative. Each PR migrates the service's final-stage container base image from a public / Amazon Linux / vanilla Corretto image to a Workiva Docker Hardened (DHI) FIPS variant, keeps `USER nobody`, and drops the historical Alpine/AL-era 99→65534 UID remap. Package managers and package updates are removed on the DHI final stage (they aren't required on DHI images), and any transitive shared-library dependencies get installed explicitly via `apt-get`.

## PR inventory

| Repo | PR | JIRA | Runtime | Base image change | Variant | Dockerfile in PR |
|---|---|---|---|---|---|---|
| [sa-tools-vc-gen](https://github.com/Workiva/sa-tools-vc-gen/pull/296) | 296 | STUD-10738 | .NET (ASP.NET Core 6.0) | `amazonlinux:2` → `dhi.io/debian-base:trixie-fips-dev` | `fips-dev` | `Dockerfile` |
| [sa-tools-validation](https://github.com/Workiva/sa-tools-validation/pull/262) | 262 | STUD-10751 | .NET (ASP.NET Core 6.0) | `amazonlinux:2` → `dhi.io/debian-base:trixie-fips-dev` | `fips-dev` | `Dockerfile` |
| [sa-tools-persistence](https://github.com/Workiva/sa-tools-persistence/pull/266) | 266 | STUD-10747 | .NET (ASP.NET Core 6.0) | `amazonlinux:2023` → `dhi.io/debian-base:trixie-fips-dev` | `fips-dev` | `Dockerfile` |
| [sa-tools-parsing](https://github.com/Workiva/sa-tools-parsing/pull/893) | 893 | STUD-10744 | .NET (ASP.NET Core 6.0) | `amazonlinux:2023` → `dhi.io/debian-base:trixie-fips-dev` | `fips-dev` | `Dockerfile` |
| [sa-tools-parsing-db](https://github.com/Workiva/sa-tools-parsing-db/pull/818) | 818 | STUD-10513 | Go | `busybox` → `dhi.io/busybox:1-debian-fips` | `fips` (non-dev) | `Dockerfile` |
| [sa-tools-changeset-service](https://github.com/Workiva/sa-tools-changeset-service/pull/2243) | 2243 | STUD-10511 | Java (Corretto 25) | `amazoncorretto:25` → `dhi.io/amazoncorretto:25-fips` | `fips` (non-dev) | `.github/docker/changeset-service.Dockerfile` |
| [publishing-service](https://github.com/Workiva/publishing-service/pull/1462) | 1462 | STUD-10492 | Java (Corretto 21) | `amazoncorretto:21` → `dhi.io/amazoncorretto:21-fips-dev` | `fips-dev` | `Dockerfile` |
| [graph-template-creation-service](https://github.com/Workiva/graph-template-creation-service/pull/1032) | 1032 | STUD-10453 | Java (Corretto 21) | `amazoncorretto:21` → `dhi.io/amazoncorretto:21-fips-dev` | `fips-dev` | `Dockerfile-Actions` |

**Reference PR:** [sa-tools-vc-gen#296](https://github.com/Workiva/sa-tools-vc-gen/pull/296). Only this PR's body contains the specific "Manual Tests" wording (`dotnet --info`, port 8080, `GET /s/sa-tools-vc-gen/api/health`, `USER nobody`, New Relic APM). The other 7 PR bodies carry only the generic "FIPS compliance restricts…" paragraph.

## Reference manual test scenario (verbatim, vc-gen)

> Built and ran the full image locally (matching CI's `amd64` target): confirmed `dotnet --info` reports the runtime correctly, and the real published app starts, binds port 8080, and returns `200 OK` from `GET /s/sa-tools-vc-gen/api/health` when run as `USER nobody`.
>
> This PR should be thoroughly tested before merging, especially anything involving the New Relic agent (APM data should still show up after the version bump) and TLS/crypto paths, since FIPS compliance restricts which cryptographic algorithms can be used at runtime (e.g. MD5, non-approved key sizes, certain TLS cipher suites) — some of these restrictions may not surface in CI and will only appear as runtime exceptions once deployed on the FIPS image.

The scenario decomposes into 7 independent checks:

| ID | Check |
|---|---|
| **T1** | Build + run image locally (matching CI's `amd64` target) |
| **T2** | `dotnet --info` reports the runtime correctly |
| **T3** | App binds port 8080 |
| **T4** | `GET /s/<service>/api/health` returns `200 OK` |
| **T5** | Runs as `USER nobody` |
| **T6** | New Relic APM data still flows after the agent bump |
| **T7** | TLS/crypto paths still work under FIPS (MD5, key sizes, cipher suites, hashing, signatures) |

## Applicability matrix

Legend: **Yes** = use as written. **Adapt** = substitute a service-specific value. **N/A** = the check does not apply to this stack.

| Repo · PR | Runtime | Variant | T1 | T2 `dotnet --info` | T3 :8080 | T4 `/api/health` | T5 `USER nobody` | T6 New Relic | T7 FIPS TLS/crypto |
|---|---|---|---|---|---|---|---|---|---|
| sa-tools-vc-gen · #296 | .NET | fips-dev | Yes | Yes | Yes | Yes | Yes | Yes | Yes |
| sa-tools-validation · #262 | .NET | fips-dev | Yes | Yes | Yes | Adapt | Yes | Yes | Yes |
| sa-tools-persistence · #266 | .NET | fips-dev | Yes | Yes | Yes | Adapt | Yes | Yes | Yes |
| sa-tools-parsing · #893 | .NET | fips-dev | Yes | Yes | Yes | Adapt | Yes | Yes | Yes |
| sa-tools-parsing-db · #818 | Go | fips | Yes | **N/A** | Yes | Adapt | Yes | **N/A** | Yes |
| sa-tools-changeset-service · #2243 | Java | fips | Yes | **N/A** | Yes | Adapt | Yes | **N/A** | Yes |
| publishing-service · #1462 | Java | fips-dev | Yes | **N/A** | **N/A** | **N/A** | Yes | **N/A** | Yes |
| graph-template-creation-service · #1032 | Java | fips-dev | Yes | **N/A** | **N/A** | **N/A** | Yes | **N/A** | Yes |

### One-line takeaways

- **T1 / T5** apply universally.
- **T2 / T6 (dotnet, New Relic .NET)** apply only to the four .NET PRs. Java repos use OpenTelemetry, Go has neither.
- **T3 (port 8080)** applies to all except `publishing-service` and `graph-template-creation-service` (they use JanusGraph/Gremlin ports `8123 / 8182 / 8345 / 5005 / 9898 / 10001` and don't touch 8080).
- **T4 (HTTP `/api/health`)** — the exact path is service-specific for all four .NET repos and for `changeset-service`, and the entire check is not applicable for `parsing-db`, `publishing-service`, or `graph-template-creation-service`.
- **T7 (FIPS TLS/crypto)** applies to all — this is the whole point of the initiative. What "TLS/crypto paths" means differs per repo (see below).

## Per-repo details

### sa-tools-vc-gen · #296 (reference)

- Base image: `amazonlinux:2` → `dhi.io/debian-base:trixie-fips-dev`
- Runtime: ASP.NET Core 6.0
- Entrypoint: `dotnet VertexCacheGeneratorAPI.dll`
- Port: 8080
- Health path: `/s/sa-tools-vc-gen/api/health` (readinessProbe, no liveness)
- APM: New Relic .NET agent  9.2.0.0 → 10.52.0 (install path change: `/usr/local/newrelic-netcore20-agent` → `/usr/local/newrelic-dotnet-agent`; `workiva.yml` env vars updated)

Local test:

```bash
docker build --platform linux/amd64 -t vc-gen:dhi \
  --build-arg ARTIFACTORY_PRO_USER=$U --build-arg ARTIFACTORY_PRO_PASS=$P .
docker run --rm -p 8080:8080 vc-gen:dhi
curl -sS -o /dev/null -w '%{http_code}\n' http://localhost:8080/s/sa-tools-vc-gen/api/health
# expect: 200
```

Test scenario applies as written.

---

### sa-tools-validation · #262

- Base image: `amazonlinux:2` → `dhi.io/debian-base:trixie-fips-dev`
- Runtime: ASP.NET Core 6.0
- Entrypoint: `dotnet ValidationAPI.dll`
- Port: 8080
- **Health path: `/s/sa-tools-validation/api/Validation`** (not `/api/health`)
- APM: New Relic .NET  9.2.0.0 → 10.52.0
- Also modifies `workiva.yml` — `CORECLR_NEWRELIC_HOME` / `CORECLR_PROFILER_PATH`

```bash
docker build --platform linux/amd64 -t sa-tools-validation:dhi \
  --build-arg ARTIFACTORY_PRO_USER=$U --build-arg ARTIFACTORY_PRO_PASS=$P .
docker run --rm -p 8080:8080 sa-tools-validation:dhi
curl -sS -o /dev/null -w '%{http_code}\n' http://localhost:8080/s/sa-tools-validation/api/Validation
```

Only substitution: the health path.

---

### sa-tools-persistence · #266

- Base image: `amazonlinux:2023` → `dhi.io/debian-base:trixie-fips-dev`
- Runtime: ASP.NET Core 6.0
- Entrypoint: `dotnet PersistenceAPI.dll`
- Port: 8080 (via `ASPNETCORE_URLS=http://*:8080`)
- **Health path: `/s/sa-tools-persistence/api/Mapping`**
- APM: New Relic .NET  9.2.0.0 → 10.52.0
- Also modifies `workiva.yml`
- Talks to DynamoDB — AWS SDK TLS is a FIPS concern

```bash
docker build --platform linux/amd64 -t sa-tools-persistence:dhi \
  --build-arg ARTIFACTORY_PRO_USER=$U --build-arg ARTIFACTORY_PRO_PASS=$P .
docker run --rm -p 8080:8080 sa-tools-persistence:dhi
curl -sS -o /dev/null -w '%{http_code}\n' http://localhost:8080/s/sa-tools-persistence/api/Mapping
```

---

### sa-tools-parsing · #893

- Base image: `amazonlinux:2023` → `dhi.io/debian-base:trixie-fips-dev`
- Runtime: ASP.NET Core 6.0
- Entrypoint: `dotnet ParsingAPI.dll`
- Port: 8080
- **Health path: `/s/sa-tools-parsing/api/parsing`**
- APM: New Relic .NET  9.2.0.0 → 10.52.0
- `workiva.yml` already used `/usr/local/newrelic-dotnet-agent` — this PR is Dockerfile-only
- Talks to S3 (SA_TOOLS_PARSING_BUCKET_NAME) — AWS SDK TLS on FIPS

```bash
docker build --platform linux/amd64 -t sa-tools-parsing:dhi \
  --build-arg ARTIFACTORY_PRO_USER=$U --build-arg ARTIFACTORY_PRO_PASS=$P .
docker run --rm -p 8080:8080 sa-tools-parsing:dhi
curl -sS -o /dev/null -w '%{http_code}\n' http://localhost:8080/s/sa-tools-parsing/api/parsing
```

---

### sa-tools-parsing-db · #818 (Go)

- Base image: `busybox` → `dhi.io/busybox:1-debian-fips` (**non-dev — no shell, no package manager inside**)
- Runtime: Go binary (`/app/sa-tools-parsing-db`)
- Port: 8080
- Health path: **no HTTP probe defined** in `workiva.yml`; the Frugal endpoint at `/frugal` is the closest thing
- APM: none (Go service)
- PR sets `ENV GODEBUG=fips140=on` — activates Go's FIPS module at runtime
- PR uses `COPY --chown=nobody:nobody` instead of `RUN chown` (no shell available)
- Removes copying `ca-certificates.crt` (DHI busybox already includes CA certs)

Applicability changes:

- T2 (`dotnet --info`) — **N/A**. Substitute: `docker inspect` for the image, or `docker run --rm sa-tools-parsing-db:dhi --version` if the binary exposes a version flag.
- T6 (New Relic) — **N/A**. Go service has no New Relic .NET agent.
- T7 (FIPS TLS/crypto) — **most critical step for this PR**. `GODEBUG=fips140=on` switches the Go crypto/tls stack to the FIPS module; test the MySQL/RDS TLS handshake (`rds-combined-ca-bundle.pem`), Frugal RPC, and any custom cipher-suite/min-version config.
- Debugging note: **no shell inside** — you cannot `docker exec sh`. Use `docker logs` and external `curl` only. For runtime shell in k8s, `kubectl debug` gets you an ephemeral debug container.

```bash
docker build --platform linux/amd64 -t sa-tools-parsing-db:dhi \
  --build-arg GIT_SSH_KEY="$(cat ~/.ssh/id_rsa)" \
  --build-arg KNOWN_HOSTS_CONTENT="$(cat ~/.ssh/known_hosts)" .
docker run --rm -p 8080:8080 \
  -e DATABASE_HOST=... -e DATABASE_USERNAME=... -e DATABASE_PASSWORD=... \
  sa-tools-parsing-db:dhi
curl -sS -o /dev/null -w '%{http_code}\n' http://localhost:8080/frugal
```

---

### sa-tools-changeset-service · #2243 (Java)

- Base image: `amazoncorretto:25` → `dhi.io/amazoncorretto:25-fips` (**non-dev — no shell**)
- Runtime: Java (Corretto 25)
- Entrypoint: `java -javaagent:opentelemetry-javaagent.jar -jar ChangesetService.jar`
- Port: 8080
- **Health paths: `/s/sa-tools-changeset/_wk/alive` (liveness) + `/s/sa-tools-changeset/_wk/ready` (readiness)** — HTTP, not exec probes
- APM: **OpenTelemetry** (not New Relic)
- Dockerfile in the PR is `.github/docker/changeset-service.Dockerfile` — build with `-f`

Applicability changes:

- T2 (`dotnet --info`) — N/A. Substitute: `docker run --rm --entrypoint /usr/bin/java changeset-service:dhi -version`.
- T6 (New Relic) — N/A. Verify OpenTelemetry span data still flows to the collector after the base-image switch.
- T7 (FIPS) — verify the Corretto FIPS provider is active and any custom TLS config is FIPS-compatible.

```bash
docker build --platform linux/amd64 \
  -f .github/docker/changeset-service.Dockerfile \
  -t changeset-service:dhi \
  --build-arg ARTIFACTORY_PRO_USER=$U --build-arg ARTIFACTORY_PRO_PASS=$P .
docker run --rm -p 8080:8080 changeset-service:dhi
curl -sS -o /dev/null -w '%{http_code}\n' http://localhost:8080/s/sa-tools-changeset/_wk/alive
```

---

### publishing-service · #1462 (Java, Gremlin/JanusGraph service)

- Base image: `amazoncorretto:21` → `dhi.io/amazoncorretto:21-fips-dev`
- Runtime: Java (Corretto 21)
- Entrypoint: `/usr/local/bin/run_service.sh` (bash — requires `-fips-dev`)
- **Ports: `8123, 8182, 8345, 5005, 9898, 10001` — does not bind 8080**
- **Health checks: `exec` probes on `liveness_check.sh` / `readiness_check.sh`** — the scripts read `/tmp/liveness_status` and `/tmp/readiness_status` files that the JVM writes. **No HTTP `/api/health` endpoint.**
- APM: OpenTelemetry
- Uses `keytool -importcert` to build `graph-key-store` from `rds-combined-ca-bundle.pem`

Applicability changes:

- T2 — N/A (Java). Substitute: `java -version`.
- T3 — N/A (no 8080). Substitute: verify Gremlin listens on `8182` and metrics on `9898`.
- T4 — N/A as written. Substitute the entire test with:

  ```bash
  # wait for the JVM to start and write /tmp/*_status files
  docker exec <container> /usr/local/bin/liveness_check.sh; echo "exit=$?"
  docker exec <container> /usr/local/bin/readiness_check.sh; echo "exit=$?"
  # both must return 0
  ```
- T6 — N/A. OpenTelemetry, not New Relic.
- T7 — critical. Verify the FIPS JDK accepts the RDS cert import (keystore type compatibility) and Gremlin/JanusGraph TLS to RDS still works.

```bash
docker build --platform linux/amd64 -t publishing-service:dhi .
docker run --rm -p 8182:8182 -p 8123:8123 -p 8345:8345 publishing-service:dhi
```

---

### graph-template-creation-service · #1032 (Java)

- Base image: `amazoncorretto:21` → `dhi.io/amazoncorretto:21-fips-dev`
- Runtime: Java (Corretto 21)
- Entrypoint: `/usr/local/bin/run_service.sh` (bash — requires `-fips-dev`)
- **Ports: `8123, 8182, 8345, 5005, 9898, 10001` — does not bind 8080**
- **Health checks: `exec` probes on `liveness_check.sh` / `readiness_check.sh` — same file-based pattern as publishing-service.** Also ships the `Workiva/no-auth-health-check` binary and a helm-compat `healthcheck.sh` symlink to `readiness_check.sh`.
- APM: OpenTelemetry
- **Only `Dockerfile-Actions` changes in this PR — build with `-f Dockerfile-Actions`**. The default `Dockerfile` is unchanged.

Applicability changes:

- Same as publishing-service. T2, T3, T4, T6 do not apply as written; T1, T5, T7 apply.
- Extra check: verify the `no-auth-health-check` binary (glibc-compat) still runs on the DHI Debian base.

```bash
docker build --platform linux/amd64 -f Dockerfile-Actions -t gtcs:dhi .
docker run --rm -p 8182:8182 -p 8123:8123 -p 8345:8345 gtcs:dhi
docker exec <container> /usr/local/bin/liveness_check.sh; echo "exit=$?"
docker exec <container> /usr/local/bin/readiness_check.sh; echo "exit=$?"
```

## Test scenarios per repo

Each repo has its own checklist below. Scenarios are grouped by phase:

- **B — Build & image sanity** (Docker build, image size, security scans)
- **R — Runtime & startup** (process starts, binds ports, runs as nobody)
- **F — FIPS TLS/crypto** (the actual FIPS compliance verification — the whole reason for this initiative)
- **O — Observability** (New Relic .NET agent for .NET repos; OpenTelemetry agent for Java repos; not applicable to Go)
- **X — Functional smoke** (service-specific business paths — verify no regression from the base-image change)
- **D — Deployment** (Aviary deploy to wk-dev / sandbox and let real K8s probes exercise the image)

Priority: **P0** = must pass before merge · **P1** = should pass · **P2** = nice to have. `<img>` = the tag you built.

### sa-tools-vc-gen · #296 (reference, .NET)

| ID | Cat | Scenario | How to verify | Expected | Prio |
|---|---|---|---|---|---|
| B1 | Build | `docker build` on new base image succeeds for `linux/amd64` | `docker build --platform linux/amd64 -t vc-gen:dhi --build-arg ARTIFACTORY_PRO_USER=$U --build-arg ARTIFACTORY_PRO_PASS=$P .` | exit 0 | P0 |
| B2 | Build | `docker build --check` reports no new warnings | `docker build --check --platform linux/amd64 .` | only pre-existing warnings | P0 |
| B3 | Build | Image size within +20% of the AL2 baseline | `docker images vc-gen:dhi` vs the pre-migration image | acceptable delta | P1 |
| B4 | Build | `gha-security check_images` passes in CI | wait for CI job on PR | pass | P0 |
| B5 | Build | Trivy/Snyk shows no new HIGH/CRITICAL CVEs | `trivy image vc-gen:dhi` | no regressions | P0 |
| R1 | Runtime | Container starts without stack traces | `docker run --rm -p 8080:8080 vc-gen:dhi` then read logs | app up | P0 |
| R2 | Runtime | ASP.NET Core 6.0.36 runtime installed | `docker run --rm --entrypoint dotnet vc-gen:dhi --info` | reports 6.0.36 | P0 |
| R3 | Runtime | Native CoreCLR deps present | `docker run --rm --entrypoint /bin/sh vc-gen:dhi -c 'dpkg -l libstdc++6 libicu76'` | both installed | P0 |
| R4 | Runtime | App binds `0.0.0.0:8080` | `curl -sI http://localhost:8080/s/sa-tools-vc-gen/api/health` | HTTP 200 | P0 |
| R5 | Runtime | Final process runs as `nobody` (uid 65534) | `docker exec <c> id` | `uid=65534 gid=65534` | P0 |
| F1 | FIPS | No MD5 hashing failures at runtime | grep code for `MD5`; exercise any hashing paths | no runtime exceptions | P0 |
| F2 | FIPS | Outbound HTTPS to Workiva services succeeds | with real env, hit any downstream Workiva service | TLS OK | P0 |
| F3 | FIPS | TLS cipher suites negotiated are FIPS-approved | capture handshake or check server logs | approved suite only | P1 |
| O1 | APM | New Relic .NET agent 10.52.0 loads | `docker run` with `NEW_RELIC_LICENSE_KEY` set; check logs for `NewRelic Application Monitoring starting` | agent init line present | P0 |
| O2 | APM | `CORECLR_PROFILER_PATH` points to a file that exists | `docker exec <c> test -f /usr/local/newrelic-dotnet-agent/libNewRelicProfiler.so && echo OK` | `OK` | P0 |
| O3 | APM | `workiva.yml` `CORECLR_*` env vars pass through to the pod | `kubectl exec <pod> -- env | grep CORECLR_` | new paths reflected | P0 |
| O4 | APM | New Relic APM UI shows post-bump data | hit endpoints in wk-dev, wait 5 min, check NR dashboard | traces/transactions appear | P0 |
| X1 | Func | Vertex cache generation endpoint operates end-to-end | integration test against wk-dev | 2xx + expected payload | P0 |
| X2 | Func | Message health topic publishes (`MSG_HEALTH_TOPIC`) | check messaging bus for topic on `metadata.name` | topic active | P1 |
| D1 | Deploy | Aviary deploy to wk-dev succeeds | Aviary pipeline | green | P0 |
| D2 | Deploy | K8s readinessProbe on `/api/health` passes | `kubectl describe pod` | Ready | P0 |
| D3 | Deploy | Rolling update completes without downtime | trigger deploy, observe | zero 5xx during rollout | P1 |

### sa-tools-validation · #262 (.NET)

| ID | Cat | Scenario | How to verify | Expected | Prio |
|---|---|---|---|---|---|
| B1 | Build | `docker build` succeeds on `linux/amd64` | `docker build --platform linux/amd64 -t sa-tools-validation:dhi --build-arg ARTIFACTORY_PRO_USER=$U --build-arg ARTIFACTORY_PRO_PASS=$P .` | exit 0 | P0 |
| B2 | Build | `docker build --check` clean | `docker build --check ...` | no new warnings | P0 |
| B3 | Build | `gha-security check_images` passes | CI | pass | P0 |
| B4 | Build | No new HIGH/CRITICAL CVEs | `trivy image` | clean | P0 |
| R1 | Runtime | Container starts | `docker run --rm -p 8080:8080 sa-tools-validation:dhi` | app up | P0 |
| R2 | Runtime | `dotnet --info` reports 6.0.36 | `docker run --rm --entrypoint dotnet <img> --info` | 6.0.36 | P0 |
| R3 | Runtime | libstdc++6 + libicu76 installed | `dpkg -l libstdc++6 libicu76` | present | P0 |
| R4 | Runtime | App binds `:8080` and `/s/sa-tools-validation/api/Validation` returns 200 | `curl -sI localhost:8080/s/sa-tools-validation/api/Validation` | HTTP 200 | P0 |
| R5 | Runtime | Runs as `nobody` (uid 65534) | `docker exec <c> id` | uid=65534 | P0 |
| F1 | FIPS | Validation service crypto paths OK on FIPS | run a validation flow that exercises signing/hashing (if any) | no runtime exceptions | P0 |
| F2 | FIPS | TLS to any downstream call the service makes | exercise endpoints in wk-dev | handshake OK | P0 |
| O1 | APM | New Relic .NET agent 10.52.0 loads | check logs | agent init present | P0 |
| O2 | APM | `CORECLR_PROFILER_PATH` valid after `workiva.yml` env change | `test -f $CORECLR_PROFILER_PATH` | OK | P0 |
| O3 | APM | New Relic sees APM data | hit endpoints, check NR | traces appear | P0 |
| X1 | Func | Validation API happy-path (`POST /api/Validation`) works | wk-dev smoke test | 2xx + expected result | P0 |
| X2 | Func | HARBOUR_SERVICE_PREFIX routing works via ingress | request through ingress host | correct routing | P1 |
| D1 | Deploy | Aviary deploy to wk-dev succeeds | pipeline | green | P0 |
| D2 | Deploy | Readiness probe on `/api/Validation` passes | pod events | Ready | P0 |

### sa-tools-persistence · #266 (.NET; DynamoDB + JWT)

| ID | Cat | Scenario | How to verify | Expected | Prio |
|---|---|---|---|---|---|
| B1 | Build | `docker build` succeeds | `docker build --platform linux/amd64 -t sa-tools-persistence:dhi ...` | exit 0 | P0 |
| B2 | Build | `docker build --check` clean | as above | no new warnings | P0 |
| B3 | Build | gha-security + CVE scan clean | CI | pass | P0 |
| R1 | Runtime | Container starts | `docker run --rm -p 8080:8080 <img>` | app up | P0 |
| R2 | Runtime | `dotnet --info` reports 6.0.36 | as above | 6.0.36 | P0 |
| R3 | Runtime | Native deps present | `dpkg -l libstdc++6 libicu76` | installed | P0 |
| R4 | Runtime | App binds `:8080`; `/s/sa-tools-persistence/api/Mapping` returns 200 | curl | HTTP 200 | P0 |
| R5 | Runtime | Runs as `nobody` | `id` | uid=65534 | P0 |
| F1 | FIPS | AWS SDK TLS to DynamoDB works | write + read a test item from persistence table | 2xx | P0 |
| F2 | FIPS | AWS SDK TLS to DynamoDB rollforward table works | write + read from rollforward table | 2xx | P0 |
| F3 | FIPS | JWT signature verification passes under FIPS | request with valid JWT | authorized | P0 |
| F4 | FIPS | JWT with a non-FIPS-approved algorithm is rejected as expected | send HS-MD5 or similar | 401 (not internal 500) | P1 |
| O1 | APM | New Relic .NET agent 10.52.0 loads | logs | init line present | P0 |
| O2 | APM | `CORECLR_*` env vars from `workiva.yml` reach pod | `kubectl exec env | grep CORECLR_` | new paths | P0 |
| O3 | APM | New Relic sees APM after bump | wk-dev + NR | data appears | P0 |
| X1 | Func | Create Mapping (CustomerID, MappingID) via API | end-to-end call | persisted in DynamoDB | P0 |
| X2 | Func | Read Mapping by (CustomerID, MappingID) | end-to-end call | correct payload | P0 |
| X3 | Func | Rollforward operation writes to rollforward table | trigger flow | correct write | P0 |
| D1 | Deploy | Deploy to wk-dev succeeds | Aviary | green | P0 |
| D2 | Deploy | IAM role assumed correctly for DynamoDB | pod logs / AWS CloudTrail | AssumeRole OK | P0 |
| D3 | Deploy | Readiness probe on `/api/Mapping` passes | pod events | Ready | P0 |

### sa-tools-parsing · #893 (.NET; S3 + Cerberus)

| ID | Cat | Scenario | How to verify | Expected | Prio |
|---|---|---|---|---|---|
| B1 | Build | `docker build` succeeds | `docker build --platform linux/amd64 -t sa-tools-parsing:dhi ...` | exit 0 | P0 |
| B2 | Build | `docker build --check` clean; gha-security passes | CI | pass | P0 |
| B3 | Build | No new HIGH/CRITICAL CVEs | `trivy` | clean | P0 |
| R1 | Runtime | Container starts | `docker run --rm -p 8080:8080 <img>` | app up | P0 |
| R2 | Runtime | `dotnet --info` reports 6.0.36 | as above | 6.0.36 | P0 |
| R3 | Runtime | Native CoreCLR deps present | `dpkg -l libstdc++6 libicu76` | installed | P0 |
| R4 | Runtime | App binds `:8080`; `/s/sa-tools-parsing/api/parsing` returns 200 | curl | HTTP 200 | P0 |
| R5 | Runtime | Runs as `nobody` | `id` | uid=65534 | P0 |
| F1 | FIPS | AWS SDK TLS to S3 (`SA_TOOLS_PARSING_BUCKET_NAME`) works | put + get an object | 2xx | P0 |
| F2 | FIPS | TLS to Cerberus (`CERBERUS_BASE_URL`) works | exercise a call | success | P0 |
| F3 | FIPS | No MD5 in ETag comparison / cache-key paths | grep + exercise | no failures | P1 |
| O1 | APM | New Relic .NET agent 10.52.0 loads | logs | init line present | P0 |
| O2 | APM | `workiva.yml` already had `/usr/local/newrelic-dotnet-agent`; verify env matches in pod | `kubectl exec env` | consistent | P0 |
| O3 | APM | New Relic sees APM after bump | NR dashboard | data appears | P0 |
| X1 | Func | Parsing API happy path (`POST /api/parsing`) | wk-dev smoke test | expected parsed output | P0 |
| X2 | Func | S3 read/write on bucket | end-to-end | object round-trips | P0 |
| D1 | Deploy | Aviary deploy to wk-dev succeeds | pipeline | green | P0 |
| D2 | Deploy | Readiness probe on `/api/parsing` passes | pod events | Ready | P0 |

### sa-tools-parsing-db · #818 (Go; **non-dev fips, no shell**)

| ID | Cat | Scenario | How to verify | Expected | Prio |
|---|---|---|---|---|---|
| B1 | Build | `docker build` succeeds on `linux/amd64` | `docker build --platform linux/amd64 -t sa-tools-parsing-db:dhi --build-arg GIT_SSH_KEY="$(cat ~/.ssh/id_rsa)" --build-arg KNOWN_HOSTS_CONTENT="$(cat ~/.ssh/known_hosts)" .` | exit 0 | P0 |
| B2 | Build | `docker build --check` clean | as above | no new warnings | P0 |
| B3 | Build | gha-security passes; CVE scan clean | CI + Trivy | pass | P0 |
| R1 | Runtime | Non-dev image has no shell — `docker exec sh` fails | `docker run --rm --entrypoint /bin/sh <img>` | error / no such file | P0 |
| R2 | Runtime | Container starts and binary runs | `docker run --rm -p 8080:8080 -e DATABASE_HOST=... -e DATABASE_USERNAME=... -e DATABASE_PASSWORD=... <img>` | binary up | P0 |
| R3 | Runtime | Runs as `nobody` (uid 65534) | `docker inspect --format '{{.Config.User}}' <img>` | `nobody` | P0 |
| R4 | Runtime | `GODEBUG=fips140=on` is set in the image env | `docker inspect --format '{{range .Config.Env}}{{println .}}{{end}}' <img> | grep GODEBUG` | `GODEBUG=fips140=on` | P0 |
| R5 | Runtime | DHI busybox provides CA certs (PR removed the COPY of ca-certificates.crt) | outbound HTTPS from container works | TLS OK | P0 |
| F1 | FIPS | Go crypto/tls handshake to RDS MySQL via `rds-combined-ca-bundle.pem` | connect to RDS | connection OK | P0 |
| F2 | FIPS | Migrations apply cleanly on FIPS-enabled Go | run migration step | applied | P0 |
| F3 | FIPS | Any custom TLS config (min version, cipher suites) still valid under Go FIPS | run integration tests | pass | P0 |
| F4 | FIPS | Frugal RPC over TLS (if used) handshakes | Frugal client test | OK | P1 |
| O1 | APM | Service starts writing metrics to Datadog / Prometheus (whichever) | scrape metrics endpoint / DD | metrics visible | P1 |
| X1 | Func | `/frugal` endpoint responds | Frugal client call, or `curl http://localhost:8080/frugal` | valid frame / 200 | P0 |
| X2 | Func | Read + write against the Parsing DB (CustomerID keyed) | end-to-end DB call | round-trip OK | P0 |
| X3 | Func | Multi-region setups still work (repo has a `prod-ca` region added) | deploy in prod-ca | reachable | P1 |
| D1 | Deploy | Aviary deploy to wk-dev succeeds | pipeline | green | P0 |
| D2 | Deploy | RDS connection via secret (`DATABASE_PASSWORD`, `PARSING_DB_ADMIN_PASSWORD`) works | pod logs | connected | P0 |
| D3 | Debug | `kubectl debug` gives an ephemeral debug container (fallback for no-shell image) | `kubectl debug -it <pod> --image=busybox --target=<c>` | shell attached | P1 |

### sa-tools-changeset-service · #2243 (Java; **non-dev fips, no shell**)

| ID | Cat | Scenario | How to verify | Expected | Prio |
|---|---|---|---|---|---|
| B1 | Build | `docker build -f .github/docker/changeset-service.Dockerfile` succeeds | `docker build --platform linux/amd64 -f .github/docker/changeset-service.Dockerfile -t changeset-service:dhi --build-arg ARTIFACTORY_PRO_USER=$U --build-arg ARTIFACTORY_PRO_PASS=$P .` | exit 0 | P0 |
| B2 | Build | `docker build --check` clean (pre-existing warnings only) | as above | no new warnings | P0 |
| B3 | Build | gha-security passes; CVE scan clean | CI + Trivy | pass | P0 |
| R1 | Runtime | Non-dev image has no shell | `docker run --rm --entrypoint /bin/sh <img>` | fails | P0 |
| R2 | Runtime | Container starts | `docker run --rm -p 8080:8080 <img>` | JVM up | P0 |
| R3 | Runtime | `java -version` reports Corretto 25 | `docker run --rm --entrypoint /usr/bin/java <img> -version` | Corretto 25 | P0 |
| R4 | Runtime | Runs as `nobody` (uid 65534) | `docker inspect --format '{{.Config.User}}' <img>` | `nobody` | P0 |
| R5 | Runtime | `-Xms16G -Xmx16G` heap allocates on target host | run with 16Gi memory limit | JVM starts | P0 |
| R6 | Runtime | HTTP livenessProbe `/s/sa-tools-changeset/_wk/alive` returns 200 | `curl -sI localhost:8080/s/sa-tools-changeset/_wk/alive` | HTTP 200 | P0 |
| R7 | Runtime | HTTP readinessProbe `/s/sa-tools-changeset/_wk/ready` returns 200 | `curl -sI localhost:8080/s/sa-tools-changeset/_wk/ready` | HTTP 200 | P0 |
| F1 | FIPS | Corretto 25 FIPS provider active | `docker run --rm --entrypoint /usr/bin/java <img> -XshowSettings:security -version 2>&1 | grep -i fips` | FIPS provider listed | P0 |
| F2 | FIPS | TLS to database (`DATABASE_SERVICE`) succeeds under FIPS | exercise a DB call | connected | P0 |
| F3 | FIPS | OAuth2 TLS to `OAUTH2_HOST` succeeds | authenticate | 2xx | P0 |
| F4 | FIPS | TLS to `VESSEL_URL` succeeds | integration call | OK | P0 |
| O1 | OTel | OpenTelemetry agent attaches | log line `otel.javaagent` on startup | present | P0 |
| O2 | OTel | Traces reach the collector | after hitting endpoints, check OTel backend / Datadog | traces present | P0 |
| X1 | Func | Changeset service Thrift/RPC endpoints respond | RPC smoke test | expected response | P0 |
| X2 | Func | `CREATE_WORKIVA_PERSONS` init routine works | exercise startup config | expected behavior | P1 |
| X3 | Func | `WRITE_ABILITIES` gating applied | exercise a write | authorized/denied as expected | P1 |
| D1 | Deploy | Aviary deploy to wk-dev succeeds | pipeline | green | P0 |
| D2 | Deploy | HTTP liveness + readiness probes pass in K8s | pod events | Ready | P0 |
| D3 | Debug | `kubectl debug` provides ephemeral shell (fallback) | `kubectl debug ...` | shell attached | P1 |

### publishing-service · #1462 (Java; Gremlin/JanusGraph; file-based probes)

| ID | Cat | Scenario | How to verify | Expected | Prio |
|---|---|---|---|---|---|
| B1 | Build | `docker build` succeeds | `docker build --platform linux/amd64 -t publishing-service:dhi .` | exit 0 | P0 |
| B2 | Build | `docker build --check` clean | as above | no new warnings | P0 |
| B3 | Build | gha-security passes; CVE scan clean | CI + Trivy | pass | P0 |
| B4 | Build | `ADD` of RDS cert bundle succeeds (replaces the removed `curl`) | image build logs | cert file present | P0 |
| R1 | Runtime | Container starts (needs bash → fips-dev variant confirmed) | `docker run --rm -p 8182:8182 -p 8123:8123 -p 8345:8345 <img>` | JVM up | P0 |
| R2 | Runtime | `run_service.sh` bash script executes end to end | container logs | no `[[: not found` errors | P0 |
| R3 | Runtime | `java -version` reports Corretto 21 | `docker run --rm --entrypoint /usr/bin/java <img> -version` | Corretto 21 | P0 |
| R4 | Runtime | Runs as `nobody` (uid 65534) | `docker inspect --format '{{.Config.User}}' <img>` | `nobody` | P0 |
| R5 | Runtime | Debian `procps` package present (replaces AL `procps-ng`) | `docker exec <c> which ps` | resolves | P0 |
| R6 | Runtime | Gremlin server binds `:8182` | `curl -sI localhost:8182/` | connection accepted | P0 |
| R7 | Runtime | Liveness `.sh` script returns 0 once `/tmp/liveness_status` is written | wait for JVM to write, then `docker exec <c> /usr/local/bin/liveness_check.sh; echo $?` | exit 0 | P0 |
| R8 | Runtime | Readiness `.sh` script returns 0 once `/tmp/readiness_status` is written | as above | exit 0 | P0 |
| F1 | FIPS | Corretto 21 FIPS provider active | `java -XshowSettings:security -version` | FIPS provider listed | P0 |
| F2 | FIPS | `keytool -importcert` of RDS cert bundle into `graph-key-store` succeeds on FIPS JDK | image build logs | keystore created | P0 |
| F3 | FIPS | JanusGraph/Gremlin TLS to RDS handshakes | production-mode smoke test | OK | P0 |
| F4 | FIPS | Gremlin ports accept TLS clients with FIPS-approved ciphers | Gremlin client with restricted cipher suites | connect OK | P1 |
| O1 | OTel | OpenTelemetry javaagent attaches | log line on startup | present | P0 |
| O2 | OTel | Traces / metrics reach collector | exercise flow, check backend | present | P0 |
| X1 | Func | Publishing API happy path | wk-dev smoke test | expected result | P0 |
| X2 | Func | RDS-backed operations succeed after keystore rebuild | exercise DB write | OK | P0 |
| D1 | Deploy | Aviary deploy to wk-dev succeeds | pipeline | green | P0 |
| D2 | Deploy | K8s `exec` liveness + readiness probes pass | pod events | Ready | P0 |
| D3 | Deploy | `securityContext` uses UID 65534 (matches image) | `kubectl get pod -o yaml` | `runAsUser: 65534` | P0 |
| D4 | Deploy | Rolling restart works despite bash-based entrypoint | trigger deploy | no downtime | P1 |

### graph-template-creation-service · #1032 (Java; Gremlin; **PR only touches `Dockerfile-Actions`**)

| ID | Cat | Scenario | How to verify | Expected | Prio |
|---|---|---|---|---|---|
| B1 | Build | `docker build -f Dockerfile-Actions` succeeds | `docker build --platform linux/amd64 -f Dockerfile-Actions -t gtcs:dhi .` | exit 0 | P0 |
| B2 | Build | `docker build --check` clean | as above | no new warnings | P0 |
| B3 | Build | gha-security passes; CVE scan clean | CI + Trivy | pass | P0 |
| B4 | Build | `Workiva/no-auth-health-check` binary copied over correctly | image contains `/usr/local/bin/health-check` | present + `+x` | P0 |
| B5 | Build | Verify the plain `Dockerfile` was intentionally left un-migrated | inspect PR | only `Dockerfile-Actions` changed | P1 |
| R1 | Runtime | Container starts (needs bash → fips-dev variant confirmed) | `docker run --rm -p 8182:8182 -p 8123:8123 -p 8345:8345 <img>` | JVM up | P0 |
| R2 | Runtime | `run_service.sh` executes end to end | container logs | no bash errors | P0 |
| R3 | Runtime | `java -version` reports Corretto 21 | `docker run --rm --entrypoint /usr/bin/java <img> -version` | Corretto 21 | P0 |
| R4 | Runtime | Runs as `nobody` (uid 65534) | `docker inspect --format '{{.Config.User}}' <img>` | `nobody` | P0 |
| R5 | Runtime | Debian `procps` + `tar` + `util-linux` installed (was AL `yum` names) | `docker exec <c> which ps tar` | both resolve | P0 |
| R6 | Runtime | `no-auth-health-check` binary runs on Debian FIPS base | `docker exec <c> /usr/local/bin/health-check --help` | help printed | P0 |
| R7 | Runtime | Gremlin ports respond | `curl -sI localhost:8182/` | connection accepted | P0 |
| R8 | Runtime | Liveness `.sh` returns 0 once `/tmp/liveness_status` written | wait, then `docker exec <c> /usr/local/bin/liveness_check.sh; echo $?` | exit 0 | P0 |
| R9 | Runtime | Readiness `.sh` (uses `health-check` internally) returns 0 | wait, then `docker exec <c> /usr/local/bin/readiness_check.sh; echo $?` | exit 0 | P0 |
| F1 | FIPS | Corretto 21 FIPS provider active | `java -XshowSettings:security` | FIPS provider listed | P0 |
| F2 | FIPS | `keytool -importcert` into `graph-key-store` succeeds on FIPS JDK | build logs | keystore created | P0 |
| F3 | FIPS | Gremlin TLS to RDS handshakes | wk-dev smoke test | connected | P0 |
| O1 | OTel | OpenTelemetry javaagent attaches | startup log | present | P0 |
| O2 | OTel | OTel traces reach collector | exercise API, check backend | present | P0 |
| X1 | Func | Template CRUD API works | wk-dev smoke test | 2xx | P0 |
| X2 | Func | NATS instance topic health check via `health-check --topic-files=...` | exercise the readiness script's NATS check | exit 0 | P0 |
| D1 | Deploy | Aviary deploy to wk-dev succeeds | pipeline | green | P0 |
| D2 | Deploy | K8s `exec` liveness + readiness probes pass | pod events | Ready | P0 |
| D3 | Deploy | Rolling restart works | trigger deploy | no downtime | P1 |

## Test execution order (recommended)

For each repo, run in this order — later steps depend on earlier ones passing:

1. **Local build (B)** — catches Dockerfile errors and package name mistakes fastest.
2. **Local runtime sanity (R)** — verifies the image starts, binds, and runs as `nobody`. This is where 90% of DHI/FIPS-migration bugs surface (missing `libstdc++6`, missing bash, wrong UID, package rename).
3. **APM/OTel attach (O)** — cheapest telemetry check; verify the agent even *starts* before deploying.
4. **FIPS crypto (F)** — must be exercised against real dependencies (RDS, DynamoDB, S3, OAuth2, JWT provider). Local docker-compose stand-in only catches obvious cases; the real signal is wk-dev deploy.
5. **Functional smoke (X)** — repo-specific happy-paths after (F) is green.
6. **Deploy (D)** — Aviary → wk-dev → sandbox. Real K8s probes are the ultimate check for the exec-probe repos (`publishing-service`, `graph-template-creation-service`).

## Universal checks that apply to every repo

These are called out per repo above but worth an at-a-glance list:

- Docker build for `--platform linux/amd64` succeeds.
- `docker build --check` reports no new warnings.
- `gha-security check_images` passes in CI.
- Trivy/Snyk shows no new HIGH/CRITICAL CVEs vs the pre-migration image.
- Container process runs as `nobody` (uid 65534). No `USER root` at the end of the final stage.
- Container starts without stack traces / bash errors / missing shared libraries.
- `kubectl get pod -o yaml` — `runAsUser` (if set) matches `65534`; capabilities include `NET_BIND_SERVICE` where needed.
- Rolling deploy in wk-dev produces zero 5xx spikes.
- Log lines look normal (no repeated errors, no infinite reconnect loops).

## Cross-cutting FIPS checks (T7) to actually run

The boilerplate "FIPS compliance restricts which cryptographic algorithms can be used at runtime" paragraph is present verbatim in every PR body, but what it means per service is different:

| Concern | Applies to | What to check |
|---|---|---|
| TLS to RDS (MySQL cert bundle) | `sa-tools-parsing-db`, `publishing-service`, `graph-template-creation-service` | `rds-combined-ca-bundle.pem` still trusted; MySQL/JanusGraph handshake completes; cipher suites permitted. |
| TLS to AWS SDKs | `sa-tools-persistence` (DynamoDB), `sa-tools-parsing` (S3) | DescribeTable / GetObject calls succeed. |
| Java keystore build | `publishing-service`, `graph-template-creation-service` | `keytool -importcert` succeeds on FIPS JDK (some providers reject certain keystore types). |
| FIPS JDK provider | all Java repos | JVM starts with the FIPS provider active (`java -XshowSettings:security` or provider list). |
| MD5 usage | all repos | grep app code + deps for MD5 usage (checksums, ETags, cache keys) — MD5 is disallowed under FIPS. |
| JWT verification | `sa-tools-persistence` (has `JWT_ENDPOINT`) | algorithm is FIPS-approved (RS256/ES256, not HS-MD5 etc.). |
| Go FIPS mode | `sa-tools-parsing-db` | `GODEBUG=fips140=on` active at runtime; Go crypto/tls handshake OK; custom cipher / min-version config compatible. |
| New Relic .NET agent bump | 4 .NET repos | APM data appears in New Relic after `9.2.0.0 → 10.52.0` (install path also changed). |
| OpenTelemetry export | Java repos | Traces / metrics still reach the OTel collector after base-image switch. |

## .NET FIPS crypto surface (repo-by-repo)

The boilerplate "TLS / cert validation / hashing / encryption / digital signatures" paragraph in each PR body maps to real .NET APIs. Grepping the four .NET service repos for those APIs gives this matrix (production code only — test-project matches are called out separately):

| Concern | vc-gen | validation | persistence | parsing (ParsingAPI) | parsing-db (Parsing_Database) |
|---|:-:|:-:|:-:|:-:|:-:|
| MD5 / SHA1 / DES / RC2 / RC4 / Rijndael / TripleDES / `PasswordDeriveBytes` | — | — | — | — | — |
| Weak RSA sizes (512 / 1024) / DSA / non-FIPS ECC curves | — | — | — | — | — |
| `RSACryptoServiceProvider` (Windows-legacy RSA wrapper) | — | — | ✅ JWT | — | ✅ JWT |
| `RSAPKCS1SignatureDeformatter` + `SetHashAlgorithm("SHA…")` (friendly-name `CryptoConfig` lookup) | — | — | ✅ JWT | — | ✅ JWT |
| `WebClient` (obsoleted; TLS delegates to OS / OpenSSL) | — | — | ✅ JWKS fetch | — | ✅ JWKS fetch |
| `HttpClientHandler` with `ServerCertificateCustomValidationCallback => true` | tests only | tests only | — | — | tests only |
| `ServicePointManager.SecurityProtocol` pin | — | — | — | — | — |
| `X509Certificate2` / `X509Store` / `.pfx` load in production | — | — | — | — | — |
| AWS SDK (S3 signing / STS / DynamoDB TLS) | — | Core + DDB v3.7 | Core + DDB + STS v4.0 | **Core + DDB + STS + S3 v4.0** | Core + DDB + STS v4.0 |
| `Microsoft.IdentityModel.Tokens` (JWT / JWKS) | — | — | tests only (v7.6.1) | tests only (v8.16.0) | — |
| `System.Security.Cryptography.Xml` (XML-DSig — FIPS-sensitive) | — | — | — | — | **v8.0.3** |
| Explicit FIPS toggles (`UseFipsCompliantAlgorithms`, `AppContext.SetSwitch(…Fips)`) | — | — | — | — | — |
| New Relic .NET agent outbound TLS to `collector.newrelic.com` | ✅ | ✅ | ✅ | ✅ | ✅ |

Legend: ✅ = used in production, "tests only" = only in `*IntegrationTests` / `*_Tests` projects (not shipped in the runtime image).

### The two live risks in production code

Both are essentially the same file, copy-pasted between `sa-tools-persistence` and the `Parsing_Database` project in `sa-tools-parsing`:

- `sa-tools-persistence/PersistenceAPI/Repositories/AuthorizeJWTAttribute.cs`
- `sa-tools-parsing/Parsing_Database/Authorization/AuthorizeJWTAttribute.cs`

The relevant slice:

```csharp
RSACryptoServiceProvider rsa = new RSACryptoServiceProvider();
rsa.ImportParameters(new RSAParameters {
    Modulus  = FromBase64Url(correctCert.Modulus),
    Exponent = FromBase64Url(correctCert.Exponent),
});

// RS512
SHA512 sha512 = SHA512.Create();
byte[] hash = sha512.ComputeHash(Encoding.UTF8.GetBytes(headerAndPayload));

RSAPKCS1SignatureDeformatter rsaDeformatter = new RSAPKCS1SignatureDeformatter(rsa);
rsaDeformatter.SetHashAlgorithm("SHA512"); // or "SHA256" for RS256
if (rsaDeformatter.VerifySignature(hash, FromBase64Url(signature))) { … }
```

Why this matters on the FIPS image:

- **SHA256 / SHA512 / RSA-PKCS#1 v1.5 are all FIPS-approved**, so the crypto itself is fine.
- Two Windows-era API surfaces are fragile on Linux-FIPS:
  1. `RSACryptoServiceProvider` — marked `[SupportedOSPlatform("windows")]` in .NET 6+. On Linux it delegates to OpenSSL, but on FIPS-hardened OpenSSL some code paths inside this wrapper have historically failed at construction / `ImportParameters`.
  2. `RSAPKCS1SignatureDeformatter.SetHashAlgorithm("SHA256")` / `"SHA512"` — the string name is resolved through `CryptoConfig.CreateFromName`. On FIPS, `CryptoConfig`'s algorithm map only returns FIPS-approved implementations, and if the friendly-name entry isn't wired up in the runtime's FIPS provider, this call returns `null` and `VerifySignature` throws.
- **If either path fails at runtime**, `OnAuthorization` returns `false` → `context.Result = new StatusCodeResult(403)` → **every authenticated request to that service returns 403 on the FIPS image**. This is exactly the "runtime exception in the deployed FIPS environment, not caught by CI" scenario the PR blurb warns about.
- The `WebClient` at the top of the file fetches JWKS over TLS. On FIPS it's subject to whatever cipher suites OpenSSL-FIPS allows — if the JWKS endpoint negotiates something out-of-suite, `PopulateJwtTokens()` silently swallows the exception (the method wraps the call in `try { … } catch { … }`) and every subsequent verification will 403.

### The AWS SDK slice

FIPS-approved AWS regions / endpoints are opt-in; the SDK constructs the endpoint hostname and TLS settings from `AWS_USE_FIPS_ENDPOINT` / `UseFIPSEndpoint = true`.

| Service | Core | DynamoDBv2 | STS | S3 | Version family |
|---|:-:|:-:|:-:|:-:|:-:|
| validation | ✔ | ✔ | — | — | 3.7.x (older) |
| persistence | ✔ | ✔ | ✔ | — | 4.0.x |
| parsing (ParsingAPI) | ✔ | ✔ | ✔ | ✔ | 4.0.x |
| parsing-db | ✔ | ✔ | ✔ | — | 4.0.x |

- AWS SDK v3.7 (validation) is on an older HTTP stack that used SHA1-based body signing in some legacy code paths (S3 SigV2). It doesn't ship S3 so this is moot here.
- AWS SDK v4.0 (persistence, parsing, parsing-db) uses SigV4 (SHA256) and TLS 1.2+ by default — FIPS-friendly *if* you point it at `-fips` endpoints in `appsettings*.json` / `AwsOptions`.

### `System.Security.Cryptography.Xml 8.0.3` (Parsing_Database only)

XML-DSig is one of the classical FIPS trip points: the default reference / canonicalization pipeline historically allowed SHA1 and non-approved transforms. If Parsing_Database consumes signed XML from any customer / upstream integration, that path needs a FIPS smoke test even though the code base itself doesn't call MD5 / SHA1 directly.

### What this means for each PR's manual-test priority

| Repo | Blurb applicability | Priority | Minimum-viable FIPS test |
|---|---|:-:|---|
| `sa-tools-vc-gen` | Boilerplate — no crypto surface except New Relic outbound TLS | **P2** | Boot on FIPS image → verify NR APM data appears within 5 min. No app-crypto test needed. |
| `sa-tools-validation` | Boilerplate — old AWS SDK v3.7 (Core / DDB only, no S3, no auth code) | **P2** | Boot on FIPS image → hit `/api/Validation` → verify DDB reads succeed and NR APM data appears. |
| `sa-tools-persistence` | **Real risk**: RSA JWT verify via legacy API + JWKS over TLS | **P0** | Curl an authenticated endpoint with a valid RS256 AND RS512 JWT; verify 200 (not 403). Also verify JWKS fetch by rotating `JWT_ENDPOINT` and watching Debug output. Then re-run the same tokens through `dotnet` on the FIPS image against production JWKS to catch `CryptoConfig` misses. |
| `sa-tools-parsing` (ParsingAPI) | AWS SDK v4 + S3 — SigV4 is FIPS-fine but endpoints need FIPS-mode | **P1** | Boot on FIPS image; run one S3 write and one DynamoDB read; set `AWS_USE_FIPS_ENDPOINT=true` and verify the SDK resolves `*.<region>.amazonaws.com` → `<service>-fips.<region>.amazonaws.com` and calls succeed. |
| `sa-tools-parsing-db` (Parsing_Database) | **Real risk**: same JWT auth path as persistence + `System.Security.Cryptography.Xml` in dependencies | **P0** | Same JWT test as persistence. Plus: if any endpoint accepts signed XML, feed it a SHA256-signed sample and confirm parsing succeeds; feed it a SHA1-signed sample and confirm it fails *cleanly* (not with a 500). |

### Additional FIPS-crypto test scenarios to add (all .NET repos)

| ID | Scenario | Verify | Priority |
|---|---|---|---|
| FIPS-DOTNET-01 | Boot the container against a real `dotnet --info` and hit every outbound integration (DB, S3, Harbour, New Relic, licensing, messaging) with `SSLKEYLOGFILE=/tmp/keys.log` | container logs show `TLSv1.2` / `TLSv1.3` only, cipher in AEAD family (GCM / ChaCha), no `CryptographicException` | P0 |
| FIPS-DOTNET-02 | Grep the running image for legacy hash / cipher references: `docker run --rm <image> bash -c 'find /app -name "*.dll" -exec strings {} +' \| rg -i 'MD5\|SHA1\|Rijndael\|TripleDES'` | every hit is either a comment / documentation string or a code path proven unreachable via the config | P1 |
| FIPS-DOTNET-03 | Simulate a customer cert with weak PBE: `openssl pkcs12 -export -legacy -macalg SHA1 -certpbe PBE-SHA1-3DES …` and mount it | log line clearly identifies the offending file / alg (controlled failure, not a runtime crash) | P1 |
| FIPS-DOTNET-04 | Force New Relic to reconnect (`kill -SIGHUP` the agent or restart) with the FIPS image and check `collector.newrelic.com` handshake completes | New Relic APM shows data for the container within 5 min | P0 |
| FIPS-DOTNET-05 | Data Protection key ring compat: encrypt a cookie / protected payload on the pre-FIPS image, deploy the FIPS image against the same key ring path, verify decrypt still works | existing sessions / cookies keep working (only applies to services issuing auth cookies or protected payloads) | P0 |

## Notes on the boilerplate PR body

The "Manual Tests" wording in the reference PR is not adapted for the other 7 PRs — it's copied verbatim (in a shortened form) into each PR's testing section. That means the string `dotnet --info` / port 8080 / `GET /s/sa-tools-vc-gen/api/health` appears (or is quoted) in PRs where none of those things actually apply. Treat that section as a description of the vc-gen reference test only, and use the per-repo commands above.

## Related artifacts

- Interactive canvas with the same content: [`~/.cursor/projects/Users-manilkumar-Documents-Workiva-SA-TOOLS/canvases/dhi-fips-test-applicability.canvas.tsx`](../../.cursor/projects/Users-manilkumar-Documents-Workiva-SA-TOOLS/canvases/dhi-fips-test-applicability.canvas.tsx)
- Initiative wiki: <https://wiki.atl.workiva.net/spaces/ARCH/pages/508330287>
- Slack: `#support-fips-compliance`
