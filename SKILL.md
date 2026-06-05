---
name: ai-bookkeeper
description: Monthly bookkeeping reconciliation for a company. User provides a bank statement PDF; Claude appends transactions to a master Google Sheet, hunts invoices in Gmail/Drive, auto-downloads portal invoices via the claude-in-chrome + Playwright MCPs (incl. Stripe-hosted), leaves only physical receipts to the user, then runs final verification. Pure Claude-driven (no app code).
user_invocable: true
---

# Monthly bank → bookkeeping reconcile

A reusable Claude skill that turns a monthly bank-statement PDF into a fully-reconciled bookkeeping sheet with every transaction matched to its invoice — pulling invoices automatically from email and billing portals, and leaving the human only the physical paper receipts.

> **This is a sanitized template.** Replace every `<PLACEHOLDER>` in the Configuration block with your own values. Nothing here is secret; all IDs are yours to fill in. Built and battle-tested against a real EU (Bulgaria) company on First Investment Bank (FIB), but the patterns generalize.

When the user invokes this skill or pastes a bank statement PDF and says "reconcile / close the month", run this workflow: **Phase 1** ingest → **Phase 2** Gmail/Drive invoice hunt → **Phase 2.5** Chrome-MCP + Playwright auto-download from billing portals → **Phase 3** short manual handoff (only physical receipts) → **Phase 4** final verification. Goal: leave the user as little manual work as possible.

## Requirements (MCP / tooling)
- **`gws` CLI** (Google Workspace CLI) — Gmail + Drive + Sheets from the terminal. The skill drives it via Bash.
- **`claude-in-chrome` MCP** — drives the user's logged-in Chrome to download portal invoices.
- **`playwright` MCP** — used ONLY for Stripe-hosted invoice pages (the claude-in-chrome extension blocks `invoice.stripe.com`).
- `pdftotext` (poppler) for quick text extraction; the `Read` tool for visual/structured PDF reading.

## Configuration — fill these in
| Item | Value |
|---|---|
| Master spreadsheet ID | `<MASTER_SHEET_ID>` |
| Transactions tab name / sheetId | `<TX_TAB>` / `<TX_SHEET_ID>` |
| Monthly-summary tab name / sheetId | `<SUMMARY_TAB>` / `<SUMMARY_SHEET_ID>` |
| Company name / Tax ID / VAT | `<COMPANY>` / `<TAX_ID>` / `<VAT>` |
| Bank IBAN (statement currency) | `<IBAN>` |
| Drive upload folder ID (flat dump) | `<UPLOAD_FOLDER_ID>` |
| Scanned mailbox (what `gws gmail` reads) | `<scanned-inbox@example.com>` |
| Accounting group/alias (delivers into the scan) | `<accounting@example.com>` |
| Local Downloads path | `<~/Downloads>` |
| Office scanner sender (for physical receipts) | `<scanner@example.com>` |

> **Mailbox note:** the `gws` CLI reads ONE authenticated mailbox. Anything that *delivers into* that mailbox (aliases, group memberships set to "Each email") is searchable. Verify which addresses actually land there — don't assume.

---

## Master sheet schema (Transactions tab)
| Col | Header | Format |
|---|---|---|
| A | Value date | **DATE** (write `YYYY-MM-DD`, stored as a date serial) |
| B | Counterparty | text |
| C | Amount (statement ccy) | signed number (− debit / + credit) |
| D | Amount (local ccy) | signed number |
| E | Description | clean text (strip PDF line-wrap garbage) |
| F | Invoice | checkbox (TRUE = invoice attached) |
| G | Period | **DATE = first-of-month** (write `YYYY-MM-01`; displays `YYYY-MM`) |
| H | Invoice ref | text (file name / invoice number) |
| I | Note | text |

> ⚠️ **GOTCHA #1 — columns A and G are DATE SERIALS, not text.** Verify with `valueRenderOption:UNFORMATTED_VALUE` (you'll see e.g. `46143` = 2026-05-01). The monthly-summary SUMIFS match on these serials. **Write G as `2026-05-01` (USER_ENTERED → date), NOT the text `"2026-05"`** — text won't match and the rollup silently reads zero.

## Two-date rule
Bank statements show BOTH a booking date and a value date. **Store the value date in column A; the Period (G) uses the booking month.** For card payments the value date = the merchant's real charge date (booking can lag 1–7 days); the vendor invoice date ≈ value date.

---

# Phase 1 — Ingest the statement
1. Read the PDF with the `Read` tool (chunk long statements via `pages:`).
2. Extract every transaction: booking, value date, debit/credit (both currencies), reference, counterparty, description, card mask.
3. **Verify totals against the statement's own period footer** — if it mismatches, find the missing/extra row before writing (a €100 gap is usually one missed ad-platform row).
4. Read existing `<TX_TAB>!A:I` to find the last row; build new rows sorted by value date; skip duplicates (reference + amount + value date).
5. Write with `gws sheets spreadsheets values update … valueInputOption:USER_ENTERED`.
6. Apply checkbox validation to the new column-F range (`setDataValidation`, BOOLEAN, strict).
7. Add a summary-tab row using SUMIFS (copy the existing pattern; criteria reference the first-of-month date).
8. Re-read the summary tab and confirm debit/credit totals match the footer **exactly**.

## Auto-tag during ingest (set column F)
TRUE (no follow-up needed): income/received transfers, taxes, ATM withdrawals, any bank fee, dividends, internal transfers, social-security. Real vendor expenses stay **FALSE** until an invoice is attached — this keeps the "% covered" metric meaningful.

---

# Phase 2 — Hunt invoices in Gmail / Drive
For every FALSE vendor-expense row, search in order:
- **Drive upload folder** (flat — no per-month subfolders): list `'<UPLOAD_FOLDER_ID>' in parents`. Watch for next-month strays already there.
- **Drive fullText** for unique refs (ASINs, transaction IDs).
- **Gmail**: `from:<vendor> has:attachment after:<date> before:<date>`; fetch the PDF attachment; decode base64; save; upload to `<UPLOAD_FOLDER_ID>`.
- **Local `<~/Downloads>`**.

**Receipt vs invoice:** attach actual **invoices**, not receipts — UNLESS a "receipt"-titled file contains a formal invoice number + both VAT numbers + reverse-charge clause (then it's valid). Skip order confirmations, shipping notices, and proformas.

**When you find one:** Read the PDF → verify vendor + amount (allow FX for USD vendors; note known discounts) + date ≈ value date → upload if not already in Drive → tick `F<row>` → fill `H<row>`.

---

# Phase 2.5 — Auto-download portal invoices (claude-in-chrome + Playwright)

## Setup
1. Load chrome tools via `ToolSearch`. Call `list_connected_browsers`, then **`AskUserQuestion` listing every connected browser** (mandatory) and `select_browser`. **Verify you're on the user's machine** — connections can change mid-run; never act in someone else's browser.
2. `tabs_context_mcp({createIfEmpty:true})`.

## 🔑 Pre-flight login (login is the real bottleneck)
**The automation CANNOT log in** — entering passwords/2FA/CAPTCHA is prohibited; only the user can. So **batch it**: open ALL portal URLs in tabs first, tell the user "log into any that show a login screen, then say ready", wait, THEN run the fetch. Turns N scattered "BLOCKED: not logged in" interruptions into one upfront pass. Ask the user to enable "stay signed in".

## Optional: run with a sub-agent
Spawn one `general-purpose` agent (model `sonnet`) that inherits the connected browser. Rules to give it: **download invoices ONLY** (never buy/cancel/change-plan/submit/log-in/solve-CAPTCHA); give up after 2–3 tries; report a table (vendor | amount | status | filename | on-screen invoice no). **Do NOT trust its "DOWNLOADED" blindly** — when it returns, `ls -lat <~/Downloads>/*.pdf` and read each PDF to verify amount/date/recipient before filing (agents mis-record filenames).

## ⚠️ GOTCHA #2 — the period gotcha (Google + arrears-billed SaaS)
Charges that post around the **1st of the month are for the PREVIOUS month's usage** (billed in arrears). So a charge with **value date = 1st** needs the **previous month's** invoice. **Match by amount, not "latest invoice".** Confirmation trick: pull the NEXT month's statement and check the 1st-of-next-month charge equals the current month's invoice total.

## ⚠️ GOTCHA #3 — Stripe-hosted invoices need Playwright, NOT the Chrome extension
`claude-in-chrome` hard-blocks `invoice.stripe.com` (payment-domain denylist) — `find`/`click` return "site is blocked". `curl` only gets a JS stub. **But the `playwright` MCP has no such denylist** and Stripe hosted-invoice URLs are public (tokenized, no login):
1. With claude-in-chrome (logged in), reach the Stripe invoice and grab its tokenized `invoice.stripe.com/i/acct_…/live_…` URL from `tabs_context_mcp`.
2. Hand the URL to Playwright: `browser_navigate` → `browser_snapshot` → `browser_click` the **"Download invoice"** button. It saves to `~/.playwright-mcp/Invoice-<NUMBER>.pdf`.
3. Read/verify → upload → tick.

## Vendor automation matrix (what actually works — verify for your own setup!)
| Channel | Typical vendors |
|---|---|
| **Email → Phase 2 (Gmail)** | most infra/SaaS that emails a PDF (hosting, Notion, Anthropic, Cloudflare, accountant, co-working) |
| **Chrome MCP fetch** (logged in) | Google Workspace/Cloud admin, Meta ads billing hub, country-specific invoicing portals |
| **Playwright** (Stripe) | OpenAI/ChatGPT consumer, Soniox top-ups, any Stripe-hosted invoice |
| **DocuSign** | works via Chrome when logged in (Billing history → click the INV link) |
| **Scanner → email** | physical/in-store receipts (the office scanner emails them) |

> **Don't assume "it's emailed" — grep the inbox first.** Many "should email" vendors don't reliably deliver a usable PDF to a scanned mailbox. Researched reality for several: OpenAI/Soniox/DocuSign have **no** customer email/API for invoices (Stripe-hosted or paid-API-only) → browser fetch is the legitimate answer. Meta card accounts only email a *notification* (no PDF). Google Workspace can be made to email invoices by adding a scanned address as an "email-only payments user" in the Google Payments profile (and **accepting** the invite — expired/unaccepted invites silently stop delivery).

---

# Phase 3 — Manual handoff (only what nothing else can get)
After Phases 2 + 2.5, the only FALSE rows left should be **physical / in-store purchases** with no online portal. List them (row, date, vendor, amount) and ask the user to drop the paper/PDF receipt into `<UPLOAD_FOLDER_ID>`. If a portal merely wasn't logged in, ask the user to log in once and re-run Phase 2.5 rather than punting to manual.

# Phase 4 — Final verification
Re-list the upload folder for new files → read each → match to remaining FALSE rows → tick. Re-read the summary tab: debit/credit still match the footer, "with invoice" went up, "without invoice" trends to 0. Output a per-period report (debit, credit, net, % covered) + any anomalies (amount mismatches, FX, gift-card discounts) noted in column I.

---

# Hard-won lessons (the reason this skill exists)
1. **Date serials in A/G** — text dates silently break the SUMIFS rollup.
2. **Period gotcha** — 1st-of-month charges = previous month's invoice; match by amount.
3. **Stripe → Playwright** — the Chrome extension blocks Stripe; a second browser-automation MCP bypasses it cleanly.
4. **Login is the bottleneck** — automation can't enter passwords; batch logins up front.
5. **Verify agent downloads** — re-read every PDF; don't trust reported filenames.
6. **Email-routing has a ceiling** — research each vendor; ~half genuinely offer no customer email/API. The robust pipeline is hybrid: Gmail + browser + Playwright + scanner = 100% coverage.
7. **Two backups** — uploading to Drive is a copy, not a move; local copies remain. The Drive folder is the permanent archive, independent of whether any email ever arrives.

---

# When NOT to use
- One-off questions about a single transaction → answer directly.
- Statements from a different bank → adapt the column/terminology mapping first (this template assumes one bank's layout).
