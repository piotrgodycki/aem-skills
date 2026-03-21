---
title: Testing React Components in AEM
impact: HIGH
impactDescription: Untested AEM React components break silently in author mode and across environments — tests catch integration issues early
tags: react, testing, jest, testing-library, msw, storybook, aem-mocks, spa-editor
---

## Testing React Components in AEM

Testing patterns for AEM React SPAs — Testing Library for component behavior, mocking ModelManager and SPA Editor internals, hook testing, MSW for GraphQL, and Storybook for visual development.

---

### 1. Testing Library Setup for AEM

```typescript
// test/setup.ts
import '@testing-library/jest-dom';

// Mock AEM SPA Editor
jest.mock('@adobe/aem-spa-page-model-manager', () => ({
  ModelManager: {
    initialize: jest.fn().mockResolvedValue({}),
    initializeAsync: jest.fn().mockResolvedValue({}),
    getData: jest.fn().mockResolvedValue({}),
    addListener: jest.fn(),
  },
}));

// Mock WCM mode meta tag
function setWcmMode(mode: 'EDIT' | 'PREVIEW' | 'DISABLED' = 'DISABLED') {
  const meta = document.createElement('meta');
  meta.setAttribute('name', 'cq:wcmmode');
  meta.setAttribute('content', mode);
  document.head.appendChild(meta);
}

// Make available globally
(globalThis as any).setWcmMode = setWcmMode;
```

---

### 2. Component Tests

```tsx
// features/hero/__tests__/Hero.test.tsx
import { render, screen } from '@testing-library/react';
import { Hero } from '../Hero';

const defaultProps = {
  title: 'Welcome to Our Site',
  subtitle: 'Discover amazing content',
  backgroundImage: '/content/dam/mysite/hero.jpg',
  ctaLabel: 'Learn More',
  ctaLink: '/en/about',
  overlayOpacity: 0.4,
  cqPath: '/content/mysite/en/jcr:content/root/hero',
  isInEditor: false,
};

describe('Hero', () => {
  it('renders title and subtitle', () => {
    render(<Hero {...defaultProps} />);

    expect(screen.getByRole('heading', { level: 1 })).toHaveTextContent(
      'Welcome to Our Site'
    );
    expect(screen.getByText('Discover amazing content')).toBeInTheDocument();
  });

  it('renders CTA link with correct href', () => {
    render(<Hero {...defaultProps} />);

    const cta = screen.getByRole('link', { name: 'Learn More' });
    expect(cta).toHaveAttribute('href', '/en/about');
  });

  it('hides CTA when no label or link', () => {
    render(<Hero {...defaultProps} ctaLabel="" ctaLink="" />);

    expect(screen.queryByRole('link')).not.toBeInTheDocument();
  });

  it('applies background image', () => {
    const { container } = render(<Hero {...defaultProps} />);

    const section = container.querySelector('.cmp-hero');
    expect(section).toHaveStyle({
      backgroundImage: 'url(/content/dam/mysite/hero.jpg)',
    });
  });

  it('applies custom overlay opacity', () => {
    const { container } = render(<Hero {...defaultProps} overlayOpacity={0.7} />);

    const overlay = container.querySelector('.cmp-hero__overlay');
    expect(overlay).toHaveStyle({ opacity: 0.7 });
  });

  it('uses custom heading level', () => {
    render(<Hero {...defaultProps} titleType="h2" />);

    expect(screen.getByRole('heading', { level: 2 })).toHaveTextContent(
      'Welcome to Our Site'
    );
  });
});
```

---

### 3. Testing Editable Components (Author Mode)

```tsx
// features/hero/__tests__/Hero.editMode.test.tsx
import { render, screen } from '@testing-library/react';
import { Hero } from '../Hero';

describe('Hero in Author Mode', () => {
  beforeEach(() => {
    (globalThis as any).setWcmMode('EDIT');
  });

  afterEach(() => {
    document.head.querySelector('meta[name="cq:wcmmode"]')?.remove();
  });

  it('shows placeholder when empty in edit mode', () => {
    render(
      <Hero
        title=""
        backgroundImage=""
        cqPath="/content/mysite/en/jcr:content/root/hero"
        isInEditor={true}
      />
    );

    expect(screen.getByText(/drag content here|open the dialog/i)).toBeInTheDocument();
  });

  it('disables animations in author mode', () => {
    render(
      <Hero
        title="Test"
        backgroundImage="/test.jpg"
        cqPath="/content/mysite/en/jcr:content/root/hero"
        isInEditor={true}
      />
    );

    const heading = screen.getByRole('heading');
    expect(heading).not.toHaveClass('cmp-hero__title--animated');
  });
});
```

---

### 4. Hook Testing

```typescript
// shared/hooks/__tests__/useBreakpoint.test.ts
import { renderHook, act } from '@testing-library/react';
import { useBreakpoint } from '../useBreakpoint';

describe('useBreakpoint', () => {
  const originalMatchMedia = window.matchMedia;

  function mockMatchMedia(width: number) {
    window.matchMedia = jest.fn().mockImplementation((query: string) => ({
      matches:
        (query.includes('max-width: 767px') && width <= 767) ||
        (query.includes('min-width: 768px') && query.includes('max-width: 1199px') && width >= 768 && width <= 1199) ||
        (query.includes('min-width: 1200px') && width >= 1200),
      addEventListener: jest.fn(),
      removeEventListener: jest.fn(),
    }));
  }

  afterEach(() => {
    window.matchMedia = originalMatchMedia;
  });

  it('returns mobile for small viewports', () => {
    mockMatchMedia(375);
    const { result } = renderHook(() => useBreakpoint());
    expect(result.current).toBe('mobile');
  });

  it('returns tablet for medium viewports', () => {
    mockMatchMedia(768);
    const { result } = renderHook(() => useBreakpoint());
    expect(result.current).toBe('tablet');
  });

  it('returns desktop for large viewports', () => {
    mockMatchMedia(1440);
    const { result } = renderHook(() => useBreakpoint());
    expect(result.current).toBe('desktop');
  });
});
```

---

### 5. MSW for GraphQL Mocking

Mock AEM GraphQL persisted queries at the network level:

```typescript
// test/mocks/handlers.ts
import { http, HttpResponse } from 'msw';

export const handlers = [
  // Mock persisted query
  http.get('*/graphql/execute.json/mysite/articles-by-category*', ({ request }) => {
    const url = new URL(request.url);
    return HttpResponse.json({
      data: {
        articleList: {
          items: [
            { _path: '/content/dam/mysite/articles/article-1', title: 'Article 1', slug: 'article-1' },
            { _path: '/content/dam/mysite/articles/article-2', title: 'Article 2', slug: 'article-2' },
          ],
        },
      },
    });
  }),

  // Mock Content Fragment API
  http.get('*/api/assets/content/dam/mysite/articles/*.json', ({ params }) => {
    return HttpResponse.json({
      properties: {
        elements: {
          title: { value: 'Mock Article' },
          body: { value: '<p>Mock body content</p>' },
        },
      },
    });
  }),
];

// test/mocks/server.ts
import { setupServer } from 'msw/node';
import { handlers } from './handlers';

export const server = setupServer(...handlers);

// test/setup.ts
import { server } from './mocks/server';

beforeAll(() => server.listen());
afterEach(() => server.resetHandlers());
afterAll(() => server.close());
```

**Using in tests:**

```tsx
// features/articles/__tests__/ArticleList.test.tsx
import { render, screen, waitFor } from '@testing-library/react';
import { QueryClient, QueryClientProvider } from '@tanstack/react-query';
import { ArticleList } from '../ArticleList';

function renderWithProviders(ui: React.ReactElement) {
  const queryClient = new QueryClient({
    defaultOptions: { queries: { retry: false } },
  });
  return render(
    <QueryClientProvider client={queryClient}>{ui}</QueryClientProvider>
  );
}

describe('ArticleList', () => {
  it('renders articles from GraphQL', async () => {
    renderWithProviders(<ArticleList category="tech" />);

    await waitFor(() => {
      expect(screen.getByText('Article 1')).toBeInTheDocument();
      expect(screen.getByText('Article 2')).toBeInTheDocument();
    });
  });
});
```

---

### 6. Storybook for AEM Components

```tsx
// features/hero/Hero.stories.tsx
import type { Meta, StoryObj } from '@storybook/react';
import { Hero } from './Hero';

const meta: Meta<typeof Hero> = {
  title: 'AEM Components/Hero',
  component: Hero,
  argTypes: {
    titleType: {
      control: 'select',
      options: ['h1', 'h2', 'h3'],
    },
    overlayOpacity: {
      control: { type: 'range', min: 0, max: 1, step: 0.1 },
    },
  },
  args: {
    cqPath: '/content/mysite/en/jcr:content/root/hero',
    isInEditor: false,
  },
};

export default meta;
type Story = StoryObj<typeof Hero>;

export const Default: Story = {
  args: {
    title: 'Welcome to Our Site',
    subtitle: 'Discover amazing content',
    backgroundImage: 'https://picsum.photos/1920/600',
    ctaLabel: 'Learn More',
    ctaLink: '/about',
    overlayOpacity: 0.4,
  },
};

export const NoCTA: Story = {
  args: {
    title: 'Informational Hero',
    backgroundImage: 'https://picsum.photos/1920/600',
    overlayOpacity: 0.6,
  },
};

export const AuthorEditMode: Story = {
  args: {
    title: '',
    backgroundImage: '',
    isInEditor: true,
  },
};
```

---

### 7. Integration Test with Playwright

```typescript
// e2e/hero.spec.ts
import { test, expect } from '@playwright/test';

test.describe('Hero Component', () => {
  test('renders hero on publish', async ({ page }) => {
    await page.goto('/content/mysite/en.html');

    const hero = page.locator('.cmp-hero');
    await expect(hero).toBeVisible();
    await expect(hero.locator('.cmp-hero__title')).toHaveText('Welcome');
  });

  test('hero CTA navigates correctly', async ({ page }) => {
    await page.goto('/content/mysite/en.html');

    await page.click('.cmp-hero__cta');
    await expect(page).toHaveURL(/\/about/);
  });
});
```

---

### Anti-Patterns

- **Testing implementation details** — test behavior (what the user sees), not internal state or class names
- **Not mocking ModelManager** — tests hang waiting for AEM server
- **Snapshot-only testing** — snapshots don't catch behavioral regressions
- **Skipping author mode tests** — bugs surface only when authors use the page editor
- **Testing with real AEM endpoints** — flaky tests, environment coupling; use MSW
- **No error state testing** — only testing the happy path; test loading, error, and empty states
- **Sharing QueryClient across tests** — cached data leaks between tests; create a fresh client per test
