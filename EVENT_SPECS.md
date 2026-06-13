# Event Specification & Implementation Guide

## Implementation Guide

This document defines the structured event specification for tracking user
behavior, ecommerce activities, and system operations. It outlines the core
payload structures and provides strict guidelines on when and why to trigger
specific events.

### 1. Core Architecture

Every event sent to the Ripple must conform to the root `Event<T>` wrapper. This
ensures top-level consistency for session tracking and identity resolution.

```ts
/**
 * Represents a generic payload for any event.
 */
type EventPayload = Record<string, unknown>;

/**
 * Information about a specific platform or environment.
 */
type PlatformInfo = {
  /** The name of the platform/browser/os (e.g., "Chrome", "iOS"). */
  name: string;
  /** The semantic version or release number. */
  version: string;
};

type WebPlatform = {
  type: "web";
  browser: PlatformInfo;
  device: PlatformInfo;
  os: PlatformInfo;
};

type NativePlatform = {
  type: "native";
  device: PlatformInfo;
  os: PlatformInfo;
};

type ServerPlatform = {
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
  /** The name of the SDK (e.g., "ripple-ts"). */
  name: string;
  /** The version of the SDK. */
  version: string;
};

/**
 * The root Event object shape.
 * @template TMetadata Defines the shape of global/app-level metadata.
 */
type Event<TMetadata = Record<string, unknown>> = {
  /** The unique name of the event (e.g., "product_viewed"). */
  name: string;
  /** The event-specific data. */
  payload: EventPayload | null;
  /** Environment/App-specific data attached to all events (e.g., app version, build number). */
  metadata: TMetadata | null;
  /** Platform context. */
  platform: Platform | null;
  /** SDK context. */
  sdk: SdkInfo;
  /** UNIX timestamp in milliseconds indicating when the event occurred. */
  issuedAt: number;
  /** A unique identifier for the anonymous user/device. */
  anonymousId: string;
  /** A unique UUID for this specific event to prevent deduplication. */
  eventId: string;
  /** The version of the tracking schema being used. */
  schemaVersion: string | null;
  /** The authenticated user's ID, if known. */
  userId: string | null;
};
```

### 2. Shared Types

To maintain data consistency across different events, the following shared
objects must be used whenever referencing monetary values, products, orders, or
browsing helpers.

```ts
/** Represents a primitive value. */
type Primitive = number | boolean | string;

/** Represents monetary values. */
type Money = {
  /** The decimal value of the money. */
  amount: number;
  /** The ISO 4217 currency code (e.g., "USD"). */
  currency: string;
};

/** Represents a product category. */
type Category = {
  /** Unique category identifier. */
  id: string;
  /** Human-readable category name. */
  title?: string;
};

/** Represents a physical address. */
type Address = {
  /** Complete formatted address. */
  fullAddress: string;
  /** City name. */
  city: string;
  /** Geographical coordinates. */
  location?: { lat: number; long: number };
};

/** Represents shipping details for an order. */
type Shipping = {
  /** Cost of shipping. */
  price: number;
  /** Shipping method name (e.g., "Standard", "Next Day"). */
  method: string;
  /** Where the shipment originates. */
  origin?: Address;
  /** Where the shipment is going. */
  destination: Address;
  /** Estimated arrival information. */
  arrival?: {
    /** Exact UNIX timestamp in milliseconds for expected arrival. */
    absoluteArriveAt?: number;
    /** Time window for arrival [start, end] in UNIX timestamp milliseconds. */
    rangeArriveAt?: [number, number];
  };
  customProperties?: Record<string, Primitive>;
};

/** Represents a discount coupon. */
type Coupon = {
  /** The string code applied by the user. */
  code: string;
  /** The monetary value discounted. */
  amount: Money;
  /** Internal ID of the coupon. */
  id?: string;
};

/** Represents an individual purchasable item. */
type Product = {
  /** Unique product identifier. */
  productId: string;
  /** Name of the product. */
  productTitle?: string;
  /** Primary category of the product. */
  category?: Category;
  /** Stock Keeping Unit identifier. */
  skuId?: string;
  /** Manufacturer or seller. */
  vendor?: string;
  /** Current selling price of a single unit. */
  price: Money;
  /** Discount applied to this specific product. */
  discount?: Money;
  /** Number of items. */
  quantity?: number;
  /** The index position of the product in a list/grid. */
  position?: number;
  customProperties?: Record<string, Primitive>;
};

/** Represents a finalized or processing order. */
type Order = {
  /** Unique order identifier. */
  orderId: string;
  /** Associated checkout session identifier. */
  checkoutId?: string;
  /** Payment gateway transaction ID. */
  transactionId?: string;
  /** Associated cart identifier. */
  cartId?: string;
  /** Final total value charged to the customer. */
  totalValue: Money;
  /** Total discount applied to the order. */
  totalDiscount?: Money;
  /** Value of the order before taxes and shipping. */
  subtotalValue?: Money;
  /** Tax amount applied. */
  tax?: Money;
  /** Shipping details and costs. */
  shipping?: Shipping;
  /** Method used for payment (e.g., "Credit Card", "PayPal"). */
  paymentMethod?: string;
  /** Coupon applied to the entire order. */
  coupon?: Coupon;
  /** Array of products purchased. */
  products: Product[];
  customProperties?: Record<string, Primitive>;
};

/** Represents an active checkout session. */
type Checkout = {
  /** Unique checkout session identifier. */
  checkoutId?: string;
  /** Order details populated so far (excludes finalized IDs). */
  order: Omit<Order, "orderId" | "transactionId" | "checkoutId">;
  /** Current step in the checkout funnel (e.g., 2 or "shipping"). */
  step: number | string;
  customProperties?: Record<string, Primitive>;
};

/** Pagination details for list views. */
type Pagination = { page: number; limit: number };

/** Key-value filter applied to a list. */
type Filter = { key: string; value: string };

/** Sorting parameter applied to a list. */
type Sort = { key: string; value: "asc" | "dsc" };

/** Represents a payment. */
type Payment = {
  paymentId: string;
  gateway?: string;
  method: string;
  amount: Money;
  order?: Order;
};

/** Represents marketing campaign parameters (usually parsed from UTMs). */
type Campaign = {
  /** The referrer or entity sending the traffic (utm_source, e.g., "google", "newsletter"). */
  source: string;
  /** The marketing medium (utm_medium, e.g., "cpc", "email", "social"). */
  medium: string;
  /** The specific campaign name (utm_campaign, e.g., "summer_sale_2026"). */
  name?: string;
  /** Identify the paid keywords (utm_term). */
  term?: string;
  /** Differentiate similar content or links within the same ad (utm_content). */
  content?: string;
};
```

### 3. Predefined Standard Events

Events that deal with user identity and navigation.

#### Identify (`user_identified`)

Associates a user’s `anonymousId` (generated by the SDK/device) with their
actual database `userId` and updates their profile traits.

**When to trigger:** Immediately after a successful login, registration, or when
a user updates their profile settings.

```ts
type UserIdentifiedPayload = {
  /** The known database ID of the user. */
  userId: string;
  /** User profile attributes. */
  traits: {
    firstName?: string;
    lastName?: string;
    fullName?: string;
    username?: string;
    age?: number;
    email?: string;
    phone?: string;
    gender?: "male" | "female" | "other";
    /** UNIX timestamp in milliseconds. */
    birthday?: number;
    /** UNIX timestamp in milliseconds. */
    createdAt?: number;
    address?: {
      city?: string;
      country?: string;
      state?: string;
      street?: string;
      timezone?: string;
    };
  };
};
```

#### Screen (Web & Mobile) (`screened`)

Records that a user loaded a specific web page or mobile app screen.

**Nuance (`screen` vs. `product_viewed`):** Trigger `screen` for every page load
(e.g., “Home”, “About Us”, “Contact”). Trigger `product_viewed` in addition to
screen specifically when the page loaded is a Product Detail Page (PDP).

**Note:** `document.referrer` will return an empty string (`""`) in a few common
scenarios:

- The user typed the URL directly into the address bar or used a bookmark.
- The user navigated from a secure (`https`) site to a non-secure (`http`) site.
- The referring site has a strict `Referrer-Policy` (e.g., `no-referrer`) that
  blocks it for privacy reasons.

```typescript
type WebScreenedPayload = {
  /** The page title. */
  title: string;
  /** Full URL of the page. */
  url: string;
  /** URL pathname. */
  pathname?: string;
  /** Previous page URL. */
  referrer?: string;
  /** Query string parameters. */
  search?: string;
  /** Extracted SEO keywords. */
  keywords?: string[];
  /** Parsed UTM campaign parameters. */
  campaign?: Campaign;
  customProperties?: Record<string, Primitive>;
};

type MobileScreenedPayload = {
  /** Name of the screen (e.g., "Home", "Product Detail"). */
  title: string;
  /** The previous screen name. */
  referrer?: string;
  /** Parsed UTM campaign parameters from a deep link. */
  campaign?: Campaign;
  customProperties?: Record<string, Primitive>;
};
```

#### App State Changes (`app_state_changed`)

Fired when the application’s visibility or active lifecycle state changes. Tied
directly to OS or browser lifecycle hooks (e.g., `visibilitychange` in web,
`AppState` changes in React Native/iOS/Android).

**Use case:** Crucial for calculating true active session duration,
understanding user engagement/idle time, and tracking how often users background
the app and return.

```ts
type AppState = "opened" | "closed" | "foreground" | "background";

type AppStateChangedPayload = {
  newState: AppState;
  previousState?: AppState;
};
```

#### Click (`clicked`)

Fired when a user clicks, taps, or otherwise interacts with a clickable UI
element. Tied to standard DOM click events or mobile tap gesture handlers.

**Use case:** Essential for tracking feature usage, measuring conversion
funnels, understanding user navigation paths, and identifying dead clicks or UI
friction points.

```ts
type ClickedPayload = {
  elementId: string; // e.g., "submit_checkout_button"
  elementType?: string; // e.g., "button", "link", "icon"
  elementTitle?: string; // The title of the element, if applicable
  customProperties?: Record<string, Primitive>;
};
```

#### View (`viewed`)

Fired when a specific element becomes visible to the user.

**Use case:** Crucial for measuring impressions, mapping the user journey,
analyzing drop-off rates in funnels, and determining which content or features
get the most visibility.

```ts
type ViewedPayload = {
  elementId: string; // e.g., "services_section"
  elementType?: string;
  elementTitle?: string; // The title of the element, if applicable
  customProperties?: Record<string, Primitive>;
};
```

---

### 4. Predefined Ecommerce Events

Events specific to the shopping lifecycle.

#### 4.1 Product Interactions

Events related to users interacting with specific items outside of checkout.

##### `product_clicked`

Fired when a user clicks on a product card from a list, grid, or recommendation
carousel.

**Use case:** Measuring the click-through rate (CTR) of product thumbnails and
evaluating the effectiveness of product imagery and pricing displays.

```ts
type ProductClickedPayload = {
  product: Product;
  customProperties?: Record<string, Primitive>;
};
```

##### `product_viewed`

Fired when the user lands on the actual Product Detail Page (PDP) and sees the
full product description.

**Use case:** Tracking deep engagement and viewing history.

```ts
type ProductViewedPayload = {
  product: Product;
  customProperties?: Record<string, Primitive>;
};
```

##### `product_shared`

Fired when a user shares a product via a native share sheet, email, or social
media link.

**Use case:** Measures virality and identifies products that generate organic
interest.

```ts
type ProductSharedPayload = {
  /** Platform shared to (e.g., "Twitter", "Email", "WhatsApp"). */
  sharingMethod?: string;
  /** User-generated message attached to the share. */
  message?: string;
  /** Recipient identifier if shared directly. */
  recipient?: string;
  product: Product;
  customProperties?: Record<string, Primitive>;
};
```

#### 4.2 Cart & Wishlist Modifications

Events related to managing items saved for later or queued for purchase.

##### `product_added_to_wishlist` / `product_removed_from_wishlist`

Fired when a user toggles an item in their wishlist/favorites.

**Use case:** Wishlist additions signal high interest but low urgency. Useful
for triggering “Price Drop” or “Low Stock” email campaigns.

```ts
type ProductWishlistPayload = {
  wishlistId?: string;
  referrer?: string;
  product: Product;
  customProperties?: Record<string, Primitive>;
};
```

##### `product_added_to_cart` / `product_removed_from_cart`

Fired when a user modifies the contents of their shopping cart.

**Use case:** Adding to cart is the strongest signal of purchase intent.
Removing from cart often indicates price sensitivity, finding a better
alternative, or a change of mind.

```ts
type CartModificationPayload = {
  cartId?: string;
  referrer?: string;
  product: Product;
  customProperties?: Record<string, Primitive>;
};
```

##### `product_reviewed`

Fired when a user submits a rating or a written review for a product.

**Use case:** Used to measure product sentiment, track post-purchase engagement,
and analyze the correlation between high ratings and repeat purchases.

```ts
type ProductReviewedPayload = {
  product: Product;
  reviewId: string;
  rating: number;
  title?: string;
  body?: string;
  customProperties?: Record<string, Primitive>;
};
```

##### `cart_viewed`

Fired when the user opens the cart sidebar, modal, or navigates to the dedicated
cart page.

**Use case:** Shows users reviewing selected items. Comparing `cart_viewed`
against `checkout_started` can indicate hesitation, often based on the subtotal.

```ts
type CartViewedPayload = {
  cartId?: string;
  products: Product[];
  customProperties?: Record<string, Primitive>;
};
```

##### `cart_emptied`

Fired when the user explicitly clears their entire cart, or when the cart
expires.

**Use case:** Helps identify session abandonment or user frustration.

```ts
type CartEmptiedPayload = {
  cartId?: string;
  products: Product[];
  customProperties?: Record<string, Primitive>;
};
```

#### 4.3 Ordering & Checkout

Events tracking the progression of an order from initiation to fulfillment or
failure.

##### `checkout_started`

Fired when the user explicitly clicks “Proceed to Checkout” from the cart.

**Use case:** The critical entry point to the conversion funnel. If a user fires
`checkout_started` but not `order_completed`, there is friction in shipping, tax
calculation, or payment forms.

```ts
type CheckoutStartedPayload = {
  checkout: Checkout;
  customProperties?: Record<string, Primitive>;
};
```

##### `checkout_step_viewed` / `checkout_step_completed`

Fired as the user navigates through a multi-step checkout flow (e.g., Address ->
Shipping Method -> Payment).

**Use case:** Identifies exact abandonment points.

```ts
type CheckoutStepPayload = {
  checkout: Checkout;
  customProperties?: Record<string, Primitive>;
};
```

##### `order_completed`

Fired when the checkout is successfully submitted and payment is authorized.

**Use case:** The primary conversion and revenue metric.

```ts
type OrderCompletedPayload = {
  order: Order;
  customProperties?: Record<string, Primitive>;
};
```

##### `order_failed`

Fired when a payment is attempted but rejected (e.g., insufficient funds,
network timeout).

**Use case:** Monitoring payment gateway health and identifying user-facing
payment errors.

```ts
type OrderFailedPayload = {
  order: Order;
  /** Detailed reason for the failure. */
  reason: string;
  customProperties?: Record<string, Primitive>;
};
```

##### `order_cancelled`

Fired when an order is created but later aborted either by the customer or an
admin.

```ts
type OrderCancelledPayload = {
  order: Order;
  /** Who cancelled the order (e.g., "customer", "admin", "system"). */
  issuer?: string;
  customProperties?: Record<string, Primitive>;
};
```

##### `order_shipped`

Fired by the backend when the physical items leave the warehouse.

**Use case:** Measures operational efficiency, specifically the time delta from
`order_completed` to `order_shipped`.

```ts
type OrderShippedPayload = {
  order: Order;
  customProperties?: Record<string, Primitive>;
};
```

##### `order_product_fulfilled`

Fired when an individual item within an order is fulfilled (useful for split
shipments).

```ts
type OrderProductFulfilledPayload = {
  order: Order;
  product: Product;
  customProperties?: Record<string, Primitive>;
};
```

##### `order_refunded` / `order_product_returned`

Fired when an order is partially or fully refunded.

**Use case:** Crucial for calculating net metrics, e.g.,
`netRevenue = grossRevenue - returns`.

```ts
type OrderRefundedPayload = {
  order: Order;
  customProperties?: Record<string, Primitive>;
};

type OrderProductReturnedPayload = {
  order: Order;
  product: Product;
  /** Reason for return (e.g., "Defective", "Wrong Size"). */
  reason: string;
  /** How the customer was refunded (e.g., "Store Credit", "Original Payment Method"). */
  refundMethod?: string;
  totalReturnedValue: Money;
  customProperties?: Record<string, Primitive>;
};
```

##### `order_updated`

Fired when an existing order’s details are modified after the initial placement.

**Use case:** Helps track post-purchase modifications, measure customer service
intervention rates, and ensure downstream systems have the latest order state.

```ts
type OrderUpdatedPayload = {
  order: Order;
  reason?: string;
  customProperties?: Record<string, Primitive>;
};
```

##### `order_fulfillment_status_updated`

Fired when an order transitions from one state to another within the fulfillment
lifecycle. Trigger when the internal state of the order fulfillment changes
(e.g., moving from pending to processing, processing to shipped, or shipped to
delivered).

**Use case:** Used to measure fulfillment velocity, monitor operational
bottlenecks in the warehouse or logistics, and trigger transactional
updates/emails to the user.

```ts
type OrderFulfillmentStatusUpdatedPayload = {
  order: Order;
  previousStatus: string;
  newStatus: string;
  reason?: string;
  customProperties?: Record<string, Primitive>;
};
```

#### 4.4 Browsing & Discovery

Events related to users searching and browsing catalogs.

##### `products_searched`

Fired when a user submits a search query.

**Use case:** High search volume for queries that yield zero results indicates a
missed product opportunity or catalog gap.

```ts
type ProductsSearchedPayload = {
  query: string;
  customProperties?: Record<string, Primitive>;
};
```

##### `product_list_viewed`

Fired when a user views a list of products, such as a category page or search
results page.

```ts
type ProductListViewedPayload = {
  listId?: string;
  category?: Category;
  pagination?: Pagination;
  products: Product[];
  customProperties?: Record<string, Primitive>;
};
```

##### `product_list_filtered`

Fired when a user applies filters or sorting to a product list.

**Use case:** Measures how heavily users rely on faceted navigation and which
attributes (e.g., size, brand, price) are most important.

```ts
type ProductListFilteredPayload = {
  listId?: string;
  category?: Category;
  filters: Filter[];
  sorts: Sort[];
  products: Product[];
  customProperties?: Record<string, Primitive>;
};
```

#### 4.5 Coupons

Events tracking the usage and validity of discount codes.

##### `coupon_entered` / `coupon_removed`

Fired when a user attempts to apply or removes a coupon code during checkout.

**Use case:** Tracks intent to find a discount.

```ts
type CouponEnteredRemovedPayload = {
  coupon: Coupon;
  checkout: Checkout;
  customProperties?: Record<string, Primitive>;
};
```

##### `coupon_denied`

Fired when the system rejects an applied coupon code.

**Use case:** High rates of `coupon_denied` indicate frustrating promotional
rules, expired codes circulating online, or user confusion.

```ts
type CouponDeniedPayload = {
  coupon: Coupon;
  checkout: Checkout;
  reason: string;
  customProperties?: Record<string, Primitive>;
};
```

##### `coupon_redeemed`

Fired when a coupon is successfully applied to a completed order (usually
simultaneous with `order_completed`).

**Use case:** Proves marketing ROI for specific campaigns.

```ts
type CouponRedeemedPayload = {
  coupon: Coupon;
  order: Order;
  customProperties?: Record<string, Primitive>;
};
```

#### 4.6 Promotions

Events related to internal marketing banners and promotional real estate.

##### `promotion_viewed` / `promotion_clicked`

Fired when a user sees or clicks an internal site banner (e.g., “Summer Sale!
20% Off” on the homepage).

**Use case:** Calculating the CTR of internal marketing real estate to detect
“banner blindness” and measure if banners effectively drive traffic to target
categories.

```ts
type PromotionPayload = {
  promotionId: string;
  promotionTitle?: string;
  customProperties?: Record<string, Primitive>;
};
```

#### 4.7 Payment

Events related to payment.

##### `payment_authorized`

Fired when the payment gateway successfully authorizes a hold on the user’s
funds. Received via webhook or backend response when the payment processor
confirms the user has sufficient funds and has placed a temporary hold.

**Use case:** Differentiates between a user clicking “Pay” and the bank actually
approving the transaction.

```ts
type PaymentAuthorized = {
  payment: Payment;
  customProperties?: Record<string, Primitive>;
};
```

##### `payment_captured`

Fired when the payment gateway successfully processes the charge and transfers
funds. Triggered via backend webhook when the charge transitions from
“authorized” to “succeeded/captured”.

**Use case:** Represents actual cash flow. Used by finance and analytics to
verify RealizedRevenue. Comparing `payment_captured` against `order_completed`
helps reconcile internal database states with gateway reality.

```ts
type PaymentCaptured = {
  payment: Payment;
  customProperties?: Record<string, Primitive>;
};
```

##### `payment_failed`

Fired when a payment attempt is rejected by the gateway or the issuing bank.
Triggered immediately upon receiving an error response from the payment
processor during the checkout attempt.

**Use case:** Crucial for monitoring system health and user friction.
Engineering teams can monitor the
`GatewayErrorRate = payment_failed / (payment_authorized + payment_failed)` to
detect outages in third-party integrations. Product teams can use this to
trigger automated “cart recovery” or “try a different card” emails.

```ts
type PaymentFailed = {
  payment: Payment;
  customProperties?: Record<string, Primitive>;
};
```

##### `payment_refunded`

Fired when funds are returned to the user’s payment method. Triggered via
backend webhook when a refund successfully processes at the gateway level.

**Use case:** While `order_refunded` tracks the platform’s logical state,
`payment_refunded` tracks the actual financial movement. Essential for
calculating `NetCashFlow = payment_captured - payment_refunded` and monitoring
fraud or return rates by specific payment methods (e.g., “Do Apple Pay users
refund more often than Credit Card users?”).

```ts
type PaymentRefunded = {
  payment: Payment;
  reason?: string;
  returnedAmount: Money;
  customProperties?: Record<string, Primitive>;
};
```
