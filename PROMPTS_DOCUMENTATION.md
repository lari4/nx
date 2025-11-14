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

## Migration Assistant Prompts

### 3. Vitest 4.0 Migration Instructions for LLM

**Purpose:** Comprehensive instructions for AI agents to systematically migrate Nx workspace projects from Vitest 3.x to Vitest 4.0. This prompt covers all breaking changes including configuration updates, test code modifications, reporter API changes, and validation steps. It's designed to be executed by an LLM autonomously with clear checklists and transformation patterns.

**Location:** `packages/vitest/src/migrations/update-22-1-0/files/ai-instructions-for-vitest-4.md`

**Created By:** Migration generator at `packages/vitest/src/migrations/update-22-1-0/create-ai-instructions-for-vitest-4.ts`

**Trigger:** Automatically created when running Nx migrations for Vitest 4.0 upgrade

**Target Files:**
- `vitest.config.{ts,js,mjs}` - Vitest configuration files
- `vitest.workspace.{ts,js,mjs}` - Workspace configuration files
- `**/*.{spec,test}.{ts,js,tsx,jsx}` - Test files
- `project.json` files with test targets

**Migration Categories Covered:**
1. Configuration File Updates (coverage, pool options, workspace rename)
2. Test Code Updates (mock function changes, spy behavior)
3. Reporter and CLI Changes
4. Snapshot Changes
5. Environment Variable Updates
6. Module Runner Changes
7. Type Definition Updates

**Key Transformation Patterns:**
- `coverage.all` → `coverage.include` (explicit patterns)
- `maxThreads`/`maxForks` → `maxWorkers` (unified)
- `workspace` → `projects` (renamed property)
- `vi.fn()` default name: `'spy'` → `'vi.fn()'`
- `invocationCallOrder`: 0-based → 1-based indexing
- `browser.provider: 'playwright'` → `browser.provider: { name: 'playwright' }`
- `@vitest/browser` → `vitest/browser` (import path change)

**Validation Commands:**
- `nx show projects --with-target test` - Find all Vitest projects
- `nx run-many -t test -p PROJECT_NAME` - Test individual projects
- `nx affected -t test` - Test all affected projects
- `nx prepush` - Full validation suite

**Prompt:** (See full content in file `packages/vitest/src/migrations/update-22-1-0/files/ai-instructions-for-vitest-4.md`)

Due to the length of this prompt (719 lines), please refer to the source file for complete instructions. Key sections include:

```markdown
# Vitest 4.0 Migration Instructions for LLM

## Overview
These instructions guide you through migrating an Nx workspace containing multiple
Vitest projects from Vitest 3.x to Vitest 4.0. Work systematically through each
breaking change category.

## Pre-Migration Checklist
1. Identify all Vitest projects
2. Locate all Vitest configuration files
3. Identify affected code

## Migration Steps by Category

### 1. Configuration File Updates
#### 1.1 Coverage Configuration
- Remove: coverage.all, coverage.extensions, coverage.ignoreEmptyLines
- Add: explicit coverage.include patterns

#### 1.2 Pool Options Restructuring
- Replace maxThreads and maxForks with single maxWorkers option
- Move poolOptions.* nested options to top-level

[... continues with all 7 categories ...]

## Post-Migration Validation
1. Run tests per project
2. Run all tests
3. Check coverage
4. Validate CI pipeline
5. Review migration checklist

## Notes for LLM Execution
- Work systematically: Complete one category before moving to the next
- Test after each change
- Use TodoWrite tool to track progress
- Create meaningful commits
```

---

### 4. Storybook CommonJS to ESM Migration Instructions for LLM

**Purpose:** Instructions for AI agents to transform Storybook configuration files from CommonJS (CJS) module syntax to ES Modules (ESM) syntax. This migration ensures compatibility with modern JavaScript tooling and Storybook versions that expect ESM configuration. The prompt provides step-by-step transformation patterns for common CJS patterns found in Storybook configs.

**Location:** `packages/storybook/src/migrations/update-22-1-0/files/ai-instructions-for-cjs-esm.md`

**Trigger:** Automatically created during Storybook migration updates

**Target Files:**
- `**/.storybook/main.js` - Storybook main configuration (JavaScript)
- `**/.storybook/main.ts` - Storybook main configuration (TypeScript)

**Transformation Patterns:**

1. **Module Exports:**
   - `module.exports = { ... }` → `export default { ... }`

2. **Require to Import:**
   - `const { x } = require('pkg')` → `import { x } from 'pkg'`

3. **Path Handling:**
   - `const path = require('path')` + `path.join(__dirname, ...)` → `import { join } from 'path'` + relative imports

4. **Dynamic Requires:**
   - Move requires from inside functions to top-level imports

**Validation Checks:**
- All `require()` calls converted to `import` statements
- All `module.exports` converted to `export default`
- Imports placed at top of file
- TypeScript typing preserved

**Prompt:**

```markdown
# Instructions for LLM: Transform Storybook Config Files from CommonJS to ESM

## Task Overview

Find all .storybook/main.ts and .storybook/main.js files in the workspace and transform
any CommonJS (CJS) configurations to ES Modules (ESM).

### Step 1: Find All Storybook Config Files

Use glob patterns to locate all Storybook main configuration files:
**/.storybook/main.js
**/.storybook/main.ts

### Step 2: Identify CommonJS vs ESM

For each file found, read its contents and determine if it uses CommonJS syntax by
checking for:

CommonJS indicators:
- `module.exports =` or `module.exports.`
- `exports.`
- `require()` function calls

ESM indicators (already correct):
- export default
- export const/export function
- import statements

### Step 3: Transform CJS to ESM

For each file identified as CommonJS, perform the following transformations:

A. Convert `module.exports`

// FROM (CJS):
module.exports = {
    stories: ['../src/**/*.stories.@(js|jsx|ts|tsx|mdx)'],
    addons: ['@storybook/addon-essentials']
};

// TO (ESM):
export default {
    stories: ['../src/**/*.stories.@(js|jsx|ts|tsx|mdx)'],
    addons: ['@storybook/addon-essentials']
};

B. Convert `require()` to import

// FROM (CJS):
const { nxViteTsPaths } = require('@nx/vite/plugins/nx-tsconfig-paths.plugin');
const { mergeConfig } = require('vite');

// TO (ESM):
import { nxViteTsPaths } from '@nx/vite/plugins/nx-tsconfig-paths.plugin';
import { mergeConfig } from 'vite';

C. Handle `path.join()` patterns

// FROM (CJS):
const path = require('path');
const rootMain = require(path.join(__dirname, '../../.storybook/main'));

// TO (ESM):
import { join } from 'path';
import rootMain from '../../.storybook/main';

D. Handle Dynamic Requires in Config Functions

// FROM (CJS):
module.exports = {
    viteFinal: async (config) => {
        const { mergeConfig } = require('vite');
        return mergeConfig(config, {});
    }
};

// TO (ESM):
import { mergeConfig } from 'vite';

export default {
    viteFinal: async (config) => {
        return mergeConfig(config, {});
    }
};

### Step 4: Validation Checks

After transformation, verify:

1. All require() calls have been converted to import statements at the top of the file
2. All module.exports have been converted to export default or named exports
3. Imports are at the top of the file (before the export)
4. The file maintains proper TypeScript typing if it's a .ts file

### Step 5: Report Results

Provide a summary of:
- Total files found
- Files that were already ESM (no changes needed)
- Files that were transformed from CJS to ESM
- List the specific files that were modified

## Important Notes

- Preserve all comments in the original files
- Maintain the same indentation and formatting style
- For TypeScript files (.ts), ensure type imports use import type when appropriate
- Test that the transformations don't break the Storybook configuration
```

---

### 5. CI Error Guidelines (Nx Cloud Integration)

**Purpose:** Instructions for AI agents to help users debug and fix CI pipeline errors when Nx Cloud is enabled. This prompt guides agents through a systematic workflow of retrieving CI pipeline execution details, analyzing error logs, and helping fix issues. These guidelines are appended to the agent configuration when Nx Cloud is detected.

**Location:** Injected in `CLAUDE.md` and agent configuration files when Nx Cloud is enabled

**Appended To:** Agent configuration prompts (see section 2)

**MCP Tools Used:**
- `nx_cloud_cipe_details` - Retrieves CI Pipeline Execution details
- `nx_cloud_fix_cipe_failure` - Gets logs for specific failed tasks

**Prompt:**

```markdown
# CI Error Guidelines

If the user wants help with fixing an error in their CI pipeline, use the following flow:

- Retrieve the list of current CI Pipeline Executions (CIPEs) using the `nx_cloud_cipe_details` tool
- If there are any errors, use the `nx_cloud_fix_cipe_failure` tool to retrieve the logs for a specific task
- Use the task logs to see what's wrong and help the user fix their problem. Use the appropriate tools if necessary
- Make sure that the problem is fixed by running the task that you passed into the `nx_cloud_fix_cipe_failure` tool
```

---

### 6. GitHub Issue Response Mode Guidelines

**Purpose:** Instructions for AI agents working on GitHub issues in the Nx repository to determine whether to plan first or implement immediately based on user intent. This helps agents provide appropriate responses based on action verbs in issue requests.

**Location:** `CLAUDE.md` - Repository-specific agent configuration

**Used By:** AI agents (Claude Code, etc.) when working on GitHub issues

**Decision Logic:**
- **Plan-First Mode:** Triggered by verbs like "analyze", "investigate", "assess", "review", "examine", "plan"
- **Immediate Implementation Mode:** Triggered by verbs like "fix", "implement", "solve", "build", "create", "update", "add"

**Prompt:**

```markdown
## GitHub Issue Response Mode

When responding to GitHub issues, determine your approach based on how the request is phrased:

### Plan-First Mode (Default)

Use this approach when users ask you to:

- "analyze", "investigate", "assess", "review", "examine", or "plan"
- Or when the request is ambiguous

In this mode:

1. Provide a detailed analysis of the issue
2. Create a comprehensive implementation plan
3. Break down the solution into clear steps
4. Then please post the plan as a comment on the issue

### Immediate Implementation Mode

Use this approach when users ask you to:

- "fix", "implement", "solve", "build", "create", "update", or "add"
- Or when they explicitly request immediate action

In this mode:

1. Analyze the issue quickly
2. Implement the complete solution immediately
3. Make all necessary code changes. Please make multiple commits so that the changes are easier to review.
4. Run appropriate tests and validation
5. If the tests, are not passing, please fix the issues and continue doing this up to 3 more times until the tests pass
6. Once the tests pass, push a branch and then suggest opening a PR which has a description of the changes made, and that it make sure that it explicitly says "Fixes #ISSUE_NUMBER" to automatically close the issue when the PR is merged.
```

---

## Interactive CLI Prompts

### 7. AI Agent Selection Prompt

**Purpose:** Interactive CLI prompt displayed during workspace initialization (`nx init`) and workspace creation (`create-nx-workspace`) to ask users which AI agents they want to configure. Uses multi-select interface to allow selecting multiple agents.

**Location:**
- `packages/nx/src/command-line/init/ai-agent-prompts.ts`
- `packages/create-nx-workspace/src/internal-utils/prompts.ts`

**Display Context:**
- Shown only in interactive mode (not in CI)
- Skipped if agents are already specified via CLI args

**Available Agents:**
- Claude Code (Claude)
- GitHub Copilot
- Cursor
- Google Gemini
- OpenAI Codex

**Prompt Text:**

```
Which AI agents, if any, would you like to set up?

[Multi-select list of agents]

Multiple selections possible. <Space> to select. <Enter> to confirm.
```

**Implementation:**

```typescript
async function aiAgentsPrompt(): Promise<Agent[]> {
  const promptConfig = {
    name: 'agents',
    message: 'Which AI agents, if any, would you like to set up?',
    type: 'multiselect',
    choices: supportedAgents.map((a) => ({
      name: a,
      message: agentDisplayMap[a],
    })),
    footer: () =>
      chalk.dim(
        'Multiple selections possible. <Space> to select. <Enter> to confirm.'
      ),
  };
  return (await prompt<{ agents: Agent[] }>([promptConfig])).agents;
}
```

**Follow-up Action:** Selected agents trigger the `@nx/nx:set-up-ai-agents` generator which creates appropriate configuration files and injects agent rules.

---
