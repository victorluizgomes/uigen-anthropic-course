# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Commands

```bash
# Development
npm run dev          # Start dev server with Turbopack
npm run build        # Build for production
npm run start        # Start production server

# Testing
npm run test         # Run all Vitest unit tests

# Linting
npm run lint         # Run ESLint

# Database
npm run setup        # Install deps, init Prisma, run migrations
npm run db:reset     # Reset database (--force flag)
```

To run a single test file: `npx vitest run src/components/chat/__tests__/ChatInterface.test.tsx`

## Architecture

UIGen is an AI-powered React component generator. Users describe components in chat; Claude generates them and they render live in a preview pane.

### Data Flow

```
User Chat Input → /api/chat (Server Route) → Claude (via Vercel AI SDK)
  → AI Tools (str_replace_editor, file_manager) → Virtual File System
  → File System Context → Preview Frame (live render)
  → Prisma/SQLite (persisted for authenticated users)
```

### Core Concepts

**Virtual File System** (`src/lib/file-system.ts`): An in-memory file tree (`VirtualFileSystem` class). All AI-generated code lives here — nothing is written to disk. Serialized to JSON and stored in the `Project.data` column for persistence.

**AI Integration** (`src/app/api/chat/`, `src/lib/tools/`, `src/lib/prompts/`):
- Uses `@ai-sdk/anthropic` with Claude Haiku 4.5. Falls back to `MockLanguageModel` if `ANTHROPIC_API_KEY` is absent.
- Claude has two tools: `str_replace_editor` (view/create/edit files) and `file_manager` (rename/delete files).
- System prompt (`src/lib/prompts/generation.tsx`) instructs Claude to generate self-contained React components styled with Tailwind.

**Preview Rendering** (`src/components/preview/PreviewFrame.tsx`): Uses Babel standalone to transform JSX at runtime and renders it in an iframe/sandbox. Generated components must be self-contained.

**State Management**: Two main React contexts:
- `FileSystemContext` (`src/lib/contexts/file-system-context.tsx`) — owns the virtual file system instance
- `ChatContext` (`src/lib/contexts/chat-context.tsx`) — owns message history and streaming state

**Auth** (`src/lib/auth.ts`, `src/actions/`): JWT sessions in HTTP-only cookies, bcrypt password hashing. Anonymous users can use the app but projects aren't persisted.

### Key Paths

| Path | Purpose |
|------|---------|
| `src/app/api/chat/route.ts` | Streaming AI chat endpoint |
| `src/lib/file-system.ts` | Virtual file system |
| `src/lib/provider.ts` | AI model provider (real + mock) |
| `src/lib/prompts/generation.tsx` | Claude system prompt |
| `src/lib/tools/` | AI tool definitions |
| `src/lib/transform/` | JSX → JS transformation for preview |
| `src/app/[projectId]/` | Dynamic project routes |
| `prisma/schema.prisma` | Database schema (User, Project) — read this whenever you need to understand the DB structure |

### Tech Stack

- **Frontend**: Next.js 15 (App Router), React 19, Tailwind CSS v4, Radix UI/shadcn
- **Editor**: Monaco Editor (`@monaco-editor/react`)
- **AI**: Vercel AI SDK, `@ai-sdk/anthropic`
- **Database**: Prisma ORM + SQLite (`prisma/dev.db`)
- **Testing**: Vitest + Testing Library (jsdom environment)
- **Path alias**: `@/*` maps to `./src/*`
