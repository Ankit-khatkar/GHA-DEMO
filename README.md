# Phase 2 — Owner Dashboard: Detailed Build Plan

> **Status:** Planning
> **Duration:** ~1 week
> **Goal:** Restaurant owners can toggle open/close, receive incoming orders in real time, accept or decline with a reason, progress accepted orders through the status lifecycle, and see today's completed/declined/expired history — all on one page.
> **Prerequisite:** Phase 1 complete — customer flow works end-to-end. Seed data includes at least one owner auth user mapped to a `restaurants` row via `owner_id`. At least one `pending` test order must be placeable by a customer account so the owner side can be tested without waiting.
>
> **Notification sounds / browser notifications are deferred to Phase 4 (stabilisation).** Owners in a busy kitchen will miss silent updates, but "proper" notification support (Android Chrome permissions, iOS Safari quirks, lock-screen behaviour) needs real-device testing. The 15-min countdown timer (this phase) gives owners a visual urgency cue even without audio. Phase 4 mobile audit is the right place to add sound or vibration.

---

## Prerequisite Migration — 004_owner_dashboard.sql

Before any Phase 2 frontend work, one migration must be applied to unblock Step 3. Without it, owners cannot see who placed each incoming order (RLS `users_self_select` — Section 4.8.1 — blocks owners from reading rows in `public.users` for other users, so a joined `users:customer_id(full_name, phone)` select would return `null` for non-admin callers).

**What the migration does:**

1. **Adds `customer_name TEXT NOT NULL` and `customer_phone TEXT NOT NULL` to `orders`.** These are snapshots at order-placement time — not foreign-keyed, not updated if the customer later renames themselves in their profile. Same pattern as `order_items.unit_price` (Section 4.4.4): *"price at the moment of ordering is what matters; updates to `menu_items.price` later must not retroactively change historical orders."* The same reasoning applies to customer contact.
2. **Updates the `place_order` RPC (Section 5.1.2)** to read `full_name` and `phone` from `public.users` for the calling user and insert them into the new columns as part of the same atomic transaction. The RPC runs `SECURITY DEFINER`, so it can read the caller's own row in `users` even though RLS on the table is restrictive. Customers still never pass their own name/phone from the frontend — the RPC reads the authoritative value from the DB.

**Why snapshot:**

- Owners need a working customer name and phone number at the moment they're preparing a delivery. If a customer changes their phone in `/profile` an hour after ordering, the delivery guy calling the number from the order card should still reach them — but *also* the snapshot preserves what was valid at order time, which is the audit-trail standard. Snapshotting is simpler than building a pointer-with-versioning scheme.
- Avoids the RLS gymnastics of opening a narrow `users` read policy just for owners' order-related lookups. That policy would need to be scoped to "users with an order in my restaurant" — expressible, but fragile and easy to get wrong.
- Matches how `order_items.unit_price` already works. Consistent mental model for the team.

**Why not just expose a `SECURITY DEFINER` function `get_customer_for_order(order_id)`:** It works, but it adds an extra round-trip per card and complicates the Realtime path (would need a fan-out call on every INSERT payload). A column snapshot is read in the same query that already fetches the order.

**Deployment note:** This is a new column on an existing table with a NOT NULL constraint. Gudha Gorji seed data is empty of real orders at Phase 2 start, so there's no backfill problem in practice. If any test orders exist, either delete them (preferred — they're test data) or backfill via a one-off SQL statement before applying the NOT NULL constraint.

### Concrete SQL (reference sketch for `004_owner_dashboard.sql`)

```sql
-- 1. Add snapshot columns (nullable first, backfill, then NOT NULL)
ALTER TABLE public.orders
  ADD COLUMN customer_name  text,
  ADD COLUMN customer_phone text;

-- Backfill (safe no-op on an empty table; otherwise reads authoritative values
-- from public.users via customer_id FK)
UPDATE public.orders o
SET customer_name  = u.full_name,
    customer_phone = u.phone
FROM public.users u
WHERE o.customer_id = u.id
  AND (o.customer_name IS NULL OR o.customer_phone IS NULL);

ALTER TABLE public.orders
  ALTER COLUMN customer_name  SET NOT NULL,
  ALTER COLUMN customer_phone SET NOT NULL;

-- 2. Replace the existing place_order RPC — it now snapshots customer contact
--    as part of the same atomic INSERT. SECURITY DEFINER is preserved so the
--    function can read the caller's own row in public.users even though RLS
--    restricts SELECT on that table.
CREATE OR REPLACE FUNCTION public.place_order(
  p_restaurant_id       uuid,
  p_delivery_address    text,
  p_special_instructions text,
  p_items               jsonb   -- [{ menu_item_id, quantity, unit_price }, ...]
) RETURNS uuid
LANGUAGE plpgsql
SECURITY DEFINER
SET search_path = public
AS $$
DECLARE
  v_order_id  uuid;
  v_full_name text;
  v_phone     text;
BEGIN
  -- Read the authoritative customer contact from public.users.
  -- Never trust these values from the client.
  SELECT full_name, phone
    INTO v_full_name, v_phone
  FROM public.users
  WHERE id = auth.uid();

  IF v_full_name IS NULL OR v_phone IS NULL OR length(trim(v_phone)) = 0 THEN
    RAISE EXCEPTION 'Profile incomplete — full_name and phone are required to place an order';
  END IF;

  INSERT INTO public.orders (
    customer_id, restaurant_id,
    customer_name, customer_phone,
    delivery_address, special_instructions,
    total_amount,  -- recalculated by the total_amount trigger after INSERT; placeholder 0 is fine
    status
  ) VALUES (
    auth.uid(), p_restaurant_id,
    v_full_name, v_phone,
    p_delivery_address, NULLIF(p_special_instructions, ''),
    0,
    'pending'
  )
  RETURNING id INTO v_order_id;

  INSERT INTO public.order_items (order_id, menu_item_id, quantity, unit_price)
  SELECT v_order_id,
         (item->>'menu_item_id')::uuid,
         (item->>'quantity')::int,
         (item->>'unit_price')::numeric(10,2)
  FROM jsonb_array_elements(p_items) AS item;

  RETURN v_order_id;
END;
$$;
```

> This is a **reference sketch**, not the final migration file. Diff against the existing `place_order` body in `002_security.sql` before committing — signature must match Phase 1's customer checkout caller (`src/pages/checkout/Checkout.tsx`). If the current RPC's parameter names differ, keep the existing names and only add the customer-snapshot logic inside the body.

### Post-migration: Supabase Dashboard Checklist

Two things the migration file cannot do for you — must be toggled manually in the Supabase Dashboard, once per environment:

1. **Enable Realtime on `public.orders`.** Database → Replication → Supabase Realtime → add `public.orders`. Step 3 and Step 7 both depend on Realtime `INSERT` and `UPDATE` events from this table; without this toggle, the subscriptions return immediately without ever firing a payload and the dashboard will silently appear dead. Phase 1 already enables Realtime on `orders` for the customer's `/orders/:id` page, so this may already be on — verify before Step 3. If `order_items` is also enabled, that's fine but not required (the dashboard re-fetches line items on every INSERT).
2. **Regenerate TypeScript types** after migration 004 applies: `npx supabase gen types typescript --local > src/types/database.ts`. Then update `src/types/models.ts`'s `Order` alias to include `customer_name` and `customer_phone`. Without this, Step 3's query select list fails type-checking.

---

## Guiding Principles

- **One page, three sections.** Pending / Active / Today's History. No separate routes for each status bucket. Matches **Section 7.4** and the repo's "owner dashboard = one page with sections (not 4 routes)" simplicity rule.
- **Real time or bust.** Owners must not refresh. A new `pending` order on the customer side must appear on the owner's dashboard within seconds. If Realtime fails, the dashboard shows stale data and the owner misses orders — so the Realtime setup and reconnection behaviour is treated as a first-class feature, not a bolt-on.
- **Trust DB triggers for business rules.** Status transitions (Section 4.7.1), decline reason (Section 4.7.3), and auto-expiry (Section 4.7.5) are all enforced at the database layer. The frontend renders UI and surfaces DB errors — it does not re-validate business rules.
- **Owner actions are idempotent at the DB.** The `WHERE status = 'pending'` race guard (Section 4.7.5) ensures a double-accept, or an accept-after-cron-expired, affects 0 rows instead of corrupting state. Frontend reacts to the zero-row result with a toast and a re-fetch.
- **One visible CTA per order card.** Pending → [Accept] [Decline]. Active → [Next Step]. No extra buttons, no dropdowns, no overflow menus in v1. This matches the tier-3 simplicity goal and mirrors Phase 1's "one clear primary action per screen" rule.
- **Countdown timer is visual only.** The 15-min expiry is enforced by the `expire-orders` cron job (Phase 0). The timer on the card is a UX hint — it does not drive status change. When the timer crosses zero, the card waits for the Realtime `UPDATE` from cron to flip its state to `expired`.

---

## Step 1: Owner Navbar + Route Guard + Post-Login Redirect

### Why First
Before the dashboard page exists, the owner's path *into* the dashboard must work: Navbar shows the right links, `<ProtectedRoute role="owner">` gates the route, and post-login redirect sends owners to `/dashboard` instead of `/`. Without this, an owner logs in and lands on the customer landing page.

### Files to Update
- `src/components/Navbar.tsx` + `Navbar.css` — **verify only, don't rebuild.** Phase 1 Step 5 already wired the owner state (`{ to: "/dashboard", label: "Dashboard" }` in the owner link list). Smoke-test it by logging in as an owner and confirming the correct links render; no changes expected.
- `src/pages/login-signup/Login.tsx` — update post-login redirect to send `owner` → `/dashboard` (currently falls back to `/`)
- `src/pages/auth/AuthCallback.tsx` — same role-based redirect for OAuth + email confirmation (currently falls back to `/` for owner)
- `src/App.tsx` — add the `/dashboard` route (placeholder for now; real content in Step 2)

### Auth-Aware Navbar (Owner State)

Per the Phase 1 Step 5 matrix, the owner state is already wired:

| Left Side | Right Side |
|---|---|
| Logo, Dashboard | Profile, Logout |

No cart badge, no "My Orders", no "Restaurants". Owners only see what they need. If smoke-testing reveals a regression, fix it here; otherwise leave `Navbar.tsx` alone.

### Post-Login Redirect

Currently `Login.tsx` and `AuthCallback.tsx` redirect customers to `/restaurants` (Phase 1 Step 5) and fall back to `/` for all other roles. Update the fallback:

- `customer` → `/restaurants`
- `owner` → `/dashboard`
- `admin` → `/` (stays on landing until Phase 3)

### DB Mapping

| What | Source |
|---|---|
| Role detection | `profile.role` from `AuthContext` → `users` table (**Section 4.1.1**) |
| Route guard | `<ProtectedRoute role="owner">` — existing component from Phase 1 |
| RLS implication | None at this step — guard is frontend-only; DB RLS enforces separately on every query |

### Testable Output
Log in as a seed owner account → Navbar shows "Dashboard" on the left and Profile/Logout on the right → user is redirected to `/dashboard` → `/dashboard` renders a placeholder ("Owner Dashboard — Step 2 pending"). Log in as a customer and try `/dashboard` directly in the URL → redirected to `/` (wrong role). Log out and try `/dashboard` → redirected to `/login`.

---

## Step 2: Dashboard Shell + Open/Close Toggle

### Why Second
The toggle is the simplest piece of owner functionality, touches only the `restaurants` table, and validates that the owner's restaurant can be fetched correctly via the UNIQUE `owner_id` constraint. Get this right before adding the order sections on top.

### Files to Create
- `src/pages/dashboard/OwnerDashboard.tsx`
- `src/pages/dashboard/OwnerDashboard.css`

### Supabase Queries

```ts
// Fetch the owner's restaurant. Use .maybeSingle() — .single() throws
// PGRST116 if no row exists; .maybeSingle() returns data: null cleanly.
// A newly signed-up owner whose restaurant row hasn't been created by admin
// yet is a real case, not an error.
const { data: restaurant, error } = await supabase
  .from('restaurants')
  .select('id, name, cuisine_type, address, is_open, is_active, updated_at')
  .eq('owner_id', user.id)
  .maybeSingle()

// Toggle is_open (optimistic update in UI; rollback on error)
await supabase
  .from('restaurants')
  .update({ is_open: !restaurant.is_open })
  .eq('id', restaurant.id)
```

### Onboarding-Incomplete State

If `.maybeSingle()` returns `null`, the dashboard renders an onboarding screen instead of the normal three sections:

> **Your restaurant isn't set up yet.**
> We're finishing your onboarding. Please contact Ankit on WhatsApp at +91 63789 39472 and he'll get your restaurant live within 24 hours.
> [ Open WhatsApp ]

Reasoning: in v1 admin creates `restaurants` rows manually via Supabase Dashboard while personally onboarding each owner. The owner's auth account may exist a few hours before their restaurant row does. A crash ("undefined is not an object") in that window would be a terrible first impression. No retry or spinner — the fix is human, not automated.

### DB Mapping

| What | Source |
|---|---|
| Table | `restaurants` (**Section 4.1.2**) |
| Editable field | `is_open` only — owners **cannot** flip `is_active` (**Section 4.1.2**: *"is_active is set by admin"*) |
| Uniqueness guarantee | `owner_id UNIQUE` — `.single()` is safe (**Section 4.1.2**) |
| RLS (select) | `restaurants_owner_select` — `owner_id = auth.uid()` (**Section 4.8.2**) |
| RLS (update) | `restaurants_owner_update` — same predicate; admin policies govern other fields (**Section 4.8.2**) |
| `updated_at` | Auto-refreshed by trigger; displayed as "Last changed at …" next to the toggle |

### Critical Rules

| Rule | Enforcement | Section |
|---|---|---|
| Owner cannot set `is_active = true` themselves | **Frontend only in v1** — the dashboard does not render an `is_active` control. Deployed `restaurants_owner_update` policy enforces row ownership only (no column-level restriction), so a determined owner could bypass via a direct SDK call. Trusted-owner pool (~12 hand-onboarded restaurants) makes this an accepted v1 trade-off. See [`v2_deferred_issues.md`](v2_deferred_issues.md) §1 for the v2 fix (SECURITY DEFINER `toggle_restaurant_open` RPC). | **4.8.2** + `v2_deferred_issues.md` §1 |
| If `is_active = false`, `is_open` toggle is **disabled** with an explanatory message: *"Your restaurant isn't active yet. Please contact admin."* | Frontend | **4.1.2** |
| Customers stop seeing the restaurant within seconds of `is_open = false` | `restaurants_customer_select` RLS filters `is_active AND is_open` on every SELECT | **4.8.2** |

### UI Details

- Restaurant header: name (charcoal, 24px, DM Serif Display), cuisine and address (slate, 14px)
- Toggle: large switch with labels `Closed` / `Open for Orders`, red when open, grey when closed
- Status line under the toggle: *"Last changed: 2:45 PM, 14 Apr"* (IST, from `updated_at`)
- When `is_active = false`: toggle is disabled and greyed out; helper text explains why

### Route
`/dashboard` — wrapped in `<ProtectedRoute role="owner">`. The `role="owner"` prop blocks customers and admins at the route layer even if RLS would already reject their queries.

### Testable Output
Log in as a seed owner → dashboard loads with the owner's restaurant → toggle `is_open` on/off → `updated_at` ticks forward → open a customer tab and confirm the restaurant appears/disappears on `/restaurants` within seconds. Manually set `is_active = false` for the seed restaurant in Supabase Dashboard → confirm the toggle is disabled with the explanatory message.

---

## Step 3: Incoming Orders Section (Pending) + Realtime INSERT + Countdown Timer

### Why Third
This is the heart of the dashboard. Once the owner can see a pending order arrive in real time with a countdown, the accept/decline flow (Step 4) has something meaningful to act on.

### Files to Update
- `src/pages/dashboard/OwnerDashboard.tsx` — add Incoming Orders section

### Supabase Queries

```ts
// Initial fetch of pending orders — customer contact comes from snapshot columns
// on orders (added by migration 004; see Prerequisite Migration section above)
const { data: pendingOrders } = await supabase
  .from('orders')
  .select(`
    id, customer_id, customer_name, customer_phone,
    total_amount, delivery_address,
    special_instructions, created_at, expires_at, status,
    order_items(quantity, unit_price, menu_items(name, is_veg))
  `)
  .eq('restaurant_id', restaurant.id)
  .eq('status', 'pending')
  .order('created_at', { ascending: true })

// Realtime subscription — INSERT = new pending order arrives
supabase
  .channel(`owner-${restaurant.id}`)
  .on(
    'postgres_changes',
    {
      event: 'INSERT',
      schema: 'public',
      table: 'orders',
      filter: `restaurant_id=eq.${restaurant.id}`
    },
    async (payload) => {
      // payload.new is the flat orders row — customer_name/customer_phone are
      // already in the payload (plain columns, not a join). But order_items
      // and menu_items are not, so re-fetch for the line items.
      const { data: fullOrder } = await supabase
        .from('orders')
        .select(`
          id, customer_id, customer_name, customer_phone,
          total_amount, delivery_address,
          special_instructions, created_at, expires_at, status,
          order_items(quantity, unit_price, menu_items(name, is_veg))
        `)
        .eq('id', payload.new.id)
        .single()

      if (fullOrder) {
        setPendingOrders(prev => [...prev, fullOrder])
      }
    }
  )
  .subscribe()
```

> **Why snapshot columns instead of a join:** Deployed RLS policy `users_self_select` (Section 4.8.1) restricts `public.users` SELECT to own row + admin. An owner-side `users:customer_id(full_name, phone)` join returns `null` for non-admin callers. Adding a narrow "users with an order in my restaurant" policy is expressible but fragile. Migration 004 (see Prerequisite Migration section) adds `orders.customer_name` and `orders.customer_phone` and populates them via `place_order` — the simplest solution that also matches the existing `order_items.unit_price` snapshot pattern.

### DB Mapping

| What | Source |
|---|---|
| Table | `orders` + joined `order_items` + `menu_items` (**Sections 4.1.4, 4.1.5, 4.1.3**) |
| RLS (select) | `orders_owner_select` — orders where `restaurant_id` belongs to the owner (**Section 4.8.4**) |
| RLS (order_items select) | `order_items_owner_select` — parent order belongs to owner's restaurant (**Section 4.8.5**) |
| Realtime filter | `restaurant_id=eq.${restaurant.id}` — server-side filter; owner only receives their own inserts |
| Index used | `(restaurant_id, status)` (**Section 4.7.7**) |
| `expires_at` | Auto-set by DB default `now() + 15 min` (**Section 4.1.4**) — drives the countdown timer |

### Countdown Timer

Each pending order card shows time remaining until `expires_at`:

```ts
// One shared ticker for the whole dashboard — update every second
useEffect(() => {
  const tick = setInterval(() => setNow(Date.now()), 1000)
  return () => clearInterval(tick)
}, [])

function remaining(expiresAt: string): { mm: string, ss: string, isNegative: boolean } {
  const ms = new Date(expiresAt).getTime() - now
  const abs = Math.max(0, ms)
  const mm = String(Math.floor(abs / 60_000)).padStart(2, '0')
  const ss = String(Math.floor((abs % 60_000) / 1000)).padStart(2, '0')
  return { mm, ss, isNegative: ms < 0 }
}
```

**Visual states of the timer:**

| Remaining | Colour | Meaning |
|---|---|---|
| > 5 min | slate (`#4A4A4A`) | Plenty of time |
| 1–5 min | amber (`#F39C12`) | Act soon |
| < 1 min | red (`#D63031`), pulsing | Act now |
| 0 / negative | grey, struck-through | Expired — waiting for cron |

**Critical:** The timer does **not** change the order's status when it hits zero. The `expire-orders` cron job (Section 4.7.5) does that, and the UPDATE propagates via Realtime (Step 7). The timer just shows *visual* urgency. If the owner's clock skews, they might see a negative timer while the order is still `pending` server-side — that's fine, the Accept button still works until the DB flips the status.

### UI Details — Pending Order Card

```
┌─────────────────────────────────────────┐
│ ⏱ 12:34  Pending                         │ ← countdown + status badge
│                                          │
│ 2 items · ₹440                           │ ← total_amount
│                                          │
│ 🟢 Paneer Tikka × 2        ₹220         │
│ 🔴 Mutton Rogan Josh × 1   ₹350         │
│                                          │
│ 👤 Ramesh Kumar                          │ ← customer_name (snapshot)
│ 📞 +91 63789 39472     [ Call ]          │ ← customer_phone + tel: link
│ 📍 123, Main Market, Gudha Gorji         │ ← delivery_address
│ 📝 Extra spicy, call when near           │ ← special_instructions (if any)
│                                          │
│  [  Decline  ]    [    Accept    ]       │ ← ghost btn + red filled btn
└─────────────────────────────────────────┘
```

**Customer contact rules:**
- `customer_name` and `customer_phone` come from the snapshot columns on `orders` (migration 004). Never try to join `public.users` from the owner side.
- The phone is a tappable `tel:+91XXXXXXXXXX` link — on mobile (the primary owner device), tapping opens the dialler. The "Call" button is a separate visible CTA for desktop clarity, wrapping the same `tel:` href.
- Customer contact is **only** visible on the pending and active cards (Sections 3 & 5). History cards (Section 6) omit it — once the order is terminal, there's no operational reason to dial the customer.

### Empty State

*"No pending orders right now. You'll hear from us when a new order arrives."* (Text only for v1 — sound/notifications deferred to Phase 4.)

### Testable Output
With the dashboard open, place an order from a customer tab → the card appears in the Pending section within ~2 seconds without refresh. Countdown ticks down from ~14:59. Refresh the dashboard → the card persists (initial fetch works). Let a pending order sit past 15 minutes → the timer goes red and then 00:00 → within a minute the `expire-orders` cron fires and the card flips to expired (handled in Step 7).

---

## Step 4: Accept / Decline Flow

### Why Fourth
Pending orders without an accept/decline action are useless. This step wires the two primary CTAs into the DB and handles the race conditions that arise when cron, owner, and customer all act on the same row.

### Files to Update
- `src/pages/dashboard/OwnerDashboard.tsx` — add Accept and Decline handlers
- New component: `src/pages/dashboard/DeclineModal.tsx` + `.css` — a portal-less dialog rendered in `OwnerDashboard`. Parent owns state: `{ declining: { orderId: string } | null }`. Decline button on a pending card sets it; modal's Cancel resets it to `null`; modal's Confirm invokes the parent-supplied `onConfirm(reason)` handler, which runs the UPDATE and then resets the state. Keep it simple — no React portal, no router-based modal.

### Supabase Calls

```ts
// Accept — race guard via WHERE status='pending' (Section 4.7.5)
const { data, error } = await supabase
  .from('orders')
  .update({ status: 'accepted' })
  .eq('id', orderId)
  .eq('status', 'pending')
  .select('id, status')

if (!data || data.length === 0) {
  // 0 rows affected — the order was expired by cron, declined from another device,
  // or already accepted. Toast: "This order is no longer pending. Refreshing…"
  // then re-fetch pending orders.
}

// Decline — requires decline_reason per Section 4.7.3 CHECK constraint
const { data, error } = await supabase
  .from('orders')
  .update({ status: 'declined', decline_reason: trimmedReason })
  .eq('id', orderId)
  .eq('status', 'pending')
  .select('id, status')
```

### DB Mapping

| What | Source |
|---|---|
| Status transitions allowed | `pending → accepted`, `pending → declined` (**Section 4.3.2, 4.7.1**) |
| Status transition trigger | Rejects any other transition from `pending` (e.g., `pending → completed`) at DB level (**Section 4.7.1**) |
| Decline reason CHECK | `status = 'declined' AND decline_reason IS NOT NULL AND length(trim(decline_reason)) > 0` (**Section 4.7.3**) |
| Race guard | `.eq('status', 'pending')` in the UPDATE — if cron or another device already moved the row, UPDATE affects 0 rows (**Section 4.7.5**) |
| RLS | `orders_owner_update` — owner's restaurant only (**Section 4.8.4**) |

### Decline Modal

```
┌─────────────────────────────────────────┐
│  Decline this order?                     │
│                                          │
│  Please tell the customer why. They'll   │
│  see this reason in their order status.  │
│                                          │
│  ┌─────────────────────────────────┐    │
│  │ e.g. "Dal Makhani unavailable    │    │ ← textarea, required, 200 char max
│  │  today"                           │    │
│  └─────────────────────────────────┘    │
│                                          │
│  [ Cancel ]         [ Confirm Decline ]  │ ← ghost | red filled
└─────────────────────────────────────────┘
```

**Modal rules:**
- Confirm button disabled until `reason.trim().length >= 3`
- After confirm: button shows spinner, disabled until response
- Character counter: "0 / 200"

### Critical Safeguards

| Rule | Enforcement | Section |
|---|---|---|
| Double-click Accept | Button disabled immediately after first click; spinner; rollback on error | Frontend |
| Accept-after-expire race | `.eq('status', 'pending')` returns 0 rows when cron already expired the order → toast + re-fetch | **4.7.5** |
| Decline without reason | DB CHECK rejects; frontend prevents it via modal validation | **4.7.3** |
| Attempt `pending → completed` (skip) | Status transition trigger rejects at DB | **4.7.1** |
| Another owner device accepted first | Same 0-row result as cron race — treated identically | **4.7.5** |

### On Success
- Accept → card moves from Pending section to Active Orders section (status `accepted`)
- Decline → card moves from Pending section to Today's History section (status `declined`, with the reason visible)
- Toast: *"Order accepted"* or *"Order declined"*

### Testable Output
Pending card → tap Accept → card moves to Active section within the same tick; customer's `/orders/:id` page updates in real time to "Order confirmed!". Pending card → tap Decline → modal → enter reason → confirm → card moves to History; customer's page shows "Order declined. Reason: …". Open two owner tabs, accept from one, try to accept from the other → second tab gets toast "No longer pending. Refreshing…" and re-fetches. Let a pending order expire (cron) → try to accept → same 0-row handling.

---

## Step 5: Active Orders Section + Status Progression

### Why Fifth
After acceptance, the owner needs to walk the order forward: `accepted → preparing → out_for_delivery → completed`. Same data, same DB, one progression button per card.

### Files to Update
- `src/pages/dashboard/OwnerDashboard.tsx` — add Active Orders section

### Supabase Queries

```ts
// Initial fetch — active statuses. Carries customer_name/customer_phone
// forward from the snapshot so the delivery person can still dial the
// customer during out_for_delivery.
const { data: activeOrders } = await supabase
  .from('orders')
  .select(`
    id, customer_name, customer_phone,
    total_amount, delivery_address, special_instructions,
    created_at, status,
    order_items(quantity, menu_items(name, is_veg))
  `)
  .eq('restaurant_id', restaurant.id)
  .in('status', ['accepted', 'preparing', 'out_for_delivery'])
  .order('created_at', { ascending: true })

// Next-step progression
const NEXT: Record<OrderStatus, OrderStatus | null> = {
  accepted:         'preparing',
  preparing:        'out_for_delivery',
  out_for_delivery: 'completed',
  pending:          null,  // handled in Step 4
  completed:        null,  // terminal
  declined:         null,  // terminal
  expired:          null,  // terminal
}

await supabase
  .from('orders')
  .update({ status: NEXT[currentStatus]! })
  .eq('id', orderId)
  .eq('status', currentStatus)   // race guard — must still be in the expected status
  .select('id, status')
```

### DB Mapping

| What | Source |
|---|---|
| Valid transitions | `accepted → preparing → out_for_delivery → completed` (**Section 4.3.2**) |
| Transition trigger | Rejects skips (e.g., `accepted → out_for_delivery`) and reversals (**Section 4.7.1**) |
| Terminal states | `completed`, `declined`, `expired` — no button rendered (**Section 4.3.2**) |
| Race guard | `.eq('status', currentStatus)` on the UPDATE |
| RLS | `orders_owner_update` (**Section 4.8.4**) |

### Button Label per Status

| Current status | Button label | Next status |
|---|---|---|
| `accepted` | Start Preparing → | `preparing` |
| `preparing` | Mark Out for Delivery → | `out_for_delivery` |
| `out_for_delivery` | Mark Delivered | `completed` |

### UI Details — Active Order Card

Similar to the pending card but without the countdown timer (active orders are no longer time-boxed) and with a single "Next Step" button. The current status is displayed as a step tracker (same visual language as the customer's `/orders/:id` page):

```
● Placed ── ● Accepted ── ◉ Preparing ── ○ Out for Delivery ── ○ Delivered
              (done)      (current)
```

### Testable Output
Accepted order → tap "Start Preparing" → card status updates, tracker advances one step → customer's `/orders/:id` page updates in real time. Progress through to `completed` → card disappears from Active Orders and appears in Today's History. Try to progress twice from the same tab rapidly → second click is blocked (button disabled during pending update). Open Supabase Dashboard and manually attempt `accepted → completed` via SQL → rejected by transition trigger.

---

## Step 6: Today's History Section (Completed / Declined / Expired)

### Why Sixth
A lightweight read-only record of what the owner has done today. Not a full analytics view — just "did we deliver that order we started an hour ago?". Full historical reporting is a v2 concern.

### Files to Update
- `src/pages/dashboard/OwnerDashboard.tsx` — add History section (collapsed by default)

### Supabase Query

```ts
// Today's terminal orders — IST day boundary.
// v1 assumption: all owners and customers are in Gudha Gorji (single region, IST).
// The owner device's local clock is trusted to be IST — acceptable at v1 scale.
// If the app ever serves multiple regions, switch to an explicit IST offset
// (Asia/Kolkata, UTC+05:30) instead of browser local.
const dayStart = new Date()
dayStart.setHours(0, 0, 0, 0)

const { data: history } = await supabase
  .from('orders')
  .select('id, status, total_amount, created_at, decline_reason')
  .eq('restaurant_id', restaurant.id)
  .in('status', ['completed', 'declined', 'expired'])
  .gte('created_at', dayStart.toISOString())
  .order('created_at', { ascending: false })
```

### DB Mapping

| What | Source |
|---|---|
| Table | `orders` |
| Statuses | Terminal states only (**Section 4.3.2**) |
| RLS | `orders_owner_select` (**Section 4.8.4**) |
| Index used | `(restaurant_id, status)` (**Section 4.7.7**) — serves the equality on `restaurant_id` + the `IN (...)` on `status`; remaining `created_at` ordering is a cheap sort over the already-narrow result set |
| Timezone | Browser local time for day boundary. **Explicit v1 assumption: single region (Gudha Gorji) on IST.** If v1 ever expands beyond a single timezone, switch to an explicit `Asia/Kolkata` offset instead of browser local. |

### UI Details

- Collapsed by default: header row says *"Today's history — 4 orders"*, tappable to expand
- When expanded: each row is a compact line (no item details) with:
  - Time (HH:MM IST)
  - Total (`₹440`)
  - Status badge (colours per Phase 1 UX doc Section 3.5)
  - For `declined`: the decline reason in slate, second line
- No click-through — read-only in v1. Owners looking up a specific historical order can use Supabase Dashboard.

### Empty State

*"No orders completed today yet."*

### Testable Output
Complete an order (Step 5) → it appears in History. Decline an order (Step 4) → appears with reason. Let one expire (Step 7) → appears with "Expired" badge. Refresh the page at midnight (or spoof the clock) → history resets to empty for the new day.

---

## Step 7: Realtime Strategy + Auto-Expiry Handling

### Why Seventh
Sections 3–6 each rely on Realtime doing the right thing. This step is where the subscription setup is consolidated and where the trickiest case — an order expiring *while the owner is looking at it* — is handled correctly.

### Files to Update
- `src/pages/dashboard/OwnerDashboard.tsx` — unify INSERT and UPDATE subscriptions into one channel

### Unified Subscription

```ts
useEffect(() => {
  if (!restaurant?.id) return

  const channel = supabase
    .channel(`owner-${restaurant.id}`)
    .on(
      'postgres_changes',
      {
        event: 'INSERT',
        schema: 'public',
        table: 'orders',
        filter: `restaurant_id=eq.${restaurant.id}`
      },
      handleOrderInsert   // Step 3 — re-fetch, append to pending
    )
    .on(
      'postgres_changes',
      {
        event: 'UPDATE',
        schema: 'public',
        table: 'orders',
        filter: `restaurant_id=eq.${restaurant.id}`
      },
      handleOrderUpdate   // this step — move between sections based on new status
    )
    .subscribe()

  return () => { supabase.removeChannel(channel) }
}, [restaurant?.id])
```

### Handling UPDATE Payloads

`payload.new` is the flat `orders` row — no joined `order_items`. Merge, don't replace, so the cached item data from the initial fetch survives. Same lesson as Phase 1 Step 7.

```ts
function handleOrderUpdate(payload: { new: Order, old: Order }) {
  const { status: newStatus } = payload.new

  // Locate the order in whichever section currently holds it
  const update = (order: Order) => ({ ...order, ...payload.new })

  if (newStatus === 'expired') {
    // Cron-driven: move from Pending → History with expired badge
    setPendingOrders(prev => prev.filter(o => o.id !== payload.new.id))
    setHistory(prev => [update(/* find it */), ...prev])
    return
  }

  if (['accepted', 'preparing', 'out_for_delivery'].includes(newStatus)) {
    // Either we just accepted (Step 4) or another device did —
    // make sure the card is in Active and not in Pending
    moveBetweenSections(payload.new.id, 'pending', 'active', update)
    return
  }

  if (['completed', 'declined'].includes(newStatus)) {
    moveBetweenSections(payload.new.id, 'active', 'history', update)
    return
  }
}
```

### Critical Cases

| Scenario | What happens | DB Section |
|---|---|---|
| Owner is viewing a `pending` card; cron fires at 15:00:01 | Realtime UPDATE fires with `status = 'expired'`; card moves to History with "Expired" badge; Accept/Decline buttons are no longer clickable because the card is no longer in Pending state | **4.7.5, 4.3.3** |
| Owner clicked Accept at 14:59:58; cron fires at 15:00:01 | The UPDATE from Step 4 arrives first (`pending → accepted`); card moves to Active. Cron's UPDATE sees `status = 'pending'` is now false and affects 0 rows. No conflict. | **4.7.5** |
| Owner has two dashboard tabs open; accepts from tab A | Tab B receives the UPDATE via Realtime; card moves from Pending to Active in both tabs. No duplicate action. | — |
| Realtime connection drops | Supabase client auto-reconnects with exponential backoff. On reconnect, re-fetch all three sections to catch up on any events missed during the outage. Hook this into the channel's `SUBSCRIBED` status event. | — |

### Reconnection Strategy

The Supabase Realtime client automatically reconnects. But when it does, it does **not** replay missed events. So on every `SUBSCRIBED` event after the first, trigger a full re-fetch of Pending / Active / History. This is a small cost (~3 queries) that guarantees the dashboard recovers from any network blip.

### Testable Output
Place a pending order → dashboard shows card. Manually run the `expire-orders` Edge Function (or wait 15 min) → card flips to History with expired badge without a refresh. Put phone/laptop in airplane mode for 30 seconds, then reconnect → dashboard's pending/active/history sections re-fetch and stay consistent with server state.

---

## Step 8: Testing & Edge Cases

### End-to-End Flows

| Flow | Steps |
|---|---|
| Happy accept path | Customer places order → appears in owner's Pending within 2s → owner accepts → customer sees "Order confirmed!" → owner progresses preparing → out_for_delivery → completed; customer UI updates at each step |
| Decline path | Customer places order → owner declines with reason → customer sees "Order declined. Reason: …" immediately |
| Auto-expiry path | Customer places order → owner does nothing for 15 min → cron fires → customer sees "Order expired"; owner's card moves to History |
| Open/Close toggle | Owner closes → within seconds, customer stops seeing the restaurant on `/restaurants`; menu page shows appropriate state if the customer already has it open |
| Multi-tab owner | Owner opens dashboard in 2 tabs → accepts in tab A → tab B reflects the change within 2s |
| Owner re-auth | Owner logs out from dashboard → redirected to `/login` → logs back in → lands on `/dashboard` |

### Edge Cases Mapped to DB Safeguards

| Test Case | Safeguard | Section |
|---|---|---|
| Customer accesses `/dashboard` directly | `<ProtectedRoute role="owner">` redirects to `/`; RLS would also reject queries | **4.8** |
| Owner attempts `pending → completed` (skip) | Status transition trigger rejects | **4.7.1** |
| Owner declines with empty / whitespace reason | DB CHECK rejects; modal prevents it | **4.7.3** |
| Owner accepts an order cron just expired | `.eq('status', 'pending')` → 0 rows → toast + re-fetch | **4.7.5** |
| Two tabs race to accept | Second tab gets 0 rows → treated as "already handled" | **4.7.5** |
| Owner flips `is_open = false` with pending orders outstanding | Existing pending orders remain actionable (restaurant closure doesn't cancel them); new orders cannot be placed per `orders` insert trigger | **4.7.2** |
| Owner tries to flip `is_active` | Frontend doesn't render the control. DB does **not** enforce column-level restriction in v1 — trusted owner pool; tracked in [`v2_deferred_issues.md`](v2_deferred_issues.md) §1 for hardening in v2. | **4.8.2** + `v2_deferred_issues.md` §1 |
| Realtime disconnect mid-shift | Auto-reconnect + re-fetch on `SUBSCRIBED` keeps data consistent | **Step 7** |
| `expire-orders` cron fails to run | Orders linger as `pending` past `expires_at`. Countdown shows 0 but UI waits for UPDATE. Monitoring: watch `expire-orders` Edge Function logs in Supabase Dashboard. | **4.7.5**, Pre-prod checklist §13 |
| Customer RLS policy leak | Customer directly queries `orders` for another restaurant → RLS returns zero rows (**Section 4.8.4**) | **4.8.4** |

---

## File Creation Map

| Step | Files Created / Modified | DB Tables Touched |
|---|---|---|
| 0 (prereq) | `supabase/migrations/004_owner_dashboard.sql` (new — adds `orders.customer_name`, `orders.customer_phone`; updates `place_order` RPC to snapshot them) | `orders` (schema + RPC) |
| 1 | `src/pages/login-signup/Login.tsx` (redirect update), `src/pages/auth/AuthCallback.tsx` (redirect update), `src/App.tsx` (add `/dashboard` route). `Navbar.tsx` is **verify-only** — owner state was already wired in Phase 1 Step 5. | `users` (via `AuthContext`) |
| 2 | `src/pages/dashboard/OwnerDashboard.tsx` + `.css` (new) | `restaurants` |
| 3 | `src/pages/dashboard/OwnerDashboard.tsx` (extended — Pending section, Realtime INSERT, countdown) | `orders`, `order_items`, `menu_items` |
| 4 | `src/pages/dashboard/OwnerDashboard.tsx` (extended — Accept handler), `src/pages/dashboard/DeclineModal.tsx` + `.css` (new) | `orders` |
| 5 | `src/pages/dashboard/OwnerDashboard.tsx` (extended — Active section + progression) | `orders` |
| 6 | `src/pages/dashboard/OwnerDashboard.tsx` (extended — History section) | `orders` |
| 7 | `src/pages/dashboard/OwnerDashboard.tsx` (unified Realtime channel + UPDATE handler + reconnect re-fetch) | `orders` (Realtime) |
| 8 | No new files — manual testing + edge case validation | All order-side tables |

All work lands in `src/pages/dashboard/OwnerDashboard.tsx` as a single, section-structured page. No additional routes, no extra components beyond the Decline modal. Matches the simplicity rule: "Owner dashboard = one page with sections (not 4 routes)."

---

## Phase 2 Exit Criteria

Per `redlotusfoods_documentation.md` **Section 7.4**:

> An owner can toggle open/close, see incoming orders in real time, accept/decline, progress orders through all statuses. Auto-expiry works.

Every DB safeguard from **Sections 4.7 and 4.8** relevant to the owner path must be validated:

- Status transition trigger (**4.7.1**) — invalid transitions rejected end-to-end
- Order placement trigger (**4.7.2**) — new orders rejected when `is_open = false` (verified from customer side)
- Decline reason CHECK (**4.7.3**) — empty reasons rejected at DB and blocked by modal
- Auto-expiry race guard (**4.7.5**) — Accept-after-expire affects 0 rows; UI handles gracefully
- Indexes on `(restaurant_id, status)`, `(customer_id, created_at DESC)`, and `(status, expires_at) WHERE status = 'pending'` (**4.7.7**) — confirmed present
- `restaurants_owner_update` (**4.8.2**) — owner can update rows they own. Column-level `is_active` restriction is frontend-only in v1; see [`v2_deferred_issues.md`](v2_deferred_issues.md) §1.
- `orders_owner_update` (**4.8.4**) — owner can update only their restaurant's orders
- Realtime INSERT + UPDATE subscriptions with `restaurant_id` filter — only owner's events arrive
- Realtime is enabled on `public.orders` in the Supabase Dashboard (Database → Replication)
- Migration 004 applied — `orders.customer_name` / `orders.customer_phone` populated by `place_order` RPC; owner dashboard reads these snapshot columns instead of joining `public.users`
- TypeScript types regenerated after migration 004 (`src/types/database.ts`); `Order` alias in `src/types/models.ts` includes `customer_name` and `customer_phone`
- Owner with no `restaurants` row sees the onboarding-incomplete screen, not a crash

**Pre-production checklist (`pre_production_checklist.md` Section 4) maps 1:1 to the above.** Ticking the Phase 2 exit criteria also ticks all of Section 4 of the pre-prod checklist.

---

## Key Improvements Over Section 7.4

`redlotusfoods_documentation.md` **Section 7.4** lists 6 bullet points in ~20 lines. This plan expands them into 8 ordered steps with four improvements:

**1. Navbar + route redirect as Step 1.** Section 7.4 assumes owners magically end up on `/dashboard`. In reality, `Login.tsx` and `AuthCallback.tsx` currently fall back to `/` for non-customer roles — Phase 1 only wired up the customer redirect. Fixing this before touching the dashboard page avoids a confusing "I logged in but nothing happened" experience.

**2. Countdown timer as a visual-only cue.** Section 7.4 lists "countdown timer to 15-min expiry" without specifying whether the timer *drives* expiry or *reflects* it. This plan makes it explicit: expiry is DB/cron, the timer is UX. Prevents a class of bugs where the frontend tries to flip status itself and desyncs from server state.

**3. Race-condition handling as a first-class concern (Steps 4 & 7).** The "accept-after-cron-expired" race is real — in a busy Gudha Gorji dinner service with the owner processing a queue of pending orders, this *will* fire. Using `.eq('status', 'pending')` + 0-row detection + re-fetch is the correct pattern, and Section 4.7.5 already supports it. This plan wires it in from the start rather than as a Phase 4 patch.

**4. Unified Realtime channel with reconnect re-fetch (Step 7).** Section 7.4 says "Realtime subscription" as a single line item. In practice, the dashboard needs two subscriptions (INSERT + UPDATE), shared state across three sections, and recovery from connection drops. Consolidating into one channel with explicit reconnect handling prevents the subtle "my dashboard went stale after the building WiFi flickered" bug that would otherwise surface only in production.
