# ADR: wiki-style database editing

**Status:** Proposed

## Context and Problem Statement

We want to allow many people to collectively edit and contribute information stored in a database (like Airtable, Notion, or SQL). This will help us gather and maintain a single, shared source of truth for [Simpler Co-op](https://github.com/orgs/simpler-coop).

The goal is to enable many people to collectively work on gathering information and contributing it to a shared source of truth in a database.

What this is NOT: This is *not* simply creating a Markdown file that contains a table of data. That approach will not meet our need for editing a database because it is ultimately stored as plain-text, not database values.

---

## Decision Drivers

### Must have

The solution must satisfy the following needs to be successful:

1. **Database foundation:** The data must live in a proper database (e.g., Airtable, SQL, Notion). We cannot just edit data inside a simple text file.
2. **Individual accountability:** Every user must have an account so that we can track who made which change.
3. **Submit new entries:** Individual users must be able to submit new data to blank fields in the database.
4. **Edit existing entries:** Individual users must be able to edit data that already exists in fields in the database.
5. **Admin review gate:** Administrators must have the ability to review and approve all contributions before they are saved to the main database. The admin review gate should be an option that can be turned on or off.
6. **Simple access:** Account creation must be easy and quick. We should reuse existing logins (like Google or GitHub accounts) to simplify this process.
7. **Easy user experience (UX):** The process for making edits must be very simple and easy to understand, even for people who are not technical. (For example, complex processes like submitting a "pull request" via GitHub are too complicated for our general users.)

### Nice to have

These features improve the experience but are not mandatory for the solution to work:

1. **Common tooling:** The solution should primarily re-use common tools (such as Airtable, Google Sheets, or others) that are proven to be stable and reliable at scale. We want to avoid building complex, custom software that introduces new bugs.
2. **User management control:** Admins should have the option to completely remove or ignore all edits made by a specific user (e.g., if a user is determined to be spamming or unreliable).
3. **Live Co-Editing:** Users should be able to see changes made by other users in real-time as they are happening.

### Not necessary

1. **Structural Changes:** It is not necessary for regular users to be able to change the fundamental structure of the database (e.g., adding new tables or columns). Only administrators should have permission to make these structural changes.

---

## Options Considered

* **Option 1: Airtable / Baserow with Forms, Linked Staging Tables, and Automation Rules**
* **Option 2: Google Sheets with Google Apps Script Approval Workflow & Protected Ranges**
* **Option 3: Dedicated Wiki-Database Portal (FastAPI / Supabase + SQLite/PostgreSQL) with Staging Queue & OAuth (Selected)**

---

## Decision Outcome

**Chosen option:** **Option 3: Dedicated Wiki-Database Portal (FastAPI / Supabase + SQLite/PostgreSQL) with Staging Queue & OAuth**, supplemented by Option 1/Option 2 patterns for zero-code deployments.

### Justification

Option 3 is the only approach that provides a seamless, uncompromised solution to all seven "Must-Have" requirements without incurring high per-seat subscription costs or brittle third-party workflow workarounds:
1. **Unconstrained Staging Queue:** A dedicated database schema with a dual-layer model (`records` production table and `proposed_edits` staging queue) allows field-level diffs, pending tags, and automated rollbacks without cluttering production tables.
2. **Zero-Friction Authentication:** Connects directly to Google and GitHub OAuth providers. Public users log in with one click without needing an Airtable Workspace seat or shared Google Sheet permissions.
3. **Turnkey Admin Review Gate:** The review gate can be switched ON (enforcing admin review of all edits) or OFF (allowing direct community edits like Wikipedia) with a single boolean configuration toggle.
4. **Spam & Moderation Control (Nice-to-Have #2):** Because every change is stored with an immutable audit log linking previous and updated values to user IDs, administrators can revert every contribution made by a specific bad actor in a single operation.
5. **Open Source & Self-Hostable:** Eliminates vendor lock-in, runs entirely open-source, and can be backed by SQLite (local/embedded) or PostgreSQL/Supabase (cloud-hosted).

---

### Architecture & Data Flow

```
                      ┌──────────────────────────────────────┐
                      │    Contributor / Community User      │
                      │   (Authenticated via Google/GitHub)  │
                      └──────────────────┬───────────────────┘
                                         │
                                         ▼ Submit Edit / New Row
                      ┌──────────────────────────────────────┐
                      │      Admin Review Gate Enabled?      │
                      └──────────────┬────────────────┬──────┘
                                     │                │
                             YES (ON)│                │ NO (OFF)
                                     ▼                │
                      ┌──────────────────────────┐    │
                      │  Staging Changeset Queue │    │
                      │    (`proposed_edits`)    │    │
                      └──────────────┬───────────┘    │
                                     │                │
                       Admin Reviews │                │
                      (Approve/Reject)│               │
                                     ▼                ▼
                      ┌──────────────────────────────────────┐
                      │       Main Production Database       │
                      │             (`records`)              │
                      └──────────────────┬───────────────────┘
                                         │
                                         ▼ Record Changes
                      ┌──────────────────────────────────────┐
                      │         Immutable Audit Log          │
                      │             (`audit_log`)            │
                      └──────────────────────────────────────┘
```

---

### Positive Consequences

* **Intuitive Contributor UX:** Users interact with a familiar tabular/spreadsheet layout, clicking on rows to edit fields without learning Git or Markdown syntax.
* **True Database Semantics:** Data is strongly typed and stored in structured relational tables rather than fragile flat files.
* **Granular Auditability:** Every applied edit logs the operation (`INSERT`, `UPDATE`, `DELETE`), target record ID, field name, previous value, new value, submitter ID, and ISO timestamp.
* **No License Cost Bottlenecks:** External contributors do not require paid workspace licenses.
* **Rapid Prototyping:** The lightweight implementation can run locally in seconds via Python standard libraries or FastAPI.

### Negative Consequences

* **Maintenance of Custom Web Layer:** Requires hosting a lightweight web application service (e.g. on Fly.io, Render, Railway, or Streamlit/FastAPI cloud) compared to a fully hosted SaaS product.
* **Real-time Co-presence:** While real-time live co-editing can be achieved via WebSockets or Supabase Realtime, initial iterations rely on periodic polling or optimistic UI updates.

---

## Pros and Cons of the Options

### Option 1: Airtable / Baserow with Linked Staging Tables & Automations

Airtable and Baserow offer polished grid interfaces and built-in database mechanics. A staging system can be approximated by having a "Submissions" table that triggers an automation to copy approved rows to a "Production" table.

* **Pros**
  * High-quality out-of-the-box grid interface.
  * Native field types (dates, dropdowns, attachments).
  * Minimal initial coding required.
* **Cons**
  * **Severe per-seat pricing:** Airtable charges per collaborator; allowing public users to directly edit requires enterprise plans or complex Interface permissions.
  * **Awkward update workflows:** Airtable forms allow creating *new* records, but editing *existing* rows anonymously or via forms requires third-party add-ons (e.g. Fillout, Stacker).
  * **Complex review gates:** Managing field-level diffs in Airtable automations requires complex scripting extensions.

### Option 2: Google Sheets + Google Apps Script Workflow

Google Sheets can be used as a front-end database with Google Apps Script handling protected ranges, staging tabs, and email/web approval prompts.

* **Pros**
  * Zero software cost; widely familiar spreadsheet interface.
  * Native Google Account authentication and change history.
  * Real-time multi-user live co-editing out of the box.
* **Cons**
  * **Not a true relational database:** High risk of accidental formula breakage, cell overwrite, and data type corruption.
  * **Fragile access controls:** Enforcing that users can only submit edits to a staging sheet without editing the master sheet requires fragile Google Drive permission partitioning.
  * **Scale limitations:** Performance degrades significantly as row counts and script execution triggers increase.

### Option 3: Dedicated Wiki-Database Portal (FastAPI / Supabase + SQLite/PostgreSQL) [Chosen]

A lightweight, dedicated web portal with a REST API backend and relational database. Contributors authenticate via Google/GitHub OAuth, view data in an interactive table, and submit new rows or field edits. The backend routes submissions through a staging queue or commits directly based on the admin gate toggle.

* **Pros**
  * Completely fulfills all 7 "Must-Have" criteria.
  * Single-toggle Admin Review Gate (`ON`/`OFF`) with side-by-side visual diffs.
  * One-click spam moderation: Admins can rollback all changes made by a specific user.
  * 100% open-source, zero per-user licensing fees, easily portable.
  * Includes automated test suite verifying data integrity and role permissions.
* **Cons**
  * Requires hosting and maintaining a simple web app container or serverless function.

---

## Working Prototype & Verification

A fully functional, tested working prototype implementing this proposed solution is included directly in this repository:

* **Location:** [`prototypes/wiki_db_editor/`](../../prototypes/wiki_db_editor/)
* **Implementation Details:**
  * Backend: FastAPI + SQLite relational engine ([`database.py`](../../prototypes/wiki_db_editor/database.py), [`app.py`](../../prototypes/wiki_db_editor/app.py))
  * Auth: Role-based user resolution ([`auth.py`](../../prototypes/wiki_db_editor/auth.py))
  * UI: Responsive spreadsheet and review queue dashboard ([`static/`](../../prototypes/wiki_db_editor/static/))
  * Verification Suite: Automated test suite ([`tests/test_wiki_editor.py`](../../prototypes/wiki_db_editor/tests/test_wiki_editor.py)) passing 100% of test cases for all ADR criteria.

---

## Links

* [Simpler Co-op Organization](https://github.com/orgs/simpler-coop)
* [FastAPI Documentation](https://fastapi.tiangolo.com/)
* [SQLite Documentation](https://www.sqlite.org/docs.html)
