# HTTP

> Notes on HTTP from *HTTP: The Definitive Guide*.

## Statelessness

HTTP is a **stateless** protocol: each request is handled in isolation, and the server keeps no memory
of previous requests from the same client. Every request must therefore carry enough context to be
understood on its own.

This is the single design constraint that explains most of what follows:

- **Why `Cookie` exists** — a client-side key the server uses to look up prior context
  ([Cookies and Sessions](Cookies-and-Sessions.md)).
- **Why `Authorization` exists** — credentials are re-sent on every request rather than implied by an
  established session ([Authentication](Authentication.md)).
- **Why any two requests may land on different servers** — anything that must persist lives in a
  database or cache, not in server memory. This is the constraint that forces horizontal scaling.
- **Why `PUT` is idempotent and `POST` is not** — a stateless protocol cannot know whether it has
  already applied your request.

Statelessness is a feature, not a limitation: it makes any server in a pool interchangeable, which is
what allows load balancing, rolling deploys and caching. The cost is that every request is bigger than
it would otherwise need to be, and continuity has to be re-established by the application layer.

---

## Contents

- [Terminology](#terminology)
- [Methods](#methods)
- [Method Categories](#method-categories)
- [Status Codes](#status-codes)
- [HTTP Messages](#http-messages)
- [Headers](#headers)
- [Media Types](#media-types)
- [Content Negotiation](#content-negotiation)
- [TCP/IP and HTTP](#tcpip-and-http)
- [Telnet](#telnet)
- [Architectural Components of the Web](#architectural-components-of-the-web)
- [Webhooks and Callbacks](#webhooks-and-callbacks)
- [Types of APIs](#types-of-apis-rest-and-rpc)
- [URLs](#urls)

### Related notes

- [HTTP Connections and Keep-Alive](HTTP-Connections.md)
- [Cookies and Sessions](Cookies-and-Sessions.md)
- [Authentication](Authentication.md)
- [Rate Limiting](Rate-Limiting.md)

---

## Terminology

**HTTP** — HyperText Markup Language (technically HyperText Transfer Protocol; "markup" is a common
misnomer, since HTTP transfers representations, it does not mark them up).

**MIME** — Multipurpose Internet Mail Extensions. Web servers attach a MIME type to all HTTP object
data. When a browser receives an object from a server, it checks the associated MIME type to see if it
knows how to handle the object.

**URI** — Uniform Resource Identifier: the postal addresses of the internet. Two types: **URL** and **URN**.

### URL — Uniform Resource Locator

```
http://www.joes-hardware.com/specials/saw-blade.gif
```

It tells the client to use the HTTP protocol, go to `www.joes-hardware.com`, and grab the resource
`specials/saw-blade.gif`.

URL format:

| Part | Example | Meaning |
|---|---|---|
| scheme | `http://` | Which protocol to use |
| server internet address | `www.joes-hardware.com` | Hostname, resolved to an IP via DNS |
| resource | `/specials/saw-blade.gif` | Path to the specific resource |

Most URIs are URLs.

### URN — Uniform Resource Name

Serves as a unique name for a particular piece of content, independent of where the resource resides.
URNs let a resource be accessed by multiple network access protocols while keeping the same name.

```text
urn:ietf:rfc:2141
```
The key contrast: a **URL** says *where* the thing is right now (and stops working if it moves), while
a **URN** says *what* the thing is (and keeps working no matter where it is fetched from).


## Methods

| Method | Purpose | Safe | Idempotent |
|---|---|---|---|
| `GET` | Retrieve a representation of the resource | yes | yes |
| `HEAD` | Same as GET, but response body omitted | yes | yes |
| `OPTIONS` | Query the server for communication options | yes | yes |
| `POST` | Submit an entity to the specified resource, often causing a change in state or side effects on the server | no | no |
| `PUT` | Replace the current representation of the target resource with the request content | no | yes |
| `PATCH` | Apply a partial modification to the target resource | no | no |
| `DELETE` | Remove the target resource | no | yes |
| `TRACE` | Loop back the received request for diagnostic purposes | no | no |
| `QUERY` | Initiate a server-side query; processes the request content in a safe and idempotent manner | yes | yes |


 `TRACE` is the one method that is neither safe nor idempotent despite being a
diagnostic — it is classified here by its intended effect, and most servers disable it anyway.

### GET

Retrieves data. A GET request should not contain a request body; anything you want to send to the
server should be encoded into the URL (query string).

### QUERY

Initiates a server-side query. Requests that target a resource are processed in a safe and idempotent
manner. Similar to GET, but it allows request content with defined semantics.

It exists to fill the gap that forced people to abuse GET (stuffing a search query into the URL, where
it ends up in logs, browser history and `Referer` headers) or to abuse POST (which is unsafe and
non-idempotent, so caches and retries handle it badly).

```text
QUERY /books?genre=scifi HTTP/1.1
Host: www.example.com
Content-Type: application/json
Content-Length: 18

{"since": "2020-01-01"}
```

### GET vs QUERY

| | GET | QUERY |
|---|---|---|
| Semantics | Safe, idempotent retrieval | Safe, idempotent server-side query |
| Request content | Not expected; encoded in the URL | Allowed, with defined semantics |
| Cacheable | Yes | Yes (safe method ⇒ cacheable by default) |
| Typical use | Fetch a representation | Search / filter / complex read |
| Body in URL | Often abused for filters | Not needed |

The short version: **QUERY is GET with a body.** Anything you can do with a GET, a QUERY can do too —
and it lets you pass structured filter criteria in the body instead of cramming them into a query
string.

### POST

Submits an entity to the specified resource, often causing a change in state or side effects on the
server. Not idempotent — sending it twice may create two resources.

```text
POST /users HTTP/1.1
Host: www.example.com
Content-Type: application/json

{"name": "Ada", "role": "backend"}
```

### PUT

Replaces the *current representation* of the target resource with the request content. Idempotent:
replaying the same PUT leaves the resource in the same state.

```text
PUT /users/42 HTTP/1.1
Host: www.example.com
Content-Type: application/json

{"name": "Ada", "role": "staff engineer"}
```

### POST vs PUT

| | POST | PUT |
|---|---|---|
| Target | Collection / processing endpoint | Specific resource URI |
| Create vs overwrite | Usually creates a new child resource | Fully replaces the resource |
| URI after create | Server assigns and returns the new URI | You already know the URI |
| Idempotent | No | Yes |
| Retry safe | No | Yes |
| Partial update | No (use PATCH) | No (use PATCH) |

Rule of thumb: **POST creates a subordinate at a URI the server chooses; PUT replaces or creates at a
URI you already know.** PUT is idempotent because "set state X" repeated any number of times equals
"set state X" once — the second call just overwrites with identical content.

### PATCH

Applies a *partial* modification to the target resource. Unlike PUT, the request body only contains
the changes. The format is negotiated by the `Content-Type` header, which must be one of the patch
media types — this is how the server knows how to interpret the body.

**JSON Merge Patch (RFC 7386)** — a partial JSON object; keys present are set, keys absent are left
alone. A `null` value means *delete the key*.

```text
PATCH /users/42 HTTP/1.1
Host: www.example.com
Content-Type: application/merge-patch+json

{ "email": "new@example.com" }
```

**JSON Patch (RFC 6902)** — an explicit operation list; nothing is inferred, and `null` is just a value.

```text
PATCH /users/42 HTTP/1.1
Host: www.example.com
Content-Type: application/json-patch+json

[
  { "op": "replace", "path": "/email", "value": "new@example.com" },
  { "op": "add",     "path": "/tags/-",  "value": "backend" },
  { "op": "remove",  "path": "/role" }
]
```

| op | Meaning |
|---|---|
| `add` | Insert or overwrite at `path` |
| `remove` | Delete the value at `path` |
| `replace` | Overwrite an existing value (fails if absent) |
| `move` | Move a value from `from` to `path` |
| `copy` | Copy a value from `from` to `path` |
| `test` | Assert `value` matches at `path`; whole patch fails if not |

The `test` op is what makes JSON Patch useful for optimistic concurrency: you can assert the current
state before applying your change.

### HEAD

Asks for a response identical to a GET request, but without the response body. Useful for checking
resource existence, size (`Content-Length`) and freshness (`Last-Modified`) cheaply — you pay for the
headers but not the payload.

```text
HEAD /index.html HTTP/1.1
Host: www.example.com

HTTP/1.1 200 OK
Content-Type: text/html
Content-Length: 1256        <-- present, but no body is sent
Last-Modified: Fri, 02 Oct 2026 16:11:02 GMT
```

### TRACE

Loops the received request back to the client so it can see what the server received, and what
intermediaries changed along the way. Used for diagnostic purposes.

- Repeated at each hop, so it can leak credentials embedded in the URL (e.g. `Basic` auth in the URL).
- Rarely enabled; historically used to diagnose content negotiation issues.
- The response body is the echoed request, sent as `message/http`.

```text
TRACE /index.html HTTP/1.1
Host: www.example.com
Max-Forwards: 0

HTTP/1.1 200 OK
Content-Type: message/http
Content-Length: 43

TRACE /index.html HTTP/1.1
Host: www.example.com
```

Real-world trace:

```console
$ curl -sS -X TRACE --max-redirs 0 http://example.com/
HTTP/1.1 405 Method Not Allowed
Allow: GET, HEAD
```

That is the answer most servers give you now: `TRACE` disabled, because it is a known attack vector.

### OPTIONS

Asks the server which communication options are available for a resource, or globally for the server.
The server replies with an `Allow` header.

```text
OPTIONS /index.html HTTP/1.1
Host: www.example.com

HTTP/1.1 200 OK
Allow: GET, HEAD, POST, OPTIONS
```

`OPTIONS *` (asterisk instead of a path) asks about the server as a whole. Browsers use this in CORS
preflight requests.

---

## Method Categories

### 1. Safe methods

Do not alter the state of the server; read-only operations.

```text
GET, HEAD, OPTIONS, TRACE, QUERY
```

Safe means *no intended side effect* — not "does nothing". A `GET` that increments a hit counter
violates the spirit of safety and confuses caches, which is why `GET` handlers should be pure reads.

### 2. Idempotent methods

The intended effect on the server of making a single request is the same as the effect of making
several identical requests.

- Includes all safe methods, plus `PUT` and `DELETE`.
- `POST` and `PATCH` are **not** guaranteed to be idempotent.
- A client can safely retry an idempotent method.

Idempotent does not mean "no side effects on the server" — it means the *client* intends none beyond
the final state. For example, the first `DELETE` returns `200`, while any successive one returns `404`;
the intended end state (the resource is gone) is the same either way.

This is why idempotency matters operationally: it is what lets a client, proxy or load balancer safely
retry a request after a timeout or a `502`, without the risk of double-charging a card or double-sending an email.

### 3. Cacheable methods

`GET` and `HEAD` can be cached. `POST` or `PATCH` requests can also be cached if freshness is
indicated and the `Content-Location` header is set, but this is rarely implemented. Whether a
*response* is cacheable is a separate question, known to the application cache.

Cacheability of a response is driven by:

- the request method (safe methods get cached by default)
- explicit response headers: `Cache-Control`, `Expires`, `ETag`, `Last-Modified`, `Vary`
- whether the response is authenticated (`Authorization` makes shared caches cautious)
- freshness vs staleness — a stale response can still be served if validators allow revalidation

---

## Status Codes

### 2xx — Success

| Code | Reason phrase | Meaning |
|---|---|---|
| 200 | OK | Success; response carries the result |
| 201 | Created | Resource created; new URI in the `Location` header |
| 203 | Non-Authoritative Information | Success, but the body was transformed by a proxy |
| 204 | No Content | Success, deliberately no body (e.g. after DELETE) |
| 206 | Partial Content | Only part of the resource, per a `Range` request |

### 3xx — Redirection

| Code | Reason phrase | Meaning |
|---|---|---|
| 301 | Moved Permanently | New permanent URI; clients and search engines should update |
| 302 | Found | Temporary redirect. Historically misused where 303 was meant |
| 303 | See Other | Redirect after a POST, so the follow-up is a `GET` |
| 304 | Not Modified | Cached copy is still fresh; no body needed |
| 307 | Temporary Redirect | Temporary, and *must not* change the request method |
| 308 | Permanent Redirect | Permanent, and *must not* change the request method |

The 307/308 rule is the important one: unlike 301/302, they preserve `POST` vs `GET` when the client
follows the redirect. `304` is what makes conditional requests cheap — the client sends `ETag`/`If-None-Match`
and gets back an empty 304 instead of the whole payload.

### 4xx — Client Error

| Code | Reason phrase | Meaning |
|---|---|---|
| 400 | Bad Request | Malformed syntax; the server refuses to guess |
| 401 | Unauthorized | **Not authenticated.** Needs credentials |
| 403 | Forbidden | Authenticated, but not allowed. Do not retry the same way |
| 404 | Not Found | Resource could not be found |
| 405 | Method Not Allowed | Method is known but unsupported here; see `Allow` |
| 410 | Gone | Deliberately removed and will not return; unlike 404, this is permanent |
| 414 | URI Too Long | Request URI exceeds server limits |
| 429 | Too Many Requests | Rate limited; retry after the delay in `Retry-After` |

401 means "who are you?", 403 means "I know who you are, and the answer is no".

### 5xx — Server Error

| Code | Reason phrase | Meaning |
|---|---|---|
| 500 | Internal Server Error | Unhandled exception; the catch-all server failure |
| 501 | Not Implemented | Server does not support the functionality required |
| 503 | Service Unavailable | Temporarily down or overloaded; often with `Retry-After` |

The distinction that matters in practice: **4xx means fix the request, 5xx means retry (possibly
elsewhere).** A load balancer in front of your service will fail over on 5xx but return 4xx straight to
the client.

---

## HTTP Messages

An HTTP message consists of:

1. Start line
2. Headers
3. Body

![Anatomy of an HTTP message](images/image.png)

### Start line

The start line comes in two forms. In a **request** it is the **request line**:

```text
<method> <request-URI> <protocol>
```

In a **response** it is the **status line**:

```text
<protocol> <status-code> <reason-phrase>
```

| Part | Request line | Status line |
|---|---|---|
| First token | Method — `GET`, `POST`, … | Protocol — `HTTP/1.1` |
| Second token | Request URI (path + query) | Status code — `200` |
| Third token | Protocol — `HTTP/1.1` | Reason phrase (optional) — `OK` |

```text
GET /index.html HTTP/1.1            <-- request line
HTTP/1.1 200 OK                      <-- status line
```

- **protocol** — HTTP version of the message, e.g. `HTTP/1.1`. In HTTP/2 and HTTP/3 this line is gone;
  the version is negotiated once at connection setup (ALPN) or in the connection preface, and frames
  carry the semantics instead.
- **status code** — numeric code, e.g. `200`
- **reason phrase** (optional) — human-readable description of the status code, e.g. `"201 (created)"`.
  Clients should ignore it; only the code carries meaning, and the phrase is often empty in HTTP/2.

### Body

The body may be delimited in three ways, and this is where most protocol confusion comes from:

| Mechanism | How | Notes |
|---|---|---|
| `Content-Length` | Explicit byte count | The body is exactly N bytes. Beware lying servers |
| Chunked transfer encoding | `Transfer-Encoding: chunked` | HTTP/1.1 only; size `<hex>\r\n<data>\r\n` … `0\r\n\r\n` |
| Close-delimited | Connection is closed to signal the end | HTTP/1.0 style; cannot be reused for pipelining |

### Header structure

```text
Header-Name: field value\r\n
```

Field names are case-insensitive (`Content-Type` = `content-type`), and leading/trailing whitespace
after the colon is stripped. Historically a space was required after the colon; HTTP/1.1 relaxed that
so `Name:value` is legal.

---

## Headers

### Request Headers

See: <https://flaviocopes.com/http-request-headers/>

| Header | Purpose |
|---|---|
| `Host` | Which server on this IP is wanted — required in HTTP/1.1, and the basis of virtual hosting. Not sent in HTTP/2+, where `Host` becomes `:authority` in the HPACK pseudo-header |
| `User-Agent` | Identifies the client software, e.g. `Mozilla/5.0 ...` |
| `Accept` | Which media types are acceptable, e.g. `text/html,application/json;q=0.9` |
| `Accept-Encoding` | Which content encodings are acceptable, e.g. `gzip, deflate, br;q=1.0` |
| `Accept-Language` | Preferred natural languages, e.g. `en-GB,en;q=0.9` |
| `Authorization` | Credentials, typically `Bearer <token>` — see [Authentication](Authentication.md) |
| `Cookie` | Cookies previously set by the server — see [Cookies and Sessions](Cookies-and-Sessions.md) |
| `Content-Type` | Media type of the request body |
| `Content-Length` | Size of the request body in bytes |
| `Connection` | Connection-specific options, e.g. `keep-alive`, `close`. Hop-by-hop; removed in HTTP/2 |
| `Origin` | Which site initiated the request — used by CORS |
| `If-None-Match` / `If-Modified-Since` | Conditional request; enables `304` |

The `q=` values weight preferences: `text/html,application/json;q=0.9` means "I want HTML, and JSON is
an acceptable fallback at 90% of the same preference."

### Response Headers

See: <https://flaviocopes.com/http-response-headers/>

| Header | Purpose |
|---|---|
| `Content-Type` | Media type of the response body, e.g. `text/html; charset=utf-8` |
| `Content-Length` | Size of the response body in bytes |
| `Location` | Where to find the resource — used with 201 Created and 3xx redirects |
| `Allow` | Which methods the resource supports — the response to `OPTIONS` |
| `Cache-Control` | Caching directives for clients and proxies, e.g. `no-store`, `max-age=3600`, `private` |
| `ETag` | Opaque validator for the current representation; used with `If-None-Match` |
| `Last-Modified` | Last modification timestamp; used with `If-Modified-Since` |
| `Set-Cookie` | Instructs the client to store a cookie |
| `Server` | Server software, e.g. `cloudflare` |
| `Retry-After` | How long to wait, with 429 or 503 — see [Rate Limiting](Rate-Limiting.md) |
| `Vary` | Which request headers affect the response — tells caches what to key on |
| `Access-Control-Allow-Origin` | Which origin may read the response (CORS) |
| `Connection` | Hop-by-hop options. Not allowed in HTTP/2 |

### Representation Headers

Describe how to interpret the data contained in the message.

| Header | Purpose |
|---|---|
| `Content-Length` | Size of the message body in bytes |
| `Content-Range` | Byte range of a partial representation, e.g. `bytes 0-499/1256` |
| `Content-Type` | Media type of the body, e.g. `text/html; charset=utf-8` |
| `Content-Encoding` | Encoding applied to the body, e.g. `gzip`, `br` |
| `Content-Location` | URI of the specific representation |
| `Content-Language` | Natural language(s) of the intended audience |

The distinction worth internalising: **`Content-Encoding` is about transport compression, not
language.** A gzipped HTML page is `Content-Encoding: gzip` and `Content-Type: text/html`. Get these
backwards and browsers will render garbage or refuse to decompress.

---

## Media Types

A MIME type (media type) says what the body actually *is*, so the receiver knows how to interpret it.
Format: `type/subtype`, optionally followed by parameters.

```text
Content-Type: text/html; charset=utf-8
              └─┬─┘ └────┬───┘
              type    parameter
```

The `type` is either `application` (data meant for a program) or `text` (data meant to be read by
humans), plus `multipart` for compound bodies.

### Common media types

| Media type | Used for | Notes |
|---|---|---|
| `application/json` | JSON payloads | The default for APIs. `charset` is always UTF-8 |
| `application/x-www-form-urlencoded` | HTML form submission | What `<form>` sends by default |
| `multipart/form-data` | Form submission with files | The only way to upload a file via `<form>` |
| `application/octet-stream` | Unspecified binary | The "I don't know" type |
| `application/xml`, `application/problem+json` | XML; structured API errors | RFC 9457 problem details |
| `application/graphql` | GraphQL | Usually as `POST`, occasionally `GET` for queries |
| `text/html` | HTML pages | |
| `text/plain` | Plain text | |
| `text/css`, `text/javascript` | Stylesheets, scripts | `text/javascript` is now the standard name |
| `image/jpeg`, `image/png`, `image/webp`, `image/avif`, `image/svg+xml` | Images | SVG is XML text; the rest are binary |
| `video/mp4`, `audio/mpeg` | Media | Range requests are essential for seeking |
| `application/pdf`, `application/zip` | Documents, archives | |
| `text/event-stream` | Server-Sent Events | |
| `application/x-ndjson` | Newline-delimited JSON | For streaming LLM-style responses |

Structured suffixes: `+json`, `+xml`, `+zip` and so on mean "a dialect of this format". They signal to
the client that the body is still the base format.

### multipart

`multipart/*` types carry a body divided into independently-typed parts, separated by a boundary
string. This is what file upload looks like on the wire:

```text
POST /upload HTTP/1.1
Content-Type: multipart/form-data; boundary=----WebKitFormBoundary7MA4YWxk
Content-Length: 234

------WebKitFormBoundary7MA4YWxk
Content-Disposition: form-data; name="title"

saw blade
------WebKitFormBoundary7MA4YWxk
Content-Disposition: form-data; name="photo"; filename="blade.jpg"
Content-Type: image/jpeg

<binary>
------WebKitFormBoundary7MA4YWxk--
```

Note the terminating `--` on the last boundary. Each part has its own headers — so a multipart body is
effectively a batch of small HTTP messages.

---

## Content Negotiation

Before the body is sent, request and response headers can negotiate *how* the exchange should look.
There are three axes, each with its own request header, response header and cache interaction.

| Axis | Request header | Response header | Negotiates |
|---|---|---|---|
| Media type | `Accept` | `Content-Type` | JSON vs XML vs HTML |
| Encoding | `Accept-Encoding` | `Content-Encoding` | gzip vs br vs identity |
| Language | `Accept-Language` | `Content-Language` | `en-GB` vs `de-DE` |
| Character set | (legacy) | `Content-Type; charset=` | UTF-8 vs Latin-1 |

```text
GET /report HTTP/1.1
Accept: application/json;q=0.9, text/html;q=0.8
Accept-Encoding: br, gzip;q=0.9
Accept-Language: en-GB, en;q=0.9
```

The server picks the best-supported option and says what it chose in the response:

```text
HTTP/1.1 200 OK
Content-Type: application/json
Content-Encoding: br
Content-Language: en-GB
Vary: Accept, Accept-Encoding, Accept-Language
```

### q values

Weights run 0 to 1, defaulting to `1`. `q=0` is a hard rejection, not a low preference:

```text
Accept: application/json, text/html;q=0.5, image/png;q=0
```

"JSON please, HTML if you must, and never PNG."

If the server honours nothing acceptable, it can still answer `406 Not Acceptable` — but in practice it
usually ignores `Accept` and sends its default, which is what makes `curl` output surprising at times.

### The caching trap

A negotiated response has **multiple representations**, and a cache that stored only one of them
would serve the wrong variant to the next client. `Vary` is what prevents that:

```text
Vary: Accept-Encoding
```

It tells caches to key the stored response on that request header. Two consequences:

- **Under-specified `Vary`** (`Vary: *`, or omitting a header you actually vary on) → caches serve one
  user's compressed variant to everyone. This is a real and repeatedly-exploited bug class.
- **Over-specified `Vary`** (listing rarely-changing headers) → the cache entry count multiplies and hit
  rate collapses. Every distinct combination is a separate entry.

### 406 and the pragmatic default

Servers almost never return `406`. The safer pattern is content negotiation with a sane default plus
an explicit override: honour `Accept`, fall back to your default representation, and let the client pin
a format with an explicit parameter (`/report?format=csv`). Negotiation by header is fragile; an
explicit parameter is never ambiguous.

---

## TCP/IP and HTTP

- HTTP is an **application layer** protocol.
- TCP/IP is the **internet transport** protocol.
- **TCP** — Transmission Control Protocol
- **IP** — Internet Protocol

TCP provides:

- Error-free data transportation
- In-order delivery (data will always arrive in the order it was sent)
- An unsegmented data stream (can dribble out data in any size at any time)

HTTP is layered over TCP, and uses TCP to transport its message data. Likewise, TCP is layered over IP.

![HTTP layered over TCP over IP](images/image2.png)

To talk to a program over TCP you need the IP address of the server and the TCP port number of the
specific program running on it. The URL gives you both:

```text
http://207.200.83.29:80/index.html
         ^-----------^ ^^
         IP address   port
```

- If the URL specifies a hostname, e.g. `http://www.netscape.com/index.html`, it is resolved to an IP
  address via **DNS**.
- If no port is given, the default port `80` is assumed (for `https`, `443`).

### The encapsulation stack

Each layer adds its own header as the data moves down, and strips it off on the way up:

```text
+-------------------------------------------+
|  Application (HTTP)                        |  <-- your HTTP message
|    GET /index.html HTTP/1.1                |
+------------------+------------------------+
|  Transport (TCP)  |  TCP segment          |  <-- ports, sequence + ack numbers
|    src/dst port, seq, ack, checksum        |
+------------------+------------------------+
|  Network (IP)     |  IP packet            |  <-- source/destination IP addresses
|    src/dst IP, TTL, protocol               |
+------------------+------------------------+
|  Link (Ethernet)  |  Frame                |  <-- MAC addresses, CRC
+------------------+------------------------+
|  Physical         |  Bits on the wire     |
+------------------+------------------------+
```

On the way back up it is the reverse: the server's NIC reads bits off the wire into a frame, strips the
link header, hands the packet to IP, which strips the IP header and hands the segment to TCP, which
reassembles the byte stream in order and hands the bytes to HTTP. HTTP is then the only layer that
cares what the message actually said.

Two consequences fall out of this:

1. **HTTP/1.1 over TCP needs the whole message length up front** — either `Content-Length` or chunked
   encoding — because TCP is a stream with no inherent message boundary. That is the root cause of
   request smuggling, and of HTTP/2's decision to use a length-prefixed framing format instead.
2. **Head-of-line blocking** lives at the transport layer: if one TCP segment is lost, every later
   segment waits, including unrelated requests. This is why HTTP/3 moved to QUIC over UDP — each stream
   is independent, so a single loss does not stall everything else.

---

## Telnet

Telnet lets you open a TCP connection to a port on a machine and type characters directly into that
port. The web server treats you as a web client, and any data that comes back over the TCP connection
is displayed on screen. It is a raw way to see the exact bytes of a request and a response, with no
client library hiding anything from you.

```console
$ telnet www.example.com 80
Trying 93.184.216.34...
Connected to www.example.com.
Escape character is '^]'.

GET /index.html HTTP/1.1
Host: www.example.com

HTTP/1.1 200 OK
Content-Type: text/html
Content-Length: 1256

<html><body><h1>It works!</h1></body></html>
```

> The blank line after `Host:` is required — it terminates the header block. That empty line is the
> single most common reason a hand-written request gets no response at all.

Because telnet is a plaintext line protocol, typing multi-line requests is awkward; `nc` or `curl -v`
is usually more convenient for the same job.

---

## Architectural Components of the Web

### The chain

```text
                     +-----------+
                     |  Client   |  (user agent: browser, curl, your code)
                     +-----+-----+
                           |
                     +-----v-----+     chain of intermediaries, each
                     |  Proxy    |     of which may be several hops deep
                     +-----+-----+
                           |
                   +-------v--------+
                   |  Cache (proxy) |  <-- may terminate here, no origin hit
                   +-------+--------+
                           |
                     +-----v-----+
                     |  Gateway  |  <-- protocol translation: HTTP <-> FTP / SMTP
                     +-----+-----+
                           |
                   +-------v--------+
                   |     Tunnel    |  <-- blind relay, e.g. CONNECT for HTTPS
                   +-------+--------+
                           |
                     +-----v-----------+
                     | Origin server    |  the one place that actually
                     +-----------------+  generates the representation
```

### Proxies

HTTP intermediaries that sit between clients and servers. A proxy receives all of the client's HTTP
requests and replays them to the server. Proxies are used for security, and can also filter requests
and responses.

Being an intermediary, a proxy sees every byte in both directions — which is why an unencrypted
`http://` request exposes the full URL and headers to anyone on the path. It is also why proxies are
the natural place to enforce policy, add caching, or terminate TLS for many hosts at once.

### Caches

A web cache (or caching proxy) is a special type of HTTP proxy that keeps copies of documents that
pass through it. The next client asking for the same document is served from the cache's copy, which
can be much faster than fetching it from a distant web server. HTTP defines many facilities to make
caching more effective and to regulate the freshness and privacy of cached content.

Caching is the single biggest lever on latency, and it is where most HTTP header complexity comes from.
The core problem is deciding whether a stored copy is still good enough. HTTP answers with two
mechanisms:

- **Freshness** — is it still within its lifetime? `Cache-Control: max-age=3600`, `Expires`,
  `Last-Modified`. If fresh, serve it without asking the origin.
- **Validation** — if stale, ask the origin cheaply before committing. `If-None-Match` with an `ETag`
  (a strong validator, an opaque string) or `If-Modified-Since` with `Last-Modified` (a weak validator).
  A matching validator yields `304 Not Modified` and no body.

Other controls worth knowing:

| Directive | Effect |
|---|---|
| `no-store` | Never write to a cache at all |
| `no-cache` | Cache it, but always revalidate before use |
| `must-revalidate` | Once stale, do not serve without contacting the origin |
| `private` | Only a browser may store this; shared caches must not |
| `public` | Explicitly storable by shared caches, even with auth |
| `immutable` | Will not change during its lifetime; do not revalidate |
| `Vary: Accept-Encoding` | Key the cached variant on that request header |

`private`/`public` matter mostly for authenticated content: a shared cache that stores one user's
personalised response and hands it to the next user is a data leak, and `private` is how you tell it not to.

### Gateways

Gateways are special servers that act as intermediaries for other servers. They are often used to
convert HTTP traffic to another protocol. A gateway always receives requests as if it were the origin
server for the resource, so the client may not be aware it is talking to a gateway.

Example: an HTTP/FTP gateway receives requests for FTP URIs over HTTP, but fetches the documents using
the FTP protocol. The resulting document is packed into an HTTP message and sent to the client.

A gateway is different from a proxy in that it is not a pass-through of HTTP — it is a translator, and
what arrives on the far side is a different protocol entirely.

### Tunnels

Tunnels are HTTP applications that, after setup, blindly relay raw data between two connections.
They are often used to transport non-HTTP data over one or more HTTP connections without inspecting the
data.

One popular use is carrying encrypted Secure Sockets Layer (SSL) traffic through an HTTP connection,
letting SSL traffic pass corporate firewalls that permit only web traffic. The tunnel receives an HTTP
request to establish an outgoing connection to a destination address and port, then blindly relays the
encrypted SSL traffic over the HTTP channel to that destination server.

`CONNECT` is the method that does this. It is the one method that is not a normal request/response
exchange at all — the tunnel is set up first, and after a `200 Connection Established` the connection
becomes an opaque byte pipe:

```text
CONNECT example.com:443 HTTP/1.1
Host: example.com:443

HTTP/1.1 200 Connection Established
```

Everything after that is TLS record data, relayed without interpretation.

### Agents

User agents (or just agents) are client programs that make HTTP requests on the user's behalf. Any
application that issues web requests is an HTTP agent.

The important boundary: there is no such thing as "the HTTP client built into the protocol". HTTP
defines the messages, not the program that sends them — so the client is an implementation detail,
and different agents make different choices about connection reuse, cookies, caching and redirect
following. That is the root of the difference in real-world behaviour you see across browsers.

## Webhooks and Callbacks

Polling asks "do I have news yet?" over and over. Webhooks invert it: the client tells the server where
to send updates, and the server pushes when something happens. Over HTTP, that means your service
makes an outbound `POST` to a URL the client registered.

### The request

It is an ordinary HTTP `POST` — which is exactly why webhooks are so easy to start and so easy to get
wrong.

```text
POST /webhooks/github HTTP/1.1
Host: client.example.com
Content-Type: application/json
X-Hub-Signature-256: sha256=5f2b8c...
X-GitHub-Event: push
X-GitHub-Delivery: 72d3162e-cc54-11e9-...

{"ref":"refs/heads/main","commits":[ ... ]}
```

### What makes it different from a normal API call

| Concern | Normal API call | Webhook |
|---|---|---|
| Caller | You | Your customer, from their server |
| Authentication | Your client authenticates to you | **You authenticate to them** |
| Delivery | Synchronous, you see the result | Asynchronous, fire-and-forget |
| Retries | Your choice | Theirs — expect duplicates |
| Ordering | Ordered | **Not ordered** |
| Payload | You chose the schema | You must version it or break them |

### The three properties you must design for

**At-least-once delivery.** Both sides retry, so the same event will arrive more than once. Deduplicate
on the delivery ID (`X-GitHub-Delivery`, or your own event UUID). Make the handler idempotent — a
unique constraint on your events table is usually enough.

**No ordering guarantee.** Event 2 may arrive before event 1. Never assume the latest event you receive
is the latest event that happened; include a monotonic sequence number or timestamp in the payload and
compare.

**You do not control the client.** Their endpoint may be slow, broken, or gone. Timeouts are mandatory
— an unbounded outbound request is a way for one customer to exhaust your connection pool. Respond
`2xx` fast, queue the work, and process it asynchronously.

### Signing: authenticate outbound calls

Because the webhook is delivered to *their* server, mutual verification is the only thing stopping
someone who guessed the URL from injecting fake events. Sign the raw request body with a shared secret
and let the receiver check it:

```text
X-Hub-Signature-256: sha256=<hex HMAC-SHA256 of raw body, keyed with the shared secret>
```

Two rules that matter: sign the **raw body bytes** as received (before any JSON parsing — reserialising
changes the bytes and breaks the signature), and compare with a **constant-time** comparison
(`hmac.compare_digest`) to avoid leaking the secret through timing.

Always also send an idempotency key (`X-GitHub-Delivery` above) so the receiver can deduplicate.

### Sagas

Related pattern: when one service's work must trigger a chain across others, a *saga* is a sequence of
steps each ending in a compensating action. Webhooks deliver the steps; the compensations are what make
the distributed transaction survivable.

## Types of APIs (REST and RPC)

REST and RPC are both styles of building an API on top of HTTP; the difference is what the URL and the
method mean.

| | REST | RPC |
|---|---|---|
| URLs name | Resources | Procedures / operations |
| Meaning comes from | The HTTP method | The URL path |
| Example | `PUT /users/42` | `POST /UserService.UpdateUser` |
| Granularity | Coarse, CRUD per resource | Fine, one call per operation |
| Evolves by | Adding resources and fields | Adding methods |
| Typical transport | HTTP | HTTP, or gRPC over HTTP/2 |
| Contract | OpenAPI / JSON Schema | Protobuf / IDL |

### REST

REST models the domain as **resources**, each with its own URI, manipulated with the standard HTTP
methods — which is exactly why the method semantics you learned earlier (`GET` safe, `PUT` idempotent)
matter so much. Using them correctly means intermediaries can cache, retry and prefetch correctly,
without understanding your application.

The usual constraints:

- Nouns in URLs, not verbs: `/users/42`, not `/getUser?id=42`.
- Nesting reflects containment: `GET /users/42/orders`.
- Status codes carry the outcome — not a `200` with `{"error": "..."}` in the body.
- Stateless: every request carries its own auth and parameters.
- A uniform interface: the same handful of methods everywhere.

```text
GET    /orders?status=open&limit=50
POST   /orders
GET    /orders/{orderId}
PATCH  /orders/{orderId}
DELETE /orders/{orderId}
```

### RPC

RPC calls a **named operation** directly. The URL names the procedure, the method is almost always
`POST`, and the request body carries all the arguments.

```text
POST /OrderService/GetOrder HTTP/1.1
Content-Type: application/json

{"order_id": "A-991"}
```

### gRPC

The common modern RPC style. It defines its schema in **protobuf** (`.proto`), compiles stubs in several
languages, and normally runs over HTTP/2.

```text
service OrderService {
  rpc GetOrder(GetOrderRequest) returns (Order);
}

message GetOrderRequest {
  string order_id = 1;   // field numbers, not names, go on the wire
}
```

The trade-offs: strong schema enforcement and generated clients on one side; HTTP/2 streaming and
efficient binary encoding on the other. What you give up is the transparency of plain HTTP — you can no
longer `curl` the API, and intermediaries cannot see or cache individual operations.

### Choosing

There is no universally correct answer, and it is a team and organisational question as much as a
technical one.

| Choose | When |
|---|---|
| REST / HTTP + JSON | Public or partner APIs; anything a browser or `curl` must call; you want caching and third-party accessibility; the surface is CRUD-shaped |
| gRPC | Internal service-to-service; you own both ends; you need strong contracts or streaming; low latency matters more than debuggability |
| Both | Common in practice — gRPC internally, REST at the edge. Costs you two contracts to keep in sync |

A workable default for backend work: **REST/JSON for anything outward-facing, gRPC for internal calls
you fully control.** The point at which a service becomes externally consumed is the point at which its
contract becomes expensive to change — that boundary is worth deciding deliberately.

## URLs

### General URL format

```text
scheme://user:password@host:port/path;params?query#frag
```

### URL components

| Component | Description | Default value |
|---|---|---|
| `scheme` | Which protocol to use when accessing a server to get a resource | None |
| `user` | The username some schemes require to access a resource | `anonymous` |
| `password` | The password that may be included after the username, separated by a colon (`:`) | `<Email address>` |
| `host` | The hostname or dotted IP address of the server hosting the resource | None |
| `port` | The port number on which the server hosting the resource is listening. Many schemes have default port numbers (the default for HTTP is 80) | Scheme-specific |
| `path` | The local name for the resource on the server, separated from the previous URL components by a slash (`/`). The syntax is server- and scheme-specific, and the path can be divided into segments, each with its own scheme-specific components | None |
| `params` | Used by some schemes to specify input parameters. Params are name/value pairs, separated from themselves and the rest of the path by semicolons (`;`) | None |
| `query` | Used by some schemes to pass parameters to active applications (such as databases, bulletin boards, search engines and other internet gateways). No common format. Separated from the rest of the URL by `?` | None |
| `frag` | A name for a piece or part of the resource. **Not sent to the server** — used internally by the client. Separated from the rest of the URL by `#` | None |

Only `scheme` and `host` are truly universal; most components are scheme-specific, and many are
optional. The minimal form is just `scheme://host` plus a `path`.

### Common scheme formats

| Scheme | Basic form | Example |
|---|---|---|
| `http` | `http://<host>:<port>/<path>?<query>#<frag>` | `http://www.joes-hardware.com/index.html`<br>`http://www.joes-hardware.com:80/index.html` |
| `https` | `https://<host>:<port>/<path>?<query>#<frag>` | `https://www.joes-hardware.com/secure.html` |

#### http

Conforms to the general URL format, except that there is no username or password. The port defaults to
`80` if omitted.

```text
http://<host>:<port>/<path>?<query>#<frag>
```

```text
http://www.joes-hardware.com/index.html
http://www.joes-hardware.com:80/index.html
```

Those two are equivalent — omitting the port is the same as saying `:80`.

#### https

A twin to the `http` scheme. The only difference is that `https` uses Netscape's Secure Sockets Layer
(SSL), which provides end-to-end encryption of HTTP connections. Its syntax is identical to that of
HTTP, with a default port of `443`.

```text
https://<host>:<port>/<path>?<query>#<frag>
```

```text
https://www.joes-hardware.com/secure.html
```

