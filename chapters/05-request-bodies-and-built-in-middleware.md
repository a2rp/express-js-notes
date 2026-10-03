# 05. Request bodies and built-in middleware

[Back to notes index](../README.md)

| Previous | Notes index | Next |
| --- | --- | --- |
| [Previous: Route parameters and query strings](./04-route-parameters-and-query-strings.md) | [Notes index](../README.md) | [Next: Routers and modular apps](./06-routers-and-modular-apps.md) |

## Middleware reads the request body

A request body carries data sent by a client, often when creating or updating a resource. Express does not parse every body automatically. Add a parser that matches the request content type before the route that reads `request.body`.

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

`express.json()` parses requests with a JSON content type. `express.urlencoded()` parses HTML form submissions encoded as URL values. Their results are placed on `request.body`.

Register parsers before routes that need them. Middleware runs in the order it is registered. If a route runs before the parser, the route cannot read the parsed body.

## Send a JSON request

A client should identify its body format with the `Content-Type` header. From PowerShell, send JSON like this:

~~~powershell
$body = @{ title = "Learning Express" } | ConvertTo-Json
Invoke-RestMethod `
  -Uri http://localhost:3000/books `
  -Method Post `
  -ContentType "application/json" `
  -Body $body
~~~

From a shell with `curl`:

~~~sh
curl -i http://localhost:3000/books \
  -H "Content-Type: application/json" \
  -d '{"title":"Learning Express"}'
~~~

The route validates `title` even though a JSON parser already ran. Parsing changes the data into JavaScript values; it does not establish that the values are safe or meaningful for the application.

## Parse HTML form data

A basic HTML form can submit URL-encoded values:

~~~html
<form method="post" action="/books">
  <label>
    Title
    <input name="title" required>
  </label>
  <button type="submit">Save book</button>
</form>
~~~

The browser sends a request with the `application/x-www-form-urlencoded` content type. `express.urlencoded({ extended: false })` handles simple key and value pairs. Choose `extended: true` only when the application needs nested values, and validate the resulting structure either way.

## Choose a parser for the content type

Express includes these body and static middleware functions:

| Middleware | Use |
| --- | --- |
| `express.json()` | Parse JSON request bodies. |
| `express.urlencoded()` | Parse URL-encoded form bodies. |
| `express.raw()` | Read a matching request body as a `Buffer`. |
| `express.text()` | Read a matching request body as text. |
| `express.static()` | Serve files from a directory. |

Most JSON APIs need only `express.json()`. A form-based app may also need `express.urlencoded()`. Add other parsers when a route needs that format.

For a webhook that must verify the exact raw bytes of a signed request, a JSON parser may change the representation before verification. Use a raw body parser for that route and follow the webhook provider's signature instructions.

## Limit the amount of input

A body parser can reject a request that exceeds its configured limit. Set a limit that fits the application's expected payload size instead of accepting arbitrarily large bodies.

~~~js
app.use(express.json({ limit: "32kb" }));
~~~

The example limit is for a small API, not a universal value. Applications that accept uploads need a purpose-built upload flow with its own size and storage rules.

## Common mistakes

- Reading `request.body` without registering a matching parser.
- Registering the parser after the route that needs it.
- Sending JSON without a JSON `Content-Type` header.
- Assuming parsed data has the expected fields or types.
- Accepting large request bodies without an application need.
- Treating static file serving as request body parsing. They are separate middleware tasks.

## Sources

- [Using middleware](https://expressjs.com/en/guide/using-middleware/)
- [Writing middleware](https://expressjs.com/en/guide/writing-middleware/)
- [Express 5 request API](https://expressjs.com/en/5x/api/request.html)