---
name: skills
description: >
  Guidelines for creating, organizing, and maintaining Claude Code skills in this project.
  Use this skill when authoring new SKILL.md files, updating existing skills, or deciding
  how to structure custom instructions for autonomous Claude agents.
---

# Skills Management Skill

## Overview

This directory defines the meta-rules for the skills system itself. A **Claude Code Skill** is a `SKILL.md` file that instructs Claude how to behave in a specific domain. Skills are registerable in Claude Code (`claude code skills`) and apply automatically when relevant context is detected.

This project's skills are designed to be **stack-agnostic and project-agnostic** — they apply to any project you work on, and Claude should apply them consistently.

---

## 1. What Skills Are (and Are Not)

### Skills ARE:
- Declarative instructions Claude should follow within a functional domain
- Reusable across projects and stacks
- The canonical source of truth for standards documented in `docs.md`
- Living documents — updated as conventions evolve

### Skills ARE NOT:
- Checklists to manually tick through
- Project-specific configurations (those belong in project repos)
- Replacements for tests, CI, or code reviews
- A substitute for judgment — Claude uses skills to inform decisions, not replace them

---

## 2. Current Skills Registry

| Folder | Skill Name | Scope |
|--------|------------|-------|
| `design/` | JenaLab Design System | Visual design, color, typography, layout, motion |
| `planning/` | Planning & Requirements | PRDs, user stories, acceptance criteria, UX flow |
| `architecture/` | Architecture | System structure, layering, API design, DB modeling |
| `code/` | Code Style & Quality | Per-language style, naming, quality rules |
| `skills/` | Skills Management | How to write and maintain skills |

---

## 3. When Claude Should Apply These Skills

Claude should activate the relevant skill(s) based on task context:

| Task Type | Apply Skill(s) |
|-----------|---------------|
| Creating UI, choosing colors/fonts | `design` |
| Writing PRD, user stories, screen spec | `planning` |
| Designing system structure, API, DB schema | `architecture` |
| Writing or reviewing code | `code` + `architecture` |
| Writing a new skill | `skills` |
| Building a full feature end-to-end | All applicable skills |

When multiple skills apply, Claude should synthesize them — they are complementary, not conflicting.

---

## 4. How to Write a New Skill

### SKILL.md Structure

Every `SKILL.md` must follow this format:

```markdown
---
name: skill-name
description: >
  One to three sentences describing exactly when Claude should apply this skill.
  Be specific about what actions/tasks trigger this skill.
---

# Skill Title

## Overview
2–3 sentence orientation. What domain does this skill cover?
What is the core philosophy or guiding principle?

## [Section 1 — Main Concept]
...

## [Section N — ...]
...

## Checklist
A final checklist Claude runs through before considering the task complete.
```

### Quality Standards for Skills

1. **Written in English.** Skills are instructions to Claude; English is the canonical author language.

2. **Actionable, not aspirational.** Every guideline must tell Claude what to DO or NOT DO. Avoid vague statements like "write clean code" without specifics.

3. **Includes concrete examples.** Show the right pattern AND the wrong pattern side by side wherever possible.

4. **Organized by what Claude needs to decide.** Group content by decision type, not by documentation tradition.

5. **No duplicate content.** If two skills cover adjacent topics, cross-reference rather than repeat.

6. **Versioned thinking.** Skills evolve. When a rule changes, update the skill. Old rules should not linger.

7. **Stack-specific rules in code/ and architecture/.** Design and planning skills stay stack-agnostic.

---

## 5. Skill Authoring Checklist

Before publishing a new or updated skill:

```
[ ] YAML frontmatter includes name and description
[ ] Description clearly states WHEN to activate this skill
[ ] Overview section explains the domain and core philosophy
[ ] Rules are actionable (DO / DO NOT, not "consider" or "try")
[ ] Concrete examples provided for non-obvious rules
[ ] Forbidden patterns explicitly called out
[ ] Final checklist section included
[ ] No content duplicated from another skill
[ ] Written in English
[ ] Tested by applying it to at least one real task
```

---

## 6. How Skills Interact with Project Context

When Claude starts a new project or task, it should:

1. **Identify which skills are relevant** based on the task description
2. **Read the applicable SKILL.md files** before generating any significant output
3. **Apply skills as default behavior** — not as an afterthought checklist
4. **Flag conflicts** between project-specific instructions and skill rules — project-specific always wins, but the conflict should be noted

### Priority Order
```
1. Explicit user instruction in the conversation (highest priority)
2. Project-level CLAUDE.md or project conventions
3. Skill rules from this skills system
4. Claude's general knowledge and judgment
```

---

## 7. Updating Skills

### When to Update a Skill

- A convention in `docs.md` changes
- A new language/framework is added to the stack
- A rule proves impractical or contradicts actual project needs
- A new pattern emerges that should be standardized

### How to Update

1. Edit the relevant `SKILL.md` in place
2. If the change is significant, also update `docs.md` as the source-of-truth reference
3. Note the change in `CHANGELOG.md` at the project root (if it exists)
4. Test the updated skill on a real task before confirming

---

## 8. Skill Naming and Organization

```
skills/
  design/
    SKILL.md         ← required
    examples/        ← optional: visual examples, preset configurations
  planning/
    SKILL.md
    templates/       ← optional: PRD, screen spec, user story templates
  architecture/
    SKILL.md
    decisions/       ← ADR files live here
  code/
    SKILL.md
  devops/
    SKILL.md
    scripts/         ← optional: setup scripts, CI templates
  skills/
    SKILL.md         ← this file (meta)
```

### Naming Rules for New Skills
- Folder name: `lowercase-hyphenated`
- Skill name in YAML frontmatter: `lowercase-hyphenated`
- SKILL.md: always exactly `SKILL.md` (uppercase, no variation)
- Supporting files: descriptive, lowercase, hyphenated

---

## 9. Documentation Maintenance

The skills system and `docs.md` must stay in sync:

```
docs.md                    → Human-readable reference (Korean + details)
skills/*/SKILL.md          → Machine-readable instructions for Claude (English + actionable)

If docs.md is updated → review and update relevant SKILL.md files
If a SKILL.md is updated → verify docs.md reflects the same intent
```

Both documents serve different audiences:
- `docs.md`: For human team members reading conventions
- `SKILL.md`: For Claude acting as an automated implementation agent

---

## 10. Anti-Patterns to Avoid When Writing Skills

| Anti-Pattern | Problem | Fix |
|---------|---------|-----|
| "Write good code" | Too vague — Claude cannot act on it | Specify naming rules, forbidden patterns, file structure |
| Duplicating exact content from another skill | Divergence when one is updated | Cross-reference: "See architecture/SKILL.md §2 for folder structure" |
| Stack-specific rules in planning/design skills | Makes skill non-reusable | Move to code/ or architecture/ |
| Checklist only, no reasoning | Claude won't prioritize correctly | Explain the "why" briefly for each major rule |
| Rules without forbidden patterns | Incomplete guidance | Always pair "do this" with "don't do this" |
| Passive voice rules ("should be considered") | Ambiguous — is this required or optional? | Use imperative: "Do X. Do NOT do Y. Prefer Z over W." |
