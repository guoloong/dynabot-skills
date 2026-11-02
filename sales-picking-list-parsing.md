# Skill: Sales Picking List parsing

## When to use

A user supplies one or more ERP exports named like:

- `Sales Picking List - <year>.csv` (2023, 2024, 2025 → **full year**: 1/1 – 31/12)
- `Sales Picking List - Level 1.csv` (2026 → **year to date**: 1/1 – 30/9)

…from **Dynamic Nutrition Sdn Bhd** and asks to "make sense of it", parse it,
summarise sales, or compare years.

These files are **printed reports dumped to CSV**, not data tables. They cannot be
read with a naive `csv.DictReader` — see the trap table below. The goal of this skill
is a tidy line-level dataset that **reconciles exactly** with the report's own printed
totals, then analysis on top of it.

## What the report contains

```
page banner   "Sales Picking ListAs At <date>" + company + "Page n of N"   (every page)
column header "Doc No ... Doc Date Code Description ... Qty"               (every page)
└── StockGroup "DEFAULT"                       (the only group ever printed)
    └── Agent  "3000/O75  Ooi Johnny"          ← customer account (code + name)
        └── Doc "90007813  2/1/2026"           ← the FIRST item is on this same line
            ├── NAC002  Super BIO-Organic 600gm  Bottle  3.00
            └── Shipping  Shipping Charges  Parcel       1.00
footer  "Total Record(s): N" → "Picking List Summary" (per-item totals) → "Total Item(s): N"
```

* Quantities only — **no prices, amounts, costs or returns anywhere**. Never quote
  revenue from these files; say plainly that it cannot be derived.
* Two document-number series exist in every year, printed as two consecutive blocks:
  `9000xxxxx` (small direct/agent orders) and `A00xxxxx` (large wholesale/pharmacy
  orders). The report's sort key list starts with `CompanyCode`
  (2023–25) or `StockGroup` (2026), which is what separates the blocks.
* `Sort By` header line is worth quoting back to the user — it documents the ERP's
  grouping keys: `CompanyCode;StockGroup;DocDate;Agent;Area;DocProject;Project;Location;Batch;ItemCode;CompanyCategory;Shipper;DocNo`.

## Traps (each one silently corrupts the data)

| # | Trap | Symptom if unhandled |
|---|---|---|
| 1 | A document's **first item is printed on the Doc No line** | lose ~48% of all lines (e.g. 1,739 of 3,609) |
| 2 | **Column offsets drift** between pages (shift of 2, and the gap between Description and UOM varies) | code/qty read from wrong column |
| 3 | `DocDate` is only printed **when it changes** | only ~411 of 7,592 physical lines carry a date |
| 4 | Descriptions may contain a **quoted comma** (`"...Warehouse, Klang."`) | `line.split(',')` drops that line |
| 5 | A doc straddling a **page break reprints** the agent + Doc No line, then continues with *new* items | double-count if treated as a new doc's first item |
| 6 | Encodings: 2026 is **Latin-1 + non-breaking spaces**; 2025 has mixed CRLF/CR; some names are already mangled to `??` in the source | UTF-8 readers raise; names corrupted |
| 7 | Agent codes are usually `3000/A123` but 2023 also has **`300-B0001`** | those agents' items get attributed to the previous agent |
| 8 | 2023 repeats some agent header lines back-to-back; the footer summary grid in 2024/25 **continues across a page break with a different column split** | mis-paired (item, qty) in the checksum |
| 9 | 2026 has one odd item `Less Discount / RM 8.00 / UNIT / 1.00` | looks like a product; it is a discount artefact |

## Line classification decision table

Parse each physical line into cells (proper CSV reader, then strip). Let `nz` = non-empty cells.

| Test (in order) | Meaning | Action |
|---|---|---|
| `nz[0]` starts `DYNAMIC NUTRITION` / `Sales Picking List` | page furniture | skip; capture `Page n of N` |
| `nz[0] == 'Doc No'` | column header | skip (count it: 1 per page) |
| `nz[0] == 'Sort By'` / `'Date'` | header block | capture `From x to y` date range, sort keys |
| `nz[0] == 'DEFAULT'` | stock group | skip (only group present) |
| `re.match(r'^\d{3,4}[/-][A-Za-z0-9]+$', nz[0])` | **agent header** | set `agent_code`; `agent_name` = first non-date cell after it |
| `cells[0]=='' and len(nz)==1 and not doc_re.match(nz[0])` | agent name whose code cell was blanked by a page break (1 case in 4 years) | set `agent_name` only |
| `cells[0]=='' and doc_re.match(cells[1])` | **doc header** (`doc_re = ^(\d{8}\|[A-Z]\d{7})$`) | set `doc_no`; if a date cell is present update `last_date`; `doc_date = last_date`; **the remaining 4 cells are the doc's first item** |
| `len(nz)==4 and qty_re.match(nz[3])` | **continuation item** | `[item_code, description, uom, qty]` |
| anything else | anomaly | log it loudly — the gate is 0 unparsed lines |

`qty_re = ^-?[\d,]*\d(?:\.\d+)?$` · `date_re = ^\d{1,2}/\d{1,2}/\d{4}$`

**Never split on commas by hand** — one description in the 2026 file contains
`"Transport charges for courier service to Matahari Warehouse, Klang."` and a naive
split both loses that line and breaks the reconciliation by exactly 1 unit.

## Reference implementation

```python
import csv, re, collections

agent_re = re.compile(r'^\d{3,4}[/-][A-Za-z0-9]+$')      # 3000/A123 and 300-B0001
doc_re   = re.compile(r'^(?:\d{8}|[A-Z]\d{7})$')          # 90007813 and A0012222
date_re  = re.compile(r'^\d{1,2}/\d{1,2}/\d{4}$')
qty_re   = re.compile(r'^-?[\d,]*\d(?:\.\d+)?$')

def parse(path, year):
    rows, unparsed = [], []
    page = 0
    agent_code = agent_name = doc_no = doc_date = last_date = None
    in_summary = False
    counts = collections.Counter()
    text = open(path, 'rb').read().decode('cp1252')       # latin-1 family; ASCII years are unaffected
    for ln, raw in enumerate(text.split('\n'), 1):
        raw = raw.rstrip('\r')                            # 2025 mixes CRLF and CR
        cells = [c.strip() for c in next(csv.reader([raw]))] if raw.strip() else []
        nz = [v for v in cells if v]
        if not nz:
            continue
        m = re.search(r'Page (\d+) of (\d+)', raw)
        if m:
            page = int(m.group(1)); counts['pages'] = max(counts['pages'], int(m.group(2)))
        if nz[0].startswith('Picking List Summary'):
            in_summary = True
        if in_summary:
            continue
        if nz[0].startswith('Total Record(s)'):
            counts['reported_records'] = int(re.search(r'(\d+)', nz[0]).group(1)); continue
        if nz[0].startswith(('DYNAMIC NUTRITION', 'Sales Picking List')):
            continue
        if nz[0] == 'Doc No':
            counts['page_headers'] += 1; continue
        if nz[0] in ('Sort By', 'Date'):
            continue
        if nz[0] == 'DEFAULT':
            continue

        def add(vals):                                    # [item_code, description, uom, qty]
            rows.append(dict(year=year, page=page, line=ln, agent_code=agent_code,
                             agent_name=agent_name, doc_no=doc_no, doc_date=doc_date,
                             item_code=vals[0], description=vals[1], uom=vals[2],
                             qty=float(vals[3].replace(',', ''))))

        if agent_re.match(nz[0]):                         # agent header
            agent_code = nz[0]
            agent_name = next((v for v in nz[1:] if not date_re.match(v)), '')
            continue
        if cells[0] == '' and len(nz) == 1 and not doc_re.match(nz[0]):
            agent_name = nz[0]; continue                  # agent code blanked by page break
        if cells[0] == '' and len(cells) > 1 and doc_re.match(cells[1]):
            doc_no = cells[1]                             # doc header (+ its first item)
            rest = nz[1:]
            d = next((v for v in rest if date_re.match(v)), None)
            if d:
                last_date = d; rest = [v for v in rest if v != d]
            doc_date = last_date                         # carry-forward
            if len(rest) == 4 and qty_re.match(rest[3]):
                add(rest)
            continue
        if len(nz) == 4 and qty_re.match(nz[3]):          # continuation item
            add(nz); continue
        unparsed.append((ln, cells))
    return rows, counts, unparsed

def footer_totals(path):
    """The printed 'Picking List Summary': 3 (code, qty) pairs per row, grid may
    continue across a page break with a different column split -> pair the
    non-empty tokens instead of using fixed columns."""
    totals, started, reported_items = {}, False, None
    text = open(path, 'rb').read().decode('cp1252')
    for raw in text.split('\n'):
        raw = raw.rstrip('\r')
        cells = [c.strip() for c in next(csv.reader([raw]))] if raw.strip() else []
        nz = [v for v in cells if v]
        if nz and nz[0].startswith('Picking List Summary'):
            started = True
        if not started or not nz:
            continue
        m = re.search(r'Total Item\(s\):\s*(\d+)', nz[0])
        if m:
            reported_items = int(m.group(1)); continue
        if nz[0] == 'Item Code':
            continue
        for a, b in zip(nz[0::2], nz[1::2]):
            if qty_re.match(b) and not qty_re.match(a):
                totals[a] = float(b.replace(',', ''))
    return totals, reported_items
```

**Tidy schema** (one row per picked line):

`year, page, line, agent_code, agent_name, doc_no, doc_date, item_code, description, uom, qty`

## Validation gates (all must pass before any analysis)

1. `len(rows) == counts['reported_records']` (the printed `Total Record(s): N`)
2. `len(distinct item_code) == reported_items` (the printed `Total Item(s): N`)
3. **per-item quantities == the printed `Picking List Summary`, zero mismatches**, and the
   grand totals equal (this is the real proof — it validates every line's item *and* qty)
4. `unparsed == []` — every physical line was classified
5. dates: after carry-forward no row has an empty `doc_date`; within each doc-number series
   there are only a handful of out-of-order docs (the source prints a few, e.g. 21 across 2023–26)

Known-good regression fixtures — if a parse of these files reports anything else, the parser is wrong:

| File | Pages | Documents | Lines (= `Total Record(s)`) | SKUs (`Total Item(s)`) | Total qty |
|---|---|---|---|---|---|
| 2023 (full year) | 144 | 1,642 | 4,715 | 84 | 22,299 |
| 2024 (full year) | 95 | 1,770 | 4,151 | 79 | 19,946 |
| 2025 (full year) | 99 | 2,149 | 4,137 | 71 | 20,282 |
| 2026 (Jan–Sep) | 84 | 1,706 | 3,609 | 69 | 15,586 |

Combined 2023–2026: **16,612 lines**, 2,176 accounts, 115 distinct SKUs. Footer-pair
extraction must also handle `Less Discount` (a non-SKU row) and `Shipping` / `Transport`.

## Analysis recipes

* **Doc-level roll-up**: group by `doc_no` → lines, units, product/FOC/freight split,
  first doc date. `documents.csv` in the workspace is the template.
* **Item × month / item × year matrices**: the workhorse for "what are the sales of
  <product> every month" questions. Emit units *and* document counts — order count is
  what separates real demand from a few big refills.
* **Account roll-up by `agent_code`, never by name**, then a per-year matrix for retention.
* **Channel split** by doc-number series (`9000xxxxx` = direct/agent, `A00xxxxx` =
  wholesale/pharmacy) — the two behave very differently.
* **Like-for-like year comparison**: 2026 is Jan–Sep only, so compare **Jan–Sep** for every
  year and never compare a full year to a partial one. Category split matters: excluding
  FOC-sample and freight units changes the 2026 story materially (2026 relaunched sampling:
  `REB001-Samples` was 133 units in 2023, **0 in 2024 and 2025**, 1,554 in Jan–Sep 2026).

## Semantics to state in every answer

1. A "unit" is a **package** — Box, Bottle, Sachet, Set, Parcel, Carton are all counted as 1.
   Totals are a picking-workload index, **not** comparable product volume; never sum across SKUs
   as if they were equivalent.
2. Quantity picked ≠ invoiced sales: no prices, no returns/credit notes, no cancellations.
3. Twin-pack (`-1`, `-2`, `-3`) and FOC (`*-Samples`, `*-Loose`) SKUs are separate codes —
   merge or exclude them deliberately, and say which you did.
4. Account identity is messy in both directions: 54 codes carry 2–4 name spellings and one code
   (`3000/S484`) maps to two different people, while one customer can hold two codes
   (Caring Pharmacy = `3000/C247` then `3000/C426`). Say that retention/churn figures are
   approximate because of this.
5. UOM case drifts (`Set`/`set`, `Box`/`BOX`).

## Workflow

1. Copy each attachment to a writable path; keep the originals read-only.
2. Decode `cp1252`, normalise `\xa0` → space, write `raw_utf8*.csv` (2026 needs this; other years are ASCII).
3. Parse every year with the reference implementation; write `parsed_items_<year>.csv` and a combined `parsed_items_all.csv`.
4. Run all five validation gates; print them. **Do not proceed to analysis if any gate fails** —
   fix the classifier instead.
5. Build summaries: `yearly_summary.csv`, `monthly_by_year.csv`, `item_year_matrix.csv`,
   `agent_year_matrix.csv`, plus doc/item/agent roll-ups.
6. Write the narrative: coverage + verification table first, then trends, then a product
   deep-dive, then the semantics caveats. Lead with the reconciliation proof so the numbers
   are trustworthy.
7. Present the markdown report and the tidy CSVs; state the reproduction command
   (`python3 parse_all_years.py && python3 make_cross_year.py`).

## Workspace reference (this machine)

| Path | Contents |
|---|---|
| `sales_report/parse_all_years.py` | canonical parser for all four files + gates |
| `sales_report/make_cross_year.py` | cross-year tables and `cross_year_units.svg` |
| `sales_report/parse_sales.py`, `make_summaries.py` | 2026-only first-pass versions |
| `sales_report/ANALYSIS.md` | 2026 deep dive |
| `sales_report/ANALYSIS_2023-2026.md` | four-year analysis |
| `sales_report/parsed_items_all.csv` | 16,612 tidy rows, 2023–2026 |
| `sales_report/hp-floragut-2026-buyers.md` | HP-FloraGut 2026 vs 2025 + the 38 accounts buying it in 2026 |
| `sales_report/floragut_2026_buyers.py` | rebuilds that report: totals, buyer list, churn (`floragut_2026_buyers.csv`) |
| `sales_report/floragut_shortfall.py` | company-level shortfall decomposition, 2025 → 2026 |
| `sales_report/analyse_floragut.py` | monthly FloraGut series + SVG chart |

> **Do not commit the sales data or analysis outputs into this repo** — they are
> confidential business data (`sales_report/` is untracked on purpose). Only this skill
> document belongs here.
