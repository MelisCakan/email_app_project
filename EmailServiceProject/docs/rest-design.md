# REST API Design & Theoretical Foundations

**Course:** Web Technologies (INF2101)  
**Semester:** WS 2026  
**Institution:** Technische Hochschule Mittelhessen (THM) - Campus MNI  
**Document:** `docs/rest-design.md`

---

## 1. REST Fundamentals

### 1.1 Check Your Understanding

#### Q1. Resources and representations
* **Selected Correct Statements:** **B**, **C**
* **Justification for Selected Statements:**
    * **B is Correct:** A resource represents a conceptual entity (e.g., an email). The same resource can have multiple representations based on content negotiation or context (e.g., a JSON representation for a client application, an HTML representation for a web browser, or a plain-text representation for legacy systems).
    * **C is Correct:** Uniform Resource Identifiers (URIs) name and identify resources, while HTTP verbs (GET, POST, PUT, PATCH, DELETE) express the desired semantics or operations to perform on those resources.
* **Explanation of Rejected Statements:**
    * **A is Incorrect:** REST is an architectural style decoupled from underlying data storage mechanisms. A resource does not need a 1:1 mapping to a database table; a single resource (e.g., `EmailDetail`) might aggregate data from multiple database tables (emails, attachments, user profiles) or exist dynamically without direct database persistence.
    * **D is Incorrect:** REST is format-agnostic. While JSON is widely used, REST APIs can transfer representations in XML, HTML, Protocol Buffers, plain text, or binary streams (`application/octet-stream`).

#### Q2. Stateless communication
* **Selected Correct Statements:** **B**
* **Justification for Selected Statements:**
    * **B is Correct:** Statelessness requires every HTTP request from a client to contain all the contextual information (authentication tokens, resource identifiers, state parameters) necessary for the server to understand and process it, along with the persistent state stored on the server.
* **Explanation of Rejected Statements:**
    * **A is Incorrect:** Statelessness refers to session context on the server (the server does not retain client session state between requests). It has no bearing on database storage; application data (like emails) is naturally persisted across requests in a database.
    * **C is Incorrect:** In a stateless architecture, the server must never rely on implicit state from previous requests (such as "assuming request #2 acts on the email selected in request #1"). Every request must explicitly state the target resource ID (e.g., `PATCH /emails/123`).

#### Q3. Safety and idempotency
* **Selected Correct Statements:** **B**, **D**
* **Justification for Selected Statements:**
    * **B is Correct:** An operation is idempotent if the intended server-side side effect of multiple identical requests is identical to that of a single request. The initial `DELETE /emails/42` removes the resource and returns `204 No Content`. A subsequent `DELETE /emails/42` leaves the server state unchanged (the resource remains deleted), even if the response status code is `404 Not Found`.
    * **D is Correct:** Idempotency strictly measures the intended state changes on the server side across repeated executions, not the HTTP status codes or client-side response payloads.
* **Explanation of Rejected Statements:**
    * **A is Incorrect:** `GET` is a safe and read-only method. Triggering an action that alters application state (such as transmitting an email over network protocols) using a `GET` request violates HTTP semantics and exposes the system to accidental triggers via browser prefetching, web crawlers, or proxy caching.
    * **C is Incorrect:** `PATCH` is not guaranteed to be idempotent by specification. While a `PATCH` payload setting absolute values (`{"subject": "Hello"}`) is idempotent, a `PATCH` payload applying delta updates (e.g., JSON Patch operations like `add` to an array or incrementing a counter) produces different server states when repeated.

#### Q4. Responses and API contracts
* **Selected Correct Statements:** **A**, **C**
* **Justification for Selected Statements:**
    * **A is Correct:** The `201 Created` status code indicates successful resource creation and should include a `Location` header containing the canonical URI of the newly created resource (e.g., `Location: /api/v1/emails/123e4567-e89b-12d3-a456-426614174000`).
    * **C is Correct:** The `204 No Content` status code explicitly specifies that the response body is empty. Including a response payload in a 204 response violates the HTTP protocol specification.
* **Explanation of Rejected Statements:**
    * **B is Incorrect:** A `202 Accepted` status code indicates that the request has been accepted for asynchronous processing, but the processing has not been completed. It offers no guarantee that final execution (e.g., actual email delivery to an external mail server) will succeed.

---

### 1.2 Explain and Apply

#### 1. Difference between PUT and PATCH
* **PUT (Full Replacement):** Replaces the entire target resource state with the representation provided in the request payload. Omitted fields are overwritten with null or default values.
* **PATCH (Partial Update):** Modifies only the specified fields of the target resource, leaving all unmentioned attributes unchanged.
* **Idempotent Partial-Update Example:**  
  Applying absolute property replacement via PATCH:
  ```json
  PATCH /emails/123
  { "subject": "Updated Meeting Subject" }
  ```
  Executing this request 1 time or 100 times results in the exact same server state: `subject` equals `"Updated Meeting Subject"`.
* **Non-Idempotent Partial-Update Example:**  
  Applying relative array manipulation or delta operations:
  ```json
  PATCH /emails/123
  { "op": "add", "path": "/to/-", "value": "user@example.com" }
  ```
  Executing this request multiple times appends `"user@example.com"` to the recipient list repeatedly, producing a different array length and state with each execution.

#### 2. Classification of HTTP Methods by Safety and Idempotency

| HTTP Method | Safe? | Idempotent? | Method Guarantee vs. Operation Behavior |
| :--- | :---: | :---: | :--- |
| **GET** | **Yes** | **Yes** | Standard guarantees read-only semantics without side effects. |
| **POST** | **No** | **No** | Used for resource creation or processing; consecutive identical requests create multiple distinct resources. |
| **PUT** | **No** | **Yes** | Replaces target resource state completely; repeated identical payloads produce identical server states. |
| **PATCH** | **No** | **No** (Optional) | Standard does not guarantee idempotency. Idempotency depends on whether the implementation uses absolute value setting vs. delta updates. |
| **DELETE** | **No** | **Yes** | Deletes the resource. The initial call removes state; subsequent calls leave the resource absent. |

* **Guarantee vs. Behavior:** HTTP specification defines method-level semantics (what clients can assume). An individual server implementation *could* incorrectly write a non-idempotent `PUT` handler, but doing so breaks protocol compliance and client expectations.

#### 3. Numeric IDs vs. UUIDs Trade-off
* **Numeric Sequential IDs (Auto-Increment):**
    * *Pros:* Compact storage size (4-8 bytes), highly efficient index lookup and B-tree performance in relational databases.
    * *Cons:* Security vulnerability due to sequential predictability (Resource Enumeration / IDOR attacks). Harder to generate independently in distributed/decentralized architectures without central coordination.
* **UUIDs (Universally Unique Identifiers - e.g., UUIDv4):**
    * *Pros:* Globally unique across distributed nodes without central DB coordination. Unpredictable, preventing resource scanning and enumeration attacks.
    * *Cons:* Larger storage footprint (16 bytes / 36 string characters), slightly higher indexing overhead and fragmentation in B-Trees compared to strictly sequential keys.

#### 4. Overfetching and Underfetching
* **Overfetching:** The client receives significantly more data than needed for a specific user interface view.
    * *Example:* Fetching a list of emails for a mailbox overview screen using `GET /emails` and receiving multi-megabyte plain-text message bodies (`content`) for every email when only `subject`, `from`, and `createdAt` are needed.
* **Underfetching:** A single client request does not return sufficient data, requiring N+1 sequential network calls to construct a single screen.
    * *Example:* Fetching a mailbox list via `GET /emails` which returns only IDs, requiring the client to execute 50 individual `GET /emails/{id}` and `GET /emails/{id}/attachments` requests to display attachment icons and subject lines for 50 emails.

---

## 2. OpenAPI Contract Setup

### 2.1 Scope & Assumptions
1. **Resource Exposure:** The API exposes `Emails`, `Attachments` (metadata + binary content), and `Transmissions`. Storage paths, database entity definitions, and internal framework classes are hidden implementation details.
2. **Authentication & User Context:** Stateless JWT authentication (handled in later project phases) attaches user identity to requests. All `/emails` endpoints operate strictly within the isolated boundary of the authenticated user's mailbox.
3. **Date-Time Convention:** All timestamps (`createdAt`, `updatedAt`, `sentAt`) adhere strictly to **RFC 3339** formatted ISO-8601 strings in UTC (e.g., `2026-10-15T10:30:00Z`).

### 2.2 Versioning
* **Difference between `openapi` and `info.version`:**
    * `openapi: 3.0.3` (or `3.1.1`): Specifies the version of the **OpenAPI Specification standard** used to structure the YAML document itself.
    * `info.version: 1.0.0`: Specifies the version of **your specific API service/contract** (Semantic Versioning for business logic changes).

---

## 3. Path & Operation Design

### 3.1 Method Selection & Choices
* `GET /emails`: List emails (with pagination & filtering).
* `POST /emails`: Create a new email draft.
* `GET /emails/{id}`: Inspect detailed email view.
* `PATCH /emails/{id}`: Update an existing draft email.
* `DELETE /emails/{id}`: Permanently remove an email and its attachments.

**Choice between PUT vs. PATCH:** We chose **`PATCH`** for email modifications. Emails contain client-managed fields (`to`, `cc`, `bcc`, `subject`, `content`) alongside server-managed fields (`status`, `sentAt`, `createdAt`). A `PATCH` request allows clients to update specific fields (e.g., updating only the `subject` or `content`) without having to send the entire payload or risk accidentally clearing omitted array fields.

### 3.2 Bad API Proposal Critique
* **Proposed Bad Design:** `GET /deleteEmail?id=42` and `POST /updateEmail`
* **Problems Identified:**
    1. **HTTP Semantic Violation:** `GET` must be safe and read-only. Performing destructive operations (deletion) via `GET` allows search engine crawlers, browser pre-fetchers, or proxy caches to delete data.
    2. **RPC-style Naming:** URIs specify actions (`deleteEmail`, `updateEmail`) instead of identifying resources (`/emails/{id}`).
    3. **Lack of Standard Verbs:** Disregards standard HTTP verbs (`DELETE`, `PATCH`).
* **RESTful Alternative:**
    * Delete: `DELETE /emails/42`
    * Update: `PATCH /emails/42`

---

## 4. Schemas, Validation & Constraints

### 4.1 Representation Separation (Theory)
* **Domain Objects vs. API Representations:** Persisted domain models include DB-specific keys, internal flags, soft-delete statuses, and storage paths. API representations expose only public interfaces, enforcing security encapsulation and clean boundaries.
* **Mailbox Summary vs. Detailed Email:** Summaries omit heavy fields like `content` to optimize bandwidth and pagination performance, while Detailed views include full message content and attachment metadata.
* **Security & User Endpoints:** Passwords and password hashes must never be serialized or returned in API responses to prevent data leaks, credential sniffing, and log exposure.

### 4.2 Required vs. Server-Controlled Fields
* **Client Writable:** `to`, `cc`, `bcc`, `replyTo`, `subject`, `content`.
* **Server Controlled (Read-Only):** `id`, `status`, `source`, `createdAt`, `updatedAt`, `sentAt`.
* **Server-side Enforcement:** If a client submits `id`, `status`, `createdAt`, or `sentAt` in a `POST` or `PATCH` request body, the server strictly ignores these fields or rejects the request with a `400 Bad Request` / `422 Unprocessable Entity` status.

### 4.3 Recipient Rule (`anyOf` / Validation)
The requirement mandates at least one recipient address across `to`, `cc`, or `bcc`.
* **Client Schema:** Modeled using `anyOf` requiring at least one of `to`, `cc`, or `bcc` to be non-empty.
* **Server Validation:** Server-side domain validators verify that `to.size() + cc.size() + bcc.size() > 0` and validate email string formats.

---

## 5. Transmission as a Resource

### 5.1 Transmission Resource Modeling
* **Path:** `POST /emails/{id}/transmissions`
* **Why direct status change (`PATCH /emails/{id}` with `status=sent`) is invalid:**
  Setting `status=sent` treats email delivery as a simple property change. In reality, sending an email is an action/side-effect involving external mail transfer protocols (SMTP/mock service) that may fail, retry, or process asynchronously. Modeling transmission as a resource (`/transmissions`) represents the execution attempt and provides execution feedback (`status`, `sentAt`, `errorMessage`).

### 5.2 Processing Model & Status Code Selection
* **Chosen Model:** Synchronous execution returning `201 Created` with a `TransmissionResult` payload.
* **Status Changes:**
    * **Success:** Email `status` transitions to `sent`, `sentAt` is populated with current date-time, `updatedAt` is updated.
    * **Failure:** Email `status` transitions to `error`, `sentAt` remains `null`, `updatedAt` is updated.
* **State Protections:**
    * Emails with status `sent` are **read-only** and cannot be edited (`PATCH` returns `409 Conflict`) or re-sent (`POST /transmissions` returns `409 Conflict`).
    * Emails with status `error` may be edited (`PATCH`) and re-sent (`POST /transmissions`).

---

## 6. Pagination, Filtering & Scalable Data

### 6.1 Pagination Strategy
* **Chosen Approach:** Page-based pagination using `page` (1-based index, default `1`) and `limit` (default `20`, max `100`).
* **Stable Ordering:** Results are ordered by `createdAt DESC`, with unique `id ASC` as a deterministic tie-breaker. Deterministic ordering guarantees that items do not shift between pages during concurrent insertions.
* **Response Wrapper:**
  ```json
  {
    "items": [...],
    "page": 1,
    "limit": 20,
    "totalItems": 45,
    "totalPages": 3
  }
  ```

### 6.2 Filtering
* Supported filters: `status` (`draft`, `sent`, `error`) and `source` (`intern`, `extern`).
* Filters apply directly to query execution prior to pagination slicing.

---

## 7. Error Handling & HTTP Status Codes

### 7.1 Idempotency of DELETE Requests (Theory)
* **Question:** Must repeated `DELETE` requests return identical status codes to be idempotent?
* **Answer:** **No.** Idempotency applies to the **server state**, not the returned status code.
    * First Request: `DELETE /emails/42` -> Server deletes email -> Returns `204 No Content`.
    * Second Request: `DELETE /emails/42` -> Resource already gone -> Server state unchanged -> Returns `404 Not Found`.
    * Because the final server state (email 42 does not exist) is identical after 1 or 2 calls, the operation is idempotent.

### 7.2 Status Code Differentiation
* `400 Bad Request`: Malformed syntax (invalid JSON format, unparseable UUID parameter).
* `404 Not Found`: Target email or attachment URI does not exist or does not belong to the user.
* `405 Method Not Allowed`: HTTP method not supported on endpoint (e.g., `PUT /emails`).
* `409 Conflict`: State rule violation (e.g., attempting to modify or re-send a `sent` email).
* `415 Unsupported Media Type`: Invalid request header (e.g., uploading attachment without `multipart/form-data`).
* `422 Unprocessable Entity`: Well-formed request breaking domain rules (e.g., subject/content empty, missing all recipients).

---

## 8. Attachment Upload and Download

### 8.1 Path Design & Separation
* Upload: `POST /emails/{emailId}/attachments` (Media type: `multipart/form-data`)
* List Metadata: `GET /emails/{emailId}/attachments`
* Get Single Metadata: `GET /emails/{emailId}/attachments/{attachmentId}`
* Download Binary: `GET /emails/{emailId}/attachments/{attachmentId}/content`
* Delete Attachment: `DELETE /emails/{emailId}/attachments/{attachmentId}`

**Separation:** Metadata endpoints return JSON (`application/json`) describing file properties (`filename`, `mimeType`, `size`). Binary endpoints stream raw byte content (`application/octet-stream` or actual file MIME type) with a `Content-Disposition: attachment; filename="..."` header.

### 8.2 Security & Storage Rules
* Server storage paths (e.g., `/var/uploads/emails/89234.dat`) are strictly hidden from clients.
* Attachments can only be added to or deleted from emails in `draft` or `error` status.
* Deleting an email (`DELETE /emails/{id}`) permanently and recursively purges all linked binary files and metadata.