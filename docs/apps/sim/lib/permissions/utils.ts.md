```typescript
import { db } from '@sim/db'
import { permissions, type permissionTypeEnum, user, workspace } from '@sim/db/schema'
import { and, eq } from 'drizzle-orm'

export type PermissionType = (typeof permissionTypeEnum.enumValues)[number]

/**
 * Get the highest permission level a user has for a specific entity
 *
 * @param userId - The ID of the user to check permissions for
 * @param entityType - The type of entity (e.g., 'workspace', 'workflow', etc.)
 * @param entityId - The ID of the specific entity
 * @returns Promise<PermissionType | null> - The highest permission the user has for the entity, or null if none
 */
export async function getUserEntityPermissions(
  userId: string,
  entityType: string,
  entityId: string
): Promise<PermissionType | null> {
  const result = await db
    .select({ permissionType: permissions.permissionType })
    .from(permissions)
    .where(
      and(
        eq(permissions.userId, userId),
        eq(permissions.entityType, entityType),
        eq(permissions.entityId, entityId)
      )
    )

  if (result.length === 0) {
    return null
  }

  const permissionOrder: Record<PermissionType, number> = { admin: 3, write: 2, read: 1 }
  const highestPermission = result.reduce((highest, current) => {
    return permissionOrder[current.permissionType] > permissionOrder[highest.permissionType]
      ? current
      : highest
  })

  return highestPermission.permissionType
}

/**
 * Check if a user has admin permission for a specific workspace
 *
 * @param userId - The ID of the user to check
 * @param workspaceId - The ID of the workspace to check
 * @returns Promise<boolean> - True if the user has admin permission for the workspace, false otherwise
 */
export async function hasAdminPermission(userId: string, workspaceId: string): Promise<boolean> {
  const result = await db
    .select({ id: permissions.id })
    .from(permissions)
    .where(
      and(
        eq(permissions.userId, userId),
        eq(permissions.entityType, 'workspace'),
        eq(permissions.entityId, workspaceId),
        eq(permissions.permissionType, 'admin')
      )
    )
    .limit(1)

  return result.length > 0
}

/**
 * Retrieves a list of users with their associated permissions for a given workspace.
 *
 * @param workspaceId - The ID of the workspace to retrieve user permissions for.
 * @returns A promise that resolves to an array of user objects, each containing user details and their permission type.
 */
export async function getUsersWithPermissions(workspaceId: string): Promise<
  Array<{
    userId: string
    email: string
    name: string
    permissionType: PermissionType
  }>
> {
  const usersWithPermissions = await db
    .select({
      userId: user.id,
      email: user.email,
      name: user.name,
      permissionType: permissions.permissionType,
    })
    .from(permissions)
    .innerJoin(user, eq(permissions.userId, user.id))
    .where(and(eq(permissions.entityType, 'workspace'), eq(permissions.entityId, workspaceId)))
    .orderBy(user.email)

  return usersWithPermissions.map((row) => ({
    userId: row.userId,
    email: row.email,
    name: row.name,
    permissionType: row.permissionType,
  }))
}

/**
 * Check if a user has admin access to a specific workspace
 *
 * @param userId - The ID of the user to check
 * @param workspaceId - The ID of the workspace to check
 * @returns Promise<boolean> - True if the user has admin access to the workspace, false otherwise
 */
export async function hasWorkspaceAdminAccess(
  userId: string,
  workspaceId: string
): Promise<boolean> {
  const workspaceResult = await db
    .select({ ownerId: workspace.ownerId })
    .from(workspace)
    .where(eq(workspace.id, workspaceId))
    .limit(1)

  if (workspaceResult.length === 0) {
    return false
  }

  if (workspaceResult[0].ownerId === userId) {
    return true
  }

  return await hasAdminPermission(userId, workspaceId)
}

/**
 * Get a list of workspaces that the user has access to
 *
 * @param userId - The ID of the user to check
 * @returns Promise<Array<{
 *   id: string
 *   name: string
 *   ownerId: string
 *   accessType: 'direct' | 'owner'
 * }>> - A list of workspaces that the user has access to
 */
export async function getManageableWorkspaces(userId: string): Promise<
  Array<{
    id: string
    name: string
    ownerId: string
    accessType: 'direct' | 'owner'
  }>
> {
  const ownedWorkspaces = await db
    .select({
      id: workspace.id,
      name: workspace.name,
      ownerId: workspace.ownerId,
    })
    .from(workspace)
    .where(eq(workspace.ownerId, userId))

  const adminWorkspaces = await db
    .select({
      id: workspace.id,
      name: workspace.name,
      ownerId: workspace.ownerId,
    })
    .from(workspace)
    .innerJoin(permissions, eq(permissions.entityId, workspace.id))
    .where(
      and(
        eq(permissions.userId, userId),
        eq(permissions.entityType, 'workspace'),
        eq(permissions.permissionType, 'admin')
      )
    )

  const ownedSet = new Set(ownedWorkspaces.map((w) => w.id))
  const combined = [
    ...ownedWorkspaces.map((ws) => ({ ...ws, accessType: 'owner' as const })),
    ...adminWorkspaces
      .filter((ws) => !ownedSet.has(ws.id))
      .map((ws) => ({ ...ws, accessType: 'direct' as const })),
  ]

  return combined
}
```

### Purpose of this file

This TypeScript file defines a set of functions for managing user permissions within a system, likely related to workspaces, workflows, or other entities. It provides functionalities to:

1.  Retrieve the highest permission level a user has for a specific entity.
2.  Check if a user has admin permission for a workspace.
3.  Retrieve a list of users with their permissions for a given workspace.
4.  Check if a user has admin access to a workspace, considering ownership.
5.  Retrieve a list of workspaces a user can manage, differentiating between owned and directly administered workspaces.

These functions interact with a database (likely using Drizzle ORM based on the imports) to query and manipulate permission-related data.

### Explanation of each line of code

**Imports:**

*   `import { db } from '@sim/db'`: Imports the database connection object, likely configured with Drizzle ORM, from the `@sim/db` module.  This allows the file to interact with the database.
*   `import { permissions, type permissionTypeEnum, user, workspace } from '@sim/db/schema'`: Imports the database schema definitions for `permissions`, `user`, and `workspace` tables, and the `permissionTypeEnum` type.  These definitions are essential for constructing database queries and understanding the structure of the data.
*   `import { and, eq } from 'drizzle-orm'`: Imports the `and` and `eq` functions from the Drizzle ORM library. These are used to build complex `WHERE` clauses in database queries, allowing for filtering data based on multiple conditions. `eq` creates an equality condition (e.g., `permissions.userId = userId`), and `and` combines multiple conditions with a logical AND.

**Type Definition:**

*   `export type PermissionType = (typeof permissionTypeEnum.enumValues)[number]`: Defines a TypeScript type `PermissionType` representing the possible values for a permission.  It extracts the allowed values from the `permissionTypeEnum` (assumed to be an enum defined in the database schema) using `permissionTypeEnum.enumValues` and creates a union type of those values.  This makes the code more type-safe, ensuring that only valid permission types are used.

**Function: `getUserEntityPermissions`**

*   `/** ... */`: This is a JSDoc comment that describes the function's purpose, parameters, and return value.  Good documentation is crucial for maintainability.
*   `export async function getUserEntityPermissions( userId: string, entityType: string, entityId: string ): Promise<PermissionType | null> { ... }`: Defines an asynchronous function named `getUserEntityPermissions` that takes a user ID, entity type, and entity ID as input.  It returns a `Promise` that resolves to either a `PermissionType` (the highest permission the user has) or `null` if the user has no permissions for that entity.
*   `const result = await db .select({ permissionType: permissions.permissionType }) .from(permissions) .where( and( eq(permissions.userId, userId), eq(permissions.entityType, entityType), eq(permissions.entityId, entityId) ) )`: This is the core database query.
    *   `db.select({ permissionType: permissions.permissionType })`:  Specifies that we want to select only the `permissionType` column from the `permissions` table. This optimizes the query by only retrieving the necessary data.
    *   `.from(permissions)`: Specifies that we are querying the `permissions` table.
    *   `.where(and(...))`:  Adds a `WHERE` clause to filter the results.  The `and` function combines three equality conditions:
        *   `eq(permissions.userId, userId)`: Filters the results to only include permissions for the specified `userId`.
        *   `eq(permissions.entityType, entityType)`: Filters the results to only include permissions for the specified `entityType` (e.g., 'workspace').
        *   `eq(permissions.entityId, entityId)`: Filters the results to only include permissions for the specified `entityId` (e.g., the ID of a specific workspace).
*   `if (result.length === 0) { return null }`: If the query returns no results (i.e., the user has no permissions for the entity), the function returns `null`.
*   `const permissionOrder: Record<PermissionType, number> = { admin: 3, write: 2, read: 1 }`: Defines a mapping between permission types and their precedence. `admin` has the highest precedence (3), followed by `write` (2), and then `read` (1).  This is used to determine the "highest" permission when a user has multiple permissions for the same entity.
*   `const highestPermission = result.reduce((highest, current) => { return permissionOrder[current.permissionType] > permissionOrder[highest.permissionType] ? current : highest })`:  This line uses the `reduce` method to iterate over the query results and determine the highest permission.
    *   `result.reduce((highest, current) => { ... })`: The `reduce` method iterates over each `current` permission record in the `result` array.  `highest` is an accumulator that stores the permission record considered to be the highest so far.
    *   `permissionOrder[current.permissionType] > permissionOrder[highest.permissionType] ? current : highest`:  Compares the precedence of the `current` permission type with the precedence of the `highest` permission type using the `permissionOrder` mapping.  If the `current` permission has higher precedence, it becomes the new `highest`; otherwise, the `highest` remains the same.
*   `return highestPermission.permissionType`:  Returns the `permissionType` of the highest permission found.

**Function: `hasAdminPermission`**

*   `export async function hasAdminPermission(userId: string, workspaceId: string): Promise<boolean> { ... }`: Defines an asynchronous function named `hasAdminPermission` that checks if a user has admin permission for a given workspace.  It returns a `Promise` that resolves to `true` if the user has admin permission, and `false` otherwise.
*   `const result = await db .select({ id: permissions.id }) .from(permissions) .where( and( eq(permissions.userId, userId), eq(permissions.entityType, 'workspace'), eq(permissions.entityId, workspaceId), eq(permissions.permissionType, 'admin') ) ) .limit(1)`:  This is the database query.
    *   `db.select({ id: permissions.id })`: Selects only the `id` column from the `permissions` table.  We only need to know *if* a record exists, not the details of the record itself.
    *   `.from(permissions)`: Specifies that we are querying the `permissions` table.
    *   `.where(and(...))`:  Adds a `WHERE` clause with multiple conditions:
        *   `eq(permissions.userId, userId)`: Filters for permissions associated with the given `userId`.
        *   `eq(permissions.entityType, 'workspace')`: Filters for permissions related to workspaces.
        *   `eq(permissions.entityId, workspaceId)`: Filters for permissions related to the specified `workspaceId`.
        *   `eq(permissions.permissionType, 'admin')`: Filters for permissions where the `permissionType` is `admin`.
    *   `.limit(1)`: Limits the result set to a maximum of one record. This optimizes the query since we only need to know if *at least one* admin permission record exists.
*   `return result.length > 0`:  Returns `true` if the query returned any results (meaning an admin permission record was found), and `false` otherwise.

**Function: `getUsersWithPermissions`**

*   `export async function getUsersWithPermissions(workspaceId: string): Promise< Array<{ userId: string email: string name: string permissionType: PermissionType }> > { ... }`: Defines an asynchronous function that retrieves a list of users and their permissions for a specific workspace. It returns a promise that resolves to an array of user objects, each including their user ID, email, name, and permission type.
*   `const usersWithPermissions = await db .select({ userId: user.id, email: user.email, name: user.name, permissionType: permissions.permissionType, }) .from(permissions) .innerJoin(user, eq(permissions.userId, user.id)) .where(and(eq(permissions.entityType, 'workspace'), eq(permissions.entityId, workspaceId))) .orderBy(user.email)`: Performs a database query to fetch the user permissions.
    *   `db.select({...})`: Specifies the columns to be selected from the joined tables. Includes `userId`, `email`, and `name` from the `user` table, and `permissionType` from the `permissions` table.
    *   `.from(permissions)`: Starts the query from the `permissions` table.
    *   `.innerJoin(user, eq(permissions.userId, user.id))`: Performs an inner join with the `user` table, linking records where `permissions.userId` matches `user.id`. This combines the user information with their respective permissions.
    *   `.where(and(eq(permissions.entityType, 'workspace'), eq(permissions.entityId, workspaceId)))`: Filters the results to only include permissions that are associated with the given `workspaceId` and are of type `workspace`.
    *   `.orderBy(user.email)`: Orders the results alphabetically by the user's email address.
*   `return usersWithPermissions.map((row) => ({ userId: row.userId, email: row.email, name: row.name, permissionType: row.permissionType, }))`: Transforms the database result into an array of user objects with the desired shape. Maps each row from the database result into an object that includes the user's `userId`, `email`, `name`, and `permissionType`.

**Function: `hasWorkspaceAdminAccess`**

*   `export async function hasWorkspaceAdminAccess( userId: string, workspaceId: string ): Promise<boolean> { ... }`: This function checks if a user has admin access to a workspace, considering both direct admin permissions and workspace ownership.
*   `const workspaceResult = await db .select({ ownerId: workspace.ownerId }) .from(workspace) .where(eq(workspace.id, workspaceId)) .limit(1)`: Retrieves the owner ID of the specified workspace.
    *   `db.select({ ownerId: workspace.ownerId })`: Selects only the `ownerId` column from the `workspace` table.
    *   `.from(workspace)`: Specifies that we are querying the `workspace` table.
    *   `.where(eq(workspace.id, workspaceId))`: Filters the results to only include the workspace with the given `workspaceId`.
    *   `.limit(1)`: Limits the result set to a maximum of one record, as we only need the owner ID of a single workspace.
*   `if (workspaceResult.length === 0) { return false }`: If the workspace doesn't exist (query returns no results), the function returns `false`.
*   `if (workspaceResult[0].ownerId === userId) { return true }`: If the user is the owner of the workspace, they automatically have admin access, so the function returns `true`.
*   `return await hasAdminPermission(userId, workspaceId)`: If the user isn't the owner, the function checks if they have explicit admin permissions using the `hasAdminPermission` function and returns its result.

**Function: `getManageableWorkspaces`**

*   `export async function getManageableWorkspaces(userId: string): Promise< Array<{ id: string name: string ownerId: string accessType: 'direct' | 'owner' }> > { ... }`:  This function retrieves a list of workspaces that a given user can manage, distinguishing between workspaces they own and those where they have direct admin permissions.
*   `const ownedWorkspaces = await db .select({ id: workspace.id, name: workspace.name, ownerId: workspace.ownerId, }) .from(workspace) .where(eq(workspace.ownerId, userId))`: Retrieves a list of workspaces owned by the user.
    *   `db.select({...})`: Selects the `id`, `name`, and `ownerId` columns from the `workspace` table.
    *   `.from(workspace)`: Specifies that we are querying the `workspace` table.
    *   `.where(eq(workspace.ownerId, userId))`: Filters the results to only include workspaces where the `ownerId` matches the given `userId`.
*   `const adminWorkspaces = await db .select({ id: workspace.id, name: workspace.name, ownerId: workspace.ownerId, }) .from(workspace) .innerJoin(permissions, eq(permissions.entityId, workspace.id)) .where( and( eq(permissions.userId, userId), eq(permissions.entityType, 'workspace'), eq(permissions.permissionType, 'admin') ) )`: Retrieves a list of workspaces where the user has admin permissions.
    *   `db.select({...})`: Selects the `id`, `name`, and `ownerId` columns from the `workspace` table.
    *   `.from(workspace)`: Starts the query from the `workspace` table.
    *   `.innerJoin(permissions, eq(permissions.entityId, workspace.id))`: Performs an inner join with the `permissions` table, linking workspaces to permissions where `permissions.entityId` matches `workspace.id`.
    *   `.where(and(...))`: Filters the results to only include permissions that are associated with the user (`eq(permissions.userId, userId)`), are of type `workspace` (`eq(permissions.entityType, 'workspace')`), and grant admin access (`eq(permissions.permissionType, 'admin')`).
*   `const ownedSet = new Set(ownedWorkspaces.map((w) => w.id))`: Creates a `Set` containing the IDs of the workspaces owned by the user.  This is used to efficiently filter out workspaces where the user is both the owner and has direct admin permissions, avoiding duplicates in the final result.
*   `const combined = [ ...ownedWorkspaces.map((ws) => ({ ...ws, accessType: 'owner' as const })), ...adminWorkspaces .filter((ws) => !ownedSet.has(ws.id)) .map((ws) => ({ ...ws, accessType: 'direct' as const })), ]`: Combines the lists of owned and directly administered workspaces into a single array.
    *   `...ownedWorkspaces.map((ws) => ({ ...ws, accessType: 'owner' as const }))`: Maps each owned workspace to a new object that includes the original workspace data and an `accessType` property set to `'owner'`.  The `as const` assertion ensures that the `accessType` is treated as a literal type.
    *   `...adminWorkspaces .filter((ws) => !ownedSet.has(ws.id)) .map((ws) => ({ ...ws, accessType: 'direct' as const }))`: Filters the `adminWorkspaces` to exclude any workspaces that are already in the `ownedSet` (i.e., workspaces the user owns).  Then, it maps each remaining workspace to a new object that includes the original workspace data and an `accessType` property set to `'direct'`.
*   `return combined`: Returns the combined array of workspaces, each with its `accessType` indicating whether the user has access as the owner or through direct admin permissions.

### Simplification and Improvements

While the code is well-structured and readable, here are some suggestions for simplification and potential improvements:

1.  **Helper Function for Building `WHERE` Clauses:**  The `and` and `eq` calls are used repeatedly. A small helper function could reduce duplication:

    ```typescript
    function buildPermissionWhereClause(userId: string, entityType: string, entityId: string) {
      return and(
        eq(permissions.userId, userId),
        eq(permissions.entityType, entityType),
        eq(permissions.entityId, entityId)
      );
    }
    ```

    Then, in `getUserEntityPermissions` and `hasAdminPermission`, you can replace the `and(...)` block with `buildPermissionWhereClause(userId, entityType, entityId)`.

2.  **Centralized Permission Order:** The `permissionOrder` object is currently only used in `getUserEntityPermissions`.  If permission order is a global concept, consider defining it as a constant outside the function, or even in a separate configuration file.  This makes it easier to maintain and update.

3.  **Error Handling:** The code assumes that the database queries will always succeed.  In a production environment, you should add error handling (e.g., `try...catch` blocks) to gracefully handle database connection errors or query failures.  Consider logging errors appropriately.

4.  **Caching:** For frequently accessed permission data, consider implementing caching (e.g., using Redis or in-memory caching).  This can significantly improve performance by reducing the number of database queries.

5.  **More Specific Types:** In the `getManageableWorkspaces` function, the `accessType` is defined as `'direct' | 'owner'`. While this is correct, you could create a specific type alias for this to improve readability:

    ```typescript
    type WorkspaceAccessType = 'direct' | 'owner';
    ```

    Then, use `WorkspaceAccessType` as the type for `accessType`.

6.  **Consider ORM Features:**  Drizzle ORM might offer more advanced features for handling relationships and permissions, such as built-in permission checking or relationship-based queries.  Explore the ORM's documentation to see if there are more efficient ways to implement these functions.

7.  **Batching Permissions Checks:** If you need to check permissions for multiple users or entities simultaneously, consider batching the queries into a single database call. This can reduce network overhead and improve performance.

By implementing these suggestions, you can further improve the readability, maintainability, and performance of the code. Remember to choose the improvements that best fit the specific requirements and constraints of your application.
