# Claude Certified Architect – Foundations: a field guide

**How the CCAR-F exam thinks, and how to think with it.**

This is the companion you read last. Other resources cover the material; this one teaches the judgment the exam actually grades, and it teaches every domain through decisions made in a real production system: a governed MCP gateway that lets an AI assistant query Looker under each user's own permissions, rolled out company-wide. Passing the exam turned out to be mostly a matter of recognizing decisions that system had already forced.

> Written after passing the exam in September 2026. Every practice item here is original, modelled on the twelve samples Anthropic publishes in the official exam guide. Nothing from the real exam appears in this repository, and nothing will. See [CONTRIBUTING](CONTRIBUTING.md).

### About the production system

Every "From production" block in this guide refers to one system: a gateway that sits between AI assistants and Looker, a business intelligence platform. It exposes a small set of read-only tools over the Model Context Protocol, runs every query under the requesting user's own permissions, enforces a policy file in code, and stamps every response with provenance. It was built and rolled out company-wide by the author's team over roughly six months, and most of the exam's judgment calls turned up in it first. The company, code identifiers, and colleagues are removed; the decisions and the numbers are exact. You do not need to know anything about Looker or the gateway to use this guide. The blocks are there to show what each concept looks like when it has consequences.

---

## The exam, in one table

| | |
|---|---|
| Items | 60, multiple-choice and multiple-response |
| Time | 120 minutes (two minutes per item) |
| Structure | 4 scenarios drawn from a bank of 6; each frames a set of items |
| Passing | 720 on a 100–1000 scale |
| Delivery | Pearson VUE, online proctored or test centre |
| Access | Registration runs through the Anthropic Partner Academy and requires affiliation with a Claude Partner Network organization |
| Valid | 12 months; renewal is a free non-proctored assessment |

**Domain weights**

| Domain | Weight |
|---|---|
| 1 · Agentic Architecture & Orchestration | 27% |
| 2 · Tool Design & MCP Integration | 18% |
| 3 · Claude Code Configuration & Workflows | 20% |
| 4 · Prompt Engineering & Structured Output | 20% |
| 5 · Context Management & Reliability | 15% |

## What is different here

1. **A judgment framework, not a summary.** Seven instincts distilled from every official sample answer, a 30-row symptom → root cause → fix lookup, a proportionate-response ladder, and the item-format tells (when a "do both" option is the trap, and when it is the answer).
2. **Production case notes in every domain.** Each concept is anchored to a decision from a real gateway: an identity resolver that refuses to guess when two accounts match, byte ceilings born from a response that reached 691,000 tokens, a filter lookup rewritten to have three states because two states let a failure masquerade as success. Identifiers are removed; the decisions are exact.
3. **Confusable pairs, drilled.** The exam's hardest items put two plausible mechanisms side by side (`Task` calls vs `fork_session`, PostToolUse vs interception hooks, `tool_choice: "any"` vs forced). A dedicated drill trains mechanism identity, the single most common cause of a wrong answer by someone who knows the material.
4. **What is *not* tested, said plainly.** OAuth, MCP hosting, infrastructure, rate limits, caching internals. If that is where your expertise lives, it is where your study time is wasted.
5. **Small on purpose.** Roughly the twenty percent of material that decides eighty percent of items.

## How to use it

**In ninety minutes** — read [`guide/00-how-the-exam-thinks.md`](guide/00-how-the-exam-thinks.md), then [`reference/symptom-to-fix.md`](reference/symptom-to-fix.md), then take [`practice/confusable-pairs.md`](practice/confusable-pairs.md). This is the pre-exam morning routine.

**In a weekend** — Day one: `guide/00` and Domain 1 (the largest, and the one most builders find least familiar). Day two: Domains 2–5, then the mock under exam timing (32 items, 64 minutes). Grade it. For every miss, name which instinct the correct answer used.

**In two weeks** — one domain every two days with its quick checks; the mock at the end of week one and again cold at the end of week two; the reference tables as your final review.

## Contents

| File | What it is |
|---|---|
| [`guide/00-how-the-exam-thinks.md`](guide/00-how-the-exam-thinks.md) | The seven instincts, the proportionate ladder, item-format tells, pacing, what is not tested |
| [`guide/01-agentic-architecture.md`](guide/01-agentic-architecture.md) | **27%** · the loop, orchestration, spawning, enforcement, hooks, decomposition, sessions |
| [`guide/02-tool-design-mcp.md`](guide/02-tool-design-mcp.md) | **18%** · descriptions, structured errors, distribution, `tool_choice`, MCP configuration, built-in tools |
| [`guide/03-claude-code-workflows.md`](guide/03-claude-code-workflows.md) | **20%** · CLAUDE.md hierarchy, rules, commands and skills, plan mode, refinement, CI |
| [`guide/04-prompt-engineering.md`](guide/04-prompt-engineering.md) | **20%** · explicit criteria, few-shot, schemas, validation and retry, Batches, review passes |
| [`guide/05-context-reliability.md`](guide/05-context-reliability.md) | **15%** · context preservation, escalation, error propagation, exploration, human review, provenance |
| [`practice/mock-exam.md`](practice/mock-exam.md) | 32 items across four scenarios, hidden answers with rationales |
| [`practice/confusable-pairs.md`](practice/confusable-pairs.md) | The mechanism-identity table and a 12-item drill |
| [`reference/symptom-to-fix.md`](reference/symptom-to-fix.md) | Thirty production symptoms → root cause → fix |
| [`reference/recall-facts.md`](reference/recall-facts.md) | The exact facts: flags, keys, limits. The one perishable file, dated |
| [`reference/by-domain-tables.md`](reference/by-domain-tables.md) | Symptom → expected technique, one table per domain, built to print |

## The seven instincts

Every official sample answer rewards the same seven judgments. On a scenario item they eliminate two or three options before you have thought about content.

| | Instinct | In practice |
|---|---|---|
| 1 | **Root cause first** | Fix the layer the evidence points at. If every subagent succeeded and the output is wrong, look up at the coordinator. |
| 2 | **Proportionate first step** | Descriptions before few-shot, few-shot before gates, gates before infrastructure. A classifier is almost never the answer. |
| 3 | **Deterministic where it matters** | Money, identity, compliance, "never" or "always" → code, not prompts. The failure percentage in the stem *is* the prompt's failure rate. |
| 4 | **Distrust soft proxies** | Self-reported confidence, sentiment, "be conservative", "high-confidence only" are always wrong. |
| 5 | **Match the API to the latency** | Blocking → synchronous. Overnight and audit work → Batches. |
| 6 | **Least privilege, scoped exceptions** | Four or five role-scoped tools, one narrow tool for a frequent cross-role need, everything complex through the coordinator. |
| 7 | **Never hide failure** | No empty-as-success, no generic "failed", no killing the run on one timeout. Structured context and partial results. |

## Not tested

Out of scope by the official guide, and a trap for experienced builders whose expertise lives here: OAuth and authentication protocols · deploying or hosting MCP servers · rate limits, quotas and pricing · streaming and server-sent events · prompt-caching internals beyond knowing it exists · token counting · fine-tuning · embeddings and vector databases · vision · computer use · model benchmarking · cloud-provider specifics.

## Official sources

Everything here is built from, and checked against, Anthropic's own material. Each chapter ends with the official pages it draws on.

| | |
|---|---|
| The exam guide, task statements, official sample items and registration | [Claude Certified Architect – Foundations](https://anthropic.skilljar.com/claude-certified-architect-foundations) on the Anthropic Partner Academy |
| Anthropic Academy courses that cover the tested stack | [Claude Code in Action](https://anthropic.skilljar.com/claude-code-in-action) · [Introduction to Model Context Protocol](https://anthropic.skilljar.com/introduction-to-model-context-protocol) · [Model Context Protocol: Advanced Topics](https://anthropic.skilljar.com/model-context-protocol-advanced-topics) · [Building with the Claude API](https://anthropic.skilljar.com/claude-with-the-anthropic-api) |
| Product documentation | [Claude Code](https://code.claude.com/docs/en/overview) · [Claude Agent SDK](https://platform.claude.com/docs/en/agent-sdk/overview) · [Tool use](https://platform.claude.com/docs/en/agents-and-tools/tool-use/overview) · [Prompt engineering](https://platform.claude.com/docs/en/build-with-claude/prompt-engineering/overview) · [Model Context Protocol specification](https://modelcontextprotocol.io/specification/2025-06-18) |
| Anthropic engineering posts the blueprint leans on | [Building effective agents](https://www.anthropic.com/engineering/building-effective-agents) · [Writing tools for agents](https://www.anthropic.com/engineering/writing-tools-for-agents) · [Effective context engineering for AI agents](https://www.anthropic.com/engineering/effective-context-engineering-for-ai-agents) · [Claude Code best practices](https://www.anthropic.com/engineering/claude-code-best-practices) · [How we built our multi-agent research system](https://www.anthropic.com/engineering/multi-agent-research-system) |

Read the official guide first. Read this last.

## Maintenance

Verified against **Exam Guide v1.0 (July 2026)**. Every fact was checked against the official documentation linked at the end of each chapter. Judgment ages slowly; flag names and frontmatter keys do not, so the perishable material is isolated in [`reference/recall-facts.md`](reference/recall-facts.md) with its verification date. Corrections welcome as issues or pull requests.

## Changelog

- **v1.0 · September 2026** · Initial public release. Written during preparation for the exam and published whole after passing it; corrections and additions land here as they happen.

## License

[CC BY 4.0](LICENSE). Share it, adapt it, keep the attribution.

Written by [Keerthi Chand](https://www.linkedin.com/in/keerthichand/).
