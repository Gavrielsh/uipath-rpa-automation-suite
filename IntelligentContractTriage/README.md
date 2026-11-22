# Intelligent Contract Triage

## Overview
This solution automates the initial review and routing of vendor Master Supply Agreements (MSAs) for ACME Corp. It combines document extraction, a compliance lookup, AI-based legal risk analysis, human review, email notification, and RPA-based vendor creation in ACME System 1.

The solution is designed from ACME Corp's perspective as the buyer/customer. Its objective is to identify suppliers that should be rejected, escalated for human review, or automatically onboarded into the ERP system.

---

## Business Outcomes
The process supports three outcomes:

| Outcome | Trigger | Result |
| :--- | :--- | :--- |
| **Red - Reject** | Supplier is blocked/sanctioned, or the contract has High overall legal risk | An email requesting clarification or correction is sent to the supplier |
| **Yellow - Human Review** | Supplier is compliant and the contract has Medium overall legal risk | A reviewer receives a Quick Form task and can approve or reject the case |
| **Green - Auto Approve** | Supplier is compliant and the contract has Low overall legal risk | An RPA workflow creates the supplier in ACME System 1 |

---

## Solution Components
The UiPath Studio Web solution includes the following components:
* **Maestro Flow (`new.flow`):** Orchestrates the complete end-to-end process.
* **Manual Trigger:** Accepts a contract file (`contract_file`) and a supplier email address (`vendor_email`).
* **IXP Extract:** Extracts supplier details from the contract, including vendor name, address, city, and country.
* **HTTP Request:** Calls a public mock compliance API and receives a list of supplier compliance statuses.
* **Autonomous Agent:** Reviews the contract against the ACME Corp Risk Playbook and returns legal risk scores.
* **Decision 1:** Routes blocked/sanctioned suppliers or High-risk contracts to rejection.
* **Decision 2:** Routes Medium-risk contracts to human review and Low-risk contracts to automated onboarding.
* **Quick Form:** Enables a human reviewer to approve or reject a Medium-risk contract.
* **Send Email:** Sends a supplier-facing clarification/rejection email on the red path or after a human rejection.
* **RPA Workflow (`Main.xaml`):** Logs in to ACME System 1 and creates a new vendor record.
* **Credential Asset (`ACME_ERP_Credentials`):** Stores the ACME System 1 username and password securely for the RPA workflow.

---

## End-to-End Flow

```text
Manual Trigger (contract_file, vendor_email)
      │
      ▼
  IXP Extract
      │
      ▼
HTTP Compliance Request
      │
      ▼
Autonomous Legal Risk Agent
      │
      ▼
Decision 1: Blocked/Sanctioned supplier OR OverallRisk = High?
  ├── Yes ──► Send Email (Red - Reject)
  └── No  ──► Decision 2: OverallRisk = Medium?
                ├── Yes ──► Quick Form (Yellow - Human Review)
                │             ├── Approve ──► RPA Workflow (ACME System 1)
                │             └── Reject  ──► Send Email
                └── No  ──► RPA Workflow (Green - Auto Approve)
```

---

## Inputs & Data Contracts

### Trigger Inputs
| Input | Type | Description |
| :--- | :--- | :--- |
| `contract_file` | File | Vendor MSA contract submitted for review |
| `vendor_email` | String | Email address used when the process sends a rejection/clarification request |

### Extracted Supplier Details
The IXP extraction model returns the supplier details required by the RPA workflow:
* Vendor name
* Vendor address
* Vendor city
* Vendor country

These values are mapped from the IXP extraction output into the RPA workflow input arguments. The RPA workflow must not use hard-coded supplier values.

### Compliance Mock API
The solution uses a public mock API whose response contains supplier names and compliance statuses. The decision logic compares the vendor name extracted from the current contract to the API response before evaluating the status.

```json
[
  {
    "name": "Organic Preparations INC.",
    "compliance_status": "Blocked"
  },
  {
    "name": "Northbridge Precision Components, Inc.",
    "compliance_status": "Approved"
  }
]
```
A supplier is routed to the red path only when the matching supplier record has a status of `Blocked` or `Sanctioned`.

### Legal Risk Agent
The Autonomous Agent evaluates five contractual risk areas using the ACME Corp Risk Playbook:
1. Governing Law and Jurisdiction
2. Limitation of Liability and Indemnification
3. Payment Terms
4. Term and Termination
5. Intellectual Property

The agent returns JSON only, assigning a score of `Low`, `Medium`, or `High` to each category and deriving `OverallRisk` (`High` if any category is `High`; otherwise `Medium` if any category is `Medium`; otherwise `Low` only if all categories are `Low`):

```json
{
  "Categories": {
    "GoverningLaw": { "Score": "Low", "Rationale": "..." },
    "Liability": { "Score": "Low", "Rationale": "..." },
    "PaymentTerms": { "Score": "Low", "Rationale": "..." },
    "Termination": { "Score": "Low", "Rationale": "..." },
    "IntellectualProperty": { "Score": "Low", "Rationale": "..." }
  },
  "OverallRisk": "Low"
}
```

---

## RPA Workflow (`Main.xaml`)
The RPA workflow uses UI Automation in ACME System 1:
1. Opens ACME System 1.
2. Retrieves credentials from `ACME_ERP_Credentials`.
3. Logs in.
4. Navigates to **Vendors > Add Vendor**.
5. Enters the extracted vendor name, address, city, and country.
6. Generates a dynamic 9-character Vendor TaxID containing uppercase letters and digits.
7. Clicks **Save Vendor**.

---

## How to Run
1. Publish the latest solution version from **UiPath Studio Web**.
2. In **Manage**, ensure the latest version is installed and the deployment status is successful.
3. Open the deployed **Maestro Flow** and start a new Flow Instance.
4. In the Manual Trigger form, upload one PDF contract as `contract_file`.
5. Enter a valid recipient address as `vendor_email`.
6. Start the Flow Instance and monitor execution through **Flow Instances**, **Element Executions**, and node outputs.

---

## Test Scenarios

| Scenario | Expected Result |
| :--- | :--- |
| Contract for **Organic Preparations INC.** | Compliance status is `Blocked`; follows **Red - Reject** and sends an email |
| Contract for **Northbridge Precision Components, Inc.** with `Low` risk | Compliance is `Approved`; follows **Green - Auto Approve** and runs the RPA workflow |
| Approved supplier with `Medium` overall risk | Creates a **Quick Form** human review task |
| Human reviewer selects **Approve** | RPA workflow creates the vendor in ACME System 1 |
| Human reviewer selects **Reject** | Sends a rejection/clarification email |
| Contract with `High` overall risk | Follows **Red - Reject** and sends an email |

---

## Monitoring & Security Notes
* **Observability:** Diagnose runs via Maestro **Flow Instances**, **Element Executions**, IXP Extract output, HTTP Request output, Autonomous Agent JSON output, and RPA job logs.
* **Credential Security:** ACME System 1 credentials are stored in a UiPath Credential Asset (`ACME_ERP_Credentials`), never hard-coded in the workflow.
* **Data Handling:** Supplier email addresses and contract files are handled according to ACME data-handling policies; the public mock API is used solely for evaluation purposes.
