# agent-shell architecture

Date: 2026-08-13
Version analyzed: 0.67.1 (`agent-shell.el` ~10.3k lines; 60 source files total)

## Overview

`agent-shell` is a native Emacs shell for ACP ([Agent Client
Protocol](https://agentclientprotocol.com)) agents — Claude, Codex, Gemini CLI,
Cursor, opencode, Goose, and 13 more. It is a thin orchestration and rendering
layer on top of two external libraries:

- **`shell-maker`** (>=0.94.1) — comint shell scaffold: prompt, input history,
  output markers, major mode.
- **`acp.el`** (>=0.13.1) — JSON-RPC ACP wire protocol: client, requests,
  notifications, responses.

```
Providers (19 thin adapters)         anthropic, openai, google, cursor, kimi...
        |  make-config alist
        v
CORE: agent-shell.el (10.3k lines)   ACP state machine + inbound dispatch
        |  fragments
        v
UI: ui.el -> markdown.el             block widgets -> rendering engine
Viewport / chat-mode / completion    alternative interaction satellites
```

## Core: `agent-shell.el`

### State

All state lives in **one buffer-local alist**, `agent-shell--state` (:1129),
built by `agent-shell--make-state` (:1068). ~35 slots: `:agent-config`,
`:client` (the acp client), init flags (`:initialized`, `:authenticated`,
`:set-model`, `:set-session-mode`), `:session` (nested alist: `:id`,
`:config-options`, `:model-id`, `:modes`...), capability flags
(`:supports-session-list/load/resume/fork`), `:tool-calls` (map id -> tool-call
alist), `:active-requests`, `:event-subscriptions`, `:usage`, `:heartbeat`,
`:idle-timer`, `:last-entry-type` (rendering coalescing key). No
`cl-defstruct` anywhere — everything is `map-elt`/`map-put!`.

### Init state machine

The heart is `agent-shell--handle` (:2029), wired into shell-maker's
`:execute-command` (:1972). Every user submission (and shell creation)
re-enters it; flags in state determine the next step:

```
client? -> subscriptions? -> handshake(initialize) -> authenticate?
        -> session/new|load|resume|fork -> default model? -> default mode?
        -> send prompt
```

Key steps: `--initialize-client` (:6038, calls the config's `:client-maker`),
`--initialize-subscriptions` (:6061), `--initiate-handshake` (:6116),
`--authenticate` (:6182), `--initiate-session` (:6375), then
`--send-command` (:7672).

Process lifecycle: `agent-shell--start` (:4456) -> `shell-maker-start-v2`
(:4491); `kill-buffer-hook` -> `--clean-up` (:4081) -> `acp-shutdown`.

### Message flow

**Outbound** (client -> agent) — all funnels through
`agent-shell--send-request` (:6082), which tracks `:active-requests`:
- `agent-shell-submit` (:1174) -> shell-maker -> `--handle` ->
  `--send-command` (:7672): builds content blocks (text, `@file` mentions,
  images, embedded resources via `--build-content-blocks` :7328), sends
  `acp-make-session-prompt-request`.
- Interrupt: `agent-shell-interrupt` (:1935) cancels pending permissions and
  sends `session/cancel`.

**Inbound** (agent -> client) — three handlers registered once at :7279:
- Notifications: `--on-notification` (:2635), optional per-agent normalization
  via config `:notification-adapter`. Dispatches on `sessionUpdate`:
  `agent_message_chunk` (coalescing via `messageId` + `:last-entry-type`),
  `tool_call`, `agent_thought_chunk`, `plan`, `tool_call_update`,
  `available_commands_update`, `usage_update`... During `session/load` replay,
  notifications accumulate in `:pending-restore` and replay later (:6892).
- Requests: `--on-request` (:3101): `session/request_permission` (:3103),
  `fs/read_text_file` (:3229), `fs/write_text_file` (:3298),
  `session/push` (experimental, :3182). Unknown methods get JSON-RPC `-32601`
  so the agent never hangs.
- Errors -> rendered as "Notices" fragments (:7284).

Prompt response (:7721-7792): extracts usage, stops heartbeat,
`shell-maker-finish-output`, truncates buffer, emits `turn-complete`,
processes the prompt queue.

### Event bus

`agent-shell-subscribe-to` (:5813) / `--emit-event` (:5986) — per-buffer
pub/sub used for cross-module coordination (viewport, prompt-queue,
chat-mode) instead of direct calls. The authoritative event catalog is at
:5823-5867 (`init-*`, `session-*`, `tool-call-update`, `permission-*`,
`agent-message-chunk`, `turn-complete`, `idle`, `error`, `clean-up`...).

### Rendering funnel

All UI output flows through `agent-shell--update-fragment` (:4739) into
`agent-shell-ui-update-fragment`, keyed by `namespace-id` (request count) +
`block-id`, with collapsible activity groups.

### Entry points

Autoloaded: `agent-shell` (:1146, DWIM main command), `agent-shell-toggle`,
`agent-shell-new-shell`, `agent-shell-restart`, `agent-shell-fork`,
`agent-shell-resume-session`, `agent-shell-prompt-compose`,
`agent-shell-shell-buffer` (:7922), `agent-shell-make-agent-config` (:601).
Programmatic: `agent-shell-start` (:1547).

## Providers (19 files)

Every provider follows the same template (e.g. `agent-shell-kimi.el` is the
minimal exemplar, `agent-shell-anthropic.el` the full-featured one):

1. **Defcustoms**: `agent-shell-X-acp-command` (executable + args to spawn the
   ACP subprocess), `agent-shell-X-environment`, default model/mode ids.
2. **`X-make-config`** calling `agent-shell-make-agent-config` (core :601) —
   returns a plain alist with keys `:identifier`, `:client-maker`,
   `:needs-authentication`, `:authenticate-request-maker`,
   `:default-model-id`, `:session-meta`, `:mcp-servers`,
   `:notification-adapter`, `:icon-name`...
3. **`X-start-agent`** interactive command -> `agent-shell--dwim`.
4. **`make-client`** -> core's `agent-shell--make-acp-client` (:377), which
   wraps `acp-make-client` and applies `agent-shell-command-prefix` (:2168 —
   also the devcontainer hook point).

Authentication strategies: env-var injection at spawn (most common;
api-key values may be functions for auth-source lookups), ACP `authenticate`
request (`:needs-authentication` + `:authenticate-request-maker`), or none
(CLI's own login).

**Registration is static**, in three places in the core: eager `require`s
(:58-90), `agent-shell-default-agent-config-makers` (:656-685, the canonical
registry of maker-function symbols), and the
`agent-shell-preferred-agent-config` customize type (:752-777). Runtime
resolution: `agent-shell--resolved-agent-configs` (:721) ->
`agent-shell-select-config` (:1579, completing-read with icons).

Known quirk: :767 offers `le-chat` as the Mistral choice, but the provider
registers `:identifier 'mistral-vibe` — preferred-config lookup would miss.

## UI layer

Streaming is the central design constraint everywhere: text properties are the
in-buffer database, and marker caches keep per-chunk work O(chunk).
See `Docs/large-buffer-performance.md`.

### `agent-shell-markdown.el` (3.8k lines) — rendering engine

Streaming-friendly, in-place markdown renderer: markup chars deleted, faces
and original source stashed in text properties. Pipeline:
`agent-shell-markdown-replace-markup` (:297) — watermark narrowing,
open-fence deferral (`agent-shell-markdown-open-fence` property, :363),
then ~15 regexp passes (emphasis, headers, code, links, images, tables...).
Inverse: `agent-shell-markdown-reconstruct` (:2795) backs
`agent-shell-copy-as-markdown`. Extension hook:
`agent-shell-markdown-render-functions` (:227) for third-party renderers.

### `agent-shell-ui.el` (1.6k lines) — block widgets

Magit-section-like collapsible fragments with left/right labels + body,
two-level grouping, TAB navigation. Central API:
`agent-shell-ui-update-fragment` (:99) — upsert by qualified id
(`"<namespace>-<block-id>"`); `:append` does O(new-chunk) body updates.
State in `agent-shell-ui-state` text property with cached `:range-markers`
(:583-636) plus a buffer-local qualified-id -> marker hash table (:674).
`agent-shell-ui-mode` (:1559) adds cursor-sensor + isearch-in-collapsed
support. Clickable text: `agent-shell-ui-add-action-to-text` (:1494).

### `agent-shell-viewport.el` (1.5k lines) — alternative interaction

A paired buffer (`*shell* [viewport]`) with an Edit mode (compose prompts,
comint-ring history) and a View mode (browse history, quick replies
`y`/`1`-`9`, quote-reply, cycle model/mode). Enabled via
`agent-shell-prefer-viewport-interaction`. Buffer-locals marked
`permanent-local` survive the edit<->view mode switches.

### Satellites

- **`agent-shell-chat-mode.el`** — global minor mode relabeling the shell as
  chat (`Me` / agent-name boxed badges), implemented purely as overlays;
  driven by an event-bus subscription + coalescing timer.
- **`agent-shell-completion.el`** — CAPFs: `@` completes project files, `/`
  completes agent slash commands (from ACP `:available-commands`); works in
  shell buffers and minibuffers.
- **`agent-shell-diff.el`** — permission-diff viewer: read-only diff-mode
  buffer with accept/reject wired to ACP permission responses, RET jumps to
  the changed code by searching file content (hunk line numbers untrusted).

## Aux modules

| File | Purpose |
|---|---|
| `agent-shell-config.el` | ACP session `configOptions` protocol: camelCase->kebab normalization, state storage, accessors, legacy model/mode adapters, shell formatting. Not user-level config. |
| `agent-shell-prompt-queue.el` | FIFO for prompts submitted while the shell is busy |
| `agent-shell-usage.el` | Token-usage tracking, context-window indicator |
| `agent-shell-heartbeat.el` | ~10Hz timer object driving UI animation during turns |
| `agent-shell-project.el` | File listing / CWD via projectile or project.el |
| `agent-shell-worktree.el` | `agent-shell-new-worktree-shell` in a fresh git worktree |
| `agent-shell-work-buffer.el` | `with-work-buffer` compat macro (Emacs 31 pool vs temp buffer) |
| `agent-shell-devcontainer.el` | Host->container path mapping; pairs with `agent-shell-command-prefix` |
| `agent-shell-artist.el` | Minor mode for drawing ASCII art into a buffer |
| `agent-shell-active-message.el` | Minibuffer progress message show/hide |
| `agent-shell-list-edit.el` | Markdown list editing (RET continues, TAB indents) |
| `agent-shell-experimental.el` | Experimental ACP `session/push` |
| `agent-shell-styles.el` / `agent-shell-faces.el` | Status-label styles; deffaces |
| `agent-shell-github.el` | Actually a provider (Copilot CLI), same pattern |
| `agent-shell-mock-agent.el` | Mock ACP agent (Swift `mock-acp` binary) for testing |

## Testing

`tests/` — ERT. `agent-shell-fakes.el` provides fakes; recorded `.traffic`
files replay real ACP sessions (permission flows, grouping bugs). Per-module
test files cover ui, markdown, diff, completion, chat-mode, command-prefix,
devcontainer, and providers (anthropic, openai, cursor).

## Key takeaways

1. **State is one big alist**, buffer-local, mutated with `map-put!`; no structs.
2. **`agent-shell--handle` (:2029) is the heart** — a lazy, recursive init
   state machine triggered by shell-maker's execute-command.
3. **Three-way ACP inbound dispatch** registered once at :7279.
4. **All outbound ACP traffic** funnels through `agent-shell--send-request` (:6082).
5. **All rendering** funnels through `agent-shell--update-fragment` (:4739);
   streamed-chunk coalescing is driven by `:last-entry-type` + `messageId`.
6. **Cross-module coordination** goes through the event bus, not direct calls.
7. **Text properties are the database**; marker caches make streaming O(chunk).
