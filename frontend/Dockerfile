# Stage 1: Build React SPA
FROM node:20-alpine AS build-stage

WORKDIR /app

# Copy package descriptors and install dependencies
COPY package*.json ./
RUN npm ci

# Copy the rest of the application files
COPY . .

# Build the project (will output to /app/dist)
RUN npm run build

# Stage 2: Serve static files using Nginx
FROM nginx:stable-alpine AS production-stage

# Copy custom Nginx configuration for single page application routing
COPY nginx.conf /etc/nginx/conf.d/default.conf

# Copy build artifacts from build-stage to Nginx default public directory
COPY --from=build-stage /app/dist /usr/share/nginx/html

EXPOSE 80

CMD ["nginx", "-g", "daemon off;"]
