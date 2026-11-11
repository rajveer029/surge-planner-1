# Hospital Surge Planner API

A FastAPI service for turning recent weather, air quality, and disease surveillance data into hospital surge recommendations. It calculates resource multipliers, translates them into staffing and procurement needs, and returns a readable briefing. OpenRouter can optionally interpret free-text events and generate the briefing.

The HTTP application is defined in `api/plan.py`; the calculation logic is in `surge_planner.py`. The repository includes a Dockerfile for running the service on port `8000` and a sample caller targeting the Railway deployment.

## API endpoints

| Method | Path | Purpose |
| --- | --- | --- |
| `GET` | `/` | Health check. Returns `{ "ok": true }`. |
| `POST` | `/plan` | Accepts one JSON object containing the planning inputs and nested `baselines`. Returns `translated` and `briefing`. |

The [sample caller](https://github.com/rajveer029/surge-planner-1/blob/main/sample-use/calling_api.py) uses `http://surge-planner-production.up.railway.app/plan`. This is the address recorded in the repository; confirm the current Railway URL and HTTPS availability in your deployment before using it in a frontend. Locally, the endpoint is `http://localhost:8000/plan`. FastAPI also exposes interactive documentation at `/docs` when the service is running.

## Workflow

1. `build_multiplier_plan(inputs, ...)` returns `(plan, features, env_blocks)`.
2. `apply_multipliers(plan, baselines, ...)` returns staffing rosters, purchase orders, and peak infrastructure needs.
3. `make_human_nlg(...)` produces the `briefing` string. If OpenRouter is unavailable, it falls back to `human_readable(...)`.

The planner applies fixed rules and thresholds. Where triggers affect the same target **in the same window**, it keeps the largest multiplier, capped at `2.0` by default. Windows are kept separate. The module does not train or run a statistical forecasting model.

## `POST /plan` request body

```json
{
  "meta": { "today": "2026-11-11" },
  "weather_past_7d": [],
  "visits_daily": [],
  "epi_signals": { "by_disease_daily": [] },
  "events": [],
  "free_text_events": [],
  "baselines": {
    "departments": {},
    "bundles_per_day": {},
    "diagnostics_per_day": {},
    "inventory_state": {},
    "infra": {}
  }
}
```

The body is **flat**: `meta`, weather, visits, epidemiology, events, and `baselines` are all top-level keys. `api/plan.py` passes this whole object to `build_multiplier_plan` and passes its nested `baselines` object to `apply_multipliers`. Do not wrap the body in `inputs`. The route does not read an `options` object.

### Top-level planning fields

| Field | Type | Format and behavior |
| --- | --- | --- |
| `meta.today` | string | Planning date in ISO `YYYY-MM-DD` format. Supply it for a meaningful `valid_for` range. An invalid date raises an error. |
| `weather_past_7d` | array of objects | Daily observations in **oldest-to-newest** order. Send seven rows for the intended seven-day calculations. The code uses every supplied row for rain and humidity, the last row for temperature, and the last three for PM2.5. Weather row dates are ignored. |
| `visits_daily` | array of objects | Total outpatient visits by date. Dates must match the disease rows for OPD share calculations. Duplicate dates overwrite earlier totals. |
| `epi_signals.by_disease_daily` | array of objects | Disease observations grouped by exact disease name and sorted by date. The latest seven rows per disease are the active period; the preceding seven are compared only when at least 14 rows exist. |
| `events` | array | Passed to the environmental rule evaluator but **not used by its current conditions**. It has no effect on the plan. |
| `free_text_events` | array of strings | Optional descriptions parsed only when `ai_events` is true. Each accepted AI result contributes multipliers. Without a key or valid result, the event is skipped. |

Absent collection fields default to empty arrays or objects in the script. The script does **not** validate request bodies, so the HTTP layer should enforce these types, date format, and sensible numeric ranges.

### Weather observation

```json
{ "rain_mm": 10, "humidity_pct": 82, "pm25": 210, "temp_c": 30 }
```

| Field | Expected value | Rule |
| --- | --- | --- |
| `rain_mm` | Nonnegative number, millimetres | Total across supplied rows ≥ 70 triggers `RAINY_WEEK` (`D7–D14`). |
| `humidity_pct` | Number, percent | Mean across supplied rows ≥ 80, together with temperature ≥ 28 °C, triggers `HUMID_HEAT` (`D0–D5`). |
| `pm25` | Nonnegative numeric PM2.5 value | Each of the last three rows ≥ 200 triggers `AQI_SPIKE` (`D2–D5`). The code compares PM2.5 directly; it does not compute an AQI index. |
| `temp_c` | Number, °C | Only the final row is used for `HUMID_HEAT`. |

Missing weather values are treated as zero. The module does not check chronological order, units, or the number of rows.

### Visits and disease observations

```json
{ "date": "2026-11-11", "total_opd": 100 }
```

```json
{
  "date": "2026-11-11",
  "disease": "dengue",
  "cases": 3,
  "tests": 10,
  "positives": 2
}
```

| Field | Expected value | Use |
| --- | --- | --- |
| `date` | ISO date string | Matches disease rows to visit totals and sorts each disease's rows. |
| `total_opd` | Nonnegative number | Daily outpatient denominator. Missing or zero values contribute no denominator. |
| `disease` | Case-sensitive string | Exact disease identifier. Use one of the supported names below for a resource action. |
| `cases` | Nonnegative number | Daily cases; missing or `null` counts as zero. |
| `tests` | Nonnegative number, optional | Test count for positivity; zero total tests gives `null` positivity. |
| `positives` | Nonnegative number, optional | Positive count for positivity. |

For each disease, the code sums cases in its latest seven **rows** (or fewer if less data exists), divides by matched OPD visits, and calculates a change in case share against the preceding seven rows when available. A disease is flagged when its minimum case count is met **and at least one** configured case-share, growth, or applicable positivity threshold is met. Without a previous comparison period, a nonzero current share yields a computed growth of 100%; send 14 daily rows for meaningful week-over-week comparisons. A flagged disease affects resources only if it has an action mapping.

Supported identifiers (exact spelling and case):

```text
dengue                  malaria               chikungunya
scrub_typhus            leptospirosis          cholera
acute_gastroenteritis   typhoid               hepatitis_A_E
influenza_ili           covid19               pneumonia_bacterial
asthma_copd_exacerb     common_cold            allergic_rhinitis
conjunctivitis          dermatophyte_tinea     hfmd
tb_suspected
```

Unknown names use the default detection threshold but have no disease-specific resource action. See `DISEASE_THRESHOLDS` and `DISEASE_ACTIONS_MULT` in the script for the exact per-disease thresholds and multipliers.

### `baselines` fields

Supply baselines when the frontend needs concrete staffing, stock, and capacity estimates. Keys must match triggered department, bundle, diagnostic, infrastructure, and SKU names exactly.

| Field | Format | How it is used |
| --- | --- | --- |
| `departments` | `{ "ER": { "staff": { "doctors": 10, "nurses": 20, "support": 5 } } }` | Final staff count for each triggered department is `ceil(baseline × multiplier)`. A missing department or staff baseline produces no roster for it. |
| `bundles_per_day` | `{ "DENGUE_CARE": { "Platelet_units": 2, "IV_set": 5 } }` | Bundle ID maps to SKU quantities per day. Demand uses the highest multiplier for that bundle across windows. |
| `diagnostics_per_day` | `{ "Dengue_NS1": 4, "CBC": 10 }` | Daily test/SKU quantities, added to bundle demand when the same SKU appears in both. |
| `inventory_state` | `{ "Platelet_units": { "on_hand": 5, "on_order": 0, "lead_time_days": 8 } }` | Stock and lead time by SKU. Missing values default to `0`, `0`, and `7`; this also applies to entirely missing SKUs. |
| `infra` | `{ "O2_Hours": 100, "Isolation_Beds": 20 }` | Baseline capacity; triggered items with nonzero baselines produce `ceil(base × highest multiplier)` peak values. |

Built-in department names are `ER`, `Medicine`, `ENT`, `Dermatology`, `Pediatrics`, and `Ophthalmology`. Bundle names include `DENGUE_CARE`, `RESPIRATORY_CARE`, `WATERBORNE_CARE`, and the other keys in `BUNDLE_CATALOG`. The catalog is a reference list of bundle contents; **the translator does not automatically expand it**. Supply the actual SKU rates in `bundles_per_day`. Diagnostic examples include `Dengue_NS1`, `CBC`, and `Stool_RDT`; infrastructure examples include `O2_Hours` and `Isolation_Beds`.

### Route settings

The current `POST /plan` handler fixes these settings in `api/plan.py`: multiplier cap `2.0`, AI event parsing enabled, minimum AI event confidence `0.6`, AI event model `gpt-4o-mini`, and briefing model `gpt-4.1-mini`. `apply_multipliers` uses its default seven-day stock coverage. The route ignores any client-supplied `options` field. `free_text_events` can be omitted or set to `[]` when no AI event parsing is needed.

## Complete example request

The following body can be sent directly to `POST /plan`. It triggers all three environmental rules and a dengue signal.

```json
{
  "meta": { "today": "2026-11-11" },
    "weather_past_7d": [
      { "rain_mm": 10, "humidity_pct": 82, "pm25": 100, "temp_c": 29 },
      { "rain_mm": 10, "humidity_pct": 82, "pm25": 110, "temp_c": 29 },
      { "rain_mm": 10, "humidity_pct": 82, "pm25": 120, "temp_c": 29 },
      { "rain_mm": 10, "humidity_pct": 82, "pm25": 130, "temp_c": 29 },
      { "rain_mm": 10, "humidity_pct": 82, "pm25": 210, "temp_c": 29 },
      { "rain_mm": 10, "humidity_pct": 82, "pm25": 220, "temp_c": 29 },
      { "rain_mm": 10, "humidity_pct": 82, "pm25": 230, "temp_c": 30 }
    ],
    "visits_daily": [
      { "date": "2026-11-05", "total_opd": 100 },
      { "date": "2026-11-06", "total_opd": 100 },
      { "date": "2026-11-07", "total_opd": 100 },
      { "date": "2026-11-08", "total_opd": 100 },
      { "date": "2026-11-09", "total_opd": 100 },
      { "date": "2026-11-10", "total_opd": 100 },
      { "date": "2026-11-11", "total_opd": 100 }
    ],
    "epi_signals": {
      "by_disease_daily": [
        { "date": "2026-11-05", "disease": "dengue", "cases": 3, "tests": 10, "positives": 2 },
        { "date": "2026-11-06", "disease": "dengue", "cases": 3, "tests": 10, "positives": 2 },
        { "date": "2026-11-07", "disease": "dengue", "cases": 3, "tests": 10, "positives": 2 },
        { "date": "2026-11-08", "disease": "dengue", "cases": 3, "tests": 10, "positives": 2 },
        { "date": "2026-11-09", "disease": "dengue", "cases": 3, "tests": 10, "positives": 2 },
        { "date": "2026-11-10", "disease": "dengue", "cases": 3, "tests": 10, "positives": 2 },
        { "date": "2026-11-11", "disease": "dengue", "cases": 3, "tests": 10, "positives": 2 }
      ]
    },
  "events": [],
  "free_text_events": [],
  "baselines": {
    "departments": {
      "ER": { "staff": { "doctors": 10, "nurses": 20, "support": 5 } },
      "Medicine": { "staff": { "doctors": 8, "nurses": 12, "support": 4 } },
      "Dermatology": { "staff": { "doctors": 2, "nurses": 2, "support": 1 } }
    },
    "bundles_per_day": {
      "DENGUE_CARE": { "Platelet_units": 2, "IV_set": 5 },
      "RESPIRATORY_CARE": { "Masks": 10 },
      "WATERBORNE_CARE": { "ORS": 8 },
      "DERMATOPHYTE_CARE": { "Topical_Azole": 3 }
    },
    "diagnostics_per_day": { "Dengue_NS1": 4, "CBC": 10, "Stool_RDT": 3 },
    "inventory_state": {
      "Platelet_units": { "on_hand": 5, "on_order": 0, "lead_time_days": 8 },
      "Dengue_NS1": { "on_hand": 4, "on_order": 0, "lead_time_days": 2 }
    },
    "infra": { "O2_Hours": 100 }
  }
}
```

## Response shape

`POST /plan` returns **only** `translated` and `briefing`. The values below are selected excerpts from the example request; actual arrays and text contain more entries.

```json
{
  "translated": {
    "rosters": {
      "D0–D5": {
        "Dermatology": { "doctors": 3, "nurses": 3, "support": 2 }
      }
    },
    "purchase_orders": [
      { "sku": "Platelet_units", "qty": 23, "urgency": "critical" }
    ],
    "infra_peak": { "O2_Hours": 120 }
  },
  "briefing": "Surge Plan for 2026-11-11/2026-11-25\n..."
}
```

| Output | Meaning |
| --- | --- |
| `translated.rosters` | Final staff counts, rounded up, for triggered departments with baselines. |
| `translated.purchase_orders` | Positive SKU shortages after subtracting `on_hand` and `on_order`. |
| `translated.infra_peak` | Peak capacity for triggered infrastructure with a nonzero baseline. |
| `briefing` | AI-generated text when OpenRouter succeeds; otherwise the deterministic summary. It can mention weather, disease signals, horizon multipliers, advisories, and translated actions. |

The planner calculates disease sentinel data, weather features, horizon multipliers, and advisories internally, but this route does **not** return those as separate JSON fields. To expose structured plan details to a frontend, `api/plan.py` would need a code change.

Purchase demand is `ceil(units_per_day × highest_multiplier × days_cover)` per triggered bundle SKU or diagnostic; demands for the same SKU are added. The order quantity is the positive remainder after subtracting stock and items on order. Urgency is `critical` when `lead_time_days > days_cover`, `expedite` when it is at least `days_cover - 1`, and `routine` otherwise.

## Calling the API

Start the service locally from the repository root:

```bash
python -m pip install -r requirements.txt
uvicorn api.plan:app --host 0.0.0.0 --port 8000
```

Then send JSON to `http://localhost:8000/plan`, or use the Railway URL configured for your deployment. For example:

```bash
curl -X POST http://localhost:8000/plan \
  -H "Content-Type: application/json" \
  -d '{"meta":{"today":"2026-11-11"},"weather_past_7d":[],"visits_daily":[],"epi_signals":{"by_disease_daily":[]},"baselines":{}}'
```

From a frontend, send the same flat object as JSON:

```javascript
const response = await fetch(`${API_BASE_URL}/plan`, {
  method: "POST",
  headers: { "Content-Type": "application/json" },
  body: JSON.stringify(payload), // payload has meta, weather_past_7d, visits_daily, epi_signals, baselines
});

if (!response.ok) throw new Error(`Plan request failed: ${response.status}`);
const { translated, briefing } = await response.json();
```

The current FastAPI configuration permits cross-origin requests from any origin. Set `API_BASE_URL` to your local or deployed service address; do not put `OPENROUTER_API_KEY` in frontend code.

The FastAPI handler accepts a generic `dict` and performs no request schema validation beyond JSON object parsing. Validate dates, numeric ranges, and required operational fields on the client or add a server-side Pydantic model. An invalid `meta.today` can raise an error during plan construction.

## Setup and optional AI

The Dockerfile installs `fastapi`, `uvicorn`, and `requests` from `requirements.txt`, then starts Uvicorn on port `8000`. On Railway, deploy the repository as a Docker service and configure the public domain for that service.

For optional OpenRouter features, set the environment variable on the **server**:

```bash
OPENROUTER_API_KEY=your-key
```

The route always enables AI event parsing and calls the AI briefing function. A missing key or failed OpenRouter call skips free-text event interpretation and makes the briefing fall back to the deterministic summary. Never include the key in frontend JSON. The model IDs in `api/plan.py` are passed to OpenRouter as written; confirm they are supported in your account if AI calls fail.

The AI event parser accepts department names `ER`, `Medicine`, `ENT`, `Dermatology`, `Pediatrics`, and `Ophthalmology`, clamps multiplier values to `[1.0, 2.0]`, and rejects results below confidence `0.6` for this route. Review accepted free-text interpretations before using them operationally.

## References

- [Example API caller](https://github.com/rajveer029/surge-planner-1/blob/main/sample-use/calling_api.py)
- [Sample input file](https://github.com/rajveer029/surge-planner-1/blob/main/sample-use/sample_input_format.py)
- [Colab demo](https://colab.research.google.com/drive/1dqAp3KSHuHRE2znd_TeNn0i0q3-UHk66?usp=sharing)

## Author

[@rajveer029](https://github.com/rajveer029)
