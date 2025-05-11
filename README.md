# APP with Code‑First Database Approach

## Some basic definitions first of all - read them if needed

### What is a database?

[Source from Wikipedia](https://en.wikipedia.org/wiki/Database) - tl;dr it's a persistent data store that is normally
depicted by tables that can be related or not really.
It's persistent, meaning that it's lifetime persist regardless of the apps retrieving data from it. They can be stored
as files or whole computer clusters.

### What is SQL?

SQL (Structured Query Language) is the international standard language for communicating
with relational databases. It lets you define, query, and manipulate data using concise, declarative statements.

SQL’s power lies in its declarative style: you describe what result you need, and the database engine figures out the how.


### What is a (Database) Schema?

A schema is the formal blueprint of how data is organised inside a database.
It describes tables, their columns (with data types and default values),
and the relationships and constraints (primary keys, foreign keys, indexes) that hold everything together.

Table view - Think of each table as a strongly‑typed Excel sheet (e.g., users, orders).

Column metadata - Every column has a name (email), a type (TEXT), and rules (UNIQUE, NOT NULL).

Relationships - Keys link rows across tables, enforcing referential integrity and enabling JOINs.

```sql

-- users table
CREATE TABLE users (
  id         INTEGER PRIMARY KEY AUTOINCREMENT,
  email      TEXT    NOT NULL UNIQUE,
  created_at DATETIME DEFAULT CURRENT_TIMESTAMP
);

-- posts table with a foreign‑key reference to users
CREATE TABLE posts (
  id      INTEGER PRIMARY KEY AUTOINCREMENT,
  user_id INTEGER NOT NULL REFERENCES users(id),
  title   TEXT    NOT NULL
);

```

Together, these statements form a schema with two tables and a relationship.

A well‑defined schema acts as a contract between the application and the database,
ensuring consistent data and predictable queries throughout the project lifecycle.

## Why we deal with databases **in** our code?

Embedding database logic in code lets us:

1. **Model the domain in one place THUS only one source of TRUTH** – classes/objects mirror real‑world concepts (e.g., `User`, `Order`).
2. **Guarantee schema correctness at compile‑time** – type‑safe queries prevent many runtime errors.
3. **Automate evolution** – version‑controlled migrations keep every environment (dev → prod) in sync.

> **Example** – A simple `Todo` table defined directly in TypeScript with Drizzle ORM:
>
> ```ts
> import { sqliteTable, int, bool, text } from "drizzle-orm/sqlite-core";
>
> export const todos = sqliteTable("todos", {
>   id:    int().primaryKey({ autoIncrement: true }),
>   title: text().notNull(),
>   done:  bool().default(false)
> });
> ```

## Database‑First Approach

* **Typical steps**

  1. A Database Admin (DBA) designs tables, columns, constraints.
  2. Developers run an introspection tool (e.g., `prisma db pull`, `sequelize-auto`).
  3. The ORM spits out model classes matching the live schema.
  4. If the schema changes, you repeat the pull/generation step.

* **Quick example** — reverse‑engineering with Prisma (DO NOT REPLICATE THIS, KARL):

  ```bash
  npx prisma db pull          # reads the current DB schema
  npx prisma generate         # recreates typed client based on it
  ```

Pros

* Clear separation between DBAs (schema owners) and developers.
* Works well when the database already exists and must remain the source of truth.

Cons

* Schema changes can break the code until you regenerate.
* Difficult to review & track changes in version control (DDL lives outside the repo).

## Code‑First Approach

A **code‑first** workflow flips the direction: the **source of truth is your code**,
and tooling converts it into SQL migrations.

* **Typical steps**

  1. Define tables/relations in code (TypeScript, C#, Python, etc.).
  2. Run a CLI that computes the diff against the current DB.
  3. The CLI emits timestamped migration files (`20250511213034_init.up.sql`).
  4. Apply migrations to dev/local DB → commit them to VCS → deploy.

* **Quick example** — Drizzle + SQLite:

  ```bash
  # 1 · define/modify src/db/schema.ts (see Todo example above)
  npx drizzle-kit generate --name=add_due_date   # 2 · create migration
  npx drizzle-kit migrate                        # 3 · apply to todo.db
  ```

Pros

* Single language for code *and* schema → easier onboarding.
* Migrations are code‑reviewable artefacts; DB history lives in Git.
* CI can validate migrations automatically.

Cons

* Requires discipline: one migration per change, avoid editing applied migrations.
* Some complex DB features (partitioning, advanced indexing) may need raw SQL.

### Choosing one

If you **own the whole stack** and value rapid iteration, code‑first is usually simpler. If you’re integrating with an **existing enterprise DB** or strict DBA governance, database‑first may fit better.

> 💡 **Hybrid strategy**: some teams start code‑first, then expose read‑only database views for external consumers—getting the best of both worlds.

## LET'S START OUR MINI WORKSHOP

### What we'll use

* Node.js and npm/pnpm whatever and NPX
* Next.js with Typescript.
* DrizzleORM
* [Documentation from Drizzle ORM](https://orm.drizzle.team/docs/get-started/sqlite-new)

### 1 - Create app

Let's create yet again a todo app with Next.js. Choose whatever you like. We'll just focus on database

```sh
npx create-next-app@latest --typescript
```

install drizzle-orm
```sh

