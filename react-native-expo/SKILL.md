---
name: react-native-expo
description: >
  Apply this skill for any React Native or Expo mobile app task — scaffolding a new
  project, creating screens, components, hooks, features, navigation, forms, or wiring
  data fetching. Trigger whenever the user mentions React Native, Expo, mobile app,
  screens, navigation, Expo Router, NativeWind, React Native Reusables, or asks to
  build/add/create anything for a mobile app. Also trigger when the user says "new
  screen", "new feature", "scaffold a mobile app", "add a component", or "add a hook".
---

# React Native Expo Skill

Standards and scaffold guide for all React Native Expo projects.

---

## COMMANDS

Parse the first word of `$ARGUMENTS` to determine the mode. Route accordingly.

| Command | Usage | What it does |
|---|---|---|
| `scaffold` | `/react-native-expo scaffold` | New project — ask the 5 CLAUDE.md questions, generate folder structure + axios + QueryProvider + CLAUDE.md |
| `feature` | `/react-native-expo feature <name>` | Scaffold a new vertical slice — creates `api/[domain]/`, `types/[domain]/`, `features/[domain]/` with all subfolders |
| `screen` | `/react-native-expo screen <name>` | Generate a single screen file using the screen boilerplate, placed at the correct Expo Router path |
| `component` | `/react-native-expo component <name>` | Generate a component using the boilerplate — ask if feature-scoped or global, place in the right folder |
| `hook` | `/react-native-expo hook <name>` | Generate a TanStack Query hook (query or mutation) — ask which type, mirror the API function name |
| `audit` | `/react-native-expo audit` | Read the existing codebase and check it against every standard in this skill. Output a report: ✅ pass / ❌ fail per rule. No code written. |

If `$ARGUMENTS` starts with none of the above, treat the full input as a task description and apply the relevant standards while completing it.

**After routing, strip the command keyword** — the remaining text is the subject. Example: `/react-native-expo feature products` → mode = feature, name = "products".

**`audit` mode — what to check:**
- Folder structure matches the defined layout (app/, src/api/, src/types/, src/features/, etc.)
- Features are flat — no nested features
- No feature-scoped files imported outside their feature folder
- Screens are thin — no business logic, state, or API calls directly in screen files
- Hook naming matches API function names (`getProducts` → `useGetProducts`)
- Main feature hook exists and composes individual query/mutation hooks
- No `StyleSheet.create` or inline `style` objects — NativeWind className only
- Icons are Hugeicons only — no Expo icons or React Native Vector Icons
- `useSharedValue` / Reanimated used for animation — no legacy `Animated` API
- No barrel exports (`index.ts`)
- No SCREAMING_SNAKE_CASE variables
- No inline functions in JSX
- `export const` for everything except screens (default export only)
- Global components that should be shared are not duplicated per feature

---

## Stack

| Layer | Tool |
|---|---|
| Framework | Expo (managed workflow) |
| Language | TypeScript |
| Navigation | Expo Router (file-based) |
| Styling | NativeWind + React Native Reusables |
| Icons | Hugeicons React Native (`@hugeicons/react-native`) |
| Data Fetching | TanStack Query |
| HTTP Client | Axios with a shared instance |
| Forms | React Hook Form + Zod |
| Global State | Zustand |
| Backend | .NET Web API (same as web projects) |

---

## Project Folder Structure

```
app/                              ← Expo Router file-based routing
├── (auth)/
│   └── login.tsx
├── (tabs)/
│   ├── _layout.tsx
│   ├── index.tsx                 ← Home tab
│   └── profile.tsx
├── _layout.tsx                   ← Root layout
└── +not-found.tsx
src/
├── api/                          ← all axios calls, grouped by domain
│   └── [domain]/
│       └── [Domain]Api.ts        ← e.g. authApi.ts, productsApi.ts
├── types/                        ← DTOs grouped by domain
│   └── [domain]/
│       └── [Domain]DTO.ts        ← e.g. AuthDTO.ts, ProductDTO.ts
├── features/                     ← vertical slice per domain, always flat
│   └── [domain-name]/            ← e.g. auth/, products/, orders/
│       ├── components/           ← sub-components only (not the main page)
│       ├── hooks/
│       │   ├── use[Domain].ts    ← main feature hook
│       │   ├── useGet[Domain]s.ts
│       │   └── useCreate[Domain].ts
│       ├── schemas/              ← Zod schemas for this feature
│       └── [Domain]Page.tsx      ← main page/form component at feature root
├── components/                   ← global shared components
│   └── ui/                       ← React Native Reusables components
├── lib/                          ← third-party config, utilities, shared resources
│   ├── axios.ts                  ← shared axios instance
│   ├── enums/                    ← e.g. Roles.ts, Status.ts
│   ├── helpers/                  ← e.g. formatCurrency.ts
│   └── constants/                ← e.g. apiRoutes.ts
├── store/                        ← Zustand stores
├── hooks/                        ← global shared hooks
└── providers/                    ← QueryProvider, etc.
```

**Feature rules:**
- Features are always flat at the top level — never nest a feature inside another feature
- Each feature is fully self-contained
- Never import a feature-scoped file from outside its feature folder
- If something is needed in 2+ features, move it to `src/components/` or `src/hooks/`

---

## Scaffolding — What to Generate

When the user says "scaffold a new project", "start a new project", or "create a new app":

**Only generate these things — nothing else:**

1. Folder structure (empty folders with `.gitkeep`)
2. `CLAUDE.md` at the project root (see CLAUDE.md Auto-Generation section)
3. `src/lib/axios.ts`
4. `src/providers/QueryProvider.tsx`

**Do NOT generate:** screens, Auth Store, types, login forms, navigation files, or any feature code. Those are written by the developer when needed.

### Folder structure to create

```
app/
├── (auth)/
├── (tabs)/
src/
├── api/
├── types/
├── features/
├── components/
│   └── ui/
├── lib/
│   ├── enums/
│   ├── helpers/
│   └── constants/
├── store/
├── hooks/
└── providers/
```

### `src/lib/axios.ts`

```typescript
import axios from "axios"

const api = axios.create({
  baseURL: process.env.EXPO_PUBLIC_API_URL,
  withCredentials: true,
})

api.interceptors.response.use(
  (response) => response,
  (error) => Promise.reject(error)
)

export default api
```

### `src/providers/QueryProvider.tsx`

```tsx
import { QueryClient, QueryClientProvider } from "@tanstack/react-query"
import { useState } from "react"

export const QueryProvider = ({ children }: { children: React.ReactNode }) => {
  const [queryClient] = useState(() => new QueryClient())

  return (
    <QueryClientProvider client={queryClient}>
      {children}
    </QueryClientProvider>
  )
}
```

### New Feature Scaffold

When the user says "add a new feature" or "create a [name] feature", always create this exact structure. Every item must be created — subfolders as real directories, never flat:

```
src/
├── api/
│   └── [domain]/
│       └── [Domain]Api.ts        ← empty file
├── types/
│   └── [domain]/
│       └── [Domain]DTO.ts        ← empty file
└── features/
    └── [domain]/
        ├── components/           ← empty folder (.gitkeep)
        ├── hooks/                ← empty folder (.gitkeep)
        └── schemas/              ← empty folder (.gitkeep)
```

**Rules:**
- `api/[domain]/`, `types/[domain]/`, `components/`, `hooks/`, `schemas/` are always real folders — never skip them
- The main page component (e.g. `LoginForm.tsx`) is added by the developer at the feature root — do NOT generate it on scaffold
- Do NOT generate any code inside these files — leave them empty
- Do NOT add any extra files beyond what is listed above

---

## Screen Standards

### Screen Boilerplate

Every screen file uses this structure:

```tsx
import { View, ScrollView } from "react-native"

export default function HomeScreen() {
  return (
    <ScrollView className="flex-1 bg-background">
      <View className="flex-1 p-4">
        {/* content */}
      </View>
    </ScrollView>
  )
}
```

- PascalCase + `Screen` suffix matching the route (`HomeScreen`, `ProfileScreen`)
- Default exports for screens only — everything else uses `export const`
- Never put business logic, state, or API calls directly in the screen
- Keep screens thin — delegate to feature components

---

## State Management Pattern

### Hook — owns all state and logic

```typescript
// src/features/products/hooks/useProducts.ts
import { useState } from "react"
import { useGetProducts } from "./useGetProducts"
import { useCreateProduct } from "./useCreateProduct"

export const useProducts = () => {
  const [searchKey, setSearchKey] = useState("")
  const [selectedId, setSelectedId] = useState<string | null>(null)
  const [isModalVisible, setIsModalVisible] = useState(false)

  const { data, isLoading } = useGetProducts({ searchKey })
  const { mutate: createProduct } = useCreateProduct()

  const handleOpenModal = (id: string) => {
    setSelectedId(id)
    setIsModalVisible(true)
  }

  const handleCloseModal = () => {
    setSelectedId(null)
    setIsModalVisible(false)
  }

  return {
    data,
    isLoading,
    searchKey,
    setSearchKey,
    selectedId,
    isModalVisible,
    handleOpenModal,
    handleCloseModal,
    createProduct,
  }
}
```

### Zustand — global state only

Use Zustand only for state that needs to persist across screens or the entire app — auth, user session, app-wide settings. Never use Zustand for feature-level state:

```typescript
// src/store/useAuthStore.ts — global, lives in store/
// src/features/products/hooks/useProducts.ts — feature state, lives in hooks/
```

---

## TanStack Query Hook Naming

Same convention as the web stack — hook name mirrors the API call exactly:

```
src/features/products/hooks/
├── useGetProducts.ts       ← mirrors getProducts
├── useGetProductById.ts    ← mirrors getProductById
├── useCreateProduct.ts     ← mirrors createProduct
├── useUpdateProduct.ts     ← mirrors updateProduct
├── useDeleteProduct.ts     ← mirrors deleteProduct
└── useProducts.ts          ← main feature hook, composes the above
```

**Query hook:**

```typescript
// src/features/products/hooks/useGetProducts.ts
import { useQuery } from "@tanstack/react-query"
import { getProducts } from "../api"
import { GetProductsParams } from "../types"

export const useGetProducts = (params: GetProductsParams) => {
  return useQuery({
    queryKey: ["products", params],
    queryFn: () => getProducts(params),
  })
}
```

**Mutation hook:**

```typescript
// src/features/products/hooks/useCreateProduct.ts
import { useMutation, useQueryClient } from "@tanstack/react-query"
import { createProduct } from "../api"

export const useCreateProduct = () => {
  const queryClient = useQueryClient()

  return useMutation({
    mutationFn: createProduct,
    onSuccess: () => {
      queryClient.invalidateQueries({ queryKey: ["products"] })
    },
  })
}
```

---

## Component Standards

### Global vs Feature-scoped

| Scope | Location |
|---|---|
| Used in 2+ unrelated features | `src/components/` |
| Used only in one feature | `src/features/[domain]/components/` |
| React Native Reusables base components | `src/components/ui/` |

### Reusable Component Checklist

Before creating any new component, ask in order:

1. **Does it already exist in `src/components/`?** → use it
2. **Will this be used in more than one feature?** → build reusable, put in `src/components/` from the start
3. **Only used in this feature?** → put in `src/features/[domain]/components/`

**Components that must always be global:**

| Component | Why |
|---|---|
| `SearchInput` | Every list screen has one |
| `EmptyState` | Every list screen needs it |
| `LoadingSpinner` | Used everywhere |
| `ConfirmModal` | Delete confirmations everywhere |
| `PageHeader` | Every screen has a title |

### Component Boilerplate

```tsx
import { View, Text } from "react-native"

interface ProductCardProps {
  name: string
  price: number
  onPress: () => void
}

export const ProductCard = ({ name, price, onPress }: ProductCardProps) => {
  return (
    <View className="rounded-lg border border-border bg-card p-4">
      <Text className="text-foreground font-medium">{name}</Text>
      <Text className="text-muted-foreground">{price}</Text>
    </View>
  )
}
```

---

## Icons

Always use Hugeicons React Native — never Expo icons or React Native Vector Icons:

```tsx
import { HugeiconsIcon } from "@hugeicons/react-native"
import { SearchIcon } from "@hugeicons/core-free-icons"

<HugeiconsIcon icon={SearchIcon} size={20} color="black" />
```

---

## Forms (React Hook Form + Zod)

```tsx
import { useForm, Controller } from "react-hook-form"
import { zodResolver } from "@hookform/resolvers/zod"
import { z } from "zod"
import { TextInput, View, Pressable, Text } from "react-native"

const schema = z.object({
  email: z.string().email(),
  password: z.string().min(8),
})

type FormValues = z.infer<typeof schema>

export const LoginForm = () => {
  const { control, handleSubmit, formState: { errors } } = useForm<FormValues>({
    resolver: zodResolver(schema),
  })

  const onSubmit = (data: FormValues) => {
    // api call
  }

  return (
    <View className="flex flex-col gap-4">
      <Controller
        control={control}
        name="email"
        render={({ field: { onChange, value } }) => (
          <TextInput
            value={value}
            onChangeText={onChange}
            placeholder="Email"
            className="border border-border rounded-lg p-3"
          />
        )}
      />
      {errors.email && (
        <Text className="text-destructive text-sm">{errors.email.message}</Text>
      )}
      <Pressable onPress={handleSubmit(onSubmit)} className="bg-primary rounded-lg p-3">
        <Text className="text-primary-foreground text-center">Login</Text>
      </Pressable>
    </View>
  )
}
```

---

## Navigation (Expo Router)

Same mental model as Next.js App Router — folder = route:

```
app/
├── (auth)/               ← auth group, no tab bar
│   └── login.tsx         → /login
├── (tabs)/               ← tab group
│   ├── _layout.tsx       ← tab bar config
│   ├── index.tsx         → / (Home tab)
│   └── profile.tsx       → /profile
└── _layout.tsx           ← root layout
```

**Navigating:**

```tsx
import { router } from "expo-router"

const handleNavigate = () => {
  router.push("/profile")
}

const handleBack = () => {
  router.back()
}
```

---

## Naming Conventions

| Type | Convention | Example |
|---|---|---|
| Screens | PascalCase + `Screen` suffix | `HomeScreen`, `ProductListScreen` |
| Components | PascalCase | `ProductCard`, `SearchInput` |
| Hooks | camelCase + `use` prefix | `useProducts`, `useGetProducts` |
| Stores | camelCase + `use` prefix + `Store` suffix | `useAuthStore` |
| Utilities | camelCase | `formatCurrency.ts` |
| Variables & constants | camelCase | `maxRetryCount`, `apiRoutes` |
| Enums | PascalCase | `Roles.ts` |
| Types/Interfaces | PascalCase | `User`, `ApiResponse` |

- camelCase for everything — no SCREAMING_SNAKE_CASE anywhere
- `export const` for all non-screen components, hooks, stores, utilities
- Default exports for screens only

---

## General Code Quality

- No inline comments — code should be self-explanatory, never narrate what the code is doing
- No logic duplication — extract to hooks or helpers
- No inline functions in JSX — always extract to named handlers
- Keep screens thin — all logic in hooks
- Use named constants, never magic values
- No barrel exports (`index.ts`) — import directly from the file

---

## CLAUDE.md Auto-Generation

When the user says "scaffold a new project", "start a new project", or "create a new app", always auto-generate a `CLAUDE.md` file at the root of the project.

### Step 1 — Ask these questions first

1. What is the app name?
2. What does it do and who is it for?
3. What roles does it have? (e.g. Admin, User, Guest)
4. What are the key features/screens?
5. Any environment variables needed beyond the defaults?

### Step 2 — Generate filled CLAUDE.md

```markdown
# [App Name]

## What is this project
[Filled from user answer]

## Stack
- Framework: Expo (managed workflow)
- Navigation: Expo Router
- Styling: NativeWind + React Native Reusables
- Icons: Hugeicons React Native
- Data fetching: TanStack Query
- HTTP: Axios
- Forms: React Hook Form + Zod
- Global state: Zustand
- Backend: .NET Web API

## Roles
- [Filled from user answer]

## Key Features / Screens
- [Filled from user answer]

## Folder Structure
[Generate based on scaffolded structure]

## Environment Variables (.env)
\```
EXPO_PUBLIC_API_URL=
\```

## Notes
[Any project-specific rules or context]
```
