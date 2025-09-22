# io-package.json Schema Reference

This document provides a complete reference for the `io-package.json` file structure, which defines adapter metadata, configuration, and capabilities.

## Table of Contents

- [Overview](#overview)
- [Root Properties](#root-properties)
- [Common Section](#common-section)
- [Native Section](#native-section)
- [Objects Section](#objects-section)
- [Instance Objects](#instance-objects)
- [Notifications](#notifications)
- [Examples](#examples)

## Overview

The `io-package.json` file is the main configuration file for ioBroker adapters. It defines:
- Adapter metadata and capabilities
- Configuration schema
- Objects created by the adapter
- Permission requirements
- UI configuration

### Basic Structure

```json
{
  "$schema": "https://raw.githubusercontent.com/ioBroker/ioBroker.js-controller/master/schemas/io-package.json",
  "common": {
    // Common adapter properties
  },
  "native": {
    // Default configuration
  },
  "objects": [
    // Global objects
  ],
  "instanceObjects": [
    // Per-instance objects
  ],
  "protectedNative": [
    // Protected configuration keys
  ],
  "encryptedNative": [
    // Encrypted configuration keys
  ],
  "notifications": [
    // Notification definitions
  ]
}
```

## Root Properties

### $schema
JSON schema reference for validation.

```json
{
  "$schema": "https://raw.githubusercontent.com/ioBroker/ioBroker.js-controller/master/schemas/io-package.json"
}
```

## Common Section

The `common` section contains all standard adapter properties.

### Required Properties

#### name
Adapter name without "ioBroker" prefix.

```json
{
  "common": {
    "name": "my-adapter"
  }
}
```

**Rules:**
- Must not start with "iobroker" (case insensitive)
- Should be lowercase with hyphens

#### version
Current adapter version using semantic versioning.

```json
{
  "common": {
    "version": "1.2.3"
  }
}
```

**Format:** `MAJOR.MINOR.PATCH[-PRERELEASE][+BUILD]`

#### platform
Target platform (always "Javascript/Node.js" for Node.js adapters).

```json
{
  "common": {
    "platform": "Javascript/Node.js"
  }
}
```

#### tier
Startup tier for dependency ordering.

```json
{
  "common": {
    "tier": 1
  }
}
```

**Values:**
- `1`: Logic adapters (highest priority)
- `2`: APIs and data processing
- `3`: Visualization and backup (default)

#### titleLang
Multi-language adapter title.

```json
{
  "common": {
    "titleLang": {
      "en": "My Adapter",
      "de": "Mein Adapter",
      "ru": "Мой адаптер",
      "pt": "Meu adaptador",
      "nl": "Mijn adapter",
      "fr": "Mon adaptateur",
      "it": "Il mio adattatore",
      "es": "Mi adaptador",
      "pl": "Mój adapter",
      "uk": "Мій адаптер",
      "zh-cn": "我的适配器"
    }
  }
}
```

**Required:** Must include "en" (English)

#### news
Changelog in multiple languages.

```json
{
  "common": {
    "news": {
      "1.2.3": {
        "en": "Bug fixes and improvements",
        "de": "Fehlerbehebungen und Verbesserungen"
      },
      "1.2.2": {
        "en": "Added new features",
        "de": "Neue Funktionen hinzugefügt"
      }
    }
  }
}
```

**Rules:**
- Keys must be valid semver versions
- Maximum 20 versions
- Each version requires "en" (English)

#### desc
Multi-language description.

```json
{
  "common": {
    "desc": {
      "en": "Adapter for connecting to my device",
      "de": "Adapter für die Verbindung zu meinem Gerät"
    }
  }
}
```

#### mode
Execution mode for the adapter.

```json
{
  "common": {
    "mode": "daemon"
  }
}
```

**Values:**
- `daemon`: Always running
- `schedule`: Scheduled execution
- `once`: Single execution
- `none`: No automatic execution
- `extension`: Extension mode

#### licenseInformation
License information object.

```json
{
  "common": {
    "licenseInformation": {
      "type": "free",
      "license": "MIT"
    }
  }
}
```

**Properties:**
- `type`: "free", "paid", "commercial", "limited"
- `license`: SPDX license identifier
- `link`: URL for license information (required for non-free)

### Optional Properties

#### authors
Array of author information.

```json
{
  "common": {
    "authors": [
      "John Doe <john@example.com>",
      {
        "name": "Jane Smith",
        "email": "jane@example.com"
      }
    ]
  }
}
```

#### keywords
Search keywords for the adapter.

```json
{
  "common": {
    "keywords": ["temperature", "sensor", "monitoring"]
  }
}
```

#### readme
URL to README file.

```json
{
  "common": {
    "readme": "https://github.com/user/ioBroker.my-adapter/blob/main/README.md"
  }
}
```

#### type
Adapter category classification.

```json
{
  "common": {
    "type": "climate-control"
  }
}
```

**Categories:**
- `alarm`, `climate-control`, `communication`, `date-and-time`
- `energy`, `garden`, `general`, `geoposition`, `hardware`
- `health`, `household`, `infrastructure`, `iot-systems`
- `lighting`, `logic`, `messaging`, `metering`, `misc-data`
- `multimedia`, `network`, `protocols`, `storage`, `utility`
- `vehicle`, `visualization`, `visualization-icons`
- `visualization-widgets`, `weather`

#### connectionType
Type of device connection.

```json
{
  "common": {
    "connectionType": "local"
  }
}
```

**Values:**
- `none`: No external connection
- `local`: Local network connection  
- `cloud`: Internet/cloud connection

#### dataSource
How data is received.

```json
{
  "common": {
    "dataSource": "poll"
  }
}
```

**Values:**
- `none`: No data source
- `poll`: Polling from device
- `push`: Push from device
- `assumption`: Calculated/assumed data

#### adminUI
Admin UI configuration.

```json
{
  "common": {
    "adminUI": {
      "config": "json",
      "tab": "html",
      "custom": "json"
    }
  }
}
```

**Properties:**
- `config`: Configuration page type ("json", "html", "materialize", "none")
- `tab`: Admin tab type ("json", "html", "materialize")
- `custom`: Custom settings type ("json")

#### supportedMessages
Supported message types.

```json
{
  "common": {
    "supportedMessages": {
      "custom": true,
      "getHistory": true,
      "stopInstance": 5000,
      "deviceManager": true,
      "notifications": true
    }
  }
}
```

**Properties:**
- `custom`: Custom messages (legacy messagebox)
- `getHistory`: History data queries
- `stopInstance`: Graceful stop (boolean or timeout in ms)
- `deviceManager`: Device manager support
- `notifications`: Notification handling

#### dependencies
Required adapters on same host.

```json
{
  "common": {
    "dependencies": [
      {"js-controller": ">=4.0.0"}
    ]
  }
}
```

#### globalDependencies
Required adapters on any host.

```json
{
  "common": {
    "globalDependencies": [
      {"admin": ">=5.0.0"}
    ]
  }
}
```

#### compact
Compact mode support.

```json
{
  "common": {
    "compact": true
  }
}
```

#### enabled
Default enabled state for new instances.

```json
{
  "common": {
    "enabled": false
  }
}
```

#### schedule
CRON schedule for schedule mode.

```json
{
  "common": {
    "schedule": "0 */6 * * *"
  }
}
```

#### loglevel
Default log level.

```json
{
  "common": {
    "loglevel": "info"
  }
}
```

**Values:** "silly", "debug", "info", "warn", "error"

#### singleton
Only one instance allowed globally.

```json
{
  "common": {
    "singleton": true
  }
}
```

#### singletonHost
Only one instance per host.

```json
{
  "common": {
    "singletonHost": true
  }
}
```

#### icon
Local icon file path.

```json
{
  "common": {
    "icon": "my-adapter.png"
  }
}
```

#### extIcon
External icon URL for repository.

```json
{
  "common": {
    "extIcon": "https://raw.githubusercontent.com/user/ioBroker.my-adapter/main/admin/my-adapter.png"
  }
}
```

#### localLinks
Links to adapter web interfaces.

```json
{
  "common": {
    "localLinks": {
      "_default": {
        "link": "http://%ip%:%port%",
        "name": {
          "en": "Web Interface",
          "de": "Web-Oberfläche"
        },
        "icon": "fa-home",
        "color": "blue",
        "intro": true,
        "order": 1
      }
    }
  }
}
```

#### docs
Documentation links.

```json
{
  "common": {
    "docs": {
      "en": "docs/en/README.md",
      "de": ["docs/de/README.md", "docs/de/advanced.md"]
    }
  }
}
```

#### adminTab
Admin tab configuration.

```json
{
  "common": {
    "adminTab": {
      "name": {
        "en": "My Adapter Tab",
        "de": "Mein Adapter Tab"
      },
      "link": "/adapter/my-adapter/tab.html",
      "fa-icon": "fa-cog",
      "singleton": true,
      "ignoreConfigUpdate": false
    }
  }
}
```

#### adminColumns
Custom columns in admin object browser.

```json
{
  "common": {
    "adminColumns": [
      {
        "name": {
          "en": "Address",
          "de": "Adresse"
        },
        "path": "native.address",
        "width": 100,
        "align": "left"
      },
      {
        "name": "Type",
        "path": "native.type",
        "width": 80,
        "align": "center",
        "type": "select",
        "edit": true,
        "objTypes": ["state"]
      }
    ]
  }
}
```

#### messages
Installation/update messages.

```json
{
  "common": {
    "messages": [
      {
        "condition": {
          "operand": "and",
          "rules": ["oldVersion<2.0.0", "newVersion>=2.0.0"]
        },
        "title": {
          "en": "Breaking Changes",
          "de": "Breaking Changes"
        },
        "text": {
          "en": "This version contains breaking changes. Please review the changelog.",
          "de": "Diese Version enthält Breaking Changes. Bitte prüfen Sie das Changelog."
        },
        "level": "warn",
        "buttons": ["ok", "cancel"]
      }
    ]
  }
}
```

#### automaticUpgrade
Automatic update policy.

```json
{
  "common": {
    "automaticUpgrade": "patch"
  }
}
```

**Values:**
- `none`: No automatic updates (recommended)
- `patch`: Patch version updates
- `minor`: Minor version updates  
- `major`: Major version updates

#### blockedVersions
Blocked version ranges.

```json
{
  "common": {
    "blockedVersions": [
      "1.0.0",
      ">=1.1.0 <1.1.5"
    ]
  }
}
```

#### restartAdapters
Adapters to restart after installation.

```json
{
  "common": {
    "restartAdapters": ["web", "admin"]
  }
}
```

#### dataFolder
Custom data folder path.

```json
{
  "common": {
    "dataFolder": "my-adapter-data"
  }
}
```

#### stopTimeout
Stop timeout in milliseconds.

```json
{
  "common": {
    "stopTimeout": 5000
  }
}
```

#### preserveSettings
Settings to preserve during updates.

```json
{
  "common": {
    "preserveSettings": ["history", "custom"]
  }
}
```

#### osDependencies
OS package dependencies.

```json
{
  "common": {
    "osDependencies": {
      "linux": ["python3", "build-essential"],
      "darwin": ["python3"],
      "win32": []
    }
  }
}
```

#### os
Supported operating systems.

```json
{
  "common": {
    "os": ["linux", "darwin", "win32"]
  }
}
```

#### visWidgets
Vis 2.0 widget definitions.

```json
{
  "common": {
    "visWidgets": {
      "myWidgets": {
        "name": "My Custom Widgets",
        "url": "widgets/my-widgets.js",
        "components": ["MyWidget", "AnotherWidget"],
        "i18n": true,
        "bundlerType": "module"
      }
    }
  }
}
```

#### visIconSets
Vis 2.0 icon set definitions.

```json
{
  "common": {
    "visIconSets": {
      "myIcons": {
        "name": {
          "en": "My Icons",
          "de": "Meine Icons"
        },
        "url": "icons/my-icons.json",
        "icon": "data:image/svg+xml;base64,..."
      }
    }
  }
}
```

## Native Section

The `native` section defines the default configuration schema for adapter instances.

```json
{
  "native": {
    "hostname": "localhost",
    "port": 80,
    "username": "",
    "password": "",
    "enabled": true,
    "interval": 60000,
    "advanced": {
      "timeout": 5000,
      "retries": 3
    }
  }
}
```

**Guidelines:**
- Use meaningful default values
- Include all configurable options
- Use appropriate data types
- Group related settings in objects

### Configuration Examples

#### Network Settings
```json
{
  "native": {
    "hostname": "192.168.1.100",
    "port": 8080,
    "secure": false,
    "timeout": 5000
  }
}
```

#### Authentication
```json
{
  "native": {
    "username": "",
    "password": "",
    "apiKey": "",
    "token": ""
  }
}
```

#### Polling Configuration
```json
{
  "native": {
    "pollingInterval": 30000,
    "enabled": true,
    "sensors": [
      {
        "id": "temp1",
        "name": "Temperature Sensor 1",
        "type": "temperature",
        "enabled": true
      }
    ]
  }
}
```

## Objects Section

Global objects created once during adapter installation.

```json
{
  "objects": [
    {
      "_id": "info",
      "type": "channel",
      "common": {
        "name": "Information"
      },
      "native": {}
    },
    {
      "_id": "info.connection",
      "type": "state",
      "common": {
        "role": "indicator.connected",
        "name": "Device connected",
        "type": "boolean",
        "read": true,
        "write": false,
        "def": false
      },
      "native": {}
    }
  ]
}
```

### Object Types

#### Channel Objects
```json
{
  "_id": "devices",
  "type": "channel",
  "common": {
    "name": {
      "en": "Devices",
      "de": "Geräte"
    },
    "desc": {
      "en": "Connected devices",
      "de": "Verbundene Geräte"
    }
  },
  "native": {}
}
```

#### State Objects
```json
{
  "_id": "status.temperature",
  "type": "state",
  "common": {
    "name": "Temperature",
    "type": "number",
    "role": "value.temperature",
    "unit": "°C",
    "min": -50,
    "max": 100,
    "read": true,
    "write": false,
    "def": 0
  },
  "native": {}
}
```

#### Device Objects
```json
{
  "_id": "devices.sensor1",
  "type": "device",
  "common": {
    "name": "Temperature Sensor 1",
    "statusStates": {
      "onlineId": "devices.sensor1.info.connection"
    }
  },
  "native": {
    "address": "192.168.1.50",
    "model": "TempSensor-Pro"
  }
}
```

## Instance Objects

Objects created for each adapter instance.

```json
{
  "instanceObjects": [
    {
      "_id": "info",
      "type": "channel",
      "common": {
        "name": "Information"
      },
      "native": {}
    },
    {
      "_id": "info.connection",
      "type": "state", 
      "common": {
        "role": "indicator.connected",
        "name": "Connected to device",
        "type": "boolean",
        "read": true,
        "write": false,
        "def": false
      },
      "native": {}
    }
  ]
}
```

## Protected Native

Configuration keys that are only accessible by the adapter itself.

```json
{
  "protectedNative": [
    "internalKey",
    "systemConfig",
    "deviceSecrets"
  ]
}
```

## Encrypted Native

Configuration keys that are automatically encrypted/decrypted.

```json
{
  "encryptedNative": [
    "password",
    "apiKey",
    "token",
    "privateKey"
  ]
}
```

## Notifications

Built-in notification system configuration.

```json
{
  "notifications": [
    {
      "scope": "my-adapter",
      "name": {
        "en": "My Adapter Notifications",
        "de": "Mein Adapter Benachrichtigungen"
      },
      "description": {
        "en": "Notifications from my adapter",
        "de": "Benachrichtigungen von meinem Adapter"
      },
      "categories": [
        {
          "category": "connection",
          "name": {
            "en": "Connection Issues",
            "de": "Verbindungsprobleme"
          },
          "severity": "alert",
          "description": {
            "en": "Problems connecting to device",
            "de": "Probleme bei der Verbindung zum Gerät"
          },
          "regex": [
            "Connection failed.*",
            "Device not reachable.*"
          ],
          "limit": 5
        }
      ]
    }
  ]
}
```

## Examples

### Complete Basic Adapter

```json
{
  "$schema": "https://raw.githubusercontent.com/ioBroker/ioBroker.js-controller/master/schemas/io-package.json",
  "common": {
    "name": "my-device",
    "version": "1.0.0",
    "platform": "Javascript/Node.js",
    "tier": 2,
    "titleLang": {
      "en": "My Device Adapter",
      "de": "Mein Geräte-Adapter"
    },
    "news": {
      "1.0.0": {
        "en": "Initial release",
        "de": "Erste Veröffentlichung"
      }
    },
    "desc": {
      "en": "Adapter for connecting to my device",
      "de": "Adapter für die Verbindung zu meinem Gerät"
    },
    "mode": "daemon",
    "licenseInformation": {
      "type": "free",
      "license": "MIT"
    },
    "authors": ["John Doe <john@example.com>"],
    "keywords": ["device", "iot", "sensor"],
    "type": "hardware",
    "connectionType": "local",
    "dataSource": "poll",
    "adminUI": {
      "config": "json"
    },
    "supportedMessages": {
      "custom": true,
      "stopInstance": true
    },
    "dependencies": [
      {"js-controller": ">=4.0.0"}
    ],
    "compact": true,
    "enabled": false,
    "icon": "my-device.png",
    "loglevel": "info"
  },
  "native": {
    "hostname": "192.168.1.100",
    "port": 8080,
    "username": "",
    "password": "",
    "interval": 30000,
    "enabled": true
  },
  "objects": [
    {
      "_id": "info",
      "type": "channel",
      "common": {
        "name": "Information"
      },
      "native": {}
    }
  ],
  "instanceObjects": [
    {
      "_id": "info.connection",
      "type": "state",
      "common": {
        "role": "indicator.connected",
        "name": "Connected to device",
        "type": "boolean",
        "read": true,
        "write": false,
        "def": false
      },
      "native": {}
    }
  ],
  "encryptedNative": [
    "password"
  ]
}
```

### Advanced Adapter with Web Interface

```json
{
  "$schema": "https://raw.githubusercontent.com/ioBroker/ioBroker.js-controller/master/schemas/io-package.json",
  "common": {
    "name": "advanced-adapter",
    "version": "2.1.0",
    "platform": "Javascript/Node.js", 
    "tier": 2,
    "titleLang": {
      "en": "Advanced Adapter",
      "de": "Erweiterter Adapter"
    },
    "news": {
      "2.1.0": {
        "en": "Added web interface and device manager",
        "de": "Web-Interface und Geräte-Manager hinzugefügt"
      }
    },
    "desc": {
      "en": "Advanced adapter with web interface",
      "de": "Erweiterter Adapter mit Web-Interface"
    },
    "mode": "daemon",
    "licenseInformation": {
      "type": "free",
      "license": "Apache-2.0"
    },
    "authors": [
      {
        "name": "Jane Smith",
        "email": "jane@example.com"
      }
    ],
    "keywords": ["advanced", "web", "api"],
    "type": "infrastructure",
    "connectionType": "local",
    "dataSource": "push",
    "adminUI": {
      "config": "materialize",
      "tab": "html",
      "custom": "json"
    },
    "supportedMessages": {
      "custom": true,
      "deviceManager": true,
      "stopInstance": 10000
    },
    "dependencies": [
      {"js-controller": ">=5.0.0"}
    ],
    "globalDependencies": [
      {"admin": ">=6.0.0"}
    ],
    "compact": true,
    "enabled": false,
    "icon": "advanced-adapter.png",
    "localLinks": {
      "_default": {
        "link": "http://%ip%:%port%",
        "name": {
          "en": "Web Interface",
          "de": "Web-Interface"
        },
        "icon": "fa-home",
        "color": "blue",
        "intro": true,
        "order": 1
      },
      "api": {
        "link": "http://%ip%:%port%/api",
        "name": {
          "en": "API Documentation",
          "de": "API-Dokumentation"
        },
        "icon": "fa-code",
        "color": "green"
      }
    },
    "adminTab": {
      "name": {
        "en": "Device Manager",
        "de": "Geräte-Manager"
      },
      "link": "/adapter/advanced-adapter/tab.html",
      "fa-icon": "fa-cogs",
      "singleton": false
    },
    "webExtendable": true,
    "loglevel": "info",
    "tier": 2
  },
  "native": {
    "port": 8080,
    "secure": false,
    "devices": [],
    "apiEnabled": true,
    "logRequests": false
  },
  "objects": [
    {
      "_id": "info",
      "type": "channel", 
      "common": {
        "name": "Information"
      },
      "native": {}
    },
    {
      "_id": "api",
      "type": "channel",
      "common": {
        "name": "API"
      },
      "native": {}
    }
  ],
  "instanceObjects": [
    {
      "_id": "info.connection",
      "type": "state",
      "common": {
        "role": "indicator.connected",
        "name": "API Server running",
        "type": "boolean",
        "read": true,
        "write": false,
        "def": false
      },
      "native": {}
    },
    {
      "_id": "api.requestCount",
      "type": "state",
      "common": {
        "role": "value",
        "name": "API Request Count",
        "type": "number",
        "read": true,
        "write": false,
        "def": 0
      },
      "native": {}
    }
  ]
}
```

<!-- 
Source metadata for automated updates:
- Primary source: ioBroker.js-controller/schemas/io-package.json (SHA: 074d72331723c7722b3fa53cd732410a87141866)
- Complete schema with 4200+ lines covering all properties
- Examples derived from real-world adapter configurations
- Property descriptions from schema documentation
- Last updated: 2024-12-22
-->