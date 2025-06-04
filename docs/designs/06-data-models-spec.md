# Data Models Specification Document

## 1. Overview

### 1.1 Database Architecture
- **Database**: PostgreSQL 15+
- **ORM**: Prisma 5+
- **Schema Management**: Prisma Migrate
- **Naming Convention**: snake_case for database, camelCase for application
- **Time Zone**: All timestamps stored in UTC
- **Soft Deletes**: Implemented for critical entities

### 1.2 Design Principles
- Normalized structure (3NF) with selective denormalization for performance
- Audit trail for all critical operations
- Bilingual support for user-facing content
- Flexible JSON fields for extensibility
- Proper indexing for query performance

## 2. Core Data Models

### 2.1 Complete Prisma Schema

```prisma
// prisma/schema.prisma

generator client {
  provider = "prisma-client-js"
}

datasource db {
  provider = "postgresql"
  url      = env("DATABASE_URL")
}

// ==================== USER & AUTH ====================

model User {
  id                String   @id // Auth0 user ID
  email             String   @unique
  name              String
  nameJa            String?
  roles             Role[]
  locale            Language @default(JA)
  timezone          String   @default("Asia/Tokyo")
  isActive          Boolean  @default(true)
  lastLoginAt       DateTime?
  createdAt         DateTime @default(now())
  updatedAt         DateTime @updatedAt
  
  // Relations
  preferences       UserPreference?
  createdEvents     Event[]  @relation("EventCreator")
  createdTasks      Task[]   @relation("TaskCreator")
  assignedTasks     Task[]   @relation("TaskAssignee")
  timeEntries       TimeEntry[]
  taskComments      TaskComment[]
  auditLogs         AuditLog[]
  notifications     Notification[]
  reportsSent       Report[] @relation("ReportCreator")
  
  @@map("users")
}

model UserPreference {
  id                String   @id @default(cuid())
  userId            String   @unique
  emailNotifications Boolean @default(true)
  slackNotifications Boolean @default(true)
  weeklyReportSubscription Boolean @default(false)
  taskReminders     Boolean  @default(true)
  eventReminders    Boolean  @default(true)
  reminderTime      String   @default("09:00") // HH:mm format
  customSettings    Json?    // Flexible settings storage
  
  user              User     @relation(fields: [userId], references: [id])
  
  @@map("user_preferences")
}

enum Role {
  ADMIN
  EVENT_MANAGER
  TEAM_LEAD
  ENGINEER
  
  @@map("roles")
}

enum Language {
  JA
  EN
  
  @@map("languages")
}

// ==================== EVENT MANAGEMENT ====================

model Event {
  id                String   @id @default(cuid())
  title             String
  titleJa           String?
  slug              String   @unique // URL-friendly identifier
  category          EventCategory
  subcategory       EventSubcategory
  status            EventStatus @default(PLANNING)
  eventDate         DateTime
  eventEndDate      DateTime?
  location          String?
  locationJa        String?
  isVirtual         Boolean  @default(false)
  virtualUrl        String?
  description       String?  @db.Text
  descriptionJa     String?  @db.Text
  planDocumentUrl   String?
  reportBlogUrl     String?
  plannedAttendees  Int?
  actualAttendees   Int?
  budget            Decimal? @db.Decimal(12, 2)
  actualCost        Decimal? @db.Decimal(12, 2)
  sponsorshipLevel  String?  // For supporting events
  tags              String[] // Array of tags
  metadata          Json?    // Flexible data storage
  createdBy         String
  deletedAt         DateTime? // Soft delete
  createdAt         DateTime @default(now())
  updatedAt         DateTime @updatedAt
  
  // Relations
  creator           User     @relation("EventCreator", fields: [createdBy], references: [id])
  guests            EventRegistration[]
  tasks             Task[]
  clientReports     ClientReport[]
  eventHistory      EventHistory[]
  googleForm        GoogleFormIntegration?
  
  @@index([category, subcategory, status])
  @@index([eventDate])
  @@index([createdBy])
  @@map("events")
}

enum EventCategory {
  SELF_HOSTING
  SUPPORTING
  
  @@map("event_categories")
}

enum EventSubcategory {
  MEETUP
  CAREER
  GRAND
  
  @@map("event_subcategories")
}

enum EventStatus {
  PLANNING
  SCHEDULED
  IN_PROGRESS
  COMPLETED
  CANCELLED
  POSTPONED
  
  @@map("event_statuses")
}

model EventHistory {
  id                String   @id @default(cuid())
  eventId           String
  userId            String
  action            EventAction
  changes           Json     // Before/after values
  timestamp         DateTime @default(now())
  
  event             Event    @relation(fields: [eventId], references: [id])
  
  @@index([eventId, timestamp])
  @@map("event_history")
}

enum EventAction {
  CREATED
  UPDATED
  STATUS_CHANGED
  COMPLETED
  CANCELLED
  GUEST_REGISTERED
  GUEST_CHECKED_IN
  
  @@map("event_actions")
}

// ==================== GUEST MANAGEMENT ====================

model Guest {
  id                String   @id @default(cuid())
  email             String   @unique
  name              String
  nameJa            String?
  company           String?
  companyJa         String?
  jobTitle          String?
  phone             String?
  dietaryRestrictions String?
  interests         String[] // Array of interest tags
  notes             String?  @db.Text
  contactPreference ContactPreference @default(EMAIL)
  preferredLanguage Language @default(JA)
  gdprConsent       Boolean  @default(false)
  gdprConsentDate   DateTime?
  marketingConsent  Boolean  @default(false)
  source            String?  // How they joined (form, manual, import)
  deletedAt         DateTime? // Soft delete for GDPR
  createdAt         DateTime @default(now())
  updatedAt         DateTime @updatedAt
  
  // Relations
  eventRegistrations EventRegistration[]
  formResponses     FormResponse[]
  preferences       GuestPreference[]
  
  @@index([email])
  @@index([company])
  @@map("guests")
}

model GuestPreference {
  id                String   @id @default(cuid())
  guestId           String
  category          String   // preference category
  value             String   // preference value
  createdAt         DateTime @default(now())
  
  guest             Guest    @relation(fields: [guestId], references: [id])
  
  @@unique([guestId, category])
  @@map("guest_preferences")
}

model EventRegistration {
  id                String   @id @default(cuid())
  eventId           String
  guestId           String
  status            RegistrationStatus @default(PENDING)
  registrationSource String? // manual, form, api
  registeredAt      DateTime @default(now())
  confirmedAt       DateTime?
  attendedAt        DateTime?
  checkInMethod     String?  // manual, qr, self
  notes             String?
  customFields      Json?    // Event-specific data
  
  event             Event    @relation(fields: [eventId], references: [id])
  guest             Guest    @relation(fields: [guestId], references: [id])
  
  @@unique([eventId, guestId])
  @@index([eventId, status])
  @@index([guestId])
  @@map("event_registrations")
}

enum RegistrationStatus {
  PENDING
  CONFIRMED
  WAITLISTED
  ATTENDED
  NO_SHOW
  CANCELLED
  
  @@map("registration_statuses")
}

enum ContactPreference {
  EMAIL
  PHONE
  SLACK
  NO_CONTACT
  
  @@map("contact_preferences")
}

// ==================== TASK MANAGEMENT ====================

model Task {
  id                String   @id @default(cuid())
  title             String
  titleJa           String?
  description       String?  @db.Text
  descriptionJa     String?  @db.Text
  priority          TaskPriority @default(MEDIUM)
  status            TaskStatus @default(TODO)
  category          TaskCategory
  tags              String[]
  dueDate           DateTime?
  startDate         DateTime?
  completedAt       DateTime?
  estimatedHours    Float?
  actualHours       Float?
  progress          Int      @default(0) // 0-100
  assigneeId        String?
  createdBy         String
  eventId           String?
  parentTaskId      String?
  position          Int?     // For ordering within parent/status
  isRecurring       Boolean  @default(false)
  recurringConfig   Json?    // Recurrence rules
  metadata          Json?    // Flexible data
  deletedAt         DateTime? // Soft delete
  createdAt         DateTime @default(now())
  updatedAt         DateTime @updatedAt
  
  // Relations
  creator           User     @relation("TaskCreator", fields: [createdBy], references: [id])
  assignee          User?    @relation("TaskAssignee", fields: [assigneeId], references: [id])
  event             Event?   @relation(fields: [eventId], references: [id])
  parentTask        Task?    @relation("SubTasks", fields: [parentTaskId], references: [id])
  subTasks          Task[]   @relation("SubTasks")
  dependencies      TaskDependency[] @relation("DependentTask")
  dependents        TaskDependency[] @relation("DependsOnTask")
  taskTags          TaskTag[]
  timeEntries       TimeEntry[]
  comments          TaskComment[]
  attachments       TaskAttachment[]
  history           TaskHistory[]
  checklistItems    TaskChecklistItem[]
  
  @@index([status, assigneeId])
  @@index([eventId, status])
  @@index([dueDate, status])
  @@index([category, priority])
  @@index([parentTaskId])
  @@map("tasks")
}

enum TaskPriority {
  LOW
  MEDIUM
  HIGH
  URGENT
  
  @@map("task_priorities")
}

enum TaskStatus {
  TODO
  IN_PROGRESS
  IN_REVIEW
  DONE
  BLOCKED
  CANCELLED
  
  @@map("task_statuses")
}

enum TaskCategory {
  EVENT_PLANNING
  EVENT_EXECUTION
  WEBSITE_MANAGEMENT
  SEO
  PERFORMANCE
  SECURITY
  CONTENT
  COORDINATOR
  GENERAL
  
  @@map("task_categories")
}

model TaskDependency {
  id                String   @id @default(cuid())
  dependentTaskId   String
  dependsOnTaskId   String
  type              DependencyType @default(FINISH_TO_START)
  lagDays           Int      @default(0) // Days between tasks
  
  dependentTask     Task     @relation("DependentTask", fields: [dependentTaskId], references: [id])
  dependsOnTask     Task     @relation("DependsOnTask", fields: [dependsOnTaskId], references: [id])
  
  @@unique([dependentTaskId, dependsOnTaskId])
  @@map("task_dependencies")
}

enum DependencyType {
  FINISH_TO_START
  START_TO_START
  FINISH_TO_FINISH
  START_TO_FINISH
  
  @@map("dependency_types")
}

model TaskTag {
  id                String   @id @default(cuid())
  name              String   @unique
  nameJa            String?
  color             String   // Hex color
  description       String?
  tasks             Task[]
  
  @@map("task_tags")
}

model TaskChecklistItem {
  id                String   @id @default(cuid())
  taskId            String
  title             String
  isCompleted       Boolean  @default(false)
  completedAt       DateTime?
  completedBy       String?
  position          Int
  
  task              Task     @relation(fields: [taskId], references: [id])
  
  @@index([taskId, position])
  @@map("task_checklist_items")
}

// ==================== TIME TRACKING ====================

model TimeEntry {
  id                String   @id @default(cuid())
  taskId            String
  userId            String
  startTime         DateTime
  endTime           DateTime?
  duration          Int?     // Minutes
  description       String?
  isBillable        Boolean  @default(false)
  createdAt         DateTime @default(now())
  
  task              Task     @relation(fields: [taskId], references: [id])
  user              User     @relation(fields: [userId], references: [id])
  
  @@index([taskId, userId])
  @@index([startTime])
  @@map("time_entries")
}

// ==================== COLLABORATION ====================

model TaskComment {
  id                String   @id @default(cuid())
  taskId            String
  userId            String
  content           String   @db.Text
  mentions          String[] // Array of user IDs
  isEdited          Boolean  @default(false)
  editedAt          DateTime?
  createdAt         DateTime @default(now())
  updatedAt         DateTime @updatedAt
  
  task              Task     @relation(fields: [taskId], references: [id])
  user              User     @relation(fields: [userId], references: [id])
  
  @@index([taskId, createdAt])
  @@map("task_comments")
}

model TaskAttachment {
  id                String   @id @default(cuid())
  taskId            String
  fileName          String
  fileUrl           String
  fileSize          Int      // Bytes
  mimeType          String
  uploadedBy        String
  uploadedAt        DateTime @default(now())
  
  task              Task     @relation(fields: [taskId], references: [id])
  
  @@index([taskId])
  @@map("task_attachments")
}

model TaskHistory {
  id                String   @id @default(cuid())
  taskId            String
  userId            String
  action            TaskAction
  field             String?  // Which field changed
  oldValue          String?
  newValue          String?
  timestamp         DateTime @default(now())
  
  task              Task     @relation(fields: [taskId], references: [id])
  
  @@index([taskId, timestamp])
  @@map("task_history")
}

enum TaskAction {
  CREATED
  UPDATED
  STATUS_CHANGED
  ASSIGNED
  COMPLETED
  COMMENTED
  ATTACHMENT_ADDED
  
  @@map("task_actions")
}

// ==================== REPORTS ====================

model Report {
  id                String   @id @default(cuid())
  templateId        String
  reportType        ReportType @default(WEEKLY)
  reportDate        DateTime // Report period start date
  reportEndDate     DateTime // Report period end date
  status            ReportStatus @default(DRAFT)
  generatedAt       DateTime?
  publishedAt       DateTime?
  data              Json     // Aggregated report data
  htmlContent       String?  @db.Text
  pdfUrl            String?
  emailSubject      String?
  emailSubjectJa    String?
  createdBy         String
  approvedBy        String?
  approvedAt        DateTime?
  createdAt         DateTime @default(now())
  updatedAt         DateTime @updatedAt
  
  // Relations
  template          ReportTemplate @relation(fields: [templateId], references: [id])
  creator           User     @relation("ReportCreator", fields: [createdBy], references: [id])
  distributions     ReportDistribution[]
  sections          ReportSection[]
  
  @@index([reportDate, status])
  @@index([reportType, status])
  @@map("reports")
}

enum ReportType {
  WEEKLY
  MONTHLY
  QUARTERLY
  ANNUAL
  CUSTOM
  
  @@map("report_types")
}

enum ReportStatus {
  DRAFT
  GENERATING
  GENERATED
  APPROVED
  PUBLISHED
  SENT
  FAILED
  
  @@map("report_statuses")
}

model ReportTemplate {
  id                String   @id @default(cuid())
  name              String   @unique
  nameJa            String
  description       String?
  type              ReportType
  isActive          Boolean  @default(true)
  isDefault         Boolean  @default(false)
  htmlTemplate      String   @db.Text // Handlebars template
  cssStyles         String?  @db.Text
  headerTemplate    String?  @db.Text
  footerTemplate    String?  @db.Text
  sections          Json     // Section configuration
  variables         Json?    // Template variables
  createdBy         String
  createdAt         DateTime @default(now())
  updatedAt         DateTime @updatedAt
  
  reports           Report[]
  
  @@map("report_templates")
}

model ReportSection {
  id                String   @id @default(cuid())
  reportId          String
  sectionType       ReportSectionType
  title             String
  titleJa           String
  content           Json     // Section data
  freeText          String?  @db.Text
  freeTextJa        String?  @db.Text
  order             Int
  isVisible         Boolean  @default(true)
  
  report            Report   @relation(fields: [reportId], references: [id])
  
  @@unique([reportId, sectionType])
  @@index([reportId, order])
  @@map("report_sections")
}

enum ReportSectionType {
  WEBSITE_MANAGEMENT
  SELF_HOSTING_MEETUP
  SELF_HOSTING_CAREER
  SUPPORTING_EVENTS
  COORDINATOR_TASKS
  SUPPORTING_GRAND
  OTHERS
  SUMMARY
  
  @@map("report_section_types")
}

model ReportDistribution {
  id                String   @id @default(cuid())
  reportId          String
  recipientEmail    String
  recipientName     String
  recipientType     RecipientType
  language          Language @default(JA)
  sentAt            DateTime?
  openedAt          DateTime?
  downloadedAt      DateTime?
  errorMessage      String?
  
  report            Report   @relation(fields: [reportId], references: [id])
  
  @@index([reportId, sentAt])
  @@map("report_distributions")
}

enum RecipientType {
  CLIENT
  INTERNAL
  STAKEHOLDER
  SUBSCRIBER
  
  @@map("recipient_types")
}

model ClientReport {
  id                String   @id @default(cuid())
  eventId           String
  title             String
  titleJa           String?
  content           String   @db.Text
  contentJa         String?  @db.Text
  reportType        ClientReportType
  attachments       Json?    // Array of attachment URLs
  isPublished       Boolean  @default(false)
  publishedAt       DateTime?
  createdBy         String
  createdAt         DateTime @default(now())
  updatedAt         DateTime @updatedAt
  
  event             Event    @relation(fields: [eventId], references: [id])
  
  @@index([eventId, reportType])
  @@map("client_reports")
}

enum ClientReportType {
  SUMMARY
  DETAILED
  METRICS
  FINANCIAL
  
  @@map("client_report_types")
}

// ==================== INTEGRATIONS ====================

model SlackIntegration {
  id                String   @id @default(cuid())
  workspaceId       String   @unique
  teamName          String
  teamDomain        String
  botUserId         String
  botAccessToken    String   @db.Text // Encrypted
  installedBy       String
  installedAt       DateTime @default(now())
  isActive          Boolean  @default(true)
  lastSyncAt        DateTime?
  
  channels          SlackChannel[]
  notifications     SlackNotificationConfig[]
  
  @@map("slack_integrations")
}

model SlackChannel {
  id                String   @id @default(cuid())
  integrationId     String
  channelId         String
  channelName       String
  channelType       SlackChannelType
  isDefault         Boolean  @default(false)
  isActive          Boolean  @default(true)
  
  integration       SlackIntegration @relation(fields: [integrationId], references: [id])
  
  @@unique([integrationId, channelId])
  @@map("slack_channels")
}

enum SlackChannelType {
  PUBLIC
  PRIVATE
  DIRECT
  
  @@map("slack_channel_types")
}

model SlackNotificationConfig {
  id                String   @id @default(cuid())
  integrationId     String
  eventType         NotificationEventType
  channelId         String
  enabled           Boolean  @default(true)
  template          String?  @db.Text
  conditions        Json?    // When to send
  
  integration       SlackIntegration @relation(fields: [integrationId], references: [id])
  
  @@unique([integrationId, eventType, channelId])
  @@map("slack_notification_configs")
}

model GoogleFormIntegration {
  id                String   @id @default(cuid())
  formId            String   @unique
  formTitle         String
  formUrl           String
  eventId           String?  @unique
  mappingConfig     Json     // Field mappings
  syncEnabled       Boolean  @default(true)
  lastSyncAt        DateTime?
  lastSyncStatus    String?
  webhookUrl        String?
  webhookSecret     String?
  totalResponses    Int      @default(0)
  createdBy         String
  createdAt         DateTime @default(now())
  updatedAt         DateTime @updatedAt
  
  event             Event?   @relation(fields: [eventId], references: [id])
  responses         FormResponse[]
  
  @@map("google_form_integrations")
}

model FormResponse {
  id                String   @id @default(cuid())
  integrationId     String
  responseId        String   @unique
  submittedAt       DateTime
  responseData      Json     // Raw response data
  mappedData        Json?    // Processed data
  processedAt       DateTime?
  processingError   String?
  guestId           String?
  
  integration       GoogleFormIntegration @relation(fields: [integrationId], references: [id])
  guest             Guest?   @relation(fields: [guestId], references: [id])
  
  @@index([integrationId, submittedAt])
  @@index([guestId])
  @@map("form_responses")
}

model EmailTemplate {
  id                String   @id @default(cuid())
  name              String   @unique
  description       String?
  subject           String
  subjectJa         String
  htmlTemplate      String   @db.Text
  textTemplate      String   @db.Text
  category          EmailCategory
  variables         Json     // Expected variables
  isActive          Boolean  @default(true)
  usageCount        Int      @default(0)
  lastUsedAt        DateTime?
  createdAt         DateTime @default(now())
  updatedAt         DateTime @updatedAt
  
  @@map("email_templates")
}

enum EmailCategory {
  NOTIFICATION
  REPORT
  EVENT
  TASK
  SYSTEM
  MARKETING
  
  @@map("email_categories")
}

// ==================== NOTIFICATIONS ====================

model Notification {
  id                String   @id @default(cuid())
  userId            String
  type              NotificationEventType
  title             String
  titleJa           String?
  message           String
  messageJa         String?
  data              Json?    // Related data
  isRead            Boolean  @default(false)
  readAt            DateTime?
  sentViaEmail      Boolean  @default(false)
  sentViaSlack      Boolean  @default(false)
  priority          NotificationPriority @default(NORMAL)
  expiresAt         DateTime?
  createdAt         DateTime @default(now())
  
  user              User     @relation(fields: [userId], references: [id])
  
  @@index([userId, isRead, createdAt])
  @@index([type, createdAt])
  @@map("notifications")
}

enum NotificationEventType {
  // Task notifications
  TASK_ASSIGNED
  TASK_COMPLETED
  TASK_OVERDUE
  TASK_DUE_SOON
  TASK_COMMENTED
  TASK_BLOCKED
  
  // Event notifications
  EVENT_CREATED
  EVENT_UPDATED
  EVENT_REMINDER
  EVENT_REGISTRATION
  EVENT_CANCELLED
  
  // Report notifications
  REPORT_GENERATED
  REPORT_PUBLISHED
  REPORT_FAILED
  
  // System notifications
  WEEKLY_SUMMARY
  SYSTEM_ANNOUNCEMENT
  INTEGRATION_ERROR
  
  @@map("notification_event_types")
}

enum NotificationPriority {
  LOW
  NORMAL
  HIGH
  URGENT
  
  @@map("notification_priorities")
}

// ==================== SYSTEM & AUDIT ====================

model AuditLog {
  id                String   @id @default(cuid())
  userId            String
  action            String   // e.g., "user.login", "event.create"
  entityType        String?  // e.g., "event", "task"
  entityId          String?
  changes           Json?    // What changed
  ipAddress         String?
  userAgent         String?
  requestId         String?
  timestamp         DateTime @default(now())
  
  user              User     @relation(fields: [userId], references: [id])
  
  @@index([userId, timestamp])
  @@index([entityType, entityId])
  @@index([action, timestamp])
  @@map("audit_logs")
}

model SystemSetting {
  id                String   @id @default(cuid())
  key               String   @unique
  value             Json
  description       String?
  isPublic          Boolean  @default(false) // Can be exposed to frontend
  updatedBy         String?
  updatedAt         DateTime @updatedAt
  
  @@map("system_settings")
}

model JobQueue {
  id                String   @id @default(cuid())
  type              JobType
  status            JobStatus @default(PENDING)
  payload           Json
  result            Json?
  error             String?
  attempts          Int      @default(0)
  maxAttempts       Int      @default(3)
  scheduledFor      DateTime @default(now())
  startedAt         DateTime?
  completedAt       DateTime?
  createdAt         DateTime @default(now())
  
  @@index([type, status, scheduledFor])
  @@map("job_queue")
}

enum JobType {
  REPORT_GENERATION
  EMAIL_SEND
  SLACK_NOTIFICATION
  FORM_SYNC
  DATA_EXPORT
  CLEANUP
  
  @@map("job_types")
}

enum JobStatus {
  PENDING
  PROCESSING
  COMPLETED
  FAILED
  CANCELLED
  
  @@map("job_statuses")
}
```

## 3. Database Indexes

### 3.1 Performance Indexes
```sql
-- Event queries
CREATE INDEX idx_events_date_range ON events(event_date, event_end_date);
CREATE INDEX idx_events_search ON events USING gin(to_tsvector('simple', title || ' ' || COALESCE(title_ja, '')));

-- Task queries
CREATE INDEX idx_tasks_assignee_status ON tasks(assignee_id, status) WHERE deleted_at IS NULL;
CREATE INDEX idx_tasks_due_date ON tasks(due_date) WHERE status NOT IN ('DONE', 'CANCELLED');
CREATE INDEX idx_tasks_search ON tasks USING gin(to_tsvector('simple', title || ' ' || COALESCE(description, '')));

-- Guest queries
CREATE INDEX idx_guests_email_lower ON guests(LOWER(email));
CREATE INDEX idx_guests_company ON guests(company) WHERE company IS NOT NULL;

-- Report queries
CREATE INDEX idx_reports_date_type ON reports(report_date, report_type);

-- Audit queries
CREATE INDEX idx_audit_logs_entity ON audit_logs(entity_type, entity_id, timestamp);
```

### 3.2 Foreign Key Indexes
```sql
-- Automatically created by Prisma for all foreign key relationships
```

## 4. Data Validation Rules

### 4.1 Business Rules
```typescript
// Event Validation
const eventValidation = {
  title: {
    minLength: 3,
    maxLength: 200,
    required: true
  },
  eventDate: {
    mustBeFuture: true,
    required: true
  },
  eventEndDate: {
    mustBeAfterStartDate: true
  },
  plannedAttendees: {
    min: 1,
    max: 10000
  },
  budget: {
    min: 0,
    max: 999999999.99
  }
}

// Task Validation
const taskValidation = {
  title: {
    minLength: 3,
    maxLength: 200,
    required: true
  },
  progress: {
    min: 0,
    max: 100
  },
  estimatedHours: {
    min: 0.25,
    max: 999.99
  },
  dueDate: {
    canBePast: false // Unless task is completed
  }
}

// Guest Validation
const guestValidation = {
  email: {
    format: 'email',
    required: true,
    unique: true
  },
  name: {
    minLength: 2,
    maxLength: 100,
    required: true
  },
  phone: {
    format: 'phone',
    optional: true
  }
}
```

### 4.2 Database Constraints
```sql
-- Check constraints
ALTER TABLE events ADD CONSTRAINT check_event_dates 
  CHECK (event_end_date IS NULL OR event_end_date >= event_date);

ALTER TABLE tasks ADD CONSTRAINT check_task_progress 
  CHECK (progress >= 0 AND progress <= 100);

ALTER TABLE tasks ADD CONSTRAINT check_task_hours 
  CHECK (estimated_hours > 0 AND actual_hours >= 0);

-- Unique constraints (in addition to Prisma)
ALTER TABLE events ADD CONSTRAINT unique_event_slug 
  UNIQUE (slug);

ALTER TABLE guests ADD CONSTRAINT unique_guest_email_case_insensitive 
  UNIQUE (LOWER(email));
```

## 5. Data Migration Strategy

### 5.1 Migration Approach
```typescript
// Migration utilities
class DataMigration {
  // Soft delete implementation
  async softDelete(model: string, id: string) {
    return prisma[model].update({
      where: { id },
      data: { deletedAt: new Date() }
    })
  }

  // Data archival
  async archiveOldData(model: string, beforeDate: Date) {
    const data = await prisma[model].findMany({
      where: { createdAt: { lt: beforeDate } }
    })
    
    // Move to archive tables
    await prisma[`${model}Archive`].createMany({ data })
    
    // Remove from main tables
    await prisma[model].deleteMany({
      where: { createdAt: { lt: beforeDate } }
    })
  }

  // Bilingual data migration
  async addJapaneseTranslations(model: string, translations: Map<string, string>) {
    for (const [id, translation] of translations) {
      await prisma[model].update({
        where: { id },
        data: { titleJa: translation }
      })
    }
  }
}
```

### 5.2 Seed Data
```typescript
// prisma/seed.ts
const seedData = {
  // Default templates
  reportTemplates: [
    {
      name: 'default_weekly',
      nameJa: 'デフォルト週次レポート',
      type: 'WEEKLY',
      isDefault: true,
      htmlTemplate: defaultWeeklyTemplate,
      sections: defaultWeeklySections
    }
  ],

  // System settings
  systemSettings: [
    {
      key: 'default_timezone',
      value: 'Asia/Tokyo',
      isPublic: true
    },
    {
      key: 'report_generation_time',
      value: '09:00',
      isPublic: false
    }
  ],

  // Default tags
  taskTags: [
    { name: 'urgent', nameJa: '緊急', color: '#FF0000' },
    { name: 'bug', nameJa: 'バグ', color: '#FF6B6B' },
    { name: 'feature', nameJa: '機能', color: '#4ECDC4' },
    { name: 'improvement', nameJa: '改善', color: '#45B7D1' }
  ]
}
```

## 6. Performance Optimization

### 6.1 Query Optimization
```typescript
// Efficient pagination
async function paginateWithCursor(model: string, cursor?: string, limit = 20) {
  return prisma[model].findMany({
    take: limit + 1,
    cursor: cursor ? { id: cursor } : undefined,
    orderBy: { createdAt: 'desc' }
  })
}

// Selective field loading
async function getEventList() {
  return prisma.event.findMany({
    select: {
      id: true,
      title: true,
      titleJa: true,
      eventDate: true,
      status: true,
      _count: {
        select: {
          guests: true,
          tasks: true
        }
      }
    }
  })
}

// Aggregation queries
async function getTaskStats(eventId: string) {
  return prisma.task.groupBy({
    by: ['status'],
    where: { eventId },
    _count: true,
    _avg: {
      progress: true
    }
  })
}
```

### 6.2 Caching Strategy
```typescript
// Redis cache keys
const cacheKeys = {
  userPreferences: (userId: string) => `user:${userId}:preferences`,
  eventGuests: (eventId: string) => `event:${eventId}:guests`,
  reportData: (reportId: string) => `report:${reportId}:data`,
  taskProgress: (taskId: string) => `task:${taskId}:progress`
}

// Cache invalidation
const cacheInvalidation = {
  onTaskUpdate: (taskId: string) => {
    redis.del(cacheKeys.taskProgress(taskId))
  },
  onEventUpdate: (eventId: string) => {
    redis.del(cacheKeys.eventGuests(eventId))
  }
}
```

## 7. Data Security

### 7.1 Encryption
```typescript
// Field-level encryption for sensitive data
const encryptedFields = {
  slackIntegration: ['botAccessToken'],
  googleFormIntegration: ['webhookSecret'],
  user: ['phoneNumber'] // If storing phone numbers
}

// Encryption middleware
prisma.$use(async (params, next) => {
  if (params.model && encryptedFields[params.model]) {
    // Encrypt on create/update
    if (params.action === 'create' || params.action === 'update') {
      encryptFields(params.args.data, encryptedFields[params.model])
    }
  }
  
  const result = await next(params)
  
  // Decrypt on read
  if (params.model && encryptedFields[params.model]) {
    decryptFields(result, encryptedFields[params.model])
  }
  
  return result
})
```

### 7.2 Data Privacy
```typescript
// GDPR compliance
class PrivacyService {
  // Anonymize guest data
  async anonymizeGuest(guestId: string) {
    return prisma.guest.update({
      where: { id: guestId },
      data: {
        email: `deleted-${guestId}@example.com`,
        name: 'Deleted User',
        nameJa: '削除されたユーザー',
        phone: null,
        company: null,
        notes: null,
        deletedAt: new Date()
      }
    })
  }

  // Export user data
  async exportUserData(userId: string) {
    const data = await prisma.$transaction([
      prisma.user.findUnique({ where: { id: userId } }),
      prisma.task.findMany({ where: { OR: [{ createdBy: userId }, { assigneeId: userId }] } }),
      prisma.event.findMany({ where: { createdBy: userId } }),
      prisma.auditLog.findMany({ where: { userId } })
    ])
    
    return {
      user: data[0],
      tasks: data[1],
      events: data[2],
      auditLogs: data[3]
    }
  }
}
```

## 8. Backup and Recovery

### 8.1 Backup Strategy
```sql
-- Daily backup script
pg_dump -h $DB_HOST -U $DB_USER -d $DB_NAME -f backup_$(date +%Y%m%d).sql

-- Point-in-time recovery setup
archive_mode = on
archive_command = 'cp %p /backup/archive/%f'
```

### 8.2 Data Recovery Procedures
```typescript
// Soft delete recovery
async function recoverDeleted(model: string, id: string) {
  return prisma[model].update({
    where: { id },
    data: { deletedAt: null }
  })
}

// Audit trail recovery
async function revertChanges(auditLogId: string) {
  const log = await prisma.auditLog.findUnique({
    where: { id: auditLogId }
  })
  
  if (log?.changes?.before) {
    await prisma[log.entityType].update({
      where: { id: log.entityId },
      data: log.changes.before
    })
  }
}
```