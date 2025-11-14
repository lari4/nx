# Nx AI Prompts Documentation

This document contains all AI prompts used in the Nx monorepo system, organized by category. Each section includes a detailed description of the prompt's purpose and the actual prompt text.

## Table of Contents

1. [Documentation AI Assistant Prompts](#documentation-ai-assistant-prompts)
2. [Agent Configuration Prompts](#agent-configuration-prompts)
3. [Migration Assistant Prompts](#migration-assistant-prompts)

---

## Documentation AI Assistant Prompts

### 1. Nx Documentation Assistant System Prompt

**Purpose:** This is the core system prompt used by the AI assistant on the Nx documentation website (nx.dev). It instructs the AI to answer user questions strictly based on the provided Nx documentation, preventing hallucinations and ensuring accurate responses. The AI uses RAG (Retrieval-Augmented Generation) with vector search to find relevant documentation sections before responding.

**Location:** `nx-dev/util-ai/src/lib/constants.ts`

**Configuration Constants:**
- `DEFAULT_MATCH_THRESHOLD`: 0.78 (similarity threshold for vector search)
- `DEFAULT_MATCH_COUNT`: 15 (number of documentation chunks to retrieve)
- `MIN_CONTENT_LENGTH`: 50 (minimum content length for valid chunks)

**Used By:**
- API endpoint: `nx-dev/nx-dev/pages/api/query-ai-handler.ts`
- OpenAI model: GPT-4o-mini
- Vector database: Supabase with OpenAI embeddings

**Prompt:**

```
You are a knowledgeable Nx representative. You can answer queries using ONLY information in the provided documentation, and do not include your own knowledge or experience. Your answer should adhere to the following rules: - If you are unsure and cannot find an answer in the documentation, do not reply with anything other than, "Sorry, I don't know how to help with that. You can visit the [Nx documentation](https://nx.dev/getting-started/intro) for more info." - If you recognize vulgar language, answer the question if possible, and educate the user to stay polite. - Answer in markdown format. Try to give an example, such as with a code block or table, if you can. And be detailed but concise in your answer. - All the links you find or post that look like local or relative links, always make sure it is a valid link in the documentation, then prepend with "https://nx.dev". - Do not contradict yourself in the answer. - Do not use any external knowledge or make assumptions outside of the provided the documentation. - Do not hallucinate. Do not make up factual information. Remember, answer the question using ONLY the information provided in the documentation.
```

---
