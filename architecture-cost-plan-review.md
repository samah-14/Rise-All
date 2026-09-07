# Review - Rise All Architecture, Technology & Operational Cost Plan

## Condense

The plan's *instincts* are good: keep Figma fidelity, keep cost low, use managed services where security matters (Supabase), defer nice-to-haves (Sentry) until justified. The direction is sound.

But **the entire architecture hinges on one unvalidated technical assumption that hasn't been proven**, and the cost table contains **two verifiable arithmetic errors**. Neither is cosmetic - one determines whether the app runs at all, the other determines whether the headline cost numbers are trustworthy. **Do not approve this plan for build until §1 and §2 below are resolved.**

---

## 1. Technical Risk: Next.js on Hostinger Business (Shared Hosting)

This is the single most important thing in the deck, and it is asserted, not demonstrated.

**The claim (Slides 3, 4, 9):** Both WordPress *and* the custom Next.js app run on "ONE HOSTINGER BUSINESS PLAN," described as "consolidated hosting... one bill, one account, lower cost" at ~$17/mo.

**Why this needs scrutiny:** Hostinger's Business plan is shared PHP/LiteSpeed hosting built for WordPress-style apps (PHP + MySQL). Next.js is not a static site generator here - per Slide 6, it needs auth, dashboards, booking logic, file uploads, and CRUD, which means **server-side rendering, API routes, and middleware** - a persistent (or serverless) Node.js process, not just static HTML output.

Some Hostinger plans do offer a Node.js app manager (Passenger-based), which *can* run a basic Next.js app under a subpath. But at "Business" tier this typically comes with real constraints that aren't addressed anywhere in the deck:
- Shared CPU/memory ceilings that Next.js SSR can exceed under concurrent load
- No confirmed WebSocket support (relevant if Supabase real-time features are ever used)
- Uncertain behavior for long-running or cold-started Node processes on shared infrastructure
- No autoscaling - directly conflicts with the original requirements' **NFR5 (Scalability)**: *"support future growth... without significant changes to the system architecture"* (see `requirements-feedback.md` §7). Shared hosting is the opposite of a scalable foundation.

**This has not been proven to work** - there's no mention of a proof-of-concept, a test deployment, or even a citation that this configuration is supported. The entire cost plan (and the reverse-proxy design in §2 below) is built on top of this assumption.

**Required before sign-off:** Stand up a minimal Next.js app (with one API route and one Supabase-authenticated page) on the actual Hostinger Business plan being purchased, and confirm it serves reliably under the intended path (`/app/*`) before committing further design or dev hours to this architecture.

**Lower-risk alternative worth putting on the table:** Host the Next.js app on **Vercel** (built by the Next.js team, generous free tier, designed exactly for this workload) and keep WordPress on cheap shared hosting (Hostinger or otherwise). Vercel's `rewrites` config can proxy the marketing routes (`/`, `/services`, `/about`, `/contact`) to the WordPress origin while serving `/app/*` natively - achieving the exact "no visible seam, one domain" goal in Slide 7 **without** needing Node to run inside shared hosting at all. This inverts the proxy direction (Vercel in front, proxying *to* WordPress, rather than Hostinger proxying *to* Node) and removes the single biggest technical risk in the deck. Worth a cost/feasibility comparison against the current plan before committing.

---

## 2. Technical Risk: Reverse Proxy Feasibility on Shared Hosting

Slide 7 correctly rejects iframes and subdomains and lands on a reverse proxy as the right call *in principle* - that reasoning is good. But **implementing** `rise-all.com/app/* → Next.js` as a reverse proxy typically requires nginx/Apache vhost-level control (or a platform with native path-based rewrites). Shared cPanel/hPanel hosting usually restricts you to `.htaccess`, where `mod_proxy` is frequently disabled by the host for security/resource reasons, and WebSocket upgrade headers are poorly supported through `.htaccess` proxying even when available.

This directly interacts with §1: if Next.js can't run natively inside Hostinger's shared environment, the reverse-proxy plan as drawn (proxy *within* Hostinger) doesn't have a target to proxy to. Confirm with Hostinger support (in writing) that: (a) a persistent Node app can run under the Business plan, and (b) path-based proxying to it from the WordPress domain is supported - before this slide's design is treated as final.

---

## 3. Cost Table Errors (Slide 9) - Verified by Hand

Recomputing the stated totals from their own line items:

| Column | Stated Total | Recomputed (excl. flagged optional/excluded item both ends) | Recomputed (incl. it both ends) | Problem |
|---|---|---|---|---|
| **Fully Managed** | $76–205 | $76–179 | $102–205 | Low end ($76) **excludes** the $26 Sentry line; high end ($205) **includes** it. Sentry is presented as a flat, non-optional $26/mo in that row - it should be in both bounds or neither. |
| **Self-Hosted** | $22–64 | $22–44 | $22–64 (only if email is added at its $20 high end) | Email is explicitly labeled "**Not recommended** (deliverability risk)" in that column - implying $0/excluded - yet the high-end total ($64) only reconciles if the $20 email cost *is* added in. Low end ($22) excludes it. |
| **Recommended** | $43–114 | $43–114 | - | ✅ Internally consistent (Sentry excluded at both ends, matching its "optional, add post-launch, $0/mo" label). |

**Only the Recommended column's math is actually correct.** The other two columns exist to make the Recommended option look like a reasonable middle ground - but if their totals are wrong, that comparison isn't trustworthy. Before this table goes in front of anyone approving a budget, fix the Fully Managed and Self-Hosted totals to consistently include or exclude Sentry and email respectively at both ends of their ranges.

**Separately:** Slide 10 ("Next Steps," item 2) calls out *"Get licenses for coding tools (e.g. Claude Code, dev tooling)"* as an action item - but no tooling/license line exists anywhere in the Slide 9 cost table, which is scoped as "**recurring infrastructure only** - developers are volunteer-based." If any paid dev tooling is actually needed, it's a real recurring cost that the "infrastructure only" framing is hiding. Either confirm tooling is $0 (free tiers only) or add a line for it.

---

## 4. Silent Scope Regressions vs. the Requirements Document

Two things in this deck quietly reduce scope from what `requirements-feedback.md` flagged as already-approved requirements, without calling it out as a decision needing sign-off:

- **Calendar sync dropped.** Slide 6 states *"Booking Engine... no calendar sync in MVP."* The original requirements doc's **FR28** explicitly required syncing sessions to mentors' Google Calendar and Microsoft Outlook. Deferring this may well be the right MVP call - but it should be presented as an explicit trade-off the team is approving, not a parenthetical on a slide, since it changes what mentors experience on day one.
- **No session-delivery mechanism.** This gap was flagged in the requirements review (P2) as the single biggest missing feature - how a booked session is actually *held* (video link, Zoom/Meet integration). The architecture deck doesn't resolve it either: Slide 6 covers booking *up to* the point of reserving a slot, and then goes silent. A user can complete every step in this architecture and still have no way to meet their mentor.
- **No payments/monetization infrastructure** - consistent with the requirements gap (P1: is this ADPList-free or TopMate-paid?). This architecture is fully buildable either way, but if payments are ever in scope, nothing here (Stripe, payouts, webhooks) accounts for it. Worth an explicit one-line assumption on the deck: "v1 assumes no paid sessions."
- **Role model unresolved.** Slide 3/4 describe Supabase Auth roles as "mentee/mentor/volunteer/admin" - phrased as if a user has *one* role. `requirements-feedback.md` §4 flagged that a real user can hold multiple roles simultaneously (a volunteer who becomes a mentee, per FR40). Confirm whether this is a single-role enum column or a many-to-many roles table in Postgres - this is a schema decision that's expensive to change later, so it should be settled now, not discovered during a plan.

---

## 5. Other Gaps Worth Flagging

- **PII/file-upload security is unspecified.** The requirements doc has mentees/mentors uploading resumes, references, and supporting documents (FR9), and admins running background checks (FR14). Slide 4 says Supabase "holds uploaded files" but nothing here addresses bucket access policy, file-size/type limits, virus scanning, or retention - all real requirements for a system storing this category of PII.
- **Sentry deferred to "post-launch" - reconsider given the above.** Sentry's free tier is $0/mo (5K events/month), so deferring it isn't actually saving money in the Recommended column (it's already priced at $0 either way) - it's just delaying visibility into bugs on a system handling sensitive documents. Recommend enabling the free tier from day one rather than waiting for "real traffic," since the risk it mitigates (silent auth/data bugs) is highest right after launch, not after.
- **No backup/disaster-recovery plan.** Supabase Cloud Pro includes backups, but the deck doesn't state retention or restore process; WordPress-side backup policy on Hostinger isn't mentioned at all.
- **No staging environment in the "Next Steps" plan (Slide 10).** Given §1 and §2 above are unproven, the plan goes straight from setup (steps 4–5) to building both systems in parallel (steps 6–7) to wiring the proxy (step 8) - with no step to validate the reverse-proxy + shared-auth-cookie behavior in a non-production environment first. Add a staging/validation step before both systems are built out fully, so the §1/§2 risk is caught early and cheaply rather than after both codebases exist.
- **No ownership/maintenance plan.** With "developers are volunteer-based" as the model, there's no stated plan for WordPress security patching, Next.js dependency upgrades, or on-call response if something breaks - worth at least a line given the org is called out (in the requirements doc) as needing an Operations Team.
- **Design-fidelity time estimates (Slide 8) have no stated methodology.** The day-estimates (~2.5–3.5 vs ~5.5–7.5 vs ~15–18 days) look plausible but are presented without basis (historical data? gut estimate?). Since "developers are volunteer-based" per Slide 9, it's also unclear whether "extra days" translates to a real cost or just elapsed calendar time - the framing in Slide 8's callout ("costs ~3–4 extra design-days") reads like a cost claim but there's no dollar figure attached anywhere for dev time. Clarify what unit this trade-off is actually being measured in.
- **Headless option's ops-editing story is thin.** Slide 8 lists the headless column's ops-editing experience as "Text fields only" but never names what system provides those text fields (a headless CMS like Sanity/Contentful?) - without one, there's no ops-editing experience at all for a "renders everything in Next.js" approach. Minor since headless isn't the recommendation, but the comparison table should be accurate for all three options, not just the winner.

---

## 6. What's Genuinely Good Here (keep these)

- **Single Figma-token source of truth** for both WordPress and Next.js (Slides 2, 5, 6) is the right call - it avoids the classic problem of a theme's design system drifting from the real app.
- **Rejecting iframe and subdomain approaches** (Slide 7) shows real reasoning, not just a default choice - the trade-offs listed for each are accurate.
- **Keeping Supabase managed while consolidating hosting elsewhere** (Slide 9) is a sensible risk-vs-cost split - don't cut corners on the layer holding auth and PII, do cut corners on hosting.
- **Deferring Sentry and calendar sync** are reasonable MVP instincts in principle - they just need to be explicit, sign-off decisions rather than embedded assumptions (see §4).

---

## 7. Open Questions Before This Plan Is Approved

1. **Has Next.js actually been test-deployed on the specific Hostinger Business plan being purchased?** If not, this must happen before any further commitment (§1).
2. **Has Hostinger confirmed (in writing) that path-based reverse proxying to a Node process is supported on this plan?** (§2)
3. **Which cost-table numbers are correct** - should Fully Managed and Self-Hosted include or exclude their flagged optional/excluded line items? (§3)
4. **Is calendar sync's removal from MVP an approved trade-off**, or an oversight carried over from drafting? (§4)
5. **How will a booked mentoring session actually happen** - is this out of scope for this architecture phase, or an oversight? (§4)
6. **Is the Supabase role model single-role-per-user or many-to-many?** This is a schema decision worth locking in now. (§4)
7. **What is the file-upload security/retention policy** for resumes and background-check materials? (§5)
8. **Is there a paid dev-tooling cost** (e.g., Claude Code licenses) that belongs in the recurring cost table? (§3)

---

## 8. Cross-Cutting Fixes for Scalability & Resilience (Apply Regardless of Which Option Is Chosen)

These four gaps sit outside the original deck entirely - they weren't caught by a feasibility/cost read, only by explicitly checking against latency/throughput and failure-mode criteria. Each is cheap relative to the risk it removes:

- **No CDN/edge caching anywhere in the plan.** Putting Cloudflare (free tier) in front of the WordPress marketing pages caches static HTML at the edge - this simultaneously cuts latency (served near the visitor, not from Hostinger's origin), raises effective throughput (origin never sees repeat requests), and absorbs traffic spikes and basic DDoS attempts. It is the single highest-leverage, lowest-cost fix available and should be in the architecture regardless of which hosting option is chosen.
- **No concurrency correctness for capacity-limited resources.** FR8 (auto-close events at seat capacity) and slot booking both have a textbook race condition: two users can both succeed in claiming the last seat/slot if the check-then-write isn't atomic. This needs a database-level guard - a unique constraint, row lock, or transactional decrement in Postgres - not an application-level "check remaining seats, then insert" pattern. This is a correctness bug under concurrent load, not a hypothetical: it will surface the first time a popular event fills up in real time.
- **WordPress and Next.js currently share a single failure domain.** Whatever hosting option is chosen, keep the two systems isolated enough that a WordPress plugin fault, resource spike, or Next.js crash can't take the other one down with it. Shared-fate infrastructure is the opposite of resilient, independent of which specific host is picked.
- **No stated connection-pooling strategy for Supabase Postgres.** Supabase has a hard cap on concurrent database connections. Whether Next.js runs as a persistent process or serverless functions changes how connections are opened and held; without pooling (Supabase's built-in Supavisor/pgbouncer), a moderate traffic spike can exhaust the connection limit and start failing requests platform-wide, well before CPU or bandwidth become the bottleneck.

---

## 9. Architecture Options Scorecard - Scalability, Resilience, Cost, Maintainability

Three concrete options, scored against the four axes actually asked for: scalability (latency + throughput), resilience, cost, and maintainability.

| Dimension | **A - Current Plan**<br>(Hostinger Business, WP + Next.js on one shared plan, proxy on Hostinger) | **B - Vercel-fronted**<br>(Next.js on Vercel, WP on cheap shared host, Vercel `rewrites` proxies marketing pages) | **C - Self-managed VPS**<br>(Both apps on one VPS, nginx reverse proxy, PM2/systemd) |
|---|---|---|---|
| **Latency** | Unproven, no CDN anywhere in the plan; SSR behind a Passenger-style Node manager on shared infra is an unknown quantity | Strong by default - Vercel's edge network serves the app globally; add Cloudflare in front of the WP origin to match | Depends entirely on manual setup - nothing is fast until you configure caching/CDN yourself |
| **Throughput / Scalability** | Weak - fixed shared CPU/RAM ceiling, no autoscaling, single box carries both systems | Strong - Vercel autoscales Next.js functions automatically under load; only the WP piece needs separate scaling thought | Moderate - can vertically resize the VPS, but no autoscaling without adding a load balancer and more boxes yourself |
| **Resilience** | Weak - WP and Next.js share fate on one host; reverse-proxy mechanism itself is unconfirmed (§1–2) | Strong - WP and app are fully separate failure domains; Vercel carries its own uptime/redundancy for the app half | Moderate - isolated from WP only if put on a second VPS; every bit of uptime, patching, and failover is on the team |
| **Cost** | Lowest *nominal* figure (~$17/mo hosting) - but the real cost is unknown until the §1 proof-of-concept either confirms it works or forces a pivot | Comparable in practice - Vercel Hobby is $0 but is **restricted to non-commercial use in Vercel's own terms**; a public nonprofit platform should budget for **Pro (~$20/mo)** to stay compliant. WP hosting elsewhere adds ~$5–10/mo | Comparable to the deck's own "Self-Hosted" row (~$22–64/mo per §3) - cash cost is low, but add the value of the volunteer time spent on ops |
| **Maintainability** | Low - requires proving and then permanently maintaining a non-standard "Node on shared PHP hosting" setup that could break silently on a Hostinger platform change | High - this is an extremely common, well-documented pattern (Next.js + rewrites to a separate origin); near-zero ops burden on the app side | Low - full ownership of OS patching, nginx config, process manager, and SSL renewal falls on the volunteer team indefinitely |

**Reading the scorecard:** Option B wins outright on three of the four axes (latency, throughput, resilience) and is maintainability-superior for the app layer specifically, at a cost difference that's smaller than it first appears once Option A's *unvalidated* status and Option B's *Pro-tier* requirement are both priced in honestly. Option A's only real advantage is that it's what's already been decided - which is not a technical advantage. **Recommend running the Option A proof-of-concept from §1 and a rough Option B spike in parallel**, and picking based on which one is actually working at the end of that week, not on which one looked cheaper on a slide.

But B still inherits the deck's core structural choice - two separate systems (WordPress + app) glued together. §10 questions whether that split needs to exist at all.

---

## 10. Proposed Target Architecture - Option D: Unified Stack, Near-Zero Cost

Everything above evaluates the deck's own framing: *how* to connect WordPress to Next.js, and on what hosting. This section steps back and asks whether that framing is the right one - because the reverse-proxy problem occupying §1–§2 (the biggest risk in the whole deck) only exists **because** the plan runs two separate systems that then need to be stitched together. A modern framework doesn't require that split.

### The core idea

**A single Next.js application can serve both the static marketing pages and the dynamic app layer natively.** Static pages (Home/Services/About/Contact) are pre-rendered at build time or incrementally revalidated and served from the edge - the same performance characteristic WordPress-behind-a-CDN would have. The app routes (auth, dashboard, booking, directory, events) run as dynamic routes in the *same* codebase. One codebase, one deploy, one domain. **No proxy to build, validate, or maintain - the entire content of §1 and §2 becomes moot.**

### What replaces WordPress's actual job

The one real requirement WordPress was solving is *"ops staff edit copy and images with no code."* Two ways to keep that without running WordPress at all:

- **Simplest (recommended):** a small `/admin/content` section inside the same Next.js app, gated by Supabase Auth's existing admin role, editing rows in a `page_content` table (Supabase Postgres) with images in Supabase Storage. Ops staff get **one** login and **one** dashboard for events, mentors, *and* page copy - not a second tool to context-switch into.
- **Alternative, closer to Elementor's feel:** a git-based visual CMS - Decap CMS or TinaCMS, both free and open-source. Edits commit to the repo and trigger an automatic redeploy. More setup than the table-based approach, but a more WYSIWYG-like editing experience if ops staff need that.

### Proposed stack - priced to be as close to $0 as the constraints allow

| Layer | Choice | Cost |
|---|---|---|
| Framework | Next.js (App Router) - one codebase for marketing + app | $0 |
| Hosting / CDN | Cloudflare Pages - free tier: unlimited bandwidth, global edge network, and (unlike Vercel Hobby) its terms don't restrict commercial/nonprofit use | $0 |
| Auth + DB + Storage | Supabase **Free tier** to start (500MB Postgres, 1GB storage, 50K monthly active users) | $0 until real usage outgrows it |
| Transactional email | Resend free tier (3,000 emails/mo) | $0 |
| Error monitoring | Sentry free tier (5K events/mo) - no reason to wait for "post-launch" per §5 | $0 |
| CI/CD + preview environments | GitHub Actions + Cloudflare Pages auto-deploy; free preview URL on every pull request | $0 |
| Domain + SSL | Registrar fee only - SSL is free everywhere now (Cloudflare / Let's Encrypt) | ~$1/mo |
| **Total at launch** | | **~$1/month** |

Supabase Pro ($25/mo) only becomes necessary once free-tier limits are actually hit by real traffic - that's a success signal to upgrade on, not a cost to plan around from day one. This is a materially better answer to *"if it can be zero cost, that would be ideal"* than anything in the original deck, including its own "Recommended" tier ($43–114/mo, §3).

**Caveat to validate, not assume:** Cloudflare Pages' Next.js support (via its build adapter) doesn't have full feature parity with Vercel - specifically confirm ISR/on-demand revalidation and middleware behave as needed for the auth-gated app routes. Vercel itself is more seamless for Next.js since it's built by the same team, but its Hobby tier's terms restrict non-commercial use (§9) - for a public nonprofit platform that likely means Vercel Pro (~$20/mo) as the honest comparison point instead of $0. Treat this the same way as the Option A/B feasibility question: **spike it, don't assume it**, in the same Kickoff week already set aside for that.

### How this addresses each constraint, explicitly

- **Performance (latency + throughput):** static pages are edge-cached by default across Cloudflare's global network; dynamic app routes run as edge/serverless functions that scale with traffic automatically - there is no shared CPU ceiling and no unproven Node-on-shared-hosting process to worry about. This resolves §1's central unknown and §8's missing-CDN gap in the same move, for free.
- **Cost:** ~$1/month at launch, scaling only with real, justified usage (see table above).
- **Iterative development:** one codebase means one CI/CD pipeline. Every pull request gets an automatic, shareable preview URL (both Cloudflare Pages and Vercel provide this free) - volunteer contributors can visually review each other's changes before merging, which is a fast, safe iteration loop suited to a distributed team, at no extra tooling cost.
- **Simplicity to maintain:** one framework, one hosting account, one auth/database provider, one admin panel covering events, mentors, *and* page content. No WordPress core or plugin security patching (historically the top attack surface on small nonprofit sites), no second database engine, no reverse proxy to keep working across host platform changes.
- **Resilience:** no shared-fate coupling between "marketing site" and "app" - they are the same deploy, so there is nothing to fall out of sync; edge functions don't share the single-box failure mode a shared-hosting plan has.

### The trade-off to raise honestly, not adopt silently

This is a bigger departure from the pptx than Option B - it drops WordPress entirely, which the team may already have mentally settled on (the "Design Fidelity vs. Effort" slide's day-estimates assume WordPress/Elementor as the build method). Ops staff also trade Elementor's full drag-and-drop visual builder for a smaller, purpose-built content-editing screen - simpler to run, but a genuinely different and less feature-rich editing experience. **This belongs in front of the team as a fourth option to weigh, not a silent substitution** - it changes an assumption the original deck treated as already decided.

### Where this fits

Spike this alongside Options A and B in the same Kickoff-week proof-of-concept already planned below - it's worth the same week's investment, since if it holds up it removes the need to resolve §1/§2 at all rather than just mitigating them.

---

## 11. Free-Tier Scalability Ceilings - What "Zero Cost" Actually Caps Out At

Direct answer to "do these layers scale on the free tier, or do they have ceilings too": **every one of them has a real ceiling.** None are infinite. The good news, verified against each provider's current published limits (not assumed from memory): the ceilings are **graduated pay-as-you-grow thresholds, not cliffs that take the site down** - with one exception worth planning around (Resend's daily cap, flagged below). Here are the actual numbers.

| Layer | Free-tier ceiling | What happens at the ceiling | Next tier & cost |
|---|---|---|---|
| **Cloudflare Pages** (static marketing pages) | **No request or bandwidth limit at all**, even on free - static assets are unmetered. The only cap is **500 builds/month**, 1 concurrent build, 20-min build timeout | Static-page serving effectively never hits a ceiling; you'd only run out of *deploys*, not traffic capacity | Pro: 5,000 builds/mo - not a scaling concern, a deploy-frequency one |
| **Cloudflare Workers** (dynamic app routes, via the Next.js adapter) | **100,000 requests/day** (resets daily) and **10ms CPU time per invocation** | Once the daily request cap is hit, further requests to dynamic routes fail *for the rest of that day* - this is the one place a real traffic spike (e.g., a popular event registration push) could cause a same-day outage on the app layer specifically | Paid plan is **$5/month minimum** - removes the daily cap entirely and raises CPU time to 30s default/5min max. Cheap enough to turn on proactively before a known high-traffic day rather than reactively after an outage |
| **Supabase** (auth, Postgres, storage) | 500MB database, 1GB file storage, 50,000 monthly active users, 5GB egress/month, 500,000 edge function invocations, capped at 2 free projects, **paused after 7 days of inactivity** | DB/storage: usage-based overage past Pro's included amounts, not a hard wall. Inactivity pause is the real near-term risk - a low-traffic staging project (or production between events) can go to sleep and needs a scheduled keep-alive ping | Pro: **$25/month** - 100K MAU (then metered), 8GB disk (then metered), 250GB egress (then metered), daily backups |
| **Resend** (transactional email) | **3,000 emails/month, capped at 100/day**, 3 sending domains | This is the **tightest real-world ceiling of the six** - a single well-attended event's confirmation + reminder emails can approach 100/day on its own. Sends beyond the daily cap don't queue, they fail - a mentee or mentor simply doesn't get their confirmation | Pro: **$20/month** - 50,000/month, **no daily cap**. Worth upgrading proactively before any event push, not after someone reports a missing confirmation |
| **Sentry** (error monitoring) | 5,000 errors/month, **1 user only**, 30-day retention | The user-count cap is the one likely to bite first, not event volume - if more than one volunteer developer needs dashboard access, free tier already doesn't fit | Team: **$26/month** - 50,000 errors, unlimited users, 90-day retention |
| **GitHub Actions** (CI/CD) | **2,000 minutes/month on a private repo - unlimited and free on a public repo** | Exceeding minutes on a private repo blocks further CI runs unless a payment method is on file | Pay-as-you-go past 2,000 min, or simply make the repo public if there's no reason it needs to be private - that removes this ceiling entirely at $0 |
| **Vercel Hobby** (if chosen instead of Cloudflare per §9/§10) | 100GB "Fast Data Transfer"/month, 1,000,000 function invocations/month, 4 CPU-hours active compute/month, 100 deployments/day, **10s default / 60s max function duration** | The 10s default duration is a real ceiling independent of traffic volume - a slow dashboard query or file-upload handler can time out under Hobby regardless of how few users hit it | Pro (~$20/mo, and likely required anyway per §9's ToS caveat): 15s default/300s max duration, usage-based beyond 1TB transfer |

### What this means in practice

- **Static pages effectively scale for free, indefinitely** (Cloudflare Pages has no meter on them at all) - the "zero cost" claim in §10 holds up best exactly where it matters most for a mostly-static marketing site.
- **The ceiling you'll hit first, in realistic order, is Resend's 100-emails/day cap** - tied directly to event and booking volume, which is the platform's actual purpose. Recommend treating the $20/mo Pro upgrade as a near-certain early cost, not a hypothetical one, and budgeting for it explicitly rather than assuming email stays free indefinitely.
- **The one ceiling that's a genuine availability risk, not just a cost trigger, is Cloudflare Workers' daily request cap** - it fails closed for the rest of the day rather than degrading gracefully. For any planned high-traffic moment (a popular event opening registration), flip to the $5/mo Workers paid plan *ahead of time* rather than discovering the cap during the spike.
- **Sentry and GitHub Actions ceilings are organizational, not load-related** - they'll bite based on team size and CI usage, not user traffic, so they're predictable and controllable rather than something usage growth forces on you.
- None of this changes the §10 conclusion: **genuinely realistic launch-scale usage for a community mentorship platform fits inside these free tiers**, with Resend's email cap as the one line item worth budgeting for from day one rather than treating as a someday-maybe cost.

**Sources (verified 2026-08-26 - free-tier terms change; re-check before relying on these for a launch-date decision):**
- [Cloudflare Workers Pricing](https://developers.cloudflare.com/workers/platform/pricing/)
- [Cloudflare Pages limits (free plan) - Cloudflare Community](https://community.cloudflare.com/t/cloudflare-pages-limits-free-plan/431961)
- [Supabase Pricing](https://supabase.com/pricing)
- [Resend Pricing](https://resend.com/pricing)
- [Sentry Pricing](https://sentry.io/pricing/)
- [GitHub Actions billing docs](https://docs.github.com/en/billing/managing-billing-for-your-products/managing-billing-for-github-actions/about-billing-for-github-actions)
- [Vercel Limits docs](https://vercel.com/docs/limits/overview)

---

## 12. Iterative Build Roadmap - Bi-Weekly Checkpoints (Aug 26 → Oct 31, 2026)

Structured so the platform is **usable end-to-end well before Oct 31** (per the team's intent), with remaining features and hardening layered on top through the final date. Each checkpoint assumes the prior one's deliverables are done - slipping one pushes everything after it.

| Checkpoint | Date | Focus | Key Deliverables |
|---|---|---|---|
| **Kickoff** | Aug 26 | De-risk the architecture decision | Spike Options A, B, and D (§1, §9, §10) in parallel over the same week; pick one based on what actually works, not what looked cheapest on a slide. Correct and re-approve the cost table (§3). Lock the Supabase role-model decision (§4 Q6). |
| **CP1** | Sep 9 | Foundations | Chosen hosting stood up (Hostinger, Vercel, or Cloudflare Pages per the PoC outcome); Supabase project created with auth/roles schema - **start on the Free tier per §10 unless the PoC shows otherwise**; CDN in front of the marketing pages; domain/DNS/SSL live; Figma Dev Mode tokens extracted. |
| **CP2** | Sep 23 | Static site + app skeleton, in staging | **If A or B was chosen:** WordPress marketing pages (Home/Services/About/Contact) built to Figma spec, plus a Next.js app skeleton (auth + routing) deployed to staging, with the reverse-proxy/rewrite path validated end-to-end with a real logged-in session before either codebase goes further. **If D was chosen:** marketing pages built as static/SSG routes in the same Next.js app, plus the `/admin/content` editing screen (or Decap/TinaCMS per §10) wired up and confirmed usable by non-technical ops staff - no proxy step needed either way. |
| **CP3** | Oct 7 | **Functional MVP - core path works end-to-end** | Mentor directory + profiles (search/filter); mentee/mentor/volunteer applications with resume upload (basic validation + access control per §5); booking engine v1 with an **atomic** capacity check (closing the §8 race-condition gap); event creation (admin) + public registration with atomic seat enforcement; transactional email confirmations wired up. *This is the milestone where a mentee can browse, apply, book, and get confirmed, and an admin can run an event - the "functions fully" bar the team asked for, even though features are still incomplete.* |
| **CP4** | Oct 21 | Dashboards, ops tooling, hardening | Mentor dashboard (stats, incoming requests, availability manager); ops/admin dashboard (manage mentors/mentees/volunteers, application review, background-check workflow); admin-configurable notifications; Sentry enabled (free tier - no reason to wait per §5); RBAC and file-storage policy audit; a deliberate concurrent-load test against booking and event registration to confirm the §8 atomicity fix actually holds. |
| **CP5 - Final** | Oct 31 | Polish, QA, launch readiness | Full visual QA against Figma across WordPress and Next.js; mobile/cross-browser pass; CDN cache and latency spot-check; backup/restore verified on both WordPress and Supabase; a written maintenance runbook (who patches what, per §5); a final, explicit list reconciled against the requirements doc of what shipped in v1 vs. what's deliberately deferred (calendar sync, payment/monetization, live session/video delivery, ratings - per `requirements-feedback.md`). |

**Note on the Oct 31 date:** it is treated here as the full-architecture completion date, not the first usable date - CP3 (Oct 7) is where the platform becomes genuinely functional for its core flow, consistent with the team's own framing that the system should "function fully before" the final completion date. If the Kickoff PoC in §1/§9 takes longer than expected, compress CP4's scope (push non-critical dashboard polish to a post-Oct-31 v1.1) rather than delaying CP3 - the functional milestone is more valuable to protect than full feature completeness by the deadline.