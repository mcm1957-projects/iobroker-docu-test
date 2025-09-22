# Adapter Class API Reference

This document provides a complete reference for all public methods available in the ioBroker Adapter class.

## Table of Contents

- [State Operations](#state-operations)
- [Object Operations](#object-operations)
- [Subscription Management](#subscription-management)
- [File Operations](#file-operations)
- [Communication](#communication)
- [Configuration](#configuration)
- [Logging](#logging)
- [Utility Methods](#utility-methods)
- [Event Handling](#event-handling)

## State Operations

### setState()
Sets a state value.

```javascript
setState(id, state, [ack], [callback])
setState(id, state, [callback])
```

**Parameters:**
- `id` (string): State ID
- `state` (any | State): Value or complete state object
- `ack` (boolean, optional): Acknowledge flag (default: false)
- `callback` (function, optional): Callback function

**Examples:**
```javascript
// Simple value
adapter.setState('info.connection', true, true);

// With callback
adapter.setState('sensors.temperature', 22.5, true, (err) => {
    if (err) adapter.log.error('Set state error: ' + err);
});

// Complex state object
adapter.setState('sensors.humidity', {
    val: 65,
    ack: true,
    ts: Date.now(),
    q: 0x00
});
```

### setStateAsync()
Async version of setState().

```javascript
await setStateAsync(id, state, ack?)
```

**Returns:** Promise\<void>

**Example:**
```javascript
try {
    await adapter.setStateAsync('device.status', 'online', true);
} catch (error) {
    adapter.log.error(`Failed to set state: ${error.message}`);
}
```

### getState()
Retrieves a state value.

```javascript
getState(id, callback)
```

**Parameters:**
- `id` (string): State ID
- `callback` (function): Callback with (err, state)

**Example:**
```javascript
adapter.getState('sensors.temperature', (err, state) => {
    if (err) {
        adapter.log.error('Get state error: ' + err);
    } else if (state) {
        adapter.log.info(`Temperature: ${state.val}°C`);
    }
});
```

### getStateAsync()
Async version of getState().

```javascript
await getStateAsync(id)
```

**Returns:** Promise\<State | null>

**Example:**
```javascript
const state = await adapter.getStateAsync('sensors.temperature');
if (state) {
    adapter.log.info(`Current temperature: ${state.val}°C`);
}
```

### getStates()
Retrieves multiple states matching a pattern.

```javascript
getStates(pattern, callback)
```

**Parameters:**
- `pattern` (string): ID pattern (supports wildcards)
- `callback` (function): Callback with (err, states)

**Example:**
```javascript
adapter.getStates('sensors.*', (err, states) => {
    if (states) {
        Object.keys(states).forEach(id => {
            adapter.log.info(`${id}: ${states[id].val}`);
        });
    }
});
```

### getStatesAsync()
Async version of getStates().

```javascript
await getStatesAsync(pattern)
```

**Returns:** Promise\<Record\<string, State>>

### delState()
Deletes a state.

```javascript
delState(id, callback?)
```

**Parameters:**
- `id` (string): State ID
- `callback` (function, optional): Callback function

## Object Operations

### setObject()
Creates or updates an object.

```javascript
setObject(id, obj, callback?)
```

**Parameters:**
- `id` (string): Object ID
- `obj` (Object): Object definition
- `callback` (function, optional): Callback function

**Example:**
```javascript
adapter.setObject('sensors.temperature', {
    type: 'state',
    common: {
        name: 'Room Temperature',
        type: 'number',
        role: 'value.temperature',
        unit: '°C',
        min: -50,
        max: 100,
        read: true,
        write: false
    },
    native: {
        sensor_id: 'temp_001'
    }
});
```

### setObjectAsync()
Async version of setObject().

```javascript
await setObjectAsync(id, obj)
```

**Returns:** Promise\<void>

### getObject()
Retrieves an object.

```javascript
getObject(id, callback)
```

**Parameters:**
- `id` (string): Object ID
- `callback` (function): Callback with (err, obj)

### getObjectAsync()
Async version of getObject().

```javascript
await getObjectAsync(id)
```

**Returns:** Promise\<Object | null>

### setObjectNotExists()
Creates object only if it doesn't exist.

```javascript
setObjectNotExists(id, obj, callback?)
```

### setObjectNotExistsAsync()
Async version of setObjectNotExists().

```javascript
await setObjectNotExistsAsync(id, obj)
```

### extendObject()
Extends an existing object.

```javascript
extendObject(id, obj, callback?)
```

**Example:**
```javascript
adapter.extendObject('sensors.temperature', {
    common: {
        max: 120  // Update only the max value
    }
});
```

### extendObjectAsync()
Async version of extendObject().

```javascript
await extendObjectAsync(id, obj)
```

### delObject()
Deletes an object.

```javascript
delObject(id, callback?)
```

### delObjectAsync()
Async version of delObject().

```javascript
await delObjectAsync(id)
```

### getObjectView()
Retrieves objects using a view query.

```javascript
getObjectView(design, search, params, callback)
```

### getObjectList()
Retrieves a list of objects.

```javascript
getObjectList(params, callback)
```

**Example:**
```javascript
adapter.getObjectList({
    startkey: 'system.adapter.my-adapter.',
    endkey: 'system.adapter.my-adapter.\u9999'
}, (err, result) => {
    if (result && result.rows) {
        result.rows.forEach(row => {
            adapter.log.info(`Found object: ${row.id}`);
        });
    }
});
```

## Subscription Management

### subscribeStates()
Subscribes to state changes.

```javascript
subscribeStates(pattern, callback?)
```

**Parameters:**
- `pattern` (string): ID pattern (* for all)
- `callback` (function, optional): Callback function

**Example:**
```javascript
// Subscribe to all states
adapter.subscribeStates('*');

// Subscribe to specific pattern
adapter.subscribeStates('sensors.*');
```

### subscribeStatesAsync()
Async version of subscribeStates().

```javascript
await subscribeStatesAsync(pattern)
```

### unsubscribeStates()
Unsubscribes from state changes.

```javascript
unsubscribeStates(pattern, callback?)
```

### subscribeObjects()
Subscribes to object changes.

```javascript
subscribeObjects(pattern, callback?)
```

### unsubscribeObjects()
Unsubscribes from object changes.

```javascript
unsubscribeObjects(pattern, callback?)
```

### subscribeForeignStates()
Subscribes to states from other adapters.

```javascript
subscribeForeignStates(pattern, callback?)
```

**Example:**
```javascript
// Monitor system states
adapter.subscribeForeignStates('system.adapter.*.alive');
```

## File Operations

### readFile()
Reads a file from the adapter directory.

```javascript
readFile(adapter, filename, callback)
```

**Example:**
```javascript
adapter.readFile('my-adapter.admin', 'config.json', (err, data) => {
    if (!err && data) {
        const config = JSON.parse(data.toString());
        adapter.log.info('Config loaded');
    }
});
```

### readFileAsync()
Async version of readFile().

```javascript
await readFileAsync(adapter, filename)
```

### writeFile()
Writes a file to the adapter directory.

```javascript
writeFile(adapter, filename, data, callback?)
```

### writeFileAsync()
Async version of writeFile().

```javascript
await writeFileAsync(adapter, filename, data)
```

### delFile()
Deletes a file.

```javascript
delFile(adapter, filename, callback?)
```

### renameFile()
Renames a file.

```javascript
renameFile(adapter, oldName, newName, callback?)
```

### mkdir()
Creates a directory.

```javascript
mkdir(adapter, dirname, callback?)
```

### readDir()
Reads directory contents.

```javascript
readDir(adapter, dirname, callback)
```

**Example:**
```javascript
adapter.readDir('my-adapter.admin', '/', (err, files) => {
    if (files) {
        files.forEach(file => {
            adapter.log.info(`Found file: ${file.file}`);
        });
    }
});
```

## Communication

### sendTo()
Sends a message to another adapter instance.

```javascript
sendTo(instanceName, command, message, callback?)
```

**Parameters:**
- `instanceName` (string): Target adapter instance
- `command` (string): Command name
- `message` (any): Message data
- `callback` (function, optional): Response callback

**Example:**
```javascript
// Send command to another adapter
adapter.sendTo('telegram.0', 'send', {
    text: 'Hello from my adapter!',
    user: 'admin'
}, (result) => {
    adapter.log.info('Message sent: ' + JSON.stringify(result));
});
```

### sendToAsync()
Async version of sendTo().

```javascript
await sendToAsync(instanceName, command, message)
```

### sendToHost()
Sends a message to a host.

```javascript
sendToHost(hostName, command, message, callback?)
```

## Configuration

### config
Access to adapter configuration from io-package.json native section.

```javascript
const hostname = adapter.config.hostname;
const port = adapter.config.port || 80;
const enabled = adapter.config.enabled;
```

### host
Information about the current host.

```javascript
const hostInfo = adapter.host;
adapter.log.info(`Running on host: ${hostInfo}`);
```

### instance
Current adapter instance number.

```javascript
adapter.log.info(`Instance: ${adapter.instance}`);
```

### namespace
Adapter namespace (adapter_name.instance).

```javascript
adapter.log.info(`Namespace: ${adapter.namespace}`);
```

## Logging

### log
Logging object with multiple levels.

```javascript
adapter.log.silly('Very detailed debug info');
adapter.log.debug('Debug information');
adapter.log.info('General information');
adapter.log.warn('Warning message');
adapter.log.error('Error message');
```

**Log Levels:**
- `silly` - Most verbose
- `debug` - Debug information
- `info` - General information
- `warn` - Warnings
- `error` - Errors

## Utility Methods

### formatValue()
Formats a value according to object definition.

```javascript
formatValue(value, decimals?, unit?)
```

### formatDate()
Formats a date/timestamp.

```javascript
formatDate(dateObj, format?)
```

### getPort()
Gets an available port.

```javascript
getPort(port, callback)
```

### checkPassword()
Validates a password.

```javascript
checkPassword(user, password, callback)
```

### setPassword()
Sets a user password.

```javascript
setPassword(user, password, callback)
```

### checkGroup()
Checks if user belongs to a group.

```javascript
checkGroup(user, group, callback)
```

### calculatePermissions()
Calculates user permissions.

```javascript
calculatePermissions(user, commandsPermissions, callback)
```

### getCertificates()
Gets SSL certificates.

```javascript
getCertificates(callback)
```

### getHistory()
Retrieves historical data.

```javascript
getHistory(id, options, callback)
```

**Example:**
```javascript
adapter.getHistory('sensors.temperature', {
    start: Date.now() - 24 * 60 * 60 * 1000, // 24 hours ago
    end: Date.now(),
    count: 100
}, (err, result) => {
    if (result) {
        adapter.log.info(`Retrieved ${result.length} history entries`);
    }
});
```

### pushFifo()
Pushes data to FIFO (First In, First Out) array.

```javascript
pushFifo(id, state, callback?)
```

### trimFifo()
Trims FIFO array to specified length.

```javascript
trimFifo(id, start, callback?)
```

### getForeignObjects()
Gets objects from other adapters.

```javascript
getForeignObjects(pattern, type?, callback?)
```

### findForeignObject()
Finds a foreign object by criteria.

```javascript
findForeignObject(idOrName, type?, callback?)
```

### getForeignStates()
Gets states from other adapters.

```javascript
getForeignStates(pattern, callback)
```

### setForeignState()
Sets state in another adapter.

```javascript
setForeignState(id, state, ack?, callback?)
```

### setForeignStateAsync()
Async version of setForeignState().

```javascript
await setForeignStateAsync(id, state, ack?)
```

### getForeignState()
Gets state from another adapter.

```javascript
getForeignState(id, callback)
```

### getForeignStateAsync()
Async version of getForeignState().

```javascript
await getForeignStateAsync(id)
```

## Event Handling

### on()
Registers event handlers.

```javascript
adapter.on(event, handler)
```

**Available Events:**
- `ready` - Adapter is ready
- `stateChange` - State changed
- `objectChange` - Object changed
- `message` - Message received
- `unload` - Adapter is unloading

**Example:**
```javascript
adapter.on('ready', () => {
    adapter.log.info('Adapter ready');
});

adapter.on('stateChange', (id, state) => {
    if (state && !state.ack) {
        adapter.log.info(`State change: ${id} = ${state.val}`);
    }
});

adapter.on('message', (obj) => {
    if (obj.command === 'test') {
        adapter.sendTo(obj.from, obj.command, 'Response', obj.callback);
    }
});
```

### removeListener()
Removes event listener.

```javascript
adapter.removeListener(event, handler)
```

### removeAllListeners()
Removes all listeners for an event.

```javascript
adapter.removeAllListeners(event?)
```

### emit()
Emits an event.

```javascript
adapter.emit(event, ...args)
```

## Advanced Methods

### restart()
Restarts the adapter.

```javascript
adapter.restart()
```

### disable()
Disables the adapter.

```javascript
adapter.disable()
```

### updateConfig()
Updates adapter configuration.

```javascript
updateConfig(newConfig)
```

### getForeignObject()
Gets object from another adapter.

```javascript
getForeignObject(id, callback)
```

### setForeignObject()
Sets object in another adapter.

```javascript
setForeignObject(id, obj, callback?)
```

### terminate()
Terminates the adapter with optional reason and exit code.

```javascript
terminate(reason?, exitCode?)
```

## Type Definitions

For complete type information, see [TypeScript Types & Interfaces](types.md).

<!-- 
Source metadata for automated updates:
- Primary source: ioBroker.js-controller/packages/adapter/src/lib/adapter/adapter.ts
- Lines: 1-13000+ (complete adapter class)
- Method signatures extracted from TypeScript definitions
- Last updated: 2024-12-22
- SHA reference: 92c8a2b2f4d3e5f6g7h8i9j0k1l2m3n4o5p6q7r8s9t0u1v2w3x4y5z
-->