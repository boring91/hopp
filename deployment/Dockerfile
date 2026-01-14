# Multi-stage Dockerfile for Hopp Backend
# Stage 1: Build the Go binary
FROM golang:1.25-bookworm AS builder

WORKDIR /build

# Copy go mod files first for better caching
COPY backend/go.mod backend/go.sum ./
RUN go mod download

# Copy the rest of the backend source
COPY backend/ ./

# Build the binary
RUN CGO_ENABLED=0 GOOS=linux go build -o server ./main.go

# Stage 2: Runtime image
FROM debian:bookworm-slim

WORKDIR /app

# Install ca-certificates for HTTPS calls
RUN apt-get update && apt-get install -y --no-install-recommends \
    ca-certificates \
    && rm -rf /var/lib/apt/lists/*

# Copy the binary from builder
COPY --from=builder /build/server .

# Copy web assets (email templates, etc.)
COPY backend/web ./web

# Copy SQL files for potential manual seeding
COPY backend/sql ./sql

ENV USE_TLS=false

EXPOSE 1926

CMD ["./server"]
