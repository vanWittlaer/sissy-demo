#syntax=docker/dockerfile:1.4

# pin versions
ARG PHP_VERSION=8.4
FROM ghcr.io/shopware/docker-base:$PHP_VERSION-frankenphp AS base-image
FROM ghcr.io/shopware/shopware-cli:latest-php-$PHP_VERSION AS shopware-cli

# build

FROM shopware-cli AS build

ARG SHOPWARE_PACKAGES_TOKEN

ADD . /src
WORKDIR /src

ENV SHOPWARE_CACHE_ID=docker

RUN --mount=type=secret,id=composer_auth,dst=/src/auth.json \
    --mount=type=cache,target=/root/.composer \
    --mount=type=cache,target=/root/.npm \
    /usr/local/bin/entrypoint.sh shopware-cli project ci /src

# build final image

FROM base-image

ENV SHOPWARE_DISABLE_UPDATE_CHECK=1

COPY --from=build --chown=82 --link /src /var/www/html
