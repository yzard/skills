---
name: restful-api-design
description: RESTful API path and method conventions across all languages. Use when defining or reviewing API routes, endpoint structure, or HTTP method choices to align paths with product components and standardize on POST usage with limited GET exceptions.
---

# Restful API Design

## Overview

Define RESTful API paths and HTTP methods with consistent, product-component-focused routing and a POST-first convention.

## Guidelines

### Path Structure

- Keep path nouns aligned to product component notions (photos, movies, tags, etc.).
- Use the pattern `/api/v1/<notion>/<operation>` for endpoint routes.

### HTTP Method

- Use `POST` for RESTful API endpoints by default.
- Use `GET` only for conventions like streaming video to the frontend or serving static assets.

## Examples

```text
/api/v1/photo/get
/api/v1/photo/delete
/api/v1/photo/add
/api/v1/tags/add
```

## Checklist

- [ ] Path uses product component notions for `<notion>`
- [ ] Route follows `/api/v1/<notion>/<operation>`
- [ ] Endpoint uses `POST` unless streaming media or serving static assets
