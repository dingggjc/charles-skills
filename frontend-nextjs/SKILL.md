---
name: frontend-nextjs
description: >
  Apply this skill for any Next.js or React frontend task — scaffolding a new project,
  creating pages, components, hooks, features, tables, modals, context, or setting up
  API layers and auth middleware. Trigger whenever the user mentions Next.js, React,
  frontend, components, pages, routes, TanStack Query, shadcn, table, modal, context,
  or asks to build/add/create anything on the frontend side. Also trigger when the user
  says "start a new project", "scaffold", "new feature", "add a page/component/hook/table/modal".
---

# Frontend Next.js Skill

Standards and scaffold guide for all Next.js/React projects.

## Stack

| Layer | Tool |
|---|---|
| Framework | Next.js (App Router) |
| Language | TypeScript |
| Styling | Tailwind CSS + shadcn/ui |
| Icons | Hugeicons (`@hugeicons/react`) — never Lucide or others |
| Data Fetching | TanStack Query (Client Components only) |
| HTTP Client | Axios with a shared instance |
| Forms | React Hook Form + Zod |
| Auth | Cookie-based with Next.js middleware |

---

## Project Folder Structure

When scaffolding a new project, generate this structure:

```
src/
├── app/                          ← Next.js App Router pages
│   ├── (auth)/
│   │   └── login/
│   │       └── page.tsx
│   ├── dashboard/
│   │   ├── page.tsx
│   │   └── components/           ← route-scoped components
│   └── layout.tsx
├── features/                     ← vertical slice per domain, always flat (never nest features)
│   └── [domain-name]/            ← plural domain name e.g. users/, products/, categories/
│       ├── context/              ← feature context (thin wrapper around the main hook)
│       │   └── UsersContext.tsx
│       ├── table/                ← table component + column definitions
│       │   └── UsersTable.tsx
│       ├── modals/               ← all modals for this feature
│       │   └── UsersModals.tsx
│       ├── components/           ← other feature-scoped components
│       ├── hooks/                ← all feature hooks
│       │   ├── useUsers.ts       ← main hook, owns all state + logic
│       │   ├── useGetUsers.ts    ← TanStack Query hook
│       │   └── useCreateUser.ts  ← TanStack Mutation hook
│       ├── schemas/              ← Zod schemas + inferred types for this feature
│       │   └── createUserSchema.ts
│       ├── api.ts                ← all axios calls for this feature
│       └── types.ts              ← all feature-specific types
├── components/                   ← global shared components (used in 2+ unrelated features)
│   └── ui/                       ← shadcn components live here
├── lib/                          ← third-party config, utilities, shared resources
│   ├── axios.ts                  ← shared axios instance
│   ├── enums/                    ← e.g. Roles.ts, Status.ts
│   ├── helpers/                  ← e.g. formatCurrency.ts, route.helper.ts
│   ├── constants/                ← e.g. apiRoutes.ts
│   └── schemas/                  ← Zod schemas shared across 2+ features
├── hooks/                        ← global shared hooks
└── providers/                    ← QueryProvider, global context providers
```

**Feature rules:**
- Features are always flat at the top level — never nest a feature inside another feature
- Each feature is fully self-contained
- Never import a feature-scoped file from outside its feature folder
- If something is needed in 2+ features, move it to `src/components/` or `src/hooks/`

---

## State Management Pattern

### Hook — owns all state and logic

The main feature hook is the single source of truth. It holds all state, API calls, and derived values:

```typescript
// src/features/users/hooks/useUsers.ts
import { useState } from "react"
import { useQuery, useMutation } from "@tanstack/react-query"
import { getUsers, deleteUser } from "../api"

export const useUsers = () => {
  const [searchKey, setSearchKey] = useState("")
  const [pageSize, setPageSize] = useState(10)
  const [pageNumber, setPageNumber] = useState(1)
  const [selectedId, setSelectedId] = useState<string | null>(null)
  const [isEditModalOpen, setIsEditModalOpen] = useState(false)
  const [isDeleteModalOpen, setIsDeleteModalOpen] = useState(false)

  const { data, isLoading } = useQuery({
    queryKey: ["users", { searchKey, pageSize, pageNumber }],
    queryFn: () => getUsers({ searchKey, pageSize, pageNumber }),
  })

  const handleOpenEdit = (id: string) => {
    setSelectedId(id)
    setIsEditModalOpen(true)
  }

  const handleOpenDelete = (id: string) => {
    setSelectedId(id)
    setIsDeleteModalOpen(true)
  }

  const handleCloseModals = () => {
    setSelectedId(null)
    setIsEditModalOpen(false)
    setIsDeleteModalOpen(false)
  }

  return {
    data,
    isLoading,
    searchKey,
    setSearchKey,
    pageSize,
    setPageSize,
    pageNumber,
    setPageNumber,
    selectedId,
    isEditModalOpen,
    isDeleteModalOpen,
    handleOpenEdit,
    handleOpenDelete,
    handleCloseModals,
  }
}

```

### Context — shares hook values across files in the same feature

Context is a thin wrapper. It does not hold any state itself — it just provides the hook's return values to all files in the feature without prop drilling:

```tsx
// src/features/users/context/UsersContext.tsx
"use client"

import { createContext, useContext } from "react"
import { useUsers } from "../hooks/useUsers"

type UsersContextType = ReturnType<typeof useUsers>

const UsersContext = createContext<UsersContextType | null>(null)

export const UsersProvider = ({ children }: { children: React.ReactNode }) => {
  const users = useUsers()

  return (
    <UsersContext.Provider value={users}>
      {children}
    </UsersContext.Provider>
  )
}

export const useUsersContext = () => {
  const context = useContext(UsersContext)
  if (!context) throw new Error("useUsersContext must be used within UsersProvider")
  return context
}

```

### Page — just wraps with Provider and renders

```tsx
// src/app/admin/users/page.tsx
import { UsersProvider } from "@/features/users/context/UsersContext"
import { UsersTable } from "@/features/users/table/UsersTable"
import { UsersModals } from "@/features/users/modals/UsersModals"

export default function UsersPage() {
  return (
    <UsersProvider>
      <UsersTable />
      <UsersModals />
    </UsersProvider>
  )
}
```

**When to use context vs not:**
- Same feature, multiple files need the same state → use context
- Single file only → pass from hook directly, no context needed
- Across completely separate features/pages → use `src/providers/` for global state

---

## Table Standards

### Structure

Every feature table lives in `src/features/[feature]/table/[Feature]Table.tsx`.

Tables always include:
- Filters (search, date range, etc.) at the top
- Skeleton loading state using pre-defined row/col arrays
- Empty state with a descriptive message
- `TablePagination` at the bottom when data is paginated
- Export button (PDF/Excel) shown conditionally when data exists

### Table Pattern

```tsx
// src/features/users/table/UsersTable.tsx
"use client"

import {
  Table, TableBody, TableCell,
  TableHead, TableHeader, TableRow,
} from "@/components/ui/table"
import { Input } from "@/components/ui/input"
import { Skeleton } from "@/components/ui/skeleton"
import { TablePagination } from "@/components/ui/table-pagination"
import { useUsersContext } from "../context/UsersContext"

const skeletonRows = Array.from({ length: 4 })
const skeletonCols = Array.from({ length: 3 })
const totalCols = 3

export const UsersTable = () => {
  const {
    data,
    isLoading,
    searchKey,
    setSearchKey,
    pageSize,
    pageNumber,
    handleOpenEdit,
    handleOpenDelete,
  } = useUsersContext()

  const handleSearchChange = (e: React.ChangeEvent<HTMLInputElement>) => {
    setSearchKey(e.target.value)
  }

  const hasData = (data?.items.length ?? 0) > 0

  return (
    <div className="flex flex-col gap-4">
      <div className="flex items-center justify-between">
        <Input
          placeholder="Search..."
          value={searchKey}
          onChange={handleSearchChange}
          className="max-w-sm"
        />
      </div>

      <div className="overflow-x-auto rounded-md border">
        <Table>
          <TableHeader>
            <TableRow>
              <TableHead>Name</TableHead>
              <TableHead>Email</TableHead>
              <TableHead>Actions</TableHead>
            </TableRow>
          </TableHeader>
          <TableBody>
            {isLoading ? (
              skeletonRows.map((_, i) => (
                <TableRow key={i}>
                  {skeletonCols.map((__, j) => (
                    <TableCell key={j}>
                      <Skeleton className="h-4 w-24" />
                    </TableCell>
                  ))}
                </TableRow>
              ))
            ) : !hasData ? (
              <TableRow>
                <TableCell
                  colSpan={totalCols}
                  className="h-20 text-center text-muted-foreground"
                >
                  No data found.
                </TableCell>
              </TableRow>
            ) : (
              data!.items.map((item) => (
                <TableRow key={item.id}>
                  <TableCell>{item.name}</TableCell>
                  <TableCell>{item.email}</TableCell>
                  <TableCell>
                    {/* row actions */}
                  </TableCell>
                </TableRow>
              ))
            )}
          </TableBody>
        </Table>
      </div>

      {data?.pageDetails && (
        <TablePagination
          pageDetails={data.pageDetails}
          pageSize={pageSize}
          onPageChange={(page) => {}}
          onPageSizeChange={(size) => {}}
          itemLabel="users"
        />
      )}
    </div>
  )
}

```

### TablePagination — always a shared global component

`TablePagination` lives in `src/components/ui/table-pagination.tsx` and is reused across all features. It handles ellipsis logic, page size select, and prev/next buttons internally. Always pass `itemLabel` to describe what's being paginated.

### Table variable naming

Use camelCase for all local table variables — never SCREAMING_SNAKE_CASE:

```typescript
// ❌ wrong
const SKELETON_ROWS = Array.from({ length: 4 })
const TOTAL_COLS = 4

// ✅ correct
const skeletonRows = Array.from({ length: 4 })
const totalCols = 4
```

---

## Modal Standards

All modals for a feature live in `src/features/[feature]/modals/`:

```tsx
// src/features/users/modals/UsersModals.tsx
"use client"

import { useUsersContext } from "../context/UsersContext"
import { UserEditModal } from "./UserEditModal"
import { UserDeleteModal } from "./UserDeleteModal"

export const UsersModals = () => {
  const { isEditModalOpen, isDeleteModalOpen, selectedId, handleCloseModals } = useUsersContext()

  return (
    <>
      <UserEditModal
        open={isEditModalOpen}
        userId={selectedId}
        onClose={handleCloseModals}
      />
      <UserDeleteModal
        open={isDeleteModalOpen}
        userId={selectedId}
        onClose={handleCloseModals}
      />
    </>
  )
}

```

---

## Scaffolding — Files to Generate

### 1. Axios Shared Instance (`src/lib/axios.ts`)

```typescript
import axios from "axios"

const api = axios.create({
  baseURL: process.env.NEXT_PUBLIC_API_URL,
  withCredentials: true,
})

api.interceptors.response.use(
  (response) => response,
  (error) => Promise.reject(error)
)

export default api
```

### 2. TanStack Query Provider (`src/providers/QueryProvider.tsx`)

```tsx
"use client"

import { QueryClient, QueryClientProvider } from "@tanstack/react-query"
import { useState } from "react"

const QueryProvider = ({ children }: { children: React.ReactNode }) => {
  const [queryClient] = useState(() => new QueryClient())

  return (
    <QueryClientProvider client={queryClient}>
      {children}
    </QueryClientProvider>
  )
}

export default QueryProvider
```

### 3. Root Layout (`src/app/layout.tsx`)

```tsx
import type { Metadata } from "next"
import { Inter } from "next/font/google"
import QueryProvider from "@/providers/QueryProvider"
import "./globals.css"

const inter = Inter({ subsets: ["latin"] })

export const metadata: Metadata = {
  title: "App",
  description: "",
}

export default function RootLayout({ children }: { children: React.ReactNode }) {
  return (
    <html lang="en">
      <body className={inter.className}>
        <QueryProvider>{children}</QueryProvider>
      </body>
    </html>
  )
}
```

### 4. Middleware (`middleware.ts`) — only if role-based auth is needed

Ask the user for their roles before generating this. Add one `pathname.startsWith()` block per role.

```typescript
import { NextRequest, NextResponse } from "next/server"
import { getDashboardRoute } from "@/lib/helpers/route.helper"

export const middleware = (request: NextRequest) => {
  const { pathname } = request.nextUrl

  const userCookie = request.cookies.get("user")
  let user = null

  if (userCookie?.value) {
    try {
      user = JSON.parse(userCookie.value)
    } catch {
      return NextResponse.redirect(new URL("/login", request.url))
    }
  }

  const role = user?.role
  const isLoggedIn = !!role

  if (pathname === "/" || pathname === "/login") {
    if (isLoggedIn) {
      return NextResponse.redirect(new URL(getDashboardRoute(role), request.url))
    }
    return NextResponse.next()
  }

  // Add role-based route guards here per project

  return NextResponse.next()
}

export const config = {
  matcher: ["/", "/login"],
}
```

### 5. New Feature Scaffold

When the user says "add a new feature" or "create a [name] feature", generate this full structure:

```
src/features/[feature-name]/
├── context/
│   └── [Feature]Context.tsx     ← thin provider wrapping the main hook
├── table/
│   └── [Feature]Table.tsx       ← table + filters, consumes context
├── modals/
│   └── [Feature]Modals.tsx      ← all modals, consumes context
├── components/                  ← other UI components for this feature
├── hooks/
│   └── use[Feature].ts          ← all state, API calls, modal control
├── api.ts                       ← all axios calls
└── types.ts                     ← all types and interfaces
```

---

## Coding Standards

### Function Syntax

Always arrow functions — never function declarations for variable-assigned functions:

```typescript
// ❌ wrong
function handleSubmit() {}

// ✅ correct
const handleSubmit = () => {}
```

Never inline functions in JSX:

```tsx
// ❌ wrong
<button onClick={() => doSomething()}>Click</button>

// ✅ correct
const handleClick = () => doSomething()
<button onClick={handleClick}>Click</button>
```

---

### Page Components

One page = one function. Keep pages thin — delegate everything to feature components:

```tsx
// ✅ correct — page is just a shell
export default function UsersPage() {
  return (
    <UsersProvider>
      <UsersTable />
      <UsersModals />
    </UsersProvider>
  )
}
```

- PascalCase + `Page` suffix matching the route
- Component name must match filename
- No anonymous default exports
- Default exports for page components only
- Never put business logic, state, or API calls directly in the page

---

### Component Placement Rules

| Scope | Location |
|---|---|
| Used in 2+ unrelated features | `src/components/` |
| Used only in one feature | `src/features/[feature]/components/` |
| Used only in one route | `src/app/[route]/components/` |

- Never import a feature-scoped file from outside its feature folder
- If a component outgrows its scope, move it up to the nearest shared ancestor

---

### Reusable Component Checklist

Before creating any new component, always ask these questions in order:

**1. Does it already exist in `src/components/`?**
→ Use it. Never recreate something that already exists globally.

**2. Will this be used in more than one feature?**
→ Build it reusable from the start and put it in `src/components/`. Never write the same component twice.

**3. Only used in this one feature?**
→ Put it in `src/features/[feature]/components/`. If it gets reused later, move it up.

**Components that must always be global (`src/components/`):**

| Component | Why |
|---|---|
| `SearchInput` | Every table has a search input |
| `TablePagination` | Every paginated table needs it |
| `ExportDropdown` | PDF/Excel export repeats across reports |
| `DatePickerWithRange` | Used across filters and reports |
| `ConfirmDialog` | Delete confirmations appear everywhere |
| `PageHeader` | Every page has a title + optional action button |

**How to build a reusable component:**
- Accept all variable parts as props — never hardcode labels, placeholders, or handlers
- Keep it purely presentational where possible — no API calls inside
- Name it generically, not after a feature (`SearchInput` not `UsersSearchInput`)

```tsx
// ✅ correct — generic, reusable
interface SearchInputProps {
  value: string
  onChange: (value: string) => void
  placeholder?: string
  className?: string
}

export const SearchInput = ({ value, onChange, placeholder = "Search...", className }: SearchInputProps) => {
  const handleChange = (e: React.ChangeEvent<HTMLInputElement>) => {
    onChange(e.target.value)
  }

  return (
    <Input
      value={value}
      onChange={handleChange}
      placeholder={placeholder}
      className={className}
    />
  )
}

```

```tsx
// ❌ wrong — hardcoded, not reusable
const UsersSearchInput = () => {
  const { searchKey, setSearchKey } = useUsersContext()

  return (
    <Input
      value={searchKey}
      onChange={(e) => setSearchKey(e.target.value)}
      placeholder="Search users..."
    />
  )
}
```

---

### Component Standards

- Always functional components — never class components
- One responsibility per file
- Use `export const` inline for all non-page components, hooks, and utilities
- Default exports for page components only
- Named exports match filename exactly (`UserCard.tsx` exports `UserCard`)
- No inline functions in JSX

---

### Icons

Always Hugeicons — never Lucide or any other icon library:

```tsx
import { HugeiconsIcon } from "@hugeicons/react"
import { SomeIcon } from "@hugeicons/core-free-icons"

<HugeiconsIcon icon={SomeIcon} size={16} />
```

---

### TanStack Query Hook Naming

Hook names always mirror the API call exactly — one hook per API call, one file per hook:

```
src/features/users/
├── api.ts
└── hooks/
    ├── useGetUsers.ts       ← mirrors getUsers
    ├── useGetUserById.ts    ← mirrors getUserById
    ├── useCreateUser.ts     ← mirrors createUser
    ├── useUpdateUser.ts     ← mirrors updateUser
    ├── useDeleteUser.ts     ← mirrors deleteUser
    └── useUsers.ts          ← main feature hook, composes the above
```

**Query hooks (useQuery):**

```typescript
// src/features/users/hooks/useGetUsers.ts
import { useQuery } from "@tanstack/react-query"
import { getUsers } from "../api"
import { GetUsersParams } from "../types"

export const useGetUsers = (params: GetUsersParams) => {
  return useQuery({
    queryKey: ["users", params],
    queryFn: () => getUsers(params),
  })
}

```

**Mutation hooks (useMutation):**

```typescript
// src/features/users/hooks/useCreateUser.ts
import { useMutation, useQueryClient } from "@tanstack/react-query"
import { createUser } from "../api"

export const useCreateUser = () => {
  const queryClient = useQueryClient()

  return useMutation({
    mutationFn: createUser,
    onSuccess: () => {
      queryClient.invalidateQueries({ queryKey: ["users"] })
    },
  })
}

```

**Main feature hook composes them:**

```typescript
// src/features/users/hooks/useUsers.ts
import { useState } from "react"
import { useGetUsers } from "./useGetUsers"
import { useCreateUser } from "./useCreateUser"
import { useDeleteUser } from "./useDeleteUser"

export const useUsers = () => {
  const [searchKey, setSearchKey] = useState("")
  const [pageSize, setPageSize] = useState(10)
  const [pageNumber, setPageNumber] = useState(1)
  const [selectedId, setSelectedId] = useState<string | null>(null)
  const [isEditModalOpen, setIsEditModalOpen] = useState(false)
  const [isDeleteModalOpen, setIsDeleteModalOpen] = useState(false)

  const { data, isLoading } = useGetUsers({ searchKey, pageSize, pageNumber })
  const { mutate: createUser } = useCreateUser()
  const { mutate: deleteUser } = useDeleteUser()

  const handleOpenEdit = (id: string) => {
    setSelectedId(id)
    setIsEditModalOpen(true)
  }

  const handleOpenDelete = (id: string) => {
    setSelectedId(id)
    setIsDeleteModalOpen(true)
  }

  const handleCloseModals = () => {
    setSelectedId(null)
    setIsEditModalOpen(false)
    setIsDeleteModalOpen(false)
  }

  return {
    data,
    isLoading,
    searchKey,
    setSearchKey,
    pageSize,
    setPageSize,
    pageNumber,
    setPageNumber,
    selectedId,
    isEditModalOpen,
    isDeleteModalOpen,
    handleOpenEdit,
    handleOpenDelete,
    handleCloseModals,
    createUser,
    deleteUser,
  }
}

```

**Rules:**
- Hook name = `use` + API function name in PascalCase (`getUsers` → `useGetUsers`)
- One hook per API call, one file per hook
- Query hooks return `useQuery` directly — no extra wrapping
- Mutation hooks always invalidate the relevant query key on `onSuccess`
- Main feature hook (`use[Feature]`) composes individual hooks + owns all UI state

---

### Forms (React Hook Form + Zod)

```tsx
import { useForm } from "react-hook-form"
import { zodResolver } from "@hookform/resolvers/zod"
import { z } from "zod"

const schema = z.object({
  email: z.string().email(),
  password: z.string().min(8),
})

type FormValues = z.infer<typeof schema>

export const LoginForm = () => {
  const { register, handleSubmit, formState: { errors } } = useForm<FormValues>({
    resolver: zodResolver(schema),
  })

  const onSubmit = (data: FormValues) => {
    // api call
  }

  return (
    <form onSubmit={handleSubmit(onSubmit)}>
      {/* fields */}
    </form>
  )
}

```

---

### Naming Conventions

| Type | Convention | Example |
|---|---|---|
| Page components | PascalCase + `Page` suffix | `DashboardPage.tsx` |
| UI components | PascalCase | `UserCard.tsx` |
| Hooks | camelCase + `use` prefix | `useAuthState.ts` |
| Utilities | camelCase | `formatDate.ts` |
| Variables & constants | camelCase | `totalCols`, `skeletonRows`, `apiRoutes` |
| Enums | PascalCase | `Roles.ts` |
| Types/Interfaces | PascalCase | `User`, `ApiResponse` |

- camelCase for everything — no SCREAMING_SNAKE_CASE anywhere
- Name variables by what they represent, not by making them stand out

---

### lib/ Folder Conventions

`lib/` is strictly for third-party config, utilities, and shared resources — nothing feature-specific goes here:

```
src/lib/
├── axios.ts             ← shared axios instance
├── enums/               ← e.g. Roles.ts, Status.ts
├── helpers/             ← e.g. formatCurrency.ts, route.helper.ts
├── constants/           ← e.g. apiRoutes.ts
└── schemas/             ← Zod schemas shared across 2+ features
```

### Zod Schema Placement

```
src/features/users/
└── schemas/
    └── createUserSchema.ts    ← feature-specific schema, stays here

src/lib/schemas/
└── paginationSchema.ts        ← used in 2+ features, goes here
```

Schema file structure:

```typescript
// src/features/users/schemas/createUserSchema.ts
import { z } from "zod"

export const createUserSchema = z.object({
  email: z.string().email(),
  password: z.string().min(8),
})

export type CreateUserFormValues = z.infer<typeof createUserSchema>
```

**Rules:**
- Schema + inferred type always in the same file
- Feature-specific schema → `features/[domain]/schemas/`
- Shared across 2+ features → `lib/schemas/`
- Never define schemas inside component files

No barrel exports (`index.ts`) — import directly from the file.

---

### General Code Quality

- No inline comments — code should be self-explanatory, never narrate what the code is doing
- No logic duplication — extract to hooks or helpers
- Use named constants, never magic values
- Prefer explicit over implicit
- No inline functions in JSX

---

## CLAUDE.md Auto-Generation

When the user says "scaffold a new project", "start a new project", or "create a new project", always auto-generate a `CLAUDE.md` file at the root of the project.

### Step 1 — Ask these questions first

1. What is the project name?
2. What does it do and who is it for?
3. What roles does it have? (e.g. Admin, Cashier, SuperAdmin)
4. What are the key features?
5. Any environment variables needed beyond the defaults?

### Step 2 — Generate filled CLAUDE.md

```markdown
# [Project Name]

## What is this project
[Filled from user answer]

## Stack

### Frontend
- Framework: Next.js 15 (App Router)
- Styling: Tailwind CSS + shadcn/ui
- Icons: Hugeicons
- Data fetching: TanStack Query
- Forms: React Hook Form + Zod
- HTTP: Axios

### Backend
- Framework: .NET 10, ASP.NET Core Web API
- Architecture: Clean Architecture + MediatR CQRS
- ORM: EF Core
- Database: PostgreSQL (Neon)
- Auth: JWT (cookie-based)

## Roles
- [Filled from user answer]

## Key Features
- [Filled from user answer]

## Folder Structure
[Generate based on scaffolded structure]

## Environment Variables

### Frontend (.env.local)
\```
NEXT_PUBLIC_API_URL=
\```

### Backend (appsettings.json)
\```
ConnectionStrings__DefaultConnection=
JwtSettings__Secret=
JwtSettings__Issuer=
JwtSettings__Audience=
\```

## Notes
[Any project-specific rules or context]
```
