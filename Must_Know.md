

# 1. What *is* the Web, really?

At its core, the web is:

> **A distributed system where clients ask for resources and servers respond over a network using standardized protocols.**

That’s it. Everything else—React, Node, APIs, databases—is layered on top of that.

Key ingredients:

* **Clients** (browsers, mobile apps, curl, Postman)
* **Servers** (machines running backend software)
* **Network** (internet)
* **Protocols** (rules for communication)
* **Resources** (data, files, actions)

---

# 2. Client vs Server (the mental model you must lock in)

## Client

A **client** is anything that:

* Initiates a request
* Wants something
* Doesn’t store the source of truth

Examples:

* Browser (Chrome, Firefox)
* Mobile app
* Frontend JS (fetch, axios)
* CLI tools (curl, wget)

Clients:

* Don’t own data
* Don’t decide business rules
* Only **ask**

---

## Server

A **server** is anything that:

* Listens for requests
* Processes logic
* Returns responses
* Owns data

Examples:

* Node.js (Express, Nest)
* Java (Spring)
* Python (Django, FastAPI)
* Go, Rust, PHP, etc.

Servers:

* Validate input
* Apply business logic
* Talk to databases
* Enforce security
* Decide **what the truth is**

---

## The relationship

```
Client  →  Request  →  Server
Client  ← Response ←  Server
```

Always:

* Client initiates
* Server responds
* Never the opposite (in classic HTTP)

---

# 3. Client–Server Architecture (zoomed out)

This is an **architecture**, meaning a way of organizing responsibility.

### Why it exists

* Separation of concerns
* Scalability
* Security
* Multiple clients (web, mobile, IoT) using same backend

### Typical modern setup

```
Browser
   ↓
Frontend (HTML/CSS/JS)
   ↓ HTTP
Backend API (Node / Java / Python)
   ↓
Database
```

Sometimes:

```
Client → API Gateway → Microservices → Databases
```

---

# 4. What happens when you type a URL in the browser?

Let’s take:

```
https://www.example.com/products/42
```

This triggers **a LOT** of steps.

---

# 5. URL vs URI (important distinction)

## URI (Uniform Resource Identifier)

A **generic identifier** for a resource.

Think:

> “A name or address that identifies *something*”

Examples:

* `https://example.com/users/1`
* `mailto:hello@example.com`
* `urn:isbn:0451450523`

---

## URL (Uniform Resource Locator)

A **type of URI** that tells:

* **Where** the resource is
* **How** to access it

URLs are actionable.

Example:

```
https://example.com:443/users?id=1
```

So:

> **All URLs are URIs, but not all URIs are URLs**

---

## URL anatomy (important for backend)

```
https://www.example.com:443/path/to/resource?x=1#section
```

| Part                | Meaning                     |
| ------------------- | --------------------------- |
| `https`             | Protocol (scheme)           |
| `www.example.com`   | Host                        |
| `443`               | Port                        |
| `/path/to/resource` | Path                        |
| `?x=1`              | Query string                |
| `#section`          | Fragment (client-side only) |

---

# 6. DNS – how domain becomes IP (this is huge)

Computers **don’t understand domain names**.
They understand **IP addresses**.

So DNS exists.

---

## DNS (Domain Name System)

Think of DNS as:

> **The phonebook of the internet**

It maps:

```
example.com → 93.184.216.34
```

---

## Step-by-step: domain → IP

When you enter `www.example.com`:

1. **Browser cache**

   * “Do I already know this IP?”
2. **OS cache**
3. **Router cache**
4. **ISP DNS resolver**
5. **Root DNS server**

   * “Who knows about `.com`?”
6. **TLD server (.com)**

   * “Who manages example.com?”
7. **Authoritative DNS server**

   * “Here’s the IP.”

Then:

* IP is returned
* Cached everywhere
* Browser now knows **where to send the request**

---

## Why DNS is hierarchical

* Scalability
* No single point of failure
* Delegation of responsibility

---

# 7. “How does it change back to domain name?”

Short answer: **It doesn’t.**

Important clarification:

* DNS resolves **domain → IP**
* Servers don’t magically convert IP → domain
* The domain is just a **human-friendly alias**

Servers only care about:

* IP
* Headers (like `Host`)

---

## Why servers still “know” the domain

Because HTTP includes:

```
Host: www.example.com
```

This allows:

* Virtual hosting
* Multiple domains on same IP

---

# 8. HTTP vs HTTPS

## HTTP

**HyperText Transfer Protocol**

* Plain text
* No encryption
* Vulnerable to:

  * Sniffing
  * MITM attacks
  * Data tampering

---

## HTTPS

**HTTP + TLS (SSL)**

Adds:

* Encryption
* Authentication
* Integrity

---

## What HTTPS guarantees

1. **Confidentiality**

   * Nobody can read the data
2. **Integrity**

   * Data isn’t altered
3. **Authenticity**

   * You’re talking to the real server

---

## TLS handshake (simplified but real)

1. Client says: “I want HTTPS”
2. Server sends **certificate**
3. Client verifies certificate (CA trust)
4. Public keys exchanged
5. Symmetric session key created
6. Encrypted communication starts

After that:

> HTTP behaves exactly the same — just encrypted

---

# 9. HTTP protocol fundamentals (this is backend gold)

HTTP is:

* Stateless
* Request–response
* Text-based (mostly)

---

## HTTP Request

```
GET /users/1 HTTP/1.1
Host: example.com
Authorization: Bearer token
```

Parts:

* Method
* Path
* Headers
* Body (optional)

---

## HTTP Methods (semantics matter)

| Method | Meaning        |
| ------ | -------------- |
| GET    | Read           |
| POST   | Create         |
| PUT    | Replace        |
| PATCH  | Partial update |
| DELETE | Remove         |

Backend devs **care deeply** about this.

---

## HTTP Response

```
HTTP/1.1 200 OK
Content-Type: application/json

{ "id": 1, "name": "Alice" }
```

Parts:

* Status code
* Headers
* Body

---

## Status codes (learn these well)

* 2xx → Success
* 3xx → Redirect
* 4xx → Client error
* 5xx → Server error

---

# 10. What is an API?

## API (Application Programming Interface)

On the web:

> **An API is a contract that exposes server functionality via HTTP**

APIs:

* Accept requests
* Return structured responses (usually JSON)
* Hide internal logic

---

## Example

```
GET /api/users/1
```

Response:

```json
{
  "id": 1,
  "email": "user@example.com"
}
```

Client doesn’t care:

* How data is stored
* What language is used
* What database exists

---

# 11. Resource (this is REST philosophy)

A **resource** is:

> **Any conceptual object you want to expose**

Examples:

* User
* Product
* Order
* Comment

In REST:

* Resources are **nouns**
* Identified by **URIs**

---

## Resource URI

```
/users
/users/1
/products/42
```

These identify **things**, not actions.

---

# 12. Sub-resources (very important)

A **sub-resource** is a resource that:

* Exists **in the context of another**
* Depends on a parent

Example:

```
/users/1/orders
/users/1/orders/99
```

Meaning:

* Orders **belong to** user 1

---

## Why this matters

* Clear data modeling
* Predictable APIs
* Authorization logic becomes clean

---

# 13. URI vs Resource vs Representation

This is deep but crucial.

* **Resource** → conceptual thing
* **URI** → identifier of that resource
* **Representation** → JSON, XML, HTML version of it

Example:

```
URI: /users/1
Resource: User #1
Representation: JSON
```

Same resource can have:

* JSON
* XML
* HTML

---

# 14. Putting it ALL together (full flow)

1. Client enters URL
2. DNS resolves domain → IP
3. TCP connection established
4. TLS handshake (HTTPS)
5. HTTP request sent
6. Server routes request
7. Backend logic executes
8. Database queried
9. Response built
10. Response sent
11. Browser renders or client processes

---

# 15. Because backend work is:

* Designing resources
* Modeling data
* Enforcing rules
* Handling failures
* Securing communication
* Scaling systems

Frameworks change.
**These fundamentals do not.**

---
