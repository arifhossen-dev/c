# Amader Kaj Finance — Technical Brief
 
> Raw technical findings only, extracted from the codebase at commit range `2026-05-01` → `2026-08-13` (122 commits). No marketing language — translate separately for client-facing use.
 
---
 
## 1. What This System Actually Does
 
This is a bookkeeping and accounting web app for a small Bengali savings-and-loan cooperative ("সমিতি" / *samity*) called **Amader Kaj**. Three business partners run the cooperative together, and a set of members pay in monthly subscriptions ("chanda"), hold fixed deposits, and take out loans from the pooled fund. The partners also occasionally borrow money from outside lenders to top up the fund ("external loans").
 
The app replaces a manual Excel workbook (`basefiles/Chanda.xlsx`) that the partners were using to track:
- Member contributions, fixed deposits, and withdrawals
- Loans given to members (with profit/interest, not conventional bank interest — framed as Islamic-style profit-sharing)
- External borrowings taken by the partners themselves
- Daily cash-in/cash-out ledger entries
- Year-end profit distribution across the three partners and interest/profit accrual to members
- Balance sheet, profit & loss, and per-member/per-loan statements, as printable PDFs, in Bengali or English
Who uses it: the three partners themselves (no separate "staff" or "member" login role exists — every authenticated user sees the full system). It solves the problem of manually reconciling a shared ledger by hand across three people, and of computing correct profit-sharing math consistently.
 
This looks like the developer's own venture (see §11) rather than a third-party client engagement — the seeded partner names (`Arif`, `Akhi`, `Joni`) match the git author identity.
 
---
 
## 2. Architecture
 
- **Style:** Single-tenant modular monolith. Laravel 13 (PHP 8.4) backend + Inertia v3 + React 19 (TypeScript) SPA-style frontend, server-rendered routing via `Inertia::render()`. No REST/JSON API layer, no separate frontend deployment — one codebase, one deploy.
- **No multi-tenancy** — this serves exactly one cooperative. There is no tenant model, no tenant-scoping middleware, no per-tenant database/schema. Not applicable to this build.
- **Layered structure**, explicitly enforced by Pest architecture tests ([tests/Architecture/LayeringTest.php](tests/Architecture/LayeringTest.php)):
  - `Routes → Controllers → Form Requests → Services → Eloquent Models`
  - `arch()` rules assert: **models never call services**, **controllers never touch `DB::` or query builders directly**, **all Finance services must use the `Money` helper** (bcmath wrapper, see §3 below). This is a static, CI-enforced boundary, not just a convention in a doc.
- **Service layer** ([app/Services/Finance](app/Services/Finance), [app/Services/Members](app/Services/Members)): ~18 single-purpose classes — calculators (`ChandaProfitCalculator`, `FixedDepositProfitCalculator`, `LoanProfitCalculator`, `ExternalLoanInterestCalculator`, `CashOnHandCalculator`, `PartnerCapitalCalculator`), report builders (`BalanceSheetReport`, `ProfitLossReport`, `ActiveLoansSummaryReport`, `MemberProfitProjectionReport`), ledgers/feeds (`LoanLedger`, `MemberLedger`, `TransactionsFeed`, `TodaysActivityFeed`, `ThisMonthCollections`), and one multi-step transactional action (`YearCloseAction`).
- **DTOs** ([app/DTOs/Finance](app/DTOs/Finance)) — readonly value objects returned by services to controllers (`BalanceSheet`, `ProfitLoss`, `LoanLedgerData`, `MemberLedgerData`, `YearClosePreview`, etc.) rather than passing raw Eloquent collections/arrays to Inertia views.
- **No queue/job usage** — `app/Jobs` doesn't exist. `QUEUE_CONNECTION=database` is configured (Laravel default) but nothing is actually dispatched async. All work is synchronous request/response.
- **No caching layer** — `CACHE_STORE=database`. Every dashboard/report figure is computed live from entity rows on every request, by explicit design (see §5, §3 "Key principle" in [docs/implementation.md](docs/implementation.md)).
### Database design notes
- All money columns: `DECIMAL(14,2)` (BDT precision), rates `DECIMAL(6,4)` / `DECIMAL(5,2)`. No floats used for money anywhere in schema or code.
- **No soft deletes anywhere** (`grep SoftDeletes` = 0 hits). Financial rows use a custom **`Voidable` trait** instead ([app/Models/Concerns/Voidable.php](app/Models/Concerns/Voidable.php)): sets `voided_at`/`voided_by`/`void_reason` columns, row stays in the table permanently, `scopeNotVoided()` filters it from active queries. This is a deliberate audit-trail decision — financial records are never hard- or soft-deleted, only marked void and struck through in the UI.
- **Human-readable auto-numbering** via a `HasAutoNumber` trait + `NumberSequence` support class ([app/Support/NumberSequence.php](app/Support/NumberSequence.php)): generates IDs like `LON202600123` (prefix + year + zero-padded counter), allocated with `SELECT ... FOR UPDATE` row locking inside a DB transaction against a `number_sequences` table — this is a real concurrency-safety mechanism, not a naive `MAX(id)+1`.
- **Audit logging** via `spatie/laravel-activitylog` — 13 of the ~16 models use the `LogsActivity` trait with `logOnlyDirty()` (only changed attributes recorded).
- No polymorphic relations found in the schema — all relations are conventional `belongsTo`/`hasMany` with explicit FKs (e.g. `Loan::member()`, `LedgerEntry::byPartner()`, `ExternalLoan::lenderPartner()` / `receivedByPartner()`).
- Indexing is deliberate but modest: composite `(fiscal_year_id, transacted_on)` and single-column `status`/`type` indexes on the transaction tables ([database/migrations/2026_05_02_120608_create_ledger_entries_table.php](database/migrations/2026_05_02_120608_create_ledger_entries_table.php)) — sized for hundreds/thousands of rows, not millions.
---
 
## 3. Notable Technical Decisions
 
- **bcmath everywhere for money** — [app/Support/Money.php](app/Support/Money.php) wraps `bcadd`/`bcsub`/`bcmul`/`bcdiv`/`bccomp` at fixed scale 2, and normalizes `string|float|int` inputs to strings before arithmetic. This avoids float rounding errors in financial calculations — a correctness detail a less careful build would skip (`0.1 + 0.2` class bugs). Enforced by the arch test in §2.
- **"Entities are the source of truth" principle** — explicitly stated in [docs/implementation.md](docs/implementation.md) §1: no denormalized running balances/totals are stored; every number (cash on hand, member balance, partner capital, P&L) is recomputed from raw transaction rows on each request. This trades some performance for correctness guarantees and eliminates an entire class of "stale cached total" bugs. The doc explicitly says caching is only allowed once a calculation is "proven correct and demonstrably slow — never before."
- **Deterministic rounding rule for profit splits** — [app/Services/Finance/YearCloseAction.php](app/Services/Finance/YearCloseAction.php): each partner's share is computed as `net × share_percent / 100` for the first N−1 partners (ordered by `id ASC`), and the *last* partner absorbs the remainder (`net − Σ(prior shares)`), guaranteeing the sum always equals net profit exactly to the taka, with no leftover fraction lost to rounding. This is a deliberately engineered detail, not an accident.
- **Idempotency guards on financial close operations** — `YearCloseAction::execute()` checks `is_closed` and an existing `ProfitDistribution` row before writing, and the migration also carries what the code comments call a "belt-and-braces" unique constraint against concurrent double-close requests.
- **Reconciliation acceptance test against the real source workbook** — [tests/Feature/Reconciliation/WorkbookReconciliationTest.php](tests/Feature/Reconciliation/WorkbookReconciliationTest.php) loads opening balances derived from `basefiles/Chanda.xlsx`, replays a month of fake activity, and asserts the app's computed dashboard/balance-sheet/statement numbers match what a partner would get by hand-tallying the workbook, "to the taka." The test file's own comment: *"If this test fails, partners cannot trust the app with real data."* This is a strong, non-obvious quality gate most small apps skip entirely.
- **Void, never delete, for financial rows** (§2 above) — correctness/audit decision that shows senior judgment about what "delete" means for money records.
- **Concurrency-safe sequence numbering** via row locking (§2) rather than `count()+1`, which is the naive approach that breaks under concurrent writes.
---
 
## 4. Integrations
 
- **None active in code.** No payment gateway, no SMS provider, no external HTTP API calls found anywhere in `app/` (`grep -rn "Http::\|Mail::\|Notification::"` returns nothing meaningful — only an internal `URL::route()` call).
- `config/services.php` has the stock Laravel starter-kit stubs for Postmark, Resend, SES, Slack — none are wired to actual code paths, just unused config scaffolding from the framework skeleton.
- Mail is only used for Fortify's built-in flows (password reset, email verification) via whatever `MAIL_MAILER` is set to in the environment; local dev defaults to `log` driver (mail written to log file, not sent).
- **No error handling for external-service failures exists, because there are no external service calls to handle.** Not a gap — there's simply nothing to integrate with yet. Phase 2 items in [docs/implementation.md](docs/implementation.md) §14 mention SMS as a future item (member `phone` column already exists on the schema for this).
---
 
## 5. Performance & Scale Considerations
 
- **Built for correctness at small scale, not high load.** This is explicitly a 3-user internal tool (three partners' phones), not a public multi-user SaaS. Every report/dashboard figure is computed live (§2, §3) — deliberately, not as an oversight, but it does mean growth to thousands of members/transactions would eventually need caching or materialized aggregates. The docs anticipate this ("optimisation by caching allowed only once proven slow").
- No Redis, despite `phpredis` being present in `.env.example` as an unused option — cache/session/queue all run on `database` driver.
- No background job/queue usage despite `QUEUE_CONNECTION=database` being configured — everything runs synchronously in the request cycle, including PDF generation (dompdf) for balance sheet, P&L, receipts, loan/member statements.
- Eager loading (`->with()`) is used in 23 places across controllers/services — evidence of *some* N+1 awareness, but not exhaustively verified against every list/index endpoint in this pass.
- Deploy target is a single shared Hostinger hosting account (see [docs/deployment.md](docs/deployment.md)) — PHP 8.4 + MySQL, no separate app server, no CDN, no load balancer. This is a small-scale production deployment, consistent with the 3-partner user base.
---
 
## 6. Testing
 
- **Framework:** Pest 4 (`pestphp/pest` + `pestphp/pest-plugin-laravel`), run via `./vendor/bin/pest` in CI ([.github/workflows/tests.yml](.github/workflows/tests.yml)).
- **Volume:** ~85 test files, approximately 447 `test()`/`it()` blocks (grep count — not a verified passing-test count from a live run in this pass).
- **Composition:**
  - `tests/Unit/Finance/*` — one test file per calculator/report service (14 files), pure logic tests.
  - `tests/Unit/Support/*` — `Money`, `NumberSequence`, `BengaliNumerals`, `PartnerShareSplitter` helpers.
  - `tests/Feature/*` — full HTTP-cycle tests per controller/route group: Members, Loans, FiscalYears, Contributions/FixedDeposits/Withdrawals/LedgerEntries (each with Store/Update/**Void** test files — the void path is explicitly tested, not just create/update), ProfitDistributions (including a dedicated `ForfeitTest`), PartnerLedger, ExternalLoans, Reports (PDF rendering, receipts, statements), Settings (profile, security/2FA), and the full Fortify auth suite (login, registration, password reset/confirmation, email verification, 2FA challenge).
  - `tests/Architecture/LayeringTest.php` — Pest arch tests enforcing the layering rules in §2.
  - `tests/Feature/Reconciliation/WorkbookReconciliationTest.php` — the workbook-vs-app reconciliation acceptance test (§3).
  - `tests/Feature/SchemaSmokeTest.php`, `Seeders/DemoDataSeederTest.php` — schema/seeder sanity checks.
- **What's covered well:** the financial calculation engine (every calculator has a dedicated unit test), the void/audit workflow, PDF report generation, 2FA/auth flows, and — notably — a real reconciliation test against source-of-truth numbers.
- **What's NOT verified in this pass (be honest):** I did not execute the suite to confirm it currently passes, and did not check code-coverage percentages — only static counts. No visible browser/E2E tests (Pest 4 supports browser testing via a plugin; not present here). No load/performance tests, unsurprisingly given scale (§5).
---
 
## 7. Security Posture
 
- **Auth:** Laravel Fortify, backend-only (no Breeze/Jetstream views) — [app/Providers/FortifyServiceProvider.php](app/Providers/FortifyServiceProvider.php), [app/Actions/Fortify](app/Actions/Fortify). Features enabled: password reset, email verification, and **two-factor authentication (TOTP/QR + recovery codes)** ([config/fortify.php:149](config/fortify.php)).
- **Login throttling:** `RateLimiter::for('login', ...)` — 5 attempts/minute, keyed by transliterated username + IP ([app/Providers/FortifyServiceProvider.php:84-87](app/Providers/FortifyServiceProvider.php)); a separate `two-factor` rate limiter also exists. Password confirmation route is throttled `6,1` ([routes/settings.php:22](routes/settings.php)).
- **Authorization: none at the application level.** There are **no Policy classes** (`find app -iname "*Polic*"` returns nothing), no `Gate::` calls, no `->authorize()` / `can()` calls anywhere in controllers. Every route only requires `auth` + `verified` middleware — any authenticated user (i.e. any of the three partners) has full read/write access to every member, loan, ledger entry, and report in the system. For a 3-trusted-partner internal tool this is a reasonable, deliberate simplification, **but it means there is no per-role restriction if this app were ever opened to more users** (e.g. member self-service login) without adding an authorization layer first.
- **Validation:** consistently done via dedicated Form Request classes (~25 of them in [app/Http/Requests](app/Http/Requests)), not inline `$request->validate()` in controllers — one per entity/action (Store/Update/Void variants for transaction entries).
- **Multi-tenant data isolation:** not applicable — single-tenant system, no tenant boundary exists to test.
- **CSRF:** standard Laravel/Inertia CSRF protection (no evidence of it being disabled).
- **Genuine flags for private review (not fixed here, just flagged):**
  - No authorization layer at all (see above) — fine today, a real gap if the user base ever grows past the 3 partners.
  - `SESSION_ENCRYPT=false` in both `.env.example` and the production `.env` template in [docs/deployment.md](docs/deployment.md) — session cookie contents aren't encrypted (though Laravel's default session driver+signing still applies; this is worth a second look, not a confirmed vulnerability).
  - Production `.env` template in `docs/deployment.md` is a plaintext runbook containing the *shape* of every credential (DB, mail) that a human fills in manually over SSH — process risk (human error, secrets in shell history/notes) more than a code vulnerability, but worth noting since deployment secrets aren't managed through a secrets manager.
---
 
## 8. Complexity & Difficulty Assessment
 
**Genuinely hard parts:**
- Getting the profit-sharing arithmetic exactly right: partner share splitting with no rounding leakage, member chanda/fixed-deposit profit accrual pro-rated by day count within a fiscal year, forfeiture handling that reroutes amounts into a "retained" pool without breaking the P&L identity, and keeping all of it deterministic and idempotent through a "close the fiscal year" wizard that partners will only ever run once a year (so bugs are expensive to discover late).
- Designing an audit-safe transaction model (void-not-delete, auto-numbering, activity log) that a bookkeeper would actually trust, while keeping the codebase small enough for one developer to maintain.
- The reconciliation acceptance test itself — translating a real, historical Excel workbook's numbers into a fixture and proving the new system reproduces them "to the taka" is unglamorous, tedious, high-stakes work that's easy to skip and expensive to get wrong.
- Bengali/English bilingual output (locale-aware PDFs, Bengali numeral formatting — [app/Support/BengaliNumerals.php](app/Support/BengaliNumerals.php)) without a full i18n framework in place yet (explicitly deferred to a later milestone per the docs).
**What a less experienced developer would likely have gotten wrong:**
- Using floats for money instead of bcmath/decimal-string arithmetic.
- Hard-deleting or soft-deleting financial rows instead of a void pattern, breaking the audit trail.
- Naive `MAX(id)+1` or `count()+1` numbering that breaks under concurrent inserts.
- Storing denormalized running balances that drift out of sync with the underlying transactions (a classic bookkeeping-app bug class), instead of computing everything live from the ledger.
- Letting rounding errors accumulate in the profit-split so shares don't sum to net profit.
- Skipping the reconciliation test entirely and just trusting the calculators looked right in isolation.
---
 
## 9. Portfolio-Worthy Features
 
1. **Deterministic profit-distribution engine with idempotent fiscal-year close** ([app/Services/Finance/YearCloseAction.php](app/Services/Finance/YearCloseAction.php)) — computes partner profit shares and member interest/chanda accrual, supports partial forfeiture, guarantees no rounding leakage, and can't be double-run. *Explanation needed for a non-technical client:* moderate — the "why" (three-way profit splitting must add up exactly, and must be safe to click twice) is intuitive; the rounding-remainder mechanism needs one sentence of translation.
2. **Void-based audit trail for all financial transactions** (Voidable trait + activity log) — nothing is ever truly deleted; corrections are transparent, reversible-looking, and logged. *Explanation needed:* low — "nothing gets erased, only marked cancelled with a reason" is self-evidently a trust feature for a client handling other people's money.
3. **Excel-to-app reconciliation acceptance test** — proves the system reproduces a real historical ledger's numbers to the exact currency unit before being trusted with live data. *Explanation needed:* low-to-moderate — "we tested it against your real old spreadsheet and it matched to the taka" is a strong, easily understood trust statement.
4. **bcmath-based money arithmetic layer with a CI-enforced architectural rule** that all financial services must use it — prevents an entire bug class (floating-point rounding errors in money) at the type-system/lint level, not just by convention. *Explanation needed:* moderate — needs the "computers can't do exact decimal math with regular numbers" framing.
5. **Bilingual (Bengali/English) PDF reporting** — balance sheet, P&L, member/loan statements, and receipts, each renderable in either language via a locale-switching middleware, including Bengali numeral formatting. *Explanation needed:* low — visually obvious once shown a sample PDF in both languages.
6. **Concurrency-safe human-readable document numbering** (`LON202600123`-style IDs via row-locked sequence allocation) — avoids duplicate/skipped numbers under simultaneous entry. *Explanation needed:* moderate-to-high for a non-technical audience; better shown than explained (e.g. "two partners can enter loans at the same time and never get the same number").
---
 
## 10. Confidentiality / Disclosure Flags
 
**Must NOT be shown publicly:**
- [basefiles/Chanda.xlsx](basefiles/Chanda.xlsx) — appears to be the real source workbook with actual member names, account numbers, and historical financial figures for the cooperative. Do not screenshot, upload, or reference its contents anywhere public.
- The production `.env` file at the repo root exists locally but is correctly gitignored (`.gitignore:18`) and not committed — confirmed not present in git history for this check, but **do not paste its contents anywhere**, and double-check before ever sharing a screen recording or terminal session from this machine.
- [docs/deployment.md](docs/deployment.md) contains real infrastructure details: production domain (`finance.amaderkaj.com`), Hostinger-specific paths, and a full `.env` *template* (structure only, no live secrets in the file itself, but it names real env var expectations and the real domain/site naming scheme). Also references 4 other personal domains (`arifhossen.pro`, `hisab.arifkotha.com`, `gotibidhi.com`, `shaluka.com`) tied to the same developer's other projects/identity — treat as personal, not for a portfolio writeup unless you want to disclose your own domain portfolio.
- [database/seeders/DemoDataSeeder.php](database/seeders/DemoDataSeeder.php) generates synthetic demo data (safe), but confirm no real seeded data is currently loaded in any environment you'd screenshot from.
- Git history includes a raw conversation transcript file at repo root: `2026-05-02-101438-the-first-conversation-with-claude.txt` — worth checking its contents privately before any public repo/portfolio exposure, since it may contain informal discussion of real business numbers or decisions.
**Borderline (might identify the client/business even without naming it):**
- The specific business model — a 3-partner cooperative charging 5% "chanda" and paying member profit-shares in BDT, tied to a `.com.bd`-adjacent domain and Bengali branding — is fairly identifying if described in detail alongside the domain name. Consider whether to name `amaderkaj.com` at all in public portfolio copy, or to genericize as "a member-owned savings cooperative" without the real domain/brand.
- Partner first names (Arif, Akhi, Joni) appear in seeders/fixtures/docs — real names of real financial stakeholders. Don't reproduce these in public copy.
---
 
## 11. Open Questions For You
 
- **Is this your own venture, or a client's?** The seeded partner name "Arif" matches your own identity (git author `Arif Hossen`) — worth confirming explicitly for how you frame authorship/ownership in the portfolio (built *for* a client vs. built *by* you *as* a co-founder/partner).
- **Is it live in production today, and for how long?** [docs/deployment.md](docs/deployment.md) describes a full cutover plan from an "old manual git-clone deploy" to a new CI pipeline at `finance.amaderkaj.com` — was this cutover completed, and is the CI deploy pipeline (`.github/workflows/deploy.yml`) the one actually in use now?
- **How many members/transactions does it actually carry?** The reconciliation test uses fictional/placeholder opening numbers "chosen so the balance sheet identity holds" — I don't have the real scale (member count, transaction volume, BDT under management) from the code alone.
- **Was there a measurable outcome** (e.g. hours saved per month vs. the old spreadsheet, error corrections caught, etc.)? Not derivable from code.
- **Timeframe framing:** commit history spans 2026-05-01 to 2026-08-13 (~3.5 months, 122 commits) — confirm if that's the full build window you want to cite, or if there's earlier/later work not in this repo.
- **The "why" companion doc** referenced in [docs/implementation.md](docs/implementation.md) header (`~/.claude/plans/this-is-actually-a-encapsulated-wind.md`) isn't in this repo — if it has requirements/decision context you want folded into the portfolio narrative, you'll need to pull it in separately.
- **Should the real domain/brand name (`amaderkaj.com`) be used at all in public portfolio materials**, given the borderline-identifying concern in §10? Your call.
 
