# ADR: wiki-style database editing

**Status:** Draft 

## Context and Problem Statement

We want to allow many people to collectively edit and contribute information stored in a database (like Airtable, Notion, or SQL). This will help us gather and maintain a single, shared source of truth for [Simpler Co-op](https://github.com/orgs/simpler-coop).

The goal is to enable many people to collectively work on gathering information and contributing it to a shared source of truth in a database. 

What this is NOT: This is *not* simply creating a Markdown file that contains a table of data. That approach will not meet our need for editing a database because it is ultimately store as plain-text, not database values.

## Decision Drivers

## Must have

The solution must satisfy the following needs to be successful:

1. **Database foundation:** The data must live in a proper database (e.g., Airtable, SQL, Notion). We cannot just edit data inside a simple text file.
2. **Individual accountability:** Every user must have an account so that we can track who made which change.
3. **Submit new entries:** Individual users must be able to submit new data to blank fields in the database.
4. **Edit existing entries:** Individual users must be able to edit data that already exists in fields in the database.
5. **Admin review gate:** Administrators must have the ability to review and approve all contributions before they are saved to the main database. The admin review gate should be an option that can be turned on or off. 
6. **Simple access:** Account creation must be easy and quick. We should reuse existing logins (like Google or GitHub accounts) to simplify this process.
7. **Easy user experience (UX):** The process for making edits must be very simple and easy to understand, even for people who are not technical. (For example, complex processes like submitting a "pull request" via GitHub are too complicated for our general users.)

## Nice to have 

These features improve the experience but are not mandatory for the solution to work:

1. **Common tooling:** The solution should primarily re-use common tools (such as Airtable, Google Sheets, or others) that are proven to be stable and reliable at scale. We want to avoid building complex, custom software that introduces new bugs.
2. **User management control:** Admins should have the option to completely remove or ignore all edits made by a specific user (e.g., if a user is determined to be spamming or unreliable).
3. **Live Co-Editing:** Users should be able to see changes made by other users in real-time as they are happening.

## Not necessary

1. **Structural Changes:** It is not necessary for regular users to be able to change the fundamental structure of the database (e.g., adding new tables or columns). Only administrators should have permission to make these structural changes.

## Options Considered

1. **Small custom web app on a SQL database** (the prototype in [`prototypes/wiki_db`](../../prototypes/wiki_db/README.md)).
2. **Airtable** (or Notion) with shared-base edit access plus its own approval tooling.
3. **Google Sheets** with an Apps Script / form layer in front of it.
4. **Self-hosted no-code database** (NocoDB or Baserow) with its built-in roles and OAuth.
5. **Git-based editing** (a CSV/Markdown table edited through pull requests). Listed only to show why it is out: it breaks "database foundation" and "easy UX".

## Decision Outcome

Chosen option: "Small custom web app on a SQL database", because it is the only option that meets all seven must-haves without paid plans or per-seat limits, and it can be shown working with one command. A review of the off-the-shelf tools is in the pros and cons below. The prototype is a **proposal and demonstration**; if the co-op prefers common tooling, option 4 is the best fallback, and the prototype's rules (a proposals queue, a gate flag, per-user rollback) are the checklist to configure there.

### How the proposal works

* **Data:** one SQLite table (`records`: name, website, contact, notes). Only admins change the structure (the "not necessary" item stays admin-only: columns are defined in code/schema).
* **Accounts:** log in with GitHub or Google (OAuth), or the name-only demo login when no OAuth app is configured. Accounts are keyed by the provider's numeric id. Reading is public; editing needs a login.
* **Contributions:** every changed cell is stored as a row in `contributions` (who, which cell, old value, new value, status `pending`/`applied`/`rejected`/`reverted`). This is the audit log.
* **Review gate:** a setting (`review_required`) that admins flip from the Admin page at runtime. On: edits and new entries are `pending` and invisible until an admin approves. Off: edits apply at once but are still logged. Admin edits always apply directly.
* **Editing UX:** a table with an **Edit** button per row, a plain form with one box per field (blank fields are flagged), a **Save** button, and a one-line confirmation. Pages are responsive and need no JavaScript to edit.
* **Conflicts:** each form remembers the values it was opened with; if someone changed a cell meanwhile, the save is refused with a plain message instead of overwriting.
* **Remove a user's edits:** admin clicks **Remove all edits and block** on a user. Their pending edits are rejected, their applied edits are rolled back cell by cell to the latest value from someone else (or the original), their new entries are hidden, their sessions end and they cannot log in. **Restore user** lifts the block (their rolled-back edits stay reverted).
* **Live co-editing:** a Server-Sent Events stream (`/events`) tells open pages that data changed; the table reloads itself, and an open edit form shows a notice instead of discarding typing.
* **Safety:** CSRF token on every POST, HTML escaping, OAuth `state` check, HttpOnly cookies, parameterised SQL.

### How the prototype meets each requirement

| Requirement | How | Test |
| --- | --- | --- |
| Must 1 Database foundation | SQLite tables, not a text file | `test_data_lives_in_sqlite_database` |
| Must 2 Individual accountability | login required; every change row stores the user | `test_every_change_is_attributed_to_an_account`, `test_anonymous_visitors_can_read_but_not_edit` |
| Must 3 Submit new entries | fill blank fields, or **Add a new entry** | `test_user_can_fill_blank_field_and_add_entry` |
| Must 4 Edit existing entries | same form on filled fields, with conflict check | `test_user_can_edit_existing_value`, `test_stale_edit_is_refused` |
| Must 5 Admin review gate, on/off | pending queue, approve/reject, runtime toggle | `test_review_gate_holds_edits_until_admin_approves`, `test_admin_can_reject`, `test_new_entry_is_hidden_until_approved`, `test_admin_can_toggle_gate_and_non_admin_cannot` |
| Must 6 Simple access | GitHub / Google OAuth, one click; demo login for trying it | `test_login_needs_only_a_name_in_demo_and_oauth_is_offered_when_configured`, `test_oauth_callback_creates_account_keyed_by_provider_id` |
| Must 7 Easy UX | plain mobile-friendly forms, clear messages, no Git | `test_pages_are_plain_mobile_friendly_forms`, `test_confirmation_message_tells_user_when_review_pending` |
| Nice 1 Common tooling | partly: only stdlib and SQLite, no custom framework; not an off-the-shelf product (see below) | n/a |
| Nice 2 User management | remove/ignore all edits of one user | `test_admin_can_remove_all_edits_of_a_spammer`, `test_admin_cannot_ban_self` |
| Nice 3 Live co-editing | SSE stream, auto-refresh | `test_live_stream_announces_changes` |

### Positive Consequences

* All seven must-haves and two of three nice-to-haves are met, and the full audit trail is queryable with plain SQL.
* Zero dependencies and one-command start; the code is about 900 lines including tests, so it can be reviewed in one sitting.
* Data stays in an open format (SQLite file) we own; no per-seat pricing or vendor lock-in.

### Negative Consequences

* It is custom software that we must host, patch and keep secure, which is the opposite of the "common tooling" nice-to-have.
* Fixed columns: changing the structure means editing code. A production version would want a schema editor, a real server (reverse proxy + HTTPS, a WSGI/ASGI stack) and backups.
* A live OAuth round trip needs real client credentials and was verified only against a stubbed provider in the tests.
* Rolling back a user's edits does not re-base other people's later edits that built on theirs.

## Pros and Cons of the Options

### 1. Small custom web app on a SQL database

Prototype in `prototypes/wiki_db`.

* **Pros**
  * Good, because it meets every must-have exactly and is easy to extend.
  * Good, because the review gate, rollback and audit log are first-class features.
  * Good, because it costs nothing and has no vendor limits.
* **Cons**
  * Bad, because we own the code, hosting and security.
  * Bad, because it is less polished than mature spreadsheet-style tools.

### 2. Airtable (or Notion)

* **Pros**
  * Good, because it is mature, with a familiar spreadsheet UI, real-time co-editing and a record history.
  * Good, because structural changes by admins are easy.
* **Cons**
  * Bad, because collaborators with edit rights write straight to the base; an approval step needs a separate form/automation layer, and a clean on/off gate is awkward.
  * Bad, because editors are paid seats on most plans and need an Airtable account (Google sign-in works, but the account must be created and invited).
  * Bad, because per-user bulk rollback is limited to manual undo from history.

### 3. Google Sheets + Apps Script / Forms

* **Pros**
  * Good, because everyone already has a Google login and the UI is familiar.
  * Good, because it has live co-editing and version history.
* **Cons**
  * Bad, because a sheet is not really a database (no constraints, easy to break a layout), which the ADR calls out as a risk.
  * Bad, because a review gate needs a custom staging sheet and scripts, and only Google accounts work, not GitHub.
  * Bad, because rolling back one user's edits means digging through version history by hand.

### 4. Self-hosted NocoDB or Baserow

* **Pros**
  * Good, because it is a real database with a spreadsheet UI, roles, SSO and open-source licences, so it best fits the "common tooling" nice-to-have.
  * Good, because structural changes stay with admins.
* **Cons**
  * Bad, because there is no built-in "propose, then approve" workflow for ordinary editors, so a review gate needs extra automation (webhooks plus a staging table) and has not been shown to work here.
  * Bad, because we still self-host and keep it running.
  * Bad, because the non-technical UX is a full spreadsheet app, heavier than a single edit form.

### 5. Git-based editing (CSV/Markdown via pull requests)

* **Pros**
  * Good, because history and review come for free.
* **Cons**
  * Bad, because data is plain text, not a database, and contributors need a Git workflow; both are explicitly ruled out.

## How to run the prototype

```sh
sh prototypes/wiki_db/run.sh        # then open http://127.0.0.1:8000
python3 -m unittest discover -s prototypes/wiki_db -p 'test_*.py' -v
```

Log in as `admin` (demo login) for the admin tools, or any other name as a regular contributor. See the
[prototype README](../../prototypes/wiki_db/README.md) for GitHub/Google setup.

## Links

* [Prototype and setup guide](../../prototypes/wiki_db/README.md)
* [Issue #4: Bounty: wiki-style database editing](https://github.com/simpler-coop/simpler-template/issues/4)
* [NocoDB](https://github.com/nocodb/nocodb), [Baserow](https://baserow.io/), [Airtable](https://airtable.com/)

