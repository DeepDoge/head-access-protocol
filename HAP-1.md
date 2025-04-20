# HAP-1: Head Access Protocol v1

> **Status:** DRAFT\
> This protocol is still under development and may change. Do not implement in
> production without understanding the limitations and stability risks.

## Summary

HAP-1 defines a message-based protocol for retrieving the rendered `<head>` HTML
and URL from a single-page application (SPA) embedded in an iframe. The protocol
uses `postMessage` and `MessageChannel` to enable a secure, sandboxed, one-time
metadata transfer from the iframe to the parent window.

## Message Format

### Request

- Type: `postMessage`
- Payload: `"HAP:request:1"`
- Transport: `MessageChannel` (with `port2` attached)
- Direction: Parent → Iframe

The parent creates a `MessageChannel`, sends the `"HAP:request:1"` string via
`postMessage`, and transfers `port2`.

### Response

- Direction: Iframe → Parent
- Sent via: `MessagePort` (`port2` sent by parent)
- Payload: JSON object matching the following interface:

```ts
interface HeadMessage {
    head: string; // Serialized HTML contents of the <head> tag
    url: string; // Value of location.href at the time of response
}
```

The response is posted through `port2` from the iframe back to the parent
window.

## Behavior

- The iframe page listens for incoming `"HAP:request:1"` messages.
- Upon receiving the message, it waits for the SPA to resolve its head metadata
  via `resolveHead()`.
- Once ready, the iframe sends a `HeadMessage` through the transferred
  `MessagePort`.

## Usage

### On the iframe (SPA) side:

```ts
const headResolvers = Promise.withResolvers<string>();

export function resolveHead(head: string) {
    headResolvers.resolve(head);
}

self.addEventListener("message", async (e) => {
    const { data, ports } = e as MessageEvent;

    if (data === "HAP:request:1" && ports[0]) {
        const head = await headResolvers.promise;
        ports[0].postMessage({
            head,
            url: location.href,
        });
    }
});
```

### On the parent (requesting) side:

```ts
function getEmbedHead(
    url: string | URL,
    timeout_ms = 5000,
): Promise<HeadMessage> {
    url = new URL(url);
    const iframe = document.createElement("iframe");
    iframe.src = url.href;
    iframe.sandbox.add("allow-scripts");
    iframe.style.position = "absolute";
    iframe.style.scale = "0";
    iframe.style.pointerEvents = "none";
    iframe.style.userSelect = "none";
    document.body.append(iframe);

    const promise = new Promise<HeadMessage>((resolve, reject) => {
        iframe.onload = () => {
            const iframeWindow = iframe.contentWindow;
            if (!iframeWindow) {
                reject(
                    new Error(`Iframe contentWindow is null for ${url.href}`),
                );
                return;
            }

            const timeout = setTimeout(() => {
                reject(
                    new Error(
                        `Timeout waiting for head metadata from ${url.href}`,
                    ),
                );
            }, timeout_ms);

            const channel = new MessageChannel();
            channel.port1.onmessage = (event) => {
                clearTimeout(timeout);
                resolve(event.data);
            };

            iframeWindow.postMessage("HAP:request:1", "*", [channel.port2]);
        };
    });

    promise.finally(() => {
        iframe.remove();
    });

    return promise;
}
```

## Requirements

- Iframe must support JavaScript execution (`sandbox="allow-scripts"`).
- The parent and iframe must allow communication via `postMessage` (CSP and
  embed restrictions must permit it).
