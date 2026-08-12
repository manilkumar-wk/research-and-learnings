# Grype scan: workiva/grc-services:8990198

## Scan metadata

| Field | Value |
| --- | --- |
| Image | `drydock.workiva.net/workiva/grc-services:8990198` |
| Command | `grype drydock.workiva.net/workiva/grc-services:8990198 --by-cve --only-fixed` |
| Packages cataloged | 250 |
| Vulnerability matches | 65 |
| By severity (scan summary) | 1 critical, 296 high, 99 medium, 9 low, 1 negligible |
| By status (scan summary) | 404 fixed, 2 not-fixed, 341 ignored |
| Distro note | 250 packages from EOL distro `amzn 2` |
| Table rows (`--only-fixed`) | 65 |

## Findings (`--only-fixed`)

| Package | Installed | Fixed in | Type | Vulnerability | Severity | EPSS | Risk |
| --- | --- | --- | --- | --- | --- | --- | --- |
| python | 3.11.9 | 3.8.20, 3.9.20, 3.10.15, 3.11.10, ... | binary | CVE-2024-7592 | High | 2.3% (81st) | 1.7 |
| python | 3.11.9 | 3.8.20, 3.9.20, 3.10.15, 3.11.10, ... | binary | CVE-2024-6232 | High | 2.2% (80th) | 1.7 |
| python | 3.11.9 | 3.8.20, 3.9.20, 3.10.15, 3.11.10, ... | binary | CVE-2023-27043 | Medium | 2.5% (83rd) | 1.3 |
| python | 3.11.9 | 3.9.23, 3.10.18, 3.11.13, 3.12.11, ... | binary | CVE-2025-4517 | Critical | 1.3% (67th) | 1.2 |
| python | 3.11.9 | 3.10.20, 3.11.15, 3.12.13, 3.13.11, ... | binary | CVE-2025-13836 | High | 1.6% (73rd) | 1.1 |
| python | 3.11.9 | 3.8.20, 3.9.20, 3.10.15, 3.11.10, ... | binary | CVE-2024-8088 | High | 1.3% (67th) | 1.0 |
| python | 3.11.9 | 3.9.23, 3.10.18, 3.11.13, 3.12.11, ... | binary | CVE-2025-4138 | High | 1.2% (65th) | 0.9 |
| python | 3.11.9 | 3.9.22, 3.10.17, 3.11.12, 3.12.9, ... | binary | CVE-2025-0938 | Medium | 1.5% (72nd) | 0.9 |
| python | 3.11.9 | 3.8.20, 3.9.20, 3.10.15, 3.11.10, ... | binary | CVE-2024-4032 | High | 1.1% (61st) | 0.8 |
| python | 3.11.9 | 3.9.23, 3.10.18, 3.11.13, 3.12.11, ... | binary | CVE-2025-4330 | High | 0.8% (54th) | 0.6 |
| python | 3.11.9 | 3.13.14, 3.14.6, 3.15.0b2 | binary | CVE-2026-7210 | High | 0.8% (52nd) | 0.6 |
| python | 3.11.9 | 3.9.21, 3.10.16, 3.11.11, 3.12.8, ... | binary | CVE-2024-50602 | Medium | 1.0% (60th) | 0.6 |
| python | 3.11.9 | 3.15.0b4 | binary | CVE-2026-11940 | High | 0.7% (50th) | 0.6 |
| python | 3.11.9 | 3.15.0 | binary | CVE-2026-15308 | High | 0.6% (46th) | 0.5 |
| python | 3.11.9 | 3.13.14, 3.14.5rc1, 3.15.0b1 | binary | CVE-2026-6100 | High | 0.6% (44th) | 0.5 |
| python | 3.11.9 | 3.9.24, 3.10.19, 3.11.14, 3.12.12, ... | binary | CVE-2025-8194 | High | 0.6% (45th) | 0.5 |
| python | 3.11.9 | 3.9.21, 3.10.16, 3.11.11, 3.12.8, ... | binary | CVE-2024-9287 | High | 0.6% (47th) | 0.5 |
| python | 3.11.9 | 3.13.13, 3.14.4, 3.15.0a8 | binary | CVE-2026-4224 | High | 0.6% (47th) | 0.5 |
| python | 3.11.9 | 3.10.20, 3.11.15, 3.12.13, 3.13.11, ... | binary | CVE-2025-12084 | Medium | 0.8% (52nd) | 0.4 |
| python | 2.7.18-1.amzn2.0.21 | 2.7.18-1.amzn2.0.22 | rpm | CVE-2026-1502 | High | 0.6% (46th) | 0.4 |
| python-libs | 2.7.18-1.amzn2.0.21 | 2.7.18-1.amzn2.0.22 | rpm | CVE-2026-1502 | High | 0.6% (46th) | 0.4 |
| python | 3.11.9 | 3.8.20, 3.9.20, 3.10.15, 3.11.10, ... | binary | CVE-2024-6923 | Medium | 0.8% (52nd) | 0.4 |
| python | 3.11.9 | 3.9.23, 3.10.18, 3.11.13, 3.12.11, ... | binary | CVE-2025-4435 | High | 0.5% (41st) | 0.4 |
| python | 3.11.9 | 3.13.14, 3.14.6, 3.15.0b2 | binary | CVE-2026-7774 | Medium | 0.6% (45th) | 0.4 |
| python | 3.11.9 | 3.15.0b4 | binary | CVE-2026-11972 | High | 0.4% (36th) | 0.3 |
| python | 3.11.9 | 3.9.23, 3.10.18, 3.11.13, 3.12.11, ... | binary | CVE-2024-12718 | Medium | 0.7% (48th) | 0.3 |
| python | 3.11.9 | 3.13.13, 3.14.4, 3.15.0a8 | binary | CVE-2026-3644 | High | 0.5% (38th) | 0.3 |
| python | 3.11.9 | 3.13.14, 3.14.6, 3.15.0b3 | binary | CVE-2026-9669 | High | 0.4% (35th) | 0.3 |
| python | 3.11.9 | 3.13.14, 3.14.5rc1, 3.15.0b1 | binary | CVE-2026-3298 | High | 0.4% (31st) | 0.3 |
| python | 3.11.9 | 3.10.20, 3.11.15, 3.12.13, 3.13.12, ... | binary | CVE-2026-1299 | Medium | 0.6% (43rd) | 0.3 |
| python | 3.11.9 | 3.13.14, 3.14.5rc1, 3.15.0b1 | binary | CVE-2026-1502 | Medium | 0.6% (43rd) | 0.3 |
| python | 3.11.9 | 3.10.20, 3.11.15, 3.12.13, 3.13.12, ... | binary | CVE-2025-11468 | Medium | 0.5% (43rd) | 0.3 |
| python | 3.11.9 | 3.13.10, 3.14.1, 3.15.0a2 | binary | CVE-2025-12781 | Medium | 0.5% (40th) | 0.3 |
| python | 3.11.9 | 3.13.14, 3.14.6, 3.15.0b2 | binary | CVE-2026-3276 | Medium | 0.5% (39th) | 0.3 |
| python | 3.11.9 | 3.10.20, 3.11.15, 3.12.13, 3.13.12, ... | binary | CVE-2025-15282 | Medium | 0.5% (38th) | 0.3 |
| python | 3.11.9 | 3.10.20, 3.11.15, 3.12.13, 3.13.12, ... | binary | CVE-2026-0865 | Medium | 0.5% (37th) | 0.3 |
| python | 3.11.9 | 3.13.14, 3.14.6, 3.15.0b2 | binary | CVE-2026-8328 | Medium | 0.5% (37th) | 0.2 |
| setuptools | 81.0.0 | 83.0.0 | python | CVE-2026-59890 | Medium | 0.4% (33rd) | 0.2 |
| python | 3.11.9 | 3.10.20, 3.11.15, 3.12.13, 3.13.12, ... | binary | CVE-2026-0672 | Medium | 0.4% (32nd) | 0.2 |
| python | 3.11.9 | 3.9.24, 3.10.19, 3.11.14, 3.12.12, ... | binary | CVE-2025-6069 | Medium | 0.5% (38th) | 0.2 |
| python | 3.11.9 | 3.13.14, 3.14.5rc1, 3.15.0b1 | binary | CVE-2026-4786 | High | 0.3% (21st) | 0.2 |
| python | 3.11.9 | 3.15.0a6 | binary | CVE-2025-15366 | Medium | 0.4% (28th) | 0.2 |
| python | 3.11.9 | 3.15.0a6 | binary | CVE-2025-15367 | Medium | 0.3% (23rd) | 0.2 |
| python | 3.11.9 | 3.9.24, 3.10.19, 3.11.14, 3.12.12, ... | binary | CVE-2025-8291 | Medium | 0.4% (27th) | 0.2 |
| vim-data | 2:9.0.2153-1.amzn2.0.8 | 9.0.2153-1.amzn2.0.9 | rpm | CVE-2026-59856 | High | 0.2% (12th) | 0.2 |
| vim-minimal | 2:9.0.2153-1.amzn2.0.8 | 9.0.2153-1.amzn2.0.9 | rpm | CVE-2026-59856 | High | 0.2% (12th) | 0.2 |
| cryptography | 47.0.0 | 49.0.0 | python | CVE-2026-69249 | High | 0.2% (9th) | 0.2 |
| python-dotenv | 1.0.0 | 1.2.2 | python | CVE-2026-28684 | Medium | 0.3% (17th) | 0.1 |
| cryptography | 47.0.0 | 50.0.0 | python | CVE-2026-69247 | High | 0.2% (7th) | 0.1 |
| python | 3.11.9 | 3.13.13, 3.14.4, 3.15.0a8 | binary | CVE-2026-4519 | Low | 0.3% (23rd) | 0.1 |
| libacl | 2.2.51-14.amzn2 | 2.2.51-14.amzn2.0.1 | rpm | CVE-2026-54369 | High | 0.2% (4th) | 0.1 |
| python | 3.11.9 | 3.13.13, 3.14.4, 3.15.0a7 | binary | CVE-2026-2297 | Medium | 0.2% (11th) | 0.1 |
| cryptography | 47.0.0 | 49.0.0 | python | CVE-2026-69248 | Medium | 0.2% (8th) | 0.1 |
| python | 3.11.9 | 3.9.23, 3.10.18, 3.11.13, 3.12.11, ... | binary | CVE-2025-4516 | Medium | 0.2% (9th) | 0.1 |
| python | 3.11.9 | 3.13.14, 3.14.5rc1, 3.15.0b1 | binary | CVE-2026-6019 | Medium | 0.2% (13th) | 0.1 |
| python | 3.11.9 | 3.13.13, 3.14.4, 3.15.0a8 | binary | CVE-2026-3446 | Medium | 0.2% (8th) | 0.1 |
| glib2 | 2.56.1-9.amzn2.0.14 | 2.56.1-9.amzn2.0.15 | rpm | CVE-2026-16118 | High | 0.1% (3rd) | 0.1 |
| python | 3.11.9 | 3.13.10, 3.14.1, 3.15.0a3 | binary | CVE-2025-13837 | Medium | 0.2% (12th) | < 0.1 |
| libxml2 | 2.9.1-6.amzn2.5.25 | 2.9.1-6.amzn2.5.26 | rpm | CVE-2026-11979 | Medium | 0.1% (4th) | < 0.1 |
| python | 3.11.9 | 3.15.0b3 | binary | CVE-2026-12003 | Medium | 0.1% (4th) | < 0.1 |
| python | 3.11.9 | 3.9.25, 3.10.20, 3.11.15, 3.12.13, ... | binary | CVE-2025-6075 | Medium | 0.1% (3rd) | < 0.1 |
| python | 3.11.9 | 3.15.0b4 | binary | CVE-2026-0864 | Medium | 0.1% (2nd) | < 0.1 |
| python | 3.11.9 | 3.13.13, 3.14.4, 3.15.0a8 | binary | CVE-2025-13462 | Low | 0.2% (6th) | < 0.1 |
| python | 3.11.9 | 3.13.13, 3.14.4, 3.15.0a8 | binary | CVE-2026-3479 | Negligible | 0.2% (14th) | < 0.1 |
| cryptography | 47.0.0 | 48.0.1 | python | GHSA-537c-gmf6-5ccf | High | N/A | N/A |

## Summary by package

| Package | Finding count |
| --- | ---: |
| python | 53 |
| cryptography | 4 |
| python-libs | 1 |
| setuptools | 1 |
| vim-data | 1 |
| vim-minimal | 1 |
| python-dotenv | 1 |
| libacl | 1 |
| glib2 | 1 |
| libxml2 | 1 |

## Summary by severity

| Severity | Count |
| --- | ---: |
| Critical | 1 |
| High | 29 |
| Medium | 32 |
| Low | 2 |
| Negligible | 1 |
