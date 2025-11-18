# cURL Command Guide for API Testing

## Overview

cURL (Client URL) is a command-line tool for making HTTP requests. It's the most reliable way to test REST APIs and is available on all operating systems. This guide will teach you how to use cURL to test your Django REST Framework APIs.

## Table of Contents

1. [What is cURL?](#what-is-curl)
2. [Installation](#installation)
3. [Basic Requests](#basic-requests)
4. [Headers and Content Types](#headers-and-content-types)
5. [Authentication](#authentication)
6. [File Uploads](#file-uploads)
7. [Advanced Usage](#advanced-usage)
8. [Examples by DRF Level](#examples-by-drf-level)
9. [Troubleshooting](#troubleshooting)

## What is cURL?

cURL is a command-line tool that allows you to transfer data to or from a server using various protocols (HTTP, HTTPS, FTP, etc.). For API testing, we primarily use it for HTTP/HTTPS requests.

**Why use cURL?**
- Available on all platforms
- No GUI needed - works in terminal
- Scriptable and automatable
- Shows raw HTTP requests/responses
- Industry standard for API testing

## Installation

### Linux

Most Linux distributions come with cURL pre-installed. If not:

```bash
# Ubuntu/Debian
sudo apt install curl

# CentOS/RHEL
sudo yum install curl

# Verify installation
curl --version
```

### macOS

cURL comes pre-installed. Verify:

```bash
curl --version
```

### Windows

**Option 1: Windows 10/11 (Built-in)**
- cURL is included in Windows 10 version 1803 and later
- Open PowerShell or Command Prompt and type: `curl --version`

**Option 2: Download**
- Download from [curl.se/windows/](https://curl.se/windows/)
- Or use Git Bash (includes cURL)

**Option 3: Using Chocolatey**
```bash
choco install curl
```

## Basic Requests

### GET Request

The simplest request - fetching data:

```bash
# Basic GET request
curl http://localhost:8000/api/tasks/

# With verbose output (see headers)
curl -v http://localhost:8000/api/tasks/

# Pretty print JSON response (requires jq)
curl http://localhost:8000/api/tasks/ | jq

# Save response to file
curl http://localhost:8000/api/tasks/ -o response.json

# Follow redirects
curl -L http://localhost:8000/api/tasks/
```

### POST Request

Creating new resources:

```bash
# Basic POST with JSON data
curl -X POST http://localhost:8000/api/tasks/ \
  -H "Content-Type: application/json" \
  -d '{"title": "Learn DRF", "desc": "Complete the learning guide", "completed": false}'

# POST with data from file
curl -X POST http://localhost:8000/api/tasks/ \
  -H "Content-Type: application/json" \
  -d @data.json

# POST with form data
curl -X POST http://localhost:8000/api/tasks/ \
  -d "title=Learn DRF" \
  -d "desc=Complete the learning guide" \
  -d "completed=false"
```

### PUT Request

Full update of a resource:

```bash
curl -X PUT http://localhost:8000/api/tasks/1/ \
  -H "Content-Type: application/json" \
  -d '{"title": "Updated Task", "desc": "Updated description", "completed": true}'
```

### PATCH Request

Partial update of a resource:

```bash
curl -X PATCH http://localhost:8000/api/tasks/1/ \
  -H "Content-Type: application/json" \
  -d '{"completed": true}'
```

### DELETE Request

Deleting a resource:

```bash
curl -X DELETE http://localhost:8000/api/tasks/1/
```

## Headers and Content Types

### Setting Headers

```bash
# Single header
curl -H "Content-Type: application/json" http://localhost:8000/api/tasks/

# Multiple headers
curl -H "Content-Type: application/json" \
     -H "Accept: application/json" \
     http://localhost:8000/api/tasks/

# Custom header
curl -H "X-Custom-Header: my-value" http://localhost:8000/api/tasks/
```

### Common Headers

```bash
# JSON content
curl -H "Content-Type: application/json" ...

# XML content
curl -H "Content-Type: application/xml" ...

# Accept only JSON response
curl -H "Accept: application/json" ...

# User agent
curl -H "User-Agent: MyApp/1.0" ...
```

## Authentication

### Basic Authentication

```bash
# Basic auth (username:password)
curl -u username:password http://localhost:8000/api/tasks/

# Or with header
curl -H "Authorization: Basic $(echo -n 'username:password' | base64)" \
  http://localhost:8000/api/tasks/
```

### Bearer Token (JWT)

```bash
# With Bearer token
curl -H "Authorization: Bearer your_jwt_token_here" \
  http://localhost:8000/api/tasks/

# Store token in variable (bash)
TOKEN="your_jwt_token_here"
curl -H "Authorization: Bearer $TOKEN" \
  http://localhost:8000/api/tasks/
```

### Getting JWT Token

```bash
# Login to get token
curl -X POST http://localhost:8000/api/token/ \
  -H "Content-Type: application/json" \
  -d '{"username": "admin", "password": "password123"}' \
  | jq -r '.access'

# Save token to variable
TOKEN=$(curl -s -X POST http://localhost:8000/api/token/ \
  -H "Content-Type: application/json" \
  -d '{"username": "admin", "password": "password123"}' \
  | jq -r '.access')

# Use token in subsequent requests
curl -H "Authorization: Bearer $TOKEN" \
  http://localhost:8000/api/tasks/
```

## File Uploads

### Upload Single File

```bash
# Upload file
curl -X POST http://localhost:8000/api/tasks/1/upload/ \
  -H "Authorization: Bearer $TOKEN" \
  -F "file=@/path/to/file.pdf"

# Upload with additional data
curl -X POST http://localhost:8000/api/tasks/1/upload/ \
  -H "Authorization: Bearer $TOKEN" \
  -F "file=@/path/to/file.pdf" \
  -F "description=Task attachment"
```

### Upload Multiple Files

```bash
curl -X POST http://localhost:8000/api/tasks/1/upload/ \
  -H "Authorization: Bearer $TOKEN" \
  -F "files=@/path/to/file1.pdf" \
  -F "files=@/path/to/file2.jpg"
```

## Advanced Usage

### Verbose Mode

See full request/response details:

```bash
curl -v http://localhost:8000/api/tasks/

# Shows:
# - Request headers
# - Response headers
# - SSL handshake (if HTTPS)
# - Connection details
```

### Silent Mode

Suppress progress meter:

```bash
curl -s http://localhost:8000/api/tasks/
```

### Show Only Response Headers

```bash
curl -I http://localhost:8000/api/tasks/

# Or
curl -D - http://localhost:8000/api/tasks/
```

### Follow Redirects

```bash
curl -L http://localhost:8000/api/tasks/
```

### Timeout

```bash
# Set timeout to 10 seconds
curl --max-time 10 http://localhost:8000/api/tasks/

# Connection timeout
curl --connect-timeout 5 http://localhost:8000/api/tasks/
```

### Save Response Headers

```bash
# Save headers to file
curl -D headers.txt http://localhost:8000/api/tasks/

# Save both headers and body
curl -D headers.txt -o response.json http://localhost:8000/api/tasks/
```

### Pretty Print JSON

```bash
# Using jq (install: sudo apt install jq)
curl http://localhost:8000/api/tasks/ | jq

# Using python
curl http://localhost:8000/api/tasks/ | python -m json.tool
```

## Examples by DRF Level

### Level 1: Basic CRUD Operations

```bash
# GET - List all tasks
curl http://localhost:8000/api/tasks/

# GET - Retrieve single task
curl http://localhost:8000/api/tasks/1/

# POST - Create new task
curl -X POST http://localhost:8000/api/tasks/ \
  -H "Content-Type: application/json" \
  -d '{
    "title": "Learn DRF",
    "desc": "Complete Level 1",
    "completed": false
  }'

# PUT - Update task
curl -X PUT http://localhost:8000/api/tasks/1/ \
  -H "Content-Type: application/json" \
  -d '{
    "title": "Learn DRF - Updated",
    "desc": "Completed Level 1",
    "completed": true
  }'

# PATCH - Partial update
curl -X PATCH http://localhost:8000/api/tasks/1/ \
  -H "Content-Type: application/json" \
  -d '{"completed": true}'

# DELETE - Delete task
curl -X DELETE http://localhost:8000/api/tasks/1/
```

### Level 2: Authenticated Requests

```bash
# Step 1: Get JWT token
TOKEN=$(curl -s -X POST http://localhost:8000/api/token/ \
  -H "Content-Type: application/json" \
  -d '{"username": "admin", "password": "password123"}' \
  | python -c "import sys, json; print(json.load(sys.stdin)['access'])")

# Step 2: Use token in requests
curl -H "Authorization: Bearer $TOKEN" \
  http://localhost:8000/api/tasks/

# Create task with authentication
curl -X POST http://localhost:8000/api/tasks/ \
  -H "Content-Type: application/json" \
  -H "Authorization: Bearer $TOKEN" \
  -d '{
    "title": "Private Task",
    "desc": "Only visible to authenticated users",
    "completed": false
  }'

# Refresh token
REFRESH_TOKEN=$(curl -s -X POST http://localhost:8000/api/token/refresh/ \
  -H "Content-Type: application/json" \
  -d "{\"refresh\": \"$REFRESH_TOKEN\"}" \
  | python -c "import sys, json; print(json.load(sys.stdin)['access'])")
```

### Level 3: Complex Nested Data

```bash
# POST with nested relationships
curl -X POST http://localhost:8000/api/posts/ \
  -H "Content-Type: application/json" \
  -H "Authorization: Bearer $TOKEN" \
  -d '{
    "title": "My First Post",
    "content": "This is the content",
    "author": 1,
    "tags": [1, 2, 3],
    "comments": [
      {
        "content": "Great post!",
        "author": 2
      }
    ]
  }'

# GET with nested data
curl -H "Authorization: Bearer $TOKEN" \
  http://localhost:8000/api/posts/1/
```

### Level 4: File Uploads and Versioned APIs

```bash
# File upload
curl -X POST http://localhost:8000/api/v1/tasks/1/attachments/ \
  -H "Authorization: Bearer $TOKEN" \
  -F "file=@/path/to/document.pdf" \
  -F "description=Task attachment"

# Versioned API (v1)
curl http://localhost:8000/api/v1/tasks/

# Versioned API (v2)
curl http://localhost:8000/api/v2/tasks/

# With version header
curl -H "Accept: application/json; version=2" \
  http://localhost:8000/api/tasks/
```

### Level 5: GraphQL Queries

```bash
# GraphQL query
curl -X POST http://localhost:8000/graphql/ \
  -H "Content-Type: application/json" \
  -H "Authorization: Bearer $TOKEN" \
  -d '{
    "query": "query { tasks { id title completed } }"
  }'

# GraphQL mutation
curl -X POST http://localhost:8000/graphql/ \
  -H "Content-Type: application/json" \
  -H "Authorization: Bearer $TOKEN" \
  -d '{
    "query": "mutation { createTask(title: \"New Task\") { id title } }"
  }'
```

## Common cURL Flags Reference

| Flag | Description |
|------|-------------|
| `-X METHOD` | HTTP method (GET, POST, PUT, DELETE, etc.) |
| `-H "Header: value"` | Add header |
| `-d "data"` | Send data in request body |
| `-F "name=value"` | Form data (for file uploads) |
| `-u user:pass` | Basic authentication |
| `-v` | Verbose output |
| `-s` | Silent mode (no progress) |
| `-i` | Include response headers |
| `-I` | Head request (headers only) |
| `-L` | Follow redirects |
| `-o file` | Save output to file |
| `-O` | Save with remote filename |
| `-D file` | Save headers to file |
| `--max-time SEC` | Maximum time for request |
| `--connect-timeout SEC` | Connection timeout |
| `-k` | Allow insecure SSL connections |
| `-c file` | Save cookies to file |
| `-b file` | Load cookies from file |

## Converting Postman to cURL

### In Postman

1. Open your request in Postman
2. Click "Code" button (bottom left)
3. Select "cURL" from dropdown
4. Copy the generated cURL command

### Manual Conversion Tips

- **Headers**: Each header becomes `-H "Header: value"`
- **Body**: JSON becomes `-d '{"key": "value"}'`
- **Auth**: Bearer token becomes `-H "Authorization: Bearer token"`
- **Files**: Use `-F "file=@path/to/file"`

## Troubleshooting

### Common Issues

#### 1. Connection Refused

```bash
# Check if server is running
curl http://localhost:8000/api/tasks/

# Error: Connection refused
# Solution: Start Django server
python manage.py runserver
```

#### 2. 404 Not Found

```bash
# Check URL is correct
curl -v http://localhost:8000/api/tasks/

# Verify URL patterns in Django
python manage.py show_urls  # If installed
```

#### 3. 401 Unauthorized

```bash
# Missing or invalid token
# Solution: Get new token
TOKEN=$(curl -s -X POST http://localhost:8000/api/token/ \
  -H "Content-Type: application/json" \
  -d '{"username": "user", "password": "pass"}' \
  | jq -r '.access')
```

#### 4. 400 Bad Request

```bash
# Check JSON syntax
curl -X POST http://localhost:8000/api/tasks/ \
  -H "Content-Type: application/json" \
  -d '{"title": "Test"}'  # Valid JSON

# Use verbose mode to see error details
curl -v -X POST http://localhost:8000/api/tasks/ \
  -H "Content-Type: application/json" \
  -d '{"title": "Test"}'
```

#### 5. SSL Certificate Errors

```bash
# For development (not recommended for production)
curl -k https://api.example.com/

# Or specify certificate
curl --cacert /path/to/cert.pem https://api.example.com/
```

### Debugging Tips

1. **Use verbose mode**: `curl -v` shows full request/response
2. **Check response headers**: `curl -I` shows only headers
3. **Save responses**: `curl -o response.json` to inspect later
4. **Test with simple request first**: Start with GET before POST
5. **Validate JSON**: Use `jq` or `python -m json.tool`

## Practice Exercises

### Exercise 1: Basic CRUD

```bash
# 1. List all tasks
curl http://localhost:8000/api/tasks/

# 2. Create a task
curl -X POST http://localhost:8000/api/tasks/ \
  -H "Content-Type: application/json" \
  -d '{"title": "Exercise Task", "completed": false}'

# 3. Get the created task (use ID from step 2)
curl http://localhost:8000/api/tasks/1/

# 4. Update the task
curl -X PATCH http://localhost:8000/api/tasks/1/ \
  -H "Content-Type: application/json" \
  -d '{"completed": true}'

# 5. Delete the task
curl -X DELETE http://localhost:8000/api/tasks/1/
```

### Exercise 2: Authentication Flow

```bash
# 1. Register a user
curl -X POST http://localhost:8000/api/register/ \
  -H "Content-Type: application/json" \
  -d '{"username": "testuser", "password": "testpass123", "email": "test@example.com"}'

# 2. Get JWT token
TOKEN=$(curl -s -X POST http://localhost:8000/api/token/ \
  -H "Content-Type: application/json" \
  -d '{"username": "testuser", "password": "testpass123"}' \
  | jq -r '.access')

# 3. Use token to access protected endpoint
curl -H "Authorization: Bearer $TOKEN" \
  http://localhost:8000/api/tasks/
```

## Next Steps

Now that you know cURL basics:

1. Practice with your Level 1 API
2. Use cURL to test all endpoints
3. Create a script with common requests
4. Refer back to this guide as you progress through levels

For more information:
- [cURL Official Documentation](https://curl.se/docs/)
- [HTTPie](https://httpie.io/) - Alternative to cURL with better UX
- [Postman](https://www.postman.com/) - GUI alternative

---

**Ready to test your APIs?** Use these cURL commands as you work through each level of the [Learning Guide](LEARNING_GUIDE.md)!

