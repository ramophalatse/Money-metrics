# Money Metrics — Claude Operating Rules

## Security
- `.env` holds all secrets (sheet IDs, API keys) — never store credentials elsewhere
- `credentials.json`, `token.json` are Google OAuth files — gitignored
- Before running any script that makes paid API calls or uses credits, check with the user first

## Budget periods
- Period = 19th of Month A → 18th of Month B, named after Month B
- e.g. 7/19/2025 → 8/18/2025 = "August 2025"

## Ledger schema (cols A–R)
```
A  Reviewed (checkbox)    B  TXN_ID         C  Date           D  Bank
E  Account                F  OrigDesc        G  CleanDesc      H  Amount
I  Direction              J  SpendingGroup   K  Category       L  Type
M  Status                 N  Destination     O  ImportDate     P  Period
Q  MM Sent (checkbox)     R  Notes
```
- TXN_ID format: TXN-XXXXX (5 digits); always use max(), not last-row
- Dedup key: Date + OrigDesc + Amount + Bank (case-insensitive)
- Data starts at row 5 (rows 1–4 are title/subtitle/blank/headers)
- **Status (col M)**: classification confidence only — `Categorised` | `UNRESOLVED` | `ESCALATE`.
  Never use `Reviewed` or `Unreviewed` — review state lives in the col A checkbox.
- **MM Sent (col Q)**: checkbox set TRUE by `mm_write.py` after a row has been written to Money Metrics.
  Notes (col R) is free for human-readable context only — never write MM tags there.
- Any script writing to specific row numbers must guard `if sheet_row_1idx < 5: continue`
  to protect the title/subtitle/blank/header rows from accidental data writes.

## Amount formatting
- Ledger col H: `"R"#,##0.00` currency format — handled automatically by `append_ledger()`
- Money Metrics cells: plain float only (e.g. `2354.0`) — never R-formatted strings like `"R2,354.00"`

## Money Metrics tabs
- `Review Fixed Transactions` — Fixed expense actuals (col I), fixed income actuals (col U)
- `Review Variable Transactions` — Variable actuals by category (col I, rows 883–903)
- `Add Exceptional Expenses` — append to A:I via `write_exceptional_expenses()`; never re-append rows already written
- `Add Adhoc Income` — same; append-only, never re-run without checking for duplicates

## Add Exceptional Expenses — rules
- **Description format**: use `: ` (colon-space) as a separator, never em dashes (`—`). E.g. `Loan: Maps (Sister)`, not `Loan — Maps (Sister)`.
- **Category**: must exactly match one of the valid dropdown options in `L7:L72` on the tab. Names are case-sensitive: `Charitable donations`, `Home improvement`, `Business expenses`, `Third party` — not title-cased variants.
- **Type**: always populated by the IF-formula in col G (auto-assigned by `write_exceptional_expenses()`). Never hard-code a Type value.
- **Budget = Actual**: set Budget (col H) equal to Actual (col I) on every new row. The user manually adjusts Budget where planned spend differed.
- **No pre-populated empty rows**: do not add blank rows as placeholders; add rows only when real data exists.
- **Ledger ↔ MM description sync**: `mm_write.py` deduplicates by matching Ledger `CleanDesc` (col G) against MM col E. Whenever a description is changed in MM (during audits or corrections), the corresponding Ledger `CleanDesc` must be updated in the same session — and vice versa. Divergence causes re-appends on the next `mm_write` run. Exception: aggregated Gifts rows use period-level dedup (handled automatically in `mm_write.py`), so individual gift row descriptions do not need to match the combined MM entry.

## Tools
- Always check `tools/` for existing scripts before writing new ones
- `.tmp/` is for disposable intermediate scripts
- Run tests with: `.venv/bin/pytest tests/ -v`
- When a script fails: fix it, verify the fix works, then update the relevant workflow in `workflows/` so the same issue doesn't recur
- Never create or overwrite files in `workflows/` without asking first — workflows are standing instructions and must be preserved
- `tools/update_cf.py` — re-applies conditional formatting rules to the Ledger tab (review state colours: green/yellow/red/amber). Run if row highlighting looks wrong or after any structural column change. Not part of the routine import workflow.

## Web app (app/)
- `app/` is a responsive web app (React + Vite + Supabase Postgres) mirroring the Ledger;
  installable as a PWA on iPhone/iPad. Optional Tauri Mac shell still works.
  Design + roadmap: `app/DESIGN.md`; setup + deploy: `app/README.md`.
- `tools/app_sync.py` mirrors Ledger -> Supabase (read-only on Sheets, idempotent upsert
  on TXN_ID). Run it after statement imports once Supabase is configured.
- Sheets remain the source of truth; in-app transaction edits are overwritten by the next sync.
- `app/node_modules` and `app/src-tauri/target` are symlinks to `*.nosync` dirs so iCloud
  ignores them: never replace with real folders.
- Keys: `app/.env` has the anon key (`VITE_SUPABASE_URL`, `VITE_SUPABASE_ANON_KEY`);
  root `.env` has the service key (`SUPABASE_URL`, `SUPABASE_SERVICE_KEY`, importer only).
  Auth: single Supabase user, sign-ups disabled; RLS policies are authenticated-only.

## Context Management
When context exceeds 50%, suggest starting a new conversation or using subagents for independent
tasks. One task per conversation — finish, then start fresh.

Proactively recommend context-saving strategies:
- Use file reads instead of asking the user to paste content
- Suggest `/compact` when context is heavy
- Recommend subagents for research or exploration tasks
- Flag when a reference file would be better than inline instructions
