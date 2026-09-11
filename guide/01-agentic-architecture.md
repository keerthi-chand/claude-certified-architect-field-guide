# Domain 1 · Agentic Architecture & Orchestration (27%)

The largest domain, and for most people who have *built* MCP servers the least familiar one, because it sits on the other side of the wire: you built the tools an agent calls; this domain builds the agent that calls them. Every time an Agent SDK concept appears, ask what it looks like from the tool server's side. Usually you have already met it there.

## 1.1 The agentic loop

The model never runs anything. Your code sends messages, the model replies, and the reply's `stop_reason` tells your code what to do next.

```
messages = [user_request]
loop:
  resp = client.messages.create(model, tools, messages)
  if resp.stop_reason == "tool_use":
      messages.append(assistant: resp.content)          # the tool_use block(s)
      results = [run(block) for block in tool_use_blocks]
      messages.append(user: tool_result blocks)          # results go back into the conversation
      continue
  if resp.stop_reason == "end_turn":
      return final_text                                  # the model decided it is done
```

Three load-bearing facts:

1. The loop **continues on `"tool_use"`** and **terminates on `"end_turn"`**. That is the model's signal; nothing else is.
2. Tool results are **appended to the conversation** so the next iteration can reason over them.
3. *Which* tool to call next is model-driven reasoning over context, not a pre-wired decision tree.

**Named anti-patterns** (the exam quotes these almost verbatim):
- Parsing the assistant's natural-language text to decide the loop is done.
- An arbitrary iteration cap as the *primary* stopping mechanism (fine as a safety net, wrong as the design).
- Treating "the response contains text" as completion. Responses can contain text *and* tool_use blocks.

> **From production.** Every client session against the gateway *is* this loop. The client sends a field-search result back into the conversation and the model decides that a query comes next. Nothing in the server ever told it the sequence; the tool descriptions and the data notes in each response shaped its reasoning. That is model-driven decision-making, steered from the tool side.

## 1.2 Coordinator–subagent orchestration

Hub-and-spoke: a coordinator agent owns task decomposition, delegation, aggregation, and error handling, and **all inter-subagent communication routes through it**. Subagents never talk to each other. The reasons the exam wants you to name: observability, consistent error handling, controlled information flow.

- **Subagents have isolated context.** They inherit nothing automatically. Whatever the searcher found, the synthesizer knows only if the coordinator puts it in the synthesizer's prompt.
- The coordinator should **dynamically select which subagents a query needs**, not push every query through the full pipeline.
- **The failure mode with its own exam question:** overly narrow decomposition. Every subagent succeeds, the output misses whole areas, and the coordinator's split was the defect.
- **Iterative refinement:** the coordinator evaluates synthesis for gaps, re-delegates targeted queries, and re-synthesizes until coverage is sufficient.
- **Partition scope** across subagents (distinct subtopics or source types) to avoid duplicated work.

## 1.3 Spawning, context passing, parallelism

| Mechanism | Fact to know cold |
|---|---|
| `Task` tool | How a coordinator spawns subagents. The coordinator's `allowedTools` **must include `"Task"`** or it cannot delegate at all. Current Claude Code has renamed the tool `Agent` (v2.1.63) and keeps `Task` as an alias; the exam guide v1.0 says `Task`, so answer `Task`. |
| `AgentDefinition` | Per-subagent configuration: description (how the coordinator picks it), system prompt, tool restrictions. |
| Context passing | Explicit only. Paste prior findings into the subagent's prompt. Use structured formats that separate content from metadata (source URL, document name, page) so attribution survives. |
| Parallelism | Emit multiple `Task` calls **in a single coordinator response** and they run in parallel. One per turn runs them sequentially. |
| Coordinator prompts | State goals and quality criteria, not step-by-step procedure, so subagents keep adaptability. |
| `fork_session` | Branch two explorations from one shared analysis baseline without re-analyzing. |

> **From production.** The gateway learned the parallel rule from the receiving end. When a client emits several tool calls in one response, they arrive at the server concurrently, which is exactly why its dispatch had to become asynchronous and its shared caches had to be locked. And its concurrency test harness passes each simulated caller's identity explicitly, because nothing is shared implicitly. Same law on both sides of the wire: **context moves only when someone moves it.**

## 1.4 Enforcement: programmatic versus prompt

The single most-tested judgment in this domain. When compliance must be guaranteed (identity verified before a refund, a cap on financial actions), a prompt is guidance with a non-zero failure rate; a **programmatic prerequisite** that blocks the downstream tool until the upstream one has returned is a guarantee. The official sample: an agent skips verification in 12% of cases, and the correct fix is the gate, not stronger wording, not few-shot, not a routing classifier.

Also here:
- **Structured handoff summaries** when escalating to a human who cannot see the transcript: customer ID, root cause, amount, recommended action.
- **Decomposing multi-concern requests** into distinct items, investigating each in parallel over shared context, then synthesizing once.

> **From production.** The gateway's policy layer is this taxonomy made concrete. Enforcement functions run *in code before any query executes*: SQL execution is blocked outright and no policy file can re-enable it, and a guard test fails the build if a tool is registered without read-only annotations. Meanwhile the data notes returned with each response ("always include a date filter on this explore") are guidance the model follows. Guarantees in code; guidance in prompts; both on purpose.

## 1.5 Hooks

- **PostToolUse** intercepts tool *results* before the model sees them. The canonical use is normalization: Unix timestamps, ISO 8601 and numeric status codes arriving from different tools become one format.
- **Tool-call interception** inspects and can block *outgoing* calls: refuse a refund over $500 and redirect to the escalation workflow. The SDK's name for this hook is **PreToolUse**; it returns a permission decision (`allow` / `deny` / `ask`) and can rewrite the call's input through `updatedInput`. Expect either name in an option.
- **Subagent lifecycle hooks**, beyond the v1.0 guide but real: `SubagentStart` fires when a subagent is spawned (observational), `SubagentStop` when it finishes and can send it back to work by exiting with code 2. Neither rewrites subagent output.
- Choose hooks over prompt rules whenever a business rule requires guaranteed compliance. Same logic as 1.4, implemented at the SDK seam.

**The confusion to drill:** PostToolUse fires *after* the action and shapes what the model reads; interception fires *before* and decides whether the action happens. A PostToolUse hook cannot prevent a refund; it can only notice one.

## 1.6 Task decomposition strategies

| Pattern | When | Example |
|---|---|---|
| Prompt chaining (fixed sequence) | Predictable multi-aspect work | Review each file individually, then one cross-file integration pass |
| Dynamic decomposition | Open-ended investigation; next steps depend on findings | "Add tests to a legacy codebase": map structure, find high-impact areas, build a plan that adapts |

Per-file passes plus a separate integration pass exist to fight **attention dilution**: one pass over fourteen files produces uneven depth and contradictory findings. Domain 4 tests the same idea from the review side.

## 1.7 Sessions: resume, fork, staleness

- `--resume <session-name>` continues a named prior conversation.
- `fork_session` branches divergent explorations from a shared baseline.
- **The staleness rule.** If the code changed since the session gathered its tool results, resuming inherits stale beliefs. When only a little changed, resume and tell the session precisely what moved. When much changed, start fresh and inject a structured summary of what still holds. The word that decides between them in a stem is the adverb: "a file changed" versus "heavily refactored."

> **From production.** The gateway's maintainers keep a curated state document that every working session loads at its start, instead of trusting a stale transcript. It even warns its reader that commit hashes go stale while "the shape of the situation" does not. That is the staleness rule, practiced daily.

## Quick checks

**QC1.** Your agentic loop sometimes stops mid-task. The harness terminates whenever the assistant's response contains any text content. What is wrong?

A. The loop should terminate only when max_tokens is reached
B. The loop should count iterations and stop at a fixed cap
C. The text should be parsed for a completion phrase before terminating
D. Responses can contain text alongside tool_use blocks; only stop_reason "end_turn" signals completion

<details><summary>Answer</summary>

**D.** Text presence is not a termination signal; a response can narrate *and* request tools. `stop_reason` is the contract. A and B are the named iteration-cap anti-pattern; C is the named natural-language-parsing anti-pattern.

</details>

**QC2.** Your coordinator invokes search, analysis and synthesis subagents one Task call per turn. Research takes three times longer than needed. The fix?

A. Replace the subagents with one large agent holding all tools
B. Emit the independent Task calls together in a single coordinator response so subagents run in parallel
C. Give the synthesis agent the search tools so it can self-serve
D. Raise max_tokens so more work fits per turn

<details><summary>Answer</summary>

**B.** Parallel spawning is multiple Task calls in one response. C violates tool scoping; D is unrelated to wall-clock time; A trades latency for degraded tool selection and loses isolation.

</details>

**QC3.** Refunds must never exceed $500 without human sign-off; this is a compliance requirement. Which implementation satisfies it?

A. A tool-call interception hook that blocks process_refund above $500 and redirects to escalation
B. A system-prompt rule: "Never process refunds above $500; escalate instead"
C. A PostToolUse hook that flags oversized refunds after they complete
D. Few-shot examples showing sub-$500 refunds and escalations above

<details><summary>Answer</summary>

**A.** "Never" means deterministic, so intercept the outgoing call. B and D are probabilistic. C fires after the money moved; PostToolUse transforms results, it does not prevent actions.

</details>

## Official references

The pages this chapter draws on. Facts here were checked against them; when they disagree with any secondary source, including this one, they win.

- [Building effective agents (Anthropic engineering)](https://www.anthropic.com/engineering/building-effective-agents)
- [How we built our multi-agent research system (Anthropic engineering)](https://www.anthropic.com/engineering/multi-agent-research-system)
- [Tool use overview: the request/response loop and `stop_reason`](https://platform.claude.com/docs/en/agents-and-tools/tool-use/overview)
- [Claude Agent SDK: overview](https://platform.claude.com/docs/en/agent-sdk/overview)
- [Claude Agent SDK: subagents](https://platform.claude.com/docs/en/agent-sdk/subagents)
- [Claude Agent SDK: sessions (resume and fork)](https://platform.claude.com/docs/en/agent-sdk/sessions)
- [Claude Agent SDK: hooks](https://platform.claude.com/docs/en/agent-sdk/hooks)
- [Claude Agent SDK: permissions](https://platform.claude.com/docs/en/agent-sdk/permissions)
- [Claude Code: hooks reference](https://code.claude.com/docs/en/hooks)
- [Claude Code: subagents](https://code.claude.com/docs/en/sub-agents)
