# Memory Hygiene — Hermes Agent 分层记忆架构 Skill

> 解决 Hermes Agent 记忆系统的结构性问题：所有记忆平铺注入、没有按需加载、项目细节污染每轮上下文。

## 痛点

Hermes 的记忆系统只有两层：`memory`（agent 笔记）和 `user`（用户信息），所有内容**每轮对话强制注入** system prompt。这带来三个问题：

**1. 噪声污染**

项目细节、工具 pitfall、环境配置这类只在特定场景才需要的信息，每轮都占 token。比如你有一个项目的数据编辑规范，但它在你写论文、查资料、调代码时全都注入了——白花 token，还干扰模型注意力。

**2. 没有"摘要 + 按需全文"机制**

理想情况下，agent 应该只看到"你有这些项目和技能"的简短索引，需要时再加载详情。但 Hermes 的 memory 没有这个能力——要么全给，要么不给。

**3. 记忆膨胀不可控**

任务进度、完成记录、临时发现不断堆积，4000 字符很快就满了。没有分层概念，用户不知道哪些该留、哪些该转、哪些该删。

## 方案

利用 Hermes 现有机制做平替，把记忆分为三层：

| 层级 | 职责 | 注入时机 | Hermes 实现 |
|------|------|---------|------------|
| **LV1** | 用户基本信息、人格、全局约束 | **每轮** system prompt | `memory`(target='memory' + 'user') + `SOUL.md` |
| **LV2** | 项目上下文、用户详细信息、工具环境 | **按需** | Skill（description 做摘要） |
| **LV3** | 操作流程、pitfall、工作流 | **按需** | Skill（同上） |

**核心洞察**：`skills_list` 返回所有 skill 的 name + description，这天然就是 LV2/LV3 的"摘要索引"。agent 扫描 `skills_list` 后按需 `skill_view`，等价于"摘要注入 + 按需全文"。

### 伪渐进式披露架构

Hermes 没有原生的"摘要注入 + 按需全文"机制，但 `skills_list` + `skill_view` 组合出了一套**伪渐进式披露**（pseudo-progressive disclosure）：

```
┌─────────────────────────────────────────────────────┐
│                    每轮 system prompt                   │
│                                                         │
│  LV1: memory + user + SOUL.md  ←  强制注入，始终在场     │
│                                                         │
│  skills_list 返回的索引（自动注入，零额外开销）：          │
│  ┌───────────────────────────────────────────────────┐ │
│  │ name: myapp-overview                              │ │
│  │ desc: XXX 项目总览。了解项目背景、技术栈时加载。       │ │
│  ├───────────────────────────────────────────────────┤ │
│  │ name: myapp-data-model                            │ │
│  │ desc: 数据模型规范。编辑数据库 schema 时加载。        │ │
│  ├───────────────────────────────────────────────────┤ │
│  │ name: tool-setup-pitfalls                         │ │
│  │ desc: 某工具安装坑点。配置该工具时加载。              │ │
│  └───────────────────────────────────────────────────┘ │
│                                                         │
│  ↑ agent 扫描索引，根据当前任务语义判断是否需要加载 ↓     │
└─────────────────────────────────────────────────────┘

          ↓ skill_view(name="myapp-data-model") ↓

┌─────────────────────────────────────────────────────┐
│              SKILL.md 全文（按需加载）                  │
│                                                         │
│  数据模型规范                                           │
│  ├── 用户表和订单表不能直接 JOIN                         │
│  ├── 迁移脚本必须可回滚                                 │
│  ├── 新增字段必须有默认值                                │
│  └── ...                                                │
└─────────────────────────────────────────────────────┘
```

**为什么叫"伪"渐进式披露**：

真正的渐进式披露是 UI 层面的——用户点击才展开详情。Hermes 没有这个 UI，但效果等价：

| 渐进式披露 | Hermes 对应 |
|-----------|------------|
| 第一层：摘要/索引 | `skills_list` 返回 name + description |
| 第二层：详情 | `skill_view` 加载 SKILL.md 全文 |
| 用户决定是否展开 | agent 根据任务语义决定是否 `skill_view` |

**代价**：agent 需要一次 `skills_list` 调用（返回所有 skill 的 name + description，约几百 token）。相比把所有项目细节塞进每轮 system prompt，这个代价可以忽略。

**效果**：agent 在新会话中只看到 LV1（全局约束）+ 索引（项目/技能摘要），真正需要时才加载全文。token 用在刀刃上。

### LV1 示例

```
记忆架构分三层：LV1=memory/user/SOUL，LV2=项目和用户详情（skill的形式存放，
description做记忆摘要，按需读取），LV3=流程技能（skill）。skill中的知识分两种：
声明性（"是什么"，如项目背景、用户信息）和过程性（"怎么做"，如操作流程、pitfall）。
任务前扫描 skills_list 决定加载哪些，不要盲目全加载。记忆架构和维护规范详见 memory-hygiene skill。
```

### LV2 示例（项目文件夹结构）

```
skills/{category}/
├── {project}-overview/SKILL.md     → description: "XXX 项目总览。了解背景、技术栈时加载。"
├── {project}-module-a/SKILL.md    → description: "模块 A 规范。编辑 XXX 时加载。"
└── {project}-module-b/SKILL.md    → description: "模块 B 规则。写 YYY 时加载。"
```

## 安装

将 `SKILL.md` 放入你的 Hermes skills 目录：

```bash
# 克隆仓库
git clone https://github.com/YOUR_USERNAME/memory-hygiene.git

# 复制到 Hermes skills 目录
cp memory-hygiene/SKILL.md ~/.hermes/skills/hermes-agent/memory-hygiene/SKILL.md
```

或通过 Hermes CLI：

```bash
hermes skills install https://github.com/YOUR_USERNAME/memory-hygiene
```

## 首次加载

本 skill 基于分层架构运作。**首次加载时**，agent 会检查 memory 中是否已包含架构声明。如果没有，会先写入，再执行后续操作。

## 包含内容

- **Skill 管理规范**：description 写法、去重规则、项目文件夹结构
- **清理流程**：逐条审查 → 用户确认 → 批量执行
- **7 条常见陷阱**：文本匹配、同步遗漏、膨胀控制等
- **自检清单**：执行后对照检查

## 适用场景

- 记忆占用 > 80%，需要清理
- 项目记忆混乱，需要按模块拆分
- 新开项目，需要规划 skill 结构
- 想让 agent 记忆更高效、减少 token 浪费

## License

MIT
