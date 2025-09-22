# Constants

This document provides a reference for all public constants and enumerations available in the ioBroker.js-controller.

## Table of Contents

- [Quality Codes](#quality-codes)
- [Object Types](#object-types)
- [State Roles](#state-roles)
- [Log Levels](#log-levels)
- [File Constants](#file-constants)
- [System Constants](#system-constants)
- [Error Codes](#error-codes)

## Quality Codes

Quality codes indicate the reliability and status of state values.

```typescript
export const QUALITY_CODES = {
    /** Good quality - value is reliable */
    GOOD: 0x00,
    
    /** General problem */
    GENERAL_PROBLEM: 0x01,
    
    /** No connection to source but value might be from cache */
    NO_CONNECTION_PROBLEM: 0x02,
    
    /** Communication error */
    COMM_ERROR: 0x10,
    
    /** Device not connected */
    DEVICE_NOT_CONNECTED: 0x11,
    
    /** Device reports an error */
    DEVICE_ERROR: 0x12,
    
    /** Device not responding */
    DEVICE_NOT_RESPONDING: 0x13,
    
    /** Device is out of service */
    OUT_OF_SERVICE: 0x14,
    
    /** Sensor error */
    SENSOR_ERROR: 0x20,
    
    /** Last known value (substitute) */
    SUBSTITUTE_VALUE: 0x40,
    
    /** Initial value after startup */
    INITIAL_VALUE: 0x41,
    
    /** Value set by another process */
    SUBSTITUTE_INITIAL_VALUE: 0x42,
    
    /** Value set from device with error */
    SUBSTITUTE_DEVICE_ERROR: 0x44,
    
    /** Value set from communication error */
    SUBSTITUTE_SENSOR_ERROR: 0x48
} as const;
```

### Usage Examples

```javascript
// Set state with good quality
adapter.setState('sensor.temperature', {
    val: 23.5,
    ack: true,
    q: adapter.constants.QUALITY_CODES.GOOD
});

// Set state indicating device error
adapter.setState('sensor.humidity', {
    val: null,
    ack: true,
    q: adapter.constants.QUALITY_CODES.DEVICE_ERROR
});

// Check quality in state change handler
adapter.on('stateChange', (id, state) => {
    if (state && state.q !== adapter.constants.QUALITY_CODES.GOOD) {
        adapter.log.warn(`State ${id} has quality issues: ${state.q}`);
    }
});
```

## Object Types

Constants for ioBroker object types.

```typescript
export const OBJECT_TYPES = {
    /** Adapter definition */
    ADAPTER: 'adapter',
    
    /** Channel grouping states */
    CHANNEL: 'channel',
    
    /** Device containing channels */
    DEVICE: 'device',
    
    /** Enumeration/category */
    ENUM: 'enum',
    
    /** Folder for organization */
    FOLDER: 'folder',
    
    /** User group */
    GROUP: 'group',
    
    /** Host system */
    HOST: 'host',
    
    /** Adapter instance */
    INSTANCE: 'instance',
    
    /** Metadata object */
    META: 'meta',
    
    /** Script object */
    SCRIPT: 'script',
    
    /** State/data point */
    STATE: 'state',
    
    /** User account */
    USER: 'user'
} as const;
```

### Usage Examples

```javascript
// Create a device object
adapter.setObject('devices.sensor1', {
    type: adapter.constants.OBJECT_TYPES.DEVICE,
    common: {
        name: 'Temperature Sensor 1'
    },
    native: {}
});

// Create a channel
adapter.setObject('devices.sensor1.measurements', {
    type: adapter.constants.OBJECT_TYPES.CHANNEL,
    common: {
        name: 'Measurements'
    },
    native: {}
});

// Create a state
adapter.setObject('devices.sensor1.measurements.temperature', {
    type: adapter.constants.OBJECT_TYPES.STATE,
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

## State Roles

Predefined roles that describe the purpose and behavior of states.

```typescript
export const STATE_ROLES = {
    // Indicators
    INDICATOR: 'indicator',
    INDICATOR_WORKING: 'indicator.working',
    INDICATOR_REACHABLE: 'indicator.reachable',
    INDICATOR_CONNECTED: 'indicator.connected',
    INDICATOR_MAINTENANCE: 'indicator.maintenance',
    INDICATOR_MAINTENANCE_UNREACH: 'indicator.maintenance.unreach',
    INDICATOR_MAINTENANCE_LOWBAT: 'indicator.maintenance.lowbat',
    INDICATOR_MAINTENANCE_ALARM: 'indicator.maintenance.alarm',
    INDICATOR_LOWBAT: 'indicator.lowbat',
    INDICATOR_ALARM: 'indicator.alarm',
    INDICATOR_ALARM_FIRE: 'indicator.alarm.fire',
    INDICATOR_ALARM_FLOOD: 'indicator.alarm.flood',
    INDICATOR_ALARM_SECURE: 'indicator.alarm.secure',
    INDICATOR_ALARM_HEALTH: 'indicator.alarm.health',
    
    // Values
    VALUE: 'value',
    VALUE_WINDOW: 'value.window',
    VALUE_TEMPERATURE: 'value.temperature',
    VALUE_HUMIDITY: 'value.humidity',
    VALUE_BRIGHTNESS: 'value.brightness',
    VALUE_MIN: 'value.min',
    VALUE_MAX: 'value.max',
    VALUE_DEFAULT: 'value.default',
    VALUE_BATTERY: 'value.battery',
    VALUE_VALVE: 'value.valve',
    VALUE_TIME: 'value.time',
    VALUE_INTERVAL: 'value.interval',
    VALUE_DATE: 'value.date',
    VALUE_DATETIME: 'value.datetime',
    VALUE_GPS: 'value.gps',
    VALUE_GPS_LONGITUDE: 'value.gps.longitude',
    VALUE_GPS_LATITUDE: 'value.gps.latitude',
    VALUE_GPS_ELEVATION: 'value.gps.elevation',
    VALUE_POWER_CONSUMPTION: 'value.power.consumption',
    VALUE_POWER: 'value.power',
    VALUE_CURRENT: 'value.current',
    VALUE_VOLTAGE: 'value.voltage',
    VALUE_ENERGY: 'value.energy',
    VALUE_VOLUME: 'value.volume',
    VALUE_PRESSURE: 'value.pressure',
    VALUE_DISTANCE: 'value.distance',
    VALUE_SPEED: 'value.speed',
    VALUE_WEIGHT: 'value.weight',
    
    // Levels
    LEVEL: 'level',
    LEVEL_DIMMER: 'level.dimmer',
    LEVEL_BLIND: 'level.blind',
    LEVEL_TEMPERATURE: 'level.temperature',
    LEVEL_VOLUME: 'level.volume',
    LEVEL_COLOR_RED: 'level.color.red',
    LEVEL_COLOR_GREEN: 'level.color.green',
    LEVEL_COLOR_BLUE: 'level.color.blue',
    LEVEL_COLOR_WHITE: 'level.color.white',
    LEVEL_COLOR_RGB: 'level.color.rgb',
    LEVEL_COLOR_HUE: 'level.color.hue',
    LEVEL_COLOR_SATURATION: 'level.color.saturation',
    LEVEL_COLOR_LUMINANCE: 'level.color.luminance',
    LEVEL_COLOR_TEMPERATURE: 'level.color.temperature',
    
    // Switches
    SWITCH: 'switch',
    SWITCH_POWER: 'switch.power',
    SWITCH_LIGHT: 'switch.light',
    SWITCH_COMFORT: 'switch.comfort',
    SWITCH_ENABLE: 'switch.enable',
    SWITCH_LOCK: 'switch.lock',
    SWITCH_MODE: 'switch.mode',
    
    // Buttons
    BUTTON: 'button',
    BUTTON_STOP: 'button.stop',
    BUTTON_START: 'button.start',
    BUTTON_OPEN: 'button.open',
    BUTTON_CLOSE: 'button.close',
    BUTTON_OPEN_DOOR: 'button.open.door',
    BUTTON_OPEN_WINDOW: 'button.open.window',
    BUTTON_MODE: 'button.mode',
    BUTTON_MODE_AUTO: 'button.mode.auto',
    BUTTON_MODE_MANUAL: 'button.mode.manual',
    BUTTON_MODE_SILENT: 'button.mode.silent',
    
    // Texts
    TEXT: 'text',
    TEXT_URL: 'text.url',
    TEXT_IP: 'text.ip',
    TEXT_MAC: 'text.mac',
    TEXT_EMAIL: 'text.email',
    TEXT_PHONE: 'text.phone',
    
    // Lists
    LIST: 'list',
    
    // HTML
    HTML: 'html',
    
    // JSON
    JSON: 'json',
    
    // States
    STATE: 'state',
    
    // Weather
    WEATHER_ICON: 'weather.icon',
    WEATHER_CHART_URL: 'weather.chart.url',
    WEATHER_CHART_URL_FORECAST: 'weather.chart.url.forecast',
    WEATHER_HTML: 'weather.html',
    WEATHER_TITLE: 'weather.title',
    WEATHER_TITLE_SHORT: 'weather.title.short',
    WEATHER_TYPE: 'weather.type',
    WEATHER_JSON: 'weather.json',
    
    // Dates
    DATE: 'date',
    DATE_START: 'date.start',
    DATE_END: 'date.end',
    
    // URLs
    URL: 'url',
    URL_ICON: 'url.icon',
    URL_CAM: 'url.cam',
    URL_BLANK: 'url.blank',
    URL_SAME: 'url.same',
    URL_AUDIO: 'url.audio',
    
    // Media
    MEDIA_DURATION: 'media.duration',
    MEDIA_DURATION_TEXT: 'media.duration.text',
    MEDIA_ELAPSED: 'media.elapsed',
    MEDIA_ELAPSED_TEXT: 'media.elapsed.text',
    MEDIA_SEEK: 'media.seek',
    MEDIA_EPISODE: 'media.episode',
    MEDIA_SEASON: 'media.season',
    MEDIA_COVER: 'media.cover',
    MEDIA_COVER_BIG: 'media.cover.big',
    MEDIA_ARTIST: 'media.artist',
    MEDIA_ALBUM: 'media.album',
    MEDIA_TITLE: 'media.title',
    MEDIA_TITLE_NEXT: 'media.title.next',
    MEDIA_STATE: 'media.state',
    MEDIA_MUTE: 'media.mute',
    MEDIA_BITRATE: 'media.bitrate',
    MEDIA_GENRE: 'media.genre',
    MEDIA_DATE: 'media.date',
    MEDIA_TRACK: 'media.track',
    MEDIA_PLAYID: 'media.playid',
    MEDIA_ADD: 'media.add',
    MEDIA_CLEAR: 'media.clear',
    MEDIA_PLAYLIST: 'media.playlist',
    MEDIA_URL: 'media.url',
    MEDIA_URL_ANNOUNCEMENT: 'media.url.announcement',
    MEDIA_JUMP: 'media.jump',
    MEDIA_REPEAT: 'media.repeat',
    MEDIA_SHUFFLE: 'media.shuffle',
    MEDIA_BROWSE: 'media.browse',
    MEDIA_INPUT: 'media.input'
} as const;
```

### Usage Examples

```javascript
// Create temperature sensor
adapter.setObject('sensors.temperature', {
    type: 'state',
    common: {
        name: 'Temperature',
        role: adapter.constants.STATE_ROLES.VALUE_TEMPERATURE,
        type: 'number',
        unit: '°C',
        read: true,
        write: false
    },
    native: {}
});

// Create light switch
adapter.setObject('lights.living_room', {
    type: 'state',
    common: {
        name: 'Living Room Light',
        role: adapter.constants.STATE_ROLES.SWITCH_LIGHT,
        type: 'boolean',
        read: true,
        write: true
    },
    native: {}
});

// Create dimmer level
adapter.setObject('lights.bedroom.brightness', {
    type: 'state',
    common: {
        name: 'Bedroom Light Brightness',
        role: adapter.constants.STATE_ROLES.LEVEL_DIMMER,
        type: 'number',
        min: 0,
        max: 100,
        unit: '%',
        read: true,
        write: true
    },
    native: {}
});
```

## Log Levels

Constants for logging levels.

```typescript
export const LOG_LEVELS = {
    /** Most verbose logging */
    SILLY: 'silly',
    
    /** Debug information */
    DEBUG: 'debug',
    
    /** General information */
    INFO: 'info',
    
    /** Warning messages */
    WARN: 'warn',
    
    /** Error messages */
    ERROR: 'error'
} as const;

export const LOG_LEVEL_VALUES = {
    [LOG_LEVELS.SILLY]: 0,
    [LOG_LEVELS.DEBUG]: 1,
    [LOG_LEVELS.INFO]: 2,
    [LOG_LEVELS.WARN]: 3,
    [LOG_LEVELS.ERROR]: 4
} as const;
```

### Usage Examples

```javascript
// Check if debug logging is enabled
if (adapter.log.level === adapter.constants.LOG_LEVELS.DEBUG) {
    adapter.log.debug('Debug mode enabled');
}

// Set different log levels programmatically
adapter.log.silly('Very detailed debug info');
adapter.log.debug('Debug information');
adapter.log.info('General information');
adapter.log.warn('Warning message');
adapter.log.error('Error occurred');
```

## File Constants

Constants related to file operations.

```typescript
export const FILE_RIGHTS = {
    /** Read permission */
    READ: 0x400,
    
    /** Write permission */
    WRITE: 0x200,
    
    /** Execute permission */
    EXECUTE: 0x100,
    
    /** Owner read */
    OWNER_READ: 0x400,
    
    /** Owner write */
    OWNER_WRITE: 0x200,
    
    /** Owner execute */
    OWNER_EXECUTE: 0x100,
    
    /** Group read */
    GROUP_READ: 0x040,
    
    /** Group write */
    GROUP_WRITE: 0x020,
    
    /** Group execute */
    GROUP_EXECUTE: 0x010,
    
    /** Other read */
    OTHER_READ: 0x004,
    
    /** Other write */
    OTHER_WRITE: 0x002,
    
    /** Other execute */
    OTHER_EXECUTE: 0x001
} as const;

export const FILE_ENCODING = {
    /** UTF-8 encoding */
    UTF8: 'utf8',
    
    /** ASCII encoding */
    ASCII: 'ascii',
    
    /** Base64 encoding */
    BASE64: 'base64',
    
    /** Binary encoding */
    BINARY: 'binary',
    
    /** Hexadecimal encoding */
    HEX: 'hex'
} as const;
```

### Usage Examples

```javascript
// Read file with specific encoding
adapter.readFile('my-adapter.admin', 'config.json', {
    encoding: adapter.constants.FILE_ENCODING.UTF8
}, (err, data) => {
    if (!err && data) {
        const config = JSON.parse(data.toString());
    }
});

// Write file with permissions
adapter.writeFile('my-adapter.admin', 'data.txt', 'Hello World', {
    mode: adapter.constants.FILE_RIGHTS.OWNER_READ | 
          adapter.constants.FILE_RIGHTS.OWNER_WRITE
});
```

## System Constants

System-level constants and identifiers.

```typescript
export const SYSTEM_NAMESPACES = {
    /** System namespace */
    SYSTEM: 'system',
    
    /** Adapter namespace */
    ADAPTER: 'system.adapter',
    
    /** Host namespace */
    HOST: 'system.host',
    
    /** Group namespace */
    GROUP: 'system.group',
    
    /** User namespace */
    USER: 'system.user',
    
    /** Config namespace */
    CONFIG: 'system.config',
    
    /** Repository namespace */
    REPOSITORY: 'system.repositories'
} as const;

export const ADAPTER_MODES = {
    /** No execution */
    NONE: 'none',
    
    /** Daemon mode - always running */
    DAEMON: 'daemon',
    
    /** Scheduled execution */
    SCHEDULE: 'schedule',
    
    /** Single execution */
    ONCE: 'once',
    
    /** Extension mode */
    EXTENSION: 'extension'
} as const;

export const CONNECTION_TYPES = {
    /** No connection */
    NONE: 'none',
    
    /** Local connection */
    LOCAL: 'local',
    
    /** Cloud connection */
    CLOUD: 'cloud'
} as const;

export const DATA_SOURCES = {
    /** No data source */
    NONE: 'none',
    
    /** Polling data */
    POLL: 'poll',
    
    /** Push data */
    PUSH: 'push',
    
    /** Assumed data */
    ASSUMPTION: 'assumption'
} as const;
```

### Usage Examples

```javascript
// Check adapter mode
if (adapter.common.mode === adapter.constants.ADAPTER_MODES.DAEMON) {
    adapter.log.info('Running in daemon mode');
}

// Create system object ID
const adapterId = `${adapter.constants.SYSTEM_NAMESPACES.ADAPTER}.${adapter.name}.${adapter.instance}`;
```

## Error Codes

Standard error codes used throughout ioBroker.

```typescript
export const ERROR_CODES = {
    /** Permission denied */
    ERROR_PERMISSION: 1,
    
    /** Object not found */
    ERROR_NOT_FOUND: 2,
    
    /** Object already exists */
    ERROR_ALREADY_EXISTS: 3,
    
    /** Invalid arguments */
    ERROR_INVALID_ARGUMENTS: 4,
    
    /** Database error */
    ERROR_DB: 5,
    
    /** Network error */
    ERROR_NETWORK: 6,
    
    /** Timeout error */
    ERROR_TIMEOUT: 7,
    
    /** Unknown error */
    ERROR_UNKNOWN: 8
} as const;

export const ERROR_MESSAGES = {
    [ERROR_CODES.ERROR_PERMISSION]: 'Permission denied',
    [ERROR_CODES.ERROR_NOT_FOUND]: 'Object not found',
    [ERROR_CODES.ERROR_ALREADY_EXISTS]: 'Object already exists',
    [ERROR_CODES.ERROR_INVALID_ARGUMENTS]: 'Invalid arguments',
    [ERROR_CODES.ERROR_DB]: 'Database error',
    [ERROR_CODES.ERROR_NETWORK]: 'Network error',
    [ERROR_CODES.ERROR_TIMEOUT]: 'Timeout error',
    [ERROR_CODES.ERROR_UNKNOWN]: 'Unknown error'
} as const;
```

### Usage Examples

```javascript
// Handle errors with standard codes
adapter.getObject('nonexistent.object', (err, obj) => {
    if (err && err.code === adapter.constants.ERROR_CODES.ERROR_NOT_FOUND) {
        adapter.log.info('Object not found, creating new one');
        // Create object
    }
});

// Throw standardized errors
function validateInput(input) {
    if (!input) {
        const error = new Error(adapter.constants.ERROR_MESSAGES[adapter.constants.ERROR_CODES.ERROR_INVALID_ARGUMENTS]);
        error.code = adapter.constants.ERROR_CODES.ERROR_INVALID_ARGUMENTS;
        throw error;
    }
}
```

## State Data Types

Constants for state data types.

```typescript
export const STATE_TYPES = {
    /** Boolean value */
    BOOLEAN: 'boolean',
    
    /** Numeric value */
    NUMBER: 'number',
    
    /** String value */
    STRING: 'string',
    
    /** Object value */
    OBJECT: 'object',
    
    /** Mixed type */
    MIXED: 'mixed',
    
    /** Array value */
    ARRAY: 'array',
    
    /** File reference */
    FILE: 'file',
    
    /** JSON string */
    JSON: 'json'
} as const;
```

### Usage Examples

```javascript
// Create states with specific types
adapter.setObject('sensors.temperature', {
    type: 'state',
    common: {
        name: 'Temperature',
        type: adapter.constants.STATE_TYPES.NUMBER,
        role: 'value.temperature',
        read: true,
        write: false
    },
    native: {}
});

adapter.setObject('switches.light', {
    type: 'state',
    common: {
        name: 'Light Switch',
        type: adapter.constants.STATE_TYPES.BOOLEAN,
        role: 'switch.light',
        read: true,
        write: true
    },
    native: {}
});
```

## Access Constants

Access to all constants through adapter.constants.

```javascript
// Example of accessing constants from adapter
const adapter = new utils.Adapter('my-adapter');

// Quality codes
adapter.setState('test.state', {
    val: 42,
    ack: true,
    q: adapter.constants.QUALITY_CODES.GOOD
});

// Object types
adapter.setObject('test.device', {
    type: adapter.constants.OBJECT_TYPES.DEVICE,
    common: { name: 'Test Device' },
    native: {}
});

// State roles
adapter.setObject('test.temperature', {
    type: adapter.constants.OBJECT_TYPES.STATE,
    common: {
        name: 'Temperature',
        role: adapter.constants.STATE_ROLES.VALUE_TEMPERATURE,
        type: adapter.constants.STATE_TYPES.NUMBER,
        unit: '°C'
    },
    native: {}
});
```

<!-- 
Source metadata for automated updates:
- Primary source: ioBroker.js-controller/packages/adapter/src/lib/adapter/constants.ts
- Additional sources: Various TypeScript definition files
- Quality codes: Based on OPC UA standards and ioBroker conventions
- State roles: Comprehensive list from ioBroker documentation and source
- Last updated: 2024-12-22
-->