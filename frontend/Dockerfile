FROM node:20-alpine AS build

WORKDIR /app

COPY frontend/package*.json ./
RUN npm ci

COPY frontend/ .
COPY questions_*.js ./
RUN npm run build && cp gestion-quiz.html dist/

FROM nginx:1.29-alpine

RUN apk upgrade --no-cache

COPY --from=build /app/dist/ /usr/share/nginx/html/
COPY frontend/nginx.conf /etc/nginx/conf.d/default.conf

EXPOSE 80
