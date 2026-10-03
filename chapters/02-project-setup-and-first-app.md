# 02. Project setup and the first app

[Back to notes index](../README.md)

| Previous | Notes index | Next |
| --- | --- | --- |
| [Previous: Express.js and the Node.js HTTP layer](./01-express-and-node-http.md) | [Notes index](../README.md) | [Next: Routing fundamentals](./03-routing-fundamentals.md) |

## Check Node.js and npm

Express runs on Node.js, and npm installs the packages used by the project:

~~~sh
node --version
npm --version
~~~

Express 5 requires Node.js 18 or later. That is the minimum version stated by Express, not a recommendation to use an old runtime. Choose a Node.js release that is still supported and keep it updated.

## Create a project and install Express

Run these commands in a new project directory:

~~~sh
mkdir express-study
cd express-study
npm init -y
npm install express@5
~~~

`npm init -y` creates a starter `package.json`. `npm install express@5` installs Express from the 5.x major line and records it as an application dependency. npm also creates `package-lock.json`, which records the resolved dependency tree for repeatable installs.

Dependencies installed for this project are stored in `node_modules`. Do not commit that directory. Commit `package.json` and `package-lock.json` so another developer can install the same dependency tree with `npm ci`.

## Use JavaScript modules

Set the package to use ECMAScript modules and add start scripts:

~~~sh
npm pkg set type=module scripts.start="node src/server.js" scripts.dev="node --watch src/server.js"
~~~

The `"type": "module"` field lets `.js` files in this package use `import` and `export`. Node.js also recognizes module syntax in `.mjs` files, but a project should choose and document one module style.

`node --watch` restarts the process when watched files change. Use it for local development. A production process should start the application normally, without watch mode.

## Create the first server

Create `src/server.js`:

~~~js
import express from "express";

const app = express();
const port = Number(process.env.PORT ?? 3000);

app.get("/", (request, response) => {
  response.status(200).json({
    message: "Express is running"
  });
});

app.get("/health", (request, response) => {
  response.status(200).json({
    status: "ok"
  });
});

app.listen(port, () => {
  console.log(`Express is listening on port ${port}`);
});
~~~

Create the source directory before saving the file:

~~~sh
mkdir src
~~~

The `app` object stores routes and middleware. Each handler receives the request and response. `response.status(200)` sets the HTTP status, and `response.json()` sends a JSON response with the correct content type.

The `PORT` environment variable makes the listening port configurable. If it is not set, this example uses `3000`.

## Start the app and make a request

Start the app in development mode:

~~~sh
npm run dev
~~~

Leave that process running and open another terminal. In PowerShell, request the root and health paths:

~~~powershell
Invoke-RestMethod -Uri http://localhost:3000/
Invoke-RestMethod -Uri http://localhost:3000/health
~~~

A shell with `curl` can use:

~~~sh
curl -i http://localhost:3000/
curl -i http://localhost:3000/health
~~~

The server should return JSON. Stop the development process with `Ctrl+C`.

To start without watch mode, run:

~~~sh
npm start
~~~

## Keep local files out of Git

Create a `.gitignore` file in the project root:

~~~gitignore
node_modules/
.env
~~~

`node_modules` can be recreated from the lock file. `.env` files often hold machine-specific or sensitive values, so keep them out of source control.

## Common setup errors

- **`Cannot use import statement outside a module`:** check that `package.json` contains `"type": "module"` and that you are running the intended project.
- **`Cannot find package 'express'`:** run `npm install` from the directory containing `package.json`.
- **The process cannot find `src/server.js`:** create the `src` directory and save the file with that exact name.
- **Port 3000 is already in use:** stop the other process or start the app with another `PORT` value.

## Sources

- [Express installation and hello world](https://expressjs.com/en/starter/hello-world/)
- [Express 5 migration guide](https://expressjs.com/en/guide/migrating-5/)
- [Node.js ECMAScript modules](https://nodejs.org/api/esm.html)
- [Node.js watch mode](https://nodejs.org/api/cli.html#--watch)