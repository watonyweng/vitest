---
title: Test Context | Guide
outline: deep
---

# Test Context

Inspired by [Playwright Fixtures](https://playwright.dev/docs/test-fixtures), Vitest's test context allows you to define utils, states, and fixtures that can be used in your tests.

## Usage

The first argument for each test callback is a test context.

```ts
import { it } from 'vitest'

it('should work', (ctx) => {
  // prints name of the test
  console.log(ctx.task.name)
})
```

## Built-in Test Context

#### `context.task`

A readonly object containing metadata about the test.

#### `context.expect`

The `expect` API bound to the current test:

```ts
import { it } from 'vitest'

it('math is easy', ({ expect }) => {
  expect(2 + 2).toBe(4)
})
```

This API is useful for running snapshot tests concurrently because global expect cannot track them:

```ts
import { it } from 'vitest'

it.concurrent('math is easy', ({ expect }) => {
  expect(2 + 2).toMatchInlineSnapshot()
})

it.concurrent('math is hard', ({ expect }) => {
  expect(2 * 2).toMatchInlineSnapshot()
})
```

#### `context.skip`

Skips subsequent test execution and marks test as skipped:

```ts
import { expect, it } from 'vitest'

it('math is hard', ({ skip }) => {
  skip()
  expect(2 + 2).toBe(5)
})
```

## Extend Test Context

Vitest provides two different ways to help you extend the test context.

### `test.extend`

Like [Playwright](https://playwright.dev/docs/api/class-test#test-extend), you can use this method to define your own `test` API with custom fixtures and reuse it anywhere.

For example, we first create `myTest` with two fixtures, `todos` and `archive`.

```ts [my-test.ts]
import { test } from 'vitest'

const todos = []
const archive = []

export const myTest = test.extend({
  todos: async ({}, use) => {
    // setup the fixture before each test function
    todos.push(1, 2, 3)

    // use the fixture value
    await use(todos)

    // cleanup the fixture after each test function
    todos.length = 0
  },
  archive
})
```

Then we can import and use it.

```ts [my-test.test.ts]
import { expect } from 'vitest'
import { myTest } from './my-test.js'

myTest('add items to todos', ({ todos }) => {
  expect(todos.length).toBe(3)

  todos.push(4)
  expect(todos.length).toBe(4)
})

myTest('move items from todos to archive', ({ todos, archive }) => {
  expect(todos.length).toBe(3)
  expect(archive.length).toBe(0)

  archive.push(todos.pop())
  expect(todos.length).toBe(2)
  expect(archive.length).toBe(1)
})
```

We can also add more fixtures or override existing fixtures by extending `myTest`.

```ts
export const myTest2 = myTest.extend({
  settings: {
    // ...
  }
})
```

#### Fixture initialization

Vitest runner will smartly initialize your fixtures and inject them into the test context based on usage.

```ts
import { test } from 'vitest'

async function todosFn({ task }, use) {
  await use([1, 2, 3])
}

const myTest = test.extend({
  todos: todosFn,
  archive: []
})

// todosFn will not run
myTest('', () => {})
myTest('', ({ archive }) => {})

// todosFn will run
myTest('', ({ todos }) => {})
```

::: warning
When using `test.extend()` with fixtures, you should always use the object destructuring pattern `{ todos }` to access context both in fixture function and test function.

```ts
myTest('context must be destructured', (context) => { // [!code --]
  expect(context.todos.length).toBe(2)
})

myTest('context must be destructured', ({ todos }) => { // [!code ++]
  expect(todos.length).toBe(2)
})
```

:::

#### Automatic fixture

Vitest also supports the tuple syntax for fixtures, allowing you to pass options for each fixture. For example, you can use it to explicitly initialize a fixture, even if it's not being used in tests.

```ts
import { test as base } from 'vitest'

const test = base.extend({
  fixture: [
    async ({}, use) => {
      // this function will run
      setup()
      await use()
      teardown()
    },
    { auto: true } // Mark as an automatic fixture
  ],
})

test('works correctly')
```

#### Default fixture

Since Vitest 3, you can provide different values in different [projects](/guide/workspace). To enable this feature, pass down `{ injected: true }` to the options. If the key is not specified in the [project configuration](/config/#provide), then the default value will be used.

:::code-group
```ts [fixtures.test.ts]
import { test as base } from 'vitest'

const test = base.extend({
  url: [
    // default value if "url" is not defined in the config
    '/default',
    // mark the fixture as "injected" to allow the override
    { injected: true },
  ],
})

test('works correctly', ({ url }) => {
  // url is "/default" in "project-new"
  // url is "/full" in "project-full"
  // url is "/empty" in "project-empty"
})
```
```ts [vitest.workspace.ts]
import { defineWorkspace } from 'vitest/config'

export default defineWorkspace([
  {
    test: {
      name: 'project-new',
    },
  },
  {
    test: {
      name: 'project-full',
      provide: {
        url: '/full',
      },
    },
  },
  {
    test: {
      name: 'project-empty',
      provide: {
        url: '/empty',
      },
    },
  },
])
```
:::

#### Scoping Values to Suite <Version>3.1.0</Version> {#scoping-values-to-suite}

Since Vitest 3.1, you can override context values per suite and its children by using the `test.scoped` API:

```ts
import { test as baseTest, describe, expect } from 'vitest'

const test = baseTest.extend({
  dependency: 'default',
  dependant: ({ dependency }, use) => use({ dependency })
})

describe('use scoped values', () => {
  test.scoped({ dependency: 'new' })

  test('uses scoped value', ({ dependant }) => {
    // `dependant` uses the new overriden value that is scoped
    // to all tests in this suite
    expect(dependant).toEqual({ dependency: 'new' })
  })

  describe('keeps using scoped value', () => {
    test('uses scoped value', ({ dependant }) => {
      // nested suite inherited the value
      expect(dependant).toEqual({ dependency: 'new' })
    })
  })
})

test('keep using the default values', ({ dependant }) => {
  // the `dependency` is using the default
  // value outside of the suite with .scoped
  expect(dependant).toEqual({ dependency: 'default' })
})
```

This API is particularly useful if you have a context value that relies on a dynamic variable like a database connection:

```ts
const test = baseTest.extend<{
  db: Database
  schema: string
}>({
  db: async ({ schema }, use) => {
    const db = await createDb({ schema })
    await use(db)
    await cleanup(db)
  },
  schema: '',
})

describe('one type of schema', () => {
  test.scoped({ schema: 'schema-1' })

  // ... tests
})

describe('another type of schema', () => {
  test.scoped({ schema: 'schema-2' })

  // ... tests
})
```

#### TypeScript

To provide fixture types for all your custom contexts, you can pass the fixtures type as a generic.

```ts
interface MyFixtures {
  todos: number[]
  archive: number[]
}

const myTest = test.extend<MyFixtures>({
  todos: [],
  archive: []
})

myTest('types are defined correctly', ({ todos, archive }) => {
  expectTypeOf(todos).toEqualTypeOf<number[]>()
  expectTypeOf(archive).toEqualTypeOf<number[]>()
})
```

### `beforeEach` and `afterEach`

The contexts are different for each test. You can access and extend them within the `beforeEach` and `afterEach` hooks.

```ts
import { beforeEach, it } from 'vitest'

beforeEach(async (context) => {
  // extend context
  context.foo = 'bar'
})

it('should work', ({ foo }) => {
  console.log(foo) // 'bar'
})
```

#### TypeScript

To provide property types for all your custom contexts, you can aggregate the `TestContext` type by adding

```ts
declare module 'vitest' {
  export interface TestContext {
    foo?: string
  }
}
```

If you want to provide property types only for specific `beforeEach`, `afterEach`, `it` and `test` hooks, you can pass the type as a generic.

```ts
interface LocalTestContext {
  foo: string
}

beforeEach<LocalTestContext>(async (context) => {
  // typeof context is 'TestContext & LocalTestContext'
  context.foo = 'bar'
})

it<LocalTestContext>('should work', ({ foo }) => {
  // typeof foo is 'string'
  console.log(foo) // 'bar'
})
```
