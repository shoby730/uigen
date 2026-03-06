# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Commands

```bash
# First-time setup
npm run setup          # install + prisma generate + migrate

# Development
npm run dev            # Next.js with Turbopack (uses node-compat.cjs shim)
npm run dev:daemon     # Same but in background, logs to logs.txt

# Build & production
npm run build
npm run start

# Testing
npm test               # vitest (jsdom environment)
npm test -- --run src/lib/__tests__/file-system.test.ts  # single test file

# Linting
npm run lint           # next lint

# Database
npm run db:reset       # reset and re-run all migrations
npx prisma migrate dev # create/apply new migrations
npx prisma generate    # regenerate client (output: src/generated/prisma)
```

The `NODE_OPTIONS='--require ./node-compat.cjs'` prefix in dev/build scripts is required — don't remove it.

## Architecture

This is a Next.js 15 App Router app where users chat with Claude to generate React components that render in a live preview.

### Core data flow

1. User sends a chat message
2. `/api/chat/route.ts` calls `streamText` with two AI tools: `str_replace_editor` and `file_manager`
3. Claude uses these tools to create/edit files in a `VirtualFileSystem` (in-memory, no disk writes)
4. Tool calls stream back to the client via Vercel AI SDK
5. `ChatProvider` (`src/lib/contexts/chat-context.tsx`) receives tool calls via `onToolCall` and dispatches them to `FileSystemProvider` (`src/lib/contexts/file-system-context.tsx`)
6. `FileSystemProvider` mutates the `VirtualFileSystem` and triggers re-render via `refreshTrigger`
7. `PreviewFrame` reads all files, compiles them with Babel (`@babel/standalone`) in-browser via `createImportMap()`, and renders in an `<iframe>` using ES module import maps

### Virtual file system

`src/lib/file-system.ts` — `VirtualFileSystem` class with a tree of `FileNode` objects stored in a `Map<string, FileNode>`. This is the source of truth for all generated code. It serializes to/from plain objects for persistence (stored as JSON in the `Project.data` DB column).

### Preview rendering (`src/lib/transform/jsx-transformer.ts`)

- `transformJSX()` — Babel-transforms JSX/TSX to JS
- `createImportMap()` — builds an ES import map: local files become blob URLs, third-party packages resolve to `https://esm.sh/<pkg>`
- `createPreviewHTML()` — generates the full iframe HTML with import map, Tailwind CDN, and error boundaries
- `@/` path alias is supported in the virtual FS

### AI tools

- `src/lib/tools/str-replace.ts` — `str_replace_editor` tool: `view`, `create`, `str_replace`, `insert` commands operating on `VirtualFileSystem`
- `src/lib/tools/file-manager.ts` — `file_manager` tool: `rename`, `delete` commands
- `src/lib/prompts/generation.tsx` — system prompt sent to Claude (cached with Anthropic's prompt caching)

### Provider / mock mode

`src/lib/provider.ts` — If `ANTHROPIC_API_KEY` is unset, a `MockLanguageModel` is used that returns static hardcoded component examples. Real mode uses `claude-haiku-4-5`.

### Auth

Custom JWT auth using `jose`. Sessions stored in `auth-token` httpOnly cookie (7-day expiry). `src/lib/auth.ts` is server-only. `src/middleware.ts` verifies sessions for protected routes.

### Database

Schema is defined in `prisma/schema.prisma` — reference it whenever you need to understand the structure of data stored in the database.

Prisma + SQLite (`prisma/dev.db`). Two models:
- `User` — email/password (bcrypt)
- `Project` — stores `messages` (JSON array) and `data` (serialized VirtualFileSystem) as string columns

Anonymous users can use the app without signing in; their work is tracked in `src/lib/anon-work-tracker.ts` (localStorage) and can be saved on sign-up.

### UI layout

`src/app/main-content.tsx` — root layout: resizable left panel (chat) + right panel (preview/code tabs). Right panel's code view shows `FileTree` + `CodeEditor` (Monaco).

Contexts must be nested: `FileSystemProvider` wraps `ChatProvider` (chat depends on file system).

### Component library

shadcn/ui components in `src/components/ui/`. Tailwind CSS v4.
