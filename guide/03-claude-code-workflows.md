# Domain 3 · Claude Code Configuration & Workflows (20%)

Daily users of Claude Code know all of this by feel. The exam wants the mechanics named precisely: which scope, which frontmatter key, which flag.

## 3.1 CLAUDE.md hierarchy and modularity

| Level | Location | Shared with the team? |
|---|---|---|
| User | `~/.claude/CLAUDE.md` | **No.** Never reaches teammates through version control. This is the classic diagnosis when a new hire "doesn't get the rules." |
| Project | root `CLAUDE.md` or `.claude/CLAUDE.md` | Yes, version-controlled |
| Directory | `subdir/CLAUDE.md` | Yes, scoped to that subtree |

- **CLAUDE.md is concatenated, not resolved.** All discovered files are appended into context from broadest to most specific; nothing overrides anything, and the docs say that when two rules contradict, Claude may pick one arbitrarily. A rule that must hold every time (a blocked tool, a permission policy) belongs in `settings.json`, which *does* have strict precedence: managed policy > local > project > user. A stem that says "the project CLAUDE.md overrides the user one" is describing settings behavior, not memory behavior.
- **`CLAUDE.local.md`** sits beside `CLAUDE.md` at any level, loads after it, and is conventionally gitignored: a project-scoped version of the user file, for personal notes about this repo. A team rule that ends up there is in the wrong file.
- `@import` pulls external files in, keeping CLAUDE.md modular (per-package standards files).
- `.claude/rules/` splits a monolith into topic files (`testing.md`, `deployment.md`).
- `/memory` shows which memory files actually loaded: the diagnostic for behavior that differs between sessions. Current Claude Code adds `/context`, which reports context-window usage; the v1.0 exam guide predates the split, so **answer `/memory`** on the exam. Neither command loads anything; they reveal what is already loaded.
- **After `/compact`**, the project-root `CLAUDE.md` is re-read and restored. Subdirectory `CLAUDE.md` files and path-scoped rules are not restored automatically; they reload on demand when matching files are touched.

## 3.2 Commands versus skills

| Thing | Where | Key facts |
|---|---|---|
| Slash command, project | `.claude/commands/` | Version-controlled; the whole team gets it on pull |
| Slash command, personal | `~/.claude/commands/` | Yours only |
| Skill | `.claude/skills/<name>/SKILL.md` | Frontmatter: `context: fork` runs it in an isolated sub-agent so verbose output does not pollute the main session · `allowed-tools` restricts tools during the skill · `argument-hint` prompts for missing arguments. Personal variants live in `~/.claude/skills/` under a different name. |
| Skill versus CLAUDE.md | | Skill = on-demand, task-specific workflow. CLAUDE.md = always-loaded universal standards. Deciding between them is a recurring item. |

In current Claude Code the two are one system: `.claude/commands/deploy.md` and `.claude/skills/deploy/SKILL.md` both create `/deploy`, the frontmatter works on both, and skills are the recommended path (a directory for supporting files, automatic discovery). The exam still tests the *scope* distinction, project versus user, so keep both locations straight.

## 3.3 Path-specific rules

`.claude/rules/` files with YAML frontmatter `paths: ["terraform/**/*"]` load **only** when editing matching files: less irrelevant context, fewer tokens. They beat directory-level CLAUDE.md when a convention follows a file *type* scattered across the tree (`**/*.test.tsx` beside sources), and they beat "one big CLAUDE.md with headers" because glob matching is deterministic while section-relevance is inference.

## 3.4 Plan mode versus direct execution

- **Plan mode:** large-scale change, several valid approaches, architectural decisions, many files (a microservice split, a 45-file migration). Explore and design before committing; prevents rework.
- **Direct execution:** well-scoped, well-understood work (a single-file fix with a clear stack trace, one validation check).
- **Combine them:** plan the migration, then execute the planned approach directly.
- **Explore subagent:** isolates verbose discovery and returns summaries, protecting the main session's context during multi-phase work.

> **From production.** The gateway's concurrency overhaul was the combination pattern by the book: a written plan with sequencing rationale (lock shared state first, build a test harness, convert to async dispatch, then tune the infrastructure) produced before any edit, followed by four direct-execution changes against it. The same week, a two-line change to a response note with an obvious shape went straight to direct execution. Being able to say why those two got different treatment is the whole of 3.4.

## 3.5 Iterative refinement

- **Concrete input/output examples** beat prose when descriptions are interpreted inconsistently. Give two or three.
- **Test-driven iteration:** write the tests first, then iterate by feeding failures back.
- **The interview pattern:** have Claude ask questions first, surfacing considerations you had not anticipated (cache invalidation, failure modes). For unfamiliar domains.
- **Interacting fixes → one detailed message. Independent fixes → sequential.**

> **From production.** The gateway's maintainers mutation-test every guard: break the fix deliberately, confirm the test fails, restore. The habit exists because of a test that once passed forever while verifying nothing; a test that cannot fail is decoration. That is test-driven iteration in its most honest form, and it is a ready anchor for any 3.5 item about knowing when iteration has actually converged.

## 3.6 Claude Code in CI/CD

| Fact | Detail |
|---|---|
| `-p` / `--print` | Non-interactive mode. Without it the job hangs waiting for input (official sample). `CLAUDE_HEADLESS` and `--batch` do not exist. |
| `--output-format json` + `--json-schema` | Machine-parseable structured findings, ready to post as inline PR comments |
| CLAUDE.md in CI | How the pipeline-invoked Claude learns testing standards, fixtures and review criteria: better tests, fewer low-value findings |
| Session isolation | The session that *wrote* the code reviews it badly; it keeps its own reasoning context. Use an independent instance. |
| Re-review de-duplication | Feed prior findings back in; instruct "report only new or still-unaddressed issues" |
| Test generation | Provide the existing test files so it does not duplicate covered scenarios |

The flags around a headless run, beyond the three the appendix names. One distinction worth knowing cold: **`--system-prompt` replaces** the default system prompt, **`--append-system-prompt` appends** to it. Append to keep the default coding-assistant behaviour and layer your rules on top; replace only when you want none of the built-in tool guidance and conventions.

| Flag | Effect |
|---|---|
| `--system-prompt`, `--system-prompt-file` | Replace the default system prompt |
| `--append-system-prompt`, `--append-system-prompt-file` | Append to it |
| `--output-format text\|json\|stream-json` | How a `-p` run prints its result: plain text, a single JSON object, or a JSON event stream for programs that consume it |
| `--json-schema` | Schema-validated output under `-p`; lands in the JSON envelope's `structured_output` |
| `--max-turns` | Cap agentic turns, then exit |
| `--permission-mode` | Starting permission mode (`plan`, `acceptEdits`, and others) |
| `--allowedTools`, `--disallowedTools` | Allow without prompting / deny (a bare tool name removes it from context) |
| `--add-dir` | Grant read and edit access to another directory |
| `--model` | Session model |

## Quick checks

**QC7.** A convention must apply to every `*.sql` file, and those files sit in nine directories. Most maintainable mechanism?

A. A section in root CLAUDE.md titled "SQL conventions"
B. A skill the developer invokes before editing SQL
C. A `.claude/rules/` file with frontmatter `paths: ["**/*.sql"]`
D. A CLAUDE.md in each of the nine directories

<details><summary>Answer</summary>

**C.** Glob-scoped rule: deterministic loading, one file, follows the type anywhere. D is nine copies that drift; A relies on inference and always spends tokens; B requires remembering to invoke it.

</details>

**QC8.** Your codebase-analysis skill floods the main conversation with exploration output, degrading the rest of the session. Which frontmatter change fixes it?

A. `context: fork`, so it runs in an isolated sub-agent and returns only its result
B. `allowed-tools`, restricting it to Read and Grep
C. Move it to `~/.claude/skills/` so only you bear the cost
D. `argument-hint`, so it prompts for a narrower target

<details><summary>Answer</summary>

**A.** `context: fork` exists precisely to keep verbose skill output out of the main session. B limits capability, not verbosity; D and C do not address context pollution.

</details>

**QC9.** After generating a large refactor you ask the same session to review its changes for bugs; it approves its own work, and a colleague later finds two defects. What went wrong?

A. The generating session retains its reasoning context and will not question its own decisions; use an independent instance for review
B. Reviews require extended thinking to be effective
C. max_tokens was too low for a thorough review
D. The prompt should have demanded a confidence score per finding

<details><summary>Answer</summary>

**A.** Session context isolation. B: extended thinking does not remove the bias. D: self-reported confidence is a distrusted proxy.

</details>

## Official references

The pages this chapter draws on. Facts here were checked against them; when they disagree with any secondary source, including this one, they win.

- [Claude Code: overview](https://code.claude.com/docs/en/overview)
- [Claude Code: memory (CLAUDE.md hierarchy and imports)](https://code.claude.com/docs/en/memory)
- [Claude Code: settings and permissions](https://code.claude.com/docs/en/settings)
- [Claude Code: slash commands](https://code.claude.com/docs/en/slash-commands)
- [Claude Code: skills](https://code.claude.com/docs/en/skills)
- [Claude Code: hooks guide](https://code.claude.com/docs/en/hooks-guide)
- [Claude Code: common workflows (plan mode, extended thinking, resuming)](https://code.claude.com/docs/en/common-workflows)
- [Claude Code: headless mode](https://code.claude.com/docs/en/headless)
- [Claude Code: CLI reference](https://code.claude.com/docs/en/cli-reference)
- [Claude Code: GitHub Actions](https://code.claude.com/docs/en/github-actions)
- [Claude Code: best practices](https://code.claude.com/docs/en/best-practices)
- [Claude Code best practices (Anthropic engineering)](https://www.anthropic.com/engineering/claude-code-best-practices)
- [Equipping agents for the real world with Agent Skills (Anthropic engineering)](https://www.anthropic.com/engineering/equipping-agents-for-the-real-world-with-agent-skills)
- [Anthropic Academy: Claude Code in Action](https://anthropic.skilljar.com/claude-code-in-action)
