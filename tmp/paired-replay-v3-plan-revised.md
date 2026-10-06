# Migrate handoff to server preview and browser commitment

## Objective and scope

Implement fresh from `305d5ada334722b47e42e97c1524551fd6d34426` on
`NT-4307_paired-replay-redux`. Verify a clean source checkout. Do not reset/delete source,
change history, copy another branch's implementation/tests, or use ignored artifacts/logs as
completed work. Preserve local environment files. This document is the complete implementation brief.

Replace the existing handoff model:

- The server commits a page, then the browser receives an emit/skip decision to avoid duplication.

With this model:

- The server prepares and evaluates events using the Experience API's per-request preflight option.
- The server hands those prepared events to the browser, which commits them through the SDK queue.

Work is in scope when it implements that migration, removes the obsolete understanding of SDK
preflight configuration, or preserves supported consumer behavior against a regression introduced
by the migration. Behavior-preserving cleanup is acceptable. Do not label new features or unrelated
behavior changes as cleanup.

Retain useful preview data, hydration, cookie options, ordinary persistence and existing supported
rendering modes. Standalone server/browser calls continue their normal commitment behavior except
for removal of obsolete preflight knobs. Server/client/edge/ISR differences remain adaptations of
request, response and rendering boundaries, not independent delivery implementations.

Do not add new example pages, Edge browser routes, SSR rendering engines, hydration modes,
persistence frameworks, profile-readiness facilities, publication protocols, replay journals,
rendering deadlines or general navigation machinery. Existing implementations are migrated;
missing unrelated capabilities are not manufactured to satisfy a broader coverage matrix.

No documentation/README/knowledge/agent-instruction, CI, version or release changes. No PR/push.
This plan file is the authorized planning artifact. Breaking changes are communicated separately.

## Required contracts

### API and prepared handoff

Keep existing API Client profile methods. Keep `ExperienceApiClientRequestOptions` free of preflight.
Retain per-request `preflight` only on `ExperienceApiClientProfileMutationRequestOptions`, an
interface extending the shared options for `createProfile`, `updateProfile` and `upsertProfile`.
Server preparation supplies true; paired browser commitment supplies false. Reads and multi-profile
submission must not enable preflight. Remove constructor-
global and ordinary SDK-request preflight controls, including Core `forRequest` options and native
bridge/config serialization. Preserve `plainText` and its separate CORS purpose.

The two new SDK operations are:

```ts
request.prepareRequestHandoff({ routeKey, initialEvents?, page? })
web.hydrateAndTrackCurrentPage(handoff, { routeKey, buildPayload? })
```

Preparation returns `{ handoff, data? }`, where `data` is the existing full `OptimizationData` for
rendering. Extend the existing `OptimizationHandoff`; preserve its state/cache/entries types:

```ts
interface OptimizationHandoff {
  cache: OptimizationCacheMetadata
  entries?: readonly ManagedEntryHandoff[]
  state?: OptimizationSelectionState
  profileId?: string
  replay?: {
    routeKey: string
    events: readonly ExperienceEvent[]
    locale?: string
  }
}

type InitialExperienceEvent =
  | ({ type: 'identify' } & IdentifyBuilderArgs)
  | ({ type: 'track' } & TrackBuilderArgs)

interface PrepareRequestHandoffOptions {
  readonly routeKey: string
  initialEvents?: readonly InitialExperienceEvent[]
  page?: PageViewBuilderArgs
}
```

Use existing builders, wire schemas and readonly conventions. No replay version, command language,
new preview/commit API methods, or duplicate wire types. Extend existing signatures only where the
new handoff requires it; do not add public binding/readiness/identity APIs for internal convenience.

### Server preparation and identity

- The Experience API is the sole authority for profile identity. Every successful Experience
  response, including preview, supplies a profile ID. The SDK must never generate a profile ID,
  reserve a UUID identity, or fabricate a profile placeholder. SDK-generated event/message and
  tracking-context IDs are separate and remain supported.
- Reuse a known API-issued profile ID from the request context/cookie when available. Otherwise
  submit the prepared events to the existing API method without an ID and use the ID returned by
  preview. No identity-only API call or evaluation prerequisite is required before Experience events.
- Build permitted identify/track inputs in order, followed by one SDK page. Run the applicable
  server event transformations once, validate, and retain the resulting IDs, timestamps, order,
  request context and effective locale. Browser replay does not rerun those transformations;
  it applies current consent and required consent metadata, then validates. Ordinary browser
  events retain their ordinary interceptor behavior.
- Adapters supply the visible pathname plus search as routeKey, independently of analytics URL
  overrides. Account for existing rewrites and framework data/RSC transport URLs/parameters;
  those must not become the browser's route identity. Add no consumer replay-matching config.
- Preview using the existing single-profile mutation with preflight true. Make returned data
  usable by existing server getters/rendering. A recoverable preview failure retains valid
  prepared events and any known API-issued ID while rendering baseline output. If preview failed
  before an ID became known, warn and retain the replay without an ID; browser commitment can use
  the existing creation path. Invalid event/route inputs produce diagnostics and no replay. Do not reconstruct preview through a profile GET.
- A returned ID can be linked to the supplied ID. Keep the API-returned profile intact and preserve
  a captured known ID for that prepared batch's retries. Subsequent operations can follow the API
  response as normal. Do not freeze profile identity, create an alias registry, or infer different
  visitors from differing API-issued IDs within the one client.
- The existing `ctfl-opt-aid` cookie stores an API-issued profile ID only. Write a known/returned ID
  when persistence consent permits, independently of event commitment, through an available server
  response writer or normal browser persistence. Never manufacture an ID to fill a missing cookie.
  Existing cookies continue to travel back to the server. Missing first-cookie persistence when no
  ID was obtained and JavaScript is disabled is an accepted limitation, not a new feature requirement.
  Preserve useful options and deletion policy. Shared/static output contains no private identity/replay.
- Server Insights emissions require a known API-issued ID supplied to or available in the request
  context, including one obtained from a preceding Experience response. If no ID is available,
  throw the existing actionable error. Do not allocate identity, add a waiting queue or require
  stateless cancellation machinery. Request-specific server profile/evaluation state stays in
  its request context; browser stateful queues serve one client, not a multi-visitor model.

### Browser admission and commitment

Use the existing Experience queue. A prepared sequence is one indivisible queued submission;
multiple distinct submissions can remain queued subject to existing capacity/consent/reset policy.
Capture known API-issued identity and locale for retries. A missing ID does not make valid replay
unusable. Prepared request metadata remains explicit even when its ID is absent; do not distinguish
prepared work from ordinary work solely by ID presence. Use an existing browser API-issued ID if
available, otherwise let the API creation response establish it. Never create an SDK identity. Reuse existing buffering, drop reporting,
backoff, circuit and response application; add no second queue or retry timer.

| Situation                                                                                           | Required behavior                                                                                                                                                                |
| --------------------------------------------------------------------------------------------------- | -------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| Eligible matching replay                                                                            | Admit that sequence with current consent; do not also submit an ordinary page for the visit                                                                                      |
| Repeated invocation while the current admission/delivery is pending, or after its page was admitted | Share the current work and do not reapply the seed or readmit the sequence; a bounded guard is sufficient                                                                        |
| No usable replay before admission                                                                   | Permit ordinary page handling subject to current consent                                                                                                                         |
| Consent initially prevents admission, then permits it on the current route                          | Reassess the still-unadmitted sequence through the normal invocation path; do not let a cached false result permanently suppress tracking or rerun event building/transformation |
| Capacity rejects admission                                                                          | Return unsuccessful admission with diagnostics; do not substitute a page that omits the prepared prefix or create an automatic waiting facility                                  |
| A page-bearing sequence is admitted/retained, including offline or after an ambiguous failure       | The queue owns the page and retries that payload; no separate fallback page                                                                                                      |
| A distinct handoff or navigation arrives                                                            | Handle the current visit through the same Web path; preserve already admitted older work within existing queue policy                                                            |
| Stateful reset/clearing event work/teardown                                                         | Invalidate pre-reset work so its completion cannot repopulate cleared state or disturb current queued work; no visitor model or general publication framework                    |

Keep the existing `EventEmissionResult` shape. Paired delivery returns `{ accepted: true }` when
its complete batch is admitted, independently of HTTP completion; denied/capacity-rejected admission
returns `{ accepted: false }`. Paired calls do not carry response-data bookkeeping or await delivery.
Successful HTTP responses populate live SDK state through the existing queue. Ordinary event methods
retain their existing response-data and online error behavior. Repeated pending/admitted work shares completion; an unadmitted attempt may be
reassessed when its prerequisites change. Do not promise exactly-once backend commitment after a
lost response. Preserve ordinary methods' online error behavior.

Stateful Core owns one reset token shared by its Experience and Insights queues. Capture it when
async event work begins; replacing it on reset/clearing event work invalidates that work, and teardown
makes the owner inactive. Check it before response publication, after async state interceptors, and
before old completion/error/retry/finally paths can change current queue state. Settle invalidated
work without exposing its old response data. No per-queue lifetime counter, latest-request ordering
policy, visitor-switch guard, transport cancellation API or stateless reset protection is required.
This token changes for explicit clearing/teardown, not normal navigation or each API request.

Insights queues append events in order. Capture the sent prefix for each profile; on acknowledgment
remove that prefix by its length, leaving events appended while the request was pending. Reset
invalidation protects a cleared queue. Use no identity-set/filtering facility for acknowledgment.

Deduplication protects the current initialization only. It is not page-visit history. New visits,
including return navigation and distinct prepared handoffs, must remain trackable. Use existing
route tracking and at most the bounded state needed to join current work; no historical registry.

### Integration and UX preservation

- Existing handoff-bearing server/client paths must deliver their prepared batch to the browser
  actually taking over that render. In App Router, request memoization alone is not delivery:
  persistent layouts must not strand a page/slot's new handoff during soft navigation. Establish
  the concrete existing client boundary/input path in S0 before changing SDK contracts. Framework
  transport belongs to Next.js, binding to React, commitment to Web/Core. Add no middleware preview
  payload, second submission effect or app-owned replay coordinator to solve this boundary.
- Keep one page-invocation path per existing integration. With a handoff, delegate replay versus
  ordinary-page handling to Web; do not duplicate its decision in React/Next. A fresh handoff and
  changed route must be considered together before an ordinary page races the prepared prefix.
- App Router's configured handler forwards sanitized request context and reuses any existing
  API-issued cookie ID; it performs no Experience evaluation, identity allocation, committed-data
  forwarding or reconstruction GET. Cookie policy can refresh/clear an existing known ID where
  applicable. Rendering evaluates once through the request resource, and browser persistence can
  store a newly returned ID. Do not move preview into middleware or add payload transport solely
  to establish a fresh cookie without JavaScript. Preserve response composition and private/shared
  route selection.
- Preserve existing usable SSR output, first-browser rendering, merge-tag context and attribution
  while browser commitment is pending or fails. Reuse existing snapshots, hydration, controllers,
  supplied entries and normal loading/fallback behavior. Fix an introduced takeover regression
  locally; do not redesign presentation lifetimes, add a general frozen-profile system or broaden
  unrelated CSR/navigation behavior. Browser responses still populate normal live state/persistence.
- Retain useful hydration and persistence; do not introduce independent preview-cache policy work
  or wholesale persistence suppression. Identity persistence must not depend on event commitment.
  Hydrated presentation/state readiness must not be mistaken for page admission or commitment.
- Register existing observers before initial browser events/child tracking. Use existing DOM
  attribution and queues for interactions. If the migration introduces an ID-known/profile-missing
  interval that drops events, use the existing PartialProfile contract/identity dependency locally;
  add no profile-completion holding facility. Do not broaden native/standalone behavior unnecessarily.
- Remove obsolete emit/skip controls, trusted server-commit forwarding and superseded code. Keep
  useful helpers as thin delegates when they still serve the migrated flow; no deprecated inert
  options/methods or duplicate delivery implementations. Preserve standalone callbacks; paired
  initial events are supplied through the server request configuration.

## Progress and session handoff

Update this document as work happens. The phase records below are the progress ledger; a chat
report alone is insufficient for resuming in another session. S0's completed inspection is recorded
below; each subsequent phase records its own progress. Later sessions consume S0's findings
rather than performing that investigation again. Check the baseline/source correspondence on resume;
reinvestigate only an affected fact if its source or an accepted decision changes.

- **Active phase:** None; S3 is complete and accepted. S4 is not started.
- **Latest user direction:** Approve Core budget formula, commit/push the branch and accept S3 (2026-10-06). Continue without subagents.
  Tests must cover supported functionality, not obsolete options or methods being removed.
  Keep preflight off shared request options; use a narrower interface for permitted mutations.
- **Resume action:** S4 is next when the user authorizes it. S3 is accepted; do not repeat its
  implementation or unchanged checks. This plan is tracked with S3 so progress is saved with source.

Use `not started`, `in progress`, `blocked`, `reported — awaiting user`, or `accepted` in each phase's
status. Completing implementation or running checks does not mean the user accepted the phase.
Only mark the acceptance box after explicit user direction; record that direction in the evidence.
Report and stop after each phase, including verification-only phases. Do not start the next phase
merely because the current checklist is complete.

For each phase, replace its evidence placeholder with concise dated notes or a repository-relative
link to a phase report. Include the relevant commit and uncommitted scope, changes or a justified
no-change result, exact check commands/results, applicable sizes, omissions and user acceptance.
A phase can span multiple sessions; retain partial results when interrupted. The next-action field
identifies the first unfinished step and any blocker, including the command/setup state that exposed it. The active-phase entry points
to that record; accepted phase records remain available to later sessions.

Check a work item only when its stated work is complete. If a check is inapplicable, record why
before checking that item; never invent a test, runner or product feature to fill a box. A blocked
required check stays unchecked. Record explicitly deferred mobile E2E as deferred, never passed.
An accepted phase with a user-approved exception must retain that exception in its evidence.

On resume, read the active record, latest user direction, Git state and relevant changes before
acting. Reuse recorded passing evidence when its source, finalized upstream inputs and relevant
environment are unchanged. Do not rerun checks solely because a session changed. If a later
correction invalidates evidence, name the affected items and reopen only those items through a
user-authorized repair phase; preserve prior results as superseded. Add that repair phase's own
status, scoped checklist and evidence here. Do not silently reopen accepted work.

## Ownership and package phases

Perform every remaining phase directly, without subagents, as directed by the user on 2026-10-06.
Keep package implementation, review and checks strictly serial. A later session resumes the recorded
phase; it does not delegate work or repeat accepted phases.

Complete one phase, report its diff/contracts/checks/blockers and stop. Wait for the user's
instruction to advance. Read-only dependency/caller inspection is allowed; source/validation is
restricted to the current phase. Only S7 groups packages. If a later phase exposes an upstream
defect, report it and obtain direction for a dedicated repair phase; do not silently reopen it.

S0 establishes any necessary internal access/signature decisions from baseline source. New
capabilities are created in their owner phase, not assumed to exist or copied from artifacts.

### S0: starting state/design

**Status:** `accepted` · **Evidence/source:** Read-only review on 2026-10-06 of
`NT-4307_paired-replay-redux` at `305d5ada334722b47e42e97c1524551fd6d34426`; tracked source clean.
Only this plan was edited. The user accepted S0 and its recorded decisions on 2026-10-06
("I accept."), including the App Router placement and configured-handler binding.
· **Next action/blocker:** S0 has no remaining work; S1 is authorized.
Runtime/environment readiness remains unvalidated.

- [x] Verify the baseline and local source state; retain environment files.
- [x] Read applicable policies and map dependencies, ownership and necessary access/signatures.
- [x] Show minimal consumer call sites and the concrete App Router client delivery path, including
      persistent-layout soft navigation; settle only necessary deltas using the minimal/DX rubric.
- [x] Record the design/source evidence, report the phase and stop.
- [x] Record explicit user acceptance/direction before advancing.

#### Baseline findings and dependency map

These are source findings, not implementation or passing runtime evidence. Inspected root,
universal/Web/package and affected hybrid/native `AGENTS.md` policies, package manifests,
`tsconfig.base.json`, owned source and the named reference call sites. `git status --short`,
`git branch --show-current` and `git rev-parse HEAD` established the source state above. No SDK
validators, builds, installs, service requests or E2E runners were run. Local environment files
and generated assets were left untouched; they supply no completed-work evidence.

The phase order matches the manifest dependencies:

| Phase | Exact package/target                                                                                                     | SDK dependencies                                                  |
| ----- | ------------------------------------------------------------------------------------------------------------------------ | ----------------------------------------------------------------- |
| S1    | `lib/mocks` (`mocks`)                                                                                                    | Injected parsers; no SDK behavior ownership                       |
| S2    | `packages/universal/api-client` (`@contentful/optimization-api-client`)                                                  | None                                                              |
| S3    | `packages/universal/core-sdk` (`@contentful/optimization-core`)                                                          | API Client                                                        |
| S4    | `packages/universal/api-schemas` (`@contentful/optimization-api-schemas`)                                                | API Client, Core                                                  |
| S5    | `packages/node/node-sdk` (`@contentful/optimization-node`)                                                               | Core                                                              |
| S6    | `packages/web/web-sdk` (`@contentful/optimization-web`)                                                                  | Core                                                              |
| S7    | Bridge (`@contentful/optimization-js-bridge`), React Native (`@contentful/optimization-react-native`), Kotlin/Swift SDKs | Bridge and React Native use Core; Kotlin/Swift consume the bridge |
| S8    | `packages/web/frameworks/react-web-sdk` (`@contentful/optimization-react-web`)                                           | Web                                                               |
| S9    | `packages/web/frameworks/nextjs-sdk` (`@contentful/optimization-nextjs`)                                                 | Node, React Web; reach Web/Core through existing pass-throughs    |
| S10   | `packages/web/preview-panel` (`@contentful/optimization-web-preview-panel`)                                              | Core, Web                                                         |

#### Recorded implementation decisions

The following decisions fill baseline gaps within the migration. Their implementation belongs to
the named later phase. They authorize no extra framework, queue, configuration mode or test suite.

- **S1 — authored mock:** `lib/mocks/src/experience-handlers.ts:121` evaluates responses and
  unconditionally adds known IDs/identified state. The two single-profile POST handlers must
  recognize `type=preflight` and evaluate against existing state without committing those writes.
  Leave reads, multi-profile behavior and generated fixture files alone. Direct mock validation is
  scoped typecheck under its local policy; behavioral evidence comes from consuming tests later.
- **S2 — existing transport:** `ExperienceApiClient.ts:410` already delegates supplied-ID upsert
  to `updateProfile`; no new method is needed. Its shared URL builder at line 490 applies preflight
  to reads and multi-profile calls as well. Remove the constructor default and restrict enabling
  preflight to the existing single-profile mutation path, including runtime option forwarding.
  The shared options interface has no preflight field; the single-profile mutation options interface
  extends it without adding methods or runtime classes.
  The v3 backend permits one ordered mutation with up to 200 events and 50 identifies; prepared
  input must fit one supported request, not be silently split into several page-bearing batches.
- **S3 — preparation access:** `CoreBase.ts:210` already exposes the API, builders and interceptors.
  `CoreStatelessRequest.ts:423` already merges request event context; its send helper at line 454
  demonstrates consent stamping, interception, validation, upsert and request-local data application.
  Implement preparation there using those facilities. `routeKey` belongs on preparation options,
  not ordinary `forRequest` construction. Use a known API-issued ID or obtain one from the preview
  response; no SDK profile-ID generation or placeholder is allowed. Keep the received profile intact.
  A failed preview can retain valid events with no ID and recover through browser API creation.
- **S3 — queue delta:** `queues/ExperienceQueue.ts:71` currently retains a set of individual offline
  events, sends online events directly, and resolves identity again at flush. Extend this existing
  queue to retain a prepared submission atomically with its identity/locale, including after a
  failed first online attempt. Do not retain paired work in a parallel collection or reintercept it
  on retry. Reject a prepared submission that cannot fit capacity intact; do not truncate its
  prefix. Preserve ordinary event drop/error behavior. Permit the minimum protected batch-admission
  access from Core's existing stateful emitter to Web; there is no need for an application-facing
  batch/replay-delivery method. Reuse `QueueFlushRuntime` and existing clear/reset/teardown points.
- **S3/S6 — early interactions and reset:** `queues/InsightsQueue.ts:88` currently requires the
  full profile signal, although the wire schema already accepts `PartialProfile`. Pass the
  existing Core anonymous-ID source into that queue as a fallback for `{ id }` when paired
  continuity is known but live data is absent. A consented paired identity must also work in memory when persistence is denied;
  cookie/storage access is not its only source. Core owns one stateful reset token and queue
  admission; Web owns browser persistence. No multi-visitor queue or stateless lifecycle machinery
  is required. Server Insights must error when no request-context API-issued ID is available.
  Do not fabricate a profile, hold interactions for a profile GET, or modify native UI behavior.
- **S3/S5 — common data:** `core-sdk/src/handoff.ts:137` owns the common handoff and cache safety;
  extend its existing safety check to exclude the added identity/replay from public/static output.
  `node-sdk/src/handoff.ts:29` is a pure data converter. Keep it pure; shallow-compose prepared
  handoff and render data/managed entries rather than inventing another Node request engine.
- **S6 — browser coordination:** `ContentfulOptimization.ts:509` and Core's
  `tracking/AcceptedCurrentStateTracker.ts` already handle the current route. Reuse that behavior
  for ordinary fallback. Prepared page `messageId` already distinguishes fresh preparations on
  the same route; add no handoff ID or history registry. Join repeated current work before identity
  or live-state seeding. Existing `handoff.ts` hydration and the combined operation must share this
  eligibility/current-work decision, including route and the stateful reset boundary. Permit only necessary
  extensions to existing hydration helper signatures; React must not implement an independent
  identity/repeat gate. An unadmitted consent-blocked attempt remains reassessable.
- **S6/S8 — presentation and bindings:** Web's existing `runtime.ts`, `analytics.ts` and hydration
  helpers cover live/snapshot/analytics surfaces. Add combined-operation pass-through and its
  inert snapshot counterpart there. React's `OptimizationProvider.tsx:214` already creates the
  first-render snapshot, hydrates before publishing the live runtime, and installs callbacks.
  Keep the provider state-only and preserve that readiness boundary without awaiting Experience
  commitment. Its paired state hydration must use the Web decision above before reseeding.
  Replace page sequencing in `auto-page/useAutoPageEmitter.ts`, `root/OptimizationRoot.tsx` and
  `root/OptimizationAnalyticsRoot.tsx` with the single Web operation. Keep standalone callbacks.
  No extra browser-request readiness API, presentation subsystem or React delivery queue is needed.
- **S6/S9 — cookies:** `packages/web/web-sdk/src/lib/cookies.ts` and
  `packages/web/frameworks/nextjs-sdk/src/cookies.ts` remain their runtime writers.
  Browser `cookie.expires` and the bound Next config use days; the low-level server writer uses
  `Date`/seconds. Reuse the existing days-to-server conversion and preserve those meanings. Add
  only the missing path/same-site/secure forwarding needed for the agreed shared policy. Use the
  same domain/path when deleting. Writers use API-issued IDs only; a known request ID or the
  preview/live response ID can be persisted through existing facilities. Keep returned profile data
  and normal browser response persistence intact; no frozen-identity policy or alias registry is
  needed. If no ID is known, leave it absent until an API response supplies it. Preserve low-level
  server-only options; paired cookies remain browser-readable. Add no second cookie or no-JavaScript
  identity-acquisition workaround.
- **S9 — identity handler/configuration:** Reuse `request-handler.ts`, its effective override-header
  composition and cookie helpers. In the existing server binding, return a configured
  `requestHandler` using the already-bound SDK/consent/cookie configuration; this is a configured
  form of the existing handler, not another operation or middleware engine. It reuses/forwards
  an existing API-issued cookie ID, applies cookie policy when appropriate and sanitizes request
  context, with no Experience evaluation or identity allocation. A fresh profile ID comes from
  rendering's preview response and can be persisted by the browser. There is no mandatory ID
  before evaluation and no special fresh-cookie guarantee without JavaScript. Remove trusted committed-data forwarding/GET from
  `app-router-request-handoff.ts` and `app-router-request-runtime.tsx`. Extend existing request
  configuration/options only for paired `initialEvents` and page inputs; a resolver can use the
  existing request context when a maintained scenario derives inputs from its URL. Remove paired
  use of the browser `beforeInitialPage` callback; preserve its standalone browser behavior.
- **S9 — route source:** Use the sanitized visible request URL in App Router and Edge, and
  Pages Router's existing `context.resolvedUrl` adaptation in `pages-router-server.ts`. Normalize
  the visible pathname/search consistently with the existing client router inputs; exclude actual
  framework-only transport parameters such as `_rsc`. Preserve visible query values across rewrites
  and keep analytics URL overrides independent. No replay-matching option is exposed to consumers.

#### App Router delivery through existing boundaries

Observed path: the reference `(request)/layout.tsx` owns `RequestOptimizationRoot`;
`lib/optimization-client.ts` makes that persistent client root read router changes and emit pages.
Meanwhile `EntryCard.tsx` uses fresh server `request.OptimizedEntry` data on the route pages.
`app-router-request-runtime.tsx:145` memoizes request inputs but its existing
`request.NextAppAutoPageTracker` at line 217 passes only the emit/skip flag. This does not deliver a
fresh page's prepared batch through a retained layout.

Use the existing state-only `request.OptimizationProvider` in the persistent layout, and the
existing `request.NextAppAutoPageTracker` in `(request)/template.tsx`, which participates in each
route render. Extend that tracker to carry the memoized handoff and its authoritative route input
through React's existing emitter to Web. The provider and server entries await the same request
resource; the tracker is the sole page-invocation path for those routes. It must receive the fresh
handoff together with route identity, never emit an ordinary page first from a retained-layout
hook. Do not key/remount the live SDK or put delivery into each entry component.

This is a small framework-boundary change, not a new route or SDK component. It replaces the
layout-root emitter; do not mount both. Private slots whose existing root rerenders with the slot
can continue using that root's combined path. Pages Router already passes changing page props to
its mounted `_app` root, so it does not need this template arrangement. Shared/static routes retain
their explicit, visitor-independent surfaces.

**Ownership:** S8 extends the existing React tracker/emitter input; S9 makes the request tracker
supply the handoff/route together; I3 replaces the private layout root with the existing provider
and adds the small template call site. The baseline App Router `AGENTS.md` rule that the layout's
request root owns sequencing is superseded specifically by this migration decision accepted in S0; its prohibition
on duplicate trackers, app-owned request plumbing and lower-layer imports remains. Do not edit
`AGENTS.md` as part of this milestone.

#### Minimal proposed consumer call sites

These snippets describe the target contract, not APIs already present at the baseline. Existing
request/context/consent and render functions remain consumer-owned.

```ts
// Node + Web server: replace identify/page commitment with one preparation.
const request = sdk.forRequest({ consent, profile, eventContext, locale })
const { handoff, data } = await request.prepareRequestHandoff({
  routeKey: `${url.pathname}${url.search}`,
  initialEvents: userId ? [{ type: 'identify', userId, traits: { identified: true } }] : [],
})
// Write handoff.profileId with the existing response writer when consent permits.
// Keep existing rendering/bootstrap using data, and transfer handoff in its existing payload.
```

```ts
// Web: call after existing observers/state subscriptions are installed.
await web.hydrateAndTrackCurrentPage(handoff, {
  routeKey: `${window.location.pathname}${window.location.search}`,
  buildPayload: () => ({ properties: { title: document.title } }),
})
// Existing consent/navigation callbacks invoke this same path, rather than another page().
```

```tsx
// App Router: aliases from the existing server binding, finalized in S9.
const optimization = bindNextjsAppRouterServerOptimization(serverConfig)
export const { requestHandler: proxy } = optimization
export const {
  OptimizationProvider: RequestOptimizationProvider,
  NextAppAutoPageTracker: RequestPageTracker,
} = optimization.request
// Persistent private layout: <RequestOptimizationProvider>{children}</RequestOptimizationProvider>
// Private route template: <><RequestPageTracker />{children}</>
```

Angular already has `TransferState`, a snapshot runtime and its server initializer in
`src/app/services/optimization.ts`; I2 transfers the prepared handoff alongside the existing
snapshot/baselines and replaces `skipNextPage` with the combined Web call. Node/Web already has an
Express response and EJS bootstrap in `src/app.ts`/`src/index.ejs`; I1 uses them without adding SSR.
Pages Router's existing `createRequestHandoff(context, options)` and Edge's
`createEdgeRequestHandoff(options)` remain thin adapter calls into Core preparation. The existing
Edge reference exercises handoff routes; no browser app capability is added to create extra coverage.

#### Validation and API evidence for later sessions

- Exact SDK filters/scripts are in the manifest table above. Package typechecks resolve imported
  SDK source through `tsconfig.base.json`; targeting one package means invoking only its checks,
  not pretending its imported types can be excluded. Keep builds/artifact mutations serial.
- `pack:pkgs` runs `scripts/pack-pkgs.sh` against accepted built outputs; it does not build SDKs.
  The ordinary package `build` scripts clean their own outputs/cache first. Bridge `postbuild`
  copies native resources; the existing S7 direct native unit commands avoid rebuilding the bridge.
- Preview panel's `test:unit` is a placeholder. In S10 record that as inapplicable evidence; do not
  claim units passed or add tests solely to replace the placeholder in a verification-only phase.
- T6 uses `pnpm implementation:run -- node-sdk implementation:test:e2e:run` after documented
  prerequisites are confirmed. The root `test:e2e:node-sdk` includes automatic setup and must not
  be used to bypass the plan's setup boundary. No E2E is run during S0 or SDK/reference phases.
- Native preflight fields are owned by bridge `src/index.ts`, Kotlin
  `core/OptimizationConfig.kt` and Swift `Core/OptimizationConfig.swift`. React Native has no
  local preflight field in `src`; S7 removes inherited references only where needed.
- Backend source was inspected with `git show` at `../ninetailed-new` commit
  `fd05a3c59501c328ef62da6c52455936cb438f4d`: v3 `UpdateProfileV3Handler` supplies the request ID
  and `skipWrite`; `profiles/storage/commands/UpsertProfile` evaluates an empty profile if no record
  exists; `ProfileEffects` and `ProfileStorageEffects` preview implementations omit profile/event
  writes and alias claims. This supports preview and later supplied-ID commitment. It certifies
  no deployed environment. Later consumer validation uses the contract-aligned local mocks.

S0 concludes with these source-backed decisions and explicit deferred runtime verification. Later
owners inspect their own phase source for implementation details; they do not repeat the dependency
survey, reopen accepted design without a demonstrated mismatch, or treat this inspection as a
substitute for their required scoped validation.

### S1: `lib/mocks`

**Status:** `accepted` · **Evidence/source:** Completed directly on 2026-10-06 from
`305d5ada334722b47e42e97c1524551fd6d34426` with clean tracked source.
At S1 completion, the only uncommitted source change was `lib/mocks/src/experience-handlers.ts`; this plan
records progress and the user's direct-execution instruction.
The user accepted S1 on 2026-10-06 ("Accepted. Mark it as such and begin S2.").
S1 source was committed together with S2 in `c88deb902385c3c04245c506490a3659110c2ecb`
on 2026-10-06; the tracked working tree is clean.
· **Next action/blocker:** S1 has no remaining work; behavioral consumer evidence is deferred as
required and S2 is authorized.

**Remove:** Preflight requests' unconditional stored-state/commit writes.

**Replace/add:** Minimal authored single-profile handler support for evaluate-without-commit under supplied ID.

**Retain/exclude:** Generated fixtures unchanged; no mock-owned tests or mock feature work.

- [x] Inspect the authored handler and confirm the minimal scoped change.
- [x] Correct evaluate-without-commit behavior without editing generated fixtures.
- [x] Format and run applicable scoped source checks; add/run no mock-owned behavior tests.
- [x] Record the change and consuming phase that will provide incidental validation, apply the
      minimal/DX rubric, report and stop. Do not claim consumer validation already passed.
- [x] Record explicit user acceptance/direction before advancing.

**Change/evidence:** The existing response builder accepts one internal `commit` boolean, defaulting
to true. Only the two single-profile POST callers derive false from `type=preflight`. Evaluation
still reads prior identified state, applies the supplied events and returns the profile ID and
evaluated fixture response; writes to both `knownProfileIds` and `identifiedState` are conditional.
Ordinary calls retain the existing default. GET, multi-profile submission, CORS and generated
fixtures were not edited. The diff adds no exported interface, separate evaluator or new tests.

**Checks on 2026-10-06:**

- `pnpm exec prettier lib/mocks/src/experience-handlers.ts tmp/paired-replay-v3-plan-revised.md --write --ignore-path /dev/null` — passed.
- `pnpm --filter mocks typecheck` — passed (`tsc --noEmit`, exit 0).
- `pnpm exec prettier lib/mocks/src/experience-handlers.ts tmp/paired-replay-v3-plan-revised.md --check --ignore-path /dev/null` — passed.
- Read-only diff/status review — only the intended authored source file changed; generated
  fixtures, dependency manifests and other packages are unchanged.

No mock-owned tests were added or run. Standalone lint, unit/build/budget and consumer/E2E runs
were omitted under the mocks' typecheck-only direct-validation exception and phase boundaries.
This does not establish passing behavioral evidence. Core's existing test setup
(`packages/universal/core-sdk/src/test/setup.ts:13`) installs these handlers, so S3 consumer
preparation/commitment coverage can provide incidental use. Later Web consumer phases use the
same authored handlers through `lib/mock-server/src/server.ts`; run only their planned scenarios,
without adding mock-owned tests or broadening coverage solely for this change.

### S2: API Client

**Status:** `accepted` · **Evidence/source:** Completed directly on 2026-10-06 at
`305d5ada334722b47e42e97c1524551fd6d34426`. This phase changes only API Client's
`src/experience/ExperienceApiClient.ts` and its existing package-owned test file. The user requested
the current source changes be committed on 2026-10-06. S1/S2 source is recorded in
`c88deb902385c3c04245c506490a3659110c2ecb`
(`refactor(api-client)!: scope preflight to profile mutations`); staged-file lint/format and
commit-message hooks passed, and the tracked working tree is clean. At that commit, this plan was local; it is tracked with the accepted S3 milestone.
The user approved S2 on 2026-10-06 ("Count S2 as approved. Then begin S3.").
· **Next action/blocker:** S2 has no remaining work; S3 is authorized.

**Remove:** Constructor-global preflight config/default.

**Replace/add:** Restrict existing per-call preflight to single-profile mutations through
`ExperienceApiClientProfileMutationRequestOptions` extending the shared request options; owned
tests cover supported transport only. Add no methods or runtime classes.

**Retain/exclude:** Existing methods/wire schemas/plainText/timeouts/retries; no handoff/queue logic.

- [x] Inspect this phase's policies/source and confirm the smallest delta and consumer call site.
- [x] Complete only the owned scope above, with necessary package-owned unit coverage; record a
      justified no-change result for verification-only work.
- [x] Format/lint this phase's source and tests.
- [x] Run this phase's applicable typecheck.
- [x] Run this phase's applicable package unit tests.
- [x] Build only this phase's package output, using finalized upstream artifacts.
- [x] Report and enforce this phase's defined bundle budgets; record inapplicable checks explicitly.
- [x] Apply the minimal/DX rubric, record exact evidence/blockers, report and stop.
- [x] Record explicit user acceptance/direction before advancing.

**Final change:** The shared `ExperienceApiClientRequestOptions` contains no `preflight` field.
`ExperienceApiClientProfileMutationRequestOptions` extends it with the retained optional field,
and is used by `createProfile`, `updateProfile` and `upsertProfile`. Constructor configuration
extends shared options without preflight; the constructor field/default was deleted. Profile reads
forward locale only to URL construction, and multi-profile calls disable preview in the existing
transport helper. Existing methods, wire schemas, plainText behavior, locale, timeout/retry handling
and dependency/entry graphs are preserved. There are no additional methods, runtime classes,
deprecated options or handoff mechanisms.

**Minimal call site:** Use the existing client and methods without a global preview toggle:

```ts
const client = new ExperienceApiClient({ spaceId, environment })
await client.upsertProfile({ profileId, events }, { preflight: true })
await client.upsertProfile({ profileId, events }, { preflight: false })
```

**Coverage:** Package-owned transport cases cover preview, explicit commitment and normal default
commitment on single-profile methods, with locale forwarding. Existing normal read/multi-profile
cases also check that preview is absent. Cases supplying obsolete options were removed as directed;
no compatibility tests or tests for deleted constructor/request settings remain. No E2E or
mock-owned tests were added. The final package suite passes 102 tests in 14 files.

**Final checks on 2026-10-06:**

- `pnpm exec prettier packages/universal/api-client/src/experience/ExperienceApiClient.ts packages/universal/api-client/src/experience/ExperienceApiClient.test.ts tmp/paired-replay-v3-plan-revised.md --write --ignore-path /dev/null` — passed.
- `pnpm exec eslint packages/universal/api-client` — passed.
- `pnpm --filter @contentful/optimization-api-client typecheck` — passed.
- `pnpm --filter @contentful/optimization-api-client test:unit` — passed; 102 tests, no skips/failures.
- `pnpm --filter @contentful/optimization-api-client build` — passed; package clean/build and dual declarations emitted.
- `pnpm --filter @contentful/optimization-api-client size:report` — passed; all defined entries reported below.
- `pnpm --filter @contentful/optimization-api-client exec build-tools bundle-size` — passed on the just-built output, without another rebuild.
- `pnpm exec prettier packages/universal/api-client/src/experience/ExperienceApiClient.ts packages/universal/api-client/src/experience/ExperienceApiClient.test.ts tmp/paired-replay-v3-plan-revised.md --check --ignore-path /dev/null` — passed.

| Bundle       | Gzip bytes | Budget bytes | Headroom bytes |
| ------------ | ---------: | -----------: | -------------: |
| `guards.cjs` |        525 |          550 |             25 |
| `guards.mjs` |        197 |          200 |              3 |
| `index.cjs`  |      4,953 |        5,000 |             47 |
| `index.mjs`  |      4,581 |        4,700 |            119 |

The final interface refinement required regenerated declarations; their inspection confirmed
the new mutation options are exported and used by the three intended methods, while constructor
and multi-profile signatures use shared options. The type-only refinement did not change measured
runtime bundle sizes. No budgets, manifests or generated source fixtures were edited. Checks were
repeated only for the user-directed test/type refinements; the results above describe the final
source. No other package, reference implementation, pack/install or E2E command ran.

**Later owners:** S3 removes Core's obsolete references to preflight in constructor/ordinary request
configuration and supplies preview through the narrowed API method options. S7 removes inherited
bridge/native configuration references. Those planned changes are not compatibility shims or
additional S2 work; the package-only result is not a claim that the unfinished repository-wide
migration is ready to publish.

### S3: Core

**Status:** `accepted` · **Evidence/source:** Authorized from
`c88deb902385c3c04245c506490a3659110c2ecb`. The user directed corrected API-only identity,
stateful reset invalidation and acknowledgment handling on 2026-10-06. Changes remain Core-only
and ready to commit with this plan. Corrected implementation and functional checks are complete.
The user accepted S3 on 2026-10-06 and approved every Core budget as
`ceil((measuredGzipBytes + 100) / 100) * 100`. Those exact values were applied and enforcement passed.
· **Next action/blocker:** Save the milestone to origin as requested. No S3 blocker remains;
S4 still requires user direction.

**Remove:** Core constructor/ordinary preflight knobs; SDK profile-ID generation and placeholders;
per-queue lifetime counters, latest-response ordering policy and general visitor-identity guards.

**Replace/add:** Request preparation and common handoff fields; API-issued profile identity with
ID-less failed-preview replay; atomic retention/admission through the existing queue; one SDK-owned
stateful reset token shared by both queues; acknowledged-prefix removal for Insights. Necessary
protected admission remains for Web. No additional application-facing delivery API is added.

**Retain/exclude:** Existing builders, consent, API methods, pure resolvers, cache rules and snapshots.
Server Insights requires an API-issued ID in the request context. Server state is request-local;
stateless calls acquire no reset machinery. No cookie/HTTP/DOM/framework source is changed in S3.

- [x] Inspect this phase's policies/source and confirm the corrected scope and consumer call site.
- [x] Complete the corrected Core-owned implementation and meaningful package-owned unit coverage.
- [x] Format/lint corrected Core source/tests.
- [x] Run Core typecheck.
- [x] Run relevant Core unit checks, recording final coverage and any reused unchanged evidence.
- [x] Build Core outputs and inspect approved emitted contracts.
- [x] Report/enforce Core's explicitly approved bundle budgets on corrected output.
- [x] Apply the meaningful-correctness/DX rubric, record evidence/blockers, report and stop.
- [x] Record explicit user acceptance before advancing.

**Implementation constraints for dependent phases:**

```ts
const request = core.forRequest({ consent, profile, eventContext, locale })
const { handoff, data } = await request.prepareRequestHandoff({ routeKey, initialEvents })
```

Profile identity can be absent before preview. A successful response supplies the ID; an unsuccessful
preview retains the validated replay and any already-known ID, with no fabricated identity. Full
server evaluation remains separate from browser handoff; rendering state/entries use existing
composition facilities. Web's protected admission accepts optional `profileId` for this recovery
path. Its prepared request metadata remains distinguishable from ordinary queued events even when
ID is absent. Once the API supplies an ID, reuse it normally.

Stateful reset invalidates previously-started event work, including a response awaiting async state
interceptors. Invalidated work must not repopulate cleared state, expose old response data, erase
new queued events or change current retry/flush state. Use one token from the stateful SDK; no
per-queue epoch or latest-request ordering rule is introduced. Explicit clearing of queued event
work and teardown use this same boundary. Normal response ordering, navigation and stateless
request processing retain their existing behavior.

The Insights queue is append-only between snapshot and acknowledgment. Remove the acknowledged
prefix by length; leave later appended events queued. No Set/filter membership algorithm is needed.
Existing reset invalidation handles a queue that was cleared meanwhile.

**Baseline measurement record:** The pre-S3 runtime-size baseline
remains applicable: it was rebuilt from `c88deb902385c3c04245c506490a3659110c2ecb` in isolated
`tmp/core-size-baseline-c5uxxkjg/packages/universal/core-sdk`, with the original runtime config,
entries and dependencies; only DTS generation was disabled for this JS size comparison. Its CJS
and ESM main gzip sizes are 22,918 / 22,209 bytes. No baseline implementation was copied into
product source, and no upstream package was rebuilt for the comparison. Do not repeat it merely
to resume; measure the corrected current Core output against existing budgets.

**Final corrections implemented:**

- No SDK-generated profile UUID, identity allocator, continuation-ID field or partial-profile
  placeholder remains. Preparation uses request-local profile identity and the successful API
  response; failed preview with no known ID retains an ID-less replay. SDK message/tracking IDs
  remain separate. Server Insights continues to throw the existing request-context ID errors.
- One stateful SDK-owned reset token is shared by both queues. Per-queue lifetime counters,
  latest-request response ordering and visitor-ID equality checks were deleted. Invalidated work
  does not publish old data, start queue retries, or disturb current flush/queued state. Stateless
  request clients have no reset machinery.
- Insights acknowledgment uses prefix length, with no identity Set/filter. Experience uses one FIFO
  collection of stored batches. Ordinary offline events coalesce at admission; its grouping-pass
  helper was removed. Acknowledgment preserves later appends, including when ordinary capacity
  trimming occurred during an earlier request. Bound prepared batches remain indivisible.
- Paired admission returns its result immediately. Per-call delivery-result collection and the
  additional flush wrapper were removed; the existing flush path owns transport/retry/publication.
  Ordinary APIs retain their response behavior.
- Plan instructions were replaced throughout: identity/context handlers reuse existing API-issued
  cookies and never allocate identity; browser persistence can store a new API-issued ID. There is
  no fresh-cookie/no-JavaScript requirement or multi-visitor server/client queue model. Correctness
  safeguards must address a supported flow with a meaningful consumer/end-user consequence.

**Corrected functional evidence (2026-10-06):**

- `pnpm exec eslint packages/universal/core-sdk` — passed on corrected source; final touched-file
  checks also passed after local fixes.
- `pnpm --filter @contentful/optimization-core typecheck` — passed on final source.
- `pnpm --filter @contentful/optimization-core test:unit` — 397 tests in 37 files passed after the
  identity/reset changes. Subsequent queue/admission simplifications were verified with
  `pnpm --filter @contentful/optimization-core exec rstest src/CoreStateful.handoff.test.ts src/CoreStateful.test.ts src/queues/ExperienceQueue.test.ts`;
  all 64 affected cases passed, including one added in-flight capacity regression. Unaffected
  package cases are unchanged. No tests cover the deleted identity allocator or ordering policy.
- `pnpm --filter @contentful/optimization-core build` — passed, including clean and declaration
  emission. The protected admission signature has optional ID; ordinary API and snapshot contracts
  remain intact. No other package was built or validated.
- Changed Core files and the plan were formatted/checked using scoped Prettier. There were no
  mock-owned tests, consumer units, E2E runs, native validation or documentation/CI changes.

**Final bundle evidence:** `pnpm --filter @contentful/optimization-core size:report` reported all
entries. The user approved the formula above on 2026-10-06. After updating the manifest,
`pnpm --filter @contentful/optimization-core exec build-tools bundle-size` passed for all entries
on the same current outputs, without a rebuild. The table records the approved policy and headroom.

| Bundle                | Measured gzip | Approved budget | Headroom |
| --------------------- | ------------: | --------------: | -------: |
| `index.cjs`           |        24,596 |          24,700 |      104 |
| `index.mjs`           |        23,879 |          24,000 |      121 |
| `bridge-support.cjs`  |           852 |           1,000 |      148 |
| `bridge-support.mjs`  |           622 |             800 |      178 |
| `runtime.cjs`         |         3,502 |           3,700 |      198 |
| `runtime.mjs`         |         3,954 |           4,100 |      146 |
| `entry-source.cjs`    |         1,983 |           2,100 |      117 |
| `entry-source.mjs`    |         1,724 |           1,900 |      176 |
| `preview-support.cjs` |         5,158 |           5,300 |      142 |
| `preview-support.mjs` |         4,551 |           4,700 |      149 |

The runtime revision is smaller than the superseded S3 draft by 163 CJS / 145 ESM gzip bytes.
The reduction is modest: the necessary paired flow still adds 1,678 / 1,670 bytes over the pre-S3
baseline. Runtime source changes are tracked separately from tests: the current diff has 10 runtime
files, net 256 added lines over pre-S3, versus net 278 in the earlier draft. Test additions do not
contribute to the production bundles. Remaining cost is current-change feature cost, with unchanged
entry/dependency graphs and unchanged sizes for all other measured entries. No budget-only code
obfuscation or behavior removal was made. Budgets changed only under the user's exact formula;
all 10 were evaluated, including the reductions for runtime/preview ESM.

**Save/acceptance record:** The user requested the S3 source, approved budget changes and accepted
plan be committed and the branch pushed to `origin`. This plan is deliberately tracked despite
its `tmp/` location; size-analysis snapshots remain ignored. The milestone commit records the
complete accepted source state. Git history/upstream tracking provide its commit and remote-save
identity on resume. No later phase begins with this acceptance.

**Resume:** S3 is complete and accepted. S4 is not started. Further source changes invalidate only
affected evidence and require the appropriate authorized phase. Do not restore superseded ID,
visitor or response-ordering mechanisms to satisfy another item.

### S4: schema facade

**Status:** `not started` · **Evidence/source:** — · **Next action/blocker:** —

**Remove:** Deleted owned re-exports only if present.

**Replace/add:** Necessary pass-through repair; otherwise verification-only.

**Retain/exclude:** No new schema, facade removal or compatibility shim.

- [ ] Inspect this phase's policies/source and confirm the smallest delta and consumer call site.
- [ ] Complete only the owned scope above, with necessary package-owned unit coverage; record a
      justified no-change result for verification-only work.
- [ ] Format/lint this phase's source and tests.
- [ ] Run this phase's applicable typecheck.
- [ ] Run this phase's applicable package unit tests.
- [ ] Build only this phase's package output, using finalized upstream artifacts.
- [ ] Report and enforce this phase's defined bundle budgets; record inapplicable checks explicitly.
- [ ] Apply the minimal/DX rubric, record exact evidence/blockers, report and stop.
- [ ] Record explicit user acceptance/direction before advancing.

### S5: Node SDK

**Status:** `not started` · **Evidence/source:** — · **Next action/blocker:** —

**Remove:** Obsolete owned options/usages/assertions.

**Replace/add:** Consume inherited Core request preparation; compose its handoff with existing pure `createRequestHandoffFromData` where render data is needed.

**Retain/exclude:** Keep helper/API unchanged if ordinary shallow composition suffices. No Node delivery engine, browser state or generic cookie framework.

- [ ] Inspect this phase's policies/source and confirm the smallest delta and consumer call site.
- [ ] Complete only the owned scope above, with necessary package-owned unit coverage; record a
      justified no-change result for verification-only work.
- [ ] Format/lint this phase's source and tests.
- [ ] Run this phase's applicable typecheck.
- [ ] Run this phase's applicable package unit tests.
- [ ] Build only this phase's package output, using finalized upstream artifacts.
- [ ] Report and enforce this phase's defined bundle budgets; record inapplicable checks explicitly.
- [ ] Apply the minimal/DX rubric, record exact evidence/blockers, report and stop.
- [ ] Record explicit user acceptance/direction before advancing.

### S6: Web SDK

**Status:** `not started` · **Evidence/source:** — · **Next action/blocker:** —

**Remove:** Emit/skip types/fields/validators/branches and obsolete owned sequencing.

**Replace/add:** Combined operation, bounded current-work protection, identity/hydration reuse and queue delegation; necessary runtime/analytics pass-through and local introduced-regression fixes.

**Retain/exclude:** Browser cookies/storage/interaction and entry-controller code stay here; pure resolution stays Core. Preserve existing useful hydration/persistence and standalone callbacks; no HTTP/React/cache-resource logic.

- [ ] Inspect this phase's policies/source and confirm the smallest delta and consumer call site.
- [ ] Complete only the owned scope above, with necessary package-owned unit coverage; record a
      justified no-change result for verification-only work.
- [ ] Format/lint this phase's source and tests.
- [ ] Run this phase's applicable typecheck.
- [ ] Run this phase's applicable package unit tests.
- [ ] Build only this phase's package output, using finalized upstream artifacts.
- [ ] Report and enforce this phase's defined bundle budgets; record inapplicable checks explicitly.
- [ ] Apply the minimal/DX rubric, record exact evidence/blockers, report and stop.
- [ ] Record explicit user acceptance/direction before advancing.

### S7: bridge/native group

**Status:** `not started` · **Evidence/source:** — · **Next action/blocker:** —

**Remove:** Each package's preflight config/serialization and corresponding obsolete assertions.

**Replace/add:** Minimal caller/empty-config mapping repairs only; React Native is verification-only if no local field exists.

**Retain/exclude:** No native handoff, UI/lifecycle/storage changes. Bridge owns serialization; Kotlin/Swift own their config; checks serial.

- [ ] Inspect this phase's policies/source and confirm the smallest delta and consumer call site.
- [ ] Complete only the owned scope above, with necessary package-owned unit coverage; record a
      justified no-change result for verification-only work.
- [ ] Format/lint this phase's source and tests.
- [ ] Run scoped bridge and React Native checks serially; build the bridge once and use its
      generated native copies, following the S7 commands in the validation policy.
- [ ] Run Android SDK unit/compile verification; record command/result or blocker.
- [ ] Run iOS SDK unit/compile verification; record command/result or blocker.
- [ ] Report/enforce React Native's defined budgets; record bridge/native budget inapplicability.
      No local mobile E2E or build-for-testing.
- [ ] Apply the minimal/DX rubric, record exact evidence/blockers, report and stop.
- [ ] Record explicit user acceptance/direction before advancing.

### S8: React Web

**Status:** `not started` · **Evidence/source:** — · **Next action/blocker:** —

**Remove:** Owned emit/skip controls/re-exports and paired callback sequencing that would duplicate server-prepared inputs.

**Replace/add:** One invocation path to finalized Web behavior; only binding/context changes needed for migrated handoffs and existing UX.

**Retain/exclude:** Provider is state-only. Next-named router hooks in this package belong here. Web owns controller semantics; no React queue/identity/storage machinery.

- [ ] Inspect this phase's policies/source and confirm the smallest delta and consumer call site.
- [ ] Complete only the owned scope above, with necessary package-owned unit coverage; record a
      justified no-change result for verification-only work.
- [ ] Format/lint this phase's source and tests.
- [ ] Run this phase's applicable typecheck.
- [ ] Run this phase's applicable package unit tests.
- [ ] Build only this phase's package output, using finalized upstream artifacts.
- [ ] Report and enforce this phase's defined bundle budgets; record inapplicable checks explicitly.
- [ ] Apply the minimal/DX rubric, record exact evidence/blockers, report and stop.
- [ ] Record explicit user acceptance/direction before advancing.

### S9: Next.js SDK

**Status:** `not started` · **Evidence/source:** — · **Next action/blocker:** —

**Remove:** Trusted server-data options/header/parser, reconstruction GET, middleware evaluation and owned emit/skip controls.

**Replace/add:** Existing paired helpers delegate to Core; existing-ID/context configured handler, request memoization and actual client handoff transport; minimal existing writer/input/config extensions.

**Retain/exclude:** Own HTTP/context/cookie adaptation; preserve normal server-only helpers, cache/private-slot behavior, rewrites/redirects and Edge safety. No copied Core/Web/React logic.

- [ ] Inspect this phase's policies/source and confirm the smallest delta and consumer call site.
- [ ] Complete only the owned scope above, with necessary package-owned unit coverage; record a
      justified no-change result for verification-only work.
- [ ] Format/lint this phase's source and tests.
- [ ] Run this phase's applicable typecheck.
- [ ] Run this phase's applicable package unit tests.
- [ ] Build only this phase's package output, using finalized upstream artifacts.
- [ ] Report and enforce this phase's defined bundle budgets; record inapplicable checks explicitly.
- [ ] Apply the minimal/DX rubric, record exact evidence/blockers, report and stop.
- [ ] Record explicit user acceptance/direction before advancing.

### S10: preview panel

**Status:** `not started` · **Evidence/source:** — · **Next action/blocker:** —

**Remove:** Deleted local symbol usage only if required.

**Replace/add:** Minimal caller repair if necessary; otherwise verification-only.

**Retain/exclude:** Keep panel/explicit live updates; no new features or lifecycle changes.

- [ ] Inspect this phase's policies/source and confirm the smallest delta and consumer call site.
- [ ] Complete only the owned scope above, with necessary package-owned unit coverage; record a
      justified no-change result for verification-only work.
- [ ] Format/lint this phase's source and tests.
- [ ] Run this phase's applicable typecheck.
- [ ] Run this phase's applicable package unit tests.
- [ ] Build only this phase's package output, using finalized upstream artifacts.
- [ ] Report and enforce this phase's defined bundle budgets; record inapplicable checks explicitly.
- [ ] Apply the minimal/DX rubric, record exact evidence/blockers, report and stop.
- [ ] Record explicit user acceptance/direction before advancing.

API Client owns wire schemas and encoding. Core owns event construction/admission, queues and real
response application. Node's data-to-handoff helper remains pure. Web owns browser identity,
hydration/storage and existing presentation controllers. React binds those facilities and owns
one invocation site. Next.js adapts HTTP/framework boundaries. A caller or re-export does not
become another implementation owner.

### Package validation policy

The checkboxes belong to each phase above. This policy defines how to perform those checks; it is
not a separate completion checklist. Run only applicable checks for the active phase, serially,
using current source/current-phase output and finalized upstream artifacts. Record exact results
in that phase's evidence. Apply the minimal-implementation/DX rubric before reporting completion.

Use exact package filters and scoped file paths. No root aggregate lint/typecheck/unit/build/size/
format run, recursive dependency filters, or validation of other packages. Package builds/cleaning
may touch their documented outputs; do not broadly delete ignored/local files. Do not invent tests,
runners or budgets to fill a checklist. Placeholder scripts are not evidence. A budget overage is
reported under existing policy, never fixed by hiding code, removing behavior or changing budgets.

```sh
pnpm exec prettier <changed-current-package-files> --write
pnpm exec eslint <current-package-path>
pnpm --filter <exact-package-name> typecheck
pnpm --filter <exact-package-name> test:unit
pnpm --filter <exact-package-name> build
pnpm --filter <exact-package-name> size:report
pnpm --filter <exact-package-name> exec build-tools bundle-size
```

The last command is the existing enforcement checker on just-built output, avoiding size:check's
extra rebuild. Run only applicable scripts, serially. Report-only measurement covers every current-
package entry before enforcement. Mocks receive scoped source checks, never mock-owned tests.

S7's exact scope is `packages/universal/optimization-js-bridge`, `packages/react-native-sdk`,
`packages/android/ContentfulOptimization` and `packages/ios/ContentfulOptimization`. Build the bridge
once and use its generated copies for native units; never hand-edit assets. Run React Native's
package-only checks/budgets. No invented bridge/native JS budgets or root native wrappers that
rebuild the bridge. Native unit/compile verification is scoped directly:

```sh
./packages/android/ContentfulOptimization/gradlew -p packages/android/ContentfulOptimization testDebugUnitTest
swift test --package-path packages/ios/ContentfulOptimization
```

No Metro, devices, Detox, Maestro, XCUITest or mobile build-for-testing. Missing toolchains are
reported precisely; they do not justify broadening checks or changing product behavior.

## Reference implementation phases

Only after packages are finalized and the user authorizes progression, package current outputs
with `pnpm pack:pkgs`; do not rebuild the workspace with `pnpm build:pkgs`. Then update one reference
implementation per phase, stopping/reporting for approval after each. Install current tarballs with
its documented implementation wrapper; run only its applicable lint/typecheck/build and relevant
existing app-local units. E2E files remain untouched until these phases finish.

### I0: package finalized outputs

**Status:** `not started` · **Evidence/source:** — · **Next action/blocker:** —

Pack finalized outputs only with `pnpm pack:pkgs`; do not rebuild SDK packages.

- [ ] Confirm all SDK phases are accepted and outputs correspond to their recorded source.
- [ ] Run `pnpm pack:pkgs` serially; record artifacts and source correspondence.
- [ ] Report results/blockers and stop; no implementation edits or E2E work in this phase.
- [ ] Record explicit user acceptance/direction before advancing.

### I1: Node + Web

**Status:** `not started` · **Evidence/source:** — · **Next action/blocker:** —

**Target:** `implementations/node-sdk+web-sdk`. Replace server commitment/hydration-page sequencing with preparation/commitment. Preserve current templates/rendering, consent/cookies and functionality; add no SSR renderer.

- [ ] Inspect this target's policies/callers and confirm its minimal migration or no-change scope.
- [ ] Complete only the migration above; maintain no new SDK unit coverage here and leave E2E
      files untouched.
- [ ] Install finalized tarballs where this target consumes them; run only applicable target
      lint/typecheck/build and genuine existing app-local units. Record each result or inapplicability.
- [ ] Apply the minimal/DX rubric, record changes/results/blockers, report and stop.
- [ ] Record explicit user acceptance/direction before advancing.

### I2: Angular

**Status:** `not started` · **Evidence/source:** — · **Next action/blocker:** —

**Target:** `implementations/web-sdk_angular`. Change the existing request/transfer/browser integration to shared prepared replay. Retain SSR/CSR output, callbacks and existing entry/fetch behavior.

- [ ] Inspect this target's policies/callers and confirm its minimal migration or no-change scope.
- [ ] Complete only the migration above; maintain no new SDK unit coverage here and leave E2E
      files untouched.
- [ ] Install finalized tarballs where this target consumes them; run only applicable target
      lint/typecheck/build and genuine existing app-local units. Record each result or inapplicability.
- [ ] Apply the minimal/DX rubric, record changes/results/blockers, report and stop.
- [ ] Record explicit user acceptance/direction before advancing.

### I3: Next.js App Router

**Status:** `not started` · **Evidence/source:** — · **Next action/blocker:** —

**Target:** `implementations/nextjs-sdk_app-router`. Consume the existing-ID/context handler and finalized request/client delivery boundary. Server configuration supplies paired initial inputs; keep existing routes/cache/private slots.

- [ ] Inspect this target's policies/callers and confirm its minimal migration or no-change scope.
- [ ] Complete only the migration above; maintain no new SDK unit coverage here and leave E2E
      files untouched.
- [ ] Install finalized tarballs where this target consumes them; run only applicable target
      lint/typecheck/build and genuine existing app-local units. Record each result or inapplicability.
- [ ] Apply the minimal/DX rubric, record changes/results/blockers, report and stop.
- [ ] Record explicit user acceptance/direction before advancing.

### I4: Next.js Pages Router

**Status:** `not started` · **Evidence/source:** — · **Next action/blocker:** —

**Target:** `implementations/nextjs-sdk_pages-router`. Migrate server helpers/bootstrap/cookies and existing mounted-root navigation inputs. Keep SSR/ISR/static behavior.

- [ ] Inspect this target's policies/callers and confirm its minimal migration or no-change scope.
- [ ] Complete only the migration above; maintain no new SDK unit coverage here and leave E2E
      files untouched.
- [ ] Install finalized tarballs where this target consumes them; run only applicable target
      lint/typecheck/build and genuine existing app-local units. Record each result or inapplicability.
- [ ] Apply the minimal/DX rubric, record changes/results/blockers, report and stop.
- [ ] Record explicit user acceptance/direction before advancing.

### I5: Next.js Edge

**Status:** `not started` · **Evidence/source:** — · **Next action/blocker:** —

**Target:** `implementations/nextjs-sdk_app-router_edge-runtime`. Migrate existing handoff endpoints/config to shared SDK facilities. Keep actual Edge configuration and useful behavior; add no browser routes/components.

- [ ] Inspect this target's policies/callers and confirm its minimal migration or no-change scope.
- [ ] Complete only the migration above; maintain no new SDK unit coverage here and leave E2E
      files untouched.
- [ ] Install finalized tarballs where this target consumes them; run only applicable target
      lint/typecheck/build and genuine existing app-local units. Record each result or inapplicability.
- [ ] Apply the minimal/DX rubric, record changes/results/blockers, report and stop.
- [ ] Record explicit user acceptance/direction before advancing.

### I6: standalone Node

**Status:** `not started` · **Evidence/source:** — · **Next action/blocker:** —

**Target:** `implementations/node-sdk`. Verify/repair only required callers; preserve standalone server commitment and behavior.

- [ ] Inspect this target's policies/callers and confirm its minimal migration or no-change scope.
- [ ] Complete only the migration above; maintain no new SDK unit coverage here and leave E2E
      files untouched.
- [ ] Install finalized tarballs where this target consumes them; run only applicable target
      lint/typecheck/build and genuine existing app-local units. Record each result or inapplicability.
- [ ] Apply the minimal/DX rubric, record changes/results/blockers, report and stop.
- [ ] Record explicit user acceptance/direction before advancing.

### I7: standalone Web

**Status:** `not started` · **Evidence/source:** — · **Next action/blocker:** —

**Target:** `implementations/web-sdk`. Verify/repair only required callers; preserve standalone browser commitment, callbacks and behavior.

- [ ] Inspect this target's policies/callers and confirm its minimal migration or no-change scope.
- [ ] Complete only the migration above; maintain no new SDK unit coverage here and leave E2E
      files untouched.
- [ ] Install finalized tarballs where this target consumes them; run only applicable target
      lint/typecheck/build and genuine existing app-local units. Record each result or inapplicability.
- [ ] Apply the minimal/DX rubric, record changes/results/blockers, report and stop.
- [ ] Record explicit user acceptance/direction before advancing.

### I8: Web SDK in React

**Status:** `not started` · **Evidence/source:** — · **Next action/blocker:** —

**Target:** `implementations/web-sdk_react`. Verify/repair only required callers; preserve existing browser integration and behavior.

- [ ] Inspect this target's policies/callers and confirm its minimal migration or no-change scope.
- [ ] Complete only the migration above; maintain no new SDK unit coverage here and leave E2E
      files untouched.
- [ ] Install finalized tarballs where this target consumes them; run only applicable target
      lint/typecheck/build and genuine existing app-local units. Record each result or inapplicability.
- [ ] Apply the minimal/DX rubric, record changes/results/blockers, report and stop.
- [ ] Record explicit user acceptance/direction before advancing.

### I9: React Web

**Status:** `not started` · **Evidence/source:** — · **Next action/blocker:** —

**Target:** `implementations/react-web-sdk`. Verify/repair only required callers; preserve existing React integration and behavior.

- [ ] Inspect this target's policies/callers and confirm its minimal migration or no-change scope.
- [ ] Complete only the migration above; maintain no new SDK unit coverage here and leave E2E
      files untouched.
- [ ] Install finalized tarballs where this target consumes them; run only applicable target
      lint/typecheck/build and genuine existing app-local units. Record each result or inapplicability.
- [ ] Apply the minimal/DX rubric, record changes/results/blockers, report and stop.
- [ ] Record explicit user acceptance/direction before advancing.

### I10: React Native reference

**Status:** `not started` · **Evidence/source:** — · **Next action/blocker:** —

**Target:** `implementations/react-native-sdk`. Remove obsolete native app configuration callers only if present. Add no UI/runtime behavior; run no local mobile E2E.

- [ ] Inspect this target's policies/callers and confirm its minimal migration or no-change scope.
- [ ] Complete only the migration above; maintain no new SDK unit coverage here and leave E2E
      files untouched.
- [ ] Install finalized tarballs where this target consumes them; run only applicable target
      lint/typecheck/build and genuine existing app-local units. Record each result or inapplicability.
- [ ] Apply the minimal/DX rubric, record changes/results/blockers, report and stop.
- [ ] Record explicit user acceptance/direction before advancing.

### I11: Android reference

**Status:** `not started` · **Evidence/source:** — · **Next action/blocker:** —

**Target:** `implementations/android-sdk`. Remove obsolete native app configuration callers only if present. Add no UI/runtime behavior; run no local mobile E2E.

- [ ] Inspect this target's policies/callers and confirm its minimal migration or no-change scope.
- [ ] Complete only the migration above; maintain no new SDK unit coverage here and leave E2E
      files untouched.
- [ ] Install finalized tarballs where this target consumes them; run only applicable target
      lint/typecheck/build and genuine existing app-local units. Record each result or inapplicability.
- [ ] Apply the minimal/DX rubric, record changes/results/blockers, report and stop.
- [ ] Record explicit user acceptance/direction before advancing.

### I12: iOS reference

**Status:** `not started` · **Evidence/source:** — · **Next action/blocker:** —

**Target:** `implementations/ios-sdk`. Remove obsolete native app configuration callers only if present. Add no UI/runtime behavior; run no local mobile E2E.

- [ ] Inspect this target's policies/callers and confirm its minimal migration or no-change scope.
- [ ] Complete only the migration above; maintain no new SDK unit coverage here and leave E2E
      files untouched.
- [ ] Install finalized tarballs where this target consumes them; run only applicable target
      lint/typecheck/build and genuine existing app-local units. Record each result or inapplicability.
- [ ] Apply the minimal/DX rubric, record changes/results/blockers, report and stop.
- [ ] Record explicit user acceptance/direction before advancing.

A missing SDK capability returns to its owner through an authorized repair phase. Do not implement
an app-owned queue, identity guard, replay engine or new general rendering utility to compensate.
Cleanup can remove dead/stale code without changing supported behavior.

## Test locations and late E2E

New/expanded SDK unit coverage belongs only in the responsible package under `packages/**`, including
bridge/Kotlin/Swift SDK units. No new unit files/cases/helpers/runners/dependencies under
`implementations/**` or `lib/e2e-web/**`. Existing genuine app-local units may be minimally maintained;
do not repurpose them into SDK suites or move app-only utilities into SDKs to create unit coverage.
No mock-owned behavior tests. Narrow unit fixtures/handlers belong to the SDK package using them.

Unit coverage is proportional to the migration: transport selection/removal, ordered preparation,
identity, relevant consent/interception, retained queue admission/retries, bounded repeat protection,
framework forwarding/composition, and prevention of introduced hydration/rendering/persistence
regressions. One case may establish several invariants. No unit test must demonstrate a hypothetical
new mechanism merely because it was proposed.

Do not test obsolete options or functionality as supported inputs. Remove their obsolete cases
alongside the deleted surface; no deprecation or compatibility behavior is required. Cover only
the supported contracts that remain, including the API Client's per-request single-profile
`preflight` option. This applies to every phase, including native configuration cleanup.

After implementations finish, audit/clean existing E2E and add only missing migration coverage.
Keep valid consumer outcomes while removing cache-key/DTO/header/runtime-witness/SDK-counter/private-
loading assertions. Replace removed tracking evidence with actual delivery meaning. Observe visible
content/controls, cookie continuity, ordinary rendering/navigation and external attribution. Do not
require specific SDK object structure, endpoint/batch layout, node identity or retry message IDs.
Do not build a new app scenario/harness merely to extend coverage beyond maintained integrations.

### T0: shared Web E2E

**Status:** `not started` · **Evidence/source:** — · **Next action/blocker:** —

Update only `lib/e2e-web` shared E2E assertions/necessary cases. No unit suite for E2E helpers.

- [ ] Confirm reference phases are accepted; audit shared assertions for migration relevance.
- [ ] Remove internal assertions and add only missing consumer coverage in this target.
- [ ] Format/lint scoped changed files; record which later target phases exercise those cases.
      Run no unit suite or aggregate E2E here; later execution evidence remains outstanding.
- [ ] Record edits/scenarios/results/blockers, report and stop.
- [ ] Record explicit user acceptance/direction before advancing.

### T1: Node + Web E2E

**Status:** `not started` · **Evidence/source:** — · **Next action/blocker:** —

Update/run only Node/Web-owned consumer E2E. Preserve actual supported behavior; add no SSR capability.

**Runner:** `pnpm test:e2e:node-sdk+web-sdk`; use only relevant files/filters.

- [ ] Identify this target's affected consumer outcomes and necessary owned assertion changes.
- [ ] Complete only necessary E2E edits; format/lint scoped changed files. Add no unit tests.
- [ ] Confirm current services/artifacts and run the target's relevant documented E2E cases;
      record the exact runner/filter, freshness evidence, outcomes and blockers.
- [ ] Record preserved/missing scenarios, report and stop.
- [ ] Record explicit user acceptance/direction before advancing.

### T2: App Router E2E

**Status:** `not started` · **Evidence/source:** — · **Next action/blocker:** —

Update/run affected App Router consumer cases.

**Runner:** `pnpm test:e2e:nextjs-sdk_app-router`; use only relevant files/filters.

- [ ] Identify this target's affected consumer outcomes and necessary owned assertion changes.
- [ ] Complete only necessary E2E edits; format/lint scoped changed files. Add no unit tests.
- [ ] Confirm current services/artifacts and run the target's relevant documented E2E cases;
      record the exact runner/filter, freshness evidence, outcomes and blockers.
- [ ] Record preserved/missing scenarios, report and stop.
- [ ] Record explicit user acceptance/direction before advancing.

### T3: Pages Router E2E

**Status:** `not started` · **Evidence/source:** — · **Next action/blocker:** —

Update/run affected Pages Router consumer cases.

**Runner:** `pnpm test:e2e:nextjs-sdk_pages-router`; use only relevant files/filters.

- [ ] Identify this target's affected consumer outcomes and necessary owned assertion changes.
- [ ] Complete only necessary E2E edits; format/lint scoped changed files. Add no unit tests.
- [ ] Confirm current services/artifacts and run the target's relevant documented E2E cases;
      record the exact runner/filter, freshness evidence, outcomes and blockers.
- [ ] Record preserved/missing scenarios, report and stop.
- [ ] Record explicit user acceptance/direction before advancing.

### T4: Angular E2E

**Status:** `not started` · **Evidence/source:** — · **Next action/blocker:** —

Update/run affected Angular consumer cases.

**Runner:** `pnpm test:e2e:web-sdk_angular`; use only relevant files/filters.

- [ ] Identify this target's affected consumer outcomes and necessary owned assertion changes.
- [ ] Complete only necessary E2E edits; format/lint scoped changed files. Add no unit tests.
- [ ] Confirm current services/artifacts and run the target's relevant documented E2E cases;
      record the exact runner/filter, freshness evidence, outcomes and blockers.
- [ ] Record preserved/missing scenarios, report and stop.
- [ ] Record explicit user acceptance/direction before advancing.

### T5: Edge E2E

**Status:** `not started` · **Evidence/source:** — · **Next action/blocker:** —

Update/run meaningful affected existing Edge cases. Remove internal-only tests without adding an app feature; document any absent consumer scenario precisely.

**Runner:** `pnpm test:e2e:nextjs-sdk_app-router_edge-runtime`; use only relevant files/filters.

- [ ] Identify this target's affected consumer outcomes and necessary owned assertion changes.
- [ ] Complete only necessary E2E edits; format/lint scoped changed files. Add no unit tests.
- [ ] Confirm current services/artifacts and run the target's relevant documented E2E cases;
      record the exact runner/filter, freshness evidence, outcomes and blockers.
- [ ] Record preserved/missing scenarios, report and stop.
- [ ] Record explicit user acceptance/direction before advancing.

### T6: standalone Node E2E

**Status:** `not started` · **Evidence/source:** — · **Next action/blocker:** —

Verify required standalone Node regressions through its existing runner; do not invent a runner or consumer scenario.

**Runner:** `pnpm implementation:run -- node-sdk implementation:test:e2e:run`, with the target's
supported filter mechanism where applicable. Confirm documented prerequisites first; do not use
its root setup-running wrapper automatically.

- [ ] Identify this target's affected consumer outcomes and necessary owned assertion changes.
- [ ] Complete only necessary E2E edits; format/lint scoped changed files. Add no unit tests.
- [ ] Confirm current services/artifacts and run the target's relevant documented E2E cases;
      record the exact runner/filter, freshness evidence, outcomes and blockers.
- [ ] Record preserved/missing scenarios, report and stop.
- [ ] Record explicit user acceptance/direction before advancing.

### T7: standalone Web E2E

**Status:** `not started` · **Evidence/source:** — · **Next action/blocker:** —

Update/run required standalone Web regressions.

**Runner:** `pnpm test:e2e:web-sdk`; use only relevant files/filters.

- [ ] Identify this target's affected consumer outcomes and necessary owned assertion changes.
- [ ] Complete only necessary E2E edits; format/lint scoped changed files. Add no unit tests.
- [ ] Confirm current services/artifacts and run the target's relevant documented E2E cases;
      record the exact runner/filter, freshness evidence, outcomes and blockers.
- [ ] Record preserved/missing scenarios, report and stop.
- [ ] Record explicit user acceptance/direction before advancing.

### T8: Web SDK in React E2E

**Status:** `not started` · **Evidence/source:** — · **Next action/blocker:** —

Update/run required Web SDK in React regressions.

**Runner:** `pnpm test:e2e:web-sdk_react`; use only relevant files/filters.

- [ ] Identify this target's affected consumer outcomes and necessary owned assertion changes.
- [ ] Complete only necessary E2E edits; format/lint scoped changed files. Add no unit tests.
- [ ] Confirm current services/artifacts and run the target's relevant documented E2E cases;
      record the exact runner/filter, freshness evidence, outcomes and blockers.
- [ ] Record preserved/missing scenarios, report and stop.
- [ ] Record explicit user acceptance/direction before advancing.

### T9: React Web E2E

**Status:** `not started` · **Evidence/source:** — · **Next action/blocker:** —

Update/run required React Web regressions.

**Runner:** `pnpm test:e2e:react-web-sdk`; use only relevant files/filters.

- [ ] Identify this target's affected consumer outcomes and necessary owned assertion changes.
- [ ] Complete only necessary E2E edits; format/lint scoped changed files. Add no unit tests.
- [ ] Confirm current services/artifacts and run the target's relevant documented E2E cases;
      record the exact runner/filter, freshness evidence, outcomes and blockers.
- [ ] Record preserved/missing scenarios, report and stop.
- [ ] Record explicit user acceptance/direction before advancing.

### T10: React Native E2E cleanup

**Status:** `not started` · **Evidence/source:** — · **Next action/blocker:** —

Only if necessary, clean React Native-owned internal E2E assertions. Preserve valid consumer scenarios; execution is deferred.

- [ ] Audit owned assertions and record whether cleanup is necessary.
- [ ] Complete only necessary assertion cleanup and applicable scoped formatting/lint, or record
      a justified no-change result. Add no SDK unit tests here.
- [ ] Record preserved scenarios and deferred execution; run no local mobile E2E/build-for-testing.
- [ ] Report results/blockers and stop.
- [ ] Record explicit user acceptance/direction before advancing.

### T11: Android E2E cleanup

**Status:** `not started` · **Evidence/source:** — · **Next action/blocker:** —

Only if necessary, clean Android-owned internal E2E assertions. Preserve valid consumer scenarios; execution is deferred.

- [ ] Audit owned assertions and record whether cleanup is necessary.
- [ ] Complete only necessary assertion cleanup and applicable scoped formatting/lint, or record
      a justified no-change result. Add no SDK unit tests here.
- [ ] Record preserved scenarios and deferred execution; run no local mobile E2E/build-for-testing.
- [ ] Report results/blockers and stop.
- [ ] Record explicit user acceptance/direction before advancing.

### T12: iOS E2E cleanup

**Status:** `not started` · **Evidence/source:** — · **Next action/blocker:** —

Only if necessary, clean iOS-owned internal E2E assertions. Preserve valid consumer scenarios; execution is deferred.

- [ ] Audit owned assertions and record whether cleanup is necessary.
- [ ] Complete only necessary assertion cleanup and applicable scoped formatting/lint, or record
      a justified no-change result. Add no SDK unit tests here.
- [ ] Record preserved scenarios and deferred execution; run no local mobile E2E/build-for-testing.
- [ ] Report results/blockers and stop.
- [ ] Record explicit user acceptance/direction before advancing.

Prioritize existing consumer evidence for: initial browser commitment without a duplicate page;
usable existing personalized SSR during delayed/failed browser delivery; consent/configured cookie
continuity; soft navigation that actually supplies a server handoff; ordinary navigation/standalone
regressions; and offline recovery where the maintained integration supports it. Metadata, exact
transport selection and retry integrity are SDK unit subjects. No exactly-once API assertion after
an ambiguous response. Missing meaningful coverage is reported, not fabricated with internal tests.

Use only the current target's documented runner and relevant file/filter. SDK outputs are already
finalized; do not rebuild/revalidate other packages as a prerequisite. Confirm local mock/app services
load current handlers, installed SDKs and artifacts and target the local Experience/Insights mocks;
a health response alone is not freshness proof. Preserve .env files and user-owned processes, use
runner cleanup and obtain permission for manual termination. No automatic aggregate setup or live
API writes. A package/app defect is repaired in its owning phase after user direction; repeat only
affected authorized evidence, not all suites.

## Minimal implementation and completion

The following rubric is mandatory in S0 and the existing phase reviews. It adds no review stage
or product feature. Public contracts and observable outcomes are mandatory; internal mechanisms
are implementation choices within scope and ownership. A requirement already met by existing or
shared logic needs no separate implementation. Demonstrate the connection with source and
meaningful evidence instead of adding machinery to mirror a checklist item.

- **Consumer DX first:** show the smallest realistic consumer call site before approving a
  signature/config change and compare it with baseline setup. Preserve useful callbacks/options
  and normal standalone behavior. Business inputs, consent and cookie policy are legitimate
  configuration; manual hydration seeds, internal delivery controls or redundant context are not.
- **Reuse before addition:** use information and facilities already available to the owning SDK.
  Do not introduce an option for a fact the SDK can derive. Reuse shared logic across applicable
  cases; adapters handle actual framework differences without another delivery implementation.
- **Invariants before rendering modes:** branch on relevant inputs and ownership, such as replay
  presence, identity, route, consent and admission. Do not create a branch/helper/state machine per
  named rendering mode or requirement when the same decision already covers it.
- **One authoritative fact:** prefer existing admission results, route tracking and lifecycle
  boundaries over mirrored flags/signals/registries. Add retained state only when necessary to
  represent a supported transition; document why that fact cannot safely be derived.
- **Meaningful correctness:** review every proposed safeguard against a concrete supported flow
  and observable end-user/consumer consequence. A harmless internal ordering difference does not
  justify a mechanism. Server request isolation is achieved by request-local state; client queues
  serve one client. Explicit state clearing must remain effective, but does not justify generalized
  visitor, lifetime or latest-response systems. No first-cookie guarantee for disabled JavaScript
  is introduced by this migration.
- **Proof before machinery:** every new helper, field, callback, option, registry, timer or
  abstraction must address a named supported migration case that existing facilities cannot
  meet. Explain the smallest extension. Future flexibility, two call sites alone, or a test of
  an invented mechanism does not justify another facility. Generalization must have real current
  consumers and reduce duplicated logic without obscuring ownership.
- **Logical coverage:** a shared path or meaningful unit case may establish several requirements.
  Trace those outcomes; do not demand a dedicated mechanism/test for each item or every mode
  permutation. Keep required consumer checks and test-location boundaries; logical inference
  does not turn an unrun required check into a pass.
- **Readable simplicity:** prefer fewer independent decisions, state authorities and consumer
  obligations, with clear types and normal repository idioms. Line/module count alone is not the
  criterion. Do not obscure code, weaken functionality or change budgets to reduce measured size.

For example, an atomic retained submission can provide batch integrity, offline recovery,
retry metadata and replacement-page suppression together. The existing presentation boundary
can keep content, context and attribution together without a separate subsystem for each.
Use these as reuse opportunities, not mandates to introduce new abstractions.

If a proposed mechanism or API exceeds scope, report the concrete supported failure and smallest
alternative at the current phase checkpoint before coding it. Do not expand source ownership,
rename new behavior as cleanup, or create extra tests to make scope growth appear necessary.

The Experience API supports preview without committed profile/event effects and returns a profile
ID on every successful response, including creation preview. It alone creates profile identity.
Upsert accepts a known API-issued ID before the profile record is committed; a returned different
ID may be linked. The SDK never creates, reserves or invents profile IDs. The
source reference is `../ninetailed-new` at `fd05a3c595`; this is source evidence, not deployed-backend
certification. Use local contract-aligned mocks and SDK consumer tests; edit no backend or generated
fixtures and perform no live-profile writes.

Finish with an evidence report from accepted phases: server preview/client commitment aligned;
obsolete controls removed; meaningful hydration/cookie/persistence and supported UX retained;
package checks/budgets accounted for; affected consumer evidence passed or concretely blocked.
No final aggregate validation, documentation, CI, PR or release work. No new functionality is
required solely to make the evidence look more complete.
