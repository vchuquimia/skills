---
name: Stakeholder Alignment
description: Routine for generating executive-level stakeholder communications (in English) about issues or change requests, based on an Azure DevOps work item. Output is email-ready for NAD managers, with SAD team ownership of resolution.
inclusion: manual
---

# Stakeholder Alignment Routine

When the user provides a work item ID, generate an executive-level email communication in English for a managerial stakeholder at NAD. The email informs about a problem or requests approval for a change.

## Required Input

- **Work item ID** — mandatory. The skill will not proceed without it.

## Writing Style (enforced on ALL output)

1. **Lead with the action.** First line = result or next step.
2. **Number multi-step work.** Each step = one bounded action.
3. **End with one concrete next action.**
4. **No tangents.**
5. **Restate progress every turn.**
6. **Specific time estimates.**
7. **Make completed work visible.**
8. **Matter-of-fact errors.** No "Uh oh." State cause → fix.
9. **Cap lists at 5.**
10. **No preamble, no recap, no pleasantries.**

### Pre-send check

Delete:
- First sentence if it announces what you're about to do.
- Last sentence if it asks "anything else?" or recaps.
- Any hedging adverb.
- Any idiom — replace with literal action.

---

## Output Language

**English only.** The email will be sent to an NAD (North American Division) manager. All content — subject line, body, closing — must be in professional English.

---

## Email Writing Style (voice and tone)

The email must match this specific writing style. Study the following characteristics and apply them consistently:

### Subject line
- Format: `CR - {Short descriptive title}` for change requests, `Issue - {Short descriptive title}` for problem notifications.
- No work item number (the stakeholder has no access to Azure DevOps).
- Short, plain English, no brackets or tags beyond `CR -` or `Issue -`.

### Opening
- Use the appropriate time-of-day greeting based on when the email is generated:
  - Before 12:00 → `Good morning Michael,`
  - 12:00–17:59 → `Good afternoon Michael,`
  - 18:00+ → `Good evening Michael,`
- Use the stakeholder name provided by the user, or default to "Michael".
- Never: "Hi Team," or "Dear Michael," or any other greeting style.

### Body structure and voice
- **Conversational and direct.** Write as one colleague explaining to another. Not corporate-formal, not casual-sloppy.
- **Start with origin/context.** First sentence tells where the request came from or what triggered the issue. Example: "There is a request from North Brazilian Union NBU to..."
- **Explain current behavior.** Describe what happens today in plain language. Use "Currently..." to anchor.
- **Explain the gap or pain.** One or two sentences on why the current behavior is a problem. Use concrete examples when possible (entity names, screen names, real scenarios).
- **List affected areas.** Use a simple bullet list or inline list of screens/modules affected. No technical jargon — use screen names the stakeholder would recognize.
- **Propose the change.** State what would change. Be specific but non-technical.
- **Offer an alternative.** Always offer a safe fallback: "If you prefer to keep it as it is, we can create a configuration that when enabled will behave as requested. By default this configuration will be disabled to preserve the actual behavior."
- **Close with invitation.** Always: "Would you agree to change this behavior?" or "Please share your thoughts with us about this request."

### Closing
- Always: `Regards.` (with period, no name block, no "SAD Development Team" signature).
- Never: "Best regards," or "Thanks," or multi-line signature blocks.

### Things to NEVER include
- Work item numbers or Azure DevOps links (stakeholder has no access).
- Technical terms: no "sprint", "allocation config entity", "UI component", "API endpoint".
- Priority/severity labels.
- Estimated effort in sprints or story points.
- Risk level classifications.
- Any formatting like `**bold**` or bullet markers — the email is plain text.

### Tone rules
- No drama. No urgency language unless truly urgent.
- No hedging ("I think maybe we could possibly..."). Be direct but polite.
- Acknowledge the stakeholder's context: "I know that NAD user does not have many open periods but..."
- Use "we" for the team, "you" for the stakeholder.

### Reference example

```
Subject: CR - Default selected period / actual period

Good morning Michael,

There is a request from North Brazilian Union NBU to select the "actual" period, meaning by the time I am sending this email to you we are in May, so May is actual period. According to this request May should be selected by default. Currently the default selected period is the last opened period.

I know that NAD user does not have many open periods but in SAD sadly that is a kind of regular practice, so always the default selected period is a period way up front. (September in the case of NBU).

This change would affect the following screens:
New Journal.
GL.
Trial Balance and TB Analysis Tool.
Journal Report.

Would you agree to change this behavior? If you prefer to keep it as it is, we can create a configuration that when enabled will behave as requested. By default this configuration will be disabled to preserve the actual selection.

Please share your thoughts with us about this request.

Regards.
```

---

## Key Context

- **SAD** = South American Division (the development team resolving the issue).
- **NAD** = North American Division (the stakeholder's division).
- When reporting a **problem**: inform the manager that the SAD team is handling resolution and will share findings/fix with the NAD team.
- When requesting a **change**: present the problem first, then the proposed solution, and ask for alignment/approval.

---

## Steps

### 1. Retrieve the work item (~1 min)

1. Fetch the work item by ID using Azure DevOps MCP (action: `get`, expand: `Relations`).
2. Fetch comments (action: `list_comments`).
3. If related/child/parent items exist, fetch their title and state.

Record: Title, Description, State, Type, Assigned To, Iteration, linked items, tags.

### 2. Classify communication type (~30 sec)

Based on the work item content, determine:

- **Problem notification** — The work item describes a bug, incident, or operational issue. The stakeholder needs to be informed.
- **Change request** — The work item proposes a change to behavior, configuration, or feature. The stakeholder needs to approve or align.

If ambiguous, ask the user: "Is this a problem notification or a change request?"

### 3. Gather context from the codebase (~3 min)

1. Identify the affected system area.
2. Search for related entities, tables, stored procedures, or configuration.
3. Understand the current behavior and the impact.

Stop when you can answer: "What is affected, what is the impact, and what is the path forward?"

### 4. Generate the email (~3 min)

Write the email following the voice/tone/structure defined in the "Email Writing Style" section above. Use the reference example as your north star.

#### For PROBLEM NOTIFICATION:

Structure (adapt naturally, not rigidly):

1. Opening: `Good morning/afternoon/evening {Name},` (based on current time of day)
2. What happened: one sentence describing the issue and where it was detected (entity, screen, module).
3. Current impact: who is affected and how, in plain language.
4. What we are doing: state that the SAD team is working on a resolution and will share findings with the NAD team.
5. Affected areas: list screens/modules in plain text (no bullets with dashes — use line breaks with periods or commas).
6. Closing: "We will keep you updated on progress. Regards."

#### For CHANGE REQUEST:

Structure (adapt naturally, not rigidly):

1. Opening: `Good morning/afternoon/evening {Name},` (based on current time of day)
2. Origin: where the request comes from (which entity, union, or user group).
3. Current behavior: "Currently..." — explain what happens today.
4. The gap: why current behavior is a problem. Use concrete examples.
5. Affected areas: list screens/modules that would change.
6. The ask: "Would you agree to change this behavior?"
7. Safe fallback: "If you prefer to keep it as it is, we can create a configuration that when enabled will behave as requested. By default this configuration will be disabled to preserve the actual behavior."
8. Invitation: "Please share your thoughts with us about this request."
9. Closing: `Regards.`

**Default stakeholder name:** Michael (unless the user specifies otherwise).

### 5. Present preview to user

Format the response exactly like this:

1. First line: "Email draft ready. Type: {Problem Notification | Change Request}."
2. The full email (this IS the preview).
3. Link: `[Open Work Item #{ID}](https://dev.azure.com/sda-iatec/Sda.Aasi.Net/_workitems/edit/{ID})`
4. Last line: **"Ready to send? (yes / adjust / cancel)"**

Do NOT send anything. Wait for explicit user confirmation or adjustment.

### 6. Handle user response

- **"yes" / "send" / "ok" / "dale"** → Confirm: "Email finalized. Copy the text above and send to the stakeholder."
- **"adjust" / feedback** → Incorporate feedback, regenerate, show new preview.
- **"cancel" / "no"** → End. "Cancelled — no email generated."

Optionally, after confirmation ask: **"Add comment to work item with stakeholder communication summary? (yes / no)"**

If yes → Add a comment via MCP (action: `add`) with:
```
**Stakeholder Alignment:** Email sent to NAD management on {today's date}.
Type: {Problem Notification | Change Request}
Summary: {One sentence summary of what was communicated.}
```

### 7. Handle acceptance confirmation

When the user indicates the request was accepted (trigger words: "aceptado", "accepted", "approved", "aprobado", "le pareció bien", "dijo que sí"), perform both actions on the specified work item(s):

1. **Add tag `SA OK`** — via MCP update (path: `/fields/System.Tags`, append `;SA OK` to existing tags).
2. **Add comment** — via MCP (action: `add`) with:
```
**Stakeholder Alignment:** Request accepted by NAD via email on {today's date}.
```

Confirm: "Tag `SA OK` added and acceptance comment posted on #{ID}."

---

## Quality Gate (check before sending)

- [ ] Communication type (problem vs change) is correctly identified.
- [ ] Email reads like the reference example — conversational, direct, colleague-to-colleague.
- [ ] No work item numbers, no Azure DevOps links, no technical jargon.
- [ ] For problems: SAD ownership stated, NAD sharing promised.
- [ ] For changes: current behavior explained BEFORE the proposed change, safe fallback offered.
- [ ] Subject line starts with `CR -` or `Issue -`, no brackets, no IDs.
- [ ] Opens with "Good morning/afternoon/evening {Name}," (matching time of day) — closes with "Regards."
- [ ] Plain text feel — no bold markers, no bullet dashes, no formatting artifacts.
- [ ] First line of response = result or action.
- [ ] Last line of response = one thing to do next.
