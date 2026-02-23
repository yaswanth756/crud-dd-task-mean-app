# Stage: Production
FROM node:18-alpine

# Set working directory inside the container
WORKDIR /app

# Copy package.json and package-lock.json first (for Docker layer caching)
# This means if your code changes but dependencies don't, Docker reuses the cached layer
COPY package*.json ./

# Install dependencies
RUN npm install --production

# Copy the rest of the backend source code
COPY . .

# Expose the port the backend runs on
EXPOSE 8080

# Start the server
CMD ["node", "server.js"]
