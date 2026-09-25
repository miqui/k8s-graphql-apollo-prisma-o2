# GraphQL DataLoader: Batching Per-Parent Lookups

A design note on eliminating the N+1 query problem in `issue-service` and `message-service`. This is the structural follow-up to [GRAPHQL-API-DESIGN.md](GRAPHQL-API-DESIGN.md) (Layer 4).

> **Status:** not yet implemented. The query-depth/complexity guards in `src/guards.ts` and `issue-service/src/guards.ts` are the current backstop; this doc is the implementation plan.

## The problem today

Nearly every field resolver in `issue-service/src/resolvers.ts` issues its own Prisma query per parent object. The schema is full of bidirectional edges, so a nested query triggers one query per edge per parent at every level.

### A concrete example

```graphql
{
  workspace(id: "w") {
    projects {                # 1 query for projects
      issues(first: 10) { edges { node {
        labels {              # 10 queries (one per issue)
          issues {            # 10 × however many labels each issue has
            project {         # 100+ queries
              workspace {     # 100+ queries
                projects {   # ...
                  issues(first: 10) { edges { node { id } } }
                }
              }
            }
          }
        }
      } } }
    }
  }
}
```

Resolver calls ≈ database queries because `Issue.labels`, `Label.issues`, `Issue.project`, `Project.workspace`, `Workspace.projects`, `Project.issues`, etc. each do an independent `prisma.*.findMany` / `findUnique` call per parent.

The 164-byte query from [GRAPHQL-API-DESIGN.md](GRAPHQL-API-DESIGN.md) triggers **11,411 resolver calls** with stub resolvers returning 10 items per list. DataLoader turns each level into **one batched query**, dropping the database round-trips from thousands to single digits.

## What DataLoader does

[DataLoader](https://github.com/graphql/dataloader) is a per-request batching and memoization utility. For each incoming GraphQL operation, you create a set of loaders. Each loader:

1. **Queues keys** during a single tick of the event loop. When `issueLoader.load("i1")`, `issueLoader.load("i2")`, … are called from sibling resolvers, nothing happens immediately.
2. **Dispatches one batched query** once the event loop is free, e.g. `prisma.issue.findMany({ where: { id: { in: ["i1", "i2", …] } } })`.
3. **Returns results in key order**, so each resolver gets the correct object.
4. **Caches per-request**, so `projectLoader.load("p1")` called twice in the same operation returns the same promise without a second database hit.

DataLoader is **not** a cache between requests (it is created and discarded per-request). It is **not** a replacement for depth/complexity guards — it makes legitimate queries fast; guards still reject abusive shapes.

## Where it fits in the Apollo lifecycle

DataLoader loaders are created in the **Apollo context function**, which runs once per request. That makes them naturally scoped to a single operation.

```ts
// issue-service/src/index.ts (sketch — not yet implemented)
import DataLoader from "dataloader";
import { prisma } from "./prisma";

function createLoaders() {
  return {
    workspaceById: new DataLoader(async (ids: readonly string[]) => {
      const rows = await prisma.workspace.findMany({ where: { id: { in: [...ids] } } });
      return ids.map(id => rows.find(r => r.id === id));
    }),
    projectById: new DataLoader(async (ids: readonly string[]) => { … }),
    issueById: new DataLoader(async (ids: readonly string[]) => { … }),
    labelById: new DataLoader(async (ids: readonly string[]) => { … }),
    commentById: new DataLoader(async (ids: readonly string[]) => { … }),
    incidentById: new DataLoader(async (ids: readonly string[]) => { … }),
    problemById: new DataLoader(async (ids: readonly string[]) => { … }),
    changeById: new DataLoader(async (ids: readonly string[]) => { … }),
    serviceRequestById: new DataLoader(async (ids: readonly string[]) => { … }),
    // List fields need a different pattern (see below)
  };
}

// In the ApolloServer config:
app.use(
  "/graphql",
  express.json(),
  expressMiddleware(apollo, {
    context: async () => ({ loaders: createLoaders(), prisma }),
  }),
);
```

Resolvers then read from the loader instead of hitting Prisma directly:

```ts
// Before
Issue: {
  async project(parent: { projectId: string }) {
    return prisma.project.findUniqueOrThrow({ where: { id: parent.projectId } });
  },
}

// After
Issue: {
  async project(parent, _args, { loaders }) {
    return loaders.projectById.load(parent.projectId);
  },
}
```

### Type safety

Add the loaders to the context type so resolvers are typed:

```ts
// issue-service/src/context.ts (new file)
import type DataLoader from "dataloader";
import type { PrismaClient } from "@prisma/client";

export interface GraphQLContext {
  prisma: PrismaClient;
  loaders: {
    workspaceById: DataLoader<string, Workspace | undefined>;
    projectById: DataLoader<string, Project | undefined>;
    // …
  };
}
```

Then type the resolver functions against `GraphQLContext` instead of `unknown` for the context argument.

## List fields: one-to-many

DataLoader's batching is for **many loads → one query**. For list fields (e.g., `Project.issues`, `Issue.comments`, `Comment.replies`), the load key is the **parent ID** and the result is a **list of children**. This still works — the batch function receives all parent IDs and returns arrays in the same order:

```ts
issuesByProjectId: new DataLoader(async (projectIds: readonly string[]) => {
  const rows = await prisma.issue.findMany({
    where: { projectId: { in: [...projectIds] } },
    orderBy: [{ createdAt: "asc" }, { id: "asc" }],
  });
  return projectIds.map(pid => rows.filter(r => r.projectId === pid));
}),
```

Resolver:

```ts
Project: {
  async issues(parent, args, { loaders }) {
    // The loader gives us the raw list; pagination/cursor logic still applies on top.
    const all = await loaders.issuesByProjectId.load(parent.id);
    return applyConnectionPagination(all, args.first, args.after);
  },
}
```

**Caveat:** If the list is large (e.g. thousands of comments on an issue), loading the entire list into memory to paginate in JS defeats the purpose. For those fields, keep Prisma's `findMany({ take, skip/cursor })` in the resolver and add DataLoader only for the foreign-key lookups. The highest-value wins are the **bidirectional single-object edges**: `Issue.project`, `Project.workspace`, `Incident.issue`, `Comment.parent`, etc.

## Priority order: which loaders first

Not every field needs a loader. Start with the ones that fire most often in nested queries:

### issue-service (highest impact)

| Resolver | Current query pattern | Loader |
|---|---|---|
| `Issue.project` | `findUniqueOrThrow` per issue | `projectById` |
| `Project.workspace` | `findUniqueOrThrow` per project | `workspaceById` |
| `Workspace.projects` | `findMany` per workspace | `projectsByWorkspaceId` |
| `Incident.issue` | `findUniqueOrThrow` per incident | `issueById` |
| `Problem.issue` | `findUniqueOrThrow` per problem | `issueById` |
| `Change.issue` | `findUniqueOrThrow` per change | `issueById` |
| `ServiceRequest.issue` | `findUniqueOrThrow` per SR | `issueById` |
| `Comment.parent` | `findUnique` per comment | `commentById` |
| `Comment.issue` | `findUniqueOrThrow` per comment | `issueById` |
| `Comment.replies` | `findMany` per comment | `repliesByCommentId` |
| `Label.issues` | `findMany` per label | `issuesByLabelId` |
| `Incident.problem` | `findUnique` per incident | `problemById` |
| `Problem.incidents` | `findMany` per problem | `incidentsByProblemId` |
| `Problem.changes` | `findMany` per problem | `changesByProblemId` |
| `Change.problem` | `findUnique` per change | `problemById` |

Notice `issueById` serves five different resolvers (`Incident.issue`, `Problem.issue`, `Change.issue`, `ServiceRequest.issue`, `Comment.issue`) — that is exactly the kind of fan-out DataLoader fixes.

### message-service (lower impact, smaller schema)

| Resolver | Loader |
|---|---|
| `Message.author` | `authorById` |
| `Author.messages` | `messagesByAuthorId` |

## How this changes the cost model

[GRAPHQL-API-DESIGN.md](GRAPHQL-API-DESIGN.md)'s complexity estimator assumes **resolver calls ≈ database queries**. Once DataLoader is in place, that assumption loosens:

- A single-object lookup by ID costs **one query per distinct ID set**, regardless of how many resolvers reference it.
- The "×10" placeholder on unbounded list fields is still needed because the lists themselves aren't capped, but the *bidirectional ping-pong* stops being catastrophic.
- After adding DataLoader, the next step is to **paginate or cap the remaining unbounded lists** (Layer 4 in the design doc). Once `Workspace.projects`, `Issue.comments`, `Comment.replies`, etc. have real bounds, the complexity estimator can drop the ×10 guess and the `MAX_COMPLEXITY = 5000` limit can be lowered.

## Dependency

```sh
npm install dataloader
# Both workspaces
```

`dataloader@^2.2.2` is a single file with no runtime dependencies beyond a `Promise` implementation. It type-checks cleanly under this repo's `tsconfig` (`strict`, `module: node16`).

## Rollout plan

1. **Add `dataloader` to both `package.json` and `issue-service/package.json`.**
2. **issue-service first** — the N+1 surface is largest there. Create `issue-service/src/context.ts` with the `GraphQLContext` type and a `createLoaders()` factory. Wire it into `issue-service/src/index.ts` via `expressMiddleware(apollo, { context: … })`.
3. **Convert the highest-value single-object resolvers first**: `Issue.project`, `Project.workspace`, `Incident.issue`, `Problem.issue`, `Change.issue`, `ServiceRequest.issue`, `Comment.issue`, `Comment.parent`.
4. **Add list-field loaders** for `Workspace.projects`, `Issue.comments`, `Comment.replies`, `Label.issues`, `Problem.incidents`, `Problem.changes`.
5. **message-service second** — `Message.author` and `Author.messages`.
6. **Verify with `EXAMPLES.md` queries** (run them through a real ApolloServer with stub resolvers and log Prisma query counts) to confirm the batching works.
7. **Lower `MAX_COMPLEXITY`** as real bounds replace the ×10 guess on paginated fields.

## Alternatives considered

| Option | Verdict |
|---|---|
| Prisma's built-in [query engine batching](https://www.prisma.io/docs/orm/prisma-client/queries/query-optimization-performance) | Prisma batches *some* `findUnique` calls automatically when they happen in the same tick, but it is heuristic, opt-in (`prisma.$transaction([…])`), and does not cover `findMany` or cross-tick resolver boundaries. DataLoader is explicit and works the same way for every field. |
| Custom in-memory request cache (plain `Map`) | Gives memoization but not batching. If 100 issues all need `projectId: "p1"`, a Map still fires 99 no-ops and 1 query; DataLoader fires exactly 1 query. |
| Redis / cross-request caching | Out of scope. The Hazelcast cache in `message-service` is for hot `Message` lookups by ID (same pattern, different layer). DataLoader is per-request; adding cross-request caching is a separate decision with invalidation complexity. |

## Adjacent work

- **Paginate unbounded lists** — see the Layer 4 table in [GRAPHQL-API-DESIGN.md](GRAPHQL-API-DESIGN.md). DataLoader makes the ping-pong cheap, but capping list sizes is still needed so a single field doesn't return 10,000 rows.
- **Add a test runner** — a small integration test that counts Prisma query invocations (via `prisma.$on("query", …)`) would prove DataLoader is batching. The repo has no test runner today; this is another good reason to add one.
