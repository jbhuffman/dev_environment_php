# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

This is a containerized PHP web application development environment using Docker Compose. It provides a complete infrastructure stack for running PHP applications with MySQL database support.

## Technology Stack

- **PHP 8.3** (FPM - FastCGI Process Manager)
- **Nginx 1.25** - Web server
- **MySQL 8.0** - Database
- **Caddy 2** - Reverse proxy with automatic HTTPS
- **phpMyAdmin 5** - Database administration

## Common Commands

### Starting the Environment
```bash
docker-compose -f docker_compose.yml up -d    # Start all services
docker-compose -f docker_compose.yml down     # Stop all services
docker-compose -f docker_compose.yml build    # Rebuild PHP image after Dockerfile changes
```

### Viewing Logs
```bash
docker-compose -f docker_compose.yml logs -f           # All services
docker-compose -f docker_compose.yml logs -f app_php   # PHP logs only
docker-compose -f docker_compose.yml logs -f app_nginx # Nginx logs only
```

### Accessing Containers
```bash
docker exec -it app_php sh       # Shell into PHP container
docker exec -it app_mysql bash   # Shell into MySQL container
```

### Composer (run inside PHP container)
```bash
docker exec -it app_php composer install
docker exec -it app_php composer require <package>
```

## Architecture

### Container Services
- **app_caddy**: Entry point (ports 80/443), handles SSL/TLS termination, routes `/pma` to phpMyAdmin, everything else to Nginx
- **app_nginx**: Serves PHP application, routes all requests to `/index.php`
- **app_php**: PHP-FPM runtime on port 9000
- **app_mysql**: Database on port 3306
- **app_phpmyadmin**: Database admin (accessible at `/pma` with basic auth)

### Directory Mapping
- `./src` → `/var/www/html` in Nginx/PHP containers (application source code)
- `./src/public` → Web root

### Network
All services communicate through internal `appnet` bridge network. Only Caddy exposes ports 80/443 externally.

## Configuration Files

- `.env` - Database credentials and ACME email
- `docker/caddy/Caddyfile` - Reverse proxy routing and SSL
- `docker/nginx/sites/default.conf` - Nginx server configuration
- `docker/php/php.ini` - PHP settings (512MB memory, 100MB uploads, 300s max execution)
- `docker/php/Dockerfile` - PHP image with extensions (PDO, Intl, ImageMagick, etc.)

## PHP Configuration Highlights

- Memory limit: 512MB
- Upload max: 100MB
- Max execution time: 300 seconds
- Timezone: America/New_York
- Opcache enabled for production
