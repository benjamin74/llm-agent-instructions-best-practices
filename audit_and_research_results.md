# Agent instruction files: expert best practices and audit reference
 
**Scope:** `CLAUDE.md`, `AGENTS.md`, supporting docs, path-scoped rules, skills, hooks and permissions
**State of practice as of:** 26 November 2026
**Audience:** (1) a human maintainer, and (2) a coding agent that has to audit a repository against this document
 
---
 
## 0. Instructions for the auditing agent (read first)
 
You are auditing the repository you are running in against the expert practices in this document.
 
1. **Inventory first, opinions second.** List every instruction surface that exists:
   - root and nested `CLAUDE.md`, `CLAUDE.local.md`, `.claude/CLAUDE.md`, `AGENTS.md`, `AGENTS.override.md`
   - `.claude/rules/**`, `.cursor/rules/**`, `.github/copilot-instructions.md`
   - `.claude/skills/**`, `.agents/skills/**`, `.claude/commands/**`, `.claude/agents/**`
   - `.claude/settings.json` (permissions, hooks), `.mcp.json`
   - any `docs/`, `agent_docs/` or `ai/` folder that the instruction files point to
   - CI config, linters, formatters, architecture tests, and git hooks, because those are the enforcement layer
   Also report on each file: its size (lines, `wc -m`), whether it's loaded always or on demand, and who it applies to.
2. **Check what actually loads.** Note which tool or tools the repo targets (Claude Code, Codex, Cursor, Copilot, etc.). Use §2.3 to work out what each tool will actually read. For example, Claude Code ignores `AGENTS.md` whenever a `CLAUDE.md` exists, unless `CLAUDE.md` imports it.
3. **Classify every instruction** in the always-loaded files with the placement decision tree in §6. For each line, say whether to keep it, move it (and where), delete it, or turn it into enforcement (a hook, permission, CI check or test).
4. **Run the checklist in §10** and give each item one of these statuses: PASS / WARN / FAIL / N/A.
5. **Output:** use the report format in §10.3. Rank findings by impact. Propose concrete diffs or new file contents. **Don't make the changes** unless the user asks you to.
6. **Ground every claim in the repo.** Quote the offending line with its file:line reference. Don't suggest documenting something the agent can already discover from the code. That's an anti-pattern here (§3, consensus C3).
7. **Verify version-specific behaviour** (file precedence, size limits, frontmatter fields) against the installed tool's docs or `--help`. Tooling changes quickly, and the facts in §2.3 were correct on the date above.
---
 
## 1. TL;DR: the twelve rules most experts converge on
 
1. **The root file is a briefing, not a manual.** Keep it short, hand-written, and limited to what is universally relevant. Common targets are under 60 lines (HumanLayer, the Ralph-loop practitioners), about 100 lines (OpenAI's harness team), and under 200 lines (Anthropic's documentation ceiling).
2. **Only write what the agent can't discover by reading the repo,** or would waste time rediscovering every session. Examples: exact commands, misleading code, team decisions, environment facts, and anything you've had to correct twice.
3. **Commands are the most valuable content.** That means exact build, test, lint and typecheck commands, including how to run a *single* test file, and a definition of "done".
4. **Use progressive disclosure.** The root file routes to deeper sources: path-scoped rules, nested files, `docs/`, and skills. Use pointers, not copies. Say *when and why* the agent should read each target.
5. **Anything that must always (or never) happen gets enforced, not requested.** Use hooks, permission deny rules, CI, tests, OS permissions and scoped credentials. Instruction files are advisory. The model may decide a line doesn't apply.
6. **Don't use the LLM as a linter.** Formatting, naming and style that a tool can check belong in a formatter, linter or architecture test, with at most a one-line pointer.
7. **Turn procedures into skills.** Multi-step workflows such as release, deploy, migrations and incident runbooks become skills. Side-effecting ones must be **user-invoked only**.
8. **Don't count on auto-triggered skills for critical knowledge.** Measured trigger rates are unreliable. Put an always-present index or trigger in the root file, use path-scoped loading, or use hook-based activation.
9. **Avoid auto-generated context files** (such as raw `/init` output kept as-is). Studies find they add cost without improving success. Generate a draft if you like, then cut it down hard.
10. **Every "never" needs an "instead".** A prohibition with no alternative leaves the agent stuck or improvising.
11. **Ratchet on observed failures, and prune on a schedule.** Add a line (or a tool) only when the agent actually made a mistake. Delete lines that newer models no longer need, and fix stale paths.
12. **Pick one source of truth across tools.** Make `AGENTS.md` canonical, and give `CLAUDE.md` an `@AGENTS.md` import plus Claude-only extras (or the reverse). Don't keep two hand-edited copies that drift apart.
---
 
## 2. Mental model
 
### 2.1 The instruction surfaces, by how reliably they reach the model
 
| Layer | Example locations | When it enters context | Guarantee | Cost |
|---|---|---|---|---|
| **Always-loaded root file** | `AGENTS.md`, `CLAUDE.md`, `~/.claude/CLAUDE.md`, managed `/etc/claude-code/CLAUDE.md` | Every session; re-injected after compaction (Claude Code, project root) | Always present, but attention is diluted and following it is still the model's choice | Paid on every turn of every task |
| **Directory- or path-scoped instructions** | Nested `AGENTS.md`/`CLAUDE.md`; `.claude/rules/*.md` with `paths:`; Cursor `.mdc` with `globs` | Deterministically, when the agent reads or edits matching files | Loading is deterministic; following is not | Paid only when working in that area |
| **Hook-injected context** | `SessionStart` / `UserPromptSubmit` hooks that print context | Deterministically, at that event | Loading is deterministic | Small; you control it |
| **User-invoked skill / command** | `/deploy-prod` (`disable-model-invocation: true`) | When a human types it | The procedure is loaded every time it's invoked | Zero until invoked |
| **Pointer to a supporting doc** | "Before touching billing, read `docs/billing.md`" | When the model decides to follow the pointer | Model-dependent; much better with an explicit trigger | Zero until read |
| **Auto-invoked skill** | `.claude/skills/x/SKILL.md`, `.agents/skills/x/SKILL.md` | When the model matches the request to the description | Model-dependent; measured as unreliable (§4.2) | Description is always present; body on demand |
| **Enforcement** | Permission deny rules, `PreToolUse`/`Stop` hooks, CI, tests, linters, architecture tests, OS file permissions, scoped DB/cloud credentials | Not context at all: it acts on the agent's actions | Deterministic (the harness or OS decides, not the model) | Maintenance only |
 
**Key principle (Anthropic's docs, echoed by practitioners).** Every instruction is either *non-deterministic* (a prompt, `CLAUDE.md` or a skill, which the model may decide not to follow) or *deterministic* (a hook, deny rule or managed setting, which the harness enforces whatever the model decides). If you keep adding emphasis to a line and the agent still misses it, the line belongs in a hook. The reverse also holds: a hook for a mere preference should be deleted.
 
### 2.2 Why "more instructions" backfires
 
- **Instruction-density limits.** The IFScale benchmark (Distyl AI, 2025) found that even the best frontier models reach only about 68% accuracy at 500 simultaneous instructions. It also found a bias towards earlier instructions. HumanLayer turns this into a working budget: frontier models follow roughly 150–200 instructions reliably, and the harness's own system prompt already uses a chunk of that.
- **Relevance gating.** Claude Code delivers `CLAUDE.md` content together with a reminder that the context "may or may not be relevant". Content that is clearly irrelevant to the current task is therefore easy for the model to set aside. The more non-universal content you add, the more of it gets discounted (HumanLayer).
- **Context rot and the "dumb zone".** Recall and reasoning degrade as context fills (Anthropic, "context rot"). Dex Horthy (HumanLayer) puts the start of degradation at roughly 40% of the window for complex work.
- **Everything marked important means nothing is.** OpenAI's harness team gave four reasons their giant `AGENTS.md` failed: context is scarce; too much guidance becomes non-guidance; it rots quickly; and a single blob can't be checked mechanically.
### 2.3 Tool facts that matter for audits (verify against current docs)
 
**Claude Code** (memory docs, Nov 2026)
- **Files and precedence.** It loads managed, user (`~/.claude/CLAUDE.md` and `~/.claude/rules/`), project (`./CLAUDE.md` or `./.claude/CLAUDE.md`) and local (`./CLAUDE.local.md`, which should be gitignored) files. These are concatenated, not overridden. Files closer to the launch directory are read last.
- **Subdirectory files load lazily.** A `CLAUDE.md` in a subdirectory loads only when Claude reads, writes or edits files in that subdirectory.
- **Imports.** The `@path` syntax supports up to four levels of nested import. Imports organise content but **don't reduce context cost**: an imported file loads together with its parent.
- **Rules.** `.claude/rules/*.md` files without `paths:` load at launch. With `paths:` globs, they load only when a matching file is touched.
- **Size.** The documented target is under 200 lines per `CLAUDE.md`. On-demand files are truncated past a per-file character limit: 40,000 characters with a 200K context window.
- **HTML comments** (`<!-- -->`) are stripped before injection, so maintainer notes cost nothing.
- **`AGENTS.md` support (v2.1.277+).** Claude Code reads `AGENTS.md` only when **no** `CLAUDE.md`, `.claude/CLAUDE.md` or `CLAUDE.local.md` exists in the working directory or any ancestor. The recommended way to share one file is a `CLAUDE.md` that contains `@AGENTS.md` followed by any Claude-only additions. A symlink also works, but the Edit and Write tools won't write through it, and Windows clones may check it out as a plain text file.
- **Audit and debug tools.** `/context all` shows what loaded. `/doctor prompt-audit` finds stale references, conflicts and instructions written for older models. The `InstructionsLoaded` hook logs what loaded and why.
- **Auto memory** is machine-local (`~/.claude/projects/<project>/memory/MEMORY.md`, first 200 lines or 25 KB). It is separate from the repo and not shared with the team.
**Claude Code skills**
- A skill is `.claude/skills/<name>/SKILL.md`. The description (plus `when_to_use`) is always in context, truncated at 1,536 characters. The body loads when the skill is invoked; supporting files load only when referenced; scripts run without being loaded.
- Keep `SKILL.md` under 500 lines.
- `disable-model-invocation: true` means only a human can invoke the skill, and its description is not kept in context. `user-invocable: false` hides it from the `/` menu. Other fields include `paths:` (limits auto-activation), `allowed-tools`, `context: fork` and `!`command`` (dynamic context injection).
- After compaction, the most recently invoked skills are re-attached: up to the first 5,000 tokens of each, and 25,000 tokens in total. Older skills can drop out of context entirely.
**Codex** (OpenAI docs)
- Codex walks from the git root down to the current working directory, taking at most one file per directory. In each directory it checks `AGENTS.override.md`, then `AGENTS.md`, then any configured fallback names.
- Files are concatenated root first, so closer files take precedence.
- There's a combined cap: `project_doc_max_bytes` defaults to **32 KiB**, and files beyond it are not added.
- Skills live in `.agents/skills/`, which Codex scans from the working directory up to the repo root.
**AGENTS.md (open format, stewarded by the Agentic AI Foundation under the Linux Foundation)**
- It's plain Markdown with no required sections.
- The nearest file wins, and explicit user prompts override everything.
- It is read natively by Codex, Cursor, Copilot's coding agent, Gemini CLI (with configuration), Jules, Amp, Zed, Warp, Junie and others.
**Agent Skills (open standard, agentskills.io, December 2025)**
- The same `SKILL.md` format is read by Claude Code, Codex, Cursor, Copilot, Gemini CLI and others. Locations differ by tool: `.claude/skills/`, `.agents/skills/`, `.github/skills/`.
---
 
## 3. What there is clear consensus on
 
These positions are shared by the vendors (Anthropic, OpenAI, GitHub, the AAIF), the empirical studies, *and* the independent practitioners. If a repo violates one of them, report it as a FAIL.
 
| # | Consensus position | Who holds it (examples) |
|---|---|---|
| C1 | **Keep the always-loaded file short and universally applicable.** Move task-specific material elsewhere. | Anthropic docs (<200 lines), HumanLayer (<60), OpenAI harness (about 100, "map, not manual"), Huntley/Ralph playbooks (about 60), AAIF, ETH study |
| C2 | **Exact, runnable commands** for build, test, lint and typecheck are the highest-value content. Put them near the top, and include the single-test or single-file variant. | GitHub 2,500-repo analysis, the AGENTS.md spec, Anthropic ("the single most useful line"), AAIF, HumanLayer |
| C3 | **Don't document what the agent can discover.** Generic overviews, file tours, language defaults and "write clean code" platitudes cost tokens and don't help navigation. | ETH Zurich (overviews didn't reduce steps to relevant files), Anthropic ("stop describing your architecture and layout"), AAIF exclusion list, GitHub (vague files fail) |
| C4 | **Deterministic enforcement for must-never and must-always rules** (hooks, deny rules, CI, tests). Instructions are guidance, not controls. | Anthropic best practices, HumanLayer, Shrivu Shankar (block-at-submit hooks), OpenAI harness (custom linters), Hashimoto ("programmed tools") |
| C5 | **Don't use the model as a linter or formatter.** Run the tool, and feed its errors back to the agent. | HumanLayer, AAIF ("rules a formatter, linter or CI check can enforce"), OpenAI harness, Codex docs ("leave formatting and lint checks to CI") |
| C6 | **Progressive disclosure.** The root file points to deeper documents, path-scoped rules or skills, loaded only when relevant. | Anthropic (rules and skills), HumanLayer (`agent_docs/`), OpenAI harness (`docs/` as the system of record), Vercel (compressed index), Steinberger (`docs/` folder) |
| C7 | **Iterate from real failures.** Start minimal and add a line, or build a tool, each time the agent repeats a mistake. | Boris Cherny/Anthropic team ("anytime Claude does something incorrectly we add it"), Mitchell Hashimoto ("engineer the harness"), GitHub, Shrivu ("guardrails, not a manual") |
| C8 | **Specific beats vague.** "Run `npm test` before committing" works; "test your changes" doesn't. Name versions, paths and commands. | Anthropic memory docs, GitHub (exact stack with versions), AAIF |
| C9 | **Give the agent a way to verify its own work,** such as a test, a typecheck, a script that diffs output, or a screenshot. | Anthropic ("give Claude a check to run"), Hashimoto, Huntley ("backpressure"), the AGENTS.md spec (agents run the listed checks) |
| C10 | **Commit the shared instruction file to git and maintain it as a team.** Personal preferences go in a local or user-scope file. | Boris Cherny, Anthropic docs (`CLAUDE.local.md`), Codex docs (global vs. project) |
| C11 | **Contradictions and stale lines are actively harmful.** Review and prune regularly. | Anthropic (`/doctor prompt-audit`; conflicts get "resolved arbitrarily"), OpenAI ("rots quickly"), AAIF, Steinberger (lean beats half-wrong) |
| C12 | **Secrets and credentials never go in instruction files,** and agents shouldn't hold credentials that can destroy production. | GitHub ("never commit secrets" was the most common helpful constraint), AAIF, post-mortems of the PocketOS (April 2026) and Replit (July 2025) incidents |
 
---
 
## 4. What most experts agree on (with notable dissent)
 
### 4.1 Majority positions
 
| # | Majority position | Dissent or nuance |
|---|---|---|
| M1 | **Write it by hand, or heavily edit a generated draft.** The ETH study (ICLR 2026 workshop) found LLM-generated context files *slightly reduced* success and raised cost by about 20%. Developer-written files did a bit better (about +2.4%, not statistically significant). HumanLayer says to avoid `/init`. | Anthropic's docs say to use `/init` as a start and then "mostly remove lines". Steinberger and Ronacher let the agent write and maintain docs and tools. **Reconciled view:** a generated draft is fine *as a draft*, but every surviving line must pass the "would it change the agent's behaviour?" test. |
| M2 | **Pointers, not copies.** Avoid code snippets that go stale; refer to `path/file.php` (or `file:line`) and to exemplar files. | GitHub found that one real code example of the desired output beats paragraphs of prose. **Reconciled view:** point to a *living exemplar file* in the repo rather than pasting snippets into the instruction file. |
| M3 | **Use a single canonical cross-tool file (`AGENTS.md`)** with thin tool-specific shims. | Steinberger notes that GPT and Claude models respond best to different prompting, so one shared file is a compromise. Keep model-specific tweaks in the shim. |
| M4 | **Skills are for procedures; root and docs are for facts.** A section that has grown into a procedure becomes a skill. | Vercel's evals favour passive context for *framework knowledge* (§4.2). Skills still win for "vertical, action-specific workflows users explicitly trigger". |
| M5 | **Use CLIs and scripts over MCP where possible.** Models already know common CLIs; MCP tool definitions cost context on every turn. | Shrivu, Ronacher, Steinberger and Willison all prefer CLIs or skills. MCP stays useful as a *secure gateway* (Shrivu) and for systems with no CLI (Anthropic). Example: Laravel Boost's DB and schema tools. |
| M6 | **Plain wording, no ALL-CAPS shouting.** Anthropic now says piling on emphasis makes Claude *over-apply* a rule. Steinberger says GPT-5 doesn't respond to caps: "use words like a human". | Some Claude users still find emphasis helpful for rare critical lines (Gonsalvez). **Reconciled view:** if a line needs shouting, it probably needs a hook instead. |
| M7 | **Describe structure only where its meaning isn't obvious.** Say which package owns what, whether apps deploy independently, and which directory is legacy-but-live. Skip file tours. | HumanLayer's original WHAT/WHY/HOW included a codebase map (useful in monorepos). The ETH study and Anthropic's 2026 docs say generic overviews don't help. **Reconciled view:** keep the "map of meaning" (AAIF), drop the "tour". |
| M8 | **Nested or path-scoped files for monorepos and distinct areas.** The closest file wins. | Both approaches are fine. Nested `AGENTS.md` files are portable across tools; `.claude/rules` with `paths:` is Claude-specific but doesn't clutter directories. |
 
### 4.2 The evidence on skills vs. always-present context (important for "will it be followed?")
 
- **Vercel's Next.js 16 evals (January 2026).** The baseline with no docs passed 53% of tasks. Adding a skill with default behaviour still passed 53%. A skill plus explicit instructions to use it passed 79%. A **compressed docs index in `AGENTS.md`** (40 KB cut to 8 KB, pointing at doc files) passed **100%**. In 56% of cases the skill was never invoked at all. Results were also very sensitive to wording: "invoke the skill first" made the agent skip exploring the project.
- **Community reports.** Many practitioners report unreliable automatic triggering and use `UserPromptSubmit` hooks that match the prompt against keyword rules and inject "use skill X". Tools include the `skill-rules.json` pattern and forced-evaluation hooks.
- **Anthropic's guidance on descriptions.** The description is the main selection signal. It should say *what the skill does and when to use it*, in the third person, with trigger terms. Vague descriptions fail.
- **Takeaway for audits.** If the agent *must* know something whenever it works in an area, don't rely on auto-triggered skills alone. Use one of these instead:
  1. a one-line trigger in the root file ("When editing `app/Billing/**`, first read `docs/billing.md`"),
  2. a path-scoped rule or nested file (deterministic loading), or
  3. a hook that injects context.
  Keep auto-triggered skills for *helpful but non-critical* know-how, and user-invoked skills for procedures.
### 4.3 Empirical results in one place
 
| Study | Finding | Practical implication |
|---|---|---|
| Gloaguen et al., ETH Zurich, "Evaluating AGENTS.md" (arXiv 2602.11988, Feb 2026) | LLM-generated files: about −0.5 to −2% success, +20–23% cost. Developer-written files: about +2.4% (not significant), still more steps. Agents *do* follow instructions: `uv` was used about 1.6 times per task when the file mentioned it, versus under 0.01 when it didn't. Overviews don't speed up finding relevant files. | Keep files minimal. Include only non-obvious requirements and tooling. Every extra requirement makes tasks harder. |
| Vercel (Jan 2026) | Passive indexed context gave 100% vs. 53–79% for skills | Put critical knowledge indexes in the always-loaded file; compress them aggressively. |
| AAIF (Aug 2026), citing Lulla et al. | Across 124 PR tasks, `AGENTS.md` cut median runtime by about 29% and output tokens by about 17%. In AAIF's own 5-run test, no-file runs sometimes wandered or ran unrequested production builds. | A short, curated file improves efficiency even where correctness doesn't change. |
| GitHub (Matt Nigh, 2,500+ files) | The best files cover commands, testing, structure, style (by example), git workflow and boundaries (always / ask first / never). Vague files fail. | Use these six areas as a coverage checklist, not a template to fill. |
| IFScale (Distyl, NeurIPS 2025) | Instruction-following accuracy falls as density rises (68% at 500 instructions), with a bias towards earlier instructions | Keep the instruction count low and put the most important items first. |
 
---
 
## 5. Opinionated techniques (pick what fits your style)
 
Each technique below is a credible, used-in-anger approach. **Pros and cons are relative to the "generic" approach:** a short root file plus path rules plus a few skills plus hooks.
 
### 5.1 "Ultra-minimal root" (HumanLayer; Huntley/Ralph practitioners)
- **What:** the root file is under about 60 lines, covering WHAT/WHY/HOW only. Everything else lives in `agent_docs/*.md`, listed with one-line descriptions. The agent is told to read the relevant ones (or ask before reading). In Ralph loops, `AGENTS.md` is purely an "operational guide": how to build, test and validate, in about 60 lines.
- **Pros:** maximum attention on what matters; cheap; works the same across tools.
- **Cons:** relies on the model choosing to open the docs. Add explicit "read X before Y" triggers.
- **Fits:** most repos, especially with smaller or cheaper models and long autonomous runs.
### 5.2 "Table of contents + docs/ as the system of record" (OpenAI harness engineering, Feb 2026)
- **What:** about 100 lines of `AGENTS.md` serve as a map. A structured `docs/` holds `ARCHITECTURE.md`, `SECURITY.md`, `RELIABILITY.md`, `design-docs/`, `exec-plans/active|completed/`, `generated/db-schema`, and so on. **CI linters check that docs are cross-linked and fresh.** A recurring "doc-gardening" agent opens fix-up PRs when docs drift from the code. Custom lint error messages include *remediation instructions*, so the agent learns the fix from the error. The guiding idea: what the agent can't see in context doesn't exist, so knowledge from chats and heads gets moved into the repo.
- **Pros:** scales to very large codebases; docs are checkable; human and agent docs converge.
- **Cons:** real up-front investment, and it needs CI tooling and discipline. Overkill for small projects.
- **Fits:** teams, long-lived products, and heavy agent autonomy.
### 5.3 "Compressed always-on index" (Vercel)
- **What:** instead of a skill, embed a pipe-delimited *index* of the docs (topic → file path) directly in `AGENTS.md`, with the instruction to *prefer retrieval-led reasoning over pre-training-led reasoning*. Full docs live in a folder the agent reads on demand.
- **Pros:** the best measured reliability for knowledge the model lacks (new framework versions, internal APIs).
- **Cons:** costs tokens every turn (keep it to about 8 KB or less); needs regenerating when docs change (script it).
- **Fits:** framework-heavy repos, internal libraries, post-training-cutoff APIs.
### 5.4 "Big, agent-maintained AGENTS.md" (Peter Steinberger)
- **What:** about 800 lines, *written by the agent*. When something comes up, he asks the agent to add a concise note. A global `AGENTS.md` is shared across projects (his `agent-scripts` repo symlinks shared rules and skills). Each project's `docs/` is also agent-written, with a docs-listing script that makes the model read docs on given topics. He prunes guidance that newer models no longer need.
- **Pros:** very low friction, and the rules stay in the model's own phrasing.
- **Cons:** contradicts the minimalism consensus and the ETH findings. It works for him with strong models and constant supervision, but drift and bloat are real risks.
- **Fits:** solo developers who watch their agents closely and like to "just talk to it".
### 5.5 "Mistake ratchet / compound engineering" (Boris Cherny; Mitchell Hashimoto; Kieran Klaassen at Every)
- **What:** every observed mistake becomes either a line in the shared file or a programmed tool, paired with a line telling the agent the tool exists. Cherny tags Claude on PRs to add learnings. Hashimoto says each line in Ghostty's `src/inspector/AGENTS.md` came from a real bad behaviour. Every's loop is Plan → Work → Review → **Compound**: learnings go into `CLAUDE.md`, `docs/solutions/` and skills.
- **Pros:** grounded in real failures, so there are no imaginary rules.
- **Cons:** without pruning, the file grows without bound. Klaassen says his `CLAUDE.md` eventually got too big and needed a more deliberate memory system. Pair the ratchet with a size budget and periodic pruning.
- **Fits:** everyone. This is the default way to *evolve* the files.
### 5.6 "Mandatory skills framework" (Jesse Vincent's Superpowers)
- **What:** a bootstrap skill tells the agent to check for and use applicable skills before *any* response. Process skills (brainstorming, TDD, systematic debugging) are followed rigidly; pattern skills are adapted.
- **Pros:** strong process discipline; composable workflows.
- **Cons:** heavy-handed, and it competes with every other skill. Reviewers flag its "applies to every conversation" trigger as a distinctiveness problem. It adds ceremony, which Steinberger explicitly argues against.
- **Fits:** people who want enforced TDD, debugging and planning rituals.
### 5.7 "Hook-forced skill activation" (community pattern)
- **What:** a `UserPromptSubmit` hook matches the prompt (and optionally file paths) against `skill-rules.json` and injects "use these skills".
- **Pros:** makes skill loading close to deterministic.
- **Cons:** keyword matching is brittle, and it's extra machinery. Path-scoped rules or a root-file trigger often achieve the same thing more simply.
### 5.8 "Master-clone instead of specialist subagents" (Shrivu Shankar)
- **What:** put the key context in `CLAUDE.md` and let the main agent spawn *clones* of itself through the built-in task tool. Avoid a zoo of custom subagents that hide context and impose rigid delegation. Keep slash commands to a handful.
- **Contrast:** Boris Cherny's team uses some subagents (for example a code simplifier), and Anthropic's docs show read-only reviewer subagents. **Consensus-ish:** subagents are good for parallel work, noisy output and independent review. They don't produce smarter answers for the same task.
### 5.9 "Block at submit, not at write" (Shrivu Shankar)
- **What:** enforce tests with a `PreToolUse` hook on `git commit` that requires a "tests passed" marker. Don't block `Edit`/`Write` mid-plan, because that confuses the agent. Anthropic shows the related `Stop`-hook pattern: the agent can't end its turn until the typecheck passes.
- **Pros:** the agent finishes its plan and then is forced through a test-and-fix loop.
### 5.10 "CLAUDE.md as a forcing function for simpler tooling" (Shrivu; Ronacher)
- **What:** if a CLI or workflow is too complex to explain in a line or two, write a simpler wrapper (`make test-one FILE=…`, `bin/db-query`) and document the wrapper. Ronacher adds: make tools fast, protect them against misuse (for example, refuse to start a second dev server), and send all logs to one file the agent can tail.
- **Pros:** better for humans too; a shorter root file.
---
 
## 6. Placement: where each kind of instruction goes
 
### 6.1 Decision tree (apply to every line)
 
```
1. Could the agent find this by reading the code, AND would it actually look?
      yes → DELETE (don't document the discoverable).
2. Must it ALWAYS or NEVER happen, with zero acceptable exceptions?
      yes → ENFORCE: hook / permission deny / CI / test / linter / OS permission / scoped credential.
            Keep at most a one-line note in the root file so the agent understands
            why it got blocked and what to do instead.
3. Can a deterministic tool check it (format, naming, imports, layering)?
      yes → TOOL (formatter, linter, architecture test). One-line pointer at most.
4. Is it relevant to (nearly) every task in this repo?
      yes → ROOT FILE (AGENTS.md / CLAUDE.md), as one concrete line.
5. Is it relevant only when working in a specific area of the code?
      yes → PATH-SCOPED: nested AGENTS.md, or .claude/rules/<topic>.md with paths:.
6. Is it a multi-step procedure (deploy, release, migration, data fix, incident)?
      yes → SKILL. Side effects → user-invoked only (disable-model-invocation: true),
            backed by a script. Reference material goes in the skill's references/.
7. Is it long reference knowledge (schema, API contracts, domain rules, ADRs)?
      yes → docs/ (system of record), plus an index line or trigger in the root file.
            If it's critical and post-training (new framework or internal API),
            use the compressed-index technique (§5.3).
8. Is it about this machine or server rather than the repo (OS, installed services, sudo limits)?
      yes → MACHINE SCOPE: ~/.claude/CLAUDE.md or managed /etc/claude-code/CLAUDE.md
            on that server, plus managed settings and OS permissions. Not the repo file,
            because the repo is also cloned to laptops and CI.
9. Is it a personal preference?
      yes → CLAUDE.local.md (gitignored) or the user-scope file.
```
 
### 6.2 What MUST stay in the root file
 
Keep each item to one or a few lines. Omit an item if it doesn't apply. Empty template sections are noise (AAIF).
 
1. **One-paragraph identity:** what the project is, and any fact that changes behaviour (for example "this is a collection of independent samples, not one app", or "monorepo: `apps/*` deploy independently").
2. **Stack with versions,** only where versions matter. For example: PHP 8.3, Laravel 12, MySQL 8.0, Node 22. Say "not PHP 7 idioms" only if the agent tends to get that wrong.
3. **Commands:** install, dev server, full test suite, **single test file or single test**, lint, format, static analysis, typecheck, build. Include the working directory if it isn't the root.
4. **Definition of done:** which checks must pass before the agent says it's finished, and how validation scales with the size of the change (AAIF).
5. **Landmines:**
   - misleading code ("`src/Legacy/` is live but don't copy its patterns")
   - generated paths, their source, and the regeneration command
   - "use X, not Y" rules where Y looks correct (Kubo: `make build`, not `go build`)
6. **Boundaries:** always / ask first / never, each "never" with an alternative. Note which ones are *enforced* by a hook or permission, so the agent doesn't fight them.
7. **Routing table:** a short list of the supporting docs, rules and skills, each with a *trigger* ("read before…", "use when…").
8. **Environment facts the agent can't see** (pointer to the machine-scope file if it's server-specific). For example: required env vars, the services it may assume are running, sandbox limits.
9. **Git workflow essentials,** only if they're non-default: branch naming, commit message format, "never push to main".
### 6.3 What must NOT be in the root file
 
- Style rules a formatter or linter enforces, and language or framework defaults the model already knows.
- File tours and generic architecture overviews.
- Long procedures (move them to skills) and long reference material (move it to `docs/`).
- `@imports` of large docs, which cost context on every turn anyway (Shrivu; Anthropic's import semantics).
- Code snippets that will go stale. Point to exemplar files instead.
- Secrets, hostnames with credentials, or production connection strings.
- Time-relative statements ("recently we migrated…"), contradictions, and duplicates of README text (VS Code's `AGENTS.md` points to its existing instructions rather than duplicating them).
- Emphasis inflation (IMPORTANT, CRITICAL, ALL CAPS) used as a substitute for enforcement.
- Model-specific workarounds for older models ("think step by step", "don't be lazy", "keep going until done"). Anthropic lists these as obsolete; use effort settings, goals or hooks instead.
---
 
## 7. Making specific workflows reliably followed
 
### 7.1 The reliability ladder (choose the lowest rung that is reliable enough)
 
| Rung | Mechanism | Reliability | Use for |
|---|---|---|---|
| 1 | Line in the root file | High *visibility*, but following it is model-dependent | Universal facts and preferences |
| 2 | Root-file trigger → supporting doc ("Before X, read Y") | Medium–high | Area knowledge too long for the root |
| 3 | Path-scoped rule or nested file | Loads deterministically for matching files | Area conventions (naming in `app/Http/**`, test conventions in `tests/**`) |
| 4 | Skill, auto-invoked | Variable (Vercel measured it as unused in 56% of cases without extra prompting) | Optional know-how |
| 5 | Skill, user-invoked (`/name`) | Deterministic loading when invoked | Procedures with side effects |
| 6 | Hook that injects context (`SessionStart`, `UserPromptSubmit`) | Deterministic loading | Dynamic facts (current branch, environment = PROD) |
| 7 | Outcome checks: tests, linters, architecture tests, CI, `Stop` hook | Deterministic detection; agent self-corrects | Anything checkable |
| 8 | Prevention: `PreToolUse` hook (exit 2), permission `deny`, sandbox | Deterministic blocking *at the tool layer* | "Never do X" via agent tools |
| 9 | Infrastructure: OS file permissions, unprivileged user, read-only DB credentials, no production credentials on dev machines, branch protection | Holds even if the agent finds a workaround | Anything catastrophic or irreversible |
 
Permission rules match tool calls, not intent. For example, a `Read` deny rule doesn't stop a script that opens the file (Anthropic). **For catastrophic risks, rung 9 is the real control.** Rungs 1–8 are defence in depth.
 
### 7.2 Pattern: instruction + script + enforcement (the "triad")
 
For any workflow that matters:
1. **Script it:** `bin/test-one`, `bin/db-query`, `bin/deploy`. These are idempotent, return a clear exit code and print actionable errors.
2. **Teach it:** one line in the root file ("Run tests with `bin/test-one <file>`"), or a skill for multi-step use.
3. **Enforce it:** a hook, deny rule or CI job makes the unscripted path fail, with an error message that names the scripted path. That's OpenAI's "lint messages carry remediation" idea.
---
 
## 8. The six example workflows, worked through
 
> The examples assume a PHP repo deployed to a VPS. Adapt the names. Snippets show *shape*, not drop-in config: check the syntax against your tool versions.
 
### 8.1 Naming files and folders
 
| Layer | What goes there |
|---|---|
| **Enforce** | Encode the rules in tooling: PHP-CS-Fixer or PHP_CodeSniffer for code style; **Pest architecture tests** (`arch()->expect('App\Http\Controllers')->toHaveSuffix('Controller')`, `->toBeClasses()`, etc.) or PHPStan rules for class and namespace naming. Use a small CI script for file and folder naming (kebab-case assets, `YYYY_MM_DD_` migration prefixes). |
| **Root file** | One line: "Naming is enforced by `composer lint` + `tests/Architecture`; run them, and see `docs/conventions/naming.md` for the rationale." Add a rule only if the agent *repeatedly* gets one wrong. |
| **Path-scoped rule** | `.claude/rules/naming-http.md` with `paths: ["app/Http/**"]`, for conventions that can't be linted: "Form requests: `{Verb}{Entity}Request`; one invokable controller per action under `app/Http/Controllers/{Domain}/`". |
| **Avoid** | A 40-line naming table in the root file. |
 
### 8.2 PHP (or any language) architecture preferences
 
| Layer | What goes there |
|---|---|
| **Enforce** | Dependency direction and layering as **architecture tests** (Pest `arch()`, deptrac, or PHPUnit-based arch-test libraries). Examples: "Domain doesn't use Illuminate\\Http", "Controllers don't use DB facade", `preset()->php()`, `preset()->security()`, strict types. Error messages should say what to do instead. |
| **Root file** | 3–6 lines of *decisions the code doesn't reveal*. Examples: "New business logic goes in `app/Domain/{Context}/Actions` (single-method action classes); follow `app/Domain/Billing/Actions/CreateInvoice.php` as the exemplar"; "`app/Services/*` is legacy and live, so don't add new services there". |
| **Supporting doc** | `docs/architecture.md` (or ADRs in `docs/adr/`) for the *why*, boundaries, and trade-offs. Reach it with a trigger line: "Before adding a module or crossing a bounded context, read `docs/architecture.md`." |
| **Path-scoped rules** | Per-layer conventions, for example `.claude/rules/domain.md` (`paths: app/Domain/**`) and `.claude/rules/http.md`. |
| **Skill (optional)** | `scaffold-feature`: a procedure plus template files for creating a new vertical slice (action, request, controller, test). This is user-invoked or auto-invoked, depending on risk. |
 
### 8.3 Testing flow and commands (test suite)
 
| Layer | What goes there |
|---|---|
| **Root file (non-negotiable)** | Exact commands: full suite, **single file**, single test (`--filter`), parallel, coverage. Include prerequisites ("needs `docker compose up -d mysql-test`", "uses `.env.testing`"). Give a definition of done: "Before finishing: `composer test-one <changed tests>` while iterating; `composer check` (lint + stan + arch + tests) before saying done". |
| **Enforce** | A `Stop` hook running the fast checks (exit 2 sends the agent back to work); or a pre-commit hook or `PreToolUse` on `git commit` requiring green tests (Shrivu's block-at-submit); plus CI. |
| **Path-scoped rule** | `.claude/rules/tests.md` (`paths: tests/**`): factories over fixtures, which base `TestCase`, how to fake external HTTP, "never comment out or skip a failing test; fix it or ask". Optionally back that last rule with a grep check for `->skip(` or `markTestSkipped` in CI. |
| **Skill** | Complex flows such as E2E/browser tests, seeding a realistic dataset, snapshot updates, or "reproduce bug → failing test → fix" (Anthropic's prompt pattern). |
| **Script** | `bin/test-one` that picks the right config and prints concise failures, which saves context. |
 
### 8.4 Letting the LLM run commands on the database
 
| Layer | What goes there |
|---|---|
| **Infrastructure (the real control)** | Give the agent a **dedicated read-only DB user** for dev or staging data. **No production write credentials anywhere the agent can read.** Post-mortems (PocketOS, April 2026: a production DB and its co-located backups deleted in a single API call; Replit, July 2025) all conclude that an agent can do whatever its credentials allow. Keep backups off-volume. |
| **Script / tool** | `bin/db-query "<SQL>"` uses the read-only DSN, refuses non-SELECT statements, adds `LIMIT` by default and redacts sensitive columns. Or use a framework tool, such as Laravel Boost's MCP (schema, read-only query, logs, routes) or a dedicated read-only MCP. |
| **Writes / migrations** | Schema changes only through migrations (`php artisan make:migration`, `migrate` on dev). Data fixes go through a **user-invoked skill** (`/db-data-fix`) that follows plan → validate → execute: write a SQL or script file, dry-run it in a transaction, show affected row counts, wait for human approval, then run. That matches Anthropic's plan-validate-execute pattern for destructive batches. |
| **Enforce** | Deny rules for raw clients against non-dev hosts (for example `Bash(mysql -h prod*)`) and destructive patterns (`DROP`, `TRUNCATE`) as defence in depth, plus a `PreToolUse` hook that inspects SQL passed to the client. Remember that rules match strings and can be bypassed, which is why the credential layer comes first. |
| **Root file** | "DB: inspect with `bin/db-query` (read-only). Schema changes only via migrations. Data fixes: `/db-data-fix` (ask first). You have no production DB access, by design." |
| **Supporting doc** | `docs/database.md` lists the main tables, invariants, soft-delete conventions and multi-tenancy rules. Or point to the generated schema dump (OpenAI keeps `docs/generated/db-schema.md`, regenerated automatically). |
 
### 8.5 Doing `git pull` properly on the production server
 
| Layer | What goes there |
|---|---|
| **Script (single path)** | `bin/deploy` (idempotent):<br>1. check you're on the deploy branch and the working tree is clean (else abort)<br>2. `git fetch` then `git pull --ff-only` (never merge or rebase on prod)<br>3. `composer install --no-dev --optimize-autoloader`<br>4. run migrations with `--force`, maintenance mode if needed<br>5. cache config/routes/views<br>6. reload PHP-FPM via a narrowly allowed sudo rule<br>7. health check, and print a rollback hint (previous commit SHA) |
| **Skill** | `.claude/skills/deploy-prod/SKILL.md` with `disable-model-invocation: true`, so only `/deploy-prod` typed by a human runs it. Steps: pre-flight checks → run `bin/deploy` → verify health endpoint and logs → report. Anthropic's own release example uses this pattern. |
| **Enforce** | Deny or ask rules for `git reset --hard`, `git checkout -- .`, `git clean -fd`, `git push --force`, `git pull` without `--ff-only`, plus a `PreToolUse` hook that blocks raw git write commands when `APP_ENV=production` and tells the agent to use `bin/deploy`. Better still, make CI/CD the only route to production (post-mortem recommendation) and protect the branch. |
| **Root file** | "Deploys: never by hand. Human runs `/deploy-prod` (wraps `bin/deploy`)." |
| **Machine scope (if the agent runs ON the server)** | Put the facts in the **server's** `~/.claude/CLAUDE.md` or `/etc/claude-code/CLAUDE.md`, plus managed settings for the deny rules. These are facts about the server, not the repo. Add a `SessionStart` hook that prints "ENVIRONMENT: PRODUCTION – read-only unless /deploy-prod". |
 
### 8.6 Server layout, technical stack and environment limitations
 
Examples of limitations: "don't touch the global nginx config", "don't modify the VPS's system Python".
 
| Layer | What goes there |
|---|---|
| **Infrastructure (the real control)** | Run the agent as an **unprivileged user**. `/etc/nginx/*` and the system Python stay root-owned. Allow only specific sudo commands (`systemctl reload php8.3-fpm`). Use a project virtualenv or `uv` for any Python tooling. OS permissions hold even when the model improvises. |
| **Enforce** | Permission rules such as `deny: ["Edit(//etc/nginx/**)", "Bash(sudo pip *)", "Bash(pip install --break-system-packages*)", "Bash(apt *)"]` (check the path-rule syntax for your version). Turn on the sandbox for OS-level confinement, and use a `PreToolUse` hook for patterns that are hard to express. |
| **Machine-scope file** | A short "server facts" file: OS, PHP-FPM pool, which nginx *site* file the project may edit (if any) and how to test it (`nginx -t` before reload), where logs live, cron or supervisor jobs, disk and memory limits, and the known "don'ts", **each with the allowed alternative**. For example: "Need a Python tool? Use `uv run` in `~/tools/.venv`, never system pip." |
| **Repo `docs/`** | `docs/infrastructure.md`: the stack diagram and environment matrix (local / staging / prod), which is useful to humans too. Point to it from the root file with a trigger: "Before changing deployment, infra or config files, read `docs/infrastructure.md`." |
| **Root file** | 2–4 lines of the constraints that apply everywhere, plus the pointer. |
 
---
 
## 9. Reference skeletons
 
### 9.1 Recommended layout (cross-tool)
 
```
AGENTS.md                      # canonical root briefing (≈40–120 lines)
CLAUDE.md                      # "@AGENTS.md" + Claude-only extras (few lines)
CLAUDE.local.md                # personal, gitignored
.claude/
  settings.json                # shared permissions (deny/ask/allow) + hooks
  rules/
    tests.md                   # paths: tests/**
    domain.md                  # paths: app/Domain/**
    http.md                    # paths: app/Http/**
  skills/
    deploy-prod/SKILL.md       # disable-model-invocation: true → bin/deploy
    db-data-fix/SKILL.md       # disable-model-invocation: true
    scaffold-feature/SKILL.md  # + templates/
.agents/skills -> ../.claude/skills   # optional symlink so Codex/Cursor see the same skills
docs/
  architecture.md
  database.md
  infrastructure.md
  conventions/naming.md
  adr/
bin/
  test-one  db-query  deploy
tests/Architecture/            # arch tests = executable conventions
```
 
Optional: nested `AGENTS.md` in sub-packages or apps of a monorepo, instead of (or as well as) `.claude/rules`.
 
### 9.2 Root `AGENTS.md` example (shape, about 45 lines)
 
```markdown
# Acme Billing — agent briefing
Laravel 12 / PHP 8.3 monolith; MySQL 8; queues on Redis. Single deployable app.
 
## Commands
- Install: `composer install && npm ci`
- Tests (one file, use while iterating): `bin/test-one tests/Feature/InvoiceTest.php`
- Tests (one case): `php artisan test --filter=creates_invoice`
- Full check before you say "done": `composer check`  (pint + phpstan L8 + arch tests + suite)
- Needs: `docker compose up -d mysql-test redis`
 
## Conventions that aren't obvious from the code
- New business logic → `app/Domain/{Context}/Actions/*` (one public `handle()`).
  Exemplar: `app/Domain/Billing/Actions/CreateInvoice.php`.
- `app/Services/` is legacy but live: fix bugs there, don't add new services.
- Money is integer cents (`App\Support\Money`); never floats.
- Naming & layering are enforced by `tests/Architecture` — if it fails, read the message.
 
## Database
- Inspect with `bin/db-query "SELECT …"` (read-only user). You have no prod DB access by design.
- Schema changes only via migrations. Data fixes: ask the human to run `/db-data-fix`.
 
## Boundaries
- Ask first: adding a composer/npm dependency; changing `config/*.php`; anything in `infra/`.
- Never: deploy by hand (human runs `/deploy-prod`); edit `resources/js/generated/`
  (regenerate with `npm run gen:api`); skip or comment out tests (fix them or ask).
- Enforced by hooks/permissions: `.env*` unreadable; force-push and `git reset --hard` blocked.
 
## Read before…
- crossing bounded contexts or adding a module → `docs/architecture.md`
- touching tenancy, invoices or tax → `docs/database.md` (tenancy invariants)
- changing deploy/infra/nginx/cron → `docs/infrastructure.md`
```
 
### 9.3 `CLAUDE.md` shim
 
```markdown
@AGENTS.md
 
<!-- Claude-specific additions only below. HTML comments are stripped and cost nothing. -->
- Path-scoped rules live in .claude/rules/ and load automatically.
```
 
### 9.4 Path-scoped rule
 
```markdown
---
paths:
  - "tests/**/*.php"
---
- Use model factories, never raw inserts. Fake HTTP with `Http::fake()`.
- Extend `Tests\TestCase`; DB tests use `RefreshDatabase`.
- A failing test is never skipped or commented out: fix it, or stop and ask.
```
 
### 9.5 Side-effecting skill (user-invoked only)
 
```markdown
---
name: deploy-prod
description: Deploys the current main branch to production via bin/deploy, then verifies health. Only run when the user explicitly invokes /deploy-prod.
disable-model-invocation: true
---
1. Pre-flight: confirm CI is green for the target commit (`gh run list -b main -L 1`). If not green, stop.
2. Run `bin/deploy` and stream its output. Do not run git, composer or artisan commands by hand.
3. If it exits non-zero: do NOT retry blindly. Report the failing step and the rollback command it printed.
4. Verify: `curl -fsS https://app.example.com/health` and `tail -n 50 storage/logs/laravel.log`.
5. Report: deployed SHA, migrations run, health result.
```
 
### 9.6 Settings: deny rules and hooks (shape)
 
```json
{
  "permissions": {
    "deny": [
      "Read(./.env)", "Read(./.env.*)",
      "Bash(git push --force*)", "Bash(git reset --hard*)", "Bash(git clean -fd*)",
      "Bash(sudo pip *)", "Bash(apt *)", "Edit(//etc/nginx/**)"
    ],
    "ask": ["Bash(composer require *)", "Bash(git push *)"]
  },
  "hooks": {
    "Stop": [{ "hooks": [{ "type": "command", "command": "composer check-fast 1>&2 || exit 2" }] }],
    "PostToolUse": [{ "matcher": "Edit|Write",
      "hooks": [{ "type": "command", "command": "vendor/bin/pint --dirty -q || true" }] }]
  }
}
```
 
Keep the enforced set small. Every hook or deny rule will sometimes fire when it shouldn't, and each needs maintenance. Forty of them usually means most are preferences (Anthropic).
 
---
 
## 10. Audit checklist and report format
 
### 10.1 Checklist
 
**A. Loading and topology**
- A1. The files that actually load for each targeted tool are known and intended. In particular: if both `CLAUDE.md` and `AGENTS.md` exist, `CLAUDE.md` imports `AGENTS.md` (or deliberately doesn't).
- A2. There's a single source of truth across tools, with no hand-maintained duplicates (`.cursorrules`, `copilot-instructions.md`, `GEMINI.md`) drifting apart.
- A3. No symlink pitfalls (Windows contributors; agent edits through symlinks).
- A4. Codex: the combined `AGENTS.md` chain is under 32 KiB (or `project_doc_max_bytes` has been raised on purpose).
- A5. Personal preferences are in `CLAUDE.local.md` or user scope, not the shared file.
- A6. Machine- and server-specific facts are in machine-scope files, not the repo root.
**B. Root file content**
- B1. Length: under about 200 lines (WARN above 150, FAIL above 300), and ideally well under that.
- B2. Commands are present, exact and runnable, including single-test and single-file variants. Spot-check that they really run.
- B3. There's a definition of done.
- B4. No discoverable filler: file tours, generic overviews, language defaults, platitudes.
- B5. No style or format rules a tool already enforces.
- B6. Every "never" has an alternative, and enforced rules are marked as enforced.
- B7. A routing table with explicit triggers exists for the supporting docs, rules and skills.
- B8. No stale references: every path and command mentioned exists. Grep each path.
- B9. No contradictions with the code, config or linters (for example, "no default exports" while ESLint allows them and the code is full of them).
- B10. No emphasis inflation, and no obsolete model workarounds.
- B11. No secrets, production hostnames with credentials, or connection strings.
- B12. No large `@imports` that silently bloat context.
**C. Progressive disclosure**
- C1. Area-specific guidance is in path-scoped rules or nested files, not the root.
- C2. Supporting docs are under about 300 lines each, with a table of contents if over 100 lines, and linked one level deep from the root or `SKILL.md`.
- C3. Docs are the system of record. There's a freshness mechanism (CI link and path check, doc-gardening, or a review cadence).
**D. Skills**
- D1. Each skill is a *procedure* or packaged capability, not a dump of facts.
- D2. Descriptions say what the skill does and when to use it, in the third person, with trigger terms. Under 1,024 characters (agentskills.io) and well under Claude Code's 1,536-character listing cap.
- D3. Side-effecting skills (deploy, release, data fix, migrations on shared environments) set `disable-model-invocation: true`.
- D4. `SKILL.md` is under 500 lines; references sit one level deep; deterministic steps are scripts with clear errors.
- D5. Critical knowledge does *not* depend solely on auto-invocation (see §4.2).
- D6. Skills are visible to all targeted tools (`.claude/skills` and/or `.agents/skills`).
- D7. No duplicate or overlapping skills competing for the same trigger.
**E. Enforcement**
- E1. Every "always" or "never" in the instruction files has been checked: either it's enforced (hook, permission, CI, test, OS permission) or it's acceptable as advisory.
- E2. Tests, lint, static analysis and architecture checks run in CI, and ideally in a `Stop` or pre-commit hook.
- E3. Architecture and naming rules exist as executable tests where possible.
- E4. Secret files are denied for reading; destructive git commands are denied or ask first.
- E5. The agent has no production write credentials, uses a read-only DB user for inspection, and backups are not co-located with production data.
- E6. Production changes go through a single scripted path (CI/CD or `bin/deploy`), with branch protection.
- E7. Hooks are few, fast and documented.
**F. Maintenance**
- F1. The instruction files are in git with team ownership, and changes are reviewed.
- F2. Evidence of the ratchet (lines that trace back to real mistakes) and of pruning (git history).
- F3. A lightweight eval exists: a few representative tasks re-run after big changes to the instruction files (Anthropic's skill guidance; ETH's "evaluate rigorously before adopting").
### 10.2 Severity guide
 
- **Critical:** an agent could cause irreversible damage. Examples: production credentials readable, no deploy guardrail, destructive DB access, secrets in instruction files.
- **High:** important rules that are only advisory when they should be enforced; the file that actually loads differs from the one maintained; broken commands; critical knowledge reachable only through an auto-triggered skill.
- **Medium:** root-file bloat; discoverable filler; missing triggers; stale paths; linter-able rules written as prose.
- **Low:** wording, ordering, emphasis inflation, missing ToCs.
### 10.3 Required report format
 
```markdown
# Agent-instructions audit — <repo> — <date>
## 1. Inventory
| File | Tool(s) | Loads when | Lines / chars | Owner |
## 2. What actually loads (per tool) + surprises
## 3. Findings (sorted by severity)
### [CRITICAL|HIGH|MEDIUM|LOW] <title>
- Evidence: `path:line` — "<quoted line>"
- Expert basis: §<section of reference doc> (consensus C#/majority M#/technique 5.#)
- Recommendation: <keep/move/delete/enforce> → <target file or mechanism>
- Proposed change: <diff or new file content>
## 4. Line-by-line triage of the root file
| Line(s) | Content (short) | Verdict (keep/move/delete/enforce) | Destination |
## 5. Proposed target structure (tree) + migration steps (ordered, smallest-risk first)
## 6. Enforcement gaps: rules currently advisory that should be deterministic
## 7. Open questions for the maintainer
```
 
---
 
## 11. Keeping it effective over time
 
- **Measure, don't guess.**
  - Use `/context` (Claude Code) to see what loaded and how big it is.
  - Watch a few sessions: which pointers does the agent follow or ignore, and which mistakes recur?
  - Keep a small set of benchmark tasks and re-run them after major edits. Vercel and the ETH authors both found that intuitive "improvements" often don't help.
- **Ratchet, then prune.** Add only in response to observed failures. On a schedule (monthly, or after each model upgrade), delete:
  - lines the current model no longer needs
  - stale paths
  - rules now enforced by tooling
  - anything contradicted by the code
  Use `/doctor prompt-audit` and a CI step that checks every path and command named in the instruction files.
- **Order matters.** Put the most important items first: commands, then landmines, then boundaries. The IFScale benchmark measured a bias towards earlier instructions.
- **Write for the model you run.** Smaller or cheaper models need more explicit, shorter instructions. GPT-family and Claude-family models respond to different styles (Steinberger), so keep model-specific tweaks in the per-tool shim.
- **Use HTML comments for maintainer notes** in `CLAUDE.md`. They're stripped before injection.
- **Make lints and test failures teach.** Error messages should say exactly what to do. The agent reads them in context (OpenAI harness).
---
 
## 12. Sources (accessed November 2026)
 
**Vendor documentation and standards**
- Anthropic — Claude Code memory (CLAUDE.md, rules, imports, AGENTS.md support): https://code.claude.com/docs/en/memory
- Anthropic — Claude Code best practices (deterministic vs. non-deterministic; "what you can stop doing"): https://code.claude.com/docs/en/best-practices
- Anthropic — Claude Code skills: https://code.claude.com/docs/en/skills
- Anthropic — Agent Skills authoring best practices: https://platform.claude.com/docs/en/agents-and-tools/agent-skills/best-practices
- Anthropic — Effective context engineering for AI agents: https://www.anthropic.com/engineering/effective-context-engineering-for-ai-agents
- OpenAI — Codex AGENTS.md discovery: https://learn.chatgpt.com/docs/agent-configuration/agents-md
- OpenAI — Codex skills: https://developers.openai.com/codex/skills
- OpenAI — Harness engineering (Ryan Lopopolo, Feb 2026): https://openai.com/index/harness-engineering/
- AGENTS.md format: https://agents.md/ · Agent Skills spec: https://agentskills.io
- AAIF — Writing an effective AGENTS.md (Aug 2026): https://aaif.io/blog/writing-an-effective-agents-md
- GitHub — How to write a great agents.md (2,500+ repos): https://github.blog/ai-and-ml/github-copilot/how-to-write-a-great-agents-md-lessons-from-over-2500-repositories/
**Empirical studies**
- Gloaguen, Mündler, Müller, Raychev, Vechev — Evaluating AGENTS.md (ETH Zurich, 2026): https://arxiv.org/abs/2602.11988
- Vercel (Jude Gao) — AGENTS.md outperforms skills in our agent evals (Jan 2026): https://vercel.com/blog/agents-md-outperforms-skills-in-our-agent-evals
- Jaroslawicz et al. — How Many Instructions Can LLMs Follow at Once? (IFScale): https://arxiv.org/abs/2507.11538
**Practitioners**
- HumanLayer (Kyle) — Writing a good CLAUDE.md: https://www.humanlayer.dev/blog/writing-a-good-claude-md
- Dex Horthy — Advanced context engineering / research-plan-implement / "dumb zone": https://ai.engineer/talks/context-engineering-for-complex-codebases
- Shrivu Shankar — How I use every Claude Code feature: https://blog.sshh.io/p/how-i-use-every-claude-code-feature
- Mitchell Hashimoto — My AI adoption journey ("Engineer the harness"): https://mitchellh.com/writing/my-ai-adoption-journey
- Peter Steinberger — Just Talk To It: https://steipete.me/posts/just-talk-to-it · Shipping at inference speed: https://steipete.me/posts/2025/shipping-at-inference-speed
- Armin Ronacher — Skills vs. dynamic MCP loadouts: https://lucumr.pocoo.org/2025/12/13/skills-vs-mcp/ · Agentic coding talk notes (via Simon Willison): https://simonwillison.net/b/8806
- Simon Willison — Claude Skills are awesome, maybe a bigger deal than MCP: https://simonwillison.net/2025/Oct/16/claude-skills/
- Boris Cherny (Claude Code team workflow), as reported by InfoQ: https://www.infoq.com/news/2026/01/claude-code-creator-workflow
- Kieran Klaassen / Every — Compound engineering: https://every.to/guides/compound-engineering
- Jesse Vincent — Superpowers (using-superpowers skill): https://github.com/obra/superpowers
- Geoffrey Huntley's Ralph loop as documented in The Ralph Playbook: https://claytonfarr.github.io/ralph-playbook
- Addy Osmani — Agent harness engineering: https://addyosmani.com/blog/agent-harness-engineering/
**Domain-specific (PHP, safety)**
- Laravel Boost (guidelines, skills, MCP incl. DB schema and query tools): https://laravel.com/docs/boost
- Pest architecture testing: https://pestphp.com/docs/arch-testing
- PocketOS production-database deletion (Apr 2026) analysis: https://mondoo.com/blog/5-lessons-from-9-seconds-ai-agent-deleted-production-database
 
