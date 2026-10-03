# 08. Error handling

[Back to notes index](../README.md)

| Previous | Notes index | Next |
| --- | --- | --- |
| [Previous: Middleware flow and custom middleware](./07-middleware-flow-and-custom-middleware.md) | [Notes index](../README.md) | [Next: Static files and view rendering](./09-static-files-and-view-rendering.md) |

## Separate missing routes from application errors

A request that matches no route is a normal not-found result. Add a final regular middleware after routes to return a consistent `404` response:

~~~js
app.use((request, response) => {
  response.status(404).json({
    error: "Route not found"
  });
});
~~~

This middleware has three arguments, so Express treats it as ordinary middleware. It should be registered after the routes so it only handles requests that reached the end of the route list.

## Handle synchronous errors

Express catches errors thrown while a route handler or middleware is running synchronously and passes them to error handling:

~~~js
app.get("/reports", (request, response) => {
  throw new Error("Report service is unavailable");
});
~~~

A custom error handler has four parameters: `(error, request, response, next)`. Define it after routes and the 404 middleware:

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

The handler returns a safe generic message for unexpected server errors instead of exposing stack traces or internal details. If the response has already started, it delegates to Express's default error handler.

## Let Express 5 forward rejected promises

In Express 5, a route or middleware function that returns a rejected Promise automatically passes the error to the error handler:

~~~js
app.get("/books/:bookId", async (request, response) => {
  const book = await bookStore.findById(request.params.bookId);

  if (!book) {
    return response.status(404).json({ error: "Book not found" });
  }

  response.json(book);
});
~~~

If `findById()` rejects, Express forwards the failure. The handler must return or await the Promise. A detached Promise that is not returned is outside this automatic error flow.

This behavior differs from Express 4 examples that often use a wrapper or `.catch(next)`. Follow the behavior for the Express major version used by the project.

## Handle errors outside the returned Promise

Some asynchronous callbacks are not returned from a route handler. Catch their errors and pass them to `next(error)`:

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

Use an error-first callback's error argument in the same way. The goal is to pass the failure into Express's error pipeline instead of leaving it as an unhandled exception.

## Create an error with an HTTP status

An application can attach a status to an error when that status is safe to return:

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

The final handler above uses a valid 4xx or 5xx status and hides messages for 5xx errors. Avoid attaching raw database or infrastructure details to messages intended for clients.

## Keep error middleware last

A useful application order is:

~~~js
app.use(requestLogger);
app.use(express.json());
app.use("/api/books", booksRouter);

// Handle requests that reached no route.
app.use(notFoundHandler);

// Handle errors passed with next(error) or rejected by a handler.
app.use(errorHandler);
~~~

The error middleware is last so it can receive errors from earlier middleware and routes. An error handler that calls `next(error)` passes the failure to a later error handler or to Express's default one.

## Common mistakes

- Registering the error handler before the routes.
- Defining error middleware with only three parameters.
- Treating every unmatched route as an application exception.
- Sending an error's stack trace or internal message to a production client.
- Throwing inside a detached callback without catching and forwarding the error.
- Sending a response and then trying to send a second response.

## Sources

- [Express error handling](https://expressjs.com/en/guide/error-handling.html)
- [Express 5 migration guide](https://expressjs.com/en/guide/migrating-5/)
- [Express 5 request and response API](https://expressjs.com/en/5x/api.html)