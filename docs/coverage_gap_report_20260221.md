---
layout: "akamai"
page_title: "Akamai Terraform Provider: Coverage Gap Report"
description: |-
  Formal TechDocs vs. codebase coverage gap report for the Akamai Terraform Provider v9.3,
  documenting per-product breakdowns of documented items, implemented items, coverage
  percentages, and gap status across all subproviders.
---

# Akamai Terraform Provider — TechDocs Coverage Gap Report

**Report Date:** 2026-02-21

**Provider Version:** v9.3

This report presents a systematic comparison of the Akamai TechDocs v9.3 sidebar against all 20
subprovider `provider.go` registrations in the Akamai Terraform Provider codebase. It documents
coverage percentages, gaps, and status per product, serving as the baseline capture and formal
validation artifact for the provider's resource and data source completeness.

The authoritative indexes used as the basis for this report are
[`docs/resources/resources.md`](resources/resources.md) and
[`docs/data-sources/data-sources.md`](data-sources/data-sources.md), which link to TechDocs pages
for each subprovider.

## Executive Summary

**Key Finding:** All 19 TechDocs-listed subproviders are at **100% coverage** of non-deprecated
resources and data sources. The codebase implements every resource and data source documented in
the TechDocs v9.3 sidebar.

- **TechDocs effective total:** 345 items (141 resources + 204 data sources), excluding deprecated
  items.
- **Codebase total:** 365 items (147 resources + 218 data sources), across all 20 subproviders.
- **Gaps found:** 0 — no new resource or data source implementations are required.
- **Coverage targets:**
  - All Priority 1 products (appsec, botman, gtm, property, edgeworkers) exceed the ≥90% target.
  - All Priority 2 products exceed the ≥70% target.
  - All thin subproviders (clientlists, datastream) meet the 100% requirement.

**Exclusions:**

- The 20th subprovider, **Cloud Certificate Manager** (`cloudcertificates`), is a Beta product
  (since v9.2.0) and is not listed in the TechDocs v9.3 sidebar. It is excluded from coverage
  metrics per the BETA/preview exclusion rules.
- 2 deprecated Bot Manager items (`akamai_botman_challenge_interception_rules` resource and data
  source) were excluded per the deprecation exclusion rules. These items were previously removed
  from the codebase and replaced by `akamai_botman_challenge_injection_rules`.

## Methodology

### TechDocs Parsing

Every TechDocs page linked from
[`docs/resources/resources.md`](resources/resources.md) (21 rows) and
[`docs/data-sources/data-sources.md`](data-sources/data-sources.md) (21 rows) was parsed to
extract the complete list of documented resources and data sources per subprovider. This list
constitutes the "TechDocs set" — the authoritative inventory of what the provider should implement.

### Codebase Extraction

Registration counts were extracted from each product's `provider.go` by inspecting the four
registration methods:

- `SDKResources()` — returns `map[string]*schema.Resource` for SDK-pattern resources
- `SDKDataSources()` — returns `map[string]*schema.Resource` for SDK-pattern data sources
- `FrameworkResources()` — returns `[]func() resource.Resource` for Framework-pattern resources
- `FrameworkDataSources()` — returns `[]func() datasource.DataSource` for Framework-pattern data sources

The sum of SDK and Framework registrations per category produces the "implemented set" for each
product.

### Comparison

Gaps are defined as items present in the TechDocs set but absent from the implemented set.
Coverage percentage is calculated as:

```
Coverage % = min(100%, Implemented items that match TechDocs / TechDocs items) × 100
```

Products where implementations exceed the TechDocs count are capped at 100%.

### Priority Classification

| Priority | Target | Products |
|---|---|---|
| **Priority 1** | ≥90% coverage | appsec, botman, gtm, property, edgeworkers |
| **Priority 2** | ≥70% coverage | accountprotection, apidefinitions, cloudaccess, cloudlets, cloudwrapper, cps, dns, iam, imaging, mtlskeystore, mtlstruststore, networklists |
| **Thin** | 100% required | clientlists (4 items), datastream (4 items) — products with fewer than 5 TechDocs items |

### Exclusion Criteria

- **Deprecated items:** Resources or data sources marked as deprecated in TechDocs are excluded
  from the TechDocs set and coverage calculations.
- **BETA/preview items:** Resources or data sources for Beta products not listed in TechDocs are
  excluded unless they are in Priority 1 and not deprecated.

### Scope-Limiting Rules

The following caps apply to limit implementation scope when gaps are large:

| Rule | Threshold | Cap | Triggered? |
|---|---|---|---|
| Per-product cap | >30 gaps in a single product | Implement only first 20 | **No** — 0 gaps per product |
| Global cap | >100 total gaps across all products | Implement Priority 1 only | **No** — 0 total gaps |
| Priority 1 cap | >60 gaps in Priority 1 alone | Implement top 50 only | **No** — 0 Priority 1 gaps |

None of these caps were triggered as all products are at 100% coverage.

## Coverage Summary

| Product | Priority | Resources (TechDocs) | Resources (Implemented) | Data Sources (TechDocs) | Data Sources (Implemented) | Coverage % | Status |
|---|---|---|---|---|---|---|---|
| Application Security (appsec) | P1 | 55 | 55 | 52 | 56 | 100% | Met |
| Bot Manager (botman) | P1 | 24 | 26 | 29 | 32 | 100% | Met |
| Global Traffic Management (gtm) | P1 | 7 | 7 | 11 | 11 | 100% | Met |
| Property (property) | P1 | 9 | 11 | 25 | 29 | 100% | Met |
| EdgeWorkers (edgeworkers) | P1 | 4 | 4 | 6 | 6 | 100% | Met |
| Account Protection (accountprotection) | P2 | 4 | 4 | 4 | 4 | 100% | Met |
| API Definitions (apidefinitions) | P2 | 3 | 3 | 3 | 3 | 100% | Met |
| Client Lists (clientlists) | Thin | 2 | 2 | 2 | 2 | 100% | Met |
| Cloud Access Manager (cloudaccess) | P2 | 1 | 1 | 4 | 4 | 100% | Met |
| Cloudlets (cloudlets) | P2 | 4 | 4 | 12 | 12 | 100% | Met |
| Cloud Wrapper (cloudwrapper) | P2 | 2 | 2 | 6 | 6 | 100% | Met |
| Certificates (cps) | P2 | 4 | 4 | 5 | 5 | 100% | Met |
| DataStream (datastream) | Thin | 1 | 1 | 3 | 3 | 100% | Met |
| Edge DNS (dns) | P2 | 2 | 2 | 3 | 3 | 100% | Met |
| Identity and Access Management (iam) | P2 | 7 | 7 | 25 | 25 | 100% | Met |
| Image and Video Manager (imaging) | P2 | 3 | 3 | 2 | 2 | 100% | Met |
| mTLS Origin Keystore (mtlskeystore) | P2 | 3 | 3 | 3 | 3 | 100% | Met |
| mTLS Edge Truststore (mtlstruststore) | P2 | 2 | 2 | 8 | 8 | 100% | Met |
| Network Lists (networklists) | P2 | 4 | 4 | 1 | 1 | 100% | Met |
| Cloud Certificate Manager (cloudcertificates) | N/A | — | 2 | — | 3 | N/A | Beta (Out of Scope) |
| **TOTALS** | | **141** | **147** | **204** | **218** | **100%** | **All Met** |

> **Note:** Coverage % is calculated as the percentage of TechDocs-documented items that have
> corresponding implementations in the codebase. Products where implementations exceed the
> TechDocs count are capped at 100%. The TechDocs totals (141 resources + 204 data sources = 345)
> represent the effective non-deprecated documented items. The implementation totals
> (147 resources + 218 data sources = 365) include codebase extras not documented in TechDocs as
> well as the Beta Cloud Certificate Manager subprovider (5 items). The TOTALS row excludes
> Cloud Certificate Manager from TechDocs columns since it has no TechDocs entries.

### Implementation Count Breakdown

Resources (Implemented) and Data Sources (Implemented) include both SDK and Framework
registrations from each product's `provider.go`:

| Product | SDK Resources | Framework Resources | Total Resources | SDK Data Sources | Framework Data Sources | Total Data Sources |
|---|---|---|---|---|---|---|
| appsec | 54 | 1 | 55 | 54 | 2 | 56 |
| botman | 26 | 0 | 26 | 32 | 0 | 32 |
| gtm | 7 | 0 | 7 | 3 | 8 | 11 |
| property | 6 | 5 | 11 | 19 | 10 | 29 |
| edgeworkers | 4 | 0 | 4 | 6 | 0 | 6 |
| accountprotection | 4 | 0 | 4 | 4 | 0 | 4 |
| apidefinitions | 0 | 3 | 3 | 0 | 3 | 3 |
| clientlists | 2 | 0 | 2 | 0 | 2 | 2 |
| cloudaccess | 0 | 1 | 1 | 0 | 4 | 4 |
| cloudlets | 4 | 0 | 4 | 10 | 2 | 12 |
| cloudwrapper | 0 | 2 | 2 | 0 | 6 | 6 |
| cps | 4 | 0 | 4 | 5 | 0 | 5 |
| datastream | 1 | 0 | 1 | 3 | 0 | 3 |
| dns | 2 | 0 | 2 | 2 | 1 | 3 |
| iam | 4 | 3 | 7 | 9 | 16 | 25 |
| imaging | 3 | 0 | 3 | 2 | 0 | 2 |
| mtlskeystore | 0 | 3 | 3 | 0 | 3 | 3 |
| mtlstruststore | 0 | 2 | 2 | 0 | 8 | 8 |
| networklists | 4 | 0 | 4 | 1 | 0 | 1 |
| cloudcertificates | 0 | 2 | 2 | 0 | 3 | 3 |
| **TOTALS** | **125** | **22** | **147** | **150** | **68** | **218** |

## Priority Classification

### Priority 1 — Target ≥90% Coverage

| Product | TechDocs Items | Implemented Items | Coverage % | Status |
|---|---|---|---|---|
| Application Security (appsec) | 107 | 111 | 100% | Target exceeded |
| Bot Manager (botman) | 53 | 58 | 100% | Target exceeded |
| Global Traffic Management (gtm) | 18 | 18 | 100% | Target met |
| Property (property) | 34 | 40 | 100% | Target exceeded |
| EdgeWorkers (edgeworkers) | 10 | 10 | 100% | Target met |

All Priority 1 products are at 100% coverage, exceeding the ≥90% target. Several products
(appsec, botman, property) have codebase implementations that exceed the TechDocs-documented
count due to internal utilities and legacy data sources.

### Priority 2 — Target ≥70% Coverage

| Product | TechDocs Items | Implemented Items | Coverage % | Status |
|---|---|---|---|---|
| Account Protection (accountprotection) | 8 | 8 | 100% | Target exceeded |
| API Definitions (apidefinitions) | 6 | 6 | 100% | Target exceeded |
| Cloud Access Manager (cloudaccess) | 5 | 5 | 100% | Target exceeded |
| Cloudlets (cloudlets) | 16 | 16 | 100% | Target exceeded |
| Cloud Wrapper (cloudwrapper) | 8 | 8 | 100% | Target exceeded |
| Certificates (cps) | 9 | 9 | 100% | Target exceeded |
| Edge DNS (dns) | 5 | 5 | 100% | Target exceeded |
| Identity and Access Management (iam) | 32 | 32 | 100% | Target exceeded |
| Image and Video Manager (imaging) | 5 | 5 | 100% | Target exceeded |
| mTLS Origin Keystore (mtlskeystore) | 6 | 6 | 100% | Target exceeded |
| mTLS Edge Truststore (mtlstruststore) | 10 | 10 | 100% | Target exceeded |
| Network Lists (networklists) | 5 | 5 | 100% | Target exceeded |

All Priority 2 products are at 100% coverage, exceeding the ≥70% target.

### Thin Subproviders — 100% Required

Products with fewer than 5 TechDocs items must achieve 100% coverage.

| Product | TechDocs Items | Threshold | Implemented Items | Coverage % | Status |
|---|---|---|---|---|---|
| Client Lists (clientlists) | 4 | <5 → 100% required | 4 | 100% | Requirement met |
| DataStream (datastream) | 4 | <5 → 100% required | 4 | 100% | Requirement met |

Both thin subproviders meet the 100% requirement.

> **Note:** Cloud Access Manager (cloudaccess) has exactly 5 TechDocs items and therefore does
> **not** qualify as a thin subprovider (the threshold is strictly fewer than 5). It is classified
> under Priority 2 rules.

## Deprecated Exclusions

The following deprecated items were identified in TechDocs v9.3 and excluded from coverage
calculations per the deprecation exclusion rules.

### `akamai_botman_challenge_interception_rules`

| Attribute | Detail |
|---|---|
| **Product** | Bot Manager (botman) |
| **Resource** | `akamai_botman_challenge_interception_rules` |
| **Data Source** | `akamai_botman_challenge_interception_rules` |
| **Replacement** | `akamai_botman_challenge_injection_rules` (resource and data source) |
| **Lifecycle** | Originally added → deprecated → **removed from codebase** (confirmed via CHANGELOG) |
| **TechDocs Status** | Still listed in the v9.3 sidebar with a deprecation notice; end-of-life scheduled for v10 |
| **Residual Artifacts** | `.tf` testdata files remain as removal artifacts in the `pkg/providers/botman/testdata/` directory, but no Go implementation or provider registration exists |
| **Exclusion Basis** | AAP Section 0.1.2 — deprecated resources/data sources are OUT OF SCOPE |

No other deprecated items were identified in the TechDocs v9.3 sidebar analysis.

## Codebase Extras

The following items are implemented in the codebase but are not documented in the TechDocs v9.3
sidebar. These represent internal utilities, legacy data sources, or additional functionality
beyond the TechDocs surface.

| Product | Item | Type | Likely Reason |
|---|---|---|---|
| appsec | `eval` | SDK Data Source | Internal evaluation data source |
| appsec | `export_configuration` | SDK Data Source | Configuration export utility |
| appsec | `rate_policy_actions` | SDK Data Source | Granular rate policy action listing |
| appsec | `custom_rules_usage` | Framework Data Source | Custom rules usage analytics |
| botman | `bot_analytics_cookie` (resource) | SDK Resource | Analytics cookie management |
| botman | `bot_analytics_cookie` (data source) | SDK Data Source | Analytics cookie management |
| botman | `bot_analytics_cookie_values` | SDK Data Source | Analytics cookie value listing |
| botman | `custom_code` (resource) | SDK Resource | Custom code injection |
| botman | `custom_code` (data source) | SDK Data Source | Custom code injection |
| property | `Bootstrap` | Framework Resource | Domain ownership bootstrap |
| property | `DomainOwnershipLateValidation` | Framework Resource | Late domain validation variant |
| property | `contract` | SDK Data Source | Legacy data source (no `property_` prefix) |
| property | `contracts` | SDK Data Source | Legacy data source (no `property_` prefix) |
| property | `group` | SDK Data Source | Legacy data source (no `property_` prefix) |
| property | `groups` | SDK Data Source | Legacy data source (no `property_` prefix) |

**Total codebase extras:** 15 items across 3 subproviders (appsec: 4, botman: 5, property: 6).

These extras account for the difference between the TechDocs total (345 items) and the codebase
total for the 19 TechDocs-listed subproviders (360 items). The remaining 5 items belong to the
Beta Cloud Certificate Manager subprovider (2 resources + 3 data sources), bringing the overall
codebase total to 365.

## Scope-Limiting Rules

The following scope-limiting rules were defined to cap implementation effort when gaps are large.
All rules are documented for completeness. **None were triggered** as all products are at 100%
coverage with 0 total gaps.

| Rule | Condition | Action | Status |
|---|---|---|---|
| **Per-product cap** | A single product has more than 30 gaps | Implement only the first 20 (by TechDocs listing order); document remainder as Phase 2 | **Not triggered** — no product has any gaps |
| **Global cap** | Total gaps across all products exceed 100 | Implement Priority 1 only (to ≥90% each); document Priority 2 as future work | **Not triggered** — 0 total gaps |
| **Priority 1 cap** | Priority 1 alone has more than 60 gaps | Implement the top 50 (CRUD resources first, then data sources); document rest in gap report | **Not triggered** — Priority 1 has 0 gaps |

## Conclusion

The Akamai Terraform Provider v9.3 achieves **100% coverage** of all non-deprecated
TechDocs-documented resources and data sources across all 19 listed subproviders. Every resource
and data source in the TechDocs v9.3 sidebar has a corresponding implementation registered in the
provider codebase.

**No new resource or data source implementations are required at this time.**

The codebase exceeds TechDocs coverage with 20 additional items (365 codebase total vs. 345
TechDocs total), comprising 15 internal utilities, legacy data sources, and extra functionality
across appsec, botman, and property, plus 5 items from the Beta Cloud Certificate Manager
subprovider.

### Recommendations

1. **Continue monitoring TechDocs** for newly documented resources and data sources that may
   require future implementation as the provider evolves beyond v9.3.
2. **Track Cloud Certificate Manager separately.** The `cloudcertificates` subprovider (Beta since
   v9.2.0) should be added to formal coverage tracking once it exits Beta status and appears in
   the TechDocs sidebar.
3. **Document codebase extras.** The 15 items implemented beyond TechDocs coverage should be
   reviewed to determine if they warrant TechDocs documentation or if they serve internal purposes
   only.
4. **Clean up deprecated artifacts.** Residual `.tf` testdata files from the removed
   `challenge_interception_rules` resource and data source remain in the botman testdata directory
   and could be cleaned up in a future maintenance pass.

## Risk Assessment

| Severity | Risk | Affected Systems | Mitigation | Rollback |
|---|---|---|---|---|
| **LOW** | Codebase extras not documented in TechDocs may confuse users who discover them via provider introspection or state inspection | End users, documentation consumers | Document these extras in TechDocs or mark them as internal/unsupported in provider documentation | N/A — documentation-only change |
| **LOW** | Beta subprovider (cloudcertificates) has no TechDocs coverage tracking, creating a blind spot for future gap analysis | Coverage tracking processes | Track cloudcertificates separately in future gap reports until it reaches GA status and is added to TechDocs | N/A — process change only |
| **LOW** | Deprecated `challenge_interception_rules` testdata artifacts may cause confusion during codebase audits | Developer experience, code maintainability | Remove residual `.tf` testdata files in a future maintenance release | N/A — cleanup-only change |
