This document explains a TypeScript test file (`parallels.test.ts`) that validates the functionality of the `ParallelManager` class. This class is responsible for orchestrating the execution of parallel blocks within a workflow engine.

---

## Explanation: `parallels.test.ts`

This file contains unit tests for the `ParallelManager` class, which is a core component in a workflow execution engine. Its primary role is to manage the lifecycle and state of "parallel" blocks – workflow steps that run multiple iterations simultaneously or concurrently based on a distribution of items.

### Purpose of this File

The main goal of `parallels.test.ts` is to ensure that `ParallelManager` correctly:

1.  **Initializes** parallel execution states.
2.  **Manages** iteration-specific data.
3.  **Tracks** the completion of "virtual" blocks (individual instances of a block running within a parallel iteration).
4.  **Generates** new virtual block instances for unexecuted iterations.
5.  **Stores** results from each parallel iteration.
6.  **Determines** when a parallel block has finished all its iterations and can either be re-executed or marked as fully complete.

By testing each method of `ParallelManager` in isolation, the developers ensure the robustness and correctness of parallel execution logic, which is often one of the more complex parts of a workflow system.

### Simplified Complex Logic

The `ParallelManager` deals with several key concepts:

*   **Parallel Blocks**: These are special workflow blocks that can take a list or object of items (the "distribution") and run a sub-workflow for each item.
*   **Virtual Blocks**: When a parallel block runs, its child blocks (e.g., a "function" block inside a parallel) aren't executed once. Instead, for each item in the distribution, a *virtual instance* of that child block is created and executed. For example, if a parallel block distributes 3 items to a "func-1" block, it will effectively run "func-1" three times, creating virtual IDs like `func-1_parallel_parallel-1_iteration_0`, `func-1_parallel_parallel-1_iteration_1`, etc.
*   **Execution Context**: This is a central object that holds the entire state of a workflow execution, including what blocks have been executed, active paths, loop/parallel specific data, and results. `ParallelManager` interacts heavily with this context to store and retrieve parallel-related information.

The tests simulate various scenarios of parallel execution, checking if the manager correctly updates the execution context, generates correct IDs, and makes the right decisions about execution flow.

### Line-by-Line Explanation

```typescript
import { describe, expect, test, vi } from 'vitest'
import { createParallelExecutionState } from '@/executor/__test-utils__/executor-mocks'
import { BlockType } from '@/executor/consts'
import { ParallelManager } from '@/executor/parallels/parallels'
import type { ExecutionContext } from '@/executor/types'
import type { SerializedWorkflow } from '@/serializer/types'
```

*   **`import { describe, expect, test, vi } from 'vitest'`**: These are imports from `vitest`, a fast unit testing framework for JavaScript/TypeScript.
    *   `describe`: Used to group related tests into a suite.
    *   `expect`: Used to make assertions about values (e.g., `expect(value).toBe(expected)`).
    *   `test`: Defines an individual test case.
    *   `vi`: Vitest's utility for mocking functions and modules.
*   **`import { createParallelExecutionState } from '@/executor/__test-utils__/executor-mocks'`**: Imports a helper function used in tests to easily create a mock `ParallelExecutionState` object, which represents the current state of a parallel block. This avoids needing to manually construct complex state objects in every test.
*   **`import { BlockType } from '@/executor/consts'`**: Imports `BlockType`, an enum (or similar constant) likely defining different types of blocks in the workflow (e.g., `FUNCTION`, `ROUTER`, `CONDITION`).
*   **`import { ParallelManager } from '@/executor/parallels/parallels'`**: This is the core class being tested. It's responsible for managing parallel executions.
*   **`import type { ExecutionContext } from '@/executor/types'`**: Imports the `ExecutionContext` type definition. This is crucial for type checking when creating mock contexts in the tests.
*   **`import type { SerializedWorkflow } from '@/serializer/types'`**: Imports the `SerializedWorkflow` type, which likely represents the structure of a workflow definition after it has been serialized (e.g., loaded from a file or database). Used in tests to define mock workflow structures.

---

```typescript
vi.mock('@/lib/logs/console/logger', () => ({
  createLogger: () => ({
    info: vi.fn(),
    error: vi.fn(),
    warn: vi.fn(),
    debug: vi.fn(),
  }),
}))
```

*   **`vi.mock(...)`**: This line uses Vitest's mocking utility to replace the actual logger module with a mock version.
    *   **Why mock?**: When running tests, you usually don't want actual log messages cluttering the console or writing to files. Mocking ensures that the `ParallelManager` (or any other component using the logger) doesn't produce real logs during tests.
    *   **`createLogger: () => (...)`**: The mock provides a `createLogger` function that, when called, returns an object.
    *   **`info: vi.fn(), error: vi.fn(), ...`**: This object has `info`, `error`, `warn`, and `debug` methods, all of which are `vi.fn()`. A `vi.fn()` is a mock function that records how it was called (e.g., arguments, number of calls) but doesn't perform any actual action. This allows tests to assert if certain log messages *would have been* logged, without actually logging them.

---

```typescript
describe('ParallelManager', () => {
  const createMockContext = (): ExecutionContext => ({
    workflowId: 'test-workflow',
    blockStates: new Map(),
    blockLogs: [],
    metadata: { startTime: new Date().toISOString(), duration: 0 },
    environmentVariables: {},
    decisions: { router: new Map(), condition: new Map() },
    loopIterations: new Map(),
    loopItems: new Map(),
    completedLoops: new Set(),
    executedBlocks: new Set(),
    activeExecutionPath: new Set(),
    workflow: { blocks: [], connections: [], loops: {}, parallels: {}, version: '2.0' },
    parallelExecutions: new Map(),
  })
```

*   **`describe('ParallelManager', () => { ... })`**: This is the main test suite for the `ParallelManager` class. All tests related to this class will be nested inside this block.
*   **`const createMockContext = (): ExecutionContext => ({ ... })`**: This defines a helper function `createMockContext`.
    *   **Purpose**: Many tests need an `ExecutionContext` object to interact with the `ParallelManager`. This function provides a consistent, pre-initialized mock context, saving boilerplate code in each test.
    *   **`workflowId: 'test-workflow'`**: A simple ID for the mock workflow.
    *   **`blockStates: new Map()`**: A `Map` to hold the state of individual blocks.
    *   **`blockLogs: []`**: An array to store logs associated with blocks.
    *   **`metadata: { ... }`**: Mock metadata for the workflow execution.
    *   **`environmentVariables: {}`**: Empty object for environment variables.
    *   **`decisions: { ... }`**: Maps to track decisions made by router or condition blocks.
    *   **`loopIterations: new Map()`**: Stores the current iteration index for active loops.
    *   **`loopItems: new Map()`**: Stores the item being processed in the current loop/parallel iteration.
    *   **`completedLoops: new Set()`**: A `Set` to track IDs of completed loops or parallels.
    *   **`executedBlocks: new Set()`**: A `Set` to store the IDs of blocks that have already been executed.
    *   **`activeExecutionPath: new Set()`**: A `Set` to store the IDs of blocks currently in the active execution path.
    *   **`workflow: { ... }`**: A mock `workflow` object, containing empty arrays for blocks and connections, and empty objects for loops and parallels, along with a version.
    *   **`parallelExecutions: new Map()`**: A `Map` specifically for storing the execution state of each parallel block. This is crucial for `ParallelManager`'s operations.

---

### `describe('initializeParallel', ...)`

This suite tests the `initializeParallel` method of `ParallelManager`, which sets up the initial state for a parallel execution.

```typescript
  describe('initializeParallel', () => {
    test('should initialize parallel state for array distribution', () => {
      const manager = new ParallelManager()
      const items = ['apple', 'banana', 'cherry']

      const state = manager.initializeParallel('parallel-1', items)

      expect(state.parallelCount).toBe(3)
      expect(state.distributionItems).toEqual(items)
      expect(state.completedExecutions).toBe(0)
      expect(state.executionResults).toBeInstanceOf(Map)
      expect(state.activeIterations).toBeInstanceOf(Set)
      expect(state.currentIteration).toBe(1)
    })

    test('should initialize parallel state for object distribution', () => {
      const manager = new ParallelManager()
      const items = { first: 'alpha', second: 'beta', third: 'gamma' }

      const state = manager.initializeParallel('parallel-1', items)

      expect(state.parallelCount).toBe(3)
      expect(state.distributionItems).toEqual(items)
    })
  })
```

*   **`test('should initialize parallel state for array distribution', () => { ... })`**:
    *   **`const manager = new ParallelManager()`**: Creates a new instance of the `ParallelManager`.
    *   **`const items = ['apple', 'banana', 'cherry']`**: Defines an array of items to be distributed for parallel execution.
    *   **`const state = manager.initializeParallel('parallel-1', items)`**: Calls the `initializeParallel` method with a parallel ID (`'parallel-1'`) and the `items` array. This method should return the initial state for this parallel execution.
    *   **`expect(state.parallelCount).toBe(3)`**: Asserts that `parallelCount` (total number of iterations) is 3, matching the number of items in the array.
    *   **`expect(state.distributionItems).toEqual(items)`**: Asserts that the `distributionItems` in the state are the same as the input `items`.
    *   **`expect(state.completedExecutions).toBe(0)`**: Asserts that initially, no executions have been completed.
    *   **`expect(state.executionResults).toBeInstanceOf(Map)`**: Asserts that `executionResults` (where iteration outputs will be stored) is an empty `Map`.
    *   **`expect(state.activeIterations).toBeInstanceOf(Set)`**: Asserts that `activeIterations` (tracking currently running iterations) is an empty `Set`.
    *   **`expect(state.currentIteration).toBe(1)`**: Asserts that the `currentIteration` is initialized to 1 (often iterations are 1-indexed for display, though sometimes 0-indexed for internal arrays).

*   **`test('should initialize parallel state for object distribution', () => { ... })`**:
    *   Similar to the array test, but `items` is an object.
    *   **`const items = { first: 'alpha', second: 'beta', third: 'gamma' }`**: Defines an object for distribution.
    *   **`expect(state.parallelCount).toBe(3)`**: Asserts that `parallelCount` is 3, matching the number of key-value pairs in the object.
    *   **`expect(state.distributionItems).toEqual(items)`**: Asserts that the distribution items are correctly stored.

---

### `describe('getIterationItem', ...)`

This suite tests the `getIterationItem` method, which retrieves the specific item or entry for a given parallel iteration index.

```typescript
  describe('getIterationItem', () => {
    test('should get item from array distribution', () => {
      const manager = new ParallelManager()
      const state = createParallelExecutionState({
        parallelCount: 3,
        distributionItems: ['apple', 'banana', 'cherry'],
      })

      expect(manager.getIterationItem(state, 0)).toBe('apple')
      expect(manager.getIterationItem(state, 1)).toBe('banana')
      expect(manager.getIterationItem(state, 2)).toBe('cherry')
    })

    test('should get entry from object distribution', () => {
      const manager = new ParallelManager()
      const state = createParallelExecutionState({
        parallelCount: 3,
        distributionItems: { first: 'alpha', second: 'beta', third: 'gamma' },
      })

      expect(manager.getIterationItem(state, 0)).toEqual(['first', 'alpha'])
      expect(manager.getIterationItem(state, 1)).toEqual(['second', 'beta'])
      expect(manager.getIterationItem(state, 2)).toEqual(['third', 'gamma'])
    })

    test('should return null for null distribution items', () => {
      const manager = new ParallelManager()
      const state = createParallelExecutionState({
        parallelCount: 0,
        distributionItems: null,
      })

      expect(manager.getIterationItem(state, 0)).toBeNull()
    })
  })
```

*   **`test('should get item from array distribution', () => { ... })`**:
    *   Uses `createParallelExecutionState` to quickly set up a mock parallel state with an array distribution.
    *   **`expect(manager.getIterationItem(state, 0)).toBe('apple')`**: Calls `getIterationItem` with the state and iteration index (0) and asserts it returns the first item from the array. This is repeated for indices 1 and 2.

*   **`test('should get entry from object distribution', () => { ... })`**:
    *   Sets up a mock state with an object distribution.
    *   **`expect(manager.getIterationItem(state, 0)).toEqual(['first', 'alpha'])`**: For an object, `getIterationItem` is expected to return a `[key, value]` tuple for the given index (representing the order of insertion or a sorted order). This is repeated for indices 1 and 2.

*   **`test('should return null for null distribution items', () => { ... })`**:
    *   Sets up a state where `distributionItems` is `null`.
    *   **`expect(manager.getIterationItem(state, 0)).toBeNull()`**: Asserts that if there are no distribution items, `null` is returned.

---

### `describe('areAllVirtualBlocksExecuted', ...)`

This suite tests `areAllVirtualBlocksExecuted`, a crucial method that checks if all virtual instances of a block within a parallel loop have completed their execution.

```typescript
  describe('areAllVirtualBlocksExecuted', () => {
    test('should return true when all virtual blocks are executed', () => {
      const manager = new ParallelManager()
      const executedBlocks = new Set([
        'func-1_parallel_parallel-1_iteration_0',
        'func-1_parallel_parallel-1_iteration_1',
        'func-1_parallel_parallel-1_iteration_2',
      ])
      const parallel = {
        id: 'parallel-1',
        nodes: ['func-1'],
        distribution: ['a', 'b', 'c'],
      }
      const state = createParallelExecutionState({
        parallelCount: 3,
        distributionItems: ['a', 'b', 'c'],
      })

      const context = {
        workflow: {
          blocks: [],
          connections: [],
        },
        decisions: {
          condition: new Map(),
          router: new Map(),
        },
        executedBlocks: new Set(),
      } as any

      const result = manager.areAllVirtualBlocksExecuted(
        'parallel-1',
        parallel,
        executedBlocks,
        state,
        context
      )

      expect(result).toBe(true)
    })

    test('should return false when some virtual blocks are not executed', () => {
      const manager = new ParallelManager()
      const executedBlocks = new Set([
        'func-1_parallel_parallel-1_iteration_0',
        'func-1_parallel_parallel-1_iteration_1',
        // Missing iteration_2
      ])
      const parallel = {
        id: 'parallel-1',
        nodes: ['func-1'],
        distribution: ['a', 'b', 'c'],
      }
      const state = createParallelExecutionState({
        parallelCount: 3,
        distributionItems: ['a', 'b', 'c'],
      })

      // Create context with external connection to make func-1 a legitimate entry point
      const context = {
        workflow: {
          blocks: [{ id: 'func-1', metadata: { id: 'function' } }],
          connections: [
            {
              source: 'external-block',
              target: 'func-1',
              sourceHandle: 'output',
              targetHandle: 'input',
            },
          ],
        },
        decisions: {
          condition: new Map(),
          router: new Map(),
        },
        executedBlocks: new Set(),
      } as any

      const result = manager.areAllVirtualBlocksExecuted(
        'parallel-1',
        parallel,
        executedBlocks,
        state,
        context
      )

      expect(result).toBe(false)
    })
  })
```

*   **`test('should return true when all virtual blocks are executed', () => { ... })`**:
    *   **`const executedBlocks = new Set([...])`**: Creates a `Set` containing the IDs of all three virtual block instances (`func-1_parallel_parallel-1_iteration_0`, `_1`, `_2`). These IDs follow a specific pattern: `<originalBlockId>_parallel_<parallelId>_iteration_<index>`.
    *   **`const parallel = { ... }`**: Defines a mock `parallel` block structure from the workflow definition, indicating it contains a node `func-1` and a distribution of 3 items.
    *   **`const state = createParallelExecutionState(...)`**: Creates a mock parallel execution state with `parallelCount: 3`.
    *   **`const context = { ... } as any`**: A minimal mock `ExecutionContext`. The important part here is the `executedBlocks` passed directly to the function, which will be checked.
    *   **`const result = manager.areAllVirtualBlocksExecuted(...)`**: Calls the method with the necessary arguments.
    *   **`expect(result).toBe(true)`**: Asserts that `true` is returned because all virtual blocks are present in `executedBlocks`.

*   **`test('should return false when some virtual blocks are not executed', () => { ... })`**:
    *   **`const executedBlocks = new Set([...])`**: Here, one virtual block ID (`_iteration_2`) is intentionally *missing* from the `executedBlocks` set.
    *   **`const context = { ... } as any`**: This mock context is more detailed. It explicitly includes `func-1` in `workflow.blocks` and adds a connection to it.
        *   **Why the connection?**: In a real workflow, a block might only be considered for execution if it's reachable. By adding a connection where `func-1` is a target, it makes `func-1` a valid "entry point" or reachable block, ensuring the `areAllVirtualBlocksExecuted` logic can correctly identify it as a block that *should* be executed.
    *   **`expect(result).toBe(false)`**: Asserts that `false` is returned because `func-1_parallel_parallel-1_iteration_2` is not in `executedBlocks`.

---

### `describe('createVirtualBlockInstances', ...)`

This suite tests `createVirtualBlockInstances`, a method that generates IDs for virtual block instances that still need to be executed for a parallel.

```typescript
  describe('createVirtualBlockInstances', () => {
    test('should create virtual block instances for unexecuted blocks', () => {
      const manager = new ParallelManager()
      const block = {
        id: 'func-1',
        position: { x: 0, y: 0 },
        config: { tool: BlockType.FUNCTION, params: {} },
        inputs: {},
        outputs: {},
        enabled: true,
      }
      const executedBlocks = new Set(['func-1_parallel_parallel-1_iteration_0'])
      const activeExecutionPath = new Set(['func-1'])
      const state = createParallelExecutionState({
        parallelCount: 3,
        distributionItems: ['a', 'b', 'c'],
      })

      const virtualIds = manager.createVirtualBlockInstances(
        block,
        'parallel-1',
        state,
        executedBlocks,
        activeExecutionPath
      )

      expect(virtualIds).toEqual([
        'func-1_parallel_parallel-1_iteration_1',
        'func-1_parallel_parallel-1_iteration_2',
      ])
    })

    test('should skip blocks not in active execution path', () => {
      const manager = new ParallelManager()
      const block = {
        id: 'func-1',
        position: { x: 0, y: 0 },
        config: { tool: BlockType.FUNCTION, params: {} },
        inputs: {},
        outputs: {},
        enabled: true,
      }
      const executedBlocks = new Set<string>()
      const activeExecutionPath = new Set<string>() // Block not in active path
      const state = createParallelExecutionState({
        parallelCount: 3,
        distributionItems: ['a', 'b', 'c'],
      })

      const virtualIds = manager.createVirtualBlockInstances(
        block,
        'parallel-1',
        state,
        executedBlocks,
        activeExecutionPath
      )

      expect(virtualIds).toEqual([])
    })
  })
```

*   **`test('should create virtual block instances for unexecuted blocks', () => { ... })`**:
    *   **`const block = { id: 'func-1', ... }`**: Defines a mock child block (`func-1`) that's part of the parallel.
    *   **`const executedBlocks = new Set(['func-1_parallel_parallel-1_iteration_0'])`**: Indicates that only the first iteration (`_0`) has completed.
    *   **`const activeExecutionPath = new Set(['func-1'])`**: Specifies that the original `func-1` block is currently in the active execution path, meaning its virtual instances should be considered.
    *   **`const state = createParallelExecutionState(...)`**: Sets up a parallel state with 3 iterations.
    *   **`const virtualIds = manager.createVirtualBlockInstances(...)`**: Calls the method.
    *   **`expect(virtualIds).toEqual([ ... ])`**: Asserts that the method returns the virtual IDs for the *remaining* unexecuted iterations (`_1` and `_2`).

*   **`test('should skip blocks not in active execution path', () => { ... })`**:
    *   **`const activeExecutionPath = new Set<string>()`**: Crucially, `func-1` is *not* in the active execution path. This simulates a scenario where `func-1` might be part of the parallel but is not currently relevant for execution (e.g., due to a prior conditional branch).
    *   **`expect(virtualIds).toEqual([])`**: Asserts that no virtual IDs are generated because the parent block (`func-1`) is not active.

---

### `describe('setupIterationContext', ...)`

This suite tests `setupIterationContext`, which modifies the `ExecutionContext` to provide iteration-specific data for a given parallel iteration.

```typescript
  describe('setupIterationContext', () => {
    test('should set up context for array distribution', () => {
      const manager = new ParallelManager()
      const context = createMockContext()

      const state = {
        parallelCount: 3,
        distributionItems: ['apple', 'banana', 'cherry'],
        completedExecutions: 0,
        executionResults: new Map(),
        activeIterations: new Set<number>(),
        currentIteration: 1,
      }

      context.parallelExecutions?.set('parallel-1', state)

      manager.setupIterationContext(context, {
        parallelId: 'parallel-1',
        iterationIndex: 1,
      })

      expect(context.loopItems.get('parallel-1_iteration_1')).toBe('banana')
      expect(context.loopItems.get('parallel-1')).toBe('banana')
      expect(context.loopIterations.get('parallel-1')).toBe(1)
    })

    test('should set up context for object distribution', () => {
      const manager = new ParallelManager()
      const context = createMockContext()

      const state = createParallelExecutionState({
        parallelCount: 2,
        distributionItems: { key1: 'value1', key2: 'value2' },
      })

      context.parallelExecutions?.set('parallel-1', state)

      manager.setupIterationContext(context, {
        parallelId: 'parallel-1',
        iterationIndex: 0,
      })

      expect(context.loopItems.get('parallel-1_iteration_0')).toEqual(['key1', 'value1'])
      expect(context.loopItems.get('parallel-1')).toEqual(['key1', 'value1'])
      expect(context.loopIterations.get('parallel-1')).toBe(0)
    })
  })
```

*   **`test('should set up context for array distribution', () => { ... })`**:
    *   **`const context = createMockContext()`**: Creates a fresh execution context.
    *   **`const state = { ... }`**: Manually defines a mock parallel state with an array distribution.
    *   **`context.parallelExecutions?.set('parallel-1', state)`**: Stores the mock parallel state in the context, as the manager would expect to find it there.
    *   **`manager.setupIterationContext(context, { parallelId: 'parallel-1', iterationIndex: 1 })`**: Calls the method to set up the context for the *second* iteration (index 1).
    *   **`expect(context.loopItems.get('parallel-1_iteration_1')).toBe('banana')`**: Asserts that the item for the *specific iteration key* (`parallel-1_iteration_1`) is stored in `context.loopItems`.
    *   **`expect(context.loopItems.get('parallel-1')).toBe('banana')`**: Asserts that the item for the *general parallel key* (`parallel-1`) is also stored, making it easily accessible as "the current item".
    *   **`expect(context.loopIterations.get('parallel-1')).toBe(1)`**: Asserts that the current iteration index is recorded for the parallel.

*   **`test('should set up context for object distribution', () => { ... })`**:
    *   Similar to the array test, but with an object distribution.
    *   **`expect(context.loopItems.get('parallel-1_iteration_0')).toEqual(['key1', 'value1'])`**: Asserts the correct `[key, value]` tuple for the specific iteration key.
    *   **`expect(context.loopItems.get('parallel-1')).toEqual(['key1', 'value1'])`**: Asserts the correct `[key, value]` tuple for the general parallel key.
    *   **`expect(context.loopIterations.get('parallel-1')).toBe(0)`**: Asserts the correct iteration index.

---

### `describe('storeIterationResult', ...)`

This suite tests `storeIterationResult`, which saves the output of a completed parallel iteration into the parallel's state.

```typescript
  describe('storeIterationResult', () => {
    test('should store iteration result in parallel state', () => {
      const manager = new ParallelManager()
      const context = createMockContext()

      const state = {
        parallelCount: 3,
        distributionItems: ['a', 'b', 'c'],
        completedExecutions: 0,
        executionResults: new Map(), // This is where results are stored
        activeIterations: new Set<number>(),
        currentIteration: 1,
      }

      context.parallelExecutions?.set('parallel-1', state)

      const output = { result: 'test result' }

      manager.storeIterationResult(context, 'parallel-1', 1, output)

      expect(state.executionResults.get('iteration_1')).toEqual(output)
    })
  })
```

*   **`test('should store iteration result in parallel state', () => { ... })`**:
    *   **`const context = createMockContext()`**: Creates a mock execution context.
    *   **`const state = { ... }`**: Creates a mock parallel state, notably with `executionResults: new Map()`.
    *   **`context.parallelExecutions?.set('parallel-1', state)`**: Stores the mock state in the context.
    *   **`const output = { result: 'test result' }`**: Defines a sample output from an iteration.
    *   **`manager.storeIterationResult(context, 'parallel-1', 1, output)`**: Calls the method to store the `output` for iteration `1` of `parallel-1`.
    *   **`expect(state.executionResults.get('iteration_1')).toEqual(output)`**: Asserts that the `output` is correctly stored in the `executionResults` map within the parallel `state`, using a key like `'iteration_1'`.

---

### `describe('processParallelIterations', ...)`

This suite tests `processParallelIterations`, a core method that manages the overall state of parallel executions. It decides if a parallel block needs to be re-executed, marked as complete, or if new virtual blocks should be generated.

```typescript
  describe('processParallelIterations', () => {
    test('should re-execute parallel block when all virtual blocks are complete', async () => {
      const parallels: SerializedWorkflow['parallels'] = {
        'parallel-1': {
          id: 'parallel-1',
          nodes: ['func-1'],
          distribution: ['a', 'b', 'c'],
        },
      }

      const manager = new ParallelManager(parallels)
      const context = createMockContext()

      // Set up context as if parallel has been executed and all virtual blocks completed
      context.executedBlocks.add('parallel-1')
      context.executedBlocks.add('func-1_parallel_parallel-1_iteration_0')
      context.executedBlocks.add('func-1_parallel_parallel-1_iteration_1')
      context.executedBlocks.add('func-1_parallel_parallel-1_iteration_2')

      const state = {
        parallelCount: 3,
        distributionItems: ['a', 'b', 'c'],
        completedExecutions: 0, // This value might get updated by the manager, but not crucial for this test.
        executionResults: new Map(),
        activeIterations: new Set<number>(),
        currentIteration: 1,
      }

      context.parallelExecutions?.set('parallel-1', state)

      await manager.processParallelIterations(context)

      // Should remove parallel from executed blocks and add to active path
      expect(context.executedBlocks.has('parallel-1')).toBe(false)
      expect(context.activeExecutionPath.has('parallel-1')).toBe(true)

      // Should remove child nodes from active path
      expect(context.activeExecutionPath.has('func-1')).toBe(false)
    })

    test('should skip completed parallels', async () => {
      const parallels: SerializedWorkflow['parallels'] = {
        'parallel-1': {
          id: 'parallel-1',
          nodes: ['func-1'],
          distribution: ['a', 'b', 'c'],
        },
      }

      const manager = new ParallelManager(parallels)
      const context = createMockContext()

      // Mark parallel as completed
      context.completedLoops.add('parallel-1')

      await manager.processParallelIterations(context)

      // Should not modify execution state
      expect(context.executedBlocks.size).toBe(0)
      expect(context.activeExecutionPath.size).toBe(0)
    })

    test('should handle empty parallels object', async () => {
      const manager = new ParallelManager({})
      const context = createMockContext()

      // Should complete without error
      await expect(manager.processParallelIterations(context)).resolves.toBeUndefined()
    })
  })
```

*   **`test('should re-execute parallel block when all virtual blocks are complete', async () => { ... })`**:
    *   **`const parallels: SerializedWorkflow['parallels'] = { ... }`**: Provides a mock `parallels` definition to the `ParallelManager` constructor. This is how `ParallelManager` knows about the structure of parallels in the workflow.
    *   **`context.executedBlocks.add('parallel-1')`**: Marks the *parent* parallel block itself as executed.
    *   **`context.executedBlocks.add('func-1_parallel_parallel-1_iteration_0') ...`**: Marks *all* virtual instances of the child block `func-1` as executed.
    *   **`const state = { ... }`**: Provides a mock parallel execution state.
    *   **`await manager.processParallelIterations(context)`**: Calls the main processing method. It's `async` because it might involve asynchronous operations (though not mocked here).
    *   **`expect(context.executedBlocks.has('parallel-1')).toBe(false)`**: After processing, if all virtual blocks are complete, the manager should reset the parent parallel block. This means removing it from `executedBlocks` so it can be re-evaluated (e.g., to proceed to the next block after the parallel).
    *   **`expect(context.activeExecutionPath.has('parallel-1')).toBe(true)`**: The parent parallel block should now be in the `activeExecutionPath`, indicating it's ready for its next logical step (e.g., exiting the parallel and continuing the workflow).
    *   **`expect(context.activeExecutionPath.has('func-1')).toBe(false)`**: The individual child blocks (`func-1`) should be removed from the active path, as their iterations are complete.

*   **`test('should skip completed parallels', async () => { ... })`**:
    *   **`context.completedLoops.add('parallel-1')`**: Intentionally marks the parallel block as `completed` in the context.
    *   **`await manager.processParallelIterations(context)`**: Calls the method.
    *   **`expect(context.executedBlocks.size).toBe(0)` and `expect(context.activeExecutionPath.size).toBe(0)`**: Asserts that if a parallel is already marked as completed, the manager does nothing to modify the execution path or executed blocks.

*   **`test('should handle empty parallels object', async () => { ... })`**:
    *   **`const manager = new ParallelManager({})`**: Initializes the manager with an empty `parallels` object, simulating a workflow with no parallel blocks.
    *   **`await expect(manager.processParallelIterations(context)).resolves.toBeUndefined()`**: Asserts that calling the method with an empty parallels definition completes successfully without throwing an error and resolves to `undefined` (or any non-error value).

---

This comprehensive breakdown covers the purpose, simplified logic, and a line-by-line explanation of the `parallels.test.ts` file, highlighting the key concepts and assertions involved in testing the `ParallelManager`.