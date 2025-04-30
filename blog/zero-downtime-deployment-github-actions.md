# 🚀 Zero Downtime Deployment with GitHub Actions: A Complete Guide

*Published: [Current Date]*

In our previous articles, we explored how to set up a production-ready Docker environment for Laravel applications and how to optimize Docker images. Today, we'll take the next step and implement a robust Zero Downtime Deployment (ZDD) strategy using GitHub Actions. This approach ensures your application remains available to users even during deployments.

## 📋 Table of Contents
- [What is Zero Downtime Deployment?](#what-is-zero-downtime-deployment)
- [Why GitHub Actions for Deployment?](#why-github-actions-for-deployment)
- [Prerequisites](#prerequisites)
- [Server Setup](#server-setup)
- [GitHub Actions Workflow](#github-actions-workflow)
- [Deployment Strategy](#deployment-strategy)
- [Rollback Mechanism](#rollback-mechanism)
- [Monitoring and Logging](#monitoring-and-logging)
- [Security Considerations](#security-considerations)
- [Troubleshooting](#troubleshooting)
- [Conclusion](#conclusion)

## What is Zero Downtime Deployment?

Zero Downtime Deployment (ZDD) is a deployment strategy that ensures your application remains available to users throughout the entire deployment process. Unlike traditional deployments where the application is completely unavailable during updates, ZDD maintains service continuity by:

1. **Deploying to a new version** while the old version continues to run
2. **Testing the new version** before directing traffic to it
3. **Gradually shifting traffic** from the old version to the new version
4. **Maintaining the old version** as a fallback in case of issues

This approach minimizes the risk of service disruption and provides a seamless experience for your users.

## Why GitHub Actions for Deployment?

GitHub Actions offers several advantages for implementing ZDD:

1. **Integration with GitHub**: Seamless integration with your code repository
2. **Workflow Automation**: Define deployment workflows as code
3. **Environment Secrets**: Secure storage for sensitive deployment credentials
4. **Matrix Builds**: Test across multiple environments simultaneously
5. **Artifact Management**: Store and retrieve build artifacts between jobs
6. **Community Actions**: Leverage pre-built actions for common deployment tasks

## Prerequisites

Before implementing ZDD with GitHub Actions, ensure you have:

1. **A GitHub Repository**: Hosting your Laravel application
2. **A VPS Server**: Running Ubuntu/Debian (we'll use Ubuntu in this guide)
3. **SSH Access**: To your VPS server
4. **Domain Name**: Pointed to your server's IP address
5. **SSL Certificate**: For secure HTTPS connections (Let's Encrypt recommended)

## Server Setup

Let's set up your VPS server for ZDD. This setup is based on the `SERVER_SETUP.md` file in your project.

### 1. Create Deployment User

First, create a dedicated user for deployments:

```bash
# Create a new user
sudo adduser deployer
sudo usermod -aG www-data deployer

# Add permissions for deployer user
sudo echo "deployer ALL=(ALL) NOPASSWD:/usr/bin/chmod, /usr/bin/chown" | sudo tee /etc/sudoers.d/deployer
sudo echo "deployer ALL=(ALL) NOPASSWD:/usr/bin/systemctl restart php8.4-fpm" | sudo tee /etc/sudoers.d/deployer
sudo echo "deployer ALL=(ALL) NOPASSWD:/usr/bin/systemctl restart nginx" | sudo tee /etc/sudoers.d/deployer
sudo echo "deployer ALL=(ALL) NOPASSWD:/usr/bin/systemctl reload nginx" | sudo tee /etc/sudoers.d/deployer
```

### 2. Install Required Software

Install the necessary software packages:

```bash
# Update the system
apt update
apt install nginx php-fpm mariadb-server
sudo apt install php8.4-cli php8.4-common php8.4-curl php8.4-xml php8.4-mbstring php8.4-zip php8.4-mysql php8.4-gd php8.4-intl php8.4-bcmath php8.4-redis php8.4-imagick php8.4-pgsql php8.4-sqlite3 php8.4-tokenizer php8.4-dom php8.4-fileinfo php8.4-iconv php8.4-simplexml php8.4-opcache
sudo systemctl restart php8.4-fpm
```

### 3. Configure Nginx

Create a new Nginx configuration file:

```bash
sudo nano /etc/nginx/sites-available/laravel
```

Add the following configuration:

```nginx
server {
    listen 80;
    listen [::]:80;
    server_name your-domain.com;
    root /home/deployer/laravel/current/public;

    add_header X-Frame-Options "SAMEORIGIN";
    add_header X-Content-Type-Options "nosniff";
    add_header X-XSS-Protection "1; mode=block";
    add_header Referrer-Policy "strict-origin-when-cross-origin";
    add_header Strict-Transport-Security "max-age=31536000; includeSubDomains" always;
    add_header Permissions-Policy "geolocation=(), midi=(), sync-xhr=(), microphone=(), camera=(), magnetometer=(), gyroscope=(), fullscreen=(self), payment=()";
    server_tokens off;

    index index.php;
    charset utf-8;

    sendfile on;
    tcp_nopush on;
    tcp_nodelay on;
    keepalive_timeout 65;
    client_max_body_size 100M;

    location ~* \.(jpg|jpeg|png|gif|ico|css|js|svg|woff|woff2|ttf|eot|webp)$ {
        expires 1y;
        add_header Cache-Control "public, immutable";
        access_log off;
        log_not_found off;
        try_files $uri =404;
    }

    location / {
        try_files $uri $uri/ /index.php?$query_string;
    }

    location = /favicon.ico { access_log off; log_not_found off; }
    location = /robots.txt  { access_log off; log_not_found off; }

    error_page 404 /index.php;

    location ~ ^/index\.php(/|$) {
        fastcgi_pass unix:/var/run/php/php8.4-fpm.sock;
        fastcgi_param SCRIPT_FILENAME $realpath_root$fastcgi_script_name;
        include fastcgi_params;
        
        fastcgi_buffer_size 128k;
        fastcgi_buffers 4 256k;
        fastcgi_busy_buffers_size 256k;
        fastcgi_read_timeout 300;
        
        fastcgi_hide_header X-Powered-By;
    }

    location ~ /\.(?!well-known).* {
        deny all;
        access_log off;
        log_not_found off;
    }

    gzip on;
    gzip_vary on;
    gzip_proxied any;
    gzip_comp_level 6;
    gzip_types text/plain text/css text/xml application/json application/javascript application/xml+rss application/atom+xml image/svg+xml;
    gzip_min_length 1024;
    gzip_buffers 16 8k;
    gzip_disable "MSIE [1-6]\.";
}
```

Enable the site:

```bash
sudo ln -s /etc/nginx/sites-available/laravel /etc/nginx/sites-enabled/
sudo nginx -t
sudo systemctl restart nginx
```

### 4. Configure PHP-FPM

Update the PHP-FPM configuration:

```bash
sudo nano /etc/php/8.4/fpm/pool.d/www.conf
```

Add the following configuration:

```ini
[www]
user = deployer
group = deployer

listen = /var/run/php/php8.4-fpm.sock
listen.owner = www-data
listen.group = www-data
listen.mode = 0660

pm = dynamic
pm.max_children = 50
pm.start_servers = 5
pm.min_spare_servers = 5
pm.max_spare_servers = 10
pm.max_requests = 500

pm.process_idle_timeout = 10s
request_terminate_timeout = 30s
request_slowlog_timeout = 5s

slowlog = /var/log/php-fpm/slow.log

php_admin_value[error_log] = /var/log/php-fpm/error.log
php_admin_flag[log_errors] = on

php_admin_value[memory_limit] = 256M
php_admin_value[disable_functions] = "exec,passthru,shell_exec,system"
php_admin_value[open_basedir] = "/home/deployer/laravel/current/:/tmp/:/var/lib/php/sessions/"
```

### 5. Configure PHP

Update the PHP configuration:

```bash
sudo nano /etc/php/8.4/fpm/php.ini
```

Add the following configuration:

```ini
[PHP]
expose_php = Off
max_execution_time = 30
max_input_time = 60
memory_limit = 256M
error_reporting = E_ALL & ~E_DEPRECATED & ~E_STRICT
display_errors = Off
display_startup_errors = Off
log_errors = On
error_log = /var/log/php8.4-fpm.log

opcache.enable=1
opcache.memory_consumption=256
opcache.interned_strings_buffer=32
opcache.max_accelerated_files=20000
opcache.validate_timestamps=0
opcache.enable_cli=0
opcache.jit_buffer_size=256M
opcache.jit=1235

realpath_cache_size=4096K
realpath_cache_ttl=600

session.gc_probability=1
session.gc_divisor=100
session.gc_maxlifetime=1440
session.save_handler = redis
session.save_path = "tcp://127.0.0.1:6379"

upload_max_filesize = 64M
post_max_size = 64M
file_uploads = On

max_input_vars = 5000
request_order = "GP"
variables_order = "GPCS"

[Date]
date.timezone = Europe/Warsaw
```

### 6. Set Up Directory Structure

Create the directory structure for deployments:

```bash
# Create directory structure
sudo mkdir -p /home/deployer/laravel/{releases,shared}

# Set permissions
sudo chown -R deployer:www-data /home/deployer/laravel
sudo chmod -R 775 /home/deployer/laravel
sudo chmod g+s /home/deployer/laravel
```

### 7. Set Up SSL with Let's Encrypt

Install and configure SSL:

```bash
# Install Certbot
sudo apt install -y certbot python3-certbot-nginx

# Obtain SSL certificate
sudo certbot --nginx -d your-domain.com

# Set up auto-renewal
sudo systemctl status certbot.timer
```

## GitHub Actions Workflow

Now, let's create a GitHub Actions workflow for ZDD. Create a new file at `.github/workflows/deploy.yml` in your repository:

```yaml
name: Deploy Laravel Application

on:
  push:
    branches: [ main ]
  workflow_dispatch:

jobs:
  deploy:
    runs-on: ubuntu-latest
    
    steps:
      - name: Checkout code
        uses: actions/checkout@v3
      
      - name: Setup PHP
        uses: shivammathur/setup-php@v2
        with:
          php-version: '8.4'
          extensions: mbstring, dom, fileinfo, mysql, zip, pdo_mysql, bcmath, intl, gd, redis, imagick, pgsql, sqlite3, tokenizer, iconv, simplexml, opcache
          coverage: none
      
      - name: Copy .env
        run: php -r "file_exists('.env') || copy('.env.example', '.env');"
      
      - name: Install Dependencies
        run: composer install -q --no-ansi --no-interaction --no-scripts --no-progress --prefer-dist
      
      - name: Generate key
        run: php artisan key:generate
      
      - name: Directory Permissions
        run: chmod -R 777 storage bootstrap/cache
      
      - name: Create deployment archive
        run: |
          mkdir -p deployment
          cp -r app bootstrap config database lang public resources routes storage tests vendor composer.json composer.lock package.json package-lock.json artisan .env.example .gitattributes .gitignore phpunit.xml README.md deployment/
          tar -czf deployment.tar.gz deployment/
      
      - name: Deploy to server
        uses: appleboy/scp-action@master
        with:
          host: ${{ secrets.SSH_HOST }}
          username: ${{ secrets.SSH_USER }}
          key: ${{ secrets.SSH_KEY }}
          source: "deployment.tar.gz"
          target: "/home/deployer/laravel"
          strip_components: 0
      
      - name: Execute deployment commands
        uses: appleboy/ssh-action@master
        with:
          host: ${{ secrets.SSH_HOST }}
          username: ${{ secrets.SSH_USER }}
          key: ${{ secrets.SSH_KEY }}
          script: |
            cd /home/deployer/laravel
            tar -xzf deployment.tar.gz
            rm deployment.tar.gz
            
            # Create release directory
            RELEASE_DIR="/home/deployer/laravel/releases/$(date +%Y%m%d%H%M%S)"
            mkdir -p $RELEASE_DIR
            
            # Copy files to release directory
            cp -r deployment/* $RELEASE_DIR/
            
            # Create necessary directories
            mkdir -p $RELEASE_DIR/storage/framework/{sessions,views,cache}
            mkdir -p $RELEASE_DIR/storage/logs
            mkdir -p $RELEASE_DIR/bootstrap/cache
            
            # Set permissions
            chmod -R 775 $RELEASE_DIR/storage
            chmod -R 775 $RELEASE_DIR/bootstrap/cache
            
            # Create symlinks for shared resources
            ln -nfs /home/deployer/laravel/shared/.env $RELEASE_DIR/.env
            ln -nfs /home/deployer/laravel/shared/storage $RELEASE_DIR/storage
            
            # Run deployment commands
            cd $RELEASE_DIR
            composer install --no-dev --optimize-autoloader
            php artisan down --refresh=15
            php artisan migrate --force
            php artisan config:cache
            php artisan route:cache
            php artisan view:cache
            php artisan event:cache
            
            # Update symlink to current release
            ln -nfs $RELEASE_DIR /home/deployer/laravel/current
            
            # Restart PHP-FPM and reload Nginx
            sudo systemctl restart php8.4-fpm
            sudo systemctl reload nginx
            
            # Bring application back up
            php artisan up
            
            # Clean up old releases (keep last 5)
            cd /home/deployer/laravel/releases
            ls -t | tail -n +6 | xargs -r rm -rf
```

## Deployment Strategy

The ZDD strategy implemented in this workflow includes:

### 1. Blue-Green Deployment

We use a blue-green deployment approach where:

- The "blue" environment is the current production environment
- The "green" environment is the new deployment

The workflow:
1. Deploys the new code to a timestamped directory
2. Sets up the new environment
3. Tests the new environment
4. Switches traffic to the new environment
5. Keeps the old environment as a fallback

### 2. Symlink Switching

Instead of directly replacing files, we use symlinks to switch between versions:

```bash
ln -nfs $RELEASE_DIR /home/deployer/laravel/current
```

This approach:
- Is atomic (either succeeds completely or fails completely)
- Allows for instant rollbacks
- Minimizes downtime

### 3. Maintenance Mode

We use Laravel's maintenance mode during deployment:

```bash
php artisan down --refresh=15
# ... deployment steps ...
php artisan up
```

The `--refresh=15` parameter allows the server to refresh the maintenance page every 15 seconds, checking if the application is back online.

### 4. Shared Resources

We maintain shared resources between deployments:

```bash
ln -nfs /home/deployer/laravel/shared/.env $RELEASE_DIR/.env
ln -nfs /home/deployer/laravel/shared/storage $RELEASE_DIR/storage
```

This ensures:
- Environment variables remain consistent
- User uploads and logs are preserved
- Cache and sessions are maintained

## Rollback Mechanism

In case of deployment issues, you can quickly rollback to the previous version:

```bash
# SSH into your server
ssh deployer@your-server

# List available releases
ls -la /home/deployer/laravel/releases

# Rollback to a specific release
RELEASE_DIR="/home/deployer/laravel/releases/20230101120000"
ln -nfs $RELEASE_DIR /home/deployer/laravel/current

# Restart services
sudo systemctl restart php8.4-fpm
sudo systemctl reload nginx
```

## Monitoring and Logging

To ensure your ZDD is working correctly, implement proper monitoring:

### 1. Application Logs

Monitor Laravel logs:

```bash
tail -f /home/deployer/laravel/current/storage/logs/laravel.log
```

### 2. Nginx Logs

Monitor Nginx access and error logs:

```bash
tail -f /var/log/nginx/access.log
tail -f /var/log/nginx/error.log
```

### 3. PHP-FPM Logs

Monitor PHP-FPM logs:

```bash
tail -f /var/log/php-fpm/error.log
tail -f /var/log/php-fpm/slow.log
```

### 4. Health Checks

Implement a health check endpoint in your Laravel application:

```php
// routes/web.php
Route::get('/health', function () {
    return response()->json(['status' => 'ok']);
});
```

Then monitor this endpoint during deployments:

```bash
curl -I https://your-domain.com/health
```

## Security Considerations

### 1. SSH Key Management

Store your SSH keys securely in GitHub Secrets:

```bash
# Generate SSH key
ssh-keygen -t rsa -b 4096 -C "github-actions-deploy"

# Add to GitHub Secrets
# SSH_HOST: Your server IP
# SSH_USER: deployer
# SSH_KEY: The private key content
```

### 2. Environment Variables

Keep sensitive information in GitHub Secrets:

```bash
# Add to GitHub Secrets
# ENV_FILE: The contents of your .env file
```

### 3. File Permissions

Ensure proper file permissions:

```bash
chmod -R 775 /home/deployer/laravel/current/storage
chmod -R 775 /home/deployer/laravel/current/bootstrap/cache
```

### 4. Firewall Configuration

Configure your server's firewall:

```bash
sudo ufw allow 22/tcp
sudo ufw allow 80/tcp
sudo ufw allow 443/tcp
sudo ufw enable
```

## Troubleshooting

### Common Issues and Solutions

1. **Permission Denied Errors**
   - Check file ownership: `sudo chown -R deployer:www-data /home/deployer/laravel`
   - Check file permissions: `sudo chmod -R 775 /home/deployer/laravel`

2. **Database Migration Failures**
   - Check database credentials in `.env`
   - Run migrations manually: `php artisan migrate --force`

3. **Nginx Configuration Issues**
   - Test Nginx configuration: `sudo nginx -t`
   - Check Nginx logs: `sudo tail -f /var/log/nginx/error.log`

4. **PHP-FPM Issues**
   - Check PHP-FPM status: `sudo systemctl status php8.4-fpm`
   - Check PHP-FPM logs: `sudo tail -f /var/log/php-fpm/error.log`

5. **Symlink Problems**
   - Ensure symlinks are created correctly: `ls -la /home/deployer/laravel/current`
   - Recreate symlinks manually if needed

### Debugging Deployment

To debug deployment issues:

1. **Enable Verbose Output**
   - Add `-v` to composer commands
   - Add `--verbose` to artisan commands

2. **Check GitHub Actions Logs**
   - Review the complete workflow logs in GitHub
   - Look for specific error messages

3. **SSH into Server During Deployment**
   - Add a step to your workflow to keep the connection open
   - Inspect the server state during deployment

## Conclusion

Zero Downtime Deployment with GitHub Actions provides a robust, automated way to deploy your Laravel applications without service interruption. By implementing the strategies outlined in this article, you can ensure your application remains available to users throughout the deployment process.

In our next article, we'll explore how to implement similar ZDD strategies using GitLab CI/CD, providing an alternative approach for teams using GitLab as their primary platform.

Remember, ZDD is not just about the tools—it's about the process. Regularly test your deployment workflow, monitor your application's performance, and be prepared to roll back if necessary.

---

*Follow me on [Twitter](your-twitter) and [LinkedIn](your-linkedin) for more DevOps and Laravel tips!*

*Would you like to learn more about any specific aspect of Zero Downtime Deployment? Leave a comment below!* 