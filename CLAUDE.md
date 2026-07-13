# CLAUDE.md — Webby World Project Brief (v2.1)

You are working on **Webby World** with a 3-person non-developer team who
build through Claude Code: **Matthew** (platform foundation / Supabase owner),
**Mo** (content, broadcast, client-facing admin), **JT** (game client, sale
side). Each person works on their own long-lived branch (see section 6 —
Git workflow) and merges to `main` through pull requests. Explain changes in
plain English, work in small verifiable steps, never leave the app broken.

**At session start: ask which person you are working with (Matthew, Mo, or
JT), check out their branch, and run the SYNC DOWN ritual in section 6.**

---

## ⚡ PRIORITY: SME'S LIVE EVENT IS IN < 1 WEEK

Everything in section "THIS WEEK" overrides everything else. The long-term
platform vision (section 8, post-event ownership) exists so we don't build throwaway work — but no
one starts post-event features until the event has run.

## 1. What this product is

Webby World is a **multi-client platform** (SaaS) that replaces webinar links
and course platforms (Kajabi / Skool / Circle) with an explorable 2D
pixel-art world. Three product surfaces:

1. **Setup side** (client/org admin configures their world) — Mo's domain
2. **Sale side** (the live event room: broadcast, pitch, application or
   checkout) — JT's domain
3. **Student side** (post-purchase delivery: courses + community inside the
   world) — Matthew's domain

Tenant #1 is SME itself, running its **Product Dynamics** event this week.
The event funnel: opt-in → magic-link login → character creator → tutorial →
Lobby (Sorting Hat, calendar, leaderboard) → Hall of Fame → Main Stage (live
video) → pitch (spots counter, desk rush) → application → VIP lounge.

## 2. THIS WEEK — the event sprint (governs all sessions)

Target: SME's event runs end-to-end on the live URL. Scope is frozen to the
tasks below. **Explicitly deferred:** student world, Stripe checkout, client
onboarding, Vite refactor (too risky pre-event), presence avatars, AI desk,
analytics.

### Day 1–2
- [ ] **Matthew:** run `webby_world_schema_v2.sql` (multi-tenant foundation —
      organizations + events tables). Verify v30 magic-link round trip works
      on the live URL.
- [ ] **Matthew:** ⚠️ **SMTP for auth emails.** The default Supabase sender
      allows ~2 magic links/hour — that is an event-killer with real
      registrants. Supabase → Authentication → SMTP Settings → connect a
      real sender (Resend is the fast option; GHL SMTP also works). Test 5
      sign-ins in a row. NOTHING SHIPS WITHOUT THIS.
- [ ] **Matthew:** upgrade Supabase to Pro before event day (realtime limits).
- [ ] **Mo:** finish all admin-panel content → Export config JSON → give to
      Matthew, who pastes it into the `events` row `config` cell (Table
      Editor). Real proof stats, real testimonials, real bonus PDFs.
- [ ] **JT:** repo access working (clone, push, see deploy). Then start B-1.

### Day 2–4 — JT's wiring (the critical path)
- [ ] **B-1 Config loading:** on boot, fetch the SME event row
      (`events` table, org slug 'sme'); where `config` has content, use it
      instead of the hardcoded consts (FAME, SCHED, BONUSES, SORTQ, QUIZ,
      landing copy, countdown from `starts_at`); fall back to built-in
      placeholders when absent. ✔ editing the DB row changes the live site.
- [ ] **B-2 Live state subscription:** subscribe (Supabase Realtime) to the
      event row. `phase` drives the room: lobby/countdown → webinar (doors
      lock, video starts) → pitch (desk opens) → ended. `spots` drives the
      counter; `announcement` shows as a toast. This REPLACES the local
      240s timer and demo button in LIVE mode (keep them in DEV mode).
      ✔ changing `phase` in Table Editor flips two open browsers within ~1s.
- [ ] **B-3 Live video:** hls.js; stage screen / fullscreen / theater play
      `events.stream_url` when set; existing MP4/slide fallback otherwise.
      ✔ Mo's Mux URL plays in all three views, on a phone too.
- [ ] **B-4 Real chat (GO/NO-GO decision Day 4):** replace simulated chat
      with Realtime inserts/subscription on `chat_messages` (org_id +
      event_id set, respect `deleted`). If not solid by end of Day 4:
      **NO-GO → hide the chat panel and silence bot chatter for the event**
      (real attendees must never see "[simulated chat]" lines). Either
      outcome is fine; a broken chat live is not.

### Day 2–4 — Mo (parallel)
- [ ] Mux account → Live Stream → OBS/StreamYard test broadcast → confirm
      the `.m3u8` playback URL → to JT (blocks B-3) and into
      `events.stream_url`.
- [ ] GHL: two inbound webhooks (new registration → show-up sequence; new
      application → strategy-call pipeline) → URLs to Matthew.
- [ ] Legal: income disclaimer near $ claims, privacy policy link, CASL
      consent line on the opt-in (Canadian audience).
- [ ] Promote the event / drive opt-ins to the live URL.

### Day 3–4 — Matthew (parallel)
- [ ] Edge functions: on new `registrants` row and new `applications` row,
      POST {email, name, answers} to Mo's GHL webhooks. Secrets in function
      env vars only.
- [ ] Fallback plan if edge functions slip: manual CSV export of
      registrants/applications into GHL after the event. Acceptable.

### Day 5 — FULL DRESS REHEARSAL (all three + Claude sessions on standby)
Mo streams for real via Mux; everyone joins on the live URL (one laptop, one
phone minimum each); run the complete flow: opt-in → magic link → character →
lobby → phase flip to webinar (Table Editor) → watch stream → phase flip to
pitch → apply → VIP. Log every bug.

### Day 6 — fix list from rehearsal, second shorter rehearsal, then
**CODE FREEZE by end of day.** Only show-stopper fixes after freeze.

### Day 7 — EVENT. Show roles: Mo on stage · JT drives phase/spots/
announcements from Table Editor + monitors stream · Matthew monitors
signups/chat/support and handles fires.

### Scale note
Current stack (Supabase Pro + Realtime chat + Mux) is comfortable to ~500
concurrent. If expected concurrency is higher, tell Matthew's Claude session
immediately — chat throttling and sharding decisions change.

## 3. Current code state (v30)

Everything is ONE file, `index.html` (~200KB): HTML + CSS + two script
blocks (game, then admin IIFE). **No refactor this week.**

Working: full world/rooms/quests/chests/levels (single-player) · ~18
labeled bots (wander, visit points of interest, simulated chat, pitch rush) ·
Supabase magic-link auth · registrant auto-row · player persistence (name,
avatar, archetype, level, quiz_answers, applied, rsvp) · returning players
resume · `is_admin` reveals in-app admin screen (🎛 top-right) · admin panel
(13 tabs → localStorage `wwAdmin_v2` + JSON export/import) · MP4 demo video
with slide fallback.

LIVE vs DEV mode: Supabase URL/key consts near top (search `🔌 SUPABASE`).
Empty = DEV (fake email, 🔑 admin bypass button, nothing saved). **Never
remove the DEV fallback.**

Code map (script 1): DEV banner → Supabase block (`LIVE`, `sb`, `sbUser`,
`sbProfile`, `savePlayer`, `loadProfileAndResume`) → data consts → creator →
`startGame()` → world data → bot system → panels → quizzes → synthesized
audio → chat sim (`postChat`/`chatTick`) → `awardLevel` → webinar phases
(`startPitch`) → video (`VIDEO_URL`) → render loop. Script 2: admin IIFE
(`DEFAULTS`, tab renderers `R.*`, `window.wwOpenAdmin()`).

## 4. Infrastructure

| Thing | Value |
|---|---|
| Repo | GitHub `webby-world`, branch `main` (file must stay `index.html`) |
| Hosting | Vercel auto-deploy → https://webby-world.vercel.app |
| Database | Supabase; Site URL = the Vercel URL |
| Team | shared Supabase/GitHub login; all three have is_admin=true |

## 5. Database schema (v2 — multi-tenant)

- `organizations` — tenant per client. SME = fixed id
  `00000000-0000-0000-0000-000000000001`, slug `sme`, `theme jsonb`.
- `events` — per-org events. Content (`config jsonb` = admin panel export)
  AND live state in one row: `phase` (lobby|webinar|pitch|ended), `spots`,
  `stream_url`, `announcement`, plus `type` (webby|challenge), `starts_at`,
  `offer_mode` (application|checkout|both). Realtime-enabled.
- `registrants` — per user; now has `org_id`. avatar jsonb, archetype,
  level, progress, quiz_answers, applied, rsvp_stage_b, is_admin, last_seen.
  Auto-created on signup by trigger.
- `chat_messages` — org_id, event_id, name, body ≤300, deleted (soft).
  Realtime-enabled.
- `applications` — org_id, event_id, user_id, answers jsonb.

RLS is ON everywhere + auto-RLS for new tables (a new table is LOCKED until
policies exist — add policies, never disable RLS). `public.is_admin()` is
the helper; this week admins are global, per-org roles come later.
Single-tenant reads this week: filter by org slug `sme` / the one event row.

## 6. Git workflow — branch per person (ALL sessions)

Branches: `main` (deployed truth → the live Vercel URL) plus one long-lived
branch per person — the Mo branch, the JT branch, the Matthew branch
(confirm exact branch names with `git branch -r` and pick the one matching
the person you're working with; never work directly on `main`).

**Vercel deploys every branch.** Pushing to a personal branch creates a
preview deployment with its own URL (visible in the GitHub commit checks or
the Vercel dashboard). Test on YOUR branch preview; only `main` is the real
site that registrants use.

**SYNC DOWN — start of every session (prevents merge hell):**
```
git checkout <my-branch>
git pull origin main        # bring in everyone's merged work FIRST
git push                    # update my branch + its preview
```
If this pull produces conflicts, resolve them NOW, at session start, while
they're small — with one big index.html, a week of drift makes branches
unmergeable. Never skip this step.

**MERGE UP — when a task is done and tested on your branch preview:**
1. Push the branch, open a **Pull Request** into `main` on GitHub.
2. PR description: what changed, how it was tested, anything the other two
   must know.
3. Merge it yourself (no approval gate this week — speed matters), then tell
   the team chat so the other two SYNC DOWN before their next edit.
4. Confirm the live URL still works after Vercel redeploys `main`.

Rules of thumb: merge up at least **once a day** — small frequent merges
beat one giant scary one. If two people must touch the same area of
index.html the same day, say so in team chat and sequence it. If a conflict
looks bad, do not force it — resolve together with a Claude session, taking
`main` as the base and re-applying your change on top.

## 7. Working agreements (ALL sessions)

1. Follow section 6 for all git operations. `main` must always be a
   working, event-ready build.
2. Test on your branch preview URL after every push (normal + incognito +
   phone); test the live URL after every merge to `main`.
3. Never commit `service_role` key, DB password, or SMTP creds (anon key in
   index.html is fine — it's public by design).
4. Never weaken RLS to "make it work" — write the correct policy.
5. Don't break DEV mode or the phone layout.
6. Bots stay, but: labeled bot chatter must never be visible to real
   attendees in LIVE mode once real chat ships (or chat is hidden).
7. CODE FREEZE end of Day 6 — `main` freezes; after that, show-stopper
   fixes only, merged with all three aware.
8. Finish every session by stating what changed, how tested, what the
   other two must know — and put it in the PR description.

## 8. Post-event ownership (starts AFTER the event)

- **Mo — Setup surface:** admin panel becomes client onboarding: create org
  → brand (theme jsonb → the scoped CSS variables) → create event → content
  → offer config → invite team. Admin "Save" writes to the DB, not
  localStorage. Two admin levels: org admins vs platform superadmins.
- **JT — Sale surface:** `offer_mode` checkout path via Stripe Checkout
  (hosted page from the in-world desk; webhook on payment → enrollment,
  unlocks student side). Director console UI wired to the events row.
  Presence avatars (real players visible). Vite refactor lands here, first.
- **Matthew — Student surface:** post-purchase world inside the game:
  course halls (lessons as walk-up stations, video via Mux, resources via
  the chest pattern), community rooms (persistent chat channels), progress
  via levels/badges. New tables: courses, modules, lessons, enrollments,
  lesson_progress — all org-scoped. Plus platform foundation: memberships,
  org routing (`/slug`), Stripe account, SMTP hardening.

Milestones: **M1** SME event (this week) → **M2** first external client,
done-for-you → **M3** self-serve onboarding.

## 9. Gotchas
- Default auth email sender ≈ 2 magic links/hour — SMTP is mandatory
  pre-event (Day 1–2 task).
- `index.html` name is what makes Vercel serve the bare URL.
- Supabase Site URL must equal the Vercel URL or magic links break.
- Auto-RLS: new tables start locked; add policies.
- TextEdit rich-text mode corrupts the file — plain text only.
- events/organizations single-tenant this week: always scope queries
  (org slug 'sme'); never assume one row exists globally.
- Admin screen suppresses game keys while open — preserve if touching input.
- Magic-link emails may land in spam from new senders — tell registrants
  in the confirmation copy.
- Branch preview URLs differ from the Site URL, so magic-link redirects go
  to the LIVE site, not your preview. To test auth flows on a preview,
  paste the link's URL and swap the domain, or test auth on main only.
- JT's B-2 replaces the demo timer/button in LIVE mode — after that lands,
  "nothing happens" in lobby is correct until someone flips `phase`.
