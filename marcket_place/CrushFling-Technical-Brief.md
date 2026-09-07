# CrushFling — Technical Brief
 
**Codebase:** `crushfling` (Laravel 8 / PHP 8.0)
**Analysis date:** 2026-08-28
**Git history:** 887 commits, 2021-04-26 → 2023-09-10 (~2.5 years of active, incremental development)
**Role confirmed:** sole developer (single commit author pattern throughout history — not independently verified against GitHub contributor list)
**Client / business type:** unclear, needs your input
**Original ask:** unclear, needs your input
**Deployment status:** unclear, needs your input (no `.env`, no CI/CD, no deploy scripts found in repo — cannot confirm production deployment from code alone)
 
---
 
## 1. What This System Actually Does
 
This is a web-based social discovery / dating-adjacent directory site. Users create a profile (age, gender, "looking for" orientation, country, bio, profile photo, and up to 3 interest tags), then browse a filterable grid of other users' profiles — filterable by gender/orientation, age range, country/location, interest tag, and by which external social app the user connects through.
 
The distinctive mechanic: instead of building in-app messaging, the platform links each profile out to the user's account on an external social/chat app (Instagram, Snapchat, Kik, WhatsApp, etc. — configurable, not hardcoded). Each profile shows a QR code and a "connect on [App]" button that deep-links to the user's handle on that external service. The site also has a public blog (posts), a lightweight CMS for static pages ("Documents" — About, Privacy Policy, Terms, etc.), a like/dislike reaction system on profiles, a comment wall on each profile, a user-reporting/moderation flow, an ad-slot system, and a "boost"/"scroll-up" promoted-profile feature.
 
**Who uses it:**
- **End users** — people building a dating/social profile to be discovered by others filtering by orientation, age, country, or shared interest, and to be contacted off-platform via their existing social app.
- **Site admin (the operator)** — a custom admin panel (AdminLTE-based) for managing users, countries, interests, connectable social apps, blog posts, SEO metadata, static pages, ad placements, promoted profiles, and moderation reports.
**Problem solved:** gives users a searchable/filterable discovery surface for a niche audience, without the platform having to build/host chat infrastructure — it hands the conversation off to whatever app the two people already use.
 
---
 
## 2. Architecture
 
**Structure:** Classic Laravel monolith — server-rendered Blade views + Alpine.js for interactivity, Tailwind CSS, Laravel Mix build pipeline. No SPA, no separate frontend build, no `api.php` routes beyond the stock Sanctum/Passport stub (`/api/user`). This is not a multi-tenant system — single tenant, single admin operator.
 
**Patterns in use:**
- **Query Filter pattern** — `App\Filter\QueryFilter` (abstract base) + `App\Filter\UserFilters` reflectively maps incoming query-string params (`for`, `app`, `s`, `age`, `loc`) to filter methods and chains them onto the Eloquent query. This is how the whole "browse/filter directory" page works. ([QueryFilter.php](app/Filter/QueryFilter.php), [UserFilters.php](app/Filter/UserFilters.php))
- **Fortify Actions pattern** — Laravel Fortify handles auth (registration, login, password reset), with the app supplying custom `Actions\Fortify\*` classes and custom `LoginResponse`/`RegisterResponse` to redirect admins to the admin dashboard vs. regular users to the home page.
- **Trait-based shared behavior** — `Likeable` trait (`app/Models/Helper/Likeable.php`) adds like/dislike/aggregate-vote-count behavior, applied to `User`.
- **Facade/Service wrapper for a 3rd-party SDK** — `App\Facade\Tinify` + `App\Service\TinifyService` wrap the Tinify image-compression SDK behind Laravel's facade pattern (though see §3 — this appears to be retired in favor of Intervention/Image, per the "Tinify removed" commit at the tip of history).
- **Static support classes** for cross-cutting concerns: `Support\Bunny` (BunnyCDN cache purge on file replace), `Support\QrCode` (SVG QR generation via BaconQrCode), `Support\Settings`, `Support\UrlParamSorting` (canonicalizes filter query-string param order — SEO-motivated, see §3).
- **Custom validation rules** as first-class objects: `IsAvatarNudeFree` (calls a 3rd-party moderation API), `UserName` (regex format rule), `MatchCurrentPassword`.
- **Route-model binding with custom keys** — `Interest`, `SocialMedium`, `Document` all override `getRouteKeyName()` to bind by `slug` instead of `id`, and `User` binds by `username` in profile routes.
**Database design notes:**
- **Soft deletes** used selectively (`users`, `interests` have `softDeletesTz()`; most child tables don't) — mixed/inconsistent soft-delete policy across the schema, not a deliberate audit-trail strategy.
- **No audit trail / activity log** — no `activity_log`-style table; changes to user records, moderation actions, etc. are not historically tracked.
- **Composite primary key** on the `interest_user` pivot (`[interest_id, user_id]`) instead of a surrogate `id` — correct, idiomatic choice for a pure many-to-many pivot with no extra pivot data.
- **Unique composite index** on `reactions` (`receiver_id, from_id`) enforced at the DB layer — correctly prevents a user from casting two likes/dislikes on the same profile; the app layer uses `updateOrCreate` against this same key, so the DB constraint and the app logic agree (good — this is a place a less careful implementation gets a race condition wrong).
- **No polymorphic relations** anywhere in the schema — comments, reactions, and reports are all hardcoded to `User`, not built as reusable polymorphic "commentable/reactable/reportable" traits, despite `Likeable` being structured as a trait. This is a real limitation if the client ever wants to extend likes/comments to posts or other entities.
- **`username` column has no unique index** — see §7, this is a genuine defect, not a design choice.
- Legacy schema drift: `SocialMedium::userMeta()` and `InterestSocialMediumController` reference a `UserMeta` model/relation that no longer exists in the codebase (the git history shows a `user_metas` table existed early on and was apparently folded into `users` directly). These are dead references now — see §9/§10 for handling.
---
 
## 3. Notable Technical Decisions
 
- **SEO-driven canonical query-string ordering** — `Support\UrlParamSorting` deliberately sorts filter query params into a fixed order before building `route()` URLs (used by `Interest::path()`, `SocialMedium::path()`), so `?for=x&app=y` and `?app=y&for=x` always canonicalize to the same URL. Combined with `UnknownUrlParamTo404` middleware (which 404s any request carrying a query param outside an explicit allow-list: `for`, `app`, `s`, `loc`, `age`, `sort`, `page`), this is a considered anti-duplicate-content strategy for a filterable-listing site — the kind of SEO hygiene a less experienced developer typically skips entirely, and it's paired correctly with `laravel-trailing-slash` in composer.json.
- **Image pipeline hardening** — every avatar upload is (a) validated for minimum pixel dimensions, (b) checked against a rules-based `IsAvatarNudeFree` rule before it's accepted, (c) resized into both a full and a square crop on upload, and (d) purged from BunnyCDN's edge cache on replace/delete so stale images don't linger on the CDN. That's four separate, correctly-sequenced concerns (validation → moderation → transformation → cache invalidation) around a single file upload — this is the kind of "boring but correct" plumbing that's easy to under-build.
- **Server-side IP geolocation cached in session** — `GetGeoCityFromIp` middleware calls an external IP geolocation API once per session and caches the result in the session object rather than calling it on every request, avoiding a per-request external HTTP call. See §7 for the header-spoofing caveat.
- **Migration deprecated in favor of a newer library**: `tinify/tinify` is still in `composer.json` and `App\Service\TinifyService` / `App\Facade\Tinify` still exist, but the most recent commit on the branch is literally titled "Tinify removed" — signals a mid-flight migration away from Tinify (likely toward `intervention/image`, also in composer.json) that wasn't fully cleaned up. Worth confirming with the repo owner what state this migration is actually in before describing image compression as a current feature.
- **Two-factor auth columns exist but appear unused** — `users` table has `two_factor_secret` / `two_factor_recovery_codes` (standard Fortify/Jetstream columns), but no controller, route, or view was found wiring up 2FA setup/challenge. This looks like unused Fortify scaffolding rather than a shipped feature — confirm before claiming 2FA as a feature.
---
 
## 4. Integrations
 
| Service | Purpose | Error handling |
|---|---|---|
| **Sightengine** (`api.sightengine.com`) | Nudity/content moderation on every avatar upload (`IsAvatarNudeFree` rule) | **None** — the HTTP call result is dereferenced directly (`->json()['summary']`) with no try/catch, no timeout, no fallback if the API is down or returns an unexpected shape. An API outage or malformed response would throw an uncaught exception on every profile-photo upload. |
| **BunnyCDN** (`api.bunny.net`) | Edge-cache purge when a user's avatar changes | **None** — fire-and-forget HTTP POST, response not checked, no retry, no logging of failure. If a purge fails, stale images silently persist on the CDN with no operator visibility. |
| **ip-api.com** (`pro.ip-api.com`) | IP → city/country lookup for the "user location" chip shown in the header | **Partial** — checks `isset($geo['city'])` before use, so a failed/empty response degrades gracefully to "no location shown" rather than crashing. No retry or timeout configured. |
| **Tinify** | Image compression (see §3 — appears to be mid-deprecation) | Constructor throws if API key is missing; no handling for API-level failures during actual compression calls. |
| **AWS S3** (`league/flysystem-aws-s3-v3`) | Configured as an available filesystem disk for avatar/media storage | Standard Flysystem — no custom error handling layered on top. |
| **Laravel Fortify** | Authentication (register/login/password reset), rate-limited login attempts | Built-in framework rate limiting (5/min by email+IP) via `RateLimiter::for('login', ...)`. |
 
**Overall pattern:** none of the three custom third-party integrations (Sightengine, Bunny, ip-api) have retry logic, circuit breaking, or structured failure logging. This is consistent with a small/solo-maintained project rather than one built for high external-dependency reliability — worth being direct about this if a client asks about resilience.
 
---
 
## 5. Performance & Scale Considerations
 
- **No caching layer in use for application data** — `config/cache.php` defaults to file-based cache (`.env.example` sets `CACHE_DRIVER=file`); no Redis-backed query or fragment caching found anywhere in the controllers/views searched.
- **No queue/job usage** — `QUEUE_CONNECTION=sync` in `.env.example`, and no `app/Jobs`, `app/Listeners`, or custom `app/Events` directory exists. The Sightengine moderation call, the Bunny purge call, and the ip-api call all happen **synchronously, inline, in the request/response cycle** — a slow or hanging third-party API directly slows down the user-facing request (e.g., profile save or first page load per session). This is the single clearest "prototype-scale, not production-hardened-for-load" signal in the codebase.
- **Eager loading is used correctly** in the main listing queries — `User::with('country', 'socialMedium')` on the home/browse page avoids the obvious N+1 on those two relations. However, `interests` and the like/dislike aggregate (`reactions()`, called per-profile via the `Likeable` trait) are **not** eager-loaded on listing pages, so a profile-grid page render risks an N+1 on interests and reaction counts if those are rendered per-card in the Blade views (would need to check the actual Blade templates to confirm whether this fires per-row).
- **Pagination is used throughout** (`paginate(36)`, `paginate(20)`, `paginate(10)`) rather than loading full tables — a basic but correct scale precaution.
- **No database read replicas, no sharding, no per-tenant partitioning** — none needed at this schema's apparent scale, and none attempted.
- **`reports` admin index does a `groupBy('user_id')` + `selectRaw('count(*) as total')`** across the whole reports table with no status/date bound beyond `pending()` — fine at low report volume, would need an index review (`user_id` + `status`) if report volume grew significantly.
**Bottom line:** built like a real, iteratively-developed product (pagination, eager loading on the hot path, SEO URL canonicalization) but not load-tested or architected for high traffic — no caching, no queues, no async offload of slow third-party calls.
 
---
 
## 6. Testing
 
- **Framework:** PHPUnit 9 (Laravel's default), configured correctly in `phpunit.xml` with separate `Unit` and `Feature` suites and an in-memory SQLite option available.
- **Actual coverage: effectively zero.** The only test files present are the two Laravel-generated boilerplate examples:
  - `tests/Feature/ExampleTest.php` — asserts `/` returns HTTP 200.
  - `tests/Unit/ExampleTest.php` — asserts `true === true`.
- No tests exist for registration, profile update/validation, the moderation/nudity-check flow, the filter query system, likes/reactions, comments, reports, admin CRUD, or any model. `database/factories` exists (Laravel scaffolding) but there is no evidence factories are exercised by real tests.
- **Be direct about this with the client-facing framing:** there is no automated regression safety net on this codebase. Any correctness claims about business logic are based on manual/code-read verification, not test evidence.
---
 
## 7. Security Posture
 
**Authentication:** Laravel Fortify — session-based auth, hashed passwords (`Hash::make`), login rate-limited to 5 attempts/minute keyed by email+IP. Standard, correctly wired.
 
**Authorization:** Extremely simple, single-tier model — `AdminMiddleware` checks `auth()->check() && in_array(auth()->user()->email, config('crushfling.admin'))`, where the admin list is a comma-separated email list from the `ADMIN_LIST` env var. There is **no roles/permissions table, no policy classes, no gates** — every admin has full, undifferentiated access to every admin action. Fine for a single-operator site; would need real RBAC before onboarding a second admin/staff account with restricted scope.
 
**Input validation:** Consistently uses Laravel's `Request::validate()` / Form Request-style inline rules across controllers, with custom `Rule` objects for username format and moderation. This is solid, idiomatic Laravel practice throughout — no evidence of raw, unvalidated user input being used in queries.
 
**Multi-tenant data isolation:** **Not applicable** — this is a single-tenant application (one operator, one admin email list, one dataset). There is no tenant/organization concept, no row-level tenant scoping, and therefore nothing to assess on that specific axis. Flag this explicitly if you're using this codebase as multi-tenant SaaS portfolio evidence — it isn't one, and shouldn't be described as one.
 
**Flagged issues (private list — do not fix, just noted for you):**
1. **`username` column has no database-level unique constraint.** In `database/migrations/2014_10_12_000000_create_users_table.php` line 18: `$table->string('username')->unique;` — missing the `()`, so this is a property access, not a method call. It silently does nothing; no unique index is created. Uniqueness is enforced only at the application/validation layer (`Rule::unique('users')`), which is vulnerable to a race condition (two near-simultaneous registrations with the same username could both pass validation and both insert). **This is a genuine, confirmed defect**, not a design tradeoff.
2. **No error handling around the Sightengine and BunnyCDN HTTP calls** (see §4) — not a security hole per se, but an uncaught-exception / unhandled-failure risk on user-facing upload flows.
3. **`GetGeoCityFromIp` middleware trusts client-controllable headers** (`HTTP_CLIENT_IP`, `HTTP_X_FORWARDED_FOR`, etc.) for IP detection without restricting to a configured trusted-proxy list, so the displayed "location" is spoofable by the client. Low impact (display-only, not used for access control), but worth knowing.
4. **Every avatar upload is transmitted to a third-party API (Sightengine)** for nudity screening. Not a vulnerability, but a privacy/data-flow fact worth disclosing if the client ever asks "where does our users' data go."
5. **Dead code referencing a nonexistent `UserMeta` relation/model** (`SocialMedium::userMeta()`, and `InterestSocialMediumController`'s use of `User::with('UserMeta')`, `femaleFirst()`, `filterBy()`, `genderFilter()` — none of these methods/relations exist on `User` today). Confirmed via `grep` that no route currently points at `InterestSocialMediumController`, so this is unreachable, not an active bug — but it would throw a fatal error immediately if any route were ever pointed at it again.
No hardcoded credentials, API keys, or secrets were found in tracked files; `.env` is correctly gitignored and not present in the working tree.
 
---
 
## 8. Complexity & Difficulty Assessment
 
**Genuinely hard/easy-to-get-wrong parts, done correctly here:**
- The **filter/query-string system** (`QueryFilter` + `UserFilters` + `UrlParamSorting` + `UnknownUrlParamTo404`) is a small, self-consistent mini-framework for turning arbitrary query params into safe, chainable Eloquent scopes while simultaneously canonicalizing URLs for SEO and 404-ing unexpected params. Building this coherently (rather than as a pile of `if` statements in the controller) is a design choice that pays off as the number of filters grows — three separate mechanisms (filter mapping, canonical ordering, param allow-listing) work together correctly.
- The **avatar upload pipeline** correctly sequences validation → third-party moderation → dual-size image transformation → old-file cleanup → CDN cache purge, and does so in both the public "edit my profile" controller and the separate admin "edit any user" controller, with the logic reasonably consistent between the two (this duplication is itself a minor maintainability smell worth naming — see below).
- **The reaction (like/dislike) system's unique constraint + `updateOrCreate` pairing** correctly prevents double-voting and duplicate rows — a spot where an inexperienced developer commonly ships a race condition or allows duplicate reaction rows to accumulate.
**Where a less experienced developer would likely have gotten it wrong:**
- Skipping the DB-level unique constraint entirely (rather than just mistyping it — see §7, item 1) and relying purely on app-level validation.
- Calling the nudity-moderation and CDN-purge APIs synchronously inside the request without ever considering queuing them — easy to not notice until an API is slow in production.
- Building the "connect via external social app" concept at all — routing users off-platform to Instagram/Snapchat/etc. instead of building in-app chat is an unusual product decision that has real technical consequences (QR code generation, per-app profile-URL templating via `SocialMedium::profilePath()`, per-app input rules like `is_phone`) that had to be threaded through registration, profile editing, and the admin panel consistently.
- The upload-and-resize logic is duplicated near-verbatim between `ProfileController::updateProfile()` and `Admin\UserController::validateData()` rather than extracted into a shared service — not wrong, but a shared trait/service would have been the more senior refactor.
---
 
## 9. Portfolio-Worthy Features
 
1. **Dynamic, SEO-canonicalized filter/discovery engine** — multi-axis profile filtering (gender/orientation, age range, country, interest, connected app) built as a reusable query-filter pattern, paired with URL canonicalization and strict query-param allow-listing to avoid duplicate-content SEO penalties. *Client explanation needed: low–medium — "search filters that always produce the same clean URL so Google doesn't see duplicate pages" is a one-sentence pitch.*
2. **Automated image moderation + CDN-aware upload pipeline** — every profile photo is validated, screened by a third-party nudity-detection API, resized into two variants, and old images are purged from the CDN edge cache on replacement. *Client explanation needed: low — "uploaded photos are automatically screened and optimized before they go live" is intuitive.*
3. **"Connect via your existing app" identity model** — instead of in-house messaging, profiles generate a QR code and deep link to the user's account on an external app of their choice (configurable per-app URL templates, phone-number vs. username input modes, per-app branding colors). *Client explanation needed: medium — worth walking through why this avoids building/maintaining chat infrastructure and what it trades away.*
4. **Custom admin panel with content, moderation, and monetization tooling** — full CRUD for users, countries, interests (with per-app custom descriptions for SEO), social apps, blog, static pages, SEO metadata, ad placements, and a "promoted profile" boost feature, plus a moderation queue for user reports grouped by offender. *Client explanation needed: low — a standard "admin dashboard" pitch, though the SEO-metadata-per-entity and per-interest-per-app description system is a nice added detail.*
5. **Like/dislike reputation system with race-safe voting** — aggregate vote counting via a single raw SQL query per profile, backed by a database-level unique constraint that prevents duplicate votes even under concurrent requests. *Client explanation needed: low — "each person can only vote once per profile, enforced at the database level" is a quick, credible technical-competence signal.*
---
 
## 10. Confidentiality / Disclosure Flags
 
**Must NOT be shown publicly / must be redacted before sharing this repo or its history:**
- Nothing found in the current tracked codebase itself — no `.env` file, no hardcoded API keys/secrets in `app/`, `config/`, `database/`, `routes/`, or `resources/` (all secrets are correctly sourced via `env()`).
- **Recommend you double-check before publishing anything derived from git history**, not just the current tree: `git log -p` across 887 commits could contain an accidentally-committed `.env`, credential, or client-identifying string at some point in history even if it's absent from `HEAD`. I did not do a full historical diff scan — only checked the current working tree.
- **Database seeders/factories**: `database/factories` and `database/seeders` exist per Laravel convention — open these directly and confirm no real user data, real emails, or real client names were used as seed/fixture data before showing them to anyone. I did not read their contents in this pass.
**Borderline — could indirectly identify the client even without naming them:**
- The specific **list of connectable social apps** and their exact URL templates (stored in the `social_media` table, not in code) could be identifying if the app-selection list is unusual or niche. Review the actual seeded data, not just the schema.
- The **domain name and branding** implied by `crushfling` itself, and any copy in the `Document`/CMS records (Privacy Policy, About page body text) — these are data, not code, so not covered by this code-only pass, but flag them before you publish screenshots or describe the product by name.
- The **admin email list** (`ADMIN_LIST` env var value) is not in the repo, but if you have local `.env` files or deployment configs outside this repo, don't include them in anything you share.
**I did not reproduce any sensitive values in this document** — everything above points you to the file/location to check yourself.
 
---
 
## 11. Open Questions For You
 
These can't be determined from the code alone and will change how you present this project:
 
1. **Was this ever deployed to production, and is it still live?** No CI/CD config, no `Procfile`/`Dockerfile`/deployment scripts, and no `.env` were found in the repo — deployment method and current live status are unknown from code alone.
2. **What was the client relationship and original brief?** You marked this "unknown" — was this a from-scratch build, a takeover of an existing MVP, or a long-running maintenance retainer? The 2.5-year, 887-commit history suggests either a long single engagement or several distinct phases of work — worth clarifying which, since it changes how you'd frame scope and duration on a profile.
3. **How many real users/tenants does this run for, if any?** No access to a live database or analytics from this pass — can't speak to actual usage/scale, only to what the code is built to handle.
4. **Was there a measurable outcome** (users acquired, engagement metrics, revenue from the ad-slot or "boost" features) that the client shared with you? Nothing in the code answers this.
5. **What is the actual current status of the Tinify → Intervention/Image migration** referenced in §3 ("Tinify removed" is the tip commit, but the Tinify service/facade/dependency are still present)? Worth a quick manual check before describing image compression as a shipped, stable feature.
6. **Do you want the dead `UserMeta`/`InterestSocialMediumController` code cleaned up** before you screenshot or walk through this codebase with anyone, given it references relations/scopes that no longer exist?
 
