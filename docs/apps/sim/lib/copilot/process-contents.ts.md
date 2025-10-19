```typescript
import { db } from '@sim/db'
import { copilotChats, document, knowledgeBase, templates } from '@sim/db/schema'
import { and, eq, isNull } from 'drizzle-orm'
import { createLogger } from '@/lib/logs/console/logger'
import { loadWorkflowFromNormalizedTables } from '@/lib/workflows/db-helpers'
import { sanitizeForCopilot } from '@/lib/workflows/json-sanitizer'
import type { ChatContext } from '@/stores/copilot/types'

// Defines the possible types of context that can be provided to the agent.  These types help the agent understand the nature of the 'content' being provided.
export type AgentContextType =
  'past_chat' // Refers to previous conversation history.
  | 'workflow' // Represents a workflow definition.
  | 'current_workflow' // Represents the workflow currently being worked on.
  | 'blocks' // Represents metadata about individual blocks.
  | 'logs' // Represents execution logs.
  | 'knowledge' // Represents information from a knowledge base.
  | 'templates' // Represents a workflow template.
  | 'workflow_block' //Represents a specific block within a workflow.
  | 'docs' // Represents documentation content.

// Defines the structure of a context object that will be passed to the agent.
export interface AgentContext {
  type: AgentContextType // The type of context (e.g., 'past_chat', 'workflow').
  tag: string // A short, human-readable identifier for the context.  This is often used to refer to the context in the prompt.
  content: string // The actual data/information associated with this context.
}

// Creates a logger instance for this module, using the name 'ProcessContents'.
const logger = createLogger('ProcessContents')

/**
 * Processes an array of ChatContext objects to create an array of AgentContext objects.
 * This is the client-side variant.
 *
 * @param contexts - An array of ChatContext objects, or undefined.  Each ChatContext describes a source of information.
 * @returns A promise that resolves to an array of AgentContext objects.  Each AgentContext is a processed and formatted version of the original ChatContext, ready to be used by the agent.
 */
export async function processContexts(
  contexts: ChatContext[] | undefined
): Promise<AgentContext[]> {
  // If the input is not an array or is an empty array, return an empty array.
  if (!Array.isArray(contexts) || contexts.length === 0) return []

  // Maps each ChatContext object in the input array to an asynchronous task.
  const tasks = contexts.map(async (ctx) => {
    try {
      // Processes a past chat based on its ID.
      if (ctx.kind === 'past_chat') {
        return await processPastChatViaApi(ctx.chatId, ctx.label ? `@${ctx.label}` : '@')
      }

      // Processes a workflow based on its ID.
      if ((ctx.kind === 'workflow' || ctx.kind === 'current_workflow') && ctx.workflowId) {
        return await processWorkflowFromDb(
          ctx.workflowId,
          ctx.label ? `@${ctx.label}` : '@',
          ctx.kind
        )
      }

      // Processes a knowledge base based on its ID.
      if (ctx.kind === 'knowledge' && (ctx as any).knowledgeId) {
        return await processKnowledgeFromDb(
          (ctx as any).knowledgeId,
          ctx.label ? `@${ctx.label}` : '@'
        )
      }

      // Processes a block based on its ID.
      if (ctx.kind === 'blocks' && (ctx as any).blockId) {
        return await processBlockMetadata((ctx as any).blockId, ctx.label ? `@${ctx.label}` : '@')
      }

      // Processes a template based on its ID.
      if (ctx.kind === 'templates' && (ctx as any).templateId) {
        return await processTemplateFromDb(
          (ctx as any).templateId,
          ctx.label ? `@${ctx.label}` : '@'
        )
      }

      // Processes execution logs based on its ID.
      if (ctx.kind === 'logs' && (ctx as any).executionId) {
        return await processExecutionLogFromDb(
          (ctx as any).executionId,
          ctx.label ? `@${ctx.label}` : '@'
        )
      }

      // Processes a workflow block based on its workflow and block IDs.
      if (ctx.kind === 'workflow_block' && ctx.workflowId && (ctx as any).blockId) {
        return await processWorkflowBlockFromDb(ctx.workflowId, (ctx as any).blockId, ctx.label)
      }

      // If the context kind is not supported, return null.
      return null
    } catch (error) {
      // Logs any errors that occur during context processing.
      logger.error('Failed processing context', { ctx, error })
      return null
    }
  })

  // Waits for all the asynchronous tasks to complete.
  const results = await Promise.all(tasks)

  // Filters out any null results and casts the remaining results to AgentContext.
  return results.filter((r): r is AgentContext => !!r) as AgentContext[]
}

/**
 * Processes an array of ChatContext objects to create an array of AgentContext objects.
 * This is the server-side variant.
 *
 * @param contexts - An array of ChatContext objects, or undefined.  Each ChatContext describes a source of information.
 * @param userId - The ID of the user associated with the context.
 * @param userMessage - The user's message (optional).  This is used for document searching.
 * @returns A promise that resolves to an array of AgentContext objects.  Each AgentContext is a processed and formatted version of the original ChatContext, ready to be used by the agent.
 */
export async function processContextsServer(
  contexts: ChatContext[] | undefined,
  userId: string,
  userMessage?: string
): Promise<AgentContext[]> {
  // If the input is not an array or is an empty array, return an empty array.
  if (!Array.isArray(contexts) || contexts.length === 0) return []

  // Maps each ChatContext object in the input array to an asynchronous task.
  const tasks = contexts.map(async (ctx) => {
    try {
      // Processes a past chat based on its ID and user ID.
      if (ctx.kind === 'past_chat' && ctx.chatId) {
        return await processPastChatFromDb(ctx.chatId, userId, ctx.label ? `@${ctx.label}` : '@')
      }

      // Processes a workflow based on its ID.
      if ((ctx.kind === 'workflow' || ctx.kind === 'current_workflow') && ctx.workflowId) {
        return await processWorkflowFromDb(
          ctx.workflowId,
          ctx.label ? `@${ctx.label}` : '@',
          ctx.kind
        )
      }

      // Processes a knowledge base based on its ID.
      if (ctx.kind === 'knowledge' && (ctx as any).knowledgeId) {
        return await processKnowledgeFromDb(
          (ctx as any).knowledgeId,
          ctx.label ? `@${ctx.label}` : '@'
        )
      }

      // Processes a block based on its ID.
      if (ctx.kind === 'blocks' && (ctx as any).blockId) {
        return await processBlockMetadata((ctx as any).blockId, ctx.label ? `@${ctx.label}` : '@')
      }

      // Processes a template based on its ID.
      if (ctx.kind === 'templates' && (ctx as any).templateId) {
        return await processTemplateFromDb(
          (ctx as any).templateId,
          ctx.label ? `@${ctx.label}` : '@'
        )
      }

      // Processes execution logs based on its ID.
      if (ctx.kind === 'logs' && (ctx as any).executionId) {
        return await processExecutionLogFromDb(
          (ctx as any).executionId,
          ctx.label ? `@${ctx.label}` : '@'
        )
      }

      // Processes a workflow block based on its workflow and block IDs.
      if (ctx.kind === 'workflow_block' && ctx.workflowId && (ctx as any).blockId) {
        return await processWorkflowBlockFromDb(ctx.workflowId, (ctx as any).blockId, ctx.label)
      }

      // Processes documentation content by searching the documentation.
      if (ctx.kind === 'docs') {
        try {
          // Dynamically imports the searchDocumentationServerTool.
          const { searchDocumentationServerTool } = await import(
            '@/lib/copilot/tools/server/docs/search-documentation'
          )
          // Determines the search query based on the user message or context label.
          const rawQuery = (userMessage || '').trim() || ctx.label || 'Sim documentation'
          // Sanitizes the search query for documentation search.
          const query = sanitizeMessageForDocs(rawQuery, contexts)
          // Executes the documentation search tool.
          const res = await searchDocumentationServerTool.execute({ query, topK: 10 })
          // Stringifies the search results into a JSON string.
          const content = JSON.stringify(res?.results || [])
          // Returns an AgentContext object with the search results.
          return { type: 'docs', tag: ctx.label ? `@${ctx.label}` : '@', content }
        } catch (e) {
          // Logs any errors that occur during documentation context processing.
          logger.error('Failed to process docs context', e)
          return null
        }
      }

      // If the context kind is not supported, return null.
      return null
    } catch (error) {
      // Logs any errors that occur during context processing.
      logger.error('Failed processing context (server)', { ctx, error })
      return null
    }
  })
  // Waits for all the asynchronous tasks to complete.
  const results = await Promise.all(tasks)
  // Filters out any null results or results with empty content strings.
  const filtered = results.filter(
    (r): r is AgentContext => !!r && typeof r.content === 'string' && r.content.trim().length > 0
  )

  // Logs information about the processed contexts.
  logger.info('Processed contexts (server)', {
    totalRequested: contexts.length,
    totalProcessed: filtered.length,
    kinds: Array.from(filtered.reduce((s, r) => s.add(r.type), new Set<string>())),
  })
  // Returns the filtered array of AgentContext objects.
  return filtered
}

/**
 * Escapes regular expression special characters in a string.
 *
 * @param input - The string to escape.
 * @returns The escaped string.
 */
function escapeRegExp(input: string): string {
  return input.replace(/[.*+?^${}()|[\]\\]/g, '\\$&')
}

/**
 * Sanitizes a message for documentation search by removing or modifying mentions.
 *
 * @param rawMessage - The original message.
 * @param contexts - An array of ChatContext objects, or undefined.
 * @returns The sanitized message.
 */
function sanitizeMessageForDocs(rawMessage: string, contexts: ChatContext[] | undefined): string {
  // If the message is empty, return an empty string.
  if (!rawMessage) return ''

  // If there are no contexts, remove all mentions.
  if (!Array.isArray(contexts) || contexts.length === 0) {
    // No context mapping; conservatively strip all @mentions-like tokens
    const stripped = rawMessage
      .replace(/(^|\s)@([^\s]+)/g, ' ')
      .replace(/\s{2,}/g, ' ')
      .trim()
    return stripped
  }

  // Gather labels by kind
  const blockLabels = new Set(
    contexts
      .filter((c) => c.kind === 'blocks')
      .map((c) => c.label)
      .filter((l): l is string => typeof l === 'string' && l.length > 0)
  )
  const nonBlockLabels = new Set(
    contexts
      .filter((c) => c.kind !== 'blocks')
      .map((c) => c.label)
      .filter((l): l is string => typeof l === 'string' && l.length > 0)
  )

  let result = rawMessage

  // 1) Remove all non-block mentions entirely
  for (const label of nonBlockLabels) {
    const pattern = new RegExp(`(^|\\s)@${escapeRegExp(label)}(?!\\S)`, 'g')
    result = result.replace(pattern, ' ')
  }

  // 2) For block mentions, strip the '@' but keep the block name
  for (const label of blockLabels) {
    const pattern = new RegExp(`@${escapeRegExp(label)}(?!\\S)`, 'g')
    result = result.replace(pattern, label)
  }

  // 3) Remove any remaining @mentions (unknown or not in contexts)
  result = result.replace(/(^|\s)@([^\s]+)/g, ' ')

  // Normalize whitespace
  result = result.replace(/\s{2,}/g, ' ').trim()
  return result
}

/**
 * Processes a past chat from the database.
 *
 * @param chatId - The ID of the chat.
 * @param userId - The ID of the user.
 * @param tag - The tag to use for the context.
 * @returns A promise that resolves to an AgentContext object, or null if the chat is not found.
 */
async function processPastChatFromDb(
  chatId: string,
  userId: string,
  tag: string
): Promise<AgentContext | null> {
  try {
    // Selects the messages from the copilotChats table based on the chat ID and user ID.
    const rows = await db
      .select({ messages: copilotChats.messages })
      .from(copilotChats)
      .where(and(eq(copilotChats.id, chatId), eq(copilotChats.userId, userId)))
      .limit(1)

    // Extracts the messages from the result.
    const messages = Array.isArray(rows?.[0]?.messages) ? (rows[0] as any).messages : []

    // Formats the messages into a single string.
    const content = messages
      .map((m: any) => {
        const role = m.role || 'user'
        let text = ''
        if (Array.isArray(m.contentBlocks) && m.contentBlocks.length > 0) {
          text = m.contentBlocks
            .filter((b: any) => b?.type === 'text')
            .map((b: any) => String(b.content || ''))
            .join('')
            .trim()
        }
        if (!text && typeof m.content === 'string') text = m.content
        return `${role}: ${text}`.trim()
      })
      .filter((s: string) => s.length > 0)
      .join('\n')

    // Logs information about the processed chat.
    logger.info('Processed past_chat context from DB', {
      chatId,
      length: content.length,
      lines: content ? content.split('\n').length : 0,
    })

    // Returns an AgentContext object with the formatted chat messages.
    return { type: 'past_chat', tag, content }
  } catch (error) {
    // Logs any errors that occur during chat processing.
    logger.error('Error processing past chat from db', { chatId, error })
    return null
  }
}

/**
 * Processes a workflow from the database.
 *
 * @param workflowId - The ID of the workflow.
 * @param tag - The tag to use for the context.
 * @param kind - The kind of workflow ('workflow' or 'current_workflow').
 * @returns A promise that resolves to an AgentContext object, or null if the workflow is not found.
 */
async function processWorkflowFromDb(
  workflowId: string,
  tag: string,
  kind: 'workflow' | 'current_workflow' = 'workflow'
): Promise<AgentContext | null> {
  try {
    // Loads the workflow data from the database.
    const normalized = await loadWorkflowFromNormalizedTables(workflowId)

    // If the workflow is not found, log a warning and return null.
    if (!normalized) {
      logger.warn('No normalized workflow data found', { workflowId })
      return null
    }

    // Extracts the workflow state from the normalized data.
    const workflowState = {
      blocks: normalized.blocks || {},
      edges: normalized.edges || [],
      loops: normalized.loops || {},
      parallels: normalized.parallels || {},
    }

    // Sanitizes the workflow state for copilot.
    const sanitizedState = sanitizeForCopilot(workflowState)

    // Converts the sanitized workflow state to a JSON string.
    const content = JSON.stringify(sanitizedState, null, 2)

    // Logs information about the processed workflow.
    logger.info('Processed sanitized workflow context', {
      workflowId,
      blocks: Object.keys(sanitizedState.blocks || {}).length,
    })

    // Returns an AgentContext object with the formatted workflow data.
    return { type: kind, tag, content }
  } catch (error) {
    // Logs any errors that occur during workflow processing.
    logger.error('Error processing workflow context', { workflowId, error })
    return null
  }
}

/**
 * Processes a past chat by fetching it from an API endpoint.
 * @deprecated use processPastChatFromDb instead
 * @param chatId - The ID of the chat to process.
 * @param tagOverride - An optional tag to override the default tag.
 * @returns A promise that resolves to an AgentContext object, or null if the chat cannot be fetched.
 */
async function processPastChat(chatId: string, tagOverride?: string): Promise<AgentContext | null> {
  try {
    // Fetches the chat data from the API endpoint.
    const resp = await fetch(`/api/copilot/chat/${encodeURIComponent(chatId)}`)

    // If the API request fails, log an error and return null.
    if (!resp.ok) {
      logger.error('Failed to fetch past chat', { chatId, status: resp.status })
      return null
    }

    // Parses the response body as JSON.
    const data = await resp.json()

    // Extracts the messages from the parsed JSON data.
    const messages = Array.isArray(data?.chat?.messages) ? data.chat.messages : []

    // Formats the messages into a single string.
    const content = messages
      .map((m: any) => {
        const role = m.role || 'user'
        // Prefer contentBlocks text if present (joins text blocks), else use content
        let text = ''
        if (Array.isArray(m.contentBlocks) && m.contentBlocks.length > 0) {
          text = m.contentBlocks
            .filter((b: any) => b?.type === 'text')
            .map((b: any) => String(b.content || ''))
            .join('')
            .trim()
        }
        if (!text && typeof m.content === 'string') text = m.content
        return `${role}: ${text}`.trim()
      })
      .filter((s: string) => s.length > 0)
      .join('\n')

    // Logs information about the processed chat.
    logger.info('Processed past_chat context via API', { chatId, length: content.length })

    // Returns an AgentContext object with the formatted chat messages.
    return { type: 'past_chat', tag: tagOverride || '@', content }
  } catch (error) {
    // Logs any errors that occur during chat processing.
    logger.error('Error processing past chat', { chatId, error })
    return null
  }
}

// Back-compat alias; used by processContexts above
async function processPastChatViaApi(chatId: string, tag?: string) {
  return processPastChat(chatId, tag)
}

/**
 * Processes a knowledge base from the database.
 *
 * @param knowledgeBaseId - The ID of the knowledge base.
 * @param tag - The tag to use for the context.
 * @returns A promise that resolves to an AgentContext object, or null if the knowledge base is not found.
 */
async function processKnowledgeFromDb(
  knowledgeBaseId: string,
  tag: string
): Promise<AgentContext | null> {
  try {
    // Load KB metadata
    const kbRows = await db
      .select({
        id: knowledgeBase.id,
        name: knowledgeBase.name,
        updatedAt: knowledgeBase.updatedAt,
      })
      .from(knowledgeBase)
      .where(and(eq(knowledgeBase.id, knowledgeBaseId), isNull(knowledgeBase.deletedAt)))
      .limit(1)
    const kb = kbRows?.[0]
    if (!kb) return null

    // Load up to 20 recent doc filenames
    const docRows = await db
      .select({ filename: document.filename })
      .from(document)
      .where(and(eq(document.knowledgeBaseId, knowledgeBaseId), isNull(document.deletedAt)))
      .limit(20)

    const sampleDocuments = docRows.map((d: any) => d.filename).filter(Boolean)
    // We don't have total via this quick select; fallback to sample count
    const summary = {
      id: kb.id,
      name: kb.name,
      docCount: sampleDocuments.length,
      sampleDocuments,
    }
    const content = JSON.stringify(summary)
    return { type: 'knowledge', tag, content }
  } catch (error) {
    logger.error('Error processing knowledge context (db)', { knowledgeBaseId, error })
    return null
  }
}

/**
 * Processes block metadata.
 *
 * @param blockId - The ID of the block.
 * @param tag - The tag to use for the context.
 * @returns A promise that resolves to an AgentContext object, or null if the block is not found.
 */
async function processBlockMetadata(blockId: string, tag: string): Promise<AgentContext | null> {
  try {
    // Reuse registry to match get_blocks_metadata tool result
    const { registry: blockRegistry } = await import('@/blocks/registry')
    const { tools: toolsRegistry } = await import('@/tools/registry')
    const SPECIAL_BLOCKS_METADATA: Record<string, any> = {}

    let metadata: any = {}
    if ((SPECIAL_BLOCKS_METADATA as any)[blockId]) {
      metadata = { ...(SPECIAL_BLOCKS_METADATA as any)[blockId] }
      metadata.tools = metadata.tools?.access || []
    } else {
      const blockConfig: any = (blockRegistry as any)[blockId]
      if (!blockConfig) {
        return null
      }
      metadata = {
        id: blockId,
        name: blockConfig.name || blockId,
        description: blockConfig.description || '',
        longDescription: blockConfig.longDescription,
        category: blockConfig.category,
        bgColor: blockConfig.bgColor,
        inputs: blockConfig.inputs || {},
        outputs: blockConfig.outputs || {},
        tools: blockConfig.tools?.access || [],
        hideFromToolbar: blockConfig.hideFromToolbar,
      }
      if (blockConfig.subBlocks && Array.isArray(blockConfig.subBlocks)) {
        metadata.subBlocks = (blockConfig.subBlocks as any[]).map((sb: any) => ({
          id: sb.id,
          name: sb.name,
          type: sb.type,
          description: sb.description,
          default: sb.default,
          options: Array.isArray(sb.options) ? sb.options : [],
        }))
      } else {
        metadata.subBlocks = []
      }
    }

    if (Array.isArray(metadata.tools) && metadata.tools.length > 0) {
      metadata.toolDetails = {}
      for (const toolId of metadata.tools) {
        const tool = (toolsRegistry as any)[toolId]
        if (tool) {
          metadata.toolDetails[toolId] = { name: tool.name, description: tool.description }
        }
      }
    }

    const content = JSON.stringify({ metadata })
    return { type: 'blocks', tag, content }
  } catch (error) {
    logger.error('Error processing block metadata', { blockId, error })
    return null
  }
}

/**
 * Processes a template from the database.
 *
 * @param templateId - The ID of the template.
 * @param tag - The tag to use for the context.
 * @returns A promise that resolves to an AgentContext object, or null if the template is not found.
 */
async function processTemplateFromDb(
  templateId: string,
  tag: string
): Promise<AgentContext | null> {
  try {
    // Selects template details from the database based on the template ID.
    const rows = await db
      .select({
        id: templates.id,
        name: templates.name,
        description: templates.description,
        category: templates.category,
        author: templates.author,
        stars: templates.stars,
        state: templates.state,
      })
      .from(templates)
      .where(eq(templates.id, templateId))
      .limit(1)

    // Extracts the template from the result.
    const t = rows?.[0]
    if (!t) return null

    // Extracts the workflow state from the template.
    const workflowState = (t as any).state || {}

    const summary = {
      id: t.id,
      name: t.name,
      description: t.description || '',
      category: t.category,
      author: t.author,
      stars: t.stars || 0,
      workflow: workflowState,
    }
    // Converts the template summary to a JSON string.
    const content = JSON.stringify(summary)

    // Returns an AgentContext object with the formatted template data.
    return { type: 'templates', tag, content }
  } catch (error) {
    // Logs any errors that occur during template processing.
    logger.error('Error processing template context (db)', { templateId, error })
    return null
  }
}

/**
 * Processes a workflow block from the database.
 *
 * @param workflowId - The ID of the workflow.
 * @param blockId - The ID of the block.
 * @param label - An optional label for the block.
 * @returns A promise that resolves to an AgentContext object, or null if the workflow or block is not found.
 */
async function processWorkflowBlockFromDb(
  workflowId: string,
  blockId: string,
  label?: string
): Promise<AgentContext | null> {
  try {
    // Loads the workflow data from the database.
    const normalized = await loadWorkflowFromNormalizedTables(workflowId)

    // If the workflow is not found, return null.
    if (!normalized) return null

    // Extracts the block from the workflow data.
    const block = (normalized.blocks as any)[blockId]

    // If the block is not found, return null.
    if (!block) return null

    // Creates a tag for the block.
    const tag = label ? `@${label} in Workflow` : `@${block.name || blockId} in Workflow`

    // Build content: isolate the block and include its subBlocks fully
    const contentObj = {
      workflowId,
      block: block,
    }
    // Converts the block data to a JSON string.
    const content = JSON.stringify(contentObj)
    // Returns an AgentContext object with the formatted block data.
    return { type: 'workflow_block', tag, content }
  } catch (error) {
    // Logs any errors that occur during block processing.
    logger.error('Error processing workflow_block context', { workflowId, blockId, error })
    return null
  }
}

/**
 * Processes an execution log from the database.
 *
 * @param executionId - The ID of the execution log.
 * @param tag - The tag to use for the context.
 * @returns A promise that resolves to an AgentContext object, or null if the execution log is not found.
 */
async function processExecutionLogFromDb(
  executionId: string,
  tag: string
): Promise<AgentContext | null> {
  try {
    const { workflowExecutionLogs, workflow } = await import('@sim/db/schema')
    const { db } = await import('@sim/db')
    const rows = await db
      .select({
        id: workflowExecutionLogs.id,
        workflowId: workflowExecutionLogs.workflowId,
        executionId: workflowExecutionLogs.executionId,
        level: workflowExecutionLogs.level,
        trigger: workflowExecutionLogs.trigger,
        startedAt: workflowExecutionLogs.startedAt,
        endedAt: workflowExecutionLogs.endedAt,
        totalDurationMs: workflowExecutionLogs.totalDurationMs,
        executionData: workflowExecutionLogs.executionData,
        cost: workflowExecutionLogs.cost,
        workflowName: workflow.name,
      })
      .from(workflowExecutionLogs)
      .innerJoin(workflow, eq(workflowExecutionLogs.workflowId, workflow.id))
      .where(eq(workflowExecutionLogs.executionId, executionId))
      .limit(1)

    const log = rows?.[0] as any
    if (!log) return null

    const summary = {
      id: log.id,
      workflowId: log.workflowId,
      executionId: log.executionId,
      level: log.level,
      trigger: log.trigger,
      startedAt: log.startedAt?.toISOString?.() || String(log.startedAt),
      endedAt: log.endedAt?.toISOString?.() || (log.endedAt ? String(log.endedAt) : null),
      totalDurationMs: log.totalDurationMs ?? null,
      workflowName: log.workflowName || '',
      // Include trace spans and any available details without being huge
      executionData: log.executionData
        ? {
            traceSpans: (log.executionData as any).traceSpans || undefined,
            errorDetails: (log.executionData as any).errorDetails || undefined,
          }
        : undefined,
      cost: log.cost || undefined,
    }

    const content = JSON.stringify(summary)
    return { type: 'logs', tag, content }
  } catch (error) {
    logger.error('Error processing execution log context (db)', { executionId, error })
    return null
  }
}
```

### Purpose of this file:

The primary purpose of this file is to process various types of context data, format it, and prepare it for consumption by an agent (likely an AI agent or chatbot). It defines functions to fetch data related to past chats, workflows, knowledge bases, blocks, templates, logs, and documentation, and transform them into a standardized `AgentContext` format. This allows the agent to access and understand relevant information to enhance its responses and actions. The file provides two main entry points: a client-side variant (`processContexts`) and a server-side variant (`processContextsServer`).  The server-side version is more secure as it can access the database directly and includes user ID validation.  It also includes a utility function `sanitizeMessageForDocs` which is used to clean user input before it is used to search the documentation, preventing injection attacks and improving search accuracy.

### Simplification of Complex Logic:

The code simplifies complex logic through the following approaches:

1.  **Abstraction with Functions:** Each type of context (past chat, workflow, etc.) has its processing logic encapsulated in a separate function (`processPastChatFromDb`, `processWorkflowFromDb`, etc.). This makes the main `processContexts` functions more readable and maintainable.
2.  **Consistent Error Handling:** Each processing function includes a `try...catch` block to handle potential errors. This prevents errors in one context from crashing the entire processing pipeline. The errors are logged for debugging purposes.
3.  **Standardized Output:** The `AgentContext` interface provides a consistent data structure for all context types. This simplifies the agent's logic for accessing and using the context data.
4.  **Asynchronous Processing:**  Using `Promise.all` allows for concurrent processing of different contexts, improving performance.
5.  **Filtering and Type Assertions:** The code uses `.filter((r): r is AgentContext => !!r)` to remove `null` values and simultaneously assert the correct type, ensuring type safety.
6.  **Helper Functions:** The `sanitizeMessageForDocs` and `escapeRegExp` functions handle specific tasks related to sanitizing user input, making the code more modular and readable.

### Explanation of each line of code:

```typescript
import { db } from '@sim/db'
```

*   **Purpose:** Imports the `db` object, which is likely a database connection or client from the `@sim/db` module. This allows the code to interact with the database.

```typescript
import { copilotChats, document, knowledgeBase, templates } from '@sim/db/schema'
```

*   **Purpose:** Imports specific schema definitions (tables or data models) from the `@sim/db/schema` module. These definitions are used to query and manipulate data in the database. `copilotChats`, `document`, `knowledgeBase`, and `templates` likely represent tables or collections in the database.

```typescript
import { and, eq, isNull } from 'drizzle-orm'
```

*   **Purpose:** Imports helper functions from the `drizzle-orm` library for constructing database queries.  These are used to build `WHERE` clauses with logical `AND` conditions, equality (`eq`), and checking for `NULL` values (`isNull`).

```typescript
import { createLogger } from '@/lib/logs/console/logger'
```

*   **Purpose:** Imports a function `createLogger` from a local module (`@/lib/logs/console/logger`). This function is used to create a logger instance for logging messages to the console or other logging destinations.

```typescript
import { loadWorkflowFromNormalizedTables } from '@/lib/workflows/db-helpers'
```

*   **Purpose:** Imports a function `loadWorkflowFromNormalizedTables` from a local module (`@/lib/workflows/db-helpers`). This function is responsible for loading workflow data from the database, likely from multiple related tables, and normalizing it into a usable format.

```typescript
import { sanitizeForCopilot } from '@/lib/workflows/json-sanitizer'
```

*   **Purpose:** Imports a function `sanitizeForCopilot` from a local module (`@/lib/workflows/json