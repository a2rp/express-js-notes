# 03. Routing fundamentals

[Back to notes index](../README.md)

| Previous | Notes index | Next |
| --- | --- | --- |
| [Previous: Project setup and the first app](./02-project-setup-and-first-app.md) | [Notes index](../README.md) | [Next: Route parameters and query strings](./04-route-parameters-and-query-strings.md) |

## What a route does

A route connects an HTTP method and a path to one or more handler functions. Express checks the request against registered routes and runs the handler for the first matching method and path.

~~~js
app.get("/books", (request, response) => {
  response.status(200).json([
    { id: "bk-101", title: "A Small Express App" }
  ]);
});
~~~

The method is `GET`, and the path is `/books`. The response sends a JSON array with a success status.

## Match routes in a useful order

Register a specific path before a dynamic path that could also match it:

~~~js
app.get("/books/search", (request, response) => {
  response.json({ result: "Search results" });
});

app.get("/books/:bookId", (request, response) => {
  response.json({ result: "A book detail route" });
});
~~~

In Express 5, the second path can match `/books/search` too, with `"search"` treated as the route parameter. If the detail route is registered first, it handles that request before the search route can run.

Route order also matters when several handlers can match the same method and path. Put the handler that should respond first in the appropriate position.

## Use HTTP methods to describe the action

Common API conventions map methods to operations:

| Method | Common use |
| --- | --- |
| `GET` | Read a resource or collection. |
| `POST` | Create a resource or start an operation. |
| `PUT` | Replace a resource. |
| `PATCH` | Change selected fields. |
| `DELETE` | Remove a resource. |

These conventions help clients understand an API. The route handler still needs to validate input and return an appropriate status.

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

These examples demonstrate route shape and response choices. They do not store data. Input parsing and persistent storage are covered in later chapters.

## Group handlers for one path

When several methods share a path, `app.route()` keeps them together:

~~~js
app.route("/books")
  .get((request, response) => {
    response.json([{ id: "bk-101", title: "A Small Express App" }]);
  })
  .post((request, response) => {
    response.status(201).json({ id: "bk-102", title: "A New Book" });
  });
~~~

A dynamic path can be grouped the same way:

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

The `:bookId` segment is a route parameter. Express makes its value available on `request.params`; the next chapter explains how to read and validate it.

## Return the right kind of response

Common response patterns include:

~~~js
response.status(200).json({ items: [] });
response.status(201).json({ id: "bk-102" });
response.status(204).end();
response.status(404).json({ error: "Book not found" });
~~~

Send one response for each request. After calling `response.json()`, `response.send()`, or `response.end()`, return from the handler if more code could try to send another response.

A `204 No Content` response has no response body. Use `.end()` after setting that status.

## Try the routes

With the server from the previous chapter running, request the collection and search path:

~~~sh
curl -i http://localhost:3000/books
curl -i http://localhost:3000/books/search
curl -i http://localhost:3000/books/bk-101
~~~

The search request should reach the specific `/books/search` route because it was registered first. The last request uses the dynamic detail route.

Use `-i` to inspect the HTTP status and response headers as well as the body.

## Sources

- [Express routing guide](https://expressjs.com/en/guide/routing.html)
- [Express 5 migration guide](https://expressjs.com/en/guide/migrating-5/)
- [Express 5.x API reference](https://expressjs.com/en/5x/api.html)