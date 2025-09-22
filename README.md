# ioBroker Developer Documentation

This documentation provides comprehensive reference material for developers creating ioBroker adapters. It covers all public APIs, types, and configuration options available in the ioBroker.js-controller.

## Table of Contents

1. [Getting Started](docs/getting-started.md) - Introduction to ioBroker adapter development
2. [Adapter Class API](docs/adapter-api.md) - Complete reference for the Adapter class methods
3. [TypeScript Types & Interfaces](docs/types.md) - All public TypeScript definitions
4. [Constants](docs/constants.md) - Available constants and enumerations
5. [Utility Functions](docs/utils.md) - Helper functions and utilities
6. [io-package.json Schema](docs/io-package-schema.md) - Complete configuration schema reference
7. [Events & Callbacks](docs/events.md) - Event handling and callback patterns
8. [State Management](docs/states.md) - Working with states, objects, and data
9. [File Operations](docs/files.md) - File system operations and management
10. [Network & Communication](docs/network.md) - HTTP requests, messaging, and communication
11. [Error Handling](docs/errors.md) - Error types and handling patterns
12. [Best Practices](docs/best-practices.md) - Recommended patterns and approaches

## Quick Reference

### Core Adapter Methods
- [`setState()`](docs/adapter-api.md#setstate) - Set state values
- [`getState()`](docs/adapter-api.md#getstate) - Retrieve state values
- [`setObject()`](docs/adapter-api.md#setobject) - Create/update objects
- [`getObject()`](docs/adapter-api.md#getobject) - Retrieve objects
- [`sendTo()`](docs/adapter-api.md#sendto) - Send messages between adapters

### Essential Types
- [`AdapterConfig`](docs/types.md#adapterconfig) - Adapter configuration interface
- [`State`](docs/types.md#state) - State object definition
- [`ioBrokerObject`](docs/types.md#iobrokerobject) - Base object interface

### Configuration Schema
- [Common Properties](docs/io-package-schema.md#common) - Standard adapter properties
- [Native Configuration](docs/io-package-schema.md#native) - Adapter-specific settings
- [Instance Objects](docs/io-package-schema.md#instanceobjects) - Per-instance object definitions

## Source References

This documentation is generated from the ioBroker.js-controller source code:
- Main repository: https://github.com/ioBroker/ioBroker.js-controller
- Primary source: `packages/adapter/src/lib/adapter/adapter.ts`
- Type definitions: `packages/adapter/src/lib/_Types.ts`, `packages/types-dev/index.d.ts`
- Schema: `schemas/io-package.json`

<!-- 
Source metadata for automated updates:
- adapter.ts: SHA 92c8a2b2f4d3e5f6g7h8i9j0k1l2m3n4o5p6q7r8s9t0u1v2w3x4y5z
- types files: Multiple sources from js-controller master branch
- io-package.json schema: SHA 074d72331723c7722b3fa53cd732410a87141866
- Last updated: 2024-12-22
-->

## Documentation Structure

Each documentation file is structured for quick navigation and includes:
- Complete method signatures with parameters
- Return type information
- Usage examples
- Cross-references to related functionality
- Source file references for automated updates

## Contributing

This documentation is maintained automatically based on the source code. For corrections or improvements, please refer to the source metadata included in each file.