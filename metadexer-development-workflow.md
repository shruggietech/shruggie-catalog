# Metadexer Development Workflow

- **Project:** metadexer
- **Author:** William Thompson (ShruggieTech LLC)
- **Date:** 2026-03-08
- **Status:** DRAFT
- **Audience:** AI-first, Human-second

## 1. Purpose

This document defines the standardized development workflow for the metadexer project. It governs how administrative planning, coding, review, and integration are conducted across all development sessions. All participants (human and AI agents) MUST follow this workflow unless an explicit, documented exception is approved by the project lead.

This workflow supersedes all ad-hoc development patterns previously used on the project. It incorporates parallel agent execution, worktree-based isolation, and a structured handoff protocol between strategic planning and implementation.

## 2. Workflow Layers

Development activity is organized into two distinct layers. These layers operate independently and communicate through version-controlled artifacts only.

### 2.1 Admin Layer

The admin layer is responsible for strategic direction, specification authoring, sprint planning, brainstorming, and roadblock resolution. It operates in browser-based AI chat sessions (Claude.ai projects, or equivalent) where rich project history and personal context are available.

**Outputs produced by the admin layer:**

- Technical specifications (e.g., `metadexer-spec.md`, `shruggie-indexer-spec.md`)
- Sprint planning documents (filed in `.archive/` per Appendix D of the overview)
- Session prompt templates (`_TEMPLATE.txt` files paired with sprint documents)
- `CLAUDE.md` or equivalent agent context files at the repository root
- Workflow documents (this document)

**Inputs consumed by the admin layer:**

- Commit history and changelogs from the coding layer
- Test results and CI/CD pipeline status
- Bug reports and behavioral observations from runtime testing
- Agent session transcripts (when debugging agent behavior)

The admin layer MUST NOT make direct code changes. All implementation flows through the coding layer.

### 2.2 Coding Layer

The coding layer is responsible for implementing the work defined by the admin layer. It operates in IDE-based and terminal-based AI coding agent sessions (Claude Code, VS Code background/cloud agents, or equivalent tools).

**Inputs consumed by the coding layer:**

- Sprint planning documents from `.archive/`
- Session prompt templates (`_TEMPLATE.txt`)
- The authoritative specification for the component under modification
- `CLAUDE.md` or equivalent agent context file
- The current codebase state (verified independently, not assumed)

**Outputs produced by the coding layer:**

- Code changes committed to the repository
- Test results and verification evidence
- Changelog entries
- Agent session artifacts (diffs, logs) for admin review

The coding layer MUST NOT make architectural decisions, redefine module boundaries, or deviate from the specification without explicit authorization from the admin layer.

## 3. Handoff Protocol

The admin-to-coding handoff is the critical integration point. All task definitions MUST pass through a written artifact before entering the coding layer.

### 3.1 Admin-to-Coding Handoff

1. The admin layer produces a sprint document following the five-section structure defined in Appendix D of the project overview: header block, purpose and context, implementation ordering, work item sections, and specification update directive.
2. Each work item section within the sprint document is written as a self-contained unit suitable for execution by an AI coding agent within a single context window, without interactive clarification.
3. The sprint document is committed to `.archive/` in the repository.
4. A paired `_TEMPLATE.txt` prompt file is committed alongside the sprint document.
5. The coding layer reads the sprint document and template to initiate work.

### 3.2 Coding-to-Admin Handoff

1. When a sprint's work items are complete, all changes are committed and pushed.
2. `CHANGELOG.md` is updated to reflect the completed work.
3. Test results (pass/fail, coverage deltas) are recorded in the commit message or a summary document.
4. The admin layer reviews the commit history, changelog, and test results before initiating the next planning cycle.

## 4. Agent Execution Model

The coding layer supports three execution modes. The project lead selects the appropriate mode based on task characteristics.

### 4.1 Sequential Mode

A single agent session processes one work item at a time. The agent operates directly on the main branch. Changes are reviewed and committed before the next work item begins.

**When to use:** Simple, low-risk changes. Bug fixes with narrow scope. Specification updates. Tasks where the overhead of worktree setup exceeds the time saved by parallelism.

**Procedure:**

1. Open a coding agent session (VS Code local agent, Claude Code interactive, or equivalent).
2. Provide the sprint document and template as context.
3. The agent implements the work item.
4. Review the changes. Run acceptance criteria verification commands from the sprint document.
5. Commit to main. Proceed to the next work item.

### 4.2 Parallel Independent Mode

Multiple agent sessions run simultaneously, each in an isolated Git worktree on a separate branch. Agents work on independent tasks and cannot interfere with each other.

**When to use:** Two or more work items from the same sprint are independent (no shared file modifications, no sequential dependency). Tasks are well-defined and self-contained. Time savings from parallelism justify the merge overhead.

**Procedure:**

1. Identify independent work items in the sprint document. Verify independence by checking the "affected file matrix" in each work item section. If two items modify the same file, they are NOT independent and MUST be executed sequentially.
2. For each independent work item, start a separate agent session in its own worktree:
   - **Using VS Code:** Start a Background Agent session per task (Chat view > New Chat (+) > New Background Agent > select "Worktree" isolation).
   - **Using Superset:** Create a new task (Cmd+T) per work item. Superset handles worktree creation automatically.
   - **Using Claude Code directly:** Use the `--worktree` flag or `isolation: worktree` in the agent definition.
3. Provide each agent with: the relevant work item section from the sprint document, the specification, and the `CLAUDE.md` context file.
4. Monitor agent progress. Intervene only if an agent requests clarification or stalls.
5. When an agent completes, review the diff against the sprint document's acceptance criteria. Run the specified verification commands.
6. Merge completed branches into main one at a time, in the dependency order specified by the sprint document's implementation ordering section:
   ```bash
   git checkout main
   git merge --no-ff <branch-name>
   ```
   If using VS Code background agents, use the "Apply" action in the Chat view instead of manual merge commands.
7. After each merge, verify that the cumulative codebase passes all tests before proceeding to the next merge.
8. Delete completed worktrees after successful merge.

**Branch naming convention:**

```
sprint/<YYYYMMDD>-<ZZZ>/<work-item-number>-<short-description>
```

Example: `sprint/20260310-001/03-vault-put-operation`

### 4.3 Coordinated Team Mode

A lead agent decomposes a complex task and delegates subtasks to multiple subordinate agents that can communicate with each other. This mode is experimental and carries significantly higher token costs.

**When to use:** Complex tasks that benefit from multiple perspectives operating on the same logical problem (e.g., parallel exploration of architecture options, simultaneous implementation and test writing for a tightly coupled module). Reserved for tasks where the cost is justified by the complexity.

**Procedure (Claude Code Agent Teams):**

1. Verify the `CLAUDE_CODE_EXPERIMENTAL_AGENT_TEAMS` environment variable is set to `1`.
2. Start a Claude Code session and describe the task, specifying how many teammates to spawn and what each should focus on.
3. The lead agent coordinates work through a shared task list. Monitor progress in split-pane terminal mode (requires tmux or iTerm2).
4. Review the combined output. The lead agent synthesizes results from all teammates.
5. Commit the coordinated result to main (or to a worktree branch for review first).

**Cost expectation:** Approximately 5x the token usage of a single-agent session per spawned teammate. Use sparingly and only for tasks where the coordination overhead produces measurably better outcomes.

## 5. Git Conventions

### 5.1 Branch Strategy

The project uses a single `main` branch as the integration target. All work, whether executed sequentially or in parallel, merges to `main`.

Feature branches exist only as temporary worktree branches during parallel execution. They are deleted after merge. Long-lived feature branches are not used.

### 5.2 Commit Discipline

- Each logical change is one commit. Agents SHOULD NOT bundle unrelated changes into a single commit.
- Commit messages follow the pattern: `<module>: <imperative description>` (e.g., `vault: implement put operation for local backend`).
- Sprint-related commits MAY reference the sprint document in the message body: `Sprint: 20260310-001, Item 3`.
- Merge commits from parallel branches use `--no-ff` to preserve branch history in the log.

### 5.3 Worktree Cleanup

After a successful merge, the worktree and its branch MUST be deleted:

```bash
git worktree remove <path>
git branch -d <branch-name>
```

If using Superset or VS Code, worktree cleanup is handled automatically when you dismiss a completed task or delete an agent session.

## 6. Agent Context Management

### 6.1 CLAUDE.md

A `CLAUDE.md` file at the repository root provides persistent project-level context to all coding agents. This file is read automatically by Claude Code at session start and can be referenced explicitly by other agent systems.

**Required contents:**

- Project identity (name, organization, language, Python version requirement)
- Repository structure summary (top-level directory layout)
- Specification location and authority hierarchy
- Coding conventions (encoding, line endings, import ordering, logging patterns)
- Testing conventions (how to run tests, marker registration, coverage requirements)
- CLI conventions (click, stdout/stderr separation, dry-run support)
- Explicit prohibitions (no silent data loss, no implicit deletion, no recomputing identity)

**Maintenance:** The admin layer is responsible for keeping `CLAUDE.md` current. Updates to `CLAUDE.md` are committed as standalone commits with the message `docs: update CLAUDE.md`.

### 6.2 Sprint Document as Agent Context

When executing a work item, the coding agent's context MUST include:

1. The `CLAUDE.md` file (always).
2. The specific work item section from the sprint document (always).
3. The authoritative specification for the component being modified (always).
4. The `_TEMPLATE.txt` prompt file, if one is paired with the sprint document (when available).
5. Any referenced prior sprint documents or changelogs (only when the work item explicitly depends on prior work).

Agents MUST NOT be given the entire sprint document if they are only executing one work item. Unnecessary context degrades agent performance and increases cost.

### 6.3 Agent Session Discipline

These rules are inherited from the project overview (Appendix D) and are restated here for emphasis:

- Agents MUST NOT trust prior implementation work without independent verification.
- Agents MUST use grep-based evidence collection to verify codebase state before making changes.
- Agents MUST verify acceptance criteria against actual runtime behavior, not assumed correctness from code inspection alone.
- Each work item section is self-contained. An agent implementing Item 3 MUST NOT assume that Item 2 has been completed unless the sprint document's implementation ordering explicitly establishes that dependency and the agent has verified the dependency is satisfied in the current codebase.

## 7. Review and Integration

### 7.1 Review Checklist

Before merging any agent-produced changes (whether from sequential or parallel execution), the project lead verifies:

1. **Acceptance criteria:** Every criterion listed in the sprint document's work item is met, verified by running the specified commands.
2. **Specification compliance:** Changes conform to the authoritative specification. No new behavior is introduced that the specification does not define or permit.
3. **Test passage:** `pytest` runs clean with no failures or unexpected warnings.
4. **Encoding:** All new or modified files use UTF-8 without BOM and LF line endings. Spot-check with `file --mime-encoding <path>`.
5. **No scope creep:** Changes are limited to what the sprint document authorized. Agents occasionally introduce "helpful" improvements outside the defined scope; these are reverted or deferred to a future sprint.
6. **Changelog:** `CHANGELOG.md` reflects the changes accurately.

### 7.2 Merge Conflict Resolution

When merging parallel branches, conflicts may arise if:

- Two agents modified the same file despite the affected file matrix indicating independence (an error in the sprint document).
- An agent modified a shared dependency (e.g., `__init__.py` imports, `pyproject.toml` dependencies).

Conflict resolution is the project lead's responsibility. Resolution follows these rules:

- If both changes are correct and non-overlapping within the file, combine them manually.
- If the changes are logically conflicting (two different approaches to the same problem), choose one and file a follow-up item for the discarded approach if needed.
- After resolving conflicts, re-run the full test suite to verify the combined result.

## 8. Tooling Configuration

### 8.1 Recommended Stack

| Layer | Primary Tool | Alternative |
|-------|-------------|-------------|
| Admin | Claude.ai (Project-based) | Any browser-based AI chat with persistent context |
| Coding (interactive) | VS Code Local Agent or Claude Code (interactive) | Any IDE with integrated AI chat |
| Coding (parallel) | Superset + Claude Code | VS Code Background Agents |
| Coding (cloud/CI) | VS Code Cloud Agent (Copilot coding agent) | Claude Code on web (claude.ai/code) |
| Review | VS Code (final codebase), Superset (per-agent diffs) | `git diff` on the command line |

### 8.2 VS Code Configuration

The following VS Code settings are recommended for multi-agent workflows:

```json
{
  "chat.agent.enabled": true,
  "github.copilot.chat.claudeAgent.enabled": true,
  "editor.formatOnSave": true,
  "files.encoding": "utf8",
  "files.eol": "\n"
}
```

### 8.3 Superset Configuration

If Superset is used for orchestration, the repository SHOULD include a `.superset/` directory with:

```
.superset/
├── config.json
├── setup.sh
└── teardown.sh
```

**config.json:**
```json
{
  "setup": ["./.superset/setup.sh"],
  "teardown": ["./.superset/teardown.sh"]
}
```

**setup.sh:**
```bash
#!/bin/bash
# Copy environment variables from the root worktree
cp ../.env .env 2>/dev/null || true
# Install development dependencies
pip install -e ".[dev]" --quiet
echo "Worktree ready."
```

**teardown.sh:**
```bash
#!/bin/bash
echo "Worktree cleanup complete."
```

## 9. Workflow Summary

The following sequence describes one complete development cycle from planning through integration.

1. **Admin layer** reviews the current state of the project (recent commits, test results, open issues).
2. **Admin layer** authors a sprint document defining the next batch of work items. The document follows the five-section structure from Appendix D.
3. **Admin layer** commits the sprint document and its `_TEMPLATE.txt` to `.archive/`.
4. **Project lead** reviews the sprint document's work items and determines the execution mode:
   - If all items are sequential dependencies: use Sequential Mode (§4.1).
   - If some items are independent: use Parallel Independent Mode (§4.2) for independent items, Sequential Mode for dependent items.
   - If a single complex item benefits from multi-perspective exploration: use Coordinated Team Mode (§4.3).
5. **Coding layer** executes work items per the selected mode.
6. **Project lead** reviews agent output per the Review Checklist (§7.1).
7. **Coding layer** merges approved changes to main.
8. **Project lead** verifies the cumulative codebase state (full test suite, encoding checks).
9. **Admin layer** consumes the coding layer's output (commits, changelog, test results) and begins planning the next cycle.

## 10. Document History

| Date | Change |
|------|--------|
| 2026-03-08 | Initial draft. Establishes the two-layer workflow, three execution modes, handoff protocol, and tooling configuration for the metadexer project. |
