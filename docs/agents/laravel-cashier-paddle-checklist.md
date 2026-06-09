# Laravel Cashier Paddle Agent Checklist

Source baseline: Laravel 13.x Cashier Paddle docs.
Source URL: https://laravel.com/docs/13.x/cashier-paddle
Generated: 2026-06-08

Use this checklist when implementing, reviewing, or refactoring Paddle Billing features in a Laravel app.

## Preflight discovery

- [ ] Confirm `laravel/cashier-paddle` is installed in `composer.json`.
- [ ] Confirm this app uses Paddle Billing, not Paddle Classic.
- [ ] Locate the billable model or models, usually `User` and/or `Team`.
- [ ] Confirm the billable model uses `Laravel\Paddle\Billable`.
- [ ] Locate existing billing routes, controllers, services, policies, middleware, and event listeners.
- [ ] Locate existing price ID catalog/config before adding new price IDs.
- [ ] Confirm whether the app has single subscription, multiple subscriptions, multi-product subscriptions, one-time purchases, trials, or all of these.
- [ ] Confirm whether local domain records exist, such as `Order`, `Plan`, `License`, `Team`, or `Entitlement`.

## Installation/configuration checklist

- [ ] Package installed: `composer require laravel/cashier-paddle`.
- [ ] Cashier migrations published: `php artisan vendor:publish --tag="cashier-migrations"`.
- [ ] Migrations run: `php artisan migrate`.
- [ ] `.env.example` includes `PADDLE_CLIENT_SIDE_TOKEN`.
- [ ] `.env.example` includes `PADDLE_API_KEY`.
- [ ] `.env.example` includes `PADDLE_WEBHOOK_SECRET`.
- [ ] `.env.example` includes `PADDLE_SANDBOX`.
- [ ] `.env.example` includes `PADDLE_RETAIN_KEY` only if Retain is used.
- [ ] `.env.example` includes `CASHIER_CURRENCY_LOCALE` if money formatting locale is customized.
- [ ] `@paddleJS` appears once in the base layout before `</head>`.
- [ ] If a custom webhook URL is used, `CASHIER_WEBHOOK` is set and matches the Paddle dashboard.

## Paddle dashboard checklist

- [ ] Products are created in Paddle dashboard.
- [ ] Prices are fixed and active in Paddle dashboard.
- [ ] Default payment link is configured in Paddle checkout settings.
- [ ] Sandbox price IDs are not confused with production price IDs.
- [ ] Webhook URL points to `/paddle/webhook` unless intentionally overridden.
- [ ] Required events are enabled:
  - [ ] Customer Updated
  - [ ] Transaction Completed
  - [ ] Transaction Updated
  - [ ] Subscription Created
  - [ ] Subscription Updated
  - [ ] Subscription Paused
  - [ ] Subscription Canceled
- [ ] Production domain approval is complete before production launch.

## Security checklist

- [ ] No Paddle API keys or webhook secrets committed.
- [ ] `PADDLE_WEBHOOK_SECRET` is present outside local throwaway environments.
- [ ] `paddle/*` is excluded from CSRF in `bootstrap/app.php`.
- [ ] Webhook signature verification is not bypassed.
- [ ] Billing routes use auth middleware where required.
- [ ] User-controlled price IDs are validated against server-side allowlists.
- [ ] `customData` is treated as untrusted input and validated before use.
- [ ] Download invoice routes ensure the transaction belongs to the authenticated billable entity, unless route model binding/scoping already enforces this.

## Checkout implementation checklist

- [ ] Checkout session is created server-side with Cashier.
- [ ] One-time purchases use `$billable->checkout($priceIds)`.
- [ ] Subscriptions use `$billable->subscribe($priceId, $type)`.
- [ ] Guest checkout uses `Laravel\Paddle\Checkout::guest()` only when no account is needed.
- [ ] Checkout includes `returnTo()`.
- [ ] Checkout view uses `<x-paddle-button>` or `<x-paddle-checkout>` unless a custom frontend needs manual Paddle.js.
- [ ] For local orders, pending record is created before checkout.
- [ ] Local order/record ID is passed via `customData`.
- [ ] Success page handles pending state until webhook processing completes.
- [ ] Feature access is not granted solely because the user returned from checkout.

## Subscription implementation checklist

- [ ] Subscription type is stable, internal, has no spaces, and is not user-facing.
- [ ] Single subscription apps consistently use the same type, usually `default`.
- [ ] Multiple subscription apps use explicit types and never rely on implicit defaults accidentally.
- [ ] Access checks use Cashier methods such as `subscribed()`, `subscribedToPrice()`, `recurring()`, `onTrial()`, and `onGracePeriod()`.
- [ ] Middleware/policies distinguish trial/grace/recurring behavior according to product requirements.
- [ ] Plan swaps use `swap()`, `swapAndInvoice()`, `noProrate()`, or `doNotBill()` intentionally.
- [ ] Quantity changes use `incrementQuantity()`, `decrementQuantity()`, or `updateQuantity()` intentionally.
- [ ] Multi-product quantity changes pass the specific price ID when needed.
- [ ] Past-due subscriptions are not swapped or quantity-updated without payment remediation.
- [ ] Paused subscriptions are resumed before modifications.
- [ ] Cancelation behavior is explicit: `cancel()` vs `cancelNow()`.
- [ ] Grace-period logic is tested.
- [ ] Canceled subscriptions are not resumed; users create a new subscription.

## Trial checklist

- [ ] Payment-method-up-front trials are configured on the Paddle price.
- [ ] No-payment-method-up-front trials use `createAsCustomer(['trial_ends_at' => ...])`.
- [ ] Generic trial behavior is distinguished from subscription trial behavior.
- [ ] UI communicates trial end date.
- [ ] Users are notified before automatic billing when required by product/legal policy.
- [ ] Tests cover `onTrial()`, `hasExpiredTrial()`, `trialEndsAt()`, and `onGenericTrial()` where used.

## Webhook implementation checklist

- [ ] Application-specific state changes are handled through Cashier/Paddle events.
- [ ] Listener handles duplicate delivery idempotently.
- [ ] Listener validates local records referenced by `customData`.
- [ ] Listener logs enough context to diagnose webhook failures without logging secrets.
- [ ] Listener safely handles missing/unknown payload fields.
- [ ] Listener is queued if work is slow or external side effects are involved.
- [ ] Tests simulate relevant webhook event payloads.
- [ ] Local/staging webhook testing uses ngrok, Expose, or Sail site sharing.

## Transaction/invoice checklist

- [ ] Transaction lists use `$user->transactions` or scoped relations.
- [ ] Transaction display uses Cashier helpers such as `total()` and `tax()`.
- [ ] Invoice download uses `redirectToInvoicePdf()`.
- [ ] Refund flows explain Paddle approval behavior.
- [ ] Credit flows are limited to manually collected transactions.
- [ ] Subscription credit expectations are delegated to Paddle.
- [ ] `lastPayment()` and `nextPayment()` nullable cases are handled.

## Testing checklist

- [ ] Manual sandbox checkout tested.
- [ ] Manual sandbox subscription creation tested.
- [ ] Manual sandbox cancelation tested.
- [ ] Manual sandbox webhook delivery tested.
- [ ] Automated tests fake Paddle HTTP calls using Laravel HTTP client fakes.
- [ ] Automated tests cover pending checkout return state.
- [ ] Automated tests cover successful webhook completion.
- [ ] Automated tests cover duplicate webhook delivery.
- [ ] Automated tests cover authorization for invoice download and billing management routes.
- [ ] Automated tests cover subscription gates/middleware.
- [ ] Automated tests cover plan swaps and proration behavior where business logic depends on it.

## Review questions for agents

Ask these before finalizing billing changes:

1. What is the source of truth for this state: local domain record, Cashier table, or Paddle webhook?
2. What happens if the user closes checkout or returns before webhook delivery?
3. What happens if Paddle sends the webhook twice?
4. What happens if Paddle sends webhook events out of order?
5. Are sandbox and live price IDs separated safely?
6. Are all price IDs server-side allowlisted?
7. Can a user access or download another user's invoice?
8. Does the app intentionally treat trials/grace periods as entitled access?
9. Are subscriptions modified while paused or past due?
10. Are secrets and webhook payloads logged safely?

## Suggested file organization

Use the app's existing conventions first. If none exist, prefer this style:

```text
app/
  Billing/
    PaddlePrices.php
    StartsPaddleCheckout.php
    SubscriptionEntitlements.php
  Listeners/
    CompleteOrderFromPaddle.php
    SyncEntitlementsFromPaddle.php
  Http/
    Controllers/
      BillingController.php
      SubscriptionController.php
    Middleware/
      EnsureUserIsSubscribed.php
config/
  billing.php
routes/
  billing.php or web.php
tests/
  Feature/
    Billing/
      PaddleCheckoutTest.php
      PaddleWebhookTest.php
      SubscriptionAccessTest.php
```

## Red flags

- [ ] Controller accepts arbitrary `price_id` from request and passes it directly to Cashier.
- [ ] Checkout success route sets `paid=true` or grants entitlements without webhook confirmation.
- [ ] Webhook listener is not idempotent.
- [ ] Webhook signature verification is disabled.
- [ ] `paddle/*` CSRF exception is missing.
- [ ] Price IDs are scattered through Blade templates.
- [ ] `PADDLE_SANDBOX` is true in production or false in local/staging by accident.
- [ ] Same database contains mixed sandbox/live Paddle IDs.
- [ ] App assumes `lastPayment()` is always non-null.
- [ ] App assumes a canceled subscription can be resumed.
