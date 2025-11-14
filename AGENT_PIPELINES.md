# Nx AI Agent Pipelines Documentation

This document describes all AI agent workflows and pipelines in the Nx ecosystem, showing how different components interact, which prompts are used, and how data flows through the system.

## Table of Contents

1. [Documentation AI Assistant Pipeline](#documentation-ai-assistant-pipeline)
2. [Agent Configuration Pipeline](#agent-configuration-pipeline)
3. [Migration Assistant Pipeline](#migration-assistant-pipeline)
4. [CI Error Resolution Pipeline](#ci-error-resolution-pipeline)

---

## Documentation AI Assistant Pipeline

**Purpose:** Enables users to ask questions about Nx on the documentation website (nx.dev) and receive accurate answers based on documentation content using RAG (Retrieval-Augmented Generation).

**Components:**
- Frontend: React components in `nx-dev/feature-ai/`
- API: Next.js API routes in `nx-dev/nx-dev/pages/api/`
- Utilities: AI utilities in `nx-dev/util-ai/`
- Database: Supabase (vector storage)
- LLM: OpenAI GPT-4o-mini

### Pipeline Flow

```
┌─────────────────────────────────────────────────────────────────────────┐
│                    DOCUMENTATION AI ASSISTANT PIPELINE                  │
└─────────────────────────────────────────────────────────────────────────┘

User Query
    │
    ├──[1. Input Validation]──────────────────────────────────────────────┐
    │                                                                       │
    │   • Component: prompt.tsx                                            │
    │   • Validation: Non-empty, reasonable length                         │
    │   • Moderation: moderateContent() check                             │
    │                                                                       │
    └───────────────────────────┬───────────────────────────────────────────┘
                                │
                                ▼
                    [2. Generate Query Embedding]
                                │
    ┌───────────────────────────┴───────────────────────────────────────────┐
    │                                                                         │
    │   • Function: getTokenizedContext()                                    │
    │   • Model: text-embedding-ada-002 (OpenAI)                            │
    │   • Input: User query string                                           │
    │   • Output: 1536-dimensional vector                                    │
    │                                                                         │
    └───────────────────────────┬─────────────────────────────────────────────┘
                                │
                                ▼
                    [3. Vector Similarity Search]
                                │
    ┌───────────────────────────┴───────────────────────────────────────────┐
    │                                                                         │
    │   • Database: Supabase (nods.page_section table)                      │
    │   • Function: match_page_sections (RPC call)                           │
    │   • Similarity threshold: 0.78 (DEFAULT_MATCH_THRESHOLD)              │
    │   • Max results: 15 (DEFAULT_MATCH_COUNT)                             │
    │   • Output: Relevant documentation chunks with metadata               │
    │                                                                         │
    └───────────────────────────┬─────────────────────────────────────────────┘
                                │
                                ▼
                    [4. Context Tokenization & Preparation]
                                │
    ┌───────────────────────────┴───────────────────────────────────────────┐
    │                                                                         │
    │   • Function: getTokenizedContext()                                    │
    │   • Tokenizer: GPT-3 tokenizer                                         │
    │   • Context assembly: Concatenate matched sections                     │
    │   • Format: Plain text with section separators                         │
    │   • Output: { tokenizedContext, contextText, sources }                │
    │                                                                         │
    └───────────────────────────┬─────────────────────────────────────────────┘
                                │
                                ▼
                    [5. Initialize Chat with System Prompt]
                                │
    ┌───────────────────────────┴───────────────────────────────────────────┐
    │                                                                         │
    │   • Function: initializeChat()                                         │
    │   • Prompt: PROMPT constant (see PROMPTS_DOCUMENTATION.md #1)         │
    │   • Messages structure:                                                │
    │       [                                                                 │
    │         { role: "system", content: PROMPT },                           │
    │         { role: "user", content: contextText },                        │
    │         { role: "user", content: userQuery }                           │
    │       ]                                                                 │
    │                                                                         │
    └───────────────────────────┬─────────────────────────────────────────────┘
                                │
                                ▼
                    [6. Stream LLM Response]
                                │
    ┌───────────────────────────┴───────────────────────────────────────────┐
    │                                                                         │
    │   • API: OpenAI Chat Completions API                                   │
    │   • Model: gpt-4o-mini                                                 │
    │   • Stream: true (SSE - Server-Sent Events)                            │
    │   • Temperature: 0.0 (deterministic)                                   │
    │   • Handler: /api/query-ai-handler                                     │
    │                                                                         │
    └───────────────────────────┬─────────────────────────────────────────────┘
                                │
                                ▼
                    [7. Format & Display Response]
                                │
    ┌───────────────────────────┴───────────────────────────────────────────┐
    │                                                                         │
    │   • Component: feed-answer.tsx                                         │
    │   • Format: Markdown rendering                                         │
    │   • Sources: formatMarkdownSources()                                   │
    │   • Display: Streaming text with syntax highlighting                   │
    │                                                                         │
    └───────────────────────────┬─────────────────────────────────────────────┘
                                │
                                ▼
                        [8. Store History]
                                │
    ┌───────────────────────────┴───────────────────────────────────────────┐
    │                                                                         │
    │   • Function: storeQueryForUid()                                       │
    │   • Storage: Supabase (query history table)                            │
    │   • Data: { uid, query, response, timestamp }                          │
    │                                                                         │
    └─────────────────────────────────────────────────────────────────────────┘
```

### Data Flow Details

#### Step 1: Input Validation
- **File:** `nx-dev/feature-ai/src/lib/prompt.tsx`
- **Input:** Raw user query string
- **Process:** Validate non-empty, run moderation check
- **Output:** Validated query or error message

#### Step 2: Generate Query Embedding
- **File:** `nx-dev/nx-dev/lib/getTokenizedContext.ts`
- **API Call:** `openai.embeddings.create()`
- **Input Data:**
  ```typescript
  {
    model: 'text-embedding-ada-002',
    input: sanitizedQuery
  }
  ```
- **Output:** Vector embedding `[0.123, -0.456, ...]` (1536 dimensions)

#### Step 3: Vector Similarity Search
- **File:** `nx-dev/nx-dev/lib/getTokenizedContext.ts`
- **Database Function:** Supabase RPC `match_page_sections`
- **Input Data:**
  ```typescript
  {
    embedding: queryEmbedding,
    match_threshold: 0.78,
    match_count: 15,
    min_content_length: 50
  }
  ```
- **SQL Query (conceptual):**
  ```sql
  SELECT id, slug, heading, content, similarity
  FROM nods.page_section
  WHERE similarity(embedding, $1) > 0.78
  ORDER BY similarity DESC
  LIMIT 15
  ```
- **Output:** Array of documentation chunks with metadata

#### Step 4: Context Preparation
- **File:** `nx-dev/util-ai/src/lib/chat-utils.ts`
- **Process:** Concatenate matched sections into context
- **Format:**
  ```
  Section: Getting Started
  Content: [documentation content]

  Section: Configuration
  Content: [documentation content]
  ```
- **Output:** `{ tokenizedContext, contextText, sources }`

#### Step 5: Initialize Chat
- **File:** `nx-dev/util-ai/src/lib/chat-utils.ts`
- **Function:** `initializeChat()`
- **Messages Array:**
  ```typescript
  [
    {
      role: 'system',
      content: PROMPT  // "You are a knowledgeable Nx representative..."
    },
    {
      role: 'user',
      content: contextText  // Documentation context
    },
    {
      role: 'user',
      content: query  // User's actual question
    }
  ]
  ```

#### Step 6: Stream LLM Response
- **File:** `nx-dev/nx-dev/pages/api/query-ai-handler.ts`
- **API Call:** `openai.chat.completions.create()`
- **Parameters:**
  ```typescript
  {
    model: 'gpt-4o-mini',
    messages: chatMessages,
    temperature: 0,
    stream: true
  }
  ```
- **Streaming:** Server-Sent Events (SSE) to client

#### Step 7: Display Response
- **Component:** `feed-answer.tsx`
- **Rendering:** Markdown with code syntax highlighting
- **Sources:** Links to original documentation pages

#### Step 8: Store History
- **File:** `nx-dev/util-ai/src/lib/history.ts`
- **Storage:** Supabase query_history table
- **Purpose:** Analytics and user session tracking

### Error Handling

```
Error Points:
├── [1] Moderation failure → Show warning message
├── [2] Embedding API failure → Retry or fallback to keyword search
├── [3] No matching docs → "I don't know" response (per system prompt)
├── [4] Tokenization overflow → Truncate context
├── [5] LLM API failure → Show error message
└── [6] Geographic restriction → "Service not available" message
```

---

## Agent Configuration Pipeline

**Purpose:** Automatically configures AI agents (Claude Code, GitHub Copilot, Cursor, Gemini, Codex) to work effectively with Nx workspaces by creating configuration files and injecting Nx-specific guidelines.

**Components:**
- CLI Prompts: `packages/nx/src/command-line/init/ai-agent-prompts.ts`
- Generator: `packages/nx/src/ai/set-up-ai-agents/set-up-ai-agents.ts`
- Rules Generator: `packages/nx/src/ai/set-up-ai-agents/get-agent-rules.ts`
- MCP Integration: `packages/nx/src/command-line/mcp/mcp.ts`

### Pipeline Flow

```
┌─────────────────────────────────────────────────────────────────────────┐
│                     AGENT CONFIGURATION PIPELINE                        │
└─────────────────────────────────────────────────────────────────────────┘

Trigger Event
    │
    ├─[A] nx init (existing workspace)
    ├─[B] create-nx-workspace (new workspace)
    └─[C] nx configure-ai-agents (manual command)
    │
    ▼
[1. Determine Which Agents to Configure]
    │
┌───┴─────────────────────────────────────────────────────────────────────┐
│                                                                           │
│   IF: CLI args provided (--aiAgents=claude,copilot)                     │
│       → Skip prompt, use provided agents                                 │
│                                                                           │
│   IF: Non-interactive mode or CI environment                             │
│       → Skip agent configuration                                          │
│                                                                           │
│   ELSE: Show interactive prompt                                          │
│       → Component: aiAgentsPrompt()                                      │
│       → Prompt: "Which AI agents would you like to set up?"             │
│       → Type: Multi-select (Space to select, Enter to confirm)          │
│       → Options:                                                          │
│           • Claude Code                                                   │
│           • GitHub Copilot                                                │
│           • Cursor                                                        │
│           • Google Gemini                                                 │
│           • OpenAI Codex                                                  │
│                                                                           │
│   Output: Array<Agent> (e.g., ['claude', 'copilot'])                    │
│                                                                           │
└───┬─────────────────────────────────────────────────────────────────────┘
    │
    ▼
[2. Detect Nx Cloud Status]
    │
┌───┴─────────────────────────────────────────────────────────────────────┐
│                                                                           │
│   • Check: Look for nx.json with nxCloud configuration                  │
│   • Check: Environment variables (NX_CLOUD_ACCESS_TOKEN, etc.)          │
│   • Output: boolean (nxCloudEnabled)                                     │
│   • Purpose: Determines if CI error guidelines should be included       │
│                                                                           │
└───┬─────────────────────────────────────────────────────────────────────┘
    │
    ▼
[3. Generate Agent Rules Content]
    │
┌───┴─────────────────────────────────────────────────────────────────────┐
│                                                                           │
│   • Function: getAgentRules(nxCloud)                                     │
│   • Prompt Used: See PROMPTS_DOCUMENTATION.md #2                         │
│   • Base Content:                                                         │
│       - Nx command usage guidelines                                       │
│       - MCP tool usage instructions                                       │
│       - Best practices for Nx workspaces                                  │
│                                                                           │
│   • IF nxCloud === true:                                                 │
│       Append CI Error Guidelines (PROMPTS_DOCUMENTATION.md #5)           │
│                                                                           │
│   • Output: String (markdown content)                                    │
│                                                                           │
└───┬─────────────────────────────────────────────────────────────────────┘
    │
    ▼
[4. Create Configuration Files for Each Agent]
    │
┌───┴─────────────────────────────────────────────────────────────────────┐
│                                                                           │
│   For EACH selected agent:                                               │
│                                                                           │
│   ┌─ CLAUDE CODE ────────────────────────────────────────────────────┐  │
│   │                                                                    │  │
│   │   • Create/Update: CLAUDE.md (workspace root)                     │  │
│   │   • Content: Repository-specific instructions + agent rules      │  │
│   │   • Sections:                                                      │  │
│   │       - Essential Commands                                         │  │
│   │       - Testing workflows                                          │  │
│   │       - GitHub Issue Response Mode                                 │  │
│   │       - Agent rules (injected between markers)                     │  │
│   │   • Markers:                                                       │  │
│   │       <!-- nx configuration start-->                              │  │
│   │       [Agent rules content]                                        │  │
│   │       <!-- nx configuration end-->                                │  │
│   │                                                                    │  │
│   └────────────────────────────────────────────────────────────────────┘  │
│                                                                           │
│   ┌─ GITHUB COPILOT / CURSOR ────────────────────────────────────────┐  │
│   │                                                                    │  │
│   │   • Create/Update: AGENTS.md (workspace root)                     │  │
│   │   • Content: Generic agent instructions                           │  │
│   │   • Same structure as CLAUDE.md but without repo-specific rules  │  │
│   │                                                                    │  │
│   └────────────────────────────────────────────────────────────────────┘  │
│                                                                           │
│   ┌─ GOOGLE GEMINI ──────────────────────────────────────────────────┐  │
│   │                                                                    │  │
│   │   • Create: .gemini/ directory                                    │  │
│   │   • Create: .gemini/settings.json                                 │  │
│   │   • Content (JSON):                                                │  │
│   │       {                                                            │  │
│   │         "instructions": [                                          │  │
│   │           {                                                         │  │
│   │             "instruction": "nx configuration",                     │  │
│   │             "content": "# General Guidelines for working with Nx"  │  │
│   │           }                                                         │  │
│   │         ]                                                           │  │
│   │       }                                                             │  │
│   │                                                                    │  │
│   └────────────────────────────────────────────────────────────────────┘  │
│                                                                           │
│   ┌─ OPENAI CODEX ───────────────────────────────────────────────────┐  │
│   │                                                                    │  │
│   │   • Create: .codex/ directory                                     │  │
│   │   • Create: .codex/config.toml                                    │  │
│   │   • Content (TOML):                                                │  │
│   │       [mcp.nx]                                                     │  │
│   │       command = "npx"                                              │  │
│   │       args = ["-y", "@nx/mcp"]                                    │  │
│   │                                                                    │  │
│   └────────────────────────────────────────────────────────────────────┘  │
│                                                                           │
│   ┌─ MCP CONFIGURATION ──────────────────────────────────────────────┐  │
│   │                                                                    │  │
│   │   • Create: .mcp.json (workspace root)                            │  │
│   │   • Content:                                                       │  │
│   │       {                                                            │  │
│   │         "mcpServers": {                                            │  │
│   │           "nx": {                                                  │  │
│   │             "command": "npx",                                      │  │
│   │             "args": ["-y", "@nx/mcp"]                             │  │
│   │           }                                                         │  │
│   │         }                                                           │  │
│   │       }                                                             │  │
│   │   • Purpose: Enables MCP server for workspace understanding       │  │
│   │                                                                    │  │
│   └────────────────────────────────────────────────────────────────────┘  │
│                                                                           │
└───┬─────────────────────────────────────────────────────────────────────┘
    │
    ▼
[5. Update .gitignore]
    │
┌───┴─────────────────────────────────────────────────────────────────────┐
│                                                                           │
│   • Read existing .gitignore                                             │
│   • Check if AI config files are already ignored                         │
│   • Add if missing:                                                      │
│       # AI Agent Configuration                                           │
│       .codex/                                                            │
│       .gemini/                                                           │
│       .mcp.json                                                          │
│   • Note: CLAUDE.md and AGENTS.md are typically committed               │
│                                                                           │
└───┬─────────────────────────────────────────────────────────────────────┘
    │
    ▼
[6. Display Success Message]
    │
┌───┴─────────────────────────────────────────────────────────────────────┐
│                                                                           │
│   Output to user:                                                        │
│                                                                           │
│   ✅ AI agents configured successfully!                                 │
│                                                                           │
│   Configured agents:                                                     │
│   • Claude Code (CLAUDE.md)                                             │
│   • GitHub Copilot (AGENTS.md)                                          │
│                                                                           │
│   MCP Server: Configured in .mcp.json                                   │
│                                                                           │
│   Next steps:                                                            │
│   1. Restart your AI agent/IDE to load new configuration                │
│   2. Try asking your agent about Nx workspace structure                 │
│                                                                           │
└─────────────────────────────────────────────────────────────────────────┘
```

### Configuration File Templates

#### CLAUDE.md Structure
```markdown
[Repository-specific instructions]

<!-- nx configuration start-->
# General Guidelines for working with Nx
[Agent rules content from getAgentRules()]

# CI Error Guidelines
[Optional: If Nx Cloud enabled]
<!-- nx configuration end-->
```

#### AGENTS.md Structure
```markdown
<!-- nx configuration start-->
# General Guidelines for working with Nx
[Agent rules content from getAgentRules()]

# CI Error Guidelines
[Optional: If Nx Cloud enabled]
<!-- nx configuration end-->
```

#### .gemini/settings.json Structure
```json
{
  "instructions": [
    {
      "instruction": "nx configuration",
      "content": "[Agent rules markdown content]"
    }
  ]
}
```

#### .mcp.json Structure
```json
{
  "mcpServers": {
    "nx": {
      "command": "npx",
      "args": ["-y", "@nx/mcp"]
    }
  }
}
```

#### .codex/config.toml Structure
```toml
[mcp.nx]
command = "npx"
args = ["-y", "@nx/mcp"]
```

### MCP Tools Available After Configuration

Once configured, agents can use these MCP tools:

1. **nx_workspace**
   - Purpose: Get workspace architecture overview
   - Returns: Project list, dependencies, configuration errors
   - Usage: "Show me the workspace structure"

2. **nx_project_details**
   - Purpose: Analyze individual project
   - Input: Project name
   - Returns: Project config, dependencies, targets
   - Usage: "What are the dependencies of the 'api' project?"

3. **nx_docs**
   - Purpose: Query Nx documentation
   - Input: Search query
   - Returns: Relevant documentation content
   - Usage: "How do I configure caching in Nx?"

4. **nx_cloud_cipe_details** (if Nx Cloud enabled)
   - Purpose: Get CI pipeline execution details
   - Returns: List of CIPEs with status and failures
   - Usage: "Show me recent CI failures"

5. **nx_cloud_fix_cipe_failure** (if Nx Cloud enabled)
   - Purpose: Get detailed logs for failed tasks
   - Input: CIPE ID and task ID
   - Returns: Task logs and error details
   - Usage: "Show me logs for the failed build task"

### Agent Behavior After Configuration

**Before Configuration:**
```
User: "Run tests for the affected projects"
Agent: *Runs npm test or yarn test directly*
```

**After Configuration:**
```
User: "Run tests for the affected projects"
Agent: *Reads agent rules from CLAUDE.md/AGENTS.md*
Agent: *Uses nx affected -t test (following guidelines)*
```

### Update Mechanism

Configuration files use markers to allow updates:

```markdown
<!-- nx configuration start-->
[Content managed by Nx generator]
<!-- nx configuration end-->
```

When running `nx configure-ai-agents` again:
1. Generator finds existing markers
2. Preserves custom content outside markers
3. Updates only content between markers
4. Prevents accidental overwrites

---


