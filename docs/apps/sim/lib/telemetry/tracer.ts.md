## Code Explanation: OpenTelemetry Integration for Sim Execution Pipeline

This TypeScript file provides integration between a simulation execution pipeline and OpenTelemetry (OTel) for tracing purposes. It essentially bridges the gap between the existing logging system (which uses `TraceSpan` and `BlockLog` data structures) and OTel's span-based tracing.  The goal is to convert the simulator's execution logs into a format that OTel can understand and visualize, providing insights into the performance and behavior of the simulation.

**Purpose of this file:**

-   **Integrate OpenTelemetry with the Sim Execution Pipeline:**  Connect the simulation's execution logging to OTel, enabling distributed tracing.
-   **Convert `TraceSpan`s to OTel Spans:** Transform the simulation's custom log format (`TraceSpan`) into standard OTel spans.
-   **Provide Semantic Conventions:**  Enrich OTel spans with semantic attributes relevant to the simulation domain (e.g., GenAI attributes).
-   **Enable Real-time Tracing:** Allow tracing of individual block executions as they happen.
-   **Track Platform Events:** Facilitate tracking of important platform events such as workflow creation and knowledge base operations.

**Simplifying Complex Logic:**

The code achieves clarity through:

-   **Configuration-based approach:**  The `BLOCK_TYPE_MAPPING` provides a centralized configuration for how different block types are translated into OTel spans, making it easier to manage and extend the integration.
-   **Helper functions:** Functions like `getTracer`, `createOTelSpanFromTraceSpan`, `createOTelSpansForWorkflowExecution`, `traceBlockExecution`, and `trackPlatformEvent` encapsulate specific tasks, improving readability and reusability.
-   **Descriptive variable names:**  The code uses meaningful variable names that clearly describe the purpose of each variable.
-   **Error handling:**  `try...catch` blocks are used to handle potential errors during span creation and prevent the entire process from crashing.

**Line-by-line Explanation:**

```typescript
/**
 * OpenTelemetry Integration for Sim Execution Pipeline
 *
 * This module integrates OpenTelemetry tracing with the existing execution logging system.
 * It converts TraceSpans and BlockLogs into proper OpenTelemetry spans with semantic conventions.
 *
 * Architecture:
 * - LoggingSession tracks workflow execution start/complete
 * - Executor generates BlockLogs for each block execution
 * - TraceSpans are built from BlockLogs
 * - This module converts TraceSpans -> OpenTelemetry Spans
 *
 * Integration Points:
 * 1. LoggingSession.start() -> Create root workflow span
 * 2. LoggingSession.complete() -> End workflow span with all block spans as children
 * 3. Block execution -> Create span for each block type with proper attributes
 */

import { context, type Span, SpanStatusCode, trace } from '@opentelemetry/api'
import { createLogger } from '@/lib/logs/console/logger'
import type { TraceSpan } from '@/lib/logs/types'

/**
 * GenAI Semantic Convention Attributes
 */
const GenAIAttributes = {
  // System attributes
  SYSTEM: 'gen_ai.system',
  REQUEST_MODEL: 'gen_ai.request.model',
  RESPONSE_MODEL: 'gen_ai.response.model',

  // Token usage
  USAGE_INPUT_TOKENS: 'gen_ai.usage.input_tokens',
  USAGE_OUTPUT_TOKENS: 'gen_ai.usage.output_tokens',
  USAGE_TOTAL_TOKENS: 'gen_ai.usage.total_tokens',

  // Request/Response
  REQUEST_TEMPERATURE: 'gen_ai.request.temperature',
  REQUEST_TOP_P: 'gen_ai.request.top_p',
  REQUEST_MAX_TOKENS: 'gen_ai.request.max_tokens',
  RESPONSE_FINISH_REASON: 'gen_ai.response.finish_reason',

  // Agent-specific
  AGENT_ID: 'gen_ai.agent.id',
  AGENT_NAME: 'gen_ai.agent.name',
  AGENT_TASK: 'gen_ai.agent.task',

  // Workflow-specific
  WORKFLOW_ID: 'gen_ai.workflow.id',
  WORKFLOW_NAME: 'gen_ai.workflow.name',
  WORKFLOW_VERSION: 'gen_ai.workflow.version',
  WORKFLOW_EXECUTION_ID: 'gen_ai.workflow.execution_id',

  // Tool-specific
  TOOL_NAME: 'gen_ai.tool.name',
  TOOL_DESCRIPTION: 'gen_ai.tool.description',

  // Cost tracking
  COST_TOTAL: 'gen_ai.cost.total',
  COST_INPUT: 'gen_ai.cost.input',
  COST_OUTPUT: 'gen_ai.cost.output',
}

const logger = createLogger('OTelIntegration')

// Lazy-load tracer
let _tracer: ReturnType<typeof trace.getTracer> | null = null

function getTracer() {
  if (!_tracer) {
    _tracer = trace.getTracer('sim-ai-platform', '1.0.0')
  }
  return _tracer
}

/**
 * Map block types to OpenTelemetry span kinds and semantic conventions
 */
const BLOCK_TYPE_MAPPING: Record<
  string,
  {
    spanName: string
    spanKind: string
    getAttributes: (traceSpan: TraceSpan) => Record<string, string | number | boolean | undefined>
  }
> = {
  agent: {
    spanName: 'gen_ai.agent.execute',
    spanKind: 'gen_ai.agent',
    getAttributes: (span) => {
      const attrs: Record<string, string | number | boolean | undefined> = {
        [GenAIAttributes.AGENT_ID]: span.blockId || span.id,
        [GenAIAttributes.AGENT_NAME]: span.name,
      }

      if (span.model) {
        attrs[GenAIAttributes.REQUEST_MODEL] = span.model
      }

      if (span.tokens) {
        if (typeof span.tokens === 'number') {
          attrs[GenAIAttributes.USAGE_TOTAL_TOKENS] = span.tokens
        } else {
          attrs[GenAIAttributes.USAGE_INPUT_TOKENS] = span.tokens.input || span.tokens.prompt || 0
          attrs[GenAIAttributes.USAGE_OUTPUT_TOKENS] =
            span.tokens.output || span.tokens.completion || 0
          attrs[GenAIAttributes.USAGE_TOTAL_TOKENS] = span.tokens.total || 0
        }
      }

      if (span.cost) {
        attrs[GenAIAttributes.COST_INPUT] = span.cost.input || 0
        attrs[GenAIAttributes.COST_OUTPUT] = span.cost.output || 0
        attrs[GenAIAttributes.COST_TOTAL] = span.cost.total || 0
      }

      return attrs
    },
  },
  workflow: {
    spanName: 'gen_ai.workflow.execute',
    spanKind: 'gen_ai.workflow',
    getAttributes: (span) => ({
      [GenAIAttributes.WORKFLOW_ID]: span.blockId || 'root',
      [GenAIAttributes.WORKFLOW_NAME]: span.name,
    }),
  },
  tool: {
    spanName: 'gen_ai.tool.call',
    spanKind: 'gen_ai.tool',
    getAttributes: (span) => ({
      [GenAIAttributes.TOOL_NAME]: span.name,
      'tool.id': span.id,
      'tool.duration_ms': span.duration,
    }),
  },
  model: {
    spanName: 'gen_ai.model.request',
    spanKind: 'gen_ai.model',
    getAttributes: (span) => ({
      'model.name': span.name,
      'model.id': span.id,
      'model.duration_ms': span.duration,
    }),
  },
  api: {
    spanName: 'http.client.request',
    spanKind: 'http.client',
    getAttributes: (span) => {
      const input = span.input as { method?: string; url?: string } | undefined
      const output = span.output as { status?: number } | undefined
      return {
        'http.request.method': input?.method || 'GET',
        'http.request.url': input?.url || '',
        'http.response.status_code': output?.status || 0,
        'block.id': span.blockId,
        'block.name': span.name,
      }
    },
  },
  function: {
    spanName: 'function.execute',
    spanKind: 'function',
    getAttributes: (span) => ({
      'function.name': span.name,
      'function.id': span.blockId,
      'function.execution_time_ms': span.duration,
    }),
  },
  router: {
    spanName: 'router.evaluate',
    spanKind: 'router',
    getAttributes: (span) => {
      const output = span.output as { selectedPath?: { blockId?: string } } | undefined
      return {
        'router.name': span.name,
        'router.id': span.blockId,
        'router.selected_path': output?.selectedPath?.blockId,
      }
    },
  },
  condition: {
    spanName: 'condition.evaluate',
    spanKind: 'condition',
    getAttributes: (span) => {
      const output = span.output as { conditionResult?: boolean | string } | undefined
      return {
        'condition.name': span.name,
        'condition.id': span.blockId,
        'condition.result': output?.conditionResult,
      }
    },
  },
  loop: {
    spanName: 'loop.execute',
    spanKind: 'loop',
    getAttributes: (span) => ({
      'loop.name': span.name,
      'loop.id': span.blockId,
      'loop.iterations': span.children?.length || 0,
    }),
  },
  parallel: {
    spanName: 'parallel.execute',
    spanKind: 'parallel',
    getAttributes: (span) => ({
      'parallel.name': span.name,
      'parallel.id': span.blockId,
      'parallel.branches': span.children?.length || 0,
    }),
  },
}

/**
 * Convert a TraceSpan to an OpenTelemetry span
 * Creates a proper OTel span with all the metadata from the trace span
 */
export function createOTelSpanFromTraceSpan(traceSpan: TraceSpan, parentSpan?: Span): Span | null {
  try {
    const tracer = getTracer()

    const blockMapping = BLOCK_TYPE_MAPPING[traceSpan.type] || {
      spanName: `block.${traceSpan.type}`,
      spanKind: 'internal',
      getAttributes: (span: TraceSpan) => ({
        'block.type': span.type,
        'block.id': span.blockId,
        'block.name': span.name,
      }),
    }

    const attributes = {
      ...blockMapping.getAttributes(traceSpan),
      'span.type': traceSpan.type,
      'span.duration_ms': traceSpan.duration,
      'span.status': traceSpan.status,
    }

    const ctx = parentSpan ? trace.setSpan(context.active(), parentSpan) : context.active()

    const span = tracer.startSpan(
      blockMapping.spanName,
      {
        attributes,
        startTime: new Date(traceSpan.startTime),
      },
      ctx
    )

    if (traceSpan.status === 'error') {
      const errorMessage =
        typeof traceSpan.output?.error === 'string'
          ? traceSpan.output.error
          : 'Block execution failed'

      span.setStatus({
        code: SpanStatusCode.ERROR,
        message: errorMessage,
      })

      if (errorMessage && errorMessage !== 'Block execution failed') {
        span.recordException(new Error(errorMessage))
      }
    } else {
      span.setStatus({ code: SpanStatusCode.OK })
    }

    if (traceSpan.children && traceSpan.children.length > 0) {
      for (const childTraceSpan of traceSpan.children) {
        createOTelSpanFromTraceSpan(childTraceSpan, span)
      }
    }

    if (traceSpan.toolCalls && traceSpan.toolCalls.length > 0) {
      for (const toolCall of traceSpan.toolCalls) {
        const toolSpan = tracer.startSpan(
          'gen_ai.tool.call',
          {
            attributes: {
              [GenAIAttributes.TOOL_NAME]: toolCall.name,
              'tool.status': toolCall.status,
              'tool.duration_ms': toolCall.duration || 0,
            },
            startTime: new Date(toolCall.startTime),
          },
          trace.setSpan(context.active(), span)
        )

        if (toolCall.status === 'error') {
          toolSpan.setStatus({
            code: SpanStatusCode.ERROR,
            message: toolCall.error || 'Tool call failed',
          })
          if (toolCall.error) {
            toolSpan.recordException(new Error(toolCall.error))
          }
        } else {
          toolSpan.setStatus({ code: SpanStatusCode.OK })
        }

        toolSpan.end(new Date(toolCall.endTime))
      }
    }

    span.end(new Date(traceSpan.endTime))

    return span
  } catch (error) {
    logger.error('Failed to create OTel span from trace span', {
      error,
      traceSpanId: traceSpan.id,
      traceSpanType: traceSpan.type,
    })
    return null
  }
}

/**
 * Create OpenTelemetry spans for an entire workflow execution
 * This is called from LoggingSession.complete() with the final trace spans
 */
export function createOTelSpansForWorkflowExecution(params: {
  workflowId: string
  workflowName?: string
  executionId: string
  traceSpans: TraceSpan[]
  trigger: string
  startTime: string
  endTime: string
  totalDurationMs: number
  status: 'success' | 'error'
  error?: string
}): void {
  try {
    const tracer = getTracer()

    const rootSpan = tracer.startSpan(
      'gen_ai.workflow.execute',
      {
        attributes: {
          [GenAIAttributes.WORKFLOW_ID]: params.workflowId,
          [GenAIAttributes.WORKFLOW_NAME]: params.workflowName || params.workflowId,
          [GenAIAttributes.WORKFLOW_EXECUTION_ID]: params.executionId,
          'workflow.trigger': params.trigger,
          'workflow.duration_ms': params.totalDurationMs,
        },
        startTime: new Date(params.startTime),
      },
      context.active()
    )

    if (params.status === 'error') {
      rootSpan.setStatus({
        code: SpanStatusCode.ERROR,
        message: params.error || 'Workflow execution failed',
      })
      if (params.error) {
        rootSpan.recordException(new Error(params.error))
      }
    } else {
      rootSpan.setStatus({ code: SpanStatusCode.OK })
    }

    for (const traceSpan of params.traceSpans) {
      createOTelSpanFromTraceSpan(traceSpan, rootSpan)
    }

    rootSpan.end(new Date(params.endTime))

    logger.debug('Created OTel spans for workflow execution', {
      workflowId: params.workflowId,
      executionId: params.executionId,
      spanCount: params.traceSpans.length,
    })
  } catch (error) {
    logger.error('Failed to create OTel spans for workflow execution', {
      error,
      workflowId: params.workflowId,
      executionId: params.executionId,
    })
  }
}

/**
 * Create a real-time OpenTelemetry span for a block execution
 * Can be called from block handlers during execution for real-time tracing
 */
export async function traceBlockExecution<T>(
  blockType: string,
  blockId: string,
  blockName: string,
  fn: (span: Span) => Promise<T>
): Promise<T> {
  const tracer = getTracer()

  const blockMapping = BLOCK_TYPE_MAPPING[blockType] || {
    spanName: `block.${blockType}`,
    spanKind: 'internal',
    getAttributes: () => ({}),
  }

  return tracer.startActiveSpan(
    blockMapping.spanName,
    {
      attributes: {
        'block.type': blockType,
        'block.id': blockId,
        'block.name': blockName,
      },
    },
    async (span) => {
      try {
        const result = await fn(span)
        span.setStatus({ code: SpanStatusCode.OK })
        return result
      } catch (error) {
        span.setStatus({
          code: SpanStatusCode.ERROR,
          message: error instanceof Error ? error.message : 'Block execution failed',
        })
        span.recordException(error instanceof Error ? error : new Error(String(error)))
        throw error
      } finally {
        span.end()
      }
    }
  )
}

/**
 * Track platform events (workflow creation, knowledge base operations, etc.)
 */
export function trackPlatformEvent(
  eventName: string,
  attributes: Record<string, string | number | boolean>
): void {
  try {
    const tracer = getTracer()
    const span = tracer.startSpan(eventName, {
      attributes: {
        ...attributes,
        'event.name': eventName,
        'event.timestamp': Date.now(),
      },
    })
    span.setStatus({ code: SpanStatusCode.OK })
    span.end()
  } catch (error) {
    // Silently fail
  }
}
```

1.  **Imports:** Imports necessary modules from the `@opentelemetry/api` package for interacting with OpenTelemetry, a logger, and the custom `TraceSpan` type.
    ```typescript
    import { context, type Span, SpanStatusCode, trace } from '@opentelemetry/api'
    import { createLogger } from '@/lib/logs/console/logger'
    import type { TraceSpan } from '@/lib/logs/types'
    ```

2.  **`GenAIAttributes` Constant:** Defines an object containing semantic attribute keys for various aspects of a GenAI workflow.  These attributes will be attached to OTel spans to provide context and meaning.
    ```typescript
    const GenAIAttributes = { ... }
    ```

3.  **`logger` Constant:** Creates a logger instance for logging messages related to the OTel integration.  This helps in debugging and monitoring.
    ```typescript
    const logger = createLogger('OTelIntegration')
    ```

4.  **`getTracer` Function:** Implements lazy initialization of the OpenTelemetry tracer. This ensures that the tracer is only created when it is needed. The tracer is responsible for creating and managing spans.
    ```typescript
    let _tracer: ReturnType<typeof trace.getTracer> | null = null

    function getTracer() {
      if (!_tracer) {
        _tracer = trace.getTracer('sim-ai-platform', '1.0.0')
      }
      return _tracer
    }
    ```

5.  **`BLOCK_TYPE_MAPPING` Constant:** Defines a mapping between block types in the simulation and OpenTelemetry span configurations.  For each block type (e.g., `agent`, `workflow`, `tool`), it specifies the span name, span kind, and a function to extract attributes from the `TraceSpan`.  This is the core configuration that drives the conversion of simulation logs to OTel spans.
    ```typescript
    const BLOCK_TYPE_MAPPING: Record<
      string,
      {
        spanName: string
        spanKind: string
        getAttributes: (traceSpan: TraceSpan) => Record<string, string | number | boolean | undefined>
      }
    > = { ... }
    ```
    *   `spanName`:  The name of the OTel span to create for this block type.
    *   `spanKind`: Represents the relationship between the span, its parent, and its children in OTel
    *   `getAttributes`:  A function that takes a `TraceSpan` as input and returns an object containing attributes to be added to the OTel span. The returned object is a `Record` where keys are strings and values can be strings, numbers, booleans, or undefined.  This function extracts relevant information from the `TraceSpan` and formats it as OTel attributes, leveraging the `GenAIAttributes` constant for semantic consistency.

6.  **`createOTelSpanFromTraceSpan` Function:** This is the primary function for converting a `TraceSpan` into an OTel span.
    ```typescript
    export function createOTelSpanFromTraceSpan(traceSpan: TraceSpan, parentSpan?: Span): Span | null { ... }
    ```
    *   It retrieves the appropriate span configuration from `BLOCK_TYPE_MAPPING` based on the `traceSpan.type`.
    *   It calls the `getAttributes` function to extract attributes from the `traceSpan`.
    *   It starts a new OTel span using the `tracer.startSpan` method, passing in the span name, attributes, and start time.
    *   It sets the span status to `OK` or `ERROR` based on the `traceSpan.status`.
    *   If the `traceSpan` has children, it recursively calls `createOTelSpanFromTraceSpan` for each child, creating a nested span structure.
    *   If the `traceSpan` has tool calls, it creates spans for these tool calls
    *   It ends the OTel span using the `span.end` method, passing in the end time.
    *   It handles errors during span creation and logs them.
    *   It uses `context` and `trace.setSpan` to properly set the parent-child relationship between spans. If a `parentSpan` is provided, it sets the current context to use that parent. Otherwise, it uses the current active context.

7.  **`createOTelSpansForWorkflowExecution` Function:** Creates OTel spans for an entire workflow execution, taking a root `TraceSpan` and recursively converting all its children into OTel spans.
    ```typescript
    export function createOTelSpansForWorkflowExecution(params: { ... }): void { ... }
    ```
    *   It creates a root span representing the entire workflow execution.
    *   It iterates over the `traceSpans` passed in the `params` and calls `createOTelSpanFromTraceSpan` to convert each `TraceSpan` into an OTel span, making the root span the parent.
    *   Sets the workflow status based on the input parameters.
    *   Logs debugging information about the created spans.
    *   Handles errors during span creation and logs them.

8.  **`traceBlockExecution` Function:** Creates a real-time OTel span for a single block execution. This is intended to be called directly from block handlers during execution.
    ```typescript
    export async function traceBlockExecution<T>(
      blockType: string,
      blockId: string,
      blockName: string,
      fn: (span: Span) => Promise<T>
    ): Promise<T> { ... }
    ```
    *   It starts an active span using `tracer.startActiveSpan`, which automatically manages the span's lifecycle.
    *   It executes the provided function `fn` within the context of the span, passing the span as an argument.
    *   Sets the span status to `OK` or `ERROR` based on the outcome of `fn`.
    *   Records exceptions if `fn` throws an error.
    *   Automatically ends the span when `fn` completes, regardless of whether it succeeds or fails.

9.  **`trackPlatformEvent` Function:** Creates a span for tracking generic platform events.
    ```typescript
    export function trackPlatformEvent(
      eventName: string,
      attributes: Record<string, string | number | boolean>
    ): void { ... }
    ```
    *   It starts and ends a simple span with the given `eventName` and `attributes`.
    *   Includes an `event.timestamp` attribute.
    *   It silently fails if an error occurs during span creation (i.e., it doesn't throw an exception).

**In Summary:**

This file provides a comprehensive integration of OpenTelemetry with a simulation execution pipeline. It leverages a configuration-based approach to map simulation concepts (block types, trace spans) to OTel spans, enabling detailed tracing and monitoring of workflow executions. The code is well-structured, uses descriptive names, and includes error handling to ensure robustness. The functions provided allow for both batch processing of execution logs and real-time tracing of individual block executions.
