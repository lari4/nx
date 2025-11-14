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

