<p align="center">
  <img src="./assets/wordmark.png" alt="TransparentCars" width="360">
</p>

# TransparentCars Public API

**Affordable used cars — no hidden costs.** A key-free JSON API for the Portuguese
local used-car market, from [transparent.pt](https://transparent.pt). It serves the same
data the public site runs on: live stock, a fair-price valuation, an annual road-tax (IUC)
lookup, and consent-gated lead capture.

- **Base URL:** `https://transparent.pt/api/public`
- **Auth:** none — public and key-free by design.
- **Format:** JSON. `GET` for reads, one `POST` for leads.
- **Spec:** [`openapi.yaml`](./openapi.yaml) (OpenAPI 3.1).
- **Docs:** https://transparent.pt/en/api
- **MCP server:** a TransparentCars MCP is on the way — it wraps these same endpoints so an
  AI assistant can search stock, value a car and estimate road tax. See `../mcp`.

## What it is

TransparentCars works the **local** Portuguese used-car market — it does **not** import.
So this API is about the questions a local budget buyer actually asks: *what's for sale,
is the price fair, and what will the car cost me to run?* No import tax, no despachante
workflow — just stock, fair price, and road tax.

## Endpoints

| Method & path | What it does |
|---|---|
| `GET /cars?site=transparentcars&lang=en` | Published stock. Filters: `make`, `fuel`, `gearbox`, `max_price`, `limit`, `offset`. |
| `GET /cars/{slug}` | Full detail for one car (specs, description, photos, YouTube). |
| `GET /valuation?make=&model=&year=&km=` | **Fair-price band** from comparable Portuguese listings. Optional: `fuel`, `gearbox`, `hp`. |
| `GET /isv?cc=&co2=&fuel=&year=` | **Annual road tax** — read the `iuc` field (EUR/year). |
| `GET /car_refs?make=` | Canonical make/model vocabulary for `/valuation`. |
| `POST /lead` | Consent-gated contact / callback request. |

### Quick start

```bash
BASE=https://transparent.pt/api/public

# Published stock (English labels)
curl "$BASE/cars?site=transparentcars&lang=en"

# Is 8900 EUR fair for a 2015 Clio at 118k km?
curl "$BASE/valuation?make=Renault&model=Clio&year=2015&km=118000&fuel=diesel"

# Annual road tax (IUC) — read the .iuc field
curl "$BASE/isv?cc=1200&co2=120&fuel=petrol&year=2016" | jq '.iuc'

# Vocabulary for the valuation endpoint
curl "$BASE/car_refs?make=BMW"
```

### Road tax (IUC)

`GET /isv` returns an `iuc` object — the vehicle's **annual circulation tax** (IUC), the
number that matters for running-cost planning:

```json
{
  "iuc": { "iuc": 214.23, "exempt": false, "category": "B",
           "taxa_cilindrada": 127.35, "taxa_co2": 65.15, "coef": 1.15,
           "diesel_adic": 0.0, "co2_adic": 0.0 }
}
```

Pure-electric vehicles come back `"exempt": true`, `"iuc": 0`. The figure is a maintained
estimate cross-checked against public tax tables — use it for planning, not as an official
assessment. The response also carries an `isv` object for completeness; for local running
costs, read `iuc`.

### Leads

`POST /lead` requires explicit `consent: true` and at least one contact channel
(`phone` or `email`):

```bash
curl -X POST "$BASE/lead" -H 'Content-Type: application/json' -d '{
  "kind": "contact",
  "name": "Maria Silva",
  "email": "maria@example.com",
  "message": "Is the Clio still available?",
  "lang": "en",
  "consent": true
}'
```

Without `consent: true` the request is rejected (`422 {"detail":"consent required"}`).

## Rate limits

Public and key-free, protected only by **per-IP fair-use** limits. Exceeding them returns
`429`; the road-tax endpoint includes a `Retry-After` (seconds) header. Calculator and
valuation responses may be cached at the edge, so repeated identical calls are cheap.

Ballpark: road tax ~60 req/min/IP (cached), valuation and leads ~20 req / 5 min / IP.
These are generous for real use — batch or cache on your side if you need more.

## Notes

- **`site` defaults to `transparentcars`.** Unknown site slugs return `404`.
- **`lang`** accepts `pt`, `en`, `ru`, `ua` (Ukrainian) and localises human-readable labels.
- **Photo paths** in car responses are **site-relative** (`/photos/…`) — prefix with
  `https://transparent.pt`.
- **No PII except on `/lead`.** Every other endpoint takes vehicle parameters and returns
  numbers or public listing data.

## License

API responses and this specification are published under
[CC BY 4.0](https://creativecommons.org/licenses/by/4.0/). Attribute **TransparentCars
(transparent.pt)**.

Questions: [info@transparent.pt](mailto:info@transparent.pt) ·
[transparent.pt/en/api](https://transparent.pt/en/api)
