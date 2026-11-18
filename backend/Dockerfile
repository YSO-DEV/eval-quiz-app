
# BUILD

FROM alpine:3.24 AS build

WORKDIR /app

# Installation de Node.js et npm
RUN apk add --no-cache nodejs npm

# Copie des fichiers de dépendances
COPY package*.json ./

# Installation des dépendances de production
RUN npm ci --omit=dev


# PRODUCTION
FROM alpine:3.24 AS production

WORKDIR /app

# Installation de Node.js uniquement
RUN apk add --no-cache nodejs

# Création d'un utilisateur non privilégié
RUN addgroup -S appgroup && adduser -S -G appgroup appuser

# Récupération des dépendances
COPY --from=build --chown=appuser:appgroup /app/node_modules ./node_modules

# Copie des fichiers nécessaires à l'application
COPY --chown=appuser:appgroup package.json server.js ./

# Configuration de l'environnement
ENV NODE_ENV=production
ENV PORT=3007

# Exécution sans privilèges root
USER appuser

# Port du backend
EXPOSE 3007

# Démarrage de l'application
CMD ["node", "server.js"]
