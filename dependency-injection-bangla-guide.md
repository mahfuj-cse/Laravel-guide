# 1️⃣3️⃣ Laravel Dependency Injection - বিস্তারিত বাংলা গাইড

## 📋 সূচিপত্র
- [Dependency Injection কি?](#dependency-injection-কি)
- [Constructor Injection](#constructor-injection)
- [Method Injection](#method-injection)
- [Service Container](#service-container)
- [Route Model Binding](#route-model-binding)
- [Interface Binding](#interface-binding)
- [Advanced Techniques](#advanced-techniques)
- [Real-world Examples](#real-world-examples)

---

## Dependency Injection কি?

**Dependency Injection (DI)** হলো একটি **Design Pattern** যেখানে **Dependencies** class এর বাইরে থেকে **Inject** করা হয়।

### 🎯 কেন Dependency Injection ব্যবহার করবেন?
- ✅ **Loose Coupling** - Classes আলাদা থাকে
- ✅ **Testability** - Mock objects ব্যবহার করা যায়
- ✅ **Maintainability** - Code পরিবর্তন সহজ
- ✅ **Flexibility** - Runtime এ dependencies পরিবর্তন
- ✅ **Single Responsibility** - Each class has one job

### Without DI vs With DI:
```php
// ❌ Without Dependency Injection (Tight Coupling)
class OrderController
{
    public function store(Request $request)
    {
        // Hard-coded dependencies
        $paymentGateway = new StripePaymentGateway();
        $emailService = new SMTPEmailService();
        $logger = new FileLogger();
        
        $order = Order::create($request->all());
        
        $paymentGateway->charge($order->total);
        $emailService->send($order->customer->email, 'Order Confirmation');
        $logger->log('Order created: ' . $order->id);
        
        return response()->json($order);
    }
}

// ✅ With Dependency Injection (Loose Coupling)
class OrderController
{
    protected $paymentGateway;
    protected $emailService;
    protected $logger;
    
    public function __construct(
        PaymentGatewayInterface $paymentGateway,
        EmailServiceInterface $emailService,
        LoggerInterface $logger
    ) {
        $this->paymentGateway = $paymentGateway;
        $this->emailService = $emailService;
        $this->logger = $logger;
    }
    
    public function store(Request $request)
    {
        $order = Order::create($request->all());
        
        $this->paymentGateway->charge($order->total);
        $this->emailService->send($order->customer->email, 'Order Confirmation');
        $this->logger->log('Order created: ' . $order->id);
        
        return response()->json($order);
    }
}
```

---

## Constructor Injection

### ১. Basic Constructor Injection:
```php
<?php
// Service Class
class EmailService
{
    public function send($to, $subject, $message)
    {
        // Send email logic
        Mail::to($to)->send(new GenericMail($subject, $message));
    }
}

// Controller with Constructor Injection
class UserController extends Controller
{
    protected $emailService;
    
    public function __construct(EmailService $emailService)
    {
        $this->emailService = $emailService;
    }
    
    public function store(Request $request)
    {
        $user = User::create($request->validated());
        
        // Use injected service
        $this->emailService->send(
            $user->email,
            'Welcome!',
            'Thank you for registering'
        );
        
        return response()->json($user);
    }
}
```

### ২. Multiple Dependencies:
```php
<?php
class PostController extends Controller
{
    protected $postService;
    protected $cacheService;
    protected $notificationService;
    
    public function __construct(
        PostService $postService,
        CacheService $cacheService,
        NotificationService $notificationService
    ) {
        $this->postService = $postService;
        $this->cacheService = $cacheService;
        $this->notificationService = $notificationService;
    }
    
    public function store(Request $request)
    {
        $post = $this->postService->create($request->validated());
        
        $this->cacheService->forget('recent_posts');
        $this->notificationService->notifySubscribers($post);
        
        return response()->json($post);
    }
    
    public function index()
    {
        $posts = $this->cacheService->remember('recent_posts', 3600, function () {
            return $this->postService->getRecent(10);
        });
        
        return response()->json($posts);
    }
}
```

### ৩. Service Classes:
```php
<?php
// app/Services/PostService.php
namespace App\Services;

use App\Models\Post;
use App\Repositories\PostRepositoryInterface;

class PostService
{
    protected $postRepository;
    
    public function __construct(PostRepositoryInterface $postRepository)
    {
        $this->postRepository = $postRepository;
    }
    
    public function create(array $data)
    {
        $data['slug'] = Str::slug($data['title']);
        $data['user_id'] = auth()->id();
        
        return $this->postRepository->create($data);
    }
    
    public function getRecent($limit = 10)
    {
        return $this->postRepository->getRecent($limit);
    }
    
    public function publish(Post $post)
    {
        return $this->postRepository->update($post->id, [
            'status' => 'published',
            'published_at' => now()
        ]);
    }
}

// app/Services/CacheService.php
class CacheService
{
    public function remember($key, $ttl, $callback)
    {
        return Cache::remember($key, $ttl, $callback);
    }
    
    public function forget($key)
    {
        return Cache::forget($key);
    }
    
    public function flush($tags = null)
    {
        if ($tags) {
            return Cache::tags($tags)->flush();
        }
        
        return Cache::flush();
    }
}
```

---

## Method Injection

### ১. Route Method Injection:
```php
<?php
class PostController extends Controller
{
    // Method-level injection
    public function store(Request $request, PostService $postService)
    {
        $post = $postService->create($request->validated());
        
        return response()->json($post);
    }
    
    public function show(Post $post, ViewTracker $viewTracker)
    {
        $viewTracker->track($post);
        
        return response()->json($post);
    }
    
    public function sendNewsletter(EmailService $emailService, UserRepository $userRepository)
    {
        $users = $userRepository->getSubscribers();
        
        foreach ($users as $user) {
            $emailService->send($user->email, 'Newsletter', 'Latest updates...');
        }
        
        return response()->json(['message' => 'Newsletter sent']);
    }
}
```

### ২. Closure Injection:
```php
<?php
// Routes with dependency injection
Route::get('/posts', function (PostService $postService) {
    return $postService->getRecent();
});

Route::post('/posts', function (Request $request, PostService $postService) {
    return $postService->create($request->all());
});

Route::get('/stats', function (AnalyticsService $analytics, CacheService $cache) {
    return $cache->remember('site_stats', 3600, function () use ($analytics) {
        return $analytics->getSiteStats();
    });
});
```

---

## Service Container

### ১. Manual Binding:
```php
<?php
// app/Providers/AppServiceProvider.php

class AppServiceProvider extends ServiceProvider
{
    public function register()
    {
        // Simple binding
        $this->app->bind('App\Services\PaymentService', function ($app) {
            return new PaymentService($app['config']['payment.gateway']);
        });
        
        // Singleton binding
        $this->app->singleton('App\Services\CacheService', function ($app) {
            return new CacheService($app['cache']);
        });
        
        // Bind with parameters
        $this->app->bind('App\Services\EmailService', function ($app) {
            return new EmailService(
                $app['config']['mail.driver'],
                $app['config']['mail.from']
            );
        });
        
        // Contextual binding
        $this->app->when('App\Http\Controllers\AdminController')
                  ->needs('App\Services\LoggerInterface')
                  ->give('App\Services\AdminLogger');
                  
        $this->app->when('App\Http\Controllers\UserController')
                  ->needs('App\Services\LoggerInterface')
                  ->give('App\Services\UserLogger');
    }
}
```

### ২. Interface Binding:
```php
<?php
// Interface
interface PaymentGatewayInterface
{
    public function charge($amount);
    public function refund($transactionId);
}

// Implementations
class StripePaymentGateway implements PaymentGatewayInterface
{
    public function charge($amount)
    {
        // Stripe implementation
        return Stripe::charge($amount);
    }
    
    public function refund($transactionId)
    {
        return Stripe::refund($transactionId);
    }
}

class PaypalPaymentGateway implements PaymentGatewayInterface
{
    public function charge($amount)
    {
        // PayPal implementation
        return PayPal::charge($amount);
    }
    
    public function refund($transactionId)
    {
        return PayPal::refund($transactionId);
    }
}

// Binding in Service Provider
public function register()
{
    $this->app->bind(PaymentGatewayInterface::class, function ($app) {
        $gateway = config('payment.default_gateway');
        
        switch ($gateway) {
            case 'stripe':
                return new StripePaymentGateway();
            case 'paypal':
                return new PaypalPaymentGateway();
            default:
                throw new Exception("Unsupported payment gateway: {$gateway}");
        }
    });
}

// Usage in Controller
class PaymentController extends Controller
{
    protected $paymentGateway;
    
    public function __construct(PaymentGatewayInterface $paymentGateway)
    {
        $this->paymentGateway = $paymentGateway;
    }
    
    public function charge(Request $request)
    {
        $result = $this->paymentGateway->charge($request->amount);
        
        return response()->json($result);
    }
}
```

---

## Route Model Binding

### ১. Implicit Model Binding:
```php
<?php
// Routes
Route::get('/posts/{post}', [PostController::class, 'show']);
Route::put('/posts/{post}', [PostController::class, 'update']);
Route::delete('/posts/{post}', [PostController::class, 'destroy']);

// Controller
class PostController extends Controller
{
    // Laravel automatically injects Post model
    public function show(Post $post)
    {
        return response()->json($post);
    }
    
    public function update(Request $request, Post $post)
    {
        $post->update($request->validated());
        
        return response()->json($post);
    }
    
    public function destroy(Post $post)
    {
        $post->delete();
        
        return response()->json(['message' => 'Post deleted']);
    }
}
```

### ২. Custom Key Binding:
```php
<?php
// Model with custom route key
class Post extends Model
{
    public function getRouteKeyName()
    {
        return 'slug'; // Use slug instead of id
    }
}

// Or in routes
Route::get('/posts/{post:slug}', [PostController::class, 'show']);
Route::get('/users/{user:email}', [UserController::class, 'show']);

// Controller remains the same
public function show(Post $post)
{
    // $post is loaded by slug
    return response()->json($post);
}
```

### ৩. Explicit Model Binding:
```php
<?php
// app/Providers/RouteServiceProvider.php

public function boot()
{
    parent::boot();
    
    // Custom model binding
    Route::bind('post', function ($value) {
        return Post::where('slug', $value)->firstOrFail();
    });
    
    // Binding with additional conditions
    Route::bind('published_post', function ($value) {
        return Post::where('slug', $value)
                   ->where('status', 'published')
                   ->firstOrFail();
    });
    
    // Binding with relationships
    Route::bind('user_post', function ($value) {
        return Post::with(['user', 'comments'])
                   ->where('slug', $value)
                   ->firstOrFail();
    });
}

// Usage in routes
Route::get('/posts/{post}', [PostController::class, 'show']);
Route::get('/blog/{published_post}', [BlogController::class, 'show']);
```

### ৪. Nested Model Binding:
```php
<?php
// Routes
Route::get('/users/{user}/posts/{post}', [PostController::class, 'show']);

// Controller
public function show(User $user, Post $post)
{
    // Ensure post belongs to user
    if ($post->user_id !== $user->id) {
        abort(404);
    }
    
    return response()->json($post);
}

// Or use scoped binding
Route::get('/users/{user}/posts/{post:slug}', [PostController::class, 'show'])
     ->scopeBindings();

// Model relationship
class Post extends Model
{
    public function resolveRouteBinding($value, $field = null)
    {
        return $this->where($field ?? $this->getRouteKeyName(), $value)
                    ->where('user_id', request()->route('user')->id)
                    ->firstOrFail();
    }
}
```

---

## Interface Binding

### ১. Repository Pattern:
```php
<?php
// Repository Interface
interface PostRepositoryInterface
{
    public function all();
    public function find($id);
    public function create(array $data);
    public function update($id, array $data);
    public function delete($id);
}

// Eloquent Implementation
class EloquentPostRepository implements PostRepositoryInterface
{
    protected $model;
    
    public function __construct(Post $model)
    {
        $this->model = $model;
    }
    
    public function all()
    {
        return $this->model->all();
    }
    
    public function find($id)
    {
        return $this->model->findOrFail($id);
    }
    
    public function create(array $data)
    {
        return $this->model->create($data);
    }
    
    public function update($id, array $data)
    {
        $post = $this->find($id);
        $post->update($data);
        return $post;
    }
    
    public function delete($id)
    {
        return $this->find($id)->delete();
    }
}

// Cache Implementation
class CachedPostRepository implements PostRepositoryInterface
{
    protected $repository;
    protected $cache;
    
    public function __construct(PostRepositoryInterface $repository, CacheManager $cache)
    {
        $this->repository = $repository;
        $this->cache = $cache;
    }
    
    public function all()
    {
        return $this->cache->remember('posts.all', 3600, function () {
            return $this->repository->all();
        });
    }
    
    public function find($id)
    {
        return $this->cache->remember("posts.{$id}", 3600, function () use ($id) {
            return $this->repository->find($id);
        });
    }
    
    public function create(array $data)
    {
        $post = $this->repository->create($data);
        $this->cache->forget('posts.all');
        return $post;
    }
    
    public function update($id, array $data)
    {
        $post = $this->repository->update($id, $data);
        $this->cache->forget("posts.{$id}");
        $this->cache->forget('posts.all');
        return $post;
    }
    
    public function delete($id)
    {
        $result = $this->repository->delete($id);
        $this->cache->forget("posts.{$id}");
        $this->cache->forget('posts.all');
        return $result;
    }
}

// Service Provider Binding
public function register()
{
    $this->app->bind(PostRepositoryInterface::class, EloquentPostRepository::class);
    
    // Or with caching
    $this->app->bind(PostRepositoryInterface::class, function ($app) {
        $eloquentRepo = new EloquentPostRepository($app->make(Post::class));
        return new CachedPostRepository($eloquentRepo, $app['cache']);
    });
}
```

---

## Advanced Techniques

### ১. Factory Pattern with DI:
```php
<?php
// Factory Interface
interface NotificationFactoryInterface
{
    public function create($type);
}

// Factory Implementation
class NotificationFactory implements NotificationFactoryInterface
{
    protected $app;
    
    public function __construct(Application $app)
    {
        $this->app = $app;
    }
    
    public function create($type)
    {
        switch ($type) {
            case 'email':
                return $this->app->make(EmailNotification::class);
            case 'sms':
                return $this->app->make(SMSNotification::class);
            case 'push':
                return $this->app->make(PushNotification::class);
            default:
                throw new InvalidArgumentException("Unknown notification type: {$type}");
        }
    }
}

// Usage
class NotificationService
{
    protected $factory;
    
    public function __construct(NotificationFactoryInterface $factory)
    {
        $this->factory = $factory;
    }
    
    public function send($type, $recipient, $message)
    {
        $notification = $this->factory->create($type);
        return $notification->send($recipient, $message);
    }
}
```

### ২. Decorator Pattern:
```php
<?php
// Base Service
class BaseEmailService implements EmailServiceInterface
{
    public function send($to, $subject, $body)
    {
        // Basic email sending
        return Mail::to($to)->send(new GenericMail($subject, $body));
    }
}

// Logging Decorator
class LoggingEmailService implements EmailServiceInterface
{
    protected $emailService;
    protected $logger;
    
    public function __construct(EmailServiceInterface $emailService, LoggerInterface $logger)
    {
        $this->emailService = $emailService;
        $this->logger = $logger;
    }
    
    public function send($to, $subject, $body)
    {
        $this->logger->info("Sending email to: {$to}");
        
        $result = $this->emailService->send($to, $subject, $body);
        
        $this->logger->info("Email sent successfully to: {$to}");
        
        return $result;
    }
}

// Queue Decorator
class QueuedEmailService implements EmailServiceInterface
{
    protected $emailService;
    
    public function __construct(EmailServiceInterface $emailService)
    {
        $this->emailService = $emailService;
    }
    
    public function send($to, $subject, $body)
    {
        dispatch(new SendEmailJob($to, $subject, $body));
    }
}

// Binding with decorators
public function register()
{
    $this->app->bind(EmailServiceInterface::class, function ($app) {
        $baseService = new BaseEmailService();
        $loggedService = new LoggingEmailService($baseService, $app['log']);
        return new QueuedEmailService($loggedService);
    });
}
```

### ৩. Conditional Binding:
```php
<?php
public function register()
{
    // Environment-based binding
    if ($this->app->environment('production')) {
        $this->app->bind(LoggerInterface::class, ProductionLogger::class);
    } else {
        $this->app->bind(LoggerInterface::class, DevelopmentLogger::class);
    }
    
    // Feature flag binding
    if (config('features.use_redis_cache')) {
        $this->app->bind(CacheInterface::class, RedisCache::class);
    } else {
        $this->app->bind(CacheInterface::class, FileCache::class);
    }
    
    // User-based binding
    $this->app->bind(PaymentGatewayInterface::class, function ($app) {
        $user = $app['auth']->user();
        
        if ($user && $user->hasFeature('premium_payment')) {
            return new PremiumPaymentGateway();
        }
        
        return new StandardPaymentGateway();
    });
}
```

---

## Real-world Examples

### ১. E-commerce Order Processing:
```php
<?php
// Services
interface PaymentProcessorInterface
{
    public function process(Order $order);
}

interface InventoryManagerInterface
{
    public function reserve(Order $order);
    public function release(Order $order);
}

interface NotificationServiceInterface
{
    public function sendOrderConfirmation(Order $order);
}

// Order Service
class OrderService
{
    protected $paymentProcessor;
    protected $inventoryManager;
    protected $notificationService;
    
    public function __construct(
        PaymentProcessorInterface $paymentProcessor,
        InventoryManagerInterface $inventoryManager,
        NotificationServiceInterface $notificationService
    ) {
        $this->paymentProcessor = $paymentProcessor;
        $this->inventoryManager = $inventoryManager;
        $this->notificationService = $notificationService;
    }
    
    public function processOrder(array $orderData)
    {
        DB::beginTransaction();
        
        try {
            // Create order
            $order = Order::create($orderData);
            
            // Reserve inventory
            $this->inventoryManager->reserve($order);
            
            // Process payment
            $paymentResult = $this->paymentProcessor->process($order);
            
            if ($paymentResult->successful()) {
                $order->update(['status' => 'paid']);
                
                // Send confirmation
                $this->notificationService->sendOrderConfirmation($order);
                
                DB::commit();
                return $order;
            } else {
                // Release inventory on payment failure
                $this->inventoryManager->release($order);
                DB::rollback();
                
                throw new PaymentException('Payment failed');
            }
        } catch (Exception $e) {
            DB::rollback();
            throw $e;
        }
    }
}

// Controller
class OrderController extends Controller
{
    protected $orderService;
    
    public function __construct(OrderService $orderService)
    {
        $this->orderService = $orderService;
    }
    
    public function store(Request $request)
    {
        try {
            $order = $this->orderService->processOrder($request->validated());
            return response()->json($order, 201);
        } catch (PaymentException $e) {
            return response()->json(['error' => $e->getMessage()], 400);
        }
    }
}
```

### ২. Multi-tenant Application:
```php
<?php
// Tenant-aware services
interface TenantRepositoryInterface
{
    public function setTenant(Tenant $tenant);
    public function all();
    public function find($id);
}

class TenantAwarePostRepository implements TenantRepositoryInterface
{
    protected $tenant;
    
    public function setTenant(Tenant $tenant)
    {
        $this->tenant = $tenant;
    }
    
    public function all()
    {
        return Post::where('tenant_id', $this->tenant->id)->get();
    }
    
    public function find($id)
    {
        return Post::where('tenant_id', $this->tenant->id)
                   ->where('id', $id)
                   ->firstOrFail();
    }
}

// Middleware to set tenant
class SetTenantMiddleware
{
    protected $repository;
    
    public function __construct(TenantRepositoryInterface $repository)
    {
        $this->repository = $repository;
    }
    
    public function handle($request, Closure $next)
    {
        $tenant = Tenant::where('domain', $request->getHost())->firstOrFail();
        
        $this->repository->setTenant($tenant);
        
        return $next($request);
    }
}

// Controller
class PostController extends Controller
{
    protected $repository;
    
    public function __construct(TenantRepositoryInterface $repository)
    {
        $this->repository = $repository;
    }
    
    public function index()
    {
        // Automatically filtered by tenant
        return $this->repository->all();
    }
}
```

### ৩. Testing with Dependency Injection:
```php
<?php
// Test case
class OrderServiceTest extends TestCase
{
    public function test_order_processing_success()
    {
        // Mock dependencies
        $paymentProcessor = Mockery::mock(PaymentProcessorInterface::class);
        $inventoryManager = Mockery::mock(InventoryManagerInterface::class);
        $notificationService = Mockery::mock(NotificationServiceInterface::class);
        
        // Set expectations
        $paymentProcessor->shouldReceive('process')
                        ->once()
                        ->andReturn(new SuccessfulPaymentResult());
                        
        $inventoryManager->shouldReceive('reserve')->once();
        $notificationService->shouldReceive('sendOrderConfirmation')->once();
        
        // Create service with mocked dependencies
        $orderService = new OrderService(
            $paymentProcessor,
            $inventoryManager,
            $notificationService
        );
        
        // Test
        $order = $orderService->processOrder([
            'customer_id' => 1,
            'total' => 100
        ]);
        
        $this->assertEquals('paid', $order->status);
    }
    
    public function test_order_processing_payment_failure()
    {
        $paymentProcessor = Mockery::mock(PaymentProcessorInterface::class);
        $inventoryManager = Mockery::mock(InventoryManagerInterface::class);
        $notificationService = Mockery::mock(NotificationServiceInterface::class);
        
        $paymentProcessor->shouldReceive('process')
                        ->once()
                        ->andReturn(new FailedPaymentResult());
                        
        $inventoryManager->shouldReceive('reserve')->once();
        $inventoryManager->shouldReceive('release')->once();
        $notificationService->shouldNotReceive('sendOrderConfirmation');
        
        $orderService = new OrderService(
            $paymentProcessor,
            $inventoryManager,
            $notificationService
        );
        
        $this->expectException(PaymentException::class);
        
        $orderService->processOrder([
            'customer_id' => 1,
            'total' => 100
        ]);
    }
}
```

---

## 🎯 Best Practices:

### ✅ **Design Principles:**
- **Depend on abstractions**, not concretions
- Use **interfaces** for better testability
- Keep **constructor parameters** minimal
- Follow **Single Responsibility Principle**

### ✅ **Performance:**
- Use **singletons** for expensive objects
- Avoid **circular dependencies**
- Consider **lazy loading** for heavy services
- Use **contextual binding** when needed

### ✅ **Testing:**
- **Mock dependencies** in unit tests
- Use **dependency injection** for better test isolation
- Create **test doubles** for external services
- Test **different scenarios** with different implementations

### ✅ **Organization:**
- Group related **services** in folders
- Use **service providers** for binding
- Document **dependencies** clearly
- Follow **naming conventions**

---

## 📚 আরও জানতে:
- [Laravel Service Container](https://laravel.com/docs/container)
- [Dependency Injection](https://laravel.com/docs/providers)
- [Route Model Binding](https://laravel.com/docs/routing#route-model-binding)