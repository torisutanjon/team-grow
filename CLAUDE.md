# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Development Commands

`yarn` is our package manager. DO NOT USE ANY OTHER PACKAGE MANAGERS. Use the following commands for development, quality checks, and testing:

```bash
# Development
yarn dev                # Start dev server with Turbopack
yarn build              # Production build (use NODE_ENV=production)
yarn start              # Start production server

# Quality Checks
yarn type-check         # TypeScript type checking
yarn lint               # ESLint with Prettier
yarn prettier:fix       # Auto-fix formatting
yarn format:check       # Check formatting only

# Testing (TDD workflow required)
yarn test               # Run all tests
yarn test:watch         # Watch mode for development
yarn test:coverage      # Generate coverage report
```

## Architecture Overview

### Next.js 15 App Router

- **App directory only** - no pages/ directory
- **Server Components by default** - only use 'use client' when necessary (state, events, browser APIs, or dependencies requiring it)
- **No /api routes** - use Supabase client directly (exceptions: EHR integration, payments, webhooks, email/SMS)
- **Server Actions** - located in `/app/actions/` for server-side operations

### Supabase Integration

**Four distinct clients** (use correct one for context):

```tsx
// 1. Server Components & Route Handlers
import { createClient } from "@/lib/supabase/server";
const supabase = await createClient();

// 2. Client Components
import { createClient } from "@/lib/supabase/client";
const supabase = createClient();

// 3. Middleware (session refresh)
import { updateSession } from "@/lib/supabase/middleware";

// 4. Admin operations (service role)
import { createClient } from "@/lib/supabase/service";
const supabase = createClient();
```

**Critical:**

- **NEVER** use `@supabase/auth-helpers-nextjs` (forbidden)
- All database types in `lib/types/supabase.ts`
- RLS policies must be implemented for all tables

### Data Fetching Patterns

```tsx
// Server Components: Direct queries
async function Page() {
  const supabase = await createClient();
  const { data } = await supabase.from("table").select();
  return <Component data={data} />;
}

// Client Components: React Query hooks
("use client");
import { useFetchCategories } from "@/lib/hooks/queries";
function Component() {
  const { data } = useFetchCategories();
  return <div>{data}</div>;
}
```

### Component Standards (Radix UI)

**Always use Radix UI components** (never native HTML):

```tsx
import { Box, Heading, Text, Button, TextField } from '@radix-ui/themes'

// ✅ Correct
<Box>
  <Heading>Title</Heading>
  <Text>Description</Text>
  <TextField.Root placeholder="Input" />
  <Button>Submit</Button>
</Box>

// ❌ Wrong - never use native HTML elements
<div>
  <h1>Title</h1>
  <p>Description</p>
  <input placeholder="Input" />
  <button>Submit</button>
</div>
```

**Component rules:**

- `<Box>` instead of `<div>`
- `<Heading>` instead of `<h1>`-`<h6>`
- `<Text>` instead of `<p>`
- `<Button>` instead of `<button>`
- `TextField.Root` for all text inputs
- `Dialog` from `@radix-ui/themes`
- Minimize use of `<Card>` (prefer `<Box>`)

### State Management

- **React Context** - global state (auth, cart, filters, dialogs)
- **TanStack Query** - server state and data fetching
- **Provider tree** - composed with `buildProviderTree()` helper in `/lib/utils/provider.ts`

### Required Tech Stack

**Approved packages only:**

- Next.js 15
- TypeScript (strict mode, no `any` types)
- Tailwind CSS
- Supabase (`@supabase/ssr`, `@supabase/supabase-js`)
- Radix UI (`@radix-ui/themes`, `@radix-ui/react-icons`)
- `react-dropzone` (file uploads)
- `@dnd-kit/core` (drag-and-drop reordering)
- `@react-pdf/renderer` (PDF generation)
- `react-hook-form` (forms)
- `react-quill-new` (rich text)
- `resend`, `twilio` (email/SMS)

**Package manager:** yarn only (never npm)

## Test-Driven Development

**TDD is mandatory:**

1. Write tests FIRST
2. Make them fail
3. Implement feature
4. Iterate until passing

**Coverage thresholds:**

- Statements: 90%
- Branches: 85%
- Functions: 95%
- Lines: 90%

**Test structure:**

- Follow arrange-act-assert pattern
- Test edge cases (empty states, errors, loading)
- Use React Testing Library
- Tests in `__tests__/` folders or co-located

## File Conventions

- **Naming:** lowercase-with-hyphens for files/folders
- **Path alias:** `@/*` maps to project root
- **Barrel exports:** use `index.ts` for component exports
- **Client components:** extract to separate files when using 'use client'

## TypeScript Standards

- Strict mode enabled
- No `any` types (must be documented if absolutely necessary)
- Explicit return types required for all functions
- Null checks handled properly
- Database types synchronized with Supabase schema

## Build Requirements

- Use `NODE_ENV=production yarn build` for production builds
- No TypeScript errors allowed
- No linting errors allowed
- All tests must pass

## Supabase Query Optimization

See `docs/ai/supabase-query-optimization.md` for detailed guide.

- **Select specific columns** — never use `.select('*')`
- **Use joins** — `!inner` for required relations, avoid N+1 queries
- **Pagination** — `.range()` with `{ count: 'exact' }`, structured response with `{ success, data, pagination }`
- **Use `.maybeSingle()`** when row might not exist (`.single()` throws on zero rows)
- **Use `.rpc()`** for complex queries involving stored procedures
- **Index filter columns** — columns in `.eq()`, `.ilike()`, `.overlaps()` need database indexes
- **RLS performance** — wrap `auth.uid()` in `(select auth.uid())` to prevent per-row evaluation

```tsx
// ✅ Correct
const { data } = await supabase
  .from("products")
  .select("id, name, price, product_variants!inner(id, name)")
  .eq("status", "active")
  .range(0, 19);

// ❌ Wrong
const { data } = await supabase.from("products").select("*");
```

## React Hooks

See `docs/ai/react-hooks-conventions.md` for detailed guide.

- **Complete dependency arrays** — include all values used inside hooks
- **`useCallback`** — only when passing functions as props to memoized children
- **`useMemo`** — only for expensive computations (O(n) operations on large lists)
- **Custom hooks** — `lib/hooks/` for domain, `lib/hooks/utils/` for utilities, `lib/query/hooks/` for TanStack Query
- **No conditional hooks** — always call at the top level of the component
- **No data fetching in `useEffect`** — use TanStack Query hooks instead

## Styling & Theming

See `docs/ai/styling-and-theming.md` for detailed guide.

- **CSS variables for theme colors** — defined in `tailwind.config.ts`, never hardcode hex values
- **Radix UI theme tokens first** — use component props for color/size/variant
- **Tailwind utilities second** — for layout, spacing, custom styling
- **Fonts** — `font-main` and `font-accent` only, never import fonts directly
- **Breakpoints** — use existing `xs` (360px) through `xl` (1880px), don't add new ones
- **No CSS modules, no inline styles** — Tailwind + CSS variables only

## Rollbar Logging

See `docs/ai/rollbar-logging.md` for detailed guide.

- **Server-side** — `import { serverInstance } from '@/lib/rollbar'`
- **Client-side** — `import { clientConfig } from '@/lib/rollbar'` for Rollbar provider
- **NEVER log PHI** — no patient names, DOBs, health conditions, EHR response bodies
- **Severity levels** — `critical` (app crash), `error` (operation failed), `warning` (degraded), `info` (notable events)
- **Always include context** — store ID, action name, non-PHI identifiers
- **React error boundaries** — use for client-side crash reporting with Rollbar

## Webhook & Event Logging

See `docs/ai/webhook-event-logging.md` for detailed guide.

- **Webhook routes in `/app/api/`** — approved exception to the no-API-routes rule
- **Use `ServerWebhookProcessor` pattern** — class-based routing in `lib/webhook/webhook-processor.ts`
- **Always log via `logWebhookEvent()`** — both success and failure states
- **Fire-and-forget** — logging failures must never affect webhook response
- **Required fields** — `storeId`, `source`, `eventType`, `status`, `summary`, `requestPayload`
- **New integrations** — add processor in `lib/webhook/processes/`, route through `ServerWebhookProcessor`

## Server Actions

See `docs/ai/server-actions.md` for detailed guide.

- **Location** — all actions in `/app/actions/`, one file per domain, `'use server'` at file top
- **Return structured responses** — `{ success: boolean, data?: T, error?: string }`
- **Revalidate after mutations** — call `revalidatePath()` or `revalidateTag()`
- **Error handling** — try/catch, log to Rollbar, return sanitized error message
- **Auth check** — verify authentication at the start of every action

## TanStack Query

See `docs/ai/tanstack-query.md` for detailed guide. Also see `lib/query/README.md`.

- **Hook location** — `lib/query/hooks/`, named `useFetch<Entity>` and `useUpdate<Entity>`
- **Query keys** — arrays with entity name and dependencies: `['products', { categoryId, status }]`
- **Invalidate after mutations** — `queryClient.invalidateQueries({ queryKey: ['entity'] })`
- **Use `enabled`** — for queries that depend on other data: `enabled: !!storeId`
- **No direct `fetch()` in client components** — always use TanStack Query

## Error Handling

See `docs/ai/error-handling.md` for detailed guide.

- **Server Components** — let errors bubble to `error.tsx` boundaries
- **Client Components** — use TanStack Query `error`/`isError` states
- **Server Actions** — try/catch → Rollbar → `{ success: false, error: 'message' }`
- **API routes** — try/catch → Rollbar → HTTP status code
- **Never swallow errors** — always log to Rollbar or surface to user
- **Sanitize user-facing errors** — no stack traces, no internal details, no DB column names

## Common Pitfalls to Avoid

❌ Using pages/ directory → use app/ directory
❌ Creating /api routes → use Supabase client directly
❌ Using `@supabase/auth-helpers-nextjs` → use `@/lib/supabase` clients
❌ Using native HTML elements → use Radix UI components
❌ Writing code before tests → TDD required
❌ Using npm → use yarn
❌ Using `any` type → define proper types
❌ Skipping RLS policies → security requirement
❌ Using `.select('*')` → specify columns explicitly
❌ Using `.single()` for optional rows → use `.maybeSingle()`
❌ Data fetching in `useEffect` → use TanStack Query
❌ Hardcoded hex colors → use CSS variables via Tailwind
❌ Logging PHI to Rollbar → use error codes and non-PHI context
❌ Empty catch blocks → log to Rollbar, return error response
❌ Direct `fetch()` in client components → use TanStack Query hooks
