# Symptom → root cause → fix

The exam as a lookup table. Every row is phrased the way a stem is phrased, so on the day you are matching the *symptom* rather than recalling an objective. Thirty rows cover most of what sixty items can ask.

| Symptom in the stem | Root cause | Fix |
|---|---|---|
| Agent skips a required tool in N% of cases; money or identity at stake | No enforcement, only instructions | Programmatic prerequisite gate blocking the downstream calls |
| Picks the wrong tool among similar ones; descriptions are thin | Descriptions are the selection mechanism | Expand each: purpose, input formats, example queries, edge cases, boundaries versus neighbours |
| Still misroutes *after* descriptions are good | System-prompt keyword collision | Audit standing instructions for tool-name-shaped imperatives |
| Two tools genuinely overlap in function | Ambiguous boundary | Rename and re-scope, or split the generic tool into purpose-specific ones |
| Escalates easy cases, attempts hard ones | Undefined decision boundary | Explicit escalation criteria plus few-shot showing escalate versus resolve |
| Report misses whole topic areas; every subagent succeeded | Coordinator decomposition too narrow | Fix the coordinator's decomposition to enumerate the domain's breadth |
| Downstream agent ignores upstream findings | Subagent context is isolated | Put the findings in that subagent's prompt, explicitly |
| Coordinator never delegates at all | `"Task"` absent from `allowedTools` | Add it |
| Independent subagents run three times slower than needed | One Task call per turn | Multiple Task calls in one response |
| One subagent timeout aborts the whole run | Hard failure or generic status | Structured error context plus partial results; coordinator proceeds and annotates coverage gaps |
| Zero rows treated as a failure; endless retries | Valid-empty conflated with access failure | Distinguish them in the response contract |
| Claims in the final report cannot be traced | Attribution lost in summarization | Structured claim-source mappings preserved through every synthesis step |
| Credible sources "contradict" each other | Dates were dropped | Require publication or collection dates; annotate conflicts with attribution, never average or pick |
| Fourteen-file review: uneven depth, contradictory findings | Attention dilution | Per-file passes plus one separate cross-file integration pass |
| Model approves its own generated code | Generator retains reasoning context | Independent review instance, not "review critically," not extended thinking |
| CI job hangs waiting for input | Interactive mode | `claude -p` / `--print` |
| CI findings unparseable for inline comments | Free-form output | `--output-format json` plus `--json-schema` |
| Same findings re-posted on every commit | No memory of the prior review | Include prior findings; instruct "report only new or still-unaddressed" |
| Developers ignore the review bot entirely | False positives poison trust | Categorical report/skip criteria plus severity with code examples; temporarily disable the noisy category |
| Generated tests duplicate existing coverage | No visibility of the suite | Provide existing test files; document standards and fixtures in CLAUDE.md |
| About 4% of JSON replies will not parse | Prompt-formatted output | `tool_use` with a JSON schema; read the `tool_use` block |
| Schema-valid but line items do not sum, or a value is in the wrong field | Semantic, not syntactic | Validator plus retry including the specific error; `calculated_total` versus `stated_total` |
| Model invents values for fields the source lacks | Required fields force a value | Nullable or optional fields; enum `"unclear"`; `"other"` plus detail |
| Retries never succeed on one field | Information absent from the source | Stop retrying; null it and route to review |
| Misquotes figures agreed twenty turns ago | Progressive summarization blurs values | Persistent case-facts block outside the summarized history |
| Findings from the middle of a long input go missing | Lost in the middle | Key findings first; explicit section headers |
| Quality drops after a few tool calls; costs spike | Forty-field results accumulating | Trim to relevant fields before they enter context (in the tool, or with a PostToolUse hook) |
| Long session drifts to "typical patterns" | Context degradation | Scratchpad files, subagent delegation for verbose work, `/compact`, phase summaries |
| "97% accurate, let's drop human review" | Aggregate masks a bad segment | Segment accuracy by document type *and* field; stratified sampling of high-confidence output |
| Tool returns three matching customers | Ambiguous identity | Ask for another identifier; never heuristic selection |

## The proportionate ladder

Climb, do not leap.

1. **Tool descriptions and explicit criteria.** Cheapest, highest leverage, usually "the first step."
2. **Few-shot examples.** When instructions are clear but output is inconsistent, or the case is ambiguous.
3. **Schema or structural change.** Nullable fields, enum escape values, split tools.
4. **Programmatic gate or hook.** Only when a guarantee is required.
5. **New infrastructure** (classifier, routing layer, ML). Almost always the wrong answer on this exam.

Exception to rung order: if the stem says *never*, *must*, *compliance* or *financial*, jump straight to rung 4. Prompt rungs cannot make guarantees.

## Item-format tells

- **"X only" or "X alone is sufficient"** in an option is frequently planted to be beaten by a "both are required" option below it.
- **A compound option** is the trap when both parts fix the *same* hole, and the answer when they fix *different* holes and the stem says "completely," "fully" or "all cases."
- **"Most effective first step"** means the cheapest root-cause fix, not the most thorough one.
- **Evidence in the stem** (logs, percentages, a quoted decomposition) names the layer to fix. Options blaming other layers are distractors.
- **Bulk operations** ("read all files," "give it all the tools," "a larger context window") are anti-patterns by default.
- **Multiple-response items** state how many to pick. Pick exactly that many.
