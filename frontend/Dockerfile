# Stage 1: Build the Angular app
# We use a full Node image to compile the Angular project
FROM node:18-alpine AS build

WORKDIR /app

# Copy package.json first for Docker layer caching
COPY package*.json ./

# Install all dependencies (including devDependencies needed for building)
RUN npm install

# Copy the rest of the Angular source code
COPY . .

# Build the Angular app for production
# This creates optimized static files in /app/dist/angular-15-crud
RUN npm run build -- --configuration production

# Stage 2: Serve the built files with Nginx
# We only copy the compiled static files — no Node.js needed in production!
FROM nginx:1.25-alpine

# Remove the default Nginx welcome page
RUN rm -rf /usr/share/nginx/html/*

# Copy our built Angular files to Nginx's serving directory
COPY --from=build /app/dist/angular-15-crud /usr/share/nginx/html

# Copy custom Nginx config (for Angular routing support)
COPY nginx.conf /etc/nginx/conf.d/default.conf

# Expose port 80
EXPOSE 80

# Start Nginx
CMD ["nginx", "-g", "daemon off;"]
