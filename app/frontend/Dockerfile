FROM node:20-alpine AS build
WORKDIR /src
COPY app.js style.css index.html build.js ./
RUN node build.js

FROM nginx:alpine
COPY --from=build /src/dist/ /usr/share/nginx/html/
COPY nginx.conf /etc/nginx/conf.d/default.conf
EXPOSE 80