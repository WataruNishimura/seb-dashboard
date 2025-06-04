# Integration Features Design Specification

## 1. Overview

### 1.1 Purpose
This specification defines the integration capabilities of the project management tool with external services including Slack, Email, and Google Forms to enhance collaboration and automate workflows.

### 1.2 Integration Scope
- **Slack**: Notifications, commands, and interactive workflows
- **Email**: Automated reports, notifications, and task creation
- **Google Forms**: Event registration and data synchronization
- **Future**: Calendar systems, GitHub, JIRA

## 2. Slack Integration

### 2.1 Architecture
```typescript
// Slack App Configuration
interface SlackConfig {
  appId: string
  clientId: string
  clientSecret: string
  signingSecret: string
  botToken: string
  appToken: string // For Socket Mode
  webhookUrl?: string
}

// Slack Integration Model
model SlackIntegration {
  id              String   @id @default(cuid())
  workspaceId     String   @unique
  teamName        String
  botUserId       String
  botAccessToken  String   @db.Text // Encrypted
  installedBy     String   // Auth0 user ID
  installedAt     DateTime @default(now())
  isActive        Boolean  @default(true)
  
  channels        SlackChannel[]
  notifications   SlackNotificationConfig[]
}

model SlackChannel {
  id              String   @id @default(cuid())
  integrationId   String
  channelId       String
  channelName     String
  channelType     ChannelType
  isDefault       Boolean  @default(false)
  
  integration     SlackIntegration @relation(fields: [integrationId], references: [id])
  
  @@unique([integrationId, channelId])
}

enum ChannelType {
  PUBLIC
  PRIVATE
  DIRECT
}
```

### 2.2 Slack Commands
```typescript
// Command Registry
const slackCommands = {
  '/task': {
    description: 'Task management commands',
    subcommands: {
      'create': 'Create a new task',
      'list': 'List your tasks',
      'complete': 'Mark a task as complete',
      'assign': 'Assign a task to someone'
    }
  },
  '/event': {
    description: 'Event management commands',
    subcommands: {
      'list': 'List upcoming events',
      'register': 'Register for an event',
      'info': 'Get event details'
    }
  },
  '/report': {
    description: 'Report commands',
    subcommands: {
      'latest': 'Get the latest weekly report',
      'generate': 'Generate a custom report'
    }
  }
}

// Command Handler
class SlackCommandHandler {
  async handleCommand(payload: SlackCommandPayload) {
    const { command, text, user_id, channel_id } = payload
    
    switch (command) {
      case '/task':
        return this.handleTaskCommand(text, user_id, channel_id)
      case '/event':
        return this.handleEventCommand(text, user_id, channel_id)
      case '/report':
        return this.handleReportCommand(text, user_id, channel_id)
      default:
        return this.unknownCommand(command)
    }
  }

  async handleTaskCommand(text: string, userId: string, channelId: string) {
    const [subcommand, ...args] = text.split(' ')
    
    switch (subcommand) {
      case 'create':
        return this.showTaskCreationModal(userId)
      
      case 'list':
        const tasks = await this.getUserTasks(userId)
        return this.formatTaskList(tasks)
      
      case 'complete':
        const taskId = args[0]
        await this.completeTask(taskId, userId)
        return { text: `Task ${taskId} marked as complete ✓` }
      
      case 'assign':
        const [taskId, assignee] = args
        return this.assignTask(taskId, assignee, userId)
      
      default:
        return this.showTaskHelp()
    }
  }
}
```

### 2.3 Interactive Components
```typescript
// Modal Definitions
const taskCreationModal = {
  type: 'modal',
  callback_id: 'task_create_modal',
  title: { type: 'plain_text', text: 'Create New Task' },
  blocks: [
    {
      type: 'input',
      block_id: 'title',
      label: { type: 'plain_text', text: 'Task Title' },
      element: {
        type: 'plain_text_input',
        action_id: 'title_input',
        placeholder: { type: 'plain_text', text: 'Enter task title' }
      }
    },
    {
      type: 'input',
      block_id: 'description',
      label: { type: 'plain_text', text: 'Description' },
      element: {
        type: 'plain_text_input',
        action_id: 'description_input',
        multiline: true,
        optional: true
      }
    },
    {
      type: 'input',
      block_id: 'priority',
      label: { type: 'plain_text', text: 'Priority' },
      element: {
        type: 'static_select',
        action_id: 'priority_select',
        options: [
          { text: { type: 'plain_text', text: 'Low' }, value: 'LOW' },
          { text: { type: 'plain_text', text: 'Medium' }, value: 'MEDIUM' },
          { text: { type: 'plain_text', text: 'High' }, value: 'HIGH' },
          { text: { type: 'plain_text', text: 'Urgent' }, value: 'URGENT' }
        ],
        initial_option: { text: { type: 'plain_text', text: 'Medium' }, value: 'MEDIUM' }
      }
    },
    {
      type: 'input',
      block_id: 'due_date',
      label: { type: 'plain_text', text: 'Due Date' },
      element: {
        type: 'datepicker',
        action_id: 'due_date_picker',
        optional: true
      }
    }
  ],
  submit: { type: 'plain_text', text: 'Create Task' }
}

// Interactive Message Handler
class SlackInteractionHandler {
  async handleInteraction(payload: SlackInteractionPayload) {
    switch (payload.type) {
      case 'view_submission':
        return this.handleModalSubmission(payload)
      
      case 'block_actions':
        return this.handleBlockAction(payload)
      
      case 'shortcut':
        return this.handleShortcut(payload)
    }
  }

  async handleModalSubmission(payload: ViewSubmissionPayload) {
    const { view, user } = payload
    
    if (view.callback_id === 'task_create_modal') {
      const values = view.state.values
      
      const taskData = {
        title: values.title.title_input.value,
        description: values.description?.description_input.value,
        priority: values.priority.priority_select.selected_option.value,
        dueDate: values.due_date?.due_date_picker.selected_date,
        createdBy: await this.getSystemUserId(user.id)
      }
      
      const task = await this.createTask(taskData)
      
      // Send confirmation message
      await this.sendMessage(user.id, {
        text: `Task "${task.title}" created successfully!`,
        blocks: this.formatTaskBlocks(task)
      })
    }
  }
}
```

### 2.4 Notification System
```typescript
model SlackNotificationConfig {
  id              String   @id @default(cuid())
  integrationId   String
  eventType       NotificationEventType
  channelId       String
  enabled         Boolean  @default(true)
  template        String?  @db.Text
  
  integration     SlackIntegration @relation(fields: [integrationId], references: [id])
}

enum NotificationEventType {
  TASK_ASSIGNED
  TASK_COMPLETED
  TASK_OVERDUE
  EVENT_CREATED
  EVENT_REMINDER
  EVENT_REGISTRATION
  REPORT_GENERATED
  WEEKLY_SUMMARY
}

class SlackNotificationService {
  async sendNotification(type: NotificationEventType, data: any) {
    const configs = await this.getNotificationConfigs(type)
    
    for (const config of configs) {
      if (!config.enabled) continue
      
      const message = await this.formatMessage(type, data, config)
      await this.sendToChannel(config.channelId, message)
    }
  }

  private formatMessage(type: NotificationEventType, data: any, config: SlackNotificationConfig) {
    switch (type) {
      case 'TASK_ASSIGNED':
        return {
          text: `New task assigned to <@${data.assigneeSlackId}>`,
          blocks: [
            {
              type: 'section',
              text: {
                type: 'mrkdwn',
                text: `*New Task Assigned*\n*Title:* ${data.task.title}\n*Priority:* ${data.task.priority}\n*Due:* ${data.task.dueDate || 'No due date'}`
              }
            },
            {
              type: 'actions',
              elements: [
                {
                  type: 'button',
                  text: { type: 'plain_text', text: 'View Task' },
                  url: `${process.env.APP_URL}/tasks/${data.task.id}`
                },
                {
                  type: 'button',
                  text: { type: 'plain_text', text: 'Start Working' },
                  action_id: 'start_task',
                  value: data.task.id
                }
              ]
            }
          ]
        }
      
      case 'EVENT_REMINDER':
        return {
          text: `Event reminder: ${data.event.title}`,
          blocks: [
            {
              type: 'section',
              text: {
                type: 'mrkdwn',
                text: `*Upcoming Event Reminder*\n*Event:* ${data.event.title}\n*Date:* ${data.event.eventDate}\n*Location:* ${data.event.location || 'Virtual'}\n*Registered:* ${data.registeredCount} attendees`
              }
            }
          ]
        }
      
      // ... other notification types
    }
  }
}
```

## 3. Email Integration

### 3.1 Email Service Configuration
```typescript
// Email Provider Configuration
interface EmailConfig {
  provider: 'sendgrid' | 'ses' | 'smtp'
  apiKey?: string
  smtpConfig?: {
    host: string
    port: number
    secure: boolean
    auth: {
      user: string
      pass: string
    }
  }
  fromAddress: string
  fromName: string
  replyTo?: string
}

// Email Template Model
model EmailTemplate {
  id              String   @id @default(cuid())
  name            String   @unique
  subject         String
  subjectJa       String
  htmlTemplate    String   @db.Text
  textTemplate    String   @db.Text
  category        EmailCategory
  variables       Json     // Expected template variables
  isActive        Boolean  @default(true)
  createdAt       DateTime @default(now())
  updatedAt       DateTime @updatedAt
}

enum EmailCategory {
  NOTIFICATION
  REPORT
  EVENT
  TASK
  SYSTEM
}
```

### 3.2 Email Notification Service
```typescript
class EmailNotificationService {
  async sendEmail(options: EmailOptions) {
    const template = await this.getTemplate(options.templateId)
    const recipient = await this.getRecipient(options.to)
    
    const emailData = {
      to: options.to,
      subject: this.renderSubject(template, options.data, recipient.preferredLanguage),
      html: this.renderHtml(template, options.data, recipient.preferredLanguage),
      text: this.renderText(template, options.data, recipient.preferredLanguage),
      attachments: options.attachments
    }
    
    await this.send(emailData)
    await this.logEmailSent(emailData)
  }

  async sendBulkEmail(templateId: string, recipients: EmailRecipient[], data: any) {
    const template = await this.getTemplate(templateId)
    
    // Batch recipients by language preference
    const batches = this.groupByLanguage(recipients)
    
    for (const [language, batch] of batches) {
      const personalizations = batch.map(recipient => ({
        to: recipient.email,
        substitutions: this.getPersonalizations(recipient, data)
      }))
      
      await this.sendBatch({
        personalizations,
        subject: language === 'ja' ? template.subjectJa : template.subject,
        html: this.renderHtml(template, data, language),
        text: this.renderText(template, data, language)
      })
    }
  }
}
```

### 3.3 Email Templates
```typescript
// Task Assignment Email Template
const taskAssignmentTemplate = {
  name: 'task_assignment',
  subject: 'New Task Assigned: {{task.title}}',
  subjectJa: '新しいタスクが割り当てられました: {{task.titleJa}}',
  htmlTemplate: `
    <!DOCTYPE html>
    <html>
    <head>
      <style>
        .container { max-width: 600px; margin: 0 auto; font-family: Arial, sans-serif; }
        .header { background: #4A5568; color: white; padding: 20px; }
        .content { padding: 20px; }
        .button { display: inline-block; padding: 10px 20px; background: #3182CE; color: white; text-decoration: none; border-radius: 5px; }
      </style>
    </head>
    <body>
      <div class="container">
        <div class="header">
          <h1>{{#if isJapanese}}新しいタスク{{else}}New Task Assignment{{/if}}</h1>
        </div>
        <div class="content">
          <p>{{#if isJapanese}}{{user.name}}様{{else}}Hi {{user.name}}{{/if}},</p>
          
          <p>{{#if isJapanese}}
            新しいタスクが割り当てられました：
          {{else}}
            You have been assigned a new task:
          {{/if}}</p>
          
          <h2>{{#if isJapanese}}{{task.titleJa}}{{else}}{{task.title}}{{/if}}</h2>
          
          <p>{{#if isJapanese}}{{task.descriptionJa}}{{else}}{{task.description}}{{/if}}</p>
          
          <ul>
            <li>{{i18n "Priority"}}: {{task.priority}}</li>
            <li>{{i18n "Due Date"}}: {{formatDate task.dueDate}}</li>
            <li>{{i18n "Category"}}: {{task.category}}</li>
          </ul>
          
          <a href="{{appUrl}}/tasks/{{task.id}}" class="button">
            {{#if isJapanese}}タスクを表示{{else}}View Task{{/if}}
          </a>
        </div>
      </div>
    </body>
    </html>
  `,
  textTemplate: `
    {{#if isJapanese}}{{user.name}}様{{else}}Hi {{user.name}}{{/if}},
    
    {{#if isJapanese}}新しいタスクが割り当てられました{{else}}You have been assigned a new task{{/if}}:
    
    {{#if isJapanese}}{{task.titleJa}}{{else}}{{task.title}}{{/if}}
    
    {{#if isJapanese}}{{task.descriptionJa}}{{else}}{{task.description}}{{/if}}
    
    {{i18n "Priority"}}: {{task.priority}}
    {{i18n "Due Date"}}: {{formatDate task.dueDate}}
    
    {{i18n "View task at"}}: {{appUrl}}/tasks/{{task.id}}
  `
}
```

### 3.4 Email-to-Task Feature
```typescript
// Inbound Email Processing
interface InboundEmailProcessor {
  async processEmail(email: InboundEmail) {
    // 1. Verify sender is authorized
    const user = await this.verifyUser(email.from)
    if (!user) {
      await this.sendRejectionEmail(email.from)
      return
    }
    
    // 2. Parse email content
    const taskData = this.parseEmailToTask(email)
    
    // 3. Extract metadata from subject
    const metadata = this.extractMetadata(email.subject)
    
    // 4. Create task
    const task = await this.createTaskFromEmail({
      ...taskData,
      ...metadata,
      createdBy: user.id,
      attachments: await this.processAttachments(email.attachments)
    })
    
    // 5. Send confirmation
    await this.sendConfirmationEmail(email.from, task)
  }

  parseEmailToTask(email: InboundEmail): Partial<Task> {
    // Subject format: [PRIORITY] Task Title #category @assignee !due:2024-03-15
    const subjectRegex = /(?:\[(\w+)\])?\s*(.+?)(?:\s+#(\w+))?(?:\s+@(\w+))?(?:\s+!due:(\d{4}-\d{2}-\d{2}))?$/
    const match = email.subject.match(subjectRegex)
    
    if (match) {
      const [_, priority, title, category, assignee, dueDate] = match
      
      return {
        title: title.trim(),
        description: email.text || email.html,
        priority: priority as TaskPriority || 'MEDIUM',
        category: this.mapCategory(category),
        dueDate: dueDate ? new Date(dueDate) : undefined
      }
    }
    
    return {
      title: email.subject,
      description: email.text || email.html
    }
  }
}
```

## 4. Google Forms Integration

### 4.1 Google Forms Configuration
```typescript
model GoogleFormIntegration {
  id              String   @id @default(cuid())
  formId          String   @unique
  formTitle       String
  eventId         String?
  mappingConfig   Json     // Field mappings
  syncEnabled     Boolean  @default(true)
  lastSyncAt      DateTime?
  webhookUrl      String?  // For real-time updates
  createdBy       String
  createdAt       DateTime @default(now())
  
  event           Event?   @relation(fields: [eventId], references: [id])
  responses       FormResponse[]
}

interface FieldMapping {
  formFieldId: string
  formFieldTitle: string
  targetField: 'name' | 'email' | 'company' | 'dietary' | 'interests' | 'custom'
  customFieldName?: string
  transform?: 'lowercase' | 'uppercase' | 'trim' | 'split'
}

model FormResponse {
  id              String   @id @default(cuid())
  integrationId   String
  responseId      String   @unique
  submittedAt     DateTime
  responseData    Json
  processedAt     DateTime?
  guestId         String?
  
  integration     GoogleFormIntegration @relation(fields: [integrationId], references: [id])
  guest           Guest?   @relation(fields: [guestId], references: [id])
}
```

### 4.2 Google Forms Service
```typescript
class GoogleFormsService {
  private auth: OAuth2Client

  async connectForm(formId: string, eventId: string, userId: string) {
    // 1. Authenticate with Google
    await this.authenticate(userId)
    
    // 2. Get form structure
    const form = await this.getFormStructure(formId)
    
    // 3. Auto-detect field mappings
    const mappings = this.autoDetectMappings(form.items)
    
    // 4. Create integration record
    const integration = await prisma.googleFormIntegration.create({
      data: {
        formId,
        formTitle: form.title,
        eventId,
        mappingConfig: mappings,
        createdBy: userId
      }
    })
    
    // 5. Set up webhook for real-time updates
    await this.setupWebhook(formId, integration.id)
    
    // 6. Import existing responses
    await this.importExistingResponses(formId, integration.id)
    
    return integration
  }

  private autoDetectMappings(items: GoogleFormItem[]): FieldMapping[] {
    const mappings: FieldMapping[] = []
    
    for (const item of items) {
      const title = item.title.toLowerCase()
      
      // Auto-detect common fields
      if (title.includes('name') || title.includes('名前')) {
        mappings.push({
          formFieldId: item.id,
          formFieldTitle: item.title,
          targetField: 'name'
        })
      } else if (title.includes('email') || title.includes('メール')) {
        mappings.push({
          formFieldId: item.id,
          formFieldTitle: item.title,
          targetField: 'email',
          transform: 'lowercase'
        })
      } else if (title.includes('company') || title.includes('会社')) {
        mappings.push({
          formFieldId: item.id,
          formFieldTitle: item.title,
          targetField: 'company'
        })
      } else if (title.includes('dietary') || title.includes('食事') || title.includes('アレルギー')) {
        mappings.push({
          formFieldId: item.id,
          formFieldTitle: item.title,
          targetField: 'dietary'
        })
      } else if (title.includes('interest') || title.includes('興味')) {
        mappings.push({
          formFieldId: item.id,
          formFieldTitle: item.title,
          targetField: 'interests',
          transform: 'split'
        })
      }
    }
    
    return mappings
  }

  async processFormResponse(webhookData: GoogleFormWebhook) {
    const integration = await this.getIntegration(webhookData.formId)
    if (!integration || !integration.syncEnabled) return
    
    // 1. Parse response data
    const responseData = this.parseWebhookData(webhookData)
    
    // 2. Map fields to guest data
    const guestData = this.mapResponseToGuest(responseData, integration.mappingConfig)
    
    // 3. Create or update guest
    const guest = await this.upsertGuest(guestData)
    
    // 4. Register for event if linked
    if (integration.eventId) {
      await this.registerGuestForEvent(guest.id, integration.eventId)
    }
    
    // 5. Store raw response
    await prisma.formResponse.create({
      data: {
        integrationId: integration.id,
        responseId: webhookData.responseId,
        submittedAt: new Date(webhookData.timestamp),
        responseData,
        processedAt: new Date(),
        guestId: guest.id
      }
    })
  }

  private mapResponseToGuest(response: any, mappings: FieldMapping[]): Partial<Guest> {
    const guest: Partial<Guest> = {}
    
    for (const mapping of mappings) {
      const value = response[mapping.formFieldId]
      if (!value) continue
      
      // Apply transformations
      let processedValue = value
      if (mapping.transform) {
        switch (mapping.transform) {
          case 'lowercase':
            processedValue = value.toLowerCase()
            break
          case 'uppercase':
            processedValue = value.toUpperCase()
            break
          case 'trim':
            processedValue = value.trim()
            break
          case 'split':
            processedValue = value.split(',').map(s => s.trim())
            break
        }
      }
      
      // Map to guest field
      if (mapping.targetField === 'custom' && mapping.customFieldName) {
        guest[mapping.customFieldName] = processedValue
      } else {
        guest[mapping.targetField] = processedValue
      }
    }
    
    return guest
  }
}
```

### 4.3 Sync Management
```typescript
class GoogleFormsSyncService {
  async syncAllForms() {
    const integrations = await prisma.googleFormIntegration.findMany({
      where: { syncEnabled: true }
    })
    
    for (const integration of integrations) {
      try {
        await this.syncForm(integration)
      } catch (error) {
        await this.handleSyncError(integration.id, error)
      }
    }
  }

  async syncForm(integration: GoogleFormIntegration) {
    // 1. Get latest responses since last sync
    const responses = await this.getNewResponses(
      integration.formId,
      integration.lastSyncAt
    )
    
    // 2. Process each response
    for (const response of responses) {
      await this.processFormResponse({
        formId: integration.formId,
        responseId: response.id,
        timestamp: response.timestamp,
        answers: response.answers
      })
    }
    
    // 3. Update last sync time
    await prisma.googleFormIntegration.update({
      where: { id: integration.id },
      data: { lastSyncAt: new Date() }
    })
  }

  async validateIntegration(integrationId: string): ValidationResult {
    const integration = await this.getIntegration(integrationId)
    
    const checks = {
      formAccessible: await this.checkFormAccess(integration.formId),
      mappingsValid: this.validateMappings(integration.mappingConfig),
      eventExists: integration.eventId ? await this.checkEventExists(integration.eventId) : true,
      webhookActive: await this.checkWebhookStatus(integration.webhookUrl)
    }
    
    return {
      isValid: Object.values(checks).every(v => v === true),
      checks
    }
  }
}
```

## 5. Integration Dashboard

### 5.1 Integration Management UI
```typescript
interface IntegrationDashboardProps {
  slack: SlackIntegration[]
  email: EmailConfig
  googleForms: GoogleFormIntegration[]
}

// Features:
// - Connection status for each service
// - Configuration management
// - Test integration functionality
// - Activity logs
// - Error monitoring
```

### 5.2 Slack Configuration Page
```typescript
interface SlackConfigPageProps {
  integration: SlackIntegration
  channels: SlackChannel[]
  notifications: SlackNotificationConfig[]
}

// Components:
// 1. OAuth connection flow
// 2. Channel selector
// 3. Notification preferences
// 4. Command configuration
// 5. Test message sender
```

### 5.3 Google Forms Manager
```typescript
interface GoogleFormsManagerProps {
  integrations: GoogleFormIntegration[]
  events: Event[]
}

// Features:
// - Add new form integration
// - Field mapping editor
// - Response viewer
// - Sync status and history
// - Manual sync trigger
```

## 6. Security and Authentication

### 6.1 OAuth2 Management
```typescript
class OAuthService {
  async initiateOAuth(provider: 'slack' | 'google', userId: string) {
    const state = await this.generateSecureState(userId)
    const authUrl = this.getAuthorizationUrl(provider, state)
    
    // Store state for verification
    await this.storeOAuthState(state, provider, userId)
    
    return authUrl
  }

  async handleCallback(provider: string, code: string, state: string) {
    // 1. Verify state
    const stateData = await this.verifyState(state)
    if (!stateData) throw new Error('Invalid state')
    
    // 2. Exchange code for tokens
    const tokens = await this.exchangeCodeForTokens(provider, code)
    
    // 3. Get user/workspace info
    const info = await this.getProviderInfo(provider, tokens.access_token)
    
    // 4. Store integration
    await this.storeIntegration(provider, stateData.userId, tokens, info)
    
    // 5. Clean up state
    await this.deleteOAuthState(state)
  }

  async refreshToken(integrationId: string) {
    const integration = await this.getIntegration(integrationId)
    const newTokens = await this.refreshProviderToken(
      integration.provider,
      integration.refreshToken
    )
    
    await this.updateIntegrationTokens(integrationId, newTokens)
  }
}
```

### 6.2 Webhook Security
```typescript
class WebhookSecurityService {
  async verifySlackRequest(request: Request): boolean {
    const signature = request.headers['x-slack-signature']
    const timestamp = request.headers['x-slack-request-timestamp']
    const body = await request.text()
    
    // Verify timestamp to prevent replay attacks
    if (Math.abs(Date.now() / 1000 - parseInt(timestamp)) > 300) {
      return false
    }
    
    // Verify signature
    const sigBasestring = `v0:${timestamp}:${body}`
    const mySignature = 'v0=' + crypto
      .createHmac('sha256', process.env.SLACK_SIGNING_SECRET)
      .update(sigBasestring)
      .digest('hex')
    
    return crypto.timingSafeEqual(
      Buffer.from(mySignature),
      Buffer.from(signature)
    )
  }

  async verifyGoogleWebhook(request: Request): boolean {
    // Implement Google's push notification verification
    const token = request.headers['authorization']?.replace('Bearer ', '')
    return token === process.env.GOOGLE_WEBHOOK_TOKEN
  }
}
```

## 7. Error Handling and Monitoring

### 7.1 Integration Health Monitoring
```typescript
class IntegrationHealthMonitor {
  async checkAllIntegrations() {
    const results = {
      slack: await this.checkSlackHealth(),
      email: await this.checkEmailHealth(),
      googleForms: await this.checkGoogleFormsHealth()
    }
    
    // Store health check results
    await this.storeHealthCheckResults(results)
    
    // Alert if any service is down
    if (Object.values(results).some(r => !r.healthy)) {
      await this.sendHealthAlert(results)
    }
  }

  async checkSlackHealth(): HealthCheckResult {
    try {
      const integrations = await prisma.slackIntegration.findMany({
        where: { isActive: true }
      })
      
      for (const integration of integrations) {
        // Test API connection
        const response = await slack.auth.test({
          token: integration.botAccessToken
        })
        
        if (!response.ok) {
          return {
            healthy: false,
            service: 'slack',
            error: response.error,
            integrationId: integration.id
          }
        }
      }
      
      return { healthy: true, service: 'slack' }
    } catch (error) {
      return {
        healthy: false,
        service: 'slack',
        error: error.message
      }
    }
  }
}
```

### 7.2 Retry and Fallback Mechanisms
```typescript
class IntegrationRetryService {
  async executeWithRetry<T>(
    operation: () => Promise<T>,
    options: RetryOptions
  ): Promise<T> {
    const { maxRetries = 3, backoffMs = 1000, exponential = true } = options
    
    for (let attempt = 1; attempt <= maxRetries; attempt++) {
      try {
        return await operation()
      } catch (error) {
        if (attempt === maxRetries) throw error
        
        const delay = exponential 
          ? backoffMs * Math.pow(2, attempt - 1)
          : backoffMs
          
        await this.delay(delay)
      }
    }
  }

  async sendNotificationWithFallback(
    notification: Notification,
    channels: NotificationChannel[]
  ) {
    for (const channel of channels) {
      try {
        await this.sendViaChannel(notification, channel)
        return // Success, no need for fallback
      } catch (error) {
        console.error(`Failed to send via ${channel}:`, error)
        continue // Try next channel
      }
    }
    
    // All channels failed, store for manual review
    await this.storeFailedNotification(notification)
  }
}
```

## 8. Performance Optimization

### 8.1 Batch Processing
```typescript
class BatchIntegrationProcessor {
  async processBatchNotifications(notifications: Notification[]) {
    // Group by channel and recipient
    const batches = this.groupNotifications(notifications)
    
    // Process each batch in parallel
    await Promise.all(
      batches.map(batch => this.processBatch(batch))
    )
  }

  private groupNotifications(notifications: Notification[]) {
    const groups = new Map<string, Notification[]>()
    
    for (const notification of notifications) {
      const key = `${notification.channel}-${notification.recipient}`
      if (!groups.has(key)) {
        groups.set(key, [])
      }
      groups.get(key).push(notification)
    }
    
    return Array.from(groups.values())
  }
}
```

### 8.2 Caching Strategy
```typescript
class IntegrationCacheService {
  private cache: Map<string, CacheEntry> = new Map()

  async getCachedOrFetch<T>(
    key: string,
    fetcher: () => Promise<T>,
    ttlMs: number = 300000 // 5 minutes
  ): Promise<T> {
    const cached = this.cache.get(key)
    
    if (cached && Date.now() - cached.timestamp < ttlMs) {
      return cached.value as T
    }
    
    const value = await fetcher()
    this.cache.set(key, {
      value,
      timestamp: Date.now()
    })
    
    return value
  }

  invalidate(pattern: string) {
    const keys = Array.from(this.cache.keys())
    const regex = new RegExp(pattern)
    
    for (const key of keys) {
      if (regex.test(key)) {
        this.cache.delete(key)
      }
    }
  }
}
```