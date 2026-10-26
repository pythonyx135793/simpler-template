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

1. **Structural Changes:** It is not necessary for regular users to be able to change the fundamental structure of the database (e.g., adding new tables or columns). Only administrators should have permission to make these structural changes## Options Considered

# Response from urmomswiperight 

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


# Response from Standisto

Options Considered: 

* Custom moderation app on a SQL database (Postgres/Supabase, prototyped with SQLite)
* NocoDB (self-hosted, open source "Airtable alternative")
* Baserow (open source "Airtable alternative")
* Airtable with public forms and an approval view
* Notion database
* Google Sheets with Google Forms

### Comparison against the must-haves

| Must have | Custom SQL app | NocoDB | Baserow | Airtable + forms | Notion | Google Sheets + Forms |
|---|---|---|---|---|---|---|
| 1. Real database | ✅ Postgres/SQLite | ✅ Postgres/MySQL/SQLite | ✅ Postgres | ✅ | ⚠️ pages, not relational | ⚠️ spreadsheet |
| 2. Per-user accountability | ✅ every change logged with author | ✅ audit log | ✅ row history (paid tiers for full audit log) | ⚠️ anonymous forms; editors are paid seats | ✅ for members (paid seats) | ⚠️ form email only |
| 3. Submit new entries | ✅ | ✅ | ✅ | ✅ form | ✅ | ✅ form |
| 4. Edit existing entries | ✅ | ✅ | ✅ | ❌ forms only create rows; editing needs a paid seat | ✅ | ❌ via forms / ⚠️ direct sheet edit has no gate |
| 5. Admin review gate, toggleable | ✅ built in, one switch | ❌ no native pending-change queue (edits go live) | ❌ no native approval of edits | ⚠️ for new rows only, via a "status" field and filtered views | ❌ | ⚠️ for new rows only |
| 6. Google/GitHub login | ✅ both | ⚠️ Google SSO; GitHub not built in | ⚠️ SSO in paid/enterprise | ❌ needs Airtable account | ⚠️ Google only, needs Notion account | ✅ Google only |
| 7. Simple for non-technical users | ✅ a table and a form | ✅ | ✅ | ✅ | ✅ | ✅ |

No off-the-shelf tool meets **must-have 5 for edits to existing rows**: a pending queue where edits only reach the main table after approval, and that can be switched off. Most of them also require each contributor to hold a seat or account on the vendor's platform, which costs money and slows sign-up.

## Decision Outcome

Chosen option: "Custom moderation app on a SQL database". It is the only option that meets all seven must-haves, in particular the toggleable review gate for both new and edited entries combined with one-click GitHub/Google login. The app is deliberately tiny (one Python file, about 300 lines, using standard Flask + OAuth2). The data lives in a standard SQL database, so it can later be connected to common tools (Supabase dashboard, NocoDB, Metabase, Google Sheets export) for admins. This partially addresses the "common tooling" nice-to-have.

**How it works**

* `entries` holds the approved, public data (the "main database").
* Every contribution, new or edit, is first written to `proposals` with its author, timestamp and status (`pending`/`approved`/`rejected`).
* **Gate ON:** proposals wait in the admin queue. Approve copies the values into `entries`; Reject leaves `entries` untouched.
* **Gate OFF:** proposals are auto-approved and applied instantly, but they are still recorded, so accountability is kept.
* Admins (configured by email or GitHub login) can toggle the gate, review proposals and **ban a user**. A ban rejects all of that user's pending changes and blocks new ones (nice-to-have 2).
* Every entry has a public **History** page listing who changed what.
* Only admins change the schema (columns are defined in code/SQL). Structural changes are not exposed to users.

Prototype, setup instructions and tests: [`prototypes/wiki_db_editing/`](../../prototypes/wiki_db_editing/README.md).

### Positive Consequences

* Meets every must-have, including a review gate that works for edits as well as new entries.
* Zero per-user licence cost; contributors only need an existing GitHub or Google account.
* Data is in a standard SQL database we own. It is easy to back up, query, export, or move to managed Postgres (Supabase, Neon, etc.).
* A full, per-user change history comes for free from the `proposals` table.

### Negative Consequences

* It is custom code: someone has to host it (any small Python host) and maintain it. This goes against the "common tooling" nice-to-have.
* There is no live co-editing (nice-to-have 3). Pages show the latest data on reload. This could be added later with websockets or Supabase Realtime.
* Adding a column means a small code/SQL change by an admin.
* Edits are whole-row proposals. If two people edit the same row while the gate is on, the later approval wins. A future improvement is to show a diff against current values in the queue.

## Pros and Cons of the Options

### Custom moderation app on a SQL database

A small Flask app with GitHub/Google OAuth, a `proposals` queue and an admin toggle, backed by SQLite (prototype) or Postgres/Supabase (production).

* **Pros**
  * Good, because it is the only option meeting all must-haves.
  * Good, because it is free for unlimited contributors.
  * Good, because the data stays in an open, standard database.
* **Cons**
  * Bad, because it needs hosting and some maintenance.
  * Bad, because it has no real-time co-editing out of the box.

### NocoDB

An open-source spreadsheet-style UI on top of a SQL database.

* **Pros**
  * Good, because it gives a polished spreadsheet UX on a real SQL database, with an audit log.
  * Good, because it is open source and self-hostable.
* **Cons**
  * Bad, because it has no pending-approval queue for edits; edits by editors go live immediately (fails must-have 5).
  * Bad, because GitHub login is not built in.
  * It could still be added later as an admin UI on the same database used by the chosen app.

### Baserow

An open-source Airtable alternative.

* **Pros**
  * Good, because it has a good UX, a real Postgres database and row comments/history.
* **Cons**
  * Bad, because it has no approval workflow for edits (fails must-have 5).
  * Bad, because SSO and full audit logs are in paid tiers.

### Airtable with forms and an approval view

The public submits through a form, rows get a "Status" field, and the public view shows only approved rows.

* **Pros**
  * Good, because it is a proven, stable tool and admins likely know it already.
  * Good, because a review gate for *new* rows is easy to set up.
* **Cons**
  * Bad, because forms cannot edit existing rows; editing needs a paid collaborator seat, and then there is no gate (fails must-haves 4 and 5).
  * Bad, because form submitters are not authenticated (fails must-haves 2 and 6).

### Notion database

* **Pros**
  * Good, because it is very friendly UX with page history.
* **Cons**
  * Bad, because it has no approval gate (fails must-have 5).
  * Bad, because every contributor must join the workspace, and paid seats add cost.
  * Bad, because it is a weak "real database" (limited querying and API rate limits).

### Google Sheets + Google Forms

* **Pros**
  * Good, because it is free, familiar and uses Google login.
* **Cons**
  * Bad, because it is a spreadsheet, not a proper database (fails must-have 1).
  * Bad, because forms only add rows; editing the sheet directly bypasses any review (fails must-haves 4 and 5).
  * Bad, because there is no GitHub login.

## Links
* [Prototype source and README](../../prototypes/wiki_db_editing/README.md)
* [NocoDB](https://nocodb.com/) · [Baserow](https://baserow.io/) · [Airtable forms](https://support.airtable.com/docs/creating-forms-in-airtable) · [Supabase](https://supabase.com/)
* [GitHub OAuth apps](https://docs.github.com/en/apps/oauth-apps/building-oauth-apps/authorizing-oauth-apps) · [Google OAuth 2.0](https://developers.google.com/identity/protocols/oauth2/web-server)

# Response from tamamatuaai 

## Additional proposed option: SQLite contribution-review notebook

This is a proposed option for maintainers to evaluate, not a claim that the organization has adopted it.

**Working prototype and startup instructions:** [prototypes/wiki_sql](../../prototypes/wiki_sql/README.md).
Run `sh prototypes/wiki_sql/run.sh --demo` from the repository root. The self-contained Python standard-library application stores data in SQLite and opens at `http://127.0.0.1:8000`. Real GitHub OAuth is separately configurable through environment variables; demonstration mode is explicitly labeled and isolated from the real-login database.

### How it satisfies the must-haves

| Requirement | Implementation |
| --- | --- |
| Database foundation | Canonical values live in SQLite `records`; proposed changes live separately in `changes`. |
| Individual accountability | GitHub numeric IDs identify individual accounts; every change records the author and review decisions record the reviewer. |
| Submit new data to blank fields | Empty fields in seeded records have editable text areas with a submit button. |
| Edit existing fields | The same form submits before/after values for filled fields. |
| Optional admin review | An administrator can approve/reject pending contributions or toggle immediate publication. Gate changes affect new submissions; old pending changes still require review. |
| Simple access | Continue with GitHub reuses an existing account without requesting email/repository scope. The administrator supplies the OAuth app configuration once. |
| Easy UX | Contributors see plain labeled fields and save/review buttons, plus visible status/history; no PR workflow is required of contributors. |

### Pros

- One startup command for a credential-free, explicitly synthetic local demonstration; no dependency downloads.
- Clear separation of canonical and pending values, stable user attribution, atomic approval and audit history.
- Explicit administrator IDs, session CSRF protection, escaped output, and stale-value conflict detection.
- Review can be toggled without losing pending work or change history.
- Twelve automated database and HTTP tests cover permissions, persistence, rejection, conflicts, gate transitions, and mocked OAuth state/identity handling.

### Cons and follow-up work

- This is custom application code to maintain, rather than an off-the-shelf managed service. SQLite is common; the UI/review layer is bespoke.
- The local development HTTP server and in-memory sessions are unsuitable for shared production deployment. HTTPS, durable sessions, operational controls, backups and membership policy need design before hosting.
- A real GitHub OAuth app must be configured by the reviewer/operator. Provider calls are mocked in tests; a live credentialed OAuth round trip is not claimed as tested.
- No live co-editing, schema editor, bulk removal of a user's historical contributions, or creation/deletion of whole records. The must-have blank-field and existing-field workflows are demonstrated; broader workflows remain extensions.
- Conflict detection compares before-values and prevents overwriting a different current value; it is not a full versioned collaborative merge engine.
