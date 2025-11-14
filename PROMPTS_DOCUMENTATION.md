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

## Agent Configuration Prompts

### 2. Nx MCP (Model Context Protocol) Agent Rules

**Purpose:** These are the core instructions provided to AI agents (Claude Code, GitHub Copilot, Cursor, Google Gemini, OpenAI Codex) when they are configured to work with Nx workspaces. The prompt guides agents to use Nx commands properly, leverage the MCP server for workspace understanding, and follow Nx best practices. This prompt is injected into agent configuration files during setup.

**Location:** `packages/nx/src/ai/set-up-ai-agents/get-agent-rules.ts`

**Injected Into:**
- `CLAUDE.md` - For Claude Code
- `AGENTS.md` - For GitHub Copilot, Cursor, and other general agents
- `.gemini/settings.json` - For Google Gemini
- `.mcp.json` - Model Context Protocol configuration
- `.codex/config.toml` - For OpenAI Codex

**Used By:**
- Command: `nx configure-ai-agents`
- Generator: `@nx/nx:set-up-ai-agents`
- During workspace initialization: `nx init`

**Dynamic Parameters:**
- `nxCloud` (boolean): Whether Nx Cloud is enabled (adds CI error handling instructions)

**Prompt:**

```
# General Guidelines for working with Nx

- When running tasks (for example build, lint, test, e2e, etc.), always prefer running the task through `nx` (i.e. `nx run`, `nx run-many`, `nx affected`) instead of using the underlying tooling directly
- You have access to the Nx MCP server and its tools, use them to help the user
- When answering questions about the repository, use the `nx_workspace` tool first to gain an understanding of the workspace architecture where applicable.
- When working in individual projects, use the `nx_project_details` mcp tool to analyze and understand the specific project structure and dependencies
- For questions around nx configuration, best practices or if you're unsure, use the `nx_docs` tool to get relevant, up-to-date docs. Always use this instead of assuming things about nx configuration
- If the user needs help with an Nx configuration or project graph error, use the `nx_workspace` tool to get any errors
```

**Note:** When `nxCloud` is enabled, additional CI error handling guidelines are appended (documented in CI Error Guidelines section below).

---
