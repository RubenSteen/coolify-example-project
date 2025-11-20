# Versions
# https://hub.docker.com/r/serversideup/php/tags?name=8.4-fpm-nginx-alpine
ARG SERVERSIDEUP_PHP_VERSION=8.3-fpm-nginx-alpine

# =================================================================
# Stage 1: Composer dependencies
# =================================================================
FROM serversideup/php:${SERVERSIDEUP_PHP_VERSION} AS base

USER root

ARG USER_ID
ARG GROUP_ID

# Install ICU and required PHP extensions
RUN apk update && \
    apk add --no-cache icu-dev icu-libs && \
    docker-php-ext-install intl

RUN docker-php-serversideup-set-id www-data $USER_ID:$GROUP_ID && \
    docker-php-serversideup-set-file-permissions --owner $USER_ID:$GROUP_ID --service nginx

WORKDIR /var/www/html

COPY --chown=www-data:www-data composer.json composer.lock ./

RUN --mount=type=cache,target=/tmp/cache \
    COMPOSER_CACHE_DIR=/tmp/cache composer install --no-dev --no-interaction --no-plugins --no-scripts --prefer-dist

USER www-data

# =================================================================
# Stage 2: Frontend assets compilation
# =================================================================
FROM node:24-alpine AS static-assets

WORKDIR /app

COPY --chown=node:node package*.json vite.config.js ./
COPY --chown=node:node resources ./resources
COPY --from=base --chown=node:node /var/www/html/vendor ./vendor

RUN --mount=type=cache,target=/root/.npm \
    npm ci

RUN npm run build

# =================================================================
# Final Stage: Production image
# =================================================================

FROM serversideup/php:${SERVERSIDEUP_PHP_VERSION} AS final-stage

ARG USER_ID
ARG GROUP_ID

WORKDIR /var/www/html

USER root

# Install ICU and required PHP extensions
RUN apk update && \
    apk add --no-cache icu-dev icu-libs && \
    docker-php-ext-install intl

RUN docker-php-serversideup-set-id www-data $USER_ID:$GROUP_ID && \
    docker-php-serversideup-set-file-permissions --owner $USER_ID:$GROUP_ID --service nginx

# Install system dependencies
RUN --mount=type=cache,target=/var/cache/apk \
    apk upgrade && \
    apk add --no-cache \
    openssh-client \
    git \
    git-lfs \
    jq \
    lsof \
    vim

# Configure PHP
# COPY .docker/production/etc/php/conf.d/zzz-custom-php.ini /usr/local/etc/php/conf.d/zzz-custom-php.ini
ENV PHP_OPCACHE_ENABLE=1

# Configure entrypoint
COPY --chmod=755 .docker/production/entrypoint.d/ /etc/entrypoint.d

# Copy application files from previous stages
COPY --from=base --chown=www-data:www-data /var/www/html/vendor ./vendor
COPY --from=static-assets --chown=www-data:www-data /app/public/build ./public/build

# Copy application source code without the files/folders specified in .dockerignore
COPY --chown=www-data:www-data . /var/www/html

COPY --chown=www-data:www-data composer.json composer.lock ./

RUN composer dump-autoload

# Configure Nginx and S6 overlay
COPY .docker/production/etc/nginx/conf.d/custom.conf /etc/nginx/conf.d/custom.conf
COPY .docker/production/etc/nginx/site-opts.d/http.conf /etc/nginx/site-opts.d/http.conf

RUN mkdir -p /etc/nginx/conf.d && \
    chown -R www-data:www-data /etc/nginx && \
    chmod -R 755 /etc/nginx

USER www-data