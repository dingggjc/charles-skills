# agent-skills

Personal Claude skill collection for consistent coding standards across all projects.

## Skills

| Skill | Stack | Description |
|---|---|---|
| `frontend-nextjs` | Next.js, TypeScript, shadcn/ui, TanStack Query | Standards and scaffold guide for Next.js web projects |
| `backend-dotnet` | .NET 10, Clean Architecture, MediatR, EF Core | Standards and scaffold guide for .NET Web API projects |
| `react-native-expo` | Expo, NativeWind, React Native Reusables, TanStack Query | Standards and scaffold guide for React Native mobile apps |
| `design-taste-mobile` | Expo, NativeWind, Reanimated, React Native Reusables | Anti-slop mobile design skill — the taste layer for React Native screens (mobile counterpart of tasteskill.dev) |

---

## Usage

### Install a skill

```bash
# Frontend
npx skills add github:dingggjc/charles-skills/frontend-nextjs

# Backend
npx skills add github:dingggjc/charles-skills/backend-dotnet

# React Native
npx skills add github:dingggjc/charles-skills/react-native-expo

# Mobile design taste
npx skills add github:dingggjc/charles-skills/design-taste-mobile
```

### Install from local clone

```bash
git clone https://github.com/dingggjc/charles-skills.git

npx skills add ./charles-skills/frontend-nextjs
npx skills add ./charles-skills/backend-dotnet
npx skills add ./charles-skills/react-native-expo
```

---

## How it works

1. Install the relevant skill for your project
2. Say "scaffold a new project called X"
3. Claude asks you — name, description, roles, key features
4. Claude generates the full folder structure + a filled `CLAUDE.md` automatically
5. Every session after, Claude Code reads `CLAUDE.md` + the skill — no manual prompting needed

Use `CLAUDE_TEMPLATE.md` at the root as a reference for what gets generated.

---

## What each skill does

### `frontend-nextjs`
- Scaffolds full Next.js App Router project structure
- Enforces vertical slice feature architecture
- Standards for components, hooks, context, tables, modals, forms
- TanStack Query hook naming conventions (`useGetUsers` mirrors `getUsers`)
- Zod schema placement rules (`features/[domain]/schemas/`)
- Reusable component checklist
- Auto-generates filled `CLAUDE.md` on project scaffold

### `backend-dotnet`
- Scaffolds Clean Architecture .NET solution
- CQRS with MediatR — commands, queries, handlers co-located
- `CommandResult` / `QueryPageResult` response pattern
- `ICurrentUserService` — never pass UserId/UserRole as command params
- FluentValidation per feature
- Controller standards — only `IMediator`, no business logic
- Auto-generates filled `CLAUDE.md` on project scaffold

### `react-native-expo`
- Scaffolds Expo managed workflow project structure
- Expo Router file-based navigation
- NativeWind + React Native Reusables theming
- Same vertical slice feature architecture as web
- Zustand for global state, TanStack Query for server state
- Hugeicons React Native for icons
- Auto-generates filled `CLAUDE.md` on project scaffold

### `design-taste-mobile`
- Anti-slop design skill for React Native / Expo screens — mobile counterpart of [tasteskill.dev](https://www.tasteskill.dev/)
- Brief inference + one-line Design Read before any code
- Three dials (variance / motion / density) tuned per screen type
- Design language map — HIG-native vs Material 3 vs brand-first
- Reanimated motion playbook with canonical skeletons (collapsing header, press-scale, stagger, swipe-dismiss, skeleton pulse)
- Mobile AI-tell bans (Poppins, gradient CTAs, greeting-bell headers, 5-tab defaults, em-dashes) + strict pre-flight checklist
- Commands: `new`, `redesign`, `audit`, `palette`, `motion`, `check`
- Companion to `react-native-expo` — that skill owns architecture, this one owns taste

---

## Standards that apply to all skills

- No inline comments — code speaks for itself
- No inline functions in JSX/TSX
- camelCase for everything — no SCREAMING_SNAKE_CASE
- `export const` for all non-page/screen components, hooks, utilities
- Default exports for pages/screens only
- No barrel exports (`index.ts`) — import directly from file
- Feature-specific schemas → `features/[domain]/schemas/`
- Shared schemas → `lib/schemas/`
- `lib/` is strictly — axios config, enums, helpers, constants, shared schemas

---

## Stack Reference

### Web (Next.js)
- Framework: Next.js 15 App Router
- Styling: Tailwind CSS + shadcn/ui
- Icons: Hugeicons (`@hugeicons/react`)
- Data fetching: TanStack Query
- HTTP: Axios
- Forms: React Hook Form + Zod
- Auth: Cookie-based middleware

### Backend (.NET)
- Framework: .NET 10, ASP.NET Core Web API
- Architecture: Clean Architecture
- CQRS: MediatR
- ORM: EF Core + PostgreSQL (Neon)
- Validation: FluentValidation
- Auth: JWT

### Mobile (React Native)
- Framework: Expo (managed workflow)
- Navigation: Expo Router
- Styling: NativeWind + React Native Reusables
- Icons: Hugeicons React Native (`@hugeicons/react-native`)
- Data fetching: TanStack Query
- HTTP: Axios
- Forms: React Hook Form + Zod
- Global state: Zustand
