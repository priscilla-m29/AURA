# AURA
AI Assistance
How Aura was built — a tour

Aura is an AI-powered workplace assistant. Here's the full breakdown of how it works under the hood.

1. Stack & framework

- TanStack Start v1 (React 19 + Vite 7) — a full-stack framework with file-based routing and server functions/routes that run on the edge.
- TypeScript (strict) throughout.
- Tailwind CSS v4 with semantic design tokens defined in `src/styles.css`.
- shadcn/ui primitives + AI Elements (the official Vercel chat UI kit) for the conversation surface.
- Vercel AI SDK (`ai`, `@ai-sdk/react`, `@ai-sdk/openai-compatible`) for streaming chat.
- Lovable AI Gateway (`google/gemini-3-flash-preview`) as the model provider — no API key setup needed, it's auto-provisioned via `LOVABLE_API_KEY`.

2. Design system (Midnight Focus)

In `src/styles.css` I replaced the default tokens with a deep navy / electric indigo palette in `oklch`:

- `--background: oklch(0.13 0.04 280)` — near-black midnight
- `--primary: oklch(0.62 0.23 275)` — electric indigo
- `--gradient-aurora` — multi-stop indigo→violet gradient used on the logo, primary buttons, and the H1
- `--shadow-glow` — soft indigo halo used on hover/active states

Every color in components is referenced via these tokens (e.g. `bg-primary`, `bg-card`, `text-muted-foreground`) — no hardcoded colors.

3. Routing structure

File-based routes under `src/routes/`:

```
__root.tsx              → app shell: <html>, sidebar layout, providers
index.tsx               → / landing page (logo, quick-start cards, CTA)
chat.$threadId.tsx      → /chat/:threadId — actual conversation page
api/chat.ts             → server route (POST) — streams model responses
```

Each conversation lives at its own URL (`/chat/t_abc123`), so it's reloadable, shareable in this browser, and the back button works.

4. Conversation history (localStorage + threads)

Per your choice, threads persist in the browser only — no database needed.

`src/lib/threads.ts` exposes a `useThreads()` hook that:
- Hydrates from `localStorage["aura.threads.v1"]` on mount (guarded by `typeof window !== "undefined"` so SSR doesn't break).
- Provides `createThread`, `deleteThread`, `updateThread`.
- Auto-derives a thread title from the first user message (truncated to 48 chars).
- Re-orders threads by recency on every update.

The sidebar (`src/components/AppSidebar.tsx`) renders the thread list, highlights the active one via `useParams`, and has a hover-to-reveal delete button.

5. The chat page (`chat.$threadId.tsx`)

This is the heart of the app:

1. `useChat` from `@ai-sdk/react` — manages the message array, streaming state, and submission. Keyed by `threadId` so messages can't bleed between threads.
2. `DefaultChatTransport` points at `/api/chat`.
3. Initial messages are loaded from the localStorage thread.
4. Persistence: a `useEffect` watches `messages` + `status`; when a stream finishes, it calls `updateThread(threadId, messages)`.
5. Seeded prompts: the landing page's quick-start cards navigate with `?q=...`, and the chat page auto-sends that prompt once on mount — so clicking "Draft an email" starts the conversation immediately.
6. Focus management: the textarea auto-focuses on mount, after sending, and after streaming finishes.

The UI uses AI Elements primitives: `Conversation` + `ConversationContent` (with auto-scroll), `Message` + `MessageContent`, `PromptInput` + `PromptInputTextarea` + `PromptInputSubmit`, and `Shimmer` for the "Aura is thinking…" state. Markdown is rendered with `react-markdown` so the model's lists, headings, and bold text display properly.

6. The server (`src/routes/api/chat.ts`)

A TanStack server route that:

```ts
const gateway = createLovableAiGatewayProvider(process.env.LOVABLE_API_KEY!);
const result = streamText({
  model: gateway("google/gemini-3-flash-preview"),
  system: SYSTEM_PROMPT,
  messages: await convertToModelMessages(messages),
});
return result.toUIMessageStreamResponse({ originalMessages: messages });
```

The `LOVABLE_API_KEY` stays server-side; the browser only ever sees streamed UI message parts.

7. Prompt engineering (the system prompt)

The system prompt in `api/chat.ts` is what makes Aura feel purpose-built rather than generic. It:

- Defines the persona: "Aura, an AI workplace assistant" for busy professionals.
- Specifies the four core capabilities with concrete output expectations (e.g. emails get a subject line + tone; meeting summaries get TL;DR + decisions + action items with owner & due date; project plans get milestones + dependencies).
- Sets a style contract: warm-but-efficient, markdown-formatted, asks clarifying questions before producing long outputs when context is missing.
- Adds ethical guardrails: refuse harmful, deceptive, or non-consensual tasks; never fabricate sources, recipients, or data.

This is why clicking "Draft an email" doesn't immediately spit out a generic email — Aura asks for recipient, goal, and tone first.

8. The flow end-to-end

1. User lands on `/`, clicks "Plan a project".
2. `createThread()` creates a new thread in localStorage; navigate to `/chat/t_xyz?q=...`.
3. Chat page mounts, `useChat` initializes with empty history, the seeded prompt is sent.
4. Browser POSTs the UI messages to `/api/chat`.
5. Server calls Lovable AI Gateway → Gemini 3 Flash with the system prompt + messages.
6. Tokens stream back; AI Elements renders them progressively with markdown.
7. On stream completion, the assistant message is saved into the thread in localStorage.
8. Thread title auto-updates to the first user message; sidebar reorders.

<img width="1366" height="720" alt="Screenshot 2026-05-15 101042" src="https://github.com/user-attachments/assets/187c4384-44a5-4788-b96b-f2603fc7abe8" />

