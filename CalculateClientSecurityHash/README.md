# Calculate Client Security Hash — UiPath RPA Process

Automation of the **"Calculate Client Security Hash" (WI5)** business process in **ACME System 1**, implemented according to the Process Design Document (PDD) Section 2.3 (To-Be process).

The robot authenticates into ACME System 1, collects all open `WI5` work items across paginated tables, retrieves each client's details, calculates the SHA1 security hash on `sha1-online.com`, and updates each work item with the hash comment and status `Completed`.
---

## Tech Stack & Prerequisites

| Component | Details |
| :--- | :--- |
| **UiPath Studio / Robot** | `2026.x` (developed on Robot `26.0.x`) |
| **Project Type & Language** | Standard Process / Windows / VB.NET|
| **Dependencies** | `UiPath.System.Activities` `[26.2.6]`, `UiPath.UIAutomation.Activities` `[26.10.5]` |
| **Browser** | Google Chrome with the UiPath Browser Automation extension enabled |
| **Target Applications** | ACME System 1 (`https://acme-test.uipath.com`), SHA1 Online (`http://www.sha1-online.com`) |

---

## Project Architecture

```text
CalculateClientSecurityHash/
├── Main.xaml                      # End-to-end orchestration, configuration, loop & summary
├── Main.xaml.json                 # Entry point metadata
├── workflows/
│   ├── Login.xaml                 # Open ACME System 1 and authenticate (PDD 1.1–1.2)
│   ├── GetWorkItems.xaml          # Extract all paginated Work Items & filter WI5 / Open (PDD 1.3–1.4)
│   ├── GetClientDetails.xaml      # Retrieve Client ID, Name, and Country (PDD 1.5.A)
│   ├── CalculateHash.xaml         # Compute SHA1 hash on sha1-online.com (PDD 1.5.B–C)
│   └── UpdateWorkItem.xaml        # Add hash comment, set status Completed & confirm alert (PDD 1.5.E–F)
└── project.json                   # Project manifest & dependencies
```

### Workflow Arguments

| Workflow | Arguments |
| :--- | :--- |
| `Login.xaml` | `in_AcmeLoginUrl`, `in_Email`, `in_Password` |
| `GetWorkItems.xaml` | `in_AcmeBaseUrl`, `out_dtWI5Items` |
| `GetClientDetails.xaml` | `in_AcmeBaseUrl`, `in_WIID`, `out_ClientId`, `out_ClientName`, `out_ClientCountry`|
| `CalculateHash.xaml` | `in_Sha1Url`, `in_TextToHash`, `out_SecurityHash` |
| `UpdateWorkItem.xaml` | `in_AcmeBaseUrl`, `in_WIID`, `in_SecurityHash` |

---

## Process Flow (`Main.xaml`)

1. **Log Start:** Records process start in the execution log (`Info`).
2. **Authenticate (`Login.xaml`):** Opens ACME System 1 and logs in using configured credentials.
3. **Collect & Filter (`GetWorkItems.xaml`):** Navigates to Work Items, extracts the full table across all pages (`114` total items), and filters for `Type = "WI5"` and `Status = "Open"`.
4. **Business Rule Check:** If `0` open `WI5` items are found, logs a warning (`Warn`) and terminates gracefully.
5. **Process Each `WI5` Item:**
   * **`GetClientDetails.xaml`:** Opens `/work-items/{WIID}` and parses `Client ID`, `Client Name`, and `Client Country`.
   * **Format Hash Input:** Constructs `ClientID-ClientName-ClientCountry`.
   * **`CalculateHash.xaml`:** Generates the 40-character SHA1 digest on `sha1-online.com`.
   * **`UpdateWorkItem.xaml`:** Navigates directly to `/work-items/update/{WIID}`, writes the hash comment, sets status to `Completed`, submits, and confirms the native dialog.
6. **Log Summary:** Outputs `Process completed. <X> of <Y> WI5 work items processed.`

---

## Key Engineering & QA Design Decisions

* **Modular Sub-Workflows:** Each business step is isolated with explicit `In`/`Out` arguments, enabling unit testing (`Run File`) before E2E integration.
* **Deterministic Pagination Extraction:** Extracts all pages of the Work Items table upfront so updating records does not shift pagination indices mid-run.
* **Direct URL Navigation:** Navigates straight to `/work-items/{WIID}` and `/work-items/update/{WIID}` in the same tab, bypassing list search and separate pop-up windows.
* **Dynamic Selectors:** Uses wildcard window titles (`ACME System 1*`) with zero hardcoded `WIID` or hash values inside selectors.
* **Regex Parsing & Cache Prevention:** Extracts client fields and the SHA1 digest (`\b[0-9a-f]{40}\b`) via regular expressions, reloading `sha1-online.com` on each iteration to prevent stale reads.
* **Event-Driven Synchronization:** Uses `Check App State` (no fixed delays) and handles native JavaScript confirmation dialogs via Hardware Events.

---

## How to Run

1. Open `CalculateClientSecurityHash/project.json` in **UiPath Studio**.
2. In `Main.xaml` (**Data Manager → Variables**), set `strEmail` and `strPassword` for your ACME System 1 account.
3. In ACME System 1, go to **User Options → Reset Test Data**, then **Log Out**.
4. Close any open ACME / SHA1 Online tabs, keep Google Chrome open, and run `Main.xaml`.

---

## Verification & Test Results

| Test Case | Outcome |
| :--- | :--- |
| **End-to-End Run (Post-Reset)** | `14 of 14` open `WI5` work items processed and updated to `Completed` in ~6 minutes (~25s/item). |
| **Pagination Extraction** | All pages of the Work Items table extracted accurately (`114` total items). |
| **Hash Integrity** | Extracted SHA1 digests verified against independent SHA1 calculations. |
| **Empty Queue / Idempotency Run** | Re-running without resetting test data found `0` open `WI5` items, logged a warning, and stopped cleanly. |
| **Unit Testing** | All 5 sub-workflows verified independently via `Run File` prior to integration. |
