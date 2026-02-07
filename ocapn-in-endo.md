# OCApN in Endo

## Overview

**OCApN** (Object Capability Network) is a distributed networking protocol implemented in Endo that extends local object-capability programming patterns to networked systems. It enables secure, authenticated communication between distributed objects using cryptographic identities, capability-based access control, and promise pipelining.

The implementation lives primarily in two packages:

- **`@endo/ocapn`** (`packages/ocapn`) — Core protocol implementation (~8,000 lines, private/unpublished, version 0.2.2)
- **`@endo/ocapn-noise`** (`packages/ocapn-noise`) — Noise Protocol transport layer (Rust compiled to WASM)

## Architecture

The OCApN stack in Endo is layered as follows:

```
┌─────────────────────────────────────────────┐
│ Application Layer                           │
│   E() eventual send, Far objects, Tagged    │
└──────────────────┬──────────────────────────┘
                   │
┌──────────────────▼──────────────────────────┐
│ Client Layer (src/client/)                  │
│   Session management, SturdyRef creation    │
│   and enlivening, grant tracking,           │
│   netlayer registration                     │
└──────────────────┬──────────────────────────┘
                   │
┌──────────────────▼──────────────────────────┐
│ CapTP Layer (src/captp/)                    │
│   Slot management, reference counting,      │
│   promise pipelining, answer handling       │
└──────────────────┬──────────────────────────┘
                   │
┌──────────────────▼──────────────────────────┐
│ Codec Layer (src/codecs/, src/syrup/)       │
│   Operations, descriptors, passables,       │
│   Syrup binary encoding/decoding            │
└──────────────────┬──────────────────────────┘
                   │
┌──────────────────▼──────────────────────────┐
│ Netlayer Abstraction (src/netlayers/)       │
│   Transport-agnostic async iterators,       │
│   send/receive hooks, connection lifecycle  │
└──────────────────┬──────────────────────────┘
                   │
┌──────────────────▼──────────────────────────┐
│ Transport Layer                             │
│   TCP (test-only), Noise Protocol (WASM)    │
└─────────────────────────────────────────────┘
```

## Key Concepts

### CapTP (Capability Transport Protocol)

The message-passing core lives in `src/captp/`. It manages bidirectional **slot tables** that track exported and imported references between peers. Slots are branded strings following the pattern `{type}{direction}{position}`:

- Types: `o` (object), `p` (promise), `a` (answer)
- Direction: `+` (local), `-` (remote)
- Position: numeric index

Example: `o+1` is local object 1, `p-42` is remote promise 42.

CapTP supports **reference counting** and **garbage collection** via `op:gc-*` messages, using `PairwiseTable` for bidirectional tracking with pending/committed counts and finalizer-based cleanup.

### Syrup Codec

A binary serialization format (`src/syrup/`) for all OCapN messages. Supported types:

| Type | Format | Example |
|------|--------|---------|
| Boolean | `t` or `f` | `t` |
| Integer | bigint | `42+` |
| Float64 | `D` + 8 bytes IEEE 754 | `D...` |
| Bytestring | length `:` bytes | `3:cat` |
| String | length `"` UTF-8 | `3"cat` |
| List | `[` items `]` | `[1+2+]` |
| Dictionary | `{` key-value pairs `}` | `{1"a1+}` |

Encoding is strictly canonical with no JavaScript-specific extensions.

### Sessions

A **session** is an established connection between two OCapN nodes. The handshake works as follows:

1. Client A opens a connection via a netlayer
2. Client A sends `op:start-session` with its public key and location signature
3. Client B responds with its own `op:start-session`
4. Both sides verify the location signature using the peer's public key
5. A `SessionId` is derived from a hash of both public keys
6. An `Ocapn` instance is created for message dispatch

Crossed hellos (simultaneous connections) are detected and the preferred connection is kept.

### SturdyRefs

**Sturdy references** are persistent, serializable references to remote objects. They consist of:

- An **OcapnLocation** — the network endpoint (designator + transport + hints)
- A **SwissNum** — a cryptographic random identifier

SturdyRefs can be "enlivened" by:
1. Checking if the location is local (direct fetch)
2. If remote, establishing a session to the target location
3. Calling `bootstrap.fetch(swissNum)` on the remote node
4. Receiving a live reference to the remote object

### Third-Party Handoffs

A mechanism for securely transferring capabilities between three parties without requiring direct relationships:

1. **Gifter** creates a signed `HandoffGive` (containing receiver's key, exporter location, session ID, gift ID)
2. **Gifter** sends it to the **receiver** out-of-band
3. **Receiver** creates a signed `HandoffReceive`
4. **Receiver** sends it to the **exporter** (the third party)
5. **Exporter** validates both signatures and grants access

The `GrantTracker` (`src/client/grant-tracker.js`) tracks imported capabilities with metadata about their origin.

### Promise Pipelining

Messages can target the **result** of a pending remote call before it resolves. Questions are assigned answer positions, and subsequent calls reference those positions. This eliminates round-trip latencies — multiple chained calls can be dispatched without waiting for intermediate results.

### Network Locations

An **OcapnLocation** represents a network endpoint:

```
type: 'ocapn-peer'
designator: string
transport: string
hints: object
```

Serialized as URIs: `ocapn://host1.tcp?hints={"port":"5000"}`

## Dependencies on Other Endo Packages

| Package | Role in OCApN |
|---------|---------------|
| `@endo/eventual-send` | `E()` proxy and `HandledPromise` for async message dispatch |
| `@endo/marshal` | `Far()` and `Remotable` for creating capability objects |
| `@endo/pass-style` | Type tagging and pass-by-copy/reference semantics |
| `@endo/promise-kit` | Promise/resolver pairs |
| `@endo/netstring` | Length-prefixed framing for messages |
| `@endo/nat` | BigInt utilities |
| `@endo/errors` | Error handling conventions |
| `@endo/immutable-arraybuffer` | Safe ArrayBuffer handling |

## Relationship to `@endo/captp`

The older `packages/captp` is a simpler predecessor that uses JSON encoding. `@endo/ocapn` is the more comprehensive successor with:

- **Syrup binary encoding** instead of JSON
- **Cryptographic authentication** via Ed25519 keys
- **Sturdy references** for persistent capability storage
- **Third-party handoffs** for delegated capability transfer
- **Noise Protocol transport** for encrypted channels

Both share the core concepts of eventual send and promise pipelining from `@endo/eventual-send`.

## Noise Protocol Transport

The `@endo/ocapn-noise` package and `rust/ocapn_noise` crate provide authenticated encryption:

- **XX handshake pattern** — 3-message mutual authentication
- **X25519** — ephemeral key agreement for forward secrecy
- **ChaCha20Poly1305** — AEAD message encryption
- **Blake2s** — key derivation hash
- **Ed25519** — long-term identity key ownership proof

Compiled from Rust to WASM with zero heap allocations on the Rust side.

### Handshake Flow

1. **SYN**: Initiator sends Ed25519 public key + signed ephemeral X25519 key
2. **SYNACK**: Responder sends their keys + signature
3. **ACK**: Initiator finalizes the handshake

## Source Structure

```
packages/ocapn/
├── src/
│   ├── client/              — Session management, SturdyRef, grant tracking
│   │   ├── index.js         — makeClient() entry point
│   │   ├── ocapn.js         — Core message dispatch
│   │   ├── ref-kit.js       — Reference management kit
│   │   ├── grant-tracker.js — Imported capability tracking
│   │   ├── sturdyrefs.js    — SturdyRef creation/enlivening
│   │   └── util.js          — Location IDs, swissnum encoding
│   ├── captp/               — Slot tables, reference counting
│   │   ├── pairwise.js      — Bidirectional slot management
│   │   ├── ocapn-tables.js  — OCapN-specific extensions
│   │   ├── finalize.js      — GC finalizer maps
│   │   └── refcount.js      — Reference counting logic
│   ├── codecs/              — Message encoding/decoding
│   │   ├── operations.js    — CapTP operations
│   │   ├── descriptors.js   — Reference descriptors
│   │   ├── passable.js      — Value encoding
│   │   └── components.js    — Locations, keys, signatures
│   ├── syrup/               — Binary serialization
│   │   ├── encode.js        — Syrup encoder
│   │   ├── decode.js        — Syrup decoder
│   │   └── codec.js         — Codec framework
│   ├── netlayers/
│   │   └── tcp-test-only.js — Test TCP transport
│   ├── cryptography.js      — Ed25519 key management
│   └── selector.js          — Selector handling
├── test/                    — Comprehensive test suite
└── package.json

packages/ocapn-noise/        — WASM Noise Protocol bindings
rust/ocapn_noise/            — Rust Noise Protocol source
```

## Current Status

- **Private packages** — not published to npm, marked `"private": true`
- **Tentative implementation** — tracking the evolving OCapN specification
- **Good test coverage** — client workflows, sturdy refs, handoffs, GC, crypto, codecs, Syrup fuzzing
- **Known limitations**:
  - Syrup: no Maps/Sets, dictionary keys must be strings, no single-precision floats
  - Only a test TCP netlayer (no production transport yet)
  - Limited hints support (string values only)
  - Import collection via WeakRefs (configurable)
