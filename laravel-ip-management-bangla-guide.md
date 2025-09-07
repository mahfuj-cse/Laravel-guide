# Laravel IP Management - API Security গাইড

## 📋 সূচিপত্র
- [IP Address কি এবং কেন গুরুত্বপূর্ণ](#ip-address-কি-এবং-কেন-গুরুত্বপূর্ণ)
- [Laravel এ IP Address পাওয়া](#laravel-এ-ip-address-পাওয়া)
- [IP Based Rate Limiting](#ip-based-rate-limiting)
- [IP Blocking System](#ip-blocking-system)
- [Same IP Multiple Registration Block](#same-ip-multiple-registration-block)
- [IP Geolocation](#ip-geolocation)
- [IP Whitelist/Blacklist](#ip-whitelistblacklist)
- [Advanced IP Security](#advanced-ip-security)
- [Production Best Practices](#production-best-practices)

---

## IP Address কি এবং কেন গুরুত্বপূর্ণ

### 🌐 IP Address কি?
```php
<?php
// IP Address হলো প্রতিটি device এর unique identifier
// Example: 192.168.1.100, 103.230.107.50

// IPv4: 192.168.1.1 (সবচেয়ে common)
// IPv6: 2001:0db8:85a3:0000:0000:8a2e:0370:7334 (নতুন standard)
```

### 🔒 API Security এ IP এর ভূমিকা:
- **Rate Limiting**: একই IP থেকে অতিরিক্ত request block
- **Fraud Prevention**: Suspicious IP detect করা
- **Geographic Restrictions**: নির্দিষ্ট দেশ থেকে access block
- **Multiple Account Prevention**: একই IP থেকে multiple registration block
- **DDoS Protection**: Attack pattern detect করা

---

## Laravel এ IP Address পাওয়া

### ১. Basic IP Detection:

```php
<?php
// Controller এ IP পাওয়ার বিভিন্ন উপায়

class ApiController extends Controller
{
    public function getUserIP(Request $request)
    {
        // Method 1: Laravel Request helper
        $ip1 = $request->ip();
        
        // Method 2: Server variable
        $ip2 = $_SERVER['REMOTE_ADDR'] ?? null;
        
        // Method 3: Helper function
        $ip3 = request()->ip();
        
        // Method 4: Facade
        $ip4 = \Request::ip();
        
        return response()->json([
            'method_1' => $ip1,
            'method_2' => $ip2, 
            'method_3' => $ip3,
            'method_4' => $ip4,
            'all_headers' => $request->headers->all()
        ]);
    }
    
    public function getDetailedIP(Request $request)
    {
        return response()->json([
            'ip' => $request->ip(),
            'user_agent' => $request->userAgent(),
            'headers' => [
                'x_forwarded_for' => $request->header('X-Forwarded-For'),
                'x_real_ip' => $request->header('X-Real-IP'),
                'cf_connecting_ip' => $request->header('CF-Connecting-IP'), // Cloudflare
                'x_client_ip' => $request->header('X-Client-IP'),
            ],
            'server_info' => [
                'remote_addr' => $_SERVER['REMOTE_ADDR'] ?? null,
                'http_x_forwarded_for' => $_SERVER['HTTP_X_FORWARDED_FOR'] ?? null,
            ]
        ]);
    }
}
```

### ২. Proxy/Load Balancer এর জন্য IP Detection:

```php
<?php
// app/Http/Middleware/TrustProxies.php

namespace App\Http\Middleware;

use Illuminate\Http\Middleware\TrustProxies as Middleware;
use Illuminate\Http\Request;

class TrustProxies extends Middleware
{
    // Cloudflare, AWS Load Balancer এর জন্য
    protected $proxies = [
        '103.21.244.0/22',
        '103.22.200.0/22', 
        '103.31.4.0/22',
        // AWS ALB
        '10.0.0.0/8',
        '172.16.0.0/12',
        '192.168.0.0/16',
    ];

    protected $headers = Request::HEADER_X_FORWARDED_ALL;
}

// Custom IP Helper
class IPHelper
{
    public static function getRealIP()
    {
        $headers = [
            'HTTP_CF_CONNECTING_IP',     // Cloudflare
            'HTTP_CLIENT_IP',            // Proxy
            'HTTP_X_FORWARDED_FOR',      // Load balancer
            'HTTP_X_FORWARDED',          // Proxy
            'HTTP_X_CLUSTER_CLIENT_IP',  // Cluster
            'HTTP_FORWARDED_FOR',        // Proxy
            'HTTP_FORWARDED',            // Proxy
            'REMOTE_ADDR'                // Standard
        ];

        foreach ($headers as $header) {
            if (!empty($_SERVER[$header])) {
                $ips = explode(',', $_SERVER[$header]);
                $ip = trim($ips[0]);
                
                if (filter_var($ip, FILTER_VALIDATE_IP, FILTER_FLAG_NO_PRIV_RANGE | FILTER_FLAG_NO_RES_RANGE)) {
                    return $ip;
                }
            }
        }

        return $_SERVER['REMOTE_ADDR'] ?? 'unknown';
    }
}
```

---

## IP Based Rate Limiting

### ১. Laravel Built-in Rate Limiting:

```php
<?php
// app/Http/Kernel.php

protected $middlewareGroups = [
    'api' => [
        'throttle:api',
        \Illuminate\Routing\Middleware\SubstituteBindings::class,
    ],
];

// app/Providers/RouteServiceProvider.php
use Illuminate\Cache\RateLimiting\Limit;
use Illuminate\Http\Request;
use Illuminate\Support\Facades\RateLimiter;

public function boot()
{
    // Basic API rate limiting
    RateLimiter::for('api', function (Request $request) {
        return Limit::perMinute(60)->by($request->ip());
    });
    
    // Login rate limiting
    RateLimiter::for('login', function (Request $request) {
        return [
            Limit::perMinute(5)->by($request->ip()),
            Limit::perMinute(3)->by($request->ip())->when(
                $request->filled('email') && User::where('email', $request->email)->exists()
            ),
        ];
    });
    
    // Registration rate limiting
    RateLimiter::for('register', function (Request $request) {
        return Limit::perHour(3)->by($request->ip());
    });
    
    // Different limits for different IPs
    RateLimiter::for('dynamic', function (Request $request) {
        $ip = $request->ip();
        
        // VIP IPs - higher limit
        if (in_array($ip, ['103.230.107.50', '45.64.99.22'])) {
            return Limit::perMinute(1000)->by($ip);
        }
        
        // Suspicious IPs - lower limit
        if (IPBlacklist::where('ip', $ip)->exists()) {
            return Limit::perMinute(10)->by($ip);
        }
        
        // Default limit
        return Limit::perMinute(60)->by($ip);
    });
}
```

### ২. Route এ Rate Limiting Apply:

```php
<?php
// routes/api.php

// Basic throttling
Route::middleware('throttle:60,1')->group(function () {
    Route::get('/posts', [PostController::class, 'index']);
    Route::get('/users', [UserController::class, 'index']);
});

// Custom rate limiter
Route::middleware('throttle:login')->group(function () {
    Route::post('/login', [AuthController::class, 'login']);
});

Route::middleware('throttle:register')->group(function () {
    Route::post('/register', [AuthController::class, 'register']);
});

// Different limits for different endpoints
Route::prefix('api/v1')->group(function () {
    Route::middleware('throttle:100,1')->get('/search', [SearchController::class, 'search']);
    Route::middleware('throttle:10,1')->post('/upload', [FileController::class, 'upload']);
    Route::middleware('throttle:5,1')->post('/send-email', [EmailController::class, 'send']);
});
```

### ৩. Custom Rate Limiting Middleware:

```php
<?php
// app/Http/Middleware/CustomRateLimit.php

namespace App\Http\Middleware;

use Closure;
use Illuminate\Http\Request;
use Illuminate\Support\Facades\Cache;
use Illuminate\Support\Facades\RateLimiter;

class CustomRateLimit
{
    public function handle(Request $request, Closure $next, $maxAttempts = 60, $decayMinutes = 1)
    {
        $ip = $request->ip();
        $key = 'rate_limit:' . $ip;
        
        // Current attempts
        $attempts = Cache::get($key, 0);
        
        if ($attempts >= $maxAttempts) {
            return response()->json([
                'error' => 'Too many requests',
                'message' => "Rate limit exceeded. Try again in {$decayMinutes} minute(s)",
                'retry_after' => $decayMinutes * 60,
                'ip' => $ip
            ], 429);
        }
        
        // Increment attempts
        Cache::put($key, $attempts + 1, now()->addMinutes($decayMinutes));
        
        $response = $next($request);
        
        // Add rate limit headers
        $response->headers->set('X-RateLimit-Limit', $maxAttempts);
        $response->headers->set('X-RateLimit-Remaining', max(0, $maxAttempts - $attempts - 1));
        $response->headers->set('X-RateLimit-Reset', now()->addMinutes($decayMinutes)->timestamp);
        
        return $response;
    }
}

// Usage
Route::middleware('custom.rate.limit:30,5')->post('/api/heavy-task', [TaskController::class, 'process']);
```

---

## IP Blocking System

### ১. IP Blacklist Model:

```php
<?php
// Migration
php artisan make:migration create_ip_blacklists_table

// database/migrations/xxxx_create_ip_blacklists_table.php
public function up()
{
    Schema::create('ip_blacklists', function (Blueprint $table) {
        $table->id();
        $table->string('ip')->unique();
        $table->string('reason')->nullable();
        $table->enum('type', ['manual', 'auto', 'temporary']);
        $table->timestamp('blocked_at');
        $table->timestamp('expires_at')->nullable();
        $table->json('metadata')->nullable(); // Additional info
        $table->timestamps();
        
        $table->index(['ip', 'expires_at']);
    });
}

// app/Models/IPBlacklist.php
namespace App\Models;

use Illuminate\Database\Eloquent\Model;

class IPBlacklist extends Model
{
    protected $fillable = [
        'ip', 'reason', 'type', 'blocked_at', 'expires_at', 'metadata'
    ];
    
    protected $casts = [
        'blocked_at' => 'datetime',
        'expires_at' => 'datetime',
        'metadata' => 'array'
    ];
    
    public function scopeActive($query)
    {
        return $query->where(function ($q) {
            $q->whereNull('expires_at')
              ->orWhere('expires_at', '>', now());
        });
    }
    
    public function isExpired()
    {
        return $this->expires_at && $this->expires_at->isPast();
    }
    
    public static function isBlocked($ip)
    {
        return self::where('ip', $ip)->active()->exists();
    }
}
```

### ২. IP Blocking Middleware:

```php
<?php
// app/Http/Middleware/BlockBlacklistedIPs.php

namespace App\Http\Middleware;

use Closure;
use Illuminate\Http\Request;
use App\Models\IPBlacklist;

class BlockBlacklistedIPs
{
    public function handle(Request $request, Closure $next)
    {
        $ip = $request->ip();
        
        // Check if IP is blacklisted
        $blacklisted = IPBlacklist::where('ip', $ip)->active()->first();
        
        if ($blacklisted) {
            \Log::warning("Blocked IP attempted access: {$ip}", [
                'reason' => $blacklisted->reason,
                'url' => $request->fullUrl(),
                'user_agent' => $request->userAgent()
            ]);
            
            return response()->json([
                'error' => 'Access Denied',
                'message' => 'Your IP address has been blocked',
                'reason' => $blacklisted->reason,
                'contact' => 'support@example.com'
            ], 403);
        }
        
        return $next($request);
    }
}

// Register middleware
// app/Http/Kernel.php
protected $middleware = [
    \App\Http\Middleware\BlockBlacklistedIPs::class,
];
```

### ৩. Auto IP Blocking System:

```php
<?php
// app/Services/IPSecurityService.php

namespace App\Services;

use App\Models\IPBlacklist;
use Illuminate\Support\Facades\Cache;

class IPSecurityService
{
    public function trackSuspiciousActivity($ip, $activity, $severity = 1)
    {
        $key = "suspicious_activity:{$ip}";
        $activities = Cache::get($key, []);
        
        $activities[] = [
            'activity' => $activity,
            'severity' => $severity,
            'timestamp' => now()->toISOString()
        ];
        
        // Keep last 100 activities
        $activities = array_slice($activities, -100);
        Cache::put($key, $activities, now()->addHours(24));
        
        // Calculate threat score
        $threatScore = $this->calculateThreatScore($activities);
        
        if ($threatScore >= 10) {
            $this->autoBlockIP($ip, $activities, $threatScore);
        }
        
        return $threatScore;
    }
    
    private function calculateThreatScore($activities)
    {
        $score = 0;
        $recentActivities = collect($activities)->where('timestamp', '>=', now()->subHour()->toISOString());
        
        foreach ($recentActivities as $activity) {
            $score += $activity['severity'];
        }
        
        // Bonus score for rapid requests
        if ($recentActivities->count() > 50) {
            $score += 5;
        }
        
        return $score;
    }
    
    private function autoBlockIP($ip, $activities, $threatScore)
    {
        IPBlacklist::create([
            'ip' => $ip,
            'reason' => "Auto-blocked due to suspicious activity (Score: {$threatScore})",
            'type' => 'auto',
            'blocked_at' => now(),
            'expires_at' => now()->addHours(24), // 24 hour temporary block
            'metadata' => [
                'threat_score' => $threatScore,
                'activities' => array_slice($activities, -10), // Last 10 activities
                'auto_blocked' => true
            ]
        ]);
        
        \Log::warning("Auto-blocked IP: {$ip}", [
            'threat_score' => $threatScore,
            'activities_count' => count($activities)
        ]);
    }
    
    public function reportFailedLogin($ip)
    {
        $this->trackSuspiciousActivity($ip, 'failed_login', 2);
    }
    
    public function reportInvalidToken($ip)
    {
        $this->trackSuspiciousActivity($ip, 'invalid_token', 3);
    }
    
    public function reportSQLInjectionAttempt($ip)
    {
        $this->trackSuspiciousActivity($ip, 'sql_injection', 10);
    }
}

// Usage in Controller
class AuthController extends Controller
{
    protected $ipSecurity;
    
    public function __construct(IPSecurityService $ipSecurity)
    {
        $this->ipSecurity = $ipSecurity;
    }
    
    public function login(Request $request)
    {
        $ip = $request->ip();
        
        if (!Auth::attempt($request->only('email', 'password'))) {
            $this->ipSecurity->reportFailedLogin($ip);
            
            return response()->json([
                'error' => 'Invalid credentials'
            ], 401);
        }
        
        // Successful login
        return response()->json([
            'token' => auth()->user()->createToken('api')->plainTextToken
        ]);
    }
}
```

---

## Same IP Multiple Registration Block

### ১. IP Registration Tracking:

```php
<?php
// Migration for tracking registrations
php artisan make:migration create_ip_registrations_table

public function up()
{
    Schema::create('ip_registrations', function (Blueprint $table) {
        $table->id();
        $table->string('ip');
        $table->foreignId('user_id')->constrained()->onDelete('cascade');
        $table->timestamp('registered_at');
        $table->json('metadata')->nullable();
        $table->timestamps();
        
        $table->index(['ip', 'registered_at']);
    });
}

// Model
namespace App\Models;

class IPRegistration extends Model
{
    protected $fillable = ['ip', 'user_id', 'registered_at', 'metadata'];
    
    protected $casts = [
        'registered_at' => 'datetime',
        'metadata' => 'array'
    ];
    
    public function user()
    {
        return $this->belongsTo(User::class);
    }
    
    public static function getRegistrationCount($ip, $hours = 24)
    {
        return self::where('ip', $ip)
                   ->where('registered_at', '>=', now()->subHours($hours))
                   ->count();
    }
    
    public static function getRegistrationsForIP($ip)
    {
        return self::where('ip', $ip)
                   ->with('user:id,name,email,created_at')
                   ->orderBy('registered_at', 'desc')
                   ->get();
    }
}
```

### ২. Registration Validation:

```php
<?php
// app/Http/Requests/RegisterRequest.php

namespace App\Http\Requests;

use Illuminate\Foundation\Http\FormRequest;
use App\Models\IPRegistration;
use App\Rules\IPRegistrationLimit;

class RegisterRequest extends FormRequest
{
    public function rules()
    {
        return [
            'name' => 'required|string|max:255',
            'email' => 'required|email|unique:users',
            'password' => 'required|min:8|confirmed',
            'ip' => ['required', new IPRegistrationLimit()]
        ];
    }
    
    protected function prepareForValidation()
    {
        $this->merge([
            'ip' => $this->ip()
        ]);
    }
}

// app/Rules/IPRegistrationLimit.php
namespace App\Rules;

use Illuminate\Contracts\Validation\Rule;
use App\Models\IPRegistration;

class IPRegistrationLimit implements Rule
{
    private $maxRegistrations;
    private $timeWindow;
    
    public function __construct($maxRegistrations = 3, $timeWindow = 24)
    {
        $this->maxRegistrations = $maxRegistrations;
        $this->timeWindow = $timeWindow;
    }
    
    public function passes($attribute, $value)
    {
        $count = IPRegistration::getRegistrationCount($value, $this->timeWindow);
        return $count < $this->maxRegistrations;
    }
    
    public function message()
    {
        return "Too many registrations from this IP address. Maximum {$this->maxRegistrations} registrations allowed in {$this->timeWindow} hours.";
    }
}
```

### ৩. Registration Controller with IP Tracking:

```php
<?php
// app/Http/Controllers/AuthController.php

class AuthController extends Controller
{
    public function register(RegisterRequest $request)
    {
        $ip = $request->ip();
        
        // Check existing registrations from this IP
        $existingRegistrations = IPRegistration::getRegistrationsForIP($ip);
        
        if ($existingRegistrations->count() >= 3) {
            return response()->json([
                'error' => 'Registration limit exceeded',
                'message' => 'Maximum 3 accounts allowed per IP address',
                'existing_accounts' => $existingRegistrations->count(),
                'contact' => 'support@example.com'
            ], 422);
        }
        
        // Create user
        $user = User::create([
            'name' => $request->name,
            'email' => $request->email,
            'password' => Hash::make($request->password),
        ]);
        
        // Track IP registration
        IPRegistration::create([
            'ip' => $ip,
            'user_id' => $user->id,
            'registered_at' => now(),
            'metadata' => [
                'user_agent' => $request->userAgent(),
                'headers' => $request->headers->all(),
                'referrer' => $request->header('referer')
            ]
        ]);
        
        // Log registration
        \Log::info("New user registered from IP: {$ip}", [
            'user_id' => $user->id,
            'email' => $user->email,
            'total_from_ip' => $existingRegistrations->count() + 1
        ]);
        
        return response()->json([
            'message' => 'Registration successful',
            'user' => $user,
            'token' => $user->createToken('api')->plainTextToken
        ], 201);
    }
    
    public function getIPRegistrations(Request $request)
    {
        $ip = $request->ip();
        $registrations = IPRegistration::getRegistrationsForIP($ip);
        
        return response()->json([
            'ip' => $ip,
            'total_registrations' => $registrations->count(),
            'registrations' => $registrations
        ]);
    }
}
```

---

## IP Geolocation

### ১. GeoIP Setup:

```bash
# Install GeoIP package
composer require torann/geoip

# Publish config
php artisan vendor:publish --provider="Torann\GeoIP\GeoIPServiceProvider" --tag=config

# Download GeoIP database
php artisan geoip:update
```

### ২. GeoIP Configuration:

```php
<?php
// config/geoip.php

return [
    'default' => 'maxmind_database',
    
    'services' => [
        'maxmind_database' => [
            'class' => \Torann\GeoIP\Services\MaxMindDatabase::class,
            'database_path' => storage_path('app/geoip.mmdb'),
            'update_url' => 'https://geolite.maxmind.com/download/geoip/database/GeoLite2-City.mmdb.gz',
            'locales' => ['en'],
        ],
        
        'ipapi' => [
            'class' => \Torann\GeoIP\Services\IPApi::class,
            'secure' => true,
            'key' => env('IPAPI_KEY'),
            'continent_path' => storage_path('app/continents.json'),
            'lang' => 'en',
        ],
    ],
];
```

### ৩. GeoIP Usage:

```php
<?php
// app/Services/GeoLocationService.php

namespace App\Services;

use Torann\GeoIP\Facades\GeoIP;

class GeoLocationService
{
    public function getLocationInfo($ip)
    {
        try {
            $location = GeoIP::getLocation($ip);
            
            return [
                'ip' => $location->ip,
                'country' => $location->country,
                'country_code' => $location->iso_code,
                'state' => $location->state,
                'city' => $location->city,
                'postal_code' => $location->postal_code,
                'latitude' => $location->lat,
                'longitude' => $location->lon,
                'timezone' => $location->timezone,
                'currency' => $location->currency,
            ];
        } catch (\Exception $e) {
            return [
                'ip' => $ip,
                'error' => 'Unable to determine location',
                'message' => $e->getMessage()
            ];
        }
    }
    
    public function isFromBangladesh($ip)
    {
        $location = GeoIP::getLocation($ip);
        return $location->iso_code === 'BD';
    }
    
    public function isFromAllowedCountries($ip, $allowedCountries = ['BD', 'IN', 'US'])
    {
        $location = GeoIP::getLocation($ip);
        return in_array($location->iso_code, $allowedCountries);
    }
}

// Controller usage
class ApiController extends Controller
{
    protected $geoLocation;
    
    public function __construct(GeoLocationService $geoLocation)
    {
        $this->geoLocation = $geoLocation;
    }
    
    public function getLocationInfo(Request $request)
    {
        $ip = $request->ip();
        $location = $this->geoLocation->getLocationInfo($ip);
        
        return response()->json([
            'location' => $location,
            'is_bangladesh' => $this->geoLocation->isFromBangladesh($ip),
            'is_allowed_country' => $this->geoLocation->isFromAllowedCountries($ip)
        ]);
    }
}
```

### ৪. Country-based Access Control:

```php
<?php
// app/Http/Middleware/CountryRestriction.php

namespace App\Http\Middleware;

use Closure;
use Illuminate\Http\Request;
use App\Services\GeoLocationService;

class CountryRestriction
{
    protected $geoLocation;
    
    public function __construct(GeoLocationService $geoLocation)
    {
        $this->geoLocation = $geoLocation;
    }
    
    public function handle(Request $request, Closure $next, ...$allowedCountries)
    {
        $ip = $request->ip();
        
        if (!$this->geoLocation->isFromAllowedCountries($ip, $allowedCountries)) {
            $location = $this->geoLocation->getLocationInfo($ip);
            
            \Log::warning("Access denied from restricted country", [
                'ip' => $ip,
                'country' => $location['country'],
                'country_code' => $location['country_code'],
                'url' => $request->fullUrl()
            ]);
            
            return response()->json([
                'error' => 'Access Denied',
                'message' => 'Service not available in your country',
                'country' => $location['country'],
                'allowed_countries' => $allowedCountries
            ], 403);
        }
        
        return $next($request);
    }
}

// Usage in routes
Route::middleware('country.restriction:BD,IN,US')->group(function () {
    Route::get('/api/restricted-content', [ContentController::class, 'index']);
});
```

---

## IP Whitelist/Blacklist

### ১. IP Whitelist System:

```php
<?php
// Migration
php artisan make:migration create_ip_whitelists_table

public function up()
{
    Schema::create('ip_whitelists', function (Blueprint $table) {
        $table->id();
        $table->string('ip');
        $table->string('description')->nullable();
        $table->enum('type', ['admin', 'api', 'trusted']);
        $table->json('permissions')->nullable();
        $table->timestamp('expires_at')->nullable();
        $table->timestamps();
        
        $table->unique(['ip', 'type']);
    });
}

// Model
namespace App\Models;

class IPWhitelist extends Model
{
    protected $fillable = ['ip', 'description', 'type', 'permissions', 'expires_at'];
    
    protected $casts = [
        'permissions' => 'array',
        'expires_at' => 'datetime'
    ];
    
    public function scopeActive($query)
    {
        return $query->where(function ($q) {
            $q->whereNull('expires_at')
              ->orWhere('expires_at', '>', now());
        });
    }
    
    public static function isWhitelisted($ip, $type = null)
    {
        $query = self::where('ip', $ip)->active();
        
        if ($type) {
            $query->where('type', $type);
        }
        
        return $query->exists();
    }
}
```

### ২. Admin IP Restriction:

```php
<?php
// app/Http/Middleware/AdminIPRestriction.php

namespace App\Http\Middleware;

use Closure;
use Illuminate\Http\Request;
use App\Models\IPWhitelist;

class AdminIPRestriction
{
    public function handle(Request $request, Closure $next)
    {
        $ip = $request->ip();
        
        // Check if IP is whitelisted for admin access
        if (!IPWhitelist::isWhitelisted($ip, 'admin')) {
            \Log::warning("Unauthorized admin access attempt", [
                'ip' => $ip,
                'url' => $request->fullUrl(),
                'user_agent' => $request->userAgent()
            ]);
            
            return response()->json([
                'error' => 'Access Denied',
                'message' => 'Admin access restricted to authorized IPs only'
            ], 403);
        }
        
        return $next($request);
    }
}

// Usage
Route::middleware(['auth:sanctum', 'admin.ip'])->prefix('admin')->group(function () {
    Route::get('/dashboard', [AdminController::class, 'dashboard']);
    Route::get('/users', [AdminController::class, 'users']);
});
```

### ৩. API Key + IP Validation:

```php
<?php
// app/Http/Middleware/APIKeyIPValidation.php

namespace App\Http\Middleware;

use Closure;
use Illuminate\Http\Request;
use App\Models\APIKey;

class APIKeyIPValidation
{
    public function handle(Request $request, Closure $next)
    {
        $apiKey = $request->header('X-API-Key');
        $ip = $request->ip();
        
        if (!$apiKey) {
            return response()->json(['error' => 'API key required'], 401);
        }
        
        $keyRecord = APIKey::where('key', $apiKey)->active()->first();
        
        if (!$keyRecord) {
            return response()->json(['error' => 'Invalid API key'], 401);
        }
        
        // Check IP restrictions
        if ($keyRecord->ip_restrictions && !in_array($ip, $keyRecord->ip_restrictions)) {
            \Log::warning("API key used from unauthorized IP", [
                'api_key' => substr($apiKey, 0, 8) . '...',
                'ip' => $ip,
                'allowed_ips' => $keyRecord->ip_restrictions
            ]);
            
            return response()->json([
                'error' => 'IP not authorized for this API key',
                'ip' => $ip
            ], 403);
        }
        
        // Track usage
        $keyRecord->increment('usage_count');
        $keyRecord->update(['last_used_at' => now(), 'last_used_ip' => $ip]);
        
        $request->merge(['api_key_record' => $keyRecord]);
        
        return $next($request);
    }
}

// API Key Model
namespace App\Models;

class APIKey extends Model
{
    protected $fillable = [
        'name', 'key', 'ip_restrictions', 'rate_limit', 
        'usage_count', 'last_used_at', 'last_used_ip', 'expires_at'
    ];
    
    protected $casts = [
        'ip_restrictions' => 'array',
        'last_used_at' => 'datetime',
        'expires_at' => 'datetime'
    ];
    
    public function scopeActive($query)
    {
        return $query->where(function ($q) {
            $q->whereNull('expires_at')
              ->orWhere('expires_at', '>', now());
        });
    }
}
```

---

## Advanced IP Security

### ১. IP Reputation System:

```php
<?php
// app/Services/IPReputationService.php

namespace App\Services;

use Illuminate\Support\Facades\Http;
use Illuminate\Support\Facades\Cache;

class IPReputationService
{
    public function checkReputation($ip)
    {
        $cacheKey = "ip_reputation:{$ip}";
        
        return Cache::remember($cacheKey, now()->addHours(6), function () use ($ip) {
            return $this->fetchReputationData($ip);
        });
    }
    
    private function fetchReputationData($ip)
    {
        $reputation = [
            'ip' => $ip,
            'is_malicious' => false,
            'threat_score' => 0,
            'categories' => [],
            'sources' => []
        ];
        
        try {
            // AbuseIPDB check
            $abuseResponse = Http::withHeaders([
                'Key' => config('services.abuseipdb.key'),
                'Accept' => 'application/json'
            ])->get('https://api.abuseipdb.com/api/v2/check', [
                'ipAddress' => $ip,
                'maxAgeInDays' => 90,
                'verbose' => ''
            ]);
            
            if ($abuseResponse->successful()) {
                $data = $abuseResponse->json()['data'];
                $reputation['threat_score'] += $data['abuseConfidencePercentage'];
                $reputation['categories'] = array_merge($reputation['categories'], $data['usageType']);
                $reputation['sources'][] = 'AbuseIPDB';
            }
            
            // VirusTotal check
            $vtResponse = Http::withHeaders([
                'x-apikey' => config('services.virustotal.key')
            ])->get("https://www.virustotal.com/vtapi/v2/ip-address/report", [
                'apikey' => config('services.virustotal.key'),
                'ip' => $ip
            ]);
            
            if ($vtResponse->successful()) {
                $data = $vtResponse->json();
                if (isset($data['detected_urls']) && count($data['detected_urls']) > 0) {
                    $reputation['threat_score'] += 30;
                    $reputation['categories'][] = 'malware';
                    $reputation['sources'][] = 'VirusTotal';
                }
            }
            
        } catch (\Exception $e) {
            \Log::error("IP reputation check failed: " . $e->getMessage());
        }
        
        $reputation['is_malicious'] = $reputation['threat_score'] > 50;
        
        return $reputation;
    }
    
    public function isTrustedIP($ip)
    {
        $reputation = $this->checkReputation($ip);
        return $reputation['threat_score'] < 10;
    }
}
```

### ২. Honeypot Detection:

```php
<?php
// app/Http/Middleware/HoneypotDetection.php

namespace App\Http\Middleware;

use Closure;
use Illuminate\Http\Request;
use App\Models\IPBlacklist;

class HoneypotDetection
{
    public function handle(Request $request, Closure $next)
    {
        // Check for honeypot field (should be empty)
        if ($request->filled('website') || $request->filled('url')) {
            $ip = $request->ip();
            
            // Auto-block bot IP
            IPBlacklist::create([
                'ip' => $ip,
                'reason' => 'Bot detected via honeypot field',
                'type' => 'auto',
                'blocked_at' => now(),
                'expires_at' => now()->addDays(7),
                'metadata' => [
                    'honeypot_field' => $request->only(['website', 'url']),
                    'user_agent' => $request->userAgent(),
                    'detection_method' => 'honeypot'
                ]
            ]);
            
            \Log::warning("Bot detected and blocked", [
                'ip' => $ip,
                'honeypot_data' => $request->only(['website', 'url'])
            ]);
            
            return response()->json(['error' => 'Invalid request'], 400);
        }
        
        return $next($request);
    }
}
```

### ৩. IP Analytics Dashboard:

```php
<?php
// app/Http/Controllers/IPAnalyticsController.php

class IPAnalyticsController extends Controller
{
    public function dashboard()
    {
        $stats = [
            'total_unique_ips' => $this->getUniqueIPCount(),
            'blocked_ips' => IPBlacklist::active()->count(),
            'whitelisted_ips' => IPWhitelist::active()->count(),
            'top_countries' => $this->getTopCountries(),
            'suspicious_activities' => $this->getSuspiciousActivities(),
            'recent_blocks' => $this->getRecentBlocks()
        ];
        
        return response()->json($stats);
    }
    
    private function getUniqueIPCount()
    {
        return \DB::table('ip_registrations')
                  ->distinct('ip')
                  ->count();
    }
    
    private function getTopCountries()
    {
        return \DB::table('ip_registrations')
                  ->select('country', \DB::raw('count(*) as count'))
                  ->groupBy('country')
                  ->orderBy('count', 'desc')
                  ->limit(10)
                  ->get();
    }
    
    private function getSuspiciousActivities()
    {
        return \DB::table('ip_blacklists')
                  ->where('created_at', '>=', now()->subDays(7))
                  ->where('type', 'auto')
                  ->count();
    }
    
    private function getRecentBlocks()
    {
        return IPBlacklist::with('metadata')
                         ->orderBy('created_at', 'desc')
                         ->limit(20)
                         ->get();
    }
    
    public function ipDetails($ip)
    {
        $geoLocation = new GeoLocationService();
        $ipReputation = new IPReputationService();
        
        return response()->json([
            'ip' => $ip,
            'location' => $geoLocation->getLocationInfo($ip),
            'reputation' => $ipReputation->checkReputation($ip),
            'registrations' => IPRegistration::getRegistrationsForIP($ip),
            'is_blacklisted' => IPBlacklist::isBlocked($ip),
            'is_whitelisted' => IPWhitelist::isWhitelisted($ip),
            'recent_activities' => $this->getIPActivities($ip)
        ]);
    }
}
```

---

## Production Best Practices

### ১. Environment Configuration:

```php
<?php
// .env
RATE_LIMIT_API=60
RATE_LIMIT_LOGIN=5
RATE_LIMIT_REGISTER=3

IP_BLOCK_ENABLED=true
IP_GEOLOCATION_ENABLED=true
IP_REPUTATION_CHECK=true

ABUSEIPDB_API_KEY=your_key_here
VIRUSTOTAL_API_KEY=your_key_here

// config/ip-security.php
return [
    'rate_limits' => [
        'api' => env('RATE_LIMIT_API', 60),
        'login' => env('RATE_LIMIT_LOGIN', 5),
        'register' => env('RATE_LIMIT_REGISTER', 3),
    ],
    
    'blocking' => [
        'enabled' => env('IP_BLOCK_ENABLED', true),
        'auto_block_threshold' => 10,
        'temp_block_duration' => 24, // hours
    ],
    
    'geolocation' => [
        'enabled' => env('IP_GEOLOCATION_ENABLED', true),
        'allowed_countries' => ['BD', 'IN', 'US', 'GB'],
    ],
    
    'reputation' => [
        'enabled' => env('IP_REPUTATION_CHECK', true),
        'threat_threshold' => 50,
        'cache_duration' => 360, // minutes
    ]
];
```

### ২. Monitoring & Alerts:

```php
<?php
// app/Console/Commands/IPSecurityMonitor.php

namespace App\Console\Commands;

use Illuminate\Console\Command;
use App\Models\IPBlacklist;
use App\Notifications\SecurityAlert;

class IPSecurityMonitor extends Command
{
    protected $signature = 'ip:monitor';
    protected $description = 'Monitor IP security metrics';
    
    public function handle()
    {
        $this->info('Starting IP security monitoring...');
        
        // Check for unusual blocking patterns
        $recentBlocks = IPBlacklist::where('created_at', '>=', now()->subHour())->count();
        
        if ($recentBlocks > 50) {
            $this->sendAlert("High blocking activity: {$recentBlocks} IPs blocked in last hour");
        }
        
        // Check for attacks from specific countries
        $suspiciousCountries = $this->checkSuspiciousCountries();
        
        if (!empty($suspiciousCountries)) {
            $this->sendAlert("Suspicious activity from countries: " . implode(', ', $suspiciousCountries));
        }
        
        $this->info('IP security monitoring completed.');
    }
    
    private function sendAlert($message)
    {
        // Send to admin
        $admin = User::where('role', 'admin')->first();
        $admin->notify(new SecurityAlert($message));
        
        // Log alert
        \Log::warning("IP Security Alert: {$message}");
    }
}

// Schedule in Kernel.php
protected function schedule(Schedule $schedule)
{
    $schedule->command('ip:monitor')->everyFiveMinutes();
}
```

### ৩. Performance Optimization:

```php
<?php
// Use Redis for caching IP data
// config/cache.php
'ip_cache' => [
    'driver' => 'redis',
    'connection' => 'default',
    'prefix' => 'ip_security',
],

// Optimized IP checking
class OptimizedIPService
{
    public function isBlocked($ip)
    {
        return Cache::store('ip_cache')->remember("blocked:{$ip}", 300, function () use ($ip) {
            return IPBlacklist::where('ip', $ip)->active()->exists();
        });
    }
    
    public function getLocationInfo($ip)
    {
        return Cache::store('ip_cache')->remember("location:{$ip}", 3600, function () use ($ip) {
            return GeoIP::getLocation($ip);
        });
    }
}

// Database indexing
// Add indexes for better performance
Schema::table('ip_blacklists', function (Blueprint $table) {
    $table->index(['ip', 'expires_at']);
    $table->index(['created_at', 'type']);
});

Schema::table('ip_registrations', function (Blueprint $table) {
    $table->index(['ip', 'registered_at']);
});
```

### ৪. Security Headers:

```php
<?php
// app/Http/Middleware/SecurityHeaders.php

class SecurityHeaders
{
    public function handle($request, Closure $next)
    {
        $response = $next($request);
        
        $response->headers->set('X-Content-Type-Options', 'nosniff');
        $response->headers->set('X-Frame-Options', 'DENY');
        $response->headers->set('X-XSS-Protection', '1; mode=block');
        $response->headers->set('Strict-Transport-Security', 'max-age=31536000; includeSubDomains');
        $response->headers->set('Content-Security-Policy', "default-src 'self'");
        $response->headers->set('X-Real-IP', $request->ip());
        
        return $response;
    }
}
```

---

## সারসংক্ষেপ

### 🔒 IP Security Checklist:
- ✅ Rate limiting implementation
- ✅ IP blocking system
- ✅ Multiple registration prevention
- ✅ Geolocation restrictions
- ✅ Reputation checking
- ✅ Monitoring & alerts
- ✅ Performance optimization

### 📊 Key Commands:
```bash
# IP security setup
php artisan make:middleware BlockBlacklistedIPs
php artisan make:model IPBlacklist -m
php artisan make:job CheckIPReputation

# Monitoring
php artisan ip:monitor
php artisan queue:work --queue=ip-security

# Cache management
php artisan cache:clear
php artisan config:cache
```

এই গাইড Laravel API এর জন্য comprehensive IP management এবং security system তৈরি করতে সাহায্য করবে।