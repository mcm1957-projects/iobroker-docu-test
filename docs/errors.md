# Error Handling

This document covers error types, handling patterns, and recovery strategies in ioBroker adapters.

## Table of Contents

- [Error Types](#error-types)
- [Error Handling Patterns](#error-handling-patterns)
- [Graceful Error Recovery](#graceful-error-recovery)
- [Error Reporting](#error-reporting)
- [Debugging Techniques](#debugging-techniques)
- [Best Practices](#best-practices)

## Error Types

### System Errors

Common system-level errors that adapters may encounter:

```javascript
// Network errors
const NETWORK_ERRORS = {
    ECONNREFUSED: 'Connection refused - service not available',
    ENOTFOUND: 'DNS lookup failed - hostname not found',
    ETIMEDOUT: 'Connection timeout',
    ECONNRESET: 'Connection reset by peer',
    EHOSTUNREACH: 'Host unreachable',
    ENETUNREACH: 'Network unreachable'
};

// File system errors
const FILE_ERRORS = {
    ENOENT: 'File or directory not found',
    EACCES: 'Permission denied',
    EEXIST: 'File already exists',
    EISDIR: 'Expected file but found directory',
    ENOTDIR: 'Expected directory but found file',
    ENOSPC: 'No space left on device'
};

// ioBroker specific errors
const IOBROKER_ERRORS = {
    ERROR_PERMISSION: 'Permission denied',
    ERROR_NOT_FOUND: 'Object not found',
    ERROR_ALREADY_EXISTS: 'Object already exists',
    ERROR_INVALID_ARGUMENTS: 'Invalid arguments',
    ERROR_DB: 'Database error',
    ERROR_TIMEOUT: 'Operation timeout'
};
```

### Custom Error Classes

```javascript
class AdapterError extends Error {
    constructor(message, code, details = {}) {
        super(message);
        this.name = 'AdapterError';
        this.code = code;
        this.details = details;
        this.timestamp = Date.now();
        
        // Capture stack trace
        if (Error.captureStackTrace) {
            Error.captureStackTrace(this, AdapterError);
        }
    }

    toJSON() {
        return {
            name: this.name,
            message: this.message,
            code: this.code,
            details: this.details,
            timestamp: this.timestamp,
            stack: this.stack
        };
    }
}

class ConnectionError extends AdapterError {
    constructor(message, host, port) {
        super(message, 'CONNECTION_ERROR', { host, port });
        this.name = 'ConnectionError';
    }
}

class ValidationError extends AdapterError {
    constructor(message, field, value) {
        super(message, 'VALIDATION_ERROR', { field, value });
        this.name = 'ValidationError';
    }
}

class DeviceError extends AdapterError {
    constructor(message, deviceId, deviceType) {
        super(message, 'DEVICE_ERROR', { deviceId, deviceType });
        this.name = 'DeviceError';
    }
}

class ConfigurationError extends AdapterError {
    constructor(message, configKey) {
        super(message, 'CONFIGURATION_ERROR', { configKey });
        this.name = 'ConfigurationError';
    }
}

// Usage
throw new ConnectionError('Failed to connect to device', '192.168.1.100', 8080);
throw new ValidationError('Temperature value out of range', 'temperature', -300);
throw new DeviceError('Device not responding', 'sensor-01', 'temperature-sensor');
```

## Error Handling Patterns

### Try-Catch with Async/Await

```javascript
class RobustAdapter {
    async connectToDevice() {
        try {
            await this.establishConnection();
            await this.authenticateDevice();
            await this.initializeDevice();
            
            this.adapter.log.info('Device connected successfully');
            return true;
        } catch (error) {
            if (error instanceof ConnectionError) {
                this.adapter.log.error(`Connection failed: ${error.message}`);
                this.scheduleReconnect();
            } else if (error instanceof ValidationError) {
                this.adapter.log.error(`Validation error: ${error.message}`);
                // Don't retry validation errors
            } else {
                this.adapter.log.error(`Unexpected error: ${error.message}`);
                this.handleUnexpectedError(error);
            }
            return false;
        }
    }

    async safeStateUpdate(id, value) {
        try {
            await this.adapter.setStateAsync(id, value, true);
        } catch (error) {
            this.adapter.log.error(`Failed to update state ${id}: ${error.message}`);
            
            // Try alternative update method or store for later retry
            this.queueStateUpdate(id, value);
        }
    }

    async withRetry(operation, maxRetries = 3, delay = 1000) {
        let lastError;
        
        for (let attempt = 1; attempt <= maxRetries; attempt++) {
            try {
                return await operation();
            } catch (error) {
                lastError = error;
                
                if (attempt === maxRetries) {
                    throw error;
                }
                
                this.adapter.log.warn(`Attempt ${attempt} failed: ${error.message}, retrying in ${delay}ms`);
                await this.sleep(delay);
                
                // Exponential backoff
                delay *= 2;
            }
        }
        
        throw lastError;
    }

    sleep(ms) {
        return new Promise(resolve => setTimeout(resolve, ms));
    }
}
```

### Promise Error Handling

```javascript
class PromiseErrorHandler {
    constructor(adapter) {
        this.adapter = adapter;
    }

    // Handle multiple promises with individual error handling
    async processMultipleDevices(devices) {
        const results = await Promise.allSettled(
            devices.map(device => this.processDevice(device))
        );

        const successful = [];
        const failed = [];

        results.forEach((result, index) => {
            if (result.status === 'fulfilled') {
                successful.push({
                    device: devices[index],
                    result: result.value
                });
            } else {
                failed.push({
                    device: devices[index],
                    error: result.reason
                });
                
                this.adapter.log.error(
                    `Failed to process device ${devices[index].id}: ${result.reason.message}`
                );
            }
        });

        this.adapter.log.info(`Processed ${successful.length} devices, ${failed.length} failed`);
        return { successful, failed };
    }

    async processDevice(device) {
        // This might throw
        const data = await this.fetchDeviceData(device);
        await this.updateDeviceState(device, data);
        return data;
    }

    // Timeout wrapper for promises
    withTimeout(promise, timeoutMs) {
        const timeoutPromise = new Promise((_, reject) => {
            setTimeout(() => {
                reject(new Error(`Operation timed out after ${timeoutMs}ms`));
            }, timeoutMs);
        });

        return Promise.race([promise, timeoutPromise]);
    }

    // Circuit breaker pattern
    createCircuitBreaker(operation, threshold = 5, timeout = 60000) {
        let failures = 0;
        let lastFailureTime = 0;
        let state = 'CLOSED'; // CLOSED, OPEN, HALF_OPEN

        return async (...args) => {
            const now = Date.now();

            if (state === 'OPEN') {
                if (now - lastFailureTime > timeout) {
                    state = 'HALF_OPEN';
                    this.adapter.log.info('Circuit breaker half-open, testing...');
                } else {
                    throw new Error('Circuit breaker is OPEN');
                }
            }

            try {
                const result = await operation(...args);
                
                if (state === 'HALF_OPEN') {
                    state = 'CLOSED';
                    failures = 0;
                    this.adapter.log.info('Circuit breaker closed, operation successful');
                }
                
                return result;
            } catch (error) {
                failures++;
                lastFailureTime = now;

                if (failures >= threshold) {
                    state = 'OPEN';
                    this.adapter.log.error(`Circuit breaker opened after ${failures} failures`);
                }

                throw error;
            }
        };
    }
}

// Usage
const errorHandler = new PromiseErrorHandler(adapter);

// Circuit breaker for device communication
const safeDeviceOperation = errorHandler.createCircuitBreaker(
    async (deviceId) => {
        return await this.communicateWithDevice(deviceId);
    },
    3, // Open after 3 failures
    30000 // Reset after 30 seconds
);
```

### Event-Based Error Handling

```javascript
class EventErrorHandler extends EventEmitter {
    constructor(adapter) {
        super();
        this.adapter = adapter;
        this.errorCounts = new Map();
        this.setupErrorHandling();
    }

    setupErrorHandling() {
        // Handle different error types
        this.on('connection-error', (error) => {
            this.handleConnectionError(error);
        });

        this.on('device-error', (error) => {
            this.handleDeviceError(error);
        });

        this.on('validation-error', (error) => {
            this.handleValidationError(error);
        });

        this.on('critical-error', (error) => {
            this.handleCriticalError(error);
        });
    }

    reportError(type, error, context = {}) {
        const errorKey = `${type}:${context.deviceId || 'system'}`;
        const count = this.errorCounts.get(errorKey) || 0;
        this.errorCounts.set(errorKey, count + 1);

        const errorData = {
            type,
            error,
            context,
            count: count + 1,
            timestamp: Date.now()
        };

        this.emit(type, errorData);

        // Emit critical-error if threshold exceeded
        if (count >= 5) {
            this.emit('critical-error', errorData);
        }
    }

    handleConnectionError(errorData) {
        this.adapter.log.error(`Connection error #${errorData.count}: ${errorData.error.message}`);
        
        if (errorData.count === 1) {
            // First failure - schedule immediate retry
            this.scheduleRetry(errorData.context, 5000);
        } else if (errorData.count < 5) {
            // Multiple failures - increase delay
            const delay = Math.min(errorData.count * 30000, 300000); // Max 5 minutes
            this.scheduleRetry(errorData.context, delay);
        } else {
            // Too many failures - disable device
            this.disableDevice(errorData.context.deviceId);
        }
    }

    handleDeviceError(errorData) {
        this.adapter.log.warn(`Device error #${errorData.count}: ${errorData.error.message}`);
        
        // Update device status
        this.adapter.setState(
            `devices.${errorData.context.deviceId}.error`,
            errorData.error.message,
            true
        );
    }

    handleValidationError(errorData) {
        this.adapter.log.error(`Validation error: ${errorData.error.message}`);
        
        // Validation errors usually don't need retry
        // but we should notify about configuration issues
        this.adapter.setState('info.configurationError', true, true);
    }

    handleCriticalError(errorData) {
        this.adapter.log.error(`CRITICAL: ${errorData.type} occurred ${errorData.count} times`);
        
        // Notify administrators
        this.sendCriticalAlert(errorData);
        
        // Consider restarting adapter
        if (errorData.count >= 10) {
            this.adapter.log.error('Too many critical errors, restarting adapter');
            this.adapter.restart();
        }
    }
}

// Usage
const errorHandler = new EventErrorHandler(adapter);

// Report errors
try {
    await connectToDevice(deviceId);
} catch (error) {
    errorHandler.reportError('connection-error', error, { deviceId });
}
```

## Graceful Error Recovery

### State Recovery

```javascript
class StateRecovery {
    constructor(adapter) {
        this.adapter = adapter;
        this.stateBackup = new Map();
        this.recoveryStrategies = new Map();
        this.setupRecoveryStrategies();
    }

    setupRecoveryStrategies() {
        this.recoveryStrategies.set('connection-lost', async (context) => {
            // Try to reconnect
            await this.attemptReconnection(context.deviceId);
        });

        this.recoveryStrategies.set('invalid-data', async (context) => {
            // Use last known good value
            await this.restoreLastKnownValue(context.stateId);
        });

        this.recoveryStrategies.set('timeout', async (context) => {
            // Retry with increased timeout
            await this.retryWithTimeout(context.operation, context.timeout * 2);
        });
    }

    async backupState(stateId) {
        try {
            const state = await this.adapter.getStateAsync(stateId);
            if (state) {
                this.stateBackup.set(stateId, {
                    value: state.val,
                    timestamp: state.ts,
                    quality: state.q
                });
            }
        } catch (error) {
            this.adapter.log.warn(`Failed to backup state ${stateId}: ${error.message}`);
        }
    }

    async restoreState(stateId, reason = 'error recovery') {
        const backup = this.stateBackup.get(stateId);
        if (backup) {
            try {
                await this.adapter.setStateAsync(stateId, {
                    val: backup.value,
                    ack: true,
                    q: 0x40, // Substitute value quality
                    c: `Restored due to ${reason}`
                });
                
                this.adapter.log.info(`Restored state ${stateId} from backup`);
                return true;
            } catch (error) {
                this.adapter.log.error(`Failed to restore state ${stateId}: ${error.message}`);
            }
        }
        return false;
    }

    async executeRecovery(errorType, context) {
        const strategy = this.recoveryStrategies.get(errorType);
        if (strategy) {
            try {
                await strategy(context);
                this.adapter.log.info(`Recovery successful for ${errorType}`);
                return true;
            } catch (error) {
                this.adapter.log.error(`Recovery failed for ${errorType}: ${error.message}`);
            }
        } else {
            this.adapter.log.warn(`No recovery strategy for error type: ${errorType}`);
        }
        return false;
    }

    async attemptReconnection(deviceId, maxAttempts = 3) {
        for (let attempt = 1; attempt <= maxAttempts; attempt++) {
            try {
                await this.connectToDevice(deviceId);
                this.adapter.log.info(`Reconnection successful for device ${deviceId}`);
                return true;
            } catch (error) {
                this.adapter.log.warn(`Reconnection attempt ${attempt} failed: ${error.message}`);
                
                if (attempt < maxAttempts) {
                    await this.sleep(attempt * 5000); // Exponential backoff
                }
            }
        }
        
        this.adapter.log.error(`Failed to reconnect to device ${deviceId} after ${maxAttempts} attempts`);
        return false;
    }
}
```

### Fallback Mechanisms

```javascript
class FallbackManager {
    constructor(adapter) {
        this.adapter = adapter;
        this.fallbackSources = new Map();
        this.primarySources = new Map();
    }

    registerFallback(dataType, primarySource, fallbackSource) {
        this.primarySources.set(dataType, primarySource);
        this.fallbackSources.set(dataType, fallbackSource);
    }

    async getData(dataType, context = {}) {
        const primarySource = this.primarySources.get(dataType);
        const fallbackSource = this.fallbackSources.get(dataType);

        try {
            // Try primary source first
            const data = await primarySource(context);
            this.adapter.log.debug(`Primary source successful for ${dataType}`);
            return { data, source: 'primary' };
        } catch (primaryError) {
            this.adapter.log.warn(`Primary source failed for ${dataType}: ${primaryError.message}`);
            
            if (fallbackSource) {
                try {
                    const data = await fallbackSource(context);
                    this.adapter.log.info(`Fallback source successful for ${dataType}`);
                    return { data, source: 'fallback' };
                } catch (fallbackError) {
                    this.adapter.log.error(`Fallback source also failed for ${dataType}: ${fallbackError.message}`);
                    throw new Error(`Both primary and fallback sources failed for ${dataType}`);
                }
            } else {
                throw primaryError;
            }
        }
    }

    async getDataWithCache(dataType, context = {}, maxAge = 300000) { // 5 minutes
        try {
            return await this.getData(dataType, context);
        } catch (error) {
            // Try cached data as last resort
            const cached = await this.getCachedData(dataType, maxAge);
            if (cached) {
                this.adapter.log.warn(`Using cached data for ${dataType} due to error`);
                return { data: cached, source: 'cache' };
            }
            throw error;
        }
    }

    async getCachedData(dataType, maxAge) {
        try {
            const cacheKey = `cache.${dataType}`;
            const state = await this.adapter.getStateAsync(cacheKey);
            
            if (state && (Date.now() - state.ts) < maxAge) {
                return JSON.parse(state.val);
            }
        } catch (error) {
            this.adapter.log.debug(`Failed to get cached data: ${error.message}`);
        }
        return null;
    }

    async setCachedData(dataType, data) {
        try {
            const cacheKey = `cache.${dataType}`;
            await this.adapter.setStateAsync(cacheKey, JSON.stringify(data), true);
        } catch (error) {
            this.adapter.log.debug(`Failed to cache data: ${error.message}`);
        }
    }
}

// Usage
const fallbackManager = new FallbackManager(adapter);

// Register weather data sources
fallbackManager.registerFallback(
    'weather',
    async () => await primaryWeatherApi.getCurrentWeather(),
    async () => await backupWeatherApi.getCurrentWeather()
);

// Get weather data with fallback
try {
    const { data, source } = await fallbackManager.getDataWithCache('weather');
    adapter.log.info(`Weather data from ${source}: ${JSON.stringify(data)}`);
} catch (error) {
    adapter.log.error(`All weather sources failed: ${error.message}`);
}
```

## Error Reporting

### Error Logging

```javascript
class ErrorLogger {
    constructor(adapter) {
        this.adapter = adapter;
        this.errorLog = [];
        this.maxLogEntries = 1000;
        this.setupErrorLogging();
    }

    setupErrorLogging() {
        // Capture all uncaught errors
        process.on('uncaughtException', (error) => {
            this.logError('UNCAUGHT_EXCEPTION', error, { fatal: true });
        });

        process.on('unhandledRejection', (reason, promise) => {
            this.logError('UNHANDLED_REJECTION', reason, { promise });
        });
    }

    logError(type, error, context = {}) {
        const errorEntry = {
            id: this.generateErrorId(),
            type,
            message: error.message,
            stack: error.stack,
            code: error.code,
            context,
            timestamp: Date.now(),
            adapterName: this.adapter.name,
            adapterVersion: this.adapter.version
        };

        // Add to in-memory log
        this.errorLog.push(errorEntry);
        
        // Trim log if too large
        if (this.errorLog.length > this.maxLogEntries) {
            this.errorLog = this.errorLog.slice(-this.maxLogEntries / 2);
        }

        // Log to adapter log
        this.adapter.log.error(`[${errorEntry.id}] ${type}: ${error.message}`);

        // Save to file for persistent logging
        this.saveErrorToFile(errorEntry);

        // Update error statistics
        this.updateErrorStats(type);

        return errorEntry.id;
    }

    async saveErrorToFile(errorEntry) {
        try {
            const errorLogPath = 'logs/errors.jsonl';
            const logLine = JSON.stringify(errorEntry) + '\n';
            
            // Append to error log file
            const existingLog = await this.adapter.readFileAsync(
                `${this.adapter.name}.0`, 
                errorLogPath
            ).catch(() => '');
            
            const updatedLog = existingLog + logLine;
            
            await this.adapter.writeFileAsync(
                `${this.adapter.name}.0`,
                errorLogPath,
                updatedLog
            );
        } catch (error) {
            this.adapter.log.warn(`Failed to save error to file: ${error.message}`);
        }
    }

    updateErrorStats(type) {
        this.adapter.setState(`statistics.errors.${type}`, {
            val: this.getErrorCount(type),
            ack: true
        });

        this.adapter.setState('statistics.errors.total', {
            val: this.errorLog.length,
            ack: true
        });
    }

    getErrorCount(type) {
        return this.errorLog.filter(entry => entry.type === type).length;
    }

    getRecentErrors(minutes = 60) {
        const cutoff = Date.now() - (minutes * 60 * 1000);
        return this.errorLog.filter(entry => entry.timestamp > cutoff);
    }

    generateErrorId() {
        return Date.now().toString(36) + Math.random().toString(36).substr(2, 5);
    }

    async getErrorReport() {
        const recent = this.getRecentErrors();
        const byType = {};
        
        recent.forEach(error => {
            byType[error.type] = (byType[error.type] || 0) + 1;
        });

        return {
            totalErrors: this.errorLog.length,
            recentErrors: recent.length,
            errorsByType: byType,
            lastError: this.errorLog[this.errorLog.length - 1],
            topErrors: this.getTopErrors()
        };
    }

    getTopErrors(limit = 5) {
        const errorCounts = {};
        
        this.errorLog.forEach(error => {
            const key = `${error.type}:${error.message}`;
            errorCounts[key] = (errorCounts[key] || 0) + 1;
        });

        return Object.entries(errorCounts)
            .sort(([,a], [,b]) => b - a)
            .slice(0, limit)
            .map(([error, count]) => ({ error, count }));
    }
}
```

### Health Monitoring

```javascript
class HealthMonitor {
    constructor(adapter) {
        this.adapter = adapter;
        this.healthScore = 100;
        this.metrics = {
            errors: 0,
            warnings: 0,
            successfulOperations: 0,
            failedOperations: 0,
            lastHealthCheck: Date.now()
        };
        
        this.startHealthChecks();
    }

    startHealthChecks() {
        // Run health check every minute
        setInterval(() => {
            this.performHealthCheck();
        }, 60000);

        // Update health score every 5 minutes
        setInterval(() => {
            this.updateHealthScore();
        }, 300000);
    }

    async performHealthCheck() {
        const checks = [
            this.checkConnectivity(),
            this.checkMemoryUsage(),
            this.checkErrorRate(),
            this.checkResponseTime()
        ];

        const results = await Promise.allSettled(checks);
        const failures = results.filter(r => r.status === 'rejected').length;
        const healthPercentage = ((results.length - failures) / results.length) * 100;

        this.adapter.setState('health.score', Math.round(healthPercentage), true);
        this.adapter.setState('health.lastCheck', Date.now(), true);

        if (healthPercentage < 70) {
            this.adapter.log.warn(`Health check failed: ${healthPercentage}% healthy`);
        }

        this.metrics.lastHealthCheck = Date.now();
    }

    async checkConnectivity() {
        // Check if adapter can communicate with its services
        if (this.adapter.config.testEndpoint) {
            const start = Date.now();
            try {
                await this.adapter.httpClient.get(this.adapter.config.testEndpoint);
                const responseTime = Date.now() - start;
                this.adapter.setState('health.connectivity', true, true);
                this.adapter.setState('health.responseTime', responseTime, true);
                return true;
            } catch (error) {
                this.adapter.setState('health.connectivity', false, true);
                throw new Error(`Connectivity check failed: ${error.message}`);
            }
        }
        return true;
    }

    checkMemoryUsage() {
        const usage = process.memoryUsage();
        const usageMB = Math.round(usage.heapUsed / 1024 / 1024);
        
        this.adapter.setState('health.memoryUsage', usageMB, true);
        
        if (usageMB > 500) { // 500MB threshold
            throw new Error(`High memory usage: ${usageMB}MB`);
        }
        
        return true;
    }

    checkErrorRate() {
        const recentMinutes = 15;
        const recentOperations = this.metrics.successfulOperations + this.metrics.failedOperations;
        
        if (recentOperations > 0) {
            const errorRate = (this.metrics.failedOperations / recentOperations) * 100;
            this.adapter.setState('health.errorRate', errorRate, true);
            
            if (errorRate > 10) { // 10% error rate threshold
                throw new Error(`High error rate: ${errorRate}%`);
            }
        }
        
        return true;
    }

    checkResponseTime() {
        // This would be implemented based on your specific metrics
        return true;
    }

    updateHealthScore() {
        const errorWeight = Math.min(this.metrics.errors * 2, 50);
        const warningWeight = Math.min(this.metrics.warnings * 1, 30);
        
        this.healthScore = Math.max(100 - errorWeight - warningWeight, 0);
        this.adapter.setState('health.overallScore', this.healthScore, true);
    }

    recordSuccess() {
        this.metrics.successfulOperations++;
    }

    recordFailure() {
        this.metrics.failedOperations++;
        this.metrics.errors++;
    }

    recordWarning() {
        this.metrics.warnings++;
    }
}
```

## Debugging Techniques

### Debug Logging

```javascript
class DebugLogger {
    constructor(adapter) {
        this.adapter = adapter;
        this.debugEnabled = adapter.config.debug || false;
        this.traceEnabled = adapter.config.trace || false;
        this.debugBuffer = [];
        this.maxBufferSize = 1000;
    }

    debug(message, context = {}) {
        if (this.debugEnabled) {
            this.adapter.log.debug(this.formatDebugMessage(message, context));
        }
        
        this.addToBuffer('debug', message, context);
    }

    trace(operation, context = {}) {
        if (this.traceEnabled) {
            this.adapter.log.silly(`TRACE: ${operation} - ${JSON.stringify(context)}`);
        }
        
        this.addToBuffer('trace', operation, context);
    }

    formatDebugMessage(message, context) {
        if (Object.keys(context).length > 0) {
            return `${message} | Context: ${JSON.stringify(context)}`;
        }
        return message;
    }

    addToBuffer(level, message, context) {
        this.debugBuffer.push({
            level,
            message,
            context,
            timestamp: Date.now(),
            stack: this.traceEnabled ? new Error().stack : null
        });

        if (this.debugBuffer.length > this.maxBufferSize) {
            this.debugBuffer = this.debugBuffer.slice(-this.maxBufferSize / 2);
        }
    }

    async dumpDebugInfo() {
        const debugInfo = {
            adapterName: this.adapter.name,
            adapterVersion: this.adapter.version,
            nodeVersion: process.version,
            platform: process.platform,
            memoryUsage: process.memoryUsage(),
            uptime: process.uptime(),
            debugBuffer: this.debugBuffer.slice(-100), // Last 100 entries
            configuration: this.sanitizeConfig(this.adapter.config),
            timestamp: Date.now()
        };

        const filename = `debug-dump-${Date.now()}.json`;
        
        try {
            await this.adapter.writeFileAsync(
                `${this.adapter.name}.0`,
                `debug/${filename}`,
                JSON.stringify(debugInfo, null, 2)
            );
            
            this.adapter.log.info(`Debug info dumped to ${filename}`);
            return filename;
        } catch (error) {
            this.adapter.log.error(`Failed to dump debug info: ${error.message}`);
            return null;
        }
    }

    sanitizeConfig(config) {
        const sanitized = { ...config };
        
        // Remove sensitive information
        const sensitiveKeys = ['password', 'apiKey', 'token', 'secret'];
        sensitiveKeys.forEach(key => {
            if (sanitized[key]) {
                sanitized[key] = '***';
            }
        });
        
        return sanitized;
    }

    // Performance measurement decorator
    measurePerformance(target, propertyName, descriptor) {
        const originalMethod = descriptor.value;
        
        descriptor.value = async function(...args) {
            const start = Date.now();
            const operationId = `${target.constructor.name}.${propertyName}`;
            
            this.debugLogger.trace(`Starting ${operationId}`, { args });
            
            try {
                const result = await originalMethod.apply(this, args);
                const duration = Date.now() - start;
                
                this.debugLogger.trace(`Completed ${operationId}`, { duration, success: true });
                
                if (duration > 5000) { // Log slow operations
                    this.adapter.log.warn(`Slow operation: ${operationId} took ${duration}ms`);
                }
                
                return result;
            } catch (error) {
                const duration = Date.now() - start;
                this.debugLogger.trace(`Failed ${operationId}`, { duration, error: error.message });
                throw error;
            }
        };
        
        return descriptor;
    }
}

// Usage with decorator
class MyAdapter {
    constructor() {
        this.debugLogger = new DebugLogger(this.adapter);
    }

    @this.debugLogger.measurePerformance
    async fetchDeviceData(deviceId) {
        this.debugLogger.debug('Fetching device data', { deviceId });
        // Implementation
    }
}
```

## Best Practices

### Error Handling Checklist

```javascript
class ErrorHandlingBestPractices {
    static getChecklist() {
        return {
            errorTypes: [
                '✓ Define custom error classes for different error types',
                '✓ Use descriptive error messages with context',
                '✓ Include error codes for programmatic handling',
                '✓ Preserve original error information in wrapped errors'
            ],
            
            handlingPatterns: [
                '✓ Use try-catch blocks for async operations',
                '✓ Handle Promise rejections with .catch() or try-catch',
                '✓ Implement retry logic with exponential backoff',
                '✓ Use circuit breakers for external service calls',
                '✓ Validate inputs before processing'
            ],
            
            recovery: [
                '✓ Implement graceful degradation strategies',
                '✓ Provide fallback mechanisms for critical operations',
                '✓ Cache data for offline scenarios',
                '✓ Restore to last known good state when possible',
                '✓ Schedule automatic recovery attempts'
            ],
            
            logging: [
                '✓ Log errors with sufficient context',
                '✓ Use appropriate log levels (error, warn, info)',
                '✓ Avoid logging sensitive information',
                '✓ Include error IDs for tracking',
                '✓ Implement structured logging'
            ],
            
            monitoring: [
                '✓ Track error rates and patterns',
                '✓ Monitor system health indicators',
                '✓ Set up alerts for critical errors',
                '✓ Collect performance metrics',
                '✓ Implement health check endpoints'
            ],
            
            prevention: [
                '✓ Validate configuration on startup',
                '✓ Use defensive programming techniques',
                '✓ Implement timeouts for all external calls',
                '✓ Test error scenarios in development',
                '✓ Use TypeScript for better error prevention'
            ]
        };
    }

    static validateImplementation(adapter) {
        const issues = [];

        // Check if adapter has error handling
        if (!adapter.on || typeof adapter.on !== 'function') {
            issues.push('Adapter should implement event handling for errors');
        }

        // Check for unhandled rejection handlers
        const hasUnhandledRejectionHandler = process.listenerCount('unhandledRejection') > 0;
        if (!hasUnhandledRejectionHandler) {
            issues.push('Should implement unhandledRejection handler');
        }

        // Check for uncaught exception handlers
        const hasUncaughtExceptionHandler = process.listenerCount('uncaughtException') > 0;
        if (!hasUncaughtExceptionHandler) {
            issues.push('Should implement uncaughtException handler');
        }

        return {
            isValid: issues.length === 0,
            issues
        };
    }
}

// Comprehensive error handling setup
function setupRobustErrorHandling(adapter) {
    // Global error handlers
    process.on('uncaughtException', (error) => {
        adapter.log.error(`Uncaught Exception: ${error.message}\n${error.stack}`);
        
        // Attempt graceful shutdown
        adapter.terminate('Uncaught exception occurred', 1);
    });

    process.on('unhandledRejection', (reason, promise) => {
        adapter.log.error(`Unhandled Rejection at: ${promise}, reason: ${reason}`);
        
        // Don't terminate on unhandled rejection, but log it
        if (reason instanceof Error) {
            adapter.log.error(`Stack: ${reason.stack}`);
        }
    });

    // Adapter error handling
    adapter.on('error', (error) => {
        adapter.log.error(`Adapter error: ${error.message}`);
    });

    // Setup graceful shutdown
    const gracefulShutdown = (signal) => {
        adapter.log.info(`Received ${signal}, shutting down gracefully`);
        
        // Cleanup and exit
        adapter.terminate('Graceful shutdown', 0);
    };

    process.on('SIGTERM', gracefulShutdown);
    process.on('SIGINT', gracefulShutdown);
}

// Usage
setupRobustErrorHandling(adapter);
const validation = ErrorHandlingBestPractices.validateImplementation(adapter);
if (!validation.isValid) {
    adapter.log.warn('Error handling validation issues:', validation.issues);
}
```

<!-- 
Source metadata for automated updates:
- Primary source: ioBroker.js-controller error handling patterns
- Node.js error handling best practices
- JavaScript error types and codes
- ioBroker adapter lifecycle error management
- Circuit breaker and retry patterns from resilient systems
- Health monitoring patterns from production systems
- Last updated: 2024-12-22
-->