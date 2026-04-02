# open-vibe-island

> 我不想在自己的电脑上运行一个闭源、付费的软件来监视我所有的生产过程。<br>
> 所以我 build 了这个开源的版本。<br>
>
> To all vibe coders: 我们自己构建自己的产品。

An open-source macOS companion for terminal-native AI coding agents.

`open-vibe-island` is building a native control surface for local agent workflows on macOS: monitor sessions, surface permission requests and questions, and jump back to the right terminal context without replacing the CLI.

## Why This Exists

The point of this project is straightforward:

- local AI workflows should stay local
- approval and monitoring tools should be inspectable and hackable
- developers should not need a closed-source paid app watching their machine to stay in control

This repository is an attempt to build that layer in the open.

## What It Is

`open-vibe-island` is a native Swift app for macOS that sits in the notch or top bar and acts as a lightweight companion for coding agents. It is designed to stay out of the way until you need it, then give you a fast control surface for:

- session visibility
- permission approval
- answering agent questions
- jumping back into the right terminal or editor context

This project is not trying to replace the terminal. The terminal remains the primary interface. The island attaches to that workflow and makes it easier to manage.

## Project Status

This is an early but buildable prototype.

The repository currently includes:

- `VibeIslandApp` for the SwiftUI and AppKit shell
- `VibeIslandCore` for shared event and session state logic
- `VibeIslandHooks` for Codex hook ingestion over stdin/stdout
- `VibeIslandSetup` for reversible Codex hook installation and removal
- a local Unix-socket bridge between the app and external hook processes
- core tests for session state transitions

The repository name is now `open-vibe-island`. The Swift package and modules still use the `VibeIsland` prefix for now.

## Current Scope

The current implementation is intentionally narrow:

- macOS only
- native SwiftUI/AppKit app
- local-first communication over Unix sockets
- Codex-first integration
- focus on interaction, not passive dashboards

That narrow scope is deliberate. The goal is to get one end-to-end loop working well before expanding to more agents.

## Current Capabilities

Today, the project can already cover the main skeleton of the workflow:

- receive Codex hook events through a local helper
- normalize those events into shared session state
- surface session and approval state in the app shell
- install or uninstall managed Codex hooks from `~/.codex`
- record terminal hints for best-effort jump back behavior

## Architecture At A Glance

The system shape is currently:

1. Codex runs in the user’s existing terminal session.
2. Codex invokes `VibeIslandHooks` from `hooks.json`.
3. The helper forwards hook payloads to the app bridge over a Unix socket.
4. The app consumes normalized agent events and renders state.
5. Approval decisions can be sent back through the same bridge.

Two design rules matter here:

- keep the terminal entrypoint unchanged
- keep installation and rollback explicit and reversible

More detail lives in [docs/architecture.md](docs/architecture.md) and [docs/product.md](docs/product.md).

## Getting Started

### Build And Test

```bash
swift test
swift build
open Package.swift
```

Open the package in Xcode to run the macOS app target. The app starts a local bridge and waits for Codex hook events. If you want to see the older mock timeline, use `Restart Demo` in the UI.

### Codex Hook Setup

Enable the official Codex hook feature once:

```toml
[features]
codex_hooks = true
```

Build the helper:

```bash
swift build -c release --product VibeIslandHooks
```

Install or inspect the managed hook setup:

```bash
swift run VibeIslandSetup install --hooks-binary "$(pwd)/.build/release/VibeIslandHooks"
swift run VibeIslandSetup status --hooks-binary "$(pwd)/.build/release/VibeIslandHooks"
swift run VibeIslandSetup uninstall
```

The installer:

- enables `[features].codex_hooks = true` if needed
- merges Vibe Island hook handlers into `~/.codex/hooks.json` without deleting unrelated hooks
- writes a manifest so uninstall only removes what this project added
- creates timestamped backups before rewriting `config.toml` or `hooks.json`

If you want to manage the files yourself, a minimal `~/.codex/hooks.json` shape looks like:

```json
{
  "hooks": {
    "SessionStart": [
      {
        "matcher": "startup|resume",
        "hooks": [
          {
            "type": "command",
            "command": "/Users/you/path/to/open-vibe-island/.build/release/VibeIslandHooks"
          }
        ]
      }
    ],
    "UserPromptSubmit": [
      {
        "hooks": [
          {
            "type": "command",
            "command": "/Users/you/path/to/open-vibe-island/.build/release/VibeIslandHooks"
          }
        ]
      }
    ],
    "PreToolUse": [
      {
        "matcher": "Bash",
        "hooks": [
          {
            "type": "command",
            "command": "/Users/you/path/to/open-vibe-island/.build/release/VibeIslandHooks"
          }
        ]
      }
    ],
    "PostToolUse": [
      {
        "matcher": "Bash",
        "hooks": [
          {
            "type": "command",
            "command": "/Users/you/path/to/open-vibe-island/.build/release/VibeIslandHooks"
          }
        ]
      }
    ],
    "Stop": [
      {
        "hooks": [
          {
            "type": "command",
            "command": "/Users/you/path/to/open-vibe-island/.build/release/VibeIslandHooks"
          }
        ]
      }
    ]
  }
}
```

## Jump Back

Codex hook ingestion captures terminal hints from the hook process environment, including `TERM_PROGRAM`, `ITERM_SESSION_ID`, and Ghostty-specific variables. The app uses those hints to power a best-effort jump action:

- store terminal-specific locators when available
- focus the matching iTerm session, Ghostty terminal, or Terminal tab before falling back
- reopen the recorded working directory in that terminal as the final fallback

The point is not perfect pane restoration on day one. The point is to make returning to the live agent session fast enough to preserve flow.

## Roadmap

1. `v0.1` Single-agent MVP with real Codex hook monitoring and overlay UI
2. `v0.2` Approval flow hardening, terminal jump, and install automation
3. `v0.3` Multi-session state and better external-display behavior
4. `v0.4` Multi-agent adapters and setup automation

## Development Principles

- keep the app local-first
- prefer native platform APIs over cross-platform abstractions
- build narrow end-to-end slices before adding breadth
- treat hooks, IPC, and focus restoration as first-class engineering concerns
- keep third-party config edits reversible

## Repository Layout

- `Package.swift` Swift package entry point
- `Sources/VibeIslandApp` macOS UI shell
- `Sources/VibeIslandCore` shared models, bridge protocol, installer logic, and session reducer
- `Sources/VibeIslandHooks` Codex hook executable
- `Sources/VibeIslandSetup` setup and uninstall CLI
- `Tests/VibeIslandCoreTests` core test coverage
- `docs/product.md` product scope and MVP boundary
- `docs/architecture.md` architecture notes and event flow

## Contributing

Issues and pull requests are welcome.

If you want to contribute:

- keep changes incremental and reviewable
- prefer one coherent change per PR
- include the most relevant verification for the change
- avoid broad speculative refactors

If you want to propose a larger direction change, open an issue first so the scope is explicit before code starts moving.
