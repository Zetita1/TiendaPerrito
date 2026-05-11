# ── Stage 1: builder ────────────────────────────────────────────────────────
FROM node:18-alpine AS builder

WORKDIR /app

# Copiar archivos estáticos
COPY index.html app.js ./

# ── Stage 2: production ──────────────────────────────────────────────────────
FROM nginx:alpine AS production

# Limpiar contenido por defecto
RUN rm -rf /usr/share/nginx/html/*

# Copiar archivos estáticos desde el builder
COPY --from=builder /app/index.html /app/app.js /usr/share/nginx/html/

# Copiar configuración personalizada de Nginx
COPY default.conf /etc/nginx/conf.d/default.conf

# nginx:alpine ya ejecuta workers como usuario 'nginx' (no root)
EXPOSE 80

CMD ["nginx", "-g", "daemon off;"]
