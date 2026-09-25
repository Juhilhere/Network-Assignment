# JC/1 Binary HTTP Protocol

## 1. Purpose and connection rules

JC/1 carries a small HTTP-like request and response stream over one TCP connection. The client opens one connection, sends one or more request frames in order, reads one response for each, and closes after the final response. The server keeps the connection open until the peer closes it, then accepts another connection. All integers are unsigned big-endian unless stated otherwise.

## 2. Frame layout

Every frame starts with this fixed 12-byte header:

| Offset | Width | Field |
|---:|---:|---|
| 0 | 2 | ASCII magic `JC` |
| 2 | 1 | version `1` |
| 3 | 1 | type: `1` request, `2` response |
| 4 | 1 | flags, currently `0` |
| 5 | 1 | encoded header count |
| 6 | 2 | reserved, must be zero |
| 8 | 4 | payload length |

The payload length is the number of bytes after the header. A receiver reads exactly that many bytes before looking for the next frame. An unknown frame type is skipped by consuming its declared payload and then parsing the next frame. A bad magic, version, reserved field, or truncated payload is malformed; a complete malformed request receives status `400`.

## 3. Header block

Each header is encoded as `name-index` followed by a two-byte value length and value bytes. Indexes 1 through 10 use this static table; index 0 is followed by a one-byte literal name length and name bytes.

| Index | Name | Index | Name |
|---:|---|---:|---|
| 1 | host | 6 | connection |
| 2 | user-agent | 7 | server |
| 3 | accept | 8 | date |
| 4 | content-length | 9 | cache-control |
| 5 | content-type | 10 | transfer-encoding |

The request header count is followed by a four-byte body length and body bytes. A response payload starts with a two-byte status, followed by its headers, a four-byte body length, and body bytes. Requests require a non-empty `host` header and a zero-length body.

## 4. Application behavior

`GET /add?a=2&b=3`, `/sub`, `/mul`, and `/div` require exactly one integer `a` and one integer `b`. A successful response is `200` with the decimal result and no trailing newline. Division by zero, missing or repeated parameters, and invalid integers are `400`. Unknown routes such as `/pow` are `404`; unsupported methods such as `POST` are `405`.

Any other GET path is resolved under the server's document root. An existing regular file returns `200` with its exact bytes. A missing file returns `404`. Paths containing `..` or outside the root are `400`. Responses include `content-length`, `content-type`, `connection: keep-alive`, and `server` headers.

## 5. Commands and interoperability

```text
serve <root> <port>
b curl [-v] host:port/path [additional paths...]
```

`b` writes response bodies to stdout, writes complete request and response hexdumps to stderr with `-v`, and exits non-zero on 4xx/5xx. Both programs implement the same wire contract; the client does not depend on server-private behavior.

