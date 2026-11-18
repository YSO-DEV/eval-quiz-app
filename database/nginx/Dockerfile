# Récupération des fichiers du frontend
FROM quizacr755520.azurecr.io/eval-quiz-app-frontend:latest AS frontend

# Image Nginx finale
FROM nginx:1.29-alpine

# Adresse du backend sur Azure (affichée par : az containerapp show ... ingress.fqdn)
ARG BACKEND_HOST=quiz-backend.happyforest-f3382df7.francecentral.azurecontainerapps.io

# Copie des fichiers frontend
COPY --from=frontend /usr/share/nginx/html/ /usr/share/nginx/html/

# Copie de la configuration Nginx existante (non modifiée dans le projet)
COPY nginx.conf /etc/nginx/conf.d/default.conf

# Adaptation de la config pour Azure Container Apps, uniquement dans l'image :
# 1. le serveur du backend : adresse Azure, port 443
# 2. proxy_pass en https, avec l'envoi du nom du serveur (SNI)
# 3. l'en-tête Host doit être l'adresse du backend, sinon Azure répond 404
RUN sed -i \
  -e "s#server backend:3007;#server ${BACKEND_HOST}:443;#" \
  -e "s#proxy_pass http://backend;#proxy_pass https://backend;\n        proxy_ssl_server_name on;\n        proxy_ssl_name ${BACKEND_HOST};#" \
  -e "s#proxy_set_header Host \$host;#proxy_set_header Host ${BACKEND_HOST};#" \
  -e "s#location /health {#location /health {\n        proxy_set_header Host ${BACKEND_HOST};#" \
  /etc/nginx/conf.d/default.conf