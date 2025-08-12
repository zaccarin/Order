Here’s a clean, GitHub‑ready **README.md** you can drop into the repo for your current Purchase‑Order web app. It matches the exact code you last shared (PO‑photo check, PO Date saved as date‑only, Target Date present, and duplicate guard on **PO + Style + SKU + Material**, no cache).

---

# Purchase‑Order Web App (Google Apps Script)

A Google Apps Script + HTML web app to capture Purchase Orders into Google Sheets with client/style autocomplete, SKU/Item lookups, duplicate protection, PO‑photo linkage via Google Forms, and lightweight logging.

---

## Features

* **Fast PO entry** with multi‑style, multi‑product rows
* **Autocomplete**

  * Clients & Style IDs from `O2D_Styles`
  * SKU & Item from `MasterItems` (+ optional image preview)
* **Data integrity**

  * Duplicate guard on **PO Number + Style ID + SKU + Material**
  * Requires **PO photo** uploaded via Google Form (checked before submit)
* **Dates**

  * **PO Date** stored as **date‑only (`dd/MM/yyyy`)**
  * **Target Date** required, date‑only
* **Ops**

  * XLOOKUP auto‑fills photo URL in `Indent Details` (col K)
  * Temp save / restore draft
  * Execution & error logging (+ optional email summary)
  * Time‑based triggers to maintain formulas / summaries

---

## Tech Stack

* **Google Apps Script** (server): `Code.gs`
* **HTML/CSS/JS** (client): `Index.html`
* **Google Sheets** (data store):

  * `Indent Details` (main)
  * `Form_responses_1` (Google Form responses for PO photo)
  * `O2D_Styles` (Client ↔ Style lookup)
  * `MasterItems` (SKU/Item/UOM lookup)
  * `Temp Form`, `Helper`, `Execution Log`, `Error Log`, `New_Entries`

---

## Sheet Setup & Schema

### 1) `Indent Details` (destination)

| Col | Field         | Source                                  |
| --- | ------------- | --------------------------------------- |
| A   | PO Date       | `dd/MM/yyyy`                            |
| B   | PO Number     | Text, stored with leading `'`           |
| C   | Style ID      | From UI                                 |
| D   | Client        | From UI                                 |
| E   | Supplier      | From UI                                 |
| F   | SKU Code      | From MasterItems / UI                   |
| G   | Material Name | From MasterItems / UI                   |
| H   | Quantity      | Number                                  |
| I   | UOM           | From MasterItems / UI                   |
| J   | Target Date   | `dd/MM/yyyy`                            |
| K   | Photo URL     | Set by `extendXLOOKUP()`                |
| L   | Timestamp     | `dd/MM/yyyy HH:mm:ss` (submission time) |

> Column K is maintained by a formula inserted by `extendXLOOKUP()`.

### 2) `Form_responses_1` (Google Form)

* **B**: PO Number
* **C**: Photo URL

> The app checks this sheet to confirm a PO photo exists before accepting a submission.

### 3) `O2D_Styles`

* **A**: Client
* **B**: Style ID

### 4) `MasterItems`

* Must have headers including: `Client`, `SKU Code`, `Item`, `UOM`, `Image URL` (optional)

### 5) Other sheets

* `Temp Form` – draft storage
* `Helper` – uses `A1` as a progress marker for `extendXLOOKUP`
* `Execution Log` – `[Timestamp, Level, Message]`
* `Error Log` – `[Timestamp, Level, Message]` *(see “Notes” re: email summary)*
* `New_Entries` – tracks new clients encountered during submit

---

## Data Flow

1. **User opens web app** → `doGet()` serves `Index.html`.
2. **On load**, client:

   * Restores draft via `getLatestTempForm()`
   * Loads master items via `getMasterItems()`
3. **Autocomplete**

   * Client → server: `getClients(query)` → returns up to 10 matches from `O2D_Styles!A`
   * Client → server: `getStyleIDs(client, query)` → returns up to 10 matches from `O2D_Styles!B` filtered by client
   * SKU/Item suggestions are client‑aware, with allow‑list override (`ALWAYS`)
4. **Submit**

   * Client builds payload `{ poNumber, poDate, client, supplier, targetDate, styles(JSON) }`
   * Server `submitIndent(form)`:

     * Validates required fields
     * **Blocks** if no PO photo found in `Form_responses_1` (`isPhotoUploaded`)
     * Parses & formats dates (PO Date as **date‑only**)
     * For each product row: checks **duplicates** with `isDuplicatePOStyleSKUMat`
     * Appends rows to `Indent Details`
     * Logs execution
5. **Post‑submit ops**

   * `extendXLOOKUP()` (on open + hourly trigger) fills Photo URL (col K) via XLOOKUP from `Form_responses_1`.

---

## Server Functions (Apps Script)

### Web entry

* `doGet()` → serves UI with title “PO Submission Form”

### Autocomplete & lookups

* `getClients(query)` → distinct client names from `O2D_Styles!A`
* `getStyleIDs(client, query)` → style IDs from `O2D_Styles!(A:B)` filtered by client
* `getMasterItems()` → returns all rows from `MasterItems` as objects (keyed by header)

### Submission & validation

* `submitIndent(form)`

  * **Validates** fields (PO Number, Client, Supplier, PO Date, Target Date, at least one product row)
  * `isPhotoUploaded(poNumber)` ensures a photo exists in `Form_responses_1` (B: PO, C: URL)
  * **Duplicate guard**: `isDuplicatePOStyleSKUMat(poNum, styleId, sku, material)`

    * Pure sheet scan (no cache) across `Indent Details`:

      * PO → **B**, Style → **C**, SKU → **F**, Material → **G**
    * Normalizes with `norm(s)` (strips leading `'`, trims, uppercases)
  * Writes rows as per schema above
  * `logExecution(msg)` / `logError(msg)`

### Photo URL backfill

* `extendXLOOKUP()`

  * For each new row in `Indent Details`, sets **K** to:

    * `XLOOKUP(PO, 'Form_responses_1'!B:B, 'Form_responses_1'!C:C, "No photo uploaded", 0, -1)`
    * Tries both raw PO (with leading `'`) and stripped PO
  * Uses `Helper!A1` to remember last processed row

### Draft save/restore

* `saveTempForm(form)` → appends draft to `Temp Form`
* `getLatestTempForm()` → returns latest draft object

### Ops: triggers & email

* `onOpen()` → calls `extendXLOOKUP()`
* `createTriggers()` → hourly `extendXLOOKUP` + every‑3‑hours `sendErrorSummary`
* `sendErrorSummary()` → emails a simple error digest and clears `Error Log` rows (A2\:C)

### Utilities

* `norm(s)` → normalize string for comparisons
* `convertAncientDate(input)` → helper for serial→date conversion (rarely used)
* `logNewClient(client)`, `logExecution(msg)`, `logError(msg)`

---

## Client (Index.html)

* **Layout:** responsive container, header grid for PO meta, per‑style table with SKU/Item rows

* **Inputs (header):**

  * `poNumber` (text)
  * `poDate` (**type="date"**; saved as `dd/MM/yyyy`)
  * `client` (autocomplete)
  * `supplier` (text)
  * `targetDate` (**type="date"**; required)

* **Per‑style table:**

  * `styleIdInput` (autocomplete from server)
  * `skuInput` & `itemInput` (suggest from MasterItems; optional image preview)
  * `quantity`, `uom`

* **Actions:**

  * Add/Remove style & product rows
  * Save draft (temp save)
  * Submit PO
  * Open Google Form (upload PO photo)

* **API calls (google.script.run):**

  * `getLatestTempForm()` → hydrate draft
  * `getMasterItems()` → cache master items in browser
  * `getClients(query)` / `getStyleIDs(client, query)`
  * `saveTempForm(draft)`
  * `submitIndent(payload)`

**Payload (submit):**

```json
{
  "poNumber": "FTPL/0002",
  "poDate": "2025-07-04",
  "client": "Client A",
  "supplier": "Supplier X",
  "targetDate": "2025-07-18",
  "styles": "[{ \"styleId\": \"STY-001\", \"products\": [{\"productCode\": \"SKU-123\", \"materialName\": \"Cotton\", \"quantity\": \"100\", \"uom\": \"M\"}]}]"
}
```

---

## Deployment

1. **Sheet prep:** Ensure all sheets above exist and headers match.
2. **Form link:** Confirm your Google Form writes to `Form_responses_1` with **B=PO Number**, **C=Photo URL**, and update the **Upload Photo** button link in `Index.html` if needed.
3. **Apps Script**

   * Set script time zone (File → Project properties) to IST (or desired).
   * Run `onOpen()` once to initialize formulas.
   * Run `createTriggers()` to install time‑based triggers.
4. **Publish Web App**

   * Deploy → **New deployment** → type **Web app**
   * Execute as **Me**, Who has access **Anyone with the link** (or per your policy)
   * Copy the URL

---

## Testing Checklist

* [ ] Client & Style‑ID autocompletes show expected values
* [ ] SKU/Item suggestions filter by client (except allow‑listed)
* [ ] **PO photo workflow**: Photo is uploaded to the Google Form first → then app allows submit
* [ ] Duplicate guard blocks on **PO + Style + SKU + Material**
* [ ] **PO Date** stored as **dd/MM/yyyy** (no time) in `Indent Details!A`
* [ ] **Target Date** stored as \*\*dd/MM/yyyy`in`Indent Details!J\`
* [ ] Column **K** fills with photo URL (or “No photo uploaded”) after `extendXLOOKUP`
* [ ] `Execution Log` and `Error Log` record events

---

## Notes & Gotchas

* **Duplicate check scope:** The guard intentionally **does not** use any time‑window cache; it purely scans what’s already in `Indent Details`.
* **Error summary email:** `sendErrorSummary()` currently summarizes `Error Log` as if the columns are `[Timestamp, PO Number, Error]`.
  However, `logError(msg)` appends `[Timestamp, "ERROR", msg]`.

  * If you want PO Number in the email, either:

    * Change `logError()` to write `[Timestamp, poNumber, msg]`, **or**
    * Adjust `sendErrorSummary()` to use the actual column layout.
* **Leading apostrophes:** PO Numbers are written with a leading `'` to force text. `extendXLOOKUP()` handles both raw and stripped versions.
* **ALWAYS client list:** In the client‑side suggestions, certain clients (`['reliance','first cry','threads']`) bypass strict client filtering for convenience. Adjust or remove as needed.

---

## Security & Permissions

This project uses standard Apps Script scopes:

* Spreadsheet read/write
* Mail (for error summaries)
* Triggers (time‑based)

Keep the deployment access aligned with your organization’s policy.

---

## Performance Tips

* Keep `MasterItems` tidy; large lists still work but increase client‑side filtering time.
* Avoid volatile sheet formulas in `Indent Details`.
* If `Form_responses_1` grows very large, consider indexing or moving to a lookup range (named range) for faster XLOOKUP.

---

## Roadmap (optional)

* Role‑based access (view/submit per team)
* Server‑side SKU search with pagination
* Attachment upload to Drive + link in `Indent Details`
* Structured error objects with PO Number for better summaries

---

## License

Internal use. © Your Organization.

---

If you want this saved as a real `README.md` file with a minimal folder tree and sample screenshots placeholders, say the word and I’ll generate the file(s) for download.
