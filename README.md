# HAP - Head Access Protocol

**HAP** is a message-based protocol for retrieving rendered `<head>` metadata
from single-page applications (SPAs) embedded in iframes. It uses `postMessage`
and `MessageChannel` to enable secure metadata transfer from a sandboxed SPA to
a parent document.

## What is it?

HAP defines how a parent document can request the final `<head>` content and
resolved `location.href` from an iframe-hosted SPA after JavaScript rendering.

## Why?

Many SPAs inject their metadata (like `<title>` and `<meta>` tags) dynamically
via JavaScript. This protocol provides a reliable way to extract that data once
it's fully rendered, without DOM access or cross-origin issues.

## How does it work?

- The parent creates a sandboxed iframe and sends a versioned message:
  - `"HAP:request:1"`
- The iframe listens for this message and responds with:
  - `head`: HTML string of the `<head>` tag
  - `url`: Final `location.href`

Communication is done via a `MessageChannel`, which provides a clean, one-to-one
messaging pipe.

## Message format

- `"HAP:request:1"` (parent → iframe)
- `{ head, url }` (iframe → parent, via `MessagePort`)

See [HAP-1](./HAP-1.md) for complete details and example implementations.

## Versions

- [HAP-1](./HAP-1.md): Initial protocol for metadata transfer

## License

Public domain.
