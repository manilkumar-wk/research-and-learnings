---
name: dependabot-cve
description: >-
  Fix Dependabot security alerts and CVE Jira tickets using the safest
  maintainable approach: identify repo/dependency/path, prefer root-package
  upgrades over overrides, validate lockfiles/tests, then open a PR without
  merging. Use when the user asks to fix a Dependabot alert, CVE, vulnerable
  dependency, or Dependabot Jira ticket (e.g. GRC-*).
---

# Dependabot CVE fix

Fix Dependabot / CVE failures using the safest and most maintainable approach.
Do not merge the PR.

## 1. Read the alert / ticket

Read the Dependabot alert and related Jira ticket(s).

## 2. Identify

- The affected repository and dependency
- The vulnerability and affected versions
- The recommended fixed version
- Whether the dependency is direct or transitive
- The dependency path that introduces it

## 3. Inspect before changing

Before making changes, inspect the repository’s dependency files, lockfiles,
build configuration, existing overrides, and contribution guidelines.

Also inspect existing dependency-update PRs for conventions.

## Fix approach

### Direct dependency

- Upgrade it to the minimum safe compatible version.
- Resolve any build or dependency errors caused by the upgrade.

### Transitive dependency

- Identify the direct/root dependency introducing it.
- Check whether a safe version of that root dependency is available.
- Prefer upgrading the root dependency when it safely resolves the
  vulnerability.
- If no suitable root upgrade is available, add a narrowly scoped dependency
  override or constraint for the vulnerable transitive dependency.
- Clearly document why the override was necessary.
- Follow the package manager’s recommended syntax and the repository’s
  existing patterns.

## Safety requirements

- Do not make business-logic or unrelated code changes.
- Preserve existing behavior.
- Use the smallest safe dependency change.
- Consider compatibility, runtime, deployment, and downstream-consumer
  implications.
- Do not apply breaking major-version upgrades without strong justification.
- Do not weaken or disable security checks, tests, or scanners.
- Do not suppress or dismiss the alert as the primary solution.
- Do not expose credentials, tokens, customer data, or internal secrets.
- If the change is too risky, unclear, requires significant logic changes, or
  cannot be safely validated or fixed without major risk, stop and explain
  rather than forcing a fix.

## Validate

- Regenerate the lockfile using the correct package-manager command.
- Confirm the resolved dependency version is no longer vulnerable.
- Run the relevant build, unit tests, linting, and security/dependency scans.
- Review the dependency tree again to confirm that no vulnerable version
  remains.
- If any validation fails, investigate and fix it before continuing. Do not
  hide or bypass failures.

## Git and PR

1. Create a separate branch for each repository or logically related fix.
   Prefer repository naming conventions; if none, use
   `fix/dependabot-<dependency-name>`.
2. Commit only the required dependency and lockfile changes.
3. Push the branch and create a pull request.
4. Use a clear PR title, such as:
   `Fix Dependabot vulnerability in <dependency>`

### PR description must include

- Related Jira ticket and Dependabot alert links
- Vulnerability summary
- Whether the dependency was direct or transitive
- Dependency path and root cause
- Previous and updated versions
- Why this fix approach was selected
- Files changed
- Compatibility or breaking-change assessment
- Commands and tests executed
- Validation results
- Any remaining risks or follow-up work

Do not merge the PR.

## Done criteria

Provide the branch name, commit hash, PR link, dependency path, selected fix,
and validation summary.
