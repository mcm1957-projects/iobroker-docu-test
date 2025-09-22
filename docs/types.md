# TypeScript Types & Interfaces

This document provides a comprehensive reference for all public TypeScript types, interfaces, and enumerations available in the ioBroker.js-controller.

## Table of Contents

- [Core Types](#core-types)
- [State Types](#state-types)
- [Object Types](#object-types)
- [Configuration Types](#configuration-types)
- [Message Types](#message-types)
- [File Types](#file-types)
- [Permission Types](#permission-types)
- [Utility Types](#utility-types)

## Core Types

### AdapterConfig
Configuration interface for adapter instances.

```typescript
interface AdapterConfig {
    /** Adapter name */
    name: string;
    /** Instance number */
    instance: number;
    /** Custom configuration from io-package.json native section */
    [key: string]: any;
}
```

### AdapterOptions
Options for creating an adapter instance.

```typescript
interface AdapterOptions {
    /** Adapter name */
    name: string;
    /** Instance number (default: 0) */
    instance?: number;
    /** Use objects cache */
    objects?: boolean;
    /** Use states cache */
    states?: boolean;
    /** File synchronization */
    fileChange?: boolean;
    /** Custom log level */
    logLevel?: LogLevel;
    /** Compact mode */
    compact?: boolean;
    /** Config object */
    config?: Record<string, any>;
    /** Ready callback */
    ready?: () => void;
    /** Object change callback */
    objectChange?: (id: string, obj: ioBrokerObject | null) => void;
    /** State change callback */
    stateChange?: (id: string, state: State | null) => void;
    /** Message callback */
    message?: (obj: Message) => void;
    /** Unload callback */
    unload?: (callback: () => void) => void;
}
```

### Adapter
Main adapter class interface.

```typescript
interface Adapter extends EventEmitter {
    /** Adapter name */
    readonly name: string;
    /** Host name */
    readonly host: string;
    /** Instance number */
    readonly instance: number;
    /** Adapter namespace */
    readonly namespace: string;
    /** Configuration */
    readonly config: AdapterConfig;
    /** Logging interface */
    readonly log: Logger;
    /** Version information */
    readonly version: string;
    /** Connected state */
    readonly connected: boolean;
}
```

## State Types

### State
Represents a state value in ioBroker.

```typescript
interface State {
    /** State value */
    val: any;
    /** Acknowledged flag */
    ack: boolean;
    /** Timestamp */
    ts: number;
    /** Last change timestamp */
    lc: number;
    /** Quality code */
    q: number;
    /** User who changed the state */
    user?: string;
    /** Change comment */
    c?: string;
    /** Expire time in seconds */
    expire?: number;
    /** Source of change */
    from: string;
}
```

### StateValue
Union type for state values.

```typescript
type StateValue = string | number | boolean | null;
```

### SettableState
State object that can be set.

```typescript
interface SettableState {
    val: StateValue;
    ack?: boolean;
    ts?: number;
    q?: number;
    c?: string;
    expire?: number;
}
```

### QualityCodes
State quality codes enumeration.

```typescript
enum QualityCodes {
    /** Good */
    GOOD = 0x00,
    /** General problem */
    GENERAL_PROBLEM = 0x01,
    /** No connection problem */
    NO_CONNECTION_PROBLEM = 0x02,
    /** Communication error */
    COMM_ERROR = 0x10,
    /** No connection to device */
    DEVICE_NOT_CONNECTED = 0x11,
    /** Device reports error */
    DEVICE_ERROR = 0x12,
    /** Device not responding */
    DEVICE_NOT_RESPONDING = 0x13,
    /** Out of service */
    OUT_OF_SERVICE = 0x14,
    /** Sensor error */
    SENSOR_ERROR = 0x20,
    /** Substitute value */
    SUBSTITUTE_VALUE = 0x40,
    /** Initial value */
    INITIAL_VALUE = 0x41
}
```

## Object Types

### ioBrokerObject
Base interface for all ioBroker objects.

```typescript
interface ioBrokerObject {
    /** Object ID */
    _id: string;
    /** Object type */
    type: ObjectType;
    /** Common properties */
    common: ObjectCommon;
    /** Native adapter-specific properties */
    native?: Record<string, any>;
    /** Access control list */
    acl?: AccessControlList;
    /** Timestamp */
    ts?: number;
    /** User who created/modified */
    user?: string;
    /** Source adapter */
    from?: string;
}
```

### ObjectType
Enumeration of object types.

```typescript
type ObjectType = 
    | 'adapter'
    | 'channel'
    | 'device'
    | 'enum'
    | 'folder'
    | 'group'
    | 'host'
    | 'instance'
    | 'meta'
    | 'script'
    | 'state'
    | 'user';
```

### ObjectCommon
Common properties for all objects.

```typescript
interface ObjectCommon {
    /** Object name */
    name: string | Record<string, string>;
    /** Role/function */
    role?: string;
    /** Description */
    desc?: string | Record<string, string>;
    /** Custom properties */
    custom?: Record<string, any>;
    /** Color */
    color?: string;
    /** Icon */
    icon?: string;
    /** Enabled flag */
    enabled?: boolean;
    /** Expert mode */
    expert?: boolean;
}
```

### StateCommon
Common properties specific to state objects.

```typescript
interface StateCommon extends ObjectCommon {
    /** Data type */
    type: 'boolean' | 'number' | 'string' | 'object' | 'mixed' | 'array' | 'file' | 'json';
    /** Readable flag */
    read: boolean;
    /** Writable flag */
    write: boolean;
    /** Minimum value */
    min?: number;
    /** Maximum value */
    max?: number;
    /** Step value */
    step?: number;
    /** Unit */
    unit?: string;
    /** Default value */
    def?: any;
    /** States for select */
    states?: Record<string | number, string> | string[];
    /** Working hours */
    workingID?: string;
    /** Alias */
    alias?: {
        id: string;
        read?: string;
        write?: string;
    };
    /** History configuration */
    history?: {
        enabled: boolean;
        retention?: number;
        debounce?: number;
        maxLength?: number;
    };
}
```

### ChannelCommon
Common properties for channel objects.

```typescript
interface ChannelCommon extends ObjectCommon {
    /** Channel description */
    desc?: string | Record<string, string>;
}
```

### DeviceCommon
Common properties for device objects.

```typescript
interface DeviceCommon extends ObjectCommon {
    /** Device status ID */
    statusStates?: {
        onlineId?: string;
        errorId?: string;
    };
}
```

### EnumCommon
Common properties for enumeration objects.

```typescript
interface EnumCommon extends ObjectCommon {
    /** Enumeration members */
    members: string[];
}
```

## Configuration Types

### InstanceObject
Represents an adapter instance configuration.

```typescript
interface InstanceObject extends ioBrokerObject {
    common: InstanceCommon;
    native: Record<string, any>;
}
```

### InstanceCommon
Common properties for instance objects.

```typescript
interface InstanceCommon extends ObjectCommon {
    /** Adapter version */
    version: string;
    /** Platform */
    platform: string;
    /** Execution mode */
    mode: 'daemon' | 'schedule' | 'once' | 'none';
    /** Schedule for schedule mode */
    schedule?: string;
    /** Enabled flag */
    enabled: boolean;
    /** Host restriction */
    host: string;
    /** Log level */
    loglevel: LogLevel;
    /** Compact mode */
    compact?: boolean;
    /** Restart schedule */
    restartSchedule?: string;
    /** Memory limit */
    memoryLimitMB?: number;
}
```

### AdapterObject
Represents an adapter definition.

```typescript
interface AdapterObject extends ioBrokerObject {
    common: AdapterCommon;
}
```

### AdapterCommon
Common properties for adapter objects.

```typescript
interface AdapterCommon extends ObjectCommon {
    /** Adapter version */
    version: string;
    /** Longer name */
    title?: string;
    /** Multi-language title */
    titleLang?: Record<string, string>;
    /** Description */
    desc: Record<string, string>;
    /** Authors */
    authors: string[];
    /** License */
    license?: string;
    /** Platform */
    platform: string;
    /** Dependencies */
    dependencies?: Dependency[];
    /** Global dependencies */
    globalDependencies?: Dependency[];
    /** Admin UI configuration */
    adminUI?: {
        config: 'json' | 'html' | 'materialize' | 'none';
        tab?: 'json' | 'html' | 'materialize';
        custom?: 'json';
    };
    /** Supported messages */
    supportedMessages?: {
        custom?: boolean;
        getHistory?: boolean;
        stopInstance?: boolean | number;
        deviceManager?: boolean;
        notifications?: boolean;
    };
    /** Connection type */
    connectionType?: 'none' | 'local' | 'cloud';
    /** Data source */
    dataSource?: 'none' | 'poll' | 'push' | 'assumption';
    /** Keywords */
    keywords?: string[];
    /** Readme URL */
    readme?: string;
    /** Type classification */
    type?: AdapterType;
    /** Tier level */
    tier?: 1 | 2 | 3;
    /** License information */
    licenseInformation?: {
        type: 'free' | 'paid' | 'commercial' | 'limited';
        license?: string;
        link?: string;
    };
}
```

### AdapterType
Adapter type classification.

```typescript
type AdapterType = 
    | 'alarm'
    | 'climate-control'
    | 'communication'
    | 'date-and-time'
    | 'energy'
    | 'garden'
    | 'general'
    | 'geoposition'
    | 'hardware'
    | 'health'
    | 'household'
    | 'infrastructure'
    | 'iot-systems'
    | 'lighting'
    | 'logic'
    | 'messaging'
    | 'metering'
    | 'misc-data'
    | 'multimedia'
    | 'network'
    | 'protocols'
    | 'storage'
    | 'utility'
    | 'vehicle'
    | 'visualization'
    | 'visualization-icons'
    | 'visualization-widgets'
    | 'weather';
```

### Dependency
Dependency specification.

```typescript
interface Dependency {
    [adapterName: string]: string; // semver range
}
```

## Message Types

### Message
Represents a message between adapters.

```typescript
interface Message {
    /** Message command */
    command: string;
    /** Message data */
    message: any;
    /** Sender */
    from: string;
    /** Callback ID */
    callback?: {
        message: any;
        id: number;
        ack: boolean;
        time: number;
    };
}
```

### SendToOptions
Options for sendTo method.

```typescript
interface SendToOptions {
    /** Target instance */
    instance: string;
    /** Command */
    command: string;
    /** Message data */
    message?: any;
    /** Timeout in milliseconds */
    timeout?: number;
}
```

## File Types

### FileOptions
Options for file operations.

```typescript
interface FileOptions {
    /** MIME type */
    mimeType?: string;
    /** Encoding */
    encoding?: BufferEncoding;
}
```

### FileObject
Represents a file in the adapter file system.

```typescript
interface FileObject {
    /** File name */
    file: string;
    /** File stats */
    stats: {
        size: number;
        atime: Date;
        mtime: Date;
        ctime: Date;
        birthtime: Date;
    };
    /** Is directory */
    isDir: boolean;
    /** Access control */
    acl: AccessControlList;
    /** Modified by */
    modifiedAt: number;
    /** Created by */
    createdAt: number;
}
```

## Permission Types

### AccessControlList
Access control list for objects and files.

```typescript
interface AccessControlList {
    /** Owner */
    owner: string;
    /** Owner group */
    ownerGroup: string;
    /** Permissions */
    permissions: number;
    /** Object permissions */
    object?: {
        read: boolean;
        write: boolean;
        delete: boolean;
        list: boolean;
    };
    /** State permissions */
    state?: {
        read: boolean;
        write: boolean;
        create: boolean;
        delete: boolean;
        list: boolean;
    };
    /** File permissions */
    file?: {
        read: boolean;
        write: boolean;
        create: boolean;
        delete: boolean;
        list: boolean;
    };
}
```

### UserPermissions
User permission calculation result.

```typescript
interface UserPermissions {
    /** User ID */
    user: string;
    /** Groups */
    groups: string[];
    /** Permissions */
    permissions: {
        object: {
            read: boolean;
            write: boolean;
            delete: boolean;
            list: boolean;
        };
        state: {
            read: boolean;
            write: boolean;
            create: boolean;
            delete: boolean;
            list: boolean;
        };
        file: {
            read: boolean;
            write: boolean;
            create: boolean;
            delete: boolean;
            list: boolean;
        };
        users: {
            read: boolean;
            write: boolean;
            create: boolean;
            delete: boolean;
            list: boolean;
        };
    };
}
```

## Utility Types

### LogLevel
Logging level enumeration.

```typescript
type LogLevel = 'silly' | 'debug' | 'info' | 'warn' | 'error';
```

### Logger
Logging interface.

```typescript
interface Logger {
    silly(message: string): void;
    debug(message: string): void;
    info(message: string): void;
    warn(message: string): void;
    error(message: string): void;
    level: LogLevel;
}
```

### GetHistoryOptions
Options for retrieving historical data.

```typescript
interface GetHistoryOptions {
    /** Start time */
    start?: number;
    /** End time */
    end?: number;
    /** Number of values */
    count?: number;
    /** Include from/to values */
    from?: boolean;
    /** Ignore null values */
    ign?: boolean;
    /** Aggregate function */
    aggregate?: 'minmax' | 'min' | 'max' | 'average' | 'total' | 'count' | 'none';
    /** Step in milliseconds for aggregation */
    step?: number;
    /** Limit number of results */
    limit?: number;
    /** Add ID to results */
    addId?: boolean;
    /** Return as object */
    returnNewestEntries?: boolean;
}
```

### GetHistoryResult
Result from getHistory operation.

```typescript
interface GetHistoryResult {
    /** Values */
    values: HistoryValue[];
    /** Step used */
    step?: number;
    /** Session ID */
    sessionId?: string;
}
```

### HistoryValue
Single history value.

```typescript
interface HistoryValue {
    /** Timestamp */
    ts: number;
    /** Value */
    val: any;
    /** Acknowledged */
    ack: boolean;
    /** Quality */
    q: number;
    /** Last change */
    lc: number;
    /** User */
    user?: string;
    /** Comment */
    c?: string;
    /** From */
    from: string;
}
```

### Timer
Timer handle type.

```typescript
type Timer = NodeJS.Timeout;
```

### Interval
Interval handle type.

```typescript
type Interval = NodeJS.Timeout;
```

### Callback
Generic callback function type.

```typescript
type Callback<T = any> = (error: Error | null, result?: T) => void;
```

### ObjectCallback
Callback for object operations.

```typescript
type ObjectCallback = Callback<ioBrokerObject>;
```

### StateCallback
Callback for state operations.

```typescript
type StateCallback = Callback<State>;
```

### GetObjectsCallback
Callback for getting multiple objects.

```typescript
type GetObjectsCallback = Callback<Record<string, ioBrokerObject>>;
```

### GetStatesCallback
Callback for getting multiple states.

```typescript
type GetStatesCallback = Callback<Record<string, State>>;
```

## Generic Types

### StringOrTranslated
Type for strings that can be translated.

```typescript
type StringOrTranslated = string | Record<string, string>;
```

### ObjectPattern
Pattern for object queries.

```typescript
type ObjectPattern = string | RegExp;
```

### StatePattern
Pattern for state queries.

```typescript
type StatePattern = string | RegExp;
```

## Event Types

### AdapterEvents
Event map for adapter event emitter.

```typescript
interface AdapterEvents {
    'ready': () => void;
    'stateChange': (id: string, state: State | null) => void;
    'objectChange': (id: string, obj: ioBrokerObject | null) => void;
    'message': (obj: Message) => void;
    'unload': (callback: () => void) => void;
    'exit': (exitCode: number) => void;
    'log': (data: LogData) => void;
}
```

### LogData
Log data structure.

```typescript
interface LogData {
    severity: LogLevel;
    ts: number;
    message: string;
    from: string;
}
```

## Type Guards

### isState
Type guard for State objects.

```typescript
function isState(obj: any): obj is State {
    return obj && typeof obj.val !== 'undefined' && typeof obj.ack === 'boolean';
}
```

### isObject
Type guard for ioBroker objects.

```typescript
function isObject(obj: any): obj is ioBrokerObject {
    return obj && typeof obj._id === 'string' && typeof obj.type === 'string';
}
```

## Examples

### Creating a State Object
```typescript
const tempState: StateCommon = {
    name: 'Temperature',
    type: 'number',
    role: 'value.temperature',
    unit: '°C',
    read: true,
    write: false,
    min: -50,
    max: 100
};
```

### Message Handling
```typescript
adapter.on('message', (obj: Message) => {
    if (obj.command === 'getData') {
        const response = {
            success: true,
            data: getSomeData()
        };
        adapter.sendTo(obj.from, obj.command, response, obj.callback);
    }
});
```

### Type-Safe Configuration
```typescript
interface MyAdapterConfig {
    hostname: string;
    port: number;
    username: string;
    password: string;
    enabled: boolean;
}

const config: MyAdapterConfig = adapter.config as MyAdapterConfig;
```

<!-- 
Source metadata for automated updates:
- Primary sources: 
  - ioBroker.js-controller/packages/adapter/src/lib/_Types.ts
  - ioBroker.js-controller/packages/types-dev/index.d.ts
  - ioBroker.js-controller/packages/adapter/src/lib/adapter/adapter.ts
- Type definitions extracted from official TypeScript declarations
- Last updated: 2024-12-22
- Covers all public interfaces and types used by adapters
-->