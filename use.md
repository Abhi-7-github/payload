# Polymorphic Joins Draft Support & Folder Location Guide

This guide explains what was done to resolve the issue where draft documents moved between folders did not update in the folder view, and how to use the feature in your Payload CMS project.

---

## 1. What Was Done

### Background & Problem
In Payload CMS, folders and hierarchical views use **polymorphic joins** (`documentsAndFolders` or `children`) to connect parent folders to multiple child collections (e.g. `posts`, `media`, `pages`).

Prior to this fix, single-collection joins had support for version/draft tables (`_<table>_v`), but **polymorphic joins only queried the base published database tables**. Consequently:
- When a draft document with `versions: { drafts: true }` was assigned to or moved to a folder, the change only existed in the versions table (`_posts_v`), while the base table (`posts`) still had the old folder (or `null`).
- The folder view's polymorphic join evaluated parent-child relationships against the base table, so documents remained in their old folder or failed to appear in their assigned folder until published.

---

### Implementation Details

We implemented draft awareness for polymorphic joins across the database layer and core types:

#### A. Core API Types ([`packages/payload/src/types/index.ts`](file:///c:/Users/hp/Desktop/payload/packages/payload/src/types/index.ts))
- Added `draft?: boolean` to the `JoinQuery` type interface so queries can explicitly request draft documents for any join.

#### B. Drizzle Adapter (`@payloadcms/drizzle` for SQLite / PostgreSQL)
1. **[`buildPolymorphicJoinQuery.ts`](file:///c:/Users/hp/Desktop/payload/packages/drizzle/src/find/buildPolymorphicJoinQuery.ts)**:
   - Evaluates `useDrafts = Boolean(draft && hasDraftsEnabled(collectionConfig))` for each collection in the polymorphic join.
   - For collections with drafts enabled, targets the version table (`_<table>_v` / `_<table>_versions`), filters by `latest: true`, projects `parent` as `id`, and reads foreign relationship keys and sort fields via `version_<fieldName>`.
   - Preserves fallback support for top-level columns like `createdAt`, `updatedAt`, and `parent`.
2. **[`createPolymorphicJoinWherePlan.ts`](file:///c:/Users/hp/Desktop/payload/packages/drizzle/src/find/createPolymorphicJoinWherePlan.ts)** & **[`buildPolymorphicJoinWhere.ts`](file:///c:/Users/hp/Desktop/payload/packages/drizzle/src/find/buildPolymorphicJoinWhere.ts)**:
   - Resolves filter constraints in polymorphic joins against `version_<column>` for draft collections and maps `id` to the version record's `parent` column.
3. **[`traverseFields.ts`](file:///c:/Users/hp/Desktop/payload/packages/drizzle/src/find/traverseFields.ts)**:
   - Forwards `draft` configuration down to `buildPolymorphicJoinQuery`.

#### C. MongoDB Adapter (`@payloadcms/db-mongodb`)
1. **[`buildJoinAggregation.ts`](file:///c:/Users/hp/Desktop/payload/packages/db-mongodb/src/utilities/buildJoinAggregation.ts)**:
   - In the polymorphic join pipeline loop, checks `useDrafts = Boolean((draftJoin ?? (draftsEnabled || versions)) && hasDraftsEnabled(collectionConfig))`.
   - When active, targets `adapter.versions[collectionSlug]`, matches on `version.<on>` with `{ latest: true }`, and projects `value: '$parent'`.
2. **[`resolveJoins.ts`](file:///c:/Users/hp/Desktop/payload/packages/db-mongodb/src/utilities/resolveJoins.ts)** & **[`find.ts`](file:///c:/Users/hp/Desktop/payload/packages/db-mongodb/src/find.ts)**:
   - Supports passing and honoring `draftJoin` and top-level `draftsEnabled` when resolving joins.

---

## 2. How to Use the Fixed Feature

### Use Case 1: Folders with Draft Collections (Admin UI)

When you enable `folders: true` alongside `versions: { drafts: true }`, any draft moves or creations inside folders are immediately visible in the Admin UI folder view without needing to publish first.

#### Collection Configuration Example

```typescript
import type { CollectionConfig } from 'payload'
import { createFolderField } from 'payload'

export const Posts: CollectionConfig = {
  slug: 'posts',
  admin: {
    useAsTitle: 'title',
  },
  fields: [
    {
      name: 'title',
      type: 'text',
      required: true,
    },
    // Attach document to folders
    createFolderField({ relationTo: 'payload-folders' }),
  ],
  folders: true, // Enables folder support
  versions: {
    drafts: true, // Enables draft workflow
  },
}
```

#### What happens now:
- Create a document as a **Draft** inside "Folder A" -> It immediately appears in "Folder A".
- Edit the document, change its folder to "Folder B", and click **Save Draft** -> It immediately moves to "Folder B" in the Admin Panel without needing to be published.

---

### Use Case 2: Querying Polymorphic Joins in Code (`payload.find`)

You can now use `draft: true` directly inside your `joins` query parameter on any polymorphic join.

#### Defining a Polymorphic Join on a Collection

```typescript
import type { CollectionConfig } from 'payload'

export const Folders: CollectionConfig = {
  slug: 'folders',
  fields: [
    {
      name: 'name',
      type: 'text',
      required: true,
    },
    {
      name: 'documents',
      type: 'join',
      // Polymorphic: joins across multiple collections
      collection: ['posts', 'pages', 'media'],
      on: 'folder',
    },
  ],
}
```

#### Querying with `draft: true`

Pass `draft: true` inside the specific join options to include latest draft versions:

```typescript
const result = await payload.find({
  collection: 'folders',
  where: {
    id: { equals: 'folder-id-123' },
  },
  joins: {
    documents: {
      draft: true, // Requests draft versions for all draft-enabled collections
      limit: 50,
      sort: 'title',
      where: {
        title: { contains: 'My Document' },
      },
    },
  },
})

// result.docs[0].documents.docs will now contain documents in their draft state:
// - id: Document's parent ID (not the version row ID)
// - relationTo: 'posts' | 'pages' | 'media'
// - latest draft changes (including updated folder assignments)
```

#### Top-level Draft Queries

If a top-level query is executed with `draft: true`, polymorphic joins automatically inherit draft mode:

```typescript
const result = await payload.find({
  collection: 'folders',
  draft: true, // Top-level draft query propagates to polymorphic joins
  joins: {
    documents: {
      sort: 'name',
    },
  },
})
```

---

## 3. Running the Dev Server & Verifying

To run the local development server (Frontend Admin UI and Backend API):

```powershell
# Run with SQLite (no docker required)
$env:PAYLOAD_DATABASE="sqlite"
node ./node_modules/tsx/dist/cli.mjs ./test/dev.ts folders
```

- **Admin URL:** [http://localhost:3000/admin](http://localhost:3000/admin)
- **Login Credentials:** `dev@payloadcms.com` / `test` (auto-login enabled)
