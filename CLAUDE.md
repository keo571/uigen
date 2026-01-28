# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

UIGen is an AI-powered React component generator with live preview. It allows users to describe components in natural language and generates them in real-time using Claude AI. The application features a virtual file system (no files written to disk), live preview with hot reload, and component persistence for registered users.

## Development Commands

### Setup
```bash
npm run setup
```
Installs dependencies, generates Prisma client, and runs database migrations.

### Development Server
```bash
npm run dev
```
Starts Next.js development server with Turbopack on http://localhost:3000

```bash
npm run dev:daemon
```
Starts dev server in background, writing logs to logs.txt

### Testing
```bash
npm test
```
Runs Vitest test suite (configured with jsdom for React component testing)

### Database
```bash
npm run db:reset
```
Resets database (force resets migrations and clears all data)

```bash
npx prisma studio
```
Opens Prisma Studio to view/edit database contents

```bash
npx prisma generate
```
Regenerates Prisma client after schema changes (client outputs to `src/generated/prisma`)

### Build
```bash
npm run build
```
Creates production build

```bash
npm run start
```
Runs production server

### Linting
```bash
npm run lint
```
Runs Next.js ESLint

## Architecture

### Virtual File System
The core of UIGen is a **virtual file system** ([src/lib/file-system.ts](src/lib/file-system.ts)) that exists entirely in memory. Files are never written to disk during component generation.

- `VirtualFileSystem` class provides CRUD operations for files/directories
- Files are stored in a `Map<string, FileNode>` structure
- Supports serialization/deserialization for persistence to database
- Implements text editor commands (view, create, str_replace, insert) that the AI uses to modify files

### AI Agent Architecture
The AI agent uses the Vercel AI SDK's `streamText` API with tool calling:

1. **API Route** ([src/app/api/chat/route.ts](src/app/api/chat/route.ts)) - Main endpoint that:
   - Deserializes virtual file system from request
   - Injects system prompt from [src/lib/prompts/generation.tsx](src/lib/prompts/generation.tsx)
   - Streams responses with tool calls
   - Saves conversation and files to database on completion (for authenticated users)

2. **AI Tools** ([src/lib/tools/](src/lib/tools/)):
   - `str_replace_editor` - Main tool for file operations (view, create, str_replace, insert)
   - `file_manager` - Additional file management utilities

3. **Provider** ([src/lib/provider.ts](src/lib/provider.ts)):
   - Falls back to `MockLanguageModel` if no `ANTHROPIC_API_KEY` is set
   - Mock provider returns static component templates for demo purposes
   - Real provider uses Claude Haiku 4.5

### Live Preview System
Preview is powered by a client-side JSX transformer and import map:

1. **JSX Transformation** ([src/lib/transform/jsx-transformer.ts](src/lib/transform/jsx-transformer.ts)):
   - Uses `@babel/standalone` to transform JSX/TSX to JavaScript in the browser
   - Creates blob URLs for each transformed file
   - Generates an ES import map mapping file paths to blob URLs
   - Handles `@/` alias for imports (maps to root `/`)
   - Detects and collects CSS imports
   - Provides comprehensive error handling for syntax errors

2. **Preview Frame** ([src/components/preview/PreviewFrame.tsx](src/components/preview/PreviewFrame.tsx)):
   - Renders transformed code in sandboxed iframe
   - Detects entry point (App.jsx, App.tsx, index.jsx, etc.)
   - Injects Tailwind CSS via CDN
   - Updates preview on file system changes
   - Shows syntax errors with formatted display

3. **File System Context** ([src/lib/contexts/file-system-context.tsx](src/lib/contexts/file-system-context.tsx)):
   - React context providing virtual file system access
   - Triggers preview refresh via `refreshTrigger` state
   - Manages file operations throughout the app

### Authentication & Projects
- **Authentication** ([src/lib/auth.ts](src/lib/auth.ts)): JWT-based sessions with bcrypt password hashing
- **Anonymous Work Tracker** ([src/lib/anon-work-tracker.ts](src/lib/anon-work-tracker.ts)): Prevents anonymous users from creating unlimited projects
- **Database** ([prisma/schema.prisma](prisma/schema.prisma)): SQLite with User and Project models
  - Projects store serialized messages and virtual file system state as JSON strings
- **Middleware** ([src/middleware.ts](src/middleware.ts)): Protects API routes requiring authentication

### UI Components
- **shadcn/ui** components in [src/components/ui/](src/components/ui/)
- Custom components:
  - Chat interface ([src/components/chat/](src/components/chat/))
  - Code editor with Monaco ([src/components/editor/CodeEditor.tsx](src/components/editor/CodeEditor.tsx))
  - File tree ([src/components/editor/FileTree.tsx](src/components/editor/FileTree.tsx))
  - Preview frame ([src/components/preview/PreviewFrame.tsx](src/components/preview/PreviewFrame.tsx))

## Key Conventions

### Import Aliases
- Use `@/` prefix for all local imports (e.g., `import { VirtualFileSystem } from "@/lib/file-system"`)
- This is configured in [tsconfig.json](tsconfig.json) and understood by the AI agent

### File System Paths
- All paths in the virtual file system start with `/` (e.g., `/App.jsx`, `/components/Button.jsx`)
- The AI is instructed to always create an `/App.jsx` as the entry point
- No traditional folder structure exists - this is a virtual FS starting at root

### AI System Prompt
The AI agent is instructed to ([src/lib/prompts/generation.tsx](src/lib/prompts/generation.tsx)):
- Create React components using Tailwind CSS (never hardcoded styles)
- Always begin projects with a `/App.jsx` file that exports a default React component
- Use `@/` import alias for all non-library imports
- Keep responses brief
- Never create HTML files (App.jsx is the entry point)

### Database Schema
Prisma client is generated to `src/generated/prisma` (not the default `node_modules/.prisma/client`). This is specified in [prisma/schema.prisma](prisma/schema.prisma).

### Testing
- Vitest configured for React testing with jsdom environment
- Tests located in `__tests__` subdirectories (e.g., [src/components/chat/__tests__/](src/components/chat/__tests__/))
- Uses @testing-library/react for component testing

## Tech Stack
- **Framework**: Next.js 15 with App Router, React 19
- **Styling**: Tailwind CSS v4
- **Database**: Prisma with SQLite
- **AI**: Anthropic Claude via Vercel AI SDK
- **Code Editor**: Monaco Editor
- **JSX Transform**: Babel Standalone (client-side)
- **Testing**: Vitest + Testing Library
- **UI Components**: Radix UI primitives (via shadcn/ui)
