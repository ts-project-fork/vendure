---
title: "What's Up With E2E Tests?"
date: 2021-03-18T08:00:00
draft: false
author: "Michael Bromley"
images: 
    - "/blog/2021/03/whats-up-with-e2e-tests/banner.jpg"
---

End-to-end tests are slow, unreliable, and a pain to maintain, right? That's the reason they occupy the smallest slice of the testing pyramid. So why does Vendure currently have ~1000 e2e tests, vs ~500 unit tests?

Let's take a look at what's up with e2e tests, how testing an API differs from testing a UI, and strategies to maximise the value of e2e tests!

{{< figure src="./banner.jpg" caption="Photo by William Phipps on Unsplash" >}}

## The problem with E2E tests

The Google Testing blogpost [Just Say No to More End-to-End Tests](https://testing.googleblog.com/2015/04/just-say-no-to-more-end-to-end-tests.html) discusses the problems associated with e2e testing in detail. They sum it up as:

property              | unit tests | e2e tests
----------------------|------------|----------
Fast                  | ✔         | ❌
Reliable              | ✔         | ❌
Isolates Failures     | ✔         | ❌
Simulates a Real User | ❌         | ✔

These trade-offs are distilled into the concept of the "Testing Pyramid": 

{{< figure src="./testing-pyramid.png" caption="The testing pyramid" >}}

The pyramid shape implies you should concentrate your testing strategy on unit tests, and write progressively fewer higher-level tests. 

**So why bother with e2e tests at all if they are so much trouble?** 

Kent C. Dodds, in his article [Static vs Unit vs Integration vs E2E Testing for Frontend Apps](https://kentcdodds.com/blog/unit-vs-integration-vs-e2e-tests) (in which he describes an alternate shape to the pyramid, which he terms the "testing trophy"), uses the term **"confidence coefficient"**, which he defines thus:

> As you move up the testing trophy, you're increasing what I call the "confidence coefficient." This is the relative confidence that each test can get you at that level. You can imagine that above the trophy is manual testing. That would get you really great confidence from those tests, but the tests would be really expensive and slow.

So we are trading time and effort (and money and frustration) for **increased confidence**.

Actual example: An e2e test can pick up a subtle bug introduced by a change to the API middleware layer interacting with some aspect of the database schema definition. Any testing lower on the pyramid has no chance of catching something like that.

Martin Fowler writes about this pyramid and the reasoning behind it in his [TestPyramid article](https://martinfowler.com/bliki/TestPyramid.html). In that article, he also includes this small footnote (my emphasis added):

> 2: The pyramid is based on the assumption that broad-stack tests are expensive, slow, and brittle compared to more focused tests, such as unit tests. While this is usually true, there are exceptions. **If my high level tests are fast, reliable, and cheap to modify - then lower-level tests aren't needed.**

This footnote, hidden way down at the very end of the page, suggests another possible strategy: **fix the problems with e2e tests and reap the benefits.** Have your cake and eat it!

## UI vs API e2e testing

Most articles which discuss the trade-offs of unit vs e2e testing do so in the context of apps with a UI. Vendure, however, has no UI; it is a headless server where the interface is a GraphQL API. 

Indeed, with the current shift towards API-driven architectures (think [Jamstack](https://jamstack.org/), [headless commerce](https://headlesscommerce.org/)), this will be an increasingly common scenario. So how does this change the calculus of our testing strategy?

1. **Stability:** A UI is not static. It can change for reasons of aesthetics, marketing concerns, refactoring, changing underlying libraries etc. Even when it remains visually identical, the structure of the HTML can be changed markedly. All of these changes can result in failing e2e tests, making the tests time-consuming to maintain.

    On the other hand, an API is a public contract that is _expected_ not to change.  Internal refactors should have no effect on the way the API behaves. Thus tests of an API are inherently much more resilient and stable than UI tests.

2. **Test environment:** UI tests require a browser to render the UI, and specialized tools to allow a test script to interact with the UI elements. This introduces both speed and complexity penalties.

    API tests need nothing more than the ability to send and receive HTTP requests - a basic capability of most platforms. This means fewer moving parts. Fewer places for things to break down. Lower resource requirements to run locally and in CI.

These two facts already get us part of the way toward our goal of fixing the issues around e2e testing.

## How Vendure solves e2e testing

Vendure is an e-commerce framework. It provides core functionality and then expects developers to build out their particular business requirements as _plugins_. E-commerce applications must pay particular attention to correctness, so Vendure tries to encourage testing by making it easy to write **fast, reliable, maintainable, easy-to-debug e2e tests**.

### Tooling

Vendure provides the [@vendure/testing package]({{< relref "/docs/developer-guide/testing" >}}) which provides everything you need to start up a real Vendure server, backed by an actual database, populated with data. It also supplies pre-configured test clients which you can use to make requests to this test server.

Every database supported by Vendure can be used in your tests - for the Vendure core itself we run all ~1000 e2e tests against Postgres, MySQL, MariaDB & SQLite on [every push to GitHub](https://github.com/vendure-ecommerce/vendure/actions/workflows/build_and_test.yml).

Setting up a server populated with test data and with pre-configured clients looks like this:
```TypeScript
import { createTestEnvironment, testConfig } from '@vendure/testing';
import { MyPlugin } from '../my-plugin.ts';

describe('my plugin', () => {
  const { server, adminClient, shopClient } = createTestEnvironment({
    ...testConfig,
    plugins: [MyPlugin],
  });
   
  beforeAll(async () => {
    await server.init({
      productsCsvPath: './fixtures/e2e-products.csv',
      initialData: myInitialData,
      customerCount: 2,
    });
    await adminClient.asSuperAdmin();
  });
});
```

### Speed

Perhaps the slowest part of running an e2e test suite is populating the data required by the test. In the case of Vendure tests you'll typically want some products, customers, administrators, and basic configurations set up. 

One strategy we use is that we can cache this initialization data in the form of an SQLite snapshot. Subsequent runs of the test suite load that cached database directly, entirely skipping the need to re-populate the data. After loading the snapshot, all further database operations are performed in-memory only, using [sql.js](https://github.com/sql-js/sql.js).

This optimization typically cuts the time to run a test suite in half, allowing fast feedback during development and even enabling a **test-driven-development approach to e2e testing** (see video below for an example).

### Maintenance

As mentioned earlier, API tests don't need any special tools or frameworks to allow the test scripts to interact with the interface. In Vendure we use Jest (though you can use any similar testing library), and our e2e tests look just like any typical JavaScript test suite:

```TypeScript
import gql from 'graphql-tag';

it('myNewQuery returns the expected result', async () => {
  adminClient.asSuperAdmin();

  const query = gql`
    query MyNewQuery($id: ID!) {
        myNewQuery(id: $id) {
            field1
            field2
        }
    }
  `;
  const result = await adminClient.query(query, { id: 123 });

  expect(result.myNewQuery).toEqual({ /* ... */ })
});
```

For added maintainability, we use [graphql-code-generator](https://www.graphql-code-generator.com/) to generate TypeScript types based on our GraphQL queries. Combined with the statically-typed nature of GraphQL, this means that any even with breaking changes to the API, refactoring our tests remains straightforward.

### Debugging

One of the tradeoffs of e2e vs unit tests highlighted in the Google Testing blog post is how well the test "isolates failures", meaning how hard it is to go from a failing test to discovering and correcting the source of the failure. With UI e2e tests this can indeed be a pain point - the test is running in at least 2 environments - the test script driving the browser, then a separate server running in a different process and perhaps even in an entirely different language.

In Vendure, we are running the server under test in _the same process_ as the test script. This means the test can be run in the Node debugger, with breakpoints set in both the test script _and_ the server source. Switching between the two feels seamless, and logs are unified. Diagnosing a failing test means setting a breakpoint in the failing test, another in the corresponding GraphQL resolver, and then stepping through until the bug is found.

We can even do things like define a spy function in the test script, pass it to the server, and then later in the test script we can assert whether it was called, and with what arguments.

## Conclusion

This all means that, in Vendure, we can have our testing cake _and_ eat it. Unit & integration tests are still useful - we have ~500 of those. But I tend to limit them to pure, algorithmically-complex functions - data parsing and transformation, testing financial calculation logic etc.

Depending on your application, a different mix of tests may be more appropriate. But I hope that I've demonstrated here that, at least in testing APIs, e2e tests can be successfully brought _down the testing pyramid_. You can **boost your confidence coefficient** without increasing your costs.

If you're interested to see how all of this looks in practice, I recorded a screencast (35 min) in which I demonstrate a TDD approach to e2e testing, implementing a couple of new features in a Vendure plugin:

{{< youtube V8xqSXCs5Wk >}}
