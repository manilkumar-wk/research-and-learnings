# wdesk_ts Architecture Analysis

Comprehensive analysis of the wdesk_ts monorepo — purpose, architecture, Dart bridge integration, session/auth flow, and backend communication patterns.

---

## 1. Repository Overview

### Why This Repo Exists

wdesk_ts is a **TypeScript rewrite of Workiva's legacy Dart-based wdesk web application**. The goal is to replace the Dart UI with a modern React/TypeScript stack using TanStack Start (TanStack Router + Nitro SSR). Key motivations:

- Modernize the frontend to a mainstream, well-supported ecosystem (React/TypeScript)
- Enable SSR for performance
- Use file-based, type-safe routing
- Move toward a stack with broader community and hiring support than Dart web

### Is It a Replacement for Dart wdesk?

**Yes — it's a phased replacement, not a rewrite-from-scratch.** The architecture is a "replace + bridge" model:

- **Natively rewritten in TS:** App shell, landing dashboard, files page, sidebar, tabs, utility nav, session/auth
- **Bridged from Dart:** 30+ "drawer" experiences (profile, connections, audit forms, etc.) and "rich" experiences (documents, spreadsheets) are loaded as headless Dart custom elements inside the TS shell
- **Rollout is incremental:** Workspaces are enabled one-by-one via nginx ingress routing. Enabled workspaces get the TS app at `/a/<workspaceId>/*`; everyone else still hits the Dart fallback
- **End state:** As more Dart experiences get rewritten in TS (or deprecated), the Dart dependency shrinks until it's fully removed

### Tech Stack

| Component        | Technology                              |
| ---------------- | --------------------------------------- |
| Framework        | TanStack Start (SSR + CSR)              |
| Router           | TanStack Router (file-based, type-safe) |
| Server           | Nitro (Node.js SSR)                     |
| Build            | Vite                                    |
| UI Components    | @workiva/unify (Workiva design system)  |
| Styling          | Emotion (CSS-in-JS)                     |
| State            | TanStack React Query + Zustand          |
| Auth             | OAuth2 via iam-sdk-ts                   |
| Dart Interop     | Custom element bridge                   |
| Monorepo         | pnpm workspaces                         |
| Testing          | Vitest + Playwright                     |

The codebase is **purely TypeScript/React** — no Dart code lives in this repo. Dart is loaded at runtime as a headless library from the existing deployment.

### What's Built

| Metric          | Value                        |
| --------------- | ---------------------------- |
| Total TS/TSX    | ~83,800 lines                |
| Commits         | ~6,900                       |
| PRs merged      | 1,365+                       |
| Route files     | 40+ (landing, files, dashboard, 30+ Dart bridge routes) |
| Packages        | 8 internal + main app        |
| Test files       | ~333                         |

**Production-ready features:**

- App shell (sidebar, tabs, layout, utility nav)
- Landing dashboard with customizable boards/widgets (tasks, files, comments, reviews)
- Files page (via `@workiva/files` package)
- Full OAuth2 session management with Dart-compatible token sharing
- 30+ Dart drawer/rich experience bridge routes
- Workspace switcher, profile menu, help menu with Zendesk chat
- CI/CD pipeline, Lighthouse performance monitoring, Helm deployment charts

### Packages

| Package                    | Path                       | Purpose                                        |
| -------------------------- | -------------------------- | ---------------------------------------------- |
| `@wdesk-ts/app`            | `/app`                     | Main application: routing, views, Nitro server |
| `@wdesk-ts/session`        | `/packages/session`        | OAuth/JWT session management                   |
| `@wdesk-ts/session-react`  | `/packages/session-react`  | React hooks for session utilities              |
| `@wdesk-ts/landing-page`   | `/packages/landing-page`   | Landing dashboard: boards, widgets             |
| `@wdesk-ts/shell`          | `/packages/shell`          | Shell UI: layout, sidebar, tabs                |
| `@wdesk-ts/app-components` | `/packages/app-components` | Utility nav menus                              |
| `@wdesk-ts/host-api`       | `/packages/host-api`       | Navigation abstraction (host/client adapter)   |
| `@wdesk-ts/experience-metrics` | `/packages/experience-metrics` | OpenTelemetry tracing              |
| `@wdesk-ts/vite-plugins`   | `/packages/vite-plugins`   | Custom Vite plugins (Dart proxy, dev server)   |

---

## 2. Dart Bridge Integration

### High-Level Architecture

The TS app **owns the shell** (routing, layout, sidebar, tabs). Legacy Dart features are loaded **on-demand as custom HTML elements** inside the TS shell. There are 7 ADRs governing this in `docs/decisions/dart-integration/`.

### Loading Dart — Singleton Promise Pattern (ADR-003)

When a user first navigates to any Dart route, `loadHeadlessDartWdesk()` fires **once** (`app/src/dart/interop/headless-dart.ts`):

1. Installs a `window.__wk_dart_ready` callback
2. Fetches the FEWS manifest to find the CDN asset URL
3. Loads assets in order: React/Unify (dependencies) → CSS (in a `@layer dart-legacy` to prevent style bleed) → support scripts → `headless.dart.js`
4. Dart initializes and calls `__wk_dart_ready(payload)` with **element factory functions**
5. TS registers custom elements and resolves the promise

Subsequent calls return the cached promise instantly. On failure, the promise is cleared so retries work. A 60-second timeout guards against hangs.

### Custom Elements — The Contract

| Element                         | Used For                                              | Key Attributes                                                               |
| ------------------------------- | ----------------------------------------------------- | ---------------------------------------------------------------------------- |
| `<dart-drawer-experience>`      | Sidebar panels (profile, connections, audit forms)     | `data-segment`, `data-route-tail`                                            |
| `<dart-rich-experience>`        | Full-page content (documents, spreadsheets)            | `data-segment`, `data-resource-id`, `data-route-tail`, `data-toolbar-collapsed`, `data-fullscreen` |
| `<dart-rich-experience-tab>`    | Tab bar metadata (title, close, context menu)          | `data-segment`, `data-resource-id`                                           |
| `<dart-create-menu>`            | Sidebar's "+ Create" button popover                    | (lifecycle tied to popover, not routes)                                      |

### Routing — Pathless Layout Routes

All Dart routes live under a **pathless `_dart` layout** so they don't add a URL segment:

```
/a/$workspaceId/_dart/
├── _drawer/
│   ├── profile.$.tsx        → URL: /a/{ws}/profile/**
│   ├── connections.$.tsx    → URL: /a/{ws}/connections/**
│   └── ... (50+ experiences)
└── _rich/
    └── markup/
        ├── index.tsx        → URL: /a/{ws}/markup/          (create new)
        └── $resourceId.$.tsx → URL: /a/{ws}/markup/{id}/**  (open existing)
```

Each route file calls a utility like `dartDrawerExperienceRoute({ segment: "profile" })` which:

- Sets `ssr: false` (client-only rendering)
- Runs `loadHeadlessDartWdesk()` in the route loader
- Renders the appropriate `<dart-*-experience>` component

### Navigation Bridge — Custom Events

**TS → Dart:** TS sets `data-route-tail` attribute on the custom element. Dart's `attributeChanged` callback picks it up and navigates internally.

**Dart → TS:** Dart dispatches custom DOM events that the `useDartNavigation()` hook listens for:

| Event                                      | Meaning                            | TS Response                    |
| ------------------------------------------ | ---------------------------------- | ------------------------------ |
| `dart-experience-route-changed`            | User clicked a link inside Dart    | `navigate()` to new URL       |
| `dart-experience-title-changed`            | Page title updated                 | Update `document.title` / tab |
| `dart-experience-resource-id-changed`      | New resource created               | Navigate to new resource URL   |
| `dart-experience-toolbar-collapsed-changed`| Toolbar toggled                    | Update shell toolbar state     |
| `dart-experience-fullscreen-changed`       | Fullscreen toggled                 | Update shell fullscreen state  |
| `dart-create-menu-close-requested`         | Create menu item selected          | Close the popover              |

### Lifecycle Management

- **Drawer experiences:** On `disconnected()`, they **deactivate** (stay in memory, max 5 cached) — reactivated instantly on return
- **Rich experiences:** On tab switch, the element disconnects but the experience stays alive. On **tab close**, the shell calls `ref.current.onClose()` which fully unloads it
- **Tab metadata:** `<dart-rich-experience-tab>` is rendered in the tab bar (hidden). It exposes `canClose()`, `getTitle()`, and `getContextMenuItems()` as a public API

### CSS Isolation

Dart's global CSS (web-skin, shell styles) is injected into a low-priority `@layer dart-legacy` cascade layer. This prevents Dart's `html`, `body`, `*` selectors from overriding the TS app's Unify design system styles.

### Complete Flow Example

**User navigates to `/a/workspace123/profile/settings`:**

1. Router matches `_dart/_drawer/profile.$.tsx` (splat = `/settings`)
2. Route loader calls `loadHeadlessDartWdesk()` (cached if already loaded)
3. Component renders `<dart-drawer-experience data-segment="profile" data-route-tail="/settings">`
4. Dart `connected()` → loads/activates the profile experience, renders into the element
5. User clicks a link inside the Dart UI → Dart fires `dart-experience-route-changed` with `{ route: "/preferences" }`
6. `useDartNavigation()` catches it → calls `navigate({ href: "/a/workspace123/profile/preferences" })`
7. Router updates, component re-renders with new `data-route-tail="/preferences"`
8. Dart's `attributeChanged` handles the sub-navigation

---

## 3. Session / Auth Flow

### Overview

The auth system is an **OAuth2 implicit grant** flow using Workiva's IAM service. Tokens are stored in `sessionStorage` with Dart-compatible keys so both the TS app and headless Dart share the same session. The code lives in two packages: `@wdesk-ts/session` (core logic) and `@wdesk-ts/session-react` (React hooks).

### Route Protection

Every route is guarded at the root. `app/src/routes/__root.tsx` uses a `beforeLoad: requireAuth` loader that calls `getSession()`. If it fails and the user isn't in mock-auth mode, they're redirected to:

```
{clusterUrl}/login?next_url={encodedCurrentUrl}
```

### Token Acquisition — `getSession()`

This is the single entry point for all auth (`packages/session/src/get-session.ts`):

1. **Check cache first** — `findCachedAccessToken()` scans `sessionStorage` for keys matching `w_oauth2:*:accessToken:*`. If a valid, non-expired token exists (with a 60-second safety buffer), returns immediately.
2. **Fetch from IAM** — If no cached token, calls `getTokensFromIam()`:
   - **Endpoint:** `GET {iam-service}/api/v1/oauth2/auth`
   - **Params:** `client_id=wdesk-client`, `response_type=id_token+token`, `redirect_uri=postmessage`, `workspace={workspaceId}`
   - **Credentials:** `include` (same-origin cookies carry the IAM session)
   - Returns `access_token` (JWT) + `id_token` (JWT with user profile) + `expires_in`
3. **Save to sessionStorage** — Uses Dart-compatible key format:
   ```
   w_oauth2:wdesk-client:accessToken:ws-123:app.wdesk.com:dataentity|r dataentity|w ...
   ```
4. **Decode JWT** — Custom decoder (no library dependency), returns `SessionData`:
   - `accessToken`, `baseUrl` (from JWT `iss`)
   - `userId`, `organizationId`, `workspaceId`, `workspaceMembershipId` (from JWT `context`)
   - `profile` (name, email, avatar, locale — from `id_token`)
   - `authTime` (from `id_token` `auth_time`)

### Token Refresh — Activity-Based Keep-Alive

`perpetuateSession()` is called in the root route's `useEffect`:

- Listens for `keydown` and `mousemove` events
- Throttled to once per 60 seconds
- On activity: calls `getSession()`, which re-fetches from IAM if the cached token is within 60 seconds of expiry
- No background polling — refresh only happens on user activity

### Dart Compatibility — Shared sessionStorage

Both TS and Dart read/write the same `sessionStorage` keys:

| Aspect          | Detail                                                            |
| --------------- | ----------------------------------------------------------------- |
| Key format      | `w_oauth2:<clientId>:<tokenType>:<accountResourceId>:<hostname>:<sorted scopes>` |
| Scope sorting   | UTF-16 code-unit order (matches Dart's `Comparable`)              |
| Hostname in key | Prevents cross-cluster token reuse                                |
| Storage         | `sessionStorage` (tab-scoped, same-origin)                        |

When headless Dart loads, it finds the TS-written token in `sessionStorage` and uses it directly. Last write wins — there's no coordination lock between the two.

### Workspace Switching

`switchWorkspace(membershipResourceId)`:

1. Calls `activateMembership()` — `PUT {iam-service}/session/activation/memberships/{id}` with CSRF token
2. Clears all cached tokens from `sessionStorage`
3. React hook navigates to `/a/{newWorkspaceId}` with `reloadDocument: true` (full page reload)

### Logout

1. Removes all `w_oauth2:*` keys from `sessionStorage`
2. Redirects to `{clusterUrl}/auth/logout?next_url={currentUrl}`

### Mock Auth (Development)

Appending `?mockAuth=true` to any URL bypasses real authentication, returning a mock `SessionData`.

### Security Properties

| Property           | Implementation                                 |
| ------------------ | ---------------------------------------------- |
| Storage            | `sessionStorage` (tab-scoped, not localStorage)|
| Expiry buffer      | 60 seconds before JWT `exp`                    |
| CSRF               | Token on workspace activation requests         |
| Cluster isolation  | Hostname embedded in storage key               |
| No console logging | Tokens never logged                            |

---

## 4. Backend Communication

### No Frugal / Thrift / gRPC

There are **zero Frugal (Thrift RPC) calls** in this repo. Every backend call is a standard **REST HTTP request using the browser's native `fetch` API**. No Frugal, gRPC, Thrift, or protobuf dependencies exist.

### The Pattern

Every network function follows the same shape:

1. Call `getSession()` to get `accessToken` and `baseUrl` (from the JWT `iss` claim)
2. Build a URL: `{baseUrl}/s/{service-name}/api/...`
3. `fetch()` with `Authorization: Bearer {accessToken}`

There's no shared HTTP client or interceptor — each network function does this independently (except the wdata-query widget which has a small reusable `apiGet`/`apiPost` wrapper).

### Two Execution Contexts

**Client-side (browser):** Network functions call `getSession()` directly and `fetch()` from the browser.

```typescript
const { accessToken, baseUrl, userId } = await getSession();
const response = await fetch(`${baseUrl}/s/user-task-service/api/v3/tasks/...`, {
  headers: { authorization: `Bearer ${accessToken}` },
});
```

**Server-side (Nitro SSR):** Uses TanStack `createServerFn()` with `authMiddleware`. The client half attaches the JWT as a header, the server half extracts it and resolves internal service URLs:

```
Browser → authMiddleware.client (attaches Bearer token)
       → Nitro server → authMiddleware.server (extracts JWT, resolves service URL)
       → fetch to internal service (e.g., http://configui-service.workiva:8080)
```

In production, server-side calls use **in-cluster DNS** instead of the public URL. In dev, they proxy through the staging cluster.

### Services Called

| Service              | URL Pattern                                  | Used For                               |
| -------------------- | -------------------------------------------- | -------------------------------------- |
| identity (IAM)       | `/api/v1/organizations/{id}`, `/workspaces/` | Org metadata, workspace, branding      |
| configui             | `/api/v0/bff/landingPageBoards`              | Landing page boards (server-side fn)   |
| user-task-service    | `/s/user-task-service/api/v3/tasks`          | Task widget                            |
| comments             | `/s/comments/v2/comments`                    | Comments widget                        |
| policy-evaluator     | `/s/policy-evaluator/api/v1/can`, `/canMany` | Permission checks                      |
| messaging-frontend   | `/s/messaging-frontend/`                     | Real-time board events (WebSocket)     |
| wdata                | various `/s/wdata/...` paths                 | WData query widget                     |
| files                | (via `@workiva/files` package)               | Files page                             |
| reviews              | `/s/...`                                     | Reviews widget                         |
| ESG news feed        | `/s/...`                                     | ESG widget                             |
| explorer bookmarks   | `/s/...`                                     | Explorer bookmarks widget              |

### Real-Time: Messaging SDK (WebSocket)

The only non-REST communication is `@workiva/messaging-sdk` for live board updates. It's a WebSocket connection initialized once via `makeMessagingClient()`:

```typescript
const messagingClient = new MessagingClientProvider({
  baseUrl: `${baseUrl}/s/messaging-frontend/`,
  tokenFetcher: () => getSession().then(s => s.accessToken),
});
```

It subscribes to board events, board-updated events, and board-user events to invalidate React Query caches in real-time.

---

## 5. Rollout Strategy (ADR-010)

- `/s/wdesk-ts/*` — Standalone TS entry point (always available)
- `/a/<enabled-workspace-id>/*` — TS app for opted-in workspaces (via custom nginx ingress)
- `/a/*` — Dart fallback (FEWS backend, unchanged, for non-enabled workspaces)

Two build artifacts from one repo version: `wdesk-ts` (standalone) + `wdesk-ts-rollout` (workspace rollout). When a workspace is enabled, the TS app becomes its primary shell; users see native TS routes and lazy-loaded Dart for unmigrated features.
