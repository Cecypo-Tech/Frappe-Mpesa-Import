## Frappe Mpesa Import

Imports M-Pesa Utility Account statements (the `.xls` exported from the Safaricom
org portal) into `Mpesa C2B Payment Register`, so payments that never arrived by
webhook or pull can be backfilled from the statement of record.

Built by [Cecypo.Tech](https://cecypo.tech). Requires
[frappe_mpsa_payments](https://github.com/navariltd/frappe-mpsa-payments), which
owns the `Mpesa C2B Payment Register` doctype.

### How the statement is read

A Safaricom Utility Account statement lists **every transaction twice**: once as
a `Pay Bill Charge` line (negative `Withdrawn`, blank `Paid In`) and once as the
actual customer payment (`Paid In` > 0). Only the second is a real receipt.

The importer therefore keeps rows where **`Paid In` > 0**. On a representative
statement that reduces 62 data rows to 29 payments, whose amounts sum exactly to
the `Total Paid In:` figure in the file's own header — the parser checks that
reconciliation so a mis-parsed file fails loudly instead of importing silently.

`Receipt No.` becomes `transid` and is the deduplication key.

### Column mapping

| Mpesa C2B Payment Register | Statement column |
| -------------------------- | ---------------- |
| `transid`                  | `Receipt No.` |
| `transtime`                | `Completion Time`, reformatted to `YYYYMMDDHHMMSS` |
| `transamount`              | `Paid In` |
| `businessshortcode`        | `Short Code:` from the file header |
| `billrefnumber`            | `A/C No.` |
| `transactiontype`          | `Reason Type` |
| `msisdn`                   | `Other Party Info` (masked — see below) |
| `firstname` / `middlename` / `lastname` | `Other Party Info` |
| `currency` / `default_currency` | `Currency` |
| `posting_date` / `posting_time` | `Completion Time` |

`company`, `mode_of_payment` and `full_name` are derived by the target
doctype's own `set_missing_values`, from the shortcode's
`Mpesa C2B Payment Register URL`.

### Statement data is masked

Safaricom masks personal data in statement exports:

```
Other Party Info:  25471****456 - JANE **** DOE
```

Records created from a statement therefore carry a **masked `msisdn`** and no
middle name, where the same transaction arriving by webhook would carry the full
`2547XXXXXXXX` number and the full name. This is stored as-is and is deliberate: the
masked value keeps the record traceable to its statement line and is visibly
distinguishable from live webhook data, so it is never mistaken for a dialable
number. Mask tokens (`****`) are dropped from name fields rather than stored.

Consequently: **never commit a real statement to this repository.** The tests use
a synthetic fixture with fictional data, and `.gitignore` blocks `*.xls` as a
safety net.

### Duplicate protection

`transid` has no unique constraint on the target doctype, so deduplication is
entirely application-level. Four layers:

1. **In-file** — the `Paid In > 0` filter collapses the duplicated charge rows.
2. **Pre-flight** — any receipt still appearing twice among payment rows aborts
   the import before a single record is written.
3. **Per-row** — `transid` already in `Mpesa C2B Payment Register` → *skipped*.
4. **Per-file** — `file_hash` is unique, so re-uploading the same statement is
   rejected outright.

A fifth backstop comes from the target doctype: it refuses transactions already
captured as an `Mpesa Express Request` (STK push). Those are pre-checked and
counted as *blocked* rather than being allowed to raise.

### Importing

1. Open **Mpesa Statement Import**, attach the `.xls`, save. Parsing happens on
   save: shortcode, period and row counts are filled in for you to sanity-check.
2. **Submit.** Rows are imported and the result is written back to the document:

   - **Created** — new records
   - **Skipped** — `transid` already present
   - **Blocked** — already captured as an STK push Express Request
   - **Failed** — errored; the reason is in the summary

Each run writes one **Error Log** entry titled
`Mpesa Statement Import <name>: <n> created, <n> skipped`, with the full per-row
outcome in the body.

> **Note:** imported records follow the same path as live webhook records. If
> `auto_reconcile_c2b` is enabled for the shortcode, submitting an import will
> auto-submit each new record, create a Payment Entry, and run reconciliation.
> Review the parsed counts on save before you submit.

### The restricted uploader

The app ships a `Mpesa Statement Importer` role with create/read/write/submit on
`Mpesa Statement Import` and nothing else — no delete, no cancel, no amend, no
export, and no access at all to `Mpesa C2B Payment Register`.

Create such a user (no password on the command line; Frappe emails a setup link):

```bash
bench --site <site> execute \
    frappe_mpesa_import.setup.install.create_importer_user \
    --kwargs '{"email": "importer@example.com", "first_name": "Statement"}'
```

### Installation

```bash
cd $PATH_TO_YOUR_BENCH
bench get-app https://github.com/Cecypo-Tech/Frappe-Mpesa-Import --branch version-16
bench --site <site> install-app frappe_mpesa_import
```

### Development

```bash
cd apps/frappe_mpesa_import
../../env/bin/python -m pytest        # parser unit tests, no site required
```

This app uses `pre-commit` for formatting and linting (ruff, eslint, prettier,
pyupgrade):

```bash
cd apps/frappe_mpesa_import
pre-commit install
```

### License

MIT
