# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

This is a Docker Compose-based development environment for **Magento 2.3.4** with PHP 7.2, MySQL 5.7, and Nginx. The project provides a containerized local development setup for building and testing Magento extensions and customizations.

## Architecture Overview

### Container Structure

The environment consists of three main services defined in `docker-compose.yml`:

1. **PHP-FPM Service** (`magento_234_php`)
   - Base image: `php:7.2-fpm-buster`
   - Runs as non-root user `magento:magento` (UID 1000:1000)
   - Mounts `./src` to `/var/www/html` (application code)
   - Configured with high PHP limits for Magento: 2048MB memory, 18000s max execution time
   - Xdebug available but disabled by default (`XDEBUG_MODE=off`)
   - Uses `docker/php/Dockerfile` for custom PHP extensions
   - Composer 1.10.27 pre-installed

2. **MySQL Database Service** (`magento_234_mysql`)
   - Image: `mysql:5.7`
   - Database credentials in `.env.example` (default: user `magento`, password `magento`)
   - Volume `mysql_data` persists database data
   - UTF8MB4 charset configured by default
   - Performance tuning in `docker/mysql/my.cnf` (1GB InnoDB buffer pool, 256MB max_allowed_packet)
   - Slow query logging enabled (queries >2s logged)

3. **Nginx Web Server** (`magento_234_nginx`)
   - Image: `nginx:alpine`
   - Mounts `./src` to `/var/www/html` (serves from `pub/` subdirectory)
   - Serves HTTP on port 80, HTTPS configuration available (commented out)
   - Nginx config at `docker/nginx/nginx.conf` with gzip compression enabled
   - Magento-specific routing rules in `docker/nginx/conf.d/magento.conf`

### Network and Volumes

- **Network**: `magento_234` (bridge network connects all services)
- **Volumes**:
  - `mysql_data`: Persistent database storage
  - Source code volume: `./src` (to be created with Magento installation)

## Setup and Common Commands

### Initial Setup

```bash
# Copy environment file
cp .env.example .env

# Build and start containers
docker-compose up -d --build

# The PHP container will run as non-root (magento:1000), volumes are mounted automatically
```

### Running Commands in Containers

```bash
# Execute commands in PHP container
docker-compose exec php [command]

# Example: Run Magento CLI command
docker-compose exec php bin/magento cache:flush

# Access PHP container shell
docker-compose exec php bash

# Access MySQL container
docker-compose exec mysql mysql -u magento -pmagento magento
```

### Key Magento Commands

```bash
# Clear all caches
docker-compose exec php bin/magento cache:flush

# Compile static assets and DI (required before testing)
docker-compose exec php bin/magento setup:di:compile
docker-compose exec php bin/magento setup:static-content:deploy

# Reindex data
docker-compose exec php bin/magento indexer:reindex

# Check Magento version and status
docker-compose exec php bin/magento --version
```

### Logs

```bash
# View all container logs
docker-compose logs -f

# View specific service logs
docker-compose logs -f php      # PHP-FPM logs
docker-compose logs -f nginx    # Nginx logs
docker-compose logs -f mysql    # MySQL logs
```

### Stopping and Cleanup

```bash
# Stop containers (preserves data)
docker-compose stop

# Remove containers (preserves data volumes)
docker-compose down

# Remove containers and all volumes (DESTRUCTIVE)
docker-compose down -v
```

## PHP Configuration

The PHP environment is configured in:
- `docker/php/Dockerfile`: PHP extensions and system dependencies
- `docker/php/php.ini`: PHP settings

### Installed PHP Extensions

- GD (image processing)
- ZIP
- Intl (internationalization)
- BCMath (arbitrary precision math)
- PDO MySQL (database driver)
- XSL (XML transformation)
- SOAP (web services)
- Sockets

### Development-Specific Settings

- OPCache disabled (`opcache.enable = 0`) for easier development debugging
- Error reporting set to `E_ALL` with display_errors on
- High memory_limit (2048M) and upload_max_filesize (128M) for large Magento operations
- Realpath cache optimized (10M size, 7200s TTL)

## Database Configuration

MySQL 5.7 is configured specifically for Magento performance:

- **Character Set**: UTF8MB4 (unicode support for multi-language stores)
- **InnoDB Buffer Pool**: 1GB (critical for Magento performance)
- **Log File Size**: 512MB
- **Max Connections**: 500
- **Slow Query Log**: Enabled with 2-second threshold (logged to `/var/log/mysql/slow-query.log`)

Default credentials:
```
Username: magento
Password: magento
Database: magento
```

## Nginx Configuration

### Key Magento Routes

The `docker/nginx/conf.d/magento.conf` handles several Magento entry points:

1. **Setup Application** (`/setup/*`): Used for Magento installation/setup wizard
2. **Update Application** (`/update/*`): Magento maintenance/upgrade tool
3. **Main Application** (all other routes): Routed through `index.php`
4. **Static Content** (`/static/*`): Version-based cache busting with 1-year expiration
5. **Media Files** (`/media/*`): Public user-generated content
6. **Health Check** (`/health`): Simple endpoint returning "healthy" (useful for container monitoring)

### Security Features

- Denied access to sensitive files: `.user.ini`, `.git`, `.htaccess`
- Restricted media paths: `/media/customer/`, `/media/downloadable/`, `/media/import/`
- Setup/update access restricted (only allows designated entry files)
- Client max body size: 128MB (for large file uploads)

### HTTPS

HTTPS configuration is available but commented out. To enable:
1. Uncomment the `server` block at the end of `magento.conf`
2. Place SSL certificates at `docker/nginx/ssl/cert.pem` and `docker/nginx/ssl/key.pem`
3. Rebuild containers

## Environment Variables

Configuration is managed via `.env` file (copy from `.env.example`):

```
# MySQL settings
MYSQL_ROOT_PASSWORD=root
MYSQL_DATABASE=magento
MYSQL_USER=magento
MYSQL_PASSWORD=magento
MYSQL_PORT=3306

# PHP tuning
PHP_MEMORY_LIMIT=2048M
PHP_MAX_EXECUTION_TIME=18000
PHP_UPLOAD_MAX_FILESIZE=128M
PHP_POST_MAX_SIZE=128M

# Nginx
NGINX_PORT=80
NGINX_SSL_PORT=443

# Application
APP_DEBUG=true
APP_ENV=development
APP_TIMEZONE=UTC
```

## Development Workflow

1. **Mount Magento Source**: Place Magento codebase in `./src/` directory
2. **Start Services**: `docker-compose up -d`
3. **Access Container**: `docker-compose exec php bash`
4. **Run Magento Setup**: Use `bin/magento` commands or web installer at `http://localhost/setup`
5. **Compile Assets**: `bin/magento setup:di:compile && bin/magento setup:static-content:deploy`
6. **Access Application**: Open `http://localhost` in browser

## Debugging with Xdebug

Xdebug is pre-configured but disabled by default. To enable:

1. Uncomment or modify `XDEBUG_MODE` in `docker-compose.yml` (set to `debug`, `profile`, etc.)
2. Rebuild containers: `docker-compose up -d --build`
3. Configure your IDE to listen on port 9003 with path mappings:
   - Container path: `/var/www/html`
   - Local path: `./src`

Note: Xdebug can significantly slow down performance, only enable when actively debugging.

## Performance Considerations

- Database has 1GB InnoDB buffer pool configured for typical Magento workloads
- Nginx gzip compression enabled for static assets
- Static content caching set to 1-year expiration (requires content versioning)
- Realpath cache in PHP optimized to reduce filesystem checks
- Non-root user (`magento:1000`) reduces security attack surface
