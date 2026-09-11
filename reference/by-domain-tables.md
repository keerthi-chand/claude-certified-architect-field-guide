# Problem → correct technique, by domain

One table per domain. Built to be printed and pinned: two pages at 100% scale, portrait.

## 1 · Agentic Architecture & Orchestration

| Symptom | What the exam expects |
|---|---|
| Deciding when the agentic loop continues or ends | Continue while `stop_reason = "tool_use"`; stop on `"end_turn"` |
| Loop ends early, or never ends | Never parse text for "done"; never use an iteration cap as the primary stop |
| Model must reason over what a tool returned | Append tool results to conversation history before the next call |
| A required step is skipped in N% of cases (identity, verification) | Programmatic prerequisite gate blocking downstream tool calls |
| A business rule must never be violated (refund cap) | Tool-call interception hook: block the call, redirect to escalation |
| Tools return timestamps and codes in mixed formats | PostToolUse hook normalizing results before the model reads them |
| Coordinator never delegates to any subagent | Add `"Task"` to the coordinator's `allowedTools` |
| Independent subagents run one after another | Emit multiple `Task` calls in a *single* coordinator response |
| A subagent ignores what an earlier agent found | Pass findings explicitly in its prompt; no context is inherited |
| Output misses whole topic areas, yet every subagent succeeded | Widen the coordinator's task decomposition |
| Synthesis comes back with coverage gaps | Coordinator re-delegates targeted queries, then re-synthesizes |
| Subagents duplicate each other's work | Partition scope: distinct subtopics or source types per agent |
| Escalating to a human who cannot see the transcript | Structured handoff: ID, root cause, amount, recommended action |
| Predictable multi-aspect work *versus* open-ended investigation | Prompt chaining *versus* dynamic decomposition that adapts to findings |
| Comparing two approaches from one costly baseline | `fork_session`: branch, do not re-analyze |
| Resuming work whose files have changed heavily since | Fresh session plus injected structured summary, not `--resume` |

## 2 · Tool Design & MCP Integration

| Symptom | What the exam expects |
|---|---|
| Wrong tool chosen among similar ones | Expand descriptions: purpose, input formats, example queries, boundaries |
| Still misroutes after descriptions are good | Audit the system prompt for tool-name-shaped keywords |
| Two tools genuinely overlap in function | Rename and re-scope, or split into purpose-specific tools |
| Agent cannot decide how to recover from a failure | Structured error: `errorCategory`, `isRetryable`, readable message |
| A policy rejection the customer must understand | `retriable: false` plus a customer-friendly explanation |
| Zero results reported as a failure | Distinguish valid empty results from access failures |
| Transient failure inside a subagent | Recover locally; propagate only the unresolvable, with partial results |
| Eighteen tools on one agent, selection degrades | Scope each agent to its role's four or five tools |
| One frequent cross-role need (85% simple, 15% complex) | One narrow scoped tool; complex cases still route via the coordinator |
| Model replies in prose when structure is required | `tool_choice: "any"` |
| A specific tool must run before the others | `tool_choice: {"type":"tool","name":"…"}` |
| Shared team MCP server needing a credential | `.mcp.json` in the repo plus `${VAR}` environment expansion |
| Personal or experimental MCP server | `~/.claude.json` |
| Calls wasted discovering what data exists | Expose the catalog as MCP *resources*, not another tool |
| Search text inside files *versus* find files by name | Grep *versus* Glob · Edit anchor not unique → Read + Write |

## 3 · Claude Code Configuration & Workflows

| Symptom | What the exam expects |
|---|---|
| A teammate never receives the project instructions | They sit in user-level `~/.claude/CLAUDE.md`; move them to the project |
| CLAUDE.md has grown monolithic | `@import` external standards, or split into `.claude/rules/` |
| Behavior differs between sessions for no clear reason | `/memory` to see which memory files actually loaded |
| A convention follows a file *type* across many directories | `.claude/rules/` with YAML `paths:` glob patterns |
| A command must reach everyone *versus* just you | `.claude/commands/` (versioned) *versus* `~/.claude/commands/` |
| A verbose skill floods the main conversation | `context: fork` in SKILL.md frontmatter |
| A skill must not touch destructive tools, or needs an argument | `allowed-tools` · `argument-hint` frontmatter |
| Always-on standards *versus* an on-demand workflow | CLAUDE.md *versus* a skill |
| Large-scale, several valid approaches, architectural | Plan mode: explore and design before changing anything |
| Scoped single-file fix with a clear cause | Direct execution |
| Verbose discovery threatening the context window | Explore subagent: isolates output, returns a summary |
| Prose descriptions interpreted inconsistently | Two or three concrete input/output examples; or test-first, iterate on failures |
| CI job hangs; findings unparseable for inline comments | `-p` / `--print` · `--output-format json` plus `--json-schema` |
| Review reposts the same comments; tests duplicate the suite | Feed prior findings and existing test files into context |

## 4 · Prompt Engineering & Structured Output

| Symptom | What the exam expects |
|---|---|
| Output shape drifts between runs | Two or three examples of the exact shape wanted |
| Malformed or unparseable JSON | `tool_use` with a JSON schema; eliminates syntax errors |
| Fabricated values for fields the source lacks | Optional or nullable schema fields |
| Enum cannot express an ambiguous case | Add `"unclear"`; `"other"` plus a detail string for open categories |
| Extraction sums do not match the stated total | Validation-retry loop; schemas fix syntax, never semantics |
| Retry keeps failing on one field | Information is absent from the source: stop, null it, route to review |
| Facts buried in prose get skipped | An example that shows the prose variant being handled |
| Unknown document type, several extraction schemas | `tool_choice: "any"` |
| Too many false positives; reviewers stop trusting output | Explicit report/skip categories, not "be conservative" |
| Severity labels applied inconsistently | Define each level with a concrete code example |
| Uneven depth and contradictory findings across many files | Per-file passes plus one separate cross-file integration pass |
| Model approves its own generated output | An independent instance, without the generator's reasoning context |
| Latency-tolerant bulk work *versus* a blocking check | Message Batches (50%, ≤24 h, no SLA, `custom_id`) *versus* synchronous |

## 5 · Context Management & Reliability

| Symptom | What the exam expects |
|---|---|
| Exact figures blur as a long session is summarized | Persistent case-facts block, outside the summarized history |
| Findings from the middle of a long input go missing | Key findings first, then details under explicit section headers |
| Verbose tool results consume the window | Trim to the relevant fields before they accumulate |
| Long exploration drifts toward "typical patterns" | Scratchpad files; delegate verbose investigation; `/compact` |
| Workflow crashes partway through | Structured state manifests the coordinator reloads on resume |
| Customer asks for a human, or policy is silent on the request | Escalate immediately; no investigation first |
| Customer is frustrated but the issue is resolvable | Acknowledge, offer to resolve, escalate only if they insist |
| A lookup returns several matching records | Ask for another identifier; never select heuristically |
| A subagent fails mid-run | Structured error context plus partial results; annotate coverage gaps |
| Claims in the final output cannot be traced | Claim-source mappings preserved through every synthesis step |
| Credible sources report conflicting figures | Annotate both with attribution and dates; never average or pick |
| "97% accurate, can we drop human review?" | Segment accuracy by document type and field; stratified sampling |
