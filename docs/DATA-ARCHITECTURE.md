# Data architecture — webshops + central product database

_Decision captured 2026-10 (La Capitana / Shipstore / Hooked Japan)._

## Principle

One **central product database** (dad's self-hosted app) is the single source of
truth for every store. Each store is just a "section" of that database. The
webshops are **headless Next.js** sites. **Shopify is used only for the
checkout/payment rail** — nothing else.

```
Hoenderop live prices
      │  (dad's app pulls, every few days, via API)
      ▼
Central database (MASTER)
  per store section: shipstore, hookedjapan, lacapitana, …
  per product: sku, barcode, title, brand, category,
               images, description, EVERY custom spec, live price
      │
      ├── exposes a read API  ──►  Next.js storefront (all display data)
      │
      └── price-sync job  ──►  Shopify (thin: sku + price + stock)  ──►  checkout / iDEAL
```

- **Next.js site** reads all catalog/display data (images, descriptions, the
  unlimited custom specs, price) from **dad's API**.
- **Shopify** holds only a thin variant per SKU (price + stock) so checkout can
  charge the right amount. All rich data stays in the database; it never needs
  to go into Shopify.
- The **SKU** is the glue that ties a database product to its Shopify checkout
  line.

## Why the price must still reach Shopify

Shopify charges whatever price sits on the Shopify variant at checkout. So the
few-days price refresh must **also push the new price into Shopify** (Admin API).
Rule to avoid a mismatch: the storefront shows the price that is current in
Shopify (the synced one), so what's displayed always equals what's charged.

## The API contract (dad's app exposes this)

Per store section, two read-only endpoints:

- `GET /api/{store}/products` → array of product objects
- `GET /api/{store}/products/{sku}` → one product object

Product object:

```json
{
  "sku": "A00004359",
  "barcode": "8712345678901",
  "title": "Werkdoek Viscose 38x38cm",
  "brand": "Hoenderop",
  "category": "onderhoud",
  "description": "…",
  "images": ["https://dads-server.nl/img/A00004359-1.jpg"],
  "priceExclVat": 12.34,
  "priceInclVat": 14.93,
  "currency": "EUR",
  "inStock": true,
  "specs": { "Materiaal": "Viscose", "Afmeting": "38x38cm", "Inhoud": "18 st" }
}
```

- `specs` is an **open object** — any number of custom fields; the storefront
  renders them all. This is how we get unlimited specs with zero Shopify
  involvement.
- `category` matches the storefront's category slugs.

## Shopify sync (price rail)

On the same few-days cadence, for each changed SKU, push to Shopify Admin API:
`{ sku, priceInclVat, inStock, title }`. Create the variant if it doesn't exist
yet. Everything else (images, description, specs) is NOT sent to Shopify.

Open choice: dad's app does this push itself, **or** a small separate sync
script sits between the database and Shopify.

## Storefront wiring (Next.js)

All of it flows through the single swap point **`lib/data.ts`**. Point its
getters at `dad's API` (shape above) and the whole site fills with real data.
For checkout, the cart uses the Shopify variant for each SKU → Shopify hosted
checkout.

## Open items

- Confirm dad's API base URL + auth (if any).
- Confirm each store has (or will have) its own Shopify store for checkout.
- Decide: dad's app pushes prices to Shopify, or a separate sync script does.
