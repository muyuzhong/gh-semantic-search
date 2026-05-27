# gh-semantic-search

**按架构思想搜索 GitHub 项目，而非关键词匹配 / Discover GitHub repos by concept, not keywords**

[![skills.sh](https://skills.sh/b/muyuzhong/gh-semantic-search)](https://skills.sh/muyuzhong/gh-semantic-search)
[![License: MIT](https://img.shields.io/badge/License-MIT-yellow.svg)](LICENSE)

---

当你想找"multi-agent 项目"，普通搜索只返回名字里带 "multi-agent" 的仓库。

这个 skill 会理解你的意图，自动展开为 "agent swarm"、"agentic workflow"、"autogen"、"crewai"、"langgraph" 等多组关键词，**并行搜索 → 读 README → 语义排序**，找到真正相关的项目。

## 安装 / Install

在你的 agent 里说：

```
帮我安装 muyuzhong/gh-semantic-search 这个 skill
```

## 使用 / Usage

安装后直接用自然语言描述需求，skill 会自动触发：

> "帮我找高星的 multi-agent 框架项目"
>
> "Find repos that implement event-driven CQRS"
>
> "有没有类似 LangGraph 但更轻量的 agent 编排框架？"
>
> "找一个用 Go 写的、支持分布式事务的框架"

## 适用场景 / When to Use

| 用 | 不用 |
|---|---|
| 按架构概念找项目 | 已知名字，想看详情 |
| 找某个框架的替代品 | 搜某个仓库内的代码 |
| 发现某领域的高质量开源项目 | 搜 issues 或 PRs |

## 环境要求 / Requirements

- [Claude Code](https://docs.anthropic.com/en/docs/claude-code) 或其他支持 Skills 的 agent
- [gh CLI](https://cli.github.com/)（需已认证）
- (可选) [anysearch](https://github.com/anthropics/skills) skill 用于补充 Web 搜索

## License

MIT
