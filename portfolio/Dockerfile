FROM node:20-alpine AS builder
WORKDIR /app
COPY perecolletsite/package*.json ./
RUN npm ci
COPY perecolletsite/ .
RUN npm run build

FROM nginx:alpine
COPY --from=builder /app/dist /usr/share/nginx/html
EXPOSE 80
