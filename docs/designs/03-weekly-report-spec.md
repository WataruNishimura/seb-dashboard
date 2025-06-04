# Weekly Report System Design Specification

## 1. Overview

### 1.1 Purpose
The Weekly Report System automatically generates comprehensive reports every Monday, aggregating data from events and tasks to provide stakeholders with a structured overview of IT engineering activities.

### 1.2 Tech Stack
- **Frontend**: Next.js 14+ with App Router
- **Backend**: Next.js API Routes
- **Database**: PostgreSQL with Prisma ORM
- **PDF Generation**: Puppeteer or React PDF
- **Template Engine**: Handlebars.js
- **Scheduling**: Vercel Cron Jobs
- **Email Service**: SendGrid or AWS SES

## 2. Data Models

### 2.1 Report Template Model
```typescript
model ReportTemplate {
  id              String   @id @default(cuid())
  name            String
  nameJa          String
  description     String?
  isActive        Boolean  @default(true)
  isDefault       Boolean  @default(false)
  htmlTemplate    String   @db.Text // Handlebars template
  cssStyles       String?  @db.Text
  sections        Json     // Section configuration
  createdBy       String   // Admin user ID
  createdAt       DateTime @default(now())
  updatedAt       DateTime @updatedAt
  
  reports         Report[]
}

interface SectionConfig {
  id: string
  name: string
  nameJa: string
  type: 'website_management' | 'self_hosting_meetup' | 'self_hosting_career' | 
        'supporting_events' | 'coordinator_tasks' | 'supporting_grand' | 'others'
  enabled: boolean
  order: number
  config: {
    showProgress?: boolean
    timeRange?: 'week' | 'month' | 'custom'
    includeCompleted?: boolean
    maxItems?: number
  }
}
```

### 2.2 Report Model
```typescript
model Report {
  id              String   @id @default(cuid())
  templateId      String
  reportDate      DateTime // Monday of the report week
  status          ReportStatus @default(DRAFT)
  generatedAt     DateTime?
  publishedAt     DateTime?
  data            Json     // Aggregated report data
  htmlContent     String?  @db.Text
  pdfUrl          String?
  createdBy       String
  createdAt       DateTime @default(now())
  updatedAt       DateTime @updatedAt
  
  template        ReportTemplate @relation(fields: [templateId], references: [id])
  distributions   ReportDistribution[]
  sections        ReportSection[]
}

enum ReportStatus {
  DRAFT
  GENERATING
  GENERATED
  PUBLISHED
  FAILED
}
```

### 2.3 Report Sections Model
```typescript
model ReportSection {
  id              String   @id @default(cuid())
  reportId        String
  sectionType     String
  title           String
  titleJa         String
  content         Json     // Section-specific data
  freeText        String?  @db.Text
  freeTextJa      String?  @db.Text
  order           Int
  
  report          Report   @relation(fields: [reportId], references: [id])
}
```

### 2.4 Report Distribution Model
```typescript
model ReportDistribution {
  id              String   @id @default(cuid())
  reportId        String
  recipientEmail  String
  recipientName   String
  recipientType   RecipientType
  sentAt          DateTime?
  openedAt        DateTime?
  downloadedAt    DateTime?
  
  report          Report   @relation(fields: [reportId], references: [id])
}

enum RecipientType {
  CLIENT
  INTERNAL
  STAKEHOLDER
}
```

## 3. Report Data Structure

### 3.1 Report Data Schema
```typescript
interface WeeklyReportData {
  reportPeriod: {
    startDate: Date // Previous Monday
    endDate: Date   // Sunday
    weekNumber: number
    year: number
  }
  
  sections: {
    websiteManagement: WebsiteManagementSection
    selfHostingMeetup: EventSection
    selfHostingCareer: EventSection
    supportingEvents: EventSection
    coordinatorTasks: CoordinatorSection
    supportingGrand: FreeTextSection
    others: FreeTextSection
  }
  
  summary: {
    totalEvents: number
    completedEvents: number
    upcomingEvents: number
    totalTasks: number
    completedTasks: number
    overdueTasks: number
  }
}
```

### 3.2 Section Data Types
```typescript
interface WebsiteManagementSection {
  issues: Array<{
    id: string
    title: string
    titleJa: string
    category: 'SEO' | 'Performance' | 'Security' | 'Content' | 'Other'
    status: 'Open' | 'In Progress' | 'Resolved'
    priority: TaskPriority
    progress: number // 0-100
    assignee: string
    createdDate: Date
    updatedDate: Date
    linkedTasks: Array<{
      id: string
      title: string
      status: TaskStatus
      progress: number
    }>
  }>
  nextActions: string
  nextActionsJa: string
}

interface EventSection {
  upcomingEvents: Array<{
    id: string
    title: string
    titleJa: string
    date: Date
    location: string
    expectedAttendees: number
    status: EventStatus
    tasks: Array<{
      id: string
      title: string
      status: TaskStatus
      assignee: string
      dueDate: Date
    }>
  }>
  recentlyCompleted: Array<{
    id: string
    title: string
    titleJa: string
    date: Date
    actualAttendees: number
    reportUrl?: string
  }>
  tasksProgress: {
    total: number
    completed: number
    inProgress: number
    blocked: number
  }
}

interface CoordinatorSection {
  tasks: Array<{
    id: string
    title: string
    titleJa: string
    status: TaskStatus
    priority: TaskPriority
    assignee: string
    dueDate?: Date
    progress: number
  }>
  summary: {
    total: number
    byStatus: Record<TaskStatus, number>
    byPriority: Record<TaskPriority, number>
  }
}

interface FreeTextSection {
  content: string
  contentJa: string
  lastUpdated: Date
  updatedBy: string
}
```

## 4. Template System

### 4.1 Handlebars Template Structure
```handlebars
<!DOCTYPE html>
<html lang="{{language}}">
<head>
  <meta charset="UTF-8">
  <title>{{#if isJapanese}}週次レポート{{else}}Weekly Report{{/if}} - {{reportDate}}</title>
  <style>{{{cssStyles}}}</style>
</head>
<body>
  <header>
    <h1>{{#if isJapanese}}週次活動レポート{{else}}Weekly Activity Report{{/if}}</h1>
    <p>{{formatDate reportPeriod.startDate}} - {{formatDate reportPeriod.endDate}}</p>
  </header>

  {{#if sections.websiteManagement.enabled}}
  <section class="website-management">
    <h2>{{#if isJapanese}}ウェブサイト管理{{else}}Website Management{{/if}}</h2>
    
    {{#each websiteManagement.issues}}
    <div class="issue-item">
      <h3>{{#if ../isJapanese}}{{titleJa}}{{else}}{{title}}{{/if}}</h3>
      <div class="progress-bar">
        <div class="progress-fill" style="width: {{progress}}%"></div>
      </div>
      <p>{{i18n "Status"}}: {{status}} | {{i18n "Progress"}}: {{progress}}%</p>
      
      {{#if linkedTasks}}
      <ul class="task-list">
        {{#each linkedTasks}}
        <li>{{title}} - {{status}}</li>
        {{/each}}
      </ul>
      {{/if}}
    </div>
    {{/each}}
    
    <div class="next-actions">
      <h3>{{i18n "Next Actions"}}</h3>
      <p>{{#if isJapanese}}{{websiteManagement.nextActionsJa}}{{else}}{{websiteManagement.nextActions}}{{/if}}</p>
    </div>
  </section>
  {{/if}}

  {{#if sections.selfHostingMeetup.enabled}}
  <section class="self-hosting-meetup">
    <h2>{{#if isJapanese}}自社開催ミートアップイベント{{else}}Self-hosting Meetup Events{{/if}}</h2>
    
    {{> eventSectionPartial section=selfHostingMeetup}}
  </section>
  {{/if}}

  <!-- Continue for other sections... -->
</body>
</html>
```

### 4.2 Template Helpers
```typescript
const templateHelpers = {
  formatDate: (date: Date, format?: string) => {
    // Format date based on locale
    return format === 'ja' 
      ? `${date.getFullYear()}年${date.getMonth() + 1}月${date.getDate()}日`
      : date.toLocaleDateString('en-US')
  },
  
  i18n: (key: string, locale: string = 'ja') => {
    // Translation helper
    return translations[locale][key] || key
  },
  
  progressClass: (progress: number) => {
    if (progress >= 80) return 'progress-good'
    if (progress >= 50) return 'progress-medium'
    return 'progress-low'
  },
  
  statusIcon: (status: string) => {
    const icons = {
      'COMPLETED': '✓',
      'IN_PROGRESS': '◐',
      'TODO': '○',
      'BLOCKED': '✗'
    }
    return icons[status] || '•'
  }
}
```

## 5. Report Generation Service

### 5.1 Report Generator
```typescript
class ReportGenerationService {
  async generateWeeklyReport(templateId: string, reportDate: Date) {
    try {
      // 1. Create report record
      const report = await this.createReportRecord(templateId, reportDate)
      
      // 2. Collect data for each section
      const reportData = await this.collectReportData(reportDate)
      
      // 3. Generate HTML content
      const htmlContent = await this.renderTemplate(templateId, reportData)
      
      // 4. Generate PDF
      const pdfUrl = await this.generatePDF(htmlContent, report.id)
      
      // 5. Update report status
      await this.updateReportStatus(report.id, 'GENERATED', {
        data: reportData,
        htmlContent,
        pdfUrl
      })
      
      return report
    } catch (error) {
      await this.handleGenerationError(report.id, error)
      throw error
    }
  }

  private async collectReportData(reportDate: Date): WeeklyReportData {
    const startDate = startOfWeek(reportDate, { weekStartsOn: 1 }) // Monday
    const endDate = endOfWeek(reportDate, { weekStartsOn: 1 }) // Sunday
    
    const [
      websiteManagement,
      selfHostingMeetup,
      selfHostingCareer,
      supportingEvents,
      coordinatorTasks
    ] = await Promise.all([
      this.collectWebsiteManagementData(startDate, endDate),
      this.collectEventData('SELF_HOSTING', 'MEETUP', startDate, endDate),
      this.collectEventData('SELF_HOSTING', 'CAREER', startDate, endDate),
      this.collectSupportingEventData(startDate, endDate),
      this.collectCoordinatorTasks(startDate, endDate)
    ])
    
    return {
      reportPeriod: {
        startDate,
        endDate,
        weekNumber: getWeek(reportDate),
        year: reportDate.getFullYear()
      },
      sections: {
        websiteManagement,
        selfHostingMeetup,
        selfHostingCareer,
        supportingEvents,
        coordinatorTasks,
        supportingGrand: await this.getFreeTextSection('supporting_grand'),
        others: await this.getFreeTextSection('others')
      },
      summary: await this.calculateSummary(startDate, endDate)
    }
  }
}
```

### 5.2 Data Collection Methods
```typescript
class ReportDataCollector {
  async collectWebsiteManagementData(startDate: Date, endDate: Date) {
    // Get all website-related tasks
    const issues = await prisma.task.findMany({
      where: {
        category: 'WEBSITE_MANAGEMENT',
        OR: [
          { createdAt: { gte: startDate, lte: endDate } },
          { updatedAt: { gte: startDate, lte: endDate } },
          { status: { not: 'DONE' } }
        ]
      },
      include: {
        subTasks: true,
        timeEntries: {
          where: {
            startTime: { gte: startDate, lte: endDate }
          }
        }
      }
    })
    
    return {
      issues: issues.map(issue => ({
        id: issue.id,
        title: issue.title,
        titleJa: issue.titleJa,
        category: this.categorizeWebsiteIssue(issue),
        status: this.mapTaskStatusToIssueStatus(issue.status),
        priority: issue.priority,
        progress: this.calculateTaskProgress(issue),
        assignee: issue.assigneeId,
        createdDate: issue.createdAt,
        updatedDate: issue.updatedAt,
        linkedTasks: issue.subTasks.map(task => ({
          id: task.id,
          title: task.title,
          status: task.status,
          progress: this.calculateTaskProgress(task)
        }))
      })),
      nextActions: '', // To be filled by admin
      nextActionsJa: ''
    }
  }

  async collectEventData(
    category: EventCategory, 
    subcategory: EventSubcategory,
    startDate: Date,
    endDate: Date
  ) {
    const upcomingEvents = await prisma.event.findMany({
      where: {
        category,
        subcategory,
        eventDate: {
          gte: endDate,
          lte: addWeeks(endDate, 2) // Next 2 weeks
        },
        status: { in: ['PLANNING', 'SCHEDULED'] }
      },
      include: {
        tasks: {
          where: { status: { not: 'CANCELLED' } }
        }
      }
    })
    
    const recentlyCompleted = await prisma.event.findMany({
      where: {
        category,
        subcategory,
        status: 'COMPLETED',
        eventDate: {
          gte: startDate,
          lte: endDate
        }
      }
    })
    
    const allTasks = await prisma.task.findMany({
      where: {
        event: {
          category,
          subcategory
        },
        OR: [
          { updatedAt: { gte: startDate, lte: endDate } },
          { status: { not: 'DONE' } }
        ]
      }
    })
    
    return {
      upcomingEvents: upcomingEvents.map(event => ({
        id: event.id,
        title: event.title,
        titleJa: event.titleJa,
        date: event.eventDate,
        location: event.location,
        expectedAttendees: event.plannedAttendees,
        status: event.status,
        tasks: event.tasks.map(task => ({
          id: task.id,
          title: task.title,
          status: task.status,
          assignee: task.assigneeId,
          dueDate: task.dueDate
        }))
      })),
      recentlyCompleted: recentlyCompleted.map(event => ({
        id: event.id,
        title: event.title,
        titleJa: event.titleJa,
        date: event.eventDate,
        actualAttendees: event.actualAttendees,
        reportUrl: event.reportBlogUrl
      })),
      tasksProgress: this.calculateTasksProgress(allTasks)
    }
  }
}
```

## 6. PDF Generation

### 6.1 PDF Generator Service
```typescript
class PDFGenerationService {
  async generatePDF(htmlContent: string, reportId: string): string {
    const browser = await puppeteer.launch({
      headless: true,
      args: ['--no-sandbox', '--disable-setuid-sandbox']
    })
    
    try {
      const page = await browser.newPage()
      
      // Set content with proper encoding for Japanese
      await page.setContent(htmlContent, {
        waitUntil: 'networkidle0'
      })
      
      // Generate PDF with Japanese font support
      const pdfBuffer = await page.pdf({
        format: 'A4',
        margin: {
          top: '20mm',
          right: '20mm',
          bottom: '20mm',
          left: '20mm'
        },
        printBackground: true,
        displayHeaderFooter: true,
        headerTemplate: this.getHeaderTemplate(),
        footerTemplate: this.getFooterTemplate()
      })
      
      // Upload to storage
      const pdfUrl = await this.uploadPDF(pdfBuffer, reportId)
      
      return pdfUrl
    } finally {
      await browser.close()
    }
  }

  private getHeaderTemplate(): string {
    return `
      <div style="font-size: 10px; text-align: center; width: 100%;">
        <span class="date"></span>
      </div>
    `
  }

  private getFooterTemplate(): string {
    return `
      <div style="font-size: 10px; text-align: center; width: 100%;">
        <span class="pageNumber"></span> / <span class="totalPages"></span>
      </div>
    `
  }
}
```

## 7. Report Distribution

### 7.1 Email Distribution Service
```typescript
class ReportDistributionService {
  async distributeReport(reportId: string) {
    const report = await this.getReportWithDistributionList(reportId)
    
    for (const recipient of report.distributions) {
      try {
        await this.sendReportEmail(report, recipient)
        await this.markAsSent(recipient.id)
      } catch (error) {
        await this.logDistributionError(recipient.id, error)
      }
    }
  }

  private async sendReportEmail(report: Report, recipient: ReportDistribution) {
    const emailData = {
      to: recipient.recipientEmail,
      subject: this.getEmailSubject(report, recipient),
      html: this.getEmailBody(report, recipient),
      attachments: [{
        filename: `weekly-report-${report.reportDate}.pdf`,
        path: report.pdfUrl
      }]
    }
    
    await this.emailService.send(emailData)
  }

  private getEmailSubject(report: Report, recipient: ReportDistribution): string {
    const date = format(report.reportDate, 'yyyy-MM-dd')
    
    if (recipient.preferredLanguage === 'ja') {
      return `週次活動レポート - ${date}`
    }
    return `Weekly Activity Report - ${date}`
  }
}
```

## 8. Admin Interface

### 8.1 Report Management Page (`/admin/reports`)
```typescript
interface ReportManagementProps {
  reports: Report[]
  templates: ReportTemplate[]
}

// Features:
// - List all generated reports
// - Filter by date range, status
// - Preview HTML/PDF
// - Edit free text sections
// - Manage distribution list
// - Regenerate reports
// - Download reports
```

### 8.2 Template Editor (`/admin/reports/templates`)
```typescript
interface TemplateEditorProps {
  template: ReportTemplate
  previewData: WeeklyReportData
}

// Features:
// - Visual template editor
// - Handlebars syntax highlighting
// - Live preview with sample data
// - CSS editor
// - Section configuration
// - Save/publish templates
```

### 8.3 Free Text Editor Component
```typescript
interface FreeTextEditorProps {
  reportId: string
  sectionType: 'supporting_grand' | 'others' | 'next_actions'
  content: string
  contentJa: string
  onSave: (content: string, contentJa: string) => void
}

// Features:
// - Rich text editor
// - Bilingual input
// - Auto-save
// - Version history
// - Preview mode
```

## 9. Automation and Scheduling

### 9.1 Cron Job Configuration
```typescript
// vercel.json
{
  "crons": [{
    "path": "/api/cron/weekly-report",
    "schedule": "0 9 * * 1" // Every Monday at 9 AM JST
  }]
}

// API Route
export async function GET(request: Request) {
  // Verify cron secret
  const authHeader = request.headers.get('authorization')
  if (authHeader !== `Bearer ${process.env.CRON_SECRET}`) {
    return new Response('Unauthorized', { status: 401 })
  }
  
  try {
    const reportDate = new Date()
    const template = await getDefaultTemplate()
    
    const report = await reportService.generateWeeklyReport(
      template.id,
      reportDate
    )
    
    await distributionService.distributeReport(report.id)
    
    return Response.json({ success: true, reportId: report.id })
  } catch (error) {
    await notifyAdminOfError(error)
    return Response.json({ success: false, error }, { status: 500 })
  }
}
```

### 9.2 Manual Report Generation
```typescript
interface ManualReportGenerationProps {
  dateRange: { start: Date; end: Date }
  templateId: string
  sections: string[] // Selected sections
  recipients: string[] // Email addresses
}

class ManualReportService {
  async generateCustomReport(props: ManualReportGenerationProps) {
    // 1. Validate date range
    // 2. Create custom report configuration
    // 3. Generate report with selected sections
    // 4. Send to specified recipients
    // 5. Save as ad-hoc report
  }
}
```

## 10. Performance and Optimization

### 10.1 Data Aggregation Optimization
```typescript
// Use database views for complex aggregations
CREATE VIEW weekly_task_summary AS
SELECT 
  DATE_TRUNC('week', created_at) as week,
  category,
  status,
  COUNT(*) as task_count,
  AVG(EXTRACT(EPOCH FROM (completed_at - created_at))/3600) as avg_completion_hours
FROM tasks
GROUP BY week, category, status;

// Parallel data collection
const sections = await Promise.all([
  this.collectSection1Data(),
  this.collectSection2Data(),
  // ... etc
])
```

### 10.2 Caching Strategy
- Cache generated reports for 30 days
- Cache template renders with data hash
- Invalidate cache on template changes
- Pre-generate common date ranges

## 11. Error Handling and Recovery

### 11.1 Generation Error Handling
```typescript
class ReportErrorHandler {
  async handleGenerationError(reportId: string, error: Error) {
    // 1. Log detailed error
    await this.logError({
      reportId,
      error: error.message,
      stack: error.stack,
      timestamp: new Date()
    })
    
    // 2. Update report status
    await this.updateReportStatus(reportId, 'FAILED')
    
    // 3. Notify administrators
    await this.notifyAdmins({
      subject: 'Weekly Report Generation Failed',
      reportId,
      error: error.message
    })
    
    // 4. Schedule retry if appropriate
    if (this.isRetryableError(error)) {
      await this.scheduleRetry(reportId)
    }
  }
}
```

### 11.2 Partial Report Generation
- Generate available sections even if some fail
- Mark failed sections clearly in report
- Allow manual completion of failed sections
- Provide fallback data for critical sections