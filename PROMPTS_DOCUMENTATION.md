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

## Universal AI Pieces

These are provider-agnostic AI actions that work with multiple AI providers (OpenAI, Anthropic, Google, etc.).

### 3. Ask AI (Conversational AI)

**Location:** `packages/pieces/community/text-ai/src/lib/actions/ask-ai.ts`

**Purpose:** A general-purpose conversational AI action that maintains conversation history and can perform web searches. This action supports multi-turn conversations using a conversation key to store message history. It can be configured for different creativity levels and supports web search capabilities for real-time information retrieval.

**Features:**
- Multi-turn conversation memory using storage keys
- Configurable creativity (temperature) from 0-100
- Web search integration with source citations
- Token limit control (default: 2000)
- Provider-agnostic (works with OpenAI, Anthropic, Google, etc.)

**Prompt Structure:**
This action doesn't use a fixed system prompt. Instead, it sends the user's prompt directly with conversation history:

```typescript
messages: [
  ...(conversationHistory ?? []),
  {
    role: 'user',
    content: context.propsValue.prompt,
  },
]
```

**Parameters:**
- `prompt` - User's question or instruction (Long Text, Required)
- `conversationKey` - Key for maintaining conversation context (Short Text, Optional)
- `creativity` - Temperature setting 0-100, higher = more creative (Number, Default: 100)
- `maxOutputTokens` - Maximum response length (Number, Default: 2000)
- `webSearch` - Enable web search tool (Boolean)
- `webSearchOptions` - Web search configuration (max uses, include sources)

**Usage Example:**
```typescript
// First message in conversation
{ prompt: "What is quantum computing?", conversationKey: "user-123-session" }

// Follow-up in same conversation
{ prompt: "How is it different from classical computing?", conversationKey: "user-123-session" }
```

---

### 4. Summarize Text

**Location:** `packages/pieces/community/text-ai/src/lib/actions/summarize-text.ts:18-22`

**Purpose:** Condense long text into concise summaries while preserving key information. Uses a customizable prompt to guide the summarization style and approach.

**Default Prompt:**
```
Summarize the following text in a clear and concise manner, capturing the key points and main ideas while keeping the summary brief and informative.
```

**Full Message Template:**
```
{prompt} Summarize the following text : {text}
```

**Parameters:**
- `text` - The text to summarize (Long Text, Required)
- `prompt` - Customizable summarization instruction (Short Text, Default: see above)
- `maxOutputTokens` - Maximum summary length (Number, Default: 2000)

**Provider Options:**
- For OpenAI: Sets `reasoning_effort: 'minimal'` for faster responses
- Temperature: Fixed at 1.0 for natural language generation

**Usage Example:**
```typescript
{
  text: "Long article about climate change...",
  prompt: "Create a 3-sentence executive summary focusing on actionable insights:"
}
```

---

### 5. Classify Text

**Location:** `packages/pieces/community/utility-ai/src/lib/actions/classify-text.ts:49-52`

**Purpose:** Automatically categorize text into predefined categories using AI classification. The AI analyzes the text content and assigns the most appropriate category from the provided list.

**Prompt Template:**
```
As a text classifier, your task is to assign one of the following categories to the provided text: {categories}. Please respond with only the selected category as a single word, and nothing else.
Text to classify: "{text}"
```

**Variables:**
- `{categories}` - Comma-separated list of category options
- `{text}` - The text content to classify

**Validation:**
The action validates that the AI's response matches one of the provided categories. If not, it throws an error: "Unable to classify the text into the provided categories."

**Parameters:**
- `text` - Text to classify (Long Text, Required)
- `categories` - Array of category options (Array, Required)

**Usage Example:**
```typescript
{
  text: "I'm very unhappy with the slow delivery and damaged packaging.",
  categories: ["Positive", "Negative", "Neutral"]
}
// Returns: "Negative"
```

---

### 6. Extract Structured Data

**Location:** `packages/pieces/community/utility-ai/src/lib/actions/extract-structured-data.ts:35`

**Purpose:** Extract structured information from unstructured text, images, or PDF documents using AI vision and language models. Supports both simple field-based extraction and advanced JSON Schema definitions.

**Default Prompt:**
```
Extract the following data from the provided data.
```

**Message Structure:**
When text is provided:
```
{prompt}

Text to analyze:
{text}
```

When files (images/PDFs) are provided, they are attached as multimodal content alongside the prompt.

**Tool Description:**
The AI is given a tool called `extractData` with description:
```
Extract structured data from the provided content
```

**Modes:**

1. **Simple Mode** - Define fields with name, description, type (string/number/boolean), and required flag
2. **Advanced Mode** - Use full JSON Schema for complex data structures

**Features:**
- Multi-modal support (text, images, PDFs)
- Tool-based extraction with `toolChoice: 'required'` forcing structured output
- JSON Schema validation using AJV
- Field name validation (alphanumeric, underscore, dot, hyphen only)

**Parameters:**
- `text` - Text to extract from (Long Text, Optional)
- `files` - Array of images or PDFs (File Array, Optional)
- `prompt` - Guide prompt (Long Text, Default: "Extract the following data from the provided data.")
- `mode` - Simple or Advanced schema type (Dropdown, Default: "simple")
- `schama` - Field definitions or JSON Schema (Dynamic)
- `maxOutputTokens` - Maximum response tokens (Number, Default: 2000)

**Usage Example (Simple):**
```typescript
{
  text: "John Doe, age 35, works at Acme Corp. Email: john@acme.com",
  prompt: "Extract contact information:",
  mode: "simple",
  schema: {
    fields: [
      { name: "name", type: "string", isRequired: true, description: "Full name" },
      { name: "age", type: "number", isRequired: false, description: "Age in years" },
      { name: "company", type: "string", isRequired: false, description: "Employer" },
      { name: "email", type: "string", isRequired: true, description: "Email address" }
    ]
  }
}
// Returns: { name: "John Doe", age: 35, company: "Acme Corp", email: "john@acme.com" }
```

**Usage Example (Advanced with JSON Schema):**
```typescript
{
  files: [{ file: invoicePDF }],
  mode: "advanced",
  schema: {
    fields: {
      type: "object",
      properties: {
        invoiceNumber: { type: "string" },
        totalAmount: { type: "number" },
        items: {
          type: "array",
          items: {
            type: "object",
            properties: {
              description: { type: "string" },
              quantity: { type: "number" },
              price: { type: "number" }
            }
          }
        }
      },
      required: ["invoiceNumber", "totalAmount"]
    }
  }
}
```

---

## AI Provider Integrations

These are provider-specific integrations for commercial AI services.

### 7. OpenAI - Ask ChatGPT

**Location:** `packages/pieces/community/openai/src/lib/actions/send-prompt.ts:113-115`

**Purpose:** Direct integration with OpenAI's ChatGPT models for conversational AI. Supports conversation memory, role-based instructions, and fine-tuned control over response generation parameters.

**Default System Prompt:**
```json
[
  { "role": "system", "content": "You are a helpful assistant." }
]
```

**Features:**
- Multi-turn conversation with persistent memory (via memory key)
- Automatic token management and history trimming (90% of 32K token limit)
- Role-based message system (system/user/assistant)
- Full parameter control (temperature, top_p, frequency_penalty, presence_penalty)
- Dynamic model selection from user's available models

**Message Structure:**
```typescript
messages: [
  ...roles,              // System/user/assistant role definitions
  ...messageHistory      // Previous conversation if memory key is set
]
```

**Parameters:**
- `model` - GPT model selection (Dropdown, Default: "gpt-3.5-turbo")
- `prompt` - User question (Long Text, Required)
- `temperature` - Randomness 0-1 (Number, Default: 1)
- `maxTokens` - Maximum completion tokens (Number, Default: 2048)
- `topP` - Nucleus sampling parameter (Number, Default: 1)
- `frequencyPenalty` - Repetition penalty -2.0 to 2.0 (Number, Default: 0)
- `presencePenalty` - Topic diversity penalty -2.0 to 2.0 (Number, Optional)
- `memoryKey` - Conversation persistence key (Short Text, Optional, Max: 128 chars)
- `roles` - System/user/assistant instructions (JSON Array, Default: see above)

**Memory Management:**
- Stores conversation history in project scope
- Automatically reduces context size when approaching token limits
- Preserves conversation flow across multiple runs

**Usage Example:**
```typescript
{
  model: "gpt-4o",
  prompt: "Explain quantum entanglement",
  roles: [
    { role: "system", content: "You are a physics professor." },
    { role: "user", content: "I'm a beginner in physics." }
  ],
  memoryKey: "physics-tutorial-session",
  temperature: 0.7,
  maxTokens: 1500
}
```

---

### 8. OpenAI - Vision Prompt

**Location:** `packages/pieces/community/openai/src/lib/actions/vision-prompt.ts:91-93`

**Purpose:** Analyze images using OpenAI's GPT-4o vision capabilities. Allows users to ask questions about image content, extract information from visual data, or describe what's in an image.

**Default System Prompt:**
```json
[
  { "role": "system", "content": "You are a helpful assistant." }
]
```

**Features:**
- GPT-4o model (fixed) for vision analysis
- Multi-modal message with text + image
- Configurable detail level (low/high/auto) for image processing
- Base64 image encoding support
- Full parameter control like text completions

**Message Structure:**
```typescript
messages: [
  ...roles,
  {
    role: 'user',
    content: [
      { type: 'text', text: prompt },
      { type: 'image_url', image_url: { url: 'data:image/...;base64,...' } }
    ]
  }
]
```

**Parameters:**
- `image` - Image file to analyze (File, Required)
- `prompt` - Question about the image (Long Text, Required)
- `detail` - Image processing detail level (Dropdown, Default: "auto", Options: low/high/auto)
- `temperature` - Randomness 0-1 (Number, Default: 0.9)
- `maxTokens` - Maximum response tokens (Number, Default: 2048)
- `topP` - Nucleus sampling (Number, Default: 1)
- `frequencyPenalty` - Repetition penalty (Number, Default: 0)
- `presencePenalty` - Topic diversity penalty (Number, Default: 0.6)
- `roles` - System instructions (JSON Array, Default: see above)

**Usage Example:**
```typescript
{
  image: uploadedImage,
  prompt: "What objects are visible in this image? List them with their approximate locations.",
  detail: "high",
  temperature: 0.5,
  roles: [
    { role: "system", content: "You are an image analysis expert. Provide detailed, structured descriptions." }
  ]
}
```

---

