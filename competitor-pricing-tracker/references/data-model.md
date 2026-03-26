# Competitor Pricing Data Model

Reference documentation for the `competitor-pricing-data.json` schema used by the competitor-pricing-tracker skill.

## Full Schema

```json
{
  "meta": {
    "product": "Your Product Name",
    "industry": "your industry/category",
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
          "upToClients": 50,
          "monthlyPrice": 49,
          "originalCurrencyPrice": 39,
          "originalCurrency": "GBP"
        }
      ],

      "overageRate": {
        "aboveClients": 50,
        "perClient": 2.00,
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
| `product` | string | Yes | Your own product name (for context) |
| `industry` | string | Yes | The competitive category (e.g., "project management software") |
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
| `pricingModel` | string | Yes | Human-readable pricing model description (e.g., "Flat tiers by seat count") |
| `currency` | string | Yes | ISO 4217 currency code prices are stored in |
| `lastVerified` | date string | Yes | When this platform's pricing was last verified |

### `tiers[]` — Pricing Tiers

| Field | Type | Required | Description |
|---|---|---|---|
| `name` | string | Yes | Tier name (e.g., "Starter", "Pro", "Enterprise") |
| `upToClients` | number \| null | Yes | Max users/clients/seats on this tier. `null` = unlimited |
| `monthlyPrice` | number | Yes | Monthly price in `meta.defaultCurrency`. `0` for free tiers |
| `originalCurrencyPrice` | number | No | Price in the platform's native currency (only if different from `defaultCurrency`) |
| `originalCurrency` | string | No | ISO 4217 code of the original currency |

### `overageRate` — Per-Unit Pricing Above Top Tier

Nullable — set to `null` if the platform has no overage model (e.g., unlimited top tier, or enterprise-only above the cap).

| Field | Type | Description |
|---|---|---|
| `aboveClients` | number | Threshold above which overage pricing applies |
| `perClient` | number | Cost per additional user/client/seat |
| `note` | string | Explanation of how overage is calculated |

### `features` — Feature Availability

Object mapping feature keys (defined during bootstrap) to booleans.
- `true` = the platform offers this feature on at least one paid tier
- `false` = the platform does not offer this feature at all

Feature keys are project-specific — they're defined during the bootstrap phase based on what matters in your industry.

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

## Design Principles

1. **Include your own product** in the platforms array — makes side-by-side comparison easy
2. **Normalize all prices** to one currency (the `defaultCurrency`). Store original prices separately when conversion is needed
3. **`null` means unlimited** in `upToClients` and means "no overage model" in `overageRate`
4. **Feature keys are consistent** across all platforms — every platform entry should have the same set of keys in `features`
5. **Don't guess enterprise pricing** — use `null` with a note rather than an estimate
6. **Monthly prices only** — if a platform shows annual pricing, convert to monthly equivalent and note the annual discount in `featureNotes` or the tier data
