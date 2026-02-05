# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

UIGen is an AI-powered React component generator. Users describe components in natural language, Claude generates them, and a live preview renders in a sandboxed iframe. Supports anonymous usage and authenticated users with project persistence.

## Commands

- `npm run dev` — Start dev server (Next.js + Turbopack)
- `npm run build` — Production build
- `npm run lint` — ESLint
- `npm test` — Run all tests (Vitest)
- `npx vitest run src/path/to/test.test.ts` — Run a single test file
- `npm run setup` — First-time setup (install, Prisma generate, migrate)
- `npm run db:reset` — Reset SQLite database
- `npx prisma generate` — Regenerate Prisma client after schema changes (output: `src/generated/prisma/`)

## Environment Variables

All optional. Defined in `.env` in the project root.

- `ANTHROPIC_API_KEY` — If empty/missing, the app uses a `MockLanguageModel` that generates static demo components (useful for development without API access).
- `JWT_SECRET` — Defaults to `"development-secret-key"` if not set.

## Architecture

**Stack:** Next.js 15 (App Router) / React 19 / TypeScript / Tailwind CSS v4 / Prisma + SQLite

### Core Data Flow

User message → `/api/chat` (streaming) → Claude (`claude-haiku-4-5`) responds with tool calls → Virtual FileSystem updated → Babel transforms JSX in-browser → Preview renders in sandboxed iframe via ESM import maps + blob URLs

### Key Subsystems

- **Virtual File System** (`lib/file-system.ts`): In-memory tree of `FileNode` objects. No disk writes. Serialized as JSON for DB persistence. Exposed to components via `FileSystemContext`.

- **AI Integration** (`lib/provider.ts`, `api/chat/route.ts`): Uses Vercel AI SDK with Anthropic Claude (max 40 tool-use steps). Falls back to `MockLanguageModel` when no API key. AI operates on the virtual FS through two tools:
  - `str_replace_editor` — create/view/edit files (defined in `lib/tools/str-replace.ts`)
  - `file_manager` — rename/delete files (defined in `lib/tools/file-manager.ts`)
  - System prompt is in `lib/prompts/generation.tsx`

- **Preview Pipeline** (`lib/transform/jsx-transformer.ts`, `components/preview/PreviewFrame.tsx`): Babel standalone transforms JSX/TSX → extracts imports → builds ESM import map pointing to esm.sh CDN → each virtual file becomes a blob URL → renders in sandboxed iframe. Entry point is always `/App.jsx` or `/App.tsx`.

- **Auth** (`lib/auth.ts`, `actions/index.ts`): JWT sessions in httpOnly cookies (7-day expiry, jose + bcrypt). Middleware (`middleware.ts`) protects `/api/projects` and `/api/filesystem`.

- **Anonymous Mode** (`lib/anon-work-tracker.ts`): Tracks anonymous user work via localStorage. No DB persistence—work is lost on refresh unless the user signs in.

### State Management

Two React Contexts manage global state:
- `FileSystemContext` (`lib/contexts/file-system-context.tsx`) — virtual FS instance, selected file, file operations, refresh triggers. Its `handleToolCall` method processes both AI tool types.
- `ChatContext` (`lib/contexts/chat-context.tsx`) — message history, streaming status, tool call integration with FS via Vercel AI SDK's `useAIChat`.

### Layout

`main-content.tsx` uses `react-resizable-panels` to create a three-panel layout: Chat | Editor (with FileTree + CodeEditor tabs) | Preview.

### UI Components

Shadcn/ui components live in `components/ui/` (style: new-york). Path alias `@/*` maps to `src/*`.

### Routes

- `/` — Home. Authenticated users redirect to their most recent project; anonymous users get `MainContent` directly.
- `/[projectId]` — Project page. Requires auth, validates ownership.
- `/api/chat` — POST. Streaming AI chat endpoint. Saves project to DB on finish if authenticated.

## Testing

Vitest with jsdom environment and React Testing Library. Tests are colocated in `__tests__/` directories next to their source files.

## Database

Prisma with SQLite (`prisma/dev.db`). Prisma client is generated to a non-default path: `src/generated/prisma/`. Two models: `User` (auth) and `Project` (stores messages and virtual FS data as JSON strings).
