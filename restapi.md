### _1. What is REST? (History & Definition)_

- _The Origin:_ In the 1990s, Tim Berners-Lee invented the World Wide Web. However, as it grew exponentially, it faced a massive scalability crisis. In 2000, Roy Fielding proposed a standardized architectural style to solve this, which he named **REST (Representational State Transfer)**.
- _Breaking Down the Name:_
  - _Representational:_ Resources (data) on the web can have multiple representations based on who is asking. For an API client, it might be represented as JSON; for a browser, as HTML.
  - _State:_ The current condition or attributes of a specific resource (e.g., the items and total price currently sitting in a user's shopping cart).
  - _Transfer:_ Moving these representations between the client and server using standard HTTP methods.

### _2. The 6 Constraints of REST Architecture_

To be truly "RESTful" and achieve high scalability, a system must follow these constraints proposed by Roy Fielding:

1.  _Client-Server:_ A strict separation of concerns. The client handles the UI/UX, and the server handles data and business logic.
2.  _Uniform Interface:_ A standardized way for all web components to communicate, providing a consistent interface across services.
3.  _Layered System:_ Architecture is composed of hierarchical layers (e.g., adding load balancers or proxy servers in the middle). A layer can only see the immediate layer below it, improving security and scaling.
4.  _Cache:_ Server responses must explicitly label themselves as cacheable or non-cacheable to help reduce server load and improve speed.
5.  _Stateless:_ The server retains no memory of past requests. Every client request must contain all the information necessary to process it.
6.  _Code on Demand (Optional):_ Servers can temporarily extend client functionality by sending executable code (like JavaScript) to the client.

### _3. Anatomy of a RESTful Route_

When designing the URL for an API, you should follow standard, hierarchical naming conventions:

- _The Structure:_ A typical API URL follows this format: `https://api.example.com/v1/books`. This includes the secure scheme (`https`), an API subdomain (`api.`), API versioning (`v1`), and the path/resource (`books`).
- _Always Use Plural Nouns:_ The path segment identifying a resource should always be a plural noun (e.g., `/books` or `/organizations`), even if you are only fetching a single item via an ID (e.g., `/books/123`).
- _Formatting:_ Never use spaces or underscores in a URL. If a resource has a readable slug (like "Harry Potter"), convert it to lowercase and replace spaces with hyphens (e.g., `/books/harry-potter`).
- _Hierarchical Nesting:_ The forward slash `/` denotes a hierarchy. For instance, `/organizations/123/projects` semantically means "fetch the projects that belong specifically to organization 123".

### _4. HTTP Methods and "Idempotency"_

_Idempotency_ is a crucial concept: an operation is idempotent if performing it multiple times yields the exact same side-effects on the server as performing it just once.

- _GET (Idempotent):_ Used to fetch data. Calling it a thousand times won't alter the server's state.
- _PATCH vs. PUT (Idempotent):_ Both are used to update data, and applying the same update payload repeatedly doesn't change the final result.
  - _PATCH:_ Used when updating only partial fields (e.g., just changing a user's status).
  - _PUT:_ Used when completely replacing the entire resource representation.
- _DELETE (Idempotent):_ Used to remove a resource. The first call deletes it; subsequent calls will return a 404 Not Found error, but no further side-effects happen to the server state.
- _POST (Non-Idempotent):_ Used to create new resources. If you send the same POST request 10 times, you will create 10 distinct resources with 10 different database IDs.

### _5. Custom Actions (The Exception to CRUD)_

Sometimes, an action doesn't fit neatly into standard CRUD (Create, Read, Update, Delete) boxes. For example, "cloning" a project or "archiving" an organization triggers a complex web of background tasks that goes beyond a simple database update.

- _The Rule:_ When an action does not fall under standard methods, REST specifications dictate making it a _POST_ request.
- _Route Design:_ Append the custom action as a verb at the very end of a specific resource route. Example: `POST /projects/123/clone` or `POST /organizations/5/archive`.

### _6. Designing robust "List" APIs (GET)_

When returning a list of items, an API must be equipped to handle heavy data loads using three features:

1.  _Pagination:_ Sending thousands of records at once crashes clients and bottlenecks networks. Instead, send data in chunks (pages). A paginated response should include:
    - `data`: The array of objects for that specific page.
    - `total`: The absolute count of items in the database (useful for frontend UI).
    - `page`: The current page number being viewed.
    - `totalPages`: The maximum number of pages available.
2.  _Sorting:_ Clients use query parameters like `sortBy=name` and `sortOrder=ascending` to organize the returned data.
3.  _Filtering:_ Clients pass properties in the query parameters to filter results (e.g., `?status=active&name=org`).

### _7. Handling HTTP Status Codes Properly_

- `200 OK`: Standard success response for fetching, updating, or performing custom actions.
- `201 Created`: Explicitly returned when a POST request successfully creates a new database entity.
- `204 No Content`: Returned after a successful DELETE operation, indicating success but an intentionally empty response body.
- _The 404 (Not Found) Rule:_
  - Return a `404` only when a client requests a specific, single resource ID that does not exist (e.g., `GET /users/999`).
  - If a client requests a _List API_ (e.g., fetching all users with the name "Zack") and no matches are found, do _not_ return a 404. Instead, return a `200 OK` with an empty array `[]`.

### _8. Golden Rules & Best Practices for API Engineers_

- _Extract Nouns from UI:_ Before coding, look at the frontend wireframes (like Figma) to figure out what data the user interacts with. Identify the "nouns" (e.g., projects, users, tasks); these become your API's core resources.
- _Implement "Sane Defaults":_ Your server should not crash if a client forgets to send optional data. If a client doesn't pass a pagination limit, default it to `10`. If they don't provide a sort order, default to sorting by `created_at` in `descending` order. If a new organization is created without a status, default it to `active`.
- _Total Consistency:_ Be ruthlessly consistent. JSON payloads should always use `camelCase`. If an endpoint expects a field called `description`, do not abbreviate it to `desc` in another endpoint. Inconsistencies force other developers to guess, leading to bugs.
- _Interactive Documentation:_ Always use tools like Swagger/OpenAPI to generate an interactive playground. This acts as both documentation for front-end developers and a testing ground for you.
