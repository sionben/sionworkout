# RECOMP / sionworkout — Project Context

A single-file fitness app for body recomposition. The **entire app is `index.html`** (HTML + CSS + inline vanilla JS, ~1300 lines). No build step, no framework.

> This file exists so any Claude session (phone/web/desktop) can pick up instantly. Read it first.

## Stack & deploy
- **Hosting:** GitHub Pages from the `main` branch of `sionben/sionworkout` → custom domain **sionworkout.com** (see `CNAME`). **Push to `main` = live in ~1 min.**
- **Backend:** Supabase (URL + anon key hardcoded near the top of the `<script>`). Per-user data is in a table **`rc_store`** (`user_id, key, value jsonb, updated_at`; PK `(user_id,key)`; RLS = own rows; the `authenticated` role has table grants).
- **AI:** Cloudflare Worker `sionworkout-ai.sionbenharari.workers.dev` proxies the Anthropic API (key in worker env `ANTHROPIC_KEY`). Models: workouts = `claude-sonnet-4-6`, food/parsing = `claude-haiku-4-5`.
- **Worker routes:** `/` (Claude proxy, POST) · `/ical` (read-only calendar proxy, GET, host-allowlisted) · `/classes` (POST = iPhone Shortcut posts calendar text → Claude parses → Cloudflare KV `CLASSES`; GET = app reads parsed classes by `code`).

## Auth & data model
- Login = a personal **code** → email `u_<code>@recomp.app`, password `rc_<code>_v2_pw` (Supabase email auth). Admin code `sion1234`.
- **User profile lives in Supabase Auth `user_metadata.profile`** — this is the source of truth and what gates onboarding. localStorage + `rc_store` are mirrors. Do **not** make the onboarding decision depend on a separate `rc_store` query (that bug caused repeat onboarding).
- localStorage keys (mirrored to `rc_store`): `rc_profile`, `rc_d_<date>` (daily macros/log), `rc_w_<date>` (workout), `rc_p` (weigh-ins), `rc_prs`, `rc_wdates`, `rc_acts` (logged classes/activities), `rc_sched` (recurring class schedule), `rc_calev` (synced/calendar classes), `rc_ical` (iCal feed URL).

## Shipped features
- Onboarding (10 questions incl. daily **activity level**) → `computePlan` sets calorie/macro targets.
- AI coach workout generation, food-macro chat, muscle-focus body figure, journal, weigh-in trend chart.
- **Activity & Recovery engine:** one-tap class logger (Solidcore/Pilates/yoga/run/etc), recurring class schedule, **Recovery & Readiness** card (7-day load, days-active, rest-day recommendation, recent muscles), and an **AI weekly recap**.
- **Adding classes:** paste a confirmation email/list → Claude parses it (the easy front door); iCal feed sync; iPhone Shortcut sync (pulls from `/classes`).

## Open / paused thread (as of 2026-06-04)
Auto-syncing her near-daily **Solidcore** classes is the active work.
- iPhone Shortcut path is built (app pulls from Worker `/classes` on load) but **blocked**: Cloudflare **KV binding `CLASSES` is not active** (POST parses `count:1` but GET reads empty because `env.CLASSES` is undefined → need namespace created + binding named exactly `CLASSES` + redeploy), and building the iOS Shortcut is too fiddly.
- Her classes live in **Apple Calendar**; she wants it **private** (not Apple "Public Calendar").
- Leading idea to try next: **email-forwarding via Cloudflare Email Routing** (one Gmail rule → Worker parses → fully hands-off).

## How to make changes
Edit `index.html`, commit, push to `main`. Validate JS by extracting the inline `<script>` and running `new Function(it)` (must throw no syntax error); smoke-test logic in a stubbed `window`/`document`/`localStorage`/`fetch` env when touching the activity/recovery engine.
