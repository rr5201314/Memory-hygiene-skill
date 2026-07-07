---
name: memory-hygiene
description: "记忆架构操作手册：清理记忆、迁移记忆到 skill、创建项目 skill、管理 skill 结构时加载。"
tags: [memory, hygiene, hermes-agent, maintenance, architecture]
---

# Memory Hygiene — 记忆架构操作手册

> [English](./SKILL.md) | 中文

## ⚠️ 首次加载前置检查

本 skill 基于一套分层记忆架构运作。**执行本 skill 的任何操作前**，先检查 memory（target='memory'）中是否已包含以下架构声明：

> 记忆架构分三层：LV1=memory/user/SOUL，LV2=项目和用户详情（skill的形式存放，description做记忆摘要，按需读取），LV3=流程技能（skill）。skill中的知识分两种：声明性（"是什么"，如项目背景、用户信息）和过程性（"怎么做"，如操作流程、pitfall）。任务前扫描 skills_list 决定加载哪些，不要盲目全加载。记忆架构和维护规范详见 memory-hygiene skill。

**如果 memory 中没有这段话**，在执行任何操作之前，先把这段话写入 memory。否则 agent 不知道分层架构的存在，本 skill 的规范无法生效。

---

> 架构定义（LV1/LV2/LV3 是什么、怎么映射到 Hermes）在上述 LV1 记忆中，每轮注入，不在这里重复。
> 本 skill 只写**怎么操作**和**有什么约束**。

---

## Skill 管理规范

### Description 写法规范

description 是 agent 决定是否加载的唯一依据，必须做到：

1. **语义可检索**：包含关键词，让 agent 能通过任务内容匹配到它
2. **说明触发条件**：写明"什么时候该加载这个 skill"
3. **简短**：一两句话，不要写成摘要文章

```
# 好的 description
description: "XXX 项目开发坑点。编辑 YYY 数据、ZZZ 模块时加载。"

# 差的 description
description: "这个 skill 记录了关于 XXX 项目的一些注意事项和开发经验"
```

### 去重规则

写入前必须 `skills_list` 查重。已有相关 skill → `patch` 追加，不新建。

### 项目记忆的文件夹结构

大型项目的 LV2 记忆不应挤在一个 skill 里，应拆成**一个 category 文件夹 + 多个子 skill**：

```
skills/
└── {category}/                    # 项目专属 category
    ├── {project}-overview/        # 项目总览
    │   └── SKILL.md              # 背景、技术栈、仓库地址、功能清单
    ├── {project}-module-a/        # 模块 A
    │   └── SKILL.md              # 模块介绍、约束、规则、坑点
    ├── {project}-module-b/        # 模块 B
    │   └── SKILL.md
    └── ...
```

**示例：一个典型的 App 项目**

```
skills/{category}/
├── {project}-overview/SKILL.md        → description: "XXX 项目总览。了解项目背景、技术栈时加载。"
├── {project}-module-a/SKILL.md       → description: "模块 A 开发规范。编辑 XXX 数据时加载。"
├── {project}-module-b/SKILL.md       → description: "模块 B 开发规则。写 YYY 功能时加载。"
└── {project}-integration/SKILL.md    → description: "XXX 集成方案。对接 YYY 服务时加载。"
```

**拆分原则**：

- 每个子 skill 对应一个**独立的功能模块或工作场景**
- 子 skill 的 description 写清楚**这个模块是什么 + 什么时候加载**
- 总览 skill 放项目级信息（背景、技术栈、仓库、功能清单），不堆具体细节
- 坑点跟模块走，不单独建 "pitfalls" 文件——在对应模块 skill 内标注

---

## 清理流程

当用户要求清理记忆，或记忆占用 > 80% 时：

### Step 1：逐条审查

对每条记忆问三个问题：

1. 这条是否每次会话都必须知道？→ **是** = LV1，留
2. 这条是否只在特定场景下有用？→ **是** = LV2/LV3，转 skill
3. 这条是否已过时/重复？→ **是** = 删

### Step 2：列清单给用户确认

```
**可删**：
- #X 描述（原因）

**建议转 skill**：
- #X 描述 → 建议转入 xxx skill

**重复可合并**：
- #X 和 #Y 内容重叠
```

**等用户确认后再操作**，不要自作主张。

### Step 3：批量执行

- 转 skill 的条目：先写入 skill，再用 memory(operations=[...]) 一次完成所有删除/替换（两步，不能漏）
- 目标：LV1 记忆 usage < 50%，把空间留给真正必要的内容

---

## 常见陷阱

1. **老条目文本匹配失败**：memory remove 的 old_text 必须是条目的**精确子串**。如果 remove 失败，用 memory(operations=[]) 的 current_entries 获取完整文本再重试。

2. **不要在清理时误加条目**：想"看看记忆"不代表要 add 什么。用 system prompt 中的 MEMORY 段直接读，不要调 memory(action='add')。

3. **skill 创建后别忘删 memory**：转存是两步操作（写 skill + 删 memory），漏删 memory 等于没清理。

4. **批量操作是原子的**：operations 数组中任何一条失败，整批不执行。确保每条 old_text 都精确。

5. **LV1 不要膨胀**：每次清理时审视 LV1 条目，问"这条真的每次都需要吗？"。LV1 越精简，每轮 token 花在刀刃上的比例越高。

6. **记忆和 skill 必须同步**：当用户修改一条偏好，这条信息可能同时存在于 memory 和某个 skill 的示例中。只改 memory 不改 skill = 下次加载 skill 时旧规则又回来了。改记忆时 grep 一下 skills 目录，确认没有残留引用。

7. **不要用用户强制要求的条目当反例**：在 skill 文档中写"不该存"的示例时，确认该示例不是用户明确要求保留的 LV1 条目。否则 skill 和 memory 自相矛盾，agent 无所适从。

---

## 自检清单

执行完本 skill 后，对照检查：

- [ ] LV1 条目是否都在 memory（target='memory' 或 'user'）中，没有遗漏
- [ ] LV1 每条都是"每次会话必须知道"的，没有混入按需内容
- [ ] memory 中没有项目细节、工具 pitfall、操作流程（这些应是 skill）
- [ ] memory 和 skill 内容一致，没有改了一边忘了另一边
- [ ] LV2/LV3 skill 的 description 都写清楚了触发条件
- [ ] 项目 skill 按文件夹结构组织，子模块独立、坑点跟模块走
- [ ] skills_list 中没有功能重叠的 skill

发现问题直接修，不用等用户要求。
