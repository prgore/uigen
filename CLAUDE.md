# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Commands

```bash
# First-time setup (install deps, generate Prisma client, run migrations)
npm run setup

# Development server (Turbopack)
npm run dev

# Build for production
npm run build

# Run all tests
npm test

# Run a single test file
npx vitest run src/components/chat/__tests__/ChatInterface.test.tsx

# Lint
npm run lint

# Reset the database
npm run db:reset

# After modifying prisma/schema.prisma
npx prisma migrate dev
npx prisma generate
```

The app requires `ANTHROPIC_API_KEY` in `.env`. Without it, a `MockLanguageModel` in `src/lib/provider.ts` is used instead, which returns static pre-written React components.

Auth uses JWT stored in an `httpOnly` cookie (`auth-token`). The default `JWT_SECRET` is `"development-secret-key"` — set `JWT_SECRET` in `.env` for production.

## Architecture

### Data flow overview

The app is a two-panel layout: a chat on the left drives a live React preview on the right. The AI generates files into an **in-memory virtual file system** (never written to disk). When a user is authenticated, the file system state and chat messages are serialized to JSON and persisted in a SQLite `Project` row.

```
User chat → POST /api/chat → Vercel AI SDK streamText → Claude tool calls
                                                          ↓
                                             VirtualFileSystem (server-side)
                                                          ↓
                                    onToolCall callback → FileSystemContext (client)
                                                          ↓
                                             refreshTrigger → PreviewFrame
                                                          ↓
                                      jsx-transformer → Blob URLs → <iframe srcdoc>
```

### Virtual File System (`src/lib/file-system.ts`)

`VirtualFileSystem` is a pure in-memory tree. It lives **in two places simultaneously**: once on the server inside the API route (to let the AI tools operate on it during a stream), and once on the client inside `FileSystemContext`. The server serializes it to `Record<string, FileNode>` after each stream; the client reconstructs it via `deserializeFromNodes`. The client copy is the source of truth for the UI; the server copy is reconstructed fresh from the client-sent `files` body on each POST.

### AI tools (`src/lib/tools/`)

Two tools are registered with `streamText`:
- `str_replace_editor` — create/str_replace/insert operations on the VFS
- `file_manager` — rename/delete operations

Both tools call `VirtualFileSystem` methods directly and return string results. The AI is instructed (system prompt in `src/app/api/chat/route.ts`) to always produce a root `/App.jsx` with a default export, and to use `@/` import alias for cross-file imports.

### Preview rendering (`src/lib/transform/jsx-transformer.ts`)

`PreviewFrame` calls `createImportMap` on every `refreshTrigger`. This:
1. Transforms each `.jsx/.tsx/.js/.ts` file with `@babel/standalone` (in the browser)
2. Creates a `Blob` URL for each transformed file
3. Builds an ES module import map: local files → blob URLs, third-party packages → `esm.sh` CDN, `@/` alias → root path
4. Injects it all into an `<iframe srcdoc>` HTML document that uses `<script type="importmap">` and `<script type="module">` to mount the React app

The iframe uses `allow-scripts allow-same-origin allow-forms` sandbox so blob URL imports work. Syntax errors from Babel are collected and displayed inside the iframe rather than crashing the preview.

### Context providers (`src/lib/contexts/`)

`FileSystemProvider` wraps the VFS instance and exposes mutation methods. It also contains `handleToolCall`, which the `ChatProvider` calls via the Vercel AI SDK's `onToolCall` hook so that AI tool calls are applied to the client-side VFS in real time during streaming.

`ChatProvider` wraps `useChat` from `@ai-sdk/react`, serializes the current VFS state into the POST body on every submission, and calls `handleToolCall` to keep the client VFS in sync with what the AI is doing.

### Auth (`src/lib/auth.ts`, `src/middleware.ts`)

Custom JWT auth using `jose`. Sessions last 7 days and are stored in an `httpOnly` cookie. Passwords are hashed with `bcrypt`. The middleware is minimal — auth checks happen in Server Actions and the API route directly.

### Project persistence

Authenticated users get their work saved: the API route's `onFinish` callback serializes both the full message history and the VFS state as JSON strings into the `Project.messages` and `Project.data` SQLite columns. The `[projectId]` page loads this back via `getProject` Server Action and passes it to `MainContent`, which hydrates `FileSystemProvider` via `initialData` and `ChatProvider` via `initialMessages`.

### Anonymous users

Anonymous sessions are tracked via `sessionStorage` (`src/lib/anon-work-tracker.ts`). When an anon user signs up or logs in after doing work, `HeaderActions` reads the stored session data so the work can be preserved (currently stored but migration to a new project is a manual flow).

### Testing

Tests use Vitest + jsdom + `@testing-library/react`. Test files live in `__tests__/` subdirectories next to the code they test. The vitest config (`vitest.config.mts`) uses `vite-tsconfig-paths` to resolve `@/` imports.

### Prisma

The generated client outputs to `src/generated/prisma` (not the default location). Import it as `import { prisma } from "@/lib/prisma"` which is a singleton with connection reuse for Next.js hot-reload safety.
