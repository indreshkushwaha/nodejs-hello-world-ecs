# docker/Dockerfile

# Use official Node.js LTS Alpine image for smaller size and better security
FROM node:18-alpine AS base

# Set working directory
WORKDIR /usr/src/app

# Create non-root user for security
RUN addgroup -g 1001 -S nodejs && \
    adduser -S nodejs -u 1001

# Copy package files
COPY app/package*.json ./

# Install dependencies with specific npm version for consistency
RUN npm ci --only=production && \
    npm cache clean --force

# Copy application code
COPY app/ .

# Change ownership to nodejs user
RUN chown -R nodejs:nodejs /usr/src/app

# Switch to non-root user
USER nodejs

# Expose port
EXPOSE 3000

# Add health check
HEALTHCHECK --interval=30s --timeout=3s --start-period=5s --retries=3 \
    CMD node -e "require('http').get('http://localhost:3000/health', (res) => { \
        process.exit(res.statusCode === 200 ? 0 : 1) \
    }).on('error', () => process.exit(1))"

# Use exec form for better signal handling
CMD ["node", "server.js"]
