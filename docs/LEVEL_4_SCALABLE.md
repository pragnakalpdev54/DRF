# Level 4: Scalable API Design - Caching, Versioning & Advanced Features

## Goal

Learn to build APIs that scale and integrate with external services. Master caching, API versioning, file uploads, async views, and API documentation. By the end of this level, you'll be able to build production-ready, scalable APIs.

## Note on Examples

This guide primarily uses the **Task API** as the main example throughout, building on the Task API from Levels 2 and 3:

- **Task API**: Used as the primary example because:
  - It's familiar from previous levels (Levels 2 and 3)
  - Demonstrates scalable features well (caching, versioning, file uploads)
  - Shows real-world production patterns
  - Allows incremental learning (building on what you know)
  
- **Other examples**: Some sections may reference Book API or other models when demonstrating specific patterns that are better illustrated with different examples.

**Why Task API?**
- **Continuity**: Builds on your existing knowledge from Levels 2-3
- **Practical**: Task management is a common real-world use case
- **Scalable**: Tasks demonstrate caching, versioning, and file uploads naturally
- **Progressive**: You can see how a simple API evolves into a production-ready system

All concepts apply to any API (Book, Product, Post, etc.). The patterns are universal.

## Table of Contents

1. [Custom Middleware](#custom-middleware)
2. [Caching Strategies](#caching-strategies)
3. [Rate Limiting](#rate-limiting)
4. [API Versioning](#api-versioning)
5. [Pagination Strategies](#pagination-strategies)
6. [File Uploads](#file-uploads)
7. [Third-Party API Integration](#third-party-api-integration)
8. [Async Views](#async-views)
9. [API Documentation](#api-documentation)
10. [Testing](#testing)
11. [Step-by-Step: Task Management API](#step-by-step-task-management-api)
12. [Performance Monitoring](#performance-monitoring)
13. [Exercises](#exercises)
14. [Add-ons](#add-ons)

## Custom Middleware

### What is Middleware?

Middleware is a framework of hooks into Django's request/response processing. It's a lightweight plugin system for globally altering input or output.

### Creating Custom Middleware

```python
# api/middleware.py
import time
from django.utils.deprecation import MiddlewareMixin

class RequestLoggingMiddleware(MiddlewareMixin):
    """Log all API requests"""
    
    def process_request(self, request):
        request.start_time = time.time()
        return None
    
    def process_response(self, request, response):
        if hasattr(request, 'start_time'):
            duration = time.time() - request.start_time
            print(f"{request.method} {request.path} - {response.status_code} - {duration:.2f}s")
        return response

class CustomHeaderMiddleware(MiddlewareMixin):
    """Add custom headers to responses"""
    
    def process_response(self, request, response):
        response['X-API-Version'] = '1.0'
        response['X-Powered-By'] = 'Django REST Framework'
        return response
```

### Register Middleware

```python
# core/settings.py
MIDDLEWARE = [
    'django.middleware.security.SecurityMiddleware',
    'django.contrib.sessions.middleware.SessionMiddleware',
    'django.middleware.common.CommonMiddleware',
    'django.middleware.csrf.CsrfViewMiddleware',
    'django.contrib.auth.middleware.AuthenticationMiddleware',
    'django.contrib.messages.middleware.MessageMiddleware',
    'django.middleware.clickjacking.XFrameOptionsMiddleware',
    'api.middleware.RequestLoggingMiddleware',  # Add custom middleware
    'api.middleware.CustomHeaderMiddleware',
]
```

## Caching Strategies

### Django Cache Framework

Django provides a robust caching framework.

### Redis Setup

```bash
# Install Redis (Linux)
sudo apt install redis-server

# Install Redis (macOS)
brew install redis

# Start Redis
redis-server

# Install Python Redis client
pip install django-redis
```

### Configure Redis Cache

```python
# core/settings.py
CACHES = {
    'default': {
        'BACKEND': 'django_redis.cache.RedisCache',
        'LOCATION': 'redis://127.0.0.1:6379/1',
        'OPTIONS': {
            'CLIENT_CLASS': 'django_redis.client.DefaultClient',
        },
        'KEY_PREFIX': 'rest_api',
        'TIMEOUT': 300,  # 5 minutes default
    }
}
```

### Per-View Caching

```python
# api/views.py
from django.views.decorators.cache import cache_page
from django.utils.decorators import method_decorator
from rest_framework import viewsets

@method_decorator(cache_page(60 * 15), name='dispatch')  # Cache for 15 minutes
class BookViewSet(viewsets.ModelViewSet):
    # ...
    pass
```

### Template Fragment Caching

```python
# In templates
{% load cache %}
{% cache 500 sidebar %}
    <!-- Sidebar content -->
{% endcache %}
```

### Low-Level Cache API

```python
# api/views.py
from django.core.cache import cache

class BookViewSet(viewsets.ModelViewSet):
    def list(self, request, *args, **kwargs):
        cache_key = 'books_list'
        cached_data = cache.get(cache_key)
        
        if cached_data is None:
            queryset = self.filter_queryset(self.get_queryset())
            serializer = self.get_serializer(queryset, many=True)
            cached_data = serializer.data
            cache.set(cache_key, cached_data, 300)  # Cache for 5 minutes
        
        return Response(cached_data)
    
    def create(self, request, *args, **kwargs):
        response = super().create(request, *args, **kwargs)
        cache.delete('books_list')  # Invalidate cache
        return response
```

### Cache Decorators

```python
# api/views.py
from django.views.decorators.cache import cache_page
from django.views.decorators.vary import vary_on_headers

@cache_page(60 * 15)
@vary_on_headers('Authorization')
def my_view(request):
    # ...
    pass
```

## Rate Limiting

### DRF Throttling

```python
# core/settings.py
REST_FRAMEWORK = {
    'DEFAULT_THROTTLE_CLASSES': [
        'rest_framework.throttling.AnonRateThrottle',
        'rest_framework.throttling.UserRateThrottle'
    ],
    'DEFAULT_THROTTLE_RATES': {
        'anon': '100/hour',
        'user': '1000/hour',
        'create': '10/hour',  # Custom rate
    }
}
```

### Custom Throttle Classes

```python
# api/throttles.py
from rest_framework.throttling import UserRateThrottle

class CreateThrottle(UserRateThrottle):
    rate = '10/hour'

class BurstThrottle(UserRateThrottle):
    rate = '100/hour'

class SustainedThrottle(UserRateThrottle):
    rate = '1000/day'
```

```python
# api/views.py
from .throttles import CreateThrottle, BurstThrottle

class BookViewSet(viewsets.ModelViewSet):
    throttle_classes = [BurstThrottle]
    throttle_scope = 'create'
    
    def get_throttles(self):
        if self.action == 'create':
            return [CreateThrottle()]
        return [BurstThrottle()]
```

### Scoped Rate Throttling

```python
# core/settings.py
REST_FRAMEWORK = {
    'DEFAULT_THROTTLE_CLASSES': [
        'rest_framework.throttling.ScopedRateThrottle',
    ],
    'DEFAULT_THROTTLE_RATES': {
        'books': '100/hour',
        'tasks': '200/hour',
    }
}
```

```python
# api/views.py
class BookViewSet(viewsets.ModelViewSet):
    throttle_scope = 'books'
    # ...
```

## API Versioning

### URL Versioning

```python
# core/urls.py
from django.urls import path, include

urlpatterns = [
    path('admin/', admin.site.urls),
    path('api/v1/', include('api.v1.urls')),
    path('api/v2/', include('api.v2.urls')),
]
```

```python
# api/v1/urls.py
from django.urls import path, include
from rest_framework.routers import DefaultRouter
from . import views

router = DefaultRouter()
router.register(r'books', views.BookViewSet, basename='book')

urlpatterns = [
    path('', include(router.urls)),
]
```

### Namespace Versioning

```python
# core/urls.py
urlpatterns = [
    path('api/v1/', include(('api.urls', 'api'), namespace='v1')),
    path('api/v2/', include(('api.urls', 'api'), namespace='v2')),
]
```

### Header Versioning

```python
# core/settings.py
REST_FRAMEWORK = {
    'DEFAULT_VERSIONING_CLASS': 'rest_framework.versioning.AcceptHeaderVersioning',
    'DEFAULT_VERSION': '1.0',
    'ALLOWED_VERSIONS': ['1.0', '2.0'],
    'VERSION_PARAM': 'version',
}
```

```python
# api/views.py
from rest_framework.versioning import AcceptHeaderVersioning

class BookViewSet(viewsets.ModelViewSet):
    versioning_class = AcceptHeaderVersioning
    # ...
```

**Usage:**

```bash
curl -H "Accept: application/json; version=2.0" \
  http://localhost:8000/api/books/
```

### Query Parameter Versioning

```python
# core/settings.py
REST_FRAMEWORK = {
    'DEFAULT_VERSIONING_CLASS': 'rest_framework.versioning.QueryParameterVersioning',
    'DEFAULT_VERSION': '1.0',
    'ALLOWED_VERSIONS': ['1.0', '2.0'],
}
```

**Usage:**

```bash
curl http://localhost:8000/api/books/?version=2.0
```

### Versioning in Views

```python
# api/views.py
class BookViewSet(viewsets.ModelViewSet):
    def get_serializer_class(self):
        if self.request.version == '2.0':
            return BookV2Serializer
        return BookSerializer
```

## Pagination Strategies

### PageNumberPagination

```python
# api/pagination.py
from rest_framework.pagination import PageNumberPagination

class StandardResultsSetPagination(PageNumberPagination):
    page_size = 20
    page_size_query_param = 'page_size'
    max_page_size = 100
```

### LimitOffsetPagination

```python
# api/pagination.py
from rest_framework.pagination import LimitOffsetPagination

class LargeResultsSetPagination(LimitOffsetPagination):
    default_limit = 50
    limit_query_param = 'limit'
    offset_query_param = 'offset'
    max_limit = 1000
```

### CursorPagination

```python
# api/pagination.py
from rest_framework.pagination import CursorPagination

class TaskCursorPagination(CursorPagination):
    page_size = 25
    ordering = '-created_at'
    cursor_query_param = 'cursor'
```

### Custom Pagination

```python
# api/pagination.py
from rest_framework.pagination import PageNumberPagination
from rest_framework.response import Response

class CustomPagination(PageNumberPagination):
    page_size = 20
    
    def get_paginated_response(self, data):
        return Response({
            'links': {
                'next': self.get_next_link(),
                'previous': self.get_previous_link()
            },
            'count': self.page.paginator.count,
            'total_pages': self.page.paginator.num_pages,
            'current_page': self.page.number,
            'results': data
        })
```

## File Uploads

### Basic File Upload

```python
# api/models.py
from django.db import models

class TaskAttachment(models.Model):
    task = models.ForeignKey(Task, on_delete=models.CASCADE, related_name='attachments')
    file = models.FileField(upload_to='attachments/')
    name = models.CharField(max_length=200)
    uploaded_at = models.DateTimeField(auto_now_add=True)

    def __str__(self):
        return self.name
```

### Configure Media Settings

```python
# core/settings.py
MEDIA_URL = '/media/'
MEDIA_ROOT = BASE_DIR / 'media'
```

```python
# core/urls.py
from django.conf import settings
from django.conf.urls.static import static

urlpatterns = [
    # ... your URLs ...
]

if settings.DEBUG:
    urlpatterns += static(settings.MEDIA_URL, document_root=settings.MEDIA_ROOT)
```

### File Upload Serializer

```python
# api/serializers.py
class TaskAttachmentSerializer(serializers.ModelSerializer):
    file_size = serializers.SerializerMethodField()
    file_type = serializers.SerializerMethodField()

    class Meta:
        model = TaskAttachment
        fields = ['id', 'task', 'file', 'name', 'file_size', 'file_type', 'uploaded_at']
        read_only_fields = ['id', 'uploaded_at']

    def get_file_size(self, obj):
        if obj.file:
            return obj.file.size
        return None

    def get_file_type(self, obj):
        if obj.file:
            return obj.file.name.split('.')[-1].upper()
        return None

    def validate_file(self, value):
        max_size = 10 * 1024 * 1024  # 10MB
        if value.size > max_size:
            raise serializers.ValidationError("File size cannot exceed 10MB")
        
        allowed_types = ['pdf', 'doc', 'docx', 'jpg', 'jpeg', 'png']
        file_ext = value.name.split('.')[-1].lower()
        if file_ext not in allowed_types:
            raise serializers.ValidationError(f"File type not allowed. Allowed: {allowed_types}")
        
        return value
```

### File Upload View

```python
# api/views.py
from rest_framework import viewsets, status
from rest_framework.decorators import action
from rest_framework.response import Response
from .models import TaskAttachment
from .serializers import TaskAttachmentSerializer

class TaskAttachmentViewSet(viewsets.ModelViewSet):
    queryset = TaskAttachment.objects.all()
    serializer_class = TaskAttachmentSerializer
    permission_classes = [IsAuthenticated]

    @action(detail=False, methods=['post'], url_path='upload')
    def upload_file(self, request):
        serializer = self.get_serializer(data=request.data)
        serializer.is_valid(raise_exception=True)
        serializer.save()
        return Response(serializer.data, status=status.HTTP_201_CREATED)
```

### Image Upload with Pillow

```bash
pip install Pillow
```

```python
# api/models.py
from django.db import models

class Task(models.Model):
    # ... fields ...
    image = models.ImageField(upload_to='task_images/', blank=True, null=True)
```

```python
# api/serializers.py
class TaskSerializer(serializers.ModelSerializer):
    # ... fields ...
    
    def validate_image(self, value):
        if value:
            if value.size > 5 * 1024 * 1024:  # 5MB
                raise serializers.ValidationError("Image size cannot exceed 5MB")
            # Validate image format
            from PIL import Image
            try:
                img = Image.open(value)
                img.verify()
            except Exception:
                raise serializers.ValidationError("Invalid image format")
        return value
```

## Third-Party API Integration

### Using requests Library

```bash
pip install requests
```

```python
# api/services.py
import requests
from django.conf import settings

class ExternalAPIService:
    def __init__(self):
        self.base_url = 'https://api.example.com'
        self.api_key = settings.EXTERNAL_API_KEY
    
    def get_data(self, endpoint):
        headers = {
            'Authorization': f'Bearer {self.api_key}',
            'Content-Type': 'application/json'
        }
        response = requests.get(
            f'{self.base_url}/{endpoint}',
            headers=headers,
            timeout=10
        )
        response.raise_for_status()
        return response.json()
    
    def post_data(self, endpoint, data):
        headers = {
            'Authorization': f'Bearer {self.api_key}',
            'Content-Type': 'application/json'
        }
        response = requests.post(
            f'{self.base_url}/{endpoint}',
            headers=headers,
            json=data,
            timeout=10
        )
        response.raise_for_status()
        return response.json()
```

### Using in Views

```python
# api/views.py
from .services import ExternalAPIService
from rest_framework.decorators import action
from rest_framework.response import Response

class TaskViewSet(viewsets.ModelViewSet):
    # ... existing code ...
    
    @action(detail=True, methods=['post'])
    def sync_external(self, request, pk=None):
        task = self.get_object()
        service = ExternalAPIService()
        
        try:
            data = service.post_data('tasks', {
                'title': task.title,
                'description': task.desc,
            })
            return Response({'status': 'synced', 'external_id': data['id']})
        except requests.exceptions.RequestException as e:
            return Response({'error': str(e)}, status=status.HTTP_500_INTERNAL_SERVER_ERROR)
```

### Error Handling

```python
# api/services.py
import requests
import logging

logger = logging.getLogger(__name__)

class ExternalAPIService:
    def get_data(self, endpoint):
        try:
            response = requests.get(f'{self.base_url}/{endpoint}', timeout=10)
            response.raise_for_status()
            return response.json()
        except requests.exceptions.Timeout:
            logger.error(f"Timeout calling {endpoint}")
            raise
        except requests.exceptions.HTTPError as e:
            logger.error(f"HTTP error calling {endpoint}: {e}")
            raise
        except requests.exceptions.RequestException as e:
            logger.error(f"Request error calling {endpoint}: {e}")
            raise
```

## Async Views

### Async Function-Based Views

```python
# api/views.py
from django.http import JsonResponse
import asyncio
import aiohttp

async def async_external_api_call():
    async with aiohttp.ClientSession() as session:
        async with session.get('https://api.example.com/data') as response:
            return await response.json()

async def async_task_list(request):
    data = await async_external_api_call()
    return JsonResponse(data)
```

### Async Class-Based Views

```python
# api/views.py
from django.views import View
from django.utils.decorators import method_decorator
from django.views.decorators.csrf import csrf_exempt
import json

@method_decorator(csrf_exempt, name='dispatch')
class AsyncTaskView(View):
    async def get(self, request):
        data = await async_external_api_call()
        return JsonResponse(data)
    
    async def post(self, request):
        body = json.loads(request.body)
        # Process async
        return JsonResponse({'status': 'created'})
```

### Async in DRF (Django 4.1+)

```python
# api/views.py
from rest_framework.views import APIView
from rest_framework.response import Response
import asyncio

class AsyncBookView(APIView):
    async def get(self, request):
        # Async operations
        data = await asyncio.gather(
            self.fetch_books(),
            self.fetch_authors(),
        )
        return Response({'books': data[0], 'authors': data[1]})
    
    async def fetch_books(self):
        # Simulate async operation
        await asyncio.sleep(1)
        return list(Book.objects.values())
    
    async def fetch_authors(self):
        await asyncio.sleep(1)
        return list(Author.objects.values())
```

## API Documentation

### drf-spectacular (OpenAPI 3.0)

```bash
pip install drf-spectacular
```

```python
# core/settings.py
INSTALLED_APPS = [
    # ...
    'drf_spectacular',
]

REST_FRAMEWORK = {
    'DEFAULT_SCHEMA_CLASS': 'drf_spectacular.openapi.AutoSchema',
}

SPECTACULAR_SETTINGS = {
    'TITLE': 'Your API',
    'DESCRIPTION': 'Your API description',
    'VERSION': '1.0.0',
    'SERVE_INCLUDE_SCHEMA': False,
}
```

```python
# core/urls.py
from drf_spectacular.views import SpectacularAPIView, SpectacularSwaggerView, SpectacularRedocView

urlpatterns = [
    # ... existing URLs ...
    path('api/schema/', SpectacularAPIView.as_view(), name='schema'),
    path('api/docs/', SpectacularSwaggerView.as_view(url_name='schema'), name='swagger-ui'),
    path('api/redoc/', SpectacularRedocView.as_view(url_name='schema'), name='redoc'),
]
```

### Custom Schema

```python
# api/serializers.py
from drf_spectacular.utils import extend_schema, OpenApiParameter

class TaskViewSet(viewsets.ModelViewSet):
    @extend_schema(
        summary="List tasks",
        description="Get a list of all tasks",
        parameters=[
            OpenApiParameter(name='completed', description='Filter by completion status', required=False, type=bool),
        ],
        responses={200: TaskSerializer(many=True)}
    )
    def list(self, request):
        return super().list(request)
```

## Testing

### pytest Setup

```bash
pip install pytest pytest-django pytest-cov
```

```python
# pytest.ini
[pytest]
DJANGO_SETTINGS_MODULE = core.settings
python_files = tests.py test_*.py *_tests.py
```

### Basic Tests

```python
# api/tests.py
import pytest
from django.contrib.auth.models import User
from rest_framework.test import APIClient
from .models import Task

@pytest.fixture
def api_client():
    return APIClient()

@pytest.fixture
def user():
    return User.objects.create_user(username='testuser', password='testpass123')

@pytest.fixture
def task(user):
    return Task.objects.create(title='Test Task', owner=user)

@pytest.mark.django_db
def test_create_task(api_client, user):
    api_client.force_authenticate(user=user)
    response = api_client.post('/api/tasks/', {
        'title': 'New Task',
        'desc': 'Description',
        'completed': False
    })
    assert response.status_code == 201
    assert Task.objects.count() == 1

@pytest.mark.django_db
def test_list_tasks(api_client, user, task):
    api_client.force_authenticate(user=user)
    response = api_client.get('/api/tasks/')
    assert response.status_code == 200
    assert len(response.data) == 1
```

### Run Tests

```bash
# Run all tests
pytest

# Run with coverage
pytest --cov=api --cov-report=html

# Run specific test
pytest api/tests.py::test_create_task
```

## Step-by-Step: Task Management API

Let's build a complete Task Management API with all advanced features.

### Step 1: Update Models

```python
# api/models.py
from django.db import models
from django.contrib.auth.models import User

class TaskAttachment(models.Model):
    task = models.ForeignKey(Task, on_delete=models.CASCADE, related_name='attachments')
    file = models.FileField(upload_to='attachments/')
    name = models.CharField(max_length=200)
    uploaded_at = models.DateTimeField(auto_now_add=True)
```

### Step 2: Create Serializers

```python
# api/serializers.py
class TaskAttachmentSerializer(serializers.ModelSerializer):
    class Meta:
        model = TaskAttachment
        fields = ['id', 'task', 'file', 'name', 'uploaded_at']
```

### Step 3: Create ViewSet with Caching

```python
# api/views.py
from django.core.cache import cache
from rest_framework import viewsets
from .models import Task, TaskAttachment
from .serializers import TaskSerializer, TaskAttachmentSerializer

class TaskViewSet(viewsets.ModelViewSet):
    serializer_class = TaskSerializer
    permission_classes = [IsAuthenticated]
    
    def get_queryset(self):
        cache_key = f'tasks_user_{self.request.user.id}'
        queryset = cache.get(cache_key)
        
        if queryset is None:
            queryset = Task.objects.filter(owner=self.request.user)
            cache.set(cache_key, queryset, 300)  # Cache for 5 minutes
        
        return queryset
    
    def perform_create(self, serializer):
        serializer.save(owner=self.request.user)
        cache.delete(f'tasks_user_{self.request.user.id}')  # Invalidate cache
```

### Step 4: Add File Upload Endpoint

```python
# api/views.py
class TaskAttachmentViewSet(viewsets.ModelViewSet):
    queryset = TaskAttachment.objects.all()
    serializer_class = TaskAttachmentSerializer
    permission_classes = [IsAuthenticated]
```

### Step 5: Configure URLs with Versioning

```python
# api/v1/urls.py
from django.urls import path, include
from rest_framework.routers import DefaultRouter
from . import views

router = DefaultRouter()
router.register(r'tasks', views.TaskViewSet, basename='task')
router.register(r'attachments', views.TaskAttachmentViewSet, basename='attachment')

urlpatterns = [
    path('', include(router.urls)),
]
```

```python
# core/urls.py
urlpatterns = [
    path('admin/', admin.site.urls),
    path('api/v1/', include('api.v1.urls')),
]
```

## Performance Monitoring

### Django Debug Toolbar (Development)

```bash
pip install django-debug-toolbar
```

```python
# core/settings.py
if DEBUG:
    INSTALLED_APPS += ['debug_toolbar']
    MIDDLEWARE += ['debug_toolbar.middleware.DebugToolbarMiddleware']
    INTERNAL_IPS = ['127.0.0.1']
```

### Logging

```python
# core/settings.py
LOGGING = {
    'version': 1,
    'disable_existing_loggers': False,
    'handlers': {
        'file': {
            'level': 'INFO',
            'class': 'logging.FileHandler',
            'filename': 'api.log',
        },
    },
    'loggers': {
        'api': {
            'handlers': ['file'],
            'level': 'INFO',
        },
    },
}
```

## Exercises

### Exercise 1: Complete Task Management API

1. Add file upload functionality
2. Implement caching
3. Add API versioning
4. Create API documentation
5. Write tests

### Exercise 2: External API Integration

1. Integrate with a third-party API
2. Handle errors gracefully
3. Add retry logic
4. Cache external API responses

### Exercise 3: Performance Optimization

1. Implement query optimization
2. Add caching layers
3. Monitor performance
4. Optimize slow endpoints

## Add-ons

### Add-on 1: Custom Caching Strategy

Implement cache invalidation patterns.

### Add-on 2: API Rate Limiting Dashboard

Create a dashboard to monitor API usage.

### Add-on 3: File Processing

Add background processing for uploaded files.

## Next Steps

Congratulations! You've completed Level 4. You now know:

- ✅ Custom middleware
- ✅ Caching strategies (Redis)
- ✅ Rate limiting
- ✅ API versioning
- ✅ File uploads
- ✅ Third-party API integration
- ✅ Async views
- ✅ API documentation
- ✅ Testing

**Ready for Level 5?** Continue to [Level 5: Expert](LEVEL_5_EXPERT.md) to learn about GraphQL, WebSockets, CI/CD, and production deployment!

---

**Resources:**
- [DRF Caching](https://www.django-rest-framework.org/api-guide/caching/)
- [DRF Versioning](https://www.django-rest-framework.org/api-guide/versioning/)
- [drf-spectacular](https://drf-spectacular.readthedocs.io/)
- [Django File Uploads](https://docs.djangoproject.com/en/stable/topics/files/)

