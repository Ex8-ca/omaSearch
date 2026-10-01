# Changelog

## 1.2.0-hermes.4

`--ask` with image, plus investigation notes on Hermes' per-tool approval model.

### `--ask --image <path>`

- New `--image <path>` flag on `ask.py --ask`. Validates the path lives in `~/.cache/omasearch/shots`, has a recognized extension, and is under 5 MB (reuses `peek_shot`). If validation fails, emits `code: image-rejected` and a human-readable error; the ask does not run.
- `invoke_for` and `session_invoke` thread the image path through to `base_invoke_for`, which appends `--image <path>` for Hermes alongside `--query-file`.
- The shot is unlinked after the ask completes (success or failure) so pasted images don't pile up in `~/.cache/omasearch/shots/`.

### Hermes per-tool approval — investigation notes

Hermes has its own approval model, separate from Claude's per-tool interactive prompts:

- **Hardline blocklist** (`hermes approvals test` → verdict `hardline-deny`, exit 3): always blocked, even under `--yolo`. Recursive `rm`, sudo, disk writes outside `~/.cache`, etc. Matches Claude's behavior under `--permission-mode default` for the truly destructive class.
- **`ask-approval` verdict** (exit 2): commands Hermes would prompt the user for in an interactive session — `systemctl restart <svc>`, `dd`, disk-copy operations, etc. The prompt happens inside Hermes itself, which assumes a TTY. In `ask.py`'s non-TTY context this hangs the process.
- **`allow` verdict** (exit 0): no guard matched; runs without a prompt.

Because the prompt happens inside Hermes (not before the tool call), there's no clean way for `ask.py` to relay "Allow / Always / Deny" to the overlay's UI. The overlay's button stays a no-op for Hermes (`kind: approve_unsupported`).

The cleanest faithful design is:

1. `omaSearch`'s Safe mode = `hermes --safe-mode`. This is the **runtime safety tier** and gates what tools Hermes will use, but does NOT prompt per call.
3. `omaSearch`'s trust-it mode = `hermes --yolo`. Hermes' hardline blocklist still fires (good); the rest runs unprompted.
4. The overlay's `kind: approve_unsupported` for any approval message keeps the Allow/Deny buttons hidden in the UI — surfacing them would be dishonest about what Hermes can gate.

If a future Hermes CLI gains a stdin-based approval protocol, the overlay's `--safe` flow can become interactive.

## 1.2.0-hermes.3

Image paste (Ctrl+V) for Hermes in `--serve` mode.

- New `peek_shot(path)` validates an overlay-supplied image without deleting it: path must live in `~/.cache/omasearch/shots`, have a recognized extension, and be under `MAX_IMAGE_BYTES` (5 MB).
- `_serve_hermes` threads `peek_shot`'s path through `start_hermes`, which appends `--image <path>` to `hermes chat`. Hermes reads the file itself; omaSearch cleans up the shot after the turn completes.
- A path that fails validation surfaces as `kind: image_rejected` with a human-readable reason; the chat stays alive.
- `--ask` mode does not accept images (the existing flow doesn't include them).

## 1.2.0-hermes.2

Streaming `ask.py --serve` for Hermes.

- `serve()` now branches by selected agent: `_serve_claude()` (unchanged) and the new `_serve_hermes()`.
- `_serve_hermes()` maps Hermes' `--format stream-json` events onto omaSearch's `kind` event set: `text` → `delta`, `tool_use` → `tool`, `tool_result` → `tool_result`, `result` → `done`. Tool calls are surfaced for the overlay to show, but the overlay's Allow/Deny flow is a no-op for Hermes — `--safe-mode` is the safety gate, mapped from the overlay's Safe mode toggle; `--yolo` is the trust-it path.
- Multi-turn chat works: each turn restarts Hermes (one prompt per process) with `--resume <sid>` so context carries. The session id is captured from Hermes' `result` event and threaded into the next turn via a shared locked variable.
- `--safe`, `--temp`, and `--detailed` semantics line up with the other agents. Approval messages from the overlay are answered with `kind: approve_unsupported` so the overlay can hide it.
- Stderr from Hermes is filtered (`↻ Resumed session`, `↪ restored workspace`, `session_id:` lines are dropped) and the rest is surfaced as `kind: stderr` for logging.
- `--serve` exit semantics: `kind: exit` is emitted only when the overlay closes stdin, not after every turn.

## 1.2.0-hermes.1

Fork by `Ex8-ca`. Adds [Hermes Agent](https://hermes-agent.nousresearch.com) as an overlay backend.

- New provider `hermes` (alias `hermes-agent`) in both the Python helper (`ask.py`) and the QML agent map (`AskModel.js`); appears in the logo menu when `hermes` is on `PATH`
- One-shot ask: `hermes chat -Q --oneshot --format text` with `--safe-mode` (overlay default) or `--yolo`; reasoning forced to `low` for snappy overlays (matches Claude's `QUICK_EFFORT`)
- Session continuity: follow-ups reuse the same Hermes session via `--resume <sid>`, just like Claude / Codex / OpenCode
- `--temp` and `--safe` semantics line up with the other agents; `--detailed` works (Hermes handles longer answers natively)
- `--serve` (warm process, streaming deltas) is **not** wired for Hermes yet — Hermes' `--format stream-json` shape is different and the parser lives in `parse_events()`; v2 will add it
- `manifest.json` plugin id renamed to `io.github.ex8-ca.omasearch` to avoid colliding with the upstream package

Install over the existing plugin (Omarchy keeps the Super+Q bind):

```shell
omarchy plugin remove io.github.5h3rd1l.omasearch --yes
omarchy plugin add https://github.com/Ex8-ca/omaSearch.git --enable --yes
omarchy default agent hermes   # or pick it in the overlay's logo menu
```

## 1.1.0

- Security: omaSearch's folders (`~/.local/state/omasearch`, `~/.cache/omasearch`) are created and made private without ever following a symlink (opened with `O_NOFOLLOW`, then `fchmod`), and folders you don't own are refused
- Security: settings and the model cache are written through a fresh temp file with a random name (`mkstemp`) and renamed into place, so a planted link can't redirect a write; pasted images get `mktemp` names
- Ctrl+I opens a temporary chat: nothing is saved (no recent-chats entry, title or question history), and Claude / Codex keep no session on disk. It has its own violet look, like a private window
- Shift+Enter adds a new line; the input grows with your text (up to about six lines, then scrolls), and line breaks are kept when you send

## 1.0.0 — omaSearch

First release as omaSearch, based on omAsk 1.0.5 by Ali Shabdar.

- Chat in the overlay: follow-ups continue the same session; Claude answers stream from a warm process
- Markdown answers with syntax-coloured code, a Copy button and clickable links
- Tool steps you can expand to see each command and its output
- Safe mode (on by default): Allow / Always in this chat / Deny before commands run
- Recent chats under the input, including older Claude chats; filter, keyboard, pins, clear; short titles for new chats
- Paste images (Ctrl+V) or pick one from clipboard history for Claude
- Select to copy; click to copy a message or code block
- Short / Detailed answers, model and agent menus, Ctrl+Up / Down question history
- Liquid-glass look, toast confirmations, Ctrl+H help

## omAsk (before the rename)

### 1.0.5

Open in browser was leaving Chromium on about:blank.

- `wl-copy` / `wtype` / `hyprctl` inherit `WAYLAND_DISPLAY` (and the Hyprland session vars)
- The continuation packet still never appears on argv

### 1.0.4

Fix blank Open in browser composers.

- Keep the continuation packet off Chromium argv (`about:blank`, then paste the encoded URL)
- Flush overlay stdin before closing it so the JSON handoff is not truncated
- If Chromium reuses a window, still paste into it

### 1.0.3

Open in browser continues from the overlay exchange.

- Seeds a continuation packet (`I asked` / `You answered (desktop overlay)`) as `?q=`
- Confirms Send so the web chat is already running
- Prompt and overlay answer travel on stdin JSON, not argv

### 1.0.2

Keep user prompts off process command lines.

- Overlay writes the question to `ask.py --ask` on stdin (`Process.write`)
- `wl-copy` and `open_chat.py` take clipboard/prompt bytes on stdin
- Agent CLIs receive the question via stdin or `--prompt-file /dev/stdin`, not `-p` / positional argv
- Copilot and OpenCode overlay answers fail closed to Open in browser (those CLIs require the prompt as an argument)
- Browser handoff launches the chat host without `?q=` on Chromium argv, then pastes the seeded URL
- Notifications no longer include the question text

### 1.0.1

Marketplace-ready identity.

- Plugin id `io.github.shabdar.omask` (the `omarchy.*` namespace is reserved for built-ins)
- Display name `omAsk`
- Docs: install path, IPC, update/remove commands, external dependencies
- Install still does not write Hyprland binds or other user config
- Migration: `omarchy plugin remove omask --yes`, re-add from git, then update the Super+Q bind to toggle `io.github.shabdar.omask`

### 1.0.0

First public release.

- Plugin id `omask` (the `omarchy.*` namespace is reserved for built-ins)
- Centered overlay for a one-shot question to `omarchy default agent`
- Short on-screen answer via that agent's CLI (Grok, Claude, Gemini, Copilot, Codex, OpenCode, Crush, Pi, Oh My Pi)
- Open in browser starts the agent's web chat with `?q=` in a new Chromium window; grok.com Send is confirmed
