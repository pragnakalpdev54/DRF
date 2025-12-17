# Postman Beginner Guide - Part 2: Advanced Features

## Goal

Master advanced Postman features including authentication, automated testing, scripting, mock servers, and comprehensive DRF API testing. By the end of Part 2, you'll be able to build robust API test suites and automate your testing workflow.

## Table of Contents

1. [Authentication in Postman](#authentication-in-postman)
2. [Writing Tests and Scripts](#writing-tests-and-scripts)
3. [Pre-request Scripts](#pre-request-scripts)
4. [Running Collections and Automation](#running-collections-and-automation)
5. [Mock Servers](#mock-servers)
6. [API Documentation](#api-documentation)
7. [Testing DRF APIs - Step by Step](#testing-drf-apis---step-by-step)
8. [Common Errors and Solutions](#common-errors-and-solutions)
9. [Best Practices](#best-practices)
10. [Exercises](#exercises)

## Authentication in Postman

### Why Authentication Matters

Most APIs require authentication to:
- **Protect data**: Only authorized users can access
- **Track usage**: Know who made which requests
- **Enforce permissions**: Control what users can do
- **Security**: Prevent unauthorized access

**Real-world analogy**: Authentication is like showing ID at a bank - you need to prove who you are before accessing your account.

### Authentication Types in Postman

Postman supports multiple authentication methods:

1. **No Auth**: Public APIs (no authentication)
2. **Bearer Token**: JWT tokens, API keys
3. **Basic Auth**: Username and password
4. **API Key**: Custom API key authentication
5. **OAuth 2.0**: Advanced authentication protocol
6. **Digest Auth**: Challenge-response authentication

### Bearer Token Authentication (JWT)

**Most common for DRF APIs** - Used with JWT (JSON Web Tokens) or simple token authentication.

**How it works:**
1. User logs in → Server returns JWT token
2. Client stores token
3. Client sends token in `Authorization` header: `Bearer <token>`
4. Server validates token and grants access

**Setting up in Postman:**

**Method 1: Authorization Tab**

1. Open request
2. Go to **"Authorization"** tab
3. **Type**: Select **"Bearer Token"**
4. **Token**: Enter your token (or use variable: `{{access_token}}`)
5. Click **"Send"**

**Method 2: Headers Tab**

1. Go to **"Headers"** tab
2. Add header:
   - **Key**: `Authorization`
   - **Value**: `Bearer your_token_here`
   - Or: `Bearer {{access_token}}`

**Example: DRF JWT Authentication**

**Step 1: Get Token (Login Request)**

```
Method: POST
URL: {{base_url}}/api/token/
Headers:
  Content-Type: application/json
Body (raw JSON):
{
  "username": "admin",
  "password": "password123"
}
```

**Response:**
```json
{
  "access": "eyJ0eXAiOiJKV1QiLCJhbGc...",
  "refresh": "eyJ0eXAiOiJKV1QiLCJhbGc..."
}
```

**Step 2: Save Token to Environment**

In **Tests** tab of login request:

```javascript
// Parse response
const response = pm.response.json();

// Save tokens to environment
pm.environment.set("access_token", response.access);
pm.environment.set("refresh_token", response.refresh);

// Optional: Set token expiration time
pm.environment.set("token_expires_at", Date.now() + 3600000); // 1 hour
```

**Step 3: Use Token in Protected Requests**

In **Authorization** tab:
- **Type**: Bearer Token
- **Token**: `{{access_token}}`

Now all requests will automatically include: `Authorization: Bearer <token>`

### Basic Authentication

**When to use**: Simple username/password authentication.

**Setting up:**

1. Go to **"Authorization"** tab
2. **Type**: Select **"Basic Auth"**
3. **Username**: Enter username
4. **Password**: Enter password
5. Postman automatically encodes credentials

**How it works:**
- Postman encodes `username:password` in base64
- Sends as: `Authorization: Basic <encoded_credentials>`

**Example:**

```
Username: admin
Password: secret123
```

Postman sends: `Authorization: Basic YWRtaW46c2VjcmV0MTIz`

**Using Variables:**

```
Username: {{username}}
Password: {{password}}
```

Store credentials in environment variables for security.

### API Key Authentication

**When to use**: Custom API key authentication (not standard Bearer Token).

**Setting up:**

1. Go to **"Authorization"** tab
2. **Type**: Select **"API Key"**
3. **Key**: Header name (e.g., `X-API-Key`)
4. **Value**: Your API key
5. **Add to**: Choose where to add (Header, Query Params)

**Example:**

```
Key: X-API-Key
Value: {{api_key}}
Add to: Header
```

This adds header: `X-API-Key: your_api_key_here`

**Common API Key Patterns:**

- Header: `X-API-Key: <key>`
- Header: `Authorization: ApiKey <key>`
- Query Param: `?api_key=<key>`

### OAuth 2.0 Authentication

**When to use**: Advanced authentication with access tokens, refresh tokens, and scopes.

**OAuth 2.0 Flow:**

1. **Authorization Request**: Get authorization code
2. **Token Request**: Exchange code for access token
3. **Access Resource**: Use access token for API calls
4. **Refresh Token**: Get new access token when expired

**Setting up in Postman:**

1. Go to **"Authorization"** tab
2. **Type**: Select **"OAuth 2.0"**
3. Configure:
   - **Grant Type**: Authorization Code, Client Credentials, etc.
   - **Auth URL**: Authorization endpoint
   - **Access Token URL**: Token endpoint
   - **Client ID**: Your OAuth client ID
   - **Client Secret**: Your OAuth client secret
   - **Scope**: Requested permissions
4. Click **"Get New Access Token"**
5. Click **"Use Token"**

**For DRF with OAuth:**

Most DRF APIs use JWT (Bearer Token) instead of full OAuth 2.0. Use Bearer Token method above.

### Digest Authentication

**When to use**: Challenge-response authentication (less common).

**Setting up:**

1. Go to **"Authorization"** tab
2. **Type**: Select **"Digest Auth"**
3. Enter **Username** and **Password**
4. Postman handles the challenge-response automatically

### Collection-Level Authentication

**Why**: Set authentication once for entire collection.

**Setting up:**

1. Right-click collection → **"Edit"**
2. Go to **"Authorization"** tab
3. Select authentication type
4. Configure credentials
5. All requests in collection inherit this auth

**Override per Request:**

- Individual requests can override collection auth
- Set different auth in request's Authorization tab

**Example Collection Auth:**

```
Type: Bearer Token
Token: {{access_token}}
```

All requests in collection automatically use this token.

### Token Refresh Flow

**Problem**: JWT tokens expire. Need to refresh automatically.

**Solution**: Pre-request script to check and refresh token.

**Step 1: Check Token Expiration**

In **Pre-request Script** tab of protected requests:

```javascript
// Get token expiration time
const tokenExpiresAt = pm.environment.get("token_expires_at");
const now = Date.now();

// If token expired or expires in next 5 minutes, refresh it
if (!tokenExpiresAt || now >= (tokenExpiresAt - 300000)) {
    // Token expired or expiring soon, refresh it
    pm.sendRequest({
        url: pm.environment.get("base_url") + "/api/token/refresh/",
        method: "POST",
        header: {
            "Content-Type": "application/json"
        },
        body: {
            mode: "raw",
            raw: JSON.stringify({
                refresh: pm.environment.get("refresh_token")
            })
        }
    }, function (err, res) {
        if (!err && res.code === 200) {
            const response = res.json();
            pm.environment.set("access_token", response.access);
            pm.environment.set("token_expires_at", Date.now() + 3600000);
        }
    });
}
```

**Step 2: Use Token**

Token is automatically refreshed before request, so it's always valid.

## Writing Tests and Scripts

### Why Write Tests?

**Benefits:**
- **Automated Validation**: Check if API works correctly
- **Catch Bugs Early**: Find issues before production
- **Documentation**: Tests show expected behavior
- **Regression Testing**: Ensure changes don't break existing features
- **CI/CD Integration**: Run tests in automated pipelines

**Real-world analogy**: Tests are like quality checks - you verify the product works before shipping.

### Test Scripts Tab

**Location**: **"Tests"** tab in request builder

**When tests run**: After response is received

**Language**: JavaScript (Postman uses a JavaScript runtime)

### Basic Test Examples

**Test 1: Check Status Code**

```javascript
pm.test("Status code is 200", function () {
    pm.response.to.have.status(200);
});
```

**Test 2: Check Response Time**

```javascript
pm.test("Response time is less than 500ms", function () {
    pm.expect(pm.response.responseTime).to.be.below(500);
});
```

**Test 3: Check Response Body Contains Data**

```javascript
pm.test("Response has tasks array", function () {
    const response = pm.response.json();
    pm.expect(response).to.have.property("results");
    pm.expect(response.results).to.be.an("array");
});
```

**Test 4: Check Specific Field Value**

```javascript
pm.test("Task has title field", function () {
    const response = pm.response.json();
    pm.expect(response).to.have.property("title");
    pm.expect(response.title).to.be.a("string");
});
```

### Common Test Assertions

**Status Code Tests:**

```javascript
pm.test("Status is 200", () => pm.response.to.have.status(200));
pm.test("Status is 201", () => pm.response.to.have.status(201));
pm.test("Status is not 404", () => pm.expect(pm.response.code).to.not.equal(404));
pm.test("Status is success", () => pm.response.to.be.success);
pm.test("Status is client error", () => pm.response.to.be.clientError);
pm.test("Status is server error", () => pm.response.to.be.serverError);
```

**Response Time Tests:**

```javascript
pm.test("Response time < 200ms", () => {
    pm.expect(pm.response.responseTime).to.be.below(200);
});
```

**Header Tests:**

```javascript
pm.test("Content-Type is JSON", () => {
    pm.expect(pm.response.headers.get("Content-Type")).to.include("application/json");
});
```

**Body Tests:**

```javascript
pm.test("Response is JSON", () => {
    pm.response.to.be.json;
});

pm.test("Response has property", () => {
    const json = pm.response.json();
    pm.expect(json).to.have.property("id");
});

pm.test("Property value matches", () => {
    const json = pm.response.json();
    pm.expect(json.completed).to.be.false;
});
```

### Advanced Test Examples

**Test: Validate JSON Schema**

```javascript
pm.test("Response matches schema", function () {
    const schema = {
        type: "object",
        properties: {
            id: { type: "number" },
            title: { type: "string" },
            completed: { type: "boolean" }
        },
        required: ["id", "title", "completed"]
    };
    
    pm.response.to.have.jsonSchema(schema);
});
```

**Test: Check Array Length**

```javascript
pm.test("Returns at least 5 tasks", function () {
    const response = pm.response.json();
    pm.expect(response.results).to.have.length.of.at.least(5);
});
```

**Test: Validate All Items in Array**

```javascript
pm.test("All tasks have required fields", function () {
    const response = pm.response.json();
    response.results.forEach(function(task) {
        pm.expect(task).to.have.property("id");
        pm.expect(task).to.have.property("title");
        pm.expect(task.id).to.be.a("number");
    });
});
```

**Test: Save Data for Next Request**

```javascript
pm.test("Save task ID for next request", function () {
    const response = pm.response.json();
    pm.environment.set("task_id", response.id);
    pm.collectionVariables.set("last_created_task_id", response.id);
});
```

**Test: Compare with Previous Response**

```javascript
pm.test("Task count increased", function () {
    const currentCount = pm.response.json().count;
    const previousCount = pm.environment.get("previous_task_count") || 0;
    pm.expect(currentCount).to.be.above(previousCount);
    pm.environment.set("previous_task_count", currentCount);
});
```

### Test Organization

**Multiple Tests in One Request:**

```javascript
// Test 1: Status
pm.test("Status is 200", () => pm.response.to.have.status(200));

// Test 2: Response time
pm.test("Fast response", () => {
    pm.expect(pm.response.responseTime).to.be.below(300);
});

// Test 3: Data validation
pm.test("Valid task data", () => {
    const task = pm.response.json();
    pm.expect(task).to.have.property("id");
    pm.expect(task).to.have.property("title");
    pm.expect(task.completed).to.be.a("boolean");
});
```

**Descriptive Test Names:**

```javascript
// ❌ Bad
pm.test("Test 1", () => {});

// ✅ Good
pm.test("GET /api/tasks/ returns 200 status", () => {
    pm.response.to.have.status(200);
});
```

### Collection-Level Tests

**Why**: Run tests after every request in collection.

**Setting up:**

1. Right-click collection → **"Edit"**
2. Go to **"Tests"** tab
3. Add tests that run after each request

**Example Collection Test:**

```javascript
// Log all requests
console.log("Request:", pm.request.url.toString());

// Check if request was successful
pm.test("Request completed", () => {
    pm.expect(pm.response.code).to.be.oneOf([200, 201, 204]);
});
```

## Pre-request Scripts

### What are Pre-request Scripts?

Scripts that run **before** the request is sent. Useful for:
- Setting dynamic headers
- Generating tokens
- Calculating values
- Preparing request data

**When they run**: Just before sending the request

**Location**: **"Pre-request Script"** tab

### Common Use Cases

**1. Set Timestamp Header**

```javascript
// Set current timestamp
pm.environment.set("timestamp", Date.now());

// Or ISO format
pm.environment.set("iso_timestamp", new Date().toISOString());
```

**2. Generate Random Data**

```javascript
// Generate random email
const randomEmail = `user${Math.floor(Math.random() * 10000)}@example.com`;
pm.environment.set("random_email", randomEmail);

// Generate random ID
const randomId = Math.floor(Math.random() * 1000);
pm.variables.set("random_id", randomId);
```

**3. Calculate Hash/Signature**

```javascript
// Example: Calculate HMAC signature
const crypto = require('crypto-js');
const message = pm.request.url.toString() + pm.request.body;
const secret = pm.environment.get("api_secret");
const signature = crypto.HmacSHA256(message, secret).toString();
pm.environment.set("signature", signature);
```

**4. Refresh Token if Expired**

```javascript
// Check if token is expired
const tokenExpiresAt = pm.environment.get("token_expires_at");
const now = Date.now();

if (!tokenExpiresAt || now >= tokenExpiresAt) {
    // Refresh token
    pm.sendRequest({
        url: pm.environment.get("base_url") + "/api/token/refresh/",
        method: "POST",
        header: { "Content-Type": "application/json" },
        body: {
            mode: "raw",
            raw: JSON.stringify({
                refresh: pm.environment.get("refresh_token")
            })
        }
    }, function (err, res) {
        if (!err && res.code === 200) {
            const response = res.json();
            pm.environment.set("access_token", response.access);
            pm.environment.set("token_expires_at", Date.now() + 3600000);
        }
    });
}
```

**5. Set Dynamic Headers**

```javascript
// Set custom header with timestamp
pm.request.headers.add({
    key: "X-Request-ID",
    value: pm.variables.replaceIn("{{$randomUUID}}")
});

// Set authorization header dynamically
const token = pm.environment.get("access_token");
if (token) {
    pm.request.headers.add({
        key: "Authorization",
        value: "Bearer " + token
    });
}
```

**6. Modify Request Body**

```javascript
// Get current body
const body = JSON.parse(pm.request.body.raw);

// Modify it
body.created_at = new Date().toISOString();
body.user_id = pm.environment.get("user_id");

// Update request body
pm.request.body.raw = JSON.stringify(body);
```

### Collection-Level Pre-request Scripts

**Why**: Run script before every request in collection.

**Setting up:**

1. Right-click collection → **"Edit"**
2. Go to **"Pre-request Script"** tab
3. Add script

**Example: Set Common Headers**

```javascript
// Set common headers for all requests
pm.request.headers.add({
    key: "X-API-Version",
    value: "v1"
});

pm.request.headers.add({
    key: "X-Client-ID",
    value: pm.environment.get("client_id")
});
```

## Running Collections and Automation

### Collection Runner

**What it does**: Runs multiple requests in sequence automatically.

**Use cases:**
- Test entire API workflow
- Regression testing
- Performance testing
- Automated test suites

**How to use:**

1. Right-click collection → **"Run collection"**
2. Configure settings:
   - **Iterations**: How many times to run (e.g., 1, 5, 10)
   - **Delay**: Milliseconds between requests
   - **Data File**: CSV/JSON with test data
   - **Select Requests**: Choose which requests to run
3. Click **"Run <Collection Name>"**
4. View results

### Collection Runner Settings

**Iterations:**
- Run collection multiple times
- Useful for load testing
- Example: 10 iterations = run all requests 10 times

**Delay:**
- Wait time between requests (milliseconds)
- Prevents overwhelming server
- Example: 1000ms = 1 second delay

**Data File:**
- CSV or JSON file with test data
- Each iteration uses different row
- Example: Test with different users

**Request Order:**
- Drag requests to reorder
- Run in specific sequence
- Example: Login → Create → Read → Update → Delete

**Stop on Error:**
- Stop if any request fails
- Useful for debugging
- Or continue and report all errors

### Viewing Results

**Summary:**
- Total requests run
- Passed/Failed count
- Average response time

**Request Details:**
- Individual request results
- Test results for each
- Response data

**Export Results:**
- Save as JSON/HTML
- Share with team
- Include in reports

### Newman - Command Line Runner

**What is Newman?** Postman's command-line collection runner.

**Why use it?**
- Run in CI/CD pipelines
- Automated testing
- No GUI required
- Integrate with scripts

**Installation:**

```bash
# Using npm
npm install -g newman

# Using yarn
yarn global add newman

# Verify installation
newman --version
```

**Basic Usage:**

```bash
# Export collection from Postman first
# Then run:
newman run collection.json
```

**With Environment:**

```bash
newman run collection.json -e environment.json
```

**Generate HTML Report:**

```bash
# Install reporter
npm install -g newman-reporter-html

# Run with HTML report
newman run collection.json -r html --reporter-html-export report.html
```

**Common Options:**

```bash
# Run with iterations
newman run collection.json -n 5

# Run with delay
newman run collection.json -d 1000

# Stop on first error
newman run collection.json --bail

# Verbose output
newman run collection.json --verbose
```

**CI/CD Integration Example (GitHub Actions):**

```yaml
name: API Tests

on: [push]

jobs:
  test:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v2
      - name: Setup Node.js
        uses: actions/setup-node@v2
        with:
          node-version: '16'
      - name: Install Newman
        run: npm install -g newman
      - name: Run API Tests
        run: newman run collection.json -e environment.json
```

## Mock Servers

### What are Mock Servers?

Mock servers simulate your API without needing the actual backend running.

**Why use them?**
- **Frontend Development**: Test frontend before backend is ready
- **Team Collaboration**: Backend and frontend teams work in parallel
- **Testing**: Test error scenarios easily
- **Documentation**: Show API behavior

**Real-world analogy**: Mock server is like a practice dummy - you can practice without the real thing.

### Creating a Mock Server

**Method 1: From Collection**

1. Right-click collection → **"Mock Collection"**
2. Enter mock server name
3. Select environment (optional)
4. Click **"Create Mock Server"**
5. Copy mock server URL

**Method 2: From Request**

1. Open request
2. Click **"Examples"** tab
3. Click **"Add Example"**
4. Set example response
5. Save example
6. Create mock server from collection

### Setting Up Example Responses

**Why**: Mock server uses examples to return responses.

**Steps:**

1. Open request in collection
2. Click **"Examples"** tab (next to Params, Headers, etc.)
3. Click **"Add Example"**
4. Name example (e.g., "Success Response")
5. Set response:
   - **Status Code**: 200
   - **Body**: JSON response
6. Click **"Save"**

**Example: Task List Response**

```
Status: 200 OK
Body (raw JSON):
{
  "count": 2,
  "next": null,
  "previous": null,
  "results": [
    {
      "id": 1,
      "title": "Example Task 1",
      "completed": false
    },
    {
      "id": 2,
      "title": "Example Task 2",
      "completed": true
    }
  ]
}
```

### Using Mock Server

**Get Mock URL:**

After creating mock server, you'll get a URL like:
```
https://abc123.mock.pstmn.io
```

**Use in Requests:**

Replace your base URL with mock URL:
```
https://abc123.mock.pstmn.io/api/tasks/
```

**Benefits:**

- Works even if backend is down
- Consistent responses for testing
- Fast responses
- No database needed

### Advanced Mock Server Features

**Multiple Examples:**

Create different examples for different scenarios:
- "Success Response" - 200 OK
- "Not Found" - 404
- "Unauthorized" - 401
- "Server Error" - 500

**Conditional Responses:**

Use pre-request scripts to return different responses based on request data.

**Private Mock Servers:**

- Only accessible with API key
- More secure
- For team use only

## API Documentation

### Auto-Generate Documentation

Postman can automatically generate API documentation from your collections.

**Why:**
- **Always Up-to-date**: Documentation matches your API
- **Easy Sharing**: Share with team or public
- **Interactive**: Try requests directly from docs
- **Professional**: Looks polished

### Generating Documentation

**Steps:**

1. Open collection
2. Click **"View Documentation"** (or right-click → **"View Documentation"**)
3. Click **"Publish"** (if not published)
4. Configure:
   - **Visibility**: Public, Team, or Private
   - **Description**: Add overview
   - **Custom Domain**: Use your domain (optional)
5. Click **"Publish"**
6. Copy documentation URL

### Documenting Requests

**Add Descriptions:**

1. Open request
2. Click request name to edit
3. Add description in markdown:

```markdown
## Get Task List

Retrieves a list of all tasks for the authenticated user.

### Parameters

- `page` (optional): Page number for pagination
- `limit` (optional): Number of items per page

### Response

Returns a paginated list of tasks.
```

**Add Examples:**

1. Go to **"Examples"** tab
2. Add multiple examples:
   - Success response
   - Error responses
   - Edge cases

**Add Request/Response Schemas:**

Document the expected request and response structure.

### Sharing Documentation

**Public Documentation:**

- Anyone with link can view
- Great for public APIs
- Searchable

**Team Documentation:**

- Only team members can view
- Requires Postman account
- Can comment and collaborate

**Private Documentation:**

- Only you can view
- For personal use
- Still accessible via link

## Testing DRF APIs - Step by Step

### Complete Workflow Example

Let's test a complete DRF Task API workflow.

**Prerequisites:**
- Django server running on `http://localhost:8000`
- Task API with authentication
- User account created

### Step 1: Setup Environment

**Create Environment Variables:**

1. Create "Development" environment
2. Add variables:
   - `base_url`: `http://localhost:8000`
   - `username`: `admin`
   - `password`: `password123`
   - `access_token`: (leave empty)
   - `refresh_token`: (leave empty)

### Step 2: Create Collection

**Create "DRF Task API" Collection**

Organize requests in folders:
- Authentication
- Tasks (CRUD operations)

### Step 3: Login Request

**Request Configuration:**

```
Method: POST
URL: {{base_url}}/api/token/
Headers:
  Content-Type: application/json
Body (raw JSON):
{
  "username": "{{username}}",
  "password": "{{password}}"
}
```

**Tests:**

```javascript
// Check status
pm.test("Status is 200", () => {
    pm.response.to.have.status(200);
});

// Save tokens
const response = pm.response.json();
pm.environment.set("access_token", response.access);
pm.environment.set("refresh_token", response.refresh);

// Verify tokens exist
pm.test("Access token received", () => {
    pm.expect(response).to.have.property("access");
    pm.expect(response.access).to.be.a("string");
});
```

### Step 4: List Tasks

**Request Configuration:**

```
Method: GET
URL: {{base_url}}/api/tasks/
Authorization:
  Type: Bearer Token
  Token: {{access_token}}
```

**Tests:**

```javascript
pm.test("Status is 200", () => {
    pm.response.to.have.status(200);
});

pm.test("Response is JSON", () => {
    pm.response.to.be.json;
});

pm.test("Has results array", () => {
    const response = pm.response.json();
    pm.expect(response).to.have.property("results");
    pm.expect(response.results).to.be.an("array");
});

pm.test("Fast response", () => {
    pm.expect(pm.response.responseTime).to.be.below(500);
});
```

### Step 5: Create Task

**Request Configuration:**

```
Method: POST
URL: {{base_url}}/api/tasks/
Authorization:
  Type: Bearer Token
  Token: {{access_token}}
Headers:
  Content-Type: application/json
Body (raw JSON):
{
  "title": "Test Task from Postman",
  "description": "Created via Postman",
  "completed": false
}
```

**Tests:**

```javascript
pm.test("Status is 201", () => {
    pm.response.to.have.status(201);
});

pm.test("Task created with ID", () => {
    const response = pm.response.json();
    pm.expect(response).to.have.property("id");
    pm.environment.set("created_task_id", response.id);
});

pm.test("Task has correct data", () => {
    const response = pm.response.json();
    pm.expect(response.title).to.equal("Test Task from Postman");
    pm.expect(response.completed).to.be.false;
});
```

### Step 6: Get Single Task

**Request Configuration:**

```
Method: GET
URL: {{base_url}}/api/tasks/{{created_task_id}}/
Authorization:
  Type: Bearer Token
  Token: {{access_token}}
```

**Tests:**

```javascript
pm.test("Status is 200", () => {
    pm.response.to.have.status(200);
});

pm.test("Returns correct task", () => {
    const response = pm.response.json();
    pm.expect(response.id).to.equal(parseInt(pm.environment.get("created_task_id")));
});
```

### Step 7: Update Task

**Request Configuration:**

```
Method: PATCH
URL: {{base_url}}/api/tasks/{{created_task_id}}/
Authorization:
  Type: Bearer Token
  Token: {{access_token}}
Headers:
  Content-Type: application/json
Body (raw JSON):
{
  "completed": true
}
```

**Tests:**

```javascript
pm.test("Status is 200", () => {
    pm.response.to.have.status(200);
});

pm.test("Task updated", () => {
    const response = pm.response.json();
    pm.expect(response.completed).to.be.true;
});
```

### Step 8: Delete Task

**Request Configuration:**

```
Method: DELETE
URL: {{base_url}}/api/tasks/{{created_task_id}}/
Authorization:
  Type: Bearer Token
  Token: {{access_token}}
```

**Tests:**

```javascript
pm.test("Status is 204", () => {
    pm.response.to.have.status(204);
});
```

### Step 9: Test Error Cases

**Test 401 Unauthorized:**

```
Method: GET
URL: {{base_url}}/api/tasks/
Authorization: (remove or use invalid token)
```

**Tests:**

```javascript
pm.test("Status is 401", () => {
    pm.response.to.have.status(401);
});
```

**Test 404 Not Found:**

```
Method: GET
URL: {{base_url}}/api/tasks/99999/
Authorization:
  Type: Bearer Token
  Token: {{access_token}}
```

**Tests:**

```javascript
pm.test("Status is 404", () => {
    pm.response.to.have.status(404);
});
```

### Step 10: Run Collection

1. Right-click collection → **"Run collection"**
2. Ensure requests are in correct order:
   - Login
   - List Tasks
   - Create Task
   - Get Task
   - Update Task
   - Delete Task
3. Click **"Run"**
4. Review results

## Common Errors and Solutions

### Error: "Could not get response"

**Causes:**
- Server not running
- Wrong URL
- Network issues
- Firewall blocking

**Solutions:**
1. Check if Django server is running: `python manage.py runserver`
2. Verify URL is correct
3. Check network connection
4. Try `http://localhost:8000` in browser first

### Error: "401 Unauthorized"

**Causes:**
- Missing authentication
- Invalid token
- Expired token

**Solutions:**
1. Check Authorization tab is configured
2. Verify token is valid (not expired)
3. Re-login to get new token
4. Check token format: `Bearer <token>`

### Error: "404 Not Found"

**Causes:**
- Wrong URL path
- API endpoint doesn't exist
- Typo in URL

**Solutions:**
1. Verify URL matches Django URLs configuration
2. Check if endpoint exists: `python manage.py show_urls`
3. Test URL in browser first
4. Check for trailing slashes

### Error: "400 Bad Request"

**Causes:**
- Invalid JSON in body
- Missing required fields
- Wrong data types

**Solutions:**
1. Validate JSON syntax (use JSON validator)
2. Check required fields in API documentation
3. Verify data types match API expectations
4. Check Content-Type header is `application/json`

### Error: "500 Internal Server Error"

**Causes:**
- Server-side error
- Database issue
- Code bug

**Solutions:**
1. Check Django server logs
2. Verify database is accessible
3. Check for Python errors in console
4. Review API code for bugs

### Error: "CORS Error"

**Causes:**
- CORS not configured in Django
- Wrong origin

**Solutions:**
1. Install django-cors-headers: `pip install django-cors-headers`
2. Add to INSTALLED_APPS: `'corsheaders'`
3. Add middleware
4. Configure CORS_ALLOWED_ORIGINS

### Error: "SSL Certificate Error"

**Causes:**
- Self-signed certificate
- Invalid certificate

**Solutions:**
1. For development: Disable SSL verification (Settings → General)
2. For production: Use valid SSL certificate
3. Add certificate to Postman (Settings → Certificates)

## Best Practices

### 1. Organize Your Collections

**Do:**
- Group related requests
- Use folders for categories
- Name requests descriptively
- Add descriptions

**Don't:**
- Put everything in one collection
- Use vague names like "Test 1"
- Mix different APIs

### 2. Use Environments

**Do:**
- Create separate environments for dev/staging/prod
- Use variables for URLs and credentials
- Keep sensitive data in environment variables

**Don't:**
- Hardcode URLs in requests
- Commit environment files with real credentials
- Mix environment values

### 3. Write Meaningful Tests

**Do:**
- Test status codes
- Validate response structure
- Check response times
- Use descriptive test names

**Don't:**
- Skip tests
- Write vague test names
- Test only happy paths

### 4. Document Your Requests

**Do:**
- Add descriptions to requests
- Document parameters
- Include examples
- Explain expected responses

**Don't:**
- Leave requests undocumented
- Assume others know what requests do

### 5. Version Control

**Do:**
- Export collections regularly
- Commit collection files to Git
- Tag versions
- Document changes

**Don't:**
- Only keep collections in Postman
- Forget to export
- Lose work

### 6. Security

**Do:**
- Use environment variables for secrets
- Don't commit tokens
- Rotate API keys regularly
- Use different credentials per environment

**Don't:**
- Hardcode passwords
- Share environment files publicly
- Use production credentials in dev

## Exercises

### Exercise 1: Complete CRUD Workflow

**Task:**
1. Create a collection for a Book API
2. Set up environment with base URL
3. Create requests for:
   - Login (get token)
   - List books
   - Create book
   - Get single book
   - Update book
   - Delete book
4. Add tests to each request
5. Run collection and verify all pass

### Exercise 2: Error Handling

**Task:**
1. Create requests to test error cases:
   - 401 Unauthorized (no token)
   - 404 Not Found (invalid ID)
   - 400 Bad Request (invalid data)
   - 403 Forbidden (no permission)
2. Write tests to verify correct error codes
3. Test error response structure

### Exercise 3: Pagination Testing

**Task:**
1. Create requests to test pagination:
   - Get first page
   - Get second page
   - Test page size parameter
2. Write tests to verify:
   - Correct number of items per page
   - Next/previous page links
   - Total count

### Exercise 4: Filtering and Searching

**Task:**
1. Test filtering endpoints:
   - Filter by status
   - Filter by date range
   - Search by keyword
2. Write tests to verify:
   - Filters work correctly
   - Results match filters
   - Empty results handled

### Exercise 5: Token Refresh Flow

**Task:**
1. Create pre-request script to:
   - Check if token is expired
   - Refresh token if needed
   - Use new token automatically
2. Test with expired token
3. Verify token refresh works

### Exercise 6: Mock Server

**Task:**
1. Create mock server for your API
2. Add example responses for:
   - Success cases
   - Error cases
3. Test frontend with mock server
4. Verify mock responses match real API

### Exercise 7: Collection Runner

**Task:**
1. Set up collection runner
2. Configure:
   - Multiple iterations
   - Delay between requests
   - Data file with test data
3. Run collection
4. Analyze results
5. Export report

### Exercise 8: Documentation

**Task:**
1. Add descriptions to all requests
2. Document parameters and responses
3. Add multiple examples
4. Publish documentation
5. Share with team

## Next Steps

Congratulations! You've completed both parts of the Postman guide. You now know:

**Part 1:**
- ✅ Installation and setup
- ✅ Interface navigation
- ✅ Basic requests
- ✅ Collections and environments

**Part 2:**
- ✅ Authentication (JWT, Basic, API Key)
- ✅ Writing tests and scripts
- ✅ Automation and collection runner
- ✅ Mock servers and documentation
- ✅ Complete DRF API testing workflow

**Continue Learning:**

1. **Practice**: Test your own APIs
2. **Explore**: Try advanced Postman features
3. **Automate**: Set up CI/CD with Newman
4. **Collaborate**: Share collections with team
5. **Document**: Create comprehensive API docs

**Resources:**

- [Postman Learning Center](https://learning.postman.com/)
- [Postman API Documentation](https://www.postman.com/api-documentation/)
- [Newman Documentation](https://github.com/postmanlabs/newman)
- [Postman Community](https://community.postman.com/)

**Related Guides:**

- [cURL Guide](CURL_GUIDE.md) - Command-line API testing
- [Level 2: Intermediate](LEVEL_2_INTERMEDIATE.md) - DRF authentication and permissions

---

**Happy API Testing!** 🚀

