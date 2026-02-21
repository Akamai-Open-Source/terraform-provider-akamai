# Technical Specification

# 0. Agent Action Plan

## 0.1 Intent Clarification


### 0.1.1 Core Feature Objective

Based on the prompt, the Blitzy platform understands that the new feature requirement is to **achieve complete Terraform resource and data source coverage across the Akamai provider**, ensuring the provider matches the current Akamai TechDocs surface for each subprovider. The specific requirements are:

- **Gap-Driven Discovery and Reporting**: Execute a systematic TechDocs vs. codebase comparison for every subprovider listed in `docs/resources/resources.md` and `docs/data-sources/data-sources.md`, producing a Markdown coverage gap report (`docs/coverage_gap_report_YYYYMMDD.md`) with per-product breakdowns of TechDocs-documented items, implemented items, coverage percentages, and gap status
- **Priority 1 Coverage ≥90%**: Application Security (`appsec`), Bot Manager (`botman`), Global Traffic Management (`gtm`), Property (`property`), and EdgeWorkers (`edgeworkers`) must reach at least 90% coverage of non-deprecated TechDocs-documented resources and data sources
- **Priority 2 Coverage ≥70%**: All remaining subproviders (`apidefinitions`, `cps`, `clientlists`, `cloudaccess`, `cloudcertificates`, `cloudwrapper`, `cloudlets`, `datastream`, `dns`, `iam`, `imaging`, `mtlstruststore`, `mtlskeystore`, `networklists`, `accountprotection`) must reach at least 70% coverage
- **Thin Subprovider 100% Coverage**: Any product with fewer than 5 resources plus data sources in TechDocs must have 100% of those items implemented
- **Scope-Limiting Rules**: Per-product cap of 20 implementations when gaps exceed 30; global cap of 100 total gap implementations; Priority 1 cap of 50 if Priority 1 alone exceeds 60 gaps
- **Backward Compatibility**: Zero breaking changes to existing resource/data source schemas, behavior, attributes, or test patterns

The following implicit requirements have been detected:

- **Baseline Capture**: Before any implementation, capture per-product registration counts from every `provider.go`, test coverage via `go test -cover`, and test pass counts — these are mandatory for validation gates
- **Pattern Fidelity**: New resources and data sources must exactly replicate the architectural pattern (SDK or Framework) already used by the target subprovider — no pattern mixing or innovation
- **Test Coverage ≥75%**: Every new resource and data source must have corresponding `*_test.go` files with at least one test case, targeting ≥75% line coverage for new code
- **API-Surface Auditing**: Beyond TechDocs, the codebase must be searched for EdgeGrid SDK API calls that create, update, read, or delete entities without corresponding Terraform resources or data sources

### 0.1.2 Special Instructions and Constraints

**Minimal Change Clause (Critical):**
Make only the changes that are absolutely necessary. Do not refactor, optimize, or modify existing code unless directly required. Isolate new code in dedicated files. Document all changes to existing files with clear comments.

**Architectural Requirements:**
- Determine each product's pattern by inspecting `pkg/providers/<product>/provider.go` — if `SDKResources()`/`SDKDataSources()` are non-empty, use the SDK pattern; if only `FrameworkResources()`/`FrameworkDataSources()` are populated, use the Framework pattern
- Do not introduce new patterns; copy structure and style from the existing reference files in each product
- Register new resources/data sources only in the product's `provider.go` — do not change top-level provider wiring, plugin registration, or other subproviders

**Backward Compatibility:**
- Zero changes to existing resource/data source schemas or behavior
- Zero changes to public exports, subprovider interface methods, or client usage patterns
- All existing tests must continue to pass; post-implementation test pass count must match or exceed baseline

**TechDocs as Source of Truth:**
- `docs/resources/resources.md` and `docs/data-sources/data-sources.md` are the authoritative indexes
- TechDocs pages linked from those indexes define what resources and data sources should exist
- Do not invent resources or data sources not documented in TechDocs or implied by existing API usage

**Scope-Limiting Rules (Mandatory):**
- Per product: If a single product has more than 30 gaps, implement only the first 20 (by TechDocs listing order); document the remainder as Phase 2
- Global: If total gaps across all products exceed 100, implement Priority 1 only (to ≥90% each) and document Priority 2 as future work
- Global: If Priority 1 alone has more than 60 gaps, implement the top 50; document the rest in the gap report

**Deprecated/Beta Exclusions:**
- Resources or data sources marked deprecated in TechDocs are OUT OF SCOPE (e.g., `akamai_botman_challenge_interception_rules` — deprecated with end-of-life scheduled for v10, replaced by `challenge_injection_rules`)
- BETA/preview resources are OUT OF SCOPE unless in Priority 1 and not deprecated

### 0.1.3 Technical Interpretation

These feature requirements translate to the following technical implementation strategy:

- **To execute TechDocs & Gap Discovery**, we will parse every TechDocs page linked from `docs/resources/resources.md` and `docs/data-sources/data-sources.md`, list each documented resource and data source as the "TechDocs set," then compare against the "implemented set" extracted from each product's `provider.go` registration methods (`SDKResources()`, `FrameworkResources()`, `SDKDataSources()`, `FrameworkDataSources()`). Gaps are items in TechDocs set but not in the implemented set.
- **To produce the coverage gap report**, we will create `docs/coverage_gap_report_YYYYMMDD.md` with a Markdown table per the specified columns: Product, Resources (TechDocs), Resources (Implemented), Data Sources (TechDocs), Data Sources (Implemented), Coverage %, Status. Every gap will be categorized as Implemented, Deferred (with reason), or API-unavailable.
- **To implement missing resources/data sources** (if any are found), we will create new Go files under `pkg/providers/<product>/` following the exact naming convention (`resource_akamai_<product>_<name>.go`, `data_akamai_<product>_<name>.go`), register them in the product's `provider.go`, create corresponding `*_test.go` files with `testdata/` fixtures, and follow the product's established client access and error handling patterns.
- **To capture baselines**, we will run `go test -cover ./pkg/providers/...` and `go test ./pkg/providers/...` before implementation, record per-product registration counts, and save outputs for post-implementation comparison.
- **To validate backward compatibility**, we will confirm that all existing tests pass, no resource/data source schemas are modified, and the test pass count matches or exceeds the baseline.

**Critical Finding from Gap Analysis:** Comprehensive analysis of the TechDocs v9.3 sidebar against all 20 subprovider `provider.go` registrations reveals that **all 19 TechDocs-listed subproviders are currently at 100% coverage** of non-deprecated resources and data sources. The codebase implements 360 total resources/data sources against 345 effective TechDocs items (excluding 2 deprecated botman items). Several subproviders exceed TechDocs coverage with additional resources/data sources (appsec: 111 vs 107; botman: 58 vs 53; property: 40 vs 34). The 20th subprovider, Cloud Certificate Manager (`cloudcertificates`), is Beta and not listed in TechDocs v9.3 sidebar. This means the primary deliverable is the **formal coverage gap report** documenting this achieved state, with minimal or zero new resource/data source implementations required.


## 0.2 Repository Scope Discovery


### 0.2.1 Comprehensive File Analysis

**Repository Root Structure:**
The repository is `github.com/akamai/terraform-provider-akamai/v9`, a Go module (Go 1.24.10) using the Mozilla Public License 2.0. The provider supports Terraform Plugin Protocol v6 via `terraform-plugin-mux`, bridging both the older `terraform-plugin-sdk/v2` (SDK pattern) and the newer `terraform-plugin-framework` (Framework pattern).

**Subprovider Registration Architecture:**
All 20 subproviders are registered via a blank-import pattern in `pkg/providers/providers.go`, which triggers each subprovider's `init()` hook calling `registry.RegisterSubprovider()`. Each subprovider implements the `subprovider.Subprovider` interface with four registration methods:
- `SDKResources() map[string]*schema.Resource`
- `SDKDataSources() map[string]*schema.Resource`
- `FrameworkResources() []func() resource.Resource`
- `FrameworkDataSources() []func() datasource.DataSource`

**Complete Subprovider Inventory — Existing Files (provider.go per product):**

| Subprovider | Package Path | Pattern | SDK Res | SDK DS | FW Res | FW DS | Total | Lines (non-test) |
|---|---|---|---|---|---|---|---|---|
| accountprotection | `pkg/providers/accountprotection/` | SDK | 4 | 4 | 0 | 0 | 8 | 1,480 |
| apidefinitions | `pkg/providers/apidefinitions/` | Framework | 0 | 0 | 3 | 3 | 6 | 2,609 |
| appsec | `pkg/providers/appsec/` | Mixed (SDK-dominant) | 54 | 54 | 1 | 2 | 111 | 23,555 |
| botman | `pkg/providers/botman/` | SDK | 26 | 32 | 0 | 0 | 58 | 8,656 |
| clientlists | `pkg/providers/clientlists/` | Mixed | 2 | 0 | 0 | 2 | 4 | 2,098 |
| cloudaccess | `pkg/providers/cloudaccess/` | Framework | 0 | 0 | 1 | 4 | 5 | 2,057 |
| cloudcertificates | `pkg/providers/cloudcertificates/` | Framework | 0 | 0 | 2 | 3 | 5 | 2,097 |
| cloudlets | `pkg/providers/cloudlets/` | Mixed | 4 | 10 | 0 | 2 | 16 | 7,789 |
| cloudwrapper | `pkg/providers/cloudwrapper/` | Framework | 0 | 0 | 2 | 6 | 8 | 2,554 |
| cps | `pkg/providers/cps/` | SDK | 4 | 5 | 0 | 0 | 9 | 4,469 |
| datastream | `pkg/providers/datastream/` | SDK | 1 | 3 | 0 | 0 | 4 | 3,089 |
| dns | `pkg/providers/dns/` | Mixed | 2 | 2 | 0 | 1 | 5 | 4,879 |
| edgeworkers | `pkg/providers/edgeworkers/` | SDK | 4 | 6 | 0 | 0 | 10 | 3,002 |
| gtm | `pkg/providers/gtm/` | Mixed (FW-dominant) | 7 | 3 | 0 | 8 | 18 | 9,975 |
| iam | `pkg/providers/iam/` | Mixed | 4 | 9 | 3 | 16 | 32 | 8,729 |
| imaging | `pkg/providers/imaging/` | SDK | 3 | 2 | 0 | 0 | 5 | 6,737 |
| mtlskeystore | `pkg/providers/mtlskeystore/` | Framework | 0 | 0 | 3 | 3 | 6 | 3,459 |
| mtlstruststore | `pkg/providers/mtlstruststore/` | Framework | 0 | 0 | 2 | 8 | 10 | 4,351 |
| networklists | `pkg/providers/networklists/` | SDK | 4 | 1 | 0 | 0 | 5 | 1,595 |
| property | `pkg/providers/property/` | Mixed | 6 | 19 | 5 | 10 | 40 | 309,129 |
| **TOTALS** | | | **125** | **150** | **22** | **68** | **365** | **404,309** |

**TechDocs vs. Codebase Coverage Gap Analysis:**

The TechDocs v9.3 sidebar was systematically parsed for all 19 documented subproviders (Cloud Certificate Manager is absent from TechDocs — Beta product). Deprecated items (botman `challenge_interception_rules`: 1 resource + 1 data source) were excluded. The complete gap report:

| Product | Priority | TechDocs (Res) | TechDocs (DS) | Effective Total | Codebase Total | Coverage % | Gaps | Status |
|---|---|---|---|---|---|---|---|---|
| appsec | P1 | 55 | 52 | 107 | 111 | 100% | 0 | Met |
| botman | P1 | 24 | 29 | 53 | 58 | 100% | 0 | Met |
| gtm | P1 | 7 | 11 | 18 | 18 | 100% | 0 | Met |
| property | P1 | 9 | 25 | 34 | 40 | 100% | 0 | Met |
| edgeworkers | P1 | 4 | 6 | 10 | 10 | 100% | 0 | Met |
| accountprotection | P2 | 4 | 4 | 8 | 8 | 100% | 0 | Met |
| apidefinitions | P2 | 3 | 3 | 6 | 6 | 100% | 0 | Met |
| cloudaccess | P2 | 1 | 4 | 5 | 5 | 100% | 0 | Met |
| cloudlets | P2 | 4 | 12 | 16 | 16 | 100% | 0 | Met |
| cloudwrapper | P2 | 2 | 6 | 8 | 8 | 100% | 0 | Met |
| cps | P2 | 4 | 5 | 9 | 9 | 100% | 0 | Met |
| dns | P2 | 2 | 3 | 5 | 5 | 100% | 0 | Met |
| iam | P2 | 7 | 25 | 32 | 32 | 100% | 0 | Met |
| imaging | P2 | 3 | 2 | 5 | 5 | 100% | 0 | Met |
| mtlskeystore | P2 | 3 | 3 | 6 | 6 | 100% | 0 | Met |
| mtlstruststore | P2 | 2 | 8 | 10 | 10 | 100% | 0 | Met |
| networklists | P2 | 4 | 1 | 5 | 5 | 100% | 0 | Met |
| clientlists | Thin | 2 | 2 | 4 | 4 | 100% | 0 | Met |
| datastream | Thin | 1 | 3 | 4 | 4 | 100% | 0 | Met |
| cloudcertificates | N/A | — | — | — | 5 | N/A | — | Beta (OOS) |
| **TOTALS** | | **141** | **204** | **345** | **365** | **100%** | **0** | **All Met** |

**Codebase Extras (items implemented but not in TechDocs):**

| Product | Extra Item | Type | Likely Reason |
|---|---|---|---|
| appsec | `eval` (DS) | SDK DS | Internal evaluation data source |
| appsec | `export_configuration` (DS) | SDK DS | Configuration export utility |
| appsec | `rate_policy_actions` (DS) | SDK DS | Granular rate policy action listing |
| appsec | `custom_rules_usage` (DS) | FW DS | Custom rules usage analytics |
| botman | `bot_analytics_cookie` (Res + DS) | SDK | Analytics cookie management |
| botman | `bot_analytics_cookie_values` (DS) | SDK DS | Analytics cookie value listing |
| botman | `custom_code` (Res + DS) | SDK | Custom code injection |
| property | `Bootstrap` (Res) | FW Res | Domain ownership bootstrap |
| property | `DomainOwnershipLateValidation` (Res) | FW Res | Late domain validation variant |
| property | `contract`, `contracts`, `group`, `groups` (DS) | SDK DS | Legacy data sources (no `property_` prefix) |

**Deprecated Item — Complete Lifecycle (Confirmed via CHANGELOG):**

The `akamai_botman_challenge_interception_rules` resource and data source was originally added to the provider, later deprecated in favor of `challenge_injection_rules`, and then **removed** from the codebase entirely. The CHANGELOG confirms:
- **Added**: Original implementation with read, update, import, and list operations
- **Deprecated**: `akamai_botman_challenge_interception_rules` data source and resource; use `challenge_injection_rules` instead
- **Removed**: "Removed the deprecated `akamai_botman_challenge_interception_rules` data source and resource"

Residual `.tf` testdata files remain as removal artifacts but no Go implementation or provider registration exists. TechDocs v9.3 still lists the item in the sidebar with a deprecation notice. This item is **OUT OF SCOPE** per the user's explicit exclusion of deprecated resources.

**Integration Point Discovery:**

- **Provider Registration**: `pkg/providers/<product>/provider.go` — the sole integration point for new resources/data sources
- **API Client Access**: `pkg/meta/meta.go` (`meta.Must(m)`) → `inst.Client(meta)` (SDK products) or package-level `Client()` set in `Configure` (Framework products)
- **Shared Utilities**: `pkg/common/tf/` (Terraform helpers), `pkg/common/str/` (string utilities), `pkg/meta/` (meta/session management)
- **Test Infrastructure**: `testutils.LoadFixtureBytes()`, mock clients, `terraform-plugin-testing/helper/resource`
- **EdgeGrid SDK**: `github.com/akamai/AkamaiOPEN-edgegrid-golang/v12` — all Akamai API interfaces

### 0.2.2 Web Search Research Conducted

- **TechDocs v9.3 Sidebar Parsing**: Fetched and parsed the complete Akamai TechDocs v9.3 documentation sidebar from `https://techdocs.akamai.com/terraform/v9.3/docs/botman-resources` and `https://techdocs.akamai.com/terraform/v9.3/docs/ccm-resources`, extracting all documented resources and data sources across 19 subprovider sections (Cloud Certificate Manager has no dedicated section — confirmed Beta)
- **Botman Challenge Interception Rules**: Verified via TechDocs that `challenge_interception_rules` is listed with a deprecation notice, scheduled for removal in v10, replaced by `challenge_injection_rules`
- **Cloud Certificate Manager (ccm-resources)**: Confirmed that the CCM resources page renders a generic "Resources" title with no product-specific listings in the sidebar, confirming Beta status and absence from the TechDocs navigation

### 0.2.3 New File Requirements

Based on the gap analysis showing 100% coverage across all TechDocs-listed subproviders, the following new files are required:

**Gap Report (Mandatory First Deliverable):**
- `docs/coverage_gap_report_YYYYMMDD.md` — Formal coverage gap report with per-product Markdown table: Product | Resources (TechDocs) | Resources (Implemented) | Data Sources (TechDocs) | Data Sources (Implemented) | Coverage % | Status

**New Resource/Data Source Files (Conditional — only if gaps are discovered during formal validation):**
- `pkg/providers/<product>/resource_akamai_<product>_<name>.go` — New resource implementation following product's SDK or Framework pattern
- `pkg/providers/<product>/data_akamai_<product>_<name>.go` — New data source implementation
- `pkg/providers/<product>/resource_akamai_<product>_<name>_test.go` — Unit tests with at least one test case
- `pkg/providers/<product>/data_akamai_<product>_<name>_test.go` — Data source tests
- `pkg/providers/<product>/testdata/TestRes<Name>/` — Test fixture directory with `.tf` and `.json` files
- `pkg/providers/<product>/testdata/TestDS<Name>/` — Data source test fixture directory

**No new configuration files, middleware, or shared utility files are anticipated** given that:
- All TechDocs-documented resources/data sources are already implemented
- The gap report is the primary new artifact
- Any discovered gaps will use existing patterns and infrastructure exclusively


## 0.3 Dependency Inventory


### 0.3.1 Private and Public Packages

All packages are sourced from `go.mod` at the repository root. No new dependencies are required — all new code must use existing packages already in the dependency graph.

| Registry | Package | Version | Purpose |
|---|---|---|---|
| github.com | `akamai/AkamaiOPEN-edgegrid-golang/v12` | v12.3.0 | Akamai API client SDK — all API interfaces for every subprovider |
| github.com | `hashicorp/terraform-plugin-framework` | v1.17.0 | Terraform plugin framework for newer (Framework pattern) resources/data sources |
| github.com | `hashicorp/terraform-plugin-sdk/v2` | v2.38.1 | Terraform plugin SDK for older (SDK pattern) resources/data sources |
| github.com | `hashicorp/terraform-plugin-mux` | v0.21.0 | Protocol multiplexer bridging SDK and Framework servers on Plugin Protocol v6 |
| github.com | `hashicorp/terraform-plugin-go` | v0.28.0 | Low-level Terraform plugin protocol Go bindings |
| github.com | `hashicorp/terraform-plugin-testing` | v1.12.0 | Acceptance/unit testing framework for Terraform plugins |
| github.com | `hashicorp/go-hclog` | v1.6.3 | Structured logging used across all subproviders |
| github.com | `hashicorp/go-cty` | v1.4.1-0.20200414143053-d3edf31b6320 | Type system for Terraform value marshaling |
| github.com | `stretchr/testify` | v1.10.0 | Test assertions (`require`, `assert`, `mock` packages) |
| github.com | `tj/assert` | v0.0.3 | Additional test assertion library |
| github.com | `jedib0t/go-pretty/v6` | v6.6.7 | Table formatting for CLI output |
| github.com | `google/go-cmp` | v0.7.0 | Deep comparison for test assertions |

**EdgeGrid SDK Package Imports per Subprovider:**

Each subprovider imports its dedicated API package from the EdgeGrid SDK. Cross-package imports indicate inter-product dependencies:

| Subprovider | Primary SDK Package | Additional SDK Imports | Cross-Product Deps |
|---|---|---|---|
| accountprotection | `appsec` | log, session | Shares appsec API client |
| apidefinitions | `apidefinitions` | (none) | None |
| appsec | `appsec` | log, session | None |
| botman | `botman` | log, session | **appsec** (getLatestConfigVersion) |
| clientlists | `clientlists` | log, session | None |
| cloudaccess | `cloudaccess` | (none) | None |
| cloudcertificates | `cloudcertificates` | (none) | None |
| cloudlets | `cloudlets` | log, session | None |
| cloudwrapper | `cloudwrapper` | (none) | None |
| cps | `cps` | log, session | None |
| datastream | `datastream` | log, session | None |
| dns | `dns` | log, session | None |
| edgeworkers | `edgeworkers` | log, session | None |
| gtm | `gtm` | log, session | None |
| iam | `iam` | log, session | **papi** (property users) |
| imaging | `imaging` | log, session | None |
| mtlskeystore | `mtlskeystore` | (none) | None |
| mtlstruststore | `mtlstruststore` | (none) | None |
| networklists | `networklists` | log, session | None |
| property | `papi` | log, session | **domainownership**, **hapi**, **iam**, **ptr** |

### 0.3.2 Dependency Updates

**No new dependencies are required.** All implementation work uses the existing dependency set from `go.mod`. The minimal change clause explicitly prohibits adding new dependencies unless required by three or more new resources in the same product.

**Import Updates for New Files (If Gaps Are Found):**

New resource/data source files will require the following imports, matching the pattern established by existing files in the same product:

- **SDK pattern files** (`pkg/providers/<product>/resource_akamai_<product>_<name>.go`):
  - `github.com/akamai/AkamaiOPEN-edgegrid-golang/v12/pkg/<sdk_package>`
  - `github.com/akamai/terraform-provider-akamai/v9/pkg/meta`
  - `github.com/hashicorp/terraform-plugin-sdk/v2/helper/schema`
  - `github.com/hashicorp/terraform-plugin-sdk/v2/diag`
  - `github.com/akamai/terraform-provider-akamai/v9/pkg/common/tf`

- **Framework pattern files** (`pkg/providers/<product>/resource_akamai_<product>_<name>.go`):
  - `github.com/akamai/AkamaiOPEN-edgegrid-golang/v12/pkg/<sdk_package>`
  - `github.com/akamai/terraform-provider-akamai/v9/pkg/meta`
  - `github.com/hashicorp/terraform-plugin-framework/resource`
  - `github.com/hashicorp/terraform-plugin-framework/resource/schema`
  - `github.com/hashicorp/terraform-plugin-framework/types`

- **Test files** (`pkg/providers/<product>/*_test.go`):
  - `github.com/hashicorp/terraform-plugin-testing/helper/resource`
  - `github.com/stretchr/testify/require`
  - Product-specific test utilities and mock packages

**External Reference Updates:**

- `pkg/providers/<product>/provider.go` — Add new entries to the appropriate registration map/slice only (no structural changes)
- `docs/coverage_gap_report_YYYYMMDD.md` — New documentation file (not a code dependency)
- No changes to `go.mod`, `go.sum`, `.github/workflows/`, `Dockerfile`, or CI/CD configuration files


## 0.4 Integration Analysis


### 0.4.1 Existing Code Touchpoints

**Registration Integration — provider.go (Per Product):**

Every new resource or data source integrates into the codebase through exactly one touchpoint per product: the `provider.go` file in that product's package. No other files in the provider wiring chain require modification.

| Integration Point | File Pattern | Change Type | Description |
|---|---|---|---|
| SDK Resource Registration | `pkg/providers/<product>/provider.go` → `SDKResources()` | Add map entry | New `"akamai_<product>_<name>": resource<Name>()` entry in the returned `map[string]*schema.Resource` |
| SDK Data Source Registration | `pkg/providers/<product>/provider.go` → `SDKDataSources()` | Add map entry | New `"akamai_<product>_<name>": dataSource<Name>()` entry in the returned `map[string]*schema.Resource` |
| Framework Resource Registration | `pkg/providers/<product>/provider.go` → `FrameworkResources()` | Add slice entry | New `New<Name>Resource` constructor appended to the returned `[]func() resource.Resource` |
| Framework Data Source Registration | `pkg/providers/<product>/provider.go` → `FrameworkDataSources()` | Add slice entry | New `New<Name>DataSource` constructor appended to the returned `[]func() datasource.DataSource` |

**Client Access Patterns — Per Product:**

New resources/data sources must obtain API clients using the exact pattern established by existing code in the target product. Two distinct patterns exist:

- **Handler-level client (SDK pattern, majority of products):**
  ```go
  meta := meta.Must(m)
  client := inst.Client(meta)
  ```
  Used by: `appsec`, `botman`, `cps`, `edgeworkers`, `networklists`, `imaging`, `datastream`, `accountprotection`, `clientlists`, `cloudlets`, `dns`, `gtm`, `property` (SDK resources)

- **Package-level client (Framework pattern):**
  ```go
  client = <package>.Client(metaConfig.Session())
  ```
  Set in the struct's `Configure()` method and stored as a package-level variable. Used by: `apidefinitions`, `cloudaccess`, `cloudcertificates`, `cloudwrapper`, `mtlskeystore`, `mtlstruststore`, `iam` (Framework resources), `property` (Framework resources)

**Shared Utility Dependencies:**

New files consume shared utilities but do not modify them:

| Utility Package | Path | Functions Used | By Pattern |
|---|---|---|---|
| `pkg/meta` | Meta provider data | `meta.Must(m)`, `meta.Meta` interface | Both |
| `pkg/common/tf` | Terraform helpers | `tf.ErrValueSet`, `tf.ErrNotFound`, `tf.SetAttrs()` | SDK |
| `pkg/common/str` | String utilities | `str.AddPrefix()`, string manipulation helpers | Both |
| `pkg/logger` | Logging | `logger.Get()`, structured logging | SDK |

**Error Handling Integration:**

New code must follow the error handling pattern of its target product — the two patterns must never be mixed:

- **SDK pattern**: `return diag.FromErr(err)` for API errors; `return diag.Errorf(...)` for formatted errors; `errors.Is(err, tf.ErrNotFound)` for optional attribute checks
- **Framework pattern**: `resp.Diagnostics.AddError("Title", "Description")` for all errors; `resp.Diagnostics.Append(req.Plan.Get(ctx, &data)...)` for state operations

**Test Infrastructure Integration:**

New test files integrate with the existing test infrastructure without modifying it:

| Component | Path / Package | Integration Method |
|---|---|---|
| Fixture loading | `testutils.LoadFixtureBytes(t, "testdata/...")` | Load JSON/TF fixtures from `testdata/` subdirectories |
| Test step runner | `resource.Test(t, resource.TestCase{...})` | Standard `terraform-plugin-testing` test runner |
| Mock clients | Product-specific mock packages under `pkg/providers/<product>/` | Mock API client interfaces for unit tests |
| Provider factory | `testutils.ProviderTestChecked(t)` or product-specific `ProviderTest` | Register provider for test execution |

**Database/Schema Updates:**

No database or schema changes are required. Terraform providers are stateless; all state is managed by the Terraform state file. New resources/data sources define schemas declaratively in their Go source files and read/write to the Akamai API via the EdgeGrid SDK.

**Cross-Product Dependencies (Existing — Do Not Modify):**

- `botman` → `appsec`: Imports `getLatestConfigVersion()` and `getModifiableConfigVersion()` from the appsec package for configuration version resolution
- `iam` → `papi`: Uses the PAPI SDK package for property-related user operations
- `property` → `domainownership`, `hapi`, `iam`, `ptr`: Multi-SDK-package integration for domain validation, edge hostname management, user context, and pointer records
- `accountprotection` → `appsec`: Shares the appsec API client interface

These cross-product dependencies are read-only context for this feature; no modifications are permitted unless a new resource explicitly requires cross-product API calls (not anticipated given 100% current coverage).


## 0.5 Technical Implementation


### 0.5.1 File-by-File Execution Plan

The implementation is structured into three groups reflecting the mandatory execution order: baseline capture and gap discovery first, then conditional resource implementation, then validation and documentation.

**Group 1 — Baseline Capture and Gap Report (Mandatory First Deliverable):**

Every file listed below MUST be created or verified.

| Action | File | Purpose |
|---|---|---|
| RUN | `pkg/providers/*/provider.go` | Extract registration counts for all 20 subproviders from `SDKResources()`, `FrameworkResources()`, `SDKDataSources()`, `FrameworkDataSources()` |
| RUN | `go test -cover ./pkg/providers/...` | Capture baseline test coverage percentages per package |
| RUN | `go test ./pkg/providers/...` | Capture baseline test pass/fail counts |
| READ | `docs/resources/resources.md` | Parse subprovider index table — extract TechDocs URLs for each product's resources |
| READ | `docs/data-sources/data-sources.md` | Parse subprovider index table — extract TechDocs URLs for each product's data sources |
| FETCH | TechDocs pages (20 resource URLs + 20 data source URLs) | List every resource and data source documented per subprovider — this is the "TechDocs set" |
| COMPARE | TechDocs set vs. implemented set per product | Compute gaps: items in TechDocs but not in `provider.go` registrations |
| CREATE | `docs/coverage_gap_report_YYYYMMDD.md` | Formal gap report with columns: Product, Resources (TechDocs), Resources (Implemented), Data Sources (TechDocs), Data Sources (Implemented), Coverage %, Status |

**Group 2 — Conditional Resource/Data Source Implementation:**

Based on the comprehensive gap analysis already performed, all 19 TechDocs-listed subproviders are at 100% coverage of non-deprecated items. The following conditional implementation applies only if the formal TechDocs page parsing at build time reveals gaps not detected in this analysis:

- **If gaps are found in Priority 1 products** (appsec, botman, gtm, property, edgeworkers):
  - CREATE: `pkg/providers/<product>/resource_akamai_<product>_<name>.go` — New resource implementing the SDK or Framework pattern per that product
  - CREATE: `pkg/providers/<product>/data_akamai_<product>_<name>.go` — New data source matching product pattern
  - CREATE: `pkg/providers/<product>/*_test.go` — Test files with at least one test case per resource/data source
  - CREATE: `pkg/providers/<product>/testdata/TestRes<Name>/` — `.tf` config and `.json` API response fixtures
  - CREATE: `pkg/providers/<product>/testdata/TestDS<Name>/` — `.tf` config and `.json` fixtures for data sources
  - MODIFY: `pkg/providers/<product>/provider.go` — Add registration entries only

- **If gaps are found in Priority 2 products** (all other subproviders):
  - Same file creation pattern as Priority 1, subject to scope-limiting rules (max 20 per product if >30 gaps, global cap of 100)

- **If gaps are found in thin subproviders** (products with <5 TechDocs items):
  - Implement 100% of gaps using the same file creation pattern

**SDK Pattern — New Resource File Template:**

```go
func resource<Name>() *schema.Resource {
  return &schema.Resource{
    CreateContext: resourceCreate,
    ReadContext:   resourceRead,
```

**Framework Pattern — New Resource File Template:**

```go
type <Name>Resource struct {
  meta meta.Meta
}
func (r *<Name>Resource) Metadata(...) {
```

**Group 3 — Tests, Documentation, and Validation:**

| Action | File | Purpose |
|---|---|---|
| CREATE | `pkg/providers/<product>/*_test.go` (per new resource/DS) | Unit tests importing production code, using mock clients and `testdata/` fixtures |
| CREATE | `pkg/providers/<product>/testdata/Test*/*.tf` | Terraform config fixtures for test cases |
| CREATE | `pkg/providers/<product>/testdata/Test*/*.json` | Mock API response JSON fixtures |
| UPDATE | `docs/coverage_gap_report_YYYYMMDD.md` | Finalize with post-implementation counts: Before count, After count, Coverage %, Status (Met/Deferred) |
| RUN | `go build ./pkg/providers/...` | Verify zero compilation errors |
| RUN | `go test ./pkg/providers/...` | Verify all tests pass, pass count ≥ baseline |
| RUN | `go test -cover ./pkg/providers/...` | Verify new code coverage ≥ 75% |

### 0.5.2 Implementation Approach per File

**Step 1 — Establish Formal Baseline:**
Before any code changes, execute baseline capture commands to record per-product registration counts, test coverage percentages, and test pass/fail counts. Store these results for post-implementation comparison against validation gates.

**Step 2 — Execute TechDocs Gap Discovery:**
For each of the 20 subproviders listed in `docs/resources/resources.md` and `docs/data-sources/data-sources.md`, fetch the linked TechDocs page and extract every documented resource and data source name. Compare against the implemented set from each product's `provider.go`. Categorize every gap as: Implemented (to be built), Deferred (with reason), or API-unavailable. Apply scope-limiting rules: per-product cap of 20 if >30 gaps, global cap of 100, Priority 1 cap of 50 if >60 gaps.

**Step 3 — Produce the Coverage Gap Report:**
Create `docs/coverage_gap_report_YYYYMMDD.md` as a structured Markdown document with:
- Summary table: Product | Resources (TechDocs) | Resources (Implemented) | Data Sources (TechDocs) | Data Sources (Implemented) | Coverage % | Status
- Per-product detail sections listing each gap with its categorization
- Scope-limiting rule application notes
- Priority classification (P1/P2/Thin) per product

**Step 4 — Implement Missing Resources/Data Sources (If Any):**
For each gap categorized as "Implemented," create new Go files following the exact pattern of existing resources in that product:
- Determine pattern (SDK vs Framework) from the product's `provider.go`
- Copy the structure of an existing resource/data source in the same product as a template
- Implement CRUD operations using the same client access pattern
- Map TechDocs attributes to schema fields: required → `Required: true`, optional → `Optional: true`, read-only → `Computed: true`
- Add inline comments documenting which TechDocs page/section the resource aligns with
- Register in `provider.go` using the appropriate method

**Step 5 — Create Tests for New Code:**
For each new resource/data source:
- Create `*_test.go` in the same directory, importing the production resource function
- Use mock API clients matching the product's existing mock pattern
- Create `testdata/` fixtures (`.tf` configs and `.json` API responses)
- Implement at minimum: create, read, update, delete tests for resources; read test for data sources
- Target ≥75% line coverage for new code

**Step 6 — Validate and Finalize:**
- Run `go build ./pkg/providers/...` — zero compilation errors required
- Run `go test ./pkg/providers/...` — all tests must pass, count ≥ baseline
- Run `go test -cover ./pkg/providers/...` — new code coverage ≥ 75%
- Update the gap report with post-implementation metrics
- Verify no existing resource/data source schemas or behaviors were modified

**Critical Implementation Finding:**
Based on the definitive TechDocs v9.3 sidebar analysis against all 20 subprovider registrations, **no new resource or data source implementations are anticipated**. All 19 TechDocs-documented subproviders are at 100% coverage. The primary implementation deliverable is the formal coverage gap report and baseline validation. Steps 4 and 5 are conditional — they execute only if the formal build-time TechDocs parsing reveals previously undetected gaps.


## 0.6 Scope Boundaries


### 0.6.1 Exhaustively In Scope

**Mandatory Deliverables (All Products):**

| Category | File Pattern | Purpose |
|---|---|---|
| Gap Report | `docs/coverage_gap_report_YYYYMMDD.md` | Formal TechDocs vs. codebase coverage report — mandatory first deliverable |
| Baseline Data | Output of `go test -cover ./pkg/providers/...` | Pre-implementation test coverage baseline |
| Baseline Data | Output of `go test ./pkg/providers/...` | Pre-implementation test pass/fail counts |
| Registration Audit | `pkg/providers/*/provider.go` | Read-only audit of all 20 subprovider registration methods |

**Priority 1 Subproviders — Target ≥90% Coverage:**

| Subprovider | Package Path | Current Items | TechDocs Items | Current Coverage | Files In Scope |
|---|---|---|---|---|---|
| appsec | `pkg/providers/appsec/` | 111 | 107 | 100% (exceeds) | `provider.go` (audit only) |
| botman | `pkg/providers/botman/` | 58 | 53 | 100% (exceeds) | `provider.go` (audit only) |
| gtm | `pkg/providers/gtm/` | 18 | 18 | 100% | `provider.go` (audit only) |
| property | `pkg/providers/property/` | 40 | 34 | 100% (exceeds) | `provider.go` (audit only) |
| edgeworkers | `pkg/providers/edgeworkers/` | 10 | 10 | 100% | `provider.go` (audit only) |

**Priority 2 Subproviders — Target ≥70% Coverage:**

| Subprovider | Package Path | Current Items | TechDocs Items | Current Coverage | Files In Scope |
|---|---|---|---|---|---|
| accountprotection | `pkg/providers/accountprotection/` | 8 | 8 | 100% | `provider.go` (audit only) |
| apidefinitions | `pkg/providers/apidefinitions/` | 6 | 6 | 100% | `provider.go` (audit only) |
| clientlists | `pkg/providers/clientlists/` | 4 | 4 | 100% | `provider.go` (audit only) |
| cloudaccess | `pkg/providers/cloudaccess/` | 5 | 5 | 100% | `provider.go` (audit only) |
| cloudlets | `pkg/providers/cloudlets/` | 16 | 16 | 100% | `provider.go` (audit only) |
| cloudwrapper | `pkg/providers/cloudwrapper/` | 8 | 8 | 100% | `provider.go` (audit only) |
| cps | `pkg/providers/cps/` | 9 | 9 | 100% | `provider.go` (audit only) |
| datastream | `pkg/providers/datastream/` | 4 | 4 | 100% | `provider.go` (audit only) |
| dns | `pkg/providers/dns/` | 5 | 5 | 100% | `provider.go` (audit only) |
| edgeworkers | `pkg/providers/edgeworkers/` | 10 | 10 | 100% | `provider.go` (audit only) |
| iam | `pkg/providers/iam/` | 32 | 32 | 100% | `provider.go` (audit only) |
| imaging | `pkg/providers/imaging/` | 5 | 5 | 100% | `provider.go` (audit only) |
| mtlskeystore | `pkg/providers/mtlskeystore/` | 6 | 6 | 100% | `provider.go` (audit only) |
| mtlstruststore | `pkg/providers/mtlstruststore/` | 10 | 10 | 100% | `provider.go` (audit only) |
| networklists | `pkg/providers/networklists/` | 5 | 5 | 100% | `provider.go` (audit only) |

**Thin Subproviders — Target 100% Coverage:**

| Subprovider | TechDocs Items | Threshold | Current Coverage | Status |
|---|---|---|---|---|
| clientlists | 4 | <5 → 100% required | 100% | Met |
| datastream | 4 | <5 → 100% required | 100% | Met |
| cloudaccess | 5 | ≥5 → not thin | N/A | Standard P2 rules apply |

**Conditional Implementation Files (Only If Gaps Discovered During Formal Validation):**

- New resource files: `pkg/providers/<product>/resource_akamai_<product>_<name>.go`
- New data source files: `pkg/providers/<product>/data_akamai_<product>_<name>.go`
- New test files: `pkg/providers/<product>/*_test.go`
- New test fixtures: `pkg/providers/<product>/testdata/TestRes<Name>/*.tf`, `*.json`
- New test fixtures: `pkg/providers/<product>/testdata/TestDS<Name>/*.tf`, `*.json`
- Modified registrations: `pkg/providers/<product>/provider.go` (add entries only)

**Documentation In Scope:**

- `docs/coverage_gap_report_YYYYMMDD.md` — Mandatory gap report
- Per-resource/data-source docs only if the repo already uses per-resource doc files (check `docs/resources/` and `docs/data-sources/` for existing per-resource `.md` files beyond the index)

### 0.6.2 Explicitly Out of Scope

**Excluded Subprovider:**

| Subprovider | Package Path | Reason |
|---|---|---|
| cloudcertificates | `pkg/providers/cloudcertificates/` | Beta since v9.2.0; absent from TechDocs v9.3 sidebar; BETA/preview resources are explicitly OUT OF SCOPE per the user's instructions |

**Excluded Resources/Data Sources:**

| Item | Product | Reason |
|---|---|---|
| `akamai_botman_challenge_interception_rules` | botman | Deprecated → removed from codebase (CHANGELOG confirmed full lifecycle: added → deprecated in favor of `challenge_injection_rules` → removed). Testdata `.tf` files are cleanup artifacts. |
| Any resource/data source marked "deprecated" in TechDocs | All | User instructions explicitly exclude deprecated items unless in Priority 1 and not deprecated |
| Any resource/data source marked "BETA/preview" in TechDocs | All | Excluded unless in Priority 1 and not deprecated |

**Excluded Activities:**

- Refactoring existing resources from SDK to Framework pattern
- Changing provider architecture, plugin registration, or `terraform-plugin-mux` wiring
- Modifying existing resource/data source schemas, attribute names, or behavior
- Performance optimizations for existing code
- New shared utilities unless required by three or more new resources in the same product
- Rewriting existing tests or introducing new test frameworks
- Comprehensive provider documentation overhaul
- Architectural diagrams beyond the gap report
- Changes to `go.mod`, `go.sum`, or dependency versions
- Changes to CI/CD workflows (`.github/workflows/`)
- Changes to top-level provider wiring (`pkg/providers/providers.go`, `pkg/providers/registry/`)
- Changes to any subprovider other than the one being actively worked on
- Modifications to `pkg/common/`, `pkg/meta/`, or any shared infrastructure packages
- Any changes to the `main.go` entry point or plugin server setup


## 0.7 Rules for Feature Addition


**Minimal Change Clause (Paramount):**
- Make only the changes that are absolutely necessary to implement this feature
- Do not refactor, optimize, or modify existing code unless it is directly required for the new feature to work
- Isolate new code in dedicated files (new `resource_akamai_*.go`, `data_akamai_*.go`, `*_test.go`)
- Document all changes made to existing files (only `provider.go` registration additions) with clear comments
- If issues are identified in existing code, note them in the gap report but do not fix unless required for the feature
- When multiple implementation approaches exist, choose the one that requires the least modification to existing code

**Pattern Fidelity (Mandatory):**
- Determine each product's pattern by inspecting its `provider.go`: non-empty `SDKResources()`/`SDKDataSources()` → use SDK pattern; only `FrameworkResources()`/`FrameworkDataSources()` → use Framework pattern
- If a product mixes patterns, follow the mix (e.g., property has both SDK and Framework resources — choose the pattern appropriate to the resource being added based on the most similar existing resource)
- Do not introduce new patterns; copy structure and style from the referenced files in each product
- SDK reference: `pkg/providers/appsec/resource_akamai_appsec_configuration.go`
- Framework reference: `pkg/providers/apidefinitions/resource_akamai_apidefinitions_api.go`

**Registration Rules:**
- Register new resources and data sources only in the product's `provider.go`
- SDK resources: add to the map returned by `SDKResources()` with key `"akamai_<product>_<name>"`
- SDK data sources: add to the map returned by `SDKDataSources()` with key `"akamai_<product>_<name>"`
- Framework resources: append constructor to the slice returned by `FrameworkResources()`
- Framework data sources: append constructor to the slice returned by `FrameworkDataSources()`
- Do not change top-level provider wiring, plugin registration, or other subproviders

**Client Access Patterns (Per Product — Copy Exactly):**
- SDK products: `meta := meta.Must(m)` then `client := inst.Client(meta)` in each CRUD handler
- Framework products: Package-level client set in `Configure()` via `<package>.Client(metaConfig.Session())`
- Other products: Open an existing resource in that product and replicate the same client-obtainment pattern

**Error Handling (Must Match Product Pattern):**
- SDK: `return diag.FromErr(err)` for API errors; `return diag.Errorf(...)` for formatted errors; `errors.Is(err, tf.ErrNotFound)` for optional attributes
- Framework: `resp.Diagnostics.AddError("Title", "Description")` for all errors; never use `diag.FromErr` in Framework code
- Never mix SDK and Framework error handling in the same file

**Schema Mapping (TechDocs → Terraform):**
- TechDocs "required" → `Required: true`
- TechDocs "optional" → `Optional: true`
- TechDocs "read-only" → `Computed: true`
- Lists/arrays → `TypeList` or `TypeSet` with `Elem` (SDK) or equivalent Framework attribute types
- Nested objects → nested `schema.Resource` (SDK) or nested attributes (Framework)

**File Naming Convention (Strict):**
- Resources: `resource_akamai_<product>_<name>.go`
- Data sources: `data_akamai_<product>_<name>.go`
- Tests: `resource_akamai_<product>_<name>_test.go` or `data_akamai_<product>_<name>_test.go`
- Test fixtures: `testdata/TestRes<Name>/` or `testdata/TestDS<Name>/` with `.tf` and `.json` files

**Testing Requirements:**
- Every new resource/data source must have a corresponding `*_test.go` with at least one test case
- Use mock clients and `testutils.LoadFixtureBytes` per the product's existing test pattern
- Target ≥75% line coverage for new code
- Test CRUD operations for resources (create, read, update, delete) and read for data sources
- Do not rewrite existing tests or introduce new test frameworks
- Import and test production code directly — do not reimplement business logic in tests

**Scope-Limiting Rules (Mandatory Enforcement):**
- Per product: If a single product has more than 30 gaps, implement only the first 20 (by TechDocs listing order); document remainder as Phase 2
- Global: If total gaps across all products exceed 100, implement Priority 1 only and document Priority 2 as future work
- Global: If Priority 1 alone has more than 60 gaps, implement the top 50 (CRUD resources first, then data sources); document rest in gap report
- Thin subproviders (<5 TechDocs items): always implement 100% regardless of other limits

**TechDocs Alignment Documentation:**
- Every new resource/data source must include a comment or gap report entry identifying which TechDocs page/section it aligns with
- Example: `// Implements akamai_appsec_<name> per TechDocs appsec-resources`

**Backward Compatibility (Zero Tolerance):**
- Zero breaking changes to existing resource/data source schemas (no removed or renamed attributes)
- Zero changes to existing resource behavior except where strictly required for a new resource
- All existing tests must pass: post-implementation test pass count must match or exceed baseline
- Before modifying any public interface, identify all existing callers; preserve all existing behavior
- If a change to a public contract is necessary: add the new version alongside the old, mark the old as deprecated with a migration note

**Code Explainability:**
- Every new function, type, and method must include a docstring with purpose, parameters, and return values
- Inline comments must explain WHY decisions were made (alternatives considered, assumptions, trade-offs)
- Do not add comments that merely restate what the code does

**Risk Assessment (Mandatory Documentation):**
- Document all risks introduced by code changes, categorized by severity (HIGH, MEDIUM, LOW)
- Each risk must include: what could go wrong, affected systems, mitigation strategy, and rollback step
- Changes to public APIs or infrastructure without corresponding risk entries fail review


## 0.8 References


**Codebase Files and Folders Searched:**

| Path | Purpose of Search |
|---|---|
| `/` (repository root) | Root structure discovery — identified Go module, license, key directories |
| `go.mod` | Dependency inventory — Go version (1.24.10), all module dependencies with exact versions |
| `pkg/` | Top-level package structure — 5 subfolders (akamai, common, logger, meta, providers) |
| `pkg/providers/` | Subprovider directory listing — all 20 product folders plus registry and shared files |
| `docs/` | Documentation structure — resources/, data-sources/, guides/, index.md |
| `docs/resources/resources.md` | Subprovider resource index — 21-row table mapping products to TechDocs URLs |
| `docs/data-sources/data-sources.md` | Subprovider data source index — 21-row table mapping products to TechDocs URLs |
| `pkg/providers/appsec/provider.go` | Priority 1 — AppSec registrations: 54 SDK res, 54 SDK ds, 1 FW res, 2 FW ds (111 total) |
| `pkg/providers/botman/provider.go` | Priority 1 — Bot Manager registrations: 26 SDK res, 32 SDK ds (58 total) |
| `pkg/providers/gtm/provider.go` | Priority 1 — GTM registrations: 7 SDK res, 3 SDK ds, 8 FW ds (18 total) |
| `pkg/providers/property/provider.go` | Priority 1 — Property registrations: 6 SDK res, 19 SDK ds, 5 FW res, 10 FW ds (40 total) |
| `pkg/providers/edgeworkers/provider.go` | Priority 1 — EdgeWorkers registrations: 4 SDK res, 6 SDK ds (10 total) |
| `pkg/providers/apidefinitions/provider.go` | Framework-only subprovider pattern reference: 3 FW res, 3 FW ds (6 total) |
| `pkg/providers/accountprotection/provider.go` | Priority 2 — Account Protection registrations: 4 SDK res, 4 SDK ds (8 total) |
| `pkg/providers/clientlists/provider.go` | Priority 2 / Thin — Client Lists registrations: 2 SDK res, 2 FW ds (4 total) |
| `pkg/providers/cloudaccess/provider.go` | Priority 2 — Cloud Access registrations: 1 FW res, 4 FW ds (5 total) |
| `pkg/providers/cloudcertificates/provider.go` | Beta subprovider (OOS) — Cloud Certificates: 2 FW res, 3 FW ds (5 total) |
| `pkg/providers/cloudlets/provider.go` | Priority 2 — Cloudlets registrations: 4 SDK res, 10 SDK ds, 2 FW ds (16 total) |
| `pkg/providers/cloudwrapper/provider.go` | Priority 2 — Cloud Wrapper registrations: 2 FW res, 6 FW ds (8 total) |
| `pkg/providers/cps/provider.go` | Priority 2 — CPS registrations: 4 SDK res, 5 SDK ds (9 total) |
| `pkg/providers/datastream/provider.go` | Priority 2 / Thin — DataStream registrations: 1 SDK res, 3 SDK ds (4 total) |
| `pkg/providers/dns/provider.go` | Priority 2 — DNS registrations: 2 SDK res, 2 SDK ds, 1 FW ds (5 total) |
| `pkg/providers/iam/provider.go` | Priority 2 — IAM registrations: 4 SDK res, 9 SDK ds, 3 FW res, 16 FW ds (32 total) |
| `pkg/providers/imaging/provider.go` | Priority 2 — Imaging registrations: 3 SDK res, 2 SDK ds (5 total) |
| `pkg/providers/mtlskeystore/provider.go` | Priority 2 — mTLS Keystore registrations: 3 FW res, 3 FW ds (6 total) |
| `pkg/providers/mtlstruststore/provider.go` | Priority 2 — mTLS Trust Store registrations: 2 FW res, 8 FW ds (10 total) |
| `pkg/providers/networklists/provider.go` | Priority 2 — Network Lists registrations: 4 SDK res, 1 SDK ds (5 total) |
| `pkg/providers/appsec/resource_akamai_appsec_configuration.go` | SDK pattern reference — CRUD handlers, schema definition, client access, error handling |
| `pkg/providers/apidefinitions/resource_akamai_apidefinitions_api.go` | Framework pattern reference — struct-based resource, Configure, Metadata, Schema, CRUD |
| `pkg/providers/registry/` | Provider registry — `RegisterSubprovider()` mechanism with mutex-protected registration |
| `pkg/providers/providers.go` | Blank-import pattern triggering all subprovider `init()` hooks |
| `pkg/common/tf/` | Shared Terraform utilities — `ErrValueSet`, `ErrNotFound`, `SetAttrs()` |
| `pkg/common/str/` | Shared string utilities — `AddPrefix()`, string manipulation |
| `pkg/meta/` | Meta provider data — `meta.Must()`, `meta.Meta` interface, client access |
| `CHANGELOG.md` | Version history — confirmed challenge_interception_rules lifecycle (added → deprecated → removed) |
| `pkg/providers/botman/testdata/` | Verified .tf fixture artifacts from removed challenge_interception_rules |
| All `pkg/providers/*/` directories | Listed all resource/data source implementation files per subprovider via bash |

**TechDocs URLs Referenced (From docs/resources/resources.md):**

| Subprovider | Resource TechDocs URL | Data Source TechDocs URL |
|---|---|---|
| accountprotection | `techdocs.akamai.com/.../apr-resources` | `techdocs.akamai.com/.../apr-datasources` |
| apidefinitions | `techdocs.akamai.com/.../apidef-resources` | `techdocs.akamai.com/.../apidef-datasources` |
| appsec | `techdocs.akamai.com/.../appsec-resources` | `techdocs.akamai.com/.../appsec-datasources` |
| botman | `techdocs.akamai.com/.../botman-resources` | `techdocs.akamai.com/.../botman-datasources` |
| clientlists | `techdocs.akamai.com/.../cli-resources` | `techdocs.akamai.com/.../cli-datasources` |
| cloudaccess | `techdocs.akamai.com/.../cam-resources` | `techdocs.akamai.com/.../cam-datasources` |
| cloudlets | `techdocs.akamai.com/.../cl-resources` | `techdocs.akamai.com/.../cl-datasources` |
| cloudwrapper | `techdocs.akamai.com/.../cw-resources` | `techdocs.akamai.com/.../cw-datasources` |
| cps | `techdocs.akamai.com/.../cps-resources` | `techdocs.akamai.com/.../cps-datasources` |
| datastream | `techdocs.akamai.com/.../ds-resources` | `techdocs.akamai.com/.../ds-datasources` |
| dns | `techdocs.akamai.com/.../edns-resources` | `techdocs.akamai.com/.../edns-datasources` |
| edgeworkers | `techdocs.akamai.com/.../ew-resources` | `techdocs.akamai.com/.../ew-datasources` |
| gtm | `techdocs.akamai.com/.../gtm-resources` | `techdocs.akamai.com/.../gtm-datasources` |
| iam | `techdocs.akamai.com/.../iam-resources` | `techdocs.akamai.com/.../iam-datasources` |
| imaging | `techdocs.akamai.com/.../ivm-resources` | `techdocs.akamai.com/.../ivm-datasources` |
| mtlskeystore | `techdocs.akamai.com/.../moks-resources` | `techdocs.akamai.com/.../moks-datasources` |
| mtlstruststore | `techdocs.akamai.com/.../mets-resources` | `techdocs.akamai.com/.../mets-datasources` |
| networklists | `techdocs.akamai.com/.../nl-resources` | `techdocs.akamai.com/.../nl-datasources` |
| property | `techdocs.akamai.com/.../pm-resources` | `techdocs.akamai.com/.../pm-datasources` |

**Existing Tech Spec Sections Referenced:**

| Section | Content Used |
|---|---|
| 1.1 Executive Summary | Project overview (v9.3.0, MPL 2.0, Plugin Protocol v6.0, 20 subproviders), stakeholder groups, dual-mode architecture |
| 2.1 Feature Catalog | All 25 features (F-001 to F-025), 6 categories, completion status — confirmed all features are Completed |
| 3.1 Programming Languages | Go 1.24.10, CGO_ENABLED=0, 15 OS/arch build matrix, HCL for examples |
| 5.1 High-Level Architecture | 4-layer architecture, 8 core components, data flow paths, external integration points |

**Web Searches Conducted:**

| Search Topic | Finding |
|---|---|
| TechDocs v9.3 sidebar parsing (botman-resources page) | Extracted full sidebar navigation listing all subprovider resources and data sources across 19 documented products |
| Botman challenge_interception_rules deprecation | Confirmed deprecated in favor of challenge_injection_rules, subsequently removed from codebase |
| Cloud Certificate Manager (CCM) TechDocs status | Confirmed absent from TechDocs v9.3 sidebar — Beta product not yet documented |
| Akamai Terraform provider coverage targets | Validated coverage methodology and gap report format against provider community practices |

**Attachments:**
No attachments were provided for this project.

**Figma URLs:**
No Figma designs were provided for this project.


