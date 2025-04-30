# VPS Server Setup Guide

This guide will help you set up your VPS for Laravel deployment with zero downtime.
## 0. Create User (optional)

```bash
# Create a new user
sudo adduser deployer
sudo usermod -aG www-data deployer

# Add permissions for deployer user
sudo echo "deployer ALL=(ALL) NOPASSWD:/usr/bin/chmod, /usr/bin/chown" | sudo tee /etc/sudoers.d/deployer
sudo echo "deployer ALL=(ALL) NOPASSWD:/usr/bin/systemctl restart php8.3-fpm" | sudo tee /etc/sudoers.d/deployer
sudo echo "deployer ALL=(ALL) NOPASSWD:/usr/bin/systemctl restart nginx" | sudo tee /etc/sudoers.d/deployer
sudo echo "deployer ALL=(ALL) NOPASSWD:/usr/bin/systemctl reload nginx" | sudo tee /etc/sudoers.d/deployer
```

## 1. Initial Server Setup

```bash
# Update the system
apt update
apt install nginx php-fpm mariadb-server
sudo apt install php8.3-cli php8.3-common php8.3-curl php8.3-xml php8.3-mbstring php8.3-zip php8.3-mysql php8.3-gd php8.3-intl php8.3-bcmath php8.3-redis php8.3-imagick php8.3-pgsql php8.3-sqlite3 php8.3-tokenizer php8.3-dom php8.3-fileinfo php8.3-iconv php8.3-simplexml php8.3-opcache
sudo systemctl restart php8.3-fpm
```

## 2. Configure Nginx

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
        fastcgi_pass unix:/var/run/php/php8.3-fpm.sock;
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

## 3. Update PHP-FPM Configuration

```bash
# nano /etc/php/8.3/fpm/pool.d/www.conf
[www]
user = deployer
group = deployer

listen = /var/run/php/php8.3-fpm.sock
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

## 4. Update PHP Configuration

```bash
# nano /etc/php/8.3/fpm/php.ini
[PHP]
expose_php = Off
max_execution_time = 30
max_input_time = 60
memory_limit = 256M
error_reporting = E_ALL & ~E_DEPRECATED & ~E_STRICT
display_errors = Off
display_startup_errors = Off
log_errors = On
error_log = /var/log/php8.3-fpm.log

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

## 5. Set Up Directory Structure

```bash
# Create directory structure
sudo mkdir -p /home/deployer/laravel/{releases,shared}

# Set permissions
sudo chown -R deployer:www-data /home/deployer/laravel
sudo chmod -R 775 /home/deployer/laravel
sudo chmod g+s /home/deployer/laravel
```

## 6. Set Up SSH Key for GitHub Actions (as deployer user)

```bash
# Create SSH directory
mkdir -p ~/.ssh
chmod 700 ~/.ssh

# Generate SSH key
ssh-keygen -t rsa -b 4096 -C "github-actions-deploy"

# Add public key to authorized_keys
cat ~/.ssh/id_rsa.pub >> ~/.ssh/authorized_keys
chmod 600 ~/.ssh/authorized_keys

# Display the private key
cat ~/.ssh/id_rsa
```

## 7. Add GitHub Secrets

Add the following secrets to your GitHub repository:

- `SSH_HOST`: Your VPS IP address or domain
- `SSH_USER`: Your VPS username
- `SSH_KEY`: The private SSH key generated above

Add variable for .env production file
- `ENV_FILE`: The contents of your .env file

## 8. Set Up SSL with Let's Encrypt (Optional but Recommended)

```bash
# Install Certbot
sudo apt install -y certbot python3-certbot-nginx

# Obtain SSL certificate
sudo certbot --nginx -d your-domain.com

# Set up auto-renewal
sudo systemctl status certbot.timer
```

## 9. Final Steps

1. Push your code to the `main` branch to trigger the deployment.
2. Monitor the GitHub Actions workflow to ensure it completes successfully.
3. Check your website to verify the deployment.

## Troubleshooting

- **Permission Issues**: Ensure all directories have the correct ownership and permissions.
- **Nginx Errors**: Check the Nginx error logs with `sudo tail -f /var/log/nginx/error.log`.
- **PHP-FPM Errors**: Check the PHP-FPM error logs with `sudo tail -f /var/log/php8.3-fpm.log`.
- **Deployment Failures**: Check the GitHub Actions logs for detailed error messages.

## Conclusion
