# Competitor Pricing Data Model

Reference documentation for the `competitor-pricing-data.json` schema used by the competitor-pricing-tracker skill.

## Full Schema

```json
{
  "meta": {
    "product": "Your Product Name",
    "industry": "your industry/category",
    "unitName": "clients",
    "lastFullAudit": "YYYY-MM-DD",
    "defaultCurrency": "USD",
    "currencyConversions": {
      "GBP_to_USD": 1.34
    },
    "notes": "Any global notes about the dataset"
  },
  "platforms": [
    {
      "id": "slug-id",
      "name": "Display Name",
      "url": "https://platform.com",
      "pricingUrl": "https://platform.com/pricing",
      "pricingModel": "Human-readable description of the pricing model",
      "currency": "USD",
      "lastVerified": "YYYY-MM-DD",

      "tiers": [
        {
          "name": "Tier Name",
          "maxUnits": 50,
          "monthlyPrice": 49,
          "originalCurrencyPrice": 39,
          "originalCurrency": "GBP"
        }
      ],

      "overageRate": {
        "above": 50,
        "perUnit": 2.00,
        "note": "Explanation of overage calculation"
      },

      "features": {
        "featureKeyA": true,
        "featureKeyB": false
      },

      "addons": {
        "featureKeyB": 9
      },

      "featureMinPrice": {
        "featureKeyA": 69
      },

      "featureNotes": {
        "featureKeyA": "Only available on Pro tier ($69/mo) or above"
      },

      "highlightNote": "Optional standout differentiator for this platform"
    }
  ]
}
```

## Field Reference

### `meta` — Dataset Metadata

| Field | Type | Required | Description |
|---|---|---|---|
| `product` | string | Yes | The user's own product name (for context) |
| `industry` | string | Yes | The competitive category (e.g., "project management software") |
| `unitName` | string | Yes | What the pricing scales by (e.g., "clients", "seats", "users", "contacts", "projects"). Used to label `maxUnits` and `perUnit` in reports |
| `lastFullAudit` | date string | Yes | Last time ALL platforms were verified in a single run |
| `defaultCurrency` | string | Yes | ISO 4217 currency code used for price normalization |
| `currencyConversions` | object | No | Exchange rates used for conversion (e.g., `"GBP_to_USD": 1.34`). Keys follow `{FROM}_to_{TO}` format |
| `notes` | string | No | Any global notes about the dataset |

### `platforms[]` — Individual Competitor Entries

| Field | Type | Required | Description |
|---|---|---|---|
| `id` | string | Yes | Unique slug identifier (lowercase, hyphenated) |
| `name` | string | Yes | Display name |
| `url` | string | Yes | Product homepage URL |
| `pricingUrl` | string | Yes | Direct link to the pricing page (for re-scraping) |
| `pricingModel` | string | Yes | Human-readable pricing model description (e.g., "Flat tiers by seat count", "Usage-based per API call", "Custom / sales-only") |
| `currency` | string | Yes | ISO 4217 currency code prices are stored in |
| `lastVerified` | date string | Yes | When this platform's pricing was last verified |

### `tiers[]` — Pricing Tiers

| Field | Type | Required | Description |
|---|---|---|---|
| `name` | string | Yes | Tier name (e.g., "Starter", "Pro", "Enterprise") |
| `maxUnits` | number \| null | Yes | Max units (see `meta.unitName`) on this tier. `null` = unlimited |
| `monthlyPrice` | number | Yes | Monthly price in `meta.defaultCurrency`. `0` for free tiers |
| `originalCurrencyPrice` | number | No | Price in the platform's native currency (only if different from `defaultCurrency`) |
| `originalCurrency` | string | No | ISO 4217 code of the original currency |

### `overageRate` — Per-Unit Pricing Above Top Tier

Nullable — set to `null` if the platform has no overage model (e.g., unlimited top tier, or enterprise-only above the cap).

| Field | Type | Description |
|---|---|---|
| `above` | number | Unit threshold above which overage pricing applies |
| `perUnit` | number | Cost per additional unit (see `meta.unitName`) |
| `note` | string | Explanation of how overage is calculated |

### `features` — Feature Availability

Object mapping feature keys (defined during bootstrap) to booleans.
- `true` = the platform offers this feature on at least one paid tier
- `false` = the platform does not offer this feature at all

Feature keys are project-specific — they're defined during the bootstrap phase based on what matters in the user's industry. All platforms must use the same set of keys for consistent comparison.

### `addons` — Optional Paid Add-ons

Object mapping feature keys to their **monthly add-on price**. Only include entries for features that cost extra on top of the base tier price.

Example: `{ "advancedAnalytics": 15 }` means advanced analytics is a $15/mo add-on.

### `featureMinPrice` — Tier-Gated Features

Object mapping feature keys to the **minimum monthly tier price** required to access that feature. Only include entries for features that aren't available on all tiers.

Example: `{ "sso": 99 }` means SSO is only available on tiers costing $99/mo or more.

### `featureNotes` — Feature Availability Notes

Object mapping feature keys to human-readable notes about availability, limitations, or caveats. Useful for nuance that booleans can't capture.

### `highlightNote`

Optional string. A standout differentiator or unique selling point for this platform worth noting in competitive comparisons.

## Handling Non-Tier Pricing Models

Not all competitors use straightforward tier-based pricing. Here's how to represent other models:

### Usage-based pricing (per-API-call, per-GB, per-transaction)
Represent each usage bracket as a "tier" where the tier name describes the usage level:
```json
{
  "pricingModel": "Usage-based per API call with volume brackets",
  "tiers": [
    { "name": "Up to 10K calls/mo", "maxUnits": null, "monthlyPrice": 29 },
    { "name": "Up to 100K calls/mo", "maxUnits": null, "monthlyPrice": 99 },
    { "name": "Up to 1M calls/mo", "maxUnits": null, "monthlyPrice": 299 }
  ]
}
```
Set `maxUnits` to `null` since the scaling dimension isn't the same as `meta.unitName`. The tier names carry the usage details.

### Fully custom / sales-only pricing
```json
{
  "pricingModel": "Custom pricing — sales conversation required",
  "tiers": [],
  "overageRate": null,
  "featureNotes": {
    "pricing": "No public pricing. Sales-only. Estimated range: $X–$Y/mo based on review sites."
  }
}
```

### Flat-rate (single price, no tiers)
```json
{
  "pricingModel": "Flat rate — single plan",
  "tiers": [
    { "name": "Standard", "maxUnits": null, "monthlyPrice": 49 }
  ]
}
```

### Freemium + paid tiers
Include the free tier explicitly with `monthlyPrice: 0`:
```json
{
  "tiers": [
    { "name": "Free", "maxUnits": 5, "monthlyPrice": 0 },
    { "name": "Pro", "maxUnits": null, "monthlyPrice": 29 }
  ]
}
```

## Design Principles

1. **Include the user's own product** in the platforms array — makes side-by-side comparison easy
2. **Normalize all prices** to one currency (the `defaultCurrency`). Store original prices separately when conversion is needed
3. **`null` means unlimited** in `maxUnits` and means "no overage model" in `overageRate`
4. **Feature keys are consistent** across all platforms — every platform entry must have the same set of keys in `features`
5. **Don't guess enterprise pricing** — use `null` with a note rather than an estimate
6. **Monthly prices only** — if a platform shows annual pricing, convert to monthly equivalent and note the annual discount in `featureNotes` or the tier data
7. **`unitName` is global** — it defines what `maxUnits` and `perUnit` mean across the dataset (e.g., "clients", "seats"). If a competitor uses a different scaling dimension, explain the mismatch in their `pricingModel` description
