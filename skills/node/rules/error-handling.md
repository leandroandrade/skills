---
name: error-handling
description: Error handling patterns in Node.js
metadata:
  tags: errors, exceptions, try-catch, error-handling
---

# Error Handling in Node.js

## Custom Errors with @fastify/create-error

Use [@fastify/create-error](https://github.com/fastify/fastify-error) for creating custom errors with codes:

```javascript
const createError = require('@fastify/create-error');

const NotFoundError = createError('NOT_FOUND', '%s not found', 404);
const ValidationError = createError('VALIDATION_ERROR', '%s', 400);
const DatabaseError = createError('DATABASE_ERROR', 'Database operation failed: %s', 500);

// Usage
throw new NotFoundError('User');
throw new ValidationError('Email is required');
```

## Minimal Error Code Implementation

If you prefer no dependencies, use this minimal pattern:

```javascript
function createAppError(message, options) {
  const error = new Error(message, { cause: options.cause });
  error.code = options.code;
  error.statusCode = options.statusCode ?? 500;
  Error.captureStackTrace(error, createAppError);
  return error;
}

// Factory functions for common errors
function notFound(resource) {
  return createAppError(`${resource} not found`, { code: 'NOT_FOUND', statusCode: 404 });
}

function validationError(message) {
  return createAppError(message, { code: 'VALIDATION_ERROR', statusCode: 400 });
}

function databaseError(message, cause) {
  return createAppError(message, { code: 'DATABASE_ERROR', statusCode: 500, cause });
}

// Usage
throw notFound('User');
throw validationError('Email is required');
```

## Checking Error Codes

Check errors by code, not by class:

```javascript
function isAppError(error) {
  return error instanceof Error && 'code' in error && 'statusCode' in error;
}

try {
  await fetchUser(id);
} catch (error) {
  if (isAppError(error) && error.code === 'NOT_FOUND') {
    return null;
  }
  throw error;
}
```

## Async Error Handling

Always use try-catch with async/await and propagate errors properly:

```javascript
async function fetchUser(id) {
  try {
    const user = await db.users.findById(id);
    if (!user) {
      throw notFound('User');
    }
    return user;
  } catch (error) {
    if (isAppError(error)) {
      throw error;
    }
    throw databaseError('Failed to fetch user', error);
  }
}
```

## Unhandled Rejections and Exceptions

Do not handle `unhandledRejection` and `uncaughtException` manually. Use [close-with-grace](https://github.com/fastify/close-with-grace) which handles these automatically and triggers graceful shutdown.

See [graceful-shutdown.md](./graceful-shutdown.md) for proper shutdown handling.

## Fastify Error Handling

Fastify has built-in error handling:

```javascript
const Fastify = require('fastify');

const app = Fastify({ logger: true });

app.setErrorHandler((error, request, reply) => {
  const statusCode = error.statusCode ?? 500;
  const code = error.code ?? 'INTERNAL_ERROR';

  request.log.error(error);

  reply.status(statusCode).send({
    error: {
      code,
      message: error.message,
    },
  });
});
```

## Never Swallow Errors

Never use empty catch blocks that hide errors:

```javascript
// BAD - error is swallowed
try {
  await riskyOperation();
} catch (error) {
  // Do nothing
}

// GOOD - handle or re-throw
try {
  await riskyOperation();
} catch (error) {
  logger.error({ err: error }, 'Operation failed');
  throw error;
}
```

## Error Cause Chain

Use the `cause` option to preserve error chains:

```javascript
try {
  await externalService.call();
} catch (error) {
  throw new Error('Service call failed', { cause: error });
}
```
