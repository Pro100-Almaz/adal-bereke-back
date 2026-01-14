# Deployment Instructions for Adal-Bereke Backend

## Prerequisites

- Server with Docker and Docker Compose installed
- Git installed on the server
- SSH access to the server
- Domain name (optional, for production)

## Environment Variables

Create a `.env` file on the server with the following variables:

```bash
# Required
DEBUG=False
DJANGO_SECRET_KEY=<generate-a-strong-secret-key>
DATABASE_URL=postgres://postgres:postgres@db:5432/postgres

# Redis/Celery
REDIS_URL=redis://redis:6379/0
CELERY_BROKER_URL=redis://redis:6379/0

# Security (required for production)
ALLOWED_HOSTS=your-domain.com,your-ip-address
CORS_ALLOWED_ORIGINS=https://your-frontend-domain.com
SENTRY_DSN=<your-sentry-dsn>  # Required when DEBUG=False

# Email (optional)
EMAIL_HOST=smtp.gmail.com
EMAIL_USE_TLS=True
EMAIL_PORT=587
EMAIL_HOST_USER=your-email@gmail.com
EMAIL_HOST_PASSWORD=your-app-password
```

### Generate a Secret Key

```bash
python -c "from django.core.management.utils import get_random_secret_key; print(get_random_secret_key())"
```

Or use:
```bash
openssl rand -base64 50
```

## Deployment Steps

### 1. Clone the Repository

```bash
git clone <repository-url> /opt/adal-bereke/backend
cd /opt/adal-bereke/backend
```

### 2. Create Environment File

```bash
cp .env.example .env
nano .env  # Edit with your production values
```

### 3. Create Production Docker Compose Override (Optional)

For production, create `docker-compose.prod.yml`:

```yaml
services:
  db:
    environment:
      - POSTGRES_PASSWORD=<strong-password>
    restart: always

  redis:
    restart: always

  backend:
    command: sh -c "python manage.py migrate && python manage.py collectstatic --noinput && gunicorn conf.wsgi:application --bind 0.0.0.0:8000 --workers 4"
    environment:
      - DEBUG=False
      - DJANGO_SECRET_KEY=${DJANGO_SECRET_KEY}
      - DATABASE_URL=postgres://postgres:<db-password>@db:5432/postgres
      - REDIS_URL=redis://redis:6379/0
      - CELERY_BROKER_URL=redis://redis:6379/0
      - ALLOWED_HOSTS=${ALLOWED_HOSTS}
      - CORS_ALLOWED_ORIGINS=${CORS_ALLOWED_ORIGINS}
      - SENTRY_DSN=${SENTRY_DSN}
    restart: always

  worker:
    environment:
      - DEBUG=False
      - DJANGO_SECRET_KEY=${DJANGO_SECRET_KEY}
      - DATABASE_URL=postgres://postgres:<db-password>@db:5432/postgres
      - REDIS_URL=redis://redis:6379/0
      - CELERY_BROKER_URL=redis://redis:6379/0
      - SENTRY_DSN=${SENTRY_DSN}
    restart: always

  beat:
    environment:
      - DEBUG=False
      - DJANGO_SECRET_KEY=${DJANGO_SECRET_KEY}
      - DATABASE_URL=postgres://postgres:<db-password>@db:5432/postgres
      - REDIS_URL=redis://redis:6379/0
      - CELERY_BROKER_URL=redis://redis:6379/0
      - SENTRY_DSN=${SENTRY_DSN}
    restart: always
```

### 4. Build and Start Services

For development server:
```bash
docker compose build
docker compose up -d
```

For production:
```bash
docker compose -f docker-compose.yml -f docker-compose.prod.yml build
docker compose -f docker-compose.yml -f docker-compose.prod.yml up -d
```

### 5. Run Migrations and Create Superuser

```bash
# Migrations run automatically on startup, but you can run manually:
docker compose exec backend python manage.py migrate

# Create superuser
docker compose exec backend python manage.py createsuperuser
```

### 6. Verify Deployment

```bash
# Check running containers
docker compose ps

# Check logs
docker compose logs -f backend

# Test API health
curl http://localhost:8000/api/health/
```

## Available Make Commands

```bash
make up              # Start all services
make down            # Stop all services
make build           # Build Docker image
make rebuild         # Rebuild and restart
make logs            # View backend logs
make migrate         # Run migrations
make superuser       # Create superuser
make test            # Run tests
```

## API Endpoints

| Endpoint | Method | Description |
|----------|--------|-------------|
| `/api/health/` | GET | Health check |
| `/api/docs/` | GET | API documentation (Swagger UI) |
| `/api/auth/create/` | POST | Register new user |
| `/api/auth/login/` | POST | User login |
| `/api/auth/logout/` | POST | User logout |
| `/api/auth/profile/` | GET/PUT/PATCH | User profile |
| `/admin/` | GET | Django admin |

## Troubleshooting

### Database Connection Issues
```bash
docker compose logs db
docker compose exec db pg_isready -U postgres
```

### Redis Connection Issues
```bash
docker compose logs redis
docker compose exec redis redis-cli ping
```

### View All Logs
```bash
docker compose logs -f
```

### Restart Services
```bash
docker compose restart backend worker beat
```

### Clean Restart (removes volumes)
```bash
docker compose down -v
docker compose up -d
```

## Security Checklist for Production

- [ ] Set `DEBUG=False`
- [ ] Generate strong `DJANGO_SECRET_KEY`
- [ ] Configure `ALLOWED_HOSTS` properly
- [ ] Configure `CORS_ALLOWED_ORIGINS`
- [ ] Set up Sentry for error tracking
- [ ] Use strong database password
- [ ] Set up HTTPS with reverse proxy (nginx/traefik)
- [ ] Configure firewall rules
- [ ] Set up log rotation
- [ ] Enable automatic container restarts
