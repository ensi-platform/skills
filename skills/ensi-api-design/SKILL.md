---
name: ensi-api-design
description: Apply Ensi API Design Guide principles when designing, implementing, or reviewing REST API endpoints in Ensi projects. Use this skill when creating API endpoints, defining request/response formats, implementing filters and pagination, working with OpenAPI specifications, or any API-related work in Ensi services.
---

# Ensi API Design Guide

This skill ensures that all API endpoints in Ensi projects follow the official API Design Guide. Apply these standards when designing, implementing, or reviewing REST APIs.

## Core Principles

**Always follow these priorities:**
1. REST architectural style for all API implementations
2. JSON format for all data transfer
3. Consistent naming conventions (kebab-case for URLs, snake_case for fields)
4. Versioning in URL paths (/v1/, /v2/)
5. Standard response format with data, errors, and meta fields
6. Design-first approach with OpenAPI 3.0 specifications

## General Rules

### URL Structure

**Resource names MUST be plural (except single-instance resources):**
```
Good: POST /v1/users
Bad: POST /v1/user
Good: GET /v1/profile (single instance allowed)
```

**Resource names MUST be kebab-case:**
```
Good: GET /v1/customer-addresses
Bad: GET /v1/customerAddresses
```

**Query parameters and body fields MUST be snake_case:**
```
Good: ?last_login_at=2020-01-01
Bad: ?lastLoginAt=2020-01-01
```

**API version MUST always be in URL:**
```
Good: GET /v1/users
Bad: GET /users
```

**Non-existent pages MUST return 404 with proper error format:**
```json
{
  "data": null,
  "errors": [
    {
      "code": "NotFoundHttpException",
      "message": "Resource not found"
    }
  ]
}
```

### Field Formats

**Integer for all entity IDs:**
```json
Good: { "id": 12345 }
Bad: { "id": "12345" }
```

**ISO-8601 for datetime fields (UTC):**
```json
Good: { "updated_at": "2020-01-01T15:47:21.000000Z" }
Bad: { "updated_at": "2020-01-01 15:47:21" }
```

**ISO-8601 full date for dates:**
```json
Good: { "birthday": "1990-01-25" }
Bad: { "birthday": "1990/01/25" }
```

**Prices in kopecks (integer):**
```json
Good: { "price": 1000 }  // 10 rubles
Bad: { "price": 10.00 }
```

## Response Format

All responses MUST contain only these fields:

**data** - main response object:
- `null` - when no data
- `object` - single entity response
- `array` - list of entities

**errors** - optional array of error objects:
```json
{
  "code": "ErrorType",
  "message": "Human-readable error description",
  "meta": {
    "additional": "debug information"
  }
}
```

**meta** - optional object with additional information:
```json
{
  "meta": {
    "pagination": { /* ... */ },
    "debug": { /* ... */ }
  }
}
```

## API Architecture Types

### Back Services (Internal API)

Design APIs around resources with standard CRUD methods.

#### Standard Methods Table

| Method | HTTP Method | Purpose |
|--------|-------------|---------|
| Get | `GET /v1/users/{id}` | Get entity by ID |
| Create | `POST /v1/users` | Create new entity |
| Replace | `PUT /v1/users/{id}` | Update all fields |
| Patch | `PATCH /v1/users/{id}` | Update specific fields |
| Delete | `DELETE /v1/users/{id}` | Delete entity |
| Search | `POST /v1/users:search` | Search with filters |
| SearchOne | `POST /v1/users:search-one` | Get single entity by filters |

#### Get Method

**Request:**
```
GET /v1/users/17?include=addresses,loyality_cards
```

**Response:**
```json
{
  "data": {
    "id": 17,
    "name": "John Doe",
    "last_login_at": "2020-01-01T15:47:21.000000Z",
    "addresses": [/* ... */],
    "loyality_cards": [/* ... */]
  }
}
```

#### Create Method

**Request:**
```
POST /v1/users
{
  "name": "John Doe",
  "last_login_at": "2020-01-01T15:47:21.000000Z"
}
```

**Response:**
```json
{
  "data": {
    "id": 1006779,
    "name": "John Doe",
    "last_login_at": "2020-01-01T15:47:21.000000Z"
  }
}
```

#### Replace Method

**Request:**
```
PUT /v1/users/1006779
{
  "name": "John Doe",
  "last_login_at": null
}
```

**Requirements:**
- MUST be idempotent
- Optional fields set to default if omitted
- `id` field MUST be ignored

**Response:** Same as Get method

#### Patch Method

**Request:**
```
PATCH /v1/users/1006779
{
  "name": "Jane Doe"
}
```

**Requirements:**
- Only specified fields are updated
- `id` field MUST be ignored

**Response:** Same as Get method

#### Delete Method

**Request:**
```
DELETE /v1/users/1006779
```

**Requirements:**
- MUST NOT return 404 if already deleted
- Deleting methods should be idempotent

#### Search Method

**Request:**
```
POST /v1/users:search
{
  "sort": ["-last_login_at", "id"],
  "filter": {
    "id": [12125, 1006779],
    "last_login_gte": "2020-01-01T15:47:21.000000Z"
  },
  "include": ["roles", "loyality_cards"],
  "pagination": { /* ... */ }
}
```

**Requirements:**
- POST method to avoid GET limitations
- All fields optional

**Response:**
```json
{
  "data": [/* array of users */],
  "meta": {
    "pagination": { /* ... */ }
  }
}
```

#### SearchOne Method

**Request format:** Same as Search

**Response:** Single object (same format as Get)

#### Additional Methods

**Format:** `POST /v1/users:method-name`

**Examples:**
```
POST /v1/users:mass-delete
POST /v1/offer-certificates/5:upload-file
```

**Requirements:**
- MUST use POST
- Method name added via colon
- Request/response formats flexible but similar to standard methods

### Filters

**All filters MUST be in single filter object:**
```json
{
  "filter": {
    "active": true,
    "id": [1, 2],
    "last_login_at_gte": "2020-01-01T15:47:21.000000Z"
  }
}
```

**Rules:**
- No modifier = equality operator (=)
- Array values = OR condition
- All filters combined with AND

### Filter Modifiers

| Modifier | Description | Array Support |
|----------|-------------|---------------|
| `_gt` | Greater than | - |
| `_gte` | Greater or equal | - |
| `_lt` | Less than | - |
| `_lte` | Less or equal | - |
| `_not` | Not equal | + |
| `_and` | AND instead of OR for arrays | + |
| `_like` | LIKE '%value%' | + |
| `_llike` | LIKE '%value' | + |
| `_rlike` | LIKE 'value%' | + |
| `_regex` | Regex match | + |
| `has_*` | Has related entities | - |
| `*_count` | Count of related entities | + |

**Examples:**
```json
{
  "filter": {
    "last_login_at_gte": "2020-01-01",
    "id_not": [1, 2],
    "name_like": "john",
    "roles_count_gte": 2
  }
}
```

**Modifiers can be combined:**
```json
{
  "filter": {
    "orders_count_gte": 5
  }
}
```

### Reserved Filters

| Filter | Description | Array Support |
|--------|-------------|---------------|
| `id` | Primary key filter | + |
| `trashed` | Soft delete filter | - |
| `query` | DSL query string | - |

**trashed values:**
- `"with"` - include trashed
- `"only"` - only trashed
- omitted - only non-trashed

### Pagination

**Two pagination types MUST be supported:**

#### Offset Pagination

**Request:**
```json
{
  "pagination": {
    "type": "offset",
    "offset": 40,
    "limit": 20
  }
}
```

**Response:**
```json
{
  "meta": {
    "pagination": {
      "type": "offset",
      "offset": 40,
      "limit": 20,
      "total": 253
    }
  }
}
```

#### Cursor Pagination

**Request:**
```json
{
  "pagination": {
    "type": "cursor",
    "cursor": "eyJpZCI6MTAsIl9wb2ludHNUb05leHRJdGVtcyI6dHJ1ZX0",
    "limit": 20
  }
}
```

**Response:**
```json
{
  "meta": {
    "pagination": {
      "type": "cursor",
      "cursor": "eyJpZCI6MTAsIl9wb2ludHNUb05leHRJdGVtcyI6dHJ1ZX0",
      "limit": 20,
      "next_cursor": "eyJpZCI6MjEsIl9wb2ludHNUb05leHRJdGVtcyI6dHJ1ZX1",
      "previous_cursor": "eyJpZCI6MTIsIl9wb2ludHNUb05leHRJdGVtcyI6ZmFsc2V9"
    }
  }
}
```

**Default pagination:**
```json
{
  "type": "offset",
  "offset": 0,
  "limit": 10
}
```

**Limit behavior:**
- 0 = return 0 elements
- -1 = return all elements (can be disabled)
- > max limit = auto-reduce to max (can be configured)

### Subresources

**NOT RECOMMENDED** - use separate resources instead.

**Exceptions when subresources are acceptable:**
1. Non-unique identifiers (requires {parent_id, child_id} pair)
2. All conditions met:
   - Clear hierarchy
   - Resource cannot exist without parent
   - No meaning outside parent context
   - Constant relationship

**Max nesting depth: 2**

**Module prefix is NOT a subresource:**
```
/api/v1/customers/addresses  // OK (module prefix)
```

### Includes

**Use include parameter for related resources:**

**Get method:**
```
GET /v1/users/17?include=addresses_count,loyality_cards,profile
```

**Search method:**
```json
{
  "include": ["addresses_count", "loyality_cards", "profile"]
}
```

**Include types:**
1. Object - X to One relation (e.g., `profile`)
2. Array - X to Many relation (e.g., `loyality_cards`)
3. Number - Count with `_count` suffix (e.g., `addresses_count`)

**Response:**
```json
{
  "data": {
    "id": 17,
    "addresses_count": 7,
    "loyality_cards": [/* ... */],
    "profile": {/* ... */}
  }
}
```

**Limitations:**
- No complex sorting, filtering, or pagination for includes
- Use separate resources for complex cases

### File Upload

**Large files MUST use separate POST request:**
```
Content-Type: multipart/form-data

POST /v1/offer-certificates/5:upload-file
```

## Gateway Services (External API)

Design for external consumers (facades, external systems).

### Principles

- Specialized API, not flexible
- Don't push business logic to consumers
- One task = one method
- BFF: One screen = one GET method + save methods

### Module Selection

**API for frontend:**
- Module matches frontend section (lk, catalog, cart, checkout)

**Connectors:**
- Module matches domain owner (orders, customers)
- Small connectors may not need domain separation

### When to Use Standard Methods

Use standard Back-service methods when:
1. Consumer uses CRUD approach (e.g., address management on facade)
   - Merge backend resources if needed for frontend
2. Resource search needed
   - Adapt standard approach (e.g., always include required relations)

## HTTP Status Codes

**Use ONLY these status codes:**

| Code | Usage |
|------|-------|
| 200 OK | All successful responses (unless 201 is auto-generated) |
| 201 Created | When framework auto-generates it (otherwise prefer 200) |
| 401 Unauthorized | Authentication required |
| 403 Forbidden | Authorized but not permitted |
| 404 Not Found | Resource/instance doesn't exist |
| 400 Bad Request | Any client error |
| 500 Internal Server Error | Application error (not client fault) |

**Documentation MUST list all possible codes for each endpoint.**

## API Versioning

**Breaking changes strategy:**

1. **Known consumers, easy change** → update immediately
2. **Unknown/hard to update** → evolution principle, technical debt cleanup
3. **Massive changes, impossible evolution** → versioning

**Versioning rules:**
- Global versioning (one version per client)
- Increment version (/v1/ → /v2/)
- Version MUST be in URL
- Separate OpenAPI specs and controllers per version
- Keep all versions in single swagger (use Servers selector, descending order)

## API Documentation

**Requirements:**
- All api endpoints MUST use OpenAPI 3.0 specification
- Documentation MUST be accessible via Stoplight browser
- Design-first approach
- Keep all versions in single swagger
- Select version via Servers dropdown
- Arrange versions in descending order
- Follow OpenAPI specification requirements

## When Applying This Skill

**Design new API endpoint:**
1. Determine if Back-service or Gateway service
2. Choose appropriate standard methods or create custom
3. Define request/response formats
4. Implement filters with proper modifiers
5. Add pagination (both offset and cursor)
6. Design includes if needed
7. Document with OpenAPI 3.0

**Review existing API:**
1. Check URL structure (kebab-case, plural, versioned)
2. Verify field formats (integer IDs, ISO-8601 dates)
3. Validate response format (data, errors, meta)
4. Ensure proper HTTP status codes
5. Check pagination implementation
6. Review filter modifiers and reserved filters
7. Verify OpenAPI documentation

**Implement API in Laravel:**
1. Create controller with proper naming
2. Define routes with kebab-case URLs
3. Use snake_case for request body fields
4. Implement standard CRUD methods
5. Add search endpoint with filters
6. Implement pagination
7. Create FormRequest classes
8. Write API tests

## Common Pitfalls to Avoid

- ❌ Singular resource names (use plural)
- ❌ camelCase in URLs (use kebab-case)
- ❌ Missing API version in URL
- ❌ Using string IDs instead of integers
- ❌ Incorrect datetime formats (use ISO-8601 UTC)
- ❌ Prices in rubles (use kopecks)
- ❌ Missing pagination in list methods
- ❌ Using GET for search (use POST)
- ❌ Subresources when separate resources are better
- ❌ Missing OpenAPI documentation
- ❌ Using HTTP status codes outside the approved list

Remember: **Consistency with existing APIs in the project is paramount**. When working with established codebases, maintain consistency even if it differs slightly from these guidelines.
