FROM node:18-alpine

WORKDIR /email-api

COPY package*.json .

RUN npm install

COPY . .

EXPOSE 5001

CMD ["node", "server.js"]
