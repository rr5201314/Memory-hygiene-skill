---
name: memory-hygiene
description: "Memory architecture operations manual: load when cleaning memory, migrating to skills, creating project skills, or managing skill structure."
tags: [memory, hygiene, hermes-agent, maintenance, architecture]
---

# Memory Hygiene — Memory Architecture Operations Manual

> English | [中文](./SKILL.zh-CN.md)

## ⚠️ Pre-Load Check

This skill operates on a layered memory architecture. **Before executing any operation**, check whether memory (target='memory') already contains the following architecture declaration:

> Memory architecture has three layers: LV1=memory/user/SOUL, LV2=project and user details (stored as skills, description as summary, load on demand), LV3=procedural skills (skills). Skills contain two types of knowledge: declarative ("what is it", e.g. project background, user info) and procedural ("how to do it", e.g. workflows, pitfalls). Scan skills_list before tasks to decide what to load, don't load blindly. See memory-hygiene skill for architecture details.

**If this text is missing from memory**, write it into memory before doing anything else. Otherwise the agent won't know about the layered architecture and this skill's rules won't take effect.

---

> Architecture definitions (what LV1/LV2/LV3 are, how they map to Hermes) live in the above LV1 memory entry, injected every turn, not repeated here.
> This skill only covers **how to operate** and **what constraints apply**.

---

## Skill Management Rules

### Description Writing

The description is the only basis for the agent to decide whether to load a skill. It must:

1. **Be semantically retrievable**: include keywords so the agent can match it by task content
2. **State trigger conditions**: write clearly "when should this skill be loaded"
3. **Be brief**: one or two sentences, not an abstract

```
# Good description
description: "XXX project pitfalls. Load when editing YYY data or ZZZ module."

# Bad description
description: "This skill records some notes and development experiences about the XXX project"
```

### Deduplication

Before writing, check existing skills via `skills_list`. If a related skill exists, `patch` to append — don't create a new one.

### Project Memory Folder Structure

Large projects' LV2 memory shouldn't be crammed into one skill. Split into **one category folder + multiple sub-skills**:

```
skills/
└── {category}/                    # Project-specific category
    ├── {project}-overview/        # Project overview
    │   └── SKILL.md              # Background, tech stack, repo, feature list
    ├── {project}-module-a/        # Module A
    │   └── SKILL.md              # Module intro, constraints, rules, pitfalls
    ├── {project}-module-b/        # Module B
    │   └── SKILL.md
    └── ...
```

**Example: A Typical App Project**

```
skills/{category}/
├── {project}-overview/SKILL.md        → description: "Project overview. Load for background, tech stack."
├── {project}-module-a/SKILL.md       → description: "Module A spec. Load when editing XXX data."
├── {project}-module-b/SKILL.md       → description: "Module B rules. Load when writing YYY feature."
└── {project}-integration/SKILL.md    → description: "XXX integration. Load when connecting to YYY service."
```

**Splitting Principles**:

- Each sub-skill covers an **independent module or work scenario**
- Sub-skill description states clearly **what the module is + when to load**
- Overview skill holds project-level info (background, tech stack, repo, feature list), no granular details
- Pitfalls stay with their module — don't create standalone "pitfalls" files

---

## Cleanup Workflow

Trigger when user requests cleanup, or memory usage > 80%:

### Step 1: Per-Item Review

Ask three questions for each memory entry:

1. Does this need to be known in every session? → **Yes** = LV1, keep
2. Is this only useful in specific scenarios? → **Yes** = LV2/LV3, move to skill
3. Is this outdated/duplicate? → **Yes** = delete

### Step 2: List for User Confirmation

```
**Can delete**:
- #X description (reason)

**Suggest moving to skill**:
- #X description → suggest moving to xxx skill

**Duplicates to merge**:
- #X and #Y overlap
```

**Wait for user confirmation** before acting. Don't decide on your own.

### Step 3: Batch Execute

- Items moving to skill: write to skill first, then use memory(operations=[...]) to delete/replace in one batch (two steps, don't skip either)
- Goal: LV1 memory usage < 50%, save space for what truly matters

---

## Common Pitfalls

1. **Old entry text matching fails**: memory remove's old_text must be an **exact substring** of the entry. If remove fails, use memory(operations=[]) to get current_entries and retry with the full text.

2. **Don't accidentally add entries during cleanup**: "Let me check the memory" doesn't mean you should add anything. Read the MEMORY section from the system prompt directly, don't call memory(action='add').

3. **Don't forget to delete memory after creating skill**: Transfer is a two-step operation (write skill + delete memory). Missing the delete means it wasn't actually cleaned up.

4. **Batch operations are atomic**: If any entry in the operations array fails, the entire batch is rejected. Ensure every old_text is exact.

5. **Don't let LV1 bloat**: Review LV1 entries during every cleanup, ask "does this really need to be known every time?". The leaner LV1 is, the higher the proportion of tokens spent effectively per turn.

6. **Memory and skill must stay in sync**: When a user preference changes, it may exist in both memory and a skill example. Only changing memory without changing skill = the old rule comes back next time the skill loads. Grep the skills directory after editing memory to confirm no stale references.

7. **Don't use user-mandated entries as negative examples**: When writing "shouldn't store" examples in a skill doc, confirm the example isn't a LV1 entry the user explicitly requested. Otherwise skill and memory contradict each other and the agent can't reconcile them.

---

## Self-Check Checklist

After executing this skill, verify:

- [ ] All LV1 entries are in memory (target='memory' or 'user'), nothing missing
- [ ] Every LV1 entry is "must know every session", no on-demand content mixed in
- [ ] Memory has no project details, tool pitfalls, or procedures (those should be skills)
- [ ] Memory and skill content are consistent, no one-side-only updates
- [ ] All LV2/LV3 skill descriptions have clear trigger conditions
- [ ] Project skills follow folder structure, sub-modules independent, pitfalls with modules
- [ ] No functionally overlapping skills in skills_list

Fix issues immediately, don't wait for the user to ask.
