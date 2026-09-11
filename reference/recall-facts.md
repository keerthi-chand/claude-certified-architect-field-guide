# Recall facts

The exact facts: flags, keys, limits, defaults. This is the one perishable file in the repository, so it carries a date. Judgment ages slowly; names and defaults do not.

**Verified against Exam Guide v1.0 (July 2026); tooling facts checked against the Claude Code documentation on 2026-09-11.** If a fact here has moved, open an issue.

| Topic | Recall |
|---|---|
| Agentic loop | Continue on `stop_reason = "tool_use"` · stop on `"end_turn"` · append tool results to history before the next call · anti-patterns: parsing natural language for "done," an iteration cap as the primary stop, text presence as completion |
| Spawning | `Task` tool spawns subagents (renamed `Agent` in current Claude Code, `Task` still works; the exam says `Task`) · coordinator's `allowedTools` must include `"Task"` · no automatic context inheritance · parallel = multiple Task calls in **one** response · `AgentDefinition` holds description, system prompt, tool restrictions |
| Hooks | PostToolUse transforms *results* (normalize formats) · **PreToolUse** is the interception hook: blocks *outgoing* calls (`allow` / `deny` / `ask`, optional `updatedInput`) · SubagentStart observes spawns, SubagentStop can block completion (exit 2) · hooks give guarantees, prompts give guidance |
| Sessions | `--resume <name>` · `fork_session` branches off a baseline · stale results → fresh session plus injected summary |
| `tool_choice` | `"auto"` may answer in text · `"any"` must call some tool · `{"type":"tool","name":…}` must call that one |
| Error taxonomy | transient / validation / business / permission · MCP `isError` flag · `errorCategory`, `isRetryable`, readable message · business rules: `retriable: false` plus customer-friendly reason · valid empty ≠ failure · propagate partial results |
| Tool distribution | Around eighteen tools degrades selection; four or five role-scoped is healthy · one scoped tool for a frequent cross-role need · constrain generics (`fetch_url` → `load_document`) |
| MCP configuration | project `.mcp.json` (shared, versioned) · user `~/.claude.json` (personal) · `${VAR}` expansion for secrets · all servers' tools discovered at connection, available simultaneously · resources expose content catalogs · community servers for standard integrations |
| Built-in tools | Grep = contents (scopable by glob) · Glob = paths and names · Edit = unique anchor, otherwise Read + Write · Bash = execute, never to re-do search or read · explore incrementally: Grep an entry point, Read to follow imports |
| CLAUDE.md | user `~/.claude/CLAUDE.md` (**not shared**) · project root or `.claude/CLAUDE.md` (shared) · directory-scoped · `CLAUDE.local.md` beside any of them, gitignored, personal · files are **concatenated** broadest→specific, never overriding (guarantees belong in `settings.json`, which has strict precedence) · `@import` · `.claude/rules/` · `/memory` shows what loaded (`/context` shows usage; exam answer is `/memory`) · after `/compact` root CLAUDE.md is restored, nested files and path rules reload on demand |
| Rules | `.claude/rules/` files with YAML `paths:` globs load only when editing matching files |
| Commands | project `.claude/commands/` (versioned) · personal `~/.claude/commands/` |
| Skills | `.claude/skills/<name>/SKILL.md` · frontmatter `context: fork`, `allowed-tools`, `argument-hint` · personal variants in `~/.claude/skills/` · skill = on-demand, CLAUDE.md = always loaded |
| Plan mode | architectural, several approaches, many files → plan · scoped single-file fix → direct · combine: plan then execute · Explore subagent isolates discovery |
| CLI in CI | `-p` / `--print` non-interactive · `--output-format text\|json\|stream-json` · `--json-schema` (→ `structured_output`) · `--append-system-prompt` appends, `--system-prompt` replaces · `--max-turns` · `--allowedTools` / `--disallowedTools` · `--permission-mode` · `--add-dir` · CLAUDE.md carries standards into CI · independent instance reviews, the generator never self-reviews · feed prior findings to de-duplicate · provide existing tests |
| Structured output | `tool_use` plus JSON schema eliminates syntax errors; semantic errors remain · nullable beats fabrication · enums with `"unclear"` and `"other"` plus detail · normalization rules in the prompt · retry fixes format, never absence |
| Validation | Pydantic or any validator runs *after* the model replies · semantic rules a schema cannot express (sums, cross-field consistency) · retry with the original document, the failed extraction, and the specific error · `detected_pattern`, `calculated_total`, `conflict_detected` |
| Batches | 50% cheaper · up to 24 hours · no latency SLA · `custom_id` correlates and identifies failures to resubmit · no multi-turn tool calling inside a request · never for blocking gates · cadence plus 24 hours must fit the promised SLA · refine on a sample first |
| Context | case-facts block for exact values · key findings first (lost in the middle) · trim verbose tool outputs · scratchpad files and state manifests for crash recovery · `/compact` |
| Escalation | explicit request for a human → immediately · policy gap or silence → escalate · no progress → escalate · sentiment and self-confidence are unreliable · multiple matches → ask for another identifier |
| Human review | segment accuracy by document type and field · stratified random sampling of high-confidence output · confidence usable only after calibration on labeled data |
| Provenance | claim-source mappings preserved through synthesis · conflicts annotated with attribution and dates, never averaged · render financial data as tables, news as prose, technical findings as lists |
| The exam | 60 items · 120 minutes · 4 scenarios from a bank of 6 · pass 720 of 1000 · D1 27% · D2 18% · D3 20% · D4 20% · D5 15% · valid 12 months · renewal is a free non-proctored assessment |
