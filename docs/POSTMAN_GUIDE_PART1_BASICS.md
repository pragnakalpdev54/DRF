# Postman Beginner Guide - Part 1: Basics

## Goal

Learn how to use Postman to test, debug, and interact with REST APIs. This first part covers installation, interface navigation, basic requests, collections, and environments. By the end of Part 1, you'll be able to efficiently test your Django REST Framework APIs and organize your requests.

## Table of Contents

1. [What is Postman?](#what-is-postman)
2. [Installation and Setup](#installation-and-setup)
3. [Understanding the Postman Interface](#understanding-the-postman-interface)
4. [Sending Your First Request](#sending-your-first-request)
5. [HTTP Methods Explained](#http-methods-explained)
6. [Working with Request Headers](#working-with-request-headers)
7. [Request Body and Data Types](#request-body-and-data-types)
8. [Organizing Requests with Collections](#organizing-requests-with-collections)
9. [Using Environments and Variables](#using-environments-and-variables)
10. [Next Steps](#next-steps)

## What is Postman?

Postman is a powerful API development and testing tool that provides a user-friendly interface for making HTTP requests, testing APIs, and organizing your API workflows.

**Why use Postman?**

- **Visual Interface**: Easy-to-use GUI instead of command-line tools
- **Request Organization**: Group related requests into collections
- **Environment Management**: Switch between dev, staging, and production easily
- **Automated Testing**: Write tests to validate API responses
- **Team Collaboration**: Share collections and work together
- **Documentation**: Auto-generate API documentation
- **Mock Servers**: Test frontend without backend ready

**Postman vs cURL**

| Feature | Postman | cURL |
|---------|---------|------|
| Interface | Graphical (GUI) | Command-line |
| Learning Curve | Easier for beginners | Requires terminal knowledge |
| Organization | Collections and folders | Scripts and files |
| Testing | Built-in test scripts | Manual validation |
| Collaboration | Built-in sharing | Manual file sharing |
| Automation | Collection Runner, Newman | Shell scripts |

**When to use Postman:**
- Testing APIs during development
- Exploring and learning new APIs
- Sharing API requests with team
- Automated API testing
- API documentation

**When to use cURL:**
- CI/CD pipelines
- Quick terminal testing
- Scripting and automation
- Server environments without GUI

Both tools are valuable - use Postman for development and exploration, cURL for automation and scripts.

## Installation and Setup

### Windows Installation

**Method 1: Direct Download (Recommended)**

1. Visit the [Postman Download Page](https://www.postman.com/downloads/)
2. Click the **"Download for Windows"** button
3. Run the downloaded `.exe` installer
4. Follow the installation wizard
5. Launch Postman from the Start menu

**Method 2: Using Chocolatey**

If you have Chocolatey package manager installed:

```bash
choco install postman
```

**Method 3: Using Winget**

```bash
winget install Postman.Postman
```

**Verification:**

After installation, open Postman. You should see the welcome screen.

### Linux Installation

**Method 1: Using Snap (Recommended for Ubuntu/Debian)**

```bash
# Install snapd if not already installed
sudo apt update
sudo apt install snapd

# Install Postman
sudo snap install postman
```

**Method 2: Using Tarball (All Linux Distributions)**

1. Download the Linux tarball from [Postman Download Page](https://www.postman.com/downloads/)
2. Extract the archive:

```bash
# Extract to a directory
tar -xzf Postman-linux-x64-*.tar.gz

# Move to /opt for system-wide access
sudo mv Postman /opt/

# Create a symbolic link for easy access
sudo ln -s /opt/Postman/Postman /usr/local/bin/postman
```

3. Create a desktop entry (optional):

```bash
# Create desktop entry file
cat > ~/.local/share/applications/postman.desktop << EOF
[Desktop Entry]
Name=Postman
Exec=/opt/Postman/Postman
Icon=/opt/Postman/app/resources/app/assets/icon.png
Terminal=false
Type=Application
Categories=Development;
EOF
```

**Method 3: Using AppImage**

1. Download the AppImage from the Postman website
2. Make it executable:

```bash
chmod +x Postman-*.AppImage
./Postman-*.AppImage
```

**Verification:**

Launch Postman from terminal:

```bash
postman
```

Or from applications menu if desktop entry was created.

### Creating a Postman Account

**Why create an account?**

- **Sync Across Devices**: Access your collections from any computer
- **Team Collaboration**: Share collections with teammates
- **Cloud Backup**: Your work is saved automatically
- **Advanced Features**: Access to team workspaces and integrations

**Steps to Create Account:**

1. Open Postman
2. Click **"Sign Up"** or **"Create Account"**
3. Enter your email address
4. Choose a password
5. Verify your email (check inbox)
6. Sign in to Postman

**Note**: You can use Postman without an account, but you'll miss out on syncing and collaboration features.

### Initial Setup and Preferences

**Accessing Settings:**

- **Windows/Linux**: Click the gear icon (⚙️) in the top-right corner
- Or go to: **File → Settings** (or **Postman → Preferences** on Linux)

**Important Settings to Configure:**

1. **Theme**: Choose Light or Dark theme (Settings → Theme)
2. **Editor Font Size**: Adjust for readability (Settings → Editor)
3. **Request Timeout**: Set default timeout (Settings → General)
4. **SSL Certificate Verification**: Keep enabled for security (Settings → General)
5. **Send Cookies**: Enable to maintain session (Settings → General)

**Keyboard Shortcuts:**

- **Ctrl+N** (Windows/Linux): New request
- **Ctrl+S**: Save request
- **Ctrl+Enter**: Send request
- **Ctrl+/**: Toggle comment
- **Ctrl+B**: Toggle sidebar

## Understanding the Postman Interface

### Main Components

```
┌─────────────────────────────────────────────────────────────┐
│  Postman                                 [⚙️] [☁️] [👤]     │  ← Top Bar
├──────────┬──────────────────────────────────────────────────┤
│          │  GET  [URL Bar]                    [Send]       │  ← Request Builder
│ Sidebar  │  ┌──────────────────────────────────────────┐   │
│          │  │ Params │ Authorization │ Headers │ Body   │   │
│          │  └──────────────────────────────────────────┘   │
│          │                                                  │
│          │  ┌──────────────────────────────────────────┐   │
│          │  │ Response                                │   │  ← Response Viewer
│          │  │ Status: 200 OK                          │   │
│          │  │ Body │ Headers │ Cookies │ Test Results │   │
│          │  └──────────────────────────────────────────┘   │
└──────────┴──────────────────────────────────────────────────┘
```

### Sidebar Components

**1. Collections**
- Organize related requests into folders
- Share collections with team
- Run multiple requests in sequence

**2. APIs**
- Define API schemas
- Generate documentation
- Create mock servers

**3. Environments**
- Store variables for different environments
- Switch between dev/staging/production
- Manage configuration values

**4. History**
- View all requests you've sent
- Re-run previous requests
- Save requests to collections

### Request Builder

**1. HTTP Method Dropdown**
- Select GET, POST, PUT, PATCH, DELETE, etc.
- Custom methods also supported

**2. URL Bar**
- Enter the API endpoint URL
- Supports variables: `{{base_url}}/api/tasks/`
- Auto-complete for saved URLs

**3. Tabs**
- **Params**: Query parameters (e.g., `?page=1&limit=10`)
- **Authorization**: Authentication settings
- **Headers**: HTTP headers
- **Body**: Request body (for POST, PUT, PATCH)
- **Pre-request Script**: JavaScript code before request
- **Tests**: JavaScript code to test response

**4. Send Button**
- Executes the request
- Shows response below

### Response Viewer

**1. Status Code**
- Shows HTTP status (200, 404, 500, etc.)
- Color-coded (green = success, red = error)

**2. Response Time**
- How long the request took
- Useful for performance testing

**3. Response Size**
- Size of the response body
- Helps identify large responses

**4. Tabs**
- **Body**: Response content (JSON, XML, HTML, etc.)
- **Headers**: Response headers
- **Cookies**: Set cookies
- **Test Results**: Results of test scripts

**5. Body View Options**
- **Pretty**: Formatted JSON/XML
- **Raw**: Unformatted text
- **Preview**: Rendered HTML
- **Visualize**: Custom visualization

### Console

**Access**: Click **"Console"** button at bottom or press **Ctrl+Alt+C**

**What it shows:**
- All HTTP requests and responses
- Console.log() output from scripts
- Errors and warnings
- Network request details

**Why it's useful:**
- Debug request issues
- See exact headers sent
- View script execution logs
- Troubleshoot problems

## Sending Your First Request

### Step-by-Step: Your First GET Request

Let's test a public API to get started:

**1. Create a New Request**

- Click **"New"** button (top-left)
- Select **"HTTP Request"**
- Or press **Ctrl+N**

**2. Set the Request**

- **Method**: Select **GET** (default)
- **URL**: Enter `https://jsonplaceholder.typicode.com/posts/1`
- Click **"Send"**

**3. View the Response**

You should see:
- **Status**: `200 OK`
- **Body**: JSON data with post information
- **Time**: Response time in milliseconds

**Congratulations!** You've sent your first API request.

### Testing Your Local Django API

**Prerequisites:**
- Django server running on `http://localhost:8000`
- An API endpoint (e.g., `/api/tasks/`)

**Steps:**

1. **Create New Request**
   - Click **"New"** → **"HTTP Request"**

2. **Configure Request**
   - **Method**: GET
   - **URL**: `http://localhost:8000/api/tasks/`
   - Click **"Send"**

3. **View Response**
   - If server is running: You'll see your API data
   - If server is not running: Connection error

**Common First Request Issues:**

| Issue | Solution |
|-------|----------|
| "Could not get response" | Check if Django server is running |
| "Connection refused" | Verify URL and port (usually 8000) |
| "404 Not Found" | Check URL path matches your Django URLs |
| "CORS error" | Add CORS headers in Django settings |

## HTTP Methods Explained

### GET - Retrieve Data

**Purpose**: Fetch data from the server

**When to use:**
- List all items: `GET /api/tasks/`
- Get single item: `GET /api/tasks/1/`
- Search/filter: `GET /api/tasks/?completed=true`

**Example in Postman:**

```
Method: GET
URL: http://localhost:8000/api/tasks/
```

**No body required** - data comes in URL or query parameters.

### POST - Create New Resource

**Purpose**: Create a new item on the server

**When to use:**
- Create new task: `POST /api/tasks/`
- Register user: `POST /api/register/`
- Submit form data

**Example in Postman:**

```
Method: POST
URL: http://localhost:8000/api/tasks/
Body (raw JSON):
{
  "title": "Learn Postman",
  "description": "Complete the Postman guide",
  "completed": false
}
```

**Important**: Set **Content-Type** header to `application/json`

### PUT - Full Update

**Purpose**: Replace entire resource with new data

**When to use:**
- Update all fields of an item
- Replace existing resource completely

**Example in Postman:**

```
Method: PUT
URL: http://localhost:8000/api/tasks/1/
Body (raw JSON):
{
  "title": "Updated Task",
  "description": "Updated description",
  "completed": true
}
```

**Note**: PUT requires ALL fields, even if unchanged.

### PATCH - Partial Update

**Purpose**: Update only specific fields

**When to use:**
- Update single field: `PATCH /api/tasks/1/` with `{"completed": true}`
- Update multiple fields without sending all data

**Example in Postman:**

```
Method: PATCH
URL: http://localhost:8000/api/tasks/1/
Body (raw JSON):
{
  "completed": true
}
```

**Note**: PATCH only updates provided fields.

### DELETE - Remove Resource

**Purpose**: Delete an item from the server

**When to use:**
- Delete task: `DELETE /api/tasks/1/`
- Remove user account
- Delete any resource

**Example in Postman:**

```
Method: DELETE
URL: http://localhost:8000/api/tasks/1/
```

**No body required** - item ID is in URL.

### Summary Table

| Method | Purpose | Body Required? | Idempotent? |
|--------|---------|----------------|-------------|
| GET | Read data | No | Yes |
| POST | Create new | Yes | No |
| PUT | Full update | Yes | Yes |
| PATCH | Partial update | Yes | No |
| DELETE | Delete | No | Yes |

**Idempotent** = Can be called multiple times with same result.

## Working with Request Headers

### What are Headers?

Headers are metadata sent with HTTP requests that provide additional information about the request or response.

**Common Headers:**

- **Content-Type**: Type of data being sent (JSON, form-data, etc.)
- **Authorization**: Authentication credentials
- **Accept**: What response format you want
- **User-Agent**: Information about the client

### Adding Headers in Postman

**Method 1: Headers Tab**

1. Click **"Headers"** tab in request builder
2. Click **"Add Header"** or use key-value fields
3. Enter header name (e.g., `Content-Type`)
4. Enter header value (e.g., `application/json`)
5. Postman auto-completes common headers

**Method 2: Bulk Edit**

1. Click **"Bulk Edit"** in Headers tab
2. Paste headers in format:
```
Content-Type: application/json
Accept: application/json
Authorization: Bearer token123
```

### Common Headers for DRF APIs

**Content-Type**

```
Content-Type: application/json
```

**Why**: Tells server you're sending JSON data. Required for POST/PUT/PATCH with JSON body.

**Accept**

```
Accept: application/json
```

**Why**: Tells server you want JSON response. DRF returns JSON by default, but good practice to specify.

**Authorization (Bearer Token)**

```
Authorization: Bearer eyJ0eXAiOiJKV1QiLCJhbGc...
```

**Why**: Authenticates requests. Used with JWT or token authentication.

**Authorization (Basic Auth)**

```
Authorization: Basic dXNlcm5hbWU6cGFzc3dvcmQ=
```

**Why**: Basic username/password authentication (base64 encoded).

### Preset Headers

Postman provides preset headers for common scenarios:

1. Click **"Presets"** dropdown in Headers tab
2. Select from:
   - **JSON**: Sets `Content-Type: application/json`
   - **XML**: Sets `Content-Type: application/xml`
   - **Form URL Encoded**: Sets appropriate content type
   - **Multipart Form**: For file uploads

### Managing Headers

**Disable Header Temporarily:**
- Uncheck the checkbox next to header
- Header stays but won't be sent

**Delete Header:**
- Hover over header row
- Click **"X"** icon

**Duplicate Header:**
- Right-click header
- Select **"Duplicate"**

## Request Body and Data Types

### When to Use Request Body

Request body is used with:
- **POST**: Creating new resources
- **PUT**: Full updates
- **PATCH**: Partial updates

**GET and DELETE** don't use request body - data goes in URL or headers.

### Body Types in Postman

Postman supports multiple body formats:

**1. none**
- No body (for GET, DELETE requests)
- Default for GET requests

**2. form-data**
- HTML form submission
- Key-value pairs
- Supports file uploads

**3. x-www-form-urlencoded**
- URL-encoded form data
- Similar to form-data but encoded differently
- No file upload support

**4. raw**
- Send raw data (JSON, XML, text, etc.)
- Most common for REST APIs
- Choose format from dropdown (JSON, XML, HTML, Text)

**5. binary**
- Upload binary files (images, PDFs, etc.)
- Select file from computer

**6. GraphQL**
- For GraphQL queries and mutations
- Advanced feature

### JSON Body (Most Common)

**When to use**: Most DRF APIs use JSON

**Steps:**

1. Select **"Body"** tab
2. Choose **"raw"**
3. Select **"JSON"** from dropdown (or "Text" and type JSON)
4. Enter JSON data:

```json
{
  "title": "My Task",
  "description": "Task description",
  "completed": false
}
```

**Example: Creating a Task**

```
Method: POST
URL: http://localhost:8000/api/tasks/
Headers:
  Content-Type: application/json
Body (raw JSON):
{
  "title": "Learn Postman",
  "description": "Complete Postman guide",
  "completed": false
}
```

### Form Data

**When to use**: HTML forms, file uploads

**Steps:**

1. Select **"Body"** tab
2. Choose **"form-data"**
3. Add key-value pairs:
   - **Key**: `title`
   - **Value**: `My Task`
4. For files, change type to **"File"** and select file

**Example:**

```
Method: POST
URL: http://localhost:8000/api/tasks/
Body (form-data):
  title: "My Task"
  description: "Task description"
  file: [Select File]
```

### URL-Encoded Form

**When to use**: Traditional form submissions

**Steps:**

1. Select **"Body"** tab
2. Choose **"x-www-form-urlencoded"**
3. Add key-value pairs

**Example:**

```
Method: POST
URL: http://localhost:8000/api/tasks/
Body (x-www-form-urlencoded):
  title: My Task
  description: Task description
```

### Binary Body

**When to use**: Uploading files directly

**Steps:**

1. Select **"Body"** tab
2. Choose **"binary"**
3. Click **"Select File"**
4. Choose file from computer

**Example:**

```
Method: POST
URL: http://localhost:8000/api/tasks/1/upload/
Body (binary):
  [Select PDF file]
```

### JSON Formatting Tips

**Valid JSON:**

```json
{
  "title": "Task",
  "completed": false,
  "tags": ["work", "urgent"]
}
```

**Invalid JSON (Common Mistakes):**

```json
// ❌ Trailing comma
{
  "title": "Task",
  "completed": false,  // ← Remove this comma
}

// ❌ Single quotes (use double quotes)
{
  'title': 'Task'  // ❌ Wrong
}

// ❌ Missing quotes around keys
{
  title: "Task"  // ❌ Wrong
}
```

**Postman Auto-Formatting:**

- Postman validates JSON as you type
- Red underline = syntax error
- Click **"Beautify"** to format JSON nicely

## Organizing Requests with Collections

### What are Collections?

Collections are groups of related API requests. Think of them as folders for organizing your API testing.

**Why use Collections?**

- **Organization**: Group related requests together
- **Sharing**: Share entire collection with team
- **Automation**: Run all requests in sequence
- **Documentation**: Auto-generate API docs
- **Version Control**: Export/import collections

### Creating a Collection

**Method 1: New Collection Button**

1. Click **"New"** button (top-left)
2. Select **"Collection"**
3. Enter collection name (e.g., "DRF Task API")
4. (Optional) Add description
5. Click **"Create"**

**Method 2: From Request**

1. Create or open a request
2. Click **"Save"** button
3. Click **"+ Create Collection"**
4. Enter name and create
5. Save request to new collection

**Method 3: Sidebar**

1. Right-click in sidebar
2. Select **"New Collection"**
3. Enter name and create

### Adding Requests to Collection

**Method 1: Save Existing Request**

1. Create or modify a request
2. Click **"Save"** button
3. Select collection from dropdown
4. (Optional) Change request name
5. Click **"Save"**

**Method 2: Drag and Drop**

1. Find request in **History**
2. Drag it to collection in sidebar
3. Drop to add

**Method 3: Create in Collection**

1. Right-click collection
2. Select **"Add Request"**
3. Configure request
4. It's automatically saved to collection

### Organizing with Folders

**Create Folder:**

1. Right-click collection
2. Select **"Add Folder"**
3. Enter folder name (e.g., "Authentication", "Tasks", "Users")

**Organize Requests:**

- Drag requests into folders
- Create nested folders for better organization

**Example Structure:**

```
DRF API Collection
├── Authentication
│   ├── Login
│   ├── Register
│   └── Refresh Token
├── Tasks
│   ├── List Tasks
│   ├── Create Task
│   ├── Get Task
│   ├── Update Task
│   └── Delete Task
└── Users
    ├── List Users
    └── Get Profile
```

### Collection Settings

**Access**: Right-click collection → **"Edit"**

**Settings:**

1. **Name**: Collection name
2. **Description**: What this collection is for
3. **Authorization**: Set auth for all requests (can override per request)
4. **Pre-request Script**: Run before each request
5. **Tests**: Run after each request
6. **Variables**: Collection-level variables

### Collection Variables

**Why**: Store values used across multiple requests in the collection.

**Example**: Base URL for all requests

**Set Collection Variable:**

1. Right-click collection → **"Edit"**
2. Go to **"Variables"** tab
3. Add variable:
   - **Variable**: `base_url`
   - **Initial Value**: `http://localhost:8000`
   - **Current Value**: `http://localhost:8000`
4. Click **"Save"**

**Use in Requests:**

In URL bar, use: `{{base_url}}/api/tasks/`

Postman replaces `{{base_url}}` with the actual value.

### Running Collections

**Collection Runner:**

1. Right-click collection
2. Select **"Run collection"**
3. Configure run settings:
   - Select requests to run
   - Set iterations (how many times)
   - Set delay between requests
4. Click **"Run DRF API Collection"**
5. View results

**Use Cases:**

- Test all endpoints after code changes
- Verify API still works
- Run automated tests
- Performance testing

### Sharing Collections

**Export Collection:**

1. Right-click collection
2. Select **"Export"**
3. Choose format (Collection v2.1 recommended)
4. Save file
5. Share file with team

**Import Collection:**

1. Click **"Import"** button (top-left)
2. Select **"File"** or **"Link"**
3. Choose collection file or paste link
4. Click **"Import"**

**Share via Link (Postman Account Required):**

1. Right-click collection
2. Select **"Share Collection"**
3. Choose visibility (Team, Public, Private)
4. Copy link and share

## Using Environments and Variables

### What are Environments?

Environments are sets of variables that you can switch between easily. Common use cases:

- **Development**: `http://localhost:8000`
- **Staging**: `https://staging-api.example.com`
- **Production**: `https://api.example.com`

**Why use Environments?**

- **Quick Switching**: Change base URL with one click
- **No Manual Editing**: Don't change URLs in every request
- **Team Consistency**: Everyone uses same environment values
- **Security**: Store sensitive data (API keys) securely

### Creating an Environment

**Steps:**

1. Click **"Environments"** in sidebar (or eye icon in top-right)
2. Click **"+"** button or **"Create Environment"**
3. Enter environment name (e.g., "Development")
4. Add variables:
   - **Variable**: `base_url`
   - **Initial Value**: `http://localhost:8000`
   - **Current Value**: `http://localhost:8000`
5. Click **"Save"**

**Example Environment Variables:**

| Variable | Initial Value | Current Value |
|----------|---------------|---------------|
| `base_url` | `http://localhost:8000` | `http://localhost:8000` |
| `api_key` | `dev_key_123` | `dev_key_123` |
| `token` | (leave empty) | (will be set dynamically) |

### Switching Environments

**Method 1: Dropdown**

1. Click environment dropdown (top-right, shows "No Environment")
2. Select environment (e.g., "Development")
3. All `{{variable}}` references update automatically

**Method 2: Quick Switch**

- Use keyboard shortcut (if configured)
- Or click environment name in top-right

### Using Variables in Requests

**Syntax**: `{{variable_name}}`

**Example:**

```
URL: {{base_url}}/api/tasks/
```

When "Development" environment is selected and `base_url = http://localhost:8000`, the actual URL becomes:
```
http://localhost:8000/api/tasks/
```

**Where Variables Work:**

- **URL**: `{{base_url}}/api/tasks/`
- **Headers**: `Authorization: Bearer {{token}}`
- **Body**: `{"user_id": {{user_id}}}`
- **Query Params**: `?page={{page_number}}`

### Variable Scopes

Variables have different scopes (priority order):

1. **Local Variables** (highest priority)
   - Set in scripts
   - Only available in current request
   - Example: `pm.variables.set("temp_id", "123")`

2. **Environment Variables**
   - Defined in environment
   - Available to all requests when environment is active
   - Example: `{{base_url}}`

3. **Collection Variables**
   - Defined in collection settings
   - Available to all requests in collection
   - Example: `{{api_version}}`

4. **Global Variables** (lowest priority)
   - Available everywhere
   - Use sparingly
   - Example: `{{global_timeout}}`

**Resolution Order**: Local → Environment → Collection → Global

### Dynamic Variables

Postman provides built-in dynamic variables:

**Random Values:**

- `{{$randomInt}}`: Random integer (0-1000)
- `{{$randomFirstName}}`: Random first name
- `{{$randomLastName}}`: Random last name
- `{{$randomEmail}}`: Random email
- `{{$randomUUID}}`: Random UUID

**Example:**

```json
{
  "name": "{{$randomFirstName}} {{$randomLastName}}",
  "email": "{{$randomEmail}}",
  "age": {{$randomInt}}
}
```

**Timestamp:**

- `{{$timestamp}}`: Current Unix timestamp
- `{{$isoTimestamp}}`: ISO 8601 timestamp

**Example:**

```json
{
  "created_at": "{{$isoTimestamp}}"
}
```

### Setting Variables from Response

**Why**: Extract data from response and use in next request.

**Example: Save JWT Token**

1. **Login Request** (POST `/api/token/`)

In **Tests** tab:

```javascript
// Parse response
const response = pm.response.json();

// Save token to environment variable
pm.environment.set("access_token", response.access);
pm.environment.set("refresh_token", response.refresh);
```

2. **Use Token in Next Request**

In **Authorization** tab:
- Type: **Bearer Token**
- Token: `{{access_token}}`

Or in **Headers**:
```
Authorization: Bearer {{access_token}}
```

**Common Use Cases:**

- Save authentication tokens
- Extract IDs from created resources
- Store pagination tokens
- Capture session data

### Environment Best Practices

**1. Use Descriptive Names**

```
✅ Good: base_url, api_key, user_id
❌ Bad: url, key, id
```

**2. Separate Environments**

- Don't mix dev and prod values
- Create separate environments for each

**3. Use Initial Values**

- Set default/example values
- Team members can override with Current Value

**4. Secure Sensitive Data**

- Don't commit environment files with real API keys
- Use Postman's built-in secret management
- Rotate keys regularly

**5. Document Variables**

- Add descriptions in environment
- Document what each variable is for
- Note any special requirements

## Next Steps

Congratulations! You've completed Part 1 of the Postman guide. You now know how to:

- ✅ Install Postman on Windows and Linux
- ✅ Navigate the Postman interface
- ✅ Send basic HTTP requests (GET, POST, PUT, PATCH, DELETE)
- ✅ Work with headers and request bodies
- ✅ Organize requests into collections
- ✅ Use environments and variables

**Continue to Part 2** to learn:
- Authentication (JWT, Token, OAuth)
- Writing tests and scripts
- Automation and collection runner
- Mock servers and documentation
- Advanced DRF API testing examples
- Troubleshooting common issues

**Practice Exercises:**

1. Create a collection for your DRF API
2. Set up Development and Production environments
3. Create requests for all CRUD operations
4. Use variables for base URL and authentication
5. Organize requests into logical folders

**Ready for more?** Continue to [Postman Guide - Part 2: Advanced Features](POSTMAN_GUIDE_PART2_ADVANCED.md)

