---
name: file-triage
description: >
  Triage, rename, and categorize documents dropped into the yearly company folder.
  Use this skill whenever the user asks to sort, organize, rename, categorize, or triage
  files or documents in their company folder. Also trigger when the user says things like
  "er staan nieuwe documenten", "check de folder", "nieuwe facturen", "sorteer mijn bestanden",
  or drops files and asks what to do with them. Works for all document types: invoices,
  contracts, tax documents, quotes, official correspondence, and more.
---

# File Triage

You help Dieter organize incoming company documents for Baobab. Documents get dropped into the yearly root folder (e.g. `2026/`) and need to be identified, renamed, and moved to the right subfolder.

## The golden rule

**Always propose before acting.** Never rename or move a file without explicit user confirmation. This is non-negotiable — Dieter wants to review every rename and destination before it happens.

## Workflow

### Step 1: Scan for loose files

Check the root of the yearly folder (e.g. `2026/`) for files that don't belong in a subfolder yet:

```bash
ls /path/to/2026/*.*
```

If there are no loose files, let the user know and stop.

### Step 2: Identify each document

Read each file to understand what it is. For PDFs, read the first page. Look for:
- **Vendor/sender** — who issued the document?
- **Date** — invoice date, document date, or issue date
- **Amount** — total amount including VAT (for financial documents)
- **Type** — invoice, credit note, contract, tax document, certificate, quote, etc.
- **Description** — what is it for? (e.g. "Fiber XL Boosted", "Hospitalisatieverzekering")

### Step 2b: Duplicate detection

Before proposing anything, check whether the document already exists somewhere in the folder structure. A document is a duplicate if it matches on any of these:
- Same vendor + same date + same amount as an existing file
- Same invoice/document number as an existing file
- Very similar filename to an existing file in a subfolder

To check, scan the relevant category folder for the vendor:

```bash
ls "2026/Uitgaven/VendorName/" 2>/dev/null
```

Also check `finances.json` for entries with matching date + amount + category.

If a duplicate is found, flag it clearly in your proposal table with a ⚠️ marker and a note explaining what it matches. For example:

| # | Current file | Rename to | Destination | Note |
|---|---|---|---|---|
| 1 | `invoice.pdf` | — | — | ⚠️ Duplicate: matches `Uitgaven/Telenet/2026-03-02 Telenet Diensten €60,73.pdf` |

Then let the user decide: keep, replace, or delete the duplicate.

### Step 3: Propose rename and destination

Present a clear overview table to the user with your proposals:

| # | Current file | Rename to | Destination |
|---|---|---|---|
| 1 | `original-name.pdf` | `YYYY-MM-DD Vendor Description €amount.pdf` | `Subfolder/Category/` |

**Naming convention:** `YYYY-MM-DD Vendor Description €amount.ext`
- Date in ISO format
- Vendor name (short, recognizable)
- Brief description of what the document is
- Amount with euro sign and comma as decimal separator (e.g. `€59,96`)
- For non-financial documents, omit the amount

**Destination folders** (under the yearly folder):

| Folder | What goes here |
|---|---|
| `Verkoopsfacturen/` | Sales invoices (income) |
| `Uitgaven/Category/` | Expenses, organized by vendor or type |
| `Belastingaangifte/` | Tax documents, fiscal certificates |
| `Contracten/` | Contracts |
| `Documenten/` | Official documents (FOD, Xerius, etc.) |
| `Offertes/` | Quotes |
| `Timesheets/` | Timesheets |

For expenses, the category subfolder is usually the vendor name (e.g. `Uitgaven/Telenet/`, `Uitgaven/Adobe/`). Check existing folders first so you match what's already there.

### Step 4: Wait for confirmation

Do not proceed until the user confirms. They may want to:
- Adjust a filename
- Change a destination
- Skip a file
- Delete a duplicate

### Step 5: Execute

Once confirmed:
1. Create any new subfolders if needed
2. Rename and move each file
3. Log financial documents (expenses or income) to `.claude/finances.json`
4. Regenerate the financial dashboard at `.claude/finances.html`

### Step 6: Update the dashboard

After adding entries to `finances.json`, update the embedded data in `finances.html`:
- Find the `const data = {...};` block in the HTML
- Replace it with the updated JSON data
- Use string slicing (not regex substitution) to avoid escaping issues

## finances.json structure

```json
{
  "year": 2026,
  "income": [
    {"date": "YYYY-MM-DD", "description": "...", "amount": 0.00, "client": "...", "file": "relative/path.pdf"}
  ],
  "expenses": [
    {"date": "YYYY-MM-DD", "category": "Vendor", "description": "...", "amount": 0.00, "file": "relative/path.pdf"}
  ]
}
```

- Amounts are numbers (not strings), using dot as decimal separator
- File paths are relative to the yearly folder
- Category should match the subfolder name under `Uitgaven/`

## Known vendors and categories

These are vendors Dieter regularly deals with. Use this to recognize documents faster:

| Vendor | Category | Type |
|---|---|---|
| Adobe | Adobe | Software subscription |
| Al Jannah | Representatie | Restaurant |
| Alternate | IT Benodigdheden | Hardware/IT supplies |
| Anthropic | Anthropic | AI/API services |
| Bij den Boer | Representatie | Restaurant |
| Billit | Billit | Accounting software |
| Bol.com | IT Benodigdheden | Hardware/IT supplies |
| Combell | Website | Domain hosting |
| Coolblue | IT Benodigdheden | Hardware/IT supplies |
| Cursor | Cursor | IDE subscription |
| DATS 24 | Brandstof | Fuel |
| DKV | VERZ - hospitalisatie | Health insurance |
| Edenred | Edenred | Meal vouchers |
| EDPNet | EDPNet | Internet/telecom |
| ELMOS NV | (income) | Client - React (Native) development work (via FL2025/0712) |
| ENRA | ENRA | Bicycle insurance |
| Gabriels | Brandstof | Fuel |
| Il Rifugio | Representatie | Restaurant |
| Imagen AI | Fotografie | AI photo editing |
| KBC | KBC | Banking |
| Le Cuneo | Representatie | Restaurant |
| Lukoil | Brandstof | Fuel |
| Mobile Vikings | Mobile Vikings | Mobile telecom |
| NMBS | Mobiliteit | Train tickets |
| Office / Microsoft 365 | Office | Microsoft Office |
| Olympus Mobility | Mobiliteit | Public transport |
| OpenAI | OpenAI | AI services |
| Poule & Poulette | Representatie | Restaurant |
| Q8 | Brandstof | Fuel |
| Servauto | Brandstof | Fuel |
| Telenet | Telenet | Telecom |
| TotalEnergies | Brandstof | Fuel |
| Xerius | Sociale Bijdragen | Social contributions |

When you encounter a new vendor not in this list, create a new category folder for them and mention it in your proposal.

## Tips

- When in doubt about the category, ask the user
- Some documents come in pairs (invoice + receipt) — ask if both are needed
- Credit notes have negative amounts
- Watch for duplicate documents and flag them
- Scanned receipts (image-only PDFs) need to be rendered as images to read — use PyMuPDF to convert to PNG and view visually
- Fuel receipts go under `Uitgaven/Brandstof/`, restaurant receipts under `Uitgaven/Representatie/`
