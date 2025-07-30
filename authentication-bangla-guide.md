# 🔟 Laravel Authentication - বিস্তারিত বাংলা গাইড

## 📋 সূচিপত্র
- [Authentication কি?](#authentication-কি)
- [Auth Scaffolding](#auth-scaffolding)
- [Login/Register System](#loginregister-system)
- [Guards কি?](#guards-কি)
- [Token-based Authentication](#token-based-authentication)
- [Custom Authentication](#custom-authentication)
- [সম্পূর্ণ উদাহরণ](#সম্পূর্ণ-উদাহরণ)

---

## Authentication কি?

**Authentication** হলো ব্যবহারকারীর **পরিচয় যাচাই** করার প্রক্রিয়া। Laravel এ Authentication এর মাধ্যমে:
- ✅ **Login/Register** সিস্টেম তৈরি
- ✅ **Password Reset** ফিচার
- ✅ **Email Verification** 
- ✅ **Multi-Guard** সাপোর্ট
- ✅ **API Token** ভিত্তিক Auth

---

## Auth Scaffolding

Laravel এ Authentication এর জন্য বিভিন্ন প্যাকেজ রয়েছে:

### ১. Laravel Breeze (সবচেয়ে সহজ):
```bash
# Breeze ইনস্টল
composer require laravel/breeze --dev

# Breeze Setup
php artisan breeze:install

# Frontend Assets Build
npm install && npm run dev

# Database Migration
php artisan migrate
```

### ২. Laravel UI (Bootstrap/Vue/React):
```bash
# Laravel UI ইনস্টল
composer require laravel/ui

# Bootstrap সহ Auth
php artisan ui bootstrap --auth

# Vue.js সহ Auth
php artisan ui vue --auth

# React সহ Auth
php artisan ui react --auth

# Assets Build
npm install && npm run dev
```

### ৩. Laravel Jetstream (Advanced):
```bash
# Jetstream ইনস্টল
composer require laravel/jetstream

# Livewire সহ
php artisan jetstream:install livewire

# Inertia.js সহ
php artisan jetstream:install inertia

# Migration
php artisan migrate
```

---

## Login/Register System

### ১. Manual Authentication Controller:
```php
<?php
// app/Http/Controllers/AuthController.php

namespace App\Http\Controllers;

use App\Http\Controllers\Controller;
use Illuminate\Http\Request;
use Illuminate\Support\Facades\Auth;
use Illuminate\Support\Facades\Hash;
use App\Models\User;

class AuthController extends Controller
{
    // Registration Form দেখানো
    public function showRegisterForm()
    {
        return view('auth.register');
    }

    // User Registration
    public function register(Request $request)
    {
        $request->validate([
            'name' => 'required|string|max:255',
            'email' => 'required|string|email|max:255|unique:users',
            'password' => 'required|string|min:8|confirmed',
        ]);

        $user = User::create([
            'name' => $request->name,
            'email' => $request->email,
            'password' => Hash::make($request->password),
        ]);

        Auth::login($user);

        return redirect()->route('dashboard');
    }

    // Login Form দেখানো
    public function showLoginForm()
    {
        return view('auth.login');
    }

    // User Login
    public function login(Request $request)
    {
        $credentials = $request->validate([
            'email' => 'required|email',
            'password' => 'required',
        ]);

        if (Auth::attempt($credentials, $request->boolean('remember'))) {
            $request->session()->regenerate();
            return redirect()->intended('dashboard');
        }

        return back()->withErrors([
            'email' => 'The provided credentials do not match our records.',
        ])->onlyInput('email');
    }

    // User Logout
    public function logout(Request $request)
    {
        Auth::logout();
        $request->session()->invalidate();
        $request->session()->regenerateToken();
        
        return redirect('/');
    }
}
```

### ২. Routes Setup:
```php
<?php
// routes/web.php

use App\Http\Controllers\AuthController;

// Guest Routes
Route::middleware('guest')->group(function () {
    Route::get('/register', [AuthController::class, 'showRegisterForm'])->name('register');
    Route::post('/register', [AuthController::class, 'register']);
    
    Route::get('/login', [AuthController::class, 'showLoginForm'])->name('login');
    Route::post('/login', [AuthController::class, 'login']);
});

// Authenticated Routes
Route::middleware('auth')->group(function () {
    Route::get('/dashboard', function () {
        return view('dashboard');
    })->name('dashboard');
    
    Route::post('/logout', [AuthController::class, 'logout'])->name('logout');
});
```

### ৩. Login Form (Blade):
```blade
<!-- resources/views/auth/login.blade.php -->
@extends('layouts.app')

@section('content')
<div class="container">
    <div class="row justify-content-center">
        <div class="col-md-8">
            <div class="card">
                <div class="card-header">Login</div>
                <div class="card-body">
                    <form method="POST" action="{{ route('login') }}">
                        @csrf
                        
                        <!-- Email -->
                        <div class="mb-3">
                            <label for="email" class="form-label">Email</label>
                            <input type="email" class="form-control @error('email') is-invalid @enderror" 
                                   id="email" name="email" value="{{ old('email') }}" required>
                            @error('email')
                                <div class="invalid-feedback">{{ $message }}</div>
                            @enderror
                        </div>

                        <!-- Password -->
                        <div class="mb-3">
                            <label for="password" class="form-label">Password</label>
                            <input type="password" class="form-control @error('password') is-invalid @enderror" 
                                   id="password" name="password" required>
                            @error('password')
                                <div class="invalid-feedback">{{ $message }}</div>
                            @enderror
                        </div>

                        <!-- Remember Me -->
                        <div class="mb-3 form-check">
                            <input type="checkbox" class="form-check-input" id="remember" name="remember">
                            <label class="form-check-label" for="remember">Remember Me</label>
                        </div>

                        <button type="submit" class="btn btn-primary">Login</button>
                        <a href="{{ route('register') }}" class="btn btn-link">Don't have account?</a>
                    </form>
                </div>
            </div>
        </div>
    </div>
</div>
@endsection
```

---

## Guards কি?

**Guards** হলো Authentication এর **বিভিন্ন পদ্ধতি**। Laravel এ Multiple Guards ব্যবহার করে বিভিন্ন ধরনের User authenticate করা যায়।

### ১. Guards Configuration:
```php
<?php
// config/auth.php

return [
    'defaults' => [
        'guard' => 'web',
        'passwords' => 'users',
    ],

    'guards' => [
        'web' => [
            'driver' => 'session',
            'provider' => 'users',
        ],

        'admin' => [
            'driver' => 'session',
            'provider' => 'admins',
        ],

        'api' => [
            'driver' => 'sanctum',
            'provider' => 'users',
        ],
    ],

    'providers' => [
        'users' => [
            'driver' => 'eloquent',
            'model' => App\Models\User::class,
        ],

        'admins' => [
            'driver' => 'eloquent',
            'model' => App\Models\Admin::class,
        ],
    ],
];
```

### ২. Multiple Guards ব্যবহার:
```php
<?php
// Admin Model
// app/Models/Admin.php

namespace App\Models;

use Illuminate\Foundation\Auth\User as Authenticatable;

class Admin extends Authenticatable
{
    protected $fillable = [
        'name', 'email', 'password',
    ];

    protected $hidden = [
        'password', 'remember_token',
    ];
}
```

### ৩. Admin Authentication Controller:
```php
<?php
// app/Http/Controllers/AdminAuthController.php

class AdminAuthController extends Controller
{
    public function showLoginForm()
    {
        return view('admin.auth.login');
    }

    public function login(Request $request)
    {
        $credentials = $request->validate([
            'email' => 'required|email',
            'password' => 'required',
        ]);

        if (Auth::guard('admin')->attempt($credentials)) {
            return redirect()->route('admin.dashboard');
        }

        return back()->withErrors(['email' => 'Invalid credentials']);
    }

    public function logout()
    {
        Auth::guard('admin')->logout();
        return redirect()->route('admin.login');
    }
}
```

### ৪. Guard-specific Routes:
```php
<?php
// routes/web.php

// Admin Routes
Route::prefix('admin')->name('admin.')->group(function () {
    Route::middleware('guest:admin')->group(function () {
        Route::get('/login', [AdminAuthController::class, 'showLoginForm'])->name('login');
        Route::post('/login', [AdminAuthController::class, 'login']);
    });

    Route::middleware('auth:admin')->group(function () {
        Route::get('/dashboard', function () {
            return view('admin.dashboard');
        })->name('dashboard');
        
        Route::post('/logout', [AdminAuthController::class, 'logout'])->name('logout');
    });
});
```

---

## Token-based Authentication

API এর জন্য Token-based Authentication ব্যবহার করা হয়।

### ১. Laravel Sanctum Setup:
```bash
# Sanctum ইনস্টল
composer require laravel/sanctum

# Configuration Publish
php artisan vendor:publish --provider="Laravel\Sanctum\SanctumServiceProvider"

# Migration চালানো
php artisan migrate
```

### ২. User Model এ HasApiTokens:
```php
<?php
// app/Models/User.php

namespace App\Models;

use Illuminate\Foundation\Auth\User as Authenticatable;
use Laravel\Sanctum\HasApiTokens;

class User extends Authenticatable
{
    use HasApiTokens;

    protected $fillable = [
        'name', 'email', 'password',
    ];
}
```

### ৩. API Authentication Controller:
```php
<?php
// app/Http/Controllers/Api/AuthController.php

namespace App\Http\Controllers\Api;

use App\Http\Controllers\Controller;
use Illuminate\Http\Request;
use Illuminate\Support\Facades\Hash;
use App\Models\User;

class AuthController extends Controller
{
    // User Registration
    public function register(Request $request)
    {
        $request->validate([
            'name' => 'required|string|max:255',
            'email' => 'required|string|email|max:255|unique:users',
            'password' => 'required|string|min:8',
        ]);

        $user = User::create([
            'name' => $request->name,
            'email' => $request->email,
            'password' => Hash::make($request->password),
        ]);

        $token = $user->createToken('auth-token')->plainTextToken;

        return response()->json([
            'message' => 'User registered successfully',
            'user' => $user,
            'token' => $token,
        ], 201);
    }

    // User Login
    public function login(Request $request)
    {
        $request->validate([
            'email' => 'required|email',
            'password' => 'required',
        ]);

        $user = User::where('email', $request->email)->first();

        if (!$user || !Hash::check($request->password, $user->password)) {
            return response()->json([
                'message' => 'Invalid credentials'
            ], 401);
        }

        $token = $user->createToken('auth-token')->plainTextToken;

        return response()->json([
            'message' => 'Login successful',
            'user' => $user,
            'token' => $token,
        ]);
    }

    // User Logout
    public function logout(Request $request)
    {
        $request->user()->currentAccessToken()->delete();

        return response()->json([
            'message' => 'Logged out successfully'
        ]);
    }

    // User Profile
    public function profile(Request $request)
    {
        return response()->json([
            'user' => $request->user()
        ]);
    }
}
```

### ৪. API Routes:
```php
<?php
// routes/api.php

use App\Http\Controllers\Api\AuthController;

// Public Routes
Route::post('/register', [AuthController::class, 'register']);
Route::post('/login', [AuthController::class, 'login']);

// Protected Routes
Route::middleware('auth:sanctum')->group(function () {
    Route::get('/profile', [AuthController::class, 'profile']);
    Route::post('/logout', [AuthController::class, 'logout']);
    
    // Other protected routes
    Route::get('/posts', [PostController::class, 'index']);
});
```

---

## Custom Authentication

### ১. Password Reset:
```php
<?php
// Password Reset Controller

use Illuminate\Support\Facades\Password;

class ForgotPasswordController extends Controller
{
    public function sendResetLinkEmail(Request $request)
    {
        $request->validate(['email' => 'required|email']);

        $status = Password::sendResetLink(
            $request->only('email')
        );

        return $status === Password::RESET_LINK_SENT
            ? back()->with(['status' => __($status)])
            : back()->withErrors(['email' => __($status)]);
    }
}

class ResetPasswordController extends Controller
{
    public function reset(Request $request)
    {
        $request->validate([
            'token' => 'required',
            'email' => 'required|email',
            'password' => 'required|min:8|confirmed',
        ]);

        $status = Password::reset(
            $request->only('email', 'password', 'password_confirmation', 'token'),
            function ($user, $password) {
                $user->forceFill([
                    'password' => Hash::make($password)
                ]);
                $user->save();
            }
        );

        return $status === Password::PASSWORD_RESET
            ? redirect()->route('login')->with('status', __($status))
            : back()->withErrors(['email' => [__($status)]]);
    }
}
```

### ২. Email Verification:
```php
<?php
// User Model এ MustVerifyEmail

use Illuminate\Contracts\Auth\MustVerifyEmail;

class User extends Authenticatable implements MustVerifyEmail
{
    use HasApiTokens, Notifiable;
    
    // ...
}

// Routes
Route::middleware(['auth', 'verified'])->group(function () {
    Route::get('/dashboard', function () {
        return view('dashboard');
    });
});
```

---

## সম্পূর্ণ উদাহরণ

### ১. Complete Auth System:
```php
<?php
// Complete Authentication System

// 1. User Registration with Email Verification
public function register(Request $request)
{
    $user = User::create([
        'name' => $request->name,
        'email' => $request->email,
        'password' => Hash::make($request->password),
    ]);

    // Send Email Verification
    $user->sendEmailVerificationNotification();

    Auth::login($user);
    
    return redirect()->route('verification.notice');
}

// 2. Login with Remember Me
public function login(Request $request)
{
    $credentials = $request->only('email', 'password');
    $remember = $request->boolean('remember');

    if (Auth::attempt($credentials, $remember)) {
        $request->session()->regenerate();
        
        // Log login activity
        activity()
            ->causedBy(Auth::user())
            ->log('User logged in');
            
        return redirect()->intended('dashboard');
    }

    return back()->withErrors(['email' => 'Invalid credentials']);
}

// 3. Two-Factor Authentication
public function enableTwoFactor(Request $request)
{
    $user = $request->user();
    $user->two_factor_secret = encrypt(Google2FA::generateSecretKey());
    $user->save();

    $qrCodeUrl = Google2FA::getQRCodeUrl(
        config('app.name'),
        $user->email,
        decrypt($user->two_factor_secret)
    );

    return response()->json(['qr_code' => $qrCodeUrl]);
}
```

### ২. Middleware for Role-based Access:
```php
<?php
// app/Http/Middleware/CheckRole.php

class CheckRole
{
    public function handle($request, Closure $next, ...$roles)
    {
        if (!Auth::check()) {
            return redirect('login');
        }

        $user = Auth::user();
        
        if (!in_array($user->role, $roles)) {
            abort(403, 'Unauthorized');
        }

        return $next($request);
    }
}

// Usage in routes
Route::middleware(['auth', 'role:admin,manager'])->group(function () {
    Route::get('/admin/users', [UserController::class, 'index']);
});
```

### ৩. API Authentication with Abilities:
```php
<?php
// Token with specific abilities

public function login(Request $request)
{
    // ... validation

    $user = User::where('email', $request->email)->first();
    
    // Create token with specific abilities
    $token = $user->createToken('auth-token', [
        'posts:read',
        'posts:create',
        'profile:update'
    ])->plainTextToken;

    return response()->json(['token' => $token]);
}

// Check abilities in controller
public function store(Request $request)
{
    if (!$request->user()->tokenCan('posts:create')) {
        return response()->json(['message' => 'Insufficient permissions'], 403);
    }
    
    // Create post
}
```

---

## 🎯 Best Practices:

### Security Tips:
- ✅ সবসময় HTTPS ব্যবহার করুন
- ✅ Strong Password Policy implement করুন
- ✅ Rate Limiting যোগ করুন
- ✅ Session Security configure করুন
- ✅ CSRF Protection enable রাখুন

### Performance Tips:
- ✅ Remember Me token ব্যবহার করুন
- ✅ Session Driver optimize করুন
- ✅ Token expiration সেট করুন
- ✅ Unnecessary queries এড়িয়ে চলুন

---

## 📚 আরও জানতে:
- [Laravel Authentication](https://laravel.com/docs/authentication)
- [Laravel Sanctum](https://laravel.com/docs/sanctum)
- [Laravel Breeze](https://laravel.com/docs/starter-kits#laravel-breeze)