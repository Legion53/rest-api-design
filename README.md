# Library Catalogue REST API Design

## Topic: Library Catalogue System
**Entities:** Books, Authors, Categories, Members, Loans

---

## 1. Functional & Non-Functional Requirements

### Functional Requirements

| # | Requirement |
|---|-------------|
| FR-01 | Users can browse, search, and filter books by title, author, category, year, availability |
| FR-02 | Administrators can create, update, and delete books, authors, and categories |
| FR-03 | Members can borrow and return books (create/close loans) |
| FR-04 | The system tracks loan history per member |
| FR-05 | Books can belong to multiple categories |
| FR-06 | Authors can have multiple books; books can have multiple authors |
| FR-07 | Pagination is supported on all list endpoints |
| FR-08 | All mutating operations require authentication (JWT) |
| FR-09 | Read operations for the public catalogue do not require authentication |
| FR-10 | The API returns HATEOAS links in responses |

### Non-Functional Requirements

| # | Requirement |
|---|-------------|
| NFR-01 | **Performance:** List endpoints must respond within 300 ms for up to 100 000 books |
| NFR-02 | **Availability:** 99.9 % uptime SLA |
| NFR-03 | **Security:** All data transmitted over HTTPS/TLS 1.2+ |
| NFR-04 | **Caching:** Public GET responses cached with appropriate `Cache-Control` headers |
| NFR-05 | **Scalability:** Stateless — any instance can serve any request |
| NFR-06 | **Versioning:** API version included in URL path (`/api/v1/`) |
| NFR-07 | **Standards:** JSON (application/json) as the default media type |
| NFR-08 | **Observability:** Every response includes a `X-Request-ID` header for tracing |

---

## 2. Entity (Data Model) Description

### Book
```json
{
  "id": "uuid",
  "isbn": "978-0-06-112008-4",
  "title": "To Kill a Mockingbird",
  "year": 1960,
  "totalCopies": 5,
  "availableCopies": 3,
  "authorIds": ["uuid-author-1"],
  "categoryIds": ["uuid-cat-1", "uuid-cat-2"],
  "_links": {
    "self":       { "href": "/api/v1/books/{id}" },
    "authors":    { "href": "/api/v1/books/{id}/authors" },
    "categories": { "href": "/api/v1/books/{id}/categories" },
    "loans":      { "href": "/api/v1/books/{id}/loans" }
  }
}
```

### Author
```json
{
  "id": "uuid",
  "firstName": "Harper",
  "lastName": "Lee",
  "birthYear": 1926,
  "bio": "American novelist...",
  "_links": {
    "self":  { "href": "/api/v1/authors/{id}" },
    "books": { "href": "/api/v1/authors/{id}/books" }
  }
}
```

### Category
```json
{
  "id": "uuid",
  "name": "Classic Fiction",
  "description": "Timeless works of literary fiction",
  "_links": {
    "self":  { "href": "/api/v1/categories/{id}" },
    "books": { "href": "/api/v1/categories/{id}/books" }
  }
}
```

### Member
```json
{
  "id": "uuid",
  "email": "jane.doe@example.com",
  "fullName": "Jane Doe",
  "memberSince": "2022-03-15",
  "status": "ACTIVE",
  "_links": {
    "self":  { "href": "/api/v1/members/{id}" },
    "loans": { "href": "/api/v1/members/{id}/loans" }
  }
}
```

### Loan
```json
{
  "id": "uuid",
  "bookId": "uuid",
  "memberId": "uuid",
  "borrowedAt": "2024-11-01T10:00:00Z",
  "dueAt": "2024-11-15T10:00:00Z",
  "returnedAt": null,
  "status": "ACTIVE",
  "_links": {
    "self":   { "href": "/api/v1/loans/{id}" },
    "book":   { "href": "/api/v1/books/{bookId}" },
    "member": { "href": "/api/v1/members/{memberId}" }
  }
}
```

---

## 3. Authentication

### Method: Bearer Token (JWT)

All write operations (`POST`, `PUT`, `PATCH`, `DELETE`) and member-specific reads require an `Authorization` header:

```
Authorization: Bearer <JWT>
```

### Obtaining a Token

```
POST /api/v1/auth/token
Content-Type: application/json

{ "email": "admin@library.com", "password": "secret" }
```

**Response 200 OK:**
```json
{
  "accessToken": "<JWT>",
  "tokenType": "Bearer",
  "expiresIn": 3600
}
```

### JWT Structure

**Header:**
```json
{ "alg": "RS256", "typ": "JWT" }
```

**Payload:**
```json
{
  "sub": "uuid-member-or-admin",
  "role": "ADMIN | MEMBER",
  "iat": 1700000000,
  "exp": 1700003600
}
```

### Roles

| Role | Permissions |
|------|-------------|
| `MEMBER` | Read catalogue; manage own loans |
| `ADMIN` | Full CRUD on all resources |

### Authentication Error Responses

| Status | Code | Description |
|--------|------|-------------|
| 401 | `MISSING_TOKEN` | `Authorization` header absent |
| 401 | `INVALID_TOKEN` | Malformed or expired JWT |
| 403 | `FORBIDDEN` | Valid token but insufficient role |

---

## 4. REST API Description

### Base URL
```
https://api.library.example.com/api/v1
```

### Richardson Maturity Model — Level 3 (HATEOAS)

All responses include a `_links` object with hypermedia controls. Clients discover available actions from the links rather than hardcoding URLs.

---

### 4.1 Books

#### `GET /books` — List books

**Query parameters:**

| Parameter | Type | Description |
|-----------|------|-------------|
| `q` | string | Full-text search (title, ISBN) |
| `authorId` | uuid | Filter by author |
| `categoryId` | uuid | Filter by category |
| `year` | integer | Filter by publication year |
| `available` | boolean | Filter only available copies |
| `page` | integer ≥ 1 | Page number (default: 1) |
| `pageSize` | integer 1–100 | Items per page (default: 20) |
| `sort` | string | `title`, `year`, `-year` (prefix `-` = descending) |

**Headers (response):**

| Header | Value |
|--------|-------|
| `Cache-Control` | `public, max-age=60` |
| `X-Request-ID` | UUID |

**Response 200 OK:**
```json
{
  "data": [ { /* Book */ } ],
  "pagination": {
    "page": 1,
    "pageSize": 20,
    "totalItems": 342,
    "totalPages": 18
  },
  "_links": {
    "self":  { "href": "/api/v1/books?page=1&pageSize=20" },
    "next":  { "href": "/api/v1/books?page=2&pageSize=20" },
    "prev":  null,
    "first": { "href": "/api/v1/books?page=1&pageSize=20" },
    "last":  { "href": "/api/v1/books?page=18&pageSize=20" }
  }
}
```

**Status codes:**

| Code | Meaning |
|------|---------|
| 200 | Success |
| 400 | Invalid query parameter (e.g. `pageSize=0`) |

---

#### `POST /books` — Create a book

**Auth required:** ADMIN

**Request body:**
```json
{
  "isbn": "978-0-06-112008-4",
  "title": "To Kill a Mockingbird",
  "year": 1960,
  "totalCopies": 5,
  "authorIds": ["uuid-author-1"],
  "categoryIds": ["uuid-cat-1"]
}
```

**Response 201 Created:**
```
Location: /api/v1/books/new-uuid
```
Body: created Book object.

**Status codes:**

| Code | Meaning |
|------|---------|
| 201 | Book created |
| 400 | Validation error (missing fields, invalid ISBN) |
| 401 | Missing or invalid token |
| 403 | Caller is not ADMIN |
| 409 | ISBN already exists |

---

#### `GET /books/{id}` — Get single book

**Headers (response):** `Cache-Control: public, max-age=300`

**Status codes:**

| Code | Meaning |
|------|---------|
| 200 | Success |
| 404 | Book not found |

---

#### `PUT /books/{id}` — Full update

**Auth required:** ADMIN

**Status codes:**

| Code | Meaning |
|------|---------|
| 200 | Updated successfully |
| 400 | Validation error |
| 401 | Missing or invalid token |
| 403 | Not ADMIN |
| 404 | Book not found |
| 409 | ISBN conflict with another book |

---

#### `PATCH /books/{id}` — Partial update

**Auth required:** ADMIN

Request body contains only fields to update (e.g. `{ "totalCopies": 10 }`).

**Status codes:** same as `PUT /books/{id}`.

---

#### `DELETE /books/{id}` — Delete book

**Auth required:** ADMIN

**Status codes:**

| Code | Meaning |
|------|---------|
| 204 | Deleted; no content |
| 401 | Missing or invalid token |
| 403 | Not ADMIN |
| 404 | Book not found |
| 409 | Cannot delete — active loans exist |

---

#### `GET /books/{id}/authors` — Authors of a book

**Headers (response):** `Cache-Control: public, max-age=300`

**Status codes:** 200, 404

---

#### `GET /books/{id}/categories` — Categories of a book

**Headers (response):** `Cache-Control: public, max-age=300`

**Status codes:** 200, 404

---

### 4.2 Authors

#### `GET /authors` — List authors

**Query parameters:** `q` (name search), `page`, `pageSize`, `sort` (`lastName`, `-lastName`)

**Headers (response):** `Cache-Control: public, max-age=300`

**Status codes:** 200, 400

---

#### `POST /authors` — Create author

**Auth required:** ADMIN

**Status codes:** 201, 400, 401, 403

---

#### `GET /authors/{id}` — Get author

**Headers (response):** `Cache-Control: public, max-age=300`

**Status codes:** 200, 404

---

#### `PUT /authors/{id}` — Full update

**Auth required:** ADMIN

**Status codes:** 200, 400, 401, 403, 404

---

#### `DELETE /authors/{id}` — Delete author

**Auth required:** ADMIN

**Status codes:** 204, 401, 403, 404, 409 (books reference this author)

---

#### `GET /authors/{id}/books` — Books by author

**Query parameters:** `page`, `pageSize`

**Headers (response):** `Cache-Control: public, max-age=60`

**Status codes:** 200, 404

---

### 4.3 Categories

#### `GET /categories` — List categories

**Headers (response):** `Cache-Control: public, max-age=600`

**Status codes:** 200

---

#### `POST /categories` — Create category

**Auth required:** ADMIN

**Status codes:** 201, 400, 401, 403, 409

---

#### `GET /categories/{id}` — Get category

**Headers (response):** `Cache-Control: public, max-age=600`

**Status codes:** 200, 404

---

#### `PUT /categories/{id}` — Update category

**Auth required:** ADMIN

**Status codes:** 200, 400, 401, 403, 404, 409

---

#### `DELETE /categories/{id}` — Delete category

**Auth required:** ADMIN

**Status codes:** 204, 401, 403, 404, 409

---

#### `GET /categories/{id}/books` — Books in category

**Query parameters:** `page`, `pageSize`, `available`

**Headers (response):** `Cache-Control: public, max-age=60`

**Status codes:** 200, 404

---

### 4.4 Members

#### `POST /members` — Register member (sign-up)

No auth required.

**Status codes:** 201, 400, 409 (email already registered)

---

#### `GET /members/{id}` — Get member profile

**Auth required:** MEMBER (own profile) or ADMIN

**Caching:** No caching (private data). `Cache-Control: no-store`

**Status codes:** 200, 401, 403, 404

---

#### `PATCH /members/{id}` — Update member

**Auth required:** MEMBER (own) or ADMIN

**Status codes:** 200, 400, 401, 403, 404

---

#### `DELETE /members/{id}` — Delete member

**Auth required:** ADMIN

**Status codes:** 204, 401, 403, 404, 409 (active loans)

---

#### `GET /members/{id}/loans` — Loan history of member

**Auth required:** MEMBER (own) or ADMIN

**Query parameters:** `status` (`ACTIVE`, `RETURNED`), `page`, `pageSize`

**Caching:** `Cache-Control: no-store`

**Status codes:** 200, 401, 403, 404

---

### 4.5 Loans

#### `POST /loans` — Borrow a book

**Auth required:** MEMBER or ADMIN

**Request body:**
```json
{
  "bookId": "uuid",
  "memberId": "uuid",
  "dueAt": "2024-11-15T10:00:00Z"
}
```

**Status codes:**

| Code | Meaning |
|------|---------|
| 201 | Loan created |
| 400 | Validation error or invalid date |
| 401 | Missing or invalid token |
| 403 | Member may not borrow on behalf of another member |
| 404 | Book or member not found |
| 409 | No copies available / member has overdue loans |

---

#### `GET /loans/{id}` — Get loan details

**Auth required:** MEMBER (own loan) or ADMIN

**Caching:** `Cache-Control: no-store`

**Status codes:** 200, 401, 403, 404

---

#### `PATCH /loans/{id}/return` — Return a book

**Auth required:** MEMBER (own) or ADMIN

**Request body:** `{}` (no body needed; action is implicit)

**Status codes:**

| Code | Meaning |
|------|---------|
| 200 | Book returned; loan status set to `RETURNED` |
| 400 | Loan already returned |
| 401 | Missing or invalid token |
| 403 | Not own loan and not ADMIN |
| 404 | Loan not found |

---

#### `GET /loans` — List all loans (admin view)

**Auth required:** ADMIN

**Query parameters:** `memberId`, `bookId`, `status`, `page`, `pageSize`

**Caching:** `Cache-Control: no-store`

**Status codes:** 200, 401, 403

---

### 4.6 Authentication

#### `POST /auth/token` — Obtain JWT

No auth required.

**Status codes:**

| Code | Meaning |
|------|---------|
| 200 | Token issued |
| 400 | Missing email or password |
| 401 | Invalid credentials |

---

#### `POST /auth/refresh` — Refresh JWT

**Status codes:** 200, 401 (refresh token expired)

---

## 5. Error Response Format

All errors use a consistent envelope:

```json
{
  "error": {
    "status": 404,
    "code": "BOOK_NOT_FOUND",
    "message": "No book found with id '3fa85f64-5717-4562-b3fc-2c963f66afa6'",
    "timestamp": "2024-11-01T12:00:00Z",
    "requestId": "7f000001-0000-0000-0000-000000000abc"
  }
}
```

### Error Code Reference

| HTTP Status | Error Code | Scenario |
|-------------|------------|----------|
| 400 | `VALIDATION_ERROR` | Body/query fails validation |
| 400 | `INVALID_ISBN` | ISBN format wrong |
| 400 | `LOAN_ALREADY_RETURNED` | Returning an already-returned loan |
| 401 | `MISSING_TOKEN` | No `Authorization` header |
| 401 | `INVALID_TOKEN` | Expired or malformed JWT |
| 401 | `INVALID_CREDENTIALS` | Wrong email/password at login |
| 403 | `FORBIDDEN` | Token valid but role insufficient |
| 404 | `BOOK_NOT_FOUND` | Book ID does not exist |
| 404 | `AUTHOR_NOT_FOUND` | Author ID does not exist |
| 404 | `CATEGORY_NOT_FOUND` | Category ID does not exist |
| 404 | `MEMBER_NOT_FOUND` | Member ID does not exist |
| 404 | `LOAN_NOT_FOUND` | Loan ID does not exist |
| 409 | `ISBN_CONFLICT` | Duplicate ISBN on create/update |
| 409 | `NO_COPIES_AVAILABLE` | Book fully borrowed |
| 409 | `HAS_ACTIVE_LOANS` | Delete attempted with open loans |
| 409 | `OVERDUE_LOANS` | Member has overdue books |
| 422 | `BUSINESS_RULE_VIOLATION` | Any other domain rule violation |
| 500 | `INTERNAL_ERROR` | Unexpected server error |

---

## 6. Caching Policy

| Endpoint | Method | Cache-Control | Reason |
|----------|--------|---------------|--------|
| `GET /books` | GET | `public, max-age=60` | Public catalogue; refreshed per minute |
| `GET /books/{id}` | GET | `public, max-age=300` | Stable data; 5-min TTL |
| `GET /books/{id}/authors` | GET | `public, max-age=300` | Rarely changes |
| `GET /books/{id}/categories` | GET | `public, max-age=300` | Rarely changes |
| `GET /authors` | GET | `public, max-age=300` | Stable list |
| `GET /authors/{id}` | GET | `public, max-age=300` | Stable |
| `GET /authors/{id}/books` | GET | `public, max-age=60` | May change when books added |
| `GET /categories` | GET | `public, max-age=600` | Very stable |
| `GET /categories/{id}` | GET | `public, max-age=600` | Very stable |
| `GET /categories/{id}/books` | GET | `public, max-age=60` | Books may change |
| `GET /members/{id}` | GET | `no-store` | Private; must not be cached |
| `GET /members/{id}/loans` | GET | `no-store` | Private; dynamic |
| `GET /loans` | GET | `no-store` | Private admin data |
| `GET /loans/{id}` | GET | `no-store` | Private |
| All POST/PUT/PATCH/DELETE | — | `no-store` | Mutating; never cached |

**Note:** Responses with `no-store` instruct both clients and intermediary proxies never to store the response. `public, max-age=N` allows shared caches (CDN) to serve the cached copy for N seconds, reducing database load for the public catalogue.

ETag support is recommended for `GET /books/{id}`, `GET /authors/{id}`, and `GET /categories/{id}` to enable conditional GET (`If-None-Match: "etag-value"`) returning `304 Not Modified` when unchanged.

---

## 7. Pagination Strategy

All list endpoints support cursor-free offset pagination via `page` and `pageSize`.

**Request example:**
```
GET /api/v1/books?page=3&pageSize=10
```

**Response envelope:**
```json
{
  "data": [...],
  "pagination": {
    "page": 3,
    "pageSize": 10,
    "totalItems": 342,
    "totalPages": 35
  },
  "_links": {
    "self":  { "href": "/api/v1/books?page=3&pageSize=10" },
    "first": { "href": "/api/v1/books?page=1&pageSize=10" },
    "prev":  { "href": "/api/v1/books?page=2&pageSize=10" },
    "next":  { "href": "/api/v1/books?page=4&pageSize=10" },
    "last":  { "href": "/api/v1/books?page=35&pageSize=10" }
  }
}
```

`prev` is `null` on the first page; `next` is `null` on the last page.

---

## 8. Richardson Maturity Model — HATEOAS Demonstration

This API operates at **Level 3 (HATEOAS)**:

- **Level 0 ✅** — Uses HTTP as a transport protocol.
- **Level 1 ✅** — Each resource has a distinct URI (`/books`, `/books/{id}`, `/authors/{id}`, etc.).
- **Level 2 ✅** — HTTP verbs are used semantically: GET (read), POST (create), PUT/PATCH (update), DELETE (remove). Status codes carry meaning (201 Created, 204 No Content, 409 Conflict, etc.).
- **Level 3 ✅** — Every response includes `_links` that tell the client what it can do next, without out-of-band knowledge.

**Example — borrowing a book flow driven by links:**

1. `GET /books/abc` → response includes `_links.loans: { "href": "/api/v1/books/abc/loans", "method": "POST" }`
2. Client POSTs to that link → `201` with `_links.return: { "href": "/api/v1/loans/xyz/return", "method": "PATCH" }`
3. Client PATCHes that link → `200` returned.

The client never hardcodes URLs — it follows hypermedia controls.
