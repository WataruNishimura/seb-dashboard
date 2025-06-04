# SEB Dashboard

A comprehensive project management tool designed for IT engineers, featuring event management, task tracking, and automated weekly reporting.

## Features

- **Event Management**: Organize and track self-hosting and supporting events
- **Task Management**: Advanced task tracking with dependencies and time tracking
- **Weekly Reports**: Automated report generation with customizable templates
- **Multi-language Support**: Japanese and English interface
- **Authentication**: Email/Password and SSO (Google, Microsoft)
- **Integrations**: Slack, Email, and Google Forms

## Tech Stack

- **Frontend**: Next.js 14+ with App Router
- **Backend**: Next.js API Routes
- **Database**: PostgreSQL with Prisma ORM
- **Authentication**: Auth0
- **Hosting**: Vercel
- **Testing**: Vitest
- **Package Manager**: pnpm

## Prerequisites

- Node.js 22+
- pnpm
- PostgreSQL
- Auth0 account

## Getting Started

1. Clone the repository:
```bash
git clone git@github.com:WataruNishimura/seb-dashboard.git
cd seb-dashboard
```

2. Install dependencies:
```bash
pnpm install
```

3. Set up environment variables:
```bash
cp .env.example .env.local
```

4. Run database migrations:
```bash
pnpm prisma migrate dev
```

5. Start the development server:
```bash
pnpm dev
```

## Development

### Commands

```bash
# Development
pnpm dev                  # Start development server
pnpm build                # Build for production
pnpm start                # Start production server

# Code Quality
pnpm lint                 # Run ESLint
pnpm lint --fix           # Fix ESLint issues
pnpm prettier             # Check formatting
pnpm prettier --write .   # Fix formatting
pnpm typecheck            # TypeScript type checking

# Testing
pnpm test                 # Run all tests
pnpm test:unit            # Run unit tests
pnpm test:integration     # Run integration tests
pnpm test:coverage        # Generate coverage report

# Database
pnpm prisma generate      # Generate Prisma client
pnpm prisma migrate dev   # Run migrations
pnpm prisma studio        # Open Prisma Studio
```

### Project Structure

```
seb-dashboard/
├── docs/
│   └── designs/          # Design specifications
├── src/
│   ├── app/              # Next.js app directory
│   ├── components/       # React components
│   ├── lib/              # Utility functions
│   ├── services/         # Business logic
│   └── types/            # TypeScript types
├── prisma/
│   └── schema.prisma     # Database schema
├── public/               # Static assets
└── tests/                # Test files
```

## Contributing

Please read our contributing guidelines before submitting PRs.

## License

[MIT License](LICENSE)