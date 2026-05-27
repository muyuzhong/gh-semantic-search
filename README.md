# gh-semantic-search

**按架构思想搜索 GitHub 项目，而非关键词匹配 / Discover GitHub repos by concept, not keywords**

[![skills.sh](https://skills.sh/b/muyuzhong/gh-semantic-search)](https://skills.sh/muyuzhong/gh-semantic-search)

> 当你说"帮我找 multi-agent 项目"，普通搜索只会返回名字里有 "multi-agent" 的仓库。
> 这个 skill 会理解你的意图，自动扩展为 "agent swarm"、"agentic workflow"、"autogen"、"crewai"、"langgraph" 等多组关键词，
> 并行搜索、阅读 README、判断相关性，最终给出按语义相关度排序的结果。

---

## 它能做什么 / What It Does

| 场景 | 普通搜索 | gh-semantic-search |
|------|---------|-------------------|
| "找 multi-agent 项目" | 只返回名字含 "multi-agent" 的 repo | 找到 autogen、crewAI、langgraph、MetaGPT... |
| "event-driven 架构的参考实现" | 几乎无结果 | 找到 CQRS、Actor Model、消息队列等相关项目 |
| "类似 LangGraph 的替代品" | 不理解"类似" | 展开为图编排、agent workflow、状态机等概念搜索 |

### 核心能力

- **查询扩展** — 将一个概念展开为 4-8 个搜索词（同义词、已知框架、相关模式）
- **多路并行搜索** — 同时搜 name、description、README、topics
- **语义排序** — 读取 README 后判断真正相关性，而非纯按 star 排
- **Web 补充搜索** — 搜博客、对比文章、awesome-lists 发现更多项目

---

## 快速开始 / Quickstart

### 方式一：手动复制（推荐）

```bash
# 在你的项目根目录下执行
mkdir -p .claude/skills/gh-semantic-search
curl -o .claude/skills/gh-semantic-search/SKILL.md \
  https://raw.githubusercontent.com/muyuzhong/gh-semantic-search/main/.claude/skills/gh-semantic-search/SKILL.md
```

### 方式二：克隆仓库

```bash
git clone https://github.com/muyuzhong/gh-semantic-search.git
cp -r gh-semantic-search/.claude/skills/gh-semantic-search <your-project>/.claude/skills/
```

### 方式三：通过 Skills CLI

```bash
npx skills add muyuzhong/gh-semantic-search
```

安装完成后，直接用自然语言描述你想要的项目即可，skill 会自动触发。

---

## 使用示例 / Usage

Skill 会在你描述项目需求时自动触发。以下是一些示例：

**按架构思想搜索：**
> "帮我找高星的 multi-agent 框架项目"

**按设计模式搜索：**
> "Find repos that implement event-driven CQRS with event sourcing"

**找替代品：**
> "有没有类似 LangGraph 但更轻量的 agent 编排框架？"

**按领域搜索：**
> "Show me production-ready projects using the actor model pattern"

**组合条件：**
> "找一个用 Go 写的、高星的、支持分布式事务的框架"

---

## 工作流程 / How It Works

```
用户描述概念
    │
    ▼
┌─────────────────┐
│  1. 意图解析      │  提取核心概念、语言约束、质量门槛
└────────┬────────┘
         │
         ▼
┌─────────────────┐
│  2. 查询扩展      │  概念 → 同义词 + 已知框架 + 相关模式 (4-8 个搜索词)
└────────┬────────┘
         │
         ▼
┌─────────────────┐
│  3. 并行搜索      │  gh search repos × 多路并发 (name/desc/readme/topics)
└────────┬────────┘
         │
         ▼
┌─────────────────┐
│  4. Web 补充      │  anysearch 搜博客、对比文章、awesome-lists
└────────┬────────┘
         │
         ▼
┌─────────────────┐
│  5. 去重过滤      │  按 fullName 去重，排除 fork/archived，应用 star 门槛
└────────┬────────┘
         │
         ▼
┌─────────────────┐
│  6. 深度检查      │  GraphQL 获取 README + topics，Claude 判断相关性
└────────┬────────┘
         │
         ▼
┌─────────────────┐
│  7. 排序输出      │  语义相关性 > star 数 > 活跃度 > 文档质量
└─────────────────┘
```

---

## 输出示例 / Output Example

```
## GitHub Repositories for: Multi-Agent 框架/编排

### 1. microsoft/autogen -- A programming framework for agentic AI
- **Stars:** 58,448 | **Language:** Python | **Last pushed:** 2026-04-15
- **Why it matches:** 微软出品的多智能体对话框架，支持 agent 之间的自主协作
- **Install:** `pip install autogen`
- **URL:** https://github.com/microsoft/autogen

### 2. crewAIInc/crewAI -- Framework for orchestrating role-playing, autonomous AI agents
- **Stars:** 52,281 | **Language:** Python | **Last pushed:** 2026-05-27
- **Why it matches:** 基于角色扮演的多 agent 协作框架，每个 agent 有独立角色和目标
- **Install:** `pip install crewai`
- **URL:** https://github.com/crewAIInc/crewAI
...
```

---

## 环境要求 / Requirements

| 依赖 | 说明 |
|------|------|
| [Claude Code](https://docs.anthropic.com/en/docs/claude-code) | 运行环境 |
| [gh CLI](https://cli.github.com/) | GitHub 搜索接口（需已认证） |
| [anysearch](https://github.com/anthropics/skills) (可选) | 补充 Web 搜索能力 |

---

## 适用场景 / When to Use

**Use when:**
- 你想按架构概念找项目，而非按精确名称
- 你想找某个框架的替代品
- 你想发现某个领域的高质量开源项目
- 你想了解某个技术方向的生态全景

**Don't use when:**
- 你已知项目名，只是想看详情（直接 `gh repo view`）
- 你想搜索某个仓库内的代码（用 `gh search code`）
- 你想搜 issues 或 PRs（用 `gh search issues/prs`）

---

## License

MIT
