This TypeScript file is a comprehensive collection of utilities designed to facilitate **testing** in a project that likely deals with complex workflow execution, possibly involving a visual workflow builder or a backend process orchestrator.

Its primary purpose is to:

1.  **Create Mock Objects:** Provide factory functions for generating mock versions of various core components like block handlers, store modules, and executor managers (e.g., `PathTracker`, `LoopManager`, `ParallelManager`). These mocks are crucial for isolating the code under test and controlling its dependencies during unit and integration tests.
2.  **Define Test Workflows:** Offer pre-configured `SerializedWorkflow` objects that represent different workflow structures (minimal, with conditions, loops, parallels, errors) to use as consistent test data.
3.  **Generate Mock Execution Contexts:** Allow for the creation of `MockContext` objects, which simulate the state of a workflow execution, enabling precise control over testing scenarios.

In essence, this file provides a "test harness" – a set of tools to set up realistic yet controlled environments for testing the application's core logic related to workflow execution. It heavily leverages `vitest`'s mocking capabilities.

---

### Understanding Vitest Mocks (A Quick Primer)

Before diving into the code, it's essential to understand a few `vitest` concepts used extensively here:

*   `vi.fn()`: Creates a "mock function." This function records how many times it was called, with what arguments, and allows you to define its return value or behavior.
*   `mockImplementation((...args) => { ... })`: Specifies the actual implementation for a mock function. This lets you define custom logic for the mock.
*   `mockReturnValue(value)`: Configures a mock function to return a specific `value` every time it's called.
*   `mockResolvedValue(value)`: Configures an *async* mock function to return a promise that resolves with `value`.
*   `vi.doMock(modulePath, () => ({ ... }))`: This is a powerful feature. It tells Vitest to replace a real module (e.g., `src/executor/handlers.ts`) with a completely custom mock implementation *before* any test code imports that module. This allows you to control the behavior of external dependencies.

---

### Detailed Code Explanation

Let's break down the file section by section.

```typescript
import { vi } from 'vitest'
import type { SerializedWorkflow } from '@/serializer/types'
```

*   `import { vi } from 'vitest'`: Imports the `vi` object from `vitest`, which provides all the mocking utilities like `vi.fn()`, `vi.doMock()`, etc.
*   `import type { SerializedWorkflow } from '@/serializer/types'`: Imports the TypeScript `type` definition for `SerializedWorkflow`. This is used for type safety when defining workflow objects. The `@/` alias indicates an absolute path within the project.

---

### Handler Mock Factory

This section defines how to create mock "block handlers." In a workflow system, a "handler" is responsible for knowing if it can process a specific type of block and then executing that block's logic.

```typescript
/**
 * Mock handler factory - creates consistent handler mocks
 */
export const createMockHandler = (
  handlerName: string,
  options?: {
    canHandleCondition?: (block: any) => boolean
    executeResult?: any | ((inputs: any) => any)
  }
) => {
  const defaultCanHandle = (block: any) =>
    block.metadata?.id === handlerName || handlerName === 'generic'

  const defaultExecuteResult = {
    result: `${handlerName} executed`,
  }

  return vi.fn().mockImplementation(() => ({
    canHandle: options?.canHandleCondition || defaultCanHandle,
    execute: vi.fn().mockImplementation(async (block, inputs) => {
      if (typeof options?.executeResult === 'function') {
        return options.executeResult(inputs)
      }
      return options?.executeResult || defaultExecuteResult
    }),
  }))
}
```

*   `export const createMockHandler = (...) => { ... }`: This is a factory function that generates a mock handler object.
    *   `handlerName: string`: The unique identifier for the handler (e.g., 'trigger', 'agent'). This is used for default `canHandle` logic.
    *   `options?`: An optional object to customize the mock's behavior.
        *   `canHandleCondition?: (block: any) => boolean`: A custom function to determine if this handler can process a given `block`. If not provided, a default is used.
        *   `executeResult?: any | ((inputs: any) => any)`: The result the `execute` method should return. It can be a static value or a function that takes `inputs` and returns a value. If not provided, a default is used.

*   `const defaultCanHandle = (block: any) => ...`: A helper function defining the default logic for `canHandle`. It returns `true` if the `block`'s `metadata.id` matches the `handlerName`, or if the `handlerName` itself is 'generic' (a catch-all).
*   `const defaultExecuteResult = { result: `${handlerName} executed`, }`: A default object that the `execute` method will return if no custom `executeResult` is provided in the options.
*   `return vi.fn().mockImplementation(() => ({ ... }))`: This is the core of the mock handler.
    *   `vi.fn()`: Creates a mock function for the *entire handler instance*. This allows tests to check if a handler was instantiated.
    *   `.mockImplementation(() => ({ ... }))`: Defines what the mock handler *instance* itself looks like. It returns an object with two properties:
        *   `canHandle`: This is either the `options.canHandleCondition` (if provided) or `defaultCanHandle`. This method determines if a block can be processed by this handler.
        *   `execute`: This is another `vi.fn().mockImplementation()`, creating a mock for the `execute` method itself.
            *   `async (block, inputs) => { ... }`: The asynchronous mock implementation of the `execute` method.
            *   `if (typeof options?.executeResult === 'function') { return options.executeResult(inputs) }`: If a function was provided for `executeResult` in options, it's called with the `inputs` and its result is returned. This allows dynamic execution results.
            *   `return options?.executeResult || defaultExecuteResult`: Otherwise, it returns the static `executeResult` from options (if present) or the `defaultExecuteResult`.

---

### Setup Handler Mocks

This function uses `createMockHandler` to replace the actual block handler implementations with mock versions during tests.

```typescript
/**
 * Setup all handler mocks with default behaviors
 */
export const setupHandlerMocks = () => {
  vi.doMock('@/executor/handlers', () => ({
    TriggerBlockHandler: createMockHandler('trigger', {
      canHandleCondition: (block) =>
        block.metadata?.category === 'triggers' || block.config?.params?.triggerMode === true,
      executeResult: (inputs: any) => inputs || {},
    }),
    AgentBlockHandler: createMockHandler('agent'),
    RouterBlockHandler: createMockHandler('router'),
    // ... (other handlers)
    ResponseBlockHandler: createMockHandler('response'),
  }))
}
```

*   `export const setupHandlerMocks = () => { ... }`: A function to set up all default handler mocks.
*   `vi.doMock('@/executor/handlers', () => ({ ... }))`: This tells Vitest to replace the entire `src/executor/handlers` module with the object provided in the second argument. This means that whenever a test imports `TriggerBlockHandler` (or any other handler) from this path, it will get the mock version created here.
    *   Inside the `() => ({ ... })` object, each property corresponds to an exported handler from the real module. For example, `TriggerBlockHandler` is replaced with a mock created by `createMockHandler('trigger', { ... })`.
        *   `TriggerBlockHandler`: This specific handler gets a custom `canHandleCondition` (it handles blocks with `category: 'triggers'` or `triggerMode: true`) and its `executeResult` simply returns the `inputs` it received (or an empty object).
        *   Other handlers like `AgentBlockHandler`, `RouterBlockHandler`, etc., use `createMockHandler` with just their `handlerName`, meaning they'll use the default `canHandle` and `executeResult` behavior (e.g., `AgentBlockHandler` will `canHandle` blocks with `metadata.id === 'agent'` and `execute` will return `{ result: 'agent executed' }`).

---

### Setup Store Mocks

This section provides mock implementations for various Zustand (or similar state management) stores used in the application.

```typescript
/**
 * Setup store mocks with configurable options
 */
export const setupStoreMocks = (options?: {
  isDebugModeEnabled?: boolean
  consoleAddFn?: ReturnType<typeof vi.fn>
  consoleUpdateFn?: ReturnType<typeof vi.fn>
}) => {
  const consoleAddFn = options?.consoleAddFn || vi.fn()
  const consoleUpdateFn = options?.consoleUpdateFn || vi.fn()

  vi.doMock('@/stores/settings/general/store', () => ({
    useGeneralStore: {
      getState: () => ({
        isDebugModeEnabled: options?.isDebugModeEnabled ?? false,
      }),
    },
  }))

  vi.doMock('@/stores/execution/store', () => ({
    useExecutionStore: {
      getState: () => ({
        setIsExecuting: vi.fn(),
        reset: vi.fn(),
        setActiveBlocks: vi.fn(),
        setPendingBlocks: vi.fn(),
        setIsDebugging: vi.fn(),
      }),
      setState: vi.fn(),
    },
  }))

  vi.doMock('@/stores/console/store', () => ({
    useConsoleStore: {
      getState: () => ({
        addConsole: consoleAddFn,
      }),
    },
  }))

  vi.doMock('@/stores/panel/console/store', () => ({
    useConsoleStore: {
      getState: () => ({
        addConsole: consoleAddFn,
        updateConsole: consoleUpdateFn,
      }),
    },
  }))

  return { consoleAddFn, consoleUpdateFn }
}
```

*   `export const setupStoreMocks = (options?: { ... }) => { ... }`: A function to set up mocks for application stores.
    *   `options?`: Allows custom configuration for debug mode and specific console functions.
        *   `isDebugModeEnabled?: boolean`: Controls the mock `isDebugModeEnabled` state.
        *   `consoleAddFn?`, `consoleUpdateFn?`: Allows providing pre-existing mock functions for `addConsole` and `updateConsole` to inspect their calls in tests.
*   `const consoleAddFn = options?.consoleAddFn || vi.fn()`: If `consoleAddFn` is provided in options, use it; otherwise, create a new simple `vi.fn()`.
*   `const consoleUpdateFn = options?.consoleUpdateFn || vi.fn()`: Same for `consoleUpdateFn`.
*   `vi.doMock('@/stores/settings/general/store', () => ({ ... }))`: Mocks the general settings store.
    *   `useGeneralStore`: Replaces the `useGeneralStore` object.
    *   `getState: () => ({ isDebugModeEnabled: options?.isDebugModeEnabled ?? false })`: The `getState` method returns an object where `isDebugModeEnabled` is based on the `options` or defaults to `false`.
*   `vi.doMock('@/stores/execution/store', () => ({ ... }))`: Mocks the execution store.
    *   `useExecutionStore`: Replaces the `useExecutionStore` object.
    *   `getState: () => ({ ... })`: Its `getState` method returns an object containing mock functions for various execution-related actions (`setIsExecuting`, `reset`, etc.), allowing tests to verify if these actions were called.
    *   `setState: vi.fn()`: Mocks the `setState` method for the execution store.
*   `vi.doMock('@/stores/console/store', () => ({ ... }))`: Mocks the console store, returning the `consoleAddFn` for its `addConsole` method.
*   `vi.doMock('@/stores/panel/console/store', () => ({ ... }))`: Mocks another console store path (perhaps for a specific panel), providing both `consoleAddFn` and `consoleUpdateFn`.
*   `return { consoleAddFn, consoleUpdateFn }`: Returns the mock console functions so that tests can assert their call count and arguments.

---

### Setup Executor Core Mocks

This section mocks critical components of the workflow executor, such as path tracking, input resolution, and loop/parallel management.

```typescript
/**
 * Setup core executor mocks (PathTracker, InputResolver, LoopManager, ParallelManager)
 */
export const setupExecutorCoreMocks = () => {
  vi.doMock('@/executor/path', () => ({
    PathTracker: vi.fn().mockImplementation(() => ({
      updateExecutionPaths: vi.fn(),
      isInActivePath: vi.fn().mockReturnValue(true),
    })),
  }))

  vi.doMock('@/executor/resolver', () => ({
    InputResolver: vi.fn().mockImplementation(() => ({
      resolveInputs: vi.fn().mockReturnValue({}),
      resolveBlockReferences: vi.fn().mockImplementation((value) => value),
      resolveVariableReferences: vi.fn().mockImplementation((value) => value),
      resolveEnvVariables: vi.fn().mockImplementation((value) => value),
    })),
  }))

  vi.doMock('@/executor/loops', () => ({
    LoopManager: vi.fn().mockImplementation(() => ({
      processLoopIterations: vi.fn().mockResolvedValue(false),
      getLoopIndex: vi.fn().mockImplementation((loopId, blockId, context) => {
        return context.loopIterations?.get(loopId) || 0
      }),
    })),
  }))

  vi.doMock('@/executor/parallels', () => ({
    ParallelManager: vi.fn().mockImplementation(() => ({
      processParallelIterations: vi.fn().mockResolvedValue(false),
      createVirtualBlockInstances: vi.fn().mockReturnValue([]),
      setupIterationContext: vi.fn(),
      storeIterationResult: vi.fn(),
      initializeParallel: vi.fn(),
      getIterationItem: vi.fn(),
      areAllVirtualBlocksExecuted: vi.fn().mockReturnValue(false),
    })),
  }))
}
```

*   `export const setupExecutorCoreMocks = () => { ... }`: A function to set up mocks for core executor modules.
*   `vi.doMock('@/executor/path', () => ({ ... }))`: Mocks the `PathTracker` module.
    *   `PathTracker: vi.fn().mockImplementation(() => ({ ... }))`: Replaces the `PathTracker` class constructor. When `new PathTracker()` is called in tests, it returns an object with mock methods.
        *   `updateExecutionPaths: vi.fn()`: A simple mock function.
        *   `isInActivePath: vi.fn().mockReturnValue(true)`: Always returns `true`, simulating that all blocks are in the active execution path by default.
*   `vi.doMock('@/executor/resolver', () => ({ ... }))`: Mocks the `InputResolver` module.
    *   `InputResolver: vi.fn().mockImplementation(() => ({ ... }))`: Replaces the `InputResolver` class constructor.
        *   `resolveInputs: vi.fn().mockReturnValue({})`: By default, resolves inputs to an empty object.
        *   `resolveBlockReferences`, `resolveVariableReferences`, `resolveEnvVariables`: Mocked to simply return the `value` they received, effectively doing no resolution by default.
*   `vi.doMock('@/executor/loops', () => ({ ... }))`: Mocks the `LoopManager` module.
    *   `LoopManager: vi.fn().mockImplementation(() => ({ ... }))`: Replaces the `LoopManager` class constructor.
        *   `processLoopIterations: vi.fn().mockResolvedValue(false)`: A mock async function that always resolves to `false`, indicating no further loop iterations are needed by default.
        *   `getLoopIndex: vi.fn().mockImplementation(...)`: Returns the current loop index from the `context.loopIterations` map, or `0` if not found.
*   `vi.doMock('@/executor/parallels', () => ({ ... }))`: Mocks the `ParallelManager` module.
    *   `ParallelManager: vi.fn().mockImplementation(() => ({ ... }))`: Replaces the `ParallelManager` class constructor.
        *   `processParallelIterations: vi.fn().mockResolvedValue(false)`: By default, an async function that resolves to `false`, indicating no parallel iterations are processed.
        *   `createVirtualBlockInstances: vi.fn().mockReturnValue([])`: Returns an empty array of virtual blocks by default.
        *   Other methods (`setupIterationContext`, `storeIterationResult`, `initializeParallel`, `getIterationItem`, `areAllVirtualBlocksExecuted`) are mocked to either do nothing or return `false`.

---

### Workflow Factory Functions

These functions create pre-defined `SerializedWorkflow` objects, serving as test data for various scenarios.

```typescript
/**
 * Workflow factory functions
 */
export const createMinimalWorkflow = (): SerializedWorkflow => ({
  version: '1.0',
  blocks: [
    // ... two blocks: 'starter' and 'block1'
  ],
  connections: [
    { source: 'starter', target: 'block1' },
  ],
  loops: {},
})

export const createWorkflowWithCondition = (): SerializedWorkflow => ({
  version: '1.0',
  blocks: [
    // ... four blocks: 'starter', 'condition1', 'block1' (true path), 'block2' (false path)
  ],
  connections: [
    { source: 'starter', target: 'condition1' },
    { source: 'condition1', target: 'block1', sourceHandle: 'condition-true' }, // Connection for true path
    { source: 'condition1', target: 'block2', sourceHandle: 'condition-false' }, // Connection for false path
  ],
  loops: {},
})

export const createWorkflowWithLoop = (): SerializedWorkflow => ({
  version: '1.0',
  blocks: [
    // ... three blocks: 'starter', 'block1', 'block2'
  ],
  connections: [
    { source: 'starter', target: 'block1' },
    { source: 'block1', target: 'block2' },
    { source: 'block2', target: 'block1' }, // Creates a loop
  ],
  loops: {
    loop1: {
      id: 'loop1',
      nodes: ['block1', 'block2'], // Blocks within the loop
      iterations: 5,
      loopType: 'forEach',
      forEachItems: [1, 2, 3, 4, 5],
    },
  },
})

export const createWorkflowWithErrorPath = (): SerializedWorkflow => ({
  version: '1.0',
  blocks: [
    // ... four blocks: 'starter', 'block1' (function), 'error-handler', 'success-block'
  ],
  connections: [
    { source: 'starter', target: 'block1' },
    { source: 'block1', target: 'success-block', sourceHandle: 'source' }, // Normal success path
    { source: 'block1', target: 'error-handler', sourceHandle: 'error' }, // Error path
  ],
  loops: {},
})

export const createWorkflowWithParallel = (distribution?: any): SerializedWorkflow => ({
  version: '2.0',
  blocks: [
    // ... four blocks: 'starter', 'parallel-1', 'function-1', 'endpoint'
  ],
  connections: [
    { source: 'starter', target: 'parallel-1' },
    { source: 'parallel-1', target: 'function-1', sourceHandle: 'parallel-start-source' }, // Connection into parallel sub-workflow
    { source: 'parallel-1', target: 'endpoint', sourceHandle: 'parallel-end-source' }, // Connection out of parallel sub-workflow
  ],
  loops: {},
  parallels: {
    'parallel-1': {
      id: 'parallel-1',
      nodes: ['function-1'], // Block(s) that run in parallel
      distribution: distribution || ['apple', 'banana', 'cherry'], // Items to distribute across parallel executions
    },
  },
})

export const createWorkflowWithResponse = (): SerializedWorkflow => ({
  version: '1.0',
  blocks: [
    // ... two blocks: 'starter', 'response'
  ],
  connections: [{ source: 'starter', target: 'response' }],
  loops: {},
})
```

These functions are straightforward: they return a `SerializedWorkflow` object, which is a plain JavaScript object conforming to a specific structure (defined by the `SerializedWorkflow` type).

*   Each function creates a workflow with different block arrangements, connections, and metadata to represent distinct execution paths or features (e.g., `createWorkflowWithCondition` shows how conditional routing might look with `sourceHandle` properties on connections).
*   `createWorkflowWithLoop` explicitly defines a `loops` object, specifying which blocks are part of `loop1` and how many iterations it has.
*   `createWorkflowWithParallel` defines a `parallels` object, indicating the blocks that run in parallel and the `distribution` data.
*   These are used as static test data for integration tests where you need a concrete workflow structure.

---

### Mock Execution Context and Specialized Mocks

This section deals with creating and configuring the "context" of a workflow execution, which holds all runtime state. It also provides more specialized mocks for `LoopManager` and `ParallelManager` for specific testing scenarios.

```typescript
/**
 * Create a mock execution context with customizable options
 */
export interface MockContextOptions {
  workflowId?: string
  loopIterations?: Map<string, number>
  // ... other context properties
  workflow?: SerializedWorkflow
  blockStates?: Map<string, any>
}

export const createMockContext = (options: MockContextOptions = {}) => {
  const workflow = options.workflow || createMinimalWorkflow()

  return {
    workflowId: options.workflowId || 'test-workflow-id',
    blockStates: options.blockStates || new Map(),
    blockLogs: [],
    // ... other default context properties
    workflow, // The actual workflow definition being executed
    completedLoops: options.completedLoops || new Set<string>(),
    parallelExecutions: options.parallelExecutions, // State for parallel executions
    parallelBlockMapping: options.parallelBlockMapping, // Mapping for virtual parallel blocks
    currentVirtualBlockId: options.currentVirtualBlockId, // Current block ID in a virtual parallel context
  }
}
```

*   `export interface MockContextOptions { ... }`: Defines the optional properties that can be passed to `createMockContext` to customize the mock execution context.
*   `export const createMockContext = (options: MockContextOptions = {}) => { ... }`: A factory function to create a mock execution context.
    *   It takes an `options` object to override default values.
    *   `const workflow = options.workflow || createMinimalWorkflow()`: If no `workflow` is provided in options, it defaults to `createMinimalWorkflow()`.
    *   The function returns a large object representing the execution context, pre-populated with default values for `workflowId`, `blockStates`, `blockLogs`, `metadata`, `environmentVariables`, `decisions`, `loopIterations`, `loopItems`, `executedBlocks`, `activeExecutionPath`, `completedLoops`, `parallelExecutions`, `parallelBlockMapping`, and `currentVirtualBlockId`. These properties store the runtime state of a workflow execution.

---

```typescript
/**
 * Mock implementations for testing loops
 */
export const createLoopManagerMock = (options?: {
  processLoopIterationsImpl?: (context: any) => Promise<boolean>
  getLoopIndexImpl?: (loopId: string, blockId: string, context: any) => number
}) => ({
  LoopManager: vi.fn().mockImplementation(() => ({
    processLoopIterations: options?.processLoopIterationsImpl || vi.fn().mockResolvedValue(false),
    getLoopIndex:
      options?.getLoopIndexImpl ||
      vi.fn().mockImplementation((loopId, blockId, context) => {
        return context.loopIterations.get(loopId) || 0
      }),
  })),
})
```

*   `export const createLoopManagerMock = (options?: { ... }) => ({ ... })`: This factory creates a mock `LoopManager` constructor with customizable implementations for its methods.
    *   `processLoopIterationsImpl?`: Allows providing a custom async function for `processLoopIterations`.
    *   `getLoopIndexImpl?`: Allows providing a custom function for `getLoopIndex`.
    *   By default, `processLoopIterations` resolves to `false`, and `getLoopIndex` fetches the index from the `context.loopIterations` map, or `0`. This allows tests to specifically control how loops behave.

---

```typescript
/**
 * Create a parallel execution state object for testing
 */
export const createParallelExecutionState = (options?: {
  parallelCount?: number
  distributionItems?: any[] | Record<string, any> | null
  completedExecutions?: number
  // ... other parallel state properties
}) => ({
  parallelCount: options?.parallelCount ?? 3,
  distributionItems:
    options?.distributionItems !== undefined ? options.distributionItems : ['a', 'b', 'c'],
  completedExecutions: options?.completedExecutions ?? 0,
  executionResults: options?.executionResults ?? new Map<string, any>(),
  activeIterations: options?.activeIterations ?? new Set<number>(),
  currentIteration: options?.currentIteration ?? 1,
  parallelType: options?.parallelType,
})
```

*   `export const createParallelExecutionState = (options?: { ... }) => ({ ... })`: A helper to create a consistent object representing the state of a single parallel execution (e.g., how many items, which ones are done, etc.).
    *   It takes `options` to override default properties like `parallelCount`, `distributionItems`, `completedExecutions`, etc.
    *   This object is typically stored within the `context.parallelExecutions` map.

---

```typescript
/**
 * Mock implementations for testing parallels
 */
export const createParallelManagerMock = (options?: {
  maxChecks?: number
  processParallelIterationsImpl?: (context: any) => Promise<void>
}) => ({
  ParallelManager: vi.fn().mockImplementation(() => {
    const executionCounts = new Map() // Tracks how many times processParallelIterations is called for a given parallelId
    const maxChecks = options?.maxChecks || 2

    return {
      processParallelIterations:
        options?.processParallelIterationsImpl ||
        vi.fn().mockImplementation(async (context) => {
          // COMPLEX LOGIC: Simulates the advancement and completion of parallel execution over multiple calls.
          // It checks if 'virtual blocks' (blocks within a parallel iteration) have been executed.
          // After `maxChecks` calls for a given parallelId, it marks the parallel as completed.
          for (const [parallelId, parallel] of Object.entries(context.workflow?.parallels || {})) {
            if (context.completedLoops.has(parallelId)) {
              continue // Already completed, skip
            }

            const parallelState = context.parallelExecutions?.get(parallelId)
            if (!parallelState || parallelState.currentIteration === 0) {
              continue // No parallel state or not started
            }

            const checkCount = executionCounts.get(parallelId) || 0
            executionCounts.set(parallelId, checkCount + 1)

            if (checkCount >= maxChecks) {
              context.completedLoops.add(parallelId) // Simulate completion after maxChecks
              continue
            }

            let allVirtualBlocksExecuted = true
            const parallelNodes = (parallel as any).nodes || []
            for (const nodeId of parallelNodes) {
              for (let i = 0; i < parallelState.parallelCount; i++) {
                const virtualBlockId = `${nodeId}_parallel_${parallelId}_iteration_${i}`
                if (!context.executedBlocks.has(virtualBlockId)) {
                  allVirtualBlocksExecuted = false // If any virtual block is not executed, it's not complete
                  break
                }
              }
              if (!allVirtualBlocksExecuted) break
            }

            if (allVirtualBlocksExecuted && !context.completedLoops.has(parallelId)) {
              // If all virtual blocks are executed and not already completed,
              // simulate completion by adjusting context paths.
              context.executedBlocks.delete(parallelId)
              context.activeExecutionPath.add(parallelId)

              for (const nodeId of parallelNodes) {
                context.activeExecutionPath.delete(nodeId)
              }
            }
          }
        }),
      createVirtualBlockInstances: vi.fn().mockImplementation((block, parallelId, state) => {
        // Creates mock virtual block IDs for each parallel iteration.
        const instances = []
        for (let i = 0; i < state.parallelCount; i++) {
          instances.push(`${block.id}_parallel_${parallelId}_iteration_${i}`)
        }
        return instances
      }),
      // ... other mocked ParallelManager methods
      areAllVirtualBlocksExecuted: vi
        .fn()
        .mockImplementation((parallelId, parallel, executedBlocks, state, context) => {
          // Simple mock: checks if all expected virtual block IDs exist in executedBlocks.
          for (const nodeId of parallel.nodes) {
            for (let i = 0; i < state.parallelCount; i++) {
              const virtualBlockId = `${nodeId}_parallel_${parallelId}_iteration_${i}`
              if (!executedBlocks.has(virtualBlockId)) {
                return false
              }
            }
          }
          return true
        }),
    }
  }),
})
```

*   `export const createParallelManagerMock = (options?: { ... }) => ({ ... })`: This factory creates a mock `ParallelManager` constructor, crucial for testing parallel workflow execution.
    *   `maxChecks?`: An option to control how many times `processParallelIterations` needs to be called before it simulates completion for a parallel block.
    *   `processParallelIterationsImpl?`: Allows providing a custom implementation for this method.
    *   **Simplified Complex Logic (`processParallelIterations`):** The default implementation for `processParallelIterations` is fairly complex because it simulates the *progression* of a parallel block.
        *   It maintains an `executionCounts` map to track how many times `processParallelIterations` has been called for each parallel block ID.
        *   If a parallel block has been checked `maxChecks` times, it's marked as `completed` in `context.completedLoops`. This simulates a parallel workflow taking multiple "turns" or checks to complete.
        *   It then iterates through the workflow's parallel definitions and their nodes. For each node, it constructs "virtual block IDs" (e.g., `block1_parallel_parallel-1_iteration_0`).
        *   It checks if *all* these virtual blocks for a given parallel iteration have been marked as `executed` in `context.executedBlocks`.
        *   If they are all executed, it simulates the parallel completion by removing the parallel's nodes from the active path and adding the parallel block itself to the active path (so the next block after the parallel can be executed).
    *   `createVirtualBlockInstances`: This mock generates the expected virtual block IDs used internally by parallel processing.
    *   `areAllVirtualBlocksExecuted`: This mock implementation checks if all virtual blocks for a given parallel are present in the `executedBlocks` set, indicating their completion.

---

### Custom Handler & Resolver Mocks for Specific Tests

These mocks provide highly specific behavior for particular block types or resolvers, often to facilitate focused testing of complex features.

```typescript
/**
 * Setup function block handler that executes code
 */
export const createFunctionBlockHandler = vi.fn().mockImplementation(() => ({
  canHandle: (block: any) => block.metadata?.id === 'function',
  execute: vi.fn().mockImplementation(async (block, inputs) => {
    // This mock actually executes JavaScript code provided in the block's inputs.
    return {
      result: inputs.code ? new Function(inputs.code)() : { key: inputs.key, value: inputs.value },
      stdout: '',
    }
  }),
}))
```

*   `export const createFunctionBlockHandler = vi.fn().mockImplementation(() => ({ ... }))`: A mock for the `FunctionBlockHandler`.
    *   `canHandle`: Returns `true` for blocks with `metadata.id === 'function'`.
    *   **`execute` (Simplified Complex Logic):** This is a powerful mock. It takes code from `inputs.code` and executes it using `new Function(...)()`. This allows tests to define JavaScript code directly in a mock function block and assert its output, mimicking how a real function block might execute dynamic code. If no `code` is provided, it returns a simple object from `key`/`value` inputs.

---

```typescript
/**
 * Create a custom parallel block handler for testing
 */
export const createParallelBlockHandler = vi.fn().mockImplementation(() => {
  return {
    canHandle: (block: any) => block.metadata?.id === 'parallel',
    execute: vi.fn().mockImplementation(async (block, inputs, context) => {
      const parallelId = block.id
      const parallel = context.workflow?.parallels?.[parallelId]

      if (!parallel) {
        throw new Error('Parallel configuration not found')
      }

      if (!context.parallelExecutions) {
        context.parallelExecutions = new Map()
      }

      let parallelState = context.parallelExecutions.get(parallelId)

      // COMPLEX LOGIC: This mock mimics the actual ParallelBlockHandler's runtime logic.
      if (!parallelState) {
        // First execution: Initialize parallel state
        // It calculates parallelCount and distributionItems from the workflow's parallel config.
        // It sets up initial state in context.parallelExecutions and context.loopItems.
        // It then activates the 'parallel-start-source' connections, pushing child blocks into the active path.
        // Returns a 'started' message.
      } else {
        // Subsequent executions: Check for completion
        // It iterates through all expected virtual blocks for this parallel.
        // If all virtual blocks are found in context.executedBlocks, it means the parallel iterations are done.
        // If completed, it marks the parallel in context.completedLoops and activates 'parallel-end-source' connections.
        // Returns a 'completed' message.
        // Otherwise, returns a 'waiting' message.
      }
    }),
  }
})
```

*   `export const createParallelBlockHandler = vi.fn().mockImplementation(() => { ... })`: This mock specifically replaces the *handler* for the `parallel` block type itself. Unlike `createParallelManagerMock` which mocks the *manager* of parallel executions, this mocks the *block's own execution logic*.
    *   `canHandle`: Returns `true` for blocks with `metadata.id === 'parallel'`.
    *   **`execute` (Simplified Complex Logic):** This is highly complex because it simulates the *control flow* of a parallel block.
        *   **Initialization Phase (if `!parallelState`):** When the parallel block is encountered for the first time, this `execute` method:
            1.  Reads the `parallel` configuration from the `workflow` in the `context`.
            2.  Calculates `parallelCount` and `distributionItems` based on the `parallel.distribution` configuration.
            3.  Initializes a `parallelState` object (using the structure from `createParallelExecutionState`) and stores it in `context.parallelExecutions`.
            4.  Populates `context.loopItems` with the distribution data for reference by child blocks.
            5.  Crucially, it finds all connections stemming from this parallel block with `sourceHandle: 'parallel-start-source'` and adds their `target` blocks to the `context.activeExecutionPath`, effectively "starting" the parallel sub-workflow.
            6.  Returns a success object indicating the parallel has started.
        *   **Completion Check Phase (if `parallelState` exists):** On subsequent calls to this parallel block's `execute` method (which can happen if the executor re-evaluates it while child blocks are running), it:
            1.  Checks if *all* virtual blocks (e.g., `function-1_parallel_parallel-1_iteration_0`) for all parallel iterations have been marked as `executed` in `context.executedBlocks`.
            2.  If all are completed, it marks the parallel block as `completed` in `context.completedLoops`.
            3.  It then finds connections from the parallel block with `sourceHandle: 'parallel-end-source'` and adds their `target` blocks to `context.activeExecutionPath`, allowing the workflow to continue *after* the parallel section.
            4.  Returns a success object indicating completion or a "waiting" message if not all iterations are done.

---

```typescript
/**
 * Create an input resolver mock that handles parallel references
 */
export const createParallelInputResolver = (distributionData: any) => ({
  InputResolver: vi.fn().mockImplementation(() => ({
    resolveInputs: vi.fn().mockImplementation((block, context) => {
      // This mock specifically resolves inputs for blocks *within* a parallel execution.
      if (block.metadata?.id === 'function') {
        const virtualBlockId = context.currentVirtualBlockId
        if (virtualBlockId && context.parallelBlockMapping) {
          const mapping = context.parallelBlockMapping.get(virtualBlockId)
          if (mapping) {
            // It uses the distributionData and mapping (iterationIndex)
            // to dynamically generate the 'code' for a function block,
            // simulating how <parallel.currentItem> and <parallel.index> would be resolved.
            if (Array.isArray(distributionData)) {
              const currentItem = distributionData[mapping.iterationIndex]
              const currentIndex = mapping.iterationIndex
              return {
                code: `return { item: "${currentItem}", index: ${currentIndex} }`,
              }
            }
            if (typeof distributionData === 'object') {
              const entries = Object.entries(distributionData)
              const [key, value] = entries[mapping.iterationIndex]
              return {
                code: `return { key: "${key}", value: "${value}" }`,
              }
            }
          }
        }
      }
      return {}
    }),
  })),
})
```

*   `export const createParallelInputResolver = (distributionData: any) => ({ ... })`: This mock replaces the `InputResolver` specifically for parallel testing.
    *   `distributionData`: The data (array or object) that the parallel block is distributing.
    *   **`resolveInputs` (Simplified Complex Logic):** When `resolveInputs` is called for a `function` block, it checks if it's currently executing *within a parallel iteration*.
        1.  It uses `context.currentVirtualBlockId` and `context.parallelBlockMapping` to determine the current `iterationIndex` for the parallel.
        2.  Based on this `iterationIndex` and the `distributionData` provided, it constructs a `code` string that simulates resolving `parallel.currentItem` and `parallel.index` into the actual data for that iteration.
        3.  This dynamically generated `code` is then passed to the `createFunctionBlockHandler`'s `execute` method, allowing the test to verify that parallel items are correctly processed.

---

### Specialized Workflow Factories for Parallels

These are additional workflow factories specifically for parallel testing, showing different distribution types.

```typescript
/**
 * Create a workflow with parallel blocks for testing
 */
export const createWorkflowWithParallelArray = (
  items: any[] = ['apple', 'banana', 'cherry']
): SerializedWorkflow => ({
  // ... similar to createWorkflowWithParallel, but specifically for array distribution
  parallels: {
    'parallel-1': {
      id: 'parallel-1',
      nodes: ['function-1'],
      distribution: items, // Array of items
    },
  },
})

/**
 * Create a workflow with parallel blocks for object distribution
 */
export const createWorkflowWithParallelObject = (
  items: Record<string, any> = { first: 'alpha', second: 'beta', third: 'gamma' }
): SerializedWorkflow => ({
  // ... similar to createWorkflowWithParallel, but specifically for object distribution
  parallels: {
    'parallel-1': {
      id: 'parallel-1',
      nodes: ['function-1'],
      distribution: items, // Object with key-value pairs
    },
  },
})
```

*   These are specialized versions of `createWorkflowWithParallel` that explicitly define either an array (`createWorkflowWithParallelArray`) or an object (`createWorkflowWithParallelObject`) for the `distribution` property of the parallel block. This is useful for testing how the parallel executor handles different types of input collections.

---

### Utility Setup Functions

These functions combine multiple mock setups for common testing scenarios.

```typescript
/**
 * Mock all modules needed for parallel tests
 */
export const setupParallelTestMocks = (options?: {
  distributionData?: any
  maxParallelChecks?: number
}) => {
  setupStoreMocks() // Mocks stores
  setupExecutorCoreMocks() // Mocks core executor components

  vi.doMock('@/executor/parallels', () =>
    createParallelManagerMock({
      maxChecks: options?.maxParallelChecks,
    })
  ) // Overrides default ParallelManager mock with a custom one
  vi.doMock('@/executor/loops', () => createLoopManagerMock()) // Overrides default LoopManager mock with a custom one
}
```

*   `export const setupParallelTestMocks = (options?: { ... }) => { ... }`: A convenience function to set up all mocks typically required for parallel-related tests.
    *   It calls `setupStoreMocks()` and `setupExecutorCoreMocks()`.
    *   It then *re-mocks* `@/executor/parallels` and `@/executor/loops` using `createParallelManagerMock` and `createLoopManagerMock` respectively, potentially providing specific options (`maxChecks` for `ParallelManager`). This allows specific parallel testing behavior to be injected, overriding the more generic `setupExecutorCoreMocks` defaults.

---

```typescript
/**
 * Sets up all standard mocks for executor tests
 */
export const setupAllMocks = (options?: {
  isDebugModeEnabled?: boolean
  consoleAddFn?: ReturnType<typeof vi.fn>
  consoleUpdateFn?: ReturnType<typeof vi.fn>
}) => {
  setupHandlerMocks() // Mocks all block handlers
  const storeMocks = setupStoreMocks(options) // Mocks stores, returning console fns
  setupExecutorCoreMocks() // Mocks core executor components

  return storeMocks // Returns console functions for assertion
}
```

*   `export const setupAllMocks = (options?: { ... }) => { ... }`: A general utility function to set up all standard mocks needed for most executor-related tests.
    *   It calls `setupHandlerMocks()`, `setupStoreMocks()`, and `setupExecutorCoreMocks()`.
    *   It returns the `storeMocks` (specifically, the mock `consoleAddFn` and `consoleUpdateFn`) so that tests can easily inspect interactions with the console.

---

This file is a fantastic example of how to use a testing framework like Vitest to thoroughly mock dependencies and create controlled test environments for complex application logic, particularly in a domain like workflow orchestration.