# What Knock's in-app feed gives Cadence

Research for [#2](https://github.com/ocornu-bhvr/cohort-005-project/issues/2), part of the map [#1](https://github.com/ocornu-bhvr/cohort-005-project/issues/1).

Sources are Knock's docs (docs.knock.app, read as of 2026-11-09), Knock's GitHub repos, and the npm registry. This note reports facts and options. It does not choose between them.

Package versions on npm at time of writing: `@knocklabs/node` 1.41.0, `@knocklabs/react` 0.15.1, `@knocklabs/client` 0.23.1.

---

## 1. Recipients: identification and client feed auth

### Identifying users

- A user needs an `id`, its only required property. Knock recommends a non-transient internal id such as a primary key. The id can be up to 256 characters, cannot contain `/` or `#`, and cannot be changed once set. ([Users](https://docs.knock.app/concepts/users))
- Reserved properties are `name`, `email`, `avatar` (a URL), `locale`, `phone_number`, `timezone` and `created_at`. Any other key/value pairs are stored as custom properties, capped at 256 kB per recipient (a `422` above that). ([Identifying recipients](https://docs.knock.app/managing-recipients/identifying-recipients))
- Every identify call is an upsert, and properties are deep-merged. ([Identifying recipients](https://docs.knock.app/managing-recipients/identifying-recipients), [Users](https://docs.knock.app/concepts/users))
- There are three ways to identify a user ([Identifying recipients](https://docs.knock.app/managing-recipients/identifying-recipients)):
  - **Direct:** `PUT /v1/users/{user_id}`, server-side.
  - **Inline:** pass full recipient objects in a workflow trigger (and in schedules and subscriptions). Knock guarantees the recipient is identified before the workflow runs for them. Inline identification needs objects (`[{ "id": "user-123" }]`), not id strings. Passing a plain string means "a user already identified".
  - **Bulk:** `POST /v1/users/bulk/identify`.
- **Client-side inline identification.** When `KnockProvider` is mounted with a `user` prop, it upserts that user in Knock using the public key. Setting `identificationStrategy="skip"` turns this off. ([Identifying recipients](https://docs.knock.app/managing-recipients/identifying-recipients), [KnockProvider](https://docs.knock.app/in-app-ui/react/sdk/components/knock-provider))
- Fetching the feed for a user Knock has not identified yet returns an empty feed, not an error. ([List feed items](https://docs.knock.app/api-reference/users/feeds/list_items))
- Users are per-environment. Development users and production users are separate datasets. ([Environments](https://docs.knock.app/version-control/environments))
- Deleting a user (`users.delete`) stops them being a recipient or actor. In-app messages that reference them then show a "missing user" marker. A merge API exists for guest-to-registered flows. ([Users](https://docs.knock.app/concepts/users))

### Authenticating the browser feed

Source: [Security and authentication](https://docs.knock.app/in-app-ui/security-and-authentication) unless noted otherwise.

- The backend uses a **secret API key**. The browser uses a **public API key**. Each environment has its own pair. ([Environments](https://docs.knock.app/version-control/environments))
- **Without enhanced security mode (the default):** the public key plus a user id is enough. The docs warn: "Without enhanced security mode, a request with your public key can act as any user." Knock tells you to enable it before going live. A development environment can keep it off.
- **With enhanced security mode:** Knock rejects public-key requests that lack a **signed user token**.
  - The token is a JWT signed **RS256** with a per-environment signing key, taken from Dashboard > Platform > API keys. That key is shown only once, and is stored in `KNOCK_SIGNING_KEY` by convention.
  - The `sub` claim (the user id) is required. `iat` and `exp` are recommended.
  - The token can be signed locally, with no network call to Knock.
- The current Node SDK README imports the signer from the package root: `import { signUserToken } from '@knocklabs/node'`, called as `signUserToken('user-1', { expiresInSeconds, grants, shouldGenerateJti })`. It uses `jose`, which is ESM-only, and the README covers a CJS workaround. The docs page still shows the older path `@knocklabs/node/lib/tokenSigner`. The default expiry is 3600 s. ([knock-node README](https://github.com/knocklabs/knock-node#jwt-token-signing))
- On the client, the token goes in as `knockClient.authenticate({ id }, token)`, or as the `userToken` prop on **`KnockProvider`**. The React feed page's callout says to pass it to `KnockFeedProvider`, but `KnockFeedProvider`'s source has no such prop. ([KnockProvider source](https://github.com/knocklabs/javascript/blob/main/packages/react-core/src/modules/core/context/KnockProvider.tsx), [KnockFeedProvider source](https://github.com/knocklabs/javascript/blob/main/packages/react-core/src/modules/feed/context/KnockFeedProvider.tsx))
- **Expiry:** Knock suggests matching the token's expiry to the app session. `onUserTokenExpiring` (plus `timeBeforeExpirationInMs`) on the client or `KnockProvider` lets the app refresh the token. Calling `authenticate` again restarts the real-time connection.
- `KnockProvider` takes `enabled` (default `true`). When it is `false`, the client makes no identify call, no requests and no websocket. Changing `enabled` reconnects, or disconnects and clears data. Knock recommends this over mounting and unmounting the provider when the identity changes. ([KnockProvider](https://docs.knock.app/in-app-ui/react/sdk/components/knock-provider))
- **Grants:** a token can carry UCAN-style `grants` and `explicit_grants: true` to limit what it can do. Examples are read-only feed (`user/feed_read`), `user/feed_write` for marking items, and tenant-scoped feed access.
- **Public-key endpoints** include get/update user, list feed items, and mark messages seen/read/archived (single, batch and bulk). Under default settings, the public key can call `PUT /v1/users/:id`.
- **Proxying** every feed request through the backend avoids client auth entirely. Knock documents it but does not recommend it, because of latency.

## 2. Server-side triggering (Node)

- The Node SDK is `@knocklabs/node`, generated by Stainless. ([knock-node README](https://github.com/knocklabs/knock-node))
  ```ts
  import Knock from '@knocklabs/node';
  const client = new Knock({ apiKey: process.env['KNOCK_API_KEY'] });
  const { workflow_run_id } = await client.workflows.trigger('workflow-key', {
    recipients: ['user-1'],
    data: { /* template data */ },
  });
  ```
- **The endpoint** is `POST /v1/workflows/{key}/trigger` (rate-limit tier 5). The body takes `recipients` (required), `actor`, `data`, `tenant`, `cancellation_key` and `settings`. ([Trigger API ref](https://docs.knock.app/api-reference/workflows/trigger), [Triggering via API](https://docs.knock.app/send-notifications/triggering-workflows/api))
  - The response is a `workflow_run_id` (UUID v4). "All workflow runs are executed asynchronously": a 200 means accepted, not delivered.
  - `data` is capped at 10 MB in total. String values over 1024 bytes are truncated in logs only.
  - An `actor` is linked to the in-app feed messages and fed into batch `actors`.
- **Multiple recipients:** up to **1000 recipients per trigger**. For larger lists Knock points to subscriptions (objects). Bulk triggering is not supported. Per-recipient data goes under `$trigger_data` on each inline recipient and is merged into `data`. ([Triggering via API](https://docs.knock.app/send-notifications/triggering-workflows/api))
- **Idempotency** ([Idempotent requests](https://docs.knock.app/api-reference/overview/idempotent-requests)):
  - The `Idempotency-Key` header (up to 255 chars) is accepted **only** on the trigger endpoint. A repeat within 24 h returns the original response.
  - A request with the same key but different params returns an error.
  - Only 2xx responses are cached. Replayed responses carry `idempotent-replayed: true`.
  - Knock suggests semantic keys such as `order-placed:user-123:order-456`, or a job id.
- **The SDK's own idempotency key and retries** ([knock-node README](https://github.com/knocklabs/knock-node), [client.ts](https://github.com/knocklabs/knock-node/blob/main/src/client.ts)):
  - The SDK sends an `Idempotency-Key` on every non-GET request: `stainless-node-retry-<uuid>` unless the caller passes `idempotencyKey` in request options.
  - It retries connection errors, 408, 409, 429 and 5xx twice by default (`maxRetries`) with exponential backoff.
  - The default timeout is 60 s (`timeout`).
  - Its auto-generated key covers only its own retries. A semantic key from Cadence would also cover app-level retries.
- **Cancellation** ([Canceling workflows](https://docs.knock.app/send-notifications/canceling-workflows)):
  - Trigger with a `cancellation_key`, then call `client.workflows.cancel(key, { cancellation_key, recipients? })`.
  - Only runs that are paused can be cancelled: in a batch, delay, wait-for-event, or fetch retry. A workflow with no such step sends immediately.
  - A cancel returns 204 when the request is queued. Knock advises against cancelling within 5 s of the trigger.
  - A finished in-app message cannot be cancelled. Archive it instead.
- **Trigger `settings`:** `sandbox_mode` and `skip_delay` exist for testing. The in-app channel does **not** support sandbox mode (see §6). ([Triggering via API](https://docs.knock.app/send-notifications/triggering-workflows/api), [Knock plans](https://docs.knock.app/manage-your-account/knock-plans))
- **Tenants:** passing a `tenant` scopes the in-app feed messages to that tenant. The feed can then be filtered by tenant. Tenants work on all plans, and only per-tenant preferences, branding, translations and guides are Enterprise-only. ([Triggering via API](https://docs.knock.app/send-notifications/triggering-workflows/api), [Filtering feeds](https://docs.knock.app/in-app-ui/feeds/filtering-in-app-feeds), [Knock plans](https://docs.knock.app/manage-your-account/knock-plans))
- **Other triggers:** besides the API, workflows can be triggered by source events, schedules or audiences. A trigger step can be set to run "once per recipient" (optionally per tenant). An inactive workflow returns `workflow_inactive`. ([Triggering overview](https://docs.knock.app/send-notifications/triggering-workflows))

## 3. Feed item model

- **Endpoint:** `GET /v1/users/{user_id}/feeds/{channel_id}`, rate-limit tier 2 (5 req/s). For this endpoint the rate limit is always per-user and per-environment, even without a user token. Items come back newest first. ([List feed items](https://docs.knock.app/api-reference/users/feeds/list_items), [Rate limits](https://docs.knock.app/api-reference/overview/rate-limits))
- **Response:**
  - `entries[]` (feed items).
  - `meta`: `total_count`, `unread_count` and `unseen_count`, used for badges.
  - `page_info`: `after`, `before` and `page_size`.
  - `vars`.
- **`InAppFeedItem` fields:** `id`, `inserted_at`, `updated_at`, `actors[]`, `activities[]`, `total_activities`, `total_actors`, `blocks[]`, `data` (trigger data), `source` (workflow key, categories, version), `tenant`, `seen_at`, `read_at`, `archived_at`, `interacted_at`, `clicked_at` and `link_clicked_at`. (Knock OpenAPI spec, [docs.knock.app/openapi.json](https://docs.knock.app/openapi.json))
- **Content blocks:** `MessageInAppFeedContentBlock` is `{ name, type: "markdown", content, rendered }`, where `rendered` is HTML. `MessageInAppFeedButtonSetBlock` is `{ name, type: "button_set", buttons: [{ name, label, action }] }`. ([content block](https://docs.knock.app/api-reference/messages/schemas/message_in_app_feed_content_block), [button set block](https://docs.knock.app/api-reference/messages/schemas/message_in_app_feed_button_set_block))
- **Templates** are edited in Knock's template editor with Liquid. There are three variants: Standard (content plus an optional **action URL**, the cell's click target), Single-action (one button) and Multi-action (two buttons, each with its own URL). The action URL can be dynamic via Liquid. ([In-app feeds](https://docs.knock.app/integrations/in-app/feeds))
- **Content is immutable once generated.** Only engagement statuses can change. To correct a message, archive it and trigger a new one. ([In-app feeds FAQ](https://docs.knock.app/integrations/in-app/feeds))
- **Engagement states:**
  - Unseen, seen and read. Seen means rendered in the open feed; read means explicitly marked.
  - Archived removes an item from the default feed view.
  - Interacted is recorded on a cell or button click. Link-clicked is recorded only when link tracking is on, and action URLs don't produce it.
  - By default the prebuilt badge counts **unseen** items. `badgeCountType="unread"` switches it to unread.
  
  ([In-app feeds](https://docs.knock.app/integrations/in-app/feeds), [Message statuses](https://docs.knock.app/send-notifications/message-statuses))
- **Query parameters:**
  - `status` is `unread|read|unseen|seen|all`. `archived` is `exclude|include|only`.
  - Further filters: `source` (workflow key), `tenant`, `has_tenant`, `workflow_categories`, `trigger_data` (JSON) and `inserted_at.{gt,gte,lt,lte}`.
  - `exclude` takes field paths. `mode` is `rich|compact` (compact drops `activities` and nested data).
  - Cursors are `after` and `before`. `page_size` defaults to 50.
  - Filters set in `defaultFeedOptions` also apply to real-time updates and badge counts.
  
  ([List feed items](https://docs.knock.app/api-reference/users/feeds/list_items), [Filtering feeds](https://docs.knock.app/in-app-ui/feeds/filtering-in-app-feeds))
- **Response filter:** what the feed returns about actors, the recipient, `data` and `vars` is set per environment in the dashboard (channel settings > Advanced > API response filter). By default it returns all trigger data and every actor and recipient property, `email` included. Secret variables are never returned. ([In-app feeds](https://docs.knock.app/integrations/in-app/feeds#customizing-api-response-content))
- **Real-time:** Knock runs a managed websocket service, and the SDKs handle connection and reconnection. Options are `auto_manage_socket_connection` (disconnect inactive tabs after `auto_manage_socket_connection_delay`, default 2000 ms) and `__experimentalCrossBrowserUpdates` (BroadcastChannel sync across tabs). ([Socket behavior](https://docs.knock.app/in-app-ui/feeds/socket-behavior), [React feed](https://docs.knock.app/in-app-ui/react/feed#socket-behavior-options))
- **Rate limit on status updates:** the batch status endpoints (mark seen/read/archived) are limited to 1 update per second per entity. Rejected ids still return 2xx and come back in an `x-ratelimited-{param}` header. ([Batch rate limits](https://docs.knock.app/api-reference/overview/batch-rate-limits))

## 4. UI options: prebuilt `@knocklabs/react` versus headless

### Prebuilt components

Source: [React feed](https://docs.knock.app/in-app-ui/react/feed) and the SDK component pages.

- `KnockProvider` takes `apiKey`, `user`, `userToken`, `enabled`, `host`, `i18n`, `onUserTokenExpiring` and `logLevel`.
- `KnockFeedProvider` takes `feedId` (the channel id), `colorMode` (`light|dark`) and `defaultFeedOptions`.
- `NotificationIconButton` is the bell plus badge, with a `badgeCountType` prop.
- `NotificationFeedPopover` is positioned by popper.js. It takes `buttonRef`, `isVisible`, `onOpen`, `onClose`, `closeOnClickOutside` and `placement`.
- `NotificationFeed` takes `renderItem`, `renderHeader`, `onNotificationClick`, `onNotificationButtonClick`, `onMarkAllAsReadClick`, `initialFilterStatus` and `EmptyComponent`.
- `NotificationCell` and `Avatar` can be reused inside `renderItem`.

### Default behaviour of the prebuilt feed

- Opening the popover marks items seen.
- Clicking a cell marks it read and redirects to `action_url`.
- The header has "mark all as read" and status filters.
- The "x" on a cell archives it. There is no built-in archived view.
- English strings ship by default. `i18n` overrides them.

### Styling the prebuilt feed

- Import `@knocklabs/react/dist/index.css`, then override the `--rnf-*` CSS variables. `theme.css` defines 45 of them for fonts, spacing and colours, alongside a larger `--knock-*` token set.
- Alternatively, drop the stylesheet and restyle the `rnf-*` classes yourself.
- Button colours are set by this front-end CSS, not by the Knock dashboard.
- The docs don't mention Tailwind or shadcn. Theming is plain CSS variables and classes.

([Styling](https://docs.knock.app/in-app-ui/feeds/styling), [theme.css](https://github.com/knocklabs/javascript/blob/main/packages/react/src/theme.css))

### Dependencies of the prebuilt feed

- `@knocklabs/react` 0.15.1 depends on Knock's `@telegraph/*` design-system packages, `@radix-ui/react-dialog`, `@radix-ui/react-hover-card`, `@popperjs/core`, `lucide-react` and `@knocklabs/react-core`.
- Its peer dependencies are React 17, 18 or 19. Next and `@tanstack/react-router` are optional peers.

(npm registry, `npm view @knocklabs/react`)

### Headless option

Sources: [Headless feed](https://docs.knock.app/in-app-ui/react/headless/feed), [useNotifications](https://docs.knock.app/in-app-ui/react/sdk/hooks/use-notifications), [FeedClient](https://docs.knock.app/in-app-ui/javascript/sdk/feed-client), [FeedStoreState](https://docs.knock.app/in-app-ui/javascript/sdk/types/feed-store-state).

- **Hooks** come from the same package:
  - `useKnockClient()` / `useAuthenticatedKnockClient()` return the client.
  - `useNotifications(knock, channelId, options)` returns a `Feed` client.
  - `useNotificationStore(feedClient)` returns `{ items, metadata, pageInfo, loading, networkStatus }`, a zustand store.
  - `useKnockFeed()` gives `{ feedClient, items, metadata }` inside `KnockFeedProvider`.
- **`Feed` client methods:**
  - `fetch` and `fetchNextPage`.
  - `markAsSeen/Unseen/Read/Unread/Archived/Unarchived/Interacted`, each taking one item or many.
  - `markAllAsSeen`, `markAllAsRead` and `markAllAsArchived`.
  - `on/off` for the events `items.received.realtime`, `items.received.page`, `items.read`, `items.archived` and so on (useful for toasts), plus `listenForUpdates`.
- **With headless, the app must mark items seen itself** (for example `markAllAsSeen()` on open). The prebuilt components do this automatically.
- The item body arrives as pre-rendered HTML (`blocks[n].rendered`), so a custom UI renders HTML that Knock generated from the template. (See §3.)
- **Plain JS:** the `@knocklabs/client` package exposes the same `Feed` client without React. ([Custom UI](https://docs.knock.app/in-app-ui/feeds/custom-ui))

## 5. Batching, throttling and digests

- **Batch step** ([Batch function](https://docs.knock.app/designing-workflows/batch-function)):
  - It collects triggers for the same recipient, optionally per **batch key** (for example `{{data.course_id}}`), into one notification.
  - **Windows** can be fixed, dynamic (from data: ISO timestamp, duration or window rule), relative, or sliding. A sliding window extends on each trigger up to a max. No batch can stay open longer than 90 days.
  - An optional **max activity count** (2–1000) closes the batch early.
  - **"Immediately flush leading item"** sends the first event at once and batches the rest.
  - **Template variables** are `total_activities`, `total_actors`, `activities` and `actors`. `activities` and `actors` hold at most **10** items, the first or last ten by setting. Enterprise can raise that to 100.
  - After a batch, `data` is flattened to scalars, with strings truncated to 256 chars.
  - **Version mode:** "latest" (the default) keeps an open batch across workflow commits.
  - There is no API to close a batch early. Items can be removed by cancelling their run via `cancellation_key`, and an empty batch sends nothing. Per-recipient batch windows and timezones are not supported.
- **Throttle step:** limits how many times a workflow runs per recipient within a window (fixed or dynamic), with a threshold (default 1) and an optional throttle key. Runs over the limit simply stop. ([Throttle function](https://docs.knock.app/designing-workflows/throttle-function))
- **Digests:** Knock says batch is "not exactly" a digest. For fixed daily or weekly summaries it recommends a recurring-digest pattern built on schedules, with the app's own system as the source of truth. ([Batch FAQ](https://docs.knock.app/designing-workflows/batch-function#faq-batching-vs-digesting), [Recurring digests tutorial](https://docs.knock.app/tutorials/building-recurring-digests))
- **Other workflow steps that pause a run** (and so make it cancellable): delay, wait-for-event, and fetch during retry backoff. ([Canceling workflows](https://docs.knock.app/send-notifications/canceling-workflows))

## 6. Environments, limits and free tier

- **Environments** ([Environments](https://docs.knock.app/version-control/environments)):
  - An account starts with **production only**. "Enable environments" adds Development (and allows Staging and others) and cannot be undone. After that, Development is the default target of the CLI and Management API.
  - Each environment has its own API keys and data (users, tenants, messages, logs).
  - Workflows are version-controlled: edited in the environment where they were created, committed, and promoted upward.
  - Enhanced security mode and the signing key are set per environment.
- **Sandbox and usage** ([Knock plans](https://docs.knock.app/manage-your-account/knock-plans)):
  - Usage counts in **every** environment.
  - The in-app channel does **not** support sandbox mode. To limit in-app message creation outside production, Knock points to channel conditions.
  - A message is counted per user per channel per delivery. A skipped channel step is not counted.
- **Rate limits** ([Rate limits](https://docs.knock.app/api-reference/overview/rate-limits)):
  - Endpoints are tiered: tier 1 = 1 req/s, tier 2 = 5, tier 3 = 60, tier 4 = 200, tier 5 = 1,000.
  - Limits apply per environment, or per environment and user when a signed user token is used. Going over returns `429`.
  - Trigger is tier 5. Feed list is tier 2, per user. Batch message-status updates are additionally capped at 1 update/s per message.
- **Plans** ([pricing page](https://knock.app/pricing), [Knock plans](https://docs.knock.app/manage-your-account/knock-plans)):
  - **Free:** $0, **10,000 messages/month**, unlimited stored profiles, workflows, feeds per recipient, environments and team members, 30-day log retention, Knock branding kept.
  - **Starter:** $250/month with 50,000 messages, then $0.005 each.
  - **Enterprise only:** batch render limit above 10, translations, per-tenant preferences and branding, SSO.
  - No EU data residency yet (expected by the end of 2026).

---

## Things that may matter downstream

These are observations, not recommendations.

- **Identity.** Cadence's id goes into Knock as the user `id` permanently, and ids can't contain `/` or `#`. The feed is empty, not an error, until a user is identified. `KnockProvider` with `user` will itself upsert the user from the browser unless `identificationStrategy="skip"`.
- **DevUI switching.** With enhanced security off (the default), any id passed to `KnockProvider` reads that user's feed, which makes switching users in dev trivial. With it on, the server must sign a token for the current user on every switch. `enabled` and re-`authenticate` are the documented mechanisms for changing identity.
- **Dev environment.** A Development environment exists only after someone enables environments in the dashboard, and that can't be undone. The map assumes "dev-environment keys" exist. That is worth confirming.
- **Testing.** Dev and test traffic to the in-app channel creates real messages that count toward the 10k/month free quota, because in-app has no sandbox mode.
- **Failure semantics.** Triggers are asynchronous: 200 means accepted. The Node SDK retries on errors with a per-call auto idempotency key, and waits up to 60 s by default. The `timeout` and `maxRetries` options control how long a Cadence action could block on Knock.
- **Fan-out.** One trigger can carry up to 1000 recipients, for example all Platform Admins, with per-recipient `$trigger_data`.
- **Privacy.** By default the feed API returns full trigger `data` and actor and recipient properties, emails included, to the browser. The per-environment response filter controls this.
