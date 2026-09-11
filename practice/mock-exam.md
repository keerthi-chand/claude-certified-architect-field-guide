# Timed mock: 32 items, 64 minutes

Original items in the official style and difficulty, calibrated against the twelve samples Anthropic publishes. Four scenarios, eight items each, single best answer unless an item says otherwise. That is half an exam at the real pace of two minutes per item. Set a timer. Grade at the end, and for every miss, name which instinct in [`guide/00`](../guide/00-how-the-exam-thinks.md) the correct answer used.

**Scoring yourself.** 26 or more correct (about 81%) puts you comfortably above the 720 bar. 22 to 25 is the pass zone; shore up your two weakest domains. Under 22, reread the two domains you missed most and retake this cold the next morning.

---

## Scenario A · Internal Analytics Assistant

*An Agent SDK assistant answers business questions through MCP tools: `get_user_entitlements`, `search_metrics`, `run_metric_query`, `run_report`, `escalate_to_analyst`. Data access is identity-bound; every answer must cite its source query.*

**A1.** In 9% of sessions the assistant calls `run_metric_query` before `get_user_entitlements`, occasionally returning numbers the user should not see. Compliance requires this never happens. Most effective change?

A. Lower the temperature so the model follows instructions more consistently
B. Add few-shot examples always showing entitlements checked first
C. Strengthen the system prompt: "You MUST verify entitlements before any query"
D. Add a programmatic prerequisite blocking `run_metric_query` until `get_user_entitlements` has returned for this session

<details><summary>Answer</summary>

**D.** "Never" plus compliance means a deterministic gate. C and B are probabilistic; the observed 9% *is* their failure rate. A does not change instruction-following guarantees.

</details>

**A2.** `search_metrics` ("Searches metrics") and `run_report` ("Runs reports") get confused: users asking "find the churn report" trigger metric searches. First step?

A. Merge them into one `search_everything` tool
B. Add eight few-shot routing examples to the system prompt
C. Rewrite both descriptions with purpose, input formats, example queries and boundaries ("use run_report when the user names a saved report; use search_metrics to discover individual metrics")
D. Add a pre-classifier that routes by keyword before the model sees the request

<details><summary>Answer</summary>

**C.** Descriptions are the selection mechanism; enriching them is the proportionate first step. D over-engineers, A is a larger change than a first step, B adds tokens without fixing the root.

</details>

**A3.** `run_metric_query` times out on a warehouse hiccup. The tool returns `isError: true` with "Query failed." The agent retries the identical query four times, then tells the user the metric does not exist. Fix the tool's contract.

A. Have the tool retry internally until the warehouse responds
B. Return `errorCategory: "transient"`, `isRetryable: true`, the attempted query, and a suggestion to narrow the date range, distinct from the "no rows matched" success case
C. Return `isError: false` with an empty result so the agent stops retrying
D. Return a stack trace so the agent has maximum information

<details><summary>Answer</summary>

**B.** Structured error metadata enables intelligent recovery and separates failure from valid-empty. C lies (empty-as-success produces "does not exist" answers); A hides the failure and blocks; D leaks internals without structure.

</details>

**A4.** The assistant has grown to seventeen tools. Selection accuracy dropped; the report agent sometimes calls raw metric tools mid-report. The architecture uses a coordinator with report, metrics and access subagents. Best restructure?

A. Scope each subagent to its role's three to five tools; if the report agent frequently needs one metric lookup, give it a single scoped tool for that and route complex metric work through the coordinator
B. Give every subagent all seventeen tools plus clearer descriptions
C. Collapse the subagents into one agent to remove routing complexity
D. Set `tool_choice` to forced selection per turn from the coordinator

<details><summary>Answer</summary>

**A.** Tool scoping plus the scoped-exception pattern. B keeps the oversized decision space; D misuses forced choice as routing; C concentrates all seventeen tools on one agent, the original problem made worse.

</details>

**A5.** Transcripts show that after about thirty turns the assistant misstates previously fetched figures (quotes $48,200 as "$48k or so," then "about $50k"). Which change most directly prevents this?

A. Instruct the model to "always quote numbers exactly"
B. Increase the context window by upgrading the model
C. Summarize more aggressively so less history accumulates
D. Persist fetched figures into a structured facts block (metric, value, filters, as-of date) included with every prompt, outside summarized history

<details><summary>Answer</summary>

**D.** The case-facts pattern for exact values. B postpones; A is a vague instruction; C increases the value-blurring that caused this.

</details>

**A6.** A user writes: "This is wrong AGAIN. Just get me a human analyst." The issue is a simple filter fix the assistant can make. The assistant should…

A. Ask the user to rate their frustration from 1 to 10 to calibrate
B. Escalate immediately, honoring the explicit request for a human
C. Run sentiment analysis and escalate only if negativity exceeds a threshold
D. Fix the filter first, then offer escalation if the user is still unhappy

<details><summary>Answer</summary>

**B.** An explicit request for a human means immediate escalation, no investigation first. D is the pattern for frustration *without* an explicit request; C and A are the distrusted proxies.

</details>

**A7.** When escalating, the analyst receives only "user needs help with churn numbers." Analysts lack transcript access. What should escalation carry?

A. Just the session ID for lookup
B. The full raw transcript, unedited
C. A structured handoff: user identity, the question, queries already run with results, a root-cause hypothesis, a recommended next step
D. The assistant's confidence score and sentiment reading

<details><summary>Answer</summary>

**C.** Structured handoff summaries are the named pattern when the human lacks transcript access. B dumps unstructured noise the analyst cannot access anyway; A assumes access that does not exist; D forwards unreliable proxies.

</details>

**A8.** `run_report` returns about sixty fields per row; the assistant needs six. After three reports, answer quality degrades and costs spike. Best remedy?

A. Trim tool output to the relevant fields (in the tool, or via a PostToolUse hook) before results enter context
B. Ask users to request fewer reports per session
C. Raise max_tokens for longer answers
D. Have the model re-summarize the full outputs each turn

<details><summary>Answer</summary>

**A.** Trim verbose tool outputs before accumulation, with the hook variant available. C and D add cost without removing the tax; B shifts the burden to users.

</details>

---

## Scenario B · Multi-Agent Competitive Research

*A coordinator delegates to a web-search subagent, a filings-analysis subagent and a synthesis subagent to produce cited competitive briefs.*

**B1.** The coordinator never spawns any subagent; it answers from its own knowledge. Its `AgentDefinition`s are correct. Likeliest configuration cause?

A. `tool_choice` is `"auto"` instead of `"any"`
B. The subagents lack descriptions
C. The subagents' system prompts are too long
D. `"Task"` is missing from the coordinator's `allowedTools`

<details><summary>Answer</summary>

**D.** No Task in `allowedTools`, no delegation. A would matter only to force *some* tool call; B affects selection among subagents, not the absence of any spawn.

</details>

**B2.** The synthesis subagent's output ignores everything the filings agent found. Filings results are visible in the coordinator's transcript. Why?

A. Subagent context is isolated; the coordinator must include the filings findings explicitly in the synthesis agent's prompt
B. Subagents share memory only when running in parallel
C. The filings agent must write to a shared conversation the synthesis agent inherits
D. The synthesis agent's temperature is too high to attend to details

<details><summary>Answer</summary>

**A.** No automatic inheritance; explicit passing only. B and C describe sharing mechanisms that do not exist; D is noise.

</details>

**B3.** Briefs on "the competitor landscape in game engines" consistently cover only rendering technology. Logs: the coordinator decomposed into "rendering benchmarks," "graphics API adoption," "GPU partnerships." Subagents each succeeded. Root cause?

A. The synthesis agent should detect coverage gaps
B. The coordinator's decomposition is too narrow for the topic; fix its decomposition prompt to enumerate the domain's breadth (pricing, ecosystem, licensing, tooling)
C. The web-search agent needs broader queries
D. The filings agent filters non-rendering documents too aggressively

<details><summary>Answer</summary>

**B.** The evidence points at the coordinator's split; downstream agents executed their assignments correctly.

</details>

**B4.** Briefs take eleven minutes; tracing shows search → filings → synthesis run strictly one after another, though search and filings are independent. Fix?

A. Give synthesis the search tools to begin early
B. Cache prior briefs and reuse overlapping sections
C. The coordinator emits the search and filings Task calls together in one response; synthesis runs after both return
D. Merge search and filings into one subagent

<details><summary>Answer</summary>

**C.** Parallel means multiple Task calls in a single response; the dependent step stays sequenced. D loses specialization; A breaks scoping; B does not address fresh topics.

</details>

**B5.** The filings subagent hits a paywalled document set and reports "analysis unavailable." The coordinator then cancels the entire brief. Design the correct propagation.

A. Keep termination; a brief missing filings data is misleading
B. The subagent returns structured context (failure type: permission, which sources were attempted, partial results from open sources, suggested alternatives) and the coordinator proceeds, annotating the brief's coverage gaps
C. The subagent returns its partial results marked fully successful
D. The subagent retries the paywall with exponential backoff before reporting the same generic status

<details><summary>Answer</summary>

**B.** Structured error context plus partial results plus coverage annotation. D retries a non-retryable permission failure; C hides the gap; A kills recoverable work.

</details>

**B6.** Final briefs contain claims nobody can trace ("revenue grew 40%" with no source). Findings pass through two summarization steps. Structural fix?

A. Require every subagent to emit structured claim-source mappings (claim, excerpt, source name or URL, date) and require each downstream step to preserve them through synthesis
B. Reduce to one summarization step
C. Append a bibliography of all consulted sources to the end of the brief
D. Prompt the synthesis agent to "always cite sources"

<details><summary>Answer</summary>

**A.** Provenance survives only as preserved structure. D is a vague instruction; C lists sources without linking claims to them; B reduces but does not prevent loss.

</details>

**B7.** Two credible sources give competitor headcount as 3,800 and 5,200. One is from 2023, one from 2026, but dates were dropped during analysis. The report calls this "conflicting data." What prevents this class of error?

A. Flag all numeric disagreements for human review
B. Let the synthesis agent average conflicting numerics
C. Require publication or collection dates in every subagent's structured output so temporal differences are not misread as contradictions
D. Prefer the source with higher domain authority

<details><summary>Answer</summary>

**C.** The temporal-data rule. D and B arbitrate silently; A outsources a problem structure would dissolve; both values are likely correct for their dates.

</details>

**B8.** You want to compare two report formats (narrative versus tabular) from the same expensive analysis baseline without re-running research. Mechanism?

A. Run the full pipeline twice with different synthesis prompts
B. Ask one session to produce both, then `/compact`
C. `--resume` the original session twice in parallel terminals
D. `fork_session` from the post-analysis baseline and explore each format in its own branch

<details><summary>Answer</summary>

**D.** `fork_session` exists for divergent branches off a shared baseline. A pays twice; C resumes one linear session; B risks cross-contaminating the comparison.

</details>

---

## Scenario C · Claude Code for CI Review

*Your team wires Claude Code into a CI pipeline for pull-request review and test generation across a monorepo with per-package conventions.*

**C1.** The CI job runs `claude "Review this diff"` and hangs until the runner times out. Fix?

A. `claude "Review this diff" < /dev/null`
B. `claude --batch "Review this diff"`
C. `claude -p "Review this diff"`
D. `CLAUDE_HEADLESS=true claude "Review this diff"`

<details><summary>Answer</summary>

**C.** `-p` / `--print` is non-interactive mode. B and D reference features that do not exist; A is a shell workaround, not the documented mechanism.

</details>

**C2.** You need findings as machine-readable objects (file, line, severity, message) to post as inline PR comments. Which invocation?

A. `claude -p --output-format xml "…"`
B. `claude -p --output-format json --json-schema findings.json "…"`
C. `claude -p "… respond in JSON please"`
D. Parse the prose output with a regex in the pipeline

<details><summary>Answer</summary>

**B.** The CLI's structured-output flags. C hopes for the format; A is not the tested flag; D is the fragile parsing the flags exist to avoid.

</details>

**C3.** Reviews flag dozens of nitpicks; developers now ignore the bot entirely, including its two real bug catches last week. Most effective prompt change?

A. Define explicit report/skip categories (report: logic bugs, security, data loss; skip: style, naming, local idioms) with a concrete example per severity, and temporarily disable the noisiest category while its prompt improves
B. Limit output to the five most important findings
C. Have the reviewer self-rate each finding from 1 to 10 and drop those under 7
D. Add "be conservative; only report high-confidence issues"

<details><summary>Answer</summary>

**A.** Categorical criteria plus the sanctioned disable-while-fixing move; trust is the stake. D is officially ineffective; C is self-reported confidence; B caps volume without changing what is reported.

</details>

**C4.** On a sixteen-file PR, single-pass review gives detailed feedback on early files, superficial on later ones, and flags a pattern in one file it approved in another. Restructure how?

A. A larger-context model so all sixteen files fit comfortably
B. Three full-PR passes; keep findings appearing in at least two
C. Require smaller PRs from developers
D. Per-file passes for local issues plus one separate cross-file integration pass

<details><summary>Answer</summary>

**D.** Attention dilution calls for multi-pass review. C shifts the burden; A mistakes window size for attention quality; B suppresses intermittently caught real bugs.

</details>

**C5.** The pipeline generates code fixes and then asks the same session to review them; the review approves nearly everything. Improve the defect catch rate.

A. Add "review critically, as if someone else wrote it" to the same session
B. Have the same session review twice and diff the findings
C. Run the review in a fresh, independent instance without the generator's context
D. Enable extended thinking on the self-review

<details><summary>Answer</summary>

**C.** Session context isolation: the generator retains its reasoning and defends it; A, D and B keep that context in place.

</details>

**C6.** Each new commit re-triggers review, and the bot reposts the same six findings every time. Fix within the review design?

A. Review only once per PR, on open
B. Only review the newest commit's diff hunk
C. De-duplicate comments in the GitHub API layer by string match
D. Include prior findings in context and instruct: report only new or still-unaddressed issues

<details><summary>Answer</summary>

**D.** The tested pattern. B loses cross-commit context; C de-duplicates text, not judgment (rephrased duplicates slip through); A misses regressions introduced later.

</details>

**C7.** Generated tests duplicate scenarios the suite already covers and ignore the team's fixture conventions. **Pick two** changes the exam expects.

A. Generate tests in a fresh session with no repo context for objectivity
B. Provide existing test files in context during generation
C. Document testing standards, valuable-test criteria and available fixtures in CLAUDE.md
D. Raise max_tokens so more tests fit

<details><summary>Answer</summary>

**B and C.** Both are named skills. D adds volume, not fit; A removes exactly the context that prevents duplication.

</details>

**C8.** The team also wants a nightly technical-debt report over the whole monorepo and a pre-merge review gate. Cost pressure says use Batches for both. Correct evaluation?

A. Batch the nightly report (latency-tolerant, 50% cheaper); keep the pre-merge gate synchronous (developers block on it, and batches have no latency SLA)
B. Batch both; poll status for the gate
C. Batch both with a timeout fallback to synchronous
D. Keep both synchronous; batches cannot return results reliably

<details><summary>Answer</summary>

**A.** Match the API to latency tolerance. B and C gamble a blocking gate on "usually fast"; D forfeits legitimate savings.

</details>

---

## Scenario D · Contract Data Extraction

*Thousands of vendor contracts → validated JSON (parties, dates, renewal terms, totals) for a downstream system; limited human reviewers.*

**D1.** Prompt-formatted JSON output breaks parsing about 4% of the time (trailing commas, prose preambles). Most reliable fix?

A. A regex post-processor stripping non-JSON text
B. Ask for YAML instead
C. Few-shot examples of perfectly formatted JSON
D. Define an extraction tool whose input schema is the output structure; call with `tool_choice` forcing it; read the `tool_use` block

<details><summary>Answer</summary>

**D.** `tool_use` plus schema eliminates syntax errors by construction. C reduces but cannot guarantee; A patches symptoms; B changes the format, not the guarantee.

</details>

**D2.** Contracts arrive as one of three types (MSA, SOW, amendment), each with its own extraction schema, and the type is not known upfront. You must guarantee structured output. Configure:

A. `tool_choice: "any"` with all three extraction tools; the model must call one and picks the fitting schema
B. `tool_choice: "auto"` with all three extraction tools
C. Three sequential forced calls, one per schema
D. `tool_choice` forced to the MSA tool as the most common

<details><summary>Answer</summary>

**A.** `"any"` guarantees a tool call and lets the model select among schemas, the exact pattern for unknown document types. B permits a text reply; D misfits two-thirds of documents; C triples cost and produces two wrong extractions.

</details>

**D3.** Validation fails on an extraction: the date is "March 2026" where the schema wants ISO 8601, and separately `governing_law` is missing because the contract genuinely never states it. Handle the retry decision.

A. Retry both errors with error feedback until valid
B. Retry `governing_law` with a higher-capability model
C. Retry the date (a format error; feedback fixes it); do not retry `governing_law` (information absent from the source); make it nullable and route onward
D. Retry neither; send the whole document to human review

<details><summary>Answer</summary>

**C.** The retry boundary: format and structure errors are retryable; absent information is not, and no model tier conjures it (B fails the same way). D wastes reviewers on a mechanical fix.

</details>

**D4.** The `auto_renewal` field is an enum `["yes","no"]`, and reviewers notice the model answers "no" for contracts with genuinely ambiguous renewal language. Schema fix?

A. Keep the binary enum but lower the temperature
B. Add `"unclear"` to the enum (and, for open categories elsewhere, an `"other"` plus detail-string pattern); route `"unclear"` to review
C. Change the enum to free text
D. Make `auto_renewal` required so the model commits

<details><summary>Answer</summary>

**B.** Enums need an escape valve for ambiguity or the model launders uncertainty into a confident wrong value. C loses structure; A and D increase forced commitment.

</details>

**D5.** Ten thousand legacy contracts must be extracted within 36 hours for a migration; results feed a nightly import, not a waiting user. Cheapest compliant design?

A. Synchronous API with twenty parallel workers
B. Batches for the first 5,000, synchronous for the rest as the deadline nears
C. Message Batches: 50% cheaper; with processing up to 24 hours, submit in windows sized so submission plus 24 hours fits 36 hours; track failures by `custom_id` and resubmit only those, chunking any oversized documents
D. One batch at hour 30 for maximum prompt-refinement time

<details><summary>Answer</summary>

**C.** Latency-tolerant plus a deadline means batch with SLA arithmetic and `custom_id` failure handling. A pays double for unneeded speed; B is A with extra steps; D's 30 plus 24 exceeds 36 and can miss the deadline outright.

</details>

**D6.** Extraction quality is inconsistent on scanned amendments whose tables render as prose. Detailed instructions have not helped. Next lever?

A. Reject scanned documents at intake
B. Lower the temperature
C. A longer system prompt describing table structures abstractly
D. Two to four few-shot examples demonstrating correct extraction from exactly these structural variants (prose-rendered tables, inline amendments)

<details><summary>Answer</summary>

**D.** Few-shot targeted at the structural variety is the named remedy when instructions alone produce inconsistency. C is more of what already failed; B does not teach structure; A abandons the workload.

</details>

**D7.** Overall accuracy is 97%, so leadership wants human review removed for high-confidence extractions. The tested prerequisite before agreeing?

A. Verify accuracy segmented by document type and by field, and set up stratified random sampling of high-confidence extractions for ongoing error measurement
B. Accept; 97% exceeds the 95% target
C. Have the model self-report confidence and trust anything 8 or above
D. Review only documents the model flags as difficult

<details><summary>Answer</summary>

**A.** Aggregates mask segments; sampling watches where you have stopped looking. B trusts the mask; C is uncalibrated self-report; D lets the model choose its own oversight.

</details>

**D8.** You add field-level confidence scores to route review. Before thresholds go live, what makes them trustworthy?

A. Route everything below 100% confidence to review
B. Calibrate the scores against a labeled validation set, then set routing thresholds from measured error rates
C. Use them as is; the model knows its own uncertainty
D. Average confidence across fields per document

<details><summary>Answer</summary>

**B.** Confidence is usable only after calibration. C is the distrusted proxy; A erases the point of routing; D hides the low-confidence field that matters inside a high average.

</details>
