# Multi-stage Dockerfile for Hopp Backend + Web App

# ====================
# Stage 1: Build Web App
# ====================
FROM node:22-bookworm AS webapp-builder

# Build arg for the API base URL (your domain)
ARG VITE_API_BASE_URL
ENV VITE_API_BASE_URL=${VITE_API_BASE_URL}

WORKDIR /build

# Enable corepack for yarn
RUN corepack enable

# Copy package files for better caching
COPY package.json yarn.lock .yarnrc.yml ./
COPY .yarn ./.yarn
COPY web-app/package.json ./web-app/

# Install dependencies
RUN yarn install --mode=skip-build

# Copy web-app source
COPY web-app/ ./web-app/

# Build the web app
WORKDIR /build/web-app
RUN yarn build

# ====================
# Stage 2: Build Go Binary
# ====================
FROM golang:1.25-bookworm AS go-builder

WORKDIR /build

# Copy go mod files first for better caching
COPY backend/go.mod backend/go.sum ./
RUN go mod download

# Copy the rest of the backend source
COPY backend/ ./

# Build the binary
RUN CGO_ENABLED=0 GOOS=linux go build -o server ./main.go

# ====================
# Stage 3: Runtime Image
# ====================
FROM debian:bookworm-slim

WORKDIR /app

# Install ca-certificates for HTTPS calls
RUN apt-get update && apt-get install -y --no-install-recommends \
    ca-certificates \
    && rm -rf /var/lib/apt/lists/*

# Copy the Go binary
COPY --from=go-builder /build/server .

# Copy backend web assets (email templates, etc.)
COPY backend/web ./web

# Copy the built web app (single HTML file with embedded JS/CSS)
COPY --from=webapp-builder /build/web-app/static/react/assets/index.html ./web/web-app.html

# Copy SQL files for potential manual seeding
COPY backend/sql ./sql

ENV USE_TLS=false

EXPOSE 1926

CMD ["./server"]
