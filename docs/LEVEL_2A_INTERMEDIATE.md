# Level 2A: Intermediate - Authentication, Permissions & Throttling

## Goal

Build secure APIs with robust authentication, permissions, and rate limiting. By the end of this level, you'll be able to create secure REST APIs with user authentication, access control, and protection against API abuse.

## Note on Examples

This guide uses **Task API** as the primary example throughout explanation sections for consistency with Level 1. Complete step-by-step implementations are provided for both **Task API** (user-owned, private data) and **Book API** (can be public or user-owned) to show different use cases:

- **Task API examples**: Used in explanations (authentication, permissions, throttling) - represents private, user-owned data
- **Book API examples**: Used in complete step-by-step section - represents content that can be public or user-owned
- **Both examples**: Help you understand different scenarios and patterns

**Note**: Filtering, searching, ordering, and pagination are covered in **Level 2B**. This level (2A) focuses on authentication, permissions, and throttling.

You can apply all concepts to any model (Task, Book, Product, Post, etc.).

## Class-Based Views (CBV) Approach

**This guide follows the Class-Based Views (CBV) approach** throughout, which is the recommended and modern way to build DRF APIs:

- ✅ **ViewSets** (`ModelViewSet`, `ReadOnlyViewSet`) - For CRUD operations
- ✅ **Generic Views** (`CreateAPIView`, `ListAPIView`, etc.) - For specific operations
- ✅ **APIView** - For custom logic when needed
- ❌ **Function-based views** (`@api_view`) - Only used when absolutely necessary

**Why Class-Based Views?**
- **Reusable**: Easy to extend and customize
- **DRY**: Less code duplication
- **Built-in features**: Authentication, permissions, filtering work automatically
- **Industry standard**: Most DRF projects use CBV
- **Better organization**: Clear separation of concerns

All examples in this guide use class-based views unless explicitly noted otherwise.

## Table of Contents

1. [Class-Based Views (CBV) Approach](#class-based-views-cbv-approach)
2. [Django Authentication System](#django-authentication-system)
3. [Django Abstract Models](#django-abstract-models)
4. [DRF Authentication Classes](#drf-authentication-classes)
5. [JWT Authentication Setup](#jwt-authentication-setup)
6. [Permissions](#permissions)
7. [Step-by-Step: Secure Book API](#step-by-step-secure-book-api)
8. [Step-by-Step: Secure Task API](#step-by-step-secure-task-api)
9. [UserProfile API](#userprofile-api)
10. [Throttling](#throttling)
    - [What is Throttling?](#what-is-throttling)
    - [How Throttling is Determined](#how-throttling-is-determined)
    - [Setting the Throttling Policy](#setting-the-throttling-policy)
    - [How Clients are Identified](#how-clients-are-identified)
    - [Setting up the Cache](#setting-up-the-cache)
    - [A Note on Concurrency](#a-note-on-concurrency)
    - [API Reference](#api-reference)
    - [Custom Throttles](#custom-throttles)
11. [Exercises](#exercises)
12. [Common Errors and Solutions](#common-errors-and-solutions)
13. [Add-ons](#add-ons)

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
user.username      # Username
user.email         # Email address
user.password      # Hashed password (never access directly)
user.first_name    # First name
user.last_name     # Last name
user.is_staff      # Admin access (can access Django admin)
user.is_active     # Account status (False = disabled account)
user.is_superuser  # Superuser (all permissions)
user.date_joined   # When user account was created
```

**Explanation of key fields**:
- **`username`**: Unique identifier for login
- **`email`**: User's email address
- **`password`**: Stored as hash (never plain text)
- **`is_staff`**: Can access Django admin panel
- **`is_active`**: If False, user cannot login
- **`is_superuser`**: Has all permissions automatically

### Creating Users

**Why multiple methods?**
- Different methods serve different purposes
- Some hash passwords (secure), others don't (insecure)
- Always use methods that hash passwords!

```python
# In Django shell: python manage.py shell
# Open Django shell: python manage.py shell

# Import Django's built-in User model
from django.contrib.auth.models import User

# Method 1: create_user (recommended - hashes password automatically)
# This is the SECURE way to create users
# create_user() automatically:
#   - Hashes the password (converts to secure hash)
#   - Validates username/email format
#   - Sets is_active=True by default
user = User.objects.create_user(
    username='john',  # Unique username for login
    email='john@example.com',  # User's email address
    password='securepassword123'  # Password will be hashed automatically
)
# After this, user.password contains a hash like "pbkdf2_sha256$..." (not plain text!)
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
# Creates a user with admin privileges
# Use this for site administrators who need full access
admin = User.objects.create_superuser(
    username='admin',  # Admin username
    email='admin@example.com',  # Admin email
    password='admin123'  # Admin password (will be hashed)
)
# After this, admin has:
#   - is_staff=True (can access Django admin panel)
#   - is_superuser=True (has all permissions automatically)
#   - Password is hashed (secure)
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

## Django Abstract Models

### What are Abstract Models?

**Abstract Models** are base model classes that are not created as database tables themselves. Instead, they serve as templates that other models can inherit from to share common fields and methods.

**Why use Abstract Models?**
- **DRY Principle**: Don't Repeat Yourself - define common fields once
- **Consistency**: All models have the same timestamp fields, soft delete, etc.
- **Maintainability**: Update common fields in one place
- **Reusability**: Share common functionality across multiple models

**Real-world analogy**:
- **Abstract Model** = Blueprint template (not a real house)
- **Regular Model** = Actual house built from blueprint
- **Inheritance** = Multiple houses can use the same blueprint

### Creating Abstract Base Models

**Key Point**: Set `abstract = True` in the `Meta` class. This tells Django: "Don't create a database table for this model, it's just a template."

```python
# api/models.py
from django.db import models

# Abstract model - won't create a database table
# This is just a template for other models to inherit from
class TimestampedModel(models.Model):
    """
    Abstract base model that provides created_at and updated_at fields.
    Any model that inherits from this will automatically get these fields.
    """
    # Auto-set when object is first created (never changes)
    created_at = models.DateTimeField(auto_now_add=True)
    
    # Auto-updated every time object is saved
    updated_at = models.DateTimeField(auto_now=True)
    
    class Meta:
        # This makes it abstract - no database table will be created
        abstract = True
        # This means: "This is a template, not a real model"
```

**What `abstract = True` does**:
- Django will NOT create a database table for this model
- Other models can inherit from it
- All fields and methods are inherited by child models

### Inheriting from Abstract Models

**How to use**: Simply inherit from the abstract model instead of `models.Model`:

```python
# api/models.py
from django.db import models
from django.contrib.auth.models import User

# Inherit from TimestampedModel instead of models.Model
# This automatically gives us created_at and updated_at fields
class Task(TimestampedModel):
    """
    Task model that automatically has created_at and updated_at fields
    thanks to inheriting from TimestampedModel.
    """
    # Regular fields specific to Task
    title = models.CharField(max_length=100)
    desc = models.TextField(null=True, blank=True)
    completed = models.BooleanField(default=False)
    owner = models.ForeignKey(User, on_delete=models.CASCADE, related_name='tasks')
    
    # No need to define created_at and updated_at here!
    # They're inherited from TimestampedModel automatically
    
    def __str__(self):
        return self.title

    class Meta:
        ordering = ['-created_at']
        # Note: We don't need abstract = True here
        # This is a real model that will create a database table
```

**What happens**:
- `Task` model gets all fields from `TimestampedModel` (created_at, updated_at)
- Database table `api_task` will have: id, title, desc, completed, owner_id, created_at, updated_at
- No separate table for `TimestampedModel` (because it's abstract)

### Common Abstract Model Patterns

#### Pattern 1: TimestampedModel (Most Common)

```python
# api/models.py
from django.db import models

class TimestampedModel(models.Model):
    """
    Provides created_at and updated_at timestamps to all models.
    Use this when you want to track when objects are created/modified.
    """
    created_at = models.DateTimeField(auto_now_add=True)
    updated_at = models.DateTimeField(auto_now=True)
    
    class Meta:
        abstract = True

# Usage in any model:
class Book(TimestampedModel):
    title = models.CharField(max_length=200)
    # Automatically has created_at and updated_at

class Post(TimestampedModel):
    content = models.TextField()
    # Automatically has created_at and updated_at
```

#### Pattern 2: Soft Delete Model

```python
# api/models.py
from django.db import models
from django.utils import timezone

class SoftDeleteModel(models.Model):
    """
    Provides soft delete functionality.
    Instead of actually deleting, sets deleted_at timestamp.
    Objects can be "restored" by setting deleted_at to None.
    """
    # When object was deleted (None = not deleted)
    deleted_at = models.DateTimeField(null=True, blank=True)
    
    class Meta:
        abstract = True
    
    def delete(self, using=None, keep_parents=False):
        """
        Override delete() to do soft delete instead of hard delete.
        Instead of removing from database, just mark as deleted.
        """
        self.deleted_at = timezone.now()
        self.save()
    
    def restore(self):
        """Restore a soft-deleted object"""
        self.deleted_at = None
        self.save()
    
    @property
    def is_deleted(self):
        """Check if object is soft-deleted"""
        return self.deleted_at is not None

# Usage:
class Task(SoftDeleteModel, TimestampedModel):
    """
    Task model with both soft delete and timestamps.
    Can inherit from multiple abstract models!
    """
    title = models.CharField(max_length=100)
    # Has: created_at, updated_at, deleted_at, and all methods above
```

#### Pattern 3: BaseModel (Combining Multiple Features)

```python
# api/models.py
from django.db import models
from django.utils import timezone

class BaseModel(models.Model):
    """
    Comprehensive base model with timestamps and soft delete.
    Use this as the foundation for most of your models.
    """
    # Timestamp fields
    created_at = models.DateTimeField(auto_now_add=True)
    updated_at = models.DateTimeField(auto_now=True)
    
    # Soft delete field
    deleted_at = models.DateTimeField(null=True, blank=True)
    
    class Meta:
        abstract = True
        # Default ordering: newest first
        ordering = ['-created_at']
    
    def delete(self, using=None, keep_parents=False):
        """Soft delete instead of hard delete"""
        self.deleted_at = timezone.now()
        self.save()
    
    def restore(self):
        """Restore soft-deleted object"""
        self.deleted_at = None
        self.save()
    
    @property
    def is_deleted(self):
        """Check if object is soft-deleted"""
        return self.deleted_at is not None

# Usage:
class Task(BaseModel):
    """
    Task model inheriting from BaseModel.
    Gets timestamps, soft delete, and default ordering automatically.
    """
    title = models.CharField(max_length=100)
    desc = models.TextField(blank=True)
    # Automatically has: created_at, updated_at, deleted_at
    # Plus: delete(), restore(), is_deleted property
```

### Multiple Inheritance from Abstract Models

**You can inherit from multiple abstract models**:

```python
# api/models.py
from django.db import models

class TimestampedModel(models.Model):
    created_at = models.DateTimeField(auto_now_add=True)
    updated_at = models.DateTimeField(auto_now=True)
    
    class Meta:
        abstract = True

class SlugModel(models.Model):
    """
    Abstract model that provides slug field and auto-generation.
    """
    slug = models.SlugField(max_length=200, unique=True, blank=True)
    
    class Meta:
        abstract = True
    
    def save(self, *args, **kwargs):
        """Auto-generate slug from title if not provided"""
        from django.utils.text import slugify
        if not self.slug:
            # Generate slug from title (if model has title field)
            if hasattr(self, 'title'):
                self.slug = slugify(self.title)
        super().save(*args, **kwargs)

# Inherit from multiple abstract models
class Post(TimestampedModel, SlugModel):
    """
    Post model with timestamps AND slug functionality.
    Inherits from both abstract models.
    """
    title = models.CharField(max_length=200)
    content = models.TextField()
    # Has: created_at, updated_at, slug (all inherited)
```

### Best Practices

**1. Use Abstract Models for Common Fields**
```python
# ✅ GOOD: Use abstract model for common fields
class TimestampedModel(models.Model):
    created_at = models.DateTimeField(auto_now_add=True)
    updated_at = models.DateTimeField(auto_now=True)
    class Meta:
        abstract = True

class Task(TimestampedModel):
    title = models.CharField(max_length=100)
    # created_at and updated_at inherited

# ❌ BAD: Repeating fields in every model
class Task(models.Model):
    title = models.CharField(max_length=100)
    created_at = models.DateTimeField(auto_now_add=True)  # Repeated
    updated_at = models.DateTimeField(auto_now=True)  # Repeated
```

**2. Keep Abstract Models Simple**
```python
# ✅ GOOD: Simple, focused abstract model
class TimestampedModel(models.Model):
    created_at = models.DateTimeField(auto_now_add=True)
    updated_at = models.DateTimeField(auto_now=True)
    class Meta:
        abstract = True

# ❌ BAD: Too complex, mixing concerns
class EverythingModel(models.Model):
    created_at = models.DateTimeField(auto_now_add=True)
    updated_at = models.DateTimeField(auto_now=True)
    deleted_at = models.DateTimeField(null=True)
    slug = models.SlugField()
    # ... too many things
    class Meta:
        abstract = True
```

**3. Document Your Abstract Models**
```python
# ✅ GOOD: Well-documented abstract model
class TimestampedModel(models.Model):
    """
    Abstract base model providing created_at and updated_at timestamps.
    
    Usage:
        class MyModel(TimestampedModel):
            name = models.CharField(max_length=100)
            # Automatically gets created_at and updated_at
    """
    created_at = models.DateTimeField(auto_now_add=True)
    updated_at = models.DateTimeField(auto_now=True)
    
    class Meta:
        abstract = True
```

### When to Use Abstract Models

**Use Abstract Models when**:
- ✅ Multiple models need the same fields (timestamps, soft delete, etc.)
- ✅ You want to ensure consistency across models
- ✅ You want to reduce code duplication
- ✅ You want to share common methods/properties

**Don't use Abstract Models when**:
- ❌ Only one model needs the fields (just add them directly)
- ❌ The fields are model-specific (not shared)
- ❌ You need a database table for the base model (use regular inheritance)

### Step-by-Step: Adding Abstract Models to Existing Project

**Step 1: Create Abstract Model**

```python
# api/models.py
from django.db import models

# Create abstract model
class TimestampedModel(models.Model):
    """Base model with timestamp fields"""
    created_at = models.DateTimeField(auto_now_add=True)
    updated_at = models.DateTimeField(auto_now=True)
    
    class Meta:
        abstract = True  # Important: makes it abstract
```

**Step 2: Update Existing Models**

```python
# api/models.py
# Before:
class Task(models.Model):
    title = models.CharField(max_length=100)
    created_at = models.DateTimeField(auto_now_add=True)  # Remove this
    updated_at = models.DateTimeField(auto_now=True)  # Remove this

# After:
class Task(TimestampedModel):  # Inherit from TimestampedModel
    title = models.CharField(max_length=100)
    # created_at and updated_at are inherited automatically
```

**Step 3: Create and Run Migrations**

```bash
# Create migration for the changes
python manage.py makemigrations

# Review the migration (it should remove created_at/updated_at from Task
# and add them back via inheritance - but database structure stays same)

# Apply migration
python manage.py migrate
```

**Note**: If your models already have `created_at` and `updated_at` fields, Django will handle this in migrations. The database structure won't change, but your code becomes cleaner.

### Common Abstract Model Examples

**Example 1: UUID Primary Key Model**

```python
# api/models.py
import uuid
from django.db import models

class UUIDModel(models.Model):
    """
    Abstract model that uses UUID instead of auto-incrementing integer ID.
    Useful for security (can't guess IDs) and distributed systems.
    """
    # Use UUID as primary key instead of auto-incrementing integer
    id = models.UUIDField(primary_key=True, default=uuid.uuid4, editable=False)
    
    class Meta:
        abstract = True

# Usage:
class Task(UUIDModel, TimestampedModel):
    """
    Task with UUID primary key and timestamps.
    ID will be something like: 'a1b2c3d4-e5f6-7890-abcd-ef1234567890'
    """
    title = models.CharField(max_length=100)
    # Has: UUID id, created_at, updated_at
```

**Example 2: Owner Model (User Ownership)**

```python
# api/models.py
from django.db import models
from django.contrib.auth.models import User

class OwnedModel(models.Model):
    """
    Abstract model that adds owner field to track who created the object.
    Useful for user-owned resources (tasks, posts, etc.).
    """
    # ForeignKey to User - tracks who owns this object
    owner = models.ForeignKey(User, on_delete=models.CASCADE, related_name='%(class)s_set')
    
    class Meta:
        abstract = True

# Usage:
class Task(OwnedModel, TimestampedModel):
    """
    Task model with owner and timestamps.
    Every task automatically has an owner field.
    """
    title = models.CharField(max_length=100)
    # Has: owner, created_at, updated_at
```

### Summary

**Key Takeaways**:
- Abstract models are templates, not real database tables
- Set `abstract = True` in `Meta` class
- Inherit from abstract models to get their fields/methods
- Use for common fields like timestamps, soft delete, etc.
- Follows DRY principle - define once, use everywhere
- Can inherit from multiple abstract models

**Next**: Now that you understand abstract models, you can create reusable base models for your project!

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
# Import timedelta to specify token expiration times
from datetime import timedelta

# DRF configuration - applies to all views unless overridden
REST_FRAMEWORK = {
    # Authentication: How users prove their identity
    # JWT Authentication uses tokens instead of sessions
    'DEFAULT_AUTHENTICATION_CLASSES': [
        'rest_framework_simplejwt.authentication.JWTAuthentication',
    ],
    # Permissions: What authenticated users can do
    # IsAuthenticated = Must be logged in to access (default for all views)
    'DEFAULT_PERMISSION_CLASSES': [
        'rest_framework.permissions.IsAuthenticated',
    ],
}

# JWT-specific settings - controls token behavior
SIMPLE_JWT = {
    # How long access token is valid (short = more secure, but users need to refresh more often)
    'ACCESS_TOKEN_LIFETIME': timedelta(minutes=60),
    
    # How long refresh token is valid (longer = users stay logged in longer)
    'REFRESH_TOKEN_LIFETIME': timedelta(days=1),
    
    # Security: Issue new refresh token when old one is used (prevents token reuse)
    'ROTATE_REFRESH_TOKENS': True,
    
    # Security: Add old refresh token to blacklist after rotation (prevents reuse)
    'BLACKLIST_AFTER_ROTATION': True,
    
    # Track when user last logged in (useful for security monitoring)
    'UPDATE_LAST_LOGIN': True,
    
    # Algorithm to sign tokens (HS256 = HMAC SHA-256, fast and secure)
    'ALGORITHM': 'HS256',
    
    # Secret key to sign tokens (use Django's SECRET_KEY for security)
    'SIGNING_KEY': SECRET_KEY,
    
    # Token type in Authorization header (Bearer is standard OAuth2 format)
    'AUTH_HEADER_TYPES': ('Bearer',),
    
    # Header name where token is sent (HTTP_AUTHORIZATION = "Authorization: Bearer <token>")
    'AUTH_HEADER_NAME': 'HTTP_AUTHORIZATION',
    
    # Which User model field to use as user ID (usually 'id')
    'USER_ID_FIELD': 'id',
    
    # Where to store user ID in token payload (used when decoding token)
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
# Import Django admin and URL routing
from django.contrib import admin
from django.urls import path, include
# Import JWT view classes for token management
from rest_framework_simplejwt.views import (
    TokenObtainPairView,  # POST /api/token/ - Get tokens (login)
    TokenRefreshView,     # POST /api/token/refresh/ - Get new access token
    TokenVerifyView,      # POST /api/token/verify/ - Check if token is valid
)

# URL patterns - maps URLs to views
urlpatterns = [
    # Django admin interface
    path('admin/', admin.site.urls),
    # Include API URLs from api app
    path('api/', include('api.urls')),
    
    # JWT authentication endpoints
    # POST /api/token/ - Login: Send username/password, get access+refresh tokens
    path('api/token/', TokenObtainPairView.as_view(), name='token_obtain_pair'),
    
    # POST /api/token/refresh/ - Refresh: Send refresh token, get new access token
    path('api/token/refresh/', TokenRefreshView.as_view(), name='token_refresh'),
    
    # POST /api/token/verify/ - Verify: Send token, check if it's valid
    path('api/token/verify/', TokenVerifyView.as_view(), name='token_verify'),
]
```

### Step 5: Create User Registration View

```python
# api/serializers.py
# Import DRF serializers for data validation and conversion
from rest_framework import serializers
# Import Django's built-in User model
from django.contrib.auth.models import User

# Serializer for user registration
# ModelSerializer automatically creates fields based on User model
class UserRegistrationSerializer(serializers.ModelSerializer):
    # Password field: write_only means it's only used when creating (not returned in response)
    # min_length=8 enforces minimum password length
    password = serializers.CharField(write_only=True, min_length=8)
    
    # Password confirmation field: also write_only, used to verify passwords match
    password_confirm = serializers.CharField(write_only=True)

    class Meta:
        # Base serializer on User model
        model = User
        # Fields to include in registration form
        fields = ['username', 'email', 'password', 'password_confirm', 'first_name', 'last_name']

    # Object-level validation: Check if passwords match
    # This method is called after field-level validation
    def validate(self, data):
        # Compare password and password_confirm
        if data['password'] != data['password_confirm']:
            # Raise validation error if they don't match
            raise serializers.ValidationError("Passwords don't match")
        return data

    # Override create() to handle password hashing
    # This is called when serializer.save() is called
    def create(self, validated_data):
        # Remove password_confirm (not a User model field, just for validation)
        validated_data.pop('password_confirm')
        # Use create_user() instead of create() - this hashes the password automatically
        # create_user() is secure, create() would store plain text password (BAD!)
        user = User.objects.create_user(**validated_data)
        return user
```

```python
# api/views.py
# Import generic views - CreateAPIView provides POST endpoint automatically
from rest_framework import generics, status
# Import Response to return custom JSON responses
from rest_framework.response import Response
# Import JWT token classes to generate tokens for new users
from rest_framework_simplejwt.tokens import RefreshToken
# Import User model
from django.contrib.auth.models import User
# Import our registration serializer
from .serializers import UserRegistrationSerializer

# CreateAPIView automatically creates a POST endpoint
# Users can POST to this endpoint to register
class UserRegistrationView(generics.CreateAPIView):
    # Base queryset (not really used for creation, but required)
    queryset = User.objects.all()
    
    # Serializer to validate and create user
    serializer_class = UserRegistrationSerializer
    
    # Empty permission_classes = Allow anyone to register (public endpoint)
    # No authentication required to create an account
    permission_classes = []

    # Override create() to return tokens after registration
    # This method is called when POST request is made
    def create(self, request, *args, **kwargs):
        # Get serializer instance with request data
        serializer = self.get_serializer(data=request.data)
        
        # Validate data - raise_exception=True returns 400 error if invalid
        serializer.is_valid(raise_exception=True)
        
        # Save user to database (calls serializer.create() which hashes password)
        user = serializer.save()
        
        # Generate JWT tokens for the newly created user
        # RefreshToken contains both refresh and access tokens
        refresh = RefreshToken.for_user(user)
        
        # Return user data and tokens in response
        # User gets tokens immediately after registration (auto-login)
        return Response({
            'user': serializer.data,  # User information (username, email, etc.)
            'refresh': str(refresh),  # Refresh token (long-lived, used to get new access tokens)
            'access': str(refresh.access_token),  # Access token (short-lived, used for API requests)
        }, status=status.HTTP_201_CREATED)  # 201 = Created (successful resource creation)
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
# Note: By default, JWT uses 'username' for authentication. 
# The example below shows username (default). If you've configured JWT to use email, 
# replace 'username' with 'email' in the request body.

curl -X POST http://localhost:8000/api/token/ \
  -H "Content-Type: application/json" \
  -d '{"username": "testuser", "password": "testpass123"}'

# Alternative: If using email-based authentication (requires custom JWT serializer)
# curl -X POST http://localhost:8000/api/token/ \
#   -H "Content-Type: application/json" \
#   -d '{"email": "test@example.com", "password": "testpass123"}'

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

class TaskViewSet(viewsets.ModelViewSet):
    permission_classes = [AllowAny]
    # ...
```

**Note**: In this example, we use `TaskViewSet`, but the same pattern applies to any ViewSet (Book, Product, etc.).

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

class TaskViewSet(viewsets.ModelViewSet):
    permission_classes = [IsAuthenticated]
    # ...
```

**Why this configuration?**
- Applies to all actions (list, create, retrieve, update, delete)
- Can override per-action if needed
- Most common permission for user data

**Note**: This is the most common permission for user-owned resources like tasks, where users should only see their own data.

#### 3. IsAdminUser

User must be staff/admin.

```python
from rest_framework.permissions import IsAdminUser

class TaskViewSet(viewsets.ModelViewSet):
    permission_classes = [IsAdminUser]
    # ...
```

**Note**: Use this for admin-only endpoints. For user data like tasks, you typically want `IsAuthenticated` instead.

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

class TaskViewSet(viewsets.ModelViewSet):
    permission_classes = [IsAuthenticatedOrReadOnly]
    # ...
```

**What this means**:
- `GET /api/tasks/` → Anyone can access ✅
- `POST /api/tasks/` → Must be authenticated ✅
- `PUT /api/tasks/1/` → Must be authenticated ✅
- `DELETE /api/tasks/1/` → Must be authenticated ✅

**Note**: For private user data like personal tasks, you'd typically use `IsAuthenticated` instead. This permission is better for public content.

### Custom Permissions

#### IsOwnerOrReadOnly

Only the owner can edit/delete, others can read.

```python
# api/permissions.py
# Import DRF permissions module
from rest_framework import permissions

# Custom permission class - inherits from BasePermission
# BasePermission is the base class for all permission classes
class IsOwnerOrReadOnly(permissions.BasePermission):
    """
    Custom permission to only allow owners to edit/delete objects.
    Anyone can read (GET), but only the owner can modify (POST, PUT, PATCH, DELETE).
    """

    # This method is called for object-level permissions (detail views)
    # For list views, use has_permission() instead
    def has_object_permission(self, request, view, obj):
        # SAFE_METHODS = GET, HEAD, OPTIONS (read-only operations)
        # If it's a read operation, allow it for everyone
        if request.method in permissions.SAFE_METHODS:
            return True  # Allow read access to anyone

        # For write operations (POST, PUT, PATCH, DELETE), check ownership
        # obj.owner = the user who owns this object (from model's owner field)
        # request.user = the currently logged-in user (from authentication)
        # Only allow if they match
        return obj.owner == request.user
```

**Use in views:**

```python
# api/views.py
from rest_framework.permissions import IsAuthenticated
from .permissions import IsOwnerOrReadOnly

class TaskViewSet(viewsets.ModelViewSet):
    permission_classes = [IsAuthenticated, IsOwnerOrReadOnly]
    
    def perform_create(self, serializer):
        serializer.save(owner=self.request.user)
```

**Why this works for tasks:**
- Users can read all tasks (or filter to their own in `get_queryset()`)
- Users can only edit/delete their own tasks
- Perfect for user-owned resources like tasks, notes, etc.

### Permission at View Level

**Why different permissions per action?**
- **List/Retrieve**: Maybe public (AllowAny) or authenticated (IsAuthenticated)
- **Create**: Usually requires authentication
- **Update/Delete**: Usually requires ownership (IsOwnerOrReadOnly)

```python
# Different permissions for different actions
# Import permission classes
from rest_framework.permissions import IsAuthenticated
from .permissions import IsOwnerOrReadOnly

class TaskViewSet(viewsets.ModelViewSet):
    # Override get_permissions() to set different permissions per action
    # This method is called by DRF to determine which permissions to check
    def get_permissions(self):
        # self.action = current action: 'list', 'create', 'retrieve', 'update', 'destroy', etc.
        
        # For create action (POST /api/tasks/)
        if self.action == 'create':
            # Anyone authenticated can create tasks
            permission_classes = [IsAuthenticated]
        
        # For update/delete actions (PUT, PATCH, DELETE /api/tasks/1/)
        elif self.action in ['update', 'partial_update', 'destroy']:
            # Must be authenticated AND must be the owner
            # IsAuthenticated checks if user is logged in
            # IsOwnerOrReadOnly checks if user owns the task
            permission_classes = [IsAuthenticated, IsOwnerOrReadOnly]
        
        # For list/retrieve actions (GET /api/tasks/ or GET /api/tasks/1/)
        else:
            # Must be logged in to view tasks
            permission_classes = [IsAuthenticated]
        
        # Return list of permission instances
        # DRF will check all permissions - all must pass for access to be granted
        return [permission() for permission in permission_classes]
```

**Explanation**:
- **`create`**: Anyone authenticated can create tasks
- **`update/delete`**: Only task owner can modify
- **`list/retrieve`**: Authenticated users can view (you might filter to own tasks in `get_queryset()`)

**Alternative for private tasks** (users only see their own):
```python
class TaskViewSet(viewsets.ModelViewSet):
    permission_classes = [IsAuthenticated]  # Same for all actions
    
    def get_queryset(self):
        # Users only see their own tasks
        return Task.objects.filter(owner=self.request.user)
```

## Step-by-Step: Secure Book API

**Note**: This section uses Book API as an example of content that can be public or user-owned. The same patterns apply to Task API or any other model.

Let's upgrade the Book API with authentication and permissions. (Filtering, searching, and pagination will be covered in Level 2B.)

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

### Step 3: Update Book ViewSet

```python
# api/views.py
# Import DRF viewsets for CRUD operations
from rest_framework import viewsets
# Import permission classes to control access
from rest_framework.permissions import IsAuthenticatedOrReadOnly
# Import models and serializers
from .models import Book
from .serializers import BookSerializer
# Import custom permission class (we'll create this)
from .permissions import IsOwnerOrReadOnly

# ViewSet provides all CRUD operations automatically
# ModelViewSet = List, Create, Retrieve, Update, Delete
class BookViewSet(viewsets.ModelViewSet):
    # Base queryset - all books in database
    # We'll filter this in get_queryset() if needed
    queryset = Book.objects.all()
    
    # Serializer converts Book objects to/from JSON
    serializer_class = BookSerializer
    
    # Permission classes control who can access this endpoint
    # IsAuthenticatedOrReadOnly = Anyone can read, only authenticated users can write
    # IsOwnerOrReadOnly = Only owner can edit/delete (custom permission)
    permission_classes = [IsAuthenticatedOrReadOnly, IsOwnerOrReadOnly]
    
    # Note: Filtering, searching, and ordering will be added in Level 2B
    
    # Override perform_create to automatically set the owner
    # This is called when a new book is created via POST
    def perform_create(self, serializer):
        # Set owner to the currently logged-in user
        # self.request.user is automatically set by authentication
        serializer.save(owner=self.request.user)
```

### Step 5: Test Secure Book API

```bash
# Get token
# Note: By default, JWT uses 'username' for authentication.
# If you've configured JWT to use email, replace 'username' with 'email' below.

TOKEN=$(curl -s -X POST http://localhost:8000/api/token/ \
  -H "Content-Type: application/json" \
  -d '{"username": "testuser", "password": "testpass123"}' \
  | python -c "import sys, json; print(json.load(sys.stdin)['access'])")

# Alternative: If using email-based authentication (requires custom JWT serializer)
# TOKEN=$(curl -s -X POST http://localhost:8000/api/token/ \
#   -H "Content-Type: application/json" \
#   -d '{"email": "test@example.com", "password": "testpass123"}' \
#   | python -c "import sys, json; print(json.load(sys.stdin)['access'])")

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

# List books (anyone can read - public endpoint)
curl http://localhost:8000/api/books/

# Get specific book (anyone can read)
curl http://localhost:8000/api/books/1/

# Update book (must be authenticated and owner)
curl -X PATCH http://localhost:8000/api/books/1/ \
  -H "Content-Type: application/json" \
  -H "Authorization: Bearer $TOKEN" \
  -d '{"title": "Updated Title"}'

# Delete book (must be authenticated and owner)
curl -X DELETE http://localhost:8000/api/books/1/ \
  -H "Authorization: Bearer $TOKEN"

# Note: Filtering, searching, and ordering will be covered in Level 2B
```

## Step-by-Step: Secure Task API

**Note**: This section focuses on Task API as the primary example (consistent with Level 1). Tasks are typically private, user-owned data, making them perfect for demonstrating authentication and permissions.

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
from rest_framework import serializers
from .models import Task

class TaskSerializer(serializers.ModelSerializer):
    owner_username = serializers.CharField(source='owner.username', read_only=True)

    class Meta:
        model = Task
        fields = ['id', 'title', 'desc', 'completed', 'owner', 'owner_username', 
                  'created_at', 'updated_at']
        read_only_fields = ['id', 'owner', 'created_at', 'updated_at']
```

### Step 3: Update Task ViewSet

```python
# api/views.py
# Import DRF viewsets for CRUD operations
from rest_framework import viewsets
# Import permission class - users must be authenticated
from rest_framework.permissions import IsAuthenticated
# Import models and serializers
from .models import Task
from .serializers import TaskSerializer

# ViewSet provides all CRUD operations automatically
# ModelViewSet = List, Create, Retrieve, Update, Delete
class TaskViewSet(viewsets.ModelViewSet):
    # Serializer converts Task objects to/from JSON
    serializer_class = TaskSerializer
    
    # Permission class: Only authenticated users can access
    # Unauthenticated requests will get 401 Unauthorized
    permission_classes = [IsAuthenticated]
    
    # Note: Filtering, searching, and ordering will be added in Level 2B
    
    # Override get_queryset to filter tasks by owner
    # This ensures users only see their own tasks
    # Called for list and retrieve operations
    def get_queryset(self):
        # Filter tasks where owner matches the current logged-in user
        # self.request.user is automatically set by authentication
        return Task.objects.filter(owner=self.request.user)
    
    # Override perform_create to automatically set the owner
    # This is called when a new task is created via POST
    def perform_create(self, serializer):
        # Set owner to the currently logged-in user
        # self.request.user is automatically set by authentication
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

# List my tasks (only shows tasks owned by authenticated user)
curl -H "Authorization: Bearer $TOKEN" \
  http://localhost:8000/api/tasks/

# Get specific task (must be owner)
curl -H "Authorization: Bearer $TOKEN" \
  http://localhost:8000/api/tasks/1/

# Update task (must be authenticated and owner)
curl -X PATCH http://localhost:8000/api/tasks/1/ \
  -H "Content-Type: application/json" \
  -H "Authorization: Bearer $TOKEN" \
  -d '{"completed": true}'

# Delete task (must be authenticated and owner)
curl -X DELETE http://localhost:8000/api/tasks/1/ \
  -H "Authorization: Bearer $TOKEN"

# Try to access without token (should get 401 Unauthorized)
curl http://localhost:8000/api/tasks/

# Note: Filtering, searching, and ordering will be covered in Level 2B
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
from rest_framework import serializers
from .models import UserProfile

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
# Import viewsets for CRUD operations
from rest_framework import viewsets
# Import permission class - users must be authenticated
from rest_framework.permissions import IsAuthenticated
# Import models and serializers
from .models import UserProfile
from .serializers import UserProfileSerializer

# ViewSet provides all CRUD operations automatically
class UserProfileViewSet(viewsets.ModelViewSet):
    # Serializer converts UserProfile objects to/from JSON
    serializer_class = UserProfileSerializer
    
    # Permission: Only authenticated users can access
    # Unauthenticated requests will get 401 Unauthorized
    permission_classes = [IsAuthenticated]

    # Override get_queryset to filter profiles by user
    # This ensures users only see their own profile
    # Called for list and retrieve operations
    def get_queryset(self):
        # Filter profiles where user matches the current logged-in user
        # self.request.user is automatically set by authentication
        return UserProfile.objects.filter(user=self.request.user)

    # Override perform_create to automatically set the user
    # This is called when a new profile is created via POST
    def perform_create(self, serializer):
        # Set user to the currently logged-in user
        # self.request.user is automatically set by authentication
        serializer.save(user=self.request.user)
```

## Throttling

### What is Throttling?

**Throttling** is similar to permissions, in that it determines if a request should be authorized. However, throttles indicate a **temporary state**, and are used to control the **rate of requests** that clients can make to an API.

**Why Throttling?**
- **Rate limiting**: Prevent API abuse and overuse
- **Resource protection**: Protect server resources from being overwhelmed
- **Fair usage**: Ensure fair distribution of API access
- **Cost control**: Limit expensive operations
- **Business tiers**: Implement different rate limits for different user tiers

**Real-world analogy**:
- **No throttling** = Unlimited free samples (everyone takes everything)
- **With throttling** = Limited samples per person (fair distribution)

**Important Security Note**: 
The application-level throttling that REST framework provides should **not** be considered a security measure or protection against brute forcing or denial-of-service attacks. Deliberately malicious actors will always be able to spoof IP origins. The built-in throttling implementations use Django's cache framework and non-atomic operations, which may sometimes result in some fuzziness.

**The application-level throttling provided by REST framework is intended for:**
- Implementing policies such as different business tiers
- Basic protections against service over-use
- Fair usage policies

### How Throttling is Determined

As with permissions and authentication, throttling in REST framework is always defined as a **list of classes**.

**The Process**:
1. Before running the main body of the view, each throttle in the list is checked
2. If any throttle check fails, an `exceptions.Throttled` exception will be raised
3. The main body of the view will not run
4. A `429 Too Many Requests` response is returned

**Why multiple throttles?**
- Different limits for authenticated vs unauthenticated users
- Different constraints on different parts of the API
- Both burst throttling rates and sustained throttling rates
- Example: Limit user to 60 requests per minute AND 1000 requests per day

### Setting the Throttling Policy

#### Global Throttling Policy

Set default throttling policy globally using `DEFAULT_THROTTLE_CLASSES` and `DEFAULT_THROTTLE_RATES`:

```python
# core/settings.py
# DRF configuration - applies to all views unless overridden
REST_FRAMEWORK = {
    # Throttle classes to use globally (applies to all views)
    # DRF will check all throttles - if any fails, request is throttled
    'DEFAULT_THROTTLE_CLASSES': [
        # AnonRateThrottle: Throttles unauthenticated users by IP address
        'rest_framework.throttling.AnonRateThrottle',
        # UserRateThrottle: Throttles authenticated users by user ID
        'rest_framework.throttling.UserRateThrottle'
    ],
    # Rate limits for each throttle class
    # Keys must match the 'scope' of throttle classes
    'DEFAULT_THROTTLE_RATES': {
        # 'anon' = scope for AnonRateThrottle
        # 100 requests per hour for anonymous (unauthenticated) users
        'anon': '100/hour',
        # 'user' = scope for UserRateThrottle
        # 1000 requests per hour for authenticated users
        'user': '1000/hour'
    }
}
```

**Rate Format**:
- The rates can be specified over a period of second, minute, hour or day
- The period must be specified after the `/` separator using `s`, `m`, `h` or `d`
- Examples: `'100/second'`, `'60/minute'`, `'1000/hour'`, `'10000/day'`
- Extended units are also allowed: `'second'`, `'minute'`, `'hour'`, `'day'`
- Abbreviations work too: `'sec'`, `'min'`, `'hr'` (only first character matters)

#### Per-View Throttling

Set throttling policy on a per-view basis using `APIView`:

```python
# api/views.py
# Import Response to return JSON responses
from rest_framework.response import Response
# Import throttle class
from rest_framework.throttling import UserRateThrottle
# Import APIView for class-based views
from rest_framework.views import APIView

# APIView provides basic view functionality
# Use this when you need custom logic (not using ViewSet)
class ExampleView(APIView):
    # Set throttle classes for this specific view
    # This overrides DEFAULT_THROTTLE_CLASSES from settings
    # UserRateThrottle will limit requests per user
    throttle_classes = [UserRateThrottle]

    # Handle GET requests
    def get(self, request, format=None):
        # Return simple response
        content = {
            'status': 'request was permitted'
        }
        return Response(content)
```

#### Per-ViewSet Throttling

Set throttling policy on a per-viewset basis:

```python
# api/views.py
# Import viewsets for CRUD operations
from rest_framework import viewsets
# Import throttle class
from rest_framework.throttling import UserRateThrottle
# Import models and serializers
from .models import Book
from .serializers import BookSerializer

# ModelViewSet provides all CRUD operations automatically
class BookViewSet(viewsets.ModelViewSet):
    # Base queryset
    queryset = Book.objects.all()
    # Serializer for data conversion
    serializer_class = BookSerializer
    
    # Set throttle classes for this ViewSet
    # Applies to all actions (list, create, retrieve, update, delete)
    # UserRateThrottle limits requests per authenticated user
    throttle_classes = [UserRateThrottle]
    # ...
```

#### Per-Action Throttling

Set throttle classes for specific actions using the `@action` decorator:

```python
# api/views.py
# Import viewsets for CRUD operations
from rest_framework import viewsets
# Import @action decorator to create custom endpoints
from rest_framework.decorators import action
# Import throttle class
from rest_framework.throttling import UserRateThrottle
# Import Response to return JSON
from rest_framework.response import Response

class BookViewSet(viewsets.ModelViewSet):
    # ViewSet-level throttling (applies to all default actions)
    # This throttle applies to: list, create, retrieve, update, delete
    throttle_classes = [UserRateThrottle]
    
    # @action decorator creates a custom endpoint
    # detail=True means it's for a specific book (POST /api/books/1/upload_cover/)
    # methods=["post"] means only POST requests are allowed
    # throttle_classes overrides the viewset-level throttle for this action only
    @action(detail=True, methods=["post"], throttle_classes=[UserRateThrottle])
    def upload_cover(self, request, pk=None):
        # This is a custom action (not part of standard CRUD)
        # It has its own throttle class that overrides viewset-level throttle_classes
        # Useful when you want different rate limits for different actions
        content = {
            'status': 'upload was permitted'
        }
        return Response(content)
```

**Note**: Throttle classes set in `@action` decorator will override any viewset-level class settings.

### How Clients are Identified

**IP Address Identification**:
- The `X-Forwarded-For` HTTP header and `REMOTE_ADDR` WSGI variable are used to uniquely identify client IP addresses
- If the `X-Forwarded-For` header is present, it will be used
- Otherwise, the value of the `REMOTE_ADDR` variable from the WSGI environment will be used

**NUM_PROXIES Setting**:
If you need to strictly identify unique client IP addresses, configure the number of application proxies:

```python
# core/settings.py
REST_FRAMEWORK = {
    'NUM_PROXIES': 1,  # Number of proxies in front of your application
}
```

**How it works**:
- If `NUM_PROXIES` is set to non-zero, the client IP will be identified as the last IP address in the `X-Forwarded-For` header, after excluding application proxy IP addresses
- If set to zero, the `REMOTE_ADDR` value will always be used

**Important**: If you configure `NUM_PROXIES`, all clients behind a unique NAT'd gateway will be treated as a single client.

### Setting up the Cache

**Cache Backend Requirements**:
- The throttle classes provided by REST framework use Django's cache backend
- You should make sure you've set appropriate cache settings
- The default value of `LocMemCache` backend should be okay for simple setups

**Custom Cache Backend**:
If you need to use a cache other than `'default'`, create a custom throttle class:

```python
# api/throttles.py
from django.core.cache import caches
from rest_framework.throttling import AnonRateThrottle

class CustomAnonRateThrottle(AnonRateThrottle):
    cache = caches['alternate']  # Use alternate cache backend
```

**Remember**: Set your custom throttle class in `'DEFAULT_THROTTLE_CLASSES'` settings key, or using the `throttle_classes` view attribute.

### A Note on Concurrency

**Race Conditions**:
- The built-in throttle implementations are open to race conditions
- Under high concurrency, they may allow a few extra requests through
- This is due to non-atomic operations in the cache framework

**When to Implement Custom Throttles**:
- If your project relies on guaranteeing the exact number of requests during concurrent requests
- If you need stricter rate limiting
- If you need custom throttling logic beyond what built-in throttles provide

**For production applications requiring strict rate limiting**, consider implementing your own throttle class with atomic operations or using external rate limiting services.

### API Reference

#### AnonRateThrottle

**What it does**: The `AnonRateThrottle` will only ever throttle **unauthenticated users**. The IP address of the incoming request is used to generate a unique key to throttle against.

**How rate is determined** (in order of preference):
1. The `rate` property on the class (if you override `AnonRateThrottle` and set the property)
2. The `DEFAULT_THROTTLE_RATES['anon']` setting

**When to use**:
- Restrict the rate of requests from unknown sources
- Protect against anonymous abuse
- Encourage user registration

**Example**:
```python
# core/settings.py
REST_FRAMEWORK = {
    'DEFAULT_THROTTLE_CLASSES': [
        'rest_framework.throttling.AnonRateThrottle',
    ],
    'DEFAULT_THROTTLE_RATES': {
        'anon': '100/hour',  # 100 requests per hour for anonymous users
    }
}
```

#### UserRateThrottle

**What it does**: The `UserRateThrottle` will throttle users to a given rate of requests across the API. The user id is used to generate a unique key to throttle against. Unauthenticated requests will fall back to using the IP address of the incoming request.

**How rate is determined** (in order of preference):
1. The `rate` property on the class (if you override `UserRateThrottle` and set the property)
2. The `DEFAULT_THROTTLE_RATES['user']` setting

**When to use**:
- Simple global rate restrictions per-user
- Fair usage policies
- Prevent individual users from overusing the API

**Example**:
```python
# core/settings.py
REST_FRAMEWORK = {
    'DEFAULT_THROTTLE_CLASSES': [
        'rest_framework.throttling.UserRateThrottle',
    ],
    'DEFAULT_THROTTLE_RATES': {
        'user': '1000/hour',  # 1000 requests per hour for authenticated users
    }
}
```

**Multiple User Throttles (Scoped Throttles)**:
An API may have multiple `UserRateThrottles` in place at the same time. Override `UserRateThrottle` and set a unique "scope" for each class:

```python
# api/throttles.py
from rest_framework.throttling import UserRateThrottle

class BurstRateThrottle(UserRateThrottle):
    scope = 'burst'

class SustainedRateThrottle(UserRateThrottle):
    scope = 'sustained'
```

```python
# core/settings.py
REST_FRAMEWORK = {
    'DEFAULT_THROTTLE_CLASSES': [
        'api.throttles.BurstRateThrottle',
        'api.throttles.SustainedRateThrottle'
    ],
    'DEFAULT_THROTTLE_RATES': {
        'burst': '60/min',      # 60 requests per minute
        'sustained': '1000/day'  # 1000 requests per day
    }
}
```

**What this means**:
- Users are limited to 60 requests per minute (burst limit)
- Users are also limited to 1000 requests per day (sustained limit)
- Both limits apply simultaneously
- Useful for preventing both short-term spikes and long-term overuse

#### ScopedRateThrottle

**What it does**: The `ScopedRateThrottle` class can be used to restrict access to **specific parts of the API**. This throttle will only be applied if the view that is being accessed includes a `.throttle_scope` property.

**How it works**:
- The unique throttle key is formed by concatenating the "scope" of the request with the unique user id or IP address
- The allowed request rate is determined by the `DEFAULT_THROTTLE_RATES` setting using a key from the request "scope"

**When to use**:
- Different rate limits for different API endpoints
- Protect resource-intensive endpoints
- Implement tiered access (e.g., read vs write operations)

**Example**:
```python
# api/views.py
from rest_framework.views import APIView

class ContactListView(APIView):
    throttle_scope = 'contacts'
    # ...

class ContactDetailView(APIView):
    throttle_scope = 'contacts'
    # ...

class UploadView(APIView):
    throttle_scope = 'uploads'
    # ...
```

```python
# core/settings.py
REST_FRAMEWORK = {
    'DEFAULT_THROTTLE_CLASSES': [
        'rest_framework.throttling.ScopedRateThrottle',
    ],
    'DEFAULT_THROTTLE_RATES': {
        'contacts': '1000/day',  # 1000 requests per day for contact views
        'uploads': '20/day'       # 20 requests per day for upload views
    }
}
```

**What this means**:
- User requests to either `ContactListView` or `ContactDetailView` would be restricted to a total of 1000 requests per day
- User requests to `UploadView` would be restricted to 20 requests per day
- Different scopes allow different rate limits for different parts of your API

### Custom Throttles

**Creating Custom Throttles**:
To create a custom throttle, override `BaseThrottle` and implement `.allow_request(self, request, view)`. The method should return `True` if the request should be allowed, and `False` otherwise.

**Optional `.wait()` Method**:
You may also override the `.wait()` method. If implemented, `.wait()` should return a recommended number of seconds to wait before attempting the next request, or `None`. The `.wait()` method will only be called if `.allow_request()` has previously returned `False`.

**Retry-After Header**:
If the `.wait()` method is implemented and the request is throttled, then a `Retry-After` header will be included in the response, telling the client how long to wait.

**Example 1: Random Rate Throttle**:
```python
# api/throttles.py
import random
from rest_framework import throttling

class RandomRateThrottle(throttling.BaseThrottle):
    """
    Randomly throttle 1 in every 10 requests.
    """
    def allow_request(self, request, view):
        return random.randint(1, 10) != 1
```

**Example 2: Custom Throttle with Wait Time**:
```python
# api/throttles.py
from rest_framework import throttling
from django.core.cache import cache
from django.utils import timezone

class CustomRateThrottle(throttling.BaseThrottle):
    """
    Custom throttle that limits to 10 requests per minute per IP.
    """
    def allow_request(self, request, view):
        # Get client IP
        ip = self.get_ident(request)
        
        # Create cache key
        cache_key = f'throttle_{ip}'
        
        # Get current request count
        request_count = cache.get(cache_key, 0)
        
        # Check if limit exceeded
        if request_count >= 10:
            return False
        
        # Increment count
        cache.set(cache_key, request_count + 1, 60)  # Expire in 60 seconds
        return True
    
    def wait(self):
        """
        Return number of seconds to wait before next request.
        """
        return 60  # Wait 60 seconds
    
    def get_ident(self, request):
        """
        Identify the client by IP address.
        """
        x_forwarded_for = request.META.get('HTTP_X_FORWARDED_FOR')
        if x_forwarded_for:
            ip = x_forwarded_for.split(',')[0]
        else:
            ip = request.META.get('REMOTE_ADDR')
        return ip
```

**Usage**:
```python
# api/views.py
from rest_framework import viewsets
from .throttles import CustomRateThrottle

class BookViewSet(viewsets.ModelViewSet):
    throttle_classes = [CustomRateThrottle]
    # ...
```

### Custom Throttling Example

**Per-Action Custom Throttle**:
```python
# api/throttles.py
# Import base throttle class
from rest_framework.throttling import UserRateThrottle

# Custom throttle that limits book creation
# Inherits from UserRateThrottle (throttles per user)
class BookCreateThrottle(UserRateThrottle):
    # Set rate limit: 5 requests per day
    # Format: 'number/period' where period is: second, minute, hour, day
    rate = '5/day'  # Limit book creation to 5 per day per user
```

```python
# api/views.py
# Import viewsets and custom throttle
from rest_framework import viewsets
from .throttles import BookCreateThrottle
from .models import Book
from .serializers import BookSerializer

class BookViewSet(viewsets.ModelViewSet):
    queryset = Book.objects.all()
    serializer_class = BookSerializer
    
    # Override get_throttles() to apply different throttles per action
    # This method is called by DRF to determine which throttles to check
    def get_throttles(self):
        """
        Override to apply different throttles to different actions.
        """
        # For create action (POST /api/books/), use strict throttle
        if self.action == 'create':
            # Return list of throttle instances
            # BookCreateThrottle will limit to 5 books per day
            return [BookCreateThrottle()]
        
        # For other actions (list, retrieve, update, delete), use default throttles
        # super().get_throttles() returns throttles from throttle_classes attribute
        return super().get_throttles()
```

**Alternative: Using throttle_classes attribute**:
```python
# api/views.py
from rest_framework import viewsets
from rest_framework.decorators import action
from .throttles import BookCreateThrottle

class BookViewSet(viewsets.ModelViewSet):
    throttle_classes = [UserRateThrottle]  # Default for all actions
    
    @action(detail=False, methods=['post'], throttle_classes=[BookCreateThrottle])
    def create_book(self, request):
        # This action uses BookCreateThrottle
        # ...
```

## Exercises

### Exercise 1: Complete Secure Book API (Authentication & Permissions)

1. Add owner field to Book model
2. Implement IsOwnerOrReadOnly permission class
3. Configure JWT authentication
4. Set up permissions (IsAuthenticatedOrReadOnly for public reading, IsOwnerOrReadOnly for editing)
5. Test all endpoints with cURL (create, read, update, delete with proper authentication)

### Exercise 2: UserProfile API

1. Create UserProfile model with OneToOne relationship to User
2. Create UserProfileSerializer and UserProfileViewSet
3. Allow users to view/edit only their own profile
4. Add authentication and permissions
5. Test with cURL

### Exercise 3: Custom Throttling

1. Create a custom throttle class that limits book creation to 5 per day per user
2. Apply the throttle to the BookViewSet's create action
3. Test throttling by making multiple requests
4. Verify that the 6th request returns 429 Too Many Requests

### Exercise 4: Abstract Models

1. Create a TimestampedModel abstract base class
2. Update your Task and Book models to inherit from TimestampedModel
3. Remove duplicate created_at and updated_at fields from models
4. Create and run migrations
5. Verify that timestamps still work correctly

## Common Errors and Solutions

### Error 1: `rest_framework.exceptions.AuthenticationFailed: Given token not valid for any token type`

**Error Message**:
```
rest_framework.exceptions.AuthenticationFailed: Given token not valid for any token type
```

**Why This Happens**:
- Token expired
- Invalid token format
- Token not sent correctly in header
- Wrong token type

**How to Fix**:
```bash
# Check token format
# Should be: Authorization: Bearer <token>
# NOT: Authorization: <token>
# NOT: Authorization: Token <token>

# Correct format
curl -H "Authorization: Bearer eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9..." \
  http://localhost:8000/api/tasks/

# Get new token if expired
curl -X POST http://localhost:8000/api/token/ \
  -H "Content-Type: application/json" \
  -d '{"username": "user", "password": "pass"}'
```

**Prevention**: 
- Always use `Bearer` prefix for JWT tokens
- Check token expiration
- Refresh token before it expires

---

### Error 2: `rest_framework.exceptions.PermissionDenied: You do not have permission to perform this action`

**Error Message**:
```
rest_framework.exceptions.PermissionDenied: You do not have permission to perform this action
```

**Why This Happens**:
- User not authenticated (no token/session)
- User authenticated but lacks required permissions
- Permission class too restrictive

**How to Fix**:
```python
# Check if user is authenticated
# In view, check:
from django.contrib.auth.models import AnonymousUser

print(request.user)  # Should be User object, not AnonymousUser
print(request.user.is_authenticated)  # Should be True

# Check permissions
# Option 1: Make endpoint public (if appropriate)
from rest_framework.permissions import AllowAny

class TaskViewSet(viewsets.ModelViewSet):
    permission_classes = [AllowAny]  # Anyone can access (rare for tasks)

# Option 2: Require authentication (most common for tasks)
from rest_framework.permissions import IsAuthenticated

class TaskViewSet(viewsets.ModelViewSet):
    permission_classes = [IsAuthenticated]  # Must be logged in

# Option 3: Custom permission check in class-based view
from rest_framework.views import APIView
from rest_framework.response import Response
from rest_framework import status

class TaskAPIView(APIView):
    permission_classes = [IsAuthenticated]
    
    def patch(self, request, pk):
        # Check custom permission
        if not request.user.has_perm('api.change_task'):
            return Response(
                {'error': 'Permission denied'}, 
                status=status.HTTP_403_FORBIDDEN
            )
        # ... rest of logic
```

**Prevention**: 
- Check permission classes in view
- Verify user is authenticated before accessing protected endpoints
- Test with different user roles

---

### Error 3: `django.contrib.auth.models.User.DoesNotExist: User matching query does not exist`

**Error Message**:
```
django.contrib.auth.models.User.DoesNotExist: User matching query does not exist
```

**Why This Happens**:
- Trying to get user that doesn't exist
- User ID in token doesn't match any user
- User was deleted but token still valid

**How to Fix**:
```python
# Option 1: Check if user exists
try:
    user = User.objects.get(id=user_id)
except User.DoesNotExist:
    return Response({'error': 'User not found'}, status=404)

# Option 2: Use get_object_or_404
from django.shortcuts import get_object_or_404
user = get_object_or_404(User, id=user_id)

# Option 3: Handle in JWT token validation
# In custom token serializer, verify user exists
def validate(self, attrs):
    user = authenticate(**attrs)
    if not user or not user.is_active:
        raise serializers.ValidationError('User not found or inactive')
    return attrs
```

**Prevention**: Always verify user exists before using, especially in token validation.

---

### Error 4: `rest_framework.exceptions.ValidationError: {'refresh': ['This field is required.']}`

**Error Message**:
```
rest_framework.exceptions.ValidationError: {'refresh': ['This field is required.']}
```

**Why This Happens**:
- Trying to refresh token without providing refresh token
- Sending access token instead of refresh token
- Missing refresh token in request

**How to Fix**:
```bash
# WRONG - Using access token
curl -X POST http://localhost:8000/api/token/refresh/ \
  -H "Content-Type: application/json" \
  -d '{"refresh": "access_token_here"}'  # ❌ Wrong token type

# CORRECT - Use refresh token
curl -X POST http://localhost:8000/api/token/refresh/ \
  -H "Content-Type: application/json" \
  -d '{"refresh": "refresh_token_here"}'  # ✅ Correct

# Get tokens first
TOKENS=$(curl -X POST http://localhost:8000/api/token/ \
  -H "Content-Type: application/json" \
  -d '{"username": "user", "password": "pass"}')

# Extract refresh token
REFRESH=$(echo $TOKENS | jq -r '.refresh')

# Use refresh token
curl -X POST http://localhost:8000/api/token/refresh/ \
  -H "Content-Type: application/json" \
  -d "{\"refresh\": \"$REFRESH\"}"
```

**Prevention**: 
- Store both access and refresh tokens
- Use refresh token only for refreshing
- Don't confuse access and refresh tokens

---

### Error 5: `django.db.utils.IntegrityError: NOT NULL constraint failed: api_task.owner_id`

**Error Message**:
```
django.db.utils.IntegrityError: NOT NULL constraint failed: api_task.owner_id
```

**Why This Happens**:
- Trying to create object without required ForeignKey
- Owner not set in serializer
- `perform_create` not setting owner

**How to Fix**:
```python
# WRONG - Owner not set
class TaskViewSet(viewsets.ModelViewSet):
    def create(self, request):
        serializer = TaskSerializer(data=request.data)
        serializer.save()  # ❌ Owner not set!

# CORRECT - Set owner in perform_create
class TaskViewSet(viewsets.ModelViewSet):
    serializer_class = TaskSerializer
    permission_classes = [IsAuthenticated]
    
    def perform_create(self, serializer):
        serializer.save(owner=self.request.user)  # ✅ Owner set automatically

# Alternative: Set owner in serializer
class TaskSerializer(serializers.ModelSerializer):
    def create(self, validated_data):
        validated_data['owner'] = self.context['request'].user
        return super().create(validated_data)
```

**Prevention**: Always set owner in `perform_create` or serializer's `create` method.

---

### General Debugging Tips for Level 2A

**1. Check Authentication**
```python
# In view, verify authentication
from django.contrib.auth.models import AnonymousUser

print("User:", request.user)
print("Authenticated:", request.user.is_authenticated)
print("Is anonymous:", isinstance(request.user, AnonymousUser))
```

**2. Test Permissions**
```python
# Check what permissions user has
print("Permissions:", request.user.get_all_permissions())
print("Is staff:", request.user.is_staff)
print("Is superuser:", request.user.is_superuser)
```

**3. Verify Token**
```python
# Decode JWT token to see contents
from rest_framework_simplejwt.tokens import AccessToken

token = request.META.get('HTTP_AUTHORIZATION', '').split(' ')[1]
decoded = AccessToken(token)
print("Token payload:", decoded.payload)
```

**4. Common Authentication Issues**
- **Token expired**: Get new token or refresh
- **Wrong header format**: Use `Authorization: Bearer <token>`
- **Token not sent**: Check request headers
- **User inactive**: Check `user.is_active`

**5. Common Permission Issues**
- **Too restrictive**: Check permission classes
- **User not authenticated**: Verify token/session
- **Wrong user**: Check if user owns resource
- **Missing permission**: Verify user has required permission

## Next Steps

Congratulations! You've completed Level 2A. You now know:

- ✅ Django authentication system
- ✅ Django Abstract Models
- ✅ JWT authentication setup
- ✅ Permissions (built-in and custom)
- ✅ Throttling and rate limiting
- ✅ Building secure APIs with authentication

**Ready for Level 2B?** Continue to [Level 2B: Intermediate - Filtering, Searching & Advanced Features](LEVEL_2B_INTERMEDIATE.md) to learn about filtering, searching, ordering, pagination, exception handling, and testing!

---

**Resources:**
- [DRF Authentication](https://www.django-rest-framework.org/api-guide/authentication/)
- [DRF Permissions](https://www.django-rest-framework.org/api-guide/permissions/)
- [djangorestframework-simplejwt](https://github.com/jazzband/djangorestframework-simplejwt)

