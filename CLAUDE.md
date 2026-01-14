# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

This is a production-ready containerized PHP Laravel 12 web application development environment using Docker Compose. It provides a complete infrastructure stack with automatic HTTPS, health monitoring, and MySQL database support.

## Technology Stack

- **PHP 8.4** (FPM - FastCGI Process Manager) with Composer 2
- **Laravel 12** - PHP Framework with Breeze authentication
- **Nginx 1.27** - Web server
- **MySQL 8.4** - Database (LTS release)
- **Caddy 2** - Reverse proxy with automatic HTTPS via Let's Encrypt
- **phpMyAdmin 5** - Database administration interface

## Documentation

- **DEVELOPMENT.md** - Complete local development setup and workflows
- **DEPLOYMENT.md** - Production deployment guide with security best practices
- **README.md** - Project overview and quick start

## Project Structure

```
.
├── docker/
│   ├── caddy/
│   │   ├── Caddyfile           # Reverse proxy config (domain configurable via APP_DOMAIN)
│   │   └── Caddyfile.local     # Local overrides (gitignored)
│   ├── nginx/
│   │   ├── nginx.conf          # Main Nginx configuration
│   │   └── sites/
│   │       └── default.conf    # Virtual host config for Laravel
│   └── php/
│       ├── Dockerfile          # PHP-FPM image with extensions
│       ├── php.ini             # PHP configuration
│       └── php-fpm-healthcheck # Health check script
├── src/                        # Laravel application root
│   ├── app/
│   ├── public/                 # Web root
│   ├── resources/
│   └── .env                    # Laravel environment config
├── .env                        # Docker environment variables
├── .env.example                # Template for .env
├── docker_compose.yml          # Service definitions with health checks
└── CLAUDE.md                   # This file
```

## Common Commands

### Starting the Environment
```bash
docker-compose -f docker_compose.yml up -d    # Start all services
docker-compose -f docker_compose.yml down     # Stop all services
docker-compose -f docker_compose.yml ps       # Check service status and health
docker-compose -f docker_compose.yml build    # Rebuild PHP image after Dockerfile changes
```

### Laravel/Artisan Commands
```bash
docker exec -it app_php php artisan migrate              # Run migrations
docker exec -it app_php php artisan migrate:rollback     # Rollback migrations
docker exec -it app_php php artisan cache:clear          # Clear application cache
docker exec -it app_php php artisan config:cache         # Cache configuration
docker exec -it app_php php artisan route:cache          # Cache routes
docker exec -it app_php php artisan test                 # Run tests
docker exec -it app_php php artisan make:controller Name # Generate controller
docker exec -it app_php php artisan make:model Name      # Generate model
docker exec -it app_php php artisan make:migration name  # Generate migration
```

### Viewing Logs
```bash
docker-compose -f docker_compose.yml logs -f              # All services
docker-compose -f docker_compose.yml logs -f app_php      # PHP logs only
docker-compose -f docker_compose.yml logs -f app_nginx    # Nginx logs only
docker exec -it app_php tail -f storage/logs/laravel.log  # Laravel application logs
```

### Accessing Containers
```bash
docker exec -it app_php sh       # Shell into PHP container
docker exec -it app_mysql bash   # Shell into MySQL container
docker exec -it app_nginx sh     # Shell into Nginx container
```

### Composer Package Management
```bash
docker exec -it app_php composer install              # Install dependencies
docker exec -it app_php composer require vendor/pkg   # Add package
docker exec -it app_php composer update               # Update packages
docker exec -it app_php composer dump-autoload        # Regenerate autoloader
```

### Database Operations
```bash
# Backup database
docker exec app_mysql mysqldump -u root -p${MYSQL_ROOT_PASSWORD} app > backup.sql

# Restore database
cat backup.sql | docker exec -i app_mysql mysql -u root -p${MYSQL_ROOT_PASSWORD} app

# Access MySQL CLI
docker exec -it app_mysql mysql -u app -p
```

## Architecture

### Container Services

All services include health checks for monitoring:

- **app_caddy**: Entry point (ports 80/443), handles SSL/TLS termination via Let's Encrypt, routes `/pma` to phpMyAdmin with Basic Auth, everything else to Nginx
- **app_nginx**: Serves Laravel application from `/var/www/html/public`, proxies PHP to PHP-FPM
- **app_php**: PHP-FPM 8.4 runtime on port 9000 with Composer and extensions (PDO MySQL, Intl, mbstring, zip, opcache)
- **app_mysql**: MySQL 8.4 database with UTF8MB4 character set
- **app_phpmyadmin**: Web-based database admin accessible at `/pma` (Basic Auth protected)

### Service Dependencies

```
Caddy → Nginx → PHP-FPM
  ↓
phpMyAdmin → MySQL
```

### Directory Mapping
- `./src` → `/var/www/html` in Nginx/PHP containers (Laravel application root)
- `./src/public` → Web root (entry point: index.php)
- `./docker/*` → Configuration files mounted read-only

### Network & Storage
- All services communicate through internal `appnet` bridge network
- Only Caddy exposes ports 80/443 externally
- Persistent volumes: `mysql_data`, `caddy_data`, `caddy_config`
- Source code mounted with `:delegated` flag for performance

## Configuration Files

### Environment Variables (.env)
```bash
ACME_EMAIL           # Email for Let's Encrypt SSL certificates
APP_DOMAIN           # Domain name (e.g., example.com or localhost)
DB_DATABASE          # Database name
DB_USERNAME          # Database user
DB_PASSWORD          # Database password
MYSQL_ROOT_PASSWORD  # MySQL root password
PMA_BCRYPT_HASH      # phpMyAdmin password hash (generate with: docker run --rm caddy:2-alpine caddy hash-password)
APP_ENV              # Environment: local, staging, production
APP_DEBUG            # Debug mode: true or false
```

### Key Configuration Files
- `.env` - Docker environment variables (root level, gitignored)
- `src/.env` - Laravel environment config (gitignored)
- `.env.example` - Template with documentation for .env
- `docker/caddy/Caddyfile` - Reverse proxy routing, SSL config (uses `{$APP_DOMAIN}` variable)
- `docker/nginx/sites/default.conf` - Nginx virtual host for Laravel
- `docker/php/php.ini` - PHP settings (memory, uploads, execution time, opcache)
- `docker/php/Dockerfile` - PHP-FPM image with extensions and healthcheck

## PHP Configuration Highlights

- Memory limit: 512MB
- Upload max: 100MB
- Post max size: 100MB
- Max execution time: 300 seconds
- Timezone: America/New_York
- Opcache enabled with validate_timestamps=0 (restart PHP after code changes in production)

## Security Considerations

- `.env` files are gitignored (never commit credentials)
- phpMyAdmin requires Basic Auth (username: admin, password via bcrypt hash)
- Caddy automatically handles HTTPS with Let's Encrypt
- Database credentials managed via environment variables
- Production settings: `APP_ENV=production`, `APP_DEBUG=false`
- All Docker configs mounted read-only where possible

## Development vs Production

### Development (APP_ENV=local)
- APP_DEBUG=true for error visibility
- APP_DOMAIN=localhost (no SSL certificate required)
- Code changes reflect immediately (except opcache)
- Access via http://localhost

### Production (APP_ENV=production)
- APP_DEBUG=false for security
- APP_DOMAIN set to actual domain
- Caddy obtains real SSL certificate from Let's Encrypt
- Opcache requires container restart for code changes
- Run `php artisan config:cache`, `route:cache`, `view:cache` for performance

## Health Monitoring

All services have health checks configured in docker_compose.yml:
- **Caddy**: HTTP check on port 80
- **Nginx**: HTTP check on port 80
- **PHP-FPM**: Custom healthcheck script using fcgi
- **MySQL**: mysqladmin ping
- **phpMyAdmin**: HTTP check on port 80

View health status: `docker-compose -f docker_compose.yml ps`

## Common Workflows

### Making Code Changes
1. Edit files in `./src/`
2. Changes reflect immediately (volume mounted)
3. For config changes: `docker exec -it app_php php artisan config:clear`
4. For opcache in production: `docker-compose -f docker_compose.yml restart php`

### Adding PHP Extensions
1. Edit `docker/php/Dockerfile`
2. Add to RUN command: `&& docker-php-ext-install extension_name`
3. Rebuild: `docker-compose -f docker_compose.yml build php`
4. Restart: `docker-compose -f docker_compose.yml up -d`

### Changing PHP Settings
1. Edit `docker/php/php.ini`
2. Rebuild: `docker-compose -f docker_compose.yml build php`
3. Restart: `docker-compose -f docker_compose.yml up -d`

### Database Migrations
1. Create migration: `docker exec -it app_php php artisan make:migration name`
2. Edit migration file in `src/database/migrations/`
3. Run: `docker exec -it app_php php artisan migrate`
4. Rollback if needed: `docker exec -it app_php php artisan migrate:rollback`

## Troubleshooting

### Container won't start
- Check logs: `docker-compose -f docker_compose.yml logs service_name`
- Check health: `docker-compose -f docker_compose.yml ps`
- Rebuild: `docker-compose -f docker_compose.yml build --no-cache`

### Permission errors
```bash
docker exec -it app_php chown -R www-data:www-data /var/www/html/storage
docker exec -it app_php chown -R www-data:www-data /var/www/html/bootstrap/cache
docker exec -it app_php chmod -R 775 /var/www/html/storage
docker exec -it app_php chmod -R 775 /var/www/html/bootstrap/cache
```

### Code changes not reflecting
```bash
# Clear all Laravel caches
docker exec -it app_php php artisan optimize:clear

# Restart PHP-FPM (if opcache issue)
docker-compose -f docker_compose.yml restart php
```

### Database connection issues
- Verify .env credentials match between root .env and src/.env
- Check MySQL is healthy: `docker-compose -f docker_compose.yml ps app_mysql`
- Check logs: `docker-compose -f docker_compose.yml logs app_mysql`

## Laravel Breeze Authentication

This project includes Laravel Breeze for authentication scaffolding. It provides:
- Login/Register pages
- Password reset
- Email verification
- Profile management

Frontend stack depends on Breeze installation choice (Blade/Livewire/Inertia+React/Inertia+Vue).

## Best Practices for AI Assistance

When modifying this project:
1. Always check which environment you're working in (local vs production)
2. Test changes locally before production deployment
3. Run tests: `docker exec -it app_php php artisan test`
4. Clear caches after config changes
5. Review security implications of any credential or access changes
6. Keep Docker images updated for security patches
7. Maintain database backups before major changes
8. Document any new environment variables in .env.example
