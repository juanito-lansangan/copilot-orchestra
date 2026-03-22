---
name: api-documentation
description: 'Expert API documentation skill using Bruno or Postman. Use when creating API collections, documenting endpoints, generating request files from Laravel routes, organizing API requests into folders, adding authentication headers, environment variables, request examples, or exporting API docs. Activates when the user mentions Bruno, Postman, API collection, API docs, endpoint documentation, request examples, or needs to document REST APIs.'
argument-hint: 'Describe what API endpoints or collections you need documented'
---

# API Documentation with Bruno & Postman

## Before You Start (REQUIRED)

Before generating any request files, **always** perform these discovery steps:

### 1. Check for Existing Collections

Search the workspace for existing API documentation:
- **Bruno**: Look for `bruno.json`, `*.bru` files, or folders with collection structure
- **Postman**: Look for `*.postman_collection.json` files
- **Common locations**: `api-docs/`, `docs/api/`, `bruno/`, root-level collection folders

### 2. Ask the User

Based on what you find, ask the user:

**If an existing collection is found:**
> "I found an existing [Bruno/Postman] collection at `[path]`. Would you like me to:
> 1. **Add new routes** — Create request files only for routes not yet documented
> 2. **Update existing routes** — Update request files that are outdated or incomplete
> 3. **Both** — Add missing routes and update existing ones"

**If no collection is found:**
> "No existing API collection found. Would you like me to:
> 1. Create a new **Bruno** collection (recommended — git-friendly, file-based)
> 2. Create a new **Postman** collection (JSON export)
>
> Where should I put it? (e.g., `api-docs/`, `docs/api/`)"

### 3. Diff Routes vs Existing Docs

When adding to an existing collection:
1. Run `php artisan route:list --json` to get current routes
2. List all existing `.bru` files or Postman items in the collection
3. Compare and identify:
   - **New routes**: Exist in route list but not in collection → create new files
   - **Stale routes**: Exist in collection but not in route list → flag for removal (ask user)
   - **Changed routes**: Route exists but method, URI, or middleware changed → update file
4. Present the diff summary to the user before making changes

## When to Use

- Creating new API collections from scratch
- Generating request files from Laravel route definitions
- Documenting existing endpoints with descriptions, examples, and auth
- Organizing API requests into logical folder structures
- Setting up environment variables for different stages (local, staging, production)
- Exporting or converting between Bruno and Postman formats
- Adding pre-request scripts, tests, or assertions to requests

## Tool Selection

### Bruno (Recommended for Git-based projects)

- **File-based**: Each request is a `.bru` file — version-controlled, diffable, merge-friendly.
- **Folder structure**: Mirrors API resource grouping naturally.
- **No cloud dependency**: Runs fully offline.
- **Use when**: The project is in Git, team collaboration via PRs is important, or the user prefers open-source tools.

### Postman

- **JSON collections**: Single `.postman_collection.json` file per collection.
- **Rich GUI**: Better for visual exploration and manual testing.
- **Cloud sync**: Team sharing via Postman workspaces.
- **Use when**: The user explicitly requests Postman, or the team already uses Postman workspaces.

**Default to Bruno** unless the user specifies Postman.

## Bruno Collection Structure

```
api-docs/
├── bruno.json                    # Collection metadata
├── environments/
│   ├── Local.bru                 # Local dev environment
│   ├── Staging.bru               # Staging environment
│   └── Production.bru            # Production environment
├── Auth/
│   ├── Login.bru
│   ├── Register.bru
│   └── Logout.bru
├── Users/
│   ├── List Users.bru
│   ├── Get User.bru
│   ├── Create User.bru
│   ├── Update User.bru
│   └── Delete User.bru
└── Products/
    ├── List Products.bru
    ├── Get Product.bru
    └── ...
```

### bruno.json

```json
{
  "version": "1",
  "name": "Project Name API",
  "type": "collection",
  "ignore": ["node_modules", ".git"]
}
```

### Environment File Format (.bru)

```bru
vars {
  baseUrl: http://api.example.test
  token:
}

vars:secret [
  token
]
```

### Request File Format (.bru)

```bru
meta {
  name: Get User
  type: http
  seq: 1
}

get {
  url: {{baseUrl}}/api/v1/users/:id
  body: none
  auth: bearer
}

auth:bearer {
  token: {{token}}
}

params:path {
  id: 1
}

headers {
  Accept: application/json
  Content-Type: application/json
}

docs {
  # Get User

  Retrieve a single user by ID.

  ## Path Parameters
  - `id` (integer, required) — The user ID

  ## Response 200
  ```json
  {
    "data": {
      "id": 1,
      "name": "John Doe",
      "email": "john@example.com"
    }
  }
  ```
}
```

### POST/PUT Request with Body

```bru
meta {
  name: Create User
  type: http
  seq: 1
}

post {
  url: {{baseUrl}}/api/v1/users
  body: json
  auth: bearer
}

auth:bearer {
  token: {{token}}
}

headers {
  Accept: application/json
  Content-Type: application/json
}

body:json {
  {
    "name": "John Doe",
    "email": "john@example.com",
    "password": "password123"
  }
}

docs {
  # Create User

  Create a new user account.

  ## Request Body
  - `name` (string, required) — Full name
  - `email` (string, required) — Email address
  - `password` (string, required) — Min 8 characters

  ## Response 201
  ```json
  {
    "data": {
      "id": 1,
      "name": "John Doe",
      "email": "john@example.com"
    }
  }
  ```
}
```

## Postman Collection Structure

```json
{
  "info": {
    "name": "Project Name API",
    "schema": "https://schema.getpostman.com/json/collection/v2.1.0/collection.json"
  },
  "variable": [
    { "key": "baseUrl", "value": "http://api.example.test" },
    { "key": "token", "value": "" }
  ],
  "auth": {
    "type": "bearer",
    "bearer": [{ "key": "token", "value": "{{token}}" }]
  },
  "item": [
    {
      "name": "Auth",
      "item": [
        {
          "name": "Login",
          "request": {
            "method": "POST",
            "url": "{{baseUrl}}/api/v1/login",
            "header": [
              { "key": "Accept", "value": "application/json" },
              { "key": "Content-Type", "value": "application/json" }
            ],
            "body": {
              "mode": "raw",
              "raw": "{\n  \"email\": \"user@example.com\",\n  \"password\": \"password\"\n}",
              "options": { "raw": { "language": "json" } }
            }
          }
        }
      ]
    }
  ]
}
```

## Workflow: Generate from Laravel Routes

### Step 1: Discover Routes

Run `php artisan route:list --json` (or via Docker) to get all registered routes. Filter to API routes only.

### Step 2: Group by Resource

Organize routes by their URL prefix or controller:
- `/api/v1/users/*` → `Users/` folder
- `/api/v1/products/*` → `Products/` folder
- `/api/v1/auth/*` → `Auth/` folder

### Step 3: Map HTTP Methods

| Method | Action | Naming Convention |
|--------|--------|-------------------|
| GET (index) | List resources | `List {Resource}` |
| GET (show) | Get single resource | `Get {Resource}` |
| POST | Create resource | `Create {Resource}` |
| PUT/PATCH | Update resource | `Update {Resource}` |
| DELETE | Delete resource | `Delete {Resource}` |

### Step 4: Extract Request Details

For each route:
1. Check the **Form Request** class for validation rules → document required/optional fields
2. Check the **API Resource** class for response shape → add example responses
3. Check **middleware** for auth requirements → set auth type
4. Check **route parameters** → add path params

### Step 5: Create Files

Generate `.bru` files (Bruno) or add items to the collection JSON (Postman) following the templates above.

### Step 6: Setup Environments

Create environment files for each stage with appropriate `baseUrl` values:
- Local: `http://api.project.test` or `http://localhost:8000`
- Staging: `https://api-staging.project.com`
- Production: `https://api.project.com`

## Documentation Best Practices

1. **Every request needs a description** — What it does, not just the endpoint.
2. **Include example responses** — Show the actual JSON shape with realistic data.
3. **Document error responses** — 401, 403, 404, 422 with example bodies.
4. **List all parameters** — Type, required/optional, constraints, defaults.
5. **Group logically** — By resource or feature, not by HTTP method.
6. **Use variables** — `{{baseUrl}}`, `{{token}}` — never hardcode URLs or tokens.
7. **Add auth to collection level** — Inherit auth in child requests, override only when different.
8. **Keep in sync** — Update docs when routes change. Regenerate if needed.

## Authentication Patterns

### Bearer Token (Sanctum/JWT)

Login request should save token to environment:

Bruno post-response script:
```javascript
const body = res.getBody();
bru.setEnvVar("token", body.token);
```

Postman test script:
```javascript
const response = pm.response.json();
pm.environment.set("token", response.token);
```

### API Key

```bru
headers {
  X-API-Key: {{apiKey}}
}
```

## Converting Between Formats

- **Postman → Bruno**: Bruno can import `.postman_collection.json` directly via the GUI (Collection → Import).
- **Bruno → Postman**: No direct export. Read `.bru` files and construct the Postman JSON structure manually.
