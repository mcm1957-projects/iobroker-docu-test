# Events & Callbacks

This document covers event handling and callback patterns in ioBroker adapters.

## Table of Contents

- [Adapter Events](#adapter-events)
- [Event Handlers](#event-handlers)
- [Callback Patterns](#callback-patterns)
- [Custom Events](#custom-events)
- [Error Handling](#error-handling)

## Adapter Events

ioBroker adapters are EventEmitter instances that emit various lifecycle and data events.

### Core Events

#### ready
Emitted when the adapter is fully initialized and ready to start operation.

```javascript
adapter.on('ready', () => {
    adapter.log.info('Adapter is ready');
    // Initialize your adapter logic here
    initializeAdapter();
});
```

#### stateChange
Emitted when a state value changes.

```javascript
adapter.on('stateChange', (id, state) => {
    if (state) {
        adapter.log.info(`State ${id} changed: ${state.val} (ack: ${state.ack})`);
        
        // Handle only user changes (not acknowledged)
        if (!state.ack) {
            handleUserCommand(id, state.val);
        }
    } else {
        adapter.log.info(`State ${id} deleted`);
    }
});
```

#### objectChange
Emitted when an object is created, modified, or deleted.

```javascript
adapter.on('objectChange', (id, obj) => {
    if (obj) {
        adapter.log.info(`Object ${id} changed`);
        // Object was created or modified
        if (obj.type === 'state') {
            handleStateObjectChange(id, obj);
        }
    } else {
        adapter.log.info(`Object ${id} deleted`);
        handleObjectDeletion(id);
    }
});
```

#### message
Emitted when a message is received from another adapter.

```javascript
adapter.on('message', (obj) => {
    adapter.log.info(`Received message: ${obj.command}`);
    
    switch (obj.command) {
        case 'getData':
            handleGetData(obj);
            break;
        case 'setConfig':
            handleSetConfig(obj);
            break;
        default:
            adapter.log.warn(`Unknown command: ${obj.command}`);
    }
});
```

#### unload
Emitted when the adapter is being stopped.

```javascript
adapter.on('unload', (callback) => {
    adapter.log.info('Adapter is unloading');
    
    // Cleanup resources
    if (intervalTimer) {
        clearInterval(intervalTimer);
    }
    
    if (connection) {
        connection.close();
    }
    
    // Call callback to confirm shutdown
    callback();
});
```

#### log
Emitted for log messages (when using custom log handling).

```javascript
adapter.on('log', (data) => {
    // Custom log processing
    console.log(`[${data.severity}] ${data.message}`);
});
```

## Event Handlers

### Setting Up Event Handlers

```javascript
class MyAdapter {
    constructor() {
        this.adapter = new utils.Adapter('my-adapter');
        this.setupEventHandlers();
    }

    setupEventHandlers() {
        this.adapter.on('ready', this.onReady.bind(this));
        this.adapter.on('stateChange', this.onStateChange.bind(this));
        this.adapter.on('objectChange', this.onObjectChange.bind(this));
        this.adapter.on('message', this.onMessage.bind(this));
        this.adapter.on('unload', this.onUnload.bind(this));
    }

    onReady() {
        this.adapter.log.info('Adapter started');
        this.initialize();
    }

    onStateChange(id, state) {
        if (!state || state.ack) return;
        
        // Handle command
        this.handleCommand(id, state.val);
    }

    onObjectChange(id, obj) {
        if (obj && obj.type === 'device') {
            this.registerDevice(id, obj);
        }
    }

    onMessage(obj) {
        this.processMessage(obj);
    }

    onUnload(callback) {
        this.cleanup();
        callback();
    }
}
```

### Conditional Event Handling

```javascript
adapter.on('stateChange', (id, state) => {
    // Only handle states from this adapter
    if (!id.startsWith(adapter.namespace)) return;
    
    // Only handle actual changes (not acknowledgments)
    if (!state || state.ack) return;
    
    // Extract state name
    const stateName = id.replace(adapter.namespace + '.', '');
    
    switch (stateName) {
        case 'commands.power':
            handlePowerCommand(state.val);
            break;
        case 'commands.brightness':
            handleBrightnessCommand(state.val);
            break;
        default:
            adapter.log.debug(`Unhandled state change: ${stateName}`);
    }
});
```

### Subscribing to Specific States

```javascript
// Subscribe to all states
adapter.subscribeStates('*');

// Subscribe to specific pattern
adapter.subscribeStates('devices.*');

// Subscribe to foreign states
adapter.subscribeForeignStates('system.adapter.*.alive');

adapter.on('stateChange', (id, state) => {
    if (id.includes('.alive')) {
        // Handle adapter alive status
        handleAdapterStatus(id, state);
    }
});
```

## Callback Patterns

### Standard Callback Pattern

Most ioBroker methods use the Node.js callback pattern: `(error, result) => void`

```javascript
// Get state with callback
adapter.getState('sensor.temperature', (err, state) => {
    if (err) {
        adapter.log.error(`Failed to get state: ${err.message}`);
        return;
    }
    
    if (state) {
        adapter.log.info(`Temperature: ${state.val}°C`);
    }
});

// Set object with callback
adapter.setObject('newDevice', deviceObject, (err) => {
    if (err) {
        adapter.log.error(`Failed to create object: ${err.message}`);
    } else {
        adapter.log.info('Device object created successfully');
    }
});
```

### Promise-based Patterns

Use async versions of methods for cleaner code:

```javascript
async function processData() {
    try {
        // Get multiple states
        const tempState = await adapter.getStateAsync('sensor.temperature');
        const humidityState = await adapter.getStateAsync('sensor.humidity');
        
        if (tempState && humidityState) {
            const comfort = calculateComfort(tempState.val, humidityState.val);
            await adapter.setStateAsync('calculated.comfort', comfort, true);
        }
    } catch (error) {
        adapter.log.error(`Processing failed: ${error.message}`);
    }
}
```

### Error-First Callbacks

```javascript
function processWithCallback(callback) {
    adapter.getState('input.value', (err, state) => {
        if (err) {
            return callback(err);
        }
        
        if (!state) {
            return callback(new Error('State not found'));
        }
        
        // Process the value
        const result = processValue(state.val);
        callback(null, result);
    });
}

// Usage
processWithCallback((err, result) => {
    if (err) {
        adapter.log.error(`Processing failed: ${err.message}`);
    } else {
        adapter.log.info(`Result: ${result}`);
    }
});
```

### Promisifying Callbacks

```javascript
const { promisify } = require('util');

// Promisify a callback-based function
const getStatePromise = promisify(adapter.getState.bind(adapter));

async function example() {
    try {
        const state = await getStatePromise('sensor.temperature');
        adapter.log.info(`Temperature: ${state.val}`);
    } catch (error) {
        adapter.log.error(`Error: ${error.message}`);
    }
}
```

## Custom Events

### Creating Custom Events

```javascript
class MyAdapter extends EventEmitter {
    constructor() {
        super();
        this.adapter = new utils.Adapter('my-adapter');
        this.setupAdapter();
    }

    setupAdapter() {
        this.adapter.on('ready', () => {
            this.emit('adapterReady');
        });
        
        this.adapter.on('stateChange', (id, state) => {
            if (id.includes('sensor')) {
                this.emit('sensorUpdate', id, state);
            }
        });
    }

    // Custom event methods
    notifyDeviceConnected(deviceId) {
        this.emit('deviceConnected', {
            deviceId: deviceId,
            timestamp: Date.now()
        });
    }

    notifyError(error) {
        this.emit('error', {
            message: error.message,
            stack: error.stack,
            timestamp: Date.now()
        });
    }
}

// Usage
const myAdapter = new MyAdapter();

myAdapter.on('adapterReady', () => {
    console.log('My adapter is ready!');
});

myAdapter.on('sensorUpdate', (id, state) => {
    console.log(`Sensor ${id} updated: ${state.val}`);
});

myAdapter.on('deviceConnected', (data) => {
    console.log(`Device ${data.deviceId} connected at ${data.timestamp}`);
});
```

### Event Emitter with State Machine

```javascript
class DeviceManager extends EventEmitter {
    constructor(adapter) {
        super();
        this.adapter = adapter;
        this.devices = new Map();
        this.state = 'idle';
    }

    setState(newState) {
        const oldState = this.state;
        this.state = newState;
        this.emit('stateChanged', { from: oldState, to: newState });
    }

    addDevice(deviceId, config) {
        this.devices.set(deviceId, config);
        this.emit('deviceAdded', deviceId, config);
        
        if (this.state === 'idle') {
            this.setState('scanning');
        }
    }

    removeDevice(deviceId) {
        const device = this.devices.get(deviceId);
        this.devices.delete(deviceId);
        this.emit('deviceRemoved', deviceId, device);
        
        if (this.devices.size === 0) {
            this.setState('idle');
        }
    }
}

// Usage
const deviceManager = new DeviceManager(adapter);

deviceManager.on('stateChanged', ({ from, to }) => {
    adapter.log.info(`Device manager state: ${from} -> ${to}`);
});

deviceManager.on('deviceAdded', (deviceId, config) => {
    adapter.log.info(`Added device: ${deviceId}`);
});
```

## Error Handling

### Event Error Handling

```javascript
adapter.on('error', (error) => {
    adapter.log.error(`Adapter error: ${error.message}`);
    // Handle critical errors
    if (error.code === 'CRITICAL') {
        adapter.terminate('Critical error occurred');
    }
});

// Handle uncaught exceptions
process.on('uncaughtException', (error) => {
    adapter.log.error(`Uncaught exception: ${error.message}`);
    adapter.terminate('Uncaught exception', 1);
});

// Handle unhandled promise rejections
process.on('unhandledRejection', (reason, promise) => {
    adapter.log.error(`Unhandled rejection at: ${promise}, reason: ${reason}`);
});
```

### Safe Event Handling

```javascript
function safeEventHandler(handler) {
    return (...args) => {
        try {
            handler(...args);
        } catch (error) {
            adapter.log.error(`Event handler error: ${error.message}`);
        }
    };
}

// Use safe handlers
adapter.on('stateChange', safeEventHandler((id, state) => {
    // Handler code that might throw
    const result = riskyOperation(state.val);
    adapter.setState('result', result, true);
}));
```

### Timeout Handling

```javascript
function withTimeout(promise, timeoutMs) {
    return Promise.race([
        promise,
        new Promise((_, reject) => {
            setTimeout(() => reject(new Error('Operation timeout')), timeoutMs);
        })
    ]);
}

// Usage
adapter.on('message', async (obj) => {
    try {
        const result = await withTimeout(
            processMessage(obj),
            5000 // 5 second timeout
        );
        
        adapter.sendTo(obj.from, obj.command, result, obj.callback);
    } catch (error) {
        adapter.log.error(`Message processing failed: ${error.message}`);
        adapter.sendTo(obj.from, obj.command, { error: error.message }, obj.callback);
    }
});
```

### Resource Cleanup

```javascript
class ResourceManager {
    constructor(adapter) {
        this.adapter = adapter;
        this.resources = [];
        this.setupCleanup();
    }

    setupCleanup() {
        this.adapter.on('unload', (callback) => {
            this.cleanup()
                .then(() => callback())
                .catch((error) => {
                    this.adapter.log.error(`Cleanup error: ${error.message}`);
                    callback();
                });
        });
    }

    addResource(resource) {
        this.resources.push(resource);
    }

    async cleanup() {
        this.adapter.log.info('Cleaning up resources...');
        
        for (const resource of this.resources) {
            try {
                if (resource.close) {
                    await resource.close();
                } else if (resource.destroy) {
                    resource.destroy();
                }
            } catch (error) {
                this.adapter.log.error(`Failed to cleanup resource: ${error.message}`);
            }
        }
        
        this.resources = [];
        this.adapter.log.info('Cleanup completed');
    }
}

// Usage
const resourceManager = new ResourceManager(adapter);

// Add resources that need cleanup
const server = http.createServer();
resourceManager.addResource(server);

const connection = new DatabaseConnection();
resourceManager.addResource(connection);
```

## Best Practices

### Event Handler Organization

```javascript
class AdapterEventHandlers {
    constructor(adapter) {
        this.adapter = adapter;
        this.registerHandlers();
    }

    registerHandlers() {
        this.adapter.on('ready', this.handleReady.bind(this));
        this.adapter.on('stateChange', this.handleStateChange.bind(this));
        this.adapter.on('objectChange', this.handleObjectChange.bind(this));
        this.adapter.on('message', this.handleMessage.bind(this));
        this.adapter.on('unload', this.handleUnload.bind(this));
    }

    handleReady() {
        this.adapter.log.info('Adapter ready - initializing...');
        this.initialize();
    }

    handleStateChange(id, state) {
        if (!this.shouldHandleState(id, state)) return;
        
        const command = this.parseStateCommand(id, state);
        this.executeCommand(command);
    }

    shouldHandleState(id, state) {
        return state && 
               !state.ack && 
               id.startsWith(this.adapter.namespace);
    }

    parseStateCommand(id, state) {
        // Parse state change into command object
        return {
            id: id,
            value: state.val,
            timestamp: state.ts
        };
    }

    executeCommand(command) {
        // Execute the command
        this.adapter.log.debug(`Executing command: ${JSON.stringify(command)}`);
    }
}

// Usage
const eventHandlers = new AdapterEventHandlers(adapter);
```

<!-- 
Source metadata for automated updates:
- Primary source: ioBroker.js-controller/packages/adapter/src/lib/adapter/adapter.ts
- Event system based on Node.js EventEmitter
- Callback patterns from ioBroker conventions
- Error handling patterns from best practices
- Last updated: 2024-12-22
-->