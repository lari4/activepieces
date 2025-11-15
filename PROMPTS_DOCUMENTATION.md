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

### 9. Anthropic - Ask Claude

**Location:** `packages/pieces/community/claude/src/lib/actions/send-prompt.ts:45`

**Purpose:** Direct integration with Anthropic's Claude models for conversational AI and vision tasks. Supports extended thinking mode for complex reasoning, multi-modal inputs (text + images), and role-based conversations.

**Default System Prompt:**
```
You're a helpful assistant.
```

**Features:**
- Support for all Claude models (3, 3.5, 4, 4.5 Haiku/Sonnet/Opus)
- Extended Thinking Mode using Claude 3.7 Sonnet for complex reasoning tasks
- Multi-modal support (text + images via base64)
- Role-based conversations (user/assistant only)
- Automatic retry with exponential backoff on rate limits
- Vision capabilities with MIME type detection

**Extended Thinking Mode:**
When enabled, uses Claude 3.7 Sonnet with a configurable "budget_tokens" parameter (default: 1024) for internal reasoning.

**Message Structure:**
```typescript
messages: [
  {
    role: 'user',
    content: [
      { type: 'text', text: prompt },
      // Optional image
      {
        type: 'image',
        source: {
          type: 'base64',
          media_type: 'image/jpeg',
          data: base64Data
        }
      }
    ]
  },
  ...roles  // Additional user/assistant messages
]
```

**Parameters:**
- `model` - Claude model selection (Dropdown, Default: "claude-3-haiku-20240307")
- `systemPrompt` - System instructions (Long Text, Default: "You're a helpful assistant.")
- `prompt` - User question (Long Text, Required)
- `image` - Optional image for vision analysis (File, Optional)
- `temperature` - Randomness 0-1 (Number, Default: 0.5)
- `maxTokens` - Maximum response tokens (Number, Default: 1000)
- `roles` - Additional user/assistant messages (JSON Array, Default: [])
- `thinkingMode` - Enable extended reasoning (Checkbox, Default: false)
- `budgetTokens` - Thinking mode token budget (Number, Default: 1024, only if thinking mode enabled)

**Usage Example (Standard):**
```typescript
{
  model: "claude-sonnet-4-5-20250929",
  systemPrompt: "You are an expert data analyst.",
  prompt: "Analyze the sales trends in this dataset.",
  temperature: 0.3,
  maxTokens: 2000
}
```

**Usage Example (Thinking Mode):**
```typescript
{
  model: "claude-3-7-sonnet-latest",
  systemPrompt: "You are a mathematics problem solver.",
  prompt: "Solve this complex calculus problem step by step: ...",
  thinkingMode: true,
  budgetTokens: 2048,  // More tokens for complex reasoning
  maxTokens: 3000
}
```

**Usage Example (Vision):**
```typescript
{
  model: "claude-sonnet-4-5-20250929",
  prompt: "What's in this medical scan image?",
  image: xrayImage,
  systemPrompt: "You are a medical imaging assistant.",
  temperature: 0.2
}
```

---

### 10. Perplexity AI - Ask AI

**Location:** `packages/pieces/community/perplexity-ai/src/lib/actions/create-chat-completion.action.ts:92-94`

**Purpose:** Search-powered AI responses using Perplexity's Sonar models. Provides AI-generated answers with citations from web sources, making it ideal for research and fact-checking tasks.

**Default System Prompt:**
```json
[
  { "role": "system", "content": "You are a helpful assistant." }
]
```

**Features:**
- Search-grounded responses with citations
- Sonar and Sonar Reasoning models
- Role-based conversation system (system/user/assistant)
- Returns both answer text and citations array
- Full parameter control for response tuning

**Available Models:**
- `sonar-reasoning-pro` - Advanced reasoning with search
- `sonar-reasoning` - Standard reasoning with search
- `sonar-pro` - Premium search-powered model (default)
- `sonar` - Standard search-powered model

**Message Structure:**
```typescript
messages: [
  ...roles,  // System/user/assistant role definitions
  { role: 'user', content: prompt }
]
```

**Parameters:**
- `model` - Sonar model selection (Dropdown, Default: "sonar-pro")
- `prompt` - User question (Long Text, Required)
- `temperature` - Randomness 0-2 (Number, Default: 0.2)
- `max_tokens` - Maximum response tokens (Number, Optional)
- `top_p` - Nucleus sampling 0-1 (Number, Default: 0.9)
- `presence_penalty` - Topic diversity -2.0 to 2.0 (Number, Default: 0)
- `frequency_penalty` - Repetition penalty >0 (Number, Default: 1.0)
- `roles` - System/user/assistant instructions (JSON Array, Default: see above)

**Response Format:**
```typescript
{
  result: "AI-generated answer text",
  citations: [
    "https://source1.com",
    "https://source2.com",
    ...
  ]
}
```

**Usage Example:**
```typescript
{
  model: "sonar-reasoning-pro",
  prompt: "What are the latest developments in quantum computing?",
  temperature: 0.3,
  roles: [
    { role: "system", content: "You are a technology research assistant. Provide accurate, well-cited information." }
  ]
}
// Returns: { result: "...", citations: ["https://..."] }
```

---

### 11. LocalAI - Self-Hosted AI

**Location:** `packages/pieces/community/localai/src/lib/actions/send-prompt.ts:115-117`

**Purpose:** Integration with self-hosted LocalAI instances for privacy-focused AI deployments. Provides OpenAI-compatible API interface for local LLM models.

**Default System Prompt:**
```json
[
  { "role": "system", "content": "You are a helpful assistant." }
]
```

**Features:**
- Self-hosted, privacy-focused AI
- OpenAI-compatible API
- Dynamic model selection from LocalAI instance
- Role-based conversation system
- Automatic retry with exponential backoff
- Full parameter control

**Message Structure:**
```typescript
messages: [
  ...roles,              // System/user/assistant role definitions
  { role: 'user', content: prompt }
]
```

**Parameters:**
- `model` - Available local model (Dropdown, Default: "gpt-3.5-turbo")
- `prompt` - User question (Long Text, Required)
- `temperature` - Randomness (Number, Default: 0.9)
- `maxTokens` - Maximum response tokens (Number, Default: 2048)
- `topP` - Nucleus sampling (Number, Default: 1)
- `frequencyPenalty` - Repetition penalty -2.0 to 2.0 (Number, Default: 0)
- `presencePenalty` - Topic diversity -2.0 to 2.0 (Number, Default: 0.6)
- `roles` - System/user/assistant instructions (JSON Array, Default: see above)

**Configuration Required:**
- LocalAI Base URL (configured in auth)
- API Key (optional, configured in auth)

**Usage Example:**
```typescript
{
  model: "llama-2-7b-chat",  // Your locally deployed model
  prompt: "Explain machine learning in simple terms",
  temperature: 0.7,
  roles: [
    { role: "system", content: "You are an educational tutor." }
  ]
}
```

---

## Specialized AI Services

### 12. Eden AI - Multi-Provider Text Generation

**Location:** `packages/pieces/community/eden-ai/src/lib/actions/generate-text.ts`

**Purpose:** Unified interface for accessing multiple AI providers (OpenAI, Anthropic, Google, Meta, Mistral, Cohere, XAI, Amazon, Microsoft, DeepSeek, Groq) through Eden AI's platform. Provides provider fallbacks and consistent API interface across different AI services.

**Prompt Structure:**
No fixed default prompt - users provide custom system and user prompts. The action constructs messages in this format:

```typescript
messages: [
  // Optional system message
  {
    role: 'system',
    content: [{ type: 'text', text: system_prompt }]
  },
  // User message with optional image
  {
    role: 'user',
    content: [
      { type: 'text', text: prompt },
      // Optional for vision models
      {
        type: 'image_url',
        image_url: { url: image_url }
      }
    ]
  }
]
```

**Supported Providers & Default Models:**
- **OpenAI**: `gpt-4o`
- **Anthropic Claude**: `claude-3-sonnet-latest`
- **Google Gemini**: `gemini-2.0-flash`
- **Meta Llama**: `llama-3.1-70b-instruct`
- **Mistral**: `mistral-large-latest`
- **Cohere**: `command-r-plus`
- **XAI Grok**: `grok-2-latest`
- **Amazon Nova**: `nova-pro-v1:0`
- **Microsoft**: `gpt-4o`
- **DeepSeek**: `deepseek-chat`
- **Groq**: `llama-3.1-70b-versatile`

**Features:**
- Multi-provider support with automatic fallbacks
- Vision capabilities (image + text prompts)
- Consistent API across providers
- Reasoning effort control (low/medium/high)
- Custom model selection per provider

**Parameters:**
- `provider` - AI provider (Dropdown, Required)
- `prompt` - Main user prompt (Long Text, Required)
- `system_prompt` - Behavior/context instructions (Long Text, Optional)
- `model` - Specific model override (Short Text, Optional, auto-selected if empty)
- `temperature` - Creativity 0-2 (Number, Default: 0.7)
- `max_completion_tokens` - Max response length (Number, Default: 1000)
- `reasoning_effort` - Depth of reasoning: low/medium/high (Dropdown, Optional)
- `fallback_providers` - Alternative providers (Multi-Select, Optional)
- `include_image` - Enable vision mode (Checkbox, Default: false)
- `image_url` - Image URL for analysis (Short Text, Optional)

**Usage Example:**
```typescript
{
  provider: "anthropic",
  prompt: "Analyze this customer feedback and extract sentiment",
  system_prompt: "You are a customer service analyst. Be objective and thorough.",
  model: "claude-3-opus-latest",
  temperature: 0.3,
  fallback_providers: ["openai", "google"]
}
```

**Usage Example (Vision):**
```typescript
{
  provider: "openai",
  prompt: "What products are visible in this shelf image?",
  model: "gpt-4o",
  include_image: true,
  image_url: "https://example.com/shelf.jpg",
  temperature: 0.2
}
```

---

## TextCortex AI Services

TextCortex provides specialized AI writing tools with multi-language support, formality control, and various creative functions.

### 13. TextCortex - Send Prompt (Text Completion)

**Location:** `packages/pieces/community/textcortex-ai/src/lib/actions/send-prompt.ts`

**Purpose:** General-purpose text completion using TextCortex AI. Given a starting text, the AI continues and completes it. Supports multiple models, formality levels, and language translation.

**Prompt Structure:**
No fixed system prompt - users provide starting text that AI completes:

```typescript
{
  text: "The benefits of renewable energy are"
}
// AI completes: "...numerous and far-reaching. They include reduced carbon emissions..."
```

**Default Configuration:**
- Model: `gemini-2-0-flash`
- Formality: `default`
- Source Language: `en` (English)
- Target Language: `en` (English)
- Max Tokens: `2048`
- Number of Outputs: `1`

**Features:**
- Text completion from partial input
- Multi-language support (source and target)
- Formality control (default, formal, informal)
- Multiple model options
- Generate multiple variations (1-5 outputs)
- Automatic language detection

**Parameters:**
- `text` - Starting text to complete (Long Text, Required)
- `model` - AI model selection (Dropdown, Default: "gemini-2-0-flash")
- `formality` - Output formality level (Dropdown, Default: "default")
- `source_lang` - Input text language (Dropdown, Default: "en", supports auto-detect)
- `target_lang` - Output language (Dropdown, Default: "en")
- `max_tokens` - Maximum completion length (Number, Default: 2048, Range: 1-4096)
- `temperature` - Creativity control (Number, Range: 0-2)
- `n` - Number of completions (Number, Default: 1, Range: 1-5)

**Usage Example:**
```typescript
{
  text: "Dear valued customer, we are writing to inform you",
  model: "gemini-2-0-flash",
  formality: "formal",
  target_lang: "en",
  max_tokens: 500,
  temperature: 0.5,
  n: 3  // Generate 3 variations
}
```

---

### 14. TextCortex - Create Summary

**Location:** `packages/pieces/community/textcortex-ai/src/lib/actions/create-summary.ts`

**Purpose:** Condense long text or uploaded files into concise summaries. Supports two modes: default summarization and embeddings-based summarization for better semantic understanding.

**Prompt Structure:**
No explicit prompt - summarization is controlled by mode and parameters:

```typescript
{
  text: "Long article text...",
  mode: "default" | "embeddings"
}
```

**Default Configuration:**
- Mode: `default`
- Model: `gemini-2-0-flash`
- Formality: `default`
- Source Language: `en`
- Target Language: `en`
- Max Tokens: `2048`
- Number of Outputs: `1`

**Summarization Modes:**
1. **Default Mode** - Standard extractive/abstractive summarization
2. **Embeddings Mode** - Uses semantic embeddings for better context understanding

**Features:**
- Text or file-based summarization
- Multi-language input and output
- Formality control for summary style
- Multiple summary variations
- Credit usage tracking

**Parameters:**
- `text` - Text to summarize (Long Text, Optional)
- `file_id` - Alternative: File ID to summarize (Short Text, Optional)
- `mode` - Summarization approach (Dropdown, Default: "default", Options: default/embeddings)
- `model` - AI model (Dropdown, Default: "gemini-2-0-flash")
- `formality` - Summary style (Dropdown, Default: "default")
- `source_lang` - Input language (Dropdown, Default: "en")
- `target_lang` - Summary language (Dropdown, Default: "en")
- `max_tokens` - Maximum summary length (Number, Default: 2048)
- `temperature` - Creativity (Number, Range: 0-2)
- `n` - Number of summaries (Number, Default: 1, Range: 1-5)

**Usage Example (Text):**
```typescript
{
  text: "Very long research paper content...",
  mode: "embeddings",  // Better semantic understanding
  formality: "formal",
  source_lang: "en",
  target_lang: "es",  // Summarize in Spanish
  max_tokens: 500,
  temperature: 0.3
}
```

**Usage Example (File):**
```typescript
{
  file_id: "file_abc123",  // Previously uploaded file
  mode: "default",
  max_tokens: 1000,
  n: 2  // Generate 2 summary variations
}
```

---

## Summary of All AI Prompts

This application contains **14 major AI prompt categories** across different integration levels:

### Agent System (1 prompt)
1. **Agent Core System Prompt** - Autonomous task completion with tool usage directives

### Universal AI Pieces (4 prompts)
2. **Ask AI** - General conversational AI (no fixed prompt)
3. **Summarize Text** - "Summarize the following text in a clear and concise manner..."
4. **Classify Text** - "As a text classifier, your task is to assign one of the following categories..."
5. **Extract Structured Data** - "Extract the following data from the provided data."

### AI Provider Integrations (6 prompts)
6. **OpenAI ChatGPT** - "You are a helpful assistant."
7. **OpenAI Vision** - "You are a helpful assistant."
8. **Anthropic Claude** - "You're a helpful assistant."
9. **Perplexity AI** - "You are a helpful assistant."
10. **LocalAI** - "You are a helpful assistant."

### Specialized AI Services (3 prompt systems)
11. **Eden AI** - Multi-provider (custom prompts, no defaults)
12. **TextCortex Send Prompt** - Text completion (no fixed prompt)
13. **TextCortex Create Summary** - Summarization (mode-based, no explicit prompt)

### Common Patterns

**Most Common Default System Prompt:**
```
"You are a helpful assistant."
```

Used by: OpenAI ChatGPT, OpenAI Vision, Perplexity AI, LocalAI

**Anthropic Variant:**
```
"You're a helpful assistant."
```

**Agent-Specific Prompt Pattern:**
```
"You are an autonomous assistant designed to efficiently accomplish the user's goal."
+ Core directives + Current date/time
```

**Task-Specific Prompts:**
- Classification: Instructs AI to respond with only category names
- Summarization: Guides toward concise, informative summaries
- Extraction: Directs AI to extract specific structured data

---

