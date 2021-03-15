---
title: "Announcing Vendure v1.0.0-beta"
date: 2021-03-09T13:00:00
draft: false
author: "Michael Bromley"
images: 
    - "/blog/2021/03/announcing-vendure-v1.0.0-beta/banner-01.jpg"
---

It is with great excitement that we announce the release of Vendure v1.0.0-beta.1! This is the first release of the v1.0 pre-release phase, in which we plan no further breaking changes, and only relatively small changes in the run-up to the final v1.0 stable release!

<!--more-->

{{< vimeo id="521373866" >}}

Video sections:

* [Worker improvements](https://vimeo.com/521373866#t=0m31s)
* [Custom field relations](https://vimeo.com/521373866#t=4m19s)
* [Payment improvements](https://vimeo.com/521373866#t=6m18s)
* [Migration](https://vimeo.com/521373866#t=7m43s)
* [Roadmap to final release](https://vimeo.com/521373866#t=8m20s)

## From here to v1.0

The `-beta.1` suffix means that this is not _yet_ the final, stable Vendure v1.0.0 release. There are still some outstanding issues to be handled, and rough edges to smooth out. However, there are no more breaking changes planned between now and v1.0.0. Over the coming weeks we'll be releasing a series of `-beta.n` releases which will include these changes. Just before the final release we'll release a `-rc.1` ("release candidate 1"), and if all is well then that version will then become the final v1.0.0 release.

Future releases during this phase will not be accompanied by blog posts, just changelog entries and announcement via the usual channels (Slack, Twitter).

To recap:

* `1.0.0-beta.1` <- we are here
* `1.0.0-beta.2`
* ...`1.0.0-beta.n` (as needed)
* `1.0.0-rc.1`
* `1.0.0-rc.n` (if needed)
* `1.0.0` <- goal: 1 - 2 months from now

With that established, let's take a look at some of the new features in this release!


## Major improvement of worker architecture

Vendure uses the concept of a "worker" process to process long-running tasks in the background (a pattern known as a _job queue_ or _task queue_). Previously, the worker process was implemented as a [NestJS microservice](https://docs.nestjs.com/microservices/basics). Jobs would be added to a queue in the database, and then the server would typically send those jobs to the worker over the network via TCP using the `WorkerService`. This arrangement had a number of issues:

* The need for network connectivity between the server and worker made deployment more complex
* It was difficult to successfully scale the worker to multiple instances
* The TCP connection was not robust, leading to the possibility of job loss
* Creating plugins which made use of the worker was overly complex and verbose

This release introduces a more efficient and stable architecture which addresses these issues. We have switched from the NestJS microservice-based architecture to a much simpler pure job queue solution, which essentially combines the roles of the `JobQueueService` with the `WorkerService`, so that any job added to a JobQueue will by default run on the worker.

The advantages of this approach are:

* It is now trivial to spin up as many parallel worker processes as you need (see the [demo in the release video](https://vimeo.com/521373866#t=2m24s))
* Less plugin code is needed to run tasks on the worker
* More performant job queue strategies can result in much reduced load on the database (see chart below)
* The internals are simpler - no network layer is needed to run the worker, meaning we no longer have the overhead of sending & receiving network requests to send jobs to the worker

{{< figure src="./worker-load.png" caption="Load reduction on the database after updating to use the new worker architecture and implementing a custom JobQueueStrategy based on Google Cloud Pub/Sub" >}}

This change involved removing a bunch of code and APIs which are made obsolete, as can be seen when we look at the number of new lines vs deleted lines when this change was merged in: 
```shell
$ git diff --shortstat 2693174b dac1aac3
 
99 files changed, 786 insertions(+), 2265 deletions(-)
```

This change is breaking, and the amount of work you'll need to do to migrate depends on how much you made use of the worker in your custom plugins. See the migration guide at the end for full details.

A **huge** thanks is due to community member [Fred Cox](https://github.com/mcfedr) for his outstanding work on this ([PR #663](https://github.com/vendure-ecommerce/vendure/pull/663) & [PR #771](https://github.com/vendure-ecommerce/vendure/pull/771)).

## `relation` custom field type

Custom fields enable the extension of built-in models with data specific to your business use-case. Up until now, only simple data types (string, booleans, etc.) have been supported. With this release, you can now define custom relations between entities in just a single line of configuration!

For example, here's how you can define a relation between `Customer` and `Asset` to allow your customers to upload an avatar image:

```TypeScript
const config: VendureConfig = {
  // ...
  customFields: {
    Customer: [
      { name: 'avatar', type: 'relation', entity: Asset },
    ]
  }
}
```

This then allows you to query the avatar via the GraphQL API like this:

```GraphQL
query {
  customer(id: 1) {
    id
    firstName
    lastName
    customFields {
      avatar {
        id
        name
        preview
      }
    }
  }
}
```

See the newly-expanded [custom fields documentation]({{< relref "customizing-models" >}}) for more details.

## Payment process improvements

A number of improvements have been made to the way payment methods are defined and handled:

* Just like the Order & Fulfillment processes, the Payments process can now be customized, allowing you to define your own states and transitions in the payment process. See the [Payment Integrations guide]({{< relref "payment-integrations" >}}#custom-payment-flows) for an example.
* PaymentMethods are now channel-aware, so if you make use of multiple Channels, you can now limit PaymentMethods to specific Channels.
* You can now define a [PaymentMethodEligibilityChecker]({{< relref "/docs/typescript-api/payment/payment-method-eligibility-checker" >}}) which is used to determine whether a particular PaymentMethod is eligible to be used against the current active Order. This works in much the same way as the existing checkers for ShippingMethods, and accordingly introduces a new query on the Shop API:
    ```GraphQL
    type Query {
      eligiblePaymentMethods: [PaymentMethodQuote!]!
    }
    ```

## Displaying stock level in the Shop API

Up until now, it has not been possible to view stock levels via the Shop API without creating a custom plugin. The reason for this was that by default, we did not want to expose potentially sensitive stock information publicly. However, some kind of indication of whether an item is in or out of stock is an extremely common requirement, so with this release we are introducing a `ProductVariant.stockLevel` field and the accompanying [StockDisplayStrategy]({{< relref "stock-display-strategy" >}}).

The [DefaultStockDisplayStrategy]({{< relref "default-stock-display-strategy" >}}) will return either `'IN_STOCK'`, `'OUT_OF_STOCK'` or `'LOW_STOCK'`, but you can provide your own strategy to expose even more granular data if required (e.g. _"Only 2 items left"_) by setting the `catalogOptions.stockDisplayStrategy` config option. 

## Other notable improvements

* Assets, Facets and FacetValues are now channel-aware, meaning they can be assigned to specific Channels in a multi-Channel configuration.
* The `Administrator` and `Channel` entities now support custom fields.
* The new [ChangedPriceHandlingStrategy]({{< relref "changed-price-handling-strategy" >}}) allows you full control over how you handle changes to the price of items which are already in an active Order.
* The new [OrderPlacedStrategy]({{< relref "order-placed-strategy" >}}) enables even more control over custom Order processes, allowing you to define the exact point at which the Order is considered "placed" (i.e. Customer has checked out, Order no longer active).
* The TaxCategory entity has a new `isDefault` property, which is used to decide on the default TaxCategory to use when creating new ProductVariants.
* Assets can now be tagged. Tags are simple text labels which can be used to classify and group Assets. For example, if you are creating Customer avatars, you can tag them
with an "avatar" tag, which allows you to easily filter for only avatar assets.
* Performance when dealing with very large orders (OrderLines with quantity of > 100) has been massively improved ([#705](https://github.com/vendure-ecommerce/vendure/pull/705))
* All major dependencies (NestJS, TypeORM, Graphql-js, Apollo Server, Angular) have been updated to the latest versions


**📖 See all changes in the [v1.0.0-beta.1 Changelog](https://github.com/vendure-ecommerce/vendure/blob/419761b88c01503208e3b0e779d2a0925926c62b/CHANGELOG.md#100-beta1-2021-03-09)**

## Contributor acknowledgements

The following community members contributed to this release. Thank you for your support and participation in making this the best version of Vendure!

* [Martijn](https://github.com/martijnvdbrug) implemented channel-aware Assets ([#700](https://github.com/vendure-ecommerce/vendure/pull/700))
* [Rohan Rajpal](https://github.com/rohanrajpal) added support for custom fields on the Channel entity ([#670](https://github.com/vendure-ecommerce/vendure/pull/670))
* [Fred Cox](https://github.com/mcfedr) did incredible work on the worker improvements ([#663](https://github.com/vendure-ecommerce/vendure/pull/663) & [#771](https://github.com/vendure-ecommerce/vendure/pull/771))
* [Thomas Blommaert](https://github.com/thomas-advantitge) improved performance of large order modifications plus other fixes ([#705](https://github.com/vendure-ecommerce/vendure/pull/705), [#713](https://github.com/vendure-ecommerce/vendure/pull/713), [#714](https://github.com/vendure-ecommerce/vendure/pull/714))
* [William Milne](https://github.com/WilliamMilne) added support for custom generators & senders in the EmailPlugin ([#707](https://github.com/vendure-ecommerce/vendure/pull/707))
* [Hendrik Depauw](https://github.com/hendrik-advantitge) made fixes to SessionCacheStrategy and the auth resolvers ([#731](https://github.com/vendure-ecommerce/vendure/pull/731), [#748](https://github.com/vendure-ecommerce/vendure/pull/748))
* [Karel Van De Winkel](https://github.com/karel-advantitge) fixed product deletion in the Elasticsearch plugin ([#743](https://github.com/vendure-ecommerce/vendure/pull/743))
* [Drayke](https://github.com/Draykee) improved plugin docs ([#702](https://github.com/vendure-ecommerce/vendure/pull/702))
* [Jean Carlos Farias](https://github.com/jeancx) fixed our Brazilian Portuguese Admin UI translations ([#725](https://github.com/vendure-ecommerce/vendure/pull/725))


---

## BREAKING CHANGES / Migration Guide

{{< alert warning >}}
**🚧 Read this section carefully!**
{{< /alert >}}

👉 For general instructions on upgrading, please see the new [Updating Vendure guide]({{< relref "updating-vendure" >}}).

### Database migration

1. Create a complete backup of your database schema and data.
2. Generate a migration script as described in the [Migrations guide]({{< relref "migrations" >}}).
3. You must now modify the generated migration script in accordance with the examples given in this [Vendure 1.0.0-beta.1 Migration gist](https://gist.github.com/michaelbromley/5edc01ab07b3f2101cc1f0cb3b60e598). This will involve moving around the generated queries and adding calls to the provided utility functions which will deal with the more complex data migrations.
    * [Example migration script](https://gist.github.com/michaelbromley/08cba2ad0101a2901516b6199fa84a00#file-example-mysql-migration-ts) (if using postgres, the syntax will be slightly different, but the sequence of commands should match this example)
    * [Migration utility functions](https://gist.github.com/michaelbromley/08cba2ad0101a2901516b6199fa84a00#file-migration-utils-ts) (copy the contents of this file into the parent directory of the `/migrations` folder)
4. **IMPORTANT** test the migration first on data you are prepared to lose to ensure that it works as expected. Do not run on production data without testing. For production data, **make a full backup first!**


### Update worker-related code

1. The `index-worker.ts` file needs to be updated:
    ```diff
     bootstrapWorker(config)
    +  .then(worker => worker.startJobQueue())
       .catch((err: any) => {
         console.log(err);
         process.exit(1);
       });
   ```
2. The `VendureConfig.workerOptions` config object has been removed. Since Vendure no longer uses TCP to connect to the Worker, these options are not needed. If you currently rely on the `WorkerOptions.runInMainProcess` setting, this can be replaced by [this solution]({{< relref "vendure-worker" >}}#running-jobs-on-the-main-process).
3. If you have any custom plugins which use the worker (making use of the `WorkerService` and controllers with `@MessagePattern()` decorator) then you'll need to make some substantial changes: Specialized controllers are no longer needed, and the `workers: []` array of the VendurePlugin metadata has been removed. Instead, the work previously performed in the controller can now be placed in a regular service.
    ```diff
    - @Controller()
    - class OrderProcessingController {
    -   @MessagePattern(ProcessOrderMessage.pattern)
    -   async processOrder(orderId: ProcessOrderMessage['data']) {
    -      // long-running async work
    -   }
    - }
   
      class OrderProcessingService {
         private jobQueue: JobQueue<ID>;
      
         constructor(
    -      private workerService: WorkerService,
           private jobQueueService: JobQueueService,
         ) {
           this.jobQueue = this.jobQueueService.createQueue({
             name: ['process-order-analytics'],
    -        concurrency: 1,
             process: async job => {
    -          this.workerService
    -            .send(new ProcessOrderMessage(job.data.id)).subscribe();      
    +          return this.processOrder(job.data.id);
             }
         }
   
        addOrderJobToQueue(order: Order) {
          return this.jobQueue.add(order.id);
        }
   
    +   async processOrder(orderId: ID) {
    +     // long-running async work
    +   }
      }
     
      @VendurePlugin({
        imports: [PluginCommonModule],
        providers: [OrderProcessingService],
    -   workers: [OrderProcessingController],
      })
      export class OrderAnalyticsPlugin {}
    ```

### Update EmailPlugin, AssetServerPlugin config
 
 The EmailPlugin & AssetServerPlugin now run directly as part of the main Vendure server process. This means that you do not need to specify a `port` for them to run on. These plugins also require an explicit `route` to be specified:
 
 ```diff
  plugins: [
    DefaultJobQueuePlugin,
    AssetServerPlugin.init({
      route: 'vendure-assets',
      assetUploadDir: path.join(__dirname, '../static/assets'),
 -    port: 3001,
    }),
    EmailPlugin.init({
 +    route: 'mailbox',
      handlers: defaultEmailHandlers,
      templatePath: path.join(__dirname, '../static/email/templates'),
      outputPath: path.join(__dirname, '../static/email/output'),
 -    mailboxPort: 3003,
      devMode: true,
      globalTemplateVars: {
        // ...
      },
    }),
    AdminUiPlugin.init({
      port: 3002,
 +    route: 'admin',
    }),
  ],
 ```

### Replace Vendure-specific plugin lifecycle hooks

If your plugins make use of the Vendure-specific lifecycle hooks, you will need to replace them with the [standard NestJS hooks](https://docs.nestjs.com/fundamentals/lifecycle-events#lifecycle-events-1):

Vendure hook | Replace with
-------------|--------------
beforeVendureBootstrap | [configure](https://docs.nestjs.com/middleware#applying-middleware)
beforeVendureWorkerBootstrap | [configure](https://docs.nestjs.com/middleware#applying-middleware)
onVendureBootstrap | onApplicationBootstrap
onVendureWorkerBootstrap | onApplicationBootstrap
onVendureClose | onModuleDestroy
onVendureWorkerClose | onModuleDestroy


### Update e2e tests

If you have e2e test which rely on adding payments to an Order, you will need to update the InitialData object which you pass to the `TestServer.init()` method. This is because Vendure no longer automatically creates a PaymentMethod for each PaymentMethodHandler passed into the config. Instead, you need to specify the creation of a new PaymentMethod via the InitialData:

```diff
 describe('my plugin', () => {
   const { server, adminClient, shopClient } = createTestEnvironment({
     ...testConfig,
     plugins: [MyPlugin],
     paymentOptions: {
       paymentMethodHandlers: [testPaymentMethod],
     },
   });
 
   beforeAll(async () => {
     await server.init({
-      initialData,
+      initialData: {
+        ...initialData,
+        paymentMethods: [
+          {
+            name: testPaymentMethod.code,
+            handler: {
+              code: testPaymentMethod.code,
+              arguments: [],
+            },
+          },
+        ],
+      },
       productsCsvPath: PRODUCTS_CSV_PATH,
       customerCount: 1,
     });
     await adminClient.asSuperAdmin();
   }, TEST_SETUP_TIMEOUT_MS);
   
   // ...
 }
```
