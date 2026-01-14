# Development Guide

This guide covers local development setup and workflows.

## Quick Start

### 1. Clone and Setup

```bash
git clone <your-repository-url>
cd dev_environment_php
```

### 2. Configure Environment

```bash
# Copy and configure root .env
cp .env.example .env
nano .env
```

For local development, you can use these settings:

```bash
ACME_EMAIL=dev@localhost
APP_DOMAIN=localhost
DB_DATABASE=app
DB_USERNAME=app
DB_PASSWORD=local_dev_password
MYSQL_ROOT_PASSWORD=local_root_password
PMA_BCRYPT_HASH=$$2a$$14$$9UlMfM1m1HW/KfaXxd2Bge7lCtrE3p.oYlOM5wYYt1Aie5XHe4Jhq
APP_ENV=local
APP_DEBUG=true
```

Generate phpMyAdmin password hash (optional):

```bash
docker run --rm caddy:2-alpine caddy hash-password --plaintext 'admin'
```

### 3. Configure Laravel

```bash
cd src
cp .env.example .env
nano .env
```

Update for local development:

```bash
APP_ENV=local
APP_DEBUG=true
APP_URL=http://localhost

DB_CONNECTION=mysql
DB_HOST=mysql
DB_PORT=3306
DB_DATABASE=app
DB_USERNAME=app
DB_PASSWORD=local_dev_password
```

### 4. Start Development Environment

```bash
cd ..
docker-compose -f docker_compose.yml up -d
```

### 5. Install Dependencies and Setup

```bash
# Install PHP dependencies
docker exec -it app_php composer install

# Generate application key
docker exec -it app_php php artisan key:generate

# Run migrations
docker exec -it app_php php artisan migrate

# Seed database (if you have seeders)
docker exec -it app_php php artisan db:seed
```

### 6. Access Application

- **Application**: http://localhost
- **phpMyAdmin**: http://localhost/pma (username: `admin`, password: `admin` if using default hash)

## Development Workflow

### Composer Commands

```bash
# Install package
docker exec -it app_php composer require vendor/package

# Update dependencies
docker exec -it app_php composer update

# Dump autoloader
docker exec -it app_php composer dump-autoload
```

### Artisan Commands

```bash
# Run migrations
docker exec -it app_php php artisan migrate

# Rollback last migration
docker exec -it app_php php artisan migrate:rollback

# Create new migration
docker exec -it app_php php artisan make:migration create_table_name

# Create new controller
docker exec -it app_php php artisan make:controller ControllerName

# Create new model
docker exec -it app_php php artisan make:model ModelName

# Clear caches
docker exec -it app_php php artisan cache:clear
docker exec -it app_php php artisan config:clear
docker exec -it app_php php artisan route:clear
docker exec -it app_php php artisan view:clear
```

### Running Tests

```bash
# Run all tests
docker exec -it app_php php artisan test

# Run specific test file
docker exec -it app_php php artisan test tests/Feature/ExampleTest.php

# Run with coverage
docker exec -it app_php php artisan test --coverage
```

### Frontend Development (if using Vite/Laravel Mix)

```bash
# Install Node dependencies (run on host or in container)
cd src
npm install

# Run development server
npm run dev

# Build for production
npm run build
```

## Accessing Containers

### PHP Container

```bash
docker exec -it app_php sh

# Or use bash if needed
docker exec -it app_php bash
```

### MySQL Container

```bash
# Shell access
docker exec -it app_mysql bash

# Direct MySQL access
docker exec -it app_mysql mysql -u root -p
```

### Nginx Container

```bash
docker exec -it app_nginx sh
```

## Debugging

### Viewing Logs

```bash
# Laravel logs
docker exec -it app_php tail -f storage/logs/laravel.log

# Nginx access logs
docker-compose -f docker_compose.yml logs -f app_nginx

# PHP-FPM logs
docker-compose -f docker_compose.yml logs -f app_php

# MySQL logs
docker-compose -f docker_compose.yml logs -f app_mysql

# All containers
docker-compose -f docker_compose.yml logs -f
```

### Enable Xdebug (Optional)

Add to `docker/php/Dockerfile`:

```dockerfile
RUN apk add --no-cache $PHPIZE_DEPS \
    && pecl install xdebug \
    && docker-php-ext-enable xdebug
```

Add to `docker/php/php.ini`:

```ini
[xdebug]
xdebug.mode=debug
xdebug.start_with_request=yes
xdebug.client_host=host.docker.internal
xdebug.client_port=9003
```

Rebuild:

```bash
docker-compose -f docker_compose.yml build php
docker-compose -f docker_compose.yml up -d
```

## Database Management

### Using phpMyAdmin

1. Access http://localhost/pma
2. Login with credentials from `.env`
3. Select database from left sidebar

### Command Line

```bash
# Export database
docker exec app_mysql mysqldump -u root -plocal_root_password app > dump.sql

# Import database
cat dump.sql | docker exec -i app_mysql mysql -u root -plocal_root_password app

# Connect to MySQL CLI
docker exec -it app_mysql mysql -u app -plocal_dev_password app
```

### Resetting Database

```bash
docker exec -it app_php php artisan migrate:fresh
# Or with seeders:
docker exec -it app_php php artisan migrate:fresh --seed
```

## Common Issues and Solutions

### Permission Issues

If you encounter permission errors:

```bash
docker exec -it app_php chown -R www-data:www-data /var/www/html/storage
docker exec -it app_php chown -R www-data:www-data /var/www/html/bootstrap/cache
docker exec -it app_php chmod -R 775 /var/www/html/storage
docker exec -it app_php chmod -R 775 /var/www/html/bootstrap/cache
```

### Port Already in Use

If port 80 or 443 is already in use, modify `docker_compose.yml`:

```yaml
ports:
  - "8080:80"   # Change 80 to 8080
  - "8443:443"  # Change 443 to 8443
```

Then access at http://localhost:8080

### Container Won't Start

```bash
# Check logs
docker-compose -f docker_compose.yml logs app_php

# Remove and recreate
docker-compose -f docker_compose.yml down
docker-compose -f docker_compose.yml up -d
```

### Code Changes Not Reflecting

For PHP code changes:
```bash
# Clear Laravel cache
docker exec -it app_php php artisan cache:clear
docker exec -it app_php php artisan config:clear
docker exec -it app_php php artisan route:clear
docker exec -it app_php php artisan view:clear

# If opcache is enabled, restart PHP-FPM
docker-compose -f docker_compose.yml restart php
```

For configuration changes in Docker:
```bash
docker-compose -f docker_compose.yml restart
```

## Making Changes to Docker Configuration

### Modifying PHP Configuration

1. Edit `docker/php/php.ini`
2. Rebuild and restart:
   ```bash
   docker-compose -f docker_compose.yml build php
   docker-compose -f docker_compose.yml up -d
   ```

### Modifying Nginx Configuration

1. Edit `docker/nginx/sites/default.conf`
2. Restart Nginx:
   ```bash
   docker-compose -f docker_compose.yml restart nginx
   ```

### Installing New PHP Extensions

1. Edit `docker/php/Dockerfile`
2. Add to the RUN command:
   ```dockerfile
   && docker-php-ext-install extension_name
   ```
3. Rebuild:
   ```bash
   docker-compose -f docker_compose.yml build php
   docker-compose -f docker_compose.yml up -d
   ```

## Git Workflow

### Before Committing

```bash
# Run tests
docker exec -it app_php php artisan test

# Check code style (if using Laravel Pint)
docker exec -it app_php ./vendor/bin/pint

# Clear any development artifacts
docker exec -it app_php php artisan cache:clear
```

### .gitignore

Important files already ignored:
- `.env` (never commit!)
- `vendor/`
- `node_modules/`
- Docker volumes

## Environment Variables Reference

### Root .env (Docker Configuration)

| Variable | Description | Example |
|----------|-------------|---------|
| `ACME_EMAIL` | Email for SSL certificates | `dev@localhost` |
| `APP_DOMAIN` | Application domain | `localhost` |
| `DB_DATABASE` | Database name | `app` |
| `DB_USERNAME` | Database user | `app` |
| `DB_PASSWORD` | Database password | `secure_password` |
| `MYSQL_ROOT_PASSWORD` | MySQL root password | `root_password` |
| `PMA_BCRYPT_HASH` | phpMyAdmin password hash | See setup |
| `APP_ENV` | Environment | `local` or `production` |
| `APP_DEBUG` | Debug mode | `true` or `false` |

### src/.env (Laravel Configuration)

See Laravel documentation for complete list: https://laravel.com/docs/configuration

## Development Tips

1. **Use Git Branches**: Create feature branches for new work
2. **Run Tests**: Always run tests before committing
3. **Database Migrations**: Create migrations for schema changes
4. **Code Style**: Follow PSR-12 coding standards
5. **Comments**: Document complex logic
6. **Commit Messages**: Write clear, descriptive commit messages

## Performance in Development

Development mode has performance considerations:

- Opcache timestamp validation is disabled by default
- Restart PHP-FPM after significant code changes
- Use `php artisan optimize:clear` to clear all caches at once

## Next Steps

- Review Laravel documentation: https://laravel.com/docs
- Explore Laravel Breeze for authentication (already installed)
- Set up your IDE for Laravel development
- Configure Xdebug for step debugging
- Add database seeders for test data
- Write feature and unit tests
