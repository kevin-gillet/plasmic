# Build stage
FROM node:18-alpine AS builder

# Build arguments for configuration
ARG DB_HOST
ARG DB_PASSWORD

WORKDIR /plasmic

# Install essential build dependencies only
RUN apk add --no-cache bash curl python3 py3-pip build-base

# Optimize file watching for development
RUN echo "fs.inotify.max_user_watches=524288" >> /etc/sysctl.conf && \
    sysctl -p /etc/sysctl.conf || echo "Warning: Could not apply sysctl settings in container"

# Copy package files first to leverage Docker's cache layers
COPY package.json yarn.lock ./
COPY packages/*/package.json ./packages/
COPY platform/*/package.json ./platform/

# Install global dependencies
RUN npm install -g concurrently nx

# Install dependencies (cached if package.json unchanged)
RUN yarn install --frozen-lockfile

# Now copy the rest of the code
COPY . .

# Configure database connection
RUN sed -i "s/\"host\": \"localhost\"/\"host\": \"${DB_HOST}\"/g" platform/wab/ormconfig.json && \
    grep -q '"password":' platform/wab/ormconfig.json || \
    sed -i '/"database":/a \ "password": "'${DB_PASSWORD}'",' platform/wab/ormconfig.json

# Run setup (build, etc.)
RUN yarn setup-all && nx reset && \
    chmod -R 777 /plasmic/node_modules/.cache || true

# Runtime stage
FROM node:18-alpine

# Runtime dependencies only
RUN apk add --no-cache bash curl postgresql-client

# Install global dependencies
RUN npm install -g concurrently

# Create non-root user
RUN addgroup -S plasmic && adduser -S plasmic -G plasmic
USER plasmic

WORKDIR /plasmic

# Copy only what's needed from the build stage
COPY --from=builder --chown=plasmic:plasmic /plasmic/node_modules /plasmic/node_modules
COPY --from=builder --chown=plasmic:plasmic /plasmic/packages /plasmic/packages
COPY --from=builder --chown=plasmic:plasmic /plasmic/platform /plasmic/platform
COPY --from=builder --chown=plasmic:plasmic /plasmic/platform/wab/tools/docker-dev/secrets.json /home/plasmic/.plasmic

# Create a better entrypoint script
COPY --chown=plasmic:plasmic <<EOF /usr/local/bin/entrypoint.sh
#!/bin/bash
cd /plasmic
echo "Starting development servers..."
yarn dev
EOF

RUN chmod +x /usr/local/bin/entrypoint.sh

EXPOSE 3003
ENTRYPOINT ["/usr/local/bin/entrypoint.sh"]