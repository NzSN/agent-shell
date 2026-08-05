# Large-buffer performance: investigation, fixes, review

Branch: `fix-large-buffer-performance` (on `nzsn` fork)
Date: 2026-08-05
Status: 6 commits, all 439 tests passing, byte-compile + checkdoc clean.

## Problem

Long agent-shell sessions hang Emacs: the shell buffer grows without bound,
and per-chunk work degrades quadratically as messages stream in.

## Root causes (found by profiling, not guessing)

1. **Interval-tree walks dominate.** Emacs's
   `next/previous-single-property-change` must step over *every* property
   interval boundary in range. Rendered markup creates ~6 intervals per chunk
   (bold/italic/code/link spans), so a growing message accumulates thousands.
   Per chunk, agent-shell ran ~7 fragment-range computations
   (`agent-shell-ui--nearest-range-matching-property` for the update path,
   3× `block-range`, plus return-value construction) and 3 full-body
   `pad-rendered-blocks` walks — each O(total intervals), i.e. O(n²) per
   message. Measured: per-call `nearest-range` 100µs → 746µs as the buffer
   grew 4×; ~46% of stream time in that one function, ~22% in pad walks.

2. **O(block²) markdown re-rendering while streaming.** The watermark backed
   off to the start of an open fenced code block, so every chunk re-ran ~15
   regexp passes over the whole accumulated block. Open blocks are rendered
   as raw text anyway, so this was pure waste.

3. **Unbounded buffer growth.** Nothing ever truncated the shell buffer
   (shell-maker never calls `comint-truncate-buffer`, ACP content bypasses
   comint), so every O(buffer) mechanism degraded monotonically.

## Fixes (6 commits)

| Commit | Change |
|---|---|
| `ac5acbd` | New `agent-shell-buffer-maximum-size` defcustom (default 10000 lines, nil disables). After each turn, oldest content is deleted at fragment boundaries — whole fragments and whole activity groups, never split; the newest fragment/group and the live prompt are always kept. Full history remains in the transcript file. |
| `092d58f` | Defer markdown rendering of open fenced blocks. A new `agent-shell-markdown-open-fence` text property (backtick count . scan position) is stamped with the watermark; while set, a chunk only scans the new tail for the closing fence (O(chunk)) and the chunk carrying the closer gets the full render. Split closers detected. |
| `488cc93` | `pad-rendered-blocks` runs only around construct activity (a pass that rendered a block/table/list, or the pass right after one, which covers a table/list stopping). Prose chunks skip the whole-body walk entirely. |
| `73275c7` | Fragment ranges (block/body/labels) computed once and cached as markers in the fragment state's `:range-markers` alist, replacing ~7 per-chunk interval walks with O(1) marker reads. Markers released on fragment delete, regenerate, and truncation. |
| `b6db556` | Bug fix for the above: section-end markers now use insertion-type `t`, so they advance past insertions at their position — including the inline-code renderer's delete-and-reinsert of a span that closes exactly at the streamed body end (previously scrambled trailing code spans). |
| `1b3dc9d` | Review fixups: indentation, `map-put!` convention. |
| `0f6a545` | Old fragments (no `:range-markers` key) now cache via in-place alist growth (`nconc`), plus a test for marker release on truncation. |
| `HEAD` | Remaining tails: `tool_call_update` appends the output suffix instead of replacing + re-rendering the whole body (identical resends skipped entirely; 6× faster on expanded tool bodies). Renderer pass markers (source-block ranges, inline-code ranges, frozen ranges, per-block markers) are now detached after use instead of accumulating in the buffer's marker list. `agent-shell-cache-dir` is memoized (was an `expand-file-name` + `make-directory` syscall per chunk). |

## Verification

- Tests: 439/439 pass (13 new across truncation, fence deferral, markers).
- Benchmarks (batch, prose message):
  - 800 chunks / 77KB: 14.11s → ~2s wall (post-fix profile is flat;
    `nearest-range` and `pad-rendered-blocks` no longer appear).
  - 4000-line fenced block in 400 chunks: 4.59s → 0.30s (old code
    quadratic, new linear).
- Marker-drift probe: 0 mismatches cached-vs-searched across 800 chunks.
- Regression tests cover the trailing-code-span scramble and marker release.

## Review notes / known limitations

- Truncation is **enabled by default** (10000 lines) — visible old content
  disappears from the buffer (transcript retains it). Line-based, so a
  pathological single-line megablob is only bounded at fragment granularity.
- Tables: still re-render per streamed row. A defer-to-close redesign
  was attempted and reverted: live row-folding into an extending table
  is designed behavior (covered by `table-extends-on-streamed-rows`,
  `table-below-gap-held-back-for-pending-row`, and the pending-table
  watermark tests). A proper fix needs incremental width handling, not
  deferral.
- Suggested upstream path: split into 3 PRs (truncation / fence deferral /
  marker-cache+pad-gate) per the maintainer's small-PR preference.
