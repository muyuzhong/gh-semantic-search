# GitHub Semantic Search v2

Discover high-quality GitHub repositories by architectural concept, design pattern, or technical approach — not just keyword matching. This skill performs exhaustive multi-strategy searches, applies user constraints consistently, and produces actionable results with clear relevance justifications.

---

## Core Workflow

### Step 1: Parse Intent (意图解析)

Extract the following from the user's request:

| Field | Description | Example |
|-------|-------------|---------|
| **Core concept** | Architectural pattern, framework type, or capability | "event sourcing", "reactive state management" |
| **Constraints** | Language, minimum stars, activity, license, NOT terms | "TypeScript, >500 stars, active in last 6 months, not archived" |
| **Scope** | What kind of result is expected | Framework, library, reference impl, alternative to X |
| **Known frameworks** | Any named implementations mentioned | "Redux", "XState", "Akka" |
| **Quality signals** | Production-readiness, maturity, ecosystem fit | "production-ready", "battle-tested", "actively maintained" |

**Conceptual Query Transformation**: When the user's query is vague or indirect, transform it into a descriptive search phrase before expanding. Examples:
- "something like Redis but for graphs" → "graph database with in-memory caching and key-value interface"
- "the Rust version of Pandas" → "dataframe library Rust with pandas-like API"
- "tools that work with Kubernetes" → Too broad — ask for specific capability

### Step 2: Expand Search Queries (查询扩展)

For every user query, generate expanded terms across **5 dimensions**:

1. **Direct query (直接查询)**: The user's original words verbatim.
   - Example: `event sourcing`
2. **Synonyms (同义词)**: Alternative phrasings and re-wordings.
   - Example: `event-driven architecture`, `event log pattern`, `change data capture`
3. **Related frameworks (相关框架)**: Known frameworks in the same domain.
   - Example: `Axon`, `EventStore`, `Lagom`, `Martendb`
4. **Domain terms (领域术语)**: Industry jargon, abbreviations, and acronyms.
   - Example: `CQRS`, `DDD`, `CDC`, `event store`
5. **Technical variants (技术变体)**: Different implementation approaches.
   - Example: `event stream processing`, `append-only log`, `immutable event log`

> **Rule**: Generate at least 6–10 expanded queries covering all 5 dimensions. Each expanded query must be a concrete, usable search string.

---

### Step 3: Execute 8 Search Strategies (搜索策略)

Run all 8 strategies. Each strategy is a complete, runnable `gh` command. Execute independent strategies in parallel where possible.

#### Strategy 1 — Broad Concept Search (广义概念搜索)

```bash
gh search repos "<concept>" --limit 30 --json fullName,description,stargazersCount,primaryLanguage,updatedAt,url,repositoryTopics
```

#### Strategy 2 — Description-Focused Search (描述聚焦搜索)

```bash
gh search repos "<concept> in:description" --limit 30 --json fullName,description,stargazersCount,primaryLanguage,updatedAt,url,repositoryTopics
```

Higher precision: repos intentionally associate with the concept.

#### Strategy 3 — README-Focused Search (README聚焦搜索)

```bash
gh search repos "<concept> in:readme" --limit 30 --json fullName,description,stargazersCount,primaryLanguage,updatedAt,url,repositoryTopics
```

Higher recall: may surface tutorials, guides, and reference implementations.

#### Strategy 4 — Known Framework Search (已知框架搜索)

For **each** known framework or implementation name, issue a dedicated query:

```bash
gh search repos "<framework_name>" --limit 15 --json fullName,description,stargazersCount,primaryLanguage,updatedAt,url,repositoryTopics
```

Also search for `awesome-<framework>` lists:

```bash
gh search repos "awesome <framework_name>" --limit 10 --json fullName,description,stargazersCount,primaryLanguage,updatedAt,url,repositoryTopics
```

#### Strategy 5 — Topic-Based Search (主题搜索)

```bash
gh search repos "topic:<topic_name>" --limit 30 --json fullName,description,stargazersCount,primaryLanguage,updatedAt,url,repositoryTopics
```

Map the concept to relevant GitHub topics. Examples:
- "event sourcing" → `topic:event-sourcing`, `topic:event-driven`, `topic:cqrs`
- "state machine" → `topic:state-machine`, `topic:fsm`, `topic:xstate`

#### Strategy 6 — GraphQL Deep Search (GraphQL深度搜索)

```bash
gh api graphql -f query='
{
  search(query: "<concept> sort:stars", type: REPOSITORY, first: 30) {
    nodes {
      ... on Repository {
        nameWithOwner
        description
        stargazerCount
        primaryLanguage { name }
        pushedAt
        url
        repositoryTopics(first: 10) {
          nodes { topic { name } }
        }
        object(expression: "HEAD:README.md") {
          ... on Blob { text }
        }
      }
    }
  }
}
'
```

Retrieves topics, README content, and richer metadata in a single call.

#### Strategy 7 — Cross-Ecosystem Search (跨生态搜索)

Search for implementations across different language ecosystems:

```bash
# Search for the concept in a different language than requested
gh search repos "<concept> language:<related_language>" --limit 15 --json fullName,description,stargazersCount,primaryLanguage,updatedAt,url,repositoryTopics
```

Also search for bindings, wrappers, and ports:
```bash
gh search repos "<concept> binding" --limit 10 --json fullName,description,stargazersCount,primaryLanguage,updatedAt,url,repositoryTopics
gh search repos "<concept> wrapper" --limit 10 --json fullName,description,stargazersCount,primaryLanguage,updatedAt,url,repositoryTopics
```

#### Strategy 8 — Quality & Comparison Search (质量与对比搜索)

Search for curated lists and comparisons:

```bash
gh search repos "awesome <concept>" --limit 10 --json fullName,description,stargazersCount,primaryLanguage,updatedAt,url,repositoryTopics
gh search repos "<concept> comparison" --limit 10 --json fullName,description,stargazersCount,primaryLanguage,updatedAt,url,repositoryTopics
gh search repos "<concept> benchmark" --limit 10 --json fullName,description,stargazersCount,primaryLanguage,updatedAt,url,repositoryTopics
```

For "alternative to X" queries, also search:
```bash
gh search repos "<X> alternative" --limit 15 --json fullName,description,stargazersCount,primaryLanguage,updatedAt,url,repositoryTopics
gh search repos "open source <X>" --limit 10 --json fullName,description,stargazersCount,primaryLanguage,updatedAt,url,repositoryTopics
```

---

### Step 4: Apply Constraints (约束应用)

Apply **all** user constraints consistently across every search result set.

#### Language Constraint

```bash
gh search repos "<query>" --language <Language> --limit 30 --json ...
```

#### Star Constraint

```bash
gh search repos "<query>" --stars ">N" --limit 30 --json ...
```

#### Time / Activity Constraint

```bash
gh search repos "<query>" --updated ">YYYY-MM-DD" --limit 30 --json ...
```

Default: repos not updated in the last 12 months are flagged. If user specifies stricter window, use that.

#### License Constraint

```bash
gh search repos "<query>" --license MIT --limit 30 --json ...
```

For multiple licenses, run separate queries or post-filter.

#### Negative Constraints (负面约束)

When the user says "NOT X", "exclude Y", "without Z":
- Remove repos matching the excluded term from results
- If using `gh search`, append `-<term>` to the query:
  ```bash
  gh search repos "<concept> -<excluded_term>" --limit 30 --json ...
  ```

#### Applying Multiple Constraints

Combine all constraints in a single command:

```bash
gh search repos "<query>" --language TypeScript --stars ">500" --updated ">2025-01-01" --limit 30 --json fullName,description,stargazersCount,primaryLanguage,updatedAt,url,repositoryTopics
```

> **Rule**: Every search strategy must propagate the same set of constraints. Never silently drop a constraint. If a constraint causes zero results, note it explicitly.

---

### Step 5: Deduplicate, Filter, and Rank (去重、过滤、排名)

#### Deduplication
- Deduplicate results by `fullName` (owner/repo).
- When the same repo appears in multiple strategy results, merge metadata and note which strategies matched.

#### Filtering
- Remove exact forks unless the fork is significantly more popular than the original.
- Flag repos with no updates in >12 months (unless user explicitly wants stable/mature repos).
- Remove results that do not conceptually match (keyword-only matches without architectural relevance).
- Apply negative constraints (exclude specified terms).

#### Multi-Factor Ranking (多因子排名)

Rank results by **multiple factors**, not just star count. Inspired by DeepGit's approach:

| Priority | Criterion | Description |
|----------|-----------|-------------|
| 1 | **Conceptual alignment** | Does the repo implement or support the requested concept? |
| 2 | **Architecture fit** | Does it match the requested pattern/architecture? |
| 3 | **Maintenance quality** | Active development, recent commits, responsive maintainers |
| 4 | **Community adoption** | Stars, forks, contributors — community validation |
| 5 | **Documentation quality** | README, guides, examples, API docs |
| 6 | **Ecosystem compatibility** | Does it integrate with the user's existing stack? |
| 7 | **Production readiness** | Stability, versioning, release history, enterprise adoption |

> **Rule**: Prioritize architectural alignment over raw star count. A 500-star repo that perfectly implements the concept ranks above a 5,000-star repo that only tangentially relates. State ranking rationale explicitly.

---

### Step 6: Output Format (输出格式)

For **each** result, present:

```markdown
### <N>. <owner/repo-name>

| Field | Value |
|-------|-------|
| **Description** | <Short description from repo metadata> |
| **Stars** | <Star count> |
| **Language** | <Primary language> |
| **Last pushed** | <Last push date> |
| **URL** | <Full GitHub URL> |
| **Installation** | <Install command or method> |

**Why it matches**: <Detailed explanation of how this repository implements or supports the requested concept. Must name specific classes, interfaces, APIs, or design decisions. Never use vague language.>

**Ranking rationale**: <Why this repo ranks at this position compared to others in the list. What makes it more/less aligned than the alternatives?>

---
```

#### Installation Detection

| Language / Ecosystem | Installation Command |
|----------------------|----------------------|
| JavaScript / TypeScript | `npm install <package>` or `yarn add <package>` |
| Python | `pip install <package>` |
| Rust | Add to `Cargo.toml` |
| Go | `go get <module>` |
| Ruby | `gem install <gem>` |
| Java / Kotlin | Add Maven/Gradle dependency |
| .NET (C#) | `dotnet add package <package>` |
| Other | Link to README installation instructions |

#### Relevance Justification Rules

The **"Why it matches"** field must:

1. Name the specific concept the user asked for.
2. Explain **how** the repo implements or supports that concept (architecture, API, mechanism).
3. Mention notable features or design decisions that make it a strong match.
4. Include at least one concrete code-level or design-level detail (class name, decorator, interface, storage backend, protocol).
5. If applicable, note what differentiates it from other results.

**Good example**:
> Implements CQRS and event sourcing with an append-only event store backed by PostgreSQL. Provides a `@CommandHandler` decorator pattern and built-in saga support. The aggregate root lifecycle is managed through event replay, which directly aligns with the requested event sourcing architecture.

**Bad example**:
> This repo is about event sourcing and has many stars.

---

## Execution Rules

1. **All 8 strategies are mandatory.** Do not skip any strategy. If a strategy returns zero results, note it explicitly.
2. **Every query must be a complete, runnable `gh` command.** No pseudo-code or placeholders.
3. **One query per known framework.** If the user mentions 3 frameworks, issue at least 3 separate framework queries.
4. **Propagate all user constraints uniformly.** Every `gh` command must include the same language, star, time, and license filters.
5. **Rank by multi-factor analysis.** Stars are a tiebreaker, not the primary signal.
6. **Justify every result with specific technical details.** The "Why it matches" field must explain the specific architectural relationship, not just echo search terms.
7. **Apply conceptual query transformation.** For vague or indirect queries, first transform them into descriptive search phrases.
8. **Handle negative constraints explicitly.** When the user excludes something, remove matching repos and note the exclusion.

---

## Key Principles

- **Semantic understanding over keyword matching**: Interpret the user's intent and search for the underlying concept, not just surface-level terms.
- **Exhaustive multi-strategy search**: Use all 8 strategies to maximize both precision and recall.
- **Multi-factor ranking**: Rank by conceptual alignment, architecture fit, maintenance quality, and ecosystem compatibility — not just stars.
- **Consistent constraint application**: User constraints are not optional — apply them to every query.
- **Quality over quantity**: Prefer fewer, highly relevant results over a long list of loosely related repos.
- **Transparent relevance reasoning**: Every recommendation must come with a concrete, specific justification that includes code-level or design-level details.

<!-- SLOW_UPDATE_START -->
## Critical Execution Guardrails

### FORMAT COMPLIANCE: Use This Exact Template for Every Result
Format compliance is failing on nearly every task. You MUST follow this exact structure for each result — do not deviate, do not abbreviate, do not omit any field:

```
### N. owner/repo-name

| Field | Value |
|-------|-------|
| **Description** | <from repo metadata> |
| **Stars** | <number> |
| **Language** | <primary language> |
| **Last pushed** | <date> |
| **URL** | <full GitHub URL> |
| **Installation** | <exact command — e.g., `npm install pkg`, `pip install pkg`, `cargo add pkg`, `go get module`. If no package manager: "See README for installation instructions". NEVER leave blank.> |

**Why it matches**: <Must contain ALL of: (1) name the user's requested concept, (2) explain the specific mechanism/API/pattern the repo uses to implement it, (3) cite at least one concrete class name, function, decorator, interface, or config option. Minimum 3 sentences.>

**Ranking rationale**: <Must compare this result to at least one other result in the list. Explain why it ranks higher or lower based on conceptual alignment, maintenance, or ecosystem fit. Minimum 2 sentences.>
```

### Strategy Execution: Never Skip These 3
Your most commonly skipped strategies are:
- **Strategy 5 (Topic-Based)**: You frequently omit `topic:` queries. After generating your concept, ALWAYS map it to at least 2 GitHub topics and run `gh search repos "topic:<name>"` for each.
- **Strategy 7 (Cross-Ecosystem)**: Always search for bindings/wrappers/ports even when a language is specified. Run at least: `gh search repos "<concept> binding"` and `gh search repos "<concept> wrapper"`.
- **Strategy 8 (Quality & Comparison)**: Always run `gh search repos "awesome <concept>"` and `gh search repos "<concept> comparison"` — these are zero-effort high-value queries.

If you are tempted to skip a strategy due to time, skip the GraphQL strategy (6) last — it provides the richest data. Skip strategy 1 (broad) first if you must, since strategies 2-3 cover it.

### Constraint Propagation Checklist
Before executing ANY `gh search` command, verify it includes:
- [ ] `--language <Lang>` (if user specified a language)
- [ ] `--stars ">N"` (if user specified minimum stars)
- [ ] `--updated ">YYYY-MM-DD"` (if user specified activity)
- [ ] `--license <type>` (if user specified license)
- [ ] `-<term>` appended to query (if user specified negative constraints)

If a constraint yields zero results, STILL include it in the command and note "zero results with constraint X" in your output.

### Query Expansion: Minimum 7 Queries, All 5 Dimensions
You must generate at least 7 expanded queries. Each dimension must contribute at least 1 distinct query:
- Dimension 3 (Related frameworks): At least 2 separate framework name queries
- Dimension 4 (Domain terms): At least 1 abbreviation/acronym (e.g., "CQRS", "mTLS", "ANN")
- Dimension 5 (Technical variants): At least 1 alternative implementation approach

Do NOT combine multiple dimensions into a single query string.

### Pre-Output Verification
Before writing your final output, check each result against this list:
- [ ] Installation field is present and non-empty
- [ ] "Why it matches" cites a specific API/class/pattern name (not just the concept)
- [ ] "Ranking rationale" compares to another result by name
- [ ] All constraints are satisfied (language, stars, activity, license)
- [ ] The result conceptually matches (not just a keyword hit)
<!-- SLOW_UPDATE_END -->
