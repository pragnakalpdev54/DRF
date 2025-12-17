# Level 3A: Advanced - Relationships & Query Optimization

## Goal

Build efficient APIs with model relationships and optimized database queries. Master ForeignKey, ManyToMany, OneToOne relationships, and learn to optimize queries to prevent N+1 problems. By the end of this level, you'll be able to build efficient APIs that handle complex relationships without performance issues.

## Note on Examples

This guide uses **Blog API (Post/Comment/Tag)** as the primary example to demonstrate relationship patterns and query optimization. The concepts apply to any model with relationships (Task, Book, Product, etc.).

**Prerequisites**: Complete [Level 2B](LEVEL_2B_INTERMEDIATE.md) first to understand filtering, searching, and pagination.

## Table of Contents

1. [Model Relationships](#model-relationships)
2. [Query Optimization](#query-optimization)
3. [Advanced Query Patterns](#advanced-query-patterns)
4. [Exercises](#exercises)
5. [Common Errors and Solutions](#common-errors-and-solutions)
6. [Add-ons](#add-ons)

## Model Relationships

### ForeignKey (Many-to-One)

One model references another. Example: Many Posts belong to One User.

```python
# api/models.py
# Import Django models for database models
from django.db import models
# Import User model for authentication
from django.contrib.auth.models import User

# Post model - represents a blog post
# ForeignKey relationship: Many Posts belong to One User (author)
class Post(models.Model):
    # Post title
    title = models.CharField(max_length=200)
    # Post content (can be long text)
    content = models.TextField()
    
    # ForeignKey: Many-to-One relationship
    # Many posts can belong to one user (author)
    # on_delete=models.CASCADE = If user is deleted, delete all their posts
    # related_name='posts' = Access user's posts via user.posts.all()
    author = models.ForeignKey(User, on_delete=models.CASCADE, related_name='posts')
    
    # Timestamp when post was created
    created_at = models.DateTimeField(auto_now_add=True)

    def __str__(self):
        return self.title
```

**Access patterns:**

```python
# Get all posts by a user
user.posts.all()

# Get author of a post
post.author
```

### OneToOneField (One-to-One)

One-to-one relationship. Example: One User has One Profile.

```python
# api/models.py
class UserProfile(models.Model):
    user = models.OneToOneField(User, on_delete=models.CASCADE, related_name='profile')
    bio = models.TextField(blank=True)
    phone_number = models.CharField(max_length=15, blank=True)
```

**Access patterns:**

```python
# Get user's profile
user.profile

# Get profile's user
profile.user
```

### ManyToManyField (Many-to-Many)

Many-to-many relationship. Example: Posts can have many Tags, Tags can be on many Posts.

```python
# api/models.py
class Tag(models.Model):
    name = models.CharField(max_length=50, unique=True)
    created_at = models.DateTimeField(auto_now_add=True)

    def __str__(self):
        return self.name

class Post(models.Model):
    title = models.CharField(max_length=200)
    content = models.TextField()
    author = models.ForeignKey(User, on_delete=models.CASCADE, related_name='posts')
    tags = models.ManyToManyField(Tag, related_name='posts', blank=True)
    created_at = models.DateTimeField(auto_now_add=True)
```

**Access patterns:**

```python
# Add tag to post
post.tags.add(tag)

# Get all tags for a post
post.tags.all()

# Get all posts with a tag
tag.posts.all()

# Remove tag
post.tags.remove(tag)

# Clear all tags
post.tags.clear()
```

### Related Managers

Django automatically creates related managers for relationships:

```python
# ForeignKey creates reverse manager
user.posts.all()  # All posts by user
user.posts.filter(title__icontains='django')

# ManyToMany creates manager
post.tags.all()
post.tags.add(tag1, tag2)
```

### on_delete Options

```python
# CASCADE: Delete related objects
author = models.ForeignKey(User, on_delete=models.CASCADE)

# PROTECT: Prevent deletion if related objects exist
author = models.ForeignKey(User, on_delete=models.PROTECT)

# SET_NULL: Set to NULL (requires null=True)
author = models.ForeignKey(User, on_delete=models.SET_NULL, null=True)

# SET_DEFAULT: Set to default value
author = models.ForeignKey(User, on_delete=models.SET_DEFAULT, default=1)

# DO_NOTHING: Do nothing (not recommended)
author = models.ForeignKey(User, on_delete=models.DO_NOTHING)
```

## Query Optimization

### The N+1 Problem

**Problem**: Making multiple database queries when one would suffice.

```python
# BAD: N+1 queries problem
# This makes 1 query to get all posts, then 1 query per post to get author
# If you have 100 posts, this makes 101 queries! (1 + 100)
posts = Post.objects.all()  # Query 1: Get all posts
for post in posts:
    # Query 2, 3, 4, ... N: One query per post to get author
    # Each post.author access hits the database again!
    print(post.author.username)  # One query per post! ❌
```

**Solution**: Use `select_related` and `prefetch_related`.

### select_related (ForeignKey, OneToOne)

Follows foreign key relationships and fetches related objects in a single query.

```python
# GOOD: Single query using select_related
# select_related() follows ForeignKey/OneToOne relationships
# Fetches related objects in the same query using SQL JOIN
# This makes only 1 query total (instead of N+1 queries)
posts = Post.objects.select_related('author').all()  # Query 1: Get posts + authors in one query
for post in posts:
    # No additional queries! Author data already loaded
    print(post.author.username)  # No additional queries! ✅
```

### prefetch_related (ManyToMany, Reverse ForeignKey)

Prefetches related objects in a separate query.

```python
# BAD: Multiple queries for ManyToMany/Reverse ForeignKey
# This makes 1 query to get posts, then 1 query per post to get tags
# If you have 100 posts, this makes 101 queries!
posts = Post.objects.all()  # Query 1: Get all posts
for post in posts:
    # Query 2, 3, 4, ... N: One query per post to get tags
    print(post.tags.all())  # One query per post! ❌

# GOOD: Two queries total using prefetch_related
# prefetch_related() is for ManyToMany and reverse ForeignKey relationships
# Makes 2 queries: 1 for posts, 1 for all related tags (then Django matches them)
# If you have 100 posts, this makes only 2 queries!
posts = Post.objects.prefetch_related('tags').all()  # Query 1: Posts, Query 2: All tags
for post in posts:
    # No additional queries! Tags already loaded and matched
    print(post.tags.all())  # No additional queries! ✅
```

### Combined Optimization

```python
# Optimize multiple relationships at once
# Use select_related for ForeignKey/OneToOne (author)
# Use prefetch_related for ManyToMany/reverse ForeignKey (tags, comments)
# This makes only 3 queries total: 1 for posts+author, 1 for tags, 1 for comments
posts = Post.objects.select_related('author').prefetch_related('tags', 'comments').all()

# You can combine with filtering
# Filtering happens in the database (efficient)
# Optimization still works - only fetches related data for filtered posts
posts = Post.objects.select_related('author').prefetch_related('tags').filter(
    author__is_active=True  # Only get posts from active authors
).all()
```

### only() and defer()

Fetch only needed fields.

```python
# Fetch only specific fields (faster, less memory)
# only() = Fetch only these fields, defer everything else
# Useful when you don't need all fields (e.g., don't need large text fields)
posts = Post.objects.only('id', 'title', 'author_id').all()  # Only fetch these 3 fields

# Defer heavy fields (skip expensive fields)
# defer() = Don't fetch these fields, fetch everything else
# Useful for large fields (images, long text) that you don't always need
posts = Post.objects.defer('content').all()  # Don't fetch content field (saves bandwidth/memory)
```

### annotate() and aggregate()

Add computed fields and aggregations.

```python
# Import aggregation functions
from django.db.models import Count, Avg

# annotate() = Add computed field to each object in queryset
# Adds comment_count field to each post (computed in database, not Python)
# Each post will have a comment_count attribute
posts = Post.objects.annotate(comment_count=Count('comments')).all()
# Now you can do: post.comment_count (number of comments for that post)

# aggregate() = Compute a single value across entire queryset
# Returns a dictionary with the computed value
# Gets average comment count across ALL posts (single number)
avg_comments = Post.objects.aggregate(avg_comments=Avg('comments__id'))
# Returns: {'avg_comments': 5.5} (average number of comments per post)
```

### Using in Views

```python
# api/views.py
# Import necessary classes
from rest_framework import viewsets
from django.db.models import Count
from .models import Post

class PostViewSet(viewsets.ModelViewSet):
    # Override get_queryset() to optimize database queries
    # This is called for list and retrieve operations
    def get_queryset(self):
        return Post.objects.select_related('author').prefetch_related(
            'tags',  # ManyToMany: prefetch all tags
            'comments',  # Reverse ForeignKey: prefetch all comments
            'comments__author'  # Nested: prefetch author of each comment
        ).annotate(comment_count=Count('comments')).all()  # Add comment_count field
        # This makes only 4 queries total:
        # 1. Posts + authors (select_related)
        # 2. All tags (prefetch_related)
        # 3. All comments (prefetch_related)
        # 4. All comment authors (prefetch_related nested)
        # Instead of potentially hundreds of queries!
```

## Advanced Query Patterns

### Complex Filtering

```python
# Import Q object for complex queries and aggregation functions
from django.db.models import Q, Count, Avg
from django.utils import timezone

# OR conditions using Q object
# | = OR operator
# Finds tasks that are high priority OR past due date
tasks = Task.objects.filter(
    Q(priority='high') | Q(due_date__lt=timezone.now())
)

# AND conditions using Q object
# & = AND operator (can also use comma in filter())
# Finds tasks that are not completed AND high priority
tasks = Task.objects.filter(
    Q(completed=False) & Q(priority='high')
)

# Exclude certain records
# exclude() = NOT condition
# Gets all tasks except completed ones
tasks = Task.objects.exclude(completed=True)

# Annotate with multiple computed fields and filter by annotation
# Adds comment_count and tag_count to each post
# Then filters to only posts with 5+ comments
posts = Post.objects.annotate(
    comment_count=Count('comments'),  # Count comments per post
    tag_count=Count('tags')  # Count tags per post
).filter(comment_count__gte=5)  # Only posts with 5 or more comments
```

## Exercises

### Exercise 1: Optimize Blog API Queries

1. Create Post, Comment, and Tag models with relationships
2. Identify N+1 query problems in your views
3. Use select_related() to optimize ForeignKey relationships
4. Use prefetch_related() to optimize ManyToMany relationships
5. Use annotate() to add comment counts to posts
6. Measure query count before and after optimization

### Exercise 2: Complex Relationships

1. Create a Task model with ForeignKey to User (assignee)
2. Create a Category model with ManyToMany to Task
3. Create a Priority model with OneToOne to Task
4. Optimize queries to fetch all related data efficiently
5. Test with Django Debug Toolbar to verify optimization

### Exercise 3: Advanced Query Patterns

1. Use Q objects for complex OR/AND conditions
2. Use annotate() to add computed fields
3. Use aggregate() to compute statistics
4. Filter by annotated fields
5. Combine multiple optimizations in a single queryset

## Common Errors and Solutions

### Error 1: N+1 Query Problem

**Problem**: Making hundreds of database queries when a few would suffice.

**Symptoms**:
- Slow API responses
- High database load
- Many queries in Django Debug Toolbar

**How to Fix**:
```python
# BAD: N+1 queries
posts = Post.objects.all()  # 1 query
for post in posts:
    print(post.author.username)  # N queries (one per post)

# GOOD: Optimized with select_related
posts = Post.objects.select_related('author').all()  # 1 query total
for post in posts:
    print(post.author.username)  # No additional queries
```

**Prevention**: Always use `select_related()` for ForeignKey/OneToOne and `prefetch_related()` for ManyToMany/reverse ForeignKey.

---

### Error 2: `django.db.utils.OperationalError: no such column`

**Error Message**:
```
django.db.utils.OperationalError: no such column: api_post.author_id
```

**Why This Happens**:
- Model has ForeignKey but migration not run
- Database out of sync with models

**How to Fix**:
```bash
# Create migrations
python manage.py makemigrations

# Apply migrations
python manage.py migrate

# If still issues, check if model field name matches database column
```

**Prevention**: Always run migrations after adding/changing model relationships.

---

### Error 3: `RelatedObjectDoesNotExist: Post has no author`

**Error Message**:
```
RelatedObjectDoesNotExist: Post has no author
```

**Why This Happens**:
- Accessing ForeignKey that is None
- Post created without author
- Author deleted but post still exists (if on_delete not CASCADE)

**How to Fix**:
```python
# Check if related object exists
if post.author:
    print(post.author.username)
else:
    print("No author")

# Or use getattr with default
author_name = getattr(post.author, 'username', 'Unknown')
```

**Prevention**: 
- Set `null=True, blank=True` if ForeignKey can be empty
- Or ensure related object is always set when creating

---

### Error 4: `django.core.exceptions.FieldError: Cannot resolve keyword`

**Error Message**:
```
django.core.exceptions.FieldError: Cannot resolve keyword 'author__username' into field
```

**Why This Happens**:
- Typo in field name
- Using double underscore on non-relationship field
- Field doesn't exist

**How to Fix**:
```python
# WRONG - Typo
posts = Post.objects.filter(authr__username='john')  # ❌ 'authr' should be 'author'

# CORRECT
posts = Post.objects.filter(author__username='john')  # ✅

# WRONG - Can't use __ on non-relationship
posts = Post.objects.filter(title__username='test')  # ❌ title is not a relationship

# CORRECT - Use __ only for relationships
posts = Post.objects.filter(title__icontains='test')  # ✅ icontains is a lookup
```

**Prevention**: 
- Double-check field names
- Use `__` only for following relationships
- Use lookups (icontains, exact, etc.) for filtering

---

## Add-ons

### Add-on 1: Django Debug Toolbar

Install Django Debug Toolbar to visualize database queries:

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

### Add-on 2: Query Count Middleware

Create middleware to log query counts:

```python
# api/middleware.py
from django.db import connection

class QueryCountMiddleware:
    def __init__(self, get_response):
        self.get_response = get_response

    def __call__(self, request):
        response = self.get_response(request)
        print(f"Total queries: {len(connection.queries)}")
        return response
```

## Next Steps

Congratulations! You've completed Level 3A. You now know:

- ✅ Model relationships (ForeignKey, ManyToMany, OneToOne)
- ✅ Query optimization (select_related, prefetch_related)
- ✅ Advanced query patterns (Q objects, annotate, aggregate)
- ✅ Preventing N+1 query problems

**Ready for Level 3B?** Continue to [Level 3B: Advanced - Serializers & Advanced Patterns](LEVEL_3B_ADVANCED.md) to learn about nested serializers, custom serializer fields, validation, Django signals, and background tasks!

---

**Resources:**
- [Django Model Relationships](https://docs.djangoproject.com/en/stable/topics/db/models/#relationships)
- [Django Query Optimization](https://docs.djangoproject.com/en/stable/topics/db/optimization/)
- [Django Debug Toolbar](https://django-debug-toolbar.readthedocs.io/)

