# Laravel Sail — Docker Development Environment

## What is Sail?

Laravel Sail is the official Docker Compose-based development environment for Laravel. It provides a full local stack (PHP, MySQL/PostgreSQL, Redis, Meilisearch, Mailpit, etc.) with zero manual configuration.

```bash
# New project with Sail pre-installed
curl -s "https://laravel.build/my-app?with=mysql,redis,meilisearch,mailpit" | bash
cd my-app && ./vendor/bin/sail up
```

---

## Installation on Existing Project

```bash
composer require laravel/sail --dev
php artisan sail:install
# Select services: mysql, pgsql, mariadb, redis, memcached, meilisearch,
#                  typesense, minio, mailpit, selenium, soketi
```

---

## Shell Alias (recommended)

```bash
# ~/.zshrc or ~/.bashrc
alias sail='sh $([ -f sail ] && echo sail || echo vendor/bin/sail)'

# Then use:
sail up -d          # start detached
sail down           # stop containers
sail shell          # bash inside app container
sail php artisan migrate
sail composer install
sail npm install
sail npm run dev
```

---

## docker-compose.yml Overview

Sail generates `docker-compose.yml` at the project root. Key services:

```yaml
services:
  laravel.test:           # PHP app container (your code)
    build:
      context: ./vendor/laravel/sail/runtimes/8.3
    ports:
      - '${APP_PORT:-80}:80'
      - '${VITE_PORT:-5173}:${VITE_PORT:-5173}'
    volumes:
      - '.:/var/www/html'  # live code sync
    depends_on: [mysql, redis]

  mysql:
    image: 'mysql/mysql-server:8.0'
    ports:
      - '${FORWARD_DB_PORT:-3306}:3306'
    environment:
      MYSQL_DATABASE: '${DB_DATABASE}'
      MYSQL_ROOT_PASSWORD: '${DB_PASSWORD}'
    volumes:
      - 'sail-mysql:/var/lib/mysql'   # persistent data

  redis:
    image: 'redis:alpine'
    ports:
      - '${FORWARD_REDIS_PORT:-6379}:6379'
    volumes:
      - 'sail-redis:/data'
```

---

## .env Configuration

Sail sets sensible defaults. Key variables:

```bash
APP_PORT=80             # Change if port 80 is taken
APP_SERVICE=laravel.test

DB_CONNECTION=mysql
DB_HOST=mysql           # Use service name, NOT 127.0.0.1
DB_PORT=3306
DB_DATABASE=laravel
DB_USERNAME=sail
DB_PASSWORD=password

REDIS_HOST=redis        # Use service name
REDIS_PORT=6379

MEILISEARCH_HOST=http://meilisearch:7700

MAIL_HOST=mailpit
MAIL_PORT=1025

VITE_PORT=5173
```

**Critical:** Always use Docker service names (`mysql`, `redis`) as hostnames inside the container — never `127.0.0.1`.

---

## Common Commands

```bash
# Lifecycle
sail up                  # start (foreground)
sail up -d               # start (detached/background)
sail down                # stop and remove containers
sail restart             # restart all services
sail restart mysql       # restart single service

# Logs
sail logs                # all services
sail logs -f             # follow logs
sail logs mysql          # specific service

# Application
sail php artisan migrate
sail php artisan tinker
sail php artisan queue:work
sail php artisan horizon
sail composer require spatie/laravel-data

# Testing
sail php artisan test
sail php artisan test --parallel
sail php artisan dusk    # browser tests (requires selenium service)

# Node
sail npm install
sail npm run dev
sail npm run build
sail npx vite

# Database
sail mysql               # open MySQL CLI
sail redis               # open Redis CLI
sail php artisan db:seed
```

---

## Custom PHP Configuration

```bash
# Publish Dockerfile to customize the PHP runtime
sail:publish
# Creates: docker/8.3/Dockerfile and docker/8.3/supervisord.conf
```

Common Dockerfile customizations:

```dockerfile
# docker/8.3/Dockerfile
FROM ubuntu:22.04

# Add extensions
RUN apt-get install -y php8.3-imagick php8.3-gmp

# Install Node version
RUN curl -fsSL https://deb.nodesource.com/setup_20.x | bash -
RUN apt-get install -y nodejs

# Increase PHP memory limit
RUN echo "memory_limit = 512M" > /etc/php/8.3/cli/conf.d/99-sail.ini
```

After editing, rebuild:
```bash
sail build --no-cache
sail up -d
```

---

## Adding Services

Edit `docker-compose.yml` to add services not included by default:

```yaml
# Add PostgreSQL alongside MySQL
pgsql:
  image: 'postgres:15'
  ports:
    - '${FORWARD_PGSQL_PORT:-5432}:5432'
  environment:
    POSTGRES_DB: '${DB_DATABASE}'
    POSTGRES_USER: '${DB_USERNAME}'
    POSTGRES_PASSWORD: '${DB_PASSWORD}'
  volumes:
    - 'sail-pgsql:/var/lib/postgresql/data'

# Add RabbitMQ
rabbitmq:
  image: 'rabbitmq:3-management'
  ports:
    - '5672:5672'
    - '15672:15672'  # management UI
```

```bash
sail up -d pgsql rabbitmq  # start specific services only
```

---

## Xdebug

```bash
# Enable Xdebug for step debugging (slows requests — dev only)
SAIL_XDEBUG_MODE=develop,debug,coverage sail up
```

```ini
# docker/8.3/php.ini (after sail:publish)
[xdebug]
xdebug.mode = develop,debug
xdebug.start_with_request = yes
xdebug.client_host = host.docker.internal
xdebug.client_port = 9003
xdebug.idekey = PHPSTORM
```

**VS Code launch.json:**
```json
{
    "name": "Listen for Xdebug (Sail)",
    "type": "php",
    "request": "launch",
    "port": 9003,
    "pathMappings": {
        "/var/www/html": "${workspaceFolder}"
    }
}
```

---

## Selenium & Dusk (Browser Tests)

```bash
# Add selenium to services during install
sail:install  # select selenium

# Or add manually to docker-compose.yml
selenium:
  image: 'selenium/standalone-chromium'
  extra_hosts:
    - 'host.docker.internal:host-gateway'
  volumes:
    - '/dev/shm:/dev/shm'
```

```bash
sail dusk          # runs browser tests inside the container
sail dusk --group=checkout
```

---

## Mailpit (Email Testing)

Mailpit catches all outgoing mail during development.

```bash
# .env
MAIL_HOST=mailpit
MAIL_PORT=1025

# View caught emails
open http://localhost:8025
```

---

## MinIO (S3-Compatible Storage)

```bash
# .env
AWS_ACCESS_KEY_ID=sail
AWS_SECRET_ACCESS_KEY=password
AWS_DEFAULT_REGION=us-east-1
AWS_BUCKET=local
AWS_ENDPOINT=http://minio:9000
AWS_USE_PATH_STYLE_ENDPOINT=true

# View bucket contents
open http://localhost:8900  # MinIO Console
```

---

## Multiple Projects on Same Machine

Different projects need different ports to avoid conflicts:

```bash
# project-a/.env
APP_PORT=80
FORWARD_DB_PORT=3306
FORWARD_REDIS_PORT=6379

# project-b/.env
APP_PORT=8080
FORWARD_DB_PORT=3307
FORWARD_REDIS_PORT=6380
```

---

## Performance Tips

```bash
# macOS: use mutagen for faster file sync (avoids bind mount slowness)
sail:publish  # then add mutagen config

# Or use docker-compose.override.yml with delegated mounts
volumes:
  - '.:/var/www/html:delegated'
```

**Linux:** Bind mounts are native speed — no extra config needed.

**Windows:** Use WSL2 and store the project inside the WSL2 filesystem (`~/projects/`), not on the Windows drive (`/mnt/c/`).

---

## CI/CD with Sail

For GitHub Actions, use the native PHP setup instead of Sail (Sail adds overhead in CI):

```yaml
# .github/workflows/tests.yml
- uses: shivammathur/setup-php@v2
  with:
    php-version: '8.3'

- run: composer install
- run: cp .env.ci .env && php artisan key:generate
- run: php artisan migrate
- run: php artisan test --parallel
```

Keep Sail for local development; use direct PHP in CI.

---

## Common Issues

| Problem | Cause | Fix |
|---|---|---|
| `Port 80 already in use` | Another service on port 80 | Set `APP_PORT=8080` in `.env` |
| `DB_HOST=127.0.0.1` fails | Wrong hostname | Set `DB_HOST=mysql` (service name) |
| File changes not reflected | Volume cache | `sail build --no-cache` then `sail up` |
| Slow on macOS | Bind mount overhead | Use Mutagen or store in Docker volume |
| `sail: command not found` | No alias set | Add alias or use `./vendor/bin/sail` |
| npm run dev not accessible | Vite port not forwarded | Set `VITE_PORT=5173` and expose in docker-compose.yml |
