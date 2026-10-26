# Root Cause Analysis: Folder Location Not Updating for Draft Documents

## Overview

In Payload CMS, when a collection is configured with both `folders: true` and `versions: { drafts: true }`, moving a document or assigning it to a new folder while in a draft state does not update the document's location in the admin folder view until the document is published.

If a document created in **Folder A** is updated as a draft to be in **Folder B**, it continues to appear in Folder A (and does not appear in Folder B). Furthermore, if a document is newly created as a draft inside a folder, it may not appear in the folder view at all.

---

## Technical Background: How Folders Work in Payload

Folders are implemented via relationships and the **Join field** mechanism:
1. When folders are enabled globally, Payload registers a folder collection (by default `payload-folders`).
2. When `folders: true` is enabled on collections (e.g., `posts`), Payload injects a relationship field (by default named `folder`) that references the `payload-folders` collection.
3. On the `payload-folders` collection, Payload adds a polymorphic join field named `documentsAndFolders`:
   ```ts
   {
     name: 'documentsAndFolders',
     type: 'join',
     collection: ['payload-folders', ...folderEnabledCollections], // e.g. ['payload-folders', 'posts', 'media']
     hasMany: true,
     on: 'folder', // relates back to the folder field on child collections
   }
   ```
4. When rendering a folder in the admin UI, Payload calls `getFolderData`, which executes `queryDocumentsAndFoldersFromJoin`. This queries the current folder and requests its joined items:
   ```ts
   await payload.find({
     collection: 'payload-folders',
     where: { id: { equals: parentFolderID } },
     joins: {
       documentsAndFolders: {
         limit: 100_000_000,
         sort: 'name',
         where: combineWhereConstraints([folderWhere, documentWhere], 'or'),
       },
     },
   })
   ```

---

## The Root Cause

The root cause is an **architectural gap in database adapters where polymorphic joins do not support draft/version tables**, querying only the base collections.

### 1. The Separation of Base Tables vs. Version Tables
When `versions: { drafts: true }` is enabled on a collection:
- **Base Table / Primary Collection** (`posts`): Stores the currently **published** version of a document.
- **Versions Table / Collection** (`_posts_v` in SQLite/Postgres, `_posts_versions` in MongoDB): Stores every draft revision, with the most recent revision flagged as `latest: true`.

When a user edits a document and changes its `folder` field to **Folder B** without publishing:
- The base table (`posts`) continues to hold the published record with `folder = Folder A`.
- The versions table (`_posts_v`) holds the draft record with `folder = Folder B`.

### 2. Disparity Between Single Joins and Polymorphic Joins in Database Adapters

In both `@payloadcms/drizzle` and `@payloadcms/db-mongodb`, standard (single-collection) joins have explicit draft-handling logic, whereas polymorphic joins completely bypass it:

#### A. Drizzle Adapter (`@payloadcms/drizzle`)
In [`packages/drizzle/src/find/traverseFields.ts`](file:///c:/Users/hp/Desktop/payload/packages/drizzle/src/find/traverseFields.ts#L411-L450):
- **Single-collection join** ([`traverseFields.ts:L432-L448`](file:///c:/Users/hp/Desktop/payload/packages/drizzle/src/find/traverseFields.ts#L432-L448)):
  ```ts
  const useDrafts =
    (versions || draftsEnabled) &&
    hasDraftsEnabled(adapter.payload.collections[field.collection].config)

  const joinCollectionTableName = adapter.tableNameMap.get(
    useDrafts
      ? `_${toSnakeCase(field.collection)}${adapter.versionsSuffix}`
      : toSnakeCase(field.collection),
  )
  ```
  It actively checks `useDrafts` and switches the join target from the base table to the version table (`_<table>_v`) filtering by `latest: true`.
- **Polymorphic join** ([`traverseFields.ts:L411-L425`](file:///c:/Users/hp/Desktop/payload/packages/drizzle/src/find/traverseFields.ts#L411-L425)):
  It invokes [`buildPolymorphicJoinQuery`](file:///c:/Users/hp/Desktop/payload/packages/drizzle/src/find/buildPolymorphicJoinQuery.ts#L78-L99):
  ```ts
  const collectionTables = collections.map((collection) => {
    const tableName = adapter.tableNameMap.get(toSnakeCase(collection))
    const table = tableName ? adapter.tables[tableName] : undefined
    // ...
    return { collection, table }
  })
  ```
  It unconditionally targets the base table of each collection in `field.collection`. There is no check for `hasDraftsEnabled`, no version table resolution, and no mechanism for querying drafts.

#### B. MongoDB Adapter (`@payloadcms/db-mongodb`)
In [`packages/db-mongodb/src/utilities/buildJoinAggregation.ts`](file:///c:/Users/hp/Desktop/payload/packages/db-mongodb/src/utilities/buildJoinAggregation.ts#L124-L181):
- For polymorphic joins, the aggregation pipeline loops over `join.field.collection`:
  ```ts
  for (const collectionSlug of join.field.collection) {
    const { Model: JoinModel } = getCollection({ adapter, collectionSlug })

    aggregate.push({
      $lookup: {
        from: JoinModel.collection.name, // base collection only
        // ...
      }
    })
  }
  ```
  It performs `$lookup` operations directly against `JoinModel.collection.name` (the base collection), ignoring draft collections and `versions.drafts`.

### 3. Why PR #17412 Did Not Resolve the Location Issue

PR [#17412](https://github.com/payloadcms/payload/pull/17412) addressed a related symptom: **document titles in folder view**.
- When rendering document cards or rows in the folder view, PR #17412 ensured that the UI loads and displays the draft title instead of the published title.
- However, PR #17412 only changed how the document *properties* are rendered once the document is already included in the folder query result.
- Because the membership of documents within a folder is dictated by the polymorphic join on `documentsAndFolders`, the database join still evaluates `where: { folder: parentFolderID }` against the **base collection table**.
- As a result, the document remains tied to its published folder location (Folder A) with its draft title rendered inside Folder A, while Folder B receives no joined document record.

---

## Summary of Root Cause

1. The folder view relies on a **polymorphic join** (`documentsAndFolders`) on the folder collection targeting all folder-enabled collections.
2. Draft edits to a document's `folder` field exist only in the **version table/collection**, while the **base table/collection** continues to reference the previously published folder (or null).
3. The database adapters (`@payloadcms/drizzle` and `@payloadcms/db-mongodb`) only implement version/draft table resolution for **single-collection joins**; their polymorphic join queries exclusively query the **base tables**.
4. The join therefore evaluates parent-child relationships against outdated published state, preventing draft relocations from appearing in the new folder.
