Content model

Types
- SOPRegistry: container for SOPs
- SOP: master record (identity; stores immutable SOP code and title)
- SOPRevision: child under SOP; holds per-revision metadata

Behaviors
- All three types use plone.app.dexterity.behaviors.metadata.IBasic for Title/Description
- SOP and SOPRevision also use referenceable and ID behavior like your Journal type

Fields

SOP (master)
- Title: from IBasic (SOP title)
- Description: from IBasic
- sop_code: TextLine
  - Pattern: “A-000” enforced with regex ^[A-Z]-\d{3}$
  - Unique within SOPRegistry (validate against siblings)
- Derived for UX (not stored as fields, but exposed in views and optionally cached on the SOP):
  - latest_version: max(SOPRevision.version)
  - latest_active_version: newest SOPRevision.version with date <= today; if none, it’s empty
  - code_with_version: sop_code + “-” + version (padded to 3 digits) for the chosen revision (active preferred)
  - active_revision: reference to the SOPRevision for latest_active_version (computed or cached)

SOPRevision (child)
- Title: inherit master title (no custom title). Hide on the form and auto-set it to SOP’s Title on save.
- Description: inherit from master on creation (optional)
- version: Int, auto-increment on add (1-based; no upper limit)
- author: single user (reuse @@users_search)
- date: date-only (implementation start)
- expiration_date: date-only
- appendixes: list of rows
  - number: Int (auto-increment within the list starting from 1)
  - date: date-only
  - attachment: NamedBlobFile (one simple file per appendix)
- copies: list of rows
  - number: Int (auto-increment within the list starting from 1)
  - date: date-only
- attachments: one simple file (NamedBlobFile) for the revision itself

Validation/invariants

SOP
- sop_code matches ^[A-Z]-\d{3}$
- sop_code unique within its SOPRegistry (check siblings in the container)

SOPRevision
- version unique within the parent SOP
- expiration_date >= date if both are present
- appendixes: if a row has date or attachment but missing number, assign next number; numbers must be positive and unique per revision
- copies: if a row has date but missing number, assign next number; numbers must be positive and unique per revision

Revisions behavior

Auto-increment version
- When adding a new SOPRevision under an SOP, default version = max(existing versions) + 1
- If user supplies version, validate uniqueness; else set automatically
- No cap on the number of revisions

Latest active vs future-dated revisions
- “Latest active” is the newest revision with date <= today (midnight); if none, “active” is considered empty
- “Latest” is simply the highest version regardless of date
- Views use latest active by default; if none is active yet, show a notice and the next scheduled (earliest future-date) revision

Caching on SOP (optional but recommended)
- latest_version: Int
- latest_active_version: Int
- Recompute in subscribers on revision add/modify/delete so registry listing doesn’t cause N+1 catalog queries
- If you prefer lazy compute first, add the caching later when needed

UI and views

Registry landing (root)
- Your tile view will automatically include SOPRegistry (portal_type endswith “Registry”)

SOPRegistry view (listing)
- One row per SOP (master), not per revision
- Columns:
  - SOP Title (link to SOP view)
  - SOP Code (sop_code)
  - Version (show the “active” version; if none active, show “—” plus a subtle “scheduled vX on date” hint)
  - Date (from latest active revision)
  - Expiration date (from latest active revision)
  - Author (from latest active revision)
  - Attachment (from latest active revision; show a link if present, otherwise “—”)
- Sorting: default by sop_code ascending; alternative sort by date, expiration_date, or version
- Context action: “Add SOP” with permission AddSOP

SOP view (master)
- Header: Title + sop_code
- Highlight panel “Active revision”
  - Code with version: K-001-00X
  - Author, Date, Expiration date
  - Attachments (revision’s file) if present
- If no active revision:
  - Show “No active revision yet.” and optionally “Next scheduled: vX on YYYY-MM-DD”
- Revisions table:
  - version, author, date, expiration_date, “view” link
- Context action: “Add SOP Revision” (always available)
- Optional action: “Add SOP” siblings via breadcrumb

SOPRevision view (child)
- Show all fields including appendixes and copies
- Appendixes list shows number, date, and an attachment link per appendix row
- Breadcrumb: SOPRegistry → SOP → SOPRevision (vX)

Widgets
- author: QuerySelect with @@users_search (like Journal.responsible)
- date fields: date-only widgets, no time
- attachments: single NamedBlobFile for revision; NamedBlobFile inside each appendix row
- lists: use a rows widget (e.g., z3c.form object list pattern) that allows adding/removing rows; auto-number on save

Indexing

SOP (master)
- sop_code: FieldIndex (for filtering and unique checks)
- latest_version: FieldIndex (for sorting/filtering)
- latest_active_version: FieldIndex (for sorting/filtering)

SOPRevision (child)
- version: FieldIndex
- author: FieldIndex (userid)
- date: DateIndex
- expiration_date: DateIndex

Security and permissions

New permissions
- AddSOP: add an SOP in SOPRegistry
- AddSOPRevision: add a revision under SOP

Suggested rolemap defaults
- LabClerk: AddSOPRevision
- LabManager: AddSOP, AddSOPRevision
- Manager: all

Workflow
- Single-state (“active”) for SOPRegistry, SOP, SOPRevision
- No transitions needed at this stage; “active” status is temporal based on revision date

Subscribers and events

On SOPRevision add/modify/delete
- Ensure revision.title = SOP.title (parent)
- Autonumber appendixes and copies:
  - If number missing, fill sequentially starting at 1
  - Enforce uniqueness per list
- Recompute SOP.latest_version and SOP.latest_active_version:
  - latest_version = max(child.version)
  - latest_active_version = max(child.version where child.date <= today)
  - If no active yet, set latest_active_version to 0 or None
- Reindex SOP to update listing columns

On SOP edit (Title change)
- Propagate new title to children revisions (set revision.title = SOP.title where it differs) and reindex them

On SOP creation
- Validate sop_code uniqueness in container

On SOPRegistry install
- Create default folder /senaite_registries/sops (title “SOPs”)
- Add to displayed types for nav
- Add icons (see below)

Icons and theming
- Add sop-registry.svg (registry icon) and sop.svg (SOP icon)
- Icon mapping: extend your IconProvider or rely on name match; your registries root tiles view will use the registry icon

Registry listing performance

Avoid N+1 child fetches in the SOPRegistry view:
- Use cached latest_active_version on each SOP; if empty, fetch latest future (optional)
- Fetch the corresponding revision via a single catalog query by parent path + version, or store latest_active_revision_uid on SOP (optional) to avoid per-row child queries
- Start without the extra UID field; add later if needed

Edge cases and UX notes
- If only future-dated revisions exist:
  - SOP view shows “No active revision yet” and “Next scheduled vX on date”
  - Registry listing shows “—” for active date/version and can show a subtle hint (optional)
- If expiration_date is before today:
  - It’s still the latest active until a newer active revision’s date starts; “expired” state can be derived in UI if you want a badge
- Duplicate sop_code:
  - Block create/edit with a clear error “SOP code must be unique within the SOP Registry”
- sop_code normalization:
  - Strip spaces; uppercase letter; zero-padded 3-digit number; validate strictly against ^[A-Z]-\d{3}$

Setup and structure

GenericSetup
- FTIs: SOPRegistry, SOP, SOPRevision
- Workflows binding: single-state workflow for all three types (can reuse your existing single-state JournalRegistry workflow or define a new minimal workflow)
- Rolemap: add AddSOP and AddSOPRevision permissions and role assignments
- Types.xml: register the three FTIs
- Setuphandlers: create /senaite_registries/sops container; add to displayed types; reindex subtree; map catalogs via set_catalogs like Journals; optionally set ID formatting rules for SOP and SOPRevision if desired

Testing plan (functional)

Happy path
- Create SOP “Bioanalytical method validation” with code K-001
- Add revision v1 with date=today, author=user, attachments present, appendixes (2 rows with files), copies (3 rows)
- Registry lists SOP with code_with_version K-001-001 and the correct author/date
- Add revision v2 with date=tomorrow
  - Registry and SOP view still show v1 as active; SOP view shows “Next scheduled: v2 on …”
- Advance date or change v2 date to today
  - SOP.latest_active_version updates to 2; listing shows K-001-002

Validation
- sop_code “K-01” rejected; “K-001” accepted
- Duplicate sop_code rejected in the same registry
- expiration_date before date rejected
- Missing numbers in appendixes/copies auto-assigned; duplicates rejected

Security
- LabClerk can add SOPRevision but not SOP by default
- LabManager can add both
- Manager can do all
