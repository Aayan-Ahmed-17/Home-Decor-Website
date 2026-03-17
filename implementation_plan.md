# LuxeDeco — Implementation Plan (PRD)
**Version:** 1.0 | **Date:** 2026-03-16 | **Author:** Senior Architect Review

---

## 1. Project Overview

**Brand:** LuxeDeco — Gen Z-centric home decor e-commerce brand  
**Tagline:** *"Elevate Your Space."*  
**Aesthetic:** Glassmorphic, Purple-dominant, premium-playful, mobile-first  
**Goal:** High-conversion e-commerce with custom basket builder ("Create" flow)

---

## 2. Tech Stack (Free-Tier Friendly, Modern)

### 2.1 Frontend
| Layer | Technology | Version | Rationale |
|-------|-----------|---------|-----------|
| Framework | **Next.js** (App Router) | 14.x | SSR/SSG/ISR for SEO + performance; React ecosystem |
| Language | **TypeScript** | 5.x | Type safety, refactoring confidence |
| Styling | **Tailwind CSS** | 3.x | Utility-first, JIT, perfect for design tokens |
| UI Components | **shadcn/ui** | latest | Unstyled, accessible, fully customizable with Tailwind |
| Animations | **Framer Motion** | 11.x | Page transitions, scroll-triggered reveals, micro-interactions |
| Icons | **Lucide React** | latest | Lightweight, consistent icon set |
| Fonts | Google Fonts (Playfair Display + DM Sans) | — | Serif display + clean body |
| State (Client) | **Zustand** | 4.x | Lightweight cart + UI state management |
| State (Server) | **TanStack Query** | 5.x | Data fetching, caching, revalidation |
| Forms | **React Hook Form + Zod** | latest | Performant forms with schema validation |

### 2.2 Backend
| Layer | Technology | Version | Rationale |
|-------|-----------|---------|-----------|
| Runtime | **Next.js API Routes** (Edge Runtime) | 14.x | Co-located, serverless, free on Vercel |
| Auth | **NextAuth.js v5** (Auth.js) | 5.x | OAuth (Google), magic links, JWT sessions |
| Payments | **Stripe** | latest | Best-in-class; free sandbox for dev |
| Email | **Resend + React Email** | latest | Transactional emails, generous free tier |
| File Uploads | **Uploadthing** | latest | Free tier, S3-backed, Next.js native |
| Search | **Algolia (free tier)** | latest | Instant product search with faceting |

### 2.3 Database
| Layer | Technology | Rationale |
|-------|-----------|---------|
| Primary DB | **PostgreSQL** (via Neon.tech) | Serverless Postgres, generous free tier, branchable |
| ORM | **Drizzle ORM** | Type-safe, lightweight, SQL-first, fast migrations |
| Cache / Sessions | **Upstash Redis** | Serverless Redis; free tier; rate limiting & session caching |

### 2.4 Infrastructure / DevOps
| Service | Tool | Rationale |
|---------|------|---------|
| Hosting | **Vercel** | Zero-config Next.js, Edge Network, free hobby plan |
| CDN / Images | **Vercel Image Optimization** | Built-in `next/image`, AVIF/WebP, lazy load |
| CMS (optional) | **Contentlayer + MDX** or **Sanity.io (free)** | Blog/lookbook content management |
| CI/CD | **GitHub Actions** | Automated lint, type-check, deploy previews |
| Monitoring | **Vercel Analytics + Sentry (free tier)** | Real user metrics, error tracking |

---

## 3. Architecture Overview

```
┌─────────────────────────────────────────────────┐
│                  Vercel Edge Network             │
│  ┌─────────────────────────────────────────┐    │
│  │         Next.js App (App Router)         │    │
│  │  ┌──────────┐  ┌──────────────────────┐ │    │
│  │  │  Pages   │  │   API Routes (Edge)  │ │    │
│  │  │ (SSR/SSG)│  │  /api/products       │ │    │
│  │  │          │  │  /api/cart           │ │    │
│  │  │          │  │  /api/checkout       │ │    │
│  │  │          │  │  /api/auth           │ │    │
│  │  └──────────┘  └──────────────────────┘ │    │
│  └─────────────────────────────────────────┘    │
└─────────────────────────────────────────────────┘
         │                    │
    ┌────▼────┐         ┌─────▼────┐
    │  Neon   │         │ Upstash  │
    │Postgres │         │  Redis   │
    └─────────┘         └──────────┘
         │
    ┌────▼────┐    ┌──────────┐    ┌────────┐
    │Drizzle  │    │  Stripe  │    │ Resend │
    │  ORM    │    │ Payments │    │ Email  │
    └─────────┘    └──────────┘    └────────┘
```

---

## 4. Site Map & Page Structure

```
/                          → Landing Page (Hero, Featured, Vibes Grid)
/shop                      → All Products (grid, filters)
/shop/[category]           → Category Page
/product/[slug]            → Product Detail Page (PDP)
/collections               → Curated Collections
/collections/[slug]        → Collection Detail
/create                    → Custom Basket Builder (3-step flow)
  /create/step-1           → Pick Your Vibe
  /create/step-2           → Add the Goods
  /create/step-3           → Gift or Keep
/cart                      → Cart Drawer / Page
/checkout                  → Stripe Checkout Flow
/account                   → User Dashboard (orders, wishlist)
/account/orders/[id]       → Order Detail
/lookbook                  → Editorial/Blog (lifestyle imagery)
/about                     → Brand Story
/contact                   → Contact Form
```

---

## 5. Database Schema (Drizzle ORM)

```typescript
// schema.ts
export const products = pgTable('products', {
  id: uuid('id').primaryKey().defaultRandom(),
  slug: text('slug').notNull().unique(),
  name: text('name').notNull(),
  description: text('description'),
  price: integer('price').notNull(),          // cents
  compareAtPrice: integer('compare_at_price'),
  categoryId: uuid('category_id').references(() => categories.id),
  images: json('images').$type<string[]>(),
  tags: json('tags').$type<string[]>(),
  stock: integer('stock').default(0),
  featured: boolean('featured').default(false),
  createdAt: timestamp('created_at').defaultNow(),
});

export const categories = pgTable('categories', {
  id: uuid('id').primaryKey().defaultRandom(),
  name: text('name').notNull(),
  slug: text('slug').notNull().unique(),
  image: text('image'),
});

export const orders = pgTable('orders', {
  id: uuid('id').primaryKey().defaultRandom(),
  userId: text('user_id'),
  stripeSessionId: text('stripe_session_id'),
  status: text('status').default('pending'),
  total: integer('total').notNull(),
  items: json('items').$type<OrderItem[]>(),
  createdAt: timestamp('created_at').defaultNow(),
});

export const wishlists = pgTable('wishlists', {
  id: uuid('id').primaryKey().defaultRandom(),
  userId: text('user_id').notNull(),
  productId: uuid('product_id').references(() => products.id),
});

export const customBaskets = pgTable('custom_baskets', {
  id: uuid('id').primaryKey().defaultRandom(),
  sessionId: text('session_id'),
  vibe: text('vibe'),                          // "cozy", "minimal", "bold"
  items: json('items').$type<string[]>(),
  giftMessage: text('gift_message'),
  isGift: boolean('is_gift').default(false),
  createdAt: timestamp('created_at').defaultNow(),
});
```

---

## 6. Key Components & Features

### 6.1 Landing Page Sections
1. **HeroSection** — Full-viewport, video/parallax bg, animated headline, dual CTA
2. **VibeStrip** — Horizontal scrollable "mood" categories (Cozy / Minimal / Bold / Ethereal)
3. **FeaturedGrid** — 3-col masonry product grid with hover reveal
4. **CreateBanner** — 3-step custom basket teaser with animated steps
5. **LookbookReel** — Mobile: vertical scroll reel; Desktop: horizontal filmstrip
6. **TestimonialCarousel** — User reviews with avatar + room photos
7. **NewsletterSection** — Email capture with discount incentive

### 6.2 Custom Basket Builder ("Create" Flow)
```
Step 1: Pick Your Vibe
  → Card selector: Cozy / Minimal / Bold / Ethereal
  → Each vibe pre-filters product suggestions

Step 2: Add the Goods
  → Filtered product grid, drag-to-basket UI
  → Live basket preview (floating sidebar)
  → Quantity controls, remove items

Step 3: Gift or Keep
  → Toggle: "For Me" / "Gift Someone"
  → If Gift: name, message, wrapping style
  → Price summary + Add to Cart → Checkout
```

### 6.3 Cart (Zustand Store)
```typescript
interface CartStore {
  items: CartItem[];
  addItem: (product: Product, qty: number) => void;
  removeItem: (id: string) => void;
  updateQty: (id: string, qty: number) => void;
  clearCart: () => void;
  total: number;
  itemCount: number;
}
```

---

## 7. Performance Strategy

| Concern | Solution |
|---------|----------|
| Image optimization | `next/image` with AVIF/WebP, blur placeholders |
| Font loading | `next/font` (zero CLS, preloaded) |
| Product pages | ISR with `revalidate: 3600` |
| Shop page | Server Components + streaming |
| Animations | `will-change`, reduced-motion media query |
| Bundle size | Dynamic imports for heavy components (basket builder, carousel) |
| Core Web Vitals | Target LCP < 2.5s, CLS < 0.1, FID < 100ms |

---

## 8. Design System Tokens

```css
:root {
  /* Purple Palette */
  --color-lavender: #E8D5FF;
  --color-digital-lavender: #C4A8FF;
  --color-electric-purple: #9B4DFF;
  --color-deep-grape: #4A0080;
  --color-midnight: #1A0030;

  /* Neutrals */
  --color-cream: #FAF7FF;
  --color-mist: #F0EBF8;
  --color-stone: #6B5C7C;

  /* Semantic */
  --color-cta: var(--color-electric-purple);
  --color-cta-hover: #7B2FE8;
  --color-surface: rgba(255,255,255,0.12);  /* Glassmorphism */
  --color-border: rgba(155,77,255,0.2);

  /* Typography */
  --font-display: 'Playfair Display', serif;
  --font-body: 'DM Sans', sans-serif;

  /* Radius */
  --radius-sm: 8px;
  --radius-md: 16px;
  --radius-lg: 24px;
  --radius-xl: 32px;

  /* Shadows */
  --shadow-glow: 0 0 40px rgba(155,77,255,0.25);
  --shadow-card: 0 8px 32px rgba(74,0,128,0.12);
}
```

---

## 9. Implementation Phases

### Phase 1 — Foundation (Week 1–2)
- [ ] Next.js project scaffold (TypeScript, Tailwind, App Router)
- [ ] Design system tokens + shadcn/ui setup
- [ ] Neon PostgreSQL + Drizzle ORM + migrations
- [ ] NextAuth.js (Google OAuth + email magic link)
- [ ] Landing page (Hero + VibeStrip + FeaturedGrid)

### Phase 2 — Core Commerce (Week 3–4)
- [ ] Product listing page (server components + filters)
- [ ] Product detail page (PDP with image gallery)
- [ ] Cart (Zustand) + cart drawer component
- [ ] Stripe checkout integration
- [ ] Order confirmation + Resend email

### Phase 3 — Custom Basket Builder (Week 5)
- [ ] 3-step builder UI (Step 1, 2, 3)
- [ ] Vibe-based product filtering
- [ ] Gift flow + message input
- [ ] Save basket to DB

### Phase 4 — Polish & Optimization (Week 6)
- [ ] Lookbook/editorial section
- [ ] User account dashboard
- [ ] Algolia search integration
- [ ] Mobile animations + transitions
- [ ] Performance audit + Core Web Vitals tuning
- [ ] Vercel deployment + domain setup

---

## 10. Environment Variables

```env
# Database
DATABASE_URL=postgresql://...

# Auth
NEXTAUTH_SECRET=
NEXTAUTH_URL=
GOOGLE_CLIENT_ID=
GOOGLE_CLIENT_SECRET=

# Payments
STRIPE_SECRET_KEY=
STRIPE_WEBHOOK_SECRET=
NEXT_PUBLIC_STRIPE_PUBLISHABLE_KEY=

# Email
RESEND_API_KEY=

# Search
ALGOLIA_APP_ID=
ALGOLIA_API_KEY=
NEXT_PUBLIC_ALGOLIA_SEARCH_KEY=

# Upload
UPLOADTHING_SECRET=
UPLOADTHING_APP_ID=

# Redis
UPSTASH_REDIS_REST_URL=
UPSTASH_REDIS_REST_TOKEN=
```

---

## 11. Folder Structure

```
luxedeco/
├── app/
│   ├── (store)/
│   │   ├── page.tsx               # Landing
│   │   ├── shop/page.tsx
│   │   ├── shop/[category]/page.tsx
│   │   ├── product/[slug]/page.tsx
│   │   ├── collections/page.tsx
│   │   ├── create/
│   │   │   ├── page.tsx           # Step 1
│   │   │   ├── step-2/page.tsx
│   │   │   └── step-3/page.tsx
│   │   ├── cart/page.tsx
│   │   └── checkout/page.tsx
│   ├── (auth)/
│   │   ├── login/page.tsx
│   │   └── register/page.tsx
│   ├── account/
│   │   └── page.tsx
│   └── api/
│       ├── auth/[...nextauth]/route.ts
│       ├── products/route.ts
│       ├── cart/route.ts
│       ├── checkout/route.ts
│       └── webhooks/stripe/route.ts
├── components/
│   ├── ui/                        # shadcn/ui primitives
│   ├── layout/                    # Header, Footer, Nav
│   ├── home/                      # Landing sections
│   ├── product/                   # ProductCard, Gallery, etc.
│   ├── cart/                      # CartDrawer, CartItem
│   └── create/                    # Basket builder steps
├── lib/
│   ├── db/                        # Drizzle + schema
│   ├── auth/                      # NextAuth config
│   ├── stripe/
│   ├── algolia/
│   └── utils.ts
├── store/
│   ├── cart.ts                    # Zustand cart store
│   └── basket.ts                  # Zustand basket builder store
├── styles/
│   └── globals.css                # Design tokens + base styles
└── public/
    └── images/
```

---

*End of PRD — LuxeDeco Implementation Plan v1.0*
