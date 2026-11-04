FROM node:22-alpine
WORKDIR /app
COPY package.json package-lock.json ./
RUN npm ci --omit=dev && npm cache clean --force
COPY src ./src
ENV SQLITE_DB_LOCATION=/data/todo.db
RUN mkdir /data && chown node:node /data
USER node
EXPOSE 3000
CMD ["node", "src/index.js"]
