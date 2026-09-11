# Domain 2 · Tool Design & MCP Integration (18%)

> Case blocks marked **From production** refer to the system described in [the README](../README.md#about-the-production-system).

If you have shipped an MCP server, this is home ground. The exam tests the principles you learned by watching an agent misuse your tools, plus a handful of exact configuration facts to memorize.

## 2.1 Descriptions decide selection

Tool descriptions are the **primary** mechanism the model uses to choose among tools. Minimal descriptions ("Retrieves customer information" versus "Retrieves order details") cause misrouting between similar tools. A description that works carries: purpose, input formats, example queries, edge cases, and **boundaries**, meaning when to use this tool rather than its neighbours.

Three subtleties the exam probes:

- **Descriptions before few-shot.** The official sample ranks "expand the descriptions" as the correct *first* step, ahead of few-shot examples (token overhead, does not fix the root), a routing layer (over-engineered), or merging the tools (bigger than a first step).
- **The system prompt can override good descriptions.** An imperative that contains tool-name-shaped keywords ("always analyze documents thoroughly") drags the model to `analyze_document` for jobs another tool should handle. When descriptions are already good and selection is still wrong, audit the standing instructions.
- **Genuine overlap is a naming problem.** Rename and re-scope (`analyze_content` → `extract_web_results`), or split one generic tool into purpose-specific tools with defined contracts.

> **From production.** The sharpest version of this lesson: thirty-three carefully written parameter descriptions on the gateway's tools were *silently discarded* for weeks, because the framework read descriptions from one attribute and they had been written to another. The dropped guidance included the exact instruction that stops an agent from running an entire dashboard when it needs one tile. Descriptions are not documentation; they are the routing layer, and a routing layer that is silently empty produces confident, wrong behavior.

## 2.2 Structured error responses

- MCP signals tool failure with the **`isError`** flag; the payload should still carry information.
- **Taxonomy to recite:** transient (timeout, service down) · validation (bad input) · business (policy violation) · permission. Return `errorCategory`, `isRetryable`, and a human-readable message. Business errors get `retriable: false` plus a customer-friendly explanation.
- A generic "operation failed" prevents any intelligent recovery; structured metadata prevents wasted retries.
- **Empty is not an error.** A successful query with zero matches is a valid result. Conflate the two and the agent either retries forever or tells the user the data does not exist.
- Subagents recover locally from transient failures and propagate upward only what they cannot fix, *with partial results and what was attempted*.

> **From production.** The gateway appended "the name or ID may be wrong" to every not-found error. Measured against an admin's view, that message was wrong about three quarters of the time: the resource usually existed and the caller lacked access. Agents responded by inventing nearby names, one after another. The fix was a structured reason on every not-found error (no access, wrong parent, genuinely absent), which is `errorCategory` by another name. The same system's timeout message tells the agent what to narrow and says explicitly *do not retry unchanged*, which is `isRetryable: false` with instructions attached.

## 2.3 Distribution and `tool_choice`

- **Too many tools degrades selection.** Around eighteen tools on one agent produces unreliable routing; four or five role-scoped tools is the healthy shape.
- **The scoped-exception pattern.** A synthesis agent that needs simple fact checks 85% of the time gets one narrow `verify_fact` tool; the complex 15% still routes through the coordinator. The official sample rejects batching the checks (creates blocking dependencies), giving the agent all search tools (over-provisioning), and speculative caching (cannot predict need).
- **Constrain generic tools.** Replace `fetch_url` with `load_document` that validates its inputs.

| `tool_choice` | Behavior | Reach for it when |
|---|---|---|
| `"auto"` | Model may call a tool or answer in text | Normal agentic operation |
| `"any"` | Model **must** call some tool, its choice | Guaranteed structured output when several schemas exist and the document type is unknown |
| `{"type":"tool","name":"…"}` | Model **must** call that specific tool | Force a specific step first (extract metadata before enrichment) |

## 2.4 MCP configuration facts

| Fact | Detail |
|---|---|
| Project scope | `.mcp.json` in the repository: shared team tooling, version-controlled |
| User scope | `~/.claude.json`: personal and experimental servers |
| Credentials | `${GITHUB_TOKEN}`-style environment expansion in `.mcp.json`; secrets never committed |
| Discovery | Tools from **all** configured servers are discovered at connection time and available simultaneously |
| MCP resources | Expose content catalogs (schemas, issue lists, documentation trees) so agents see what exists **without exploratory tool calls** |
| Build versus adopt | Community servers for standard integrations (Jira); custom servers only for team-specific workflows |
| Description power | Rich MCP tool descriptions also stop the agent from preferring a built-in (Grep) over your more capable tool |

> **From production.** The gateway ships a curated catalog (recommended dashboards, an index of sanctioned data sources, data notes) through an access tool, precisely so agents stop making exploratory calls and stop inventing names. That is the job MCP *resources* exist for. On the exam, "reduce exploratory tool calls for content the agent just needs to see" is resources, not another tool.

## 2.5 Built-in tools

| Tool | Select when |
|---|---|
| Grep | Searching file **contents**: callers of a function, an error string, imports. Scopable with a glob filter. |
| Glob | Finding files by **path or name pattern**: `**/*.test.tsx` |
| Read / Write | Full file load and full overwrite; the reliable fallback pair |
| Edit | A targeted change anchored on unique text. When the anchor is not unique, fall back to Read + Write. |
| Bash | Executing: tests, builds, git. Never to re-implement Grep, Glob or Read. |

Two named skills:

- **Incremental understanding.** Grep for an entry point, Read to follow imports, repeat along the actual path. Never "read all files upfront"; it exhausts context on files that turn out irrelevant.
- **Wrapper tracing.** An imported function has more than one address. To find every caller of something re-exported through a wrapper module, first enumerate the wrapper's exported names, then search for each name across the codebase.

The rule that generalizes: **Grep when the target set is defined by content; Glob when it is defined by name or path.**

**The deprecation scenario** (one of the twelve official sample items) uses both. To retire a function you need every caller *and* the tests that exercise them: **Grep** for the function name (content: direct callers and any tests importing it), then **Glob** for sibling test files by naming convention (`OrderProcessor.ts` → `**/OrderProcessor.test.*`, a name-defined set), then a second **Grep** pass over each name exported by a wrapper module, because a caller may reach the function through a re-exported alias. Grep, Glob, Grep. Glob never leads, because the callers are defined by content, but it does earn its place once the set you want is defined by a filename pattern.

## Quick checks

**QC4.** A subagent's document query returns zero rows because no documents match. The tool returns `isError: true`, "search failed." Consequence and fix?

A. Suppress the error and return the most similar documents instead
B. Terminate the workflow, since research cannot proceed without documents
C. The coordinator will treat a valid outcome as a failure and waste retries; return success with an empty set and a note distinguishing "no matches" from an access failure
D. Correct as is; empty results indicate the query failed to find anything

<details><summary>Answer</summary>

**C.** Valid-empty versus access-failure is an explicit task statement. D mislabels success as error; A fabricates relevance; B is the kill-the-workflow anti-pattern.

</details>

**QC5.** Your team's shared Jira MCP server needs an API token, and the configuration must be safe to commit. Where does it go?

A. `~/.claude.json` with the token inline, per developer
B. `.mcp.json` in the repository with `"${JIRA_TOKEN}"` environment-variable expansion
C. CLAUDE.md, so the token loads into every session's context
D. `.claude/rules/jira.md` with paths frontmatter

<details><summary>Answer</summary>

**B.** Shared team tooling is project-scoped; secrets stay out via expansion. A makes it personal rather than shared; C leaks a secret into context and is not configuration; D is convention loading, not server configuration.

</details>

**QC6.** Logs show your document-analysis subagent occasionally attempts web searches, and tool selection got flakier after you granted every subagent the full eighteen-tool set "for flexibility." Best correction?

A. Set `tool_choice: "any"` so the model always calls some tool
B. Prefix each tool description with the intended agent's name
C. Add few-shot examples of correct tool selection for all eighteen tools
D. Restrict each subagent to its role's tools; where one cross-role need is frequent, add a single scoped tool for it

<details><summary>Answer</summary>

**D.** Scoped access plus the sanctioned scoped-exception pattern. C treats symptoms with token overhead; A makes it worse by forcing calls; B leaves the oversized decision space intact.

</details>

## Official references

The pages this chapter draws on. Facts here were checked against them; when they disagree with any secondary source, including this one, they win.

- [Writing tools for agents (Anthropic engineering)](https://www.anthropic.com/engineering/writing-tools-for-agents)
- [Tool use: implementing tool use, descriptions and `tool_choice`](https://platform.claude.com/docs/en/agents-and-tools/tool-use/implement-tool-use)
- [Model Context Protocol: server concepts (tools, resources, prompts)](https://modelcontextprotocol.io/docs/learn/server-concepts)
- [Model Context Protocol: specification](https://modelcontextprotocol.io/specification/2025-06-18)
- [Claude Code: connecting MCP servers (scopes, `.mcp.json`, environment expansion)](https://code.claude.com/docs/en/mcp)
- [MCP connector for the Claude API](https://platform.claude.com/docs/en/agents-and-tools/mcp-connector)
- [Claude Agent SDK: custom tools](https://platform.claude.com/docs/en/agent-sdk/custom-tools)
- [Code execution with MCP (Anthropic engineering)](https://www.anthropic.com/engineering/code-execution-with-mcp)
- [Anthropic Academy: Introduction to Model Context Protocol](https://anthropic.skilljar.com/introduction-to-model-context-protocol)
- [Anthropic Academy: Model Context Protocol – Advanced Topics](https://anthropic.skilljar.com/model-context-protocol-advanced-topics)
