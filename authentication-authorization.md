### _1. Core Definitions_

- _Authentication (The "Who"):_ The mechanism used to assign an identity to a subject. It answers the question, _"Who are you in a given context?"_.
- _Authorization (The "What"):_ The process of determining a user's permissions and capabilities. It answers the question, _"What can you do in this context?"_.

### _2. The Evolution of Authentication_

Understanding how authentication evolved helps explain modern systems:

- _Pre-Industrial (Implicit Trust):_ Relied on human contextual trust, like handshakes or a village elder vouching for someone.
- _Medieval Era (Explicit Authentication):_ Used wax seals as physical tokens of identity based on _possession_. This era saw the first "bypass attacks" via forgery.
- _Industrial Revolution:_ Telegraph operators used pre-agreed passphrases, shifting the paradigm to _something you know_.
- _1960s (Mainframes):_ MIT introduced passwords for multi-user systems. Initially stored in plain text, an incident where the password file was printed led to the invention of _hashing_ (transforming passwords into secure, irreversible, fixed-length strings).
- _1970s:_ The invention of Diffie-Hellman introduced asymmetric cryptography, forming the backbone of modern Public Key Infrastructure (PKI) and early ticket-based authentication (Kerberos).
- _1990s (MFA):_ To combat brute force attacks, Multi-Factor Authentication (MFA) was introduced, combining three principles: something you know (password), something you have (OTP), and something you are (biometrics).
- _Future Trends:_ Post-Quantum Cryptography (to secure data against quantum computers), decentralized identity (blockchain), and behavioral biometrics.

### _3. Three Core Components of Modern Authentication_

#### _A. Sessions_

- _The Problem:_ HTTP is fundamentally a stateless protocol, meaning it remembers nothing about past requests. Modern web apps (like e-commerce sites) need stateful memory.
- _The Solution:_ The server creates a unique _Session ID_ upon login, storing the user's data alongside this ID in a persistent, fast in-memory store like Redis. The Session ID is sent to the client and included in all future requests, giving the server memory.

#### _B. JWTs (JSON Web Tokens)_

- _The Problem:_ As apps scaled globally, storing and synchronizing millions of sessions across distributed servers caused latency and high storage costs.
- _The Solution:_ JWTs are a _stateless_ mechanism that offloads storage from the server.
- _Structure:_ They contain three base64-encoded parts:
  1.  _Header:_ Metadata like the signing algorithm.
  2.  _Payload:_ Contains "claims" such as the user ID (`sub`), issued at time (`iat`), and roles.
  3.  _Signature:_ Verified using a secret key held only by the server to ensure the token hasn't been tampered with.

#### _C. Cookies_

- _Definition:_ A mechanism allowing servers to store small pieces of information directly in the user's browser.
- _Workflow:_ The server sets an `HTTP-only` cookie (so JavaScript cannot access it) containing the Auth Token. The browser then automatically attaches this cookie to every subsequent request sent to that specific server.

### _4. Major Types of Authentication_

Backend engineers typically use four main workflows:

1.  _Stateful Authentication:_
    - _Workflow:_ User logs in $\rightarrow$ Server stores session in Redis $\rightarrow$ Server sends Session ID via cookie $\rightarrow$ Client sends cookie on next request $\rightarrow$ Server looks up ID in Redis.
    - _Pros/Cons:_ Offers centralized control and easy token revocation, but has limited scalability and higher operational complexity. Ideal for standard web applications.
2.  _Stateless Authentication:_
    - _Workflow:_ User logs in $\rightarrow$ Server signs a JWT with a secret key $\rightarrow$ Client sends JWT in the `Authorization` header on next request $\rightarrow$ Server cryptographically verifies the token without a database lookup.
    - _Pros/Cons:_ Highly scalable and portable, but revoking access before the token expires is extremely complex without forcing all users to log out by changing the secret key.
3.  _API Key-Based Authentication:_
    - _Purpose:_ Designed for programmatic, _machine-to-machine_ communication (e.g., your backend server requesting data from OpenAI's server).
    - _Workflow:_ A user generates a cryptographically random string via a UI. Their server then includes this key in automated requests to bypass human visual interactions (like login forms).
4.  _OAuth 2.0 & OpenID Connect (OIDC):_
    - _The Delegation Problem:_ Users historically shared passwords so one app could access resources on another (e.g., a travel app scanning Gmail). This was a massive security risk and made revocation impossible.
    - _OAuth 2.0:_ Solved this by allowing a client app to request a specific _token_ with limited permissions (e.g., read-only access to contacts) from an Authorization Server. This handles _authorization_.
    - _OpenID Connect (OIDC):_ Because OAuth didn't verify identity*, OIDC was built on top. It introduced the \*\*ID Token* (a JWT), which securely shares user profile data (like email and name). This powers the "Sign in with Google" features we use today.

### _5. Authorization: Role-Based Access Control (RBAC)_

- _The Concept:_ Not all users should have the same capabilities (e.g., standard users vs. admins accessing a "dead zone" of deleted files).
- _How RBAC Works:_ Users are assigned specific roles (User, Admin, Moderator). Each role maps to strict resource permissions (Read, Write, Delete).
- _Execution:_ During the request cycle, the server deduces the user's role from their token. If an unauthorized user attempts an action, the server rejects it with a `403 Forbidden` status.

### _6. Critical Security Best Practices_

- _Use Generic Error Messages:_ Never respond with "User not found" or "Incorrect password." Attackers use these friendly messages to deduce valid usernames and increase their attack surface. Always use a generic response like _"Authentication failed"_.
- _Prevent Timing Attacks:_ Attackers can measure the time it takes for a server to respond. If a username is invalid, the server fails instantly. If the username is valid but the password is wrong, the server takes longer because it has to run the heavy password-hashing algorithm.
  - Solution: Backend engineers must equalize response times using _constant-time operations_ or by _simulating a fake response delay_ (e.g., a standard 200ms delay) so attackers cannot measure the difference.
