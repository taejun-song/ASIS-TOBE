# ASEKR Standard Map Output Format

Derived from analysis of 6 sample output files in `output/`.

## Structure

```
Line  1: ASEKR standard map version : 001
Line  2: Customer : {customer_name}
Line  3: Supplier : {supplier_name | "N/A"}
Line  4: Wafer ID : {wafer_id}
Line  5: Wafer lot : {lot_id}
Line  6: Wafer No : {wafer_no}
Line  7: Row(Y) : {row_count}
Line  8: Column(X) : {column_count}
Line  9: Notch : {notch_direction | ""}
Line 10: Good bin code : {good_bin_code}
Line 11: Good bin count : {good_bin_count}
Line 12: Total good qty : {total_good_qty}
Lines 13-50: (blank lines — 38 lines of padding)
Lines 51+: Wafer map grid (one line per row)
```

## Field Rules

| Field | Required | Format | Example |
|-------|----------|--------|---------|
| version | yes | Always "001" | `001` |
| Customer | yes | Free text, placeholder if unknown | `SILICONMITUS` |
| Supplier | no | Free text or "N/A" | `N/A` |
| Wafer ID | yes | Alphanumeric with hyphens | `1ACB86_W25` |
| Wafer lot | yes | Alphanumeric | `1ACB86` |
| Wafer No | yes | Numeric string or alphanumeric | `25` |
| Row(Y) | yes | Integer | `36` |
| Column(X) | yes | Integer | `90` |
| Notch | no | Direction or degree or blank | `BOTTOM`, `315`, `0` |
| Good bin code | yes | Single code or comma-separated | `1` |
| Good bin count | yes | Integer | `2563` |
| Total good qty | yes | Integer | `2563` |

## Wafer Map Grid

- Each row is a string of single characters
- `.` = no die at this position (empty/outside wafer)
- `1` = passing die (good bin)
- Other characters = failing/edge/reference bins (vendor-specific)
- Row count must match `Row(Y)` value
- Max line length must match `Column(X)` value

## Variations Observed in Samples

| Sample | Notch style | Bin chars used | Good bin trailing comma |
|--------|-------------|----------------|----------------------|
| 1ACB86_W25 | (blank) | 1, X | no |
| 60XHHA-22 | 0 | 1, X, C, J | yes (inconsistent) |
| 68ZBC3P-02 | BOTTOM | 1, X | no |
| B07092.05-12 | Bottom | 0, 1 | no |
| BN1737-23 | 0 | 1, Z | no |
| GP300P043-05 | 315 | 1, 9, X, Y | no |
