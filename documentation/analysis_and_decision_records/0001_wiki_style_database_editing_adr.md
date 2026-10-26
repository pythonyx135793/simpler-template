# ADR: wiki-style database editing

**Status:** Proposed 

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

1. **Structural Changes:** It is not necessary for regular users to be able to change the fundamental structure of the database (e.g., adding new tables or columns). Only administrators should have permission to make these structural changes## Options Considered

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
