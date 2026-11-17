FROM postgres:16-alpine

RUN apk upgrade --no-cache

COPY init.sql /docker-entrypoint-initdb.d/01-init.sql
