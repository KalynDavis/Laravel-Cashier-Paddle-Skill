# AGENTS.md — Laravel Cashier Paddle Billing Rules

Source baseline: Laravel 13.x Cashier Paddle docs, Cashier Paddle 2.x, Paddle Billing.
Source URL: https://laravel.com/docs/13.x/cashier-paddle
Generated: 2026-06-08

These instructions apply whenever an AI coding agent touches billing, subscriptions, Paddle checkouts, Paddle webhooks, transactions, invoices, access control, or tests around paid features.

## Non-negotiable rules

1. Use `laravel/cashier-paddle` for Paddle Billing. Do not use Paddle Classic APIs or assumptions.
2. Treat Paddle webhooks as the billing source of truth. Do not mark subscriptions, orders, or entitlements complete only because a user returned from checkout.
3. Never commit Paddle secrets. Read secrets from environment variables and configuration.
4. Never hard-code live price IDs in arbitrary controllers or views. Put price IDs behind config, constants, enum-like value objects, or a dedicated billing catalog layer.
5. Subscription `type` values are internal identifiers. Use stable strings such as `default`, `primary`, `team`, `swimming`, etc. Do not display them to users and do not rename them after subscriptions exist.
6. Handle delayed webhook arrival. UI after checkout should show a pending/processing state if local subscription or transaction records are not yet synced.
7. Make webhook listeners idempotent. Paddle may retry events. Use Paddle IDs, transaction IDs, subscription IDs, order IDs, and database constraints to avoid duplicate work.
8. Verify webhook signatures with `PADDLE_WEBHOOK_SECRET`. Exclude Paddle webhook routes from CSRF, but do not skip signature verification.
9. Do not modify paused subscriptions. Resume first, then swap plans or update quantities.
10. Do not swap or update quantity for `past_due` subscriptions unless payment info is updated or the application intentionally keeps past-due subscriptions active.

## Required setup

Install and publish migrations:

```bash
composer require laravel/cashier-paddle
php artisan vendor:publish --tag="cashier-migrations"
php artisan migrate
```

Cashier creates tables for customers, subscriptions, subscription items, and transactions. Only completed transactions are stored locally.

Add `Laravel\Paddle\Billable` to the billable model, usually `App\Models\User`:

```php
use Laravel\Paddle\Billable;

class User extends Authenticatable
{
    use Billable;
}
```

Required/expected environment variables:

```dotenv
PADDLE_CLIENT_SIDE_TOKEN=
PADDLE_API_KEY=
PADDLE_WEBHOOK_SECRET=
PADDLE_SANDBOX=true
# Optional, only when using Paddle Retain:
PADDLE_RETAIN_KEY=
# Optional, only if overriding default webhook URL:
CASHIER_WEBHOOK=https://example.com/my-paddle-webhook-url
# Optional currency locale for invoice/money formatting:
CASHIER_CURRENCY_LOCALE=en_US
```

For production, set `PADDLE_SANDBOX=false` and confirm Paddle has approved the production domain.

Include Paddle.js once in the base Blade layout before `</head>`:

```blade
@paddleJS
```

## Webhooks

Cashier registers a default webhook endpoint at `/paddle/webhook`.

Enable these Paddle events in the Paddle control panel:

- Customer Updated
- Transaction Completed
- Transaction Updated
- Subscription Created
- Subscription Updated
- Subscription Paused
- Subscription Canceled

In `bootstrap/app.php`, exclude Paddle webhooks from CSRF protection:

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

When using checkout `customData`, read it from the event payload and validate that referenced local records exist and belong to the expected billable entity.

## Checkout rules

Use billable model methods for authenticated users:

```php
$checkout = $request->user()
    ->checkout('pri_example')
    ->returnTo(route('dashboard'));
```

Use `subscribe($priceId, $type = 'default')` for subscriptions:

```php
$checkout = $request->user()
    ->subscribe('pri_monthly', 'default')
    ->customData(['plan' => 'monthly'])
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

## Subscriptions and entitlements

Common state checks:

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

Use middleware or policies for paid feature access. Prefer checking subscription state through Cashier methods instead of directly reading raw columns.

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
$user->subscription()->cancel();      // ends after billing period / grace period
$user->subscription()->cancelNow();   // immediate
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

## Transactions, refunds, credits, and invoices

Use `$user->transactions` to list completed transactions.

Use transaction helpers for display:

```php
$transaction->total();
$transaction->tax();
$transaction->redirectToInvoicePdf();
```

Refunds may be full or line-item specific and must be approved by Paddle before fully processing.

Credits only apply to manually collected transactions. Automatically collected subscription credits are handled by Paddle.

## Testing expectations

1. Manually test the full sandbox billing flow with Paddle sandbox cards.
2. Test webhook paths locally through a tunnel such as ngrok, Expose, or Sail site sharing.
3. In automated tests, fake outbound Paddle HTTP calls using Laravel's HTTP client fakes.
4. Test delayed webhook behavior: checkout success should not immediately grant entitlements unless the relevant webhook has updated local records.
5. Test duplicate webhook delivery and listener idempotency.
6. Test canceled, grace-period, paused, trial, expired-trial, and past-due states.

## Agent workflow

Before editing billing code:

1. Inspect `composer.json` for `laravel/cashier-paddle` version.
2. Inspect `config/cashier.php`, `.env.example`, `bootstrap/app.php`, migrations, and billable models.
3. Search for existing price ID catalog/config and reuse it.
4. Search event listeners for Cashier/Paddle events before adding new ones.
5. Prefer small, explicit services such as `BillingCatalog`, `StartsPaddleCheckout`, `CompletesOrderFromPaddle`, and `SubscriptionEntitlements` over large controller methods.

After editing billing code:

1. Add or update tests.
2. Run the relevant PHP tests.
3. Run Laravel Pint if the project uses it.
4. Ensure new environment variables are documented in `.env.example`.
5. Confirm no secret values, live API keys, or webhook secrets were committed.
