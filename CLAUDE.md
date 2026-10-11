# CLAUDE.md — ProjectX: Complete Build Reference

This file is the single source of truth for this project.
It contains everything needed to continue building after a chat session is deleted.
Read this fully before exploring code, planning changes, or writing any new feature.

---

## What This Project Is

**ProjectX** — a multi-tenant B2B textile wholesaler SaaS platform built on Cloudflare's edge stack.

Textile wholesalers use this to manage product catalogs and share them with retailers
via rotating storefront links. Three distinct user roles exist:

| Role | Entry Point | Purpose |
|---|---|---|
| Master Admin | `/master-admin` | Owner-only control center — manage wholesaler accounts, inspect DB, view health |
| Wholesaler | `/dashboard` | Upload catalogs via Excel, manage product images, review/approve orders |
| Retailer | `/[token]` | Browse a wholesaler's catalog on mobile, place orders |

---

## Tech Stack

| Layer | Technology | Why |
|---|---|---|
| Frontend | Next.js 14 App Router → **Cloudflare Pages** | SSR + edge rendering |
| API Backend | **Cloudflare Workers** + **Hono** web framework | Full D1/KV/R2 bindings, runs at edge |
| Database | **Cloudflare D1** (SQLite) via **Drizzle ORM** | Serverless SQL at the edge |
| File Storage | **Cloudflare R2** | Zero egress cost, 3 image slots per product |
| OTP / Rate-limit | **Cloudflare KV** | Fast ephemeral key-value for OTP TTL + rate counters |
| Styling | Tailwind CSS + shadcn/ui | Rapid component-based UI |
| Monorepo | pnpm workspaces + Turborepo | Shared schema package between frontend and backend |

**Why a separate Worker instead of Next.js Route Handlers?**
`@cloudflare/next-on-pages` cannot reliably access D1/KV/R2 bindings from Route Handlers.
The standalone Worker has full access to all Cloudflare bindings and deploys independently.

**Why parse Excel client-side?**
Avoids uploading raw binary files to the Worker. The wholesaler previews and corrects data
in a table before submitting clean JSON rows.

**Why HttpOnly cookies for JWTs?**
JWTs in localStorage are vulnerable to XSS. HttpOnly cookies are inaccessible to JavaScript
and are the correct storage for auth tokens.

---

## Monorepo Structure

```
wholesaler-dashboard/
├── CLAUDE.md               ← this file — full build reference
├── SECURITY.md             ← security model, auth flows, AI coding rules
├── pnpm-workspace.yaml
├── turbo.json
├── .gitignore              # must include: .env, .env.local, .dev.vars
├── apps/
│   ├── web/                # Next.js 14 frontend → Cloudflare Pages
│   │   ├── package.json
│   │   ├── next.config.mjs
│   │   ├── tailwind.config.ts
│   │   ├── components.json     # shadcn/ui config
│   │   └── src/
│   │       ├── app/            # see folder structure below
│   │       ├── components/
│   │       │   ├── ui/             # shadcn primitives (auto-generated)
│   │       │   ├── master-admin/
│   │       │   ├── wholesaler/
│   │       │   └── retailer/
│   │       ├── lib/
│   │       │   ├── api-client.ts   # typed fetch wrapper (sets credentials: "include")
│   │       │   ├── auth.ts         # reads meta cookie for UI display
│   │       │   └── utils.ts
│   │       ├── hooks/
│   │       │   ├── use-products.ts
│   │       │   └── use-orders.ts
│   │       └── stores/
│   │           ├── auth.ts         # Zustand: role + businessName from meta cookie
│   │           ├── retailer.ts     # Zustand: name, shopName, phone (localStorage)
│   │           └── cart.ts         # Zustand: cart items (sessionStorage)
│   │
│   └── worker/             # Hono API → Cloudflare Workers
│       ├── package.json
│       ├── wrangler.toml
│       ├── tsconfig.json
│       └── src/
│           ├── index.ts        # Hono app entry, CORS, error handler
│           ├── middleware/
│           │   ├── auth.ts     # verifyJwt(role) factory — reads HttpOnly cookie
│           │   └── link.ts     # token expiry check for retailer routes
│           ├── routes/
│           │   ├── admin.ts
│           │   ├── wholesaler.ts
│           │   ├── products.ts
│           │   ├── orders.ts
│           │   └── links.ts
│           ├── lib/
│           │   ├── otp.ts      # KV OTP: generate, store, verify, delete
│           │   ├── twilio.ts   # SMS dispatch
│           │   ├── r2.ts       # presigned PUT URL generation
│           │   └── jwt.ts      # sign/verify with jose
│           └── scheduled.ts    # cron: rotate dynamic_links daily
│
└── packages/
    └── db/                 # Shared Drizzle schema + inferred types + Zod validators
        ├── package.json
        └── src/
            ├── schema.ts   # all table definitions (source of truth)
            ├── types.ts    # InferSelectModel exports
            └── index.ts    # re-exports everything
```

---

## App Router Folder Structure (apps/web/src/app)

```
app/
├── layout.tsx
├── page.tsx                        # redirects to /master-admin or /dashboard
│
├── master-admin/
│   ├── login/
│   │   └── page.tsx                # Step 1: username+password; Step 2: OTP modal
│   └── (protected)/
│       ├── layout.tsx              # auth guard: reads meta cookie, redirects if missing
│       ├── page.tsx                # stats overview (wholesaler count, active links, orders)
│       ├── wholesalers/
│       │   └── page.tsx            # DataTable + CreateWholesalerDialog
│       ├── d1-explorer/
│       │   └── page.tsx            # SQL textarea + ResultTable
│       └── health/
│           └── page.tsx            # Worker status + D1 metrics cards
│
├── dashboard/
│   ├── login/
│   │   └── page.tsx
│   └── (protected)/
│       ├── layout.tsx              # wholesaler auth guard
│       ├── page.tsx                # ProductGrid with inline image upload
│       ├── upload/
│       │   └── page.tsx            # ExcelDropzone → PreviewTable → POST
│       ├── orders/
│       │   └── page.tsx            # tab-filtered OrderCards with approve/stockout
│       └── settings/
│           └── page.tsx            # ChangePasswordForm + LinkManager
│
└── [token]/
    ├── layout.tsx                  # Server Component: verify token, notFound() if expired
    └── page.tsx                    # StorefrontPage: catalog + WhatsApp modal + CartFab
```

---

## NPM Packages

### apps/web
| Package | Purpose |
|---|---|
| `next@^14.2`, `react@^18.3`, `react-dom` | Framework |
| `@cloudflare/next-on-pages` | Next.js → Cloudflare Pages adapter |
| `tailwindcss@^3.4`, `postcss`, `autoprefixer` | Styling |
| shadcn/ui (CLI-installed) + `@radix-ui/*` | Component primitives |
| `class-variance-authority`, `clsx`, `tailwind-merge` | Class utilities |
| `lucide-react` | Icons |
| `react-hook-form` + `@hookform/resolvers` + `zod` | Forms + validation |
| `xlsx` | Client-side Excel parse (no binary upload to server) |
| `@tanstack/react-query@^5` | Server state — products, orders |
| `zustand@^4` | Client state — cart, retailer identity, auth meta |
| `jose` | JWT decode on client (for meta cookie only) |
| `sonner` | Toast notifications |

### apps/worker
| Package | Purpose |
|---|---|
| `hono@^4` | Web framework for Cloudflare Workers |
| `@hono/zod-validator` | Route-level Zod validation middleware |
| `drizzle-orm@^0.30` | D1 query builder |
| `@cloudflare/workers-types` | TypeScript types for D1, KV, R2 bindings |
| `zod` | Schema validation |
| `jose` | JWT sign/verify (Web Crypto, works in Workers) |
| `wrangler@^3` (devDependency) | Local dev server + deploy CLI |

### packages/db
| Package | Purpose |
|---|---|
| `drizzle-orm@^0.30` | Schema definitions |
| `drizzle-kit@^0.21` (devDependency) | Migration file generation |
| `drizzle-zod` | Auto-generate Zod validators from Drizzle tables |

---

## Complete Database Schema (packages/db/src/schema.ts)

```typescript
import { sqliteTable, text, integer, real } from "drizzle-orm/sqlite-core";
import { sql } from "drizzle-orm";

export const wholesalers = sqliteTable("wholesalers", {
  id:             integer("id").primaryKey({ autoIncrement: true }),
  username:       text("username").notNull().unique(),
  passwordHash:   text("password_hash").notNull(),
  businessName:   text("business_name").notNull(),
  whatsappNumber: text("whatsapp_number").notNull(),
  status:         text("status", { enum: ["active", "suspended"] }).notNull().default("active"),
  createdAt:      integer("created_at", { mode: "timestamp" }).notNull().default(sql`(unixepoch())`),
});

export const products = sqliteTable("products", {
  id:             integer("id").primaryKey({ autoIncrement: true }),
  wholesalerId:   integer("wholesaler_id").notNull().references(() => wholesalers.id, { onDelete: "cascade" }),
  itemCode:       text("item_code").notNull(),
  hsnCode:        text("hsn_code"),
  material:       text("material"),
  mrpPrice:       real("mrp_price").notNull(),
  wholesalePrice: real("wholesale_price").notNull(),
  image1:         text("image_1"),          // R2 object key
  image2:         text("image_2"),          // R2 object key
  image3:         text("image_3"),          // R2 object key
  clothType:      text("cloth_type"),       // e.g. kurti, saree, leggings, dupatta
  description:    text("description"),      // free text: slitted/unslitted/2-piece/3-piece/etc.
  isInStock:      integer("is_in_stock", { mode: "boolean" }).notNull().default(true),
  createdAt:      integer("created_at", { mode: "timestamp" }).notNull().default(sql`(unixepoch())`),
});

export const dynamicLinks = sqliteTable("dynamic_links", {
  id:           integer("id").primaryKey({ autoIncrement: true }),
  wholesalerId: integer("wholesaler_id").notNull().references(() => wholesalers.id, { onDelete: "cascade" }),
  token:        text("token").notNull().unique(),   // 32-char hex from crypto.randomUUID()
  expiresAt:    integer("expires_at", { mode: "timestamp" }).notNull(),  // createdAt + 3 days
  createdAt:    integer("created_at", { mode: "timestamp" }).notNull().default(sql`(unixepoch())`),
});

export const retailers = sqliteTable("retailers", {
  id:             integer("id").primaryKey({ autoIncrement: true }),
  name:           text("name").notNull(),
  shopName:       text("shop_name"),
  phoneNumber:    text("phone_number").notNull(),
  whatsappNumber: text("whatsapp_number").notNull(),
  createdAt:      integer("created_at", { mode: "timestamp" }).notNull().default(sql`(unixepoch())`),
});

export const orders = sqliteTable("orders", {
  id:           integer("id").primaryKey({ autoIncrement: true }),
  wholesalerId: integer("wholesaler_id").notNull().references(() => wholesalers.id),
  retailerId:   integer("retailer_id").notNull().references(() => retailers.id),
  totalAmount:  real("total_amount").notNull().default(0),
  status:       text("status", { enum: ["pending", "approved", "partially_approved"] }).notNull().default("pending"),
  createdAt:    integer("created_at", { mode: "timestamp" }).notNull().default(sql`(unixepoch())`),
});

export const orderItems = sqliteTable("order_items", {
  id:        integer("id").primaryKey({ autoIncrement: true }),
  orderId:   integer("order_id").notNull().references(() => orders.id, { onDelete: "cascade" }),
  productId: integer("product_id").notNull().references(() => products.id),
  quantity:  integer("quantity").notNull(),
  unitPrice: real("unit_price").notNull(),
  status:    text("status", { enum: ["approved", "stockout"] }).notNull().default("approved"),
});
```

### Index Recommendations (add to migration SQL)
```sql
CREATE INDEX idx_products_wholesaler ON products(wholesaler_id);
CREATE INDEX idx_dynamic_links_wholesaler ON dynamic_links(wholesaler_id);
CREATE INDEX idx_retailers_whatsapp ON retailers(whatsapp_number);
CREATE INDEX idx_orders_wholesaler_status ON orders(wholesaler_id, status);
-- dynamic_links.token already has UNIQUE constraint (implicit index)
```

---

## API Route Specification (apps/worker)

| Method | Path | Auth Required | Purpose |
|---|---|---|---|
| `GET` | `/api/link/:token` | None | Verify token exists + not expired; return wholesalerId or 404 |
| `POST` | `/api/admin/login` | None | Verify credentials, generate OTP, store in KV (5-min TTL), send via Twilio |
| `POST` | `/api/admin/verify-otp` | None | Check KV OTP, delete it (single-use), issue Admin JWT HttpOnly cookie |
| `GET` | `/api/admin/wholesalers` | Admin JWT | List all wholesalers |
| `POST` | `/api/admin/wholesalers` | Admin JWT | Create new wholesaler account |
| `PATCH` | `/api/admin/wholesalers/:id` | Admin JWT | Update wholesaler (status, password reset) |
| `DELETE` | `/api/admin/wholesalers/:id` | Admin JWT | Delete wholesaler + cascade products/links |
| `POST` | `/api/admin/d1/query` | Admin JWT | Run SQL against D1 (read-only by default) |
| `GET` | `/api/admin/health` | Admin JWT | D1 ping + KV ping + metrics |
| `POST` | `/api/wholesaler/login` | None | Verify password, issue Wholesaler JWT HttpOnly cookie |
| `PUT` | `/api/wholesaler/password` | Wholesaler JWT | Change password (verify current, hash new, update D1) |
| `GET` | `/api/wholesaler/links` | Wholesaler JWT | List this wholesaler's dynamic links |
| `POST` | `/api/wholesaler/links` | Wholesaler JWT | Generate a new 3-day token link |
| `POST` | `/api/products/upload` | Wholesaler JWT | Bulk insert parsed Excel rows; D1 batch in chunks of 50 |
| `GET` | `/api/products/upload-url` | Wholesaler JWT | Return R2 presigned PUT URL for image slot (15-min expiry) |
| `PATCH` | `/api/products/:id` | Wholesaler JWT | Update any product field incl. `isInStock=false` for stockout |
| `DELETE` | `/api/products/:id` | Wholesaler JWT | Delete product |
| `GET` | `/api/products` | Wholesaler JWT or valid token | List products filtered by wholesalerId; in-stock only for retailers |
| `POST` | `/api/orders` | Valid token (retailer route) | Create order + order_items; retailer data in body |
| `GET` | `/api/orders` | Wholesaler JWT | List all orders with nested items |
| `PUT` | `/api/orders/:id/items/:itemId/status` | Wholesaler JWT | Set item status (approved/stockout); recalculate order total_amount |
| `PATCH` | `/api/orders/:id/status` | Wholesaler JWT | Finalize order-level status |

**Scheduled Cron (`apps/worker/src/scheduled.ts`):**
- Trigger: `0 2 * * *` (2am UTC daily)
- Deletes all `dynamic_links` where `expires_at < unixepoch()`
- For each active wholesaler whose current link expires within 24h: generates a new token (`crypto.randomUUID().replace(/-/g,"")`), inserts with `expires_at = unixepoch() + 259200` (3 days)

---

## Component Trees

### Master Admin (`/master-admin`)
```
MasterAdminLayout
├── Sidebar (Dashboard | Wholesalers | D1 Explorer | Health)
├── TopBar (logout button)
│
├── LoginPage
│   ├── LoginForm (username + password fields)
│   └── OtpModal (6-digit input, 5-min countdown, resend button)
│
├── DashboardPage
│   └── StatsGrid (wholesaler count, active links, pending orders today)
│
├── WholesalersPage
│   ├── WholesalerDataTable (shadcn DataTable)
│   │   └── columns: businessName, username, status badge, whatsapp, created, actions
│   └── CreateWholesalerDialog
│       └── form: username, password, businessName, whatsappNumber
│
├── D1ExplorerPage
│   ├── SqlEditor (textarea, "Allow Writes" toggle, Run button)
│   └── ResultTable (dynamic columns from query result)
│
└── HealthPage
    ├── WorkerStatusCard (ping latency)
    └── D1MetricsCard (row counts per table)
```

### Wholesaler Dashboard (`/dashboard`)
```
WholesalerLayout
├── Sidebar (Products | Upload | Orders | Settings)
├── TopBar (businessName, logout)
│
├── LoginPage → LoginForm
│
├── ProductsPage (default view)
│   ├── ProductToolbar (search by item code, filter: all/in-stock/out-of-stock)
│   └── ProductGrid (React Query paginated)
│       └── ProductRow (one row per product)
│           ├── ImageSlots × 3
│           │   ├── Thumbnail preview (if image exists)
│           │   ├── DropZone (drag & drop or click to upload)
│           │   └── Upload flow: GET presigned URL → PUT to R2 → PATCH product with key
│           ├── Fields: itemCode, hsnCode, material, clothType, mrpPrice, wholesalePrice, description
│           └── StockoutToggle (switches isInStock; syncs to D1 immediately)
│
├── UploadPage
│   ├── ExcelDropzone (xlsx.js parses file client-side)
│   ├── PreviewTable (editable parsed rows before submit)
│   └── UploadButton → POST /api/products/upload
│
├── OrdersPage
│   ├── StatusTabs (Pending | Approved | Partially Approved)
│   └── OrderList
│       └── OrderCard
│           ├── RetailerInfo (name, shop, phone)
│           ├── OrderItemTable
│           │   └── OrderItemRow: image thumb, itemCode, clothType, qty, price, [Approve] [Stockout]
│           ├── OrderTotal (auto-recalculates as items change)
│           └── FinalizeButton → PATCH /api/orders/:id/status
│
└── SettingsPage
    ├── ChangePasswordForm (currentPassword, newPassword, confirm)
    └── LinkManager
        ├── CurrentLinkDisplay (token URL, copy button, expiry countdown)
        └── GenerateNewLinkButton → POST /api/wholesaler/links
```

### Retailer Storefront (`/[token]`)
```
[token]/layout.tsx  (Server Component)
└── validates token via fetch to Worker → notFound() if expired/invalid

StorefrontPage
├── BusinessHeader (wholesaler businessName)
├── WhatsAppLoginModal  (shown on first visit, Zustand persisted)
│   └── fields: name, shopName, phoneNumber, whatsappNumber → stored in retailer store
├── ProductCatalog
│   ├── FilterBar (by clothType)
│   └── ProductCard × N
│       ├── ImageCarousel (image1/2/3)
│       ├── clothType badge + itemCode
│       ├── description text
│       ├── MRP (strikethrough) vs wholesale price
│       └── QuantitySelector + "Add to Cart" button
└── CartFab (floating, shows item count) → opens CheckoutDrawer

CheckoutDrawer
├── CartItemList (item rows with editable quantity, remove)
├── OrderSummary (subtotal, item count)
└── PlaceOrderButton → POST /api/orders
    └── SuccessModal
        ├── Order confirmation message
        └── WhatsApp CTA (deep link to wholesaler's WhatsApp)
```

---

## State Management

| State | Store | Persistence | Notes |
|---|---|---|---|
| Auth meta (role, businessName) | Zustand `auth` store | `localStorage` | Mirrors non-HttpOnly meta cookie; for UI display only |
| Retailer identity (name, shop, phone) | Zustand `retailer` store | `localStorage` | Set after WhatsApp modal; sent in order POST body |
| Cart items | Zustand `cart` store | `sessionStorage` | Cleared on tab close |
| Products list | React Query | In-memory | `useQuery(['products', wholesalerId])` |
| Orders list | React Query | In-memory | `useQuery(['orders'])` |
| OTP countdown timer | Local `useState` + `useEffect` | Component-local | — |
| Excel upload preview rows | Local `useState` | Component-local | Discarded after POST |

**Auth JWTs** are `HttpOnly; Secure; SameSite=Strict` cookies set by the Worker.
The frontend NEVER reads them. A separate non-HttpOnly `meta` cookie carries `{ role, display }` for rendering only.

---

## wrangler.toml Structure (apps/worker/wrangler.toml)

```toml
name = "projectx-worker"
main = "src/index.ts"
compatibility_date = "2024-09-23"

[[d1_databases]]
binding = "DB"
database_name = "projectx-db"
database_id = "<from: wrangler d1 create projectx-db>"

[[kv_namespaces]]
binding = "OTP_KV"
id = "<from: wrangler kv:namespace create OTP_KV>"

[[r2_buckets]]
binding = "PRODUCT_IMAGES"
bucket_name = "projectx-images"

[triggers]
crons = ["0 2 * * *"]
```

---

## Environment Variables

### Cloudflare Dashboard → Worker Secrets (NEVER in code or wrangler.toml)
```
JWT_SECRET                   # 64-char random hex
MASTER_ADMIN_USERNAME        # set once, never change
MASTER_ADMIN_PASSWORD_HASH   # bcrypt hash of admin password
MASTER_ADMIN_PHONE           # phone number to receive OTP SMS
TWILIO_ACCOUNT_SID
TWILIO_AUTH_TOKEN
TWILIO_FROM_NUMBER
ALLOWED_ORIGIN               # exact Cloudflare Pages URL (no trailing slash)
R2_PUBLIC_BASE_URL           # base URL for serving images via Worker proxy
```

### apps/web → .env.local (gitignored) + Cloudflare Pages env vars
```
NEXT_PUBLIC_API_URL=https://projectx-worker.<account>.workers.dev
```

---

## Implementation Phases

### Phase 0 — Scaffold (Day 1–2)
1. Create `pnpm-workspace.yaml` at repo root listing `apps/*` and `packages/*`
2. Create root `package.json` with `"private": true` and turbo pipeline
3. `cd apps && npx create-next-app@latest web --typescript --tailwind --app --no-src-dir` — then move src into `src/`
4. `cd apps && npm create hono@latest worker` — select `cloudflare-workers` template
5. Create `packages/db/package.json` as plain TS package; add as dependency in web + worker
6. `npx shadcn-ui@latest init` inside `apps/web`
7. Run `wrangler d1 create projectx-db`, `wrangler kv:namespace create OTP_KV`, `wrangler r2 bucket create projectx-images` — paste IDs into `wrangler.toml`
8. Create Cloudflare Pages project linked to this repo, build command: `pnpm --filter web build`
9. Update `.gitignore`: add `.env`, `.env.local`, `.dev.vars`

### Phase 1 — Database (Day 2–3)
1. Write full schema in `packages/db/src/schema.ts` (exact code above)
2. `cd packages/db && npx drizzle-kit generate:sqlite` → creates `drizzle/0000_init.sql`
3. Add index SQL to the migration file
4. `wrangler d1 execute projectx-db --local --file=./packages/db/drizzle/0000_init.sql`
5. `wrangler d1 execute projectx-db --file=./packages/db/drizzle/0000_init.sql` (remote)
6. Add `drizzle-zod` exports from `packages/db/src/index.ts`

### Phase 2 — Worker API Core (Day 3–6)
Build in this exact order (each step unblocks the next):
1. `src/index.ts` — Hono app, CORS middleware (`ALLOWED_ORIGIN` env), 404 handler, error handler
2. `src/lib/jwt.ts` — `signJwt(payload, expiresIn)` and `verifyJwt(token)` using `jose`
3. `src/middleware/auth.ts` — `requireAuth(role)` Hono middleware factory; reads cookie, verifies JWT
4. `src/lib/otp.ts` — `generateOtp()`, `storeOtp(kv, username, otp)` (5-min KV TTL), `verifyAndConsumeOtp(kv, username, otp)`
5. `src/lib/twilio.ts` — `sendSms(to, body)` using Twilio REST API
6. `src/routes/admin.ts` — login, verify-otp, wholesalers CRUD, d1/query, health
7. `src/routes/wholesaler.ts` — login, password change
8. `src/middleware/link.ts` — validates `:token` param against D1 `dynamic_links` table
9. `src/routes/links.ts` — list links, create link
10. `src/lib/r2.ts` — `generatePresignedPutUrl(bucket, key, expiresIn)`
11. `src/routes/products.ts` — upload (D1 batch), upload-url, PATCH, DELETE, GET
12. `src/routes/orders.ts` — create order, list orders, update item status (recalc total), finalize
13. `src/scheduled.ts` — cron handler: delete expired, create new links

### Phase 3 — Master Admin UI (Day 6–8)
1. `app/master-admin/login/page.tsx` — two-step: LoginForm then OtpModal (timer, resend)
2. `app/master-admin/(protected)/layout.tsx` — reads meta cookie, redirects to login if absent
3. `app/master-admin/(protected)/page.tsx` — stats grid (React Query)
4. `app/master-admin/(protected)/wholesalers/page.tsx` — DataTable + CreateWholesalerDialog
5. `app/master-admin/(protected)/d1-explorer/page.tsx` — SQL editor + results
6. `app/master-admin/(protected)/health/page.tsx` — status cards

### Phase 4 — Wholesaler Dashboard (Day 8–12)
1. `app/dashboard/login/page.tsx`
2. `app/dashboard/(protected)/layout.tsx` — auth guard
3. `app/dashboard/(protected)/page.tsx` — ProductGrid with 3-slot image upload per row
   - Image upload flow: call `GET /api/products/upload-url?productId=X&slot=1`, then browser `PUT` to the presigned URL, then `PATCH /api/products/:id` with the returned R2 key
4. `app/dashboard/(protected)/upload/page.tsx` — ExcelDropzone + PreviewTable + POST
5. `app/dashboard/(protected)/orders/page.tsx` — tabbed orders, per-item actions
6. `app/dashboard/(protected)/settings/page.tsx` — password form + link manager

### Phase 5 — Retailer Storefront (Day 12–15)
1. `app/[token]/layout.tsx` — Server Component, fetch Worker `/api/link/:token`, `notFound()` if invalid
2. `app/[token]/page.tsx` — catalog with WhatsApp modal, product cards, CartFab
3. Zustand `retailer` store with `localStorage` persistence (via zustand/middleware `persist`)
4. Zustand `cart` store with `sessionStorage` persistence
5. CheckoutDrawer component → `POST /api/orders` → SuccessModal with WhatsApp deep link
6. Mobile-first responsive polish: 16px gutters, touch-friendly tap targets, image lazy loading, skeleton states

### Phase 6 — Hardening (Day 15–18)
1. Rate limiting middleware in Worker: KV counter per `${route}:${ip}`, 10 req/15 min, 429 response
2. Zod validation on every Worker route (use `@hono/zod-validator`)
3. R2 image serving via Worker proxy route (if bucket is private — preferred over public bucket)
4. Error boundaries in each Next.js dashboard section
5. Loading skeletons for product grid and orders list
6. Optimistic UI for order item status toggles (React Query `useMutation`)
7. `next.config.mjs` — configure `@cloudflare/next-on-pages` edge runtime for `[token]` route
8. End-to-end smoke test of all three user journeys (see Verification section)

---

## Verification Checklist

### Unit Tests (Vitest)
- Drizzle schema types compile correctly
- Zod validators reject invalid inputs
- OTP generate/store/verify/consume logic

### Worker Integration Tests
- `wrangler dev --local` + `hono/testing` test client
- Cover: admin login → OTP → JWT cookie → CRUD, wholesaler login → product upload, link validation

### E2E Smoke Test (manual, in order)
1. Admin: open `/master-admin/login` → enter credentials → receive OTP SMS → enter OTP → land on dashboard
2. Admin: create a wholesaler account with username/password/businessName/whatsapp
3. Wholesaler: open `/dashboard/login` → enter credentials → land on products page
4. Wholesaler: go to Upload → drag in an Excel file → verify preview table → submit → verify products appear in grid
5. Wholesaler: on product grid, upload an image to slot 1 → verify thumbnail appears
6. Wholesaler: go to Settings → copy the storefront link (token URL)
7. Retailer: open the token URL on mobile → verify catalog loads → fill WhatsApp modal → add 3 items to cart → place order
8. Wholesaler: go to Orders → verify new pending order appears → mark 1 item as Stockout → verify total recalculates → approve remaining → finalize
9. Verify: the stockout product now shows `is_in_stock = false` and disappears from the retailer catalog
10. Admin: go to D1 Explorer → run `SELECT * FROM orders` → verify order record exists
11. Trigger cron manually (`wrangler dev` supports `/__scheduled`) → verify old link deleted and new one created

---

## How to Run Locally

```bash
# 1. Install all dependencies
pnpm install

# 2. Run Next.js frontend
pnpm --filter web dev
# opens at http://localhost:3000

# 3. Run Cloudflare Worker (local D1/KV/R2 simulation)
cd apps/worker
wrangler dev
# opens at http://localhost:8787

# 4. Set NEXT_PUBLIC_API_URL=http://localhost:8787 in apps/web/.env.local

# 5. Apply DB migration locally
wrangler d1 execute projectx-db --local --file=./packages/db/drizzle/0000_init.sql

# 6. After any schema change
cd packages/db && npx drizzle-kit generate:sqlite
# then re-run the execute command above

# 7. After any code change, update knowledge graph
graphify update .
```

---

## Security Summary

Full security model is in `SECURITY.md`. Key rules for any Claude session:
- All auth JWTs are **HttpOnly cookies** — never suggest localStorage for tokens
- All secrets live in **Cloudflare Dashboard** — never in code, wrangler.toml, or .env
- Every Worker route must have **Zod validation** — never skip it
- Never use string concatenation for SQL — always **Drizzle ORM parameterized queries**
- Dynamic link tokens use **`crypto.randomUUID()`** (32-char hex) — enforced server-side on every request
- Admin D1 Explorer is **read-only by default** — writes require an explicit toggle
- R2 presigned PUT URLs expire in **15 minutes**
- Rate limit all auth routes via **KV counter** (10 attempts / 15 min / IP)

---

## graphify

This project has a knowledge graph at graphify-out/ with god nodes, community structure, and cross-file relationships.

Rules:
- For codebase questions, first run `graphify query "<question>"` when graphify-out/graph.json exists. Use `graphify path "<A>" "<B>"` for relationships and `graphify explain "<concept>"` for focused concepts. These return a scoped subgraph, usually much smaller than GRAPH_REPORT.md or raw grep output.
- If graphify-out/wiki/index.md exists, use it for broad navigation instead of raw source browsing.
- Read graphify-out/GRAPH_REPORT.md only for broad architecture review or when query/path/explain do not surface enough context.
- After modifying code, run `graphify update .` to keep the graph current (AST-only, no API cost).
