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

* {option 1}  
* {option 2}  
* ...

## Decision Outcome

Chosen option: "{option 1}", because {justification. e.g., only option which meets a key decision driver \| which satisfies x condition \| ... }.

### Positive Consequences

* {e.g., improved performance on quality metric, new capability enabled, ...}  
* ...

### Negative Consequences

* {e.g., decreased performance on quality metric, risk, follow-up decisions required, ...}  
* ...

## Pros and Cons of the Options

### {option 1}

{example \| description \| pointer to more information \| ...}

* **Pros**  
  * Good, because {argument a}  
  * Good, because {argument b}  
  * ...  
* **Cons**  
  * Bad, because {argument c}  
  * ...

### {option 2}

{example \| description \| pointer to more information \| ...}

* **Pros**  
  * Good, because {argument a}  
  * Good, because {argument b}  
  * ...  
* **Cons**  
  * Bad, because {argument c}  
  * ...

## Links

* \[{Link name}\](link to external resource)  
* ...


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
