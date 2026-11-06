# Tokenmaxxing Clan Edition — Implementation Task Index

> Planning document only. This index does not approve product decisions or authorize implementation.
>
> Last grounded against the repository on 2026-11-07.

## Contents

- [How to use this index](#how-to-use-this-index)
- [Verified existing baseline](#verified-existing-baseline)
- [Confirmed constraints](#confirmed-constraints)
- [Recommendations, not confirmed requirements](#recommendations-not-confirmed-requirements)
- [Scope boundaries](#scope-boundaries)
- [Decision register](#decision-register)
- [Proposed role and permission matrix](#proposed-role-and-permission-matrix)
- [Real-time chat architecture options](#real-time-chat-architecture-options)
- [Recommended implementation order](#recommended-implementation-order)
- [1. Foundations, contracts, permissions, and migrations](#1-foundations-contracts-permissions-and-migrations)
- [2. Clan identity, discovery, recruitment, and membership](#2-clan-identity-discovery-recruitment-and-membership)
- [3. Profiles, friends, blocking, and privacy](#3-profiles-friends-blocking-and-privacy)
- [4. Clan home, roster, stats, activity, and announcements](#4-clan-home-roster-stats-activity-and-announcements)
- [5. Clan chat](#5-clan-chat)
- [6. Clan wars](#6-clan-wars)
- [7. Clan progression and cooperative goals](#7-clan-progression-and-cooperative-goals)
- [8. Leaderboards](#8-leaderboards)
- [9. Seasons, leagues, matchmaking, and rivalries](#9-seasons-leagues-matchmaking-and-rivalries)
- [10. Notifications, security, moderation, operations, tests, and rollout](#10-notifications-security-moderation-operations-tests-and-rollout)
- [11. Optional later ideas](#11-optional-later-ideas)
- [Definition of complete for an implementation task](#definition-of-complete-for-an-implementation-task)

## How to use this index

Every implementation item is intentionally unchecked and has:

- a stable ID (`CLAN-NNN`);
- a priority (`MVP`, `Next`, or `Later`);
- explicit dependencies by ID or decision;
- a short, reviewable done criterion.

Complete tasks in the [recommended implementation order](#recommended-implementation-order), not strictly by numeric ID. If a requirement changes, preserve IDs and mark the old task superseded rather than renumbering the file. Product decisions in the register must be resolved before tasks that depend on them. Each task should normally fit one focused pull request; split a task without changing its ID if implementation reveals an unsafe review size.

## Verified existing baseline

The following statements were checked against the current repository. They describe the baseline to extend; they are not claims that every adjacent feature is missing.

- The monorepo uses Cloudflare Workers/Alchemy, Cloudflare D1, Drizzle ORM and generated migration folders under `packages/db/migrations/`.
- The API is modeled with Effect HTTP API groups, Effect services/repositories, typed errors, and layers. Expected domain failures use the Effect error channel.
- Browser authentication is custom GitHub/Google OAuth plus hashed server-side sessions in D1. CLI authentication uses revocable `tmx_` tokens and a device-code login flow. No clan task should introduce a second auth system.
- Published CLI request/response compatibility is frozen by `packages/api-contract/fixtures/` and `apps/api/src/http/cli-compat.test.ts`. Clan work should add browser-facing contracts without changing released CLI shapes or typed CLI error tags.
- Daily usage is aggregated by `(deviceId, date, source, model)`. Public usage exposes aggregates, not raw coding-agent logs.
- `date` is an opaque local-day `YYYY-MM-DD` key. Window arithmetic uses `shiftDayKey`/`utcDayKey`; ranking compares fixed-width day keys lexicographically and accepts the documented UTC/local edge wobble.
- Raw ingestion uses a server `PARSER_VERSION` (`ccusage-v20-raw-7` when this document was written). Same-parser lower recounts are retained as lost-log protection; a parser bump is required for a legitimate correction that lowers or drops usage.
- Cost is frozen on unchanged token counts, replaced when token counts change, and may be filled when the stored value is zero. Clan scoring must consume the canonical stored aggregate instead of repricing history.
- Existing public leaderboard dimensions are `spend|tokens` and `7d|30d|all`; the default is 30-day spend. Current global ranking produces a unique row-number rank with user ID as the deterministic tie-break.
- Public profiles, aggregate stats, and leaderboards already exclude shadow-banned users. Admin authorization is based on verified configured emails. Clan visibility and moderation must compose with those rules.
- The web app uses TanStack Router/Query and the shared API contract. Signed-in routes use the existing session viewer check.
- D1 has a 100-bound-parameter limit, enforced by the test harness. Existing code uses batching and `json_each` to avoid unbounded binds.
- The API Worker already binds D1, R2, Cloudflare rate-limit bindings, an hourly cleanup cron, and observability. There is no existing Durable Object, Queue, or WebSocket chat binding in the inspected configuration.

## Confirmed constraints

These are constraints supplied for clan edition and should not be changed silently:

1. This is friendly competition over coding-agent token and API-equivalent spend aggregates. It is not combat, productivity scoring, or code-quality measurement.
2. Preserve Cloudflare D1 + Drizzle, the Effect API architecture, the existing frontend, and custom OAuth/session + CLI sync flows.
3. Do not expose or ingest raw logs for clans, chat, wars, leaderboards, progression, or moderation. Use canonical aggregates and explicit clan-domain records only.
4. Preserve sync idempotency, parser-version correction behavior, frozen-cost behavior, opaque date-key semantics, CLI compatibility, and D1 binding limits.
5. Cover leader, co-leader, and member as the baseline roles and define an explicit permission model. This does not permanently forbid future roles.
6. Clan war score metric, roster fairness, attribution, correction windows, ties, and rewards are unresolved product decisions. Proposals in this document are not approvals.
7. Implement tasks one by one; do not bundle the entire clan edition into one release.

## Recommendations, not confirmed requirements

- Keep D1 as the system of record and the existing auth/session service as the only identity authority.
- Add clan domains as Effect service/repository pairs and additive API groups; use `Schema.TaggedError` for public errors and `Data.TaggedError` for internal typed failures.
- Prefer append-only audit/event rows for sensitive membership, moderation, war, reward, and leadership changes.
- Use durable idempotency keys for accept/approve/finalize/reward operations and D1 batches for multi-row atomicity.
- For MVP, prefer at most one active clan membership per person, subject to `DEC-021`; keep the schema/history model capable of recording past memberships without moving usage ownership.
- Use tokens as the initial war metric because it is less sensitive to price-table and speed-tier changes; keep spend as a possible secondary display metric. This remains blocked on `DEC-010`.
- Prototype per-clan Durable Objects with hibernating WebSockets for live delivery, while retaining D1 as chat history/unread truth. Do not select it until `CLAN-067` measures local development, reconnect, auth, fanout, and operational behavior.
- Add Queues only if asynchronous notification fanout or moderation processing demonstrates a need; do not use a Queue as the chat transport.
- Defer DMs unless a later proposal is explicitly approved; the planned chat tasks cover clan rooms only.
- Start without seasons, leagues, raids, inventories, currency, monetization, or referral rewards. Treat cosmetics as display-only. Add optional systems only after the core clan loop is stable and measured.

## Scope boundaries

### In scope

- Clan identity, discovery, membership, roles, moderation, home, stats, announcements, chat, wars, progression, leaderboards, friends, notifications, and operational controls.
- Public and private presentation of canonical token/spend aggregates.
- Additive schema, contract, API, UI, tests, observability, migration, and rollout work.

### Out of scope unless separately approved

- Reading source code, prompts, messages, file names, repository names, or raw agent logs.
- Evaluating developer productivity or code quality.
- DMs, voice/video, file uploads, arbitrary rich embeds, bots, or third-party chat bridges.
- Payments, purchasable currency, loot boxes, tradable items, inventory, marketplace, or pay-to-win mechanics.
- Replacing D1, Drizzle, Effect, the web frontend, OAuth/session auth, CLI auth, or the usage sync protocol.

## Decision register

Resolve decisions by recording the selected option, rationale, date, and owner. A recommendation is a starting point, not consent.

| ID | Status | Product/architecture question | Initial recommendation | Blocks |
| --- | --- | --- | --- | --- |
| `DEC-001` | Unresolved | Clan name/tag rules, uniqueness, rename cooldown, reserved words | Case-insensitive unique slug; display name mutable; tag 2–6 characters; 30-day rename cooldown | `CLAN-017`, `CLAN-018` |
| `DEC-002` | Unresolved | MVP clan capacity and whether level can raise it | Fixed 50 for MVP; evaluate progression-based increases later | `CLAN-030`, `CLAN-116` |
| `DEC-003` | Unresolved | Recruitment defaults and discovery visibility | Public/listed + request-to-join default; open and invite-only available | `CLAN-022`–`CLAN-028` |
| `DEC-004` | Unresolved | Leave/rejoin cooldown, kick cooldown, and ban duration | 24-hour voluntary rejoin cooldown; kick configurable up to 7 days; bans explicit and indefinite until lifted | `CLAN-031`, `CLAN-032`, `CLAN-034` |
| `DEC-005` | Unresolved | Last-leader and inactive-leader policy | Require transfer before leave/disband; no automatic transfer until a separately reviewed inactivity policy exists | `CLAN-041`, `CLAN-042` |
| `DEC-006` | Unresolved | Clan disband retention and name reuse | Soft-delete clan; preserve audit/war results; redact chat on retention schedule; delay slug reuse 30 days | `CLAN-019`, `CLAN-020` |
| `DEC-007` | Unresolved | Friends/profile privacy levels | Public, friends-only, clan-only, private per surface; keep public profile default only if explicitly approved | `CLAN-051`–`CLAN-056` |
| `DEC-008` | Unresolved | Friend block semantics | Block cancels pending friendship, removes friendship, suppresses discovery/mentions/notifications, and does not reveal the block | `CLAN-052`, `CLAN-053`, `CLAN-083` |
| `DEC-009` | Unresolved | Real-time clan chat transport | Time-box Durable Object + hibernating WebSocket prototype; keep D1 authoritative; compare with polling/SSE before selection | `CLAN-067`, `CLAN-076`–`CLAN-079` |
| `DEC-010` | Unresolved | War score: tokens, API-equivalent cost, or both | Tokens for score; frozen spend shown as context only; never reprice past results | `CLAN-087`, `CLAN-098`, `CLAN-104` |
| `DEC-011` | Unresolved | War roster size, opt-in rules, lock time, substitutions | Equal-size opt-in rosters; lock before prep ends; no substitutions after start except admin recovery | `CLAN-089`–`CLAN-095` |
| `DEC-012` | Unresolved | Fairness model for unequal historical usage/budgets | Match by recent canonical usage bands and roster size; publish the formula; never use raw logs | `CLAN-092`, `CLAN-093`, `CLAN-106` |
| `DEC-013` | Unresolved | War attribution when joining/leaving clans | Freeze participant-to-clan attribution at roster lock; membership changes do not move in-progress points | `CLAN-096`, `CLAN-099` |
| `DEC-014` | Unresolved | Late sync, offline devices, parser corrections, and finalization grace | Provisional end plus 24-hour sync grace; one deterministic re-finalization window; parser-bump corrections handled explicitly | `CLAN-100`–`CLAN-103` |
| `DEC-015` | Unresolved | War ties, cancellation, forfeits, rewards | Explicit draw result; symmetric base rewards; cancellation yields no competitive reward; admin repair is audited | `CLAN-105`, `CLAN-109`, `CLAN-111` |
| `DEC-016` | Unresolved | Clan XP formula and lifetime versus seasonal progression | Lifetime clan level; seasonal track later; XP from capped participation, goals, and wars with idempotent grants | `CLAN-114`–`CLAN-120` |
| `DEC-017` | Unresolved | Ranking ties and eligibility | Competition rank for equal score, stable secondary display order, minimum active members/usage, disclose eligibility | `CLAN-125`–`CLAN-131` |
| `DEC-018` | Unresolved | Chat history/edit/delete/retention rules | Cursor pagination; 15-minute self-edit; soft-delete tombstones; 90-day retention initially | `CLAN-068`–`CLAN-072`, `CLAN-153` |
| `DEC-019` | Unresolved | Mention limits and `@all` authority | Member mentions allowed; role mentions for co-leader+; `@all` leader only, rate-limited, preference-respecting | `CLAN-073`, `CLAN-074`, `CLAN-151` |
| `DEC-020` | Unresolved | Seasonal leagues, matchmaking, and rivalries | Later, opt-in, and no impact on core clan access | `CLAN-134`–`CLAN-139` |
| `DEC-021` | Unresolved | Membership cardinality: one active clan per person or multi-clan membership | One active clan for MVP, with historical memberships retained; do not encode the rule until approved | `CLAN-003`, `CLAN-016`, `CLAN-026`, `CLAN-028`, `CLAN-045` |

## Proposed role and permission matrix

This matrix is a review proposal for the requested leader/co-leader/member baseline, not a claim that these must be the only roles forever. Enforce approved permissions server-side through shared policy helpers; UI hiding is not authorization. Internal admins may moderate through separately audited admin tools and do not implicitly become clan members.

| Action | Leader | Co-leader | Member | Notes |
| --- | :---: | :---: | :---: | --- |
| View clan home/roster/chat | Yes | Yes | Yes | Subject to clan visibility and membership state |
| Send/edit/delete own chat message | Yes | Yes | Yes | Within retention/edit policy |
| Delete another member's chat message | Yes | Yes | No | Moderation event required |
| Pin/unpin message | Yes | Yes | No | Limit active pins |
| Mention a member | Yes | Yes | Yes | Respect blocks/preferences |
| Mention a role | Yes | Yes | No | Rate-limited |
| Mention `@all` | Yes | Proposed no | No | `DEC-019` may grant co-leaders permission |
| Create/edit announcements | Yes | Yes | No | Deletion remains audited |
| Edit clan profile/recruitment | Yes | Yes | No | Rename/tag changes may remain leader-only |
| Invite applicants/members | Yes | Yes | No | Capacity and ban checks always apply |
| Approve/reject join requests | Yes | Yes | No | Cannot bypass the approved membership-cardinality rule |
| Kick or clan-ban a member | Yes | Yes | No | Co-leader cannot target leader or peer co-leader by default |
| Unban a person | Yes | Yes | No | Audited |
| Promote member to co-leader | Yes | No | No | Leader-only |
| Demote co-leader | Yes | No | No | Leader-only |
| Transfer leadership | Yes | No | No | Explicit acceptance recommended |
| Disband clan | Yes | No | No | Re-auth/confirmation and last-leader guard |
| Configure/accept war | Yes | Yes | No | Roster participants opt in individually |
| Lock war roster | Yes | Yes | No | Cannot override participant opt-out |
| Cancel/forfeit war | Yes | Proposed yes | No | State-dependent; `DEC-015` |
| Configure progression cosmetics | Yes | Yes | No | Display-only rewards |
| View clan audit log | Yes | Proposed yes | No | Sensitive actor/target data minimized |

## Real-time chat architecture options

Do not select a transport until the prototype task is complete.

| Option | Fit | Advantages | Risks/limits | Recommendation |
| --- | --- | --- | --- | --- |
| Durable Object per clan + hibernating WebSockets; D1 history | Strong | Single room coordinator, ordered fanout, presence/reconnect hooks, hibernation keeps idle sockets connected | New binding/migration/runtime path; must restore socket metadata after hibernation; still needs D1 durability and backpressure tests | Prototype first; leading candidate |
| Worker HTTP + D1 + short/long polling | Acceptable MVP fallback | No new Cloudflare product; easiest local/operational model | More read load/latency; weaker presence and delivery behavior; D1 is single-threaded per database | Keep as fallback and degraded mode |
| Server-Sent Events for receive + HTTP send | Conditional | Simpler one-way stream than WebSockets | Reconnect/fanout coordination still needed; browser/proxy limits; does not solve ordering or durable history | Compare in prototype, not default |
| Cloudflare Queue | Supporting only | Decouples notification/moderation fanout and retries | Asynchronous, not an interactive room transport; delivery semantics require idempotent consumers | Add only for demonstrated async fanout need |

Cloudflare references: [Durable Object WebSockets](https://developers.cloudflare.com/durable-objects/best-practices/websockets/), [Durable Object lifecycle](https://developers.cloudflare.com/durable-objects/concepts/durable-object-lifecycle/), [Queues](https://developers.cloudflare.com/queues/get-started/), and [D1 limits](https://developers.cloudflare.com/d1/platform/limits/).

## Recommended implementation order

1. Resolve the MVP subset of the decision register, especially identity, capacity, privacy, chat transport, and war scoring/attribution.
2. Complete foundations (`CLAN-001`–`CLAN-015`) with additive schema/contracts, policies, compatibility tests, and indexes.
3. Ship clan identity and membership (`CLAN-016`–`CLAN-045`) behind a flag, then exercise migration and approved membership-cardinality concurrency paths.
4. Add friends/privacy (`CLAN-046`–`CLAN-056`) before friend leaderboards, mentions, and notification fanout rely on those relationships.
5. Add the clan home and aggregate views (`CLAN-057`–`CLAN-065`), then baseline clan/member leaderboards (`CLAN-123`–`CLAN-133`).
6. Run the chat transport prototype (`CLAN-067`) before implementing the rest of chat (`CLAN-066`–`CLAN-085`).
7. Resolve all war decisions, then implement the war state machine and frozen attribution (`CLAN-086`–`CLAN-113`) without rewards first.
8. Add idempotent progression/rewards (`CLAN-114`–`CLAN-122`) only after war finalization is repairable and observed.
9. Complete notifications, moderation, recovery, performance, tests, and staged rollout (`CLAN-140`–`CLAN-155`) throughout each phase, not as a final cleanup batch.
10. Consider seasons (`CLAN-134`–`CLAN-139`) and optional ideas (`CLAN-156`–`CLAN-161`) only after core retention, fairness, abuse, and cost data are available.

## 1. Foundations, contracts, permissions, and migrations

- [ ] **CLAN-001 · MVP · deps: decisions used by MVP** — Record approved clan decisions as short ADRs and map every unresolved choice to a feature flag or blocked task. **Done:** no implementation path relies on an unstated default.
- [ ] **CLAN-002 · MVP · deps: CLAN-001** — Define clan-domain state diagrams for membership, invitation/request, ban, war, chat message, notification, and reward grant states. **Done:** legal transitions, terminal states, idempotent repeats, and rejecting transitions are documented and testable.
- [ ] **CLAN-003 · MVP · deps: CLAN-002, DEC-021** — Add core Drizzle tables for clans and active/historical memberships with foreign keys, timestamps, role, status, and normalized unique slug/tag fields. **Done:** schema enforces the approved membership cardinality and supports membership history without rewriting usage rows.
- [ ] **CLAN-004 · MVP · deps: CLAN-003** — Generate and review the initial clan migration using the repository migration workflow. **Done:** forward migration works on empty and representative existing databases; snapshots/tests show no unrelated rebuild or data loss.
- [ ] **CLAN-005 · MVP · deps: CLAN-002, CLAN-003** — Add tables/indexes for join requests, invitations, invitation links, clan bans, and leadership transfer offers. **Done:** uniqueness and expiry constraints prevent duplicate active records and cover lookup paths.
- [ ] **CLAN-006 · MVP · deps: CLAN-003** — Add append-only clan audit events with actor, target, action, bounded metadata, correlation/idempotency key, and timestamp. **Done:** sensitive state changes can be reconstructed without raw logs or mutable prose payloads.
- [ ] **CLAN-007 · Next · deps: CLAN-003** — Add friendship, friend-request, and user-block tables with canonical pair ordering and state constraints. **Done:** duplicate reverse-direction rows are impossible and block lookups are indexed.
- [ ] **CLAN-008 · Next · deps: CLAN-003, DEC-018** — Add chat messages, revisions/tombstones, replies, reactions, pins, and per-user read cursors. **Done:** cursor pagination and moderation queries are indexed and retention can run in bounded batches.
- [ ] **CLAN-009 · Next · deps: CLAN-003, CLAN-002** — Add war, participant, roster snapshot, score ledger, result, and finalization/idempotency tables. **Done:** frozen attribution and score derivation do not require copying or modifying `usage_days`.
- [ ] **CLAN-010 · Next · deps: CLAN-003, DEC-016** — Add clan XP ledger, level snapshot, cooperative goal, contribution, and reward-grant tables. **Done:** every XP/reward mutation has a unique source key and can be replayed safely.
- [ ] **CLAN-011 · Next · deps: CLAN-003** — Add notifications/preferences, reports, and clan announcements tables. **Done:** fanout can be deduplicated; reports preserve evidence references without copying raw usage logs.
- [ ] **CLAN-012 · MVP · deps: CLAN-002** — Add additive clan schemas, branded IDs, tagged public errors, API groups, and OpenAPI coverage in `@tokenmaxxing/api-contract`. **Done:** contracts reject excess fields where appropriate and frozen CLI fixtures remain byte/shape compatible.
- [ ] **CLAN-013 · MVP · deps: CLAN-012** — Scaffold Effect service/repository pairs and live D1 layers for clan domains. **Done:** expected failures stay typed in the Effect channel and services join the existing `ServicesLive` composition.
- [ ] **CLAN-014 · MVP · deps: CLAN-013, permission matrix approval** — Implement shared membership/role/visibility policy functions used by every clan handler. **Done:** table-driven tests cover each role/action/state combination and handlers do not duplicate authorization logic.
- [ ] **CLAN-015 · MVP · deps: CLAN-003–CLAN-014** — Add compatibility/performance guardrails: indexes, 100-bind-safe batching/`json_each`, migration tests, CLI compatibility tests, and query-plan fixtures for hot paths. **Done:** existing sync/profile/leaderboard tests remain unchanged and clan queries have bounded page sizes.

## 2. Clan identity, discovery, recruitment, and membership

- [ ] **CLAN-016 · MVP · deps: CLAN-003, CLAN-012, DEC-001, DEC-021** — Implement create-clan service/API with normalized slug/tag validation, reserved names, approved membership-cardinality check, and creator-as-leader batch. **Done:** concurrent creates cannot violate the approved cardinality or leave a leaderless clan.
- [ ] **CLAN-017 · MVP · deps: CLAN-016, DEC-001** — Implement clan profile edits for name, tag, description, avatar/banner references, links, and recruitment settings. **Done:** field limits, safe URL schemes, rename cooldown, audit, and permissions are enforced server-side.
- [ ] **CLAN-018 · MVP · deps: CLAN-017** — Build create/edit clan UI with accessible validation, preview, destructive-change warnings, and optimistic updates only where rollback is safe. **Done:** keyboard/screen-reader paths and API error states are covered.
- [ ] **CLAN-019 · MVP · deps: CLAN-006, DEC-006** — Implement leader-only disband preflight that reports blockers such as live wars, pending leadership transfer, or required retention. **Done:** preflight is read-only and returns typed blockers.
- [ ] **CLAN-020 · MVP · deps: CLAN-019** — Implement confirmed disband as an idempotent batch/repairable workflow. **Done:** active memberships close, discovery hides the clan, pending membership actions terminate, history policy is honored, and an audit event is written once.
- [ ] **CLAN-021 · MVP · deps: CLAN-016** — Add canonical public clan detail lookup by slug with stable redirects after rename if approved. **Done:** shadow-banned/disbanded/private visibility follows shared policy and avoids existence leaks.
- [ ] **CLAN-022 · MVP · deps: CLAN-021, DEC-003** — Implement paginated clan discovery with deterministic cursors and listed/unlisted visibility. **Done:** empty/inactive clans follow explicit eligibility rules and pagination cannot duplicate/skip on equal sort values.
- [ ] **CLAN-023 · MVP · deps: CLAN-022** — Add normalized name/tag search with bounded input and indexed prefix behavior. **Done:** case/Unicode policy is tested and query cost is bounded.
- [ ] **CLAN-024 · Next · deps: CLAN-022** — Add discovery filters for recruitment mode, capacity availability, activity band, language/region labels if approved, and clan level. **Done:** every filter has contract, index/query-plan coverage, URL state, and an empty-state explanation.
- [ ] **CLAN-025 · MVP · deps: CLAN-017, DEC-003** — Implement recruitment modes `open`, `request`, and `invite-only` with optional listed/unlisted setting. **Done:** mode changes define what happens to outstanding requests/invites and write audit events.
- [ ] **CLAN-026 · MVP · deps: CLAN-025, CLAN-030, DEC-021** — Implement open join in one atomic membership operation. **Done:** capacity, approved membership cardinality, ban, cooldown, clan state, and duplicate retry checks happen at commit time.
- [ ] **CLAN-027 · MVP · deps: CLAN-025** — Implement request-to-join creation with optional bounded note. **Done:** duplicates, blocks/bans, existing membership, invite interaction, rate limits, and privacy-safe errors are covered.
- [ ] **CLAN-028 · MVP · deps: CLAN-027, DEC-021** — Implement applicant cancel and authorized approve/reject transitions. **Done:** approval rechecks capacity/cardinality/ban atomically; terminal retries are idempotent; rejection reasons are bounded and optional.
- [ ] **CLAN-029 · MVP · deps: CLAN-025, CLAN-005** — Implement direct invites and shareable invitation links with scope, expiry, use count, cancellation, and creator attribution. **Done:** accept/decline/expire/cancel paths are typed, audited, and safe against replay.
- [ ] **CLAN-030 · MVP · deps: CLAN-003, DEC-002** — Enforce capacity through one shared policy at join, approve, invite accept, restore, and admin repair boundaries. **Done:** concurrent final-slot claims yield one success and a typed full-clan result.
- [ ] **CLAN-031 · MVP · deps: CLAN-003, DEC-004** — Implement voluntary leave with war/leadership blockers and membership-history closure. **Done:** canonical usage ownership is untouched, current clan views update, and retry does not duplicate events.
- [ ] **CLAN-032 · MVP · deps: CLAN-031, DEC-004** — Enforce rejoin/kick cooldowns consistently across open join, request, invite, and links. **Done:** clients receive privacy-safe eligibility timestamps and staff have an audited override only if approved.
- [ ] **CLAN-033 · MVP · deps: CLAN-005, CLAN-014** — Implement clan bans with actor, reason code, optional expiry, and evidence/report reference. **Done:** banning cancels pending requests/invites and optionally removes active membership according to one atomic policy.
- [ ] **CLAN-034 · MVP · deps: CLAN-033** — Implement unban and expiry cleanup. **Done:** scheduled cleanup is bounded/idempotent, unban does not auto-rejoin, and both paths are audited.
- [ ] **CLAN-035 · MVP · deps: CLAN-014, CLAN-031** — Implement kick with role-target restrictions, cooldown, war-state behavior, and optional clan ban. **Done:** co-leaders cannot target leader/peer co-leaders under the approved matrix and frozen war attribution is unchanged.
- [ ] **CLAN-036 · MVP · deps: CLAN-014** — Define role values and expose effective permissions in member-management responses. **Done:** clients can render capabilities without reimplementing policy, while the server remains authoritative.
- [ ] **CLAN-037 · MVP · deps: CLAN-036** — Implement member-to-co-leader promotion. **Done:** only authorized actors can promote, role caps if any are enforced, and the operation is idempotent/audited.
- [ ] **CLAN-038 · MVP · deps: CLAN-037** — Implement co-leader demotion and voluntary step-down. **Done:** target/actor restrictions and live-war implications are explicit and tested.
- [ ] **CLAN-039 · MVP · deps: CLAN-036, DEC-005** — Implement leadership transfer offer/accept/cancel/expire flow. **Done:** acceptance atomically swaps roles, rechecks both memberships, and writes one correlated audit event.
- [ ] **CLAN-040 · MVP · deps: CLAN-039** — Add immediate leadership transfer only if separately approved, protected by recent authentication/confirmation. **Done:** it reuses the same atomic role-swap primitive and cannot create two/no leaders.
- [ ] **CLAN-041 · MVP · deps: CLAN-031, CLAN-039, DEC-005** — Enforce last-leader rules on leave, kick, demotion, account merge/deletion, and disband. **Done:** every path either transfers, disbands, or returns a typed blocker.
- [ ] **CLAN-042 · Next · deps: CLAN-006, DEC-005** — Design and implement inactive-leader handling only after inactivity is precisely defined. **Done:** no automatic transfer relies merely on missing sync data; notice, grace, appeal, and audit paths exist.
- [ ] **CLAN-043 · Next · deps: CLAN-006, CLAN-014** — Add paginated clan audit-log API/UI for authorized roles with event-type filters. **Done:** sensitive metadata is minimized, immutable, and inaccessible to ordinary members unless approved.
- [ ] **CLAN-044 · MVP · deps: CLAN-018, CLAN-022–CLAN-043** — Build discovery, application/invite inbox, roster-management, and membership-state UI. **Done:** pending/expired/full/banned/cooldown/race outcomes are understandable and accessible.
- [ ] **CLAN-045 · MVP · deps: CLAN-016–CLAN-044, DEC-021** — Add model-based and concurrency tests for the membership state machine. **Done:** create/join/approve/invite/leave/kick/ban/transfer/disband interleavings preserve approved membership cardinality, one-leader, capacity, and audit invariants.

## 3. Profiles, friends, blocking, and privacy

- [ ] **CLAN-046 · Next · deps: CLAN-007, CLAN-012** — Implement friend request creation with canonical pair lookup, rate limiting, privacy settings, and no existence leak for blocked/private targets. **Done:** reverse pending requests can atomically become accepted if approved by product.
- [ ] **CLAN-047 · Next · deps: CLAN-046** — Implement accept and decline. **Done:** acceptance creates one friendship state, terminal retries are idempotent, and both users receive deduplicated notifications.
- [ ] **CLAN-048 · Next · deps: CLAN-046** — Implement outgoing request cancel and request expiry/cleanup. **Done:** cursor lists stay consistent and expiry is bounded, scheduled, and silent unless notification policy says otherwise.
- [ ] **CLAN-049 · Next · deps: CLAN-047** — Implement friend removal. **Done:** it is asymmetric in intent but removes the shared relationship once, does not reveal blocks, and leaves audit/minimal abuse evidence per policy.
- [ ] **CLAN-050 · Next · deps: CLAN-046–CLAN-049** — Build incoming/outgoing/friends UI with accessible actions, search, pagination, and pending counts. **Done:** optimistic UI handles races and privacy-safe error responses.
- [ ] **CLAN-051 · Next · deps: DEC-007** — Define per-surface privacy for profile aggregates, clan badge/membership, friends list, friend requests, online/presence, and leaderboard inclusion. **Done:** defaults, precedence, viewer relationship evaluation, and migration/backfill are approved.
- [ ] **CLAN-052 · Next · deps: CLAN-007, DEC-008** — Implement user blocking as a single idempotent operation that resolves existing friendship/requests and suppresses future interaction. **Done:** block state is never disclosed directly to the blocked user.
- [ ] **CLAN-053 · Next · deps: CLAN-052** — Apply blocks consistently to search/discovery, requests, clan invites, chat rendering/mentions, notifications, and reports. **Done:** a shared relationship policy has cross-domain tests and no handler-specific gaps.
- [ ] **CLAN-054 · Next · deps: CLAN-051, CLAN-053** — Extend public profile responses/UI with clan identity and relationship actions only when viewer policy allows. **Done:** existing anonymous profile URLs and aggregate fields remain backward compatible.
- [ ] **CLAN-055 · Next · deps: CLAN-047, CLAN-051, CLAN-123** — Add friends leaderboard endpoints/UI for approved windows/metrics. **Done:** only mutual visible friends and the viewer are eligible; blocks/shadow bans/privacy are applied before ranking.
- [ ] **CLAN-056 · Next · deps: CLAN-046–CLAN-055** — Add friendship/privacy integration tests. **Done:** anonymous, self, friend, clanmate, stranger, blocked, shadow-banned, and admin viewers have explicit expected results.

## 4. Clan home, roster, stats, activity, and announcements

- [ ] **CLAN-057 · MVP · deps: CLAN-021, CLAN-051** — Define clan-home response slices for identity, viewer membership/capabilities, recruitment, aggregate stats, roster preview, announcements, activity, and live/upcoming war. **Done:** public and member-only fields are explicitly separated.
- [ ] **CLAN-058 · MVP · deps: CLAN-057** — Implement paginated roster with role, join date, approved aggregate contribution, last aggregate activity band, and stable sort/filter options. **Done:** exact private last-seen/device details are not exposed.
- [ ] **CLAN-059 · MVP · deps: CLAN-057, canonical usage rules** — Implement clan aggregate totals for approved windows and metrics by joining current/frozen membership attribution to canonical visible usage. **Done:** no raw logs, repricing, duplicate device totals, or date parsing is introduced.
- [ ] **CLAN-060 · MVP · deps: CLAN-059** — Implement member contribution breakdowns with eligibility/privacy rules and explicit denominator. **Done:** totals reconcile to the clan aggregate and hidden members are handled without misleading percentages.
- [ ] **CLAN-061 · Next · deps: CLAN-006, CLAN-057** — Create a bounded clan activity feed from domain events such as joins, promotions, announcements, milestones, and war results. **Done:** chat messages and raw usage events are not copied into the feed.
- [ ] **CLAN-062 · MVP · deps: CLAN-011, CLAN-014** — Implement announcement create/edit/archive with pinned ordering and bounded plain-text/link content. **Done:** permissions, audit, revision metadata, notification choice, and safe link rendering are tested.
- [ ] **CLAN-063 · MVP · deps: CLAN-057–CLAN-062** — Build responsive clan-home UI with roster, stats, contributions, activity, announcements, recruitment CTA, and management actions. **Done:** public/member/leader empty and error states are accessible.
- [ ] **CLAN-064 · Next · deps: CLAN-059** — Add cache policy and invalidation for clan-home aggregates. **Done:** membership/usage/war changes have bounded staleness, cache keys include visibility inputs, and correctness does not depend on cache success.
- [ ] **CLAN-065 · MVP · deps: CLAN-057–CLAN-064** — Add reconciliation tests across clan totals, member contributions, public visibility, membership changes, and current global/profile aggregates. **Done:** fixture totals match exactly for multi-device and boundary-day cases.

## 5. Clan chat

- [ ] **CLAN-066 · Next · deps: CLAN-008, CLAN-014, DEC-018** — Define chat contracts and state model for messages, revisions, tombstones, replies, reactions, pins, mention entities, cursors, unread state, and delivery acknowledgements. **Done:** payload sizes and page limits are bounded and typed.
- [ ] **CLAN-067 · Next · deps: CLAN-066, DEC-009** — Prototype Durable Object hibernating WebSockets versus polling/SSE using existing session auth and D1 history. **Done:** a decision note compares ordering, reconnect/resume, fanout, local testability, cost, failure recovery, and deployment/migration impact; no transport is selected before review.
- [ ] **CLAN-068 · Next · deps: CLAN-066** — Implement D1 chat-history cursor pagination with newest/older traversal and deterministic `(createdAt,id)` cursor. **Done:** no offset pagination, duplicates, or gaps under concurrent sends.
- [ ] **CLAN-069 · Next · deps: CLAN-066, CLAN-014** — Implement message sending with membership recheck, content/size limits, idempotency key, rate limit, and stored server timestamp. **Done:** retries create one message and former/banned members cannot send.
- [ ] **CLAN-070 · Next · deps: CLAN-069, DEC-018** — Implement author edit within policy and preserve bounded revision metadata. **Done:** edited state is visible, mentions are re-evaluated safely, and edits cannot re-notify abusively.
- [ ] **CLAN-071 · Next · deps: CLAN-069, DEC-018** — Implement self-delete and moderator delete with tombstones. **Done:** replies remain structurally valid, deleted content is not delivered, and moderator action is audited.
- [ ] **CLAN-072 · Next · deps: CLAN-069** — Implement replies with one-level reference and deleted/missing-parent rendering. **Done:** cross-clan references are impossible and reply previews obey block/moderation policy.
- [ ] **CLAN-073 · Next · deps: CLAN-066, CLAN-069, DEC-019** — Parse/store member, role, and `@all` mentions as validated entities rather than raw display-name matches. **Done:** renamed users/roles render correctly and unauthorized/broad mentions are rejected or downgraded explicitly.
- [ ] **CLAN-074 · Next · deps: CLAN-073, CLAN-140** — Fan out mention notifications with dedupe, block/privacy/preference checks, and broad-mention caps. **Done:** sender retry/edit does not multiply notifications and fanout avoids unbounded D1 binds.
- [ ] **CLAN-075 · Next · deps: CLAN-069** — Implement reactions with an approved bounded emoji set and toggle semantics. **Done:** one reaction per user/emoji/message, aggregate counts are deterministic, and removed members' behavior is defined.
- [ ] **CLAN-076 · Next · deps: CLAN-069, CLAN-014** — Implement pin/unpin and pinned-message list. **Done:** permission matrix, maximum pins, tombstones, audit, and stable ordering are enforced.
- [ ] **CLAN-077 · Next · deps: CLAN-067–CLAN-069** — Implement selected live delivery transport with authenticated connection bootstrap and per-clan isolation. **Done:** connection authorization is rechecked on membership/ban changes and D1 remains the durable source of truth.
- [ ] **CLAN-078 · Next · deps: CLAN-077** — Implement delivery sequence/ack and reconnect resume. **Done:** clients can recover missed messages from D1 after disconnect, duplicate frames are harmless, and no in-memory state is required for correctness.
- [ ] **CLAN-079 · Next · deps: CLAN-077** — Implement bounded presence/typing indicators only if approved. **Done:** indicators are ephemeral, preference-aware, rate-limited, and never treated as durable last-seen evidence.
- [ ] **CLAN-080 · Next · deps: CLAN-068, CLAN-078** — Implement per-user read cursor and unread/mention counts. **Done:** read updates are monotonic/idempotent and work across devices without marking unseen messages read.
- [ ] **CLAN-081 · Next · deps: CLAN-068–CLAN-080** — Build accessible virtualized chat UI with history loading, live insertions, reconnect status, composer, reply/edit/delete, reactions, pins, unread jump, and reduced-motion support. **Done:** keyboard navigation, focus restoration, screen-reader announcements, and high-contrast states are tested.
- [ ] **CLAN-082 · Next · deps: CLAN-069, CLAN-147** — Add layered chat rate limits for sends, edits, reactions, mentions, connection attempts, and history reads. **Done:** rules have unique namespaces/keys, typed retry hints, and fail behavior appropriate to abuse risk.
- [ ] **CLAN-083 · Next · deps: CLAN-053, CLAN-068** — Apply user blocks in a shared chat policy. **Done:** blocked content/mentions/notifications follow `DEC-008`, moderation evidence remains available to authorized staff, and message counts do not leak hidden content.
- [ ] **CLAN-084 · Next · deps: CLAN-011, CLAN-069, CLAN-071** — Implement report-message flow with reason, message/revision reference, reporter context, and anti-spam controls. **Done:** reports retain minimum necessary evidence and cannot be used to fetch unrelated/deleted content.
- [ ] **CLAN-085 · Next · deps: CLAN-066–CLAN-084** — Add chat fault/concurrency tests. **Done:** send retries, out-of-order delivery, reconnect, membership revocation, moderation, hibernation/restart, pagination races, blocks, rate limits, and retention have deterministic coverage.

## 6. Clan wars

- [ ] **CLAN-086 · Next · deps: CLAN-002, CLAN-009, DEC-010–DEC-015** — Finalize and document the war lifecycle (`draft`, `challenged`/`queued`, `accepted`, `prep`, `live`, `provisional`, `final`, `cancelled`/`forfeit`). **Done:** every transition has actor, preconditions, timeout, idempotency behavior, and repair route.
- [ ] **CLAN-087 · Next · deps: CLAN-086, DEC-010** — Define score specification using canonical stored aggregates. **Done:** metric, included token fields, spend display, date bounds, rounding, future-day ceiling, visibility/moderation exclusions, and formula version are explicit.
- [ ] **CLAN-088 · Next · deps: CLAN-086, CLAN-014** — Implement per-member war opt-in/opt-out and availability window. **Done:** consent is explicit, revocable until roster lock, and leaders cannot silently enroll members.
- [ ] **CLAN-089 · Next · deps: CLAN-088, DEC-011** — Implement roster proposal and validation. **Done:** membership, opt-in, roster size, duplicates, bans, live-war overlap, and eligibility are rechecked server-side.
- [ ] **CLAN-090 · Next · deps: CLAN-089** — Implement roster lock as an immutable snapshot of user IDs, clan IDs, membership IDs, score formula version, and baseline boundaries. **Done:** later membership/role changes cannot mutate the snapshot.
- [ ] **CLAN-091 · Next · deps: CLAN-086, CLAN-089** — Implement direct challenge create/cancel/expire/decline. **Done:** target visibility/preferences, cooldowns, duplicate challenges, rate limits, roster compatibility, and notifications are covered.
- [ ] **CLAN-092 · Later · deps: CLAN-089, DEC-012** — Implement matchmaking queue entry/exit with explicit fairness inputs and privacy-safe bands. **Done:** queue state is idempotent and no raw usage or individual private budget is exposed.
- [ ] **CLAN-093 · Later · deps: CLAN-092** — Implement deterministic matchmaking candidate selection and explainability. **Done:** test fixtures cover unequal roster size/activity/budget, rematch avoidance, wait-time expansion, and abuse resistance.
- [ ] **CLAN-094 · Next · deps: CLAN-091 or CLAN-093** — Implement challenge/match accept as an atomic transition into preparation. **Done:** both rosters and clans are revalidated and competing accepts cannot create overlapping wars.
- [ ] **CLAN-095 · Next · deps: CLAN-090, CLAN-094** — Implement prep countdown, roster review, and lock deadline. **Done:** timezone display uses instants while score windows use documented day-key boundaries; late withdrawals follow `DEC-011`.
- [ ] **CLAN-096 · Next · deps: CLAN-090, DEC-013** — Implement frozen participant attribution independent of current clan membership. **Done:** leaving, kicking, joining another clan, role changes, and disband behavior have explicit tested outcomes without editing usage ownership.
- [ ] **CLAN-097 · Next · deps: CLAN-086, CLAN-095** — Implement scheduled start transition with retry-safe cron/alarm/worker processing. **Done:** start occurs once, captures start boundary/version, and can recover after missed scheduler execution.
- [ ] **CLAN-098 · Next · deps: CLAN-087, CLAN-090, CLAN-097** — Implement incremental live score query/ledger from canonical `usage_days`. **Done:** repeated syncs and duplicate rows do not double count; D1 query cost is bounded; score reconciles from source aggregates.
- [ ] **CLAN-099 · Next · deps: CLAN-096, CLAN-098** — Implement per-participant contribution and clan totals using the frozen roster. **Done:** contributions reconcile exactly, privacy display is separate from score inclusion, and join/leave cannot transfer points.
- [ ] **CLAN-100 · Next · deps: CLAN-098, DEC-014** — Handle late/offline sync with a provisional result and explicit grace deadline. **Done:** UI distinguishes live/provisional/final and never implies unsynced usage is final.
- [ ] **CLAN-101 · Next · deps: CLAN-087, CLAN-098** — Preserve duplicate/lost-log/correction behavior in score recomputation. **Done:** same-parser lower recount remains retained; a parser bump can lower/drop rows; score recomputation is deterministic and logged.
- [ ] **CLAN-102 · Next · deps: CLAN-101, DEC-014** — Define parser-version changes during live/provisional wars. **Done:** formula/parser version pinning or controlled re-finalization is approved; mixed-version results cannot silently ship.
- [ ] **CLAN-103 · Next · deps: CLAN-100–CLAN-102** — Implement idempotent finalization with score snapshot, winner/draw, finalized timestamp, formula/parser metadata, and source boundary. **Done:** retries produce one result and a full reconciliation can prove it from canonical aggregates.
- [ ] **CLAN-104 · Next · deps: CLAN-087, CLAN-103** — Display token score and, only if approved, frozen API-equivalent spend context without presenting either as code quality/productivity. **Done:** labels/tooltips explain the competition metric and canonical pricing caveat.
- [ ] **CLAN-105 · Next · deps: CLAN-103, DEC-015** — Implement ties using the approved ranking/result rule. **Done:** equal score is a first-class draw unless an approved deterministic tiebreak applies; display ordering does not alter the result.
- [ ] **CLAN-106 · Next · deps: CLAN-098, DEC-012** — Add fairness/imbalance diagnostics for roster size and recent usage bands. **Done:** warnings are explainable and do not expose private individual usage; enforcement versus advisory behavior is explicit.
- [ ] **CLAN-107 · Next · deps: CLAN-098, CLAN-147** — Add war abuse controls for rapid challenge spam, roster cycling, coordinated account/device manipulation signals, implausible usage, and repeated forfeits. **Done:** controls flag/review rather than reinterpret canonical usage silently.
- [ ] **CLAN-108 · Next · deps: CLAN-086, CLAN-097** — Implement pre-start cancellation and expiry reasons. **Done:** actor/state rules, notifications, cooldowns, and audit events are deterministic and no rewards/record inflation occurs.
- [ ] **CLAN-109 · Next · deps: CLAN-097, DEC-015** — Implement live forfeit/admin cancellation/recovery. **Done:** outcome and reward policy are explicit; admin mutation requires reason, correlation ID, and audit.
- [ ] **CLAN-110 · Next · deps: CLAN-103** — Add paginated war history/detail/results APIs and clan/profile UI. **Done:** immutable final score, roster snapshot, contributions, formula version, draw/cancel state, and privacy rules are represented.
- [ ] **CLAN-111 · Next · deps: CLAN-103, CLAN-114, DEC-015** — Grant war XP/rewards through idempotent ledger keys after finalization. **Done:** re-finalization/repair cannot duplicate grants and revocation/correction policy is explicit.
- [ ] **CLAN-112 · Next · deps: CLAN-086–CLAN-111** — Build challenge, queue, prep, live, provisional, result, and history UI. **Done:** countdowns, sync grace, fairness warnings, opt-in, cancellation, and accessible live updates are clear.
- [ ] **CLAN-113 · Next · deps: CLAN-086–CLAN-112** — Add end-to-end war fixtures and property/concurrency tests. **Done:** multi-device duplicates, delayed sync, same-parser loss, parser bump, cost freeze, timezone edges, membership churn, ties, cancellation, and retry recovery are covered.

## 7. Clan progression and cooperative goals

- [ ] **CLAN-114 · Next · deps: CLAN-010, DEC-016** — Define versioned XP sources, caps, anti-farming rules, level curve, and lifetime/seasonal separation. **Done:** a worked fixture explains every XP point and avoids rewarding raw spend without approved safeguards.
- [ ] **CLAN-115 · Next · deps: CLAN-114** — Implement append-only XP ledger and derived clan total/level. **Done:** unique source keys make replay idempotent and a reconciliation job can rebuild snapshots.
- [ ] **CLAN-116 · Later · deps: CLAN-115, DEC-002** — Add level milestone benefits without pay-to-win or hidden ranking boosts. **Done:** benefits are display/cosmetic or explicitly approved capacity changes and are reversible/configurable.
- [ ] **CLAN-117 · Next · deps: CLAN-115** — Implement milestone and achievement definitions with versioned criteria. **Done:** grants are idempotent, historical criteria changes do not silently revoke earned display rewards, and visibility is defined.
- [ ] **CLAN-118 · Later · deps: CLAN-115, DEC-016** — Implement cooperative clan goals/clan games over canonical aggregate contributions. **Done:** goal window, eligibility, frozen/current membership attribution, caps, corrections, and completion finalization are explicit.
- [ ] **CLAN-119 · Later · deps: CLAN-118** — Add individual contribution view and cooperative progress UI. **Done:** totals reconcile, privacy is respected, and copy emphasizes cooperation rather than productivity judgment.
- [ ] **CLAN-120 · Next · deps: CLAN-115, CLAN-117** — Implement display-only clan cosmetics/reward unlocks. **Done:** no inventory, trading, purchase, currency, or power system is introduced.
- [ ] **CLAN-121 · Next · deps: CLAN-111, CLAN-115–CLAN-120** — Add XP/reward repair and admin reconciliation tools. **Done:** dry-run diff, reason, actor, before/after, idempotency, and audit are required.
- [ ] **CLAN-122 · Next · deps: CLAN-114–CLAN-121** — Add progression tests for replay, concurrent grants, correction, cap boundaries, level transitions, seasonal/lifetime separation, and revoked/cancelled sources. **Done:** ledger and derived snapshots always reconcile.

## 8. Leaderboards

- [ ] **CLAN-123 · MVP · deps: CLAN-012, CLAN-059** — Define a shared leaderboard query contract for scope (`global`, `clans`, `clan-members`, `friends`), metric, window, filters, cursor, and viewer context. **Done:** existing global endpoint remains compatible or receives a separately reviewed additive path.
- [ ] **CLAN-124 · MVP · deps: CLAN-123** — Preserve existing global `7d|30d|all` and `spend|tokens` behavior while extracting reusable ranking primitives. **Done:** current fixtures/ranks do not change without an explicit migration decision.
- [ ] **CLAN-125 · MVP · deps: CLAN-123, DEC-017** — Implement approved tie/rank semantics per leaderboard. **Done:** rank, stable display order, cursor, and equal-score examples are specified and tested; ties are not accidentally broken into different competitive outcomes.
- [ ] **CLAN-126 · MVP · deps: CLAN-059, CLAN-125** — Implement clan leaderboard over canonical visible usage and approved current/frozen membership rule. **Done:** aggregate totals reconcile to clan home and empty/ineligible clans are handled explicitly.
- [ ] **CLAN-127 · MVP · deps: CLAN-058, CLAN-125** — Implement clan-member leaderboard. **Done:** one clan scope, member privacy, shadow bans, zero/inactive members, roles, and stable pagination are covered.
- [ ] **CLAN-128 · Next · deps: CLAN-055, CLAN-125** — Implement friends leaderboard. **Done:** relationship/block/privacy changes invalidate eligibility and no non-friend position leaks through total/rank metadata.
- [ ] **CLAN-129 · Next · deps: CLAN-123** — Add approved filters such as source/agent and region/language only where data is canonical and safe. **Done:** filters do not expose device/raw-log details and query indexes are measured.
- [ ] **CLAN-130 · MVP · deps: CLAN-125–CLAN-129, DEC-017** — Enforce eligibility for inactive users, inactive/empty clans, minimum member/activity thresholds, shadow bans, and disbanded clans. **Done:** eligibility reasons are versioned, testable, and explainable in UI.
- [ ] **CLAN-131 · Next · deps: CLAN-126, CLAN-134** — Store leaderboard history/snapshots only for approved season or milestone needs. **Done:** snapshot cadence, correction policy, retention, and backfill are explicit; current queries remain canonical.
- [ ] **CLAN-132 · MVP · deps: CLAN-123–CLAN-130** — Add cache keys, bounded TTL, and invalidation strategy by scope/window/metric/filter/visibility. **Done:** cache failure degrades to correct reads and membership/privacy changes cannot serve unauthorized cached data.
- [ ] **CLAN-133 · MVP · deps: CLAN-123–CLAN-132** — Build unified accessible leaderboard UI with scope/window/metric filters, tie display, eligibility explanation, current-viewer row, empty states, and canonical profile/clan links. **Done:** URL state is shareable and existing global views remain functional.

## 9. Seasons, leagues, matchmaking, and rivalries

- [ ] **CLAN-134 · Later · deps: stable core metrics, DEC-020** — Define season boundaries, timezone/day-key interaction, immutable season IDs, grace/finalization, and reset versus lifetime fields. **Done:** season changes never mutate lifetime usage or clan membership history.
- [ ] **CLAN-135 · Later · deps: CLAN-134, DEC-017** — Define league/division placement and movement using transparent clan-level metrics. **Done:** promotion/relegation, inactivity, new clans, ties, and corrections are specified without hidden productivity judgments.
- [ ] **CLAN-136 · Later · deps: CLAN-092, CLAN-134** — Add season-aware matchmaking bands only after base matchmaking is measured. **Done:** queue expansion and fairness inputs are versioned and explainable.
- [ ] **CLAN-137 · Later · deps: CLAN-110, CLAN-134** — Add opt-in rivalry records derived from repeated matches or mutual acceptance. **Done:** rivalry never bypasses block/challenge preferences and can be ended without penalty.
- [ ] **CLAN-138 · Later · deps: CLAN-120, CLAN-134** — Add seasonal display-only rewards with lifetime archive. **Done:** grants are idempotent, no inventory/currency exists, and late corrections have a published policy.
- [ ] **CLAN-139 · Later · deps: CLAN-134–CLAN-138** — Run a season simulation/backfill on synthetic fixtures before launch. **Done:** boundaries, corrections, league movement, rewards, and rollback/re-run reconcile deterministically.

## 10. Notifications, security, moderation, operations, tests, and rollout

- [ ] **CLAN-140 · MVP · deps: CLAN-011** — Define notification taxonomy and dedupe keys for requests, invites, role changes, announcements, mentions, moderation, wars, goals, and rewards. **Done:** every producer has one stable key and bounded payload with canonical destination.
- [ ] **CLAN-141 · MVP · deps: CLAN-140** — Implement in-app notification list, unread cursor/count, mark-read, pagination, and deep links. **Done:** multi-device updates are monotonic/idempotent and deleted/private targets degrade safely.
- [ ] **CLAN-142 · Next · deps: CLAN-140, CLAN-051** — Implement granular preferences by category/channel plus broad-mention and war reminders. **Done:** security/moderation notices remain mandatory where required; defaults and migration are documented.
- [ ] **CLAN-143 · Later · deps: CLAN-140–CLAN-142** — Evaluate email/push delivery separately; do not add it by implication. **Done:** provider, consent, unsubscribe, privacy, retries, cost, and secret handling receive an approved design before implementation.
- [ ] **CLAN-144 · MVP · deps: CLAN-014, CLAN-051, CLAN-053** — Perform a cross-domain authorization/privacy threat model. **Done:** IDOR, enumeration, role escalation, stale membership, membership-cardinality bypass, block bypass, cached private data, CSRF/session, and WebSocket-auth threats map to tests/controls.
- [ ] **CLAN-145 · MVP · deps: CLAN-144** — Add input/output security controls for names, links, announcements, chat, reports, search, and cursor IDs. **Done:** content is rendered as safe text/approved links, payloads are bounded, and no secrets/raw logs enter audit or telemetry.
- [ ] **CLAN-146 · MVP · deps: CLAN-144** — Reuse existing session, OAuth, CLI token, admin-email, and recent-auth patterns; document any sensitive-action re-auth requirement. **Done:** no new auth provider/store is introduced and CLI tokens do not gain browser/clan mutation powers accidentally.
- [ ] **CLAN-147 · MVP · deps: domain endpoint definitions** — Expand rate-limit rules for high-abuse unauthenticated/authenticated endpoints with appropriate keys and fail-open/fail-closed choices. **Done:** rules are tested, observable, namespaced uniquely, and do not treat rate limiting as authorization.
- [ ] **CLAN-148 · MVP · deps: CLAN-011, CLAN-144** — Implement user/clan/message/war reporting and admin review queue with reason taxonomy, status, assignment, action, appeal/reference, and audit. **Done:** access is admin-gated and evidence is minimized/retained per policy.
- [ ] **CLAN-149 · MVP · deps: CLAN-148** — Implement moderation actions for clan visibility, recruitment/chat freeze, message removal, member/clan suspension, and war/result hold. **Done:** actions are scoped, reversible where possible, typed, audited, and compose with existing shadow bans.
- [ ] **CLAN-150 · MVP · deps: CLAN-006, CLAN-009–CLAN-011** — Add operational repair/reconciliation commands for membership invariants, leaderlessness, capacity, chat cursors, war scores/finalization, XP/rewards, and notification fanout. **Done:** dry-run is default, writes require explicit target/reason, and repairs are idempotent/audited.
- [ ] **CLAN-151 · MVP · deps: all domain services** — Add structured observability for transition success/failure, typed errors, latency, D1 rows read/written, cache hit/miss, rate limits, live connections/reconnects, fanout lag, war finalization, and repair drift. **Done:** logs contain IDs/counters but no chat bodies, raw usage, tokens, session cookies, or private notes.
- [ ] **CLAN-152 · MVP · deps: CLAN-008, CLAN-011, DEC-006, DEC-018** — Implement bounded retention/cleanup for expired invites/requests/transfers/bans, chat revisions/tombstones, notifications, reports, and ephemeral transport state. **Done:** the existing cleanup schedule or a reviewed alternative processes chunks safely and records failures.
- [ ] **CLAN-153 · MVP · deps: schema/query tasks** — Add migration, repository, service, HTTP contract, policy, UI, accessibility, and compatibility tests proportional to each domain. **Done:** tests capture expected logs, keep `bun run test` clean, and never depend on external production state.
- [ ] **CLAN-154 · MVP · deps: CLAN-045, CLAN-065, CLAN-085, CLAN-113, CLAN-122, CLAN-133, CLAN-153** — Build a cross-feature scenario suite. **Done:** account merge/deletion, shadow ban, block, membership-cardinality race, capacity race, delayed sync, parser correction, timezone edge, disband, and admin recovery preserve all approved invariants.
- [ ] **CLAN-155 · MVP · deps: CLAN-151–CLAN-154** — Roll out with server-side feature flags, internal clan fixtures, staged cohorts, migration backup/recovery plan, dashboards/alerts, and explicit rollback criteria. **Done:** schema remains backward compatible during rollback, write paths can be disabled independently, and launch review signs off on abuse/support runbooks.

## 11. Optional later ideas

- [ ] **CLAN-156 · Later · deps: stable clan home/progression** — Explore a shared clan HQ as a themed dashboard composition, not a second social network. **Done:** proposal reuses existing stats/announcements/goals and adds no inventory/economy.
- [ ] **CLAN-157 · Later · deps: CLAN-118** — Explore cooperative “raids” as time-boxed aggregate goals with non-combat naming alternatives. **Done:** proposal measures token participation only, uses canonical aggregates, and avoids boss damage/productivity metaphors unless explicitly approved.
- [ ] **CLAN-158 · Later · deps: CLAN-117** — Expand display achievements for participation, consistency, cooperation, and community milestones. **Done:** criteria are transparent, idempotent, privacy-safe, and not code-quality claims.
- [ ] **CLAN-159 · Later · deps: CLAN-120** — Expand cosmetic clan themes/badges/banners earned through approved milestones. **Done:** assets are accessible, moderated, display-only, and have no inventory/trading/payment layer.
- [ ] **CLAN-160 · Later · deps: CLAN-118, CLAN-134** — Add rotating cooperative challenges after correction/finalization behavior is proven. **Done:** schedules, caps, eligibility, replay, and late sync rules are versioned and tested.
- [ ] **CLAN-161 · Later · deps: abuse review and explicit approval** — Evaluate referrals only as attribution/recognition, not currency or growth spam. **Done:** consent, fraud controls, privacy, reward limits, and no-monetization boundary are approved before any schema or UI work.

## Definition of complete for an implementation task

A checked task should include, where applicable:

- approved dependent decisions and unchanged confirmed constraints;
- additive Drizzle schema/migration with forward and representative existing-data tests;
- shared API contract, typed errors, Effect service/repository implementation, and policy enforcement;
- accessible frontend states with privacy-safe error handling;
- idempotency/concurrency behavior and D1 100-bind-safe queries;
- focused unit/integration/compatibility tests with clean logs;
- observability without raw logs, secrets, chat bodies, or unnecessary personal data;
- documentation of migration, rollback, repair, and feature-flag behavior;
- reconciliation against canonical usage/date/parser/cost rules when aggregates are involved.

If a task would require changing a confirmed constraint—especially usage ownership, date bucketing, parser correction, cost freeze, CLI compatibility, or auth/database stack—stop and add/resolve a decision instead of implementing the conflict silently.
