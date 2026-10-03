# 11. Input validation and security

[Back to notes index](../README.md)

| Previous | Notes index | Next |
| --- | --- | --- |
| [Previous: REST API design and responses](./10-rest-api-design-and-responses.md) | [Notes index](../README.md) | [Next: Authentication and authorization](./12-authentication-and-authorization.md) |

## Treat every request as untrusted

A request can come from a browser, a mobile app, a script, another server, or a client that has been modified. Browser-side checks improve the user experience, but anyone can skip them and send a request directly.

Validate data on the server before using it. Check both:

- Syntax: is the value the expected type and format?
- Meaning: is the value allowed for this action and consistent with related values?

A book title might need to be a string between 1 and 120 characters. A page size might need to be a positive integer no greater than 100. A status might need to match one of a fixed set of values.

## Validate each input location

Express exposes different parts of the request for different purposes:

| Input | Express property | Example |
| --- | --- | --- |
| Path value | `request.params` | `/api/books/:bookId` |
| Query value | `request.query` | `/api/books?limit=20` |
| Parsed JSON or form | `request.body` | `POST` data |
| Header | `request.get("Header-Name")` | `Authorization` |

Parsing is not validation. `express.json()` can parse JSON, but it does not prove that a field is present, has the expected type, or makes sense to the application. Treat all four input locations as untrusted.

## Write field-specific rules

Use allowlists for fixed choices and explicit limits for free-form values. Do not try to remove every character that looks suspicious. That can damage ordinary names and still does not make a database query or HTML output safe.

For example, a title can contain punctuation and non-English text. A reasonable rule is to require a string, trim its edges, reject an empty result, and set a maximum length. The allowed values depend on what the application is supposed to accept.

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

A type check, a length limit, and a meaningful error make the rule clear. A regular expression is useful for a field with a defined format, such as a short product code. It is a poor replacement for a full email, URL, Unicode name, or free-form comment policy.

## Build an allowlisted body

Never copy the entire request body into a database record or application object. A caller might send fields that the route never intended to let them change, such as `role`, `ownerId`, or `isAdmin`.

Instead, read the fields this action accepts, validate them, and build a new object from those fields:

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

The route can now use only the value produced by the validator:

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

If the request includes an unexpected field, decide whether to reject it or ignore it. Either way, do not silently pass it to a persistence layer.

## Validate path and query values

A path parameter is a string. Convert it deliberately, then check the result before using it:

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

Query strings also arrive as text. Check that a value is a single string, parse it, then enforce a range. Avoid trusting a conversion such as `Number(request.query.limit)` without checking whether it produced a finite integer in the allowed range.

If a query chooses a sort field, map accepted names to known database columns. Do not insert a raw query value into SQL syntax. Database values should use parameterized queries, and table or column choices should come from a fixed allowlist.

## Limit request body size

Large bodies consume memory and processing time. Set a limit appropriate for the endpoint instead of accepting an unlimited JSON payload:

~~~js
app.use(express.json({ limit: "10kb" }));
app.use(express.urlencoded({ extended: false, limit: "10kb" }));
~~~

A file upload endpoint needs its own deliberate size limits, file type checks, safe storage names, and storage location. A filename extension and the request's content type can be forged, so they are not proof of file contents.

A parser limit does not replace field validation. A small request can still contain an invalid ID, an unexpected property, or a string far longer than the application allows.

## Add security response headers

Helmet sets a group of HTTP response headers that help reduce exposure to common browser-side attacks. Install it and place it early in the middleware stack:

~~~sh
npm install helmet
~~~

~~~js
import express from "express";
import helmet from "helmet";

const app = express();

app.use(helmet());
app.disable("x-powered-by");
app.use(express.json({ limit: "10kb" }));
~~~

Helmet is a useful layer, not a complete security system. Review its Content Security Policy for the resources your application actually loads. Use HTTPS in deployment and keep Node.js, Express, and installed packages patched.

## Configure cross-origin access narrowly

Cross-Origin Resource Sharing (CORS) tells browsers which origins may read a response. It does not authenticate a user, grant a permission, or stop command-line clients from calling an endpoint.

If the app needs cross-origin browser requests, allow only the origins and methods it needs. Avoid combining a wildcard origin with credentialed requests. The authentication chapter explains how browser credentials and cookies change the design.

## Keep output safe for its destination

Input validation and output encoding solve different problems. A valid comment can still contain characters that have meaning in HTML. When rendering HTML, use a template engine's escaped output by default and do not mark user content as trusted HTML without a specific sanitizing policy.

For JSON APIs, use `response.json()` to serialize data. Do not build JSON by concatenating strings. For SQL, use parameterized queries rather than placing values inside SQL text. For shell commands, avoid passing user-controlled strings to a command interpreter.

## Protect sensitive routes from abuse

Limit repeated attempts on routes such as sign-in, password recovery, and expensive search. Apply rate limits at the application or trusted edge proxy, and return a clear `429 Too Many Requests` response when the limit is reached.

A production limit should account for how the app is deployed. In a multi-process or multi-server setup, an in-memory counter in one process does not provide a shared limit. Use a suitable shared store or an edge service, and do not trust a client-supplied forwarding header as the real IP address.

Authentication checks and permission checks are separate. A well-formed user ID does not prove that the signed-in user may read or change that user's record. Check ownership or role before performing the operation.

## Keep error details out of client responses

The error handler should return a safe, consistent shape. Keep stack traces, database messages, tokens, passwords, and internal paths in protected server logs, not in public responses.

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

Only expose `error.message` when it is safe and intentionally created for a client error. For unexpected failures, use a general message and log the details privately. If the response has already started, pass the error to Express's default handler.

## Security review checklist

Before releasing an endpoint, check that:

- Request size limits are configured.
- Body, path, query, and header values are validated on the server.
- The code uses only allowed fields from a request.
- The caller is authenticated and authorized for the requested record.
- Database values use parameterized queries.
- HTML output is escaped for its context.
- Sensitive routes have suitable abuse limits.
- HTTPS, safe response headers, and protected secrets are configured.
- Client responses do not reveal internal error details.

## Main references

- [Express production security practices](https://expressjs.com/en/advanced/best-practice-security/)
- [Express 5 API](https://expressjs.com/en/5x/api.html)
- [OWASP Input Validation Cheat Sheet](https://cheatsheetseries.owasp.org/cheatsheets/Input_Validation_Cheat_Sheet.html)
- [OWASP SQL Injection Prevention Cheat Sheet](https://cheatsheetseries.owasp.org/cheatsheets/SQL_Injection_Prevention_Cheat_Sheet.html)
- [Helmet](https://helmetjs.github.io/)