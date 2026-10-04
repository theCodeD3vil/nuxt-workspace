---
name: write-tests
description: Write tests for existing or new code, following the project's test framework and conventions. Use when the user asks for tests or coverage.
---

# write-tests

Write tests for the project using Vitest and `@nuxt/test-utils`.

## Before writing

1. Check `package.json` and `vitest.config.ts` for the existing setup. Follow it. Do not introduce a second test framework.
2. Look at nearby existing tests and match their naming, structure, and style.
3. If no setup exists, install `@nuxt/test-utils`, `vitest`, `@vue/test-utils`, and `happy-dom`, and ask before changing config.

## Scope: critical paths only

Tests must cover critical paths and behavior only. Depth over coverage.

- Test what would hurt if it broke: core user flows, business logic, auth, payments, data writes, and API contracts.
- Do not write tests for trivial code: simple getters, pass-through wrappers, static components, config, or type-only files.
- Do not write tests for small helpers unless a critical path depends on their logic.
- Do not write tests for edge cases that rarely occur in practice.
- Aim for under 100 tests in the whole codebase. Check the existing test count before adding more. If near the limit, replace low-value tests instead of adding.
- Prefer a few thorough tests of a critical path over many shallow tests. Do not chase a coverage percentage.
- When unsure whether something deserves a test, skip it and say so in the report.

## Pick the right kind of test

| Code under test | Test type | Environment |
|---|---|---|
| `utils/`, `lib/`, `server/utils/`, pure functions | Unit | `node` |
| Composables using Nuxt auto-imports, `useState`, `useRuntimeConfig`, etc. | Nuxt unit | `nuxt` |
| Components | Component | `nuxt` |
| `server/api`, `server/routes` | Endpoint (e2e) | `setup()` + `$fetch` |
| Full pages and flows | E2E | `@nuxt/test-utils/e2e` or Playwright |

Use the cheapest test that proves the behavior. Prefer unit over component over e2e.

## Vitest projects

Split environments with Vitest projects so each test runs in the right one:

```ts
// vitest.config.ts
import { defineConfig } from 'vitest/config'
import { defineVitestProject } from '@nuxt/test-utils/config'

export default defineConfig({
  test: {
    projects: [
      {
        test: {
          name: 'unit',
          include: ['test/unit/**/*.{test,spec}.ts'],
          environment: 'node',
        },
      },
      await defineVitestProject({
        test: {
          name: 'nuxt',
          include: ['test/nuxt/**/*.{test,spec}.ts'],
          environment: 'nuxt',
        },
      }),
      {
        test: {
          name: 'e2e',
          include: ['test/e2e/**/*.{test,spec}.ts'],
          environment: 'node',
        },
      },
    ],
  },
})
```

Alternative: a single config where individual files opt in with a `// @vitest-environment nuxt` comment, or use the `.nuxt.spec.ts` suffix.

## File layout

- Unit and component tests: `test/unit/`, `test/nuxt/`, mirroring the source path (`utils/format.ts` -> `test/unit/utils/format.test.ts`).
- E2E tests: `test/e2e/`.
- Name files `<name>.test.ts`.

## Unit tests (utils, lib, pure logic)

```ts
import { describe, it, expect } from 'vitest'
import { formatPrice } from '~/utils/format'

describe('formatPrice', () => {
  it('formats cents as dollars', () => {
    expect(formatPrice(1250)).toBe('$12.50')
  })
})
```

## Composables (Nuxt environment)

```ts
import { describe, it, expect } from 'vitest'
import { mockNuxtImport } from '@nuxt/test-utils/runtime'

mockNuxtImport('useRuntimeConfig', () => () => ({ public: { apiBase: '/test' } }))

describe('useApi', () => {
  it('uses the configured base url', () => {
    expect(useApi().baseUrl).toBe('/test')
  })
})
```

- Call composables inside the Nuxt environment so auto-imports and `useNuxtApp()` work.
- Use `mockNuxtImport` to mock auto-imported composables. It is hoisted, so only one mock per import per file.
- Use `registerEndpoint('/api/x', () => ({ ... }))` to mock network calls made by `useFetch` or `$fetch`.

## Components

```ts
import { describe, it, expect } from 'vitest'
import { mountSuspended } from '@nuxt/test-utils/runtime'
import UserCard from '~/components/UserCard.vue'

describe('UserCard', () => {
  it('renders the name', async () => {
    const wrapper = await mountSuspended(UserCard, { props: { name: 'Ada' } })
    expect(wrapper.text()).toContain('Ada')
  })

  it('emits select on click', async () => {
    const wrapper = await mountSuspended(UserCard, { props: { name: 'Ada' } })
    await wrapper.find('button').trigger('click')
    expect(wrapper.emitted('select')).toHaveLength(1)
  })
})
```

- Use `mountSuspended` (not plain `mount`) so the component gets the Nuxt context, plugins, and async setup.
- Use `renderSuspended` with `@testing-library/vue` if the project already uses Testing Library.
- Use `mockComponent('Name', ...)` to stub heavy child components.
- Test behavior the user sees: rendered text, emitted events, accessible roles. Do not assert on internal state or implementation details.
- Do not snapshot whole components. Snapshots rot. Assert specific things.

## Server endpoints

```ts
import { describe, it, expect } from 'vitest'
import { setup, $fetch } from '@nuxt/test-utils/e2e'

describe('GET /api/health', async () => {
  await setup({ server: true })

  it('returns ok', async () => {
    expect(await $fetch('/api/health')).toEqual({ status: 'ok' })
  })
})
```

- Prefer extracting logic into `server/utils/` and unit testing it in `node`. Keep endpoint tests for wiring, validation, and status codes.
- Test error paths: invalid input should return the expected `statusCode`.
- Use `$fetch` with `{ ignoreResponseError: true }` or `fetch` to assert on error statuses.

## E2E

- Use `setup({ browser: true })` with `createPage` for browser flows, or Playwright if the project uses it.
- Keep e2e tests few and focused on critical user flows. They are slow.

## Rules

- One behavior per test. Name tests by behavior: `it('rejects an empty email')`.
- Arrange, act, assert. Keep tests short and readable.
- Cover the happy path and the error paths that matter. Skip rare edge cases unless requested.
- Tests must be independent. No shared mutable state or ordering dependence.
- Mock at the boundary (network, time, randomness). Do not mock the code under test.
- Use `vi.useFakeTimers()` for time. Reset mocks with `vi.restoreAllMocks()` in `afterEach`.
- No `any` in tests. Type fixtures.
- No `.only` or `.skip` left behind.
- Do not test the framework or third-party libraries.

## After writing

1. Run the new tests (`npx vitest run <path>`) and confirm they pass.
2. Confirm the tests fail when the behavior breaks. A test that cannot fail is useless.
3. Report which files were added and what is covered. Say what is not covered.
