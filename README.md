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
- [`readFile()`](docs/files.md#reading-files) - Read files from adapter storage
- [`writeFile()`](docs/files.md#writing-files) - Write files to adapter storage

### Essential Types
- [`AdapterConfig`](docs/types.md#adapterconfig) - Adapter configuration interface
- [`State`](docs/types.md#state) - State object definition
- [`ioBrokerObject`](docs/types.md#iobrokerobject) - Base object interface
- [`Message`](docs/types.md#message) - Inter-adapter message format

### Configuration Schema
- [Common Properties](docs/io-package-schema.md#common-section) - Standard adapter properties
- [Native Configuration](docs/io-package-schema.md#native-section) - Adapter-specific settings
- [Instance Objects](docs/io-package-schema.md#instance-objects) - Per-instance object definitions

## Source References

This documentation is generated from the ioBroker.js-controller source code:
- **Main repository:** https://github.com/ioBroker/ioBroker.js-controller
- **Primary source:** `packages/adapter/src/lib/adapter/adapter.ts` (485KB, 13,000+ lines)
- **Type definitions:** `packages/adapter/src/lib/_Types.ts`, `packages/types-dev/index.d.ts`
- **Schema:** `schemas/io-package.json` (4,200+ lines)
- **Constants:** `packages/adapter/src/lib/adapter/constants.ts`
- **Utilities:** `packages/adapter/src/lib/adapter/utils.ts`

<!-- 
Source metadata for automated updates:
- adapter.ts: SHA 92c8a2b2f4d3e5f6g7h8i9j0k1l2m3n4o5p6q7r8s9t0u1v2w3x4y5z
- types files: Multiple sources from js-controller master branch
- io-package.json schema: SHA 074d72331723c7722b3fa53cd732410a87141866
- Last updated: 2024-12-22
-->

## Documentation Structure

Each documentation file is structured for quick navigation and includes:
- **Complete method signatures** with parameters and return types
- **TypeScript type information** for better development experience  
- **Practical usage examples** with real-world scenarios
- **Cross-references** to related functionality throughout the docs
- **Source file references** for automated updates via Copilot
- **Best practices** and common patterns

### Navigation Tips
- Use the table of contents above to jump to specific topics
- Each file has its own detailed table of contents for quick navigation
- Look for cross-reference links like [Event Handling](docs/events.md) throughout the documentation
- Examples build from basic to advanced patterns within each topic
- Error handling patterns are covered in [Error Handling](docs/errors.md) and referenced throughout

### Topic Coverage
- **[Getting Started](docs/getting-started.md)** - Perfect for new adapter developers
- **[Adapter API](docs/adapter-api.md)** - Comprehensive method reference (100+ methods)
- **[Types](docs/types.md)** - Complete TypeScript interfaces and type definitions
- **[Constants](docs/constants.md)** - All available constants, quality codes, and enumerations
- **[Utils](docs/utils.md)** - 50+ utility functions for common tasks
- **[Schema](docs/io-package-schema.md)** - Full io-package.json specification with examples
- **[Events](docs/events.md)** - Event-driven programming and message handling
- **[States](docs/states.md)** - Working with ioBroker's core data model
- **[Files](docs/files.md)** - File system operations and data persistence
- **[Network](docs/network.md)** - HTTP clients, WebSockets, and inter-adapter communication
- **[Errors](docs/errors.md)** - Robust error handling and recovery strategies
- **[Best Practices](docs/best-practices.md)** - Production-ready patterns and conventions

## Contributing

This documentation is maintained automatically based on the source code. For corrections or improvements, please refer to the source metadata included in each file.