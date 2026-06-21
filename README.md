# Reconciliation Suite

A single, self-contained HTML tool (no install, no server — open `reconciliation_suite.html` in a browser) for month-end reconciliation. It cross-checks NCC accounting documents against the official CFDI invoice file, reconciles the NCC bank journal against the bank statement, and assembles the DIOT compliance report. Everything runs locally in the browser; uploaded workbooks never leave the machine.

The suite has three modules, selectable from the top navigation:

1. **NCC × CFDI Reconciliation** — line-by-line validation of the reimbursement export against the CFDI file (Checks 1–8). The **CFDI file is the source of truth** for invoice status, amounts, exchange rate, payment form, and issuer RFC.
2. **Bank Reconciliation** — matches the NCC bank journal against one bank statement (ICBC, Santander, or BBVA), separately for inflows and outflows, and validates against control totals.
3. **DIOT Creation** — aggregates IVA per invoice across the AP and PY files, enriches each UUID with figures pulled from the CFDI, and produces the DIOT compliance report.

---

## Module 1 — NCC × CFDI Reconciliation

### Inputs

**NCC file (reimbursement export).** A workbook containing a **main table** (header row starting with `PayableBill_$Head` for AP / payable layouts or `PayBill_$Head` for PY / payment layouts) and a **detail table** (header row starting with `BodyS`). Columns are matched by header name, not position, because AP and PY layouts order them differently.

| Table  | Columns used |
|--------|--------------|
| Main   | `DocNo.`, `DocDate`, `ReimbursementNo.`, `Supplier` (`供应商`), `A/PType`, `PmtType` |
| Detail | `Currency`, `Ex.Rate`, `AmtIncl.VAT(OC)`, `VATCode`, `UUID`, `Material`, `VATAmt(FC)`, `AmtAfterWHT(OC)` |

The first detail column is the line sequence number; each detail line is tied back to its record via that sequence.

**CFDI file (source of truth).** One row per invoice. Columns matched by header name (with sensible fallbacks):

| Field | Header(s) matched | Used by |
|-------|-------------------|---------|
| UUID | `UUID` | All checks (join key) |
| Estatus | `Estatus` / `Status` (fallback: column C) | Check 2, DIOT |
| Emisión | `Emisión` / `Emision` | Check 3 |
| IVA | `IVA` | Check 6 |
| IVA Retenido / ISR Retenido | `IVA Retenido`, `ISR Retenido` | Check 8 |
| Total Original XML | `Total Original XML` / `Total Original` / `Total XML` | Check 5, Check 6, Check 9 |
| Tipo Cambio | `Tipo Cambio` / `Tipo de Cambio` | Check 7 |
| Forma Pago | `Forma Pago` / `Forma de Pago` | Check 4, Check 9 |
| Conceptos Descripción | `Conceptos Descripción` / `Descripción` / `Concepto` | Check 4 |
| Emisor Nombre | `Emisor Nombre` / `Nombre Emisor` / `Nombre del Emisor` | Check 4 (gasoline detection + CFDI display) |
| Método Pago | `Método Pago` / `Metodo Pago` | Duplicate-vs-amortized signal |
| Emisor RFC | `Emisor RFC` | DIOT report |

> Note: **Forma Pago** (payment *form*: `01` = Efectivo / cash, `03` = transfer, …) and **Método Pago** (`PUE` / `PPD`) are two distinct fields. Check 4 (gas) and Check 9 (cash) read Forma Pago.

### UUID handling

Every UUID is normalized before comparison so cosmetic differences never cause a false mismatch: whitespace removed, a stray leading `:` prefix removed, all Unicode dash variants (en dash, em dash, minus sign, …) unified to a plain `-`, and uppercased. A value is only treated as a *real* invoice UUID if it then matches the canonical `8-4-4-4-12` hex pattern; placeholders such as `-`, `NA`, `tip`, or free text are recognized as no-invoice items and never flagged as duplicates.

### The checks

Reconciliation runs per detail line and per `(record × UUID)` group. A single invoice split across several expense-type lines within one record is **summed within the group first**, so total and VAT are reconciled on the group sum, not line by line.

**Check 1 — Duplicate UUIDs.** Runs first. When a real UUID appears on more than one record, the tool tells two cases apart automatically: equal partial shares that sum within the invoice total (or a `Bank Charges` item / `PPD` payment) are treated as an **amortized invoice** — expected, with total/VAT/month relaxed. Full-amount repeats are flagged as a **possible duplicate reimbursement**. Placeholder UUIDs and repeats within one record are never counted as duplicates.

**Check 2 — Valid UUID.** A UUID is valid only when it satisfies **both** conditions: it **exists** in the CFDI file **and** the matched invoice's `Estatus` is exactly `Vigente`. A stray `:` prefix, spaces, or unusual dashes are tolerated; anything still unmatched is flagged as *not found*. If a UUID *is* found but its `Estatus` is not `Vigente` (e.g. `Cancelado`), the line is flagged — the **CFDI** column shows the actual status and the **NCC** column shows the UUID — and downstream gas/total/VAT/FX/withholding/cash checks are skipped because the invoice is already invalid. A line whose `Material = Non-Deductible Expense` is exempt from the *not-found* case only — these legitimately have no CFDI; a non-`Vigente` status is flagged regardless of material. If the CFDI file carries no status at all, the Vigente sub-check is silently skipped.

**Check 3 — Posting month.** NCC `DocDate` month must equal CFDI `Emisión` month. Runs only when the record is a `DocNo` starting with `AP` *and* `A/PType = CA Reimbursement`, or when the supplier contains `PETTY CASH`.

**Check 4 — Gas material.** Runs when the invoice is gasoline — CFDI `Conceptos Descripción` contains `gasolina` **or** CFDI `Emisor Nombre` contains `GASOLINEROS` (both case-insensitive). The required `Material` depends on the payment form, and the **NCC (recorded)** column is left blank in both cases:

- **Forma de Pago ≠ 01** (not cash) — `Material` must be `Fuel - Private Vehicle` or `Fuel - Operational Vehicle`. Otherwise flagged "For gasolina, Material = Fuel & Supplier = Employee related if it's for private cars"; the **CFDI (correct)** column shows the `Emisor Nombre`.
- **Forma de Pago = 01** (cash) — `Material` must be `Non-Deductible Expense` (SAT disallows the deduction). Otherwise flagged "For cash-paid gasolina, Material = Non-Deductible Expense & Supplier = Employee related if it's for private cars"; the **CFDI (correct)** column shows the `Forma de Pago` value.

**Check 5 — Total amount.** NCC `AmtIncl.VAT(OC)` must equal CFDI `Total Original XML` to the cent (no tolerance), summed within the group first. Booked *below* the invoice reads "Please verify the bank payment amount"; booked *above*, "does not match".

**Check 6 — VAT amount.** Default: NCC `VATAmt(FC)` must equal CFDI `IVA` to the cent (summed per invoice within a record). Exception: for `Travel – Meals`, VAT equal to **8.5%** of the CFDI IVA (±0.01) is accepted; if that pattern matches but `Material ≠ Travel – Meals`, it is still flagged with `Meal ≤ 50 KM but incorrect material`. **Fuel lines** — any line whose `Material` contains `Fuel` — use the prorated gas rule instead: `VATAmt(FC)` must equal `AmtIncl.VAT(OC) × IVA ÷ Total Original XML`. If it matches, the line passes; otherwise it is flagged "Correct Gas VAT = …" with the computed amount — the **CFDI (correct)** column shows the `IVA`, the **NCC (recorded)** column shows the `AmtIncl.VAT(OC)`. Skipped for `PY` non-MXN lines (settled at the payment-date FX rate).

**Check 7 — Exchange rate.** Only when NCC `Currency` is not `MXN`: NCC `Ex.Rate` must equal CFDI `Tipo Cambio` (within 0.1%). Skipped for `PY` payment documents, whose foreign-currency lines use the payment-date rate rather than the invoice rate.

**Check 8 — Withholding tax.** For lines where `VATCode = PresetWHTL0 Donot used`, the absolute value of `AmtIncl.VAT(OC)` must equal CFDI `IVA Retenido + ISR Retenido`.

**Check 9 — Cash payment.** Runs only for invoices whose CFDI `Total Original XML > 2000`. If the CFDI `Forma Pago = 01` (Efectivo / cash; `1`, `01`, or `01 …` are all recognized), the line is flagged "Please verify payment method at first. If paid via cash, select 'Non-Deductible Expense' for payments > $2,000"; the **CFDI (correct)** column shows the `Forma de Pago` value and the **NCC (recorded)** column is left blank. Gasoline is no longer handled here — see Check 4.

### Exemptions (skip all checks)

- Records whose supplier is `GREE AIRCONDITIONING MEXICO` or `CDMX GOV` (salary / tax related), or `MEXICO CUSTOMS` (Pedimento / customs fees).
- `PY` documents whose `PmtType` is `Cash Advance Payment` or `Cash Avance Return` (not invoice related).
- Any UUID or `DocNo.` marked reviewed on the **whitelist** panel. A whitelisted standard SAT UUID exempts that invoice wherever it appears; a `DocNo.` exempts the whole expense record.
- A reviewed whitelist entry that is **neither** a standard UUID **nor** a matched `DocNo.` — e.g. a **foreign-invoice number** such as `26427000000277781101` or `ZSPE26021337A` that legitimately has no CFDI — is matched against the NCC invoice-number column, exempting that line from the *UUID-not-found* flag (and every downstream check) once reviewed.
- Any valid line (UUID **found** and **`Vigente`**) booked as `Material = Non-Deductible Expense` whose CFDI `Forma de Pago = 01` (cash) — a correctly-recorded non-deductible cash expense has nothing left to reconcile, so the line is skipped.

Because the cash and gas checks run inside the grouped pass, they inherit all of these exemptions automatically, as well as the amortized-invoice relaxation from Check 1.

### Output

An **Issue list** of flagged lines (DocNo. · Reimbursement No. · Material · Check · What's wrong · CFDI correct · NCC recorded · UUID · Status), exportable to Excel. Cancelled invoices are tinted red with a status badge. A separate panel lists amortized invoices for visibility (not as errors).

---

## Module 2 — Bank Reconciliation

Matches the **NCC bank journal** against **one bank statement** (ICBC, Santander, or BBVA — autodetected from the uploaded file), separately for **inflows** and **outflows**.

- The NCC journal is read in either the new OC/FC layout (separate original- and foreign-currency debit/credit columns) or the legacy single-amount layout, autodetected.
- Bank parsers normalize each statement's format: ICBC (UTF-16, tab-separated; direction inferred from the running-balance delta), Santander, and BBVA.
- A `Nómina / Payroll` toggle includes or excludes payroll entries from the bank outflow side.
- A **data-integrity panel** compares each side's parsed row sum against the file's own control total (NCC "Current Month Total"; ICBC `合计金额` row) and shows a Match / Mismatch / N/A badge.

Results show inflow/outflow matched counts and discrepancies, with a downloadable Excel export.

---

## Module 3 — DIOT Creation

Cross-references NCC accounting documents (the CA Reimbursement and Pmt Doc For SCM exports) against the received CFDI file by UUID to build the Mexican DIOT compliance report.

- Reads the NCC detail tables and **keeps** a row when `IVA (NCC) ≠ 0`, or when `IVA (NCC) = 0` **and** its `Material` is not `Non-Deductible Expense`. Salary / payroll-tax rows are also excluded: `Material` containing `Fixed Salary` (incl. `Net Salary`), `ISN Expense`, or `IMSS Expense`. A `Material = MX Import VAT` row is **always kept** regardless of those rules; its `IVA (NCC)` is taken from `AmtIncl.VAT(OC)` (not `VATAmt(FC)`) and its `Emisor RFC` is forced to `Mexico Customs`. Rows are kept even when the UUID is a placeholder, a foreign/Pedimento number, or blank.
- Well-formed (8-4-4-4-12) UUIDs are aggregated per folio across both NCC files; any other value is kept as its own per-line row so unrelated placeholders are never merged.
- Aggregates `IVA (NCC)` (`VATAmt(FC)`, or `AmtIncl.VAT(OC)` for MX Import VAT) and collects `Material` and `IVA Rate (NCC)` (`* VATCode`) per row.
- **Enrichment:** when the UUID is a well-formed folio **and** found in the CFDI file, every other field is taken from the CFDI: `Emisor RFC`, `Estatus`, `Base IVA 16`, `Base IVA 8`, `Base IVA 0`, `Base IVA Exento`, `Descuento`, `IVA`, `IVA Retenido`, `ISR Retenido`. Otherwise (non-standard UUID, or folio not in the file) the `UUID` column shows the NCC value **together with its DocNo.** (e.g. `26 16 3167 6015444 | PY-MX-2605-00183`) so the line stays traceable, and every CFDI-sourced field is left blank. The `Material` is shown in its own dedicated first column.
- Adds a `Note` column (only for matched rows) comparing `IVA (NCC)` against the CFDI `IVA` (compared to the cent): **equal** → blank; **greater** → `Wrong, please check AP/PY Doc` (highlighted red); **less** → `reimbursed below the invoice`.
- Outputs the report columns in sequence — **Material · IVA (NCC) · IVA Rate (NCC) · Emisor RFC · UUID · Estatus · Base IVA 16 · Base IVA 8 · Base IVA 0 · Base IVA Exento · Descuento · IVA · IVA Retenido · ISR Retenido · Note** — with a flag panel for matched-but-not-`Vigente` (e.g. Cancelado) invoices, exportable to Excel.

---

## Shared internals

All three modules reuse the same helpers: `normUuid` / `isUuidFormat` (UUID normalization and validation), `parseNum` / `cents` (money parsed and compared as integer cents to avoid floating-point noise), `pickCfdiRows` / `sheetToRows` (header sniffing), and `makeEmisionParser` (file-level CFDI date format detection — the format is detected once from the first dated row and reused for the whole file, so e.g. `2025-12-31` parses as month 12, not day 31).

## Notes

- Pure client-side: open the HTML file in any modern browser; no data leaves the machine.
- Amounts are compared exactly to the cent unless a rule states a tolerance.
- The on-screen **"What gets checked (Validation Rules)"** panel mirrors Checks 1–8 and stays in sync with this README.
