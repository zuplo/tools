# Zuplo Policy Configuration Guide

How to discover, understand, and configure Zuplo policies.

## What are policies?

Policies are reusable middleware components in the Zuplo request pipeline. They execute sequentially in declaration order — each policy's output becomes the next policy's input.

- **Inbound policies** run before the handler (auth, rate limiting, validation)
- **Outbound policies** run after the handler (response transformation, caching)
- Any inbound policy can **short-circuit** the pipeline by returning a `Response` instead of a `ZuploRequest`

## Discovering available policies

### Policy catalog

Fetch the machine-readable policy catalog:

```
https://cdn.zuplo.com/portal/policies.v5.json
```

Returns a JSON object with a `policies` array. Each entry includes:

| Field               | Description                                        |
| ------------------- | -------------------------------------------------- |
| `id`                | Policy identifier (e.g. `cors-inbound`)            |
| `name`              | Human-readable name                                |
| `description`       | What the policy does                               |
| `products`          | Which Zuplo products support it                    |
| `documentationUrl`  | Link to full documentation                         |
| `defaultHandler`    | Default configuration (`module`, `export`, `options`) |
| `isBeta`            | Whether the policy is in beta                      |
| `isDeprecated`      | Whether the policy is deprecated                   |

### Reading policy documentation

Before configuring any policy, **always** fetch its documentation page:

```
https://zuplo.com/docs/policies/{policy-id}.md
```

For example:
- `https://zuplo.com/docs/policies/cors-inbound.md`
- `https://zuplo.com/docs/policies/rate-limit-inbound.md`
- `https://zuplo.com/docs/policies/api-key-inbound.md`

The doc page contains the correct configuration format, all available options, and usage examples.

## Configuring policies in routes.oas.json

Policies are referenced by name in the route's `x-zuplo-route` extension:

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
              "baseUrl": "https://api.example.com"
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

## Writing custom policies

### Inbound policy signature

```ts
import { ZuploRequest, ZuploContext } from "@zuplo/runtime";

export default async function (
  request: ZuploRequest,
  context: ZuploContext,
  options: any,
  policyName: string,
): Promise<ZuploRequest | Response> {
  // Return ZuploRequest → continue pipeline
  // Return Response → short-circuit (e.g., 401, 403, 429)
  return request;
}
```

### Outbound policy signature

```ts
import { ZuploContext, ZuploRequest } from "@zuplo/runtime";

export default async function (
  response: Response,
  request: ZuploRequest,
  context: ZuploContext,
): Promise<Response> {
  return response;
}
```

### Programmatic policy invocation

Policies can be invoked conditionally from handlers or other policies:

```ts
const result = await context.invokeInboundPolicy("rate-limit-policy", request);
if (result instanceof Response) return result; // short-circuit
request = result; // continue with modified request

const transformed = await context.invokeOutboundPolicy("json-transform", response, request);
```

## Workflow: Adding a policy to a project

1. **Identify the policy** you need from the policy catalog
2. **Fetch the doc page** for that policy:
   ```
   https://zuplo.com/docs/policies/{policy-id}.md
   ```
3. **Read the configuration format** and required options from the docs
4. **Add the policy** to `routes.oas.json` in the route's `policies.inbound` or `policies.outbound` array
5. **Verify the build** succeeded after saving
