# ISC CAN Studio

Desktop application for **flashing, monitoring, and debugging CAN messages** on
the ISC Racing Team's Formula Student ECUs. The CLI ([`can-flasher`](../../README.md))
covers power-user / CI workflows; the [VS Code extension](../../editor/vscode/)
covers in-editor flashing for developers; this app is the surface for everyone
else — mechanics at a workbench, hardware engineers at a test bench, race-day
operators in the pit.

**Status: v0 scaffold.** Window comes up, frontend loads, two trivial
`#[tauri::command]` calls prove the Rust ↔ JS bridge. Tier 0 (real features)
lands in the next PR.

## Architecture

```
┌───────────────────────────────────────────────┐
│  Tauri 2 native window (Mac / Linux / Win)    │
│  ┌─────────────────────┐                      │
│  │ Svelte 5 frontend   │  ← src/, index.html  │
│  │ (TypeScript + Vite) │                      │
│  └──────────┬──────────┘                      │
│             │ tauri.invoke(…)                 │
│  ┌──────────▼────────────────────────────┐    │
│  │ Rust backend (src-tauri/)             │    │
│  │ ├─ #[tauri::command] surface          │    │
│  │ └─ embeds can-flasher crate by path   │    │
│  │    → protocol/, transport/, flash/    │    │
│  └───────────────────────────────────────┘    │
└───────────────────────────────────────────────┘
```

Same Rust on both sides of the IPC bridge — no shell-out tax, the bootloader
protocol code is reused directly. When a new adapter or a new opcode lands in
`can-flasher`, the Studio picks it up by a Cargo bump.

## Tier roadmap

Same shape as the VS Code extension's evolution.

| Tier | Surface | Status |
|---|---|---|
| **0** | Adapter picker, flash button, DTC viewer + clear, health, port the live-data chart | 🔜 next PR |
| **1** | Generic CAN bus monitor — live frame list, filter by ID, per-ID rate, pause / capture-to-file | 🔜 |
| **2** | DBC file support — decoded signal column, dedicated Signals view, signal-trigger expressions | 🔜 |
| **3** | Frame transmitter — single-shot + cyclic + signal-triggered sends | 🔜 |
| **4** | Record / replay sessions (candump format), multi-channel scope-style charts | 🔜 |

Tier 0 wraps existing CLI capability. Tier 1 is the inflection point where this
becomes a real CAN tool — it needs a generic bus-monitor module in the Rust
core (new crate or new module under `can-flasher`).

## Why Tauri

- Reuses the existing `can-flasher` Rust crates **by path dependency** — no
  shell-out, no JSON parsing of CLI output. Same wire-format code in CLI and
  app, can't drift.
- Native binaries on Mac, Linux, and Windows from one codebase (~10 MB each).
- Web frontend (Svelte 5 + Vite) iterates UI fast without learning a separate
  GUI toolkit.

## Why Svelte 5

- Smallest runtime among modern frameworks. Tauri's own examples lean Svelte.
- Component model is simple enough that anyone on the team can learn it.
- Reactive runes (`$state`, `$derived`, `$effect`) compose cleanly without the
  hooks dance.

## Development

### Prerequisites

- **Node 20+** and **npm** (for the frontend toolchain)
- **Rust 1.95+** with the `rustup` standard target — same toolchain as
  `can-flasher`
- Platform native deps for Tauri (Webkit/GTK on Linux, Xcode CLT on macOS,
  WebView2 on Windows). See <https://tauri.app/start/prerequisites/>.

### Dev loop

```bash
cd apps/can-studio
npm install                # one-time
npm run tauri:dev          # opens the dev window, HMR for the frontend,
                           # cargo-watch for the Rust side
```

### Release build

```bash
npm run tauri:build        # produces a platform-native bundle in
                           # src-tauri/target/release/bundle/
```

Outputs:
- macOS: `bundle/macos/ISC CAN Studio.app` and `bundle/dmg/*.dmg`
- Linux: `bundle/deb/*.deb`, `bundle/appimage/*.AppImage`, `bundle/rpm/*.rpm`
- Windows: `bundle/msi/*.msi`, `bundle/nsis/*.exe`

### Icon generation

The committed `src-tauri/icons/icon.png` is the source. For release builds the
team's tauri CLI should run `npm run tauri icon src-tauri/icons/icon.png` once
to generate `.ico` / `.icns` / multi-resolution PNGs. CI does this automatically;
local dev usually doesn't need it (dev builds use the bare `icon.png`).

## Repository layout

```
apps/can-studio/
├── README.md                  ← you are here
├── package.json               ← frontend tooling + tauri CLI
├── tsconfig.json
├── vite.config.ts
├── svelte.config.js
├── index.html                 ← Vite entry
├── public/icon.png            ← static asset served by Vite
├── src/                       ← Svelte 5 frontend
│   ├── main.ts
│   ├── App.svelte
│   └── app.css
└── src-tauri/                 ← Rust backend (Tauri 2)
    ├── Cargo.toml             ← member of the root workspace
    ├── tauri.conf.json
    ├── build.rs
    ├── icons/icon.png
    └── src/
        ├── main.rs            ← binary entry; calls into lib.rs
        └── lib.rs             ← #[tauri::command] surface
```

## License

MIT — see [LICENSE](../../LICENSE) at the repo root.
