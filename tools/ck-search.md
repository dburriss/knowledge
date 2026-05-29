---
description: "A semantic code search tool that enables searching codebases by meaning rather than just keywords."
tags: [search, dev-tools, ai]
---

# CK Search

A semantic code search tool that enables searching codebases by meaning rather than just keywords. Built with Rust for performance, ck combines traditional regex search with semantic understanding powered by local embedding models.

## Why CK Search Exists

Traditional code search (grep, ripgrep, ack) relies on pattern matching - you must know the exact keywords or regex patterns to find what you're looking for. This breaks down when:

- You understand the **concept** you're searching for but don't know the exact terminology used in the codebase
- Different developers use different naming conventions for the same concept
- You want to find **semantically similar code** across a large codebase
- You need to understand unfamiliar code without knowing where to start

CK Search bridges this gap by understanding **what code does**, not just what it's called.

## Key Concepts

### Three Search Modes

**1. Semantic Search (`--sem`)**
Finds code by meaning using embedding models. Ideal for conceptual queries:

```bash
ck --sem "error handling" src/
ck --sem "database connection pooling" lib/
ck --sem "authentication logic" .
```

**2. Lexical Search (`--lex`)**
Traditional full-text search for exact terms and identifiers:

```bash
ck --lex "getUserById" src/
ck --lex "TODO" .
ck --lex "AuthController" .
```

**3. Hybrid Search (`--hybrid`)**
Combines semantic understanding with keyword matching using Reciprocal Rank Fusion (RRF):

```bash
ck --hybrid "JWT token validation" src/
ck --hybrid --threshold 0.025 "retry logic" .
```

### Embedding Models

CK Search uses local embedding models (no API calls required):

| Model | Chunk Size | Context Window | Best For |
|-------|------------|----------------|----------|
| `bge-small` (default) | 400 tokens | 512 tokens | Fast indexing, general code |
| `nomic-v1.5` | 1024 tokens | 8K tokens | Large functions, documentation |
| `jina-code` | 1024 tokens | 8K tokens | Code-specialized understanding |

Switch models based on your needs:

```bash
ck --index --model jina-code .
ck --switch-model nomic-v1.5 .
```

### Relevance Tuning

Control result quality with thresholds (0.0-1.0 scale):

- `0.3` - Exploratory (cast wide net)
- `0.5` - Balanced (default)
- `0.7` - High-confidence (recommended for [[AI Agents]])
- `0.9` - Very strict (only closest matches)

```bash
ck --sem --threshold 0.7 "authentication" src/
```

## Multiple Interfaces

CK Search adapts to different workflows:

### CLI - Command Line Interface

For scripts, automation, CI/CD:

```bash
ck --sem --limit 20 "pattern" src/
ck --json --sem "auth" . | jq -r '.[].file'
```

### TUI - Terminal User Interface

Interactive exploration with live search:

```bash
ck-tui
```

Features:

- Live results as you type
- Visual navigation (↑/↓ browse, → preview)
- Mode switching (semantic/regex/hybrid)
- Color-coded relevance scores
- Quick file opening (Enter)

### Editor Extension

VSCode/Cursor integration for in-editor search:

- `Cmd+Shift+;` - Open search
- `Cmd+Shift+'` - Search selected code
- Live results with relevance scores

### MCP Server

[[Model Context Protocol]] integration for AI agents:

```bash
ck --serve
```

Enables AI assistants like Claude Desktop to search codebases semantically through the MCP protocol.

## Integration with Second-Brain and OpenCode

### Using CK Search in This Repository

This second-brain contains 1000+ markdown files covering software engineering, AI/ML, cloud technologies, and leadership. CK Search can transform how you explore and connect concepts.

#### Setup for Second-Brain

```bash
# Index the repository once
cd /Users/devon.burriss/Documents/second-brain
ck --index .

# Check index status
ck --status .
```

#### Example Searches

**Find related concepts:**

```bash
# Semantic search for AI topics
ck --sem "retrieval augmented generation" .
ck --sem "context window management" .
ck --sem "prompt engineering techniques" .

# Find leadership concepts
ck --sem "team topologies" .
ck --sem "staff engineer responsibilities" .

# Cloud architecture patterns
ck --sem "serverless architecture" .
ck --sem "infrastructure as code" .
```

**Find specific notes:**

```bash
# Exact titles or tags
ck --lex "OpenCode" .
ck --lex "type: technology" .

# Combined approach
ck --hybrid "AWS CDK" .
```

### OpenCode Skill Integration

You could create an [[OpenCode Skills|OpenCode skill]] that uses CK Search for enhanced knowledge discovery:

**`.opencode/skill/semantic-search/SKILL.md`**:

```yaml
---
name: semantic-search
description: Search second-brain notes by concept using CK semantic search
license: MIT
compatibility: opencode
metadata:
  category: knowledge-management
  audience: second-brain-users
---

# Semantic Search Skill

Use CK Search to find related notes by meaning, not just keywords.

## When to Use
- User asks "what do I know about X?"
- User wants to explore connections between concepts
- User asks conceptual questions that span multiple notes
- User wants to find notes they've forgotten about

## Instructions

1. First check if CK index exists:
   ```bash
   ck --status /Users/devon.burriss/Documents/second-brain
   ```

1. If index is stale or missing, reindex:

   ```bash
   ck --index /Users/devon.burriss/Documents/second-brain
   ```

2. Perform semantic search with appropriate threshold:

   ```bash
   # For broad exploration
   ck --sem --threshold 0.5 --limit 20 "user query" /Users/devon.burriss/Documents/second-brain
   
   # For precise matches
   ck --sem --threshold 0.7 --limit 10 "user query" /Users/devon.burriss/Documents/second-brain
   ```

3. Present results with:
   - File names (strip path prefix)
   - Relevance scores
   - Brief context snippets
   - Suggest wiki-links to connect concepts

## Example Usage

User: "What do I know about grounding in AI?"

Agent:

- Runs: `ck --sem --threshold 0.7 --limit 10 "grounding in AI" .`
- Finds: Grounding.md, Context.md, RAG.md, Retrieval.md
- Suggests connections between [[Grounding]], [[RAG]], and [[Retrieval]]

### OpenCode Slash Command

Create a command for quick semantic searches:

**`.opencode/command/search.md`**:

```yaml
---
description: Semantic search across second-brain notes
agent: general
---

You are helping search a personal knowledge management system (second-brain) with 1000+ markdown notes.

The user wants to find notes related to: $ARGUMENTS

Use ck semantic search:
1. Run: ck --sem --threshold 0.7 --limit 15 "$ARGUMENTS" /Users/devon.burriss/Documents/second-brain
2. Present results with file names and relevance scores
3. Identify the top 3-5 most relevant notes
4. Suggest potential wiki-link connections between concepts
5. If results are sparse, try lowering threshold to 0.5

Keep output concise - just filenames, scores, and brief context.
```

Usage: `/search prompt engineering`

### MCP Server Configuration

For Claude Desktop integration, configure ck as an MCP server in `~/Library/Application Support/Claude/claude_desktop_config.json`:

```json
{
  "mcpServers": {
    "ck-second-brain": {
      "command": "ck",
      "args": ["--serve"],
      "cwd": "/Users/devon.burriss/Documents/second-brain",
      "env": {}
    }
  }
}
```

This enables Claude Desktop to semantically search your second-brain through natural conversation.

## Best Practices for Knowledge Management

### Index Strategy

- Index once per session or after significant additions
- ck handles incremental updates automatically
- Manual rebuild only needed after model changes

```bash
# Check if reindex needed
ck --status .

# Force rebuild if switching models
ck --switch-model jina-code --force .
```

### Search Patterns

**Start broad, then narrow:**

```bash
# Initial exploration
ck --sem --threshold 0.5 "kubernetes" .

# Refine to specific concept
ck --sem --threshold 0.7 "kubernetes operators" .
```

**Use appropriate modes:**

- Semantic: For concepts ("team dynamics", "error handling patterns")
- Lexical: For exact terms ("DDD", "CQRS", "Staff+")
- Hybrid: For technical terms with context ("AWS Lambda cold start")

### Output Formats for Automation

**JSONL for AI agents** (one JSON object per line):

```bash
ck --jsonl --sem --threshold 0.7 "pattern" . | while read -r line; do
  echo "$line" | jq '.file'
done
```

**JSON for scripting:**

```bash
# Get unique files above threshold
ck --json --sem --threshold 0.8 --scores "aws" . | jq -r '.[].file' | sort -u
```

### Exclusions for Second-Brain

Create `.ckignore` to optimize indexing:

```
# Exclude attachments (images don't need semantic search)
_attachments/

# Exclude templates (you know where they are)
_templates/

# Optionally exclude work-specific notes
# work/
```

## Advantages Over Traditional Search

### Conceptual Understanding

**Traditional (grep/ripgrep):**

- Must know exact keywords
- Case-sensitive matching issues
- Requires regex knowledge for complex patterns

**CK Search:**

- Search by meaning: "what is event sourcing?"
- Finds semantically similar content across different terminology
- No need to know exact phrasing used in notes

### Discovery

- Find notes you've forgotten about
- Discover unexpected connections between concepts
- Explore knowledge graph through semantic similarity

### AI-First Design

- JSONL output optimized for streaming
- Near-miss threshold hints for intelligent tuning
- Structured output (JSON/JSONL) for easy parsing
- Relevance scores guide result prioritization

## Performance Characteristics

Typical performance for 1000+ markdown files:

| Operation | Time | Notes |
|-----------|------|-------|
| Initial index | ~5-10s | One-time cost |
| Semantic search | <500ms | With cached index |
| Hybrid search | <300ms | Leverages both indices |
| Lexical search | <100ms | Full-text index |
| Incremental update | <1s | Only changed files |

## Comparison to Related Tools

### vs. Ripgrep/grep

- **Ripgrep**: Fast regex search, no semantic understanding
- **CK Search**: Semantic + regex, understands meaning, slightly slower

### vs. Obsidian Search

- **[[Obsidian]]**: Full-text search within Obsidian UI
- **CK Search**: Semantic search, CLI/TUI/editor integration, AI agent support

### vs. Semantic Scholar/Research Tools

- **Research tools**: Academic paper search, external databases
- **CK Search**: Local codebase/notes search, no external dependencies

### vs. RAG Systems

- **[[RAG]]**: Requires LLM API, generates answers
- **CK Search**: Local embeddings, returns matches, no generation

## Use Cases for Second-Brain

1. **Answering "What do I know about X?"**

   ```bash
   ck --sem "event driven architecture" .
   ck --sem "staff engineer career path" .
   ```

2. **Finding forgotten notes**

   ```bash
   ck --sem "domain driven design" . | grep -v "DDD"
   ```

   (Finds notes about the concept even if they don't mention "DDD")

3. **Connecting concepts for writing**

   ```bash
   ck --sem "platform engineering" .
   ck --sem "developer experience" .
   # Compare results to find connections
   ```

4. **Refining existing notes**

   ```bash
   # Find all notes on similar topic
   ck --sem --threshold 0.6 "kubernetes best practices" .
   # Identify gaps or redundancies
   ```

5. **Daily note preparation**

   ```bash
   # Review what you know before a meeting
   ck --sem "cato platform" work/
   ```

## Integration with OpenCode Agents

The [[AI Agents|AI agent]] setup guide specifically recommends CK Search for:

- Codebase onboarding (understanding new projects)
- Code review assistance (finding patterns)
- Refactoring support (finding similar code)
- Documentation generation (gathering related code)

For this second-brain, similar patterns apply:

- Knowledge onboarding (understanding existing notes)
- Note review assistance (finding related concepts)
- Content refactoring (consolidating similar topics)
- Writing generation (gathering research from notes)

## Limitations

1. **First-time indexing cost**: 5-10s for 1000 files (but only needed once)
2. **Local models only**: No cloud-based embeddings (privacy-first design)
3. **Markdown/code focus**: Optimized for text, not binary formats
4. **No generation**: Returns matches only, doesn't summarize or answer questions
5. **Threshold tuning**: Requires experimentation to find optimal values

See [[Context]] for why local processing matters for [[LLM|LLM]] workflows.

## Resources

- [CK Search Documentation](https://beaconbay.github.io/ck/)
- [Basic Usage Guide](https://beaconbay.github.io/ck/guide/basic-usage.html)
- [Advanced Usage Guide](https://beaconbay.github.io/ck/guide/advanced-usage.html)
- [AI Agent Setup Guide](https://beaconbay.github.io/ck/guide/ai-agent-setup.html)
- [GitHub Repository](https://github.com/BeaconBay/ck)
- [Serena](https://oraios.github.io/serena/) - Code-focused semantic retrieval toolkit with symbol-level tools
