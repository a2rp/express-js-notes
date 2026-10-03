# 01. Express.js and the Node.js HTTP layer

[Back to notes index](../README.md)

| Previous | Notes index | Next |
| --- | --- | --- |
| [Previous: Notes index](../README.md) | [Notes index](../README.md) | [Next: Project setup and the first app](./02-project-setup-and-first-app.md) |

## Start with Node.js HTTP

Node.js can create an HTTP server without a web framework. The server receives a request object and a response object, then your code decides what to send:

~~~js
import { createServer } from "node:http";

const server = createServer((request, response) => {
  if (request.method === "GET" && request.url === "/") {
    response.writeHead(200, { "content-type": "text/plain" });
    response.end("Hello from Node.js");
    return;
  }

  response.writeHead(404, { "content-type": "text/plain" });
  response.end("Not found");
});

server.listen(3000, () => {
  console.log("Server listening on port 3000");
});
~~~

The `request` describes the incoming HTTP request. It includes values such as the method, URL, headers, and request stream. The `response` lets the server set status and headers, then send a response body.

As an application grows, writing path checks, parsing request data, and organizing handlers directly in the HTTP callback becomes repetitive.

## What Express adds

Express is a web framework built on Node.js HTTP. It adds a route and middleware layer, plus convenient request and response methods:

~~~js
import express from "express";

const app = express();

app.get("/", (request, response) => {
  response.status(200).send("Hello from Express");
});

app.listen(3000, () => {
  console.log("Express app listening on port 3000");
});
~~~

The same request now passes through Express. A route can match the HTTP method and path, then a handler can send the response. Express also provides helpers such as `response.status()`, `response.json()`, `request.params`, and `request.query`.

Express does not choose your database, validation library, authentication system, or application folder layout. You add the parts your application needs.

## Follow one request through the app

A simplified request flow is:

1. The client sends an HTTP request to the Node.js server.
2. Express runs the registered middleware in order.
3. Express checks routes for a matching method and path.
4. A route handler reads the request and sends a response.
5. The response travels back to the client.

Middleware receives the request (`req`), response (`res`), and a `next` function. It can do work, end the response, or call `next()` to pass control onward. A route handler usually ends the response with a method such as `res.send()` or `res.json()`.

If middleware neither sends a response nor calls `next()`, the request stays open and the client waits.

## Compare the two server shapes

| Node.js HTTP | Express |
| --- | --- |
| `createServer()` receives every request. | `express()` creates an application that handles requests. |
| Your code matches methods and paths. | Routes such as `app.get()` match methods and paths. |
| You write response status and headers with the HTTP response API. | Helpers such as `res.status()` and `res.json()` make common responses shorter. |
| You decide how to compose request handling. | Middleware and routers provide a shared composition model. |

Express does not replace Node.js. It uses the Node HTTP server and adds a web application interface on top of it. Knowing the underlying request and response objects helps when you need lower-level behavior.

## Try the Express response

Start the app, then request its root path:

~~~sh
curl -i http://localhost:3000/
~~~

The `-i` option displays the response headers with the body. Check for an HTTP success status and the text `Hello from Express`.

If the request cannot connect, confirm that the process is running and listening on port `3000`. The next chapter sets up the project and shows how to start the app.

## Key ideas

- Node.js provides the HTTP server and request/response primitives.
- Express builds route matching and middleware on top of Node.js HTTP.
- Middleware runs in registration order and must pass control or finish the response.
- Express is intentionally small. Application choices such as data storage and authentication remain explicit.

## Sources

- [Express 5.x API reference](https://expressjs.com/en/5x/api.html)
- [Express routing guide](https://expressjs.com/en/guide/routing.html)
- [Node.js HTTP module](https://nodejs.org/api/http.html)
- [Express FAQ](https://expressjs.com/en/5x/starter/faq.html)