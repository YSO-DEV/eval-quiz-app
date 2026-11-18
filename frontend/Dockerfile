
# BUILD
FROM node:24-alpine AS build

WORKDIR /app

# Copie des fichiers de dépendances
COPY frontend/package*.json ./

# Installation reproductible des dépendances
RUN npm ci

# Copie du code source
COPY frontend/ .
COPY questions_*.js ./

# Compilation du frontend
RUN npm run build && cp gestion-quiz.html dist/


# PRODUCTION
FROM nginx:1.29-alpine AS production

# Copie des fichiers compilés
COPY --from=build --chown=nginx:nginx /app/dist/ /usr/share/nginx/html/

# Configuration personnalisée de Nginx
COPY frontend/nginx.conf /etc/nginx/conf.d/default.conf

EXPOSE 80
