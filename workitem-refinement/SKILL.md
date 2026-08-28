---
name: Work Item Refinement
description: Routine for refining Azure DevOps work items (Features, Issues, Bugs). Fetches the work item, analyzes the codebase for technical context, and produces a structured description using the Problema / Solución / Contexto format. Does NOT create child tasks.
inclusion: manual
---

# Work Item Refinement Routine

When the user provides a work item ID, analyze it and produce a refined, structured description ready to be applied to the work item.

## Critical Rule

**NEVER write to Azure DevOps without explicit user confirmation.** Always show a full preview first and wait for the user to say "si", "dale", "aplicar", or "yes" before making any change. If the user does not confirm, do NOT proceed.

## Required Input

- **Work item ID** — mandatory. The skill will not proceed without it.

## Language

- Description output in **Spanish** (matching the team's convention).
- Keep domain terms and code identifiers in English (as they appear in the schema and codebase).

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

## Output Structure (mandatory format)

The refined description MUST follow this exact markdown structure:

```markdown
### PROBLEMA

- bullet point 1 describing a specific problem or symptom
- bullet point 2 (if applicable)

### SOLUCIÓN

**Hacer:**

- bullet point 1 describing a specific action to take
- bullet point 2 (if applicable)

**Aceptación:**

- ☐ verification criterion 1
- ☐ verification criterion 2

**No tocar:**

- constraint 1
- constraint 2

**Bordes:**

- edge case 1
- edge case 2

### CONTEXTO

| Qué | Dónde/Detalle |
|-----|---------------|
| item | location or detail |
```

Rules:
- **PROBLEMA**: describe observable symptoms from the user's perspective. Be specific — include entity names, screen names, error messages when known.
- **SOLUCIÓN / Hacer**: describe what needs to change. Reference specific classes, stored procedures, or configuration when the codebase analysis reveals them. Keep it actionable.
- **SOLUCIÓN / Aceptación**: checkboxes a dev can verify after implementing.
- **SOLUCIÓN / No tocar**: explicit constraints on what must remain unchanged.
- **SOLUCIÓN / Bordes**: edge cases discovered during codebase analysis — null values, empty collections, boundary conditions, concurrent operations.
- **CONTEXTO**: table with technical references — affected files, modules, tables, mappings.

---

## Steps

### 1. Retrieve the work item (~1 min)

1. Fetch the work item by ID using Azure DevOps MCP (action: `get`, expand: `All`).
2. Fetch comments (action: `list_comments`).
3. If attachments exist, note their URLs for embedding in the Contexto section.
4. If parent/child/related items exist, fetch their titles.

Record: Title, Description (current), State, Type, Assigned To, Iteration, Area Path, Tags, attachments, linked items.

### 2. Understand the request (~2 min)

From the existing description, comments, and attachments, extract:

- What is broken or missing (the problem).
- Who reported it and from where (origin entity, ticket reference).
- What the expected behavior should be.

**Format normalization:** If the existing description is in HTML format (contains `<div>`, `<span>`, `<ul>`, `<img>`, etc.), convert it mentally to markdown for analysis. When writing the refined description, ALWAYS output pure markdown. Azure DevOps supports markdown in the Description field.

If the description is too vague to produce a useful Problema section, ask the user ONE clarifying question.

### 3. Analyze the codebase (~5 min)

Based on the affected area:

1. **Locate relevant code** — search for entities, business classes, data access classes, stored procedures, and NHibernate (NH) mappings.
2. **Trace the execution flow** — understand how the current behavior works (API → Business → Data → SQL).
3. **Identify root cause** — what specifically causes the reported problem.
4. **Note affected areas** — other screens, modules, or integrations that touch the same code.

Use the project structure:
- Entities: `DataLayer/Sda.Aasi.Data.Entities*/`
- Data access: `DataLayer/Sda.Aasi.Data.NhContexts*/<Entity>Data.cs`
- NH mappings: `DataLayer/Sda.Aasi.Data.NhContexts*/Mappings/*.hbm.xml`
- Business: `BusinessLayer/Sda.Aasi.Business*/<Entity>Business.cs`
- API controllers: `Api/Sda.Aasi.Api/Areas/*/Controllers/`
- SQL: `DatabaseScripts/Procedures/`, `DatabaseScripts/Functions/`, `DatabaseScripts/Views/`

### 4. Generate the refined description (~3 min)

Write the description in the mandatory PROBLEMA / SOLUCIÓN / CONTEXTO format.

- Keep bullet points concise — one idea per bullet.
- Embed existing screenshot URLs from attachments using `![Image](url)`.
- Reference code artifacts by name (e.g., "`JournalItemBusiness.cs`", "`aasi_journalitem_import_validation`").

### 5. Show preview and STOP (mandatory)

**This step is a hard stop. Do NOT proceed to step 6 until the user explicitly responds.**

Format the response as:

1. The full refined description (raw markdown the user can read).
2. Link: `[Open Work Item #{ID}](https://dev.azure.com/sda-iatec/Sda.Aasi.Net/_workitems/edit/{ID})`
3. Last line: **"Aplicar al work item? (si / ajustar / cancelar)"**

**Do NOT call any Azure DevOps write API at this point. Wait for user input.**

### 6. Handle user response (only after explicit confirmation)

- **"si" / "yes" / "dale" / "aplicar"** → Update the work item description via MCP (action: `update`, path: `/fields/System.Description`, value: the refined description as markdown). Confirm: "Descripción actualizada en #{ID}."
- **"ajustar" / feedback** → Incorporate feedback, regenerate, show new preview (return to step 5).
- **"cancelar" / "no"** → "Cancelado — sin cambios."

---

## Quality Gate (check before presenting)

- [ ] Problema section describes observable symptoms, not implementation details.
- [ ] Solución section is actionable — a developer knows what to change.
- [ ] Contexto table includes all relevant technical references found in the codebase.
- [ ] Existing screenshots/attachments from the work item are preserved (embedded via URL).
- [ ] Format matches the ### PROBLEMA / ### SOLUCIÓN / ### CONTEXTO structure exactly.
- [ ] No preamble before the preview. No recap after.
- [ ] Last line = "Aplicar al work item? (si / ajustar / cancelar)".
