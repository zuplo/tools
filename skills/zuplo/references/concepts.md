# Zuplo Core Concepts

Full reference for Zuplo's architecture, runtime, and configuration.

## What Zuplo is

Zuplo is a **programmable API gateway** built on **web standards**. It lets developers configure routes, apply policies (middleware), and write custom handlers using TypeScript/JavaScript. The gateway runs on a custom JS engine (not Node.js) optimized for speed and security, based on Web Workers APIs.

## Core design principles

1. **OpenAPI-as-config** — The routing configuration _is_ the OpenAPI 3.1 spec. `routes.oas.json` serves as both the API documentation and the runtime routing config.
2. **Web Standards First** — The runtime implements standard web platform APIs (`Request`, `Response`, `Headers`, `fetch()`, Web Crypto, Streams, URL, etc.). No Node.js APIs — no `fs`, `process`, `Buffer`, etc.
3. **Policy Pipeline** — Cross-cutting concerns (auth, rate limiting, validation, transformation) are expressed as composable, reusable policies in an ordered request/response pipeline.
4. **Edge-Native** — Deployed to a global edge network by default (`managed-edge`).
5. **Security by Default** — `eval()` and `new Function()` are blocked. GET/HEAD bodies are stripped. Secrets are separated from env vars. Errors use RFC 7807 Problem Details.
6. **Compatibility Dates** — `compatibilityDate` in `zuplo.jsonc` pins runtime behavior to a specific date.

## Project structure

```
zuplo.jsonc                    # Project config (version, compatibilityDate, projectType)
/config/
  routes.oas.json              # OpenAPI 3.1 spec = routing configuration
/modules/
  *.ts                         # Custom handlers, policies, shared utilities
  zuplo.runtime.ts             # Global runtime extensions, hooks, plugin registration
```

### zuplo.jsonc

```jsonc
{
  "version": 1,
  "compatibilityDate": "2025-02-06",
  "projectType": "managed-edge",        // or "managed-dedicated", "self-hosted"
  "allowDuplicateRoutes": false,
  "allowHostHeaderOverride": false       // managed-dedicated only
}
```

## The request pipeline

```
Request
  → Pre-Routing Hooks       (URL normalization, header-based routing)
  → Route Matching           (path + method against routes.oas.json)
  → Request Hooks            (early validation, correlation IDs)
  → Inbound Policies         (auth, rate limiting, validation — sequential, can short-circuit)
  → Handler                  (core logic — proxy, Lambda, custom code, etc.)
  → Outbound Policies        (response transformation, caching headers)
  → Response Sending Hooks   (security headers, final modifications)
  → Response Sending Final   (read-only — logging, analytics only)
  → Response sent to client
  → waitUntil tasks          (background promises continue executing)
```

**Short-circuiting:** Any inbound policy can return a `Response` instead of a modified `ZuploRequest`, which stops the pipeline immediately (e.g., returning 401 or 429).

## Routes

Defined in `/config/routes.oas.json` as standard OpenAPI 3.1 path items, extended with `x-zuplo-route`:

```json
{
  "paths": {
    "/products/:productId": {
      "get": {
        "operationId": "get-product",
        "x-zuplo-route": {
          "handler": {
            "module": "$import(@zuplo/runtime)",
            "export": "urlForwardHandler",
            "options": {
              "baseUrl": "https://api.example.com",
              "forwardSearch": true
            }
          },
          "policies": {
            "inbound": ["api-key-inbound", "rate-limit-policy"],
            "outbound": ["cache-policy"]
          }
        }
      }
    }
  }
}
```

- **Path parameters:** `:paramName` syntax → `request.params.paramName`
- **Catch-all:** `{*path}` captures remaining URL segments
- **Custom route data:** Any `x-` prefixed property is accessible via `context.route.raw<T>()`
- **`$import()` syntax:** References modules in JSON config — `$import(@zuplo/runtime)` for built-ins, `$import(./modules/my-handler)` for custom code

## Handlers

Every route has exactly one handler. Handlers process a matched request and produce a response.

### Custom handler signature

```ts
export default async function (
  request: ZuploRequest,
  context: ZuploContext,
): Promise<Response> {
  return new Response(JSON.stringify({ hello: "world" }), {
    status: 200,
    headers: { "content-type": "application/json" },
  });
}
```

### Built-in handlers

| Handler          | Export                | Purpose                              |
| ---------------- | --------------------- | ------------------------------------ |
| URL Forward      | `urlForwardHandler`   | Proxy requests to an upstream service |
| URL Rewrite      | `urlRewriteHandler`   | Rewrite URL path before forwarding   |
| AWS Lambda       | `awsLambdaHandler`    | Invoke AWS Lambda functions          |
| WebSocket        | `webSocketHandler`    | Bidirectional WebSocket connections   |
| OpenAPI Spec     | `openApiSpecHandler`  | Serve the OpenAPI document           |
| Redirect         | `redirectHandler`     | HTTP redirects (301/302/307/308)     |
| MCP Server       | `mcpServerHandler`    | Model Context Protocol server        |

All exported from `@zuplo/runtime`.

## Runtime extensions (modules/zuplo.runtime.ts)

Global gateway behavior is configured by exporting `runtimeInit`:

```ts
import { RuntimeExtensions } from "@zuplo/runtime";

export function runtimeInit(runtime: RuntimeExtensions) {
  // Register hooks, plugins, custom error formatting
}
```

**Any error in `runtimeInit` prevents gateway startup** — all requests get 500.

### Lifecycle hooks

| Hook                          | When                          | Can Modify?                |
| ----------------------------- | ----------------------------- | -------------------------- |
| `addPreRoutingHook`           | Before route matching         | URL, headers               |
| `addRequestHook`              | After routing, before policies | Request, or short-circuit  |
| `addResponseSendingHook`      | Before response sent          | Response (headers, body)   |
| `addResponseSendingFinalHook` | After all processing          | No (read-only, for logging)|

### Custom error formatting

```ts
runtime.problemResponseFormat = ({ problem, statusText, additionalHeaders }, request, context) => {
  return new Response(JSON.stringify(problem), {
    status: problem.status,
    headers: { ...additionalHeaders, "content-type": "application/problem+json" },
  });
};
```

### Custom 404 handler

```ts
runtime.notFoundHandler = async (request, context, notFoundOptions) => {
  if (notFoundOptions.routesMatchedByPathOnly.length > 0) {
    return HttpProblems.methodNotAllowed(request, context);
  }
  return HttpProblems.notFound(request, context);
};
```

## Error handling

Errors follow **RFC 7807 Problem Details** format:

```json
{
  "type": "https://httpproblems.com/http-status/404",
  "title": "Not Found",
  "status": 404,
  "detail": "Not Found",
  "instance": "/not-a-path",
  "trace": { "timestamp": "...", "requestId": "...", "buildId": "...", "rayId": "..." }
}
```

### HttpProblems utility

Static methods: `badRequest()`, `unauthorized()`, `forbidden()`, `notFound()`, `methodNotAllowed()`, `conflict()`, `internalServerError()`, `serviceUnavailable()`, etc.

### Error classes

- **`RuntimeError`** — general runtime errors with optional `extensionMembers`
- **`ConfigurationError`** — missing/invalid env vars or secrets

## Authentication

Auth is implemented as inbound policies. On success, `request.user` is populated with `{ sub, data }`.

### API key system

- **Consumers** represent API users, each with unique IDs and custom metadata
- Keys are validated by the `api-key-inbound` policy
- Sets `request.user.sub` to consumer ID, `request.user.data` to metadata

### JWT / OpenID Connect

- Validates tokens, extracts claims into `request.user.data`
- Supports OpenID discovery for public key resolution
- Provider-specific policies: Auth0, Clerk, Cognito, Firebase, Supabase, Okta

## Caching

| Cache                       | Class                              | Use Case                          |
| --------------------------- | ---------------------------------- | --------------------------------- |
| Zone Cache                  | `ZoneCache<T>`                     | Distributed KV, TTL-based         |
| Memory Read-Through         | `MemoryZoneReadThroughCache<T>`    | In-memory with auto-loading       |
| Streaming Cache             | `StreamingZoneCache`               | Caching `ReadableStream` (beta)   |

```ts
const cache = new ZoneCache<UserData>("user-cache", context);
await cache.put(userId, userData, 300); // TTL in seconds
const data = await cache.get(userId);
```

## Environment variables & secrets

```ts
import { environment } from "@zuplo/runtime";
const apiKey = environment.EXTERNAL_API_KEY;
```

- Injected at runtime (not build time)
- **Secrets** are encrypted, stored separately from regular env vars
- Available per environment (production, preview, development)

## Observability

- `context.log.debug/info/warn/error()` — structured, auto-tagged with `requestId`
- Every request gets a UUID (`context.requestId`) in `zp-rid` response header
- `context.incomingRequestProperties` provides geolocation data
- Logging plugins: CloudWatch, Datadog, Dynatrace, Google Cloud, Grafana Loki, New Relic, Splunk, Sumo Logic

## Deployment models

| Model            | `projectType`      | Description                                      |
| ---------------- | ------------------- | ------------------------------------------------ |
| Managed Edge     | `managed-edge`      | Global edge network (default, lowest latency)    |
| Managed Dedicated| `managed-dedicated` | Dedicated resources managed by Zuplo             |
| Self-Hosted      | `self-hosted`       | Customer's Kubernetes infrastructure via Helm    |

### Environments

Each deployment gets separate: configuration, env vars/secrets, API keys, consumers, version history. Environment types: `production`, `preview`, `development`.

Deployments are fast (~20 seconds) and free — teams often run hundreds of environments.

## Web standards APIs available

`fetch()`, `Request`, `Response`, `Headers`, Web Crypto (full `SubtleCrypto`), `ReadableStream`, `WritableStream`, `TransformStream`, `CompressionStream`, `DecompressionStream`, `URL`, `URLPattern`, `TextEncoder`, `TextDecoder`, `atob()`, `btoa()`, `setTimeout`, `setInterval`, `crypto.randomUUID()`.

### Restrictions

- `eval()` and `new Function()` are **blocked**
- No filesystem or process access
- GET/HEAD bodies are stripped
- NPM packages work only if bundled as pure ESM without Node.js dependencies

## Key imports

```ts
import {
  ZuploRequest,
  ZuploContext,
  RuntimeExtensions,
  HttpProblems,
  RuntimeError,
  ConfigurationError,
  ZoneCache,
  MemoryZoneReadThroughCache,
  ContextData,
  BackgroundLoader,
  ZuploServices,
  environment,
} from "@zuplo/runtime";
```

## Test APIs

- **Echo API** (`https://echo.zuplo.io`) — returns request details as JSON
- **E-Commerce API** (`https://ecommerce-api.zuplo.io`) — `/users`, `/products`, `/transactions`
