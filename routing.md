### _1. What is Routing?_

- _The "What" vs. The "Where":_ In a backend system, HTTP methods (like GET, POST, DELETE) express the what or the intent of a request (e.g., fetching or adding data). Routing expresses the _where_—the specific resource or destination you want to apply that action to.
- _Definition:_ Routing is the process of mapping a combination of an HTTP method and a URL path to a specific server-side Handler (a set of instructions or business logic).
- _Uniqueness:_ The server concatenates the HTTP method and the route to form a unique key. For example, a `GET` request to `/api/books` and a `POST` request to `/api/books` will trigger completely different logic in the server without clashing.

### _2. Types of Routes_

There are two primary ways to structure a route path:

- _Static Routes:_ These are constant strings that do not contain any variable parameters. For example, `/api/books` will always stay consistent and point to the same general resource.
- _Dynamic Routes:_ These include variable slots within the URL that the server can extract as data. In most backend frameworks (like Node.js, Python, or Go), these are denoted by a colon, such as `/api/users/:id`. If a client requests `/api/users/123`, the server extracts "123" as the ID to fetch that specific user's data.

### _3. Path Parameters vs. Query Parameters_

When sending data through a URL, backend engineers use two distinct types of parameters:

- _Path Parameters (Route Parameters):_ These are the variables placed directly inside the route's path, right after a forward slash `/` (e.g., the `123` in `/api/users/123`). They are used to express **semantic meaning**, specifically identifying a unique resource.
- _Query Parameters:_ Because `GET` requests do not have a data body, query parameters are used to send key-value pairs of metadata to the server.
  - _Syntax:_ They are attached to the end of the route after a question mark `?` (e.g., `/api/search?query=some+value`).
  - _Use Cases:_ They are heavily used for _pagination_ (e.g., `page=2&limit=20`), filtering user-defined values, or determining sorting orders (ascending/descending).

### _4. Nested Routing_

Nested routing is a standard REST API practice used to express a hierarchy between different resources.

- _Semantic Hierarchy:_ By nesting paths, you create a highly readable, semantic expression of what data you want.
- _Example Workflow:_
  - `/api/users`: Fetches a list of all users.
  - `/api/users/123`: Goes one level deep to fetch a specific user.
  - `/api/users/123/posts`: Goes another level deep to fetch all posts created by user 123.
  - `/api/users/123/posts/456`: Fetches one highly specific post (ID 456) belonging to that specific user.

### _5. Route Versioning and Deprecation_

As applications grow, business requirements change, which might require you to completely alter the format of the data your API returns (e.g., switching the key `name` to `title`).

- _The Problem:_ If you change the response format on a live route, you will break the frontend application (like an iOS or React app) currently relying on it.
- _The Solution (Versioning):_ Engineers add version numbers to routes, such as `/api/v1/products` and `/api/v2/products`.
- _Deprecation:_ This allows the server to simultaneously support both the old and new data structures. It provides frontend engineers a safe window of time to migrate their code to `v2` before the backend team officially deprecates and removes `v1`.

### _6. Catch-All Routes_

- _Purpose:_ A catch-all route acts as a safety net for invalid requests.
- _How it Works:_ It is placed at the very end of the server's routing logic, often using a wildcard syntax like `/*`. If a request trickles down through all the previous route matching algorithms without finding a match, it hits the catch-all.
- _Benefit:_ Instead of the server defaulting to a broken or null response, the catch-all handler cleanly returns a user-friendly "Route Not Found" (404) message to the client.
