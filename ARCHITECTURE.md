# uigen — AI Engineering Deep Dive

Live app: https://code-editor-pi-tan.vercel.app/
GitHub: https://github.com/Hritikd/Code_Editor

This document explains the full architecture, how everything works under the hood,
and how to talk about it confidently in AI engineering interviews.

---

## What is this?

An AI-powered React component generator. You describe a UI in plain English,
and GPT-4o-mini writes the React code and renders a live preview in real time.

Similar products: v0.dev (Vercel), Bolt.new (StackBlitz), Lovable.dev

---

## Full System Architecture

```
User Browser
    |
    | HTTP POST /api/chat
    v
Vercel (Next.js 15 - App Router)
    |
    | streamText() via AI SDK
    v
OpenAI API (gpt-4o-mini)
    |
    | streams back tool calls + text
    v
Vercel (executes tool calls, updates virtual FS)
    |
    | streams data back to browser
    v
User Browser (renders live preview)
    |
    | on auth/save
    v
Neon (PostgreSQL) - stores users + projects
```

---

## Key Technologies & Why They Were Chosen

| Technology | Purpose | Why |
|---|---|---|
| Next.js 15 (App Router) | Full-stack framework | Server components + API routes in one repo |
| Vercel AI SDK (`ai` package) | LLM streaming + tool use | Handles streaming protocol, tool calling, multi-step agents |
| OpenAI gpt-4o-mini | Code generation | Fast, cheap, good at structured output |
| Neon PostgreSQL | Database | Serverless-compatible (Vercel cant use SQLite) |
| Prisma | ORM | Type-safe database queries |
| Virtual File System | In-memory FS | Sandboxed code execution without a real filesystem |

---

## How Streaming Works

This is the most important concept for AI engineering interviews.

### The Problem
LLMs generate tokens one at a time. If you wait for the full response before
showing it, users stare at a blank screen for 10-30 seconds. Bad UX.

### The Solution: Server-Sent Events (SSE)
```
OpenAI generates: "import" → " React" → " from" → " react" → ";" → ...
         ↓ each token streamed immediately
Vercel pipes each token to the browser
         ↓
Browser renders each token as it arrives
```

### How it works in the code

**Server** (`src/app/api/chat/route.ts`):
```typescript
const result = streamText({
  model,
  messages,
  tools: { str_replace_editor, file_manager }
})
return result.toDataStreamResponse()  // streams back as SSE
```

`streamText()` from the AI SDK:
1. Calls OpenAI with streaming enabled
2. Receives tokens as they arrive
3. Pipes them to `toDataStreamResponse()` which formats them as SSE

**Client** (`src/lib/contexts/chat-context.tsx`):
```typescript
const { messages } = useAIChat({
  api: "/api/chat",
  onToolCall: ({ toolCall }) => handleToolCall(toolCall)
})
```

`useAIChat` from `@ai-sdk/react`:
1. Sends the POST request
2. Reads the SSE stream
3. Updates `messages` state in real time as tokens arrive
4. Triggers `onToolCall` when a tool call arrives in the stream

### Interview answer for "How does streaming work in your app?"
> "The server uses the Vercel AI SDK's streamText() which calls OpenAI with
> streaming enabled. As OpenAI generates tokens, they're immediately piped
> to the client via Server-Sent Events using toDataStreamResponse(). The
> React client uses useChat from @ai-sdk/react which reads the SSE stream
> and updates state in real time, giving the user instant feedback."

---

## How Tool Calling Works (Function Calling)

This is the core of how the AI writes files.

### The Concept
Instead of just generating text, you can give an LLM a set of "tools" (functions)
it can call. The LLM decides WHEN to call them and with WHAT arguments.

### The Tools in this app

**str_replace_editor** (`src/lib/tools/str-replace.ts`):
- `create` — create a new file with content
- `str_replace` — find and replace text in an existing file
- `view` — read a file

**file_manager** (`src/lib/tools/file-manager.ts`):
- `delete_file` — delete a file
- `rename_file` — rename a file

### The Flow
```
User: "create a podcast app"
         ↓
System prompt tells GPT: "You have a str_replace_editor tool.
Use it to create and modify files."
         ↓
GPT responds with a tool call:
{
  tool: "str_replace_editor",
  args: {
    command: "create",
    path: "/App.jsx",
    file_text: "import React..."
  }
}
         ↓
AI SDK executes the tool (calls our function)
         ↓
Our function writes to the VirtualFileSystem
         ↓
GPT sees the tool result and continues
         ↓
GPT calls str_replace_editor again for next file
         ↓ (up to 40 tool calls = maxSteps: 40)
GPT sends final text message: "I created 3 files..."
```

### Why this matters (interview talking point)
This is called an **agentic loop** or **ReAct pattern** (Reason + Act).
The LLM doesnt just answer — it takes actions, observes results, and continues.
This is the same pattern used in:
- Cursor/Copilot (edit files)
- Devin (run terminal commands)
- ChatGPT with tools (browse web, run code)

### Interview answer for "How does tool calling work?"
> "I define tools as typed functions with JSON schemas. The AI SDK sends
> these schemas to OpenAI alongside the messages. When GPT decides to use
> a tool, it returns a structured tool_call object instead of text. The
> AI SDK intercepts this, executes our function with the LLM's arguments,
> appends the result to the conversation, and sends it back to GPT. This
> loop continues until GPT stops calling tools and returns a final message.
> We cap it at 40 steps to prevent infinite loops."

---

## The Virtual File System

### Why not use real files?
- Vercel is serverless — each request gets a fresh, ephemeral container
- No persistent disk between requests
- Multiple users cant share a filesystem

### How it works (`src/lib/file-system.ts`)
```typescript
class VirtualFileSystem {
  private files: Map<string, string>  // path → content

  createFile(path, content) { ... }
  readFile(path) { ... }
  serialize() { ... }       // converts to JSON for storage
  deserializeFromNodes() { ... }  // restores from JSON
}
```

The entire filesystem lives in memory (and in the database as JSON).
When the page loads, it deserializes from the database.
When files change, they serialize back to the database.

### Interview talking point
> "We use an in-memory virtual filesystem because Vercel's serverless
> functions are stateless — there's no persistent disk. The entire file
> tree is serialized as JSON and stored in PostgreSQL. This is similar
> to how CodeSandbox and StackBlitz handle in-browser code execution."

---

## The System Prompt (`src/lib/prompts/generation.tsx`)

The system prompt is what tells GPT HOW to behave. It instructs the model to:
1. Always use str_replace_editor to create/modify files (never output raw code)
2. Create proper React components with Tailwind CSS
3. Always create an App.jsx entry point
4. Follow specific file structure conventions

### Why this matters
Prompt engineering is a core AI engineering skill. The quality of the output
depends entirely on the system prompt. Key techniques used here:
- **Constrained output**: Force the model to use tools instead of plain text
- **Few-shot examples**: Show the model the expected format
- **Role definition**: Tell the model it is a UI generation assistant

---

## Database Schema

```sql
-- Users table
User {
  id        String   (cuid)
  email     String   (unique)
  password  String   (bcrypt hashed)
  projects  Project[]
}

-- Projects table
Project {
  id        String   (cuid)
  name      String
  userId    String?  (nullable = anonymous projects)
  messages  String   (JSON array of chat messages)
  data      String   (JSON of virtual filesystem)
}
```

### Key design decision: messages + data as JSON strings
Instead of separate tables for messages and files, they are stored as
serialized JSON in the Project table. This is a pragmatic choice:
- Simpler schema
- No complex joins
- Easy to serialize/deserialize the entire project state
- Trade-off: cant query individual messages, but not needed here

---

## Authentication

Uses custom JWT auth (`src/lib/auth.ts`):
- Passwords hashed with bcrypt
- JWT tokens stored in HTTP-only cookies (not localStorage — XSS protection)
- jose library for JWT signing/verification

### Why not NextAuth/Clerk?
The auth is intentionally simple and custom-built. Good for learning.
In production at scale, you would use Clerk or Auth.js.

---

## Deployment Architecture

```
Local Development:
  npm run dev → Next.js dev server on localhost:3000
  SQLite (prisma/dev.db) for local database

Production (Vercel):
  GitHub push → Vercel webhook → build → deploy
  prisma generate runs in build (generates Linux-compatible binary)
  Neon PostgreSQL for production database
  Environment variables stored in Vercel dashboard
```

### Why Vercel?
- Zero-config Next.js deployment
- Automatic SSL
- Global CDN
- Serverless functions for API routes
- Free tier sufficient for side projects

---

## Interview Questions & Answers

### "Walk me through your system architecture"
> "It is a Next.js 15 full-stack app deployed on Vercel. The frontend is
> React with the Vercel AI SDK's useChat hook for streaming. The API route
> uses streamText() to call GPT-4o-mini with tool definitions. GPT uses
> the str_replace_editor tool to create and modify files in a virtual
> filesystem. The virtual FS is serialized as JSON and persisted in a
> Neon PostgreSQL database via Prisma. Auth is custom JWT with bcrypt."

### "How do you handle LLM errors and retries?"
> "Currently the onError handler logs errors to the console. For production,
> I would add exponential backoff retries for rate limit errors (429s),
> streaming error recovery by catching mid-stream failures, and user-facing
> error messages when the LLM is unavailable."

### "How would you scale this to 10,000 users?"
> "The main bottleneck is OpenAI API costs and rate limits. I would add:
> 1. Request queuing with Redis/BullMQ for rate limit management
> 2. Usage tracking per user to enforce limits
> 3. Caching common component patterns with Redis
> 4. Connection pooling for the database (Neon supports this natively)
> 5. Streaming responses are already efficient — no change needed there"

### "What is the ReAct pattern?"
> "ReAct stands for Reason + Act. The LLM alternates between reasoning
> (generating text explaining what it will do) and acting (calling tools).
> Each tool result is fed back into the context, allowing the LLM to
> reason about the result before taking the next action. This loop
> continues until the LLM decides it is done. It is the foundation of
> AI agents. In this app, GPT reasons about what files to create, calls
> str_replace_editor to create them, sees the result, then creates the
> next file."

### "What is the difference between streaming and non-streaming LLM calls?"
> "Non-streaming: you send a request and wait for the complete response —
> could be 10-30 seconds of waiting. Streaming: the model sends tokens as
> it generates them via Server-Sent Events. The client renders each token
> immediately, giving sub-100ms time-to-first-token. For user-facing
> applications, streaming is almost always the right choice."

### "How would you add RAG to this?"
> "RAG (Retrieval-Augmented Generation) would let the model reference a
> library of existing components. I would embed all components into a
> vector database (like Pinecone or pgvector on Neon). When a user prompts,
> I would embed the prompt, find the top-k similar components, and inject
> them into the system prompt as examples. This reduces hallucinations and
> improves consistency."

---

## What to Build Next (Skill Progression)

| Feature | Skill it teaches |
|---|---|
| Usage limits + Stripe | Full-stack product thinking |
| RAG with component library | Vector embeddings, similarity search |
| Multi-file refactoring | Complex agentic workflows |
| Code execution sandbox | WebContainers, security sandboxing |
| Fine-tuned model | Fine-tuning, RLHF basics |
| Evals pipeline | LLM evaluation, benchmarking |

---

## Key Concepts Checklist for AI Engineering Interviews

- [ ] Streaming LLM responses (SSE / chunked transfer)
- [ ] Tool calling / function calling
- [ ] Agentic loops (ReAct pattern)
- [ ] System prompt engineering
- [ ] Token limits and context windows
- [ ] RAG (Retrieval-Augmented Generation)
- [ ] Vector embeddings and similarity search
- [ ] LLM evaluation and evals pipelines
- [ ] Cost optimization (model selection, caching, batching)
- [ ] Observability (tracing LLM calls with Langsmith/Helicone)
