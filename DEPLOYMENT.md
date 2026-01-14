# Deployment Guide

This guide covers deploying the PHP Laravel application to a production environment.

## Prerequisites

- Linux server (Ubuntu 20.04+ or similar recommended)
- Docker and Docker Compose installed
- Domain name pointing to your server
- Firewall configured to allow ports 80 and 443

## Initial Setup

### 1. Clone Repository

```bash
git clone <your-repository-url>
cd dev_environment_php
```

### 2. Configure Environment Variables

Copy the example environment file and customize it:

```bash
cp .env.example .env
nano .env
```

**Required Configuration:**

```bash
# Your email for SSL certificate notifications
ACME_EMAIL=your-email@example.com

# Your production domain
APP_DOMAIN=yourdomain.com

# Strong database passwords (generate with: openssl rand -base64 32)
DB_DATABASE=app
DB_USERNAME=app
DB_PASSWORD=<generate-strong-password>
MYSQL_ROOT_PASSWORD=<generate-strong-password>

# Generate bcrypt hash for phpMyAdmin access:
# docker run --rm caddy:2-alpine caddy hash-password --plaintext 'your-password'
PMA_BCRYPT_HASH=<your-bcrypt-hash>

# Production settings
APP_ENV=production
APP_DEBUG=false
```

### 3. Configure Laravel Application

```bash
cd src
cp .env.example .env
nano .env
```

Update the Laravel .env file with:

```bash
APP_ENV=production
APP_DEBUG=false
APP_URL=https://yourdomain.com

DB_CONNECTION=mysql
DB_HOST=mysql
DB_PORT=3306
DB_DATABASE=app
DB_USERNAME=app
DB_PASSWORD=<same-as-root-.env>
```

Generate application key:

```bash
docker-compose -f ../docker_compose.yml run --rm php php artisan key:generate
```

### 4. Build and Start Services

```bash
docker-compose -f docker_compose.yml build
docker-compose -f docker_compose.yml up -d
```

### 5. Run Database Migrations

```bash
docker exec -it app_php php artisan migrate --force
```

### 6. Set Permissions

```bash
docker exec -it app_php chown -R www-data:www-data /var/www/html/storage
docker exec -it app_php chown -R www-data:www-data /var/www/html/bootstrap/cache
```

## SSL/TLS Certificates

Caddy automatically obtains and renews SSL certificates from Let's Encrypt when:

1. Your domain (`APP_DOMAIN`) is publicly accessible
2. Ports 80 and 443 are open
3. DNS points to your server
4. `ACME_EMAIL` is set in `.env`

For the first deployment, certificates may take a few minutes to provision.

## Health Checks

All services include health checks. View status:

```bash
docker-compose -f docker_compose.yml ps
```

Healthy services show "healthy" in the STATUS column.

## Accessing Services

- **Main Application**: `https://yourdomain.com`
- **phpMyAdmin**: `https://yourdomain.com/pma` (Basic Auth: username `admin`, password configured in `PMA_BCRYPT_HASH`)

## Database Backups

### Manual Backup

```bash
docker exec app_mysql mysqldump -u root -p${MYSQL_ROOT_PASSWORD} app > backup_$(date +%Y%m%d_%H%M%S).sql
```

### Automated Backup (Recommended)

Create a backup script at `/root/backup-db.sh`:

```bash
#!/bin/bash
BACKUP_DIR="/var/backups/mysql"
mkdir -p $BACKUP_DIR
cd /path/to/dev_environment_php
source .env
docker exec app_mysql mysqldump -u root -p${MYSQL_ROOT_PASSWORD} ${DB_DATABASE} | gzip > $BACKUP_DIR/backup_$(date +%Y%m%d_%H%M%S).sql.gz

# Keep only last 7 days of backups
find $BACKUP_DIR -name "backup_*.sql.gz" -mtime +7 -delete
```

Make it executable and add to crontab:

```bash
chmod +x /root/backup-db.sh
crontab -e
# Add: 0 2 * * * /root/backup-db.sh
```

### Restore from Backup

```bash
gunzip < backup_20260114_020000.sql.gz | docker exec -i app_mysql mysql -u root -p${MYSQL_ROOT_PASSWORD} app
```

## Monitoring

### View Logs

```bash
# All services
docker-compose -f docker_compose.yml logs -f

# Specific service
docker-compose -f docker_compose.yml logs -f app_php
docker-compose -f docker_compose.yml logs -f app_nginx
docker-compose -f docker_compose.yml logs -f app_mysql
```

### Container Status

```bash
docker-compose -f docker_compose.yml ps
```

### Resource Usage

```bash
docker stats
```

## Updates and Maintenance

### Update Application Code

```bash
cd /path/to/dev_environment_php
git pull origin main
docker-compose -f docker_compose.yml exec php composer install --no-dev --optimize-autoloader
docker exec -it app_php php artisan migrate --force
docker exec -it app_php php artisan config:cache
docker exec -it app_php php artisan route:cache
docker exec -it app_php php artisan view:cache
docker-compose -f docker_compose.yml restart php nginx
```

### Update Docker Images

```bash
docker-compose -f docker_compose.yml pull
docker-compose -f docker_compose.yml up -d
```

### Rebuild PHP Image

After changes to `docker/php/Dockerfile` or `php.ini`:

```bash
docker-compose -f docker_compose.yml build php
docker-compose -f docker_compose.yml up -d php
```

## Rollback Procedure

If an update causes issues:

1. **Rollback Code:**
   ```bash
   git log --oneline -n 10  # Find commit to rollback to
   git checkout <commit-hash>
   ```

2. **Rollback Database:**
   ```bash
   gunzip < backup_YYYYMMDD_HHMMSS.sql.gz | docker exec -i app_mysql mysql -u root -p${MYSQL_ROOT_PASSWORD} app
   ```

3. **Restart Services:**
   ```bash
   docker-compose -f docker_compose.yml restart
   ```

## Troubleshooting

### Check Container Logs

```bash
docker logs app_php
docker logs app_nginx
docker logs app_mysql
docker logs app_caddy
```

### PHP Container Shell Access

```bash
docker exec -it app_php sh
```

### MySQL Container Access

```bash
docker exec -it app_mysql bash
mysql -u root -p
```

### Clear Laravel Cache

```bash
docker exec -it app_php php artisan cache:clear
docker exec -it app_php php artisan config:clear
docker exec -it app_php php artisan route:clear
docker exec -it app_php php artisan view:clear
```

### Rebuild All Containers

```bash
docker-compose -f docker_compose.yml down
docker-compose -f docker_compose.yml build --no-cache
docker-compose -f docker_compose.yml up -d
```

## Security Best Practices

1. **Keep passwords secure** - Use strong, unique passwords for all services
2. **Regular updates** - Keep Docker images and application dependencies updated
3. **Monitor logs** - Regularly check logs for suspicious activity
4. **Backups** - Maintain regular automated backups
5. **Firewall** - Only expose necessary ports (80, 443)
6. **SSL only** - Never serve the application over plain HTTP in production
7. **Disable DEBUG** - Always set `APP_DEBUG=false` in production

## Performance Optimization

### Laravel Optimization

```bash
docker exec -it app_php php artisan config:cache
docker exec -it app_php php artisan route:cache
docker exec -it app_php php artisan view:cache
docker exec -it app_php composer install --no-dev --optimize-autoloader --classmap-authoritative
```

### Database Optimization

- Regularly analyze slow query logs
- Add appropriate indexes
- Consider read replicas for high-traffic applications

## Scaling Considerations

For high-traffic applications:

1. **Load Balancer** - Place multiple app servers behind a load balancer
2. **Separate Database** - Move MySQL to dedicated server or managed service
3. **Redis Cache** - Add Redis for session and cache storage
4. **CDN** - Use CDN for static assets
5. **Queue Workers** - Implement job queues for background processing

## Support

For issues or questions:
- Check logs first: `docker-compose -f docker_compose.yml logs`
- Review this documentation
- Check Laravel documentation: https://laravel.com/docs
- Review Docker Compose documentation: https://docs.docker.com/compose/
