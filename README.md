# Vodafone Data‑Virtualisation API (RAW)

> **This README covers the REST/SQL endpoint layer that sits *after* the ETL stage.** If you are looking for the ETL mechanics (Excel → SQLite pipeline) head over to **[README-ETL.md](README-ETL.md)**.

---

## 1 · What problem does this solve?
Vodafone UK kept seven operational spreadsheets in SharePoint—sites, power capacity, space/sections, Opex and lease information. Querying them meant VLOOKUP chains, copy‑paste errors and siloed access rights. The *RAW* Data‑Virtualisation layer turns those **static files into live, filterable REST endpoints** so that:

* ⭐ **Data analysts** can answer *cross‑domain* questions with a single HTTPS call and zero Excel joins.
* ⭐ **Squirro Chat + GPT agents** retrieve the **exact** result set; WHERE‑clauses live in SQL, not in opaque prompt prose.
* ⭐ **Security & Governance** stay in control—workbooks never leave the VPC and row/column masking happens in‑band.

During User‑Acceptance‑Testing the approach achieved **92.9 % weighted accuracy** with **≈ 4 s median latency**, exceeding every KPI in the statement of work.

---

## 2 · High‑level architecture
```
Excel / CSV  ─┐
              │  python  (excel_to_sqlite.py)   SQL cleansing (vf_*.sql)       RAW Gateway (OpenAPI 3.1)
SharePoint  ──┼──▶  SQLite  ───────────────────────▶  views (vf_sites, vf_capacity …)  ──▶  HTTPS consumers
Manual drop ─┘
```

**Pipeline explained**
1. **Extract & Load** – `excel_to_sqlite.py` unmerges cells, normalises headers and writes one *raw* table per sheet.
2. **Transform** – a suite of `vf_*.sql` scripts cleanses the data: splits composite site‑codes, coerces numeric types and computes derived KPIs.
3. **Virtualiσe** – RAW reads the *views* directly and compiles them into an OpenAPI 3.1 contract during the container build. The contract is served by the Gateway pod and version‑pinned via semantic tags.
4. **Consume** – Squirro Chat, BI dashboards or curl scripts call the endpoints; responses stream as JSON (or CSV if `Accept: text/csv`).

The combined solution runs on a single‑node **k3s** cluster inside Vodafone’s private GCP VPC; images are pulled from a signed ECR registry and refreshed every five hours.

---

## 3 · Endpoint catalogue
| Path | Purpose | Core filters | Persona scope |
|------|---------|-------------|---------------|
| **`GET /vf/sites`** | Master site dimension; geo/type lookup | `site_types`, `site_regions`, `status`, … | `vodafone:admin` |
| **`GET /vf/sites/space/trends`** | 2024 monthly free‑section arrays | `site_codes`, `page`, `page_size` | infrastructure |
| **`GET /vf/sites/opex/monthly`** | Zero‑imputed 2024 cost per site | `year`, `min_month`, `max_month` | finance |
| **`GET /vf/sites/opex/{positive,negative}_trends`** | Strictly ↑ / ↓ Opex patterns | same as above | finance |
| **`GET /vf/elements/capacity`** | Element‑level power KPIs (MTX + fixed) | ~20 range filters (`remaining_power_capacity_in_kw_minimum` …) | infrastructure |
| **`GET /vf/outliers/{space,capacity,opex}`** | Mismatch diagnostics | `site_codes` | super‑user |
| **`GET /vf/sites/combined`** | **One‑shot join of sites + capacity + space + opex**; ~60 optional filters incl. `topN` | all roles (columns masked per scope) |

> **💡 Tip** For natural‑language prompts we default to `/vf/sites/combined`; it eliminates multi‑step call‑chaining which caused **≈ 70 % of early failures**.

---

## 4 · Parameter grammar
* **Range pairs** – Every numeric KPI offers a `_minimum` and `_maximum`. Omit either side for −∞ / +∞.
* **Comma‑lists** – `site_codes`, `site_types`, `site_regions` are parsed via `string_to_array()`; order is irrelevant.
* **Substring search** – Free‑text fields (`site_name`, `site_address`, `comments`) are wrapped in `ILIKE '%value%'`.
* **Pagination** – Uniform `page` (1‑based, default 1) and `page_size` (default 500). SQL applies `LIMIT/OFFSET` once per template for plan stability.
* **Top‑N** – On `/vf/sites/combined` a final `ORDER BY … DESC LIMIT :topN` selects the p‑most expensive or highest‑capacity rows after all joins.

A canonical example looks like:
```bash
curl -H "Authorization: Bearer $TOKEN" \
  "https://api.vf.example/vf/sites/combined?site_regions=London,South&remaining_power_capacity_in_kw_minimum=30&free_sections_minimum=2&topN=25&page=1&page_size=100"
```

---

## 5 · Authentication & row‑level security
1. **Gateway** injects `userid=<workspace_uid>` into every call.
2. Each SQL template starts with one or more **blacklist CTEs** which pull masked IDs from the Credentials store and bind against the supplied user ID:
   ```sql
   WITH user_blacklist_opex AS (
     SELECT id
     FROM environment.secrets,
          unnest(string_to_array(secret, ',')) id
     WHERE name = 'user-blacklist-opex' AND id = :userid
   )
   ```
3. A final clause – `WHERE NOT EXISTS (SELECT 1 FROM user_blacklist_opex)` – redacts sensitive rows on the fly.
