# Sakpase Implementation Plan

## Product target

Build Sakpase, a Haiti-first, mobile-first messaging and social networking app for people aged 16 and older. The first release targets iOS, Android, and web. Its core success measure is that a new user completes onboarding and exchanges a message within 24 hours.

The app has four destinations: Home, Messages, Explore, and Profile. Use native tabs on iOS and Android; use responsive web navigation (bottom navigation on narrow screens and a side rail on wide screens). The visual style is clean and accessible, using NativeWind. Support Haitian Creole, French, and English, selecting the device language when supported and English otherwise; allow a manual language choice. Follow the system light/dark theme by default and provide a manual override.

### V1 scope

- Clerk authentication with Google, Apple, and optional phone/SMS sign-in; account linking only after the user explicitly verifies and links another method.
- Usernames and profiles, search, opt-in contact matching, follow requests for private accounts, blocking, and reporting.
- 24-hour photo/video statuses with captions, audience controls, likes, comments, viewer lists, and sharing into chats.
- One-to-one and group messaging with text, image/video, voice-note and file attachments, emoji reactions, replies, delivery/read state, search, and push notifications.
- Offline reading of recently cached conversations and a retryable local outbox for text messages.
- Account deletion, report handling, moderator tools, and operational monitoring.

### Explicitly out of scope for V1

End-to-end encryption, voice/video calls, message editing, payments, recommendation systems beyond simple discovery, and advanced group administration. Do not describe messages as end-to-end encrypted. Keep E2EE as a separately designed future milestone because V1 server-side message search requires readable message content on the backend.

## Decisions and assumptions to preserve

- **ASSUMPTION — Public profiles:** profiles are public by default; users may switch to private, which requires approval for new followers. Phone numbers remain private unless explicitly shared.
- **ASSUMPTION — Age gate:** collect a self-attestation that the user is at least 16; do not collect or retain date of birth.
- Usernames are unique case-insensitively, ASCII letters/numbers/underscores only, 3–24 characters. Display names may use accents. Require username and display name after authentication; avatar and bio are optional.
- Contact matching is opt-in and mobile-only. Ask for permission just in time, normalize numbers, send them securely for exact matching, and discard the submitted address book after matching. Username search works on all platforms. Invites use the platform share sheet and a deep link; do not send SMS automatically.
- A user can message someone if either account follows the other or the recipient matches the sender's contacts. One-way follow eligibility routes the first message to requests; contact matches go to the inbox. **ASSUMPTION:** accepting a request opens the ordinary chat; declining prevents further messages until the recipient allows them.
- Public follows take effect immediately; private-account follows require approval. No duplicate or self-follow relationships.
- Status audience defaults to followers. A status may target eligible users, followers/connections, selected contacts, or exclude selected users. Blocks and private-account rules always take precedence. Sharing a status sends a permission-checked reference; it becomes unavailable when its source expires or is deleted.
- Status viewer names are visible only to the author while the status is active and read receipts are enabled. Presence, last seen, and read receipts default on and have separate privacy controls.
- **ASSUMPTION — Receipt semantics:** delivered means a recipient client acknowledged the message; read means it was viewed while read receipts are enabled. Define exact UI labels consistently across direct and group chats.
- Blocking prevents DMs, follows, status visibility, and social interactions. Existing shared group membership remains; blocked users' direct interactions are restricted.
- Direct-message deletion for me is available at any time. Delete for everyone is available for one hour and leaves a removal marker. Group creators start as admins; admins can manage settings and membership; anyone may leave. Groups are limited to 256 participants. New members see messages from after they joined.
- Group read receipts show delivered/read counts in the thread; named readers appear in message details when receipts are enabled.
- Incoming push content is generic by default. Respect mute, notification preferences, blocks, and user settings. iOS/Android receive push; web uses in-app notifications only.
- Statuses stop appearing in active feeds after 24 hours; their media is purged after 30 days. Report evidence selected by a user is retained through case resolution and 30 days afterward. Preserve a separate evidence copy if normal status cleanup would otherwise remove active-case evidence.
- Deleting an account immediately removes its profile, statuses/media, and relationships. Sent messages remain visible to recipients with the sender shown as a deleted account. Evidence associated with an active report is retained under the report policy.
- **ASSUMPTION — Initial scale:** private beta below 1,000 monthly active users. Do a cost and capacity review before public launch.
- No fixed monthly service budget has been supplied. Set cost alerts and review usage before launch; preserve core text messaging if a service approaches a soft cost threshold and constrain costly media operations only when required by a provider hard limit.

## Architecture and technology

- Keep the specified stack: Expo/React Native, Expo Router, NativeWind, Clerk, and Convex. Current repo versions observed: Expo SDK 57 (`~57.0.26`), Expo Router (`~57.0.24`), React Native (`0.86.3`), React (`19.2.3`), and TypeScript (`~6.0.3`). Recheck installed and compatible versions before adding packages and preserve the npm lockfile.
- Use NativeWind 4.2.7 (the documented Expo SDK 57 compatible line); avoid NativeWind 5 while it is pre-release. Use Expo Router SDK 57 native tabs via `expo-router/unstable-native-tabs` on native; isolate the unstable API behind the tab layout so migration is contained. Use a platform-specific responsive web tab layout.
- Use Clerk (`@clerk/expo`, currently observed `4.8.1`) for authentication and identity, and Convex (`1.46.0`) for application data, queries, mutations, actions, and real-time subscriptions. Integrate Convex with Clerk using the supported Clerk Expo provider. Clerk is the source of truth for identity and sign-in methods; Convex stores the app profile keyed by Clerk subject. Every sensitive Convex operation performs server-side authorization.
- Use Clerk-managed Google, Apple, and SMS authentication. Implement custom branded sign-in/sign-up screens. Native provider sign-in and remote push need development builds; use EAS development builds rather than relying on Expo Go for these paths.
- Use Cloudflare R2 for media objects and Convex for media metadata and authorization. Authorize uploads server-side and deliver private objects via short-lived signed access. Optimize on device; validate file type and size at upload. No server transcoding in V1.
- Use Expo Notifications and Expo Push Service for native pushes. Use Cloudflare Pages for the static web output. Use Sentry plus Convex diagnostics for crash/error visibility; never log message bodies, credentials, tokens, or unnecessary personal data.
- Create isolated development, staging, and production Clerk and Convex environments, with separate app configuration and secrets. Initial Convex region: US East (N. Virginia); Convex region changes require migration, so confirm before production data is created.
- Existing app configuration identifies Sakpase, scheme `sakpase`, iOS bundle ID `com.kreyotek.sakpase`, and web static output. Preserve unrelated worktree changes while implementing.

### Media limits

| Media | Limit |
| --- | ---: |
| Profile/status image | 10 MB |
| Status video | 50 MB and 30 seconds |
| Voice note | 20 MB |
| Other file | 25 MB |

## Core data model

All app records are owned by a Clerk subject ID. Add indexes for lookup, pagination, uniqueness validation, and expiry cleanup. Enforce uniqueness and authorization in Convex mutations; do not assume database indexes alone enforce application rules.

- **UserProfile:** `clerkSubject`, normalized unique `username`, display name, avatar media ID, bio, optional public phone visibility, profile privacy, status/presence/read-receipt preferences, locale, theme, notification settings, created/updated timestamps, account state.
- **Follow:** follower ID, followed ID, state (`pending`/`accepted`), created timestamp. Unique follower/followed pair.
- **Block:** blocker ID, blocked ID, timestamp. Unique pair; checked by every relevant read and write.
- **ContactMatch:** do not persist raw address books. Persist only minimal, consented relationship/match data if required for messaging eligibility, with revocation and retention behavior defined before implementation.
- **Status:** author ID, media ID, caption, audience configuration, created/expiry/purge timestamps, state. Use a separate audience/exception representation if needed to avoid unbounded status documents.
- **StatusLike:** status ID, user ID, timestamp; unique status/user pair.
- **StatusComment:** status ID, author ID, body, timestamps, deletion state.
- **StatusView:** status ID, viewer ID, viewed timestamp; visible to author only under the agreed active-status/read-receipt rule.
- **Conversation:** kind (`direct`/`group`), participant or group metadata, creator, last-message summary, timestamps, settings. Direct conversations have a canonical participant key to prevent duplicates.
- **Membership:** conversation ID, user ID, role, joined/left timestamps, notification settings, last-read cursor. A joined-at boundary controls group history visibility.
- **Message:** conversation ID, sender ID, type, text or media reference, optional reply target/status reference, client idempotency key, server sequence/time, deletion state, timestamps. Enforce membership and visibility before returning any history.
- **Reaction:** message ID, user ID, emoji, timestamp; unique message/user/emoji tuple.
- **Delivery/read state:** per-user message cursor or receipt record, bounded to the product's receipt semantics and privacy settings; avoid unbounded per-message fanout where cursors suffice.
- **Notification:** recipient, event/reference, type, read/dedupe key, created timestamp. Do not store sensitive message content in push payloads.
- **Report:** reporter, target type/ID, selected evidence snapshot/reference, explanation, status, moderator/audit fields, retention timestamps. Restrict reads to the reporter's allowed status and authorized moderators.
- **MediaObject:** owner, storage key, content type, byte size, access classification, linked entity, lifecycle state, created/deletion timestamps. Remove orphaned objects through retryable cleanup.

## Primary user journeys

1. **First launch and account setup:** select language/theme from defaults → age attestation → Clerk sign-in/sign-up → create username and display name → optional avatar/bio, contact discovery, and notification permission prompts → Home with an obvious search/start-chat action. Never request contacts or push permission before explaining their value.
2. **Find and follow someone:** username search or optional contact match → privacy-filtered result → open profile → follow immediately or request approval → reflect pending/accepted state and update counts in real time.
3. **Start a conversation:** choose eligible user or group → show request/inbox state based on relationship → compose text/media → optimistic pending state keyed by client idempotency key → Convex real-time confirmation and delivery/read indicators → retry safely after connection failure.
4. **Create/view status:** choose or record media → preview, caption, audience selection → upload with progress and retry → publish → Explore shows eligible active statuses → view, react/comment, inspect permitted viewer list, or share reference to chat → expiration and later media cleanup.
5. **Create/manage group:** select participants → name/photo → creator becomes admin → manage membership/settings → participants see messages only from their join time → leave or admin removal updates access immediately.
6. **Safety and account controls:** block/report from profile, status, comment, or message → capture only selected evidence and explanation → moderator queue with audited access → settings for presence, receipts, notifications, privacy → confirm account deletion and apply retention exceptions.

## Implementation phases

### Phase 0 — Repository and provider foundations

- Inspect the current Expo app/configuration and preserve existing uncommitted changes. Confirm package versions and obtain current versioned Expo, Clerk, Convex, and NativeWind documentation before touching their APIs.
- Establish app-wide route/layout structure, native tabs, responsive web navigation, NativeWind setup, design tokens, translation resources, theme selection, and accessible component patterns.
- Create dev/staging/prod Clerk and Convex projects; configure URL schemes, OAuth callbacks, Apple capabilities, environment secrets, and EAS development profiles. Configure web hosting and the production domain later in this phase.
- Add provider error reporting and a safe configuration check that fails clearly when required environment variables are missing.

### Phase 1 — Identity, profiles, and discovery

- Implement branded Clerk auth and verified method linking; add the age attestation and required username/display-name onboarding.
- Implement Convex user provisioning keyed to Clerk subject, profile editing, avatar upload, unique username validation, profile privacy, and account settings.
- Add username search, opt-in contact permission/matching, invitations via share sheet/deep link, follows, private follow requests, block enforcement, and follower/following lists.
- Verify server authorization and prevent profile/phone data leakage in every query and mutation.

### Phase 2 — Status and moderation

- Add R2 upload authorization, client preview/progress/retry, MIME/size/duration checks, media metadata, and cleanup jobs.
- Implement status audiences and consistent visibility checks for feeds, sharing, interactions, notifications, and viewer lists.
- Add expiry filtering at read time and scheduled purge after 30 days; make cleanup idempotent and retryable.
- Add likes, comments, reports, user blocking, moderator-only web queue, audit records, and evidence retention snapshots.

### Phase 3 — Direct and group messaging

- Implement direct-conversation eligibility and message requests, conversation list, paginated history, text messages, replies, reactions, search, and status references.
- Add media, voice-note, and file attachments with agreed size limits and safe delivery.
- Add groups, participant roles, membership changes, 256-member limit, joined-at history boundary, leave/remove behavior, and group receipt details.
- Add idempotent send, unread/read cursors, delivery/read state, optimistic UI reconciliation, local text outbox, cached recent history, and manual retry.
- Ensure every query and action checks current membership, blocks, conversation state, and participant-specific deletion visibility.

### Phase 4 — Push, presence, polish, and release operations

- Implement optional online/last-seen/typing, privacy controls, push registration, event deduplication, mute/preferences, generic default payloads, and in-app web notifications.
- Complete loading/empty/offline/error states, accessibility, translations, dark/light themes, responsive layouts, and account deletion workflow.
- Set up EAS builds and release channels, Cloudflare Pages deployments, environment separation, rollback notes, Sentry alerts, Convex operational dashboards, and cost alerts.
- Run a closed beta below 1,000 MAU; review delivery, latency, crash-free sessions, abuse, cost, and support feedback before public release.

## Security, reliability, and operational rules

- Clerk proves identity; Convex functions authorize every access. Never trust client-supplied actor IDs or treat UI hiding as access control.
- Centralize reusable authorization checks for ownership, membership, follow/privacy eligibility, block state, status audiences, moderator access, and account deletion state.
- Validate all user input and media server-side; rate-limit auth-adjacent, search, upload, message, follow, comment, and report operations. Apply abuse controls without exposing contact lists or private phone numbers.
- Use per-operation idempotency for message submissions, reactions, follow actions, notifications, and cleanup. Make scheduled expiry, media deletion, and push jobs retry-safe.
- Paginate status feeds, search results, follower lists, and message history. Use indexed bounded queries and real-time subscriptions only where live updates materially help (chat, conversation list/unread state, active status interactions).
- Keep recent authorized conversations readable offline. Queue text only; media requires connectivity. Persist pending text locally with stable IDs, reconcile on reconnect, retain failed drafts, and expose retry state.
- Notify with generic content by default, and never include secrets or unnecessary personal data in logs, analytics, or push payloads.
- Region: Convex US East (N. Virginia). Before production, verify provider pricing, SMS delivery/rates for Haiti, R2 costs/retention, push constraints, and projected Convex subscription/storage usage.

## Quality gates and acceptance criteria

- Unit coverage for username normalization/uniqueness, follow and messaging eligibility, privacy/block rules, status visibility/expiry, delete semantics, report retention, and idempotency.
- Convex integration coverage for Clerk identity, authorization boundaries, direct/group membership, status permissions, R2 upload authorization/cleanup, notification deduplication, and unread/read state.
- End-to-end coverage on iOS, Android, and web for onboarding and sign-in, contact permission granted/denied, search/follow requests, message request and acceptance, groups/history boundaries, status upload/view/share/expiry, blocking/reporting, and account deletion.
- Exercise offline/reconnect, duplicate send, slow/failing upload, push denied/muted, expired media, and partial service failures. Verify accessibility, responsive navigation, all three locales, and theme settings.
- Security acceptance: a user cannot read another user's private profile fields, unauthorized conversation/status/media, or reports; moderator actions are role-limited and audited; no message contents or authentication secrets appear in diagnostics.
- Initial service targets: p95 message appearance under 2 seconds on a healthy connection; main-screen data under 2.5 seconds; crash-free sessions at least 99.5%. Validate at the beta scale, including 256-member group scenarios, before public launch.
- V1 is ready only when the critical journeys pass on all target platforms, authorization and deletion/retention behavior are verified, and monitoring/cost alerts are active.

## Open risks and pre-launch checks

- **E2EE future migration:** server-side plaintext search and message history conflict with E2EE. Before that milestone, decide protocol, device/key management, group membership changes, history transfer, media encryption, backup/recovery, and local search.
- **SMS reach and cost in Haiti:** verify Clerk country availability, deliverability, anti-abuse controls, and current international SMS charges with real devices/carriers.
- **SDK 57 native tabs:** `expo-router/unstable-native-tabs` is explicitly unstable in this SDK line; isolate its use and budget a migration when adopting the stable API in a later SDK.
- **Privacy and age policy:** self-attested 16+ and contact matching need legal/privacy review for target launch regions and app-store requirements.
- **Media performance:** client-side optimization only may be insufficient on low-cost devices or slow connections; validate before public launch and revisit server processing only with measured evidence.
- **Realtime/group cost:** measure Convex subscription and receipt costs under large active groups; use cursors and bounded subscriptions to avoid per-message fanout.
- **Report evidence vs deletion:** implementation must preserve only selected evidence needed for active cases while honoring ordinary status/account cleanup elsewhere.
- **Service budget and provider setup:** no ceiling, production OAuth configuration, production domains, or provider accounts have been finalized. Complete cost modeling and provider setup before public launch.
- **Repository delivery:** the worktree contains existing uncommitted changes and the repository has no configured Git remote. Avoid reverting unrelated changes; CI automation depends on attaching a remote.

## Documentation references

Recheck these official docs and package versions when implementing, since versions and service terms change:

- [Expo SDK 57 Notifications](https://docs.expo.dev/versions/v57.0.0/sdk/notifications/)
- [Expo Router Native Tabs](https://docs.expo.dev/router/advanced/native-tabs/)
- [NativeWind installation](https://www.nativewind.dev/docs/getting-started/installation)
- [Clerk Expo quickstart](https://clerk.com/docs/expo/getting-started/quickstart)
- [Convex Clerk authentication](https://docs.convex.dev/auth/clerk)
- [Convex production regions](https://docs.convex.dev/production/regions)
- [Cloudflare R2 pricing](https://developers.cloudflare.com/r2/pricing/)
- [Clerk pricing](https://clerk.com/pricing)

