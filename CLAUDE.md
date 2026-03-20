# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Commands

```bash
# Initial setup
npm run setup          # install deps + prisma generate + migrate

# Development
npm run dev            # Next.js dev server with turbopack
npm run build          # Production build
npm run lint           # ESLint

# Testing
npm test               # Run all tests (Vitest)
npm test -- src/lib/transform/__tests__/jsx-transformer.test.ts  # Single test file

# Database
npx prisma studio      # DB GUI
npm run db:reset       # Reset and re-run migrations
```

`NODE_OPTIONS='--require ./node-compat.cjs'` is prepended automatically by npm scripts — this patches Web Storage APIs for Node.js 25+ SSR compatibility.

## Architecture

UIGen is an AI-powered React component generator with live preview. Users describe components in a chat interface; Claude generates code via tool calls that modify an in-memory virtual file system, which is then compiled and rendered in a sandboxed iframe.

### Data Flow

1. **Chat** → `POST /api/chat` (route.ts) → Vercel AI SDK streams response with tool calls
2. **Tool calls** (str_replace_editor, file_manager) → modify `VirtualFileSystem` (in-memory, no disk writes)
3. **Preview** → Babel transforms JSX, esm.sh CDN resolves imports, rendered in sandboxed iframe
4. **Persistence** → Project messages + VFS serialized as JSON strings in SQLite via Prisma

### Key Abstractions

- **VirtualFileSystem** (`src/lib/file-system.ts`): In-memory map of filename → content. Serialized to JSON for DB storage.
- **AI Tools** (`src/lib/tools/`): `str_replace_editor` handles create/view/edit/insert operations; `file_manager` handles rename/delete.
- **JSX Transformer** (`src/lib/transform/jsx-transformer.ts`): Babel transforms JSX → JS, generates an import map pointing to esm.sh, produces full HTML for the preview iframe.
- **Contexts**: `ChatContext` owns AI interaction state and streams; `FileSystemContext` owns VFS state. Both live in `src/lib/contexts/`.
- **Provider** (`src/lib/provider.ts`): Returns Anthropic Claude (`claude-haiku-4-5`) if `ANTHROPIC_API_KEY` is set, otherwise a mock model that returns static output.

### Auth

JWT sessions via `jose`, stored in httpOnly cookies. `src/middleware.ts` protects `/[projectId]` routes. `src/lib/auth.ts` manages session creation/retrieval/deletion. Passwords hashed with bcrypt via server actions in `src/actions/index.ts`.

### Anonymous Users

Projects can be created without authentication (`userId` is optional in the schema). Anonymous project state is stored in browser storage and associated with a session on sign-in.

### Database Schema

```
User: id, email, password, timestamps
Project: id, name, userId (nullable), messages (JSON string), data (JSON string = VFS), timestamps
```

### Path Alias

`@/*` maps to `./src/*` (configured in `tsconfig.json`).

## Tech Stack

| Concern | Library |
|---|---|
| Framework | Next.js 15 App Router, React 19 |
| Styling | Tailwind CSS v4, Shadcn UI (new-york style, neutral) |
| Code editor | Monaco Editor |
| AI | Anthropic Claude via Vercel AI SDK |
| DB | SQLite + Prisma |
| Auth | jose (JWT) + bcrypt |
| Testing | Vitest + React Testing Library (jsdom) |
| Code transform | @babel/standalone |
| Module resolution (preview) | esm.sh CDN |
