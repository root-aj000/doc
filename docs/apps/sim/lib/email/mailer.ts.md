```typescript
import { EmailClient, type EmailMessage } from '@azure/communication-email'
import { Resend } from 'resend'
import { generateUnsubscribeToken, isUnsubscribed } from '@/lib/email/unsubscribe'
import { getFromEmailAddress } from '@/lib/email/utils'
import { env } from '@/lib/env'
import { createLogger } from '@/lib/logs/console/logger'
import { getBaseUrl } from '@/lib/urls/utils'

const logger = createLogger('Mailer')

export type EmailType = 'transactional' | 'marketing' | 'updates' | 'notifications'

export interface EmailAttachment {
  filename: string
  content: string | Buffer
  contentType: string
  disposition?: 'attachment' | 'inline'
}

export interface EmailOptions {
  to: string | string[]
  subject: string
  html?: string
  text?: string
  from?: string
  emailType?: EmailType
  includeUnsubscribe?: boolean
  attachments?: EmailAttachment[]
  replyTo?: string
}

export interface BatchEmailOptions {
  emails: EmailOptions[]
}

export interface SendEmailResult {
  success: boolean
  message: string
  data?: any
}

export interface BatchSendEmailResult {
  success: boolean
  message: string
  results: SendEmailResult[]
  data?: any
}

interface ProcessedEmailData {
  to: string | string[]
  subject: string
  html?: string
  text?: string
  senderEmail: string
  headers: Record<string, string>
  attachments?: EmailAttachment[]
  replyTo?: string
}

const resendApiKey = env.RESEND_API_KEY
const azureConnectionString = env.AZURE_ACS_CONNECTION_STRING

const resend =
  resendApiKey && resendApiKey !== 'placeholder' && resendApiKey.trim() !== ''
    ? new Resend(resendApiKey)
    : null

const azureEmailClient =
  azureConnectionString && azureConnectionString.trim() !== ''
    ? new EmailClient(azureConnectionString)
    : null

/**
 * Check if any email service is configured and available
 */
export function hasEmailService(): boolean {
  return !!(resend || azureEmailClient)
}

export async function sendEmail(options: EmailOptions): Promise<SendEmailResult> {
  try {
    // Check if user has unsubscribed (skip for critical transactional emails)
    if (options.emailType !== 'transactional') {
      const unsubscribeType = options.emailType as 'marketing' | 'updates' | 'notifications'
      // For arrays, check the first email address (batch emails typically go to similar recipients)
      const primaryEmail = Array.isArray(options.to) ? options.to[0] : options.to
      const hasUnsubscribed = await isUnsubscribed(primaryEmail, unsubscribeType)
      if (hasUnsubscribed) {
        logger.info('Email not sent (user unsubscribed):', {
          to: options.to,
          subject: options.subject,
          emailType: options.emailType,
        })
        return {
          success: true,
          message: 'Email skipped (user unsubscribed)',
          data: { id: 'skipped-unsubscribed' },
        }
      }
    }

    // Process email data with unsubscribe tokens and headers
    const processedData = await processEmailData(options)

    // Try Resend first if configured
    if (resend) {
      try {
        return await sendWithResend(processedData)
      } catch (error) {
        logger.warn('Resend failed, attempting Azure Communication Services fallback:', error)
      }
    }

    // Fallback to Azure Communication Services if configured
    if (azureEmailClient) {
      try {
        return await sendWithAzure(processedData)
      } catch (error) {
        logger.error('Azure Communication Services also failed:', error)
        return {
          success: false,
          message: 'Both Resend and Azure Communication Services failed',
        }
      }
    }

    // No email service configured
    logger.info('Email not sent (no email service configured):', {
      to: options.to,
      subject: options.subject,
      from: processedData.senderEmail,
    })
    return {
      success: true,
      message: 'Email logging successful (no email service configured)',
      data: { id: 'mock-email-id' },
    }
  } catch (error) {
    logger.error('Error sending email:', error)
    return {
      success: false,
      message: 'Failed to send email',
    }
  }
}

async function processEmailData(options: EmailOptions): Promise<ProcessedEmailData> {
  const {
    to,
    subject,
    html,
    text,
    from,
    emailType = 'transactional',
    includeUnsubscribe = true,
    attachments,
    replyTo,
  } = options

  const senderEmail = from || getFromEmailAddress()

  // Generate unsubscribe token and add to content
  let finalHtml = html
  let finalText = text
  const headers: Record<string, string> = {}

  if (includeUnsubscribe && emailType !== 'transactional') {
    // For arrays, use the first email for unsubscribe (batch emails typically go to similar recipients)
    const primaryEmail = Array.isArray(to) ? to[0] : to
    const unsubscribeToken = generateUnsubscribeToken(primaryEmail, emailType)
    const baseUrl = getBaseUrl()
    const unsubscribeUrl = `${baseUrl}/unsubscribe?token=${unsubscribeToken}&email=${encodeURIComponent(primaryEmail)}`

    headers['List-Unsubscribe'] = `<${unsubscribeUrl}>`
    headers['List-Unsubscribe-Post'] = 'List-Unsubscribe=One-Click'

    if (html) {
      finalHtml = html.replace(/\{\{UNSUBSCRIBE_TOKEN\}\}/g, unsubscribeToken)
    }
    if (text) {
      finalText = text.replace(/\{\{UNSUBSCRIBE_TOKEN\}\}/g, unsubscribeToken)
    }
  }

  return {
    to,
    subject,
    html: finalHtml,
    text: finalText,
    senderEmail,
    headers,
    attachments,
    replyTo,
  }
}

async function sendWithResend(data: ProcessedEmailData): Promise<SendEmailResult> {
  if (!resend) throw new Error('Resend not configured')

  const fromAddress = data.senderEmail

  const emailData: any = {
    from: fromAddress,
    to: data.to,
    subject: data.subject,
    headers: Object.keys(data.headers).length > 0 ? data.headers : undefined,
  }

  if (data.html) emailData.html = data.html
  if (data.text) emailData.text = data.text
  if (data.replyTo) emailData.replyTo = data.replyTo
  if (data.attachments) {
    emailData.attachments = data.attachments.map((att) => ({
      filename: att.filename,
      content: typeof att.content === 'string' ? att.content : att.content.toString('base64'),
      contentType: att.contentType,
      disposition: att.disposition || 'attachment',
    }))
  }

  const { data: responseData, error } = await resend.emails.send(emailData)

  if (error) {
    throw new Error(error.message || 'Failed to send email via Resend')
  }

  return {
    success: true,
    message: 'Email sent successfully via Resend',
    data: responseData,
  }
}

async function sendWithAzure(data: ProcessedEmailData): Promise<SendEmailResult> {
  if (!azureEmailClient) throw new Error('Azure Communication Services not configured')

  // Azure Communication Services requires at least one content type
  if (!data.html && !data.text) {
    throw new Error('Azure Communication Services requires either HTML or text content')
  }

  // For Azure, use just the email address part (no display name)
  // Azure will use the display name configured in the portal for the sender address
  const senderEmailOnly = data.senderEmail.includes('<')
    ? data.senderEmail.match(/<(.+)>/)?.[1] || data.senderEmail
    : data.senderEmail

  const message: EmailMessage = {
    senderAddress: senderEmailOnly,
    content: data.html
      ? {
          subject: data.subject,
          html: data.html,
        }
      : {
          subject: data.subject,
          plainText: data.text!,
        },
    recipients: {
      to: Array.isArray(data.to)
        ? data.to.map((email) => ({ address: email }))
        : [{ address: data.to }],
    },
    headers: data.headers,
  }

  const poller = await azureEmailClient.beginSend(message)
  const result = await poller.pollUntilDone()

  if (result.status === 'Succeeded') {
    return {
      success: true,
      message: 'Email sent successfully via Azure Communication Services',
      data: { id: result.id },
    }
  }
  throw new Error(`Azure Communication Services failed with status: ${result.status}`)
}

export async function sendBatchEmails(options: BatchEmailOptions): Promise<BatchSendEmailResult> {
  try {
    const results: SendEmailResult[] = []

    // Try Resend first for batch emails if available
    if (resend) {
      try {
        return await sendBatchWithResend(options.emails)
      } catch (error) {
        logger.warn('Resend batch failed, falling back to individual sends:', error)
      }
    }

    // Fallback to individual sends (works with both Azure and Resend)
    logger.info('Sending batch emails individually')
    for (const email of options.emails) {
      try {
        const result = await sendEmail(email)
        results.push(result)
      } catch (error) {
        results.push({
          success: false,
          message: error instanceof Error ? error.message : 'Failed to send email',
        })
      }
    }

    const successCount = results.filter((r) => r.success).length
    return {
      success: successCount === results.length,
      message:
        successCount === results.length
          ? 'All batch emails sent successfully'
          : `${successCount}/${results.length} emails sent successfully`,
      results,
      data: { count: successCount },
    }
  } catch (error) {
    logger.error('Error in batch email sending:', error)
    return {
      success: false,
      message: 'Failed to send batch emails',
      results: [],
    }
  }
}

async function sendBatchWithResend(emails: EmailOptions[]): Promise<BatchSendEmailResult> {
  if (!resend) throw new Error('Resend not configured')

  const results: SendEmailResult[] = []
  const batchEmails = emails.map((email) => {
    const senderEmail = email.from || getFromEmailAddress()
    const emailData: any = {
      from: senderEmail,
      to: email.to,
      subject: email.subject,
    }
    if (email.html) emailData.html = email.html
    if (email.text) emailData.text = email.text
    return emailData
  })

  try {
    const response = await resend.batch.send(batchEmails as any)

    if (response.error) {
      throw new Error(response.error.message || 'Resend batch API error')
    }

    // Success - create results for each email
    batchEmails.forEach((_, index) => {
      results.push({
        success: true,
        message: 'Email sent successfully via Resend batch',
        data: { id: `batch-${index}` },
      })
    })

    return {
      success: true,
      message: 'All batch emails sent successfully via Resend',
      results,
      data: { count: results.length },
    }
  } catch (error) {
    logger.error('Resend batch send failed:', error)
    throw error // Let the caller handle fallback
  }
}
```

## Detailed Explanation of the Code:

This TypeScript file defines a module for sending emails using either Resend or Azure Communication Services as the backend. It provides functions for sending single emails and batch emails, handling unsubscribe functionality, and managing attachments.  The module prioritizes Resend and falls back to Azure Communication Services if Resend fails or is not configured. If neither is configured, it logs the email details without sending.

### 1. Imports:

```typescript
import { EmailClient, type EmailMessage } from '@azure/communication-email'
import { Resend } from 'resend'
import { generateUnsubscribeToken, isUnsubscribed } from '@/lib/email/unsubscribe'
import { getFromEmailAddress } from '@/lib/email/utils'
import { env } from '@/lib/env'
import { createLogger } from '@/lib/logs/console/logger'
import { getBaseUrl } from '@/lib/urls/utils'
```

-   `EmailClient`, `EmailMessage` from `@azure/communication-email`:  These are classes and types from the Azure Communication Services Email SDK, used for sending emails through Azure.  `EmailClient` is the main client for interacting with the Azure Email service. `EmailMessage` is an interface that defines the structure of the email to be sent.
-   `Resend` from `resend`: This is the main class from the Resend SDK, used for sending emails through Resend.
-   `generateUnsubscribeToken`, `isUnsubscribed` from `'@/lib/email/unsubscribe'`: These are custom functions for handling email unsubscriptions. `generateUnsubscribeToken` creates a unique token for users to unsubscribe, and `isUnsubscribed` checks if a user has already unsubscribed from a particular email type.
-   `getFromEmailAddress` from `'@/lib/email/utils'`: A custom utility function to retrieve the default "from" email address, likely from a configuration or environment variable.
-   `env` from `'@/lib/env'`: This imports an object that provides access to environment variables. This allows the application to be configured differently in different environments (e.g., development, production).
-   `createLogger` from `'@/lib/logs/console/logger'`: This imports a function that creates a logger instance. This allows the application to log messages to the console or other logging destinations.
-   `getBaseUrl` from `'@/lib/urls/utils'`: A custom utility to determine the base URL of the application, likely used for constructing unsubscribe links.

### 2. Logger:

```typescript
const logger = createLogger('Mailer')
```

-   Creates a logger instance named 'Mailer'. This allows logging messages related to email sending, making debugging and monitoring easier.

### 3. Type Definitions:

```typescript
export type EmailType = 'transactional' | 'marketing' | 'updates' | 'notifications'

export interface EmailAttachment {
  filename: string
  content: string | Buffer
  contentType: string
  disposition?: 'attachment' | 'inline'
}

export interface EmailOptions {
  to: string | string[]
  subject: string
  html?: string
  text?: string
  from?: string
  emailType?: EmailType
  includeUnsubscribe?: boolean
  attachments?: EmailAttachment[]
  replyTo?: string
}

export interface BatchEmailOptions {
  emails: EmailOptions[]
}

export interface SendEmailResult {
  success: boolean
  message: string
  data?: any
}

export interface BatchSendEmailResult {
  success: boolean
  message: string
  results: SendEmailResult[]
  data?: any
}

interface ProcessedEmailData {
  to: string | string[]
  subject: string
  html?: string
  text?: string
  senderEmail: string
  headers: Record<string, string>
  attachments?: EmailAttachment[]
  replyTo?: string
}
```

-   `EmailType`: Defines a type for email categories (transactional, marketing, updates, notifications).
-   `EmailAttachment`: Defines the structure for email attachments, including filename, content (as a string or Buffer), content type, and disposition (attachment or inline).
-   `EmailOptions`: Defines the structure for email options, including recipient(s), subject, HTML/text content, sender, email type, unsubscribe settings, attachments, and reply-to address.
-   `BatchEmailOptions`: Defines the structure for sending multiple emails at once. It contains an array of `EmailOptions`.
-   `SendEmailResult`: Defines the structure for the result of sending a single email, indicating success, a message, and optional data.
-   `BatchSendEmailResult`: Defines the structure for the result of sending a batch of emails, including overall success, a message, an array of `SendEmailResult` for each email, and optional data.
-   `ProcessedEmailData`: Defines the structure for email data after it has been processed (e.g., unsubscribe tokens added).  This contains all necessary data for sending an email via either Resend or Azure.

### 4. Configuration:

```typescript
const resendApiKey = env.RESEND_API_KEY
const azureConnectionString = env.AZURE_ACS_CONNECTION_STRING

const resend =
  resendApiKey && resendApiKey !== 'placeholder' && resendApiKey.trim() !== ''
    ? new Resend(resendApiKey)
    : null

const azureEmailClient =
  azureConnectionString && azureConnectionString.trim() !== ''
    ? new EmailClient(azureConnectionString)
    : null
```

-   Retrieves the Resend API key and Azure Communication Services connection string from environment variables.
-   Initializes the `Resend` and `EmailClient` instances based on the availability of the respective API keys/connection strings. If the keys are missing, invalid or set to 'placeholder', the corresponding client is set to `null`. This pattern ensures the application doesn't crash if an email service isn't configured.

### 5. `hasEmailService` Function:

```typescript
/**
 * Check if any email service is configured and available
 */
export function hasEmailService(): boolean {
  return !!(resend || azureEmailClient)
}
```

-   Checks if either Resend or Azure Communication Services is configured and available. It returns `true` if at least one of the services is configured, `false` otherwise.  This function provides a simple way to determine if the application is capable of sending emails.

### 6. `sendEmail` Function:

```typescript
export async function sendEmail(options: EmailOptions): Promise<SendEmailResult> {
  try {
    // Check if user has unsubscribed (skip for critical transactional emails)
    if (options.emailType !== 'transactional') {
      const unsubscribeType = options.emailType as 'marketing' | 'updates' | 'notifications'
      // For arrays, check the first email address (batch emails typically go to similar recipients)
      const primaryEmail = Array.isArray(options.to) ? options.to[0] : options.to
      const hasUnsubscribed = await isUnsubscribed(primaryEmail, unsubscribeType)
      if (hasUnsubscribed) {
        logger.info('Email not sent (user unsubscribed):', {
          to: options.to,
          subject: options.subject,
          emailType: options.emailType,
        })
        return {
          success: true,
          message: 'Email skipped (user unsubscribed)',
          data: { id: 'skipped-unsubscribed' },
        }
      }
    }

    // Process email data with unsubscribe tokens and headers
    const processedData = await processEmailData(options)

    // Try Resend first if configured
    if (resend) {
      try {
        return await sendWithResend(processedData)
      } catch (error) {
        logger.warn('Resend failed, attempting Azure Communication Services fallback:', error)
      }
    }

    // Fallback to Azure Communication Services if configured
    if (azureEmailClient) {
      try {
        return await sendWithAzure(processedData)
      } catch (error) {
        logger.error('Azure Communication Services also failed:', error)
        return {
          success: false,
          message: 'Both Resend and Azure Communication Services failed',
        }
      }
    }

    // No email service configured
    logger.info('Email not sent (no email service configured):', {
      to: options.to,
      subject: options.subject,
      from: processedData.senderEmail,
    })
    return {
      success: true,
      message: 'Email logging successful (no email service configured)',
      data: { id: 'mock-email-id' },
    }
  } catch (error) {
    logger.error('Error sending email:', error)
    return {
      success: false,
      message: 'Failed to send email',
    }
  }
}
```

-   The main function for sending emails.  It takes an `EmailOptions` object as input and returns a `Promise<SendEmailResult>`.
-   It first checks if the user has unsubscribed (unless the email is transactional). If the user has unsubscribed, it logs the event and returns a successful result with a message indicating that the email was skipped.
-   It then processes the email data using `processEmailData` to add unsubscribe tokens and headers.
-   It attempts to send the email using Resend first. If Resend is configured and the sending is successful, it returns the result from `sendWithResend`. If Resend fails, it logs a warning and falls back to Azure Communication Services.
-   If Resend is not configured or fails, it attempts to send the email using Azure Communication Services. If Azure Communication Services is configured and the sending is successful, it returns the result from `sendWithAzure`. If Azure Communication Services also fails, it logs an error and returns a failed result.
-   If neither Resend nor Azure Communication Services is configured, it logs the email details and returns a successful result with a message indicating that the email was not sent because no email service is configured.
-   The `try...catch` block handles any errors that occur during the email sending process. If an error occurs, it logs the error and returns a failed result.

### 7. `processEmailData` Function:

```typescript
async function processEmailData(options: EmailOptions): Promise<ProcessedEmailData> {
  const {
    to,
    subject,
    html,
    text,
    from,
    emailType = 'transactional',
    includeUnsubscribe = true,
    attachments,
    replyTo,
  } = options

  const senderEmail = from || getFromEmailAddress()

  // Generate unsubscribe token and add to content
  let finalHtml = html
  let finalText = text
  const headers: Record<string, string> = {}

  if (includeUnsubscribe && emailType !== 'transactional') {
    // For arrays, use the first email for unsubscribe (batch emails typically go to similar recipients)
    const primaryEmail = Array.isArray(to) ? to[0] : to
    const unsubscribeToken = generateUnsubscribeToken(primaryEmail, emailType)
    const baseUrl = getBaseUrl()
    const unsubscribeUrl = `${baseUrl}/unsubscribe?token=${unsubscribeToken}&email=${encodeURIComponent(primaryEmail)}`

    headers['List-Unsubscribe'] = `<${unsubscribeUrl}>`
    headers['List-Unsubscribe-Post'] = 'List-Unsubscribe=One-Click'

    if (html) {
      finalHtml = html.replace(/\{\{UNSUBSCRIBE_TOKEN\}\}/g, unsubscribeToken)
    }
    if (text) {
      finalText = text.replace(/\{\{UNSUBSCRIBE_TOKEN\}\}/g, unsubscribeToken)
    }
  }

  return {
    to,
    subject,
    html: finalHtml,
    text: finalText,
    senderEmail,
    headers,
    attachments,
    replyTo,
  }
}
```

-   Processes the email data to add unsubscribe tokens and headers. It takes `EmailOptions` as input and returns a `Promise<ProcessedEmailData>`.
-   It extracts the necessary properties from the `EmailOptions` object.
-   It determines the sender email address, using the `from` property if provided, or falling back to the `getFromEmailAddress` function.
-   If `includeUnsubscribe` is true and the email type is not `transactional`, it generates an unsubscribe token using the `generateUnsubscribeToken` function. It also constructs the unsubscribe URL using the `getBaseUrl` function.  Then, it adds the `List-Unsubscribe` and `List-Unsubscribe-Post` headers which are important for email deliverability and allow recipients to easily unsubscribe.
-   It replaces any occurrences of `{{UNSUBSCRIBE_TOKEN}}` in the HTML and text content with the generated unsubscribe token.
-   It returns a `ProcessedEmailData` object containing the processed email data.

### 8. `sendWithResend` Function:

```typescript
async function sendWithResend(data: ProcessedEmailData): Promise<SendEmailResult> {
  if (!resend) throw new Error('Resend not configured')

  const fromAddress = data.senderEmail

  const emailData: any = {
    from: fromAddress,
    to: data.to,
    subject: data.subject,
    headers: Object.keys(data.headers).length > 0 ? data.headers : undefined,
  }

  if (data.html) emailData.html = data.html
  if (data.text) emailData.text = data.text
  if (data.replyTo) emailData.replyTo = data.replyTo
  if (data.attachments) {
    emailData.attachments = data.attachments.map((att) => ({
      filename: att.filename,
      content: typeof att.content === 'string' ? att.content : att.content.toString('base64'),
      contentType: att.contentType,
      disposition: att.disposition || 'attachment',
    }))
  }

  const { data: responseData, error } = await resend.emails.send(emailData)

  if (error) {
    throw new Error(error.message || 'Failed to send email via Resend')
  }

  return {
    success: true,
    message: 'Email sent successfully via Resend',
    data: responseData,
  }
}
```

-   Sends an email using the Resend service. It takes `ProcessedEmailData` as input and returns a `Promise<SendEmailResult>`.
-   It first checks if the Resend client is configured. If not, it throws an error.
-   It constructs the `emailData` object with the necessary properties for sending an email via Resend, including the sender, recipient, subject, HTML/text content, attachments, and headers.
-   Attachment content is converted to base64 if it's not already a string.
-   It calls the `resend.emails.send` method to send the email.
-   If the sending is successful, it returns a successful result with the response data from Resend. If the sending fails, it throws an error.

### 9. `sendWithAzure` Function:

```typescript
async function sendWithAzure(data: ProcessedEmailData): Promise<SendEmailResult> {
  if (!azureEmailClient) throw new Error('Azure Communication Services not configured')

  // Azure Communication Services requires at least one content type
  if (!data.html && !data.text) {
    throw new Error('Azure Communication Services requires either HTML or text content')
  }

  // For Azure, use just the email address part (no display name)
  // Azure will use the display name configured in the portal for the sender address
  const senderEmailOnly = data.senderEmail.includes('<')
    ? data.senderEmail.match(/<(.+)>/)?.[1] || data.senderEmail
    : data.senderEmail

  const message: EmailMessage = {
    senderAddress: senderEmailOnly,
    content: data.html
      ? {
          subject: data.subject,
          html: data.html,
        }
      : {
          subject: data.subject,
          plainText: data.text!,
        },
    recipients: {
      to: Array.isArray(data.to)
        ? data.to.map((email) => ({ address: email }))
        : [{ address: data.to }],
    },
    headers: data.headers,
  }

  const poller = await azureEmailClient.beginSend(message)
  const result = await poller.pollUntilDone()

  if (result.status === 'Succeeded') {
    return {
      success: true,
      message: 'Email sent successfully via Azure Communication Services',
      data: { id: result.id },
    }
  }
  throw new Error(`Azure Communication Services failed with status: ${result.status}`)
}
```

-   Sends an email using Azure Communication Services. It takes `ProcessedEmailData` as input and returns a `Promise<SendEmailResult>`.
-   It first checks if the Azure Email Client is configured. If not, it throws an error.
-   It enforces that either HTML or text content must be provided, because Azure Communication Services requires at least one.
-   Extracts only the email address from the sender email, discarding any display name.  This is because Azure handles the display name configuration separately.
-   It constructs the `EmailMessage` object with the necessary properties for sending an email via Azure Communication Services, including the sender address, subject, HTML/text content, recipients, and headers.
-   It calls the `azureEmailClient.beginSend` method to start the email sending process, retrieves the poller, and waits until complete.
-   If the sending is successful (status is 'Succeeded'), it returns a successful result with the message ID from Azure Communication Services. If the sending fails, it throws an error.

### 10. `sendBatchEmails` Function:

```typescript
export async function sendBatchEmails(options: BatchEmailOptions): Promise<BatchSendEmailResult> {
  try {
    const results: SendEmailResult[] = []

    // Try Resend first for batch emails if available
    if (resend) {
      try {
        return await sendBatchWithResend(options.emails)
      } catch (error) {
        logger.warn('Resend batch failed, falling back to individual sends:', error)
      }
    }

    // Fallback to individual sends (works with both Azure and Resend)
    logger.info('Sending batch emails individually')
    for (const email of options.emails) {
      try {
        const result = await sendEmail(email)
        results.push(result)
      } catch (error) {
        results.push({
          success: false,
          message: error instanceof Error ? error.message : 'Failed to send email',
        })
      }
    }

    const successCount = results.filter((r) => r.success).length
    return {
      success: successCount === results.length,
      message:
        successCount === results.length
          ? 'All batch emails sent successfully'
          : `${successCount}/${results.length} emails sent successfully`,
      results,
      data: { count: successCount },
    }
  } catch (error) {
    logger.error('Error in batch email sending:', error)
    return {
      success: false,
      message: 'Failed to send batch emails',
      results: [],
    }
  }
}
```

-   Sends a batch of emails. It takes a `BatchEmailOptions` object as input and returns a `Promise<BatchSendEmailResult>`.
-   It first tries to send the batch using `sendBatchWithResend` if Resend is configured. If this fails it logs a warning and falls back to sending emails individually.
-   If `sendBatchWithResend` fails or Resend is not configured, it iterates over the emails in the `options.emails` array and sends each email individually using the `sendEmail` function.
-   It collects the results of each email sending in the `results` array.
-   Finally, it aggregates the results and returns a `BatchSendEmailResult` object indicating the overall success, a message, the individual results, and the number of successfully sent emails.

### 11. `sendBatchWithResend` Function:

```typescript
async function sendBatchWithResend(emails: EmailOptions[]): Promise<BatchSendEmailResult> {
  if (!resend) throw new Error('Resend not configured')

  const results: SendEmailResult[] = []
  const batchEmails = emails.map((email) => {
    const senderEmail = email.from || getFromEmailAddress()
    const emailData: any = {
      from: senderEmail,
      to: email.to,
      subject: email.subject,
    }
    if (email.html) emailData.html = email.html
    if (email.text) emailData.text = email.text
    return emailData
  })

  try {
    const response = await resend.batch.send(batchEmails as any)

    if (response.error) {
      throw new Error(response.error.message || 'Resend batch API error')
    }

    // Success - create results for each email
    batchEmails.forEach((_, index) => {
      results.push({
        success: true,
        message: 'Email sent successfully via Resend batch',
        data: { id: `batch-${index}` },
      })
    })

    return {
      success: true,
      message: 'All batch emails sent successfully via Resend',
      results,
      data: { count: results.length },
    }
  } catch (error) {
    logger.error('Resend batch send failed:', error)
    throw error // Let the caller handle fallback
  }
}
```

-   Sends a batch of emails using Resend's batch API.  It takes an array of `EmailOptions` as input and returns a `Promise<BatchSendEmailResult>`.
-   It first checks if the Resend client is configured. If not, it throws an error.
-   It transforms the array of `EmailOptions` into an array of objects suitable for Resend's batch API, extracting the necessary properties (from, to, subject, html, text).
-   It calls the `resend.batch.send` method to send the batch of emails.
-   If the sending is successful, it creates a `SendEmailResult` object for each email in the batch, indicating success. It returns a `BatchSendEmailResult` object indicating the overall success, a message, and the individual results.
-   If the sending fails, it logs the error and re-throws it, allowing the caller function (`sendBatchEmails`) to handle the error and fall back to individual email sending.

### Summary:

This file provides a robust and flexible email sending module that supports both Resend and Azure Communication Services. It includes features such as unsubscribe handling, attachment support, and batch email sending. The module is designed to be fault-tolerant, with a fallback mechanism in case one of the email services fails. It also includes logging to help with debugging and monitoring.  The use of interfaces and type definitions makes the code more maintainable and easier to understand. The module