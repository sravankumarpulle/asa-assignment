FROM node:20.18.0-alpine3.20

ARG APP_UID=10001
ARG APP_GID=10001

ENV NODE_ENV=production \
    PORT=3001

WORKDIR /opt/notify

RUN addgroup -g "${APP_GID}" -S notifygroup \
    && adduser -u "${APP_UID}" -S notifyuser -G notifygroup

COPY package*.json ./

RUN npm ci --omit=dev \
    && npm cache clean --force

COPY --chown=notifyuser:notifygroup . .

USER notifyuser

EXPOSE 3001

HEALTHCHECK \
    --interval=30s \
    --timeout=5s \
    --start-period=10s \
    --retries=3 \
    CMD node -e "require('http').get('http://127.0.0.1:3001/health', r => process.exit(r.statusCode >= 200 && r.statusCode < 400 ? 0 : 1)).on('error', () => process.exit(1))"

CMD ["npm", "start"]