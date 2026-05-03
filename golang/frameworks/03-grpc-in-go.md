---
title: "gRPC in Go"
date: 2026-05-03
updated: 2026-05-03
tags: [golang, frameworks, grpc, protobuf, streaming]
---

# gRPC in Go

**Date:** 2026-05-03 | **Updated:** 2026-05-03
**Tags:** `golang` `frameworks` `grpc` `protobuf` `streaming`

---

## Table of Contents

1. [gRPC briefly: HTTP/2, protobuf, codegen](#1-grpc-briefly-http2-protobuf-codegen)
2. [Toolchain: protoc, the Go plugins, and buf](#2-toolchain-protoc-the-go-plugins-and-buf)
3. [Defining a service in `.proto` and generating Go](#3-defining-a-service-in-proto-and-generating-go)
4. [Server: implement the generated interface](#4-server-implement-the-generated-interface)
5. [Client: `grpc.NewClient` and the `Dial` deprecation](#5-client-grpcnewclient-and-the-dial-deprecation)
6. [Streaming: server, client, bidirectional](#6-streaming-server-client-bidirectional)
7. [Interceptors: auth, logging, retry, panic recovery](#7-interceptors-auth-logging-retry-panic-recovery)
8. [Deadline propagation through `context.Context`](#8-deadline-propagation-through-contextcontext)
9. [Error model: `codes`, `status`, mapping to HTTP](#9-error-model-codes-status-mapping-to-http)
10. [TLS, mTLS, and the credentials package](#10-tls-mtls-and-the-credentials-package)
11. [grpc-gateway and connect-go: HTTP/JSON edges](#11-grpc-gateway-and-connect-go-httpjson-edges)
12. [When to choose gRPC over REST/JSON](#12-when-to-choose-grpc-over-restjson)

## Summary

gRPC is the canonical Go RPC framework — Go is a first-class supported language and gRPC's reference implementation lives in the `grpc-go` repository. Compared to REST/JSON, you trade ad-hoc payload shape for a strict protobuf schema, ad-hoc HTTP semantics for a typed status code system, and code-by-convention for code-by-codegen. In return you get streaming as a first-class concept, deadline propagation that actually works end-to-end, and contract-first development that scales across teams.

The Go ergonomics are excellent: protobuf compiles to a Go file you import, the service definition compiles to an interface you implement, and the runtime threads `context.Context` through every call. The mental model is "implement an interface, register it on a server" — exactly the same shape as the [`net/http` Handler](01-net-http-deep-dive.md#1-handler-is-a-one-method-interface) story, with codegen filling in the boilerplate.

This doc covers the toolchain, the wire-up, the streaming modes, the interceptor middleware model, the error model, TLS, and the two common HTTP/JSON adapters (grpc-gateway and connect-go).

## 1. gRPC briefly: HTTP/2, protobuf, codegen

Three pillars:

| Pillar | What it gives you |
|---|---|
| **HTTP/2 transport** | Multiplexed streams over one TCP connection, header compression, native support for streaming bodies (server, client, bidi). |
| **Protocol Buffers serialization** | Compact binary format, schema-driven. Field numbers (not names) are the wire identity, which is what makes backward compatibility tractable. |
| **Codegen-driven service definitions** | A `.proto` file describes services, methods, and messages. Tools generate idiomatic Go: an interface to implement on the server side, a typed client struct on the caller side. |

The wire is HTTP/2 framing carrying length-prefixed protobuf messages, with metadata in HTTP/2 headers and trailers. You almost never deal with the wire directly — you implement an interface, the runtime handles the rest.

This is fundamentally different from REST/JSON: there is one schema, one transport, one error model, and the client and server share generated code derived from that schema.

## 2. Toolchain: protoc, the Go plugins, and buf

The traditional toolchain has three pieces:

- **`protoc`** — the Protocol Buffers compiler (C++ binary). Reads `.proto` files, dispatches to language plugins.
- **`protoc-gen-go`** — generates Go types from `.proto` messages.
- **`protoc-gen-go-grpc`** — generates Go service stubs (server interface + client) for gRPC services.

Install per the gRPC Go quickstart:

```bash
go install google.golang.org/protobuf/cmd/protoc-gen-go@latest
go install google.golang.org/grpc/cmd/protoc-gen-go-grpc@latest
export PATH="$PATH:$(go env GOPATH)/bin"
```

Generate once you have a `.proto`:

```bash
protoc --go_out=. --go_opt=paths=source_relative \
       --go-grpc_out=. --go-grpc_opt=paths=source_relative \
       helloworld/helloworld.proto
```

This produces `helloworld.pb.go` (messages) and `helloworld_grpc.pb.go` (service code).

**buf** is the higher-level alternative most teams settle on. It wraps protoc with:

- A `buf.yaml` / `buf.gen.yaml` config so you don't memorize flag soup.
- A linter for `.proto` files (naming, package layout, breaking-style rules).
- Breaking-change detection against a previous version (`buf breaking`).
- A schema registry (BSR) for sharing protos across repos without git submodules.

For a single repo with one or two services, raw protoc + a `Makefile` works. Once you have multiple services, multiple repos, or external consumers, buf pays for itself fast. The buf docs position it as "Protobuf on easy mode" — accurate for what it removes from your daily workflow.

## 3. Defining a service in `.proto` and generating Go

A minimal service:

```protobuf
syntax = "proto3";

package greet.v1;

option go_package = "example.com/api/greet/v1;greetv1";

service Greeter {
  rpc SayHello(SayHelloRequest) returns (SayHelloResponse);
}

message SayHelloRequest {
  string name = 1;
}

message SayHelloResponse {
  string message = 1;
}
```

Versioning the package (`greet.v1`) is conventional — when you make a breaking change, you publish `greet.v2` alongside.

After running protoc, you get (roughly):

```go
// greetv1.pb.go (messages)
type SayHelloRequest struct  { Name string `protobuf:"..."` /* ... */ }
type SayHelloResponse struct { Message string /* ... */ }

// greetv1_grpc.pb.go (service)
type GreeterServer interface {
    SayHello(context.Context, *SayHelloRequest) (*SayHelloResponse, error)
    mustEmbedUnimplementedGreeterServer()
}

type GreeterClient interface {
    SayHello(ctx context.Context, in *SayHelloRequest, opts ...grpc.CallOption) (*SayHelloResponse, error)
}
```

`mustEmbedUnimplementedGreeterServer` is the forward-compat trick: by embedding `UnimplementedGreeterServer` in your concrete server, new methods added to the proto don't break compilation — they just fall back to "unimplemented" until you implement them. This is unusual in Go style but is the convention here.

## 4. Server: implement the generated interface

Implementing the server is identical in shape to implementing any Go interface — see [Interfaces & Structural Typing](../fundamentals/05-interfaces-and-structural-typing.md):

```go
import (
    "context"
    "log"
    "net"

    "google.golang.org/grpc"
    greetv1 "example.com/api/greet/v1"
)

type greeter struct {
    greetv1.UnimplementedGreeterServer // forward-compat
}

func (greeter) SayHello(ctx context.Context, req *greetv1.SayHelloRequest) (*greetv1.SayHelloResponse, error) {
    return &greetv1.SayHelloResponse{Message: "hello " + req.Name}, nil
}

func main() {
    lis, err := net.Listen("tcp", ":50051")
    if err != nil { log.Fatal(err) }

    s := grpc.NewServer()
    greetv1.RegisterGreeterServer(s, greeter{})
    log.Fatal(s.Serve(lis))
}
```

`grpc.NewServer()` returns the analogue of `http.Server`. `RegisterGreeterServer` plumbs your implementation into the dispatcher. `s.Serve(lis)` runs forever, spawning one goroutine per request — same one-goroutine-per-call story as `net/http`, see [Goroutines & the Scheduler](../concurrency/01-goroutines-and-scheduler.md).

For graceful shutdown, `s.GracefulStop()` mirrors `http.Server.Shutdown`: stop accepting, drain in-flight calls, return.

## 5. Client: `grpc.NewClient` and the `Dial` deprecation

The client side has changed in recent grpc-go versions. The current API:

```go
import (
    "google.golang.org/grpc"
    "google.golang.org/grpc/credentials/insecure"
)

conn, err := grpc.NewClient("localhost:50051",
    grpc.WithTransportCredentials(insecure.NewCredentials()),
)
if err != nil { log.Fatal(err) }
defer conn.Close()

client := greetv1.NewGreeterClient(conn)

ctx, cancel := context.WithTimeout(context.Background(), 5*time.Second)
defer cancel()

resp, err := client.SayHello(ctx, &greetv1.SayHelloRequest{Name: "world"})
```

Per the package documentation, `NewClient` is the current API:

> NewClient creates a new gRPC "channel" for the target URI provided. No I/O is performed. Use of the ClientConn for RPCs will automatically cause it to connect.

`grpc.Dial` and `grpc.DialContext` are deprecated, with the docs stating:

> Deprecated: use NewClient instead. Will be supported throughout 1.x.

The differences that matter:
- `Dial` blocked until the connection was established (or used `WithBlock` to opt in). `NewClient` is lazy — no I/O happens until the first RPC.
- `Dial` had a tangled set of default options (default service config, default name resolver) that surprised people. `NewClient` has stricter, less-magical defaults.

If you read older blog posts or library code, you'll see `grpc.Dial(...)` everywhere. Prefer `NewClient` in new code.

`insecure.NewCredentials()` is the explicit way to say "no TLS." The package was renamed and isolated specifically so this choice is loud — see §10 for the real-world TLS setup.

## 6. Streaming: server, client, bidirectional

Beyond unary (request/response), gRPC supports three streaming modes. The streaming syntax in `.proto`:

```protobuf
service ChatService {
  // server streaming: one request, stream of responses
  rpc Subscribe(SubscribeRequest) returns (stream Event);

  // client streaming: stream of requests, one response
  rpc Upload(stream Chunk) returns (UploadResponse);

  // bidirectional streaming
  rpc Chat(stream Message) returns (stream Message);
}
```

**Server streaming** — the server pushes a sequence of messages:

```go
func (s *server) Subscribe(req *pb.SubscribeRequest, stream pb.ChatService_SubscribeServer) error {
    for event := range eventsFor(req.Topic) {
        if err := stream.Send(&pb.Event{Payload: event}); err != nil {
            return err
        }
    }
    return nil
}
```

The handler returns when there are no more events; gRPC closes the stream cleanly. The stream's `Context()` is canceled if the client disconnects, which is what you `select` on in long-lived subscribers.

**Client streaming** — the client uploads a sequence, gets one response:

```go
func (s *server) Upload(stream pb.ChatService_UploadServer) error {
    var total int64
    for {
        chunk, err := stream.Recv()
        if errors.Is(err, io.EOF) {
            return stream.SendAndClose(&pb.UploadResponse{TotalBytes: total})
        }
        if err != nil { return err }
        total += int64(len(chunk.Data))
    }
}
```

`io.EOF` from `Recv()` signals "client done sending" — same `io.EOF` convention as the standard library's `io.Reader`. See [Errors as Values](../fundamentals/06-errors-as-values.md).

**Bidirectional** — both sides stream independently:

```go
func (s *server) Chat(stream pb.ChatService_ChatServer) error {
    for {
        msg, err := stream.Recv()
        if errors.Is(err, io.EOF) { return nil }
        if err != nil { return err }
        // process msg, then send a reply (or many)
        if err := stream.Send(reply(msg)); err != nil {
            return err
        }
    }
}
```

Inside the loop you can fan out the receive and send to two goroutines if you need them to proceed independently — gRPC's stream is safe for concurrent `Send` and concurrent `Recv` (one goroutine per direction; do **not** call `Send` from two goroutines simultaneously).

When to use which:
- **Server streaming**: subscriptions, log tails, slow paginated results, server-pushed updates.
- **Client streaming**: uploads where the total size is large or unknown, batching writes from a producer.
- **Bidirectional**: chat protocols, interactive sessions, long-lived control planes.

Streaming requires HTTP/2, so it does not pass through HTTP/1.1 reverse proxies or load balancers cleanly. Plan your edge accordingly.

## 7. Interceptors: auth, logging, retry, panic recovery

Interceptors are gRPC's middleware. They come in four shapes — unary/stream × server/client.

```go
type UnaryServerInterceptor  func(ctx context.Context, req any, info *UnaryServerInfo, handler UnaryHandler) (any, error)
type StreamServerInterceptor func(srv any, ss ServerStream, info *StreamServerInfo, handler StreamHandler) error
type UnaryClientInterceptor  func(ctx context.Context, method string, req, reply any, cc *ClientConn, invoker UnaryInvoker, opts ...CallOption) error
type StreamClientInterceptor func(ctx context.Context, desc *StreamDesc, cc *ClientConn, method string, streamer Streamer, opts ...CallOption) (ClientStream, error)
```

Logging interceptor (server, unary):

```go
func loggingInterceptor(ctx context.Context, req any, info *grpc.UnaryServerInfo, handler grpc.UnaryHandler) (any, error) {
    start := time.Now()
    resp, err := handler(ctx, req)
    log.Printf("%s took %s err=%v", info.FullMethod, time.Since(start), err)
    return resp, err
}

s := grpc.NewServer(
    grpc.UnaryInterceptor(loggingInterceptor),
)
```

For multiple interceptors use `grpc.ChainUnaryInterceptor` and `grpc.ChainStreamInterceptor` (single registration accepts only one of each).

Common interceptor uses:

| Use | Where |
|---|---|
| Auth — extract token, validate, attach principal to context | Server, unary + stream |
| Logging / metrics — record `info.FullMethod`, status, duration | Server |
| Tracing — extract / inject `traceparent` headers | Both |
| Retry / hedging on idempotent methods | Client |
| Panic recovery — recover, return `Internal` status | Server (`grpc-ecosystem/go-grpc-middleware/recovery`) |
| Request validation — call `req.Validate()` if `protovalidate`-generated | Server |

The `grpc-ecosystem/go-grpc-middleware` repo provides production-quality interceptors for most of the above.

## 8. Deadline propagation through `context.Context`

The single biggest day-to-day win of gRPC over REST/JSON is automatic deadline propagation.

When the client calls with a context that has a deadline:

```go
ctx, cancel := context.WithTimeout(context.Background(), 2*time.Second)
defer cancel()
resp, err := client.SayHello(ctx, req)
```

The deadline is serialized into the request as a `grpc-timeout` header. On the server side, the `context.Context` you receive **already has that deadline applied**. If your handler calls a downstream service or database, you pass `ctx` straight through and the deadline carries on:

```go
func (s *server) SayHello(ctx context.Context, req *pb.SayHelloRequest) (*pb.SayHelloResponse, error) {
    // ctx already has the client's 2s deadline
    row := s.db.QueryRowContext(ctx, "SELECT ...")
    // ...
}
```

If anything in the chain blows the deadline, the failing call returns a `DeadlineExceeded` error and the whole tree unwinds. There is no manual budget tracking, no "remaining time" math. This is what people mean when they say gRPC has "real" cancellation — it is end-to-end, free, and consistent.

The full context model (deadlines, cancellation, value propagation, the `context.Background` / `context.TODO` distinction) is covered in [Context Propagation](../concurrency/04-context-propagation.md).

## 9. Error model: `codes`, `status`, mapping to HTTP

gRPC errors are not free-form strings — they are typed with a fixed set of canonical codes from `google.golang.org/grpc/codes`:

| Code | When |
|---|---|
| `OK` (0) | Success |
| `Canceled` (1) | Caller canceled |
| `Unknown` (2) | Unknown error |
| `InvalidArgument` (3) | Bad client argument |
| `DeadlineExceeded` (4) | Deadline blew |
| `NotFound` (5) | Entity not found |
| `AlreadyExists` (6) | Duplicate |
| `PermissionDenied` (7) | Authorized but not allowed |
| `ResourceExhausted` (8) | Quota / rate limit |
| `FailedPrecondition` (9) | System not in required state |
| `Aborted` (10) | Concurrency conflict |
| `OutOfRange` (11) | Past valid range |
| `Unimplemented` (12) | Method not implemented |
| `Internal` (13) | Internal invariant broken |
| `Unavailable` (14) | Try again later |
| `DataLoss` (15) | Unrecoverable corruption |
| `Unauthenticated` (16) | Missing / invalid credentials |

Returning a typed error from a handler:

```go
import (
    "google.golang.org/grpc/codes"
    "google.golang.org/grpc/status"
)

func (s *server) GetUser(ctx context.Context, req *pb.GetUserRequest) (*pb.User, error) {
    u, err := s.repo.Find(ctx, req.Id)
    if errors.Is(err, ErrUserNotFound) {
        return nil, status.Errorf(codes.NotFound, "user %s not found", req.Id)
    }
    if err != nil {
        return nil, status.Errorf(codes.Internal, "lookup: %v", err)
    }
    return u, nil
}
```

Returning a plain `error` (not `status.Error`) gets wrapped as `codes.Unknown`, which is almost always wrong. Always wrap.

Mapping to HTTP at gateway boundaries (REST adapters, grpc-gateway) follows a documented table — `NotFound` → 404, `PermissionDenied` → 403, `Unauthenticated` → 401, `ResourceExhausted` → 429, `Unavailable` → 503, `Internal` → 500. Both grpc-gateway and connect-go handle this automatically.

For richer error details (validation field paths, retry-after hints), gRPC supports `status.Details` — typed messages attached to the status. This is the moral equivalent of an RFC 7807 `application/problem+json` payload, but typed.

## 10. TLS, mTLS, and the credentials package

In production, TLS is mandatory. The `credentials` package wraps the configuration:

```go
import (
    "crypto/tls"
    "google.golang.org/grpc"
    "google.golang.org/grpc/credentials"
)

// Server side
creds, err := credentials.NewServerTLSFromFile("server.crt", "server.key")
if err != nil { log.Fatal(err) }
s := grpc.NewServer(grpc.Creds(creds))

// Client side
creds, err := credentials.NewClientTLSFromFile("ca.crt", "")
if err != nil { log.Fatal(err) }
conn, err := grpc.NewClient("server.example.com:443", grpc.WithTransportCredentials(creds))
```

For mutual TLS — the server requires a client certificate signed by a CA it trusts, and vice versa — build the `tls.Config` directly:

```go
serverCert, _ := tls.LoadX509KeyPair("server.crt", "server.key")
caPool := x509.NewCertPool()
caBytes, _ := os.ReadFile("ca.crt")
caPool.AppendCertsFromPEM(caBytes)

cfg := &tls.Config{
    Certificates: []tls.Certificate{serverCert},
    ClientCAs:    caPool,
    ClientAuth:   tls.RequireAndVerifyClientCert,
}
s := grpc.NewServer(grpc.Creds(credentials.NewTLS(cfg)))
```

mTLS is the typical inter-service auth in service meshes (Istio, Linkerd). Outside a mesh, you usually pair plain TLS with token-based auth (OIDC bearer tokens, mTLS only on the perimeter).

For development and tests, `insecure.NewCredentials()` makes the absence of TLS explicit and loud. See [Security — TLS, auth](../../security/INDEX.md) and [Networking — TLS, HTTP/2](../../networking/INDEX.md) for the deeper background.

## 11. grpc-gateway and connect-go: HTTP/JSON edges

gRPC-only services lose two things: browser clients and `curl`. Two projects address this differently.

**grpc-gateway** generates a reverse proxy. From the project README:

> reads protobuf service definitions and generates a reverse-proxy server which translates a RESTful HTTP API into gRPC

You annotate your `.proto` with `google.api.http` rules:

```protobuf
service Greeter {
  rpc SayHello(SayHelloRequest) returns (SayHelloResponse) {
    option (google.api.http) = {
      post: "/v1/hello"
      body: "*"
    };
  }
}
```

Run an extra protoc plugin (`protoc-gen-grpc-gateway`), and you get a separate Go binary (or a `mux` you mount in your existing process) that serves HTTP/JSON, translates incoming requests to gRPC, calls the gRPC backend, and translates responses back. The architecture is two processes: the gRPC server, and the gateway in front of it. Browser clients talk JSON to the gateway; service-to-service calls go straight to gRPC.

**connect-go** is the alternative, by Buf. From the README:

> Connect-Go is the Go implementation of Connect, a library for constructing browser and gRPC-compatible HTTP APIs.

> Handlers and clients also support the gRPC and gRPC-Web protocols, including streaming, headers, trailers, and error details.

Connect speaks three protocols on a single endpoint: its own protocol (HTTP/1.1 + HTTP/2 friendly), gRPC, and gRPC-Web. You implement a Connect handler — same shape as a gRPC handler — and any client (gRPC, gRPC-Web, plain `curl` with JSON) can call it. Quoting the README:

> Calling a Connect API is as easy as using `curl`.

The choice:

| Need | Pick |
|---|---|
| Existing gRPC service, want to expose JSON edge to browsers | grpc-gateway |
| Greenfield, want one process serving gRPC + JSON + browser | connect-go |
| Want strict gRPC interop and grpc-go is already in use | grpc-gateway (uses real gRPC under the hood) |
| Schema-first ergonomics, lighter than full gRPC | connect-go |

connect-go has been gaining momentum; it sits cleanly on top of `net/http` and feels lighter than the gRPC stack while staying schema-first.

## 12. When to choose gRPC over REST/JSON

gRPC pays off when:

- **Internal service-to-service** is a primary concern. Strict contracts, generated clients, end-to-end deadlines, streaming.
- **Polyglot teams**. The schema is the source of truth across Go, Java, Python, TS, Swift — codegen reaches all of them.
- **Streaming** is in scope. Server push for subscriptions, client streaming for uploads, bidi for chat-like protocols.
- **Wire size or CPU matters**. Protobuf is dramatically more compact and cheaper to parse than JSON for large or hot payloads.
- **API evolution is real**. Field numbers + the protobuf compatibility rules + buf's breaking-change checker make backward compatibility tractable in a way ad-hoc JSON never reaches.

REST/JSON stays a better fit when:

- **Public API for a wide audience.** Documentation, debuggability with `curl`, broad tooling support.
- **Browser clients.** gRPC-Web exists and works, but plain JSON over fetch is still simpler. Connect's protocol is the middle ground.
- **CDN cacheability.** REST/JSON's plain GET-with-URL is cacheable in places gRPC is not.
- **Small services / small teams** where the codegen + tooling overhead does not pay back.

A common pattern in mature Go shops: **gRPC east-west (between services) + REST/JSON north-south (to browsers and external partners)**, with grpc-gateway or connect-go converting between them at the edge. This gets you the internal benefits without forcing JSON consumers through a binary protocol.

For the choice between routers, frameworks, and gRPC at the architecture level, weigh this against [`net/http` Deep Dive §10](01-net-http-deep-dive.md#10-when-you-actually-need-a-framework) and [chi vs gin vs echo vs fiber §7](02-chi-vs-gin-vs-echo-vs-fiber.md#7-decision-matrix--what-to-pick-when).

## Related

- [`net/http` Deep Dive](01-net-http-deep-dive.md) — the HTTP/1.1 + HTTP/2 path you bypass with gRPC
- [chi vs gin vs echo vs fiber](02-chi-vs-gin-vs-echo-vs-fiber.md) — the routers gRPC replaces
- [Interfaces & Structural Typing](../fundamentals/05-interfaces-and-structural-typing.md) — implementing the generated server interface
- [Errors as Values](../fundamentals/06-errors-as-values.md) — `status.Error`, `errors.Is`, `io.EOF` in streams
- [Goroutines & the Scheduler](../concurrency/01-goroutines-and-scheduler.md) — one goroutine per RPC
- [Context Propagation](../concurrency/04-context-propagation.md) — the deadline-propagation backbone
- [NestJS for Spring devs](../../typescript/frameworks/nestjs-for-spring-devs.md) — TS RPC analogues (tRPC, NestJS gRPC microservices)
- [Networking — HTTP/2, TLS](../../networking/INDEX.md)
- [Security — TLS, auth](../../security/INDEX.md)

## References

- gRPC Go quickstart — https://grpc.io/docs/languages/go/quickstart/
- grpc-go (reference Go implementation) — https://github.com/grpc/grpc-go
- `google.golang.org/grpc` package — https://pkg.go.dev/google.golang.org/grpc
- `google.golang.org/grpc/codes` (status code definitions) — https://pkg.go.dev/google.golang.org/grpc/codes
- `google.golang.org/grpc/status` — https://pkg.go.dev/google.golang.org/grpc/status
- `protocolbuffers/protobuf-go` — https://github.com/protocolbuffers/protobuf-go
- gRPC status code reference (canonical list across languages) — https://github.com/grpc/grpc/blob/master/doc/statuscodes.md
- buf — https://buf.build/
- `grpc-ecosystem/go-grpc-middleware` (interceptors: auth, logging, recovery, retry) — https://github.com/grpc-ecosystem/go-grpc-middleware
- grpc-gateway — https://github.com/grpc-ecosystem/grpc-gateway
- connect-go — https://github.com/connectrpc/connect-go
- Protocol Buffers language guide (proto3) — https://protobuf.dev/programming-guides/proto3/
