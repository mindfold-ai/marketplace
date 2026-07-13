---
name: mem-recall
description: Search and recall past AI conversations across Claude Code, Codex, and Pi (OpenCode reader temporarily unavailable on 0.6.0-beta.4) via the `trellis mem` CLI. Use whenever the user asks to remember, find, or look up anything discussed in previous AI sessions — across platforms, projects, or time. Triggers on phrases like "我之前跟 Claude/Codex 讨论过 X", "上次怎么处理 Y", "翻一下历史对话", "我们当时怎么决定 X 的", "为什么我们选了 X 而不是 Y", "find what I said about Z", "what did I discuss last week", "the rationale for choosing X", "find the brainstorm where we picked Z over alternatives". Use even when the user doesn't say "history" or "recall" — any reference to past AI-conversation content should trigger this skill. The tool reads sessions directly from each platform's local storage; nothing is uploaded.
---

# Mem Recall

Cross-platform conversation memory for Claude Code, Codex CLI, and Pi. The `trellis mem` command reads each platform's local session storage, cleans the dialogue (strips system prompts, tool noise, hook injections, compact summaries handled correctly), and exposes a focused 5-command CLI for recall workflows. **OpenCode reader is temporarily unavailable on 0.6.0-beta.4** — `--platform opencode` returns empty results and prints a one-shot stderr warning; will return in a future release with an install-resilient backend.

## Prerequisite

Trellis CLI **0.6.7 or later** installed globally:

```bash
npm install -g @mindfoldhq/trellis@latest
trellis --version   # must be 0.6.7 or later
```

`trellis mem` ships bundled with the CLI; no extra setup. Version 0.6.7 adds
project-local Pi `sessionDir` discovery (see the storage table below).

The OpenCode reader is **temporarily unavailable in 0.6.0-beta.4** (returns
empty + a one-shot stderr warning). Reverted from 0.6.0-beta.3 due to
native-dependency install failures on Windows. Will return in a future
release with an install-resilient backend.

## When to use this skill

Use proactively whenever the user asks any of:

- "我之前跟 Codex/Claude 讨论过 X，你了解下"
- "上次我们怎么解决 Y 的？"
- "翻一下历史看看"
- "我之前在 trellis 项目里聊过哪些 plugin 设计？"
- "find sessions about memory architecture"
- "what did I tell another AI about this last week"

**Design-rationale flavour** (favour `--phase brainstorm` for these — see below):

- "我们当时怎么决定 X 的？"
- "之前讨论过 X 的 trade-off"
- "为什么我们选了 X 而不是 Y"
- "what was the design discussion about Y"
- "the rationale for choosing X"
- "find the brainstorm where we picked Z over alternatives"

Don't second-guess. The user's intent of "use my past conversations as context" is what this skill serves. Cross-platform / cross-project recall has no alternative tool — `git log` only sees commits, your own memory has no record of what you said in another CLI.

## The recall workflow (memorize this pattern)

Recall is a two-step drill-down, not a single query. Step 1 narrows to a session; step 2 pulls the actual content.

```
Step 1 (discover):  trellis mem search "<topic>" [--cwd <project>] [--since <date>]
Step 2 (drill):     trellis mem context <session-id> --grep <topic> --turns 3 --around 1
```

If the user is vague about which project, run `trellis mem projects` first to surface recently-active project cwds, then pick the most plausible one.

## Commands

### `trellis mem projects` — list active project cwds

Use first when the user references "the project" without saying which one. Shows distinct cwds across all platforms, ranked by last-active timestamp, with per-platform session counts. This is the AI-routing entry point.

```bash
trellis mem projects --since 2026-04-20 --limit 10
```

Output:
```
2026-05-04 03:42  sessions= 1 (claude:1)  ~/workspace/nb_project/mem-poc
2026-05-04 01:00  sessions=114 (claude:51 codex:63)  ~/workspace/.../Trellis
...
```

### `trellis mem search <keyword>` — find candidate sessions

Multi-token AND search across cleaned dialogue. Returns ranked sessions with a chunk excerpt per match. Defaults to the current working directory; use `--global` to search across all projects.

```bash
trellis mem search "trellis memory" --cwd ~/workspace/.../Trellis --since 2026-04-13
```

Output per session:
```
[claude  ] 2026-04-20 11:39  4cda3c7f-8f9  ~/.../Trellis  score=6.027  hits=169 (u=27,a=142)  turns=37
    [user] 是我们的用户想要搞类似记忆系统的东西…
    [assistant] ## Memory plugin 调研结论…
```

**How to read the score**: `(3 × user_hits + asst_hits) / total_turns`. Higher = the topic is concentrated AND the user themselves brought it up. User-turn hits weighted ×3 because user wording is the strongest topic signal — AI elaboration carries the same word repeatedly and inflates raw counts.

**Excerpts are paragraph-aligned chunks**, not char windows. They respect markdown / code-block boundaries, and prefer chunks that visibly contain ALL query tokens. When the query has multiple tokens far apart in a long turn, the chunk falls back to anchoring on the **rarest** token (more discriminating).

### `trellis mem context <session-id>` — drill into a session

After `search` picks a candidate, use this to retrieve specific hit turns plus surrounding context. Token-budgeted for direct AI consumption.

```bash
trellis mem context 4cda3c7f --grep memory --turns 3 --around 1
# top-3 hit turns + 1 turn before/after each, ≤6000 chars total

trellis mem context 4cda3c7f --turns 5 --around 0
# no grep: returns the first 5 turns (lets you see how the session opens)
```

Default budget 6000 chars (~1500 tokens); per-turn cap is half that. Use `--max-chars N` to adjust.

### `trellis mem extract <session-id>` — dump cleaned dialogue

Full conversation dump after platform-specific cleaning. Use for long-form inspection, not for budget-constrained recall.

```bash
trellis mem extract 4cda3c7f --grep memory   # filter to turns matching keyword
trellis mem extract 4cda3c7f --json          # structured output
```

### `trellis mem extract --phase brainstorm` — slice the discussion portion

A Trellis session often opens with brainstorming (the user thinking aloud, the
AI proposing options, alternatives being rejected, decisions being made),
followed by implementation work once the user runs `task.py start`. The
`--phase` flag slices the cleaned dialogue along that boundary so you can
recover the **discussion** without the implementation noise — or vice versa.

**The boundary**: a brainstorm window is everything between
`task.py create ...` and the matching `task.py start ...` Bash invocation in
the same session. Multi-task sessions produce multiple windows.

**Three values**:

| `--phase` | What you get |
|-----------|--------------|
| `all` (default) | Full cleaned dialogue — same as before this flag existed |
| `brainstorm` | Only the `[create, start)` windows — discussion / decisions |
| `implement` | Everything OUTSIDE every brainstorm window — the work itself |

**When to prefer `--phase brainstorm`**: design-rationale questions ("为什么我们当时选了 X", "the trade-off discussion about Y", "what alternatives did we reject") have much higher signal density inside brainstorm windows than across the full session. Drop straight to `extract --phase brainstorm` instead of `context --grep`, which can land you in the middle of the implementation phase where the topic is just being referenced, not decided.

**Examples**:

```bash
# Single session, brainstorm only
trellis mem extract 4cda3c7f --phase brainstorm

# Multi-task session — output is split with separators:
#   --- task: my-feature ---
#   ## Human ...
#   --- task: another-task ---
#   ## Human ...
trellis mem extract 4cda3c7f --phase brainstorm

# Filter inside the brainstorm window only (--phase runs first, then --grep)
trellis mem extract 4cda3c7f --phase brainstorm --grep "trade-off"

# Structured output: get window metadata for downstream scripts / AI
trellis mem extract 4cda3c7f --phase brainstorm --json
# JSON adds:
#   "phase": "brainstorm"
#   "windows": [{ "label": "my-feature", "startTurn": 1, "endTurn": 14 }, ...]
#   "groups":  [{ "label": "my-feature", "turns": [...] }, ...]
#   "turns":   [...]   // flat concatenation, legacy-compatible

# The inverse: just the implementation work
trellis mem extract 4cda3c7f --phase implement
```

**Platform support**:

| Platform | `--phase brainstorm` / `implement` |
|----------|------------------------------------|
| Claude | Native — boundary detection on raw JSONL `tool_use` Bash blocks |
| Codex | Native — boundary detection on `function_call` (`exec_command`) events |
| Pi | Native — boundary detection on active-branch session entries |
| OpenCode | Unavailable in 0.6.0-beta.4 — returns empty + warning |

**Edge cases handled gracefully**:

- `create` found but no following `start` (still working on the task) → window stays open through end of session.
- `start` found but no preceding `create` (task created in an earlier session) → brainstorm window is `[0, start)`.
- Neither found → full dialogue + stderr warning.

### `trellis mem list` — enumerate sessions

Mostly for browsing/debugging. Project-scoped by default; `--global` to widen.

```bash
trellis mem list --since 2026-04-27
```

OpenCode child sessions show `↳ child of <parent-id>` annotation (currently no-op — see OpenCode reader status above).

## Flags reference

```
--platform claude|codex|pi|opencode|all   default all
--since YYYY-MM-DD                     inclusive lower bound
--until YYYY-MM-DD                     inclusive upper bound
--global                               include all projects (default: cwd-scoped)
--cwd <path>                           override the project cwd
--limit N                              cap output (default 50)
--grep KW                              extract / context: filter turns by keyword
--turns N                              context: top-N hit turns (default 3)
--around N                             context: surrounding turns per hit (default 1)
--max-chars N                          context: char budget (default 6000)
--phase brainstorm|implement|all       extract: slice by [task.py create, start) (default all; Claude, Codex & Pi)
--include-children                     search / context: merge OpenCode sub-agent sessions into parent
--json                                 emit JSON
--help, -h                             show help
```

Run `trellis mem help` for the canonical flag reference.

## Where data comes from (per platform)

The tool reads these locations directly. No daemon, no index, no upload.

| Platform | Storage | Notes |
|---|---|---|
| **Claude Code** | `~/.claude/projects/<sanitized-cwd>/*.jsonl` | One JSONL per session; cwd path encoded in dirname (`/` and `_` → `-`) |
| **Codex** | `~/.codex/sessions/YYYY/MM/DD/rollout-*.jsonl` | One JSONL per session; cwd in `session_meta` payload of first event |
| **Pi** | Default `~/.pi/agent/sessions/`; env overrides; global `~/.pi/agent/settings.json`; scoped project `.pi/settings.json` | One JSONL per session; relative `sessionDir` values resolve from the settings file directory. Project-local settings are discovered for the current cwd or `--cwd`, not by an unrestricted `--global` scan. Only the active `id`/`parentId` branch is extracted. |
| **OpenCode** | Reader temporarily unavailable in 0.6.0-beta.4 | Returns empty + one-shot stderr warning |

## Cleaning rules (what's stripped from raw data)

The tool extracts only real human-AI dialogue and strips:

- **System / prompt injections**: `<system-reminder>`, `<workflow-state>`, `<INSTRUCTIONS>`, `<environment_context>`, `<permissions instructions>`, `<collaboration_mode>`, etc. (case-insensitive)
- **Bootstrap turns**: Codex injects AGENTS.md preamble as the first user message — entire turn is dropped, not just the tags
- **Tool calls and their results**: only `text` blocks are kept
- **Compact summaries**: when a session is compacted (Claude `isCompactSummary` / Codex `compacted` event), pre-compact turns are dropped and replaced by the summary, so old content isn't double-counted

This means search hits are reliable signals of "the actual conversation discussed this", not "the keyword appeared in some hook injection".

## Cross-platform sub-agent semantics

| Platform | Sub-agent storage | Recoverable? |
|---|---|---|
| Claude | Same JSONL — main agent's `Agent`/`Task` tool_use logs the prompt; tool_result has the final output. **Sub-agent's internal turns are NOT recorded** | Only prompt + final result |
| Codex | **New rollout JSONL per `codex exec` spawn**, no `parent_id` field | Treated as independent session |
| Pi | Single JSONL per session; abandoned branches dropped from the active branch, but each abandoned branch's `branch_summary` entry is kept as one summary turn | Active branch + abandoned-branch summaries |
| OpenCode | Reader temporarily unavailable in 0.6.0-beta.4 | n/a until reader returns |

`--include-children` only meaningfully changes behavior for OpenCode searches (no-op in 0.6.0-beta.4 while OpenCode reader is unavailable).

## Worked example: "what did I discuss about memory in Trellis last week?"

```bash
# 1. Confirm the project name (skip if user already named it explicitly)
trellis mem projects --since 2026-04-27
# → finds "~/workspace/.../Trellis" with 114 sessions

# 2. Find candidate sessions
trellis mem search "memory" \
  --cwd ~/workspace/.../Trellis \
  --since 2026-04-27

# → top: codex 019dcc75 (score 2.43, Codex memory subagent + Trellis hook)
#   then: claude 12d26622 (user interview about "项目记忆 4 形态")

# 3. Drill into the most relevant
trellis mem context 12d26622 --grep memory --turns 3 --around 1
# → returns the actual interview question block listing 4 memory archetypes

# 4. Now answer the user with concrete content recovered from past sessions
```

Don't run `extract` for recall unless the user explicitly wants the full session — it's expensive on token budget and rarely needed.

## Citing recalled content

When you surface recovered content to the user, cite the **session id + the actual quoted line**. Don't say "I remember we discussed X" without backing it up — the user has no way to verify and may have meant a different conversation.

Format:
```
From session 12d26622 (claude, 2026-04-20):
> 是我们的用户想要搞类似记忆系统的东西…

You proposed four memory archetypes that day: …
```

## When NOT to use this skill

- User wants to search code (use `Grep` / `Read`)
- User wants commit history (use `git log` / `gh`)
- User wants to search docs/files in current project (use `Read` / `Glob`)

This skill is specifically about **recovering past AI-conversation content**, not file content.

## Performance notes

- Project-scoped 3-week search: ~0.85s on a typical Mac
- Global search no time filter: ~3s (whole-machine session corpus scan)
- Each invocation is stateless — no cache, no daemon. Cold runs and warm runs perform similarly because macOS / Linux page cache absorbs file reads
- For interactive use, prefer `--cwd` + `--since` to narrow the corpus
