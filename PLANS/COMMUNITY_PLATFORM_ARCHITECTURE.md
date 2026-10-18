# Community Platform on Buzz — Architecture & Gap Analysis

**Status:** Draft v1 · **Date:** 2026-10-14
**Companion to:** *Building My Own Community Platform on Buzz* (Oct 14, 2026)
**Evidence base:** a static code read of `cdcore09/buzz` at `1d9f2c81`
(12 commits behind `block/buzz`) and `cdcore09/buzz-app` at `f208fa61`
(33 commits behind `block/buzz-app`). Every claim marked SHIPPED, PARTIAL or
ABSENT was checked in code, not in docs. Several Buzz docs lag the code;
see §2. Nothing here was exercised against a running relay.

Status legend:
- **SHIPPED**: wired and reachable.
- **PARTIAL**: exists with material gaps.
- **DOC-ONLY**: specified, not implemented.
- **ABSENT**: not in the code.

---

## 1. Executive summary

**Verdict: Buzz is a sound substrate for this community, and in several areas it is further along than the planning doc assumes.** The relay already has:

- a real community model (owner, admin and member roles; open and private channels);
- invite links with expiry and use caps;
- join-policy acceptance (terms, privacy, age);
- a full moderation pipeline: report, ban, timeout, kick, delete, tombstone, audit;
- forum channels, voice notes and live huddles with agent participation;
- a mature agent harness that already supports Hermes.

The work is the **community layer**: membership tiers, a join funnel, notifications that reach people outside the app, course structure, and a few targeted agent-harness changes.

The findings that should change the plan:

1. **You may not need to fork the clients for phase one.** Block publishes signed desktop builds that connect to any relay (`README.md:117-131`). If phase-one features are built from relay-side primitives, agents and a sidecar service, your members use the official app and you carry no client fork, signing or distribution burden. Build your own client only when a feature needs new UI (native course pages).

2. **Moderation is mostly built, not "thin".**
   - The relay implements NIP-56 private reports, ban, timeout, kick, delete with tombstones, guard rails, and audit rows.
   - The desktop app exposes report, timeout and ban actions on messages.
   - The real gaps are a **community-owner queue screen** (the existing queue UI is operator-only) and **no moderation on mobile**.
   - `buzz moderation reports|resolve` in the CLI covers the queue today.

3. **Agents cannot answer a member's DM.** In a DM the ACP harness responds only to the agent's owner or sibling agents, even with `--respond-to anyone` (`crates/buzz-acp/src/lib.rs:577-581`). An onboarding or Q&A agent that members can DM needs a small harness change. Channel @mentions work today.

4. **Voice notes have no transcript for anyone, humans included.**
   - The PDF says humans get transcripts. They don't: buzz-app's `VoiceNoteCard.tsx:278` notes that "user-recorded voice notes do not include a transcript."
   - On-device speech-to-text exists only in desktop huddles (`desktop/src-tauri/src/huddle/stt.rs`).
   - Issue #7339 is open upstream. The narrowest fix is in `buzz-acp`.

5. **"Notifications" has three halves with different maturity:**

   | Channel | Status |
   |---|---|
   | In-app desktop alerts | SHIPPED |
   | Mobile push | Experimental, iOS/APNs only, off by default; tied to one Apple app per gateway |
   | Email | ABSENT |

   For a community that has to pull people back, **email is the highest-leverage gap**. The relay already contains a durable mention-notification webhook outbox (`crates/buzz-relay/src/operator_listener.rs`) that a sidecar can turn into email.

6. **There is no public web client.** buzz-app's browser mode is dev-only, and there's no browser key custody (no NIP-07/NIP-46). The relay's `web/` app has only an invite page and a git repo browser. A Medium reader's path today is: landing page → invite page → download a desktop app → join. Acceptable for a technical founding cohort; the biggest long-term funnel risk.

7. **Courses have zero existing concepts** (no lessons, ordering or progress). They also hit two media limits:
   - Server-side video is capped at **600 s** (hard-coded, `crates/buzz-media/src/validation.rs:369`).
   - Video must be H.264/AAC, transcoded client-side by the uploader's system `ffmpeg`, with no HLS.

   A "course v0" can be assembled from forum channels, canvases and deep links with no code. Native courses come after.

8. **Payments and entitlements are fully absent**, and that's fine. Channel membership is the access gate, and a sidecar can drive it through existing signed admin commands (NIP-29 `9000`/`9001`, NIP-43 `9030`/`9031`). You can model tiers from day one without patching the relay.

9. **Upstream moves fast.** In the last 30 days `block/buzz` landed ~231 commits and `block/buzz-app` ~702. The fork strategy (§10) matters as much as any feature: keep a thin patch queue, put community logic in a sidecar, and upstream every generic fix.

---

## 2. Corrections to the planning doc

| Planning-doc claim | What the code says | Evidence |
|---|---|---|
| Voice notes have "transcripts for humans" | No transcripts anywhere. Voice notes are MP4s with a black H.264 track (the relay rejects audio uploads). | `buzz-app/src/bundled/voice-notes/VoiceNoteCard.tsx:278`; `crates/buzz-media/src/validation.rs:196-205`; `desktop/src-tauri/src/commands/media_transcode.rs:279-331` |
| Moderation: "removing, muting, reporting are thin" | Relay pipeline SHIPPED: reports `1984`, ban/unban/timeout/untimeout/resolve `9040-9044`, kick, delete, tombstones, audit. Desktop has the actions. Gaps: owner-facing queue UI, mobile, moderator tier. | `crates/buzz-relay/src/handlers/report.rs:44`, `moderation_commands.rs:92`, `moderation_authz.rs:83`; `desktop/src/features/moderation/` |
| Push "still an aspiration" | iOS push is built but experimental (`BUZZ_PUSH_ENABLED=false` by default); Android/FCM absent; email absent | `docs/buzz-push.md:3`; `crates/buzz-push-gateway`; `handlers/push_lease.rs:206-210` |
| "Channel membership is the only gate" | True for content. There is also a community-level gate (`relay_members`, NIP-43) plus invites, allowlist and join policy, all deployment-wide. | `handlers/auth.rs:318-650`; `config.rs:221,227,380` |
| "Browser surface is mostly the invite page" | Accurate. The invite page can claim in-browser only with a NIP-07 extension; otherwise it deep-links `buzz://join`. | `web/src/features/invite/ui/InvitePage.tsx` |
| (Not in PDF) ARCHITECTURE.md "no rate limiting" | Stale: a Redis rate limiter is wired into admission, plus per-key limiters for media and invite claims | `crates/buzz-relay/src/state.rs:1290-1303,1487` |

---

## 3. The substrate in one picture

```
            ┌──────────────────────── Clients ────────────────────────┐
            │ Official Buzz desktop (Tauri)   Buzz mobile (Flutter)   │
            │ buzz-app (plugin shell, successor; desktop-first)       │
            │ relay-served web/: invite page + git browser only       │
            └───────────────┬─────────────────────────────────────────┘
                            │  WebSocket (NIP-01/42) · HTTP (NIP-98)
┌───────────────────────────▼──────────────────────────────────────────┐
│ buzz-relay (Rust/axum)                                               │
│  host→community binding (fail-closed) · auth gates · event ingest    │
│  NIP-29 groups · NIP-43 membership · moderation commands · invites   │
│  Blossom media (S3) · huddle Opus relay · workflows · push matcher   │
│  operator-listener outbox · /events /query /count bridge · git HTTP  │
└──────┬──────────────┬───────────────┬───────────────┬────────────────┘
       │ Postgres 17   │ Redis          │ S3/MinIO       │ push gateway (APNs)
       │ events (part.)│ pub/sub,       │ media blobs    │ (separate service)
       │ audit chain   │ presence,      │                │
       │ FTS search    │ rate limits    │                │
       └───────────────┴────────────────┴────────────────┘
                            ▲
               buzz-acp harness (one per agent identity)
               ↕ ACP stdio: goose · Claude Code · Codex · Hermes (hermes-acp) · …
```

Key properties to build on:

- **Everything is a signed event**, and new features are new kind integers (`crates/buzz-core/src/kind.rs`, 127 kinds).
- **Access is enforced in the relay** at three seams:
  - connect: `handlers/auth.rs`
  - write: `handlers/ingest.rs:501` (scope) and `:2538` (ban/timeout)
  - read: `handlers/req.rs:109-194` (private-channel filtering)
- **The relay prefers events over new HTTP endpoints** (CLAUDE.md, "Prefer Nostr events"). Follow this for course data. Keep HTTP for things that are inherently HTTP: Stripe webhooks and email links, which live in the sidecar anyway.

---

## 4. Capability inventory

### 4.1 Community, identity, membership

| Capability | Status | Notes / evidence |
|---|---|---|
| Community = tenant, host-bound | SHIPPED | `crates/buzz-relay/src/tenant.rs:71` fails closed; every table keyed by `community_id` |
| Community roles: owner / admin / member | SHIPPED | `relay_members`; kinds `9030-9032` in `handlers/relay_admin.rs` |
| Channel roles: owner / admin / member / guest / bot | SHIPPED | `channel_members.role`; policy in `handlers/channel_authz.rs` |
| Open vs private channels; stream / forum / dm / workflow types | SHIPPED | `schema/tables/public/channels.sql` (`max_members`, `topic_required`, `ttl_seconds`) |
| Invite links (stateless HMAC v1; DB-backed v2 with `max_uses` / `expires_at`) | SHIPPED | `crates/buzz-relay/src/api/invites.rs`; owner/admin mint, capped at member role; claims rate-limited 10/min |
| Join policy: terms, privacy, age attestation, acceptance receipts | SHIPPED (opt-in) | `/api/join-policy*`, `join_policy_acceptances`; deployment-global config |
| Open signup vs allowlist vs membership-required | SHIPPED, deployment-global | `BUZZ_REQUIRE_RELAY_MEMBERSHIP` (default open), `BUZZ_PUBKEY_ALLOWLIST` |
| Channel self-join (open channels) | SHIPPED | kind `9021`; private channels reject; no request-to-join queue |
| Community guest role | PARTIAL | `guest` exists only at channel level |
| NIP-29 `9009` create-invite | Stub | `side_effects.rs:344` "deferred" |
| NIP-29 `39003` roles event | Defined, never emitted | `side_effects.rs:1189` |
| NIP-05 handles per community | SHIPPED | `/.well-known/nostr.json`, `users.nip05_handle` |
| Tiers / entitlements / plan metadata | ABSENT | Only agent rate-limit tiers exist (`buzz-auth/src/rate_limit.rs:84-112`) |
| Server-side member onboarding | ABSENT | Desktop `onboarding/welcomeKickoff.ts` is first-run for a user's *own* agents |
| Community creation | SHIPPED (operator API) | `POST /operator/communities` (NIP-98, `RELAY_OPERATOR_PUBKEYS`); not in `buzz-admin` |
| Postgres RLS backstop | DOC-ONLY | Isolation relies on query discipline; no `CREATE POLICY` anywhere |

### 4.2 Conversation and content

| Capability | Status | Notes |
|---|---|---|
| Stream chat, threads, edits, deletes, reactions, typing | SHIPPED | |
| DMs (up to 9 participants) | SHIPPED | |
| Forum channels: post `45001`, comment `45003` | SHIPPED (desktop + mobile) | Flat threads. buzz-app reads them but has no forum UI. |
| Forum votes `45002` | PARTIAL | Relay validates; no client UI |
| Pins `40004` / scheduled messages `40006` | Relay-accepted, no UI / no scheduler | |
| Canvas (one markdown doc per channel, `40100`) | SHIPPED (desktop, CLI) | History/restore; no co-editing; no mobile |
| Custom emoji (NIP-30) | SHIPPED | |
| Personal reminders (`30300`, encrypted to self) | SHIPPED | |
| Search (Postgres FTS, permission-aware) | SHIPPED | `crates/buzz-search/src/query.rs` |
| Polls, kudos, confetti, coffee roulette | DOC-ONLY | `VISION.md` "Culture Features" |
| Courses / lessons / progress / leaderboard / points | ABSENT | |
| Events calendar / RSVP (NIP-52) | ABSENT | |

### 4.3 Media, voice, huddles

| Capability | Status | Notes |
|---|---|---|
| Blossom upload/download on S3 | SHIPPED | Auth required for GET as well; no list / delete / mirror |
| Limits | SHIPPED | Image 50 MB (jpeg/png/gif/webp only), GIF 10 MB, video 500 MB, files 100 MB (`buzz-media/src/config.rs:36-79`) |
| Video | SHIPPED, constrained | H.264 + AAC, one track each, ≤600 s (hard-coded), ≤4K, faststart. No server transcoding; uploader's `ffmpeg` transcodes. Range requests yes; HLS no. |
| Audio uploads | Rejected by design | "until Buzz has an explicit sanitizer" (`validation.rs:196-205`) |
| Timecoded video review comments | SHIPPED (desktop) | Thread replies become scrubber markers (`desktop/src/shared/ui/VideoPlayer.tsx`) |
| Media library / gallery per channel | ABSENT | |
| Voice notes: record, playback, waveform | SHIPPED (desktop, mobile, buzz-app) | Encoded as MP4. Recognized by the `voice-note-*` filename. Waveform computed at playback. Max 5 min. |
| Voice-note transcription | ABSENT | |
| Huddles (Opus over WebSocket, ≤25 peers, lifecycle events) | SHIPPED (desktop, mobile MVP) | `crates/buzz-relay/src/audio/`, kinds `48100-48106` |
| Huddle live STT + agent voice (TTS, barge-in) | SHIPPED (desktop only) | sherpa-onnx Parakeet; Pocket TTS |
| Huddle recording | ABSENT | Reserved kinds, no producer |

### 4.4 Notifications and engagement

| Capability | Status | Notes |
|---|---|---|
| Desktop OS notifications (mentions, DMs, thread replies, needs-action) | SHIPPED | Only while the app runs; settings for sound and badge |
| Unread counts, mentions index, read-state sync | SHIPPED | NIP-RS `30078`, `event_mentions` |
| Home feed | SHIPPED (client-assembled) | Server `query_activity` exists but unused by desktop |
| iOS push via gateway (APNs) | PARTIAL, experimental | Kinds 9 / 40002 / 45001 / 45003 only; DMs, mentions, thread participation; one Apple app per gateway |
| Android push | ABSENT | Relay rejects non-configured transports |
| Desktop-active push suppression | DOC-ONLY | `docs/mobile-push-suppression.md:129-131` |
| Operator-listener mention outbox | SHIPPED | Durable, NIP-98-signed POST of `{pubkey, community_host, event_id, event_kind}` for registered pubkeys (`operator_listener.rs`; `POST/DELETE /operator/listener/pubkeys`) |
| Email (transactional or digest) | ABSENT | No SMTP or provider code anywhere |

### 4.5 Agents and automation

| Capability | Status | Notes |
|---|---|---|
| Agents as members: own keypair, profile, owner attestation (NIP-OA), bot role, "managed by" label | SHIPPED | |
| Owner-sent presence for agents (`20003`) | SHIPPED | Latest upstream commit |
| ACP harness: goose, Claude Code, Codex, buzz-agent, plus presets including **Hermes** (`hermes-acp`) | SHIPPED | `desktop/src-tauri/src/managed_agents/discovery/presets.rs:199-209` |
| Subscription modes (mentions / all / config rules) | SHIPPED | Expression filters per channel (`buzz-acp/src/filter.rs`) |
| `--respond-to` owner-only / allowlist / anyone / nobody | SHIPPED for channels | **DMs are always owner/sibling-only** (`lib.rs:577-581`) |
| Heartbeat prompts, reminders, turn steering, lazy worker pools | SHIPPED | |
| Personas (name, avatar, prompt, model, runtime) as kind `30175`; teams `30176` | SHIPPED (desktop) | `buzz-persona` pack spec is mostly validated-but-not-executed |
| Agent memory (NIP-AE `30174`, core memory injected) | SHIPPED | |
| Agent sees attachments | PARTIAL | Raw `imeta` tag JSON only. Images can be fetched with `view_image`. No audio. |
| Workflows: triggers message / reaction / diff / schedule / webhook | SHIPPED | Channel-scoped |
| Workflow actions `send_message`, `call_webhook`, `delay` | SHIPPED | |
| Workflow `add_reaction` | Likely broken | Target route not found |
| Workflow `send_dm`, `set_channel_topic` | Stub (WF-07) | |
| Workflow `request_approval` | Fails the run (WF-08) | |
| Member-joined trigger | ABSENT | Relay emits a `member_joined` system message `40099` and NIP-43 `8000`, but no trigger matches them |
| Outbound integrations (email, newsletter, Slack, Zapier) | ABSENT | Only generic `call_webhook` |

### 4.6 Clients

| Client | State | Fit |
|---|---|---|
| **Official Buzz desktop** (`buzz/desktop`, Tauri) | Most complete: forum, huddles, moderation actions, admin console, onboarding, Pulse | **Phase-one member client.** Signed macOS builds, AppImage/deb, unsigned Windows. Connects to any relay. |
| **buzz-app** ("Buzz Foundation") | Upstream's successor; plugin system (pages, panels, settings, composer extensions) | Long-term home for native course UI. Gaps: forum UI, huddles; not release-ready (`docs/status.md:3-5`). Plugins can't publish new kinds without host changes (`src-tauri/src/relay.rs:259-318`). |
| **Mobile** (Flutter) | Channels, forum, voice notes, huddles MVP, invites, pairing, iOS push | No moderation. Your own iOS build means your own App ID, APNs and push gateway. |
| **web/** (relay-served) | Invite page (NIP-07 claim or `buzz://join`), git repo browser | Becomes your public front door; no chat UI |
| **admin-web** | Operator console: reports, feedback | Operator-level, not community-owner-level |

---

## 5. Gap analysis against Skool and Circle

| Skool/Circle feature | Buzz today | Gap size | Bucket / phase |
|---|---|---|---|
| Community feed / discussion board | Forum channels + Home feed | Small (pins, votes UI) | Place / 1 |
| Real-time chat, DMs | Shipped | None | — |
| Live calls / events | Huddles (no recording, no calendar) | Medium | Place / later |
| Courses: modules, lessons, ordering, progress, drip | None | **Large** | Place / 1 (v0) → 1.5 (native) |
| Video hosting | Blossom, 10-min cap, no HLS | Medium | Place / 1 |
| Member directory and profiles | Profiles, member lists | Small | — |
| Notifications: push + email + digest | In-app only; iOS push experimental | **Large (email)** | Place / 1 |
| Moderation | Relay pipeline shipped | Small (owner queue UI) | Place / 1 |
| Onboarding, welcome flow | None server-side | Medium | Place / 1 |
| Gamification (points, levels, leaderboard) | None | Medium | Place / later (optional; arguably off-brand) |
| Membership tiers, paywalls | None | Medium (sidecar) | Business / 2 |
| Payments, subscriptions | None | Medium (Stripe) | Business / 2 |
| Web access without installing anything | None | **Large** | Place / 2 (strategic) |
| Branded custom domain | Host-bound community; NIP-05 | Small (DNS) | — |
| Multi-community, analytics | Multi-tenant model + metrics; RLS doc-only | Medium | Mine / later |
| **AI agents as first-class members** | Shipped, best-in-class | None (Buzz's advantage) | Differentiator |

---

## 6. Target architecture

### 6.1 Principles

1. **Relay-first, client-agnostic.** Prefer features built from events, agents and server-side services. They work in the official clients you don't have to ship. Build client UI only when a feature can't be expressed with existing surfaces.
2. **Thin fork, thick sidecar.**
   - Community-specific logic (tiers, funnel, email, payments, analytics) lives in a new service, the **Steward**.
   - The Steward talks to the relay only through public seams: the NIP-98 HTTP bridge, the WebSocket, signed admin commands, the invite API and operator-listener webhooks.
   - Relay patches are small, generic and upstreamed.
3. **Events over endpoints.** New *community* concepts (course manifests, progress) are new kinds in `kind.rs`, per Buzz's own rule. Private business data (emails, Stripe IDs) never becomes an event. It stays in the Steward database.
4. **Tier from day one.** Every member has a tier; at launch every tier is free. The Steward is the source of truth, and channel membership is the enforcement mechanism.
5. **Inherit Buzz's review-proven rules.** Every Steward side effect is driven from a durable journal with idempotent retries (Rules 1 and 5). Reconciliation loops have backoff and a terminal state (Rule 4).

### 6.2 Components

```
 Medium post ──► Founding-member link
                      │
                      ▼
 ┌──────────────── Steward (new service, your code) ────────────────┐
 │ Join funnel (landing page, email + newsletter opt-in)            │
 │ Member registry: pubkey ↔ email ↔ tier ↔ cohort                  │
 │ Entitlement reconciler: tier → desired channel set → 9000/9001   │
 │ Notification worker: listener webhook + digests → email          │
 │ (Phase 2) Stripe Checkout / Portal / webhooks → tier changes     │
 │ (Later) Analytics exporter                                       │
 │ Own keypair: community admin + admin of gated channels           │
 └──────┬─────────────────────┬──────────────────────┬──────────────┘
        │ NIP-98 HTTP          │ WS subscribe          │ Email provider,
        │ (invites, /events,   │ (8000 member-added,   │ newsletter API,
        │  /query, operator    │  40099 joins, open-   │ Stripe
        │  listener register)  │  channel activity)    │
        ▼                     ▼                      │
 ┌──────────── buzz-relay (lightly patched fork) ─────────────────┐
 └──────────────────────────────▲─────────────────────────────────┘
                                │
          buzz-acp × N (patched: DM policy, voice transcripts)
            Miles (onboarding) · Ops · Q&A · Moderation-assist · Digest
            runtime: Hermes via hermes-acp
```

**Steward stack.** Any language works because it only speaks HTTP and WebSocket. Two good options:

- **Rust**, reusing `buzz-ws-client` and `buzz-sdk` (typed event builders, NIP-42 client).
- **TypeScript**, reusing `buzz/web/src/shared/lib/nip98.ts` and `nostr-client.ts`.

Rust keeps one toolchain and lets you vendor the typed builders. The Steward uses its own small Postgres database, which can be a second database on the relay's Postgres server.

**Steward identity.** The Steward holds a keypair that is:

- a community **admin** (`buzz-admin add-member` or a kind `9030` from the owner);
- an **admin of each gated channel**.

Community admin authority can't promote anyone to admin (owner-only, `relay_admin.rs:315-330`), which is the right limit for an automated actor.

---

## 7. Phase-one designs

### 7.0 Founding-member join and the tier model (prerequisite for everything)

**Goal:** one action in a Medium post adds the reader to the community as a founding member and subscribes them to the newsletter.

**Flow:**

1. **Link → Steward landing page** (`join.<domain>`). The reader enters an email and confirms terms. Before writing anything, the Steward records a durable *pending signup* row `{email, cohort: founding, created_at}`; this is the single atomic persist (Rule 5).
2. **Newsletter subscribe** runs as a retried job from that row (provider API: Buttondown, Kit, Beehiiv, etc.). A failure never blocks the invite and is retried (Rule 1).
3. **Invite.** The Steward mints a **single-use v2 invite** (`POST /api/invites`, `max_uses=1`, expiry ~14 days) and emails it. Emailing the link instead of showing it on-page:
   - verifies the address;
   - binds the invite to that person;
   - lets the link work **before the community is live**, because the Steward can queue invites and send them at launch. This resolves the PDF's "where does the link land" question.
4. **Invite page** (relay-served `web/`, `/invite/<code>`) shows the join policy and records receipt. Then either:
   - the reader joins in-browser with NIP-07 (rare for this audience), or
   - **"Accept in Buzz"** deep-links `buzz://join?...` after they install the official desktop app.
5. **Binding pubkey ↔ email.** The claim inserts `relay_members` and publishes NIP-43 `8000`. The Steward watches `8000` over WebSocket, but the event carries the pubkey, not the email.
   - The invite code is the natural join key because the Steward minted it for that email.
   - **However, the relay doesn't record who claimed a v2 code.** `relay_invites` (`schema/tables/public/relay_invites.sql`) stores only `token_hash`, `use_count`, `max_uses`, `expires_at` and `created_by`.
   - **Required patch (small, generic, upstreamable):** record redemptions as `(community_id, invite_id, claimant_pubkey, claimed_at)` in the same transaction as the claim, and let the invite's creator read them. The Steward then maps invite → email → pubkey exactly.
   - Without the patch, the fallbacks are fragile (timestamp correlation of `use_count` with `8000`) or add friction (a second "link your account" step on the Steward site, signed with NIP-98).
6. **Tier assignment.** The Steward writes `member{pubkey, email, tier: 'founding', cohort, joined_at}`. The reconciler grants the tier's channels.

**Tier data model** (Steward database):

```
tiers(id, name, price_ref NULL, grants JSONB)   -- grants: channel ids, flags
members(pubkey PK, email, tier_id, cohort, joined_at, newsletter_state,
        email_prefs JSONB, stripe_customer_id NULL)
entitlement_journal(id, pubkey, from_tier, to_tier, reason, created_at,
                    applied_at NULL, attempts, last_error)
```

**Reconciler.** For each member, it computes the desired channel set from the tier's grants. It reads actual membership from relay-signed `39002` events and emits NIP-29 `9000` put-user / `9001` remove-user, signed by the Steward. It is idempotent, journaled, retried with backoff, and dead-lettered after N attempts (Rules 1 and 4).

**Tiers at launch:** `founding` and `member`, both free. Phase 2 adds paid tiers by setting `price_ref` and new grants, with no new concepts.

**Showing the tier to members (optional).** Do **not** add a column to `relay_members` in phase one; that's a relay fork patch with no enforcement value. If a badge is wanted later, the Steward can publish a signed attestation event (NIP-58-style) that clients render.

### 7.1 Notifications

Layered by leverage:

| Layer | What | Work |
|---|---|---|
| **Email: mention / DM alerts** | Register opted-in members' pubkeys with the operator listener (`POST /operator/listener/pubkeys`; configure `BUZZ_OPERATOR_LISTENERS=<steward_pubkey>:<url>`). On each NIP-98-signed webhook, look up the member's email and preferences, then send "X mentioned you in #channel" with a `buzz://message?...` deep link. | Steward worker; email provider (Postmark, Resend, SES) |
| **Email: weekly digest** | Steward (or a *Digest agent*) reads open-channel and forum activity via `/query`, then sends a per-member digest of new forum posts, unanswered questions and upcoming huddles. It doubles as the community's heartbeat between the monthly newsletters. | Steward job + optional agent |
| **In-app desktop alerts** | Already shipped | None |
| **iOS push** | Turn on only when you ship your own iOS build: own App ID, APNs key and `buzz-push-gateway` deployment. Until then, the official app's push needs Block's gateway, which is not under your control. | Defer |
| **Android push** | Absent upstream | Defer / upstream |

**Privacy rules for email:**

- Never put private-channel or DM content in email. Use "You have a new DM from X" plus a deep link.
- Open-channel snippets are fine.
- **The operator listener's exact behavior** (`crates/buzz-db/src/store/operator_listener.rs`):
  - It enqueues only kinds `9`, `40002`, `45001` and `45003`, and only for registered pubkeys that appear in a `p` tag (`:16`, `:68-85`). Channel mentions and forum mentions are covered.
  - DM messages are covered **only if the client p-tags the recipients**. Verify that against a live DM before promising DM emails, or detect DMs another way (for example a Steward-side unread poll).
  - Registrations expire after **30 days** (`PUBKEY_RETENTION`), so the Steward must re-register opted-in members on a schedule. That's a renewal path; Rule 2 applies.
  - Undelivered notifications are dropped after **15 minutes** (`OUTBOX_RETENTION`) or 9 attempts. A Steward outage longer than that silently loses alerts. That's acceptable for courtesy email, but monitor it.

**Respect the "zero by default" product contract** (`VISION.md`): email defaults to *mentions + DMs + weekly digest*, with per-member preferences on the landing site. That's a deliberate, stated tension with Buzz's zero-notification default for in-app streams. It's justified because email is opt-in at signup.

### 7.2 Courses and structured video

**v0: no code, ships in days.** Built entirely from shipped primitives in the official client:

- **Course = one private forum channel** (`#course-trustworthy-ai`). Channel membership is enrollment, so the tier reconciler controls access later.
- **Lesson = one forum post** (`45001`) containing the video, plus a markdown body. Discussion happens as the post's comments. Video replies get the timecoded-review UI.
- **Syllabus = the channel canvas** (`40100`): an ordered table of contents with `buzz://message?channel=…&id=…` deep links per lesson.
- **Progress:** none, or a ✅ reaction convention. Be explicit that reactions are visible to classmates.
- **Video:** keep lessons ≤10 min (the 600 s cap), H.264. You upload from the desktop, which transcodes with your local `ffmpeg`.

**v1: native courses, phase 1.5.** New kinds plus UI.

| Concept | Representation | Rationale |
|---|---|---|
| Course | New addressable kind *Course Manifest* `{d: course-slug, h: channel-id}`: title, description, cover, ordered `a` tags pointing at lessons, optional per-module grouping tags, drip rules (`not_before` per lesson) | Ordering and structure in one replaceable event |
| Lesson | **NIP-23 long-form** (`30023`, already in `kind.rs`) with `h` = course channel; video via `imeta` | Reuses a standard kind; renders in any NIP-23 client |
| Progress | New addressable kind *Lesson Progress* `{d: course-slug}`, NIP-44 encrypted to self, following the reminders pattern (`30300`, `kind.rs:96-102`) | Private by default; no relay read-policy work |
| Aggregate completion (owner analytics) | Later: member opt-in shares progress, or a relay read-policy that lets channel admins read progress events | Avoids building an access rule before it's needed |

**Kind numbers.** Choose unused addressable numbers in `30000-39999` and register them in `kind.rs` + `ALL_KINDS`. Avoid numbers taken by public NIPs (for example `30402` is NIP-99 classifieds). Add the scope mapping in `ingest.rs:required_scope_for_kind`.

**Client.** Build the course UI as a **buzz-app page plugin** (`pages.register`, versioned routes, `conversation.ui.*` reuse for lesson discussion). That needs two FOUNDATION-level host changes:

1. Add the new kinds plus `30023` to the native signer allowlist (`buzz-app/src-tauri/src/relay.rs:259-318`) and the dev broker (`browser-host/session-commands.mjs`).
2. Nothing else for persistence, since course data is events.

The alternative is a new `desktop/src/features/courses/` in the older client. That's faster to ship today but invests in the client upstream is replacing.

**Video, beyond v0.**

- **Raise the cap:** make the 600 s limit configurable (`BUZZ_MAX_VIDEO_SECONDS`, `validation.rs:369`). Small, generic and upstreamable. Uploads still depend on the uploader's `ffmpeg` and progressive MP4.
- **Long-form or adaptive video:** either an external stream host (Bunny Stream, Cloudflare Stream, Mux) referenced by URL in the lesson, or a server-side transcode-to-HLS worker. The external host trades a dependency for zero media engineering. **Recommendation:** short native lessons first, external host only if long lectures become common.

### 7.3 Moderation

Mostly configuration and one screen:

1. **Use what ships.**
   - Members report from the desktop (`ReportMessageDialog`).
   - You and admins act from message menus (timeout, ban) and resolve the queue via `buzz moderation reports` / `buzz moderation resolve` in the CLI.
   - Configure the join policy (terms, code of conduct) so every member accepts rules before joining.
2. **Owner queue screen.** The desktop has a `listReports` API client (`desktop/src/shared/api/moderation.ts`) with no UI consumer. Two options:
   - a small **buzz-app plugin page** over `/moderation/reports|audit|restricted`;
   - a page on the Steward's web admin, signed with NIP-07 like `admin-web`.
3. **Moderation-assist agent.** An agent subscribed with `--subscribe all` on public channels can **file NIP-56 reports** with a reason when it sees likely spam. This respects the vision's rule that reports are signals, never triggers: a human still decides, and the agent just widens the net. Pair it with a daily "queue summary" DM to admins.
4. **Later:**
   - a volunteer *moderator* role (authority is capability-structured, `moderation_authz.rs:6`);
   - mobile report UI (absent);
   - an escalation pipeline (currently a status value only).

### 7.4 Voice-note transcription for agents (#7339)

**Recommended: transcribe in the harness.**

- When a prompted event has an `imeta` whose filename starts `voice-note-`, `buzz-acp`:
  1. downloads the blob with Blossom auth;
  2. decodes the AAC track;
  3. transcribes it;
  4. appends a `Transcript:` block in `format_event_block` (`crates/buzz-acp/src/queue.rs:1638-1656`).
- **Escape the transcript** with `prompt_framing::escape_semantic_text` / `semantic_section` (`prompt_framing.rs:7-50`). A spoken message must not be able to inject prompt structure.
- **STT engine, two options:**
  - local: whisper.cpp or sherpa-onnx Parakeet, which desktop huddles already use;
  - API: an OpenAI-compatible transcription endpoint.
- **Cache** transcripts by blob SHA-256 so retries and steer re-prompts don't pay twice.
- **Bound everything** (Rule 4): max 5-minute input (the recorder already caps it), a timeout, and a size cap.
- **Failure mode:** emit `Transcript: unavailable (<reason>)` rather than silently dropping (Rule 1).

**Why the harness:**

- It covers every sender (desktop, mobile, buzz-app) and every runtime (Hermes included) from one place.
- It needs no client or event-format change.
- It is the direct fix for upstream #7339 option (a), so it can be upstreamed and dropped from your patch queue.

**Later: human-facing transcripts.** The sender's client adds a `transcript` field to the voice note's `imeta` before signing. Desktop already ships an STT model; iOS can use on-device speech. This also fixes the accessibility gap noted in `VoiceNoteCard.tsx`.

### 7.5 Agents with names and personalities

What works today:

- **Personas.** In the desktop, create personas (kind `30175`) with display name, avatar, system prompt, model and runtime (`hermes-acp`). Publish them `shared` so they appear in the community catalog.
- **Channel Q&A.** Run each agent with `buzz-acp --respond-to anyone` in the channels it serves. Members @mention it, and it gets thread context plus up to 100 recent messages.
- **Proactive ops.** Use heartbeat prompts (`--heartbeat-interval`), reminders, and schedule-triggered workflows that @mention the agent.
- **Memory.** NIP-AE core memory carries the community FAQ and the agent's personality canon.

**Required patches** (small, upstreamable):

1. **DM policy.** Add `--dm-respond-to` (default: the current owner/sibling-only behavior). For your agents, set it to `members`, meaning any authenticated community member.
   - Pair it with a **per-member turn budget** in the harness (for example N turns per hour per pubkey) so one member can't run up your LLM bill. Rule 4 applies.
   - Code: `author_allowed`, `crates/buzz-acp/src/lib.rs:569-592`.
2. **Member-joined trigger.** Add a workflow trigger on the relay's `member_joined` system message (`40099`) and/or NIP-43 `8000`. That gives "welcome new member" as YAML.
   - Until then, the Steward already sees `8000`. It can post a welcome that @mentions Miles in `#welcome` (via `send_message`, or a `/hooks/{id}` webhook to a workflow) and ask Miles to DM the newcomer, which needs patch 1 for the conversation to continue.
3. **Workflow `send_dm` (WF-07).** Optional once the above exist; upstream it if you implement it.

**Cast** (per your naming rule):

| Agent | Role | Subscribes | Notes |
|---|---|---|---|
| **Miles Morales** | Onboarding helper | `#welcome`, DMs | Friendly first contact; walks newcomers to courses and introductions |
| **Ops** (functional name) | Operations | Admin channels, heartbeat | Queue summaries, digest drafting, health checks |
| **Q&A** (themed or functional) | Member questions | `#ask`, forum, DMs | FAQ in core memory; answers voice notes once 7.4 lands |
| **Digest** | Weekly email digest drafting | Open channels (read) | Output reviewed by you, then sent by the Steward |

Every agent is visibly an agent: "managed by" label and bot role. Keep that; it's part of the trust story for a "trustworthy systems" audience.

---

## 8. Phase two: payments and entitlements

- **Processor.** Stripe Checkout (subscriptions and one-off products) plus the Customer Portal. A merchant of record (Paddle, Lemon Squeezy) costs more per transaction but handles global VAT and sales tax. Worth it if you sell internationally and want no tax admin.
  - "No platform cut" is achievable. Processing fees are not avoidable. Stripe is ~2.9% + 30¢ in the US.
- **Identity binding.** Paying happens on the Steward site, signed in by a NIP-98/NIP-07 challenge or by email magic link plus the member's stored pubkey. Pass the pubkey as `client_reference_id`.
- **Webhooks.** These Stripe events become journaled tier transitions, and the reconciler does the rest:
  - `checkout.session.completed`
  - `customer.subscription.updated` / `.deleted`
  - `invoice.payment_failed`

  Use a grace period on failure. Downgrades remove gated channel membership but **never** delete content or the member.
- **Gating options, in increasing cost:**
  1. Paid channels and courses (private channels). Already the model above.
  2. Paid community (`BUZZ_REQUIRE_RELAY_MEMBERSHIP=true` plus NIP-43 `9030`/`9031` from the Steward). Note this is deployment-wide.
  3. Per-lesson gating inside a course. Needs read-policy work; avoid.
- **Lightning / zaps (NIP-57).** Absent in Buzz, and a natural Nostr fit. Treat it as an experiment, not the primary rail, for a mainstream technical audience.

---

## 9. Later: multiple communities and centralized analytics

**One relay per community** fits the code, for two reasons:

- Signup policy, join policy, allowlist and NIP-FI mode are **deployment-global** (`config.rs`).
- Multi-tenant isolation currently rests on query discipline (RLS is DOC-ONLY).

Separate deployments give each community its own policy and a hard isolation boundary. Operationally:

- one Postgres server with a database per relay;
- one Redis with key prefixes, or separate instances;
- one S3 bucket per relay;
- Railway's one-click relay template or `docker-compose.yml` as the unit.

Member identity is portable: the same keypair joins every community, and profiles are per community by design.

**Analytics consumer:**

| Source | Use |
|---|---|
| **Read replica → ETL** (recommended) | Pull `events` (partitioned, `community_id`, `kind`, `channel_id`, `created_at`), `channel_members`, `relay_members` and `moderation_actions` into a warehouse (DuckDB/MotherDuck, ClickHouse, or Postgres), keyed `(relay_id, community_id)`. **Exclude private kinds**: reuse the relay's own exclusion list from `search_tsv` (1059, 30179, 30300, 30350, 30622, 44100, 44101, 44200) plus DMs, and drop `content` for anything not in an open channel. |
| Prometheus | Fleet and usage gauges (`docs/usage-metrics.md`); requires a verified `READ_DATABASE_URL` |
| Steward database | Funnel (signup → claim → first post), tiers, revenue |

Join keys across relays are pubkeys, so cross-community cohorts ("members active in two communities") come free.

---

## 10. Fork and upstream strategy

Upstream velocity, last 30 days: `block/buzz` ~231 commits (desktop 63, mobile 57, crates 104); `block/buzz-app` ~702.

1. **Phase one needs no client fork.** Members use official builds. Your fork carries only relay and harness patches, deployed by you.
2. **Patch queue, kept under ~10 small commits on a `community` branch**, rebased weekly onto `upstream/main`:

   | Patch | Upstreamable? |
   |---|---|
   | ACP DM response policy + per-member budget | Yes |
   | ACP voice-note transcripts (#7339) | Yes |
   | Configurable max video duration | Yes |
   | Workflow member-joined trigger | Yes |
   | Invite redemption record (claimant pubkey) | Likely |
   | Course kinds registered in `kind.rs` + scopes | Possibly (as a NIP-style proposal); otherwise carry |

   Open each as a PR to `block/buzz` (they follow the repo's PR checklist and DCO `-s`). Each merged PR leaves your queue.
3. **New code lives in new places:** the Steward repo, buzz-app plugins (a separate package, installed with `buzzodz`), and agent configs. None of these rebase.
4. **Branding.** Rebranding buzz-app means editing scattered `Buzz` strings (~124 lines), the `buzz://` scheme and app IDs. Defer until you ship your own client. Then do it with build-time substitution rather than editing upstream files, to keep rebases clean.

---

## 11. Ranked phase-one punch list (impact ÷ effort)

Effort: **XS** < 2 days, **S** ≤ 1 week, **M** 1–2 weeks, **L** > 2 weeks (solo, agent-assisted).

| # | Item | Impact | Effort | Why this rank |
|---|---|---|---|---|
| 1 | **Relay up + founding-member funnel + tier model** (§7.0) | Blocking | M | Nothing else matters without members. Includes the landing page, newsletter job, single-use invites, the invite-redemption patch and reconciler. |
| 2 | **Courses v0** from forum + canvas (§7.2) | High | XS | Mostly content work; gives the "course" story on day one |
| 3 | **Agents in channels** (Miles, Q&A personas on Hermes) | High | XS–S | Works today with `--respond-to anyone`; it's the differentiator |
| 4 | **ACP DM policy + per-member budget** (§7.5) | High | S | Unlocks DM onboarding and Q&A; upstreamable |
| 5 | **Email: mention/DM alerts + weekly digest** (§7.1) | High | M | The return-visit engine. Uses the operator-listener outbox; digest can be agent-drafted. |
| 6 | **Voice-note transcripts for agents** (§7.4) | Medium | S–M | Fixes #7339 for every client; good build-in-public story |
| 7 | **Moderation:** join policy + CLI queue now, owner queue page next (§7.3) | Medium (rises with size) | XS → S | Most of it ships; the queue page can wait until reports exist |
| 8 | **Member-joined workflow trigger** | Low–Medium | S | The Steward can cover this until it lands |
| 9 | **Courses v1 native** (kinds + buzz-app plugin) | High | L | Phase 1.5; needs a client build you distribute |

A sensible order for a 4–8-week build: **1 → 3 → 2 → 4 → 5 → 6 → 7**, with 8 and 9 after launch.

---

## 12. Mapping build milestones to the Medium series

| Post | Milestone | Artifact readers can join |
|---|---|---|
| 1 | Why I'm forking Buzz; this architecture | Founding link (queued invite) |
| 2 | Standing up the relay; join policy; the founding funnel | Invites start flowing |
| 3 | Agents with personalities: meet Miles | Channel Q&A live |
| 4 | Teaching agents to DM responsibly (the DM-policy patch + budgets) | DM onboarding |
| 5 | Agents that listen: voice-note transcripts (#7339 PR) | Voice Q&A |
| 6 | Bringing people back: email without becoming spam | Digests |
| 7 | Courses v0 from primitives; what native courses need | First course |
| 8 | Moderation and governance; one-month retrospective | Open doors |

The newsletter (monthly) recaps two to four posts and the community's own highlights. Pull them from the digest pipeline so it costs no extra writing.

---

## 13. Risks and open questions

**Risks:**

- **Client distribution and keys.**
  - Members must install a desktop app and hold a Nostr key. The official app creates one, but key backup is on them.
  - Windows builds are unsigned. Mobile availability of the official iOS app for third-party relays (and its push) depends on Block.
  - *Mitigation:* a technical founding cohort; a written "set up in 3 minutes" guide; a public web client as a phase-2 goal. buzz-app has a host-neutral `Signer` seam (`src/features/relay/transport.ts:194-203`) and `web/` has NIP-07 and signer code to start from.
- **Doc drift in Buzz.** Several docs disagree with the code (rate limiting, persona spec, NIP-29 `9009`). Verify against code before building on any doc claim.
- **Isolation.** If you ever host multiple communities on one deployment, isolation is query discipline only (no RLS). Prefer relay-per-community until upstream lands RLS.
- **Experimental surfaces.** Push and remote agents are explicitly experimental. Workflows have stubbed actions (WF-07, WF-08). Design around them, not on them.
- **LLM cost and abuse.** Opening agents to all members (channel `anyone`, DM `members`) needs budgets and rate limits before the community grows.
- **Upstream product direction.** Buzz is an engineering-workspace product first ("the relay is the workspace"). Community features (courses, tiers) are yours to carry. Keep them in the Steward and plugins so a divergent upstream can't strand them.

**Open questions to resolve before building:**

1. Which newsletter provider holds the 2.2K subscribers today? Is the Medium email list exportable, or does the founding funnel start a fresh list?
2. ~~Does the invites API expose the claimant?~~ No. A redemption-record patch is needed (§7.0).
3. ~~Does the listener cover DMs?~~ Only if DM messages carry recipient `p` tags. Verify on a live DM (§7.1).
4. Can members use Block's official iOS app against your relay, including push through Block's gateway? Ask upstream. If not, mobile is desktop-only until you ship an iOS build.
5. Hosting: Railway one-click vs self-managed Docker. Who holds the relay and operator keys, and how are they backed up?
6. Course video length: will most lessons fit in 10 minutes, or is an external stream host needed early?

---

## Appendix A: key file references

| Area | Location |
|---|---|
| Event kinds | `crates/buzz-core/src/kind.rs` |
| Connect/auth gates | `crates/buzz-relay/src/handlers/auth.rs:318-650` |
| Write scopes, ban/timeout write block | `crates/buzz-relay/src/handlers/ingest.rs:501, 2228-2269, 2538` |
| Read filtering | `crates/buzz-relay/src/handlers/req.rs:109-194` |
| Community admin commands | `crates/buzz-relay/src/handlers/relay_admin.rs` |
| Channel membership policy | `crates/buzz-relay/src/handlers/channel_authz.rs`, `side_effects.rs:1429-2166` |
| Invites | `crates/buzz-relay/src/api/invites.rs`, `invite_token.rs` |
| Moderation | `crates/buzz-relay/src/handlers/{report,moderation_commands,moderation_authz}.rs`, `report_resolution.rs` |
| Operator listener (mention webhooks) | `crates/buzz-relay/src/operator_listener.rs`, `api/operator.rs:150-215` |
| Media validation | `crates/buzz-media/src/validation.rs` (audio reject `:196-205`, video cap `:369`) |
| Push | `crates/buzz-push-gateway`, `crates/buzz-relay/src/push_runtime.rs`, `docs/buzz-push.md` |
| ACP author gate | `crates/buzz-acp/src/lib.rs:569-592` |
| ACP prompt event block | `crates/buzz-acp/src/queue.rs:1617-1680`; framing `prompt_framing.rs` |
| Workflows | `crates/buzz-workflow/src/{schema,executor}.rs` |
| buzz-app plugin contract | `buzz-app/docs/plugin-architecture.md`, `src/plugins/author.ts` |
| buzz-app signer kind allowlist | `buzz-app/src-tauri/src/relay.rs:259-318` |
| Invite web page | `web/src/features/invite/ui/InvitePage.tsx` |

## Appendix B: relevant configuration

| Variable | Purpose |
|---|---|
| `BUZZ_REQUIRE_RELAY_MEMBERSHIP` | Closed community (default open) |
| `BUZZ_PUBKEY_ALLOWLIST` | Allowlist gate |
| Join-policy config (`config.rs:380`) | Terms, privacy, age gate |
| `BUZZ_OPERATOR_LISTENERS` | `pubkey:url` pairs for mention webhooks (Steward) |
| `RELAY_OPERATOR_PUBKEYS` | Who may call `/operator/*` |
| `BUZZ_PUSH_ENABLED` | Push matcher (default off) |
| `BUZZ_MAX_*_BYTES` | Media size limits |
| `BUZZ_RATE_LIMIT_*` | Human and agent rate limits |
| `BUZZ_WEB_DIR` / `BUZZ_ADMIN_WEB_DIR` | Serve `web/` and `admin-web/` |
| `BUZZ_ALLOW_NIP_OA_AUTH` | Agents admitted via owner membership (default off) |
| `READ_DATABASE_URL` | Verified replica; required for usage gauges and recommended for analytics |
