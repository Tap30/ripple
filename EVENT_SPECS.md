# Event Specification & Implementation Guide

## Implementation Guide

This document defines the structured event specification for tracking user
behavior, ecommerce activities, and system operations. It outlines the core
payload structures and provides strict guidelines on when and why to trigger
specific events.

### 1. Core Architecture

Every event sent to Ripple must conform to the root `Event<T>` wrapper. This
ensures top-level consistency for session tracking and identity resolution.

```typescript
/**
 * Represents a generic payload for any event.
 */
type EventPayload = Record<string, unknown>;

/**
 * Information about a specific platform or environment.
 */
type PlatformInfo = {
  /**
   * The name of the platform/browser/os (e.g., "Chrome", "iOS").
   */
  name: string;
  /**
   * The semantic version or release number.
   */
  version: string;
};

/**
 * Web-specific platform details.
 */
type WebPlatform = {
  /**
   * Identifies the platform type.
   */
  type: "web";
  /**
   * Information about the user's browser.
   */
  browser: PlatformInfo;
  /**
   * Information about the hardware device.
   */
  device: PlatformInfo;
  /**
   * Information about the operating system.
   */
  os: PlatformInfo;
};

/**
 * Native app-specific platform details.
 */
type NativePlatform = {
  /**
   * Identifies the platform type.
   */
  type: "native";
  /**
   * Information about the hardware device.
   */
  device: PlatformInfo;
  /**
   * Information about the operating system.
   */
  os: PlatformInfo;
};

/**
 * Server-side platform details.
 */
type ServerPlatform = {
  /**
   * Identifies the platform type.
   */
  type: "server";
};

/**
 * The platform from which the event was issued.
 */
type Platform = WebPlatform | NativePlatform | ServerPlatform;

/**
 * Information about the tracking SDK sending the event.
 */
type SdkInfo = {
  /**
   * The name of the SDK (e.g., "ripple-ts").
   */
  name: string;
  /**
   * The version of the SDK.
   */
  version: string;
};

/**
 * The root Event object shape.
 * @template TMetadata Defines the shape of global/app-level metadata.
 */
type Event<TMetadata = Record<string, unknown>> = {
  /**
   * The unique name of the event (e.g., "product_viewed").
   */
  name: string;
  /**
   * The event-specific data.
   */
  payload: EventPayload | null;
  /**
   * Environment/App-specific data attached to all events (e.g., app version, build number).
   */
  metadata: TMetadata | null;
  /**
   * Platform context.
   */
  platform: Platform | null;
  /**
   * SDK context.
   */
  sdk: SdkInfo;
  /**
   * UNIX timestamp in milliseconds indicating when the event occurred.
   */
  issuedAt: number;
  /**
   * A unique identifier for the anonymous user/device.
   */
  anonymousId: string;
  /**
   * A unique UUID for this specific event to prevent deduplication.
   */
  eventId: string;
  /**
   * The version of the tracking schema being used.
   */
  schemaVersion: string | null;
  /**
   * The authenticated user's ID, if known.
   */
  userId: string | null;
};
```

### 2. Shared Types

To maintain data consistency across different events, the following shared
objects must be used whenever referencing monetary values, products, orders,
campaigns, or other common entities.

```typescript
/**
 * Represents a primitive value.
 */
type Primitive = number | boolean | string;

/**
 * Represents monetary values.
 */
type Money = {
  /**
   * The decimal value of the money.
   */
  amount: number;
  /**
   * The ISO 4217 currency code (e.g., "USD").
   */
  currency: string;
};

/**
 * Represents a product category.
 */
type Category = {
  /**
   * Unique category identifier.
   */
  id: string;
  /**
   * Human-readable category name.
   */
  title?: string;
};

/**
 * Represents a physical address.
 */
type Address = {
  /**
   * Complete formatted address.
   */
  fullAddress: string;
  /**
   * City name.
   */
  city: string;
  /**
   * Geographical coordinates.
   */
  location?: {
    /**
     * Latitude coordinate.
     */
    lat: number;
    /**
     * Longitude coordinate.
     */
    long: number;
  };
};

/**
 * Represents shipping details for an order.
 */
type Shipping = {
  /**
   * Cost of shipping.
   */
  price: number;
  /**
   * Shipping method name (e.g., "Standard", "Next Day").
   */
  method: string;
  /**
   * Where the shipment originates.
   */
  origin?: Address;
  /**
   * Where the shipment is going.
   */
  destination: Address;
  /**
   * Estimated arrival information.
   */
  arrival?: {
    /**
     * Exact UNIX timestamp in milliseconds for expected arrival.
     */
    absoluteArriveAt?: number;
    /**
     * Time window for arrival [start, end] in UNIX timestamp milliseconds.
     */
    rangeArriveAt?: [number, number];
  };
  /**
   * Custom additional properties.
   */
  customProperties?: Record<string, Primitive>;
};

/**
 * Represents a discount coupon.
 */
type Coupon = {
  /**
   * The string code applied by the user.
   */
  code: string;
  /**
   * The monetary value discounted.
   */
  amount: Money;
  /**
   * Internal ID of the coupon.
   */
  id?: string;
};

/**
 * Represents an individual purchasable item.
 */
type Product = {
  /**
   * Unique product identifier.
   */
  productId: string;
  /**
   * Name of the product.
   */
  productTitle?: string;
  /**
   * Primary category of the product.
   */
  category?: Category;
  /**
   * Stock Keeping Unit identifier.
   */
  skuId?: string;
  /**
   * Manufacturer or seller.
   */
  vendor?: string;
  /**
   * Current selling price of a single unit.
   */
  price: Money;
  /**
   * Discount applied to this specific product.
   */
  discount?: Money;
  /**
   * Number of items.
   */
  quantity?: number;
  /**
   * The index position of the product in a list/grid.
   */
  position?: number;
  /**
   * Custom additional properties.
   */
  customProperties?: Record<string, Primitive>;
};

/**
 * Represents a finalized or processing order.
 */
type Order = {
  /**
   * Unique order identifier.
   */
  orderId: string;
  /**
   * Associated checkout session identifier.
   */
  checkoutId?: string;
  /**
   * Payment gateway transaction ID.
   */
  transactionId?: string;
  /**
   * Associated cart identifier.
   */
  cartId?: string;
  /**
   * Final total value charged to the customer.
   */
  totalValue: Money;
  /**
   * Total discount applied to the order.
   */
  totalDiscount?: Money;
  /**
   * Value of the order before taxes and shipping.
   */
  subtotalValue?: Money;
  /**
   * Tax amount applied.
   */
  tax?: Money;
  /**
   * Shipping details and costs.
   */
  shipping?: Shipping;
  /**
   * Method used for payment (e.g., "Credit Card", "PayPal").
   */
  paymentMethod?: string;
  /**
   * Incentives applied to this order.
   */
  incentives?: Incentive[];
  /**
   * Array of products purchased.
   */
  products: Product[];
  /**
   * Custom additional properties.
   */
  customProperties?: Record<string, Primitive>;
};

/**
 * Represents an active checkout session.
 */
type Checkout = {
  /**
   * Unique checkout session identifier.
   */
  checkoutId?: string;
  /**
   * Order details populated so far (excludes finalized IDs).
   */
  order: Omit<Order, "orderId" | "transactionId" | "checkoutId">;
  /**
   * Current step in the checkout funnel (e.g., 2 or "shipping").
   */
  step: number | string;
  /**
   * Custom additional properties.
   */
  customProperties?: Record<string, Primitive>;
};

/**
 * Pagination details for list views.
 */
type Pagination = {
  /**
   * The current page number.
   */
  page: number;
  /**
   * The number of items per page.
   */
  limit: number;
};

/**
 * Key-value filter applied to a list.
 */
type Filter = {
  /**
   * The filter parameter key.
   */
  key: string;
  /**
   * The filter parameter value.
   */
  value: string;
};

/**
 * Sorting parameter applied to a list.
 */
type Sort = {
  /**
   * The sorting parameter key.
   */
  key: string;
  /**
   * The sorting direction.
   */
  value: "asc" | "dsc";
};

/**
 * Represents a payment transaction.
 */
type Payment = {
  /**
   * Unique payment identifier.
   */
  paymentId: string;
  /**
   * Payment gateway used.
   */
  gateway?: string;
  /**
   * Method of payment (e.g., "credit_card").
   */
  method: string;
  /**
   * Amount processed in this payment.
   */
  amount: Money;
  /**
   * The order associated with this payment.
   */
  order?: Order;
};

/**
 * Represents marketing campaign parameters (usually parsed from UTMs).
 */
type Campaign = {
  /**
   * The referrer or entity sending the traffic (utm_source).
   */
  source: string;
  /**
   * The marketing medium (utm_medium).
   */
  medium: string;
  /**
   * The specific campaign name (utm_campaign).
   */
  name?: string;
  /**
   * Identify the paid keywords (utm_term).
   */
  term?: string;
  /**
   * Differentiate similar content or links within the same ad (utm_content).
   */
  content?: string;
};

/**
 * Represents an incentive reward value.
 */
type Reward = {
  /**
   * The numerical amount of the reward.
   */
  amount: number;
  /**
   * The unit of the reward (e.g., "points", "dollars").
   */
  unit: string;
};

/**
 * Represents an incentive (e.g., loyalty points, cashback).
 */
type Incentive = {
  /**
   * Unique identifier for the incentive.
   */
  incentiveId: string;
  /**
   * Human-readable title of the incentive.
   */
  incentiveTitle?: string;
  /**
   * Type of incentive (e.g., "cashback", "points").
   */
  type: string;
  /**
   * The reward value associated with the incentive.
   */
  reward: Reward;
  /**
   * Custom additional properties.
   */
  customProperties?: Record<string, Primitive>;
};

/**
 * Represents a gamification challenge.
 */
type Challenge = {
  /**
   * Unique identifier for the challenge.
   */
  challengeId: string;
  /**
   * Human-readable title of the challenge.
   */
  challengeTitle?: string;
  /**
   * Category grouping for the challenge.
   */
  category?: Category;
  /**
   * Custom additional properties.
   */
  customProperties?: Record<string, Primitive>;
};

/**
 * Represents a referral object.
 */
type Referral = {
  /**
   * The unique referral code used.
   */
  referralCode: string;
  /**
   * The ID of the user who owns the referral code.
   */
  referrerId?: string;
  /**
   * Custom additional properties.
   */
  customProperties?: Record<string, Primitive>;
};
```

---

### 3. Predefined Standard Events

Events that deal with user identity and navigation.

#### Identify (`user_identified`)

Associates a user’s `anonymousId` with their actual database `userId` and
updates their profile traits. **When to trigger:** Immediately after a
successful login, registration, or when a user updates their profile settings.

```typescript
type UserIdentifiedPayload = {
  /**
   * The known database ID of the user.
   */
  userId: string;
  /**
   * User profile attributes.
   */
  traits: {
    /**
     * User's first name.
     */
    firstName?: string;
    /**
     * User's last name.
     */
    lastName?: string;
    /**
     * User's full name.
     */
    fullName?: string;
    /**
     * User's chosen username.
     */
    username?: string;
    /**
     * User's age.
     */
    age?: number;
    /**
     * User's email address.
     */
    email?: string;
    /**
     * User's phone number.
     */
    phone?: string;
    /**
     * User's gender.
     */
    gender?: "male" | "female" | "other";
    /**
     * UNIX timestamp in milliseconds of user's birthday.
     */
    birthday?: number;
    /**
     * UNIX timestamp in milliseconds of account creation.
     */
    createdAt?: number;
    /**
     * User's physical address details.
     */
    address?: {
      /**
       * City name.
       */
      city?: string;
      /**
       * Country code or name.
       */
      country?: string;
      /**
       * State or province.
       */
      state?: string;
      /**
       * Street address.
       */
      street?: string;
      /**
       * User's timezone.
       */
      timezone?: string;
    };
  };
};
```

#### Screen (Web & Mobile) (`screened`)

Records that a user loaded a specific web page or mobile app screen. **Nuance:**
Trigger `screened` for _every_ page load. Trigger `product_viewed` _in addition_
to `screened` when the page loaded is a Product Detail Page (PDP).

```typescript
type WebScreenedPayload = {
  /**
   * The page title.
   */
  title: string;
  /**
   * Full URL of the page.
   */
  url: string;
  /**
   * URL pathname.
   */
  pathname?: string;
  /**
   * Previous page URL.
   */
  referrer?: string;
  /**
   * Query string parameters.
   */
  search?: string;
  /**
   * Extracted SEO keywords.
   */
  keywords?: string[];
  /**
   * Parsed UTM campaign parameters.
   */
  campaign?: Campaign;
  /**
   * Custom additional properties.
   */
  customProperties?: Record<string, Primitive>;
};

type MobileScreenedPayload = {
  /**
   * Name of the screen (e.g., "Home", "Product Detail").
   */
  title: string;
  /**
   * The previous screen name.
   */
  referrer?: string;
  /**
   * Parsed UTM campaign parameters from a deep link.
   */
  campaign?: Campaign;
  /**
   * Custom additional properties.
   */
  customProperties?: Record<string, Primitive>;
};
```

#### App State Changes (`app_state_changed`)

Fired when the application’s visibility or active lifecycle state changes. **Use
case:** Crucial for calculating true active session duration, understanding user
engagement/idle time, and tracking how often users background the app.

```typescript
type AppState = "opened" | "closed" | "foreground" | "background";

type AppStateChangedPayload = {
  /**
   * The new application state.
   */
  newState: AppState;
  /**
   * The previous application state.
   */
  previousState?: AppState;
};
```

#### Click (`clicked`)

Fired when a user clicks, taps, or otherwise interacts with a clickable UI
element. **Use case:** Essential for tracking feature usage, measuring
conversion funnels, and identifying dead clicks.

```typescript
type ClickedPayload = {
  /**
   * The unique identifier of the clicked element (e.g., "submit_checkout_button").
   */
  elementId: string;
  /**
   * The type of element clicked (e.g., "button", "link", "icon").
   */
  elementType?: string;
  /**
   * The visible text or title of the element, if applicable.
   */
  elementTitle?: string;
  /**
   * Custom additional properties.
   */
  customProperties?: Record<string, Primitive>;
};
```

#### View (`viewed`)

Fired when a specific element becomes visible to the user. **Use case:** Crucial
for measuring impressions, mapping the user journey, and analyzing drop-off
rates in funnels.

```typescript
type ViewedPayload = {
  /**
   * The unique identifier of the viewed element (e.g., "services_section").
   */
  elementId: string;
  /**
   * The type of element viewed.
   */
  elementType?: string;
  /**
   * The title of the element, if applicable.
   */
  elementTitle?: string;
  /**
   * Custom additional properties.
   */
  customProperties?: Record<string, Primitive>;
};
```

---

### 4. Predefined Ecommerce Events

#### 4.1 Product Interactions

##### `product_clicked`

Fired when a user clicks on a product card from a list, grid, or recommendation
carousel. **Use case:** Measuring the click-through rate (CTR) of product
thumbnails.

```typescript
type ProductClickedPayload = {
  /**
   * The product that was clicked.
   */
  product: Product;
  /**
   * Custom additional properties.
   */
  customProperties?: Record<string, Primitive>;
};
```

##### `product_viewed`

Fired when the user lands on the actual Product Detail Page (PDP) and sees the
full product description. **Use case:** Tracking deep engagement and viewing
history.

```typescript
type ProductViewedPayload = {
  /**
   * The product that was viewed.
   */
  product: Product;
  /**
   * Custom additional properties.
   */
  customProperties?: Record<string, Primitive>;
};
```

##### `product_shared`

Fired when a user shares a product via a native share sheet, email, or social
media link. **Use case:** Measures virality and identifies products that
generate organic interest.

```typescript
type ProductSharedPayload = {
  /**
   * Platform shared to (e.g., "Twitter", "Email", "WhatsApp").
   */
  sharingMethod?: string;
  /**
   * User-generated message attached to the share.
   */
  message?: string;
  /**
   * Recipient identifier if shared directly.
   */
  recipient?: string;
  /**
   * The product that was shared.
   */
  product: Product;
  /**
   * Custom additional properties.
   */
  customProperties?: Record<string, Primitive>;
};
```

#### 4.2 Cart & Wishlist Modifications

##### `product_added_to_wishlist` / `product_removed_from_wishlist`

Fired when a user toggles an item in their wishlist/favorites. **Use case:**
Wishlist additions signal high interest but low urgency.

```typescript
type ProductWishlistPayload = {
  /**
   * The unique identifier for the user's wishlist.
   */
  wishlistId?: string;
  /**
   * The UI location from where the product was added/removed.
   */
  referrer?: string;
  /**
   * The product being modified in the wishlist.
   */
  product: Product;
  /**
   * Custom additional properties.
   */
  customProperties?: Record<string, Primitive>;
};
```

##### `product_added_to_cart` / `product_removed_from_cart`

Fired when a user modifies the contents of their shopping cart. **Use case:**
Adding to cart is the strongest signal of purchase intent. Removing indicates a
change of mind or price sensitivity.

```typescript
type CartModificationPayload = {
  /**
   * The unique identifier for the cart.
   */
  cartId?: string;
  /**
   * The UI location from where the modification occurred.
   */
  referrer?: string;
  /**
   * The product being added or removed.
   */
  product: Product;
  /**
   * Custom additional properties.
   */
  customProperties?: Record<string, Primitive>;
};
```

##### `product_reviewed`

Fired when a user submits a rating or a written review for a product. **Use
case:** Used to measure product sentiment and track post-purchase engagement.

```typescript
type ProductReviewedPayload = {
  /**
   * The product being reviewed.
   */
  product: Product;
  /**
   * Unique identifier for the review.
   */
  reviewId: string;
  /**
   * Numerical rating given.
   */
  rating: number;
  /**
   * Summary/title of the review.
   */
  title?: string;
  /**
   * Body text of the review.
   */
  body?: string;
  /**
   * Custom additional properties.
   */
  customProperties?: Record<string, Primitive>;
};
```

##### `cart_viewed`

Fired when the user opens the cart sidebar, modal, or navigates to the dedicated
cart page. **Use case:** Shows users reviewing selected items.

```typescript
type CartViewedPayload = {
  /**
   * The unique identifier for the cart.
   */
  cartId?: string;
  /**
   * The current list of products in the cart.
   */
  products: Product[];
  /**
   * Custom additional properties.
   */
  customProperties?: Record<string, Primitive>;
};
```

##### `cart_emptied`

Fired when the user explicitly clears their entire cart, or when the cart
expires. **Use case:** Helps identify session abandonment or user frustration.

```typescript
type CartEmptiedPayload = {
  /**
   * The unique identifier for the cart.
   */
  cartId?: string;
  /**
   * The list of products that were cleared.
   */
  products: Product[];
  /**
   * Custom additional properties.
   */
  customProperties?: Record<string, Primitive>;
};
```

#### 4.3 Ordering & Checkout

##### `checkout_started`

Fired when the user explicitly clicks “Proceed to Checkout” from the cart. **Use
case:** The critical entry point to the conversion funnel.

```typescript
type CheckoutStartedPayload = {
  /**
   * The checkout session details.
   */
  checkout: Checkout;
  /**
   * Custom additional properties.
   */
  customProperties?: Record<string, Primitive>;
};
```

##### `checkout_step_viewed` / `checkout_step_completed`

Fired as the user navigates through a multi-step checkout flow. **Use case:**
Identifies exact abandonment points.

```typescript
type CheckoutStepPayload = {
  /**
   * The checkout session details at this step.
   */
  checkout: Checkout;
  /**
   * Custom additional properties.
   */
  customProperties?: Record<string, Primitive>;
};
```

##### `order_completed`

Fired when the checkout is successfully submitted and payment is authorized.
**Use case:** The primary conversion and revenue metric.

```typescript
type OrderCompletedPayload = {
  /**
   * The finalized order details.
   */
  order: Order;
  /**
   * Custom additional properties.
   */
  customProperties?: Record<string, Primitive>;
};
```

##### `order_failed`

Fired when a payment is attempted but rejected (e.g., insufficient funds,
network timeout). **Use case:** Monitoring payment gateway health and
identifying user-facing payment errors.

```typescript
type OrderFailedPayload = {
  /**
   * The order that failed to process.
   */
  order: Order;
  /**
   * Detailed reason for the failure.
   */
  reason: string;
  /**
   * Custom additional properties.
   */
  customProperties?: Record<string, Primitive>;
};
```

##### `order_cancelled`

Fired when an order is created but later aborted either by the customer or an
admin.

```typescript
type OrderCancelledPayload = {
  /**
   * The order being cancelled.
   */
  order: Order;
  /**
   * Who cancelled the order (e.g., "customer", "admin", "system").
   */
  issuer?: string;
  /**
   * Custom additional properties.
   */
  customProperties?: Record<string, Primitive>;
};
```

##### `order_shipped`

Fired by the backend when the physical items leave the warehouse. **Use case:**
Measures operational efficiency.

```typescript
type OrderShippedPayload = {
  /**
   * The order that was shipped.
   */
  order: Order;
  /**
   * Custom additional properties.
   */
  customProperties?: Record<string, Primitive>;
};
```

##### `order_product_fulfilled`

Fired when an individual item within an order is fulfilled (useful for split
shipments).

```typescript
type OrderProductFulfilledPayload = {
  /**
   * The overall order.
   */
  order: Order;
  /**
   * The specific product within the order that was fulfilled.
   */
  product: Product;
  /**
   * Custom additional properties.
   */
  customProperties?: Record<string, Primitive>;
};
```

##### `order_refunded` / `order_product_returned`

Fired when an order is partially or fully refunded. **Use case:** Crucial for
calculating net metrics, e.g., $netRevenue = grossRevenue - returns$.

```typescript
type OrderRefundedPayload = {
  /**
   * The order that was refunded.
   */
  order: Order;
  /**
   * Custom additional properties.
   */
  customProperties?: Record<string, Primitive>;
};

type OrderProductReturnedPayload = {
  /**
   * The order containing the returned product.
   */
  order: Order;
  /**
   * The specific product being returned.
   */
  product: Product;
  /**
   * Reason for return (e.g., "Defective", "Wrong Size").
   */
  reason: string;
  /**
   * How the customer was refunded (e.g., "Store Credit").
   */
  refundMethod?: string;
  /**
   * The monetary value of the return.
   */
  totalReturnedValue: Money;
  /**
   * Custom additional properties.
   */
  customProperties?: Record<string, Primitive>;
};
```

##### `order_updated`

Fired when an existing order’s details are modified after the initial placement.
**Use case:** Helps track post-purchase modifications and ensure downstream
systems have the latest state.

```typescript
type OrderUpdatedPayload = {
  /**
   * The updated order details.
   */
  order: Order;
  /**
   * The reason for the update.
   */
  reason?: string;
  /**
   * Custom additional properties.
   */
  customProperties?: Record<string, Primitive>;
};
```

##### `order_fulfillment_status_updated`

Fired when an order transitions from one state to another within the fulfillment
lifecycle. **Use case:** Used to measure fulfillment velocity and monitor
operational bottlenecks.

```typescript
type OrderFulfillmentStatusUpdatedPayload = {
  /**
   * The order undergoing a status change.
   */
  order: Order;
  /**
   * The previous fulfillment status.
   */
  previousStatus: string;
  /**
   * The new fulfillment status.
   */
  newStatus: string;
  /**
   * The reason for the status change, if applicable.
   */
  reason?: string;
  /**
   * Custom additional properties.
   */
  customProperties?: Record<string, Primitive>;
};
```

##### `order_reviewed`

Fired when a user submits a rating, feedback, or a written review for an entire
order. **Use case:** Used to measure overall customer satisfaction and evaluate
fulfillment performance.

```typescript
type OrderReviewedPayload = {
  /**
   * The order being reviewed.
   */
  order: Order;
  /**
   * Unique identifier for the submitted review.
   */
  reviewId: string;
  /**
   * The numerical rating given by the user (e.g., 1 to 5).
   */
  rating: number;
  /**
   * The title or summary of the review.
   */
  title?: string;
  /**
   * The full text body of the review.
   */
  body?: string;
  /**
   * Custom additional properties.
   */
  customProperties?: Record<string, Primitive>;
};
```

#### 4.4 Browsing & Discovery

##### `products_searched`

Fired when a user submits a search query. **Use case:** High search volume for
queries that yield zero results indicates a missed product opportunity.

```typescript
type ProductsSearchedPayload = {
  /**
   * The raw text query submitted by the user.
   */
  query: string;
  /**
   * Custom additional properties.
   */
  customProperties?: Record<string, Primitive>;
};
```

##### `product_list_viewed`

Fired when a user views a list of products, such as a category page or search
results page.

```typescript
type ProductListViewedPayload = {
  /**
   * Unique identifier for the product list/grid.
   */
  listId?: string;
  /**
   * The category being viewed, if applicable.
   */
  category?: Category;
  /**
   * Pagination state of the list.
   */
  pagination?: Pagination;
  /**
   * The products currently visible in the list.
   */
  products: Product[];
  /**
   * Custom additional properties.
   */
  customProperties?: Record<string, Primitive>;
};
```

##### `product_list_filtered`

Fired when a user applies filters or sorting to a product list. **Use case:**
Measures how heavily users rely on faceted navigation.

```typescript
type ProductListFilteredPayload = {
  /**
   * Unique identifier for the product list being filtered.
   */
  listId?: string;
  /**
   * The active category context.
   */
  category?: Category;
  /**
   * The active filters applied.
   */
  filters: Filter[];
  /**
   * The active sorting parameters applied.
   */
  sorts: Sort[];
  /**
   * The resulting products after filtering/sorting.
   */
  products: Product[];
  /**
   * Custom additional properties.
   */
  customProperties?: Record<string, Primitive>;
};
```

#### 4.5 Coupons

##### `coupon_entered` / `coupon_removed`

Fired when a user attempts to apply or removes a coupon code during checkout.
**Use case:** Tracks intent to find a discount.

```typescript
type CouponEnteredRemovedPayload = {
  /**
   * The coupon being entered or removed.
   */
  coupon: Coupon;
  /**
   * The checkout session context.
   */
  checkout: Checkout;
  /**
   * Custom additional properties.
   */
  customProperties?: Record<string, Primitive>;
};
```

##### `coupon_denied`

Fired when the system rejects an applied coupon code. **Use case:** High rates
of `coupon_denied` indicate confusing rules or expired codes.

```typescript
type CouponDeniedPayload = {
  /**
   * The coupon that was rejected.
   */
  coupon: Coupon;
  /**
   * The checkout session context.
   */
  checkout: Checkout;
  /**
   * Reason the coupon was denied.
   */
  reason: string;
  /**
   * Custom additional properties.
   */
  customProperties?: Record<string, Primitive>;
};
```

#### 4.6 Promotions

##### `promotion_viewed` / `promotion_clicked`

Fired when a user sees or clicks an internal site banner. **Use case:**
Calculating the CTR of internal marketing real estate to detect "banner
blindness."

```typescript
type PromotionPayload = {
  /**
   * Unique identifier for the promotion/banner.
   */
  promotionId: string;
  /**
   * Human-readable title of the promotion.
   */
  promotionTitle?: string;
  /**
   * Custom additional properties.
   */
  customProperties?: Record<string, Primitive>;
};
```

#### 4.7 Payment

##### `payment_authorized`

Fired when the payment gateway successfully authorizes a hold on the user’s
funds. **Use case:** Differentiates between a user clicking “Pay” and the bank
actually approving the transaction.

```typescript
type PaymentAuthorized = {
  /**
   * The payment authorization details.
   */
  payment: Payment;
  /**
   * Custom additional properties.
   */
  customProperties?: Record<string, Primitive>;
};
```

##### `payment_captured`

Fired when the payment gateway successfully processes the charge and transfers
funds. **Use case:** Represents actual cash flow and used to verify realized
revenue.

```typescript
type PaymentCaptured = {
  /**
   * The payment capture details.
   */
  payment: Payment;
  /**
   * Custom additional properties.
   */
  customProperties?: Record<string, Primitive>;
};
```

##### `payment_failed`

Fired when a payment attempt is rejected by the gateway or the issuing bank.
**Use case:** Monitor
$GatewayErrorRate = payment\_failed / (payment\_authorized + payment\_failed)$
to detect integration outages.

```typescript
type PaymentFailed = {
  /**
   * The failed payment details.
   */
  payment: Payment;
  /**
   * Custom additional properties.
   */
  customProperties?: Record<string, Primitive>;
};
```

##### `payment_refunded`

Fired when funds are returned to the user’s payment method. **Use case:**
Essential for calculating $NetCashFlow = payment\_captured - payment\_refunded$.

```typescript
type PaymentRefunded = {
  /**
   * The refunded payment details.
   */
  payment: Payment;
  /**
   * Reason for the refund.
   */
  reason?: string;
  /**
   * The amount returned in this specific transaction.
   */
  returnedAmount: Money;
  /**
   * Custom additional properties.
   */
  customProperties?: Record<string, Primitive>;
};
```

#### 4.8 Referral

##### `referral_shared`

Fired when a user shares their referral code or link with others. **Use case:**
Measure the top of the referral funnel and identify which share channels are
most popular.

```typescript
type ReferralSharedPayload = {
  /**
   * The referral object that was shared.
   */
  referral: Referral;
  /**
   * The medium through which it was shared (e.g., "whatsapp", "email").
   */
  medium?: string;
  /**
   * Custom additional properties.
   */
  customProperties?: Record<string, Primitive>;
};
```

##### `referral_applied`

Fired when a new or existing user successfully enters a referral code. **Use
case:** Attribute new user acquisition to specific referrers and calculate
conversion rates.

```typescript
type ReferralAppliedPayload = {
  /**
   * The referral object that was applied.
   */
  referral: Referral;
  /**
   * The UI flow or surface where the referral was applied (e.g., "checkout", "registration").
   */
  flow?: string;
  /**
   * Custom additional properties.
   */
  customProperties?: Record<string, Primitive>;
};
```

#### 4.9 Incentive

##### `incentive_granted`

Fired when the system automatically awards an incentive to a user without
requiring a manual “claim” action. **Use case:** Calculate program ROI by
comparing distributed reward liability against subsequent purchases.

```typescript
type IncentiveGrantedPayload = {
  /**
   * The incentive that was granted.
   */
  incentive: Incentive;
  /**
   * Identifier for the source triggering the grant (e.g., Order ID).
   */
  sourceId?: string;
  /**
   * Descriptive title of the source.
   */
  sourceTitle?: string;
  /**
   * Custom additional properties.
   */
  customProperties?: Record<string, Primitive>;
};
```

##### `incentive_redeemed`

Fired when a user spends or applies their earned incentives. **Use Case:**
Calculate loyalty velocity and track the depletion of outstanding financial
liabilities.

```typescript
type IncentiveRedeemedPayload = {
  /**
   * The incentive being redeemed.
   */
  incentive: Incentive;
  /**
   * Identifier for what consumed the incentive (e.g., Order ID).
   */
  redeemerId?: string;
  /**
   * Descriptive title of the redeemer.
   */
  redeemerTitle?: string;
  /**
   * Custom additional properties.
   */
  customProperties?: Record<string, Primitive>;
};
```

##### `incentive_claimed`

Fired when a user explicitly interacts with the UI to collect a reward that is
available to them. **Use Case:** Identify highly engaged users and measure the
adoption rate of gamified collection mechanics.

```typescript
type IncentiveClaimedPayload = {
  /**
   * The incentive that was claimed.
   */
  incentive: Incentive;
  /**
   * Identifier for the origin of the claimable reward.
   */
  sourceId?: string;
  /**
   * Descriptive title of the source.
   */
  sourceTitle?: string;
  /**
   * Custom additional properties.
   */
  customProperties?: Record<string, Primitive>;
};
```

##### `incentive_expired`

Fired by the server when an unused incentive passes its validity period and is
removed from the user’s balance. **Use Case:** Recognize unredeemed liability
(breakage) and measure win-back campaign effectiveness.

```typescript
type IncentiveExpiredPayload = {
  /**
   * The incentive that expired.
   */
  incentive: Incentive;
  /**
   * Custom additional properties.
   */
  customProperties?: Record<string, Primitive>;
};
```

#### 4.10 Gamification

##### `challenge_started`

Fired when a user opts into, or automatically triggers the beginning of, a
specific challenge/quest. **Use Case:** Understand gamification participation
rates and compare challenge appeal.

```typescript
type ChallengeStartedPayload = {
  /**
   * The challenge that was started.
   */
  challenge: Challenge;
  /**
   * Custom additional properties.
   */
  customProperties?: Record<string, Primitive>;
};
```

##### `challenge_completed`

Fired when the user successfully fulfills all requirements of a challenge. **Use
Case:** Measure overall challenge success rates and long-term user behavior
shifts.

```typescript
type ChallengeCompletedPayload = {
  /**
   * The challenge that was completed.
   */
  challenge: Challenge;
  /**
   * Custom additional properties.
   */
  customProperties?: Record<string, Primitive>;
};
```

##### `challenge_step_completed`

Fired when a user hits a milestone or completes a sub-task within a broader
challenge. **Use Case:** Build gamification funnel reports to identify where
users lose motivation.

```typescript
type ChallengeStepCompletedPayload = {
  /**
   * The challenge containing the step.
   */
  challenge: Challenge;
  /**
   * The identifier or index of the step completed.
   */
  step: string | number;
  /**
   * Custom additional properties.
   */
  customProperties?: Record<string, Primitive>;
};
```
