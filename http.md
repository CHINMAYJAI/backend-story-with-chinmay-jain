### _1. Core Principles of HTTP_

HTTP (Hypertext Transfer Protocol) is an application-layer protocol (Layer 7 in the OSI model) used by clients and servers to communicate. It is built on two fundamental ideas:

- _Statelessness:_ The server retains no memory of past interactions. Every request is entirely self-contained and must include all the necessary information (like authentication tokens or cookies) for the server to process it.
  - Benefits: This simplifies server architecture and improves scalability, because a single server doesn't need to keep track of user sessions, and a server crash won't destroy a client's state.
- _Client-Server Model:_ Communication is always initiated by the client (e.g., a web browser) to request resources or actions, and the server waits for these requests to process and respond.

### _2. Transport Protocol & HTTP Versions_

HTTP relies on a reliable, connection-based transport protocol, almost universally **TCP (Transmission Control Protocol)**. Over the years, HTTP has evolved to improve how these TCP connections are handled:

- _HTTP 1.0:_ Opened a new TCP connection for every single request and response, which was highly inefficient and slow.
- _HTTP 1.1:_ Introduced _persistent connections_ (`keep-alive`) as the default, allowing multiple requests to be sent over a single reused connection.
- _HTTP 2.0:_ Introduced multiplexing (multiple requests/responses concurrently on one connection), binary framing, header compression, and server push.
- _HTTP 3.0:_ Replaced TCP with QUIC (built over UDP) to establish faster connections and handle packet loss better, eliminating head-of-line blocking.

### _3. Anatomy of HTTP Messages_

Client-server communication happens via structured text messages.

- _Request Message (Client to Server):_ Contains a Request Method (e.g., GET/POST), the Resource URL, the HTTP version, Host domain, Headers, a blank line, and an optional Request Body.
- _Response Message (Server to Client):_ Contains the HTTP version, a Status Code (e.g., 200), a Status Value (e.g., OK), Headers, a blank line, and the Response Body.

### _4. HTTP Headers_

Headers are key-value pairs that act as metadata for the package being transmitted, allowing the system to be highly extensible and act as a "remote control" to dictate server behavior.

- _Request Headers:_ Sent by the client to provide context (e.g., `User-Agent` identifies the browser, `Authorization` sends credentials).
- _General Headers:_ Apply to both requests and responses (e.g., `Date`, `Connection`, `Cache-Control`).
- _Representation Headers:_ Describe the message body (e.g., `Content-Type` for media format like JSON/HTML, `Content-Length` for byte size, `Content-Encoding` for gzip compression).
- _Security Headers:_ Protect against attacks (e.g., `Strict-Transport-Security` forces HTTPS, `Content-Security-Policy` prevents cross-site scripting, `Set-Cookie` with HTTP-only flags).

### _5. HTTP Methods and Idempotency_

Methods define the semantic _intent_ of the client's request.

- _GET:_ Fetches data from the server without modifying anything.
- _POST:_ Submits new data to the server (includes a request body).
- _PATCH:_ Partially updates an existing resource.
- _PUT:_ Completely replaces an existing resource with the provided body.
- _DELETE:_ Removes a resource.
- _OPTIONS:_ Inquires about the server's capabilities (used heavily in CORS).

_Idempotency_ is a crucial concept here:

- _Idempotent Methods:_ Can be executed multiple times and yield the exact same result on the server state (e.g., GET, PUT, DELETE).
- _Non-Idempotent Methods:_ Running them multiple times creates different results (e.g., submitting a POST request twice creates two separate resources).

### _6. Cross-Origin Resource Sharing (CORS)_

Browsers enforce a Same-Origin Policy, blocking web apps from making requests to different domains (origins). CORS is a security mechanism to bypass this safely.

- _Simple Requests:_ (Usually GET or POST with standard headers/content types). The browser automatically adds an `Origin` header. If the server allows the request, it replies with the `Access-Control-Allow-Origin` header containing the client's domain (or a `*` wildcard). If missing, the browser blocks the response.
- _Pre-flight Requests:_ Triggered if a request uses a non-simple method (PUT/DELETE), requires authorization headers, or uses a `application/json` content type.
  - The browser first fires an _OPTIONS_ request asking the server if the route supports the intended method and headers.
  - The server replies with a `204 No Content` status, explicitly listing allowed origins, methods, headers, and a `max-age` to cache this configuration.
  - If successful, the browser then sends the actual, original request.

### _7. Standardized Status Codes_

Status codes are three-digit numbers that act as a universal language to indicate the outcome of a request.

- _1xx (Informational):_ Indicates headers received; client can proceed (e.g., `100 Continue` for large uploads).
- _2xx (Success):_
  - `200 OK`: Successful operation.
  - `201 Created`: Usually follows a POST request.
  - `204 No Content`: Successful, but no body to return (used in OPTIONS or DELETE).
- _3xx (Redirection):_
  - `301 Moved Permanently`: The resource has a new URL.
  - `302 Found/Temporary Redirect`: Temporarily forward to a new route.
  - `304 Not Modified`: Tells the client to use its locally cached version.
- _4xx (Client Errors):_
  - `400 Bad Request`: Invalid data format sent by client.
  - `401 Unauthorized`: Missing or invalid authentication token.
  - `403 Forbidden`: Authenticated, but lacks necessary permissions.
  - `404 Not Found`: Incorrect URL or deleted resource.
  - `405 Method Not Allowed`: Using the wrong method for a route.
  - `409 Conflict`: Business logic violation (e.g., duplicate username).
  - `429 Too Many Requests`: Client has hit rate limits.
- _5xx (Server Errors):_
  - `500 Internal Server Error`: An unhandled exception crashed the server.
  - `501 Not Implemented`: Feature not yet supported.
  - `502 Bad Gateway` / `504 Gateway Timeout`: Issues originating from proxies or load balancers failing to reach upstream servers.
  - `503 Service Unavailable`: Server down or under maintenance.

### _8. HTTP Caching_

Caching reuses previously downloaded responses to save bandwidth and load times.

- When a client first fetches a resource, the server responds with the payload alongside three headers: `Cache-Control` (sets max duration), `ETag` (a unique hash of the payload), and `Last-Modified`.
- On subsequent requests, the client sends conditional headers: `If-None-Match` (carrying the ETag) or `If-Modified-Since`.
- If the data on the server hasn't changed, the server saves bandwidth by sending an empty `304 Not Modified` response, instructing the browser to use its cached copy. If it has changed, it sends a `200 OK` with the new data and a new ETag.

### _9. Content Negotiation and Compression_

Clients and servers can negotiate the best format to exchange data.

- The client sends preferences via `Accept` (e.g., `application/json` vs `application/xml`), `Accept-Language` (e.g., `en` vs `es`), and `Accept-Encoding` (e.g., `gzip`).
- The server responds with the appropriate format.
- _Compression:_ By negotiating an encoding like `gzip`, a server can drastically compress text responses (e.g., shrinking a 26MB JSON payload down to 3.8MB) to save massive amounts of network bandwidth.

### _10. Handling Large Data Transfers_

- _Large Client Uploads (Images/Video):_ Standard JSON is terrible for binary data. Instead, clients use a `multipart/form-data` request. This breaks the file into chunks separated by a unique string delimiter defined in the `boundary` header.
- _Large Server Downloads:_ To prevent timing out, the server streams the file in chunks using `Content-Type: text/event-stream` and `Connection: keep-alive`. The browser continually appends these chunks until the transfer finishes.

### _11. Security (SSL/TLS & HTTPS)_

- _TLS (Transport Layer Security):_ The modern, secure replacement for the outdated SSL protocol.
- It encrypts data in transit to prevent interception (eavesdropping) or tampering, utilizing certificates to verify the server's identity.
- _HTTPS:_ Simply the standard HTTP protocol wrapped inside a secure TLS connection.
