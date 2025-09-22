# Utility Functions

This document provides a reference for utility functions and helper methods available in the ioBroker.js-controller.

## Table of Contents

- [String Utilities](#string-utilities)
- [Date and Time Utilities](#date-and-time-utilities)
- [Number Formatting](#number-formatting)
- [Object Utilities](#object-utilities)
- [Validation Utilities](#validation-utilities)
- [Network Utilities](#network-utilities)
- [Security Utilities](#security-utilities)
- [File Path Utilities](#file-path-utilities)

## String Utilities

### formatValue()
Formats a value according to common object definition.

```javascript
formatValue(value, decimals?, unit?)
```

**Parameters:**
- `value` (any): Value to format
- `decimals` (number, optional): Number of decimal places
- `unit` (string, optional): Unit to append

**Returns:** string

**Example:**
```javascript
// Format temperature
const temp = adapter.formatValue(23.456, 1, '°C');
// Result: "23.5°C"

// Format percentage
const percent = adapter.formatValue(0.856, 2, '%');
// Result: "85.60%"

// Format without unit
const number = adapter.formatValue(1234.5678, 2);
// Result: "1234.57"
```

### validateId()
Validates an ioBroker object ID.

```javascript
validateId(id, isPattern?)
```

**Parameters:**
- `id` (string): ID to validate
- `isPattern` (boolean, optional): Allow pattern characters

**Returns:** boolean

**Example:**
```javascript
// Valid IDs
adapter.validateId('my-adapter.0.temperature'); // true
adapter.validateId('system.adapter.admin.0'); // true

// Invalid IDs
adapter.validateId('invalid..id'); // false
adapter.validateId('invalid id with spaces'); // false

// Pattern validation
adapter.validateId('my-adapter.*.temperature', true); // true
adapter.validateId('my-adapter.*.temperature', false); // false
```

### parseUrl()
Parses a URL string into components.

```javascript
parseUrl(url)
```

**Parameters:**
- `url` (string): URL to parse

**Returns:** object with protocol, hostname, port, pathname, etc.

**Example:**
```javascript
const urlInfo = adapter.parseUrl('http://192.168.1.100:8080/api/data');
// Result: {
//   protocol: 'http:',
//   hostname: '192.168.1.100', 
//   port: '8080',
//   pathname: '/api/data'
// }
```

### escapeRegExp()
Escapes special characters in a string for use in RegExp.

```javascript
escapeRegExp(string)
```

**Parameters:**
- `string` (string): String to escape

**Returns:** string

**Example:**
```javascript
const pattern = adapter.escapeRegExp('test.value[0]');
const regex = new RegExp(pattern);
// Safely matches literal string "test.value[0]"
```

### replaceVariables()
Replaces variables in a string template.

```javascript
replaceVariables(template, variables)
```

**Parameters:**
- `template` (string): Template string with variables
- `variables` (object): Variable values

**Returns:** string

**Example:**
```javascript
const template = 'Device %name% has temperature %temp%°C';
const result = adapter.replaceVariables(template, {
    name: 'Sensor1',
    temp: 23.5
});
// Result: "Device Sensor1 has temperature 23.5°C"
```

## Date and Time Utilities

### formatDate()
Formats a date according to specified format.

```javascript
formatDate(dateObj, format?)
```

**Parameters:**
- `dateObj` (Date | number): Date object or timestamp
- `format` (string, optional): Format string

**Returns:** string

**Formats:**
- `YYYY` - 4-digit year
- `MM` - 2-digit month
- `DD` - 2-digit day
- `hh` - 2-digit hour (24h)
- `mm` - 2-digit minute
- `ss` - 2-digit second
- `sss` - 3-digit milliseconds

**Example:**
```javascript
const date = new Date('2024-03-15T14:30:45.123Z');

// Default format
adapter.formatDate(date);
// Result: "2024-03-15 14:30:45"

// Custom format
adapter.formatDate(date, 'DD.MM.YYYY hh:mm');
// Result: "15.03.2024 14:30"

// With milliseconds
adapter.formatDate(date, 'YYYY-MM-DD hh:mm:ss.sss');
// Result: "2024-03-15 14:30:45.123"

// From timestamp
adapter.formatDate(Date.now(), 'DD/MM/YYYY');
```

### getTimeString()
Gets formatted time string from timestamp.

```javascript
getTimeString(timestamp)
```

**Parameters:**
- `timestamp` (number): Unix timestamp

**Returns:** string (HH:MM:SS format)

**Example:**
```javascript
const timeStr = adapter.getTimeString(Date.now());
// Result: "14:30:45"
```

### parseDuration()
Parses duration string to milliseconds.

```javascript
parseDuration(duration)
```

**Parameters:**
- `duration` (string): Duration string (e.g., "5m", "2h", "30s")

**Returns:** number (milliseconds)

**Example:**
```javascript
const ms1 = adapter.parseDuration('5m');    // 300000 (5 minutes)
const ms2 = adapter.parseDuration('2h');    // 7200000 (2 hours)
const ms3 = adapter.parseDuration('30s');   // 30000 (30 seconds)
const ms4 = adapter.parseDuration('1d');    // 86400000 (1 day)
```

## Number Formatting

### roundTo()
Rounds a number to specified decimal places.

```javascript
roundTo(value, decimals)
```

**Parameters:**
- `value` (number): Number to round
- `decimals` (number): Number of decimal places

**Returns:** number

**Example:**
```javascript
const rounded1 = adapter.roundTo(3.14159, 2); // 3.14
const rounded2 = adapter.roundTo(123.456, 0); // 123
const rounded3 = adapter.roundTo(0.123456, 4); // 0.1235
```

### formatBytes()
Formats byte count to human-readable string.

```javascript
formatBytes(bytes, decimals?)
```

**Parameters:**
- `bytes` (number): Number of bytes
- `decimals` (number, optional): Decimal places (default: 2)

**Returns:** string

**Example:**
```javascript
adapter.formatBytes(1024);       // "1.00 KB"
adapter.formatBytes(1048576);    // "1.00 MB"
adapter.formatBytes(1073741824); // "1.00 GB"
adapter.formatBytes(1536, 1);    // "1.5 KB"
```

### parseNumber()
Safely parses a string to number.

```javascript
parseNumber(value, defaultValue?)
```

**Parameters:**
- `value` (any): Value to parse
- `defaultValue` (number, optional): Default if parsing fails

**Returns:** number

**Example:**
```javascript
adapter.parseNumber('123.45');      // 123.45
adapter.parseNumber('invalid', 0);  // 0
adapter.parseNumber(null, -1);      // -1
adapter.parseNumber('42');          // 42
```

## Object Utilities

### deepClone()
Creates a deep copy of an object.

```javascript
deepClone(obj)
```

**Parameters:**
- `obj` (any): Object to clone

**Returns:** any (cloned object)

**Example:**
```javascript
const original = {
    name: 'sensor',
    config: { interval: 1000, enabled: true },
    data: [1, 2, 3]
};

const copy = adapter.deepClone(original);
copy.config.interval = 2000; // Original unchanged
```

### extend()
Recursively extends an object with properties from other objects.

```javascript
extend(target, ...sources)
```

**Parameters:**
- `target` (object): Target object
- `sources` (object[]): Source objects

**Returns:** object (extended target)

**Example:**
```javascript
const defaults = {
    timeout: 5000,
    retries: 3,
    config: { debug: false }
};

const userConfig = {
    timeout: 10000,
    config: { debug: true, verbose: true }
};

const final = adapter.extend({}, defaults, userConfig);
// Result: {
//   timeout: 10000,
//   retries: 3,
//   config: { debug: true, verbose: true }
// }
```

### getObjectPath()
Safely gets a nested property value.

```javascript
getObjectPath(obj, path, defaultValue?)
```

**Parameters:**
- `obj` (object): Source object
- `path` (string): Property path (dot notation)
- `defaultValue` (any, optional): Default value if path not found

**Returns:** any

**Example:**
```javascript
const obj = {
    device: {
        sensor: {
            temperature: 23.5
        }
    }
};

const temp = adapter.getObjectPath(obj, 'device.sensor.temperature');
// Result: 23.5

const missing = adapter.getObjectPath(obj, 'device.missing.value', 0);
// Result: 0
```

### setObjectPath()
Sets a nested property value.

```javascript
setObjectPath(obj, path, value)
```

**Parameters:**
- `obj` (object): Target object
- `path` (string): Property path (dot notation)
- `value` (any): Value to set

**Returns:** void

**Example:**
```javascript
const obj = {};
adapter.setObjectPath(obj, 'device.sensor.temperature', 23.5);
// Result: { device: { sensor: { temperature: 23.5 } } }
```

### filterObject()
Filters object properties by predicate function.

```javascript
filterObject(obj, predicate)
```

**Parameters:**
- `obj` (object): Source object
- `predicate` (function): Filter function (key, value) => boolean

**Returns:** object

**Example:**
```javascript
const obj = { a: 1, b: 2, c: 3, d: 4 };
const filtered = adapter.filterObject(obj, (key, value) => value > 2);
// Result: { c: 3, d: 4 }
```

## Validation Utilities

### isValidEmail()
Validates an email address.

```javascript
isValidEmail(email)
```

**Parameters:**
- `email` (string): Email to validate

**Returns:** boolean

**Example:**
```javascript
adapter.isValidEmail('user@example.com');     // true
adapter.isValidEmail('invalid.email');       // false
adapter.isValidEmail('test@domain.co.uk');   // true
```

### isValidIPAddress()
Validates an IP address (IPv4 or IPv6).

```javascript
isValidIPAddress(ip, version?)
```

**Parameters:**
- `ip` (string): IP address to validate
- `version` (number, optional): IP version (4 or 6)

**Returns:** boolean

**Example:**
```javascript
adapter.isValidIPAddress('192.168.1.1');          // true
adapter.isValidIPAddress('192.168.1.256');        // false
adapter.isValidIPAddress('::1');                  // true
adapter.isValidIPAddress('192.168.1.1', 4);       // true
adapter.isValidIPAddress('::1', 4);               // false
```

### isValidPort()
Validates a network port number.

```javascript
isValidPort(port)
```

**Parameters:**
- `port` (number | string): Port number to validate

**Returns:** boolean

**Example:**
```javascript
adapter.isValidPort(80);      // true
adapter.isValidPort(8080);    // true
adapter.isValidPort(65536);   // false
adapter.isValidPort(-1);      // false
```

### isValidUrl()
Validates a URL.

```javascript
isValidUrl(url)
```

**Parameters:**
- `url` (string): URL to validate

**Returns:** boolean

**Example:**
```javascript
adapter.isValidUrl('http://example.com');        // true
adapter.isValidUrl('https://example.com:8080');  // true
adapter.isValidUrl('ftp://files.example.com');   // true
adapter.isValidUrl('invalid-url');               // false
```

## Network Utilities

### getPort()
Finds an available port number.

```javascript
getPort(startPort, callback)
```

**Parameters:**
- `startPort` (number): Starting port to check
- `callback` (function): Callback (err, port) => void

**Example:**
```javascript
adapter.getPort(8080, (err, port) => {
    if (!err) {
        adapter.log.info(`Using port: ${port}`);
        // Start server on available port
    }
});
```

### getPortAsync()
Async version of getPort().

```javascript
await getPortAsync(startPort)
```

**Returns:** Promise\<number>

**Example:**
```javascript
try {
    const port = await adapter.getPortAsync(8080);
    adapter.log.info(`Server will run on port: ${port}`);
} catch (error) {
    adapter.log.error(`Failed to find available port: ${error.message}`);
}
```

### ping()
Pings a host to check connectivity.

```javascript
ping(host, callback)
```

**Parameters:**
- `host` (string): Hostname or IP address
- `callback` (function): Callback (err, result) => void

**Example:**
```javascript
adapter.ping('8.8.8.8', (err, result) => {
    if (!err && result) {
        adapter.log.info(`Ping successful: ${result.time}ms`);
    } else {
        adapter.log.warn('Ping failed');
    }
});
```

### resolveHostname()
Resolves hostname to IP address.

```javascript
resolveHostname(hostname, callback)
```

**Parameters:**
- `hostname` (string): Hostname to resolve
- `callback` (function): Callback (err, ip) => void

**Example:**
```javascript
adapter.resolveHostname('google.com', (err, ip) => {
    if (!err) {
        adapter.log.info(`Resolved to: ${ip}`);
    }
});
```

## Security Utilities

### encrypt()
Encrypts a string value.

```javascript
encrypt(text)
```

**Parameters:**
- `text` (string): Text to encrypt

**Returns:** string (encrypted text)

**Example:**
```javascript
const encrypted = adapter.encrypt('my-secret-password');
// Store encrypted value
adapter.setState('config.encryptedPassword', encrypted, true);
```

### decrypt()
Decrypts an encrypted string.

```javascript
decrypt(encryptedText)
```

**Parameters:**
- `encryptedText` (string): Encrypted text

**Returns:** string (decrypted text)

**Example:**
```javascript
const state = await adapter.getStateAsync('config.encryptedPassword');
const password = adapter.decrypt(state.val);
// Use decrypted password
```

### generateRandomString()
Generates a random string.

```javascript
generateRandomString(length, chars?)
```

**Parameters:**
- `length` (number): Length of string
- `chars` (string, optional): Character set to use

**Returns:** string

**Example:**
```javascript
// Default alphanumeric
const token1 = adapter.generateRandomString(16);
// Result: "aB3xY9mN4kL7sE2q"

// Custom character set
const token2 = adapter.generateRandomString(8, '0123456789');
// Result: "12738456"

// Hex string
const token3 = adapter.generateRandomString(12, '0123456789ABCDEF');
// Result: "A3B7C2E9F1D4"
```

### hashString()
Creates a hash of a string.

```javascript
hashString(text, algorithm?)
```

**Parameters:**
- `text` (string): Text to hash
- `algorithm` (string, optional): Hash algorithm (default: 'sha256')

**Returns:** string (hash)

**Example:**
```javascript
const hash1 = adapter.hashString('password');
const hash2 = adapter.hashString('password', 'md5');
const hash3 = adapter.hashString('password', 'sha1');
```

## File Path Utilities

### joinPath()
Joins path segments safely.

```javascript
joinPath(...segments)
```

**Parameters:**
- `segments` (string[]): Path segments

**Returns:** string

**Example:**
```javascript
const path1 = adapter.joinPath('folder', 'subfolder', 'file.txt');
// Result: "folder/subfolder/file.txt" (or "folder\subfolder\file.txt" on Windows)

const path2 = adapter.joinPath('/root', '../parent', 'file.txt');
// Result: "/parent/file.txt"
```

### normalizePath()
Normalizes a file path.

```javascript
normalizePath(path)
```

**Parameters:**
- `path` (string): Path to normalize

**Returns:** string

**Example:**
```javascript
const normalized = adapter.normalizePath('folder/../subfolder/./file.txt');
// Result: "subfolder/file.txt"
```

### getBasename()
Gets the base name of a file path.

```javascript
getBasename(path, ext?)
```

**Parameters:**
- `path` (string): File path
- `ext` (string, optional): Extension to remove

**Returns:** string

**Example:**
```javascript
adapter.getBasename('/path/to/file.txt');        // "file.txt"
adapter.getBasename('/path/to/file.txt', '.txt'); // "file"
adapter.getBasename('C:\\folder\\image.jpg');     // "image.jpg"
```

### getDirname()
Gets the directory name of a file path.

```javascript
getDirname(path)
```

**Parameters:**
- `path` (string): File path

**Returns:** string

**Example:**
```javascript
adapter.getDirname('/path/to/file.txt');     // "/path/to"
adapter.getDirname('C:\\folder\\file.txt');  // "C:\\folder"
```

### getExtension()
Gets the file extension.

```javascript
getExtension(path)
```

**Parameters:**
- `path` (string): File path

**Returns:** string

**Example:**
```javascript
adapter.getExtension('file.txt');      // ".txt"
adapter.getExtension('image.jpeg');    // ".jpeg"
adapter.getExtension('config.json');   // ".json"
adapter.getExtension('noextension');   // ""
```

## Usage Examples

### Complete Utility Usage
```javascript
class MyAdapter {
    async processData() {
        // Format values
        const temp = this.adapter.formatValue(23.456, 1, '°C');
        this.adapter.log.info(`Temperature: ${temp}`);
        
        // Date formatting
        const timestamp = this.adapter.formatDate(new Date(), 'DD.MM.YYYY hh:mm');
        this.adapter.log.info(`Current time: ${timestamp}`);
        
        // Validate configuration
        if (!this.adapter.isValidIPAddress(this.adapter.config.hostname)) {
            this.adapter.log.error('Invalid IP address in configuration');
            return;
        }
        
        // Deep clone configuration
        const configCopy = this.adapter.deepClone(this.adapter.config);
        
        // Generate secure token
        const token = this.adapter.generateRandomString(32);
        
        // Find available port
        const port = await this.adapter.getPortAsync(8080);
        this.adapter.log.info(`Using port: ${port}`);
        
        // Encrypt sensitive data
        const encrypted = this.adapter.encrypt(this.adapter.config.password);
        await this.adapter.setStateAsync('internal.encryptedPassword', encrypted, true);
    }
}
```

<!-- 
Source metadata for automated updates:
- Primary source: ioBroker.js-controller/packages/adapter/src/lib/adapter/utils.ts
- Additional utilities from: packages/adapter/src/lib/tools.ts  
- Helper functions extracted from adapter.ts main class
- Date formatting based on ioBroker conventions
- Security functions from controller crypto utilities
- Last updated: 2024-12-22
-->