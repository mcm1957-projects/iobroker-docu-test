# State Management

This document covers working with states, objects, and data in ioBroker adapters.

## Table of Contents

- [States Overview](#states-overview)
- [Object Management](#object-management)
- [State Operations](#state-operations)
- [Data Types](#data-types)
- [Quality Codes](#quality-codes)
- [Patterns and Best Practices](#patterns-and-best-practices)

## States Overview

States are the core data elements in ioBroker, representing current values of sensors, switches, and other data points.

### State Structure

```javascript
{
    val: any,           // Current value
    ack: boolean,       // Acknowledged flag
    ts: number,         // Timestamp (milliseconds)
    lc: number,         // Last change timestamp
    q: number,          // Quality code
    from: string,       // Source adapter
    user: string,       // User who changed (optional)
    c: string,          // Comment (optional)
    expire: number      // Expire time in seconds (optional)
}
```

### State vs Object

- **Object**: Defines metadata, structure, and configuration for a data point
- **State**: Contains the actual current value and metadata for that value

```javascript
// Object definition
{
    _id: 'my-adapter.0.temperature',
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
}

// State value
{
    val: 23.5,
    ack: true,
    ts: 1640995200000,
    lc: 1640995200000,
    q: 0,
    from: 'my-adapter.0'
}
```

## Object Management

### Creating Objects

```javascript
// Basic state object
await adapter.setObjectAsync('sensors.temperature', {
    type: 'state',
    common: {
        name: 'Temperature',
        type: 'number',
        role: 'value.temperature',
        unit: '°C',
        read: true,
        write: false,
        min: -50,
        max: 100,
        def: 0
    },
    native: {
        sensor_id: 'temp_001'
    }
});

// Channel for grouping
await adapter.setObjectAsync('devices.sensor1', {
    type: 'channel',
    common: {
        name: 'Temperature Sensor 1'
    },
    native: {}
});

// Device for hierarchy
await adapter.setObjectAsync('devices', {
    type: 'device',
    common: {
        name: 'Sensors'
    },
    native: {}
});
```

### Object Hierarchy

Organize objects in a logical hierarchy:

```
adapter.instance
├── info
│   ├── connection
│   └── version
├── devices
│   ├── sensor1
│   │   ├── temperature
│   │   ├── humidity
│   │   └── battery
│   └── sensor2
│       ├── temperature
│       └── status
└── commands
    ├── refresh
    └── reset
```

### Object Creation Patterns

```javascript
class ObjectManager {
    constructor(adapter) {
        this.adapter = adapter;
    }

    async createDeviceStructure(deviceId, deviceConfig) {
        // Create device
        await this.adapter.setObjectAsync(`devices.${deviceId}`, {
            type: 'device',
            common: {
                name: deviceConfig.name,
                statusStates: {
                    onlineId: `devices.${deviceId}.info.connection`
                }
            },
            native: deviceConfig
        });

        // Create info channel
        await this.adapter.setObjectAsync(`devices.${deviceId}.info`, {
            type: 'channel',
            common: {
                name: 'Information'
            },
            native: {}
        });

        // Create connection state
        await this.adapter.setObjectAsync(`devices.${deviceId}.info.connection`, {
            type: 'state',
            common: {
                name: 'Connected',
                role: 'indicator.connected',
                type: 'boolean',
                read: true,
                write: false,
                def: false
            },
            native: {}
        });

        // Create data states based on device type
        await this.createDataStates(deviceId, deviceConfig.capabilities);
    }

    async createDataStates(deviceId, capabilities) {
        for (const capability of capabilities) {
            await this.createCapabilityState(deviceId, capability);
        }
    }

    async createCapabilityState(deviceId, capability) {
        const stateId = `devices.${deviceId}.${capability.name}`;
        
        await this.adapter.setObjectAsync(stateId, {
            type: 'state',
            common: {
                name: capability.displayName,
                type: capability.dataType,
                role: capability.role,
                unit: capability.unit,
                min: capability.min,
                max: capability.max,
                read: true,
                write: capability.writable || false,
                states: capability.states
            },
            native: {
                capability: capability.name,
                deviceId: deviceId
            }
        });
    }
}

// Usage
const objectManager = new ObjectManager(adapter);

await objectManager.createDeviceStructure('sensor001', {
    name: 'Living Room Sensor',
    type: 'multi-sensor',
    capabilities: [
        {
            name: 'temperature',
            displayName: 'Temperature',
            dataType: 'number',
            role: 'value.temperature',
            unit: '°C',
            min: -40,
            max: 80
        },
        {
            name: 'humidity',
            displayName: 'Humidity',
            dataType: 'number',
            role: 'value.humidity',
            unit: '%',
            min: 0,
            max: 100
        },
        {
            name: 'power',
            displayName: 'Power Switch',
            dataType: 'boolean',
            role: 'switch.power',
            writable: true
        }
    ]
});
```

## State Operations

### Setting States

```javascript
// Simple value
await adapter.setStateAsync('temperature', 23.5, true);

// With quality code
await adapter.setStateAsync('temperature', {
    val: 23.5,
    ack: true,
    q: 0x00 // Good quality
});

// With expiration
await adapter.setStateAsync('heartbeat', {
    val: Date.now(),
    ack: true,
    expire: 60 // Expire after 60 seconds
});

// Batch updates
const updates = [
    { id: 'temperature', val: 23.5 },
    { id: 'humidity', val: 65 },
    { id: 'pressure', val: 1013.25 }
];

await Promise.all(updates.map(update => 
    adapter.setStateAsync(update.id, update.val, true)
));
```

### Reading States

```javascript
// Single state
const state = await adapter.getStateAsync('temperature');
if (state) {
    console.log(`Temperature: ${state.val}°C (${new Date(state.ts)})`);
}

// Multiple states
const states = await adapter.getStatesAsync('sensors.*');
Object.keys(states).forEach(id => {
    if (states[id]) {
        console.log(`${id}: ${states[id].val}`);
    }
});

// Foreign states (from other adapters)
const systemStates = await adapter.getForeignStatesAsync('system.adapter.*.alive');
```

### State Subscriptions

```javascript
// Subscribe to own states
await adapter.subscribeStatesAsync('*');

// Subscribe to specific pattern
await adapter.subscribeStatesAsync('sensors.*');

// Subscribe to foreign states
await adapter.subscribeForeignStatesAsync('system.adapter.admin.0.alive');

// Handle state changes
adapter.on('stateChange', (id, state) => {
    if (!state || state.ack) return; // Only handle commands
    
    const relativePath = id.replace(adapter.namespace + '.', '');
    
    switch (relativePath) {
        case 'commands.refresh':
            if (state.val) {
                handleRefreshCommand();
                // Reset command state
                adapter.setState('commands.refresh', false, true);
            }
            break;
            
        case 'commands.reset':
            if (state.val) {
                handleResetCommand();
                adapter.setState('commands.reset', false, true);
            }
            break;
            
        default:
            if (relativePath.startsWith('devices.') && relativePath.includes('.control.')) {
                handleDeviceControl(relativePath, state.val);
            }
    }
});
```

## Data Types

### Basic Data Types

```javascript
// Boolean states
await adapter.setObjectAsync('switch.power', {
    type: 'state',
    common: {
        type: 'boolean',
        role: 'switch.power',
        read: true,
        write: true,
        def: false
    }
});

// Number states
await adapter.setObjectAsync('sensor.temperature', {
    type: 'state',
    common: {
        type: 'number',
        role: 'value.temperature',
        unit: '°C',
        min: -50,
        max: 100,
        read: true,
        write: false
    }
});

// String states
await adapter.setObjectAsync('info.status', {
    type: 'state',
    common: {
        type: 'string',
        role: 'text',
        read: true,
        write: false,
        states: {
            'online': 'Online',
            'offline': 'Offline',
            'error': 'Error'
        }
    }
});

// Object/JSON states
await adapter.setObjectAsync('config.settings', {
    type: 'state',
    common: {
        type: 'object',
        role: 'json',
        read: true,
        write: true
    }
});
```

### Complex Data Structures

```javascript
// Color picker
await adapter.setObjectAsync('light.color', {
    type: 'state',
    common: {
        type: 'string',
        role: 'level.color.rgb',
        read: true,
        write: true,
        def: '#ffffff'
    }
});

// File reference
await adapter.setObjectAsync('camera.snapshot', {
    type: 'state',
    common: {
        type: 'file',
        role: 'media.image',
        read: true,
        write: false
    }
});

// Array data
await adapter.setStateAsync('history.values', JSON.stringify([
    { timestamp: Date.now(), value: 23.5 },
    { timestamp: Date.now() - 60000, value: 23.2 }
]), true);
```

### Custom State Types

```javascript
class StateTypeManager {
    static createTimestampState(id, name) {
        return {
            type: 'state',
            common: {
                name: name,
                type: 'number',
                role: 'date',
                read: true,
                write: false
            }
        };
    }

    static createEnumState(id, name, values) {
        return {
            type: 'state',
            common: {
                name: name,
                type: 'string',
                role: 'state',
                read: true,
                write: true,
                states: values
            }
        };
    }

    static createProgressState(id, name, max = 100) {
        return {
            type: 'state',
            common: {
                name: name,
                type: 'number',
                role: 'value',
                unit: '%',
                min: 0,
                max: max,
                read: true,
                write: false
            }
        };
    }
}

// Usage
await adapter.setObjectAsync('device.lastSeen', 
    StateTypeManager.createTimestampState('device.lastSeen', 'Last Seen')
);

await adapter.setObjectAsync('device.mode',
    StateTypeManager.createEnumState('device.mode', 'Mode', {
        'auto': 'Automatic',
        'manual': 'Manual',
        'off': 'Off'
    })
);
```

## Quality Codes

Quality codes indicate the reliability and status of state values.

### Common Quality Codes

```javascript
const QUALITY = {
    GOOD: 0x00,                    // Good quality
    GENERAL_PROBLEM: 0x01,         // General problem
    NO_CONNECTION: 0x02,           // Connection lost
    SUBSTITUTE_VALUE: 0x40,        // Substitute/cached value
    DEVICE_ERROR: 0x12,           // Device error
    SENSOR_ERROR: 0x20,           // Sensor error
    INITIAL_VALUE: 0x41           // Initial value
};

// Set state with quality
await adapter.setStateAsync('sensor.temperature', {
    val: 23.5,
    ack: true,
    q: QUALITY.GOOD
});

// Set substitute value when device is offline
await adapter.setStateAsync('sensor.temperature', {
    val: lastKnownTemperature,
    ack: true,
    q: QUALITY.SUBSTITUTE_VALUE
});

// Indicate device error
await adapter.setStateAsync('sensor.temperature', {
    val: null,
    ack: true,
    q: QUALITY.DEVICE_ERROR
});
```

### Quality-Aware Reading

```javascript
adapter.on('stateChange', (id, state) => {
    if (!state) return;
    
    switch (state.q) {
        case QUALITY.GOOD:
            // Process reliable data
            processGoodValue(id, state.val);
            break;
            
        case QUALITY.SUBSTITUTE_VALUE:
            // Handle cached/substitute data
            adapter.log.warn(`Using substitute value for ${id}`);
            processSubstituteValue(id, state.val);
            break;
            
        case QUALITY.DEVICE_ERROR:
            // Handle device errors
            adapter.log.error(`Device error for ${id}`);
            handleDeviceError(id);
            break;
            
        default:
            adapter.log.warn(`Unknown quality code ${state.q} for ${id}`);
    }
});
```

## Patterns and Best Practices

### State Caching

```javascript
class StateCache {
    constructor(adapter) {
        this.adapter = adapter;
        this.cache = new Map();
        this.loadCache();
    }

    async loadCache() {
        const states = await this.adapter.getStatesAsync('*');
        Object.keys(states).forEach(id => {
            if (states[id]) {
                this.cache.set(id, states[id]);
            }
        });
    }

    async setState(id, value, ack = true) {
        const fullId = this.adapter.namespace + '.' + id;
        const cached = this.cache.get(fullId);
        
        // Only update if value changed
        if (!cached || cached.val !== value) {
            const state = { val: value, ack: ack, ts: Date.now() };
            this.cache.set(fullId, state);
            await this.adapter.setStateAsync(id, value, ack);
        }
    }

    getState(id) {
        const fullId = this.adapter.namespace + '.' + id;
        return this.cache.get(fullId);
    }
}
```

### Structured State Updates

```javascript
class DeviceStateManager {
    constructor(adapter, deviceId) {
        this.adapter = adapter;
        this.deviceId = deviceId;
        this.lastUpdate = new Map();
    }

    async updateStates(data) {
        const updates = [];

        // Connection status
        if (data.connected !== undefined) {
            updates.push({
                id: `devices.${this.deviceId}.info.connection`,
                value: data.connected
            });
        }

        // Sensor data
        if (data.temperature !== undefined) {
            updates.push({
                id: `devices.${this.deviceId}.temperature`,
                value: data.temperature,
                quality: data.tempQuality || 0x00
            });
        }

        if (data.humidity !== undefined) {
            updates.push({
                id: `devices.${this.deviceId}.humidity`,
                value: data.humidity,
                quality: data.humidityQuality || 0x00
            });
        }

        // Battery level
        if (data.battery !== undefined) {
            updates.push({
                id: `devices.${this.deviceId}.battery`,
                value: data.battery,
                quality: data.battery < 10 ? 0x20 : 0x00 // Low battery = sensor error
            });
        }

        // Apply all updates
        await this.applyUpdates(updates);

        // Update last seen
        await this.adapter.setStateAsync(
            `devices.${this.deviceId}.info.lastSeen`,
            Date.now(),
            true
        );
    }

    async applyUpdates(updates) {
        const promises = updates.map(update => {
            const state = {
                val: update.value,
                ack: true,
                q: update.quality || 0x00
            };

            return this.adapter.setStateAsync(update.id, state);
        });

        await Promise.allSettled(promises);
    }
}
```

### State Validation

```javascript
class StateValidator {
    static validateTemperature(value) {
        if (typeof value !== 'number') return false;
        return value >= -273.15 && value <= 1000; // Absolute zero to reasonable max
    }

    static validateHumidity(value) {
        if (typeof value !== 'number') return false;
        return value >= 0 && value <= 100;
    }

    static validatePressure(value) {
        if (typeof value !== 'number') return false;
        return value >= 800 && value <= 1200; // hPa
    }

    static validateColor(value) {
        if (typeof value !== 'string') return false;
        return /^#[0-9A-Fa-f]{6}$/.test(value);
    }
}

// Usage in state updates
async function updateTemperature(deviceId, temperature) {
    if (!StateValidator.validateTemperature(temperature)) {
        adapter.log.warn(`Invalid temperature value: ${temperature}`);
        return;
    }

    await adapter.setStateAsync(`devices.${deviceId}.temperature`, temperature, true);
}
```

### State Aggregation

```javascript
class StateAggregator {
    constructor(adapter) {
        this.adapter = adapter;
        this.aggregationInterval = setInterval(() => {
            this.performAggregation();
        }, 60000); // Every minute
    }

    async performAggregation() {
        await this.aggregateTemperatures();
        await this.aggregateDeviceStatus();
    }

    async aggregateTemperatures() {
        const tempStates = await this.adapter.getStatesAsync('devices.*.temperature');
        const temperatures = Object.values(tempStates)
            .filter(state => state && typeof state.val === 'number')
            .map(state => state.val);

        if (temperatures.length > 0) {
            const avg = temperatures.reduce((a, b) => a + b) / temperatures.length;
            const min = Math.min(...temperatures);
            const max = Math.max(...temperatures);

            await this.adapter.setStateAsync('aggregated.temperature.average', avg, true);
            await this.adapter.setStateAsync('aggregated.temperature.minimum', min, true);
            await this.adapter.setStateAsync('aggregated.temperature.maximum', max, true);
        }
    }

    async aggregateDeviceStatus() {
        const connectionStates = await this.adapter.getStatesAsync('devices.*.info.connection');
        const connectedDevices = Object.values(connectionStates)
            .filter(state => state && state.val === true).length;
        const totalDevices = Object.keys(connectionStates).length;

        await this.adapter.setStateAsync('info.devicesConnected', connectedDevices, true);
        await this.adapter.setStateAsync('info.devicesTotal', totalDevices, true);
        await this.adapter.setStateAsync('info.allDevicesOnline', connectedDevices === totalDevices, true);
    }

    cleanup() {
        if (this.aggregationInterval) {
            clearInterval(this.aggregationInterval);
        }
    }
}
```

<!-- 
Source metadata for automated updates:
- Primary source: ioBroker.js-controller/packages/adapter/src/lib/adapter/adapter.ts
- State structure from ioBroker state definitions
- Quality codes from OPC UA/IEC 62541 standards used by ioBroker
- Object patterns from ioBroker documentation and examples
- Best practices from community adapters
- Last updated: 2024-12-22
-->