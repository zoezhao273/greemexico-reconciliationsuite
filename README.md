# NCC × CFDI Reconciliation Suite

A single, self-contained HTML tool (no install, no server — open `reconciliation_suite.html` in a browser) that cross-checks NCC accounting documents against the official CFDI invoice file and assembles the DIOT compliance report. The **CFDI file is always the source of truth** for invoice status, amounts, exchange rate, and issuer RFC.

The suite contains two tools:

1. **NCC × CFDI Reconciliation** — line-by-line validation of the reimbursement export against the CFDI file (Checks 1–7).
2. **DIOT Report Builder** — aggregates VAT per invoice across the AP and PY files and produces the four-column DIOT report.

Everything runs locally in the browser; uploaded workbooks never leave the machine.

---

## Inputs

### NCC file (reimbursement export)
A workbook containing a **main table** (header row starting with `PayableBill_$Head` for AP / payable layouts or `PayBill_$Head` for PY / payment layouts) and a **detail table** (header row starting with `BodyS`).

| Table | Columns used |
|-------|--------------|
| Main | `DocNo.`, `DocDate`, `ReimbursementNo.`, `Supplier` (`供应商`), `A/PType`, `PmtType` |
| Detail | `Currency`, `Ex.Rate`, `AmtIncl.VAT(OC)`, `VATCode`, `UUID`, `Material`, `VATAmt(FC)`, `AmtAfterWHT(OC)` |

The first detail column is the line sequence number; each detail line is tied back to its record via that sequence.

### CFDI file (source of truth)
One row per invoice. Columns matched by header name (with sensible fallbacks):

| Field | Header(s) matched | Used by |
|-------|-------------------|---------|
| UUID | `UUID` | All checks (join key) |
| Estatus | `Estatus` / `Status` (fallback: column C) | Check 2, DIOT |
| Emisión | `Emisión` / `Emision` | Check 3 |
| IVA | `IVA` | Check 5 |
| IVA Retenido / ISR Retenido | `IVA Retenido`, `ISR Retenido` | Check 7 |
| Total Original XML | `Total Original XML` / `Total Original` / `Total XML` | Check 4 |
| Tipo Cambio | `Tipo Cambio` / `Tipo de Cambio` | Check 6 |
| Método Pago | `Método Pago` / `Metodo Pago` | Duplicate-vs-amortized signal |
| Emisor RFC | `Emisor RFC` | DIOT report |

---

## UUID handling

Every UUID is normalized before comparison so that cosmetic differences never cause a false mismatch:

- Whitespace removed.
- A stray leading `:` prefix removed.
- All Unicode dash variants (en dash, em dash, minus sign, etc.) unified to a plain `-`.
- Upper-cased.

A value is treated as a **real UUID** only if it matches the canonical format `8-4-4-4-12` hexadecimal. Anything else (`-`, `NA`, `Taxi`, free text) is treated as a *no-invoice placeholder* and is never counted as a duplicate or matched to the CFDI file.

---

## Exemptions (records skipped from all checks)

A record is excluded entirely when any of the following holds:

- **Supplier** contains `GREE AIRCONDITIONING MEXICO` or `CDMX GOV` — salary / tax related.
- **PY** document whose `PmtType` is `Cash Advance Payment` or `Cash Avance Return` — not invoice related.
- **Whitelist match** — a reviewed entry on the approved list. If the entry looks like a UUID it exempts that invoice wherever it appears; otherwise it is treated as a `DocNo.` and exempts the whole record.

---

## Tool 1 — NCC × CFDI Reconciliation

The seven checks below run on the non-exempt records. Money is compared as integer cents to avoid floating-point noise; "exact to the cent" means zero tolerance.

### Check 1 — Duplicate UUIDs *(runs first)*
When a real UUID appears on **more than one distinct record**, the two cases are told apart automatically:

- **Amortized invoice (expected, not an error).** One invoice intentionally split across periods. Recognized when the per-record portions are roughly equal, each portion is clearly below the full invoice (`< 95%` of total), the portions sum to no more than the invoice total (`≤ 100.5%`), **and** either:
  - the ratio test resolves `Total / portion` and `IVA / vatPortion` to the *same* integer k between 2 and 36, **or**
  - the material is `Bank Charges` (recurring charge against a single invoice).
  
  `PPD` payment method is treated as a supporting signal. Amortized records have their **Total / VAT / month checks relaxed**.
- **Possible duplicate reimbursement (flagged).** The same UUID recorded at the full invoice amount on multiple records.

Placeholder UUIDs and repeats within a single record are never counted as duplicates.

### Check 2 — Valid UUID
A UUID is valid **only when both conditions hold**:

1. it exists in the CFDI file, **and**
2. the matched invoice's `Estatus` is exactly `Vigente`.

Outcomes:
- **Not found in CFDI** → flagged as *UUID not found*. Exception: a line whose `Material = Non-Deductible Expense` is exempt from this case only, since those legitimately have no CFDI.
- **Found but not `Vigente`** (e.g. `Cancelado`, or a blank status) → flagged as *exists in CFDI but status is not Vigente*. The report shows expected `Vigente` versus the actual status, and the row is tinted with a status badge.

An invalid UUID stops the downstream Total / VAT / FX / withholding checks for that line — the invoice is already invalid, so there is no point reconciling its amounts.

### Check 3 — Posting month
NCC `DocDate` month must equal CFDI `Emisión` month. Run **only** when the record is a `DocNo.` starting with `AP` *and* `A/PType = CA Reimbursement`, or when the supplier contains `PETTY CASH`.

### Check 4 — Total amount
NCC `AmtIncl.VAT(OC)` must equal CFDI `Total Original XML`, exact to the cent. An invoice split by expense type within one record is summed first. If booked **below** the invoice, the message is *"Please verify the bank payment amount"*; if **above**, *"does not match"*.

### Check 5 — VAT amount
NCC `VATAmt(FC)` (summed per invoice within a record) must equal CFDI `IVA`, exact to the cent. Exceptions:

- **Meals within 50 km.** Booked VAT equal to **8.5%** of the CFDI IVA (±0.01) is accepted, because SAT allows only 8.5% deductible for meals within 50 km.
- **Mislabeled meal.** If the 8.5% pattern matches but `Material ≠ Travel – Meals`, the mismatch is still flagged with the reason `Meal ≤ 50 KM but incorrect material`.
- Skipped for `PY` non-MXN lines (settled at the payment-date FX rate).

### Check 6 — Exchange rate
Only when NCC `Currency` is **not** `MXN`: NCC `Ex.Rate` must equal CFDI `Tipo Cambio` (within a 0.1% relative tolerance). Skipped for `PY` payment documents, whose foreign-currency lines use the payment-date rate rather than the invoice rate.

### Check 7 — Withholding tax
For lines where `VATCode = PresetWHTL0 Donot used`, the absolute value of `AmtIncl.VAT(OC)` must equal CFDI `IVA Retenido + ISR Retenido` (within one cent).

### Reconciliation output
- On-screen summary cards (records, lines, invoices, errors) plus breakdown chips per check.
- A possible-duplicates panel and an amortized-invoices panel.
- A flagged-rows table; cancelled / non-Vigente rows are tinted.
- **Export to Excel** — an Issues sheet with columns `DocNo.`, `Reimbursement No.`, `Material`, `Check`, `What's wrong`, `CFDI (correct)`, `NCC (recorded)`, `UUID`, `Status`, plus a duplicates sheet.

---

## Tool 2 — DIOT Report Builder

1. **Index the CFDI file** by UUID into `{ Emisor RFC, Estatus }` for O(1) lookups.
2. **Extract and aggregate** the NCC detail lines from both the AP and PY files, summing VAT per valid-format UUID. Invalid / placeholder UUIDs are skipped and counted.
3. **Left-join** each aggregated UUID to the CFDI index and classify it:
   - **Vigente** — found in CFDI with `Estatus = Vigente`.
   - **Cancelado / other** — found in CFDI but not `Vigente`.
   - **Not in CFDI** — no matching invoice.
4. The report is grouped by issuer RFC (DIOT is filed per third party), then by UUID. Non-Vigente and missing rows are surfaced in a review panel but kept in the report with their resolved status.

**Export to Excel** produces the four DIOT columns in sequence: `VAT`, `Emisor RFC`, `UUID`, `Estatus`.

---

## Notes & tolerances

- Amounts are compared in integer cents; Total and VAT use **zero tolerance**, withholding allows **±1 cent**, exchange rate allows **0.1% relative**.
- The amortization ratio test allows split counts **k = 2…36**.
- The `Emisión` date format is sniffed once per file and reused for every row.
- All processing is synchronous and in-memory; no data is uploaded anywhere.
