# Tyagi's Twitter (TT)

**AI-Powered Twitter Growth Engine for @tyagicapx**

---

## Overview

Tyagi's Twitter is a private, single-user web application built to automate and accelerate Bhatia Tyagi's Twitter presence. The system operates as a two-pronged content engine: it generates original tweet drafts on a daily schedule and surfaces high-value reply opportunities from monitored accounts through an AI-powered ingestion pipeline. All content is filtered, ranked, and presented in a Tinder-style swipe interface for rapid human-in-the-loop review before publishing via Typefully.

The application runs on a React 19 + Express 4 + tRPC 11 stack, backed by a MySQL (TiDB) database, and is hosted on Manus infrastructure at `tyagitweet-mu6ci3ur.manus.space` with a custom domain at `tyagitwitter.bluecow.xyz`. Authentication is handled via a secret PIN (no OAuth dependency), and all LLM calls default to the free built-in model to minimize operational costs.

---

## Architecture

### Tech Stack

| Layer | Technology | Notes |
|-------|-----------|-------|
| Frontend | React 19, Tailwind CSS 4, shadcn/ui | Single-page application with client-side routing via Wouter |
| Backend | Express 4, tRPC 11, TypeScript | Type-safe RPC layer with Superjson serialization |
| Database | MySQL (TiDB) via Drizzle ORM | Schema-first workflow with migration tooling |
| LLM | Built-in Manus model (default), Claude, OpenAI | Configurable via Settings; cron jobs use the free built-in model |
| Publishing | Typefully API | Draft creation and direct posting to Twitter/X |
| Notifications | Telegram Bot API | Daily tweet batches sent to a private Telegram chat for review |
| Scheduling | node-cron (in-process) | Self-contained with keep-alive and missed-job catch-up |
| Hosting | Manus Platform | Built-in hosting with custom domain support |

### File Structure

```
tyagis-twitter/
├── client/
│   ├── src/
│   │   ├── components/
│   │   │   ├── AppLayout.tsx          ← Pill nav + profile dropdown + mobile bottom tabs
│   │   │   ├── SwipeCard.tsx          ← Tinder-style draggable reply card
│   │   │   ├── DashboardLayout.tsx    ← Legacy sidebar layout (unused)
│   │   │   └── ...ui/                 ← shadcn/ui primitives
│   │   ├── pages/
│   │   │   ├── Home.tsx               ← Dashboard with pipeline metrics + strategy breakdown
│   │   │   ├── RepliesPage.tsx        ← Swipe deck + sources management (sub-tab)
│   │   │   ├── TweetContent.tsx       ← Generated tweet drafts + compose + Typefully posting
│   │   │   ├── Settings.tsx           ← Model config, API keys, filter pipeline settings
│   │   │   └── PinLogin.tsx           ← Numpad-style PIN authentication
│   │   ├── App.tsx                    ← Route definitions
│   │   └── index.css                  ← TT Aesthetic design tokens
│   └── index.html                     ← Inter font via Google Fonts
├── server/
│   ├── _core/                         ← Framework plumbing (OAuth, context, LLM helpers)
│   ├── cronManager.ts                 ← Cron v2: keep-alive, catch-up, DB-tracked last-run
│   ├── db.ts                          ← All database query helpers
│   ├── routers.ts                     ← tRPC procedure definitions
│   ├── llmRouter.ts                   ← LLM integration (tweet gen, reply gen, persona prompt)
│   ├── filterPipeline.ts              ← Structural + LLM-based tweet filtering
│   ├── ingestionPipeline.ts           ← Full fetch → filter → generate → card pipeline
│   ├── twitterFetcher.ts              ← twitterapi.io client (standalone, no Manus deps)
│   ├── telegram.ts                    ← Telegram Bot API helper
│   ├── typefully.ts                   ← Typefully API helper
│   └── storage.ts                     ← S3 file storage helpers
├── drizzle/
│   └── schema.ts                      ← All table definitions
└── shared/                            ← Shared constants and types
```

---

## Core Features

### 1. Reply Pipeline (Swipe System)

The reply pipeline is the primary feature of the application. It operates as a fully automated ingestion system that discovers reply-worthy tweets and presents them for rapid review.

**Pipeline stages:**

1. **Fetch** — The hourly cron job iterates over all active sources (Twitter handles) and fetches their recent tweets via the twitterapi.io API. Tweets are deduplicated by `platformTweetId` to prevent reprocessing.

2. **Filter** — Each tweet passes through a configurable filter pipeline with two layers:
   - **Structural rules** (fast, no LLM cost): skip retweets, skip replies, skip quotes, minimum character length, link-only detection, self-tweet detection, engagement thresholds (min likes, min views), max tweet age, and blocked keyword matching.
   - **LLM topic relevance** (optional, toggleable): sends the tweet text to the LLM with the user's topic keywords and asks for a relevance score. Tweets scoring below the threshold are rejected.

3. **Generate** — Tweets that pass filtering are sent to the LLM with the full Tyagi persona prompt to generate a reply draft. The draft includes strategy metadata: tweet category, reply style, narrative theme, and momentum score.

4. **Card creation** — A reply card is created linking the original tweet to the generated draft. Cards enter the `pending_review` state.

**State machine for reply cards:**

```
pending_review → rejected       (swipe left)
pending_review → editing        (tap to edit)
editing        → pending_review (save edits)
pending_review → posting        (swipe right)
posting        → posted         (Typefully success)
posting        → post_failed    (Typefully error)
```

**Swipe interface:**

The Replies page presents cards one at a time in a Tinder-style deck. Each card shows the original tweet (author, handle, text, engagement metrics) and the AI-generated reply draft below it. Actions available:

| Action | Gesture | Keyboard | Effect |
|--------|---------|----------|--------|
| Reject | Swipe left | Button | Card moves to `rejected` state |
| Approve & Post | Swipe right | Button | Posts via Typefully, card moves to `posted` |
| Edit | Tap / Swipe up | Button | Opens edit modal with live character count |
| Regenerate | — | Button | Generates a new reply draft for the same tweet |

### 2. Source Management

Sources are Twitter handles that the pipeline monitors for reply opportunities. They are managed in a sub-tab within the Replies page.

**Adding sources:**
- Single handle input (e.g., `naval`)
- Bulk add via comma-separated handles (e.g., `naval, pmarca, balajis, sama`)

**Account type classification:**

Each source is tagged with an account type that influences reply strategy:

| Account Type | Description |
|-------------|-------------|
| `ecosystem_partner` | Projects and people in the CAPX ecosystem |
| `domain_relevant` | Thought leaders in AI, crypto, and capital markets |
| `non_domain` | General interest accounts for broader reach |
| `investor` | VCs and angel investors |
| `builder` | Solo builders and indie hackers |

**Profile avatars** are fetched from twitterapi.io when a source is added and cached in the database. A "Refresh All" button in Settings re-fetches all avatars.

### 3. Tweet Content Generation

The app generates original tweet drafts three times daily (11 AM, 5 PM, 9 PM IST). Each batch produces 3 tweets across different styles:

| Style | Description |
|-------|-------------|
| Hot Take | Provocative, opinionated statement on a trending topic |
| Narrative | Longer-form insight aligned with Tyagi's core themes |
| Witty | Humor-driven observation or one-liner |

Generated tweets are saved to the database and sent to a Telegram bot for mobile review. From the Tweets page, users can edit drafts, post directly via Typefully, or reject them.

**Narrative themes** tracked across all content:

- `ai_capital_markets` — AI's impact on financial markets and trading
- `solo_builders` — The solo founder / indie hacker movement
- `ai_app_economy` — Building AI-native applications
- `builder_culture` — Hustle, shipping, and maker culture
- `hot_take` — Contrarian or provocative takes
- `general` — Catch-all for other topics

### 4. Cron System (v2 — Resilient)

The cron system runs entirely within the Node.js process using `node-cron`. It is designed to survive server hibernation, which is a characteristic of the Manus hosting environment.

**Scheduled jobs:**

| Job | Schedule (IST) | Schedule (UTC) | Purpose |
|-----|---------------|----------------|---------|
| `daily-tweets-11am` | 11:00 AM | 05:30 | Generate tweet batch #1 |
| `daily-tweets-5pm` | 5:00 PM | 11:30 | Generate tweet batch #2 |
| `daily-tweets-9pm` | 9:00 PM | 15:30 | Generate tweet batch #3 |
| `hourly-pipeline` | Every hour | `:00` | Run reply ingestion pipeline |

**Resilience mechanisms:**

1. **Self-ping keep-alive** — In production, the server pings its own `/api/cron/status` endpoint every 4 minutes to prevent the hosting platform from hibernating the process.

2. **Missed-job catch-up** — On every server startup, the system checks which daily jobs should have already run today (based on current UTC time) but haven't (based on DB-stored last-run timestamps). Any missed jobs are executed immediately with a 5-second delay between them.

3. **DB-backed last-run tracking** — Each job records its last execution timestamp in the `app_settings` table under keys like `cron_last_run_daily-tweets-11am`. This persists across server restarts.

4. **Error isolation** — Individual job failures are caught and logged without affecting other jobs.

**Health check endpoint:**

```
GET /api/cron/status
```

Returns JSON with all job statuses and last-run timestamps. No authentication required.

### 5. Filter Pipeline Settings

All filter thresholds are configurable through the Settings page and stored in the `app_settings` table. Changes take effect on the next pipeline run.

| Setting | Key | Default | Description |
|---------|-----|---------|-------------|
| Min Likes | `filter_min_likes` | 5 | Minimum like count to pass |
| Min Views | `filter_min_views` | 100 | Minimum view count to pass |
| Min Engagement | `filter_min_engagement` | 0.01 | Minimum engagement rate (likes+replies+retweets / views) |
| Max Tweet Age | `filter_max_tweet_age_hours` | 24 | Maximum age in hours |
| Min Length | `filter_min_length` | 30 | Minimum character count |
| Topic Keywords | `filter_topic_keywords` | (empty) | Comma-separated keywords for LLM relevance check |
| Blocked Keywords | `filter_blocked_keywords` | (empty) | Comma-separated keywords to auto-reject |
| Skip Replies | `filter_skip_replies` | true | Filter out reply tweets |
| Skip Quotes | `filter_skip_quotes` | false | Filter out quote tweets |
| LLM Filter Enabled | `filter_llm_enabled` | true | Toggle LLM-based topic relevance check |

### 6. Telegram Integration

A Telegram bot sends daily tweet batches as formatted messages to a private chat. Each message includes the tweet content, narrative theme, content tone, and topic. The bot is configured via `TELEGRAM_BOT_TOKEN` and `TELEGRAM_CHAT_ID` environment variables.

### 7. Typefully Integration

All publishing flows through the Typefully API. When a user approves a reply (swipe right) or posts a tweet draft, the app creates a Typefully draft and optionally publishes it immediately. The Typefully API key is configured in Settings.

---

## Database Schema

### Active Tables

| Table | Purpose | Key Fields |
|-------|---------|------------|
| `sources` | Twitter handles to monitor | `canonicalValue`, `accountType`, `isActive`, `avatarUrl` |
| `ingested_tweets` | Raw tweets fetched from sources | `platformTweetId`, `processingStatus`, engagement metrics |
| `filter_results` | Filter pipeline audit trail | `finalDecision`, `failedRules`, `reasonSummary` |
| `reply_drafts` | AI-generated reply drafts | `draftText`, `tweetCategory`, `replyStyle`, `narrativeTheme` |
| `reply_cards` | Swipe review cards (state machine) | `status`, `finalReplyText`, `typefullyDraftId` |
| `tweet_content` | Generated original tweets | `content`, `type`, `narrativeTheme`, `contentTone`, `status` |
| `app_settings` | Key-value configuration store | `settingKey`, `settingValue`, `category` |
| `activity_log` | System event log | `action`, `description`, `metadata` |
| `users` | User accounts (PIN auth) | `openId`, `name`, `role` |

### Legacy Tables (kept, not actively used)

`target_accounts`, `generated_replies`, `persona_documents`, `persona_inputs`, `admin_logs`, `reply_strategy_config`

---

## Authentication

The app uses a secret PIN for authentication instead of OAuth. The PIN is stored as the `AUTH_PIN` environment variable. On the frontend, a numpad-style login page collects the PIN, which is verified against the backend. A JWT session cookie is issued on success.

All tRPC procedures use `protectedProcedure` which requires a valid session. The only unauthenticated endpoints are:

- `GET /api/cron/status` — Health check
- `GET /api/cron/daily-tweets` — Webhook trigger (requires PIN as query param)
- `GET /api/cron/pipeline` — Webhook trigger (requires PIN as query param)

---

## API Endpoints

### tRPC Procedures (via `/api/trpc/*`)

**Dashboard:**
- `dashboard.stats` — Combined legacy and swipe pipeline statistics

**Swipe System:**
- `swipe.deck` — Get pending review cards for the swipe interface
- `swipe.stats` — Pipeline funnel metrics (ingested, filtered, review, posted)
- `swipe.approve` — Approve a card and post via Typefully
- `swipe.reject` — Reject a card
- `swipe.updateDraft` — Edit the reply text on a card
- `swipe.regenerate` — Generate a new reply draft for a card's tweet

**Sources:**
- `sources.list` — All sources (active and inactive)
- `sources.add` — Add a single source handle
- `sources.addBulk` — Add multiple handles at once
- `sources.toggleActive` — Activate/deactivate a source
- `sources.delete` — Remove a source

**Filter Settings:**
- `filterSettings.get` — Current filter configuration
- `filterSettings.update` — Update filter thresholds

**Avatars:**
- `avatars.refreshAll` — Re-fetch all source profile images

**Tweet Content:**
- `tweets.list` — All generated tweet drafts
- `tweets.generate` — Trigger on-demand tweet generation
- `tweets.updateStatus` — Approve/reject a tweet
- `tweets.postToTypefully` — Publish a tweet via Typefully

**Telegram:**
- `telegram.testConnection` — Verify bot token and chat ID
- `telegram.triggerDailyContent` — Manually trigger a daily batch

**Settings:**
- `settings.get` / `settings.set` / `settings.setMultiple` — Key-value settings CRUD

### REST Endpoints

| Method | Path | Auth | Purpose |
|--------|------|------|---------|
| GET | `/api/cron/status` | None | Cron health check with last-run timestamps |
| GET | `/api/cron/daily-tweets?pin=XXX` | PIN | Webhook to trigger daily tweet generation |
| POST | `/api/cron/daily-tweets` | PIN (body) | Webhook to trigger daily tweet generation |
| GET | `/api/cron/pipeline?pin=XXX` | PIN | Webhook to trigger ingestion pipeline |
| POST | `/api/cron/pipeline` | PIN (body) | Webhook to trigger ingestion pipeline |

---

## Design System (TT Aesthetic)

The UI follows a custom design language called "TT Aesthetic" — a layered dark neutral palette with precision engineering details.

### Color Palette

| Token | Hex | Usage |
|-------|-----|-------|
| Base | `#0B0F14` | Page background |
| Surface-1 | `#0F141B` | Elevated sections |
| Surface-2 | `#151A21` | Cards, panels |
| Card | `#12161C` | Card backgrounds |
| Surface-3 | `#1C222B` | Hover states, active items |
| Border | `rgba(255,255,255,0.06)` | Default borders |
| Border Hover | `rgba(255,255,255,0.12)` | Hover borders |
| Text Primary | `#E5E7EB` | Headings, primary text |
| Text Secondary | `#9CA3AF` | Body text, descriptions |
| Text Muted | `#6B7280` | Labels, timestamps |

### Accent Colors

| Color | Hex | Usage |
|-------|-----|-------|
| Blue | `#4DA3FF` | Primary actions, links |
| Green | `#00FFA3` | Success, posted status |
| Amber | `#FFB84D` | Warnings, pending status |
| Red | `#FF5A5A` | Errors, reject actions |
| Purple | `#7B61FF` | Special indicators |

### Typography

- **Font family:** Inter (loaded via Google Fonts CDN)
- **Titles:** 18px, font-weight 600, letter-spacing -0.02em
- **Body:** 14px, font-weight 400, color #9CA3AF
- **Labels:** 12px, UPPERCASE, letter-spacing 0.12em, monospace `//` prefix (e.g., `// PIPELINE`, `// STATUS`)

### Component Patterns

- **Cards:** 12px border-radius, `#12161C` background, 20px padding, hairline border
- **Shadows:** `0 10px 30px rgba(0,0,0,0.35)` with `inset 0 1px 0 rgba(255,255,255,0.03)` highlight
- **Status indicators:** Small colored dots (4px) with subtle glow, plus colored left-border stripes on cards
- **Navigation:** Pill-shaped floating top nav bar (desktop), minimal bottom tab bar (mobile)
- **Profile:** Avatar + dropdown in top-right corner for Settings and Sign Out

---

## Environment Variables

| Variable | Purpose | Required |
|----------|---------|----------|
| `AUTH_PIN` | Secret PIN for authentication | Yes |
| `JWT_SECRET` | Session cookie signing | Yes (auto-set) |
| `TWITTER_API_KEY` | twitterapi.io API key for tweet fetching | Yes |
| `TYPEFULLY_API_KEY` | Typefully API key for publishing | Yes |
| `TELEGRAM_BOT_TOKEN` | Telegram bot token for notifications | Yes |
| `TELEGRAM_CHAT_ID` | Telegram chat ID for notifications | Yes |
| `DATABASE_URL` | MySQL/TiDB connection string | Yes (auto-set) |
| `BUILT_IN_FORGE_API_URL` | Manus built-in LLM API URL | Yes (auto-set) |
| `BUILT_IN_FORGE_API_KEY` | Manus built-in LLM API key | Yes (auto-set) |

---

## Operational Notes

### Monitoring Cron Health

Visit the public health endpoint at any time to verify cron jobs are active and check when each last ran:

```
https://tyagitweet-mu6ci3ur.manus.space/api/cron/status
```

The response includes all 4 jobs with their schedules, active status, and last-run ISO timestamps.

### Minimizing Costs

The `active_model_provider` setting is set to `builtin` (free Manus model) by default. All cron jobs — both daily tweet generation and hourly pipeline — use this free model. To use Claude or OpenAI for higher quality, change the provider in Settings, but be aware this will consume API credits on every cron run.

### Manual Triggers

If a cron job fails or you need to run the pipeline on demand:

- **From the UI:** Click "Run Pipeline" on the Replies page, or "Generate Mix" on the Home page
- **Via webhook:** `GET /api/cron/pipeline?pin=YOUR_PIN` or `GET /api/cron/daily-tweets?pin=YOUR_PIN`

### Test Suite

The project includes 52 vitest tests across 7 test files covering authentication, settings, features, Twitter API connectivity, Typefully integration, Telegram bot, and the full swipe system (filter pipeline rules, router endpoints, source management, dashboard stats, filter settings, and avatar refresh).

Run tests with:

```bash
pnpm test
```

---

## Domains

| Domain | Type |
|--------|------|
| `tyagitweet-mu6ci3ur.manus.space` | Auto-generated Manus domain |
| `tyagitwitter.bluecow.xyz` | Custom domain |

---

*Last updated: March 14, 2026*
