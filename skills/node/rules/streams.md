---
name: streams
description: Working with Node.js streams
metadata:
  tags: streams, readable, writable, transform, pipeline
---

# Node.js Streams

## Use pipeline for Stream Composition

Always use `pipeline` instead of `.pipe()` for proper error handling:

```javascript
const { pipeline } = require('node:stream/promises');
const { createReadStream, createWriteStream } = require('node:fs');
const { createGzip } = require('node:zlib');

async function compressFile(input, output) {
  await pipeline(
    createReadStream(input),
    createGzip(),
    createWriteStream(output)
  );
}
```

### Async Generators in Pipeline

Use async generators for transformation:

```javascript
const { pipeline } = require('node:stream/promises');
const { createReadStream, createWriteStream } = require('node:fs');

async function* toUpperCase(source) {
  for await (const chunk of source) {
    yield chunk.toString().toUpperCase();
  }
}

async function processFile(input, output) {
  await pipeline(
    createReadStream(input),
    toUpperCase,
    createWriteStream(output)
  );
}
```

### Multiple Transformations

Chain multiple async generators:

```javascript
const { pipeline } = require('node:stream/promises');

async function* parseLines(source) {
  let buffer = '';
  for await (const chunk of source) {
    buffer += chunk.toString();
    const lines = buffer.split('\n');
    buffer = lines.pop() ?? '';
    for (const line of lines) {
      yield line;
    }
  }
  if (buffer) yield buffer;
}

async function* filterNonEmpty(source) {
  for await (const line of source) {
    if (line.trim()) {
      yield line + '\n';
    }
  }
}

await pipeline(
  createReadStream('input.txt'),
  parseLines,
  filterNonEmpty,
  createWriteStream('output.txt')
);
```

## Async Iterators with Streams

Use async iterators for consuming streams:

```javascript
const { createReadStream } = require('node:fs');
const { createInterface } = require('node:readline');

async function processLines(filePath) {
  const fileStream = createReadStream(filePath);
  const rl = createInterface({
    input: fileStream,
    crlfDelay: Infinity,
  });

  for await (const line of rl) {
    await processLine(line);
  }
}
```

## Readable.from for Creating Streams

Create readable streams from iterables:

```javascript
const { Readable } = require('node:stream');

async function* generateData() {
  for (let i = 0; i < 100; i++) {
    yield JSON.stringify({ id: i, timestamp: Date.now() }) + '\n';
  }
}

const stream = Readable.from(generateData());
```

## Backpressure Handling

Respect backpressure signals using `once` from events:

```javascript
const { Writable } = require('node:stream');
const { once } = require('node:events');

async function writeData(writable, data) {
  for (const chunk of data) {
    const canContinue = writable.write(chunk);
    if (!canContinue) {
      await once(writable, 'drain');
    }
  }
}
```

## Stream Consumers (Node.js 18+)

Use stream consumers for common operations:

```javascript
const { text, json, buffer } = require('node:stream/consumers');
const { Readable } = require('node:stream');

async function readStreamAsJson(stream) {
  return json(stream);
}

async function readStreamAsText(stream) {
  return text(stream);
}
```
