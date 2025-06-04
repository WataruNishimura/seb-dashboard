# API Specification Document

## 1. Overview

### 1.1 API Architecture
- **Framework**: Next.js 14+ App Router with Route Handlers
- **Protocol**: REST over HTTPS
- **Authentication**: Auth0 with JWT tokens
- **Rate Limiting**: 100 requests per minute per user
- **Versioning**: URL path versioning (e.g., `/api/v1/`)
- **Response Format**: JSON with consistent structure
- **Localization**: Accept-Language header support for ja/en

### 1.2 Base URL Structure
```
Production: https://[your-domain].vercel.app/api/v1
Development: http://localhost:3000/api/v1
```

## 2. Authentication & Authorization

### 2.1 Authentication Flow
```typescript
// Auth Middleware
interface AuthenticatedRequest extends NextRequest {
  user: {
    id: string
    email: string
    roles: string[]
    locale: 'ja' | 'en'
  }
}

// JWT Token Structure
interface JWTPayload {
  sub: string // User ID
  email: string
  roles: string[]
  locale: string
  iat: number
  exp: number
  iss: string
}
```

### 2.2 Authorization Headers
```http
Authorization: Bearer <JWT_TOKEN>
Accept-Language: ja-JP
Content-Type: application/json
X-Request-ID: <UUID> // Optional for request tracking
```

### 2.3 Role-Based Access Control
```typescript
enum UserRole {
  ADMIN = 'admin',
  EVENT_MANAGER = 'event_manager',
  TEAM_LEAD = 'team_lead',
  ENGINEER = 'engineer'
}

// Endpoint permissions
const permissions = {
  '/api/v1/events': {
    GET: ['all'],
    POST: ['admin', 'event_manager'],
    PUT: ['admin', 'event_manager'],
    DELETE: ['admin']
  },
  '/api/v1/tasks': {
    GET: ['all'],
    POST: ['all'],
    PUT: ['owner', 'assignee', 'admin', 'team_lead'],
    DELETE: ['owner', 'admin']
  },
  '/api/v1/reports': {
    GET: ['admin', 'event_manager', 'team_lead'],
    POST: ['admin'],
    PUT: ['admin'],
    DELETE: ['admin']
  }
}
```

## 3. Response Format

### 3.1 Success Response
```typescript
interface SuccessResponse<T> {
  success: true
  data: T
  meta?: {
    pagination?: {
      page: number
      perPage: number
      total: number
      totalPages: number
    }
    timestamp: string
    requestId: string
  }
}

// Example: Single Resource
{
  "success": true,
  "data": {
    "id": "clg1234567890",
    "title": "Tech Meetup #45",
    "titleJa": "テクミートアップ #45",
    "category": "SELF_HOSTING",
    "subcategory": "MEETUP",
    "eventDate": "2024-04-15T18:00:00.000Z"
  },
  "meta": {
    "timestamp": "2024-03-20T10:30:00.000Z",
    "requestId": "550e8400-e29b-41d4-a716-446655440000"
  }
}

// Example: Collection
{
  "success": true,
  "data": [...],
  "meta": {
    "pagination": {
      "page": 1,
      "perPage": 20,
      "total": 150,
      "totalPages": 8
    },
    "timestamp": "2024-03-20T10:30:00.000Z",
    "requestId": "550e8400-e29b-41d4-a716-446655440000"
  }
}
```

### 3.2 Error Response
```typescript
interface ErrorResponse {
  success: false
  error: {
    code: string
    message: string
    messageJa?: string
    details?: any
    field?: string // For validation errors
  }
  meta: {
    timestamp: string
    requestId: string
  }
}

// Example: Validation Error
{
  "success": false,
  "error": {
    "code": "VALIDATION_ERROR",
    "message": "Event date cannot be in the past",
    "messageJa": "イベント日は過去の日付にできません",
    "field": "eventDate"
  },
  "meta": {
    "timestamp": "2024-03-20T10:30:00.000Z",
    "requestId": "550e8400-e29b-41d4-a716-446655440000"
  }
}
```

### 3.3 Standard Error Codes
```typescript
enum ErrorCode {
  // Client Errors (4xx)
  VALIDATION_ERROR = 'VALIDATION_ERROR',
  UNAUTHORIZED = 'UNAUTHORIZED',
  FORBIDDEN = 'FORBIDDEN',
  NOT_FOUND = 'NOT_FOUND',
  CONFLICT = 'CONFLICT',
  RATE_LIMIT_EXCEEDED = 'RATE_LIMIT_EXCEEDED',
  
  // Server Errors (5xx)
  INTERNAL_ERROR = 'INTERNAL_ERROR',
  SERVICE_UNAVAILABLE = 'SERVICE_UNAVAILABLE',
  DATABASE_ERROR = 'DATABASE_ERROR',
  EXTERNAL_SERVICE_ERROR = 'EXTERNAL_SERVICE_ERROR'
}
```

## 4. Event Management APIs

### 4.1 Event Endpoints

#### Create Event
```http
POST /api/v1/events
```

**Request Body:**
```json
{
  "title": "Spring Tech Conference",
  "titleJa": "春のテックカンファレンス",
  "category": "SELF_HOSTING",
  "subcategory": "GRAND",
  "eventDate": "2024-05-20T09:00:00.000Z",
  "eventEndDate": "2024-05-20T18:00:00.000Z",
  "location": "Tokyo Convention Center",
  "locationJa": "東京コンベンションセンター",
  "isVirtual": false,
  "plannedAttendees": 500,
  "planDocumentUrl": "https://docs.google.com/document/d/..."
}
```

**Response:** `201 Created`
```json
{
  "success": true,
  "data": {
    "id": "clg1234567890",
    "title": "Spring Tech Conference",
    "titleJa": "春のテックカンファレンス",
    "category": "SELF_HOSTING",
    "subcategory": "GRAND",
    "status": "PLANNING",
    "eventDate": "2024-05-20T09:00:00.000Z",
    "eventEndDate": "2024-05-20T18:00:00.000Z",
    "location": "Tokyo Convention Center",
    "locationJa": "東京コンベンションセンター",
    "isVirtual": false,
    "plannedAttendees": 500,
    "planDocumentUrl": "https://docs.google.com/document/d/...",
    "createdBy": "auth0|1234567890",
    "createdAt": "2024-03-20T10:30:00.000Z",
    "updatedAt": "2024-03-20T10:30:00.000Z"
  }
}
```

#### List Events
```http
GET /api/v1/events?category=SELF_HOSTING&status=SCHEDULED&page=1&perPage=20
```

**Query Parameters:**
- `category`: EventCategory (optional)
- `subcategory`: EventSubcategory (optional)
- `status`: EventStatus (optional)
- `dateFrom`: ISO 8601 date (optional)
- `dateTo`: ISO 8601 date (optional)
- `search`: Search in title (optional)
- `page`: Page number (default: 1)
- `perPage`: Items per page (default: 20, max: 100)
- `sort`: Sort field (default: eventDate)
- `order`: asc|desc (default: asc)

**Response:** `200 OK`
```json
{
  "success": true,
  "data": [
    {
      "id": "clg1234567890",
      "title": "Tech Meetup #45",
      "titleJa": "テクミートアップ #45",
      "category": "SELF_HOSTING",
      "subcategory": "MEETUP",
      "status": "SCHEDULED",
      "eventDate": "2024-04-15T18:00:00.000Z",
      "location": "Tech Hub Tokyo",
      "plannedAttendees": 50,
      "tasks": {
        "total": 12,
        "completed": 8,
        "inProgress": 3,
        "pending": 1
      }
    }
  ],
  "meta": {
    "pagination": {
      "page": 1,
      "perPage": 20,
      "total": 45,
      "totalPages": 3
    }
  }
}
```

#### Update Event
```http
PUT /api/v1/events/{eventId}
```

**Request Body:** (Partial update supported)
```json
{
  "status": "SCHEDULED",
  "plannedAttendees": 75
}
```

#### Complete Event
```http
POST /api/v1/events/{eventId}/complete
```

**Request Body:**
```json
{
  "actualAttendees": 68,
  "reportBlogUrl": "https://company.blog/events/tech-meetup-45",
  "clientReport": {
    "title": "Tech Meetup #45 Report",
    "titleJa": "テクミートアップ #45 レポート",
    "content": "The event was successful...",
    "contentJa": "イベントは成功しました..."
  }
}
```

#### Guest Registration
```http
POST /api/v1/events/{eventId}/guests
```

**Request Body:**
```json
{
  "email": "john@example.com",
  "name": "John Doe",
  "company": "Tech Corp",
  "dietaryRestrictions": "Vegetarian",
  "interests": ["AI", "Web Development"],
  "contactPreference": "EMAIL"
}
```

## 5. Task Management APIs

### 5.1 Task Endpoints

#### Create Task
```http
POST /api/v1/tasks
```

**Request Body:**
```json
{
  "title": "Prepare venue for meetup",
  "titleJa": "ミートアップの会場準備",
  "description": "Set up chairs, projector, and refreshments",
  "priority": "HIGH",
  "category": "EVENT_EXECUTION",
  "dueDate": "2024-04-14T15:00:00.000Z",
  "assigneeId": "auth0|9876543210",
  "eventId": "clg1234567890",
  "estimatedHours": 3
}
```

#### List Tasks
```http
GET /api/v1/tasks?status=TODO,IN_PROGRESS&assignee=me&sort=dueDate
```

**Query Parameters:**
- `status`: Comma-separated TaskStatus values
- `priority`: Comma-separated TaskPriority values
- `category`: TaskCategory
- `assignee`: User ID or "me" for current user
- `eventId`: Filter by event
- `dueFrom`: Tasks due after this date
- `dueTo`: Tasks due before this date
- `overdue`: true|false
- `search`: Search in title/description

#### Update Task Status
```http
PATCH /api/v1/tasks/{taskId}/status
```

**Request Body:**
```json
{
  "status": "IN_PROGRESS"
}
```

#### Bulk Update Tasks
```http
POST /api/v1/tasks/bulk
```

**Request Body:**
```json
{
  "operation": "update",
  "taskIds": ["task1", "task2", "task3"],
  "updates": {
    "priority": "URGENT",
    "assigneeId": "auth0|1234567890"
  }
}
```

#### Task Dependencies
```http
POST /api/v1/tasks/{taskId}/dependencies
```

**Request Body:**
```json
{
  "dependsOnTaskId": "clg0987654321",
  "type": "FINISH_TO_START"
}
```

#### Time Tracking
```http
POST /api/v1/tasks/{taskId}/time/start
```

**Response:**
```json
{
  "success": true,
  "data": {
    "id": "time123",
    "taskId": "task123",
    "startTime": "2024-03-20T10:30:00.000Z",
    "userId": "auth0|1234567890"
  }
}
```

```http
POST /api/v1/tasks/{taskId}/time/stop
```

**Request Body:**
```json
{
  "timeEntryId": "time123",
  "description": "Completed venue setup"
}
```

## 6. Report APIs

### 6.1 Report Endpoints

#### Generate Weekly Report
```http
POST /api/v1/reports/generate
```

**Request Body:**
```json
{
  "templateId": "default",
  "reportDate": "2024-03-18",
  "sections": ["website_management", "self_hosting_meetup", "coordinator_tasks"]
}
```

#### Get Report
```http
GET /api/v1/reports/{reportId}
```

**Response:**
```json
{
  "success": true,
  "data": {
    "id": "report123",
    "templateId": "default",
    "reportDate": "2024-03-18T00:00:00.000Z",
    "status": "GENERATED",
    "generatedAt": "2024-03-18T09:00:00.000Z",
    "pdfUrl": "https://storage.../reports/2024-03-18.pdf",
    "sections": {
      "websiteManagement": {
        "issues": [...],
        "nextActions": "Focus on SEO improvements"
      },
      "selfHostingMeetup": {
        "upcomingEvents": [...],
        "recentlyCompleted": [...],
        "tasksProgress": {
          "total": 25,
          "completed": 18,
          "inProgress": 5,
          "blocked": 2
        }
      }
    }
  }
}
```

#### Update Report Section
```http
PATCH /api/v1/reports/{reportId}/sections/{sectionId}
```

**Request Body:**
```json
{
  "freeText": "Updated grand event information...",
  "freeTextJa": "更新されたグランドイベント情報..."
}
```

#### Export Report
```http
GET /api/v1/reports/{reportId}/export?format=pdf
```

**Query Parameters:**
- `format`: pdf|html|email

#### Send Report
```http
POST /api/v1/reports/{reportId}/send
```

**Request Body:**
```json
{
  "recipients": [
    {
      "email": "client@example.com",
      "name": "Client Name",
      "type": "CLIENT"
    }
  ],
  "subject": "Weekly Activity Report - March 18, 2024",
  "message": "Please find attached our weekly report."
}
```

## 7. Integration APIs

### 7.1 Slack Integration

#### Connect Slack Workspace
```http
GET /api/v1/integrations/slack/connect
```

**Response:** Redirect to Slack OAuth

#### Slack Webhook
```http
POST /api/v1/webhooks/slack
```

**Headers:**
```
X-Slack-Request-Timestamp: 1234567890
X-Slack-Signature: v0=...
```

#### Configure Slack Notifications
```http
PUT /api/v1/integrations/slack/notifications
```

**Request Body:**
```json
{
  "channelId": "C1234567890",
  "notifications": {
    "TASK_ASSIGNED": true,
    "TASK_OVERDUE": true,
    "EVENT_REMINDER": true,
    "REPORT_GENERATED": false
  }
}
```

### 7.2 Google Forms Integration

#### Connect Google Form
```http
POST /api/v1/integrations/google-forms
```

**Request Body:**
```json
{
  "formId": "1FAIpQLSd...",
  "eventId": "clg1234567890"
}
```

#### Sync Form Responses
```http
POST /api/v1/integrations/google-forms/{integrationId}/sync
```

#### Google Forms Webhook
```http
POST /api/v1/webhooks/google-forms
```

**Headers:**
```
Authorization: Bearer {WEBHOOK_TOKEN}
```

## 8. User & Admin APIs

### 8.1 User Profile

#### Get Current User
```http
GET /api/v1/users/me
```

**Response:**
```json
{
  "success": true,
  "data": {
    "id": "auth0|1234567890",
    "email": "user@example.com",
    "name": "User Name",
    "roles": ["event_manager", "engineer"],
    "locale": "ja",
    "preferences": {
      "notifications": {
        "email": true,
        "slack": true
      },
      "timezone": "Asia/Tokyo",
      "language": "ja"
    }
  }
}
```

#### Update User Preferences
```http
PATCH /api/v1/users/me/preferences
```

**Request Body:**
```json
{
  "notifications": {
    "email": false,
    "slack": true
  },
  "language": "en"
}
```

### 8.2 Admin Endpoints

#### List Users
```http
GET /api/v1/admin/users
```

**Required Role:** `admin`

#### Update User Roles
```http
PUT /api/v1/admin/users/{userId}/roles
```

**Request Body:**
```json
{
  "roles": ["event_manager", "team_lead"]
}
```

#### System Health
```http
GET /api/v1/admin/health
```

**Response:**
```json
{
  "success": true,
  "data": {
    "status": "healthy",
    "services": {
      "database": "healthy",
      "slack": "healthy",
      "email": "healthy",
      "googleForms": "degraded"
    },
    "metrics": {
      "activeUsers": 42,
      "eventsThisWeek": 8,
      "tasksCompleted": 156,
      "reportsPending": 2
    }
  }
}
```

## 9. Webhook Management

### 9.1 Webhook Registration
```http
POST /api/v1/webhooks
```

**Request Body:**
```json
{
  "url": "https://example.com/webhook",
  "events": ["task.completed", "event.created", "report.generated"],
  "secret": "webhook_secret_123"
}
```

### 9.2 Webhook Payload Format
```json
{
  "id": "webhook_event_123",
  "type": "task.completed",
  "timestamp": "2024-03-20T10:30:00.000Z",
  "data": {
    "task": {
      "id": "task123",
      "title": "Prepare venue",
      "completedAt": "2024-03-20T10:30:00.000Z",
      "completedBy": "auth0|1234567890"
    }
  },
  "signature": "sha256=..."
}
```

## 10. Rate Limiting & Quotas

### 10.1 Rate Limits
```http
X-RateLimit-Limit: 100
X-RateLimit-Remaining: 95
X-RateLimit-Reset: 1234567890
```

### 10.2 Rate Limit Response
```json
{
  "success": false,
  "error": {
    "code": "RATE_LIMIT_EXCEEDED",
    "message": "Too many requests. Please try again in 60 seconds.",
    "messageJa": "リクエストが多すぎます。60秒後に再試行してください。",
    "retryAfter": 60
  }
}
```

## 11. API Versioning

### 11.1 Version Header
```http
X-API-Version: 1.0
```

### 11.2 Deprecation Notice
```http
X-API-Deprecated: true
X-API-Sunset: 2024-12-31
```

## 12. Search & Filtering

### 12.1 Advanced Search
```http
POST /api/v1/search
```

**Request Body:**
```json
{
  "query": "spring conference",
  "types": ["events", "tasks", "guests"],
  "filters": {
    "dateRange": {
      "from": "2024-03-01",
      "to": "2024-05-31"
    },
    "categories": ["SELF_HOSTING"],
    "status": ["SCHEDULED", "IN_PROGRESS"]
  },
  "limit": 10
}
```

### 12.2 Aggregations
```http
GET /api/v1/analytics/overview?period=month
```

**Response:**
```json
{
  "success": true,
  "data": {
    "period": "2024-03",
    "events": {
      "total": 15,
      "byCategory": {
        "SELF_HOSTING": 10,
        "SUPPORTING": 5
      },
      "byStatus": {
        "COMPLETED": 8,
        "SCHEDULED": 5,
        "PLANNING": 2
      }
    },
    "tasks": {
      "total": 234,
      "completed": 189,
      "overdue": 12,
      "avgCompletionTime": 3.5
    }
  }
}
```

## 13. File Upload

### 13.1 Upload Attachment
```http
POST /api/v1/uploads
Content-Type: multipart/form-data
```

**Form Data:**
- `file`: Binary file data
- `type`: attachment|report|document
- `entityType`: task|event
- `entityId`: Related entity ID

**Response:**
```json
{
  "success": true,
  "data": {
    "id": "upload123",
    "fileName": "venue-layout.pdf",
    "fileUrl": "https://storage.../uploads/venue-layout.pdf",
    "fileSize": 1048576,
    "mimeType": "application/pdf",
    "uploadedAt": "2024-03-20T10:30:00.000Z"
  }
}
```

## 14. WebSocket Events (Future)

### 14.1 Connection
```javascript
const ws = new WebSocket('wss://api.example.com/ws');
ws.send(JSON.stringify({
  type: 'auth',
  token: 'Bearer ...'
}));
```

### 14.2 Event Types
```typescript
interface WebSocketEvent {
  type: 'task.updated' | 'event.updated' | 'notification'
  data: any
  timestamp: string
}
```

## 15. Development & Testing

### 15.1 Sandbox Environment
```
Base URL: https://sandbox.api.example.com/api/v1
```

### 15.2 Test Credentials
```json
{
  "testUsers": {
    "admin": "test-admin@example.com",
    "eventManager": "test-event@example.com",
    "engineer": "test-engineer@example.com"
  }
}
```

### 15.3 API Documentation
- Swagger/OpenAPI: `/api/docs`
- Postman Collection: Available on request
- GraphQL Playground: `/api/graphql` (Future)