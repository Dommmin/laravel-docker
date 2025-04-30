# 🚀 Mastering Docker for Laravel: A Beginner's Guide to Setup

Are you tired of the "it works on my machine" syndrome? Looking for a way to streamline your Laravel development environment? Docker is the answer, and this guide will walk you through setting up a powerful, consistent development environment for your Laravel projects.

## 📋 Table of Contents
- [Introduction to Docker](#introduction-to-docker)
- [Why Docker for Laravel?](#why-docker-for-laravel)
- [Getting Started](#getting-started)
- [Project Structure](#project-structure)
- [Docker Configuration](#docker-configuration)
- [Performance Optimizations](#performance-optimizations)
- [Using Makefile for Simplicity](#using-makefile-for-simplicity)
- [Development Tools](#development-tools)
- [Troubleshooting](#troubleshooting)
- [Best Practices](#best-practices)
- [Conclusion](#conclusion)
- [Source Code](#source-code)
- [Docker & Laravel Glossary](#docker--laravel-glossary)

## Introduction to Docker

Before diving into the Laravel setup, let's understand what Docker is and why it's revolutionary for development.

Docker is a platform that enables developers to package applications and their dependencies into standardized units called **containers**. These containers are lightweight, standalone, and executable packages that include everything needed to run the application: code, runtime, system tools, libraries, and settings.

**Key Docker concepts:**
- **Container**: A lightweight, standalone, executable package that includes everything needed to run a piece of software
- **Image**: A blueprint for a container (like a class in programming)
- **Dockerfile**: A script containing instructions to build a Docker image
- **Docker Compose**: A tool for defining and running multi-container Docker applications
- **Volume**: A persistent data storage mechanism that exists outside containers
- **Network**: A communication system that allows containers to talk to each other

## Why Docker for Laravel?

Docker has revolutionized how we develop and deploy applications. For Laravel projects, it provides:

- **Consistent environments**: No more "it works on my machine" problems
- **Easy onboarding**: New team members can start developing in minutes, not days
- **Production-like environment**: Develop in an environment that mirrors production
- **Isolated services**: Keep your PHP, Nginx, MySQL, and Redis services separated and easily manageable
- **Version control for environments**: Track changes to your environment alongside your code
- **No local dependency conflicts**: Each project can use different versions of PHP, MySQL, etc. without conflicts

## Getting Started

### Creating a New Laravel Project with Docker

You can either create a new Laravel project or use Docker with an existing one. For a new project:

```bash
# Clone the example repository
git clone https://github.com/Dommmin/laravel-docker.git my-project
cd my-project
```

## Project Structure

Our Docker setup follows a modular approach, keeping Docker configuration separate from application code:

```
project/
├── docker/                  # Docker configuration files
│   ├── php/                 # PHP configuration
│   │   ├── Dockerfile       # Instructions to build PHP image
│   │   ├── php.ini          # PHP settings
│   │   └── www.conf         # PHP-FPM settings
│   ├── nginx/               # Nginx configuration
│   │   └── conf.d/          # Server blocks
│   └── supervisord.conf     # Process manager config
├── docker-compose.yml       # Multi-container definition
├── Makefile                 # Helper commands
└── .env                     # Environment variables
```

## Docker Configuration

### Base PHP Image

We're using a custom PHP 8.4 FPM image hosted on Docker Hub (`dommin/php-8.4-fpm`). This image comes pre-configured with:
- PHP-FPM optimization
- Common PHP extensions
- Composer
- Node.js and npm
- Git

> **Note**: In the next article, we'll explore how to create and optimize your own Docker images for PHP applications, including using Alpine-based images for production to significantly reduce image size.

### Dockerfile Explained

Below is our `Dockerfile` with comments explaining each line:

```Dockerfile
# Use our custom PHP 8.4 FPM image as the base
FROM dommin/php-8.4-fpm:latest

# Copy our custom PHP and PHP-FPM configuration files
COPY docker/php/php.ini /usr/local/etc/php/conf.d/custom.ini
COPY docker/php/www.conf /usr/local/etc/php-fpm.d/www.conf
COPY docker/start.sh /usr/local/bin/start.sh
COPY ./docker/supervisord.conf /etc/supervisor/supervisord.conf

# Switch to root user to perform privileged operations
USER root

# Allow customizing user/group ID for better host filesystem compatibility
ARG USER_ID=1000
ARG GROUP_ID=1000

# Update the www-data user to match your host user's UID/GID
RUN usermod -u ${USER_ID} www-data && groupmod -g ${GROUP_ID} www-data

# Make the startup script executable
RUN chmod +x /usr/local/bin/start.sh

# Create necessary directories for Supervisor
RUN mkdir -p /var/log/supervisor /var/run/supervisor

# Set proper ownership for directories
RUN chown -R www-data:www-data /var/www /var/log/supervisor /var/run/supervisor

# Create log directory for Supervisor
RUN mkdir -p /var/log/supervisor
RUN chown -R www-data:www-data /var/log/supervisor

# Switch back to non-root user for security
USER www-data
```

### Docker Compose Services Explained

Our `docker-compose.yml` creates a complete development environment with multiple interconnected services:

```yml
services:
  # PHP Application Container
  app:
    build:
      context: .  # Use the current directory as build context
      dockerfile: docker/php/Dockerfile
      args:
        # Pass host user/group IDs to avoid permission issues
        - USER_ID=${USER_ID:-1000}
        - GROUP_ID=${GROUP_ID:-1000}
    container_name: ${COMPOSE_PROJECT_NAME}_app  # Dynamic container name based on project
    command: ["sh", "-c", "/usr/local/bin/start.sh"]  # Execute start script on container startup
    restart: unless-stopped  # Auto restart container if it crashes
    working_dir: /var/www  # Set working directory inside container
    volumes:
        - ./:/var/www  # Mount project directory into container
        - ./docker/php/php.ini:/usr/local/etc/php/conf.d/custom.ini  # Custom PHP config
        - ./docker/php/www.conf:/usr/local/etc/php-fpm.d/www.conf  # Custom PHP-FPM config
        - ./docker/supervisord.conf:/etc/supervisor/supervisord.conf  # Supervisor config
    networks:
      - laravel-network  # Attach to our custom network
    ports:
        - "5173:5173"  # Vite development server port
        - "9000:9000"  # PHP-FPM port
    depends_on:  # Wait for these services to start first
        -   mysql
        -   redis

  # Nginx Web Server
  nginx:
    image: nginx:alpine  # Use lightweight Alpine-based image
    container_name: ${COMPOSE_PROJECT_NAME}_nginx
    restart: unless-stopped
    ports:
      - "80:80"  # Map host port 80 to container port 80
    volumes:
      - ./:/var/www  # Mount project directory
      - ./docker/nginx/conf.d:/etc/nginx/conf.d  # Custom Nginx config
      - ./docker/nginx/log:/var/log/nginx  # Nginx logs
    depends_on:
        -   app  # Wait for app container to start
    networks:
      - laravel-network

  # MySQL Database
  mysql:
    image: mysql:8.0
    container_name: ${COMPOSE_PROJECT_NAME}_mysql
    restart: unless-stopped
    environment:  # Set environment variables
        MYSQL_ROOT_PASSWORD: ${DB_PASSWORD}  # Use value from .env file
        MYSQL_DATABASE: ${DB_DATABASE}
        MYSQL_DATABASE_TEST: laravel_test
    command:  # Custom MySQL settings
        - --character-set-server=utf8mb4
        - --collation-server=utf8mb4_unicode_ci
    healthcheck:  # Regular health checks
        test: [ "CMD", "mysqladmin", "ping", "-h", "localhost" ]
        interval: 10s
        timeout: 5s
        retries: 5
        start_period: 60s
    ports:
        - "3306:3306"
    volumes:
        - mysql_data:/var/lib/mysql  # Persistent data storage
    networks:
        - laravel-network

  # Redis for Cache and Queue
  redis:
    image: redis:alpine
    container_name: ${COMPOSE_PROJECT_NAME}_redis
    restart: unless-stopped
    healthcheck:
        test: [ "CMD", "redis-cli", "ping" ]
        interval: 10s
        timeout: 5s
        retries: 5
        start_period: 60s
    ports:
      - "6379:6379"
    networks:
      - laravel-network

  # Email Testing Service
  mailhog:
      image: mailhog/mailhog:latest
      container_name: ${COMPOSE_PROJECT_NAME}_mailhog
      restart: unless-stopped
      ports:
          - "1025:1025"  # SMTP port
          - "8025:8025"  # Web interface port
      volumes:
          - mailhog_data:/maildir  # Persistent storage
      networks:
          - laravel-network

# Define our custom network
networks:
  laravel-network:
    driver: bridge  # Use bridge network driver

# Define persistent volumes
volumes:
  mysql_data:  # Store MySQL data
    driver: local
  mailhog_data:  # Store MailHog data
    driver: local
```

## Performance Optimizations

### PHP-FPM Configuration
- **Process Manager Settings**: Optimized `pm` settings for better resource usage
- **Worker Configuration**: Proper settings for number of workers and requests per worker
- **Memory Limits**: Adjusted for development needs

### Supervisor Configuration for Queue Workers
```ini
[supervisord]
nodaemon=true  # Run in foreground
logfile=/var/log/supervisor/supervisord.log
pidfile=/var/run/supervisor/supervisord.pid
user=www-data  # Run as non-root user

[program:horizon]  # Laravel Horizon queue manager
process_name=%(program_name)s
command=php /var/www/artisan horizon
autostart=true  # Start automatically
autorestart=true  # Restart if crashes
redirect_stderr=true
stdout_logfile=/var/log/supervisor/horizon.log
stopwaitsecs=3600  # Wait for jobs to complete on stop
user=www-data
```

### Nginx Optimization
- **Gzip Compression**: Reduces response size
- **FastCGI Caching**: Improves performance for repeated requests
- **Static File Serving**: Optimized for assets
- **Security Headers**: Adds protection against common vulnerabilities

## Using Makefile for Simplicity

One of the most powerful additions to our Docker setup is the `Makefile`. This file provides simple commands that abstract complex Docker operations, making it easier for developers to work with the environment.

### What is a Makefile?

A Makefile is a configuration file used by the `make` utility. It defines a set of tasks to be executed when you run the `make` command with a specific target.

### Benefits of Using Make with Docker

- **Simplified Commands**: Instead of typing long docker-compose commands, use short aliases
- **Standardized Workflows**: Everyone on the team uses the same commands
- **Documentation**: The Makefile itself serves as documentation for common operations
- **Automation**: Chain multiple commands together in a single make target

### Our Docker Makefile

```makefile
.PHONY: up down build install migrate fresh test setup-test-db

# Start the application
up:
	docker compose up -d

# Stop the application
down:
	docker compose down

# Build containers
build:
	@echo "Setting up the project..."
	@if [ ! -f .env ]; then \
		cp .env.local .env; \
		echo "Created .env file from .env.local"; \
	fi
	docker compose build

# Install dependencies
install:
	docker compose exec app composer install
	docker compose exec app npm install

# Run migrations
migrate:
	docker compose exec app php artisan migrate

# Fresh migrations
fresh:
	docker compose exec app php artisan migrate:fresh

# Setup test database
setup-test-db:
	docker compose exec mysql mysql -uroot -psecret -e "CREATE DATABASE IF NOT EXISTS laravel_test;"
	docker compose exec app php artisan migrate --env=testing

# Run tests
test: setup-test-db
	docker compose exec app php artisan test --env=testing

# Setup project from scratch
setup: build up
	docker compose exec app composer install
	docker compose exec app npm install
	docker compose exec app php artisan key:generate
	docker compose exec app php artisan migrate
	@echo "Project setup completed!"

# Show logs
logs:
	docker compose logs -f

# Enter app container
shell:
	docker compose exec app bash

# Clear all caches
clear:
	docker compose exec app php artisan cache:clear
	docker compose exec app php artisan config:clear
	docker compose exec app php artisan route:clear
	docker compose exec app php artisan view:clear

# Start Vite development server
vite:
	docker compose exec app npm run dev
```

### Using the Makefile

With our Makefile, setting up a fresh project is as simple as:

```bash
# Set up the entire project with one command
make setup

# Start the development environment
make up

# Run database migrations
make migrate

# Access the app container shell
make shell

# Start the Vite development server
make vite
```

This drastically simplifies developer onboarding. New team members can get a fully functioning environment with just a few commands!

## Development Tools

### Testing URLs
- **Main application**: `http://localhost`
- **MailHog interface**: `http://localhost:8025` (for email testing)
- **Queue monitoring**: `http://localhost/horizon` (if Horizon is installed)

### Workflow Example

Here's a typical workflow using our Docker setup:

1. **Clone the project and start containers**:
   ```bash
   git clone https://github.com/your-username/your-project.git
   cd your-project
   make setup  # Builds containers, installs dependencies, runs migrations
   ```

2. **Make code changes**: Edit your Laravel files as usual in your IDE

3. **Run migrations after database changes**:
   ```bash
   make migrate
   ```

4. **Start frontend asset compilation**:
   ```bash
   make vite
   ```

5. **Test emails using MailHog**:
   - Trigger an email in your application
   - View it at `http://localhost:8025`

6. **Run tests**:
   ```bash
   make test
   ```

7. **Shut down when done**:
   ```bash
   make down
   ```

### Useful Commands
```bash
# View logs in real-time
make logs

# Clear all Laravel caches
make clear

# Get a shell in the app container
make shell

# Completely rebuild the containers
make build
```

## Troubleshooting

### Common Issues and Solutions

#### 1. Permission Denied Errors
```
Error: EACCES: permission denied, open '/var/www/storage/logs/laravel.log'
```

**Solution**: Fix permissions and update USER_ID/GROUP_ID in your .env file
```bash
# Add to your .env file
USER_ID=$(id -u)
GROUP_ID=$(id -g)

# Then rebuild
make build
```

#### 2. Port Already in Use
```
Error starting userland proxy: listen tcp4 0.0.0.0:80: bind: address already in use
```

**Solution**: Change the port in docker-compose.yml or stop the service using port 80
```yaml
# In docker-compose.yml
ports:
  - "8080:80"  # Change 80 to 8080
```

#### 3. MySQL Connection Issues
```
SQLSTATE[HY000] [2002] Connection refused
```

**Solution**: Check your .env file settings
```
DB_CONNECTION=mysql
DB_HOST=mysql  # Must match service name in docker-compose.yml
DB_PORT=3306
DB_DATABASE=laravel
DB_USERNAME=root
DB_PASSWORD=your_password
```

#### 4. Missing Node Modules
If you're getting errors about missing Node modules:

**Solution**: Reinstall Node modules
```bash
make install
```

#### 5. Docker Containers Not Starting
If containers fail to start, check the logs:

**Solution**:
```bash
docker compose logs
```

## Best Practices

1. **Security**
    - **Non-root user** in containers
    - **Proper file permissions**
    - **Environment variable management**
    - **Regular security updates**

2. **Performance**
    - **Volume mounting optimization**
    - **Cache configuration**
    - **Database optimization**
    - **Queue worker management**

3. **Development Experience**
    - **Hot reload for frontend**
    - **Debug tools integration**
    - **Easy access to logs**
    - **Simple command execution**

## Conclusion

A well-configured Docker environment is crucial for modern Laravel development. This setup provides a solid foundation for building an efficient, secure, and maintainable development environment. The combination of Docker and the Makefile creates a powerful, yet easy-to-use system that will boost your productivity.

In our next article, we'll cover optimizing Docker images for production, where we'll explore Alpine-based images and other techniques to create lightweight, secure containers ready for deployment.

## Source Code

You can find the complete code for this article on [GitHub](https://github.com/Dommmin/laravel-docker). Feel free to clone it and use it as a starting point for your own projects.

## Docker & Laravel Glossary

- **Container**: A lightweight, standalone, executable software package
- **Image**: A template used to create containers
- **Docker Compose**: A tool for defining and running multi-container applications
- **Volume**: Persistent storage that exists outside containers
- **Dockerfile**: A script with instructions to build a Docker image
- **Laravel Horizon**: A queue monitoring dashboard for Laravel Redis queues
- **MailHog**: A development tool for email testing
- **PHP-FPM**: PHP FastCGI Process Manager for handling PHP requests
- **Supervisor**: A process control system to manage processes
- **Makefile**: A file containing a set of directives used by the make build automation tool

---

*Follow me on [LinkedIn](https://www.linkedin.com/in/dominik-jasi%C5%84ski/) for more Laravel and DevOps content!*

*Would you like to learn more about Docker with Laravel? Let me know in the comments what topics you'd like to see covered next!*

