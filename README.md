# ai-bookkeeper — a Claude skill for monthly bookkeeping

Turn a **bank-statement PDF** into a fully reconciled bookkeeping sheet where every
transaction is matched to its invoice — with invoices pulled **automatically** from
email and billing portals, so the only thing left for a human is the physical paper
receipts.

This is a [Claude Code](https://claude.com/claude-code) **skill** (a `SKILL.md` the
agent loads on demand). It's pure prompt + tool orchestration — no application code to
deploy. It was built and battle-tested against a real EU company and refined over a full
month-end close; the file ships the hard-won gotchas inline.

---

## What it does

1. **Ingest** — reads the bank statement PDF, extracts every transaction, writes them to
   a master Google Sheet, and reconciles the totals against the statement footer to the cent.
2. **Hunt** — searches Gmail + Google Drive for each vendor invoice.
3. **Auto-download** — for invoices that only live in a billing portal, it drives the
   user's logged-in Chrome (via the `claude-in-chrome` MCP) and, for Stripe-hosted
   invoices, a second browser via the `playwright` MCP, to download the PDFs.
4. **Handoff** — leaves the user only the physical/in-store receipts to scan.
5. **Verify** — re-checks coverage and totals, and reports the month.

Result in practice: **100% of transactions matched to invoices**, with the human doing
~2 minutes of work (uploading a couple of paper receipts).

---

## Requirements

| Tool | Why | Notes |
|---|---|---|
| **Claude Code** | runs the skill | put `SKILL.md` where Claude can load it (below) |
| **`gws` CLI** | Gmail + Drive + Sheets from the terminal | the skill drives it via Bash |
| **`claude-in-chrome` MCP** | downloads invoices from logged-in portals | the user must be logged into the portals |
| **`playwright` MCP** | Stripe-hosted invoices only | the Chrome extension blocks `invoice.stripe.com`; Playwright doesn't |
| `pdftotext` (poppler) | fast text extraction | the `Read` tool handles visual/scanned PDFs |

A Google Workspace account (for `gws`) and a single "scanned" mailbox that aggregates
your vendor emails make this sing.

---

## Setup

1. **Install the skill.** Copy `SKILL.md` into a skill folder:
   - User-level: `~/.claude/skills/ai-bookkeeper/SKILL.md`
   - or Project-level: `<repo>/.claude/skills/ai-bookkeeper/SKILL.md`
2. **Fill the Configuration block** at the top of `SKILL.md` — replace every
   `<PLACEHOLDER>` (sheet IDs, tab names, Drive upload-folder ID, company tax IDs,
   bank IBAN, the scanned mailbox, your Downloads path). None of these are secret;
   they just point the skill at your own accounts.
3. **Create the master sheet** with two tabs:
   - a **Transactions** tab with the 9 columns described in `SKILL.md`
     (⚠️ columns A and G must be real **dates**, not text — see Gotcha #1),
   - a **Monthly** summary tab with SUMIFS rows.
4. **Create one flat Drive folder** for invoice uploads and put its ID in the config.
5. **Connect the MCPs** (`gws`, `claude-in-chrome`, `playwright`) and log Chrome into
   your billing portals once (Meta, Google admin, Cloudflare, etc.).

Then just paste your bank statement PDF into Claude and say **"reconcile this month"**.

---

## The gotchas it encodes (the actually-useful part)

- **Date serials, not text** — Google Sheets stores dates as serials; writing `"2026-05"`
  as text silently breaks the monthly SUMIFS rollup. Write real dates.
- **Period gotcha** — SaaS charges that post on the **1st** are for the **previous**
  month's usage (billed in arrears). Match the invoice **by amount**, not "the latest one".
- **Stripe needs Playwright** — `claude-in-chrome` blocks `invoice.stripe.com` (payment
  domain). A second browser-automation MCP downloads them cleanly. `curl` only gets a JS stub.
- **Login is the bottleneck** — the agent can't enter passwords/2FA; batch all portal
  logins into one up-front step.
- **Verify agent downloads** — re-read each PDF; sub-agents mis-report filenames.
- **Email-routing has a ceiling** — research each vendor; many (e.g. OpenAI/Soniox/DocuSign
  consumer plans) offer no customer email/API for invoices. The robust answer is a hybrid
  pipeline: Gmail + browser + Playwright + scanner.

---

## Security / privacy

The shared `SKILL.md` is **sanitized** — all IDs, emails, IBANs, tax numbers and local
paths are `<PLACEHOLDER>`s. Fill in your own. The skill performs **read + download +
sheet-write** operations and is explicitly instructed to **never** make purchases, change
plans, modify settings, submit forms, or enter credentials — those stay with the human.

---

## License

MIT — do whatever you want, no warranty. Adapt the bank-statement parsing and vendor
matrix to your own bank and stack.
