# Metrics

Metrics are aggregate measures defined in the `metrics` array of a table element. They standardize common calculations so workbook authors don't need to rewrite them.

## Metric

```json
"metrics": [
  {
    "id": "metric-revenue",
    "formula": "Sum([Price])",
    "name": "Total Revenue"
  },
  {
    "id": "metric-unique-skus",
    "formula": "CountDistinct([Sku Number])",
    "name": "Unique Products"
  }
]
```

## Metric schema

| Field | Type | Required | Notes |
|---|---|---|---|
| `id` | string | yes | Short alphanumeric ID |
| `formula` | string | yes | Sigma aggregate formula |
| `name` | string | no | Display name |
| `description` | string | no | |
| `format` | Format object | no | See [formatting.md](formatting.md) |
| `timeline` | Timeline object | no | See below |

## Metric timeline

> **`dateColumnId`:** Use the column's ID as defined in the same spec. The API resolves this cross-reference at submission time — if you submit with an inode column ID (e.g., `"inode-5FCsrDpnzcdw5YYJRBbY6l/DATE"`), the returned spec will show the server-assigned column ID. The timeline will be present and correct in the returned spec.

Add a `timeline` object to a metric to enable time-series trend tracking.

```json
{
  "id": "metric-revenue",
  "formula": "Sum([Price])",
  "name": "Revenue",
  "timeline": {
    "dateColumnId": "<column-id-of-date-column>",
    "truncation": "month",
    "comparison": {
      "comparisonPeriod": "year",
      "direction": "higher-is-better"
    }
  }
}
```

**`truncation` values:** `"year"`, `"quarter"`, `"month"`, `"week-starting-sunday"`, `"week-starting-monday"`, `"day"`, `"hour"`, `"minute"`

**`comparisonPeriod` values:** `"year"`, `"quarter"`, `"month"`, `"week"`, `"day"`

**`direction` values:** `"higher-is-better"`, `"lower-is-better"`

## Grain limits — a metric can't express two-stage aggregation

A metric is scoped to the single table element it's defined on. That means it can express **one level** of aggregation over that element's rows — it cannot, within one metric, "roll up to some intermediate grain, then aggregate again across that rolled-up grain." A common shape that needs exactly this: a portfolio-level rate that should first collapse to one row per account (does this account qualify, yes/no) and only then be averaged across accounts. A metric or KPI built directly against the detail-grain table for a ratio like this can compile and query cleanly while silently returning the **wrong number** — it aggregates at the row grain the table actually has, not the account grain the business question is really asking about. There's no error; the formula is syntactically fine, it's just answering a different, finer-grained question than intended.

The fix is to do the first-stage rollup **before** the metric — typically a SQL or derived source that pre-aggregates to the grain the ratio needs (e.g., one row per account, with that account's own qualifying totals already computed) — so the metric sitting on top of it is a plain single-level ratio over already-correct-grain rows. If a metric's result looks implausible next to the same calculation done "the long way" (grouped by hand, then aggregated in a second pass), suspect a grain mismatch before suspecting the formula syntax — the two most often disagree because the metric is silently aggregating one level too shallow.
