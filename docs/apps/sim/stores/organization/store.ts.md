```typescript
import { create } from 'zustand'
import { devtools } from 'zustand/middleware'
import { client } from '@/lib/auth-client'
import { checkEnterprisePlan } from '@/lib/billing/subscriptions/utils'
import { getEnv, isTruthy } from '@/lib/env'
import { createLogger } from '@/lib/logs/console/logger'
import type {
  OrganizationStore,
  Subscription,
  WorkspaceInvitation,
} from '@/stores/organization/types'
import {
  calculateSeatUsage,
  generateSlug,
  validateEmail,
  validateSlug,
} from '@/stores/organization/utils'

// Purpose of this file:
// This file defines a Zustand store called `useOrganizationStore`.  This store manages the global state related to organizations,
// including organization data, active organization, subscription information, user workspaces, billing data, loading states,
// form data, errors, and success messages. It also provides actions to load, create, update, and manage organizations and their members.

// ---- Imports and Setup ----

// Import Zustand for state management. The `create` function is used to create the store.
import { create } from 'zustand'
// Import `devtools` middleware from Zustand. This allows for debugging the store's state changes in the browser's Redux DevTools.
import { devtools } from 'zustand/middleware'
// Import the `client` object from `@/lib/auth-client`. This client is likely used to make API requests related to organizations, authentication, and subscriptions.
import { client } from '@/lib/auth-client'
// Import `checkEnterprisePlan` function to determine if a given subscription is an Enterprise plan.
import { checkEnterprisePlan } from '@/lib/billing/subscriptions/utils'
// Import `getEnv` and `isTruthy` from `@/lib/env`. These are utility functions to retrieve environment variables and check if they are truthy values, respectively.
import { getEnv, isTruthy } from '@/lib/env'
// Import `createLogger` from `@/lib/logs/console/logger`. This is a custom logger for the store, allowing for better debugging and monitoring of the store's actions.
import { createLogger } from '@/lib/logs/console/logger'
// Import the `OrganizationStore`, `Subscription`, and `WorkspaceInvitation` types from `@/stores/organization/types`.  These types define the shape of the state and data used in the store.
import type {
  OrganizationStore,
  Subscription,
  WorkspaceInvitation,
} from '@/stores/organization/types'
// Import utility functions for the organization store: `calculateSeatUsage`, `generateSlug`, `validateEmail`, and `validateSlug`.
import {
  calculateSeatUsage,
  generateSlug,
  validateEmail,
  validateSlug,
} from '@/stores/organization/utils'

// Create a logger instance specifically for this store.
const logger = createLogger('OrganizationStore')

// Determine if billing is enabled based on the `NEXT_PUBLIC_BILLING_ENABLED` environment variable.
const isBillingEnabled = isTruthy(getEnv('NEXT_PUBLIC_BILLING_ENABLED'))

// Define a cache duration of 30 seconds. This value is used to determine how long the store will use cached data before fetching fresh data from the server.
const CACHE_DURATION = 30 * 1000

// ---- Zustand Store Definition ----

// Create the Zustand store using the `create` function, enhanced with `devtools` middleware.
export const useOrganizationStore = create<OrganizationStore>()(
  devtools(
    (set, get) => ({
      // ---- Initial State ----

      // Array to store the organizations the user belongs to.
      organizations: [],
      // The currently active organization.
      activeOrganization: null,
      // Data related to the organization's subscription.
      subscriptionData: null,
      // Array to store the user's workspaces (where they have admin rights).
      userWorkspaces: [],
      // Billing information specific to the organization.
      organizationBillingData: null,
      // Form data for creating or updating an organization.
      orgFormData: {
        name: '',
        slug: '',
        logo: '',
      },
      // Boolean flags to indicate loading states.
      isLoading: false,
      isLoadingSubscription: false,
      isLoadingOrgBilling: false,
      isCreatingOrg: false,
      isInviting: false,
      isSavingOrgSettings: false,
      // Error messages.
      error: null,
      orgSettingsError: null,
      // Success flags.
      inviteSuccess: false,
      orgSettingsSuccess: null,
      // Timestamps for when data was last fetched, used for caching.
      lastFetched: null,
      lastSubscriptionFetched: null,
      lastOrgBillingFetched: null,
      // Booleans to store user plan types
      hasTeamPlan: false,
      hasEnterprisePlan: false,

      // ---- Actions ----

      // **loadData**: Loads all organization-related data, including organizations, active organization, and subscription information.
      loadData: async () => {
        // If billing is disabled, skip loading data.
        if (!isBillingEnabled) {
          logger.debug('Billing disabled, skipping organization data loading')
          set({
            organizations: [],
            activeOrganization: null,
            hasTeamPlan: false,
            hasEnterprisePlan: false,
            isLoading: false,
            error: null,
            lastFetched: Date.now(),
          })
          return
        }

        // Get the current state.
        const state = get()

        // If data was fetched recently (within CACHE_DURATION), use the cached data.
        if (state.lastFetched && Date.now() - state.lastFetched < CACHE_DURATION) {
          logger.debug('Using cached data')
          return
        }

        // If data is already loading, prevent duplicate requests.
        if (state.isLoading) {
          logger.debug('Data already loading, skipping duplicate request')
          return
        }

        // Set the loading state and clear any existing errors.
        set({ isLoading: true, error: null })

        try {
          // Load organizations, active organization, and user subscription info in parallel
          const [orgsResponse, activeOrgResponse, billingResponse] = await Promise.all([
            client.organization.list(),
            client.organization.getFullOrganization().catch(() => ({ data: null })),
            fetch('/api/billing?context=user'),
          ])

          // Extract data from responses
          const organizations = orgsResponse.data || []
          const activeOrganization = activeOrgResponse.data || null

          // Check billing and assign plan types for user
          let hasTeamPlan = false
          let hasEnterprisePlan = false

          if (billingResponse.ok) {
            const billingResult = await billingResponse.json()
            const billingData = billingResult.data
            hasTeamPlan = billingData.isTeam
            hasEnterprisePlan = billingData.isEnterprise
          }

          // Update the state with the fetched data.
          set({
            organizations,
            activeOrganization,
            hasTeamPlan,
            hasEnterprisePlan,
            isLoading: false,
            error: null,
            lastFetched: Date.now(),
          })

          logger.debug('Organization data loaded successfully', {
            organizationCount: organizations.length,
            activeOrganizationId: activeOrganization?.id,
            hasTeamPlan,
            hasEnterprisePlan,
          })

          // Load subscription data for the active organization
          if (activeOrganization?.id) {
            await get().loadOrganizationSubscription(activeOrganization.id)
          }
        } catch (error) {
          // Handle errors during data fetching.
          const errorMessage =
            error instanceof Error ? error.message : 'Failed to load organization data'
          logger.error('Failed to load organization data', { error })
          set({
            isLoading: false,
            error: errorMessage,
          })
        }
      },

      // **loadOrganizationSubscription**: Loads the subscription data for a specific organization.
      loadOrganizationSubscription: async (orgId: string) => {
        const state = get()

        // Check if subscription data is cached and still valid.
        if (
          state.subscriptionData &&
          state.lastSubscriptionFetched &&
          Date.now() - state.lastSubscriptionFetched < CACHE_DURATION
        ) {
          logger.debug('Using cached subscription data')
          return
        }

        // Prevent duplicate requests if already loading.
        if (state.isLoadingSubscription) {
          logger.debug('Subscription data already loading, skipping duplicate request')
          return
        }

        // Set the loading state.
        set({ isLoadingSubscription: true })

        try {
          logger.info('Loading subscription for organization', { orgId })

          // Fetch subscription data from the API.
          const { data, error } = await client.subscription.list({
            query: { referenceId: orgId },
          })

          // Handle API errors.
          if (error) {
            logger.error('Error fetching organization subscription', { error })
            set({ error: 'Failed to load subscription data' })
            return
          }

          // Find active team or enterprise subscription
          const teamSubscription = data?.find(
            (sub) => sub.status === 'active' && sub.plan === 'team'
          )
          const enterpriseSubscription = data?.find((sub) => checkEnterprisePlan(sub))
          const activeSubscription = enterpriseSubscription || teamSubscription

          // If an active subscription is found, update the state.
          if (activeSubscription) {
            logger.info('Found active subscription', {
              id: activeSubscription.id,
              plan: activeSubscription.plan,
              seats: activeSubscription.seats,
            })
            set({
              subscriptionData: activeSubscription,
              isLoadingSubscription: false,
              lastSubscriptionFetched: Date.now(),
            })
          } else {
            logger.warn('No active subscription found for organization', { orgId })
            set({
              subscriptionData: null,
              isLoadingSubscription: false,
              lastSubscriptionFetched: Date.now(),
            })
          }
        } catch (error) {
          // Handle errors during data fetching.
          logger.error('Error loading subscription data', { error })
          set({
            error: error instanceof Error ? error.message : 'Failed to load subscription data',
            isLoadingSubscription: false,
          })
        }
      },

      // **loadOrganizationBillingData**: Loads the billing data for a specific organization.
      loadOrganizationBillingData: async (organizationId: string, force?: boolean) => {
        const state = get()

        // Check if data is cached and still valid. If `force` is true, skip the cache check.
        if (
          state.organizationBillingData &&
          state.lastOrgBillingFetched &&
          Date.now() - state.lastOrgBillingFetched < CACHE_DURATION &&
          !force
        ) {
          logger.debug('Using cached organization billing data')
          return
        }

        // Prevent duplicate requests if already loading.
        if (state.isLoadingOrgBilling) {
          logger.debug('Organization billing data already loading, skipping duplicate request')
          return
        }

        // Set the loading state.
        set({ isLoadingOrgBilling: true })

        try {
          // Fetch billing data from the API.
          const response = await fetch(`/api/billing?context=organization&id=${organizationId}`)

          // Handle HTTP errors.
          if (!response.ok) {
            throw new Error(`HTTP error! status: ${response.status}`)
          }

          const result = await response.json()
          const data = result.data

          // Update the state with the fetched data.
          set({
            organizationBillingData: { ...data, userRole: result.userRole },
            isLoadingOrgBilling: false,
            lastOrgBillingFetched: Date.now(),
          })

          logger.debug('Organization billing data loaded successfully')
        } catch (error) {
          // Handle errors during data fetching.
          logger.error('Failed to load organization billing data', { error })
          set({ isLoadingOrgBilling: false })
        }
      },

      // **loadUserWorkspaces**: Loads the workspaces where the user has admin permissions.
      loadUserWorkspaces: async (userId?: string) => {
        try {
          // Get all workspaces the user is a member of
          const workspacesResponse = await fetch('/api/workspaces')
          if (!workspacesResponse.ok) {
            logger.error('Failed to fetch workspaces')
            return
          }

          const workspacesData = await workspacesResponse.json()
          const allUserWorkspaces = workspacesData.workspaces || []

          // Filter to only show workspaces where user has admin permissions
          const adminWorkspaces = []

          for (const workspace of allUserWorkspaces) {
            try {
              const permissionResponse = await fetch(`/api/workspaces/${workspace.id}/permissions`)
              if (permissionResponse.ok) {
                const permissionData = await permissionResponse.json()

                // Check if current user has admin permission
                // Use userId if provided, otherwise fall back to checking isOwner from workspace data
                let hasAdminAccess = false

                if (userId && permissionData.users) {
                  const currentUserPermission = permissionData.users.find(
                    (user: any) => user.id === userId || user.userId === userId
                  )
                  hasAdminAccess = currentUserPermission?.permissionType === 'admin'
                }

                // Also check if user is the workspace owner
                const isOwner = workspace.isOwner || workspace.ownerId === userId

                if (hasAdminAccess || isOwner) {
                  adminWorkspaces.push({
                    ...workspace,
                    isOwner: isOwner,
                    canInvite: true,
                  })
                }
              }
            } catch (error) {
              logger.warn(`Failed to check permissions for workspace ${workspace.id}:`, error)
            }
          }

          set({ userWorkspaces: adminWorkspaces })

          logger.info('Loaded admin workspaces for invitation', {
            total: allUserWorkspaces.length,
            adminWorkspaces: adminWorkspaces.length,
            userId: userId || 'not provided',
          })
        } catch (error) {
          logger.error('Failed to load workspaces:', error)
        }
      },

      // **refreshOrganization**: Refreshes the currently active organization's data.
      refreshOrganization: async () => {
        if (!isBillingEnabled) {
          logger.debug('Billing disabled, skipping organization refresh')
          return
        }

        // Get the active organization from the state.
        const { activeOrganization } = get()
        // If there is no active organization, do nothing.
        if (!activeOrganization?.id) return

        try {
          // Fetch the latest organization data from the API.
          const fullOrgResponse = await client.organization.getFullOrganization()
          const updatedOrg = fullOrgResponse.data

          logger.info('Refreshed organization data', {
            orgId: updatedOrg?.id,
            members: updatedOrg?.members?.length ?? 0,
            invitations: updatedOrg?.invitations?.length ?? 0,
            pendingInvitations:
              updatedOrg?.invitations?.filter((inv: any) => inv.status === 'pending').length ?? 0,
          })

          // Update the state with the refreshed data.
          set({ activeOrganization: updatedOrg })

          // Also refresh subscription data
          if (updatedOrg?.id) {
            await get().loadOrganizationSubscription(updatedOrg.id)
          }
        } catch (error) {
          // Handle errors during data fetching.
          logger.error('Failed to refresh organization data', { error })
          set({
            error: error instanceof Error ? error.message : 'Failed to refresh organization data',
          })
        }
      },

      // **createOrganization**: Creates a new organization.
      createOrganization: async (name: string, slug: string) => {
        if (!isBillingEnabled) {
          logger.debug('Billing disabled, skipping organization creation')
          set({
            error: 'Organizations are only available when billing is enabled',
            isCreatingOrg: false,
          })
          return
        }

        // Set the creating state and clear any existing errors.
        set({ isCreatingOrg: true, error: null })

        try {
          logger.info('Creating team organization', { name, slug })

          // Create the organization via the API.
          const result = await client.organization.create({ name, slug })
          // Handle API errors.
          if (!result.data?.id) {
            throw new Error('Failed to create organization')
          }

          const orgId = result.data.id
          logger.info('Organization created', { orgId })

          // Set as active organization
          await client.organization.setActive({ organizationId: orgId })

          // Handle subscription transfer if needed
          const { hasTeamPlan, hasEnterprisePlan } = get()
          if (hasTeamPlan || hasEnterprisePlan) {
            await get().transferSubscriptionToOrganization(orgId)
          }

          // Refresh data
          await get().loadData()

          // Update the state.
          set({ isCreatingOrg: false })
        } catch (error) {
          // Handle errors during organization creation.
          logger.error('Failed to create organization', { error })
          set({
            error: error instanceof Error ? error.message : 'Failed to create organization',
            isCreatingOrg: false,
          })
        }
      },

      // **setActiveOrganization**: Sets the currently active organization.
      setActiveOrganization: async (orgId: string) => {
        // Set the loading state.
        set({ isLoading: true })

        try {
          // Set the active organization via the API.
          await client.organization.setActive({ organizationId: orgId })

          // Load the active organization's full details.
          const activeOrgResponse = await client.organization.getFullOrganization()
          const activeOrganization = activeOrgResponse.data

          // Update the state with the active organization.
          set({ activeOrganization })

          if (activeOrganization?.id) {
            await get().loadOrganizationSubscription(activeOrganization.id)
          }
        } catch (error) {
          // Handle errors.
          logger.error('Failed to set active organization', { error })
          set({
            error: error instanceof Error ? error.message : 'Failed to set active organization',
          })
        } finally {
          // Clear the loading state.
          set({ isLoading: false })
        }
      },

      // **updateOrganizationSettings**: Updates the settings of the currently active organization.
      updateOrganizationSettings: async () => {
        // Get the active organization and form data from the state.
        const { activeOrganization, orgFormData } = get()
        // If there is no active organization, do nothing.
        if (!activeOrganization?.id) return

        // Validate form
        if (!orgFormData.name.trim()) {
          set({ orgSettingsError: 'Organization name is required' })
          return
        }

        if (!orgFormData.slug.trim()) {
          set({ orgSettingsError: 'Organization slug is required' })
          return
        }

        // Validate slug format
        if (!validateSlug(orgFormData.slug)) {
          set({
            orgSettingsError:
              'Slug can only contain lowercase letters, numbers, hyphens, and underscores',
          })
          return
        }

        // Set the saving state and clear any existing errors/success messages.
        set({ isSavingOrgSettings: true, orgSettingsError: null, orgSettingsSuccess: null })

        try {
          // Update the organization settings via the API.
          const response = await fetch(`/api/organizations/${activeOrganization.id}`, {
            method: 'PUT',
            headers: {
              'Content-Type': 'application/json',
            },
            body: JSON.stringify({
              name: orgFormData.name.trim(),
              slug: orgFormData.slug.trim(),
              logo: orgFormData.logo.trim() || null,
            }),
          })

          // Handle API errors.
          if (!response.ok) {
            const errorData = await response.json()
            throw new Error(errorData.error || 'Failed to update organization settings')
          }

          // Set a success message.
          set({ orgSettingsSuccess: 'Organization settings updated successfully' })

          // Refresh organization data
          await get().refreshOrganization()

          // Clear success message after 3 seconds
          setTimeout(() => {
            set({ orgSettingsSuccess: null })
          }, 3000)
        } catch (error) {
          // Handle errors during settings update.
          logger.error('Failed to update organization settings', { error })
          set({
            orgSettingsError: error instanceof Error ? error.message : 'Failed to update settings',
          })
        } finally {
          // Clear the saving state.
          set({ isSavingOrgSettings: false })
        }
      },

      // **inviteMember**: Invites a new member to the organization.
      inviteMember: async (email: string, workspaceInvitations?: WorkspaceInvitation[]) => {
        // Get the active organization and subscription data from the state.
        const { activeOrganization, subscriptionData } = get()
        // If there is no active organization, do nothing.
        if (!activeOrganization) return

        // Set the inviting state and clear any existing errors/success messages.
        set({ isInviting: true, error: null, inviteSuccess: false })

        try {
          // calculate seat usage and determine if invite can occur
          const { used: totalCount } = calculateSeatUsage(activeOrganization)
          const seatLimit = subscriptionData?.seats || 0

          if (totalCount >= seatLimit) {
            throw new Error(
              `You've reached your team seat limit of ${seatLimit}. Please upgrade your plan for more seats.`
            )
          }

          // Validate the email address.
          if (!validateEmail(email)) {
            throw new Error('Please enter a valid email address')
          }

          logger.info('Sending invitation to member', {
            email,
            organizationId: activeOrganization.id,
            workspaceInvitations,
          })

          // Use direct API call with workspace invitations if selected
          if (workspaceInvitations && workspaceInvitations.length > 0) {
            const response = await fetch(
              `/api/organizations/${activeOrganization.id}/invitations?batch=true`,
              {
                method: 'POST',
                headers: {
                  'Content-Type': 'application/json',
                },
                body: JSON.stringify({
                  email,
                  role: 'member',
                  workspaceInvitations,
                }),
              }
            )

            if (!response.ok) {
              const errorData = await response.json()
              throw new Error(errorData.error || 'Failed to send invitation')
            }
          } else {
            // Use existing client method for organization-only invitations
            const inviteResult = await client.organization.inviteMember({
              email,
              role: 'member',
              organizationId: activeOrganization.id,
            })

            if (inviteResult.error) {
              throw new Error(inviteResult.error.message || 'Failed to send invitation')
            }
          }

          // Set a success message.
          set({ inviteSuccess: true })
          // Refresh organization data.
          await get().refreshOrganization()
        } catch (error) {
          // Handle errors during invitation.
          logger.error('Error inviting member', { error })
          set({ error: error instanceof Error ? error.message : 'Failed to invite member' })
        } finally {
          // Clear the inviting state.
          set({ isInviting: false })
        }
      },

      // **removeMember**: Removes a member from the organization.
      removeMember: async (memberId: string, shouldReduceSeats = false) => {
        // Get the active organization and subscription data from the state.
        const { activeOrganization, subscriptionData } = get()
        // If there is no active organization, do nothing.
        if (!activeOrganization) return

        logger.info('Removing member', {
          memberId,
          organizationId: activeOrganization.id,
          shouldReduceSeats,
        })

        // Set the loading state.
        set({ isLoading: true })

        try {
          // Use our custom API endpoint for member removal instead of better-auth client
          const response = await fetch(
            `/api/organizations/${activeOrganization.id}/members/${memberId}`,
            {
              method: 'DELETE',
              headers: {
                'Content-Type': 'application/json',
              },
              credentials: 'include', // Ensure cookies are sent
              body: JSON.stringify({ shouldReduceSeats }),
            }
          )

          if (!response.ok) {
            const data = await response.json().catch(() => ({} as any))
            throw new Error((data as any).error || 'Failed to remove member')
          }

          // If the user opted to reduce seats as well (handled by the API endpoint)
          // No need to call reduceSeats separately as it's handled in the endpoint

          // Refresh organization data.
          await get().refreshOrganization()
        } catch (error) {
          // Handle errors during member removal.
          logger.error('Failed to remove member', { error })
          set({ error: error instanceof Error ? error.message : 'Failed to remove member' })
        } finally {
          // Clear the loading state.
          set({ isLoading: false })
        }
      },

      // **cancelInvitation**: Cancels a pending invitation.
      cancelInvitation: async (invitationId: string) => {
        // Get the active organization from the state.
        const { activeOrganization } = get()
        // If there is no active organization, do nothing.
        if (!activeOrganization) return

        // Set the loading state.
        set({ isLoading: true })

        try {
          const response = await fetch(
            `/api/organizations/${activeOrganization.id}/invitations?invitationId=${encodeURIComponent(
              invitationId
            )}`,
            { method: 'DELETE' }
          )

          if (!response.ok) {
            const data = await response.json().catch(() => ({} as any))

            // If the invitation is not found (404), it might have already been processed
            // Just refresh the organization data to get the latest state
            if (response.status === 404) {
              logger.info(
                'Invitation not found or already processed, refreshing organization data',
                { invitationId }
              )
              await get().refreshOrganization()
              return
            }

            throw new Error((data as any).error || 'Failed to cancel invitation')
          }

          // Refresh organization data.
          await get().refreshOrganization()
        } catch (error) {
          // Handle errors during invitation cancellation.
          logger.error('Failed to cancel invitation', { error })
          set({ error: error instanceof Error ? error.message : 'Failed to cancel invitation' })
        } finally {
          // Clear the loading state.
          set({ isLoading: false })
        }
      },

      // **addSeats**: Adds additional seats to the organization's subscription.
      addSeats: async (newSeatCount: number) => {
        // Get the active organization and subscription data from the state.
        const { activeOrganization, subscriptionData } = get()
        // If there is no active organization or subscription data, do nothing.
        if (!activeOrganization || !subscriptionData) return

        // Set the loading state and clear any existing errors.
        set({ isLoading: true, error: null })

        try {
          // Upgrade subscription via API.
          const { error } = await client.subscription.upgrade({
            plan: 'team',
            referenceId: activeOrganization.id,
            subscriptionId: subscriptionData.id,
            seats: newSeatCount,
            successUrl: window.location.href,
            cancelUrl: window.location.href,
          })

          // Handle API errors.
          if (error) {
            throw new Error(error.message || 'Failed to update seats')
          }

          // Refresh organization data.
          await get().refreshOrganization()
        } catch (error) {
          // Handle errors during seat addition.
          logger.error('Failed to add seats', { error })
          set({ error: error instanceof Error ? error.message : 'Failed to update seats' })
        } finally {
          // Clear the loading state.
          set({ isLoading: false })
        }
      },

      // **reduceSeats**: Reduces the number of seats in the organization's subscription.
      reduceSeats: async (newSeatCount: number) => {
        // Get the active organization and subscription data from the state.
        const { activeOrganization, subscriptionData } = get()
        // If there is no active organization or subscription data, do nothing.
        if (!activeOrganization || !subscriptionData) return

        // Don't allow enterprise users to modify seats
        if (checkEnterprisePlan(subscriptionData)) {
          set({ error: 'Enterprise plan seats can only be modified by contacting support' })
          return
        }

        // Validate that the new seat count is not less than 1
        if (newSeatCount <= 0) {
          set({ error: 'Cannot reduce seats below 1' })
          return
        }

        // Calculate the current seat usage.
        const { used: totalCount } = calculateSeatUsage(activeOrganization)
        // Prevent reducing seats below the current usage.
        if (totalCount > newSeatCount) {
          set({
            error: `You have ${totalCount} active members/invitations. Please remove members or cancel invitations before reducing seats.`,
          })
          return
        }

        // Set the loading state and clear any existing errors.
        set({ isLoading: true, error: null })

        try {
          // Upgrade subscription via API.
          const { error } = await client.subscription.upgrade({
            plan: 'team',
            referenceId: activeOrganization.id,
            subscriptionId: subscriptionData.id,
            seats: newSeatCount,
            successUrl: window.location.href,
            cancelUrl: window.location.href,
          })

          // Handle API errors.
          if (error) {
            throw new Error(error.message || 'Failed to reduce seats')
          }

          // Refresh organization data.
          await get().refreshOrganization()
        } catch (error) {
          // Handle errors during seat reduction.
          logger.error('Failed to reduce seats', { error })
          set({ error: error instanceof Error ? error.message : 'Failed to reduce seats' })
        } finally {
          // Clear the loading state.
          set({ isLoading: false })
        }
      },

      // **transferSubscriptionToOrganization**: Transfers a user's subscription to an organization (private helper method).
      transferSubscriptionToOrganization: async (orgId: string) => {
        const { hasTeamPlan, hasEnterprisePlan } = get()

        try {
          // Get the user's subscription.
          const userSubResponse = await client.subscription.list()
          let teamSubscription: Subscription | null =
            (userSubResponse.data?.find(
              (sub) => (sub.plan === 'team' || sub.plan === 'enterprise') && sub.status === 'active'
            ) as Subscription | undefined) || null

          // If no subscription found through client API but user has enterprise plan
          if (!teamSubscription && hasEnterprisePlan) {
            const billingResponse = await fetch('/api/billing?context=user')
            if (billingResponse.ok) {
              const billingData = await billingResponse.json()
              if (billingData.success && billingData.data.isEnterprise && billingData.data.status) {
                teamSubscription = {
                  id: `subscription_${Date.now()}`,
                  plan: billingData.data.plan,
                  status: billingData.data.status,
                  seats: billingData.data.seats,
                  referenceId: billingData.data.organizationId || 'unknown',
                }
              }
            }
          }

          // If a subscription is found, transfer it to the organization.
          if (teamSubscription) {
            const transferResponse = await fetch(
              `/api/users/me/subscription/${teamSubscription.id}/transfer`,
              {
                method: 'POST',
                headers: { 'Content-Type': 'application/json' },
                body: JSON.stringify({
                  organizationId: orgId,
                }),
              }
            )

            // Handle API errors.
            if (!transferResponse.ok) {
              const errorText = await transferResponse.text()
              let errorMessage = 'Failed to transfer subscription'

              try {
                if (errorText?.trim().startsWith('{')) {
                  const errorData = JSON.parse(errorText)
                  errorMessage = errorData.error || errorMessage
                }
              } catch (_e) {
                errorMessage = errorText || errorMessage
              }

              throw new Error(errorMessage)
            }
          }
        } catch (error) {
          // Handle errors during subscription transfer.
          logger.error('Subscription transfer failed', { error })
          throw error
        }
      },

      // ---- Computed Getters ----

      // **getUserRole**: Returns the role of a user within the active organization.
      getUserRole: (userEmail?: string) => {
        // Get the active organization from the state.
        const { activeOrganization } = get()
        // If there is no email or active organization, return 'member'.
        if (!userEmail || !activeOrganization?.members) {
          return 'member'
        }
        // Find the member with the given email.
        const currentMember = activeOrganization.members.find((m) => m.user?.email === userEmail)
        // Return the member's role, or 'member' if not found.
        return currentMember?.role ?? 'member'
      },

      // **isAdminOrOwner**: Checks if a user is an admin or owner of the active organization.
      isAdminOrOwner: (userEmail?: string) => {
        // Get the user's role.
        const role = get().getUserRole(userEmail)
        // Return true if the role is 'owner' or 'admin'.
        return role === 'owner' || role === 'admin'
      },

      // **getUsedSeats**: Calculates the number of used seats in the active organization.
      getUsedSeats: () => {
        // Get the active organization from the state.
        const { activeOrganization } = get()
        // Calculate and return the seat usage.
        return calculateSeatUsage(activeOrganization)
      },

      // ---- Form Handlers ----

      // **setOrgFormData**: Updates the organization form data in the state.
      setOrgFormData: (data) => {
        set((state) => ({
          orgFormData: { ...state.orgFormData, ...data },
        }))

        // Auto-generate slug from name if name is being set
        if (data.name) {
          const autoSlug = generateSlug(data.name)
          set((state) => ({
            orgFormData: { ...state.orgFormData, slug: autoSlug },
          }))
        }
      },

      // ---- Utility Methods ----

      // **clearError**: Clears the error message in the state.
      clearError: () => {
        set({ error: null })
      },

      // **clearSuccessMessages**: Clears the success messages in the state.
      clearSuccessMessages: () => {
        set({ inviteSuccess: false, orgSettingsSuccess