# Domain 5 · Context Management & Reliability (15%)

> Case blocks marked **From production** refer to the system described in [the README](../README.md#about-the-production-system).

The smallest domain, and the one where production scars pay off most directly. Anchor each concept to an incident and move on.

## 5.1 Preserving critical information in long interactions

- **Progressive summarization erodes exact values.** Amounts, dates, order numbers and promised expectations blur into "the customer has a billing issue." Fix: a persistent **case-facts block** of structured transactional facts, carried in every prompt *outside* the summarized history.
- **Lost in the middle.** Long inputs are processed reliably at the start and end; findings in the middle get dropped. Put key-findings summaries first and use explicit section headers.
- **Trim verbose tool outputs before they accumulate.** A forty-field record when five fields matter is pure context tax.
- Multi-agent: upstream agents return **structured data** (facts, citations, relevance scores) with metadata (dates, source locations) rather than prose and reasoning chains, because downstream budgets are finite.

> **From production.** The gateway's defining incident here was a dashboard response that reached 691,000 tokens. Payloads grew with rows times columns, blew past any context window, and the client silently dropped them; users experienced "flakiness." The row limits had counted rows while nothing counted bytes. The fix was this section's fix: byte ceilings, trimming to relevant fields, and an honest disclosure of what was cut. The same system's provenance block on every response (exact query, filters, timestamp, identity) is a case-facts block: persisted structure that survives any summarization around it.

## 5.2 Escalation and ambiguity

- **Escalate on:** an explicit request for a human (immediately, without investigating first) · policy gaps and exceptions (policy is silent on competitor price-matching) · inability to progress. Complexity alone is not a trigger.
- Frustrated but resolvable: acknowledge, offer to resolve, escalate if they insist.
- **Sentiment and self-reported confidence are unreliable proxies.** They are the officially wrong answers of this task.
- **Multiple matches → ask for another identifier. Never select heuristically.**

> **From production.** The gateway impersonates each user to run queries under their own permissions, so resolving an email to exactly one account is the whole security model. When duplicate accounts share an email, its resolver disambiguates by explicit criteria, and when those criteria tie it *refuses loudly* rather than picking the first match, because a heuristic pick once silently ran an administrator's session as a stub account. That is task 5.2 in code: on a tie, ask; never guess.

## 5.3 Error propagation strategies across multi-agent systems

Structured error context (failure type, attempted query, partial results, alternatives) lets the coordinator retry differently, reroute, or proceed with what it has. The four graded wrongs: a generic status ("search unavailable"), empty-as-success, killing the whole workflow on one failure, and hiding what was attempted. Synthesis output carries **coverage annotations**: which findings are well supported, which areas have gaps because a source was unavailable.

> **From production.** Every partial dashboard run from the gateway states exactly how many tiles ran, why it stopped (time budget or size budget), and how to continue, and its pagination advances by what was *returned*, never by what was requested. A failed filter lookup produces the loudest warning in the codebase instead of a clean-looking unfiltered run. Coverage annotation, shipped.

## 5.4 Context in large codebase exploration

- Degradation smell: the model drifts from *specific classes it read* to "typical patterns." It has lost the details.
- Remedies: **scratchpad files** persisting key findings · **subagent delegation** for verbose investigation (summaries return, noise does not) · summarize each phase before spawning the next · `/compact` when discovery output fills the window · **crash recovery** through structured state exports: each agent writes a manifest to a known location and the coordinator loads it on resume.

> **From production.** The gateway's state document, loaded at the start of every working session and updated at the end, is a scratchpad and a manifest at once. Its maintainers hit the degradation smell early, which is why it warns that commit hashes go stale while "the shape of the situation" does not.

## 5.5 Human review and confidence calibration

- **Aggregate accuracy masks segments.** Ninety-seven percent overall can hide one document type failing badly. Verify by document type *and* field before automating away review.
- **Stratified random sampling** of high-confidence output measures error where you have stopped looking and catches novel error patterns.
- Field-level confidence is usable only after **calibration against a labeled validation set**; route low-confidence and contradictory sources to the limited human capacity.

> **From production.** The gateway's health reviews never leave an error rate as an aggregate. A single figure was split into newcomers versus experienced users (the newcomers failed at three times the rate), into access failures versus naming failures (almost all were access), and by tool and cohort. When an option says "overall accuracy is high, reduce review," the question to ask is which segment is hiding in the average.

## 5.6 Provenance and uncertainty in synthesis

- Summarization loses attribution unless **claim-source mappings** (URL, document name, excerpt) are structured in and preserved through every synthesis step.
- Credible sources conflict → **annotate both values with attribution**; never arbitrarily pick or average; the coordinator reconciles. Reports separate well-established from contested findings.
- **Temporal data:** require publication or collection dates in structured output, or two measurements from different years masquerade as a contradiction.
- Render content natively: financial data as tables, news as prose, technical findings as lists.

> **From production.** The gateway's provenance block is a claim-source mapping: every number an agent quotes carries who ran it, when, from which source, with which filters, plus a link to verify. And its rule that a saved report carrying a filter for a date years in the past must never be presented as current data is the temporal rule, shipped before anyone had read this blueprint.

## Quick checks

**QC13.** A forty-turn support session: the agent begins misquoting the refund amount agreed in turn six. Summarization is on. Best fix?

A. Turn summarization off and carry the full transcript
B. Prompt the summarizer to "be careful with numbers"
C. Cap sessions at ten turns and force escalation after
D. Maintain a structured case-facts block (amounts, order IDs, dates, commitments) injected into every prompt outside the summarized history

<details><summary>Answer</summary>

**D.** The named pattern for exact values under summarization. A delays failure until the window fills; B is a vague instruction; C punishes the customer for an architecture problem.

</details>

**QC14.** Two credible market reports give different 2025 revenue figures. Your synthesis agent picks the larger one. Correct behavior?

A. Pick the more recent source's figure
B. Include both values with source attribution and dates, annotate the conflict, and let the report distinguish established from contested findings
C. Drop the metric entirely
D. Average them for a best estimate

<details><summary>Answer</summary>

**B.** Conflicts are annotated with attribution, never arbitrated silently. A and D arbitrate silently (and dates may make both correct for their periods); C discards signal.

</details>

**QC15.** `lookup_customer` returns three accounts for "Sam Lee." The agent picks the one with the latest login and proceeds to modify billing. What should it have done?

A. Asked the customer for an additional identifier (account email, order number) before any action
B. Escalated to a human immediately
C. Correct; recency is the best available heuristic
D. Modified all three to be safe

<details><summary>Answer</summary>

**A.** Multiple matches → clarification, never heuristic selection. B is disproportionate (the agent can resolve the ambiguity itself); C is the graded anti-pattern; D is malpractice.

</details>

## Official references

The pages this chapter draws on. Facts here were checked against them; when they disagree with any secondary source, including this one, they win.

- [Effective context engineering for AI agents (Anthropic engineering)](https://www.anthropic.com/engineering/effective-context-engineering-for-ai-agents)
- [Context windows](https://platform.claude.com/docs/en/build-with-claude/context-windows)
- [Context editing](https://platform.claude.com/docs/en/build-with-claude/context-editing)
- [How we built our multi-agent research system (Anthropic engineering)](https://www.anthropic.com/engineering/multi-agent-research-system)
- [Claude Agent SDK: sessions](https://platform.claude.com/docs/en/agent-sdk/sessions)
- [Claude Code: memory](https://code.claude.com/docs/en/memory)
