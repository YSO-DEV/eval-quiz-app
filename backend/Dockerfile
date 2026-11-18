FROM node:24-alpine

WORKDIR /app

RUN apk upgrade --no-cache

COPY package*.json ./
RUN npm ci --omit=dev \
    && npm cache clean --force \
    && rm -rf /usr/local/lib/node_modules/npm /usr/local/bin/npm /usr/local/bin/npx

COPY server.js ./

USER node

EXPOSE 3007

CMD ["node", "server.js"]