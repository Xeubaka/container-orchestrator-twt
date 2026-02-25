# Codecon — Nginx

Reverse proxy that acts as the single public entry point for the application.

## Tech Stack

| Technology  | Purpose                              |
|-------------|--------------------------------------|
| Nginx 1.27  | Reverse proxy, static-asset caching  |

## What It Does

Nginx sits in front of both services and routes traffic based on the request path:

| Path pattern        | Routed to           | Notes                                                     |
|---------------------|---------------------|-----------------------------------------------------------|
| `/api/*`            | `backend:3000`      | `/api` prefix is **stripped** before forwarding           |
| `/uploads/*`        | `backend:3000`      | User-uploaded files; cached for 7 days                    |
| `/_next/static/*`   | `frontend:3001`     | Hashed static assets; cached for 1 year (immutable)       |
| `/*` (everything else) | `frontend:3001` | SSR pages, Server Actions, HMR WebSocket                  |

### URL rewriting

The `/api` prefix exists only at the Nginx level. NestJS never sees it:

```
Client:  GET /api/auth/login
           ↓  rewrite ^/api/(.*) /$1
Backend: GET /auth/login
```

### Compression

Gzip is enabled for `text/plain`, `application/json`, `application/javascript`, and `text/css` responses larger than 1 KB.

### Upload limits

`client_max_body_size 10m` — allows files up to 10 MB for avatar and post image uploads.

### WebSocket / SSE support

The catch-all `location /` block passes `Upgrade` and `Connection` headers, supporting Next.js Hot Module Replacement WebSockets in development and any Server-Sent Events used by the frontend.

## Configuration File

The entire configuration lives in a single file: [nginx.conf](nginx.conf).

It is mounted into the container as a **read-only bind mount**:

```yaml
volumes:
  - ./nginx/nginx.conf:/etc/nginx/nginx.conf:ro
```

To apply changes, rebuild/restart the container:

```bash
docker compose restart nginx
```

## Running in Docker Compose

Nginx is started automatically as part of `docker compose up`. It depends on both `frontend` and `backend` being up before it starts.

The host port is configurable via the `NGINX_PORT` environment variable (default: `80`):

```env
NGINX_PORT=8080
```

## Local Testing (without Docker)

Nginx itself is only intended to run inside Docker Compose. For local development, run the frontend (`npm run dev`) and backend (`npm run start:dev`) directly and access them on their native ports:

- Backend: [http://localhost:3000](http://localhost:3000)
- Frontend: [http://localhost:3001](http://localhost:3001)
