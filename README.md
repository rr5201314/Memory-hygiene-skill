# Memory Hygiene — Hermes Agent Layered Memory Architecture Skill

> English | [中文](./README.zh-CN.md)

> Solves structural problems in Hermes Agent's memory system: flat injection of all memories, no on-demand loading, project details polluting every turn's context.

## Pain Points

Hermes memory only has two layers: `memory` (agent notes) and `user` (user profile), all of which are **force-injected into every turn's system prompt**. This causes three problems:

**1. Noise Pollution**

Project details, tool pitfalls, environment configs — information only needed in specific scenarios — consumes tokens every turn. For example, your project's data editing spec is injected when you're writing papers, browsing docs, or debugging code — wasted tokens that also distract the model.

**2. No "Summary + On-Demand Full Text" Mechanism**

Ideally, the agent should only see a short index of "you have these projects and skills", loading details on demand. But Hermes memory doesn't support this — it's all or nothing.

**3. Uncontrolled Memory Bloat**

Task progress, completion records, temporary findings pile up constantly. The 4,000 character limit fills up fast. Without a layered concept, users don't know what to keep, what to move, or what to delete.

## Solution

Leverage Hermes's existing mechanisms as a workaround, splitting memory into three layers:

| Layer | Responsibility | Injection Timing | Hermes Implementation |
|-------|---------------|------------------|----------------------|
| **LV1** | User basic info, personality, global constraints | **Every turn** system prompt | `memory`(target='memory' + 'user') + `SOUL.md` |
| **LV2** | Project context, user details, tool environment | **On demand** | Skill (description as summary) |
| **LV3** | Procedures, pitfalls, workflows | **On demand** | Skill (same) |

**Key Insight**: `skills_list` returns all skills' name + description, which naturally serves as LV2/LV3's "summary index". The agent scans `skills_list` then loads via `skill_view` on demand — equivalent to "summary injection + on-demand full text".

### Pseudo-Progressive Disclosure Architecture

Hermes has no native "summary injection + on-demand full text" mechanism, but `skills_list` + `skill_view` combine into a **pseudo-progressive disclosure** system:

```
┌─────────────────────────────────────────────────────┐
│                    Every Turn's System Prompt          │
│                                                         │
│  LV1: memory + user + SOUL.md  ←  Always present       │
│                                                         │
│  skills_list Index (auto-injected, zero extra cost):    │
│  ┌───────────────────────────────────────────────────┐ │
│  │ name: myapp-overview                              │ │
│  │ desc: Project overview. Load for background/stack. │ │
│  ├───────────────────────────────────────────────────┤ │
│  │ name: myapp-data-model                            │ │
│  │ desc: Data model spec. Load when editing schema.   │ │
│  ├───────────────────────────────────────────────────┤ │
│  │ name: tool-setup-pitfalls                         │ │
│  │ desc: Tool setup pitfalls. Load when configuring.  │ │
│  └───────────────────────────────────────────────────┘ │
│                                                         │
│  ↑ Agent scans index, decides what to load by semantics │
└─────────────────────────────────────────────────────┘

          ↓ skill_view(name="myapp-data-model") ↓

┌─────────────────────────────────────────────────────┐
│              SKILL.md Full Text (on demand)             │
│                                                         │
│  Data Model Spec                                        │
│  ├── Users and Orders tables must not be directly JOINed│
│  ├── Migration scripts must be reversible               │
│  ├── New columns must have default values               │
│  └── ...                                                │
└─────────────────────────────────────────────────────┘
```

**Why "Pseudo" Progressive Disclosure**:

Real progressive disclosure is a UI pattern — users click to expand details. Hermes has no such UI, but the effect is equivalent:

| Progressive Disclosure | Hermes Equivalent |
|----------------------|-------------------|
| Layer 1: Summary/Index | `skills_list` returns name + description |
| Layer 2: Full details | `skill_view` loads full SKILL.md |
| User decides to expand | Agent decides by task semantics whether to `skill_view` |

**Cost**: One `skills_list` call (returns all skills' name + description, ~a few hundred tokens). Negligible compared to injecting all project details into every turn's system prompt.

**Effect**: The agent only sees LV1 (global constraints) + index (project/skill summaries) at the start of a new session. Full content loads only when needed. Tokens spent where they matter.

### LV1 Example

```
Memory architecture has three layers: LV1=memory/user/SOUL, LV2=project and user details (stored as skills, description as summary, load on demand), LV3=procedural skills (skills). Skills contain two types of knowledge: declarative ("what is it", e.g. project background, user info) and procedural ("how to do it", e.g. workflows, pitfalls). Scan skills_list before tasks to decide what to load, don't load blindly. See memory-hygiene skill for architecture details.
```

### LV2 Example (Project Folder Structure)

```
skills/{category}/
├── {project}-overview/SKILL.md     → description: "Project overview. Load for background, tech stack."
├── {project}-module-a/SKILL.md    → description: "Module A spec. Load when editing XXX."
└── {project}-module-b/SKILL.md    → description: "Module B rules. Load when writing YYY."
```

## Installation

Place `SKILL.md` into your Hermes skills directory:

```bash
# Clone the repo
git clone https://github.com/YOUR_USERNAME/memory-hygiene.git

# Copy to Hermes skills directory
cp memory-hygiene/SKILL.md ~/.hermes/skills/hermes-agent/memory-hygiene/SKILL.md
```

Or via Hermes CLI:

```bash
hermes skills install https://github.com/YOUR_USERNAME/memory-hygiene
```

## First Load

This skill operates on the layered architecture. **On first load**, the agent checks whether the architecture declaration exists in memory. If not, it writes it first before proceeding.

## What's Included

- **Skill Management Rules**: description writing, dedup, project folder structure
- **Cleanup Workflow**: per-item review → user confirmation → batch execution
- **7 Common Pitfalls**: text matching, sync gaps, bloat control, etc.
- **Self-Check Checklist**: post-operation verification

## Use Cases

- Memory usage > 80%, needs cleanup
- Project memories are messy, need modular split
- New project, need to plan skill structure
- Want more efficient agent memory, reduce token waste

## License

MIT
