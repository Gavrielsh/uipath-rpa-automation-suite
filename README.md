# UiPath RPA & Intelligent Automation Suite

End-to-end RPA and intelligent document automation workflows built with **UiPath Studio**, **Studio Web**, **IXP Document Understanding**, **Maestro Flow**, and **Action Center**.

---

## Projects Included

| Project | Platform & Capabilities | Summary |
| :--- | :--- | :--- |
| **[Intelligent Contract Triage](./IntelligentContractTriage/)** | UiPath Studio Web, IXP Document Understanding, Maestro Flow, Action Center, Quick Forms, Credential Assets | Automates vendor MSA contract review: extracts supplier data, checks compliance via REST API, evaluates legal risk via an Autonomous Agent, routes Medium-risk cases to human review, and onboards approved vendors into ACME System 1. |
| **[Calculate Client Security Hash](./CalculateClientSecurityHash/)** | UiPath Studio (Windows / VB.NET), UI Automation, Object Repository, Regex, Credential Management | Modular E2E RPA workflow processing `WI5` work items in ACME System 1: extracts paginated tables, parses client data via Regex, computes SHA1 digests, and updates records with dynamic selectors and event-driven synchronization. |

---

## Core Engineering & QA Principles Applied
* **Modular & Testable Design:** Isolated sub-workflows with explicit input/output contracts, supporting standalone unit verification prior to end-to-end execution.
* **Human-in-the-Loop & Safety Controls:** Confidence thresholds and deterministic routing rules ensure ambiguous or medium-risk records are escalated to human reviewers rather than processed blindly.
* **Resilient UI & Data Handling:** Dynamic selectors, regex-based parsing, pagination-safe extraction, and event-driven synchronization (`Check App State`) without hard-coded delays.
* **Credential & Data Security:** Sensitive credentials are isolated in Orchestrator Credential Assets (`ACME_ERP_Credentials`) with zero hard-coded secrets in workflow definitions.
