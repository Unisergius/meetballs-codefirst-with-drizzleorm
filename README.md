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

**Example** – A simple `Todo` table defined directly in TypeScript with Drizzle ORM:

```ts
import { sqliteTable, int, text } from "drizzle-orm/sqlite-core";

export const todos = sqliteTable("todos", {
  id:    int().primaryKey({ autoIncrement: true }),
  title: text().notNull(),
  done:  int({mode: "boolean"}).default(false) // this is specific to sqlite-core version
});
```

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
* SQLITE (simple and easy, database in a file)
* [Documentation from Drizzle ORM](https://orm.drizzle.team/docs/get-started/sqlite-new)

### 1 - [Create app](https://orm.drizzle.team/docs/get-started/sqlite-new)

install drizzle-orm and tsx

```sh
npm i drizzle-orm @libsql/client dotenv
npm i -D drizzle-kit tsx
```

create .env file at root (./.env)

```env
DB_FILE_NAME=file:local.db
```

Create a src folder at root, then create an index.ts file in it.
(./src/index.ts)
In there we'll initialize the connection.

```ts
import 'dotenv/config';
import { drizzle } from 'drizzle-orm/libsql';

const db = drizzle(process.env.DB_FILE_NAME!);
```

### Schema file - your database blueprint in code

Separated from our app files, we'll have a db folder with the schema.
With it we have our single source of truth that defines what the database should look like.

Let's try this. We want to create yet again a todo list.
Because todo lists are not boring at all and the world needs more of them.

1. Create a ./src/db/schema.ts file
2. in the file you created, put the following content in it

```ts
import { sqliteTable, int, text } from "drizzle-orm/sqlite-core";

export const todos = sqliteTable("todos", {
  id:    int().primaryKey({ autoIncrement: true }),
  title: text().notNull(),
  done:  int({mode: "boolean"}).default(false) // this is specific to sqlite-core version
});
```

[More info on drizzle-orm sqlite types](https://orm.drizzle.team/docs/column-types/sqlite)

**Talk with each other what this file is about. What are we really doing here?**

### Drizzle config file

We need a config file to point out where our schema is.

Create a ./drizzle.config.ts

```ts
import 'dotenv/config';
import { defineConfig } from 'drizzle-kit';

export default defineConfig({
  out: './drizzle',
  schema: './src/db/schema.ts',
  dialect: 'sqlite',
  dbCredentials: {
    url: process.env.DB_FILE_NAME!,
  },
});
```

### Now the magic of migrations comes up

We have the schema done.
We don't have the database yet created.
What do we need to setup our database then?

Simple. Let's do it first by separated steps, without magic:

1. Generate Migration files (basically drizzle plans out what the migration will do)
2. Apply migration files (execute said plans)

Step 1:

```sh
    npx drizzle-kit generate --name="createtable_todo"
```

**Check out the files created. What do they do? Commit them.**

Step 2:

```sh
    npx drizzle-kit migrate
```

**What happened now? If you have a database client app, check your sqlite contents. Is there any new table?**

> If using VSCODE you can install florian sqlite viewer extension. Then just simply double click local.db to see contents*

"TL;DR I don't like doing Step 1 and 2."
Ok then. Use **push** instead. It does both steps and quickly tests out your new schema changes.

Step 1 + 2:

```sh
    npx drizzle-kit push
```

### Setting up an users table

Now we need to create a table representing the owner of the task on todo table.

Creating a users table. let's go to the schema.ts again and add our users table.

```ts
import { sqliteTable, int, text } from "drizzle-orm/sqlite-core";

export const todos = sqliteTable("todos", {
  id:    int().primaryKey({ autoIncrement: true }),
  title: text().notNull(),
  done:  int({mode: "boolean"}).default(false), 
  userId: int('user_id') // our field to reference the users table
        .notNull()
        .references(() => users.id), 
});

// Our users table
export const users = sqliteTable('users', {
    id: int().primaryKey().notNull(),
    username: text().notNull(),
    email: text().notNull(),
});
```

We added a new table users and altered our schema for todos table.

Let's create another migration file and update it.

Again

```sh
    npx drizzle-kit generate --name="createtable_users"
    npx drizzle-kit migrate
```

*What changed? Any new files? what do they do? How's the database?*

### Using the ORM to seed and query the database

Create a new file ./src/index.ts

```ts
import 'dotenv/config';
import { drizzle } from 'drizzle-orm/libsql';
import { eq } from 'drizzle-orm';
import { users } from './db/schema';
  
const db = drizzle(process.env.DB_FILE_NAME!);

async function main() {

  // $inferInsert is a type that infers the columns of the table. It's a
  // utility type that makes it easier to insert data into the table
  // without having to manually specify the columns. 
  // Since we're using typescript, it also complies to the table schema
  // and will show you an error if your data doesn't follow it's structure
  const user: typeof users.$inferInsert = {
    username: 'johnexample',
    email: 'john@example.com',
  };

  // in here we pick the user var and insert it in the db.
  await db.insert(users).values(user);
  console.log('New user created!')

  const usersData = await db.select().from(users);
  console.log('Getting all users from the database: ', usersData)

  await db
    .update(users)
    .set({
      email: "john.example@example.com",
    })
    .where(eq(users.email, user.email));
  console.log('User info updated!')


  const usersData2 = await db.select().from(users);
  console.log('Getting all users from the database again: ', usersData2)

  if (usersData2.length !== 0) {
    await db.delete(users).where(eq(users.email, usersData2.at(0)!.email));
    console.log('User deleted!')
  }


  const usersData3 = await db.select().from(users);
  console.log('Getting all users from the database yet again: ', usersData3)
}

main();
```

Now run it with

```sh
    npx tsx src/index.ts
```

*What happened? Discuss what it did with your meetballs fellowship.*

#### Users with todo tasks

Let's try more queries.

Create user then create two tasks. Let's do it on ./src/twotasks.ts

```ts
import 'dotenv/config';
import { drizzle } from 'drizzle-orm/libsql';
import { eq } from 'drizzle-orm';
import { todos, users } from './db/schema';

const db = drizzle(process.env.DB_FILE_NAME!);

async function main() {
  const user: typeof users.$inferInsert = {
    username: 'johnexample',
    email: 'john@example.com',
  };

  const insertedUserId = (await db.insert(users).values(user).returning({ id: users.id })).at(0)!;
  console.log('New user created!')

  // get user id from the user we just created
  
  const tasks: typeof todos.$inferInsert[] = [
    {
      title: 'Task 1',
      userId: insertedUserId.id,
    },
    {
      title: 'Task 2',
      userId: insertedUserId.id,
    },
  ];

  await db.insert(todos).values(tasks);

  console.log('Two tasks created!')


  const usersData = await db.select().from(users);
  console.log('Getting all users from the database: ', usersData)

  const tasksData = await db.select().from(todos);
  console.log('Getting all tasks from the database: ', tasksData)

  const tasksData2 = await db.select({
    todoId: todos.id,
    todoTitle: todos.title,
    todoDone: todos.done,
    username: users.username,
    userEmail: users.email
  })
  .from(todos)
  .leftJoin(users,  eq(todos.userId, users.id))
  .where(eq(todos.userId, insertedUserId.id));
  console.log(`Getting all tasks from the database where user is the user with id ${insertedUserId.id}: `, tasksData2)
}

main();
```

*What happened? What does each query do? What is the select doing? What does the left join do?*


## Conclusion

### Migrations. Why bother?

Imagine you're in a team? You have changes on the database. How does your teammate update their system according to your changes?

Your application is ever evolving. New columns on tables, new tables, tables removed. Foreign keys removed. New names. Your team needs to keep up with these updates.

Changing the database manually is asking for a hard time on your company. Without a disciplined process your local copy and your teammate’s copy drift apart, causing runtime errors and lost time.

Migrations are version‑controlled change scripts that keep every environment (laptop, CI, production) in the exact same state.

1) You keep a single source of truth. Every schema change produces a timestamped migration file committed to Git.
2) You can reproduce all migration steps on every machine with just a command.
3) With other migration systems you can even revert migration steps, similar to how you do reverts on git commit history
4) There's a safe collaboration between you and your team mates.
5) Migrations can be part of a CI/CD process, meaning a git push can be enough to trigger database changes. You can even test migrations with a throwaway db and see if the current build is succesful before you reach production.

**Rule of thumb**: treat migrations like any other code—review them, name them clearly, and never edit an applied migration; create a follow‑up migration instead.

## Note about DrizzleORM

Drizzle’s migration tool doesn’t currently include a built‑in “revert” or “rollback” command. Instead, you have a couple of options:

Generate a New Migration to Undo Changes:
Create a new migration file that reverses the changes you want to undo. For example, if you added a table or column, write the SQL (or use code‑first schema changes) to drop those changes.

Restore from Backup or a Snapshot:
If you’ve taken backups or are using version‑controlled snapshots of your DB, you can restore the previous state.

In practice, many teams using a code‑first workflow opt for the first approach—always creating a new migration that “undoes” the unwanted changes rather than reverting a previous migration automatically.
