# Codecon

A full-stack Twitter/X clone built with a modern, containerised architecture.

## Tech Stack

| Layer       | Technology                               |
|-------------|------------------------------------------|
| Frontend    | Next.js 14, React 18, Tailwind CSS       |
| Backend     | NestJS 10, Prisma 5, TypeScript          |
| Database    | PostgreSQL 16                            |
| Queue       | Redis 7 + BullMQ                         |
| Reverse proxy | Nginx 1.27                             |
| Container   | Docker + Docker Compose                  |

## Architecture Overview

```
Browser
   │
   ▼
┌─────────────────────────────────────────┐
│              Nginx  :80                 │  ← single public entry point
│                                         │
│  /api/*  ──────────────► backend:3000   │  (NestJS REST API)
│  /_next/static/  ──────► frontend:3001  │  (cached static assets)
│  /*  ──────────────────► frontend:3001  │  (SSR + Server Actions)
│  /uploads/*  ──────────► backend:3000   │  (user-uploaded files)
└─────────────────────────────────────────┘
         │                    │
         ▼                    ▼
   ┌──────────┐        ┌──────────────┐
   │ NestJS   │        │  Next.js 14  │
   │ backend  │        │  frontend    │
   └────┬─────┘        └──────────────┘
        │  reads/writes
        ▼
   ┌──────────┐    ┌──────────┐
   │ Postgres │    │  Redis   │
   │    DB    │    │  Queue   │
   └──────────┘    └──────────┘
```

**Key design decisions:**
- Nginx strips the `/api` prefix before forwarding to the backend, so the NestJS app receives clean routes (e.g. `/api/auth/login` → `/auth/login`).
- Next.js Server Components and Server Actions call the backend **directly** via the internal Docker network (`http://backend:3000`), bypassing Nginx for server-to-server traffic.
- The backend is not exposed to the host machine; only Nginx's port 80 is published.

## Features

- User registration and JWT authentication
- Create posts with optional image upload
- Threaded replies (comments)
- Likes, reposts, and bookmarks
- Follow / unfollow users
- Real-time-style notifications (like, repost, comment, follow)
- Direct messages between users, with optional post sharing

## Repository Structure

```
codecon/
├── backend/        # NestJS REST API + Prisma ORM
├── frontend/       # Next.js 14 app (App Router)
├── nginx/          # Nginx reverse-proxy config
└── docker-compose.yml
```

## Running the Project

### Prerequisites

- [Docker](https://docs.docker.com/get-docker/) and Docker Compose v2

### 1. Configure environment variables

Create a `.env` file in the project root:

```env
DATABASE_URL=postgresql://codecon:codecon@postgres:5432/codecon
JWT_SECRET=your_super_secret_key_here
NGINX_PORT=80          # optional, defaults to 80
```

### 2. Start all services

```bash
docker compose up --build
```

Docker Compose will:
1. Start PostgreSQL and wait until it is healthy.
2. Start Redis and wait until it is healthy.
3. Build and start the NestJS backend; on first run, `prisma db push` applies the schema automatically.
4. Build and start the Next.js frontend.
5. Start Nginx as the single public-facing entry point.

### 3. Open the app

Navigate to [http://localhost](http://localhost) (or the port configured via `NGINX_PORT`).

### Useful commands

```bash
# Run in background
docker compose up -d --build

# View logs
docker compose logs -f backend
docker compose logs -f frontend

# Tear down (keep volumes)
docker compose down

# Tear down and delete all data
docker compose down -v
```
