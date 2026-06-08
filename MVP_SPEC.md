---
name: Stitch Ops MVP Spec
description: Enterprise embroidery platform — MVP build specification
date: 2026-06-07
version: 0.1
---

# Stitch Ops — MVP Specification

> White-glove enterprise embroidery. 24-hour turnaround. NYC only.
> A Keel Merchant Bank Company.

---

## 1. Product Definition

**What it is**: A web platform where enterprise clients (banks, PE firms, law firms) describe or upload a garment, place their logo, approve a mockup, and receive embroidered pieces within 24 hours.

**What it is NOT**: A garment retailer. We facilitate blanks; clients BYO or we source on their behalf.

**Core loop**: Describe → Find SKU → Place Logo → Preview → Approve → Embroider → Deliver.

---

## 2. User Flows

### 2.1 New Client Onboarding

```
┌─────────────┐     ┌─────────────┐     ┌─────────────┐
│  Request    │────▶│  Approval   │────▶│  Account    │
│  Access     │     │  (manual)   │     │  Created    │
│  (form)     │     │             │     │             │
└─────────────┘     └─────────────┘     └──────┬──────┘
                                                │
                       ┌────────────────────────┘
                       ▼
                ┌─────────────┐
                │  Dashboard  │
                │  (empty)    │
                └─────────────┘
```

- Form: company name, email, estimated monthly volume, primary use case
- Manual approval: admin reviews, sends invite link
- Account creation: Supabase auth, company profile, billing setup

### 2.2 Create Order (Primary Flow)

```
┌─────────────────┐
│  INPUT          │
│  · Text desc    │
│  · Upload photo │
│  · SKU if known │
└────────┬────────┘
         │
         ▼
┌─────────────────┐
│  AI SEARCH      │
│  · Parse intent │
│  · Query APIs   │
│  · Return SKUs  │
└────────┬────────┘
         │
    ┌────┴────┐
    ▼         ▼
┌───────┐ ┌───────────┐
│ Exact │ │ Suggested │
│ Match │ │ Alternat. │
└───┬───┘ └─────┬─────┘
    │           │
    └─────┬─────┘
          ▼
┌─────────────────┐
│  MOCKUP ENGINE  │
│  · 2D flatlay   │
│  · Logo place   │
│  · Scale/rotate │
└────────┬────────┘
         │
         ▼
┌─────────────────┐
│  CLIENT REVIEW  │
│  [ ] Approve    │
│  [ ] Revise     │
│  [ ] Cancel     │
└────────┬────────┘
         │
         ▼
┌─────────────────┐
│  CHECKOUT       │
│  · Confirm qty  │
│  · Payment      │
│  · Schedule     │
│    pickup       │
└────────┬────────┘
         │
         ▼
┌─────────────────┐
│  PRODUCTION     │
│  · Pickup       │
│  · Digitize     │
│  · Embroider    │
│  · QC           │
│  · Deliver      │
└─────────────────┘
```

### 2.3 Order Status Tracking

```
Client Dashboard ──▶ Order card ──▶ Timeline view
                                    · Pickup scheduled
                                    · In production
                                    · Quality check
                                    · Out for delivery
                                    · Delivered
```

---

## 3. Technical Architecture

### 3.1 Stack

| Layer | Technology |
|-------|-----------|
| Frontend | React 19, TypeScript, Vite, Tailwind CSS, shadcn/ui |
| State | React Query, Zustand |
| Backend | Supabase (Postgres, Auth, RLS, Edge Functions) |
| AI/Search | Custom agent (web search + retailer APIs) |
| Mockup | HTML5 Canvas / SVG 2D flatlay → Three.js 3D preview |
| File Storage | Supabase Storage |
| Hosting | Vercel |
| Payments | Stripe |

### 3.2 Database Schema (Core Tables)

```sql
-- Companies (clients)
companies:
  id uuid pk
  name text
  tier text -- pilot, retainer, enterprise
  contact_email text
  billing_address jsonb
  created_at timestamp

-- Users (within companies)
users:
  id uuid pk
  company_id uuid fk
  email text
  role text -- admin, member
  auth_id uuid -- Supabase Auth

-- Garment SKUs (cached from search)
skus:
  id uuid pk
  brand text
  name text
  category text -- shirt, hoodie, etc.
  color text
  size_options text[]
  image_url text
  retailer_url text
  price_cents integer
  last_verified timestamp

-- Designs (logos + placements)
designs:
  id uuid pk
  company_id uuid fk
  name text
  logo_url text
  digitized_file_url text
  stitch_count integer
  created_at timestamp

-- Orders
orders:
  id uuid pk
  company_id uuid fk
  user_id uuid fk
  status text -- draft, pending, approved, in_production, qc, delivered, cancelled
  sku_id uuid fk
  design_id uuid fk
  placement text -- chest, left_sleeve, right_sleeve, back, custom
  stitch_type text -- standard, 3d_puff, metallic
  quantity integer
  unit_price_cents integer
  total_cents integer
  pickup_address jsonb
  delivery_address jsonb
  scheduled_pickup timestamp
  scheduled_delivery timestamp
  mockup_url text
  approved_at timestamp
  created_at timestamp

-- Order Events (audit trail)
order_events:
  id uuid pk
  order_id uuid fk
  event_type text -- created, mockup_generated, approved, picked_up, embroidered, qc_passed, delivered
  metadata jsonb
  created_at timestamp
```

### 3.3 AI Search Agent

**Input**: Natural language string (e.g., "white Hermes t-shirt, logo on left sleeve")

**Pipeline**:
1. **Intent Parser**: Extract brand, category, color, size, placement preference
2. **SKU Resolver**: Query retailer APIs / web search for exact match
3. **Fallback Suggester**: If exact unavailable, return ranked alternatives
4. **Availability Check**: Verify stock at major retailers
5. **Price Fetch**: Get current retail price

**Retailer APIs to integrate**:
- Google Shopping API (primary)
- Shopify Storefront API (for Shopify brands)
- Direct retailer APIs (Nike, Alo, Lululemon — where available)
- Web scraping fallback (Puppeteer/Playwright for retailer sites)

**Output**: Array of SKU objects with confidence score

### 3.4 Mockup Engine

**Phase 1 — 2D Flatlay (MVP)**:
- HTML5 Canvas or SVG overlay
- Garment template library (silhouettes for common categories)
- Logo upload → auto-center on placement zone
- Drag to reposition, corner handles to resize
- Rotation control
- Color overlay matching garment color

**Phase 2 — 3D Preview (Post-MVP)**:
- Three.js draped garment model
- Realistic fabric texture
- Lighting and shadow
- Export as shareable image

### 3.5 Order Management Pipeline

```
Status Machine:

draft ──▶ pending ──▶ approved ──▶ in_production ──▶ qc ──▶ delivered
            │            │              │              │
            └────────────┴──────────────┴──────────────┘
                         (cancelled from any state)
```

**Automated transitions**:
- `draft` → `pending`: client submits order
- `pending` → `approved`: client checks approval box + pays
- `approved` → `in_production`: pickup confirmed, digitizing complete
- `in_production` → `qc`: embroidery complete
- `qc` → `delivered`: QC pass + handoff to delivery

**Notifications** (Telegram bot to client):
- Order received
- Mockup ready for approval
- Order approved → in production
- Out for delivery
- Delivered

---

## 4. UI/UX Specification

### 4.1 Design System

- **Brand**: Argonav design system (already applied in mockup)
- **Colors**: Space Blue `#0B1A2D`, Carbon `#1C1C1E`, Atmosphere Blue `#7DD2F3`, Light Gray `#F5F5F7`
- **Typography**: Plus Jakarta Sans (Regular headlines, Light body), Space Mono (labels)
- **Components**: Glass panels, pill buttons, orbit diagrams, mission labels

### 4.2 Pages

| Page | Route | Purpose |
|------|-------|---------|
| Landing | `/` | Marketing page with pricing, process, generator demo |
| Login | `/login` | Supabase auth |
| Dashboard | `/dashboard` | Order history, create new order |
| Generator | `/generator` | Full product generator tool |
| Order Detail | `/orders/:id` | Status timeline, mockup, approval |
| Account | `/account` | Company profile, billing, users |
| Admin | `/admin` | Order queue, client approvals, analytics |

### 4.3 Key Interactions

**Generator Page**:
1. Text input with placeholder: "Describe what you want..."
2. AI search indicator (spinning orbit)
3. SKU results as cards (image, brand, name, price, availability)
4. Click SKU → load mockup canvas
5. Logo upload zone (drag + drop)
6. Placement selector: chest / left sleeve / right sleeve / back / custom
7. Stitch type: standard / 3D puff / metallic
8. Mockup canvas: logo draggable, resizable, rotatable
9. Approval checkbox + "Confirm & Place Order" button

**Dashboard**:
- Order cards: SKU image, status badge, quantity, delivery date
- Status colors: draft (gray), pending (yellow), approved (blue), in production (purple), qc (orange), delivered (green)
- "New Order" CTA → generator

---

## 5. Pricing Integration

**Stripe Setup**:
- Products: Pilot, Retainer, Enterprise (recurring subscriptions)
- Per-unit charges: usage-based billing for embroidery
- Concierge sourcing: separate line item (garment cost + 15% + $25)

**Billing Rules**:
- Pilot: pay per order, no subscription
- Retainer: monthly subscription + per-unit overage
- Enterprise: custom contract, invoice net-30

---

## 6. MVP Scope (Phase 1)

### In Scope
- [ ] Landing page (marketing)
- [ ] Auth (login/signup)
- [ ] Dashboard (order list)
- [ ] Product generator (text input → SKU search → 2D mockup)
- [ ] Order creation + approval flow
- [ ] Order status tracking
- [ ] Admin panel (order queue)
- [ ] Telegram notifications
- [ ] Stripe billing (Pilot tier only for MVP)

### Out of Scope (Phase 2+)
- [ ] 3D preview engine
- [ ] Concierge garment purchasing
- [ ] API access for Enterprise
- [ ] White-label packaging
- [ ] Same-day rush
- [ ] Multi-city expansion
- [ ] Mobile app

---

## 7. Build Phases

### Sprint 1 (Week 1): Foundation
- Supabase project setup
- Auth + companies table
- Landing page (static, deploy to Vercel)
- Design system component library

### Sprint 2 (Week 2): Generator
- SKU search agent (web search integration)
- 2D mockup canvas
- Logo upload + placement
- Mockup approval flow

### Sprint 3 (Week 3): Orders
- Order creation from generator
- Order status pipeline
- Dashboard order list
- Admin order queue

### Sprint 4 (Week 4): Polish
- Stripe integration
- Telegram notifications
- QA + bug fixes
- Soft launch (internal Keel users)

---

## 8. Open Questions

1. **Operation Embroidery acquisition**: Timeline? Will their existing clients migrate?
2. **SKU search depth**: How many brands/categories for MVP? Start with 20 most common?
3. **Garment templates**: Do we have silhouette assets or need to create?
4. **Pickup logistics**: In-house courier or third-party (Uber Direct, etc.)?
5. **Digitizing**: Automated (Ink/Stitch) or manual for complex logos?

---

*Stitch Ops MVP Spec v0.1 — 2026-06-07*
