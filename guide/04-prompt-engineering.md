# Domain 4 · Prompt Engineering & Structured Output (20%)

> Case blocks marked **From production** refer to the system described in [the README](../README.md#about-the-production-system).

Precision over vibes: explicit criteria, few-shot for the ambiguous, schemas for the guaranteed, batches for the patient.

**A framing note before the content.** This domain reads as if it were about document extraction, because "documents" appear everywhere. It is not. Domain 4 is the primary domain for *two* exam scenarios, CI code review and structured data extraction, so every technique gets illustrated in both costumes. Two of the six task statements (4.1 explicit criteria and 4.6 multi-pass review) never mention a document at all. Answer the *technique*, ignore the noun: "output format is inconsistent despite detailed instructions" means few-shot whether the output is review findings or extracted fields.

## 4.1 Explicit criteria beat vague instructions

- "Flag comments only when claimed behavior contradicts actual code behavior" beats "check that comments are accurate."
- **"Be conservative" and "only report high-confidence findings" do not improve precision.** Categorical criteria do: report bugs and security issues, skip minor style; define each severity with a concrete code example.
- False positives destroy trust asymmetrically: one noisy category undermines the accurate ones. The pragmatic move the exam endorses is to disable the noisy category temporarily while its prompt is fixed.

> **From production.** The gateway once appended a vague blanket hint ("the name may be wrong") to every not-found error, and measurement showed it was wrong three quarters of the time. Replacing it with categorical, probed truth (no access / wrong parent / absent) was the same trade this domain grades: precision comes from explicit categories grounded in evidence, not from cautious adjectives.

## 4.2 Few-shot prompting

- The strongest lever for **consistent formatting** and for **ambiguous-case judgment**; the model generalizes the demonstrated reasoning to new cases.
- Two to four *targeted* examples, aimed at the ambiguity, showing why one action beat plausible alternatives; show the exact output format (location, issue, severity, fix).
- Also the fix for varied document structures (inline citations versus bibliographies) and for empty or null extraction of fields that exist in unusual forms. Reduces extraction hallucination.

## 4.3 Structured output through `tool_use` and JSON Schema

- Define a "tool" whose input schema *is* your output schema, then read the `tool_use` block. This is **the most reliable structured output**; JSON syntax errors are eliminated.
- Schemas kill *syntax* errors, **not semantic ones**. Line items can still fail to sum to the stated total; values can land in the wrong field. Validation stays necessary.
- Design moves: fields **optional or nullable** when documents may lack them, because a required field the source lacks invites fabrication · enums get `"unclear"` for ambiguity and `"other"` plus a detail string for extensibility · format-normalization rules go in the prompt beside the schema.
- `tool_choice` recap: `"any"` guarantees a structured reply and lets the model pick the schema (unknown document type); a forced `{"type":"tool","name":…}` guarantees this exact extraction, now.

> **From production.** Nullable-not-fabricated is house style in the gateway: field listings omit empty descriptions and derivable labels rather than emitting filler, and a response-contract file declares each tool's required and optional keys so strictly that adding a key fails the build. When an item asks how to stop the model inventing a value for a missing field, the answer is to let the schema admit absence.

## 4.4 Validation, retry, feedback loops

- **Retry with error feedback:** resend the original document, the failed extraction and the specific validation error. Fixes format and structure mistakes.
- **Retries cannot conjure absent information.** If the value is not in the source, no retry helps; detect it, null the field, route to review.
- Feedback design: a `detected_pattern` field on findings lets you analyze which constructs drive dismissals. Self-correcting extraction: emit `calculated_total` beside `stated_total` and a `conflict_detected` boolean.

**About Pydantic**, which the official guide names. It is a Python validation library, and on the exam its role is narrow: it is *the validator that runs after the model replies*. JSON Schema constrains what the model may emit; Pydantic (or any validator) checks what arrived, including rules a schema cannot express, such as "line items must sum to the total." Items that mention it are asking about the validation stage, never about library syntax.

## 4.5 Message Batches API

| Property | Value, exactly |
|---|---|
| Cost | 50% cheaper than synchronous |
| Latency | Up to 24 hours; **no latency SLA** |
| Correlation | `custom_id` per request pairs requests with responses; resubmit only failures |
| Limitation | **No multi-turn tool calling inside a request**; tools cannot execute mid-request |
| Fit | Overnight reports, weekly audits, nightly test generation. Never a blocking pre-merge check |
| SLA arithmetic | To promise 30 hours with 24-hour processing, submit in 4-hour windows: cadence plus processing must fit the promise. Expect one item like this. |
| Cost hygiene | Refine the prompt on a sample first; maximize first-pass success before paying for volume |

> **From production.** The gateway's weekly audit jobs are batch-shaped work: latency-tolerant, run overnight, read in the morning. A live tool call inside a user's session is synchronous-shaped. The trap tailor-made for operators of such systems: one of those weekly jobs drives the tools *multi-turn*, and that part could never move to Batches, which cannot execute tools mid-request.

## 4.6 Multi-instance and multi-pass review

- **Self-review is structurally weak.** The generator keeps its reasoning context and defends its choices; extended thinking does not cure it. An independent instance brings fresh eyes.
- **Multi-pass for scale.** Per-file passes for local issues plus a separate cross-file integration pass cure attention dilution: uneven depth, contradictory findings on identical code in the same PR.
- Verification passes can attach *calibrated* confidence for routing, calibrated against labeled data, never raw self-report.

## Quick checks

**QC10.** Invoice extraction: line items always parse, but they sometimes do not sum to `stated_total`, and a discount lands in the fees field. The schema is already strict via `tool_use`. What is true?

A. Switch to prompt-formatted JSON with few-shot examples instead of `tool_use`
B. Make all fields required so the model must be careful
C. These are semantic errors schemas cannot prevent; add validation (calculated versus stated total, conflict flags) with retry-on-error feedback
D. Strict mode failed; tighten the schema types further

<details><summary>Answer</summary>

**C.** Schemas eliminate syntax errors only; sums and field placement are semantics, so validate and feed errors back. A regresses to syntax risk; B increases fabrication pressure.

</details>

**QC11.** Nightly, you analyze about 3,000 support transcripts for themes; results are read next morning. Costs are high on the synchronous API. Move?

A. Keep synchronous but lower max_tokens
B. Message Batches: 50% cheaper, processing within 24 hours fits an overnight window, `custom_id` to track and resubmit failures
C. Batches for half the volume as a hedge against the missing SLA
D. One giant synchronous request concatenating all transcripts

<details><summary>Answer</summary>

**B.** Latency-tolerant overnight volume is the canonical batch fit. A cuts quality rather than cost class; C hedges what the workload's tolerance already covers; D breaks context limits and loses per-item correlation.

</details>

**QC12.** Some contracts omit the renewal date entirely. Your extractor keeps producing plausible dates for them. Best fix?

A. Add retry with error feedback until a date is produced
B. Add "be accurate" to the system prompt
C. Lower the temperature to zero
D. Make `renewal_date` nullable and instruct that absent means null; validation routes nulls to review

<details><summary>Answer</summary>

**D.** Required fields the source lacks invite fabrication; a nullable schema plus routing is the design fix. A retries toward a value that does not exist; B is a vague instruction; C changes sampling, not incentives.

</details>

## Official references

The pages this chapter draws on. Facts here were checked against them; when they disagree with any secondary source, including this one, they win.

- [Prompt engineering overview](https://platform.claude.com/docs/en/build-with-claude/prompt-engineering/overview)
- [Be clear and direct](https://platform.claude.com/docs/en/build-with-claude/prompt-engineering/be-clear-and-direct)
- [Use examples (multishot prompting)](https://platform.claude.com/docs/en/build-with-claude/prompt-engineering/multishot-prompting)
- [Let Claude think (chain of thought)](https://platform.claude.com/docs/en/build-with-claude/prompt-engineering/chain-of-thought)
- [Use XML tags](https://platform.claude.com/docs/en/build-with-claude/prompt-engineering/use-xml-tags)
- [Chain complex prompts](https://platform.claude.com/docs/en/build-with-claude/prompt-engineering/chain-prompts)
- [Structured outputs](https://platform.claude.com/docs/en/build-with-claude/structured-outputs)
- [Tool use: JSON schemas and `tool_choice`](https://platform.claude.com/docs/en/agents-and-tools/tool-use/implement-tool-use)
- [Message Batches API](https://platform.claude.com/docs/en/build-with-claude/batch-processing)
- [Anthropic Academy: Building with the Claude API](https://anthropic.skilljar.com/claude-with-the-anthropic-api)
