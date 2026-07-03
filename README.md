# burst

[![License: MIT](https://img.shields.io/badge/License-MIT-yellow.svg)](LICENSE)

**A [Claude Code](https://claude.com/claude-code) skill that ships a small change end-to-end in one shot.**
Describe what you want fixed or added, and burst runs the full cycle - issue, branch, implement,
validate, squash, PR - with a single confirmation gate.

```
you: /burst the dropdown closes when clicking inside it on mobile
burst: confirms plan -> creates issue -> worktree -> fix -> validates in browser -> squash -> PR

Burst complete:
- Issue: https://github.com/org/repo/issues/42
- PR: https://github.com/org/repo/pull/43
- Confidence: high
- What I'd check manually: nothing - verified at mobile viewport
```

You stay in control: burst confirms its read of your intent before touching anything, and every PR
ships with an honest confidence self-assessment.

---

- [When to use burst](#when-to-use-burst)
- [Prerequisites](#prerequisites)
- [Install](#install)
- [The pipeline](#the-pipeline)
- [What burst doesn't do](#what-burst-doesnt-do)
- [The PR it creates](#the-pr-it-creates)
- [Design principles](#design-principles)
- [License](#license)

## When to use burst

Burst is for changes where you already know **what** needs to happen.
You've seen the bug, you know which file it's in, and the fix is scoped to a handful of files.

Good fits:
- "The VAT label says 'incl.' but it should say 'excl.' for business shops"
- "Add a `gap` prop to the `Stack` component"
- "The SAYT dropdown is missing the hover state on the second row"

Bad fits:
- Large features that cross multiple concerns (use a plan instead)
- Exploratory work where you're not sure what the fix is (use systematic debugging)
- Anything that needs design discussion before code

If burst detects the change is wider than expected (~3+ files across unrelated concerns), it'll flag
it and ask if you want to plan first.

## Prerequisites

- **[Claude Code](https://claude.com/claude-code)** CLI on your `PATH`
- **`gh`** authenticated (`gh auth status`)
- **Git remotes configured**: `upstream` pointing at the org repo, `origin` at your fork
- **`agent-browser`** (only for frontend changes - burst will prompt to install if missing)

## Install

Burst is a Claude Code skill - a markdown file that Claude reads when you invoke `/burst`.
There's no runtime, no dependencies, no build step.

### Claude Code (CLI or Desktop)

Copy the skill into your skills directory:

```bash
git clone https://github.com/mtthsnc/burst
mkdir -p ~/.claude/skills/burst
cp burst/SKILL.md ~/.claude/skills/burst/SKILL.md
```

Or symlink it so `git pull` keeps it updated:

```bash
git clone https://github.com/mtthsnc/burst ~/dev/burst
mkdir -p ~/.claude/skills/burst
ln -sf ~/dev/burst/SKILL.md ~/.claude/skills/burst/SKILL.md
```

### Any agent harness

Burst is a single markdown file with no external dependencies.
Open `SKILL.md`, copy its contents, and paste them into your agent's instructions or system prompt.
The skill is self-contained - it carries its own pipeline, principles, and guardrails inline.

### Verify

Start a new Claude Code session and type `/burst` - you should see it in the skill list.

## The pipeline

Burst runs a 10-step pipeline after a single user confirmation:

```
you describe the change
        |
    1. sniff codebase (remotes, default branch, relevant files, conventions)
    2. confirm plan with you  <-- the only gate
        |
    3. create issue on upstream (with metadata)
    4. create worktree + branch (issue-<N>)
    5. implement (baseline screenshot if frontend)
    6. validate in browser (frontend only, 3-attempt cap)
    7. squash commits (explicit file staging)
    8. push to fork
    9. create PR on upstream (before/after screenshots, confidence)
   10. report URLs + confidence
```

**Frontend validation** (step 6) uses `agent-browser` to drive the change in a real browser -
golden path, edge cases, mobile viewport, accessibility basics.
If it finds a bug, it diagnoses and retries up to 3 times.
After 3 failures, it either scopes down (edge case) or stops the pipeline (core ask broken) rather
than shipping something broken.

**When the pipeline stops**, burst closes the upstream issue with an explanation so you don't end up
with orphaned issues.

## What burst doesn't do

Burst is deliberately narrow.
It trades coverage for speed - here's what it skips and why:

| Skipped | Why |
|---|---|
| Tests | Speed. Burst ships exactly what you described. Add tests in a follow-up or ask for them explicitly. |
| Planning / brainstorming | You already know the fix. If you don't, use `/plan` or `/brainstorm` instead. |
| Refactoring nearby code | Scope discipline. "While I'm here" changes are how small PRs become big ones. |
| CI status check | Burst reports the PR URL. You check CI before merging. |
| Worktree cleanup | You may need the worktree for PR review iterations. Clean up when you're done. |

## The PR it creates

Every burst PR includes:

- **Before/after screenshots** (frontend changes) - uploaded as a private gist, embedded in the PR body
- **Summary** - what changed and why
- **Test plan** - checked items for what burst verified, unchecked for what needs human review
- **Confidence** - honest self-assessment (high / medium / low) with a one-sentence explanation

```markdown
## Confidence
Medium. Logic change works in the happy path but I couldn't test the
empty-cart edge case because the dev server doesn't seed that state.
```

The confidence section is the most useful part.
It tells reviewers exactly where to focus instead of making them guess.

## Design principles

- **Gaps over fabrication.** If something is unclear, say so.
  A PR that notes "I'm unsure whether this covers RTL" is more useful than one that ships without checking.
- **Scope is sacred.** Burst ships what you described. Not more, not less.
  If the change wants to grow, that's a signal to stop and check with you.
- **Recommend, don't assume.** When your intent is ambiguous, burst proposes its read and lets you correct it.
  It never fills in blanks silently.
- **One gate, then go.** You confirm the plan once. After that, burst runs autonomously to completion.
  No mid-pipeline interruptions unless something is actually broken.

## License

[MIT](LICENSE).
