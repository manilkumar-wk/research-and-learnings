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

## Notes on the boilerplate PR body

The "Manual Tests" wording in the reference PR is not adapted for the other 7 PRs — it's copied verbatim (in a shortened form) into each PR's testing section. That means the string `dotnet --info` / port 8080 / `GET /s/sa-tools-vc-gen/api/health` appears (or is quoted) in PRs where none of those things actually apply. Treat that section as a description of the vc-gen reference test only, and use the per-repo commands above.

## Related artifacts

- Interactive canvas with the same content: [`~/.cursor/projects/Users-manilkumar-Documents-Workiva-SA-TOOLS/canvases/dhi-fips-test-applicability.canvas.tsx`](../../.cursor/projects/Users-manilkumar-Documents-Workiva-SA-TOOLS/canvases/dhi-fips-test-applicability.canvas.tsx)
- Initiative wiki: <https://wiki.atl.workiva.net/spaces/ARCH/pages/508330287>
- Slack: `#support-fips-compliance`
