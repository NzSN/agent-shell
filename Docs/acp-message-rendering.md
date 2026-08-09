# ACP message rendering: architecture and workflow

Date: 2026-08-09
Status: reference documentation of the current implementation.

## Key files

| File | Role |
|---|---|
| `agent-shell.el` | Core: ACP client setup, subscription wiring, `session/update` dispatch, fragment render orchestration, tool-call/diff/plan formatting, turn lifecycle |
| `agent-shell-ui.el` | Fragment engine: text-property-tagged blocks with labels/body, insert/append/replace/delete, fold groups, marker caching |
| `agent-shell-markdown.el` | In-place markdown renderer (streaming-friendly, watermark-based incremental rendering) |
| `agent-shell-diff.el` | Standalone diff viewer buffer with accept/reject actions (used by permission dialogs) |
| `agent-shell-viewport.el` | Optional mirror buffer (compose/view modes); every fragment update is mirrored here when in view mode |
| External: `acp` package | JSON-RPC transport: spawns agent subprocess, parses messages, invokes subscribed handlers |
| External: `shell-maker` package | comint shell framework: process filter, prompt markers, busy state, `shell-maker-finish-output` |
| `agent-shell-*.el` (per-agent) | Agent configs; some provide `:notification-adapter` hooks to rewrite notifications before dispatch |

## Data flow — receipt to display

**Step 1 — Transport (external `acp` package).** The agent subprocess is
spawned via `acp-make-client` (`agent-shell--make-acp-client`,
agent-shell.el:377). The `acp` package owns the process filter and JSON-RPC
framing; it parses incoming messages and routes them to subscribed callbacks.

**Step 2 — Subscription wiring.** `agent-shell--handle` (agent-shell.el:2028)
is a lazy-init state machine (client → subscriptions → handshake → auth →
session → prompt). Subscriptions are installed by
`agent-shell--initialize-subscriptions` (agent-shell.el:5945) →
`agent-shell--subscribe-to-client-events` (agent-shell.el:7163), which
registers three handlers:

- `acp-subscribe-to-notifications` → `agent-shell--on-notification` (line
  7178), first passed through `agent-shell--adapt-notification`
  (agent-shell.el:2241) so agent configs can rewrite notifications
- `acp-subscribe-to-requests` → `agent-shell--on-request` (agent-shell.el:3096)
- `acp-subscribe-to-errors` → renders a "Notices" fragment

**Step 3 — Dispatch.** `agent-shell--on-notification` (agent-shell.el:2620)
matches `method == "session/update"` then dispatches on
`(params update sessionUpdate)` via a large `cond` (see content-type dispatch
below).

**Step 4 — Render orchestration.** Each branch calls
`agent-shell--update-fragment` (agent-shell.el:4623),
`agent-shell--update-text` (agent-shell.el:4839), or
`agent-shell--delete-fragment` (agent-shell.el:4573). These:

1. Build a fragment model with `agent-shell-ui-make-fragment-model`
   (agent-shell-ui.el:59): `:namespace-id` (defaults to `state
   :request-count`, the shell-maker request id), `:block-id`, labels, body,
   group membership.
2. Call `agent-shell-ui-update-fragment` (agent-shell-ui.el:99) in both the
   viewport buffer (if in `agent-shell-viewport-view-mode`) and the shell
   buffer.
3. Narrow to the returned body range and run `agent-shell--render-markdown`
   (agent-shell.el:316), which dispatches to
   `agent-shell-markdown-render-function` (default
   `agent-shell-markdown-replace-markup`, agent-shell-markdown.el:297); right
   labels get markdown too (no images).
4. Mark inserted ranges with `field output` text property so comint prompt
   navigation works.

**Step 5 — Fragment engine.** `agent-shell-ui-update-fragment`
(agent-shell-ui.el:99) locates an existing block by searching backward from
`point-max` for the `agent-shell-ui-state` text property matching the
qualified-id `"<namespace>-<block-id>"`. Then one of:

- **Existing block**: per-section update — `agent-shell-ui--replace-label`
  (ui.el:473), `agent-shell-ui--append-body` (ui.el:393) for streaming
  appends, `agent-shell-ui--replace-body` (ui.el:440) for replacement. If a
  body arrives on a labels-only block, it deletes and regenerates the
  fragment.
- **New group member**: materializes group header via
  `agent-shell-ui--insert-group-header` (ui.el:756), inserts at
  `agent-shell-ui--group-insertion-point`.
- **New block**: `agent-shell-ui--insert-fragment` (ui.el:839) at `point-max`.

**Step 6 — Markdown rendering (incremental).**
`agent-shell-markdown-replace-markup` (agent-shell-markdown.el:297):

- Reads a per-body **watermark** (`agent-shell-markdown-watermark` text
  property on first char), narrows to watermark→point-max so only the
  streamed tail is reprocessed.
- **Open-fence deferral**: while a code fence is unclosed (tracked via
  `agent-shell-markdown-open-fence` property), all rendering is deferred
  until the closing fence streams in (O(N+K) instead of O(N×K)).
- Runs styling passes: escape encoding, italic/bold/strikethrough loop,
  headers, inline code, links, images (`--replace-images`, downloads remote
  images to cache dir), dividers, blockquotes, lists, source blocks (with
  major-mode font-lock), tables; mirrors `face`→`font-lock-face`; pads
  rendered blocks; updates watermark; releases markers.

## Content-type dispatch

Two dispatch levels:

**Level 1 — `sessionUpdate` kind** (cond in `agent-shell--on-notification`,
agent-shell.el:2637–3094):

| sessionUpdate | Line | Rendering |
|---|---|---|
| `agent_message_chunk` | 2644 | Markdown body; block-id `<messageId\|group-count>-agent_message_chunk`, `:append t`, `:create-new` on message boundary (messageId or turn heuristic); images enabled |
| `tool_call` | 2693 | Saves to `state :tool-calls` via `agent-shell--save-tool-call` (3889); labels from `agent-shell-make-tool-call-label` (4181); block-id = toolCallId; nested under activity group; optional `-plan` fragment |
| `agent_thought_chunk` | 2745 | "Thinking" fragment, shares activity group with tool calls |
| `user_message_chunk` | 2802 | Replayed prompt via `agent-shell--update-text` (only during session/load or session/push); live prompts are suppressed no-ops |
| `plan` | 2864 | "Plan" fragment, body from `agent-shell--format-plan` (4225, aligned status+content table) |
| `tool_call_update` | 2873 | Merges status/content/diffs into `:tool-calls`; body = content blocks + fenced command block + diff text; **incremental**: `agent-shell--tool-call-body-update` (3819) sends only the grown suffix when possible; deletes stale `permission-<id>` fragment on completion |
| `available_commands_update` | 3024 | Bootstrapping fragment (slash commands) |
| `current_mode_update` | 3032 | Header/mode-line only |
| `session_info_update` | 3043 | Session title only |
| `config_option_update` | 3049 | Header/mode-line only |
| `usage_update` | 3059 | `agent-shell--update-usage-from-notification`, header |
| `session_push_end` | 3068 | Experimental (`agent-shell-experimental.el`) |
| pending-restore | 2642 | During `session/load`, notifications are buffered by `agent-shell--append-restore-notification` (6722) grouped by turn, then replayed through the same dispatch by `agent-shell--render-pending-restore` (6775) |

**Level 2 — ContentBlock type** within a chunk:
`agent-shell--content-block-to-markdown` (agent-shell.el:7387) — `pcase` on
`type`: `text` → raw text; `image` → `![name](uri-or-cache-file)`;
`resource_link` → markdown link; `audio` → decoded cache file link;
`resource` → blockquote (text) or binary link (blob); unknown →
`[unsupported content: TYPE]`.

**Requests** (`agent-shell--on-request`, agent-shell.el:3096):
`session/request_permission` → permission fragment
`permission-<toolCallId>` with option buttons
(`agent-shell--make-tool-call-permission-text`, 8264) + optional diff viewer
(`agent-shell--make-diff-viewing-function`, 8471 → `agent-shell-diff`,
agent-shell-diff.el:101); `fs/read_text_file` / `fs/write_text_file` → file
I/O with `acp-send-response`.

**Diffs**: extracted by `agent-shell--make-diff-infos` (agent-shell.el:3471)
from `content` items of type `"diff"` or agent-specific `rawInput` shapes
(Copilot `old_str`/`new_str`, goose `before`/`after`, unified-diff parse);
rendered inline in tool-call body as propertized unified diff text tagged
`agent-shell-markdown-frozen` (`agent-shell--format-diffs-as-text`, 3652,
shells out to `diff -U3`), or in the standalone `agent-shell-diff` viewer
with accept/reject.

## Incremental-rendering mechanisms (deep dive)

The design goal: a streamed chunk should cost O(chunk), not O(accumulated
content). Five cooperating mechanisms achieve that: per-section fragment
edits, cached range markers, suffix diffing, a markdown watermark, and
open-fence deferral.

### 1. Fragment identity and lookup

Each block's first char carries an `agent-shell-ui-state` alist with a
`:qualified-id` of `"<namespace>-<block-id>"` (namespace = request-count,
`"bootstrapping"`, or `"out-of-turn"`). Lookup is a
`text-property-search-backward` from `point-max` (agent-shell-ui.el:137) —
recent fragments are found quickly, which is the common streaming case.

### 2. Per-section updates (only what changed)

`agent-shell-ui-update-fragment` (agent-shell-ui.el:99) applies edits per
section to an existing block:

- **Labels**: `agent-shell-ui--replace-label` (ui.el:473) rewrites only the
  named label region. A `string=` guard (ui.el:507) skips the rewrite
  entirely when the text is unchanged — tool-call updates resend identical
  status/title labels on every chunk, so this avoids a
  delete+insert+re-propertize per chunk.
- **Body append**: `agent-shell-ui--append-body` (ui.el:393) inserts at
  body-end only; existing chars keep their faces and `frozen` tags.
- **Body replace**: `agent-shell-ui--replace-body` (ui.el:440) touches only
  the body chars; labels, indicator, and padding stay put.

Ordering subtlety (ui.el:189–212): when a chunk changes both labels and
body, the body range is re-derived *after* label replacement, because a
label length change shifts the body boundary. The re-derivation reads the
cached range markers (updated by the label replacement), so it stays O(1).

Visibility is derived from the body's current state, not the caller's
`:collapsed` (ui.el:403–406), because label-less fragments don't follow the
stored collapse flag.

### 3. Cached range markers

`agent-shell-ui--compute-range-markers` (ui.el:581) does the
interval-walking property searches **once** per fragment and stores
block/body/label start/end markers on the state's `:range-markers`
(ui.el:615). Two critical details:

- **End markers use insertion-type `t`** (ui.el:607–611): chunk appends,
  the inline-code renderer's delete-and-reinsert of a trailing span, and
  framing-gap inserts all land exactly at a section end and must extend it.
  This makes every per-chunk range read O(1) instead of O(intervals) — the
  fix that removed the O(n²) interval-tree walk documented in
  `large-buffer-performance.md`.
- **Release on delete/regenerate** (ui.el:664, called from ui.el:551):
  markers are pointed nowhere so they don't accumulate in the buffer's
  marker list.

Old fragments (state alist lacking the key) are upgraded in place via
`nconc` so the text property keeps referencing the same list (ui.el:628–633).

### 4. Streaming body append details

`agent-shell-ui--append-body` (ui.el:393):

- **Trailing-whitespace invisibility**: a visible body hides its trailing
  whitespace tail via the `invisible` property (`rear-nonsticky` so new
  chars don't inherit it, ui.el:380). On append, only that tail is cleared
  and re-derived (ui.el:423–429) — never the whole body, which would be
  O(body) per chunk. For a hidden (collapsed) body the whole-body
  `invisible` stays untouched.
- New chars are indented to nest under any group, then re-tagged with the
  section/state properties so block identity is contiguous.
- `agent-shell-ui--body-invisible-p` (ui.el:368) reads collapse state from
  the **first body char's** `invisible` property — reliable because the
  trailing-whitespace handler never marks the first char.

### 5. Tool-call suffix diffing

Tool output streams as repeated `tool_call_update` notifications carrying
the *full* accumulated body. `agent-shell--tool-call-body-update`
(agent-shell.el:3819) diffs against `:last-sent-body`:

| Situation | Action |
|---|---|
| Body identical to last sent | Send `nil` body — update skipped entirely |
| Body grew (prefix match) and fragment still in buffer | Send only the grown suffix with `:append t` |
| Otherwise (rewritten, or fragment truncated away) | Send full body for replacement |

The fragment-exists check (agent-shell.el:3827–3838) does a property search
for the qualified-id, because buffer truncation (`agent-shell-buffer-maximum-size`)
may have deleted the fragment — appending a suffix to a missing fragment
would lose the earlier output. `:last-sent-body` is recorded by
`agent-shell--record-tool-call-body` (agent-shell.el:3848). Measured: 6×
faster on expanded tool bodies.

### 6. Markdown watermark

`agent-shell-markdown-replace-markup` (agent-shell-markdown.el:297) narrows
to `[watermark, point-max]` for all its styling passes (markdown.el:396), so
a chunk only re-scans the unresolved tail.

- **Storage**: the watermark is a text property on the region's first char
  (`agent-shell-markdown--watermark-start`, markdown.el:3387) — it travels
  with the text through `agent-shell-markdown-convert` and needs no
  buffer-local state. Missing/out-of-range → `point-min` (conservative full
  scan).
- **Frontier computation** (`--update-watermark`, markdown.el:3485):
  `min(last-line-start, open-fence-start, extending-table-start,
  external-candidates...)`.
  - `last-line-start`: single-line patterns (bold, italic, strike, header,
    link, image, inline code, divider) can't span a newline, so backing off
    to the start of the last line covers delimiters split across chunks.
  - `extending-table-start` (markdown.el:3408): walks backward through
    pipe-row lines so a table that is still accumulating rows (rendered or
    not yet rendered) gets re-scanned — otherwise the watermark would
    advance past streamed rows one chunk at a time and the table would
    never render/fold.
  - `external-candidates`: external renderers in
    `agent-shell-markdown-render-functions` can hold the watermark behind
    their own open delimiters (e.g. unclosed `$$`).
- `:force t` drops watermark/open-fence/construct-activity properties for a
  full re-render (used by `agent-shell-markdown-convert` and tests).

### 7. Open-fence deferral

An unclosed fenced code block is raw code that no pass can style, so
re-scanning it per chunk is pure waste (was O(N×K) for an N-line block over
K chunks; see `large-buffer-performance.md`).

- When a fence is open, `--update-watermark` stamps
  `agent-shell-markdown-open-fence` = `(backtick-count . scan-position)` on
  the first char (markdown.el:3542–3550).
- On the next call, if the property is present and
  `--fence-closer-since-p` (markdown.el:3466) finds no closing fence in the
  newly streamed tail, **all rendering is deferred**: the only work is the
  tail scan plus re-stamping the scan position (markdown.el:376–384).
  Result: O(N+K) total.
- Split closers are handled by scanning from the *beginning of the line*
  containing the last scan position (a fence line begun by one chunk and
  completed by the next still matches).
- While deferred, the raw region gets `fontified t` and a `yank-handler`
  that strips properties, so font-lock and yanks behave.
- The chunk carrying the closer gets the full render, which also covers any
  prose that followed the fence in the same chunk.

### 8. Lazy render on expand

Rendering is skipped for collapsed bodies: `agent-shell--update-fragment`
checks `agent-shell-ui--body-invisible-p` before calling
`agent-shell--render-markdown` (agent-shell.el:4706 and 4803). On expand,
`agent-shell-ui--toggle-leaf-fragment-at-point` (ui.el:1104) flips the
`invisible` property, then runs
`agent-shell-ui-post-expand-fragment-at-point-hook` narrowed to the body
(ui.el:1148–1151); `agent-shell--render-markdown` is hooked in at
agent-shell.el:4445 (and agent-shell-viewport.el:1525 for the viewport).
So a long-collapsed tool output costs nothing until the user opens it.

### 9. Out-of-turn insertion above a live prompt

Notifications arriving after `end_turn` must land above the active prompt
(`:above-last-prompt`, agent-shell.el:4735–4837):

1. Verify `comint-last-prompt` is a live input prompt
   (`agent-shell--live-input-prompt-p`).
2. Flip the prompt-start marker's insertion type to `t` so it advances past
   the inserted text instead of being stranded inside it.
3. Narrow to `(point-min, prompt-start)` so the fragment engine's
   `point-max` insertions land in the right place.
4. Afterwards restore the insertion type, widen, and restore point / mark /
   window-start — with a special case: if the user was at absolute eob
   (typing in the input area), put them back at the true `point-max`, not
   the narrowed one (agent-shell.el:4828–4829). Unsubmitted typed input is
   pushed down with the prompt, never clobbered.

### 10. Read-only, undo, and stickiness discipline

- All programmatic updates run under `inhibit-read-only` with
  `buffer-undo-list` bound to `t` (`:no-undo`) — streaming output must not
  flood the user's undo history.
- Fragment text is `read-only t` with `front-sticky (read-only)` so typed
  input at a boundary can't inherit read-only.
- Inserted ranges get `field output` so comint prompt navigation
  (`agent-shell-next-item` / `previous-item`) skips them
  (agent-shell.el:4785–4791).
- Turn end: `agent-shell--truncate-buffer` (agent-shell.el:3759) deletes
  oldest content at whole-fragment/group boundaries (never splitting one),
  releasing their cached markers; newest fragment and live prompt always
  kept; full history remains in the transcript file.

## Groups and turn lifecycle

- **Activity groups**: tool calls and thoughts share a group id from
  `agent-shell--activity-group-id` (agent-shell.el:2298); the header is
  auto-created on first member (`agent-shell-ui--insert-group-header`,
  ui.el:756) and relabeled with status counts by
  `agent-shell--refresh-activity-group-header` (agent-shell.el:2607). An
  update to an existing member never re-routes groups (ui.el:144–170) —
  otherwise a caller whose group-id advanced mid-turn would spawn an empty
  group.
- **Turn end**: the `session/prompt` response is handled in
  `agent-shell--send-command` (agent-shell.el:7556, on-success at 7605):
  clears `:tool-calls`, renders usage/stop-reason fragments, calls
  `shell-maker-finish-output`, truncates the buffer, emits `turn-complete`,
  and processes the prompt queue.

## Other rendering mechanisms

Beyond the default in-place markdown renderer, the codebase has several
alternative/auxiliary rendering paths:

- **Overlay-based renderer (deprecated)** — `agent-shell--markdown-overlays-put`
  (agent-shell.el:267) wraps the external `markdown-overlays` package: it only
  adds overlays instead of rewriting buffer text. Selectable via the
  `agent-shell-markdown-render-function` defcustom (agent-shell.el:286);
  default is `agent-shell-markdown-replace-markup`. Kept for backwards
  compatibility, slated for removal (see blog-post.org — the in-place
  renderer was PR #597, "biggest internal change in some time").
- **External renderer hook** — `agent-shell-markdown-render-functions`
  (agent-shell-markdown.el:227): functions run before the built-in passes,
  receive the render context (source blocks + inline code ranges), claim
  regions by tagging them `agent-shell-markdown-frozen` (built-in passes
  then skip them), and can hold the watermark back via returned
  `:watermark` values (e.g. a LaTeX-math package behind an unclosed `$$`).
  Suppressed on single-line label spans (`:external-renderers nil`,
  agent-shell.el:316–344).
- **Plain-text path** — `agent-shell--update-text` (agent-shell.el:4839) →
  `agent-shell-ui-update-text`: no markdown at all; used for replayed user
  prompts and simple notices.
- **Diff viewer** — `agent-shell-diff` (agent-shell-diff.el:101): a
  separate `diff-mode` buffer with accept/reject actions, offered from
  permission dialogs.
- **Viewport mirror** — `agent-shell-viewport.el`: when the viewport buffer
  is in `agent-shell-viewport-view-mode`, every fragment update is applied
  twice — once in the viewport, once in the shell buffer
  (agent-shell.el:4666–4720).
- **Image rendering** — `agent-shell-markdown--replace-images`: remote
  image URLs are downloaded into a cache dir (`agent-shell-cache-dir
  "content"`) and displayed inline; disabled on labels.

## When each renderer activates

| Renderer / path | Activated when |
|---|---|
| In-place markdown (`agent-shell-markdown-replace-markup`) | Every `agent-shell--update-fragment` call that touches a body (message chunks, tool updates, plans, thoughts…) or a right label — i.e. effectively every `session/update` that carries displayable content. Skipped while the body is collapsed; re-runs on expand via `agent-shell-ui-post-expand-fragment-at-point-hook` (ui.el:1151). |
| Overlay renderer (deprecated) | Only if the user sets `agent-shell-markdown-render-function` to `agent-shell--markdown-overlays-put`; never active by default. |
| External renderers (`agent-shell-markdown-render-functions`) | Inside every in-place render of a **body**, when the hook is non-nil (e.g. a LaTeX package). Explicitly suppressed on label spans (`:external-renderers nil`, agent-shell.el:4815). |
| Open-fence deferral | Whenever the watermark pass finds an unclosed fence at `point-max`; stays active per chunk until the closing fence streams in. |
| Watermark narrowing | Always, on every in-place render (degenerates to whole region on first render or after a body replace). |
| Plain-text path (`agent-shell--update-text`) | Replayed `user_message_chunk`s during `session/load` / `session/push` (live prompts are suppressed no-ops). |
| Diff viewer (`agent-shell-diff`) | Only on user action: pressing the "view diff" button in a permission fragment (`session/request_permission` with diffs, agent-shell.el:8471). Inline diff text in tool bodies renders unconditionally as part of the body. |
| Viewport mirror | Every fragment/text update, but only when a viewport buffer exists and is in `agent-shell-viewport-view-mode` (agent-shell.el:4666–4671). |
| Image rendering | During body renders (`:render-images t`; labels pass nil). Remote URLs only fetched because agent-shell always passes its cache dir (agent-shell.el:344); incomplete `![` markup is held back until `:complete` (force/final render). |
| Lazy expand render | Once per fragment, the first time a collapsed body is expanded. |
| Full re-render (`:force t`) | `agent-shell-markdown-convert` (static strings, markdown.el:294) and tests. |
| Buffer truncation | Once per turn end (and on error), not per chunk. |

## Residual O(accumulated content) paths

The incremental machinery covers the hot streaming paths, but some paths
are still linear in accumulated content per call (all with small constants
or bounded scope):

| Path | When executed | Cost | Why it's acceptable |
|---|---|---|---|
| Deprecated overlay renderer (`agent-shell--markdown-overlays-put`) | Every fragment render, but only if the user customized `agent-shell-markdown-render-function` to it | Whole-body re-scan + re-overlay per chunk → O(n²) per message | Deprecated, off by default; the in-place renderer exists precisely because of this |
| `tool_call_update` body construction (agent-shell.el:2928–2934) | Every `tool_call_update` notification — i.e. each time the agent reports output progress (frequent for streaming terminal output) | `mapconcat` over all content items rebuilds the full body string → O(accumulated) per update | C-level string ops; the buffer-side cost is avoided by suffix diffing |
| Suffix-diff compares (agent-shell.el:3840–3843) | Every `tool_call_update` that reaches the body stage — the `equal` check is what detects a skippable resend | `equal` / `string-prefix-p` against `:last-sent-body` are O(accumulated) per update | memcmp-speed; `string-prefix-p` only scans the old length |
| Tool-call body **rewrites** (non-prefix changes) | A `tool_call_update` whose new body isn't a pure suffix extension: agent rewrote/truncated earlier output (spinners, progress bars, agent-side truncation), or the fragment was removed by buffer truncation | `replace-body` deletes the body — including the char carrying the watermark — so the next render is a full O(body) re-render | Agents almost always append; prefix growth takes the O(suffix) path |
| Extending tables | Each `agent_message_chunk` that lands while the body tail is table rows (raw pipe-row streak, or an already-rendered table at the tail) — i.e. for the duration of table streaming | Watermark backs off to the table start, so all passes re-scan the accumulated table per chunk → O(table²) worst case; `--extending-table-start` itself walks O(rows) per chunk | Tables are small and bounded; required so new rows fold into the rendered table |
| Activity group header relabel (agent-shell.el:2607 → ui.el:137) | Every `tool_call` (agent-shell.el:2734), `agent_thought_chunk` (2800), and `tool_call_update` (3018) | Header sits *before* its members, so the backward property search from `point-max` is O(members) per call → O(members²) per turn; the search runs even when the `string=` guard then skips the rewrite | Members per turn are few |
| Label replacement (ui.el:481–488) | Any update carrying a label for an existing fragment (tool status/title changes, plan updates) | Own backward property search per label — O(distance from end); runs before the unchanged-text guard | Latest fragments are near the end |
| Buffer truncation (agent-shell.el:3759) | Once per `session/prompt` response (turn end) and in the error handler | O(buffer) scan | Not per chunk |

## Design: eliminating the residual O(accumulated) paths

Goal: O(chunk) **allocation, buffer, and render work** per notification.
Note the floor: every notification re-sends full content, so JSON parsing
is already O(accumulated) — comparisons of parsed strings (`equal`,
`string-prefix-p`) are memcmp-class, allocate nothing, and are *below* the
parse cost already paid. What must go is O(accumulated) **allocation**
(rebuilding full body strings), **rendering**, and **property searches**.

Two shared mechanisms serve multiple paths:

### S1. Fragment-location cache (fixes: group header relabel, label search)

Root cause: every fragment lookup is a `text-property-search-backward` from
`point-max` — O(fragments after target). The group header sits *before* its
members, so it pays O(members) on every `tool_call` / `tool_call_update` /
`agent_thought_chunk`.

Design:

- Keep a per-buffer hash table `qualified-id → block-start marker`,
  populated by the single insertion writer (`agent-shell-ui--insert-fragment`)
  and by `--insert-group-header`. Store it on the fragment machinery's side
  (e.g. buffer-local `agent-shell-ui--location-cache`).
- Lookup: `gethash` → validate `(equal (map-elt (get-text-property marker
  'agent-shell-ui-state) :qualified-id) qualified-id)` → O(1). On mismatch
  (fragment regenerated, truncated, or marker stale) fall back to the
  current property search and repopulate. Lazy validation means no hooking
  of delete/truncate paths is required for correctness; deletes may still
  proactively `remhash` to keep the table small.
- Truncation/regeneration: markers point nowhere or at foreign text →
  validation fails → fallback. Regeneration re-inserts → cache refreshed.
- Viewport vs shell buffer: cache is buffer-local, so the two never
  interfere.
- Also cache `:last-group-label` per group in shell state; skip
  `--refresh-activity-group-header` entirely when the recomputed label is
  `equal` — moves the no-op guard from after the search (current
  `string=` guard in `--replace-label`) to before it.

### S2. Diff-before-construct with prefix-preserving replace (fixes: body mapconcat, rewrite re-render)

Root cause: `tool_call_update` rebuilds the *entire* body string via
`mapconcat` per notification (O(accumulated) allocation + GC), and a
non-append change triggers `replace-body`, which deletes the watermark
anchor and forces a full O(body) re-render.

Design — extend `agent-shell--tool-call-body-update` into a structural diff:

1. The previous update's parsed `content` is already stored in
   `state :tool-calls`. Walk old/new item lists with `equal` (memcmp, no
   allocation) to find the first differing item index.
2. All items before it are unchanged → their rendered text is unchanged and
   *already in the buffer*. Construct markdown only for items from the
   first difference onward → allocation is O(changed suffix).
3. Compute the byte offset of that item's start within the last-sent body
   (store per-item rendered lengths — a small fixnum list, updated per
   update, O(items)).
4. Return `(:replace-from OFFSET . NEW-TAIL)` instead of a full body:
   - **Buffer side**: delete from `body-start + OFFSET` (derived from the
     cached `:body-start` marker, O(1)), insert the new tail. Never touch
     the unchanged prefix — its faces, frozen ranges, and images survive.
   - **Render side**: re-stamp the markdown watermark at the line start of
     the replacement point before rendering, so the render pass covers only
     the tail. O(chunk) render for rewrites, not just appends.
5. The identical-resend check stays `equal` on the content structure
   (memcmp, no allocation) — same complexity class as the JSON parse
   already paid, so it is not the bottleneck; the win is skipping
   construction and buffer work, which this preserves.
6. Spinner/progress-bar agents (rewrite last line only): first difference
   lands in the last item near its end → suffix is a line-sized tail →
   effectively O(chunk).

### P1. Remove the deprecated overlay renderer

No algorithmic fix — delete `agent-shell--markdown-overlays-put` and the
`markdown-overlays` dependency once the maintainer confirms the in-place
renderer has settled (it is already the default and the only tested path).
Until then it costs nothing by default.

### P2. Open-table deferral (fixes: O(table²) streaming)

Mirror the open-fence mechanism exactly:

- When the watermark pass ends with an unterminated pipe-row streak at
  `point-max`, stamp `agent-shell-markdown-open-table` =
  `(table-start . scan-position)` on the first char.
- Per chunk, scan only the new tail for a non-pipe line ("table closer") —
  O(chunk). While open, skip the table pass entirely (raw pipe text is
  already how a deferred table looks) and skip `--extending-table-start`
  (replaced by the O(1) property check).
- The chunk carrying the first non-pipe line (or turn end / force render)
  renders the whole table once → O(N+K) total instead of O(N×K).
- Trade-off needing maintainer buy-in: tables pop in when complete instead
  of growing row-by-row. This matches the fence-deferral precedent. Variant
  if live growth is wanted: render rows incrementally with widths frozen
  from the header, re-laying out only when a wider cell arrives (amortized
  near-O(table), more complex).

### P3. Memoize diff text (fixes: per-update `diff -U3` subprocess)

`agent-shell--format-diffs-as-text` shells out to `diff` on every
`tool_call_update` while diffs are present. Diffs are almost always set
once (at completion). Store `(:diff-input-hash . :rendered-diff-text)` on
the tool call; recompute only when `equal` on the diff inputs fails.
Steady state: zero subprocess spawns during streaming.

### P4. Truncation precheck

Gate the O(buffer) line walk behind an O(1) check: compare
`(buffer-size)` against a byte threshold derived from
`agent-shell-buffer-maximum-size` (lines ≈ bytes / conservative-bytes-per-line,
or simply skip when `(buffer-size)` is under a generous floor). Exact walk
only when truncating — once per turn at most, O(deleted).

### Validation plan

- **Golden equivalence**: replay recorded notification streams (message
  chunks, tool streams with rewrites, tables, thoughts) through old and new
  code; assert final buffer text + key text properties identical.
- **Benchmarks**: reuse the `large-buffer-performance` harness — add a
  "500 tool_call_updates with growing output" case and a "200-row streaming
  table" case; expect flat profiles (no `mapconcat`/render/search growth).
- **Marker-drift probe**: existing cached-vs-searched parity check extended
  to the location cache (assert cache hit + validation passes > 99%).
- **Unit tests**: replace-from offset mapping (including multi-item,
  first-item-change, and truncation-fallback cases), location-cache
  invalidation on regenerate/truncate, table deferral renders exactly once
  and matches eager rendering.

### Sequencing

1. S1 location cache — self-contained, kills two paths, no UX change.
2. P3 diff memoization — trivial, removes subprocess churn.
3. S2 diff-before-construct — biggest win, moderate risk; gate behind the
   golden-equivalence suite.
4. P4 truncation precheck — trivial.
5. P2 table deferral — UX-visible, needs maintainer sign-off.
6. P1 overlay removal — when the maintainer is ready.

## Implementation status (2026-08-09)

Implemented: S1, P3, S2 (partial), P4. Deferred: P2 (UX sign-off), P1
(maintainer timing), and S2's `:replace-from` refinement (see below).

- **S1**: buffer-local `qualified-id → block-start marker` hash table
  (`agent-shell-ui--fragment-locations`), validated against the state
  property on every read with lazy repopulation. All qualified-id lookups
  converted: `update-fragment`, `--replace-label`, `delete-fragment`,
  `--group-header-range`, `update-text`, `collapse-fragment-by-id`, and the
  body-update existence check. Block *ends* intentionally come from a fresh
  `--block-range`, not cached markers (see latent bug below). The group
  header relabel additionally skips the buffer update entirely when the
  recomputed label is unchanged and the header fragment exists
  (`:activity-group-labels` in state).
- **Latent bug found and fixed**: inserting a new group member lands
  exactly on the previous element's block-end position, advancing its
  cached (insertion-type `t`) end markers past the new member. A later body
  regeneration of the previous member then deleted the new member, and a
  body append landed after it. Reproduced on pre-change HEAD with
  `agent-shell-ui-group-member-insert-keeps-sibling-markers-test`; fixed by
  invalidating the previous element's range markers at insertion
  (`agent-shell-ui-update-fragment`, group-member branch).
- **P3**: `agent-shell--tool-call-diff-text` memoizes rendered diff text on
  the tool call's `:body-render` record, keyed by the diffs input — no more
  `diff(1)` subprocess per `tool_call_update`.
- **S2**: `agent-shell--tool-call-body-update` now diffs structurally
  against the stored render record (`:header`, raw `:items`, `:diff-text`)
  before constructing anything: identical resend → skip; pure growth
  (including growth *within* the final content item) → append only the
  suffix; anything else → full replace via `agent-shell--make-tool-call-body`
  (byte-equivalent to the old assembly, including the double blank line
  before diff text and whole-tail trimming). The full-body `mapconcat` per
  update is gone.
  - **Deviation from design**: the `:replace-from` prefix-preserving
    replace (for in-place rewrites such as spinners) was deferred — it
    needs watermark re-stamping at the cut point and has correctness risk
    when the cut lands inside a rendered multi-line construct. Rewrites
    still take the full-replace path, same as before this change.
- **P4**: `agent-shell--truncate-buffer` skips its scan when
  `(buffer-size) <= agent-shell-buffer-maximum-size` (every line occupies
  at least one byte), and now drops location-cache entries of deleted
  fragments.

Benchmarks (batch, this machine): updating the first of 2000 fragments
200× — 5.097s → 0.004s (S1). Tool-body streaming is memcmp/allocation
bound at realistic sizes; its win is removed allocation churn (no per-update
full-body string), removed `diff(1)` spawns, and O(1) existence checks.

Verification: 450 tests (449 pass, 1 pre-existing skip), including 10 new
tests covering suffix math edge cases (trailing-whitespace base, diff
appearing after output, diff displacement → replace), byte-level body
assembly, diff memoization, label-skip, and the cache. Byte-compile and
checkdoc clean (only pre-existing warnings).

## Quick reference — entry points

- Notification entry: `agent-shell--on-notification` — agent-shell.el:2620
- Request entry: `agent-shell--on-request` — agent-shell.el:3096
- Render orchestrator: `agent-shell--update-fragment` — agent-shell.el:4623
- Fragment engine: `agent-shell-ui-update-fragment` — agent-shell-ui.el:99
- Fragment inserter: `agent-shell-ui--insert-fragment` — agent-shell-ui.el:839
- Markdown renderer: `agent-shell-markdown-replace-markup` — agent-shell-markdown.el:297
- Content block dispatch: `agent-shell--content-block-to-markdown` — agent-shell.el:7387
- Prompt send / turn end: `agent-shell--send-command` — agent-shell.el:7556
