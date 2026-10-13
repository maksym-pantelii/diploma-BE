# Coachly — Backend

REST API for **Coachly**, a platform that connects clients with personal trainers: trainer search, schedule management, session booking with online payment, reviews, achievements and an admin panel for trainer verification.

Built as a diploma project.

## Tech stack

- **.NET 10 / ASP.NET Core** Web API
- **PostgreSQL** + Entity Framework Core (Npgsql), database-first
- **Redis** distributed cache (with graceful fallback to the database)
- **Stripe** Checkout, webhooks and refunds
- **Azure Blob Storage** for avatars and trainer documents (time-limited SAS links)
- **MailKit** (SMTP) for transactional emails
- **JWT** authentication with rotating refresh tokens, role-based authorization
- **AutoMapper**, **Swagger / OpenAPI**
- **xUnit**, **Moq**, **FluentAssertions** (116 unit tests)

## Features

**Auth & users**
- Registration as a client or trainer, login, password reset via email
- Passwords hashed with BCrypt; refresh tokens stored as SHA-256 hashes and rotated on every refresh
- Profiles with avatars, tags (specializations, methodologies, accessibility) and booking history

**Trainers**
- Search with filters: specialization, city, price range, rating, name, verification status, accessibility
- Schedule slots (online / offline): create, update, delete
- Verification flow: trainers upload documents, admins approve or reject them
- Per-trainer statistics, client list and reviews

**Bookings & payments**
- Booking a slot creates a Stripe Checkout session; the Stripe webhook confirms the booking and payment
- Unpaid bookings expire after a reservation window; retrying payment after the window adds a late fee
- Cancellation with automatic Stripe refund: 100% more than 24 hours before the session, 50% otherwise

**Engagement**
- Reviews and ratings after completed sessions
- Achievements system (sessions completed, trainers tried, specializations, cities, etc.)
- Session notes shared between the client and the trainer of a booking
- Email reminders about upcoming sessions

**Admin**
- Platform statistics, user list
- Support tickets with replies
- Trainer document moderation

**Background jobs** (`BackgroundService`)
- `BookingExpiryBackgroundService` cancels unpaid bookings after the payment window
- `NotificationBackgroundService` sends reminders about upcoming sessions
- `SlotCompletionBackgroundService` completes past sessions, requests reviews and awards achievements

## Architecture

```
CaoachlyBE/
├── Controllers/          HTTP endpoints
├── Services/             Business logic
├── Repositories/         Data access + Unit of Work
├── Entities/             EF Core entities (scaffolded from the database)
├── Models/               DTOs and domain models
├── BackgroundServices/   Scheduled background jobs
├── Helpers/              JWT, mapping profiles, time zone helpers
└── Migrations/           SQL schema and migration scripts
CaoachlyBE.Tests/         Unit tests
```

Controllers → Services → Repositories, with dependency injection throughout. External integrations (Stripe, Blob Storage, email, time) are behind interfaces, so the business logic is unit-tested with mocks.

## Getting started

### Prerequisites

- [.NET SDK 10](https://dotnet.microsoft.com/download)
- PostgreSQL 14+
- Redis (optional: tag lookups fall back to the database without it)
- Stripe account in test mode, Azure Storage account or [Azurite](https://learn.microsoft.com/azure/storage/common/storage-use-azurite), SMTP credentials

### 1. Create the database

```bash
createdb CoachlyAppDb
psql -d CoachlyAppDb -f CaoachlyBE/Migrations/schema.sql
psql -d CoachlyAppDb -f CaoachlyBE/Migrations/2026-05-21_seed_achievements.sql
```

### 2. Configure the app

```bash
cp CaoachlyBE/appsettings.Example.json CaoachlyBE/appsettings.json
```

Fill in the placeholders in `appsettings.json`. This file is git-ignored, so secrets never get committed. `Jwt:SecretKey` must be at least 32 characters long.

To receive Stripe webhooks locally:

```bash
stripe listen --forward-to http://localhost:5064/api/webhooks/stripe
```

### 3. Run

```bash
dotnet run --project CaoachlyBE --launch-profile http
```

Swagger UI is available at `http://localhost:5064/swagger` in the Development environment.

### 4. Tests

```bash
dotnet test
```

## API overview

| Area | Base route |
|---|---|
| Auth | `/api/auth` |
| Users | `/api/users` |
| Clients | `/api/clients` |
| Trainers, slots, documents | `/api/trainers`, `/api/slots` |
| Bookings | `/api/bookings` |
| Reviews | `/api/reviews` |
| Session notes | `/api/session-notes` |
| Achievements | `/api/achievements` |
| Tags | `/api/tags` |
| Support tickets | `/api/tickets` |
| Platform stats | `/api/stats` |
| Admin | `/api/admin` |
| Stripe webhook | `/api/webhooks/stripe` |

Full request and response schemas are in Swagger.
