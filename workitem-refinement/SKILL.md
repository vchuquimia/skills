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

## Output Structure (mandatory format)

The refined description MUST follow this exact markdown structure:

```markdown
## Problema

- bullet point 1 describing a specific problem or symptom
- bullet point 2 (if applicable)

## Solución

- bullet point 1 describing a specific action to take
- bullet point 2 (if applicable)

## Casos Borde

Un caso borde es una situación límite o poco frecuente que el sistema debe manejar correctamente — por ejemplo: valores nulos, listas vacías, registros duplicados, permisos insuficientes, o condiciones de carrera entre operaciones concurrentes.

- bullet point describing an edge case the developer should handle
- bullet point describing another edge case (if applicable)

## Contexto

- supporting information: affected screens, entities, modules, code paths
- links to emails, attachments, or related work items if available
- screenshots (embed existing attachment URLs from the work item)
```

Rules:
- **Problema**: describe observable symptoms from the user's perspective. Be specific — include entity names, screen names, error messages when known.
- **Solución**: describe what needs to change. Reference specific classes, stored procedures, or configuration when the codebase analysis reveals them. Keep it actionable but not prescriptive about implementation details.
- **Casos Borde**: list edge cases discovered during codebase analysis — situations that could break or produce unexpected results if not handled. Think: null values, empty collections, duplicate records, concurrent operations, missing permissions, boundary values (max int, empty string, zero-length periods).
- **Contexto**: provide technical and domain context that helps the developer understand the full picture — affected code paths, related modules, screenshots, and references.

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

**Format normalization:** If the existing description is in HTML format (contains `<div>`, `<span>`, `<ul>`, `<img>`, etc.), convert it mentally to markdown for analysis. When writing the refined description, ALWAYS output pure markdown. Azure DevOps supports markdown in the Description field — the `multilineFieldsFormat` will be set to `"markdown"` automatically when the value is sent as markdown text.

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

Write the description in the mandatory Problema / Solución / Contexto format.

- Keep bullet points concise — one idea per bullet.
- Embed existing screenshot URLs from attachments using `![Image](url)`.
- Reference code artifacts by name (e.g., "`JournalItemBusiness.cs`", "`aasi_journalitem_import_validation`").

### 5. Show preview and STOP (mandatory)

**This step is a hard stop. Do NOT proceed to step 6 until the user explicitly responds.**

Format the response as:

1. A header: `### Preview de descripción refinada`
2. The full refined description in a markdown code block (so the user sees the raw markdown that will be saved).
3. Link: `[Open Work Item #{ID}](https://dev.azure.com/sda-iatec/Sda.Aasi.Net/_workitems/edit/{ID})`
4. Last line: **"Aplicar al work item? (si / ajustar / cancelar)"**

**Do NOT call any Azure DevOps write API at this point. Wait for user input.**

### 6. Handle user response (only after explicit confirmation)

- **"si" / "yes" / "dale" / "aplicar"** → NOW update the work item description via MCP (action: `update`, path: `/fields/System.Description`, value: the refined description as **markdown**). Always send markdown format. Confirm: "Descripción actualizada en #{ID}."
- **"ajustar" / "adjust" / feedback** → Incorporate feedback, regenerate, show new preview (return to step 5). Do NOT save.
- **"cancelar" / "cancel" / "no"** → End. "Cancelado — sin cambios."
- **Any other response** → Treat as adjustment feedback, regenerate, return to step 5.

---

## Quality Gate (check before presenting)

- [ ] Problema section describes observable symptoms, not implementation details.
- [ ] Solución section is actionable — a developer knows what to change.
- [ ] Contexto section includes all relevant technical references found in the codebase.
- [ ] Existing screenshots/attachments from the work item are preserved (embedded via URL).
- [ ] Format matches the ## Problema / ## Solución / ## Contexto structure exactly.
- [ ] No preamble before the preview. No recap after.
- [ ] First line of response = the refined description.
- [ ] Last line = "Aplicar al work item? (si / ajustar / cancelar)".
