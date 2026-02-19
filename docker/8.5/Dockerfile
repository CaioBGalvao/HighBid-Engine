FROM node:24-bookworm-slim AS node_upstream
FROM dunglas/frankenphp:1.11.2-php8.5

ARG WWWGROUP
ARG POSTGRES_VERSION=18
ARG TZ=UTC

WORKDIR /var/www/html

# Timezone
RUN ln -snf /usr/share/zoneinfo/$TZ /etc/localtime && echo $TZ > /etc/timezone

# System dependencies
RUN apt-get update \
    && mkdir -p /etc/apt/keyrings \
    # Install essential packages
    && apt-get install -y gnupg gosu curl ca-certificates zip unzip git supervisor libcap2-bin libpng-dev python3 dnsutils librsvg2-bin fswatch ffmpeg nano \
    # Postgres Client
    && curl -sS https://www.postgresql.org/media/keys/ACCC4CF8.asc | gpg --dearmor | tee /etc/apt/keyrings/pgdg.gpg >/dev/null \
    && echo "deb [signed-by=/etc/apt/keyrings/pgdg.gpg] http://apt.postgresql.org/pub/repos/apt noble-pgdg main" > /etc/apt/sources.list.d/pgdg.list \
    && apt-get update \
    && apt-get install -y postgresql-client-$POSTGRES_VERSION \
    && apt-get -y autoremove \
    && apt-get clean \
    && rm -rf /var/lib/apt/lists/* /tmp/* /var/tmp/*

# PHP Extensions
RUN install-php-extensions \
    pcntl \
    pdo_pgsql \
    pgsql \
    gd \
    intl \
    zip \
    bcmath \
    redis \
    opcache

# Node.js
COPY --from=node_upstream /usr/local/bin/node /usr/local/bin/node
COPY --from=node_upstream /usr/local/include/node /usr/local/include/node
COPY --from=node_upstream /usr/local/lib/node_modules /usr/local/lib/node_modules
COPY --from=node_upstream /opt/ /opt/

# Symlink npm, npx, corepack
RUN ln -s /usr/local/lib/node_modules/npm/bin/npm-cli.js /usr/local/bin/npm \
    && ln -s /usr/local/lib/node_modules/npm/bin/npx-cli.js /usr/local/bin/npx \
    && ln -s /usr/local/lib/node_modules/corepack/dist/corepack.js /usr/local/bin/corepack


# Add Sail user
RUN groupadd --force -g $WWWGROUP sail
RUN useradd -ms /bin/bash --no-user-group -g $WWWGROUP -u 1337 sail

COPY start-container /usr/local/bin/start-container
COPY supervisord.conf /etc/supervisor/conf.d/supervisord.conf
COPY php.ini /usr/local/etc/php/conf.d/99-sail.ini

RUN chmod +x /usr/local/bin/start-container

EXPOSE 8000

ENTRYPOINT ["start-container"]
