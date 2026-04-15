# Tauri-with-AI

A `CLAUDE.md` template that teaches [Claude Code](https://docs.anthropic.com/en/docs/claude-code) modern Tauri v2 + React best practices. Drop it into any Tauri + React desktop project and Claude will follow production-grade conventions for TypeScript, Rust, IPC, capabilities-based security, accessibility, testing, code signing, and more — automatically, on every interaction.

## Why

Claude Code reads `CLAUDE.md` from your working directory and applies its rules to every conversation (including sub-agents). Without guardrails, AI-generated Tauri code can drift in ways that are hard to see until release day: commands that return `anyhow::Result` and fail to compile, `std::sync::Mutex` held across `.await`, `unsafe-inline` CSP, blanket `fs:allow-*` permissions, tokens in `localStorage`, missing updater signatures, `.unwrap()` in command handlers. This template codifies the conventions experienced Tauri teams enforce in code review — for both the React frontend and the Rust backend — so Claude follows them from the first line.

## Two Versions

Pick the one that fits your project.

| File | Size | Contains | Best for |
|---|---|---|---|
| [`CLAUDE.md.FULL`](./CLAUDE.md.FULL) | 39,944 chars | All rules **plus BAD/GOOD code examples** | Teams that want rules plus concrete patterns — Claude follows rules-with-examples more reliably |
| [`CLAUDE.md.SHORT`](./CLAUDE.md.SHORT) | 26,132 chars | All rules, **no code examples** | Token-constrained setups, or projects that just need guardrails without inline samples |

Both files cover the **same rules** — the only difference is whether each rule is illustrated with a BAD/GOOD code block. SHORT is ~35% smaller because it strips all fenced code blocks while preserving every rule, table, and checklist.

<details>
<summary><strong>What's inside (click to expand)</strong></summary>

| Area | Key rules |
|---|---|
| **TypeScript** | Strict mode, no `any`, discriminated unions, exhaustive switch guards |
| **Rust Backend** | `thiserror`-backed `AppError` with `Serialize`, no `.unwrap()` in commands, `#![deny(unsafe_code)]`, `clippy::pedantic` in CI |
| **Tauri IPC** | Typed `invoke` wrappers in `src/ipc/`, `Result<T, AppError>` returns, `Channel<T>` for streaming, event cleanup on unmount |
| **Capabilities & Permissions** | v2 ACL, least privilege, one capability file per window/feature, scoped `fs:allow-*` paths |
| **CSP & Isolation** | Strict CSP, no `unsafe-inline`/`eval`, isolation pattern for third-party frontend code |
| **Shared Types** | `tauri-specta` or `ts-rs` to generate TS from Rust — single source of truth across the IPC boundary |
| **State Management (Rust)** | `.manage(...)`, no `Arc`, `tokio::sync::Mutex` across `.await`, minimal critical sections |
| **State Management (React)** | Decision tree: SWR/Query → useState → Context → Zustand; never store derived state |
| **Hooks** | useEffect only for external sync, mandatory cleanup, async safety |
| **Data Fetching** | SWR/TanStack Query wrapping typed IPC; `reqwest` in Rust for CORS/header/token cases |
| **Error Handling** | Error boundaries at app/feature/list level; sanitized error variants from Rust |
| **Performance** | Route-level code splitting, minimize IPC payload size, `spawn_blocking`, release profile (`strip`, `lto`) |
| **Accessibility** | Semantic HTML, ARIA labels, headless UI libraries, eslint-plugin-jsx-a11y |
| **Frontend Security** | No unsafe HTML, no tokens in localStorage, OS keychain for secrets |
| **Testing** | Vitest + RTL with `@tauri-apps/api/mocks`; Rust `mock_builder`; WebDriver via `tauri-driver` |
| **Plugins** | Official-first (`fs`, `dialog`, `sql`, `store`, `updater`, `stronghold`, …), version-pinned to Tauri 2.x |
| **Build & Distribution** | Updater key hygiene, macOS Developer ID + notarization + JIT entitlements, Windows Authenticode, bundle budgets |
| **Project Structure** | `src/` + `src-tauri/` layout, colocation, import cycle prevention |
| **Tooling** | ESLint (6 plugins), Prettier, Husky, lint-staged, clippy, rustfmt, cargo-audit |
| **Large Projects** | Dep management, dead code removal, incremental migration |

Both versions also include token-efficient output rules, a **new-project setup checklist**, an **existing-project onboarding checklist**, a **pre-completion verification checklist**, and a customizable **Project-Specific Overrides** section at the bottom.

</details>

## Installation

1. **Download** the version you want into your Tauri project root:

   ```bash
   # FULL (with code examples)
   curl -o CLAUDE.md https://raw.githubusercontent.com/maxart/Tauri-with-AI/main/CLAUDE.md.FULL

   # Or SHORT (rules only)
   curl -o CLAUDE.md https://raw.githubusercontent.com/maxart/Tauri-with-AI/main/CLAUDE.md.SHORT
   ```

   The `curl` commands above already rename the file to `CLAUDE.md` — if you download manually, rename it yourself.

2. **Customize** the `Project-Specific Overrides` section at the bottom of `CLAUDE.md` with your stack details — state management choice, styling approach, IPC conventions, database, updater manifest URL, signing setup.

Claude Code picks it up automatically on every conversation. No slash commands, no configuration.

## How It Works

```
your-tauri-project/
  CLAUDE.md          <-- Claude reads this automatically
  src/               <-- React frontend
  src-tauri/         <-- Rust backend
  package.json
```

Claude Code loads `CLAUDE.md` from the working directory (and parent directories) at the start of every conversation. The instructions become part of Claude's system context and apply to the main agent and all sub-agents.

<details>
<summary><strong>How scope works</strong></summary>

| Placement | Effect |
|---|---|
| Project root | Applies to all work in the project |
| `src/` subdirectory | Applies only when working in `src/` (frontend-specific rules) |
| `src-tauri/` subdirectory | Applies only when working in `src-tauri/` (Rust-specific rules) |
| Multiple levels | Rules stack — deeper files add to (or override) parent rules |

</details>

## Customization

The template is a starting point, not a straitjacket.

- **Remove** rules that conflict with your project (e.g., delete the Tailwind section if you use CSS Modules).
- **Add** project-specific rules anywhere — database conventions, capability-review process, release automation, design system usage.
- **Override** defaults in the `Project-Specific Overrides` block at the bottom.

Rules with code examples (FULL version) are followed more reliably than abstract guidelines. If Claude keeps missing a rule, make it more specific or add an example.

<details>
<summary><strong>Example: Project-Specific Overrides</strong></summary>

```markdown
## Project-Specific Overrides

### State Management
- Global state uses Zustand; server/IPC state uses TanStack Query with 10-minute staleTime.

### IPC
- All commands live under src-tauri/src/commands/<domain>.rs
- All typed wrappers live in src/ipc/<domain>.ts (auto-generated via tauri-specta)
- New commands require a matching capability entry reviewed in PR

### Database
- SQLite via tauri-plugin-sql; migrations in src-tauri/migrations/ run at startup

### Updater
- Manifest at https://releases.example.com/latest.json
- Private key stored as TAURI_SIGNING_PRIVATE_KEY in GitHub Actions secrets

### Distribution
- macOS: Developer ID + notarization via apple-codesign in CI
- Windows: Azure Trusted Signing
- Linux: AppImage + .deb, GPG-signed
```

</details>

## Companion File

| File | Description |
|---|---|
| [`docs/TAURI_REACT_BEST_PRACTICES_GUIDE.md`](./docs/TAURI_REACT_BEST_PRACTICES_GUIDE.md) | Long-form tutorial covering each rule in depth with explanations, alternatives, and gotchas. For learning or onboarding — not meant to be loaded as `CLAUDE.md`. |

## Compatibility

- **Claude Code**: CLI, desktop app, web app, IDE extensions (VS Code, JetBrains)
- **Tauri**: v2.x (stable, released October 2024). Not for Tauri v1 — the security model (capabilities vs. allowlist) and IPC are incompatible.
- **React**: 18.x and 19.x (including the React Compiler)
- **TypeScript**: 5.x
- **Rust**: stable, MSRV tracks Tauri's (1.77+ at time of writing)
- **Build tools**: Vite (default), any bundler supported by Tauri
- **Styling**: Tailwind, CSS Modules, or any approach
- **Platforms**: macOS, Windows, Linux (desktop); iOS, Android (mobile) — Tauri 2 added mobile support

## FAQ

**Which version should I pick?**
Start with **FULL** if you're unsure. The BAD/GOOD examples make Claude more consistent at applying each rule, especially on Tauri-specific patterns (error serialization, capability scoping, Mutex-across-await) where a subtle mistake costs hours. Switch to **SHORT** if context size matters (large monorepo, heavy sub-agent use) or once your team has settled on conventions and Claude reliably follows them.

**Does this work with Tauri v1?**
No. Tauri v2 replaced the v1 allowlist with a capability/permission ACL, and the IPC layer was rewritten with custom protocols and typed channels. Many rules in this template reference v2 concepts that don't exist in v1. If you're on v1, the right move is to migrate — v2 is stable and actively developed.

**Does this work without React?** (Vue, Svelte, Solid, etc.)
The frontend half of the template is React-specific. The Rust/Tauri/IPC/security/distribution rules are framework-agnostic and worth keeping either way. Strip the React sections and drop in your framework's equivalents.

**Will this slow Claude down?**
No. `CLAUDE.md` is loaded once per conversation and is negligible in the context window.

**Can I use this with other AI coding tools?**
The file targets Claude Code, but the content is standard Tauri + React practice and portable:

- **Agents that read `AGENTS.md`** (Codex, OpenCode, and others adopting that convention): download it directly as `AGENTS.md`.

  ```bash
  curl -o AGENTS.md https://raw.githubusercontent.com/maxart/Tauri-with-AI/main/CLAUDE.md.FULL
  ```

- **Both Claude Code and an `AGENTS.md`-based agent in the same repo**: save the file as `AGENTS.md`, then create a one-line `CLAUDE.md` that imports it. Claude Code expands `@path` imports into context at session start, so both tools read identical instructions from a single source — no duplication, no drift.

  ```bash
  curl -o AGENTS.md https://raw.githubusercontent.com/maxart/Tauri-with-AI/main/CLAUDE.md.FULL
  echo '@AGENTS.md' > CLAUDE.md
  ```

- **Cursor, Windsurf, Copilot**: adapt the rules into `.cursor/rules/`, `.windsurfrules`, or `.github/copilot-instructions.md`.

**Claude is ignoring a rule.**
`CLAUDE.md` is high-priority but not absolute. If a rule is consistently ignored, make it more specific or add an example. Rules with code examples are followed more reliably — the main reason to prefer FULL over SHORT.

**Does the template cover mobile (iOS/Android)?**
The core rules apply — Tauri 2's mobile support uses the same IPC, capabilities, and plugin model. Mobile adds its own concerns (code signing via Apple/Google, platform-specific permissions, different webview quirks). Add mobile-specific overrides to the `Project-Specific Overrides` section as you discover them.

## Further Reading

- [Tauri v2 docs](https://v2.tauri.app) — the authoritative reference
- [Tauri Security overview](https://v2.tauri.app/security/) — capabilities, permissions, CSP, isolation
- [Tauri Plugins workspace](https://github.com/tauri-apps/plugins-workspace) — all official plugins in one repo
- [Bulletproof React](https://github.com/alan2207/bulletproof-react) — reference architecture for production React apps
- [React docs: You Might Not Need an Effect](https://react.dev/learn/you-might-not-need-an-effect) — when to reach for `useEffect` and when not to
- [TanStack Query docs](https://tanstack.com/query/latest) — authoritative data-fetching patterns
- [Testing Library docs](https://testing-library.com/docs/react-testing-library/intro/) — query priority and best practices
- [thiserror](https://docs.rs/thiserror) and [anyhow](https://docs.rs/anyhow) — Rust error handling crates used in the template
- [React-with-AI](https://github.com/maxart/React-with-AI) — the React-only companion template this project is based on

## License

MIT
