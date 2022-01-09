---
title: Laravel 8 RateLimit functionality
date: '2022-01-08'
spoiler: Working with rate limiters in Laravel 8
---

From Laravel 8 we were introduced with `RateLimit` facade which brings awesome new features for route protection, actions execution, and other things where rate limiting is needed.


### What do we need to start working with rate limiters?

```php
// Use RateLimiter facade
use Illuminate\Support\Facades\RateLimiter;
```

### Caching
Rate limiter uses cache for storing values of the first attempt and it compares its unique value with rate limit check attempts.
All configuration is located in `config/cache.php`

By default cache driver is `file`, but Laravel offers us much more drivers: `apc`, `array`, `database`, `memcached`, `redis`, `dynamodb`, `octane`
   
```php
    [
        //...
        'default' => env('CACHE_DRIVER', 'file'),
    ];
```

This option is also available through `.env`

```sh
CACHE_DRIVER=file
```
After changing its value through `.env` make sure to clean cached config values.

`php artisan optimize:clear`

### Using for actions in controllers or some other services

```php

    // For example
    use Acamposm\Ping\Ping;
    use Acamposm\Ping\PingCommandBuilder;

    $executed = RateLimiter::attempt(
        'pingServer='.$ip, // Unique key of this limit record
        $attempts = 2,     // Max attempts
        function($ip) {    // Closure
            
            // Ping specific ip
            $command = (new PingCommandBuilder($ip))->count(10)->packetSize(200)->ttl(128);

            // Run ping
            $ping = (new Ping($command))->run();

            // Handle response...
        },
        60 // 1 min is per default
    );

    // Check if RateLimit has allowed execution of this ping request
    if (!$executed) {
        return 'Too many pings sent!';
    }

```

For simpler usage `RateLimit` has provided us with `tooManyAttempts` static method which checks if we hit the request limit for a specific key in a one-minute interval.

```php

use Illuminate\Support\Facades\RateLimiter;

// Our custom non-existing exception handler for the Limit break
use App\Exceptions\LimitException;

if (RateLimiter::tooManyAttempts('pingServer='.$ip, $attempts = 1)) {
    throw new LimitException('You reached your ping limit for one minute.');      
}

```

Most useful by my opinion method on `RateLimit` is `availableIn`
What this method provides for us is to limit expiration.
The return time value is `seconds`.

```php

use Illuminate\Support\Facades\RateLimiter;
use App\Exceptions\LimitException;

if (RateLimiter::tooManyAttempts('pingServer='.$ip, $attempts = 1)) {

    // Call method
    $seconds = RateLimiter::availableIn('pingServer='.$ip);

    // Throw exception
    throw new LimitException('You reached your ping limit for one minute. Limit will expire in ' . $seconds);      
}

```

The last method is to clear the limit from the cache.

```php

use Illuminate\Support\Facades\RateLimiter;

// Call clear method
RateLimiter::clear('pingServer='.$ip);

```

## Using rate limiter with middlewares (api/web)

Rate limit also comes in handy with limiting hits on `api` and `web` endpoints.
This feature is available on single routes and a group of routes.

This middleware is already loaded by default in `app/Http/Kernel.php`

```php

/**
 * The application's route middleware.
 *
 * These middleware may be assigned to groups or used individually.
 *
 * @var array<string, class-string|string>
 */
protected $routeMiddleware = [
    //...
    'throttle' => \Illuminate\Routing\Middleware\ThrottleRequests::class,
];

```

The easiest way to configure rate limiting is to define a specific limit rule on `configureRateLimiting` method from `App\Providers\RouteServiceProvider`

Quick example.

```php
use Illuminate\Cache\RateLimiting\Limit;
use Illuminate\Support\Facades\RateLimiter;

/**
 * Configure the rate limiters for the application.
 *
 * @return void
 */
protected function configureRateLimiting()
{
    RateLimiter::for('global', function (Request $request) {
        return Limit::perMinute(1000);
    });
}

```

Rate limiting can also be dynamic where we can set other limits for different types of users.

In the example below we use `User` model provided with `$request`, where we check if the given user is premium type. If yes, we give him unlimited uploads, if not 1 per minute.

```php
use Illuminate\Cache\RateLimiting\Limit;
use Illuminate\Support\Facades\RateLimiter;

/**
 * Configure the rate limiters for the application.
 *
 * @return void
 */
protected function configureRateLimiting()
{
    RateLimiter::for('uploads', function (Request $request) {
        return $request->user()->type === 'premium'
                ? Limit::none() // Set unlimited
                : Limit::perMinute(1); 
    });
}

```

**Attaching rate limiters to routes**

*   Single routes
    ```php
    Route::middleware('throttle:uploads')->post('/internal/proof', [UploadController::class, 'createPost']);
    ```

*   Trough groups
    ```php
    Route::middleware(['throttle:uploads'])->group(function () {
        Route::post('/audio', function () {
            //
        });

        Route::post('/video', function () {
            //
        });
    });
    ```

**Return custom response on limit reach**

Limit cache helper contains method `response` which can invoke a response.

We can return any supported `response` method.
In the example below we use redirect back with errors in flash session so in views we can display error.

```php
use Illuminate\Cache\RateLimiting\Limit;
use Illuminate\Support\Facades\RateLimiter;

/**
 * Configure the rate limiters for the application.
 *
 * @return void
 */
protected function configureRateLimiting()
{
    RateLimiter::for('uploads', function (Request $request) {
        return Limit::perDay(1)->by($request->ip())->response(function(){
                return redirect()->back()->withErrors('limit', 'Limit reached');
            });
    });
}

```

**Extending rate limit middleware**

Why we would need to extend and override original methods?
One use case is to hide `X-RateLimit-Limit: x X-RateLimit-Remaining: x` headers from Laravel response on the route which has a rate limit.

Firstly we would need to create a new middleware in `app\Http\Midleware`. We can leave the same name.


```php

namespace App\Http\Middleware;

use Illuminate\Routing\Middleware\ThrottleRequests as OriginalThrottleRequests;

/**
...
 */
class ThrottleRequests extends OriginalThrottleRequests
{
    /**
     * @inheritdoc
     */
    protected function getHeaders($maxAttempts, $remainingAttempts, $retryAfter = null)
    {
        // Set empty array as headers response
        // Custom headers can be set here too
        return [];
    }
}

```

The last thing to make this work is to register this new middleware in `app/Http/Kernel.php`


```php

/**
 * The application's route middleware.
 *
 * These middleware may be assigned to groups or used individually.
 *
 * @var array<string, class-string|string>
 */
protected $routeMiddleware = [
    //...
    //'throttle' => \Illuminate\Routing\Middleware\ThrottleRequests::class,
    'throttle' => \App\Http\Middleware\ThrottleRequests::class,
    //...
];

```

We can still use both middlewares, but then we would need to change the name of middleware.

```php

/**
 * The application's route middleware.
 *
 * These middleware may be assigned to groups or used individually.
 *
 * @var array<string, class-string|string>
 */
protected $routeMiddleware = [
    //...
    'throttle' => \Illuminate\Routing\Middleware\ThrottleRequests::class,
    'throttle-noheaders' => \App\Http\Middleware\ThrottleRequests::class,
    //...
];

```

And then in routing files, we would need to set this new middleware.

```php

// throttle
Route::middleware('throttle:uploads')->post('/internal/video', [UploadController::class, 'uploadVideo']);

// throttle-noheaders
Route::middleware('throttle-noheaders:uploads')->post('/internal/noheaders', [UploadController::class, 'noHeaders']);

```
