# Laravel API Caching - সম্পূর্ণ বাংলা গাইড

## 📋 সূচিপত্র
- [API Caching কি?](#api-caching-কি)
- [কেন API Caching প্রয়োজন?](#কেন-api-caching-প্রয়োজন)
- [Laravel এ Caching Types](#laravel-এ-caching-types)
- [কখন কোন Cache ব্যবহার করবেন?](#কখন-কোন-cache-ব্যবহার-করবেন)
- [Basic API Caching](#basic-api-caching)
- [Advanced Caching Strategies](#advanced-caching-strategies)
- [Cache Invalidation](#cache-invalidation)
- [Performance Optimization](#performance-optimization)
- [Best Practices](#best-practices)

---

## API Caching কি?

**API Caching** হলো **frequently requested data** কে **temporary storage** এ রেখে **faster response** প্রদান করার একটি technique। এটি **database queries** কমিয়ে **API performance** বৃদ্ধি করে।

### 🎯 সহজ ভাষায়:
- **Expensive operations** এর result store করা
- **Repeated requests** এর জন্য **cached data** return করা
- **Database load** কমানো এবং **response time** কমানো

### 📊 Without vs With Caching:

```php
// ❌ Without Caching (Slow)
class UserController extends Controller
{
    public function index()
    {
        // Every request hits database
        $users = User::with(['posts', 'profile'])
                    ->withCount(['posts', 'comments'])
                    ->orderBy('created_at', 'desc')
                    ->paginate(20);
        
        return UserResource::collection($users);
    }
}
// Response Time: 500ms, Database Queries: 15+
```

```php
// ✅ With Caching (Fast)
class UserController extends Controller
{
    public function index()
    {
        $cacheKey = 'users.index.page.' . request('page', 1);
        
        $users = Cache::remember($cacheKey, 300, function () {
            return User::with(['posts', 'profile'])
                      ->withCount(['posts', 'comments'])
                      ->orderBy('created_at', 'desc')
                      ->paginate(20);
        });
        
        return UserResource::collection($users);
    }
}
// Response Time: 50ms, Database Queries: 0 (from cache)
```

---

## কেন API Caching প্রয়োজন?

### 🚀 **Performance Benefits:**

#### 1. **Faster Response Time**
```php
// Database query: 200ms
// Cache retrieval: 5ms
// 40x faster response!

$stats = Cache::remember('dashboard.stats', 600, function () {
    return [
        'total_users' => User::count(),
        'active_users' => User::where('last_login_at', '>', now()->subDays(30))->count(),
        'total_posts' => Post::count(),
        'popular_posts' => Post::withCount('views')->orderBy('views_count', 'desc')->take(10)->get(),
    ];
});
```

#### 2. **Reduced Database Load**
```php
// Without cache: 1000 requests = 1000 database queries
// With cache: 1000 requests = 1 database query + 999 cache hits

class ProductController extends Controller
{
    public function popular()
    {
        return Cache::remember('products.popular', 3600, function () {
            return Product::with(['category', 'reviews'])
                         ->withAvg('reviews', 'rating')
                         ->withCount('orders')
                         ->orderBy('orders_count', 'desc')
                         ->take(20)
                         ->get();
        });
    }
}
```

#### 3. **Better User Experience**
```php
// Mobile API responses
class MobileApiController extends Controller
{
    public function dashboard()
    {
        $userId = auth()->id();
        
        $dashboard = Cache::remember("user.{$userId}.dashboard", 300, function () use ($userId) {
            return [
                'profile' => User::with('profile')->find($userId),
                'recent_orders' => Order::where('user_id', $userId)->latest()->take(5)->get(),
                'notifications' => Notification::where('user_id', $userId)->unread()->take(10)->get(),
                'recommendations' => $this->getRecommendations($userId),
            ];
        });
        
        return response()->json($dashboard);
    }
}
```

#### 4. **Cost Reduction**
```php
// Expensive external API calls
class WeatherController extends Controller
{
    public function current($city)
    {
        $cacheKey = "weather.{$city}";
        
        return Cache::remember($cacheKey, 1800, function () use ($city) {
            // Expensive external API call
            $response = Http::get("https://api.weather.com/current/{$city}");
            return $response->json();
        });
    }
}
```

---

## Laravel এ Caching Types

### 1. **File Cache (Default)**
```php
// config/cache.php
'default' => 'file',

'stores' => [
    'file' => [
        'driver' => 'file',
        'path' => storage_path('framework/cache/data'),
    ],
],

// Usage
Cache::put('key', 'value', 600); // 10 minutes
$value = Cache::get('key');
```

### 2. **Redis Cache (Recommended for APIs)**
```php
// config/cache.php
'default' => 'redis',

'stores' => [
    'redis' => [
        'driver' => 'redis',
        'connection' => 'cache',
        'lock_connection' => 'default',
    ],
],

// config/database.php
'redis' => [
    'cache' => [
        'url' => env('REDIS_URL'),
        'host' => env('REDIS_HOST', '127.0.0.1'),
        'password' => env('REDIS_PASSWORD'),
        'port' => env('REDIS_PORT', '6379'),
        'database' => env('REDIS_CACHE_DB', '1'),
    ],
],
```

### 3. **Memcached**
```php
// config/cache.php
'stores' => [
    'memcached' => [
        'driver' => 'memcached',
        'persistent_id' => env('MEMCACHED_PERSISTENT_ID'),
        'sasl' => [
            env('MEMCACHED_USERNAME'),
            env('MEMCACHED_PASSWORD'),
        ],
        'options' => [
            // Memcached::OPT_CONNECT_TIMEOUT => 2000,
        ],
        'servers' => [
            [
                'host' => env('MEMCACHED_HOST', '127.0.0.1'),
                'port' => env('MEMCACHED_PORT', 11211),
                'weight' => 100,
            ],
        ],
    ],
],
```

### 4. **Database Cache**
```php
// Create cache table
php artisan cache:table
php artisan migrate

// config/cache.php
'stores' => [
    'database' => [
        'driver' => 'database',
        'table' => 'cache',
        'connection' => null,
        'lock_connection' => null,
    ],
],
```

### 5. **Array Cache (Testing)**
```php
// config/cache.php
'stores' => [
    'array' => [
        'driver' => 'array',
        'serialize' => false,
    ],
],

// Testing environment
// config/cache.php (testing)
'default' => 'array',
```

---

## কখন কোন Cache ব্যবহার করবেন?

### 📊 **Cache Comparison:**

| Cache Type | Speed | Persistence | Scalability | Use Case |
|------------|-------|-------------|-------------|----------|
| **Redis** | ⭐⭐⭐⭐⭐ | ✅ | ⭐⭐⭐⭐⭐ | Production APIs |
| **Memcached** | ⭐⭐⭐⭐⭐ | ❌ | ⭐⭐⭐⭐ | High-traffic sites |
| **File** | ⭐⭐⭐ | ✅ | ⭐⭐ | Small applications |
| **Database** | ⭐⭐ | ✅ | ⭐⭐⭐ | Shared hosting |
| **Array** | ⭐⭐⭐⭐⭐ | ❌ | ⭐ | Testing only |

### 🎯 **Recommendations:**

#### **Production APIs → Redis**
```php
// High performance, persistence, clustering support
// Perfect for API caching

class ApiController extends Controller
{
    public function products()
    {
        return Cache::store('redis')->remember('api.products', 3600, function () {
            return Product::with(['category', 'images'])->active()->get();
        });
    }
}
```

#### **Development → File Cache**
```php
// Simple setup, no additional services required
// Good for local development

Cache::put('user.1', $user, 600);
```

#### **Testing → Array Cache**
```php
// Fast, no persistence needed
// Perfect for unit tests

// phpunit.xml
<env name="CACHE_DRIVER" value="array"/>
```

---

## Basic API Caching

### 1. **Simple Cache Operations**
```php
class UserController extends Controller
{
    public function show($id)
    {
        // Cache::put() - Store data
        Cache::put("user.{$id}", $user, 600); // 10 minutes
        
        // Cache::get() - Retrieve data
        $user = Cache::get("user.{$id}");
        
        // Cache::remember() - Get or store
        $user = Cache::remember("user.{$id}", 600, function () use ($id) {
            return User::with(['posts', 'profile'])->findOrFail($id);
        });
        
        return new UserResource($user);
    }
}
```

### 2. **Cache with Default Values**
```php
class ProductController extends Controller
{
    public function index()
    {
        // Cache::get() with default
        $products = Cache::get('products.featured', collect());
        
        // Cache::remember() with complex default
        $categories = Cache::remember('categories.tree', 3600, function () {
            return Category::with('children')->whereNull('parent_id')->get();
        });
        
        return response()->json([
            'products' => $products,
            'categories' => $categories,
        ]);
    }
}
```

### 3. **Conditional Caching**
```php
class NewsController extends Controller
{
    public function latest()
    {
        $cacheKey = 'news.latest';
        $cacheDuration = config('app.env') === 'production' ? 1800 : 60;
        
        $news = Cache::remember($cacheKey, $cacheDuration, function () {
            return News::with(['author', 'category'])
                      ->published()
                      ->latest()
                      ->take(20)
                      ->get();
        });
        
        return NewsResource::collection($news);
    }
}
```

### 4. **User-Specific Caching**
```php
class DashboardController extends Controller
{
    public function index()
    {
        $userId = auth()->id();
        $cacheKey = "dashboard.user.{$userId}";
        
        $dashboard = Cache::remember($cacheKey, 300, function () use ($userId) {
            return [
                'stats' => $this->getUserStats($userId),
                'recent_activity' => $this->getRecentActivity($userId),
                'notifications' => $this->getNotifications($userId),
            ];
        });
        
        return response()->json($dashboard);
    }
}
```

---

## Advanced Caching Strategies

### 1. **Cache Tags (Redis/Memcached)**
```php
class PostController extends Controller
{
    public function index()
    {
        return Cache::tags(['posts', 'public'])->remember('posts.all', 3600, function () {
            return Post::with(['author', 'category'])->published()->get();
        });
    }
    
    public function store(Request $request)
    {
        $post = Post::create($request->validated());
        
        // Invalidate tagged cache
        Cache::tags(['posts', 'public'])->flush();
        
        return new PostResource($post);
    }
    
    public function userPosts($userId)
    {
        return Cache::tags(['posts', "user.{$userId}"])->remember(
            "posts.user.{$userId}", 
            1800, 
            function () use ($userId) {
                return Post::where('user_id', $userId)->with('category')->get();
            }
        );
    }
}
```

### 2. **Cache Hierarchies**
```php
class CategoryController extends Controller
{
    public function products($categoryId)
    {
        // Hierarchical cache keys
        $cacheKey = "category.{$categoryId}.products";
        
        $products = Cache::remember($cacheKey, 3600, function () use ($categoryId) {
            return Product::where('category_id', $categoryId)
                         ->with(['images', 'reviews'])
                         ->active()
                         ->get();
        });
        
        return ProductResource::collection($products);
    }
    
    public function tree()
    {
        return Cache::remember('categories.tree', 7200, function () {
            return $this->buildCategoryTree();
        });
    }
    
    private function buildCategoryTree()
    {
        $categories = Category::with('children')->get();
        return $this->formatTree($categories);
    }
}
```

### 3. **Cache Warming**
```php
// Artisan Command for cache warming
class WarmCacheCommand extends Command
{
    protected $signature = 'cache:warm';
    
    public function handle()
    {
        $this->info('Warming up cache...');
        
        // Warm popular data
        Cache::remember('products.popular', 3600, function () {
            return Product::withCount('orders')->orderBy('orders_count', 'desc')->take(50)->get();
        });
        
        Cache::remember('categories.all', 7200, function () {
            return Category::with('children')->get();
        });
        
        Cache::remember('settings.public', 86400, function () {
            return Setting::where('is_public', true)->pluck('value', 'key');
        });
        
        $this->info('Cache warmed successfully!');
    }
}

// Schedule cache warming
// app/Console/Kernel.php
protected function schedule(Schedule $schedule)
{
    $schedule->command('cache:warm')->hourly();
}
```

### 4. **Distributed Caching**
```php
class DistributedCacheService
{
    public function remember($key, $ttl, $callback)
    {
        // Try local cache first (faster)
        $localKey = "local.{$key}";
        if (Cache::store('array')->has($localKey)) {
            return Cache::store('array')->get($localKey);
        }
        
        // Try Redis cache
        $value = Cache::store('redis')->remember($key, $ttl, $callback);
        
        // Store in local cache for 60 seconds
        Cache::store('array')->put($localKey, $value, 60);
        
        return $value;
    }
}

// Usage
class ProductController extends Controller
{
    protected $cache;
    
    public function __construct(DistributedCacheService $cache)
    {
        $this->cache = $cache;
    }
    
    public function featured()
    {
        return $this->cache->remember('products.featured', 3600, function () {
            return Product::featured()->with('category')->get();
        });
    }
}
```

### 5. **Cache Serialization**
```php
class CacheService
{
    public function storeModel($key, $model, $ttl = 3600)
    {
        // Serialize Eloquent model
        $serialized = serialize($model);
        Cache::put($key, $serialized, $ttl);
    }
    
    public function getModel($key)
    {
        $serialized = Cache::get($key);
        return $serialized ? unserialize($serialized) : null;
    }
    
    public function storeCollection($key, $collection, $ttl = 3600)
    {
        // Store collection as array
        Cache::put($key, $collection->toArray(), $ttl);
    }
    
    public function getCollection($key, $modelClass)
    {
        $data = Cache::get($key);
        return $data ? $modelClass::hydrate($data) : collect();
    }
}
```

---

## Cache Invalidation

### 1. **Manual Cache Invalidation**
```php
class PostController extends Controller
{
    public function store(Request $request)
    {
        $post = Post::create($request->validated());
        
        // Clear specific caches
        Cache::forget('posts.latest');
        Cache::forget('posts.popular');
        Cache::forget("category.{$post->category_id}.posts");
        
        return new PostResource($post);
    }
    
    public function update(Request $request, Post $post)
    {
        $post->update($request->validated());
        
        // Clear post-specific cache
        Cache::forget("post.{$post->id}");
        Cache::forget('posts.latest');
        
        return new PostResource($post);
    }
}
```

### 2. **Event-Based Cache Invalidation**
```php
// Model Events
class Post extends Model
{
    protected static function booted()
    {
        static::created(function ($post) {
            Cache::tags(['posts'])->flush();
            Cache::forget('posts.latest');
        });
        
        static::updated(function ($post) {
            Cache::forget("post.{$post->id}");
            Cache::tags(['posts'])->flush();
        });
        
        static::deleted(function ($post) {
            Cache::forget("post.{$post->id}");
            Cache::tags(['posts'])->flush();
        });
    }
}

// Custom Events
class PostCreated
{
    public $post;
    
    public function __construct(Post $post)
    {
        $this->post = $post;
    }
}

// Event Listener
class ClearPostCache
{
    public function handle(PostCreated $event)
    {
        Cache::tags(['posts', 'public'])->flush();
        Cache::forget("category.{$event->post->category_id}.posts");
    }
}
```

### 3. **Time-Based Cache Invalidation**
```php
class CacheManager
{
    public function rememberWithTags($key, $tags, $ttl, $callback)
    {
        $timestampKey = "{$key}.timestamp";
        $dataKey = "{$key}.data";
        
        $timestamp = Cache::get($timestampKey);
        $now = now()->timestamp;
        
        if (!$timestamp || ($now - $timestamp) > $ttl) {
            $data = $callback();
            
            Cache::put($dataKey, $data, $ttl * 2); // Store longer
            Cache::put($timestampKey, $now, $ttl * 2);
            
            return $data;
        }
        
        return Cache::get($dataKey);
    }
}
```

### 4. **Cache Versioning**
```php
class VersionedCache
{
    private $version;
    
    public function __construct()
    {
        $this->version = config('app.cache_version', 1);
    }
    
    public function remember($key, $ttl, $callback)
    {
        $versionedKey = "v{$this->version}.{$key}";
        return Cache::remember($versionedKey, $ttl, $callback);
    }
    
    public function forget($key)
    {
        $versionedKey = "v{$this->version}.{$key}";
        return Cache::forget($versionedKey);
    }
    
    public function bumpVersion()
    {
        $newVersion = $this->version + 1;
        
        // Update config or database
        config(['app.cache_version' => $newVersion]);
        
        return $newVersion;
    }
}
```

---

## Performance Optimization

### 1. **Cache Preloading**
```php
class ApiController extends Controller
{
    public function dashboard()
    {
        // Preload multiple cache keys in parallel
        $keys = [
            'stats.users' => 'users.count',
            'stats.posts' => 'posts.count',
            'stats.orders' => 'orders.today',
            'popular.products' => 'products.popular',
        ];
        
        $data = [];
        
        foreach ($keys as $responseKey => $cacheKey) {
            $data[$responseKey] = Cache::get($cacheKey);
        }
        
        // Fill missing data
        if (!$data['stats.users']) {
            $data['stats.users'] = Cache::remember('users.count', 3600, function () {
                return User::count();
            });
        }
        
        return response()->json($data);
    }
}
```

### 2. **Cache Compression**
```php
class CompressedCache
{
    public function put($key, $value, $ttl)
    {
        $compressed = gzcompress(serialize($value));
        return Cache::put($key, $compressed, $ttl);
    }
    
    public function get($key)
    {
        $compressed = Cache::get($key);
        
        if (!$compressed) {
            return null;
        }
        
        $uncompressed = gzuncompress($compressed);
        return unserialize($uncompressed);
    }
    
    public function remember($key, $ttl, $callback)
    {
        $value = $this->get($key);
        
        if ($value !== null) {
            return $value;
        }
        
        $value = $callback();
        $this->put($key, $value, $ttl);
        
        return $value;
    }
}
```

### 3. **Cache Partitioning**
```php
class PartitionedCache
{
    public function getPartitionKey($key, $partitions = 10)
    {
        $hash = crc32($key);
        $partition = abs($hash) % $partitions;
        return "partition.{$partition}.{$key}";
    }
    
    public function remember($key, $ttl, $callback)
    {
        $partitionedKey = $this->getPartitionKey($key);
        return Cache::remember($partitionedKey, $ttl, $callback);
    }
    
    public function clearPartition($partition)
    {
        $pattern = "partition.{$partition}.*";
        
        // Redis-specific
        if (Cache::getStore() instanceof \Illuminate\Cache\RedisStore) {
            $redis = Cache::getStore()->getRedis();
            $keys = $redis->keys($pattern);
            
            if (!empty($keys)) {
                $redis->del($keys);
            }
        }
    }
}
```

### 4. **Cache Monitoring**
```php
class CacheMonitor
{
    public function remember($key, $ttl, $callback)
    {
        $startTime = microtime(true);
        
        $value = Cache::get($key);
        $hit = $value !== null;
        
        if (!$hit) {
            $value = $callback();
            Cache::put($key, $value, $ttl);
        }
        
        $endTime = microtime(true);
        $duration = ($endTime - $startTime) * 1000; // milliseconds
        
        // Log cache performance
        Log::info('Cache Access', [
            'key' => $key,
            'hit' => $hit,
            'duration_ms' => round($duration, 2),
            'ttl' => $ttl,
        ]);
        
        return $value;
    }
    
    public function getStats()
    {
        return [
            'hit_rate' => $this->calculateHitRate(),
            'memory_usage' => $this->getMemoryUsage(),
            'key_count' => $this->getKeyCount(),
        ];
    }
}
```

---

## Best Practices

### 1. **Cache Key Naming Convention**
```php
class CacheKeys
{
    // Hierarchical naming
    const USER_PROFILE = 'user.{id}.profile';
    const USER_POSTS = 'user.{id}.posts';
    const CATEGORY_PRODUCTS = 'category.{id}.products';
    
    // Environment-specific
    const GLOBAL_SETTINGS = 'app.{env}.settings';
    
    // Time-based
    const DAILY_STATS = 'stats.daily.{date}';
    const HOURLY_REPORTS = 'reports.hourly.{hour}';
    
    public static function userProfile($id)
    {
        return str_replace('{id}', $id, self::USER_PROFILE);
    }
    
    public static function categoryProducts($categoryId)
    {
        return str_replace('{id}', $categoryId, self::CATEGORY_PRODUCTS);
    }
}

// Usage
$user = Cache::remember(CacheKeys::userProfile($userId), 3600, function () use ($userId) {
    return User::with('profile')->find($userId);
});
```

### 2. **Cache TTL Strategy**
```php
class CacheTTL
{
    const SHORT = 300;      // 5 minutes - frequently changing data
    const MEDIUM = 1800;    // 30 minutes - moderately changing data
    const LONG = 3600;      // 1 hour - stable data
    const VERY_LONG = 86400; // 24 hours - rarely changing data
    
    public static function dynamic($dataType)
    {
        return match($dataType) {
            'user_activity' => self::SHORT,
            'product_list' => self::MEDIUM,
            'category_tree' => self::LONG,
            'site_settings' => self::VERY_LONG,
            default => self::MEDIUM,
        };
    }
}

// Usage
$products = Cache::remember('products.featured', CacheTTL::MEDIUM, function () {
    return Product::featured()->get();
});
```

### 3. **Cache Middleware**
```php
class CacheResponse
{
    public function handle($request, Closure $next, $ttl = 300)
    {
        // Generate cache key from request
        $key = $this->generateCacheKey($request);
        
        // Check if response is cached
        if (Cache::has($key)) {
            $cachedResponse = Cache::get($key);
            
            return response($cachedResponse['content'])
                ->withHeaders($cachedResponse['headers'])
                ->header('X-Cache', 'HIT');
        }
        
        // Process request
        $response = $next($request);
        
        // Cache successful responses
        if ($response->getStatusCode() === 200) {
            Cache::put($key, [
                'content' => $response->getContent(),
                'headers' => $response->headers->all(),
            ], $ttl);
        }
        
        return $response->header('X-Cache', 'MISS');
    }
    
    private function generateCacheKey($request)
    {
        $url = $request->fullUrl();
        $user = $request->user();
        $userId = $user ? $user->id : 'guest';
        
        return 'response.' . md5($url . $userId);
    }
}

// Register middleware
// app/Http/Kernel.php
protected $routeMiddleware = [
    'cache.response' => \App\Http\Middleware\CacheResponse::class,
];

// Use in routes
Route::get('/api/products', [ProductController::class, 'index'])
     ->middleware('cache.response:600'); // 10 minutes
```

### 4. **Cache Configuration**
```php
// config/cache.php
return [
    'default' => env('CACHE_DRIVER', 'redis'),
    
    'stores' => [
        'redis' => [
            'driver' => 'redis',
            'connection' => 'cache',
            'lock_connection' => 'default',
        ],
        
        'redis_sessions' => [
            'driver' => 'redis',
            'connection' => 'sessions',
        ],
        
        'file' => [
            'driver' => 'file',
            'path' => storage_path('framework/cache/data'),
        ],
    ],
    
    'prefix' => env('CACHE_PREFIX', Str::slug(env('APP_NAME', 'laravel'), '_').'_cache'),
];

// Environment-specific settings
// .env.production
CACHE_DRIVER=redis
REDIS_HOST=redis-cluster.example.com
REDIS_PASSWORD=secure_password

// .env.local
CACHE_DRIVER=file
```

### 5. **Error Handling**
```php
class SafeCache
{
    public function remember($key, $ttl, $callback)
    {
        try {
            return Cache::remember($key, $ttl, $callback);
        } catch (\Exception $e) {
            // Log cache error
            Log::warning('Cache operation failed', [
                'key' => $key,
                'error' => $e->getMessage(),
            ]);
            
            // Fallback to direct execution
            return $callback();
        }
    }
    
    public function get($key, $default = null)
    {
        try {
            return Cache::get($key, $default);
        } catch (\Exception $e) {
            Log::warning('Cache get failed', [
                'key' => $key,
                'error' => $e->getMessage(),
            ]);
            
            return $default;
        }
    }
    
    public function put($key, $value, $ttl)
    {
        try {
            return Cache::put($key, $value, $ttl);
        } catch (\Exception $e) {
            Log::warning('Cache put failed', [
                'key' => $key,
                'error' => $e->getMessage(),
            ]);
            
            return false;
        }
    }
}
```

### 6. **Testing Cache**
```php
// tests/Feature/CacheTest.php
class CacheTest extends TestCase
{
    public function test_api_response_is_cached()
    {
        // Clear cache
        Cache::flush();
        
        // First request - should hit database
        $response1 = $this->get('/api/products');
        $response1->assertStatus(200);
        $response1->assertHeader('X-Cache', 'MISS');
        
        // Second request - should hit cache
        $response2 = $this->get('/api/products');
        $response2->assertStatus(200);
        $response2->assertHeader('X-Cache', 'HIT');
        
        // Responses should be identical
        $this->assertEquals(
            $response1->getContent(),
            $response2->getContent()
        );
    }
    
    public function test_cache_invalidation_on_model_update()
    {
        $product = Product::factory()->create();
        
        // Cache product
        Cache::put("product.{$product->id}", $product, 3600);
        
        // Update product
        $product->update(['name' => 'Updated Name']);
        
        // Cache should be cleared
        $this->assertNull(Cache::get("product.{$product->id}"));
    }
}
```

---

## সারসংক্ষেপ

### 🎯 **কখন Cache ব্যবহার করবেন:**

**✅ ব্যবহার করুন:**
- Expensive database queries
- External API calls
- Complex calculations
- Frequently accessed data
- High-traffic APIs

**❌ ব্যবহার করবেন না:**
- Frequently changing data
- User-specific sensitive data
- Real-time data requirements
- Simple queries

### 🚀 **Performance Impact:**

| Scenario | Without Cache | With Cache | Improvement |
|----------|---------------|------------|-------------|
| Database Query | 200ms | 5ms | 40x faster |
| External API | 1000ms | 5ms | 200x faster |
| Complex Calculation | 500ms | 2ms | 250x faster |

### 🛠️ **Implementation Checklist:**

1. ✅ Choose appropriate cache driver (Redis for production)
2. ✅ Design cache key naming convention
3. ✅ Set appropriate TTL values
4. ✅ Implement cache invalidation strategy
5. ✅ Add cache monitoring and logging
6. ✅ Handle cache failures gracefully
7. ✅ Write cache-related tests
8. ✅ Monitor cache hit rates

### 📊 **Best Practices Summary:**
- **Redis** for production APIs
- **Hierarchical cache keys** for organization
- **Event-based invalidation** for data consistency
- **Cache tags** for bulk invalidation
- **Monitoring** for performance optimization
- **Error handling** for reliability
- **Testing** for quality assurance

Laravel API Caching সঠিকভাবে implement করলে আপনার API **10x-100x faster** হতে পারে এবং **database load** উল্লেখযোগ্যভাবে কমবে।