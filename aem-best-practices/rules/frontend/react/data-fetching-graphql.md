---
title: Data Fetching & GraphQL for AEM React
impact: HIGH
impactDescription: Wrong fetching patterns bypass CDN caching, cause waterfalls, and break ISR — degrading both performance and authoring experience
tags: react, graphql, aem-headless-sdk, data-fetching, swr, tanstack-query, isr, nextjs, persisted-queries
---

## Data Fetching & GraphQL for AEM React

Production patterns for fetching AEM content in React applications — AEM Headless SDK, persisted GraphQL queries, TanStack Query/SWR integration, pagination, error handling, and Next.js ISR.

---

### 1. AEM Headless SDK Setup

```typescript
// shared/utils/aem-headless.ts
import AEMHeadless from '@adobe/aem-headless-client-js';

const aemHost = process.env.REACT_APP_AEM_HOST || '';
const aemToken = process.env.REACT_APP_AEM_TOKEN;

export const aemClient = new AEMHeadless({
  serviceURL: aemHost,
  endpoint: '/content/cq:graphql/mysite/endpoint.json',
  auth: aemToken ? `Bearer ${aemToken}` : undefined,
});

// Type-safe wrapper for persisted queries
export async function runPersistedQuery<T>(
  queryPath: string,
  variables?: Record<string, unknown>
): Promise<T> {
  const response = await aemClient.runPersistedQuery(queryPath, variables);
  return response.data as T;
}
```

---

### 2. TanStack Query Integration

```typescript
// shared/hooks/useAemQuery.ts
import { useQuery, type UseQueryOptions } from '@tanstack/react-query';
import { runPersistedQuery } from '@shared/utils/aem-headless';

export function useAemQuery<T>(
  queryPath: string,
  variables?: Record<string, unknown>,
  options?: Omit<UseQueryOptions<T, Error>, 'queryKey' | 'queryFn'>
) {
  return useQuery<T, Error>({
    queryKey: ['aem', queryPath, variables],
    queryFn: () => runPersistedQuery<T>(queryPath, variables),
    staleTime: 5 * 60 * 1000,     // 5 minutes
    gcTime: 30 * 60 * 1000,       // 30 minutes (formerly cacheTime)
    retry: 2,
    ...options,
  });
}

// Usage:
interface ArticleListResult {
  articleList: {
    items: Array<{ _path: string; title: string; slug: string }>;
  };
}

function ArticleList({ category }: { category: string }) {
  const { data, isLoading, error } = useAemQuery<ArticleListResult>(
    'mysite/articles-by-category',
    { category },
    { enabled: !!category }
  );

  if (isLoading) return <Spinner />;
  if (error) return <ErrorMessage error={error} />;

  return (
    <ul>
      {data?.articleList.items.map(article => (
        <li key={article._path}>
          <a href={`/articles/${article.slug}`}>{article.title}</a>
        </li>
      ))}
    </ul>
  );
}
```

---

### 3. SWR Alternative

```typescript
// shared/hooks/useAemSWR.ts
import useSWR, { type SWRConfiguration } from 'swr';
import { runPersistedQuery } from '@shared/utils/aem-headless';

export function useAemSWR<T>(
  queryPath: string | null,
  variables?: Record<string, unknown>,
  config?: SWRConfiguration<T, Error>
) {
  const key = queryPath
    ? ['aem', queryPath, JSON.stringify(variables)]
    : null;

  return useSWR<T, Error>(
    key,
    () => runPersistedQuery<T>(queryPath!, variables),
    {
      revalidateOnFocus: false,
      dedupingInterval: 60_000,
      ...config,
    }
  );
}
```

---

### 4. Paginated Queries

**Offset-based pagination:**

```typescript
// features/articles/useArticlesPaginated.ts
import { useAemQuery } from '@shared/hooks/useAemQuery';
import { keepPreviousData } from '@tanstack/react-query';

interface ArticlesPage {
  articlePaginated: {
    items: Array<{ _path: string; title: string; teaser: string }>;
    _references: Array<{ _path: string; _dynamicUrl: string }>;
  };
}

export function useArticlesPaginated(page: number, pageSize = 10) {
  return useAemQuery<ArticlesPage>(
    'mysite/articles-paginated',
    { offset: (page - 1) * pageSize, limit: pageSize },
    { placeholderData: keepPreviousData }
  );
}

// AEM Persisted Query:
// query ArticlesPaginated($offset: Int!, $limit: Int!) {
//   articlePaginated: articleList(
//     offset: $offset
//     limit: $limit
//     sort: "publishDate DESC"
//   ) {
//     items { _path title teaser }
//     _references { ... on ImageRef { _path _dynamicUrl } }
//   }
// }
```

**Cursor-based pagination:**

```typescript
export function useArticlesCursor(cursor?: string) {
  return useAemQuery<ArticlesCursorPage>(
    'mysite/articles-cursor',
    { after: cursor, first: 10 },
    { enabled: true }
  );
}
```

---

### 5. Prefetching for Instant Navigation

```typescript
// Prefetch the next page on hover
import { useQueryClient } from '@tanstack/react-query';
import { runPersistedQuery } from '@shared/utils/aem-headless';

function ArticleCard({ slug, title }: { slug: string; title: string }) {
  const queryClient = useQueryClient();

  const prefetch = () => {
    queryClient.prefetchQuery({
      queryKey: ['aem', 'mysite/article-by-slug', { slug }],
      queryFn: () => runPersistedQuery('mysite/article-by-slug', { slug }),
      staleTime: 5 * 60_000,
    });
  };

  return (
    <a href={`/articles/${slug}`} onMouseEnter={prefetch}>
      {title}
    </a>
  );
}
```

---

### 6. Error Boundary for Data Fetching

```tsx
// shared/components/QueryErrorBoundary.tsx
import { QueryErrorResetBoundary } from '@tanstack/react-query';
import { ErrorBoundary, type FallbackProps } from 'react-error-boundary';

function ErrorFallback({ error, resetErrorBoundary }: FallbackProps) {
  return (
    <div className="cmp-error" role="alert">
      <h3>Content unavailable</h3>
      <p>{error.message}</p>
      <button onClick={resetErrorBoundary}>Retry</button>
    </div>
  );
}

export function QueryErrorBoundary({ children }: { children: React.ReactNode }) {
  return (
    <QueryErrorResetBoundary>
      {({ reset }) => (
        <ErrorBoundary onReset={reset} FallbackComponent={ErrorFallback}>
          {children}
        </ErrorBoundary>
      )}
    </QueryErrorResetBoundary>
  );
}
```

---

### 7. Next.js ISR with AEM

Incremental Static Regeneration for AEM headless content:

```typescript
// pages/articles/[slug].tsx (Next.js Pages Router)
import { runPersistedQuery } from '@shared/utils/aem-headless';

export async function getStaticProps({ params }: { params: { slug: string } }) {
  try {
    const data = await runPersistedQuery<ArticleData>(
      'mysite/article-by-slug',
      { slug: params.slug }
    );

    return {
      props: { article: data.articleBySlug.item },
      revalidate: 60, // ISR: re-validate every 60 seconds
    };
  } catch {
    return { notFound: true };
  }
}

export async function getStaticPaths() {
  const data = await runPersistedQuery<ArticleListData>(
    'mysite/all-article-slugs'
  );

  return {
    paths: data.articleList.items.map(a => ({
      params: { slug: a.slug },
    })),
    fallback: 'blocking', // SSR on first request, then cache
  };
}
```

**Next.js App Router (Server Components):**

```typescript
// app/articles/[slug]/page.tsx
import { runPersistedQuery } from '@shared/utils/aem-headless';

export const revalidate = 60; // ISR

export default async function ArticlePage({
  params,
}: {
  params: { slug: string };
}) {
  const data = await runPersistedQuery<ArticleData>(
    'mysite/article-by-slug',
    { slug: params.slug }
  );

  const article = data.articleBySlug.item;

  return (
    <article>
      <h1>{article.title}</h1>
      <div dangerouslySetInnerHTML={{ __html: article.body.html }} />
    </article>
  );
}
```

---

### 8. Multi-Environment Configuration

```typescript
// shared/utils/aem-config.ts
interface AemConfig {
  host: string;
  graphqlEndpoint: string;
  token?: string;
}

const configs: Record<string, AemConfig> = {
  development: {
    host: 'http://localhost:4502',
    graphqlEndpoint: '/content/cq:graphql/mysite/endpoint.json',
  },
  staging: {
    host: 'https://author-p1234-e5678.adobeaemcloud.com',
    graphqlEndpoint: '/content/cq:graphql/mysite/endpoint.json',
    token: process.env.AEM_STAGING_TOKEN,
  },
  production: {
    host: 'https://publish-p1234-e9012.adobeaemcloud.com',
    graphqlEndpoint: '/content/cq:graphql/mysite/endpoint.json',
    token: process.env.AEM_PROD_TOKEN,
  },
};

export const aemConfig = configs[process.env.NODE_ENV || 'development'];
```

---

### Anti-Patterns

- **POST queries in production** — always use persisted queries (GET) so CDN can cache them
- **Fetching in useEffect without caching** — causes duplicate requests; use TanStack Query or SWR
- **No error boundaries** — one failed query crashes the entire SPA
- **Hardcoded AEM URLs** — breaks across dev/stage/prod environments
- **Fetching in components instead of hooks** — logic duplication, no reuse
- **Ignoring `staleTime`** — default `staleTime: 0` re-fetches on every mount
- **Not using `placeholderData` for pagination** — causes layout shift between pages
- **Fetching author-tier in production** — publish tier has CDN caching; author does not
