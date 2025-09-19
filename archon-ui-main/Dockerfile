# Fast single-stage frontend build with security hardening
FROM node:24-alpine AS builder

# Install curl for health checks, bash for entrypoint, and set environment
RUN apk add --no-cache curl bash

# Set environment defaults (NODE_ENV set after npm install to get devDependencies)
ENV ARCHON_UI_HOST=0.0.0.0
ENV ARCHON_UI_PORT=5173

WORKDIR /app

# Copy package files first for better layer caching
COPY package*.json ./

# Install all dependencies (including devDependencies for vite)
# Set NODE_ENV after install so we get dev dependencies
RUN npm ci --include=dev && npm cache clean --force

# Copy source code
COPY . .

RUN npm run build

FROM nginx:alpine-slim

# Set nginx port (can be overridden at runtime)
ENV NGINX_PORT=5173

# Create non-root user for security (UID 1001 to match other containers)
RUN addgroup -g 1001 archon && \
    adduser -D -u 1001 -G archon archon

# Copy built files from builder and set ownership
COPY --from=builder --chown=archon:archon /app/dist /usr/share/nginx/html

# Create nginx configuration template that will be processed at runtime
# This uses the official nginx image's envsubst templating
RUN mkdir -p /etc/nginx/templates && \
    echo 'server { \
    listen ${NGINX_PORT}; \
    listen [::]:${NGINX_PORT}; \
    server_name localhost; \
    root /usr/share/nginx/html; \
    index index.html; \
    \
    # Health check endpoint - return 200 OK \
    location /health { \
        access_log off; \
        return 200 "healthy\\n"; \
        add_header Content-Type text/plain; \
    } \
    \
    # SPA routing - serve index.html for all routes \
    location / { \
        try_files $uri $uri/ /index.html; \
    } \
    \
    # Security headers \
    add_header X-Frame-Options "SAMEORIGIN" always; \
    add_header X-Content-Type-Options "nosniff" always; \
    add_header X-XSS-Protection "1; mode=block" always; \
}' > /etc/nginx/templates/default.conf.template

# Install curl for healthcheck
RUN apk add --no-cache curl

# Configure nginx to run as non-root
# nginx needs write access to these directories
RUN touch /var/run/nginx.pid && \
    chown -R archon:archon \
        /var/run/nginx.pid \
        /var/cache/nginx \
        /var/log/nginx \
        /usr/share/nginx/html \
        /etc/nginx/conf.d \
        /etc/nginx/templates && \
    # Also need write access to /run for nginx.pid
    chown archon:archon /run && \
    # Make directories writable
    chmod -R 755 /usr/share/nginx/html /etc/nginx/conf.d /etc/nginx/templates && \
    chmod 755 /run

# Switch to non-root user
USER 1001

EXPOSE ${NGINX_PORT}

# Health check
HEALTHCHECK --interval=30s --timeout=3s --start-period=10s --retries=3 \
    CMD curl -f http://localhost:${NGINX_PORT}/health || exit 1

