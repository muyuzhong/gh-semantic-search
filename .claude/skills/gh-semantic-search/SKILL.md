---
name: github-search
description: Semantic search for GitHub repositories by architectural concept, design pattern, or technical approach -- not just keyword matching. Triggers on "find repos for X", "projects that do Y", "alternatives to Z", "what's the best library for", "high-star projects for", or any request to discover GitHub projects by concept rather than exact name.
---

# GitHub Semantic Search

Discover high-quality GitHub repositories by architectural concept, design pattern, or technical approach -- not just keyword matching.

## When to Use This Skill

- User describes a concept and wants matching repositories (e.g., "multi-agent orchestration", "event-driven CQRS", "actor model framework")
- User asks for "alternatives to" or "projects like" a known repo
- User wants the "best" or "most popular" repo for a category
- User asks "what repos should I look at for X"
- User describes a software architecture and wants reference implementations

Do NOT use for: searching within a specific repo, searching code files, or looking up a repo the user already named.

## Workflow

### Step 1: Parse Intent

Identify before searching:
- **Core concept**: What architectural pattern, framework type, or capability?
- **Constraints**: Language preference? Minimum quality bar? Active maintenance?
- **Scope**: Framework, reference implementation, library, or awesome-list?

If ambiguous, ask one clarifying question. If clear, proceed immediately.

### Step 2: Expand into Diverse Search Terms

This is the core of "semantic" search. Do NOT just search the user's exact words.

Generate 4-8 queries covering:
1. **Direct terms**: User's exact words (e.g., "multi-agent")
2. **Synonyms**: Alternative phrasings (e.g., "agent swarm", "agent orchestration", "agentic workflow")
3. **Known implementations**: Named frameworks in the space (e.g., "autogen", "crewai", "langgraph")
4. **Related patterns**: Adjacent concepts (e.g., "task decomposition", "tool-use agent")
5. **Domain-specific terms**: Industry jargon (e.g., "actor framework" for "concurrent message passing")

Example for "multi-agent orchestration":
- "multi-agent" (direct)
- "agent orchestration framework" (synonym)
- "agentic workflow" (adjacent)
- "agent swarm" (alternative architecture)
- "autogen", "crewai", "langgraph" (known implementations)
- "tool-use agent framework" (domain term)

### Step 3: Parallel GitHub Searches

Run multiple `gh search repos` calls **in parallel**, varying parameters for coverage.

**Template:**
```bash
gh search repos "<QUERY>" --sort=stars --stars=">100" --limit=10 --json fullName,stargazersCount,description,updatedAt,pushedAt,url,language
```

**Search strategy (run in parallel):**

1. **Broad concept** (default: name+description+readme):
   ```bash
   gh search repos "<CONCEPT>" --sort=stars --stars=">100" --limit=10 --json fullName,stargazersCount,description,updatedAt,pushedAt,url,language
   ```

2. **Description-focused** (catches repos describing the pattern):
   ```bash
   gh search repos "<CONCEPT>" --match=description --sort=stars --stars=">100" --limit=10 --json fullName,stargazersCount,description,updatedAt,pushedAt,url,language
   ```

3. **README-focused** (catches repos discussing the pattern in docs, higher threshold):
   ```bash
   gh search repos "<CONCEPT>" --match=readme --sort=stars --stars=">500" --limit=5 --json fullName,stargazersCount,description,updatedAt,pushedAt,url,language
   ```

4. **Known framework names** (from Step 2):
   ```bash
   gh search repos "<FRAMEWORK>" --sort=stars --stars=">100" --limit=5 --json fullName,stargazersCount,description,updatedAt,pushedAt,url,language
   ```

5. **Topic-based** (if concept maps to a GitHub topic):
   ```bash
   gh search repos --topic=<TOPIC> --sort=stars --limit=10 --json fullName,stargazersCount,description,updatedAt,pushedAt,url,language
   ```

6. **GraphQL with topics** (richer data, topics not available via CLI search):
   ```bash
   gh api graphql -f query='{ search(query: "<CONCEPT>", type: REPOSITORY, first: 10) { nodes { ... on Repository { nameWithOwner stargazerCount description repositoryTopics(first: 10) { nodes { topic { name } } } updatedAt pushedAt } } } }' --jq '.data.search.nodes[] | {fullName: .nameWithOwner, stars: .stargazerCount, description, topics: [.repositoryTopics.nodes[].topic.name], updatedAt: .updatedAt, pushedAt: .pushedAt}'
   ```

**Adjust star threshold by niche size:**
- Well-known domain (agents, web frameworks): `--stars=">500"`
- Niche domain (CQRS, actor model): `--stars=">100"`
- Very niche (specific protocol): `--stars=">50"`

### Step 4: Supplementary Web Search

Use `anysearch` to find context GitHub search misses -- awesome-lists, comparison articles, newer projects gaining traction:

- Search: `"<CONCEPT> github repositories comparison 2025 2026"`
- Search: `"awesome <CONCEPT> github"`

### Step 5: Deduplicate and Pre-filter

1. **Deduplicate by fullName** (owner/repo), keep entry with most data
2. **Remove forks** unless clearly more popular than original
3. **Flag inactive** if `pushedAt` > 12 months ago (but don't auto-exclude)
4. **Apply minimum star threshold**: 100 for broad, 50 for niche

### Step 6: Deep Inspection of Top Candidates

For top 10-15 candidates, fetch richer data via GraphQL:

```bash
gh api graphql -f query='{ repository(owner: "<OWNER>", name: "<REPO>") { description stargazerCount forkCount updatedAt pushedAt repositoryTopics(first: 20) { nodes { topic { name } } } object(expression: "main:README.md") { ... on Blob { text } } } }' --jq '{description: .repository.description, stars: .repository.stargazerCount, forks: .repository.forkCount, updated: .repository.updatedAt, pushed: .repository.pushedAt, topics: [.repository.repositoryTopics.nodes[].topic.name], readme_preview: (.repository.object.text | .[0:1500])}'
```

**Relevance assessment** (use your judgment):
- Does the README describe the concept the user asked about?
- Do the topics align?
- Is the description a real match or a false positive?
- Is the project actively maintained?
- Is it mature enough (stars, forks, community)?

### Step 7: Rank and Present

Rank by composite score:
1. **Relevance to concept** (most important -- your semantic judgment)
2. **Star count** (proxy for quality and adoption)
3. **Recency** (actively maintained > stale)
4. **Documentation quality** (good README > no README)

Present top 5-10 results. Expand to 15 if user asks for more.

### Step 8: Output Format

```
## GitHub Repositories for: <user's concept>

### 1. owner/repo-name -- <one-line description>
- **Stars:** N | **Language:** X | **Last pushed:** YYYY-MM-DD
- **Topics:** topic1, topic2, topic3
- **Why it matches:** <1-2 sentences on relevance>
- **Install:** `<pip install X / npm install Y / go get Z>`
- **URL:** https://github.com/owner/repo-name

### 2. ...
```

Add **"Also worth exploring"** with 3-5 additional repos (lower stars, newer, tangentially related).

Add **"Further reading"** if anysearch found relevant awesome-lists or comparison articles.

## Edge Cases

**No results:** Broaden search terms, lower star threshold, use anysearch to check if concept exists under a different name.

**Too many results (>50):** Raise star threshold to >500, focus on description matches, limit by language, present top 10.

**Ambiguous query:** List possible interpretations, ask user to clarify.

**Too broad:** Ask for constraints (language, scale, use case).

**Too niche:** Search components separately, be honest about narrowness.

## Rate Limits

- `gh search repos`: 30 requests/minute
- `gh api graphql`: 5000 points/hour
- Keep total API calls under 20 per session
