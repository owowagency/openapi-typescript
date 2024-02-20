---
title: Middleware & Auth
---

# Middleware & Auth

Middleware allows you to modify either the request, response, or both for all fetches. One of the most common usecases is authentication, but can also be used for logging/telemetry, throwing errors, or handling specific edge cases.

## Middleware

Each middleware can provide `onRequest()` and `onResponse()` callbacks, which can observe and/or mutate requests and responses.

```ts
import createClient from "openapi-fetch";
import type { paths } from "./api/v1";

const myMiddleware: Middleware = {
  async onRequest(req, options) {
    // set "foo" header
    req.headers.set("foo", "bar");
    return req;
  },
  async onResponse(res, options) {
    const { body, ...resOptions } = res;
    // change status of response
    return new Response(body, { ...resOptions, status: 200 });
  },
};

const client = createClient<paths>({ baseUrl: "https://myapi.dev/v1/" });

// register middleware
client.use(myMiddleware);
```

::: tip

The order in which middleware are registered matters. For requests, `onRequest()` will be called in the order registered. For responses, `onResponse()` will be called in **reverse** order. That way the first middleware gets the first “dibs” on requests, and the final control over the end response.

:::

### Skipping

If you want to skip the middleware under certain conditions, just `return` as early as possible:

```ts
onRequest(req) {
  if (req.schemaPath !== "/projects/{project_id}") {
    return undefined;
  }
  // …
}
```

This will leave the request/response unmodified, and pass things off to the next middleware handler (if any). There’s no internal callback or observer library needed.

### Throwing

Middleware can also be used to throw an error that `fetch()` wouldn’t normally, useful in libraries like [TanStack Query](https://tanstack.com/query/latest):

```ts
onResponse(res) {
  if (res.error) {
    throw new Error(res.error.message);
  }
}
```

### Ejecting middleware

To remove middleware, call `client.eject(middleware)`:

```ts{9}
const myMiddleware = {
  // …
};

// register middleware
client.use(myMiddleware);

// remove middleware
client.eject(myMiddleware);
```

### Handling statefulness

Since middleware uses native `Request` and `Response` instances, it’s important to remember that [bodies are stateful](https://developer.mozilla.org/en-US/docs/Web/API/Response/bodyUsed). This means:

- **Create new instances** when modifying (`new Request()` / `new Response()`)
- **Clone** when NOT modifying (`res.clone().json()`)

By default, `openapi-fetch` will **NOT** arbitrarily clone requests/responses for performance; it’s up to you to create clean copies.

<!-- prettier-ignore -->
```ts
const myMiddleware: Middleware = {
  onResponse(res) {
    if (res) {
      const data = await res.json(); // [!code --]
      const data = await res.clone().json(); // [!code ++]
      return undefined;
    }
  },
};
```

## Auth

This library is unopinionated and can work with any [Authorization](https://developer.mozilla.org/en-US/docs/Web/HTTP/Headers/Authorization) setup. But here are a few suggestions that may make working with auth easier.

### Basic auth

This basic example uses middleware to retrieve the most up-to-date token at every request. In our example, the access token is kept in JavaScript module state, which is safe to do for client applications but should be avoided for server applications.

```ts
import createClient, { type Middleware } from "openapi-fetch";
import type { paths } from "./api/v1";

let accessToken: string | undefined = undefined;

const authMiddleware: Middleware = {
  async onRequest(req) {
    // fetch token, if it doesn’t exist
    if (!accessToken) {
      const authRes = await someAuthFunc();
      if (authRes.accessToken) {
        accessToken = authRes.accessToken;
      } else {
        // handle auth error
      }
    }

    // (optional) add logic here to refresh token when it expires

    // add Authorization header to every request
    req.headers.set("Authorization", `Bearer ${accessToken}`);
    return req;
  },
};

const client = createClient<paths>({ baseUrl: "https://myapi.dev/v1/" });
client.use(authMiddleware);

const authRequest = await client.get("/some/auth/url");
```

### Conditional auth

If authorization isn’t needed for certain routes, you could also handle that with middleware:

```ts
const UNPROTECTED_ROUTES = ["/v1/login", "/v1/logout", "/v1/public/"];

const authMiddleware = {
  onRequest(req) {
    if (UNPROTECTED_ROUTES.some((pathname) => req.url.startsWith(pathname))) {
      return undefined; // don’t modify request for certain paths
    }

    // for all other paths, set Authorization header as expected
    req.headers.set("Authorization", `Bearer ${accessToken}`);
    return req;
  },
};
```
