# Level 2B: Intermediate - Filtering, Searching & Advanced Features

## Goal

Build APIs with powerful filtering, searching, ordering, pagination, and robust error handling. By the end of this level, you'll be able to create production-ready APIs with advanced query capabilities and professional error handling.

## Note on Examples

This guide builds on **Level 2A** where you learned authentication and permissions. We'll use **Task API** and **Book API** as examples throughout, adding filtering, searching, ordering, and pagination to the secure APIs you built in Level 2A.

**Prerequisites**: Complete [Level 2A](LEVEL_2A_INTERMEDIATE.md) first to understand authentication and permissions.

You can apply all concepts to any model (Task, Book, Product, Post, etc.).

## Table of Contents

1. [Filtering](#filtering)
2. [Searching](#searching)
3. [Ordering](#ordering)
4. [Pagination](#pagination)
5. [Exception Handling](#exception-handling)
6. [Testing Authenticated APIs](#testing-authenticated-apis)
7. [Exercises](#exercises)
8. [Common Errors and Solutions](#common-errors-and-solutions)
9. [Add-ons](#add-ons)

## Filtering

### What is Filtering?

**Filtering** allows clients to request specific subsets of data based on criteria.

**The Problem**: Without filtering, clients get ALL data, even if they only need a subset.

**Example**:
- Without filtering: `GET /api/tasks/` → Returns all tasks (10,000 tasks) ❌
- With filtering: `GET /api/tasks/?completed=false` → Returns only incomplete tasks ✅

**Why Filtering?**
- **Performance**: Return only needed data (faster queries)
- **User experience**: Users find what they're looking for
- **Bandwidth**: Less data transferred
- **Database efficiency**: Queries only relevant records

**Real-world analogy**:
- **No filtering** = Library with no organization (find book by searching all shelves)
- **With filtering** = Library with sections (go directly to "Fiction" section)

**Common Filter Types**:
- **Exact match**: `?completed=true` (completed exactly equals true)
- **Contains**: `?title__icontains=django` (title contains "django")
- **Boolean**: `?completed=false` (filter by boolean field)
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
# Add django_filters to installed apps
INSTALLED_APPS = [
    # ... other apps ...
    'django_filters',  # Package for advanced filtering
    'rest_framework',  # Django REST Framework
]

# DRF configuration - applies to all views unless overridden
REST_FRAMEWORK = {
    # Filter backends: Systems that handle filtering, searching, ordering
    # These are checked in order - first one that can handle the request is used
    'DEFAULT_FILTER_BACKENDS': [
        # DjangoFilterBackend: Handles filtering with django-filter (exact matches, ranges, etc.)
        'django_filters.rest_framework.DjangoFilterBackend',
        # SearchFilter: Handles searching across multiple fields (built into DRF)
        'rest_framework.filters.SearchFilter',
        # OrderingFilter: Handles sorting results (built into DRF)
        'rest_framework.filters.OrderingFilter',
    ],
}
```

### Basic Filtering

**How it works**:
1. Client adds query parameters to URL: `?completed=false`
2. Django Filter intercepts request
3. Applies filters to queryset
4. Returns filtered results

**Why `filter_backends`?**
- Tells DRF which filtering system to use
- Can use multiple backends (filtering + searching + ordering)
- Each backend handles different aspects

```python
# api/views.py
# Import DjangoFilterBackend for filtering functionality
from django_filters.rest_framework import DjangoFilterBackend
# Import viewsets and filters
from rest_framework import viewsets, filters
# Import models and serializers
from .models import Task
from .serializers import TaskSerializer

# ViewSet provides all CRUD operations automatically
class TaskViewSet(viewsets.ModelViewSet):
    # Base queryset - all tasks in database
    queryset = Task.objects.all()
    # Serializer for data conversion
    serializer_class = TaskSerializer
    
    # Enable filtering by adding filter backends
    # filter_backends is a list - can have multiple backends (filtering + searching + ordering)
    filter_backends = [DjangoFilterBackend]  # Enable filtering with django-filter
    
    # Simple filtering: List fields that can be filtered
    # filterset_fields allows exact match filtering on these fields
    # Example: ?completed=false will filter tasks where completed equals False
    filterset_fields = ['completed', 'created_at']  # Which fields can be filtered
```

**Explanation**:
- **`filter_backends`**: List of filtering systems to use
- **`filterset_fields`**: Simple list of fields that can be filtered
- **Exact match**: `?completed=false` finds tasks where completed exactly equals False

**Usage:**

```bash
# Filter by completed status
curl -H "Authorization: Bearer $TOKEN" \
  http://localhost:8000/api/tasks/?completed=false
```

**What happens**:
- URL parameter `completed=false`
- Django Filter finds tasks where `completed` field equals False
- Returns only incomplete tasks

```bash
# Filter by multiple fields (if you have date field)
curl -H "Authorization: Bearer $TOKEN" \
  "http://localhost:8000/api/tasks/?completed=false&created_at=2024-01-01"
```

**What happens**:
- Both filters applied: `completed=false` AND `created_at=2024-01-01`
- Returns tasks matching BOTH criteria
- Filters are combined with AND logic

### Advanced Filtering with FilterSet

**Why use FilterSet instead of `filterset_fields`?**
- **Advanced lookups**: `icontains`, `gte`, `lte`, etc.
- **Custom field names**: `created_after` instead of `created_at__gte`
- **More control**: Complex filtering logic
- **Better UX**: User-friendly filter names

```python
# api/filters.py
# Import django_filters for advanced filtering
import django_filters
# Import the model to filter
from .models import Task

# FilterSet class defines how filtering works
# More powerful than filterset_fields - allows custom field names and advanced lookups
class TaskFilter(django_filters.FilterSet):
    # Custom filter: Filter by title containing text (case-insensitive)
    # lookup_expr='icontains' means "contains" (case-insensitive)
    # Example: ?title=django finds tasks with "django" in title
    title = django_filters.CharFilter(lookup_expr='icontains')
    
    # Custom filter: Filter by date range (created after a date)
    # field_name='created_at' = which model field to filter
    # lookup_expr='gte' = greater than or equal (created on or after this date)
    # Example: ?created_after=2024-01-01 finds tasks created on or after Jan 1, 2024
    created_after = django_filters.DateFilter(field_name='created_at', lookup_expr='gte')
    
    # Custom filter: Filter by date range (created before a date)
    # lookup_expr='lte' = less than or equal (created on or before this date)
    # Example: ?created_before=2024-12-31 finds tasks created before Dec 31, 2024
    created_before = django_filters.DateFilter(field_name='created_at', lookup_expr='lte')
    
    # Filter by completed status (exact match)
    # BooleanFilter handles True/False values
    # Example: ?completed=false finds incomplete tasks
    completed = django_filters.BooleanFilter()

    class Meta:
        # Model to filter
        model = Task
        # Fields that can be used in URL query parameters
        # Must match the filter field names defined above
        fields = ['completed', 'title', 'created_after', 'created_before']
```

**Explanation**:
- **`title`**: `?title=django` finds tasks with "django" in title (case-insensitive)
- **`created_after`**: `?created_after=2024-01-01` finds tasks created on or after date
- **`created_before`**: `?created_before=2024-12-31` finds tasks created before date
- **`completed`**: `?completed=false` finds incomplete tasks

```python
# api/views.py
# Import the FilterSet class we created
from .filters import TaskFilter
# Import other necessary classes
from rest_framework import viewsets
from .models import Task
from .serializers import TaskSerializer

class TaskViewSet(viewsets.ModelViewSet):
    # Base queryset
    queryset = Task.objects.all()
    # Serializer for data conversion
    serializer_class = TaskSerializer
    
    # Use FilterSet class instead of simple filterset_fields
    # filterset_class provides advanced filtering (custom field names, date ranges, etc.)
    # When client sends ?title=django&created_after=2024-01-01, TaskFilter handles it
    filterset_class = TaskFilter  # Use FilterSet instead of filterset_fields
```

**Usage:**
```bash
# Filter by title containing "django"
curl -H "Authorization: Bearer $TOKEN" \
  "http://localhost:8000/api/tasks/?title=django"

# Filter by date range
curl -H "Authorization: Bearer $TOKEN" \
  "http://localhost:8000/api/tasks/?created_after=2024-01-01&created_before=2024-12-31"

# Combine filters
curl -H "Authorization: Bearer $TOKEN" \
  "http://localhost:8000/api/tasks/?completed=false&title=important"
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
# Import viewsets and filters
from rest_framework import viewsets, filters
# Import models and serializers
from .models import Task
from .serializers import TaskSerializer

class TaskViewSet(viewsets.ModelViewSet):
    # Base queryset
    queryset = Task.objects.all()
    # Serializer for data conversion
    serializer_class = TaskSerializer
    
    # Enable searching by adding SearchFilter to filter_backends
    # SearchFilter is built into DRF (no extra package needed)
    filter_backends = [filters.SearchFilter]  # Enable searching
    
    # List of fields to search in
    # When user sends ?search=django, DRF searches for "django" in these fields
    # Uses case-insensitive partial matching by default
    # Returns tasks where ANY field contains the search term (OR logic)
    search_fields = ['title', 'desc']  # Fields to search in
```

**Explanation**:
- **`SearchFilter`**: DRF's built-in search backend (no extra package needed)
- **`search_fields`**: List of fields to search in
- **Default behavior**: Case-insensitive, partial matching
- **OR logic**: Matches if search term found in ANY field

**Usage:**

```bash
# Search in title and description
curl -H "Authorization: Bearer $TOKEN" \
  "http://localhost:8000/api/tasks/?search=django"
```

**What happens**:
- Searches for "django" in `title` and `desc` fields
- Returns tasks where ANY field contains "django"
- Case-insensitive: "Django", "DJANGO", "django" all match
- Partial match: "Learn Django REST Framework" matches "django"

### Search Lookup Expressions

**Advanced search patterns:**

```python
class TaskViewSet(viewsets.ModelViewSet):
    search_fields = [
        '^title',      # Starts with (e.g., "Learn" matches "Learn Django")
        '=title',      # Exact match (e.g., "Learn Django" only)
        '@desc',       # Full text search (PostgreSQL only)
        '$title',      # Regex search (advanced)
    ]
```

**When to use each:**
- **`^title`**: When you want tasks starting with specific text
- **`=title`**: When you need exact match
- **`@desc`**: PostgreSQL full-text search (most powerful, database-specific)
- **`$title`**: Complex pattern matching (use sparingly)

**Most common**: Just use field name without prefix for simple case-insensitive search.

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
# Import viewsets and filters
from rest_framework import viewsets, filters
# Import models and serializers
from .models import Task
from .serializers import TaskSerializer

class TaskViewSet(viewsets.ModelViewSet):
    # Base queryset
    queryset = Task.objects.all()
    # Serializer for data conversion
    serializer_class = TaskSerializer
    
    # Enable ordering by adding OrderingFilter to filter_backends
    # OrderingFilter is built into DRF (no extra package needed)
    filter_backends = [filters.OrderingFilter]  # Enable ordering
    
    # List of fields users can sort by
    # Security: Prevents sorting by sensitive fields (like password hash)
    # Users can only sort by fields in this list
    ordering_fields = ['title', 'completed', 'created_at', 'updated_at']  # Allowed fields
    
    # Default ordering if user doesn't specify ?ordering= parameter
    # '-' prefix means descending (newest/highest first)
    # Without '-' means ascending (oldest/lowest first)
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
curl -H "Authorization: Bearer $TOKEN" \
  "http://localhost:8000/api/tasks/?ordering=title"
```

**What happens**: Tasks sorted alphabetically by title (A to Z)

```bash
# Order by created_at (descending: newest first)
curl -H "Authorization: Bearer $TOKEN" \
  "http://localhost:8000/api/tasks/?ordering=-created_at"
```

**What happens**: Tasks sorted by creation date, newest first (most recent at top)

```bash
# Multiple fields: First by completed status, then by created_at (descending)
curl -H "Authorization: Bearer $TOKEN" \
  "http://localhost:8000/api/tasks/?ordering=completed,-created_at"
```

**What happens**: 
1. First sorts by completed (False first, then True)
2. Within same completion status, sorts by created_at (newest first)
3. Useful for showing incomplete tasks first, then completed

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
# DRF configuration - applies to all list endpoints globally
REST_FRAMEWORK = {
    # Pagination class to use for all list views
    # PageNumberPagination = User-friendly pagination with page numbers (?page=2)
    'DEFAULT_PAGINATION_CLASS': 'rest_framework.pagination.PageNumberPagination',
    
    # How many records to show per page
    # Can be overridden per-view if needed
    # Example: If you have 100 tasks, this shows 10 per page (10 pages total)
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
    "next": "http://localhost:8000/api/tasks/?page=2",  // URL for next page
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
curl -H "Authorization: Bearer $TOKEN" \
  "http://localhost:8000/api/tasks/?page=2"
```

**What happens**: Returns records 11-20 (page 2, assuming 10 per page)

**Example response:**
```json
{
    "count": 50,
    "next": "http://localhost:8000/api/tasks/?page=3",
    "previous": "http://localhost:8000/api/tasks/?page=1",
    "results": [
        {"id": 11, "title": "Task 11", ...},
        {"id": 12, "title": "Task 12", ...},
        ...
    ]
}
```

### LimitOffsetPagination

```python
# api/pagination.py
# Import LimitOffsetPagination class
from rest_framework.pagination import LimitOffsetPagination

# Custom pagination class using limit/offset approach
# LimitOffsetPagination = Flexible pagination (?limit=20&offset=40)
# Good for APIs where clients control exactly how many records to fetch
class BookLimitOffsetPagination(LimitOffsetPagination):
    # Default number of records if limit is not specified
    # Example: ?offset=40 (no limit) returns default_limit records starting from offset 40
    default_limit = 10
    
    # URL parameter name for limit (how many records to return)
    # Example: ?limit=20 returns 20 records
    limit_query_param = 'limit'
    
    # URL parameter name for offset (how many records to skip)
    # Example: ?offset=40 skips first 40 records, returns next default_limit records
    offset_query_param = 'offset'
    
    # Maximum limit allowed (prevents users from requesting too many records at once)
    # Example: If user sends ?limit=10000, it's capped at max_limit=100
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
# Import CursorPagination class
from rest_framework.pagination import CursorPagination

# Custom pagination class using cursor-based approach
# CursorPagination = Best for large datasets, provides stable pagination
# Uses a cursor (pointer) instead of page numbers - more stable when data changes
class BookCursorPagination(CursorPagination):
    # Number of records per page
    page_size = 10
    
    # Field to use for ordering (required for cursor pagination)
    # Cursor pagination needs a consistent ordering to work
    # '-' prefix = descending (newest first)
    ordering = '-created_at'
    
    # URL parameter name for cursor
    # Example: ?cursor=cD0yMDI0LTAxLTAx returns next page after that cursor
    cursor_query_param = 'cursor'
```

## Step-by-Step: Secure Book API

**Note**: This section uses Book API as an example of content that can be public or user-owned. The same patterns apply to Task API or any other model.

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
from rest_framework.response import Response
from rest_framework.throttling import UserRateThrottle
from rest_framework.views import APIView

class ExampleView(APIView):
    throttle_classes = [UserRateThrottle]

    def get(self, request, format=None):
        content = {
            'status': 'request was permitted'
        }
        return Response(content)
```

#### Per-ViewSet Throttling

Set throttling policy on a per-viewset basis:

```python
# api/views.py
from rest_framework import viewsets
from rest_framework.throttling import UserRateThrottle
from .models import Book
from .serializers import BookSerializer

class BookViewSet(viewsets.ModelViewSet):
    queryset = Book.objects.all()
    serializer_class = BookSerializer
    throttle_classes = [UserRateThrottle]
    # ...
```

#### Per-Action Throttling

Set throttle classes for specific actions using the `@action` decorator:

```python
# api/views.py
from rest_framework import viewsets
from rest_framework.decorators import action
from rest_framework.throttling import UserRateThrottle
from rest_framework.response import Response

class BookViewSet(viewsets.ModelViewSet):
    # ViewSet-level throttling (applies to all actions)
    throttle_classes = [UserRateThrottle]
    
    @action(detail=True, methods=["post"], throttle_classes=[UserRateThrottle])
    def upload_cover(self, request, pk=None):
        # This action has its own throttle class
        # Overrides viewset-level throttle_classes
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
from rest_framework.throttling import UserRateThrottle

class BookCreateThrottle(UserRateThrottle):
    rate = '5/day'  # Limit book creation to 5 per day
```

```python
# api/views.py
from rest_framework import viewsets
from .throttles import BookCreateThrottle
from .models import Book
from .serializers import BookSerializer

class BookViewSet(viewsets.ModelViewSet):
    queryset = Book.objects.all()
    serializer_class = BookSerializer
    
    def get_throttles(self):
        """
        Override to apply different throttles to different actions.
        """
        if self.action == 'create':
            return [BookCreateThrottle()]
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

## Exception Handling

### What is Exception Handling?

**Exception Handling** allows you to customize how errors are returned to clients.

**The Problem**: By default, DRF returns errors in its own format. You might want:
- Consistent error format across all endpoints
- More user-friendly error messages
- Additional error details
- Custom error codes

**Why Custom Exception Handling?**
- **Consistency**: All errors follow same format
- **User experience**: Clear, helpful error messages
- **Debugging**: Include useful debugging information
- **Security**: Don't expose sensitive details
- **Professional**: Production-ready error responses

**Real-world analogy**:
- **Default errors** = Generic error messages (not helpful)
- **Custom errors** = Clear, actionable error messages (helpful)

### Basic Exception Handler

**How it works**:
1. Exception occurs in view
2. DRF calls your custom exception handler
3. Handler processes exception
4. Returns custom error response

```python
# api/exceptions.py (create this file)
# Import DRF's default exception handler - we'll call it first
from rest_framework.views import exception_handler
# Import Response to return custom error responses
from rest_framework.response import Response
# Import HTTP status codes
from rest_framework import status
# Import Django settings to check DEBUG mode
from django.conf import settings
# Import logging to log errors for debugging
import logging

# Create logger for this module
# Logs will help you debug issues in production
logger = logging.getLogger(__name__)

# Custom exception handler function
# DRF calls this function whenever an exception occurs in a view
# exc = the exception that was raised
# context = dictionary with request, view, and other context
def custom_exception_handler(exc, context):
    """
    Custom exception handler for DRF.
    Returns consistent error format for all exceptions.
    """
    # Step 1: Call DRF's default exception handler first
    # This handles common DRF exceptions (ValidationError, PermissionDenied, etc.)
    # Returns Response object if it handled the exception, None otherwise
    response = exception_handler(exc, context)
    
    # Step 2: If DRF handled it, customize the response format
    if response is not None:
        # Create custom error response structure
        # This ensures all errors follow the same format
        custom_response_data = {
            'success': False,  # Quick indicator that request failed
            'error': {
                'status_code': response.status_code,  # HTTP status code (400, 401, 403, etc.)
                'message': get_error_message(response.status_code, exc),  # User-friendly message
                'details': response.data,  # Detailed error information
                'type': exc.__class__.__name__  # Exception class name (for debugging)
            }
        }
        # Replace DRF's default error format with our custom format
        response.data = custom_response_data
        
        # Log error for debugging (helps track issues in production)
        # exc_info=True includes full traceback in logs
        logger.error(f"Error {response.status_code}: {exc}", exc_info=True)
    
    # Step 3: Handle exceptions DRF doesn't catch (unexpected errors)
    else:
        # For unexpected errors (500 Internal Server Error)
        # These are errors DRF doesn't know how to handle
        custom_response_data = {
            'success': False,
            'error': {
                'status_code': 500,
                'message': 'An unexpected error occurred',
                # In DEBUG mode: show actual error (helpful for development)
                # In production: show generic message (don't expose sensitive details)
                'details': str(exc) if settings.DEBUG else 'Internal server error',
                'type': exc.__class__.__name__
            }
        }
        # Create new Response for unexpected errors
        response = Response(custom_response_data, status=status.HTTP_500_INTERNAL_SERVER_ERROR)
        
        # Log unexpected errors with full traceback
        # logger.exception() automatically includes traceback
        logger.exception(f"Unexpected error: {exc}")
    
    # Return the response (either customized DRF response or new 500 response)
    return response

def get_error_message(status_code, exc):
    """
    Get user-friendly error message based on status code.
    Converts technical HTTP status codes into readable messages for users.
    """
    # Dictionary mapping HTTP status codes to user-friendly messages
    # These messages are shown to API clients (more helpful than just status codes)
    messages = {
        400: 'Invalid request data',  # Bad Request - client sent invalid data
        401: 'Authentication required',  # Unauthorized - need to login
        403: 'Permission denied',  # Forbidden - logged in but no permission
        404: 'Resource not found',  # Not Found - resource doesn't exist
        405: 'Method not allowed',  # Method Not Allowed - wrong HTTP method
        429: 'Too many requests',  # Too Many Requests - rate limit exceeded
        500: 'Internal server error',  # Internal Server Error - server problem
    }
    # Return message for status code, or default message if status code not in dictionary
    # .get() returns None if key not found, so we provide default 'An error occurred'
    return messages.get(status_code, 'An error occurred')
```

**Add to settings.py:**

```python
# core/settings.py
# DRF configuration
REST_FRAMEWORK = {
    # ... other settings (authentication, permissions, etc.) ...
    
    # Set custom exception handler
    # This tells DRF to use our custom_exception_handler function instead of the default one
    # Path format: 'app_name.module_name.function_name'
    # When any exception occurs in any view, DRF will call this function
    'EXCEPTION_HANDLER': 'api.exceptions.custom_exception_handler',
}
```

**Why this location?**
- Centralized: All errors go through one handler
- Consistent: Same format for all errors
- Maintainable: Update error format in one place

### Handling Authentication Errors

**Customize authentication error responses:**

```python
# api/exceptions.py
from rest_framework.views import exception_handler
from rest_framework.exceptions import AuthenticationFailed, NotAuthenticated, PermissionDenied
from rest_framework import status

def custom_exception_handler(exc, context):
    response = exception_handler(exc, context)
    
    if response is not None:
        # Handle authentication errors specifically
        if isinstance(exc, (AuthenticationFailed, NotAuthenticated)):
            custom_response_data = {
                'success': False,
                'error': {
                    'status_code': response.status_code,
                    'message': 'Authentication failed. Please provide valid credentials.',
                    'details': {
                        'code': 'authentication_failed',
                        'hint': 'Make sure you include a valid token in the Authorization header'
                    },
                    'type': exc.__class__.__name__
                }
            }
            response.data = custom_response_data
        
        # Handle permission errors
        elif isinstance(exc, PermissionDenied):
            custom_response_data = {
                'success': False,
                'error': {
                    'status_code': response.status_code,
                    'message': 'You do not have permission to perform this action',
                    'details': {
                        'code': 'permission_denied',
                        'required_permission': str(exc)
                    },
                    'type': exc.__class__.__name__
                }
            }
            response.data = custom_response_data
        
        # Handle other errors
        else:
            custom_response_data = {
                'success': False,
                'error': {
                    'status_code': response.status_code,
                    'message': get_error_message(response.status_code, exc),
                    'details': response.data,
                    'type': exc.__class__.__name__
                }
            }
            response.data = custom_response_data
    
    return response
```

**Why handle authentication errors separately?**
- **User-friendly**: Clear message about what went wrong
- **Actionable**: Tells user how to fix it
- **Security**: Doesn't expose sensitive details
- **Consistent**: Same format for all auth errors

### Handling Validation Errors

**Customize validation error format:**

```python
# api/exceptions.py
from rest_framework.views import exception_handler
from rest_framework.exceptions import ValidationError

def custom_exception_handler(exc, context):
    response = exception_handler(exc, context)
    
    if response is not None:
        # Handle validation errors
        if isinstance(exc, ValidationError):
            # Format field errors nicely
            formatted_errors = {}
            if isinstance(response.data, dict):
                for field, errors in response.data.items():
                    if isinstance(errors, list):
                        formatted_errors[field] = errors[0] if errors else 'Invalid value'
                    else:
                        formatted_errors[field] = str(errors)
            
            custom_response_data = {
                'success': False,
                'error': {
                    'status_code': response.status_code,
                    'message': 'Validation failed',
                    'details': {
                        'code': 'validation_error',
                        'fields': formatted_errors
                    },
                    'type': exc.__class__.__name__
                }
            }
            response.data = custom_response_data
        
        # ... handle other errors ...
    
    return response
```

**Why format validation errors?**
- **Clarity**: Each field error is clear
- **User-friendly**: Easy to understand what's wrong
- **Actionable**: User knows which fields to fix

### Handling JWT Token Errors

**Specific handling for JWT errors:**

```python
# api/exceptions.py
from rest_framework.views import exception_handler
from rest_framework_simplejwt.exceptions import InvalidToken, TokenError

def custom_exception_handler(exc, context):
    response = exception_handler(exc, context)
    
    if response is not None:
        # Handle JWT token errors
        if isinstance(exc, (InvalidToken, TokenError)):
            error_message = str(exc)
            
            # Provide helpful messages based on error
            if 'expired' in error_message.lower():
                message = 'Token has expired. Please refresh your token.'
                code = 'token_expired'
            elif 'invalid' in error_message.lower():
                message = 'Invalid token. Please login again.'
                code = 'token_invalid'
            else:
                message = 'Token error. Please login again.'
                code = 'token_error'
            
            custom_response_data = {
                'success': False,
                'error': {
                    'status_code': 401,
                    'message': message,
                    'details': {
                        'code': code,
                        'hint': 'Use /api/token/ to get a new token, or /api/token/refresh/ to refresh'
                    },
                    'type': exc.__class__.__name__
                }
            }
            response.data = custom_response_data
            response.status_code = 401
        
        # ... handle other errors ...
    
    return response
```

**Why handle JWT errors specifically?**
- **Clear guidance**: Tells user how to fix token issues
- **Actionable**: Provides endpoint URLs for token refresh
- **User-friendly**: Explains what went wrong

### Production vs Development Error Messages

**Show detailed errors in development, generic in production:**

```python
# api/exceptions.py
from rest_framework.views import exception_handler
from django.conf import settings
import traceback

def custom_exception_handler(exc, context):
    response = exception_handler(exc, context)
    
    if response is not None:
        # In development: show detailed errors
        # In production: show generic errors
        if settings.DEBUG:
            error_details = {
                'message': str(exc),
                'traceback': traceback.format_exc() if hasattr(exc, '__traceback__') else None,
                'context': {
                    'view': context.get('view').__class__.__name__ if context.get('view') else None,
                    'request_method': context.get('request').method if context.get('request') else None,
                }
            }
        else:
            # Production: don't expose sensitive details
            error_details = {
                'message': get_error_message(response.status_code, exc) if 'get_error_message' in globals() else 'An error occurred',
                'code': 'error'
            }
        
        custom_response_data = {
            'success': False,
            'error': {
                'status_code': response.status_code,
                **error_details,
                'type': exc.__class__.__name__
            }
        }
        response.data = custom_response_data
    
    return response
```

**Why different messages?**
- **Development**: Detailed errors help debugging
- **Production**: Generic errors protect security
- **Best practice**: Never expose stack traces in production

### Complete Example

**Full-featured exception handler:**

```python
# api/exceptions.py
from rest_framework.views import exception_handler
from rest_framework.response import Response
from rest_framework import status
from rest_framework.exceptions import (
    AuthenticationFailed,
    NotAuthenticated,
    PermissionDenied,
    ValidationError,
    NotFound,
    MethodNotAllowed
)
from rest_framework_simplejwt.exceptions import InvalidToken, TokenError
from django.conf import settings
import logging
import traceback

logger = logging.getLogger(__name__)

def custom_exception_handler(exc, context):
    """
    Comprehensive exception handler for DRF.
    Provides consistent, user-friendly error responses.
    """
    # Call DRF's default exception handler
    response = exception_handler(exc, context)
    
    # Request object
    request = context.get('request')
    
    if response is not None:
        # Customize based on exception type
        error_data = {
            'success': False,
            'error': {}
        }
        
        # Authentication errors
        if isinstance(exc, (AuthenticationFailed, NotAuthenticated)):
            error_data['error'] = {
                'status_code': 401,
                'message': 'Authentication required',
                'details': {
                    'code': 'authentication_required',
                    'hint': 'Include a valid token in Authorization header: Bearer <token>'
                }
            }
        
        # Permission errors
        elif isinstance(exc, PermissionDenied):
            error_data['error'] = {
                'status_code': 403,
                'message': 'Permission denied',
                'details': {
                    'code': 'permission_denied',
                    'hint': 'You do not have permission to perform this action'
                }
            }
        
        # JWT token errors
        elif isinstance(exc, (InvalidToken, TokenError)):
            error_data['error'] = {
                'status_code': 401,
                'message': 'Invalid or expired token',
                'details': {
                    'code': 'token_error',
                    'hint': 'Get a new token at /api/token/ or refresh at /api/token/refresh/'
                }
            }
        
        # Validation errors
        elif isinstance(exc, ValidationError):
            formatted_errors = format_validation_errors(response.data)
            error_data['error'] = {
                'status_code': 400,
                'message': 'Validation failed',
                'details': {
                    'code': 'validation_error',
                    'fields': formatted_errors
                }
            }
        
        # Not found errors
        elif isinstance(exc, NotFound):
            error_data['error'] = {
                'status_code': 404,
                'message': 'Resource not found',
                'details': {
                    'code': 'not_found',
                    'hint': 'The requested resource does not exist'
                }
            }
        
        # Method not allowed
        elif isinstance(exc, MethodNotAllowed):
            error_data['error'] = {
                'status_code': 405,
                'message': 'Method not allowed',
                'details': {
                    'code': 'method_not_allowed',
                    'allowed_methods': exc.allowed_methods if hasattr(exc, 'allowed_methods') else []
                }
            }
        
        # Generic error
        else:
            error_data['error'] = {
                'status_code': response.status_code,
                'message': get_error_message(response.status_code),
                'details': response.data if settings.DEBUG else {'code': 'error'}
            }
        
        # Add exception type for debugging
        error_data['error']['type'] = exc.__class__.__name__
        
        # Log error
        logger.error(
            f"Error {error_data['error']['status_code']}: {exc}",
            extra={
                'request_path': request.path if request else None,
                'request_method': request.method if request else None,
            },
            exc_info=settings.DEBUG
        )
        
        response.data = error_data['error']
    
    # Handle unexpected errors (500)
    else:
        error_data = {
            'success': False,
            'error': {
                'status_code': 500,
                'message': 'Internal server error',
                'details': {
                    'code': 'server_error',
                    'message': str(exc) if settings.DEBUG else 'An unexpected error occurred'
                },
                'type': exc.__class__.__name__
            }
        }
        
        # Log unexpected errors
        logger.exception(f"Unexpected error: {exc}")
        
        response = Response(error_data['error'], status=status.HTTP_500_INTERNAL_SERVER_ERROR)
    
    return response

def format_validation_errors(errors):
    """Format DRF validation errors into user-friendly format"""
    formatted = {}
    
    if isinstance(errors, dict):
        for field, field_errors in errors.items():
            if isinstance(field_errors, list):
                # Take first error message
                formatted[field] = field_errors[0] if field_errors else 'Invalid value'
            elif isinstance(field_errors, dict):
                # Nested errors
                formatted[field] = format_validation_errors(field_errors)
            else:
                formatted[field] = str(field_errors)
    
    return formatted

def get_error_message(status_code):
    """Get user-friendly error message"""
    messages = {
        400: 'Bad request',
        401: 'Authentication required',
        403: 'Permission denied',
        404: 'Resource not found',
        405: 'Method not allowed',
        429: 'Too many requests',
        500: 'Internal server error',
    }
    return messages.get(status_code, 'An error occurred')
```

**Why this comprehensive handler?**
- **Handles all error types**: Authentication, permissions, validation, etc.
- **User-friendly messages**: Clear, actionable error messages
- **Consistent format**: Same structure for all errors
- **Production-ready**: Different messages for dev/prod
- **Logging**: Tracks all errors for debugging

### Error Response Format

**Standard error response structure:**

```json
{
    "success": false,
    "error": {
        "status_code": 401,
        "message": "Authentication required",
        "details": {
            "code": "authentication_required",
            "hint": "Include a valid token in Authorization header"
        },
        "type": "AuthenticationFailed"
    }
}
```

**Why this format?**
- **`success`**: Quick check if request succeeded
- **`error.status_code`**: HTTP status code
- **`error.message`**: Human-readable message
- **`error.details`**: Additional information
- **`error.type`**: Exception class name (for debugging)

### Testing Exception Handler

**Test your exception handler:**

```python
# api/tests.py
from rest_framework.test import APITestCase
from rest_framework import status
from django.contrib.auth.models import User

class ExceptionHandlerTest(APITestCase):
    def setUp(self):
        self.user = User.objects.create_user(
            username='testuser',
            password='testpass123'
        )
    
    def test_authentication_error(self):
        # Request without token
        response = self.client.get('/api/tasks/')
        self.assertEqual(response.status_code, 401)
        self.assertIn('error', response.data)
        self.assertEqual(response.data['error']['details']['code'], 'authentication_required')
    
    def test_validation_error(self):
        # Login first
        self.client.force_authenticate(user=self.user)
        
        # Invalid data - empty title
        response = self.client.post('/api/tasks/', {
            'title': '',  # Empty title should fail validation
            'completed': False
        })
        self.assertEqual(response.status_code, 400)
        self.assertIn('error', response.data)
        self.assertIn('fields', response.data['error']['details'])
        self.assertIn('title', response.data['error']['details']['fields'])
    
    def test_permission_denied(self):
        # Login as regular user
        self.client.force_authenticate(user=self.user)
        
        # Try to access admin-only endpoint (if you have one)
        # This would return 403 if permission denied
        # response = self.client.get('/api/admin/tasks/')
        # self.assertEqual(response.status_code, 403)
```

**Why test exception handler?**
- **Verify format**: Ensure errors follow expected format
- **Catch bugs**: Find issues in error handling
- **Documentation**: Shows how errors are returned

### Best Practices

**1. Consistent Format**
- Use same structure for all errors
- Include status code, message, details
- Make it easy for clients to parse

**2. User-Friendly Messages**
- Avoid technical jargon
- Provide actionable hints
- Explain what went wrong

**3. Security**
- Don't expose sensitive details in production
- Hide stack traces from users
- Log detailed errors server-side

**4. Logging**
- Log all errors for debugging
- Include request context
- Use appropriate log levels

**5. Error Codes**
- Use consistent error codes
- Make codes searchable
- Document all error codes

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

### Exercise 1: Complete Secure Book API (Filtering & Pagination)

1. Add filtering by author and date range to Book API
2. Add search functionality across title, author, and description
3. Add ordering by title, author, published_date, and created_at
4. Implement pagination (PageNumberPagination with 10 items per page)
5. Test all filtering, searching, ordering, and pagination with cURL

### Exercise 2: Task Filtering & Searching

1. Add priority field to Task model
2. Create TaskFilter with priority filtering and date range filtering
3. Add search functionality for title and description
4. Add ordering by priority, title, and created_at
5. Test filtering, searching, and ordering with cURL

### Exercise 3: Custom Exception Handler

1. Create custom exception handler with consistent error format
2. Add user-friendly error messages for common status codes
3. Configure exception handler in settings.py
4. Test error responses (send invalid data, access non-existent resource, etc.)
5. Verify error format is consistent across all endpoints

### Exercise 4: Advanced Pagination

1. Implement LimitOffsetPagination for Book API
2. Set default_limit to 20 and max_limit to 100
3. Test pagination with different limit and offset values
4. Compare with PageNumberPagination (which is easier to use?)

## Common Errors and Solutions

### Error 1: `django_filters.exceptions.FieldNotFound: Field 'title_contains' not found`

**Error Message**:
```
django_filters.exceptions.FieldNotFound: Field 'title_contains' not found
```

**Why This Happens**:
- Trying to filter by field that doesn't exist in model
- Field name typo in filterset_fields
- Custom filter field not defined before using in Meta.fields

**How to Fix**:
```python
# WRONG - Field 'title_contains' not defined
class TaskFilter(django_filters.FilterSet):
    # title_contains not defined here!
    
    class Meta:
        model = Task
        fields = ['title_contains']  # ❌ Field not defined

# CORRECT - Define field first
class TaskFilter(django_filters.FilterSet):
    # Define custom filter field
    title_contains = django_filters.CharFilter(
        field_name='title',  # Model field name
        lookup_expr='icontains'  # Lookup type
    )
    
    class Meta:
        model = Task
        fields = ['title_contains']  # ✅ Now defined
```

**Prevention**: Always define custom filter fields before listing them in `Meta.fields`.

---

### Error 2: `rest_framework.exceptions.ParseError: Malformed request`

**Error Message**:
```
rest_framework.exceptions.ParseError: Malformed request
```

**Why This Happens**:
- Invalid JSON in request body
- Missing Content-Type header
- Malformed query parameters

**How to Fix**:
```bash
# WRONG - Missing Content-Type header
curl -X POST http://localhost:8000/api/tasks/ \
  -d '{"title": "Task"}'

# CORRECT - Include Content-Type header
curl -X POST http://localhost:8000/api/tasks/ \
  -H "Content-Type: application/json" \
  -H "Authorization: Bearer $TOKEN" \
  -d '{"title": "Task"}'

# WRONG - Invalid JSON
curl -X POST http://localhost:8000/api/tasks/ \
  -H "Content-Type: application/json" \
  -d '{title: "Task"}'  # ❌ Missing quotes around key

# CORRECT - Valid JSON
curl -X POST http://localhost:8000/api/tasks/ \
  -H "Content-Type: application/json" \
  -d '{"title": "Task"}'  # ✅ Valid JSON
```

**Prevention**: 
- Always include `Content-Type: application/json` header
- Validate JSON syntax before sending
- Use proper JSON formatting

---

### Error 3: `django.core.exceptions.FieldError: Cannot resolve keyword 'title__icontains' into field`

**Error Message**:
```
django.core.exceptions.FieldError: Cannot resolve keyword 'title__icontains' into field
```

**Why This Happens**:
- Using lookup expression in wrong place
- Field doesn't exist in model
- Typo in field name

**How to Fix**:
```python
# WRONG - Using lookup expression directly in filterset_fields
class TaskViewSet(viewsets.ModelViewSet):
    filterset_fields = ['title__icontains']  # ❌ Can't use lookup expressions here

# CORRECT - Use FilterSet with custom field
class TaskFilter(django_filters.FilterSet):
    title = django_filters.CharFilter(lookup_expr='icontains')  # ✅ Define in FilterSet
    
    class Meta:
        model = Task
        fields = ['title']

class TaskViewSet(viewsets.ModelViewSet):
    filterset_class = TaskFilter  # ✅ Use FilterSet
```

**Prevention**: 
- Use `filterset_fields` only for exact matches
- Use `FilterSet` for advanced lookups (icontains, gte, lte, etc.)

---

### Error 4: `rest_framework.exceptions.NotFound: Invalid page.`

**Error Message**:
```
rest_framework.exceptions.NotFound: Invalid page.
```

**Why This Happens**:
- Requesting page number that doesn't exist
- Invalid page parameter format
- Page number out of range

**How to Fix**:
```bash
# WRONG - Page number too high (only 5 pages exist, requesting page 10)
curl "http://localhost:8000/api/tasks/?page=10"

# CORRECT - Check total pages first
curl "http://localhost:8000/api/tasks/" | jq '.count, .next, .previous'

# WRONG - Invalid page format
curl "http://localhost:8000/api/tasks/?page=abc"  # ❌ Not a number

# CORRECT - Use valid page number
curl "http://localhost:8000/api/tasks/?page=1"  # ✅ Valid
```

**Prevention**: 
- Always check response for `count` and `next` fields
- Handle pagination errors in client code
- Validate page numbers before making requests

---

### Error 5: Filtering not working - all results returned

**Problem**: Filtering doesn't work, returns all results instead of filtered results.

**Why This Happens**:
- Filter backend not configured
- FilterSet not assigned to view
- Field name mismatch

**How to Fix**:
```python
# WRONG - Missing filter backend
class TaskViewSet(viewsets.ModelViewSet):
    filterset_fields = ['completed']  # ❌ No filter_backends!

# CORRECT - Add filter backend
class TaskViewSet(viewsets.ModelViewSet):
    filter_backends = [DjangoFilterBackend]  # ✅ Enable filtering
    filterset_fields = ['completed']

# WRONG - FilterSet not assigned
class TaskFilter(django_filters.FilterSet):
    # ... filter definition ...

class TaskViewSet(viewsets.ModelViewSet):
    # filterset_class not set!  # ❌

# CORRECT - Assign FilterSet
class TaskViewSet(viewsets.ModelViewSet):
    filter_backends = [DjangoFilterBackend]
    filterset_class = TaskFilter  # ✅ Use FilterSet
```

**Prevention**: 
- Always include filter backend in `filter_backends`
- Assign `filterset_class` when using FilterSet
- Verify filter configuration in view

---

### General Debugging Tips for Level 2B

**1. Test Filters Separately**
```bash
# Test each filter individually
curl "http://localhost:8000/api/tasks/?completed=false"
curl "http://localhost:8000/api/tasks/?created_after=2024-01-01"
curl "http://localhost:8000/api/tasks/?title=django"
```

**2. Check Filter Configuration**
```python
# In view, verify filter setup
print("Filter backends:", self.filter_backends)
print("FilterSet class:", self.filterset_class)
print("Search fields:", self.search_fields)
print("Ordering fields:", self.ordering_fields)
```

**3. Verify Pagination**
```python
# Check pagination settings
from rest_framework.pagination import PageNumberPagination
pagination = PageNumberPagination()
print("Page size:", pagination.page_size)
```

**4. Test Exception Handler**
```bash
# Test different error scenarios
# Invalid data
curl -X POST http://localhost:8000/api/tasks/ \
  -H "Content-Type: application/json" \
  -d '{"invalid": "data"}'

# Non-existent resource
curl http://localhost:8000/api/tasks/99999/
```

## Add-ons

### Add-on 1: Custom Filter Backend

Create a custom filter backend for advanced filtering logic:

```python
# api/filters.py
from rest_framework.filters import BaseFilterBackend

class CustomFilterBackend(BaseFilterBackend):
    """
    Custom filter backend for complex filtering logic.
    """
    def filter_queryset(self, request, queryset, view):
        # Add custom filtering logic here
        if request.query_params.get('my_custom_filter'):
            queryset = queryset.filter(...)
        return queryset
```

### Add-on 2: Custom Pagination Class

Create a custom pagination class with additional metadata:

```python
# api/pagination.py
from rest_framework.pagination import PageNumberPagination

class CustomPageNumberPagination(PageNumberPagination):
    page_size = 20
    page_size_query_param = 'page_size'
    max_page_size = 100
    
    def get_paginated_response(self, data):
        response = super().get_paginated_response(data)
        # Add custom metadata
        response.data['custom_field'] = 'custom_value'
        return response
```

## Next Steps

Congratulations! You've completed Level 2B. You now know:

- ✅ Filtering with django-filter
- ✅ Searching across multiple fields
- ✅ Ordering and sorting results
- ✅ Pagination strategies
- ✅ Exception handling
- ✅ Testing authenticated APIs

**Ready for Level 3?** Continue to [Level 3A: Advanced - Relationships & Query Optimization](LEVEL_3A_ADVANCED.md) to learn about model relationships, nested serializers, and query optimization!

---

**Resources:**
- [DRF Filtering](https://www.django-rest-framework.org/api-guide/filtering/)
- [DRF Searching](https://www.django-rest-framework.org/api-guide/filtering/#searchfilter)
- [DRF Pagination](https://www.django-rest-framework.org/api-guide/pagination/)
- [django-filter](https://django-filter.readthedocs.io/)

