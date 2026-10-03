# 14. Testing Express applications

[Back to notes index](../README.md)

| Previous | Notes index | Next |
| --- | --- | --- |
| [Previous: Database integration and asynchronous work](./13-database-integration-and-async-work.md) | [Notes index](../README.md) | [Next: Configuration, logging, and debugging](./15-configuration-logging-and-debugging.md) |

## Test observable behavior

A useful test checks what a caller can observe: the response status, headers, body, and whether the requested operation happened. Test success paths and failure paths, including invalid input, missing resources, denied access, and unexpected dependency errors.

Keep tests independent. Each test should arrange its own data, make a request, and assert the result. A test should not depend on another test running first or on records left in a shared development database.

## Use Node's built-in test runner

Node includes a test runner and assertion library, so a small project can start without choosing a separate test framework. Add a test command:

~~~json
{
  "scripts": {
    "test": "node --test"
  }
}
~~~

Node discovers files such as `*.test.js` and `*.test.mjs` when running `node --test`. With an ESM project, use `.js` when `package.json` has `"type": "module"` or use the `.mjs` extension.

## Make the Express app importable

Export a function that creates the app, and keep `listen()` in a separate entry file. This lets a test pass the app directly to Supertest without binding a fixed port:

~~~js
import express from "express";

export function createApp({ books }) {
  const app = express();
  app.use(express.json({ limit: "10kb" }));

  app.get("/api/books/:bookId", async (request, response) => {
    const book = await books.findById(request.params.bookId);

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

  app.post("/api/books", async (request, response) => {
    const { title } = request.body ?? {};

    if (typeof title !== "string" || title.trim() === "") {
      return response.status(422).json({
        error: {
          code: "INVALID_TITLE",
          message: "Title must be a non-empty string."
        }
      });
    }

    const book = await books.create({ title: title.trim() });

    response
      .location(`/api/books/${book.id}`)
      .status(201)
      .json({ data: book });
  });

  app.use((error, request, response, next) => {
    if (response.headersSent) {
      return next(error);
    }

    const status = Number.isInteger(error.status) &&
      error.status >= 400 &&
      error.status < 500
      ? error.status
      : 500;

    response.status(status).json({
      error: {
        code: status === 500 ? "INTERNAL_ERROR" : "REQUEST_FAILED",
        message: status === 500
          ? "An unexpected error occurred."
          : error.message
      }
    });
  });

  return app;
}
~~~

A small entry file can provide the real repository and open the port:

~~~js
import { createApp } from "./app.js";
import { bookRepository } from "./repositories/book-repository.js";

const app = createApp({ books: bookRepository });
const port = Number(process.env.PORT ?? 3000);

app.listen(port, () => {
  console.log(`Server listening on port ${port}`);
});
~~~

The app receives its data dependency as an argument. Tests can pass a small in-memory implementation, while the real entry file passes the database repository. This is dependency injection in a simple form.

## Send HTTP requests with Supertest

Install Supertest as a development dependency:

~~~sh
npm install --save-dev supertest
~~~

Create `test/books.test.js`:

~~~js
import assert from "node:assert/strict";
import { test } from "node:test";
import request from "supertest";
import { createApp } from "../src/app.js";

function createBookStore(initialBooks = []) {
  const books = new Map(initialBooks.map((book) => [book.id, book]));
  let nextId = books.size + 1;

  return {
    async findById(id) {
      return books.get(id) ?? null;
    },

    async create(input) {
      const book = {
        id: String(nextId++),
        title: input.title
      };

      books.set(book.id, book);
      return book;
    }
  };
}

test("GET a book returns its data", async () => {
  const store = createBookStore([
    { id: "1", title: "Clean Code" }
  ]);
  const app = createApp({ books: store });

  const response = await request(app)
    .get("/api/books/1")
    .expect("Content-Type", /json/)
    .expect(200);

  assert.deepEqual(response.body, {
    data: {
      id: "1",
      title: "Clean Code"
    }
  });
});

test("GET a missing book returns 404", async () => {
  const app = createApp({ books: createBookStore() });

  const response = await request(app)
    .get("/api/books/404")
    .expect(404);

  assert.equal(response.body.error.code, "BOOK_NOT_FOUND");
});

test("POST a book returns 201 and its Location", async () => {
  const app = createApp({ books: createBookStore() });

  const response = await request(app)
    .post("/api/books")
    .send({ title: "Working Effectively with Legacy Code" })
    .expect("Location", "/api/books/1")
    .expect(201);

  assert.equal(
    response.body.data.title,
    "Working Effectively with Legacy Code"
  );
});

test("POST rejects an empty title", async () => {
  const app = createApp({ books: createBookStore() });

  const response = await request(app)
    .post("/api/books")
    .send({ title: "   " })
    .expect(422);

  assert.equal(response.body.error.code, "INVALID_TITLE");
});

test("database errors do not leak to the response", async () => {
  const store = {
    async findById() {
      throw new Error("private database connection detail");
    },
    async create() {
      throw new Error("private database connection detail");
    }
  };
  const app = createApp({ books: store });

  const response = await request(app)
    .get("/api/books/1")
    .expect(500);

  assert.equal(response.body.error.code, "INTERNAL_ERROR");
  assert.doesNotMatch(
    JSON.stringify(response.body),
    /private database connection detail/
  );
});
~~~

Supertest sends a real HTTP request to the app using a temporary local server and resolves with the response. The test runner waits for the returned promise, so `await` each request.

Run the suite:

~~~sh
npm test
~~~

If a test fails, read the first failing assertion and response. Fix the behavior or the test expectation based on the API contract. Do not weaken a useful assertion only to make the test pass.

## Test middleware and access rules

Test authentication and authorization through requests, not only by calling middleware functions directly. A direct unit test can check the middleware's branch, but a request test also checks that the middleware is mounted on the right route and stops the request when access is denied.

Useful cases include:

- A request without a session receives `401`.
- A signed-in user without a required role receives `403`.
- A user cannot read or edit another user's record.
- An administrator can perform the permitted operation.
- A successful sign-in rotates the session and sets cookie flags.
- A sign-out invalidates the session.

Do not use a real production account or production database in a test. Use isolated fixtures and test-only secrets.

## Separate unit, request, and database tests

| Test type | What it checks | Example |
| --- | --- | --- |
| Unit | One function without HTTP or a database | Validate a title |
| Request or integration | Express route, middleware, status, headers, and JSON | `POST /api/books` returns `201` |
| Database integration | Real SQL against an isolated test database | A unique constraint rejects a duplicate |
| End-to-end | Several running components through an actual client environment | Sign in, create a record, then see it in the UI |

Mock a repository for most route tests so they run quickly and do not require a database server. Keep separate database tests for SQL, migrations, constraints, and transaction behavior.

## Keep database tests isolated

Use a dedicated test database or a disposable database instance. Apply the same schema migrations before a suite, insert known fixtures, and clean up after each test or suite. Never point a test command at production data.

Database tests should verify behavior that a mocked repository cannot prove, such as a unique constraint, a foreign key, a rollback, or the actual SQL column mapping. Do not use a test database that multiple developers or jobs can modify without coordination.

## Make tests deterministic

A deterministic test produces the same result each time under the same conditions. To improve repeatability:

- Use fixed input data and an isolated store.
- Avoid depending on the current time, random values, or network services.
- Inject a clock or external service when the test needs to control it.
- Await every asynchronous operation and close resources after use.
- Avoid sharing mutable fixtures between tests that can run concurrently.
- Check responses and side effects, not just whether a function was called.

If a test uses a live database pool, close it after the test run with `await pool.end()`. Do not leave timers or servers running after a suite completes.

## Main references

- [Node.js test runner](https://nodejs.org/api/test.html)
- [Node.js assert](https://nodejs.org/api/assert.html)
- [Supertest](https://github.com/forwardemail/supertest)
- [Express error handling](https://expressjs.com/en/guide/error-handling.html)
- [Express migration to version 5](https://expressjs.com/en/guide/migrating-5.html)