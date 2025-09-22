# Getting Started with ioBroker Adapter Development

This guide provides an introduction to developing adapters for the ioBroker home automation platform.

## What is an ioBroker Adapter?

An ioBroker adapter is a Node.js module that extends the ioBroker platform by:
- Connecting to external devices or services
- Processing and transforming data
- Providing new functionality to the ioBroker ecosystem

## Basic Adapter Structure

Every ioBroker adapter consists of:

### 1. Main Entry Point (`main.js` or specified in package.json)
```javascript
const utils = require('@iobroker/adapter-core');
const adapter = new utils.Adapter('your-adapter-name');

adapter.on('ready', () => {
    adapter.log.info('Adapter started');
    // Your adapter logic here
});

adapter.on('stateChange', (id, state) => {
    if (state) {
        adapter.log.info(`State ${id} changed: ${state.val}`);
    }
});

adapter.on('unload', (callback) => {
    // Cleanup code
    callback();
});
```

### 2. Configuration File (`io-package.json`)
Defines adapter metadata, capabilities, and configuration schema. See [io-package.json Schema](io-package-schema.md) for complete reference.

### 3. Admin Interface (optional)
- HTML/JSON configuration pages
- Custom admin tabs
- Material-UI integration

## Core Concepts

### States
States represent data points in ioBroker:
```javascript
// Set a state
await adapter.setStateAsync('info.connection', true, true);

// Get a state
const state = await adapter.getStateAsync('sensors.temperature');
console.log(state.val); // Current value
```

### Objects
Objects define the structure and metadata for states:
```javascript
// Create an object
await adapter.setObjectAsync('sensors.temperature', {
    type: 'state',
    common: {
        name: 'Temperature',
        type: 'number',
        role: 'value.temperature',
        unit: '°C',
        read: true,
        write: false
    },
    native: {}
});
```

### Events
Adapters respond to various events:
- `ready` - Adapter is initialized
- `stateChange` - A state value changed
- `objectChange` - An object was modified
- `message` - Received a message from another adapter
- `unload` - Adapter is being stopped

## Essential Methods

### State Operations
- [`setState()`](adapter-api.md#setstate) / [`setStateAsync()`](adapter-api.md#setstateasync) - Update state values
- [`getState()`](adapter-api.md#getstate) / [`getStateAsync()`](adapter-api.md#getstateasync) - Read state values
- [`subscribeStates()`](adapter-api.md#subscribestates) - Monitor state changes

### Object Operations
- [`setObject()`](adapter-api.md#setobject) / [`setObjectAsync()`](adapter-api.md#setobjectasync) - Create/update objects
- [`getObject()`](adapter-api.md#getobject) / [`getObjectAsync()`](adapter-api.md#getobjectasync) - Read objects
- [`delObject()`](adapter-api.md#delobject) / [`delObjectAsync()`](adapter-api.md#delobjectasync) - Delete objects

### Communication
- [`sendTo()`](adapter-api.md#sendto) - Send messages to other adapters
- [`log`](adapter-api.md#log) - Logging functionality

## Development Workflow

### 1. Setup Development Environment
```bash
# Clone adapter template
git clone https://github.com/ioBroker/ioBroker.template my-adapter

# Install dependencies
cd my-adapter
npm install

# Link for local testing
npm link
cd /opt/iobroker
npm link my-adapter
```

### 2. Configure Adapter
Edit `io-package.json` to define:
- Adapter metadata
- Configuration schema
- Supported features
- Dependencies

### 3. Implement Core Logic
```javascript
class MyAdapter {
    constructor() {
        this.adapter = new utils.Adapter('my-adapter');
        this.setupEventHandlers();
    }

    setupEventHandlers() {
        this.adapter.on('ready', this.onReady.bind(this));
        this.adapter.on('stateChange', this.onStateChange.bind(this));
    }

    async onReady() {
        // Initialize adapter
        await this.createObjects();
        await this.subscribeStates('*');
        
        // Start your main logic
        this.startMainLoop();
    }

    async createObjects() {
        // Define your adapter's objects
    }

    onStateChange(id, state) {
        // Handle state changes
    }
}

new MyAdapter();
```

### 4. Testing
```bash
# Local testing
iobroker add my-adapter --custom
iobroker start my-adapter.0

# Check logs
iobroker logs my-adapter.0
```

## Best Practices

### Error Handling
```javascript
try {
    await adapter.setStateAsync('test.value', 42);
} catch (error) {
    adapter.log.error(`Failed to set state: ${error.message}`);
}
```

### Resource Cleanup
```javascript
adapter.on('unload', (callback) => {
    if (this.intervalTimer) {
        clearInterval(this.intervalTimer);
    }
    if (this.connection) {
        this.connection.close();
    }
    callback();
});
```

### Configuration Validation
```javascript
if (!adapter.config.hostname) {
    adapter.log.error('Hostname not configured');
    return;
}
```

## Next Steps

- Review the [Adapter Class API](adapter-api.md) for complete method reference
- Understand [TypeScript Types](types.md) for better development experience
- Configure your adapter with [io-package.json Schema](io-package-schema.md)
- Learn about [State Management](states.md) patterns
- Explore [Best Practices](best-practices.md) for production adapters

## Example Adapters

Study these official adapters for reference:
- [ioBroker.template](https://github.com/ioBroker/ioBroker.template) - Basic template
- [ioBroker.ping](https://github.com/ioBroker/ioBroker.ping) - Simple polling adapter
- [ioBroker.admin](https://github.com/ioBroker/ioBroker.admin) - Complex web adapter

<!-- 
Source metadata for automated updates:
- Based on: ioBroker.js-controller/packages/adapter/src/lib/adapter/adapter.ts
- Template reference: ioBroker.template repository
- Last updated: 2024-12-22
-->