---
name: laravel-cashier-paddle-development
description: Build, review, and test Laravel Cashier Paddle Billing features, including checkouts, subscriptions, webhooks, transactions, invoices, trials, and paid-feature access.
---

# Laravel Cashier Paddle Development

## When to use this skill

Use this skill whenever work touches Laravel billing code backed by Paddle Billing, including:

- installing or configuring `laravel/cashier-paddle`
- creating Paddle checkout sessions
- starting, swapping, pausing, canceling, or resuming subscriptions
- handling Paddle webhooks or Cashier Paddle events
- syncing orders, licenses, teams, entitlements, invoices, transactions, refunds, or credits
- protecting paid features with middleware, gates, policies, or tests

Source baseline: Laravel 13.x Cashier Paddle docs, Cashier Paddle 2.x, Paddle Billing.
Source URL: https://laravel.com/docs/13.x/cashier-paddle
Generated: 2026-06-08

## Non-negotiable rules

1. Use `laravel/cashier-paddle` for Paddle Billing. Do not use Paddle Classic APIs or assumptions.
2. Treat Paddle webhooks as the billing source of truth. A checkout return is not proof of payment or subscription state.
3. Never commit Paddle secrets. Read secrets from environment variables and configuration.
4. Do not scatter live price IDs through controllers, views, or tests. Centralize them in config, constants, enum-like value objects, or a billing catalog service.
5. Subscription `type` values are internal stable identifiers such as `default`, `primary`, `team`, or `swimming`. Do not display them to users or rename them after subscriptions exist.
6. Handle delayed webhook arrival. Checkout return pages should show pending or processing state until local Cashier/domain records are synced.
7. Make webhook listeners idempotent. Paddle may retry events; use Paddle IDs, transaction IDs, subscription IDs, order IDs, and database constraints to prevent duplicate side effects.
8. Verify webhook signatures with `PADDLE_WEBHOOK_SECRET`. Exclude Paddle webhook routes from CSRF, but do not bypass signature verification.
9. Do not modify paused subscriptions. Resume first, then swap plans or update quantities.
10. Do not swap plans or update quantity for `past_due` subscriptions unless payment info is updated or the application intentionally keeps past-due subscriptions active.

## Preflight workflow

Before editing billing code:

1. Inspect `composer.json` for `laravel/cashier-paddle` and note the installed version.
2. Inspect `config/cashier.php`, `.env.example`, `bootstrap/app.php`, migrations, and billable models.
3. Confirm the billable model, usually `App\Models\User` or `Team`, uses `Laravel\Paddle\Billable`.
4. Search for an existing price ID catalog/config and reuse it.
5. Search existing routes, controllers, services, policies, middleware, and event listeners for billing conventions.
6. Search for Cashier event listeners before adding new listeners.
7. Prefer small explicit services such as `BillingCatalog`, `StartsPaddleCheckout`, `CompletesOrderFromPaddle`, and `SubscriptionEntitlements` over large controller methods.

After editing billing code:

1. Add or update focused tests.
2. Run the relevant PHP tests.
3. Run Laravel Pint if the project uses it.
4. Ensure new environment variables are documented in `.env.example`.
5. Confirm no secret values, live API keys, webhook secrets, or raw sensitive payloads were committed.

## Required setup

Install and publish migrations:

```bash
composer require laravel/cashier-paddle
php artisan vendor:publish --tag="cashier-migrations"
php artisan migrate
```

Cashier creates tables for customers, subscriptions, subscription items, and transactions. Only completed transactions are stored locally. Do not invent duplicate billing tables unless the app needs separate domain records such as `orders`, `licenses`, `teams`, or `entitlements`.

Add `Laravel\Paddle\Billable` to every billable model:

```php
use Laravel\Paddle\Billable;

class User extends Authenticatable
{
    use Billable;
}
```

Expected environment values:

```dotenv
PADDLE_CLIENT_SIDE_TOKEN=
PADDLE_API_KEY=
PADDLE_WEBHOOK_SECRET=
PADDLE_SANDBOX=true
# Optional, only when using Paddle Retain:
PADDLE_RETAIN_KEY=
# Optional, only if overriding the default webhook URL:
CASHIER_WEBHOOK=https://example.com/my-paddle-webhook-url
# Optional currency locale for invoice/money formatting:
CASHIER_CURRENCY_LOCALE=en_US
```

For production, set `PADDLE_SANDBOX=false` and confirm Paddle has approved the production domain.

Load Paddle.js once in the base Blade layout before `</head>`:

```blade
@paddleJS
```

## Price IDs and catalog

Paddle price IDs usually start with `pri_`. Never accept arbitrary user-controlled price IDs and pass them directly to Cashier.

For small apps, a dedicated class is acceptable:

```php
final class PaddlePrices
{
    public const BASIC_MONTHLY = 'pri_...';
    public const BASIC_YEARLY = 'pri_...';
}
```

For larger apps or apps with sandbox/live differences, prefer `config/billing.php` backed by environment variables and expose prices through a `BillingCatalog` service.

## Checkout rules

Use billable model methods for authenticated users:

```php
$checkout = $request->user()
    ->checkout('pri_example')
    ->returnTo(route('billing.status'));
```

Use `subscribe($priceId, $type = 'default')` for subscriptions:

```php
$checkout = $request->user()
    ->subscribe('pri_monthly', 'default')
    ->customData(['order_id' => $order->id])
    ->returnTo(route('billing.status'));
```

Render overlay checkout with:

```blade
<x-paddle-button :checkout="$checkout">Subscribe</x-paddle-button>
```

Render inline checkout with:

```blade
<x-paddle-checkout :checkout="$checkout" class="w-full" height="500" />
```

Use guest checkout only when no app account is required:

```php
use Laravel\Paddle\Checkout;

$checkout = Checkout::guest(['pri_example'])
    ->returnTo(route('home'));
```

## Webhook-first state management

Checkout is asynchronous. A user can return before Paddle has delivered and Cashier has processed the webhook.

Recommended pattern:

1. Create a pending local domain record before checkout, such as `Order(status: incomplete)`.
2. Pass the local record ID through `customData`.
3. On `TransactionCompleted` or the relevant subscription event, validate payload data and update the local record.
4. Keep the UI in a pending/processing state until local records confirm the purchase or subscription.

```php
use Laravel\Paddle\Events\TransactionCompleted;

final class CompleteOrderFromPaddle
{
    public function handle(TransactionCompleted $event): void
    {
        $orderId = data_get($event->payload, 'data.custom_data.order_id');

        if (! $orderId) {
            return;
        }

        $order = Order::query()->find($orderId);

        if (! $order || $order->status === 'completed') {
            return;
        }

        $order->markCompletedFromPaddle($event->payload);
    }
}
```

When using checkout `customData`, treat it as untrusted input. Validate that referenced local records exist and belong to the expected billable entity.

## Webhook endpoint and events

Cashier registers the default webhook endpoint at `/paddle/webhook`.

Enable these Paddle dashboard events:

- Customer Updated
- Transaction Completed
- Transaction Updated
- Subscription Created
- Subscription Updated
- Subscription Paused
- Subscription Canceled

Exclude Paddle webhooks from CSRF in `bootstrap/app.php`:

```php
->withMiddleware(function (Middleware $middleware): void {
    $middleware->preventRequestForgery(except: [
        'paddle/*',
    ]);
})
```

Use Cashier events for application-specific work:

- `Laravel\Paddle\Events\WebhookReceived`
- `Laravel\Paddle\Events\WebhookHandled`
- `Laravel\Paddle\Events\CustomerUpdated`
- `Laravel\Paddle\Events\TransactionCompleted`
- `Laravel\Paddle\Events\TransactionUpdated`
- `Laravel\Paddle\Events\SubscriptionCreated`
- `Laravel\Paddle\Events\SubscriptionUpdated`
- `Laravel\Paddle\Events\SubscriptionPaused`
- `Laravel\Paddle\Events\SubscriptionCanceled`

## Subscriptions and entitlements

Common checks:

```php
$user->subscribed();
$user->subscribed('default');
$user->subscribedToPrice('pri_monthly', 'default');
$user->subscription()->onTrial();
$user->subscription()->recurring();
$user->subscription()->canceled();
$user->subscription()->onGracePeriod();
$user->subscription()->onPausedGracePeriod();
```

Use middleware, gates, policies, or explicit entitlement services for paid feature access. Prefer Cashier methods over direct raw-column checks.

Plan changes:

```php
$user->subscription('default')->swap('pri_yearly');
$user->subscription('default')->noProrate()->swap('pri_yearly');
$user->subscription('default')->swapAndInvoice('pri_yearly');
$user->subscription('default')->doNotBill()->swap('pri_yearly');
```

Quantities:

```php
$user->subscription()->incrementQuantity();
$user->subscription()->incrementQuantity(5);
$user->subscription()->decrementQuantity();
$user->subscription()->updateQuantity(10);
$user->subscription()->noProrate()->updateQuantity(10);
```

Multi-product subscriptions:

```php
$checkout = $request->user()->subscribe([
    'pri_base_monthly',
    'pri_live_chat' => 5,
], 'default');
```

Do not remove the final price from a subscription; cancel the subscription instead.

Cancellation and pause behavior:

```php
$user->subscription()->cancel();
$user->subscription()->cancelNow();
$user->subscription()->stopCancelation();
$user->subscription()->pause();
$user->subscription()->pauseNow();
$user->subscription()->pauseUntil(now()->plusMonth());
$user->subscription()->resume();
```

Paddle subscriptions cannot be resumed after cancelation. The customer must create a new subscription.

## Trials

For payment-method-up-front trials, configure the trial on the Paddle price and create checkout normally.

For no-payment-method-up-front trials, create a Paddle customer with `trial_ends_at`:

```php
$user->createAsCustomer([
    'trial_ends_at' => now()->plusDays(10),
]);
```

Trial helpers:

```php
$user->onTrial();
$user->onTrial('default');
$user->hasExpiredTrial();
$user->trialEndsAt();
$user->onGenericTrial();
$user->subscription()->extendTrial(now()->plusDays(5));
$user->subscription()->activate();
```

Notify customers before a trial ends if they will be charged automatically.

## Transactions, invoices, refunds, and credits

Use `$user->transactions` to list completed transactions.

Use transaction helpers for display:

```php
$transaction->total();
$transaction->tax();
$transaction->redirectToInvoicePdf();
```

Authorize invoice routes so users cannot access another billable entity's transactions.

Refunds may be full or line-item specific and must be approved by Paddle before fully processing. Credits only apply to manually collected transactions; automatically collected subscription credits are handled by Paddle.

## Testing expectations

Manual tests:

- full sandbox billing flow with Paddle sandbox cards
- webhook delivery through a tunnel such as ngrok, Expose, or Sail site sharing
- subscription creation, cancelation, pause/resume, and checkout return states

Automated tests:

- fake outbound Paddle HTTP calls using Laravel's HTTP client fakes
- delayed webhook behavior: checkout success should not immediately grant entitlements
- successful webhook completion
- duplicate webhook delivery and listener idempotency
- canceled, grace-period, paused, trial, expired-trial, and past-due states
- authorization for invoice download and billing management routes

## Review questions

Ask these before finalizing billing changes:

1. What is the source of truth for this state: local domain record, Cashier table, or Paddle webhook?
2. What happens if the user closes checkout or returns before webhook delivery?
3. What happens if Paddle sends the webhook twice?
4. What happens if Paddle sends webhook events out of order?
5. Are sandbox and live price IDs separated safely?
6. Are all price IDs server-side allowlisted?
7. Can a user access or download another user's invoice?
8. Does the app intentionally treat trials or grace periods as entitled access?
9. Are subscriptions modified while paused or past due?
10. Are secrets and webhook payloads logged safely?
