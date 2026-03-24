# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Commands

```bash
npm run setup        # First-time setup: install deps, generate Prisma client, run migrations
npm run dev          # Start dev server with Turbopack
npm run build        # Production build
npm run lint         # ESLint
npm run test         # Run all tests
npm run test -- --run src/lib/__tests__/file-system.test.ts  # Run a single test file
npm run db:reset     # Reset database (drops and recreates)
```

Environment: add `ANTHROPIC_API_KEY` to `.env` to enable real AI responses. Without it, the app uses a `MockLanguageModel` that returns static component examples.

## Architecture

UIGen is a Next.js 15 App Router app where users describe React components in a chat and see a live preview. The key design decision is a **virtual file system** — generated files never touch disk; everything lives in memory and is serialized to the database.

### Virtual File System (`src/lib/file-system.ts`)

The `VirtualFileSystem` class stores files in a `Map` and is the core abstraction. It supports `createFile`, `updateFile`, `deleteFile`, `rename`, `viewFile`, `replaceInFile`, `insertInFile`, `serialize`/`deserialize`. The AI manipulates files through this interface, not the real filesystem.

### AI Integration (`src/app/api/chat/route.ts`)

Uses Vercel AI SDK's `streamText` with two tools exposed to Claude:
- `str_replace_editor` — view/create/replace in files
- `file_manager` — delete/rename files

The route saves the updated project state (serialized VFS + message history) to the database after each response completes.

Model is configured in `src/lib/provider.ts` — defaults to Claude Haiku 4.5, falls back to mock when no API key is present.

### State Management

Two React contexts manage the client state:
- `FileSystemContext` (`src/lib/contexts/file-system-context.tsx`) — holds VFS state, handles tool call results from the AI stream
- `ChatContext` (`src/lib/contexts/chat-context.tsx`) — wraps Vercel AI SDK's `useChat`, tracks anonymous user work for later persistence

### Live Preview (`src/components/preview/PreviewFrame.tsx` + `src/lib/transform/jsx-transformer.ts`)

The preview iframe renders JSX by:
1. Using Babel Standalone to transform JSX → JS in the browser
2. Building an import map pointing to `esm.sh` CDN for npm packages
3. Injecting the result into a sandboxed iframe via blob URLs

Entry point detection checks for `App.jsx`, `index.jsx`, etc. in the VFS.

### Authentication (`src/lib/auth.ts`)

JWT cookies via `jose`. Sessions expire after 7 days. Supports anonymous users — their projects are stored with a `null` userId and can be claimed after registration.

### Database

Prisma with SQLite. `User` and `Project` models. `Project.data` stores the serialized VFS as JSON string. `Project.messages` stores the chat history as JSON string.

### UI Layout (`src/app/main-content.tsx`)

Three-panel layout: left = chat, right = preview/code tabs. Code tab shows a file tree (`src/components/editor/`) and Monaco editor side-by-side. Uses `react-resizable-panels`.

UI components come from shadcn/ui (new-york style, Tailwind v4) in `src/components/ui/`.

## Code Style

Agrega comentarios en el código regularmente, sobretodo en las partes complejas.

## Testing

Tests use Vitest + React Testing Library with jsdom. Test files are colocated in `__tests__` directories next to the code they test. The file system has the most comprehensive tests (`src/lib/__tests__/file-system.test.ts`).
