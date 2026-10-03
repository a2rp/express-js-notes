# 07. Middleware flow and custom middleware

[Back to notes index](../README.md)

| Previous | Notes index | Next |
| --- | --- | --- |
| [Previous: Routers and modular apps](./06-routers-and-modular-apps.md) | [Notes index](../README.md) | [Next: Error handling](./08-error-handling.md) |

## Understand the middleware function

Middleware is a function that runs during the request and response cycle. It receives the request, response, and `next` function:

~~~js
function middleware(request, response, next) {
  // Read or update request and response state.
  next();
}
~~~

A middleware function can:

1. Run code for the request.
2. Add information to the request or response.
3. Send a response and finish the cycle.
4. Call `next()` to let the next matching middleware or route run.
5. Call `next(error)` to pass a failure to error handling.

If middleware neither ends the response nor calls `next()`, the request stays open.

## Registration order defines the flow

Express runs matching middleware in registration order. A simplified pipeline might look like this:

~~~text
request
  -> request logger
  -> request body parser
  -> route-specific checks
  -> route handler
  -> response
~~~

Register shared middleware before routes that depend on it:

~~~js
app.use(requestLogger);
app.use(express.json());
app.use("/api/books", booksRouter);
~~~

A middleware mounted with a path runs only for requests that match that path:

~~~js
app.use("/api", apiRequestLogger);
~~~

Path-scoped middleware is useful when one part of an app needs behavior that should not run for every route.

## Add a request logger

This logger records the method, URL, status, and time after the response finishes:

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

The listener is registered before `next()` so it is in place when the route sends its response. The response `finish` event runs after the response has been handed off.

For production logging, use a structured logger and avoid recording credentials, authorization headers, or sensitive request bodies.

## Add a request identifier

A generated request identifier can connect an incoming request to log entries:

~~~js
import { randomUUID } from "node:crypto";

function addRequestId(request, response, next) {
  request.requestId = randomUUID();
  response.setHeader("X-Request-Id", request.requestId);
  next();
}

app.use(addRequestId);
~~~

A new identifier is created by the server for each request. Middleware later in the chain can include `request.requestId` in diagnostic output.

## Use route-specific middleware

A route can use middleware for a check that only applies to that route:

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

This example assumes earlier authentication middleware has attached a trusted `request.user`. A permission check does not establish identity by itself. Authentication and authorization are covered in a later chapter.

## Avoid ending or advancing twice

After sending a response, stop the handler:

~~~js
app.get("/books/:bookId", (request, response) => {
  const book = findBook(request.params.bookId);

  if (!book) {
    return response.status(404).json({ error: "Book not found" });
  }

  response.json(book);
});
~~~

The `return` prevents execution from continuing after the missing-book response. Otherwise, later code might try to send a second response and trigger a headers-already-sent error.

Call `next()` only when control should continue. Do not call it both before and after asynchronous work.

## Know the special `next` values

- `next()` passes control to the next matching non-error middleware or route handler.
- `next(error)` skips ordinary handlers and passes the error to error-handling middleware.
- `next("route")` skips the remaining callbacks for the current route and checks later matching routes.

The four-argument error handler is covered in the next chapter. Use it for failures rather than sending an internal error through ordinary route handlers.

## Sources

- [Using middleware](https://expressjs.com/en/guide/using-middleware/)
- [Writing middleware](https://expressjs.com/en/guide/writing-middleware/)
- [Express 5 routing guide](https://expressjs.com/en/guide/routing.html)