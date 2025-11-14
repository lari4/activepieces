# AI Agent Pipelines Documentation

This document describes all possible agent workflows and data pipelines in the Activepieces AI system. It shows how prompts flow through the system, which components call which, and how data is transformed between stages.

---

## Table of Contents

1. [Agent System Pipeline](#agent-system-pipeline)
2. [Universal AI Pieces Pipelines](#universal-ai-pieces-pipelines)
3. [Provider Integration Pipelines](#provider-integration-pipelines)
4. [Multi-Provider Pipeline (Eden AI)](#multi-provider-pipeline-eden-ai)
5. [Web Search Enhanced Pipeline](#web-search-enhanced-pipeline)

---

## Pipeline 1: Agent System Pipeline

### Overview
The autonomous agent pipeline for task completion with tool usage and MCP (Model Context Protocol) integration.

### Flow Diagram

```
┌─────────────────────────────────────────────────────────────────┐
│                        USER INPUT                                │
│  - prompt: "Extract all PDFs from folder and summarize them"    │
│  - aiModel: "anthropic-claude-sonnet-4-5-20250929"              │
│  - agentTools: [MCP tools, flows, pieces]                       │
│  - structuredOutput: [optional output schema]                   │
│  - maxSteps: 20                                                  │
└───────────────────────────┬─────────────────────────────────────┘
                            │
                            ▼
┌─────────────────────────────────────────────────────────────────┐
│              STEP 1: System Prompt Construction                  │
│  Location: packages/pieces/community/agent/src/lib/common.ts    │
│                                                                  │
│  agentCommon.constructSystemPrompt(userPrompt)                  │
│                                                                  │
│  Output:                                                         │
│  ┌───────────────────────────────────────────────────────┐     │
│  │ You are an autonomous assistant designed to           │     │
│  │ efficiently accomplish the user's goal.               │     │
│  │                                                         │     │
│  │ Core Directives:                                       │     │
│  │ 1. Always perform task before calling 'mark as        │     │
│  │    complete' tool                                      │     │
│  │ 2. Always call 'mark as complete' after finishing     │     │
│  │ 3. After using any tool, provide one-line explanation │     │
│  │ 4. Be concise, factual, and action-driven            │     │
│  │                                                         │     │
│  │ Current Date: 2025-11-14T10:30:00.000Z                │     │
│  │                                                         │     │
│  │ [USER'S CUSTOM PROMPT APPENDED HERE]                   │     │
│  └───────────────────────────────────────────────────────┘     │
└───────────────────────────┬─────────────────────────────────────┘
                            │
                            ▼
┌─────────────────────────────────────────────────────────────────┐
│              STEP 2: Tool Initialization                         │
│  Location: packages/pieces/community/agent/src/lib/common.ts    │
│                                                                  │
│  agentCommon.agentTools({                                       │
│    outputFields,                                                │
│    tools: agentTools,                                           │
│    apiUrl, token, flowId, flowVersionId, stepName              │
│  })                                                             │
│                                                                  │
│  Output:                                                         │
│  {                                                               │
│    tools: {                                                      │
│      "mark_as_complete": [Built-in tool],                      │
│      ...mcpTools  // From MCP server                            │
│    }                                                             │
│  }                                                               │
└───────────────────────────┬─────────────────────────────────────┘
                            │
                            ▼
┌─────────────────────────────────────────────────────────────────┐
│              STEP 3: MCP Server Connection                       │
│  Location: packages/server/api/src/app/mcp/mcp-server/         │
│                                                                  │
│  MCP Transport: StreamableHTTPClientTransport                   │
│  URL: {apiUrl}v1/flows/{flowId}/versions/{flowVersionId}/      │
│       steps/{stepName}/mcp                                      │
│                                                                  │
│  Creates experimental_createMCPClient with:                     │
│  - Authorization header (Bearer token)                          │
│  - Retrieves available tools from MCP registry                  │
│                                                                  │
│  Tool Types:                                                     │
│  1. Built-in: mark_as_complete                                  │
│  2. MCP Pieces: Activepieces piece actions                     │
│  3. MCP Flows: Other flows as tools                            │
└───────────────────────────┬─────────────────────────────────────┘
                            │
                            ▼
┌─────────────────────────────────────────────────────────────────┐
│              STEP 4: AI Model Selection & Configuration         │
│  Location: packages/pieces/community/agent/src/lib/common.ts    │
│                                                                  │
│  Model Registry: SUPPORTED_AI_PROVIDERS                         │
│  - OpenAI (gpt-4o, gpt-4o-mini, gpt-3.5-turbo, o1-preview,     │
│             o1-mini, o1, gpt-4-turbo)                           │
│  - Anthropic (claude-sonnet-4-5, claude-haiku-4-5,             │
│               claude-opus-4-1, claude-sonnet-4, etc.)           │
│  - Google (gemini-2.0-flash-exp, gemini-1.5-flash,             │
│             gemini-1.5-pro)                                     │
│                                                                  │
│  createModel({                                                   │
│    model: selectedModel,                                        │
│    token: engineToken,                                          │
│    baseURL: {apiUrl}v1/ai-providers/proxy/{provider},         │
│    metadata: {                                                   │
│      feature: AIUsageFeature.MCP,                              │
│      mcpid: `flow:${flowId}`                                    │
│    }                                                             │
│  })                                                             │
└───────────────────────────┬─────────────────────────────────────┘
                            │
                            ▼
┌─────────────────────────────────────────────────────────────────┐
│              STEP 5: Streaming Text Generation                   │
│  Location: packages/pieces/community/agent/src/lib/actions/     │
│            run-agent.ts                                          │
│                                                                  │
│  streamText({                                                    │
│    model: modelInstance,                                        │
│    system: systemPrompt,  // From Step 1                        │
│    prompt: userPrompt,                                          │
│    stopWhen: stepCountIs(maxSteps),  // Default: 20            │
│    tools: await agentToolInstance.tools()  // From Step 2      │
│  })                                                             │
│                                                                  │
│  Returns: fullStream (async iterator)                           │
└───────────────────────────┬─────────────────────────────────────┘
                            │
                            ▼
┌─────────────────────────────────────────────────────────────────┐
│              STEP 6: Stream Processing Loop                      │
│                                                                  │
│  for await (const chunk of fullStream) {                        │
│    ┌──────────────────────────────────────────────┐            │
│    │ Chunk Type: 'text-delta'                     │            │
│    │ - Accumulate text in currentText             │            │
│    │ - No output update yet                       │            │
│    └──────────────────────────────────────────────┘            │
│    ┌──────────────────────────────────────────────┐            │
│    │ Chunk Type: 'tool-call'                      │            │
│    │ - Save accumulated text as MARKDOWN block    │            │
│    │ - Create TOOL_CALL block with:               │            │
│    │   * toolName                                  │            │
│    │   * toolCallId                                │            │
│    │   * input parameters                          │            │
│    │   * status: IN_PROGRESS                      │            │
│    │   * startTime                                 │            │
│    │ - Update real-time output to user            │            │
│    └──────────────────────────────────────────────┘            │
│    ┌──────────────────────────────────────────────┐            │
│    │ Chunk Type: 'tool-result'                    │            │
│    │ - Find matching TOOL_CALL block by ID        │            │
│    │ - Update block with:                         │            │
│    │   * status: COMPLETED                        │            │
│    │   * output: tool execution result            │            │
│    │   * endTime                                   │            │
│    │ - Update real-time output to user            │            │
│    └──────────────────────────────────────────────┘            │
│    ┌──────────────────────────────────────────────┐            │
│    │ Chunk Type: 'error'                          │            │
│    │ - Set status: FAILED                         │            │
│    │ - Extract error message from APICallError    │            │
│    │ - Return error result                        │            │
│    └──────────────────────────────────────────────┘            │
│  }                                                               │
└───────────────────────────┬─────────────────────────────────────┘
                            │
                            ▼
┌─────────────────────────────────────────────────────────────────┐
│              STEP 7: Completion Check                            │
│                                                                  │
│  Search for 'mark_as_complete' tool call in result.steps       │
│                                                                  │
│  IF found:                                                       │
│    status = AgentTaskStatus.COMPLETED                          │
│  ELSE:                                                           │
│    status = AgentTaskStatus.FAILED                             │
│                                                                  │
│  message = Concatenated markdown from all steps                │
└───────────────────────────┬─────────────────────────────────────┘
                            │
                            ▼
┌─────────────────────────────────────────────────────────────────┐
│                        FINAL OUTPUT                              │
│                                                                  │
│  {                                                               │
│    prompt: "Original user prompt",                              │
│    steps: [                                                      │
│      {                                                           │
│        type: ContentBlockType.MARKDOWN,                         │
│        markdown: "Agent's thinking/response"                    │
│      },                                                          │
│      {                                                           │
│        type: ContentBlockType.TOOL_CALL,                        │
│        toolName: "read_pdf",                                    │
│        toolCallType: ToolCallType.PIECE,                        │
│        status: ToolCallStatus.COMPLETED,                        │
│        input: { path: "/folder/doc.pdf" },                      │
│        output: "PDF content...",                                │
│        startTime: "2025-11-14T10:30:01.000Z",                  │
│        endTime: "2025-11-14T10:30:03.000Z"                     │
│      },                                                          │
│      {                                                           │
│        type: ContentBlockType.TOOL_CALL,                        │
│        toolName: "mark_as_complete",                            │
│        toolCallType: ToolCallType.INTERNAL,                     │
│        status: ToolCallStatus.COMPLETED,                        │
│        input: { output: { summary: "..." } },                   │
│        output: "Marked as Complete"                             │
│      }                                                           │
│    ],                                                            │
│    status: AgentTaskStatus.COMPLETED,                          │
│    message: "Combined markdown text from all steps"            │
│  }                                                               │
└─────────────────────────────────────────────────────────────────┘
```

### Data Flow Summary

1. **User Input** → System prompt construction
2. **System Prompt** → Combines agent directives + user instructions
3. **Tool Setup** → Loads built-in + MCP tools
4. **Model Selection** → Creates provider-proxied AI model
5. **Streaming** → AI generates response with tool calls
6. **Tool Execution** → MCP server executes requested tools
7. **Completion** → Validates 'mark_as_complete' was called
8. **Output** → Structured result with all steps and status

### Key Prompts in Pipeline

- **Input Prompt**: User's task description
- **System Prompt**: Agent behavior directives + current date + user prompt
- **Tool Descriptions**: Automatically generated from MCP tool schemas

---

