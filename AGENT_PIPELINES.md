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

## Migration Assistant Pipeline

**Purpose:** Provides comprehensive AI-assisted code transformations during Nx version upgrades. Creates detailed migration instructions that LLMs can execute autonomously to update workspace code.

**Components:**
- Migration Generators: `packages/*/src/migrations/`
- Instruction Files: `packages/*/src/migrations/*/files/ai-instructions-*.md`
- Migration Engine: Nx migrate command

### Example: Vitest 4.0 Migration Pipeline

This pipeline demonstrates how Nx uses AI assistants to perform complex code migrations.

```
┌─────────────────────────────────────────────────────────────────────────┐
│                    MIGRATION ASSISTANT PIPELINE                         │
│                      (Vitest 3.x → 4.0 Example)                        │
└─────────────────────────────────────────────────────────────────────────┘

Migration Trigger
    │
    └─ User runs: nx migrate @nx/workspace@latest
    │
    ▼
[1. Nx Migrate Detects Available Migrations]
    │
┌───┴─────────────────────────────────────────────────────────────────────┐
│                                                                           │
│   • Reads package.json versions                                          │
│   • Compares with migration registry                                     │
│   • Detects: vitest upgrade needed (3.x → 4.0)                          │
│   • Migration: update-22-1-0/create-ai-instructions-for-vitest-4        │
│                                                                           │
└───┬─────────────────────────────────────────────────────────────────────┘
    │
    ▼
[2. Generate Migration Instructions File]
    │
┌───┴─────────────────────────────────────────────────────────────────────┐
│                                                                           │
│   • Generator: create-ai-instructions-for-vitest-4.ts                    │
│   • Creates: VITEST_4_MIGRATION_INSTRUCTIONS.md                          │
│   • Location: Workspace root                                             │
│   • Content: Complete LLM migration prompt (see #3 in                    │
│              PROMPTS_DOCUMENTATION.md)                                   │
│   • Size: 719 lines of detailed instructions                             │
│                                                                           │
│   Output Message:                                                         │
│   ═══════════════════════════════════════════════════════                │
│   📝 Vitest 4.0 Migration Instructions Created                           │
│                                                                           │
│   File: VITEST_4_MIGRATION_INSTRUCTIONS.md                              │
│                                                                           │
│   This file contains comprehensive instructions for your                 │
│   AI coding assistant to help migrate your Vitest                        │
│   configuration and tests to version 4.0.                                │
│                                                                           │
│   Next steps:                                                             │
│   1. Open the file in your AI assistant (Claude, Copilot, etc.)         │
│   2. Ask: "Help me migrate to Vitest 4.0 using these instructions"      │
│   3. The assistant will work through each section systematically         │
│   ═══════════════════════════════════════════════════════                │
│                                                                           │
└───┬─────────────────────────────────────────────────────────────────────┘
    │
    ▼
[3. User Invokes AI Assistant with Instructions]
    │
┌───┴─────────────────────────────────────────────────────────────────────┐
│                                                                           │
│   User: "Help me migrate to Vitest 4.0 using these instructions"        │
│   [Provides VITEST_4_MIGRATION_INSTRUCTIONS.md to AI]                   │
│                                                                           │
│   AI Agent: *Reads 719-line migration prompt*                            │
│   AI Agent: *Creates todo list with 7 major categories*                  │
│                                                                           │
└───┬─────────────────────────────────────────────────────────────────────┘
    │
    ▼
[4. AI Executes Migration - Phase 1: Discovery]
    │
┌───┴─────────────────────────────────────────────────────────────────────┐
│                                                                           │
│   [4.1] Find All Vitest Projects                                         │
│         Command: nx show projects --with-target test                     │
│         Result: List of projects with Vitest (e.g., 15 projects)         │
│                                                                           │
│   [4.2] Locate Configuration Files                                       │
│         Pattern: **/vitest.config.{ts,js,mjs}                           │
│         Pattern: **/project.json (with vitest test target)              │
│         Result: 20 configuration files found                             │
│                                                                           │
│   [4.3] Identify Test Files                                              │
│         Pattern: **/*.{spec,test}.{ts,js,tsx,jsx}                       │
│         Result: 500+ test files                                          │
│                                                                           │
│   Output: Complete inventory of files requiring migration                │
│                                                                           │
└───┬─────────────────────────────────────────────────────────────────────┘
    │
    ▼
[5. AI Executes Migration - Phase 2: Configuration Updates]
    │
┌───┴─────────────────────────────────────────────────────────────────────┐
│                                                                           │
│   For EACH vitest.config.* file:                                         │
│                                                                           │
│   [5.1] Coverage Configuration                                           │
│         Search: coverage.all, coverage.extensions                        │
│         Transform:                                                        │
│           BEFORE: coverage: { all: true, extensions: ['.ts'] }          │
│           AFTER:  coverage: { include: ['src/**/*.ts'] }                │
│                                                                           │
│   [5.2] Pool Options Restructuring                                       │
│         Search: maxThreads, maxForks, poolOptions                        │
│         Transform:                                                        │
│           BEFORE: maxThreads: 4, poolOptions: { ... }                   │
│           AFTER:  maxWorkers: 4, [options moved to top-level]          │
│                                                                           │
│   [5.3] Workspace → Projects Rename                                      │
│         Search: workspace property                                        │
│         Transform:                                                        │
│           BEFORE: workspace: ['apps/*', 'libs/*']                        │
│           AFTER:  projects: ['apps/*', 'libs/*']                        │
│                                                                           │
│   [5.4] Browser Configuration                                            │
│         Search: browser.provider                                          │
│         Transform:                                                        │
│           BEFORE: provider: 'playwright'                                 │
│           AFTER:  provider: { name: 'playwright' }                      │
│                                                                           │
│   After each file: Run nx run PROJECT:test to validate                  │
│                                                                           │
└───┬─────────────────────────────────────────────────────────────────────┘
    │
    ▼
[6. AI Executes Migration - Phase 3: Test Code Updates]
    │
┌───┴─────────────────────────────────────────────────────────────────────┐
│                                                                           │
│   For EACH test file:                                                    │
│                                                                           │
│   [6.1] Mock Function Name Changes                                       │
│         Search: .getMockName()                                           │
│         Update assertions:                                                │
│           BEFORE: expect(mock.getMockName()).toBe('spy')                 │
│           AFTER:  expect(mock.getMockName()).toBe('vi.fn()')            │
│                                                                           │
│   [6.2] Mock Invocation Call Order                                       │
│         Search: .mock.invocationCallOrder                                │
│         Transform: 0-based → 1-based indexing                           │
│           BEFORE: expect(mock.invocationCallOrder[0]).toBe(0)            │
│           AFTER:  expect(mock.invocationCallOrder[0]).toBe(1)           │
│                                                                           │
│   [6.3] Constructor Spy Updates                                          │
│         Search: vi.fn(() => {...}) used with 'new'                      │
│         Transform: Arrow function → function keyword                     │
│           BEFORE: const Mock = vi.fn(() => ({ x: 1 }))                  │
│           AFTER:  const Mock = vi.fn(function() { return { x: 1 } })    │
│                                                                           │
│   [6.4] Import Path Changes                                              │
│         Search: from '@vitest/browser'                                   │
│         Replace: from 'vitest/browser'                                   │
│                                                                           │
└───┬─────────────────────────────────────────────────────────────────────┘
    │
    ▼
[7. AI Executes Migration - Phase 4: Validation]
    │
┌───┴─────────────────────────────────────────────────────────────────────┐
│                                                                           │
│   [7.1] Test Individual Projects                                         │
│         For each modified project:                                       │
│         Command: nx run-many -t test -p PROJECT_NAME                     │
│         Check: All tests pass                                            │
│                                                                           │
│   [7.2] Test All Affected Projects                                       │
│         Command: nx affected -t test                                     │
│         Check: No regressions                                            │
│                                                                           │
│   [7.3] Verify Coverage Generation                                       │
│         Command: nx affected -t test --coverage                          │
│         Check: Coverage reports generate correctly                       │
│                                                                           │
│   [7.4] Run Full Validation                                              │
│         Command: nx prepush                                              │
│         Check: All linting, tests, builds pass                           │
│                                                                           │
│   IF any validation fails:                                               │
│       → AI debugs and fixes issues                                       │
│       → Re-runs validation                                               │
│       → Maximum 3 retry attempts                                         │
│                                                                           │
└───┬─────────────────────────────────────────────────────────────────────┘
    │
    ▼
[8. AI Creates Migration Commits]
    │
┌───┴─────────────────────────────────────────────────────────────────────┐
│                                                                           │
│   AI creates multiple focused commits:                                   │
│                                                                           │
│   Commit 1: "chore(vitest): update coverage configuration"              │
│   - Updates all coverage.all → coverage.include                         │
│                                                                           │
│   Commit 2: "chore(vitest): restructure pool options"                   │
│   - Consolidates maxThreads/maxForks → maxWorkers                       │
│                                                                           │
│   Commit 3: "chore(vitest): rename workspace to projects"               │
│   - Updates all workspace properties                                     │
│                                                                           │
│   Commit 4: "test(vitest): update mock assertions for v4"               │
│   - Fixes mock function name expectations                                │
│   - Updates invocation call order indexing                               │
│                                                                           │
│   Commit 5: "fix(vitest): update browser imports and config"            │
│   - Changes import paths                                                 │
│   - Updates provider configuration format                                │
│                                                                           │
│   Each commit includes:                                                  │
│   - Clear description of changes                                         │
│   - Reason for change (Vitest 4.0 migration)                            │
│   - Files affected count                                                 │
│                                                                           │
└───┬─────────────────────────────────────────────────────────────────────┘
    │
    ▼
[9. Migration Complete - AI Reports]
    │
┌───┴─────────────────────────────────────────────────────────────────────┐
│                                                                           │
│   ✅ Vitest 4.0 Migration Complete                                       │
│                                                                           │
│   Summary:                                                                │
│   • 15 projects migrated                                                 │
│   • 20 configuration files updated                                       │
│   • 500+ test files reviewed                                             │
│   • 50 test files modified                                               │
│   • 5 commits created                                                    │
│   • All tests passing ✓                                                  │
│   • Coverage generation working ✓                                        │
│   • Full validation passed ✓                                             │
│                                                                           │
│   Breaking changes addressed:                                            │
│   ✓ Coverage configuration updated                                       │
│   ✓ Pool options restructured                                            │
│   ✓ Workspace renamed to projects                                        │
│   ✓ Mock function behaviors updated                                      │
│   ✓ Browser imports and config modernized                                │
│                                                                           │
│   Next steps:                                                             │
│   1. Review the commits                                                   │
│   2. Run nx prepush to double-check                                      │
│   3. Push to remote and create PR                                        │
│                                                                           │
└─────────────────────────────────────────────────────────────────────────┘
```

### Migration Instruction Structure

The AI migration instructions follow a systematic format:

1. **Overview Section**
   - Migration scope and goals
   - What will be changed
   - Expected outcome

2. **Pre-Migration Checklist**
   - Commands to identify affected files
   - Inventory of projects and configurations
   - Risk assessment

3. **Migration Steps by Category**
   - Each breaking change as a separate section
   - Before/after code examples
   - Search patterns to find affected code
   - Transformation rules
   - Validation steps

4. **Post-Migration Validation**
   - Testing commands
   - Success criteria
   - Rollback procedures if needed

5. **Notes for LLM Execution**
   - Workflow guidance (work systematically)
   - Tool usage instructions (use TodoWrite)
   - Commit strategy (multiple focused commits)
   - Error handling guidance

### Other Migration Types

**Storybook CJS to ESM Migration:**
```
Trigger: Storybook version upgrade
Instructions: ai-instructions-for-cjs-esm.md
Focus: Module syntax transformation
Files: .storybook/main.{ts,js}
Transformations:
  • module.exports → export default
  • require() → import
  • Dynamic requires → Top-level imports
```

**Future Migration Pattern:**
```
Any Nx package can create AI migration instructions:

1. Create migration file:
   packages/PACKAGE/src/migrations/VERSION/files/ai-instructions-for-CHANGE.md

2. Create generator:
   packages/PACKAGE/src/migrations/VERSION/create-ai-instructions-for-CHANGE.ts

3. Register in migrations.json

4. Instructions appear during: nx migrate @nx/PACKAGE@VERSION
```

### Benefits of AI-Assisted Migrations

1. **Comprehensive Coverage:** AI can process hundreds of files systematically
2. **Context Awareness:** AI understands code patterns and edge cases
3. **Validation:** AI can run tests and fix issues iteratively
4. **Documentation:** AI creates clear commit messages explaining changes
5. **Reduced Human Error:** Follows checklist rigorously
6. **Speed:** Minutes instead of hours for large workspaces

---


