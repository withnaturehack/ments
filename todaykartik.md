# Today's Work — Kartik
**Date:** June 20, 2026

**Custom events we wired up:**
| Action | Event |
|--------|-------|
| Email signup started | `signup_initiated` |
| Signup completed | `signup_completed` |
| Signup failed | `signup_failed` |
| Google signup clicked | `google_signup_clicked` |
| Onboarding interests picked | `onboarding_interests_set` |
| Onboarding finished | `onboarding_completed` |
| Followed someone | `user_followed` |
| Investor connected to startup | `investor_connection_sent` |
| Application started | `application_started` |

**Where to see it:** [app.posthog.com](https://app.posthog.com)
- Activity tab → live events
- Insights → build funnels
- Recordings → watch real sessions

**Full reference doc:** `posthog.md`

### 4. Full SEO Overhaul

**New files:**
- `src/app/robots.ts` → tells Google what pages to crawl
- `src/app/sitemap.ts` → gives Google every URL (startups, mentors, jobs, gigs) from your live DB

**Static pages with proper metadata:**
`/register`, `/explore`, `/startups`, `/hub`, `/trending`, `/people`

**Dynamic pages that pull real data for SEO:**
- `/startups/[id]` → startup name + logo in Google/Slack previews
- `/mentor/[username]` → mentor name + photo
- `/hub/job/[id]` → job title + company
- `/hub/gig/[id]` → gig title + budget
- `/hub/event/[id]` → event name + banner image

**Root layout improvements:**
- Title template: `"Page Name | Ments"` on every page
- Full Open Graph (WhatsApp, LinkedIn, Slack previews)
- Twitter Cards
- Keywords, canonical URLs, robots directives

---

## Files Created/Changed Today

| File | What it does |
|------|-------------|
| `src/components/PostHogProvider.tsx` | PostHog client wrapper |
| `src/lib/posthog-events.ts` | Typed event names + track() helper |
| `src/lib/posthog-server.ts` | Server-side PostHog for API routes |
| `src/app/robots.ts` | `/robots.txt` |
| `src/app/sitemap.ts` | `/sitemap.xml` (dynamic from DB) |
| `src/app/layout.tsx` | Updated — full SEO metadata + PostHogProvider |
| `src/app/register/layout.tsx` | Register page SEO |
| `src/app/explore/layout.tsx` | Explore page SEO |
| `src/app/startups/layout.tsx` | Startups page SEO |
| `src/app/hub/layout.tsx` | Hub page SEO |
| `src/app/trending/layout.tsx` | Trending page SEO |
| `src/app/people/layout.tsx` | People page SEO |
| `src/app/startups/[id]/layout.tsx` | Dynamic startup SEO |
| `src/app/mentor/[username]/layout.tsx` | Dynamic mentor SEO |
| `src/app/hub/job/[id]/layout.tsx` | Dynamic job SEO |
| `src/app/hub/gig/[id]/layout.tsx` | Dynamic gig SEO |
| `src/app/hub/event/[id]/layout.tsx` | Dynamic event SEO |
| `src/app/register/page.tsx` | + PostHog signup tracking |
| `src/app/onboarding/page.tsx` | + PostHog onboarding tracking |
| `src/app/people/page.tsx` | + PostHog follow tracking |
| `src/app/api/investor/connections/route.ts` | + PostHog connection tracking |
| `src/app/api/applications/start/route.ts` | + PostHog application tracking |
| `next.config.ts` | Fixed iframe preview + allowedDevOrigins |
| `posthog.md` | Full PostHog reference doc |
| `todaykartik.md` | This file |

---

## Tomorrow / Next Steps

1. **Add missing secrets** in Replit Secrets tab (GROQ, Redis token, SMTP password)
2. **Set `NEXT_PUBLIC_SITE_URL=https://ments.app`** so canonical URLs are correct in production
3. **Submit sitemap** to Google Search Console → `https://ments.app/sitemap.xml`
4. **Check PostHog dashboard** — verify events are arriving from real users
5. **Add more event tracking** — post creation, post likes, profile views

---

## Deep Codebase Analysis — What Needs to Be Improved

> This is a real audit from reading the actual code. Not guesses — specific files and lines.
> Each item includes: **what's broken/weak**, **why it matters**, and **what to do about it**.

---

### SECTION A — Feed Engine (Critical — this is the core product)

#### A1. LLM Ranking Has No Retry or Fallback Model
**File:** `src/lib/feed/llm-ranker.ts`

**What's wrong:** The LLM re-ranker uses short 8-character IDs in the prompt to save tokens. If the AI model returns a hallucinated or malformed ID, those posts just silently fall back to chronological order. There is zero retry logic, and the model is hardcoded to `amazon.nova-pro-v1:0` — if that model is unavailable or rate-limited, the entire Tier 2 ranking fails silently.

**Why it matters:** Your feed quality IS your product. If the AI ranker fails quietly, users see a worse feed without anyone knowing. Over time this erodes trust without leaving a trace.

**Fix:** Add a JSON schema validator on the LLM response. If validation fails, retry once with a simplified prompt. Add a fallback to `amazon.nova-lite-v1:0`. Log failures to PostHog as `feed_llm_ranking_failed` so you can see the rate.

---

#### A2. LLM Ranking Blocks the Entire Request (500ms–1.5s Added Latency)
**File:** `src/lib/feed/pipeline.ts`

**What's wrong:** Tier 2 LLM re-ranking runs synchronously in the main `generatePersonalizedFeed` pipeline. Every cold-cache feed request waits for an AWS Bedrock round-trip before anything returns to the user.

**Why it matters:** 500ms+ latency on the main feed is a significant retention killer. Research consistently shows that every 100ms of load time reduces engagement. Users on mobile networks will see a blank feed for 1–2 seconds regularly.

**Fix:** Serve Tier 1 (fast heuristic) results immediately from cache. Fire the LLM re-ranker in the background and update the feed order via a second request or websocket push. Users see content instantly; the smarter order arrives within seconds.

---

#### A3. New Creator Boost Uses Wrong Signal
**File:** `src/lib/feed/reranker.ts` (line ~41)

**What's wrong:** The "boost new creators" feature identifies new creators by `follower_count_normalized < 0.01` — i.e., anyone with very few followers. There's even a comment in the code acknowledging this should use account creation date, but it was never implemented.

**Why it matters:** This means established creators who lost followers, or niche users who never grew, also get boosted. It's inaccurate and could surface low-quality content from veteran accounts incorrectly.

**Fix:** Add account creation date to the scoring query. New creator = created within last 90 days AND fewer than 50 followers.

---

#### A4. Cache Miss Fallback Does a `NOT IN` Query on Potentially Thousands of IDs
**File:** `src/app/api/feed/route.ts` (line ~190)

**What's wrong:** When the feed cache is cold, the fallback chronological query excludes already-seen post IDs using a `NOT IN (...)` clause. As a user sees more posts, this list grows unboundedly.

**Why it matters:** Postgres performance degrades significantly with large `NOT IN` lists. For an active user who has scrolled hundreds of posts, this query could take seconds and time out.

**Fix:** Use cursor-based pagination (a `before_timestamp` or `last_seen_post_id` cursor) instead of exclusion lists. This keeps the query O(1) regardless of history length.

---

#### A5. Real-time Post Injection Re-scores on Every Cache-Hit Request
**File:** `src/lib/feed/realtime-injector.ts` (line ~12)

**What's wrong:** Even when the main feed is served from cache (fast path), the real-time injector fetches new posts since the last cache computation, extracts their features, and re-scores them on every request.

**Why it matters:** This negates the performance benefit of caching. A user who refreshes frequently will cause repeated Bedrock + Supabase calls.

**Fix:** Cache the injected real-time posts separately with a short TTL (30–60 seconds). Only re-fetch if that short-TTL cache also misses.

---

### SECTION B — Messaging System (High Priority — core engagement)

#### B1. Two Completely Separate Message UIs That Are Diverging
**Files:** `src/app/messages/[conversationId]/page.tsx` vs `src/components/chat/ChatPage.tsx`

**What's wrong:** There are two entirely different messaging implementations. The main messages page supports reactions, blocking, and bulk delete. The Hub/Explore chat (`ChatPage.tsx`) supports chat request approvals and infinite scroll — but has none of the reactions or blocking features.

**Why it matters:** Users experience different functionality depending on how they open a conversation. This is a support nightmare and will get worse as each version drifts further. Features built in one place don't automatically appear in the other.

**Fix:** Unify into a single `<MessageThread>` component with feature flags. Both routes render the same component, features are toggled by props based on context.

---

#### B2. Image Upload in Chat Is a `console.log` Placeholder
**File:** `src/components/chat/ChatInput.tsx` (line ~123)

**What's wrong:** The UI shows icons for image upload and voice recording. The `handleImageSelect` function is literally just `console.log`. This is dead UI — clicking the image button does nothing.

**Why it matters:** If users discover this, it damages trust. Buttons that do nothing are worse than buttons that don't exist.

**Fix:** Either wire it up to your media storage pipeline (Supabase storage bucket) or remove the icon entirely until it's built.

---

#### B3. Mentor Booking Has a Race Condition — Double Booking Is Possible
**File:** `src/app/api/mentor/book/route.ts` (line ~18)

**What's wrong:** When a user books a time slot, the API inserts the booking without first checking if that slot was already booked by someone else in the same second. Two users can hit the endpoint simultaneously and both get a success response for the same slot.

**Why it matters:** A mentor shows up for a session and two people are waiting. This is an embarrassing and trust-destroying experience.

**Fix:** Wrap the insert in a database transaction with a unique constraint on `(mentor_id, slot_time)`. The second insert will fail with a constraint error and return a "slot no longer available" message.

---

#### B4. Notification Bell Shows No Count
**File:** `src/components/layout/NotificationBell.tsx` (line ~53)

**What's wrong:** The notification bell shows a pulsing dot when there are unread notifications. It never shows a number (e.g. "3 unread"). The dot is based on a client-side `isEnabled` flag, not an actual unread count from the server.

**Why it matters:** Users on major platforms (Instagram, LinkedIn, Twitter) expect a count. A dot with no number provides no urgency signal — users learn to ignore it.

**Fix:** Add an unread count query (one simple Supabase count query) on load and after each notification event. Display the count as a small badge.

---

### SECTION C — Startup & Investor Features

#### C1. Startup Wizard Has No Autosave — Long Form with Total Data Loss Risk
**File:** `src/components/startups/StartupCreateWizard.tsx` (line ~286)

**What's wrong:** The 8-step startup creation wizard only saves to the database on the final submit. If a user fills out 7 steps and their browser crashes, tab closes, or they accidentally navigate away — all data is lost.

**Why it matters:** The startup wizard is one of the highest-value flows in the product. Losing a half-completed startup form is likely the #1 reason founders don't complete onboarding. It also means your "incomplete startup" rate in PostHog is inflated by technical data loss, not just genuine abandonment.

**Fix:** Autosave to `localStorage` after every step. On page load, check for a saved draft and offer to restore it. Optionally, also save a draft record in the DB after step 3.

---

#### C2. Startup Financial Data Displayed as Plain Text — No Visualisation
**File:** `src/components/startups/StartupProfileView.tsx` (line ~71)

**What's wrong:** Revenue, burn rate, MRR growth and other traction metrics are collected in structured form but displayed as raw text labels on the startup profile. There are no charts, graphs, or visual traction indicators.

**Why it matters:** Investors make snap judgments. A text label saying "MRR: $8,000" is far less compelling than a 6-month growth chart. Competing platforms (Visible, Carta) show this visually. You're collecting the data but not using it to its full potential.

**Fix:** Add a simple line chart (Recharts or Chart.js — already likely in the project) for MRR growth and a burn rate bar. Even a single sparkline chart makes the profile significantly more investor-ready.

---

#### C3. Investor Match Scoring Is a Heuristic Mock, Not a Real Algorithm
**File:** `src/app/api/investor/deal-flow/route.ts` (line ~72)

**What's wrong:** The investor–startup match score adds fixed points for sector fit (+25), stage match (+20), etc. The comment in the code calls it "heuristic-based." It's essentially a lookup table, not a meaningful match score.

**Why it matters:** Investors will notice quickly if the "match" percentage doesn't reflect real compatibility. This is a core value prop for investor users — if it feels random, they won't trust the platform.

**Fix:** Feed the actual investor preferences (sectors, stages, check sizes, geography) and startup profile into the LLM ranker you already have. Generate a genuine compatibility summary. This is a high-leverage use of infrastructure you've already built.

---

#### C4. Invest Arena — Real-time Leaderboard Requires Manual Refresh
**File:** `src/app/invest/arena/[eventId]/[stallId]/page.tsx` (line ~101)

**What's wrong:** During a live pitch event, the investment leaderboard only updates when the user navigates or refreshes the page. In a room of investors all allocating capital simultaneously, seeing a stale leaderboard kills the competitive energy.

**Why it matters:** The arena's value is real-time — people want to see their ranking live. A static leaderboard in a live event context is a broken experience.

**Fix:** Subscribe to Supabase real-time changes on the `investments` table for the event. The leaderboard updates live for all participants without any refresh.

---

#### C5. Invest Arena — Guest Users Can Invest With Fake Names
**File:** `src/app/invest/arena/[eventId]/[token]/page.tsx` (line ~184)

**What's wrong:** Guest investors (non-registered users invited via token/QR code) enter only a name and email before investing virtual capital. There is no OTP, no email verification, and no backend uniqueness check beyond the token itself.

**Why it matters:** In a competitive event, a bad actor could open multiple incognito tabs with the same token and vote multiple times. This could distort leaderboard results and invalidate the entire event.

**Fix:** Send a 6-digit OTP to the email the guest enters. Verify it before they can allocate capital. Store the verified email in the session to prevent re-entry.

---

### SECTION D — Auth, Search & UX

#### D1. Account Deletion Opens a `mailto:` Link — GDPR Risk
**File:** `src/app/settings/page.tsx` (line ~466)

**What's wrong:** The "Delete Account" option in Settings opens a pre-filled email to support. This is a manual, async, human-dependent process.

**Why it matters:** GDPR (EU), CCPA (California), and DPDP (India) all require that users can delete their data without friction. A `mailto:` link does not satisfy this legally. If Ments ever has EU or California users, this is a compliance liability. It also just feels terrible as a user experience.

**Fix:** Build a proper in-app deletion flow with a confirmation step, soft-delete the user record, queue a background job to purge their data from all tables, and send a confirmation email.

---

#### D2. Search Uses Basic `ilike` — No Typo Tolerance
**File:** `src/app/api/search/route.ts`

**What's wrong:** Search uses Postgres `ilike '%term%'` pattern matching. This means:
- Searching "kartk" won't find "kartik"
- Searching "machne learning" won't find "machine learning"
- Results are not ranked by relevance — a verified user doesn't appear before an unverified one

**Why it matters:** Search is often the first interaction new users have. If they can't find the person or startup they're looking for, they assume the platform is small or broken. For a community platform, this is a retention problem.

**Fix:** Enable Postgres `pg_trgm` extension (already available in Supabase) for fuzzy matching. Sort results by `similarity()` score. Long-term: consider Supabase's built-in full-text search or integrate Typesense.

---

#### D3. Profile Edit Is Split Across Two Completely Different Places
**Files:** `src/app/profile/edit/page.tsx` vs `src/app/settings/profile/page.tsx`

**What's wrong:** There are two separate pages for editing a user profile. One is accessed from the public profile page, another is inside Settings. They likely update different subsets of the same fields.

**Why it matters:** Users don't know which one to use. If they update their tagline in one place and it doesn't appear in the other, they'll think the app is broken. Support requests will increase.

**Fix:** Consolidate into a single canonical profile edit experience. The Settings route should redirect to it, or the Settings version should be removed entirely.

---

#### D4. Register Form Has One Global Error State — No Field-Level Errors
**File:** `src/app/register/page.tsx`

**What's wrong:** When signup fails (e.g., email already taken), the entire form shows one generic error message at the top. There is no per-field validation — entering an invalid email doesn't highlight the email field, and no password strength indicator exists.

**Why it matters:** Poor form UX increases abandonment. If someone doesn't know their email is already registered vs. their password is too weak, they'll try random things and give up. Every % of register completion improvement is direct user growth.

**Fix:** Add field-level validation with inline error messages. Show password strength as a visual bar. On "email already in use," show the error under the email field with a "Sign in instead?" link.

---

### SECTION E — Performance & Code Quality

#### E1. Multiple Supabase Client Instances Being Created
**Pattern observed across:** multiple page components

**What's wrong:** Several client-side components call `createClient()` directly at the top level of the component instead of using a shared context or singleton. This can result in multiple simultaneous Supabase auth subscriptions being opened.

**Why it matters:** Each Supabase client instance opens a websocket. Multiple clients = multiple open sockets = memory leaks, duplicated auth events, and unexpected behaviour when one instance gets a session update the others don't.

**Fix:** Enforce a single shared Supabase client instance via React context or a module-level singleton. Already partially done with `supabase-server.ts` on the server side — apply the same pattern client-side.

---

#### E2. Media Upload Has No File Size or Type Validation on the Client
**Observed in:** startup wizard, profile picture upload, resume upload

**What's wrong:** File pickers accept uploads but validation only happens (if at all) server-side or after upload begins. A user can attempt to upload a 500MB video to what expects a profile picture.

**Why it matters:** Large accidental uploads consume your Supabase storage quota, slow the upload experience for the user, and can cause silent failures. Resume parsing (`/api/resume/parse`) failing on a 50MB file will timeout.

**Fix:** Add client-side validation before upload begins: check `file.size` against a max (e.g. 5MB for images, 50MB for pitch videos), validate `file.type` against an allowed list, and show a clear error immediately.

---

#### E3. PostHog Analytics Gaps — Key Flows Not Tracked
**Current coverage vs. what's missing:**

| Flow | Tracked? |
|------|---------|
| Signup funnel | ✅ Yes |
| Onboarding | ✅ Yes |
| Follow user | ✅ Yes |
| View startup profile | ❌ No |
| View mentor profile | ❌ No |
| Send a message | ❌ No |
| Create a post | ❌ No |
| Post a job/gig | ❌ No |
| Search something | ❌ No |
| Startup wizard step progress | ❌ No (critical for funnel) |
| Feed scroll depth | ❌ No |
| Pitch reel swipe | ❌ No |

**Why it matters:** Without tracking startup wizard step completions, you can't know which step loses founders. Without tracking message sends, you can't measure if networking is actually happening.

**Fix:** Add `startup_wizard_step_completed` (with step number), `profile_viewed`, `message_sent`, `post_created`, `search_performed`, `feed_scrolled` as the next batch of events.

---

### Priority Order for Fixes

| # | Fix | Impact | Effort |
|---|-----|--------|--------|
| 1 | Mentor booking race condition (double booking) | 🔴 Critical | Low |
| 2 | Startup wizard autosave | 🔴 Critical | Medium |
| 3 | Account deletion (GDPR) | 🔴 Critical | Medium |
| 4 | Invest arena double-voting by guests | 🔴 Critical | Medium |
| 5 | LLM ranker retry + failure logging | 🟠 High | Low |
| 6 | Feed LLM latency (async ranking) | 🟠 High | High |
| 7 | Unify messaging UI (two codebases) | 🟠 High | High |
| 8 | Remove dead image upload button in chat | 🟠 High | Low |
| 9 | Notification bell show count | 🟡 Medium | Low |
| 10 | Invest arena live leaderboard | 🟡 Medium | Low |
| 11 | Financial data charts on startup profile | 🟡 Medium | Medium |
| 12 | Search fuzzy matching (`pg_trgm`) | 🟡 Medium | Low |
| 13 | Register form field-level errors | 🟡 Medium | Low |
| 14 | More PostHog events (wizard steps, messages) | 🟡 Medium | Low |
| 15 | Consolidate profile edit pages | 🟡 Medium | Medium |
| 16 | Investor match algorithm (real scoring) | 🟢 Long-term | High |
