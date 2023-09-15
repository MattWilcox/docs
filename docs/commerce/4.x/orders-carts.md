---
sidebarDepth: 2
containsGeneratedContent: yes
---
# Orders & Carts

Variants are added to a _cart_ that can be completed to become an _order_. Carts and orders are both listed in the control panel under **Commerce** → **Orders**.

When we use the terms “cart” and “order”, we’re always referring to an [Order](commerce4:craft\commerce\elements\Order) element; a cart is simply an order that hasn’t been completed—meaning its `isCompleted` property is `false` and its `dateCompleted` is `null`.

Typically, a cart is completed in response to a customer [making a payment](./making-payments.md)—or by satisfying other requirements you’ve defined through [configuration](./config-settings.md) or an [extension](./extend/README.md).

## Carts

As a customer or store manager is building a cart, the goal is to maintain an up-to-date list of items with their relevant costs, discounts, and promotions. For this reason, the cart will be recalculated each time a change is made.

Once a cart is completed, however, it becomes an [order](#orders) that represents choices explicitly finalized by whoever completed the cart. The order’s behavior changes slightly at this point: the customer will no longer be able to make edits, and changes made by a store manager will not automatically trigger [recalculation](#recalculating-orders).

Carts and orders are both listed on the Orders index page in the control panel, where you can further limit your view to _active_ carts updated in the last hour, and _inactive_ carts older than an hour that are likely to be abandoned. (You can customize that time limit using the [`activeCartDuration`](config-settings.md#activecartduration) setting.)

Craft will automatically to purge (delete) abandoned carts after 90 days, and you can customize this behavior with the [`purgeInactiveCarts`](config-settings.md#purgeinactivecarts) and [`purgeInactiveCartsDuration`](config-settings.md#purgeinactivecartsduration) settings.

Let’s go over a few common actions you may want to perform on a cart:

- [Fetching a Cart](#fetching-a-cart)
- [Adding Items to a Cart](#adding-items-to-a-cart)
- [Working with Line Items](#working-with-line-items)
- [Loading a Cart](#loading-a-cart)
- [Using Custom Fields](#storing-data-in-custom-fields)
- [Forgetting](#forget-a-cart) and [loading](#load-a-cart) carts

More topics are covered in separate pages:

- [Addresses](addresses.md)
- [cart.availableShippingMethodOptions](shipping.md#cart-availableshippingmethodoptions)

### Fetching a Cart

In your templates, you can get the current user’s cart like this:

```twig
{% set cart = craft.commerce.carts.cart %}

{# Same thing: #}
{% set cart = craft.commerce.getCarts().getCart() %}
```

The current cart is also available via Ajax:

```js
fetch('/actions/commerce/cart/get-cart', {
  headers: {
    'Accept': 'application/json',
  },
})
  .then(r => r.json())
  .then(console.log);

// -> { cart: { ... } }
```

Either of the examples above will generate a new cart _cookie_ if none exists. However, Commerce will only create an order element when a change is made to the cart—this prevents runaway creation of empty carts for guests or bots who happen to request a page that displays some cart information like the current item quantity or total.

To see what cart information you can use in your templates, take a look at the [Order](commerce4:craft\commerce\elements\Order) class reference. You can also see sample Twig in the example templates’ [`shop/cart/index.twig`](https://github.com/craftcms/commerce/blob/main/example-templates/dist/shop/cart/index.twig).

<toggle-tip title="Example Order">

<<< @/docs/commerce/4.x/example-objects/order.php

</toggle-tip>

Once a cart’s completed and turned into an order, calling `craft.commerce.carts.cart` starts this process over.

### Adding Items to a Cart

You can add purchasables (like [a variant](products-variants.md#variants)) to the cart by submitting post requests to the `commerce/cart/update-cart` action. Items can be added one at a time or in an array.

#### Adding a Single Item

You can add a single item to a cart by submitting its `purchasableId`.

This gets a product and creates a form that will add its default variant to the cart:

```twig
{% set product = craft.products().one() %}
{% set variant = product.defaultVariant %}

<form method="post">
  {{ csrfInput() }}
  {{ actionInput('commerce/cart/update-cart') }}
  {{ hiddenInput('purchasableId', variant.id) }}
  <button>Add to Cart</button>
</form>
```

If the product has multiple variants, you could provide a dropdown menu to allow the customer to choose one of them:

```twig{10-14}
{% set product = craft.products().one() %}

<form method="post">
  {{ csrfInput() }}
  {{ actionInput('commerce/cart/update-cart') }}
  {{ redirectInput('shop/cart') }}
  {{ successMessageInput('Added ' ~ product.title ~ ' to cart.') }}
  {{ hiddenInput('qty', 1) }}

  <select name="purchasableId">
    {% for variant in product.variants %}
      <option value="{{ variant.id }}">{{ variant.sku }}</option>
    {% endfor %}
  </select>
  <button>Add to Cart</button>
</form>
```

We’re sneaking three new things in here as well:

1. The `successMessage` parameter can be used to customize the default “Cart updated.” flash message.
2. The `qty` parameter can be used to specify a quantity, which defaults to `1` if not supplied.
3. Craft’s [`redirectInput`](/4.x/dev/functions.md#redirectinput) tag can be used to take the user to a specific URL after the cart is updated successfully. **If any part of the cart update action fails, the user will not be redirected.**

#### Adding Multiple Items

You can add multiple purchasables to the cart in a single request using a `purchasables` array instead of a single `purchasableId`. This form adds all a product’s variants to the cart at once:

```twig{7-10}
{% set product = craft.products().one() %}
<form method="post">
  {{ csrfInput() }}
  {{ actionInput('commerce/cart/update-cart') }}
  {{ redirectInput('shop/cart') }}
  {{ successMessageInput('Products added to the cart.') }}
  {% for variant in product.variants %}
     {{ hiddenInput('purchasables[' ~ loop.index ~ '][id]', variant.id) }}
     {{ hiddenInput('purchasables[' ~ loop.index ~ '][qty]', 1) }}
  {% endfor %}
  <button>Add all variants to cart</button>
</form>
```

A unique index key is required to group the purchasable `id` with its `qty`, and in this example we’re using `{{ loop.index }}` as a convenient way to provide it.

::: tip
You can use the [`purchasableAvailable`](extend/events.md#purchasableavailable) event to control whether a line item should be available to the current user and cart.
:::

::: warning
Commerce Lite is limited to a single line in the cart.\
If a customer adds a new item, it replaces whatever was already in the cart.\
If multiple items are added in a request, only the last one gets added to the cart.
:::

### Working with Line Items

Whenever a purchasable is added to the cart, the purchasable becomes a [line item](#purchasables-and-line-items) in that cart. In the examples above we set the line item’s quantity with the `qty` parameter, but we also have `note` and `options` to work with.

#### Line Item Options and Notes

An `options` parameter can be submitted with arbitrary data to include with that item. A `note` parameter can include a message from the customer.

In this example, we’re providing the customer with an option to include a note, preset engraving message, and giftwrap option:

::: code
```twig Single Item
{% set product = craft.products().one() %}
{% set variant = product.defaultVariant %}
<form method="post">
  {{ csrfInput() }}
  {{ actionInput('commerce/cart/update-cart') }}
  {{ hiddenInput('qty', 1) }}
  <input type="text" name="note" value="">
  <select name="options[engraving]">
    <option value="happy-birthday">Happy Birthday</option>
    <option value="good-riddance">Good Riddance</option>
  </select>
  <select name="options[giftwrap]">
    <option value="yes">Yes Please</option>
    <option value="no">No Thanks</option>
  </select>
  {{ hiddenInput('purchasableId', variant.id) }}
  <button>Add to Cart</button>
</form>
```
```twig Multiple Items
{% set product = craft.products().one() %}
<form method="post">
  {{ csrfInput() }}
  {{ actionInput('commerce/cart/update-cart') }}
  {% for variant in product.variants %}
    {{ hiddenInput('purchasables[' ~ loop.index ~ '][id]', variant.id) }}
    {{ hiddenInput('purchasables[' ~ loop.index ~ '][qty]', 1) }}
    <input type="text" name="purchasables[{{ loop.index }}][note]" value="">
    <select name="purchasables[{{ loop.index }}][options][engraving]">
      <option value="happy-birthday">Happy Birthday</option>
      <option value="good-riddance">Good Riddance</option>
    </select>
    <select name="purchasables[{{ loop.index }}][options][giftwrap]">
      <option value="yes">Yes Please</option>
      <option value="no">No Thanks</option>
    </select>
  {% endfor %}
  <button>Add to Cart</button>
</form>
```
:::

::: warning
Commerce does not validate the `options` and `note` parameters. If you’d like to limit user input, use front-end validation or use the [`Model::EVENT_DEFINE_RULES`](craft4:craft\base\Model::EVENT_DEFINE_RULES) event to add validation rules for the [`LineItem`](commerce4:craft\commerce\models\LineItem) model.
:::

The note and options will be visible on the order’s detail page in the control panel:

![Line Item Option Review](./images/lineitem-options-review.png)

#### Updating Line Items

The `commerce/cart/update-cart` endpoint is for both adding *and updating* cart items.

You can directly modify any line item’s `qty`, `note`, and `options` using that line item’s ID in a `lineItems` array. Here we’re exposing each line item’s quantity and note for the customer to edit:

```twig{6,7}
{% set cart = craft.commerce.carts.cart %}
<form method="post">
  {{ csrfInput() }}
  {{ actionInput('commerce/cart/update-cart') }}
  {% for item in cart.lineItems %}
    <input type="number" name="lineItems[{{ item.id }}][qty]" min="1" value="{{ item.qty }}">
    <input type="text" name="lineItems[{{ item.id }}][note]" placeholder="My Note" value="{{ item.note }}">
  {% endfor %}
  <button>Update Line Item</button>
</form>
```

You can remove a line item by including a `remove` parameter in the request. This example adds a checkbox the customer can use to remove the line item from the cart:

```twig{8}
{% set cart = craft.commerce.carts.cart %}
<form method="post">
  {{ csrfInput() }}
  {{ actionInput('commerce/cart/update-cart') }}
  {% for item in cart.lineItems %}
    <input type="number" name="lineItems[{{ item.id }}][qty]" min="1" value="{{ item.qty }}">
    <input type="text" name="lineItems[{{ item.id }}][note]" placeholder="My Note" value="{{ item.note }}">
    <input type="checkbox" name="lineItems[{{ item.id }}][remove]" value="1"> Remove item<br>
  {% endfor %}
  <button>Update Line Item</button>
</form>
```

::: tip
The [example templates](example-templates.md) include a [detailed cart template](https://github.com/craftcms/commerce/blob/main/example-templates/dist/shop/cart/index.twig) for adding and updating items in a full checkout flow.
:::

#### Options Uniqueness

If a purchasable is posted that’s identical to one already in the cart, the relevant line item’s quantity will be incremented by the submitted `qty`.

If you posted a purchasable or tried to update a line item with different `options` values, however, a new line item will be created instead. This behavior is consistent whether you’re updating one item at a time or multiple items in a single request.

Each line item’s uniqueness is determined behind the scenes by an `optionsSignature` attribute, which is a hash of the line item’s options. You can think of each line item as being unique based on a combination of its `purchasableId` and `optionsSignature`.

::: tip
The `note` parameter is not part of a line item’s uniqueness; it will always be updated on a matching line item.
:::

#### Line Item Totals

Each line item includes several totals:

- **lineItem.subtotal** is the product of the line item’s `qty` and `salePrice`.
- **lineItem.adjustmentsTotal** is the sum of each of the line item’s adjustment `amount` values.
- **lineItem.total** is the sum of the line item’s `subtotal` and `adjustmentsTotal`.

### Loading and Forgetting Carts

“Loading” and “forgetting” are a pair of actions that affect what cart is associated with the customer’s session.

#### Load a Cart

Commerce provides a [`commerce/cart/load-cart`](dev/controller-actions.md#get-post-cart-load-cart) endpoint for loading an existing cart into a cookie for the current customer.

You can have the user interact with the endpoint by either [navigating to a URL](#loading-a-cart-with-a-url) or by [submitting a form](#loading-a-cart-with-a-form). Either way, the cart number is required.

Each method will store any errors in the session’s error flash data (`craft.app.session.getFlash('error')`), and the cart being loaded can be active or inactive.

::: tip
If the desired cart belongs to a user, that user must be logged in to load it into a browser cookie.
:::

The [`loadCartRedirectUrl`](config-settings.md#loadcartredirecturl) setting determines where the customer will be sent by default after the cart has been loaded.

#### Loading a Cart with a URL

Have the customer navigate to the `commerce/cart/load-cart` endpoint, including the `number` parameter in the URL.

A quick way for a store manager to grab the URL is by navigating in the control panel to **Commerce** → **Orders**, selecting one item from **Active Carts** or **Inactive Carts**, and choosing **Share cart…** from the context menu:

![Share cart context menu option](./images/share-cart.png)

You can also do this from an order edit page by choosing the gear icon and then **Share cart…**.

To do this programmatically, you’ll need to create an absolute URL for the endpoint and include a reference to the desired cart number.

This example sets `loadCartUrl` to an absolute URL the customer can access to load their cart. Again we’re assuming a `cart` object already exists for the cart that should be loaded:

::: code
```twig
{% set loadCartUrl = actionUrl(
  'commerce/cart/load-cart',
  { number: cart.number }
) %}
```

```php
$loadCartUrl = craft\helpers\UrlHelper::actionUrl(
    'commerce/cart/load-cart',
    ['number' => $cart->number]
);
```
:::

::: tip
This URL can be presented to the user however you’d like. It’s particularly useful in an email that allows the customer to retrieve an abandoned cart.
:::

#### Loading a Cart with a Form

Send a GET or POST action request with a `number` parameter referencing the cart you’d like to load. When posting the form data, you can include a specific redirect location like you can with any other Craft post data.

This is a simplified version of [`shop/cart/load.twig`](https://github.com/craftcms/commerce/tree/main/example-templates/dist/shop/cart/load.twig) in the example templates, where a `cart` variable has already been set to the cart that should be loaded:

```twig
<form method="post">
  {{ csrfInput() }}
  {{ actionInput('commerce/cart/load-cart') }}
  {{ redirectInput('/shop/cart') }}

  <input type="text" name="number" value="{{ cart.number }}">
  <button>Submit</button>
</form>
```

#### Restoring Previous Cart Contents

If the customer is a registered user, they may want to continue shopping from another browser or computer. If they have an empty cart on the second device (as they would by default) and they log in, their most recently-created cart will automatically be loaded.

::: tip
When a guest with an active cart creates an account, that cart will be remain active after logging in.
:::

You can allow a logged-in customer to see their previously used carts:

::: code
```twig
{% if currentUser %}
  {% set currentCart = craft.commerce.carts.cart %}
  {% if currentCart.id %}
    {# Return all incomplete carts *except* the currently-loaded one #}
    {% set oldCarts = craft.orders()
      .isCompleted(false)
      .id('not '~currentCart.id)
      .user(currentUser)
      .all() %}
  {% else %}
    {% set oldCarts = craft.orders()
      .isCompleted(false)
      .user(currentUser)
      .all() %}
  {% endif %}
{% endif %}
```
```php
use craft\commerce\Plugin as Commerce;
use craft\commerce\elements\Order;

$currentUser = Craft::$app->getUser()->getIdentity();

if ($currentUser) {
    $currentCart = Commerce::getInstance()->getCarts()->getCart();
    if ($currentCart->id) {
        // Return all incomplete carts *except* the currently-loaded one
        $oldCarts = Order::findAll()
            ->isCompleted(false)
            ->id('not '.$currentCart->id)
            ->user($currentUser);
    } else {
        $oldCarts = Order::findAll()
            ->isCompleted(false)
            ->user($currentUser);
    }
}
```
:::

You could then loop over the line items in those older carts and allow the customer to add them to their current cart:

```twig
<h2>Previous Cart Items</h2>

<form method="post">
  {{ csrfInput() }}
  {{ actionInput('commerce/cart/update-cart') }}

  {% for oldCart in oldCarts %}
    {% for lineItem in oldCart.lineItems %}
      {{ lineItem.description }}
      <label>
        {{ input('checkbox', 'purchasables[][id]', lineItem.purchasableId) }}
        Add to cart
      </label>
    {% endfor %}
  {% endfor %}

  <button>Add Items</button>
</form>
```

### Forgetting a Cart <Since ver="4.3.0" product="Commerce" repo="craftcms/commerce" feature="The ability to forget a cart" />

A logged-in customer’s cart is stored in a cookie that persists across sessions, so they can close their browser and return to the store without losing their cart. If the customer logs out, their cart will automatically be forgotten.

Removing all the items from a cart doesn’t mean that the cart is forgotten, though—sometimes, fully detaching a cart from the session is preferable to emptying it. To remove a cart from the customer’s session (without logging out or clearing the items), make a `POST` request to the [`cart/forget-cart` action](dev/controller-actions.md#post-cart-forget-cart). A cart number is _not_ required—Commerce can only detach the customer’s current cart.

The next time the customer makes a request to the [`update-cart` action](dev/controller-actions.md#post-cart-update-cart), they will be given a new cart

::: tip
In previous versions, you can call the `forgetCart()` method manually to remove the current cart cookie. The cart itself will not be deleted—just disassociated with the customer’s session until it’s loaded again by some other means.

::: code
```twig
{# Forget the cart in the current session. #}
{{ craft.commerce.carts.forgetCart() }}
```
```php
use craft\commerce\Plugin as Commerce;

// Forget the cart in the current session.
Commerce::getInstance()->getCarts()->forgetCart();
```
:::
:::

### Storing Data in Custom Fields

Like any other type of [element](/4.x/elements.md), orders can have custom fields associated with them via a field layout. To customize the fields available on carts and orders, visit **Commerce** &rarr; **System Settings** &rarr; **Order Fields**.

Custom fields are perfect for storing information about an order that falls outside [line item options or notes](orders-carts.md#line-item-options-and-notes).

::: tip
Now that [Addresses](/4.x/addresses.md#setup) are stored as elements, they support custom fields, too!
:::

You can update custom fields on a cart by posting data to the [`commerce/cart/update-cart`](dev/controller-actions.md#post-cart-update-cart) action under a `fields` key:

```twig
<form method="post">
  {{ csrfInput() }}
  {{ actionInput('commerce/cart/update-cart') }}
  {{ redirectInput('shop/cart') }}

  <label>
    Gift Message
    {{ input('text', 'fields[giftMessage]', cart.giftMessage) }}
  </label>

  <button>Update Cart</button>
</form>
```

## Orders

Orders are [Element Types](/4.x/extend/element-types.md) and can have custom fields associated with them. You can browse orders in the control panel by navigating to **Commerce** → **Orders**.

When a cart becomes an order, the following things happen:

1. The `dateOrdered` order attribute is set to the current date.
2. The `isCompleted` order attribute is set to `true`.
3. The default [order status](custom-order-statuses.md) is set on the order and any emails for this status are sent.
4. The order reference number is generated for the order, based on the “Order Reference Number Format” setting. (In the control panel: **Commerce** → **Settings** → **System Settings** → **General Settings**.)

Instead of being recalculated on each change like a cart, the order will only be recalculated if you [manually trigger recalculation](#recalculating-orders).

Adjustments for discounts, shipping, and tax may be applied when an order is recalcuated. Each adjustment is related to its order, and can optionally relate to a specific line item.

If you’d like to jump straight to displaying order information in your templates, take a look at the the <commerce4:craft\commerce\elements\Order> class reference for a complete list of available properties.

### Order Numbers

There are three ways to identify an order: by order number, short order number, and order reference number.

#### Order Number

The order number is a hash generated when the cart cookie is first created. It exists even before the cart is saved in the database, and remains the same for the entire life of the order.

This is different from the order reference number, which is only generated after the cart has been completed and becomes an order.

We recommend using the order number when referencing the order in URLs or anytime the order is retrieved publicly.

#### Short Order Number

The short order number is the first seven characters of the order number. This is short enough to still be unique, and is a little friendlier to customers, although not as friendly as the order reference number.

#### Order Reference Number

The order reference number is generated on cart completion according to the **Order Reference Number Format** in **Commerce** → **Settings** → **System Settings** → **General Settings**.

This number is usually best used as the customer-facing identifier of the order, but shouldn’t be used in URLs.

```twig
{{ object.reference }}
```

The “Order Reference Number Format” is a mini Twig template that’s rendered when the order is completed. It can use order attributes along with Twig filters and functions. For example:

```twig
{{ object.dateCompleted|date('Y') }}-{{ id }}
```

Output:

```
2018-43
```

In this example, `{{ id }}` refers to the order’s element ID, which is not sequential. If you would rather generate a unique sequential number, a simple way would be to use Craft’s [seq()](https://craftcms.com/docs/4.x/dev/functions.html#seq) Twig function, which generates a next unique number based on the `name` parameter passed to it.

The `seq()` function takes the following parameters:

1. A key name. If this name is changed, a new sequence starting at one is started.
2. An optional padding character length. For example if the next sequence number is `14` and the padding length is `8`, the generation number will be `00000014`

For example:

```twig
{{ object.dateCompleted|date('Y') }}-{{ seq(object.dateCompleted|date('Y'), 8) }}
```

Ouput:

```
2018-00000023
```

In this example we’ve used the year as the sequence name so we automatically get a new sequence, starting at `1`, when the next year arrives.

### Creating Orders

An order is usually created on the front end as a customer [adds items](#adding-items-to-a-cart) to and completes a [cart](#carts). With Commerce Pro, An order may also be created in the control panel.

To create a new order, navigate to **Commerce** → **Orders**, and choose **New Order**. This will create a new order that behaves like a cart. As [purchasables](purchasables.md) are added and removed from the order, it will automatically recalculate its [sales](sales.md) and adjustments.

::: warning
You must be using [Commerce Pro](editions.md) and have “Edit Orders” permission to create orders from the control panel.
:::

To complete the order, press **Mark as completed**.

### Editing Orders

Orders can be edited in the control panel by visiting the order edit page and choosing **Edit**. You’ll enter edit mode where you can change values within the order.

While editing the order, it will refresh subtotals and totals and display any errors. It will _not_ automatically recalculate the order based on system rules like shipping, taxes, or promotions. Choose **Recalculate order** to have it fully recalculate including those system rules.

Once you’re happy with your changes, choose **Update Order** to save it to the database.

### Order Totals

Every order includes a few important totals:

- **order.itemSubtotal** is the sum of the order’s [line item `subtotal` amounts](#line-item-totals).
- **order.itemTotal** is the sum of the order’s [line item `total` amounts](#line-item-totals).
- **order.adjustmentSubtotal** is the sum of the order’s adjustments.
- **order.total** is the sum of the order’s `itemSubtotal` and `adjustmentsTotal`.
- **order.totalPrice** is the total order price with a minimum enforced by the [minimumTotalPriceStrategy](config-settings.html#minimumtotalpricestrategy) setting.

::: warning
You’ll also find an `order.adjustmentsSubtotal` which is identical to `order.adjustmentsTotal`. It will be removed in Commerce 4.
:::

### Recalculating Orders

Let’s take a closer look at how carts and orders can be recalculated.

A cart or order must always be in one of three calculation modes:

- **Recalculate All** is active for a cart, or an order which is not yet completed.\
This mode refreshes each line item’s details from the related purchasable and re-applies all adjustments to the cart. In other words, it rebuilds and recalculates cart details based on information from purchasables and how Commerce is configured. This mode merges duplicate line items and removes any that are out of stock or whose purchasables were deleted.
- **Adjustments Only** doesn’t touch line items, but re-applies adjustments on the cart or order. This can only be set programmatically and is not available from the control panel.
- **None** doesn’t change anything at all, and is active for an order (completed cart) so only manual edits or admin-initiated recalculation can modify order details.

Cart/order subtotals and totals are computed values that always reflect the sum of line items. A few examples are `totalPrice`, `itemTotal`, and `itemSubtotal`.

You can manually recalculate an order by choosing “Recalculate order” at the bottom of the order edit screen:

![](./images/recalculate-order.png)

This will set temporarily the order’s calculation mode to *Recalculate All* and trigger recalculation. You can then apply the resulting changes to the order by choosing “Update Order”, or discard them by choosing “Cancel”.

### Order Notices

Notices are added to an order whenever it changes, whether it’s the customer saving the cart or a store manager recalculating from the control panel. Each notice is an [OrderNotice](https://github.com/craftcms/commerce/blob/main/src/models/OrderNotice.php) model describing what changed and could include the following:

- A previously-valid coupon or shipping method was removed from the order.
- A line item’s purchasable was no longer available so that line item was removed from the cart.
- A line item’s sale price changed for some reason, like the sale no longer applied for example.

The notice includes a human-friendly terms that can be exposed to the customer, and references to the order and attribute it’s describing.

#### Accessing Order Notices

Notices are stored on the order and accessed with `getNotices()` and `getFirstNotice()` methods.

Each can take optional `type` and `attribute` parameters to limit results to a certain order attribute or kind of notice.

A notice’s `type` will be one of the following:

- `lineItemSalePriceChanged`
- `lineItemRemoved`
- `shippingMethodChanged`
- `invalidCouponRemoved`

::: code
```twig
{# @var order craft\commerce\elements\Order #}

{# Get a multi-dimensional array of notices by attribute key #}
{% set notices = order.getNotices() %}

{# Get an array of notice models for the `couponCode` attribute only #}
{% set couponCodeNotices = order.getNotices(null, 'couponCode') %}

{# Get the first notice only for the `couponCode` attribute #}
{% set firstCouponCodeNotice = order.getFirstNotice(null, 'couponCode') %}

{# Get an array of notice models for changed line item prices #}
{% set priceChangeNotices = order.getNotices('lineItemSalePriceChanged') %}
```
```php
// @var craft\commerce\elements\Order $order

// Get a multi-dimensional array of notices by attribute key
$notices = $order->getNotices();

// Get an array of notice models for the `couponCode` attribute only
$couponCodeNotices = $order->getNotices(null, 'couponCode');

// Get the first notice only for the `couponCode` attribute
$firstCouponCodeNotice = $order->getFirstNotice(null, 'couponCode');

// Get an array of notice models for changed line item prices
$priceChangeNotices = $order->getNotices('lineItemSalePriceChanged');
```
:::

Each notice can be output as a string in a template:

```twig
{% set notice = order.getFirstNotice('salePrice') %}
<p>{{ notice }}</p>
{# result: <p>The price of x has changed</p> #}
```

#### Clearing Order Notices

Notices remain on an order until they’re cleared. You can clear all notices by posting to the cart update form action, or call the order’s `clearNotices()` method to clear specific notices or all of them at once.

This example clears all the notices on the cart:

::: code
```twig{5}
<form method="post">
  {{ csrfInput() }}
  {{ actionInput('commerce/cart/update-cart') }}
  {{ successMessageInput('All notices dismissed') }}
  {{ hiddenInput('clearNotices') }}
  <button>Dismiss</button>
</form>
```
```php{7}
use craft\commerce\Plugin as Commerce;

// Get the current cart
$cart = Commerce::getInstance()->getCarts()->getCart();

// Clear all notices
$cart->clearNotices();
```
:::

This only clears `couponCode` notices on the cart:

::: code
```twig
{# Clear notices on the `couponCode` attribute #}
{% do cart.clearNotices(null, 'couponCode') %}
```
```php{7}
use craft\commerce\Plugin as Commerce;

// Get the current cart
$cart = Commerce::getInstance()->getCarts()->getCart();

// Clear notices on the `couponCode` attribute
$cart->clearNotices(null, 'couponCode');
```
:::

The notices will be cleared from the cart object in memory. If you’d like the cleared notices to persist, you’ll need to save the cart:

::: code
```twig
{# Save the cart #}
{% do craft.app.elements.saveElement(cart) %}
```
```php
// Save the cart
Craft::$app->getElements()->saveElement($cart);
```
:::

### Order Status Emails

If [status emails](emails.md) are set up for a newly-updated order status, messages will be sent when the updated order is saved.

You can manually re-send an order status email at any time. Navigate to an order’s edit page, then select the desired email from the Send Email menu at the top of the page.

## Querying Orders

You can fetch orders in your templates or PHP code using **order queries**.

::: code
```twig
{# Create a new order query #}
{% set myOrderQuery = craft.orders() %}
```
```php
// Create a new order query
$myOrderQuery = \craft\commerce\elements\Order::find();
```
:::

Once you’ve created an order query, you can set [parameters](#parameters) on it to narrow down the results, and then [execute it](https://craftcms.com/docs/4.x/element-queries.html#executing-element-queries) by calling `.all()`. An array of [Order](commerce4:craft\commerce\elements\Order) objects will be returned.

::: tip
See [Element Queries](https://craftcms.com/docs/4.x/element-queries.html) in the Craft docs to learn about how element queries work.
:::

### Example

We can display an order with a given order number by doing the following:

1. Create an order query with `craft.orders()`.
2. Set the [number](#number) parameter on it.
3. Fetch the order with `.one()`.
4. Output information about the order as HTML.

```twig
{# Get the requested order number from the query string #}
{% set orderNumber = craft.app.request.getQueryParam('number') %}

{# Create an order query with the 'number' parameter #}
{% set myOrderQuery = craft.orders()
  .number(orderNumber) %}

{# Fetch the order #}
{% set order = myOrderQuery.one() %}

{# Make sure it exists #}
{% if not order %}
  {% exit 404 %}
{% endif %}

{# Display the order #}
<h1>Order {{ order.getShortNumber() }}</h1>
<!-- ... ->
```

## Order Query Parameters

Order queries support the following parameters:

<!-- BEGIN PARAMS -->



<!-- textlint-disable -->

| Param                                         | Description
| --------------------------------------------- | --------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------
| [afterPopulate](#afterpopulate)               | Performs any post-population processing on elements.
| [andRelatedTo](#andrelatedto)                 | Narrows the query results to only orders that are related to certain other elements.
| [asArray](#asarray)                           | Causes the query to return matching orders as arrays of data, rather than [Order](commerce4:craft\commerce\elements\Order) objects.
| [cache](#cache)                               | Enables query cache for this Query.
| [clearCachedResult](#clearcachedresult)       | Clears the [cached result](https://craftcms.com/docs/4.x/element-queries.html#cache).
| [customer](#customer)                         | Narrows the query results based on the customer’s user account.
| [customerId](#customerid)                     | Narrows the query results based on the customer, per their user ID.
| [dateAuthorized](#dateauthorized)             | Narrows the query results based on the orders’ authorized dates.
| [dateCreated](#datecreated)                   | Narrows the query results based on the orders’ creation dates.
| [dateOrdered](#dateordered)                   | Narrows the query results based on the orders’ completion dates.
| [datePaid](#datepaid)                         | Narrows the query results based on the orders’ paid dates.
| [dateUpdated](#dateupdated)                   | Narrows the query results based on the orders’ last-updated dates.
| [email](#email)                               | Narrows the query results based on the customers’ email addresses.
| [expiryDate](#expirydate)                     | Narrows the query results based on the orders’ expiry dates.
| [fixedOrder](#fixedorder)                     | Causes the query results to be returned in the order specified by [id](#id).
| [gateway](#gateway)                           | Narrows the query results based on the gateway.
| [gatewayId](#gatewayid)                       | Narrows the query results based on the gateway, per its ID.
| [hasLineItems](#haslineitems)                 | Narrows the query results to only orders that have line items.
| [hasPurchasables](#haspurchasables)           | Narrows the query results to only orders that have certain purchasables.
| [hasTransactions](#hastransactions)           | Narrows the query results to only carts that have at least one transaction.
| [id](#id)                                     |
| [ignorePlaceholders](#ignoreplaceholders)     | Causes the query to return matching orders as they are stored in the database, ignoring matching placeholder elements that were set by [craft\services\Elements::setPlaceholderElement()](https://docs.craftcms.com/api/v4/craft-services-elements.html#method-setplaceholderelement).
| [inReverse](#inreverse)                       | Causes the query results to be returned in reverse order.
| [isCompleted](#iscompleted)                   | Narrows the query results to only orders that are completed.
| [isPaid](#ispaid)                             | Narrows the query results to only orders that are paid.
| [isUnpaid](#isunpaid)                         | Narrows the query results to only orders that are not paid.
| [itemSubtotal](#itemsubtotal)                 | Narrows the query results based on the order’s item subtotal.
| [itemTotal](#itemtotal)                       | Narrows the query results based on the order’s item total.
| [limit](#limit)                               | Determines the number of orders that should be returned.
| [number](#number)                             | Narrows the query results based on the order number.
| [offset](#offset)                             | Determines how many orders should be skipped in the results.
| [orderBy](#orderby)                           | Determines the order that the orders should be returned in. (If empty, defaults to `id ASC`.)
| [orderLanguage](#orderlanguage)               | Narrows the query results based on the order language, per the language string provided.
| [orderSiteId](#ordersiteid)                   | Narrows the query results based on the order language, per the language string provided.
| [orderStatus](#orderstatus)                   | Narrows the query results based on the order statuses.
| [orderStatusId](#orderstatusid)               | Narrows the query results based on the order statuses, per their IDs.
| [origin](#origin)                             | Narrows the query results based on the origin.
| [preferSites](#prefersites)                   | If [unique()](https://docs.craftcms.com/api/v4/craft-elements-db-elementquery.html#method-unique) is set, this determines which site should be selected when querying multi-site elements.
| [prepareSubquery](#preparesubquery)           | Prepares the element query and returns its subquery (which determines what elements will be returned).
| [reference](#reference)                       | Narrows the query results based on the order reference.
| [relatedTo](#relatedto)                       | Narrows the query results to only orders that are related to certain other elements.
| [search](#search)                             | Narrows the query results to only orders that match a search query.
| [shippingMethodHandle](#shippingmethodhandle) | Narrows the query results based on the shipping method handle.
| [shortNumber](#shortnumber)                   | Narrows the query results based on the order short number.
| [siteSettingsId](#sitesettingsid)             | Narrows the query results based on the orders’ IDs in the `elements_sites` table.
| [total](#total)                               | Narrows the query results based on the total.
| [totalDiscount](#totaldiscount)               | Narrows the query results based on the total discount.
| [totalPaid](#totalpaid)                       | Narrows the query results based on the total paid amount.
| [totalPrice](#totalprice)                     | Narrows the query results based on the total price.
| [totalQty](#totalqty)                         | Narrows the query results based on the total qty of items.
| [totalTax](#totaltax)                         | Narrows the query results based on the total tax.
| [trashed](#trashed)                           | Narrows the query results to only orders that have been soft-deleted.
| [uid](#uid)                                   | Narrows the query results based on the orders’ UIDs.
| [with](#with)                                 | Causes the query to return matching orders eager-loaded with related elements.
| [withAddresses](#withaddresses)               | Eager loads the the shipping and billing addressees on the resulting orders.
| [withAdjustments](#withadjustments)           | Eager loads the order adjustments on the resulting orders.
| [withAll](#withall)                           | Eager loads all relational data (addresses, adjustents, customers, line items, transactions) for the resulting orders.
| [withCustomer](#withcustomer)                 | Eager loads the user on the resulting orders.
| [withLineItems](#withlineitems)               | Eager loads the line items on the resulting orders.
| [withTransactions](#withtransactions)         | Eager loads the transactions on the resulting orders.


<!-- textlint-enable -->


#### `afterPopulate`

Performs any post-population processing on elements.










#### `andRelatedTo`

Narrows the query results to only orders that are related to certain other elements.



See [Relations](https://craftcms.com/docs/4.x/relations.html) for a full explanation of how to work with this parameter.



::: code
```twig
{# Fetch all orders that are related to myCategoryA and myCategoryB #}
{% set orders = craft.orders()
  .relatedTo(myCategoryA)
  .andRelatedTo(myCategoryB)
  .all() %}
```

```php
// Fetch all orders that are related to $myCategoryA and $myCategoryB
$orders = \craft\commerce\elements\Order::find()
    ->relatedTo($myCategoryA)
    ->andRelatedTo($myCategoryB)
    ->all();
```
:::


#### `asArray`

Causes the query to return matching orders as arrays of data, rather than [Order](commerce4:craft\commerce\elements\Order) objects.





::: code
```twig
{# Fetch orders as arrays #}
{% set orders = craft.orders()
  .asArray()
  .all() %}
```

```php
// Fetch orders as arrays
$orders = \craft\commerce\elements\Order::find()
    ->asArray()
    ->all();
```
:::


#### `cache`

Enables query cache for this Query.










#### `clearCachedResult`

Clears the [cached result](https://craftcms.com/docs/4.x/element-queries.html#cache).






#### `customer`

Narrows the query results based on the customer’s user account.

Possible values include:

| Value | Fetches orders…
| - | -
| `1` | with a customer with a user account ID of 1.
| a [User](https://docs.craftcms.com/api/v4/craft-elements-user.html) object | with a customer with a user account represented by the object.
| `'not 1'` | not the user account with an ID 1.
| `[1, 2]` | with an user account ID of 1 or 2.
| `['not', 1, 2]` | not with a user account ID of 1 or 2.



::: code
```twig
{# Fetch the current user's orders #}
{% set orders = craft.orders()
  .customer(currentUser)
  .all() %}
```

```php
// Fetch the current user's orders
$user = Craft::$app->user->getIdentity();
$orders = \craft\commerce\elements\Order::find()
    ->customer($user)
    ->all();
```
:::


#### `customerId`

Narrows the query results based on the customer, per their user ID.

Possible values include:

| Value | Fetches orders…
| - | -
| `1` | with a user with an ID of 1.
| `'not 1'` | not with a user with an ID of 1.
| `[1, 2]` | with a user with an ID of 1 or 2.
| `['not', 1, 2]` | not with a user with an ID of 1 or 2.



::: code
```twig
{# Fetch the current user's orders #}
{% set orders = craft.orders()
  .customerId(currentUser.id)
  .all() %}
```

```php
// Fetch the current user's orders
$user = Craft::$app->user->getIdentity();
$orders = \craft\commerce\elements\Order::find()
    ->customerId($user->id)
    ->all();
```
:::


#### `dateAuthorized`

Narrows the query results based on the orders’ authorized dates.

Possible values include:

| Value | Fetches orders…
| - | -
| `'>= 2018-04-01'` | that were authorized on or after 2018-04-01.
| `'< 2018-05-01'` | that were authorized before 2018-05-01
| `['and', '>= 2018-04-04', '< 2018-05-01']` | that were completed between 2018-04-01 and 2018-05-01.



::: code
```twig
{# Fetch orders that were authorized recently #}
{% set aWeekAgo = date('7 days ago')|atom %}

{% set orders = craft.orders()
  .dateAuthorized(">= #{aWeekAgo}")
  .all() %}
```

```php
// Fetch orders that were authorized recently
$aWeekAgo = new \DateTime('7 days ago')->format(\DateTime::ATOM);

$orders = \craft\commerce\elements\Order::find()
    ->dateAuthorized(">= {$aWeekAgo}")
    ->all();
```
:::


#### `dateCreated`

Narrows the query results based on the orders’ creation dates.



Possible values include:

| Value | Fetches orders…
| - | -
| `'>= 2018-04-01'` | that were created on or after 2018-04-01.
| `'< 2018-05-01'` | that were created before 2018-05-01.
| `['and', '>= 2018-04-04', '< 2018-05-01']` | that were created between 2018-04-01 and 2018-05-01.
| `now`/`today`/`tomorrow`/`yesterday` | that were created at midnight of the specified relative date.



::: code
```twig
{# Fetch orders created last month #}
{% set start = date('first day of last month')|atom %}
{% set end = date('first day of this month')|atom %}

{% set orders = craft.orders()
  .dateCreated(['and', ">= #{start}", "< #{end}"])
  .all() %}
```

```php
// Fetch orders created last month
$start = (new \DateTime('first day of last month'))->format(\DateTime::ATOM);
$end = (new \DateTime('first day of this month'))->format(\DateTime::ATOM);

$orders = \craft\commerce\elements\Order::find()
    ->dateCreated(['and', ">= {$start}", "< {$end}"])
    ->all();
```
:::


#### `dateOrdered`

Narrows the query results based on the orders’ completion dates.

Possible values include:

| Value | Fetches orders…
| - | -
| `'>= 2018-04-01'` | that were completed on or after 2018-04-01.
| `'< 2018-05-01'` | that were completed before 2018-05-01
| `['and', '>= 2018-04-04', '< 2018-05-01']` | that were completed between 2018-04-01 and 2018-05-01.



::: code
```twig
{# Fetch orders that were completed recently #}
{% set aWeekAgo = date('7 days ago')|atom %}

{% set orders = craft.orders()
  .dateOrdered(">= #{aWeekAgo}")
  .all() %}
```

```php
// Fetch orders that were completed recently
$aWeekAgo = new \DateTime('7 days ago')->format(\DateTime::ATOM);

$orders = \craft\commerce\elements\Order::find()
    ->dateOrdered(">= {$aWeekAgo}")
    ->all();
```
:::


#### `datePaid`

Narrows the query results based on the orders’ paid dates.

Possible values include:

| Value | Fetches orders…
| - | -
| `'>= 2018-04-01'` | that were paid on or after 2018-04-01.
| `'< 2018-05-01'` | that were paid before 2018-05-01
| `['and', '>= 2018-04-04', '< 2018-05-01']` | that were completed between 2018-04-01 and 2018-05-01.



::: code
```twig
{# Fetch orders that were paid for recently #}
{% set aWeekAgo = date('7 days ago')|atom %}

{% set orders = craft.orders()
  .datePaid(">= #{aWeekAgo}")
  .all() %}
```

```php
// Fetch orders that were paid for recently
$aWeekAgo = new \DateTime('7 days ago')->format(\DateTime::ATOM);

$orders = \craft\commerce\elements\Order::find()
    ->datePaid(">= {$aWeekAgo}")
    ->all();
```
:::


#### `dateUpdated`

Narrows the query results based on the orders’ last-updated dates.



Possible values include:

| Value | Fetches orders…
| - | -
| `'>= 2018-04-01'` | that were updated on or after 2018-04-01.
| `'< 2018-05-01'` | that were updated before 2018-05-01.
| `['and', '>= 2018-04-04', '< 2018-05-01']` | that were updated between 2018-04-01 and 2018-05-01.
| `now`/`today`/`tomorrow`/`yesterday` | that were updated at midnight of the specified relative date.



::: code
```twig
{# Fetch orders updated in the last week #}
{% set lastWeek = date('1 week ago')|atom %}

{% set orders = craft.orders()
  .dateUpdated(">= #{lastWeek}")
  .all() %}
```

```php
// Fetch orders updated in the last week
$lastWeek = (new \DateTime('1 week ago'))->format(\DateTime::ATOM);

$orders = \craft\commerce\elements\Order::find()
    ->dateUpdated(">= {$lastWeek}")
    ->all();
```
:::


#### `email`

Narrows the query results based on the customers’ email addresses.

Possible values include:

| Value | Fetches orders with customers…
| - | -
| `'foo@bar.baz'` | with an email of `foo@bar.baz`.
| `'not foo@bar.baz'` | not with an email of `foo@bar.baz`.
| `'*@bar.baz'` | with an email that ends with `@bar.baz`.



::: code
```twig
{# Fetch orders from customers with a .co.uk domain on their email address #}
{% set orders = craft.orders()
  .email('*.co.uk')
  .all() %}
```

```php
// Fetch orders from customers with a .co.uk domain on their email address
$orders = \craft\commerce\elements\Order::find()
    ->email('*.co.uk')
    ->all();
```
:::


#### `expiryDate`

Narrows the query results based on the orders’ expiry dates.

Possible values include:

| Value | Fetches orders…
| - | -
| `'>= 2020-04-01'` | that will expire on or after 2020-04-01.
| `'< 2020-05-01'` | that will expire before 2020-05-01
| `['and', '>= 2020-04-04', '< 2020-05-01']` | that will expire between 2020-04-01 and 2020-05-01.



::: code
```twig
{# Fetch orders expiring this month #}
{% set nextMonth = date('first day of next month')|atom %}

{% set orders = craft.orders()
  .expiryDate("< #{nextMonth}")
  .all() %}
```

```php
// Fetch orders expiring this month
$nextMonth = new \DateTime('first day of next month')->format(\DateTime::ATOM);

$orders = \craft\commerce\elements\Order::find()
    ->expiryDate("< {$nextMonth}")
    ->all();
```
:::


#### `fixedOrder`

Causes the query results to be returned in the order specified by [id](#id).



::: tip
If no IDs were passed to [id](#id), setting this to `true` will result in an empty result set.
:::



::: code
```twig
{# Fetch orders in a specific order #}
{% set orders = craft.orders()
  .id([1, 2, 3, 4, 5])
  .fixedOrder()
  .all() %}
```

```php
// Fetch orders in a specific order
$orders = \craft\commerce\elements\Order::find()
    ->id([1, 2, 3, 4, 5])
    ->fixedOrder()
    ->all();
```
:::


#### `gateway`

Narrows the query results based on the gateway.

Possible values include:

| Value | Fetches orders…
| - | -
| a [Gateway](commerce4:craft\commerce\base\Gateway) object | with a gateway represented by the object.




#### `gatewayId`

Narrows the query results based on the gateway, per its ID.

Possible values include:

| Value | Fetches orders…
| - | -
| `1` | with a gateway with an ID of 1.
| `'not 1'` | not with a gateway with an ID of 1.
| `[1, 2]` | with a gateway with an ID of 1 or 2.
| `['not', 1, 2]` | not with a gateway with an ID of 1 or 2.




#### `hasLineItems`

Narrows the query results to only orders that have line items.



::: code
```twig
{# Fetch orders that do or do not have line items #}
{% set orders = craft.orders()
  .hasLineItems()
  .all() %}
```

```php
// Fetch unpaid orders
$orders = \craft\commerce\elements\Order::find()
    ->hasLineItems()
    ->all();
```
:::


#### `hasPurchasables`

Narrows the query results to only orders that have certain purchasables.

Possible values include:

| Value | Fetches orders…
| - | -
| a [PurchasableInterface](commerce4:craft\commerce\base\PurchasableInterface) object | with a purchasable represented by the object.
| an array of [PurchasableInterface](commerce4:craft\commerce\base\PurchasableInterface) objects | with all the purchasables represented by the objects.




#### `hasTransactions`

Narrows the query results to only carts that have at least one transaction.



::: code
```twig
{# Fetch carts that have attempted payments #}
{% set orders = craft.orders()
  .hasTransactions()
  .all() %}
```

```php
// Fetch carts that have attempted payments
$orders = \craft\commerce\elements\Order::find()
    ->hasTransactions()
    ->all();
```
:::


#### `id`








#### `ignorePlaceholders`

Causes the query to return matching orders as they are stored in the database, ignoring matching placeholder
elements that were set by [craft\services\Elements::setPlaceholderElement()](https://docs.craftcms.com/api/v4/craft-services-elements.html#method-setplaceholderelement).










#### `inReverse`

Causes the query results to be returned in reverse order.





::: code
```twig
{# Fetch orders in reverse #}
{% set orders = craft.orders()
  .inReverse()
  .all() %}
```

```php
// Fetch orders in reverse
$orders = \craft\commerce\elements\Order::find()
    ->inReverse()
    ->all();
```
:::


#### `isCompleted`

Narrows the query results to only orders that are completed.



::: code
```twig
{# Fetch completed orders #}
{% set orders = craft.orders()
  .isCompleted()
  .all() %}
```

```php
// Fetch completed orders
$orders = \craft\commerce\elements\Order::find()
    ->isCompleted()
    ->all();
```
:::


#### `isPaid`

Narrows the query results to only orders that are paid.



::: code
```twig
{# Fetch paid orders #}
{% set orders = craft.orders()
  .isPaid()
  .all() %}
```

```php
// Fetch paid orders
$orders = \craft\commerce\elements\Order::find()
    ->isPaid()
    ->all();
```
:::


#### `isUnpaid`

Narrows the query results to only orders that are not paid.



::: code
```twig
{# Fetch unpaid orders #}
{% set orders = craft.orders()
  .isUnpaid()
  .all() %}
```

```php
// Fetch unpaid orders
$orders = \craft\commerce\elements\Order::find()
    ->isUnpaid()
    ->all();
```
:::


#### `itemSubtotal`

Narrows the query results based on the order’s item subtotal.

Possible values include:

| Value | Fetches orders…
| - | -
| `100` | with an item subtotal of 0.
| `'< 1000000'` | with an item subtotal of less than ,000,000.
| `['>= 10', '< 100']` | with an item subtotal of between  and 0.




#### `itemTotal`

Narrows the query results based on the order’s item total.

Possible values include:

| Value | Fetches orders…
| - | -
| `100` | with an item total of 0.
| `'< 1000000'` | with an item total of less than ,000,000.
| `['>= 10', '< 100']` | with an item total of between  and 0.




#### `limit`

Determines the number of orders that should be returned.



::: code
```twig
{# Fetch up to 10 orders  #}
{% set orders = craft.orders()
  .limit(10)
  .all() %}
```

```php
// Fetch up to 10 orders
$orders = \craft\commerce\elements\Order::find()
    ->limit(10)
    ->all();
```
:::


#### `number`

Narrows the query results based on the order number.

Possible values include:

| Value | Fetches orders…
| - | -
| `'xxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx'` | with a matching order number



::: code
```twig
{# Fetch the requested order #}
{% set orderNumber = craft.app.request.getQueryParam('number') %}
{% set order = craft.orders()
  .number(orderNumber)
  .one() %}
```

```php
// Fetch the requested order
$orderNumber = Craft::$app->request->getQueryParam('number');
$order = \craft\commerce\elements\Order::find()
    ->number($orderNumber)
    ->one();
```
:::


#### `offset`

Determines how many orders should be skipped in the results.



::: code
```twig
{# Fetch all orders except for the first 3 #}
{% set orders = craft.orders()
  .offset(3)
  .all() %}
```

```php
// Fetch all orders except for the first 3
$orders = \craft\commerce\elements\Order::find()
    ->offset(3)
    ->all();
```
:::


#### `orderBy`

Determines the order that the orders should be returned in. (If empty, defaults to `id ASC`.)



::: code
```twig
{# Fetch all orders in order of date created #}
{% set orders = craft.orders()
  .orderBy('dateCreated ASC')
  .all() %}
```

```php
// Fetch all orders in order of date created
$orders = \craft\commerce\elements\Order::find()
    ->orderBy('dateCreated ASC')
    ->all();
```
:::


#### `orderLanguage`

Narrows the query results based on the order language, per the language string provided.

Possible values include:

| Value | Fetches orders…
| - | -
| `'en'` | with an order language that is `'en'`.
| `'not en'` | not with an order language that is not `'en'`.
| `['en', 'en-us']` | with an order language that is `'en'` or `'en-us'`.
| `['not', 'en']` | not with an order language that is not `'en'`.



::: code
```twig
{# Fetch orders with an order language that is `'en'` #}
{% set orders = craft.orders()
  .orderLanguage('en')
  .all() %}
```

```php
// Fetch orders with an order language that is `'en'`
$orders = \craft\commerce\elements\Order::find()
    ->orderLanguage('en')
    ->all();
```
:::


#### `orderSiteId`

Narrows the query results based on the order language, per the language string provided.

Possible values include:

| Value | Fetches orders…
| - | -
| `1` | with an order site ID of 1.
| `'not 1'` | not with an order site ID that is no 1.
| `[1, 2]` | with an order site ID of 1 or 2.
| `['not', 1, 2]` | not with an order site ID of 1 or 2.



::: code
```twig
{# Fetch orders with an order site ID of 1 #}
{% set orders = craft.orders()
  .orderSiteId(1)
  .all() %}
```

```php
// Fetch orders with an order site ID of 1
$orders = \craft\commerce\elements\Order::find()
    ->orderSiteId(1)
    ->all();
```
:::


#### `orderStatus`

Narrows the query results based on the order statuses.

Possible values include:

| Value | Fetches orders…
| - | -
| `'foo'` | with an order status with a handle of `foo`.
| `'not foo'` | not with an order status with a handle of `foo`.
| `['foo', 'bar']` | with an order status with a handle of `foo` or `bar`.
| `['not', 'foo', 'bar']` | not with an order status with a handle of `foo` or `bar`.
| a [OrderStatus](commerce4:craft\commerce\models\OrderStatus) object | with an order status represented by the object.



::: code
```twig
{# Fetch shipped orders #}
{% set orders = craft.orders()
  .orderStatus('shipped')
  .all() %}
```

```php
// Fetch shipped orders
$orders = \craft\commerce\elements\Order::find()
    ->orderStatus('shipped')
    ->all();
```
:::


#### `orderStatusId`

Narrows the query results based on the order statuses, per their IDs.

Possible values include:

| Value | Fetches orders…
| - | -
| `1` | with an order status with an ID of 1.
| `'not 1'` | not with an order status with an ID of 1.
| `[1, 2]` | with an order status with an ID of 1 or 2.
| `['not', 1, 2]` | not with an order status with an ID of 1 or 2.



::: code
```twig
{# Fetch orders with an order status with an ID of 1 #}
{% set orders = craft.orders()
  .orderStatusId(1)
  .all() %}
```

```php
// Fetch orders with an order status with an ID of 1
$orders = \craft\commerce\elements\Order::find()
    ->orderStatusId(1)
    ->all();
```
:::


#### `origin`

Narrows the query results based on the origin.

Possible values include:

| Value | Fetches orders…
| - | -
| `'web'` | with an origin of `web`.
| `'not remote'` | not with an origin of `remote`.
| `['web', 'cp']` | with an order origin of `web` or `cp`.
| `['not', 'remote', 'cp']` | not with an origin of `web` or `cp`.



::: code
```twig
{# Fetch shipped orders #}
{% set orders = craft.orders()
  .origin('web')
  .all() %}
```

```php
// Fetch shipped orders
$orders = \craft\commerce\elements\Order::find()
    ->origin('web')
    ->all();
```
:::


#### `preferSites`

If [unique()](https://docs.craftcms.com/api/v4/craft-elements-db-elementquery.html#method-unique) is set, this determines which site should be selected when querying multi-site elements.



For example, if element “Foo” exists in Site A and Site B, and element “Bar” exists in Site B and Site C,
and this is set to `['c', 'b', 'a']`, then Foo will be returned for Site B, and Bar will be returned
for Site C.

If this isn’t set, then preference goes to the current site.



::: code
```twig
{# Fetch unique orders from Site A, or Site B if they don’t exist in Site A #}
{% set orders = craft.orders()
  .site('*')
  .unique()
  .preferSites(['a', 'b'])
  .all() %}
```

```php
// Fetch unique orders from Site A, or Site B if they don’t exist in Site A
$orders = \craft\commerce\elements\Order::find()
    ->site('*')
    ->unique()
    ->preferSites(['a', 'b'])
    ->all();
```
:::


#### `prepareSubquery`

Prepares the element query and returns its subquery (which determines what elements will be returned).






#### `reference`

Narrows the query results based on the order reference.

Possible values include:

| Value | Fetches orders…
| - | -
| `'Foo'` | with a reference of `Foo`.
| `'Foo*'` | with a reference that begins with `Foo`.
| `'*Foo'` | with a reference that ends with `Foo`.
| `'*Foo*'` | with a reference that contains `Foo`.
| `'not *Foo*'` | with a reference that doesn’t contain `Foo`.
| `['*Foo*', '*Bar*']` | with a reference that contains `Foo` or `Bar`.
| `['not', '*Foo*', '*Bar*']` | with a reference that doesn’t contain `Foo` or `Bar`.



::: code
```twig
{# Fetch the requested order #}
{% set orderReference = craft.app.request.getQueryParam('ref') %}
{% set order = craft.orders()
  .reference(orderReference)
  .one() %}
```

```php
// Fetch the requested order
$orderReference = Craft::$app->request->getQueryParam('ref');
$order = \craft\commerce\elements\Order::find()
    ->reference($orderReference)
    ->one();
```
:::


#### `relatedTo`

Narrows the query results to only orders that are related to certain other elements.



See [Relations](https://craftcms.com/docs/4.x/relations.html) for a full explanation of how to work with this parameter.



::: code
```twig
{# Fetch all orders that are related to myCategory #}
{% set orders = craft.orders()
  .relatedTo(myCategory)
  .all() %}
```

```php
// Fetch all orders that are related to $myCategory
$orders = \craft\commerce\elements\Order::find()
    ->relatedTo($myCategory)
    ->all();
```
:::


#### `search`

Narrows the query results to only orders that match a search query.



See [Searching](https://craftcms.com/docs/4.x/searching.html) for a full explanation of how to work with this parameter.



::: code
```twig
{# Get the search query from the 'q' query string param #}
{% set searchQuery = craft.app.request.getQueryParam('q') %}

{# Fetch all orders that match the search query #}
{% set orders = craft.orders()
  .search(searchQuery)
  .all() %}
```

```php
// Get the search query from the 'q' query string param
$searchQuery = \Craft::$app->request->getQueryParam('q');

// Fetch all orders that match the search query
$orders = \craft\commerce\elements\Order::find()
    ->search($searchQuery)
    ->all();
```
:::


#### `shippingMethodHandle`

Narrows the query results based on the shipping method handle.

Possible values include:

| Value | Fetches orders…
| - | -
| `'foo'` | with a shipping method with a handle of `foo`.
| `'not foo'` | not with a shipping method with a handle of `foo`.
| `['foo', 'bar']` | with a shipping method with a handle of `foo` or `bar`.
| `['not', 'foo', 'bar']` | not with a shipping method with a handle of `foo` or `bar`.
| a `\craft\commerce\elements\db\ShippingMethod` object | with a shipping method represented by the object.



::: code
```twig
{# Fetch collection shipping method orders #}
{% set orders = craft.orders()
  .shippingMethodHandle('collection')
  .all() %}
```

```php
// Fetch collection shipping method orders
$orders = \craft\commerce\elements\Order::find()
    ->shippingMethodHandle('collection')
    ->all();
```
:::


#### `shortNumber`

Narrows the query results based on the order short number.

Possible values include:

| Value | Fetches orders…
| - | -
| `'xxxxxxx'` | with a matching order number



::: code
```twig
{# Fetch the requested order #}
{% set orderNumber = craft.app.request.getQueryParam('shortNumber') %}
{% set order = craft.orders()
  .shortNumber(orderNumber)
  .one() %}
```

```php
// Fetch the requested order
$orderNumber = Craft::$app->request->getQueryParam('shortNumber');
$order = \craft\commerce\elements\Order::find()
    ->shortNumber($orderNumber)
    ->one();
```
:::


#### `siteSettingsId`

Narrows the query results based on the orders’ IDs in the `elements_sites` table.



Possible values include:

| Value | Fetches orders…
| - | -
| `1` | with an `elements_sites` ID of 1.
| `'not 1'` | not with an `elements_sites` ID of 1.
| `[1, 2]` | with an `elements_sites` ID of 1 or 2.
| `['not', 1, 2]` | not with an `elements_sites` ID of 1 or 2.



::: code
```twig
{# Fetch the order by its ID in the elements_sites table #}
{% set order = craft.orders()
  .siteSettingsId(1)
  .one() %}
```

```php
// Fetch the order by its ID in the elements_sites table
$order = \craft\commerce\elements\Order::find()
    ->siteSettingsId(1)
    ->one();
```
:::


#### `total`

Narrows the query results based on the total.

Possible values include:

| Value | Fetches orders…
| - | -
| `10` | with a total price of .
| `['and', 10, 20]` | an order with a total of  or .




#### `totalDiscount`

Narrows the query results based on the total discount.

Possible values include:

| Value | Fetches orders…
| - | -
| `10` | with a total discount of 10.
| `[10, 20]` | an order with a total discount of 10 or 20.




#### `totalPaid`

Narrows the query results based on the total paid amount.

Possible values include:

| Value | Fetches orders…
| - | -
| `10` | with a total paid amount of .
| `['and', 10, 20]` | an order with a total paid amount of  or .




#### `totalPrice`

Narrows the query results based on the total price.

Possible values include:

| Value | Fetches orders…
| - | -
| `10` | with a total price of .
| `['and', 10, 20]` | an order with a total price of  or .




#### `totalQty`

Narrows the query results based on the total qty of items.

Possible values include:

| Value | Fetches orders…
| - | -
| `10` | with a total qty of 10.
| `[10, 20]` | an order with a total qty of 10 or 20.




#### `totalTax`

Narrows the query results based on the total tax.

Possible values include:

| Value | Fetches orders…
| - | -
| `10` | with a total tax of 10.
| `[10, 20]` | an order with a total tax of 10 or 20.




#### `trashed`

Narrows the query results to only orders that have been soft-deleted.





::: code
```twig
{# Fetch trashed orders #}
{% set orders = craft.orders()
  .trashed()
  .all() %}
```

```php
// Fetch trashed orders
$orders = \craft\commerce\elements\Order::find()
    ->trashed()
    ->all();
```
:::


#### `uid`

Narrows the query results based on the orders’ UIDs.





::: code
```twig
{# Fetch the order by its UID #}
{% set order = craft.orders()
  .uid('xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx')
  .one() %}
```

```php
// Fetch the order by its UID
$order = \craft\commerce\elements\Order::find()
    ->uid('xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx')
    ->one();
```
:::


#### `with`

Causes the query to return matching orders eager-loaded with related elements.



See [Eager-Loading Elements](https://craftcms.com/docs/4.x/dev/eager-loading-elements.html) for a full explanation of how to work with this parameter.



::: code
```twig
{# Fetch orders eager-loaded with the "Related" field’s relations #}
{% set orders = craft.orders()
  .with(['related'])
  .all() %}
```

```php
// Fetch orders eager-loaded with the "Related" field’s relations
$orders = \craft\commerce\elements\Order::find()
    ->with(['related'])
    ->all();
```
:::


#### `withAddresses`

Eager loads the the shipping and billing addressees on the resulting orders.

Possible values include:

| Value | Fetches addresses
| - | -
| bool | `true` to eager-load, `false` to not eager load.




#### `withAdjustments`

Eager loads the order adjustments on the resulting orders.

Possible values include:

| Value | Fetches adjustments
| - | -
| bool | `true` to eager-load, `false` to not eager load.




#### `withAll`

Eager loads all relational data (addresses, adjustents, customers, line items, transactions) for the resulting orders.

Possible values include:

| Value | Fetches addresses, adjustents, customers, line items, transactions
| - | -
| bool | `true` to eager-load, `false` to not eager load.




#### `withCustomer`

Eager loads the user on the resulting orders.

Possible values include:

| Value | Fetches adjustments
| - | -
| bool | `true` to eager-load, `false` to not eager load.




#### `withLineItems`

Eager loads the line items on the resulting orders.

Possible values include:

| Value | Fetches line items
| - | -
| bool | `true` to eager-load, `false` to not eager load.




#### `withTransactions`

Eager loads the transactions on the resulting orders.

Possible values include:

| Value | Fetches transactions…
| - | -
| bool | `true` to eager-load, `false` to not eager load.





<!-- END PARAMS -->
