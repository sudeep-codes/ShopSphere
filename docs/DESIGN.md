# DESIGN — ShopSphere UI/UX & API Contract

## 1. Design principles

1. **Demo-first** — the evaluator watches a screen recording; every page must make the data source obvious (badges), text must be readable at 1080p (min 15 px body, 125 % browser zoom while recording).
2. **One clear action per screen** — catalog → add to cart → checkout → pay → track.
3. **Honest states** — loading skeletons, empty states, and explicit error messages that name the cause (e.g. "Only 2 left of *Noise-cancelling Headphones* — order rolled back").
4. **Responsive** — works at 375 px (phone) and 1440 px (recording resolution).

## 2. Design tokens (`frontend/src/styles/tokens.css`)

```css
:root {
  /* brand */
  --color-primary: #4F46E5;      /* indigo — buttons, links */
  --color-primary-ink: #FFFFFF;
  --color-accent: #F59E0B;       /* festival / sale highlights */
  --color-bg: #F8FAFC;
  --color-surface: #FFFFFF;
  --color-text: #0F172A;
  --color-muted: #64748B;
  --color-border: #E2E8F0;
  --color-success: #16A34A;
  --color-danger: #DC2626;

  /* data-source badge colours (also used in diagrams & slides) */
  --ds-rds: #2563EB;             /* blue   */
  --ds-aurora-writer: #7C3AED;   /* violet */
  --ds-aurora-reader: #0D9488;   /* teal   */
  --ds-dynamodb: #EA580C;        /* orange */

  --font-sans: "Inter", system-ui, -apple-system, "Segoe UI", Roboto, sans-serif;
  --font-mono: "JetBrains Mono", ui-monospace, Menlo, Consolas, monospace;
  --text-sm: 0.875rem; --text-base: 1rem; --text-lg: 1.125rem; --text-xl: 1.375rem; --text-2xl: 1.75rem;
  --space-1: 4px; --space-2: 8px; --space-3: 12px; --space-4: 16px; --space-6: 24px; --space-8: 32px;
  --radius: 10px;
  --shadow: 0 1px 2px rgb(0 0 0 / .06), 0 4px 12px rgb(0 0 0 / .06);
}
@media (prefers-color-scheme: dark) {
  :root { --color-bg:#0B1120; --color-surface:#111827; --color-text:#E5E7EB; --color-muted:#94A3B8; --color-border:#1F2937; }
}
```
User preference `theme` (DynamoDB) overrides the OS setting via `data-theme` on `<html>`.

## 3. The Data-Source Badge (signature component)

```
[ ● AURORA · reader · shopsphere-aurora-instance-2 · 4 ms ]   teal
[ ● AURORA · writer · shopsphere-aurora-instance-1 · 9 ms ]   violet
[ ● RDS · shopsphere-rds · 6 ms ]                              blue
[ ● DYNAMODB · ShopSphere_Carts · 3 ms ]                       orange
```
- Source: response headers `X-Data-Source`, `X-DB-Role`, `X-Served-By`, `X-DB-Latency-Ms` (exposed via `Access-Control-Expose-Headers`).
- Placement: top-right of every data panel (catalog grid, cart drawer, order list, timeline); a page with two sources shows two badges.
- Pill, 12 px mono text, 8 px dot in the source colour, tooltip with full instance id.
- A global footer bar shows **current DB mode** (`RDS` / `Aurora`) and the region.

## 4. Page inventory

| # | Route | Page | Content | Badges |
|---|---|---|---|---|
| 1 | `/` | Catalog | Hero "Festival Sale" banner (when Aurora mode), category chips, search, sort, product grid (12/page) | reader |
| 2 | `/product/:id` | Product detail | image, name, price, stock indicator (≤ 3 → "Only N left"), qty stepper, Add to cart | reader + DynamoDB (recently viewed write) |
| 3 | drawer | Cart drawer | lines (qty +/−, remove), subtotal, Checkout button, "Saved in DynamoDB" note | DynamoDB + reader (prices) |
| 4 | `/checkout` | Checkout | address field, order summary, Place order | writer |
| 5 | `/pay/:orderId` | Payment | method tabs CARD/UPI/COD; card form with hint "use …0002 to simulate decline" | writer |
| 6 | `/orders` | My orders | table: order id, date, items, total, status chip | writer |
| 7 | `/orders/:id` | Tracking | vertical status timeline with timestamps; items list | writer |
| 8 | `/login`, `/register` | Auth | simple forms, inline validation | writer + DynamoDB (session) |
| 9 | `/profile` | Preferences | theme toggle, favourite categories, recently viewed strip | DynamoDB |
| 10 | `/admin` | Admin console | DB mode switch (RDS ⇄ Aurora) with confirmation, connection health for each pool (node id, role, ok/error), order list with "Advance status" / "Cancel", "Reset demo data" | all |
| 11 | `/insights` | Insights | bar chart of median latency (20 samples) for DynamoDB GetItem, reader query, writer query; table of last 20 requests with their source; mini comparison table | all |

### Key layouts (wireframes)

```
Catalog (1440 px)
┌──────────────────────────────────────────────────────────────────────┐
│ ShopSphere   [search………………]   Electronics Fashion Home Books   🛒 3  👤│
├──────────────────────────────────────────────────────────────────────┤
│ 🎉 FESTIVAL SALE — served by Aurora readers           [● AURORA·reader]│
│ ┌──────┐ ┌──────┐ ┌──────┐ ┌──────┐                                  │
│ │ img  │ │ img  │ │ img  │ │ img  │   name · ₹price · stock · [Add]  │
│ └──────┘ └──────┘ └──────┘ └──────┘                                  │
│                         ‹ 1 2 3 ›                                    │
├──────────────────────────────────────────────────────────────────────┤
│ Mode: AURORA · Region ap-south-1 · v1.0                              │
└──────────────────────────────────────────────────────────────────────┘

Order tracking
 ● PLACED            05 Oct 10:21
 ● PAID              05 Oct 10:22   txn MOCK-8F2K
 ● SHIPPED           05 Oct 10:40
 ○ OUT_FOR_DELIVERY
 ○ DELIVERED
```

### States
- **Loading**: skeleton cards (no spinners longer than 300 ms without skeleton).
- **Empty cart**: illustration-free text "Your cart is empty — it lives in DynamoDB under your customer id" + Browse button.
- **Errors**: toast with code + human message; checkout rollback shows which product and how many are available.
- **Auth expired** (session deleted): redirect to `/login?reason=session`.

### Accessibility
Colour is never the only signal (badges include text); focus rings visible; buttons ≥ 44 px tall; form labels bound to inputs; contrast ≥ 4.5:1.

## 5. API contract (all JSON, base `/api`)

Envelope: success `{ "data": … }`; error `{ "error": { "code": "STRING", "message": "human text", "details": {} } }`.
Auth: httpOnly cookie `token`. Money: strings with 2 decimals (`"1499.00"`).

| Method | Path | Auth | Body / query | Success | Errors | Store |
|---|---|---|---|---|---|---|
| POST | `/auth/register` | – | `{name,email,phone,password}` | 201 `{customer}` | 400 VALIDATION, 409 EMAIL_TAKEN | writer |
| POST | `/auth/login` | – | `{email,password}` | 200 `{customer}` + cookie | 401 BAD_CREDENTIALS, 429 | writer + DDB |
| POST | `/auth/logout` | ✓ | – | 204 | – | DDB |
| GET | `/auth/me` | ✓ | – | 200 `{customer}` | 401 | writer |
| GET | `/products` | – | `?category&q&sort=price_asc|price_desc|name&page=1&limit=12` | 200 `{items,total,page}` | – | reader |
| GET | `/products/:id` | – | – | 200 `{product}` | 404 | reader (+ DDB recently viewed if logged in) |
| GET | `/products/categories` | – | – | 200 `[…]` | – | reader |
| GET | `/cart` | ✓ | – | 200 `{customer_id,items:[{product_id,quantity,product}],subtotal,version}` | – | DDB + reader |
| POST | `/cart/items` | ✓ | `{product_id,quantity}` | 200 cart | 404 PRODUCT, 409 CART_CONFLICT | DDB |
| PATCH | `/cart/items/:productId` | ✓ | `{quantity}` (0 = remove) | 200 cart | 409 | DDB |
| DELETE | `/cart/items/:productId` | ✓ | – | 200 cart | – | DDB |
| POST | `/orders/checkout` | ✓ | `{shipping_address}` | 201 `{order}` | 400 CART_EMPTY, 409 INSUFFICIENT_STOCK | DDB + writer tx |
| POST | `/orders/:id/pay` | ✓ | `{method,card_number?,upi_id?,idempotency_key}` | 200 `{payment,order}` | 402 PAYMENT_DECLINED, 409 INVALID_TRANSITION | writer tx |
| GET | `/orders` | ✓ | – | 200 `[order]` | – | writer |
| GET | `/orders/:id` | ✓ | – | 200 `{order,items,history,payments}` | 404 | writer |
| POST | `/orders/:id/cancel` | ✓ | – | 200 `{order}` | 409 | writer tx |
| GET | `/preferences` | ✓ | – | 200 prefs | – | DDB |
| PUT | `/preferences` | ✓ | `{theme,favorite_categories}` | 200 prefs | 400 | DDB |
| GET | `/admin/db-mode` | admin | – | 200 `{mode, pools:{writer:{node,ok},reader:{node,ok}}}` | – | – |
| POST | `/admin/db-mode` | admin | `{mode:"rds"|"aurora"}` | 200 | 503 if target pool unhealthy | – |
| POST | `/admin/orders/:id/advance` | admin | `{note?}` | 200 `{order}` | 409 | writer tx |
| GET | `/admin/insights` | admin | `?samples=20` | 200 `{dynamodb_ms,reader_ms,writer_ms,recent:[…]}` | – | all |
| POST | `/admin/reset-demo` | admin | – | 204 | – | all |
| GET | `/health` | – | – | 200 `{ok,mode}` | 503 | all |

Every response includes the `X-Data-Source / X-DB-Role / X-Served-By / X-DB-Latency-Ms` headers (when a store was touched, the *last* store touched wins; endpoints that touch two stores also return `"sources":[…]` in the body).

## 6. Copy & naming

- Product name: **ShopSphere**; tagline "Three databases. One seamless cart."
- Currency: ₹ (INR) formatting with `Intl.NumberFormat('en-IN')` (configurable).
- Status labels: Placed · Payment failed · Paid · Shipped · Out for delivery · Delivered · Cancelled.

## 7. Diagram style (for report/slides)

Same badge colours for each store; AWS official icons optional; export to `docs/diagrams/*.png` at 2× for the proposal.
