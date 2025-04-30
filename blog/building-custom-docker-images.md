# 🔧 Building and Publishing Custom Docker Images for PHP Applications

Today, we'll dive into the process of creating custom Docker images, publishing them to Docker Hub, and using them in your projects and CI/CD pipelines.

## 📋 Table of Contents
- [Why Create Custom Docker Images?](#why-create-custom-docker-images)
- [Planning Your Custom PHP Image](#planning-your-custom-php-image)
- [Building Your Custom PHP Image](#building-your-custom-php-image)
- [Publishing to Docker Hub](#publishing-to-docker-hub)
- [Automating Builds with GitHub Actions](#automating-builds-with-github-actions)
- [Conclusion](#conclusion)

## Why Create Custom Docker Images?

Creating custom Docker images offers several significant advantages:

1. **Standardization**: Ensure all developers and environments use identical PHP configurations.
2. **Efficiency**: Reduce build times by pre-installing common dependencies.
3. **Compliance**: Enforce security and compliance requirements across all environments.
4. **Specialization**: Tailor images to your specific application needs.
5. **Control**: Manage PHP versions and extensions independently of application code.
6. **Scalability**: Simplify scaling by standardizing your deployment units.

In our previous articles, we've been using a custom PHP image (`dommin/php-8.4-fpm`) that comes pre-configured with PHP 8.4, common extensions, and tools like Node.js and Composer. Let's explore how this image was created and how you can build your own.

## Planning Your Custom PHP Image

Before building a custom image, define your requirements:

### Key Considerations:

1. **Base Image**: Choose right image Debian-based (`php:8.4-fpm`).
2. **PHP Extensions**: List the PHP extensions your applications commonly need.
3. **System Packages**: Identify system dependencies for your PHP extensions and tools.
4. **Additional Tools**: Decide if you need tools like Composer, Node.js, or Git.
5. **User Permissions**: Consider how to handle file permissions between host and container.
6. **Size vs. Functionality**: Balance image size against needed functionality.

### Requirements Example:

For a Laravel application, you might need:
- PHP 8.4 with FPM
- Extensions: PDO, MySQL, PostgreSQL, GD, Zip, BCMath, Redis, etc.
- Composer
- Node.js and NPM
- Git for Composer dependencies

## Building Your Custom PHP Image

Let's analyze and improve the Dockerfile used to build our custom PHP 8.4 image:

```dockerfile
FROM php:8.4-fpm

ARG USER_ID=1000
ARG GROUP_ID=1000

RUN apt-get update && apt-get install -y \
    git curl unzip zip \
    libpng-dev libjpeg-dev libwebp-dev libfreetype6-dev \
    libonig-dev libxml2-dev libzip-dev imagemagick libmagick++-dev \
    libpq-dev supervisor \
    && curl -fsSL https://deb.nodesource.com/setup_22.x | bash - \
    && apt-get install -y nodejs \
    && npm install -g npm@latest \
    && pecl install redis imagick \
    && docker-php-ext-enable redis imagick \
    && docker-php-ext-configure gd --with-freetype --with-jpeg --with-webp \
    && docker-php-ext-install -j$(nproc) pdo_mysql pdo_pgsql mysqli zip bcmath pcntl intl gd opcache \
    && apt-get clean && rm -rf /var/lib/apt/lists/* /tmp/* /var/tmp/*

COPY --from=composer:latest /usr/bin/composer /usr/bin/composer

RUN groupmod -g ${GROUP_ID} www-data && \
    usermod -u ${USER_ID} -g www-data www-data && \
    chown -R www-data:www-data /var/www

WORKDIR /var/www
USER www-data

EXPOSE 9000
```

### Key Components Explained:

1. **Base Image**: Uses the official PHP 8.4 FPM image.
2. **User Customization**: Allows customizing user/group IDs to match host permissions.
3. **System Dependencies**: Installs libraries required for PHP extensions.
4. **Node.js**: Installs Node.js 22.x and npm for frontend asset building.
5. **PHP Extensions**: Installs and enables common PHP extensions for Laravel.
6. **Composer**: Copies Composer from the official Composer image.
7. **Permissions**: Adjusts user permissions and sets the working directory.
8. **Port Exposure**: Exposes port 9000 for PHP-FPM.

## Publishing to Docker Hub

Docker Hub is the largest repository of container images. Publishing your image there makes it available to your team, CI/CD pipelines, and potentially the broader community.

### 1. Create a Docker Hub Account

If you haven't already, sign up for a [Docker Hub account](https://hub.docker.com/signup).

### 2. Create a Repository

1. Click "Create Repository" on your Docker Hub dashboard
2. Name it appropriately (e.g., `username/php`)
3. Add a description and set visibility (public or private)
4. Click "Create"

### 3. Log in to Docker Hub from CLI

```bash
docker login
```

Enter your Docker Hub username and password when prompted.

### 4. Build Your Image

```bash
docker build -t username/php:8.4-fpm -f ./Dockerfile .
```

### 5. Push Your Image to Docker Hub

```bash
docker push username/php:8.4-fpm
```

### 6. Verify Your Upload

Visit your repository on Docker Hub to ensure your image was uploaded successfully.

## Automating Builds with GitHub Actions

Manually building and pushing images can be tedious. Let's automate this with GitHub Actions.

Here's the workflow file you're using (make sure your secrets are set up correctly):

```yaml
name: Build images

on:
  push:
    branches:
      - main

env:
  PHP_IMAGE: dommin/php-8.4-fpm:latest

jobs:
  build:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: docker/login-action@v3
        with:
          username: ${{ secrets.DOCKER_USERNAME }}
          password: ${{ secrets.DOCKER_PASSWORD }}
      - name: Build images
        run: docker build -t $PHP_IMAGE -f ./8.4-fpm/Dockerfile .
      - name: Push images
        run: docker push $PHP_IMAGE
```

## Conclusion

Building custom Docker images for your PHP applications offers significant advantages in terms of standardization, efficiency, and control. By automating the build and publication process with GitHub Actions, you can ensure your images are always up-to-date and readily available for your projects and CI/CD pipelines.

Custom images are a powerful tool in your DevOps arsenal, enabling you to standardize your PHP environment across development, testing, and production while maintaining the flexibility to adapt to your specific application requirements.

---

*Follow me on [LinkedIn](https://www.linkedin.com/in/dominik-jasi%C5%84ski/) for more DevOps and Laravel tips!*

*Would you like to learn more about Docker, PHP, or CI/CD? Let me know in the comments below!* 
