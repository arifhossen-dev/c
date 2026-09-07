# TMS (Takaful Management Solution) — Technical Brief
 
Repo: `tmsalpha` (local), remote `git@github.com:Tamini-Insurance/tms-alpha.git`, branch `develop`.
Prepared as a raw technical fact-finding document — no marketing language. Every claim below is either sourced from code (file:line where useful) or explicitly marked UNCLEAR where it couldn't be verified from the repo alone.
 
---
 
## 1. What This System Actually Does
 
TMS is a back-office insurance management system ("insurance ERP") built for a Takaful (Islamic-finance-compliant) and conventional insurer operating in East Africa. It covers the full lifecycle of an insurance business in one system:
 
- **Selling a policy**: leads and CRM, product catalogue, quotations with pricing/discounts, multi-level approval workflows, conversion to an active policy.
- **Underwriting**: risk evaluation, policy issuance, mid-term changes (endorsements), cancellations, motor-certificate issuance for Kenya (DMVIC).
- **Claims**: intake, reserves, settlement, payments, and recovery from reinsurers.
- **Reinsurance ("retakaful")**: the company shares risk with other reinsurers through treaties — this system tracks how much risk is ceded out, how much comes back as recoveries, and the accounting for both.
- **Finance**: a full general ledger (double-entry accounting) aligned to IFRS-17 (the international insurance accounting standard), bank reconciliation, fixed assets, budgets, multi-currency.
- **HR & payroll**: employee records, leave, attendance (including fingerprint/biometric clock-in), and payroll that correctly applies Kenyan and Ugandan tax law.
- **Procurement**: purchase requests through to vendor payments.
Three groups of people use it: internal staff (underwriters, claims handlers, accountants, HR/payroll admins, procurement) through an admin panel; employees themselves through a self-service portal (payslips, leave requests); and external customers/agents through a separate portal and a mobile-facing API (buying a policy, filing a claim, making a payment).
 
The problem it solves: replacing a fragmented or legacy process (there's active tooling in this codebase for migrating data from a prior system referred to internally as "GT") with one system where a sale, a claim, or a payroll run all flow automatically into the same audited general ledger — rather than being reconciled by hand between separate tools.
 
---
 
## 2. Architecture
 
**Structure**: Single Laravel 12 application — a modular monolith, not microservices. ~295 Eloquent models, ~183 service classes, ~126 Filament admin resources, ~844 migrations. Domain separation is purely by folder convention (`app/Services/{Retakaful,Payroll,Crm,GtMigration,...}`, `app/Traits/{Entries,Quotation,Underwriting,Endorsement}`), all under one `App\` namespace — no package/monorepo boundaries.
 
**Presentation surfaces**:
- Filament v4 admin panel at `/tia` (operations, underwriting, finance, HR, retakaful, reports)
- Filament employee self-service panel at `/employee`
- Vue 3 + Inertia.js v2 customer portal at `/portal`
- Versioned REST API at `/api/v1/*`, Sanctum-gated, serving mobile/agent/partner clients
**Key patterns**:
- **Service layer**: constructor-injected, instance-method services (not static, not DTO-heavy — services largely operate directly on Eloquent models and arrays). E.g. `app/Services/Retakaful/ProfitCommissionService.php`, `app/Services/Crm/LeadConversionService.php` (wraps mutation in `DB::transaction()`, throws domain exceptions).
- **No Actions-pattern directory** (`app/Actions` doesn't exist). Business logic that would live in an "Action" class elsewhere lives in Services or Observers instead.
- **Events/Listeners are largely dead code**: only one custom `Event` class exists (`app/Events/NewQuotationGenerated.php`) and it is never actually dispatched anywhere. Of 5 Listener classes, only 2 are wired via `Event::listen()` (`app/Providers/AppServiceProvider.php:136-137`), both against a third-party payment package's events. The real reactive mechanism is **10 Eloquent Observers** registered centrally in `AppServiceProvider.php:117-127` (JournalEntry, GlEntry, VendorBill, Payment, PayrollRecord, FixedAsset, etc.) driving synchronous side effects on model changes.
- **Queue/jobs**: 24 job classes, all confirmed `ShouldQueue`, Horizon-supervised. Only one `dispatchSync()` call in the entire codebase, and it's inside an Artisan command (acceptable), not the request cycle.
- **GL posting**: domain-specific traits under `app/Traits/Entries/` (`HasClaimEntries`, `HasEndorsementEntries`, `HasTreatyEntries`, `HasCurrencyAwareEntries`, etc.) build journal lines, but the actual posting invariants are centralized in `JournalEntry::postToGl()` (`app/Models/JournalEntry.php:106-225`), which: requires prior approval, refuses double-posting, **re-verifies debit totals equal credit totals immediately before posting** (defensive re-check against upstream bugs, per its own code comment), wraps posting in `DB::beginTransaction()/commit()/rollBack()`, and builds a **hash chain** per GL entry (`entry_hash`/`prev_hash`/`reversal_of_id`) for tamper evidence.
- **Multi-country payroll**: a genuine strategy/resolver pattern — `App\Contracts\Payroll\StatutoryCalculator` interface, `KenyaStatutoryCalculator` and `UgandaStatutoryCalculator` implementations, dispatched by `StatutoryCalculatorResolver` via a country-code map resolved through the Laravel container. Extending to a third country means adding a map entry and a class — no auto-discovery, but a clean seam.
**Multi-tenancy / data isolation — important**: there is **no true multi-tenancy**. `config/filament-shield.php:44` explicitly sets `tenant_model => null`. No tenancy package (e.g. `stancl/tenancy`) is installed. No `addGlobalScope`/custom scope classes exist anywhere under `app/Models` for company/branch isolation. Isolation between branches/companies exists only as plain `branch_id`/`company_id` foreign-key columns, filtered ad hoc in roughly a dozen Filament report/resource files — not centrally enforced. The system's own README states this directly: *"current design is multi-branch / multi-company within one deployment; full tenant isolation is not implemented."*
 
**Database design notables**:
- **Polymorphic relations** (18 models): `Employee::manager()` (`morphTo`, `manager_type`/`manager_id`); `Task` via `taskable` (used by Customer/Lead/Opportunity); `PendingApprovals` via `approvable`; `GlEntry::source()` linking ledger entries back to Payment/ClaimPayment/etc.
- **Soft deletes**: used on 14 models, but deliberately **absent** from the financial core (`GlEntry`, `JournalEntry`, `JournalLine`, `Payment`) — the ledger is immutable by design; corrections happen through reversal entries, not deletion. This is the correct pattern for financial-record integrity.
- **Audit trail**: Spatie Activitylog wrapped in a `HasActivityLogging` trait, used across 239 models. Separately, a bespoke `PayrollAuditLog` model stores old/new values as **encrypted** columns plus IP/user-agent/retention metadata — a stronger, purpose-built layer for payroll specifically, on top of the generic log.
- **Multi-currency**: `ExchangeRate` model (approval-gated) plus `JournalLine` storing *both* base-currency and original-transaction-currency amounts alongside the `fx_rate` actually used — correctly normalized, not just a computed converted total (a common mistake this avoids).
- **Indexing**: core high-traffic tables (`gl_entries`, `payments`, `quotations`) have FK constraints plus well-chosen composite indexes. One clear retrofit was found: a 2026-01-05 migration adding missing indexes (`policy_number`, date ranges, composite customer/product+date) to `underwriting_policies` — direct evidence a real missing-index performance issue existed and was fixed after the fact, not designed in from day one.
---
 
## 3. Notable Technical Decisions
 
- **Hash-chained GL ledger entries** (`entry_hash`/`prev_hash`/`reversal_of_id` in `JournalEntry::postToGl()`) — tamper-evidence chaining is not a common feature in mid-market insurance ERPs; this is a deliberate, considered choice.
- **Central re-validation of debit=credit balance** at posting time even though upstream code is supposed to already balance the entry — explicit defensive programming against upstream bugs, documented in a code comment.
- **Deliberately immutable financial core** (no soft deletes on GL/Payment models) contrasted with soft deletes used elsewhere — shows judgment about where mutability is safe.
- **Currency-aware GL lines** persisting both original and base-currency amounts plus the FX rate used, not just a converted total.
- **`StatutoryCalculatorResolver`** — real interface-based strategy pattern for country-specific tax law, not a hardcoded if/country branch.
- **`FunctionalRole` enum + `RoleResolver` + `config/roles.php`** (per project history, `PR #162`) — maps semantic role identifiers (e.g. "hr", "director") to actual Spatie role names via per-environment `.env` overrides, since role display names differ between Kenya and Uganda deployments ("HR Manager" vs "HR & Administration"). Solves a genuine deployment-consistency problem without code changes per environment.
- **Weaker spot**: Events/Listeners infrastructure exists but is essentially unused/dead code (one unused Event, 2 of 5 Listeners wired) — suggests an abandoned or inconsistent architectural decision, not a deliberate one.
- **Authorization convention followed inconsistently**: `CLAUDE.md` mandates `->authorize('ability')` on Filament Actions, but of 1,438 custom `Action::make()` calls repo-wide, only 3 use it. Standard `DeleteAction`/`DeleteBulkAction` (250 occurrences) get policy-based authorization automatically from the framework, so the gap is specifically in bespoke workflow actions (void, disburse, approve, cancel, post) — these generally rely only on `requiresConfirmation()`, not an explicit per-action authorization check (see Section 7).
---
 
## 4. Integrations
 
| Integration | Status | Error handling |
|---|---|---|
| **M-Pesa** (STK push, C2B) | Real, implemented (`app/Traits/HasMpesaPayment.php`) | Try/catch + logging present, but **no HTTP timeout** set on the outbound STK call (commented out), and **inbound callback endpoints have no signature verification** — see Security section. Confirmation handler always returns success to Safaricom even on internal failure, "to avoid retries," logging internally instead. |
| **WAAFI Wallet** | **Not implemented.** README and docs claim it exists; a repo-wide grep found exactly one hit — a seeded `PaymentMethod` row named "Waafi Wallet." No controller, service, route, or job exists. `.env.example` has dead `WAAFI_*` placeholders read by nothing. |
| **DMVIC** (Kenya motor certificate authority) | Real, robust — full REST client (`DmvicCertificateService`, 1,163 lines), client-cert SSL auth, exponential-backoff retry on connection errors, structured error returns, extensive null-safety in payload building. |
| **ZKTeco biometric attendance** | Real, robust — proxies to a Node.js microservice; retry with backoff on connection errors; sync job is `ShouldQueue`, `tries=3`, `timeout=300`, has a `failed()` handler; scheduled every 15 minutes. |
| **Africa's Talking** (SMS) | Real, weaker — queued job, but **no try/catch around the outbound HTTP call**, no explicit retry/backoff/timeout configured beyond Laravel's queue defaults, no `failed()` handler. Does persist a log record of the attempt either way. |
| **DeepSeek / OpenAI / "Lumi" AI assistant** | Real — two separate stacks. **Lumi** is a customer-facing chat agent on Laravel's first-party `Laravel\Ai` SDK with ~20 tool classes (policy lookup, claims, quote assist), defensively logged, usage telemetry that's designed to never break the user-facing response on failure. **RIMA agents** use a different framework (`neuron-ai`), configurable across OpenAI/Anthropic/DeepSeek/Gemini/etc. via `RIMA_LLM` env var, used for AI-assisted quotation risk scoring — combined with a separate deterministic rules-based scorer (`RiskScoreCalculationService`), taking the max of both. Genuinely hybrid, not a stub. |
| **Firebase** (push notifications, via `kreait/laravel-firebase`) | Real, wired in — `PushNotificationService` gracefully degrades if unconfigured, handles per-token multicast failures, auto-deactivates dead device tokens. Called from Claim/Policy/PolicyPayment observers, genuinely triggered by domain events. |
| **Inspector APM** | Installed, auto-instrumentation only — no manual `addSegment()` custom instrumentation calls found anywhere in `app/`. |
| **GT legacy system migration** | Real, maintained tooling — a multi-phase migration service plus 15+ Artisan commands for import, backfill, and reconciliation (customers, policies, claims, payments, intermediaries). Has the shape of an actively-used data pipeline, not a one-off script. |
 
---
 
## 5. Performance & Scale Considerations
 
- A **purpose-built API performance middleware suite** exists specifically for the mobile/customer API layer: Redis-pattern-based response caching (`ApiCacheMiddleware`), rate limiting, request logging, and performance monitoring — this is not default Laravel caching, it was built specifically for this system's API traffic.
- **Queues**: Horizon-supervised; production queue/cache driver (Redis vs database) is configured via environment — the committed default/local config uses the database driver, so whether Redis is actually the backing store in production is environment-dependent and not verifiable from the repo alone (UNCLEAR).
- **Indexing**: core transactional tables are indexed with FK constraints and composite indexes; at least one real retrofit ("add missing indexes to `underwriting_policies`") shows an actual production performance issue was diagnosed and fixed.
- **N+1 query risk**: `CLAUDE.md` documents an eager-loading discipline as house convention, but actual compliance was not independently audited in this pass — mark as UNCLEAR, worth a dedicated review before making any specific N+1 claim.
- **Scheduled workload**: a dense job schedule including a 5-second-interval job that resolves stuck GL entries, 15-minute biometric syncs, and daily/weekly/monthly accounting-close and reporting jobs — the operational cadence reads as a system built for continuous production load, not a demo.
- **Idempotency keys and temporary API bans** (`IdempotencyKey`, `ApiTemporaryBan` models) on mutating endpoints — this is production-hardening work, not something typically present in a prototype.
---
 
## 6. Testing
 
- **Pest 3**, 396 test files total: 382 under `tests/Feature`, 12 under `tests/Unit`, 36 under `tests/Feature/Api` (a subset of Feature).
- **Strong, edge-case coverage** in the highest-risk domains:
  - Payroll statutory calculators (Kenya + Uganda) — tests boundary conditions specifically: SHIF/Housing Levy caps, disability PAYE exemption, zero-salary edge case, Uganda's seasonal LST election window, PAYE surcharge threshold at UGX 10M.
  - GL/journal entry reversal — includes negative-path tests (cannot double-reverse an entry, cannot reverse an unposted entry), not just happy path.
  - Retakaful/treaty engine has dedicated test files (proportional cession, per-line facultative, treaty bypass approval), though full depth of assertions wasn't independently re-verified beyond confirming their existence and general scope.
- **Gaps**: zero test coverage for the Vue/Inertia frontend (`resources/js`) — no test files, no test runner config found. No Pest architecture (`arch()`) tests anywhere. Unit tests are a small minority (12 files) — nearly all coverage is at the Feature level.
- The suite's actual pass/fail status was **not run** as part of this brief — only the existence and rough scope of tests was reviewed, not execution results.
---
 
## 7. Security Posture
 
**Authentication**: Laravel Sanctum for API/portal tokens, two separate auth stacks (Agent, Customer) each with real device-scoped token issuance/revocation and genuine TOTP-based two-factor authentication (enable/confirm/verify/disable flows backed by `AuthOtpService`). Fortify is present for panel/session-based admin login (not independently re-verified this pass — UNCLEAR on specifics).
 
**Authorization**: Filament Shield generates one Policy class per model (111 files) with real role-or-permission checks (not a rubber-stamped `return true`) — spot-checked several, all consistent. The gap is in **bespoke Filament workflow actions**: `CLAUDE.md` mandates `->authorize()` on custom actions, but only 3 of 1,438 `Action::make()` calls actually use it. Examples with no explicit per-action authorization: voiding a cheque, disbursing an advance-salary payment — these rely only on a confirmation dialog. Standard delete actions are covered automatically by Filament's own policy integration, so the exposure is specifically in custom financial/workflow actions. Whether this is a real security hole depends on whether resource-level page access already gates these adequately — worth confirming with the team rather than assuming either way.
 
**Data isolation**: confirmed no automatic branch/company scoping (see Section 2) — this is a real architectural gap for a multi-branch deployment: any query that forgets to filter by `branch_id`/`company_id` can leak cross-branch data to an authenticated user who otherwise has the relevant model permission.
 
**Input validation**: dedicated Form Request classes exist and are generally thorough where used, but roughly 20 API controller endpoints validate inline via `$request->validate()` instead of a Form Request class — an inconsistency against house convention, not inherently insecure.
 
**A genuine vulnerability, flagged for private review (not fixed here)**: the M-Pesa/mobile-money payment callback endpoints (`payments/confirm`, `payments/c2b/validate`, `payments/c2b/confirm`, `payments/momo/collection/initiate`) have **no signature/HMAC verification of inbound payloads**. A `VerifyWebhookSignature` middleware exists in the codebase (aliased `webhook.verify`) but is **not applied to any route**. As written, these endpoints rely entirely on infrastructure-level trust (e.g. IP allowlisting at a firewall/CDN, which isn't visible from code) — if that isn't separately enforced, a forged callback could mark a policy payment as paid. Recommend confirming with the team whether IP-level protection exists before this is deployed anywhere it matters, and wiring the existing middleware onto those routes.
 
**Already-fixed security work** (documented in the repo's own history, `security_update` branch): a previously hardcoded reCAPTCHA secret that leaked via query string was fixed; stored XSS from unsanitized `{!! !!}` rendering of admin-authored rich text was fixed with an allow-list HTML sanitizer, an Eloquent cast, and a Vue directive; raw string-interpolated SQL in two console commands was parameterized; password values were removed from log output. One residual unescaped `{!! !!}` output (a product cover description in a PDF template) wasn't covered by that sanitization pass — low risk since it requires an already-compromised or malicious admin-level content editor, but worth a look.
 
No mass-assignment risk found (no `$fillable = ['*']` or empty `$guarded`), no other raw-interpolated SQL statements found beyond what's already been fixed.
 
---
 
## 8. Complexity & Difficulty Assessment
 
Genuinely hard parts of this system:
- **The retakaful/reinsurance engine** (proportional, non-proportional/XOL, facultative, inward treaties, IBNR reserving, bordereau import) is specialist actuarial/insurance domain logic, not generic CRUD — getting cession and recovery calculations right through policy cancellations and endorsements, without double-counting or losing money on either side of a treaty, is a common source of real bugs in insurance systems.
- **Multi-jurisdiction statutory payroll** (Kenya PAYE/NSSF/Housing Levy vs Uganda PAYE/NSSF/LST with seasonal election rules and lump-sum annualisation) — a less experienced developer would likely hardcode tax logic per country inline rather than build a resolver/strategy abstraction, and would likely miss boundary conditions (caps, exemptions, thresholds) that this codebase's tests explicitly cover.
- **Double-entry GL correctness** across multi-currency, multi-branch operations, with reversal/cancellation flows touching claims, endorsements, and retakaful cession simultaneously — the balance-reverification-before-posting and hash-chaining pattern shows real awareness of where ERPs commonly fail silently (an unbalanced posting that goes unnoticed).
What a less experienced developer would likely have gotten wrong, based on what this codebase specifically got right: soft-deleting financial records (breaking audit integrity — avoided here); trusting upstream code to already balance debits and credits without a final check (avoided — explicit re-verification exists); storing only a converted currency total instead of the original amount and rate (avoided — both are stored). The one area where this pattern of care wasn't fully carried through is branch/company data isolation, which is ad hoc rather than centrally enforced — a place where the same rigor seen in the GL layer wasn't applied.
 
---
 
## 9. Portfolio-Worthy Features
 
1. **Hash-chained, balance-reverified double-entry GL posting engine.** Needs a couple of sentences of translation for a non-technical client (the concept of tamper-evidence via a hash chain, and why a final balance check matters), but is a strong, unusual differentiator.
2. **Multi-country statutory payroll engine (Kenya + Uganda, cleanly extensible to a third country).** Easy to explain to a client ("the system automatically applies the correct tax rules for each country") — minimal translation needed.
3. **Full retakaful/reinsurance engine** (treaties, facultative offers, IBNR, bordereau import). Needs more explanation since reinsurance itself is a specialist concept — best pitched to an insurance-industry-literate audience specifically.
4. **AI-assisted quotation risk scoring**, combining a deterministic rules engine with an LLM-based assessment, queued and retried. Modern and sellable; needs light explanation of the "hybrid AI + rules" framing so it doesn't read as "just called an AI API."
5. **Purpose-built API performance layer** for the mobile/customer API (Redis-backed response caching, rate limiting, security middleware, performance monitoring) — per the project's own contribution analysis, this is your dominant, verifiably-authored contribution. Strong standalone case-study candidate; needs moderate explanation (cache invalidation strategy, what the rate limiting protects against).
6. **Deployment-independent role resolution** (a functional-role abstraction that maps semantic roles to differently-named actual roles per country/environment via config, not code). A good "senior judgment" story — solves a real, non-obvious multi-country deployment problem — needs a little translation but is a clean example of engineering judgment rather than raw feature-building.
---
 
## 10. Confidentiality / Disclosure Flags
 
**Must not be shown publicly:**
 
- **Live-looking secrets committed to `.env.example`**: an Inspector APM ingestion key and a DeepSeek API key both appear to be real, non-placeholder values (unlike other entries in the same file, which are clearly empty templates). Recommend rotating both regardless of whether this brief is ever shared, since they're sitting in a tracked file today. Location: `.env.example` (Inspector key and DeepSeek key lines, near the top third of the file) — review directly, do not paste the values anywhere.
- **`database/seeders/UsersSeeder.php`**: contains real-looking employee names paired with real corporate email domains, and **hardcoded, predictable passwords** following a guessable pattern (name + a shared suffix). This file must never be shown publicly. Separately from the portfolio question: if this seeder was ever run against a real or shared environment, those passwords should be treated as compromised and flagged to the team for rotation — that's a heads-up for you to act on privately, not something addressed in this brief.
- **Client identity is pervasive**, not confined to README/CLAUDE.md — found in roughly 35 files including `config/app.php` (company name), `app/Models/User.php` (hardcoded corporate email-domain checks), a Vue component file literally named after the client, the public landing page navbar, legal terms/privacy documents, the retakaful operations guide, and various Retakaful service/notification files. Scrubbing this codebase for any kind of public code walkthrough would require far more than hiding two files.
- **CI/CD workflow** (`.github/workflows/ci-cd.yml`): no live credentials are hardcoded (secrets are correctly referenced via GitHub Secrets), but the client's real organization name, an SSH key alias, and production/staging hostnames are hardcoded in plaintext.
- **The git remote itself** identifies the client by name — be mindful of this in any screen-share, screenshot, or terminal recording.
**Borderline — could identify the client even without naming them:**
- The specific combination of features here (Kenya + Uganda statutory payroll, IFRS-17 Takaful accounting, DMVIC Kenya motor-certificate integration, a full retakaful engine) is a fairly narrow fingerprint. An East African insurance-industry reader could plausibly identify the client from the feature list alone, even with all names removed.
- `docs/retakaful_guide.md` and the Uganda payroll rulebook doc likely encode real operational/regulatory specifics tied to this client's actual treaty relationships — review before quoting any rule specifics, even generically.
---
 
## 11. Open Questions For You
 
- **Timeframe discrepancy**: you stated "July 2025 to July 2024" as the engagement window. Git evidence shows the repo was created 2025-07-06/07-12, and the most recent commits reviewed date to mid/late-2026, with your own commits (per the existing `CONTRIBUTIONS_REPORT.md`) spanning 2025-07-12 to 2026-07-06. Please confirm the actual dates before these go into any public-facing material — the stated timeframe looks like it may have a typo.
- **Actual production status/scale**: the project's own README claims production deployment "across multiple East African markets," but that document also contains other claims (a WAAFI integration, multi-tenancy) that were independently verified as false or overstated. Is production deployment real, and if so, how many branches/policyholders/staff actually use it today? Can't be verified from code alone.
- **Team size and your specific scope**: git history shows at least 4-5 other identifiable contributors and 223 total PRs in the repo. Your own `CONTRIBUTIONS_REPORT.md` already documents your verified, module-level ownership (finance/accounting, HR/payroll, approvals, and the customer API performance layer specifically) — the portfolio framing should stick to what that report already verified, not the project's full feature set.
- **The M-Pesa callback authentication gap** (Section 7): is this already mitigated at the infrastructure level (IP allowlisting, firewall rules) that wouldn't show up in this codebase? Matters for whether/how you'd want to describe payment integration work.
- **Measurable business outcomes** — time saved, a legacy system retired, error/incident reduction, anything quantifiable — none of this is visible from code and would need to come from you or the client.
- **The "sub-500ms API" performance claim** referenced in your existing contributions report has no committed benchmark evidence backing the specific number, even though the monitoring code to measure it does exist. Do you have real numbers, or should this be described qualitatively (e.g. "built a dedicated caching and rate-limiting layer") rather than with a specific latency figure?
 
