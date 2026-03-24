# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Commands

```bash
# Initial setup (installs deps, generates Prisma client, runs migrations)
npm run setup

# Development server (uses Turbopack)
npm run dev

# Build for production
npm run build

# Run tests
npm test

# Run a single test file
npx vitest run src/components/__tests__/<test-file>.test.tsx

# Lint
npm run lint

# Reset database
npm run db:reset
```

The dev command requires `node-compat.cjs` via `NODE_OPTIONS` — this is handled automatically by the npm scripts.

## Environment

Copy `.env` and set `ANTHROPIC_API_KEY`. Without it, the app falls back to a `MockLanguageModel` automatically — the app is fully functional without an API key for development.

## Architecture

### Request Flow

1. User types in `ChatInterface` → triggers Vercel AI SDK's `useChat()` hook
2. POST to `/api/chat/route.ts` → `streamText()` with Anthropic Claude claude-haiku-4-5
3. AI responds with text and/or tool calls (`str_replace_editor`, `file_manager`)
4. Tools mutate the virtual file system, updates stream back to the client
5. `FileSystemContext` reflects changes → `PreviewFrame` re-renders live preview

### Key Abstractions

**Virtual File System** (`src/lib/file-system.ts`): An in-memory `VirtualFileSystem` class — no disk writes. Files are serialized to JSON and persisted in the Prisma `Project.data` field. Tool definitions in `src/lib/tools/` expose this to the AI.

**Contexts** (`src/lib/contexts/`):
- `ChatContext` — wraps Vercel AI SDK's `useChat`, manages message state and project persistence
- `FileSystemContext` — holds the `VirtualFileSystem` instance; components subscribe to file changes

**AI Tools** (`src/lib/tools/`): Tool definitions passed to `streamText`. The AI uses `str_replace_editor` for file edits and `file_manager` for file operations (create, delete, rename).

**JSX Transformer** (`src/lib/transform/jsx-transformer.ts`): Babel-standalone converts JSX → JS at runtime for live preview in `PreviewFrame` (iframe-based).

**Authentication** (`src/lib/auth.ts`): JWT in HTTP-only cookies via JOSE. Server Actions handle sign-up/sign-in/sign-out. Middleware at `src/middleware.ts` protects `/api/projects` and `/api/filesystem` routes. Anonymous users can use the app without accounts.

### Database

Prisma + SQLite (`prisma/dev.db`). Two models:
- `User` — email/password (bcrypt-hashed)
- `Project` — belongs to User; stores `messages` (chat history) and `data` (file system) as JSON blobs

After schema changes: `npx prisma migrate dev`.

### Layout

`main-content.tsx` renders three resizable panels via `react-resizable-panels`:
- Left: chat (`ChatInterface`)
- Right top: preview (`PreviewFrame`) / code tabs
- Right bottom: file tree (`FileTree`) + code editor (`CodeEditor`, Monaco)

### Path Alias

`@/*` resolves to `./src/*` (configured in `tsconfig.json`).

### Testing

Vitest with jsdom + Testing Library. Test files live in `__tests__` directories colocated with source. Run a specific test: `npx vitest run <path>`.
