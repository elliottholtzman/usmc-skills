---
name: dts-audit
description: >-
  Assembles a DTS travel-voucher lodging audit for a service member on a Government Travel Charge
  Card (GTCC / Citibank CitiManager). It reconciles the traveler's hotel folios against their GTCC
  charges, organizes each receipt with its matching Citibank document into one folder per pay
  period, computes the nightly rate, builds a personal Excel tracker, emails the finished package to
  the traveler's own military address, and when a folio is missing emails the hotel to request it.
  Use whenever someone needs to prepare, reconcile, or organize lodging receipts and GTCC statements
  for a travel claim or "DTS audit" - trigger on "DTS audit", "lodging receipts", "GTCC",
  "CitiManager", "hotel folio", "pay period", "nightly rate", or any request to match hotel receipts
  to Citibank charges and package them by pay period, even if this skill is not named.
---

# DTS Lodging Audit (GTCC)

For any AI with a web browser, email access, and the ability to run Python / handle files. Paste
this whole file in as your instructions and work top to bottom.

Reconcile a traveler's hotel folios against their Government Travel Charge Card (GTCC / Citibank
CitiManager) charges, organize them by pay period, and email the finished package to the traveler
for upload to Teams/DTS.

## Endstate
- One folder per pay period named `<LastName> <START> to <END>` in **sortable `YYYY-MM-DD` dates**
  (e.g. `Smith 2026-04-08 to 2026-04-23`) so the folders order chronologically, each holding that
  period's **folio** and its **matching Citibank document**, named for what they actually contain.
- A courtesy **Excel** tracker (nightly rate = GTCC charge ÷ nights).
- An **email to the traveler's military address** carrying the package as **loose files named by pay
  period** — DoD mail blocks `.zip`, can't attach a folder, and often chokes on Office files, so
  convert any `.xls` to PDF and let the recipient rebuild the folders from the names — plus the Excel as
  its own file. Optionally include a single **combined PDF** (cover → one section per period with that
  period's folio + Citibank charge) as a human-readable backup.
- When a folio is missing, a **request email to the hotel** for the gap dates.

Stop at the email; uploading to Teams/DTS is the traveler's step.

## Inputs
The traveler's **rank, first name, last name, and military email** — the emails are signed with the
rank and name, and the .mil is where the package goes.

## Setup — settle email before anything else
Email gates both halves of this job (pulling the folios and sending the package), so resolve it up
front, not mid-run:
1. **Search your available connectors/tools for an email integration** (Gmail, Outlook, iCloud, IMAP).
2. **If one is connected,** use it for reading the inbox and sending — done, move on.
3. **If none is found, say so plainly** ("no email connector is available") and offer the fallback:
   *"I can work through your webmail in the browser instead — want me to set that up?"* On yes, have
   the traveler open their inbox in the browser you're driving and use that tab for both the folios
   and the outgoing mail. If their `.mil` isn't reachable in a browser, plan to hand them the final
   package email to send themselves.

Also confirm you have a browser for CitiManager. Then gather the inputs above and begin.

## Rules
- **Never enter the traveler's Citibank (or any) credentials.** They log in; you wait.
- **Verify every recipient, then send — don't ask whether to send.** Sending the two emails is part
  of the endstate: check the .mil against what the traveler gave you and the hotel address against
  the one on their folios, then send. Pause only if you truly can't send (no email tool) or can't
  verify a recipient — then hand the traveler a ready draft.
- **Never invent a folio, a night count, or an amount.** Mark unknowns and flag missing receipts.

## The model
A **pay period = one hotel charge on the GTCC.** Match a folio to its charge by the amount: the card
charge equals the folio's settlement/total. If the stay was billed in a foreign currency, the card
line also shows an exchange rate and a ~1% cross-border fee, and `card_amount × exchange_rate ≈
folio_foreign_total` — use that to match. Within a folio, **nights = the count of `Room Charge`
lines**, the **stay window = first→last room-charge date**, and **nightly rate = charge ÷ nights**.
Some deposits are intermediate top-ups with no emailed folio — those are the "missing" receipts.
Match by the **settlement amount, not the email date**: when you ask the hotel for a missing folio they
often re-issue one **comprehensive folio covering the whole reservation** (several deposits at once).
That's legitimate — file it under the period whose deposit it settles, label it by its **actual** date
span, and **never edit, split, or delete pages from an official folio**; a doctored-looking receipt fails
the audit. One folio overlapping a later period's folio is fine; just don't store the same file twice.

## Procedure

1. **Get into CitiManager (the traveler's step).** Open
   `https://home.cards.citidirect.com/CommercialCard/login`. The traveler enters User ID + password
   → `Sign On` → picks a passcode method on `Select OTP Option` → `Continue` → types the code on
   `Enter OTP` → `Continue`, landing on Home. The `System Use Notification` (FOUO) banner that pops
   up is a routine info notice — **you** clear it with `OK`. If your tool can't load a financial URL
   cold, have them open it and hand you the logged-in tab.

2. **Pull every hotel charge.** Note how the hotel appears in the transaction details, then gather
   all its charges from three places: **Recent Activity → Recent Authorization(s)** (pending),
   **Recent Activity → Unbilled Transaction(s)** (current cycle), and **Statements → each month tile
   → Billed Transactions** (older). Record transaction date, posting date, amount, and — for a
   foreign stay — the exchange rate and any ~1% `CROSS BORDER PROCESSING` fee (note it separately;
   it's not part of the nightly rate). Ignore card-payment rows (the traveler paying down the card)
   and non-hotel merchants.

3. **Get one Citibank document per charge — prefer a cropped statement PDF over a screenshot.** Live
   screenshots often can't be saved to a file (the bank's page may also block in-browser capture), but
   downloads always work. **Billed** charge → open its month tile → `DOWNLOAD` → **PDF**, then crop that
   page to the statement header plus the single highlighted charge row: that is the cleanest per-charge
   proof ("a screenshot of your statement showing those exact charges"). **Unbilled** (current cycle) →
   `DOWNLOAD` → **CSV/Excel** as an *interim* doc, and produce the cropped statement PDF once the cycle
   closes and the charge lands on a statement. **Pending** authorization → no document yet; it posts to
   the next statement. A green `File downloaded successfully` banner confirms each download.

4. **Get the folios and read off nights.** Search the traveler's email for the hotel name plus
   `invoice` / `folio` / `receipt`; each folio is a PDF showing arrival/departure, the daily room
   charges, and a closing card-payment/settlement line. Run `parse_folios.py <folder>` to extract
   arrival/departure, the night count, and the settlement from every folio, then **match each folio
   to its charge** by the amount.

5. **Handle gaps.** More charges than folios means intermediate deposits the hotel never emailed — a
   date window (between two folios, or before the first folio's billing starts) with no folio. For
   each, still build the folder with its Citibank doc, mark the receipt MISSING, and leave nights
   blank. Record the exact start/end of each gap window for the front-desk request.

6. **Assemble the package.** One folder per charge named `<LastName> <START> to <END>` in **sortable
   `YYYY-MM-DD`** dates so folders order chronologically (gap periods take the window between adjacent
   folios). Name each file for **what it actually contains** — a folio's true date span, the statement it
   was cropped from — not an assumed window; if a comprehensive folio covers several periods, say so in
   its name (e.g. `Folio 2026-03-29 to 2026-04-23 (full reservation).pdf`). Build the Excel with
   `python build_tracker.py periods.json "RANK First Last" "DTS Audit Tracker.xlsx"` (the script's
   docstring defines `periods.json`).

7. **If any folio is missing, email the hotel** — template B — for the exact gap windows, to the
   billing/front-desk address printed on the folio.

8. **QC before you send.** Open every file and confirm it belongs to its pay period (folio settlement =
   GTCC charge; the Citibank doc shows that charge), that no file is duplicated across folders, and that
   every name matches the file's real content. Don't assume — read.

9. **Deliver to the traveler's `.mil`** — template A. Attach the **loose files named by pay period** and
   the Excel (convert any `.xls` to PDF first; never a `.zip` or a bare folder), optionally with the
   combined PDF. Verify the recipient, then send.

## Running it again
These audits recur through a deployment. On a re-run, read what already exists in the destination,
add only new pay periods, and upgrade interim CSV/XLS exports to a cropped statement PDF once a cycle
closes — don't duplicate or clobber.

## When reality differs
The specifics above are the known-good path, not a rigid script. If the UI, statement cadence, folio
layout, or your available tools differ, navigate by what's on screen and keep the **endstate** fixed.
If a capability is missing (e.g., you can't reach a `.mil` inbox), produce everything you can, hand
the final send to the traveler, and say exactly what's left.

## Email templates
Fill the `<...>` placeholders from what you gathered. (Recipient-verify rule above still applies.)

### A. Package email — to the traveler's military address (always)
To: `<traveler .mil>` — Subject: `DTS Lodging Audit Package - <RANK First Last>`
```
Attached is my DTS lodging audit, organized by pay period for upload to S-1 > DTS Audits:
- Loose files named by pay period (each period's folio + matching Citibank document) - rebuild one folder per period from the names
- DTS Audit Package.pdf - the same folios and charges combined into one PDF, as a readable backup
- DTS Audit Tracker.xlsx - nightly rate (GTCC charge / nights), for personal tracking

Pay periods (GTCC):
  <START>-<END>    <charge>    <nights> nights    <rate>/night
  ... (one line per period)
  Total: <total>

Outstanding (omit if none):
  - Folio not yet received for <window(s)>; requested from the hotel on <date>.
  - <window> Citibank charge is still a pending authorization; statement PDF available after <next statement date>.

V/r,
<RANK First Last>
```

### B. Hotel folio request — only when a folio is missing
Send to the billing / front-desk address printed on the folio.
Subject: `Request for folio copies - <RANK Last>, Room <room> (<N> period(s))`
```
Dear Front Desk / Billing,

Thank you for your assistance during my stay. For a travel expense audit I need copies of my folio
for the following period(s) I don't have on file:

  <START> - <END>
  ... (one line per missing window)

For reference: Room <room>, reservation/loyalty number <number if shown>, email <traveler email>. I
already have the folios covering <windows on file>, so the above are the gaps.

Could you please email me the folio copies (PDF) at your earliest convenience? I'd appreciate it greatly.

Thank you,
<RANK First Last>
United States Marine Corps
```

## Scripts
Save each block to a `.py` file and run as shown. Both need `pip install openpyxl pdfplumber`.

### build_tracker.py
```python
#!/usr/bin/env python3
"""
Build the courtesy DTS lodging Excel tracker.

Usage:
    python build_tracker.py periods.json "RANK First Last" "out.xlsx"

periods.json is a list of pay-period objects. Only `gtcc` is strictly required;
leave `nights` blank ("" or null) for gap periods whose folio hasn't arrived yet
and the nightly-rate formula will stay blank until the traveler fills nights in.

[
  {
    "pp": 1,
    "charge_date": "04/08/2026",
    "stay_dates": "30/03/26 - 07/04/26",   # optional; "" if unknown
    "nights": null,                          # int, or null/"" if unknown
    "gtcc": 1917.43,                         # USD amount charged to the GTCC
    "posting_date": "04/10/2026",            # or "PENDING"
    "fx": 60.0,                              # foreign-currency per USD; "" if domestic/pending
    "cross_border_fee": 19.17,               # USD; "" if none/pending
    "receipt_status": "MISSING",             # "FILED" or "MISSING" or free text
    "notes": "Intermediate deposit; folio not emailed - request from front desk."
  }
]

Requires openpyxl. Produces zero formula errors (rate is guarded against blank nights).
"""
import json, sys
from openpyxl import Workbook
from openpyxl.styles import Font, PatternFill, Alignment, Border, Side

def build(periods, traveler_name, out_path):
    wb = Workbook(); ws = wb.active; ws.title = "DTS Tracker"
    hf = Font(name="Arial", bold=True, size=12, color="FFFFFF"); hfill = PatternFill("solid", fgColor="1F3864")
    namefill = PatternFill("solid", fgColor="FFFF00"); ok = PatternFill("solid", fgColor="E2EFDA")
    tbd = PatternFill("solid", fgColor="FFF2CC")
    black = Font(name="Arial", size=11, color="000000"); bold = Font(name="Arial", size=11, bold=True)
    thin = Side(style="thin", color="BFBFBF"); border = Border(left=thin, right=thin, top=thin, bottom=thin)
    center = Alignment(horizontal="center", vertical="center", wrap_text=True)
    left = Alignment(horizontal="left", vertical="center", wrap_text=True)

    headers = ["NAME", "Pay Period", "Charge Date", "Stay Dates Covered", "# of Nights",
               "GTCC Charge ($)", "Rate per night ($)", "Posting Date", "FX Rate (PHP/USD)",
               "Cross-Border Fee ($)", "Receipt", "Notes"]
    ws.append(headers)
    for c in range(1, len(headers) + 1):
        cell = ws.cell(row=1, column=c); cell.font = hf; cell.fill = hfill; cell.alignment = center; cell.border = border

    r = 2
    for p in periods:
        nights = p.get("nights")
        nights = nights if isinstance(nights, int) else None
        ws.cell(row=r, column=2, value=f"Pay Period {p.get('pp', r-1)}")
        ws.cell(row=r, column=3, value=p.get("charge_date", ""))
        ws.cell(row=r, column=4, value=p.get("stay_dates", ""))
        ws.cell(row=r, column=5, value=nights if nights is not None else "")
        g = ws.cell(row=r, column=6, value=p.get("gtcc", "")); g.number_format = '#,##0.00'
        rate = ws.cell(row=r, column=7, value=f'=IF(ISNUMBER(E{r}),F{r}/E{r},"")'); rate.number_format = '#,##0.00'
        ws.cell(row=r, column=8, value=p.get("posting_date", ""))
        fx = p.get("fx", "")
        fxc = ws.cell(row=r, column=9, value=fx)
        if isinstance(fx, (int, float)): fxc.number_format = '0.000000'
        cbf = p.get("cross_border_fee", "")
        cbc = ws.cell(row=r, column=10, value=cbf)
        if isinstance(cbf, (int, float)): cbc.number_format = '#,##0.00'
        ws.cell(row=r, column=11, value=p.get("receipt_status", ""))
        ws.cell(row=r, column=12, value=p.get("notes", ""))
        for c in range(1, 13):
            cc = ws.cell(row=r, column=c); cc.border = border
            cc.alignment = center if c in (1,2,3,5,6,7,8,9,10) else left
            if c != 1: cc.font = black
        fill = ok if nights is not None else tbd
        ws.cell(row=r, column=4).fill = fill; ws.cell(row=r, column=5).fill = fill
        if str(p.get("receipt_status", "")).upper().startswith("MISS"):
            ws.cell(row=r, column=11).font = Font(name="Arial", size=11, bold=True, color="C00000")
        r += 1

    tr = r
    ws.cell(row=tr, column=2, value="TOTAL").font = bold
    t = ws.cell(row=tr, column=6, value=f"=SUM(F2:F{r-1})"); t.number_format = '#,##0.00'; t.font = bold
    ws.cell(row=tr, column=10, value=f"=SUM(J2:J{r-1})").number_format = '#,##0.00'
    for c in range(1, 13): ws.cell(row=tr, column=c).border = border

    ws.merge_cells(start_row=2, start_column=1, end_row=tr, end_column=1)
    nm = ws.cell(row=2, column=1, value=traveler_name); nm.fill = namefill
    nm.font = Font(name="Arial", bold=True, size=12, color="000000")
    nm.alignment = Alignment(horizontal="center", vertical="center", wrap_text=True)

    note = ("Nightly rate = GTCC charge / nights. Highlighted-green rows have a filed folio; "
            "yellow rows need their folio + night count from the hotel. Citibank data is captured "
            "from CitiManager. Cross-border fees (~1%) are tracked separately and are not part of "
            "the nightly rate.")
    ws.cell(row=tr + 2, column=2, value=note).font = Font(name="Arial", size=10, italic=True)

    widths = {"A":16,"B":13,"C":14,"D":20,"E":11,"F":15,"G":15,"H":13,"I":16,"J":16,"K":14,"L":58}
    for col, w in widths.items(): ws.column_dimensions[col].width = w
    ws.row_dimensions[1].height = 42; ws.freeze_panes = "B2"
    wb.save(out_path)
    print("wrote", out_path, "with", len(periods), "pay periods")

if __name__ == "__main__":
    if len(sys.argv) != 4:
        print(__doc__); sys.exit(1)
    with open(sys.argv[1]) as f: periods = json.load(f)
    build(periods, sys.argv[2], sys.argv[3])
```

### parse_folios.py
```python
#!/usr/bin/env python3
"""
Extract the audit-relevant facts from hotel folio PDFs.

Usage:
    python parse_folios.py <folder-with-folio-pdfs>

For each folio it prints arrival/departure, the number of nights (count of
"Room Charge" lines), the first/last room-charge date, and the credit-card
settlement amount. Use the settlement to match a folio to its GTCC charge:
for a foreign stay, card_amount * exchange_rate ~= folio settlement.

Requires pdfplumber (pip install pdfplumber).
"""
import sys, re, glob, json, os

def parse(path):
    import pdfplumber
    with pdfplumber.open(path) as pdf:
        t = "\n".join((pg.extract_text() or "") for pg in pdf.pages)
    arr = re.search(r"Arrival\s*:\s*([0-9/]+)", t)
    dep = re.search(r"Departure\s*:\s*([0-9/]+)", t)
    room = re.findall(r"([0-9]{2}/[0-9]{2}/[0-9]{2})\s+Room Charge", t)
    pays = re.findall(r"([0-9]{2}/[0-9]{2}/[0-9]{2})\s+(CC-\w+|Visa|MasterCard)\s+([0-9,]+\.[0-9]{2})", t)
    return {
        "file": os.path.basename(path),
        "arrival": arr.group(1) if arr else None,
        "departure": dep.group(1) if dep else None,
        "nights": len(room),
        "stay_dates": f"{room[0]} - {room[-1]}" if room else None,
        "settlement_php": [{"date": d, "type": ty, "amount": amt} for d, ty, amt in pays],
    }

if __name__ == "__main__":
    if len(sys.argv) != 2:
        print(__doc__); sys.exit(1)
    folder = sys.argv[1]
    results = [parse(p) for p in sorted(glob.glob(os.path.join(folder, "*.pdf")))]
    print(json.dumps(results, indent=2))
```
