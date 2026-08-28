---
name: Build Work Item
description: End-to-end implementation routine. Given a work item ID, creates a branch from rc, implements the changes described in the work item, commits, pushes, and creates a Pull Request for dev review.
inclusion: manual
---

# Build Work Item Routine

Automates the full implementation cycle for an Azure DevOps work item: understand requirements, branch from `rc`, implement, commit, push, and open a Pull Request (PR).

## Required Input

- **Work item ID** — mandatory.
- **Repository context** — the skill assumes it runs inside a workspace that is a git clone of the target repo.

## Language

- Commit messages and PR title/description in **English**.
- Comments and explanations to the user in **Spanish** (matching team convention).
- Code identifiers always in English.

---

## Output Style: ADHD-Friendly Mode

All responses produced while executing this skill MUST follow these rules. No exceptions unless explicitly noted.

### Core Facts

1. Working memory is small. Anything not on screen is forgotten.
2. The friction between "got it" and "done it" is where work dies.
3. Starting is the hardest step. The first action must be obvious, small, and doable now.
4. Vague time estimates fail. Use concrete units.
5. Dopamine is scarce. Visible progress matters.

### Rules

1. **Lead with the next action.** First line = something the reader can do. Not context. Not a plan.
2. **Number multi-step tasks.** Each step is one bounded action. No step contains "and then" twice. Fewest steps that still work.
3. **End with one concrete next action.** Name ONE thing doable in under two minutes.
4. **Suppress tangents.** Finish the first issue. Offer the second as a separate question.
5. **Restate state every turn.** "Estamos en paso 3 de 5" — always explicit.
6. **Give specific time estimates.** "~15 min", "~2 min", not "some work".
7. **Make completed work visible.** Show what now works, in concrete terms.
8. **Matter-of-fact tone for errors.** No "Uh oh". State cause and fix directly.
9. **Cap lists at 5 items.** Beyond 5 → split into "ahora" vs "después".
10. **No preamble, no recap, no closing pleasantries.** Start with the answer. End when the answer is done.

### Forbidden

- Openers: "Great question," "Let me...", "I'll...", "Sure!", "Looking at your..."
- Closers: "Let me know if you need anything else," "Hope this helps," "Happy to clarify"
- Hedging adverbs: "perhaps," "might," "could possibly"
- Idioms or figurative phrases — use the literal action.

### Pre-Send Check

Before every response, delete:
1. First sentence if it announces what you are about to do.
2. Last sentence if it asks "anything else?" or recaps.
3. Any "by the way" sidebar.

Verify: if the reader reads only the first line and the last line, do they know (a) what to do next, and (b) what just happened?

---

## Steps

### 1. Fetch the work item (~1 min)

1. Retrieve the work item via Azure DevOps MCP (action: `get`, expand: `All`).
2. Fetch comments (action: `list_comments`) for additional context.
3. Record: ID, Title, Description, Type, State, Assigned To, Tags, linked items, attachments.

If the description lacks enough detail to implement (no Problema/Solución sections, no clear acceptance criteria), **stop and ask the user** whether to run the `workitem-refinement` skill first.

### 2. Plan the implementation (~3 min)

1. Parse the work item description — extract:
   - What needs to change (files, modules, SQL objects).
   - Acceptance criteria / expected behavior.
   - Explicit "do NOT touch" constraints.
   - Edge cases.
2. Analyze the codebase — locate affected files using project structure conventions:
   - Entities: `DataLayer/Sda.Aasi.Data.Entities*/`
   - Data access: `DataLayer/Sda.Aasi.Data.NhContexts*/<Entity>Data.cs`
   - NHibernate (NH) mappings: `DataLayer/Sda.Aasi.Data.NhContexts*/Mappings/*.hbm.xml`
   - Business: `BusinessLayer/Sda.Aasi.Business*/<Entity>Business.cs`
   - API controllers: `Api/Sda.Aasi.Api/Areas/*/Controllers/`
   - SQL: `DatabaseScripts/Procedures/`, `Functions/`, `Views/`
   - Frontend: `Aasi.Net.WebApp/src/app/modules/`
3. Present the implementation plan as a numbered task list (max 5 items; split into "ahora" vs "después" if more).
4. **Wait for user confirmation** before proceeding. Acceptable: "si", "dale", "go", "yes".

### 3. Create the feature branch (~1 min)

1. Branch name convention: `feature/{workItemId}-{short-slug}`
   - `short-slug` = first 4-5 meaningful words from the title, kebab-case, max 50 chars total.
   - Example: `feature/177179-fixed-asset-committee-action-real-state`
2. Ensure the local repo is clean. If dirty → **stop and ask** the user to stash or commit.
3. `git fetch origin`
4. `git checkout -b <branch-name> origin/rc`

### 4. Implement the changes (~5-30 min depending on scope)

1. Work through the plan from step 2, task by task.
2. Follow project conventions:
   - Match existing code style, indentation, naming.
   - Use existing libraries/patterns — do NOT introduce new dependencies without asking.
   - SQL: `SET QUOTED_IDENTIFIER ON / ANSI_NULLS ON / IF EXISTS DROP / CREATE` pattern.
   - NH data access: typed parameter setters (`SetInt32`, `SetAnsiString`, etc.).
3. Mark each task complete as you go (visible progress).
4. If an approach fails twice → stop, explain root cause, try a different approach.

### 5. Verify the implementation (~2 min)

1. Run build if available.
2. Review all modified files — no unintended changes, no debug artifacts.
3. Run relevant tests if a test runner is available.
4. Fix errors before proceeding.

### 6. Commit and push (~1 min)

1. Stage only implementation files: `git add <specific-files>`
2. Commit message:
   ```
   feat(#{workItemId}): <concise description>
   ```
   First line under 72 chars. Body if non-trivial (what and why).
3. `git push -u origin <branch-name>`

### 7. Create the Pull Request (~1 min)

1. Create PR targeting `rc` via CLI or Azure DevOps MCP.
2. PR title: `feat(#{workItemId}): <short description>` (under 70 chars).
3. PR description:
   ```markdown
   ## Summary
   <1-2 sentences>

   ## Changes
   - <file>: <what changed>

   ## Work Item
   AB#{workItemId}

   ## Testing
   - <what was verified>
   - <what needs manual verification>
   ```
4. Link the work item to the PR.
5. Report to user: branch name, PR URL, summary of what was done.

---

## Critical Rules

1. **Never push to `rc` or `main` directly.** Always feature branch.
2. **Never proceed past step 2 without user confirmation.**
3. **Never commit unrelated changes.** Stage specific files only.
4. **If repo is dirty at step 3, stop and ask.**
5. **If work item description is insufficient, stop and ask** — suggest `workitem-refinement` first.

## Error Recovery

- **Build fails:** Fix, re-verify, proceed.
- **Merge conflict on push:** Inform user. Suggest rebase from `origin/rc`. Never force-push.
- **No clear solution in WI:** Ask ONE clarifying question.
- **Multiple repos affected:** One at a time. Ask which first.

## Quality Gate (check before creating PR)

- [ ] All acceptance criteria addressed.
- [ ] Build passes (if verifiable).
- [ ] No unrelated file changes.
- [ ] Commit message follows convention.
- [ ] Branch name follows convention.
- [ ] PR description complete with work item link.
- [ ] "Do NOT touch" constraints respected.
