# 10. REST API design and responses

[Back to notes index](../README.md)

| Previous | Notes index | Next |
| --- | --- | --- |
| [Previous: Static files and view rendering](./09-static-files-and-view-rendering.md) | [Notes index](../README.md) | [Next: Input validation and security](./11-input-validation-and-security.md) |

## Think in resources

A resource is a thing the API lets a client read or change. A book collection can be represented by `/api/books`, and one book by `/api/books/:bookId`.

Use the HTTP method to describe the action. The URL names the resource:

| Method | Example path | Typical meaning |
| --- | --- | --- |
| `GET` | `/api/books` | Read a collection |
| `GET` | `/api/books/42` | Read one book |
| `POST` | `/api/books` | Create a book |
| `PUT` | `/api/books/42` | Replace a book |
| `PATCH` | `/api/books/42` | Change selected fields |
| `DELETE` | `/api/books/42` | Remove a book |

Prefer `/api/books/42` over action-shaped paths such as `/api/getBook?id=42`. A route should identify the resource; the method describes what the client wants to do.

## Understand method behavior

`GET` reads data and should not change it. `POST` usually creates a resource or starts an operation. Repeating a `POST` can create more than one resource, so a client should not blindly retry it after an uncertain network failure.

`PUT` replaces the target representation. Sending the same complete replacement more than once should leave the resource in the same state. `PATCH` changes selected fields, and its retry behavior depends on the operation. `DELETE` removes the target resource; a second delete should not restore or recreate it.

An idempotent request has the same intended effect when repeated. `GET`, `PUT`, and `DELETE` are defined as idempotent methods. Design clients and retries with these differences in mind.

## Choose a status code that describes the result

| Status | Use |
| --- | --- |
| `200 OK` | A request succeeded and the response has a body |
| `201 Created` | A request created a resource |
| `202 Accepted` | Work was accepted but is not complete yet |
| `204 No Content` | A request succeeded and there is no response body |
| `400 Bad Request` | The request cannot be understood or has invalid basic structure |
| `401 Unauthorized` | The client must authenticate |
| `403 Forbidden` | The client is known but cannot perform the action |
| `404 Not Found` | The requested resource does not exist |
| `409 Conflict` | The requested change conflicts with current resource state |
| `422 Unprocessable Content` | The request is understood but its supplied values fail validation |
| `429 Too Many Requests` | The client has sent too many requests |
| `500 Internal Server Error` | An unexpected server failure occurred |

Status codes are part of the API contract. A client should be able to decide what to do from the status and a documented response body, without parsing a human sentence.

## Return a useful response when creating a resource

A successful creation commonly returns `201 Created`, the new resource, and a `Location` header with its URL. Express lets a route set the header and status before sending JSON:

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

If this route creates book `42`, a response can look like:

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

The `Location` value identifies the created resource. Returning the resource also lets the client immediately use its assigned ID and server-generated fields.

## Keep JSON response shapes predictable

A consistent shape makes client code easier to write. For example, place successful values under `data` and describe failures under `error`:

~~~json
{
  "data": {
    "id": "42",
    "title": "The Pragmatic Programmer"
  }
}
~~~

~~~json
{
  "error": {
    "code": "BOOK_NOT_FOUND",
    "message": "No book exists with that ID."
  }
}
~~~

For a collection, include pagination information in a `meta` object:

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

Choose a shape and use it across endpoints. Do not expose database internals, stack traces, passwords, access tokens, or private fields as part of a response.

## Build a small resource API

This example uses an in-memory `Map` so the HTTP behavior is easy to see. The data disappears when the process stops. A later chapter connects handlers to a persistent data layer.

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

`PUT` requires a complete replacement in this example. `PATCH` copies the current record and changes only explicitly allowed fields. Neither route trusts a client-supplied `id`.

The example performs basic checks to make the route behavior visible. Real applications need a dedicated validation layer for types, lengths, formats, and cross-field rules. The next chapter develops that layer and explains how to reject unsafe input.

## Try the endpoints

Start the file with Node.js, then use `curl` from another terminal:

~~~sh
node src/server.js
~~~

List the collection:

~~~sh
curl -i "http://localhost:3000/api/books?limit=1&offset=0"
~~~

Read a book:

~~~sh
curl -i http://localhost:3000/api/books/1
~~~

Create a book. The `-i` option displays the status and `Location` header:

~~~sh
curl -i -X POST http://localhost:3000/api/books \
  -H "Content-Type: application/json" \
  -d "{\"title\":\"Working Effectively with Legacy Code\",\"author\":\"Michael Feathers\"}"
~~~

Change only its title:

~~~sh
curl -i -X PATCH http://localhost:3000/api/books/3 \
  -H "Content-Type: application/json" \
  -d "{\"title\":\"Legacy Code\"}"
~~~

Remove it:

~~~sh
curl -i -X DELETE http://localhost:3000/api/books/3
~~~

A `204` response has no JSON body. Clients should check the status before trying to parse a response body.

## Use pagination and filtering deliberately

A collection can grow beyond a practical response size. Pagination lets clients request a smaller portion using query parameters such as `limit` and `offset`. Apply a maximum limit so one request cannot ask the server to return an unbounded number of records.

Filtering also belongs in query parameters because it narrows a collection without changing its identity. For example, `/api/books?q=clean` searches the collection, while `/api/books/2` identifies one book.

For large or frequently changing collections, offset pagination can skip or repeat items as records are added. Cursor pagination can provide more stable traversal. Document the parameters, defaults, maximum page size, ordering, and response metadata so clients can follow the API consistently.

## Avoid common response mistakes

- Do not return `200` for every outcome. A missing resource should normally return `404`.
- Do not send a JSON body after selecting `204 No Content`.
- Do not return `201 Created` without identifying what was created.
- Do not use a `401` response for a signed-in user who lacks permission. Use `403`.
- Do not leak internal errors in a client response. Log useful server details and send a safe error shape.
- Do not accept arbitrary fields from a request and copy them directly into a stored object.
- Do not change a resource during a `GET` request.

## Main references

- [Express 5 response API](https://expressjs.com/en/5x/api/response.html)
- [Express routing guide](https://expressjs.com/en/guide/routing.html)
- [HTTP request methods](https://developer.mozilla.org/en-US/docs/Web/HTTP/Reference/Methods)
- [HTTP response status codes](https://developer.mozilla.org/en-US/docs/Web/HTTP/Reference/Status)
- [RFC 9110: HTTP Semantics](https://www.rfc-editor.org/rfc/rfc9110)