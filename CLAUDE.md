# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Commands

```bash
# Initial setup (install deps + generate Prisma client + run migrations)
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

# Regenerate Prisma client after schema changes
npx prisma generate

# Run migrations
npx prisma migrate dev
```

## Architecture

UIGen is a Next.js 15 App Router application where users describe React components in a chat interface and see them rendered live in a preview pane. No files are written to disk — everything lives in an in-memory virtual file system.

### Data flow

1. User sends a chat message → `POST /api/chat` (`src/app/api/chat/route.ts`)
2. The route passes the current virtual FS state and chat history to Claude (or mock provider) via Vercel AI SDK's `streamText`
3. Claude responds with tool calls using two custom tools:
   - `str_replace_editor` — create/edit files via str_replace, insert, or full-file create
   - `file_manager` — rename or delete files
4. Tool call results stream back to the client; `ChatContext` intercepts them via `onToolCall` and forwards them to `FileSystemContext.handleToolCall`
5. `FileSystemContext` applies mutations to the in-memory `VirtualFileSystem` and increments `refreshTrigger`
6. `PreviewFrame` responds to `refreshTrigger`, calls `createImportMap` on all files (Babel-transforms JSX/TSX to JS, creates blob URLs), injects an import map into a sandboxed iframe's `srcdoc`

### Key abstractions

- **`VirtualFileSystem`** (`src/lib/file-system.ts`): In-memory tree of `FileNode` objects. Provides CRUD + text-editor operations (`replaceInFile`, `insertInFile`, `viewFile`). The `serialize`/`deserializeFromNodes` methods convert to/from plain JSON for API transport and Prisma storage.

- **`FileSystemContext`** (`src/lib/contexts/file-system-context.tsx`): React context that holds a single `VirtualFileSystem` instance and exposes mutations. `handleToolCall` is the bridge that routes AI tool calls to VFS mutations.

- **`ChatContext`** (`src/lib/contexts/chat-context.tsx`): Wraps Vercel AI SDK's `useChat`, attaches the serialized VFS to every request body, and pipes tool calls to `FileSystemContext`.

- **JSX transformer** (`src/lib/transform/jsx-transformer.ts`): Babel-based, browser-side transformer. Converts VFS files to blob URLs, builds an import map (with `@/` alias support and esm.sh fallback for third-party packages), and generates the full preview iframe HTML including Tailwind CDN and an error boundary.

- **AI provider** (`src/lib/provider.ts`): Returns `anthropic("claude-haiku-4-5")` when `ANTHROPIC_API_KEY` is set, otherwise falls back to `MockLanguageModel` which streams hard-coded counter/form/card components. The model is currently hardcoded to `claude-haiku-4-5`.

### Auth & persistence

- JWT sessions via `jose`, stored as an `httpOnly` cookie (`auth-token`). Session helpers are in `src/lib/auth.ts` (server-only).
- Middleware (`src/middleware.ts`) guards `/api/projects` and `/api/filesystem`.
- Authenticated users can save projects: messages + VFS state are JSON-stringified into `Project.messages` and `Project.data` columns (SQLite via Prisma).
- Anonymous work is tracked in `sessionStorage` (`src/lib/anon-work-tracker.ts`) so it can be migrated after sign-up.
- Prisma client is generated to `src/generated/prisma/` (non-standard output path — set in `prisma/schema.prisma`).

### Environment

- `ANTHROPIC_API_KEY` — required for real AI generation; app runs with mock provider without it
- `JWT_SECRET` — defaults to `"development-secret-key"` if unset
- `node-compat.cjs` is required via `NODE_OPTIONS` in all npm scripts to polyfill Node.js built-ins for Next.js/Turbopack compatibility

### Testing

Tests use Vitest + jsdom + React Testing Library. Test files live alongside source in `__tests__/` subdirectories.

### Database schema (`prisma/schema.prisma`)

SQLite via Prisma. Client generated to `src/generated/prisma/` (non-standard path).

- **`User`** — `id` (cuid), `email` (unique), `password`, timestamps; has many `Project`s
- **`Project`** — `id` (cuid), `name`, `userId` (nullable — anonymous projects allowed), `messages` (JSON string, default `"[]"`), `data` (JSON string for VFS state, default `"{}"`), timestamps; deleting a `User` cascades to their projects

Always reference `prisma/schema.prisma` when working with database structure or writing queries.

## Style rules

- Use comments sparingly. Only comment complex code.
