This file contains a set of custom React hooks designed to interact with and manage data related to a "knowledge base" system. It leverages a global state management store (likely Zustand, given the `useKnowledgeStore` naming convention) to fetch, cache, and update knowledge bases, their documents, and the individual "chunks" of information within those documents.

The core idea is to abstract away the data fetching, loading states, error handling, and even some pagination/search logic, providing reusable components for various parts of a user interface.

Let's break down each part of the code.

---

### **Core Setup and Imports**

```typescript
import { useCallback, useEffect, useMemo, useState } from 'react'
import Fuse from 'fuse.js'
import { createLogger } from '@/lib/logs/console/logger'
import { type ChunkData, type DocumentData, useKnowledgeStore } from '@/stores/knowledge/store'

const logger = createLogger('UseKnowledgeBase')
```

*   **`import { useCallback, useEffect, useMemo, useState } from 'react'`**: These are standard React hooks used for:
    *   `useState`: Managing component-specific state.
    *   `useEffect`: Performing side effects (like data fetching, subscriptions, manual DOM manipulation) and cleanup.
    *   `useCallback`: Memoizing functions to prevent unnecessary re-renders when passed down to child components.
    *   `useMemo`: Memoizing computed values to avoid expensive recalculations on every render.
*   **`import Fuse from 'fuse.js'`**: `Fuse.js` is a powerful, lightweight fuzzy-search library. It will be used later for client-side search functionality on "chunks" of data, allowing for flexible matching even with typos or partial queries.
*   **`import { createLogger } from '@/lib/logs/console/logger'`**: This imports a utility function to create a logger instance. It's used for structured logging, helping with debugging and monitoring in development and production environments.
*   **`import { type ChunkData, type DocumentData, useKnowledgeStore } from '@/stores/knowledge/store'`**:
    *   `type ChunkData`, `type DocumentData`: These are TypeScript type definitions, likely interfaces, that define the structure of "chunk" and "document" objects, respectively. This ensures type safety when working with this data.
    *   `useKnowledgeStore`: This is the custom hook that provides access to the global knowledge base data store. It's likely a state management solution (like Zustand or Redux) that holds the application's knowledge base data, loading states, and methods to interact with the backend API.
*   **`const logger = createLogger('UseKnowledgeBase')`**: An instance of the logger is created, specifically named `UseKnowledgeBase`. This helps in identifying logs originating from this file.

---

### **1. `useKnowledgeBase(id: string)` Hook**

This hook is responsible for fetching and managing the state of a *single* knowledge base identified by its `id`.

```typescript
export function useKnowledgeBase(id: string) {
  const { getKnowledgeBase, getCachedKnowledgeBase, loadingKnowledgeBases } = useKnowledgeStore()

  const [error, setError] = useState<string | null>(null)

  const knowledgeBase = getCachedKnowledgeBase(id)
  const isLoading = loadingKnowledgeBases.has(id)

  useEffect(() => {
    if (!id || knowledgeBase || isLoading) return

    let isMounted = true

    const loadData = async () => {
      try {
        setError(null) // Clear any previous error
        await getKnowledgeBase(id) // Fetch the knowledge base data
      } catch (err) {
        if (isMounted) { // Only update state if the component is still mounted
          setError(err instanceof Error ? err.message : 'Failed to load knowledge base')
          logger.error(`Failed to load knowledge base ${id}:`, err)
        }
      }
    }

    loadData() // Execute the async loading function

    return () => {
      isMounted = false // Cleanup: component is unmounted, prevent state updates
    }
  }, [id, knowledgeBase, isLoading]) // Dependencies for useEffect

  return {
    knowledgeBase,
    isLoading,
    error,
  }
}
```

*   **`const { getKnowledgeBase, getCachedKnowledgeBase, loadingKnowledgeBases } = useKnowledgeStore()`**: This destructures specific functions and state from the global `useKnowledgeStore`.
    *   `getKnowledgeBase`: An asynchronous function to fetch a knowledge base from the backend and store it in the global state.
    *   `getCachedKnowledgeBase`: A synchronous function to retrieve a knowledge base from the global state's cache.
    *   `loadingKnowledgeBases`: A `Set` (or similar structure) that keeps track of which knowledge base `id`s are currently being loaded.
*   **`const [error, setError] = useState<string | null>(null)`**: A local state variable to store any error message that occurs during data fetching. It's `null` by default, indicating no error.
*   **`const knowledgeBase = getCachedKnowledgeBase(id)`**: This line attempts to retrieve the knowledge base data for the given `id` directly from the global store's cache. If it's already in the cache, it's immediately available.
*   **`const isLoading = loadingKnowledgeBases.has(id)`**: Checks if the current `id` is present in the `loadingKnowledgeBases` set, indicating if the data for this knowledge base is currently being fetched.
*   **`useEffect(() => { ... }, [id, knowledgeBase, isLoading])`**: This `useEffect` hook runs whenever `id`, `knowledgeBase`, or `isLoading` changes. It's responsible for initiating the data fetch if needed.
    *   **`if (!id || knowledgeBase || isLoading) return`**: This is a guard clause.
        *   If `id` is not provided, there's nothing to fetch.
        *   If `knowledgeBase` is already available (from cache), no need to refetch.
        *   If `isLoading` is true, a fetch is already in progress, so avoid duplicate requests.
    *   **`let isMounted = true`**: A flag to track if the component that uses this hook is still mounted. This is a common pattern to prevent setting state on an unmounted component, which can lead to memory leaks or errors.
    *   **`const loadData = async () => { ... }`**: An asynchronous function defined inside the `useEffect` to handle the actual data fetching.
        *   **`setError(null)`**: Clears any previous error state before attempting a new load.
        *   **`await getKnowledgeBase(id)`**: Calls the global store function to fetch the knowledge base.
        *   **`try...catch` block**: Handles potential errors during the `getKnowledgeBase` call.
            *   If an error occurs, `setError` is called with an appropriate message.
            *   **`if (isMounted)`**: The error state is only updated if the component is still mounted, using the `isMounted` flag.
            *   `logger.error(...)`: Logs the error for debugging.
    *   **`loadData()`**: Calls the `loadData` function to immediately start fetching when the `useEffect` conditions are met.
    *   **`return () => { isMounted = false }`**: This is the cleanup function for `useEffect`. When the component unmounts or before the effect runs again, this function sets `isMounted` to `false`, ensuring that no state updates occur after the component is gone.
*   **`return { knowledgeBase, isLoading, error, }`**: The hook returns an object containing the fetched `knowledgeBase` data, its `isLoading` status, and any `error` encountered.

---

### **2. `useKnowledgeBaseDocuments(...)` Hook**

This hook manages the documents associated with a specific knowledge base, including capabilities for server-side pagination, searching, and sorting.

```typescript
// Constants
const DEFAULT_PAGE_SIZE = 50

export function useKnowledgeBaseDocuments(
  knowledgeBaseId: string,
  options?: {
    search?: string
    limit?: number
    offset?: number
    sortBy?: string
    sortOrder?: string
  }
) {
  const { getDocuments, getCachedDocuments, loadingDocuments, updateDocument, refreshDocuments } =
    useKnowledgeStore()

  const [error, setError] = useState<string | null>(null)

  const documentsCache = getCachedDocuments(knowledgeBaseId)
  const isLoading = loadingDocuments.has(knowledgeBaseId)

  // Load documents with server-side pagination, search, and sorting
  const requestLimit = options?.limit || DEFAULT_PAGE_SIZE
  const requestOffset = options?.offset || 0
  const requestSearch = options?.search
  const requestSortBy = options?.sortBy
  const requestSortOrder = options?.sortOrder

  useEffect(() => {
    if (!knowledgeBaseId || isLoading) return

    let isMounted = true

    const loadDocuments = async () => {
      try {
        setError(null)
        await getDocuments(knowledgeBaseId, {
          search: requestSearch,
          limit: requestLimit,
          offset: requestOffset,
          sortBy: requestSortBy,
          sortOrder: requestSortOrder,
        })
      } catch (err) {
        if (isMounted) {
          setError(err instanceof Error ? err.message : 'Failed to load documents')
          logger.error(`Failed to load documents for knowledge base ${knowledgeBaseId}:`, err)
        }
      }
    }

    loadDocuments()

    return () => {
      isMounted = false
    }
  }, [
    knowledgeBaseId,
    isLoading,
    getDocuments, // Dependency: function from store
    requestSearch,
    requestLimit,
    requestOffset,
    requestSortBy,
    requestSortOrder,
  ])

  // Use server-side filtered and paginated results directly
  const documents = documentsCache?.documents || []
  const pagination = documentsCache?.pagination || {
    total: 0,
    limit: requestLimit,
    offset: requestOffset,
    hasMore: false,
  }

  const refreshDocumentsData = useCallback(async () => {
    try {
      setError(null)
      await refreshDocuments(knowledgeBaseId, {
        search: requestSearch,
        limit: requestLimit,
        offset: requestOffset,
        sortBy: requestSortBy,
        sortOrder: requestSortOrder,
      })
    } catch (err) {
      setError(err instanceof Error ? err.message : 'Failed to refresh documents')
      logger.error(`Failed to refresh documents for knowledge base ${knowledgeBaseId}:`, err)
    }
  }, [
    knowledgeBaseId,
    refreshDocuments, // Dependency: function from store
    requestSearch,
    requestLimit,
    requestOffset,
    requestSortBy,
    requestSortOrder,
  ])

  const updateDocumentLocal = useCallback(
    (documentId: string, updates: Partial<DocumentData>) => {
      updateDocument(knowledgeBaseId, documentId, updates)
      logger.info(`Updated document ${documentId} for knowledge base ${knowledgeBaseId}`)
    },
    [knowledgeBaseId, updateDocument] // Dependencies
  )

  return {
    documents,
    pagination,
    isLoading,
    error,
    refreshDocuments: refreshDocumentsData,
    updateDocument: updateDocumentLocal,
  }
}
```

*   **`const DEFAULT_PAGE_SIZE = 50`**: A constant defining the default number of documents to fetch per page.
*   **`export function useKnowledgeBaseDocuments(...)`**: The hook function, accepting `knowledgeBaseId` and an optional `options` object for server-side control over search, pagination, and sorting.
*   **`const { getDocuments, getCachedDocuments, loadingDocuments, updateDocument, refreshDocuments } = useKnowledgeStore()`**: Destructures relevant functions and state from the global store:
    *   `getDocuments`: Fetches a page of documents with specified search/pagination/sort parameters.
    *   `getCachedDocuments`: Retrieves documents from the cache for a given knowledge base.
    *   `loadingDocuments`: A `Set` tracking which knowledge base IDs are currently having their documents loaded.
    *   `updateDocument`: Updates a specific document in the store (and likely the backend).
    *   `refreshDocuments`: Forces a re-fetch of documents, bypassing the cache.
*   **`const [error, setError] = useState<string | null>(null)`**: Local state for error messages.
*   **`const documentsCache = getCachedDocuments(knowledgeBaseId)`**: Retrieves the cached documents and their pagination metadata for the `knowledgeBaseId`.
*   **`const isLoading = loadingDocuments.has(knowledgeBaseId)`**: Checks if documents for this knowledge base are currently loading.
*   **`const requestLimit = ...`, `const requestOffset = ...`, etc.**: These lines extract and default the `options` parameters for limit, offset, search, sort, and sort order. These will be passed to the backend API.
*   **`useEffect(() => { ... }, [...])`**: This `useEffect` handles the initial and subsequent loading of documents whenever relevant dependencies change.
    *   **`if (!knowledgeBaseId || isLoading) return`**: Guards against invalid `id` or redundant fetches if already loading.
    *   **`let isMounted = true`**: Standard flag to prevent state updates on unmounted components.
    *   **`const loadDocuments = async () => { ... }`**: Async function to fetch data.
        *   **`setError(null)`**: Clears previous errors.
        *   **`await getDocuments(knowledgeBaseId, { ... })`**: Calls the store's function to fetch documents, passing all the determined `request` parameters.
        *   **`try...catch`**: Standard error handling with `isMounted` check and logging.
    *   **`loadDocuments()`**: Initiates the fetch.
    *   **`return () => { isMounted = false }`**: Cleanup function.
    *   **Dependencies (`[...]`)**: Includes `knowledgeBaseId`, `isLoading`, `getDocuments` (the function from the store), and all the `request` parameters. This ensures that a re-fetch occurs whenever the knowledge base ID changes, the loading state clears (allowing a retry), or any of the pagination/search/sort parameters change.
*   **`const documents = documentsCache?.documents || []`**: Extracts the actual `documents` array from the cache. If cache is empty, defaults to an empty array.
*   **`const pagination = documentsCache?.pagination || { ... }`**: Extracts pagination metadata from the cache, defaulting to initial values if not present.
*   **`const refreshDocumentsData = useCallback(async () => { ... }, [...])`**:
    *   A memoized callback function to manually trigger a refresh of the documents.
    *   It calls `refreshDocuments` from the store, passing the current search/pagination/sort parameters. This typically bypasses any client-side cache in the store and re-fetches from the server.
    *   Includes error handling and logging.
    *   **Dependencies**: `knowledgeBaseId`, `refreshDocuments`, and all `request` parameters to ensure the refresh uses the currently active filters.
*   **`const updateDocumentLocal = useCallback(...)`**:
    *   A memoized callback function to update a specific document.
    *   It calls the `updateDocument` function from the store, which will handle the actual update (e.g., sending it to the backend and updating the global state).
    *   **Dependencies**: `knowledgeBaseId`, `updateDocument`.
*   **`return { ... }`**: Returns the `documents` list, `pagination` info, `isLoading` state, `error`, and the `refreshDocuments` and `updateDocument` functions for use by the consuming component.

---

### **3. `useKnowledgeBasesList(workspaceId?: string)` Hook**

This hook is responsible for fetching and managing a *list* of knowledge bases, including a retry mechanism for initial loading.

```typescript
export function useKnowledgeBasesList(workspaceId?: string) {
  const {
    getKnowledgeBasesList,
    knowledgeBasesList,
    loadingKnowledgeBasesList,
    knowledgeBasesListLoaded,
    addKnowledgeBase,
    removeKnowledgeBase,
    clearKnowledgeBasesList,
  } = useKnowledgeStore()

  const [error, setError] = useState<string | null>(null)
  const [retryCount, setRetryCount] = useState(0)
  const maxRetries = 3

  useEffect(() => {
    // Only load if we haven't loaded before AND we're not currently loading
    if (knowledgeBasesListLoaded || loadingKnowledgeBasesList) return

    let isMounted = true
    let retryTimeoutId: NodeJS.Timeout | null = null

    const loadData = async (attempt = 0) => {
      // Don't proceed if component is unmounted
      if (!isMounted) return

      try {
        setError(null) // Clear any previous error
        await getKnowledgeBasesList(workspaceId) // Fetch the list of knowledge bases

        // Reset retry count on success
        if (isMounted) {
          setRetryCount(0)
        }
      } catch (err) {
        if (!isMounted) return // Double-check if still mounted after async operation

        const errorMessage = err instanceof Error ? err.message : 'Failed to load knowledge bases'

        // Only set error and retry if we haven't exceeded max retries
        if (attempt < maxRetries) {
          console.warn(`Knowledge bases load attempt ${attempt + 1} failed, retrying...`, err)
          setRetryCount(attempt + 1)

          // Exponential backoff: 1s, 2s, 4s
          const delay = 2 ** attempt * 1000 // Calculates 1s, 2s, 4s delay
          retryTimeoutId = setTimeout(() => {
            if (isMounted) { // Check mounted status again before scheduling next attempt
              loadData(attempt + 1) // Recursively call loadData for next attempt
              logger.warn(`Failed to load knowledge bases list, retrying... ${attempt + 1}`)
            }
          }, delay)
        } else {
          logger.error('All retry attempts failed for knowledge bases list:', err)
          setError(errorMessage) // Set final error after all retries fail
          setRetryCount(maxRetries)
        }
      }
    }

    // Always start from attempt 0
    loadData(0)

    // Cleanup function
    return () => {
      isMounted = false // Component is unmounted
      if (retryTimeoutId) {
        clearTimeout(retryTimeoutId) // Clear any pending retry timeouts
      }
    }
  }, [knowledgeBasesListLoaded, loadingKnowledgeBasesList, getKnowledgeBasesList, workspaceId]) // Dependencies

  const refreshList = async () => {
    try {
      setError(null)
      setRetryCount(0) // Reset retry count for a manual refresh
      clearKnowledgeBasesList() // Clear existing list in store (optional, but ensures fresh fetch)
      await getKnowledgeBasesList(workspaceId)
    } catch (err) {
      const errorMessage = err instanceof Error ? err.message : 'Failed to refresh knowledge bases'
      setError(errorMessage)
      logger.error('Error refreshing knowledge bases list:', err)
    }
  }

  // Force refresh function that bypasses cache and resets everything
  const forceRefresh = async () => {
    setError(null)
    setRetryCount(0)
    clearKnowledgeBasesList()

    // Force reload by clearing cache and loading state directly in the store
    useKnowledgeStore.setState({
      knowledgeBasesList: [],
      loadingKnowledgeBasesList: false,
      knowledgeBasesListLoaded: false, // Reset store's loaded state
    })

    try {
      await getKnowledgeBasesList(workspaceId)
    } catch (err) {
      const errorMessage = err instanceof Error ? err.message : 'Failed to refresh knowledge bases'
      setError(errorMessage)
      logger.error('Error force refreshing knowledge bases list:', err)
    }
  }

  return {
    knowledgeBases: knowledgeBasesList,
    isLoading: loadingKnowledgeBasesList,
    error,
    refreshList,
    forceRefresh,
    addKnowledgeBase, // Functions from store
    removeKnowledgeBase, // Functions from store
    retryCount,
    maxRetries,
  }
}
```

*   **`const { ... } = useKnowledgeStore()`**: Destructures these from the global store:
    *   `getKnowledgeBasesList`: Fetches the list of knowledge bases.
    *   `knowledgeBasesList`: The actual list of knowledge bases from the store.
    *   `loadingKnowledgeBasesList`: A boolean indicating if the list is currently loading.
    *   `knowledgeBasesListLoaded`: A boolean indicating if the list has been successfully loaded at least once.
    *   `addKnowledgeBase`, `removeKnowledgeBase`, `clearKnowledgeBasesList`: Functions to modify the list in the store (and potentially the backend).
*   **`const [error, setError] = useState<string | null>(null)`**: Local state for errors.
*   **`const [retryCount, setRetryCount] = useState(0)`**: Tracks how many times the loading has been retried.
*   **`const maxRetries = 3`**: Defines the maximum number of automatic retries.
*   **`useEffect(() => { ... }, [...])`**: This `useEffect` handles the initial loading of the knowledge bases list, including the retry logic.
    *   **`if (knowledgeBasesListLoaded || loadingKnowledgeBasesList) return`**: Prevents fetching if already loaded successfully or if a fetch is already in progress.
    *   **`let isMounted = true`, `let retryTimeoutId: NodeJS.Timeout | null = null`**: `isMounted` flag for component lifecycle and `retryTimeoutId` to manage scheduled retries.
    *   **`const loadData = async (attempt = 0) => { ... }`**: An async function that attempts to load data, with `attempt` tracking the current retry number.
        *   **`if (!isMounted) return`**: Important check, especially before any state updates or API calls.
        *   **`try...catch`**: Handles loading. On success, `setError(null)` and `setRetryCount(0)`.
        *   **Retry Logic (`if (attempt < maxRetries)`)**:
            *   If retries are available, it logs a warning, increments `retryCount`.
            *   **`const delay = 2 ** attempt * 1000`**: Implements *exponential backoff*. The delay increases with each attempt (1s, 2s, 4s for attempts 0, 1, 2).
            *   **`retryTimeoutId = setTimeout(...)`**: Schedules the next `loadData` call after the calculated delay. The `isMounted` check is crucial here before calling `loadData` again.
            *   **`else`**: If `maxRetries` is exceeded, it logs an error, sets the final `error` state, and updates `retryCount` to `maxRetries`.
    *   **`loadData(0)`**: Starts the loading process from the first attempt.
    *   **Cleanup function**:
        *   **`isMounted = false`**: Marks component as unmounted.
        *   **`if (retryTimeoutId) clearTimeout(retryTimeoutId)`**: Clears any pending `setTimeout` calls to prevent them from firing after the component is gone.
    *   **Dependencies**: `knowledgeBasesListLoaded`, `loadingKnowledgeBasesList` (trigger re-run if these change, e.g., after a refresh), `getKnowledgeBasesList` (function from store), `workspaceId`.
*   **`const refreshList = async () => { ... }`**: A function to manually refresh the list.
    *   It clears errors and retry count, then calls `clearKnowledgeBasesList()` (to ensure a fresh state) and `getKnowledgeBasesList()`.
*   **`const forceRefresh = async () => { ... }`**: A more aggressive refresh function.
    *   Clears local state (`error`, `retryCount`).
    *   **`useKnowledgeStore.setState(...)`**: Directly manipulates the global `useKnowledgeStore` state, clearing `knowledgeBasesList`, `loadingKnowledgeBasesList`, and importantly, setting `knowledgeBasesListLoaded: false`. This ensures the `useEffect` will re-trigger the initial load logic from scratch.
    *   Then, it calls `getKnowledgeBasesList()` to fetch the data.
*   **`return { ... }`**: Returns the `knowledgeBases` list, `isLoading` status, `error`, the `refreshList` and `forceRefresh` functions, and the `addKnowledgeBase`, `removeKnowledgeBase` functions from the store, along with retry information.

---

### **4. `useDocumentChunks(...)` Hook**

This is the most complex hook, designed to manage the "chunks" of data within a specific document. It provides two distinct modes: client-side search/pagination or server-side search/pagination, determined by the `enableClientSearch` option.

**Simplifying the Core Idea:**
Imagine you have a long article (a document) broken into small paragraphs (chunks).
*   **Server-side mode:** When you search or go to the next page, your request goes to the server, and the server sends back *only* the relevant paragraphs for that specific page/search term. This is efficient for very large numbers of chunks.
*   **Client-side mode:** The hook first fetches *all* paragraphs of the article from the server. Then, when you search or go to the next page, all the filtering and pagination happen directly in your browser using the already-loaded data. This is good for responsiveness if the total number of chunks isn't excessively large, as it avoids repeated server requests after the initial load.

```typescript
/**
 * Hook to manage chunks for a specific document with optional client-side search
 */
export function useDocumentChunks(
  knowledgeBaseId: string,
  documentId: string,
  urlPage = 1, // Current page from URL (for server-side mode)
  urlSearch = '', // Search query from URL (for server-side mode)
  options: { enableClientSearch?: boolean } = {} // Option to enable client-side search
) {
  const { getChunks, refreshChunks, updateChunk, getCachedChunks, clearChunks, isChunksLoading } =
    useKnowledgeStore()

  const { enableClientSearch = false } = options

  // State for both modes
  const [chunks, setChunks] = useState<ChunkData[]>([]) // Currently displayed chunks
  const [allChunks, setAllChunks] = useState<ChunkData[]>([]) // All chunks (used in client-side mode)
  const [isLoading, setIsLoading] = useState(true) // Local loading state
  const [error, setError] = useState<string | null>(null)
  const [pagination, setPagination] = useState({ // Pagination metadata
    total: 0,
    limit: 50,
    offset: 0,
    hasMore: false,
  })
  const [initialLoadDone, setInitialLoadDone] = useState(false) // Tracks if initial load completed
  const [isMounted, setIsMounted] = useState(false) // Tracks component mount status

  // Client-side search state (only used if enableClientSearch is true)
  const [searchQuery, setSearchQuery] = useState('')
  const [currentPage, setCurrentPage] = useState(urlPage)

  // Handle mounting state
  useEffect(() => {
    setIsMounted(true)
    return () => setIsMounted(false)
  }, []) // Runs once on mount and unmount

  // Sync with URL page changes (client-side specific, but impacts initial current page)
  useEffect(() => {
    setCurrentPage(urlPage)
  }, [urlPage])

  const isStoreLoading = isChunksLoading(documentId) // Loading status from global store
  const combinedIsLoading = isLoading || isStoreLoading // Overall loading state

  // --- Client-side search/pagination logic ---
  if (enableClientSearch) {
    const loadAllChunks = useCallback(async () => {
      if (!knowledgeBaseId || !documentId || !isMounted) return

      try {
        setIsLoading(true) // Start local loading state
        setError(null)

        const allChunksData: ChunkData[] = []
        let hasMore = true
        let offset = 0
        const limit = 50 // Fetch in batches

        // Loop to fetch all chunks page by page until 'hasMore' is false
        while (hasMore && isMounted) {
          const response = await fetch(
            `/api/knowledge/${knowledgeBaseId}/documents/${documentId}/chunks?limit=${limit}&offset=${offset}`
          )

          if (!response.ok) {
            throw new Error('Failed to fetch chunks')
          }

          const result = await response.json()

          if (result.success) {
            allChunksData.push(...result.data) // Add fetched chunks
            hasMore = result.pagination.hasMore
            offset += limit
          } else {
            throw new Error(result.error || 'Failed to fetch chunks')
          }
        }

        if (isMounted) {
          setAllChunks(allChunksData) // Store all chunks
          setChunks(allChunksData) // For compatibility, initial display is all chunks
        }
      } catch (err) {
        if (isMounted) {
          setError(err instanceof Error ? err.message : 'Failed to load chunks')
          logger.error(`Failed to load chunks for document ${documentId}:`, err)
        }
      } finally {
        if (isMounted) {
          setIsLoading(false) // End local loading state
        }
      }
    }, [knowledgeBaseId, documentId, isMounted]) // Dependencies for loadAllChunks

    // Load chunks on mount (for client-side mode)
    useEffect(() => {
      if (isMounted) {
        loadAllChunks()
      }
    }, [isMounted, loadAllChunks])

    // Client-side filtering with fuzzy search
    const filteredChunks = useMemo(() => {
      if (!isMounted || !searchQuery.trim()) return allChunks // If no search query, return all

      const fuse = new Fuse(allChunks, { // Initialize Fuse.js
        keys: ['content'], // Search in the 'content' field
        threshold: 0.3, // Lower threshold = more strict matching
        includeScore: true,
        includeMatches: true,
        minMatchCharLength: 2,
        ignoreLocation: true,
      })

      const results = fuse.search(searchQuery) // Perform fuzzy search
      return results.map((result) => result.item) // Return the matched chunk items
    }, [allChunks, searchQuery, isMounted]) // Re-run when allChunks or searchQuery changes

    // Client-side pagination
    const CHUNKS_PER_PAGE = 50
    const totalPages = Math.max(1, Math.ceil(filteredChunks.length / CHUNKS_PER_PAGE))
    const hasNextPage = currentPage < totalPages
    const hasPrevPage = currentPage > 1

    const paginatedChunks = useMemo(() => {
      const startIndex = (currentPage - 1) * CHUNKS_PER_PAGE
      const endIndex = startIndex + CHUNKS_PER_PAGE
      return filteredChunks.slice(startIndex, endIndex) // Slice the filtered chunks for the current page
    }, [filteredChunks, currentPage]) // Re-run when filteredChunks or currentPage changes

    // Reset to page 1 when search changes
    useEffect(() => {
      if (currentPage > 1) { // Only if not already on page 1
        setCurrentPage(1)
      }
    }, [searchQuery]) // Re-run when searchQuery changes

    // Reset to valid page if current page exceeds total
    useEffect(() => {
      if (currentPage > totalPages && totalPages > 0) {
        setCurrentPage(totalPages)
      }
    }, [currentPage, totalPages]) // Re-run when currentPage or totalPages changes

    // Navigation functions (client-side)
    const goToPage = useCallback(
      (page: number) => {
        if (page >= 1 && page <= totalPages) {
          setCurrentPage(page)
        }
      },
      [totalPages]
    )

    const nextPage = useCallback(() => {
      if (hasNextPage) {
        setCurrentPage((prev) => prev + 1)
      }
    }, [hasNextPage])

    const prevPage = useCallback(() => {
      if (hasPrevPage) {
        setCurrentPage((prev) => prev - 1)
      }
    }, [hasPrevPage])

    // Operations (client-side)
    const refreshChunksData = useCallback(async () => {
      await loadAllChunks() // Refresh means re-loading all chunks
    }, [loadAllChunks])

    const updateChunkLocal = useCallback((chunkId: string, updates: Partial<ChunkData>) => {
      // Update local state for both allChunks and currently displayed chunks
      setAllChunks((prev) =>
        prev.map((chunk) => (chunk.id === chunkId ? { ...chunk, ...updates } : chunk))
      )
      setChunks((prev) =>
        prev.map((chunk) => (chunk.id === chunkId ? { ...chunk, ...updates } : chunk))
      )
      // Note: This only updates local state. A separate call to the store's updateChunk
      // might be needed if changes need to persist to backend/global store.
    }, [])

    return {
      // Data
      chunks: paginatedChunks, // The actively displayed chunks
      allChunks, // All chunks fetched
      filteredChunks, // Chunks after search filter
      paginatedChunks, // Chunks after pagination

      // Search
      searchQuery,
      setSearchQuery,

      // Pagination
      currentPage,
      totalPages,
      hasNextPage,
      hasPrevPage,
      goToPage,
      nextPage,
      prevPage,

      // State
      isLoading: combinedIsLoading,
      error,
      pagination: { // Client-side pagination derived from local state
        total: filteredChunks.length,
        limit: CHUNKS_PER_PAGE,
        offset: (currentPage - 1) * CHUNKS_PER_PAGE,
        hasMore: hasNextPage,
      },

      // Operations
      refreshChunks: refreshChunksData,
      updateChunk: updateChunkLocal,
      clearChunks: () => clearChunks(documentId), // From store

      // Legacy compatibility
      searchChunks: async (newSearchQuery: string) => {
        setSearchQuery(newSearchQuery) // Simply updates the search query
        return paginatedChunks // Returns current paginated chunks (will recompute on next render)
      },
    }
  }

  // --- Server-side search/pagination logic (if enableClientSearch is false) ---
  const serverCurrentPage = urlPage // Page from URL parameter
  const serverSearchQuery = urlSearch // Search from URL parameter

  // Computed pagination properties for server-side
  const serverTotalPages = Math.ceil(pagination.total / pagination.limit)
  const serverHasNextPage = serverCurrentPage < serverTotalPages
  const serverHasPrevPage = serverCurrentPage > 1

  // Single effect to handle all data loading and syncing for server-side
  useEffect(() => {
    if (!knowledgeBaseId || !documentId) return

    let isMounted = true

    const loadAndSyncData = async () => {
      try {
        // Check cache first
        const cached = getCachedChunks(documentId)
        const expectedOffset = (serverCurrentPage - 1) * 50 // Assuming 50 is the server-side limit

        if (
          cached &&
          cached.searchQuery === serverSearchQuery &&
          cached.pagination.offset === expectedOffset
        ) {
          // If cached data matches current request, use it
          if (isMounted) {
            setChunks(cached.chunks)
            setPagination(cached.pagination)
            setIsLoading(false)
            setInitialLoadDone(true) // Mark initial load as done
          }
          return
        }

        // Fetch from API if not in cache or cache doesn't match
        setIsLoading(true)
        setError(null)

        const limit = 50
        const offset = (serverCurrentPage - 1) * limit

        const fetchedChunks = await getChunks(knowledgeBaseId, documentId, {
          limit,
          offset,
          search: serverSearchQuery || undefined,
        })

        if (isMounted) {
          setChunks(fetchedChunks) // Update local chunks state

          // Update pagination from cache after fetch (getChunks updates store, then we read from store)
          const updatedCache = getCachedChunks(documentId)
          if (updatedCache) {
            setPagination(updatedCache.pagination)
          }

          setInitialLoadDone(true) // Mark initial load as done
        }
      } catch (err) {
        if (isMounted) {
          setError(err instanceof Error ? err.message : 'Failed to load chunks')
          logger.error(`Failed to load chunks for document ${documentId}:`, err)
        }
      } finally {
        if (isMounted) {
          setIsLoading(false) // End local loading state
        }
      }
    }

    loadAndSyncData() // Initiate loading

    return () => {
      isMounted = false // Cleanup
    }
  }, [
    knowledgeBaseId,
    documentId,
    serverCurrentPage,
    serverSearchQuery,
    // getChunks, // getChunks is implicitly handled as it updates cache
    isStoreLoading, // Re-run if store loading state changes (e.g., after an external update)
    initialLoadDone, // Important: Ensures initial sync happens
    getCachedChunks, // Dependency for reading from cache
  ])

  // Separate effect to sync with store state changes (no API calls)
  useEffect(() => {
    // Only sync if documentId is present AND initial API load has completed
    if (!documentId || !initialLoadDone) return

    const cached = getCachedChunks(documentId)
    const expectedOffset = (serverCurrentPage - 1) * 50

    if (
      cached &&
      cached.searchQuery === serverSearchQuery &&
      cached.pagination.offset === expectedOffset
    ) {
      setChunks(cached.chunks) // Update displayed chunks from store cache
      setPagination(cached.pagination) // Update pagination from store cache
    }

    // Update local loading state based on store's loading state
    if (!isStoreLoading && isLoading) {
      logger.info(`Chunks loaded for document ${documentId}`)
      setIsLoading(false)
    }
  }, [documentId, isStoreLoading, isLoading, initialLoadDone, serverSearchQuery, serverCurrentPage, getCachedChunks]) // Dependencies for syncing

  const goToPage = async (page: number) => {
    if (page < 1 || page > serverTotalPages || page === serverCurrentPage) return // Guard clauses

    try {
      setIsLoading(true)
      setError(null)

      const limit = 50
      const offset = (page - 1) * limit

      // Fetch new page from the backend
      const fetchedChunks = await getChunks(knowledgeBaseId, documentId, {
        limit,
        offset,
        search: serverSearchQuery || undefined,
      })

      // Update local state from cache (getChunks implicitly updates the store)
      const cached = getCachedChunks(documentId)
      if (cached) {
        setChunks(cached.chunks)
        setPagination(cached.pagination)
      }

      return fetchedChunks
    } catch (err) {
      setError(err instanceof Error ? err.message : 'Failed to load page')
      logger.error(`Failed to load page for document ${documentId}:`, err)
      throw err // Re-throw to allow consuming component to handle
    } finally {
      setIsLoading(false)
    }
  }

  const nextPage = () => {
    if (serverHasNextPage) {
      return goToPage(serverCurrentPage + 1)
    }
  }

  const prevPage = () => {
    if (serverHasPrevPage) {
      return goToPage(serverCurrentPage - 1)
    }
  }

  const refreshChunksData = async (options?: {
    search?: string
    limit?: number
    offset?: number
    preservePage?: boolean // Not currently used, but good for future extension
  }) => {
    try {
      setIsLoading(true)
      setError(null)

      const limit = 50
      // If preservePage is true, use current offset, otherwise default to page 1
      const offset = options?.offset ?? (serverCurrentPage - 1) * limit

      // Call store's refreshChunks (forces backend fetch)
      const fetchedChunks = await refreshChunks(knowledgeBaseId, documentId, {
        search: options?.search,
        limit,
        offset,
      })

      // Update local state from cache
      const cached = getCachedChunks(documentId)
      if (cached) {
        setChunks(cached.chunks)
        setPagination(cached.pagination)
      }

      return fetchedChunks
    } catch (err) {
      setError(err instanceof Error ? err.message : 'Failed to refresh chunks')
      logger.error(`Failed to refresh chunks for document ${documentId}:`, err)
      throw err
    } finally {
      setIsLoading(false)
    }
  }

  const searchChunks = async (newSearchQuery: string) => {
    try {
      setIsLoading(true)
      setError(null)

      const limit = 50
      // Always start from page 1 for a new search
      const searchResults = await getChunks(knowledgeBaseId, documentId, {
        search: newSearchQuery,
        limit,
        offset: 0,
      })

      // Update local state from cache
      const cached = getCachedChunks(documentId)
      if (cached) {
        setChunks(cached.chunks)
        setPagination(cached.pagination)
      }

      return searchResults
    } catch (err) {
      setError(err instanceof Error ? err.message : 'Failed to search chunks')
      logger.error(`Failed to search chunks for document ${documentId}:`, err)
      throw err
    } finally {
      setIsLoading(false)
    }
  }

  return {
    // Data (server-side mode, all are the same as chunks)
    chunks,
    allChunks: chunks, // Consistent API: allChunks refers to current paginated results
    filteredChunks: chunks, // Consistent API: filteredChunks refers to current paginated results
    paginatedChunks: chunks, // Consistent API: paginatedChunks refers to current paginated results

    // Search
    searchQuery: urlSearch, // Search query from URL
    setSearchQuery: () => {}, // No-op: client-side search function not applicable here

    isLoading: combinedIsLoading,
    error,
    pagination,
    currentPage: serverCurrentPage,
    totalPages: serverTotalPages,
    hasNextPage: serverHasNextPage,
    hasPrevPage: serverHasPrevPage,
    goToPage,
    nextPage,
    prevPage,
    refreshChunks: refreshChunksData,
    searchChunks,
    updateChunk: (chunkId: string, updates: Partial<ChunkData>) => {
      // Call store's updateChunk
      updateChunk(documentId, chunkId, updates)
      // Also update local state for immediate visual feedback
      setChunks((prevChunks) =>
        prevChunks.map((chunk) => (chunk.id === chunkId ? { ...chunk, ...updates } : chunk))
      )
    },
    clearChunks: () => clearChunks(documentId),
  }
}
```

#### **Common Elements (Applies to both client-side and server-side modes):**

*   **`export function useDocumentChunks(...)`**: Takes `knowledgeBaseId`, `documentId`, `urlPage`, `urlSearch` (for initial state), and `options` (to enable client-side search).
*   **`const { getChunks, ..., isChunksLoading } = useKnowledgeStore()`**: Destructures these from the global store:
    *   `getChunks`: Fetches chunks (will interact with the backend).
    *   `refreshChunks`: Forces a re-fetch of chunks.
    *   `updateChunk`: Updates a specific chunk in the store.
    *   `getCachedChunks`: Retrieves chunks data from the store's cache.
    *   `clearChunks`: Clears chunks from the store for a given document.
    *   `isChunksLoading`: Checks if chunks for a specific document are loading.
*   **Shared State**:
    *   `[chunks, setChunks]`: The list of chunks currently displayed.
    *   `[allChunks, setAllChunks]`: Used in client-side mode to store *all* chunks fetched. In server-side, it will be the same as `chunks`.
    *   `[isLoading, setIsLoading]`: Local loading state for this hook's operations.
    *   `[error, setError]`: Local error state.
    *   `[pagination, setPagination]`: Stores pagination metadata (total, limit, offset, hasMore).
    *   `[initialLoadDone, setInitialLoadDone]`: A flag to ensure certain effects only run after the *initial* data load is complete.
    *   `[isMounted, setIsMounted]`: A boolean to track if the component is mounted, set by a `useEffect` that runs once on mount and cleanup on unmount.
*   **`useEffect(() => { setIsMounted(true); return () => setIsMounted(false) }, [])`**: Sets the `isMounted` flag for the component's lifecycle.
*   **`useEffect(() => { setCurrentPage(urlPage) }, [urlPage])`**: Syncs the internal `currentPage` with the `urlPage` prop, allowing external control of pagination.
*   **`const isStoreLoading = isChunksLoading(documentId)`**: Gets the loading status from the global store for the current `documentId`.
*   **`const combinedIsLoading = isLoading || isStoreLoading`**: Combines the local loading state with the store's loading state to provide a comprehensive `isLoading` value.

#### **Client-Side Mode (`if (enableClientSearch)`):**

This block executes if `options.enableClientSearch` is `true`.
**Concept:** Fetch *all* chunks once, then perform search and pagination locally in the browser.

*   **`const [searchQuery, setSearchQuery] = useState('')`**: Local state for the search input.
*   **`const [currentPage, setCurrentPage] = useState(urlPage)`**: Local state for the current page number.
*   **`const loadAllChunks = useCallback(async () => { ... }, [...])`**:
    *   This is a crucial function: it iteratively fetches *all* chunks for the document from the `/api/knowledge/.../chunks` endpoint, handling the server's pagination internally until `hasMore` is `false`.
    *   It updates `setAllChunks` with the complete list.
    *   Includes `isMounted` checks and error handling.
*   **`useEffect(() => { if (isMounted) loadAllChunks() }, [isMounted, loadAllChunks])`**: Triggers `loadAllChunks` once the component is mounted.
*   **`const filteredChunks = useMemo(() => { ... }, [allChunks, searchQuery, isMounted])`**:
    *   This memoized value performs the fuzzy search using `Fuse.js` on the `allChunks` array.
    *   It re-calculates only when `allChunks` or `searchQuery` changes.
    *   **`new Fuse(allChunks, { keys: ['content'], threshold: 0.3, ... })`**: Initializes `Fuse.js` to search within the `content` field of chunks with a `threshold` (lower = stricter match).
*   **`const CHUNKS_PER_PAGE = 50`**: Constant for client-side pagination.
*   **`const totalPages = ...`, `const hasNextPage = ...`, `const hasPrevPage = ...`**: Derived pagination logic based on the `filteredChunks` and `CHUNKS_PER_PAGE`.
*   **`const paginatedChunks = useMemo(() => { ... }, [filteredChunks, currentPage])`**: Slices the `filteredChunks` array to display only the chunks for the `currentPage`.
*   **`useEffect(() => { ... }, [searchQuery])`**: Resets `currentPage` to 1 whenever the `searchQuery` changes, so the search starts from the beginning.
*   **`useEffect(() => { ... }, [currentPage, totalPages])`**: Adjusts `currentPage` if it becomes invalid (e.g., if filtering reduces the total number of pages).
*   **`goToPage`, `nextPage`, `prevPage`**: `useCallback` functions to manage client-side page navigation by updating `setCurrentPage`.
*   **`refreshChunksData = useCallback(async () => { await loadAllChunks() }, [loadAllChunks])`**: For client-side, refreshing means re-fetching *all* chunks.
*   **`updateChunkLocal = useCallback((chunkId, updates) => { ... }, [])`**: Updates the `allChunks` and `chunks` local state arrays directly. *Note: This does not automatically update the global store or backend. If persistence is needed, the store's `updateChunk` would also need to be called.*
*   **Return value for client-side**: Provides `paginatedChunks` as `chunks`, `allChunks`, `filteredChunks`, search state, pagination details, and local operation functions. The `searchChunks` legacy function just sets the `searchQuery`.

#### **Server-Side Mode (the `else` block):**

This block executes if `options.enableClientSearch` is `false` (default).
**Concept:** Each search or page change triggers a new API request to the server, which returns only the relevant subset of chunks.

*   **`const serverCurrentPage = urlPage`, `const serverSearchQuery = urlSearch`**: These directly use the `urlPage` and `urlSearch` props, meaning the URL controls the server request parameters.
*   **`const serverTotalPages = ...`, `const serverHasNextPage = ...`, `const serverHasPrevPage = ...`**: Derived pagination info based on the `pagination` state (which comes from the server).
*   **`useEffect(() => { ... }, [...])` (Main loading and syncing)**:
    *   This is the primary `useEffect` for fetching data from the server.
    *   **Cache Check**: It first checks `getCachedChunks(documentId)`. If cached data exists and matches the current `serverSearchQuery` and `offset`, it uses the cached data to update local `chunks` and `pagination` state, avoiding a redundant API call.
    *   **API Fetch**: If not cached or cache doesn't match, it sets `isLoading(true)` and calls `await getChunks(...)`, passing `limit`, `offset` (derived from `serverCurrentPage`), and `serverSearchQuery` to the backend.
    *   After `getChunks` returns, it updates local `chunks` and retrieves `pagination` from the *updated* cache (since `getChunks` updates the store).
    *   `setInitialLoadDone(true)`: Marks that the initial data fetch is complete.
    *   **Dependencies**: `knowledgeBaseId`, `documentId`, `serverCurrentPage`, `serverSearchQuery` (crucial for re-fetching on page/search changes), `isStoreLoading`, `initialLoadDone`, `getCachedChunks`.
*   **`useEffect(() => { ... }, [...])` (Store sync effect)**:
    *   This `useEffect` is separate to handle updates originating from the global `useKnowledgeStore` *after* the initial `initialLoadDone` is `true`, without making new API calls.
    *   It re-syncs the local `chunks` and `pagination` from the store's cache if the cached data matches the current request parameters.
    *   It also updates the local `isLoading` state based on `isStoreLoading`.
    *   **Dependencies**: `documentId`, `isStoreLoading`, `isLoading`, `initialLoadDone`, `serverSearchQuery`, `serverCurrentPage`, `getCachedChunks`.
*   **`goToPage = async (page: number) => { ... }`**:
    *   Function to navigate to a specific page. It calls `getChunks` with the new `offset` (calculated from `page`).
    *   Updates local `chunks` and `pagination` from the cache after the fetch.
    *   Includes error handling.
*   **`nextPage`, `prevPage`**: Helper functions that call `goToPage` for the next/previous page.
*   **`refreshChunksData = async (options?) => { ... }`**:
    *   Function to refresh chunks. It calls `refreshChunks` from the store, passing the current search/pagination parameters. This forces a backend re-fetch.
*   **`searchChunks = async (newSearchQuery: string) => { ... }`**:
    *   Function to perform a search. It calls `getChunks` with the `newSearchQuery` and always `offset: 0` (starting from the first page for a new search).
*   **Return value for server-side**:
    *   `chunks`, `allChunks`, `filteredChunks`, `paginatedChunks` all refer to the *same* `chunks` array because the server is already providing filtered/paginated data.
    *   `searchQuery` is `urlSearch` (read-only from URL).
    *   `setSearchQuery: () => {}` is a no-operation function as search is server-driven via `searchChunks`.
    *   Provides `isLoading`, `error`, `pagination`, current page, total pages, navigation functions, and `refreshChunks`, `searchChunks`, `updateChunk`, `clearChunks` operations.
    *   **`updateChunk: (chunkId, updates) => { ... }`**: Calls `updateChunk` from the global store *and* updates the local `chunks` state for immediate visual feedback.

---

### **Key Takeaways and Best Practices**

1.  **Centralized State Management (`useKnowledgeStore`)**: The code heavily relies on a global store to manage data, loading states, and caching. This avoids prop drilling and ensures data consistency across components.
2.  **`isMounted` Flag**: A consistent pattern across all `useEffect` hooks to prevent memory leaks and errors by ensuring state updates only occur if the component is still part of the render tree.
3.  **Error Handling**: Standard `try...catch` blocks are used for asynchronous operations, setting local `error` state and logging to the console.
4.  **`useEffect` Dependency Arrays**: Carefully managed dependency arrays ensure effects run only when necessary, preventing infinite loops or stale closures. For example, store functions (`getKnowledgeBase`, `getDocuments`) are included as dependencies where appropriate.
5.  **Memoization (`useCallback`, `useMemo`)**: Used to optimize performance by preventing unnecessary re-creation of functions (`refreshDocumentsData`, `updateDocumentLocal`) and re-calculation of values (`filteredChunks`, `paginatedChunks`) on every render.
6.  **Clear Loading States**: Each hook provides `isLoading` and `error` states, allowing consuming components to render appropriate UI (spinners, error messages).
7.  **Server-side vs. Client-side Logic**: The `useDocumentChunks` hook demonstrates a robust pattern for implementing both server-driven and client-driven data management strategies conditionally, based on application needs.
    *   **Server-side**: More scalable for very large datasets, offloads processing to the backend. Requires more round-trips for user interactions.
    *   **Client-side**: Provides a snappier user experience after the initial fetch for moderately sized datasets, but can be memory-intensive if all data is too large.
8.  **Retry Mechanism**: The `useKnowledgeBasesList` hook implements an exponential backoff retry strategy for initial data loading, improving resilience to transient network issues.
9.  **Direct Store Manipulation**: The `forceRefresh` function in `useKnowledgeBasesList` shows how to directly manipulate the store's state (`useKnowledgeStore.setState`) to completely reset its loaded status and force a fresh data fetch, useful for administrative actions or deep resets.