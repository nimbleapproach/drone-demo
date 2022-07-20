# syntax=docker/dockerfile:1

FROM node:latest

WORKDIR /app

COPY ["package.json", "package-lock.json*", "./"]

RUN npm install

COPY . .

CMD [ "node", "app.js" ]
