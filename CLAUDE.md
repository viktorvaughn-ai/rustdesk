# RustDesk — personal fork

Personal fork of [rustdesk/rustdesk](https://github.com/rustdesk/rustdesk) (AGPL-3.0), forked to
`viktorvaughn-ai/rustdesk`. Origin = this fork (push here); `upstream` = the original project (pull
updates from there, never push). General Rust/Flutter conventions, directory layout, and editing
rules for this codebase: see `AGENTS.md` below (upstream-authored, kept as-is).

@AGENTS.md

## Fork status

- Cloned to `~/rustdesk-fork`. `libs/hbb_common` is a git submodule — run
  `git submodule update --init --recursive` after any fresh clone (already done here).
- **Not built yet** — no `build.py` run performed so far. Building this project pulls in the full
  Rust + Flutter toolchain per `docs/` (platform-specific); do that before assuming any binary
  reflects the changes below.

## Fork customizations

Two behavioral changes from stock RustDesk, both scoped to this app's own source (no changes to
the `libs/hbb_common` submodule, so they survive a submodule bump untouched):

1. **No account/login system** (`src/common.rs::load_custom_client`) — forces the built-in
   `HARD_SETTINGS["disable-account"] = "Y"` unconditionally on every startup path (desktop FFI
   init, mobile FFI init, background service), instead of only via a signed `custom.txt` custom-
   client config. This is RustDesk's own official toggle for hiding the account/login UI (login
   button, account settings tab, audit "Note" login prompt, address-book login gating) — we just
   force it on rather than requiring a custom-client build. No hard "must log in to use the app"
   gate existed in the OSS client to begin with; this removes the *optional* account UI entirely.

2. **No Connection Manager (CM) window** (`src/server/connection.rs::start_ipc`) — the CM is a
   second full Flutter-engine process RustDesk normally spawns per incoming connection, for
   click-to-accept approval and live session management (kick/permissions/chat). `start_ipc` now
   returns immediately without ever spawning the `--cm` subprocess, trading away those features
   for the memory the second process would otherwise cost. **This only makes sense because
   `ApproveMode` is password-only** (see `src/server/connection.rs`, `password::approve_mode()`):
   password-authenticated connections are already authorized *before* `try_start_cm` is called and
   don't wait on any CM response, so they work unaffected. If approve mode is ever switched to
   `Click` or `Both`, incoming connections needing manual approval will hang forever with this
   change in place — don't flip approve mode without re-adding the CM spawn.

## Conventions specific to this fork

- Keep changes scoped to files under `src/` and `flutter/` in *this* repo — never edit
  `libs/hbb_common` (separate upstream project via submodule); if a fix seems to need a change
  there, put a thin hook/wrapper on this side instead (per `AGENTS.md`'s submodule guidance).
- Since there's no account system, don't reintroduce login-gated code paths (server sync, address
  book via account, etc.) without first reverting customization #1 above.
- Pushes go to `origin` (this fork). To pull upstream fixes: `git fetch upstream && git merge
  upstream/master` (or rebase), then re-verify both customizations still apply — upstream changes
  to `load_custom_client` or `start_ipc`/`try_start_cm` are the most likely merge-conflict or
  silent-regression points.
