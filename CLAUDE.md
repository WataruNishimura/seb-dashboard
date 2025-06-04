# CLAUDE.md - Project Context for Claude

## Project Overview
This is a project management tool designed specifically for IT engineers, focusing on event management and task management capabilities.

## Project Structure
```
seb-dashboard/
├── docs/
│   └── designs/
│       └── requirements.md    # Detailed project requirements
└── CLAUDE.md                  # This file - project context
```

# Git Rule

## Commit
- Commit should includes Co-Author

## Conventional Commit
- Commit message should follows conventional commit
- Commit describtion should be written

# Development process

### New feature

#### if requirements or requests given, following these steps:

MUST create git commit each steps.

1. update [requirements.md](./docs/designs/requirements.md)
2. create specification design doc for this requirements
3. think deeply detailed design and architecture of this specification and update document.
4. create GitHub issue with `gh` for this feature
  - smaller issue is better
  - separate one feature into some task as much as possible
5. when addressing issues, create branch and submit draft PR.
  - connect issue with branch
6. create test case for this issue
  - think deeply that test case should be exhastive, including edge case for this task
7. implementation
8. if test fails, check implementation and find what's the problem 
9. resolve the problem and re-try test
10. when test succeeded, push them.
11. request PR review to @WataruNishimura
12. if PR review is not finieshed, checkout new branch from this task branch and proceed it

### Bug fix
1. identify the issue and root cause
2. create a fix following existing code patterns
3. add tests to prevent regression

## Key Features

### Event Management System
- **Event Categories**: 
  - Self-hosting (events we organize)
  - Supporting (events we sponsor/support)
- **Event Types**: Meetup, Career, Grand
- **Before Event**: Category selection, title input, event plan document URL, guest registration
- **After Event**: Attendee numbers, client reports, blog URL attachment
- **Guest Management**: Preference sharing across events for personalization

### Task Management System
- Task creation, assignment, and tracking
- Priority levels and status tracking
- Task dependencies and time tracking
- Integration with events

## Technical Stack
*To be determined based on requirements*

## Development Guidelines
- Focus on usability for IT engineering teams
- Ensure scalability for 100+ concurrent users
- Implement role-based access control
- Mobile-responsive design

## Coding Rules
### Environment & Tools
- **Node Version**: Node.js 22
- **Package Manager**: pnpm (NOT npm or yarn)
- **Package Versions**: Always use latest stable versions
- **Language**: TypeScript (strict mode)
- NEVER use paths, import alias and baseUrls

### Code Quality
- **Linting**: ESLint with Flat Config
- **Formatting**: Prettier
- **Testing**: Vitest for unit and integration tests
- **Pre-push Hook**: Execute `pnpm lint --fix && pnpm prettier --write .`

## CSS 
- use CSS-modules and tailwind
- NEVER use styled-components and css-in-js

### Naming Conventions
- **Files & Folders**: kebab-case (e.g., `user-profile.tsx`, `auth-service.ts`)
- **Components**: UpperCamelCase (e.g., `UserProfile`, `AuthForm`)
- **Variables & Functions**: lowerCamelCase (e.g., `userName`, `getUserData`)

### Component Guidelines
- **Props Type**: Always name component props type as `Props`
- **Props Scope**: Props type should only be used within the component file
- **Example**:
  ```typescript
  // user-profile.tsx
  type Props = {
    userId: string
    onUpdate: (data: UserData) => void
  }
  
  export const UserProfile = ({ userId, onUpdate }: Props) => {
    // component implementation
  }
  ```

### Testing Guidelines
- **Test Files**: Place test files next to the code they test
  - Component tests: `user-profile.test.tsx`
  - Service tests: `auth-service.test.ts`
  - Integration tests: `__tests__/integration/` folder
- **Test Structure**: Use describe/it blocks with clear descriptions
- **Test Naming**: Use descriptive test names that explain the behavior
- **Example**:
  ```typescript
  // user-profile.test.tsx
  import { describe, it, expect, vi } from 'vitest'
  import { render, screen } from '@testing-library/react'
  import { UserProfile } from './user-profile'
  
  describe('UserProfile', () => {
    it('should display user name', () => {
      render(<UserProfile userId="123" onUpdate={vi.fn()} />)
      expect(screen.getByText('John Doe')).toBeInTheDocument()
    })
  })
  ```

## Important Commands
```bash
# Package management (ALWAYS use pnpm)
pnpm install              # Install dependencies
pnpm add <package>        # Add new package
pnpm add -D <package>     # Add dev dependency
pnpm update               # Update packages to latest

# Development
pnpm dev                  # Start development server
pnpm build                # Build for production
pnpm start                # Start production server

# Code quality
pnpm lint                 # Run ESLint
pnpm lint --fix           # Fix ESLint issues
pnpm prettier             # Check Prettier formatting
pnpm prettier --write .   # Fix formatting issues
pnpm typecheck            # Run TypeScript type checking

# Testing (Vitest)
pnpm test                 # Run all tests
pnpm test:unit            # Run unit tests only
pnpm test:integration     # Run integration tests only
pnpm test:watch           # Run tests in watch mode
pnpm test:coverage        # Run tests with coverage report
pnpm test:ui              # Open Vitest UI

# Database
pnpm prisma generate      # Generate Prisma client
pnpm prisma migrate dev   # Run migrations in development
pnpm prisma studio        # Open Prisma Studio
```

## Key Files and Locations
- Requirements: `/docs/designs/requirements.md`
- *Additional locations will be added as the project develops*

## Current Status
- Requirements documentation completed
- Ready for technical design and implementation planning