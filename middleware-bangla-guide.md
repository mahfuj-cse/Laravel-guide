# 4️⃣ Laravel Middleware - বিস্তারিত বাংলা গাইড

## 📋 সূচিপত্র
- [Middleware কি?](#middleware-কি)
- [Middleware তৈরি করা](#middleware-তৈরি-করা)
- [Global Middleware](#global-middleware)
- [Route-specific Middleware](#route-specific-middleware)
- [Middleware Groups](#middleware-groups)
- [Middleware Parameters](#middleware-parameters)
- [Terminable Middleware](#terminable-middleware)
- [Real-world Examples](#real-world-examples)

---

## Middleware কি?

**Middleware** হলো **HTTP Request** এবং **Response** এর মধ্যে **Filter** হিসেবে কাজ করে। এটি Request আসার পর এবং Response পাঠানোর আগে চলে।

### 🎯 Middleware এর কাজ:
- ✅ **Authentication** - User logged in কিনা check
- ✅ **Authorization** - Permission আছে কিনা check
- ✅ **Rate Limiting** - Request limit control
- ✅ **CORS** - Cross-origin request handle
- ✅ **Logging** - Request/Response log করা
- ✅ **Data Validation** - Input sanitization

### Request Flow with Middleware:
```
Request → Global Middleware → Route Middleware → Controller → Response
   ↑                                                              ↓
   └─────────────── Response Middleware ←─────────────────────────┘
```

---

## Middleware তৈরি করা

### ১. Basic Middleware তৈরি:
```bash
# Basic middleware
php artisan make:middleware CheckAge

# Middleware with resource
php artisan make:middleware AdminMiddleware
```

### ২. Simple Middleware Example:
```php
<?php
// app/Http/Middleware/CheckAge.php

namespace App\Http\Middleware;

use Closure;
use Illuminate\Http\Request;

class CheckAge
{
    public function handle(Request $request, Closure $next)
    {
        // Before request processing
        if ($request->age && $request->age < 18) {
            return redirect('home')->with('error', 'You must be 18 or older');
        }

        // Process request
        $response = $next($request);

        // After request processing (optional)
        // You can modify response here

        return $response;
    }
}
```

### ৩. Advanced Middleware with Logic:
```php
<?php
// app/Http/Middleware/AdminMiddleware.php

namespace App\Http\Middleware;

use Closure;
use Illuminate\Http\Request;
use Illuminate\Support\Facades\Auth;

class AdminMiddleware
{
    public function handle(Request $request, Closure $next)
    {
        // Check if user is authenticated
        if (!Auth::check()) {
            if ($request->expectsJson()) {
                return response()->json(['message' => 'Unauthenticated'], 401);
            }
            return redirect()->route('login');
        }

        // Check if user is admin
        $user = Auth::user();
        if (!$user->isAdmin()) {
            if ($request->expectsJson()) {
                return response()->json(['message' => 'Unauthorized'], 403);
            }
            abort(403, 'Access denied. Admin privileges required.');
        }

        return $next($request);
    }
}
```

### ৪. Middleware Registration:
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
        
        // Custom middleware
        'admin' => \App\Http\Middleware\AdminMiddleware::class,
        'check.age' => \App\Http\Middleware\CheckAge::class,
    ];
}
```

---

## Global Middleware

### ১. Global Middleware কি?
**Global Middleware** সব HTTP Request এ চলে।

```php
<?php
// app/Http/Middleware/LogRequests.php

namespace App\Http\Middleware;

use Closure;
use Illuminate\Http\Request;
use Illuminate\Support\Facades\Log;

class LogRequests
{
    public function handle(Request $request, Closure $next)
    {
        // Log incoming request
        Log::info('Incoming Request', [
            'method' => $request->method(),
            'url' => $request->fullUrl(),
            'ip' => $request->ip(),
            'user_agent' => $request->userAgent(),
            'timestamp' => now()->toDateTimeString()
        ]);

        $response = $next($request);

        // Log response
        Log::info('Outgoing Response', [
            'status_code' => $response->getStatusCode(),
            'content_type' => $response->headers->get('Content-Type')
        ]);

        return $response;
    }
}

// Register in Kernel.php
protected $middleware = [
    // Other middleware...
    \App\Http\Middleware\LogRequests::class,
];
```

### ২. Security Middleware:
```php
<?php
// app/Http/Middleware/SecurityHeaders.php

class SecurityHeaders
{
    public function handle(Request $request, Closure $next)
    {
        $response = $next($request);

        // Add security headers
        $response->headers->set('X-Content-Type-Options', 'nosniff');
        $response->headers->set('X-Frame-Options', 'DENY');
        $response->headers->set('X-XSS-Protection', '1; mode=block');
        $response->headers->set('Strict-Transport-Security', 'max-age=31536000; includeSubDomains');
        $response->headers->set('Content-Security-Policy', "default-src 'self'");

        return $response;
    }
}
```

### ৩. Maintenance Mode Middleware:
```php
<?php
// app/Http/Middleware/CheckMaintenanceMode.php

class CheckMaintenanceMode
{
    public function handle(Request $request, Closure $next)
    {
        if (config('app.maintenance_mode', false)) {
            // Allow admin access during maintenance
            if (auth()->check() && auth()->user()->isAdmin()) {
                return $next($request);
            }

            // Show maintenance page
            if ($request->expectsJson()) {
                return response()->json([
                    'message' => 'Site is under maintenance'
                ], 503);
            }

            return response()->view('maintenance', [], 503);
        }

        return $next($request);
    }
}
```

---

## Route-specific Middleware

### ১. Single Route Middleware:
```php
<?php
// Routes with middleware

// Single middleware
Route::get('/admin/dashboard', [AdminController::class, 'dashboard'])
     ->middleware('admin');

// Multiple middleware
Route::get('/profile', [ProfileController::class, 'show'])
     ->middleware(['auth', 'verified']);

// Middleware with parameters
Route::get('/api/posts', [PostController::class, 'index'])
     ->middleware('throttle:60,1');
```

### ২. Controller Middleware:
```php
<?php
// app/Http/Controllers/AdminController.php

class AdminController extends Controller
{
    public function __construct()
    {
        // Apply to all methods
        $this->middleware('admin');
        
        // Apply to specific methods
        $this->middleware('auth')->only(['create', 'store', 'edit', 'update']);
        
        // Apply except specific methods
        $this->middleware('throttle:10,1')->except(['index', 'show']);
    }

    public function dashboard()
    {
        return view('admin.dashboard');
    }

    public function users()
    {
        return view('admin.users');
    }
}
```

### ৩. Route Group Middleware:
```php
<?php
// Route groups with middleware

// Admin routes
Route::middleware(['auth', 'admin'])->prefix('admin')->group(function () {
    Route::get('/dashboard', [AdminController::class, 'dashboard']);
    Route::resource('/users', AdminUserController::class);
    Route::resource('/posts', AdminPostController::class);
});

// API routes
Route::middleware(['auth:sanctum', 'throttle:api'])->prefix('api')->group(function () {
    Route::get('/user', function (Request $request) {
        return $request->user();
    });
    Route::apiResource('/posts', PostController::class);
});

// Verified user routes
Route::middleware(['auth', 'verified'])->group(function () {
    Route::get('/dashboard', [DashboardController::class, 'index']);
    Route::get('/profile', [ProfileController::class, 'show']);
});
```

---

## Middleware Groups

### ১. Custom Middleware Group:
```php
<?php
// app/Http/Kernel.php

protected $middlewareGroups = [
    'web' => [
        // Default web middleware
    ],

    'api' => [
        // Default API middleware
    ],

    // Custom middleware group
    'admin' => [
        'auth',
        'admin',
        'throttle:100,1',
        'verified'
    ],

    'guest' => [
        'guest',
        'throttle:10,1'
    ],

    'secure' => [
        'auth',
        'verified',
        'password.confirm'
    ]
];

// Usage
Route::middleware('admin')->group(function () {
    Route::get('/admin/dashboard', [AdminController::class, 'dashboard']);
});
```

### ২. Dynamic Middleware Groups:
```php
<?php
// app/Providers/RouteServiceProvider.php

public function boot()
{
    parent::boot();

    // Dynamic middleware based on environment
    if (app()->environment('production')) {
        $this->app['router']->middlewareGroup('secure', [
            'auth',
            'verified',
            'throttle:strict'
        ]);
    } else {
        $this->app['router']->middlewareGroup('secure', [
            'auth'
        ]);
    }
}
```

---

## Middleware Parameters

### ১. Middleware with Parameters:
```php
<?php
// app/Http/Middleware/CheckRole.php

class CheckRole
{
    public function handle(Request $request, Closure $next, ...$roles)
    {
        if (!auth()->check()) {
            return redirect('login');
        }

        $user = auth()->user();
        
        // Check if user has any of the required roles
        if (!$user->hasAnyRole($roles)) {
            abort(403, 'Insufficient permissions');
        }

        return $next($request);
    }
}

// Register middleware
protected $routeMiddleware = [
    'role' => \App\Http\Middleware\CheckRole::class,
];

// Usage with parameters
Route::get('/admin/users', [UserController::class, 'index'])
     ->middleware('role:admin,manager');

Route::get('/editor/posts', [PostController::class, 'index'])
     ->middleware('role:admin,editor,author');
```

### ২. Complex Parameter Middleware:
```php
<?php
// app/Http/Middleware/CheckSubscription.php

class CheckSubscription
{
    public function handle(Request $request, Closure $next, $plan = 'basic', $feature = null)
    {
        $user = auth()->user();
        
        if (!$user) {
            return redirect('login');
        }

        // Check subscription plan
        if (!$user->hasSubscription($plan)) {
            return redirect('subscription.upgrade')
                   ->with('error', "You need {$plan} subscription to access this feature");
        }

        // Check specific feature if provided
        if ($feature && !$user->hasFeature($feature)) {
            return redirect('subscription.upgrade')
                   ->with('error', "Feature '{$feature}' is not available in your plan");
        }

        return $next($request);
    }
}

// Usage
Route::get('/premium/analytics', [AnalyticsController::class, 'index'])
     ->middleware('subscription:premium,analytics');

Route::get('/pro/exports', [ExportController::class, 'index'])
     ->middleware('subscription:pro,bulk_export');
```

### ৩. Rate Limiting Middleware:
```php
<?php
// Custom rate limiting

// app/Http/Middleware/CustomThrottle.php
class CustomThrottle
{
    public function handle(Request $request, Closure $next, $maxAttempts = 60, $decayMinutes = 1, $prefix = '')
    {
        $key = $this->resolveRequestSignature($request, $prefix);
        
        if (RateLimiter::tooManyAttempts($key, $maxAttempts)) {
            return response()->json([
                'message' => 'Too many requests',
                'retry_after' => RateLimiter::availableIn($key)
            ], 429);
        }

        RateLimiter::hit($key, $decayMinutes * 60);

        $response = $next($request);

        // Add rate limit headers
        $response->headers->add([
            'X-RateLimit-Limit' => $maxAttempts,
            'X-RateLimit-Remaining' => RateLimiter::remaining($key, $maxAttempts),
        ]);

        return $response;
    }

    protected function resolveRequestSignature($request, $prefix)
    {
        return sha1($prefix . '|' . $request->ip() . '|' . $request->userAgent());
    }
}

// Usage
Route::post('/api/upload', [UploadController::class, 'store'])
     ->middleware('custom.throttle:5,1,upload');
```

---

## Terminable Middleware

### ১. Terminable Middleware কি?
**Terminable Middleware** Response পাঠানোর পর চলে।

```php
<?php
// app/Http/Middleware/PerformanceLogger.php

class PerformanceLogger
{
    protected $startTime;

    public function handle(Request $request, Closure $next)
    {
        $this->startTime = microtime(true);
        
        return $next($request);
    }

    public function terminate(Request $request, $response)
    {
        $executionTime = microtime(true) - $this->startTime;
        
        // Log performance data
        Log::info('Request Performance', [
            'url' => $request->fullUrl(),
            'method' => $request->method(),
            'execution_time' => round($executionTime * 1000, 2) . 'ms',
            'memory_usage' => round(memory_get_peak_usage(true) / 1024 / 1024, 2) . 'MB',
            'status_code' => $response->getStatusCode(),
            'user_id' => auth()->id(),
        ]);

        // Clean up temporary files
        if (isset($request->temp_files)) {
            foreach ($request->temp_files as $file) {
                @unlink($file);
            }
        }
    }
}
```

### ২. Analytics Middleware:
```php
<?php
// app/Http/Middleware/TrackPageViews.php

class TrackPageViews
{
    public function handle(Request $request, Closure $next)
    {
        return $next($request);
    }

    public function terminate(Request $request, $response)
    {
        // Only track successful GET requests
        if ($request->method() === 'GET' && $response->getStatusCode() === 200) {
            
            // Track page view
            PageView::create([
                'url' => $request->fullUrl(),
                'ip_address' => $request->ip(),
                'user_agent' => $request->userAgent(),
                'user_id' => auth()->id(),
                'session_id' => session()->getId(),
                'referer' => $request->header('referer'),
                'created_at' => now()
            ]);

            // Update user activity
            if (auth()->check()) {
                auth()->user()->update(['last_activity_at' => now()]);
            }
        }
    }
}
```

---

## Real-world Examples

### ১. API Authentication & Rate Limiting:
```php
<?php
// app/Http/Middleware/ApiAuthentication.php

class ApiAuthentication
{
    public function handle(Request $request, Closure $next)
    {
        $token = $request->bearerToken() ?? $request->header('X-API-Key');
        
        if (!$token) {
            return response()->json(['message' => 'API token required'], 401);
        }

        $apiKey = ApiKey::where('key', $token)->first();
        
        if (!$apiKey || !$apiKey->isActive()) {
            return response()->json(['message' => 'Invalid API token'], 401);
        }

        // Check rate limits
        $rateLimitKey = 'api_rate_limit:' . $apiKey->id;
        $maxRequests = $apiKey->rate_limit ?? 1000;
        
        if (RateLimiter::tooManyAttempts($rateLimitKey, $maxRequests)) {
            return response()->json([
                'message' => 'Rate limit exceeded',
                'retry_after' => RateLimiter::availableIn($rateLimitKey)
            ], 429);
        }

        RateLimiter::hit($rateLimitKey, 3600); // 1 hour window

        // Set authenticated API key
        $request->attributes->set('api_key', $apiKey);

        return $next($request);
    }
}
```

### ২. Multi-tenant Middleware:
```php
<?php
// app/Http/Middleware/SetTenant.php

class SetTenant
{
    public function handle(Request $request, Closure $next)
    {
        $subdomain = $this->getSubdomain($request);
        
        if (!$subdomain) {
            return redirect('https://main.example.com');
        }

        $tenant = Tenant::where('subdomain', $subdomain)->first();
        
        if (!$tenant) {
            abort(404, 'Tenant not found');
        }

        if (!$tenant->isActive()) {
            return response()->view('tenant.suspended', [], 503);
        }

        // Set tenant context
        app()->instance('tenant', $tenant);
        config(['database.connections.tenant.database' => $tenant->database]);
        
        // Switch database connection
        DB::purge('tenant');
        DB::reconnect('tenant');

        return $next($request);
    }

    protected function getSubdomain(Request $request)
    {
        $host = $request->getHost();
        $parts = explode('.', $host);
        
        return count($parts) > 2 ? $parts[0] : null;
    }
}
```

### ৩. Content Security Policy:
```php
<?php
// app/Http/Middleware/ContentSecurityPolicy.php

class ContentSecurityPolicy
{
    public function handle(Request $request, Closure $next)
    {
        $response = $next($request);

        // Generate nonce for inline scripts
        $nonce = base64_encode(random_bytes(16));
        $request->attributes->set('csp_nonce', $nonce);

        // Build CSP header
        $csp = [
            "default-src 'self'",
            "script-src 'self' 'nonce-{$nonce}' https://cdn.jsdelivr.net",
            "style-src 'self' 'unsafe-inline' https://fonts.googleapis.com",
            "font-src 'self' https://fonts.gstatic.com",
            "img-src 'self' data: https:",
            "connect-src 'self' https://api.example.com",
            "frame-ancestors 'none'",
            "base-uri 'self'",
            "form-action 'self'"
        ];

        $response->headers->set('Content-Security-Policy', implode('; ', $csp));

        return $response;
    }
}
```

### ৪. Request Sanitization:
```php
<?php
// app/Http/Middleware/SanitizeInput.php

class SanitizeInput
{
    protected $except = [
        'password',
        'password_confirmation',
        'current_password'
    ];

    public function handle(Request $request, Closure $next)
    {
        $input = $request->all();
        
        array_walk_recursive($input, function (&$value, $key) {
            if (!in_array($key, $this->except) && is_string($value)) {
                // Remove potentially dangerous characters
                $value = strip_tags($value);
                $value = htmlspecialchars($value, ENT_QUOTES, 'UTF-8');
                
                // Remove extra whitespace
                $value = trim($value);
                $value = preg_replace('/\s+/', ' ', $value);
            }
        });

        $request->replace($input);

        return $next($request);
    }
}
```

### ৫. Feature Flag Middleware:
```php
<?php
// app/Http/Middleware/CheckFeatureFlag.php

class CheckFeatureFlag
{
    public function handle(Request $request, Closure $next, $feature)
    {
        // Check global feature flag
        if (!config("features.{$feature}", false)) {
            abort(404);
        }

        // Check user-specific feature flag
        if (auth()->check()) {
            $user = auth()->user();
            
            if (!$user->hasFeature($feature)) {
                if ($request->expectsJson()) {
                    return response()->json([
                        'message' => 'Feature not available'
                    ], 403);
                }
                
                return redirect()->back()
                       ->with('error', 'This feature is not available for your account');
            }
        }

        return $next($request);
    }
}

// Usage
Route::get('/beta/new-dashboard', [DashboardController::class, 'beta'])
     ->middleware('feature:new_dashboard');

Route::get('/premium/analytics', [AnalyticsController::class, 'index'])
     ->middleware(['auth', 'feature:premium_analytics']);
```

---

## 🎯 Best Practices:

### ✅ **Performance:**
- **Lightweight middleware** - Heavy operations এ terminable middleware ব্যবহার করুন
- **Early returns** - Conditions fail হলে তাড়াতাড়ি return করুন
- **Caching** - Expensive operations cache করুন

### ✅ **Security:**
- **Input validation** - User input সবসময় validate করুন
- **Rate limiting** - API endpoints protect করুন
- **CSRF protection** - Web routes এ CSRF enable রাখুন

### ✅ **Organization:**
- **Single responsibility** - Each middleware একটি কাজ করবে
- **Meaningful names** - Descriptive middleware names ব্যবহার করুন
- **Proper grouping** - Related middleware group করুন

### ✅ **Testing:**
```php
// Middleware testing
public function test_admin_middleware_blocks_non_admin_users()
{
    $user = User::factory()->create(['role' => 'user']);
    
    $response = $this->actingAs($user)
                     ->get('/admin/dashboard');
                     
    $response->assertStatus(403);
}

public function test_admin_middleware_allows_admin_users()
{
    $admin = User::factory()->create(['role' => 'admin']);
    
    $response = $this->actingAs($admin)
                     ->get('/admin/dashboard');
                     
    $response->assertStatus(200);
}
```

---

## 📚 আরও জানতে:
- [Laravel Middleware](https://laravel.com/docs/middleware)
- [HTTP Kernel](https://laravel.com/docs/lifecycle#http-kernel)
- [Rate Limiting](https://laravel.com/docs/routing#rate-limiting)