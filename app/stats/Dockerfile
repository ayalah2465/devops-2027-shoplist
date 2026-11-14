FROM node:20-alpine

WORKDIR /app

COPY package.json .
RUN npm install --omit=dev --no-audit --no-fund || true

COPY server.js .

EXPOSE 3000

CMD ["node", "server.js"]