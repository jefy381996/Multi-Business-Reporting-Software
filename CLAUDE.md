# CLAUDE.md — Multi-Business Reporting Software

This file is the standing context for any Claude session working on this repository. Read it fully before writing code. It captures what has already been decided, what is explicitly out of scope, and the rules that protect correctness in a piece of financial software. Do not re-derive or re-litigate anything marked "Decided" below without the user raising it first.

## 0. Current status (read this first)

No code exists yet. The repository contains only this file and a placeholder README. Nothing has been built because of a hard environment constraint (see Section 3). Before writing any GUI code, a session must:

1. Confirm it is connected to the user's actual Windows machine (via Remote Control — `list_computer_folders` / `request_computer_folder` tools). If not connected, **stop and ask the user to connect their machine.** Do not write native Windows GUI code blind on a platform that can't compile or run it.
2. Confirm the .NET SDK (with the Windows Desktop workload) is installed on that machine, or install it as the first setup step.
3. Re-confirm the open items in Section 11 are still open, or get them resolved.
4. Only then start implementation.

## 1. What this project is

A Windows desktop application for **record-keeping and financial reporting** across multiple businesses operated under one owner (one CR / Commercial Registration, or equivalent single-ownership structure). It tracks sales, purchases, expenses, and salaries per business, and produces per-business and consolidated financial reports, including VAT position.

It is a bookkeeping and reporting tool — not an invoicing system, not a tax-filing system, and not a government e-invoicing integration. See Section 2.

## 2. Hard scope boundaries — do not drift

**Explicitly out of scope unless the user asks for it by name in a future conversation:**

- Generating or issuing legally compliant tax invoices (e.g., Saudi ZATCA/Fatoora, UAE e-invoicing, or any other government e-invoicing mandate). This software *records* invoices that already exist elsewhere; it does not create the legal document.
- Submitting VAT returns or any data directly to a tax authority.
- Multi-tenant SaaS infrastructure — billing, per-customer tenant isolation, subscription management. The current purpose is an internal tool for the owner's own businesses; productizing it to sell to others is an open possibility (see Section 10), not a current requirement. Build the data model cleanly and configurably, but do not invest in SaaS platform plumbing until the user confirms that direction.
- Payroll tax computation. "Salaries" is a recording category (an expense type per business), not a payroll-tax or compliance engine, unless the user asks for that explicitly later.

**In scope:** everything in Section 4.

Why this boundary matters: e-invoicing compliance (ZATCA Phase 2, UAE's mandate) is a legally regulated, high-liability domain that competitor research (Section 8) shows even dedicated vendors struggle to verifiably prove compliance with. This project avoids that entire liability surface by design — recording data about invoices, not issuing them. Do not let "add invoice generation" creep back in without the user explicitly re-opening that decision, understanding what it reintroduces.

## 3. Platform & build environment — CRITICAL CONSTRAINT

Target: a genuine **native Windows desktop application**.

This cannot be built or verified from a Linux cloud session. Native Windows GUI frameworks (WPF, WinForms) require Windows to compile, not just to run — a Linux session can write the source files but cannot build them, cannot catch compile errors, and cannot see the UI. Any claim that such code "works" without having actually run it on Windows is false and must not be made.

Rule: **a Claude session only writes or modifies GUI/application code for this project while connected to the user's real Windows machine** via Remote Control. If that connection is not present, do research, planning, and documentation work, but do not write application code, and say plainly that you're blocked on the connection rather than producing untested code.

Once connected, build and test on that machine for real — run it, click through it, verify calculations against hand-checked numbers — before reporting a feature as done.

## 4. Core features (MVP)

1. **Multi-business management.** Add, edit, and list businesses under the owner's umbrella. Each business has its own identity, VAT configuration, and ledgers.
2. **Per-business VAT configuration.** VAT rate and basic rules are set per business at setup time, entered by the user — this software is **country-agnostic by design**, not hardcoded to one jurisdiction's rate or rules. No government e-invoicing or filing integration (Section 2).
3. **Per-business record sections:** Sales, Purchases, Expenses, Salaries. Each is its own ledger scoped to one business.
4. **Sale and purchase invoice entry** — recording invoices that already exist (issued to a customer or received from a supplier), not generating new legal invoices.
5. **Two entry paths for every invoice, user's choice each time:**
   - **Manual entry** — direct, immediate, always available, never dependent on AI/OCR working.
   - **Scan/photo entry** — user provides an image of a scanned or photographed invoice; the system extracts fields intelligently despite invoices having inconsistent, arbitrary layouts (see Section 5 for how and the rules around it).
6. **Entry granularity choice, per invoice:** the user may enter just the invoice total, or break it down line-item by line-item. Both must produce correct totals and correct VAT figures.
7. **Reporting engine:**
   - Per business: total sales, total purchases, total expenses, VAT claimable (input VAT), VAT returnable/payable (net VAT position).
   - Consolidated across all businesses: total sales, total purchases, total expenses, profit/loss, combined VAT position.
   - Consolidation must be a first-class view, not an afterthought bolted onto single-business reports (see Section 8 — this is the industry's biggest unaddressed gap, and the main reason to build this at all).

## 5. Invoice AI / OCR extraction — decided approach

**Decided:** use a cloud vision-capable LLM API (Claude- or Gemini-class), paid per use. The user explicitly chose this over free/local/offline models and over plain OCR, accepting the tradeoffs: a small per-invoice cost, a dependency on internet connectivity, and that invoice images are sent to a third-party API (a privacy consideration for financial documents the user has accepted).

**Non-negotiable design rule:** AI-extracted fields are **always shown to the user for review before being saved** as an authoritative record. Never auto-save unreviewed extraction as final. Show per-field confidence where the API provides it, and visually flag low-confidence fields. This is not a nice-to-have — competitor research (Section 8) found that even products whose *entire business* is invoice OCR (Dext, Hubdoc) still misread dates, miscode currencies, and fail on non-standard layouts. Design the review/correction step as core UX, not an edge case, and expect an ongoing (not one-time) accuracy/exception workflow.

API credentials: the user's own key, handled via a secure credential mechanism. Never hardcode, log, or commit an API key.

## 6. Financial correctness rules — non-negotiable

This is financial software; a correctness bug here is not polish, it's the product failing at its one job.

- **Never use floating-point types (`float`/`double`) for money.** Use a fixed-point/decimal type (e.g., C#'s `decimal`) or integer minor-units throughout. Floating-point rounding error is the single most common source of silent financial software bugs.
- **Every transaction record carries an explicit business/entity identifier.** Consolidated reports are computed by aggregating across businesses at query/report time. Never merge or flatten records in storage in a way that loses which business a transaction belongs to.
- **Compute VAT per business first, then sum for the consolidated view.** Never estimate consolidated VAT from consolidated totals directly — different businesses may have different VAT rates or rules, so claimable/returnable VAT must be correct at the business level before it's ever added together.
- **All calculation logic (totals, VAT claimable/returnable, profit/loss) needs automated tests with known expected values** before being considered done.
- **Pick one explicit, documented rounding rule** (e.g., round per line item vs. round per invoice total) and apply it consistently everywhere. Inconsistent rounding is a common real-world source of VAT-report discrepancies.

## 7. General engineering rules

- Don't add features, sections, or ledger types beyond what Section 4 specifies without the user asking (e.g., no "assets" module, no payroll engine, no multi-currency handling) unless they request it — check Section 11 first; if something seems missing, ask rather than assume it's wanted.
- Keep business/calculation logic separate from UI code so it can be unit-tested without driving the GUI.
- Don't build toward e-invoicing/government integration "just in case" (Section 2) — it's a deliberate exclusion, not an oversight.
- Don't claim a feature is done without having run it on the connected Windows machine (Section 3).

## 8. Competitor research summary (context — why this is worth building)

Researched and sourced during planning (not generic knowledge): **no major competitor does true, first-class multi-business consolidation.** QuickBooks Online and Xero don't have it at all (Xero's own product team says consolidated reporting is "not currently planned"); Zoho Books treats every business as a fully separate paid "organization" and its own community tells users to export to Excel to combine them; Odoo and Tally have a consolidation feature but it's a manual, setup-heavy bolt-on, not a default view. Every competitor also charges per-entity, not per-owner, so running several businesses multiplies the subscription cost with no discount found anywhere (confirmed on QuickBooks' and Zoho's own support channels).

On OCR: even vendors whose *entire product* is invoice/receipt OCR (Dext, Hubdoc) still misread dates, miscode currencies, and — in Hubdoc's case — don't extract line items at all, degrading hard on non-major-vendor invoice layouts. "Reliable AI extraction across arbitrary invoice formats" is not a solved problem anyone has shipped; budget for ongoing exception review, not a one-time build (reflected in Section 5's design rule).

On GCC-specific compliance: Qoyod, Daftra, and Wafeq all claim ZATCA/e-invoicing compliance on their own marketing pages with zero independently verifiable confirmation found. Not this project's problem since it doesn't generate invoices (Section 2) — but relevant context if the user ever pairs this tool with one of those for the generation side.

**The actual opportunity this project is aimed at:** consolidation-first design (Section 4.7) and an honest, review-first AI extraction workflow (Section 5) — the two things the market has consistently left unsolved.

## 9. Advanced / backlog features (not MVP — do not build until asked)

Validated by the gaps in Section 8, kept here so they aren't lost, but **not part of the current build**:

- Bank statement import and reconciliation — matching recorded transactions against actual bank movements. Without this, "correct totals" only means "correctly sums what was typed in," which can still be wrong from missed or duplicate entries.
- Duplicate invoice detection (same vendor + amount + date) and anomaly flags (e.g., a month's expenses spiking well past historical average).
- Role-based multi-user access with a full audit log (who entered or edited which record, when) — matters once more than one person enters data across several businesses.
- Accountant/tax-preparer-ready export formatted the way a tax preparer actually wants it, short of full e-invoicing integration.
- Owner-level pricing (not per-business) — only relevant if this is ever productized to sell to other multi-business owners (Section 10).

## 10. Decisions already made (do not re-ask these)

- **Purpose:** primarily an internal tool for the user's own businesses; productizing it later is an open possibility, not ruled out. Build cleanly and configurably; don't add commercial SaaS infrastructure yet.
- **Jurisdiction / VAT:** not fixed to one country. VAT rate/rules are configured per business at setup. No government e-invoicing or return-filing integration (Section 2).
- **Scope:** record-keeping and reporting only — this software does not generate or issue invoices.
- **Invoice AI:** cloud vision LLM API, pay-per-use, user has accepted the cost/internet/privacy tradeoffs (Section 5).
- **Platform:** genuine native Windows desktop app, to be built only once a session is connected to the user's real Windows machine (Section 3).

## 11. Open questions — resolve with the user, do not assume

- Exact native GUI framework: **recommended default is WPF or .NET MAUI (C#)** — not yet explicitly confirmed by the user beyond "I want a Windows app." Confirm or revisit when the build actually starts.
- Local database engine: **recommended default is SQLite** (single-file, zero server setup, fits a single-machine desktop bookkeeping tool well; revisit if multi-device sync is ever wanted). Not yet explicitly confirmed.
- Which cloud AI provider/API the user wants to use for invoice extraction, and how they'll supply the API key.
- Whether "Salaries" needs anything beyond being an expense-recording category (currently assumed: no payroll-tax logic).
- Whether sections beyond Sales/Purchases/Expenses/Salaries are wanted (e.g., assets, loans) — not currently requested; don't add speculatively.

## 12. How to resume work on this project

1. Check Section 0 for current status.
2. Verify the Windows-machine connection (Section 3) before touching any GUI/application code.
3. Walk through Section 11 — resolve anything still open with the user before assuming a default.
4. Build against Sections 4–7 as the spec; treat Sections 8–9 as context and backlog, not current requirements.
5. Update this file's Section 0 and Section 10 as decisions get made or status changes, so the next session doesn't have to re-derive this conversation.
