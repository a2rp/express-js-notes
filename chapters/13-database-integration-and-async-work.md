# 13. Database integration and asynchronous work

[Back to notes index](../README.md)

| Previous | Notes index | Next |
| --- | --- | --- |
| [Previous: Authentication and authorization](./12-authentication-and-authorization.md) | [Notes index](../README.md) | [Next: Testing Express applications](./14-testing-express-applications.md) |

## Keep database work outside route details

A route receives HTTP input and sends an HTTP response. Database code should handle persistence details such as SQL, table names, and connection management. Keeping these responsibilities separate makes it easier to change a query without rewriting every route.

A small application might use this structure:

~~~text
src/
|-- app.js
|-- db/
|   `-- pool.js
|-- repositories/
|   `-- book-repository.js
`-- routes/
    `-- books.js
~~~

For a small project, these can be ordinary JavaScript modules. A repository layer is a helpful boundary, not a requirement imposed by Express.

## Create one PostgreSQL connection pool

Install the `pg` package:

~~~sh
npm install pg
~~~

Create one pool for the application process. Reuse it instead of opening a new database connection for each HTTP request:

~~~js
import pg from "pg";

const { Pool } = pg;

if (!process.env.DATABASE_URL) {
  throw new Error("DATABASE_URL must be configured.");
}

export const pool = new Pool({
  connectionString: process.env.DATABASE_URL,
  max: 10,
  idleTimeoutMillis: 30000,
  connectionTimeoutMillis: 2000
});

pool.on("error", (error) => {
  console.error("Unexpected error from an idle database client:", error);
});
~~~

`DATABASE_URL` contains credentials, so provide it through the deployment environment or a local environment file excluded from Git. Never commit a real connection string or print it in logs.

A pool reuses a limited set of connections. Choose its size with the database's connection limit and the number of app instances in mind. Ten connections per process becomes one hundred connections if ten processes each create a pool of ten.

Use `pool.query()` for a single statement. It selects a client and returns it to the pool for you.

## Use parameterized queries

Pass request values separately from SQL text. PostgreSQL placeholders use `$1`, `$2`, and so on:

~~~js
import { pool } from "../db/pool.js";

export async function findBookById(bookId) {
  const result = await pool.query(
    "SELECT id, title, author FROM books WHERE id = $1",
    [bookId]
  );

  return result.rows[0] ?? null;
}
~~~

Do not build SQL by placing request text inside the query string:

~~~js
// Unsafe: user input becomes part of the SQL command.
const sql = `SELECT id FROM books WHERE title = '${request.query.title}'`;
~~~

Parameterized values remain data rather than SQL syntax. They do not parameterize table or column names. If a sort field must be dynamic, map a known public name to a fixed SQL fragment:

~~~js
const sortColumns = {
  title: "title",
  author: "author",
  created: "created_at"
};

const sortColumn = sortColumns[request.query.sort] ?? "created_at";
const result = await pool.query(
  `SELECT id, title, author FROM books ORDER BY ${sortColumn} LIMIT $1`,
  [limit]
);
~~~

Only the fixed values in `sortColumns` are inserted into the SQL text. Request values such as `limit` still use parameters after validation.

## Return data through a repository function

A repository function can hide query result details from the route:

~~~js
export async function listBooks({ limit, offset }) {
  const result = await pool.query(
    `SELECT id, title, author, created_at
     FROM books
     ORDER BY created_at DESC, id DESC
     LIMIT $1 OFFSET $2`,
    [limit, offset]
  );

  return result.rows;
}

export async function createBook({ title, author }) {
  const result = await pool.query(
    `INSERT INTO books (title, author)
     VALUES ($1, $2)
     RETURNING id, title, author, created_at`,
    [title, author]
  );

  return result.rows[0];
}
~~~

A stable `ORDER BY` matters for pagination. If two rows have the same creation time, including a unique ID gives them a consistent order.

Database constraints should also protect important rules. A `NOT NULL`, `UNIQUE`, foreign key, or check constraint protects the data even if a different route or process writes to the same database. Application validation gives a helpful response; database constraints preserve integrity.

## Call repository functions from Express

Keep HTTP status and response formatting in the route:

~~~js
import express from "express";
import { createBook, findBookById, listBooks } from "../repositories/book-repository.js";

const router = express.Router();

function validatePagination(request, response, next) {
  const rawLimit = request.query.limit ?? "20";
  const rawOffset = request.query.offset ?? "0";

  if (typeof rawLimit !== "string" || typeof rawOffset !== "string") {
    return response.status(400).json({
      error: {
        code: "INVALID_PAGINATION",
        message: "Limit and offset must each appear once."
      }
    });
  }

  const limit = Number(rawLimit);
  const offset = Number(rawOffset);

  if (!Number.isInteger(limit) || limit < 1 || limit > 100
      || !Number.isInteger(offset) || offset < 0) {
    return response.status(400).json({
      error: {
        code: "INVALID_PAGINATION",
        message: "Limit must be from 1 to 100 and offset must be zero or greater."
      }
    });
  }

  request.validatedPagination = { limit, offset };
  next();
}

router.get("/", validatePagination, async (request, response) => {
  const { limit, offset } = request.validatedPagination;

  const books = await listBooks({ limit, offset });
  response.json({ data: books });
});

router.get("/:bookId", async (request, response) => {
  const book = await findBookById(request.params.bookId);

  if (!book) {
    return response.status(404).json({
      error: {
        code: "BOOK_NOT_FOUND",
        message: "No book exists with that ID."
      }
    });
  }

  response.json({ data: book });
});

router.post("/", validateNewBook, async (request, response) => {
  const book = await createBook(request.validatedBook);

  response
    .location(`/api/books/${book.id}`)
    .status(201)
    .json({ data: book });
});

export default router;
~~~

The `validateNewBook` middleware should parse and limit `limit` and `offset` before this route passes them to PostgreSQL. Reuse the validation patterns from the previous chapter.

Express 5 forwards a rejected promise from an async route handler to error middleware. The common error handler can translate known database conditions into safe API errors. Do not send raw database error messages to clients.

## Use a transaction for work that must succeed together

A transaction groups related database statements. If one step fails, roll back the earlier changes so the database does not keep a partial operation.

This example creates an order and its line items. All statements use the same checked-out client:

~~~js
import { pool } from "../db/pool.js";

export async function createOrder(userId, items) {
  const client = await pool.connect();

  try {
    await client.query("BEGIN");

    const orderResult = await client.query(
      `INSERT INTO orders (user_id, status)
       VALUES ($1, 'pending')
       RETURNING id`,
      [userId]
    );

    const orderId = orderResult.rows[0].id;

    for (const item of items) {
      await client.query(
        `INSERT INTO order_items (order_id, product_id, quantity)
         VALUES ($1, $2, $3)`,
        [orderId, item.productId, item.quantity]
      );
    }

    await client.query("COMMIT");
    return { id: orderId, status: "pending" };
  } catch (error) {
    try {
      await client.query("ROLLBACK");
    } catch (rollbackError) {
      error.rollbackError = rollbackError;
    }

    throw error;
  } finally {
    client.release();
  }
}
~~~

A transaction is limited to its database client. Do not use `pool.query()` for some statements and a checked-out client for others inside the same transaction. Always return a checked-out client in `finally`, even if a statement fails.

Keep transactions as short as possible. Do not wait for a remote API or a human action while holding a database connection and transaction open.

## Await work and handle concurrency deliberately

Database calls are asynchronous. Await them before using their result or sending a response. An unawaited promise can reject after the request has already finished, and a route may send a response before its write is complete.

Independent reads may run together:

~~~js
const [bookResult, reviewResult] = await Promise.all([
  pool.query("SELECT id, title FROM books WHERE id = $1", [bookId]),
  pool.query(
    "SELECT id, rating, body FROM reviews WHERE book_id = $1",
    [bookId]
  )
]);
~~~

Use parallel work only when one operation does not depend on the result of another and both can safely run separately. If multiple changes must commit or roll back together, use a transaction instead.

Do not start CPU-heavy work such as large image processing inside a request handler. Node.js executes JavaScript on its event loop, so long synchronous work delays unrelated requests. Move lengthy work to a worker or background queue. If an API accepts a job that will finish later, return `202 Accepted` with a way to check its status.

## Handle database conflicts as application outcomes

A database can report a unique constraint violation after validation has passed. Another request may have created the same value between the check and the insert. The unique constraint is the final authority.

Translate known constraint failures into an appropriate API response, such as `409 Conflict`, without returning raw SQL, table names, or driver details. Do not treat every database error as a client error. Connection failures and unexpected SQL errors are server failures that should be logged and handled by centralized error middleware.

For operations that clients may retry, think about idempotency. A network can fail after a database commit but before the response reaches the client. A retry should not accidentally charge a card or create duplicate work. Use a transaction and a persisted idempotency key where the operation requires it.

## Close the pool during shutdown

Stop accepting new requests, let current requests finish, then close the database pool:

~~~js
const server = app.listen(port, () => {
  console.log(`Server listening on port ${port}`);
});

async function shutdown(signal) {
  console.log(`${signal} received; closing server.`);

  server.close(async (error) => {
    if (error) {
      console.error("Could not close the HTTP server:", error);
      process.exitCode = 1;
    }

    await pool.end();
  });
}

process.on("SIGTERM", () => shutdown("SIGTERM"));
process.on("SIGINT", () => shutdown("SIGINT"));
~~~

A real deployment should also set a shutdown timeout so the process can exit if a dependency never responds. Avoid calling `process.exit()` before pending logs, responses, and cleanup have completed.

## Main references

- [node-postgres connection pooling](https://node-postgres.com/features/pooling)
- [node-postgres parameterized queries](https://node-postgres.com/features/queries)
- [node-postgres transactions](https://node-postgres.com/features/transactions)
- [node-postgres project structure](https://node-postgres.com/guides/project-structure)
- [Express error handling](https://expressjs.com/en/guide/error-handling.html)
- [OWASP SQL Injection Prevention Cheat Sheet](https://cheatsheetseries.owasp.org/cheatsheets/SQL_Injection_Prevention_Cheat_Sheet.html)