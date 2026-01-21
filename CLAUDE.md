# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Commands

```bash
# Development
npm run dev              # Start dev server with Turbopack (http://localhost:3000)
npm run build            # Production build
npm run start            # Start production server

# Testing
npm run test             # Run all tests with Vitest
npx vitest run <path>    # Run specific test file
npx vitest --watch       # Watch mode

# Database
npm run setup            # Install deps + generate Prisma client + run migrations
npm run db:reset         # Reset database (drops all data)
npx prisma studio        # Open Prisma GUI

# Linting
npm run lint             # Run ESLint
```

## Architecture Overview

UIGen is an AI-powered React component generator that uses a **Virtual File System (VFS)** pattern - components are generated in memory, never written to disk.

### Core Data Flow

```
User Chat Input
  → POST /api/chat (with serialized VFS)
  → Claude AI with tools (str_replace_editor, file_manager)
  → Tool calls stream back to client
  → FileSystemProvider processes tool calls → VFS updates
  → PreviewFrame transforms JSX → import map → sandboxed iframe
  → onFinish: save to database (if authenticated)
```

### Key Directories

- `src/app/` - Next.js App Router pages and API routes
- `src/actions/` - Server actions for project CRUD
- `src/components/` - React components (chat/, editor/, preview/, auth/, ui/)
- `src/lib/` - Core logic:
  - `file-system.ts` - Virtual File System implementation
  - `contexts/` - ChatProvider (useAIChat wrapper) and FileSystemProvider
  - `tools/` - AI tool definitions (str_replace_editor, file_manager)
  - `transform/jsx-transformer.ts` - JSX→JS + import map generation
  - `prompts/generation.tsx` - System prompt for component generation
  - `auth.ts` - JWT-based auth with jose
  - `provider.ts` - AI model provider (Claude or MockLanguageModel fallback)

### Virtual File System

The VFS (`src/lib/file-system.ts`) stores files as `FileNode` objects in a `Map<string, FileNode>`:
- `createFile`, `readFile`, `updateFile`, `deleteFile`, `rename`
- `serialize()`/`deserialize()` for JSON conversion (database persistence)
- `viewFile`, `replaceInFile`, `insertInFile` for AI tool operations

### Preview System

`PreviewFrame` transforms VFS files into a live preview:
1. Finds entry point (`/App.jsx` or `/App.tsx`)
2. `createImportMap()` - Babel transforms JSX, creates blob URLs, builds ESM import map
3. Third-party packages resolved via esm.sh CDN
4. Rendered in sandboxed iframe with Tailwind CSS

### State Management

Two main React contexts wrap the app:
- **FileSystemProvider** - VFS instance, handles AI tool calls, triggers refresh
- **ChatProvider** - Wraps Vercel AI SDK's `useAIChat`, manages messages/input

### Database Schema

SQLite via Prisma (`prisma/schema.prisma`):
- **User**: email, password (bcrypt), projects relation
- **Project**: name, userId (nullable for anon), messages (JSON), data (serialized VFS)

### AI Tools

The AI uses two tools defined in `src/lib/tools/`:
1. **str_replace_editor** - view, create, str_replace, insert operations on files
2. **file_manager** - rename, delete operations

### Mock Provider

When `ANTHROPIC_API_KEY` is not set, `MockLanguageModel` returns static components for demo/testing.

## Testing

Tests use Vitest + React Testing Library + jsdom. Test files are co-located in `__tests__/` directories:
- `src/lib/__tests__/file-system.test.ts`
- `src/lib/contexts/__tests__/`
- `src/lib/transform/__tests__/`
- `src/components/chat/__tests__/`
- `src/components/editor/__tests__/`

## Path Aliases

- `@/*` maps to `./src/*` (configured in tsconfig.json)

