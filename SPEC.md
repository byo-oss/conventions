# BYO conventions

Status: **draft v0**, 2026-10-20. Baseline: `better-auth` `^1.6.23`.

These are the rules BYO packages follow so they work as one system instead of a pile of plugins. A BYO package is a library, not a service: it runs inside your app, keeps its data in your database, and uses better-auth users and organizations as its identity.

The keywords **MUST**, **SHOULD** and **MAY** mean what RFC 2119 says. Part 1 is **required**: every rule there can be met today with plain better-auth primitives, and most are drawn from what `better-inbox` already does. Part 2 is **proposed**: shared packages and contracts that don't exist yet. Nothing in Part 1 depends on Part 2.

## Tiers

| Tier | What it is | Example | How it integrates |
|---|---|---|---|
| **Plugin** | A better-auth plugin. Its tables come from the plugin `schema`, its endpoints from `createAuthEndpoint`. | `better-inbox` | `betterAuth({ plugins: [inbox()] })` |
| **App** | A larger product with its own migrations and request handler. | None yet (upstream `better-helpdesk` has this shape) | An `identify(request)` adapter that maps a better-auth session to the app's identity |

New packages SHOULD be plugins. A package is an app only when it needs things a plugin can't give it, such as its own route tree, inbound webhooks or a separate database. Rules marked *(plugin)* apply to plugins only. Everything else applies to both tiers.

---

# Part 1: Required (v0)

## 1. Naming and packaging

1.1 Every BYO package is published under the `@byo-oss` npm scope. A product is named for the domain it covers, not for a product noun or brand: `@byo-oss/notifications` (the inbox), `@byo-oss/support` (customer support), `@byo-oss/feedback`. A shared package is named for what it does (`@byo-oss/core`). Repos live in `github.com/byo-oss`. A package we first released under an unscoped name (`better-inbox`) moves to the scope at its next minor release, keeps its version line, and its old name is deprecated on npm with a pointer to the new one.

1.2 Subpath exports, following [`notifications/package.json`](https://github.com/byo-oss/notifications/blob/main/package.json). Ship ESM and CJS, with types.

| Export | Contents |
|---|---|
| `.` | The server plugin (or app builder), its option types, its error codes |
| `./client` | The better-auth client plugin *(plugin)* |
| `./react` | Hooks and components |
| `./<provider>` | Optional adapters for an outside service, e.g. `@byo-oss/notifications/wraps` |

1.3 Every outside service is an optional peer dependency behind its own subpath, through the provider rules in §10. The root export never imports a provider SDK. `better-auth` is a peer dependency, and so is `react` when there's a `./react` export.

1.4 The README says near the top: "Community plugin — not affiliated with the better-auth team." Don't use Better Auth's logo or name it in the package name.

1.5 One package per domain. A package covers its whole domain's data model, at the depth a young product needs:
- It owns every model in its domain. Preferences belong to `@byo-oss/notifications`; roadmap and changelog belong to `@byo-oss/feedback`. Don't split a domain into sub-packages (`@byo-oss/notification-preferences`).
- Breadth comes from opt-in options (`feedback({ changelog: true })`) and subpaths (§10), not new packages. A plugin adds tables only for the features that are switched on.
- Depth stops where the "Outgrowing this" section (§9.1) begins. Below that line the package covers it; above it, the README points to the tool people move to. Packages don't chase feature parity with those tools.
- Something is a new domain, and gets its own package, only when someone would install it without the others and it has its own models and permissions. Otherwise it's a feature of an existing package.

## 2. Plugin shape *(plugin)*

2.1 The plugin `id` is the package name without the scope (`@byo-oss/feedback` → `feedback`), and it doubles as the endpoint path prefix: `/feedback/list`, `/feedback/vote`. Notifications predates this rule and keeps the id `inbox` (`/inbox/*`) until a major release. Ids, path prefixes, model names and `auth.api` endpoint keys MUST be claimed in the [registry](#registry) before the first release. All plugins share one flat `auth.api` namespace, so a new endpoint key that's generic (`notify`, `list`) SHOULD carry the product name (`listFeedback`, `processFeedbackDigests`).

2.2 Model names are claimed globally across BYO. Prefix any generic noun with the product (`feedbackPost`, not `post`). Field names are camelCase and follow better-auth's own: `userId`, `organizationId`, `createdAt`.

2.3 Accept `options.schema` and merge it with `mergeSchema(schema, options.schema)` so apps can rename tables and fields ([`notifications/src/index.ts`](https://github.com/byo-oss/notifications/blob/main/src/index.ts)).

2.4 Define error codes with `defineErrorCodes`, export them as `<ID>_ERROR_CODES`, and attach them as `$ERROR_CODES` ([`src/error-codes.ts`](https://github.com/byo-oss/notifications/blob/main/src/error-codes.ts)). Throw with `APIError.from(status, CODE)`.

2.5 Validate options when the plugin is created, and throw a message prefixed with the package name (`[@byo-oss/notifications] routes["x"] uses channel "y", which is not in channels`). A misconfiguration should fail at boot, not on the first request.

2.6 The client plugin declares `$InferServerPlugin` and a `pathMethods` entry for every endpoint the client calls ([`src/client.ts`](https://github.com/byo-oss/notifications/blob/main/src/client.ts)).

## 3. Identity

3.1 A signed-in person is a better-auth `user`, referenced by `userId`. A tenant is a better-auth `organization`, referenced by `organizationId`. Packages MUST NOT keep their own user or customer table for people who have accounts.

3.2 *(plugin)* `userId` fields declare `references: { model: "user", field: "id", onDelete }`. Set `onDelete` explicitly:
- `"cascade"` for rows that belong to the user (notifications, preferences, votes).
- `"set null"` for content others depend on (a feedback post with replies, a ticket). The row stays and loses its author.

better-auth's built-in migrator defaults to `"cascade"` (`better-auth/dist/db/get-migration.mjs`). Other generators may not, so don't rely on the default.

3.3 Organization support is optional. When a package is given an `organizationId`, it checks that the organization plugin is installed (`plugins.some(p => p.id === "organization")`) and throws `ORGANIZATION_PLUGIN_REQUIRED` if it isn't. Membership is checked against the `member` model ([`notifications/src/routes.ts`](https://github.com/byo-oss/notifications/blob/main/src/routes.ts)).

3.4 People without an account (an email sender in support, a visitor voting on public feedback) are stored with an email address and a nullable `userId`. The package links the two when a matching account exists with that email verified.

3.5 *(app)* `identify(request)` returns the user, the organizations they are an active member of, and their permissions (§4). It never returns a role name for the package to interpret.

## 4. Access and permissions

4.1 Every endpoint is one of three kinds, and its docs say which:

| Kind | How | Used for |
|---|---|---|
| **Server-only** | `createAuthEndpoint.serverOnly` | Writes the app makes on someone's behalf (`notify`) and sweeps (`processDeliveries`) |
| **Own rows** | `sessionMiddleware`, every query filtered by `session.user.id` | A person's own data (list or mark their notifications) |
| **Permission-gated** | Session plus a permission check (4.3) | Acting on other people's data (moderating feedback, replying to tickets, reading an org's activity) |

4.2 A package that has permission-gated endpoints exports its permission statements as a constant in better-auth's access-control format, ready to spread into the app's catalog:

```ts
// @byo-oss/feedback
export const feedbackStatements = {
  feedback: ["read", "create", "moderate", "delete"],
} as const
```

```ts
// the app's access.ts
import { createAccessControl } from "better-auth/plugins/access"
import { defaultStatements } from "better-auth/plugins/organization/access"
export const statements = { ...defaultStatements, ...feedbackStatements, project: [...] } as const
export const ac = createAccessControl(statements)
```

The app decides which roles get which actions. The package's README suggests a default mapping.

4.3 Packages check permissions, never role names. Every permission-gated endpoint calls an injected `authorize` option:

```ts
authorize?: (ctx: {
  session: { user: User; session: Session }
  headers: Headers
  organizationId?: string
  permissions: Record<string, string[]>  // e.g. { feedback: ["moderate"] }
}) => Promise<boolean>
```

The default checks the member's org role through the organization plugin's `hasPermission`, and denies when there's no organization (staff acting for the vendor). Apps with more to check replace it: an app that also narrows by credential (OAuth scopes, API key grants, impersonation) plugs in a check that applies those too. A role check alone would let a read-only token through.

## 5. Background work

A plugin can't schedule anything, so a package that has deferred work (delayed sends, retries, digests, rollups) follows the contract from [`notifications/src/delivery.ts`](https://github.com/byo-oss/notifications/blob/main/src/delivery.ts):

5.1 It exposes one server-only sweep endpoint, `process<Thing>` (e.g. `processDeliveries`), that the app calls from any scheduler.

5.2 A sweep claims each row atomically with a lease before working on it, so sweeps can overlap safely. A claim older than the lease is treated as dead and retried. Failures back off and are marked `failed` after `maxAttempts`.

5.3 A sweep works through at most `batchSize` rows and returns counts plus `hasMore`: `{ sent, skipped, retrying, failed, hasMore }`.

5.4 Work that's due right away MAY run inline when better-auth has `advanced.backgroundTasks.handler` and the batch is small. Otherwise it waits for the next sweep.

5.5 The docs state the delivery guarantee: at-least-once, plus what can cause a duplicate.

## 6. Data lifecycle

6.1 Deleting a user removes or anonymizes everything the package holds about them. Under 3.2 this happens through foreign keys, with no code.

6.2 A package that stores `organizationId` exports `deleteOrganizationData(auth, organizationId)` and documents how to wire it into the organization plugin:

```ts
import { deleteOrganizationData as deleteInboxOrgData } from "@byo-oss/notifications"

organization({
  organizationHooks: {
    afterDeleteOrganization: async ({ organization }) => {
      await deleteInboxOrgData(auth, organization.id) // reads the adapter from auth.$context
    },
  },
})
```

`organizationHooks` belong to the app's `organization()` options, so a package can't register them; the app wires the cleanup in. A package MAY also clean up by itself with a plugin `hooks.after` matcher on `/organization/delete`. It still MUST export the function, because a hook on that path doesn't see deletes made some other way.

6.3 A package that stores `organizationId` per member also exports a cleanup for when someone leaves or is removed: `deleteMemberData(auth, { userId, organizationId })`.

## 7. UI *(when there's a `./react` export)*

7.1 **Headless first.** Hooks own the state and the calls (`useInbox`). Components are thin and optional, built on the hooks. Anything a component can do, the hooks can do.

7.2 Components take the better-auth `client` as a prop (no global), take `onNavigate(href)` so the app's router handles links, and take `renderItem` (or a similar render prop) for the main repeated element.

7.3 Styling uses Tailwind classes on shadcn CSS variables (`bg-background`, `text-foreground`, `border`, `bg-accent`, `text-muted-foreground`). There are no UI runtime dependencies. Accept `className` for the root, plus a named class prop for each part an app is likely to restyle (`buttonClassName`). The README documents the Tailwind `@source` line for `node_modules`.

7.4 Interactive parts work from the keyboard, close on Escape and outside click, and announce counts to screen readers.

7.5 Every user-facing string is overridable through a `labels` prop with English defaults. (`better-inbox` doesn't do this yet; see the compliance table.)

## 8. Docs and tests

8.1 The README uses this outline:
1. A one-line pitch
2. Install
3. Setup in five steps or fewer: plugin, migration, client, component, first call
4. Options table
5. API table (endpoint, kind from §4.1, input, returns)
6. Delivery guarantees, if any
7. "What this is not (yet)"
8. The disclaimer from 1.4

8.2 The repo ships an `AGENTS.md`, plus a skill if setup involves wiring several files, so a coding agent can install and use the package correctly.

8.3 Tests run against a real better-auth instance with the memory adapter. The repo includes an example app with an end-to-end suite ([`notifications/examples/tanstack-start`](https://github.com/byo-oss/notifications/tree/main/examples/tanstack-start)).

8.4 Versioning: semver, `0.x` until the API settles. Widening the `better-auth` peer range is a minor release; narrowing it is a major one.

## 9. Exit path

BYO is the right default for a young product, and some orgs will outgrow it: they need push and SMS at scale, a support team of fifty, or usage billing with invoicing. Leaving should be a planned, boring migration, not a rewrite. Every package plans for its own replacement.

9.1 The README has an **"Outgrowing this"** section covering:
- The signs you've outgrown it, said plainly ("more than N agents", "you need SMS and push").
- The third-party tools people usually move to (inbox → Novu, Knock or Courier; support → Zendesk, Intercom or Front; feedback → Canny or Featurebase; entitlements → Stripe Entitlements, Autumn or Orb).
- A field mapping from our models to each tool's import format, for at least the most common one.
- The migration steps: run in parallel, backfill, cut over, uninstall.

9.2 The schema is documented as a public contract: every model, every field, what each means. A schema change that breaks reading the old data is a major release, so exports and migration scripts keep working.

9.3 Every package exports `export<Thing>Data(auth, { organizationId?, cursor? })`. It returns all the package's data (or one org's), paged with a cursor, as plain JSON in the documented schema, with better-auth `userId`/`organizationId` and the user's email on each row so it can be matched in the destination tool. It is server-only; it is not an endpoint.

9.4 App code reaches the package through a small, stable surface (`auth.api.notify`, a hook, a component), never through its tables. When that surface is swapped for a third-party SDK, the migration touches those call sites and nothing else. Packages don't encourage apps to query their tables directly.

9.5 Uninstalling is documented: remove the plugin, then drop these tables in this order. No package leaves rows or columns on better-auth's own tables (`user`, `organization`, `member`) that have to be cleaned up by hand.

## 10. Providers

Like better-auth, which keeps users and sessions in your database but plugs in OAuth providers and email senders, a BYO package owns its data and lets outside services plug in at the edges.

10.1 **What's pluggable.** Any outside service the package *uses* goes behind a provider interface: delivery (email, SMS, push, chat), file storage, AI, search, scheduling. Examples today are inbox's `InboxChannel` ([`notifications/src/delivery.ts`](https://github.com/byo-oss/notifications/blob/main/src/delivery.ts)) and upstream `better-helpdesk`'s storage, AI, search and job adapters.

10.2 **What isn't.** The package's own models are always the store of record, so the hooks, components and permission checks read from one place. A package MUST NOT offer a provider that replaces its core store (an inbox feed backed by Knock, tickets backed by Zendesk). Moving to such a tool goes through the exit path (§9) instead. Mirroring data *out* to a third party while you run both is fine; see P8.

10.3 **The interface.**
- It is exported as a named type from the root (`InboxChannel`, `SupportStorage`).
- It is the smallest set of async methods the package needs, taking plain objects.
- It documents its error semantics: throw to retry, and how to mark a permanent failure.
- It is versioned with the package. Adding a required method is a major release.

10.4 **First-party adapters.** Each package ships at least one adapter for its pluggable interfaces, on a subpath (`@byo-oss/notifications/wraps`), with the provider SDK as an optional peer dependency. Wraps is the default for email and SMS.

10.5 **Community adapters** are separate packages outside our scope, named `byo-<name>-<provider>` (`byo-notifications-onesignal`, `byo-support-s3`) or the same under the author's own scope (`@acme/byo-notifications-onesignal`). They depend on the package's types and nothing else of ours. Only first-party packages publish under `@byo-oss`. The package README lists the known ones.

10.6 **Contract tests.** A package with a provider interface SHOULD export a test suite on `./testing` that any adapter can run against itself:

```ts
// byo-notifications-onesignal/src/channel.test.ts
import { testChannel } from "@byo-oss/notifications/testing"
testChannel(() => onesignalChannel({ appId: "test", fetch: fakeFetch }))
```

The suite checks the documented behavior: the call shape, retry-on-throw, idempotency where promised, and that it doesn't mutate its inputs. First-party adapters run the same suite.

10.7 **Exception: when the provider is the source of truth.** Where the outside system already owns the facts and our package acts as the in-app interface, the provider sits behind the core instead of at the edge. Entitlements is the case in view: Stripe, Polar or Autumn owns subscriptions, and the package defines a `BillingSource` interface, keeps a local cache in its own models, and answers `can(org, feature)` and usage counters from that cache. A package that uses this exception says so in the registry and documents how fresh the cache is and how it resyncs.

---

# Part 2: Proposed (non-normative)

Shared packages and contracts to build once a second plugin exists to prove them. Don't write a MUST against any of these until it ships.

## P1. Domain events: the glue

Every package emits typed events after its write commits. Other packages subscribe instead of calling each other directly:

```ts
type ByoEvent = {
  type: `${string}.${string}.${string}` // "<package id>.<noun>.<past-tense verb>", e.g. "feedback.post.status_changed"
  occurredAt: Date
  actorId: string | null                 // a user id, or null for the system
  userId?: string                        // whom it's about, if one person
  organizationId?: string
  subject: { type: string; id: string }  // e.g. { type: "feedbackPost", id }
  data: Record<string, unknown>
}
```

Each package accepts `onEvent?: (event) => Promise<void> | void`. Delivery is in-process and best-effort, and an error never fails the request that caused it. A consumer that needs durability writes its own rows, the way inbox does with `notificationDelivery`.

Example wiring, which `@byo-oss/core` could reduce to one line:

```ts
const onEvent = async (event: ByoEvent) => {
  await activity.record(event)                         // org activity feed
  if (shouldNotify(event)) await auth.api.notify({ body: toNotification(event) })
}
```

## P2. Notifications go through the inbox

A package sends no user-facing notification email of its own. It emits an event, or calls `auth.api.notify` when inbox is installed, using the event `type` as the notification `type`. That gives every package delay-if-unread, throttling and (once they exist) preferences for free. The exception is email that *is* the record, such as a support reply to an outside contact. That goes through a mailer (P4).

## P3. `@byo-oss/identity`

One `ByoIdentity` type and one better-auth implementation of `identify(request)` (user, active organizations, permissions), so each app package stops writing its own. Upstream `better-helpdesk`'s [`examples/adapters/identify-better-auth.ts`](https://github.com/better-helpdesk/better-helpdesk/blob/main/examples/adapters/identify-better-auth.ts) is the starting point.

## P4. `@byo-oss/mailer`

A minimal `Mailer = { send(message: { to, subject, html, text, replyTo?, headers? }): Promise<{ id: string }> }` with a Wraps implementation, shared by inbox channels and support's outbound email.

## P5. `@byo-oss/core`

- `sweepAll(auth)` runs every installed package's `process*` endpoint, so one cron route serves them all.
- `deleteOrganizationData` and `deleteMemberData` fan-out across every installed package.
- The `ByoEvent` type and an event bus.

## P6. Agent-native operations

Packages export operation metadata in one operation shape (`name` in snake_case, `method`, `path`, `summary`, `description`, zod `input`/`output`, `permissions` in the package's statements from §4.2). An app can then register them in its one registry and get REST, MCP, OpenAPI and `llms.txt` together. Open question: how a package's handler gets the app's `auth` and caller.

## P7. Preferences

One preferences model, per user and per `(type, channel)`, owned by inbox and read through its `shouldDeliver`. Other packages register their notification types and default channels with it.

---

## P8. Bridges for the move out

Where a package already has an adapter point, ship a bridge to the tool people most often move to, so they can run both side by side before cutting over:
- An inbox channel that forwards notifications to Novu or Knock (`@byo-oss/notifications/novu`).
- A support relay that mirrors new tickets into Zendesk.
- A script that turns `export<Thing>Data` output into the destination's import format.

Once events (P1) exist, a bridge is just one more event consumer. Parallel running then needs no changes to the package.

## Registry

Names claimed by BYO packages. Add a row before you release.

| Package | Plugin id / path prefix | Models | `auth.api` keys | Error codes | Status |
|---|---|---|---|---|---|
| `@byo-oss/notifications` (published as `better-inbox`) | `inbox` → `/inbox/*` | `notification`, `notificationDelivery`, `inboxPresence` | `notify`, `processDeliveries`, `listNotifications`, `markRead`, `markSeen`, `markAllRead`, `unreadCount` | `INBOX_ERROR_CODES` | Released (0.6.0) |
| *`@byo-oss/support`* | *`support`?* | *TBD* | — | — | Planned; net-new, built to this spec, borrowing from upstream `better-helpdesk` (MIT) |
| *feedback / roadmap / changelog* | *`feedback`?* | *`feedbackPost`, `feedbackVote`, `changelogEntry`?* | — | — | Planned; name not claimed |
| *org activity feed* | *`activity`?* | *`activityEvent`?* | — | — | Planned |
| *entitlements* | *`entitlements`?* | *`entitlementPlan`, `usageCounter`?* | — | — | Planned; uses the §10.7 exception (`BillingSource`: Stripe, Polar, Autumn) |

Inbox's `notify` and `processDeliveries` predate the prefix rule in 2.1. They keep their names, but `notify` blocks any other package from using that generic key.

## Compliance today

| Rule | better-inbox 0.6.0 |
|---|---|
| 1.2 export layout | ✅ `.`, `./client`, `./react`, `./wraps` |
| 1.4 disclaimer | ✅ |
| 2.x plugin shape | ✅ |
| 3.1 identity is the better-auth user | ✅ |
| 3.2 explicit `onDelete` | ⚠️ References `user` but relies on the migrator's default |
| 3.5 permissions, not roles | n/a |
| 4.1 endpoint kinds | ✅ |
| 5.x sweep contract | ✅ `processDeliveries` |
| 6.2 org cleanup | ❌ `organizationId` has no reference and there's no exported cleanup. Apps have to clean up themselves today |
| 7.3 shadcn tokens | ✅ |
| 7.5 `labels` | ❌ English only |
| 8.3 example and e2e | ✅ TanStack Start |
| 9.1 "Outgrowing this" | ⚠️ README points to Novu and betternotify for multi-channel; no mapping or steps |
| 9.3 data export | ❌ |
| 9.5 uninstall steps | ❌ |
| 10.3 exported interface | ✅ `InboxChannel` |
| 10.4 first-party adapter | ✅ `better-inbox/wraps` |
| 10.6 `./testing` suite | ❌ |

## Open questions

1. ~~**Helpdesk's future:**~~ Resolved 2026-10-20: `@byo-oss/support` is a net-new package built to this spec. It borrows from upstream `better-helpdesk` where useful, keeping the MIT notice on any code it copies.
2. **Events before or after commit:** better-auth endpoints don't expose a commit hook. Is "after the adapter write returns" good enough, or do packages need to defer until the app's transaction commits?
3. **Staff permissions:** should the vendor's own staff (support agents, operators) be modeled as better-auth admin-plugin roles, as a permission in an "operator organization", or left entirely to each app's `authorize`?
4. **The `authorize` contract (4.3):** is a boolean enough, or should it return the missing permissions so packages can show a "missing permission" hint?
5. ~~**Scope naming:**~~ Resolved 2026-10-20: every package publishes under `@byo-oss` (1.1).
