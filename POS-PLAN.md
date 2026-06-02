# Elyna Footwear — POS scope & pricing plan

Status: **proposal / not built.** Drafted 2026-06-02 after Elyna asked "do you do POS?". Decide scope with her (open questions at the bottom) before building.

## TL;DR

She already owns ~70% of a POS — the admin records sales, decrements stock, and shows daily/weekly/monthly takings. The gap is a **fast in-shop checkout flow** and **M-Pesa payment capture**. Build that as an extension of the existing system, not a new product. Skip custom hardware POS (point her to Loyverse if she needs barcodes/receipt printer/cash drawer).

## What she already has (live today)

- Per-size **stock** per item; "Record sale" decrements it.
- **Sales ledger** (`sales[]`: size, qty, price, buyer name/phone, notes, soldAt) — editable + undo (returns stock).
- **Sales dashboard**: Today / Week / Month / All-time KPIs + top categories + recent sales.
- **Clients/CRM** roster + WhatsApp re-marketing to past buyers.
- **Inventory dashboard**: units, value, low/out-of-stock.

So "I want to see my sales and what's in stock" = already done. The POS ask is about the **moment of sale in the shop**.

---

## Tier 1 — "Sell in store" mode (POS-lite)  ← recommended core

A single fast checkout screen in the admin, reusing the existing sales+inventory engine.

**UX:** big search/tap grid of items → tap item → pick size → qty (default 1) → **payment: Cash / M-Pesa** → optional buyer phone → **Record sale**. Stock decrements, sale lands in the dashboard, screen resets for the next customer. Optimised for speed and a phone/tablet at the counter.

**Data change:** add `paymentMethod` ('cash'|'mpesa') and `channel` ('shop'|'online') to each `sales[]` entry. Backwards-compatible (old entries just lack the fields).

**New dashboard value it unlocks:** today's takings **split Cash vs M-Pesa**, count of sales, shop-vs-online split. That's the daily-reconciliation number a shop owner actually wants at closing.

**Effort:** low–moderate. The hard parts (inventory, sales ledger, `apiMutateAndPublish`, dashboard) already exist. Mostly a new front-end screen + two optional fields + a split in the KPI calc. ~half a day to a day.

## Tier 2 — M-Pesa auto-capture (Daraja STK Push)  ← real upsell

Cashier enters amount + buyer phone → system sends an **STK push** (the "enter M-Pesa PIN" prompt) to the buyer's phone → on success, Safaricom calls our worker back and the sale auto-marks **paid + recorded**. No manual "did it come through?" checking.

**Requirements (client-side, gating):**
- Elyna needs a **Lipa na M-Pesa Till (Buy Goods) or Paybill** in the business name.
- A **Daraja** developer app (Consumer Key/Secret) + the **Lipa Na M-Pesa passkey**, and **production go-live approval** from Safaricom (sandbox is instant; production takes a few days + paperwork).

**Build:** worker endpoints `POST /api/mpesa/stk` (initiate) + `POST /api/mpesa/callback` (Safaricom confirmation → mark sale paid). Daraja OAuth token caching. The Worker is already publicly reachable, so it's a clean callback host.

**Effort:** moderate, but **calendar-gated by Safaricom go-live**, not by our code. Flag that to her: the integration is a few days of build, plus Safaricom's approval window we don't control.

**Caveat:** if she just wants buyers to send to a number and she eyeballs the SMS, that's not auto-reconcilable — STK Push (or C2B with a real Till) is what makes it automatic. Set that expectation.

## Tier 3 — Receipts (optional, cheap)

- **WhatsApp receipt:** after a sale, one tap sends the buyer a formatted "thanks + items + total" message (reuses the wa.me flow already on the site).
- **Printable receipt:** a clean print-CSS receipt for a normal/thermal printer via the browser. No driver work.

**Effort:** low. Add once Tier 1 exists.

## Explicitly OUT of scope (don't custom-build)

Barcode scanners, cash drawers, dedicated offline till software, multi-cashier shift management. For a single Moi Avenue shop this is over-engineering and we'd own the maintenance. If she truly needs full hardware POS, recommend **Loyverse** (free, phone/tablet, barcodes + receipts) or a **Safaricom Till** — and keep our system as the catalog + sales-intelligence layer on top.

---

## Rough pricing (confirm exact numbers with Joel — do not quote without checking)

Consistent with the Ksh 5,000/mo catalog model:
- **Tier 1 (Sell-in-store mode):** small one-off setup OR a modest monthly bump folded into her existing sub. It's an extension, price it as one.
- **Tier 2 (M-Pesa Daraja):** higher one-off (more build + Safaricom go-live handholding), optionally + a small monthly for the running integration.
- **Tier 3 (receipts):** throw-in / minor.
Anchor everything as "added to the system you already pay for," not a separate SaaS.

## Open questions for Elyna (ask before building)

1. How do you sell in the shop now — do you write it down, use a phone, a notebook, anything?
2. Do you want it to **just record** each sale fast, or to **actually take the M-Pesa payment** through the system?
3. Do you have a **Lipa na M-Pesa Till / Paybill** in the business name already? (Gates Tier 2.)
4. Do you need a **printed receipt**, or is a WhatsApp message to the customer enough?
5. One person at the counter, or several? (We assume one.)

## Implementation notes (for when it's greenlit)

- Reuse `apiMutateAndPublish(mutator)` for every counter sale (fetch-merge-publish; never blind overwrite).
- `sales[]` entry gains `{ paymentMethod, channel }`; KPI calc groups by `paymentMethod` for the Cash/M-Pesa split; default missing → treat as cash/online.
- Keep the counter screen its own admin section (`#posDash`) above the dashboards; reuse `.field`, modal, and toast styling.
- Daraja: module-level token cache in the worker (OAuth expires hourly); store `MPESA_CONSUMER_KEY/SECRET/PASSKEY/SHORTCODE` as worker secrets (same pattern as ADMIN_TOKEN); validate the callback against the checkout request id before marking paid.
