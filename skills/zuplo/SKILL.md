---
name: zuplo
description: "Comprehensive Zuplo API gateway guide. Teaches how to find current documentation, understand the request pipeline, configure routes and policies, write custom handlers, and manage deployments. Covers documentation lookup strategies (llms.txt, individual doc pages), core concepts (OpenAPI-as-config, policy pipeline, web standards runtime), and all built-in policies. Use this skill for all Zuplo development to ensure correct configuration from official docs."
license: Apache-2.0
metadata:
  author: Zuplo
  version: "1.0.0"
  repository: https://github.com/zuplo/skills
---

# Zuplo API Gateway Guide

Build and manage programmable API gateways with Zuplo. This skill teaches you how to find current documentation and correctly configure Zuplo projects.

## Critical rule: Read docs before configuring

Before configuring ANY Zuplo feature (policies, handlers, routes, CORS, rate limiting, auth, etc.), you MUST read the relevant documentation first. Do not rely on training data for Zuplo-specific configuration. The docs are the source of truth for correct configuration format, required fields, and available options.

If you skip this step and produce incorrect configuration, it will break the user's project.

## Documentation lookup guide

### Quick reference

| User Question                        | First Check                                                      | How To                                              |
| ------------------------------------ | ---------------------------------------------------------------- | --------------------------------------------------- |
| "How do I configure X?"             | [`references/docs-lookup.md`](references/docs-lookup.md)         | Fetch specific doc page from `zuplo.com/docs/`      |
| "What policies are available?"      | [`references/policies.md`](references/policies.md)               | List policies and fetch configuration details       |
| "How does the request pipeline work?"| [`references/concepts.md`](references/concepts.md)               | Core architecture and runtime concepts              |
| "How do I write a handler?"         | [`references/concepts.md`](references/concepts.md)               | Handler signatures and built-in handlers            |
| "How do I set up auth?"             | [`references/policies.md`](references/policies.md)               | Auth policies with doc page lookup                  |

### How to find the right documentation

1. **Fetch the doc index** at `https://zuplo.com/docs/llms.txt` to discover all available pages
2. **Find the relevant page** for the feature you need to configure
3. **Fetch that page** to get the correct configuration format, required fields, and examples
4. **For policies specifically**, you can also use the policy catalog at `https://cdn.zuplo.com/portal/policies.v5.json` for machine-readable policy definitions

More details: [`references/docs-lookup.md`](references/docs-lookup.md)

## Core concepts (summary)

Zuplo is a **programmable API gateway** built on **web standards**. Key principles:

- **OpenAPI-as-config** — `routes.oas.json` is both the API spec and the routing config
- **Web Standards First** — Uses `Request`, `Response`, `Headers`, `fetch()`, Web Crypto, Streams (no Node.js APIs)
- **Policy Pipeline** — Composable middleware that snaps into a request/response pipeline
- **Edge-Native** — Global edge deployment by default

### Project structure

```
zuplo.jsonc                    # Project config (version, compatibilityDate)
/config/
  routes.oas.json              # OpenAPI 3.1 spec = routing configuration
/modules/
  *.ts                         # Custom handlers, policies, shared utilities
  zuplo.runtime.ts             # Global runtime extensions and hooks
```

### Request pipeline

```
Request → Pre-Routing Hooks → Route Matching → Request Hooks
  → Inbound Policies (auth, rate limiting — can short-circuit)
  → Handler (core logic)
  → Outbound Policies (response transformation)
  → Response Hooks → Response sent
  → waitUntil tasks (background work)
```

### Key imports

```ts
import {
  ZuploRequest, ZuploContext, RuntimeExtensions, HttpProblems,
  ZoneCache, environment,
} from "@zuplo/runtime";
```

For full details on handlers, runtime objects, caching, authentication, and deployment models, see [`references/concepts.md`](references/concepts.md).

## When you see errors

1. Fetch the relevant doc page for the feature causing the error
2. Verify configuration matches the documented format exactly
3. Check that all required fields are present
4. For build failures, check deployment logs for specific error messages

## Development workflow

1. **Read the docs** for the feature you're configuring
   - Use [`references/docs-lookup.md`](references/docs-lookup.md) to find the right page
2. **Check available policies** if adding middleware
   - Use [`references/policies.md`](references/policies.md) to discover and configure policies
3. **Write configuration** based on current docs (never from memory)
4. **Verify the build** succeeded after saving changes

## Resources

- **Documentation lookup**: [`references/docs-lookup.md`](references/docs-lookup.md) — How to find and fetch Zuplo docs
- **Policy guide**: [`references/policies.md`](references/policies.md) — Discovering and configuring policies
- **Core concepts**: [`references/concepts.md`](references/concepts.md) — Full platform architecture reference
