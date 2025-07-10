# Build stage
FROM ruby:3.4.4-alpine3.21 AS builder

ARG NODE_VERSION="23.7.0"
ARG PNPM_VERSION="10.2.0"
ARG BUNDLE_WITHOUT="development:test"
ARG RAILS_ENV=production
ARG RAILS_SERVE_STATIC_FILES=true
ARG NODE_OPTIONS="--max-old-space-size=4096 --openssl-legacy-provider"

ENV NODE_VERSION=${NODE_VERSION}
ENV PNPM_VERSION=${PNPM_VERSION}
ENV BUNDLE_WITHOUT=${BUNDLE_WITHOUT}
ENV BUNDLER_VERSION=2.5.11
ENV RAILS_SERVE_STATIC_FILES=${RAILS_SERVE_STATIC_FILES}
ENV RAILS_ENV=${RAILS_ENV}
ENV NODE_OPTIONS=${NODE_OPTIONS}
ENV BUNDLE_PATH="/gems"

# Install build dependencies in one layer and clean up
RUN apk update && apk add --no-cache --virtual .build-deps \
    build-base \
    musl-dev \
    gcc \
    make \
    g++ \
    linux-headers \
    postgresql-dev \
    git \
    curl \
    xz \
    openssl-dev \
    && apk add --no-cache \
    openssl \
    tzdata \
    postgresql-client \
    vips \
    && gem install bundler \
    && rm -rf /var/cache/apk/*

# Install Node.js from official image
COPY --from=node:23-alpine /usr/local/bin/node /usr/local/bin/
COPY --from=node:23-alpine /usr/local/lib/node_modules /usr/local/lib/node_modules

# Create symlinks for npm and install pnpm
RUN ln -s /usr/local/lib/node_modules/npm/bin/npm-cli.js /usr/local/bin/npm \
    && ln -s /usr/local/lib/node_modules/npm/bin/npx-cli.js /usr/local/bin/npx \
    && npm install -g pnpm@${PNPM_VERSION} \
    && npm cache clean --force

ENV PNPM_HOME="/root/.local/share/pnpm"
ENV PATH="$PNPM_HOME:$PATH"

WORKDIR /app

# Copy dependency files
COPY Gemfile Gemfile.lock ./
COPY package.json pnpm-lock.yaml ./

# Install Ruby dependencies
RUN bundle config set --local force_ruby_platform true \
    && bundle config set without 'development test' \
    && bundle install -j 4 -r 3 \
    && bundle clean --force

# Install Node dependencies
RUN pnpm install --frozen-lockfile --prod \
    && pnpm store prune

# Copy application code
COPY . /app

# Generate git SHA and create log directory
RUN mkdir -p /app/log \
    && git rev-parse HEAD > /app/.git_sha 2>/dev/null || echo "unknown" > /app/.git_sha

# Precompile assets for production
RUN if [ "$RAILS_ENV" = "production" ]; then \
    SECRET_KEY_BASE=precompile_placeholder \
    RAILS_LOG_TO_STDOUT=enabled \
    DATABASE_URL=postgresql://dummy:dummy@localhost:5432/dummy \
    bundle exec rake assets:precompile --trace; \
    fi

# Clean up build artifacts
RUN rm -rf \
    .git \
    .gitignore \
    spec \
    test \
    tmp/cache \
    node_modules \
    /gems/ruby/3.4.0/cache/*.gem \
    && find /gems/ruby/3.4.0/gems/ \( -name "*.c" -o -name "*.o" -o -name "*.so" \) -delete \
    && apk del .build-deps \
    && rm -rf /var/cache/apk/* \
    && rm -rf /tmp/* \
    && rm -rf /root/.cache

# Production stage
FROM ruby:3.4.4-alpine3.21

ARG RAILS_ENV=production
ARG RAILS_SERVE_STATIC_FILES=true
ARG BUNDLE_WITHOUT="development:test"
ARG EXECJS_RUNTIME="Disabled"
ARG BUNDLE_FORCE_RUBY_PLATFORM=1

ENV BUNDLER_VERSION=2.5.11
ENV BUNDLE_WITHOUT=${BUNDLE_WITHOUT}
ENV EXECJS_RUNTIME=${EXECJS_RUNTIME}
ENV RAILS_SERVE_STATIC_FILES=${RAILS_SERVE_STATIC_FILES}
ENV BUNDLE_FORCE_RUBY_PLATFORM=${BUNDLE_FORCE_RUBY_PLATFORM}
ENV RAILS_ENV=${RAILS_ENV}
ENV BUNDLE_PATH="/gems"

# Install only runtime dependencies
RUN apk update && apk add --no-cache \
    openssl \
    tzdata \
    postgresql-client \
    imagemagick \
    vips \
    && gem install bundler \
    && rm -rf /var/cache/apk/*

# Copy only necessary files from builder
COPY --from=builder /gems/ /gems/
COPY --from=builder /app /app

# Copy Node.js runtime (only if needed for production)
# Remove these lines if Node.js is not needed in production
COPY --from=node:23-alpine /usr/local/bin/node /usr/local/bin/
COPY --from=node:23-alpine /usr/local/lib/node_modules/npm /usr/local/lib/node_modules/npm
RUN ln -s /usr/local/lib/node_modules/npm/bin/npm-cli.js /usr/local/bin/npm

WORKDIR /app

# Create non-root user for security
RUN addgroup -g 1000 -S chatwoot \
    && adduser -u 1000 -S chatwoot -G chatwoot \
    && chown -R chatwoot:chatwoot /app

USER chatwoot

EXPOSE 3000

CMD ["bundle", "exec", "rails", "server", "-b", "0.0.0.0"]
