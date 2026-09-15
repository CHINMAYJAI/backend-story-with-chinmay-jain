### 1. Fundamental Concepts

- _Persistence:_ The core purpose of a database is to ensure data survives after the program stops running.
- _Disk vs. RAM:_
  - _RAM (Primary Memory):_ Very fast but volatile and expensive. Used for caching (e.g., Redis).
  - _Disk (Secondary Memory):_ Slower but cheaper and persistent. Databases (Postgres, Mongo) store data here to handle large capacities (terabytes) reliably.
- _Why not Text Files?_ Storing data in simple text files causes issues with parsing speed, lack of structure/schema, and _concurrency_ (handling multiple users trying to update the same data simultaneously).

### 2. Choosing a Database

- _Relational (SQL):_ Organizes data in tables/rows/columns with a strict schema. Best for data integrity (e.g., CRM systems).
- _Non-Relational (NoSQL):_ Flexible schema (documents). Good for dynamic, unstructured content (e.g., Content Management Systems).
- _Why Postgres?_
  - _Open Source & Standard Compliant:_ Free and sticks to SQL standards, making migration easier.
  - _JSON Support:_ The "killer feature." Postgres supports `JSON` and `JSONB` types, allowing you to store unstructured dynamic data (like MongoDB) inside a relational database.

### 3. Postgres Data Types & Best Practices

- _Integers:_
  - `Serial` / `BigSerial`: Auto-incrementing integers, usually used for IDs.
  - `SmallInt`, `Integer`, `BigInt`: Choose based on the size of the number you need to store.
- _Floats vs. Decimals:_
  - _Decimal/Numeric:_ Stores exact precision. _Always use this for money/prices_ to avoid calculation errors.
  - _Float/Real:_ Floating-point numbers. Faster, but can have slight accuracy discrepancies. Use for scientific calculations.
- _Strings:_
  - `Char(n)`: Fixed length (pads with spaces). Avoid using this.
  - `Varchar(n)`: Variable length with a limit. Note: `255` is often just a legacy habit from MySQL.
  - _`Text`:_ Variable length with no limit. _Recommendation:_ Prefer `Text` over `Varchar` in Postgres. It is just as performant and avoids future migration headaches if you need to increase string length.
- _Other Types:_
  - _`UUID`:_ Universally Unique Identifier. Safer and better for distributed systems than integer IDs.
  - _`JSONB`:_ Binary JSON. Always prefer this over standard `JSON` because Postgres can index it and query it faster.
  - _`Enum`:_ A custom type restricted to a specific set of values (e.g., Status: 'Pending', 'Completed'). Great for data integrity and self-documenting code.

### 4. Database Migrations

- _Definition:_ Migrations are version control for your database schema.
- _Workflow:_
  - _Up Migration:_ Applies changes (e.g., Create Table).
  - _Down Migration:_ Reverts changes (e.g., Drop Table). Used to "roll back" if an update breaks the system.
- _Why use them?_ They track changes over time and ensure every developer/server has the exact same database structure.

### 5. Data Modeling (Relationships)

- _Naming Conventions:_ Use _plural_ for table names (`users`, `projects`) and _snake_case_ for columns (`full_name`) because Postgres is case-insensitive.
- _One-to-One (User ↔ User Profile):_
  - Split into two tables to keep the main user table lightweight.
  - The Profile table uses the User ID as both its Primary Key and Foreign Key.
- _One-to-Many (Project ↔ Tasks):_
  - A Project has many Tasks.
  - The `tasks` table contains a `project_id` foreign key.
- _Many-to-Many (Users ↔ Projects):_
  - A User can have many Projects; a Project can have many Users.
  - Requires a _Linking Table_ (e.g., `project_members`).
  - Uses a _Composite Primary Key_ (a combination of `user_id` and `project_id`).

### 6. Constraints & Integrity

- _Primary Key:_ Implicitly `Unique` and `Not Null`.
- _Foreign Key:_ Ensures you cannot reference a record that doesn't exist.
- _Check Constraint:_ Enforces custom logic at the database level (e.g., `CHECK priority BETWEEN 1 AND 5`).
- _Referential Integrity (On Delete):_
  - `Restrict`: Prevents deleting a User if they still own Projects.
  - `Cascade`: If a Project is deleted, automatically delete all its Tasks.

### 7. Performance & Security

- _Parameterized Queries (SQL Injection):_
  - _Never_ concatenate strings to build a query.
  - Use placeholders (parameters). The database treats the input strictly as a string, preventing malicious code execution.
- _Indexes:_
  - Concept: Like a book index, it allows the DB to find a row without scanning every single item (Sequential Scan).
  - When to Index: Create indexes on columns used in _`WHERE`_ clauses, _`JOIN`_ conditions, or _`ORDER BY`_ sorting.
  - Trade-off: Indexes speed up Reads but slightly slow down Writes (Insert/Update) because the index must be maintained.
- _Triggers:_
  - Used to automate tasks. A common use case is a trigger that automatically updates the `updated_at` timestamp whenever a row is modified, so the application code doesn't have to do it manually.

### 8. API Query Design

- _Fetching Lists:_ Always support _Pagination_ (`LIMIT` and `OFFSET`) to avoid fetching too much data at once.
- _Filtering:_ Use `ILIKE` for case-insensitive pattern matching (e.g., searching for a name).
- _Joins:_ Use `LEFT JOIN` if you want to keep records from the main table even if the related table has no data (e.g., get Users even if they don't have a Profile).
