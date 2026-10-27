# CDD Workspace Toolkit — setup

A standalone Google Apps Script web app that reads the CDD recruitment spreadsheet
(`[For Sharing] DATA_01_CDD_CANDIDATES_NEW_240824`) and adds a dashboard plus
Gmail, Drive, Calendar, Forms and external-API tools.

It **never writes to the CDD spreadsheet**. The existing "CDD Recruitment Database"
app keeps owning all edits, candidate IDs (`CANDIDATE_ID_COUNTER`) and duplicate checks.

## Files (`src/`)

| File | Purpose |
|---|---|
| `appsscript.json` | Time zone, V8, web-app settings, OAuth scopes |
| `Config.gs` | Sheet names, required headers, pipeline stages, Script Property keys |
| `Auth.gs` | Owner / viewer allow-list checks, `{ok, data, error}` responses |
| `SheetData.gs` | Batch reads, header-keyed rows, day/month date handling |
| `Model.gs` | Normalised snapshot of candidates, applications, job orders, contacts, ads |
| `Metrics.gs` | KPI, chart and job-performance figures |
| `Api.gs` | Dashboard, candidate and job-order endpoints |
| `Holidays.gs` | Singapore public holidays from the Nager.Date API (cached) |
| `Setup.gs` | `setup()`, settings, toolkit spreadsheet and activity log |
| `Digest.gs` | Gmail digest (MailApp) and trigger helpers |
| `DriveExport.gs` | CSV exports to a Drive folder |
| `CalendarSync.gs` | Batch / intake dates → "CDD Recruitment" calendar |
| `IntakeForm.gs` | Candidate interest Google Form and its submit trigger |
| `WebApp.gs` | `doGet()` and the HTML include helper |
| `Index.html`, `Styles.html`, `App.html` | Dashboard UI |
| `AccessDenied.html` | Page shown to people who are not on the allow-list |

## 1. Create the project

1. Go to <https://script.google.com> → **New project**. Rename it "CDD Workspace Toolkit".
2. **Project Settings** (gear) → tick **Show "appsscript.json" manifest file in editor**,
   and set the time zone to **(GMT+08:00) Singapore**.
3. In the editor, for each file in `src/` click **+ → Script** (for `.gs`) or **+ → HTML**
   (for `.html`), give it the same name without the extension, and paste the contents.
   Delete the default `Code.gs`, and paste `appsscript.json` over the existing manifest.
4. Click **Save**.

Alternative: `npm install -g @google/clasp`, `clasp login`, `clasp create --type standalone`,
copy `src/*` into the folder, `clasp push`.

## 2. Point it at the spreadsheet

The spreadsheet ID is already the default in `Config.gs`. To use another copy, set a Script
Property instead of editing code: **Project Settings → Script Properties → Add**
`CDD_SPREADSHEET_ID` = the ID between `/d/` and `/edit` in the sheet URL.

## 3. Run `setup()` once

Select `setup` in the function list → **Run**. Google asks you to authorise the scopes in
`appsscript.json`; review and allow. `setup()`:

- makes you the **owner** (`OWNER_EMAIL`),
- reads the CDD spreadsheet and logs the candidate / application / job-order counts,
- warns in the execution log about any missing header,
- creates the "CDD Toolkit Data" spreadsheet with an `ACTIVITY_LOG` tab.

## 4. Deploy the web app

**Deploy → New deployment → Web app**

| Setting | Value | Why |
|---|---|---|
| Execute as | **User accessing the web app** | Each person reads the sheet with their own Google access, and the app can see who they are. |
| Who has access | **Anyone with a Google account** (or **Only myself** while testing) | The allow-list in the app decides who actually gets in. |

Copy the `/exec` URL. Open it yourself first: you will see the dashboard and a **Tools** tab.

What this means in practice:

- A person can open the dashboard only if their email is the owner or is in **Tools → Who can
  open this dashboard**, **and** they have at least view access to the CDD spreadsheet.
- Every viewer is asked once to authorise the scopes, including Calendar and Forms, even though
  only the owner can use those tools (Apps Script asks for the whole project's scopes).
- Do **not** choose "Execute as: Me" with "Anyone": the app could then not identify personal
  Gmail users and the sheet would be read with your access for everyone.

## 5. Test it

1. Overview: compare a few numbers with the sheet (e.g. number of APPLICATIONS rows, jobs with
   Status = Active).
2. Candidates / Applications / Job Orders: search, filter, sort, page, open a detail panel.
3. Tools (owner only):
   - **Send now** → you receive the digest email.
   - **Export applications** → a CSV appears in "CDD Toolkit Exports" in your Drive.
   - **Sync now** → "CDD Recruitment" calendar is created. JOB_ORDER_BATCHES and INTAKES are
     currently empty, so it reports 0 events until dates are entered there.
   - **Create form** → open the share link, submit a test response, check the notification email
     and **View responses**. Delete the test response in the form afterwards.
   - **Turn on** the digest at your preferred hour.
4. Open the `/exec` URL in a private window with an account that is **not** on the list → you
   should see "No access".

## 6. Update later

Edit the files, **Save**, then **Deploy → Manage deployments → ✏️ → Version: New version →
Deploy**. The `/exec` URL stays the same. (`/dev` always runs the latest saved code, for the
owner only.)

## Script Properties

| Key | Set by | Meaning |
|---|---|---|
| `CDD_SPREADSHEET_ID` | `setup()` / you | CDD spreadsheet to read |
| `OWNER_EMAIL` | `setup()` | Owner; can use Tools |
| `ALLOWED_EMAILS` | Tools page | Viewer allow-list |
| `DIGEST_RECIPIENTS` | Tools page | Digest + form notification recipients (default: owner) |
| `DIGEST_HOUR` | Tools page | Digest hour (Singapore time) |
| `TOOLKIT_SPREADSHEET_ID`, `EXPORT_FOLDER_ID`, `CALENDAR_ID`, `FORM_ID` | the app | Items the toolkit created |

To hand the toolkit to someone else: they run `setup()` after you delete `OWNER_EMAIL`.

## How some numbers are worked out

- **Pipeline stage** groups each REF_DROPDOWNS "Application Status" into New, Contacted,
  Submitted, Interview, Offer, Hired or Closed. The rules are in `CONFIG.STAGE_RULES`.
- **Awaiting first contact**: stage New, Contact Status empty / "Not Contacted" / "KIV - To
  Contact", applied 2+ days ago (`CONFIG.AWAITING_CONTACT_DAYS`).
- **Dates.** Date Applied is a mix of script timestamps, `d/M/yyyy` text and hand-typed dates
  that the en_US sheet read month-first (e.g. `1/9/2026` meaning 1 September is stored as
  9 January). Script timestamps are trusted. A hand-typed ambiguous date is read as whichever
  of day/month or month/day is on or before the row's Date Updated and closest to it. The
  Data checks card counts these. Entering dates as `2026-09-01` removes the ambiguity.
- **Contact history** on a candidate is matched on full name, because CONTACT_HISTORY has
  no Candidate ID column.

## Troubleshooting

| Symptom | Cause / fix |
|---|---|
| "The toolkit is not set up yet" | Run `setup()` from the editor as the owner. |
| "Sign-in required" page | Deployment is not "Execute as: User accessing", or the browser is signed out. |
| "No access" | Add the email in Tools → Who can open this dashboard. |
| "You do not have permission to access the requested document" | The viewer has no access to the CDD spreadsheet. Share it with them (Viewer). |
| "Authorization is required" / "ScriptError" in the UI | A new scope was added: open the `/exec` URL again and approve, or re-run `setup()` as owner. |
| A figure is 0 and a yellow banner lists missing columns | A header was renamed in the sheet. Rename it back, or update `CONFIG.REQUIRED_HEADERS` and the field names in `Model.gs`. |
| "Sheet "X" not found" | A tab was renamed. Update `CONFIG.SHEETS`. |
| CSV export fails with a Drive permission error | The `drive.file` scope only covers files the toolkit created. If the export folder was replaced by hand, clear `EXPORT_FOLDER_ID` and export again. |
| Digest says quota exhausted | Consumer Gmail accounts can send about 100 MailApp emails/day. Reduce recipients or wait. |
| Daily digest did not arrive | Tools shows Status: Off → turn it on. Check **Triggers** (clock icon) and **Executions** for errors. |
| Form notifications stopped | Tools shows Notifications: off → click Create form again (it reuses the form and re-installs the trigger). |
| Holiday card shows "unavailable" | The Nager.Date API was unreachable. The rest of the dashboard is unaffected; it retries on the next load. |
| Dashboard slow / "Exceeded maximum execution time" | Each load reads 7 tabs once. With many thousands of rows, archive old rows or add caching. |
| `google.script.run` "is not a function" | The server function was renamed or ends in `_` (private). Client calls must match a public function in `Api.gs` etc. |
