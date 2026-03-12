# RevenueCat Webhook Integration — Laravel
[![Youtube][youtube-shield]][youtube-url]
[![Facebook][facebook-shield]][facebook-url]
[![Instagram][instagram-shield]][instagram-url]
[![LinkedIn][linkedin-shield]][linkedin-url]

Thanks for visiting my GitHub account!

## Overview

This integration handles RevenueCat webhook events in a Laravel application. It verifies incoming webhook authenticity, stores raw events for idempotency, resolves users, updates subscription records, and toggles premium status based on event type.

---

## Requirements

- PHP 8.1+
- Laravel 10+
- A `users` table with an `is_premium` boolean column
- RevenueCat project with webhook integration configured

---

## File Structure

```
app/
  Http/
    Controllers/
      Api/
        RevenueCatWebhookController.php
  Models/
    RevenueCatEvent.php
    Subscription.php
database/
  migrations/
    xxxx_xx_xx_create_subscriptions_table.php
    xxxx_xx_xx_create_revenue_cat_events_table.php
    xxxx_xx_xx_add_is_premium_to_users_table.php
config/
  services.php
routes/
  api.php
bootstrap/
  app.php
.env
```

---

## Installation

### Step 1: Create the migrations

Run the following to generate migration files:

```bash
php artisan make:migration create_subscriptions_table
php artisan make:migration create_revenue_cat_events_table
php artisan make:migration add_is_premium_to_users_table
```

### subscriptions migration

```php
Schema::create('subscriptions', function (Blueprint $table) {
    $table->id();
    $table->foreignId('user_id')->constrained()->onDelete('cascade');
    $table->string('original_transaction_id');
    $table->string('product_id')->nullable();
    $table->string('entitlement')->nullable();
    $table->string('status');
    $table->string('currency')->nullable();
    $table->decimal('price', 8, 2)->nullable();
    $table->decimal('price_in_purchased_currency', 8, 2)->nullable();
    $table->datetime('purchase_date');
    $table->datetime('expires_date');
    $table->string('environment')->nullable();
    $table->timestamps();
});
```

### revenue_cat_events migration

```php
Schema::create('revenue_cat_events', function (Blueprint $table) {
    $table->id();
    $table->string('rc_event_id')->unique();
    $table->string('app_id')->nullable();
    $table->string('event_type');
    $table->json('payload');
    $table->timestamps();
});
```

### add is_premium to users migration

```php
Schema::table('users', function (Blueprint $table) {
    $table->boolean('is_premium')->default(false)->after('email');
});
```

### Step 2: Run migrations

```bash
php artisan migrate
```

---

## Models

### app/Models/Subscription.php

```php
<?php

namespace App\Models;

use Illuminate\Database\Eloquent\Factories\HasFactory;
use Illuminate\Database\Eloquent\Model;

class Subscription extends Model
{
    use HasFactory;

    protected $guarded = [];

    protected $hidden = ['created_at', 'updated_at'];

    protected $casts = [
        'purchase_date'               => 'datetime',
        'expires_date'                => 'datetime',
        'created_at'                  => 'datetime',
        'updated_at'                  => 'datetime',
        'price'                       => 'decimal:2',
        'price_in_purchased_currency' => 'decimal:2',
    ];

    public function user()
    {
        return $this->belongsTo(User::class);
    }
}
```

### app/Models/RevenueCatEvent.php

```php
<?php

namespace App\Models;

use Illuminate\Database\Eloquent\Factories\HasFactory;
use Illuminate\Database\Eloquent\Model;

class RevenueCatEvent extends Model
{
    use HasFactory;

    protected $guarded = [];

    protected $hidden = ['created_at', 'updated_at'];

    protected $casts = [
        'payload'    => 'array',
        'created_at' => 'datetime',
        'updated_at' => 'datetime',
    ];
}
```

---

## Controller

### app/Http/Controllers/Api/RevenueCatWebhookController.php

```php
<?php

namespace App\Http\Controllers\Api;

use App\Http\Controllers\Controller;
use App\Models\RevenueCatEvent;
use App\Models\Subscription;
use App\Models\User;
use App\Traits\ApiResponse;
use Carbon\Carbon;
use Illuminate\Http\Request;
use Illuminate\Support\Facades\DB;
use Illuminate\Support\Facades\Log;

class RevenueCatWebhookController extends Controller
{
    use ApiResponse;

    public function handle(Request $request)
    {
        if (! $this->isAuthorized($request)) {
            Log::warning('RevenueCat webhook unauthorized request');
            return $this->error([], 'Unauthorized', 401);
        }

        $payload = $request->all();
        $event   = $payload['event'] ?? null;

        if (! $event || empty($event['id'])) {
            return $this->error([], 'Invalid event payload', 400);
        }

        Log::info('RevenueCat webhook received', [
            'event_id'   => $event['id'],
            'event_type' => $event['type'] ?? null,
        ]);

        if ($this->isDuplicateEvent($event['id'])) {
            Log::info('Duplicate RevenueCat event ignored', ['event_id' => $event['id']]);
            return $this->success([], 'Duplicate ignored', 200);
        }

        DB::beginTransaction();
        try {
            $this->storeEvent($event, $payload);
            $user = $this->resolveUser($event);

            if (! $user) {
                Log::warning('RevenueCat webhook user not found', [
                    'app_user_id' => $event['app_user_id'] ?? null,
                ]);
                DB::rollBack();
                return $this->error([], 'User not found', 404);
            }

            $this->updateSubscription($user, $event);
            $this->updateUserPremiumStatus($user, $event);

            DB::commit();
            return $this->success([], 'Webhook processed successfully');

        } catch (\Throwable $e) {
            DB::rollBack();
            Log::error('RevenueCat webhook processing failed', [
                'error' => $e->getMessage(),
                'event' => $event,
            ]);
            return $this->error([], 'Webhook processing failed', 500);
        }
    }

    private function isAuthorized(Request $request): bool
    {
        $authHeader = $request->header('Authorization');
        if (! $authHeader) {
            return false;
        }

        $receivedSecret = str_starts_with($authHeader, 'Bearer ')
            ? trim(str_replace('Bearer', '', $authHeader))
            : $authHeader;

        $expectedSecret = config('services.revenuecat.webhook_secret');

        return hash_equals($expectedSecret, $receivedSecret);
    }

    private function isDuplicateEvent(string $eventId): bool
    {
        return RevenueCatEvent::where('rc_event_id', $eventId)->exists();
    }

    private function storeEvent(array $event, array $payload): void
    {
        RevenueCatEvent::create([
            'rc_event_id' => $event['id'],
            'app_id'      => $event['app_id'] ?? null,
            'event_type'  => $event['type'] ?? null,
            'payload'     => $payload,
        ]);
    }

    private function resolveUser(array $event): ?User
    {
        return User::where('email', $event['app_user_id'] ?? null)->first();
    }

    private function updateSubscription(User $user, array $event): void
    {
        $purchaseDate  = $this->parseDate($event['purchased_at_ms'] ?? null);
        $expiresDate   = $this->parseDate($event['expiration_at_ms'] ?? null);
        $transactionId = $event['original_transaction_id'] ?? $event['transaction_id'] ?? null;

        Subscription::updateOrCreate(
            ['original_transaction_id' => $transactionId],
            [
                'user_id'                     => $user->id,
                'product_id'                  => $event['product_id'] ?? null,
                'entitlement'                 => $event['entitlement_id'] ?? null,
                'status'                      => $this->mapStatus($event['type'] ?? ''),
                'price'                       => $event['price'] ?? null,
                'currency'                    => $event['currency'] ?? null,
                'price_in_purchased_currency' => $event['price_in_purchased_currency'] ?? null,
                'purchase_date'               => $purchaseDate,
                'expires_date'                => $expiresDate,
                'environment'                 => $event['environment'] ?? null,
            ]
        );
    }

    private function updateUserPremiumStatus(User $user, array $event): void
    {
        $status = $this->mapStatus($event['type'] ?? '');

        if ($status === 'active') {
            $user->update(['is_premium' => true]);
        }

        if (in_array($status, ['expired', 'cancelled'])) {
            $user->update(['is_premium' => false]);
        }
    }

    private function parseDate(?int $timestamp): ?Carbon
    {
        return $timestamp ? Carbon::createFromTimestampMs($timestamp) : null;
    }

    private function mapStatus(string $eventType): string
    {
        return match ($eventType) {
            'INITIAL_PURCHASE', 'RENEWAL' => 'active',
            'CANCELLATION'                => 'cancelled',
            'EXPIRATION'                  => 'expired',
            default                       => 'unknown',
        };
    }
}
```

---

## Configuration

### .env

```
REVENUECAT_WEBHOOK_SECRET="your_secret_here"
```

### config/services.php

Add inside the array:

```php
'revenuecat' => [
    'webhook_secret' => env('REVENUECAT_WEBHOOK_SECRET'),
],
```

---

## Route Registration

### routes/api.php

```php
use App\Http\Controllers\Api\RevenueCatWebhookController;

Route::post('/revenuecat/webhook', [RevenueCatWebhookController::class, 'handle']);
```

The endpoint will be available at: `POST /api/revenuecat/webhook`

---

## CSRF Exemption

RevenueCat sends POST requests without a CSRF token. Exempt the route in `bootstrap/app.php`:

```php
->withMiddleware(function (Middleware $middleware) {
    $middleware->validateCsrfTokens(except: [
        'api/revenuecat/webhook',
    ]);
})
```

If your API routes already use the `api` middleware group (which excludes CSRF by default in Laravel 11+), this step may not be necessary. Include it as a safeguard regardless.

---

## RevenueCat Dashboard Setup

1. Go to your RevenueCat project: `https://app.revenuecat.com/projects/{project_id}/integrations/webhooks`
2. Click "Add Webhook"
3. Set the URL to: `https://yourdomain.com/api/revenuecat/webhook`
4. Set the Authorization header value to match your `REVENUECAT_WEBHOOK_SECRET`
5. Select the events you want to receive (see Event Handling below)
6. Save and use "Test Webhook" to verify connectivity

Reference: https://www.revenuecat.com/docs/integrations/webhooks/sample-events

---

## Event Handling

The following RevenueCat event types are handled and mapped to internal subscription statuses:

| RevenueCat Event Type | Internal Status | is_premium |
|-----------------------|-----------------|------------|
| INITIAL_PURCHASE      | active          | true       |
| RENEWAL               | active          | true       |
| CANCELLATION          | cancelled       | false      |
| EXPIRATION            | expired         | false      |
| All others            | unknown         | unchanged  |

---

## User Resolution

The webhook resolves users by matching `event.app_user_id` against the `users.email` column.

This means: when configuring RevenueCat in your mobile app, the `app_user_id` passed to the RevenueCat SDK must be the user's email address.

```swift
// iOS example
Purchases.configure(withAPIKey: "your_api_key", appUserID: user.email)
```

```kotlin
// Android example
Purchases.configure(PurchasesConfiguration.Builder(context, "your_api_key").appUserID(user.email).build())
```

If you use a different identifier (e.g. UUID), update `resolveUser()` in the controller accordingly:

```php
private function resolveUser(array $event): ?User
{
    return User::where('uuid', $event['app_user_id'] ?? null)->first();
}
```

---

## Idempotency

Each incoming event is stored in the `revenue_cat_events` table using its `event.id` as a unique key (`rc_event_id`). Before processing, the controller checks for this ID. If the event already exists, it returns a 200 response and skips processing. This prevents duplicate subscription updates from retry deliveries.

---

## Security

Webhook authenticity is verified by comparing the `Authorization` header value against `REVENUECAT_WEBHOOK_SECRET` using `hash_equals()` to prevent timing attacks. Both `Bearer token` and raw header formats are supported.

Never expose the webhook secret in version control. Always load it from environment variables.

---

## Logging

The controller logs the following events to your Laravel log channel:

| Situation | Level | Message |
|-----------|-------|---------|
| Unauthorized request | warning | RevenueCat webhook unauthorized request |
| Event received | info | RevenueCat webhook received |
| Duplicate event | info | Duplicate RevenueCat event ignored |
| User not found | warning | RevenueCat webhook user not found |
| Processing failure | error | RevenueCat webhook processing failed |

To monitor in real time during development:

```bash
tail -f storage/logs/laravel.log
```

---

## Testing the Webhook Locally

Use a tunneling tool to expose your local server:

```bash
# Using ngrok
ngrok http 8000

# Using expose (Laravel)
expose share http://localhost:8000
```

Then update the RevenueCat dashboard webhook URL with the tunnel URL and use the "Send Test" feature to fire sample events.

You can also test manually with curl:

```bash
curl -X POST https://yourdomain.com/api/revenuecat/webhook \
  -H "Content-Type: application/json" \
  -H "Authorization: Bearer your_secret_here" \
  -d '{
    "event": {
      "id": "test-event-001",
      "type": "INITIAL_PURCHASE",
      "app_user_id": "user@example.com",
      "product_id": "premium_monthly",
      "entitlement_id": "premium",
      "original_transaction_id": "txn_abc123",
      "purchased_at_ms": 1700000000000,
      "expiration_at_ms": 1702678400000,
      "price": 9.99,
      "currency": "USD",
      "price_in_purchased_currency": 9.99,
      "environment": "PRODUCTION",
      "app_id": "app_abc"
    }
  }'
```

---

## Adapting for a New Project

Checklist when reusing this integration:

- [ ] Copy the three migrations and run `php artisan migrate`
- [ ] Copy `RevenueCatWebhookController.php`, `Subscription.php`, and `RevenueCatEvent.php`
- [ ] Add `REVENUECAT_WEBHOOK_SECRET` to `.env`
- [ ] Add the `revenuecat` block to `config/services.php`
- [ ] Register the route in `routes/api.php`
- [ ] Exempt the route from CSRF in `bootstrap/app.php`
- [ ] Confirm the `app_user_id` strategy matches your `resolveUser()` logic
- [ ] Add `is_premium` boolean to the `users` table if not present
- [ ] Confirm your `User` model has `is_premium` in its fillable or uses `$guarded = []`
- [ ] Configure the webhook URL in the RevenueCat dashboard

---

## Follow Me

[<img src='https://cdn.jsdelivr.net/npm/simple-icons@3.0.1/icons/github.svg' alt='github' height='40'>](https://github.com/learnwithfair) [<img src='https://cdn.jsdelivr.net/npm/simple-icons@3.0.1/icons/facebook.svg' alt='facebook' height='40'>](https://www.facebook.com/learnwithfair/) [<img src='https://cdn.jsdelivr.net/npm/simple-icons@3.0.1/icons/instagram.svg' alt='instagram' height='40'>](https://www.instagram.com/learnwithfair/) [<img src='https://cdn.jsdelivr.net/npm/simple-icons@3.0.1/icons/twitter.svg' alt='twitter' height='40'>](https://www.twiter.com/learnwithfair/) [<img src='https://cdn.jsdelivr.net/npm/simple-icons@3.0.1/icons/youtube.svg' alt='YouTube' height='40'>](https://www.youtube.com/@learnwithfair)

 <!-- MARKDOWN LINKS & IMAGES  -->

[youtube-shield]: https://img.shields.io/badge/-Youtube-black.svg?style=flat-square&logo=youtube&color=555&logoColor=white
[youtube-url]: https://youtube.com/@learnwithfair
[facebook-shield]: https://img.shields.io/badge/-Facebook-black.svg?style=flat-square&logo=facebook&color=555&logoColor=white
[facebook-url]: https://facebook.com/learnwithfair
[instagram-shield]: https://img.shields.io/badge/-Instagram-black.svg?style=flat-square&logo=instagram&color=555&logoColor=white
[instagram-url]: https://instagram.com/learnwithfair
[linkedin-shield]: https://img.shields.io/badge/-LinkedIn-black.svg?style=flat-square&logo=linkedin&colorB=555
[linkedin-url]: https://linkedin.com/company/learnwithfair

#learnwithfair #rahtulrabbi #rahatul-rabbi #learn-with-fair
