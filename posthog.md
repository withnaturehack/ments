# PostHog Analytics — Ments

Complete reference for how PostHog is integrated, what it tracks, and how to use the data.

---

## Architecture Overview

```
Browser (user action)
        │
        ▼
PostHogProvider  ──────►  posthog-js  ──────►  PostHog Cloud (us.i.posthog.com)
  (client-side)                                         │
        │                                               │
UserIdentifier                                 ┌────────┴────────┐
  identifies user                          Dashboard        Session
  on login/logout                         Insights         Recordings
        │
        ▼
captureServerEvent  ──►  posthog-node  ──►  PostHog Cloud
  (server-side API                            (same project,
   routes only)                               unified profile)
```

### Files

| File | Role |
|------|------|
| `src/components/PostHogProvider.tsx` | Client wrapper — initialises PostHog, tracks page views, identifies users |
| `src/lib/posthog-events.ts` | Typed event name constants + `track()` helper for client components |
| `src/lib/posthog-server.ts` | Server-side PostHog Node client + `captureServerEvent()` for API routes |
| `src/app/layout.tsx` | Mounts `<PostHogProvider>` inside `<AuthProvider>` so identity is always available |

---

## Environment Variables

| Variable | Where to set | Value |
|----------|-------------|-------|
| `NEXT_PUBLIC_POSTHOG_KEY` | Replit Secrets | `phc_…` — find in PostHog → Project Settings → Project API Key |
| `NEXT_PUBLIC_POSTHOG_HOST` | Replit Env Vars | `https://us.i.posthog.com` (default, already set) |

> **Note:** `NEXT_PUBLIC_POSTHOG_KEY` is intentionally public — PostHog's project key is designed to be bundled in the browser. It controls *ingestion* only; your data stays private inside your PostHog account.

---

## What Is Tracked Automatically

No code changes needed. These fire on every page load:

### Page Views
Every route change fires a `$pageview` event with:
- `$current_url` — full URL including query string
- `$pathname` — path only
- `$referrer` — where the user came from
- `$host` — domain

### Autocapture
PostHog automatically captures:
- Button clicks (with text/label)
- Link clicks (with `href`)
- Form submissions
- Input changes (values masked on `type="password"` fields)

### Session Recordings
Full video-like recordings of every session. Passwords are automatically masked. Accessible at **PostHog → Recordings**.

### User Identity
When a user signs in, their PostHog anonymous profile is merged with:

```typescript
posthog.identify(user.id, {
  email: user.email,
  name: user.user_metadata?.full_name,
  avatar: user.user_metadata?.avatar_url,
  created_at: user.created_at,
  provider: user.app_metadata?.provider,  // "google" or "email"
})
```

When they sign out, `posthog.reset()` is called so the next session starts anonymous again.

---

## Custom Events Reference

### Auth Events

| Event | When it fires | Properties |
|-------|--------------|------------|
| `signup_initiated` | User submits the Create Account form | `method: "email"` |
| `signup_completed` | Account created successfully | `method`, `needs_confirmation: true` |
| `signup_failed` | Signup error returned | `method`, `error: "message"` |
| `google_signup_clicked` | "Sign up with Google" button clicked | — |

### Onboarding Events

| Event | When it fires | Properties |
|-------|--------------|------------|
| `onboarding_interests_set` | User taps "Get Started" on onboarding | `interests: ["building","investing"]`, `count: 2` |
| `onboarding_completed` | Onboarding POST succeeds, user redirected | `interests`, `redirect: "/explore"` |

### Social Events

| Event | When it fires | Properties |
|-------|--------------|------------|
| `user_followed` | User clicks Follow on People page | `followed_username`, `followed_user_id`, `source: "people_page"` |

### Investor Events

| Event | When it fires | Properties |
|-------|--------------|------------|
| `investor_connection_sent` | Investor connects to a startup (server-side) | `startup_id`, `startup_name`, `has_intro_message: true/false` |

### Application Events

| Event | When it fires | Properties |
|-------|--------------|------------|
| `application_started` | User starts a job/gig/event application (server-side) | `type: "job"/"gig"/"event"`, `job_id`/`gig_id`/`event_id`, `match_score: 82` |

---

## How to Add a New Tracked Event

### In a Client Component

```typescript
import { track, EVENTS } from '@/lib/posthog-events';

// Inside a handler:
track(EVENTS.POST_LIKED, { post_id: '123', author_username: 'john' });
```

### Adding a New Event Name

Open `src/lib/posthog-events.ts` and add to the `EVENTS` object:

```typescript
export const EVENTS = {
  // ...existing...
  POST_LIKED: 'post_liked',
} as const;
```

TypeScript will enforce you only use defined event names everywhere.

### In a Server-Side API Route

```typescript
import { captureServerEvent } from '@/lib/posthog-server';

// After a successful DB write:
await captureServerEvent(user.id, 'post_created', {
  post_id: newPost.id,
  has_media: !!mediaUrl,
  word_count: content.split(' ').length,
});
```

The server client uses `flushAt: 1` + `flushInterval: 0` so every server event is flushed immediately — no batching delay.

---

## Building Funnels in PostHog

### Signup → Onboarding → Active User Funnel

1. Go to **PostHog → Insights → New insight → Funnel**
2. Add steps:
   - Step 1: `signup_completed`
   - Step 2: `onboarding_completed`
   - Step 3: `$pageview` (filter: `$pathname = /startups`)
3. Set conversion window: **7 days**
4. Click **Save** — name it "New User Activation"

### Job Application Funnel

1. New Funnel insight
2. Steps:
   - `$pageview` where `$pathname contains /hub/job/`
   - `application_started` where `type = job`
   - `application_submitted`
3. This shows what % of people who view a job actually apply

---

## Useful Queries in PostHog

### Who are my most active users?
**Insights → Trends → Event:** `$pageview` → **Breakdown by:** Person → `email`

### What interests do new users pick?
**Insights → Trends → Event:** `onboarding_interests_set` → **Breakdown by:** Event property → `interests`

### Where do signups fail?
**Insights → Trends → Event:** `signup_failed` → **Breakdown by:** Event property → `error`

### How long until first connection?
**Insights → User Paths → Starting from:** `onboarding_completed` → **Ending at:** `investor_connection_sent`

---

## Session Recordings

Go to **PostHog → Recordings**.

- Filter by **Person** (email) to watch a specific user's sessions
- Filter by **Event contains** `signup_failed` to watch sessions where users struggled to sign up
- Filter by **Duration** > 5 min to find power users

Recordings automatically mask all `type="password"` inputs. Other sensitive fields can be masked by adding `ph-no-capture` class to any element.

---

## Feature Flags (Optional — Future Use)

PostHog feature flags let you roll out features to a % of users without deploying new code.

```typescript
import { useFeatureFlagEnabled } from 'posthog-js/react';

function MyComponent() {
  const showNewFeed = useFeatureFlagEnabled('new-feed-algorithm');
  return showNewFeed ? <NewFeed /> : <OldFeed />;
}
```

Create flags at **PostHog → Feature Flags → New feature flag**.

---

## Privacy & Compliance

- **Passwords:** Always masked in recordings (`maskAllInputs: true` + custom `maskInputFn`)
- **No PII in event names:** All sensitive data goes in properties only, never event names
- **User opt-out:** Call `posthog.opt_out_capturing()` to fully disable for a user
- **Data residency:** Using `https://us.i.posthog.com` (US region). Switch to `https://eu.i.posthog.com` for EU data residency
- **Cookie-free option:** PostHog uses `localStorage` persistence by default (set in init config) — no tracking cookie needed


