# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Development Commands

- `npm run dev` - Start development server (opens at http://localhost:3000)
- `npm run build` - Build production version
- `npm run start` - Start production server
- `npm run lint` - Run ESLint to check code quality
- `npx prisma db seed` - Seed database with test data

## Architecture Overview

This is a Next.js 14 school management dashboard application using the App Router with role-based authentication via Clerk.

### Core Technologies
- **Next.js 14** with App Router (`src/app/` structure)
- **Prisma** ORM with PostgreSQL database
- **Clerk** for authentication and user management
- **Tailwind CSS** for styling
- **TypeScript** for type safety
- **React Hook Form + Zod** for form validation
- **Recharts** for data visualization

### Database Architecture

The application uses a comprehensive school management schema with the following main entities:
- **Users**: Admin, Teacher, Student, Parent (with role-based access)
- **Academic**: Grade, Class, Subject, Lesson
- **Assessment**: Exam, Assignment, Result
- **Administrative**: Attendance, Event, Announcement

Key relationships:
- Students belong to Classes and Grades, have Parents
- Teachers supervise Classes and teach Subjects
- Lessons connect Teachers, Subjects, and Classes
- Results track performance on Exams and Assignments

### Project Structure

```
src/
├── app/                    # Next.js 14 App Router
│   ├── (dashboard)/        # Dashboard layout group
│   │   ├── admin/          # Admin role pages
│   │   ├── teacher/        # Teacher role pages
│   │   ├── student/        # Student role pages
│   │   ├── parent/         # Parent role pages
│   │   └── list/           # CRUD pages for entities
│   └── [[...sign-in]]/     # Catch-all auth route
├── components/             # Reusable UI components
│   └── forms/              # Entity-specific forms
└── lib/                    # Utility functions and configurations
    ├── actions.ts          # Server actions for CRUD operations
    ├── data.ts             # Temporary mock data
    ├── formValidationSchemas.ts # Zod schemas
    ├── prisma.ts           # Prisma client
    └── settings.ts         # Route access configuration
```

### Authentication & Authorization

Role-based access is enforced through:
- **Middleware** (`src/middleware.ts`) - Route protection based on user roles
- **Clerk** integration with custom roles (admin, teacher, student, parent)
- **Route access mapping** in `src/lib/settings.ts`

### Key Development Patterns

- **Server Actions**: All CRUD operations use Next.js server actions in `src/lib/actions.ts`
- **Form Handling**: React Hook Form with Zod validation schemas
- **Database Operations**: Prisma with complex relational queries
- **Role-Based UI**: Components render differently based on user role
- **Chart Components**: Recharts with container components for data fetching

### Database Setup

Database migrations are in `prisma/migrations/`. The seed file (`prisma/seed.ts`) populates the database with test data for development.

### Important Files

- `src/middleware.ts` - Handles route protection and role-based redirects
- `src/lib/actions.ts` - Server actions for all CRUD operations
- `src/lib/settings.ts` - Route access configuration
- `prisma/schema.prisma` - Complete database schema
- `src/lib/formValidationSchemas.ts` - Validation schemas for all forms