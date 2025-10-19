```typescript
export interface User {
  name?: string
  email?: string
  id?: string
}

export interface Member {
  id: string
  role: string
  user?: User
}

export interface Invitation {
  id: string
  email: string
  status: string
}

export interface Organization {
  id: string
  name: string
  slug: string
  logo?: string | null
  members?: Member[]
  invitations?: Invitation[]
  createdAt: string | Date
  [key: string]: unknown
}

export interface Subscription {
  id: string
  plan: string
  status: string
  seats?: number
  referenceId: string
  cancelAtPeriodEnd?: boolean
  periodEnd?: number | Date
  trialEnd?: number | Date
  metadata?: any
  [key: string]: unknown
}

export interface WorkspaceInvitation {
  workspaceId: string
  permission: string
}

export interface Workspace {
  id: string
  name: string
  ownerId: string
  isOwner: boolean
  canInvite: boolean
}

export interface OrganizationFormData {
  name: string
  slug: string
  logo: string
}

export interface MemberUsageData {
  userId: string
  userName: string
  userEmail: string
  currentUsage: number
  usageLimit: number
  percentUsed: number
  isOverLimit: boolean
  role: string
  joinedAt: string
  lastActive: string | null
}

export interface OrganizationBillingData {
  organizationId: string
  organizationName: string
  subscriptionPlan: string
  subscriptionStatus: string
  totalSeats: number
  usedSeats: number
  seatsCount: number
  totalCurrentUsage: number
  totalUsageLimit: number
  minimumBillingAmount: number
  averageUsagePerMember: number
  billingPeriodStart: string | null
  billingPeriodEnd: string | null
  members?: MemberUsageData[]
  userRole?: string
  billingBlocked?: boolean
}

export interface OrganizationState {
  // Core organization data
  organizations: Organization[]
  activeOrganization: Organization | null

  // Team management
  subscriptionData: Subscription | null
  userWorkspaces: Workspace[]

  // Organization billing and usage
  organizationBillingData: OrganizationBillingData | null

  // Organization settings
  orgFormData: OrganizationFormData

  // Loading states
  isLoading: boolean
  isLoadingSubscription: boolean
  isLoadingOrgBilling: boolean
  isCreatingOrg: boolean
  isInviting: boolean
  isSavingOrgSettings: boolean

  // Error states
  error: string | null
  orgSettingsError: string | null

  // Success states
  inviteSuccess: boolean
  orgSettingsSuccess: string | null

  // Cache timestamps
  lastFetched: number | null
  lastSubscriptionFetched: number | null
  lastOrgBillingFetched: number | null

  // User permissions
  hasTeamPlan: boolean
  hasEnterprisePlan: boolean
}

export interface OrganizationStore extends OrganizationState {
  loadData: () => Promise<void>
  loadOrganizationSubscription: (orgId: string) => Promise<void>
  loadOrganizationBillingData: (organizationId: string, force?: boolean) => Promise<void>
  loadUserWorkspaces: (userId?: string) => Promise<void>
  refreshOrganization: () => Promise<void>

  // Organization management
  createOrganization: (name: string, slug: string) => Promise<void>
  setActiveOrganization: (orgId: string) => Promise<void>
  updateOrganizationSettings: () => Promise<void>

  // Team management
  inviteMember: (email: string, workspaceInvitations?: WorkspaceInvitation[]) => Promise<void>
  removeMember: (memberId: string, shouldReduceSeats?: boolean) => Promise<void>
  cancelInvitation: (invitationId: string) => Promise<void>

  // Seat management
  addSeats: (newSeatCount: number) => Promise<void>
  reduceSeats: (newSeatCount: number) => Promise<void>

  transferSubscriptionToOrganization: (orgId: string) => Promise<void>

  getUserRole: (userEmail?: string) => string
  isAdminOrOwner: (userEmail?: string) => boolean
  getUsedSeats: () => { used: number; members: number; pending: number }

  setOrgFormData: (data: Partial<OrganizationFormData>) => void

  clearError: () => void
  clearSuccessMessages: () => void
}
```

## Explanation of the TypeScript Code

This TypeScript code defines a set of interfaces and a store interface related to organization and team management, billing, and user roles within a larger application.  Let's break down each section:

**Purpose of the File**

The primary purpose of this file is to define the data structures (interfaces) and the actions (methods within the `OrganizationStore` interface) that are essential for managing organizations, teams, billing, and user permissions. It acts as a contract for how these entities are shaped and how the application interacts with them. This is common in frontend applications built with state management libraries like Zustand or Redux.  This file provides the type definitions that would be used with the state management library to ensure type safety.

**General Structure and TypeScript Concepts**

*   **Interfaces:**  TypeScript interfaces define contracts.  They specify the *shape* of an object.  An object that implements an interface *must* have the properties defined in the interface, with the correct types. Interfaces promote code reusability and maintainability, and allow the Typescript compiler to do type-checking.
*   **`export`:** The `export` keyword makes the interfaces available for use in other modules (files) within the application.
*   **`?` (Optional Properties):**  The question mark (`?`) after a property name indicates that the property is *optional*. An object of that type doesn't *have* to have that property defined.
*   **`string | null`:** This defines a union type. A property defined as `string | null` can either be a string or the value `null`.
*   **`string | Date`:** Another union type. A property defined as `string | Date` can either be a string or a Date object.
*   **`any`:** The `any` type disables type checking for that property. While sometimes necessary, it's generally best to avoid `any` to maintain type safety.
*   **`[key: string]: unknown`:** This is an index signature. It allows the interface to have additional properties with string keys, and the values of these properties can be of any type (`unknown`). This is useful for representing data that might have arbitrary or dynamic fields. Using `unknown` is safer than `any` as it forces you to check the type before using the property.
*   **`Promise<void>`:** Indicates an asynchronous function that doesn't return a value.
*   **`Promise<SomeType>`:** Indicates an asynchronous function that returns a value of type `SomeType`.
*   **`Partial<OrganizationFormData>`:** This uses TypeScript's `Partial` utility type.  `Partial<T>` makes all properties of the `T` interface optional. So, `Partial<OrganizationFormData>` means the argument to `setOrgFormData` can be an object with *some or all* of the properties of `OrganizationFormData`.

**Detailed Breakdown of Each Interface**

1.  **`User`**

    ```typescript
    export interface User {
      name?: string
      email?: string
      id?: string
    }
    ```

    *   Represents a user in the system.
    *   `name`: User's name (optional string).
    *   `email`: User's email address (optional string).
    *   `id`: User's ID (optional string).
    *   All properties are optional, suggesting a user object might be partially populated in some contexts.

2.  **`Member`**

    ```typescript
    export interface Member {
      id: string
      role: string
      user?: User
    }
    ```

    *   Represents a member of an organization.
    *   `id`: Member's ID (required string).
    *   `role`: Member's role within the organization (required string).  Examples: "admin", "member", "owner".
    *   `user`:  The `User` object associated with this member (optional). This allows you to directly access user details from a member object.

3.  **`Invitation`**

    ```typescript
    export interface Invitation {
      id: string
      email: string
      status: string
    }
    ```

    *   Represents an invitation to join an organization.
    *   `id`: Invitation ID (required string).
    *   `email`: Email address the invitation was sent to (required string).
    *   `status`: Status of the invitation (required string). Examples: "pending", "accepted", "declined".

4.  **`Organization`**

    ```typescript
    export interface Organization {
      id: string
      name: string
      slug: string
      logo?: string | null
      members?: Member[]
      invitations?: Invitation[]
      createdAt: string | Date
      [key: string]: unknown
    }
    ```

    *   Represents an organization.
    *   `id`: Organization ID (required string).
    *   `name`: Organization name (required string).
    *   `slug`:  A URL-friendly version of the organization name (required string).  Example: "my-company" for an organization named "My Company".
    *   `logo`: URL of the organization's logo (optional string or null).
    *   `members`: Array of `Member` objects representing the organization's members (optional).
    *   `invitations`: Array of `Invitation` objects for pending invitations to the organization (optional).
    *   `createdAt`: Date and time when the organization was created (string or Date object).
    *   `[key: string]: unknown`: Allows the `Organization` object to have additional properties with string keys and values of any type.

5.  **`Subscription`**

    ```typescript
    export interface Subscription {
      id: string
      plan: string
      status: string
      seats?: number
      referenceId: string
      cancelAtPeriodEnd?: boolean
      periodEnd?: number | Date
      trialEnd?: number | Date
      metadata?: any
      [key: string]: unknown
    }
    ```

    *   Represents a subscription to a service.
    *   `id`: Subscription ID (required string).
    *   `plan`: Name of the subscription plan (required string). Examples: "free", "team", "enterprise".
    *   `status`: Status of the subscription (required string). Examples: "active", "canceled", "trialing".
    *   `seats`: Number of seats included in the subscription (optional number).
    *   `referenceId`: External reference ID for the subscription (required string).  Likely used to link to a payment gateway like Stripe.
    *   `cancelAtPeriodEnd`: Whether the subscription is set to cancel at the end of the current billing period (optional boolean).
    *   `periodEnd`: Date and time when the current billing period ends (optional number or Date object).
    *   `trialEnd`: Date and time when the trial period ends (optional number or Date object).
    *   `metadata`:  Additional data associated with the subscription (optional - can be any type).
    *   `[key: string]: unknown`:  Allows the `Subscription` object to have additional properties with string keys and values of any type.

6.  **`WorkspaceInvitation`**

    ```typescript
    export interface WorkspaceInvitation {
      workspaceId: string
      permission: string
    }
    ```

    *   Represents an invitation to a specific workspace within the organization.
    *   `workspaceId`: ID of the workspace (required string).
    *   `permission`: Level of access granted within the workspace (required string). Examples: "read", "write".

7.  **`Workspace`**

    ```typescript
    export interface Workspace {
      id: string
      name: string
      ownerId: string
      isOwner: boolean
      canInvite: boolean
    }
    ```

    *   Represents a workspace within an organization.
    *   `id`: Workspace ID (required string).
    *   `name`: Workspace name (required string).
    *   `ownerId`: ID of the user who owns the workspace (required string).
    *   `isOwner`: Indicates whether the current user is the owner of the workspace (required boolean).
    *   `canInvite`: Indicates whether the current user has permission to invite others to the workspace (required boolean).

8.  **`OrganizationFormData`**

    ```typescript
    export interface OrganizationFormData {
      name: string
      slug: string
      logo: string
    }
    ```

    *   Represents the data required to create or update an organization.  This is likely used in a form.
    *   `name`: Organization name (required string).
    *   `slug`: Organization slug (required string).
    *   `logo`: URL of the organization logo (required string).

9.  **`MemberUsageData`**

    ```typescript
    export interface MemberUsageData {
      userId: string
      userName: string
      userEmail: string
      currentUsage: number
      usageLimit: number
      percentUsed: number
      isOverLimit: boolean
      role: string
      joinedAt: string
      lastActive: string | null
    }
    ```

    *   Represents data about a member's usage of the application's resources.
    *   `userId`: ID of the user (required string).
    *   `userName`: Name of the user (required string).
    *   `userEmail`: Email of the user (required string).
    *   `currentUsage`: The user's current resource consumption (required number).
    *   `usageLimit`: The user's maximum resource allowance (required number).
    *   `percentUsed`: Percentage of the user's usage limit that has been consumed (required number).
    *   `isOverLimit`: Indicates whether the user has exceeded their usage limit (required boolean).
    *   `role`: The user's role within the organization (required string).
    *   `joinedAt`: Date the member joined (required string).
    *   `lastActive`: Date and time when the user was last active (string or null).

10. **`OrganizationBillingData`**

    ```typescript
    export interface OrganizationBillingData {
      organizationId: string
      organizationName: string
      subscriptionPlan: string
      subscriptionStatus: string
      totalSeats: number
      usedSeats: number
      seatsCount: number
      totalCurrentUsage: number
      totalUsageLimit: number
      minimumBillingAmount: number
      averageUsagePerMember: number
      billingPeriodStart: string | null
      billingPeriodEnd: string | null
      members?: MemberUsageData[]
      userRole?: string
      billingBlocked?: boolean
    }
    ```

    *   Represents billing and usage data for an organization.
    *   `organizationId`: ID of the organization (required string).
    *   `organizationName`: Name of the organization (required string).
    *   `subscriptionPlan`: The organization's current subscription plan (required string).
    *   `subscriptionStatus`: The status of the organization's subscription (required string).
    *   `totalSeats`: The total number of seats available to the organization (required number).
    *   `usedSeats`: The number of seats currently in use (required number).
    *   `seatsCount`:  Likely a redundant field similar to `totalSeats`. Could represent total number of seats in use by members.
    *   `totalCurrentUsage`: The organization's total resource consumption (required number).
    *   `totalUsageLimit`: The organization's overall usage limit (required number).
    *   `minimumBillingAmount`: Minimum amount to be charged for billing period (required number).
    *   `averageUsagePerMember`: Average resource usage per member (required number).
    *   `billingPeriodStart`: Date and time when the current billing period started (string or null).
    *   `billingPeriodEnd`: Date and time when the current billing period ends (string or null).
    *   `members`: An array of `MemberUsageData` objects, providing usage details for each member (optional).
    *   `userRole`: Role of the currently logged in user (optional string).
    *   `billingBlocked`: Boolean flag to indicate if the billing is blocked (optional boolean).

11. **`OrganizationState`**

    ```typescript
    export interface OrganizationState {
      // Core organization data
      organizations: Organization[]
      activeOrganization: Organization | null

      // Team management
      subscriptionData: Subscription | null
      userWorkspaces: Workspace[]

      // Organization billing and usage
      organizationBillingData: OrganizationBillingData | null

      // Organization settings
      orgFormData: OrganizationFormData

      // Loading states
      isLoading: boolean
      isLoadingSubscription: boolean
      isLoadingOrgBilling: boolean
      isCreatingOrg: boolean
      isInviting: boolean
      isSavingOrgSettings: boolean

      // Error states
      error: string | null
      orgSettingsError: string | null

      // Success states
      inviteSuccess: boolean
      orgSettingsSuccess: string | null

      // Cache timestamps
      lastFetched: number | null
      lastSubscriptionFetched: number | null
      lastOrgBillingFetched: number | null

      // User permissions
      hasTeamPlan: boolean
      hasEnterprisePlan: boolean
    }
    ```

    *   Represents the overall state of the organization-related data in the application. This is what would likely be managed by a state management library.
    *   `organizations`: Array of all `Organization` objects the user has access to.
    *   `activeOrganization`: The currently selected `Organization` object (or `null` if none is selected).
    *   `subscriptionData`: The `Subscription` object for the active organization (or `null` if no subscription exists).
    *   `userWorkspaces`: An array of `Workspace` objects that the current user is a member of.
    *   `organizationBillingData`: The `OrganizationBillingData` object for the active organization (or `null` if billing data is not available).
    *   `orgFormData`: The data currently in the organization form.
    *   **Loading States:** Boolean flags indicating whether various operations are in progress (e.g., `isLoading`, `isLoadingSubscription`).  These are used to show loading indicators in the UI.
    *   **Error States:**  Strings holding error messages if any operation fails (e.g., `error`, `orgSettingsError`).
    *   **Success States:** Boolean flags or strings indicating whether certain operations were successful (e.g., `inviteSuccess`, `orgSettingsSuccess`).
    *   **Cache Timestamps:**  Numbers representing the last time data was fetched from the server.  Used for caching strategies.
    *   **User Permissions:** Boolean flags indicating which subscription plan the user is on, and associated permissions.

12. **`OrganizationStore`**

    ```typescript
    export interface OrganizationStore extends OrganizationState {
      loadData: () => Promise<void>
      loadOrganizationSubscription: (orgId: string) => Promise<void>
      loadOrganizationBillingData: (organizationId: string, force?: boolean) => Promise<void>
      loadUserWorkspaces: (userId?: string) => Promise<void>
      refreshOrganization: () => Promise<void>

      // Organization management
      createOrganization: (name: string, slug: string) => Promise<void>
      setActiveOrganization: (orgId: string) => Promise<void>
      updateOrganizationSettings: () => Promise<void>

      // Team management
      inviteMember: (email: string, workspaceInvitations?: WorkspaceInvitation[]) => Promise<void>
      removeMember: (memberId: string, shouldReduceSeats?: boolean) => Promise<void>
      cancelInvitation: (invitationId: string) => Promise<void>

      // Seat management
      addSeats: (newSeatCount: number) => Promise<void>
      reduceSeats: (newSeatCount: number) => Promise<void>

      transferSubscriptionToOrganization: (orgId: string) => Promise<void>

      getUserRole: (userEmail?: string) => string
      isAdminOrOwner: (userEmail?: string) => boolean
      getUsedSeats: () => { used: number; members: number; pending: number }

      setOrgFormData: (data: Partial<OrganizationFormData>) => void

      clearError: () => void
      clearSuccessMessages: () => void
    }
    ```

    *   Extends the `OrganizationState` interface and defines the *actions* (methods) that can be performed to manipulate the organization data.  This is the interface that a state management library (like Zustand, Redux, or Vuex) would implement.
    *   It *inherits* all the properties of `OrganizationState`.
    *   It defines a set of asynchronous functions (using `Promise`) for:
        *   **Data Loading:**  `loadData`, `loadOrganizationSubscription`, `loadOrganizationBillingData`, `loadUserWorkspaces`, `refreshOrganization` (fetches organization data from an API or other data source).
        *   **Organization Management:** `createOrganization`, `setActiveOrganization`, `updateOrganizationSettings`.
        *   **Team Management:** `inviteMember`, `removeMember`, `cancelInvitation`.
        *   **Seat Management:** `addSeats`, `reduceSeats`.
        *   **Subscription Transfer:** `transferSubscriptionToOrganization`.
        *   **User Role and Permissions:** `getUserRole`, `isAdminOrOwner`, `getUsedSeats`.
        *   **Form Management:** `setOrgFormData`.
        *   **Error/Success Message Handling:** `clearError`, `clearSuccessMessages`.
    *   The methods provide a high-level API for interacting with the organization data, abstracting away the underlying data fetching and manipulation logic.

**Simplifying Complex Logic**

While this code primarily defines data structures, here are a few points on simplifying logic, assuming this is a store implementation:

*   **Selectors/Getters:** Instead of accessing deeply nested properties directly in the UI, create selectors (or getters in some state management libraries) within the store.  For example:

    ```typescript
    // Within the OrganizationStore implementation (not the interface)
    const getActiveOrganizationName = (state: OrganizationState) => state.activeOrganization?.name || 'No Organization Selected';
    ```

    This centralizes the logic for accessing the organization name and provides a fallback if no organization is selected.

*   **Action Creators/Reducers (if using Redux):** In Redux, keep reducers pure and focused on updating the state based on actions.  Move complex logic (e.g., data transformation, API calls) into action creators or thunks.

*   **Splitting Up the Store (if necessary):** If the `OrganizationStore` becomes too large and complex, consider breaking it down into smaller, more manageable stores (e.g., a `BillingStore`, a `TeamStore`, etc.).

*   **Using Utility Functions:** Create reusable utility functions to handle common tasks like:

    *   Formatting dates.
    *   Calculating percentages.
    *   Validating data.

**In Summary**

This code defines a comprehensive set of TypeScript interfaces and a store interface for managing organizations, teams, billing, and user roles. It provides a clear and type-safe way to represent and interact with these entities in a modern web application, especially when used in conjunction with a state management library.
