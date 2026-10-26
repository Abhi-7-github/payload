# Proposed Solutions: Resolving Folder Location for Draft Documents

This document details two distinct solutions to ensure that draft folder assignments reflect immediately in the Payload admin folder view.

---

## Solution 1: Add Draft Support to Polymorphic Joins in Database Adapters (Recommended)

### High-Level Concept
Extend polymorphic joins in all database adapters (`@payloadcms/drizzle` for SQLite/Postgres and `@payloadcms/db-mongodb` for MongoDB) to support draft versions. When a join requests drafts (e.g., via `draft: true` in the join query options or inheriting request context), the adapter determines whether each target collection has drafts enabled (`hasDraftsEnabled`). If enabled, it joins against that collection's version table/collection rather than its base table.

### Implementation Blueprint

#### 1. API Extension in Join Query Options
Allow passing `draft?: boolean` inside the `joins` query parameter:
```ts
const subfolderDoc = await payload.find({
  collection: payload.config.folders.slug,
  depth: 1,
  joins: {
    documentsAndFolders: {
      draft: true, // New option: request draft versions in the join
      limit: 100_000_000,
      sort: 'name',
      where: combineWhereConstraints([folderWhere, documentWhere], 'or'),
    },
  },
  where: { id: { equals: parentFolderID } },
})
```

#### 2. Drizzle Adapter (`@payloadcms/drizzle`)
Modify [`packages/drizzle/src/find/traverseFields.ts`](file:///c:/Users/hp/Desktop/payload/packages/drizzle/src/find/traverseFields.ts) and [`packages/drizzle/src/find/buildPolymorphicJoinQuery.ts`](file:///c:/Users/hp/Desktop/payload/packages/drizzle/src/find/buildPolymorphicJoinQuery.ts):

1. **Pass `draft` into `buildPolymorphicJoinQuery`**:
   Extract `draft` from the join config (`joins?.[join.joinPath]?.draft ?? draftsEnabled`) and forward it to `buildPolymorphicJoinQuery`.

2. **Per-Collection Table & Field Resolution**:
   In `buildPolymorphicJoinQuery`, iterate through `field.collection`:
   ```ts
   const collectionTables = collections.map((collection) => {
     const collectionConfig = adapter.payload.collections[collection]?.config
     const useDrafts = Boolean(draft && hasDraftsEnabled(collectionConfig))

     const tableName = adapter.tableNameMap.get(
       useDrafts
         ? `_${toSnakeCase(collection)}${adapter.versionsSuffix}`
         : toSnakeCase(collection),
     )
     const table = tableName ? adapter.tables[tableName] : undefined

     return {
       collection,
       table,
       useDrafts,
     }
   })
   ```

3. **Subquery / Union Mapping for Drafts**:
   - For collections where `useDrafts === true`, add filter `{ latest: { equals: true } }`.
   - Map `id` column to `parent_id` (the document ID) and read fields from the version record (handling `version.<field>` structure where necessary).
   - For collections without drafts enabled, join against the base table as before.

#### 3. MongoDB Adapter (`@payloadcms/db-mongodb`)
Modify [`packages/db-mongodb/src/utilities/buildJoinAggregation.ts`](file:///c:/Users/hp/Desktop/payload/packages/db-mongodb/src/utilities/buildJoinAggregation.ts):

1. In the polymorphic joins loop (`for (const collectionSlug of join.field.collection)`):
   ```ts
   const { collectionConfig, Model: JoinModel } = getCollection({ adapter, collectionSlug })
   const useDrafts = Boolean(draftJoin && hasDraftsEnabled(collectionConfig))

   const TargetModel = useDrafts
     ? adapter.versions[collectionSlug]
     : JoinModel
   ```

2. When `useDrafts === true`:
   - Point `$lookup` to `TargetModel.collection.name` (the versions collection).
   - In `$match`, prepend `version.` to foreign field paths (e.g. `version.folder`) and include `{ latest: true }`.
   - In the projection, project `value: '$$doc.parent'` (the actual document ID) instead of the version entry's `_id`.

### Pros & Cons
- **Pros:**
  - **Generic and systematic:** Resolves the issue across all features using polymorphic joins in Payload, not just folders.
  - **High performance:** All child documents across all target collections are fetched in a single database roundtrip / aggregation query.
  - **Consistency:** Unifies behavior between single-collection joins and polymorphic joins.
- **Cons:**
  - **Higher complexity:** Requires modifications across multiple database adapters (`@payloadcms/drizzle` and `@payloadcms/db-mongodb`).
  - **Testing surface:** Requires comprehensive test coverage across SQLite, PostgreSQL, and MongoDB.

---

## Solution 2: Query Subfolders and Documents Separately in `getFolderData` (Scoped Fix)

### High-Level Concept
Bypass the polymorphic `documentsAndFolders` join entirely within `getFolderData` / `queryDocumentsAndFoldersFromJoin`. Instead, perform separate `payload.find` queries:
1. One query for subfolders in `payload-folders`.
2. Individual `payload.find` queries for each folder-enabled collection with `draft: true`.
3. Combine, format, and sort the results in memory.

### Implementation Blueprint

Modify `queryDocumentsAndFoldersFromJoin` or replace it in `packages/payload/src/folders/utils/getFolderData.ts`:

```ts
export async function queryDocumentsAndFoldersDirectly({
  collectionSlug,
  documentWhere,
  folderWhere,
  parentFolderID,
  req,
}: QueryDocumentsAndFoldersArgs): Promise<QueryDocumentsAndFoldersResults> {
  const { payload, user } = req
  const folderSlug = payload.config.folders.slug
  const folderFieldName = payload.config.folders.fieldName

  // 1. Fetch subfolders
  const subfoldersPromise = payload.find({
    collection: folderSlug,
    depth: 0,
    limit: 1000,
    overrideAccess: false,
    req,
    user,
    where: combineWhereConstraints([
      { [folderFieldName]: { equals: parentFolderID } },
      folderWhere,
    ]),
  })

  // 2. Determine target document collections
  const targetCollections = collectionSlug
    ? [collectionSlug]
    : Object.values(payload.collections)
        .filter((col) => col.config.slug !== folderSlug && col.config.folders)
        .map((col) => col.config.slug)

  // 3. Query documents per collection with draft: true
  const documentsPromises = targetCollections.map((slug) =>
    payload.find({
      collection: slug,
      depth: 0,
      draft: true, // Ensures latest draft location and values are returned
      limit: 1000,
      overrideAccess: false,
      req,
      user,
      where: combineWhereConstraints([
        { [folderFieldName]: { equals: parentFolderID } },
        documentWhere,
      ]),
    }).then((res) =>
      res.docs.map((doc) =>
        formatFolderOrDocumentItem({
          folderFieldName,
          isUpload: Boolean(payload.collections[slug]?.config.upload),
          relationTo: slug,
          useAsTitle: payload.collections[slug]?.config.admin?.useAsTitle,
          value: doc,
        })
      )
    )
  )

  const [subfolderRes, ...documentResults] = await Promise.all([
    subfoldersPromise,
    ...documentsPromises,
  ])

  const formattedSubfolders = subfolderRes.docs.map((doc) =>
    formatFolderOrDocumentItem({
      folderFieldName,
      isUpload: false,
      relationTo: folderSlug,
      useAsTitle: 'name',
      value: doc,
    })
  )

  return {
    documents: documentResults.flat(),
    folderAssignedCollections: targetCollections,
    subfolders: formattedSubfolders,
  }
}
```

### Pros & Cons
- **Pros:**
  - **Simple and localized:** Confined to the folders utility; does not risk regressions in core database adapter querying logic.
  - **Quick to implement:** Can be shipped rapidly as a bug fix for the folders UI.
  - **Native draft support:** Automatically leverages `payload.find`'s built-in, fully tested draft handling for each collection.
- **Cons:**
  - **Performance / N+1 queries:** Executes 1 query for subfolders plus 1 query per folder-enabled collection (e.g. 5 collections = 6 database queries).
  - **Does not fix polymorphic joins:** Any other polymorphic join usage in Payload remains unaware of drafts.
  - **Pagination & Sorting across collections:** Slicing and sorting across multiple collections must occur in memory rather than in the database query.

---

## Comparison Matrix

| Attribute | Solution 1: Polymorphic Join Draft Support | Solution 2: Separate `payload.find` Queries |
| :--- | :--- | :--- |
| **Architectural Scope** | Database Adapters (`drizzle`, `db-mongodb`) | Application/Folders Utility (`getFolderData`) |
| **Performance** | High (1 consolidated DB query) | Lower (N+1 queries where N = collections count) |
| **Cross-Collection Sorting** | Handled natively by DB union / aggregation | Handled in Node.js memory |
| **Impact on Other Features** | Fixes drafts for all polymorphic joins in Payload | Fixes folder view only |
| **Implementation Complexity**| Moderate to High | Low |
| **Regression Risk** | Moderate (touches core query engines) | Very Low (isolated to folder fetching) |

---

## Recommendation

- **For immediate resolution / short-term fix:** Implement **Solution 2** if rapid unblocking of the folder UI is required without modifying database adapters.
- **For the long-term, robust solution:** Implement **Solution 1**. Polymorphic joins are a core Payload feature; making them draft-aware aligns them with single-collection joins and ensures consistent behavior across the entire framework.
