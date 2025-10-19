This file contains a comprehensive suite of unit tests for the `LoopManager` class in a TypeScript application. The `LoopManager` is evidently a crucial component responsible for managing the execution, state, and iteration logic of loops (like `for` and `forEach`) within a larger workflow execution engine.

The tests ensure that the `LoopManager` correctly initializes loops, tracks their progress, resets workflow blocks for new iterations, completes loops when their conditions are met, and properly collects results. It also covers various edge cases and interactions with other workflow block types like `ROUTER` and `CONDITION` blocks.

---

### **Core Concepts**

Before diving into the code, let's clarify some key terms used in this file:

*   **`LoopManager`**: This is the class being tested. Its primary role is to orchestrate how loops behave within a workflow.
*   **`ExecutionContext`**: This object represents the current state of a workflow execution. It holds information like which blocks have been executed, current loop iterations, block outputs, active execution paths, and the workflow definition itself.
*   **`SerializedLoop`**: A type definition for how a loop is configured and stored (e.g., its ID, the blocks it contains, number of iterations, loop type).
*   **`SerializedWorkflow`**: A type definition for the entire workflow structure, including all its blocks, connections, and loop definitions.
*   **`BlockType`**: An enum or string literal type that categorizes different kinds of blocks in the workflow (e.g., `STARTER`, `FUNCTION`, `LOOP`, `ROUTER`, `CONDITION`).
*   **Vitest**: The testing framework used (similar to Jest). Key functions:
    *   `describe()`: Groups related tests together.
    *   `test()`: Defines an individual test case.
    *   `expect()`: Used to make assertions about values.
    *   `beforeEach()`: A hook that runs before each test in its `describe` block.
    *   `vi.mock()`: Used to mock modules or functions.

---

### **Detailed Explanation**

Let's break down the code section by section.

```typescript
import { beforeEach, describe, expect, test, vi } from 'vitest'
import { createMockContext } from '@/executor/__test-utils__/executor-mocks'
import { BlockType } from '@/executor/consts'
import { LoopManager } from '@/executor/loops/loops'
import type { ExecutionContext } from '@/executor/types'
import type { SerializedLoop, SerializedWorkflow } from '@/serializer/types'
```

These lines import necessary modules and types:

*   **`beforeEach`, `describe`, `expect`, `test`, `vi`**: These are functions and objects from the `vitest` testing framework, used for defining tests, assertions, and mocking.
*   **`createMockContext`**: A utility function from a local test-utils file, used to generate a pre-configured `ExecutionContext` object for testing purposes.
*   **`BlockType`**: An enum or constant that defines various types of blocks within a workflow (e.g., `LOOP`, `FUNCTION`, `STARTER`, `ROUTER`, `CONDITION`).
*   **`LoopManager`**: This is the class under test. It's responsible for managing loop-related logic during workflow execution.
*   **`ExecutionContext`**: An interface or type defining the structure of the workflow execution context, which holds the current state of the workflow.
*   **`SerializedLoop`, `SerializedWorkflow`**: Interfaces or types defining the structure of how loops and entire workflows are represented when stored or transmitted.

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

This block uses Vitest's mocking capabilities:

*   **`vi.mock(...)`**: It tells Vitest to replace the actual module located at `@/lib/logs/console/logger` with a mock implementation.
*   **`createLogger: () => ({ ... })`**: The mock implementation provides a `createLogger` function. When called, this function returns an object.
*   **`info: vi.fn(), error: vi.fn(), ...`**: This returned object contains mock functions for `info`, `error`, `warn`, and `debug`. `vi.fn()` creates a special mock function that allows you to track calls, arguments, and return values.

**Purpose of this mock**: This ensures that during these tests, any logging calls made by the `LoopManager` (or other components it interacts with) do not produce actual console output. Instead, these calls are intercepted by the mock functions, allowing tests to assert whether specific log messages *would have been* generated, without cluttering the test output or causing unintended side effects.

---

```typescript
describe('LoopManager', () => {
  let manager: LoopManager
  let mockContext: ExecutionContext
```

*   **`describe('LoopManager', () => { ... })`**: This defines the main test suite for the `LoopManager` class. All tests related to `LoopManager` will be nested within this block.
*   **`let manager: LoopManager`**: Declares a variable `manager` that will hold an instance of the `LoopManager` class. This is the "System Under Test" (SUT).
*   **`let mockContext: ExecutionContext`**: Declares a variable `mockContext` that will hold a mocked `ExecutionContext` object. This context simulates the overall state of the workflow execution that the `LoopManager` operates on.

---

```typescript
  const createBasicLoop = (overrides?: Partial<SerializedLoop>): SerializedLoop => ({
    id: 'loop-1',
    nodes: ['block-1', 'block-2'],
    iterations: 3,
    loopType: 'for',
    ...overrides,
  })

  const createForEachLoop = (items: any, overrides?: Partial<SerializedLoop>): SerializedLoop => ({
    id: 'loop-1',
    nodes: ['block-1', 'block-2'],
    iterations: 5, // Default for forEach, can be overridden by actual items length
    loopType: 'forEach',
    forEachItems: items,
    ...overrides,
  })
```

These are helper functions to create `SerializedLoop` objects for testing:

*   **`createBasicLoop`**:
    *   Takes an optional `overrides` object to customize properties.
    *   Returns a `SerializedLoop` object representing a standard "for" loop.
    *   **`id: 'loop-1'`**: A unique identifier for the loop.
    *   **`nodes: ['block-1', 'block-2']`**: An array of block IDs that are considered part of this loop's internal execution path.
    *   **`iterations: 3`**: The number of times this "for" loop is configured to run.
    *   **`loopType: 'for'`**: Specifies that it's a traditional "for" loop.
    *   **`...overrides`**: Allows merging any provided overrides, letting tests easily change specific properties without redefining the whole object.

*   **`createForEachLoop`**:
    *   Takes `items` (which can be an array, object, or JSON string) and optional `overrides`.
    *   Returns a `SerializedLoop` object for a "forEach" loop.
    *   Similar to `createBasicLoop` but with:
        *   **`loopType: 'forEach'`**: Specifies it's a "forEach" loop.
        *   **`forEachItems: items`**: The data collection (array, object, or JSON string) over which the loop will iterate.
        *   **`iterations: 5`**: A default. For `forEach` loops, the actual number of iterations is typically derived from the `forEachItems` array/object length, but this provides a fallback or initial value.

---

```typescript
  const createWorkflowWithLoop = (loop: SerializedLoop): SerializedWorkflow => ({
    version: '2.0',
    blocks: [
      {
        id: 'starter',
        position: { x: 0, y: 0 },
        metadata: { id: BlockType.STARTER, name: 'Start' },
        config: { tool: BlockType.STARTER, params: {} },
        inputs: {},
        outputs: {},
        enabled: true,
      },
      {
        id: 'loop-1',
        position: { x: 100, y: 0 },
        metadata: { id: BlockType.LOOP, name: 'Test Loop' },
        config: { tool: BlockType.LOOP, params: {} },
        inputs: {},
        outputs: {},
        enabled: true,
      },
      {
        id: 'block-1',
        position: { x: 200, y: 0 },
        metadata: { id: BlockType.FUNCTION, name: 'Block 1' },
        config: { tool: BlockType.FUNCTION, params: {} },
        inputs: {},
        outputs: {},
        enabled: true,
      },
      {
        id: 'block-2',
        position: { x: 300, y: 0 },
        metadata: { id: BlockType.FUNCTION, name: 'Block 2' },
        config: { tool: BlockType.FUNCTION, params: {} },
        inputs: {},
        outputs: {},
        enabled: true,
      },
      {
        id: 'after-loop',
        position: { x: 400, y: 0 },
        metadata: { id: BlockType.FUNCTION, name: 'After Loop' },
        config: { tool: BlockType.FUNCTION, params: {} },
        inputs: {},
        outputs: {},
        enabled: true,
      },
    ],
    connections: [
      { source: 'starter', target: 'loop-1' },
      { source: 'loop-1', target: 'block-1', sourceHandle: 'loop-start-source' }, // Enters loop
      { source: 'block-1', target: 'block-2' },
      { source: 'block-2', target: 'loop-1' }, // Completes iteration, returns to loop block
      { source: 'loop-1', target: 'after-loop', sourceHandle: 'loop-end-source' }, // Exits loop
    ],
    loops: {
      'loop-1': loop,
    },
    parallels: {},
  })
```

This is a more complex helper function that generates a complete `SerializedWorkflow` object for testing.

*   **`createWorkflowWithLoop(loop: SerializedLoop)`**: Takes a `SerializedLoop` object (created by `createBasicLoop` or `createForEachLoop`) as input to embed within the workflow.
*   **`version: '2.0'`**: Standard workflow version.
*   **`blocks: [...]`**: Defines an array of individual blocks within the workflow:
    *   `starter`: The entry point of the workflow.
    *   `loop-1`: The loop block itself, identified as `BlockType.LOOP`.
    *   `block-1`, `block-2`: Placeholder blocks that are intended to be executed *inside* the loop.
    *   `after-loop`: A block that executes *after* the loop has completed all its iterations.
    *   Each block includes `id`, `position`, `metadata` (including `BlockType`), `config`, `inputs`, `outputs`, and `enabled` status.
*   **`connections: [...]`**: Defines how blocks are linked:
    *   `starter` -> `loop-1`: The workflow starts and enters the loop.
    *   `loop-1` -> `block-1` (`sourceHandle: 'loop-start-source'`): This special connection signifies entry into the *first* block inside the loop's current iteration.
    *   `block-1` -> `block-2`: Standard connection between internal loop blocks.
    *   `block-2` -> `loop-1`: **Crucially, this connection signifies the end of one iteration within the loop.** After `block-2` completes, control returns to the `loop-1` block to determine if another iteration is needed.
    *   `loop-1` -> `after-loop` (`sourceHandle: 'loop-end-source'`): This special connection signifies exiting the loop once all iterations are complete, and proceeding to the block immediately following the loop.
*   **`loops: { 'loop-1': loop }`**: This object maps loop IDs to their `SerializedLoop` configurations, making the `loop` object passed to the helper function available within the workflow definition.
*   **`parallels: {}`**: An empty object for parallel execution configurations (not relevant to these loop tests).

---

```typescript
  beforeEach(() => {
    const loops = {
      'loop-1': createBasicLoop(),
    }
    manager = new LoopManager(loops)

    mockContext = createMockContext({
      workflow: createWorkflowWithLoop(createBasicLoop()),
      loopIterations: new Map([['loop-1', 0]]),
      loopItems: new Map(),
      executedBlocks: new Set(),
      activeExecutionPath: new Set(['starter', 'loop-1']),
      completedLoops: new Set(),
    })
  })
```

This `beforeEach` hook runs before every single `test` in this `describe('LoopManager')` block. Its purpose is to set up a clean, consistent state for each test.

*   **`const loops = { 'loop-1': createBasicLoop() }`**: Defines a simple map of loop IDs to their configurations, containing one basic loop.
*   **`manager = new LoopManager(loops)`**: Initializes a new `LoopManager` instance with the `loops` configuration. This is the fresh instance of the class under test for each test.
*   **`mockContext = createMockContext({ ... })`**: Initializes `mockContext` using the helper function. It's configured with:
    *   **`workflow: createWorkflowWithLoop(createBasicLoop())`**: A basic workflow containing the `loop-1` definition.
    *   **`loopIterations: new Map([['loop-1', 0]])`**: A `Map` to track the current iteration count for each loop. Initially, `loop-1` is at iteration `0`.
    *   **`loopItems: new Map()`**: A `Map` to store items being iterated over for `forEach` loops (initially empty).
    *   **`executedBlocks: new Set()`**: A `Set` to keep track of which blocks have been executed so far in the workflow. Initially empty.
    *   **`activeExecutionPath: new Set(['starter', 'loop-1'])`**: A `Set` representing the blocks currently being considered for execution. It starts with the `starter` block and the `loop-1` block (as the loop block is typically the first step into the loop's execution logic).
    *   **`completedLoops: new Set()`**: A `Set` to track loops that have finished all their iterations. Initially empty.

---

### **`describe('constructor')`**

This section tests the `LoopManager`'s constructor.

```typescript
  describe('constructor', () => {
    test('should initialize with provided loops', () => {
      const loops = {
        'loop-1': createBasicLoop(),
        'loop-2': createBasicLoop({ id: 'loop-2', iterations: 5 }),
      }
      const loopManager = new LoopManager(loops) // Create a new manager for this test

      expect(loopManager.getIterations('loop-1')).toBe(3)
      expect(loopManager.getIterations('loop-2')).toBe(5)
    })

    test('should use default iterations for unknown loops', () => {
      const loopManager = new LoopManager({}) // Manager with no predefined loops
      expect(loopManager.getIterations('unknown-loop')).toBe(5) // Checks default
    })

    test('should accept custom default iterations', () => {
      const loopManager = new LoopManager({}, 10) // Manager with a custom default
      expect(loopManager.getIterations('unknown-loop')).toBe(10)
    })
  })
```

*   **`test('should initialize with provided loops')`**:
    *   Creates a `loops` object with two defined loops (`loop-1` and `loop-2`).
    *   Instantiates `LoopManager` with these loops.
    *   Asserts that `getIterations` method correctly retrieves the `iterations` value for both loops.
*   **`test('should use default iterations for unknown loops')`**:
    *   Creates a `LoopManager` without any predefined loops (`{}`).
    *   Asserts that calling `getIterations` for a non-existent loop (`'unknown-loop'`) returns a default value, which is `5`.
*   **`test('should accept custom default iterations')`**:
    *   Creates a `LoopManager` with an empty loops object but provides a custom default iteration count of `10` as the second argument to the constructor.
    *   Asserts that `getIterations` for an unknown loop now returns `10`.

---

### **`describe('processLoopIterations')`**

This is the most critical and complex method tested, as it embodies the core logic of loop management during workflow execution. This method is likely called after a block *inside* a loop has completed its execution, or when the loop block itself needs to decide its next step.

**Simplified Logic**:
The `processLoopIterations` method primarily checks if a specific loop (identified by `loopId`) has completed its current iteration. If it has:
1.  It increments the iteration counter for that loop.
2.  It then decides if the loop has reached its *maximum* number of iterations (or processed all items in a `forEach` loop).
    *   **If NOT at max iterations**: It resets the state of all blocks *inside* the loop (clearing `executedBlocks`, `blockStates`, and `activeExecutionPath` for those blocks) to prepare for the *next* iteration. The method returns `false`, signaling that the loop is not yet complete and should continue.
    *   **If AT max iterations**: It marks the loop as `completed`, aggregates all results from its iterations, updates the `loop` block's state with these final results, and activates the connections that lead *out* of the loop. It returns `true`, signaling that the loop has finished its entire execution.

```typescript
  describe('processLoopIterations', () => {
    test('should return false when no loops exist', async () => {
      const emptyManager = new LoopManager({})
      const result = await emptyManager.processLoopIterations(mockContext)
      expect(result).toBe(false)
    })

    test('should skip loops that are already completed', async () => {
      mockContext.completedLoops.add('loop-1') // Mark loop as completed before processing
      const result = await manager.processLoopIterations(mockContext)
      expect(result).toBe(false) // Should not proceed with a completed loop
    })

    test('should skip loops where loop block has not been executed', async () => {
      // Loop block ('loop-1') is not in mockContext.executedBlocks initially
      const result = await manager.processLoopIterations(mockContext)
      expect(result).toBe(false) // If the loop block itself hasn't executed, no iteration can be complete
    })

    test('should skip loops where not all blocks have been executed', async () => {
      mockContext.executedBlocks.add('loop-1')
      mockContext.executedBlocks.add('block-1')
      // block-2 not executed yet (missing from executedBlocks)

      const result = await manager.processLoopIterations(mockContext)
      expect(result).toBe(false) // Not all blocks inside the loop are executed, so current iteration isn't complete
    })

    test('should reset blocks and continue iteration when not at max iterations', async () => {
      // Set up as if we've completed one iteration:
      mockContext.executedBlocks.add('loop-1')
      mockContext.executedBlocks.add('block-1')
      mockContext.executedBlocks.add('block-2') // All blocks in the loop are executed
      mockContext.loopIterations.set('loop-1', 1) // First iteration completed

      // Add some block states to verify they get reset
      mockContext.blockStates.set('block-1', { output: { result: 'test' }, executed: true, executionTime: 100 })
      mockContext.blockStates.set('block-2', { output: { result: 'test2' }, executed: true, executionTime: 200 })

      const result = await manager.processLoopIterations(mockContext)

      expect(result).toBe(false) // Loop is NOT fully completed (not at max iterations yet)

      // Verify blocks were reset for the next iteration:
      expect(mockContext.executedBlocks.has('block-1')).toBe(false) // Removed from executed
      expect(mockContext.executedBlocks.has('block-2')).toBe(false)
      expect(mockContext.executedBlocks.has('loop-1')).toBe(false) // Loop block also reset

      // Verify block states were cleared:
      expect(mockContext.blockStates.has('block-1')).toBe(false) // Block state cleared
      expect(mockContext.blockStates.has('block-2')).toBe(false)
      expect(mockContext.blockStates.has('loop-1')).toBe(false)

      // Verify blocks were removed from active execution path (to be re-added when the loop restarts)
      expect(mockContext.activeExecutionPath.has('block-1')).toBe(false)
      expect(mockContext.activeExecutionPath.has('block-2')).toBe(false)
    })

    test('should complete loop and activate end connections when max iterations reached', async () => {
      // Set up as if we've completed all iterations:
      mockContext.executedBlocks.add('loop-1')
      mockContext.executedBlocks.add('block-1')
      mockContext.executedBlocks.add('block-2')
      mockContext.loopIterations.set('loop-1', 3) // Max iterations (3) reached

      // Set up loop execution state with some results from previous iterations
      mockContext.loopExecutions = new Map()
      mockContext.loopExecutions.set('loop-1', {
        maxIterations: 3,
        loopType: 'for',
        forEachItems: null,
        executionResults: new Map([
          ['iteration_0', { iteration: { 'block-1': { result: 'result1' } } }],
          ['iteration_1', { iteration: { 'block-1': { result: 'result2' } } }],
          ['iteration_2', { iteration: { 'block-1': { result: 'result3' } } }],
        ]),
        currentIteration: 3, // Current iteration is now 3 (0-indexed)
      })

      const result = await manager.processLoopIterations(mockContext)

      expect(result).toBe(true) // Loop is fully completed

      // Verify loop was marked as completed
      expect(mockContext.completedLoops.has('loop-1')).toBe(true)

      // Verify loop block state was updated with aggregated results
      const loopBlockState = mockContext.blockStates.get('loop-1')
      expect(loopBlockState).toBeDefined()
      expect(loopBlockState?.output.completed).toBe(true)
      expect(loopBlockState?.output.results).toHaveLength(3) // All 3 results are aggregated

      // Verify end connection was activated (execution should now proceed to 'after-loop')
      expect(mockContext.activeExecutionPath.has('after-loop')).toBe(true)
    })

    test('should handle forEach loops with array items', async () => {
      const forEachLoop = createForEachLoop(['item1', 'item2', 'item3']) // 3 items
      manager = new LoopManager({ 'loop-1': forEachLoop }) // Update manager with forEach loop
      mockContext.workflow!.loops['loop-1'] = forEachLoop // Update workflow definition

      // Simulate completion of all 3 iterations:
      mockContext.executedBlocks.add('loop-1')
      mockContext.executedBlocks.add('block-1')
      mockContext.executedBlocks.add('block-2')
      mockContext.loopIterations.set('loop-1', 3) // 3 iterations processed

      // Store items in context as the loop handler would (for current iteration)
      mockContext.loopItems.set('loop-1_items', ['item1', 'item2', 'item3']) // All items available

      const result = await manager.processLoopIterations(mockContext)

      expect(result).toBe(true) // Loop completed
      expect(mockContext.completedLoops.has('loop-1')).toBe(true)

      const loopBlockState = mockContext.blockStates.get('loop-1')
      expect(loopBlockState?.output.loopType).toBe('forEach')
      expect(loopBlockState?.output.maxIterations).toBe(3) // Max iterations derived from items length
    })

    test('should handle forEach loops with object items', async () => {
      const items = { key1: 'value1', key2: 'value2' } // 2 items (keys)
      const forEachLoop = createForEachLoop(items)
      manager = new LoopManager({ 'loop-1': forEachLoop })
      mockContext.workflow!.loops['loop-1'] = forEachLoop

      mockContext.executedBlocks.add('loop-1')
      mockContext.executedBlocks.add('block-1')
      mockContext.executedBlocks.add('block-2')
      mockContext.loopIterations.set('loop-1', 2) // All 2 items processed

      mockContext.loopItems.set('loop-1_items', items) // Store items

      const result = await manager.processLoopIterations(mockContext)

      expect(result).toBe(true) // Loop completed
      expect(mockContext.completedLoops.has('loop-1')).toBe(true)

      const loopBlockState = mockContext.blockStates.get('loop-1')
      expect(loopBlockState?.output.maxIterations).toBe(2) // Max iterations derived from object keys length
    })

    test('should handle forEach loops with string items', async () => {
      const forEachLoop = createForEachLoop('["a", "b", "c"]') // JSON string representing an array (3 items)
      manager = new LoopManager({ 'loop-1': forEachLoop })
      mockContext.workflow!.loops['loop-1'] = forEachLoop

      mockContext.executedBlocks.add('loop-1')
      mockContext.executedBlocks.add('block-1')
      mockContext.executedBlocks.add('block-2')
      mockContext.loopIterations.set('loop-1', 3) // All 3 items processed

      const result = await manager.processLoopIterations(mockContext)

      expect(result).toBe(true) // Loop completed
      expect(mockContext.completedLoops.has('loop-1')).toBe(true)
    })
  })
```

*   **`should return false when no loops exist`**: Verifies that `processLoopIterations` handles the case where the manager has no loops configured, returning `false` (meaning no loops were processed or completed).
*   **`should skip loops that are already completed`**: Tests that if a loop is already marked as completed in the `mockContext.completedLoops` Set, `processLoopIterations` correctly ignores it and returns `false`.
*   **`should skip loops where loop block has not been executed`**: Checks that if the `loop-1` block itself hasn't been executed (i.e., not in `mockContext.executedBlocks`), the `LoopManager` won't attempt to process its iterations. This is an initial check before considering internal blocks.
*   **`should skip loops where not all blocks have been executed`**: This tests the logic for determining if a single iteration is complete. If `block-2` (the last block inside the loop before returning to `loop-1`) is not in `executedBlocks`, the iteration is not yet done, and `processLoopIterations` returns `false`.
*   **`should reset blocks and continue iteration when not at max iterations`**: This is a key test for managing ongoing loop execution.
    *   **Setup**: Simulates one completed iteration by adding `loop-1`, `block-1`, `block-2` to `executedBlocks` and setting `loopIterations` to `1`. Also adds example `blockStates`.
    *   **Action**: Calls `processLoopIterations`.
    *   **Assertions**:
        *   `result` is `false` because the loop has more iterations to run.
        *   It then verifies that `block-1`, `block-2`, and `loop-1` are all removed from `executedBlocks`, `blockStates`, and `activeExecutionPath`. This "resets" the loop's internal blocks for the next iteration, effectively clearing their execution history so they can be run again.
*   **`should complete loop and activate end connections when max iterations reached`**: This test covers the loop's termination.
    *   **Setup**: Simulates completion of *all* iterations by setting `loopIterations` to `3` (the max). It also populates `mockContext.loopExecutions` with dummy results for each iteration.
    *   **Action**: Calls `processLoopIterations`.
    *   **Assertions**:
        *   `result` is `true` because the loop has finished.
        *   `loop-1` is added to `mockContext.completedLoops`.
        *   The `loop-1` block's `blockState` is updated with `completed: true` and the `results` collected from `loopExecutions`.
        *   The `after-loop` block is added to `mockContext.activeExecutionPath`, indicating that the workflow should now proceed outside the loop.
*   **`should handle forEach loops with array items` / `object items` / `string items`**: These three tests ensure that `forEach` loops are correctly handled regardless of how their `forEachItems` are defined.
    *   They update the `LoopManager` and `mockContext` with a `forEach` loop.
    *   They simulate all iterations completing based on the number of items.
    *   They assert that `processLoopIterations` correctly identifies the loop as completed (`true`) and sets the `maxIterations` in the output based on the parsed items count. The `string items` test specifically checks `JSON.parse` logic for the `forEachItems` field.

---

### **`describe('storeIterationResult')`**

This section tests the `storeIterationResult` method, which is responsible for recording the outcomes of individual loop iterations.

```typescript
  describe('storeIterationResult', () => {
    test('should create new loop state if none exists', () => {
      const output = { result: 'test result' }

      manager.storeIterationResult(mockContext, 'loop-1', 0, output) // Store result for iteration 0

      expect(mockContext.loopExecutions).toBeDefined()
      const loopState = mockContext.loopExecutions!.get('loop-1')
      expect(loopState).toBeDefined()
      expect(loopState?.maxIterations).toBe(3) // From createBasicLoop
      expect(loopState?.loopType).toBe('for')
      expect(loopState?.executionResults.get('iteration_0')).toEqual(output)
    })

    test('should add to existing loop state', () => {
      // Initialize loop state manually for the first iteration
      mockContext.loopExecutions = new Map()
      mockContext.loopExecutions.set('loop-1', {
        maxIterations: 3,
        loopType: 'for',
        forEachItems: null,
        executionResults: new Map(),
        currentIteration: 0,
      })

      const output1 = { result: 'result1' }
      const output2 = { result: 'result2' }

      manager.storeIterationResult(mockContext, 'loop-1', 0, output1) // Store first result
      manager.storeIterationResult(mockContext, 'loop-1', 0, output2) // Store second result for same iteration

      const loopState = mockContext.loopExecutions.get('loop-1')
      const iterationResults = loopState?.executionResults.get('iteration_0')

      // When multiple results are stored for the same iteration, they are combined into an array
      expect(iterationResults).toEqual([output1, output2])
    })

    test('should handle forEach loop state creation', () => {
      const forEachLoop = createForEachLoop(['item1', 'item2']) // Update manager for forEach loop
      manager = new LoopManager({ 'loop-1': forEachLoop })

      const output = { result: 'test result' }

      manager.storeIterationResult(mockContext, 'loop-1', 0, output)

      const loopState = mockContext.loopExecutions!.get('loop-1')
      expect(loopState?.loopType).toBe('forEach')
      expect(loopState?.forEachItems).toEqual(['item1', 'item2']) // forEach-specific properties stored
    })
  })
```

*   **`should create new loop state if none exists`**:
    *   Calls `storeIterationResult` for a loop when `mockContext.loopExecutions` (which stores aggregated loop states) is empty.
    *   Asserts that a new `loopExecutions` entry is created for `loop-1`, correctly capturing its `maxIterations`, `loopType`, and the provided `output` for `iteration_0`.
*   **`should add to existing loop state`**:
    *   Manually sets up an existing loop state in `mockContext.loopExecutions` for `loop-1`.
    *   Calls `storeIterationResult` *twice* for the *same* iteration (`0`) with different outputs.
    *   Asserts that the `executionResults` for `iteration_0` now contains an array combining both `output1` and `output2`. This implies that if a block inside a loop (or multiple blocks within an iteration) produces results, they are collected together.
*   **`should handle forEach loop state creation`**:
    *   Updates the `manager` to use a `forEach` loop.
    *   Calls `storeIterationResult`.
    *   Asserts that the created `loopState` correctly records `loopType: 'forEach'` and `forEachItems`.

---

### **`describe('getLoopIndex')`**

This section tests the `getLoopIndex` method, which retrieves the current iteration index for a given loop.

```typescript
  describe('getLoopIndex', () => {
    test('should return current iteration for existing loop', () => {
      mockContext.loopIterations.set('loop-1', 2) // Set current iteration to 2

      const index = manager.getLoopIndex('loop-1', 'block-1', mockContext) // 'block-1' is a dummy blockId

      expect(index).toBe(2)
    })

    test('should return 0 for non-existent loop iteration', () => {
      // 'non-existent' is not in mockContext.loopIterations
      const index = manager.getLoopIndex('non-existent', 'block-1', mockContext)

      expect(index).toBe(0) // Default to 0 if no iteration count is tracked
    })

    test('should return 0 for unknown loop', () => {
      const unknownManager = new LoopManager({}) // Manager with no loops
      const index = unknownManager.getLoopIndex('unknown', 'block-1', mockContext)

      expect(index).toBe(0) // Default to 0 for a completely unknown loop
    })
  })
```

*   **`should return current iteration for existing loop`**: Sets `loopIterations` for `loop-1` to `2` and expects `getLoopIndex` to return `2`.
*   **`should return 0 for non-existent loop iteration`**: Tests that if a `loopId` is not present in `mockContext.loopIterations`, `0` is returned.
*   **`should return 0 for unknown loop`**: Tests the same behavior but with a `LoopManager` that was initialized without any loops, further confirming the default `0` for unknown loops.

---

### **`describe('getIterations')`**

This section tests the `getIterations` method, which returns the total configured iterations for a loop.

```typescript
  describe('getIterations', () => {
    test('should return iterations for existing loop', () => {
      expect(manager.getIterations('loop-1')).toBe(3) // From createBasicLoop in beforeEach
    })

    test('should return default iterations for non-existent loop', () => {
      expect(manager.getIterations('non-existent')).toBe(5) // Uses the default (5)
    })
  })
```

*   **`should return iterations for existing loop`**: Checks that the method correctly retrieves the `iterations` value (3) for `loop-1` from its initial configuration.
*   **`should return default iterations for non-existent loop`**: Confirms that for a loop not configured in the manager, it returns the default iteration count (5).

---

### **`describe('getCurrentItem')`**

This section tests the `getCurrentItem` method, used for `forEach` loops to get the specific item for the current iteration.

```typescript
  describe('getCurrentItem', () => {
    test('should return current item for loop', () => {
      mockContext.loopItems.set('loop-1', ['current-item']) // Manually set a current item

      const item = manager.getCurrentItem('loop-1', mockContext)

      expect(item).toEqual(['current-item'])
    })

    test('should return undefined for non-existent loop item', () => {
      // 'non-existent' is not in mockContext.loopItems
      const item = manager.getCurrentItem('non-existent', mockContext)

      expect(item).toBeUndefined()
    })
  })
```

*   **`should return current item for loop`**: Sets a dummy item in `mockContext.loopItems` for `loop-1` and expects `getCurrentItem` to retrieve it.
*   **`should return undefined for non-existent loop item`**: Checks that `undefined` is returned if no item is found for the given `loopId`.

---

### **`describe('allBlocksExecuted (private method testing through processLoopIterations)')`**

This section implicitly tests the `allBlocksExecuted` internal logic within `processLoopIterations`. This logic is crucial for determining if all *reachable* blocks within a loop's current iteration have completed their execution, taking into account dynamic paths like `ROUTER` or `CONDITION` blocks, and error handling.

**Simplified Logic**: The `allBlocksExecuted` logic effectively traverses the workflow graph from the loop's entry point, considering only the paths that were actually taken (based on decisions made by `ROUTER` or `CONDITION` blocks, or error/success outcomes of blocks). It ensures that all blocks on the *active* path for the current iteration are in the `executedBlocks` set before considering the iteration complete.

```typescript
  describe('allBlocksExecuted (private method testing through processLoopIterations)', () => {
    test('should handle router blocks with selected paths', async () => {
      // Create a workflow with a router block inside the loop
      const workflow = createWorkflowWithLoop(createBasicLoop())
      workflow.blocks[2].metadata!.id = BlockType.ROUTER // Make block-1 a router block
      workflow.blocks.push({ // Add an alternative block for the router
        id: 'alternative-block',
        position: { x: 200, y: 100 },
        metadata: { id: BlockType.FUNCTION, name: 'Alternative Block' },
        config: { tool: BlockType.FUNCTION, params: {} },
        inputs: {},
        outputs: {},
        enabled: true,
      })
      workflow.connections = [
        { source: 'starter', target: 'loop-1' },
        { source: 'loop-1', target: 'block-1', sourceHandle: 'loop-start-source' },
        { source: 'block-1', target: 'block-2' }, // Router selects block-2
        { source: 'block-1', target: 'alternative-block' }, // Alternative path from router
        { source: 'block-2', target: 'loop-1' },
        { source: 'loop-1', target: 'after-loop', sourceHandle: 'loop-end-source' },
      ]

      mockContext.workflow = workflow // Update context with the new workflow
      mockContext.executedBlocks.add('loop-1')
      mockContext.executedBlocks.add('block-1')
      mockContext.executedBlocks.add('block-2') // Block-2 is on the selected path and executed
      mockContext.decisions.router.set('block-1', 'block-2') // Router decision: chose 'block-2'
      mockContext.loopIterations.set('loop-1', 1)

      const result = await manager.processLoopIterations(mockContext)

      // Should process the iteration since all reachable blocks (block-1 -> block-2) are executed
      expect(result).toBe(false) // Not at max iterations yet, so should reset for next iteration
    })

    test('should handle condition blocks with selected paths', async () => {
      // Create a workflow with a condition block inside the loop
      const workflow = createWorkflowWithLoop(createBasicLoop())
      workflow.blocks[2].metadata!.id = BlockType.CONDITION // Make block-1 a condition block
      workflow.blocks.push({ // Add an alternative block for the condition's false path
        id: 'alternative-block',
        position: { x: 200, y: 100 },
        metadata: { id: BlockType.FUNCTION, name: 'Alternative Block' },
        config: { tool: BlockType.FUNCTION, params: {} },
        inputs: {},
        outputs: {},
        enabled: true,
      })
      workflow.connections = [
        { source: 'starter', target: 'loop-1' },
        { source: 'loop-1', target: 'block-1', sourceHandle: 'loop-start-source' },
        { source: 'block-1', target: 'block-2', sourceHandle: 'condition-true' }, // Condition true path
        { source: 'block-1', target: 'alternative-block', sourceHandle: 'condition-false' }, // Condition false path
        { source: 'block-2', target: 'loop-1' },
        { source: 'loop-1', target: 'after-loop', sourceHandle: 'loop-end-source' },
      ]

      mockContext.workflow = workflow
      mockContext.executedBlocks.add('loop-1')
      mockContext.executedBlocks.add('block-1')
      mockContext.executedBlocks.add('block-2') // Block-2 is on the true path and executed
      mockContext.decisions.condition.set('block-1', 'true') // Condition decision: chose 'true' path
      mockContext.loopIterations.set('loop-1', 1)

      const result = await manager.processLoopIterations(mockContext)

      // Should process the iteration since all reachable blocks (block-1 -> block-2) are executed
      expect(result).toBe(false) // Not at max iterations yet
    })

    test('should handle error connections properly', async () => {
      // Create a workflow with error handling inside the loop
      const workflow = createWorkflowWithLoop(createBasicLoop())
      // No 'error-handler' block is added here, assuming block-1 succeeded.
      // The crucial part is that the error path is not taken, so it shouldn't be required for "all blocks executed".
      workflow.connections = [
        { source: 'starter', target: 'loop-1' },
        { source: 'loop-1', target: 'block-1', sourceHandle: 'loop-start-source' },
        { source: 'block-1', target: 'block-2', sourceHandle: 'source' }, // Success path
        { source: 'block-1', target: 'error-handler', sourceHandle: 'error' }, // Error path (not taken in this test)
        { source: 'block-2', target: 'loop-1' },
        { source: 'loop-1', target: 'after-loop', sourceHandle: 'loop-end-source' },
      ]

      mockContext.workflow = workflow
      mockContext.executedBlocks.add('loop-1')
      mockContext.executedBlocks.add('block-1')
      mockContext.executedBlocks.add('block-2')

      // Set block-1 to have no error (successful execution)
      mockContext.blockStates.set('block-1', {
        output: { result: 'success' },
        executed: true,
        executionTime: 100,
      })

      mockContext.loopIterations.set('loop-1', 1)

      const result = await manager.processLoopIterations(mockContext)

      // Should process the iteration since the success path was followed and completed
      expect(result).toBe(false) // Not at max iterations yet
    })

    test('should handle blocks with errors following error paths', async () => {
      // Create a workflow with error handling inside the loop
      const workflow = createWorkflowWithLoop(createBasicLoop())
      workflow.blocks.push({ // Add the error-handler block
        id: 'error-handler',
        position: { x: 350, y: 100 },
        metadata: { id: BlockType.FUNCTION, name: 'Error Handler' },
        config: { tool: BlockType.FUNCTION, params: {} },
        inputs: {},
        outputs: {},
        enabled: true,
      })
      workflow.loops['loop-1'].nodes.push('error-handler') // Add error-handler to loop's nodes
      workflow.connections = [
        { source: 'starter', target: 'loop-1' },
        { source: 'loop-1', target: 'block-1', sourceHandle: 'loop-start-source' },
        { source: 'block-1', target: 'block-2', sourceHandle: 'source' }, // Success path
        { source: 'block-1', target: 'error-handler', sourceHandle: 'error' }, // Error path (taken in this test)
        { source: 'error-handler', target: 'loop-1' }, // Error handler returns to loop block
        { source: 'block-2', target: 'loop-1' },
        { source: 'loop-1', target: 'after-loop', sourceHandle: 'loop-end-source' },
      ]

      mockContext.workflow = workflow
      mockContext.executedBlocks.add('loop-1')
      mockContext.executedBlocks.add('block-1')
      mockContext.executedBlocks.add('error-handler') // Error handler was executed

      // Set block-1 to have an error
      mockContext.blockStates.set('block-1', {
        output: {
          error: 'Something went wrong', // Indicates an error
        },
        executed: true,
        executionTime: 100,
      })

      mockContext.loopIterations.set('loop-1', 1)

      const result = await manager.processLoopIterations(mockContext)

      // Should process the iteration since the error path was followed and completed
      expect(result).toBe(false) // Not at max iterations yet
    })
  })
```

*   **`should handle router blocks with selected paths`**:
    *   Modifies the workflow so `block-1` is a `ROUTER`.
    *   Simulates `block-1` choosing the path to `block-2` by setting `mockContext.decisions.router`.
    *   Asserts that `processLoopIterations` correctly considers the iteration complete because `block-1` and the chosen path's block (`block-2`) are executed, ignoring the unchosen `alternative-block`.
*   **`should handle condition blocks with selected paths`**: Similar to the router test, but for a `CONDITION` block. It verifies that only the dynamically selected path's blocks need to be executed for the iteration to be considered complete.
*   **`should handle error connections properly`**:
    *   Configures a workflow where `block-1` has both a regular ('source') and an 'error' output connection.
    *   Simulates `block-1` executing *successfully* (no error in its `blockState`).
    *   Asserts that the iteration is considered complete when the success path (`block-1` -> `block-2`) is executed, and the `error-handler` block (which was not taken) is not required for completion.
*   **`should handle blocks with errors following error paths`**:
    *   Similar to the previous test, but `block-1` is simulated to *error* (by setting an `error` in its `blockState`).
    *   The workflow now expects the `error` connection to be taken, leading to an `error-handler` block.
    *   Asserts that the iteration is considered complete when the error path (`block-1` -> `error-handler`) is executed, ensuring that error handling logic correctly affects iteration completion checks.

---

### **`describe('edge cases and error handling')`**

This section focuses on how the `LoopManager` behaves under unusual or potentially problematic conditions.

```typescript
  describe('edge cases and error handling', () => {
    test('should handle empty loop nodes array', async () => {
      const emptyLoop = createBasicLoop({ nodes: [] }) // A loop with no internal blocks
      manager = new LoopManager({ 'loop-1': emptyLoop })
      mockContext.workflow!.loops['loop-1'] = emptyLoop

      mockContext.executedBlocks.add('loop-1') // Loop block itself is executed
      mockContext.loopIterations.set('loop-1', 1) // First iteration (of 3)

      const result = await manager.processLoopIterations(mockContext)

      // Should complete immediately since there are no blocks to execute inside the loop
      expect(result).toBe(false) // Still not at max iterations (1 of 3), so will reset.
                                 // The point is it doesn't get stuck waiting for non-existent nodes.
    })

    test('should handle missing workflow in context', async () => {
      mockContext.workflow = undefined // Simulate missing workflow

      const result = await manager.processLoopIterations(mockContext)

      expect(result).toBe(false) // Should gracefully return false, not throw an error
    })

    test('should handle missing loop configuration', async () => {
      // Remove loop configuration from workflow
      if (mockContext.workflow) {
        mockContext.workflow.loops = {} // No 'loop-1' defined in workflow.loops
      }

      mockContext.executedBlocks.add('loop-1')
      mockContext.executedBlocks.add('block-1')
      mockContext.executedBlocks.add('block-2')
      mockContext.loopIterations.set('loop-1', 1)

      const result = await manager.processLoopIterations(mockContext)

      // Should skip processing since loop config is missing for 'loop-1'
      expect(result).toBe(false)
    })

    test('should handle forEach loop with invalid JSON string', async () => {
      const forEachLoop = createForEachLoop('invalid json') // Invalid JSON string
      manager = new LoopManager({ 'loop-1': forEachLoop })
      mockContext.workflow!.loops['loop-1'] = forEachLoop

      mockContext.executedBlocks.add('loop-1')
      mockContext.executedBlocks.add('block-1')
      mockContext.executedBlocks.add('block-2')
      mockContext.loopIterations.set('loop-1', 1)

      const result = await manager.processLoopIterations(mockContext)

      // Should handle gracefully (e.g., using default iterations or logging an error)
      expect(result).toBe(false) // Loop continues, not completed (likely defaults to 1 iteration or handles error)
    })

    test('should handle forEach loop with null items', async () => {
      const forEachLoop = createForEachLoop(null) // Null items for forEach
      manager = new LoopManager({ 'loop-1': forEachLoop })
      mockContext.workflow!.loops['loop-1'] = forEachLoop

      mockContext.executedBlocks.add('loop-1')
      mockContext.executedBlocks.add('block-1')
      mockContext.executedBlocks.add('block-2')
      mockContext.loopIterations.set('loop-1', 1)

      const result = await manager.processLoopIterations(mockContext)

      // Should handle gracefully (e.g., treating it as an empty list or erroring appropriately)
      expect(result).toBe(false)
    })
  })
```

*   **`should handle empty loop nodes array`**: Tests a loop configured with an empty `nodes` array. It asserts that `processLoopIterations` still proceeds and does not get stuck, treating an empty node list as an "instant" completion of the inner iteration.
*   **`should handle missing workflow in context`**: Sets `mockContext.workflow` to `undefined`. Expects `processLoopIterations` to handle this gracefully (return `false`) without crashing.
*   **`should handle missing loop configuration`**: Removes the `loop-1` definition from `mockContext.workflow.loops`. Ensures that `processLoopIterations` skips processing for this loop if its configuration is unavailable.
*   **`should handle forEach loop with invalid JSON string`**: Provides an invalid JSON string as `forEachItems` for a `forEach` loop. The test expects the `LoopManager` to handle this gracefully (e.g., not crash, perhaps defaulting to 0 or 1 iteration, or logging an error, but not completing the loop prematurely).
*   **`should handle forEach loop with null items`**: Tests with `null` as `forEachItems`. Expects graceful handling.

---

### **`describe('integration scenarios')`**

This section explores how `LoopManager` handles more complex, real-world scenarios involving multiple loops or nested loops.

```typescript
  describe('integration scenarios', () => {
    test('should handle multiple loops in workflow', async () => {
      const loops = {
        'loop-1': createBasicLoop({ iterations: 2 }), // loop-1 has 2 iterations
        'loop-2': createBasicLoop({ id: 'loop-2', nodes: ['block-3'], iterations: 3 }), // loop-2 has 3 iterations, different nodes
      }
      manager = new LoopManager(loops) // Manager now has both loops

      // Set up context for both loops
      mockContext.loopIterations.set('loop-1', 2) // loop-1 is at max iterations (0-indexed 2 means 3rd iteration done)
      mockContext.loopIterations.set('loop-2', 1) // loop-2 is not at max (1 of 3 done)

      mockContext.executedBlocks.add('loop-1')
      mockContext.executedBlocks.add('block-1')
      mockContext.executedBlocks.add('block-2') // Blocks for loop-1 executed

      // Set up loop execution states for loop-1
      mockContext.loopExecutions = new Map()
      mockContext.loopExecutions.set('loop-1', {
        maxIterations: 2,
        loopType: 'for',
        forEachItems: null,
        executionResults: new Map([
          ['iteration_0', { iteration: { 'block-1': { result: 'result1' } } }],
          ['iteration_1', { iteration: { 'block-1': { result: 'result2' } } }],
        ]),
        currentIteration: 2,
      })

      const result = await manager.processLoopIterations(mockContext)

      expect(result).toBe(true) // loop-1 reached max iterations, so processLoopIterations for loop-1 completes
      expect(mockContext.completedLoops.has('loop-1')).toBe(true)
      expect(mockContext.completedLoops.has('loop-2')).toBe(false) // loop-2 should not be completed
    })

    test('should handle nested loop scenarios (loop inside another loop)', async () => {
      // This tests the scenario where a loop block might be inside another loop
      const outerLoop = createBasicLoop({
        id: 'outer-loop',
        nodes: ['inner-loop', 'block-1'], // Outer loop contains inner-loop block and block-1
        iterations: 2,
      })
      const innerLoop = createBasicLoop({
        id: 'inner-loop',
        nodes: ['block-2'], // Inner loop contains block-2
        iterations: 3,
      })

      const loops = {
        'outer-loop': outerLoop,
        'inner-loop': innerLoop,
      }
      manager = new LoopManager(loops) // Manager now handles both loops
      // Update workflow to reflect nested structure
      mockContext.workflow = createWorkflowWithLoop(outerLoop)
      mockContext.workflow.blocks.push(
        { // Add inner-loop block to workflow
          id: 'inner-loop',
          position: { x: 250, y: 0 },
          metadata: { id: BlockType.LOOP, name: 'Inner Loop' },
          config: { tool: BlockType.LOOP, params: {} },
          inputs: {},
          outputs: {},
          enabled: true,
        },
        { // Add block-2 (part of inner loop)
          id: 'block-2',
          position: { x: 275, y: 0 },
          metadata: { id: BlockType.FUNCTION, name: 'Block 2 (Inner)' },
          config: { tool: BlockType.FUNCTION, params: {} },
          inputs: {},
          outputs: {},
          enabled: true,
        },
      )
      // Redefine connections for nesting
      mockContext.workflow.connections = [
        { source: 'starter', target: 'outer-loop' },
        { source: 'outer-loop', target: 'inner-loop', sourceHandle: 'loop-start-source' }, // Outer enters inner
        { source: 'inner-loop', target: 'block-2', sourceHandle: 'loop-start-source' }, // Inner enters block-2
        { source: 'block-2', target: 'inner-loop' }, // Block-2 completes inner iteration
        { source: 'inner-loop', target: 'block-1', sourceHandle: 'loop-end-source' }, // Inner loop completes, proceeds to block-1 (within outer loop)
        { source: 'block-1', target: 'outer-loop' }, // Block-1 completes outer iteration
        { source: 'outer-loop', target: 'after-loop', sourceHandle: 'loop-end-source' }, // Outer loop completes
      ]
      mockContext.workflow.loops['inner-loop'] = innerLoop // Add inner loop definition

      // Set up context - inner loop completed, outer loop still running
      mockContext.loopIterations.set('outer-loop', 1) // Outer loop has done 1 of 2 iterations
      mockContext.loopIterations.set('inner-loop', 3) // Inner loop has done all 3 of 3 iterations

      mockContext.executedBlocks.add('outer-loop')
      mockContext.executedBlocks.add('inner-loop')
      mockContext.executedBlocks.add('block-1') // Block-1 (after inner-loop) executed
      mockContext.executedBlocks.add('block-2') // Block-2 (inside inner-loop) executed

      mockContext.completedLoops.add('inner-loop') // Inner loop is marked as completed

      const result = await manager.processLoopIterations(mockContext)
      // In this specific setup, processLoopIterations is called, likely for the 'outer-loop' because 'block-1' has just completed.
      // The outer loop's iteration (which includes the entire inner loop's execution) is now complete.

      // Should reset outer loop for next iteration
      expect(result).toBe(false) // Outer loop is not at max iterations (1 of 2 done), so it resets and continues
      expect(mockContext.executedBlocks.has('inner-loop')).toBe(false) // Inner loop block reset
      expect(mockContext.executedBlocks.has('block-1')).toBe(false) // Block-1 reset
      expect(mockContext.executedBlocks.has('block-2')).toBe(false) // Block-2 reset
      // Note: mockContext.completedLoops.has('inner-loop') would still be true, as the inner loop has indeed completed its *entire* run.
      // Resetting here is for the *outer* loop's new iteration.
    })
  })
```

*   **`should handle multiple loops in workflow`**:
    *   Configures `manager` with two distinct loops (`loop-1`, `loop-2`).
    *   Sets `loop-1` to be at its maximum iterations (`2`) and `loop-2` to be in progress (`1`).
    *   When `processLoopIterations` is called (which would process all relevant loops in the current context), it confirms that `loop-1` is correctly identified as completed (`result: true`), but `loop-2` remains uncompleted. This verifies that the manager can differentiate and manage multiple loops independently.
*   **`should handle nested loop scenarios (loop inside another loop)`**:
    *   This is the most complex test case. It sets up an `outerLoop` that contains an `innerLoop` as one of its nodes.
    *   The `workflow` connections are carefully defined to reflect this nesting: `outerLoop` enters `innerLoop`, `innerLoop` runs `block-2`, `block-2` returns to `innerLoop` (for next inner iteration), `innerLoop` exits to `block-1` (after all inner iterations), `block-1` returns to `outerLoop` (for next outer iteration), and finally `outerLoop` exits to `after-loop`.
    *   **Setup**: Simulates the `innerLoop` having completed all its iterations (`loopIterations` for `inner-loop` is `3`, and `inner-loop` is in `completedLoops`). The `outerLoop` has completed one of its two iterations (`loopIterations` for `outer-loop` is `1`).
    *   **Action**: `processLoopIterations` is called. At this point, `block-1` (which is part of the outer loop after the inner loop) has just finished.
    *   **Assertions**:
        *   `result` is `false` because the `outerLoop` has not yet reached its maximum iterations (it has 1 more to go).
        *   It verifies that the blocks associated with the *outer* loop's iteration (`inner-loop`, `block-1`, `block-2`) are all reset (removed from `executedBlocks`). This shows that even though the `inner-loop` itself is conceptually "completed," its *blocks* are reset within the context of the *outer* loop's subsequent iteration. This is crucial for correctly restarting the outer loop's internal path.

---

### **Summary of `LoopManager`'s Responsibilities (as tested)**:

The tests demonstrate that `LoopManager` effectively:

1.  **Initializes** and stores loop configurations.
2.  **Tracks iterations** for both `for` and `forEach` loops.
3.  **Determines iteration completion** by checking if all necessary blocks within a loop's current iteration have executed, dynamically accounting for `ROUTER`, `CONDITION`, and error paths.
4.  **Resets state** (executed blocks, block states, active path) of internal loop blocks to prepare for the next iteration.
5.  **Aggregates results** from all iterations when a loop completes.
6.  **Marks loops as completed** and activates outbound connections from the loop block.
7.  **Handles `forEach` loop item parsing** (arrays, objects, JSON strings).
8.  **Gracefully manages edge cases** like missing configurations or invalid data.
9.  **Supports multiple and nested loops** within a single workflow execution.

This detailed testing ensures the `LoopManager` is robust and correctly handles the complex state transitions involved in executing loops in a workflow engine.