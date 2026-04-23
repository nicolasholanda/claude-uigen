# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Commands

```bash
npm run dev          # Start dev server (Turbopack)
npm run build        # Production build
npm run start        # Start production server
npm run lint         # ESLint
npm run test         # Run all tests (Vitest)
npx vitest run src/path/to/file.test.ts  # Run a single test file
npm run setup        # Fresh install: npm install + prisma generate + prisma migrate dev
npm run db:reset     # Reset database (destructive)
```

All scripts use `cross-env` for Windows compatibility.

## Environment Variables

```
ANTHROPIC_API_KEY=   # Optional. If missing, falls back to MockLanguageModel
JWT_SECRET=          # Optional. Defaults to "development-secret-key" in dev
```

## Architecture

UIGen is an AI-powered React component generator. Users describe components in natural language; Claude generates JSX files into an in-memory virtual file system, which are then transpiled and rendered live in an iframe.

### Virtual File System

All files exist only in memory — nothing is written to disk at runtime. `src/lib/file-system.ts` implements the full FS (create, read, update, delete, rename, serialize). It's used in two places:

- **Server-side** (`/api/chat`): Claude's tools (`str_replace_editor`, `file_manager`) operate on a temporary FS instance per request.
- **Client-side** (`FileSystemContext`): Mirrors the server FS state; updated optimistically as tool call results stream in.

When a project is saved, the FS is serialized to JSON and stored in `Project.data` (SQLite string column).

### AI Streaming Flow

1. Client sends `{ messages, files, projectId }` to `POST /api/chat`.
2. Server reconstructs a FS from `files`, then calls `streamText()` (Vercel AI SDK) with Claude tools.
3. Tool calls stream back as SSE. `ChatContext` receives them via `onToolCall` and dispatches to `FileSystemContext.handleToolCall()`.
4. `FileSystemContext` applies mutations (create/edit/rename/delete) to client FS state.
5. On stream end (`onFinish`), the server saves messages + FS snapshot to the database if `projectId` is set.

### Mock Provider

`src/lib/provider.ts` exports a single `getModel()` function. If `ANTHROPIC_API_KEY` is set it returns the real Anthropic model (`claude-haiku-4-5`). Otherwise it returns `MockLanguageModel`, which generates deterministic static components — useful for local dev without an API key.

### Preview Rendering (iframe)

`PreviewFrame` builds and injects a full HTML document into an iframe's `srcdoc`:

1. All JS/JSX files in the virtual FS are transformed with Babel (browser-side, `@babel/standalone`).
2. Each file becomes a blob URL.
3. An `importmap` is generated covering absolute paths (`/App.jsx`), relative, `@/` alias, and extension-less variants.
4. Third-party packages not in the FS resolve to `esm.sh`.
5. Tailwind CSS is loaded from CDN.

The entry point is always `/App.jsx`.

### Authentication

JWT sessions via `jose`. Tokens are stored in HTTP-only cookies (7-day expiry). `src/lib/auth.ts` handles create/get/delete. `src/middleware.ts` protects `/api/projects` and `/api/filesystem` routes. Project pages do an additional server-side `getSession()` check.

Anonymous users can generate components without signing in. On sign-up, `anon-work-tracker.ts` (localStorage) detects prior anonymous work and migrates it to a new project.

### Context Hierarchy

```
<FileSystemProvider>   ← owns virtual FS state, handles tool calls
  <ChatProvider>       ← wraps useChat(), passes onToolCall to FileSystemProvider
    <MainContent />    ← 3-panel layout (chat | preview | code editor)
```

`ChatProvider` and `FileSystemProvider` are tightly coupled: chat tool call results must flow into the FS context. Both are defined in `src/lib/contexts/`.

### Database

SQLite via Prisma. Two models: `User` (email + bcrypt password) and `Project` (messages and FS stored as JSON strings). The Prisma client is generated to `src/generated/prisma` (non-default output path — use that path for imports, not `@prisma/client`).
