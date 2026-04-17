# Phase 2 — Owner Dashboard: UX/UI Design Document

> **Version:** 1.0
> **Date:** 2026-04-17
> **Author:** Ankit Khatkar
> **Status:** Design — ready for build
> **Reference:** `phase2_owner_dashboard_plan.md`, `phase1_ux_ui_design.md`, `design.md`, `CLAUDE.md`
> **Route:** `/dashboard` (one page, three stacked sections — Pending / Active / History)

---

## 0. Context & Design Goals

The Owner Dashboard is the operational cockpit for a restaurant owner in Gudha Gorji. It is used during service — phone in one hand, ladle in the other, often on a 4G connection in a hot kitchen. The screen gets a glance every 30–60 seconds, and every glance must answer three questions in under two seconds:

1. **Is there a new order I need to act on?** (Pending section, countdown-driven urgency)
2. **What am I currently cooking or delivering?** (Active section, one next-step button)
3. **What did I do today?** (History, collapsed ledger)

**Design tenets (non-negotiable):**

- **One primary action per card.** No overflow menus, no dropdowns, no "more" links.
- **Glanceable over pretty.** Typographic hierarchy wins over imagery. No food photos on cards.
- **Thumb-first, not mouse-first.** 44×44px minimum tap targets. Bottom of viewport is the "safe zone".
- **Real time is the default; refresh is a bug.** The screen must never require the owner to pull-to-refresh.
- **DB is the source of truth.** Timer is visual; cron decides expiry. Server rejects invalid transitions; the UI just renders what the DB allows.

---

## 1. Page Structure

### 1.1 Layout Hierarchy (top → bottom)

```
┌──────────────────────────────────────────┐
│ 1. Navbar (shared, owner state)          │ ← global, 64px
├──────────────────────────────────────────┤
│ 2. Restaurant Header Bar                 │ ← sticky below nav, 72px
│    • Name + cuisine                      │
│    • Open/Closed toggle (primary)        │
│    • "Last changed" timestamp            │
├──────────────────────────────────────────┤
│ 3. Pending Orders ⚠                      │ ← highest priority
│    • Section title + live count badge    │
│    • Stacked order cards (countdown)     │
├──────────────────────────────────────────┤
│ 4. Active Orders                         │ ← in-progress work
│    • Section title + count               │
│    • Stacked order cards (progression)   │
├──────────────────────────────────────────┤
│ 5. Today's History (collapsed by default)│ ← low priority
│    • Tappable row: "Today — 4 orders"    │
│    • Expanded: compact line items        │
├──────────────────────────────────────────┤
│ 6. Footer (minimal: support contact)     │ ← do NOT reuse FooterCTA
└──────────────────────────────────────────┘
```

### 1.2 Information Priority Rules

| Priority | Content | Visual weight |
|---|---|---|
| P0 | Pending order countdown + Accept/Decline | Largest typography (28px timer), red CTA, section rendered first even if empty |
| P1 | Open/Closed toggle | Sticky header — always visible while scrolling |
| P2 | Active order next-step button | Medium typography (16px), red CTA |
| P3 | Today's History | Collapsed by default; single line until expanded |
| P4 | Everything else (addresses, notes, timestamps) | Slate 14px, secondary position within card |

### 1.3 Section Grouping Rationale

- **Pending → Active → History** matches the owner's mental model of *"needs attention → in progress → done"*. No reordering, ever.
- Pending is always rendered, even when empty, so the owner can glance at the screen and confirm *"nothing waiting"* without scrolling.
- History collapses because it's retrospective — reviewing it should be an intentional action, not clutter during peak hours.
- **Never use tabs.** Tabs would hide Pending when the owner is on Active. A new order must be visible without changing views.

---

## 2. Component-Level Design

### 2.1 Navbar (Owner State)

Inherited from Phase 1 Step 5 — **verify-only, do not rebuild**.

```
DESKTOP (≥ 768px):
┌────────────────────────────────────────────────────────────┐
│ 🪷 RedLotus  |  Dashboard              Profile  [Log Out] │
└────────────────────────────────────────────────────────────┘

MOBILE (< 768px):
┌─────────────────────────────────┐
│ 🪷 RedLotus              [☰]    │
└─────────────────────────────────┘
  └─ open: Dashboard / Profile / Log Out
```

No cart, no "Restaurants" link, no "My Orders". Owners see only operational links.

---

### 2.2 Restaurant Header Bar (Sticky)

Sticks below the Navbar. Always visible while the owner scrolls through orders. **This is the only sticky element on the page** — no sticky order cards, no sticky CTAs.

#### Desktop (≥ 768px)

```
┌─────────────────────────────────────────────────────────────────────┐
│  Shree Bhojnalaya                              [ ● Open for Orders ]│
│  North Indian · Main Market, Gudha Gorji       Last changed 2:45 PM │
└─────────────────────────────────────────────────────────────────────┘
```

#### Mobile (< 768px)

```
┌──────────────────────────────────┐
│  Shree Bhojnalaya                │
│  North Indian                     │
│                                  │
│  [ ●──── Open for Orders ]      │  ← full-width toggle
│  Last changed 2:45 PM            │
└──────────────────────────────────┘
```

**Toggle specs:**

| State | Track colour | Knob | Label | Helper text |
|---|---|---|---|---|
| `is_open = true` | `red` (#D63031) | Right, white | **Open for Orders** (white on red) | "Last changed {HH:MM AM/PM}" |
| `is_open = false` | Grey (#E8E2DC) | Left, white | **Closed** (slate) | "Last changed {HH:MM AM/PM}" |
| `is_active = false` (admin-locked) | Grey, disabled | Left, no-hover | **Closed** | "Your restaurant isn't active yet. Please contact admin." (amber, 13px) |
| Pending flip (optimistic) | Slight opacity 0.6, knob mid-travel | — | — | Spinner inline: "Updating…" |

**Grouping rules:**
- Restaurant name + cuisine + address form one typographic block (24px serif + 14px slate), left-aligned.
- Toggle + timestamp form the second block, right-aligned on desktop, full-width on mobile.
- 24px horizontal padding, 16px vertical. White background, 1px bottom border (`#E8E2DC`).

**CTA placement:** The toggle is the *only* interactive control in the header. Do not add "Refresh" buttons, "Settings" gears, or "Logout" — those belong in the Navbar.

---

### 2.3 Pending Order Card

The single most important component on the dashboard.

#### Mobile (360px)

```
┌────────────────────────────────────┐
│  ⏱ 12:34           [ PENDING ]     │ ← countdown left, status badge right
│                                    │
│  ₹790 · 2 items                    │ ← charcoal 20px 700 — money first
│                                    │
│  🟢 Paneer Tikka × 2      ₹440    │ ← items, 15px
│  🔴 Mutton Rogan Josh × 1 ₹350    │
│                                    │
│  ────────────────────────────      │
│                                    │
│  👤 Ramesh Kumar                   │ ← customer snapshot
│  📞 +91 63789 39472     [ Call ]   │ ← tappable tel: link
│  📍 123, Main Market,              │
│     Gudha Gorji                    │
│  📝 Extra spicy, call on arrival   │ ← only if special_instructions
│                                    │
│  ─────────── 16px ───────────      │
│                                    │
│  [  Decline  ]   [    Accept    ]  │ ← 44px tall, side-by-side
└────────────────────────────────────┘
```

#### Desktop (≥ 1024px)

Same card, max-width 640px, horizontal padding 24px. Buttons stay side-by-side (not stacked). Cards stack vertically — no multi-column grid; kitchen owners track a queue, not a mosaic.

#### Visual Hierarchy (top-down eye path)

1. **Timer + status badge** (row 1) — tells the owner *"how urgent is this?"*
2. **Total + item count** (row 2) — tells them *"how big is this order?"*
3. **Item lines** (row 3) — *"what do I need to cook?"*
4. **Customer block** (rows 4–6) — *"who do I call if needed?"*
5. **CTAs** (row 7) — *"what do I do?"*

A tired owner at 9 PM should be able to read rows 1–2 in one glance and act on row 7 without scrolling.

#### Spacing & Grouping

| Element | Spacing |
|---|---|
| Card outer padding | 16px mobile, 24px desktop |
| Card-to-card gap | 12px |
| Section divider inside card (hr) | 16px vertical margin, border `#E8E2DC` |
| Row gap (item lines, customer lines) | 6px |
| Action row top margin | 16px |
| Button gap | 12px |

Cards have `border-radius: 12px`, `border: 1px solid #E8E2DC`, `background: white`, `box-shadow: 0 2px 8px rgba(26,26,26,0.04)`.

**Urgency escalation** (when timer < 1 min): card gains a 2px red left border and a subtle 1.5s pulse on the card background (`#fdf8f6` → `#FDEDEC` → `#fdf8f6`). No full-card flashing — too distracting during service.

#### CTA Placement & Behaviour

- **Decline:** ghost button (transparent bg, 1px slate border, charcoal text). Left position — secondary action.
- **Accept:** red filled button (`#D63031` bg, white text, 600 weight). Right position — primary action, aligned to the thumb on right-handed mobile use.
- Mobile: buttons equal width, 50/50 split with 12px gap.
- Desktop: buttons auto-width with 16px padding left/right, min-width 140px. Accept is visually heavier.
- Both buttons are **48px tall on mobile** (44px minimum enforced by WCAG; we add 4px for greasy fingers).

---

### 2.4 Active Order Card

Same skeleton as the Pending card with three differences: no countdown, a status step tracker replaces the badge, and one "Next Step" button replaces the Accept/Decline pair.

#### Mobile (360px)

```
┌────────────────────────────────────┐
│  Order #A47F · 2:45 PM             │ ← short ID + created_at time
│  ● ── ● ── ◉ ── ○ ── ○             │ ← step tracker, charcoal dots
│  Placed Accepted Preparing          │
│                                    │
│  ₹790 · 2 items                    │
│                                    │
│  🟢 Paneer Tikka × 2               │ ← prices omitted here (already accepted)
│  🔴 Mutton Rogan Josh × 1          │
│                                    │
│  ────────────────────────────      │
│                                    │
│  👤 Ramesh Kumar                   │
│  📞 +91 63789 39472     [ Call ]   │
│  📍 123, Main Market               │
│  📝 Extra spicy                    │
│                                    │
│  [  Mark Out for Delivery →    ]   │ ← full-width red button, 48px
└────────────────────────────────────┘
```

#### Step Tracker

5 nodes: **Placed → Accepted → Preparing → Out for Delivery → Delivered**. Nodes connected by 2px horizontal lines.

| Node state | Dot | Label colour |
|---|---|---|
| Completed | Filled charcoal 10px dot | Slate |
| Current | Ring with filled centre, red (`#D63031`), subtle 1.5s pulse | Charcoal, 600 weight |
| Future | Hollow 10px circle, `#E8E2DC` border | `#C4BAB0` |

Connector line between two completed nodes is charcoal; otherwise `#E8E2DC`.

#### Next-Step Button — Label by Current Status

| Current status | Button label | Next status |
|---|---|---|
| `accepted` | **Start Preparing →** | `preparing` |
| `preparing` | **Mark Out for Delivery →** | `out_for_delivery` |
| `out_for_delivery` | **Mark Delivered ✓** | `completed` |

Single full-width button on mobile. Desktop: right-aligned, min-width 240px. Always red filled — there is no secondary action on an active card.

#### Why no Cancel / Back button

V1 does not support cancellation or status reversal (enforced by the status transition trigger, Section 4.7.1). Rendering a button the DB would reject would be a trust-breaking failure. If the owner marks "Out for Delivery" by mistake, they continue to "Delivered" — or call the customer. This is an accepted v1 trade-off.

---

### 2.5 History Row (Collapsed & Expanded)

The History section is a *ledger*, not a dashboard. It's designed to be scanned once at end of day.

#### Collapsed (Default)

```
┌────────────────────────────────────┐
│  Today's History — 4 orders    [▼] │ ← 48px tall, tappable anywhere
└────────────────────────────────────┘
```

#### Expanded

```
┌────────────────────────────────────┐
│  Today's History — 4 orders    [▲] │
├────────────────────────────────────┤
│  2:45 PM  ₹790   [DELIVERED]       │ ← 40px row, compact
│  1:20 PM  ₹450   [DECLINED]        │
│           "Dal Makhani unavailable" │ ← decline reason, slate 13px
│  12:05 PM ₹320   [EXPIRED]         │
│  11:40 AM ₹580   [DELIVERED]       │
└────────────────────────────────────┘
```

- Rows have no tap action (read-only in v1).
- Status badges reuse the Phase 1 style (Section 3.5 of `phase1_ux_ui_design.md`).
- Decline reason is the only multi-line row — second line, slate 13px, 16px left indent.
- Divider between rows: 1px `#E8E2DC`.

---

### 2.6 Decline Modal

Centre-screen dialog, backdrop at `rgba(26,26,26,0.4)`. No portal — rendered in the dashboard tree, state-driven.

#### Mobile (360px)

```
╔════════════════════════════════╗ ← backdrop (dim)
║                                ║
║ ┌────────────────────────────┐ ║
║ │  Decline this order?       │ ║ ← DM Serif 22px
║ │                            │ ║
║ │  Tell the customer why —   │ ║ ← slate 14px
║ │  they'll see this message. │ ║
║ │                            │ ║
║ │  ┌──────────────────────┐  │ ║
║ │  │ e.g. "Dal Makhani   │  │ ║ ← textarea, 3 rows, autofocus
║ │  │  unavailable today"  │  │ ║
║ │  └──────────────────────┘  │ ║
║ │  0 / 200                   │ ║ ← slate 12px, updates live
║ │                            │ ║
║ │  [ Cancel ] [ Confirm ]    │ ║
║ └────────────────────────────┘ ║
║                                ║
╚════════════════════════════════╝
```

#### Rules

- **Autofocus** on textarea when modal opens.
- Confirm button disabled until `reason.trim().length >= 3`.
- Max 200 characters; counter turns amber at 180+.
- Confirm is red filled; Cancel is ghost with 1px slate border.
- **Escape** closes (Cancel action). Click on backdrop closes too.
- Modal width: 320px mobile, 440px desktop. Always vertically centred.
- On confirm: button shows inline spinner and disables both buttons until DB responds.

---

## 3. Interaction Design

### 3.1 Open/Close Toggle

| Trigger | Behaviour |
|---|---|
| Tap toggle | Immediate optimistic flip — track colour + knob position animate over 150ms |
| DB success | `Last changed …` timestamp updates, no toast |
| DB failure | Toggle flips back with a 150ms ease-in; toast: "Couldn't update. Try again." (red) |
| `is_active = false` | Toggle is disabled (not hidden). Tapping it is a no-op; helper text explains why |

Double-tapping is safe — second tap fires after the first's optimistic flip, so the DB sees the intended final state.

### 3.2 Accept Flow

1. Owner taps **Accept**. Button immediately: `[ ✓ Accepting… ]`, disabled, inline spinner.
2. DB UPDATE with `.eq('status', 'pending')` race guard fires.
3. **Success (1+ rows):** Realtime UPDATE arrives (possibly before the RPC's promise resolves — harmless). Card animates out of Pending (slide-left 200ms, fade), then mounts in Active with a brief 300ms fade-in.
4. **0-row result (race lost):** Button resets, toast: *"This order is no longer pending. Refreshing…"* — trigger full re-fetch of all three sections.
5. **Network error:** Button resets, toast: *"Couldn't accept. Check connection."* (red).

### 3.3 Decline Flow

1. Tap **Decline** on a pending card → modal opens, textarea autofocused.
2. Owner types reason (e.g., *"Out of paneer tonight"*), Confirm enables at 3 chars.
3. Tap **Confirm** → button disables, spinner.
4. DB UPDATE with race guard.
5. **Success:** Modal closes (150ms fade), card slides out of Pending into History. Toast: *"Order declined."* (default style).
6. **0-row result:** Modal closes, toast: *"This order is no longer pending. Refreshing…"*
7. **Network error:** Modal stays open, error banner inside modal: *"Couldn't send. Try again."*

### 3.4 Next-Step Button (Active cards)

1. Tap **Start Preparing →** (or the corresponding label).
2. Button: `[ Updating… ]`, spinner, disabled.
3. DB UPDATE with `.eq('status', currentStatus)` race guard.
4. **Success:** Step tracker animates forward (connector fills charcoal over 300ms; next dot pulses). Button label updates to the next transition. On final tap (→ `completed`), card slides out to History.
5. **Race lost:** Toast + re-fetch. Unusual but can happen in multi-tab scenarios.
6. **Network error:** Button resets; toast: *"Couldn't update. Try again."*

### 3.5 Loading States

| Context | Display |
|---|---|
| Initial page load | Full-page skeleton: sticky header placeholder + "Pending" title + 2 card skeletons + "Active" title + 1 card skeleton + collapsed History row skeleton |
| Card fetching line items after Realtime INSERT | Card mounts with `— Loading items —` placeholder (text only, no shimmer) — replaced within ~200ms when the join query returns |
| Button submitting (Accept / Decline / Next Step) | Inline spinner inside button, button disabled, label swaps to present-continuous verb ("Accepting…", "Updating…") |
| Toggle flipping | Knob mid-travel with opacity 0.6, "Updating…" helper text below |

**No full-page spinners.** They block the whole screen — unacceptable when new orders may arrive during a fetch.

### 3.6 Error States

| Error | Presentation |
|---|---|
| Initial restaurant fetch fails | Full-page error block (centred): "Couldn't load your dashboard. [Try Again]" |
| Realtime subscription fails after multiple reconnect attempts | Persistent banner under the sticky header: *"⚠ Live updates disconnected. We'll keep trying. [Reload]"* (amber, 48px tall, non-dismissable until reconnected) |
| Race condition (0-row UPDATE) | Toast + automatic re-fetch. No destructive UI — never show "Order disappeared" in scary language |
| Validation (decline reason too short) | In-modal: button stays disabled; no red error text (the disabled button communicates clearly enough). Character counter turns amber if user types then deletes |
| Auth expired mid-session | Redirect to `/login` via AuthContext — no custom handling on this page |

### 3.7 Real-Time Updates — Visual Behaviour

| Event | UI response | Sound (v1) |
|---|---|---|
| New `pending` INSERT | New card slides in from top of Pending section (250ms slide-down + fade). Card highlights with a 1s red-tinted background flash, then settles. Section count badge increments. | None (deferred to Phase 4) |
| `pending → accepted/declined/expired` from another tab or cron | Card slides out of Pending (200ms), slides into Active or History. No highlight flash — the action has already been observed elsewhere. | None |
| `accepted → preparing → out_for_delivery → completed` from another tab | Step tracker animates the new state; if terminal, card slides to History. | None |
| Countdown reaches 00:00 | Timer freezes at 00:00, turns struck-through grey. Card stays visible until the `expired` UPDATE arrives (usually within 60s via cron). | None |
| Realtime reconnect after drop | Brief toast (top-right): *"Reconnected."* (green, 2s). Behind the scenes: full re-fetch. | None |

**Critical:** New-order flash is the *only* attention-grabbing animation on the dashboard. Overusing animation trains the owner to ignore it.

---

## 4. Mobile UX Optimisation

The dashboard is designed mobile-first. Kitchen owners will use phones more than laptops.

### 4.1 Tap Targets

| Element | Min size | Notes |
|---|---|---|
| Open/Closed toggle | 48px × 88px | Track width generous — easy to hit while moving |
| Accept button | 48px tall, 50% width of card | Right-side (thumb zone) |
| Decline button | 48px tall, 50% width of card | Left — slightly out of thumb zone on purpose, to reduce accidental declines |
| Next-Step button (Active) | 48px tall, full card width | Can't miss it |
| Call button | 44px × 72px min | Inline with phone number |
| `tel:` phone link | Entire phone-number row is tappable — not just the number text | 44px row height |
| History expand/collapse | Entire row (48px tall) | Icon is visual only; tap anywhere |
| Card body | **Not tappable** | Only buttons trigger actions — prevents accidental open when scrolling |

### 4.2 Thumb-Friendly Design

- Primary CTA (Accept, Next Step) is always on the **right half** of the card — reachable by a right-handed thumb.
- Secondary CTA (Decline) is on the **left** — a deliberate asymmetry: the more destructive action requires a conscious reach.
- The sticky Open/Closed toggle sits at the **top** of the viewport, but since the owner scrolls downward through orders, they can tap it without scroll-back (sticky).
- History expand row sits at the **bottom** — it's rarely tapped during service, so bottom placement is fine.

### 4.3 Scrolling Behaviour

- **Single-page scroll.** No horizontal scroll anywhere. No nested scroll containers (the whole page scrolls; section headers scroll with content).
- Sticky header stays pinned; Pending section starts immediately below (no whitespace trap).
- Pending and Active sections always render, even when empty, so section titles provide spatial anchors as the owner scrolls.
- Pull-to-refresh is **not implemented**. Realtime + auto-reconnect make it redundant; implementing it would train owners to distrust the Realtime layer.

### 4.4 Sticky Elements

| Element | Sticky? | Why |
|---|---|---|
| Navbar | Yes (inherited) | Global app convention |
| Restaurant Header Bar (toggle) | **Yes** | Owner must always see and control open/close state |
| Section titles ("Pending", "Active", "History") | No | They scroll with content. Counts stay visible because Pending stays at top. |
| CTAs (Accept / Decline / Next Step) | No | They're card-local; stickying them would confuse ownership |
| Toast | No (auto-positioned overlay, not sticky) | Bottom-centre, 24px from bottom edge |

Maximum of **two** sticky layers at once: Navbar (64px) + Restaurant Header (72px on desktop, 104px on mobile). Combined ~136px mobile occlusion is acceptable.

### 4.5 Tel: Link Behaviour

- Phone number renders as `<a href="tel:+916378939472">+91 63789 39472</a>` — native dialler opens on tap.
- "Call" button wraps the same `tel:` — identical target, larger tap area.
- Desktop users see the Call button as a visible CTA (not a no-op) — tapping it triggers their OS protocol handler (Skype, FaceTime, etc.) or opens nothing silently. Acceptable; most owners are on mobile.

### 4.6 Viewport Considerations

- Test on **360×640** (common low-end Android). On this size, one full Pending card + Navbar + Header fit without scroll — the owner sees the whole card including CTAs at once.
- At **320px** (iPhone SE / old Android), card padding drops to 12px, button labels stay full-text (no icon-only). Accept at this width is still 48px tall.
- Landscape orientation is not optimised — owners overwhelmingly use portrait. The design does not break in landscape, but multi-column layouts only activate at ≥1024px width.

---

## 5. Visual Design System

Inherits **all tokens** from `phase1_ux_ui_design.md` Section 2. Nothing new is introduced except a Phase-2-specific countdown timer spec.

### 5.1 Colours (Purposeful Use)

| Token | Hex | Where it appears in Phase 2 |
|---|---|---|
| `red` (#D63031) | — | Open toggle track, Accept button, Next Step button, urgency timer (< 1 min), card left-border at peak urgency |
| `redDark` (#B71C1C) | — | Hover state on all red CTAs |
| `charcoal` (#1A1A1A) | — | Restaurant name, order totals, customer names, step tracker "current" label |
| `slate` (#4A4A4A) | — | Timestamps, addresses, special instructions, "Last changed" helper, decline reason in history |
| `border` (#E8E2DC) | — | Card borders, dividers, hollow step-tracker nodes, Closed toggle track |
| `warmBg` (#fdf8f6) | — | Page background (alternates with white inside cards) |
| `statusPending` (#F39C12) | — | PENDING badge background, amber warning text ("restaurant not active") |
| `statusActive` (#3498DB) | — | Out-for-delivery step tracker node |
| `statusSuccess` (#27AE60) | — | DELIVERED badge in history, "Reconnected" toast |
| `statusError` (#E74C3C) | — | DECLINED / EXPIRED badges in history, urgency flash tint |
| `nonVegRed` / `vegGreen` | — | Item indicator dots on all three card types |

**No hex value is used without being mapped to one of these tokens.** If a design moment doesn't fit, update this table; don't invent a colour in CSS.

### 5.2 Typography Hierarchy

| Role | Font / Weight | Size | Example |
|---|---|---|---|
| Restaurant name (header) | DM Serif Display 400 | 24px | "Shree Bhojnalaya" |
| Section titles | Plus Jakarta 700 | 20px | "Pending Orders" |
| Order total + item count | Plus Jakarta 700 | 20px | "₹790 · 2 items" |
| Countdown timer | Plus Jakarta 700 tabular-nums | 28px | "12:34" |
| Card body item lines | Plus Jakarta 500 | 15px | "🟢 Paneer Tikka × 2" |
| Customer name | Plus Jakarta 600 | 15px | "Ramesh Kumar" |
| Phone / address | Plus Jakarta 400 | 14px | "+91 63789 39472" |
| Special instructions | Plus Jakarta 400 italic | 14px | "Extra spicy, call on arrival" |
| Secondary meta (timestamps) | Plus Jakarta 400 | 13px | "Ordered 2:45 PM" |
| Buttons | Plus Jakarta 600 | 15px | "Accept" |
| Status badges | Plus Jakarta 600 uppercase | 12px | "PENDING" |
| Helper text (toggle, errors) | Plus Jakarta 400 | 13px | "Last changed 2:45 PM" |

**`tabular-nums`** on the countdown is critical — without it, the timer visibly jitters as digits change width.

### 5.3 Status Colour Map

| UI state | Background | Text | Label |
|---|---|---|---|
| PENDING badge | `#FEF3E2` | `#F39C12` | PENDING |
| Step tracker: current node | white | `#D63031` | (ring colour) |
| Active card body | white | charcoal | — |
| DELIVERED badge | `#E8F8F0` | `#27AE60` | DELIVERED |
| DECLINED badge | `#FDEDEC` | `#E74C3C` | DECLINED |
| EXPIRED badge | `#FDEDEC` | `#E74C3C` | EXPIRED |

Badges share exactly the customer-side styles from Phase 1 Section 3.5 — an owner and a customer should see the same colour for the same status. Single visual language.

### 5.4 Countdown Timer — Visual States

| Remaining | Colour | Weight | Behaviour |
|---|---|---|---|
| > 5 min | slate `#4A4A4A` | 700 | Static |
| 1–5 min | amber `#F39C12` | 700 | Static |
| < 1 min | red `#D63031` | 700 | 1.5s opacity pulse (0.7 → 1.0), card gains 2px red left border |
| 0:00 / negative | `#C4BAB0` | 700 | Struck-through, non-pulsing; card retains red border until cron flips status |

### 5.5 Button Styles

| Variant | Bg | Border | Text | Use |
|---|---|---|---|---|
| **Primary red filled** | `#D63031` | none | white 600 | Accept, Next Step, Confirm Decline, "Try Again" |
| **Ghost / secondary** | transparent | 1px `#4A4A4A` | charcoal 600 | Decline, Cancel, Call |
| **Destructive disabled** | `#F5F0ED` | 1px `#E8E2DC` | `#C4BAB0` 600 | Any button in pending/submitting state |
| **Icon + label** (Call) | white | 1px `#E8E2DC` | charcoal 600 + phone icon | Inline with phone rows |

All buttons: `border-radius: 10px`, `padding: 12px 24px` (regular) / `16px 32px` (large CTAs). Hover (desktop): primary darkens to `#B71C1C`, ghost gains 1px solid border and `#F5F0ED` bg. Focus: 3px `#D63031` outline with 2px white offset (keyboard accessibility).

### 5.6 Elevation & Borders

- Cards: `1px solid #E8E2DC`, `border-radius: 12px`, `box-shadow: 0 2px 8px rgba(26,26,26,0.04)`.
- Modal: `box-shadow: 0 12px 48px rgba(26,26,26,0.18)`, no border.
- Sticky header: `border-bottom: 1px solid #E8E2DC`, no shadow (shadow competes with pending-card shadows).

---

## 6. Edge Case UX

### 6.1 Order Expired While Viewing

**Scenario:** Owner is reading a pending card. The 15-min countdown ticks to 00:00. Cron fires within 60s and flips the row to `expired`.

- Before cron fires: timer shows `00:00` struck-through, card has red left border, Accept/Decline buttons **stay enabled** (the DB will reject via race guard if the owner taps Accept).
- If owner taps Accept in this window: 0-row result → toast *"This order was expired. Refreshing…"* → card slides out of Pending into History.
- When Realtime UPDATE arrives: card animates out (200ms slide-left + fade), re-mounts in History with EXPIRED badge. No toast on passive expiry — the card motion is feedback enough.

**Never** grey out Accept/Decline on a pending card based on the local timer. The local clock might be wrong; DB is truth.

### 6.2 Two Tabs Open

**Scenario:** Owner has dashboard open on phone and laptop. Accepts from the laptop.

- Laptop: optimistic card motion → Active section within ~200ms. Toast: *"Order accepted."*
- Phone: Realtime UPDATE arrives within ~1s. Card slides from Pending to Active silently (no toast — the owner didn't act here).
- Both tabs stay in sync. Any subsequent Next-Step action behaves identically.

**Scenario:** Both tabs tap Accept within the same second.

- First to reach the DB: 1-row UPDATE → success path.
- Second: 0-row UPDATE → toast on that tab only: *"This order is no longer pending. Refreshing…"* → re-fetch shows the card already in Active.

### 6.3 No Restaurant (Onboarding Incomplete)

**Scenario:** Owner signs up before admin has created their `restaurants` row.

```
┌──────────────────────────────────┐
│        👋                         │
│                                  │
│  Your restaurant isn't set up    │ ← DM Serif 24px, charcoal
│  yet.                            │
│                                  │
│  We're finishing your            │ ← slate 15px
│  onboarding. Please contact      │
│  Ankit on WhatsApp and he'll     │
│  get your restaurant live        │
│  within 24 hours.                │
│                                  │
│  [ 📱 Open WhatsApp →  ]         │ ← primary red, wa.me link
│                                  │
│  Or email admin@redlotusfoods.in │ ← slate 13px, mailto link
└──────────────────────────────────┘
```

- Centred card, max-width 480px, 48px vertical padding.
- Rendered *instead of* the three sections — Navbar and page shell stay visible.
- No sticky header (there's no restaurant to toggle).
- No spinner, no retry button — the fix is human, not automated.

### 6.4 No Orders States

| Section | Empty-state text |
|---|---|
| Pending (empty) | *"No pending orders. You'll see new orders here as they come in."* — slate 14px, centred in the section, 48px vertical padding |
| Active (empty) | *"No active orders right now."* — slate 14px, centred |
| History (empty, collapsed) | Header reads *"Today's History — 0 orders"* with the expand chevron greyed out (non-tappable) |
| History (empty, after force-expand attempt) | *"No orders completed today yet."* (But since the chevron is disabled at 0, this state is not normally reachable.) |

**Never** show an illustration or hero icon in Pending/Active empty states — during service, empty just means *"nothing waiting"*, and a big illustration makes the owner wonder if something is broken.

### 6.5 Restaurant Closed (is_open = false) With Active Orders Outstanding

Closing the restaurant **does not** cancel in-flight orders (Section 4.7.2 — the `is_open` check only blocks new INSERTs). All existing Active cards remain actionable. The sticky header shows "Closed"; the sections below continue to function normally.

No UI warning banner for this — it's the correct behaviour. Adding *"You still have active orders"* text every time the owner closes creates false urgency.

### 6.6 Realtime Disconnected

After 3 failed reconnect attempts (~15s of outage), show a persistent amber banner below the sticky header:

```
┌─────────────────────────────────────────┐
│ ⚠ Live updates disconnected.            │ ← amber bg, charcoal text
│   We'll keep trying.      [Reload Page] │
└─────────────────────────────────────────┘
```

- Banner is non-dismissable. It vanishes automatically when Realtime reconnects.
- On reconnect: banner fades out + brief green toast *"Reconnected."* (2s).
- **Reload Page** is a safety valve — not the primary recovery mechanism.

### 6.7 Owner's Clock Skewed

Local clock 10 minutes ahead → countdown shows negative on cards that are still genuinely pending.

- UI treats negative timer as "Expired (locally)" — struck-through, red border on card.
- Accept/Decline still work. DB race guard is clock-independent.
- No attempt to reconcile client and server clocks in v1. If owners report issues, revisit in Phase 4.

### 6.8 Very Long Customer Name / Address / Instructions

- Customer name: single-line truncation with ellipsis at 28 chars (mobile) / 40 chars (desktop). Full value accessible via tooltip (`title` attr) on desktop.
- Delivery address: wraps freely, max 3 lines, then ellipsis. Tap the address row to expand (mobile) — simple CSS `line-clamp` toggle.
- Special instructions: always fully shown. If it exceeds 4 lines, the card grows — do not truncate delivery-critical context.

### 6.9 Accept Button Pressed Twice Rapidly

Button is disabled immediately on first click (state change precedes DB call). Second click hits the disabled button — no-op. Cannot race with itself.

### 6.10 Order Has Zero Items (Malformed)

Should be impossible (checkout validates at least one item + DB trigger recomputes total). If it occurs, card renders with:

- Total from `orders.total_amount` (even if 0).
- Item section: *"— Loading items —"* if the join returned nothing; never *"No items"* (that's alarming).
- Accept/Decline still work — this is a data hygiene issue, not an operational one.

---

## 7. Microcopy

All copy is written in plain English, short sentences, no jargon. Hindi localisation deferred — layouts allow for ~30% string expansion.

### 7.1 Button Text

| Button | Label |
|---|---|
| Open toggle (closed) | **Closed** (left label) |
| Open toggle (open) | **Open for Orders** |
| Accept order | **Accept** |
| Decline order | **Decline** |
| Confirm decline (modal) | **Confirm Decline** |
| Cancel decline (modal) | **Cancel** |
| Next step (accepted) | **Start Preparing →** |
| Next step (preparing) | **Mark Out for Delivery →** |
| Next step (out_for_delivery) | **Mark Delivered ✓** |
| Call customer | **Call** |
| Expand history | **Today's History — {n} orders** |
| Retry after error | **Try Again** |
| Reload (banner) | **Reload Page** |
| Onboarding CTA | **Open WhatsApp →** |

### 7.2 Empty States

| State | Copy |
|---|---|
| Pending empty | No pending orders. You'll see new orders here as they come in. |
| Active empty | No active orders right now. |
| History empty | No orders completed today yet. |
| Onboarding (no restaurant) | Your restaurant isn't set up yet. / We're finishing your onboarding. Please contact Ankit on WhatsApp and he'll get your restaurant live within 24 hours. |

### 7.3 Error Messages

| Trigger | Message |
|---|---|
| Accept on expired / already-handled order | This order is no longer pending. Refreshing… |
| Decline on expired order | This order is no longer pending. Refreshing… |
| Next-step race failure | This order's status changed. Refreshing… |
| Open/Close toggle fails | Couldn't update. Try again. |
| Accept network error | Couldn't accept. Check your connection. |
| Decline network error (inside modal) | Couldn't send. Try again. |
| Decline reason too short | *(No text — disabled button is the signal.)* |
| Initial dashboard load fails | Couldn't load your dashboard. [Try Again] |
| Realtime disconnected banner | ⚠ Live updates disconnected. We'll keep trying. |
| Restaurant inactive helper text | Your restaurant isn't active yet. Please contact admin. |

### 7.4 Toast Messages

| Trigger | Message | Style |
|---|---|---|
| Order accepted | Order accepted. | Default (charcoal bg) |
| Order declined | Order declined. | Default |
| Status progressed | *(No toast — tracker animation is the feedback.)* | — |
| Open/Close toggled | *(No toast — "Last changed" update is the feedback.)* | — |
| Realtime reconnected | Reconnected. | Success (green) |
| Network error | Connection lost. Retrying… | Error (red) |
| Race condition re-fetch | This order is no longer pending. Refreshing… | Amber (warning) |

Toast positioning: bottom-centre on mobile (24px from bottom, above home-indicator if PWA installed), top-right on desktop (≥1024px). 3s auto-dismiss. Tap-to-dismiss anywhere on the toast body.

### 7.5 Placeholder & Helper Text

| Context | Copy |
|---|---|
| Decline modal textarea placeholder | e.g. "Dal Makhani unavailable today" |
| Decline modal body | Tell the customer why — they'll see this message. |
| Decline character counter (normal) | {n} / 200 |
| Toggle helper (active, open) | Last changed {HH:MM AM/PM} |
| Toggle helper (inactive) | Your restaurant isn't active yet. Please contact admin. |

### 7.6 Voice & Tone

- **Direct, not cheery.** "Order accepted." not "Yay, you accepted an order!"
- **Imperative for actions.** "Accept", "Decline", "Mark Delivered" — not "You can accept this".
- **Never blame the user.** "Couldn't update. Try again." is fine. *"You failed to…"* or *"Invalid action"* is not.
- **Time in IST, 12-hour.** "2:45 PM" not "14:45" and not "2:45 PM IST" (the region is implicit — all owners are in Gudha Gorji).
- **Currency in ₹ with no decimals for whole numbers.** `₹790` not `₹790.00`. If a total includes paise (unlikely in v1), show `₹790.50`.

---

## 8. Accessibility

Inherits `phase1_ux_ui_design.md` Section 7. Owner-dashboard-specific additions:

| Requirement | Implementation |
|---|---|
| Live status announcements | The Pending count + new-order flash is mirrored in an `aria-live="polite"` region: *"New pending order from Ramesh Kumar, ₹790."* Fires once per INSERT. |
| Countdown timer | Wrapped in `aria-label="12 minutes 34 seconds until expiry"` — updated every 10s (not every 1s, to avoid screen-reader spam). At < 1 min, switch to `aria-live="assertive"` to surface urgency. |
| Modal focus trap | Decline modal traps focus between textarea / Cancel / Confirm. Escape closes. Returns focus to the Decline button on close. |
| Tab order | Sticky header (toggle) → section titles (not tabbable) → each card's Decline → Accept (or Next Step). History expand last. |
| Keyboard equivalents | `Enter` / `Space` on toggle flips it. `Enter` on Accept/Decline triggers the handler (mobile owners may use Bluetooth keyboards rarely, but desktop owners reliably will). |
| Colour independence | Veg/non-veg dots also carry a shape border (green-bordered square around green dot, red-bordered around red). Step tracker nodes use filled vs hollow, not colour-only. Urgency timer pulses (motion) in addition to colour. |

---

## 9. Responsive Breakpoints

| Element | < 480px | 480–768px | 768–1024px | > 1024px |
|---|---|---|---|---|
| Page horizontal padding | 12px | 16px | 24px | 32px (content max-width 720px) |
| Card layout | Full width | Full width | Full width, max 640px centred | Full width, max 640px centred (single-column always) |
| Sticky header | Stacked (name above toggle) | Stacked | Single row | Single row |
| Card action row | Buttons 50/50 side-by-side | Buttons 50/50 side-by-side | Buttons auto-width right-aligned | Auto-width right-aligned |
| Decline modal | 88% viewport width, max 320px | 320px | 440px | 440px |
| Toast position | Bottom-centre | Bottom-centre | Top-right | Top-right |
| Countdown timer size | 24px | 28px | 28px | 32px |

**Single-column stacked cards at all breakpoints.** Do not add a two-column grid on desktop — a queue reads top-to-bottom, not left-to-right. Desktop users get larger cards, not more cards per row.

---

## 10. Design Decisions Log

| Decision | Rationale |
|---|---|
| Three stacked sections on one page (not tabs) | A new pending order must be visible even when the owner is looking at active orders. Tabs would hide it. |
| Single-column card layout at all breakpoints | Owners process a queue. Reading order must be unambiguous. Two columns on desktop creates F-pattern scanning that breaks priority. |
| No food photos on cards | Cards are operational, not marketing. Photos slow 4G loads and add visual noise that distracts from total/items/CTA. |
| Accept on the right, Decline on the left | Right-handed thumb bias for the affirmative action. Destructive action requires a deliberate reach. |
| Countdown is visual only; cron drives expiry | Frontend clocks lie. If UI flipped status itself, two tabs could show different states. DB-as-truth keeps the mental model simple. |
| No accept/decline on expired-looking cards until DB confirms | Owner's clock may be ahead. Trusting the UI timer over the DB would cause legitimate orders to be blocked. |
| History collapsed by default | During service, history is noise. Expand is one tap when the owner wants to review. |
| No click-through on history rows | V1 owners review history for "did this go through" — a list view answers that. Detail pages add routes and RLS surface with no v1 ROI. |
| No notification sound in v1 | Mobile push needs real-device testing across Android/iOS/PWA/background/lock-screen states. Too much risk to ship without it. Countdown provides visual urgency instead. Phase 4 handles audio. |
| Sticky Restaurant Header instead of floating button for open/close | The toggle is central to the owner's day. Making it persistently visible (and labelled) is more discoverable than a FAB. |
| Customer phone is a `tel:` link + visible Call button | Belt-and-braces: the `tel:` link works for anyone who knows phone numbers are tappable, the button works for everyone else. |
| No confirmation modal for Accept | Friction on the fast-path would cost seconds × dozens of orders/day. Accept is reversible by Decline-before-accept-arrives only; once accepted, the owner commits. This matches the real-world kitchen commitment (food starts being made). |
| Confirmation modal for Decline | Decline is a message to the customer. Requiring a reason both meets the DB CHECK constraint and prevents a misclick from damaging customer trust. |
| No "Refresh" button anywhere | A visible refresh button tells owners "the live updates might be broken". We don't want that signal. The Reload banner only appears when Realtime is actually dead. |
| `tabular-nums` on countdown | Without it, "12:34" → "12:33" causes horizontal jitter — owners perceive this as the screen being "unstable". |
| Footer is minimal (support contact only) | Reusing Phase 1's `FooterCTA` (with landing-page marketing) on an operational dashboard is jarring. A one-line support link is enough. |

---

## 11. Implementation-Ready Cheat Sheet

For the engineer building `OwnerDashboard.tsx` and `DeclineModal.tsx`:

- **Page structure:** 1 `<main>`, 3 `<section>` (Pending, Active, History). History wrapped in a `<details>`-style component (collapsible).
- **State shape:** `{ restaurant, pendingOrders, activeOrders, history, declining, isToggleUpdating, realtimeStatus }`.
- **Card keys:** always `order.id` (UUID from DB). Never array index.
- **Animations:** use `framer-motion` `layout` + `AnimatePresence` for card slide-between-sections. If adding motion library is out of scope, use CSS `transition: all 200ms ease` on `transform` + `opacity` with a brief `position: absolute` during unmount — simpler but jumpier.
- **Tel: link format:** `tel:+91${phone.replace(/\D/g, '').slice(-10)}` — strip formatting, prepend +91. Handles pasted numbers with spaces / dashes.
- **Counter for Decline:** `reason.length / 200` — count chars, not bytes.
- **Countdown ticker:** single `useEffect` at the dashboard level updating a `now` state every 1000ms. All cards derive remaining time from `(new Date(order.expires_at).getTime() - now)`. Do **not** create a setInterval per card.
- **aria-live region for new orders:** `<div aria-live="polite" className="sr-only">{latestAnnouncement}</div>` — populated by the INSERT handler, cleared after 5s.
- **Realtime channel name:** `owner-${restaurant.id}` — consistent with the plan.
- **Toast library:** reuse whatever Phase 1 picked; otherwise a minimal custom `Toast` with a `ToastHost` in `App.tsx` is fine. Keep the API to `toast.success(msg)` / `toast.error(msg)` / `toast.warning(msg)`.
- **CSS file:** `OwnerDashboard.css` co-located. All Phase-2 tokens live in the same `:root` as Phase 1 — no new stylesheets at the app level.

---

## 12. Phase 2 Design Exit Criteria

Design is complete when:

- [ ] Every screen in Section 2 renders at 360px, 768px, and 1280px without horizontal scroll.
- [ ] Every interactive element in Section 3 has a hover (desktop), focus (keyboard), pressed (mobile), disabled, and loading state designed.
- [ ] Every empty state in Section 6.4 has final copy from Section 7.2.
- [ ] Every error in Section 3.6 maps to final copy in Section 7.3.
- [ ] Every toast in Section 3.7 maps to final copy in Section 7.4.
- [ ] Contrast checked: all text ≥ 4.5:1, status badges ≥ 3:1 against their bg, countdown colours ≥ 4.5:1 against white card.
- [ ] Motion reduced-motion alternative specified: if `prefers-reduced-motion`, disable pulse, card-flash, and slide-between-sections; replace with instant position swaps + no colour flash.
- [ ] Tap-target audit at 320px viewport — no interactive element below 44×44px.
