# 09. Static files and view rendering

[Back to notes index](../README.md)

| Previous | Notes index | Next |
| --- | --- | --- |
| [Previous: Error handling](./08-error-handling.md) | [Notes index](../README.md) | [Next: REST API design and responses](./10-rest-api-design-and-responses.md) |

## Serve static assets

Static files are sent as they are stored. Common examples include stylesheets, images, and browser JavaScript. Express provides the `express.static()` middleware for this purpose.

Use an absolute directory path so serving files does not depend on which directory started the Node.js process:

~~~js
import path from "node:path";
import { fileURLToPath } from "node:url";

const currentDirectory = path.dirname(fileURLToPath(import.meta.url));
const publicDirectory = path.join(currentDirectory, "../public");

app.use("/assets", express.static(publicDirectory));
~~~

With this folder layout:

~~~text
project/
|-- public/
|   `-- styles.css
`-- src/
    `-- server.js
~~~

A request to `/assets/styles.css` serves `public/styles.css`. The physical folder name is not part of the URL because the middleware is mounted at `/assets`.

Register static middleware before routes that may respond to the same paths. The first middleware that sends a response ends the request cycle.

## Render a template as HTML

A template engine combines a template file with values from a route and returns generated HTML. It is useful when the server needs to produce pages with data. A JSON API usually sends JSON directly and does not need a view engine.

Install Pug, an Express-compatible template engine:

~~~sh
npm install pug
~~~

Add the view settings and route to the same `src/server.js` file. Reuse the `path` import and `currentDirectory` constant from the static assets example:

~~~js
app.set("views", path.join(currentDirectory, "../views"));
app.set("view engine", "pug");

app.get("/", (request, response) => {
  response.render("home", {
    title: "Express study notes",
    books: [
      { title: "Routing", summary: "Match a request to a handler." },
      { title: "Middleware", summary: "Compose work across a request." }
    ]
  });
});
~~~

Create `views/home.pug`:

~~~pug
doctype html
html(lang="en")
  head
    meta(charset="utf-8")
    meta(name="viewport" content="width=device-width, initial-scale=1")
    title= title
    link(rel="stylesheet" href="/assets/styles.css")
  body
    main
      h1= title
      ul
        each book in books
          li
            h2= book.title
            p= book.summary
~~~

Pug uses indentation to describe the HTML structure. Expressions such as `h1= title` escape values before inserting them into the page. Avoid unescaped template output for user-provided content because it can turn into executable HTML or script.

`response.render("home", values)` resolves the template inside the configured `views` directory and sends the rendered HTML response.

## Keep static assets and views separate

Static files are already complete files that the browser can request directly. Views are templates that the server fills with request-specific values before sending HTML.

A page can use both: Express renders the HTML view, and the browser then requests the stylesheet and images through the static middleware.

## Try the page and asset

Start the app from the project directory and request both URLs:

~~~sh
curl -i http://localhost:3000/
curl -i http://localhost:3000/assets/styles.css
~~~

The first response should contain rendered HTML. The second should return the stylesheet file. Check the mounted path, filesystem path, and current working directory if a static file is not found.

## Sources

- [Serving static files](https://expressjs.com/en/5x/starter/static-files/)
- [Using template engines](https://expressjs.com/en/5x/guide/using-template-engines/)
- [Express 5 application API](https://expressjs.com/en/5x/api/application.html)