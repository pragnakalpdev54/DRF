# PostgreSQL Setup Guide for Django

## Overview

This guide will help you set up PostgreSQL for your Django REST Framework project. While Django works with SQLite by default, PostgreSQL is recommended for production and provides better performance, advanced features, and scalability.

## Table of Contents

1. [Installation](#installation)
2. [Creating Database and User](#creating-database-and-user)
3. [Django Configuration](#django-configuration)
4. [Migration from SQLite](#migration-from-sqlite)
5. [Connection Pooling](#connection-pooling)
6. [Backup and Restore](#backup-and-restore)
7. [Optimization Tips](#optimization-tips)
8. [Environment Variables](#environment-variables)
9. [Troubleshooting](#troubleshooting)

## Installation

### Linux (Ubuntu/Debian)

```bash
# Update package list
sudo apt update

# Install PostgreSQL
sudo apt install postgresql postgresql-contrib

# Install Python PostgreSQL adapter
pip install psycopg2-binary

# Start PostgreSQL service
sudo systemctl start postgresql
sudo systemctl enable postgresql

# Verify installation
psql --version
```

### macOS

```bash
# Using Homebrew
brew install postgresql@15

# Start PostgreSQL service
brew services start postgresql@15

# Install Python adapter
pip install psycopg2-binary
```

### Windows

1. Download PostgreSQL from [postgresql.org/download/windows/](https://www.postgresql.org/download/windows/)
2. Run the installer and follow the setup wizard
3. Remember the password you set for the `postgres` user
4. Install Python adapter:
   ```bash
   pip install psycopg2-binary
   ```

## Creating Database and User

### Step 1: Access PostgreSQL

```bash
# Switch to postgres user (Linux/macOS)
sudo -u postgres psql

# Or on Windows, use psql from the PostgreSQL bin directory
# Usually: C:\Program Files\PostgreSQL\15\bin\psql.exe
```

### Step 2: Create Database

```sql
-- Create a new database
CREATE DATABASE rest_api_db;

-- Create a user with password
CREATE USER rest_api_user WITH PASSWORD 'your_secure_password_here';

-- Grant privileges
GRANT ALL PRIVILEGES ON DATABASE rest_api_db TO rest_api_user;

-- For PostgreSQL 15+, also grant schema privileges
\c rest_api_db
GRANT ALL ON SCHEMA public TO rest_api_user;

-- Exit psql
\q
```

### Step 3: Verify Connection

```bash
# Test connection
psql -U rest_api_user -d rest_api_db -h localhost

# If prompted for password, enter the password you set
# Type \q to exit
```

## Django Configuration

### Step 1: Install Required Packages

```bash
pip install psycopg2-binary python-decouple
```

### Step 2: Update settings.py

Update your `core/settings.py`:

```python
from pathlib import Path
from decouple import config

BASE_DIR = Path(__file__).resolve().parent.parent

# Database configuration
DATABASES = {
    'default': {
        'ENGINE': 'django.db.backends.postgresql',
        'NAME': config('DB_NAME', default='rest_api_db'),
        'USER': config('DB_USER', default='rest_api_user'),
        'PASSWORD': config('DB_PASSWORD', default=''),
        'HOST': config('DB_HOST', default='localhost'),
        'PORT': config('DB_PORT', default='5432'),
        'OPTIONS': {
            'connect_timeout': 10,
        },
    }
}
```

### Step 3: Create .env File

Create a `.env` file in your project root (same level as `manage.py`):

```env
# Database Configuration
DB_NAME=rest_api_db
DB_USER=rest_api_user
DB_PASSWORD=your_secure_password_here
DB_HOST=localhost
DB_PORT=5432

# Django Secret Key (keep this secret!)
SECRET_KEY=your-secret-key-here
DEBUG=True
```

### Step 4: Update settings.py to Use .env

```python
from pathlib import Path
from decouple import config

BASE_DIR = Path(__file__).resolve().parent.parent

SECRET_KEY = config('SECRET_KEY')
DEBUG = config('DEBUG', default=False, cast=bool)

# ... other settings ...

DATABASES = {
    'default': {
        'ENGINE': 'django.db.backends.postgresql',
        'NAME': config('DB_NAME'),
        'USER': config('DB_USER'),
        'PASSWORD': config('DB_PASSWORD'),
        'HOST': config('DB_HOST'),
        'PORT': config('DB_PORT'),
    }
}
```

### Step 5: Add .env to .gitignore

Create or update `.gitignore`:

```
.env
*.pyc
__pycache__/
db.sqlite3
venv/
*.log
```

## Migration from SQLite

If you've been using SQLite and want to migrate to PostgreSQL:

### Step 1: Export Data from SQLite

```bash
# Create a data dump
python manage.py dumpdata > data.json
```

### Step 2: Update Database Settings

Switch to PostgreSQL in `settings.py` (as shown above).

### Step 3: Create New Database Schema

```bash
# Create migrations (if needed)
python manage.py makemigrations

# Apply migrations to PostgreSQL
python manage.py migrate
```

### Step 4: Load Data into PostgreSQL

```bash
# Load the data
python manage.py loaddata data.json
```

### Step 5: Verify Migration

```bash
# Check data
python manage.py shell
```

```python
from api.models import Task
print(Task.objects.count())  # Should match your SQLite count
```

## Connection Pooling

For production applications, use connection pooling to improve performance:

### Option 1: django-db-connection-pool

```bash
pip install django-db-connection-pool
```

Update `settings.py`:

```python
DATABASES = {
    'default': {
        'ENGINE': 'dj_db_conn_pool.backends.postgresql',
        'NAME': config('DB_NAME'),
        'USER': config('DB_USER'),
        'PASSWORD': config('DB_PASSWORD'),
        'HOST': config('DB_HOST'),
        'PORT': config('DB_PORT'),
        'POOL_OPTIONS': {
            'POOL_SIZE': 10,
            'MAX_OVERFLOW': 20,
            'RECYCLE': 3600,
        }
    }
}
```

### Option 2: PgBouncer (Advanced)

For high-traffic applications, consider using PgBouncer as a connection pooler.

## Backup and Restore

### Backup Database

```bash
# Using pg_dump
pg_dump -U rest_api_user -d rest_api_db -F c -f backup.dump

# Or as SQL file
pg_dump -U rest_api_user -d rest_api_db > backup.sql
```

### Restore Database

```bash
# From custom format
pg_restore -U rest_api_user -d rest_api_db backup.dump

# From SQL file
psql -U rest_api_user -d rest_api_db < backup.sql
```

### Automated Backups

Create a backup script `backup_db.sh`:

```bash
#!/bin/bash
DATE=$(date +%Y%m%d_%H%M%S)
BACKUP_DIR="/path/to/backups"
DB_NAME="rest_api_db"
DB_USER="rest_api_user"

pg_dump -U $DB_USER -d $DB_NAME -F c -f $BACKUP_DIR/backup_$DATE.dump

# Keep only last 7 days of backups
find $BACKUP_DIR -name "backup_*.dump" -mtime +7 -delete
```

## Optimization Tips

### 1. Indexes

Add indexes for frequently queried fields:

```python
# In your models.py
class Task(models.Model):
    title = models.CharField(max_length=100, db_index=True)
    completed = models.BooleanField(default=False, db_index=True)
    created_at = models.DateTimeField(auto_now_add=True, db_index=True)
    
    class Meta:
        indexes = [
            models.Index(fields=['completed', 'created_at']),
        ]
```

### 2. Query Optimization

Use `select_related` and `prefetch_related`:

```python
# Instead of
tasks = Task.objects.all()

# Use
tasks = Task.objects.select_related('user').prefetch_related('tags').all()
```

### 3. Connection Settings

```python
DATABASES = {
    'default': {
        # ... other settings ...
        'CONN_MAX_AGE': 600,  # Reuse connections for 10 minutes
        'OPTIONS': {
            'connect_timeout': 10,
            'options': '-c statement_timeout=30000',  # 30 seconds
        }
    }
}
```

### 4. PostgreSQL Configuration

Edit `/etc/postgresql/15/main/postgresql.conf` (adjust path for your version):

```conf
# Memory settings
shared_buffers = 256MB
effective_cache_size = 1GB
work_mem = 16MB

# Connection settings
max_connections = 100
```

## Environment Variables

### Development (.env)

```env
DB_NAME=rest_api_db
DB_USER=rest_api_user
DB_PASSWORD=dev_password
DB_HOST=localhost
DB_PORT=5432
DEBUG=True
SECRET_KEY=dev-secret-key
```

### Production

Use environment variables or a secrets manager:

```bash
# Set environment variables
export DB_NAME=production_db
export DB_USER=prod_user
export DB_PASSWORD=secure_production_password
export DB_HOST=db.example.com
export DB_PORT=5432
export DEBUG=False
export SECRET_KEY=production-secret-key
```

Or use a `.env.production` file (never commit this):

```env
DB_NAME=production_db
DB_USER=prod_user
DB_PASSWORD=secure_production_password
DB_HOST=db.example.com
DB_PORT=5432
DEBUG=False
SECRET_KEY=production-secret-key
```

## Troubleshooting

### Common Issues

#### 1. Connection Refused

```bash
# Check if PostgreSQL is running
sudo systemctl status postgresql

# Start if not running
sudo systemctl start postgresql
```

#### 2. Authentication Failed

- Verify username and password in `.env`
- Check PostgreSQL user exists: `\du` in psql
- Verify user has database privileges

#### 3. Database Does Not Exist

```sql
-- List databases
\l

-- Create if missing
CREATE DATABASE rest_api_db;
```

#### 4. Permission Denied

```sql
-- Grant privileges
GRANT ALL PRIVILEGES ON DATABASE rest_api_db TO rest_api_user;
\c rest_api_db
GRANT ALL ON SCHEMA public TO rest_api_user;
```

#### 5. Port Already in Use

```bash
# Check what's using port 5432
sudo lsof -i :5432

# Or change PostgreSQL port in postgresql.conf
```

### Testing Connection

```python
# In Django shell
python manage.py shell
```

```python
from django.db import connection

# Test connection
connection.ensure_connection()
print("Connection successful!")

# Check database name
from django.conf import settings
print(settings.DATABASES['default']['NAME'])
```

## Next Steps

Now that PostgreSQL is set up:

1. Update your Django settings (as shown above)
2. Run migrations: `python manage.py migrate`
3. Create a superuser: `python manage.py createsuperuser`
4. Start building your API!

For more information, refer to:
- [Django Database Documentation](https://docs.djangoproject.com/en/stable/ref/databases/)
- [PostgreSQL Documentation](https://www.postgresql.org/docs/)
- [psycopg2 Documentation](https://www.psycopg.org/docs/)

---

**Ready to continue?** Return to [Level 1: Foundations](LEVEL_1_FOUNDATIONS.md) to start building your API!

