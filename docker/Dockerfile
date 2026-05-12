FROM composer/composer:2-bin AS composer

FROM ghcr.io/freeunitorg/freeunit:latest-php8.5 AS base

RUN apt-get update && apt-get install -y --no-install-recommends \
        unzip curl libpq-dev && \
    rm -rf /var/lib/apt/lists/* && \
    docker-php-ext-install --jobs=1 pdo_pgsql 2>&1 || echo "pdo_pgsql install failed"

#
# Development
#

FROM base AS dev

COPY --from=composer /composer /usr/bin/composer

COPY docker/dev/unit-config.json /docker-entrypoint.d/unit-config.json

# Dev runs as root — FreeUnit requires specific uid/gid setup for non-root
# Unit runs unprivileged by default but needs proper socket permissions

#
# Production
#

FROM base AS prod-builder
COPY --from=composer /composer /usr/bin/composer
COPY .. /app
RUN --mount=type=cache,target=/tmp/cache \
    composer install --no-dev --no-progress --no-interaction --classmap-authoritative && \
    rm composer.lock composer.json

FROM base AS prod
ENV APP_ENV=prod
COPY docker/prod/unit-config.json /var/lib/unit/config.json
COPY --from=prod-builder --chown=www-data:www-data /app /app
USER www-data
