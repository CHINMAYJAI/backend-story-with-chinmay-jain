### _1. The Core Security Mindset_

- _The Problem:_ No application is perfectly secure, but many vulnerabilities stem from a single question: _"Where did the developer make an assumption?"_. Developers often assume the "happy path"—that users will type correct inputs and only click intended buttons.
- _The Attacker's Mindset:_ Attackers do not follow the happy path. They purposefully break inputs and poke at boundaries. As a backend engineer, you must always ask: _"What could go wrong here in terms of security?"_.
- _Crossing Boundaries:_ Almost all vulnerabilities occur when data crosses a boundary (e.g., from the browser into a database query language, or from a user's markdown into HTML) and gets misinterpreted.

### _2. Injection Attacks (SQL & Command Injection)_

Injection attacks happen when an application confuses user-provided _data_ with executable **code**.

- _SQL Injection:_ If you build database queries by concatenating raw user strings, attackers can inject SQL commands.
  - The Exploit: If a user types `' OR 1=1 --` into an email login field, the first quote closes your string, `OR 1=1` creates a universally true statement, and `--` comments out the rest of your query. This can trick the database into returning all users or letting the attacker log in without a password. They can even inject `; DROP TABLE users;` to delete your database.
  - The Fix: _Parameterized Queries_ (Prepared Statements). This separates the query template from the data. The database treats whatever is inside the parameter slot purely as a string, making it impossible to execute as a command.
  - NoSQL Vulnerability: MongoDB and other NoSQL databases are also vulnerable if you pass raw JSON objects directly from the user to the query, as attackers can inject special operators like `$ne` (not equal).
- _Command (OS) Injection:_ If your server executes shell commands (e.g., using `ffmpeg` to process an uploaded image) and directly concatenates the user's filename, an attacker could input `; rm -rf /` to delete your server's files.
  - The Fix: Use programming language APIs that accept the command and the arguments as separate parameters, preventing the input from being processed by the shell interpreter.

### _3. Authentication & Password Storage_

Authentication verifies _who_ the user is. Implementing this manually in production is extremely complex, so using a third-party auth provider (like Clerk) is highly recommended. If you must store passwords:

- _Plain Text (Bad):_ Never store passwords in plain text. If your database is breached, attackers will steal them and try those credentials across other sites (Credential Stuffing).
- _Hashing (Better):_ Hashing is a one-way mathematical function that turns any string into a fixed-length output (e.g., taking `password123` and turning it into a random-looking string). You cannot reverse a hash.
  - The Flaw: Attackers use _Rainbow Tables_ (massive pre-computed lists of common passwords and their resulting hashes) to instantly decipher hashed passwords in a breached database.
- _Salting (Crucial):_ To defeat Rainbow Tables, generate a cryptographically random string (a *Salt\*\*) for each user. Concatenate the salt to their password *before hashing it. This ensures that even if two users have the same password, their hashes will look completely different.
- _Slow Hashing Functions:_ Modern GPU graphics cards can guess billions of fast hashes (like MD5 or SHA-256) per second. You must use _slow hashing functions_ explicitly designed for passwords, such as _Argon2id_ or **Bcrypt**. These include a "cost factor" that forces the calculation to take around 400 milliseconds, slowing down brute-force attacks so much that cracking passwords would take centuries.

### _4. Sessions vs. JWTs (Stateful vs. Stateless Auth)_

Once authenticated, the server must remember the user via a session mechanism.

- _Stateful Sessions (Recommended):_ The server generates a random 128-256 character **Session ID**, saves the user's data in a database/Redis alongside this ID, and sends the ID to the client via a Cookie.
  - Benefit: Immediate revocation. If an account is compromised, you instantly delete the session from the database, logging the attacker out.
- _Stateless Sessions (JWTs):_ The server packages the user's data (Claims) into a JSON Web Token (JWT), cryptographically signs it, and gives it to the client. The server stores nothing in the database.
  - The Flaw: JWTs cannot easily be revoked before they expire.
  - The Workaround: Use short-lived Access Tokens (e.g., 5 minutes) paired with longer-lived Refresh Tokens. If the token is stolen, the attacker only has access for a few minutes.
  - Warning: JWT payloads are just Base64 encoded, not encrypted. Anyone can decode and read them, so never store sensitive data inside a JWT.
- _Cookie Security Flags:_ To safely store Session IDs or JWTs in the browser, always set these cookie flags:
  1.  `HttpOnly`: Prevents malicious JavaScript from reading the cookie.
  2.  `Secure`: Ensures the cookie is only sent over encrypted HTTPS connections.
  3.  `SameSite (Strict or Lax)`: Prevents the cookie from being sent during Cross-Origin requests, mitigating CSRF attacks.

### _5. Rate Limiting_

To prevent attackers from brute-forcing passwords or crashing your server via massive request volumes, implement layered rate limiting:

1.  _Per IP Limit:_ Blocks specific IPs sending too many requests (flawed if attackers use botnets/VPNs).
2.  _Per Account Limit:_ Locks an account after multiple failed attempts (flawed if attackers try one password across thousands of accounts).
3.  _Global Limit:_ A hard system-wide cap on login attempts per minute, preventing distributed brute-force attacks.

### _6. Authorization (BOLA and BFLA)_

Authorization determines _what_ an authenticated user is allowed to do.

- _Horizontal Attacks (BOLA/IDOR):_ Broken Object Level Authorization occurs when a user alters an ID in an API request (e.g., fetching `/invoices/5`) to view another user's data.
  - The Fix: Do not rely purely on the routing layer for security. At the database layer, append `AND user_id = context.user_id` to your query to ensure the requesting user actually owns the resource. Avoid sequential IDs (101, 102) and use UUIDs to make guessing impossible.
  - Information Leakage: If a user requests an invoice they don't own, return a `404 Not Found` rather than a `403 Forbidden`. A 403 confirms the invoice exists, allowing attackers to enumerate system data for social engineering attacks.
- _Vertical Attacks (BFLA):_ Broken Function Level Authorization happens when a regular user figures out the URL for an admin endpoint (e.g., `/admin/invoices`) and calls it. Hiding the URL (security through obscurity) is not enough.
  - The Fix: Explicitly check the user's role (e.g., `role == admin`) in a middleware before executing sensitive functions.
- _Authorization Principles:_ Centralize your logic, adopt a _Default Deny_ policy (block everything unless explicitly allowed), and write automated test suites specifically testing edge-case boundary violations.

### _7. Frontend-Facing Vulnerabilities (XSS & CSRF)_

- _XSS (Cross-Site Scripting):_ Occurs when attackers inject malicious `<script>` tags into your site (e.g., via a markdown comment). When other users view the comment, the script runs in their browser, stealing their session cookies or redirecting them to phishing sites.
  - The Fix: Heavily sanitize user-provided markup at the validation layer before saving it to the database. Use a _Content Security Policy (CSP)_ header to instruct the browser to block inline scripts and only run scripts from trusted domains.
- _CSRF (Cross-Site Request Forgery):_ Tricking a user's browser into executing an action on your site while they are browsing an attacker's site (`evil.com`).
  - The Fix: Setting modern `SameSite` cookie flags to `Lax` or `Strict` entirely mitigates this issue for modern applications.

### _8. Misconfigurations and Defense in Depth_

- _Secrets Management:_ Never commit API keys, database URLs, or JWT secrets into your source code (GitHub). Always use environment variables (`.env`) or cloud secret managers like AWS Parameter Store. If you accidentally commit a secret, you must rotate (change) it immediately.
- _Production Debug Mode:_ Do not run your production server on `debug` log levels. Debug mode leaks stack traces, database queries, and sensitive user information to the terminal. Production should always be set to `info`.
- _Security Headers:_ Use modern backend middlewares to automatically inject HTTP security headers. For example, `X-Frame-Options` prevents other malicious websites from embedding your site inside an `iframe` to execute Clickjacking attacks.
- _Defense in Depth:_ No single layer is perfect. Chain your security: Input Validation $\rightarrow$ Parameterized DB Queries $\rightarrow$ Point-of-Access Authorization $\rightarrow$ Security Headers $\rightarrow$ Monitoring/Audit Logs.
