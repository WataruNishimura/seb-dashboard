# Event Management System Design Specification

## 1. Overview

### 1.1 Purpose
The Event Management System enables IT engineering teams to organize, track, and report on technical events, including self-hosted and supported events across three categories: meetups, career events, and grand events.

### 1.2 Tech Stack
- **Frontend**: Next.js 14+ with App Router
- **Backend**: Next.js API Routes
- **Database**: PostgreSQL with Prisma ORM
- **Authentication**: Auth0
- **Hosting**: Vercel
- **Localization**: i18next for Japanese/English support

## 2. Data Models

### 2.1 Event Model
```typescript
model Event {
  id                String   @id @default(cuid())
  title             String
  titleJa           String?  // Japanese title
  category          EventCategory
  subcategory       EventSubcategory
  status            EventStatus @default(PLANNING)
  eventDate         DateTime
  eventEndDate      DateTime?
  location          String?
  locationJa        String?
  isVirtual         Boolean  @default(false)
  virtualUrl        String?
  planDocumentUrl   String?
  reportBlogUrl     String?
  plannedAttendees  Int?
  actualAttendees   Int?
  budget            Decimal? @db.Decimal(10, 2)
  sponsorshipLevel  String?  // For supporting events
  createdBy         String   // Auth0 user ID
  createdAt         DateTime @default(now())
  updatedAt         DateTime @updatedAt
  
  guests            Guest[]
  tasks             Task[]
  clientReports     ClientReport[]
  eventHistory      EventHistory[]
}

enum EventCategory {
  SELF_HOSTING
  SUPPORTING
}

enum EventSubcategory {
  MEETUP
  CAREER
  GRAND
}

enum EventStatus {
  PLANNING
  SCHEDULED
  IN_PROGRESS
  COMPLETED
  CANCELLED
}
```

### 2.2 Guest Model
```typescript
model Guest {
  id                String   @id @default(cuid())
  email             String   @unique
  name              String
  nameJa            String?
  company           String?
  role              String?
  dietaryRestrictions String?
  interests         String[] // Array of interest tags
  contactPreference ContactPreference @default(EMAIL)
  preferredLanguage Language @default(JA)
  createdAt         DateTime @default(now())
  updatedAt         DateTime @updatedAt
  
  eventRegistrations EventRegistration[]
}

model EventRegistration {
  id          String   @id @default(cuid())
  eventId     String
  guestId     String
  status      RegistrationStatus @default(PENDING)
  registeredAt DateTime @default(now())
  attendedAt  DateTime?
  notes       String?
  
  event       Event    @relation(fields: [eventId], references: [id])
  guest       Guest    @relation(fields: [guestId], references: [id])
  
  @@unique([eventId, guestId])
}

enum RegistrationStatus {
  PENDING
  CONFIRMED
  ATTENDED
  NO_SHOW
  CANCELLED
}

enum ContactPreference {
  EMAIL
  PHONE
  SLACK
}

enum Language {
  JA
  EN
}
```

### 2.3 Client Report Model
```typescript
model ClientReport {
  id          String   @id @default(cuid())
  eventId     String
  title       String
  titleJa     String?
  content     String   @db.Text
  contentJa   String?  @db.Text
  reportType  ReportType
  createdBy   String
  createdAt   DateTime @default(now())
  
  event       Event    @relation(fields: [eventId], references: [id])
}

enum ReportType {
  SUMMARY
  DETAILED
  METRICS
}
```

## 3. API Endpoints

### 3.1 Event Management APIs
```typescript
// Event CRUD Operations
POST   /api/events                    // Create new event
GET    /api/events                    // List events with filters
GET    /api/events/:id                // Get event details
PUT    /api/events/:id                // Update event
DELETE /api/events/:id                // Soft delete event
POST   /api/events/:id/complete       // Mark event as completed

// Event Planning
POST   /api/events/:id/plan-document  // Attach plan document URL
GET    /api/events/:id/plan-document  // Get plan document URL

// Event Reporting
POST   /api/events/:id/report         // Add post-event report
POST   /api/events/:id/blog-url       // Attach blog URL
PUT    /api/events/:id/attendance     // Update attendance numbers
```

### 3.2 Guest Management APIs
```typescript
// Guest CRUD
POST   /api/guests                    // Create guest profile
GET    /api/guests                    // List guests with search
GET    /api/guests/:id                // Get guest details
PUT    /api/guests/:id                // Update guest preferences
GET    /api/guests/:id/history        // Get guest event history

// Event Registration
POST   /api/events/:id/register       // Register guest for event
GET    /api/events/:id/registrations  // List event registrations
PUT    /api/events/:id/registrations/:guestId // Update registration
POST   /api/events/:id/check-in       // Check-in guest at event
```

## 4. UI Components and Pages

### 4.1 Event List Page (`/events`)
```typescript
interface EventListPageProps {
  filters: {
    category?: EventCategory
    subcategory?: EventSubcategory
    status?: EventStatus
    dateRange?: { start: Date; end: Date }
  }
}

// Features:
// - Filterable table/grid view
// - Quick actions (edit, view, duplicate)
// - Bulk operations
// - Export to CSV
// - Calendar view toggle
```

### 4.2 Event Creation/Edit Page (`/events/new`, `/events/:id/edit`)
```typescript
interface EventFormProps {
  mode: 'create' | 'edit'
  eventId?: string
}

// Form sections:
// 1. Basic Information
//    - Category selection (radio)
//    - Subcategory selection (radio)
//    - Title (bilingual fields)
//    - Date/time (with timezone display)
// 2. Location
//    - Physical/Virtual toggle
//    - Location details (bilingual)
//    - Virtual meeting URL
// 3. Planning
//    - Plan document URL
//    - Expected attendees
//    - Budget (for self-hosting)
//    - Sponsorship level (for supporting)
// 4. Tasks (link to task management)
```

### 4.3 Event Detail Page (`/events/:id`)
```typescript
// Tabs:
// 1. Overview - Event details, status, key metrics
// 2. Guests - Registration list, check-in status
// 3. Tasks - Related tasks and progress
// 4. Reports - Client reports and blog posts
// 5. History - Audit log of changes
```

### 4.4 Guest Management Page (`/guests`)
```typescript
// Features:
// - Search by name, email, company
// - Filter by interests, dietary restrictions
// - Guest profile modal
// - Bulk invite functionality
// - Export guest list
// - GDPR-compliant data handling
```

## 5. Business Logic

### 5.1 Event Lifecycle
```typescript
class EventService {
  async createEvent(data: CreateEventDto, userId: string) {
    // 1. Validate category/subcategory combination
    // 2. Generate bilingual slug
    // 3. Set initial status to PLANNING
    // 4. Create audit log entry
    // 5. Send notification to team
  }

  async completeEvent(eventId: string, data: CompleteEventDto) {
    // 1. Validate all required post-event data
    // 2. Update attendance numbers
    // 3. Change status to COMPLETED
    // 4. Trigger report generation reminder
    // 5. Update guest attendance records
  }

  async linkGoogleForm(eventId: string, formUrl: string) {
    // 1. Validate Google Form URL
    // 2. Set up webhook for form responses
    // 3. Import existing responses
    // 4. Map form fields to guest preferences
  }
}
```

### 5.2 Guest Preference Sharing
```typescript
class GuestService {
  async getGuestSuggestions(eventId: string) {
    // 1. Get event details (category, date, location)
    // 2. Find guests who attended similar events
    // 3. Score by attendance history and interests
    // 4. Return ranked suggestions
  }

  async syncPreferences(guestId: string, updates: PreferenceUpdate) {
    // 1. Update guest master record
    // 2. Propagate to future event registrations
    // 3. Maintain preference history
  }
}
```

## 6. Security Considerations

### 6.1 Authorization Rules
```typescript
// Event permissions
const eventPermissions = {
  create: ['admin', 'event_manager'],
  edit: ['admin', 'event_manager', 'owner'],
  delete: ['admin'],
  view: ['admin', 'event_manager', 'team_lead', 'engineer'],
  complete: ['admin', 'event_manager', 'owner']
}

// Guest data permissions
const guestPermissions = {
  view: ['admin', 'event_manager'],
  edit: ['admin'],
  export: ['admin'],
  delete: ['admin'] // GDPR compliance
}
```

### 6.2 Data Protection
- Encrypt guest PII at rest
- Audit log for all guest data access
- Automatic PII redaction in logs
- Secure document URL storage
- Rate limiting on guest search APIs

## 7. Localization

### 7.1 Bilingual Fields
```typescript
interface BilingualField {
  value: string      // Default (Japanese)
  valueEn?: string   // English translation
}

// Applied to:
// - Event titles
// - Event descriptions
// - Location names
// - Report content
// - Error messages
```

### 7.2 Date/Time Handling
```typescript
// All dates stored in UTC
// Display in Asia/Tokyo (GMT+9) by default
// Format: YYYY年MM月DD日 HH:mm for Japanese
// Format: MMM DD, YYYY HH:mm for English
```

## 8. Integration Points

### 8.1 Google Forms Integration
```typescript
interface GoogleFormIntegration {
  formId: string
  eventId: string
  fieldMappings: {
    name: string
    email: string
    company?: string
    dietary?: string
    interests?: string[]
  }
  syncEnabled: boolean
  lastSyncAt?: Date
}
```

### 8.2 Task Management Integration
- Auto-create tasks for event milestones
- Link tasks to specific events
- Track task completion in event progress
- Include task status in event reports

## 9. Performance Requirements

### 9.1 Response Times
- Event list page: < 500ms
- Event creation: < 1s
- Guest search: < 300ms
- Report generation: < 3s

### 9.2 Scalability
- Support 100+ concurrent users
- Handle 1000+ events per year
- Manage 10,000+ guest records
- Optimize queries with proper indexing

## 10. Error Handling

### 10.1 User-Facing Errors
```typescript
interface ErrorResponse {
  code: string
  message: string
  messageJa: string
  details?: any
  timestamp: Date
}

// Examples:
// EVENT_DATE_PAST: "Event date cannot be in the past"
// GUEST_DUPLICATE: "Guest already registered for this event"
// DOCUMENT_URL_INVALID: "Please provide a valid Google Docs URL"
```

### 10.2 System Errors
- Retry failed API calls (3 attempts)
- Queue failed notifications
- Log errors to monitoring service
- Graceful degradation for non-critical features