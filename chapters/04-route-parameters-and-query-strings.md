# 04. Route parameters and query strings

[Back to notes index](../README.md)

| Previous | Notes index | Next |
| --- | --- | --- |
| [Previous: Routing fundamentals](./03-routing-fundamentals.md) | [Notes index](../README.md) | [Next: Request bodies and built-in middleware](./05-request-bodies-and-built-in-middleware.md) |

## Read a route parameter

A route parameter is a named segment in a route path. Express places its value in `request.params`:

~~~js
app.get("/books/:bookId", (request, response) => {
  response.json({
    bookId: request.params.bookId
  });
});
~~~

A request to `/books/bk-101` gives the handler a `bookId` value of `"bk-101"`. Route parameter values are strings. Convert and validate them if the application expects a number, UUID, or another specific format.

Use more than one parameter when a resource is nested:

~~~js
app.get("/authors/:authorId/books/:bookId", (request, response) => {
  response.json({
    authorId: request.params.authorId,
    bookId: request.params.bookId
  });
});
~~~

## Use query strings for optional filters

Query strings follow a question mark and do not change which route path matches:

~~~text
GET /books?q=express&page=2
~~~

The route is still `/books`. Express exposes the parsed values through `request.query`:

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

The example validates the query before using it. Query parsing can produce arrays or nested values depending on the configured parser and input. Treat all values from the URL as untrusted input.

A repeated query such as `?q=express&q=node` may not have the single string shape that the handler expects. This example returns a `400 Bad Request` instead of guessing which value to use.

## Handle missing resources

Use a route parameter to find a resource, then send a clear `404` response when it does not exist:

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

In a real application, the lookup would usually use a data layer. The route should still decide how a missing resource is represented in the HTTP response.

## Use a named wildcard in Express 5

A wildcard captures the remaining path segments. Express 5 requires a wildcard name:

~~~js
app.get("/files/*filepath", (request, response) => {
  response.json({
    segments: request.params.filepath
  });
});
~~~

A request to `/files/images/logo.png` gives `request.params.filepath` the segments `["images", "logo.png"]`. A wildcard can match more than one segment, so do not pass its value directly into a filesystem path without applying a safe path policy.

## Test the inputs

With the server running, try these requests:

~~~sh
curl -i "http://localhost:3000/books?q=express&page=1"
curl -i "http://localhost:3000/books?q=express&q=node"
curl -i "http://localhost:3000/books/bk-101"
curl -i "http://localhost:3000/books/unknown"
curl -i "http://localhost:3000/files/images/logo.png"
~~~

The repeated query should return `400`. The unknown book should return `404`. These outcomes come from application checks, not from the parameter syntax alone.

## Key ideas

- Route parameters describe required path segments and are read from `request.params`.
- Query strings carry optional filters or pagination values and are read from `request.query`.
- Express does not know the type or validity your application expects.
- Validate untrusted values before using them in a database query, file path, or calculation.

## Sources

- [Express routing guide](https://expressjs.com/en/guide/routing.html)
- [Express 5 request API](https://expressjs.com/en/5x/api/request.html)
- [Express 5 migration guide](https://expressjs.com/en/guide/migrating-5/)