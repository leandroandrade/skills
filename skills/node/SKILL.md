---
name: node-best-practices
description: Best practices for Node.js development using CommonJS
metadata:
  tags: node, nodejs, javascript, commonjs, backend, server
---

## When to use

Use this skill whenever you are dealing with Node.js code to obtain domain-specific knowledge for building robust, performant, and maintainable Node.js applications.

## CommonJS

All code must use CommonJS (`require`/`module.exports`). Do not use ES Modules (`import`/`export`) syntax.

See [rules/modules.md](rules/modules.md) for module patterns and examples.

## How to use

Read individual rule files for detailed explanations and code examples:

- [rules/error-handling.md](rules/error-handling.md) - Error handling patterns in Node.js
- [rules/async-patterns.md](rules/async-patterns.md) - Async/await and Promise patterns
- [rules/streams.md](rules/streams.md) - Working with Node.js streams
- [rules/modules.md](rules/modules.md) - CommonJS module patterns
- [rules/testing.md](rules/testing.md) - Testing strategies for Node.js applications
- [rules/flaky-tests.md](rules/flaky-tests.md) - Identifying and diagnosing flaky tests with node:test
- [rules/node-modules-exploration.md](rules/node-modules-exploration.md) - Navigating and analyzing node_modules directories
- [rules/performance.md](rules/performance.md) - Performance optimization techniques
- [rules/caching.md](rules/caching.md) - Caching patterns and libraries
- [rules/profiling.md](rules/profiling.md) - Profiling and benchmarking tools
- [rules/logging.md](rules/logging.md) - Logging and debugging patterns
- [rules/environment.md](rules/environment.md) - Environment configuration and secrets management
- [rules/graceful-shutdown.md](rules/graceful-shutdown.md) - Graceful shutdown and signal handling
