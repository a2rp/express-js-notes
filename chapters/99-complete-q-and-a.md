# 99. Complete questions and answers

[Back to notes index](../README.md)

| Previous | Notes index | Next |
| --- | --- | --- |
| [Previous: All code samples](./98-all-code-samples.md) | [Notes index](../README.md) | End of notes |

Use these questions to review the core Express and Node.js concepts in this collection. Each group links to the chapter with the longer explanation and examples.

## 1. Express and the Node.js HTTP layer

[Open chapter 1](./01-express-and-node-http.md)

### 1.1 What is Express?

Express is a web framework for Node.js. It provides routing and middleware APIs for building web servers and HTTP APIs.

### 1.2 How are Node.js HTTP and Express related?

Express runs on Node.js and builds on Node's HTTP server and request-response objects. It adds a convenient way to match routes and compose request handling.

### 1.3 What are `request` and `response` in a route?

`request` describes the incoming HTTP request, including its method, path, headers, parameters, and parsed body. `response` provides methods for setting a status, headers, and response body.

### 1.4 What is the request-response cycle?

A client sends a request, Express runs matching middleware and a route, and the app sends a response. The cycle should finish with a response or pass control to another handler.

### 1.5 What happens if a handler neither responds nor calls `next()`?

The request remains open and the client may wait indefinitely. A handler must send a response or pass control onward.

### 1.6 Does Express include a database?

No. Express does not require a particular database or data layer. An application can connect to a database using a driver, an object mapper, or another service.

### 1.7 Is an Express server the same as a browser app?

No. Express usually handles requests on a server process. It can return HTML, JSON, files, or other HTTP responses, which a browser or another client can consume.

## 2. Project setup and the first app

[Open chapter 2](./02-project-setup-and-first-app.md)

### 2.1 What does `npm init` do?

It creates a `package.json` file for the project. That file records package metadata, scripts, and dependencies.

### 2.2 How does a project enable JavaScript ES modules?

Set `"type": "module"` in `package.json`, or use `.mjs` file extensions. Then the project can use `import` and `export`.

### 2.3 Why is Express installed as a dependency?

The app imports Express when it runs, so Express is a runtime dependency. A test-only tool is usually installed as a development dependency.

### 2.4 What does `app.listen()` do?

It starts an HTTP server that accepts requests on a port. Keep it in the application's startup file so tests can import the app without opening a port.

### 2.5 How should an app choose its port?

Read `PORT` from the environment, parse it as a number, and validate that it is a valid port. A local default such as `3000` is useful for development.

### 2.6 What Node.js version does Express 5 require?

Express 5 requires Node.js 18 or later. For a deployed app, use a supported Node.js LTS release and check the framework's current compatibility guidance.

### 2.7 Why keep `createApp()` separate from `listen()`?

An app factory can create routes and middleware without starting a server. Tests can then pass the app to an HTTP test client, while the entry file starts the real listener.

## 3. Routing fundamentals

[Open chapter 3](./03-routing-fundamentals.md)

### 3.1 What makes a route match a request?

A route is selected using the HTTP method and path pattern. A `GET /books` handler does not automatically handle `POST /books`.

### 3.2 Can several routes use the same path?

Yes. Different methods can use the same path, such as `GET /books` to read and `POST /books` to create. Define each method's behavior intentionally.

### 3.3 Why does route order matter?

Express runs matching handlers in registration order. A broad route registered too early may respond before a more specific route can run.

### 3.4 What does `app.route()` provide?

It lets related handlers for one path be chained together. This keeps the path in one place and reduces duplication.

### 3.5 Should a `GET` route change data?

No. `GET` should retrieve information without changing application state. Use an appropriate state-changing method such as `POST`, `PUT`, `PATCH`, or `DELETE`.

### 3.6 How does a route pass a request to another handler?

A route or middleware can call `next()` when it has not completed the response and another handler should continue processing.

### 3.7 How should an app respond when no route matches?

Add a not-found handler after the routes. For a JSON API, return a consistent JSON error with status `404`.

## 4. Route parameters and query strings

[Open chapter 4](./04-route-parameters-and-query-strings.md)

### 4.1 What is a route parameter?

A route parameter is a named part of the path, such as `:bookId` in `/books/:bookId`. Express makes its value available on `request.params`.

### 4.2 Are route parameters already numbers?

No. They arrive as strings. Parse and validate them before using them as numbers or database identifiers.

### 4.3 What is a query string used for?

Query parameters describe options for a request, such as filtering, sorting, or pagination: `/books?limit=10`. Express exposes them on `request.query`.

### 4.4 How is a query parameter different from a route parameter?

A route parameter identifies a resource in the path. A query parameter modifies how a collection or representation is requested.

### 4.5 Can a query parameter appear more than once?

Yes. A URL can contain repeated keys. The parsed value may then be an array depending on the query parser, so validate both its type and its value.

### 4.6 How can a route read a request header?

Use `request.get("Header-Name")`. Header values are still client input and should be validated before use.

### 4.7 Does Express match a route using the query string?

No. Route matching uses the path and HTTP method. Query values are available to the handler after the route has matched.

## 5. Request bodies and built-in middleware

[Open chapter 5](./05-request-bodies-and-built-in-middleware.md)

### 5.1 What does `express.json()` do?

It parses a JSON request body and makes the result available as `request.body`. It does not validate the application's fields or rules.

### 5.2 Why might `request.body` be undefined?

The request may not include a body, may have an unsupported content type, or the matching parser may not have run before the route.

### 5.3 What does `express.urlencoded()` parse?

It parses form data encoded as `application/x-www-form-urlencoded`. Its `extended` option controls how nested form values are parsed.

### 5.4 Why must parsers run before routes?

Middleware runs in registration order. A route can read parsed request values only if the relevant parser has already processed the request.

### 5.5 What does `express.static()` do?

It serves files from a directory, such as images, stylesheets, and browser scripts. Mount it at a deliberate URL prefix and use an absolute filesystem path.

### 5.6 Why set a request body size limit?

Parsing a very large body consumes memory and CPU. A limit reduces resource abuse and should match what the endpoint actually needs.

### 5.7 Does parsing JSON make it safe to store?

No. Validate the type, allowed fields, lengths, and meaning before using the parsed object. Use parameterized database queries for SQL values.

## 6. Routers and modular apps

[Open chapter 6](./06-routers-and-modular-apps.md)

### 6.1 What is an Express router?

A router is a mountable collection of routes and middleware. It supports organizing a large app by feature or resource.

### 6.2 What does mounting a router do?

Mounting a router under a prefix adds that prefix to the paths it handles. A router path `/books` mounted at `/api` handles `/api/books`.

### 6.3 What is `request.baseUrl`?

It identifies the path prefix on which the current router was mounted. `request.originalUrl` preserves the original request URL.

### 6.4 When does `mergeParams` matter?

A child router does not inherit parent route parameters by default. Set `mergeParams: true` when a mounted child router needs values such as a parent `:userId`.

### 6.5 Can middleware be limited to a router?

Yes. Use `router.use()` for middleware shared by that router or add middleware to an individual route when only that route needs it.

### 6.6 Why split routes by feature?

Feature routers make paths, route-specific middleware, and related handlers easier to find. They help organize code without requiring a complex architecture.

### 6.7 Does a router create a separate server?

No. A router is part of the same Express app and shares its request-response cycle. It does not open its own network port.

## 7. Middleware flow and custom middleware

[Open chapter 7](./07-middleware-flow-and-custom-middleware.md)

### 7.1 What is Express middleware?

Middleware is a function that runs during a request. It can inspect or change the request and response, send a response, pass control, or forward an error.

### 7.2 What arguments does regular middleware receive?

A standard middleware function has the form `(request, response, next)`. Error middleware has a different four-argument signature.

### 7.3 What does `next()` do?

It passes control to the next matching middleware or route handler. Call it when the current function has finished its work and the response is not complete.

### 7.4 Can middleware both send a response and call `next()`?

Usually no. Sending a response completes the request. Calling `next()` afterward can cause a second response attempt and errors such as headers already sent.

### 7.5 What does `next("route")` do?

In route middleware, it skips the remaining handlers for the current route and continues looking for another matching route.

### 7.6 Where should authentication middleware run?

Run it before handlers that depend on the authenticated identity. Then check roles and record access before returning or changing protected data.

### 7.7 How does Express 5 handle rejected async middleware?

Express 5 forwards a rejected promise from an async handler or middleware to error handling. The async function should still send a response or return the expected value.

## 8. Error handling

[Open chapter 8](./08-error-handling.md)

### 8.1 How does synchronous route code report an error?

It can throw an error, which Express catches while running the route. A handler can also pass an error to `next(error)`.

### 8.2 How should an async Express 5 route handle a rejected promise?

Express 5 forwards the rejection to error middleware. For callback-based APIs, pass callback errors to `next(error)` because Express cannot observe an unrelated callback automatically.

### 8.3 What is the signature of error middleware?

It has four parameters: `(error, request, response, next)`. Keep all four even when one parameter is unused so Express recognizes it as error middleware.

### 8.4 Is a missing route automatically an application error?

No. A `404` means no route completed the request. Add a not-found handler after all routes to choose the response format.

### 8.5 What should an error handler do if headers were already sent?

Pass the error to `next(error)`. The response may already be partially written, so a new status and JSON body cannot safely replace it.

### 8.6 Should production responses include stack traces?

No. Return a safe error shape and log the details privately. Stack traces can expose internal paths and implementation details.

### 8.7 How can the same error handler support HTML and JSON?

Inspect the request's accepted response types or separate the API and page error routes. Keep each response appropriate for its client.

## 9. Static files and view rendering

[Open chapter 9](./09-static-files-and-view-rendering.md)

### 9.1 What is a static file?

A static file is served as stored, such as an image, stylesheet, or browser script. The server does not fill it with request-specific values.

### 9.2 Why use an absolute path with `express.static()`?

A relative filesystem path depends on the process's current working directory. An absolute path based on the module location behaves consistently when the app starts from different directories.

### 9.3 Does the physical static directory name appear in the URL?

Not necessarily. The URL prefix passed to `app.use()` defines the public path and can differ from the directory name.

### 9.4 What does a view engine do?

It combines a template with data and renders HTML. It is useful when the server needs to generate pages with request-specific content.

### 9.5 Why should templates escape values?

Escaping renders special characters as text instead of executable HTML. This helps prevent untrusted text from becoming markup or script.

### 9.6 Should a JSON API use a view engine?

Usually not. A JSON API can respond with `response.json()`. Use a view engine when the server is responsible for producing HTML pages.

### 9.7 How can one page use both views and static files?

The server can render HTML from a view, and the browser can then request its CSS, images, and scripts through static middleware.

## 10. REST API design and responses

[Open chapter 10](./10-rest-api-design-and-responses.md)

### 10.1 What is a resource-oriented path?

It names the thing being accessed, such as `/api/books` or `/api/books/42`. The HTTP method describes the action.

### 10.2 What is the difference between `PUT` and `PATCH`?

`PUT` replaces the full target representation. `PATCH` changes selected fields. The server should document which fields each operation accepts.

### 10.3 What does idempotent mean?

Repeating the same request has the same intended effect as sending it once. `GET`, `PUT`, and `DELETE` are defined as idempotent methods.

### 10.4 What should a successful create request return?

Usually `201 Created`, the created resource, and a `Location` header that identifies it.

### 10.5 Should `204 No Content` include JSON?

No. A `204` response has no response body. The client should check the status before parsing JSON.

### 10.6 How should an API paginate a collection?

Accept validated parameters such as `limit` and `offset`, enforce a maximum page size, define a stable ordering, and describe the page in response metadata.

### 10.7 Why use a consistent response shape?

Consistent `data`, `meta`, and `error` fields make client code predictable. Document the shape and use it across related endpoints.

## 11. Input validation and security

[Open chapter 11](./11-input-validation-and-security.md)

### 11.1 What is the difference between syntax and semantic validation?

Syntax checks the type and format, such as whether a page size is an integer. Semantic validation checks whether it makes sense, such as whether an end date is after a start date.

### 11.2 Why prefer an allowlist over a blocklist?

An allowlist defines what is accepted. A blocklist tries to guess every dangerous value and can reject valid input while still missing unsafe cases.

### 11.3 Does validation alone prevent SQL injection?

No. Validate business rules, then use parameterized queries so user values remain data rather than SQL syntax.

### 11.4 What is mass assignment?

Mass assignment happens when an app copies request fields directly into a record. A caller may then change fields such as `role` or `ownerId` that the route never intended to expose.

### 11.5 What does Helmet provide?

Helmet adds security-related HTTP headers. It is one protective layer and does not replace TLS, authorization, validation, or safe output handling.

### 11.6 Does CORS authenticate API callers?

No. CORS controls whether browser JavaScript from one origin may read a response. It does not authenticate a user or stop non-browser clients.

### 11.7 Why limit request body size?

A size limit bounds memory and parsing work. Set a limit appropriate to the route and use separate upload limits for files.

## 12. Authentication and authorization

[Open chapter 12](./12-authentication-and-authorization.md)

### 12.1 What is the difference between authentication and authorization?

Authentication establishes who the caller is. Authorization decides what that caller may do or which records they may access.

### 12.2 How should an app store user passwords?

Store a password hash created by a slow adaptive password hashing algorithm such as Argon2id. Never store plain-text passwords or use a fast hash such as SHA-256 for password storage.

### 12.3 Where does `express-session` store session data?

The browser receives a signed session identifier cookie, while the session data lives in a server-side store. The default in-memory store is only for development.

### 12.4 Why regenerate a session after login?

Regenerating the session identifier helps prevent session fixation, where an attacker tries to make a user authenticate with a known session ID.

### 12.5 What do `HttpOnly`, `Secure`, and `SameSite` cookie settings do?

`HttpOnly` prevents page JavaScript from reading the cookie. `Secure` restricts it to HTTPS. `SameSite` limits when browsers send it with cross-site requests.

### 12.6 What is CSRF?

Cross-site request forgery tricks a browser into sending an authenticated request from another site. Cookie-authenticated apps need a deliberate CSRF defense for state-changing requests.

### 12.7 Is a signed JWT encrypted?

No. A signature helps detect changes, but the claims are readable by anyone holding the token. Verify the signature and required claims before trusting it.

## 13. Database integration and asynchronous work

[Open chapter 13](./13-database-integration-and-async-work.md)

### 13.1 Why use a connection pool?

A pool reuses a bounded set of database connections. It avoids the cost of opening a new connection for each query and prevents the app from creating an unbounded number of connections.

### 13.2 When should code use `pool.query()` versus `pool.connect()`?

Use `pool.query()` for one statement. Use `pool.connect()` when several statements must use one client, such as a transaction, and always release that client.

### 13.3 Why use parameterized SQL queries?

Parameters keep user values separate from SQL syntax. This prevents a value from changing the meaning of the SQL command.

### 13.4 Why must a transaction use one client?

A database transaction belongs to one database connection. Every statement from `BEGIN` through `COMMIT` or `ROLLBACK` must use the same checked-out client.

### 13.5 Why release a checked-out client in `finally`?

If code forgets to release a client after an error, the pool can run out of available connections and later requests may stall.

### 13.6 When is `Promise.all()` suitable for database calls?

Use it for independent operations that can safely run at the same time. Use a transaction when related changes must all commit or all roll back together.

### 13.7 Why close the pool during shutdown?

Closing the pool lets the process release its database connections cleanly after active requests finish. It is part of graceful application shutdown.

## 14. Testing Express applications

[Open chapter 14](./14-testing-express-applications.md)

### 14.1 What does Node's built-in test runner provide?

It discovers and runs JavaScript tests and integrates with Node's assertion module. A small project can use it without a separate test framework.

### 14.2 What does Supertest test?

Supertest sends HTTP requests to an Express app and supports assertions on status, headers, and response bodies.

### 14.3 Why pass dependencies into an app factory?

A test can provide an in-memory repository or a controlled fake. The same app structure can use the real database repository when it starts in production.

### 14.4 What is the difference between unit and request tests?

A unit test checks one small function in isolation. A request test exercises Express routing and middleware through HTTP and checks the observable response.

### 14.5 Should tests use the production database?

No. Use fakes for most route tests and a separate isolated database for tests that need to verify real SQL or migrations.

### 14.6 What makes a test deterministic?

It uses controlled data and dependencies, awaits asynchronous work, and does not rely on test order, live external services, or uncontrolled current time.

### 14.7 What should a request test assert?

Check the contract the client relies on: status, headers, response body, and important side effects. A test that only checks that a function ran may miss an incorrect HTTP response.

## 15. Configuration, logging, and debugging

[Open chapter 15](./15-configuration-logging-and-debugging.md)

### 15.1 Why validate configuration at startup?

A missing or malformed value should stop the app with a clear error before it serves requests. This is safer than failing during a later operation.

### 15.2 Are environment variable values numbers or booleans?

They arrive as strings. Parse and validate values such as `PORT` before treating them as numbers or flags.

### 15.3 Should a real `.env` file be committed?

No. Keep credentials out of Git. Commit an example file with placeholder values and configure deployment secrets through the host.

### 15.4 What is structured logging?

Structured logging writes named fields such as request ID, status, route, and duration. Log systems can filter and group those fields more reliably than arbitrary text.

### 15.5 Why redact sensitive log fields?

Logs may be copied to shared systems and retained for a long time. Redacting authorization headers, cookies, passwords, and tokens helps prevent credential exposure.

### 15.6 What is a request ID used for?

It connects a client response to the corresponding server logs. It helps trace one request through middleware, routes, and downstream calls.

### 15.7 How can Express router activity be inspected?

Temporarily set the `DEBUG` environment variable to `express:*,router,router:*`. Disable verbose debugging after reproducing the issue.

## 16. Production deployment and operations

[Open chapter 16](./16-production-deployment-and-operations.md)

### 16.1 What does `NODE_ENV=production` change?

It enables production behavior in Express, including view caching and less verbose default errors. The deployment should still use the app's own safe error responses.

### 16.2 Where should TLS usually be terminated?

A reverse proxy or managed hosting edge commonly handles TLS and forwards traffic to Express. The app must still trust only the actual proxy path.

### 16.3 Why is `trust proxy` configuration sensitive?

It makes Express use forwarded request information such as client IP and protocol. Trusting the wrong proxy path may let a client spoof those values.

### 16.4 What is the difference between liveness and readiness?

Liveness indicates that the process is running. Readiness indicates whether the instance can safely receive normal traffic, including whether required dependencies are reachable.

### 16.5 What should happen on `SIGTERM`?

The app should stop accepting new requests, allow active work to finish, and close resources such as database pools before the host's shutdown deadline.

### 16.6 Why should production instances be stateless?

A load balancer can route requests to different instances. Sessions, uploads, and queued work must live in shared services if every instance needs access.

### 16.7 Why should migrations be a deployment step?

Running migrations once in a controlled step avoids several instances racing to change the same schema. Plan schema changes so old and new app versions can coexist during rollout.

## Main references

- [Express.js documentation](https://expressjs.com/)
- [Express 5.x API reference](https://expressjs.com/en/5x/api.html)
- [Express routing guide](https://expressjs.com/en/guide/routing.html)
- [Express middleware resources](https://expressjs.com/en/resources/middleware.html)
- [Node.js documentation](https://nodejs.org/docs/latest/api/)