# Level 2: Intermediate - Authentication, Permissions & Querysets

## Goal

Build secure APIs with robust authentication, permissions, filtering, searching, ordering, and pagination. By the end of this level, you'll be able to create production-ready, secure REST APIs.

## Table of Contents

1. [Django Authentication System](#django-authentication-system)
2. [DRF Authentication Classes](#drf-authentication-classes)
3. [JWT Authentication Setup](#jwt-authentication-setup)
4. [Permissions](#permissions)
5. [Filtering](#filtering)
6. [Searching](#searching)
7. [Ordering](#ordering)
8. [Pagination](#pagination)
9. [Step-by-Step: Secure Book API](#step-by-step-secure-book-api)
10. [Step-by-Step: Secure Task API](#step-by-step-secure-task-api)
11. [UserProfile API](#userprofile-api)
12. [Throttling](#throttling)
13. [Testing Authenticated APIs](#testing-authenticated-apis)
14. [Exercises](#exercises)
15. [Add-ons](#add-ons)

## Django Authentication System

### Why Authentication?

**The Problem**: Without authentication, anyone can access your API and modify data. This is a security risk!

**Real-world analogy**: 
- **No authentication** = House with no locks (anyone can enter)
- **With authentication** = House with keys (only authorized people enter)

**Why it matters**:
- **Security**: Protect user data
- **Personalization**: Show users only their data
- **Accountability**: Track who did what
- **Access control**: Limit what users can do

### Understanding Django Users

Django comes with a built-in User model that handles user accounts:

**What is a User model?**
- Represents a person using your application
- Stores login credentials (username, password)
- Tracks user information (email, name)
- Manages permissions and access levels

**Why use Django's User model?**
- **Ready-made**: No need to build from scratch
- **Secure**: Password hashing built-in
- **Tested**: Used by millions of applications
- **Flexible**: Can be extended for custom needs

```python
from django.contrib.auth.models import User

# User fields
user.username  # Username
user.email     # shed password
user.first_nameEmail
user.password  # Ha
user.last_name
user.is_staff  # Admin access
user.is_active # Account status
user.is_superuser
user.date_joined
```

### Creating Users

**Why multiple methods?**
- Different methods serve different purposes
- Some hash passwords (secure), others don't (insecure)
- Always use methods that hash passwords!

```python
# In Django shell: python manage.py shell

from django.contrib.auth.models import User

# Method 1: create_user (recommended - hashes password)
user = User.objects.create_user(
    username='john',
    email='john@example.com',
    password='securepassword123'
)
```

**Why `create_user`?**
- **Password hashing**: Converts password to secure hash (can't be reversed)
- **Validation**: Checks username/email format
- **Default values**: Sets sensible defaults (is_active=True)
- **Security**: Never store plain text passwords!

**What is password hashing?**
- Plain password: `"mypassword123"` ❌ (stored as-is, anyone can read)
- Hashed password: `"pbkdf2_sha256$..."` ✅ (one-way encryption, can't be reversed)
- When user logs in, Django hashes their input and compares with stored hash

```python
# Method 2: create_superuser
admin = User.objects.create_superuser(
    username='admin',
    email='admin@example.com',
    password='admin123'
)
```

**Why `create_superuser`?**
- Creates user with admin privileges (`is_staff=True`, `is_superuser=True`)
- Can access Django admin panel
- Has all permissions by default
- Use for site administrators

```python
# Method 3: Using User.objects.create (not recommended - password not hashed)
```

**Why NOT use `User.objects.create`?**
- **Security risk**: Password stored in plain text!
- No validation
- No password hashing
- **Never use this for passwords!**

### Custom User Model

**Why create a custom user model?**

**The Problem**: Django's default User model has limited fields (username, email, password). What if you need:
- Phone numbers
- Profile pictures
- Bio/description
- Custom fields

**The Solution**: Extend Django's User model with your own fields.

**When to use custom user model?**
- **Production apps**: Almost always
- **Need extra fields**: Phone, avatar, bio, etc.
- **Different login**: Email instead of username
- **Before first migration**: Must set before creating database

**Why before first migration?**
- Changing user model after migration is very difficult
- Requires database restructuring
- Can cause data loss
- **Always decide early!**

```python
# api/models.py
from django.contrib.auth.models import AbstractUser
from django.db import models

class CustomUser(AbstractUser):
    phone_number = models.CharField(max_length=15, blank=True)
    bio = models.TextField(blank=True)
    avatar = models.ImageField(upload_to='avatars/', blank=True, null=True)
    created_at = models.DateTimeField(auto_now_add=True)

    def __str__(self):
        return self.username
```

**Explanation**:
- **`AbstractUser`**: Base class with all default User fields (username, email, password, etc.)
- **Extend it**: Add your custom fields
- **Inherits everything**: All User functionality still works
- **Add custom fields**: Phone, bio, avatar, etc.

**Update settings.py:**

```python
# core/settings.py
AUTH_USER_MODEL = 'api.CustomUser'
```

**What this does**:
- Tells Django to use your custom user model
- Replaces default User model everywhere
- Must be set before first migration!

**Important**: Set this before your first migration! Once migrations are created, changing is very difficult.

## DRF Authentication Classes

### Authentication vs Authorization

**These are two different but related concepts:**

- **Authentication**: Who are you? (Identity)
  - Verifies user's identity
  - "Are you really John?"
  - Examples: Login, password check, token verification
  
- **Authorization**: What can you do? (Permissions)
  - Determines what user can access
  - "Can John edit this book?"
  - Examples: Admin only, owner only, read-only

**Real-world analogy**:
- **Authentication** = Showing ID at entrance (proving who you are)
- **Authorization** = Security guard checking if you have access to VIP area (what you can do)

**Why both?**
- Authentication without authorization = Everyone can do everything
- Authorization without authentication = Can't identify users
- **Need both** for secure APIs!

**The Flow**:
```
1. User sends credentials → Authentication (who are you?)
2. If authenticated → Check permissions → Authorization (what can you do?)
3. If authorized → Allow access
4. If not authorized → Deny access (403 Forbidden)
```

### DRF Authentication Types

#### 1. Session Authentication

**What is Session Authentication?**

Uses Django's session framework. Stores authentication state on the server.

**How it works**:
1. User logs in with username/password
2. Server creates a session (stored in database/cache)
3. Server sends session ID in cookie to browser
4. Browser sends cookie with each request
5. Server checks session to identify user

**Why use it?**
- **Simple**: Built into Django
- **Secure**: Session stored server-side
- **Good for**: Web apps (same domain)
- **Automatic**: Browser handles cookies

**Limitations**:
- **Same-origin only**: Doesn't work well with mobile apps
- **Cookie-based**: Requires browser cookie support
- **Server storage**: Needs database/cache for sessions

**When to use**:
- Web applications (React, Vue on same domain)
- Traditional web apps
- When you control both frontend and backend

```python
# core/settings.py
REST_FRAMEWORK = {
    'DEFAULT_AUTHENTICATION_CLASSES': [
        'rest_framework.authentication.SessionAuthentication',
    ],
}
```

**Why this configuration?**
- Sets default authentication for all views
- Can override per-view if needed
- Multiple authentication classes can be used (tries each until one works)

#### 2. Token Authentication

Simple token-based authentication. Good for client-server setups.

```bash
# Install
pip install djangorestframework-simplejwt
```

```python
# core/settings.py
INSTALLED_APPS = [
    # ...
    'rest_framework',
    'rest_framework_simplejwt',  # Add this
]

REST_FRAMEWORK = {
    'DEFAULT_AUTHENTICATION_CLASSES': [
        'rest_framework_simplejwt.authentication.JWTAuthentication',
    ],
}
```

#### 3. JWT Authentication (Recommended)

**What is JWT?**

**JWT** = JSON Web Token

**How it works**:
1. User logs in with username/password
2. Server creates a JWT token (contains user info)
3. Server sends token to client
4. Client stores token (localStorage, memory)
5. Client sends token with each request in header
6. Server verifies token and extracts user info

**Why JWT is popular**:
- **Stateless**: No server-side storage needed
- **Scalable**: Works across multiple servers
- **Mobile-friendly**: Works with mobile apps
- **Industry standard**: Used by major companies
- **Self-contained**: Token has all user info

**JWT Structure**:
```
Header.Payload.Signature

Example:
eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJ1c2VyX2lkIjoxfQ.signature
```

**Parts**:
- **Header**: Algorithm and token type
- **Payload**: User data (ID, username, etc.)
- **Signature**: Ensures token wasn't tampered with

**Why stateless matters**:
- **Session auth**: Server stores session → needs database lookup
- **JWT auth**: Token contains info → no database lookup needed
- **Result**: Faster, more scalable

**Security considerations**:
- Tokens expire (access token: 1 hour, refresh token: 1 day)
- Tokens can be blacklisted if compromised
- Always use HTTPS in production (tokens in headers)

**When to use JWT**:
- ✅ Mobile apps (iOS, Android)
- ✅ Single Page Applications (SPA)
- ✅ Microservices
- ✅ APIs used by multiple clients
- ❌ Not ideal for traditional server-rendered apps

## JWT Authentication Setup

### Step 1: Install Package

```bash
pip install djangorestframework-simplejwt
pip freeze > requirements.txt
```

### Step 2: Add to INSTALLED_APPS

```python
# core/settings.py
INSTALLED_APPS = [
    # ...
    'rest_framework',
    'rest_framework_simplejwt',
    'api',
]
```

### Step 3: Configure JWT Settings

```python
# core/settings.py
from datetime import timedelta

REST_FRAMEWORK = {
    'DEFAULT_AUTHENTICATION_CLASSES': [
        'rest_framework_simplejwt.authentication.JWTAuthentication',
    ],
    'DEFAULT_PERMISSION_CLASSES': [
        'rest_framework.permissions.IsAuthenticated',
    ],
}

SIMPLE_JWT = {
    'ACCESS_TOKEN_LIFETIME': timedelta(minutes=60),
    'REFRESH_TOKEN_LIFETIME': timedelta(days=1),
    'ROTATE_REFRESH_TOKENS': True,
    'BLACKLIST_AFTER_ROTATION': True,
    'UPDATE_LAST_LOGIN': True,
    'ALGORITHM': 'HS256',
    'SIGNING_KEY': SECRET_KEY,
    'AUTH_HEADER_TYPES': ('Bearer',),
    'AUTH_HEADER_NAME': 'HTTP_AUTHORIZATION',
    'USER_ID_FIELD': 'id',
    'USER_ID_CLAIM': 'user_id',
}
```

**Explanation of JWT Settings**:

- **`ACCESS_TOKEN_LIFETIME`**: How long access token is valid (60 minutes)
  - **Why short?**: If stolen, expires quickly
  - **Trade-off**: Security vs convenience
  
- **`REFRESH_TOKEN_LIFETIME`**: How long refresh token is valid (1 day)
  - **Why longer?**: Used to get new access tokens
  - **More secure**: Stored more securely, used less often
  
- **`ROTATE_REFRESH_TOKENS`**: Issue new refresh token when used
  - **Why?**: If old token stolen, it becomes invalid
  - **Security**: Limits damage from token theft
  
- **`BLACKLIST_AFTER_ROTATION`**: Add old refresh token to blacklist
  - **Why?**: Prevents reuse of old tokens
  - **Security**: One-time use tokens
  
- **`UPDATE_LAST_LOGIN`**: Update user's last login time
  - **Why?**: Track user activity, security monitoring
  
- **`ALGORITHM`**: How token is signed ('HS256' = HMAC SHA-256)
  - **Why HS256?**: Fast, secure, widely supported
  - **Alternative**: RS256 (asymmetric, more secure but slower)
  
- **`SIGNING_KEY`**: Secret key to sign tokens (use Django SECRET_KEY)
  - **Why secret?**: Anyone with key can create valid tokens
  - **Security**: Never expose this key!
  
- **`AUTH_HEADER_TYPES`**: Token type in header ('Bearer')
  - **Why Bearer?**: Standard OAuth2 format
  - **Usage**: `Authorization: Bearer <token>`
  
- **`USER_ID_FIELD`**: Which User model field to use ('id')
- **`USER_ID_CLAIM`**: Where to store user ID in token ('user_id')
```

### Step 4: Add JWT URLs

```python
# core/urls.py
from django.contrib import admin
from django.urls import path, include
from rest_framework_simplejwt.views import (
    TokenObtainPairView,
    TokenRefreshView,
    TokenVerifyView,
)

urlpatterns = [
    path('admin/', admin.site.urls),
    path('api/', include('api.urls')),
    # JWT endpoints
    path('api/token/', TokenObtainPairView.as_view(), name='token_obtain_pair'),
    path('api/token/refresh/', TokenRefreshView.as_view(), name='token_refresh'),
    path('api/token/verify/', TokenVerifyView.as_view(), name='token_verify'),
]
```

### Step 5: Create User Registration View

```python
# api/serializers.py
from rest_framework import serializers
from django.contrib.auth.models import User

class UserRegistrationSerializer(serializers.ModelSerializer):
    password = serializers.CharField(write_only=True, min_length=8)
    password_confirm = serializers.CharField(write_only=True)

    class Meta:
        model = User
        fields = ['username', 'email', 'password', 'password_confirm', 'first_name', 'last_name']

    def validate(self, data):
        if data['password'] != data['password_confirm']:
            raise serializers.ValidationError("Passwords don't match")
        return data

    def create(self, validated_data):
        validated_data.pop('password_confirm')
        user = User.objects.create_user(**validated_data)
        return user
```

```python
# api/views.py
from rest_framework import generics, status
from rest_framework.response import Response
from rest_framework_simplejwt.tokens import RefreshToken
from django.contrib.auth.models import User
from .serializers import UserRegistrationSerializer

class UserRegistrationView(generics.CreateAPIView):
    queryset = User.objects.all()
    serializer_class = UserRegistrationSerializer
    permission_classes = []  # Allow anyone to register

    def create(self, request, *args, **kwargs):
        serializer = self.get_serializer(data=request.data)
        serializer.is_valid(raise_exception=True)
        user = serializer.save()
        
        # Generate tokens
        refresh = RefreshToken.for_user(user)
        
        return Response({
            'user': serializer.data,
            'refresh': str(refresh),
            'access': str(refresh.access_token),
        }, status=status.HTTP_201_CREATED)
```

```python
# api/urls.py
from django.urls import path
from . import views

urlpatterns = [
    # ... other URLs ...
    path('register/', views.UserRegistrationView.as_view(), name='register'),
]
```

### Step 6: Test JWT Authentication

```bash
# Register user
curl -X POST http://localhost:8000/api/register/ \
  -H "Content-Type: application/json" \
  -d '{
    "username": "testuser",
    "email": "test@example.com",
    "password": "testpass123",
    "password_confirm": "testpass123"
  }'

# Get token
curl -X POST http://localhost:8000/api/token/ \
  -H "Content-Type: application/json" \
  -d '{"username": "testuser", "password": "testpass123"}'

# Use token
TOKEN="your_access_token_here"
curl -H "Authorization: Bearer $TOKEN" \
  http://localhost:8000/api/tasks/
```

## Permissions

### What are Permissions?

**Permissions** determine what authenticated users can do.

**Why Permissions?**
- **Security**: Not all users should do everything
- **Data protection**: Users should only see/edit their own data
- **Role-based access**: Admins can do more than regular users
- **Compliance**: Meet security requirements

**Real-world analogy**:
- **No permissions** = Everyone has master key (bad!)
- **With permissions** = Different keys for different access levels (good!)

**Permission Levels**:
1. **Public**: Anyone can access (AllowAny)
2. **Authenticated**: Must be logged in (IsAuthenticated)
3. **Owner**: Can only access own data (IsOwnerOrReadOnly)
4. **Admin**: Only staff can access (IsAdminUser)

### Built-in Permission Classes

#### 1. AllowAny

**What it does**: Anyone can access, no authentication required.

**Why use it?**
- Public data (blog posts, product listings)
- Registration/login endpoints
- Public APIs
- When you want maximum accessibility

**Security note**: Use carefully! Only for truly public data.

**When to use**:
- ✅ Public blog posts
- ✅ Product catalogs
- ✅ Public user profiles
- ❌ User data
- ❌ Admin functions

```python
from rest_framework.permissions import AllowAny

class BookViewSet(viewsets.ModelViewSet):
    permission_classes = [AllowAny]
    # ...
```

#### 2. IsAuthenticated

**What it does**: User must be logged in (authenticated) to access.

**Why use it?**
- **Protect data**: Only logged-in users can access
- **User-specific content**: Show personalized data
- **Account required**: Force users to create accounts
- **Security**: Basic protection for most endpoints

**When to use**:
- ✅ User's own tasks/books
- ✅ User profiles
- ✅ Private data
- ✅ Any endpoint requiring user identity

**What happens if not authenticated?**
- Returns `401 Unauthorized`
- Client must login first
- Token/session required

```python
from rest_framework.permissions import IsAuthenticated

class BookViewSet(viewsets.ModelViewSet):
    permission_classes = [IsAuthenticated]
    # ...
```

**Why this configuration?**
- Applies to all actions (list, create, retrieve, update, delete)
- Can override per-action if needed
- Most common permission for user data

#### 3. IsAdminUser

User must be staff/admin.

```python
from rest_framework.permissions import IsAdminUser

class BookViewSet(viewsets.ModelViewSet):
    permission_classes = [IsAdminUser]
    # ...
```

#### 4. IsAuthenticatedOrReadOnly

**What it does**: 
- **Authenticated users**: Can do everything (read, create, update, delete)
- **Unauthenticated users**: Can only read (GET requests)

**Why use it?**
- **Public viewing**: Anyone can see data
- **Protected editing**: Only logged-in users can modify
- **Common pattern**: Blog posts, comments, products
- **User engagement**: Encourage registration to participate

**Real-world example**:
- Blog: Anyone can read posts, only logged-in users can comment
- Products: Anyone can browse, only logged-in users can add to cart
- Comments: Anyone can read, only logged-in users can post

**When to use**:
- ✅ Public content with user contributions
- ✅ Read-mostly APIs
- ✅ When you want public access but protected writes
- ❌ Private/sensitive data

```python
from rest_framework.permissions import IsAuthenticatedOrReadOnly

class BookViewSet(viewsets.ModelViewSet):
    permission_classes = [IsAuthenticatedOrReadOnly]
    # ...
```

**What this means**:
- `GET /api/books/` → Anyone can access ✅
- `POST /api/books/` → Must be authenticated ✅
- `PUT /api/books/1/` → Must be authenticated ✅
- `DELETE /api/books/1/` → Must be authenticated ✅

### Custom Permissions

#### IsOwnerOrReadOnly

Only the owner can edit/delete, others can read.

```python
# api/permissions.py
from rest_framework import permissions

class IsOwnerOrReadOnly(permissions.BasePermission):
    """
    Custom permission to only allow owners to edit/delete objects.
    """

    def has_object_permission(self, request, view, obj):
        # Read permissions for any request
        if request.method in permissions.SAFE_METHODS:
            return True

        # Write permissions only to the owner
        return obj.owner == request.user
```

**Use in views:**

```python
# api/views.py
from .permissions import IsOwnerOrReadOnly

class BookViewSet(viewsets.ModelViewSet):
    permission_classes = [IsAuthenticated, IsOwnerOrReadOnly]
    
    def perform_create(self, serializer):
        serializer.save(owner=self.request.user)
```

### Permission at View Level

```python
# Different permissions for different actions
class BookViewSet(viewsets.ModelViewSet):
    def get_permissions(self):
        if self.action == 'create':
            permission_classes = [IsAuthenticated]
        elif self.action in ['update', 'partial_update', 'destroy']:
            permission_classes = [IsAuthenticated, IsOwnerOrReadOnly]
        else:
            permission_classes = [AllowAny]
        return [permission() for permission in permission_classes]
```

## Filtering

### What is Filtering?

**Filtering** allows clients to request specific subsets of data based on criteria.

**The Problem**: Without filtering, clients get ALL data, even if they only need a subset.

**Example**:
- Without filtering: `GET /api/books/` → Returns 10,000 books ❌
- With filtering: `GET /api/books/?author=John` → Returns only John's books ✅

**Why Filtering?**
- **Performance**: Return only needed data (faster queries)
- **User experience**: Users find what they're looking for
- **Bandwidth**: Less data transferred
- **Database efficiency**: Queries only relevant records

**Real-world analogy**:
- **No filtering** = Library with no organization (find book by searching all shelves)
- **With filtering** = Library with sections (go directly to "Fiction" section)

**Common Filter Types**:
- **Exact match**: `?author=John` (author exactly equals "John")
- **Contains**: `?title__icontains=django` (title contains "django")
- **Range**: `?price__gte=10&price__lte=50` (price between 10 and 50)
- **Date range**: `?created_after=2024-01-01` (created after date)

### Install django-filter

**Why django-filter?**
- Makes filtering easy and powerful
- Supports complex filters
- Works seamlessly with DRF
- Handles edge cases

```bash
pip install django-filter
```

### Configure in settings.py

```python
# core/settings.py
INSTALLED_APPS = [
    # ...
    'django_filters',
    'rest_framework',
]

REST_FRAMEWORK = {
    'DEFAULT_FILTER_BACKENDS': [
        'django_filters.rest_framework.DjangoFilterBackend',
        'rest_framework.filters.SearchFilter',
        'rest_framework.filters.OrderingFilter',
    ],
}
```

### Basic Filtering

**How it works**:
1. Client adds query parameters to URL: `?author=John`
2. Django Filter intercepts request
3. Applies filters to queryset
4. Returns filtered results

**Why `filter_backends`?**
- Tells DRF which filtering system to use
- Can use multiple backends (filtering + searching + ordering)
- Each backend handles different aspects

```python
# api/views.py
from django_filters.rest_framework import DjangoFilterBackend
from rest_framework import viewsets, filters

class BookViewSet(viewsets.ModelViewSet):
    queryset = Book.objects.all()
    serializer_class = BookSerializer
    filter_backends = [DjangoFilterBackend]  # Enable filtering
    filterset_fields = ['author', 'published_date']  # Which fields can be filtered
```

**Explanation**:
- **`filter_backends`**: List of filtering systems to use
- **`filterset_fields`**: Simple list of fields that can be filtered
- **Exact match**: `?author=John` finds books where author exactly equals "John"

**Usage:**

```bash
# Filter by author
curl http://localhost:8000/api/books/?author=John%20Doe
```

**What happens**:
- URL parameter `author=John%20Doe` (URL-encoded space)
- Django Filter finds books where `author` field equals "John Doe"
- Returns only matching books

```bash
# Filter by multiple fields
curl http://localhost:8000/api/books/?author=John%20Doe&published_date=2023-01-01
```

**What happens**:
- Both filters applied: `author=John Doe` AND `published_date=2023-01-01`
- Returns books matching BOTH criteria
- Filters are combined with AND logic

### Advanced Filtering with FilterSet

```python
# api/filters.py
import django_filters
from .models import Book

class BookFilter(django_filters.FilterSet):
    author = django_filters.CharFilter(lookup_expr='icontains')
    published_after = django_filters.DateFilter(field_name='published_date', lookup_expr='gte')
    published_before = django_filters.DateFilter(field_name='published_date', lookup_expr='lte')

    class Meta:
        model = Book
        fields = ['author', 'published_after', 'published_before']
```

```python
# api/views.py
from .filters import BookFilter

class BookViewSet(viewsets.ModelViewSet):
    queryset = Book.objects.all()
    serializer_class = BookSerializer
    filterset_class = BookFilter
```

## Searching

### What is Searching?

**Searching** allows users to find data by searching across multiple fields with a single query.

**Difference from Filtering**:
- **Filtering**: Exact matches on specific fields (`?author=John`)
- **Searching**: Partial matches across multiple fields (`?search=john` finds in title, author, description)

**Why Searching?**
- **User-friendly**: Users don't need to know exact field names
- **Flexible**: Search across multiple fields at once
- **Common pattern**: Search boxes in web apps
- **Better UX**: "Search for anything" vs "Filter by specific field"

**Real-world analogy**:
- **Filtering** = Library catalog with specific fields (author, genre, year)
- **Searching** = Google search (type anything, finds matches anywhere)

### SearchFilter

**How it works**:
1. User types search term: `?search=django`
2. DRF searches across specified fields
3. Returns records matching search term in ANY field
4. Uses case-insensitive partial matching by default

```python
# api/views.py
from rest_framework import viewsets, filters

class BookViewSet(viewsets.ModelViewSet):
    queryset = Book.objects.all()
    serializer_class = BookSerializer
    filter_backends = [filters.SearchFilter]  # Enable searching
    search_fields = ['title', 'author', 'description']  # Fields to search
```

**Explanation**:
- **`SearchFilter`**: DRF's built-in search backend (no extra package needed)
- **`search_fields`**: List of fields to search in
- **Default behavior**: Case-insensitive, partial matching
- **OR logic**: Matches if search term found in ANY field

**Usage:**

```bash
# Search in title, author, description
curl http://localhost:8000/api/books/?search=django
```

**What happens**:
- Searches for "django" in `title`, `author`, and `description` fields
- Returns books where ANY field contains "django"
- Case-insensitive: "Django", "DJANGO", "django" all match
- Partial match: "Django for Beginners" matches "django"

### Search Lookup Expressions

```python
class BookViewSet(viewsets.ModelViewSet):
    search_fields = [
        '^title',      # Starts with
        '=isbn',       # Exact match
        '@description', # Full text search (PostgreSQL only)
        '$title',      # Regex search
    ]
```

## Ordering

### What is Ordering?

**Ordering** controls the sequence in which results are returned.

**The Problem**: Without ordering, results come in unpredictable order (usually database insertion order).

**Why Ordering?**
- **User experience**: Show most relevant/newest first
- **Consistency**: Same query always returns same order
- **Flexibility**: Let users choose sort order
- **Default behavior**: Set sensible default (newest first, alphabetical, etc.)

**Real-world analogy**:
- **No ordering** = Bookshelf with random order
- **With ordering** = Bookshelf organized by date (newest first) or alphabetically

### OrderingFilter

**How it works**:
1. Client specifies order: `?ordering=-created_at`
2. DRF sorts queryset by specified field
3. `-` prefix means descending (newest first)
4. No prefix means ascending (oldest first)

```python
# api/views.py
from rest_framework import viewsets, filters

class BookViewSet(viewsets.ModelViewSet):
    queryset = Book.objects.all()
    serializer_class = BookSerializer
    filter_backends = [filters.OrderingFilter]  # Enable ordering
    ordering_fields = ['title', 'author', 'published_date', 'created_at']  # Allowed fields
    ordering = ['-created_at']  # Default ordering (newest first)
```

**Explanation**:
- **`OrderingFilter`**: DRF's built-in ordering backend
- **`ordering_fields`**: Which fields users can sort by (security: prevents sorting by sensitive fields)
- **`ordering`**: Default order if user doesn't specify
- **`-` prefix**: Descending order (newest/highest first)
- **No prefix**: Ascending order (oldest/lowest first)

**Usage:**

```bash
# Order by title (ascending: A-Z)
curl http://localhost:8000/api/books/?ordering=title
```

**What happens**: Books sorted alphabetically by title (A to Z)

```bash
# Order by created_at (descending: newest first)
curl http://localhost:8000/api/books/?ordering=-created_at
```

**What happens**: Books sorted by creation date, newest first (most recent at top)

```bash
# Multiple fields: First by author, then by published_date (descending)
curl http://localhost:8000/api/books/?ordering=author,-published_date
```

**What happens**: 
1. First sorts by author (A-Z)
2. Within same author, sorts by published_date (newest first)
3. Useful for grouped sorting

## Pagination

### What is Pagination?

**Pagination** splits large result sets into smaller "pages" of data.

**The Problem**: Without pagination, returning 10,000 records causes:
- **Slow response**: Large JSON payload
- **High memory**: Server/client must handle all data
- **Poor UX**: User overwhelmed with data
- **Database load**: Fetching all records at once

**Why Pagination?**
- **Performance**: Faster responses (only fetch what's needed)
- **User experience**: Manageable chunks of data
- **Scalability**: Works with millions of records
- **Bandwidth**: Less data transferred
- **Mobile-friendly**: Smaller payloads for mobile networks

**Real-world analogy**:
- **No pagination** = Dumping entire library catalog on one page
- **With pagination** = Showing 10 books per page with "Next" button

**Common Pagination Types**:
1. **Page Number**: `?page=2` (most common, user-friendly)
2. **Limit/Offset**: `?limit=20&offset=40` (flexible, good for APIs)
3. **Cursor**: `?cursor=abc123` (best for large datasets, stable)

### PageNumberPagination

**How it works**:
1. Client requests page: `?page=2`
2. Server calculates which records to return (records 11-20 for page 2)
3. Returns page data + metadata (total count, next/previous links)
4. Client can navigate to next/previous pages

**Why Page Number?**
- **User-friendly**: Easy to understand ("Go to page 2")
- **Simple**: Just specify page number
- **Good for**: Web UIs, predictable navigation
- **Limitation**: Can be slow with very large datasets (offset calculation)

```python
# core/settings.py
REST_FRAMEWORK = {
    'DEFAULT_PAGINATION_CLASS': 'rest_framework.pagination.PageNumberPagination',
    'PAGE_SIZE': 10,  # Records per page
}
```

**Explanation**:
- **`DEFAULT_PAGINATION_CLASS`**: Applies to all list endpoints
- **`PAGE_SIZE`**: How many records per page
- **Can override**: Per-view if needed

**Response format:**

```json
{
    "count": 100,  // Total number of records
    "next": "http://localhost:8000/api/books/?page=2",  // URL for next page
    "previous": null,  // URL for previous page (null if on first page)
    "results": [...]  // Actual data (10 records)
}
```

**Why this format?**
- **`count`**: Total records (for progress indicators: "Showing 1-10 of 100")
- **`next`**: Direct link to next page (convenient for clients)
- **`previous`**: Direct link to previous page
- **`results`**: Actual data array

**Usage:**

```bash
curl http://localhost:8000/api/books/?page=2
```

**What happens**: Returns records 11-20 (page 2, assuming 10 per page)

### LimitOffsetPagination

```python
# api/pagination.py
from rest_framework.pagination import LimitOffsetPagination

class BookLimitOffsetPagination(LimitOffsetPagination):
    default_limit = 10
    limit_query_param = 'limit'
    offset_query_param = 'offset'
    max_limit = 100
```

```python
# api/views.py
from .pagination import BookLimitOffsetPagination

class BookViewSet(viewsets.ModelViewSet):
    pagination_class = BookLimitOffsetPagination
    # ...
```

**Usage:**

```bash
curl http://localhost:8000/api/books/?limit=20&offset=40
```

### CursorPagination

Best for large datasets. Provides stable, consistent pagination.

```python
# api/pagination.py
from rest_framework.pagination import CursorPagination

class BookCursorPagination(CursorPagination):
    page_size = 10
    ordering = '-created_at'
    cursor_query_param = 'cursor'
```

## Step-by-Step: Secure Book API

Let's upgrade the Book API with authentication and filtering.

### Step 1: Update Book Model

```python
# api/models.py
from django.db import models
from django.contrib.auth.models import User

class Book(models.Model):
    title = models.CharField(max_length=200)
    author = models.CharField(max_length=100)
    published_date = models.DateField()
    isbn = models.CharField(max_length=13, unique=True, blank=True, null=True)
    description = models.TextField(blank=True)
    owner = models.ForeignKey(User, on_delete=models.CASCADE, related_name='books')
    created_at = models.DateTimeField(auto_now_add=True)
    updated_at = models.DateTimeField(auto_now=True)

    def __str__(self):
        return f"{self.title} by {self.author}"

    class Meta:
        ordering = ['-created_at']
```

**Create migration:**

```bash
python manage.py makemigrations
python manage.py migrate
```

### Step 2: Update Book Serializer

```python
# api/serializers.py
from rest_framework import serializers
from .models import Book

class BookSerializer(serializers.ModelSerializer):
    owner_username = serializers.CharField(source='owner.username', read_only=True)

    class Meta:
        model = Book
        fields = ['id', 'title', 'author', 'published_date', 'isbn', 'description', 
                  'owner', 'owner_username', 'created_at', 'updated_at']
        read_only_fields = ['id', 'owner', 'created_at', 'updated_at']
```

### Step 3: Create Filter

```python
# api/filters.py
import django_filters
from .models import Book

class BookFilter(django_filters.FilterSet):
    author = django_filters.CharFilter(lookup_expr='icontains')
    published_after = django_filters.DateFilter(field_name='published_date', lookup_expr='gte')
    published_before = django_filters.DateFilter(field_name='published_date', lookup_expr='lte')

    class Meta:
        model = Book
        fields = ['author', 'published_after', 'published_before']
```

### Step 4: Update Book ViewSet

```python
# api/views.py
from rest_framework import viewsets, filters
from rest_framework.permissions import IsAuthenticatedOrReadOnly
from django_filters.rest_framework import DjangoFilterBackend
from .models import Book
from .serializers import BookSerializer
from .filters import BookFilter
from .permissions import IsOwnerOrReadOnly

class BookViewSet(viewsets.ModelViewSet):
    queryset = Book.objects.all()
    serializer_class = BookSerializer
    permission_classes = [IsAuthenticatedOrReadOnly, IsOwnerOrReadOnly]
    filter_backends = [DjangoFilterBackend, filters.SearchFilter, filters.OrderingFilter]
    filterset_class = BookFilter
    search_fields = ['title', 'author', 'description']
    ordering_fields = ['title', 'author', 'published_date', 'created_at']
    ordering = ['-created_at']

    def perform_create(self, serializer):
        serializer.save(owner=self.request.user)
```

### Step 5: Test Secure Book API

```bash
# Get token
TOKEN=$(curl -s -X POST http://localhost:8000/api/token/ \
  -H "Content-Type: application/json" \
  -d '{"username": "testuser", "password": "testpass123"}' \
  | python -c "import sys, json; print(json.load(sys.stdin)['access'])")

# Create book (authenticated)
curl -X POST http://localhost:8000/api/books/ \
  -H "Content-Type: application/json" \
  -H "Authorization: Bearer $TOKEN" \
  -d '{
    "title": "Django for Beginners",
    "author": "John Doe",
    "published_date": "2023-01-15",
    "description": "Learn Django step by step"
  }'

# List books (anyone can read)
curl http://localhost:8000/api/books/

# Filter by author
curl "http://localhost:8000/api/books/?author=John"

# Search
curl "http://localhost:8000/api/books/?search=django"

# Order by title
curl "http://localhost:8000/api/books/?ordering=title"
```

## Step-by-Step: Secure Task API

### Step 1: Update Task Model

```python
# api/models.py
from django.db import models
from django.contrib.auth.models import User

class Task(models.Model):
    title = models.CharField(max_length=100)
    desc = models.TextField(null=True, blank=True)
    completed = models.BooleanField(default=False)
    owner = models.ForeignKey(User, on_delete=models.CASCADE, related_name='tasks')
    created_at = models.DateTimeField(auto_now_add=True)
    updated_at = models.DateTimeField(auto_now=True)
    
    def __str__(self):
        return self.title

    class Meta:
        ordering = ['-created_at']
```

**Create migration:**

```bash
python manage.py makemigrations
python manage.py migrate
```

### Step 2: Update Task Serializer

```python
# api/serializers.py
class TaskSerializer(serializers.ModelSerializer):
    owner_username = serializers.CharField(source='owner.username', read_only=True)

    class Meta:
        model = Task
        fields = ['id', 'title', 'desc', 'completed', 'owner', 'owner_username', 
                  'created_at', 'updated_at']
        read_only_fields = ['id', 'owner', 'created_at', 'updated_at']
```

### Step 3: Create Task Filter

```python
# api/filters.py
import django_filters
from .models import Task

class TaskFilter(django_filters.FilterSet):
    completed = django_filters.BooleanFilter()
    created_after = django_filters.DateFilter(field_name='created_at', lookup_expr='gte')
    created_before = django_filters.DateFilter(field_name='created_at', lookup_expr='lte')

    class Meta:
        model = Task
        fields = ['completed', 'created_after', 'created_before']
```

### Step 4: Update Task ViewSet

```python
# api/views.py
from rest_framework import viewsets, filters
from rest_framework.permissions import IsAuthenticated
from django_filters.rest_framework import DjangoFilterBackend
from .models import Task
from .serializers import TaskSerializer
from .filters import TaskFilter

class TaskViewSet(viewsets.ModelViewSet):
    serializer_class = TaskSerializer
    permission_classes = [IsAuthenticated]
    filter_backends = [DjangoFilterBackend, filters.SearchFilter, filters.OrderingFilter]
    filterset_class = TaskFilter
    search_fields = ['title', 'desc']
    ordering_fields = ['title', 'completed', 'created_at']
    ordering = ['-created_at']

    def get_queryset(self):
        # Users can only see their own tasks
        return Task.objects.filter(owner=self.request.user)

    def perform_create(self, serializer):
        serializer.save(owner=self.request.user)
```

### Step 5: Test Secure Task API

```bash
# Create task (authenticated)
curl -X POST http://localhost:8000/api/tasks/ \
  -H "Content-Type: application/json" \
  -H "Authorization: Bearer $TOKEN" \
  -d '{
    "title": "Learn DRF Level 2",
    "desc": "Complete authentication and permissions",
    "completed": false
  }'

# List my tasks
curl -H "Authorization: Bearer $TOKEN" \
  http://localhost:8000/api/tasks/

# Filter by completed
curl -H "Authorization: Bearer $TOKEN" \
  "http://localhost:8000/api/tasks/?completed=false"

# Search tasks
curl -H "Authorization: Bearer $TOKEN" \
  "http://localhost:8000/api/tasks/?search=DRF"
```

## UserProfile API

### Step 1: Create UserProfile Model

```python
# api/models.py
from django.db import models
from django.contrib.auth.models import User

class UserProfile(models.Model):
    user = models.OneToOneField(User, on_delete=models.CASCADE, related_name='profile')
    bio = models.TextField(blank=True)
    phone_number = models.CharField(max_length=15, blank=True)
    avatar = models.ImageField(upload_to='avatars/', blank=True, null=True)
    website = models.URLField(blank=True)
    created_at = models.DateTimeField(auto_now_add=True)
    updated_at = models.DateTimeField(auto_now=True)

    def __str__(self):
        return f"{self.user.username}'s Profile"
```

### Step 2: Create UserProfile Serializer

```python
# api/serializers.py
class UserProfileSerializer(serializers.ModelSerializer):
    username = serializers.CharField(source='user.username', read_only=True)
    email = serializers.EmailField(source='user.email', read_only=True)

    class Meta:
        model = UserProfile
        fields = ['id', 'username', 'email', 'bio', 'phone_number', 
                  'avatar', 'website', 'created_at', 'updated_at']
        read_only_fields = ['id', 'created_at', 'updated_at']
```

### Step 3: Create UserProfile ViewSet

```python
# api/views.py
from rest_framework import viewsets
from rest_framework.permissions import IsAuthenticated
from .models import UserProfile
from .serializers import UserProfileSerializer

class UserProfileViewSet(viewsets.ModelViewSet):
    serializer_class = UserProfileSerializer
    permission_classes = [IsAuthenticated]

    def get_queryset(self):
        return UserProfile.objects.filter(user=self.request.user)

    def perform_create(self, serializer):
        serializer.save(user=self.request.user)
```

## Throttling

### Configure Throttling

```python
# core/settings.py
REST_FRAMEWORK = {
    'DEFAULT_THROTTLE_CLASSES': [
        'rest_framework.throttling.AnonRateThrottle',
        'rest_framework.throttling.UserRateThrottle'
    ],
    'DEFAULT_THROTTLE_RATES': {
        'anon': '100/hour',
        'user': '1000/hour'
    }
}
```

### Custom Throttling

```python
# api/throttles.py
from rest_framework.throttling import UserRateThrottle

class BookCreateThrottle(UserRateThrottle):
    rate = '5/day'
```

```python
# api/views.py
from .throttles import BookCreateThrottle

class BookViewSet(viewsets.ModelViewSet):
    throttle_classes = [BookCreateThrottle]
    # ...
```

## Testing Authenticated APIs

See [CURL_GUIDE.md](CURL_GUIDE.md) for detailed examples. Quick reference:

```bash
# Get token
TOKEN=$(curl -s -X POST http://localhost:8000/api/token/ \
  -H "Content-Type: application/json" \
  -d '{"username": "user", "password": "pass"}' \
  | jq -r '.access')

# Use token
curl -H "Authorization: Bearer $TOKEN" \
  http://localhost:8000/api/tasks/

# Refresh token
curl -X POST http://localhost:8000/api/token/refresh/ \
  -H "Content-Type: application/json" \
  -d '{"refresh": "your_refresh_token"}'
```

## Exercises

### Exercise 1: Complete Secure Book API

1. Add owner field to Book model
2. Implement IsOwnerOrReadOnly permission
3. Add filtering by author and date range
4. Add search functionality
5. Add pagination
6. Test all endpoints with cURL

### Exercise 2: UserProfile API

1. Create UserProfile model
2. Create UserProfileSerializer and ViewSet
3. Allow users to view/edit only their own profile
4. Test with cURL

### Exercise 3: Task Filtering

1. Add priority field to Task model
2. Create TaskFilter with priority filtering
3. Add ordering by priority
4. Test filtering and ordering

## Add-ons

### Add-on 1: Custom JWT Claims

```python
# api/serializers.py
from rest_framework_simplejwt.serializers import TokenObtainPairSerializer

class CustomTokenObtainPairSerializer(TokenObtainPairSerializer):
    @classmethod
    def get_token(cls, user):
        token = super().get_token(user)
        token['username'] = user.username
        token['email'] = user.email
        return token
```

```python
# api/views.py
from rest_framework_simplejwt.views import TokenObtainPairView
from .serializers import CustomTokenObtainPairSerializer

class CustomTokenObtainPairView(TokenObtainPairView):
    serializer_class = CustomTokenObtainPairSerializer
```

### Add-on 2: Rate Limiting

Implement different rate limits for different actions.

### Add-on 3: Permission Mixins

Create reusable permission mixins for common patterns.

## Next Steps

Congratulations! You've completed Level 2. You now know:

- ✅ Django authentication system
- ✅ JWT authentication setup
- ✅ Permissions (built-in and custom)
- ✅ Filtering, searching, and ordering
- ✅ Pagination strategies
- ✅ Building secure APIs

**Ready for Level 3?** Continue to [Level 3: Advanced](LEVEL_3_ADVANCED.md) to learn about relationships, nested serializers, and query optimization!

---

**Resources:**
- [DRF Authentication](https://www.django-rest-framework.org/api-guide/authentication/)
- [DRF Permissions](https://www.django-rest-framework.org/api-guide/permissions/)
- [djangorestframework-simplejwt](https://github.com/jazzband/djangorestframework-simplejwt)
- [django-filter](https://django-filter.readthedocs.io/)

