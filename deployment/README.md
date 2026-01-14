# Hopp Self-Hosted Deployment

This directory contains everything needed to self-host the Hopp backend.

## Architecture

```
Internet
    │
    ├─► nginx (TLS termination)
    │       │
    │       ├─► Backend API (port 1926)
    │       │
    │       └─► LiveKit WebSocket (port 7880)
    │
    └─► LiveKit UDP (ports 7881, 50000-50100) [direct, no proxy]
            │
            ▼
┌─────────────────────────────────────────────┐
│           Docker Compose Stack              │
│  ┌──────────┐ ┌───────┐ ┌────────┐ ┌─────┐  │
│  │ Backend  │ │LiveKit│ │Postgres│ │Redis│  │
│  │  :1926   │ │ :7880 │ │ :5432  │ │:6379│  │
│  └──────────┘ └───────┘ └────────┘ └─────┘  │
└─────────────────────────────────────────────┘
```

## Prerequisites

- Docker and Docker Compose
- A domain with DNS configured
- Nginx (or similar) for TLS termination
- Ports 7881 and 50000-50100 UDP open on your firewall

## Quick Start

1. **Copy and configure environment:**
   ```bash
   cp .env.example .env
   # Edit .env with your values
   ```

2. **Generate secrets:**
   ```bash
   # Session secret
   openssl rand -hex 32

   # LiveKit API key (short identifier)
   openssl rand -hex 8

   # LiveKit API secret (min 32 chars)
   openssl rand -hex 32
   ```

3. **Start the stack:**
   ```bash
   docker compose up -d
   ```

4. **Check logs:**
   ```bash
   docker compose logs -f
   ```

## Configuration

### Required Environment Variables

| Variable | Description | Example |
|----------|-------------|---------|
| `PUBLIC_DOMAIN` | Your public domain | `hopp.example.com` |
| `LIVEKIT_SERVER_URL` | Public WebSocket URL for LiveKit | `wss://livekit.hopp.example.com` |
| `LIVEKIT_API_KEY` | LiveKit API key | `APIdxxxxxxxx` |
| `LIVEKIT_API_SECRET` | LiveKit API secret (32+ chars) | `xxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx` |
| `SESSION_SECRET` | Random string for session encryption | (use `openssl rand -hex 32`) |
| `POSTGRES_PASSWORD` | Database password | (choose a strong password) |

### Optional: OAuth Providers

To enable social login, configure OAuth credentials:
- **Google**: [Google Cloud Console](https://console.cloud.google.com/apis/credentials)
- **GitHub**: [GitHub Developer Settings](https://github.com/settings/developers)
- **Slack**: [Slack API Apps](https://api.slack.com/apps)

Callback URLs follow this pattern:
- `https://YOUR_DOMAIN/api/auth/social/google/callback`
- `https://YOUR_DOMAIN/api/auth/social/github/callback`
- `https://YOUR_DOMAIN/api/auth/social/slack/callback`

## Nginx Configuration

You need TWO server blocks: one for the backend API, one for LiveKit.

### Backend API

```nginx
server {
    listen 443 ssl http2;
    server_name hopp.example.com;

    ssl_certificate /path/to/cert.pem;
    ssl_certificate_key /path/to/key.pem;

    location / {
        proxy_pass http://127.0.0.1:1926;
        proxy_http_version 1.1;
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection "upgrade";
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;

        # WebSocket support
        proxy_read_timeout 86400;
    }
}
```

### LiveKit (WebSocket only - UDP goes direct)

```nginx
server {
    listen 443 ssl http2;
    server_name livekit.hopp.example.com;

    ssl_certificate /path/to/cert.pem;
    ssl_certificate_key /path/to/key.pem;

    location / {
        proxy_pass http://127.0.0.1:7880;
        proxy_http_version 1.1;
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection "upgrade";
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;

        # WebSocket support
        proxy_read_timeout 86400;
    }
}
```

## Network Requirements

LiveKit requires these ports for WebRTC:

| Port | Protocol | Purpose | Via Nginx? |
|------|----------|---------|------------|
| 443 | TCP | Backend API | Yes |
| 443 | TCP | LiveKit signaling (WSS) | Yes |
| 7881 | UDP | LiveKit RTC | **No - direct** |
| 50000-50100 | UDP | LiveKit media | **No - direct** |

The UDP ports **cannot** go through nginx and must be exposed directly to the internet.

## Connecting the Desktop App

1. Open the Hopp desktop app
2. Access the Debug screen (or settings)
3. Set **Custom Backend URL** to: `hopp.example.com` (your domain, no `https://`)
4. The app will connect to your self-hosted instance

## Troubleshooting

### Backend won't start
```bash
# Check logs
docker compose logs backend

# Common issues:
# - Database not ready: wait for postgres healthcheck
# - Missing env vars: check .env file
```

### LiveKit connection fails
```bash
# Check LiveKit logs
docker compose logs livekit

# Test LiveKit is reachable
curl -I http://localhost:7880

# Common issues:
# - LIVEKIT_SERVER_URL doesn't match your nginx config
# - UDP ports blocked by firewall
# - API key/secret mismatch between backend and livekit
```

### Video/audio not working
- Ensure UDP ports 7881 and 50000-50100 are open
- Check that `LIVEKIT_SERVER_URL` uses `wss://` (not `ws://`)
- Verify nginx is properly proxying WebSocket connections

## Seed Data (Optional)

To add test users for development:

```bash
docker compose exec -T postgres psql -U hopp -d hopp < ../backend/sql/mock_data.sql
```

This creates:
- `michael@dundermifflin.com` / `hoppless`
- `dwight@dundermifflin.com` / `hoppless`

## Upgrading

```bash
# Pull latest images and rebuild
docker compose pull
docker compose build --no-cache
docker compose up -d
```

## Backup

```bash
# Backup PostgreSQL
docker compose exec postgres pg_dump -U hopp hopp > backup.sql

# Restore
docker compose exec -T postgres psql -U hopp -d hopp < backup.sql
```
