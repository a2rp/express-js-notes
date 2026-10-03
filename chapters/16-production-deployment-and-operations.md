# 16. Production deployment and operations

[Back to notes index](../README.md)

| Previous | Notes index | Next |
| --- | --- | --- |
| [Previous: Configuration, logging, and debugging](./15-configuration-logging-and-debugging.md) | [Notes index](../README.md) | [Next: All code samples](./98-all-code-samples.md) |

## Prepare a production start command

Production starts the app from a known entry file with the environment supplied by the host:

~~~sh
NODE_ENV=production node src/server.js
~~~

On Windows PowerShell, set the variable in the process environment before starting Node:

~~~powershell
$env:NODE_ENV = "production"
node src/server.js
~~~

Use a supported Node.js LTS release and a maintained Express release. Commit the lockfile so deployment installs the dependency versions that were tested. A common CI or container install command is:

~~~sh
npm ci --omit=dev
~~~

The package `start` script should launch the app, while development-only scripts should stay separate. Production settings should be provided by the hosting environment or secret manager. Do not copy local credentials into a production image or source file.

Setting `NODE_ENV=production` lets Express use production behavior, including less verbose errors and view caching. Client responses should still be controlled by the app's error middleware.

## Put a reverse proxy in front of Express

A reverse proxy or managed platform commonly handles TLS, request buffering, compression, static files, and traffic distribution. Express can then focus on application routes.

Use HTTPS for browser and API traffic. Redirect HTTP to HTTPS at the proxy or application edge, and use secure cookies for browser sessions. Helmet can set security headers, but it does not replace TLS.

If TLS ends before the Node process, Express may need a `trust proxy` setting so `request.protocol`, secure cookies, and client IP handling reflect the original request. Configure only the actual proxy addresses or known hop count. Do not set it to `true` without confirming that the last trusted proxy overwrites forwarded headers and that there is no shorter alternate path to the app.

For example, this setting is appropriate only when exactly one trusted proxy is always between the client and Express:

~~~js
app.set("trust proxy", 1);
~~~

A different deployment may need a trusted subnet or a custom address function. Check the hosting provider's network topology and Express's proxy guide before choosing the value.

## Separate liveness from readiness

A liveness endpoint answers whether the process is running. It should not fail just because a downstream database is temporarily unavailable. A readiness endpoint answers whether the instance can accept application traffic:

~~~js
app.get("/health/live", (request, response) => {
  response.sendStatus(200);
});

app.get("/health/ready", async (request, response) => {
  await pool.query("SELECT 1");
  response.sendStatus(200);
});
~~~

For production, replace the minimal readiness route above with this version, which forwards dependency failures to a handler that returns `503 Service Unavailable`. Health endpoints should not return connection strings, server paths, environment variables, or detailed dependency errors.

~~~js
app.get("/health/ready", async (request, response, next) => {
  try {
    await pool.query("SELECT 1");
    response.sendStatus(200);
  } catch (error) {
    next(error);
  }
});

app.use((error, request, response, next) => {
  if (response.headersSent) {
    return next(error);
  }

  request.log.error({ err: error }, "Readiness check failed");
  response.sendStatus(503);
});
~~~

Configure the load balancer or host to send normal traffic only to ready instances. Keep the liveness check lightweight so it does not depend on every service being healthy.

## Shut down without dropping active work

When a deployment replaces an instance, the process should stop accepting new connections, finish active requests, and close shared resources such as a database pool:

~~~js
const server = app.listen(config.port, () => {
  logger.info({ port: config.port }, "Server started");
});

let isShuttingDown = false;

function shutdown(signal) {
  if (isShuttingDown) {
    return;
  }

  isShuttingDown = true;
  logger.info({ signal }, "Graceful shutdown started");

  server.close(async (serverError) => {
    if (serverError) {
      logger.error({ err: serverError }, "HTTP server did not close cleanly");
      process.exitCode = 1;
    }

    try {
      await pool.end();
      logger.info("Database pool closed");
    } catch (poolError) {
      logger.error({ err: poolError }, "Database pool did not close cleanly");
      process.exitCode = 1;
    }
  });
}

process.once("SIGTERM", () => shutdown("SIGTERM"));
process.once("SIGINT", () => shutdown("SIGINT"));
~~~

The hosting platform should allow enough shutdown time for active requests to finish. Set its termination grace period to match the app's behavior. Avoid calling `process.exit()` during normal shutdown because it can stop pending responses and logs.

## Keep instances stateless

A load balancer may send consecutive requests from one user to different app instances. Store shared state in a shared service rather than process memory:

- Use a shared session store for login sessions.
- Keep uploaded files in object storage or a shared file service.
- Put queued work in a shared queue.
- Use a shared cache when cached values must be consistent across instances.

Sticky sessions can route one user back to the same instance, but they do not replace durable shared storage. An instance can restart or be removed at any time.

## Protect performance and reliability

- Use asynchronous I/O for file, network, and database work.
- Avoid synchronous file operations and CPU-heavy work inside request handlers.
- Set request body and upload size limits.
- Put a bounded connection pool in front of the database.
- Add timeouts and sensible retry policies when calling dependencies.
- Use a reverse proxy or CDN for static files and response compression where appropriate.
- Cache only responses whose freshness and user-specific behavior are understood.
- Keep dependencies patched and remove packages the app no longer uses.
- Monitor request errors, latency, process memory, event-loop delay, and dependency health.

A retry is safe only when repeating the operation is safe. Do not retry a payment or create request blindly after a timeout. Use idempotency keys for operations that must not happen twice.

## Run database changes as a deployment step

Apply schema migrations in a controlled deployment step before sending traffic to code that requires the new schema. Avoid making every app instance run the same migration automatically at startup, especially when several instances can start at once.

Use changes that can coexist with both the old and new app version when deployments overlap. For a column rename, for example, a staged migration can add the new column, write both forms temporarily, move reads, then remove the old column in a later release.

Back up important data and have a tested recovery plan. A database migration can be difficult to reverse even when the application deployment itself can be rolled back.

## A practical deployment sequence

1. Run the test suite and check the lockfile.
2. Build or package the app in a clean environment.
3. Install production dependencies.
4. Apply reviewed database migrations.
5. Start the new app process with production configuration.
6. Wait for its readiness check to pass.
7. Route traffic to the new instance.
8. Observe error rate and latency, then remove the old instance after it drains.

Use the capabilities of the selected host for process restarts, TLS certificates, health checks, secret storage, and log collection. The exact interface differs by provider, but the application should still return correct HTTP responses and shut down cleanly.

## Production checklist

- `NODE_ENV` is `production`.
- The app runs on a supported Node.js and Express release.
- HTTPS is enabled and secure cookies are configured.
- Forwarded headers are trusted only from the real proxy path.
- Secrets are injected at runtime and not logged.
- Readiness and liveness checks do not disclose private details.
- A shared store is used for sessions and other cross-instance state.
- Database pools, body sizes, timeouts, and retries are bounded.
- Migrations, rollback, backups, and shutdown behavior are planned.
- Logs and health signals are collected and reviewed.

## Main references

- [Express production performance and reliability](https://expressjs.com/en/advanced/best-practice-performance/)
- [Express production security practices](https://expressjs.com/en/advanced/best-practice-security/)
- [Express behind proxies](https://expressjs.com/en/guide/behind-proxies.html)
- [Express session middleware](https://expressjs.com/en/resources/middleware/session/)
- [Node.js command-line options](https://nodejs.org/api/cli.html)