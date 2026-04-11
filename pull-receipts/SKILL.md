---
name: pull-receipts
description: "Pull gear receipts from ELMS for Marines. Give it a roster spreadsheet to pull for a whole platoon, or name Marines in chat for a quick pull. Handles CAC login, downloads Individual Issue PDFs, and names them by Marine."
user_invocable: true
---

# ELMS IIF Receipt Puller

Automates pulling Individual Issue receipts from ELMS (Enterprise Logistics Management System) for gear inspections. Works in two modes:

- **Batch mode**: Process an entire roster from a spreadsheet
- **Quick pull mode**: Pull receipts for specific Marines named in conversation

## Prerequisites

- User MUST have their CAC inserted and ready
- Chrome MUST be available via Claude in Chrome MCP

## Guided Onboarding

Not every user will know the terminology or setup. When the skill is invoked, assess the user's familiarity and guide them as needed. If the user seems unsure about anything, proactively explain — don't wait for them to ask.

**Common things new users may not know:**

- **What is ELMS?** — Enterprise Logistics Management System. It's the website where the Marine Corps tracks individually issued gear (CIF/IIF items). URL: `https://member-prod.elms.dla.mil/`
- **What is an IIF/CIF receipt?** — The "Individual Returnable Items" report. It lists every piece of gear issued to a Marine — flak jacket, kevlar, gas mask, etc. — with costs and serial numbers. Leaders pull these before gear inspections to know what each Marine should have.
- **What is an EDIPI?** — Electronic Data Interchange Personal Identifier. It's the 10-digit number on the back of every CAC card. If the user doesn't know a Marine's EDIPI, they can find it on the Marine's CAC, in MOL, or ask their admin.
- **What is a CAC?** — Common Access Card. The military ID card. It must be inserted in a card reader to log into ELMS. Claude cannot enter the PIN — the user must do that part.
- **What format does the spreadsheet need to be?** — Any format works: CSV, XLSX, XLS, TSV. It does NOT need to be a clean two-column file. The skill will scan the headers/columns and automatically find the last name and EDIPI columns. A recall roster, alpha roster, or any spreadsheet with those two pieces of data will work. If a first name column exists, it will be used for file naming.
- **Where do I get the EDIPI list for my platoon?** — Ask your platoon sergeant, check MOL, or collect them from CAC cards. Most units maintain recall rosters or alpha rosters that already have EDIPIs.

**If the user says something vague like "pull gear receipts" or "I need to do a CIF check":**
1. Explain what the skill does in plain terms
2. Ask if they want to pull for one Marine, a few Marines, or a whole roster
3. Guide them to provide what's needed based on their answer

**If the user doesn't have a spreadsheet and wants batch mode:**
1. Offer to create one for them — ask them to list the names and EDIPIs
2. Generate the CSV file and save it, then proceed with batch mode

## Determining the Mode

When this skill is invoked, determine which mode to use:

### Batch Mode
Triggered when the user provides or references a spreadsheet/CSV file, says things like "pull receipts for my platoon", "run the roster", or provides a file path.

Ask the user for:
1. **Spreadsheet file path** — path to the file (CSV, XLSX, XLS, TSV all supported)
2. **Output folder** — where to save the PDFs (default: `~/Downloads/IIF_Receipts/`)

#### Smart Column Detection

The spreadsheet does NOT need to be a clean two-column file. It can be a recall roster, alpha roster, or any spreadsheet with many columns. The skill will automatically detect the right columns:

1. Read the file and examine the header row (first row)
2. Find the **last name column** by matching headers case-insensitively against: `LastName`, `Last Name`, `Last`, `LNAME`, `Surname`, `Name` (if only one name column exists)
3. Find the **EDIPI column** by matching against: `EDIPI`, `DoD ID`, `DOD ID`, `DODID`, `ID Number`, `CAC ID`
4. Optionally find the **first name column** by matching against: `FirstName`, `First Name`, `First`, `FNAME`
5. Optionally find the **rank column** by matching against: `Rank`, `Grade`, `Pay Grade`

If the columns cannot be auto-detected:
- Show the user the first few rows of the spreadsheet
- Ask them to identify which columns contain the last name and EDIPI
- Ask if there's a first name column they want used for file naming

#### File Naming

- If **rank** is available: `{Rank} {LastName} IIF Receipt.pdf` (e.g., `Cpl Smith IIF Receipt.pdf`)
- If only **last name** is available: `{LastName} IIF Receipt.pdf`
- If duplicate names exist (same rank + last name, or same last name with no rank), append first name to disambiguate: `{Rank} {LastName}, {FirstName} IIF Receipt.pdf`
- If still duplicate after adding first name, append a number: `{Name} IIF Receipt (2).pdf`

### Quick Pull Mode
Triggered when the user names specific Marines, says things like "pull the CIF for Smith and Jones", "get me the IIF receipt for Corporal Garcia", or similar.

For each Marine mentioned:
1. Ask: **"I need the last name and EDIPI for each Marine. What are they?"**
2. Accept input in any reasonable format — the user might say "Smith 1234567890" or "Smith, 1234567890" or list multiple at once
3. Parse the names and EDIPIs into a list
4. Ask for output folder (default: `~/Downloads/IIF_Receipts/`)
5. Proceed with the same download workflow as batch mode

If the user already provided the last name and EDIPI in their initial message (e.g., "pull CIF for Smith 1234567890"), do NOT ask again — just confirm and proceed.

## Workflow

### Phase 1 — Parse the Roster (MUST complete before touching the browser)

> **HARD GATE: You MUST read and parse the spreadsheet (or collect quick-pull info) BEFORE navigating to ELMS. Do NOT open a browser, do NOT navigate to any URL until you have a concrete list of real Marines parsed from real data. If you do not have a parsed list, STOP and get one.**

1. Create the output folder if it doesn't exist
2. If batch mode:
   - Read the spreadsheet file (CSV, XLSX, XLS, or TSV)
   - Run Smart Column Detection (see above) to identify LastName, EDIPI, and optionally FirstName and Rank columns
   - Parse all rows into a list of Marines
   - Show the user what was detected: "Found {total} Marines. Detected columns — Last Name: '{col}', EDIPI: '{col}', First Name: '{col}', Rank: '{col}'"
3. If quick pull mode: use the names/EDIPIs collected from the user

**Confirmation checkpoint:** Print the first 5 Marines from the parsed list (rank, name, EDIPI) and ask: **"Here are the first 5 of {total} Marines. Does this look right? Proceed?"** Do NOT continue until the user confirms.

4. **Duration estimate:** Calculate ~15 seconds per Marine. Tell the user: **"This will take approximately {minutes} minutes for {total} Marines."** If the batch exceeds 50 Marines, warn that ELMS session timeouts (~15 min) may require re-authentication mid-batch.

### Phase 2 — Login

1. Use `tabs_context_mcp` (with `createIfEmpty: true`) to get or create a tab group
2. Check if ELMS is already open in an existing tab (URL contains `member-prod.elms.dla.mil` and title contains "Member Access - Home"). If so, skip to step 8.
3. Create a new tab or use an existing one
4. Navigate to `https://member-prod.elms.dla.mil/`
5. Tell the user: **"ELMS is loading. Please select your certificate and enter your CAC PIN when prompted."**
6. Poll the page every 3 seconds (up to 60 seconds) waiting for the page title to contain "Member Access - Home" — this confirms login succeeded
7. If login is not detected after 60 seconds, tell the user and ask them to confirm when they're in
8. Once logged in, the page shows "Reports" and "Appointments" sections
9. Check if the Reports section is already expanded (look for the EDIPI/Last Name input fields already visible). Only click the "Reports" header bar if the section is collapsed. Clicking an already-expanded section will collapse it.
10. Wait 1 second, then confirm the search form is visible

### Phase 3 — Per-Marine Loop

For each `(LastName, EDIPI)` pair:

**Step 1 — Clear previous search:**
- Click the "Reset" button on the ELMS search form
- Wait 1 second for the form to clear

**Step 2 — Enter search criteria:**
- Click the "Member's EDIPI" input field and type the EDIPI value
- Click the "Member's Last Name" input field and type the LastName value

**Step 3 — Search:**
- Click the "Search" button
- Wait 2 seconds for results to load
- Verify the results grid shows at least 1 result. If 0 results, log a warning and skip to the next Marine

**Step 4 — Select the member:**
- Click the checkbox next to the first member row in the results grid (located at approximately x=34 in the checkbox column of the first data row)
- Wait 1 second
- Verify the "Individual Issue" tab becomes active/clickable

**Step 5 — Download the receipt:**
- Note the current timestamp BEFORE clicking download
- Click the "Individual Issue" tab/button
- Wait 5 seconds for the PDF to generate and download
- A new tab may briefly open and close — this is expected

**Step 6 — Rename and move the PDF:**
- Check `~/Downloads/` for the most recently modified file matching `DWForm05I*.pdf`
- Verify the file's modification time is AFTER the timestamp noted in Step 5. If not, the download may have failed — wait 3 more seconds and check again. If still no new file, log a warning and skip this Marine.
- Name the file using the File Naming rules (see Batch Mode section above):
  - With rank: `{Rank} {LastName} IIF Receipt.pdf`
  - Last name only: `{LastName} IIF Receipt.pdf`
  - If duplicate names, add first name to disambiguate: `{Rank} {LastName}, {FirstName} IIF Receipt.pdf`
- Move/rename it to `{output_folder}/{filename}`
- If a file with that name already exists in the output folder, append a number: `{filename} (2).pdf`
- After the rename, delete any remaining `DWForm05I*.pdf` files from `~/Downloads/` to prevent Chrome from appending `(1)`, `(2)`, etc. on the next download

**Step 7 — Close any PDF tabs:**
- Check `tabs_context_mcp` for any tabs with "DWForm05I" in the title
- Close them to keep the workspace clean

**Step 8 — Verify and continue:**
- Confirm the renamed PDF exists in the output folder
- Log: "Downloaded receipt for {LastName} ({current}/{total})"
- Continue to next Marine

### Session Timeout Handling

The ELMS session times out after ~15 minutes of inactivity.

- Before each Marine, check if the ELMS page still shows the search form (look for the "Search Criteria" section or the EDIPI input field)
- If the page shows a logout/session expired message, or the URL contains "SessionExpired" or "LogOff":
  1. Stop the loop
  2. Tell the user: **"ELMS session timed out after processing {N} of {total} Marines. Please enter your CAC PIN to log back in."**
  3. Navigate back to `https://member-prod.elms.dla.mil/`
  4. Wait for the user to complete CAC authentication (same polling as Phase 2)
  5. Expand the Reports section again
  6. Resume the loop from where it left off (do NOT restart from the beginning)

### Phase 4 — Completion

After all Marines are processed:
1. List all successfully downloaded PDFs
2. List any Marines that were skipped (with reason)
3. Report total: **"Downloaded {success} of {total} IIF receipts to {output_folder}"**

## Important Notes

- "CIF receipt", "IIF receipt", "individual issue", and "gear receipt" all refer to the same thing — the Individual Returnable Items report from ELMS
- The EDIPI is a 10-digit number — validate format if provided
- ELMS is a government system on a .mil domain — expect slower response times than commercial sites
- Each PDF download overwrites the same filename (`DWForm05I.pdf`), so rename BEFORE starting the next Marine
- The "Individual Issue" button only becomes clickable after selecting a member checkbox
- Chrome may append `(1)`, `(2)`, etc. to the filename if the original still exists — always look for the most recent `DWForm05I*.pdf`
- The CAC certificate selection and PIN entry are browser-native dialogs that Claude cannot interact with — the user must handle these manually
- NEVER fabricate or hallucinate Marine names, EDIPIs, or roster data. Every name and EDIPI must come from the user's spreadsheet or direct input. If you don't have real data, STOP and ask for it.

## Element Locations (ELMS Member Access - Reports page)

These are approximate coordinates and selectors observed during development:

- **Reports header**: Clickable bar at top of page to expand/collapse the reports section
- **EDIPI input field**: Text input after "* Member's EDIPI" label
- **Last Name input field**: Text input after "* Member's Last Name" label
- **Search button**: Button with magnifying glass icon labeled "Search"
- **Reset button**: Button with refresh icon labeled "Reset"
- **Member checkbox**: Checkbox in first column of results grid row
- **Individual Issue tab**: Tab button in the Member Reports tab bar, only active when a member is selected
- **Grid results area**: Table below the tab bar showing Member, Logistics Program, UIC columns
