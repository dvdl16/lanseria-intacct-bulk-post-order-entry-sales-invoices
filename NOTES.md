

## Trial and Error


Here’s the play-by-play of what we attempted, what failed (and why), and what *actually* worked.

#### What we tried

##### 1) Read/inspect the project files

* **Worked:** We successfully read and concatenated **6 files** in the repo:

  * `Bulk_Order_Entry_Invoicing.xml`, `README.md`, `component.html`, `modal.html`, and two screenshots under `docs/`.
* **Outcome:** Enough context to understand the existing custom object + UI approach and where to integrate a POC.

---

##### 2) Original idea: do it “inside the report page”

* **Didn’t work (by platform design):** Sage Intacct doesn’t allow injecting custom JS/UI into standard report pages.
* **Outcome:** We pivoted to recreating the workflow in a custom object page.

---

##### 3) Think of approaches, choose Option 3 POC: copy/paste report output → create batch + link invoices
| Approach                          | User Experience | Recommendation                                      |
|-----------------------------------|----------------|----------------------------------------------------|
| 1. Replicate Report Filters       | ✨ Excellent    | Best, if you have the time/budget.                 |
| 2. CSV Import                     | 👍 Good         | Recommended as the best balance.                   |
| 3. Copy/Paste Invoice Numbers     | 🤔 Okay         | A simpler, but more error-prone starting point.    |
``


**Goal:** Paste AR report rows, parse invoice identifiers, create a batch, link invoices.

**3a) Build UI on the **New** page (HTML component + Script component)**

* **Worked:** UI components were added and script executed.
* **Didn’t work initially:** Button click did nothing due to a JS syntax error:

  * **`Uncaught SyntaxError: unterminated regular expression literal`**
* **Fix that worked:** Replaced fragile regex literal splitting (e.g. `/\r?\n/`) with safer string splitting logic.
* **Outcome:** Script ran and started sending API payloads.

**3b) Create batch + link invoices in a single `<create>` payload**

* **Didn’t work:** Intacct rejected multiple relationship values:

  * **`Field 'RSODOCUMENT' has more than one value`**
* We tried:

  * Multiple `<RSODOCUMENT>` nodes → rejected
  * Nesting children under one `<RSODOCUMENT>` container → still rejected
  * Removing whitespace/newlines to avoid parser quirks → still rejected
  * Renaming relationship field (e.g. to `OE_INVOICES`) → same fundamental error
* **Conclusion:** For this object/relationship, Intacct wouldn’t accept “create parent + attach many children” in one call.

**3c) Pivot workflow: parent first, then attach invoices on the **Edit** page**

* **Worked (conceptually):** This is a more “Intacct-native” workflow.
* **But we hit regressions:**

  * **`'' string literal contains an unescaped line break`** → fixed by correcting newline handling in JS strings
  * Batch key from URL was unreliable → **worked better** using merge field:

    * `{!SALESINVOICEBATCH.id!}`

**3d) Attempt to update relationship via API from UI script**

* **Didn’t work:** Updating the batch with nested `<OE_INVOICES><sodocument>...</sodocument></OE_INVOICES>` caused:

  * **`Object with id  not found`** (blank id)
* We isolated it further with **Postman**:

  * Even a minimal update using `<DOCNO>AR-INV30878</DOCNO>` failed with:

    * **`Unable to update custom object. Object with id not found`**
* **Key insight (what worked as a finding):**

  * You discovered the *correct linking method* is **not** “update the batch relationship list”.
  * The reliable link is to update each invoice (`SODOCUMENT`) and set:

    * `<RSALESINVOICEBATCH>10070</RSALESINVOICEBATCH>`
    * using `<RECORDNO>` for the invoice.
* **Worked (as an approach discovery):** You added **Record number** to the report paste output, so the script could parse `RECORDNO`.

**3e) Final blocker: UI API limitations**

* **Didn’t work:** In the **UI scripting API**, you can’t call generic `<update>`; only certain helpers like `update_sotransaction`, and those **don’t let you set `RSALESINVOICEBATCH`** the way the standard API does.
* **Outcome:** “Correct API approach” was identified, but **not executable** from the UI API context.

---

##### 4) Pivot again: simulate the UI picker (AutoSuggest / selector JS)

**Goal:** If the UI won’t let us update via API, mimic what a user does when selecting invoices in the multi-select control.

* **Worked:** Parsing the pasted table and producing the right list of invoices.
* **Worked:** Calling `rbf_selectObject` for each invoice (confirmed by logs).
* **Didn’t work:** UI did not visually add the invoices to the list even though calls ran.

We then added troubleshooting and UI triggers:

* Dispatching `change` event after adding → **didn’t refresh**
* Attempted `inputElement.meta.getDataFromHTML()` → **not present**
* Added a more realistic lifecycle step:

  * **Verified elements exist** (`txtoe_invoices`, `rtable_txtoe_invoices`)
  * **Triggered focus** to initialize `AutoSuggestControl(...)`
  * Re-ran `rbf_selectObject` loop
  * Fired `change` again

*Finally*, we changed the `inputName` for the `rbf_selectObject` to 'oe_invoices'



## Example paste data from "AR Invoices"

```

	Customer ID	Document ID	Record number	Date	GL posting date	State	Subtotal	Total
	4012	Sales Invoice-AR-INV30878	89061	07/12/2025	07/12/2025	Pending	11,400.00000	13,110.00000
	4019	Sales Invoice-AR-INV30879	89062	07/12/2025	07/12/2025	Pending	8,702.00000	10,007.30000
	FDCASHSALE	Sales Invoice-AR-INV30877	89060	07/12/2025	07/12/2025	Pending	5,890.00000	6,773.50000
	FDCASHSALE	Sales Invoice Foreign-AR-INV30876	89059	07/12/2025	07/12/2025	Pending	11,400.00000	11,400.00000
	4500	Sales Invoice-AR-INV30875	89058	05/12/2025	05/12/2025	Pending	7,203.00000	8,283.45000
	4500	Sales Invoice-AR-INV30880	89063	07/12/2025	07/12/2025	Pending	6,800,518.00000	7,820,595.70000
Sum Total							6,845,113.00000	7,870,169.95000

```