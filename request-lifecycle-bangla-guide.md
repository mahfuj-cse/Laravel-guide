# 5️⃣ Laravel Request Lifecycle - বিস্তারিত বাংলা গাইড

## 📋 সূচিপত্র
- [Request Lifecycle কি?](#request-lifecycle-কি)
- [Bootstrap Process](#bootstrap-process)
- [HTTP Kernel](#http-kernel)
- [Service Providers](#service-providers)
- [Middleware Pipeline](#middleware-pipeline)
- [Route Resolution](#route-resolution)
- [Response Generation](#response-generation)
- [Termination Process](#termination-process)

---

## Request Lifecycle কি?

**Request Lifecycle** হলো Laravel এ একটি **HTTP Request** আসার পর থেকে **Response** পাঠানো পর্যন্ত যে **পুরো প্রক্রিয়া** চলে।

### 🔄 Request Lifecycle এর ধাপসমূহ:
```
1. Entry Point (public/index.php)
2. Bootstrap Process
3. HTTP Kernel Loading
4. Service Providers Registration
5. Middleware Pipeline
6. Route Resolution
7. Controller/Action Execution
8. Response Generation
9. Middleware Response
10. Response Sending
11. Termination
```

### 🎯 কেন জানা জরুরি?
- ✅ **Performance Optimization** - কোথায় bottleneck হচ্ছে
- ✅ **Custom Middleware** - সঠিক জায়গায় logic যোগ করা
- ✅ **Service Provider** - কখন কি load হচ্ছে
- ✅ **Debugging** - সমস্যা খুঁজে বের করা

---

## Bootstrap Process

### ১. Entry Point (public/index.php):
```php
<?php
// public/index.php

use Illuminate\Contracts\Http\Kernel;
use Illuminate\Http\Request;

define('LARAVEL_START', microtime(true));

// Composer autoloader
require_once __DIR__.'/../vendor/autoload.php';

// Bootstrap Laravel application
$app = require_once __DIR__.'/../bootstrap/app.php';

// Resolve HTTP Kernel from container
$kernel = $app->make(Kernel::class);

// Handle the request
$response = $kernel->handle(
    $request = Request::capture()
);

// Send response to browser
$response->send();

// Terminate the application
$kernel->terminate($request, $response);
```

### ২. Application Bootstrap (bootstrap/app.php):
```php
<?php
// bootstrap/app.php

use Illuminate\Foundation\Application;

// Create Laravel Application instance
$app = new Application(
    $_ENV['APP_BASE_PATH'] ?? dirname(__DIR__)
);

// Bind important interfaces to concrete implementations
$app->singleton(
    Illuminate\Contracts\Http\Kernel::class,
    App\Http\Kernel::class
);

$app->singleton(
    Illuminate\Contracts\Console\Kernel::class,
    App\Console\Kernel::class
);

$app->singleton(
    Illuminate\Contracts\Debug\ExceptionHandler::class,
    App\Exceptions\Handler::class
);

return $app;
```

### ৩. Application Class Structure:
```php
<?php
// Illuminate\Foundation\Application

class Application extends Container implements ApplicationContract
{
    public function __construct($basePath = null)
    {
        if ($basePath) {
            $this->setBasePath($basePath);
        }

        $this->registerBaseBindings();
        $this->registerBaseServiceProviders();
        $this->registerCoreContainerAliases();
    }

    protected function registerBaseBindings()
    {
        static::setInstance($this);
        $this->instance('app', $this);
        $this->instance(Container::class, $this);
    }

    protected function registerBaseServiceProviders()
    {
        $this->register(new EventServiceProvider($this));
        $this->register(new LogServiceProvider($this));
        $this->register(new RoutingServiceProvider($this));
    }
}
```

---

## HTTP Kernel

### ১. HTTP Kernel Structure:
```php
<?php
// app/Http/Kernel.php

namespace App\Http;

use Illuminate\Foundation\Http\Kernel as HttpKernel;

class Kernel extends HttpKernel
{
    /**
     * Global HTTP middleware stack
     */
    protected $middleware = [
        \App\Http\Middleware\TrustProxies::class,
        \Illuminate\Http\Middleware\HandleCors::class,
        \App\Http\Middleware\PreventRequestsDuringMaintenance::class,
        \Illuminate\Foundation\Http\Middleware\ValidatePostSize::class,
        \App\Http\Middleware\TrimStrings::class,
        \Illuminate\Foundation\Http\Middleware\ConvertEmptyStringsToNull::class,
    ];

    /**
     * Route middleware groups
     */
    protected $middlewareGroups = [
        'web' => [
            \App\Http\Middleware\EncryptCookies::class,
            \Illuminate\Cookie\Middleware\AddQueuedCookiesToResponse::class,
            \Illuminate\Session\Middleware\StartSession::class,
            \Illuminate\View\Middleware\ShareErrorsFromSession::class,
            \App\Http\Middleware\VerifyCsrfToken::class,
            \Illuminate\Routing\Middleware\SubstituteBindings::class,
        ],

        'api' => [
            \Laravel\Sanctum\Http\Middleware\EnsureFrontendRequestsAreStateful::class,
            'throttle:api',
            \Illuminate\Routing\Middleware\SubstituteBindings::class,
        ],
    ];

    /**
     * Route middleware
     */
    protected $routeMiddleware = [
        'auth' => \App\Http\Middleware\Authenticate::class,
        'auth.basic' => \Illuminate\Auth\Middleware\AuthenticateWithBasicAuth::class,
        'cache.headers' => \Illuminate\Http\Middleware\SetCacheHeaders::class,
        'can' => \Illuminate\Auth\Middleware\Authorize::class,
        'guest' => \App\Http\Middleware\RedirectIfAuthenticated::class,
        'password.confirm' => \Illuminate\Auth\Middleware\RequirePassword::class,
        'signed' => \Illuminate\Routing\Middleware\ValidateSignature::class,
        'throttle' => \Illuminate\Routing\Middleware\ThrottleRequests::class,
        'verified' => \Illuminate\Auth\Middleware\EnsureEmailIsVerified::class,
    ];
}
```

### ২. Kernel Handle Method:
```php
<?php
// Illuminate\Foundation\Http\Kernel

public function handle($request)
{
    try {
        $request->enableHttpMethodParameterOverride();

        $response = $this->sendRequestThroughRouter($request);
    } catch (Throwable $e) {
        $this->reportException($e);
        $response = $this->renderException($request, $e);
    }

    $this->app['events']->dispatch(
        new RequestHandled($request, $response)
    );

    return $response;
}

protected function sendRequestThroughRouter($request)
{
    $this->app->instance('request', $request);

    Facade::clearResolvedInstance('request');

    $this->bootstrap();

    return (new Pipeline($this->app))
        ->send($request)
        ->through($this->app->shouldSkipMiddleware() ? [] : $this->middleware)
        ->then($this->dispatchToRouter());
}
```

---

## Service Providers

### ১. Service Provider কি?
**Service Provider** হলো Laravel Application এর **Bootstrap এর কেন্দ্রবিন্দু**। এরা Application এর বিভিন্ন components কে **register** এবং **boot** করে।

### ২. Service Provider Lifecycle:
```php
<?php
// Service Provider এর দুটি main method

class AppServiceProvider extends ServiceProvider
{
    /**
     * Register - Container এ services bind করা
     */
    public function register()
    {
        // Services container এ bind করা হয়
        $this->app->bind('PaymentGateway', function ($app) {
            return new PaymentGateway($app['config']['payment']);
        });

        $this->app->singleton('FileManager', function ($app) {
            return new FileManager($app['config']['filesystems']);
        });
    }

    /**
     * Boot - Application boot হওয়ার পর চলে
     */
    public function boot()
    {
        // View composers, event listeners, routes etc.
        View::composer('*', function ($view) {
            $view->with('currentUser', auth()->user());
        });

        // Model observers
        User::observe(UserObserver::class);

        // Custom validation rules
        Validator::extend('phone_number', function ($attribute, $value, $parameters, $validator) {
            return preg_match('/^01[3-9]\d{8}$/', $value);
        });
    }
}
```

### ৩. Core Service Providers:
```php
<?php
// config/app.php

'providers' => [
    // Laravel Framework Service Providers
    Illuminate\Auth\AuthServiceProvider::class,
    Illuminate\Broadcasting\BroadcastServiceProvider::class,
    Illuminate\Bus\BusServiceProvider::class,
    Illuminate\Cache\CacheServiceProvider::class,
    Illuminate\Foundation\Providers\ConsoleSupportServiceProvider::class,
    Illuminate\Cookie\CookieServiceProvider::class,
    Illuminate\Database\DatabaseServiceProvider::class,
    Illuminate\Encryption\EncryptionServiceProvider::class,
    Illuminate\Filesystem\FilesystemServiceProvider::class,
    Illuminate\Foundation\Providers\FoundationServiceProvider::class,
    Illuminate\Hashing\HashServiceProvider::class,
    Illuminate\Mail\MailServiceProvider::class,
    Illuminate\Notifications\NotificationServiceProvider::class,
    Illuminate\Pagination\PaginationServiceProvider::class,
    Illuminate\Pipeline\PipelineServiceProvider::class,
    Illuminate\Queue\QueueServiceProvider::class,
    Illuminate\Redis\RedisServiceProvider::class,
    Illuminate\Auth\Passwords\PasswordResetServiceProvider::class,
    Illuminate\Session\SessionServiceProvider::class,
    Illuminate\Translation\TranslationServiceProvider::class,
    Illuminate\Validation\ValidationServiceProvider::class,
    Illuminate\View\ViewServiceProvider::class,

    // Application Service Providers
    App\Providers\AppServiceProvider::class,
    App\Providers\AuthServiceProvider::class,
    App\Providers\EventServiceProvider::class,
    App\Providers\RouteServiceProvider::class,
],
```

### ৪. Custom Service Provider:
```php
<?php
// app/Providers/PaymentServiceProvider.php

namespace App\Providers;

use Illuminate\Support\ServiceProvider;
use App\Services\PaymentGateway;
use App\Services\StripePayment;
use App\Services\PaypalPayment;

class PaymentServiceProvider extends ServiceProvider
{
    public function register()
    {
        // Bind payment gateway based on config
        $this->app->bind(PaymentGateway::class, function ($app) {
            $gateway = config('payment.default');
            
            switch ($gateway) {
                case 'stripe':
                    return new StripePayment(config('payment.stripe'));
                case 'paypal':
                    return new PaypalPayment(config('payment.paypal'));
                default:
                    throw new \Exception("Unsupported payment gateway: {$gateway}");
            }
        });

        // Singleton for expensive services
        $this->app->singleton('payment.manager', function ($app) {
            return new PaymentManager($app);
        });
    }

    public function boot()
    {
        // Publish config files
        $this->publishes([
            __DIR__.'/../../config/payment.php' => config_path('payment.php'),
        ], 'payment-config');

        // Load routes
        $this->loadRoutesFrom(__DIR__.'/../../routes/payment.php');

        // Load views
        $this->loadViewsFrom(__DIR__.'/../../resources/views', 'payment');

        // Load migrations
        $this->loadMigrationsFrom(__DIR__.'/../../database/migrations');
    }

    public function provides()
    {
        return [PaymentGateway::class, 'payment.manager'];
    }
}
```

---

## Middleware Pipeline

### ১. Middleware Pipeline কিভাবে কাজ করে:
```php
<?php
// Middleware pipeline visualization

Request → Global Middleware → Route Middleware → Controller → Response
   ↑                                                              ↓
   └─────────────── Response Middleware ←─────────────────────────┘
```

### ২. Pipeline Implementation:
```php
<?php
// Illuminate\Pipeline\Pipeline

class Pipeline
{
    public function send($passable)
    {
        $this->passable = $passable;
        return $this;
    }

    public function through($pipes)
    {
        $this->pipes = is_array($pipes) ? $pipes : func_get_args();
        return $this;
    }

    public function then(Closure $destination)
    {
        $pipeline = array_reduce(
            array_reverse($this->pipes), 
            $this->carry(), 
            $this->prepareDestination($destination)
        );

        return $pipeline($this->passable);
    }

    protected function carry()
    {
        return function ($stack, $pipe) {
            return function ($passable) use ($stack, $pipe) {
                if (is_callable($pipe)) {
                    return $pipe($passable, $stack);
                }

                $parameters = [];
                if (is_string($pipe) && strpos($pipe, ':') !== false) {
                    [$name, $parameters] = explode(':', $pipe, 2);
                    $parameters = explode(',', $parameters);
                    $pipe = $name;
                }

                $carry = method_exists($pipe, $this->method)
                    ? $pipe->{$this->method}($passable, $stack, ...$parameters)
                    : $pipe($passable, $stack);

                return $carry;
            };
        };
    }
}
```

### ৩. Custom Middleware Example:
```php
<?php
// app/Http/Middleware/RequestLogger.php

namespace App\Http\Middleware;

use Closure;
use Illuminate\Support\Facades\Log;

class RequestLogger
{
    public function handle($request, Closure $next)
    {
        // Before request processing
        $startTime = microtime(true);
        
        Log::info('Request started', [
            'url' => $request->fullUrl(),
            'method' => $request->method(),
            'ip' => $request->ip(),
            'user_agent' => $request->userAgent(),
            'user_id' => auth()->id(),
        ]);

        // Process request
        $response = $next($request);

        // After request processing
        $endTime = microtime(true);
        $executionTime = ($endTime - $startTime) * 1000; // Convert to milliseconds

        Log::info('Request completed', [
            'url' => $request->fullUrl(),
            'status_code' => $response->getStatusCode(),
            'execution_time' => round($executionTime, 2) . 'ms',
            'memory_usage' => round(memory_get_peak_usage(true) / 1024 / 1024, 2) . 'MB',
        ]);

        return $response;
    }
}
```

---

## Route Resolution

### ১. Route Resolution Process:
```php
<?php
// Illuminate\Routing\Router

public function dispatch(Request $request)
{
    $this->currentRequest = $request;

    return $this->dispatchToRoute($request);
}

public function dispatchToRoute(Request $request)
{
    return $this->runRoute($request, $this->findRoute($request));
}

protected function findRoute($request)
{
    $this->current = $route = $this->routes->match($request);

    $this->container->instance(Route::class, $route);

    return $route;
}

protected function runRoute(Request $request, Route $route)
{
    $request->setRouteResolver(function () use ($route) {
        return $route;
    });

    $this->events->dispatch(new RouteMatched($route, $request));

    return $this->prepareResponse($request,
        $this->runRouteWithinStack($route, $request)
    );
}
```

### ২. Route Model Binding:
```php
<?php
// Route model binding process

// Route definition
Route::get('/posts/{post}', [PostController::class, 'show']);

// Laravel automatically resolves
public function show(Post $post)
{
    // $post is automatically loaded from database
    return view('posts.show', compact('post'));
}

// Custom route model binding
public function boot()
{
    Route::bind('post', function ($value) {
        return Post::where('slug', $value)->firstOrFail();
    });

    // Or in RouteServiceProvider
    Route::model('user', User::class);
}
```

### ৩. Route Caching:
```bash
# Cache routes for production
php artisan route:cache

# Clear route cache
php artisan route:clear

# List all routes
php artisan route:list
```

---

## Response Generation

### ১. Response Types:
```php
<?php
// Different response types

class PostController extends Controller
{
    public function index()
    {
        $posts = Post::all();
        
        // View response
        return view('posts.index', compact('posts'));
    }

    public function store(Request $request)
    {
        $post = Post::create($request->validated());
        
        // Redirect response
        return redirect()->route('posts.show', $post)
                        ->with('success', 'Post created successfully');
    }

    public function api()
    {
        $posts = Post::all();
        
        // JSON response
        return response()->json([
            'data' => $posts,
            'message' => 'Posts retrieved successfully'
        ]);
    }

    public function download()
    {
        // File download response
        return response()->download(storage_path('app/reports/monthly.pdf'));
    }

    public function stream()
    {
        // Streamed response
        return response()->stream(function () {
            echo 'Starting...' . PHP_EOL;
            ob_flush();
            flush();
            
            sleep(2);
            
            echo 'Processing...' . PHP_EOL;
            ob_flush();
            flush();
            
            sleep(2);
            
            echo 'Completed!' . PHP_EOL;
        }, 200, [
            'Content-Type' => 'text/plain',
        ]);
    }
}
```

### ২. Response Macros:
```php
<?php
// app/Providers/ResponseMacroServiceProvider.php

use Illuminate\Support\ServiceProvider;
use Illuminate\Support\Facades\Response;

class ResponseMacroServiceProvider extends ServiceProvider
{
    public function boot()
    {
        Response::macro('success', function ($data = null, $message = 'Success') {
            return Response::json([
                'success' => true,
                'message' => $message,
                'data' => $data
            ]);
        });

        Response::macro('error', function ($message = 'Error', $code = 400) {
            return Response::json([
                'success' => false,
                'message' => $message,
                'data' => null
            ], $code);
        });
    }
}

// Usage
return response()->success($posts, 'Posts retrieved successfully');
return response()->error('Validation failed', 422);
```

---

## Termination Process

### ১. Termination Callbacks:
```php
<?php
// app/Http/Kernel.php

public function terminate($request, $response)
{
    $this->terminateMiddleware($request, $response);

    $this->app->terminate();
}

protected function terminateMiddleware($request, $response)
{
    $middlewares = $this->app->shouldSkipMiddleware() ? [] : array_merge(
        $this->gatherRouteMiddleware($request),
        $this->middleware
    );

    foreach ($middlewares as $middleware) {
        if (! is_string($middleware)) {
            continue;
        }

        [$name] = $this->parseMiddleware($middleware);

        $instance = $this->app->make($name);

        if (method_exists($instance, 'terminate')) {
            $instance->terminate($request, $response);
        }
    }
}
```

### ২. Terminable Middleware:
```php
<?php
// app/Http/Middleware/PerformanceLogger.php

class PerformanceLogger
{
    public function handle($request, Closure $next)
    {
        $request->startTime = microtime(true);
        
        return $next($request);
    }

    public function terminate($request, $response)
    {
        $executionTime = microtime(true) - $request->startTime;
        
        Log::info('Request performance', [
            'url' => $request->fullUrl(),
            'method' => $request->method(),
            'execution_time' => $executionTime,
            'memory_peak' => memory_get_peak_usage(true),
            'status_code' => $response->getStatusCode(),
        ]);

        // Clean up resources
        if (isset($request->tempFiles)) {
            foreach ($request->tempFiles as $file) {
                @unlink($file);
            }
        }
    }
}
```

### ৩. Application Termination:
```php
<?php
// Illuminate\Foundation\Application

public function terminate()
{
    foreach ($this->terminatingCallbacks as $terminating) {
        $this->call($terminating);
    }
}

public function terminating(Closure $callback)
{
    $this->terminatingCallbacks[] = $callback;
    
    return $this;
}

// Usage in Service Provider
public function boot()
{
    $this->app->terminating(function () {
        // Cleanup code
        Cache::flush();
        Log::info('Application terminated');
    });
}
```

---

## Complete Request Flow Example

### 🔄 **Real-world Request Flow:**
```php
<?php
// Complete request lifecycle example

// 1. Request comes to public/index.php
// 2. Application bootstrapped
// 3. HTTP Kernel handles request

// 4. Global Middleware (in order)
class TrustProxies {} // Handle proxy headers
class HandleCors {} // CORS handling
class PreventRequestsDuringMaintenance {} // Maintenance mode
class ValidatePostSize {} // Check POST size
class TrimStrings {} // Trim input strings
class ConvertEmptyStringsToNull {} // Convert empty strings

// 5. Route-specific Middleware
Route::middleware(['web', 'auth', 'verified'])->group(function () {
    Route::get('/dashboard', [DashboardController::class, 'index']);
});

// 6. Controller execution
class DashboardController extends Controller
{
    public function index(Request $request)
    {
        // Business logic
        $stats = $this->getDashboardStats();
        
        return view('dashboard', compact('stats'));
    }
}

// 7. Response generation (View rendered)
// 8. Response middleware (reverse order)
// 9. Response sent to browser
// 10. Termination callbacks executed
```

---

## 🎯 Performance Optimization Tips:

### ✅ **Bootstrap Optimization:**
```php
// Cache configuration
php artisan config:cache

// Cache routes
php artisan route:cache

// Cache views
php artisan view:cache

// Optimize autoloader
composer install --optimize-autoloader --no-dev
```

### ✅ **Service Provider Optimization:**
```php
// Defer service providers when possible
class PaymentServiceProvider extends ServiceProvider
{
    protected $defer = true;

    public function provides()
    {
        return [PaymentGateway::class];
    }
}
```

### ✅ **Middleware Optimization:**
```php
// Keep middleware lightweight
// Use terminate() for heavy operations
// Cache expensive computations
```

---

## 📚 আরও জানতে:
- [Laravel Request Lifecycle](https://laravel.com/docs/lifecycle)
- [Service Providers](https://laravel.com/docs/providers)
- [Middleware](https://laravel.com/docs/middleware)