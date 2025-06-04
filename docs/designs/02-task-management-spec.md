# Task Management System Design Specification

## 1. Overview

### 1.1 Purpose
The Task Management System provides comprehensive task tracking capabilities for IT engineering teams, with deep integration to event management and automated reporting features.

### 1.2 Tech Stack
- **Frontend**: Next.js 14+ with App Router
- **Backend**: Next.js API Routes
- **Database**: PostgreSQL with Prisma ORM
- **Authentication**: Auth0
- **Real-time Updates**: Server-Sent Events (SSE)
- **Localization**: i18next for Japanese/English support

## 2. Data Models

### 2.1 Task Model
```typescript
model Task {
  id              String   @id @default(cuid())
  title           String
  titleJa         String?
  description     String?  @db.Text
  descriptionJa   String?  @db.Text
  priority        TaskPriority @default(MEDIUM)
  status          TaskStatus @default(TODO)
  category        TaskCategory
  dueDate         DateTime?
  completedAt     DateTime?
  estimatedHours  Float?
  actualHours     Float?
  assigneeId      String?  // Auth0 user ID
  createdBy       String   // Auth0 user ID
  createdAt       DateTime @default(now())
  updatedAt       DateTime @updatedAt
  
  // Relations
  eventId         String?
  parentTaskId    String?
  event           Event?   @relation(fields: [eventId], references: [id])
  parentTask      Task?    @relation("SubTasks", fields: [parentTaskId], references: [id])
  subTasks        Task[]   @relation("SubTasks")
  dependencies    TaskDependency[] @relation("DependentTask")
  dependents      TaskDependency[] @relation("DependsOnTask")
  tags            TaskTag[]
  timeEntries     TimeEntry[]
  comments        TaskComment[]
  history         TaskHistory[]
  attachments     TaskAttachment[]
}

enum TaskPriority {
  LOW
  MEDIUM
  HIGH
  URGENT
}

enum TaskStatus {
  TODO
  IN_PROGRESS
  REVIEW
  DONE
  BLOCKED
  CANCELLED
}

enum TaskCategory {
  EVENT_PLANNING
  EVENT_EXECUTION
  WEBSITE_MANAGEMENT
  COORDINATOR
  GENERAL
}
```

### 2.2 Task Dependencies
```typescript
model TaskDependency {
  id              String   @id @default(cuid())
  dependentTaskId String
  dependsOnTaskId String
  type            DependencyType @default(FINISH_TO_START)
  
  dependentTask   Task     @relation("DependentTask", fields: [dependentTaskId], references: [id])
  dependsOnTask   Task     @relation("DependsOnTask", fields: [dependsOnTaskId], references: [id])
  
  @@unique([dependentTaskId, dependsOnTaskId])
}

enum DependencyType {
  FINISH_TO_START    // Default: Task B starts after Task A finishes
  START_TO_START     // Task B starts when Task A starts
  FINISH_TO_FINISH   // Task B finishes when Task A finishes
  START_TO_FINISH    // Task B finishes when Task A starts
}
```

### 2.3 Time Tracking
```typescript
model TimeEntry {
  id          String   @id @default(cuid())
  taskId      String
  userId      String   // Auth0 user ID
  startTime   DateTime
  endTime     DateTime?
  duration    Int?     // in minutes
  description String?
  createdAt   DateTime @default(now())
  
  task        Task     @relation(fields: [taskId], references: [id])
}
```

### 2.4 Task Metadata
```typescript
model TaskTag {
  id          String   @id @default(cuid())
  name        String   @unique
  nameJa      String?
  color       String   // Hex color code
  tasks       Task[]
}

model TaskComment {
  id          String   @id @default(cuid())
  taskId      String
  userId      String
  content     String   @db.Text
  createdAt   DateTime @default(now())
  updatedAt   DateTime @updatedAt
  
  task        Task     @relation(fields: [taskId], references: [id])
}

model TaskAttachment {
  id          String   @id @default(cuid())
  taskId      String
  fileName    String
  fileUrl     String
  fileSize    Int
  mimeType    String
  uploadedBy  String
  uploadedAt  DateTime @default(now())
  
  task        Task     @relation(fields: [taskId], references: [id])
}
```

## 3. API Endpoints

### 3.1 Task Management APIs
```typescript
// Task CRUD
POST   /api/tasks                     // Create task
GET    /api/tasks                     // List tasks with filters
GET    /api/tasks/:id                 // Get task details
PUT    /api/tasks/:id                 // Update task
DELETE /api/tasks/:id                 // Soft delete task
POST   /api/tasks/:id/duplicate       // Duplicate task with subtasks

// Task Status Management
PUT    /api/tasks/:id/status          // Update task status
POST   /api/tasks/:id/complete        // Mark as complete
POST   /api/tasks/:id/reopen          // Reopen completed task

// Bulk Operations
POST   /api/tasks/bulk/update         // Bulk update tasks
POST   /api/tasks/bulk/assign         // Bulk assign tasks
POST   /api/tasks/bulk/delete         // Bulk delete tasks
```

### 3.2 Subtask and Dependency APIs
```typescript
// Subtask Management
POST   /api/tasks/:id/subtasks        // Create subtask
GET    /api/tasks/:id/subtasks        // List subtasks
PUT    /api/tasks/:id/convert-subtask // Convert to subtask

// Dependency Management
POST   /api/tasks/:id/dependencies    // Add dependency
DELETE /api/tasks/:id/dependencies/:depId // Remove dependency
GET    /api/tasks/:id/dependency-graph // Get dependency visualization
```

### 3.3 Time Tracking APIs
```typescript
// Time Entries
POST   /api/tasks/:id/time/start      // Start timer
POST   /api/tasks/:id/time/stop       // Stop timer
POST   /api/tasks/:id/time/manual     // Add manual time entry
GET    /api/tasks/:id/time            // Get time entries
DELETE /api/time-entries/:id          // Delete time entry
```

### 3.4 Collaboration APIs
```typescript
// Comments and Attachments
POST   /api/tasks/:id/comments        // Add comment
GET    /api/tasks/:id/comments        // List comments
PUT    /api/comments/:id              // Edit comment
DELETE /api/comments/:id              // Delete comment

POST   /api/tasks/:id/attachments     // Upload attachment
GET    /api/tasks/:id/attachments     // List attachments
DELETE /api/attachments/:id           // Delete attachment
```

## 4. UI Components and Pages

### 4.1 Task Dashboard (`/tasks`)
```typescript
interface TaskDashboardProps {
  view: 'kanban' | 'list' | 'calendar' | 'timeline'
  filters: TaskFilters
}

// Views:
// 1. Kanban Board - Drag & drop between status columns
// 2. List View - Sortable table with inline editing
// 3. Calendar View - Tasks plotted by due date
// 4. Timeline View - Gantt chart with dependencies
```

### 4.2 Kanban Board Component
```typescript
interface KanbanBoardProps {
  columns: {
    TODO: Task[]
    IN_PROGRESS: Task[]
    REVIEW: Task[]
    DONE: Task[]
    BLOCKED: Task[]
  }
  onDragEnd: (taskId: string, newStatus: TaskStatus) => void
}

// Features:
// - Drag & drop between columns
// - Quick add task in each column
// - Task count limits per column
// - Swimlanes by assignee/category
// - Collapsed/expanded task cards
```

### 4.3 Task Detail Modal/Page (`/tasks/:id`)
```typescript
interface TaskDetailProps {
  taskId: string
  mode: 'modal' | 'page'
}

// Sections:
// 1. Header - Title, status, priority, assignee
// 2. Details - Description, dates, category, tags
// 3. Subtasks - Nested task list with progress
// 4. Dependencies - Visual dependency graph
// 5. Time Tracking - Timer, time entries, total time
// 6. Activity - Comments, attachments, history
```

### 4.4 Task Creation/Edit Form
```typescript
interface TaskFormProps {
  mode: 'create' | 'edit'
  taskId?: string
  eventId?: string // Pre-link to event
  parentTaskId?: string // Create as subtask
}

// Form Fields:
// - Title (bilingual)
// - Description (rich text, bilingual)
// - Priority selector
// - Category selector
// - Due date picker (timezone aware)
// - Assignee selector
// - Tags multi-select
// - Event link (optional)
// - Estimated hours
// - Dependencies selector
```

## 5. Business Logic

### 5.1 Task Lifecycle Management
```typescript
class TaskService {
  async createTask(data: CreateTaskDto, userId: string) {
    // 1. Validate dependencies don't create cycles
    // 2. Set initial status based on dependencies
    // 3. Auto-assign if rules configured
    // 4. Create subtasks if template used
    // 5. Send notification to assignee
    // 6. Update parent task progress
  }

  async updateTaskStatus(taskId: string, newStatus: TaskStatus) {
    // 1. Check dependency constraints
    // 2. Validate status transition rules
    // 3. Update completion timestamp
    // 4. Trigger dependent task notifications
    // 5. Update parent task progress
    // 6. Log status change history
  }

  async calculateTaskProgress(taskId: string): number {
    // 1. Get all subtasks recursively
    // 2. Calculate weighted completion
    // 3. Consider time tracking vs estimates
    // 4. Return percentage (0-100)
  }
}
```

### 5.2 Event-Task Integration
```typescript
class EventTaskService {
  async createEventTasks(eventId: string, template: EventTemplate) {
    // 1. Load task template for event type
    // 2. Calculate task due dates from event date
    // 3. Create tasks with proper dependencies
    // 4. Assign based on role mappings
    // 5. Link all tasks to event
  }

  async getEventTaskProgress(eventId: string) {
    // 1. Get all tasks linked to event
    // 2. Group by category/phase
    // 3. Calculate completion percentages
    // 4. Identify blockers and risks
    // 5. Return structured progress report
  }
}
```

### 5.3 Automated Task Assignment
```typescript
interface AssignmentRule {
  category: TaskCategory
  eventType?: EventSubcategory
  defaultAssignee?: string
  roundRobin?: string[] // User IDs for rotation
  loadBalance?: boolean
}

class TaskAssignmentService {
  async autoAssign(task: Task): string | null {
    // 1. Check assignment rules
    // 2. Consider current workload
    // 3. Check user availability
    // 4. Apply round-robin if configured
    // 5. Return assigned user ID
  }
}
```

## 6. Advanced Features

### 6.1 Task Templates
```typescript
interface TaskTemplate {
  id: string
  name: string
  nameJa: string
  category: TaskCategory
  tasks: TemplateTask[]
}

interface TemplateTask {
  title: string
  titleJa: string
  relativeStartDays: number // Days from event/project start
  relativeDueDays: number
  estimatedHours: number
  dependencies: string[] // Template task IDs
  subtasks: TemplateTask[]
}
```

### 6.2 Recurring Tasks
```typescript
interface RecurringTaskConfig {
  pattern: 'daily' | 'weekly' | 'monthly' | 'custom'
  interval: number
  daysOfWeek?: number[] // 0-6 for weekly
  dayOfMonth?: number // 1-31 for monthly
  endDate?: Date
  maxOccurrences?: number
}

class RecurringTaskService {
  async generateNextOccurrence(taskId: string) {
    // 1. Clone task with new dates
    // 2. Link to recurring series
    // 3. Update dependencies
    // 4. Notify assignee
  }
}
```

### 6.3 Smart Notifications
```typescript
interface NotificationRule {
  trigger: 'due_soon' | 'overdue' | 'blocked' | 'mentioned' | 'assigned'
  channel: 'email' | 'slack' | 'in_app'
  timing: number // Minutes before/after trigger
}

class TaskNotificationService {
  async sendReminder(taskId: string, rule: NotificationRule) {
    // 1. Check user notification preferences
    // 2. Format message in user's language
    // 3. Include task context and links
    // 4. Send via configured channel
    // 5. Log notification sent
  }
}
```

## 7. Integration Features

### 7.1 Slack Integration
```typescript
interface SlackTaskNotification {
  taskId: string
  action: 'created' | 'assigned' | 'completed' | 'blocked'
  userId: string
  channelId: string
  threadTs?: string // For threading
}

// Slack Commands:
// /task create [title] - Create quick task
// /task list - Show my tasks
// /task complete [id] - Mark task complete
```

### 7.2 Email Integration
```typescript
interface EmailTaskSummary {
  userId: string
  frequency: 'daily' | 'weekly'
  includeOverdue: boolean
  includeUpcoming: boolean
  includeDelegated: boolean
}
```

### 7.3 Calendar Integration
```typescript
interface CalendarTaskSync {
  provider: 'google' | 'outlook'
  syncEnabled: boolean
  calendarId: string
  createEvents: boolean
  updateEvents: boolean
}
```

## 8. Performance Optimizations

### 8.1 Query Optimization
```typescript
// Indexes
@@index([status, assigneeId])
@@index([eventId, status])
@@index([dueDate, status])
@@index([category, priority])

// Eager Loading
const taskWithDetails = await prisma.task.findUnique({
  where: { id },
  include: {
    subTasks: true,
    dependencies: { include: { dependsOnTask: true } },
    timeEntries: { orderBy: { startTime: 'desc' } },
    comments: { orderBy: { createdAt: 'desc' }, take: 10 }
  }
})
```

### 8.2 Caching Strategy
- Cache task lists per user (5 min TTL)
- Cache dependency graphs (10 min TTL)
- Real-time invalidation on updates
- Optimistic UI updates

## 9. Security and Permissions

### 9.1 Task Permissions
```typescript
interface TaskPermissions {
  create: ['all_users']
  view: ['owner', 'assignee', 'team_members']
  edit: ['owner', 'assignee', 'admin']
  delete: ['owner', 'admin']
  assign: ['owner', 'team_lead', 'admin']
  complete: ['assignee', 'owner', 'admin']
}
```

### 9.2 Data Isolation
- Filter tasks by user's team/department
- Hide sensitive tasks based on category
- Audit trail for all modifications
- Secure attachment storage with presigned URLs

## 10. Reporting Integration

### 10.1 Task Metrics for Reports
```typescript
interface TaskReportMetrics {
  totalTasks: number
  completedTasks: number
  overdueTasks: number
  blockedTasks: number
  averageCompletionTime: number
  tasksByCategory: Record<TaskCategory, number>
  tasksByPriority: Record<TaskPriority, number>
  upcomingDeadlines: Task[]
}
```

### 10.2 Progress Tracking
- Real-time progress calculation
- Historical progress trends
- Burndown charts for events
- Team workload distribution