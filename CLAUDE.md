# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Overview

libsql-js is a Node.js/Bun/Deno native binding for libSQL that provides a better-sqlite3-compatible API. The project consists of:

- **Rust native module** (`src/lib.rs`, `src/auth.rs`) - N-API bindings using napi-rs
- **JavaScript wrapper layers** - Two APIs:
  - `compat.js` - Synchronous, better-sqlite3-compatible API
  - `promise.js` - Async/Promise-based API
- **TypeScript declarations** - Generated from JS files via `tsc`

## Architecture

### Two-Layer Design

The codebase uses a dual-API architecture:

1. **Native Rust Layer** (`src/lib.rs`):
   - Wraps the async `libsql` crate using Tokio runtime
   - Exports both sync and async N-API functions via napi-rs
   - Implements `Database` and `Statement` structs
   - Handles error conversion from libSQL errors to JavaScript errors (JSON-encoded)
   - Manages authentication via `Authorizer` in `src/auth.rs`

2. **JavaScript Wrapper Layers**:
   - `compat.js` - Calls `*Sync` functions from native module, blocks event loop
   - `promise.js` - Calls async functions, returns Promises via `Env::execute_tokio_future`
   - Both wrap errors using `SqliteError` class (`sqlite-error.js`)
   - Both use `Authorization` module (`auth.js`) for table-level access control

### Error Handling

Errors from Rust are JSON-encoded with structure:
```json
{
  "message": "error message",
  "libsqlError": true,
  "code": "SQLITE_ERROR",
  "rawCode": 1
}
```

JavaScript wrappers parse this and create `SqliteError` instances (except `SQLITE_AUTH` which preserves JSON, and `SQLITE_NOTOPEN` which becomes `TypeError`).

### Database Modes

The library supports three connection modes:
- **Local** - File path or `:memory:` databases
- **Remote** - URLs starting with `http://`, `https://`, `libsql://`, or `wss://`
- **Embedded Replica** - Local database with `syncUrl` option for syncing with remote

## Build Commands

### Development Build

```bash
LIBSQL_JS_DEV=1 npm run build
```

This compiles:
1. Rust native module via `napi build` (targeting current platform)
2. TypeScript declarations via `tsc` (from `compat.js` and `promise.js`)

The `LIBSQL_JS_DEV=1` environment variable is required for local development.

### Debug Build

```bash
npm run build:debug
```

Builds without optimizations for debugging.

### Production Build

```bash
npm run build
```

Builds optimized release artifacts for the current platform.

## Testing

Integration tests use the `ava` test runner and test both APIs:

```bash
# Run all integration tests (from root)
npm test

# From integration-tests directory
cd integration-tests
npm i
npm test
```

Individual test suites:
- `npm run test:sqlite` - Tests sync API with better-sqlite3 comparison
- `npm run test:libsql` - Tests sync API with libsql
- `npm run test:async` - Tests promise API
- `npm run test:extensions` - Tests extension loading
- `npm run test:concurrency` - Tests concurrent access

Tests require linking the local package:
```bash
export LIBSQL_JS_DEV=1
npm link
cd integration-tests
npm link libsql
```

## Platform-Specific Notes

### macOS Linking

The `.cargo/config.toml` sets special linker flags for macOS to allow undefined symbols during linking:
```toml
rustflags = ["-C", "link-arg=-undefined", "-C", "link-arg=dynamic_lookup"]
```

This is required for N-API modules on macOS.

### Cross-Compilation

The CI builds for multiple targets using Docker containers and cross-compilers. See `.github/workflows/CI.yml` for the build matrix, which includes:
- macOS (x64, ARM64)
- Linux GNU (x64, ARM64)
- Linux musl (x64, ARM64)
- Windows (x64)

## Key Files

- `src/lib.rs` - Main Rust implementation with Database/Statement structs
- `src/auth.rs` - Table-level authorization system
- `compat.js` - Synchronous API wrapper
- `promise.js` - Promise-based API wrapper
- `sqlite-error.js` - Error class
- `auth.js` - Authorization builder for JavaScript
- `Cargo.toml` - Rust dependencies (libsql, napi, tokio)
- `package.json` - Defines dual exports: `.` (compat) and `./promise`

## TypeScript Support

TypeScript declarations are generated from JSDoc in the JavaScript files:
- `tsc` runs with `emitDeclarationOnly: true`
- Outputs `compat.d.ts` and `promise.d.ts`
- `index.d.ts` contains native module type definitions

## Encryption Support

The library supports encryption at rest via libsql's encryption feature:
- `encryptionCipher` - Cipher to use (passed to libSQL)
- `encryptionKey` - Key for local database encryption
- `remoteEncryptionKey` - Key for remote database encryption
