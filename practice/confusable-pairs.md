# Confusable pairs: the mechanism-identity drill

The exam's hardest items are not about concepts you do not know. They put two mechanisms from the same neighborhood side by side, both plausible, and grade whether you can say what each one *does*. This file exists because that was the root cause of both misses on a 9/11 practice run by someone who knew the material cold.

Read the middle column, then drill.

## The table

| When you need… | Use | Not, because |
|---|---|---|
| N subagents working at once | Multiple `Task` calls in **one** coordinator response | `fork_session` clones a conversation, it does not delegate · one Task per turn is the sequential anti-pattern |
| Two approaches compared from one expensive baseline | `fork_session` | Task calls fan out to workers, they do not branch your conversation · `--resume` twice is one linear session |
| Reshape tool results before the model reads them | **PostToolUse** hook | PreToolUse (interception) stops outgoing calls, not results · a system prompt is probabilistic |
| Prevent an action that violates a rule | **PreToolUse** hook, the tool-call interception (block, `deny`, or rewrite the input) | PostToolUse fires after the act · removing the tool blocks the legitimate cases too |
| Extra rules while Claude stays a normal coding assistant | `--append-system-prompt` | `--system-prompt` *replaces* the default prompt and discards its tool guidance |
| A rule that must win every time regardless of other files | `settings.json` (strict precedence: managed > local > project > user) | CLAUDE.md files are *concatenated*; contradictions may resolve arbitrarily |
| Guaranteed structured output, schema not known in advance | `tool_choice: "any"` | `"auto"` permits a prose reply · forcing one schema misfits the rest |
| Guarantee a specific tool runs first | `tool_choice: {"type":"tool","name":…}` | Ordering the tools array has no sequencing semantics · `"any"` still lets the model pick · a prompt instruction is guidance |
| A convention that follows a file *type* across many directories | `.claude/rules/` with a `paths:` glob | Directory CLAUDE.md is directory-bound · root CLAUDE.md sections rely on inference |
| On-demand verbose workflow kept out of the main context | Skill with `context: fork` | CLAUDE.md is always loaded · `/compact` is cleanup after the damage |
| Returning to work whose prior tool results are now mostly stale | Fresh session + **injected structured summary** | `--resume` inherits stale beliefs. Resume plus "here is what changed" is right only when prior context is *mostly valid* |
| Agent should see what data exists without spending calls to find out | MCP **resources** (a content catalog) | Another tool still costs a call per discovery · stuffing the catalog in the system prompt bloats every request |
| Which files *contain* this symbol | **Grep** (scopable with a glob filter) | Glob matches names and paths only · reading every file first |
| Which files *match* this name shape | **Glob** | Grep searches contents, not filenames |
| A tool sequence that must hold every time (identity before money) | Programmatic **prerequisite gate** | System prompt · few-shot · a routing classifier (fixes availability, not ordering) |
| Bulk latency-tolerant work at half the cost | Message **Batches** + `custom_id` | Anything blocking (no latency SLA) · anything needing multi-turn tool calls (unsupported) |

## The compound-answer tell

Usually the exam punishes over-engineering, so a "do both" option is a trap. It is **not** a trap when the two parts close *different* holes and the stem asks for "completely," "fully," or "in all cases."

Test it: same failure mode → pick the simpler option. Different failure modes plus a completeness word → pick the compound.

And watch for an option that pre-emptively insists it is enough: "*X only*," "*X alone is sufficient*." That wording is frequently planted to be knocked down by a "both are required" option below it.

## Rapid-fire: twelve items, one minute each

One-line stems, no scenario reading. Say your answer aloud before opening each. Grade yourself on whether you could *name what each mechanism does* before choosing, not on the letter.

**P1.** The coordinator needs the search and filings subagents working at the same time.

A. Set `allowedTools` to both subagent type names · B. Multiple Task calls in a single coordinator response · C. `fork_session` twice, then delegate · D. Multiple Task calls, one per coordinator turn

<details><summary>Answer</summary>

**B.** Parallel delegation is several Task calls in one response. C branches a conversation; D is sequential; A misunderstands `allowedTools`, which takes tool names; the coordinator needs `"Task"`.

</details>

**P2.** Compare two refactoring strategies from the same completed codebase analysis, without redoing the analysis.

A. `/compact`, then explore both in sequence · B. Multiple Task calls in one response · C. `--resume` the session twice in two terminals · D. `fork_session` from the post-analysis baseline

<details><summary>Answer</summary>

**D.** Divergent branches off a shared baseline is what `fork_session` is for. B delegates rather than branches; C resumes one linear session; A discards the baseline detail you are preserving.

</details>

**P3.** Three MCP tools return timestamps as Unix epoch, ISO 8601 and MM/DD/YYYY. The agent should see one format.

A. A tool-call interception hook · B. A system-prompt instruction about date handling · C. A PostToolUse hook that normalizes results · D. `tool_choice` forced to a date-parsing tool

<details><summary>Answer</summary>

**C.** PostToolUse intercepts *results* for transformation, the canonical normalization case. A intercepts outgoing calls; B is probabilistic; D forces an unrelated call.

</details>

**P4.** Refunds above $500 must never execute; they route to human escalation.

A. A tool-call interception hook that blocks and redirects · B. Few-shot examples showing escalation above $500 · C. A PostToolUse hook that flags oversized refunds · D. Remove `process_refund` from the agent's `allowedTools`

<details><summary>Answer</summary>

**A.** "Never" means a deterministic block on the outgoing call. C fires after the money moved; B is probabilistic; D also blocks every legitimate refund under $500.

</details>

**P5.** Document type is unknown, three extraction schemas exist, and the output must never come back as prose.

A. `tool_choice: "auto"` with all three tools · B. `tool_choice` forced to the most common schema · C. `tool_choice: "any"` with all three tools · D. A prompt instruction: "always reply in JSON"

<details><summary>Answer</summary>

**C.** `"any"` forces some tool to be called and leaves the choice of schema to the model. A permits text; B misfits two-thirds of documents; D reintroduces syntax risk.

</details>

**P6.** `extract_metadata` must run before any enrichment tool, every single time.

A. A system-prompt rule to call it first · B. `tool_choice: {"type":"tool","name":"extract_metadata"}`, then continue in follow-up turns · C. `tool_choice: "any"` · D. List `extract_metadata` first in the tools array

<details><summary>Answer</summary>

**B.** Forced selection guarantees *which* tool; later steps run in later turns. C guarantees only that *some* tool is called; D, array order carries no sequencing semantics; A is guidance.

</details>

**P7.** A convention must apply to every `*.tf` file, and those files sit in six directories.

A. `.claude/rules/` with `paths: ["**/*.tf"]` · B. A CLAUDE.md in each of the six directories · C. A skill whose `allowed-tools` is restricted to terraform files · D. A "Terraform conventions" section in root CLAUDE.md

<details><summary>Answer</summary>

**A.** Glob-scoped rules follow a file type anywhere, deterministically. B drifts in six copies; C confuses tool restriction with convention loading; D relies on inference and always spends tokens.

</details>

**P8.** A verbose codebase-analysis workflow, invoked on demand, whose output must not pollute the main conversation.

A. Document it in CLAUDE.md · B. A `.claude/rules/` file with paths frontmatter · C. Run it normally, then `/compact` · D. A skill with `context: fork`

<details><summary>Answer</summary>

**D.** On-demand plus isolated equals a skill with `context: fork`. A is always-loaded standards; B is conditional convention loading; C cleans up after the pollution instead of preventing it.

</details>

**P9.** You return to an investigation session after a week; the files it analyzed have since been heavily refactored.

A. `fork_session` from the old session · B. `--resume` and list which files changed · C. Start a fresh session and inject a structured summary of what still holds · D. `--resume` and carry on

<details><summary>Answer</summary>

**C.** Mostly stale tool results make a fresh session with an injected summary more reliable than resumption. B is right when prior context is *mostly valid* and a file or two moved; D inherits stale beliefs wholesale; A forks the staleness.

</details>

**P10.** The agent should know which datasets, schemas and saved reports exist, without burning calls to discover them.

A. Paste the catalog into the system prompt · B. Add a `list_available_data` tool · C. Add few-shot examples that name the common datasets · D. Expose the catalog as MCP resources

<details><summary>Answer</summary>

**D.** Resources exist to give visibility into available content and cut exploratory tool calls. B still costs a call per discovery; A bloats every request; C names a few examples rather than exposing the catalog.

</details>

**P11.** Find every file that imports the deprecated `useAuth` hook.

A. Grep `useAuth`, optionally filtered with a glob · B. Glob `**/*.tsx`, then Read each result · C. Bash `find . -name "*useAuth*"` · D. Glob `**/useAuth*`

<details><summary>Answer</summary>

**A.** Import statements are file *contents*, so Grep. D and C search names; B reads the whole codebase to answer a content question, the named anti-pattern.

</details>

**P12.** Nightly test generation across a monorepo, reviewed by the team each morning; cost matters.

A. Synchronous API with parallel workers · B. Message Batches API, failures tracked by `custom_id` · C. Synchronous with a lower max_tokens · D. Batches with a synchronous fallback if it runs long

<details><summary>Answer</summary>

**B.** Latency-tolerant overnight work is the canonical batch fit at 50% savings. A pays double for speed nobody waits on; C cuts quality, not cost class; D adds complexity for a workload with no deadline pressure.

</details>

Nine of twelve answers are B. That is deliberate: if you were pattern-matching position instead of mechanism, you scored suspiciously well and learned nothing.
