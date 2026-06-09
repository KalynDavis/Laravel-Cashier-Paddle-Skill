# Laravel Cashier Paddle 13.x Agent Reference

Source baseline: Laravel 13.x Cashier Paddle documentation for Cashier Paddle 2.x and Paddle Billing.
Source URL: https://laravel.com/docs/13.x/cashier-paddle
Generated: 2026-06-08

This file is a condensed implementation reference for agents building or reviewing Laravel Paddle billing features.

## 1. Package and platform boundaries

- Cashier Paddle 2.x integrates with Paddle Billing.
- Do not use Paddle Classic package/API patterns.
- Paddle dashboard setup is part of the implementation: products, prices, default payment link, sandbox/live environment, webhook URL, webhook events, and production domain approval.
- For local/staging, use Paddle Sandbox and set `PADDLE_SANDBOX=true`.
- For production, use live Paddle credentials and set `PADDLE_SANDBOX=false`.

## 2. Installation and database

Install:

```bash
composer require laravel/cashier-paddle
php artisan vendor:publish --tag="cashier-migrations"
php artisan migrate
```

Cashier migrations create/maintain local billing tables for:

- customers
- subscriptions
- subscription items
- transactions

Do not invent custom duplicate tables for these concepts unless the app needs domain-specific records such as `orders`, `licenses`, `teams`, or `entitlements`.

## 3. Billable models

Use the `Billable` trait on every model that owns billing state.

```php
use Laravel\Paddle\Billable;

class User extends Authenticatable
{
    use Billable;
}
```

A non-user billable model, such as `Team`, may also use `Billable`.

Customer defaults can be supplied on the billable model:

```php
public function paddleName(): ?string
{
    return $this->name;
}

public function paddleEmail(): ?string
{
    return $this->email;
}
```

Create Paddle customers without starting checkout when needed:

```php
$customer = $user->createAsCustomer();
$customer = $user->createAsCustomer($options);
```

Find a billable model by Paddle customer ID:

```php
use Laravel\Paddle\Cashier;

$user = Cashier::findBillable($customerId);
```

## 4. Configuration and assets

Expected `.env` values:

```dotenv
PADDLE_CLIENT_SIDE_TOKEN=
PADDLE_API_KEY=
PADDLE_WEBHOOK_SECRET=
PADDLE_SANDBOX=true
PADDLE_RETAIN_KEY=
CASHIER_CURRENCY_LOCALE=en_US
CASHIER_WEBHOOK=
```

`PADDLE_RETAIN_KEY` is only required if using Paddle Retain.

If using non-English money formatting locales, verify the PHP `ext-intl` extension is installed.

In the main layout, load Paddle.js once before `</head>`:

```blade
@paddleJS
```

## 5. Price IDs and billing catalog

Use Paddle price IDs for checkout and subscription APIs. A price ID usually begins with `pri_`.

Recommended local structure:

```php
final class PaddlePrices
{
    public const BASIC_MONTHLY = 'pri_...';
    public const BASIC_YEARLY = 'pri_...';
    public const EXPERT_MONTHLY = 'pri_...';
}
```

Better for larger applications: read price IDs from `config/billing.php`, backed by environment variables for sandbox/live differences.

Avoid storing price IDs in many controllers/views. Keep all pricing identifiers centralized.

## 6. Checkout sessions

Authenticated single-charge checkout:

```php
$checkout = $request->user()
    ->checkout('pri_deluxe_album')
    ->returnTo(route('billing.status'));
```

Multiple items and quantities:

```php
$checkout = $request->user()
    ->checkout([
        'pri_tshirt',
        'pri_socks' => 5,
    ])
    ->returnTo(route('billing.status'));
```

Subscription checkout:

```php
$checkout = $request->user()
    ->subscribe('pri_basic_monthly', 'default')
    ->returnTo(route('billing.status'));
```

Subscription checkout with metadata:

```php
$checkout = $request->user()
    ->subscribe('pri_basic_monthly', 'default')
    ->customData([
        'order_id' => $order->id,
    ])
    ->returnTo(route('billing.status'));
```

Guest checkout:

```php
use Laravel\Paddle\Checkout;

$checkout = Checkout::guest(['pri_single_item'])
    ->returnTo(route('home'));
```

Overlay checkout:

```blade
<x-paddle-button :checkout="$checkout" class="px-8 py-4">
    Subscribe
</x-paddle-button>
```

Inline checkout:

```blade
<x-paddle-checkout :checkout="$checkout" class="w-full" height="500" />
```

Manual checkout rendering is possible through `$checkout->options()` and Paddle.js, but prefer Cashier Blade components unless the project needs custom frontend behavior.

## 7. Webhook-first state management

Checkout is asynchronous. A user can return to the app before Paddle has delivered and Cashier has processed the webhook.

Implementation pattern:

1. Create local pending domain record before checkout, such as `Order(status: incomplete)`.
2. Pass local ID through `customData`.
3. On `TransactionCompleted` or relevant subscription event, validate payload and update the local record.
4. UI uses pending/processing state until local records confirm the purchase/subscription.

Example listener outline:

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

Register listeners in an application service provider or event service provider, following the app's existing convention.

## 8. Webhook endpoint and security

Default endpoint: `/paddle/webhook`.

Enable these Paddle dashboard events:

- Customer Updated
- Transaction Completed
- Transaction Updated
- Subscription Created
- Subscription Updated
- Subscription Paused
- Subscription Canceled

CSRF exclusion in `bootstrap/app.php`:

```php
->withMiddleware(function (Middleware $middleware): void {
    $middleware->preventRequestForgery(except: [
        'paddle/*',
    ]);
})
```

Keep `PADDLE_WEBHOOK_SECRET` configured so Cashier's signature-verification middleware can reject invalid webhook calls.

Optional custom webhook URL:

```dotenv
CASHIER_WEBHOOK=https://example.com/my-paddle-webhook-url
```

If overriding, ensure the full URL exactly matches the Paddle dashboard URL.

## 9. Cashier webhook events

Generic events:

- `WebhookReceived`: full raw payload when received.
- `WebhookHandled`: full raw payload after handled.

Dedicated events:

- `CustomerUpdated`
- `TransactionCompleted`
- `TransactionUpdated`
- `SubscriptionCreated`
- `SubscriptionUpdated`
- `SubscriptionPaused`
- `SubscriptionCanceled`

Use generic events when the Paddle event type is not represented by a dedicated Cashier event. Use dedicated events when you need the relevant Cashier models and a typed event class.

## 10. Price previews and discounts

Preview public prices:

```php
use Laravel\Paddle\Cashier;

$prices = Cashier::previewPrices(['pri_basic', 'pri_expert']);
```

Preview for a country/postal code:

```php
$prices = Cashier::previewPrices(['pri_basic'], [
    'address' => [
        'country_code' => 'BE',
        'postal_code' => '1234',
    ],
]);
```

Preview customer-specific prices:

```php
$prices = $user->previewPrices(['pri_basic', 'pri_expert']);
```

Preview discounted prices:

```php
$prices = Cashier::previewPrices(['pri_basic'], [
    'discount_id' => 'dsc_123',
]);
```

Display through Cashier price helpers:

```blade
{{ $price->total() }}
{{ $price->subtotal() }}
{{ $price->tax() }}
```

## 11. Subscription creation and types

Create subscription checkout:

```php
$checkout = $request->user()
    ->subscribe('pri_premium_monthly', 'default')
    ->returnTo(route('billing.status'));
```

Rules:

- The first argument is the Paddle price ID.
- The second argument is the internal subscription type.
- If the app has one subscription, use `default` unless an existing convention says otherwise.
- Do not include spaces in subscription type names.
- Do not rename types after subscriptions exist.

## 12. Subscription status checks

Common methods:

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

Interpretation:

- `subscribed()` may remain true during trial/grace period.
- `recurring()` means active and no longer in trial/grace period.
- `canceled()` indicates a previously active subscription has been canceled.
- `onGracePeriod()` means canceled but still usable until the effective end date.
- `onPausedGracePeriod()` means paused but still usable during the paid-through period.

Available query scopes include:

```php
Subscription::query()->valid();
Subscription::query()->onTrial();
Subscription::query()->expiredTrial();
Subscription::query()->notOnTrial();
Subscription::query()->active();
Subscription::query()->recurring();
Subscription::query()->pastDue();
Subscription::query()->paused();
Subscription::query()->notPaused();
Subscription::query()->onPausedGracePeriod();
Subscription::query()->notOnPausedGracePeriod();
Subscription::query()->canceled();
Subscription::query()->notCanceled();
Subscription::query()->onGracePeriod();
Subscription::query()->notOnGracePeriod();
```

## 13. Access control

Use middleware, policies, gates, or service methods to protect paid features. Example middleware shape:

```php
public function handle(Request $request, Closure $next): Response
{
    if (! $request->user()?->subscribed()) {
        return redirect()->route('billing.index');
    }

    return $next($request);
}
```

For feature-tier checks, prefer explicit methods over scattered price ID comparisons:

```php
$user->subscribedToPrice(PaddlePrices::EXPERT_MONTHLY, 'default');
```

## 14. Payment method updates

Paddle stores a payment method per subscription. Redirect users to Paddle's hosted update page:

```php
return $request->user()
    ->subscription('default')
    ->redirectToUpdatePaymentMethod();
```

A `subscription_updated` webhook will update local records after completion.

## 15. Plan changes and prorations

Basic swap:

```php
$user->subscription('default')->swap('pri_premium_yearly');
```

Cashier/Paddle prorates by default.

No proration:

```php
$user->subscription('default')->noProrate()->swap('pri_premium_yearly');
```

Invoice immediately:

```php
$user->subscription('default')->swapAndInvoice('pri_premium_yearly');
```

No proration + immediate invoice:

```php
$user->subscription('default')->noProrate()->swapAndInvoice('pri_premium_yearly');
```

Do not bill for subscription change:

```php
$user->subscription('default')->doNotBill()->swap('pri_premium_yearly');
```

Guardrails:

- `past_due` subscriptions cannot be changed until payment information is updated, unless the app intentionally configured past-due subscriptions to stay active.
- Paused subscriptions cannot be modified; resume first.

## 16. Quantities

Increment/decrement:

```php
$user->subscription()->incrementQuantity();
$user->subscription()->incrementQuantity(5);
$user->subscription()->decrementQuantity();
$user->subscription()->decrementQuantity(5);
```

Set quantity:

```php
$user->subscription()->updateQuantity(10);
```

No proration:

```php
$user->subscription()->noProrate()->updateQuantity(10);
```

For multi-product subscriptions, pass the price ID as the second argument:

```php
$user->subscription()->incrementQuantity(1, 'pri_live_chat');
```

## 17. Multi-product subscriptions

Create subscription with multiple prices:

```php
$checkout = $request->user()->subscribe([
    'pri_base_monthly',
    'pri_live_chat',
], 'default');
```

With quantities:

```php
$checkout = $request->user()->subscribe([
    'pri_base_monthly',
    'pri_live_chat' => 5,
], 'default');
```

Swap to add/remove products:

```php
$user->subscription()->swap([
    'pri_base_monthly' => 2,
    'pri_live_chat',
]);
```

To remove a price, omit it from the swap list. Do not remove the final remaining price; cancel the subscription instead.

## 18. Multiple subscriptions

Use a stable type for each parallel subscription:

```php
$checkout = $request->user()
    ->subscribe('pri_swimming_monthly', 'swimming');

$user->subscription('swimming')->swap('pri_swimming_yearly');
$user->subscription('swimming')->cancel();
```

## 19. Pausing and canceling

Pause:

```php
$user->subscription()->pause();
$user->subscription()->pauseNow();
$user->subscription()->pauseUntil(now()->plusMonth());
$user->subscription()->pauseNowUntil(now()->plusMonth());
$user->subscription()->resume();
```

Cancel:

```php
$user->subscription()->cancel();
$user->subscription()->cancelNow();
$user->subscription()->stopCancelation();
```

Behavior:

- `cancel()` keeps access through the paid billing period.
- `cancelNow()` ends immediately.
- `stopCancelation()` stops a scheduled cancelation during grace period.
- Paddle subscriptions cannot be resumed after cancelation; users must subscribe again.

## 20. Trials

Payment method up front:

- Configure trial duration on the Paddle price.
- Create checkout normally.
- Notify users before automatic billing starts.

No payment method up front:

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
$user->hasExpiredTrial('default');
$user->trialEndsAt();
$user->onGenericTrial();
$user->subscription()->extendTrial(now()->plusDays(5));
$user->subscription()->activate();
```

Generic trials are customer-level trials not attached to an existing subscription.

## 21. Subscription single charges

Charge on next billing interval:

```php
$response = $user->subscription()->charge('pri_addon');
$response = $user->subscription()->charge(['pri_addon_a', 'pri_addon_b']);
```

Charge and invoice immediately:

```php
$response = $user->subscription()->chargeAndInvoice('pri_addon');
```

## 22. Transactions, refunds, credits, invoices

List completed transactions:

```php
$transactions = $user->transactions;
```

Invoice PDF redirect:

```php
Route::get('/download-invoice/{transaction}', function (Transaction $transaction) {
    return $transaction->redirectToInvoicePdf();
})->name('download-invoice');
```

Past/upcoming payment helpers:

```php
$subscription = $user->subscription();
$lastPayment = $subscription->lastPayment();
$nextPayment = $subscription->nextPayment();
```

`lastPayment()` can be `null` before webhook sync. `nextPayment()` can be `null` after the billing cycle ends or a subscription is canceled.

Refunds:

```php
$response = $transaction->refund('Accidental charge');

$response = $transaction->refund('Accidental charge', [
    'pri_a',
    'pri_b' => 200,
]);
```

Credits:

```php
$response = $transaction->credit('Compensation', 'pri_a');
```

Refunds require Paddle approval. Credits apply only to manually collected transactions.

## 23. Testing guidance

Manual testing is required for the full billing flow.

Automated tests can use Laravel HTTP Client fakes to avoid real Paddle API calls. Also test:

- checkout creation returns expected view/session object
- checkout success page shows pending until webhook sync
- webhook listeners update records idempotently
- duplicate webhook delivery does not double-complete orders
- subscription middleware allows/blocks correctly
- trial, grace period, canceled, paused, and past-due logic
- price catalog references are centralized and environment-aware
- `.env.example` documents all required values

## 24. Common failure modes

- Granting access on checkout return instead of webhook sync.
- Forgetting `@paddleJS` in the layout.
- Forgetting CSRF exclusion for `paddle/*`.
- Missing `PADDLE_WEBHOOK_SECRET` in non-local environments.
- Mixing Paddle Sandbox and live credentials/price IDs.
- Treating `subscribed()` as synonymous with actively recurring paid state.
- Renaming subscription types after records exist.
- Trying to resume canceled subscriptions.
- Modifying paused or past-due subscriptions.
- Removing the final price from a multi-product subscription instead of canceling.
- Writing tests that only mock the happy path and never test delayed webhooks.
