```typescript
import { beforeEach, describe, expect, it, vi } from 'vitest'

// Mocking the database module
vi.mock('@sim/db', () => ({
  db: {
    select: vi.fn(),
    from: vi.fn(),
    where: vi.fn(),
    limit: vi.fn(),
    innerJoin: vi.fn(),
    leftJoin: vi.fn(),
    orderBy: vi.fn(),
  },
}))

// Mocking the database schema module
vi.mock('@sim/db/schema', () => ({
  permissions: {
    permissionType: 'permission_type',
    userId: 'user_id',
    entityType: 'entity_type',
    entityId: 'entity_id',
    id: 'permission_id',
  },
  permissionTypeEnum: {
    enumValues: ['admin', 'write', 'read'] as const,
  },
  user: {
    id: 'user_id',
    email: 'user_email',
    name: 'user_name',
  },
  workspace: {
    id: 'workspace_id',
    name: 'workspace_name',
    ownerId: 'workspace_owner_id',
  },
}))

// Mocking the drizzle-orm module
vi.mock('drizzle-orm', () => ({
  and: vi.fn().mockReturnValue('and-condition'),
  eq: vi.fn().mockReturnValue('eq-condition'),
  or: vi.fn().mockReturnValue('or-condition'),
}))

import { db } from '@sim/db'
import {
  getManageableWorkspaces,
  getUserEntityPermissions,
  getUsersWithPermissions,
  hasAdminPermission,
  hasWorkspaceAdminAccess,
} from '@/lib/permissions/utils'

const mockDb = db as any
type PermissionType = 'admin' | 'write' | 'read'

function createMockChain(finalResult: any) {
  const chain: any = {}

  chain.then = vi.fn().mockImplementation((resolve: any) => resolve(finalResult))
  chain.select = vi.fn().mockReturnValue(chain)
  chain.from = vi.fn().mockReturnValue(chain)
  chain.where = vi.fn().mockReturnValue(chain)
  chain.limit = vi.fn().mockReturnValue(chain)
  chain.innerJoin = vi.fn().mockReturnValue(chain)
  chain.orderBy = vi.fn().mockReturnValue(chain)

  return chain
}

describe('Permission Utils', () => {
  beforeEach(() => {
    vi.clearAllMocks()
  })

  describe('getUserEntityPermissions', () => {
    it('should return null when user has no permissions', async () => {
      const chain = createMockChain([])
      mockDb.select.mockReturnValue(chain)

      const result = await getUserEntityPermissions('user123', 'workspace', 'workspace456')

      expect(result).toBeNull()
    })

    it('should return the highest permission when user has multiple permissions', async () => {
      const mockResults = [
        { permissionType: 'read' as PermissionType },
        { permissionType: 'admin' as PermissionType },
        { permissionType: 'write' as PermissionType },
      ]
      const chain = createMockChain(mockResults)
      mockDb.select.mockReturnValue(chain)

      const result = await getUserEntityPermissions('user123', 'workspace', 'workspace456')

      expect(result).toBe('admin')
    })

    it('should return single permission when user has only one', async () => {
      const mockResults = [{ permissionType: 'read' as PermissionType }]
      const chain = createMockChain(mockResults)
      mockDb.select.mockReturnValue(chain)

      const result = await getUserEntityPermissions('user123', 'workflow', 'workflow789')

      expect(result).toBe('read')
    })

    it('should prioritize admin over other permissions', async () => {
      const mockResults = [
        { permissionType: 'write' as PermissionType },
        { permissionType: 'admin' as PermissionType },
        { permissionType: 'read' as PermissionType },
      ]
      const chain = createMockChain(mockResults)
      mockDb.select.mockReturnValue(chain)

      const result = await getUserEntityPermissions('user999', 'workspace', 'workspace999')

      expect(result).toBe('admin')
    })

    it('should return write permission when user only has write access', async () => {
      const mockResults = [{ permissionType: 'write' as PermissionType }]
      const chain = createMockChain(mockResults)
      mockDb.select.mockReturnValue(chain)

      const result = await getUserEntityPermissions('user123', 'workspace', 'workspace456')

      expect(result).toBe('write')
    })

    it('should prioritize write over read permissions', async () => {
      const mockResults = [
        { permissionType: 'read' as PermissionType },
        { permissionType: 'write' as PermissionType },
      ]
      const chain = createMockChain(mockResults)
      mockDb.select.mockReturnValue(chain)

      const result = await getUserEntityPermissions('user123', 'workspace', 'workspace456')

      expect(result).toBe('write')
    })

    it('should work with workflow entity type', async () => {
      const mockResults = [{ permissionType: 'admin' as PermissionType }]
      const chain = createMockChain(mockResults)
      mockDb.select.mockReturnValue(chain)

      const result = await getUserEntityPermissions('user123', 'workflow', 'workflow789')

      expect(result).toBe('admin')
    })

    it('should work with organization entity type', async () => {
      const mockResults = [{ permissionType: 'read' as PermissionType }]
      const chain = createMockChain(mockResults)
      mockDb.select.mockReturnValue(chain)

      const result = await getUserEntityPermissions('user123', 'organization', 'org456')

      expect(result).toBe('read')
    })

    it('should handle generic entity types', async () => {
      const mockResults = [{ permissionType: 'write' as PermissionType }]
      const chain = createMockChain(mockResults)
      mockDb.select.mockReturnValue(chain)

      const result = await getUserEntityPermissions('user123', 'custom_entity', 'entity123')

      expect(result).toBe('write')
    })
  })

  describe('hasAdminPermission', () => {
    it('should return true when user has admin permission for workspace', async () => {
      const chain = createMockChain([{ id: 'perm1' }])
      mockDb.select.mockReturnValue(chain)

      const result = await hasAdminPermission('admin-user', 'workspace123')

      expect(result).toBe(true)
    })

    it('should return false when user has no admin permission for workspace', async () => {
      const chain = createMockChain([])
      mockDb.select.mockReturnValue(chain)

      const result = await hasAdminPermission('regular-user', 'workspace123')

      expect(result).toBe(false)
    })

    it('should return false when user has write permission but not admin', async () => {
      const chain = createMockChain([])
      mockDb.select.mockReturnValue(chain)

      const result = await hasAdminPermission('write-user', 'workspace123')

      expect(result).toBe(false)
    })

    it('should return false when user has read permission but not admin', async () => {
      const chain = createMockChain([])
      mockDb.select.mockReturnValue(chain)

      const result = await hasAdminPermission('read-user', 'workspace123')

      expect(result).toBe(false)
    })

    it('should handle non-existent workspace', async () => {
      const chain = createMockChain([])
      mockDb.select.mockReturnValue(chain)

      const result = await hasAdminPermission('user123', 'non-existent-workspace')

      expect(result).toBe(false)
    })

    it('should handle empty user ID', async () => {
      const chain = createMockChain([])
      mockDb.select.mockReturnValue(chain)

      const result = await hasAdminPermission('', 'workspace123')

      expect(result).toBe(false)
    })
  })

  describe('getUsersWithPermissions', () => {
    it('should return empty array when no users have permissions for workspace', async () => {
      const usersChain = createMockChain([])
      mockDb.select.mockReturnValue(usersChain)

      const result = await getUsersWithPermissions('workspace123')

      expect(result).toEqual([])
    })

    it('should return users with their permissions for workspace', async () => {
      const mockUsersResults = [
        {
          userId: 'user1',
          email: 'alice@example.com',
          name: 'Alice Smith',
          permissionType: 'admin' as PermissionType,
        },
      ]

      const usersChain = createMockChain(mockUsersResults)
      mockDb.select.mockReturnValue(usersChain)

      const result = await getUsersWithPermissions('workspace456')

      expect(result).toEqual([
        {
          userId: 'user1',
          email: 'alice@example.com',
          name: 'Alice Smith',
          permissionType: 'admin',
        },
      ])
    })

    it('should return multiple users with different permission levels', async () => {
      const mockUsersResults = [
        {
          userId: 'user1',
          email: 'admin@example.com',
          name: 'Admin User',
          permissionType: 'admin' as PermissionType,
        },
        {
          userId: 'user2',
          email: 'writer@example.com',
          name: 'Writer User',
          permissionType: 'write' as PermissionType,
        },
        {
          userId: 'user3',
          email: 'reader@example.com',
          name: 'Reader User',
          permissionType: 'read' as PermissionType,
        },
      ]

      const usersChain = createMockChain(mockUsersResults)
      mockDb.select.mockReturnValue(usersChain)

      const result = await getUsersWithPermissions('workspace456')

      expect(result).toHaveLength(3)
      expect(result[0].permissionType).toBe('admin')
      expect(result[1].permissionType).toBe('write')
      expect(result[2].permissionType).toBe('read')
    })

    it('should handle users with empty names', async () => {
      const mockUsersResults = [
        {
          userId: 'user1',
          email: 'test@example.com',
          name: '',
          permissionType: 'read' as PermissionType,
        },
      ]

      const usersChain = createMockChain(mockUsersResults)
      mockDb.select.mockReturnValue(usersChain)

      const result = await getUsersWithPermissions('workspace123')

      expect(result[0].name).toBe('')
    })
  })

  describe('hasWorkspaceAdminAccess', () => {
    it('should return true when user owns the workspace', async () => {
      const chain = createMockChain([{ ownerId: 'user123' }])
      mockDb.select.mockReturnValue(chain)

      const result = await hasWorkspaceAdminAccess('user123', 'workspace456')

      expect(result).toBe(true)
    })

    it('should return true when user has direct admin permission', async () => {
      let callCount = 0
      mockDb.select.mockImplementation(() => {
        callCount++
        if (callCount === 1) {
          return createMockChain([{ ownerId: 'other-user' }])
        }
        return createMockChain([{ id: 'perm1' }])
      })

      const result = await hasWorkspaceAdminAccess('user123', 'workspace456')

      expect(result).toBe(true)
    })

    it('should return false when workspace does not exist', async () => {
      const chain = createMockChain([])
      mockDb.select.mockReturnValue(chain)

      const result = await hasWorkspaceAdminAccess('user123', 'workspace456')

      expect(result).toBe(false)
    })

    it('should return false when user has no admin access', async () => {
      let callCount = 0
      mockDb.select.mockImplementation(() => {
        callCount++
        if (callCount === 1) {
          return createMockChain([{ ownerId: 'other-user' }])
        }
        return createMockChain([])
      })

      const result = await hasWorkspaceAdminAccess('user123', 'workspace456')

      expect(result).toBe(false)
    })

    it('should return false when user has write permission but not admin', async () => {
      let callCount = 0
      mockDb.select.mockImplementation(() => {
        callCount++
        if (callCount === 1) {
          return createMockChain([{ ownerId: 'other-user' }])
        }
        return createMockChain([])
      })

      const result = await hasWorkspaceAdminAccess('user123', 'workspace456')

      expect(result).toBe(false)
    })

    it('should return false when user has read permission but not admin', async () => {
      let callCount = 0
      mockDb.select.mockImplementation(() => {
        callCount++
        if (callCount === 1) {
          return createMockChain([{ ownerId: 'other-user' }])
        }
        return createMockChain([])
      })

      const result = await hasWorkspaceAdminAccess('user123', 'workspace456')

      expect(result).toBe(false)
    })

    it('should handle empty workspace ID', async () => {
      const chain = createMockChain([])
      mockDb.select.mockReturnValue(chain)

      const result = await hasWorkspaceAdminAccess('user123', '')

      expect(result).toBe(false)
    })

    it('should handle empty user ID', async () => {
      const chain = createMockChain([])
      mockDb.select.mockReturnValue(chain)

      const result = await hasWorkspaceAdminAccess('', 'workspace456')

      expect(result).toBe(false)
    })
  })

  describe('Edge Cases and Security Tests', () => {
    it('should handle SQL injection attempts in user IDs', async () => {
      const chain = createMockChain([])
      mockDb.select.mockReturnValue(chain)

      const result = await getUserEntityPermissions(
        "'; DROP TABLE users; --",
        'workspace',
        'workspace123'
      )

      expect(result).toBeNull()
    })

    it('should handle very long entity IDs', async () => {
      const longEntityId = 'a'.repeat(1000)
      const chain = createMockChain([])
      mockDb.select.mockReturnValue(chain)

      const result = await getUserEntityPermissions('user123', 'workspace', longEntityId)

      expect(result).toBeNull()
    })

    it('should handle unicode characters in entity names', async () => {
      const chain = createMockChain([{ permissionType: 'read' as PermissionType }])
      mockDb.select.mockReturnValue(chain)

      const result = await getUserEntityPermissions('user123', '📝workspace', '🏢org-id')

      expect(result).toBe('read')
    })

    it('should verify permission hierarchy ordering is consistent', () => {
      const permissionOrder: Record<PermissionType, number> = { admin: 3, write: 2, read: 1 }

      expect(permissionOrder.admin).toBeGreaterThan(permissionOrder.write)
      expect(permissionOrder.write).toBeGreaterThan(permissionOrder.read)
    })

    it('should handle workspace ownership checks with null owner IDs', async () => {
      let callCount = 0
      mockDb.select.mockImplementation(() => {
        callCount++
        if (callCount === 1) {
          return createMockChain([{ ownerId: null }])
        }
        return createMockChain([])
      })

      const result = await hasWorkspaceAdminAccess('user123', 'workspace456')

      expect(result).toBe(false)
    })

    it('should handle null user ID correctly when owner ID is different', async () => {
      let callCount = 0
      mockDb.select.mockImplementation(() => {
        callCount++
        if (callCount === 1) {
          return createMockChain([{ ownerId: 'other-user' }])
        }
        return createMockChain([])
      })

      const result = await hasWorkspaceAdminAccess(null as any, 'workspace456')

      expect(result).toBe(false)
    })
  })

  describe('getManageableWorkspaces', () => {
    it('should return empty array when user has no manageable workspaces', async () => {
      const chain = createMockChain([])
      mockDb.select.mockReturnValue(chain)

      const result = await getManageableWorkspaces('user123')

      expect(result).toEqual([])
    })

    it('should return owned workspaces', async () => {
      const mockWorkspaces = [
        { id: 'ws1', name: 'My Workspace 1', ownerId: 'user123' },
        { id: 'ws2', name: 'My Workspace 2', ownerId: 'user123' },
      ]

      let callCount = 0
      mockDb.select.mockImplementation(() => {
        callCount++
        if (callCount === 1) {
          return createMockChain(mockWorkspaces) // Owned workspaces
        }
        return createMockChain([]) // No admin workspaces
      })

      const result = await getManageableWorkspaces('user123')

      expect(result).toEqual([
        { id: 'ws1', name: 'My Workspace 1', ownerId: 'user123', accessType: 'owner' },
        { id: 'ws2', name: 'My Workspace 2', ownerId: 'user123', accessType: 'owner' },
      ])
    })

    it('should return workspaces with direct admin permissions', async () => {
      const mockAdminWorkspaces = [{ id: 'ws1', name: 'Shared Workspace', ownerId: 'other-user' }]

      let callCount = 0
      mockDb.select.mockImplementation(() => {
        callCount++
        if (callCount === 1) {
          return createMockChain([]) // No owned workspaces
        }
        return createMockChain(mockAdminWorkspaces) // Admin workspaces
      })

      const result = await getManageableWorkspaces('user123')

      expect(result).toEqual([
        { id: 'ws1', name: 'Shared Workspace', ownerId: 'other-user', accessType: 'direct' },
      ])
    })

    it('should combine owned and admin workspaces without duplicates', async () => {
      const mockOwnedWorkspaces = [
        { id: 'ws1', name: 'My Workspace', ownerId: 'user123' },
        { id: 'ws2', name: 'Another Workspace', ownerId: 'user123' },
      ]
      const mockAdminWorkspaces = [
        { id: 'ws1', name: 'My Workspace', ownerId: 'user123' }, // Duplicate (should be filtered)
        { id: 'ws3', name: 'Shared Workspace', ownerId: 'other-user' },
      ]

      let callCount = 0
      mockDb.select.mockImplementation(() => {
        callCount++
        if (callCount === 1) {
          return createMockChain(mockOwnedWorkspaces) // Owned workspaces
        }
        return createMockChain(mockAdminWorkspaces) // Admin workspaces
      })

      const result = await getManageableWorkspaces('user123')

      expect(result).toHaveLength(3)
      expect(result).toEqual([
        { id: 'ws1', name: 'My Workspace', ownerId: 'user123', accessType: 'owner' },
        { id: 'ws2', name: 'Another Workspace', ownerId: 'user123', accessType: 'owner' },
        { id: 'ws3', name: 'Shared Workspace', ownerId: 'other-user', accessType: 'direct' },
      ])
    })

    it('should handle empty workspace names', async () => {
      const mockWorkspaces = [{ id: 'ws1', name: '', ownerId: 'user123' }]

      let callCount = 0
      mockDb.select.mockImplementation(() => {
        callCount++
        if (callCount === 1) {
          return createMockChain(mockWorkspaces)
        }
        return createMockChain([])
      })

      const result = await getManageableWorkspaces('user123')

      expect(result[0].name).toBe('')
    })

    it('should handle multiple admin permissions for same workspace', async () => {
      const mockAdminWorkspaces = [
        { id: 'ws1', name: 'Shared Workspace', ownerId: 'other-user' },
        { id: 'ws1', name: 'Shared Workspace', ownerId: 'other-user' }, // Duplicate
      ]

      let callCount = 0
      mockDb.select.mockImplementation(() => {
        callCount++
        if (callCount === 1) {
          return createMockChain([]) // No owned workspaces
        }
        return createMockChain(mockAdminWorkspaces) // Admin workspaces with duplicates
      })

      const result = await getManageableWorkspaces('user123')

      expect(result).toHaveLength(2) // Should include duplicates from admin permissions
    })

    it('should handle empty user ID gracefully', async () => {
      const chain = createMockChain([])
      mockDb.select.mockReturnValue(chain)

      const result = await getManageableWorkspaces('')

      expect(result).toEqual([])
    })
  })
})
```

**Purpose of this file:**

This file contains unit tests for permission utility functions. These functions, located in `src/lib/permissions/utils`, are responsible for determining user permissions and access levels within the application.  The tests ensure that the permission logic is functioning correctly and covers various scenarios, including different permission types, entity types, edge cases, and security considerations.

**Explanation of code:**

**1. Imports and Mocking:**

*   `import { beforeEach, describe, expect, it, vi } from 'vitest'`:  Imports testing utilities from the `vitest` library.
    *   `describe`:  Defines a group of related tests.
    *   `it`: Defines a single test case.
    *   `expect`: Used to make assertions about the test results.
    *   `beforeEach`: Runs a function before each test in a `describe` block.
    *   `vi`:  The `vitest` object, provides mocking capabilities.

*   `vi.mock('@sim/db', ...)`:  Mocks the `@sim/db` module.  This is crucial for isolating the permission logic from the actual database.  Instead of querying a real database, the tests will use mock functions that return predefined results.
    *   `vi.fn()`: Creates a mock function.
    *   The mock implementation returns an object that mimics the database's query builder interface (e.g., `select`, `from`, `where`, `limit`, `innerJoin`, `leftJoin`, `orderBy`).  These mock functions will be configured in the individual tests to return specific data.

*   `vi.mock('@sim/db/schema', ...)`: Mocks the database schema module, providing mock definitions for table and column names used in database queries. This prevents tests from relying on a specific database schema and allows for controlled testing of the logic that interacts with the schema.

*   `vi.mock('drizzle-orm', ...)`: Mocks functions from the `drizzle-orm` library.  `drizzle-orm` is likely an ORM (Object-Relational Mapper) being used in the project.  By mocking `and`, `eq`, and `or`, the tests can control how these functions behave without depending on the actual ORM implementation.

*   `import { db } from '@sim/db'`: Imports the actual `db` object, which is immediately cast to `any` and assigned to `mockDb`. This is done because we're mocking the module, but we still need to import the object to be able to reference it and override its methods in the tests.
*   `import { ... } from '@/lib/permissions/utils'`: Imports the permission utility functions that are being tested.
*   `const mockDb = db as any`:  Assigns the imported (and mocked) `db` object to `mockDb` and casts it to `any`.  This is a common pattern when mocking to bypass TypeScript's type checking and allow for easier overriding of the mock object's properties.
*   `type PermissionType = 'admin' | 'write' | 'read'`: Defines a type alias for the possible permission types.

**2. `createMockChain` Function:**

*   `function createMockChain(finalResult: any)`: Creates a reusable mock object that simulates a database query chain (like what you'd get with an ORM).  This simplifies the setup of mock database interactions.
    *   It returns an object with methods like `select`, `from`, `where`, etc., which are all mock functions that return the same `chain` object.  This allows you to chain these methods together, mimicking a database query builder.
    *   The key part is the `then` method, which is mocked to resolve with the `finalResult` passed into the function.  This simulates the database query returning a specific result.

**3. `describe` Blocks and Test Cases (`it`):**

*   `describe('Permission Utils', ...)`:  Groups all the permission-related tests together.

*   `beforeEach(() => { vi.clearAllMocks() })`:  Resets all the mock functions before each test.  This ensures that each test starts with a clean slate and doesn't have any lingering state from previous tests.

*   `describe('getUserEntityPermissions', ...)`:  Groups tests specifically for the `getUserEntityPermissions` function.

*   `it('should return null when user has no permissions', async () => { ... })`: A single test case.
    *   `const chain = createMockChain([])`: Creates a mock query chain that will return an empty array (simulating no permissions found).
    *   `mockDb.select.mockReturnValue(chain)`: Configures the `mockDb.select` mock function to return the `chain` object.  This means that when the `getUserEntityPermissions` function calls `db.select`, it will interact with this mocked chain.
    *   `const result = await getUserEntityPermissions('user123', 'workspace', 'workspace456')`: Calls the function being tested.
    *   `expect(result).toBeNull()`:  Asserts that the function returns `null`, as expected when the user has no permissions.

The subsequent `it` blocks follow a similar pattern:

1.  **Arrange:** Set up the mock database interactions by configuring the mock functions (usually `mockDb.select`) to return specific results using `createMockChain`.

2.  **Act:** Call the function being tested (`getUserEntityPermissions`, `hasAdminPermission`, etc.).

3.  **Assert:** Use `expect` to verify that the function returns the expected value based on the mock data.

**4. Test Coverage and Scenarios:**

The tests cover a wide range of scenarios for each permission utility function:

*   **`getUserEntityPermissions`:**
    *   No permissions
    *   Multiple permissions with different levels (admin, write, read)
    *   Single permission
    *   Different entity types (workspace, workflow, organization, custom)
    *   Permission hierarchy (admin > write > read)

*   **`hasAdminPermission`:**
    *   User has admin permission
    *   User has no admin permission
    *   User has write or read permission but not admin
    *   Non-existent workspace
    *   Empty user ID

*   **`getUsersWithPermissions`:**
    *   No users with permissions
    *   Single user with permission
    *   Multiple users with different permission levels
    *   Users with empty names

*   **`hasWorkspaceAdminAccess`:**
    *   User owns the workspace
    *   User has direct admin permission
    *   Workspace does not exist
    *   User has no admin access
    *   User has write or read but not admin
    *   Empty workspace ID
    *   Empty user ID

*   **Edge Cases and Security:**
    *   SQL injection attempts in user IDs
    *   Very long entity IDs
    *   Unicode characters in entity names
    *   Permission hierarchy consistency
    *   Null owner IDs
    *   Null user ID

*   **`getManageableWorkspaces`:**
    *   No manageable workspaces
    *   Owned workspaces
    *   Workspaces with direct admin permissions
    *   Combination of owned and admin workspaces
    *   Empty workspace names
    *   Multiple admin permissions for the same workspace
    *   Empty user ID

**Simplifying Complex Logic**

The `createMockChain` function greatly simplifies the test setup by abstracting away the need to create complex mock objects for database queries. It allows you to easily simulate different database query results with a single function call.

**In Summary:**

This file provides a comprehensive set of unit tests for permission utility functions. It uses mocking to isolate the permission logic from external dependencies and covers a wide range of scenarios to ensure that the functions are working correctly. The `createMockChain` function simplifies the test setup and makes the tests more readable.  The "Edge Cases and Security Tests" demonstrate a good commitment to writing robust and secure code.
