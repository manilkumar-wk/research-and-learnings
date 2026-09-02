# OpenAPI Generator 7.25.0 Upgrade Failure Analysis: form_service

**PR:** [#4029](https://github.com/Workiva/form_service/pull/4029) — `chore(deps): Bump org.openapitools:openapi-generator-gradle-plugin from 7.20.0 to 7.25.0`
**Failed Run:** [33171930458](https://github.com/Workiva/form_service/actions/runs/33171930458/job/98981510561)
**Date:** 2026-09-02

---

## Root Cause

**The OpenAPI Generator Gradle plugin changed all file/directory properties from `String`/`Property<String>` to Gradle typed file properties (`RegularFileProperty`, `DirectoryProperty`) in version 7.22.0.**

The breaking change was introduced by [PR #23042 — "Modernize plugin to use Lazy Configuration API"](https://github.com/OpenAPITools/openapi-generator/pull/23042), released in [v7.22.0](https://github.com/OpenAPITools/openapi-generator/releases/tag/v7.22.0).

The failing line in `export/build.gradle:144`:

```groovy
inputSpec.set("$rootDir/export/api/v1/openapi.yaml")
```

`"$rootDir/..."` is a Groovy GString interpolation that produces a `GStringImpl` at runtime. In plugin 7.20.0, `inputSpec` was a `Property<String>` that accepted strings. In 7.22.0+, it became a `RegularFileProperty` (type `org.gradle.api.file.RegularFile`), which cannot be set with a `GStringImpl`.

**Error message:**
```
Cannot set the value of extension 'openApiGenerate' property 'inputSpec'
of type org.gradle.api.file.RegularFile using an instance of type
org.codehaus.groovy.runtime.GStringImpl.
```

---

## Affected Files and Line Numbers

### 4 OpenAPI generation configurations across 3 modules are affected:

| Module | File | Line | Property | Current Value | Issue |
|--------|------|------|----------|---------------|-------|
| **export** | `export/build.gradle` | 144 | `inputSpec` | `"$rootDir/export/api/v1/openapi.yaml"` | GString → RegularFile |
| **export** | `export/build.gradle` | 145 | `outputDir` | `"$buildDir/generated-sources"` | GString → Directory |
| **service** (DSS) | `service/build.gradle` | 190 | `inputSpec` | `dssOpenApiSpecPath.get()` | String → RegularFile |
| **service** (DSS) | `service/build.gradle` | 191 | `outputDir` | `"$buildDir/generate-resources"` | GString → Directory |
| **service** (ESG) | `service/build.gradle` | 218 | `inputSpec` | `"$rootDir/service/api/v1/esgprogramapi.yaml"` | GString → RegularFile |
| **service** (ESG) | `service/build.gradle` | 219 | `outputDir` | `"$buildDir/generate-resources"` | GString → Directory |
| **universe** | `universe/build.gradle` | 131 | `inputSpec` | `"$rootDir/universe/api/v1/openapi.yaml"` | GString → RegularFile |
| **universe** | `universe/build.gradle` | 132 | `outputDir` | `"$buildDir/generated-sources"` | GString → Directory |

The build fails at `export/build.gradle:144` first because Gradle evaluates projects alphabetically. **Fixing only `export` will expose identical failures in `service` and `universe`.**

---

## Complete Property Type Migration (7.22.0)

| Property | Old Type (≤7.21.0) | New Type (≥7.22.0) | Bridge Setter (for tasks) |
|----------|--------------------|--------------------|---------------------------|
| `inputSpec` | `Property<String>` | `RegularFileProperty` | `setInputSpecAsString(String)` |
| `outputDir` | `Property<String>` | `DirectoryProperty` | `setOutputDirAsString(String)` |
| `templateDir` | `Property<String>` | `DirectoryProperty` | `setTemplateDirAsString(String)` |
| `configFile` | `Property<String>` | `RegularFileProperty` | `setConfigFileAsString(String)` |
| `ignoreFileOverride` | `Property<String>` | `RegularFileProperty` | `setIgnoreFileOverrideAsString(String)` |
| `inputSpecRootDirectory` | `Property<String>` | `DirectoryProperty` | `setInputSpecRootDirectoryAsString(String)` |

Additionally, scalar properties changed from raw types to Gradle managed properties:
- `generatorName`, `apiPackage`, `modelPackage` → `Property<String>` (compatible with `.set(String)`)
- `configOptions` → `MapProperty<String, String>` (compatible with `.set(Map)`)

---

## Recommended Fix

### Strategy

For the **extension block** (`openApiGenerate { ... }`), use Gradle's `file()` method to convert strings to `File` objects, which the plugin's extension bridge setters accept. For `outputDir`, use `layout.buildDirectory.dir()` for Gradle 8+ best practices.

### export/build.gradle (lines 143-145)

```diff
 openApiGenerate {
     generatorName.set("spring")
-    inputSpec.set("$rootDir/export/api/v1/openapi.yaml")
-    outputDir.set("$buildDir/generated-sources")
+    inputSpec.set(file("$rootDir/export/api/v1/openapi.yaml"))
+    outputDir.set(layout.buildDirectory.dir("generated-sources").get().asFile.path)
     apiPackage.set("org.openapi.export.api")
```

**Or, using the Provider API (preferred for Gradle 8+):**

```diff
 openApiGenerate {
     generatorName.set("spring")
-    inputSpec.set("$rootDir/export/api/v1/openapi.yaml")
-    outputDir.set("$buildDir/generated-sources")
+    inputSpec.set(layout.projectDirectory.file("api/v1/openapi.yaml"))
+    outputDir.set(layout.buildDirectory.dir("generated-sources"))
     apiPackage.set("org.openapi.export.api")
```

### service/build.gradle — DSS task (lines 189-191)

```diff
 openApiGenerate {
     generatorName.set("spring")
-    inputSpec.set(dssOpenApiSpecPath.get())
-    outputDir.set("$buildDir/generate-resources")
+    inputSpec.set(file(dssOpenApiSpecPath.get()))
+    outputDir.set(layout.buildDirectory.dir("generate-resources"))
     apiPackage.set("org.openapi.datasharingservice.api")
```

**Note:** `dssOpenApiSpecPath.get()` returns a `String` from a Parsimony-resolved path. Wrapping it in `file()` converts it to a `File` that the `RegularFileProperty` accepts.

### service/build.gradle — ESG task (lines 217-219)

```diff
 openApiGenerateEsg {
     generatorName.set("spring")
-    inputSpec.set("$rootDir/service/api/v1/esgprogramapi.yaml")
-    outputDir.set("$buildDir/generate-resources")
+    inputSpec.set(layout.projectDirectory.file("api/v1/esgprogramapi.yaml"))
+    outputDir.set(layout.buildDirectory.dir("generate-resources"))
     apiPackage.set("org.openapi.esgprogram.api")
```

### universe/build.gradle (lines 130-132)

```diff
 openApiGenerate {
     generatorName.set("spring")
-    inputSpec.set("$rootDir/universe/api/v1/openapi.yaml")
-    outputDir.set("$buildDir/generated-sources")
+    inputSpec.set(layout.projectDirectory.file("api/v1/openapi.yaml"))
+    outputDir.set(layout.buildDirectory.dir("generated-sources"))
     apiPackage.set("org.openapi.universe.api")
```

---

## Generated Code Differences (7.20.0 → 7.25.0)

**Regenerating code WILL produce different output.** Key changes across the 5-version span:

| Version | Change | Impact |
|---------|--------|--------|
| **7.21.0** | Spring Boot 3.x defaults (`useSpringBoot3=true`) | May change generated imports/annotations if not already using Spring Boot 3 |
| **7.22.0** | JSpecify annotations for null safety | New `@Nullable`/`@NonNull` annotations on generated models |
| **7.22.0** | Jackson3 support added | New serialization annotations possible |
| **7.24.0** | `@JsonSetter` gated on `openApiNullable` for optional non-nullable fields | Changed annotation behavior on model fields |
| **7.24.0** | `@JsonInclude` annotation changes for optional+nullable fields | Different serialization defaults |
| **7.25.0** | `@JsonInclude`/`@JsonSetter` made opt-in | Reverts to not overriding global ObjectMapper settings |
| **7.25.0** | `PreAuthorize` from OpenAPI security scopes | New security annotations on API interfaces |

**Risk:** Generated models, APIs, and their annotations will change. This may cause:
- Compilation errors if handwritten code depends on specific generated signatures
- Runtime behavior changes in JSON serialization (null handling, optional fields)
- New security annotations that may conflict with existing authorization setup

**Recommendation:** After fixing the build configuration, diff the regenerated sources against the current committed generated code to identify and review all differences before merging.

---

## Gradle Deprecation Warnings

The build also reports deprecated Gradle features incompatible with Gradle 9.0. These are **separate from the build failure** and include:

| Warning | Cause | Action |
|---------|-------|--------|
| `$buildDir` usage | `$buildDir` is deprecated in Gradle 8+; use `layout.buildDirectory` | Migrate all `$buildDir` references |
| Task configuration avoidance | Some tasks configured eagerly instead of lazily | Use `tasks.register` instead of `tasks.create` |
| Mutating configurations after resolution | Dependencies resolved too early | Restructure dependency blocks |

These warnings do NOT cause the current build failure but should be addressed in a separate PR for Gradle 9 readiness. The `outputDir` fix above (`layout.buildDirectory.dir(...)`) partially addresses the `$buildDir` deprecation.

---

## Downstream Impact

### Generated API Artifacts

The `form_service` modules generate Spring API interfaces and models consumed by:
- The service itself (controllers implement generated API interfaces)
- The `lib` module publishes artifacts consumed by other services
- Integration tests in `:integration-tests` module

**If generated code changes** (which it will — see section above), all modules that compile against the generated code must be verified:
- `:service` compileJava/compileKotlin depends on `openApiGenerate` and `openApiGenerateEsg`
- `:export` compileJava depends on `openApiGenerate`
- `:universe` compileJava depends on `openApiGenerate`

### External Consumers

The `parsimony.yml` exposes three API specs:
- `esg_program_api.yaml` → `service/api/v1/esgprogramapi.yaml`
- `export_api.yaml` → `export/api/v1/openapi.yaml`
- `universe_api.yaml` → `universe/api/v1/openapi.yaml`

External consumers that resolve these specs via Parsimony and generate their own clients are **not affected** by the plugin upgrade — the OpenAPI spec files themselves are unchanged. Only the generated server-side code changes.

---

## Verification Commands

```bash
# 1. Fix the build.gradle files as described above

# 2. Verify the build passes
./gradlew clean openApiGenerate openApiGenerateEsg ktlintFormat build -x :integration-tests:test

# 3. Diff generated sources to review changes
git diff --stat  # See which generated files changed
git diff -- '*.java' '*.kt'  # Review actual code changes

# 4. Run full test suite
./gradlew test

# 5. Run integration tests (if environment available)
./gradlew :integration-tests:test

# 6. Check for remaining deprecation warnings
./gradlew build --warning-mode all 2>&1 | grep -i "deprecated"
```

---

## Tests That Should Pass Before Merging

| Test Suite | Command | Validates |
|-----------|---------|-----------|
| Unit tests (all modules) | `./gradlew test` | Generated code compiles, business logic works |
| Lint check | `./gradlew ktlintCheck` | Code style compliance |
| Integration tests | `./gradlew :integration-tests:test` | API contracts, serialization, end-to-end |
| OpenAPI generation | `./gradlew openApiGenerate openApiGenerateEsg` | Specs generate without errors |
| Full build | `./gradlew build -x :integration-tests:test` | Everything compiles and packages |

---

## Final Recommendation

### **Hold the upgrade. Split into two PRs.**

**Rationale:**

1. **PR 1 (low risk):** Fix the Gradle configuration to use Provider API types. This is a **configuration-only change** that works with both 7.20.0 and 7.22.0+. It also addresses some Gradle 9 deprecation warnings. Merge this first on the current 7.20.0 version to confirm no regressions.

2. **PR 2 (medium risk):** Bump the plugin version from 7.20.0 to 7.25.0. This will regenerate all API code with updated templates. The diff should be carefully reviewed for:
   - `@JsonSetter` / `@JsonInclude` annotation changes (affects serialization behavior)
   - JSpecify null-safety annotations (may conflict with existing `@Nullable` usage)
   - `@PreAuthorize` security annotations (may interfere with existing authorization)
   - Spring Boot 3.x default changes (if not already explicitly set)

**If a single PR is preferred:** Fix the configuration AND bump the version together, but commit the generated code changes separately from the configuration fix so reviewers can see both diffs clearly. Run the full integration test suite before merging.

### Risk Summary

| Risk | Severity | Mitigation |
|------|----------|------------|
| Build failure from type mismatch | **Resolved** by configuration fix | Use `file()` or `layout.projectDirectory.file()` |
| Generated code changes serialization behavior | **Medium** | Diff generated code, run integration tests |
| New security annotations on APIs | **Low-Medium** | Review generated `@PreAuthorize` annotations |
| Gradle 9 deprecation warnings | **Low** | Address in separate PR |
| Downstream Parsimony consumers | **None** | Specs unchanged, only server code changes |
