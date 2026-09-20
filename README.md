# Dirty Data Sandbox

A browser-only sandbox for practising SQL and pandas against a realistic,
deliberately messy synthetic dataset. No account, no backend, nothing saved.
Open it, practise, close the tab.

Live at **https://deo-reji.github.io/Sandbox_sql_pandas/** (GitHub Pages,
served straight from this repo's `main` branch).

Everything lives in one self-contained file: [index.html](index.html).

## Running it

It must be served over HTTP. DuckDB-WASM loads its engine from a cross-origin
CDN and spawns a Web Worker from a `blob:` URL — both are blocked under
`file://`, and the app will tell you so rather than showing a blank page.

```
python -m http.server 8000
# then open http://localhost:8000/index.html
```

First load pulls ~7 MB of compressed WASM from jsDelivr; after that it is cached.
Boot (generate 2.9 MB of CSV, start DuckDB, create 7 tables, start the editor)
takes roughly 2–3 seconds on a warm cache.

The pandas tab is lazy: nothing Python-related loads until you first click it.
That first click pulls Pyodide plus pandas/numpy (~10 MB, also cached after)
and takes 10–15 seconds; the SQL tab is unaffected and stays fully usable
while that happens in its own Web Worker.

## The dataset

Seven tables — `suppliers`, `products`, `customers`, `orders`, `order_items`,
`returns`, `web_events` — about 62,700 rows total. Generated in the browser from
a fixed-seed mulberry32 PRNG, so **every visitor sees byte-identical data**.

The data is broken on purpose. Twelve classes of defect are injected at
3–12% of rows each: mixed date formats in one column, money stored as
`"$1,240.50"` and `"(85.00)"`, nulls that mean three different things,
orphaned foreign keys, mojibake, a UTF-8 BOM, whitespace and casing noise in
join keys, mixed-type columns, impossible values, duplicate rows, and — the
headline trap — duplicated `products.sku` values that silently inflate a naive
revenue join by about 4.5%.

The **Data dictionary** button describes the schema the data was *supposed* to
conform to. The gap between that and reality is the exercise.

## Tuning the mess

Defect rates live in one object, `DEFECTS`, near the top of the script. Every
entry is a fraction of rows affected (or an absolute count where noted).
Changing a rate does not change row counts or primary keys, so nothing
downstream breaks.

Two things to know before editing the generator:

- All randomness comes from a single seeded stream. The *order* in which
  generators consume it is part of the contract — inserting a new draw in the
  middle of a generator shifts every value after it. Add draws at the end.
- Corruption is applied at serialisation time (`SECTION 6`), never during entity
  generation. Both engines are handed the same CSV strings, so SQL and pandas
  can never disagree about what the data is.

## The pandas tab

Pyodide 0.27.8, loaded from jsDelivr's `/pyodide/v.../full/` path — **not**
its `/npm/pyodide@.../` mirror, which only ships the core runtime and 404s on
`loadPackage(['pandas'])`. This contradicts some published guidance about
using the npm path; it was verified against the live CDN before being wired
up, not assumed. Runs in its own Web Worker for the same reason DuckDB does:
a bad script (infinite loop, huge computation) can't freeze the tab.

Every table is pre-loaded as a DataFrame named after the table, from the same
CSV strings DuckDB reads — `pd`, `np`, and the seven tables are listed in the
pre-loaded bar above the editor. The value of the last expression renders as
a grid (DataFrame/Series) or as text (anything else); `print()` output goes
to the console pane below the results, not into the grid.

There's no true query cancellation — that needs `SharedArrayBuffer` plus
cross-origin-isolation response headers, which a static GitHub Pages deploy
can't set. **Stop** is honest about this: it hard-terminates the Python
worker and boots a fresh one, which loses any variables you'd defined.
Pre-loaded tables come back clean either way.

## Status

- **Stage 1 — done.** Data generation, DuckDB-WASM, SQL tab, virtualised results grid.
- **Stage 2 — next.** Task list and full schema browser.
- **Stage 3 — done.** Pyodide and the pandas tab, fed from the same CSV strings.
