# 🔧 Creating Optimized Docker Images for PHP: From Basics to Advanced Techniques

*Published: [Current Date]*

In our previous article, we explored how to set up a Docker environment for Laravel applications. Today, we'll dive deeper into the art of creating optimized Docker images for PHP applications. Whether you're a DevOps engineer or a PHP developer, understanding Docker image optimization is crucial for building efficient, secure, and maintainable containerized applications.

## 📋 Table of Contents
- [Why Optimize Docker Images?](#why-optimize-docker-images)
- [Understanding Docker Image Layers](#understanding-docker-image-layers)
- [Optimization Techniques](#optimization-techniques)
- [Analyzing Your Current Setup](#analyzing-your-current-setup)
- [Step-by-Step Optimization Guide](#step-by-step-optimization-guide)
- [Advanced Optimization Strategies](#advanced-optimization-strategies)
- [Security Considerations](#security-considerations)
- [Performance Testing](#performance-testing)
- [Using Makefile for Docker Operations](#using-makefile-for-docker-operations)
- [Docker and Linux Cheatsheet](#docker-and-linux-cheatsheet)
- [Conclusion](#conclusion)

## Why Optimize Docker Images?

Optimized Docker images offer several key benefits:

1. **Faster Builds**: Smaller images build more quickly, saving development time.
2. **Reduced Storage**: Smaller images require less storage space.
3. **Faster Deployments**: Smaller images transfer faster over networks.
4. **Enhanced Security**: Fewer components mean smaller attack surface.
5. **Better Resource Utilization**: Optimized images use system resources more efficiently.

## Understanding Docker Image Layers

Docker images are built in layers, with each instruction in a Dockerfile creating a new layer. Understanding this concept is crucial for optimization:

```
Layer 1: Base image
Layer 2: Install dependencies
Layer 3: Copy application code
Layer 4: Configure application
Layer 5: Set permissions
```

Each layer adds to the image size, and once a layer is created, it cannot be modified without rebuilding all subsequent layers. This is why the order of instructions matters significantly.

## Optimization Techniques

### 1. Use Multi-Stage Builds

Multi-stage builds allow you to use multiple stages in a single Dockerfile, where each stage can have its own base image and dependencies. This is perfect for separating build dependencies from runtime dependencies.

```dockerfile
# Build stage
FROM composer:2 AS composer
WORKDIR /app
COPY composer.json composer.lock ./
RUN composer install --no-dev --no-scripts --no-autoloader --prefer-dist

# Runtime stage
FROM php:8.4-fpm
COPY --from=composer /app/vendor /var/www/vendor
COPY . /var/www
```

### 2. Choose the Right Base Image

Alpine-based images are significantly smaller than Debian-based ones:

- `php:8.4-fpm` (Debian): ~400MB
- `php:8.4-fpm-alpine` (Alpine): ~100MB

### 3. Minimize Layers

Combine related commands using `&&` to reduce the number of layers:

```dockerfile
# Bad
RUN apt-get update
RUN apt-get install -y package1 package2

# Good
RUN apt-get update && apt-get install -y package1 package2
```

### 4. Clean Up After Installation

Remove unnecessary files after installing packages:

```dockerfile
RUN apt-get update && apt-get install -y package1 package2 \
    && apt-get clean \
    && rm -rf /var/lib/apt/lists/*
```

### 5. Use .dockerignore

Create a `.dockerignore` file to exclude unnecessary files from the build context:

```
.git
.env
node_modules
tests
```

## Analyzing Your Current Setup

Let's analyze your current Docker setup and identify optimization opportunities:

### Current Dockerfile Analysis

Your current Dockerfile uses a custom PHP 8.4 FPM image (`dommin/php-8.4-fpm`). This is a good choice for your Laravel 12 application, as it provides the latest PHP features. However, there are several areas where we can improve:

1. **Layer Optimization**: Your Dockerfile could benefit from combining related commands to reduce layers.

2. **Development vs. Production**: Your current setup includes development tools that might not be needed in production.

3. **Caching Strategy**: The order of commands could be optimized to better utilize Docker's layer caching.

4. **Node.js Version**: Your project uses Node 22, which is the latest version. This should be explicitly specified in your Dockerfile.

## Step-by-Step Optimization Guide

Let's create an optimized Dockerfile for your Laravel 12 application:

```dockerfile
# Build stage for Composer dependencies
FROM composer:2 AS composer
WORKDIR /app
COPY composer.json composer.lock ./
RUN composer install --no-dev --no-scripts --no-autoloader --prefer-dist

# Build stage for Node.js dependencies
FROM node:22-alpine AS node
WORKDIR /app
COPY package.json package-lock.json ./
RUN npm ci

# Build stage for frontend assets
FROM node:22-alpine AS frontend
WORKDIR /app
COPY --from=node /app/node_modules ./node_modules
COPY . .
RUN npm run build

# Production stage
FROM dommin/php-8.4-fpm AS production
WORKDIR /var/www

# Install system dependencies
RUN apt-get update && apt-get install -y \
    nginx \
    supervisor \
    libpq-dev \
    libzip-dev \
    zip \
    unzip \
    git \
    && docker-php-ext-install pdo pdo_pgsql zip \
    && apt-get clean \
    && rm -rf /var/lib/apt/lists/*

# Install Redis extension
RUN pecl install redis && docker-php-ext-enable redis

# Copy Composer dependencies
COPY --from=composer /app/vendor /var/www/vendor

# Copy application code
COPY . /var/www

# Copy frontend assets
COPY --from=frontend /app/public/build /var/www/public/build

# Copy configuration files
COPY docker/php/php.ini /usr/local/etc/php/conf.d/custom.ini
COPY docker/php/www.conf /usr/local/etc/php-fpm.d/www.conf
COPY docker/supervisord.conf /etc/supervisor/supervisord.conf
COPY docker/nginx/conf.d/default.conf /etc/nginx/conf.d/default.conf

# Set permissions
RUN chown -R www-data:www-data /var/www \
    && chmod -R 755 /var/www/storage

# Expose ports
EXPOSE 80 9000

# Start services
CMD ["/usr/bin/supervisord", "-c", "/etc/supervisor/supervisord.conf"]
```

### Key Improvements:

1. **Multi-Stage Build**: Separates build dependencies from runtime dependencies.
2. **Optimized Layers**: Combines related commands to reduce layers.
3. **Production Focus**: Removes development tools from the production image.
4. **Proper Permissions**: Sets correct permissions for Laravel's storage directory.
5. **Node.js 22**: Uses the latest Node.js version for frontend builds.

## Advanced Optimization Strategies

### 1. Use BuildKit for Faster Builds

Enable Docker BuildKit for faster and more efficient builds:

```bash
DOCKER_BUILDKIT=1 docker build -t myapp .
```

### 2. Implement Layer Caching

Order your Dockerfile instructions from least to most frequently changing:

1. Install system dependencies
2. Install PHP extensions
3. Copy Composer files and install dependencies
4. Copy application code
5. Configure the application

### 3. Use PHP-FPM Configuration Optimization

Optimize your PHP-FPM configuration for production:

```ini
pm = dynamic
pm.max_children = 50
pm.start_servers = 5
pm.min_spare_servers = 5
pm.max_spare_servers = 35
pm.max_requests = 500
```

### 4. Implement OpCache Optimization

Enable and configure OpCache for better performance:

```ini
opcache.enable=1
opcache.memory_consumption=256
opcache.interned_strings_buffer=16
opcache.max_accelerated_files=20000
opcache.validate_timestamps=0
opcache.revalidate_freq=0
opcache.save_comments=1
opcache.fast_shutdown=1
```

## Security Considerations

### 1. Run as Non-Root User

Always run your application as a non-root user:

```dockerfile
USER www-data
```

### 2. Remove Unnecessary Packages

Only install packages that are absolutely necessary for your application to run.

### 3. Scan for Vulnerabilities

Regularly scan your Docker images for vulnerabilities:

```bash
docker scan myapp
```

### 4. Use Specific Versions

Always use specific versions for base images and packages to ensure reproducibility.

## Performance Testing

After optimizing your Docker images, it's important to test their performance:

### 1. Image Size Comparison

Compare the size of your optimized image with the original:

```bash
docker images | grep myapp
```

### 2. Build Time Comparison

Measure the build time of your optimized Dockerfile:

```bash
time docker build -t myapp .
```

### 3. Runtime Performance

Test the runtime performance of your application in the optimized container:

```bash
docker run -d --name myapp myapp
docker stats myapp
```

## Using Makefile for Docker Operations

Your project includes a Makefile that simplifies Docker operations. Here's how to use it effectively:

### Key Makefile Commands

```bash
# Start the application
make up

# Stop the application
make down

# Build containers
make build

# Install dependencies
make install

# Run migrations
make migrate

# Fresh migrations
make fresh

# Setup test database
make setup-test-db

# Run tests
make test

# Setup project from scratch
make setup

# Show logs
make logs

# Enter app container
make shell

# Clear all caches
make clear

# Start Vite development server
make vite
```

### Benefits of Using Makefile

1. **Simplified Commands**: No need to remember long Docker commands.
2. **Consistent Workflow**: Ensures everyone on the team follows the same process.
3. **Documentation**: Serves as self-documentation for common operations.
4. **Automation**: Combines multiple commands into single operations.

### Example Workflow

```bash
# Initial setup
make setup

# Development workflow
make up
make shell  # Enter container for development
make vite   # Start Vite for frontend development

# Testing
make test

# Deployment preparation
make down
make build
```

## Docker and Linux Cheatsheet

### Docker Commands

| Command | Description |
|---------|-------------|
| `docker compose up -d` | Start containers in detached mode |
| `docker compose down` | Stop and remove containers |
| `docker compose build` | Build or rebuild services |
| `docker compose ps` | List containers |
| `docker compose logs -f` | Follow log output |
| `docker compose exec app bash` | Execute command in running container |
| `docker compose exec app php artisan migrate` | Run Laravel migration |
| `docker compose exec app composer install` | Install PHP dependencies |
| `docker compose exec app npm install` | Install Node.js dependencies |
| `docker compose exec app npm run dev` | Start Vite development server |
| `docker compose exec app php artisan test` | Run Laravel tests |
| `docker compose exec app php artisan cache:clear` | Clear Laravel cache |
| `docker compose exec app php artisan config:clear` | Clear Laravel config cache |
| `docker compose exec app php artisan route:clear` | Clear Laravel route cache |
| `docker compose exec app php artisan view:clear` | Clear Laravel view cache |
| `docker compose exec mysql mysql -uroot -psecret` | Access MySQL database |
| `docker compose exec redis redis-cli` | Access Redis CLI |
| `docker compose exec mailhog mailhog` | Access MailHog CLI |

### Linux Commands

| Command | Description |
|---------|-------------|
| `ls -la` | List all files including hidden ones |
| `cd /path/to/directory` | Change directory |
| `pwd` | Print working directory |
| `mkdir -p /path/to/directory` | Create directory and parent directories |
| `rm -rf /path/to/directory` | Remove directory and its contents |
| `cp source destination` | Copy file or directory |
| `mv source destination` | Move or rename file or directory |
| `cat file` | Display file contents |
| `head -n 10 file` | Display first 10 lines of file |
| `tail -f file` | Follow file output |
| `grep "pattern" file` | Search for pattern in file |
| `find /path -name "pattern"` | Find files matching pattern |
| `chmod 755 file` | Change file permissions |
| `chown user:group file` | Change file ownership |
| `sudo command` | Run command with superuser privileges |
| `ps aux | grep process` | Find running processes |
| `kill -9 PID` | Force kill process by PID |
| `df -h` | Display disk space usage |
| `du -sh /path` | Display directory size |
| `free -m` | Display memory usage |
| `top` | Display system processes |
| `htop` | Interactive process viewer |
| `netstat -tulpn` | Display network connections |
| `curl -I URL` | Display HTTP headers |
| `wget URL` | Download file from URL |
| `tar -czvf archive.tar.gz /path` | Create tar archive |
| `tar -xzvf archive.tar.gz` | Extract tar archive |
| `zip -r archive.zip /path` | Create zip archive |
| `unzip archive.zip` | Extract zip archive |
| `ssh user@host` | Connect to remote host via SSH |
| `scp file user@host:/path` | Copy file to remote host |
| `rsync -avz /source/ user@host:/destination/` | Synchronize directories |

## Suggested Improvements for Your Current Setup

Based on the analysis of your current Docker setup, here are some specific improvements:

1. **Implement Multi-Stage Builds**: Separate build dependencies from runtime dependencies.

2. **Optimize PHP-FPM Configuration**: Fine-tune your PHP-FPM settings for better performance.

3. **Remove Development Tools**: Ensure your production images don't include development tools.

4. **Implement Proper Caching**: Optimize your Dockerfile for better layer caching.

5. **Use Specific Versions**: Instead of using `latest` tags, use specific version tags for better reproducibility.

6. **Implement Health Checks**: Add health checks to your containers for better orchestration.

7. **Optimize Nginx Configuration**: Fine-tune your Nginx settings for better performance.

8. **Leverage Makefile**: Use your Makefile for all Docker operations to ensure consistency.

## Conclusion

Creating optimized Docker images for PHP applications is both an art and a science. By implementing the techniques discussed in this article, you can significantly reduce image size, improve build times, and enhance security.

In our next article, we'll explore how to implement Zero Downtime Deployment with GitHub Actions, building on the optimized Docker images we've created.

Remember, optimization is an ongoing process. As your application evolves, so should your Docker images. Regularly review and update your Dockerfile to ensure it remains efficient and secure.

---

*Follow me on [Twitter](your-twitter) and [LinkedIn](your-linkedin) for more DevOps and Laravel tips!*

*Would you like to learn more about any specific aspect of Docker image optimization? Leave a comment below!* 
