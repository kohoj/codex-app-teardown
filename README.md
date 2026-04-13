# OpenAI Codex Desktop App — Tech Stack Teardown

Reverse-engineered tech stack analysis of the OpenAI Codex desktop application (macOS).

> **Disclaimer**: This is a third-party analysis based on publicly available files within the installed application bundle. No proprietary source code was accessed or decompiled.

---

## Overview

| Property | Value |
|----------|-------|
| Package Name | `openai-codex-electron` |
| Version | `26.409.20454` (Build 1462) |
| Bundle ID | `com.openai.codex` |
| Total Size | **460MB** |
| Min macOS | 12.0 |
| Architecture | ARM64 (Apple Silicon) |
| Package Manager | pnpm (monorepo, workspace protocol) |

---

## 1. Desktop Container Layer

| Component | Version | Purpose |
|-----------|---------|---------|
| **Electron** | 41.2.0 | Desktop app framework |
| **Chromium** | 146.0.7680.179 | Rendering engine |
| **Node.js** | v24.14.0 | Main process runtime |
| **Electron Forge** | 7.11.1 | Build/package/publish pipeline |
| **Sparkle** | - | macOS auto-update (Ed25519 signed) |
| **Squirrel** | - | Windows auto-update |
| **@electron/fuses** | 1.8.0 | Electron security fuses |
| **@electron/notarize** | 3.1.1 | macOS notarization |

### 1.1 Main Process Architecture

```
bootstrap.js → main.js (lazy loaded)
                ├── IpcRouter (inter-window message routing)
                ├── WindowContext (multi-window management)
                ├── SparkleManager (update management)
                ├── ElectronAppServerConnection (Rust CLI communication)
                │   ├── stdio transport (local)
                │   └── WebSocket transport (remote SSH)
                ├── StdioConnection (spawn Rust binary)
                └── Sentry (error reporting)
```

- **IPC channel naming**: `codex_desktop:*` prefix
- **Preload script**: Exposes safe API via `contextBridge.exposeInMainWorld('electronBridge', ...)`
- **Worker process**: Standalone worker.js, supports message routing between multiple webContents

---

## 2. Rust Core Engine (codex CLI)

**Binary size**: 144MB (ARM64 Mach-O)

### 2.1 Internal Module Structure

| Module (crate) | Responsibility |
|-----------------|---------------|
| `codex-core` | Core agent loop / tool registry / execution |
| `codex-api` | OpenAI API client (SSE, WebSocket, Responses API) |
| `codex-client` | HTTP transport / custom CA / TLS |
| `codex-cli` | CLI entry / login flow |
| `codex-tui` | Terminal TUI interface |
| `codex-mcp` / `codex-mcp-server` | MCP protocol (client + server) |
| `codex-exec-server` | Command execution / sandbox |
| `codex-linux-sandbox` | Linux sandbox (bwrap / landlock) |
| `codex-app-server` | Electron ↔ CLI RPC service |
| `codex-app-server-protocol` | RPC protocol definitions (JSON-RPC style) |
| `codex-spawn` | Sub-agent spawning |
| `codex-config` | Configuration management |
| `codex-hooks` | Hook system |
| `codex-tools` | Tool registry / permission control |
| `codex-rollout` | Conversation replay / persistence |
| `codex-cloud-tasks` | Cloud task system |
| `codex-realtime-webrtc` | WebRTC real-time audio (tokio) |
| `codex-network-proxy` | Network proxy |
| `codex-git-utils` | Git operations |
| `codex-file-search` | File search |
| `codex-js-repl` | JavaScript REPL |
| `codex-plugin` | Plugin system |
| `codex-core-skills` | Built-in skills |
| `codex-analytics` / `codex-otel` | Telemetry / analytics |

### 2.2 Key Rust Dependencies (~200+ crates)

| Domain | Crates |
|--------|--------|
| **Async Runtime** | tokio 1.49, futures 0.3.31 |
| **HTTP** | hyper 1.8, reqwest 0.12.28, axum 0.8.8, tower 0.5.3 |
| **TLS** | rustls 0.23.36, ring 0.17.14, aws-lc-rs 1.16.2 |
| **Serialization** | serde 1.0.228, serde_json 1.0.149, serde_yaml 0.9.34 |
| **Database** | sqlx-sqlite 0.8.6 |
| **Code Parsing** | tree-sitter 0.25.10, syntect 5.3.0, onig 6.5.1 |
| **MCP** | rmcp 0.15.0 (Rust MCP client) |
| **DNS** | hickory-resolver 0.25.2 |
| **JWT/Auth** | jsonwebtoken 9.3.1, keyring 3.6.3 |
| **Terminal** | portable-pty 0.9.0 |
| **Image Processing** | image 0.25.9, png 0.18.0, gif 0.14.1 |
| **Text Processing** | pulldown-cmark 0.10.3, similar 2.7.0 (diff), diffy 0.4.2 |
| **Search** | globset 0.4.18, ignore 0.4.25, rust-stemmers 1.2.0, stop-words 0.9.0 |
| **Caching** | moka 0.12.13 (concurrent cache), lru 0.12/0.16 |
| **Config** | toml 0.9.11, plist 1.8.0, dotenvy 0.15.7 |
| **Observability** | sentry 0.46.1, opentelemetry 0.31.0, tracing 0.3.22 |
| **Audio** | cpal 0.15.3, coreaudio-rs 0.11.3 |
| **Network Proxy** | rama-* 0.3.0 (HTTP proxy framework) |
| **Compression** | zstd-safe 7.2.4, zip 2.4.2, lzma-rs 0.3.0 |
| **File Watching** | notify 8.2.0, fsevent-sys 4.1.0 |
| **Starlark** | starlark 0.13.0 (config scripting language) |
| **Type Export** | ts-rs 11.1.0 (Rust → TypeScript type bindings) |

### 2.3 Communication Protocol

Electron ↔ Rust CLI uses **JSON-RPC** style RPC:
- **Local mode**: stdio transport (spawns Rust binary)
- **Remote mode**: WebSocket transport (SSH forwarding to remote servers)
- Methods: `thread/list`, `account/read`, `config/read`, `skills/list`, `model/list`, `collaborationMode/list`, etc.

---

## 3. Frontend UI Layer

### 3.1 Framework Core

| Library | Version | Purpose |
|---------|---------|---------|
| **React** | 19.2.0 | UI framework |
| **React DOM** | 19.2.0 | DOM rendering |
| **React Router** | 7.13.1 | Routing |
| **Vite** | 8.0.3 | Frontend build (via `@electron-forge/plugin-vite`) |

### 3.2 State Management

| Library | Version | Purpose |
|---------|---------|---------|
| **Jotai** | 2.19.0 | Primary state management (atomic) |
| **Jotai Effect** | 2.2.3 | Side effect handling |
| **Jotai TanStack Query** | 0.11.0 | Async data ↔ Jotai integration |
| **@preact/signals** | 1.3.4 | Reactive signals for perf-critical areas |
| **Immer** | 10.1.3 | Immutable data operations |
| **@tanstack/react-query** | 5.90.5 | Server state / caching |
| **@tanstack/react-form** | 1.27.7 | Form state |
| **use-sync-external-store** | 1.6.0 | External store sync |

### 3.3 UI Component Library

| Library | Version | Purpose |
|---------|---------|---------|
| **Radix UI** | Multiple 1.x/2.x | Unstyled primitive components (~30 packages) |
| **cmdk** | 1.1.1 | Command Palette (⌘K) |
| **Framer Motion** | 12.23.24 | Animation engine |
| **@dnd-kit** | core 6.3.1 / sortable 8.0.0 | Drag and drop sorting |
| **Embla Carousel** | 8.6.0 | Carousel component |
| **@floating-ui/react** | 0.27.16 | Floating positioning (tooltip/popover) |
| **react-colorful** | 5.6.1 | Color picker |
| **@headless-tree/core** | 1.6.1 | Tree component |
| **react-remove-scroll** | 2.7.1 | Scroll locking |
| **use-stick-to-bottom** | 1.1.1 | Chat auto-scroll to bottom |

### 3.4 Styling

| Library | Version | Purpose |
|---------|---------|---------|
| **Tailwind CSS** | (runtime) | Atomic CSS framework |
| **tailwind-merge** | 1.14.0 | Tailwind class merging/dedup |
| **tailwind-styled-components** | 2.2.0 | Styled-components wrapper for Tailwind |
| **clsx** | 2.1.1 | Conditional class name joining |
| **stylis** | 4.3.6 | CSS preprocessor |

CSS output: Only **5 CSS files** (highly optimized Vite build)

### 3.5 Editor / Text Processing

| Library | Version | Purpose |
|---------|---------|---------|
| **ProseMirror** | Full suite 1.x | Chat input editor |
| **Lexical** | 0.32.1 (incl. @lexical/react) | Rich text editor (possibly for different scenarios) |
| **Shiki** | 3.20.0 + 3.23.0 | Code syntax highlighting |
| **Highlight.js** | 11.11.1 | Code highlighting (fallback) |
| **PrismJS** | 1.30.0 | Code highlighting (fallback) |

Shiki ships with **~448 programming language** grammar files and **~50 code themes** (catppuccin, dracula, github-dark, monokai, nord, tokyo-night, etc.).

### 3.6 Terminal Emulator

| Library | Version | Purpose |
|---------|---------|---------|
| **@xterm/xterm** | 5.5.0 | Terminal rendering |
| **@xterm/addon-fit** | 0.10.0 | Terminal auto-resize |
| **@xterm/addon-web-links** | 0.11.0 | Terminal link detection |
| **@xterm/addon-clipboard** | 0.1.0 | Terminal clipboard |
| **node-pty** | 1.1.0 (native module) | Pseudo-terminal process |

### 3.7 Markdown / Math

| Library | Version | Purpose |
|---------|---------|---------|
| **react-markdown** | 10.1.0 | Markdown rendering |
| **remark** + **rehype** | Full suite | Markdown AST processing pipeline |
| **remark-gfm** | 4.0.1 | GitHub Flavored Markdown |
| **remark-breaks** | 4.0.0 | Line break handling |
| **remark-directive** | 4.0.0 | Custom directives |
| **KaTeX** | 0.16.25 | Math formula rendering (with full font set, 59 font files) |
| **rehype-katex** | 7.0.1 | KaTeX ↔ rehype integration |
| **DOMPurify** | 3.3.0 | HTML sanitization |
| **Marked** | 16.4.1 | Markdown parsing (fallback) |

### 3.8 Diagrams / Visualization

| Library | Version | Purpose |
|---------|---------|---------|
| **Mermaid** | 11.12.0 | Diagram rendering (flowchart/sequence/gantt/ER/etc.) |
| **D3.js** | 7.9.0 | Data visualization |
| **Cytoscape** | 3.33.1 | Graph/network visualization |
| **roughjs** | 4.6.6 | Hand-drawn style graphics |
| **dagre-d3-es** | 7.0.11 | Directed graph layout |
| **d3-sankey** | 0.12.3 | Sankey diagrams |
| **rrule** | 2.8.1 | Recurrence rules |

Supported Mermaid diagram types (inferred from chunk filenames):
flowDiagram, sequenceDiagram, classDiagram, stateDiagram, erDiagram, ganttDiagram, pieDiagram, gitGraph, mindmap, timeline, c4Diagram, sankeyDiagram, xychartDiagram, quadrantDiagram, blockDiagram, packet, radar, treemap, kanban, architecture, etc.

### 3.9 PDF Processing

| Library | Version | Purpose |
|---------|---------|---------|
| **react-pdf** | 10.3.0 | PDF preview |
| **pdfjs-dist** | 5.4.296 | PDF parsing engine |

### 3.10 Internationalization (i18n)

| Library | Version | Purpose |
|---------|---------|---------|
| **react-intl** | 7.1.14 | React i18n |
| **@formatjs/intl** | 3.1.8 | ICU message format |

Supports **50+ languages/regions**

### 3.11 Animation & Visual Effects

| Library | Version | Purpose |
|---------|---------|---------|
| **@lottiefiles/dotlottie-react** | 0.17.13 | Lottie animations |
| **Framer Motion** | 12.23.24 | Transition/gesture animations |
| **Codex Spritesheet** | - | Custom sprite sheet (codex-spritesheet.webp) |

Built-in animations (from JS filenames): `analyze_image_animation`, `browsing_animation`, `edit_files_animation`, `list_files_animation`, `run_command_animation`, `searching_animation`, `local_context_animation`, `to_do_animation`

### 3.12 Other Utility Libraries

| Library | Version | Purpose |
|---------|---------|---------|
| **Fuse.js** | 7.1.0 | Fuzzy search |
| **ts-pattern** | 5.8.0 | Pattern matching |
| **Zod** | 4.1.13 | Runtime type validation |
| **dayjs** | 1.11.18 | Date handling |
| **lodash** | 4.17.21 | Utility functions |
| **uuid** | 11.1.0 | UUID generation |
| **diff** | 8.0.3 | Text diff |
| **memoizee** | 0.4.17 | Function memoization |
| **jose** | 6.1.3 | JWT/JWS/JWE handling |
| **react-scan** | 0.5.3 | React performance debugging |
| **bippy** | 0.5.32 | React fiber interception (react-scan dep) |
| **react-error-boundary** | 3.1.4 | Error boundary |
| **react-textarea-autosize** | 8.5.9 | Auto-height textarea |

---

## 4. Native Modules (app.asar.unpacked)

| Module | Purpose |
|--------|---------|
| **better-sqlite3** 12.8.0 | SQLite database (conversation/thread persistence) |
| **node-pty** 1.1.0 | Native pseudo-terminal |

---

## 5. Observability / Infrastructure

| Component | Version | Purpose |
|-----------|---------|---------|
| **Sentry** | electron 7.5.0 / node 10.29.0 | Error tracking / session replay |
| **OpenTelemetry** | Full suite (32 packages) | Distributed tracing |
| **Statsig** | @statsig/js-client 3.32.2 | Feature flags / A-B experiments / gating |

---

## 6. Protocols / Integrations

| Component | Version | Purpose |
|-----------|---------|---------|
| **MCP SDK** | @modelcontextprotocol/sdk 1.24.3 (JS) + rmcp 0.15.0 (Rust) | Model Context Protocol |
| **VSCode LSP** | vscode-jsonrpc 8.2.0, vscode-languageserver 9.0.1 | Language Server Protocol |
| **@pierre/diffs** | 1.1.1 | Custom diff rendering |
| **@pierre/theme** | 0.0.22 | Custom theme system |
| **@pierre/trees** | 0.0.1-beta.4 | Custom tree component |
| **YJS** | 13.6.27 | CRDT collaborative editing |
| **Express** | 5.1.0 | Embedded HTTP server |
| **SSH Config** | 5.1.0 | SSH config parsing |
| **SOCKS Proxy** | socks-proxy-agent 8.0.5 | Proxy support |

---

## 7. Supported IDE / Editor Integrations

From `/webview/apps/` icon assets:
Android Studio, BBEdit, Cmder, **Cursor**, Finder, **Ghostty**, GoLand, IntelliJ, **iTerm2**, Microsoft Terminal, PHPStorm, PyCharm, Rider, RustRover, Sublime Text, **Terminal**, TextMate, **VS Code** (incl. Insiders), **Warp**, WebStorm, **Windsurf**, Xcode, **Zed**

---

## 8. Dev Toolchain

| Tool | Version | Purpose |
|------|---------|---------|
| **TypeScript** | 5.9.3 | Type system |
| **tsgo** | - | Go-based TS compiler (Microsoft) |
| **Vite** | 8.0.3 | Frontend build / HMR |
| **Vitest** | 4.1.2 | Unit testing |
| **Playwright** | 1.58.2 | E2E testing |
| **oxlint** | - | Rust-based ultra-fast linter |
| **oxfmt** | - | Rust-based code formatter |
| **esbuild** | 0.25.11 | Native module build |
| **cross-env** | 7.0.3 | Cross-platform env vars |

---

## 9. Frontend Asset Statistics

| Type | Count |
|------|-------|
| JS Chunks | 615 |
| CSS Files | 5 |
| Font Files | 59 (all KaTeX math fonts) |
| Image Assets | 31 (IDE icons, etc.) |
| Syntax Grammars | ~448 |
| Code Themes | ~50 |

---

## 10. Architecture Diagram

```
┌─────────────────────────────────────────────────────┐
│                  Electron Shell                      │
│  ┌──────────────────┐  ┌──────────────────────────┐ │
│  │   Main Process    │  │     Renderer (Webview)    │ │
│  │  (Node.js v24)    │  │                          │ │
│  │                  │  │  React 19 + Jotai + Radix │ │
│  │  IpcRouter       │  │  ProseMirror + Lexical    │ │
│  │  SparkleManager  │  │  xterm.js + Mermaid + D3  │ │
│  │  Sentry          │  │  Shiki + KaTeX            │ │
│  │  Statsig         │  │  Tailwind CSS             │ │
│  │  better-sqlite3  │  │  Framer Motion            │ │
│  │  node-pty        │  │                          │ │
│  └────────┬─────────┘  └──────────────────────────┘ │
│           │ stdio / WebSocket (JSON-RPC)             │
│  ┌────────▼─────────────────────────────────────────┐│
│  │          Rust Core Engine (144MB ARM64)           ││
│  │                                                  ││
│  │  codex-core ─── codex-api (OpenAI Responses API) ││
│  │  codex-exec ─── codex-sandbox (bwrap/landlock)   ││
│  │  codex-mcp ──── codex-tools (permissions)        ││
│  │  codex-tui ──── codex-spawn (sub-agents)         ││
│  │  tree-sitter ── syntect ── sqlx-sqlite            ││
│  │  tokio ──────── reqwest ── hyper ── axum          ││
│  │  sentry ─────── opentelemetry ── tracing          ││
│  │  cpal ────────── WebRTC (realtime audio)          ││
│  └──────────────────────────────────────────────────┘│
└─────────────────────────────────────────────────────┘
```

---

## Key Design Highlights

1. **Dual-language architecture**: Electron (TypeScript) handles GUI, Rust handles the core engine — they communicate via JSON-RPC over stdio/WebSocket
2. **Dual editor engines**: ProseMirror + Lexical coexist, likely used for different input scenarios
3. **Triple state management**: Jotai (primary) + @preact/signals (perf hotspots) + @tanstack/react-query (async data)
4. **Collaborative editing**: YJS CRDT framework integrated, hinting at potential future multi-user collaboration
5. **Remote development**: Built-in SSH WebSocket transport for connecting to remote Codex instances
6. **Real-time voice**: Rust side integrates cpal/coreaudio + WebRTC for voice interaction
7. **Dual MCP implementation**: JS side (MCP SDK 1.24.3) + Rust side (rmcp 0.15.0) both implement the MCP protocol
8. **Type bridging**: Uses `ts-rs` to auto-generate TypeScript type definitions from Rust, ensuring cross-language type safety

---

*Analysis performed on 2026-04-14. App version: 26.409.20454 (Build 1462).*
