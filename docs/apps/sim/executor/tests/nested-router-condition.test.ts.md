```typescript
import { beforeEach, describe, expect, it } from 'vitest'
import { BlockType } from '@/executor/consts'
import { PathTracker } from '@/executor/path/path'
import type { ExecutionContext } from '@/executor/types'
import type { SerializedWorkflow } from '@/serializer/types'

describe('Nested Routing Fix - Router → Condition → Target', () => {
  // Define variables to hold the workflow configuration, path tracking logic, and execution context.
  let workflow: SerializedWorkflow // Represents the workflow structure (blocks and connections).
  let pathTracker: PathTracker // Manages the active execution paths within the workflow.
  let mockContext: ExecutionContext // Simulates the execution environment for the workflow.

  beforeEach(() => {
    // This function runs before each test case to set up the necessary environment.
    // It creates a workflow, a path tracker, and a mock execution context.

    // Create a workflow similar to the screenshot: Router → Condition → Function/Parallel
    workflow = {
      version: '2.0', // Version of the workflow definition.
      blocks: [
        // Define the blocks in the workflow. Each block represents a step in the workflow.
        {
          id: 'starter', // Unique identifier for the block.
          position: { x: 0, y: 0 }, // Position of the block in the visual editor (not relevant to logic).
          metadata: { id: BlockType.STARTER, name: 'Start' }, // Metadata about the block, including its type.
          config: { tool: BlockType.STARTER, params: {} }, // Configuration specific to the block type.
          inputs: {}, // Input connections to the block.
          outputs: {}, // Output connections from the block.
          enabled: true, // Whether the block is enabled.
        },
        {
          id: 'router-1',
          position: { x: 100, y: 0 },
          metadata: { id: BlockType.ROUTER, name: 'Router 1' },
          config: { tool: BlockType.ROUTER, params: {} },
          inputs: {},
          outputs: {},
          enabled: true,
        },
        {
          id: 'function-2',
          position: { x: 200, y: -100 },
          metadata: { id: BlockType.FUNCTION, name: 'Function 2' },
          config: { tool: BlockType.FUNCTION, params: {} },
          inputs: {},
          outputs: {},
          enabled: true,
        },
        {
          id: 'condition-1',
          position: { x: 200, y: 100 },
          metadata: { id: BlockType.CONDITION, name: 'Condition 1' },
          config: { tool: BlockType.CONDITION, params: {} },
          inputs: {},
          outputs: {},
          enabled: true,
        },
        {
          id: 'function-4',
          position: { x: 350, y: 50 },
          metadata: { id: BlockType.FUNCTION, name: 'Function 4' },
          config: { tool: BlockType.FUNCTION, params: {} },
          inputs: {},
          outputs: {},
          enabled: true,
        },
        {
          id: 'parallel-block',
          position: { x: 350, y: 150 },
          metadata: { id: BlockType.PARALLEL, name: 'Parallel Block' },
          config: { tool: BlockType.PARALLEL, params: {} },
          inputs: {},
          outputs: {},
          enabled: true,
        },
        {
          id: 'agent-inside-parallel',
          position: { x: 450, y: 150 },
          metadata: { id: BlockType.AGENT, name: 'Agent Inside Parallel' },
          config: { tool: BlockType.AGENT, params: {} },
          inputs: {},
          outputs: {},
          enabled: true,
        },
      ],
      connections: [
        // Define the connections between the blocks, specifying the flow of execution.
        { source: 'starter', target: 'router-1' }, // The 'starter' block connects to the 'router-1' block.
        { source: 'router-1', target: 'function-2' }, // 'router-1' connects to 'function-2'.
        { source: 'router-1', target: 'condition-1' }, // 'router-1' also connects to 'condition-1', creating a branching path.
        {
          source: 'condition-1',
          target: 'function-4',
          sourceHandle: 'condition-b8f0a33c-a57f-4a36-ac7a-dc9f2b5e6c07-if', // Specific output handle from condition-1 to function-4 ('if' branch).
        },
        {
          source: 'condition-1',
          target: 'parallel-block',
          sourceHandle: 'condition-b8f0a33c-a57f-4a36-ac7a-dc9f2b5e6c07-else', // Specific output handle from condition-1 to parallel-block ('else' branch).
        },
        {
          source: 'parallel-block',
          target: 'agent-inside-parallel',
          sourceHandle: 'parallel-start-source', // Connection from the parallel block to the agent inside.
        },
      ],
      loops: {}, // Not used in this test case.
      parallels: {
        // Define parallel execution configurations.
        'parallel-block': {
          id: 'parallel-block', // ID of the parallel block.
          nodes: ['agent-inside-parallel'], // Nodes that will be executed in parallel.
          distribution: ['item1', 'item2'], // Distribution of items for parallel execution (e.g., for each item).
        },
      },
    }

    pathTracker = new PathTracker(workflow) // Initialize the path tracker with the defined workflow.

    mockContext = {
      // Create a mock execution context to simulate the workflow's runtime environment.
      workflowId: 'test-workflow', // Unique ID of the workflow.
      blockStates: new Map(), // Stores the state of each block during execution.
      blockLogs: [], // Stores the logs generated by each block.
      metadata: { duration: 0 }, // Metadata about the workflow execution.
      environmentVariables: {}, // Environment variables accessible to the workflow.
      decisions: { router: new Map(), condition: new Map() }, // Stores the decisions made by router and condition blocks.
      loopIterations: new Map(), // Stores the number of iterations for each loop.
      loopItems: new Map(), // Stores the current item for each loop.
      completedLoops: new Set(), // Stores the IDs of completed loops.
      executedBlocks: new Set(), // Stores the IDs of blocks that have been executed.
      activeExecutionPath: new Set(), // Stores the IDs of blocks that are currently in the active execution path.
      workflow, // Reference to the workflow definition.
    }

    // Initialize starter as executed and in active path
    mockContext.executedBlocks.add('starter') // Mark the 'starter' block as executed.
    mockContext.activeExecutionPath.add('starter') // Add the 'starter' block to the active execution path.
    mockContext.activeExecutionPath.add('router-1') // Add the 'router-1' block to the active execution path.
  })

  it('should handle nested routing: router selects condition, condition selects function', () => {
    // This test case verifies that the path tracker correctly handles a scenario where a router block
    // selects a condition block, and the condition block then selects a function block.

    // Step 1: Router selects the condition path (not function-2)
    mockContext.blockStates.set('router-1', {
      // Simulate the output of the router block, indicating that it selected the 'condition-1' block.
      output: {
        selectedPath: {
          blockId: 'condition-1',
          blockType: BlockType.CONDITION,
          blockTitle: 'Condition 1',
        },
      },
      executed: true,
      executionTime: 0,
    })
    mockContext.executedBlocks.add('router-1') // Mark the 'router-1' block as executed.

    // Update paths after router execution
    pathTracker.updateExecutionPaths(['router-1'], mockContext) // Update the active execution paths based on the router's decision.

    // Verify router decision
    expect(mockContext.decisions.router.get('router-1')).toBe('condition-1') // Assert that the router's decision was correctly recorded.

    // After router execution, condition should be active but not function-2
    expect(mockContext.activeExecutionPath.has('condition-1')).toBe(true) // Assert that the 'condition-1' block is now in the active execution path.
    expect(mockContext.activeExecutionPath.has('function-2')).toBe(false) // Assert that the 'function-2' block is NOT in the active execution path.

    // CRITICAL: Parallel block should NOT be activated yet
    expect(mockContext.activeExecutionPath.has('parallel-block')).toBe(false) // Assert that the 'parallel-block' is NOT in the active execution path.
    expect(mockContext.activeExecutionPath.has('agent-inside-parallel')).toBe(false) // Assert that the 'agent-inside-parallel' block is NOT in the active execution path.

    // Step 2: Condition executes and selects function-4 (not parallel)
    mockContext.blockStates.set('condition-1', {
      // Simulate the output of the condition block, indicating that it selected the 'function-4' block.
      output: {
        result: 'two',
        stdout: '',
        conditionResult: true,
        selectedPath: {
          blockId: 'function-4',
          blockType: BlockType.FUNCTION,
          blockTitle: 'Function 4',
        },
        selectedConditionId: 'b8f0a33c-a57f-4a36-ac7a-dc9f2b5e6c07-if',
      },
      executed: true,
      executionTime: 0,
    })
    mockContext.executedBlocks.add('condition-1') // Mark the 'condition-1' block as executed.

    // Update paths after condition execution
    pathTracker.updateExecutionPaths(['condition-1'], mockContext) // Update the active execution paths based on the condition's decision.

    // Verify condition decision
    expect(mockContext.decisions.condition.get('condition-1')).toBe(
      'b8f0a33c-a57f-4a36-ac7a-dc9f2b5e6c07-if'
    ) // Assert that the condition's decision was correctly recorded.

    // After condition execution, function-4 should be active
    expect(mockContext.activeExecutionPath.has('function-4')).toBe(true) // Assert that the 'function-4' block is now in the active execution path.

    // CRITICAL: Parallel block should still NOT be activated
    expect(mockContext.activeExecutionPath.has('parallel-block')).toBe(false) // Assert that the 'parallel-block' is still NOT in the active execution path.
    expect(mockContext.activeExecutionPath.has('agent-inside-parallel')).toBe(false) // Assert that the 'agent-inside-parallel' block is still NOT in the active execution path.
  })

  it('should handle nested routing: router selects condition, condition selects parallel', () => {
    // This test case verifies that the path tracker correctly handles a scenario where a router block
    // selects a condition block, and the condition block then selects a parallel block.

    // Step 1: Router selects the condition path
    mockContext.blockStates.set('router-1', {
      output: {
        selectedPath: {
          blockId: 'condition-1',
          blockType: BlockType.CONDITION,
          blockTitle: 'Condition 1',
        },
      },
      executed: true,
      executionTime: 0,
    })
    mockContext.executedBlocks.add('router-1')

    pathTracker.updateExecutionPaths(['router-1'], mockContext)

    // Step 2: Condition executes and selects parallel-block (not function-4)
    mockContext.blockStates.set('condition-1', {
      output: {
        result: 'else',
        stdout: '',
        conditionResult: false,
        selectedPath: {
          blockId: 'parallel-block',
          blockType: BlockType.PARALLEL,
          blockTitle: 'Parallel Block',
        },
        selectedConditionId: 'b8f0a33c-a57f-4a36-ac7a-dc9f2b5e6c07-else',
      },
      executed: true,
      executionTime: 0,
    })
    mockContext.executedBlocks.add('condition-1')

    pathTracker.updateExecutionPaths(['condition-1'], mockContext)

    // Verify condition decision
    expect(mockContext.decisions.condition.get('condition-1')).toBe(
      'b8f0a33c-a57f-4a36-ac7a-dc9f2b5e6c07-else'
    )

    // After condition execution, parallel-block should be active
    expect(mockContext.activeExecutionPath.has('parallel-block')).toBe(true)

    // Function-4 should NOT be activated
    expect(mockContext.activeExecutionPath.has('function-4')).toBe(false)

    // The agent inside parallel should NOT be automatically activated
    // It should only be activated when the parallel block executes
    expect(mockContext.activeExecutionPath.has('agent-inside-parallel')).toBe(false)
  })

  it('should prevent parallel blocks from executing when not selected by nested routing', () => {
    // This test simulates the exact scenario from the bug report

    // Step 1: Router selects condition path
    mockContext.blockStates.set('router-1', {
      output: {
        selectedPath: {
          blockId: 'condition-1',
          blockType: BlockType.CONDITION,
          blockTitle: 'Condition 1',
        },
      },
      executed: true,
      executionTime: 0,
    })
    mockContext.executedBlocks.add('router-1')
    pathTracker.updateExecutionPaths(['router-1'], mockContext)

    // Step 2: Condition selects function-4 (NOT parallel)
    mockContext.blockStates.set('condition-1', {
      output: {
        result: 'two',
        stdout: '',
        conditionResult: true,
        selectedPath: {
          blockId: 'function-4',
          blockType: BlockType.FUNCTION,
          blockTitle: 'Function 4',
        },
        selectedConditionId: 'b8f0a33c-a57f-4a36-ac7a-dc9f2b5e6c07-if',
      },
      executed: true,
      executionTime: 0,
    })
    mockContext.executedBlocks.add('condition-1')
    pathTracker.updateExecutionPaths(['condition-1'], mockContext)

    // Step 3: Simulate what the executor's getNextExecutionLayer would do
    const blocksToExecute = workflow.blocks.filter(
      (block) =>
        mockContext.activeExecutionPath.has(block.id) && !mockContext.executedBlocks.has(block.id)
    )

    const blockIds = blocksToExecute.map((b) => b.id)

    // Should only include function-4, NOT parallel-block
    expect(blockIds).toContain('function-4')
    expect(blockIds).not.toContain('parallel-block')
    expect(blockIds).not.toContain('agent-inside-parallel')

    // Verify that parallel block is not in active path
    expect(mockContext.activeExecutionPath.has('parallel-block')).toBe(false)

    // Verify that isInActivePath also returns false for parallel block
    const isParallelActive = pathTracker.isInActivePath('parallel-block', mockContext)
    expect(isParallelActive).toBe(false)
  })
})
```

### Purpose of this file:

This TypeScript file contains unit tests designed to verify the correct behavior of the `PathTracker` class in a workflow execution engine. The tests specifically focus on scenarios involving nested routing, where a Router block directs execution to a Condition block, which in turn selects either a Function block or a Parallel block. The tests ensure that only the blocks selected through this nested routing are added to the active execution path, preventing unintended execution of other blocks, particularly parallel blocks. This fixes a bug where parallel blocks were sometimes being executed when they shouldn't be.

### Simplification of Complex Logic:

The core logic resides in the `PathTracker` class (not directly visible in the provided code, but it's imported and used). The test cases simulate the execution of blocks and then use the `PathTracker` to update the active execution paths.  The complexity is simplified in the tests by:

1.  **Mocking the Execution Context:** The `mockContext` object simulates the workflow's runtime environment, allowing the tests to control the state of blocks and decisions made during execution.
2.  **Defining a Specific Workflow:** The `workflow` object defines a simple workflow structure that is easy to reason about.  It sets up the routing scenarios to be tested.
3.  **Directly Setting Block States:** Instead of executing the blocks, the tests directly set the `blockStates` in the `mockContext` to simulate the output of the blocks.
4.  **Using Clear Assertions:** The `expect` statements clearly define the expected state of the active execution paths and decisions made by the blocks.

### Explanation of each line of code:

**Imports:**

*   `import { beforeEach, describe, expect, it } from 'vitest'`: Imports testing utilities from the Vitest library.
    *   `describe`: Defines a test suite (a group of related tests).
    *   `it`: Defines a single test case.
    *   `expect`: Used for making assertions in the tests.
    *   `beforeEach`:  A function that runs before each test case.
*   `import { BlockType } from '@/executor/consts'`: Imports the `BlockType` enum from a module, which defines the types of blocks in the workflow (e.g., STARTER, ROUTER, CONDITION, FUNCTION, PARALLEL, AGENT).
*   `import { PathTracker } from '@/executor/path/path'`: Imports the `PathTracker` class, which is responsible for managing the active execution paths in the workflow.
*   `import type { ExecutionContext } from '@/executor/types'`: Imports the `ExecutionContext` type definition, which represents the runtime environment for the workflow.
*   `import type { SerializedWorkflow } from '@/serializer/types'`: Imports the `SerializedWorkflow` type definition, which represents the structure of the workflow.

**`describe('Nested Routing Fix - Router → Condition → Target', () => { ... })`:**

*   Defines a test suite named "Nested Routing Fix - Router → Condition → Target". This groups together tests that are all related to fixing a specific bug related to nested routing.

**Variable Declarations:**

*   `let workflow: SerializedWorkflow`: Declares a variable `workflow` to hold the serialized workflow definition.
*   `let pathTracker: PathTracker`: Declares a variable `pathTracker` to hold an instance of the `PathTracker` class.
*   `let mockContext: ExecutionContext`: Declares a variable `mockContext` to hold a mock implementation of the `ExecutionContext`.

**`beforeEach(() => { ... })`:**

*   Sets up the testing environment before each test case.
*   `workflow = { ... }`: Defines the structure of a serialized workflow.  This workflow has a starter block, a router, a condition block, a function block, and a parallel block. The connections define how these blocks are connected.
    *   `version: '2.0'`: Specifies the version of the workflow definition.
    *   `blocks: [...]`: An array of block definitions. Each block has an `id`, `position`, `metadata`, `config`, `inputs`, `outputs`, and `enabled` property.
        *   `id`: A unique identifier for the block.
        *   `metadata`: Contains information about the block, such as its type and name.  `BlockType` is used to specify the type.
        *   `config`: Configuration options for the block.
    *   `connections: [...]`: An array of connection definitions. Each connection has a `source` and `target` property, indicating the source and target blocks of the connection. `sourceHandle` is used for conditional connections, identifying the specific output of the source block that triggers the connection.
    *   `loops: {}`:  An object that would define any looping structures within the workflow. It's empty in this test case.
    *   `parallels: { ... }`: Defines parallel execution configurations.  The 'parallel-block' configuration specifies that 'agent-inside-parallel' will be executed in parallel. The `distribution` array implies the 'agent-inside-parallel' block will run once for each item ('item1', 'item2').
*   `pathTracker = new PathTracker(workflow)`: Creates an instance of the `PathTracker` class, passing in the `workflow` definition.
*   `mockContext = { ... }`: Creates a mock execution context. This object simulates the environment in which the workflow will be executed.
    *   `workflowId`: A unique identifier for the workflow execution.
    *   `blockStates`: A `Map` that stores the state of each block during execution (e.g., its output, whether it has been executed).
    *   `blockLogs`: An array to store the execution logs of each block.
    *   `metadata`: Stores metadata about the workflow execution, such as the duration.
    *   `environmentVariables`: An object to store environment variables accessible to the workflow.
    *   `decisions`: An object to store the decisions made by router and condition blocks, using Maps to associate block IDs with their selected paths.
    *   `loopIterations`: A `Map` to keep track of loop iterations.
    *   `loopItems`: A `Map` to store the current item being processed in a loop.
    *   `completedLoops`: A `Set` to store the IDs of completed loops.
    *   `executedBlocks`: A `Set` to keep track of blocks that have already been executed.
    *   `activeExecutionPath`: A `Set` to track the IDs of the blocks that are currently in the active execution path. This is the crucial set that `PathTracker` manipulates.
    *   `workflow`: A reference to the `workflow` object itself.
*   `mockContext.executedBlocks.add('starter')`: Marks the 'starter' block as executed.
*   `mockContext.activeExecutionPath.add('starter')`: Adds the 'starter' block to the active execution path.
*   `mockContext.activeExecutionPath.add('router-1')`: Adds the 'router-1' block to the active execution path.  This simulates the workflow starting and the first two blocks being in the path.

**`it('should handle nested routing: router selects condition, condition selects function', () => { ... })`:**

*   Defines a test case that verifies the correct behavior of the `PathTracker` when the router selects the condition, and the condition selects a function.
*   `mockContext.blockStates.set('router-1', { ... })`: Simulates the router block executing and selecting the condition block.
    *   `output.selectedPath`: Specifies the block that the router has selected.
*   `mockContext.executedBlocks.add('router-1')`: Marks the router block as executed.
*   `pathTracker.updateExecutionPaths(['router-1'], mockContext)`:  Calls the `updateExecutionPaths` method of the `PathTracker` to update the active execution paths based on the execution of the router block. This is the key method being tested.
*   `expect(mockContext.decisions.router.get('router-1')).toBe('condition-1')`: Asserts that the router's decision to select the condition block has been correctly recorded.
*   `expect(mockContext.activeExecutionPath.has('condition-1')).toBe(true)`: Asserts that the condition block is now in the active execution path.
*   `expect(mockContext.activeExecutionPath.has('function-2')).toBe(false)`: Asserts that the function block connected directly to the router is NOT in the active execution path (because the router chose the condition path).
*   `expect(mockContext.activeExecutionPath.has('parallel-block')).toBe(false)`:  Asserts that the parallel block is NOT in the active execution path. **This is a critical assertion for verifying the bug fix.**
*   `expect(mockContext.activeExecutionPath.has('agent-inside-parallel')).toBe(false)`: Asserts that the agent block inside the parallel block is NOT in the active execution path.  Consistent with the previous assertion.
*   `mockContext.blockStates.set('condition-1', { ... })`: Simulates the condition block executing and selecting the function block.
*   `mockContext.executedBlocks.add('condition-1')`: Marks the condition block as executed.
*   `pathTracker.updateExecutionPaths(['condition-1'], mockContext)`: Calls the `updateExecutionPaths` method of the `PathTracker` to update the active execution paths based on the execution of the condition block.
*   `expect(mockContext.decisions.condition.get('condition-1')).toBe('b8f0a33c-a57f-4a36-ac7a-dc9f2b5e6c07-if')`: Asserts that the condition's decision is recorded correctly.
*   `expect(mockContext.activeExecutionPath.has('function-4')).toBe(true)`: Asserts that the function block selected by the condition is now in the active execution path.
*   `expect(mockContext.activeExecutionPath.has('parallel-block')).toBe(false)`: Asserts that the parallel block is still NOT in the active execution path. **Again, critical to verify the bug fix.**
*   `expect(mockContext.activeExecutionPath.has('agent-inside-parallel')).toBe(false)`: Asserts that the agent inside the parallel block is also still NOT in the active execution path.

**`it('should handle nested routing: router selects condition, condition selects parallel', () => { ... })`:**

*   This test case is similar to the previous one, but it verifies the case where the condition block selects the parallel block instead of the function block. The assertions are adjusted to reflect this different path.

**`it('should prevent parallel blocks from executing when not selected by nested routing', () => { ... })`:**

*   This test case is designed to replicate the exact scenario reported in the bug report. It ensures that the `PathTracker` correctly prevents parallel blocks from being added to the active execution path when they are not explicitly selected through the nested routing.
*   `const blocksToExecute = workflow.blocks.filter(...)`: Simulates how the executor would determine the next blocks to execute based on the active execution path and the executed blocks.
*   `const blockIds = blocksToExecute.map((b) => b.id)`: Extracts the IDs of the blocks that would be executed.
*   `expect(blockIds).toContain('function-4')`: Asserts that the `function-4` block *is* in the list of blocks to execute.
*   `expect(blockIds).not.toContain('parallel-block')`: Asserts that the `parallel-block` is *not* in the list of blocks to execute. **This is the key assertion to demonstrate the bug fix.**
*   `expect(blockIds).not.toContain('agent-inside-parallel')`: Asserts that the `agent-inside-parallel` block is *not* in the list.
*   `expect(mockContext.activeExecutionPath.has('parallel-block')).toBe(false)`: Double checks that the parallel block is NOT in the active execution path.
*   `const isParallelActive = pathTracker.isInActivePath('parallel-block', mockContext)`: Calls a method of the `PathTracker` to directly check if a block is in the active path.
*   `expect(isParallelActive).toBe(false)`: Again, assert that the parallel block is *not* in the active path, using a different method to verify.

In summary, this file provides thorough unit tests to ensure that the `PathTracker` correctly manages active execution paths in complex routing scenarios, specifically addressing and preventing a bug that could lead to unintended execution of parallel blocks. The tests use mocking and clear assertions to effectively verify the expected behavior.
