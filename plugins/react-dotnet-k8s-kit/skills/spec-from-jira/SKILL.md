---
name: spec-from-jira
description: Use when given a Jira issue key (e.g. PROJ-123) to turn into a SPEC.md for orchestrate-work. Fetches the issue and its context through the Jira/Atlassian MCP, grounds it in the code, asks the user only the questions that matter, and writes a testable spec scoped to the right components.
---

# Build a spec from a Jira item

The output is a spec that the `orchestrate-work` skill can build from without coming back to ask. Every requirement must be **testable**, **assigned to a component**, and **traceable** to the ticket or to a user answer.

Argument: a Jira key, e.g. `PROJ-123`. If none is given, ask for one.

## 1. Locate the workspace
- Find `WORKSPACE.md`: check the current directory first, then parent directories. It lists the components (repos/folders), the specs directory, and the contracts.
- If there's no `WORKSPACE.md`, work out the single component from the current repo and offer to create `WORKSPACE.md` from the `orchestrate-work` template after the spec is done.
- Output path: `<specsDir>/<KEY>/SPEC.md`, where `specsDir` comes from WORKSPACE.md and defaults to `specs/`. If the file already exists, read it and **update** it (bump `spec_version`, keep the answered questions) rather than overwriting it blindly.

## 2. Fetch the ticket
- Find the Jira tools: search the available tools for "jira" or "atlassian" (e.g. `ToolSearch` with "jira issue"). Typical tools fetch an issue, search with JQL, read remote links, and read Confluence pages. Some need a cloud/site id first, so get that from the accessible-resources tool if needed.
- **No Jira tools available:** say so in one line, and ask the user to connect the Atlassian/Jira MCP or paste the ticket's description, acceptance criteria and key comments. Continue with whatever they give you.
- Fetch:
  1. **The issue**: summary, type, status, priority, labels, components, fix version, description, acceptance criteria (often a custom field or a section of the description), and the attachment names.
  2. **Comments**: read all of them. Later comments override earlier ones and the description. Note any decisions.
  3. **Hierarchy**: the parent or epic (summary and description only) and the subtasks (summary and status).
  4. **Links**: summaries of linked issues ("blocks", "is blocked by", "relates to", "duplicates"). Fetch descriptions only for blockers and for anything the description depends on.
  5. **Remote links**: fetch linked Confluence pages if a Confluence tool exists. Otherwise list them under Sources.
- Keep raw ticket text out of the spec unless it's quoted as a source. Rewrite it into clear requirements.

## 3. Ground it in the code (lightweight)
- From the ticket and WORKSPACE.md, decide which components are likely affected: frontend, backend service(s) or gitops.
- For each one, launch an `Explore` agent with a narrow question. Examples: "Which endpoints/handlers deal with <entity>, and what do their request/response DTOs look like?", "Which pages/components render <feature>?", "Is there an existing feature flag mechanism?" Ask for file paths and a 10-line summary each.
- Use what they find to make the spec concrete: real endpoint paths, entity names and screens. Note **current behaviour** where the ticket changes it.
- Don't design the implementation. The spec says **what** and **where**, and leaves **how** to the engineers.

## 4. Gap analysis
Check each item. For every gap, decide whether it's **resolvable** from the ticket or code (write it down), **low-risk** (make an assumption and record it), or **material** (ask the user).

| Area | Must be clear |
| --- | --- |
| Goal | The problem and the outcome, and who benefits |
| Scope | What's in, what's explicitly out, and which components |
| Users and permissions | Roles, who can do what, and behaviour for unauthorised users |
| Acceptance criteria | Each is testable, with inputs, expected outcomes and edge cases |
| API | New or changed endpoints, request/response, error cases, backward compatibility |
| Data | New or changed entities/columns, migrations, backfill, retention |
| UI | Screens, states (loading/empty/error/forbidden), validation, copy, accessibility |
| Non-functional | Performance targets, audit/logging, security, localisation |
| Rollout | Feature flag? Which environments, in what order? Config or secrets needed? |
| Dependencies | Other teams, services or tickets that must land first |
| Done | How we'll know it works (tests, demo, metrics) |

Material means a wrong guess would change the API, the data model, the permissions or the scope, or cause rework across components.

## 5. Ask (only if needed)
- Use `AskUserQuestion`, with up to 4 questions per round and **at most 2 rounds**.
- Make each question multiple choice. Put your recommended option first and mark it "(Recommended)", based on the ticket, the code and common practice. The user can always type their own answer.
- Ask about the biggest-impact gaps first. Everything else becomes an assumption under §12 Decisions and assumptions, or an item under §13 Open questions.
- If the user isn't there (an unattended run), don't block. Take the recommended options, mark them `assumed`, and list them first in the hand-off message.

## 6. Write the spec
- Use `templates/SPEC.template.md` (in this skill's folder) and fill every section. Write "None" where something doesn't apply, rather than deleting the section.
- Requirement IDs are `FR-1`, `FR-2` and so on, stable within the spec. Each one has a **component** (a name from WORKSPACE.md), a **source** (a Jira AC number, a comment, or a user answer), and **acceptance criteria** in Given/When/Then form.
- Map every Jira acceptance criterion to at least one FR. List the mapping in the table in §4.
- §11 Delivery plan hints should say the order (usually contract, then backend, then frontend, then deploy), what can run in parallel, and whether any component can be skipped.
- Keep it concise. Sentences should be under 25 words, and the spec should rarely run past about 250 lines for a single story. For an epic, write one spec per story, or group FRs by story with clear headings.

## 7. Self-check before handing off
- [ ] Every Jira AC maps to an FR, and every FR has a component and Given/When/Then.
- [ ] The components listed in the frontmatter match the components named in the FRs.
- [ ] API changes say whether they're backward compatible. Breaking changes have a plan.
- [ ] Data changes say whether a migration or backfill is needed, and that rollouts are safe while old and new versions run side by side.
- [ ] Config, secrets and deployment changes are listed per environment.
- [ ] No implementation detail that constrains engineers without a reason.
- [ ] Each open question names who can answer it.

## 8. Hand off
Reply with:
- the spec path, and a 2–3 line summary (FR count, the components in scope)
- any assumptions made without the user's answer
- open questions, if any
- the next step: "Run the orchestrate-work skill with `<path>/SPEC.md`", optionally limited to components (e.g. "backend only")

Write to Jira (a comment, a label, a link) **only if the user asks**. If they do, post a short comment with the spec summary and its location.
