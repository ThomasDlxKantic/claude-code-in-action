# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

UIGen is an AI-powered React component generator with live preview. It uses Claude AI to generate React components based on user descriptions, displays them in a live preview, and persists them for registered users. Components are stored in a virtual file system (not written to disk).

## Development Commands

### Setup
```bash
npm run setup
```
Installs dependencies, generates Prisma client, and runs database migrations.

### Development
```bash
npm run dev
```
Starts Next.js dev server with Turbopack on http://localhost:3000

```bash
npm run dev:daemon
```
Starts dev server in background, writing logs to logs.txt

### Testing
```bash
npm test
```
Runs all tests with Vitest

To run a single test file:
```bash
npx vitest src/components/chat/__tests__/MessageList.test.tsx
```

To run tests in watch mode:
```bash
npm test -- --watch
```

### Database
```bash
npx prisma migrate dev
```
Create and apply new migration

```bash
npm run db:reset
```
Reset database (WARNING: deletes all data)

```bash
npx prisma studio
```
Open Prisma Studio to view/edit database

### Build & Lint
```bash
npm run build
```
Build production bundle

```bash
npm run lint
```
Run ESLint

## Architecture

### AI Provider System
- **Location**: `src/lib/provider.ts`
- **Mock Provider**: If no `ANTHROPIC_API_KEY` is set in `.env`, a `MockLanguageModel` is used instead
- The mock provider generates static components (Counter, ContactForm, Card) without calling the API
- Real provider uses Claude Haiku 4.5 via Vercel AI SDK

### Virtual File System
- **Location**: `src/lib/file-system.ts`
- **Core Class**: `VirtualFileSystem` - manages an in-memory file tree
- Files are NOT written to disk; everything exists in memory
- Serializes/deserializes to JSON for database persistence
- Supports create, read, update, delete, move, copy operations
- Path normalization ensures consistent `/` prefixed paths

### AI Tools Integration
Two custom tools are provided to the AI model:

1. **str_replace_editor** (`src/lib/tools/str-replace.ts`)
   - Commands: view, create, str_replace, insert
   - Operates on the VirtualFileSystem
   - Used by AI to create/modify component files

2. **file_manager** (`src/lib/tools/file-manager.ts`)
   - Commands for file operations (move, copy, delete, list, etc.)
   - Also operates on VirtualFileSystem

### Chat Flow
- **Route**: `src/app/api/chat/route.ts`
- System prompt from `src/lib/prompts/generation.ts` is added with cache control
- VirtualFileSystem is reconstructed from serialized files sent in request
- Uses `streamText` with tools for multi-step agentic generation
- On completion, saves messages + file system state to database (if authenticated)
- Max duration: 120 seconds

### Authentication & Sessions
- **Location**: `src/lib/auth.ts`
- JWT-based sessions stored in httpOnly cookies
- Session duration: 7 days
- Anonymous users can use the app but data is not persisted
- Authenticated users have projects saved to database

### Database Schema
- **Location**: `prisma/schema.prisma`
- SQLite database at `prisma/dev.db`
- Prisma client generated to `src/generated/prisma/`
- **Models**:
  - `User`: id, email, password (bcrypt), projects relation
  - `Project`: id, name, userId (nullable), messages (JSON string), data (JSON string for VFS)

### Component Preview
- **Location**: `src/components/preview/PreviewFrame.tsx`
- Uses Babel standalone to transform JSX in-browser
- Imports resolved from virtual file system
- React 19 + ReactDOM rendered in iframe

### Code Transformation
- **Location**: `src/lib/transform/jsx-transformer.ts`
- Babel transforms JSX to JS at runtime for preview
- Handles import path resolution from virtual file system

### Server Actions
- **Location**: `src/actions/`
- `create-project.ts` - Create new project
- `get-project.ts` - Fetch single project by ID
- `get-projects.ts` - List all projects for user

## Testing

- Test framework: Vitest with jsdom environment
- Testing library: React Testing Library
- Tests located in `__tests__` directories alongside components
- Focus on UI component behavior, user interactions, and state management

## Key Patterns

### Project Persistence
- Projects are saved automatically on chat completion if user is authenticated
- Messages exclude system prompt before saving
- VirtualFileSystem is serialized to JSON string in `data` column
- On load, files are deserialized back into VirtualFileSystem

### Path Aliases
TypeScript path alias `@/*` maps to `src/*` (configured in tsconfig.json)

### Route Structure
- `/` - Landing page (anonymous or authenticated)
- `/[projectId]` - Project workspace with chat + preview
- `/api/chat` - Streaming chat endpoint

## Environment Variables

Required in `.env` (optional):
```
ANTHROPIC_API_KEY=your-api-key-here
JWT_SECRET=your-secret-key  # defaults to "development-secret-key"
```

App works without `ANTHROPIC_API_KEY` using mock provider.
