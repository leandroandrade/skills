---
name: schemas
description: JSON Schema validation in Fastify with TypeBox
metadata:
  tags: validation, json-schema, schemas, ajv, typebox
---

# JSON Schema Validation

## Use TypeBox for Schemas

**Prefer TypeBox for defining schemas.** It provides compile-time validation and compiles to JSON Schema:

```javascript
import Fastify from 'fastify';
import { Type } from '@sinclair/typebox';

const app = Fastify();

// Define schema with TypeBox
const CreateUserBody = Type.Object({
  name: Type.String({ minLength: 1, maxLength: 100 }),
  email: Type.String({ format: 'email' }),
  age: Type.Optional(Type.Integer({ minimum: 0, maximum: 150 })),
});

const UserResponse = Type.Object({
  id: Type.String({ format: 'uuid' }),
  name: Type.String(),
  email: Type.String(),
  createdAt: Type.String({ format: 'date-time' }),
});

app.post('/users', {
  schema: {
    body: CreateUserBody,
    response: {
      201: UserResponse,
    },
  },
}, async (request, reply) => {
  const user = await createUser(request.body);
  reply.code(201);
  return user;
});
```

## TypeBox Common Patterns

```javascript
import { Type } from '@sinclair/typebox';

// Enums
const Status = Type.Union([
  Type.Literal('active'),
  Type.Literal('inactive'),
  Type.Literal('pending'),
]);

// Arrays
const Tags = Type.Array(Type.String(), { minItems: 1, maxItems: 10 });

// Nested objects
const Address = Type.Object({
  street: Type.String(),
  city: Type.String(),
  country: Type.String(),
  zip: Type.Optional(Type.String()),
});

// References (reusable schemas)
const User = Type.Object({
  id: Type.String({ format: 'uuid' }),
  name: Type.String(),
  address: Address,
  tags: Tags,
  status: Status,
});

// Nullable
const NullableString = Type.Union([Type.String(), Type.Null()]);

// Record/Map
const Metadata = Type.Record(Type.String(), Type.Unknown());
```

## Register TypeBox Schemas Globally

```javascript
import { Type } from '@sinclair/typebox';

// Define shared schemas
const ErrorResponse = Type.Object({
  error: Type.String(),
  message: Type.String(),
  statusCode: Type.Integer(),
});

const PaginationQuery = Type.Object({
  page: Type.Integer({ minimum: 1, default: 1 }),
  limit: Type.Integer({ minimum: 1, maximum: 100, default: 20 }),
});

// Register globally
app.addSchema(Type.Object({ $id: 'ErrorResponse', ...ErrorResponse }));
app.addSchema(Type.Object({ $id: 'PaginationQuery', ...PaginationQuery }));

// Reference in routes
app.get('/items', {
  schema: {
    querystring: { $ref: 'PaginationQuery#' },
    response: {
      400: { $ref: 'ErrorResponse#' },
    },
  },
}, handler);
```

## Plain JSON Schema (Alternative)

You can also use plain JSON Schema directly:

```javascript
import Fastify from 'fastify';

const app = Fastify();

const createUserSchema = {
  body: {
    type: 'object',
    properties: {
      name: { type: 'string', minLength: 1, maxLength: 100 },
      email: { type: 'string', format: 'email' },
      age: { type: 'integer', minimum: 0, maximum: 150 },
    },
    required: ['name', 'email'],
    additionalProperties: false,
  },
  response: {
    201: {
      type: 'object',
      properties: {
        id: { type: 'string', format: 'uuid' },
        name: { type: 'string' },
        email: { type: 'string' },
        createdAt: { type: 'string', format: 'date-time' },
      },
    },
  },
};

app.post('/users', { schema: createUserSchema }, async (request, reply) => {
  const user = await createUser(request.body);
  reply.code(201);
  return user;
});
```

## Request Validation Parts

Validate different parts of the request:

```javascript
const fullRequestSchema = {
  // URL parameters
  params: {
    type: 'object',
    properties: {
      id: { type: 'string', format: 'uuid' },
    },
    required: ['id'],
  },

  // Query string
  querystring: {
    type: 'object',
    properties: {
      include: { type: 'string', enum: ['posts', 'comments', 'all'] },
      limit: { type: 'integer', minimum: 1, maximum: 100, default: 10 },
    },
  },

  // Request headers
  headers: {
    type: 'object',
    properties: {
      'x-api-key': { type: 'string', minLength: 32 },
    },
    required: ['x-api-key'],
  },

  // Request body
  body: {
    type: 'object',
    properties: {
      data: { type: 'object' },
    },
    required: ['data'],
  },
};

app.put('/resources/:id', { schema: fullRequestSchema }, handler);
```

## Shared Schemas with $id

Define reusable schemas with `$id` and reference them with `$ref`:

```javascript
// Add shared schemas to Fastify
app.addSchema({
  $id: 'user',
  type: 'object',
  properties: {
    id: { type: 'string', format: 'uuid' },
    name: { type: 'string' },
    email: { type: 'string', format: 'email' },
    createdAt: { type: 'string', format: 'date-time' },
  },
  required: ['id', 'name', 'email'],
});

app.addSchema({
  $id: 'userCreate',
  type: 'object',
  properties: {
    name: { type: 'string', minLength: 1 },
    email: { type: 'string', format: 'email' },
  },
  required: ['name', 'email'],
  additionalProperties: false,
});

app.addSchema({
  $id: 'error',
  type: 'object',
  properties: {
    statusCode: { type: 'integer' },
    error: { type: 'string' },
    message: { type: 'string' },
  },
});

// Reference shared schemas
app.post('/users', {
  schema: {
    body: { $ref: 'userCreate#' },
    response: {
      201: { $ref: 'user#' },
      400: { $ref: 'error#' },
    },
  },
}, handler);

app.get('/users/:id', {
  schema: {
    params: {
      type: 'object',
      properties: { id: { type: 'string', format: 'uuid' } },
      required: ['id'],
    },
    response: {
      200: { $ref: 'user#' },
      404: { $ref: 'error#' },
    },
  },
}, handler);
```

## Array Schemas

Define schemas for array responses:

```javascript
app.addSchema({
  $id: 'userList',
  type: 'object',
  properties: {
    users: {
      type: 'array',
      items: { $ref: 'user#' },
    },
    total: { type: 'integer' },
    page: { type: 'integer' },
    pageSize: { type: 'integer' },
  },
});

app.get('/users', {
  schema: {
    querystring: {
      type: 'object',
      properties: {
        page: { type: 'integer', minimum: 1, default: 1 },
        pageSize: { type: 'integer', minimum: 1, maximum: 100, default: 20 },
      },
    },
    response: {
      200: { $ref: 'userList#' },
    },
  },
}, handler);
```

## Custom Formats

Add custom validation formats:

```javascript
import Fastify from 'fastify';

const app = Fastify({
  ajv: {
    customOptions: {
      formats: {
        'iso-country': /^[A-Z]{2}$/,
        'phone': /^\+?[1-9]\d{1,14}$/,
      },
    },
  },
});

// Or add formats dynamically
app.addSchema({
  $id: 'address',
  type: 'object',
  properties: {
    street: { type: 'string' },
    country: { type: 'string', format: 'iso-country' },
    phone: { type: 'string', format: 'phone' },
  },
});
```

## Custom Keywords

Add custom validation keywords:

```javascript
import Fastify from 'fastify';

const app = Fastify({
  ajv: {
    customOptions: {
      keywords: [
        {
          keyword: 'isEven',
          type: 'number',
          validate: (schema, data) => {
            if (schema) {
              return data % 2 === 0;
            }
            return true;
          },
          errors: false,
        },
      ],
    },
  },
});

// Use custom keyword
app.post('/numbers', {
  schema: {
    body: {
      type: 'object',
      properties: {
        value: { type: 'integer', isEven: true },
      },
    },
  },
}, handler);
```

## Coercion

Fastify coerces types by default for query strings and params:

```javascript
// Query string "?page=5&active=true" becomes:
// { page: 5, active: true } (number and boolean, not strings)

app.get('/items', {
  schema: {
    querystring: {
      type: 'object',
      properties: {
        page: { type: 'integer' },      // "5" -> 5
        active: { type: 'boolean' },    // "true" -> true
        tags: {
          type: 'array',
          items: { type: 'string' },    // "a,b,c" -> ["a", "b", "c"]
        },
      },
    },
  },
}, handler);
```

## Validation Error Handling

Customize validation error responses:

```javascript
app.setErrorHandler((error, request, reply) => {
  if (error.validation) {
    reply.code(400).send({
      error: 'Validation Error',
      message: 'Request validation failed',
      details: error.validation.map((err) => ({
        field: err.instancePath || err.params?.missingProperty,
        message: err.message,
        keyword: err.keyword,
      })),
    });
    return;
  }

  // Handle other errors
  reply.code(error.statusCode || 500).send({
    error: error.name,
    message: error.message,
  });
});
```

## Schema Compiler Options

Configure the Ajv schema compiler:

```javascript
import Fastify from 'fastify';

const app = Fastify({
  ajv: {
    customOptions: {
      removeAdditional: 'all',   // Remove extra properties
      useDefaults: true,         // Apply default values
      coerceTypes: true,         // Coerce types
      allErrors: true,           // Report all errors, not just first
    },
    plugins: [
      require('ajv-formats'),    // Add format validators
    ],
  },
});
```

## Nullable Fields

Handle nullable fields properly:

```javascript
app.addSchema({
  $id: 'profile',
  type: 'object',
  properties: {
    name: { type: 'string' },
    bio: { type: ['string', 'null'] },  // Can be string or null
    avatar: {
      oneOf: [
        { type: 'string', format: 'uri' },
        { type: 'null' },
      ],
    },
  },
});
```

## Conditional Validation

Use if/then/else for conditional validation:

```javascript
app.addSchema({
  $id: 'payment',
  type: 'object',
  properties: {
    method: { type: 'string', enum: ['card', 'bank'] },
    cardNumber: { type: 'string' },
    bankAccount: { type: 'string' },
  },
  required: ['method'],
  if: {
    properties: { method: { const: 'card' } },
  },
  then: {
    required: ['cardNumber'],
  },
  else: {
    required: ['bankAccount'],
  },
});
```

## Schema Organization

Organize schemas in a dedicated file:

```javascript
// schemas/index.js
export const schemas = [
  {
    $id: 'user',
    type: 'object',
    properties: {
      id: { type: 'string', format: 'uuid' },
      name: { type: 'string' },
      email: { type: 'string', format: 'email' },
    },
  },
  {
    $id: 'error',
    type: 'object',
    properties: {
      statusCode: { type: 'integer' },
      error: { type: 'string' },
      message: { type: 'string' },
    },
  },
];

// app.js
import { schemas } from './schemas/index.js';

for (const schema of schemas) {
  app.addSchema(schema);
}
```

## OpenAPI/Swagger Integration

Schemas work directly with @fastify/swagger:

```javascript
import fastifySwagger from '@fastify/swagger';
import fastifySwaggerUi from '@fastify/swagger-ui';

app.register(fastifySwagger, {
  openapi: {
    info: {
      title: 'My API',
      version: '1.0.0',
    },
  },
});

app.register(fastifySwaggerUi, {
  routePrefix: '/docs',
});

// Schemas are automatically converted to OpenAPI definitions
```

## Performance Considerations

Response schemas enable fast-json-stringify for serialization:

```javascript
// With response schema - uses fast-json-stringify (faster)
app.get('/users', {
  schema: {
    response: {
      200: {
        type: 'array',
        items: { $ref: 'user#' },
      },
    },
  },
}, handler);

// Without response schema - uses JSON.stringify (slower)
app.get('/users-slow', handler);
```

Always define response schemas for production APIs to benefit from optimized serialization.
