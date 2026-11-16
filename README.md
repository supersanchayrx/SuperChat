# SuperChat

A real-time chat app with a React Native (Expo) mobile client, a Go WebSocket server and PostgreSQL. It is offline-first: a message is written to an on-device SQLite outbox before it is sent, so it survives dropped connections and app restarts, and the server's idempotent inserts stop retries from creating duplicates. It runs on AWS EC2 and is deployed with Docker and GitHub Actions.

> **Status:** work in progress. The design and roadmap are below. Results will be added here as each part lands.

## Why

A chat screen looks like a list and a text box. On a phone with a flaky network, it has to make a few decisions that decide whether messages arrive once, in order, and quickly:

- **Why a socket, not polling.** Polling every few seconds adds that much delay to every message and wakes the radio for nothing. A WebSocket keeps one connection open, so the server pushes a message the moment it is stored.
- **Why every message carries a client-generated ID.** A send can time out after the server has already stored it. The client retries, and without an ID the server stores it twice. With a unique `(sender, client_msg_id)` key, the retry returns the existing message instead of creating a new one.
- **Why per-conversation sequence numbers.** Timestamps from different phones disagree, and two messages can share a millisecond. The server gives each message the next number in its conversation, inside the same transaction as the insert. A reconnecting client asks for "everything after N" and can tell when it has a gap.
- **Why an outbox.** The user hits send in a lift. The message is saved to SQLite first and shown as pending, then sent when the socket is back, even if the app was killed in between.
- **Why heartbeats.** Mobile networks leave half-open TCP connections that look alive but deliver nothing. Ping/pong with a timeout finds them, so the client reconnects instead of waiting forever.
- **Why push notifications.** The OS suspends a backgrounded app's socket. When a recipient has no live connection, the server sends a push notification instead.

## Design

| Component | What it does |
|---|---|
| **Mobile client** | Expo (SDK 58) + TypeScript, expo-router screens (sign-in, chat list, chat). Messages are cached in expo-sqlite, so the app opens instantly and works offline |
| **Outbox and reconnect** | Pending messages wait in an SQLite outbox table. The socket client reconnects with exponential backoff and jitter, reconnects when the app returns to the foreground, then syncs and flushes the outbox in order |
| **Protocol** | JSON frames over one WebSocket: `auth`, `send`, `ack`, `message`, `sync` (after a sequence number), `typing`, `read`, `presence`, `ping`/`pong` |
| **Server** | Go, `net/http` + `github.com/coder/websocket`. Each connection has a reader and a writer goroutine with a bounded send queue; a slow consumer is disconnected instead of stalling the hub. A user can be connected from several devices |
| **Storage** | PostgreSQL via pgx: users, conversations, members, messages (unique `(sender_id, client_msg_id)`, unique `(conversation_id, seq)`), read cursors and push tokens. `seq` comes from `UPDATE conversations SET last_seq = last_seq + 1 ... RETURNING last_seq` in the insert's transaction |
| **Auth** | bcrypt password hashes, short-lived JWT access tokens and rotating refresh tokens. The token goes in the first socket frame, not the URL, so it never lands in access logs |
| **History** | REST endpoints with cursor pagination by sequence number |
| **Receipts and presence** | Delivered and read receipts from per-member read cursors; typing indicators and online status are relayed, not stored |
| **Push** | The Expo Push API notifies members with no live socket. Tokens are stored per device and pruned when push receipts report them invalid |
| **Deployment** | Multi-stage Dockerfile to a small static image; Docker Compose with the API, Postgres and Caddy (automatic TLS for `wss://`); GitHub Actions runs vet and tests, builds and pushes the image, and deploys to EC2 over SSH |
| **Operations** | Health and readiness endpoints, structured logs (`log/slog`), connection and message counters, and graceful shutdown that drains sockets on SIGTERM |

## Benchmarks (planned)

Every number will come with the script that produced it, the instance type and the exact settings.

- **Connections:** how many idle connections one small EC2 instance holds, and the memory each costs.
- **Latency:** p50, p95 and p99 time from send to delivery, under a load generator that runs many clients in parallel.
- **Reconnect storm:** a large batch of clients drops and reconnects at once; every message must still arrive once, in order.
- **Chaos:** connections are cut at random while sending (through a fault-injecting proxy); a checker compares each client's final message list with the server's.

## Testing

- Unit tests for frame encoding, sequence assignment and duplicate detection.
- Integration tests run the server against a real Postgres (a service container in CI) with scripted clients.
- A randomized test disconnects clients mid-send and asserts that every conversation ends with no gaps and no duplicates on any client.
- Server tests run under the race detector (`go test -race`).
- Jest tests cover the client's outbox and reconnect logic.

## Roadmap

- [x] Scope and design
- [ ] Go server: auth, WebSocket hub, one-to-one messages
- [ ] Postgres schema, per-conversation sequence numbers, idempotent sends
- [ ] Expo app: sign-in, chat list, chat screen
- [ ] SQLite cache and outbox, reconnect and sync after the last sequence number
- [ ] Group chats, typing indicators, read receipts, presence
- [ ] Push notifications for offline members
- [ ] Docker, GitHub Actions CI/CD, deployment to AWS EC2 with TLS
- [ ] Load, reconnect-storm and chaos tests
- [ ] Stretch: fan-out across several server instances with Redis pub/sub
- [ ] Write-up

## Planned structure

```
server/
  cmd/superchatd/  # entry point
  internal/
    auth/          # passwords, JWTs, refresh tokens
    hub/           # connections, fan-out, presence
    protocol/      # frame types and validation
    store/         # Postgres queries, sequence numbers
    push/          # Expo Push API client
    httpapi/       # REST handlers, health checks
  migrations/
mobile/
  app/             # expo-router screens
  src/net/         # socket client, reconnect, sync
  src/db/          # SQLite cache and outbox
deploy/            # Dockerfile, compose.yaml, Caddyfile
loadtest/          # load generator, chaos runner, checker
.github/workflows/ # CI and deploy
```

## Requirements

Go 1.26 or newer, Node 24, Expo SDK 58 and Docker. Push notifications need an Expo development build: Expo Go on Android no longer supports remote push since SDK 53. Deployment needs an AWS account and one EC2 instance.
