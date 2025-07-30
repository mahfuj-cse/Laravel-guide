# 1️⃣6️⃣ Laravel Events & Listeners - বিস্তারিত বাংলা গাইড

## 📋 সূচিপত্র
- [Events & Listeners কি?](#events--listeners-কি)
- [Event তৈরি ও ব্যবহার](#event-তৈরি-ও-ব্যবহার)
- [Listener তৈরি ও Registration](#listener-তৈরি-ও-registration)
- [Event Broadcasting](#event-broadcasting)
- [Model Events](#model-events)
- [Event Subscribers](#event-subscribers)
- [Advanced Techniques](#advanced-techniques)
- [Real-world Examples](#real-world-examples)

---

## Events & Listeners কি?

**Events & Listeners** হলো Laravel এর **Observer Pattern** implementation যা **Decoupled Architecture** তৈরি করে।

### 🎯 কেন Events & Listeners ব্যবহার করবেন?
- ✅ **Decoupled Code** - Components আলাদা থাকে
- ✅ **Reusable Logic** - একই event এ multiple listeners
- ✅ **Maintainable** - Code organization ভালো
- ✅ **Testable** - আলাদা আলাদা test করা যায়
- ✅ **Scalable** - নতুন features সহজে যোগ করা যায়

### Traditional vs Event-Driven:
```php
// ❌ Traditional Approach (Tightly Coupled)
class UserController extends Controller
{
    public function store(Request $request)
    {
        $user = User::create($request->validated());
        
        // All logic in one place
        Mail::to($user)->send(new WelcomeEmail($user));
        $user->assignRole('user');
        ActivityLog::create(['action' => 'user_created', 'user_id' => $user->id]);
        Cache::forget('users_count');
        
        return response()->json($user);
    }
}

// ✅ Event-Driven Approach (Decoupled)
class UserController extends Controller
{
    public function store(Request $request)
    {
        $user = User::create($request->validated());
        
        // Fire event - let listeners handle the rest
        event(new UserRegistered($user));
        
        return response()->json($user);
    }
}
```

---

## Event তৈরি ও ব্যবহার

### ১. Event তৈরি করা:
```bash
# Basic Event
php artisan make:event UserRegistered

# Event with broadcasting
php artisan make:event OrderStatusChanged --broadcast
```

### ২. Basic Event Class:
```php
<?php
// app/Events/UserRegistered.php

namespace App\Events;

use App\Models\User;
use Illuminate\Foundation\Events\Dispatchable;
use Illuminate\Queue\SerializesModels;

class UserRegistered
{
    use Dispatchable, SerializesModels;

    public $user;

    public function __construct(User $user)
    {
        $this->user = $user;
    }
}
```

### ৩. Event Dispatching:
```php
<?php

class UserController extends Controller
{
    public function store(Request $request)
    {
        $user = User::create($request->validated());

        // Method 1: Using event() helper
        event(new UserRegistered($user));

        // Method 2: Using Event facade
        Event::dispatch(new UserRegistered($user));

        // Method 3: Using static dispatch method
        UserRegistered::dispatch($user);

        return response()->json($user);
    }
}
```

### ৪. Event with Additional Data:
```php
<?php
// app/Events/OrderStatusChanged.php

namespace App\Events;

use App\Models\Order;
use Illuminate\Foundation\Events\Dispatchable;
use Illuminate\Queue\SerializesModels;

class OrderStatusChanged
{
    use Dispatchable, SerializesModels;

    public $order;
    public $previousStatus;
    public $newStatus;
    public $changedBy;

    public function __construct(Order $order, $previousStatus, $newStatus, $changedBy = null)
    {
        $this->order = $order;
        $this->previousStatus = $previousStatus;
        $this->newStatus = $newStatus;
        $this->changedBy = $changedBy ?? auth()->user();
    }
}
```

---

## Listener তৈরি ও Registration

### ১. Listener তৈরি করা:
```bash
# Basic Listener
php artisan make:listener SendWelcomeEmail

# Listener for specific event
php artisan make:listener SendWelcomeEmail --event=UserRegistered

# Queued Listener (background processing)
php artisan make:listener SendWelcomeEmail --event=UserRegistered --queued
```

### ২. Basic Listener:
```php
<?php
// app/Listeners/SendWelcomeEmail.php

namespace App\Listeners;

use App\Events\UserRegistered;
use App\Mail\WelcomeEmail;
use Illuminate\Support\Facades\Mail;
use Illuminate\Contracts\Queue\ShouldQueue;

class SendWelcomeEmail implements ShouldQueue
{
    public function handle(UserRegistered $event)
    {
        // Send welcome email
        Mail::to($event->user->email)->send(new WelcomeEmail($event->user));
    }

    public function failed(UserRegistered $event, $exception)
    {
        // Handle failed job
        \Log::error('Failed to send welcome email', [
            'user_id' => $event->user->id,
            'error' => $exception->getMessage()
        ]);
    }
}
```

### ৩. Event Registration:
```php
<?php
// app/Providers/EventServiceProvider.php

namespace App\Providers;

use Illuminate\Foundation\Support\Providers\EventServiceProvider as ServiceProvider;
use App\Events\UserRegistered;
use App\Events\OrderStatusChanged;
use App\Listeners\SendWelcomeEmail;
use App\Listeners\AssignUserRole;
use App\Listeners\LogUserActivity;
use App\Listeners\UpdateOrderNotification;
use App\Listeners\SendOrderStatusEmail;

class EventServiceProvider extends ServiceProvider
{
    protected $listen = [
        UserRegistered::class => [
            SendWelcomeEmail::class,
            AssignUserRole::class,
            LogUserActivity::class,
        ],

        OrderStatusChanged::class => [
            UpdateOrderNotification::class,
            SendOrderStatusEmail::class,
        ],

        // Laravel built-in events
        'Illuminate\Auth\Events\Login' => [
            'App\Listeners\LogUserLogin',
        ],

        'Illuminate\Auth\Events\Logout' => [
            'App\Listeners\LogUserLogout',
        ],
    ];

    public function boot()
    {
        parent::boot();

        // Manual event registration
        Event::listen('user.registered', function ($user) {
            // Handle event
        });

        // Wildcard listeners
        Event::listen('user.*', function ($eventName, array $data) {
            // Handle all user events
        });
    }
}
```

### ৪. Multiple Listeners Example:
```php
<?php
// Multiple listeners for UserRegistered event

// app/Listeners/AssignUserRole.php
class AssignUserRole
{
    public function handle(UserRegistered $event)
    {
        $event->user->assignRole('user');
    }
}

// app/Listeners/LogUserActivity.php
class LogUserActivity
{
    public function handle(UserRegistered $event)
    {
        ActivityLog::create([
            'action' => 'user_registered',
            'user_id' => $event->user->id,
            'ip_address' => request()->ip(),
            'user_agent' => request()->userAgent(),
        ]);
    }
}

// app/Listeners/UpdateUserStats.php
class UpdateUserStats
{
    public function handle(UserRegistered $event)
    {
        Cache::increment('total_users');
        Cache::forget('users_count');
    }
}
```

---

## Event Broadcasting

### ১. Broadcast Event:
```php
<?php
// app/Events/OrderStatusChanged.php

namespace App\Events;

use App\Models\Order;
use Illuminate\Broadcasting\Channel;
use Illuminate\Broadcasting\InteractsWithSockets;
use Illuminate\Broadcasting\PresenceChannel;
use Illuminate\Broadcasting\PrivateChannel;
use Illuminate\Contracts\Broadcasting\ShouldBroadcast;
use Illuminate\Foundation\Events\Dispatchable;
use Illuminate\Queue\SerializesModels;

class OrderStatusChanged implements ShouldBroadcast
{
    use Dispatchable, InteractsWithSockets, SerializesModels;

    public $order;
    public $status;

    public function __construct(Order $order)
    {
        $this->order = $order;
        $this->status = $order->status;
    }

    public function broadcastOn()
    {
        return [
            new PrivateChannel('orders.' . $this->order->user_id),
            new Channel('orders.public')
        ];
    }

    public function broadcastAs()
    {
        return 'order.status.changed';
    }

    public function broadcastWith()
    {
        return [
            'order_id' => $this->order->id,
            'status' => $this->status,
            'message' => "Your order #{$this->order->id} is now {$this->status}",
            'timestamp' => now()->toISOString()
        ];
    }

    public function broadcastWhen()
    {
        return $this->order->status !== 'draft';
    }
}
```

### ২. Frontend Integration:
```javascript
// resources/js/app.js
import Echo from 'laravel-echo';

window.Echo = new Echo({
    broadcaster: 'pusher',
    key: process.env.MIX_PUSHER_APP_KEY,
    cluster: process.env.MIX_PUSHER_APP_CLUSTER,
    forceTLS: true
});

// Listen to private channel
Echo.private(`orders.${userId}`)
    .listen('.order.status.changed', (e) => {
        console.log('Order status changed:', e);
        
        // Update UI
        updateOrderStatus(e.order_id, e.status);
        
        // Show notification
        showNotification(e.message);
    });

// Listen to public channel
Echo.channel('orders.public')
    .listen('.order.status.changed', (e) => {
        console.log('Public order update:', e);
    });
```

---

## Model Events

### ১. Model Event Listeners:
```php
<?php
// app/Models/Post.php

namespace App\Models;

use Illuminate\Database\Eloquent\Model;

class Post extends Model
{
    protected $fillable = ['title', 'content', 'user_id', 'status'];

    protected static function booted()
    {
        // Before creating
        static::creating(function ($post) {
            $post->slug = Str::slug($post->title);
            $post->user_id = auth()->id();
        });

        // After creating
        static::created(function ($post) {
            event(new PostCreated($post));
        });

        // Before updating
        static::updating(function ($post) {
            if ($post->isDirty('title')) {
                $post->slug = Str::slug($post->title);
            }
        });

        // After updating
        static::updated(function ($post) {
            if ($post->wasChanged('status')) {
                event(new PostStatusChanged($post, $post->getOriginal('status')));
            }
        });

        // Before deleting
        static::deleting(function ($post) {
            // Delete related comments
            $post->comments()->delete();
        });

        // After deleting
        static::deleted(function ($post) {
            event(new PostDeleted($post));
        });
    }
}
```

### ২. Model Observer:
```bash
# Create Observer
php artisan make:observer PostObserver --model=Post
```

```php
<?php
// app/Observers/PostObserver.php

namespace App\Observers;

use App\Models\Post;
use App\Events\PostCreated;
use App\Events\PostUpdated;
use App\Events\PostDeleted;

class PostObserver
{
    public function creating(Post $post)
    {
        $post->slug = Str::slug($post->title);
        $post->user_id = auth()->id();
    }

    public function created(Post $post)
    {
        event(new PostCreated($post));
        
        // Clear cache
        Cache::tags(['posts'])->flush();
    }

    public function updating(Post $post)
    {
        if ($post->isDirty('title')) {
            $post->slug = Str::slug($post->title);
        }
    }

    public function updated(Post $post)
    {
        event(new PostUpdated($post));
        
        // Clear specific cache
        Cache::forget("post.{$post->id}");
    }

    public function deleted(Post $post)
    {
        event(new PostDeleted($post));
        
        // Clean up related data
        $post->comments()->delete();
        $post->tags()->detach();
    }
}

// Register Observer in EventServiceProvider
public function boot()
{
    Post::observe(PostObserver::class);
}
```

---

## Event Subscribers

### ১. Event Subscriber:
```bash
# Create Event Subscriber
php artisan make:listener UserEventSubscriber
```

```php
<?php
// app/Listeners/UserEventSubscriber.php

namespace App\Listeners;

use App\Events\UserRegistered;
use App\Events\UserLoggedIn;
use App\Events\UserLoggedOut;
use Illuminate\Events\Dispatcher;

class UserEventSubscriber
{
    public function handleUserRegistration($event)
    {
        // Send welcome email
        Mail::to($event->user)->send(new WelcomeEmail($event->user));
        
        // Log activity
        ActivityLog::create([
            'action' => 'user_registered',
            'user_id' => $event->user->id
        ]);
    }

    public function handleUserLogin($event)
    {
        // Update last login
        $event->user->update(['last_login_at' => now()]);
        
        // Log activity
        ActivityLog::create([
            'action' => 'user_logged_in',
            'user_id' => $event->user->id
        ]);
    }

    public function handleUserLogout($event)
    {
        // Log activity
        ActivityLog::create([
            'action' => 'user_logged_out',
            'user_id' => $event->user->id
        ]);
    }

    public function subscribe(Dispatcher $events)
    {
        $events->listen(
            UserRegistered::class,
            [UserEventSubscriber::class, 'handleUserRegistration']
        );

        $events->listen(
            'Illuminate\Auth\Events\Login',
            [UserEventSubscriber::class, 'handleUserLogin']
        );

        $events->listen(
            'Illuminate\Auth\Events\Logout',
            [UserEventSubscriber::class, 'handleUserLogout']
        );
    }
}

// Register in EventServiceProvider
protected $subscribe = [
    UserEventSubscriber::class,
];
```

---

## Advanced Techniques

### ১. Conditional Event Firing:
```php
<?php

class OrderController extends Controller
{
    public function updateStatus(Request $request, Order $order)
    {
        $previousStatus = $order->status;
        $order->update(['status' => $request->status]);

        // Fire event only if status actually changed
        if ($previousStatus !== $order->status) {
            event(new OrderStatusChanged($order, $previousStatus, $order->status));
        }

        return response()->json($order);
    }
}
```

### ২. Event with Conditions:
```php
<?php
// app/Events/ProductPurchased.php

class ProductPurchased
{
    use Dispatchable, SerializesModels;

    public $product;
    public $user;
    public $quantity;

    public function __construct(Product $product, User $user, $quantity)
    {
        $this->product = $product;
        $this->user = $user;
        $this->quantity = $quantity;
    }

    public function shouldSendNotification()
    {
        return $this->quantity > 1;
    }

    public function isHighValuePurchase()
    {
        return ($this->product->price * $this->quantity) > 10000;
    }
}

// Listener
class ProcessProductPurchase
{
    public function handle(ProductPurchased $event)
    {
        // Update inventory
        $event->product->decrement('stock', $event->quantity);

        // Send notification for bulk purchases
        if ($event->shouldSendNotification()) {
            Mail::to($event->user)->send(new BulkPurchaseConfirmation($event));
        }

        // Special handling for high-value purchases
        if ($event->isHighValuePurchase()) {
            // Notify admin
            Mail::to(config('mail.admin'))->send(new HighValuePurchaseAlert($event));
            
            // Create audit log
            AuditLog::create([
                'action' => 'high_value_purchase',
                'user_id' => $event->user->id,
                'amount' => $event->product->price * $event->quantity
            ]);
        }
    }
}
```

### ৩. Event Middleware:
```php
<?php
// app/Events/Middleware/LogEvent.php

class LogEvent
{
    public function handle($event, $next)
    {
        \Log::info('Event fired: ' . get_class($event));
        
        $result = $next($event);
        
        \Log::info('Event processed: ' . get_class($event));
        
        return $result;
    }
}

// Apply middleware to event
class UserRegistered
{
    use Dispatchable, SerializesModels;

    public function middleware()
    {
        return [LogEvent::class];
    }
}
```

---

## Real-world Examples

### ১. E-commerce Order System:
```php
<?php
// Complete e-commerce event system

// Events
class OrderPlaced
{
    public $order;
    
    public function __construct(Order $order)
    {
        $this->order = $order;
    }
}

class PaymentProcessed
{
    public $payment;
    public $order;
    
    public function __construct(Payment $payment, Order $order)
    {
        $this->payment = $payment;
        $this->order = $order;
    }
}

class OrderShipped
{
    public $order;
    public $trackingNumber;
    
    public function __construct(Order $order, $trackingNumber)
    {
        $this->order = $order;
        $this->trackingNumber = $trackingNumber;
    }
}

// Listeners
class SendOrderConfirmation
{
    public function handle(OrderPlaced $event)
    {
        Mail::to($event->order->customer->email)
            ->send(new OrderConfirmationMail($event->order));
    }
}

class UpdateInventory
{
    public function handle(OrderPlaced $event)
    {
        foreach ($event->order->items as $item) {
            $item->product->decrement('stock', $item->quantity);
        }
    }
}

class ProcessPayment
{
    public function handle(OrderPlaced $event)
    {
        $paymentResult = PaymentGateway::charge($event->order);
        
        if ($paymentResult->successful()) {
            event(new PaymentProcessed($paymentResult, $event->order));
        }
    }
}

class SendShippingNotification
{
    public function handle(OrderShipped $event)
    {
        // Email notification
        Mail::to($event->order->customer->email)
            ->send(new ShippingNotificationMail($event->order, $event->trackingNumber));
        
        // SMS notification
        SMS::send($event->order->customer->phone, 
            "Your order #{$event->order->id} has been shipped. Tracking: {$event->trackingNumber}");
    }
}

// Event Registration
protected $listen = [
    OrderPlaced::class => [
        SendOrderConfirmation::class,
        UpdateInventory::class,
        ProcessPayment::class,
    ],
    
    PaymentProcessed::class => [
        UpdateOrderStatus::class,
        SendPaymentConfirmation::class,
    ],
    
    OrderShipped::class => [
        SendShippingNotification::class,
        UpdateOrderStatus::class,
    ],
];
```

### ২. Blog System with Events:
```php
<?php
// Blog system events

// Events
class PostPublished
{
    public $post;
    
    public function __construct(Post $post)
    {
        $this->post = $post;
    }
}

class CommentAdded
{
    public $comment;
    public $post;
    
    public function __construct(Comment $comment, Post $post)
    {
        $this->comment = $comment;
        $this->post = $post;
    }
}

// Listeners
class NotifyPostSubscribers
{
    public function handle(PostPublished $event)
    {
        $subscribers = $event->post->author->subscribers;
        
        foreach ($subscribers as $subscriber) {
            Mail::to($subscriber->email)
                ->queue(new NewPostNotification($event->post, $subscriber));
        }
    }
}

class UpdateSitemap
{
    public function handle(PostPublished $event)
    {
        // Regenerate sitemap
        Artisan::call('sitemap:generate');
    }
}

class ClearPostCache
{
    public function handle(PostPublished $event)
    {
        Cache::tags(['posts', 'blog'])->flush();
        Cache::forget('recent_posts');
        Cache::forget('popular_posts');
    }
}

class NotifyPostAuthor
{
    public function handle(CommentAdded $event)
    {
        if ($event->comment->user_id !== $event->post->user_id) {
            Mail::to($event->post->author->email)
                ->send(new NewCommentNotification($event->comment, $event->post));
        }
    }
}

class UpdateCommentCount
{
    public function handle(CommentAdded $event)
    {
        $event->post->increment('comments_count');
        Cache::forget("post.{$event->post->id}.comments_count");
    }
}
```

### ৩. User Activity Tracking:
```php
<?php
// User activity tracking system

// Events
class UserActivityEvent
{
    public $user;
    public $activity;
    public $metadata;
    
    public function __construct(User $user, $activity, array $metadata = [])
    {
        $this->user = $user;
        $this->activity = $activity;
        $this->metadata = $metadata;
    }
}

// Listeners
class LogUserActivity
{
    public function handle(UserActivityEvent $event)
    {
        UserActivity::create([
            'user_id' => $event->user->id,
            'activity' => $event->activity,
            'metadata' => $event->metadata,
            'ip_address' => request()->ip(),
            'user_agent' => request()->userAgent(),
            'created_at' => now()
        ]);
    }
}

class UpdateUserStats
{
    public function handle(UserActivityEvent $event)
    {
        $event->user->increment('activity_count');
        $event->user->update(['last_activity_at' => now()]);
    }
}

class CheckSuspiciousActivity
{
    public function handle(UserActivityEvent $event)
    {
        $recentActivities = UserActivity::where('user_id', $event->user->id)
            ->where('created_at', '>', now()->subMinutes(5))
            ->count();
            
        if ($recentActivities > 50) {
            // Suspicious activity detected
            event(new SuspiciousActivityDetected($event->user, $recentActivities));
        }
    }
}

// Usage in controllers
class PostController extends Controller
{
    public function show(Post $post)
    {
        if (auth()->check()) {
            event(new UserActivityEvent(auth()->user(), 'post_viewed', [
                'post_id' => $post->id,
                'post_title' => $post->title
            ]));
        }
        
        return view('posts.show', compact('post'));
    }
}
```

---

## 🎯 Best Practices:

### ✅ **Event Design:**
- Events should represent **what happened**, not **what should happen**
- Keep events **simple** and **focused**
- Include **relevant data** only
- Use **descriptive names** (UserRegistered, OrderShipped)

### ✅ **Listener Design:**
- Each listener should have **single responsibility**
- Use **queued listeners** for heavy operations
- Handle **failures** gracefully
- Keep listeners **independent** of each other

### ✅ **Performance:**
- Use **queued listeners** for non-critical operations
- Avoid **heavy processing** in synchronous listeners
- Consider **event batching** for high-frequency events
- Use **lazy loading** for event data

### ✅ **Testing:**
```php
// Test events are fired
Event::fake();
$user = User::factory()->create();
Event::assertDispatched(UserRegistered::class);

// Test specific event data
Event::fake();
event(new OrderPlaced($order));
Event::assertDispatched(OrderPlaced::class, function ($event) use ($order) {
    return $event->order->id === $order->id;
});
```

---

## 📚 আরও জানতে:
- [Laravel Events](https://laravel.com/docs/events)
- [Event Broadcasting](https://laravel.com/docs/broadcasting)
- [Model Events](https://laravel.com/docs/eloquent#events)