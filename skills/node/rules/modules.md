---
name: modules
description: CommonJS module patterns
metadata:
  tags: modules, commonjs, require, exports
---

# Node.js Modules

## Use CommonJS

Use CommonJS (`require`/`module.exports`) for all modules:

```javascript
// Named exports (preferred)
function processData(data) {
  // ...
}

const CONFIG = {
  timeout: 5000,
};

module.exports = { processData, CONFIG };

// Require modules
const { processData, CONFIG } = require('./utils');
```

## Require Built-in Modules

Always use the `node:` prefix for built-in modules:

```javascript
const { createServer } = require('node:http');
const { join } = require('node:path');
const { readFile } = require('node:fs/promises');
```

## Barrel Exports

Use index files to simplify requires:

```javascript
// src/utils/index.js
const { formatDate, parseDate } = require('./date');
const { formatCurrency } = require('./currency');
const { validateEmail } = require('./validation');

module.exports = { formatDate, parseDate, formatCurrency, validateEmail };

// Consumer
const { formatDate, formatCurrency } = require('./utils');
```

## Module Pattern

Prefer named exports for better clarity:

```javascript
// GOOD - named exports
function createServer(config) {
  // ...
}

function createClient(config) {
  // ...
}

module.exports = { createServer, createClient };

// AVOID - single default-like export when you have multiple
module.exports = createServer;
```

## Dynamic Requires

Use dynamic require for conditional loading:

```javascript
function loadPlugin(name) {
  const plugin = require(`./plugins/${name}`);
  return plugin;
}
```

## __dirname and __filename

Use `__dirname` and `__filename` which are natively available in CommonJS:

```javascript
const { join } = require('node:path');

const configPath = join(__dirname, 'config.json');
const currentFile = __filename;
```
