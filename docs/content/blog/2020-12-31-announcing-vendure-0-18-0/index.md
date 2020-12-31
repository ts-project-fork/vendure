---
title: "Announcing Vendure v0.18.0"
date: 2020-12-31T14:00:00
draft: false
author: "Michael Bromley"
images: 
    - "/blog/2020/12/announcing-vendure-v0.18.0/vendure-0.18.0-banner-01.jpg"
---

We are excited to announce the release of Vendure v0.18.0! This release brings huge improvements in the handling of taxes, fulfillments, orders and more. This will also be the final release before we move to the v1.0 phase! Watch the video below for more details about the roadmap.

<!--more-->

{{< vimeo id="496041481" >}}

Video sections:

* [Intro](https://vimeo.com/496041481)
* [Tax handling](https://vimeo.com/496041481#t=0m21s)
* [Dashboard widgets](https://vimeo.com/496041481#t=2m57s)
* [Order modification](https://vimeo.com/496041481#t=5m21s)
* [Migration](https://vimeo.com/496041481#t=8m9s)
* [Roadmap to v1.0](https://vimeo.com/496041481#t=12m25s)

## Tax handling overhaul

Taxes are complicated. Designing a system which can accommodate the various tax systems of the world, and their interaction with all possible product types and discount combinations gets _really_ complex.

In [GitHub issue #573](https://github.com/vendure-ecommerce/vendure/issues/573) I set out to solve an incorrect tax calculation when applying a promotion discount. This turned out to be the entrance to a surprisingly deep rabbit hole. The more I researched, the more shortcomings I found with the existing approach. I also better understood the full range of cases that a correct tax system must be able to handle. Furthermore, I discovered that many of the well-known e-commerce frameworks don't get this right!

I decided to do a deep-dive into the way Vendure handles taxes and attempt to get the foundations _correct_. This effort touched on many parts of the framework and unfortunately means that there are more breaking changes than usual. But the tradeoff is that now I'm quite confident that we have a very solid basis to work with, allowing us to **correctly handle a wide range of tax requirements**.

### Standardized API

The GraphQL API had inconsistent naming of prices - on the one hand we had `shipping, shippingWithTax` yet also `subTotalBeforeTax, subTotal`. This release now standardizes the naming of all taxable money fields to the format `<price>, <price>WithTax`.

For example: 

```graphql
query {
  activeOrder {
    ...on Order {
      lines {
        linePrice
        linePriceWithTax
      }
      subTotal
      subTotalWithTax
      shipping
      shippingWithTax
      total
      totalWithTax
    }
  }
}
```

### Correct handling of order-level discounts

Previously, Order-level discounts (e.g. "10% off order") were handled by _adding an adjustment_ to the Order which reduced the total by the desired amount. The problem with this approach is that it is incorrect from a taxation point-of-view. Taxes apply to the goods ordered (the OrderItems), so reducing the Order total _should_ also reduce the amount of tax being applied. 

The problem can be illustrated with an example: A customer places an Order with 2 items - T-Shirt ($10) and Shoes ($40). They apply a coupon code which gives a 50% discount on the whole Order. How much tax should be paid on the items now? Furthermore, if the customer later returns the T-Shirt, how much do we refund?

The way this _should_ be done is to distribute the discount across the entire Order by proportionally reducing the taxable price of _each_ OrderItem in the Order. This is known as "pro-rating" and is how the problem is handled in frameworks such as Magento or Sylius. This approach has now been implemented in Vendure, meaning that in the example above, the pro-rated price of the T-Shirt is now $5 (the $25 discount has been distributed across the 2 items, with the T-Shirt making up 1/5 of the overall Order total, thus receiving 1/5 of the total discount).


### Support for external tax APIs

Since tax can be so complex, many companies choose to use a service such as [Avalara](https://www.avalara.com/) or [TaxJar](https://www.taxjar.com/) to handle the calculation of taxes on orders. Previously there was no easy way to integrate such services with Vendure, since all tax calculations assumed the use of the built-in TaxRate system.

With this release you can now make asynchronous lookups for tax rates via the new [TaxLineCalculationStrategy]({{< relref "tax-line-calculation-strategy" >}}). You can read more about the development of this feature in [GitHub issue #307](https://github.com/vendure-ecommerce/vendure/issues/307)

### Multiple taxes per OrderItem

In some situations, more than one tax may be applicable to a given item. For example, in the USA, an item may be subject to both state-level and city-level sales tax. Such scenarios are now supported since the [TaxLineCalculationStrategy]({{< relref "tax-line-calculation-strategy" >}}) returns an array of [TaxLines]({{< relref "/docs/graphql-api/shop/object-types" >}}#taxline).
 
---

**📖 Read the new [Taxes guide]({{< relref "taxes" >}})** 

## Dashboard widgets

The Admin UI dashboard now supports "widgets" - self-contained components exposing useful information such as recent orders, sales totals, charts, etc.

{{< figure src="./dashboard-widgets.jpg" caption="Dashboard widgets" >}}

Currently, we're just shipping a few simple widgets, but you can start building your own now by following the new [Dashboard Widgets guide]({{< relref "dashboard-widgets" >}}).

Widgets are lazily-loaded, which means that any required JavaScript (such as a 3rd-party charting library) is only loaded if the widgets is displayed. Individual administrators can also customize the layout of their dashboards, which will be persisted to the browser's localStorage. Widgets also support permissions, which means they can be restricted only to Administrators who have the specified permissions.

## Automated Fulfillment creation

Previously, when creating a Fulfillment for an Order, the details (method, tracking code) had to be filled out manually by the administrator. This prevented automation such as the use of an external shipping API to generate tracking codes and even things like shipping labels.

With this release we introduce the concept of [FulfillmentHandlers]({{< relref "fulfillment-handler" >}}), which contain custom code that gets invoked whenever a Fulfillment is created. This code can perform tasks such as calling external APIs, and then return the data required to generate the Fulfillment.

ShippingMethods now have an associated `fulfillmentHandlerCode` property, which is used by the Admin UI to decide which FulfillmentHandler to use when creating a Fulfillment. 

By default, we a manual handler which reproduces the existing behaviour.

To read more about the development of this feature, see [GitHub issue #529](https://github.com/vendure-ecommerce/vendure/issues/529).

## Order modification

Administrators are now able to modify existing orders. This allows them to add new items, alter quantities, add surcharges and edit the shipping and billing addresses.

{{< figure src="./modify-order-01.jpg" >}}

Modifying an Order is done by transitioning to the new `Modifying` state, at which point the new [`modifyOrder` mutation]({{< relref "/docs/graphql-api/admin/mutations" >}}#modifyorder) may be used.

Modifications will often change the price of the Order, meaning that either an additional Payment is required (if the price increases) or a Refund (if the price reduces). Currently both of these must be handled manually (i.e. Vendure will create a Payment or a Refund, but the actual transaction must be performed externally, such as in your payment provider's dashboard). In future we'll be able to integrate these operations more seamlessly. To read more about the development of this feature, see [GitHub issue #314](https://github.com/vendure-ecommerce/vendure/issues/314).

{{< alert warning >}}
**Note:** Order modification turned out to be an extremely complex feature to implement (over 50 hours' work!), and there are sure to be edge-cases which may not behave as expected, particularly to do with how Promotions behave on modified Orders. We advise you to use this feature with caution for now until these edge-cases have been ironed out.
{{< /alert >}}

## Other notable improvements

* **ChannelAware ProductVariants:** The `ChannelAware` interface is used to associate entities with one or more Channels. We're had ChannelAware Products for a while, but this release allows individual ProductVariants to be assigned to Channels too. This enables things like excluding particular variants from specific channels. Thanks to Hendrik Depauw for putting in the [considerable work](https://github.com/vendure-ecommerce/vendure/pull/564) to implement this!
* **Order surcharges:** Order can now have surcharges added to them. A surcharge is intended to represent a non-SKU modification to the Order total. For example, some shops apply a percentage surcharge when using certain payment methods.
Currently, there is no GraphQL API for adding surcharges, so they must be added via custom code in plugins (e.g. as part of a CustomOrderProcess). Read more about the development of this feature in [GitHub issue #583](https://github.com/vendure-ecommerce/vendure/issues/583).
* **Override Admin UI nav:** Items in the Admin UI main navigation bar [can now be overridden]({{< relref "adding-navigation-items" >}}#overriding-existing-items). This allows you to replace entire routes with your own custom implementations.
* **Set Admin UI baseHref:** When building a custom version of the Admin UI using the `@vendure/ui-devkit` package, you can now specify a [`baseHref` value]({{< relref "ui-extension-compiler-options" >}}#basehref) to allow the Admin UI to be accessible via a path other than `/admin/`.
* **Shipping promotions:** It is now possible to define PromotionActions which apply specifically to the Order's shipping. The typical example is a "free shipping" promotion (which we now include by default).
* **Fixed order discounts:** The default PromotionActions now include a "Discount order by fixed amount" action, enabling Promotions like "$5 off order".

---

**📖 See all changes in the [v0.18.0 Changelog](https://github.com/vendure-ecommerce/vendure/blob/fb6896eb304bc57073c4a7946540380f01c2afd9/CHANGELOG.md#0180-2020-12-31)**

## BREAKING CHANGES / Migration Guide

{{< alert warning >}}
**🚧 Read this section carefully!**

💬 If you have any questions or issues with the migration, please post them to the [v0.18.0 Migration thread](https://github.com/vendure-ecommerce/vendure/discussions/606) on our GitHub Discussions forum. 
{{< /alert >}}

👉 For general instructions on upgrading, please see the new [Updating Vendure guide]({{< relref "updating-vendure" >}}).

### Database migration

This release includes more database schema changes than usual. If your project is still in development and you don't need to preserve any data, it may be preferable to simply drop all tables and start again using the [`synchronize` option]({{< relref "migrations" >}}#synchronize-vs-migrate).

If you _do_ need to preserve real data (i.e. you have Vendure in production), follow this guide carefully. There is also a visual guide at the [end of the announcement video](https://vimeo.com/496041481#t=8m9s).

1. Create a complete backup of your database schema and data.
2. Generate a migration script as described in the [Migrations guide]({{< relref "migrations" >}}).
3. You must now modify the generated migration script in accordance with the examples given in this [Vendure 0.18.0 Migration gist](https://gist.github.com/michaelbromley/08cba2ad0101a2901516b6199fa84a00). This will involve moving around the generated queries, adding some queries, and adding calls to the provided utility functions which will deal with the more complex data migrations.
    * [Example postgres migration script](https://gist.github.com/michaelbromley/08cba2ad0101a2901516b6199fa84a00#file-example-postgres-migration-ts)
    * [Example mysql migration script](https://gist.github.com/michaelbromley/08cba2ad0101a2901516b6199fa84a00#file-example-mysql-migration-ts)
    * [Migration utility functions](https://gist.github.com/michaelbromley/08cba2ad0101a2901516b6199fa84a00#file-migration-utils-ts) (copy the contents of this file into the parent directory of the `/migrations` folder)
4. **IMPORTANT** test the migration first on data you are prepared to lose to ensure that it works as expected. Do not run on production data without testing. For production data, **make a full backup first!**

### GraphQL API changes

The following changes to the GraphQL Shop API may necessitate changes to your storefront application:

* The mutations  `setOrderShippingAddress`, `setOrderBillingAddress` `setOrderCustomFields` now return a union type which includes a new `NoActiveOrderError`. Code which refers to these mutations will need to be updated to account for the union with the fragment spread syntax `...on Order {...}`.
* Taxes on OrderItems are no longer listed under `adjustments`, they now have their own field `taxLines`.
* The `Order.shippingMethod` field has been replaced by `Order.shippingLines.shippingMethod`
* The `Order.subTotal` and `Order.total` fields previously referred to the tax-inclusive price, but now refer to the price without tax. The tax-inclusive prices are now `subTotalWithTax` and `totalWithTax` respectively.
* The `ShippingMethod.description` field now holds the optional long-form description, and the data it previously pointed to is now in the `name` field.
* The `Order.adjustments` and `Order.lines.adjustments` fields have been replaced by the `discounts` field.
* The `OrderLine.totalPrice` field has been deprecated and replaced by the `linePriceWithTax` field.

### TypeScript API Changes

* The `PaymentMethodHandler.createPayment()` method now takes a new `amount`
argument. Update any custom PaymentMethodHandlers to use account for this new parameter and use
it instead of `order.total` when creating a new payment.
    ```ts
    // before
    createPayment: async (ctx, order, args, metadata) {
      const transactionAmount = order.total;
      // ...
    }

    // after
    createPayment: async (ctx, order, amount, args, metadata) {
      const transactionAmount = amount;
      // ...
    }
    ```
* The `PriceCalculationStrategy` has been renamed to `OrderItemPriceCalculationStrategy`.
