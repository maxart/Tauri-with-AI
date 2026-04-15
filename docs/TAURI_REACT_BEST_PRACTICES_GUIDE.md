# Tauri + React Best Practices Guide (2025-2026)

A long-form companion to the `CLAUDE.md` templates in this repo. Each rule there has a one-line summary; this document explains **why**, shows **alternatives**, and flags **gotchas**. Read it once when onboarding to the stack; reach for individual sections when you hit a problem.

This guide targets **Tauri v2** (stable since October 2024), **React 18/19**, **TypeScript 5**, and **stable Rust**. Tauri v1 guidance is not compatible with this document — the security model and IPC were rewritten.

---

## Table of Contents

1. [Why Tauri + React](#why-tauri--react)
2. [Architecture in one picture](#architecture-in-one-picture)
3. [Project setup](#project-setup)
4. [TypeScript on the frontend](#typescript-on-the-frontend)
5. [Rust on the backend](#rust-on-the-backend)
6. [The IPC boundary](#the-ipc-boundary)
7. [Security: capabilities, permissions, CSP, isolation](#security-capabilities-permissions-csp-isolation)
8. [State management](#state-management)
9. [Data fetching](#data-fetching)
10. [React components and hooks](#react-components-and-hooks)
11. [Error handling end-to-end](#error-handling-end-to-end)
12. [Performance](#performance)
13. [Styling and platform theming](#styling-and-platform-theming)
14. [Accessibility](#accessibility)
15. [Frontend security](#frontend-security)
16. [Project structure](#project-structure)
17. [Testing](#testing)
18. [Plugins](#plugins)
19. [Build, bundle, and distribution](#build-bundle-and-distribution)
20. [Tooling](#tooling)
21. [Large-project maintenance](#large-project-maintenance)
22. [Common pitfalls](#common-pitfalls)

---

## Why Tauri + React

Tauri is a framework for building desktop (and now mobile) applications that renders its UI in the operating system's native webview and runs business logic in Rust. React is the frontend library that most teams building serious UI already know.

The combination gets you:

- **Small binaries** — a Tauri app ships around 2.5–10 MB on desktop, vs. 80–150 MB for an Electron equivalent, because Tauri does not bundle a browser runtime.
- **Rust for the hard parts** — filesystem, system APIs, crypto, database, heavy compute — without writing native modules.
- **A single typed UI codebase** — React for desktop and, via Tauri 2, iOS and Android.
- **A strict security model by default** — capabilities and permissions are opt-in at the command level, a strong CSP is injected at build time, and the webview can be isolated from privileged APIs.

The tradeoffs:

- **Two languages.** You'll write TypeScript and Rust, and you'll maintain a contract between them.
- **Webview variance.** You render in the user's system webview (WebKit on macOS/iOS, WebView2 on Windows, WebKitGTK on Linux). They are close but not identical — always test each platform.
- **Newer ecosystem.** Plugins and tooling are strong but evolving. Pin versions, read changelogs.

This guide focuses on the two places where things most commonly go wrong in Tauri + React projects: the **IPC boundary** (typing, errors, performance) and the **security surface** (capabilities, CSP, secret storage).

---

## Architecture in one picture

```
+--------------------------------------------------------------+
|                         Tauri App                            |
|                                                              |
|  +----------------------+         +------------------------+ |
|  |   WebView (React)    | <-----> |   Tauri Core (Rust)    | |
|  |  - UI, routing       |  IPC    |  - Commands            | |
|  |  - Local UI state    | JSON /  |  - Event bus           | |
|  |  - Data-fetching lib | Channel |  - Channels<T>         | |
|  |  - Typed invoke()    |         |  - Managed State<T>    | |
|  +----------------------+         |  - Plugins             | |
|                                    +-----------+------------+ |
|                                                |              |
|                                                v              |
|                               +----------------+------------+ |
|                               |  OS APIs, fs, HTTP, DB,    | |
|                               |  keychain, notifications   | |
|                               +-----------------------------+ |
+--------------------------------------------------------------+
```

Three things to memorize:

1. **Commands** are request/response (the JS side calls `invoke`, Rust returns a `Result`).
2. **Events** are broadcast pub/sub (fire-and-forget; either side can emit and listen).
3. **Channels** are typed, ordered, one-way streams from Rust to JS (v2-only; use for progress and large/frequent data).

Everything that crosses the boundary is serialized (JSON today; the v2 custom protocol is much faster than v1's string pipe, but serialization still has cost). Design around that: keep payloads small, batch, stream.

---

## Project setup

### Scaffolding

```bash
npm create tauri-app@latest
# choose: TypeScript + React + Vite
```

This produces the canonical layout:

```
my-app/
  src/                 # React frontend (Vite)
  src-tauri/           # Rust workspace
    src/main.rs        # thin entrypoint
    tauri.conf.json    # bundle + security config
    Cargo.toml
    capabilities/
  package.json
  vite.config.ts
```

### Things to fix before the first commit

- **Enable TS strictness.** The default `tsconfig.json` is OK but turn on `exactOptionalPropertyTypes: true` and `noUncheckedIndexedAccess: true` now — retrofitting later is painful.
- **Set your reverse-DNS identifier.** In `tauri.conf.json`, replace the placeholder `identifier` (e.g., `com.acme.myapp`) before any release build. Changing it after users install breaks notifications, autolaunch, and the OS-managed application data directory. You lose users' data.
- **Pin Rust.** Add `rust-toolchain.toml` with a stable channel so contributors and CI agree.
- **Pin Tauri's plugin versions to 2.x.** Plugin version drift against the core is one of the most common and confusing build errors.
- **Generate updater keys.** `npx @tauri-apps/cli signer generate` once, now. Store the private key as a CI secret. If you skip this and later want to distribute updates, existing installs cannot be upgraded safely.

### First capability

Create `src-tauri/capabilities/default.json`:

```json
{
  "$schema": "../gen/schemas/desktop-schema.json",
  "identifier": "default",
  "description": "default capability for the main window",
  "windows": ["main"],
  "permissions": ["core:default"]
}
```

Add permissions as you add features. Do not start from a wide-open permission set and tighten later — it almost never happens.

---

## TypeScript on the frontend

### Strictness

Start with `strict: true`. Keep going:

```jsonc
// tsconfig.json
{
  "compilerOptions": {
    "strict": true,
    "exactOptionalPropertyTypes": true,
    "noUncheckedIndexedAccess": true,
    "noImplicitOverride": true,
    "noFallthroughCasesInSwitch": true
  }
}
```

`noUncheckedIndexedAccess` is the single most impactful flag beyond `strict`. It forces you to handle missing keys (`arr[0]` becomes `T | undefined`), which is exactly the kind of bug that causes runtime `Cannot read properties of undefined` crashes in release.

### No `any`, no `React.FC`

Two rules to internalize:

- `any` is a request to the compiler: "please stop helping me." Use `unknown` instead when the type is genuinely unknown — it forces you to narrow before use.
- `React.FC` used to be standard; it implicitly typed `children` and had some subtle issues. The idiomatic form is to type props directly:

```tsx
interface ButtonProps { label: string; onClick(): void }
function Button({ label, onClick }: ButtonProps) { ... }
```

### Discriminated unions for async state

One of the highest-leverage TypeScript patterns:

```ts
type Fetch<T> =
  | { status: 'idle' }
  | { status: 'loading' }
  | { status: 'success'; data: T }
  | { status: 'error'; error: string };
```

The compiler forces you to handle every state in a `switch` or conditional chain. Pair with an `assertNever` helper to catch new variants at compile time:

```ts
function assertNever(x: never): never {
  throw new Error(`Unhandled: ${JSON.stringify(x)}`);
}
```

### Shared types across the IPC boundary

This is the part most teams get wrong. You have a Rust struct and a TypeScript interface that must stay in sync. If they drift, the frontend deserializes garbage.

Three approaches, in order of preference:

1. **Generate TS from Rust.** Use [`specta`](https://docs.rs/specta) with [`tauri-specta`](https://github.com/oscartbeaumont/tauri-specta), or [`ts-rs`](https://docs.rs/ts-rs). The Rust struct is the source of truth; the TS file is emitted at build time.
2. **Generate types at the OpenAPI layer** if your commands already front a REST-like service — unusual in Tauri apps, but valid.
3. **Hand-maintain in a single `src/bindings/` folder** and audit on every command-touching PR.

Pick (1) unless you have a specific reason not to. The maintenance cost of hand-maintained duplicates is invisible until it isn't.

**Serde enums.** Rust enums with data serialize by default as externally tagged (`{"Success": {...}}`). The cleanest mapping to TypeScript union literals uses `#[serde(tag = "type")]` (internally tagged) or `#[serde(tag = "type", content = "data")]` (adjacently tagged). Decide once, at project start — changing it later is a breaking IPC change.

---

## Rust on the backend

Tauri's Rust code is mostly "normal" Rust — async with `tokio`, error handling with `thiserror`/`anyhow`, logging with `tracing`. But a few patterns are specific to the IPC context.

### The command layer is a public API

Treat `#[tauri::command]` functions the same way you'd treat HTTP handlers in a web app:

- Input validation at the top.
- Typed errors that serialize to the frontend.
- No `.unwrap()` or `.expect()` — a panic inside a command crashes that thread and leaves the frontend with a vague error string.
- Keep them thin. Delegate to service-layer functions that are easier to test.

### Error handling: `thiserror` + `Serialize`

Tauri commands must return a type that `serde::Serialize`s. Most error types in the Rust ecosystem (`std::io::Error`, `sqlx::Error`, `reqwest::Error`) do not. You'll see compile errors the first time you try to return `Result<_, io::Error>`.

The standard pattern:

```rust
#[derive(Debug, thiserror::Error)]
pub enum AppError {
    #[error("io: {0}")]
    Io(#[from] std::io::Error),
    #[error("database: {0}")]
    Db(#[from] sqlx::Error),
    #[error("parse: {0}")]
    Parse(String),
    #[error("unauthorized")]
    Unauthorized,
    #[error("not found: {0}")]
    NotFound(String),
}

impl serde::Serialize for AppError {
    fn serialize<S: serde::Serializer>(&self, s: S) -> Result<S::Ok, S::Error> {
        s.serialize_str(&self.to_string())
    }
}
```

For richer errors (code + message), serialize to a struct:

```rust
impl serde::Serialize for AppError {
    fn serialize<S: serde::Serializer>(&self, s: S) -> Result<S::Ok, S::Error> {
        use serde::ser::SerializeMap;
        let mut m = s.serialize_map(Some(2))?;
        m.serialize_entry("code", self.code())?;
        m.serialize_entry("message", &self.to_string())?;
        m.end()
    }
}
```

A short `code()` method on `AppError` (returning `"io" | "db" | "unauthorized"` etc.) lets the frontend branch on machine-readable identifiers instead of parsing English error strings.

**Do not use `anyhow::Error` as a command return type.** It doesn't implement `Serialize`. Use `anyhow` at the top level (e.g., in `main.rs`, in tests, in setup code) and convert to `AppError` at the command boundary.

### Async and the runtime

Tauri uses `tokio`. Async commands run on Tauri's runtime. The one rule to remember: **don't block the runtime**.

- Use `tokio::fs`, not `std::fs`, in async commands.
- Use `reqwest` (async), not `ureq` (sync), for HTTP.
- Use `tokio::time::sleep`, not `std::thread::sleep`.
- For blocking-only crates, wrap in `tokio::task::spawn_blocking(...).await`.

The gotcha most teams hit: **`std::sync::Mutex` held across `.await`**. It compiles, it runs, and then it deadlocks the runtime under load. Use `tokio::sync::Mutex` for state you keep locked across `.await` points.

```rust
// BAD
let guard = state.std_mutex.lock().unwrap();
network_call().await;   // runtime blocked while guard lives

// GOOD
let guard = state.tokio_mutex.lock().await;
network_call().await;   // async-aware — runtime keeps moving
```

### Ownership at the boundary

Borrows don't cross the IPC boundary — everything serializes and owns. Prefer owned types (`String`, `Vec<T>`) in command signatures. Don't try to return `&'a T`.

Derive `Debug`, `Clone`, `Serialize`, `Deserialize` deliberately. Each derive has compile-time and binary-size cost. Not every type needs every derive.

### Validating input

Everything the frontend sends is untrusted. A compromised webview (XSS, malicious extension, arbitrary user action in the devtools) can call any exposed command with any payload.

Validate:

- **Lengths.** Reject empty strings, or strings over N KB.
- **Ranges.** If a command takes a page size, cap it at something sane.
- **Paths.** Canonicalize and check that the path is under an allowed root directory. Prevents `../../etc/passwd`-style traversal.
- **Shell input.** Do not interpolate frontend strings into `std::process::Command` invocations. Use arguments vectors, and for anything user-controlled, prefer a narrowly-scoped `tauri-plugin-shell` sidecar with an allow-list in the capability.

---

## The IPC boundary

The three IPC primitives — commands, events, channels — solve different problems.

### Commands: request/response

Most of your IPC will be commands. Register them in `lib.rs`:

```rust
pub fn run() {
    tauri::Builder::default()
        .manage(AppState::new())
        .invoke_handler(tauri::generate_handler![
            commands::user::get_user,
            commands::user::save_user,
            commands::window::close_main,
        ])
        .run(tauri::generate_context!())
        .expect("error while running tauri application");
}
```

On the frontend, **wrap every invocation in a typed function** in `src/ipc/`. Do not call `invoke('some_cmd', ...)` directly in components.

```ts
// src/ipc/user.ts
import { invoke } from '@tauri-apps/api/core';
import type { User } from '@/bindings/User';

export function getUser(id: number): Promise<User> {
  return invoke<User>('get_user', { id });
}

export function saveUser(user: User): Promise<void> {
  return invoke('save_user', { user });
}
```

Benefits:

- One place to change a command signature.
- One place to centralize error handling, logging, telemetry.
- Components stay focused on rendering.

### Argument naming

Tauri converts snake_case Rust parameters to camelCase on the JS side by default (in v2 this can be reconfigured with `rename_all` on the command). Keep it consistent across the project — mixing conventions is a recurring source of "my command is called but arguments are undefined" bugs.

### Events: broadcast pub/sub

Events are untyped JSON broadcasts. Useful for:

- "The app config changed, everyone refetch."
- "A window lost focus."
- "Background job progress" (but see Channels below for a better option).

**Always** clean up listeners in `useEffect` cleanup:

```ts
useEffect(() => {
  const unlistenP = listen<ConfigChangedPayload>('config://changed', (e) => {
    queryClient.invalidateQueries({ queryKey: ['config'] });
  });
  return () => { unlistenP.then((fn) => fn()); };
}, []);
```

Scope events to a window when possible (`window.emit` / `window.listen`) rather than the global bus. If you find yourself inventing a type discriminant on event payloads, you probably want a command or channel instead.

### Channels: streaming, typed, ordered

Tauri v2's `Channel<T>` API is the right tool for high-frequency, ordered, typed data from Rust to JS: progress updates during a long download, log lines from a spawned process, updates during a long computation.

```rust
use tauri::ipc::Channel;

#[derive(serde::Serialize, specta::Type)]
#[serde(rename_all = "camelCase")]
pub struct Progress {
    pub bytes_done: u64,
    pub bytes_total: u64,
}

#[tauri::command]
async fn download(url: String, on_progress: Channel<Progress>) -> Result<(), AppError> {
    // ... in the loop:
    on_progress.send(Progress { bytes_done, bytes_total })
        .map_err(|e| AppError::Io(std::io::Error::other(e)))?;
    Ok(())
}
```

On the frontend:

```ts
import { Channel, invoke } from '@tauri-apps/api/core';

const progress = new Channel<Progress>();
progress.onmessage = (p) => setProgress(p);
await invoke('download', { url, onProgress: progress });
```

Channels are preferable to events for anything with a natural response type or a clear sender/receiver pair.

### Payload size

Everything serializes. 100 KB feels fine; 10 MB feels slow; 100 MB freezes the UI. Rules of thumb:

- For files: return a path, read via `tauri-plugin-fs` (scoped) or stream via a channel.
- For lists: paginate. Send the first page immediately, stream the rest over a channel.
- For images: base64-encoding to embed in JSON is the slowest way. Prefer streaming the bytes, or letting the webview load a local file via a custom protocol (Tauri's `asset:` protocol).

---

## Security: capabilities, permissions, CSP, isolation

Tauri v2's security model is the most important thing that changed from v1, and the most common place for AI-generated code to take shortcuts. It is worth understanding properly.

### The mental model

- **Commands** are functions exposed from Rust to JS. They are **not callable by default** from the frontend — even if registered in `generate_handler!`. A window can only call a command if the window's **capability** includes a **permission** that allows it.
- **Permissions** are named bundles of capability (`fs:allow-read-text-file`, `shell:allow-execute`). Each plugin ships its own set.
- **Capabilities** are JSON/TOML files in `src-tauri/capabilities/` that say "this permission is granted to these windows on these platforms". They're the ACL rows.
- **Scopes** attach to permissions to further restrict them: "this permission, but only for paths under `$APPCONFIG`".

This model replaces v1's allowlist (which was global, coarse-grained, and all-or-nothing). It's stricter by design.

### The principle

Grant the **narrowest** permission to the **fewest** windows, and only for the **smallest** scope. "Enable `fs:default` globally" is almost always wrong; it grants broad filesystem access to every window including any future ones.

Example of a well-scoped capability:

```json
{
  "identifier": "main-capability",
  "description": "main window: read config, open docs URLs",
  "windows": ["main"],
  "platforms": ["macOS", "windows", "linux"],
  "permissions": [
    "core:default",
    {
      "identifier": "fs:allow-read-text-file",
      "allow": [{ "path": "$APPCONFIG/*.json" }]
    },
    {
      "identifier": "opener:allow-open-url",
      "allow": [{ "url": "https://docs.example.com/**" }]
    }
  ]
}
```

Notes:

- `windows` is an array; match by glob if needed (`["settings-*"]`).
- `platforms` lets you grant a permission on some OSes but not others. Useful because what's safe on macOS may be risky on Linux.
- Scope objects (`allow`, `deny`) are specific to each permission — check the plugin docs.

Put one capability file per logical feature or window. Name them by identifier. Reference them by identifier in `tauri.conf.json`. Don't pile everything into `default.json`.

### Content Security Policy

Tauri injects a CSP at build time by parsing your frontend and adding nonces/hashes for your own scripts and styles. This is a real XSS defense. Keep it strict:

```json
// tauri.conf.json
{
  "app": {
    "security": {
      "csp": "default-src 'self'; img-src 'self' asset: data:; connect-src ipc: http://ipc.localhost https://api.example.com; script-src 'self'; style-src 'self'"
    }
  }
}
```

Three rules:

1. **Never** `unsafe-inline` or `unsafe-eval`. If a dependency requires either, find a different dependency or pin/patch it. This comes up with older charting libraries and a few Tailwind-like tools — almost always solvable.
2. Allow `connect-src ipc: http://ipc.localhost` — the IPC custom protocol needs it.
3. Keep external `connect-src` origins to the exact list you actually call. Generous CSPs accumulate as teams add features; audit quarterly.

### The isolation pattern

For apps that embed a lot of third-party frontend code (rich text editors, charting libraries, analytics), consider the **isolation pattern**. An extra sandboxed iframe sits between your main frontend and Tauri Core, encrypting IPC messages so only your application (not injected code inside, say, a malicious rich-text paste) can reach privileged APIs.

```json
// tauri.conf.json
{
  "app": {
    "security": {
      "pattern": {
        "use": "isolation",
        "options": { "dir": "../dist-isolation" }
      }
    }
  }
}
```

It adds friction. Don't enable it for a simple app. Do enable it for apps where the frontend renders untrusted content.

### Secrets and tokens

Never store auth tokens in `localStorage` — any XSS reads them. For Tauri apps, the right place is the OS keychain:

- Keychain on macOS
- Credential Manager on Windows
- Secret Service / libsecret on Linux

Options:

- **`tauri-plugin-stronghold`** — IronFish-style encrypted store with a passphrase. Good for very sensitive keys.
- **The `keyring` crate** in a Rust command — thin wrapper over the OS keychain. Good for standard tokens.
- **A custom plugin** wrapping OS-specific APIs — only if you need something exotic.

In all cases, the secret lives in Rust and never flows to JS. The frontend asks the backend to perform operations that need the secret; the secret doesn't leave.

### Dependency audit

Run both:

```bash
npm audit
cargo audit   # from cargo-audit; or `cargo deny check advisories`
```

Wire them into CI. Treat critical and high as blockers. Unlike web apps, a Tauri vulnerability can escalate to the user's machine — broken filesystem permissions, shell execution, native API misuse. Stakes are higher.

---

## State management

Two state-management worlds live in a Tauri app. Don't confuse them.

### Rust-side state (`tauri::State`)

Tauri exposes a way to share values across commands:

```rust
pub struct AppState {
    pub db: tokio::sync::Mutex<sqlx::SqlitePool>,
    pub http: reqwest::Client,
}

tauri::Builder::default()
    .manage(AppState { db: ..., http: reqwest::Client::new() })
    .invoke_handler(...)
```

In commands:

```rust
#[tauri::command]
async fn load_users(state: tauri::State<'_, AppState>) -> Result<Vec<User>, AppError> {
    let db = state.db.lock().await;
    Ok(sqlx::query_as!(User, "SELECT * FROM users").fetch_all(&*db).await?)
}
```

**You do not need `Arc`.** Tauri wraps managed state in an `Arc` internally. You still need interior mutability if you mutate.

**Lock type matters.**

- `std::sync::Mutex` — fine for short synchronous critical sections (update a counter, swap a value).
- `tokio::sync::Mutex` — required whenever you hold the guard across `.await`. Using `std::sync::Mutex` there is the most common runtime deadlock in Tauri code.
- `RwLock` — if readers vastly outnumber writers. Rarely worth it unless you've measured.

Keep critical sections tiny. Copy or move data out, drop the guard, then do the work.

### Frontend-side state

The React state decision tree, applied to Tauri:

1. **Data from commands** → SWR or TanStack Query. Don't copy command results into `useState`. You'll lose caching, dedup, and background revalidation.
2. **Local UI state** (one component: a toggle, a form field) → `useState`.
3. **Multi-field or state-machine-shaped local state** → `useReducer`, or a discriminated union + `useState`.
4. **Shared by 2+ components** → lift state up first. If prop drilling exceeds two levels, use React Context.
5. **App-wide complex state** → Zustand is the default recommendation for Tauri apps. It's tiny (~1 KB), provider-less, and plays well with IPC.
6. **Persistent state** — this is where Tauri differs from web apps. Use `tauri-plugin-store` for non-sensitive user preferences (theme, sidebar width). Use the OS keychain for sensitive data. Don't use `localStorage` for anything you care about.

### Derived state — never store it

The single most common anti-pattern in React code is storing derived values in state and syncing them via `useEffect`. It causes extra renders, race conditions, and bugs that are hard to diagnose.

Derived values belong in `const` during render:

```tsx
const filtered = users.filter((u) => u.active); // fine, cheap
const slow = useMemo(() => heavyTransform(users), [users]); // memoize only if expensive
```

If you ever feel the need for `useEffect(() => setX(computeFromY(y)), [y])`, step back. You want `const x = computeFromY(y);`.

---

## Data fetching

The frontend of a Tauri app fetches data in two ways:

1. **Tauri commands** (most common) — talk to Rust, which does the work.
2. **HTTP from the frontend** — for public APIs where you don't need to touch filesystem/keychain/etc.

### Always use a data-fetching library

Raw `invoke` in `useEffect` + `useState` has all the same problems as raw `fetch` in `useEffect` + `useState` — no caching, no deduplication, no background revalidation, race conditions. Wrap every command in SWR or TanStack Query.

```ts
// src/hooks/data/useUsers.ts
import useSWR from 'swr';
import { getUsers } from '@/ipc/user';

export function useUsers() {
  return useSWR('users', () => getUsers(), { revalidateOnFocus: true });
}
```

Both SWR and TanStack Query are good choices. TanStack Query has richer mutation/optimistic-update helpers; SWR is smaller and has a simpler mental model. Pick one and stick to it.

### HTTP from Rust vs. from JS

If you need to call an external API:

- **In Rust**, via `reqwest` in a command, when you need:
  - CORS bypass (the browser enforces CORS; your Rust code doesn't).
  - Custom TLS, certs, or proxies.
  - Tokens that must not touch the webview.
  - Streaming responses to files.
- **In JS**, via native `fetch`, when:
  - The API is public and CORS-friendly.
  - The response is small and fits in memory.
  - You'd like SWR/Query to handle caching.

A common mistake: frontend `fetch` to an API that requires an OAuth bearer token kept in the OS keychain. The token has to go through Rust anyway, so make it a command and keep the token out of the webview.

### Conditional fetching

Don't fire a query for a parameterized endpoint before the parameter exists:

```ts
// SWR: null key pauses
useSWR(userId ? ['user', userId] : null, () => getUser(userId!));

// TanStack Query: enabled
useQuery({ queryKey: ['user', userId], queryFn: () => getUser(userId!), enabled: !!userId });
```

---

## React components and hooks

Most of the React best-practice surface is the same as for web React. The highlights, filtered for what matters in a Tauri app.

### Components

- One component per file. Filename matches component name (`UserProfile.tsx`).
- Keep files under 200 lines.
- Prefer composition over configuration: pass `children`, or accept slot props (`renderHeader`), instead of piling up boolean props.
- Extract custom hooks when a component does too much. A Tauri command wrapper is almost always a hook: `useUsers`, `useDownload`, `useUpdateStatus`.
- Destructure props in the signature. Default values via destructuring.
- Never pass more than ~7 props. Group or split.

### Lists

- Stable unique keys (`item.id`), not array indices. Array indices break reorder/filter/insert in subtle ways — state attaches to the wrong row after a change.

### Hooks

- `useEffect` is for **synchronizing with external systems**: DOM listeners, Tauri events, timers, third-party libraries. It is not for deriving state or handling events.
- Every effect that creates a subscription or listener **must** return a cleanup function.
- React 18+ StrictMode runs effects twice in development precisely to surface missing cleanup. Your effects should be idempotent.
- `useMemo` / `useCallback` only where measured or where a memoized child depends on a stable reference.
- If on React 19 with the React Compiler enabled, manual memoization is usually unnecessary. Keep the compiler enabled and stop writing `useCallback` by reflex.

### Tauri event listeners

```ts
useEffect(() => {
  let cancelled = false;
  const p = listen<DeepLinkPayload>('deep-link://open', (e) => {
    if (cancelled) return;
    handle(e.payload);
  });
  return () => {
    cancelled = true;
    p.then((fn) => fn()).catch(() => {});
  };
}, []);
```

Two things:

1. Return the `unlisten` function. Missing it leaks listeners across remounts.
2. Gate on a `cancelled` flag if your handler uses state or props that may have changed between attach and callback.

---

## Error handling end-to-end

An error in a Tauri app can originate in three places. Each has its own handling rules.

### Rust — inside a command

Return `Result<T, AppError>`. Log the details server-side (`tracing::error!(err = ?e, "load_users failed")`); return a sanitized variant:

```rust
#[tauri::command]
async fn load_users(state: tauri::State<'_, AppState>) -> Result<Vec<User>, AppError> {
    let db = state.db.lock().await;
    sqlx::query_as!(User, "SELECT * FROM users")
        .fetch_all(&*db)
        .await
        .map_err(|e| {
            tracing::error!(?e, "load_users query failed");
            AppError::Db(e)
        })
}
```

### JS — at the call site

A failed invoke rejects the promise with the serialized error. Wrap in try/catch:

```ts
try {
  const users = await getUsers();
  // ...
} catch (e) {
  // e is whatever your AppError serializes to
  toast.error(formatError(e));
  logger.error('getUsers failed', { error: e });
}
```

For consistency, wrap all IPC errors in an `IpcError` class at the wrapper layer:

```ts
export class IpcError extends Error {
  constructor(public code: string, message: string) { super(message); }
}

export async function getUsers(): Promise<User[]> {
  try {
    return await invoke<User[]>('get_users');
  } catch (raw) {
    throw parseIpcError(raw);
  }
}
```

### React — rendering errors

Use `react-error-boundary`. Place boundaries at three levels:

- **App-level** — catastrophic failures (show a "please restart" screen + Sentry report).
- **Feature-level** — a broken sidebar doesn't kill the main content.
- **List-item-level** — one bad item doesn't collapse the whole list.

```tsx
<ErrorBoundary FallbackComponent={AppFallback} onError={reportToSentry}>
  <Layout>
    <ErrorBoundary FallbackComponent={FeatureFallback}>
      <Sidebar />
    </ErrorBoundary>
    <ErrorBoundary FallbackComponent={FeatureFallback}>
      <MainContent />
    </ErrorBoundary>
  </Layout>
</ErrorBoundary>
```

**Boundaries don't catch errors inside event handlers or async functions.** Use try/catch for those. A common bug: an async submit handler rejects, and the user sees no feedback. Always surface user-facing errors through a toast or inline message.

### Panics in Rust

Panics inside a command crash the Tokio task for that command and surface as a serialization error on the frontend — opaque. Panics outside command handlers (in startup, in a spawned task you don't await) can crash the whole backend process, which closes the app.

- Use `std::panic::set_hook` early in `main.rs` to log panics to file + Sentry.
- Catch them at task boundaries if you can recover (`tokio::task::spawn` returns a `JoinHandle` whose `await` returns `Err(JoinError)` on panic).
- Don't rely on panic recovery as control flow. Use `Result`.

---

## Performance

### Frontend

**Bundle size matters more than web apps realize.** Desktop users download your app once, but every feature they never touch still bloats the installer and slows cold start. Lazy-load everything that isn't on the first screen.

```ts
const Settings = lazy(() => import('./pages/Settings'));
```

Route-level code splitting is mandatory. Feature-level splitting is useful for rarely-used features (a help viewer, an about dialog).

Don't memoize reflexively. `React.memo` on every component is a waste; it adds equality checks that can be more expensive than re-rendering. Measure before memoizing. Use the React DevTools Profiler; the answer is almost always a specific list-item component.

Debounce user input (search inputs, resize handlers) to 200–500 ms. `useTransition` for expensive updates that aren't urgent (filtering a 50k-row table).

### IPC

**Payload size dominates IPC performance.** A 5 MB JSON blob takes tens of milliseconds to serialize in Rust, transmit, parse in JS — and blocks the JS main thread while it parses. Rules:

- **Stream.** Use `Channel<T>` for anything incremental. Send small messages often, not one giant message at the end.
- **Paginate.** 50–200 items per page is a good default. Let the frontend request more.
- **Don't base64 images.** Use the `asset:` custom protocol or a file path.
- **Don't serialize what the frontend doesn't need.** Project your Rust types down to a minimal DTO before returning.

### Rust

- Release profile tuning. In `Cargo.toml`:

  ```toml
  [profile.release]
  strip = true
  lto = "fat"
  codegen-units = 1
  panic = "abort"
  opt-level = 3
  ```

  `lto = "fat"` + `codegen-units = 1` slows compile time but produces smaller, faster binaries. Use `panic = "abort"` when you're sure you don't rely on unwinding (most Tauri apps don't).

- Move blocking I/O off the async runtime with `spawn_blocking`. A single blocking call can starve the runtime and freeze every other command.
- Avoid unnecessary allocations in hot paths. `&str` beats `String`, `Cow<str>` when you sometimes need ownership, `Vec::with_capacity` when size is known.
- `cargo bloat --release -n 20` shows the 20 largest functions in your binary. Trim dependencies that contribute surprisingly much.

---

## Styling and platform theming

### Choose one styling approach and stick to it

Mixing Tailwind, CSS Modules, and styled-components in the same project is a smell. Pick one:

- **Tailwind CSS** — fast iteration, great with component libraries like shadcn/ui.
- **CSS Modules** — zero runtime, native CSS ergonomics, scoped by default.
- **Vanilla Extract** — typed CSS-in-TS with zero runtime.

Runtime CSS-in-JS (styled-components, Emotion) has been falling out of favor. It adds bundle size and render cost with no clear win in modern projects.

### The `cn()` utility for Tailwind

```ts
import { twMerge } from 'tailwind-merge';
import clsx, { type ClassValue } from 'clsx';

export function cn(...inputs: ClassValue[]) {
  return twMerge(clsx(inputs));
}
```

`clsx` handles conditional composition, `tailwind-merge` de-conflicts classes (`bg-red-500 bg-blue-500` → `bg-blue-500`). Use everywhere.

### Platform feel

A desktop app that looks like a website looks cheap. Small touches matter:

- Respect OS theme: `window.matchMedia('(prefers-color-scheme: dark)')` or `tauri-plugin-os`.
- Respect OS accent color on Windows via the `os` plugin.
- Traffic-light spacing on macOS (give the top-left enough room; don't put a clickable element under the close button).
- Native-feeling scrollbars, especially on macOS.
- Use system font stacks by default (`system-ui, -apple-system, Segoe UI, ...`) unless you have a brand reason not to.

Match your target platform's conventions. If you ship one app for all three platforms, test on all three.

---

## Accessibility

The rules don't change in a desktop app — semantic HTML, keyboard accessibility, ARIA labels, screen reader support. But desktop users' expectations are higher: they know the keyboard shortcuts, they use VoiceOver/NVDA, they notice when an app feels "off."

Non-negotiables:

- Semantic HTML: `<button>` for actions, `<a>` for navigation, `<nav>`, `<main>`, `<header>`, `<footer>`.
- **Never** `<div onClick>`. It's not focusable, not keyboard-accessible, not announced.
- Every `<img>` has `alt`. Decorative: `alt=""`.
- Every icon-only button has `aria-label`.
- Every form input has an associated `<label>` (via `htmlFor`) or `aria-label`.
- Every interactive element is keyboard-reachable and has a visible focus style.

Don't hand-roll complex components (modals, comboboxes, dropdowns, tabs). Accessibility is the bulk of the work and easy to get wrong. Use:

- **Radix UI** — unstyled primitives, superb docs, wide coverage.
- **Headless UI** — Tailwind Labs' set, smaller API.
- **React Aria / React Aria Components** — Adobe's library, most comprehensive a11y coverage, Spectrum's underpinnings.
- **shadcn/ui** — Radix + Tailwind, copy-paste components you own and can modify.

Install and enforce `eslint-plugin-jsx-a11y` with the recommended config.

Testing checklist before release:

- Unplug the mouse and complete your critical flows with the keyboard only.
- Run VoiceOver (macOS: Cmd+F5) and NVDA (Windows). Every interactive element should announce something meaningful.
- Run Lighthouse's accessibility audit. The score is a signal, not a guarantee.

---

## Frontend security

The CSP and capabilities already do a lot. Don't undo their work in JS.

- **Never** `dangerouslySetInnerHTML` with user-provided content. If you must render HTML (markdown, WYSIWYG output), sanitize through DOMPurify. Better: use `react-markdown` for markdown so you never touch raw HTML.
- Enforce `react/no-danger: 'error'` in ESLint. If you need an exception, get an explicit review.
- **Never** put secrets in the bundle. Anything in `VITE_*` env vars ends up in the installer, plain-text, inspectable with a text editor. Secrets live in Rust or in the user's keychain.
- **Never** interpolate user input into `href` — `javascript:` URLs are an XSS vector. Sanitize or constrain to http/https/mailto.
- Validate user input on both sides. The frontend constraint is UX (prevent a user from submitting a too-long string). The Rust constraint is security (reject a malicious payload from a compromised webview).

---

## Project structure

The layout in the CLAUDE.md files works for most Tauri + React apps. Two things to emphasize:

### Colocation

Feature-specific code lives inside that feature's directory. A page-level component, its feature hooks, its feature styles, its feature tests — all in one place:

```
src/pages/UserSettings/
  components/
    NotificationSettings.tsx
    ThemeSwitcher.tsx
  hooks/
    useNotificationSettings.ts
  UserSettings.tsx
  UserSettings.test.tsx
```

Promote to top-level (`src/hooks/`, `src/components/`) only when a second feature imports it. Premature promotion creates a sprawling "shared" directory that nobody owns.

### Import hygiene

Enforce `import/no-cycle: 'error'`. Cycles are silent bugs waiting to happen; they cause weird undefined-at-runtime errors and break tree-shaking.

Use `simple-import-sort` for deterministic ordering. Use path aliases (`@/components/...`) for clean imports.

Keep barrel files (`index.ts`) small. A barrel with 40 exports is a re-render and tree-shaking hazard. For large projects, prefer direct imports.

In Rust, scope visibility carefully. `pub(crate)` is almost always what you want — `pub` leaks types into a public API you didn't intend.

---

## Testing

Three layers, in decreasing frequency of use:

### Frontend unit/integration (Vitest or Jest)

Test components with React Testing Library. Query by role, label, text — in that order. Use `userEvent` for interactions.

**Mock Tauri APIs.** Do not launch a webview for unit tests — it's slow and platform-coupled. Use `@tauri-apps/api/mocks`:

```ts
import { beforeEach } from 'vitest';
import { mockIPC } from '@tauri-apps/api/mocks';

beforeEach(() => {
  mockIPC((cmd, args) => {
    switch (cmd) {
      case 'get_user': return { id: (args as any).id, name: 'Mock User' };
      default: throw new Error(`unexpected IPC ${cmd}`);
    }
  });
});
```

This lets you test a component's behavior when IPC succeeds, fails, is slow, returns unexpected data — deterministic, fast, no webview.

### Rust unit and integration

Unit tests live in the same file as the code, gated by `#[cfg(test)]`. They run under `cargo test`.

For commands, use Tauri's mock runtime — it gives you a `tauri::App` without launching a native webview:

```rust
#[cfg(test)]
mod tests {
    use tauri::test::{mock_builder, mock_context, noop_assets};

    #[test]
    fn get_user_returns_expected() {
        let app = mock_builder()
            .invoke_handler(tauri::generate_handler![crate::commands::user::get_user])
            .build(mock_context(noop_assets()))
            .expect("failed to build mock app");
        // Drive commands via app.handle().invoke(...) or test them directly.
    }
}
```

Use `mockall` for mocking traits. Don't try to mock concrete structs; that way lies madness. Design with traits where substitutability matters.

### End-to-end

WebDriver is the standard, via `tauri-driver`. Supports Linux + Windows. **macOS is unsupported** as of 2025 because there is no WKWebView driver. For macOS-critical flows you have two options:

- Run E2E on Linux + Windows, and do manual smoke tests on macOS before release.
- Use Playwright against your dev server (frontend-only — skips the Rust layer).

Pick your critical path — the one flow that must never break (login, main task, save) — and cover it with E2E. A bloated E2E suite is brittle; a focused one catches real regressions.

### What to skip

- Snapshot tests, except for genuinely stable UI (icons, a static marketing page).
- Tests that assert CSS class names or internal state — they test implementation, not behavior.
- 100% coverage as a goal. Coverage is a diagnostic, not a target.

---

## Plugins

Tauri's plugin ecosystem is in `tauri-apps/plugins-workspace`. The headline plugins:

| Plugin | Purpose |
|---|---|
| `fs` | Filesystem access (scope via capabilities) |
| `dialog` | Native open/save/confirm dialogs |
| `shell` | Open URLs, run sidecar binaries |
| `opener` | Open files/URLs in default applications (often replaces `shell.open`) |
| `os` | OS/platform info |
| `store` | Persistent key-value store (non-sensitive) |
| `sql` | SQLite/Postgres/MySQL via `sqlx` |
| `updater` | In-app updates (see below) |
| `notification` | System notifications |
| `window-state` | Persist window size/position |
| `single-instance` | Prevent multiple running instances |
| `stronghold` | Encrypted secret store (passphrase-protected) |
| `log` | File and console logging |
| `deep-link` | Handle `myapp://` URLs |
| `global-shortcut` | Register global keyboard shortcuts |
| `clipboard-manager` | Clipboard read/write |
| `autostart` | Launch at login |
| `cli` | CLI argument parsing |

Rules:

- **Prefer official plugins** over hand-rolled code. They handle cross-platform edge cases (Linux secret service fallbacks, Windows path quirks) that you'll spend days on.
- **Pin plugin versions to the same Tauri major** (`2.x`). Mismatched majors don't build, and the error messages can be cryptic.
- **Always grant plugin permissions explicitly in capabilities.** If a plugin exposes `fs:allow-read-all` and `fs:allow-read-text-file`, and you only need to read text files under a specific path, grant just the narrower permission with a scope. Do not enable the plugin's `default` permission blindly.

### Custom plugins

If you need a plugin that doesn't exist (e.g., wrapping a platform-specific API), follow the official plugin template. Don't reinvent the permission wiring or the builder conventions.

---

## Build, bundle, and distribution

This is where a Tauri project ships or fails to ship. Code signing, notarization, updater keys — all of it is fiddly the first time, and none of it can be skipped for production.

### Identifier and versioning

The app's `identifier` in `tauri.conf.json` (reverse-DNS, e.g., `com.acme.myapp`) is baked into:

- macOS bundle ID
- Windows registry keys
- Application data directory paths (`$APPCONFIG`, `$APPDATA`)
- Notification source
- Deep-link handlers

**Change it after release and users lose their data.** Pick it carefully before v1.

Versions in `tauri.conf.json` and `src-tauri/Cargo.toml` must match. A release script should update both and commit them together.

### The updater

`tauri-plugin-updater` provides in-app updates. Two things to get right:

1. **Signatures are mandatory.** Every update artifact must be signed. You cannot ship unsigned updates. Generate a key pair once:

   ```bash
   npx @tauri-apps/cli signer generate -w ~/.tauri/myapp.key
   ```

   The **public key** goes in `tauri.conf.json`. The **private key** goes in CI (GitHub Actions secret `TAURI_SIGNING_PRIVATE_KEY` and optionally a passphrase in `TAURI_SIGNING_PRIVATE_KEY_PASSWORD`). If you lose the private key you can **never publish updates again** to existing installs. If an attacker gets the private key they can impersonate you to every existing user. Treat it like the key to the kingdom.

2. **Host `latest.json`** at a stable URL (GitHub Releases, S3, CrabNebula Cloud, your own CDN). The updater fetches it on startup or on a command to check for updates. The URL goes in `tauri.conf.json` under `plugins.updater.endpoints`.

Prompt for consent before installing. Silent auto-update feels great in hello-world apps and terrible to users who lost work to a surprise restart.

### Code signing

Without signing, your app is blocked or warned about on macOS and Windows. The ceiling of effort:

**macOS:**

- Enroll in the Apple Developer Program ($99/year).
- Get a Developer ID Application certificate.
- Sign the `.app` with `codesign` (Tauri wraps this).
- Notarize with Apple via `xcrun notarytool` or App Store Connect API (Tauri wraps this).
- Staple the notarization ticket.
- **Important:** Tauri's WebView needs two entitlements or the app crashes on launch after notarization:
  - `com.apple.security.cs.allow-jit`
  - `com.apple.security.cs.allow-unsigned-executable-memory`
  These are set via the macOS bundle config in `tauri.conf.json`.
- If you ship Intel + Apple Silicon, build a universal binary (`tauri build --target universal-apple-darwin`) and sign the universal.

**Windows:**

- Authenticode signature required to avoid SmartScreen warnings.
- Options:
  - **Azure Trusted Signing** — the modern option, no hardware token.
  - **EV certificate on a hardware token** — the traditional option, ~$300–$500/year, loses less reputation (SmartScreen trusts EV-signed code faster).
- Automate signing in CI. The hardware-token path is painful in CI; Azure Trusted Signing is straightforward.

**Linux:**

- Signing is optional. AppImage / .deb / .rpm, optionally GPG-signed.
- The Flathub route is a separate distribution — worth it if you have the users.

### CI/CD

Use the official [tauri-apps/tauri-action](https://github.com/tauri-apps/tauri-action). A single tag push can build, sign, notarize, and upload installers for all three platforms. The workflow is verbose (three matrix OSes, secrets for each) but it's the path of least surprise.

Do not skip hooks or add `--force` / `--no-verify` to CI. Do not modify CI workflow files without explicit approval from the maintainer — they affect every developer.

CI must run, at minimum:

1. Frontend type check, lint, test, build.
2. Rust `fmt --check`, `clippy -- -D warnings`, `test`.
3. A release `tauri build` smoke test on each target.

Before merging a PR that touches the build, run the full thing on a real PR once. Don't find out in the release that the Windows build is broken.

### Environment variables

Vite reads `VITE_*` vars at build time and inlines them. Unprefixed vars are not exposed to the bundle (but are accessible in `vite.config.ts`). **Never put secrets in `VITE_*`** — they ship inside the installer.

Commit `.env.example` with placeholder values. Never commit `.env` with real credentials.

For Rust, read config via `std::env` or `config` crate. For runtime configuration (URLs, flags), put it in an app config file in `$APPCONFIG`, not the binary.

### Bundle size

Track the installer size across releases. A 15 MB to 45 MB jump between point releases deserves investigation — you almost certainly pulled in a heavy dependency or forgot to lazy-load.

Tools:

- `vite-bundle-visualizer` — frontend.
- `cargo bloat --release -n 20` — Rust.
- On macOS, `otool -L` shows dynamically linked libraries.
- Strip the binary: `[profile.release] strip = true`.

Desktop users will pay a 30 MB installer gladly for real value, and resent a 5 MB installer that does nothing. Size is a signal, not the goal.

---

## Tooling

### Required

- **TypeScript** strict mode.
- **ESLint** with:
  - `@typescript-eslint/recommended-type-checked` (or the flat-config equivalent).
  - `eslint-plugin-react`, `eslint-plugin-react-hooks`.
  - `eslint-plugin-jsx-a11y` (recommended config).
  - `eslint-plugin-simple-import-sort`.
  - `eslint-plugin-import` with `no-cycle: 'error'`.
- **Prettier** for formatting. `prettier-plugin-tailwindcss` if using Tailwind.
- **Husky** + **lint-staged** for pre-commit gates. On Rust changes, run `cargo fmt --check` and `cargo clippy -- -D warnings`. On TS changes, run eslint and prettier.
- **Rust**: `rustfmt`, `clippy`, `cargo-audit` (or `cargo-deny`), `cargo-machete` (unused deps).

### Recommended

- `vite-plugin-checker` for real-time TS/ESLint errors in the browser during dev.
- React DevTools, SWR/Query DevTools.
- `tracing-subscriber` + `tracing` for structured Rust logs (`RUST_LOG=debug cargo tauri dev`).

### Enforced ESLint rules

A minimal rule set that catches real bugs:

```
react/no-danger: error
react/jsx-no-bind: warn         # disable if using React Compiler
import/no-cycle: error
simple-import-sort/imports: error
simple-import-sort/exports: error
no-console: [warn, { allow: [warn, error] }]
@typescript-eslint/no-explicit-any: error
@typescript-eslint/consistent-type-imports: warn
```

---

## Large-project maintenance

What to do when the codebase is past the "weekend project" phase.

### Dependencies

- `npm audit` and `cargo audit` (or `cargo deny check advisories`) monthly or in CI. Critical and high are blockers.
- One major version bump per PR. Bumping everything at once makes regressions impossible to bisect.
- Pin majors in `package.json` and `Cargo.toml`. Commit lockfiles.
- Remove unused dependencies regularly: `npx depcheck` (JS), `cargo machete` (Rust).

### Dead code

Delete unused components, hooks, commands, feature flags whose rollout is complete. Commented-out code is worse than deleted code — it survives refactors and rots.

Tools:

- `ts-unused-exports` and `knip` for TypeScript.
- `cargo +nightly udeps` for Rust (nightly-only).

### Refactoring

- **Only refactor when the task requires it**, or when the user asks. A bug fix doesn't need surrounding cleanup.
- **One refactor per PR.** Don't mix refactoring with feature work. Mixed PRs are impossible to review.
- Make sure tests exist before refactoring. If they don't, write them first against the current behavior, then refactor.
- When you see tech debt during a task, **flag it, don't silently fix it**. The user decides scope and priority.

### Incremental migration

Any migration larger than a single file is incremental. A big-bang migration that touches 50 files is never green in CI and never lands. Instead:

- Migrate one feature/module at a time.
- Old and new patterns coexist temporarily.
- Document the canonical pattern in `CLAUDE.md`: "New commands MUST return `Result<T, AppError>`. Do not add new `anyhow::Result` returns."
- Track what's left in an issue; close it when the last module flips.

### Performance budgets

- `size-limit` in CI: fail the build if bundle grows past the budget.
- Track Rust binary size across releases — a sudden jump is a signal.
- Monitor app startup time, memory, CPU in production via telemetry (Sentry performance, custom metrics).
- Check `bundlephobia.com` before adding JS deps. Check `cargo bloat` before merging a dep-heavy Rust PR.

Common swaps:

- `moment` → `date-fns` or `dayjs` (~95% smaller).
- `lodash` → native JS or `lodash-es` (~90% smaller).
- `axios` → native `fetch` with a tiny wrapper.
- `uuid` → `crypto.randomUUID()` (native, no dep).
- `classnames` → `clsx` (~76% smaller).
- Apollo Client → SWR or TanStack Query (~60–88% smaller).

---

## Common pitfalls

A short list of things this guide's readers have hit, so you don't have to:

1. **Returning `anyhow::Result` from a command.** Won't compile. Use `AppError`.
2. **Holding a `std::sync::Mutex` guard across `.await`.** Compiles, deadlocks. Use `tokio::sync::Mutex`.
3. **Forgetting to register a command in `generate_handler!`.** Silent failure — the frontend gets "command not found."
4. **Forgetting to grant a permission for a registered command.** Command exists in Rust, frontend can't reach it. Check `src-tauri/capabilities/`.
5. **Unsigned updater.** Generate signing keys now, not at release time.
6. **Changing the `identifier` after release.** Wipes the user's data directory. Commit early, don't change.
7. **Tokens in `localStorage`.** Use the OS keychain.
8. **Base64-encoded images over IPC.** Slow. Use `asset:` protocol or channels.
9. **`unsafe-inline` in CSP.** Defeats the point. Replace the dependency.
10. **Blanket `fs:allow-*` permissions.** Scope to specific paths.
11. **Mocking Tauri with a real webview in unit tests.** Use `@tauri-apps/api/mocks`.
12. **WebDriver E2E on macOS.** Unsupported. Test on Linux/Windows + manual macOS smoke.
13. **Missing cleanup in `useEffect` for Tauri `listen`.** Leaks listeners across remounts.
14. **`useEffect` to derive state from props.** Compute during render.
15. **Storing server/IPC data in `useState`.** Use SWR/TanStack Query.
16. **Plugin version mismatch with Tauri core.** Pin to the same major (2.x).
17. **Missing macOS WebView entitlements after notarization.** App crashes on launch. Add `allow-jit` and `allow-unsigned-executable-memory`.
18. **Relying on `passWithNoTests: true`.** It's a red flag, not a feature.

---

## Closing

None of this is surprising if you've shipped a production Tauri app before. It's all surprising if you haven't. The rules in this repo's `CLAUDE.md` codify the non-obvious parts — the IPC error discipline, the capability scoping, the secret storage hygiene, the updater key management — so the next contributor (human or AI) doesn't have to rediscover them.

When rules feel wrong for your project, change them. The `Project-Specific Overrides` block at the bottom of `CLAUDE.md` is the right place. Just make the change deliberately, and write down why — the "why" is what you'll thank yourself for six months in.

---

## Further reading

- [Tauri v2 documentation](https://v2.tauri.app) — the authoritative reference
- [Tauri Security: Capabilities](https://v2.tauri.app/security/capabilities/) and [Permissions](https://v2.tauri.app/security/permissions/)
- [Tauri Content Security Policy](https://v2.tauri.app/security/csp/)
- [Tauri Isolation Pattern](https://v2.tauri.app/concept/inter-process-communication/isolation/)
- [Calling Rust from the Frontend](https://v2.tauri.app/develop/calling-rust/)
- [Tauri State Management](https://v2.tauri.app/develop/state-management/)
- [Plugins Workspace](https://github.com/tauri-apps/plugins-workspace) — all official plugins
- [Tauri Updater](https://v2.tauri.app/plugin/updater/)
- [macOS Code Signing](https://v2.tauri.app/distribute/sign/macos/)
- [Tauri Tests](https://v2.tauri.app/develop/tests/) and [WebDriver](https://v2.tauri.app/develop/tests/webdriver/)
- [thiserror](https://docs.rs/thiserror) and [anyhow](https://docs.rs/anyhow) — Rust error handling
- [tauri-specta](https://github.com/oscartbeaumont/tauri-specta) — generate TypeScript types from Rust
- [Bulletproof React](https://github.com/alan2207/bulletproof-react) — reference architecture for production React apps
- [React: You Might Not Need an Effect](https://react.dev/learn/you-might-not-need-an-effect)
- [TanStack Query docs](https://tanstack.com/query/latest)
- [Testing Library docs](https://testing-library.com/docs/react-testing-library/intro/)
