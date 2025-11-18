# Level 1: Foundations - Django + REST Basics

## Goal

Understand Django, Django REST Framework, and build your first REST API. By the end of this level, you'll be able to create a fully functional CRUD API.

## Table of Contents

1. [REST Fundamentals](#rest-fundamentals)
2. [Django Project Structure](#django-project-structure)
3. [Setting Up Your Environment](#setting-up-your-environment)
4. [Installing Django REST Framework](#installing-django-rest-framework)
5. [CORS Configuration](#cors-configuration)
6. [Understanding Models](#understanding-models)
7. [Understanding Serializers](#understanding-serializers)
8. [Understanding Views](#understanding-views)
9. [URL Routing](#url-routing)
10. [Step-by-Step: Book API Implementation](#step-by-step-book-api-implementation)
11. [Step-by-Step: Task API Implementation](#step-by-step-task-api-implementation)
12. [Custom Response Formats](#custom-response-formats)
13. [Error Handling](#error-handling)
14. [Testing with cURL](#testing-with-curl)
15. [Browsable API](#browsable-api)
16. [Exercises](#exercises)
17. [Trivia](#trivia)
18. [Practice Problems](#practice-problems)
19. [Add-ons](#add-ons)

## REST Fundamentals

### What is REST?

REST (Representational State Transfer) is an architectural style for designing web services. REST APIs use HTTP methods to perform operations on resources.

### HTTP Methods

| Method | Purpose | Example |
|--------|---------|---------|
| GET | Retrieve data | Get list of books |
| POST | Create new resource | Create a new book |
| PUT | Full update | Replace entire book |
| PATCH | Partial update | Update only book title |
| DELETE | Delete resource | Delete a book |

### HTTP Status Codes

| Code | Meaning | When to Use |
|------|---------|-------------|
| 200 | OK | Successful GET, PUT, PATCH |
| 201 | Created | Successful POST |
| 204 | No Content | Successful DELETE |
| 400 | Bad Request | Invalid input data |
| 401 | Unauthorized | Missing/invalid authentication |
| 403 | Forbidden | Authenticated but no permission |
| 404 | Not Found | Resource doesn't exist |
| 500 | Server Error | Internal server error |

### Request/Response Cycle

```
Client Request → Django → URL Router → View → Serializer → Model → Database
                                                                    ↓
Client Response ← Django ← JSON Response ← Serializer ← Model ← Database
```

## Django Project Structure

### Understanding Django Files

```
rest_api_learn/
├── manage.py                 # Django management script
├── core/                     # Project settings package
│   ├── __init__.py
│   ├── settings.py           # Project configuration
│   ├── urls.py               # Root URL configuration
│   ├── wsgi.py               # WSGI configuration
│   └── asgi.py               # ASGI configuration
├── api/                      # Your app
│   ├── __init__.py
│   ├── models.py             # Database models
│   ├── views.py              # View logic
│   ├── serializers.py        # DRF serializers
│   ├── urls.py               # App URL configuration
│   ├── admin.py              # Admin interface
│   └── migrations/           # Database migrations
└── db.sqlite3                # SQLite database (default)
```

### Key Files Explained

**settings.py**: Contains all project configuration
- `INSTALLED_APPS`: List of installed Django apps
- `DATABASES`: Database configuration
- `MIDDLEWARE`: Request/response processing
- `ROOT_URLCONF`: Root URL configuration

**models.py**: Define your database structure
- Each class = database table
- Each attribute = database column

**views.py**: Handle HTTP requests
- Contains view functions/classes
- Process requests and return responses

**urls.py**: URL routing
- Maps URLs to views
- Defines API endpoints

## Setting Up Your Environment

### Step 1: Create Virtual Environment

```bash
# Create virtual environment
python -m venv venv

# Activate (Linux/macOS)
source venv/bin/activate

# Activate (Windows)
venv\Scripts\activate
```

### Step 2: Install Django and DRF

```bash
# Install packages
pip install django djangorestframework

# Or with specific versions
pip install django==5.2.8 djangorestframework==3.15.2

# Save requirements
pip freeze > requirements.txt
```

### Step 3: Create Django Project

```bash
# Create project
django-admin startproject core .

# Create app
python manage.py startapp api
```

### Step 4: Register App in settings.py

```python
# core/settings.py
INSTALLED_APPS = [
    'django.contrib.admin',
    'django.contrib.auth',
    'django.contrib.contenttypes',
    'django.contrib.sessions',
    'django.contrib.messages',
    'django.contrib.staticfiles',
    'rest_framework',  # Add DRF
    'api',             # Add your app
]
```

### Step 5: Run Initial Migration

```bash
# Apply default migrations
python manage.py migrate

# Create superuser (optional)
python manage.py createsuperuser
```

## Installing Django REST Framework

### Step 1: Add to INSTALLED_APPS

```python
# core/settings.py
INSTALLED_APPS = [
    # ... other apps ...
    'rest_framework',
    'api',
]
```

### Step 2: Configure DRF Settings (Optional)

```python
# core/settings.py
REST_FRAMEWORK = {
    'DEFAULT_RENDERER_CLASSES': [
        'rest_framework.renderers.JSONRenderer',
        'rest_framework.renderers.BrowsableAPIRenderer',  # For browsable API
    ],
    'DEFAULT_PARSER_CLASSES': [
        'rest_framework.parsers.JSONParser',
        'rest_framework.parsers.FormParser',
        'rest_framework.parsers.MultiPartParser',
    ],
    'DEFAULT_PAGINATION_CLASS': None,  # We'll add pagination in Level 2
}
```

### Step 3: Verify Installation

```bash
# Start server
python manage.py runserver

# Visit http://127.0.0.1:8000/
# You should see Django welcome page
```

**Why we verify**: This confirms Django is installed correctly and the server can run. If you see the welcome page, everything is working!

## CORS Configuration

### What is CORS?

**CORS** = Cross-Origin Resource Sharing

**The Problem**: By default, browsers block requests from one domain (origin) to another. This is a security feature called the "Same-Origin Policy."

**Example**:
- Your frontend runs on: `http://localhost:3000` (React app)
- Your API runs on: `http://localhost:8000` (Django)
- Browser blocks requests between them ❌

**The Solution**: CORS headers tell the browser "it's okay, allow this request."

**Why this matters**: 
- Frontend and backend often run on different ports/domains
- Without CORS, your frontend can't call your API
- CORS is essential for any API that will be used by web applications

### Installing django-cors-headers

```bash
# Install the package
pip install django-cors-headers

# Add to requirements.txt
pip freeze > requirements.txt
```

**Why this package**: It makes CORS configuration easy. You could configure it manually, but this package handles all the complexity.

### Step 1: Add to INSTALLED_APPS

```python
# core/settings.py
INSTALLED_APPS = [
    'django.contrib.admin',
    'django.contrib.auth',
    'django.contrib.contenttypes',
    'django.contrib.sessions',
    'django.contrib.messages',
    'django.contrib.staticfiles',
    'rest_framework',
    'corsheaders',  # Add this - must be before other apps
    'api',
]
```

**Why order matters**: `corsheaders` must be before other apps that might handle requests. Middleware order is important in Django.

### Step 2: Add CORS Middleware

```python
# core/settings.py
MIDDLEWARE = [
    'django.middleware.security.SecurityMiddleware',
    'corsheaders.middleware.CorsMiddleware',  # Add this - must be near top
    'django.contrib.sessions.middleware.SessionMiddleware',
    'django.middleware.common.CommonMiddleware',
    'django.middleware.csrf.CsrfViewMiddleware',
    'django.contrib.auth.middleware.AuthenticationMiddleware',
    'django.contrib.messages.middleware.MessageMiddleware',
    'django.middleware.clickjacking.XFrameOptionsMiddleware',
]
```

**Why middleware position matters**: 
- Middleware processes requests in order
- CORS middleware must run early to add headers to responses
- If it's too late, headers won't be added

### Step 3: Configure CORS Settings

#### Development (Allow All Origins)

```python
# core/settings.py
# For development only - allows all origins
CORS_ALLOW_ALL_ORIGINS = True
```

**Why this works for development**: 
- Quick to set up
- Allows any frontend to connect
- **Never use in production!** (security risk)

#### Production (Specific Origins)

```python
# core/settings.py
# For production - specify allowed origins
CORS_ALLOW_ALL_ORIGINS = False

CORS_ALLOWED_ORIGINS = [
    "http://localhost:3000",      # React dev server
    "http://localhost:5173",      # Vite dev server
    "https://yourdomain.com",     # Production frontend
    "https://www.yourdomain.com",
]

# Or use regex patterns
CORS_ALLOWED_ORIGIN_REGEXES = [
    r"^https://.*\.yourdomain\.com$",  # All subdomains
]
```

**Why specific origins in production**: 
- Security: Only allow trusted domains
- Prevents unauthorized sites from using your API
- Protects user data

### Step 4: Additional CORS Settings (Optional)

```python
# core/settings.py
# Allow credentials (cookies, auth headers)
CORS_ALLOW_CREDENTIALS = True

# Allowed HTTP methods
CORS_ALLOW_METHODS = [
    'DELETE',
    'GET',
    'OPTIONS',
    'PATCH',
    'POST',
    'PUT',
]

# Allowed headers
CORS_ALLOW_HEADERS = [
    'accept',
    'accept-encoding',
    'authorization',
    'content-type',
    'dnt',
    'origin',
    'user-agent',
    'x-csrftoken',
    'x-requested-with',
]

# How long browser can cache preflight response
CORS_PREFLIGHT_MAX_AGE = 86400  # 24 hours
```

**Explanation of settings**:
- **CORS_ALLOW_CREDENTIALS**: Needed if you send cookies/auth tokens
- **CORS_ALLOW_METHODS**: Which HTTP methods are allowed
- **CORS_ALLOW_HEADERS**: Which headers frontend can send
- **CORS_PREFLIGHT_MAX_AGE**: Reduces preflight requests (performance)

### Testing CORS

**Test with cURL**:
```bash
# Test CORS headers
curl -H "Origin: http://localhost:3000" \
     -H "Access-Control-Request-Method: GET" \
     -H "Access-Control-Request-Headers: X-Requested-With" \
     -X OPTIONS \
     http://localhost:8000/api/books/ \
     -v

# Should see CORS headers in response:
# Access-Control-Allow-Origin: http://localhost:3000
# Access-Control-Allow-Methods: GET, POST, PUT, DELETE, OPTIONS
```

**Test in Browser Console**:
```javascript
// In browser console (on frontend)
fetch('http://localhost:8000/api/books/')
  .then(response => response.json())
  .then(data => console.log(data))
  .catch(error => console.error('Error:', error));
```

**Why test**: 
- Confirms CORS is working
- Helps debug connection issues
- Verifies headers are correct

### Common CORS Issues

**Problem**: "CORS policy blocked"
- **Solution**: Check `CORS_ALLOWED_ORIGINS` includes your frontend URL
- **Check**: Verify middleware is in correct position

**Problem**: "Credentials not allowed"
- **Solution**: Set `CORS_ALLOW_CREDENTIALS = True`
- **Check**: Make sure origin is in allowed list (not using `*`)

**Problem**: "Method not allowed"
- **Solution**: Add method to `CORS_ALLOW_METHODS`
- **Check**: Verify you're using correct HTTP method

### Why CORS is Important

1. **Security**: Prevents unauthorized sites from accessing your API
2. **Flexibility**: Allows frontend/backend separation
3. **Modern Development**: Essential for single-page applications (SPA)
4. **Mobile Apps**: Often need CORS for web-based APIs

**Remember**: 
- Development: `CORS_ALLOW_ALL_ORIGINS = True` is fine
- Production: Always specify exact origins
- Test CORS before deploying

## Understanding Models

### What are Models?

Models are Python classes that represent database tables. Django's ORM (Object-Relational Mapping) converts model classes to SQL tables.

**Simple Explanation**: 
- **Model** = Blueprint for a database table
- **ORM** = Magic that converts Python code to SQL
- **Table** = Where data is stored (like a spreadsheet)
- **Row** = One record (one book, one task)
- **Column** = One field (title, author, price)

**Why Models?**
- **No SQL needed**: Write Python instead of SQL
- **Type safety**: Django validates data types
- **Migrations**: Django tracks database changes
- **Relationships**: Easy to connect models together

**Real-world analogy**: 
- Model = Recipe (blueprint)
- Table = Baking pan (where data goes)
- Object = Cookie (one record)
- Fields = Ingredients (title, author, etc.)

### Basic Model Example

```python
# api/models.py
from django.db import models

class Book(models.Model):
    title = models.CharField(max_length=200)
    author = models.CharField(max_length=100)
    published_date = models.DateField()
    isbn = models.CharField(max_length=13, unique=True)
    created_at = models.DateTimeField(auto_now_add=True)
    updated_at = models.DateTimeField(auto_now=True)

    def __str__(self):
        return self.title

    class Meta:
        ordering = ['-created_at']
```

### Common Field Types

| Field Type | Purpose | Example |
|------------|---------|---------|
| CharField | Short text | `title = models.CharField(max_length=200)` |
| TextField | Long text | `description = models.TextField()` |
| IntegerField | Integer | `price = models.IntegerField()` |
| DecimalField | Decimal | `price = models.DecimalField(max_digits=10, decimal_places=2)` |
| BooleanField | True/False | `is_published = models.BooleanField(default=False)` |
| DateField | Date | `published_date = models.DateField()` |
| DateTimeField | Date + Time | `created_at = models.DateTimeField(auto_now_add=True)` |
| EmailField | Email | `email = models.EmailField()` |
| URLField | URL | `website = models.URLField()` |
| ForeignKey | Many-to-One | `author = models.ForeignKey(User, on_delete=models.CASCADE)` |

### Field Options

```python
# Common options
title = models.CharField(
    max_length=200,
    null=True,              # Allow NULL in database
    blank=True,             # Allow empty in forms
    default='Untitled',     # Default value
    unique=True,            # Must be unique
    db_index=True,          # Create database index
    verbose_name='Book Title',  # Human-readable name
    help_text='Enter the book title',  # Help text
)
```

### Creating and Applying Migrations

**What are Migrations?**

Migrations are files that describe changes to your database structure. Think of them as version control for your database.

**Why Migrations?**
- **Track changes**: Know what changed and when
- **Team collaboration**: Everyone has same database structure
- **Rollback**: Can undo changes if needed
- **Production**: Apply same changes to production database

```bash
# Create migration files
python manage.py makemigrations
```

**What this does**:
- Looks at your models
- Compares with current database
- Creates migration file with changes
- **Doesn't change database yet!**

**Why two steps?**: 
- `makemigrations` = Create the plan
- `migrate` = Execute the plan
- This lets you review changes before applying

```bash
# Apply migrations to database
python manage.py migrate
```

**What this does**:
- Reads migration files
- Executes SQL to update database
- Creates/updates tables
- **Actually changes the database!**

```bash
# See SQL that will be executed
python manage.py sqlmigrate api 0001
```

**Why view SQL**: 
- Understand what Django is doing
- Debug migration issues
- Learn SQL
- Verify changes before applying

## Understanding Serializers

### What are Serializers?

Serializers convert complex data types (like Django model instances) to Python native datatypes that can be easily rendered into JSON, XML, etc.

**Simple Explanation**:
- **Serialize** = Convert Python object → JSON (for sending to client)
- **Deserialize** = Convert JSON → Python object (for receiving from client)
- **Validation** = Check if data is correct before saving

**Why Serializers?**
- **Format conversion**: Python objects ↔ JSON
- **Data validation**: Ensure data is correct
- **Field control**: Choose which fields to include
- **Type conversion**: Convert data types automatically

**Real-world analogy**:
- Serializer = Translator
- Python object = English
- JSON = Spanish
- Serializer translates between them

**The Flow**:
```
Client sends JSON → Serializer validates → Convert to Python → Save to Database
Database has Python object → Serializer converts → JSON → Send to Client
```

### ModelSerializer (Recommended)

```python
# api/serializers.py
from rest_framework import serializers
from .models import Book

class BookSerializer(serializers.ModelSerializer):
    class Meta:
        model = Book
        fields = ['id', 'title', 'author', 'published_date', 'isbn', 'created_at']
        # Or use '__all__' to include all fields
        # fields = '__all__'
        
        # Exclude specific fields
        # exclude = ['created_at']
        
        # Read-only fields
        read_only_fields = ['id', 'created_at']
```

### Serializer Class (More Control)

```python
# api/serializers.py
from rest_framework import serializers

class BookSerializer(serializers.Serializer):
    id = serializers.IntegerField(read_only=True)
    title = serializers.CharField(max_length=200)
    author = serializers.CharField(max_length=100)
    published_date = serializers.DateField()
    isbn = serializers.CharField(max_length=13)
    created_at = serializers.DateTimeField(read_only=True)

    def create(self, validated_data):
        """Create and return a new Book instance"""
        return Book.objects.create(**validated_data)

    def update(self, instance, validated_data):
        """Update and return an existing Book instance"""
        instance.title = validated_data.get('title', instance.title)
        instance.author = validated_data.get('author', instance.author)
        instance.published_date = validated_data.get('published_date', instance.published_date)
        instance.isbn = validated_data.get('isbn', instance.isbn)
        instance.save()
        return instance
```

### Serializer Field Types

```python
class ExampleSerializer(serializers.Serializer):
    # Basic fields
    title = serializers.CharField(max_length=200)
    description = serializers.CharField(allow_blank=True)
    price = serializers.DecimalField(max_digits=10, decimal_places=2)
    quantity = serializers.IntegerField(min_value=0)
    is_available = serializers.BooleanField(default=True)
    
    # Date/Time fields
    created_at = serializers.DateTimeField(read_only=True)
    published_date = serializers.DateField()
    
    # Email and URL
    email = serializers.EmailField()
    website = serializers.URLField()
    
    # Choice field
    STATUS_CHOICES = [
        ('draft', 'Draft'),
        ('published', 'Published'),
    ]
    status = serializers.ChoiceField(choices=STATUS_CHOICES)
    
    # Nested serializer (we'll cover in Level 3)
    # author = AuthorSerializer()
```

## Understanding Views

### View Evolution in DRF

DRF provides multiple ways to create views, from simple to powerful:

1. **Function-based views** → Basic, explicit
2. **APIView** → Class-based, more control
3. **Generic Views** → Less code, common patterns
4. **ViewSets** → Most powerful, works with routers

### Method 1: Function-Based View

```python
# api/views.py
from rest_framework.decorators import api_view
from rest_framework.response import Response
from rest_framework import status
from .models import Book
from .serializers import BookSerializer

@api_view(['GET', 'POST'])
def book_list(request):
    """List all books or create a new book"""
    if request.method == 'GET':
        books = Book.objects.all()
        serializer = BookSerializer(books, many=True)
        return Response(serializer.data)
    
    elif request.method == 'POST':
        serializer = BookSerializer(data=request.data)
        if serializer.is_valid():
            serializer.save()
            return Response(serializer.data, status=status.HTTP_201_CREATED)
        return Response(serializer.errors, status=status.HTTP_400_BAD_REQUEST)
```

### Method 2: APIView (Class-Based)

```python
# api/views.py
from rest_framework.views import APIView
from rest_framework.response import Response
from rest_framework import status
from .models import Book
from .serializers import BookSerializer

class BookListAPIView(APIView):
    """List all books or create a new book"""
    
    def get(self, request):
        books = Book.objects.all()
        serializer = BookSerializer(books, many=True)
        return Response(serializer.data)
    
    def post(self, request):
        serializer = BookSerializer(data=request.data)
        if serializer.is_valid():
            serializer.save()
            return Response(serializer.data, status=status.HTTP_201_CREATED)
        return Response(serializer.errors, status=status.HTTP_400_BAD_REQUEST)

class BookDetailAPIView(APIView):
    """Retrieve, update or delete a book"""
    
    def get_object(self, pk):
        try:
            return Book.objects.get(pk=pk)
        except Book.DoesNotExist:
            raise Http404
    
    def get(self, request, pk):
        book = self.get_object(pk)
        serializer = BookSerializer(book)
        return Response(serializer.data)
    
    def put(self, request, pk):
        book = self.get_object(pk)
        serializer = BookSerializer(book, data=request.data)
        if serializer.is_valid():
            serializer.save()
            return Response(serializer.data)
        return Response(serializer.errors, status=status.HTTP_400_BAD_REQUEST)
    
    def delete(self, request, pk):
        book = self.get_object(pk)
        book.delete()
        return Response(status=status.HTTP_204_NO_CONTENT)
```

### Method 3: Generic Views (Less Code)

```python
# api/views.py
from rest_framework import generics
from .models import Book
from .serializers import BookSerializer

class BookListCreateView(generics.ListCreateAPIView):
    """List all books or create a new book"""
    queryset = Book.objects.all()
    serializer_class = BookSerializer

class BookRetrieveUpdateDestroyView(generics.RetrieveUpdateDestroyAPIView):
    """Retrieve, update or delete a book"""
    queryset = Book.objects.all()
    serializer_class = BookSerializer
```

### Method 4: ViewSets (Most Powerful)

```python
# api/views.py
from rest_framework import viewsets
from .models import Book
from .serializers import BookSerializer

class BookViewSet(viewsets.ModelViewSet):
    """ViewSet for viewing and editing Book instances"""
    queryset = Book.objects.all()
    serializer_class = BookSerializer
```

## URL Routing

### Basic URL Configuration

```python
# core/urls.py
from django.contrib import admin
from django.urls import path, include

urlpatterns = [
    path('admin/', admin.site.urls),
    path('api/', include('api.urls')),
]
```

### Function-Based and APIView URLs

```python
# api/urls.py
from django.urls import path
from . import views

urlpatterns = [
    path('books/', views.book_list, name='book-list'),
    path('books/<int:pk>/', views.book_detail, name='book-detail'),
]

# Or for APIView
urlpatterns = [
    path('books/', views.BookListAPIView.as_view(), name='book-list'),
    path('books/<int:pk>/', views.BookDetailAPIView.as_view(), name='book-detail'),
]
```

### Generic Views URLs

```python
# api/urls.py
from django.urls import path
from . import views

urlpatterns = [
    path('books/', views.BookListCreateView.as_view(), name='book-list-create'),
    path('books/<int:pk>/', views.BookRetrieveUpdateDestroyView.as_view(), name='book-detail'),
]
```

### ViewSet URLs with Router

```python
# api/urls.py
from django.urls import path, include
from rest_framework.routers import DefaultRouter
from . import views

router = DefaultRouter()
router.register(r'books', views.BookViewSet, basename='book')

urlpatterns = [
    path('', include(router.urls)),
]

# This automatically creates:
# GET    /api/books/          - List all books
# POST   /api/books/          - Create new book
# GET    /api/books/{id}/     - Retrieve book
# PUT    /api/books/{id}/     - Update book
# PATCH  /api/books/{id}/     - Partial update
# DELETE /api/books/{id}/     - Delete book
```

## Step-by-Step: Book API Implementation

Let's build a complete Book API from scratch.

### Step 1: Create Book Model

```python
# api/models.py
from django.db import models

class Book(models.Model):
    title = models.CharField(max_length=200)
    author = models.CharField(max_length=100)
    published_date = models.DateField()
    isbn = models.CharField(max_length=13, unique=True, blank=True, null=True)
    description = models.TextField(blank=True)
    created_at = models.DateTimeField(auto_now_add=True)
    updated_at = models.DateTimeField(auto_now=True)

    def __str__(self):
        return f"{self.title} by {self.author}"

    class Meta:
        ordering = ['-created_at']
        verbose_name = 'Book'
        verbose_name_plural = 'Books'
```

**Create and apply migration:**

```bash
python manage.py makemigrations
python manage.py migrate
```

### Step 2: Create Book Serializer

```python
# api/serializers.py
from rest_framework import serializers
from .models import Book

class BookSerializer(serializers.ModelSerializer):
    class Meta:
        model = Book
        fields = ['id', 'title', 'author', 'published_date', 'isbn', 'description', 'created_at', 'updated_at']
        read_only_fields = ['id', 'created_at', 'updated_at']
```

### Step 3: Create Views (Using ViewSet)

```python
# api/views.py
from rest_framework import viewsets
from .models import Book
from .serializers import BookSerializer

class BookViewSet(viewsets.ModelViewSet):
    queryset = Book.objects.all()
    serializer_class = BookSerializer
```

### Step 4: Configure URLs

```python
# api/urls.py
from django.urls import path, include
from rest_framework.routers import DefaultRouter
from . import views

router = DefaultRouter()
router.register(r'books', views.BookViewSet, basename='book')

urlpatterns = [
    path('', include(router.urls)),
]
```

```python
# core/urls.py
from django.contrib import admin
from django.urls import path, include

urlpatterns = [
    path('admin/', admin.site.urls),
    path('api/', include('api.urls')),
]
```

### Step 5: Register Model in Admin (Optional)

```python
# api/admin.py
from django.contrib import admin
from .models import Book

@admin.register(Book)
class BookAdmin(admin.ModelAdmin):
    list_display = ['title', 'author', 'published_date', 'created_at']
    list_filter = ['created_at', 'published_date']
    search_fields = ['title', 'author']
```

### Step 6: Test Your API

```bash
# Start server
python manage.py runserver

# Test with cURL (see CURL_GUIDE.md for details)
curl http://localhost:8000/api/books/

# Or visit in browser
# http://localhost:8000/api/books/
```

## Step-by-Step: Task API Implementation

Now let's adapt this to your existing Task model.

### Step 1: Update Task Model (if needed)

```python
# api/models.py
from django.db import models

class Task(models.Model):
    title = models.CharField(max_length=100)
    desc = models.TextField(null=True, blank=True)
    completed = models.BooleanField(default=False)
    created_at = models.DateTimeField(auto_now_add=True)
    updated_at = models.DateTimeField(auto_now=True)
    
    def __str__(self):
        return self.title

    class Meta:
        ordering = ['-created_at']
```

**Create migration if you added fields:**

```bash
python manage.py makemigrations
python manage.py migrate
```

### Step 2: Create Task Serializer

```python
# api/serializers.py
from rest_framework import serializers
from .models import Task

class TaskSerializer(serializers.ModelSerializer):
    class Meta:
        model = Task
        fields = ['id', 'title', 'desc', 'completed', 'created_at', 'updated_at']
        read_only_fields = ['id', 'created_at', 'updated_at']
```

### Step 3: Create Task ViewSet

```python
# api/views.py
from rest_framework import viewsets
from .models import Task
from .serializers import TaskSerializer

class TaskViewSet(viewsets.ModelViewSet):
    queryset = Task.objects.all()
    serializer_class = TaskSerializer
```

### Step 4: Add Task URLs

```python
# api/urls.py
from django.urls import path, include
from rest_framework.routers import DefaultRouter
from . import views

router = DefaultRouter()
router.register(r'books', views.BookViewSet, basename='book')
router.register(r'tasks', views.TaskViewSet, basename='task')  # Add this

urlpatterns = [
    path('', include(router.urls)),
]
```

### Step 5: Test Task API

```bash
# Create a task
curl -X POST http://localhost:8000/api/tasks/ \
  -H "Content-Type: application/json" \
  -d '{"title": "Learn DRF", "desc": "Complete Level 1", "completed": false}'

# List all tasks
curl http://localhost:8000/api/tasks/

# Get specific task
curl http://localhost:8000/api/tasks/1/

# Update task
curl -X PATCH http://localhost:8000/api/tasks/1/ \
  -H "Content-Type: application/json" \
  -d '{"completed": true}'

# Delete task
curl -X DELETE http://localhost:8000/api/tasks/1/
```

## Custom Response Formats

### Custom Response Structure

```python
# api/views.py
from rest_framework import viewsets, status
from rest_framework.response import Response
from .models import Task
from .serializers import TaskSerializer

class TaskViewSet(viewsets.ModelViewSet):
    queryset = Task.objects.all()
    serializer_class = TaskSerializer

    def create(self, request, *args, **kwargs):
        serializer = self.get_serializer(data=request.data)
        serializer.is_valid(raise_exception=True)
        self.perform_create(serializer)
        
        # Custom response
        return Response({
            'success': True,
            'message': 'Task created successfully',
            'data': serializer.data
        }, status=status.HTTP_201_CREATED)
```

### Custom List Response

```python
# api/views.py
class TaskViewSet(viewsets.ModelViewSet):
    queryset = Task.objects.all()
    serializer_class = TaskSerializer

    def list(self, request, *args, **kwargs):
        queryset = self.filter_queryset(self.get_queryset())
        serializer = self.get_serializer(queryset, many=True)
        
        return Response({
            'count': queryset.count(),
            'results': serializer.data
        })
```

## Error Handling

### Basic Error Handling

```python
# api/views.py
from rest_framework.views import exception_handler
from rest_framework.response import Response
from rest_framework import status

def custom_exception_handler(exc, context):
    response = exception_handler(exc, context)
    
    if response is not None:
        custom_response_data = {
            'error': {
                'status_code': response.status_code,
                'message': 'An error occurred',
                'details': response.data
            }
        }
        response.data = custom_response_data
    
    return response
```

**Add to settings.py:**

```python
# core/settings.py
REST_FRAMEWORK = {
    'EXCEPTION_HANDLER': 'api.views.custom_exception_handler',
}
```

### Validation Errors

DRF automatically handles validation errors. When serializer validation fails:

```python
# This is handled automatically by DRF
serializer = TaskSerializer(data=request.data)
if not serializer.is_valid():
    return Response(serializer.errors, status=status.HTTP_400_BAD_REQUEST)
```

## Testing with cURL

See [CURL_GUIDE.md](CURL_GUIDE.md) for detailed cURL examples. Here are quick examples:

```bash
# GET - List all tasks
curl http://localhost:8000/api/tasks/

# POST - Create task
curl -X POST http://localhost:8000/api/tasks/ \
  -H "Content-Type: application/json" \
  -d '{"title": "Test Task", "desc": "Description", "completed": false}'

# GET - Retrieve task
curl http://localhost:8000/api/tasks/1/

# PATCH - Update task
curl -X PATCH http://localhost:8000/api/tasks/1/ \
  -H "Content-Type: application/json" \
  -d '{"completed": true}'

# DELETE - Delete task
curl -X DELETE http://localhost:8000/api/tasks/1/
```

## Browsable API

DRF provides a browsable API interface. Visit any endpoint in your browser:

```
http://localhost:8000/api/tasks/
http://localhost:8000/api/books/
```

You'll see:
- HTML form to test POST/PUT/PATCH
- Response in JSON format
- Available actions
- Request/response examples

## Exercises

### Exercise 1: Complete Book API

1. Create Book model with fields: title, author, published_date, isbn
2. Create BookSerializer
3. Create BookViewSet
4. Configure URLs
5. Test all CRUD operations with cURL

### Exercise 2: Enhance Task API

1. Add `priority` field (choices: low, medium, high)
2. Add `due_date` field
3. Update TaskSerializer to include new fields
4. Test with cURL

### Exercise 3: Create Author API

1. Create Author model (name, email, bio)
2. Create AuthorSerializer and AuthorViewSet
3. Add to URLs
4. Test CRUD operations

### Exercise 4: Custom Responses

1. Modify TaskViewSet to return custom response format
2. Include success message and data
3. Test with cURL

### Exercise 5: Practice Problem - Product API

**Challenge**: Create a complete Product API with the following requirements:

1. **Model**: Create a `Product` model with:
   - `name` (CharField, max 200)
   - `description` (TextField, optional)
   - `price` (DecimalField, max 10 digits, 2 decimal places)
   - `stock` (IntegerField, default 0)
   - `is_available` (BooleanField, default True)
   - `created_at` and `updated_at` (auto timestamps)

2. **Serializer**: Create `ProductSerializer` with:
   - All fields included
   - `id`, `created_at`, `updated_at` as read-only
   - Validation: price must be > 0, stock must be >= 0

3. **ViewSet**: Create `ProductViewSet` using `ModelViewSet`

4. **URLs**: Configure routing with DefaultRouter

5. **Testing**: Test all CRUD operations:
   - Create a product
   - List all products
   - Retrieve a specific product
   - Update a product (full and partial)
   - Delete a product

**Solution Hints**:
```python
# Model
class Product(models.Model):
    name = models.CharField(max_length=200)
    description = models.TextField(blank=True)
    price = models.DecimalField(max_digits=10, decimal_places=2)
    stock = models.IntegerField(default=0)
    is_available = models.BooleanField(default=True)
    created_at = models.DateTimeField(auto_now_add=True)
    updated_at = models.DateTimeField(auto_now=True)

# Serializer validation
def validate_price(self, value):
    if value <= 0:
        raise serializers.ValidationError("Price must be greater than 0")
    return value
```

## Trivia

Test your understanding with these questions:

### Question 1
**What does ORM stand for?**
- [ ] Object-Relational Mapping
- [x] Object-Relational Mapping ✓
- [ ] Object-Record Mapping
- [ ] Object-Resource Mapping

**Explanation**: ORM = Object-Relational Mapping. It's the technique that converts Python objects to database records and vice versa.

### Question 2
**What is the purpose of `makemigrations`?**
- [x] Create migration files without changing database
- [ ] Apply migrations to database
- [ ] Delete old migrations
- [ ] View migration history

**Explanation**: `makemigrations` creates migration files that describe database changes, but doesn't actually modify the database. Use `migrate` to apply them.

### Question 3
**Which HTTP method is used to create a new resource?**
- [ ] GET
- [x] POST
- [ ] PUT
- [ ] DELETE

**Explanation**: POST is used to create new resources. GET retrieves, PUT/PATCH updates, DELETE removes.

### Question 4
**What does CORS stand for?**
- [ ] Cross-Origin Resource Sharing
- [x] Cross-Origin Resource Sharing ✓
- [ ] Cross-Origin Request Sharing
- [ ] Cross-Origin Response Sharing

**Explanation**: CORS = Cross-Origin Resource Sharing. It allows browsers to make requests to APIs on different domains.

### Question 5
**What is the difference between `null=True` and `blank=True` in Django models?**
- [ ] They are the same
- [x] `null=True` allows NULL in database, `blank=True` allows empty in forms
- [ ] `null=True` is for strings, `blank=True` is for numbers
- [ ] `null=True` is for forms, `blank=True` is for database

**Explanation**: 
- `null=True` = Database can store NULL
- `blank=True` = Forms/API can accept empty value
- For CharField/TextField, you often need both: `null=True, blank=True`

### Question 6
**What does `auto_now_add=True` do?**
- [ ] Updates the field every time the object is saved
- [x] Sets the field to current time only when object is first created
- [ ] Deletes the field after creation
- [ ] Makes the field optional

**Explanation**: `auto_now_add=True` sets the field to the current timestamp when the object is first created. Use `auto_now=True` to update on every save.

### Question 7
**What is a ViewSet?**
- [ ] A single view function
- [x] A class that provides CRUD operations for a model
- [ ] A URL pattern
- [ ] A serializer class

**Explanation**: ViewSet is a class that combines multiple view actions (list, create, retrieve, update, delete) into one class. It works with routers to automatically create URLs.

### Question 8
**What status code should you return when creating a new resource?**
- [ ] 200
- [x] 201
- [ ] 204
- [ ] 400

**Explanation**: 201 Created is the standard status code for successful resource creation. 200 is for successful retrieval, 204 for successful deletion.

### Question 9
**What is the purpose of `read_only_fields` in a serializer?**
- [ ] Fields that can only be read from database
- [x] Fields that are included in response but cannot be set via API
- [ ] Fields that are hidden from response
- [ ] Fields that are required

**Explanation**: `read_only_fields` are included when serializing (sending to client) but cannot be set when deserializing (receiving from client). Common for auto-generated fields like `id` and `created_at`.

### Question 10
**Why do we use virtual environments?**
- [ ] To make Python run faster
- [x] To isolate project dependencies and avoid conflicts
- [ ] To secure our code
- [ ] To organize files

**Explanation**: Virtual environments isolate Python packages for each project, preventing conflicts when different projects need different versions of the same package.

## Practice Problems

### Problem 1: Understanding the Flow

**Task**: Explain what happens when a client sends a POST request to create a book. Trace the request through:
1. URL routing
2. View processing
3. Serializer validation
4. Model creation
5. Response generation

**Solution**:
```
1. Client sends POST /api/books/ with JSON data
2. URL router matches pattern, calls BookViewSet.create()
3. View receives request, gets serializer
4. Serializer validates data (checks required fields, types)
5. If valid: serializer.save() creates Book object in database
6. Serializer converts Book object to JSON
7. View returns Response with JSON and 201 status code
8. Client receives response
```

### Problem 2: Debugging

**Scenario**: You created a Book model and serializer, but when you try to create a book via API, you get a 400 error saying "title is required" even though you're sending it.

**Possible Issues**:
1. Field name mismatch (sending `book_title` but model expects `title`)
2. Serializer not including `title` in fields
3. Content-Type header missing (not `application/json`)
4. JSON syntax error in request body

**How to Debug**:
```python
# Check serializer fields
print(BookSerializer().fields.keys())

# Check request data
print(request.data)

# Check validation errors
serializer = BookSerializer(data=request.data)
if not serializer.is_valid():
    print(serializer.errors)
```

### Problem 3: Model Design

**Task**: Design a model for a Blog Post with these requirements:
- Title (required, max 200 chars)
- Content (required, can be long)
- Author name (required)
- Published date (optional, can be set later)
- Is published (boolean, default False)
- View count (integer, starts at 0)
- Created and updated timestamps

**Solution**:
```python
class BlogPost(models.Model):
    title = models.CharField(max_length=200)
    content = models.TextField()
    author_name = models.CharField(max_length=100)
    published_date = models.DateTimeField(null=True, blank=True)
    is_published = models.BooleanField(default=False)
    view_count = models.IntegerField(default=0)
    created_at = models.DateTimeField(auto_now_add=True)
    updated_at = models.DateTimeField(auto_now=True)
    
    def __str__(self):
        return self.title
```

### Problem 4: Serializer Customization

**Task**: Create a serializer that:
- Includes all fields from Book model
- Adds a computed field `is_recent` (True if created in last 7 days)
- Excludes `updated_at` from response
- Makes `isbn` optional when creating

**Solution**:
```python
from datetime import timedelta
from django.utils import timezone

class BookSerializer(serializers.ModelSerializer):
    is_recent = serializers.SerializerMethodField()
    
    class Meta:
        model = Book
        fields = ['id', 'title', 'author', 'published_date', 'isbn', 
                  'created_at', 'is_recent']
        read_only_fields = ['id', 'created_at']
    
    def get_is_recent(self, obj):
        seven_days_ago = timezone.now() - timedelta(days=7)
        return obj.created_at >= seven_days_ago
```

## Add-ons

### Add-on 1: Custom Response Format

Implement a consistent response format across all endpoints.

### Add-on 2: Model Validation

Add custom validation to models:

```python
# api/models.py
from django.core.exceptions import ValidationError

class Task(models.Model):
    # ... fields ...
    
    def clean(self):
        if self.completed and not self.desc:
            raise ValidationError('Completed tasks must have a description')
    
    def save(self, *args, **kwargs):
        self.full_clean()
        super().save(*args, **kwargs)
```

### Add-on 3: Serializer Validation

```python
# api/serializers.py
class TaskSerializer(serializers.ModelSerializer):
    # ... fields ...
    
    def validate_title(self, value):
        if len(value) < 3:
            raise serializers.ValidationError("Title must be at least 3 characters")
        return value
    
    def validate(self, data):
        if data.get('completed') and not data.get('desc'):
            raise serializers.ValidationError("Completed tasks must have a description")
        return data
```

## Next Steps

Congratulations! You've completed Level 1. You now know:

- ✅ REST fundamentals
- ✅ Django project structure
- ✅ Models and migrations
- ✅ Serializers
- ✅ Views (APIView, Generic Views, ViewSets)
- ✅ URL routing
- ✅ Building CRUD APIs

**Ready for Level 2?** Continue to [Level 2: Intermediate](LEVEL_2_INTERMEDIATE.md) to learn about authentication, permissions, filtering, and pagination!

---

**Resources:**
- [DRF Official Tutorial](https://www.django-rest-framework.org/tutorial/quickstart/)
- [Django Models Documentation](https://docs.djangoproject.com/en/stable/topics/db/models/)
- [DRF Serializers Guide](https://www.django-rest-framework.org/api-guide/serializers/)

