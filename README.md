# gh-semantic-search

A Claude Code skill that discovers GitHub repositories by **architectural concept, design pattern, or technical approach** -- not just keyword matching.

## What it does

Instead of searching "multi-agent" and getting only repos with that exact word in the name, this skill:

1. **Expands** your concept into diverse search terms (synonyms, known implementations, related patterns)
2. **Searches** GitHub in parallel across name, description, README, and topics
3. **Reads** top candidates' READMEs to assess real relevance
4. **Ranks** by semantic relevance, not just star count

## Install

Copy the skill to your project:

```bash
# In your project root
mkdir -p .claude/skills/gh-semantic-search
cp .claude/skills/gh-semantic-search/SKILL.md <your-project>/.claude/skills/gh-semantic-search/
```

Or clone and copy:

```bash
git clone https://github.com/<your-username>/gh-semantic-search.git
cp -r gh-semantic-search/.claude/skills/gh-semantic-search <your-project>/.claude/skills/
```

## Usage

Just describe what you're looking for in natural language:

- "帮我找高星的 multi-agent 框架项目"
- "Find repos that implement event-driven CQRS"
- "What are the best alternatives to LangGraph?"
- "Show me projects using the actor model pattern"

The skill will automatically trigger and run the semantic search workflow.

## How it works

| Step | What happens |
|------|-------------|
| Parse intent | Extract core concept, constraints, scope |
| Query expansion | Generate 4-8 search terms: direct, synonyms, known implementations, related patterns |
| Parallel search | Run multiple `gh search repos` with varied parameters |
| Web supplement | Use anysearch to find awesome-lists and comparison articles |
| Deep inspection | Fetch README + topics via GraphQL for top candidates |
| Rank & present | Semantic relevance > stars > activity > docs quality |

## Requirements

- Claude Code
- `gh` CLI (authenticated)
- (Optional) `anysearch` skill for supplementary web search

## License

MIT
