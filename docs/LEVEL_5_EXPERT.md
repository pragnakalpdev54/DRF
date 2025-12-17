# Level 5: Expert - Production, Performance & Advanced Patterns

## Goal

Build enterprise-grade, production-ready Django APIs. Master GraphQL, WebSockets, advanced testing, CI/CD, monitoring, and deployment. By the end of this level, you'll be able to build and deploy scalable, observable APIs.

## Note on Examples

This guide uses a **mix of Task and Book API examples** depending on which best illustrates each expert-level concept:

- **Task API**: Used for:
  - Analytics and aggregation examples (task statistics)
  - Real-time features (WebSocket task updates)
  - Production deployment patterns
  - Monitoring and observability
  
- **Book API**: Used for:
  - GraphQL examples (book queries/mutations)
  - Some async patterns
  - When a simpler model better demonstrates the concept

**Why mixed examples?**
- **Best fit**: Each example is chosen to best illustrate the specific expert concept
- **Flexibility**: Shows that patterns work with any model
- **Real-world**: Different APIs have different needs - this reflects reality
- **Learning**: Understanding concepts with different models reinforces learning

**Important**: All expert-level patterns (GraphQL, WebSockets, CI/CD, deployment) apply to any API. The examples are just illustrations - you can use these patterns with Task, Book, Product, Post, or any other model.

## Table of Contents

1. [Async Django](#async-django)
2. [GraphQL Integration](#graphql-integration)
3. [WebSockets with Django Channels](#websockets-with-django-channels)
4. [Advanced Query Optimization](#advanced-query-optimization)
5. [Advanced Testing](#advanced-testing)
6. [CI/CD Setup](#cicd-setup)
7. [Monitoring and Logging](#monitoring-and-logging)
8. [Deployment Best Practices](#deployment-best-practices)
9. [Step-by-Step: Analytics API](#step-by-step-analytics-api)
10. [AI API Integration](#ai-api-integration)
11. [Observability](#observability)
12. [Exercises](#exercises)
13. [Add-ons](#add-ons)

## Async Django

### Async Views in Django 3.1+

```python
# api/views.py
from django.http import JsonResponse
import asyncio
import aiohttp

async def async_book_list(request):
    async with aiohttp.ClientSession() as session:
        async with session.get('https://api.example.com/books') as response:
            data = await response.json()
    return JsonResponse(data)
```

### Async ORM Operations (Django 4.1+)

```python
# api/views.py
from django.db import models
from asgiref.sync import sync_to_async

async def async_task_list(request):
    tasks = await sync_to_async(list)(Task.objects.all())
    return JsonResponse({'tasks': [task.title for task in tasks]})
```

### Async Database Queries

```python
# api/views.py
from asgiref.sync import sync_to_async
from django.db import transaction

@sync_to_async
def get_tasks():
    return list(Task.objects.select_related('owner').all())

async def async_task_view(request):
    tasks = await get_tasks()
    return JsonResponse({'tasks': [t.title for t in tasks]})
```

## GraphQL Integration

### Strawberry-Django Setup

```bash
pip install strawberry-graphql-django
```

```python
# core/settings.py
INSTALLED_APPS = [
    # ...
    'strawberry.django',
]
```

### Define GraphQL Schema

```python
# api/schema.py
import strawberry
from strawberry.django import DjangoModelType
from .models import Task, Book
from django.contrib.auth.models import User

@strawberry.django.type(Task)
class TaskType(DjangoModelType):
    class Meta:
        model = Task
        fields = ['id', 'title', 'desc', 'completed', 'created_at']

@strawberry.django.type(Book)
class BookType(DjangoModelType):
    class Meta:
        model = Book
        fields = ['id', 'title', 'author', 'published_date']

@strawberry.type
class Query:
    @strawberry.field
    def tasks(self) -> list[TaskType]:
        return Task.objects.all()
    
    @strawberry.field
    def task(self, id: int) -> TaskType:
        return Task.objects.get(id=id)
    
    @strawberry.field
    def books(self) -> list[BookType]:
        return Book.objects.all()

@strawberry.type
class Mutation:
    @strawberry.mutation
    def create_task(self, title: str, desc: str = None) -> TaskType:
        task = Task.objects.create(title=title, desc=desc)
        return task
    
    @strawberry.mutation
    def update_task(self, id: int, completed: bool) -> TaskType:
        task = Task.objects.get(id=id)
        task.completed = completed
        task.save()
        return task

schema = strawberry.Schema(query=Query, mutation=Mutation)
```

### Configure GraphQL URLs

```python
# core/urls.py
from strawberry.django.views import GraphQLView
from api.schema import schema

urlpatterns = [
    # ... existing URLs ...
    path('graphql/', GraphQLView.as_view(schema=schema)),
]
```

### GraphQL with Authentication

```python
# api/schema.py
from strawberry.django.auth import login, logout
from strawberry.types import Info

@strawberry.type
class Mutation:
    @strawberry.mutation
    def login(self, info: Info, username: str, password: str) -> bool:
        from django.contrib.auth import authenticate
        user = authenticate(username=username, password=password)
        if user:
            login(info.context.request, user)
            return True
        return False
```

### Test GraphQL

```bash
# Query
curl -X POST http://localhost:8000/graphql/ \
  -H "Content-Type: application/json" \
  -d '{
    "query": "query { tasks { id title completed } }"
  }'

# Mutation
curl -X POST http://localhost:8000/graphql/ \
  -H "Content-Type: application/json" \
  -d '{
    "query": "mutation { createTask(title: \"New Task\") { id title } }"
  }'
```

## WebSockets with Django Channels

### Install Channels

```bash
pip install channels channels-redis
```

### Configure Channels

```python
# core/settings.py
INSTALLED_APPS = [
    # ...
    'channels',
    'channels_redis',
]

ASGI_APPLICATION = 'core.asgi.application'

CHANNEL_LAYERS = {
    'default': {
        'BACKEND': 'channels_redis.core.RedisChannelLayer',
        'CONFIG': {
            "hosts": [('127.0.0.1', 6379)],
        },
    },
}
```

### Update ASGI Configuration

```python
# core/asgi.py
import os
from django.core.asgi import get_asgi_application
from channels.routing import ProtocolTypeRouter, URLRouter
from channels.auth import AuthMiddlewareStack
import api.routing

os.environ.setdefault('DJANGO_SETTINGS_MODULE', 'core.settings')

application = ProtocolTypeRouter({
    "http": get_asgi_application(),
    "websocket": AuthMiddlewareStack(
        URLRouter(
            api.routing.websocket_urlpatterns
        )
    ),
})
```

### Create WebSocket Consumer

```python
# api/consumers.py
from channels.generic.websocket import AsyncWebsocketConsumer
import json

class TaskConsumer(AsyncWebsocketConsumer):
    async def connect(self):
        self.room_group_name = 'tasks'
        
        await self.channel_layer.group_add(
            self.room_group_name,
            self.channel_name
        )
        
        await self.accept()
    
    async def disconnect(self, close_code):
        await self.channel_layer.group_discard(
            self.room_group_name,
            self.channel_name
        )
    
    async def receive(self, text_data):
        text_data_json = json.loads(text_data)
        message = text_data_json['message']
        
        await self.channel_layer.group_send(
            self.room_group_name,
            {
                'type': 'task_message',
                'message': message
            }
        )
    
    async def task_message(self, event):
        message = event['message']
        
        await self.send(text_data=json.dumps({
            'message': message
        }))
```

### Configure WebSocket Routing

```python
# api/routing.py
from django.urls import re_path
from . import consumers

websocket_urlpatterns = [
    re_path(r'ws/tasks/$', consumers.TaskConsumer.as_asgi()),
]
```

### Send Messages from Views

```python
# api/views.py
from channels.layers import get_channel_layer
from asgiref.sync import async_to_sync

class TaskViewSet(viewsets.ModelViewSet):
    def perform_create(self, serializer):
        task = serializer.save()
        
        # Send WebSocket message
        channel_layer = get_channel_layer()
        async_to_sync(channel_layer.group_send)(
            'tasks',
            {
                'type': 'task_message',
                'message': f'New task created: {task.title}'
            }
        )
```

## Advanced Query Optimization

### Raw SQL Queries

```python
# api/views.py
from django.db import connection

def get_tasks_with_raw_sql():
    with connection.cursor() as cursor:
        cursor.execute("""
            SELECT t.id, t.title, u.username
            FROM api_task t
            JOIN auth_user u ON t.owner_id = u.id
            WHERE t.completed = %s
        """, [False])
        return cursor.fetchall()
```

### Database-Specific Features

```python
# PostgreSQL-specific: Full-text search
from django.contrib.postgres.search import SearchVector

tasks = Task.objects.annotate(
    search=SearchVector('title', 'desc')
).filter(search='django')

# Aggregations
from django.db.models import Count, Avg, Sum

stats = Task.objects.aggregate(
    total=Count('id'),
    completed=Count('id', filter=Q(completed=True)),
    avg_priority=Avg('priority')
)
```

### Query Analysis

```python
# api/views.py
from django.db import connection
from django.db import reset_queries

class TaskViewSet(viewsets.ModelViewSet):
    def list(self, request, *args, **kwargs):
        reset_queries()
        response = super().list(request, *args, **kwargs)
        
        # Log queries
        queries = connection.queries
        print(f"Total queries: {len(queries)}")
        for query in queries:
            print(query['sql'])
        
        return response
```

### Index Optimization

```python
# api/models.py
class Task(models.Model):
    # ... fields ...
    
    class Meta:
        indexes = [
            models.Index(fields=['owner', 'completed']),
            models.Index(fields=['-created_at']),
            models.Index(fields=['title'], name='title_idx'),
        ]
```

## Advanced Testing

### Integration Tests

```python
# api/tests/test_integration.py
import pytest
from django.test import TestCase
from django.contrib.auth.models import User
from rest_framework.test import APIClient
from .models import Task

class TaskIntegrationTest(TestCase):
    def setUp(self):
        self.client = APIClient()
        self.user = User.objects.create_user(
            username='testuser',
            password='testpass123'
        )
        self.client.force_authenticate(user=self.user)
    
    def test_task_workflow(self):
        # Create task
        response = self.client.post('/api/tasks/', {
            'title': 'Test Task',
            'desc': 'Description'
        })
        self.assertEqual(response.status_code, 201)
        task_id = response.data['id']
        
        # Update task
        response = self.client.patch(f'/api/tasks/{task_id}/', {
            'completed': True
        })
        self.assertEqual(response.status_code, 200)
        
        # Delete task
        response = self.client.delete(f'/api/tasks/{task_id}/')
        self.assertEqual(response.status_code, 204)
```

### Test Coverage

```bash
# Install coverage
pip install coverage

# Run with coverage
coverage run --source='.' manage.py test
coverage report
coverage html  # Generate HTML report
```

### Mocking External APIs

```python
# api/tests/test_external_api.py
from unittest.mock import patch, Mock
import pytest

@patch('api.services.requests.get')
def test_external_api_integration(mock_get):
    mock_response = Mock()
    mock_response.json.return_value = {'data': 'test'}
    mock_response.status_code = 200
    mock_get.return_value = mock_response
    
    service = ExternalAPIService()
    result = service.get_data('endpoint')
    
    assert result == {'data': 'test'}
    mock_get.assert_called_once()
```

### Performance Tests

```python
# api/tests/test_performance.py
import time
from django.test import TestCase

class PerformanceTest(TestCase):
    def test_query_performance(self):
        start = time.time()
        tasks = Task.objects.select_related('owner').all()
        list(tasks)  # Force evaluation
        duration = time.time() - start
        
        self.assertLess(duration, 0.1)  # Should complete in < 100ms
```

## CI/CD Setup

### GitHub Actions

```yaml
# .github/workflows/django.yml
name: Django CI

on:
  push:
    branches: [ main, develop ]
  pull_request:
    branches: [ main ]

jobs:
  test:
    runs-on: ubuntu-latest
    
    services:
      postgres:
        image: postgres:15
        env:
          POSTGRES_PASSWORD: postgres
          POSTGRES_DB: test_db
        options: >-
          --health-cmd pg_isready
          --health-interval 10s
          --health-timeout 5s
          --health-retries 5
        ports:
          - 5432:5432
    
    steps:
    - uses: actions/checkout@v3
    
    - name: Set up Python
      uses: actions/setup-python@v4
      with:
        python-version: '3.11'
    
    - name: Install dependencies
      run: |
        pip install -r requirements.txt
    
    - name: Run tests
      env:
        DATABASE_URL: postgresql://postgres:postgres@localhost:5432/test_db
        SECRET_KEY: test-secret-key
      run: |
        python manage.py test
    
    - name: Run coverage
      run: |
        coverage run --source='.' manage.py test
        coverage report
```

### Docker Setup

```dockerfile
# Dockerfile
FROM python:3.11-slim

WORKDIR /app

# Install dependencies
COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt

# Copy project
COPY . .

# Run migrations and start server
CMD python manage.py migrate && python manage.py runserver 0.0.0.0:8000
```

```yaml
# docker-compose.yml
version: '3.8'

services:
  db:
    image: postgres:15
    environment:
      POSTGRES_DB: rest_api_db
      POSTGRES_USER: postgres
      POSTGRES_PASSWORD: postgres
    volumes:
      - postgres_data:/var/lib/postgresql/data
  
  web:
    build: .
    command: python manage.py runserver 0.0.0.0:8000
    volumes:
      - .:/app
    ports:
      - "8000:8000"
    depends_on:
      - db
    environment:
      DATABASE_URL: postgresql://postgres:postgres@db:5432/rest_api_db

volumes:
  postgres_data:
```

## Monitoring and Logging

### Sentry Integration

```bash
pip install sentry-sdk
```

```python
# core/settings.py
import sentry_sdk
from sentry_sdk.integrations.django import DjangoIntegration

sentry_sdk.init(
    dsn="YOUR_SENTRY_DSN",
    integrations=[DjangoIntegration()],
    traces_sample_rate=1.0,
    send_default_pii=True
)
```

### Structured Logging

```python
# core/settings.py
LOGGING = {
    'version': 1,
    'disable_existing_loggers': False,
    'formatters': {
        'verbose': {
            'format': '{levelname} {asctime} {module} {message}',
            'style': '{',
        },
        'json': {
            '()': 'pythonjsonlogger.jsonlogger.JsonFormatter',
        },
    },
    'handlers': {
        'file': {
            'level': 'INFO',
            'class': 'logging.FileHandler',
            'filename': 'api.log',
            'formatter': 'json',
        },
        'console': {
            'level': 'DEBUG',
            'class': 'logging.StreamHandler',
            'formatter': 'verbose',
        },
    },
    'loggers': {
        'api': {
            'handlers': ['file', 'console'],
            'level': 'INFO',
        },
    },
}
```

### Custom Logging

```python
# api/views.py
import logging

logger = logging.getLogger('api')

class TaskViewSet(viewsets.ModelViewSet):
    def create(self, request, *args, **kwargs):
        logger.info(f'Task creation requested by user {request.user.id}')
        try:
            response = super().create(request, *args, **kwargs)
            logger.info(f'Task created: {response.data["id"]}')
            return response
        except Exception as e:
            logger.error(f'Task creation failed: {str(e)}', exc_info=True)
            raise
```

## Deployment Best Practices

### Environment Variables

```python
# core/settings.py
from decouple import config

SECRET_KEY = config('SECRET_KEY')
DEBUG = config('DEBUG', default=False, cast=bool)
ALLOWED_HOSTS = config('ALLOWED_HOSTS', default='', cast=lambda v: [s.strip() for s in v.split(',')])

DATABASES = {
    'default': {
        'ENGINE': 'django.db.backends.postgresql',
        'NAME': config('DB_NAME'),
        'USER': config('DB_USER'),
        'PASSWORD': config('DB_PASSWORD'),
        'HOST': config('DB_HOST'),
        'PORT': config('DB_PORT', default='5432'),
    }
}
```

### Gunicorn Configuration

```python
# gunicorn_config.py
bind = "0.0.0.0:8000"
workers = 4
worker_class = "sync"
timeout = 120
keepalive = 5
max_requests = 1000
max_requests_jitter = 50
```

```bash
# Run with Gunicorn
gunicorn core.wsgi:application --config gunicorn_config.py
```

### Nginx Configuration

```nginx
# /etc/nginx/sites-available/rest_api
server {
    listen 80;
    server_name api.example.com;

    location / {
        proxy_pass http://127.0.0.1:8000;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
    }

    location /static/ {
        alias /path/to/static/;
    }

    location /media/ {
        alias /path/to/media/;
    }
}
```

### Security Checklist

```python
# core/settings.py
# Security settings for production
SECURE_SSL_REDIRECT = True
SESSION_COOKIE_SECURE = True
CSRF_COOKIE_SECURE = True
SECURE_BROWSER_XSS_FILTER = True
SECURE_CONTENT_TYPE_NOSNIFF = True
X_FRAME_OPTIONS = 'DENY'
SECURE_HSTS_SECONDS = 31536000
SECURE_HSTS_INCLUDE_SUBDOMAINS = True
SECURE_HSTS_PRELOAD = True
```

## Step-by-Step: Analytics API

Build an analytics API with aggregation, background tasks, and GraphQL.

### Step 1: Create Analytics Models

```python
# api/models.py
class Analytics(models.Model):
    date = models.DateField()
    total_tasks = models.IntegerField(default=0)
    completed_tasks = models.IntegerField(default=0)
    created_at = models.DateTimeField(auto_now_add=True)
```

### Step 2: Create Analytics View

```python
# api/views.py
from django.db.models import Count, Q
from django.utils import timezone
from datetime import timedelta

class AnalyticsViewSet(viewsets.ViewSet):
    permission_classes = [IsAuthenticated]
    
    def list(self, request):
        # Aggregate data
        stats = Task.objects.aggregate(
            total=Count('id'),
            completed=Count('id', filter=Q(completed=True)),
            pending=Count('id', filter=Q(completed=False))
        )
        
        # Time-based stats
        last_7_days = timezone.now() - timedelta(days=7)
        recent_stats = Task.objects.filter(
            created_at__gte=last_7_days
        ).aggregate(
            created=Count('id')
        )
        
        return Response({
            'overall': stats,
            'recent': recent_stats
        })
```

### Step 3: Add GraphQL Schema

```python
# api/schema.py
@strawberry.type
class AnalyticsType:
    total: int
    completed: int
    pending: int

@strawberry.type
class Query:
    @strawberry.field
    def analytics(self) -> AnalyticsType:
        stats = Task.objects.aggregate(
            total=Count('id'),
            completed=Count('id', filter=Q(completed=True)),
            pending=Count('id', filter=Q(completed=False))
        )
        return AnalyticsType(**stats)
```

## AI API Integration

### OpenAI Integration

```bash
pip install openai
```

```python
# api/services.py
import openai
from django.conf import settings

class AIService:
    def __init__(self):
        openai.api_key = settings.OPENAI_API_KEY
    
    def generate_task_suggestion(self, user_tasks):
        prompt = f"Based on these tasks: {user_tasks}, suggest a new task."
        response = openai.Completion.create(
            engine="text-davinci-003",
            prompt=prompt,
            max_tokens=100
        )
        return response.choices[0].text.strip()
```

### Use in Views

```python
# api/views.py
from .services import AIService

class TaskViewSet(viewsets.ModelViewSet):
    @action(detail=False, methods=['post'])
    def ai_suggest(self, request):
        user_tasks = Task.objects.filter(owner=request.user).values_list('title', flat=True)
        ai_service = AIService()
        suggestion = ai_service.generate_task_suggestion(list(user_tasks))
        return Response({'suggestion': suggestion})
```

## Observability

### Metrics Collection

```python
# api/middleware.py
from prometheus_client import Counter, Histogram
import time

request_count = Counter('api_requests_total', 'Total API requests', ['method', 'endpoint'])
request_duration = Histogram('api_request_duration_seconds', 'API request duration')

class MetricsMiddleware:
    def __init__(self, get_response):
        self.get_response = get_response
    
    def __call__(self, request):
        start_time = time.time()
        response = self.get_response(request)
        duration = time.time() - start_time
        
        request_count.labels(method=request.method, endpoint=request.path).inc()
        request_duration.observe(duration)
        
        return response
```

### Distributed Tracing

```python
# Install
pip install opentelemetry-api opentelemetry-sdk opentelemetry-instrumentation-django

# Configure
from opentelemetry import trace
from opentelemetry.sdk.trace import TracerProvider

trace.set_tracer_provider(TracerProvider())
```

## Exercises

### Exercise 1: Complete Analytics API

1. Create analytics models
2. Implement aggregation queries
3. Add GraphQL endpoint
4. Set up background report generation
5. Deploy to production

### Exercise 2: Full CI/CD Pipeline

1. Set up GitHub Actions
2. Configure Docker
3. Add automated testing
4. Set up deployment pipeline

### Exercise 3: Production Deployment

1. Configure Gunicorn + Nginx
2. Set up monitoring (Sentry)
3. Configure logging
4. Implement security best practices

## Add-ons

### Add-on 1: API Gateway

Implement API gateway pattern for microservices.

### Add-on 2: Multi-tenancy

Add multi-tenant support to your API.

### Add-on 3: Real-time Dashboard

Build a real-time analytics dashboard using WebSockets.

## Next Steps

Congratulations! You've completed Level 5. You now know:

- ✅ Async Django
- ✅ GraphQL integration
- ✅ WebSockets with Django Channels
- ✅ Advanced query optimization
- ✅ Advanced testing
- ✅ CI/CD setup
- ✅ Monitoring and logging
- ✅ Production deployment

**You're now an expert in Django REST Framework!** 🎉

Continue learning by:
- Contributing to open-source Django projects
- Building your own production APIs
- Reading Django and DRF source code
- Following Django community updates

---

**Resources:**
- [Django Channels](https://channels.readthedocs.io/)
- [Strawberry GraphQL](https://strawberry.rocks/)
- [Django Deployment Checklist](https://docs.djangoproject.com/en/stable/howto/deployment/checklist/)
- [Two Scoops of Django](https://www.feldroy.com/books/two-scoops-of-django-3-x)

