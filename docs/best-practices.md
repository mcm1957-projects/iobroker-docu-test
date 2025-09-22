# Best Practices

This document outlines recommended patterns, conventions, and best practices for developing robust ioBroker adapters.

## Table of Contents

- [Code Organization](#code-organization)
- [Error Handling](#error-handling)
- [Performance](#performance)
- [Security](#security)
- [Configuration](#configuration)
- [State Management](#state-management)
- [Logging](#logging)
- [Testing](#testing)
- [Documentation](#documentation)

## Code Organization

### Project Structure

Organize your adapter project with a clear, maintainable structure:

```
my-adapter/
├── admin/                  # Admin interface files
│   ├── index_m.html       # Configuration page
│   ├── words.js           # Translations
│   └── my-adapter.png     # Icon
├── docs/                  # Documentation
│   ├── en/               # English docs
│   └── de/               # German docs
├── lib/                   # Library files
│   ├── adapter.js        # Main adapter logic
│   ├── device.js         # Device handling
│   └── utils.js          # Utility functions
├── test/                  # Test files
├── io-package.json       # Adapter metadata
├── main.js               # Entry point
├── package.json          # npm package definition
└── README.md             # Project documentation
```

### Modular Code Design

```javascript
// main.js - Entry point
const AdapterCore = require('./lib/adapter');

const adapter = new AdapterCore({
    name: 'my-adapter'
});

// lib/adapter.js - Main adapter logic
class MyAdapter {
    constructor(options) {
        this.adapter = new utils.Adapter(options);
        this.deviceManager = new DeviceManager(this.adapter);
        this.apiClient = new ApiClient(this.adapter.config);
        this.setupEventHandlers();
    }

    setupEventHandlers() {
        this.adapter.on('ready', this.onReady.bind(this));
        this.adapter.on('stateChange', this.onStateChange.bind(this));
        this.adapter.on('unload', this.onUnload.bind(this));
    }

    async onReady() {
        await this.initialize();
    }

    async initialize() {
        // Validation
        if (!this.validateConfig()) {
            this.adapter.log.error('Invalid configuration');
            return;
        }

        // Setup
        await this.createObjects();
        await this.connectToDevice();
        await this.startPolling();
    }
}

// lib/device.js - Device management
class DeviceManager {
    constructor(adapter) {
        this.adapter = adapter;
        this.devices = new Map();
    }

    async addDevice(deviceConfig) {
        const device = new Device(deviceConfig);
        await device.initialize();
        this.devices.set(device.id, device);
        await this.createDeviceObjects(device);
    }
}
```

### Class-Based Architecture

```javascript
class MyAdapter {
    constructor(options) {
        this.adapter = new utils.Adapter(options);
        this.config = this.adapter.config;
        this.log = this.adapter.log;
        this.isConnected = false;
        this.pollingInterval = null;
        
        this.setupEventHandlers();
    }

    // Separate concerns into methods
    setupEventHandlers() { /* ... */ }
    validateConfig() { /* ... */ }
    async initialize() { /* ... */ }
    async createObjects() { /* ... */ }
    async connectToDevice() { /* ... */ }
    async startPolling() { /* ... */ }
    async cleanup() { /* ... */ }
}
```

## Error Handling

### Comprehensive Error Handling

```javascript
class RobustAdapter {
    constructor(options) {
        this.adapter = new utils.Adapter(options);
        this.setupErrorHandling();
    }

    setupErrorHandling() {
        // Handle adapter errors
        this.adapter.on('error', this.handleAdapterError.bind(this));
        
        // Handle uncaught exceptions
        process.on('uncaughtException', this.handleUncaughtException.bind(this));
        
        // Handle unhandled promise rejections
        process.on('unhandledRejection', this.handleUnhandledRejection.bind(this));
    }

    handleAdapterError(error) {
        this.adapter.log.error(`Adapter error: ${error.message}`);
        
        // Attempt recovery based on error type
        if (error.code === 'ECONNREFUSED') {
            this.scheduleReconnect();
        } else if (error.code === 'CRITICAL') {
            this.adapter.terminate('Critical error occurred');
        }
    }

    handleUncaughtException(error) {
        this.adapter.log.error(`Uncaught exception: ${error.message}\n${error.stack}`);
        this.adapter.terminate('Uncaught exception', 1);
    }

    handleUnhandledRejection(reason, promise) {
        this.adapter.log.error(`Unhandled rejection: ${reason}`);
        // Don't terminate on promise rejections, but log them
    }

    // Wrapper for async operations
    async safeAsync(operation, context = '') {
        try {
            return await operation();
        } catch (error) {
            this.adapter.log.error(`${context} failed: ${error.message}`);
            throw error; // Re-throw if caller needs to handle
        }
    }

    // Wrapper for callback operations
    safeCallback(callback, context = '') {
        return (error, result) => {
            if (error) {
                this.adapter.log.error(`${context} failed: ${error.message}`);
            }
            if (callback) callback(error, result);
        };
    }
}
```

### Graceful Degradation

```javascript
class ResilientAdapter {
    async connectToDevice() {
        const maxRetries = 3;
        let retries = 0;

        while (retries < maxRetries) {
            try {
                await this.attemptConnection();
                this.adapter.log.info('Connected to device');
                return;
            } catch (error) {
                retries++;
                this.adapter.log.warn(`Connection attempt ${retries} failed: ${error.message}`);
                
                if (retries < maxRetries) {
                    const delay = Math.pow(2, retries) * 1000; // Exponential backoff
                    await this.sleep(delay);
                }
            }
        }

        // Fallback to offline mode
        this.adapter.log.error('Could not connect to device, running in offline mode');
        this.enterOfflineMode();
    }

    enterOfflineMode() {
        this.isConnected = false;
        this.adapter.setState('info.connection', false, true);
        
        // Continue with limited functionality
        this.startOfflinePolling();
    }
}
```

## Performance

### Efficient State Updates

```javascript
class PerformantAdapter {
    constructor(options) {
        this.adapter = new utils.Adapter(options);
        this.stateCache = new Map();
        this.updateQueue = [];
        this.batchUpdateTimer = null;
    }

    // Batch state updates to reduce I/O
    queueStateUpdate(id, value, ack = true) {
        this.updateQueue.push({ id, value, ack });
        
        if (!this.batchUpdateTimer) {
            this.batchUpdateTimer = setTimeout(() => {
                this.processBatchUpdates();
            }, 100); // 100ms batch window
        }
    }

    async processBatchUpdates() {
        if (this.updateQueue.length === 0) return;

        const updates = this.updateQueue.splice(0); // Clear queue
        this.batchUpdateTimer = null;

        // Group updates by state ID (keep only latest)
        const latestUpdates = new Map();
        updates.forEach(update => {
            latestUpdates.set(update.id, update);
        });

        // Apply updates
        const promises = Array.from(latestUpdates.values()).map(update => 
            this.setStateIfChanged(update.id, update.value, update.ack)
        );

        await Promise.allSettled(promises);
    }

    // Only update if value actually changed
    async setStateIfChanged(id, value, ack = true) {
        const cachedValue = this.stateCache.get(id);
        
        if (cachedValue !== value) {
            this.stateCache.set(id, value);
            await this.adapter.setStateAsync(id, value, ack);
        }
    }

    // Pre-load state cache
    async loadStateCache() {
        const states = await this.adapter.getStatesAsync('*');
        Object.keys(states).forEach(id => {
            if (states[id]) {
                this.stateCache.set(id, states[id].val);
            }
        });
    }
}
```

### Memory Management

```javascript
class MemoryEfficientAdapter {
    constructor(options) {
        this.adapter = new utils.Adapter(options);
        this.dataBuffer = [];
        this.maxBufferSize = 1000;
        
        // Periodic cleanup
        setInterval(() => this.cleanupMemory(), 60000); // Every minute
    }

    addData(data) {
        this.dataBuffer.push({
            ...data,
            timestamp: Date.now()
        });

        // Prevent memory leaks
        if (this.dataBuffer.length > this.maxBufferSize) {
            this.dataBuffer = this.dataBuffer.slice(-this.maxBufferSize / 2);
            this.adapter.log.debug('Data buffer trimmed');
        }
    }

    cleanupMemory() {
        // Remove old data
        const cutoff = Date.now() - (24 * 60 * 60 * 1000); // 24 hours
        const originalLength = this.dataBuffer.length;
        
        this.dataBuffer = this.dataBuffer.filter(item => 
            item.timestamp > cutoff
        );

        if (this.dataBuffer.length < originalLength) {
            this.adapter.log.debug(`Cleaned ${originalLength - this.dataBuffer.length} old entries`);
        }

        // Force garbage collection if available
        if (global.gc) {
            global.gc();
        }
    }
}
```

## Security

### Input Validation

```javascript
class SecureAdapter {
    validateConfig(config) {
        const schema = {
            hostname: { type: 'string', required: true, pattern: /^[a-zA-Z0-9.-]+$/ },
            port: { type: 'number', required: true, min: 1, max: 65535 },
            username: { type: 'string', required: false, maxLength: 100 },
            password: { type: 'string', required: false, maxLength: 200 },
            apiKey: { type: 'string', required: false, pattern: /^[a-zA-Z0-9]+$/ }
        };

        return this.validateObject(config, schema);
    }

    validateObject(obj, schema) {
        for (const [key, rules] of Object.entries(schema)) {
            const value = obj[key];

            // Required check
            if (rules.required && (value === undefined || value === null)) {
                throw new Error(`Required field '${key}' is missing`);
            }

            if (value !== undefined && value !== null) {
                // Type check
                if (rules.type && typeof value !== rules.type) {
                    throw new Error(`Field '${key}' must be of type ${rules.type}`);
                }

                // Pattern check
                if (rules.pattern && !rules.pattern.test(value)) {
                    throw new Error(`Field '${key}' has invalid format`);
                }

                // Range checks
                if (rules.min !== undefined && value < rules.min) {
                    throw new Error(`Field '${key}' must be >= ${rules.min}`);
                }

                if (rules.max !== undefined && value > rules.max) {
                    throw new Error(`Field '${key}' must be <= ${rules.max}`);
                }

                if (rules.maxLength && value.length > rules.maxLength) {
                    throw new Error(`Field '${key}' is too long`);
                }
            }
        }

        return true;
    }

    sanitizeInput(input) {
        if (typeof input === 'string') {
            // Remove potentially dangerous characters
            return input.replace(/[<>'"&]/g, '');
        }
        return input;
    }
}
```

### Secure Communication

```javascript
class SecureApiClient {
    constructor(config) {
        this.config = config;
        this.validateTlsConfig();
    }

    validateTlsConfig() {
        if (this.config.secure) {
            // Require valid certificates in production
            this.tlsOptions = {
                rejectUnauthorized: process.env.NODE_ENV === 'production',
                minVersion: 'TLSv1.2'
            };
        }
    }

    async makeRequest(endpoint, data) {
        const options = {
            method: 'POST',
            headers: {
                'Content-Type': 'application/json',
                'Authorization': `Bearer ${this.config.apiKey}`,
                'User-Agent': 'ioBroker-MyAdapter/1.0'
            },
            body: JSON.stringify(data),
            timeout: 10000, // Prevent hanging requests
            ...this.tlsOptions
        };

        try {
            const response = await fetch(endpoint, options);
            
            if (!response.ok) {
                throw new Error(`HTTP ${response.status}: ${response.statusText}`);
            }

            return await response.json();
        } catch (error) {
            // Don't log sensitive data
            this.adapter.log.error(`API request failed: ${error.message}`);
            throw error;
        }
    }
}
```

## Configuration

### Configuration Validation

```javascript
class ConfigurableAdapter {
    constructor(options) {
        this.adapter = new utils.Adapter(options);
        this.config = this.validateAndNormalizeConfig(this.adapter.config);
    }

    validateAndNormalizeConfig(config) {
        const defaults = {
            hostname: 'localhost',
            port: 80,
            secure: false,
            timeout: 5000,
            retries: 3,
            pollingInterval: 30000
        };

        // Merge with defaults
        const normalizedConfig = { ...defaults, ...config };

        // Validate
        this.validateConfig(normalizedConfig);

        // Normalize values
        normalizedConfig.pollingInterval = Math.max(1000, normalizedConfig.pollingInterval);
        normalizedConfig.timeout = Math.max(1000, Math.min(60000, normalizedConfig.timeout));

        return normalizedConfig;
    }

    getConfigSchema() {
        return {
            hostname: {
                type: 'string',
                title: 'Hostname',
                description: 'Device hostname or IP address',
                required: true
            },
            port: {
                type: 'number',
                title: 'Port',
                description: 'Device port number',
                min: 1,
                max: 65535,
                default: 80
            },
            secure: {
                type: 'boolean',
                title: 'Use HTTPS',
                description: 'Enable secure connection',
                default: false
            },
            pollingInterval: {
                type: 'number',
                title: 'Polling Interval (ms)',
                description: 'How often to poll for data',
                min: 1000,
                default: 30000
            }
        };
    }
}
```

### Dynamic Configuration Updates

```javascript
class DynamicAdapter {
    constructor(options) {
        this.adapter = new utils.Adapter(options);
        this.watchConfiguration();
    }

    watchConfiguration() {
        this.adapter.on('objectChange', (id, obj) => {
            if (id === `system.adapter.${this.adapter.name}.${this.adapter.instance}`) {
                if (obj && obj.native) {
                    this.handleConfigurationChange(obj.native);
                }
            }
        });
    }

    async handleConfigurationChange(newConfig) {
        this.adapter.log.info('Configuration changed, updating...');

        try {
            // Validate new configuration
            this.validateConfig(newConfig);

            // Apply changes that don't require restart
            if (newConfig.pollingInterval !== this.config.pollingInterval) {
                this.updatePollingInterval(newConfig.pollingInterval);
            }

            if (newConfig.logLevel !== this.config.logLevel) {
                this.adapter.log.level = newConfig.logLevel;
            }

            // Update config
            this.config = newConfig;

            this.adapter.log.info('Configuration updated successfully');
        } catch (error) {
            this.adapter.log.error(`Configuration update failed: ${error.message}`);
        }
    }
}
```

## State Management

### Organized State Structure

```javascript
class OrganizedAdapter {
    async createObjects() {
        const structure = [
            {
                id: 'info',
                type: 'channel',
                name: 'Information'
            },
            {
                id: 'info.connection',
                type: 'state',
                name: 'Connected',
                role: 'indicator.connected',
                dataType: 'boolean',
                read: true,
                write: false
            },
            {
                id: 'devices',
                type: 'channel',
                name: 'Devices'
            },
            {
                id: 'commands',
                type: 'channel',
                name: 'Commands'
            },
            {
                id: 'commands.refresh',
                type: 'state',
                name: 'Refresh Data',
                role: 'button',
                dataType: 'boolean',
                read: false,
                write: true
            }
        ];

        for (const item of structure) {
            await this.createObject(item);
        }
    }

    async createObject(definition) {
        const obj = {
            type: definition.type,
            common: {
                name: definition.name
            },
            native: {}
        };

        if (definition.type === 'state') {
            obj.common.role = definition.role;
            obj.common.type = definition.dataType;
            obj.common.read = definition.read;
            obj.common.write = definition.write;
            obj.common.def = definition.default;
        }

        await this.adapter.setObjectNotExistsAsync(definition.id, obj);
    }
}
```

### State Synchronization

```javascript
class SynchronizedAdapter {
    constructor(options) {
        this.adapter = new utils.Adapter(options);
        this.deviceStates = new Map();
        this.pendingUpdates = new Set();
    }

    async updateDeviceState(deviceId, states) {
        const currentStates = this.deviceStates.get(deviceId) || {};
        const changes = {};

        // Detect changes
        for (const [key, value] of Object.entries(states)) {
            if (currentStates[key] !== value) {
                changes[key] = value;
            }
        }

        if (Object.keys(changes).length === 0) {
            return; // No changes
        }

        // Update cache
        this.deviceStates.set(deviceId, { ...currentStates, ...changes });

        // Update ioBroker states
        for (const [key, value] of Object.entries(changes)) {
            const stateId = `devices.${deviceId}.${key}`;
            await this.adapter.setStateAsync(stateId, value, true);
        }

        this.adapter.log.debug(`Updated ${Object.keys(changes).length} states for device ${deviceId}`);
    }
}
```

## Logging

### Structured Logging

```javascript
class WellLoggedAdapter {
    constructor(options) {
        this.adapter = new utils.Adapter(options);
        this.setupLogging();
    }

    setupLogging() {
        // Add context to logs
        this.originalLog = this.adapter.log;
        this.adapter.log = {
            silly: (msg) => this.originalLog.silly(this.formatMessage(msg, 'silly')),
            debug: (msg) => this.originalLog.debug(this.formatMessage(msg, 'debug')),
            info: (msg) => this.originalLog.info(this.formatMessage(msg, 'info')),
            warn: (msg) => this.originalLog.warn(this.formatMessage(msg, 'warn')),
            error: (msg) => this.originalLog.error(this.formatMessage(msg, 'error'))
        };
    }

    formatMessage(message, level) {
        const timestamp = new Date().toISOString();
        const context = this.getCurrentContext();
        return `[${timestamp}] [${level.toUpperCase()}] ${context ? `[${context}] ` : ''}${message}`;
    }

    getCurrentContext() {
        // Add contextual information
        if (this.currentOperation) {
            return this.currentOperation;
        }
        return null;
    }

    async withContext(context, operation) {
        this.currentOperation = context;
        try {
            return await operation();
        } finally {
            this.currentOperation = null;
        }
    }

    logPerformance(operation, duration) {
        if (duration > 1000) {
            this.adapter.log.warn(`Slow operation: ${operation} took ${duration}ms`);
        } else {
            this.adapter.log.debug(`Operation: ${operation} took ${duration}ms`);
        }
    }
}

// Usage
await adapter.withContext('device-connection', async () => {
    const start = Date.now();
    await this.connectToDevice();
    const duration = Date.now() - start;
    this.logPerformance('device-connection', duration);
});
```

## Testing

### Unit Testing Structure

```javascript
// test/adapter.test.js
const { expect } = require('chai');
const AdapterCore = require('../lib/adapter');

describe('MyAdapter', () => {
    let adapter;

    beforeEach(() => {
        adapter = new AdapterCore({
            name: 'my-adapter',
            instance: 0
        });
    });

    afterEach(() => {
        if (adapter) {
            adapter.cleanup();
        }
    });

    describe('Configuration Validation', () => {
        it('should accept valid configuration', () => {
            const config = {
                hostname: '192.168.1.100',
                port: 8080,
                username: 'user',
                password: 'pass'
            };

            expect(() => adapter.validateConfig(config)).to.not.throw();
        });

        it('should reject invalid hostname', () => {
            const config = {
                hostname: 'invalid..hostname',
                port: 8080
            };

            expect(() => adapter.validateConfig(config)).to.throw();
        });
    });

    describe('Device Management', () => {
        it('should add device successfully', async () => {
            const deviceConfig = {
                id: 'device1',
                name: 'Test Device',
                type: 'sensor'
            };

            await adapter.addDevice(deviceConfig);
            expect(adapter.devices.has('device1')).to.be.true;
        });
    });
});
```

### Integration Testing

```javascript
// test/integration.test.js
const { createTestAdapter } = require('./helpers/test-utils');

describe('Integration Tests', () => {
    let testAdapter;

    before(async () => {
        testAdapter = await createTestAdapter();
    });

    after(async () => {
        await testAdapter.cleanup();
    });

    it('should create all required objects', async () => {
        await testAdapter.start();
        
        // Check if objects were created
        const objects = await testAdapter.getObjects();
        expect(objects['my-adapter.0.info']).to.exist;
        expect(objects['my-adapter.0.info.connection']).to.exist;
    });

    it('should handle state changes correctly', async () => {
        await testAdapter.start();
        
        // Simulate state change
        await testAdapter.setState('commands.refresh', true);
        
        // Verify response
        await testAdapter.waitFor('info.connection', true, 5000);
    });
});
```

## Documentation

### Code Documentation

```javascript
/**
 * MyAdapter - ioBroker adapter for managing my devices
 * 
 * @class MyAdapter
 * @extends EventEmitter
 */
class MyAdapter extends EventEmitter {
    /**
     * Create a new adapter instance
     * 
     * @param {Object} options - Adapter options
     * @param {string} options.name - Adapter name
     * @param {number} [options.instance=0] - Instance number
     */
    constructor(options) {
        super();
        this.adapter = new utils.Adapter(options);
    }

    /**
     * Validate adapter configuration
     * 
     * @param {Object} config - Configuration object
     * @param {string} config.hostname - Device hostname
     * @param {number} config.port - Device port
     * @returns {boolean} True if valid
     * @throws {Error} If configuration is invalid
     */
    validateConfig(config) {
        // Implementation
    }

    /**
     * Connect to the target device
     * 
     * @async
     * @returns {Promise<void>}
     * @throws {Error} If connection fails
     */
    async connectToDevice() {
        // Implementation
    }

    /**
     * Add a new device to the adapter
     * 
     * @async
     * @param {Object} deviceConfig - Device configuration
     * @param {string} deviceConfig.id - Unique device ID
     * @param {string} deviceConfig.name - Device name
     * @param {string} deviceConfig.type - Device type
     * @returns {Promise<Device>} The created device instance
     * @example
     * const device = await adapter.addDevice({
     *   id: 'sensor1',
     *   name: 'Temperature Sensor',
     *   type: 'temperature'
     * });
     */
    async addDevice(deviceConfig) {
        // Implementation
    }
}
```

### README Template

```markdown
# ioBroker.my-adapter

[![NPM version](https://img.shields.io/npm/v/iobroker.my-adapter.svg)](https://www.npmjs.com/package/iobroker.my-adapter)
[![Tests](https://github.com/user/ioBroker.my-adapter/workflows/Test%20and%20Release/badge.svg)](https://github.com/user/ioBroker.my-adapter/actions/)

## Description

Brief description of what your adapter does.

## Configuration

Describe configuration options:

| Parameter | Type | Default | Description |
|-----------|------|---------|-------------|
| hostname | string | localhost | Device hostname or IP |
| port | number | 80 | Device port |
| secure | boolean | false | Use HTTPS |

## Usage

Explain how to use the adapter.

## Changelog

### 1.0.0 (2024-12-22)
- Initial release

## License

MIT License
```

By following these best practices, you'll create robust, maintainable, and secure ioBroker adapters that provide a great user experience.

<!-- 
Source metadata for automated updates:
- Based on ioBroker community best practices
- Coding patterns from official adapters
- Security guidelines from Node.js and ioBroker documentation
- Performance optimization techniques
- Testing patterns from ioBroker development guidelines
- Last updated: 2024-12-22
-->