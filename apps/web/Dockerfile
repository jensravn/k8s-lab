# Build the static bundle.
FROM node:24-alpine AS build
WORKDIR /app
RUN npm install -g pnpm@10

# Copy manifests first so dependency installs stay cached across source edits.
COPY package.json pnpm-lock.yaml ./
RUN pnpm install --frozen-lockfile

COPY . .
RUN pnpm build

# Serve it. The final image contains no Node, only the built assets.
FROM nginxinc/nginx-unprivileged:1-alpine
COPY nginx.conf /etc/nginx/conf.d/default.conf
COPY --from=build /app/dist /usr/share/nginx/html
EXPOSE 8080
