# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Commands

```bash
npm run dev          # Start dev server with Turbopack
npm run build        # Production build
npm run lint         # Run ESLint
npm run test         # Run Vitest unit tests
npm run setup        # Install deps + generate Prisma client + run migrations
npm run db:reset     # Reset SQLite database
```

Path alias `@/*` maps to `./src/*`.

## Architecture

UIGen is an AI-powered React component generator. Users describe components in a chat, Claude generates the code, and the result renders in a live preview iframe.

**Request flow:**
1. User submits a message in `ChatInterface`
2. `chat-context.tsx` POSTs to `/api/chat` with the current virtual file system state serialized as JSON
3. `api/chat/route.ts` calls Claude (via Vercel AI SDK + `lib/provider.ts`) with tools enabled
4. Claude calls `str_replace_editor` and/or `file_manager` tools to write/edit files
5. The response streams back; `file-system-context.tsx` applies tool call results to the in-memory VFS
6. `PreviewFrame.tsx` re-renders the updated files in a sandboxed iframe via Babel standalone
7. If the user is authenticated, the project (messages + VFS state) is persisted to SQLite via Prisma

**Virtual File System (`lib/file-system.ts`):** All generated files live in memory as a `Map<path, content>`. There is no disk I/O for user files. The VFS serializes to JSON for the database and for sending to the AI on each request.

**Live preview (`components/preview/PreviewFrame.tsx`):** Generated JSX is compiled in-browser with Babel standalone and rendered inside a sandboxed `<iframe>`. The transformer lives in `lib/transform/jsx-transformer.ts`.

**AI tools (`lib/tools/`):**
- `str-replace.ts` — `str_replace_editor` tool: creates files or replaces ranges of text in existing files
- `file-manager.ts` — `file_manager` tool: rename/delete/list files

**System prompt (`lib/prompts/generation.tsx`):** Contains the full instructions Claude receives for generating React components. Edit this to change AI behavior.

**AI provider (`lib/provider.ts`):** Wraps Anthropic SDK. Falls back to a static mock response when `ANTHROPIC_API_KEY` is not set.

**Auth:** JWT sessions via `jose`, stored in HTTP-only cookies. `middleware.ts` protects API routes. Passwords hashed with bcrypt.

## Database

SQLite via Prisma. Schema: `User` (id, email, password) → `Project` (id, name, userId?, messages: JSON string, data: JSON string of serialized VFS).

After changing `prisma/schema.prisma`, run `npx prisma migrate dev`.

## Environment Variables

| Variable | Purpose |
|---|---|
| `ANTHROPIC_API_KEY` | Claude API key (optional — mock used if absent) |
| `JWT_SECRET` | Session signing key |
| `NODE_ENV` | Set to `production` for secure cookies |
