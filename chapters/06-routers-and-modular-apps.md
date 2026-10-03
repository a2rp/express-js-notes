# 06. Routers and modular apps

[Back to notes index](../README.md)

| Previous | Notes index | Next |
| --- | --- | --- |
| [Previous: Request bodies and built-in middleware](./05-request-bodies-and-built-in-middleware.md) | [Notes index](../README.md) | [Next: Middleware flow and custom middleware](./07-middleware-flow-and-custom-middleware.md) |

## Why use a router?

A router groups related routes and middleware behind one mount path. It helps keep the main application file focused on setup and lets a feature own its endpoints.

An Express router can define routes and middleware, but it does not listen on a port by itself. The application mounts it with `app.use()`.

## Create a books router

Use this project structure:

~~~text
src/
|-- routes/
|   `-- books.js
`-- server.js
~~~

Create `src/routes/books.js`:

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

The paths in `books.js` are local to the router. `/` means the router's base path, and `/:bookId` handles one book beneath that base.

## Mount the router in the app

Update `src/server.js`:

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

The router mount path is combined with each router path:

| Router path | Mount path | Final path |
| --- | --- | --- |
| `/` | `/api/books` | `/api/books` |
| `/:bookId` | `/api/books` | `/api/books/:bookId` |

The relative import includes `.js` because Node.js ECMAScript module imports use explicit file extensions.

## Mount more than one feature

A small app can mount several routers:

~~~js
import authorsRouter from "./routes/authors.js";
import booksRouter from "./routes/books.js";

app.use("/api/authors", authorsRouter);
app.use("/api/books", booksRouter);
~~~

Each feature router owns its route paths. The app remains responsible for shared setup such as parsers, logging, and error handling.

## Share parent parameters with a child router

By default, a child router does not receive parameters from its mount path. Use `mergeParams: true` when a nested route needs a parent parameter:

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

A request to `/authors/au-7/books/bk-101` makes both values available in the router. Use this option only when a child route needs parent parameters.

## Test the mounted paths

Start the application and request the final paths:

~~~sh
curl -i http://localhost:3000/api/books
curl -i http://localhost:3000/api/books/bk-101
curl -i http://localhost:3000/api/books/unknown
~~~

The first request returns the collection, the second returns a matching book, and the third returns a `404` response from the router.

## Common mistakes

- Defining a router but forgetting to mount it with `app.use()`.
- Including the mount prefix again inside each router path.
- Forgetting `mergeParams: true` when a child router needs parent parameters.
- Importing a relative JavaScript module without its `.js` extension.
- Expecting a router to start its own HTTP server.

## Sources

- [Express routing guide](https://expressjs.com/en/guide/routing.html)
- [Express 5 Router API](https://expressjs.com/en/5x/api/router.html)
- [Using middleware](https://expressjs.com/en/guide/using-middleware/)