# 98. All code samples

[Back to notes index](../README.md)

| Previous | Notes index | Next |
| --- | --- | --- |
| [Previous: Production deployment and operations](./16-production-deployment-and-operations.md) | [Notes index](../README.md) | [Next: Complete questions and answers](./99-complete-q-and-a.md) |

## Chapter 01: Express.js and the Node.js HTTP layer

[Open chapter](./01-express-and-node-http.md)

### Sample 1: Start with Node.js HTTP

~~~js
import { createServer } from "node:http";

const server = createServer((request, response) => {
  if (request.method === "GET" && request.url === "/") {
    response.writeHead(200, { "content-type": "text/plain" });
    response.end("Hello from Node.js");
    return;
  }

  response.writeHead(404, { "content-type": "text/plain" });
  response.end("Not found");
});

server.listen(3000, () => {
  console.log("Server listening on port 3000");
});
~~~

### Sample 2: What Express adds

~~~js
import express from "express";

const app = express();

app.get("/", (request, response) => {
  response.status(200).send("Hello from Express");
});

app.listen(3000, () => {
  console.log("Express app listening on port 3000");
});
~~~

### Sample 3: Try the Express response

~~~sh
curl -i http://localhost:3000/
~~~

## Chapter 02: Project setup and the first app

[Open chapter](./02-project-setup-and-first-app.md)

### Sample 1: Check Node.js and npm

~~~sh
node --version
npm --version
~~~

### Sample 2: Create a project and install Express

~~~sh
mkdir express-study
cd express-study
npm init -y
npm install express@5
~~~

### Sample 3: Use JavaScript modules

~~~sh
npm pkg set type=module scripts.start="node src/server.js" scripts.dev="node --watch src/server.js"
~~~

### Sample 4: Create the first server

~~~js
import express from "express";

const app = express();
const port = Number(process.env.PORT ?? 3000);

app.get("/", (request, response) => {
  response.status(200).json({
    message: "Express is running"
  });
});

app.get("/health", (request, response) => {
  response.status(200).json({
    status: "ok"
  });
});

app.listen(port, () => {
  console.log(`Express is listening on port ${port}`);
});
~~~

### Sample 5: Create the first server

~~~sh
mkdir src
~~~

### Sample 6: Start the app and make a request

~~~sh
npm run dev
~~~

### Sample 7: Start the app and make a request

~~~powershell
Invoke-RestMethod -Uri http://localhost:3000/
Invoke-RestMethod -Uri http://localhost:3000/health
~~~

### Sample 8: Start the app and make a request

~~~sh
curl -i http://localhost:3000/
curl -i http://localhost:3000/health
~~~

### Sample 9: Start the app and make a request

~~~sh
npm start
~~~

### Sample 10: Keep local files out of Git

~~~gitignore
node_modules/
.env
~~~

## Chapter 03: Routing fundamentals

[Open chapter](./03-routing-fundamentals.md)

### Sample 1: What a route does

~~~js
app.get("/books", (request, response) => {
  response.status(200).json([
    { id: "bk-101", title: "A Small Express App" }
  ]);
});
~~~

### Sample 2: Match routes in a useful order

~~~js
app.get("/books/search", (request, response) => {
  response.json({ result: "Search results" });
});

app.get("/books/:bookId", (request, response) => {
  response.json({ result: "A book detail route" });
});
~~~

### Sample 3: Use HTTP methods to describe the action

~~~js
app.get("/books", (request, response) => {
  response.json([{ id: "bk-101", title: "A Small Express App" }]);
});

app.post("/books", (request, response) => {
  response.status(201).json({ id: "bk-102", title: "A New Book" });
});

app.put("/books/:bookId", (request, response) => {
  response.json({ id: request.params.bookId, replaced: true });
});

app.patch("/books/:bookId", (request, response) => {
  response.json({ id: request.params.bookId, updated: true });
});

app.delete("/books/:bookId", (request, response) => {
  response.status(204).end();
});
~~~

### Sample 4: Group handlers for one path

~~~js
app.route("/books")
  .get((request, response) => {
    response.json([{ id: "bk-101", title: "A Small Express App" }]);
  })
  .post((request, response) => {
    response.status(201).json({ id: "bk-102", title: "A New Book" });
  });
~~~

### Sample 5: Group handlers for one path

~~~js
app.route("/books/:bookId")
  .get((request, response) => {
    response.json({ id: request.params.bookId });
  })
  .patch((request, response) => {
    response.json({ id: request.params.bookId, updated: true });
  })
  .delete((request, response) => {
    response.status(204).end();
  });
~~~

### Sample 6: Return the right kind of response

~~~js
response.status(200).json({ items: [] });
response.status(201).json({ id: "bk-102" });
response.status(204).end();
response.status(404).json({ error: "Book not found" });
~~~

### Sample 7: Try the routes

~~~sh
curl -i http://localhost:3000/books
curl -i http://localhost:3000/books/search
curl -i http://localhost:3000/books/bk-101
~~~

## Chapter 04: Route parameters and query strings

[Open chapter](./04-route-parameters-and-query-strings.md)

### Sample 1: Read a route parameter

~~~js
app.get("/books/:bookId", (request, response) => {
  response.json({
    bookId: request.params.bookId
  });
});
~~~

### Sample 2: Read a route parameter

~~~js
app.get("/authors/:authorId/books/:bookId", (request, response) => {
  response.json({
    authorId: request.params.authorId,
    bookId: request.params.bookId
  });
});
~~~

### Sample 3: Use query strings for optional filters

~~~text
GET /books?q=express&page=2
~~~

### Sample 4: Use query strings for optional filters

~~~js
app.get("/books", (request, response) => {
  const rawSearch = request.query.q;
  const rawPage = request.query.page;

  if (rawSearch !== undefined && typeof rawSearch !== "string") {
    return response.status(400).json({ error: "q must be one text value" });
  }

  const page = rawPage === undefined ? 1 : Number(rawPage);
  if (!Number.isInteger(page) || page < 1) {
    return response.status(400).json({ error: "page must be a positive integer" });
  }

  const search = (rawSearch ?? "").trim().toLowerCase();
  const books = [
    { id: "bk-101", title: "A Small Express App" },
    { id: "bk-102", title: "Routing with Node.js" },
    { id: "bk-103", title: "Working with Middleware" }
  ];

  const matchingBooks = books.filter((book) =>
    book.title.toLowerCase().includes(search)
  );
  const pageSize = 2;
  const start = (page - 1) * pageSize;

  response.json({
    page,
    total: matchingBooks.length,
    items: matchingBooks.slice(start, start + pageSize)
  });
});
~~~

### Sample 5: Handle missing resources

~~~js
const books = [
  { id: "bk-101", title: "A Small Express App" },
  { id: "bk-102", title: "Routing with Node.js" }
];

app.get("/books/:bookId", (request, response) => {
  const book = books.find((item) => item.id === request.params.bookId);

  if (!book) {
    return response.status(404).json({ error: "Book not found" });
  }

  response.json(book);
});
~~~

### Sample 6: Use a named wildcard in Express 5

~~~js
app.get("/files/*filepath", (request, response) => {
  response.json({
    segments: request.params.filepath
  });
});
~~~

### Sample 7: Test the inputs

~~~sh
curl -i "http://localhost:3000/books?q=express&page=1"
curl -i "http://localhost:3000/books?q=express&q=node"
curl -i "http://localhost:3000/books/bk-101"
curl -i "http://localhost:3000/books/unknown"
curl -i "http://localhost:3000/files/images/logo.png"
~~~

## Chapter 05: Request bodies and built-in middleware

[Open chapter](./05-request-bodies-and-built-in-middleware.md)

### Sample 1: Middleware reads the request body

~~~js
import express from "express";

const app = express();

app.use(express.json({ limit: "32kb" }));
app.use(express.urlencoded({ extended: false, limit: "32kb" }));

app.post("/books", (request, response) => {
  const title = request.body?.title;

  if (typeof title !== "string" || title.trim() === "") {
    return response.status(400).json({
      error: "title is required"
    });
  }

  response.status(201).json({
    book: { title: title.trim() }
  });
});
~~~

### Sample 2: Send a JSON request

~~~powershell
$body = @{ title = "Learning Express" } | ConvertTo-Json
Invoke-RestMethod `
  -Uri http://localhost:3000/books `
  -Method Post `
  -ContentType "application/json" `
  -Body $body
~~~

### Sample 3: Send a JSON request

~~~sh
curl -i http://localhost:3000/books \
  -H "Content-Type: application/json" \
  -d '{"title":"Learning Express"}'
~~~

### Sample 4: Parse HTML form data

~~~html
<form method="post" action="/books">
  <label>
    Title
    <input name="title" required>
  </label>
  <button type="submit">Save book</button>
</form>
~~~

### Sample 5: Limit the amount of input

~~~js
app.use(express.json({ limit: "32kb" }));
~~~

## Chapter 06: Routers and modular apps

[Open chapter](./06-routers-and-modular-apps.md)

### Sample 1: Create a books router

~~~text
src/
|-- routes/
|   `-- books.js
`-- server.js
~~~

### Sample 2: Create a books router

~~~js
import express from "express";

const router = express.Router();

const books = [
  { id: "bk-101", title: "A Small Express App" },
  { id: "bk-102", title: "Routing with Node.js" }
];

router.get("/", (request, response) => {
  response.json(books);
});

router.get("/:bookId", (request, response) => {
  const book = books.find((item) => item.id === request.params.bookId);

  if (!book) {
    return response.status(404).json({ error: "Book not found" });
  }

  response.json(book);
});

export default router;
~~~

### Sample 3: Mount the router in the app

~~~js
import express from "express";
import booksRouter from "./routes/books.js";

const app = express();
const port = Number(process.env.PORT ?? 3000);

app.use("/api/books", booksRouter);

app.listen(port, () => {
  console.log(`Express is listening on port ${port}`);
});
~~~

### Sample 4: Mount more than one feature

~~~js
import authorsRouter from "./routes/authors.js";
import booksRouter from "./routes/books.js";

app.use("/api/authors", authorsRouter);
app.use("/api/books", booksRouter);
~~~

### Sample 5: Share parent parameters with a child router

~~~js
const booksRouter = express.Router({ mergeParams: true });

booksRouter.get("/:bookId", (request, response) => {
  response.json({
    authorId: request.params.authorId,
    bookId: request.params.bookId
  });
});

app.use("/authors/:authorId/books", booksRouter);
~~~

### Sample 6: Test the mounted paths

~~~sh
curl -i http://localhost:3000/api/books
curl -i http://localhost:3000/api/books/bk-101
curl -i http://localhost:3000/api/books/unknown
~~~

## Chapter 07: Middleware flow and custom middleware

[Open chapter](./07-middleware-flow-and-custom-middleware.md)

### Sample 1: Understand the middleware function

~~~js
function middleware(request, response, next) {
  // Read or update request and response state.
  next();
}
~~~

### Sample 2: Registration order defines the flow

~~~text
request
  -> request logger
  -> request body parser
  -> route-specific checks
  -> route handler
  -> response
~~~

### Sample 3: Registration order defines the flow

~~~js
app.use(requestLogger);
app.use(express.json());
app.use("/api/books", booksRouter);
~~~

### Sample 4: Registration order defines the flow

~~~js
app.use("/api", apiRequestLogger);
~~~

### Sample 5: Add a request logger

~~~js
function requestLogger(request, response, next) {
  const startedAt = Date.now();

  response.on("finish", () => {
    const elapsedMs = Date.now() - startedAt;
    console.log(
      `${request.method} ${request.originalUrl} ${response.statusCode} ${elapsedMs}ms`
    );
  });

  next();
}

app.use(requestLogger);
~~~

### Sample 6: Add a request identifier

~~~js
import { randomUUID } from "node:crypto";

function addRequestId(request, response, next) {
  request.requestId = randomUUID();
  response.setHeader("X-Request-Id", request.requestId);
  next();
}

app.use(addRequestId);
~~~

### Sample 7: Use route-specific middleware

~~~js
function requireAdmin(request, response, next) {
  if (!request.user?.isAdmin) {
    return response.status(403).json({ error: "Forbidden" });
  }

  next();
}

app.get("/admin/reports", requireAdmin, (request, response) => {
  response.json({ reports: [] });
});
~~~

### Sample 8: Avoid ending or advancing twice

~~~js
app.get("/books/:bookId", (request, response) => {
  const book = findBook(request.params.bookId);

  if (!book) {
    return response.status(404).json({ error: "Book not found" });
  }

  response.json(book);
});
~~~

## Chapter 08: Error handling

[Open chapter](./08-error-handling.md)

### Sample 1: Separate missing routes from application errors

~~~js
app.use((request, response) => {
  response.status(404).json({
    error: "Route not found"
  });
});
~~~

### Sample 2: Handle synchronous errors

~~~js
app.get("/reports", (request, response) => {
  throw new Error("Report service is unavailable");
});
~~~

### Sample 3: Handle synchronous errors

~~~js
app.use((error, request, response, next) => {
  if (response.headersSent) {
    return next(error);
  }

  const status =
    Number.isInteger(error.status) && error.status >= 400 && error.status < 600
      ? error.status
      : 500;

  if (status >= 500) {
    console.error(error);
  }

  const message =
    status < 500 ? error.message : "Internal server error";

  response.status(status).json({ error: message });
});
~~~

### Sample 4: Let Express 5 forward rejected promises

~~~js
app.get("/books/:bookId", async (request, response) => {
  const book = await bookStore.findById(request.params.bookId);

  if (!book) {
    return response.status(404).json({ error: "Book not found" });
  }

  response.json(book);
});
~~~

### Sample 5: Handle errors outside the returned Promise

~~~js
app.get("/delayed-report", (request, response, next) => {
  setTimeout(() => {
    try {
      throw new Error("Delayed report failed");
    } catch (error) {
      next(error);
    }
  }, 10);
});
~~~

### Sample 6: Create an error with an HTTP status

~~~js
function createHttpError(status, message) {
  const error = new Error(message);
  error.status = status;
  return error;
}

app.get("/books/:bookId", async (request, response) => {
  const book = await bookStore.findById(request.params.bookId);

  if (!book) {
    throw createHttpError(404, "Book not found");
  }

  response.json(book);
});
~~~

### Sample 7: Keep error middleware last

~~~js
app.use(requestLogger);
app.use(express.json());
app.use("/api/books", booksRouter);

// Handle requests that reached no route.
app.use(notFoundHandler);

// Handle errors passed with next(error) or rejected by a handler.
app.use(errorHandler);
~~~

## Chapter 09: Static files and view rendering

[Open chapter](./09-static-files-and-view-rendering.md)

### Sample 1: Serve static assets

~~~js
import path from "node:path";
import { fileURLToPath } from "node:url";

const currentDirectory = path.dirname(fileURLToPath(import.meta.url));
const publicDirectory = path.join(currentDirectory, "../public");

app.use("/assets", express.static(publicDirectory));
~~~

### Sample 2: Serve static assets

~~~text
project/
|-- public/
|   `-- styles.css
`-- src/
    `-- server.js
~~~

### Sample 3: Render a template as HTML

~~~sh
npm install pug
~~~

### Sample 4: Render a template as HTML

~~~js
app.set("views", path.join(currentDirectory, "../views"));
app.set("view engine", "pug");

app.get("/", (request, response) => {
  response.render("home", {
    title: "Express study notes",
    books: [
      { title: "Routing", summary: "Match a request to a handler." },
      { title: "Middleware", summary: "Compose work across a request." }
    ]
  });
});
~~~

### Sample 5: Render a template as HTML

~~~pug
doctype html
html(lang="en")
  head
    meta(charset="utf-8")
    meta(name="viewport" content="width=device-width, initial-scale=1")
    title= title
    link(rel="stylesheet" href="/assets/styles.css")
  body
    main
      h1= title
      ul
        each book in books
          li
            h2= book.title
            p= book.summary
~~~

### Sample 6: Try the page and asset

~~~sh
curl -i http://localhost:3000/
curl -i http://localhost:3000/assets/styles.css
~~~

## Chapter 10: REST API design and responses

[Open chapter](./10-rest-api-design-and-responses.md)

### Sample 1: Return a useful response when creating a resource

~~~js
app.post("/api/books", (request, response) => {
  const book = {
    id: String(nextBookId++),
    title: request.body.title
  };

  books.set(book.id, book);

  response
    .location(`/api/books/${book.id}`)
    .status(201)
    .json({ data: book });
});
~~~

### Sample 2: Return a useful response when creating a resource

~~~http
HTTP/1.1 201 Created
Location: /api/books/42
Content-Type: application/json

{
  "data": {
    "id": "42",
    "title": "The Pragmatic Programmer"
  }
}
~~~

### Sample 3: Keep JSON response shapes predictable

~~~json
{
  "data": {
    "id": "42",
    "title": "The Pragmatic Programmer"
  }
}
~~~

### Sample 4: Keep JSON response shapes predictable

~~~json
{
  "error": {
    "code": "BOOK_NOT_FOUND",
    "message": "No book exists with that ID."
  }
}
~~~

### Sample 5: Keep JSON response shapes predictable

~~~json
{
  "data": [
    { "id": "42", "title": "The Pragmatic Programmer" }
  ],
  "meta": {
    "limit": 20,
    "offset": 0,
    "total": 1
  }
}
~~~

### Sample 6: Build a small resource API

~~~js
import express from "express";

const app = express();
app.use(express.json());

const books = new Map([
  ["1", { id: "1", title: "The Pragmatic Programmer", author: "David Thomas" }],
  ["2", { id: "2", title: "Clean Code", author: "Robert C. Martin" }]
]);

let nextBookId = 3;

function sendBookNotFound(response) {
  return response.status(404).json({
    error: {
      code: "BOOK_NOT_FOUND",
      message: "No book exists with that ID."
    }
  });
}

app.get("/api/books", (request, response) => {
  const allBooks = [...books.values()];
  const query = String(request.query.q ?? "").trim().toLowerCase();
  const matchingBooks = query
    ? allBooks.filter((book) => book.title.toLowerCase().includes(query))
    : allBooks;

  const offsetValue = Number(request.query.offset ?? 0);
  const limitValue = Number(request.query.limit ?? 20);
  const offset = Number.isInteger(offsetValue) && offsetValue >= 0 ? offsetValue : 0;
  const limit = Number.isInteger(limitValue) && limitValue > 0
    ? Math.min(limitValue, 100)
    : 20;

  response.json({
    data: matchingBooks.slice(offset, offset + limit),
    meta: {
      limit,
      offset,
      total: matchingBooks.length
    }
  });
});

app.get("/api/books/:bookId", (request, response) => {
  const book = books.get(request.params.bookId);

  if (!book) {
    return sendBookNotFound(response);
  }

  response.json({ data: book });
});

app.post("/api/books", (request, response) => {
  const { title, author } = request.body ?? {};

  if (typeof title !== "string" || title.trim() === "") {
    return response.status(400).json({
      error: {
        code: "TITLE_REQUIRED",
        message: "Provide a non-empty title."
      }
    });
  }

  const book = {
    id: String(nextBookId++),
    title: title.trim(),
    author: typeof author === "string" ? author.trim() : ""
  };

  books.set(book.id, book);

  response
    .location(`/api/books/${book.id}`)
    .status(201)
    .json({ data: book });
});

app.put("/api/books/:bookId", (request, response) => {
  const existingBook = books.get(request.params.bookId);

  if (!existingBook) {
    return sendBookNotFound(response);
  }

  const { title, author } = request.body ?? {};

  if (typeof title !== "string" || title.trim() === ""
      || typeof author !== "string") {
    return response.status(400).json({
      error: {
        code: "INVALID_BOOK",
        message: "A replacement needs a non-empty title and an author string."
      }
    });
  }

  const replacement = {
    id: existingBook.id,
    title: title.trim(),
    author: author.trim()
  };

  books.set(replacement.id, replacement);
  response.json({ data: replacement });
});

app.patch("/api/books/:bookId", (request, response) => {
  const existingBook = books.get(request.params.bookId);

  if (!existingBook) {
    return sendBookNotFound(response);
  }

  const updates = request.body ?? {};
  const allowedFields = ["title", "author"];
  const hasUnknownField = Object.keys(updates)
    .some((field) => !allowedFields.includes(field));

  if (hasUnknownField) {
    return response.status(400).json({
      error: {
        code: "UNKNOWN_FIELD",
        message: "Only title and author can be changed."
      }
    });
  }

  const updatedBook = { ...existingBook };

  if (Object.hasOwn(updates, "title")) {
    if (typeof updates.title !== "string" || updates.title.trim() === "") {
      return response.status(400).json({
        error: {
          code: "INVALID_TITLE",
          message: "Title must be a non-empty string."
        }
      });
    }

    updatedBook.title = updates.title.trim();
  }

  if (Object.hasOwn(updates, "author")) {
    if (typeof updates.author !== "string") {
      return response.status(400).json({
        error: {
          code: "INVALID_AUTHOR",
          message: "Author must be a string."
        }
      });
    }

    updatedBook.author = updates.author.trim();
  }

  books.set(updatedBook.id, updatedBook);
  response.json({ data: updatedBook });
});

app.delete("/api/books/:bookId", (request, response) => {
  if (!books.has(request.params.bookId)) {
    return sendBookNotFound(response);
  }

  books.delete(request.params.bookId);
  response.status(204).end();
});

app.use("/api", (request, response) => {
  response.status(404).json({
    error: {
      code: "ROUTE_NOT_FOUND",
      message: "No API route matches this request."
    }
  });
});

app.listen(3000, () => {
  console.log("Book API listening at http://localhost:3000");
});
~~~

### Sample 7: Try the endpoints

~~~sh
node src/server.js
~~~

### Sample 8: Try the endpoints

~~~sh
curl -i "http://localhost:3000/api/books?limit=1&offset=0"
~~~

### Sample 9: Try the endpoints

~~~sh
curl -i http://localhost:3000/api/books/1
~~~

### Sample 10: Try the endpoints

~~~sh
curl -i -X POST http://localhost:3000/api/books \
  -H "Content-Type: application/json" \
  -d "{\"title\":\"Working Effectively with Legacy Code\",\"author\":\"Michael Feathers\"}"
~~~

### Sample 11: Try the endpoints

~~~sh
curl -i -X PATCH http://localhost:3000/api/books/3 \
  -H "Content-Type: application/json" \
  -d "{\"title\":\"Legacy Code\"}"
~~~

### Sample 12: Try the endpoints

~~~sh
curl -i -X DELETE http://localhost:3000/api/books/3
~~~

## Chapter 11: Input validation and security

[Open chapter](./11-input-validation-and-security.md)

### Sample 1: Write field-specific rules

~~~js
function validateTitle(value) {
  if (typeof value !== "string") {
    return { valid: false, message: "Title must be text." };
  }

  const title = value.trim();

  if (title.length === 0) {
    return { valid: false, message: "Title is required." };
  }

  if (title.length > 120) {
    return { valid: false, message: "Title cannot exceed 120 characters." };
  }

  return { valid: true, value: title };
}
~~~

### Sample 2: Build an allowlisted body

~~~js
function validateNewBook(request, response, next) {
  const body = request.body;

  if (body === null || typeof body !== "object" || Array.isArray(body)) {
    return response.status(400).json({
      error: {
        code: "INVALID_BODY",
        message: "Send a JSON object."
      }
    });
  }

  const titleResult = validateTitle(body.title);

  if (!titleResult.valid) {
    return response.status(422).json({
      error: {
        code: "INVALID_TITLE",
        message: titleResult.message
      }
    });
  }

  let author = "";

  if (body.author !== undefined) {
    if (typeof body.author !== "string" || body.author.length > 100) {
      return response.status(422).json({
        error: {
          code: "INVALID_AUTHOR",
          message: "Author must be text with at most 100 characters."
        }
      });
    }

    author = body.author.trim();
  }

  request.validatedBook = {
    title: titleResult.value,
    author
  };

  next();
}
~~~

### Sample 3: Build an allowlisted body

~~~js
app.post("/api/books", validateNewBook, async (request, response, next) => {
  try {
    const book = await bookStore.create(request.validatedBook);

    response
      .location(`/api/books/${book.id}`)
      .status(201)
      .json({ data: book });
  } catch (error) {
    next(error);
  }
});
~~~

### Sample 4: Validate path and query values

~~~js
function requirePositiveIntegerPathParam(name) {
  return function validatePathParam(request, response, next) {
    const rawValue = request.params[name];

    if (!/^[1-9][0-9]*$/.test(rawValue)) {
      return response.status(400).json({
        error: {
          code: "INVALID_ID",
          message: `${name} must be a positive integer.`
        }
      });
    }

    request.validatedId = Number(rawValue);
    next();
  };
}

app.get(
  "/api/books/:bookId",
  requirePositiveIntegerPathParam("bookId"),
  async (request, response, next) => {
    try {
      const book = await bookStore.findById(request.validatedId);

      if (!book) {
        return response.status(404).json({
          error: {
            code: "BOOK_NOT_FOUND",
            message: "No book exists with that ID."
          }
        });
      }

      response.json({ data: book });
    } catch (error) {
      next(error);
    }
  }
);
~~~

### Sample 5: Limit request body size

~~~js
app.use(express.json({ limit: "10kb" }));
app.use(express.urlencoded({ extended: false, limit: "10kb" }));
~~~

### Sample 6: Add security response headers

~~~sh
npm install helmet
~~~

### Sample 7: Add security response headers

~~~js
import express from "express";
import helmet from "helmet";

const app = express();

app.use(helmet());
app.disable("x-powered-by");
app.use(express.json({ limit: "10kb" }));
~~~

### Sample 8: Keep error details out of client responses

~~~js
app.use((error, request, response, next) => {
  if (response.headersSent) {
    return next(error);
  }

  const status = Number.isInteger(error.status) &&
    error.status >= 400 &&
    error.status < 600
    ? error.status
    : 500;

  request.log?.error({ error }, "Request failed");

  response.status(status).json({
    error: {
      code: status === 500 ? "INTERNAL_ERROR" : "REQUEST_FAILED",
      message: status === 500
        ? "An unexpected error occurred."
        : error.message
    }
  });
});
~~~

## Chapter 12: Authentication and authorization

[Open chapter](./12-authentication-and-authorization.md)

### Sample 1: Store passwords with a password hashing algorithm

~~~sh
npm install argon2
~~~

### Sample 2: Store passwords with a password hashing algorithm

~~~js
import argon2 from "argon2";

const passwordHash = await argon2.hash(password);

// Later, compare the submitted password with the stored hash.
const passwordMatches = await argon2.verify(user.passwordHash, submittedPassword);
~~~

### Sample 3: Configure a server-side session

~~~sh
npm install express-session
~~~

### Sample 4: Configure a server-side session

~~~js
import session from "express-session";

const isProduction = process.env.NODE_ENV === "production";

if (!process.env.SESSION_SECRET) {
  throw new Error("SESSION_SECRET must be configured.");
}

app.use(session({
  name: "sid",
  secret: process.env.SESSION_SECRET,
  resave: false,
  saveUninitialized: false,
  cookie: {
    httpOnly: true,
    secure: isProduction,
    sameSite: "lax",
    maxAge: 1000 * 60 * 60 * 8
  }
}));
~~~

### Sample 5: Create a session after login

~~~js
import argon2 from "argon2";

function regenerateSession(request) {
  return new Promise((resolve, reject) => {
    request.session.regenerate((error) => {
      if (error) {
        reject(error);
      } else {
        resolve();
      }
    });
  });
}

app.post("/api/session", async (request, response) => {
  const { email, password } = request.body ?? {};

  if (typeof email !== "string" || typeof password !== "string") {
    return response.status(400).json({
      error: {
        code: "INVALID_CREDENTIALS",
        message: "Provide an email address and password."
      }
    });
  }

  const user = await users.findByEmail(email.trim().toLowerCase());
  const passwordMatches = user
    ? await argon2.verify(user.passwordHash, password)
    : false;

  if (!passwordMatches) {
    return response.status(401).json({
      error: {
        code: "SIGN_IN_FAILED",
        message: "Email or password is incorrect."
      }
    });
  }

  await regenerateSession(request);
  request.session.userId = user.id;

  response.json({
    data: {
      id: user.id,
      displayName: user.displayName
    }
  });
});
~~~

### Sample 6: Require an authenticated user

~~~js
async function requireAuthenticatedUser(request, response, next) {
  const userId = request.session?.userId;

  if (!userId) {
    return response.status(401).json({
      error: {
        code: "AUTHENTICATION_REQUIRED",
        message: "Sign in to continue."
      }
    });
  }

  const user = await users.findPublicById(userId);

  if (!user) {
    request.session.destroy(() => {});
    return response.status(401).json({
      error: {
        code: "SESSION_INVALID",
        message: "Sign in to continue."
      }
    });
  }

  request.currentUser = user;
  next();
}

app.get("/api/account", requireAuthenticatedUser, (request, response) => {
  response.json({
    data: {
      id: request.currentUser.id,
      displayName: request.currentUser.displayName
    }
  });
});
~~~

### Sample 7: Check roles and record ownership

~~~js
function requireRole(role) {
  return function checkRole(request, response, next) {
    if (!request.currentUser.roles.includes(role)) {
      return response.status(403).json({
        error: {
          code: "FORBIDDEN",
          message: "You do not have permission to perform this action."
        }
      });
    }

    next();
  };
}

app.delete(
  "/api/users/:userId",
  requireAuthenticatedUser,
  requireRole("admin"),
  async (request, response) => {
    await users.deleteById(request.params.userId);
    response.status(204).end();
  }
);
~~~

### Sample 8: Check roles and record ownership

~~~js
app.patch(
  "/api/lists/:listId",
  requireAuthenticatedUser,
  async (request, response, next) => {
    try {
      const list = await lists.findById(request.params.listId);

      if (!list) {
        return response.status(404).json({
          error: {
            code: "LIST_NOT_FOUND",
            message: "No list exists with that ID."
          }
        });
      }

      if (list.ownerId !== request.currentUser.id) {
        return response.status(403).json({
          error: {
            code: "FORBIDDEN",
            message: "You cannot change this list."
          }
        });
      }

      const updatedList = await lists.update(list.id, request.validatedList);
      response.json({ data: updatedList });
    } catch (error) {
      next(error);
    }
  }
);
~~~

### Sample 9: Sign out by destroying the session

~~~js
app.delete("/api/session", requireAuthenticatedUser, (request, response, next) => {
  request.session.destroy((error) => {
    if (error) {
      return next(error);
    }

    response.clearCookie("sid", {
      path: "/",
      httpOnly: true,
      secure: process.env.NODE_ENV === "production",
      sameSite: "lax"
    });

    response.status(204).end();
  });
});
~~~

## Chapter 13: Database integration and asynchronous work

[Open chapter](./13-database-integration-and-async-work.md)

### Sample 1: Keep database work outside route details

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

### Sample 2: Create one PostgreSQL connection pool

~~~sh
npm install pg
~~~

### Sample 3: Create one PostgreSQL connection pool

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

### Sample 4: Use parameterized queries

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

### Sample 5: Use parameterized queries

~~~js
// Unsafe: user input becomes part of the SQL command.
const sql = `SELECT id FROM books WHERE title = '${request.query.title}'`;
~~~

### Sample 6: Use parameterized queries

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

### Sample 7: Return data through a repository function

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

### Sample 8: Call repository functions from Express

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

### Sample 9: Use a transaction for work that must succeed together

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

### Sample 10: Await work and handle concurrency deliberately

~~~js
const [bookResult, reviewResult] = await Promise.all([
  pool.query("SELECT id, title FROM books WHERE id = $1", [bookId]),
  pool.query(
    "SELECT id, rating, body FROM reviews WHERE book_id = $1",
    [bookId]
  )
]);
~~~

### Sample 11: Close the pool during shutdown

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

## Chapter 14: Testing Express applications

[Open chapter](./14-testing-express-applications.md)

### Sample 1: Use Node's built-in test runner

~~~json
{
  "scripts": {
    "test": "node --test"
  }
}
~~~

### Sample 2: Make the Express app importable

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

### Sample 3: Make the Express app importable

~~~js
import { createApp } from "./app.js";
import { bookRepository } from "./repositories/book-repository.js";

const app = createApp({ books: bookRepository });
const port = Number(process.env.PORT ?? 3000);

app.listen(port, () => {
  console.log(`Server listening on port ${port}`);
});
~~~

### Sample 4: Send HTTP requests with Supertest

~~~sh
npm install --save-dev supertest
~~~

### Sample 5: Send HTTP requests with Supertest

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

### Sample 6: Send HTTP requests with Supertest

~~~sh
npm test
~~~

## Chapter 15: Configuration, logging, and debugging

[Open chapter](./15-configuration-logging-and-debugging.md)

### Sample 1: Keep configuration outside application code

~~~js
const nodeEnv = process.env.NODE_ENV ?? "development";
const allowedEnvironments = new Set([
  "development",
  "test",
  "production"
]);

if (!allowedEnvironments.has(nodeEnv)) {
  throw new Error(`Unsupported NODE_ENV value: ${nodeEnv}`);
}

const rawPort = process.env.PORT ?? "3000";
const port = Number(rawPort);

if (!Number.isInteger(port) || port < 1 || port > 65535) {
  throw new Error("PORT must be a valid TCP port number.");
}

export const config = Object.freeze({
  nodeEnv,
  port,
  logLevel: process.env.LOG_LEVEL ?? "info",
  databaseUrl: process.env.DATABASE_URL
});
~~~

### Sample 2: Use a local environment file carefully

~~~text
NODE_ENV=development
PORT=3000
DATABASE_URL=postgres://username:password@localhost:5432/app_dev
SESSION_SECRET=replace-with-a-long-random-secret
LOG_LEVEL=debug
~~~

### Sample 3: Use a local environment file carefully

~~~sh
node --env-file=.env --watch src/server.js
~~~

### Sample 4: Validate secrets and required values

~~~js
function requireEnvironmentValue(name) {
  const value = process.env[name];

  if (typeof value !== "string" || value.trim() === "") {
    throw new Error(`${name} must be configured.`);
  }

  return value;
}

export const sessionSecret = requireEnvironmentValue("SESSION_SECRET");
~~~

### Sample 5: Use structured logs

~~~sh
npm install pino pino-http
~~~

### Sample 6: Use structured logs

~~~js
import { randomUUID } from "node:crypto";
import pino from "pino";
import pinoHttp from "pino-http";

export const logger = pino({
  level: process.env.LOG_LEVEL ?? "info",
  redact: [
    "req.headers.authorization",
    "req.headers.cookie",
    "password",
    "token"
  ]
});

app.use(pinoHttp({
  logger,
  genReqId(request, response) {
    const requestId = randomUUID();
    response.setHeader("X-Request-Id", requestId);
    return requestId;
  }
}));
~~~

### Sample 7: Use structured logs

~~~js
app.patch("/api/books/:bookId", async (request, response) => {
  const book = await bookRepository.update(
    request.params.bookId,
    request.validatedBook
  );

  request.log.info(
    { bookId: book.id, userId: request.currentUser.id },
    "Book updated"
  );

  response.json({ data: book });
});
~~~

### Sample 8: Log errors once with useful context

~~~js
app.use((error, request, response, next) => {
  if (response.headersSent) {
    return next(error);
  }

  const status = Number.isInteger(error.status) &&
    error.status >= 400 &&
    error.status < 500
    ? error.status
    : 500;

  if (status >= 500) {
    request.log.error(
      { err: error, requestId: request.id },
      "Request failed"
    );
  }

  response.status(status).json({
    error: {
      code: status === 500 ? "INTERNAL_ERROR" : "REQUEST_FAILED",
      message: status === 500
        ? "An unexpected error occurred."
        : error.message,
      requestId: request.id
    }
  });
});
~~~

### Sample 9: Debug middleware and routes

~~~sh
DEBUG=express:*,router,router:* node src/server.js
~~~

### Sample 10: Debug middleware and routes

~~~powershell
$env:DEBUG = "express:*,router,router:*"
node src/server.js
~~~

### Sample 11: Debug middleware and routes

~~~powershell
Remove-Item Env:DEBUG
~~~

### Sample 12: Use the Node inspector locally

~~~sh
node --inspect-brk src/server.js
~~~

## Chapter 16: Production deployment and operations

[Open chapter](./16-production-deployment-and-operations.md)

### Sample 1: Prepare a production start command

~~~sh
NODE_ENV=production node src/server.js
~~~

### Sample 2: Prepare a production start command

~~~powershell
$env:NODE_ENV = "production"
node src/server.js
~~~

### Sample 3: Prepare a production start command

~~~sh
npm ci --omit=dev
~~~

### Sample 4: Put a reverse proxy in front of Express

~~~js
app.set("trust proxy", 1);
~~~

### Sample 5: Separate liveness from readiness

~~~js
app.get("/health/live", (request, response) => {
  response.sendStatus(200);
});

app.get("/health/ready", async (request, response) => {
  await pool.query("SELECT 1");
  response.sendStatus(200);
});
~~~

### Sample 6: Separate liveness from readiness

~~~js
app.get("/health/ready", async (request, response, next) => {
  try {
    await pool.query("SELECT 1");
    response.sendStatus(200);
  } catch (error) {
    next(error);
  }
});

app.use((error, request, response, next) => {
  if (response.headersSent) {
    return next(error);
  }

  request.log.error({ err: error }, "Readiness check failed");
  response.sendStatus(503);
});
~~~

### Sample 7: Shut down without dropping active work

~~~js
const server = app.listen(config.port, () => {
  logger.info({ port: config.port }, "Server started");
});

let isShuttingDown = false;

function shutdown(signal) {
  if (isShuttingDown) {
    return;
  }

  isShuttingDown = true;
  logger.info({ signal }, "Graceful shutdown started");

  server.close(async (serverError) => {
    if (serverError) {
      logger.error({ err: serverError }, "HTTP server did not close cleanly");
      process.exitCode = 1;
    }

    try {
      await pool.end();
      logger.info("Database pool closed");
    } catch (poolError) {
      logger.error({ err: poolError }, "Database pool did not close cleanly");
      process.exitCode = 1;
    }
  });
}

process.once("SIGTERM", () => shutdown("SIGTERM"));
process.once("SIGINT", () => shutdown("SIGINT"));
~~~

