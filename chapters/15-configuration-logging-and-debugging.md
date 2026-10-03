# 15. Configuration, logging, and debugging

[Back to notes index](../README.md)

| Previous | Notes index | Next |
| --- | --- | --- |
| [Previous: Testing Express applications](./14-testing-express-applications.md) | [Notes index](../README.md) | [Next: Production deployment and operations](./16-production-deployment-and-operations.md) |

## Keep configuration outside application code

Configuration changes between local development, tests, and deployment. Common values include the port, database connection, session secret, and log level. Read them from environment variables so each environment can provide its own values.

Environment variables are strings. Convert and validate them once at startup instead of repeating loose conversions throughout route handlers:

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

If an application requires `DATABASE_URL`, check that it exists during startup and stop with a clear configuration error when it does not. Avoid printing the value because it may contain a username and password.

## Use a local environment file carefully

A local `.env` file is convenient for development. Keep it out of version control and provide an `.env.example` containing names and safe placeholders:

~~~text
NODE_ENV=development
PORT=3000
DATABASE_URL=postgres://username:password@localhost:5432/app_dev
SESSION_SECRET=replace-with-a-long-random-secret
LOG_LEVEL=debug
~~~

Add `.env` to `.gitignore`. Never put real credentials in `.env.example`, source code, logs, or commits. In deployment, use the platform's environment configuration or secret manager.

Recent Node versions can load an environment file before starting the application:

~~~sh
node --env-file=.env --watch src/server.js
~~~

`--watch` restarts the process after source files change. This is a local development command. Production should run a stable process using values supplied by the deployment environment. Check the Node version used by the project before relying on command-line options.

## Validate secrets and required values

A missing or malformed secret should fail at startup, not halfway through a sign-in request:

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

Generate secrets with a cryptographically secure tool and keep them out of Git. Rotating a session secret can invalidate existing sessions, so plan that change. Do not use a default secret in production.

## Use structured logs

Plain text such as `"something failed"` is hard to search across many requests. Structured logs include named fields such as request ID, route, status, duration, and error. Pino is a small JSON logger that works with Express middleware.

Install Pino and its HTTP middleware:

~~~sh
npm install pino pino-http
~~~

Create a logger and attach HTTP logging before the routes:

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

The middleware records the request and response and makes a request-scoped logger available as `request.log`:

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

Use log levels consistently:

| Level | Use |
| --- | --- |
| `fatal` | The process cannot continue |
| `error` | An operation failed and needs attention |
| `warn` | A recoverable problem or unusual condition |
| `info` | Important application events |
| `debug` | Details useful during development |

Keep logs useful and limited. Do not log passwords, cookies, authorization headers, complete request bodies, or unnecessary personal data. Redaction is an extra safeguard, not a reason to send secrets to the logger in the first place. In production, write structured logs to standard output and let the hosting platform collect and retain them.

## Log errors once with useful context

An error handler can log an unexpected failure and return a safe response:

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

The request ID connects the client-visible error to the private server log. Log an error at the boundary that can handle it. Logging the same error in every repository, service, route, and error handler creates duplicate noise.

## Debug middleware and routes

Express uses the `debug` package internally. Temporarily enable its output to see application and router activity:

~~~sh
DEBUG=express:*,router,router:* node src/server.js
~~~

In PowerShell:

~~~powershell
$env:DEBUG = "express:*,router,router:*"
node src/server.js
~~~

Clear the variable after debugging:

~~~powershell
Remove-Item Env:DEBUG
~~~

Debug output helps answer questions such as whether a route matched, which middleware ran, and in what order. Do not leave verbose debugging enabled in production because it can produce large logs and reveal internal details.

## Use the Node inspector locally

The Node inspector lets a debugger pause execution, inspect variables, and follow asynchronous code:

~~~sh
node --inspect-brk src/server.js
~~~

Open the local inspector from a development tool and set a breakpoint in the route or middleware. Keep the inspector bound to the local machine. Exposing a debugger port to an untrusted network can give someone control of the process.

For simpler problems, start with a specific request and check the route path, method, middleware order, status, and request ID. Reproduce the same request with `curl` or a request test before changing unrelated code.

## Keep app creation separate from startup

Tests and tools should be able to import the Express app without opening a network port. Export `createApp()` from one module and put `listen()` in the entry file. This also makes startup configuration, logging, and shutdown behavior easier to inspect.

Avoid running migrations, opening a listener, or making network calls as an import side effect. Call those actions explicitly during startup so failures occur in a predictable place.

## Main references

- [Node.js command-line options](https://nodejs.org/api/cli.html)
- [Node.js environment variables](https://nodejs.org/api/environment_variables.html)
- [Express debugging guide](https://expressjs.com/en/5x/guide/debugging/)
- [Express production performance practices](https://expressjs.com/en/advanced/best-practice-performance/)
- [Pino redaction](https://github.com/pinojs/pino/blob/main/docs/redaction.md)
- [pino-http](https://github.com/pinojs/pino-http)