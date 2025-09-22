# Network & Communication

This document covers HTTP requests, messaging, and communication patterns in ioBroker adapters.

## Table of Contents

- [HTTP Requests](#http-requests)
- [Inter-Adapter Communication](#inter-adapter-communication)
- [WebSocket Communication](#websocket-communication)
- [Message Handling](#message-handling)
- [Network Utilities](#network-utilities)
- [Security Considerations](#security-considerations)

## HTTP Requests

### Making HTTP Requests

ioBroker adapters can make HTTP requests using Node.js built-in modules or external libraries.

```javascript
const https = require('https');
const http = require('http');

class HttpClient {
    constructor(adapter) {
        this.adapter = adapter;
        this.defaultTimeout = 10000;
    }

    async get(url, options = {}) {
        return new Promise((resolve, reject) => {
            const urlObj = new URL(url);
            const isHttps = urlObj.protocol === 'https:';
            const client = isHttps ? https : http;

            const requestOptions = {
                hostname: urlObj.hostname,
                port: urlObj.port || (isHttps ? 443 : 80),
                path: urlObj.pathname + urlObj.search,
                method: 'GET',
                timeout: options.timeout || this.defaultTimeout,
                headers: {
                    'User-Agent': `ioBroker-${this.adapter.name}/${this.adapter.version}`,
                    ...options.headers
                }
            };

            const req = client.request(requestOptions, (res) => {
                let data = '';
                
                res.on('data', (chunk) => {
                    data += chunk;
                });

                res.on('end', () => {
                    if (res.statusCode >= 200 && res.statusCode < 300) {
                        resolve({
                            statusCode: res.statusCode,
                            headers: res.headers,
                            data: data
                        });
                    } else {
                        reject(new Error(`HTTP ${res.statusCode}: ${res.statusMessage}`));
                    }
                });
            });

            req.on('error', (error) => {
                reject(error);
            });

            req.on('timeout', () => {
                req.destroy();
                reject(new Error('Request timeout'));
            });

            req.end();
        });
    }

    async post(url, data, options = {}) {
        return new Promise((resolve, reject) => {
            const urlObj = new URL(url);
            const isHttps = urlObj.protocol === 'https:';
            const client = isHttps ? https : http;

            const postData = typeof data === 'string' ? data : JSON.stringify(data);
            const contentType = options.contentType || 'application/json';

            const requestOptions = {
                hostname: urlObj.hostname,
                port: urlObj.port || (isHttps ? 443 : 80),
                path: urlObj.pathname + urlObj.search,
                method: 'POST',
                timeout: options.timeout || this.defaultTimeout,
                headers: {
                    'Content-Type': contentType,
                    'Content-Length': Buffer.byteLength(postData),
                    'User-Agent': `ioBroker-${this.adapter.name}/${this.adapter.version}`,
                    ...options.headers
                }
            };

            const req = client.request(requestOptions, (res) => {
                let responseData = '';
                
                res.on('data', (chunk) => {
                    responseData += chunk;
                });

                res.on('end', () => {
                    if (res.statusCode >= 200 && res.statusCode < 300) {
                        resolve({
                            statusCode: res.statusCode,
                            headers: res.headers,
                            data: responseData
                        });
                    } else {
                        reject(new Error(`HTTP ${res.statusCode}: ${res.statusMessage}`));
                    }
                });
            });

            req.on('error', (error) => {
                reject(error);
            });

            req.on('timeout', () => {
                req.destroy();
                reject(new Error('Request timeout'));
            });

            req.write(postData);
            req.end();
        });
    }
}

// Usage
const httpClient = new HttpClient(adapter);

try {
    const response = await httpClient.get('https://api.example.com/data');
    const apiData = JSON.parse(response.data);
    adapter.log.info('API data received successfully');
} catch (error) {
    adapter.log.error(`API request failed: ${error.message}`);
}
```

### Using External Libraries

```javascript
// Using axios (if installed as dependency)
const axios = require('axios');

class AxiosHttpClient {
    constructor(adapter) {
        this.adapter = adapter;
        
        // Create axios instance with default config
        this.client = axios.create({
            timeout: 10000,
            headers: {
                'User-Agent': `ioBroker-${adapter.name}/${adapter.version}`
            }
        });

        // Add request interceptor
        this.client.interceptors.request.use(
            (config) => {
                this.adapter.log.debug(`HTTP Request: ${config.method.toUpperCase()} ${config.url}`);
                return config;
            },
            (error) => {
                this.adapter.log.error(`Request error: ${error.message}`);
                return Promise.reject(error);
            }
        );

        // Add response interceptor
        this.client.interceptors.response.use(
            (response) => {
                this.adapter.log.debug(`HTTP Response: ${response.status} ${response.statusText}`);
                return response;
            },
            (error) => {
                if (error.response) {
                    this.adapter.log.error(`HTTP Error: ${error.response.status} ${error.response.statusText}`);
                } else if (error.request) {
                    this.adapter.log.error('No response received from server');
                } else {
                    this.adapter.log.error(`Request setup error: ${error.message}`);
                }
                return Promise.reject(error);
            }
        );
    }

    async get(url, options = {}) {
        try {
            const response = await this.client.get(url, options);
            return response.data;
        } catch (error) {
            throw new Error(`GET request failed: ${error.message}`);
        }
    }

    async post(url, data, options = {}) {
        try {
            const response = await this.client.post(url, data, options);
            return response.data;
        } catch (error) {
            throw new Error(`POST request failed: ${error.message}`);
        }
    }

    async put(url, data, options = {}) {
        try {
            const response = await this.client.put(url, data, options);
            return response.data;
        } catch (error) {
            throw new Error(`PUT request failed: ${error.message}`);
        }
    }

    async delete(url, options = {}) {
        try {
            const response = await this.client.delete(url, options);
            return response.data;
        } catch (error) {
            throw new Error(`DELETE request failed: ${error.message}`);
        }
    }
}
```

### REST API Client

```javascript
class RestApiClient {
    constructor(adapter, baseUrl, apiKey) {
        this.adapter = adapter;
        this.baseUrl = baseUrl.replace(/\/$/, ''); // Remove trailing slash
        this.apiKey = apiKey;
        this.httpClient = new HttpClient(adapter);
    }

    async request(endpoint, method = 'GET', data = null, options = {}) {
        const url = `${this.baseUrl}${endpoint}`;
        
        const requestOptions = {
            headers: {
                'Authorization': `Bearer ${this.apiKey}`,
                'Accept': 'application/json',
                ...options.headers
            },
            ...options
        };

        try {
            let response;
            
            switch (method.toUpperCase()) {
                case 'GET':
                    response = await this.httpClient.get(url, requestOptions);
                    break;
                case 'POST':
                    response = await this.httpClient.post(url, data, requestOptions);
                    break;
                case 'PUT':
                    response = await this.httpClient.put(url, data, requestOptions);
                    break;
                case 'DELETE':
                    response = await this.httpClient.delete(url, requestOptions);
                    break;
                default:
                    throw new Error(`Unsupported HTTP method: ${method}`);
            }

            // Parse JSON response if applicable
            if (response.headers['content-type']?.includes('application/json')) {
                response.data = JSON.parse(response.data);
            }

            return response;
        } catch (error) {
            this.adapter.log.error(`API request failed: ${endpoint} - ${error.message}`);
            throw error;
        }
    }

    async getDevices() {
        const response = await this.request('/devices');
        return response.data;
    }

    async getDevice(deviceId) {
        const response = await this.request(`/devices/${deviceId}`);
        return response.data;
    }

    async updateDevice(deviceId, deviceData) {
        const response = await this.request(`/devices/${deviceId}`, 'PUT', deviceData);
        return response.data;
    }

    async deleteDevice(deviceId) {
        await this.request(`/devices/${deviceId}`, 'DELETE');
        return true;
    }
}

// Usage
const apiClient = new RestApiClient(adapter, adapter.config.apiUrl, adapter.config.apiKey);

try {
    const devices = await apiClient.getDevices();
    adapter.log.info(`Found ${devices.length} devices`);
    
    for (const device of devices) {
        await this.processDevice(device);
    }
} catch (error) {
    adapter.log.error(`Failed to fetch devices: ${error.message}`);
}
```

## Inter-Adapter Communication

### Sending Messages

```javascript
// Send message to another adapter
adapter.sendTo('telegram.0', 'send', {
    text: 'Hello from my adapter!',
    user: 'admin'
}, (result) => {
    if (result && result.sent) {
        adapter.log.info('Message sent successfully');
    } else {
        adapter.log.error('Failed to send message');
    }
});

// Async message sending
async function sendNotification(message) {
    try {
        const result = await adapter.sendToAsync('telegram.0', 'send', {
            text: message,
            user: 'admin'
        });
        
        if (result && result.sent) {
            adapter.log.info('Notification sent');
            return true;
        } else {
            adapter.log.warn('Notification not sent');
            return false;
        }
    } catch (error) {
        adapter.log.error(`Failed to send notification: ${error.message}`);
        return false;
    }
}
```

### Receiving Messages

```javascript
adapter.on('message', (obj) => {
    adapter.log.debug(`Received message: ${obj.command}`);
    
    switch (obj.command) {
        case 'getData':
            handleGetDataRequest(obj);
            break;
        case 'setConfig':
            handleSetConfigRequest(obj);
            break;
        case 'refresh':
            handleRefreshRequest(obj);
            break;
        case 'test':
            handleTestRequest(obj);
            break;
        default:
            adapter.log.warn(`Unknown command: ${obj.command}`);
            adapter.sendTo(obj.from, obj.command, { error: 'Unknown command' }, obj.callback);
    }
});

async function handleGetDataRequest(obj) {
    try {
        const data = await collectCurrentData();
        adapter.sendTo(obj.from, obj.command, { success: true, data: data }, obj.callback);
    } catch (error) {
        adapter.sendTo(obj.from, obj.command, { success: false, error: error.message }, obj.callback);
    }
}

async function handleSetConfigRequest(obj) {
    try {
        if (!obj.message || !obj.message.config) {
            throw new Error('Config data missing');
        }

        await updateConfiguration(obj.message.config);
        adapter.sendTo(obj.from, obj.command, { success: true }, obj.callback);
    } catch (error) {
        adapter.sendTo(obj.from, obj.command, { success: false, error: error.message }, obj.callback);
    }
}

function handleTestRequest(obj) {
    // Simple test response
    const testResult = {
        adapter: adapter.name,
        instance: adapter.instance,
        version: adapter.version,
        uptime: process.uptime(),
        timestamp: Date.now()
    };
    
    adapter.sendTo(obj.from, obj.command, { success: true, data: testResult }, obj.callback);
}
```

### Message Routing

```javascript
class MessageRouter {
    constructor(adapter) {
        this.adapter = adapter;
        this.handlers = new Map();
        this.setupMessageHandler();
    }

    setupMessageHandler() {
        this.adapter.on('message', (obj) => {
            this.routeMessage(obj);
        });
    }

    registerHandler(command, handler) {
        this.handlers.set(command, handler);
    }

    async routeMessage(obj) {
        const handler = this.handlers.get(obj.command);
        
        if (handler) {
            try {
                const result = await handler(obj.message, obj);
                this.sendResponse(obj, { success: true, data: result });
            } catch (error) {
                this.adapter.log.error(`Handler error for ${obj.command}: ${error.message}`);
                this.sendResponse(obj, { success: false, error: error.message });
            }
        } else {
            this.adapter.log.warn(`No handler for command: ${obj.command}`);
            this.sendResponse(obj, { success: false, error: 'Unknown command' });
        }
    }

    sendResponse(obj, response) {
        if (obj.callback) {
            this.adapter.sendTo(obj.from, obj.command, response, obj.callback);
        }
    }
}

// Usage
const messageRouter = new MessageRouter(adapter);

messageRouter.registerHandler('getStatus', async (message) => {
    return {
        connected: adapter.connected,
        devices: adapter.deviceCount,
        lastUpdate: adapter.lastUpdateTime
    };
});

messageRouter.registerHandler('restart', async (message) => {
    adapter.log.info('Restart requested via message');
    adapter.restart();
    return { restarting: true };
});
```

## WebSocket Communication

### WebSocket Client

```javascript
const WebSocket = require('ws');

class WebSocketClient {
    constructor(adapter, url) {
        this.adapter = adapter;
        this.url = url;
        this.ws = null;
        this.reconnectInterval = 5000;
        this.reconnectAttempts = 0;
        this.maxReconnectAttempts = 10;
        this.messageHandlers = new Map();
    }

    connect() {
        try {
            this.ws = new WebSocket(this.url);
            this.setupEventHandlers();
        } catch (error) {
            this.adapter.log.error(`WebSocket connection failed: ${error.message}`);
            this.scheduleReconnect();
        }
    }

    setupEventHandlers() {
        this.ws.on('open', () => {
            this.adapter.log.info('WebSocket connected');
            this.reconnectAttempts = 0;
            this.onConnected();
        });

        this.ws.on('message', (data) => {
            try {
                const message = JSON.parse(data.toString());
                this.handleMessage(message);
            } catch (error) {
                this.adapter.log.error(`Failed to parse WebSocket message: ${error.message}`);
            }
        });

        this.ws.on('close', (code, reason) => {
            this.adapter.log.warn(`WebSocket closed: ${code} - ${reason}`);
            this.onDisconnected();
            this.scheduleReconnect();
        });

        this.ws.on('error', (error) => {
            this.adapter.log.error(`WebSocket error: ${error.message}`);
        });
    }

    handleMessage(message) {
        this.adapter.log.debug(`WebSocket message: ${message.type}`);
        
        const handler = this.messageHandlers.get(message.type);
        if (handler) {
            handler(message.data);
        } else {
            this.adapter.log.debug(`No handler for message type: ${message.type}`);
        }
    }

    onConnected() {
        // Send authentication or initial setup
        this.send('authenticate', {
            token: this.adapter.config.token,
            adapter: this.adapter.name
        });
    }

    onDisconnected() {
        // Update connection status
        this.adapter.setState('info.connection', false, true);
    }

    send(type, data) {
        if (this.ws && this.ws.readyState === WebSocket.OPEN) {
            const message = JSON.stringify({ type, data, timestamp: Date.now() });
            this.ws.send(message);
        } else {
            this.adapter.log.warn('Cannot send message: WebSocket not connected');
        }
    }

    scheduleReconnect() {
        if (this.reconnectAttempts < this.maxReconnectAttempts) {
            this.reconnectAttempts++;
            this.adapter.log.info(`Reconnecting in ${this.reconnectInterval}ms (attempt ${this.reconnectAttempts})`);
            
            setTimeout(() => {
                this.connect();
            }, this.reconnectInterval);
        } else {
            this.adapter.log.error('Max reconnection attempts reached');
        }
    }

    onMessage(type, handler) {
        this.messageHandlers.set(type, handler);
    }

    disconnect() {
        if (this.ws) {
            this.ws.close();
        }
    }
}

// Usage
const wsClient = new WebSocketClient(adapter, adapter.config.websocketUrl);

wsClient.onMessage('deviceUpdate', (data) => {
    adapter.log.info(`Device update: ${data.deviceId}`);
    updateDeviceState(data.deviceId, data.state);
});

wsClient.onMessage('notification', (data) => {
    adapter.log.info(`Notification: ${data.message}`);
    adapter.setState('notifications.latest', data.message, true);
});

wsClient.connect();
```

### WebSocket Server

```javascript
const WebSocket = require('ws');
const http = require('http');

class WebSocketServer {
    constructor(adapter, port) {
        this.adapter = adapter;
        this.port = port;
        this.server = null;
        this.wss = null;
        this.clients = new Set();
    }

    start() {
        this.server = http.createServer();
        this.wss = new WebSocket.Server({ server: this.server });

        this.wss.on('connection', (ws, request) => {
            this.adapter.log.info(`WebSocket client connected from ${request.socket.remoteAddress}`);
            this.handleNewClient(ws);
        });

        this.server.listen(this.port, () => {
            this.adapter.log.info(`WebSocket server listening on port ${this.port}`);
        });
    }

    handleNewClient(ws) {
        this.clients.add(ws);

        ws.on('message', (data) => {
            try {
                const message = JSON.parse(data.toString());
                this.handleClientMessage(ws, message);
            } catch (error) {
                this.adapter.log.error(`Invalid message from client: ${error.message}`);
            }
        });

        ws.on('close', () => {
            this.clients.delete(ws);
            this.adapter.log.info('WebSocket client disconnected');
        });

        ws.on('error', (error) => {
            this.adapter.log.error(`WebSocket client error: ${error.message}`);
            this.clients.delete(ws);
        });

        // Send welcome message
        this.sendToClient(ws, 'welcome', {
            server: this.adapter.name,
            version: this.adapter.version,
            timestamp: Date.now()
        });
    }

    handleClientMessage(ws, message) {
        this.adapter.log.debug(`Client message: ${message.type}`);

        switch (message.type) {
            case 'subscribe':
                this.handleSubscribe(ws, message.data);
                break;
            case 'unsubscribe':
                this.handleUnsubscribe(ws, message.data);
                break;
            case 'getState':
                this.handleGetState(ws, message.data);
                break;
            case 'setState':
                this.handleSetState(ws, message.data);
                break;
            default:
                this.sendToClient(ws, 'error', { message: 'Unknown message type' });
        }
    }

    async handleGetState(ws, data) {
        try {
            const state = await this.adapter.getStateAsync(data.id);
            this.sendToClient(ws, 'stateResponse', {
                id: data.id,
                state: state,
                requestId: data.requestId
            });
        } catch (error) {
            this.sendToClient(ws, 'error', {
                message: error.message,
                requestId: data.requestId
            });
        }
    }

    async handleSetState(ws, data) {
        try {
            await this.adapter.setStateAsync(data.id, data.value, data.ack || false);
            this.sendToClient(ws, 'stateSet', {
                id: data.id,
                success: true,
                requestId: data.requestId
            });
        } catch (error) {
            this.sendToClient(ws, 'error', {
                message: error.message,
                requestId: data.requestId
            });
        }
    }

    sendToClient(ws, type, data) {
        if (ws.readyState === WebSocket.OPEN) {
            const message = JSON.stringify({
                type: type,
                data: data,
                timestamp: Date.now()
            });
            ws.send(message);
        }
    }

    broadcast(type, data) {
        const message = JSON.stringify({
            type: type,
            data: data,
            timestamp: Date.now()
        });

        this.clients.forEach(client => {
            if (client.readyState === WebSocket.OPEN) {
                client.send(message);
            }
        });
    }

    stop() {
        if (this.wss) {
            this.wss.close();
        }
        if (this.server) {
            this.server.close();
        }
    }
}

// Usage
const wsServer = new WebSocketServer(adapter, adapter.config.websocketPort || 8080);
wsServer.start();

// Broadcast state changes to all clients
adapter.on('stateChange', (id, state) => {
    wsServer.broadcast('stateChange', { id, state });
});
```

## Message Handling

### Message Queue

```javascript
class MessageQueue {
    constructor(adapter) {
        this.adapter = adapter;
        this.queue = [];
        this.processing = false;
        this.maxQueueSize = 1000;
    }

    enqueue(message) {
        if (this.queue.length >= this.maxQueueSize) {
            this.adapter.log.warn('Message queue full, dropping oldest message');
            this.queue.shift();
        }

        this.queue.push({
            ...message,
            timestamp: Date.now(),
            id: this.generateMessageId()
        });

        this.processQueue();
    }

    async processQueue() {
        if (this.processing || this.queue.length === 0) {
            return;
        }

        this.processing = true;

        while (this.queue.length > 0) {
            const message = this.queue.shift();
            
            try {
                await this.processMessage(message);
            } catch (error) {
                this.adapter.log.error(`Failed to process message ${message.id}: ${error.message}`);
            }
        }

        this.processing = false;
    }

    async processMessage(message) {
        this.adapter.log.debug(`Processing message: ${message.type}`);
        
        switch (message.type) {
            case 'deviceUpdate':
                await this.handleDeviceUpdate(message.data);
                break;
            case 'notification':
                await this.handleNotification(message.data);
                break;
            case 'command':
                await this.handleCommand(message.data);
                break;
            default:
                this.adapter.log.warn(`Unknown message type: ${message.type}`);
        }
    }

    generateMessageId() {
        return Date.now().toString(36) + Math.random().toString(36).substr(2);
    }
}
```

## Network Utilities

### Network Discovery

```javascript
const dgram = require('dgram');

class NetworkDiscovery {
    constructor(adapter) {
        this.adapter = adapter;
        this.discoveryPort = 12345;
        this.broadcastAddress = '255.255.255.255';
        this.socket = null;
    }

    start() {
        this.socket = dgram.createSocket('udp4');

        this.socket.on('message', (msg, rinfo) => {
            this.handleDiscoveryMessage(msg, rinfo);
        });

        this.socket.on('error', (err) => {
            this.adapter.log.error(`Discovery socket error: ${err.message}`);
        });

        this.socket.bind(this.discoveryPort, () => {
            this.socket.setBroadcast(true);
            this.adapter.log.info(`Network discovery started on port ${this.discoveryPort}`);
        });
    }

    handleDiscoveryMessage(msg, rinfo) {
        try {
            const message = JSON.parse(msg.toString());
            
            if (message.type === 'device-announcement') {
                this.adapter.log.info(`Discovered device: ${message.deviceId} at ${rinfo.address}`);
                this.onDeviceDiscovered(message, rinfo);
            }
        } catch (error) {
            this.adapter.log.debug(`Invalid discovery message from ${rinfo.address}`);
        }
    }

    broadcast(message) {
        if (this.socket) {
            const buffer = Buffer.from(JSON.stringify(message));
            this.socket.send(buffer, 0, buffer.length, this.discoveryPort, this.broadcastAddress);
        }
    }

    onDeviceDiscovered(message, rinfo) {
        // Override in subclass
    }

    stop() {
        if (this.socket) {
            this.socket.close();
        }
    }
}
```

### Port Scanning

```javascript
const net = require('net');

class PortScanner {
    constructor(adapter) {
        this.adapter = adapter;
    }

    async scanPort(host, port, timeout = 3000) {
        return new Promise((resolve) => {
            const socket = new net.Socket();
            
            socket.setTimeout(timeout);
            
            socket.on('connect', () => {
                socket.destroy();
                resolve(true);
            });

            socket.on('timeout', () => {
                socket.destroy();
                resolve(false);
            });

            socket.on('error', () => {
                resolve(false);
            });

            socket.connect(port, host);
        });
    }

    async scanRange(host, startPort, endPort, timeout = 1000) {
        const openPorts = [];
        const promises = [];

        for (let port = startPort; port <= endPort; port++) {
            promises.push(
                this.scanPort(host, port, timeout).then(isOpen => {
                    if (isOpen) {
                        openPorts.push(port);
                    }
                })
            );
        }

        await Promise.all(promises);
        return openPorts.sort((a, b) => a - b);
    }

    async findDevices(network, ports, timeout = 1000) {
        const devices = [];
        const baseIp = network.split('.').slice(0, 3).join('.');
        
        const promises = [];
        
        for (let i = 1; i <= 254; i++) {
            const host = `${baseIp}.${i}`;
            
            for (const port of ports) {
                promises.push(
                    this.scanPort(host, port, timeout).then(isOpen => {
                        if (isOpen) {
                            devices.push({ host, port });
                        }
                    })
                );
            }
        }

        await Promise.all(promises);
        return devices;
    }
}

// Usage
const scanner = new PortScanner(adapter);

// Scan for devices on common ports
const commonPorts = [80, 8080, 443, 8443, 502, 1883];
const devices = await scanner.findDevices('192.168.1', commonPorts);
adapter.log.info(`Found ${devices.length} devices`);
```

## Security Considerations

### Secure HTTP Requests

```javascript
const https = require('https');
const fs = require('fs');

class SecureHttpClient {
    constructor(adapter) {
        this.adapter = adapter;
        
        // Configure TLS options
        this.tlsOptions = {
            rejectUnauthorized: process.env.NODE_ENV === 'production',
            minVersion: 'TLSv1.2',
            ciphers: 'ECDHE+AESGCM:ECDHE+CHACHA20:DHE+AESGCM:DHE+CHACHA20:!aNULL:!MD5:!DSS'
        };
    }

    async secureRequest(url, options = {}) {
        const urlObj = new URL(url);
        
        if (urlObj.protocol !== 'https:') {
            throw new Error('Only HTTPS requests are allowed');
        }

        // Merge security options
        const requestOptions = {
            ...this.tlsOptions,
            ...options,
            headers: {
                'User-Agent': `ioBroker-${this.adapter.name}/${this.adapter.version}`,
                ...options.headers
            }
        };

        // Add client certificate if configured
        if (this.adapter.config.clientCert && this.adapter.config.clientKey) {
            requestOptions.cert = fs.readFileSync(this.adapter.config.clientCert);
            requestOptions.key = fs.readFileSync(this.adapter.config.clientKey);
        }

        // Make request
        return new Promise((resolve, reject) => {
            const req = https.request(url, requestOptions, (res) => {
                let data = '';
                
                res.on('data', (chunk) => {
                    data += chunk;
                });

                res.on('end', () => {
                    resolve({
                        statusCode: res.statusCode,
                        headers: res.headers,
                        data: data
                    });
                });
            });

            req.on('error', reject);
            req.end();
        });
    }
}
```

### API Key Management

```javascript
class ApiKeyManager {
    constructor(adapter) {
        this.adapter = adapter;
        this.apiKeys = new Map();
    }

    async loadApiKeys() {
        try {
            const data = await this.adapter.readFileAsync(
                `${this.adapter.name}.admin`, 
                'api-keys.json'
            );
            
            if (data) {
                const keys = JSON.parse(data.toString());
                Object.entries(keys).forEach(([name, key]) => {
                    this.apiKeys.set(name, this.adapter.decrypt(key));
                });
            }
        } catch (error) {
            this.adapter.log.debug('No API keys file found');
        }
    }

    async saveApiKeys() {
        const encryptedKeys = {};
        
        this.apiKeys.forEach((key, name) => {
            encryptedKeys[name] = this.adapter.encrypt(key);
        });

        await this.adapter.writeFileAsync(
            `${this.adapter.name}.admin`,
            'api-keys.json',
            JSON.stringify(encryptedKeys, null, 2)
        );
    }

    setApiKey(name, key) {
        this.apiKeys.set(name, key);
    }

    getApiKey(name) {
        return this.apiKeys.get(name);
    }

    removeApiKey(name) {
        return this.apiKeys.delete(name);
    }

    getAuthHeaders(keyName) {
        const apiKey = this.getApiKey(keyName);
        if (!apiKey) {
            throw new Error(`API key '${keyName}' not found`);
        }

        return {
            'Authorization': `Bearer ${apiKey}`,
            'X-API-Key': apiKey
        };
    }
}

// Usage
const keyManager = new ApiKeyManager(adapter);
await keyManager.loadApiKeys();

keyManager.setApiKey('weather-api', adapter.config.weatherApiKey);
await keyManager.saveApiKeys();

const headers = keyManager.getAuthHeaders('weather-api');
```

<!-- 
Source metadata for automated updates:
- Primary source: ioBroker.js-controller/packages/adapter/src/lib/adapter/adapter.ts
- HTTP client methods: sendTo, sendToAsync, sendToHost
- WebSocket patterns from ioBroker web and socket adapters
- Security practices from ioBroker security documentation
- Network utilities from Node.js core modules
- Message handling from ioBroker messaging system
- Last updated: 2024-12-22
-->