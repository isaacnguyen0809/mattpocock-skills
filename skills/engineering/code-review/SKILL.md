---
name: code-review
description: "Review the changes since a fixed point (commit, branch, tag, or merge-base) along two axes: Standards (does the code follow this repo's documented coding standards?) and Spec (does the code match what the originating issue/spec asked for?). Runs both reviews in parallel sub-agents and reports them side by side. Use when the user wants to review a branch, a PR, work-in-progress changes, or asks to \"review since X\"."
---

Two-axis review of the diff between `HEAD` and a fixed point the user supplies:

- **Standards**: does the code conform to this repo's documented coding standards?
- **Spec**: does the code faithfully implement the originating issue / spec?

Both axes run as **parallel sub-agents** so they don't pollute each other's context, then this skill aggregates their findings.

The issue tracker should have been provided to you. If `docs/agents/issue-tracker.md` is missing, tell the user to run `/setup-matt-pocock-skills`.

**You are the orchestrator, not a third reviewer.** The diff belongs in the sub-agents' contexts, not yours. Every hunk you read yourself is a hunk paid for twice.

## Process

### 1. Pin the fixed point

Whatever the user said is the fixed point (a commit SHA, branch name, tag, `main`, `HEAD~5`, etc.). If they didn't specify one, ask for it.

Confirm it resolves (`git rev-parse <fixed-point>`), then size the change with `git diff --stat <fixed-point>...HEAD` (three-dot, so the comparison is against the merge-base). Note the commits via `git log <fixed-point>..HEAD --oneline`.

**Do not run the full `git diff` yourself.** The `--stat` tells you whether the diff is empty and which files changed; that is all you need. A bad ref or empty diff fails here, not inside two parallel sub-agents.

### 2. Identify the spec source

In this order, stopping at the first hit:

1. A path the user passed as an argument.
2. Issue references in the commit messages you already have from step 1 (`#123`, `Closes #45`, GitLab `!67`), fetched via the workflow in `docs/agents/issue-tracker.md`.
3. Ask the user where the spec is.

Do **not** go hunting through `docs/`, `specs/`, and `.scratch/` for something that might match the branch name. An open-ended search costs more than the question, and guesses wrong often enough that you would have to ask anyway. If the user says there is no spec, skip the Spec sub-agent and note it in the final report.

### 3. Identify the standards sources

List the **paths** of anything in the repo that documents how code should be written — `CODING_STANDARDS.md`, `CONTRIBUTING.md`, `CLAUDE.md`, `AGENTS.md`. Pass paths, not contents: the sub-agent reads what it needs.

On top of whatever the repo documents, the Standards axis always carries the **smell baseline** in [smells.md](smells.md), a fixed set of Fowler code smells (_Refactoring_, ch.3) that applies even when a repo documents nothing. Pass its absolute path (it sits next to this `SKILL.md`); do not paste its contents. A documented repo standard always overrides the baseline, and every smell is a judgement call, never a hard violation.

### 4. Spawn both sub-agents in parallel

Send a single message with two `Agent` tool calls. Use the `general-purpose` subagent for both.

Both prompts carry this **budget clause** verbatim:

> Run `git diff <fixed-point>...HEAD` exactly once, that is your copy of the change; re-running it or any other git command is wasted. Work from the diff alone. Open a source file only when a finding genuinely depends on context the hunk doesn't show, and read only that region, never a whole file to "get oriented". Report under 400 words.

**Standards sub-agent prompt** should include:

- The diff command and commit list.
- The standards-source **paths** from step 3, and the absolute path to `smells.md`.
- The brief: "Read the smell baseline file first. Then report, per file/hunk where relevant, (a) every place the diff violates a documented standard: cite the standard (file + the rule); and (b) any baseline smell you spot: name it and quote the hunk. Distinguish hard violations from judgement calls: documented-standard breaches can be hard, but baseline smells are always judgement calls, and a documented repo standard overrides the baseline. Skip anything tooling enforces."

**Spec sub-agent prompt** should include:

- The diff command and commit list.
- The path to the spec (or its fetched contents, if it came from an issue tracker and has no path).
- The brief: "Report: (a) requirements the spec asked for that are missing or partial; (b) behaviour in the diff that wasn't asked for (scope creep); (c) requirements that look implemented but where the implementation looks wrong. Quote the spec line for each finding."

### 5. Aggregate

Present the two reports under `## Standards` and `## Spec` headings, verbatim or lightly cleaned. Do **not** merge or rerank findings, because the two axes are deliberately separate (see _Why two axes_).

End with a one-line summary: total findings per axis, and the worst issue _within each axis_ (if any). Don't pick a single winner across axes: that's the reranking the separation exists to prevent.

## Quick mode

If the user explicitly asks for a quick or cheap review, run both briefs **inline** in this context instead of spawning sub-agents, Standards first and Spec second.

This trades away the isolation the two axes exist for — having read the spec, you will be tempted to excuse a standards breach, which is the masking effect step 5 forbids. It is only worth it on a small change, where two agents' fixed overhead exceeds the review itself. **Default to the sub-agent route**; take this one only when asked.

## Why two axes

A change can pass one axis and fail the other:

- Code that follows every standard but implements the wrong thing → **Standards pass, Spec fail.**
- Code that does exactly what the issue asked but breaks the project's conventions → **Spec pass, Standards fail.**

Reporting them separately stops one axis from masking the other.
