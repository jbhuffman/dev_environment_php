# PHP Laravel Development Environment

A production-ready containerized PHP Laravel 12 development environment with Docker Compose, featuring automatic HTTPS, health monitoring, and MySQL database support.

## Features

- **Laravel 12** - Latest PHP framework with Breeze authentication
- **PHP 8.4** - Latest PHP with FPM and essential extensions
- **Automatic HTTPS** - SSL certificates via Let's Encrypt (Caddy)
- **MySQL 8.4** - Latest LTS database
- **phpMyAdmin** - Web-based database management
- **Health Monitoring** - Built-in health checks for all services
- **Development & Production** - Environment-based configuration
- **Docker Compose** - Simple orchestration and deployment

## Quick Start

### Prerequisites

- Docker and Docker Compose installed
- Domain name (for production) or localhost (for development)

### Local Development Setup

1. **Clone the repository**
   ```bash
   git clone <repository-url>
   cd dev_environment_php
   ```

2. **Configure environment**
   ```bash
   cp .env.example .env
   # Edit .env with your settings (localhost is fine for dev)

   cd src
   cp .env.example .env
   # Edit src/.env for Laravel configuration
   ```

3. **Start the environment**
   ```bash
   docker-compose -f docker_compose.yml up -d
   ```

4. **Setup Laravel**
   ```bash
   docker exec -it app_php composer install
   docker exec -it app_php php artisan key:generate
   docker exec -it app_php php artisan migrate
   ```

5. **Access your application**
   - Application: http://localhost
   - phpMyAdmin: http://localhost/pma (username: `admin`)

## Documentation

- **[DEVELOPMENT.md](DEVELOPMENT.md)** - Complete local development guide
- **[DEPLOYMENT.md](DEPLOYMENT.md)** - Production deployment instructions
- **[CLAUDE.md](CLAUDE.md)** - Technical reference for AI assistance

## Architecture

```
Internet
   ↓
Caddy (SSL/HTTPS) :80, :443
   ↓
Nginx (Web Server)
   ↓
PHP-FPM 8.4 ←→ MySQL 8.4
   ↓
Laravel 12 Application
```

### Services

| Service | Description | Container Name |
|---------|-------------|----------------|
| Caddy | Reverse proxy with automatic HTTPS | app_caddy |
| Nginx | Web server for Laravel | app_nginx |
| PHP-FPM | PHP 8.4 with extensions | app_php |
| MySQL | Database server 8.4 | app_mysql |
| phpMyAdmin | Database administration | app_phpmyadmin |

All services include health checks and automatic restart policies.

## Common Commands

### Application Management
```bash
# Start services
docker-compose -f docker_compose.yml up -d

# Stop services
docker-compose -f docker_compose.yml down

# View logs
docker-compose -f docker_compose.yml logs -f

# Check service health
docker-compose -f docker_compose.yml ps
```

### Laravel Commands
```bash
# Run migrations
docker exec -it app_php php artisan migrate

# Clear cache
docker exec -it app_php php artisan cache:clear

# Run tests
docker exec -it app_php php artisan test

# Access container shell
docker exec -it app_php sh
```

### Database Operations
```bash
# Backup database
docker exec app_mysql mysqldump -u root -p${MYSQL_ROOT_PASSWORD} app > backup.sql

# Restore database
cat backup.sql | docker exec -i app_mysql mysql -u root -p${MYSQL_ROOT_PASSWORD} app
```

## Configuration

### Environment Variables

The root `.env` file controls Docker configuration:

```bash
ACME_EMAIL=your-email@example.com        # For SSL certificates
APP_DOMAIN=localhost                      # Your domain
DB_DATABASE=app                           # Database name
DB_USERNAME=app                           # Database user
DB_PASSWORD=secure_password               # Database password
MYSQL_ROOT_PASSWORD=root_password         # MySQL root password
APP_ENV=local                             # Environment (local/production)
APP_DEBUG=true                            # Debug mode (true/false)
```

The `src/.env` file controls Laravel configuration. See [Laravel documentation](https://laravel.com/docs/configuration) for details.

### Security

- All sensitive credentials are in `.env` files (gitignored)
- phpMyAdmin protected with Basic Authentication
- Automatic HTTPS in production via Let's Encrypt
- Configurable environment-based security settings

## Development Workflow

1. **Make changes** to files in `./src/`
2. **Changes reflect immediately** (volume mounted)
3. **Clear caches** if needed: `docker exec -it app_php php artisan optimize:clear`
4. **Run tests**: `docker exec -it app_php php artisan test`
5. **Check logs**: `docker-compose -f docker_compose.yml logs -f app_php`

## Production Deployment

For production deployment:

1. Review **[DEPLOYMENT.md](DEPLOYMENT.md)** for complete instructions
2. Set production environment variables in `.env`
3. Configure your domain's DNS to point to your server
4. Deploy and let Caddy handle SSL certificates automatically
5. Follow security best practices in the deployment guide

Key production settings:
```bash
APP_ENV=production
APP_DEBUG=false
APP_DOMAIN=yourdomain.com
```

## Technology Stack

- **Backend**: PHP 8.4, Laravel 12
- **Database**: MySQL 8.4
- **Web Server**: Nginx 1.27
- **Reverse Proxy**: Caddy 2
- **Orchestration**: Docker Compose
- **Authentication**: Laravel Breeze

## PHP Extensions Included

- PDO & PDO MySQL
- Intl (Internationalization)
- mbstring (Multibyte String)
- Zip
- Opcache (Performance)
- Composer 2

## Project Structure

```
.
├── docker/              # Docker configuration
│   ├── caddy/          # Reverse proxy config
│   ├── nginx/          # Web server config
│   └── php/            # PHP-FPM configuration
├── src/                # Laravel application
├── .env.example        # Environment template
├── docker_compose.yml  # Service definitions
├── DEVELOPMENT.md      # Development guide
├── DEPLOYMENT.md       # Deployment guide
└── README.md          # This file
```

## Troubleshooting

### Permission Issues
```bash
docker exec -it app_php chown -R www-data:www-data /var/www/html/storage
docker exec -it app_php chmod -R 775 /var/www/html/storage
```

### Container Won't Start
```bash
# Check logs
docker-compose -f docker_compose.yml logs app_php

# Rebuild containers
docker-compose -f docker_compose.yml build --no-cache
docker-compose -f docker_compose.yml up -d
```

### Code Changes Not Reflecting
```bash
# Clear all caches
docker exec -it app_php php artisan optimize:clear

# Restart PHP-FPM
docker-compose -f docker_compose.yml restart php
```

See [DEVELOPMENT.md](DEVELOPMENT.md) for more troubleshooting tips.

## Contributing

1. Create a feature branch
2. Make your changes
3. Run tests: `docker exec -it app_php php artisan test`
4. Submit a pull request

## Support

- **Documentation**: See DEVELOPMENT.md and DEPLOYMENT.md
- **Laravel Docs**: https://laravel.com/docs
- **Docker Docs**: https://docs.docker.com/compose/

## License

This project is licensed under the [MIT License](LICENSE).

## Acknowledgments

- Built with [Laravel](https://laravel.com/)
- Powered by [Caddy](https://caddyserver.com/) for automatic HTTPS
- Containerized with [Docker](https://www.docker.com/)
