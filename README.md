# @dangminhdev/query-resource

Typed, transport-agnostic resource factory for TanStack Query v5.

It standardizes query keys, query options, infinite queries, mutations,
optimistic cache updates, rollback, invalidation, prefetch, and SSR cache
warming. It does **not** call an HTTP API itself: `fetch`, Axios, GraphQL, DTO
mapping, and domain rules remain in your project.

## Install

```bash
pnpm add @dangminhdev/query-resource @tanstack/react-query
# or: npm install @dangminhdev/query-resource @tanstack/react-query
```

The package requires TanStack Query v5 and expects your application to already
provide a `QueryClientProvider`.

## The boundary

```text
HTTP client -> repository -> resource -> feature hook -> component
```

| Layer | Owns |
| --- | --- |
| Repository | HTTP endpoint, DTO validation/mapping, pagination normalization |
| Resource | Query keys, query/infinite options, mutations, cache operations |
| Feature hook | Permission, feature flag, filters, UI feedback and navigation |
| Component | Renders data and calls commands from its hook |

Do not put `fetch` or Axios directly in a resource. A resource consumes a
repository function returning normalized domain data, so the same resource
pattern works with any transport.

## A full CRUD module

### 1. Write the repository

```ts
// products.repository.ts
export type Product = { id: string; name: string; price: number };
export type ProductPage = { items: Product[]; nextCursor?: string };

export const productRepository = {
  async list(
    input: { q: string; cursor?: string },
    signal?: AbortSignal,
  ): Promise<ProductPage> {
    const params = new URLSearchParams({ q: input.q });
    if (input.cursor) params.set("cursor", input.cursor);

    const response = await fetch(`/api/products?${params}`, { signal });
    if (!response.ok) throw new Error("Could not load products");
    return response.json() as Promise<ProductPage>;
  },

  async detail(id: string, signal?: AbortSignal): Promise<Product> {
    const response = await fetch(`/api/products/${id}`, { signal });
    if (!response.ok) throw new Error("Could not load product");
    return response.json() as Promise<Product>;
  },

  async create(input: { name: string; price: number }): Promise<Product> {
    const response = await fetch("/api/products", {
      method: "POST",
      headers: { "content-type": "application/json" },
      body: JSON.stringify(input),
    });
    if (!response.ok) throw new Error("Could not create product");
    return response.json() as Promise<Product>;
  },

  async update(input: {
    id: string;
    patch: Partial<Pick<Product, "name" | "price">>;
  }): Promise<Product> {
    const response = await fetch(`/api/products/${input.id}`, {
      method: "PATCH",
      headers: { "content-type": "application/json" },
      body: JSON.stringify(input.patch),
    });
    if (!response.ok) throw new Error("Could not update product");
    return response.json() as Promise<Product>;
  },

  async remove(id: string): Promise<void> {
    const response = await fetch(`/api/products/${id}`, { method: "DELETE" });
    if (!response.ok) throw new Error("Could not delete product");
  },
};
```

For Axios, only the repository changes:

```ts
const page = await axios.get<ProductPage>("/api/products", {
  params: input,
  signal,
});
return page.data;
```

### 2. Declare capabilities in the resource

```ts
// products.resource.ts
import {
  createResource,
  defineInfiniteQuery,
  defineMutation,
  defineQuery,
} from "@dangminhdev/query-resource";
import { productRepository } from "./products.repository";

type Scope = { organizationId: string };

export const productResource = createResource<Scope>()({
  namespace: ["my-app"],
  name: "products",
  scopeKey: (scope) => ["organization", scope.organizationId],
  queries: {
    detail: defineQuery({
      inputKey: (input: { id: string }) => [input.id],
      queryFn: ({ input, signal }) =>
        productRepository.detail(input.id, signal),
    }),
  },
  infiniteQueries: {
    list: defineInfiniteQuery({
      inputKey: (input: { q: string }) => [input],
      initialPageParam: undefined as string | undefined,
      queryFn: ({ input, pageParam, signal }) =>
        productRepository.list({ ...input, cursor: pageParam }, signal),
      getNextPageParam: (page) => page.nextCursor,
    }),
  },
  mutations: {
    create: defineMutation({
      mutationFn: ({ variables }) => productRepository.create(variables),
      invalidates: [{ type: "infinite", operation: "list" }],
    }),
    update: defineMutation({
      mutationFn: ({ variables }) => productRepository.update(variables),
      invalidates: [
        { type: "query", operation: "detail" },
        { type: "infinite", operation: "list" },
      ],
    }),
    remove: defineMutation({
      mutationFn: ({ variables }: { variables: { id: string } }) =>
        productRepository.remove(variables.id),
      invalidates: [
        { type: "query", operation: "detail" },
        { type: "infinite", operation: "list" },
      ],
    }),
  },
});
```

The names are domain commands, not a fixed CRUD vocabulary. `approve`,
`cancel`, `checkIn`, and `markRead` are valid mutation names.

### 3. Consume it from a feature hook

```tsx
import { useInfiniteQuery, useMutation, useQueryClient } from "@tanstack/react-query";
import { productResource } from "./products.resource";

export function useProducts(organizationId: string, q: string) {
  const client = useQueryClient();
  const products = productResource.bind({ organizationId });

  const list = useInfiniteQuery(products.infiniteQueries.list.options({ q }));
  const create = useMutation(
    products.mutations.create.options({
      optimistic: ({ client, cache, variables }) =>
        cache.infiniteQueries.list.patchAll(client, (current) => ({
          ...current,
          pages: current.pages.map((page, index) =>
            index === 0
              ? {
                  ...page,
                  items: [
                    { id: `optimistic-${Date.now()}`, ...variables },
                    ...page.items,
                  ],
                }
              : page,
          ),
        })),
      onSuccess: ({ data, client, cache }) => {
        cache.queries.detail.set(client, { id: data.id }, data);
      },
    }),
  );

  return { list, create, client };
}
```

`optimistic` may return a rollback returned by `patch` or `patchAll`. On a
mutation error, the resource runs that rollback automatically. If the cache was
updated again in the meantime, it invalidates instead of overwriting newer data.

## Partial modules are first-class

Do not invent empty CRUD methods. The inferred API exposes only what is
declared.

### Read-only

```ts
const countryResource = createResource<void>()({
  namespace: ["my-app"],
  name: "countries",
  scopeKey: () => ["public"],
  queries: {
    all: defineQuery({
      inputKey: (_input: undefined) => [],
      queryFn: ({ signal }) => countryRepository.list(signal),
    }),
  },
});

countryResource.bind(undefined).queries.all.options(undefined);
// No `mutations.create` exists here.
```

### Create-only

```ts
const importResource = createResource<{ tenantId: string }>()({
  namespace: ["my-app"],
  name: "imports",
  scopeKey: (scope) => ["tenant", scope.tenantId],
  mutations: {
    upload: defineMutation({
      mutationFn: ({ scope, variables }: {
        scope: { tenantId: string };
        variables: FormData;
      }) => importRepository.upload(scope.tenantId, variables),
    }),
  },
});
```

## Cache and server helpers

After `bind(scope)`, normal and infinite queries provide:

```ts
const products = productResource.bind({ organizationId: "org-1" });

products.infiniteQueries.list.prefetch(client, { q: "chair" });
products.queries.detail.ensure(client, { id: "p-1" });
products.queries.detail.invalidate(client, { id: "p-1" });
products.queries.detail.invalidateAll(client);
products.queries.detail.set(client, { id: "p-1" }, product);
products.queries.detail.patch(client, { id: "p-1" }, (current) =>
  current ? { ...current, name: "Renamed" } : current,
);
```

This is suitable for route prefetching and SSR because these helpers receive an
explicit `QueryClient`; the package never creates a global client.

## Key rules

Generated keys follow this shape:

```text
[namespace, scopeKey(scope), resourceName, capability, operation, inputKey(input)]
```

- Put every response-affecting value in `scopeKey` or `inputKey`.
- Keep keys JSON-serializable and inputs immutable after creating options.
- Use `defineQuery` for a normal page/offset query and `defineInfiniteQuery`
  for cursor or bidirectional pagination.
- `invalidates` can target only operations declared in the same resource. Use a
  feature hook to coordinate effects across resources.

## Development and publishing

```bash
npm install
npm test
npm run pack:check
```

Publish a public scoped package after creating an npm account and enabling 2FA:

```bash
npm login
npm whoami
npm publish
```

`publishConfig.access` is already set to `public`. Before every release, change
the version with `npm version patch`, `npm version minor`, or `npm version major`.
The `prepublishOnly` script runs tests and checks the tarball contents before
npm uploads it.

## License

[MIT](LICENSE)
