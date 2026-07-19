# Build stage: compile and purge Tailwind into a single static stylesheet.
# Nothing from this stage ships except the generated CSS — no JS reaches the browser.
FROM node:22-alpine AS build
WORKDIR /build
COPY package.json package-lock.json ./
RUN npm ci
COPY src/ ./src/
COPY html/ ./html/
RUN npm run build

FROM nginx:alpine
COPY --from=build /build/html/ /usr/share/nginx/html/
