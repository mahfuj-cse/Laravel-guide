# Laravel Concepts Guide

A comprehensive guide covering essential Laravel concepts for developers.

## Table of Contents

| #️⃣    | **Concept**                            | **Description**                                                |
| ------ | -------------------------------------- | -------------------------------------------------------------- |
| 1️⃣    | [Routing](#1️⃣-routing)                                | Route definition, parameters, route groups, and named routes   |
| 2️⃣    | [Controllers](#2️⃣-controllers)                            | Route-controller separation, resource controllers              |
| 3️⃣    | [Blade Templating](#3️⃣-blade-templating)                       | Blade syntax, layout inheritance, components                   |
| 4️⃣    | [Middleware](#4️⃣-middleware)                             | Custom middleware, global vs route-specific middleware         |
| 5️⃣    | [Request Lifecycle](#5️⃣-request-lifecycle)                      | Kernel, service providers, and bootstrap process               |
| 6️⃣    | [Request & Response](#6️⃣-request--response)                     | Form requests, validation, JSON responses                      |
| 7️⃣    | [Eloquent ORM](#7️⃣-eloquent-orm)                           | Models, relationships (hasOne, hasMany, etc.), mass assignment |
| 8️⃣    | [Query Builder](#8️⃣-query-builder)                          | Fluent SQL, joins, aggregations                                |
| 9️⃣    | [Migrations & Seeders](#9️⃣-migrations--seeders)                   | Schema building, database seeding, factories                   |
| 🔟     | [Authentication](#🔟-authentication) | Auth scaffolding, login/register, guards, token-based auth     |
| 1️⃣1️⃣ | [Authorization](#1️⃣1️⃣-authorization)                          | Gates, Policies, `@can`, `authorize()`                         |
| 1️⃣2️⃣ | [Laravel Collections](#1️⃣2️⃣-laravel-collections)                    | Powerful data manipulation using collection methods            |
| 1️⃣3️⃣ | [Dependency Injection](#1️⃣3️⃣-dependency-injection)                   | Injecting services, route/model binding                        |
| 1️⃣4️⃣ | [Service Container](#1️⃣4️⃣-service-container)                      | Binding interfaces, singletons, resolving dependencies         |
| 1️⃣5️⃣ | [Queues & Jobs](#1️⃣5️⃣-queues--jobs)                          | Async job handling using Redis/database/beanstalkd             |
| 1️⃣6️⃣ | [Events & Listeners](#1️⃣6️⃣-events--listeners)                     | Decoupled architecture via custom events                       |
| 1️⃣7️⃣ | [API Development](#1️⃣7️⃣-api-development)                        | API resources, rate limiting, versioning                       |
| 1️⃣8️⃣ | [Laravel Packages](#1️⃣8️⃣-laravel-packages)                       | Using, building and publishing reusable packages               |
| 1️⃣9️⃣ | [Broadcasting & WebSockets](#1️⃣9️⃣-broadcasting--websockets)              | Real-time updates with Pusher, Laravel Echo                    |
| 2️⃣0️⃣ | [Testing](#2️⃣0️⃣-testing)               | Test classes, HTTP tests, mocking, Pest/ PHPUnit               |

---

## 1️⃣ Routing

### Basic Routes
```php
Route::get('/', function () {
    return view('welcome');
});

Route::post('/users', [UserController::class, 'store']);
```

### Route Parameters
```php
Route::get('/user/{id}', function ($id) {
    return "User ID: " . $id;
});

Route::get('/posts/{post}/comments/{comment}', function ($postId, $commentId) {
    //
});
```

### Named Routes
```php
Route::get('/profile', [ProfileController::class, 'show'])->name('profile');

// Generate URL
$url = route('profile');
```

### Route Groups
```php
Route::prefix('admin')->middleware('auth')->group(function () {
    Route::get('/users', [UserController::class, 'index']);
    Route::get('/posts', [PostController::class, 'index']);
});
```

---

## 2️⃣ Controllers

### Basic Controller
```php
class UserController extends Controller
{
    public function index()
    {
        return User::all();
    }
    
    public function show($id)
    {
        return User::findOrFail($id);
    }
}
```

### Resource Controller
```php
Route::resource('posts', PostController::class);

class PostController extends Controller
{
    public function index() { /* GET /posts */ }
    public function create() { /* GET /posts/create */ }
    public function store(Request $request) { /* POST /posts */ }
    public function show($id) { /* GET /posts/{id} */ }
    public function edit($id) { /* GET /posts/{id}/edit */ }
    public function update(Request $request, $id) { /* PUT /posts/{id} */ }
    public function destroy($id) { /* DELETE /posts/{id} */ }
}
```

---

## 3️⃣ Blade Templating

### Layout
```blade
<!-- resources/views/layouts/app.blade.php -->
<!DOCTYPE html>
<html>
<head>
    <title>@yield('title')</title>
</head>
<body>
    @yield('content')
</body>
</html>
```

### Extending Layout
```blade
<!-- resources/views/posts/index.blade.php -->
@extends('layouts.app')

@section('title', 'Posts')

@section('content')
    @foreach($posts as $post)
        <h2>{{ $post->title }}</h2>
        <p>{{ $post->content }}</p>
    @endforeach
@endsection
```

### Components
```blade
<!-- resources/views/components/alert.blade.php -->
<div class="alert alert-{{ $type }}">
    {{ $slot }}
</div>

<!-- Usage -->
<x-alert type="success">
    Post created successfully!
</x-alert>
```

---

## 4️⃣ Middleware

### Creating Middleware
```php
php artisan make:middleware CheckAge

class CheckAge
{
    public function handle($request, Closure $next)
    {
        if ($request->age <= 200) {
            return redirect('home');
        }
        
        return $next($request);
    }
}
```

### Registering Middleware
```php
// app/Http/Kernel.php
protected $routeMiddleware = [
    'age' => \App\Http\Middleware\CheckAge::class,
];

// Usage
Route::get('admin/profile', function () {
    //
})->middleware('age');
```

---

## 5️⃣ Request Lifecycle

### Service Provider
```php
class AppServiceProvider extends ServiceProvider
{
    public function register()
    {
        $this->app->bind('App\Services\PaymentGateway', function ($app) {
            return new PaymentGateway($app['config']['services.payment']);
        });
    }
    
    public function boot()
    {
        View::composer('*', function ($view) {
            $view->with('currentUser', auth()->user());
        });
    }
}
```

---

## 6️⃣ Request & Response

### Form Request
```php
php artisan make:request StorePostRequest

class StorePostRequest extends FormRequest
{
    public function authorize()
    {
        return true;
    }
    
    public function rules()
    {
        return [
            'title' => 'required|max:255',
            'content' => 'required',
        ];
    }
}
```

### JSON Response
```php
public function store(StorePostRequest $request)
{
    $post = Post::create($request->validated());
    
    return response()->json([
        'message' => 'Post created successfully',
        'data' => $post
    ], 201);
}
```

---

## 7️⃣ Eloquent ORM

### Model
```php
class Post extends Model
{
    protected $fillable = ['title', 'content', 'user_id'];
    
    public function user()
    {
        return $this->belongsTo(User::class);
    }
    
    public function comments()
    {
        return $this->hasMany(Comment::class);
    }
}
```

### Relationships
```php
// One-to-Many
$user = User::find(1);
$posts = $user->posts;

// Many-to-Many
class User extends Model
{
    public function roles()
    {
        return $this->belongsToMany(Role::class);
    }
}
```

---

## 8️⃣ Query Builder

### Basic Queries
```php
$users = DB::table('users')
    ->where('active', 1)
    ->orderBy('name')
    ->get();

$posts = DB::table('posts')
    ->join('users', 'users.id', '=', 'posts.user_id')
    ->select('posts.*', 'users.name')
    ->get();
```

### Aggregations
```php
$count = DB::table('posts')->count();
$avg = DB::table('posts')->avg('views');
$max = DB::table('posts')->max('created_at');
```

---

## 9️⃣ Migrations & Seeders

### Migration
```php
php artisan make:migration create_posts_table

public function up()
{
    Schema::create('posts', function (Blueprint $table) {
        $table->id();
        $table->string('title');
        $table->text('content');
        $table->foreignId('user_id')->constrained();
        $table->timestamps();
    });
}
```

### Seeder
```php
php artisan make:seeder PostSeeder

class PostSeeder extends Seeder
{
    public function run()
    {
        Post::factory(50)->create();
    }
}
```

---

## 🔟 Authentication

### Laravel Breeze
```bash
composer require laravel/breeze --dev
php artisan breeze:install
npm install && npm run dev
php artisan migrate
```

### Sanctum (API)
```php
// User model
use Laravel\Sanctum\HasApiTokens;

class User extends Authenticatable
{
    use HasApiTokens;
}

// Generate token
$token = $user->createToken('api-token')->plainTextToken;
```

---

## 1️⃣1️⃣ Authorization

### Gates
```php
// AuthServiceProvider
Gate::define('update-post', function ($user, $post) {
    return $user->id === $post->user_id;
});

// Usage
if (Gate::allows('update-post', $post)) {
    // User can update post
}
```

### Policies
```php
php artisan make:policy PostPolicy

class PostPolicy
{
    public function update(User $user, Post $post)
    {
        return $user->id === $post->user_id;
    }
}

// Controller
public function update(Request $request, Post $post)
{
    $this->authorize('update', $post);
    // Update post
}
```

---

## 1️⃣2️⃣ Laravel Collections

```php
$collection = collect([1, 2, 3, 4, 5]);

$filtered = $collection->filter(function ($value) {
    return $value > 2;
});

$mapped = $collection->map(function ($item) {
    return $item * 2;
});

$users = User::all();
$grouped = $users->groupBy('role');
$plucked = $users->pluck('email', 'id');
```

---

## 1️⃣3️⃣ Dependency Injection

### Constructor Injection
```php
class PostController extends Controller
{
    protected $postService;
    
    public function __construct(PostService $postService)
    {
        $this->postService = $postService;
    }
}
```

### Route Model Binding
```php
Route::get('/posts/{post}', function (Post $post) {
    return $post;
});
```

---

## 1️⃣4️⃣ Service Container

### Binding
```php
// AppServiceProvider
$this->app->bind('PaymentGateway', function ($app) {
    return new PaymentGateway($app['config']['payment']);
});

$this->app->singleton('PaymentGateway', PaymentGateway::class);

// Interface binding
$this->app->bind(PaymentInterface::class, StripePayment::class);
```

---

## 1️⃣5️⃣ Queues & Jobs

### Job
```php
php artisan make:job ProcessPayment

class ProcessPayment implements ShouldQueue
{
    use Dispatchable, InteractsWithQueue, Queueable, SerializesModels;
    
    public function handle()
    {
        // Process payment logic
    }
}

// Dispatch
ProcessPayment::dispatch($paymentData);
```

---

## 1️⃣6️⃣ Events & Listeners

### Event
```php
php artisan make:event UserRegistered

class UserRegistered
{
    public $user;
    
    public function __construct(User $user)
    {
        $this->user = $user;
    }
}

// Listener
php artisan make:listener SendWelcomeEmail

class SendWelcomeEmail
{
    public function handle(UserRegistered $event)
    {
        Mail::to($event->user)->send(new WelcomeMail());
    }
}
```

---

## 1️⃣7️⃣ API Development

### API Resource
```php
php artisan make:resource PostResource

class PostResource extends JsonResource
{
    public function toArray($request)
    {
        return [
            'id' => $this->id,
            'title' => $this->title,
            'content' => $this->content,
            'author' => $this->user->name,
            'created_at' => $this->created_at->toDateTimeString(),
        ];
    }
}

// Usage
return PostResource::collection(Post::paginate());
```

### Rate Limiting
```php
// RouteServiceProvider
RateLimiter::for('api', function (Request $request) {
    return Limit::perMinute(60)->by($request->user()?->id ?: $request->ip());
});
```

---

## 1️⃣8️⃣ Laravel Packages

### Package Structure
```
packages/
└── vendor/
    └── package-name/
        ├── src/
        ├── config/
        ├── resources/
        └── composer.json
```

### Service Provider
```php
class PackageServiceProvider extends ServiceProvider
{
    public function boot()
    {
        $this->loadRoutesFrom(__DIR__.'/routes/web.php');
        $this->loadViewsFrom(__DIR__.'/resources/views', 'package');
        $this->publishes([
            __DIR__.'/config/package.php' => config_path('package.php'),
        ]);
    }
}
```

---

## 1️⃣9️⃣ Broadcasting & WebSockets

### Event Broadcasting
```php
class OrderShipped implements ShouldBroadcast
{
    use Dispatchable, InteractsWithSockets, SerializesModels;
    
    public function broadcastOn()
    {
        return new PrivateChannel('orders.'.$this->order->user_id);
    }
}
```

### Laravel Echo (Frontend)
```javascript
import Echo from 'laravel-echo';

window.Echo = new Echo({
    broadcaster: 'pusher',
    key: process.env.MIX_PUSHER_APP_KEY,
});

Echo.private(`orders.${userId}`)
    .listen('OrderShipped', (e) => {
        console.log(e.order);
    });
```

---

## 2️⃣0️⃣ Testing

### Feature Test
```php
class PostTest extends TestCase
{
    public function test_user_can_create_post()
    {
        $user = User::factory()->create();
        
        $response = $this->actingAs($user)->post('/posts', [
            'title' => 'Test Post',
            'content' => 'Test content'
        ]);
        
        $response->assertStatus(201);
        $this->assertDatabaseHas('posts', [
            'title' => 'Test Post'
        ]);
    }
}
```

### Unit Test
```php
class PostServiceTest extends TestCase
{
    public function test_can_create_post()
    {
        $service = new PostService();
        $post = $service->create([
            'title' => 'Test',
            'content' => 'Content'
        ]);
        
        $this->assertInstanceOf(Post::class, $post);
        $this->assertEquals('Test', $post->title);
    }
}
```

---

## Contributing

Feel free to contribute to this guide by submitting pull requests or opening issues.

## License

This guide is open-sourced software licensed under the [MIT license](https://opensource.org/licenses/MIT).