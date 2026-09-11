# How the exam thinks

> Case blocks marked **From production** refer to the system described in [the README](../README.md#about-the-production-system).

CCAR-F is a judgment exam wearing a multiple-choice costume. Every item describes a production situation, offers four plausible-sounding responses, and grades whether you recognize the *shape* of the right one. Knowing the material is necessary; it is not what separates 700 from 800. What separates them is a small set of instincts that the official sample answers reward over and over.

Learn these first. On most items they eliminate two or three options before you have thought about content.

## The seven instincts

### 1. Root cause first

Fix the layer the evidence points at. An official sample describes a research system whose final report covers only visual arts: the web-search subagent worked, the analysis subagent worked, the synthesis subagent worked, and the coordinator's logs show it decomposed "creative industries" into three visual-arts subtasks. Three of the four options blame downstream agents that did exactly what they were asked. The answer is the coordinator, because that is where the evidence is.

**Tell:** when a stem quotes logs, percentages, or a decomposition, it is naming the defective layer. Options that fix a different layer are distractors.

### 2. Proportionate first step

The exam has an implicit ladder: tool descriptions and explicit criteria, then few-shot examples, then schema or structural changes, then programmatic gates and hooks, then new infrastructure. Items that ask for the "most effective first step" want the lowest rung that fixes the root cause. An official sample about two tools with minimal descriptions ranks "expand the descriptions" above few-shot (token overhead, doesn't fix the cause), above a routing layer (over-engineered), and above merging the tools (bigger than a first step warrants).

**Tell:** a classifier, an ML model, or a routing layer is almost never the answer when a prompt-level fix is untried.

### 3. Deterministic where it matters

There is one exception to the ladder, and it is the most-tested judgment in Domain 1. When the stem involves money, identity, compliance, or the words "never" or "always," jump straight to code: a programmatic prerequisite gate, a tool-call interception hook. Prompt instructions have a non-zero failure rate, and the failure percentage quoted in the stem *is* that rate. Rewording the prompt more forcefully does not change it.

**Tell:** "in 12% of cases the agent skips verification" is not a prompt problem.

### 4. Distrust soft proxies

Self-reported confidence scores, sentiment analysis, and instructions like "be conservative" or "only report high-confidence findings" appear as options constantly and are wrong every time. Confidence is poorly calibrated (the agent is already confidently wrong on the hard cases), sentiment does not correlate with case complexity, and vague cautions do not change behavior. What works instead: explicit categorical criteria, few-shot examples of the boundary, and calibration against labeled data before any confidence score is trusted.

### 5. Match the API to the latency

Blocking workflows (a pre-merge check developers wait on) use the synchronous API. Latency-tolerant workflows (overnight reports, weekly audits, nightly test generation) use the Message Batches API for half the cost. The official sample rejects "batch both with polling" and "batch both with a timeout fallback" because a blocking workflow cannot rest on "usually fast enough."

### 6. Least privilege, with scoped exceptions

Agents get the four or five tools their role needs, not eighteen. When a specialist agent has one frequent cross-role need (a synthesis agent verifying simple facts 85% of the time), give it one narrow scoped tool for that case and keep routing the complex 15% through the coordinator. Never "give it all the search tools."

### 7. Never hide failure

Four anti-patterns show up as wrong options across every domain: returning an empty result marked successful, returning a generic "operation failed," terminating the whole workflow on one failure, and hiding what was attempted. The right shape is always structured error context (failure type, what was tried, partial results, alternatives) that lets the coordinator recover intelligently and annotate coverage gaps.

> **From production.** These instincts are not exam artifacts. The gateway that anchors this guide enforces read-only access with a code-level guard rather than a prompt because "please don't write" is probabilistic. It replaced a two-state filter lookup with a three-state one because returning an empty result on failure made a broken run indistinguishable from a clean one. Every instinct above was learned in production before it was recognized on the blueprint.

## The proportionate ladder

| Rung | Move | Reach for it when |
|---|---|---|
| 1 | Tool descriptions, explicit criteria | Selection or precision is off and the descriptions or criteria are thin |
| 2 | Few-shot examples | Instructions are clear but output is inconsistent, or the case is ambiguous |
| 3 | Schema or structural change | Nullable fields, enum escape values, splitting an overloaded tool |
| 4 | Programmatic gate or hook | A guarantee is required; see instinct 3 |
| 5 | New infrastructure | Almost never |

## Item-format tells

- **"X only" / "X alone is sufficient."** An option that pre-emptively insists it is enough is frequently planted to be knocked down by a "both are required" option below it.
- **The compound option.** Usually the trap (over-engineering). It is the answer when the two parts close *different* holes and the stem says "completely," "fully," or "in all cases." Test: same failure mode → pick the simpler option; different failure modes plus a completeness word → pick the compound.
- **"Most effective first step."** Cheapest root-cause fix, not the most thorough one.
- **Bulk operations.** "Read all the files first," "give it every tool," "switch to a larger context window" are anti-patterns by default. Larger context does not fix attention or targeting.
- **Multiple-response items** state how many to select. Select exactly that many.

## Pacing

Sixty items in 120 minutes is two minutes each, but scenario stems amortize. Read each scenario once, carefully, in about ninety seconds; its items reuse it. Flag anything over three minutes and move on. A flagged item costs nothing; a stall costs three other items. Passing is 720 on a 100–1000 scale. Domain percentages on your score report are informational and do not affect pass/fail.

## Not tested

The official out-of-scope list, which for experienced builders is a list of comfort zones to stay out of:

OAuth, API keys and authentication protocols · deploying or hosting MCP servers · rate limits, quotas and pricing · streaming and server-sent events · prompt-caching internals beyond knowing it exists · token counting · fine-tuning and training · embeddings and vector databases · vision · computer use · model benchmarking · cloud-provider specifics · language or framework internals beyond what tool and schema configuration needs.

**The trap in practice:** when a stem mentions credentials, the tested answer is configuration hygiene (`${VAR}` expansion in `.mcp.json`), never an OAuth flow. When it says "server," it means the MCP process and its tools, never the infrastructure underneath.

## Official references

The pages this chapter draws on. Facts here were checked against them; when they disagree with any secondary source, including this one, they win.

- [Claude Certified Architect – Foundations: exam guide, task statements and the twelve official sample items](https://anthropic.skilljar.com/claude-certified-architect-foundations)
- [Anthropic Academy course catalogue](https://anthropic.skilljar.com/)
