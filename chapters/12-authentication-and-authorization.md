# 12. Authentication and authorization

[Back to notes index](../README.md)

| Previous | Notes index | Next |
| --- | --- | --- |
| [Previous: Input validation and security](./11-input-validation-and-security.md) | [Notes index](../README.md) | [Next: Database integration and asynchronous work](./13-database-integration-and-async-work.md) |

## Separate identity from permission

Authentication answers: who is making this request? Authorization answers: may this person perform this action on this resource?

A request can be authenticated and still be forbidden. For example, a signed-in reader should not be able to edit another reader's private book list just because they know its URL.

A protected operation usually follows this order:

1. Authenticate the request.
2. Check the user's role or permission.
3. Check access to the specific record.
4. Run the operation.

Do these checks on the server for every protected request.

## Store passwords with a password hashing algorithm

Never store a password as plain text or encrypt it with a key that the application can later use to recover the original. Store a password hash produced by a slow, adaptive password hashing algorithm. Fast hashes such as SHA-256 are designed for speed and are not appropriate for password storage.

OWASP currently recommends Argon2id for new applications. Use a maintained library so it generates a unique salt and handles the algorithm format:

~~~sh
npm install argon2
~~~

~~~js
import argon2 from "argon2";

const passwordHash = await argon2.hash(password);

// Later, compare the submitted password with the stored hash.
const passwordMatches = await argon2.verify(user.passwordHash, submittedPassword);
~~~

Store `passwordHash`, not the submitted password. Choose parameters for the environment and follow the library's current guidance. Do not make your own password hashing format. If Argon2id is unavailable in a target environment, choose a suitable alternative such as bcrypt or PBKDF2 and configure it according to current security guidance.

## Choose how the client proves its identity

Two common approaches are server-side sessions and bearer tokens.

| Approach | What the client sends | Where session state lives | Common fit |
| --- | --- | --- | --- |
| Server-side session | An opaque session cookie | Server-side session store | Browser applications |
| Bearer token | A token in an `Authorization` header | Token claims and often server-side authorization data | Mobile apps and service APIs |

A session cookie contains an identifier, not the full session record. The server looks up the associated state. A signed JSON Web Token (JWT) carries claims that can be read by the holder, even when its signature prevents undetected changes. Signing does not encrypt a JWT.

For a same-site browser app, an opaque session cookie is often straightforward to revoke centrally. Tokens can fit clients that explicitly send an authorization header, but revocation, expiry, refresh, storage, and key rotation need a deliberate design. Do not choose a token format just because it is popular.

## Configure a server-side session

Install `express-session`:

~~~sh
npm install express-session
~~~

Configure the session middleware before the routes that use it:

~~~js
import session from "express-session";

const isProduction = process.env.NODE_ENV === "production";

if (!process.env.SESSION_SECRET) {
  throw new Error("SESSION_SECRET must be configured.");
}

app.use(session({
  name: "sid",
  secret: process.env.SESSION_SECRET,
  resave: false,
  saveUninitialized: false,
  cookie: {
    httpOnly: true,
    secure: isProduction,
    sameSite: "lax",
    maxAge: 1000 * 60 * 60 * 8
  }
}));
~~~

`httpOnly` prevents browser JavaScript from reading the cookie. `secure` restricts it to HTTPS. `sameSite` limits some cross-site cookie sending. `maxAge` sets its lifetime. Keep the secret out of source control and use a strong random value from the deployment environment.

The default `MemoryStore` is for local development only. It is not suitable for production because it can leak memory and does not share state across application processes. Configure a maintained store that fits the deployment, such as a database or a dedicated session service.

If TLS ends at a reverse proxy, Express must be configured to trust only the proxy arrangement that actually exists before secure cookies can work. Do not copy a broad `trust proxy` setting without understanding which proxy connections the app accepts.

## Create a session after login

The example uses placeholder data functions. `users.findByEmail()` should use a parameterized database query, and `passwordHash` comes from the stored account record.

~~~js
import argon2 from "argon2";

function regenerateSession(request) {
  return new Promise((resolve, reject) => {
    request.session.regenerate((error) => {
      if (error) {
        reject(error);
      } else {
        resolve();
      }
    });
  });
}

app.post("/api/session", async (request, response) => {
  const { email, password } = request.body ?? {};

  if (typeof email !== "string" || typeof password !== "string") {
    return response.status(400).json({
      error: {
        code: "INVALID_CREDENTIALS",
        message: "Provide an email address and password."
      }
    });
  }

  const user = await users.findByEmail(email.trim().toLowerCase());
  const passwordMatches = user
    ? await argon2.verify(user.passwordHash, password)
    : false;

  if (!passwordMatches) {
    return response.status(401).json({
      error: {
        code: "SIGN_IN_FAILED",
        message: "Email or password is incorrect."
      }
    });
  }

  await regenerateSession(request);
  request.session.userId = user.id;

  response.json({
    data: {
      id: user.id,
      displayName: user.displayName
    }
  });
});
~~~

The response uses the same message whether the email is unknown or the password is wrong. This avoids revealing which email addresses have accounts. Apply request validation and rate limits to this route, as described in the previous chapter.

Regenerating the session identifier after successful sign-in helps prevent session fixation. Do not return the password hash or session secret to the client.

## Require an authenticated user

Load the current user from the session and attach only the fields route handlers need:

~~~js
async function requireAuthenticatedUser(request, response, next) {
  const userId = request.session?.userId;

  if (!userId) {
    return response.status(401).json({
      error: {
        code: "AUTHENTICATION_REQUIRED",
        message: "Sign in to continue."
      }
    });
  }

  const user = await users.findPublicById(userId);

  if (!user) {
    request.session.destroy(() => {});
    return response.status(401).json({
      error: {
        code: "SESSION_INVALID",
        message: "Sign in to continue."
      }
    });
  }

  request.currentUser = user;
  next();
}

app.get("/api/account", requireAuthenticatedUser, (request, response) => {
  response.json({
    data: {
      id: request.currentUser.id,
      displayName: request.currentUser.displayName
    }
  });
});
~~~

In Express 5, a rejected promise from an async middleware or route is forwarded to error handling automatically. A middleware that finishes a response should return without calling `next()`.

Never trust a user ID supplied in the body or query to decide who is signed in. Use the authenticated identity established by the server.

## Check roles and record ownership

A role check can protect an administrative route:

~~~js
function requireRole(role) {
  return function checkRole(request, response, next) {
    if (!request.currentUser.roles.includes(role)) {
      return response.status(403).json({
        error: {
          code: "FORBIDDEN",
          message: "You do not have permission to perform this action."
        }
      });
    }

    next();
  };
}

app.delete(
  "/api/users/:userId",
  requireAuthenticatedUser,
  requireRole("admin"),
  async (request, response) => {
    await users.deleteById(request.params.userId);
    response.status(204).end();
  }
);
~~~

Roles are not enough for records that belong to individual users. Check ownership or a specific permission before returning or changing the record:

~~~js
app.patch(
  "/api/lists/:listId",
  requireAuthenticatedUser,
  async (request, response, next) => {
    try {
      const list = await lists.findById(request.params.listId);

      if (!list) {
        return response.status(404).json({
          error: {
            code: "LIST_NOT_FOUND",
            message: "No list exists with that ID."
          }
        });
      }

      if (list.ownerId !== request.currentUser.id) {
        return response.status(403).json({
          error: {
            code: "FORBIDDEN",
            message: "You cannot change this list."
          }
        });
      }

      const updatedList = await lists.update(list.id, request.validatedList);
      response.json({ data: updatedList });
    } catch (error) {
      next(error);
    }
  }
);
~~~

Depending on the application, returning `404` for another user's record can avoid revealing that the record exists. Apply one policy consistently. This record-level check prevents insecure direct object references, where changing an ID gives access to someone else's data.

## Sign out by destroying the session

Remove the server-side session and clear the browser cookie using the same cookie name and scope:

~~~js
app.delete("/api/session", requireAuthenticatedUser, (request, response, next) => {
  request.session.destroy((error) => {
    if (error) {
      return next(error);
    }

    response.clearCookie("sid", {
      path: "/",
      httpOnly: true,
      secure: process.env.NODE_ENV === "production",
      sameSite: "lax"
    });

    response.status(204).end();
  });
});
~~~

Clearing a cookie in the browser is not a substitute for invalidating the server-side session. A copied session identifier must stop working after sign-out.

## Understand cookies and CSRF

Browsers attach cookies automatically to matching requests. A malicious site may try to cause a signed-in browser to submit a state-changing request. This is cross-site request forgery (CSRF).

For cookie-authenticated applications:

- Keep `SameSite` enabled where the application flow allows it.
- Use a maintained CSRF protection approach for state-changing requests.
- Consider validating the `Origin` header for browser requests as an additional check.
- Do not use `GET` routes to change data.
- Require authorization even when a CSRF token is valid.

`SameSite` is a useful layer, but it is not a complete CSRF strategy for every application. Cross-site integrations can need `SameSite=None; Secure`, which makes a deliberate CSRF defense especially important. A bearer token explicitly sent in an authorization header changes the browser's automatic-cookie behavior, but it does not protect against script running in a compromised client.

## If using bearer tokens

A bearer token is a credential: anyone who obtains it can use it until it expires or is revoked. A JWT signature lets the server detect changes; the payload is not secret by default.

When using JWTs:

- Verify the signature with an explicitly allowed algorithm.
- Check issuer, audience, expiry, and any required claims.
- Use short access-token lifetimes and plan key rotation.
- Decide how logout, account disablement, and early revocation work.
- Avoid putting long-lived tokens in browser storage that page scripts can read.
- Never accept an unsigned token or trust claims before verifying it.

For a sign-in system shared across applications, use a well-supported identity provider and standard protocols such as OpenID Connect rather than inventing a token protocol.

## Password recovery and account changes

A password reset link should use a high-entropy, single-use token with an expiry. Store a protected representation of the token, invalidate it after use, and send it only through a verified contact channel. Do not disclose whether an email account exists in recovery responses.

Require recent authentication or a second check before sensitive changes such as changing an email address, changing a password, or adding an administrator role. Send a notification for important account changes and invalidate sessions when policy requires it.

## Main references

- [Express session middleware](https://expressjs.com/en/resources/middleware/session/)
- [Express production security practices](https://expressjs.com/en/advanced/best-practice-security/)
- [OWASP Authentication Cheat Sheet](https://cheatsheetseries.owasp.org/cheatsheets/Authentication_Cheat_Sheet.html)
- [OWASP Session Management Cheat Sheet](https://cheatsheetseries.owasp.org/cheatsheets/Session_Management_Cheat_Sheet.html)
- [OWASP CSRF Prevention Cheat Sheet](https://cheatsheetseries.owasp.org/cheatsheets/Cross-Site_Request_Forgery_Prevention_Cheat_Sheet.html)
- [OWASP Password Storage Cheat Sheet](https://cheatsheetseries.owasp.org/cheatsheets/Password_Storage_Cheat_Sheet.html)
- [node-argon2](https://github.com/ranisalt/node-argon2)