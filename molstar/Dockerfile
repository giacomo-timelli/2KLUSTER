# ---------- Build stage ----------
FROM node:22-bookworm AS builder

WORKDIR /app

COPY package*.json ./
RUN npm ci

COPY . .
RUN npm run build


# ---------- Runtime stage ----------
FROM nginx:alpine

COPY --from=builder /app/build/viewer /usr/share/nginx/html

EXPOSE 80

CMD ["nginx", "-g", "daemon off;"]
