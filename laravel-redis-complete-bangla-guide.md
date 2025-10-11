# Laravel Redis - সম্পূর্ণ বাংলা গাইড (নতুনদের জন্য)

## 📋 সূচিপত্র
- [Redis কি এবং কেন ব্যবহার করবেন](#redis-কি-এবং-কেন-ব্যবহার-করবেন)
- [Laravel এ Redis Setup](#laravel-এ-redis-setup)
- [Redis for Queue - বিস্তারিত](#redis-for-queue---বিস্তারিত)
- [Redis for Caching](#redis-for-caching)
- [Redis for Session](#redis-for-session)
- [Redis Commands ও Operations](#redis-commands-ও-operations)
- [Production এ Redis](#production-এ-redis)
- [সমস্যা ও সমাধান](#সমস্যা-ও-সমাধান)

---

## Redis কি এবং কেন ব্যবহার করবেন

### 🚀 **Redis কি?**

Redis (Remote Dictionary Server) হলো একটি **in-memory data structure store** যা database, cache, এবং message broker হিসেবে কাজ করে।

### 🎯 **কেন Laravel এ Redis ব্যবহার করবেন?**

**Performance Benefits:**
- ✅ **Super Fast:** Memory-based, disk এর চেয়ে 100x দ্রুত
- ✅ **Queue Processing:** Background jobs efficiently handle করে
- ✅ **Caching:** Database load কমায়
- ✅ **Session Storage:** Fast session management
- ✅ **Real-time Features:** Broadcasting, WebSockets

**Laravel এ Redis Use Cases:**
1. **Queue Driver** - Background jobs
2. **Cache Driver** - Application caching
3. **Session Driver** - User sessions
4. **Broadcasting** - Real-time notifications
5. **Rate Limiting** - API throttling

### 💾 **Redis vs Database Queue:**

| Feature | Database Queue | Redis Queue |
|---------|---------------|-------------|
| **Speed** | Slow (disk I/O) | Very Fast (memory) |
| **Reliability** | High (persistent) | Medium (can lose data) |
| **Scalability** | Limited | Excellent |
| **Memory Usage** | Low | High |
| **Best For** | Small projects | High-traffic apps |

---

## Laravel এ Redis Setup

### 1️⃣ **Redis Installation**

**Ubuntu/Debian:**
```bash
# Redis server install
sudo apt update
sudo apt install redis-server

# Redis start
sudo systemctl start redis-server
sudo systemctl enable redis-server

# Test Redis
redis-cli ping
# Response: PONG
```

**macOS (Homebrew):**
```bash
# Install Redis
brew install redis

# Start Redis
brew services start redis

# Test
redis-cli ping
```

**Windows (WSL recommended):**
```bash
# Use Ubuntu WSL and follow Ubuntu steps
```

### 2️⃣ **Laravel Redis Package**

```bash
# Predis package install (recommended)
composer require predis/predis

# Or PhpRedis extension (faster but needs compilation)
# sudo pecl install redis
```

### 3️⃣ **Environment Configuration**

**.env file:**
```env
# Redis Configuration
REDIS_HOST=127.0.0.1
REDIS_PASSWORD=null
REDIS_PORT=6379
REDIS_DB=0

# Queue Configuration
QUEUE_CONNECTION=redis

# Cache Configuration
CACHE_DRIVER=redis

# Session Configuration
SESSION_DRIVER=redis
SESSION_LIFETIME=120

# Broadcasting (optional)
BROADCAST_DRIVER=redis
```

### 4️⃣ **Config Files Check**

**config/database.php:**
```php
'redis' => [
    'client' => env('REDIS_CLIENT', 'predis'),
    
    'options' => [
        'cluster' => env('REDIS_CLUSTER', 'redis'),
        'prefix' => env('REDIS_PREFIX', Str::slug(env('APP_NAME', 'laravel'), '_').'_database_'),
    ],

    'default' => [
        'url' => env('REDIS_URL'),
        'host' => env('REDIS_HOST', '127.0.0.1'),
        'password' => env('REDIS_PASSWORD'),
        'port' => env('REDIS_PORT', '6379'),
        'database' => env('REDIS_DB', '0'),
    ],

    'cache' => [
        'url' => env('REDIS_URL'),
        'host' => env('REDIS_HOST', '127.0.0.1'),
        'password' => env('REDIS_PASSWORD'),
        'port' => env('REDIS_PORT', '6379'),
        'database' => env('REDIS_CACHE_DB', '1'),
    ],
],
```

**config/queue.php:**
```php
'connections' => [
    'redis' => [
        'driver' => 'redis',
        'connection' => 'default',
        'queue' => env('REDIS_QUEUE', 'default'),
        'retry_after' => 90,
        'block_for' => null,
        'after_commit' => false,
    ],
],
```

---

## Redis for Queue - বিস্তারিত

### 🔄 **Queue Setup Step by Step**

**1. Job তৈরি করা:**
```bash
# Job class তৈরি
php artisan make:job SendEmailJob
```

**app/Jobs/SendEmailJob.php:**
```php
<?php

namespace App\Jobs;

use Illuminate\Bus\Queueable;
use Illuminate\Contracts\Queue\ShouldQueue;
use Illuminate\Foundation\Bus\Dispatchable;
use Illuminate\Queue\InteractsWithQueue;
use Illuminate\Queue\SerializesModels;
use Illuminate\Support\Facades\Mail;
use App\Mail\WelcomeMail;
use App\Models\User;

class SendEmailJob implements ShouldQueue
{
    use Dispatchable, InteractsWithQueue, Queueable, SerializesModels;

    public $user;
    public $tries = 3; // 3 বার try করবে
    public $timeout = 120; // 2 মিনিট timeout

    public function __construct(User $user)
    {
        $this->user = $user;
    }

    public function handle()
    {
        // Email send করার logic
        Mail::to($this->user->email)->send(new WelcomeMail($this->user));
        
        // Log for debugging
        \Log::info('Email sent to: ' . $this->user->email);
    }

    // Job fail হলে কি করবে
    public function failed(\Throwable $exception)
    {
        \Log::error('Email job failed for user: ' . $this->user->id, [
            'error' => $exception->getMessage()
        ]);
    }
}
```

**2. Job Dispatch করা:**
```php
// Controller থেকে job dispatch
use App\Jobs\SendEmailJob;

class UserController extends Controller
{
    public function register(Request $request)
    {
        $user = User::create($request->validated());
        
        // Immediate dispatch
        SendEmailJob::dispatch($user);
        
        // Delayed dispatch (5 মিনিট পর)
        SendEmailJob::dispatch($user)->delay(now()->addMinutes(5));
        
        // Specific queue এ dispatch
        SendEmailJob::dispatch($user)->onQueue('emails');
        
        return response()->json(['message' => 'User registered successfully']);
    }
}
```

### 🎯 **Queue Priority System**

**Multiple Queue Setup:**
```php
// Different priority queues
SendEmailJob::dispatch($user)->onQueue('high');    // Critical emails
SendEmailJob::dispatch($user)->onQueue('default'); // Normal emails  
SendEmailJob::dispatch($user)->onQueue('low');     // Newsletter, reports
```

**Worker Command with Priority:**
```bash
# High priority first, then default, then low
php artisan queue:work redis --queue=high,default,low --sleep=3 --tries=3
```

### 📊 **Queue Monitoring**

**Useful Commands:**
```bash
# Queue worker start
php artisan queue:work redis

# With specific options
php artisan queue:work redis --sleep=3 --tries=3 --max-time=3600 --memory=512

# Check failed jobs
php artisan queue:failed

# Retry specific failed job
php artisan queue:retry 5

# Retry all failed jobs
php artisan queue:retry all

# Clear failed jobs
php artisan queue:flush

# Monitor queue length
php artisan queue:monitor redis:default --max=100
```

### 🔧 **Advanced Queue Features**

**Job Batching:**
```php
use Illuminate\Bus\Batch;
use Illuminate\Support\Facades\Bus;

// Multiple jobs একসাথে process
$batch = Bus::batch([
    new SendEmailJob($user1),
    new SendEmailJob($user2),
    new SendEmailJob($user3),
])->then(function (Batch $batch) {
    // All jobs completed successfully
    \Log::info('All emails sent successfully');
})->catch(function (Batch $batch, \Throwable $e) {
    // First batch job failure detected
    \Log::error('Batch job failed: ' . $e->getMessage());
})->finally(function (Batch $batch) {
    // The batch has finished executing
    \Log::info('Batch execution completed');
})->dispatch();
```

**Job Chaining:**
```php
// Sequential job execution
SendEmailJob::withChain([
    new UpdateUserStatusJob($user),
    new SendWelcomeNotificationJob($user),
])->dispatch($user);
```

**Conditional Jobs:**
```php
// Job শুধু condition meet হলে চলবে
class SendEmailJob implements ShouldQueue
{
    public function handle()
    {
        // Skip if user is inactive
        if (!$this->user->is_active) {
            $this->delete(); // Job delete করে দাও
            return;
        }
        
        // Continue with email sending
        Mail::to($this->user->email)->send(new WelcomeMail($this->user));
    }
}
```

---

## Redis for Caching

### 💾 **Cache Implementation**

**Basic Caching:**
```php
use Illuminate\Support\Facades\Cache;

class PostController extends Controller
{
    public function index()
    {
        // Cache for 1 hour
        $posts = Cache::remember('posts.all', 3600, function () {
            return Post::with('user')->latest()->get();
        });
        
        return response()->json($posts);
    }
    
    public function show($id)
    {
        // Cache individual post
        $post = Cache::remember("posts.{$id}", 3600, function () use ($id) {
            return Post::with(['user', 'comments'])->findOrFail($id);
        });
        
        return response()->json($post);
    }
    
    public function update(Request $request, $id)
    {
        $post = Post::findOrFail($id);
        $post->update($request->validated());
        
        // Clear related caches
        Cache::forget("posts.{$id}");
        Cache::forget('posts.all');
        
        return response()->json($post);
    }
}
```

**Cache Tags (Advanced):**
```php
// Tagged cache for better management
class PostService
{
    public function getAllPosts()
    {
        return Cache::tags(['posts'])->remember('posts.all', 3600, function () {
            return Post::with('user')->latest()->get();
        });
    }
    
    public function getPostsByUser($userId)
    {
        return Cache::tags(['posts', 'users'])->remember("posts.user.{$userId}", 3600, function () use ($userId) {
            return Post::where('user_id', $userId)->latest()->get();
        });
    }
    
    public function clearPostCache()
    {
        // Clear all post-related cache
        Cache::tags(['posts'])->flush();
    }
}
```

**Cache Patterns:**
```php
// 1. Cache-Aside Pattern
public function getUser($id)
{
    $user = Cache::get("user.{$id}");
    
    if (!$user) {
        $user = User::find($id);
        Cache::put("user.{$id}", $user, 3600);
    }
    
    return $user;
}

// 2. Write-Through Pattern
public function updateUser($id, $data)
{
    $user = User::findOrFail($id);
    $user->update($data);
    
    // Update cache immediately
    Cache::put("user.{$id}", $user, 3600);
    
    return $user;
}

// 3. Write-Behind Pattern (using events)
class User extends Model
{
    protected static function booted()
    {
        static::updated(function ($user) {
            // Update cache asynchronously
            UpdateUserCacheJob::dispatch($user)->delay(now()->addSeconds(5));
        });
    }
}
```

---

## Redis for Session

### 🔐 **Session Configuration**

**config/session.php:**
```php
'driver' => env('SESSION_DRIVER', 'redis'),
'connection' => 'default',
```

**Benefits:**
- ✅ **Fast Access:** Memory-based session storage
- ✅ **Scalability:** Multiple servers can share sessions
- ✅ **Persistence:** Sessions survive server restarts
- ✅ **Clustering:** Redis cluster support

**Session Usage:**
```php
// Store session data
session(['user_preferences' => $preferences]);

// Retrieve session data
$preferences = session('user_preferences');

// Flash data (one-time use)
session()->flash('success', 'Profile updated successfully');

// Check if session has key
if (session()->has('cart')) {
    $cart = session('cart');
}
```

---

## Redis Commands ও Operations

### 🔧 **Laravel Redis Facade**

**Basic Operations:**
```php
use Illuminate\Support\Facades\Redis;

// String operations
Redis::set('key', 'value');
Redis::get('key');
Redis::setex('key', 3600, 'value'); // With expiry

// Hash operations
Redis::hset('user:1', 'name', 'John Doe');
Redis::hset('user:1', 'email', 'john@example.com');
Redis::hget('user:1', 'name');
Redis::hgetall('user:1');

// List operations
Redis::lpush('notifications', 'New message');
Redis::rpop('notifications');
Redis::lrange('notifications', 0, -1);

// Set operations
Redis::sadd('online_users', 'user:1');
Redis::srem('online_users', 'user:1');
Redis::smembers('online_users');

// Sorted set operations
Redis::zadd('leaderboard', 100, 'user:1');
Redis::zrange('leaderboard', 0, 9); // Top 10
```

**Advanced Operations:**
```php
// Atomic operations
Redis::multi();
Redis::incr('page_views');
Redis::sadd('visitors', $userId);
Redis::exec();

// Pipeline (batch operations)
Redis::pipeline(function ($pipe) {
    $pipe->set('key1', 'value1');
    $pipe->set('key2', 'value2');
    $pipe->set('key3', 'value3');
});

// Pub/Sub
Redis::publish('notifications', json_encode([
    'type' => 'new_message',
    'user_id' => 1,
    'message' => 'Hello World'
]));
```

### 📊 **Real-world Examples**

**1. Rate Limiting:**
```php
class ApiController extends Controller
{
    public function index(Request $request)
    {
        $key = 'rate_limit:' . $request->ip();
        $requests = Redis::incr($key);
        
        if ($requests === 1) {
            Redis::expire($key, 60); // 1 minute window
        }
        
        if ($requests > 100) {
            return response()->json(['error' => 'Rate limit exceeded'], 429);
        }
        
        // Continue with API logic
        return response()->json(['data' => 'API response']);
    }
}
```

**2. Real-time Counters:**
```php
class PostController extends Controller
{
    public function show($id)
    {
        $post = Post::findOrFail($id);
        
        // Increment view count
        Redis::incr("post:{$id}:views");
        
        // Get current view count
        $views = Redis::get("post:{$id}:views") ?: 0;
        
        return response()->json([
            'post' => $post,
            'views' => $views
        ]);
    }
}
```

**3. Shopping Cart:**
```php
class CartService
{
    public function addItem($userId, $productId, $quantity)
    {
        $key = "cart:{$userId}";
        Redis::hset($key, $productId, $quantity);
        Redis::expire($key, 86400); // 24 hours
    }
    
    public function getCart($userId)
    {
        $key = "cart:{$userId}";
        return Redis::hgetall($key);
    }
    
    public function removeItem($userId, $productId)
    {
        $key = "cart:{$userId}";
        Redis::hdel($key, $productId);
    }
    
    public function clearCart($userId)
    {
        $key = "cart:{$userId}";
        Redis::del($key);
    }
}
```

**4. Online Users Tracking:**
```php
class UserActivityService
{
    public function userOnline($userId)
    {
        Redis::setex("user:{$userId}:online", 300, time()); // 5 minutes
        Redis::sadd('online_users', $userId);
    }
    
    public function userOffline($userId)
    {
        Redis::del("user:{$userId}:online");
        Redis::srem('online_users', $userId);
    }
    
    public function getOnlineUsers()
    {
        return Redis::smembers('online_users');
    }
    
    public function getOnlineCount()
    {
        return Redis::scard('online_users');
    }
}
```

---

## Production এ Redis

### 🚀 **Production Configuration**

**Redis Configuration (redis.conf):**
```bash
# Memory optimization
maxmemory 2gb
maxmemory-policy allkeys-lru

# Persistence
save 900 1
save 300 10
save 60 10000

# Security
requirepass your_strong_password
bind 127.0.0.1

# Performance
tcp-keepalive 300
timeout 0
```

**Laravel Production Settings:**
```env
# Production Redis settings
REDIS_HOST=127.0.0.1
REDIS_PASSWORD=your_strong_password
REDIS_PORT=6379

# Separate databases for different purposes
REDIS_DB=0           # Default/Queue
REDIS_CACHE_DB=1     # Cache
REDIS_SESSION_DB=2   # Sessions
```

### 📊 **Monitoring ও Maintenance**

**Redis Monitoring Commands:**
```bash
# Redis CLI access
redis-cli -a your_password

# Monitor real-time commands
redis-cli monitor

# Get Redis info
redis-cli info

# Check memory usage
redis-cli info memory

# Check connected clients
redis-cli info clients

# Get slow queries
redis-cli slowlog get 10
```

**Laravel Commands:**
```bash
# Clear all Redis data
php artisan cache:clear

# Clear specific cache tags
php artisan cache:clear --tags=posts

# Queue monitoring
php artisan queue:monitor redis:default --max=1000

# Redis connection test
php artisan tinker
>>> Redis::ping()
```

### 🔧 **Performance Optimization**

**1. Connection Pooling:**
```php
// config/database.php
'redis' => [
    'client' => 'predis',
    'options' => [
        'cluster' => 'redis',
        'prefix' => 'myapp_',
        'parameters' => [
            'password' => env('REDIS_PASSWORD'),
            'database' => 0,
        ],
        'connection_persistent' => true,
    ],
],
```

**2. Serialization Optimization:**
```php
// Use efficient serialization
'redis' => [
    'serializer' => 'igbinary', // Faster than PHP serialize
    'compression' => 'lz4',     // Compress data
],
```

**3. Memory Management:**
```php
// Implement cache expiry strategy
class CacheService
{
    public function rememberWithTags($key, $tags, $ttl, $callback)
    {
        return Cache::tags($tags)->remember($key, $ttl, $callback);
    }
    
    public function clearExpiredCache()
    {
        // Custom cleanup logic
        $keys = Redis::keys('cache:*');
        foreach ($keys as $key) {
            if (Redis::ttl($key) < 0) {
                Redis::del($key);
            }
        }
    }
}
```

---

## সমস্যা ও সমাধান

### 🚨 **Common Issues**

**1. Redis Connection Failed:**
```bash
# Check Redis status
sudo systemctl status redis-server

# Restart Redis
sudo systemctl restart redis-server

# Check Redis logs
sudo tail -f /var/log/redis/redis-server.log

# Test connection
redis-cli ping
```

**2. Memory Issues:**
```bash
# Check Redis memory usage
redis-cli info memory

# Clear all data (DANGEROUS!)
redis-cli flushall

# Clear specific database
redis-cli -n 1 flushdb
```

**3. Queue Jobs Stuck:**
```bash
# Check queue status
php artisan queue:failed

# Clear stuck jobs
redis-cli del "queues:default"

# Restart queue workers
php artisan queue:restart
```

**4. Performance Issues:**
```bash
# Check slow queries
redis-cli slowlog get 10

# Monitor Redis operations
redis-cli monitor

# Check connected clients
redis-cli client list
```

### 🔧 **Debugging Tips**

**Laravel Debug Commands:**
```php
// Check Redis connection in tinker
php artisan tinker
>>> Redis::connection()->ping()
>>> Redis::get('test_key')
>>> Cache::put('test', 'value', 60)
>>> Cache::get('test')
```

**Redis CLI Debug:**
```bash
# Monitor all Redis commands
redis-cli monitor

# Check specific key
redis-cli get "laravel_database_cache:posts.all"

# Check key expiry
redis-cli ttl "laravel_database_cache:posts.all"

# List all keys (use carefully in production)
redis-cli keys "*"
```

### 📋 **Best Practices**

**1. Key Naming Convention:**
```php
// Good naming convention
"app:cache:posts:all"
"app:session:user:123"
"app:queue:emails:high"

// Avoid
"posts"
"data"
"temp"
```

**2. Memory Management:**
```php
// Set appropriate TTL
Cache::put('key', 'value', 3600); // 1 hour

// Use cache tags for bulk clearing
Cache::tags(['posts', 'users'])->put('key', 'value', 3600);

// Implement cache warming
Artisan::command('cache:warm', function () {
    // Pre-load frequently accessed data
    Cache::remember('popular_posts', 3600, function () {
        return Post::popular()->get();
    });
});
```

**3. Error Handling:**
```php
try {
    Redis::set('key', 'value');
} catch (\Exception $e) {
    // Fallback to database or file cache
    \Log::error('Redis connection failed: ' . $e->getMessage());
    
    // Use alternative storage
    Cache::store('file')->put('key', 'value', 3600);
}
```

---

## 🎯 **Quick Reference**

### Essential Commands:
```bash
# Redis server
sudo systemctl start redis-server
redis-cli ping

# Laravel queue
php artisan queue:work redis --sleep=3 --tries=3
php artisan queue:restart

# Cache operations
php artisan cache:clear
php artisan config:cache
```

### Common Patterns:
```php
// Queue job
SendEmailJob::dispatch($user)->onQueue('emails');

// Cache with tags
Cache::tags(['posts'])->remember('posts.all', 3600, $callback);

// Redis operations
Redis::setex('key', 3600, 'value');
Redis::incr('counter');
```

এই গাইড follow করে আপনি Laravel project এ Redis efficiently ব্যবহার করতে পারবেন। Queue, caching, এবং session management এর জন্য Redis একটি powerful solution।