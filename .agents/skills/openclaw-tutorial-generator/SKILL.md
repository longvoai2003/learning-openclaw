---
name: openclaw-tutorial-generator
description: Generate OpenClaw tutorial and curriculum markdown files in the exact Theory→Practice format used in this project. Use when the user asks to write, expand, or fix a phase document, section, project, or theory check.
---

## Purpose
You generate OpenClaw curriculum markdown documents that match the established format exactly. Every section must follow the Theory → Practice structure. You never write commands you have not verified against docs.openclaw.ai or the GitHub repo.

## Document Structure Template

Every phase document uses this exact skeleton:

```markdown
# Phase N: [Title] — [Subtitle]

## Phase Overview

**Big-picture goal:** [One sentence describing what the learner achieves by the end.]

**What you'll understand:**
- [Concept 1]
- [Concept 2]

**What you'll be able to do:**
- [Skill 1]
- [Skill 2]

---

## What You Need Before This Phase

- [ ] [Prerequisite 1]
- [ ] [Prerequisite 2]

---

## N.N [Section Title]

### Part A — Theory & Conceptual Understanding

#### [Subsection heading]

[Explanation prose. Use analogies. Explain WHY before HOW.]

#### The Analogy: [Name]

[Single clear analogy explaining the concept.]

#### Theory Check

1. [Question testing conceptual understanding]
2. [Question testing the analogy]
3. [Question testing edge case]

---

### Part B — Hands-On: [Action Title]

**Step 1:** [Clear imperative. What the user does.]

```bash
[verified command]
```

[What to expect / what success looks like]

**Step 2:** ...

#### Common Mistakes to Watch For

- **[Mistake name]:** [What it is and how to fix it]

---

## Phase N Projects

---

### Project N: "[Name]" — [Subtitle]

#### Theory Recap
[Which sections this project applies.]

#### What You'll Build
[Concrete deliverable in one paragraph.]

#### What You'll Learn
- [Learning outcome 1]

#### Prerequisites
- [What must be done first]

#### Step-by-Step Build Guide

1. **[Step]:**
   ```bash
   [command]
   ```

#### How to Test It
- ✅ [Verifiable success criterion]
- ✅ [Another criterion]

#### Common Pitfalls
- **[Pitfall]:** [Explanation and fix]

#### Stretch Goals
1. [Optional harder challenge]

---

## Phase N — End of Phase Review

### Conceptual Recap Questions

1. [Deep question requiring synthesis]

### Practical Consolidation Challenge

**"[Challenge Name]"**: [One paragraph describing the capstone task that combines all phase concepts.]
```

## Writing Rules

### Tone and style
- Explain WHY before HOW in every section
- Every command gets a one-line explanation of what it does
- Use analogies for every major concept — label them "**Analogy:**"
- Avoid passive voice in instructions ("Run this command" not "This command should be run")
- Theory Check questions must test understanding, not just recall

### Commands and config
- Only write commands you have verified via the openclaw-researcher skill
- Every JSON snippet must be valid (no trailing commas, correct nesting)
- Always show the minimal config first, then expand
- Always include the restart step after config changes: `openclaw gateway restart`

### Projects
- Every project must have a concrete, testable deliverable
- The "How to Test It" checklist must use ✅ bullets
- Common Pitfalls must include the fix, not just the problem
- Stretch Goals should be genuinely harder, not just "do it again"

## Section-Specific Guidance

### For channel sections (Phase 2)
Always include: prerequisite accounts, token/credential acquisition steps, config snippet, restart, verification command, and a test from the actual channel.

### For memory/session sections (Phase 3)
Always distinguish: what survives `/new` (memory) vs. what does not (session history). Include a test that proves persistence.

### For skills sections (Phase 4)
Always include: the SKILL.md file contents, the directory path, the restart step, and a test prompt that exercises the skill.

### For security sections (Phase 5/7)
Always include: what the setting protects against (threat model), the config, a test that proves the protection works, and a rollback path.
