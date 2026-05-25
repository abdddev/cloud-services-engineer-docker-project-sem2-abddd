FROM node:16-alpine AS build

WORKDIR /app

COPY package.json package-lock.json ./
RUN npm ci

COPY . .

ARG VUE_APP_API_URL=http://localhost:8081
ENV VUE_APP_API_URL=$VUE_APP_API_URL
RUN npm run build

FROM nginx:1.27-alpine AS run

COPY nginx/nginx.conf /etc/nginx/conf.d/default.conf
COPY --from=build /app/dist /usr/share/nginx/html/momo-store

EXPOSE 80

CMD ["nginx", "-g", "daemon off;"]
