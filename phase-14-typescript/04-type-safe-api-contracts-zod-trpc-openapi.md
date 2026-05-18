# Type-safe API Contracts: Zod, tRPC, OpenAPI Codegen

## Quick Reference

| Approach | Source of truth | Runtime validation | Type inference | Best for |
|---|---|---|---|---|
| Zod | Schema definition | Yes | Yes (from schema) | Validating any input, deriving types |
| tRPC | TypeScript types on server | No (relies on TS) | Yes (end-to-end) | Full-stack TypeScript with no REST |
| OpenAPI codegen | OpenAPI spec (.yaml/.json) | Via generated validators | Yes (from spec) | REST APIs, polyglot teams |

---

## What Is This?

TypeScript's static types are erased at runtime — your `User` type doesn't exist when the code runs. When data crosses a system boundary (HTTP request, API response, localStorage, user input), TypeScript can't validate it. At runtime, you're working with plain JavaScript values that may or may not match your expected types.

These three tools solve the boundary problem in different ways: Zod validates and derives types from a runtime schema; tRPC eliminates the boundary by sharing types directly between client and server; OpenAPI codegen generates types from a separately-maintained spec.

---

## Zod: Runtime Validation + Type Inference

Zod defines schemas that serve dual purpose: they validate data at runtime *and* TypeScript infers static types from them. One source of truth.

### Basic schemas

```ts
import { z } from 'zod';

const UserSchema = z.object({
  id: z.string().uuid(),
  name: z.string().min(1).max(100),
  email: z.string().email(),
  age: z.number().int().min(0).max(150).optional(),
  role: z.enum(['admin', 'user', 'moderator']),
  createdAt: z.string().datetime(),
});

// Type is inferred — no duplicate declaration:
type User = z.infer<typeof UserSchema>;
// { id: string; name: string; email: string; age?: number; role: 'admin' | 'user' | 'moderator'; createdAt: string }
```

### Parsing and safe-parsing

```ts
// .parse() throws ZodError on failure:
const user = UserSchema.parse(rawData); // throws if invalid

// .safeParse() returns a discriminated union — never throws:
const result = UserSchema.safeParse(rawData);
if (result.success) {
  console.log(result.data.name); // User type — fully typed
} else {
  console.error(result.error.issues); // ZodIssue[] — detailed error paths
}
```

### Transformations

Zod can transform values during parsing — parse from a string, validate, and transform to a Date:

```ts
const DateSchema = z.string().datetime().transform(str => new Date(str));
type ParsedDate = z.infer<typeof DateSchema>; // Date (post-transform)
```

### Composing schemas

```ts
const PaginationSchema = z.object({
  page: z.number().int().positive().default(1),
  limit: z.number().int().min(1).max(100).default(20),
});

const UserListQuerySchema = PaginationSchema.extend({
  search: z.string().optional(),
  role: z.enum(['admin', 'user']).optional(),
});

// Partial, Pick, Omit work like TypeScript utility types:
const UserUpdateSchema = UserSchema.partial().omit({ id: true, createdAt: true });
```

### Using Zod at API boundaries

```ts
// API route handler:
app.post('/users', async (req, res) => {
  const result = CreateUserSchema.safeParse(req.body);
  if (!result.success) {
    return res.status(400).json({
      errors: result.error.issues.map(i => ({ path: i.path, message: i.message })),
    });
  }

  const user = await userService.create(result.data); // result.data is CreateUser type
  res.json(user);
});
```

### React Hook Form + Zod

```ts
import { zodResolver } from '@hookform/resolvers/zod';
import { useForm } from 'react-hook-form';

function UserForm() {
  const { register, handleSubmit, formState: { errors } } = useForm<User>({
    resolver: zodResolver(UserSchema),
  });

  return (
    <form onSubmit={handleSubmit(onSubmit)}>
      <input {...register('name')} />
      {errors.name && <span>{errors.name.message}</span>}
    </form>
  );
}
```

> **Check yourself:** If TypeScript types are erased at runtime, why does Zod need to be a separate library rather than being built into TypeScript?

---

## tRPC: End-to-End Type Safety Without a Schema

tRPC shares TypeScript types directly between client and server — no schema language, no code generation, no REST. The server's TypeScript types become the client's types via type inference.

### Server setup

```ts
import { initTRPC } from '@trpc/server';
import { z } from 'zod';

const t = initTRPC.create();

export const appRouter = t.router({
  users: t.router({
    getById: t.procedure
      .input(z.object({ id: z.string().uuid() }))
      .query(async ({ input }) => {
        // input is typed as { id: string }
        const user = await db.user.findUnique({ where: { id: input.id } });
        return user; // return type inferred by TypeScript
      }),

    create: t.procedure
      .input(CreateUserSchema)
      .mutation(async ({ input }) => {
        return await db.user.create({ data: input });
      }),
  }),
});

export type AppRouter = typeof appRouter; // this type is exported to the client
```

### Client setup

```ts
import { createTRPCReact } from '@trpc/react-query';
import type { AppRouter } from '../server/router'; // TYPE-only import — no runtime code

export const trpc = createTRPCReact<AppRouter>();

// In a React component:
function UserProfile({ id }: { id: string }) {
  const { data, isLoading } = trpc.users.getById.useQuery({ id });
  // data is User | undefined — fully typed from server's return type
  return <div>{data?.name}</div>;
}
```

### What makes tRPC special

- No `fetch` + JSON.parse + type assertion boilerplate
- No API client generation step
- Changing the server's return type immediately causes TypeScript errors on the client
- Input validation via Zod schemas on the procedure
- Works with React Query for caching and loading states

**Limitation:** Server and client must be in the same TypeScript project (monorepo) or the client must import server types. Not suited for polyglot teams (Go backend, React frontend) or public APIs that third parties consume.

---

## OpenAPI Codegen: Type Safety from a Spec

For REST APIs — especially when the backend is not TypeScript, or when you maintain a public API with a spec — OpenAPI codegen generates TypeScript types and API clients from an OpenAPI 3.x YAML/JSON spec.

### The spec (source of truth)

```yaml
# openapi.yaml
openapi: 3.0.0
info:
  title: User API
  version: 1.0.0
paths:
  /users/{id}:
    get:
      operationId: getUserById
      parameters:
        - name: id
          in: path
          required: true
          schema:
            type: string
            format: uuid
      responses:
        '200':
          content:
            application/json:
              schema:
                $ref: '#/components/schemas/User'
components:
  schemas:
    User:
      type: object
      required: [id, name, email]
      properties:
        id:
          type: string
          format: uuid
        name:
          type: string
        email:
          type: string
          format: email
```

### Generating types

```bash
# openapi-typescript (generates types only, no runtime client):
npx openapi-typescript openapi.yaml -o src/api/types.gen.ts

# openapi-generator-cli (generates full client):
npx openapi-generator-cli generate -i openapi.yaml -g typescript-fetch -o src/api/client
```

### Generated types (openapi-typescript output)

```ts
// src/api/types.gen.ts (auto-generated — don't edit)
export interface paths {
  '/users/{id}': {
    get: {
      parameters: { path: { id: string } };
      responses: {
        200: { content: { 'application/json': components['schemas']['User'] } };
      };
    };
  };
}

export interface components {
  schemas: {
    User: {
      id: string;
      name: string;
      email: string;
    };
  };
}
```

### Using with `openapi-fetch`

```ts
import createClient from 'openapi-fetch';
import type { paths } from './api/types.gen';

const client = createClient<paths>({ baseUrl: 'https://api.example.com' });

// Fully typed — TypeScript knows the path, params, and response:
const { data, error } = await client.GET('/users/{id}', {
  params: { path: { id: '123' } },
});
// data is User | undefined, error is problem response
```

### Keeping types in sync

The spec is the contract. In CI:

```bash
# Regenerate types and fail if they differ from committed types:
npx openapi-typescript openapi.yaml -o src/api/types.gen.ts
git diff --exit-code src/api/types.gen.ts
```

This catches spec changes that weren't reflected in the generated types.

---

## Comparison: When to Use Which

| Scenario | Tool |
|---|---|
| Full-stack TypeScript, monorepo | tRPC |
| Validate form input or API responses | Zod |
| REST API with OpenAPI spec, non-TS backend | OpenAPI codegen |
| Public API consumed by multiple clients | OpenAPI codegen |
| Validating any external data (CSV, environment vars) | Zod |
| Need runtime validation + static types from one source | Zod |

These tools are not mutually exclusive: tRPC uses Zod for input validation; an OpenAPI client can use Zod for response validation; Zod schemas can be exported as OpenAPI schemas using `zod-to-json-schema`.

---

## Interview Questions

**Q (High): Why does TypeScript's static type system not protect against runtime type errors from API responses?**

Answer: TypeScript's types are a compile-time construct — they're completely erased in the compiled JavaScript output. When your code makes a `fetch` call and casts the response as a typed object (`const user = await res.json() as User`), TypeScript trusts your assertion. At runtime, the JavaScript doesn't check that the JSON actually matches `User`. If the API returns a different shape (missing fields, wrong types, extra nulls), your code proceeds with wrong data — TypeScript can't stop it because there are no types at runtime. This is why runtime validation libraries like Zod exist: Zod schemas perform actual JavaScript checks on the actual values, rejecting data that doesn't match the expected shape, and only then providing the typed result to TypeScript.

**Q (High): What is tRPC and how does it eliminate the API contract synchronization problem?**

Answer: tRPC is a TypeScript RPC framework where the server's router definition is the type source of truth. By exporting `typeof appRouter` from the server and importing it as a type-only import on the client, TypeScript can infer the exact types of every procedure's input and output. If a developer changes a server procedure's return type, TypeScript immediately reports errors at every call site on the client — no code generation step, no schema update, no deployment needed to see the type error. This eliminates the category of bugs where the API contract drifts: backend changes the response shape, frontend continues using the old type, runtime errors appear. The limitation is that both client and server must be TypeScript in the same monorepo or a package that shares the server types.

**Q (Medium): You have a Zod schema for user data. How would you use it for both API request validation and React Hook Form?**

Answer: Define the schema once: `const UserSchema = z.object({ name: z.string().min(1), email: z.string().email() })` and `type User = z.infer<typeof UserSchema>`. For API validation: `const result = UserSchema.safeParse(req.body)` — reject with 400 if `!result.success`, proceed with `result.data` (typed as `User`) if successful. For React Hook Form: pass `resolver: zodResolver(UserSchema)` in `useForm<User>({ resolver: zodResolver(UserSchema) })`. The form automatically validates inputs against the schema on submit and optionally on each field change, providing typed error messages. The same schema enforces the same rules in both contexts — changing validation logic in one place changes it everywhere.

**Q (Low): What is the difference between openapi-typescript (type generation only) and openapi-generator-cli (full client generation)?**

Answer: `openapi-typescript` generates TypeScript type definitions from an OpenAPI spec — interface declarations for paths, parameters, request bodies, and response schemas. It does not generate a fetch client; you write your own fetch calls using those types, or use `openapi-fetch` which wraps fetch and uses the generated types for parameter and response typing. `openapi-generator-cli` generates a complete API client — concrete classes or functions that make the HTTP calls, with typed parameters and typed return values. The full client is more turnkey but generates more code, can be opinionated about the HTTP client used, and may produce patterns you'd write differently. Type-only generation is lighter and gives you control over how requests are made; full client generation is faster to start with but harder to customize.

---

## Self-Assessment

- [ ] Write a Zod schema for a paginated API response with type inference
- [ ] Explain why TypeScript type assertions on `res.json()` are insufficient for type safety
- [ ] Describe how tRPC shares types between server and client and what the limitation is
- [ ] Describe the OpenAPI codegen workflow from spec to typed client calls
- [ ] Explain when you'd choose each of the three approaches for a new project

---
*Next: Monorepo Shared Type Packages — package boundaries, path mapping, and TypeScript composite projects.*
