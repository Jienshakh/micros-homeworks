FROM node:25.6.0-alpine3.23
WORKDIR /app

COPY package*.json ./
RUN npm install
COPY src ./

CMD ["node", "server.js"]
