# Workflow Visualization Proposal — native output view

Status: **design only, not implemented.** Source: analysis of a live ultracode/Workflow
agent's bridge log (`/tmp/claude_bridge.log`, session `0afc0471…`) against the current
rendering path. No bridge change is required — the data already arrives.

## 1. Root cause: the rich feed is dropped, not missing

The bridge forwards **every** `SystemMessage` generically (`bridge/main.py:1072-1079`), so
the per-tick `SystemMessage subtype=task_progress` records — each carrying a full
`workflow_progress[]` tree — **do reach the plugin**. But `_on_msg_system`'s dispatch
(`session.py:1758-1768`) only routes `compact_boundary / task_started / task_updated /
task_notification`; `task_progress` matches nothing and is **silently discarded**.

Result: an entire spawned workflow collapses to one generic Task line
`☐ Task: subagent_type - description` (`tool_formatters.py:75-78`) plus a single global
spinner — the "appears busy" state. Confirmed live: the running agent's view showed only
`Claude: ⠧ thinking…, $12.69, 8q, ctx:141k` and a *static* todo checklist, while the bridge
streamed detailed per-agent progress every few hundred ms.

**This is a consumer + renderer, not a bridge change.**

## 2. Data available per tick (dropped today)

Each `task_progress` is a full re-snapshot keyed by `task_id`, with `workflow_progress[]`
where each `workflow_agent` carries up to ~19 fields:

- `phaseIndex` / `phaseTitle` (e.g. "Engine"), `index`, `label` (e.g. `scene.common`)
- `state` (`queued`/`progress`/`done`; failure/retry vocabulary **unconfirmed** — log only showed `done`)
- `model`, `attempt`
- `lastToolName` + `lastToolSummary` — the live "what is it doing right now" + target file
  (e.g. `Read /…/picking.nim`, `Bash cd /tmp && nim c …`). **Highest-value field.**
- `tokens`, `toolCalls`, `startedAt` / `queuedAt` / `lastProgressAt`, `durationMs`
- `resultPreview` (on done — a ready-made collapsed summary), `promptPreview`, `agentId`

Also available: phase ordering (`workflow_phase[]`), and **concurrency** (multiple agents in
`progress` under one phase). Noise to ignore: `thinking_tokens` records (~67% of the log,
parent-only). There is **no terminal "workflow complete" event** — completion must be inferred.

## 3. Proposed: a live Workflow Panel

Replace the single `Task:` line (in place) with a multi-row, in-place-updating text block:

```
▼ pick-registry-rewrite · 2/5 phases · 4 agents · 78.2k tok · ⏱ 3m12s
  Engine   ✔ ·····
    ✔ scene.common     opus  16 tools  31.0k  1m40s  done
  Editor   ◐ ··
    ◐ inspector.panel  opus  Read  inspector_view.nim   9 tools 12.4k 0:48
    ◐ gizmo.transform  opus  Bash  cd /tmp && nim c …    4 tools  6.1k 0:31
  Modules  ○
```

- **Header (always live):** workflow name · phases done/total · agent count · summed token
  burn (sum per-agent `tokens`, *not* the unreliable top-level `usage`) · elapsed
  (now − earliest `startedAt`) · `▼/▶` collapse toggle.
- **Phase rows** (per `workflow_phase`, ordered by `index`): title + aggregate glyph
  (`✔` all done / `◐` any in progress / `○` all queued) + a per-agent dot-strip for at-a-glance
  width without expanding.
- **Agent rows** (per `workflow_agent`, indented, ordered by `index`): state glyph · `label` ·
  `model` · **live `lastToolName` + `lastToolSummary`** · `toolCalls` · `tokens` · live elapsed.
  On done: activity column → truncated `resultPreview`, glyph → `✔`.
- **Update:** store the latest snapshot keyed by `(phaseIndex, index, label)` (`agentId` is
  absent on first `start`) and rebuild the panel HTML. Animation/elapsed advance on the
  existing spinner timer between ticks.
- **On completion — minimal (decision):** when all agents are `done`, collapse the panel to
  **just its header line** (name · phases · total tokens · total time). Per-agent rows and
  `resultPreview`s drop from the committed view to keep history short; full state stays in
  `_workflows`.

## 4. Reuse map (existing primitives → change sites)

| Panel piece | Existing primitive | Change site |
|---|---|---|
| Consume the dropped feed | — | add `_on_sys_task_progress` to `_on_msg_system` (`session.py:1758-1768`) |
| State that survives history cap | `_bg_tools` pattern (`session.py:136`) | new `_workflows: {task_id: WorkflowState}` |
| Render the panel | **PhantomSet** (decision) anchored at the Task line position | rebuild HTML and `phantom_set.update(...)` each debounced tick — no text-region surgery |
| Anchor the phantom | tracked HIDDEN region at the Task line (`output.py:2220`) | one `claude_workflow_{task_id}` region the phantom binds to |
| Animated glyph + live elapsed | spinner timer / `advance_spinner` (`output.py:2070`) | rebuild phantom HTML on the existing tick (free) |
| Activity-column formatting | `format_tool_detail` (`tool_formatters.py:163-207`) | reuse truncation/summary for `lastToolName`+`lastToolSummary` |
| Aggregate indicator | `_status` / `_update_title` | optional "phase 2/5" in status bar |

**Using PhantomSet (decision):** an HTML phantom is easier to manage than tracked-region text
surgery — each tick you rebuild the HTML and `update()` the set; no diff/replace of buffer
text, and it sidesteps the "panel scrolled into committed history" patching problem entirely.
It also gives per-state color for free. Tradeoffs to handle: phantoms aren't selectable/
copyable (acceptable — this is a live status panel, not content) and can flicker under rapid
updates → mitigate with the existing debounce + no-op-tick suppression so `update()` only
fires on a material change.

## 5. Edge cases (designed for)

- **Throttle:** route every tick through the existing 10ms debounce; additionally **drop no-op
  ticks** (only `lastProgressAt` moved) — snapshots are near-identical every tick.
- **`thinking_tokens` noise:** keep out of the panel render path entirely.
- **History truncation:** `_workflows` holds state past `HISTORY_CAP` like `_bg_tools`; the
  visual panel may scroll away (acceptable), terminal summary survives.
- **State vocabulary:** **confirmed from the live log** (all-success run): `start → progress →
  done`, `attempt` always `1`, `task_notification.status = completed`. **Failure/retry states
  are still unobserved** — this run succeeded, and the runtime's enum can't be reliably pulled
  from the minified CLI bundle. So: glyphs for `start:○, progress:◐, done:✔` are final; the
  failure/retry map (`✘`/`↻`/`⊘`) stays **defensive with unknown → neutral `?`** until a real
  failing run is captured (see §7). Note: the bridge-connection `subtype=status` lifecycle
  (`requesting/connected/needs-auth/failed`) is unrelated to agent state — don't conflate.
- **No workflow-complete event:** infer done when all agents are `done` (soft); let the parent
  Task tool's own done/error path be the authoritative close — do not auto-destroy the panel.
- **Wide fan-outs:** phase rows always render; agent rows collapse past a threshold (active +
  recently-done, `+N queued`); dot-strip gives the full picture; bounds panel height.
- **Graceful degradation:** if `workflow_progress[]` is absent/malformed (older bridge), fall
  back to today's single `Task:` line unchanged — the panel is purely additive.

## 6. Rollout (smallest high-value first)

1. Wire the feed + `_workflows` store; no UI — verify the tree parses and keys track across ticks.
2. PhantomSet panel in place of the `Task:` line (header + phase rows + agent rows) — closes ~80% of the gap.
3. Liveness: animated `◐`, live elapsed, no-op-tick suppression, header token count.
4. Wide-fanout collapse + dot-strips; minimal collapse-to-header on completion.
5. Status-bar/title aggregate (phase x/N).

## 7. Decisions (locked)

1. **State vocabulary → get real states first.** Success path confirmed from the live log
   (`start/progress/done`, `attempt=1`). **Outstanding:** capture a real *failing/retried*
   workflow run to confirm the failure/retry enum before finalizing those glyphs — the
   succeeding run never emitted them and the minified bundle isn't a reliable source. Until
   then, failure glyphs stay defensive (`unknown → ?`). Likely capture source: the running
   build-verify workflow (a Nim build failure would emit the first non-success state) — watch
   `/tmp/claude_bridge.log`.
2. **Finished panel → minimal:** collapse to the header line on completion (§2).
3. **Rendering → PhantomSet** (HTML), not plain text — easier to manage, color for free;
   flicker mitigated by debounce + no-op-tick suppression (§3, §4).
4. **Token display → cumulative count only** (no live rate).
