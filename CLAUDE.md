# AI Workflow Hub — CLAUDE.md

## Descripción del Proyecto
SaaS multi-tenant para automatización de workflows con IA. Permite a agencias y empresas crear, ejecutar y monitorear flujos de trabajo inteligentes con nodos de IA (Claude, OpenAI). Arquitectura limpia con dominio, aplicación e infraestructura separados.

## Stack
| Capa | Tecnología |
|------|-----------|
| Framework | Next.js 14 (App Router) |
| Lenguaje | TypeScript |
| API | tRPC (type-safe RPC) |
| Auth | NextAuth v5 + @auth/prisma-adapter |
| DB | PostgreSQL (multi-tenant con schemas) + Prisma ORM |
| Cache | Redis (ioredis) + Socket.IO Redis adapter |
| Realtime | Socket.IO (servidor custom en `src/server/`) |
| AI | Vercel AI SDK (`ai`, `@ai-sdk/anthropic`, `@ai-sdk/openai`) |
| Estado | Zustand + React Query (TanStack) + Immer |
| UI | Radix UI + Tailwind CSS + CVA + lucide-react |
| Forms | React Hook Form + Zod |
| Visualización | ReactFlow (workflow canvas) + Recharts (dashboards) |
| Billing | Stripe |
| Email | Resend |
| Logging | Winston |
| Testing | Vitest + Playwright |

## Arquitectura — Clean Architecture Multi-tenant
```
src/
├── app/
│   ├── (admin)/          # Rutas del panel admin (superadmin)
│   ├── (app)/            # Rutas del app por tenant
│   ├── (auth)/           # Login, registro, verificación
│   └── (public)/         # Landing, pricing, docs públicos
│   api/                  # API routes (webhooks Stripe, etc.)
├── domain/               # Entidades, value objects, eventos de dominio
│   ├── entities/
│   ├── value-objects/
│   ├── events/
│   └── interfaces/       # Repositorio interfaces (ports)
├── application/          # Casos de uso, DTOs, servicios de app
│   ├── use-cases/
│   ├── dtos/
│   └── services/
├── infrastructure/       # Implementaciones concretas
│   ├── db/               # Prisma client
│   ├── cache/            # Redis
│   ├── repositories/     # Implementaciones de repositorios
│   └── services/         # AI, email, billing adapters
├── server/               # tRPC router + Socket.IO server
│   ├── trpc/
│   ├── ai/
│   ├── auth/
│   ├── billing/
│   ├── realtime/
│   └── workflow-engine/
├── components/           # Componentes React reutilizables
├── hooks/                # Custom hooks
├── types/                # TypeScript types globales
└── middleware.ts          # Auth + tenant resolution
```

## Multi-tenancy
- Cada tenant tiene su propio **schema de PostgreSQL** (`dbSchema` en tabla `tenants`)
- `tenant-schema.sql` en `/prisma/` define las tablas por tenant
- El middleware resuelve el tenant por dominio o slug antes de cada request
- Nunca mezclar datos entre schemas — usar `SET search_path = tenant_schema`

## Prisma DB Schemas
```
prisma/
├── schema.prisma       # Schema público: users, tenants, accounts, subscriptions
├── seed.ts             # Datos de desarrollo
└── tenant-schema.sql   # Schema por tenant (workflows, nodes, executions, etc.)
```

## Modelos principales (schema público)
- `User` — autenticación, OAuth
- `Tenant` — organización, plan, schema DB, Stripe
- `SubscriptionStatus` — TRIALING | ACTIVE | PAST_DUE | CANCELED

## Comandos
```bash
# Dev (Next.js + Socket.IO server)
npm run dev          # http://localhost:3000

# Build
npm run build

# Prisma
npx prisma migrate dev     # Nueva migración
npx prisma generate        # Regenerar client
npx prisma studio          # GUI DB
npx prisma db seed         # Seed datos

# Tests unitarios
npm run test               # Vitest

# Tests E2E
npx playwright test

# Tests integración
npm run test:integration
```

## Variables de Entorno (.env.local)
```
DATABASE_URL=postgresql://...
NEXTAUTH_SECRET=...
NEXTAUTH_URL=http://localhost:3000
GOOGLE_CLIENT_ID=...
GOOGLE_CLIENT_SECRET=...
STRIPE_SECRET_KEY=sk_test_...
STRIPE_WEBHOOK_SECRET=whsec_...
NEXT_PUBLIC_STRIPE_PUBLISHABLE_KEY=pk_test_...
REDIS_URL=redis://localhost:6379
ANTHROPIC_API_KEY=sk-ant-...
OPENAI_API_KEY=sk-...
RESEND_API_KEY=re_...
NEXT_PUBLIC_APP_URL=http://localhost:3000
```

## Convenciones
- **Domain** → sin dependencias de frameworks. Puras clases TypeScript
- **Application** → orquesta domain + llama a infraestructura via interfaces
- **Infrastructure** → implementa las interfaces del domain
- tRPC routers en `src/server/trpc/` — un archivo por feature
- React Query para todo fetching del cliente — no `useEffect` + `fetch`
- Zustand para estado global UI — Immer para mutaciones complejas
- `cn()` de `lib/utils.ts` para clases Tailwind
- Stripe webhooks → `src/app/api/stripe/webhook/route.ts`

## GitHub
Repo: `cdgutierrez6/ai-workflow-hub` (crear si no existe)
Branch default: `main`

## Estado actual
En desarrollo activo — arquitectura base completa, workflows y billing en progreso.

## Reglas de trabajo
1. Cambios de schema DB → pipeline 🟠 DB obligatorio antes de tocar prisma/
2. Nuevos casos de uso → empezar siempre desde `domain/` hacia arriba
3. Nunca importar Prisma client desde componentes — solo desde infrastructure/
4. Multi-tenancy → validar tenant en middleware, nunca en lógica de negocio
5. Stripe webhooks → siempre verificar firma con `stripe.webhooks.constructEvent()`
