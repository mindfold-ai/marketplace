# DeepSeek Harness synchronization

This addendum covers Trellis's DSH platform on the 0.7 beta line and the verified
DSH 0.2.1-alpha.2 tool contract. It does not update the historical support matrix
for other platforms. Select behavior by the actual tool catalog and schema, not
the presence of an npm package or optional plugin alone.

## Role and context ownership

DSH role prompts are child-only skills under
`.dsh/skills/trellis-agent-<role>/SKILL.md`. Every child prompt starts with
`Active task: <task path>` and tells the child to load its matching role skill
exactly once. The main session must not load those role skills itself. Children
must implement or check directly, without recursively dispatching implement/check.
Keep planning approval, task activation, independent checking and commit gates.

## Capability-based dispatch

| Available capability | Safe synchronization path |
| --- | --- |
| Model-callable `trellis_wait` | Use normal continuable dispatch, do independent work, then wait once per dependent child id. Require a successful settled outcome and inspect the actual report or transcript before a dependent gate. |
| Legacy schema accepts `run_in_background` but no waiter | Dispatch with `run_in_background: false` from the outset and inspect the actual successful result and report. |
| Alpha.2 with a verified native notification/yield path but no waiter | `subagent` returns a `subagentId` immediately and has no `run_in_background`. Yield the parent and consume the actual successful result and report or transcript before dependent work. |
| No reliable synchronization path | Do the work in the main session; do not start an unsynchronizable child or load child-only role skills in the parent. |

## Settlement evidence and failure behavior

- Child activation is not completion. An inactive catalog entry does not establish
  a successful outcome; unknown, aborted or failed outcomes block dependent gates.
- A wait result or end event does not prove a native settlement notice was queued
  or delivered. `settlementNoticeQueued: false` means unverified, not proven absent.
- The optional `dsh-trellis` plugin supplies breadcrumbs, commands and `trellis_wait`
  when loaded by the host. It is not required for safe main-session work and does
  not turn the SDK `session/wait` API into a model-callable tool.
- Never simulate waiting with shell sleep, polling loops, `job_output`, repeated
  `list_agents`, or another long-running command.

## Official contract sources

- [Alpha.2 subagent tool](https://github.com/deepseek-ai/deepseek-harness/blob/dsh-v0.2.1-alpha.2/packages/subagent/tool-subagent/src/index.ts)
- [Alpha.2 settlement manager](https://github.com/deepseek-ai/deepseek-harness/blob/dsh-v0.2.1-alpha.2/packages/subagent/subagent/src/manager.ts)
