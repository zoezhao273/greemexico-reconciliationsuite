# Reconciliation Suite

A single, self-contained HTML tool (no install, no server — open `reconciliation_suite.html` in a browser) for month-end reconciliation. It cross-checks NCC accounting documents against the official CFDI invoice file, reconciles the NCC bank journal against the bank statement, and assembles the DIOT compliance report. Everything runs locally in the browser; uploaded workbooks never leave the machine.

The suite has three modules, selectable from the top navigation:

1. **NCC × CFDI Reconciliation** — line-by-line validation of the reimbursement export against the CFDI file (Checks 1–7). The **CFDI file is the source of truth** for invoice status, amounts, exchange rate, and issuer RFC.
2. **Bank Reconciliation** — matches the NCC bank journal against one bank statement (ICBC, Santander, or BBVA), separately for inflows and outflows, and validates against control totals.
3. **DIOT Report Builder** — aggregates VAT per invoice across the AP and PY files and produces the four-column DIOT report.

Throughout, money is compared as integer cents to avoid floating-point noise.

---

# Module 1 — NCC × CFDI Reconciliation

## Inputs

### NCC file (reimbursement export)
A workbook containing a **main table** (header row starting with `PayableBill_$Head` for AP / payable layouts or `PayBill_$Head` for PY / payment layouts) and a **detail table** (header row starting with `BodyS`).

| Table | Columns used |
|-------|--------------|
| Main | `DocNo.`, `DocDate`, `ReimbursementNo.`, `Supplier` (`供应商`), `A/PType`, `PmtType` |
| Detail | `Currency`, `Ex.Rate`, `AmtIncl.VAT(OC)`, `VATCode`, `UUID`, `Material`, `VATAmt(FC)`, `AmtAfterWHT(OC)` |

The first detail column is the line sequence number; each detail line is tied back to its record via that sequence.

### CFDI file (source of truth)
One row per invoice, columns matched by header name (with sensible fallbacks):

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

## UUID handling

Every UUID is normalized before comparison so cosmetic differences never cause a false mismatch: whitespace removed, a stray leading `:` removed, all Unicode dash variants unified to a plain `-`, and upper-cased. A value is treated as a **real UUID** only if it matches the canonical `8-4-4-4-12` hexadecimal format; anything else (`-`, `NA`, free text) is a *no-invoice placeholder* and is never matched or counted as a duplicate.

## Exemptions (records skipped from all checks)

- **Supplier** contains `GREE AIRCONDITIONING MEXICO` or `CDMX GOV` — salary / tax related.
- **PY** document whose `PmtType` is `Cash Advance Payment` or `Cash Avance Return` — not invoice related.
- **Whitelist match** — a reviewed entry on the approved list. If the entry looks like a UUID it exempts that invoice wherever it appears; otherwise it is treated as a `DocNo.` and exempts the whole record.

## The seven checks

"Exact to the cent" means zero tolerance.

### Check 1 — Duplicate UUIDs *(runs first)*
When a real UUID appears on **more than one distinct record**, the two cases are told apart automatically:

- **Amortized invoice (expected, not an error).** Recognized when the per-record portions are roughly equal, each is clearly below the full invoice (`< 95%` of total), the portions sum to no more than the total (`≤ 100.5%`), **and** either the ratio test resolves `Total / portion` and `IVA / vatPortion` to the *same* integer k between 2 and 36, **or** the material is `Bank Charges`. `PPD` payment method is a supporting signal. Amortized records have their Total / VAT / month checks relaxed.
- **Possible duplicate reimbursement (flagged).** The same UUID recorded at the full invoice amount on multiple records.

Placeholder UUIDs and repeats within a single record are never counted as duplicates.

### Check 2 — Valid UUID
A UUID is valid **only when both conditions hold**: (1) it exists in the CFDI file, **and** (2) the matched invoice's `Estatus` is exactly `Vigente`.

- **Not found in CFDI** → flagged. Exception: a line whose `Material = Non-Deductible Expense` is exempt from this case only, since those legitimately have no CFDI.
- **Found but not `Vigente`** (e.g. `Cancelado`, or a blank status) → flagged, showing expected `Vigente` versus the actual status.

An invalid UUID stops the downstream Total / VAT / FX / withholding checks for that line.

### Check 3 — Posting month
NCC `DocDate` month must equal CFDI `Emisión` month. Run **only** when the record is a `DocNo.` starting with `AP` *and* `A/PType = CA Reimbursement`, or when the supplier contains `PETTY CASH`.

### Check 4 — Total amount
NCC `AmtIncl.VAT(OC)` must equal CFDI `Total Original XML`, exact to the cent. A split-by-expense-type invoice is summed within the record first. Booked **below** → *"Please verify the bank payment amount"*; **above** → *"does not match"*.

### Check 5 — VAT amount
NCC `VATAmt(FC)` (summed per invoice within a record) must equal CFDI `IVA`, exact to the cent. Exceptions: VAT equal to **8.5%** of the CFDI IVA (±0.01) is accepted for `Travel – Meals` (SAT 50 km rule); if that pattern matches but `Material ≠ Travel – Meals` it is flagged as `Meal ≤ 50 KM but incorrect material`; skipped for `PY` non-MXN lines.

### Check 6 — Exchange rate
Only when NCC `Currency` is not `MXN`: NCC `Ex.Rate` must equal CFDI `Tipo Cambio` (within 0.1% relative tolerance). Skipped for `PY` payment documents.

### Check 7 — Withholding tax
For lines where `VATCode = PresetWHTL0 Donot used`, the absolute value of `AmtIncl.VAT(OC)` must equal CFDI `IVA Retenido + ISR Retenido` (within one cent).

## Output
On-screen summary cards and per-check breakdown chips, a possible-duplicates panel, an amortized-invoices panel, and a flagged-rows table (non-Vigente rows tinted). **Export to Excel** produces an Issues sheet (`DocNo.`, `Reimbursement No.`, `Material`, `Check`, `What's wrong`, `CFDI (correct)`, `NCC (recorded)`, `UUID`, `Status`) plus a duplicates sheet.

---

# Module 2 — Bank Reconciliation

Matches the NCC bank journal against one bank statement, **separately for inflows and outflows**, then validates both sides against control totals.

## Inputs

### NCC Bank Journal (.xlsx)
Layout is auto-detected (new format vs. legacy) from the header row.

- **New format (0-indexed):** `2 = Voucher No.`, `3 = Summary`, `4 = Currency`, `6 = FC Ex. Rate`, `7 = Debit OC`, `8 = Debit FC`, `9 = Credit OC`, `10 = Credit FC`.
- **Legacy format:** `5 = Debit`, `6 = Credit` (no OC/FC split).

Each row becomes an **inflow** if it has a non-zero Debit, an **outflow** if it has a non-zero Credit. The amount used to match against the bank is the original-currency (OC) value; the functional-currency (MXN) value and the exchange rate are kept for display. Rows **without a voucher number** are headers/subtotals and skipped — except `Current Month Total` rows, which are captured as the inflow/outflow **control totals**.

### Bank Statement — one of three banks
Direction (inflow vs. outflow) is resolved per the bank's own format, and every **outflow** is tagged as **Nómina/Payroll** when its summary matches `nomina` or `payroll`:

| Bank | How direction is determined | Control totals |
|------|-----------------------------|----------------|
| **ICBC** | From the running balance (`Saldo`): balance increase → inflow, decrease → outflow. The first row's direction is resolved by comparing the next balances. | Read from a dedicated control row (inflow, outflow). |
| **Santander** | From the Cargo/Abono sign column: `+` → inflow, `-` → outflow. | Summed from the parsed rows. |
| **BBVA** | From which column carries the amount: `Cargo` column → outflow, `Abono` column → inflow. Dates `dd-mm-yyyy` are normalized; rows without a valid date (opening/carry-forward) are skipped. | Summed from the parsed rows. |

### Nómina/Payroll toggle
A checkbox controls whether bank outflows tagged Nómina/Payroll are included in the outflow reconciliation. It defaults to **included**; toggling it re-runs the outflow match live and updates the counts.

## Matching logic
Inflows and outflows are reconciled independently. For each side, the NCC list and the bank list are sorted by amount and walked with a two-pointer merge:

- **Equal amounts** → paired as a **match** (no note).
- **NCC amount with no bank counterpart** → flagged *"NCC {Inflow/Outflow} but not yet in Bank"*.
- **Bank amount with no NCC counterpart** → flagged *"Bank {Inflow/Outflow} but not yet in NCC"*.

Matching is purely amount-based (not by date or description); it is a greedy pairing over sorted amounts.

## Control-total validation
As a parsing sanity check, the **sum of the individual parsed rows** is compared against the file's **control total** for each direction (the NCC `Current Month Total` rows and the bank statement's control row / summed rows). A side shows **✓ Match** when the two agree within ~1.5 cents, **⚠ Mismatch** otherwise, or **N/A** when no control total is available. A mismatch indicates rows were missed or double-counted during parsing, not a genuine reconciliation break.

## Output
Stats cards (inflow matched / discrepancies, outflow matched / discrepancies), separate **Inflow** and **Outflow** tabs listing paired rows with discrepancy notes, the validation panel of parsed-sum-vs-control-total per direction, and an export of the reconciliation file.

---

# Module 3 — DIOT Report Builder

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

- Amounts are compared in integer cents. Module 1: Total and VAT use **zero tolerance**, withholding **±1 cent**, exchange rate **0.1% relative**. Module 2: control-total validation uses **~1.5 cents** (0.015).
- The amortization ratio test allows split counts **k = 2…36**.
- The `Emisión` date format (Module 1) and the NCC bank-journal layout (Module 2) are each sniffed once per file and reused for every row.
- All processing is synchronous and in-memory; no data is uploaded anywhere.
