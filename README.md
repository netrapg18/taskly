# Taskly — Full-Stack Task Management Application

A production-ready task management application built with React, TypeScript, Express, PostgreSQL, and Socket.IO. Supports authentication, role-based access control, real-time collaborative updates, and a responsive UI for desktop and mobile.

## Stack

- **Frontend:** React 18, TypeScript, Vite, Tailwind CSS, TanStack Query, React Router, Socket.IO client
- **Backend:** Node.js, Express, TypeScript, PostgreSQL (`pg`), Socket.IO, JWT auth, express-validator
- **Database:** PostgreSQL 16
- **Infrastructure:** Docker & Docker Compose, Nginx (frontend serving)

## Features

- Email/password registration, login, and JWT-based session management with refresh tokens
- Role-based access control (`admin` / `user`) — admins can manage users and roles
- Full task CRUD with title, description, status, priority, due date, assignee, and tags
- Real-time task creation, updates, and deletion broadcast via Socket.IO
- Kanban board, "My Tasks" list, and analytics dashboard
- Responsive layout for mobile and desktop
- Server-side validation, centralized error handling, rate limiting, and security headers (Helmet)
- Database migrations and seed data with demo accounts

## Project Structure

```
taskly/
├── backend/
│   ├── src/
│   │   ├── config/        # env config & database pool
│   │   ├── controllers/    # route handlers
│   │   ├── db/             # migration & seed scripts
│   │   ├── middleware/     # auth, validation, error handling
│   │   ├── models/         # SQL data access layer
│   │   ├── routes/         # Express routers
│   │   ├── services/       # Socket.IO service
│   │   ├── types/           # shared TS types
│   │   ├── utils/           # jwt, logger, response helpers
│   │   └── index.ts         # app entrypoint
│   ├── Dockerfile
│   ├── package.json
│   └── tsconfig.json
├── frontend/
│   ├── src/
│   │   ├── components/      # UI, layout, auth, task components
│   │   ├── contexts/         # AuthContext
│   │   ├── hooks/             # useTasks, useSocketTasks
│   │   ├── pages/             # route-level pages
│   │   ├── services/          # API + socket clients
│   │   ├── types/              # shared TS types
│   │   └── utils/               # formatting helpers
│   ├── Dockerfile
│   ├── nginx.conf
│   ├── package.json
│   └── vite.config.ts
├── docker-compose.yml
└── .env.example
```

## Getting Started (Local Development)

### Prerequisites

- Node.js 20+
- PostgreSQL 16 (or use Docker for just the database)
- npm

### 1. Clone and configure environment variables

```bash
cp .env.example .env
cp backend/.env.example backend/.env
cp frontend/.env.example frontend/.env
```

Edit the values as needed (especially `JWT_SECRET` and `JWT_REFRESH_SECRET` in production).

### 2. Start PostgreSQL

Using Docker:

```bash
docker compose up -d postgres redis
```

Or point `backend/.env` at your own local PostgreSQL instance.

### 3. Install dependencies and run migrations

```bash
cd backend
npm install
npm run migrate
npm run seed
```

The seed script creates the following demo accounts:

| Role  | Email              | Password    |
|-------|--------------------|-------------|
| Admin | admin@taskly.dev   | Admin1234!  |
| User  | sam@taskly.dev     | User1234!   |
| User  | priya@taskly.dev   | User1234!   |
| User  | jordan@taskly.dev  | User1234!   |

### 4. Start the backend

```bash
npm run dev
```

The API will be available at `http://localhost:4000`, with Socket.IO on the same port.

### 5. Start the frontend

```bash
cd ../frontend
npm install
npm run dev
```

The app will be available at `http://localhost:3000`.

## Running with Docker Compose (Full Stack)

```bash
cp .env.example .env
docker compose up --build
```

This starts PostgreSQL, Redis, the backend API (port 4000), and the frontend (port 3000). After the containers are healthy, run migrations and seed data inside the backend container:

```bash
docker compose exec backend npm run migrate
docker compose exec backend npm run seed
```

Then visit `http://localhost:3000`.

## API Overview

All endpoints are prefixed with `/api`.

### Auth

| Method | Endpoint             | Description                      | Auth |
|--------|----------------------|-----------------------------------|------|
| POST   | `/auth/register`     | Register a new user               | No   |
| POST   | `/auth/login`         | Log in and receive tokens         | No   |
| POST   | `/auth/refresh`       | Exchange refresh token for new pair | No |
| GET    | `/auth/me`            | Get current user profile          | Yes  |

### Tasks

| Method | Endpoint        | Description                          | Auth |
|--------|-----------------|---------------------------------------|------|
| GET    | `/tasks`         | List tasks (supports filters)         | Yes  |
| GET    | `/tasks/:id`      | Get a single task                     | Yes  |
| POST   | `/tasks`          | Create a task                         | Yes  |
| PATCH  | `/tasks/:id`      | Update a task                         | Yes  |
| DELETE | `/tasks/:id`      | Delete a task                         | Yes  |

Query parameters for `GET /tasks`: `status`, `priority`, `assignee_id`, `search`, `tags` (comma-separated).

### Users (Admin only)

| Method | Endpoint              | Description            | Auth        |
|--------|-----------------------|--------------------------|-------------|
| GET    | `/users`               | List all users           | Admin       |
| PATCH  | `/users/:id/role`       | Update a user's role    | Admin       |
| DELETE | `/users/:id`            | Delete a user           | Admin       |

## Real-Time Updates

The backend emits Socket.IO events whenever tasks are created, updated, or deleted:

- `task:created`
- `task:updated`
- `task:deleted`

The frontend listens for these events (see `useSocketTasks`) and updates the TanStack Query cache directly, so all connected clients see changes instantly without polling.

## Database Schema

Two tables: `users` and `tasks`.

- `users`: `id`, `email`, `name`, `password_hash`, `role` (`admin`/`user`), `avatar_initials`, timestamps
- `tasks`: `id`, `title`, `description`, `status` (`todo`/`inprogress`/`done`), `priority` (`low`/`medium`/`high`), `due_date`, `assignee_id`, `creator_id`, `tags` (text array), timestamps

See `backend/src/db/migrate.ts` for the full SQL definition, including indexes and `updated_at` triggers.

## Authorization Rules

- Any authenticated user can view all tasks and create new tasks.
- A task can be updated or deleted by its creator, its assignee, or any admin.
- Only admins can list all users, change roles, or delete users.

## Environment Variables Reference

See `.env.example`, `backend/.env.example`, and `frontend/.env.example` for the full list. Key variables:

- `JWT_SECRET` / `JWT_REFRESH_SECRET` — sign and verify access/refresh tokens (change in production)
- `DB_HOST`, `DB_PORT`, `DB_USER`, `DB_PASSWORD`, `DB_NAME` — PostgreSQL connection
- `FRONTEND_URL` — used for CORS and Socket.IO origin
- `VITE_API_URL` / `VITE_SOCKET_URL` — frontend API and WebSocket base URLs

## Production Notes

- Set strong, unique values for `JWT_SECRET` and `JWT_REFRESH_SECRET`.
- Run `npm run build` in both `backend` and `frontend` before deploying; the backend Dockerfile builds to `dist/` and runs `node dist/index.js`.
- The frontend Docker image serves a static build via Nginx and proxies `/api` and `/socket.io` to the backend container.
- Adjust `express-rate-limit` settings in `backend/src/index.ts` for your expected traffic.
