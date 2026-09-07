# Fanscritic — Technical Brief
 
Source: static analysis of the `fanscritic` repo (Laravel 10, PHP 8.1) as of 2026-08-28. Extraction only — no marketing language. Labels used throughout: **[VERIFIED]** directly evidenced in code, **[INFERRED]** reasonable reading of the code but not explicitly stated, **[UNCLEAR — needs your input]** cannot be determined from the repo alone.
 
Repo activity per prior extraction (`RESUME_EVIDENCE.md`): commits span 2021-01-28 to 2023-09-06, 790 total commits, 3 authors, you (`Arif Hossen`) authored 714 (90%). No `.env`, no CI/CD config, no remote configured locally.
 
---
 
## 1. What This System Actually Does
 
Fanscritic is a review/directory website for creators on subscription-content platforms (the composer.json description reads "Onlyfans & Premium Accounts Girls w/ Reviews | fanscritic.com"). In plain terms, it works like Yelp or Trustpilot, but for creator profiles on platforms such as OnlyFans:
 
- Visitors browse a directory of creator profiles ("models" in the code), organized by platform and category/tag, and can search for a specific creator.
- Anyone — logged in or not — can leave a star rating and written review for a creator. Logged-out visitors can submit a review and are then prompted to register to have it published (the review is held as "pending" and linked to their session).
- Registered users can like/dislike individual reviews, edit their own profile and password, and (if they are a creator) claim and manage their own listing — uploading a photo, setting their platform, country, and category tags.
- A separate admin back office (under a `/master` URL prefix) lets site staff manage creators, reviews, platforms, categories, countries, blog posts, static pages (About/Terms-style documents), homepage/site settings, ad placements, and view application logs.
- A blog module and a lightweight CMS (via a "Document" model) support editorial/marketing content alongside the review directory.
Who uses it: end users researching or rating creators, creators who want a public review profile, and internal site administrators/moderators running the platform day to day.
 
---
 
## 2. Architecture
 
**Overall shape** [VERIFIED]: A traditional Laravel monolith — server-rendered Blade views, no separate API/SPA split (routes/api.php exists but is just the Sanctum-style stub, unused by the app). Single database, no multi-tenancy — this is a single-site consumer product, not a SaaS platform serving multiple customers.
 
**Structure** [VERIFIED]:
- MVC via Laravel's conventions: `app/Http/Controllers` (public + a dedicated `Admin/` namespace), `app/Models`, Blade views under `resources/views`.
- Three route files loaded with distinct middleware stacks in [app/Providers/RouteServiceProvider.php](app/Providers/RouteServiceProvider.php:47): `routes/api.php` (api middleware, effectively unused), `routes/admin.php` (wrapped in `['web', 'admin']` middleware, named/prefixed `admin.` / `master`), `routes/web.php` (public site).
- A small `app/Service` layer holds integration wrappers (`TinifyService`, `OnlyfansScraper`, `Settings`) rather than a full service-layer architecture — most business logic still lives directly in controllers and Eloquent models (fat-model style), not a repository or CQRS pattern.
- Laravel Fortify handles authentication (login, registration, password reset, email verification), with the actual user-creation/update logic overridden in `app/Actions/Fortify/*` — this is the standard Fortify "Actions" pattern, not custom-built auth.
- Auth wiring is done in [app/Providers/FortifyServiceProvider.php](app/Providers/FortifyServiceProvider.php) — no Jetstream, no Sanctum tokens in active use.
**Key patterns**:
- **Route-model binding with scoping**: [app/Providers/RouteServiceProvider.php:41-45](app/Providers/RouteServiceProvider.php:41) binds the `{model}` route parameter to the `User` model filtered to `type = 1` (creator accounts) via a `model()` query scope on `User` — a deliberate (if confusingly named, since `App\Models\Model` is a separate entity) narrowing of implicit binding.
- **Query scopes over repositories**: heavy use of Eloquent local scopes (`active()`, `topRatedModelFirst()`, `withReviewCount()`, `totalReactions()`, `selectedField()`) to compose reusable, chainable query logic instead of a repository layer — e.g. [app/Models/Model.php:165-209](app/Models/Model.php:165).
- **Trait-based reuse**: a `Likeable` trait ([app/Models/Helper/Likeable.php](app/Models/Helper/Likeable.php)) implements the like/dislike-via-`updateOrCreate` pattern once and is mixed into `Review`.
- **Shared upload pipeline**: a single `Controller::uploadImage()` helper ([app/Http/Controllers/Controller.php](app/Http/Controllers/Controller.php)) runs every uploaded image (avatars, thumbnails) through the Tinify compression/resize API before writing to storage — centralizes image handling instead of duplicating it per controller.
- **Custom Facade + Service wrapper** around the Tinify SDK ([app/Facade/Tinify.php](app/Facade/Tinify.php), [app/Providers/TinifyServiceProvider.php](app/Providers/TinifyServiceProvider.php), [app/Service/TinifyService.php](app/Service/TinifyService.php)) — standard Laravel service-container/facade pattern applied to a third-party SDK that doesn't ship one.
- **No queues in practical use**: `QUEUE_CONNECTION=sync` in `.env.example`; no `app/Jobs` directory exists. Everything (including image compression via an external API call) runs synchronously in the request cycle.
- **No events/listeners architecture**: `EventServiceProvider` is present but unused beyond Laravel's default registration boilerplate — no custom domain events found.
**Database design notes** [VERIFIED, from `database/migrations`]:
- **Many-to-many tagging**: `category_model` pivot table links `Category` ↔ `Model` (creator profiles can carry multiple category tags), managed via `attach()`/`sync()` in [ModelController.php:55,97](app/Http/Controllers/ModelController.php:55).
- **Split identity model**: `User` (login/account) and `Model` (public creator profile) are separate tables linked by a **nullable, unique** `user_id` foreign key on `models` ([2021_02_15_135205_create_models_table.php:18-22](database/migrations/2021_02_15_135205_create_models_table.php:18)) with `onDelete('set null')`. This enforces at the DB level that a creator profile can be "claimed" by at most one account, while still allowing unclaimed/ghost profiles to exist independently. Admin can attach/detach the link (`Admin\UserController::detachModel`).
- **Soft deletes on `reviews` only** ([2021_02_15_144133_create_reviews_table.php:37](database/migrations/2021_02_15_144133_create_reviews_table.php:37)), combined with an app-level `status` column (0 pending / 1 approved / 2 "deleted", where the `status` accessor synthesizes 2 whenever `deleted_at` is set — [Review.php:64-71](app/Models/Review.php:64)). This is a hybrid soft-delete/moderation-state design, not a plain soft-delete.
- **Attempted-then-abandoned uniqueness constraint**: a `unique(['user_id', 'model_id'])` on `reviews` is present in the migration but commented out ([2021_02_15_144133_create_reviews_table.php:35](database/migrations/2021_02_15_144133_create_reviews_table.php:35)); duplicate-review prevention is instead enforced in application code (`HomeController::isUserProvideReview()`). Likely reason: the same column pair needs to allow multiple *guest* (`user_id IS NULL`) reviews per model, which a DB unique constraint can't easily express — [UNCLEAR — confirm this was the actual reasoning].
- **Real uniqueness constraint on `reactions`**: `unique(['review_id', 'user_id'])` ([2021_10_07_111033_create_reactions_table.php:27](database/migrations/2021_10_07_111033_create_reactions_table.php:27)) correctly prevents duplicate like/dislike rows, paired with `updateOrCreate` in the `Likeable` trait — this one is fully solid.
- **No polymorphic relations** anywhere in the schema — despite the review/reaction/like structure resembling a common polymorphic-comments use case, `Reaction` is tied directly to `Review` (not a generic polymorphic "commentable"), and `Avatar` uses a manual composite `user_id`/`model_id` pattern rather than `morphTo`.
- Timestamps use timezone-aware columns (`timestampsTz()`, `softDeletesTz()`) on the newer tables (reviews, models) but not on the original Laravel-default tables (users) — inconsistent but minor.
---
 
## 3. Notable Technical Decisions
 
- **Custom signed-URL verification for email verification** ([app/Http/Controllers/EmailVerificationController.php](app/Http/Controllers/EmailVerificationController.php)) reimplements Laravel's `hasValidSignature()` logic by hand (HMAC-SHA256 over the URL + query string, `hash_equals` comparison, separate expiry check) instead of using the framework's built-in signed-route middleware. The project also depends on `fsasvari/laravel-trailing-slash` (composer.json), a package that rewrites URLs to enforce/strip trailing slashes — that package is a very plausible reason the default signature check (which is sensitive to exact URL string) needed to be reimplemented to tolerate the trailing-slash rewrite. [INFERRED — the causal link to the trailing-slash package should be confirmed from memory, the repo doesn't state it]. Getting a signed-URL reimplementation right (constant-time comparison, correct expiry handling) is the kind of detail an inexperienced developer typically gets wrong.
- **Guest review capture with deferred publish**: unauthenticated visitors can submit a review immediately; it's stored with `status = 0` (pending) and `user_id = null`, and the review id + return URL are stashed in the session ([HomeController::storeReview](app/Http/Controllers/HomeController.php:73-132)) so the user can be routed back after registering. This is a deliberate conversion-funnel design (capture intent before asking for signup), not an accident.
- **Server-generated shareable review codes**: each review gets a random 6-character code (`Review::generateRandomCode()`, [Review.php:208](app/Models/Review.php:208)) used to build shareable/highlightable permalinks — `PlatformModelController::show()` uses this code to pull a specific review to the front of the paginated list ([PlatformModelController.php:27-40](app/Http/Controllers/PlatformModelController.php:27)).
- **Unclaimed/ghost creator profiles**: `PlatformModelController::show()` will render a review page for a `platform + username` combination that doesn't exist yet in the `models` table, constructing an in-memory (unsaved) `Model` instance on the fly ([PlatformModelController.php:18-25](app/Http/Controllers/PlatformModelController.php:18)) so reviews/SEO pages can exist before a creator ever claims their profile.
- **Rating formula with a deliberate rounding bias**: `Model::ratingCalculation()` computes `round(sum/count + 0.5, 1)` ([Model.php:211-218](app/Models/Model.php:211)) — adding 0.5 before rounding is a specific, non-default choice (rounds up rather than to-nearest); worth confirming this was an intentional display decision vs. a rounding bug, since it will always bias the displayed rating upward. [UNCLEAR — needs your input on intent].
- **Route-parameter naming collision handled deliberately**: because `App\Models\Model` collides with PHP's/Eloquent's own base `Model` class and with the `{model}` route-binding convention, the app aliases the base class (`use Illuminate\Database\Eloquent\Model as BaseModel`) and uses `app_model` as the route parameter alias in `Route::resource(...)->parameters(['models' => 'app_model'])` ([routes/web.php:47](routes/web.php:47), [routes/admin.php:16-19](routes/admin.php:16)) — a small but correct piece of naming-conflict management.
---
 
## 4. Integrations
 
| Integration | Purpose | Error handling |
|---|---|---|
| **Tinify API** (`tinify/tinify` SDK, wrapped in `TinifyService`/`Tinify` facade) | Compresses and resizes every uploaded avatar/thumbnail image before storage | **None at the call site.** `Controller::uploadImage()` calls the Tinify API synchronously with no try/catch — an API outage, quota exhaustion, or invalid key throws an uncaught exception straight through the controller. `TinifyService::__construct()` does fail fast if the API key is missing ([TinifyService.php:19-23](app/Service/TinifyService.php:19)), but that's a boot-time guard, not runtime resilience. |
| **AWS S3** (`league/flysystem-aws-s3-v3`) | Optional storage disk for uploaded media (configured but `FILESYSTEM_DISK=public` by default per `.env.example`) | Disk config sets `'throw' => true` ([config/filesystems.php:43,55](config/filesystems.php:43)), so Flysystem exceptions propagate rather than failing silently — reasonable default, but again no application-level retry/fallback. |
| **Mailgun** (via `symfony/mailgun-mailer`) | Transactional email (verification, password reset) through Laravel's mailer | Relies entirely on Laravel/Symfony Mailer's default behavior; no custom retry or logging wrapper found. |
| **Guzzle HTTP client + raw cURL** (`app/Service/OnlyfansScraper.php`) | Scrapes a third-party site (hardcoded default URL, see §10) to pull a creator's profile image by username, using cookie-jar-based curl requests and a regex against the HTML response | **Minimal.** `setImgPath()` does a `preg_match_all` against the scraped HTML and directly destructures the first match with `list($a, $path) = explode(...)` ([OnlyfansScraper.php:59-70](app/Service/OnlyfansScraper.php:59-70)) — if the target site's markup changes or the profile isn't found, this throws a PHP warning/error rather than failing gracefully. No retries, no timeout configuration visible, no HTTP status checking. [Note: per `RESUME_EVIDENCE.md`, the controller wiring this service into a route was removed from the codebase in 2021 — the service class itself is dead code today unless something else still calls it; confirm before citing it as a live integration.] |
| **`jackiedo/log-reader`** | Admin-facing UI to browse and delete Laravel log entries ([LogReaderController.php](app/Http/Controllers/LogReaderController.php)) | Not really an "integration" with error handling of its own — it's a package wrapped directly into admin routes. |
| **Pusher / Laravel Echo** | Config and `.env` keys present (`PUSHER_APP_*`), `BroadcastServiceProvider` registered, one broadcast channel defined ([routes/channels.php](routes/channels.php)) | **Not actually wired to any feature** — no `broadcast()` calls or event broadcasting found anywhere in `app/`. Boilerplate left in from Laravel's default scaffolding, not a live integration. |
 
---
 
## 5. Performance & Scale Considerations
 
- **Queue**: `QUEUE_CONNECTION=sync` and no `app/Jobs` — the Tinify API call (an external HTTP round-trip) runs inline on every image upload request. This works fine at low traffic but means upload latency is directly coupled to a third-party API's response time, and there's no backpressure/retry if Tinify is slow or down.
- **Caching**: Real but narrow. `App\Service\Settings::get()` uses `Cache::rememberForever()` for site settings, with explicit `cache()->forget($key)` invalidation on write in `Admin\SettingController::storeUpdate()`/`destroy()` ([SettingController.php:60,83](app/Http/Controllers/Admin/SettingController.php:60)) — this is a correctly-implemented cache-aside pattern for a low-write, high-read config value. `CACHE_DRIVER=file` by default (not Redis, despite Redis config being present) — fine for a single-server deployment, would need to move to Redis/Memcached for any multi-server setup.
- **N+1 avoidance — mostly handled well**: `Model` sets `protected $with = ['platform']` ([Model.php:49](app/Models/Model.php:49)) so the platform relation is always eager-loaded; listing queries consistently use `.with(...)` for `model`, `user`, `tags`, `platform` (e.g. [ReviewController.php:33-37](app/Http/Controllers/ReviewController.php:33), [UserController.php:44](app/Http/Controllers/UserController.php:44)). Review/rating aggregates are computed via correlated subqueries in `withReviewCount()`/`topRatedModelFirst()` ([Model.php:165-209](app/Models/Model.php:165)) rather than loading all reviews into PHP and aggregating in memory — this is the right approach for a directory listing page and shows real query-performance awareness, not prototype-level code.
- **One place this breaks down**: `Review::reactionsCount()` ([Review.php:133-151](app/Models/Review.php:133)) and the `Reaction::selectRaw(...)->first()` calls issue their own separate query when called on a single already-loaded `Review` (used from `ReactionController`) rather than reusing the `totalReactions()` scope's single aggregated query — minor, not a listing-page N+1 since it's only hit on the like/dislike AJAX endpoints, but it is duplicated logic between `reactionsCount()` and `scopeTotalReactions()`.
- **Search is `LIKE '%term%'`** everywhere (`HomeController::search/getModels/getUsers/getCountries`, `Admin\UserController::index`) — correctly parameterized (Eloquent binds the value, so no SQL injection — see §7), but a leading-wildcard `LIKE` can't use a standard B-tree index and will full-scan as the table grows. Fine at current scale; would not scale to a large catalog without a real search index (e.g. a fulltext index or external search service).
- **Pagination is used consistently** on every list-type endpoint (20–40 per page) rather than loading unbounded result sets — a basic but important scale-awareness signal.
- Overall assessment: this reads as a real production directory site built with reasonable query discipline for its scale (thousands–tens of thousands of rows), not a prototype — but it was not built with heavy horizontal-scale (multi-server cache/queue, search infrastructure) in mind.
---
 
## 6. Testing
 
**Coverage is effectively zero.** [VERIFIED]
- `tests/Feature/ExampleTest.php` and `tests/Unit/ExampleTest.php` are the untouched Laravel default stubs — one asserts the homepage returns HTTP 200, the other asserts `true === true`.
- No feature tests exist for registration, review submission, the like/dislike flow, admin CRUD, image upload/compression, or the claim/detach-model flow — all of which contain real business logic and, per §7/§8, at least one access-control gap that a test suite would plausibly have caught.
- No `phpunit.xml` customization beyond Laravel defaults (in-memory/testing DB config is standard).
- Be direct about this if it comes up with a client or in a portfolio context: there's no evidence of automated test coverage on this project. If you want to cite testing discipline as a strength elsewhere in your portfolio, this repo isn't the example to use for that claim.
---
 
## 7. Security Posture
 
**Authentication**: Laravel Fortify, standard and appropriate — bcrypt hashing via `Hash::make`, login rate-limited to 5/minute per email+IP and 2FA rate-limited to 5/minute per session ([FortifyServiceProvider.php:41-49](app/Providers/FortifyServiceProvider.php:41)), email verification enforced via `MustVerifyEmail` on the `User` model plus signed-URL verification (custom-implemented, see §3).
 
**Authorization**: Coarse-grained and inconsistent — no policy classes are registered (`AuthServiceProvider::$policies` is empty with a commented-out `ModelPolicy` placeholder, [AuthServiceProvider.php:15-17](app/Providers/AuthServiceProvider.php:15)), no use of Laravel's `Gate`/`can` middleware anywhere. Authorization is done ad hoc per-controller:
- `UserController::edit()` correctly checks `auth()->id() !== $user->id` and aborts 403 ([UserController.php:19-21](app/Http/Controllers/UserController.php:19)) — ownership is enforced here.
- **`ModelController::edit()` / `update()` do not check ownership at all.** The constructor only applies the `auth` middleware ([ModelController.php:16-19](app/Http/Controllers/ModelController.php:16)) — any logged-in user can `GET /models/{app_model}/edit` or `PUT /models/{app_model}` for **any** creator profile, not just one they own, and change its name, avatar, country, platform, and tags. A code comment even flags the intent ("auth middleware is inside modelcontroller in construct", [routes/web.php:46](routes/web.php:46)) without the corresponding ownership check ever being added. **This is a real broken-access-control (IDOR) issue** — flagging for you to fix or confirm mitigated in a later, un-reviewed part of the codebase before this is cited anywhere public-facing. Admin routes are unaffected (properly gated by `AdminMiddleware`).
- Admin area: gated by a single `AdminMiddleware` checking `Auth::user()->is_admin` (i.e. `type === 2`) ([AdminMiddleware.php](app/Http/Middleware/AdminMiddleware.php)) applied to the whole `routes/admin.php` group via `Route::middleware(['web','admin'])` ([RouteServiceProvider.php:53](app/Providers/RouteServiceProvider.php:53)). This is a coarse all-or-nothing admin flag, not role/permission-based — fine for a small internal admin team, would not scale to differentiated staff roles (e.g. "moderator can approve reviews but not edit settings") without rework.
- Admin URL is namespaced under `/master` rather than `/admin` — mild obscurity, not a real control (the middleware is what actually protects it).
**Input validation**: Consistently done via Laravel's `$request->validate()` / Form-level `Rule` objects across controllers — sizes, types, `exists:`/`unique:` checks are present on essentially every write endpoint reviewed. Custom rule classes (`UserName`, `MatchOldPassword`) are used appropriately.
 
**SQL injection**: Not found. All the `LIKE "%{$var}%"` patterns (e.g. [HomeController.php:138](app/Http/Controllers/HomeController.php:138)) pass the interpolated string as a **bound parameter value** to Eloquent's query builder, not as raw SQL — Eloquent parameterizes it correctly. This looks alarming on a first read and is worth a deliberate second look, but it is not exploitable as written.
 
**Mass assignment**: `$fillable` arrays are used consistently (no `$guarded = []`), and sensitive fields like `type`/`is_admin` are not user-settable through any validated request body reviewed.
 
**Multi-tenant data isolation**: Not applicable — this is a single-tenant application, not a multi-tenant SaaS product. There is no tenant-scoping concern to evaluate here; note this explicitly if this repo comes up in the context of your multi-tenancy portfolio niche, since it doesn't demonstrate that pattern.
 
**Dead-code / leftover risk**: `app/Http/Controllers/DatabaseQuery.php` is a one-off "backfill review codes" script left in the controllers directory ([DatabaseQuery.php](app/Http/Controllers/DatabaseQuery.php)) with a raw `DB::table()` update loop. It is **not registered in any route file** (confirmed via search), so it isn't currently reachable — but it's the kind of leftover admin/maintenance script that becomes a real risk if someone later adds a route to it without adding auth. Worth cleaning up, not worth alarming about as-is.
 
**Secrets**: No `.env` file is committed; `.gitignore` correctly excludes it. No hardcoded API keys, passwords, or tokens found in `app/` or `config/` — all secrets are sourced via `env()`. Good hygiene.
 
---
 
## 8. Complexity & Difficulty Assessment
 
What was genuinely non-trivial to get right here:
- **The claimed/unclaimed creator profile model** — designing the `User`⇄`Model` relationship as two separate tables joined by a nullable-unique FK, so that (a) reviews and SEO-friendly profile pages can exist for creators who haven't signed up yet, (b) a creator can later claim that profile and it becomes tied to their account, and (c) an admin can detach that link without destroying the review history. A less experienced developer would likely have collapsed this into a single `users` table with a `role` flag and hit a wall the moment "review a creator before they exist in the system" became a requirement — or would have allowed a profile to be claimed by multiple accounts by skipping the DB-level unique constraint.
- **The guest-review-then-register conversion flow** together with the review-code permalink/highlight system — coordinating session state, a generated review before an account exists, and re-surfacing that specific review after redirect, without a dedicated state machine or job queue to lean on. This is fiddly control flow to get right by hand (see `HomeController::storeReview` and `PlatformModelController::show`).
- **Correlated-subquery rating aggregation** (`withReviewCount()`, `topRatedModelFirst()`) instead of loading reviews into PHP to compute averages — an easy place for a less experienced developer to introduce an N+1 or an in-memory aggregation that doesn't scale past a few hundred rows.
- **The signed-URL reimplementation for email verification** — subtle to get right (constant-time comparison, correct handling of the signature/expires query params) and easy to get subtly wrong (e.g. non-constant-time comparison would be a timing side-channel; this code correctly uses `hash_equals`).
What a less experienced developer likely would have gotten wrong (and where this codebase either got it right or didn't):
- Got right: parameterized `LIKE` search, `hash_equals` for signature comparison, cache invalidation on the settings write path, DB-level unique constraint on reactions.
- Got wrong (see §7): missing ownership check on `ModelController` — a classic, common access-control gap even experienced teams miss without either tests or a policy-class convention forcing the question to be asked per-resource.
---
 
## 9. Portfolio-Worthy Features
 
1. **Claim/unclaimed creator-profile system** (`User` ⇄ `Model` split with a nullable-unique FK, admin attach/detach, ghost-profile rendering for not-yet-created profiles). *Client explanation needed: moderate.* You'd need to explain the "review before signup" business requirement first, then the data-model solution — but it's a clean, concrete story ("built a system where anyone can review a creator, and the creator can later step in and take ownership of that page").
2. **Guest review capture → deferred publish → post-registration redirect flow**, including the shareable/highlightable review-code permalink system. *Client explanation needed: low.* Easy to describe as a growth/conversion mechanic — "let people act first, ask them to sign up second."
3. **Centralized image-compression upload pipeline** (every avatar/thumbnail upload routed through a Tinify-backed Facade/Service before storage). *Client explanation needed: low.* "Every photo uploaded to the site gets automatically compressed so pages load fast" is self-explanatory to a non-technical audience.
4. **Query-level rating/ranking engine** (`topRatedModelFirst`, `withReviewCount` correlated-subquery scopes powering the homepage "top rated" sort). *Client explanation needed: low-moderate.* "The homepage ranks creators by rating and review volume computed directly in the database, not in application code" — good if paired with the performance framing from §5.
5. **Admin back office with a configurable settings system** (`Setting` model + cache-aside `Settings::get()` + rule-driven dynamic form config in `config/data.php['settings']`) letting non-technical staff edit site copy (homepage title/description) without a deploy. *Client explanation needed: low.* "Marketing/ops team can change homepage text themselves, no developer needed."
6. **Category/platform/tag directory browsing with SEO-friendly URL structure** (slug-based routing, 301 redirects for legacy URL patterns — see [routes/web.php:62-72](routes/web.php:62)). *Client explanation needed: low.* Straightforward "structured, crawlable directory site" story.
---
 
## 10. Confidentiality / Disclosure Flags
 
**Must not be shown publicly:**
- No committed `.env`, no hardcoded credentials found in `app/`/`config/` — nothing to redact on that front from the current code.
- **The hardcoded scrape target in `app/Service/OnlyfansScraper.php:18`** (`https://fansmetrics.com` as the default constructor argument) reveals the specific third-party site the scraper was built against and the scraping technique used (cookie-jar cURL + regex against their markup). Do not publish this file's contents or the target domain in a public portfolio without thinking through whether naming that source could raise ToS/legal questions for you or the client — this is business-sensitive, not just technically sensitive. Per `RESUME_EVIDENCE.md`, the controller that routed to this service was removed from the app in 2021, so confirm current relevance before deciding how to present it at all.
- Composer package name `digipona/fanscritic` ([composer.json:2](composer.json:2)) and the `fanscritic.com` domain reference in the same description string identify the client/product by name. If your engagement terms require anonymizing the client, this needs to be scrubbed from any code excerpt you publish.
- No real customer data found in seeders/factories reviewed (`database/seeders/*`, `database/factories/*` use Faker-style generation per Laravel convention) — but you should skim the actual seeder file contents yourself before publishing, since I didn't fully verify field-by-field.
**Borderline (could identify the client/niche even without naming them):**
- The specific business rules around "OnlyFans"-prefixed usernames/categories (`UserName` rule blocking usernames starting with "onlyfans", `onlyfans-` category-name convention, `GENDER_BOY_NAME`/`GENDER_GIRL_NAME` env vars) are fairly distinctive to this niche. Screenshots or code excerpts showing these strings verbatim would likely identify the vertical (adult-creator review site) even if you don't name the client — decide deliberately whether that's acceptable for your portfolio positioning.
- The admin route prefix `/master` and the internal terminology ("models" for creator profiles, "reactions" for votes) are minor but somewhat distinctive naming choices — low risk, flagging for completeness.
---
 
## 11. Open Questions For You
 
1. Was this ever deployed to production, and is it still live at fanscritic.com or elsewhere? No deployment config, CI/CD, or infra-as-code exists in this repo to confirm either way.
2. Any measurable outcome (traffic, registered users, creator signups, revenue) you can attach to this work? Nothing in the repo speaks to business impact.
3. Is the `ModelController` ownership gap (§7) still present in whatever version is live today, or was it fixed in work that isn't in this local repo? Worth confirming before you reference this project's security posture in any client-facing way.
4. Confirm the actual client/business context — the task brief for this brief left "client/business type" and "what the client asked for" as unknown; I did not attempt to infer either beyond what composer.json states.
5. Is `app/Service/OnlyfansScraper.php` still in active use anywhere in a version of the app not reflected here? It reads as dead code in this repo (see §4, §10).
6. Do you want the `/master` admin path, the `fansmetrics.com` scrape target, or the `digipona/fanscritic` package name treated as confidential in downstream materials? I've flagged them but left the call to you.
 
