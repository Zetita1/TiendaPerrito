# ── Stage 1: builder ────────────────────────────────────────────────────────
FROM node:18-alpine AS builder

WORKDIR /app

COPY package*.json ./
RUN npm install

COPY server.js ./

# ── Stage 2: production ──────────────────────────────────────────────────────
FROM node:18-alpine AS production

WORKDIR /app

# Usuario no privilegiado
RUN addgroup -S appgroup && adduser -S appuser -G appgroup

# Solo dependencias de producción
COPY package*.json ./
RUN npm install --only=production && npm cache clean --force

# Copiar código desde el builder
COPY --from=builder /app/server.js ./

# Cambiar propietario y usuario
RUN chown -R appuser:appgroup /app
USER appuser

EXPOSE 3001

CMD ["node", "server.js"]