# File Operations

This document covers file system operations and file management in ioBroker adapters.

## Table of Contents

- [File System Overview](#file-system-overview)
- [Reading Files](#reading-files)
- [Writing Files](#writing-files)
- [File Management](#file-management)
- [Directory Operations](#directory-operations)
- [File Permissions](#file-permissions)
- [Best Practices](#best-practices)

## File System Overview

ioBroker adapters can access files in their dedicated adapter directories within the ioBroker data structure. Each adapter has its own namespace for file operations.

### File System Structure

```
iobroker-data/files/
├── system.adapter.admin.0/
├── system.adapter.web.0/
├── my-adapter.admin/          # Configuration files
│   ├── config.json
│   ├── certificates/
│   └── uploads/
└── my-adapter.0/              # Instance files
    ├── data/
    ├── logs/
    └── temp/
```

### Adapter File Namespace

Files are accessed relative to the adapter's namespace:
- `adapter-name.admin` - Configuration and admin files
- `adapter-name.instance` - Instance-specific files

## Reading Files

### Basic File Reading

```javascript
// Read a configuration file
adapter.readFile('my-adapter.admin', 'config.json', (err, data) => {
    if (err) {
        adapter.log.error(`Failed to read config: ${err.message}`);
        return;
    }
    
    if (data) {
        try {
            const config = JSON.parse(data.toString());
            adapter.log.info('Configuration loaded successfully');
        } catch (parseErr) {
            adapter.log.error(`Invalid JSON in config file: ${parseErr.message}`);
        }
    }
});

// Read with encoding specified
adapter.readFile('my-adapter.0', 'data.txt', { encoding: 'utf8' }, (err, data) => {
    if (!err && data) {
        adapter.log.info(`File content: ${data}`);
    }
});
```

### Async File Reading

```javascript
async function loadConfiguration() {
    try {
        const data = await adapter.readFileAsync('my-adapter.admin', 'config.json');
        
        if (data) {
            const config = JSON.parse(data.toString());
            return config;
        }
    } catch (error) {
        adapter.log.error(`Failed to load configuration: ${error.message}`);
        return null;
    }
}

// Binary file reading
async function loadImage() {
    try {
        const imageData = await adapter.readFileAsync('my-adapter.admin', 'images/logo.png');
        
        if (imageData) {
            adapter.log.info(`Loaded image: ${imageData.length} bytes`);
            return imageData;
        }
    } catch (error) {
        adapter.log.error(`Failed to load image: ${error.message}`);
        return null;
    }
}
```

### Reading Multiple Files

```javascript
async function loadAllConfigs() {
    try {
        // Read directory contents first
        const files = await adapter.readDirAsync('my-adapter.admin', 'configs/');
        
        const configs = {};
        
        for (const file of files) {
            if (file.file.endsWith('.json') && !file.isDir) {
                const data = await adapter.readFileAsync('my-adapter.admin', `configs/${file.file}`);
                
                if (data) {
                    try {
                        configs[file.file] = JSON.parse(data.toString());
                    } catch (parseErr) {
                        adapter.log.warn(`Invalid JSON in ${file.file}: ${parseErr.message}`);
                    }
                }
            }
        }
        
        return configs;
    } catch (error) {
        adapter.log.error(`Failed to load configs: ${error.message}`);
        return {};
    }
}
```

## Writing Files

### Basic File Writing

```javascript
// Write configuration
const config = {
    setting1: 'value1',
    setting2: 42,
    lastUpdate: new Date().toISOString()
};

adapter.writeFile('my-adapter.admin', 'config.json', JSON.stringify(config, null, 2), (err) => {
    if (err) {
        adapter.log.error(`Failed to write config: ${err.message}`);
    } else {
        adapter.log.info('Configuration saved successfully');
    }
});

// Write text file
const logEntry = `${new Date().toISOString()}: System started\n`;
adapter.writeFile('my-adapter.0', 'logs/system.log', logEntry, { encoding: 'utf8' }, (err) => {
    if (!err) {
        adapter.log.debug('Log entry written');
    }
});
```

### Async File Writing

```javascript
async function saveConfiguration(config) {
    try {
        const configData = JSON.stringify(config, null, 2);
        await adapter.writeFileAsync('my-adapter.admin', 'config.json', configData);
        adapter.log.info('Configuration saved');
        return true;
    } catch (error) {
        adapter.log.error(`Failed to save configuration: ${error.message}`);
        return false;
    }
}

// Binary file writing
async function saveImage(imageBuffer, filename) {
    try {
        await adapter.writeFileAsync('my-adapter.admin', `images/${filename}`, imageBuffer);
        adapter.log.info(`Image saved: ${filename}`);
        return true;
    } catch (error) {
        adapter.log.error(`Failed to save image: ${error.message}`);
        return false;
    }
}
```

### Append to Files

```javascript
class FileLogger {
    constructor(adapter, filename) {
        this.adapter = adapter;
        this.filename = filename;
    }

    async appendLog(message) {
        const timestamp = new Date().toISOString();
        const logEntry = `[${timestamp}] ${message}\n`;
        
        try {
            // Read existing content
            let existingContent = '';
            try {
                const data = await this.adapter.readFileAsync('my-adapter.0', this.filename);
                if (data) {
                    existingContent = data.toString();
                }
            } catch (readErr) {
                // File doesn't exist yet, that's okay
            }
            
            // Append new entry
            const newContent = existingContent + logEntry;
            await this.adapter.writeFileAsync('my-adapter.0', this.filename, newContent);
            
        } catch (error) {
            this.adapter.log.error(`Failed to append to log: ${error.message}`);
        }
    }

    async rotateLog(maxLines = 1000) {
        try {
            const data = await this.adapter.readFileAsync('my-adapter.0', this.filename);
            if (!data) return;

            const lines = data.toString().split('\n');
            if (lines.length > maxLines) {
                // Keep only the last maxLines
                const rotatedContent = lines.slice(-maxLines).join('\n');
                await this.adapter.writeFileAsync('my-adapter.0', this.filename, rotatedContent);
                
                this.adapter.log.info(`Log rotated, kept ${maxLines} latest entries`);
            }
        } catch (error) {
            this.adapter.log.error(`Failed to rotate log: ${error.message}`);
        }
    }
}

// Usage
const logger = new FileLogger(adapter, 'logs/adapter.log');
await logger.appendLog('Adapter started successfully');
```

## File Management

### File Operations

```javascript
// Delete a file
adapter.delFile('my-adapter.0', 'temp/old-data.json', (err) => {
    if (err) {
        adapter.log.error(`Failed to delete file: ${err.message}`);
    } else {
        adapter.log.info('File deleted successfully');
    }
});

// Rename/move a file
adapter.renameFile('my-adapter.0', 'data/temp.json', 'data/config.json', (err) => {
    if (err) {
        adapter.log.error(`Failed to rename file: ${err.message}`);
    } else {
        adapter.log.info('File renamed successfully');
    }
});

// Check if file exists
adapter.readFile('my-adapter.admin', 'config.json', (err, data) => {
    if (err) {
        if (err.code === 'ENOENT') {
            adapter.log.info('Config file does not exist, creating default');
            createDefaultConfig();
        } else {
            adapter.log.error(`Error checking config file: ${err.message}`);
        }
    } else {
        adapter.log.info('Config file exists');
    }
});
```

### Async File Management

```javascript
class FileManager {
    constructor(adapter) {
        this.adapter = adapter;
    }

    async fileExists(namespace, filename) {
        try {
            await this.adapter.readFileAsync(namespace, filename);
            return true;
        } catch (error) {
            return error.code !== 'ENOENT';
        }
    }

    async copyFile(sourceNamespace, sourcePath, targetNamespace, targetPath) {
        try {
            const data = await this.adapter.readFileAsync(sourceNamespace, sourcePath);
            if (data) {
                await this.adapter.writeFileAsync(targetNamespace, targetPath, data);
                return true;
            }
            return false;
        } catch (error) {
            this.adapter.log.error(`Failed to copy file: ${error.message}`);
            return false;
        }
    }

    async moveFile(sourceNamespace, sourcePath, targetNamespace, targetPath) {
        try {
            const copied = await this.copyFile(sourceNamespace, sourcePath, targetNamespace, targetPath);
            if (copied) {
                await this.adapter.delFileAsync(sourceNamespace, sourcePath);
                return true;
            }
            return false;
        } catch (error) {
            this.adapter.log.error(`Failed to move file: ${error.message}`);
            return false;
        }
    }

    async getFileInfo(namespace, filename) {
        try {
            const files = await this.adapter.readDirAsync(namespace, '');
            const file = files.find(f => f.file === filename);
            
            if (file) {
                return {
                    name: file.file,
                    size: file.stats.size,
                    modified: new Date(file.stats.mtime),
                    created: new Date(file.stats.ctime),
                    isDirectory: file.isDir
                };
            }
            return null;
        } catch (error) {
            this.adapter.log.error(`Failed to get file info: ${error.message}`);
            return null;
        }
    }
}

// Usage
const fileManager = new FileManager(adapter);

if (await fileManager.fileExists('my-adapter.admin', 'backup.json')) {
    await fileManager.moveFile(
        'my-adapter.admin', 'backup.json',
        'my-adapter.admin', 'backups/backup-old.json'
    );
}
```

## Directory Operations

### Creating Directories

```javascript
// Create directory
adapter.mkdir('my-adapter.0', 'data/exports', (err) => {
    if (err) {
        adapter.log.error(`Failed to create directory: ${err.message}`);
    } else {
        adapter.log.info('Directory created successfully');
    }
});

// Async directory creation
async function ensureDirectory(namespace, path) {
    try {
        await adapter.mkdirAsync(namespace, path);
        adapter.log.debug(`Directory ensured: ${path}`);
        return true;
    } catch (error) {
        if (error.code === 'EEXIST') {
            // Directory already exists, that's fine
            return true;
        }
        adapter.log.error(`Failed to create directory ${path}: ${error.message}`);
        return false;
    }
}
```

### Reading Directories

```javascript
// List directory contents
adapter.readDir('my-adapter.0', 'data', (err, files) => {
    if (err) {
        adapter.log.error(`Failed to read directory: ${err.message}`);
        return;
    }
    
    files.forEach(file => {
        if (file.isDir) {
            adapter.log.info(`Directory: ${file.file}`);
        } else {
            adapter.log.info(`File: ${file.file} (${file.stats.size} bytes)`);
        }
    });
});

// Recursive directory listing
async function listAllFiles(namespace, path = '') {
    const allFiles = [];
    
    try {
        const files = await adapter.readDirAsync(namespace, path);
        
        for (const file of files) {
            const fullPath = path ? `${path}/${file.file}` : file.file;
            
            if (file.isDir) {
                // Recursively list subdirectory
                const subFiles = await listAllFiles(namespace, fullPath);
                allFiles.push(...subFiles);
            } else {
                allFiles.push({
                    path: fullPath,
                    size: file.stats.size,
                    modified: new Date(file.stats.mtime)
                });
            }
        }
    } catch (error) {
        adapter.log.error(`Failed to list files in ${path}: ${error.message}`);
    }
    
    return allFiles;
}

// Usage
const allFiles = await listAllFiles('my-adapter.0', 'data');
adapter.log.info(`Found ${allFiles.length} files`);
```

### Directory Management

```javascript
class DirectoryManager {
    constructor(adapter) {
        this.adapter = adapter;
    }

    async ensureDirectoryStructure(namespace, structure) {
        for (const dir of structure) {
            try {
                await this.adapter.mkdirAsync(namespace, dir);
                this.adapter.log.debug(`Created directory: ${dir}`);
            } catch (error) {
                if (error.code !== 'EEXIST') {
                    this.adapter.log.error(`Failed to create directory ${dir}: ${error.message}`);
                }
            }
        }
    }

    async cleanupOldFiles(namespace, path, maxAgeMs) {
        try {
            const files = await this.adapter.readDirAsync(namespace, path);
            const cutoffTime = Date.now() - maxAgeMs;
            
            for (const file of files) {
                if (!file.isDir && file.stats.mtime < cutoffTime) {
                    const filePath = path ? `${path}/${file.file}` : file.file;
                    
                    try {
                        await this.adapter.delFileAsync(namespace, filePath);
                        this.adapter.log.debug(`Deleted old file: ${filePath}`);
                    } catch (delErr) {
                        this.adapter.log.warn(`Failed to delete old file ${filePath}: ${delErr.message}`);
                    }
                }
            }
        } catch (error) {
            this.adapter.log.error(`Failed to cleanup directory ${path}: ${error.message}`);
        }
    }

    async getDiskUsage(namespace, path = '') {
        let totalSize = 0;
        let fileCount = 0;
        
        try {
            const files = await this.adapter.readDirAsync(namespace, path);
            
            for (const file of files) {
                if (file.isDir) {
                    const subPath = path ? `${path}/${file.file}` : file.file;
                    const subUsage = await this.getDiskUsage(namespace, subPath);
                    totalSize += subUsage.size;
                    fileCount += subUsage.files;
                } else {
                    totalSize += file.stats.size;
                    fileCount++;
                }
            }
        } catch (error) {
            this.adapter.log.error(`Failed to calculate disk usage for ${path}: ${error.message}`);
        }
        
        return { size: totalSize, files: fileCount };
    }
}

// Usage
const dirManager = new DirectoryManager(adapter);

// Ensure directory structure exists
await dirManager.ensureDirectoryStructure('my-adapter.0', [
    'data',
    'data/exports',
    'data/imports',
    'logs',
    'temp'
]);

// Cleanup files older than 7 days
const maxAge = 7 * 24 * 60 * 60 * 1000; // 7 days
await dirManager.cleanupOldFiles('my-adapter.0', 'logs', maxAge);

// Check disk usage
const usage = await dirManager.getDiskUsage('my-adapter.0');
adapter.log.info(`Disk usage: ${usage.size} bytes in ${usage.files} files`);
```

## File Permissions

### Setting File Permissions

```javascript
// Note: File permissions in ioBroker are handled differently than standard filesystem permissions
// They use the ACL system for access control

// Write file with specific permissions
const fileOptions = {
    mode: 0o644, // Read-write for owner, read-only for others
    encoding: 'utf8'
};

adapter.writeFile('my-adapter.admin', 'config.json', configData, fileOptions, (err) => {
    if (!err) {
        adapter.log.info('Config file written with specific permissions');
    }
});
```

### Access Control Lists (ACL)

```javascript
// Files in ioBroker use ACL for permissions
// This is handled automatically by the system, but you can check file access

async function checkFileAccess(namespace, filename) {
    try {
        const files = await adapter.readDirAsync(namespace, '');
        const file = files.find(f => f.file === filename);
        
        if (file && file.acl) {
            adapter.log.info(`File ACL - Owner: ${file.acl.owner}, Permissions: ${file.acl.permissions}`);
            return {
                canRead: file.acl.object && file.acl.object.read,
                canWrite: file.acl.object && file.acl.object.write,
                canDelete: file.acl.object && file.acl.object.delete
            };
        }
        
        return null;
    } catch (error) {
        adapter.log.error(`Failed to check file access: ${error.message}`);
        return null;
    }
}
```

## Best Practices

### Error Handling

```javascript
class SafeFileOperations {
    constructor(adapter) {
        this.adapter = adapter;
    }

    async safeReadFile(namespace, filename, options = {}) {
        try {
            const data = await this.adapter.readFileAsync(namespace, filename, options);
            return { success: true, data: data };
        } catch (error) {
            const errorInfo = {
                success: false,
                error: error.message,
                code: error.code
            };

            switch (error.code) {
                case 'ENOENT':
                    this.adapter.log.debug(`File not found: ${filename}`);
                    break;
                case 'EACCES':
                    this.adapter.log.error(`Permission denied: ${filename}`);
                    break;
                case 'EISDIR':
                    this.adapter.log.error(`Expected file but found directory: ${filename}`);
                    break;
                default:
                    this.adapter.log.error(`Failed to read file ${filename}: ${error.message}`);
            }

            return errorInfo;
        }
    }

    async safeWriteFile(namespace, filename, data, options = {}) {
        try {
            // Ensure directory exists
            const pathParts = filename.split('/');
            if (pathParts.length > 1) {
                const dirPath = pathParts.slice(0, -1).join('/');
                await this.ensureDirectory(namespace, dirPath);
            }

            await this.adapter.writeFileAsync(namespace, filename, data, options);
            return { success: true };
        } catch (error) {
            this.adapter.log.error(`Failed to write file ${filename}: ${error.message}`);
            return { success: false, error: error.message };
        }
    }

    async ensureDirectory(namespace, path) {
        try {
            await this.adapter.mkdirAsync(namespace, path);
        } catch (error) {
            if (error.code !== 'EEXIST') {
                throw error;
            }
        }
    }
}
```

### Configuration File Management

```javascript
class ConfigManager {
    constructor(adapter, filename = 'config.json') {
        this.adapter = adapter;
        this.filename = filename;
        this.namespace = `${adapter.name}.admin`;
        this.config = {};
        this.defaults = {};
    }

    setDefaults(defaults) {
        this.defaults = defaults;
    }

    async load() {
        try {
            const data = await this.adapter.readFileAsync(this.namespace, this.filename);
            
            if (data) {
                this.config = JSON.parse(data.toString());
                // Merge with defaults
                this.config = { ...this.defaults, ...this.config };
                return true;
            }
        } catch (error) {
            if (error.code === 'ENOENT') {
                // Create default config
                this.config = { ...this.defaults };
                await this.save();
                this.adapter.log.info('Created default configuration file');
                return true;
            } else {
                this.adapter.log.error(`Failed to load config: ${error.message}`);
                return false;
            }
        }
    }

    async save() {
        try {
            const configData = JSON.stringify(this.config, null, 2);
            await this.adapter.writeFileAsync(this.namespace, this.filename, configData);
            return true;
        } catch (error) {
            this.adapter.log.error(`Failed to save config: ${error.message}`);
            return false;
        }
    }

    get(key, defaultValue = undefined) {
        return this.config[key] !== undefined ? this.config[key] : defaultValue;
    }

    set(key, value) {
        this.config[key] = value;
    }

    async update(updates) {
        Object.assign(this.config, updates);
        return await this.save();
    }

    async backup() {
        const timestamp = new Date().toISOString().replace(/[:.]/g, '-');
        const backupFilename = `config-backup-${timestamp}.json`;
        
        try {
            const configData = JSON.stringify(this.config, null, 2);
            await this.adapter.writeFileAsync(this.namespace, `backups/${backupFilename}`, configData);
            this.adapter.log.info(`Configuration backed up to ${backupFilename}`);
            return true;
        } catch (error) {
            this.adapter.log.error(`Failed to backup config: ${error.message}`);
            return false;
        }
    }
}

// Usage
const configManager = new ConfigManager(adapter);
configManager.setDefaults({
    pollingInterval: 30000,
    enabled: true,
    timeout: 5000
});

await configManager.load();
const interval = configManager.get('pollingInterval');
configManager.set('lastStartup', Date.now());
await configManager.save();
```

### File Caching

```javascript
class FileCache {
    constructor(adapter, maxAge = 300000) { // 5 minutes default
        this.adapter = adapter;
        this.cache = new Map();
        this.maxAge = maxAge;
    }

    async readFile(namespace, filename, options = {}) {
        const cacheKey = `${namespace}:${filename}`;
        const cached = this.cache.get(cacheKey);
        
        if (cached && (Date.now() - cached.timestamp) < this.maxAge) {
            this.adapter.log.debug(`Cache hit for ${filename}`);
            return cached.data;
        }

        try {
            const data = await this.adapter.readFileAsync(namespace, filename, options);
            
            // Cache the result
            this.cache.set(cacheKey, {
                data: data,
                timestamp: Date.now()
            });
            
            this.adapter.log.debug(`File cached: ${filename}`);
            return data;
        } catch (error) {
            // Remove from cache on error
            this.cache.delete(cacheKey);
            throw error;
        }
    }

    invalidate(namespace, filename) {
        const cacheKey = `${namespace}:${filename}`;
        this.cache.delete(cacheKey);
    }

    clear() {
        this.cache.clear();
    }

    // Periodic cache cleanup
    startCleanup(intervalMs = 600000) { // 10 minutes
        this.cleanupInterval = setInterval(() => {
            const now = Date.now();
            for (const [key, value] of this.cache.entries()) {
                if ((now - value.timestamp) > this.maxAge) {
                    this.cache.delete(key);
                }
            }
        }, intervalMs);
    }

    stopCleanup() {
        if (this.cleanupInterval) {
            clearInterval(this.cleanupInterval);
        }
    }
}

// Usage
const fileCache = new FileCache(adapter, 300000); // 5 minute cache
fileCache.startCleanup();

// Cached file reading
const data = await fileCache.readFile('my-adapter.admin', 'config.json');
```

<!-- 
Source metadata for automated updates:
- Primary source: ioBroker.js-controller/packages/adapter/src/lib/adapter/adapter.ts
- File operations methods: readFile, writeFile, delFile, readDir, mkdir, etc.
- ACL system from ioBroker security model
- Error codes from Node.js filesystem operations
- Best practices from adapter development patterns
- Last updated: 2024-12-22
-->