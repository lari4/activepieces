# AI Prompts Documentation

This document contains all AI prompts used in the Activepieces application, organized by category with detailed descriptions of their purpose and usage.

---

## Table of Contents

1. [Agent System Prompts](#agent-system-prompts)
2. [Universal AI Pieces](#universal-ai-pieces)
3. [AI Provider Integrations](#ai-provider-integrations)
4. [Specialized AI Services](#specialized-ai-services)

---

## Agent System Prompts

### 1. Agent Core System Prompt

**Location:** `packages/pieces/community/agent/src/lib/common.ts:173-192`

**Purpose:** This is the foundational system prompt for the autonomous AI agent. It defines the core behavior, directives, and operational rules that the agent must follow when executing tasks. The agent uses this prompt to understand how to interact with tools, when to mark tasks as complete, and how to communicate progress.

**Used In:** `runAgent` action - the main entry point for running autonomous agent tasks

**Key Features:**
- Defines agent autonomy and goal-oriented behavior
- Establishes tool usage patterns and completion criteria
- Includes current date/time context for time-based queries
- Appends custom user system prompt for task-specific instructions

**Prompt:**
```
You are an autonomous assistant designed to efficiently accomplish the user's goal.

### Core Directives:
1. **Always** perform the requested task before calling the `mark as complete` tool.
2. **Always** call the `mark as complete` tool after finishing a task or answering a question — even if it fails.
   - Include the output, result, or failure reason in the call.
3. After using **any tool** (except `mark as complete`), you must **immediately provide a one-line explanation** of what you did with that tool.
   - If the tool call fails, briefly explain **why** it failed.
4. Be concise, factual, and action-driven. Avoid unnecessary explanations.

**Current Date:** ${new Date().toISOString()}
(Use this to interpret time-based queries like "this week" or "due tomorrow.")

---

${systemPrompt}
```

**Variables:**
- `${new Date().toISOString()}` - Current timestamp in ISO format (e.g., "2025-11-14T10:30:00.000Z")
- `${systemPrompt}` - User-provided custom system prompt for task-specific context

**Example Usage:**
```typescript
const systemPrompt = agentCommon.constructSystemPrompt("Extract all email addresses from the database and send a summary report.");
// The final prompt combines the core directives with the user's custom prompt
```

---

### 2. Mark as Complete Tool

**Location:** `packages/pieces/community/agent/src/lib/common.ts:53-59`

**Purpose:** This tool allows the agent to signal task completion. It's a required built-in tool that the agent must call after finishing any task. The tool can optionally accept structured output based on user-defined output fields.

**Description:**
```
Mark the todo as complete
```

**Input Schema:** Dynamic - generated based on user-defined output fields (text, number, boolean)

**Execution:** Returns "Marked as Complete" message

---

