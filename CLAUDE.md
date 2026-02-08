# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

Go Ethereum (geth) is a Golang execution layer implementation of the Ethereum protocol. This is a production Ethereum client that runs as a full node, archive node, or light node on the Ethereum network.

## Build and Development Commands

### Building
```bash
make geth      # Build main geth executable
make all       # Build all utilities (abigen, evm, rlpdump, clef)
make evm       # Build standalone EVM
```

### Testing
```bash
make test      # Run all tests (builds first)
go test ./...  # Run tests directly with go
go test -v ./core/blockchain.go ./core/blockchain_test.go  # Run specific test file
go test -run TestBlockchain  # Run specific test
```

### Code Quality
```bash
make fmt       # Format code with gofmt
make lint      # Run linters via build/ci.go
```

### Development Tools
```bash
make devtools  # Install stringer, gencodec, protoc-gen-go, abigen
```

### Build System Internals
- Makefile wraps `go run build/ci.go` for non-Go developers
- Direct Go builds: `go build ./cmd/geth`
- Go version: 1.24+ (see go.mod)

## High-Level Architecture

### Layer Structure
```
cmd/geth/          → CLI entry point (urfave/cli/v2)
    ↓
node/              → Service container and lifecycle management
    ↓
eth/               → Ethereum protocol implementation
    ↓
core/              → Blockchain consensus logic
    ↓
ethdb/ + rawdb/    → Database abstraction (LevelDB, Pebble, freezer tables)
```

### Key Components

**cmd/geth/** - Main CLI client using `urfave/cli/v2`. Flags defined in `cmd/utils/`, main app in `cmd/geth/main.go`.

**core/** - Core blockchain protocol:
- `blockchain.go` - `BlockChain` struct, main block processing orchestration
- `state/` - State DB (Merkle Patricia Trie), state transitions
- `vm/` - Ethereum Virtual Machine implementation
- `types/` - Block, transaction, receipt types
- `rawdb/` - Low-level database access with ancient data freezer
- `consensus/` - Consensus interface and implementations

**eth/** - Ethereum protocol layer:
- `backend.go` - `Ethereum` struct, protocol handler lifecycle
- `protocols/eth/` - ETH/66, ETH/67, ETH/68 protocol handlers
- `protocols/snap/` - Snapshot sync protocol
- `downloader/` - Block synchronization logic
- `filters/` - Log and event filtering
- `tracers/` - Transaction tracing (JS, native, live)
- `ethconfig/` - Configuration options

**p2p/** - Peer-to-peer networking:
- `server.go` - P2P server managing peer connections
- `discover/` - Node discovery (v4/v5 discv5)
- `enode/` - Node identification and URLs
- `protocols/` - Protocol multiplexing over devp2p

**params/** - Network configurations:
- `config.go` - Chain configs (MainnetChainConfig, etc.)
- Genesis definitions for mainnet, testnets

**rpc/** - JSON-RPC API server over HTTP, WebSocket, IPC

**accounts/** - Key management:
- `keystore/` - Encrypted key storage
- `abi/` - Contract ABI encoding/decoding

**crypto/** - Cryptographic primitives (secp256k1, keccak256, ecies)

## Important Architectural Patterns

### Service Architecture
- `node/` implements a service container pattern
- Services (eth, les, shh) implement `node.Service` interface
- Lifecycle: `Start()`, `Stop()`, protocols attach during startup

### Consensus Interface
- `consensus.ChainReader` - Read-only blockchain access
- `consensus.Engine` - Block validation and sealing
- Post-merge: uses Beacon Chain for consensus (consensus/beacon/*)

### Event System
- `event.TypeMux` for event feeds
- Core components publish events (ChainEvent, ChainSideEvent, etc.)
- Subscribers: `eth/backend.go` feeds into downloader, txpool, etc.

### Database Layer
- `ethdb.Database` interface wrapping key-value stores
- `rawdb` package provides low-level accessors
- Ancient data (blocks, receipts) in "freezer" append-only files
- State in LevelDB/Pebble with Merkle Patricia Trie

### Protocol Handlers
- `p2p.Protocol` struct defines each protocol
- Registered in node Service `Run()` or `Start()`
- `eth/protocols/eth/` implements ETH protocol versions
- Version negotiation via `caps` in `p2p.Server`

## Commit Convention

From CONTRIBUTING.md, commit messages should be prefixed with modified package(s):
- Format: `package1, package2: description`
- Example: `eth, rpc: make trace configs optional`
- PRs target `master` branch

## Licensing

- `cmd/` directory: GPL v3.0 (binaries)
- All other code: LGPL v3.0 (library code)
