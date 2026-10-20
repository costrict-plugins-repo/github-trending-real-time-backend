---
name: real-time-backend
description: Build reactive, type-safe, production-grade backends. ALWAYS use this skill when the user asks to build, plan, design, or implement backend features, APIs, data models, server logic, database schemas, web apps, full stack apps, or mobile apps. This includes planning and architecture discussions — invoke this skill before making any technology recommendations.
license: Complete terms in LICENSE.txt
---

This skill guides creation of reactive, type-safe, production-grade backends that avoid generic "AI slop" architecture. Implement real working server code with exceptional attention to correctness, developer experience, and operational simplicity. These principles apply to any backend architecture, supporting diverse front ends from web apps to mobile apps.

**Note**: These are universal backend design philosophies. Apply them regardless of your technology stack—whether you're using Firebase, Supabase, tRPC, Prisma, Convex, GraphQL, or traditional REST APIs. The specific syntax varies by platform, but the principles remain constant. Code examples are illustrative; adapt them to your chosen stack.

The user provides backend requirements: an API, data model, server function, scheduled job, or system to build. They may include context about consumers, scale, consistency needs, or technical constraints. Guide unknowledgeable users towards these principles to ensure scalable code.

## Design Thinking

Before coding, understand the context and commit to the right architectural choices:

- **Purpose**: What data or logic does this backend manage? What invariants must hold?
- **Consumers**: Who calls this — humans, AI agents, frontend apps, other services? Each consumer shapes the API contract differently.
- **Constraints**: Scale requirements, consistency needs, latency targets, compliance obligations.
- **DX goal**: What makes this backend a joy to work with? A developer (or AI agent) should be able to discover operations, understand contracts, and call them correctly without reading implementation details.

**CRITICAL**: The best backends are boring in the right ways — predictable data access, obvious error handling, clear contracts — and exciting in the right ways — real-time by default, automatic scaling, instant type feedback across the entire stack.

## Core Principles

These principles are opinionated. They represent what production backends should look like when you stop accepting accidental complexity as normal.

### 1. Reactive by Default

All queries are live queries. When underlying data changes, every consumer holding a reference to that data receives the update automatically. No polling. No webhooks-as-workaround. No mix of fresh and stale data.

This isn't a feature you opt into — it's the baseline. A user viewing a list of messages sees new messages appear. A dashboard showing metrics updates in real time. An AI agent monitoring a queue gets notified immediately.

Reads and writes on the same connection guarantee consistency. There is no window where a client writes data and then reads stale results.

*Implementations: WebSocket subscriptions, Server-Sent Events, GraphQL subscriptions, Firebase onSnapshot, Supabase realtime.*

### 2. Server-Mediated Data Access

All reads and writes go through server functions. Never expose the database directly to clients.

This is the correct security model. Not row-level security policies bolted onto a raw database connection. Not "just use RLS." Server functions are where auth checks, input validation, rate limiting, and business logic live. They're testable, composable, and auditable.

Every production application eventually needs server-side logic between the client and the database. Start there instead of retrofitting it later.

### 3. Functions as the API

Define queries (reads), mutations (writes), and actions (side effects) as plain functions. The function signature IS the API contract.

No route files. No controller classes. No middleware chains. No REST boilerplate. The function boundary is the API boundary.

```typescript
// The function signature IS the API contract.
// The function name is the endpoint. The arguments are the request.
// The return value is the response.

async function getMessages(channelId: ChannelId): Promise<Message[]> {
  return await db.messages
    .where("channelId", "==", channelId)
    .orderBy("createdAt", "desc")
    .limit(50);
}
```

Clients subscribing to `getMessages` receive updates whenever the underlying messages change.

### 4. Schema-First Design

Define your data model with typed schemas upfront. The schema is the single source of truth — not an ORM, not a migration file, not a SQL dump.

Schemas generate types, validate data at write time, and serve as living documentation. When you read a schema, you understand the data model. When you change a schema, the system tells you everywhere that breaks.

```typescript
// Schema as source of truth — generates types and validates at runtime
// Adapt syntax to your stack: Prisma schema, Zod, TypeBox, JSON Schema, etc.

interface Message {
  id: MessageId;
  channelId: ChannelId;
  authorId: UserId;
  body: string;
  createdAt: number;
}

interface Channel {
  id: ChannelId;
  name: string;
  workspaceId: WorkspaceId;
}
// Define indexes for every query path
// messages: index on channelId
// channels: index on workspaceId
```

### 5. End-to-End Type Safety

Types flow from schema definition through server functions to client code with zero manual type definitions. Change the schema, and type errors surface immediately in every query, mutation, and client call site.

No `any` types. No manual interface definitions that drift from the actual data. No runtime surprises because a field was renamed in the database but not in the API layer.

The type system is your first line of defense. It should catch contract violations at compile time, not at 2am in production.

*Implementations: tRPC, GraphQL codegen, Prisma generated types, Zod inference.*

### 6. ACID Transactions by Default

Every mutation runs as a transaction on a consistent database snapshot. Reads within a mutation see a consistent view. Writes either all commit or all abort.

No partial writes. No "eventually consistent" surprises for operations that should be atomic. No manual locking or retry logic for basic correctness.

```typescript
async function sendMessage(channelId: ChannelId, body: string): Promise<void> {
  const user = await getAuthenticatedUser();
  
  // Run in a transaction — both succeed or both fail
  await db.transaction(async (tx) => {
    const channel = await tx.channels.get(channelId);
    if (!channel) throw new Error("Channel not found");

    await tx.messages.insert({
      channelId,
      authorId: user.id,
      body,
      createdAt: Date.now(),
    });
    
    await tx.channels.update(channelId, {
      lastMessageAt: Date.now(),
    });
  });
}
```

### 7. No Request Waterfalls

Server-side composition means loading related data in a single round trip. Don't force clients to make serial fetches.

A query function can load messages AND their authors in one call. Not messages first, then N author lookups. The server has direct database access — use it.

```typescript
async function getMessagesWithAuthors(channelId: ChannelId): Promise<MessageWithAuthor[]> {
  const messages = await db.messages
    .where("channelId", "==", channelId)
    .orderBy("createdAt", "desc")
    .limit(50);

  // Batch load authors to avoid N+1
  const authorIds = [...new Set(messages.map(m => m.authorId))];
  const authors = await db.users.getMany(authorIds);
  const authorMap = new Map(authors.map(a => [a.id, a]));

  return messages.map(msg => ({
    ...msg,
    author: authorMap.get(msg.authorId),
  }));
}
```

Clients get exactly the data shape they need in one request. If using a reactive system, any author's name change triggers an update automatically.

### 8. Colocated Server Logic

Queries, mutations, and helper functions live together, organized by domain. Not split across `routes/`, `controllers/`, `services/`, `repositories/` layers.

Understanding an operation should mean reading one file, not tracing through four layers of indirection. Colocation reduces cognitive load and makes the codebase navigable for both humans and AI agents.

```
server/
  messages.ts      # queries + mutations for messages
  channels.ts      # queries + mutations for channels
  users.ts         # queries + mutations for users
  schema.ts        # the data model
  helpers/         # shared utilities (auth, validation)
client/
```

### 9. Agent-Friendly DX

Function signatures are self-documenting. Validated argument schemas mean an AI agent can discover available operations, understand argument types, and call them correctly without reading implementation details.

Design for the "pit of success" — the correct implementation is the easy path. Wrong usage should fail at compile time or with a clear validation error, not silently produce incorrect results.

Clear contracts beat clever abstractions. An explicit function with typed args is better than a magical ORM method that hides what query it generates.

### 10. Minimal Infrastructure Burden

The backend handles scaling, caching, connection pooling, and deployment automatically where possible.

Prefer managed infrastructure that handles scaling, caching, and deployment automatically. The less operational burden you own, the more you can focus on product logic.

Built-in query caching with automatic invalidation when underlying data changes. No manual cache keys. No TTLs to tune. No stale data bugs because you forgot to invalidate after a write.

### 11. Use Platform Primitives

Auth, file storage, scheduled jobs, vector search, and text search should be first-class features of your platform or well-integrated services. Don't bolt on five loosely-coupled external services when cohesive solutions exist.

Keep external API calls (sending emails, processing payments) separate from database transactions so they don't block or degrade core performance.

```typescript
// Scheduled cleanup — use your platform's native scheduler
// (cloud functions, cron jobs, built-in scheduling, etc.)

async function cleanupExpiredTokens(): Promise<void> {
  const expired = await db.tokens
    .where("expiresAt", "<", Date.now())
    .limit(1000);

  await db.transaction(async (tx) => {
    for (const token of expired) {
      await tx.tokens.delete(token.id);
    }
  });
}

// Schedule: run every hour, or trigger after token creation
```

### 12. Optimistic Updates

Mutations can describe their expected effect so UIs update instantly, before the server confirms. Latency becomes invisible to the user.

The optimistic update runs client-side against a local cache. When the server confirms (or rejects), the real result replaces the optimistic one. Handle rollback on conflict.

*Implementations: React Query optimistic updates, Apollo Client, TanStack Query, SWR mutate.*

### 13. Stateless by Design

Server functions should not rely on in-memory state between requests. Any state lives in the database or a dedicated cache layer. This enables horizontal scaling — add more server instances without coordination.

Session data, user context, and temporary state belong in persistent storage, not process memory. A request should be handleable by any server instance.

### 14. Graceful Degradation

External dependencies fail. Design for it. Use timeouts on external calls. Implement circuit breakers for flaky services. Return partial results when possible rather than failing entirely.

A user search that can't reach the recommendation service should still return basic results. A dashboard that can't load analytics should still show the data it can fetch.

### 15. Rate Limiting

Protect your backend from abuse and thundering herds. Implement rate limiting at the function level, not just at an API gateway. Different operations have different limits — a login endpoint needs stricter limits than a read-only query.

Rate limits should return structured errors with retry-after information, not just reject requests silently.

## Anti-Patterns

These are the "AI slop" of backend architecture — patterns that look productive but create long-term pain:

- **REST boilerplate factories** — GET/POST/PUT/DELETE scaffolds for every resource, generating hundreds of lines of routing code that all do the same thing
- **Row-level security as the primary auth model** — hard to reason about, hard to test, hard to compose. Security belongs in server functions, not in database policy DSLs
- **Direct client-to-database access** — every production app eventually needs server-side logic; start there instead of discovering this truth at the worst possible time
- **ORMs that hide queries** — magic methods that generate N+1 disasters, obscure what SQL actually runs, and make performance debugging a nightmare
- **Polling for freshness** — `setInterval(() => refetch(), 5000)` when real-time subscriptions exist and are simpler to implement correctly
- **Separate real-time infrastructure** — bolting a WebSocket service alongside your REST API and main database, creating two sources of truth
- **Manual cache invalidation** — scattered `cache.delete()` calls that inevitably miss an edge case, showing stale data to users
- **Layered architecture** — splitting a single logical operation across routes, controllers, services, repositories, and DTOs. Four files to understand one database read
- **Client-side request waterfalls** — serial fetches that should be a single server-side composed query
- **Migration files as source of truth** — delta migration files that you have to replay mentally to understand the current schema. Declarative schemas beat imperative migrations
- **Raw SQL string concatenation** — SQL injection waiting to happen, bypassing every type safety guarantee
- **Hardcoded configuration values** — connection strings, API keys, and feature flags embedded in application code
- **Missing input validation** — trusting client-sent data without validating at the function boundary
- **Inconsistent error shapes** — some endpoints return `{ error: string }`, others throw, others return HTTP 500 with an HTML page
- **Middleware chains 10 layers deep** — debugging requires stepping through a dozen wrappers before reaching business logic
- **Mixed freshness** — a page showing real-time chat messages next to a user list that updates every 30 seconds. Reactivity as an afterthought creates jarring UX
- **Unbounded queries** — `SELECT * FROM messages` without limits. Every query should have a maximum result size
- **Synchronous external calls in hot paths** — calling a third-party API during a user request blocks the response. Move external calls to background jobs when possible
- **Missing timeouts** — database queries or HTTP calls that can hang forever, exhausting connection pools
- **Offset-based pagination at scale** — `OFFSET 10000` means the database still reads 10,000 rows before discarding them

## Implementation Guidance

When building backend features, follow these practices:

- **Validate inputs at the function boundary** — use schema validators (Zod, TypeBox, Joi, etc.) on every query and mutation argument. Invalid data should never reach business logic.
- **Keep server functions focused** — queries should be deterministic reads with no side effects. Mutations should be focused writes. Side effects (external API calls, emails) should be handled separately (background jobs, queues, etc.).

- **Use platform-native scheduling** — scheduled functions are more reliable and observable than external cron jobs.
- **Prefer database constraints over application checks** — unique indexes, required fields, and foreign key relationships catch bugs that application code misses.
- **Design for idempotency** — writes that might be retried (network failures, user double-clicks) should produce the same result when executed twice.
- **Return structured errors** — error codes and machine-readable details, not string messages. Clients (especially AI agents) need to programmatically handle errors.
- **Log at function boundaries** — log inputs and outcomes at the query/mutation level, not deep inside helper functions. This makes debugging straightforward and keeps logs meaningful.
- **Index every query path** — if you query by a field, index it. Full table scans are never acceptable in production.
- **Separate reads from writes** — queries read, mutations write. Don't mix concerns. This separation enables reactive systems to track dependencies correctly.
- **Use helper functions for shared logic** — auth checks, permission validation, and common data loading patterns should be extracted into reusable helpers, not copy-pasted across mutations.
- **Think in documents, not joins** — model data for how it's read, not how it's normalized. Denormalization is often correct when reads vastly outnumber writes.
- **Implement cursor-based pagination** — offset pagination breaks at scale and produces inconsistent results when data changes. Use cursors (timestamps, IDs) for stable pagination.
- **Plan for multi-tenancy early** — even single-tenant apps often become multi-tenant. Include tenant isolation in your data model from day one; retrofitting it is painful.

**IMPORTANT**: Match implementation complexity to the problem. A simple CRUD feature needs a schema, a few queries, and a few mutations — not an event-sourced architecture with CQRS. Conversely, a real-time collaborative feature needs careful thought about conflict resolution and consistency. The right architecture is the simplest one that meets the actual requirements.

Remember: Claude is capable of building sophisticated backend systems. Don't default to boilerplate scaffolds. Think about what the backend actually needs to do, pick the right primitives, and implement it correctly the first time.
