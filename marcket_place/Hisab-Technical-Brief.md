# Hisab — Technical Brief
 
**Prepared:** 2026-08-28
**Purpose:** Raw technical findings for portfolio/Upwork translation. Not marketing copy.
**Repo:** `nextme/ortho` (local working directory name; app name is "Hisab")
**Role:** Sole developer (per `docs/REQUIREMENTS.md`: "Prepared by: Arif")
**Client:** A named individual client, referred to only as "the client" in project docs, migrating from a personal Excel/Google Sheets finance workflow. Not a company — a personal/family finance client engagement.
**Timeframe (from git history):** First commit 2026-05-04, most recent commit 2026-06-04 — approximately one month of active development, 205 commits.
**Stack:** PHP 8.4, Laravel 13, Inertia.js v3, Vue 3, Laravel Fortify, Tailwind v4, Pest v4/PHPUnit 12, Vite, Laravel Wayfinder.
 
---
 
## 1. What This System Actually Does
 
Hisab (হিসাব — Bengali for "accounting/reckoning") is a mobile-first web app that replaces a client's manual Excel/Google Sheets system for tracking personal and family finances.
 
**The problem it solves:** The client kept one spreadsheet workbook per month to track income, expenses, and money lent to or borrowed from named family members and friends (e.g., "Tarik owes me ৳5,000"). Over ~2.5 years of monthly workbooks, several structural problems accumulated:
- Loans were recorded as plain income/expense, corrupting the "money saved this month" figure.
- There was no way to ask "how much does person X currently owe me?" — that information was buried in free-text notes across many files.
- Category names drifted month to month ("Home" vs "House Rent"), breaking year-over-year comparisons.
- Bangla numerals typed into cells silently broke spreadsheet SUM formulas.
- Each month's starting balance had to be manually copied from the last, with no running ledger.
- Editing a spreadsheet on a phone (where most transactions actually happen — a taxi fare, a lunch) was painful, so entries were often skipped.
**Who uses it:** The client and select family members, who can be granted either edit or view-only access to a shared household's financial data. The app supports both Bangla and English throughout.
 
**What it does day to day:** Lets a user log income, expenses, loans given/received, loan repayments, and inter-wallet transfers (e.g., moving money from cash to a bank account, or converting BDT to USD) against a set of wallets (cash, mobile wallets like bKash/Nagad/Rocket, bank accounts, USD digital wallets). It tracks a running per-person balance for anyone money has been lent to or borrowed from, computes real-time wallet balances and net worth, and produces monthly/yearly reports with consistent categories. Recurring monthly items (salary, rent, SIP investment auto-debits) can be templated so they don't need re-entry every month.
 
---
 
## 2. Architecture
 
**Shape:** Server-rendered SPA monolith — Laravel backend + Inertia.js bridging directly to Vue 3 components, no separate REST/JSON API layer and no SPA-side routing framework. Single Laravel app, single MySQL/MariaDB database. Deployed to Hostinger shared hosting (no containers, no Redis, no dedicated queue worker).
 
**Multi-tenancy model:** Row-level tenancy scoped by `household_id`, not separate databases/schemas per tenant. Implementation:
- `app/Concerns/BelongsToHousehold.php` — a trait applied to every household-owned model (Wallet, Category, Counterparty, Transaction, Transfer, RecurringTemplate, Loan, Budget). It registers a global Eloquent scope that filters every query by `Auth::user()->current_household_id`, and a `creating` model event that auto-fills `household_id` on new records if not already set.
- Backstopped at the route layer by `app/Http/Middleware/EnsureHouseholdSelected.php`, which redirects any authenticated user with no `current_household_id` before they can reach a household-scoped route.
- This is a "fail closed by construction" design: a controller bug can't leak another household's data because the scope is applied at the model layer, not re-implemented per query.
**Architectural patterns:**
- **No dedicated service layer.** Business logic sits in controllers (`TransactionController::buildTotals()`/`signFor()`, `DashboardController::computeWalletBalances()`), a handful of `app/Support` helper classes (`CounterpartyBalance`, `CounterpartyMerger`, `BanglaDigits`, `LandingContentMerger`), and a small `app/Actions` folder (`ProvisionHousehold`, two custom Fortify actions).
- **Form Requests for validation** — 23 request classes, one Store/Update pair per resource, following Laravel convention.
- **Policy-based authorization**, enforced via Laravel 13's `HasMiddleware` interface + `Middleware('can:...')` declarations on controllers rather than inline `$this->authorize()` calls.
- **Observer pattern for audit logging** — a single `AuditObserver`, attached to models via the PHP 8 `#[ObservedBy(AuditObserver::class)]` attribute, writes to an `audit_logs` table on changes.
- **PHP 8 attributes over class properties** in places — e.g. `#[Fillable([...])]` instead of `protected $fillable`.
**Database design — non-trivial points:**
- **No stored balances.** Wallet balance, per-counterparty loan balance, and net worth are never persisted columns — they're computed on read from the transaction/transfer history (`DashboardController::computeWalletBalances()`, `app/Support/CounterpartyBalance.php`). This was an explicit design decision (documented in `docs/SPEC.md`) to keep the transaction ledger as the single source of truth and avoid balance-drift bugs, at the cost of an aggregation query on every dashboard/report view.
- **Money as integers, not floats.** All monetary columns are `bigInteger` minor units (paisa/cents), cast through `app/Casts/MoneyCast.php`, which does a bare `(int)` cast in both directions — no float ever touches a money value. FX rates use `decimal(18,8)`, and a denormalized `base_amount_bdt_minor` column on transactions avoids doing FX-rate joins at report time.
- **A five-type transaction enum** (`income`, `expense`, `loan_given`, `loan_received`, `loan_repayment`) is the domain's core fix for the spreadsheet's original sin — loans no longer pollute income/expense totals.
- **Transfers are single rows**, not two linked transaction legs (a `from_*`/`to_*` pair on one row) — a deliberate simplification since transfers don't need per-leg category attribution.
- Self-referential adjacency-list patterns: `Category` has `parent()`/`children()`; `Counterparty` has a `mergedInto()` tombstone pointer for merging duplicate people without deleting history.
- **No soft deletes anywhere** in the schema — deletion behavior is handled via foreign key policy instead: `restrictOnDelete` on `transactions.wallet_id` (can't delete a wallet with history), `nullOnDelete` on `category_id`/`counterparty_id` (detaches on delete despite the parent not being soft-deletable), `cascadeOnDelete` on `household_id` everywhere.
- Composite indexes exist on the hot query path: `[household_id, occurred_on]` and `[counterparty_id, type]` on `transactions`; `[household_id, occurred_on]` on `transfers`.
- `AuditLog` is the only polymorphic relation in the schema (`MorphTo` on `model_type`/`model_id`).
---
 
## 3. Notable Technical Decisions
 
- **Household-scoping via global scope + auto-fill, rather than per-query filtering.** This is the single most consequential decision in the codebase for a multi-tenant-adjacent niche — it makes cross-household data leakage structurally hard rather than a discipline problem, and it's specifically what the test suite tries to prove (see §6).
- **Integer money with a dedicated cast class**, rather than a money value-object library or floats. Simple, but correct — avoids the classic "$0.1 + $0.2 floating point" class of bug in a finance app, which is exactly where such a bug would be most damaging.
- **Deliberately not storing derived balances.** A less experienced developer would likely have cached a `balance` column on `wallets` and then had to solve balance-drift/reconciliation bugs when transactions are edited or deleted retroactively. This codebase sidesteps that whole failure class by always computing from the ledger, accepting a bit more read-time query cost in exchange.
- **`Rule::exists` doubling as a tenancy check.** Because `Wallet`/`Category`/`Counterparty` are `BelongsToHousehold` models, a `Rule::exists` validation rule against those tables transparently also enforces tenant isolation — a foreign-household ID fails validation before authorization is even checked. This is a "belt-and-suspenders" pattern that isn't obvious unless you understand how the global scope interacts with query-based rules.
- **Zero-downtime deploy pipeline on shared hosting.** `.github/workflows/deploy.yml` implements an atomic-symlink release pattern (timestamped release directories, shared `.env`/storage symlinks, atomic `current` symlink flip, opcache clear, prune to 5 kept releases) — considerably more sophisticated than what "Hostinger shared hosting" usually implies. This is the kind of detail that reads as senior-level infra thinking applied to a budget-constrained hosting environment.
- **Recurring-template materialization is idempotent and impersonation-aware.** `RunRecurringTemplates` (the daily cron job) tracks `last_run_on` per template to avoid double-creating entries, runs inside a `DB::transaction()`, and attributes created rows to the original template's creator for audit purposes — a level of care around idempotency and audit correctness that's easy to skip in a first pass.
- **PWA support** (`vite-plugin-pwa`, CacheFirst runtime caching for build assets) was added despite this being positioned as an MVP — gives installable/offline-asset behavior for a mobile-first app without a native app.
---
 
## 4. Integrations
 
**Nothing external is actually wired up beyond framework scaffolding.** `config/services.php` (Postmark/Resend/SES/Slack) and the S3 filesystem disk are stock Laravel boilerplate with no corresponding SDK packages installed in `composer.json`, and `.env.example` has the relevant AWS vars empty. There is:
- No payment gateway.
- No SMS provider.
- No exchange-rate API — FX rates are entered manually per transaction (`fx_rate` field), not fetched.
- **Mail is the one real integration**, configured via Hostinger SMTP for password reset and future member-invitation email.
**Error handling for integrations:** N/A — there's nothing integrated to fail. This should not be oversold as an "integration-heavy" project; it's an intentionally self-contained system.
 
---
 
## 5. Performance & Scale Considerations
 
- **Caching**: `database`-driver cache is used narrowly — help-page Markdown rendering, a parsed help manifest, and the admin-editable landing page content are cached (with explicit `Cache::forget()` invalidation on save). No caching of dashboard totals, wallet balances, or transaction aggregates — those are recomputed from the database on every request.
- **Queues**: `app/Jobs` doesn't exist. No `ShouldQueue` implementation, no `dispatch()` call anywhere in the app. `QUEUE_CONNECTION=database` is configured but unused — genuinely vestigial scaffolding, not a hidden async pipeline. The one background-ish task (recurring template materialization) runs synchronously via a daily cron command, not a queue worker (and indeed no queue worker process is ever started in the deploy setup).
- **N+1 avoidance**: Consistently good. Controllers eager-load with column-scoped selects (`with('wallet:id,name,currency_code')`), use `withSum()` for aggregate totals instead of per-row loops, and batch counterparty-balance computation across a whole list in one grouped query rather than one query per counterparty. No N+1 pattern was found in the controllers reviewed.
- **A genuine, if currently minor, performance gap**: the wallet-balance and counterparty-balance aggregation queries filter by `wallet_id IN (...) AND occurred_on <= ?` but there's no composite `(wallet_id, occurred_on)` index — only the single-column FK index exists. On a shared-hosting MariaDB instance this will degrade as transaction history grows into the tens of thousands of rows per household. Worth flagging privately; not urgent at current likely scale (a single family's transaction volume).
- **Built for a specific real load profile, not enterprise scale**: this is engineered correctly for "a handful of households, thousands of transactions per household," not for high-concurrency multi-tenant SaaS. That's an appropriate scope match to the actual client, not a shortcoming — but it should be described accurately (small-scale correctness, not load-tested at scale).
- No opcache tuning beyond Hostinger's PHP-FPM defaults; production caches (`config:cache`, `route:cache`, `view:cache`, `event:cache`) are baked into the deploy pipeline.
---
 
## 6. Testing
 
**Framework:** Pest v4 (PHPUnit 12 underneath). 68 test files: 64 Feature, 3 Unit, 1 Browser (Pest v4's new browser-testing feature, used for one wallet-deletion confirmation flow).
 
**Result at time of audit:** 381 tests, 3,208 assertions, all passing, ~12.6s run time.
 
**What's covered well:**
- Every controller has a corresponding feature test file.
- All 9 policies are exercised, either through a dedicated `tests/Feature/Policies/*` file (Category, Counterparty, Transfer, Wallet) or indirectly via `assertForbidden()` checks in the relevant controller test (Budget, Loan, RecurringTemplate, Transaction, Household member management).
- **Multi-tenant isolation is explicitly tested**, not just assumed: `tests/Feature/HouseholdScopingTest.php` unit-tests the `BelongsToHousehold` trait directly against a synthetic fixture model, and both `WalletControllerTest` and `TransactionControllerTest` have explicit "a user cannot see/update a resource from another household" cross-tenancy regression tests.
**Honest gaps:**
- Only Wallet and Transaction controllers have an explicit cross-household ("household B can't touch household A's data") test at the controller level. Budget, Loan, Category, Counterparty, Transfer, and RecurringTemplate rely on the shared trait being correct rather than each resource re-proving isolation independently.
- No dedicated model-level unit tests for `HouseholdUser` or `User`.
- No load/performance testing (consistent with a project this size).
Coverage is genuinely solid for a one-month solo build, not thin — this is honest, not inflated.
 
---
 
## 7. Security Posture
 
**Authentication:** Laravel Fortify, with custom `CreateNewUser` and `ResetUserPassword` actions. Registration, password reset, email verification, and two-factor authentication (TOTP, with password confirmation) are all enabled. **2FA is opt-in per user, not enforced** — no middleware requires it before a user can access financial data. Login is rate-limited (5/min, keyed by email+IP); the 2FA challenge step is separately rate-limited.
 
**Authorization:** All 9 policies (one per household-owned model, plus a stricter `HouseholdPolicy` for ownership actions like inviting/removing members) are confirmed wired into routes via middleware — not defined-and-forgotten. Role model is `owner`/`editor`/(`viewer`, scaffolded but not exposed in the current UI) per household member.
 
**Multi-tenant data isolation:** The core mechanism (global scope + creating-hook) is sound and tested (§6). One flagged gap for private review: `app/Support/CounterpartyMerger.php` does raw `DB::table()` writes (bypassing Eloquent's global scope) to bulk-repoint records from a merged counterparty to a survivor, filtered only by `counterparty_id` — with no explicit assertion inside that class that the absorbed and survivor counterparties belong to the same household. It's currently safe because the one caller resolves both IDs through household-scoped Eloquent queries first (so a cross-household ID would 404 before reaching the merger), but the safety is caller-enforced, not built into the class itself. **Severity: low-medium, defense-in-depth gap** — flagging for your own review, not for public disclosure.
 
**Input validation:** `household_id` is never a validatable/settable field on any Form Request — it can only be set server-side via the tenancy trait, so it isn't attacker-controllable through mass assignment. Money fields are validated as positive integers. Foreign-key fields on household-scoped tables use `Rule::exists`, which (per §3) doubles as a tenancy check.
 
**Mass assignment / XSS:** Models use explicit `#[Fillable([...])]` allowlists (Laravel's `$guarded = ['*']` default otherwise applies). `v-html` usage in Vue components (7 sites) was checked — all resolve to either framework-generated content (pagination labels, a QR code SVG from Fortify) or server-rendered Markdown from static repository files reached via a manifest-validated slug (not user path input). One item worth a manual double-check, not confirmed as a vulnerability: whether the admin-editable landing-page content could ever reach a raw-HTML render path via an injected translation key — the audit did not find a direct raw render of admin-submitted body copy, but flagged it as worth a second look.
 
**PII/secrets in the repo:** `.env` is correctly gitignored. No hardcoded API keys, tokens, or credentials found in application code. **One real finding**: `database/seeders/DatabaseSeeder.php` and `database/seeders/DemoDataSeeder.php` hardcode the developer's real personal email address (`arifhossen.dev@gmail.com`) as the seeded account. Low severity (it's the developer's own address, not a third party's), but it is a real, identifiable value committed to version control — flag before making the repo public.
 
No TODO/FIXME/HACK comments related to security or auth debt were found anywhere in the codebase (a clean grep across `app/`, `resources/js/`, `config/`, `database/`, `routes/`).
 
---
 
## 8. Complexity & Difficulty Assessment
 
**What was genuinely hard to get right:**
- Designing the five-type transaction model (income/expense/loan_given/loan_received/loan_repayment) so that loans don't corrupt income/expense totals, while still letting a loan repayment optionally carry a reporting category — this required real domain modeling against a messy real-world spreadsheet, not just CRUD scaffolding.
- Building multi-tenancy that's safe by construction (global scope + auto-fill) rather than safe by discipline (remembering to add `->where('household_id', ...)` everywhere) — this is the kind of decision that separates competent Laravel work from a junior developer's approach, where cross-tenant leaks via a forgotten `where` clause are a very common real-world bug class.
- Deriving all balances from the transaction ledger instead of caching them, which avoids an entire class of reconciliation bugs but requires disciplined, batched aggregation queries to stay performant — done correctly here (§5).
- Bilingual support with numeral normalization (Bangla digits accepted on input, normalized to Western digits, displayed consistently) — a genuinely easy thing to get subtly wrong (the exact bug the original spreadsheet had).
- A real zero-downtime deploy pipeline on shared hosting with no Node toolchain on the server itself — building assets externally and rsyncing them, atomic release swaps — is more engineering effort than most freelance projects at this budget tier receive.
**What a less experienced developer would likely have gotten wrong:**
- Storing a `balance` column on wallets and fighting drift bugs forever.
- Filtering household data per-query instead of at the model layer, leading to at least one forgotten scope somewhere.
- Using floats or `decimal` PHP arithmetic for money instead of integer minor units.
- Skipping the cross-household regression tests entirely (many solo projects under time pressure test happy paths only).
- Not testing 2FA/rate-limiting/authorization denial paths at all.
---
 
## 9. Portfolio-Worthy Features
 
1. **Structural multi-tenant data isolation via Eloquent global scope + auto-fill trait, proven by dedicated regression tests.** Client explanation needed: light — "the system is built so it's structurally impossible for one family's data to leak into another's view, and we have tests that specifically try to break that and fail."
2. **Ledger-derived balances (never-stored, always-computed) for wallets and per-person loan balances.** Client explanation needed: light-medium — "your balance is never a number that can silently drift out of sync with your transaction history, because it's recalculated from the history itself every time."
3. **Domain-correct loan/receivable modeling that fixes the client's original spreadsheet's core defect** (loans mis-classified as income/expense). Client explanation needed: light — this is the actual origin story of the project and is easy for a non-technical reader to understand because it maps directly to their pain point.
4. **Integer-minor-unit money handling with a dedicated cast, avoiding floating-point rounding bugs in a finance app.** Client explanation needed: medium — requires briefly explaining why floats are dangerous for money, but is a strong "this developer understands finance software specifically" signal to a technical reviewer.
5. **Zero-downtime atomic-release deploy pipeline built for budget shared hosting (no Docker/Kubernetes/cloud infra available).** Client explanation needed: medium — good talking point for "I can deliver production-grade engineering practices even under real-world budget/infra constraints," which is a relevant differentiator for freelance clients who assume they can't afford "proper" DevOps.
6. **Bilingual (Bangla/English) UI with numeral normalization**, solving a real, specific, previously-undetected data-corruption bug from the source spreadsheets. Client explanation needed: light — very concrete, relatable story.
---
 
## 10. Confidentiality / Disclosure Flags
 
**Must NOT be shown publicly, as-is:**
- `database/seeders/DatabaseSeeder.php` and `database/seeders/DemoDataSeeder.php` — hardcode the developer's real personal email address. Review/scrub before any public repo exposure or screen recording that shows seeded data.
- `docs/REQUIREMENTS.md` and `docs/SPEC.md` — contain the actual client's real financial category names, real named counterparties (specific first names: Tarik, Joni, Kotha, Apu, Abba, Ma, Shoyeb), a real investment provider name (IDLC), and Bangla-language business context that could plausibly identify the client if shared verbatim. Do not paste these documents into a public portfolio; summarize only.
- Any screenshots/demo recordings using `DemoDataSeeder` output should be checked for realistic-looking names/amounts that could be mistaken for real client data, even though it's seeded/fake.
- `.env` (not committed, correctly gitignored) — do not paste its contents anywhere, including to an AI assistant, without stripping values first.
**Borderline (could identify the client even without naming them):**
- The specific list of wallet types (bKash, Nagad, Rocket, multiple banks, USD digital wallets) combined with "Bangladesh," "Hostinger shared hosting," and "family finance" is a fairly narrow combination — probably fine for a portfolio piece since it's genuinely descriptive of the product, but avoid combining it with any other identifying detail (exact go-live date, domain name, real counterparty names).
- The exact monthly-workbook history detail ("10 monthly workbooks, Nov 2023 → Apr 2026") is specific enough that combined with other details it narrows down to one person's records — fine to describe the pattern generically ("the client had ~2.5 years of monthly spreadsheet history") without the exact date range if you want extra caution.
---
 
## 11. Open Questions For Me
 
These could not be determined from the code and would change how this should be presented:
 
- **Was this deployed to production and is it currently in active use?** The deploy pipeline and `docs/DEPLOY.md` runbook exist and look real, but nothing in the codebase itself proves it was actually run against Hostinger production, or is still running today.
- **How many households/users does it actually run for?** Code supports multi-household architecture generically, but the actual usage is presumably just the one client's family. Worth stating accurately rather than implying multi-tenant SaaS scale.
- **What was the client relationship/engagement structure?** Was this a fixed-price project, hourly, ongoing retainer? Is work still continuing past the last commit (2026-06-04)?
- **Was there a measurable outcome for the client** (e.g., "client can now see accurate balances," "eliminated hours per month of spreadsheet reconciliation")? This isn't inferable from code and would be the strongest client-facing hook.
- **Is the domain/custom hosting live, and is there a public or demo URL** that could be shown (with client permission) or should this stay purely descriptive/screenshot-based?
- **Phase 2 items** (recurring entries UI, budgeting) — SPEC.md defers some features to "Phase 2"; has any of that shipped since the last commit captured in this audit, or is this snapshot the full current state?
- **Client-facing name for the product** — is "Hisab" the name the client actually uses/knows it by, or purely an internal working title that shouldn't appear in a portfolio at all?
 
