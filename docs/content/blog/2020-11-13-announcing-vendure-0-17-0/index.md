---
title: "Announcing Vendure v0.17.0"
date: 2020-11-13T13:00:00
draft: false
author: "Michael Bromley"
images: 
    - "/blog/2020/11/announcing-vendure-v0.17.0/vendure-0.17.0-banner-01.jpg"
---

We are excited to announce the release of Vendure v0.17.0! This release introduces improved stock control capabilities, custom permissions, and more!

<!--more-->

{{< vimeo id="478954062" >}}

## Improved stock control

This release put you in control of how you manage your stock. First let's define a few new concepts:

* **Stock on hand:** This refers to the number of physical units of a particular variant which you have in stock right now. This can be zero or more, but not negative.
* **Allocated:** This refers to the number of units which have been assigned to Orders, but which have not yet been fulfilled.
* **Out-of-stock threshold:** This value determines the stock level at which the variant is considered "out of stock". This value is set globally, but can be overridden for specific variants. It defaults to `0`.
* **Saleable:** This means the number of units that can be sold right now. The formula is:
    `saleable = stockOnHand - allocated - outOfStockThreshold`

Here's a table to better illustrate the relationship between these concepts:

Stock on hand | Allocated | Out-of-stock threshold | Saleable
--------------|-----------|------------------------|----------
10            | 0         | 0                      | 10
10            | 0         | 3                      | 7
10            | 5         | 0                      | 5
10            | 5         | 3                      | 2
10            | 10        | 0                      | 0
10            | 10        | -5                     | 5

The saleable value is what determines whether the customer is able to add a variant to an order. If there is 0 saleable stock, then any attempt to add to the order will result in a new `InsufficientStockError`.

{{< figure src="./stock-error.gif" caption="Handling the InsufficientStockError in the storefront" >}}

### Back orders

You may have noticed that the `outOfStockThreshold` value can be set to a negative number. This allows you to sell variants even when you don't physically have them in stock. This is known as "back orders". 

Back orders can be really useful to allow orders to keep flowing even when stockOnHand temporarily drops to zero. For many businesses with predictable re-supply schedules they make a lot of sense.

Once a customer completes checkout, those variants in the order are marked as `allocated`. When a Fulfillment is created, those allocations are converted to Sales and the `stockOnHand` of each variant is adjusted. Fulfillments may only be created if there is sufficient stock on hand.

### Configurable stock allocation

By default, stock is allocated when checkout completes, which means when the Order transitions to the `'PaymentAuthorized'` or `'PaymentSettled'` state. However, you may have special requirements which mean you wish to allocate stock earlier or later in the order process. With the new [StockAllocationStrategy]({{< relref "stock-allocation-strategy" >}}) you can tailor allocation to your exact needs.

**📖 Further details in the new [Stock Control Guide]({{< relref "stock-control" >}})**

## Custom permissions

Vendure has a powerful role-based access control system built in. However, up until now it has been impossible to define your own custom permissions. This release adds support for easily defining custom permissions and using them to secure your own GraphQL & REST operations & end-points. These permissions can then be used in creating Roles, giving you total control over exactly which users may make use of the capabilities defined by your plugins.

For an example of how simple it is to define new permissions, see the new [Defining Custom Permissions guide]({{< relref "defining-custom-permissions" >}}).

What's more, your custom permissions will automatically appear in the Admin UI Roles editor, allowing seamless management of your plugin permissions!

{{< figure src="./custom-permissions.jpg" caption="The Admin UI displaying custom permissions" >}}

## Tax improvements

Tax can be a confusing subject. Our own tax handling was also a little confusing too - but this release aims to improve the situation!

### Consistent OrderItem.unitPrice

The GraphQL `OrderItem.unitPrice` field used to be given with _or_ without taxes included, depending on the Channel settings. Now it is _always_ the pre-tax price, and the `unitPriceWithTax` field is _always_ the tax-included price. 

**Note:** To convert existing Orders, you'll need to add a custom query to your database migration - see the migration guide below.

### More detailed tax information

There are also a couple of new fields which give more information about taxes applied to an Order:

* The `OrderLine` GraphQL type now includes new tax-related fields:
    * `linePrice` The total price of the line excluding tax
    * `linePriceWithTax` The total price of the line including tax
    * `lineTax` The total tax on this line
    * `taxRate` The percentage rate of tax applied to this line
* The `Order` GraphQL type now includes a [`taxSummary` field]({{< relref "/docs/graphql-api/shop/object-types" >}}#ordertaxsummary) which provides a breakdown of how much tax is being charged at each applicable tax rate, which looks like this:
    ```JSON
    {
      "data": {
        "activeOrder": {
          "code": "WFJQG3YL1XVPHZCG",
          "totalBeforeTax": 8188,
          "total": 9756,
          "taxSummary": [
            {
              "taxRate": 20,
              "taxBase": 7489,
              "taxTotal": 1498
            },
            {
              "taxRate": 10,
              "taxBase": 699,
              "taxTotal": 70
            }
          ]
        }
      } 
    }
    ```


## List filter improvements

List queries in the GraphQL APIs can now be filtered with the new `in` and `regex` filter.

The `in` filter allows filtering by matching the field against one of the provided strings:

```GraphQL
query {
  orders(options: {
    filter: {
      state: {
        in: ["PaymentAuthorized", "PaymentSettled"]
      }
    }
  }) {
    items {
      id
      # ...etc
    }
  }
}
```

The `regex` filter allows matching the field against a regular expression:
 
```GraphQL
query {
  customers(options: {
    filter: {
      emailAddress: {
        regex: "g(oogle)?mail\\.com$"
      }
    }
  }) {
    items {
      id
      # ...etc
    }
  }
}
```
 
_Note: the specific supported regex syntax varies between the different database drivers._
 
These new filters enabled a much improved list view for orders in the Admin UI:

{{< figure src="./order-filters.gif" caption="Improved filtering in the Order list view" >}}

## Other notable improvements

* The `ShippingMethod` entity is now translatable, and now features both `name` and `description` fields, allowing richer information to be provided to customers.
* All methods of configurable strategies (e.g. PaymentMethodHandler, ShippingCalculator etc.) now receive the `RequestContext` object.
* The `Fulfillment` entity now supports custom fields. 
* Performance of ShippingEligibilityCheckers has been drastically improved. Previously _every_ checker was executed on _every_ change to an Order. This could have resulted in hundreds of needless calls. For async checkers, especially those which call out to third-party APIs, this lead to poor performance. Now only the checker of the _selected_ ShippingMethod is ever called, and even those calls can be minimized using the new [`ShouldRunCheck` function]({{< relref "should-run-check-fn" >}}).

**📖 See all changes in the [v0.17.0 Changelog](https://github.com/vendure-ecommerce/vendure/blob/f8eb11fdfd6df2958dd8e416c61d5dbfd4f48e28/CHANGELOG.md#0170-2020-11-13)**

## BREAKING CHANGES / Migration Guide

{{< alert warning >}}
**🚧 Read this section carefully**
{{< /alert >}}

👉 For general instructions on upgrading, please see the new [Updating Vendure guide]({{< relref "updating-vendure" >}}).

### Database migration

_Note: The SQL syntax given here is for MySQL. If you are using Postgres, adjust the syntax accordingly, i.e. double-quotes for tables/column names, single-quote for string literals etc._

1. Generate a migration script as described in the [Migrations guide]({{< relref "migrations" >}}).
2. If you are using **MySQL or MariaDB**, you'll need to disable foreign key checks for the migration, due to an [open TypeORM issue](https://github.com/typeorm/typeorm/issues/3766): 
   ```TypeScript {hl_lines=[2,6]}
   public async up(queryRunner: QueryRunner): Promise<any> {
     await queryRunner.query("SET FOREIGN_KEY_CHECKS=0;", undefined);
   
     // ... the generated migration statements
   
     await queryRunner.query("SET FOREIGN_KEY_CHECKS=1;", undefined);
   }
   ```
3. Add the following line immediately before the first generated migration query:
   ```TypeScript
   // Migrate existing OrderItems to use the new tax convention
   await queryRunner.query("UPDATE `order_item` SET `unitPrice` = ROUND(`unitPrice` / ((`taxRate` + 100) / 100)) WHERE `unitPriceIncludesTax` = 1", undefined);
   ```
4. Add the following line immediately after the `CREATE TABLE shipping_method_translation` statement:
   ```TypeScript
   // Migrate existing ShippingMethod names to the new translatable schema structure
   await queryRunner.query("INSERT INTO `shipping_method_translation` (`languageCode`, `name`, `description`, `baseId`) SELECT 'en', `description`, '', id FROM `shipping_method`", undefined);
   ```
3. **IMPORTANT** test the migration first on data you are prepared to lose to ensure that it works as expected. Do not run on production data without testing. For production data, **make a full backup first!**

### Adding RequestContext

If you have created your own custom strategies, you'll need to add the RequestContext argument as the first argument in any methods. TypeScript will tell you about this when you try to compile. For example:
```diff
export const myShippingCalculator = new ShippingCalculator({
    code: 'my-shipping-calculator',
    description: [/* */],
    args: {},
-    calculate: (order, args) => {
+    calculate: (ctx, order, args) => {
        return { price: args.rate, priceWithTax: args.rate * ((100 + args.taxRate) / 100) };
    },
});
```

### Update references to ShippingMethod.description

If your code references the `ShippingMethod.description` field, change it to `ShippingMethod.name`. The description is now used for an (optional) long-form detailed description.

### Deprecated fields

The following fields in the GraphQL APIs are deprecated (but still exist for now):

* `OrderItem.unitPriceIncludesTax`: unitPrice is now always without tax
* `OrderLine.totalPrice`: Use `linePriceWithTax` instead
