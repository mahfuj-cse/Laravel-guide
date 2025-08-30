# Laravel Events & Listeners vs Jobs & Queues - সম্পূর্ণ তুলনামূলক গাইড

## 📋 সূচিপত্র
- [মূল পার্থক্য](#মূল-পার্থক্য)
- [Events & Listeners](#events--listeners)
- [Jobs & Queues](#jobs--queues)
- [Synchronous vs Asynchronous](#synchronous-vs-asynchronous)
- [Line by Line Execution](#line-by-line-execution)
- [কখন কোনটি ব্যবহার করবেন](#কখন-কোনটি-ব্যবহার-করবেন)
- [Performance তুলনা](#performance-তুলনা)
- [বাস্তব উদাহরণ](#বাস্তব-উদাহরণ)

---

## মূল পার্থক্য

### 🎯 Events & Listeners
- **উদ্দেশ্য**: কোন ঘটনা ঘটলে অন্য কাজগুলো করা
- **Pattern**: Observer Pattern
- **Execution**: Synchronous (default) বা Asynchronous
- **Use Case**: একটি action এর ফলে multiple reactions

### 🎯 Jobs & Queues  
- **উদ্দেশ্য**: ভারী কাজগুলো background এ করা
- **Pattern**: Command Pattern
- **Execution**: Asynchronous (background)
- **Use Case**: Time-consuming tasks

---

## Events & Listeners

### ১. Event তৈরি ও ব্যবহার:

```php
<?php
// Event তৈরি
php artisan make:event UserRegistered
php artisan make:listener SendWelcomeEmail
php artisan make:listener UpdateUserStats
php artisan make:listener LogUserActivity

// app/Events/UserRegistered.php
namespace App\Events;

use Illuminate\Foundation\Events\Dispatchable;
use Illuminate\Queue\SerializesModels;
use App\Models\User;

class UserRegistered
{
    use Dispatchable, SerializesModels;

    public $user;

    public function __construct(User $user)
    {
        $this->user = $user;
    }
}

// app/Listeners/SendWelcomeEmail.php
namespace App\Listeners;

use App\Events\UserRegistered;
use Illuminate\Support\Facades\Mail;
use App\Mail\WelcomeEmail;

class SendWelcomeEmail
{
    public function handle(UserRegistered $event)
    {
        // Synchronous execution - line by line
        echo "1. Starting welcome email process\n";
        
        Mail::to($event->user->email)->send(new WelcomeEmail($event->user));
        
        echo "2. Welcome email sent\n";
    }
}

// app/Listeners/UpdateUserStats.php
class UpdateUserStats
{
    public function handle(UserRegistered $event)
    {
        echo "3. Updating user statistics\n";
        
        // Database update
        \DB::table('user_stats')->increment('total_users');
        
        echo "4. User stats updated\n";
    }
}
```

### ২. Event Registration:

```php
<?php
// app/Providers/EventServiceProvider.php

protected $listen = [
    UserRegistered::class => [
        SendWelcomeEmail::class,
        UpdateUserStats::class,
        LogUserActivity::class,
    ],
];

// অথবা Manual registration
public function boot()
{
    Event::listen(UserRegistered::class, SendWelcomeEmail::class);
    Event::listen(UserRegistered::class, UpdateUserStats::class);
}
```

### ৩. Event Dispatch:

```php
<?php
// Controller এ Event fire করা

class UserController extends Controller
{
    public function register(Request $request)
    {
        echo "A. User creation started\n";
        
        $user = User::create($request->validated());
        
        echo "B. User created in database\n";
        
        // Event dispatch - এখানেই সব listeners চলবে
        event(new UserRegistered($user));
        
        echo "C. All event listeners completed\n";
        
        return response()->json([
            'message' => 'User registered successfully',
            'user' => $user
        ]);
        
        echo "D. Response sent\n"; // এটা execute হবে না
    }
}
```

**Output (Synchronous):**
```
A. User creation started
B. User created in database
1. Starting welcome email process
2. Welcome email sent
3. Updating user statistics  
4. User stats updated
C. All event listeners completed
```

---

## Jobs & Queues

### ১. Job তৈরি ও ব্যবহার:

```php
<?php
// Job তৈরি
php artisan make:job SendWelcomeEmailJob
php artisan make:job UpdateUserStatsJob

// app/Jobs/SendWelcomeEmailJob.php
namespace App\Jobs;

use Illuminate\Bus\Queueable;
use Illuminate\Contracts\Queue\ShouldQueue;
use Illuminate\Foundation\Bus\Dispatchable;
use Illuminate\Queue\InteractsWithQueue;
use Illuminate\Queue\SerializesModels;
use App\Models\User;

class SendWelcomeEmailJob implements ShouldQueue
{
    use Dispatchable, InteractsWithQueue, Queueable, SerializesModels;

    protected $user;

    public function __construct(User $user)
    {
        $this->user = $user;
    }

    public function handle()
    {
        // Background এ execute হবে
        echo "Job: Sending welcome email to {$this->user->email}\n";
        
        Mail::to($this->user->email)->send(new WelcomeEmail($this->user));
        
        echo "Job: Welcome email sent successfully\n";
    }
}
```

### ২. Job Dispatch:

```php
<?php
// Controller এ Job dispatch

class UserController extends Controller
{
    public function register(Request $request)
    {
        echo "A. User creation started\n";
        
        $user = User::create($request->validated());
        
        echo "B. User created in database\n";
        
        // Jobs dispatch - background এ চলবে
        SendWelcomeEmailJob::dispatch($user);
        UpdateUserStatsJob::dispatch($user);
        
        echo "C. Jobs dispatched to queue\n";
        
        return response()->json([
            'message' => 'User registered successfully',
            'user' => $user
        ]);
        
        echo "D. Response sent\n";
    }
}
```

**Output (Asynchronous):**
```
A. User creation started
B. User created in database  
C. Jobs dispatched to queue
D. Response sent

// Background এ (আলাদা process):
Job: Sending welcome email to user@example.com
Job: Welcome email sent successfully
```

---

## Synchronous vs Asynchronous

### 🔄 Synchronous Events (Default):

```php
<?php
class UserController extends Controller
{
    public function register(Request $request)
    {
        $startTime = microtime(true);
        echo "1. Process started at: " . date('H:i:s') . "\n";
        
        $user = User::create($request->validated());
        echo "2. User created at: " . date('H:i:s') . "\n";
        
        // Synchronous event - সব listeners এক সাথে চলবে
        event(new UserRegistered($user));
        echo "3. Event completed at: " . date('H:i:s') . "\n";
        
        $endTime = microtime(true);
        $duration = ($endTime - $startTime) * 1000;
        echo "Total time: {$duration}ms\n";
        
        return response()->json(['message' => 'Success']);
    }
}

// Listener
class SendWelcomeEmail
{
    public function handle(UserRegistered $event)
    {
        echo "  - Email listener started\n";
        sleep(2); // 2 সেকেন্ড delay simulate
        echo "  - Email sent\n";
    }
}

class UpdateUserStats  
{
    public function handle(UserRegistered $event)
    {
        echo "  - Stats listener started\n";
        sleep(1); // 1 সেকেন্ড delay
        echo "  - Stats updated\n";
    }
}
```

**Output:**
```
1. Process started at: 14:30:00
2. User created at: 14:30:00
  - Email listener started
  - Email sent                    // 2 সেকেন্ড পর
  - Stats listener started  
  - Stats updated                 // আরো 1 সেকেন্ড পর
3. Event completed at: 14:30:03
Total time: 3000ms               // মোট 3 সেকেন্ড
```

### ⚡ Asynchronous Jobs:

```php
<?php
class UserController extends Controller
{
    public function register(Request $request)
    {
        $startTime = microtime(true);
        echo "1. Process started at: " . date('H:i:s') . "\n";
        
        $user = User::create($request->validated());
        echo "2. User created at: " . date('H:i:s') . "\n";
        
        // Asynchronous jobs - queue এ add হবে
        SendWelcomeEmailJob::dispatch($user);
        UpdateUserStatsJob::dispatch($user);
        echo "3. Jobs queued at: " . date('H:i:s') . "\n";
        
        $endTime = microtime(true);
        $duration = ($endTime - $startTime) * 1000;
        echo "Total time: {$duration}ms\n";
        
        return response()->json(['message' => 'Success']);
    }
}
```

**Output:**
```
1. Process started at: 14:30:00
2. User created at: 14:30:00  
3. Jobs queued at: 14:30:00
Total time: 50ms                 // খুবই দ্রুত!

// Background worker এ (আলাদাভাবে):
Job: Email job started at: 14:30:01
Job: Email sent at: 14:30:03
Job: Stats job started at: 14:30:01  
Job: Stats updated at: 14:30:02
```

### 🚀 Asynchronous Events (ShouldQueue):

```php
<?php
// Listener কে Queueable করা

use Illuminate\Contracts\Queue\ShouldQueue;

class SendWelcomeEmail implements ShouldQueue
{
    use InteractsWithQueue;
    
    public function handle(UserRegistered $event)
    {
        echo "Queued listener: Sending email\n";
        sleep(2);
        echo "Queued listener: Email sent\n";
    }
}

// এখন Event dispatch করলে listener background এ চলবে
event(new UserRegistered($user)); // তাৎক্ষণিক return
```

---

## Line by Line Execution

### 📝 Synchronous Event Flow:

```php
<?php
class OrderController extends Controller
{
    public function placeOrder(Request $request)
    {
        echo "Line 1: Order processing started\n";
        
        $order = Order::create($request->validated());
        echo "Line 2: Order saved to database\n";
        
        // Event dispatch - এখানে execution থেমে যাবে
        echo "Line 3: About to fire OrderPlaced event\n";
        event(new OrderPlaced($order));
        echo "Line 4: OrderPlaced event completed\n";
        
        echo "Line 5: Preparing response\n";
        return response()->json(['order_id' => $order->id]);
        echo "Line 6: This will never execute\n";
    }
}

// Event Listeners
class SendOrderConfirmation
{
    public function handle(OrderPlaced $event)
    {
        echo "  Listener 1 Line A: Email confirmation started\n";
        Mail::to($event->order->user->email)->send(new OrderConfirmationMail());
        echo "  Listener 1 Line B: Email sent\n";
    }
}

class UpdateInventory
{
    public function handle(OrderPlaced $event)
    {
        echo "  Listener 2 Line A: Inventory update started\n";
        foreach ($event->order->items as $item) {
            $item->product->decrement('stock', $item->quantity);
        }
        echo "  Listener 2 Line B: Inventory updated\n";
    }
}

class LogOrderActivity
{
    public function handle(OrderPlaced $event)
    {
        echo "  Listener 3 Line A: Logging started\n";
        ActivityLog::create([
            'type' => 'order_placed',
            'order_id' => $event->order->id
        ]);
        echo "  Listener 3 Line B: Activity logged\n";
    }
}
```

**Execution Output:**
```
Line 1: Order processing started
Line 2: Order saved to database
Line 3: About to fire OrderPlaced event
  Listener 1 Line A: Email confirmation started
  Listener 1 Line B: Email sent
  Listener 2 Line A: Inventory update started  
  Listener 2 Line B: Inventory updated
  Listener 3 Line A: Logging started
  Listener 3 Line B: Activity logged
Line 4: OrderPlaced event completed
Line 5: Preparing response
```

### 🔄 Asynchronous Job Flow:

```php
<?php
class OrderController extends Controller
{
    public function placeOrder(Request $request)
    {
        echo "Line 1: Order processing started\n";
        
        $order = Order::create($request->validated());
        echo "Line 2: Order saved to database\n";
        
        // Jobs dispatch - তাৎক্ষণিক
        echo "Line 3: About to dispatch jobs\n";
        SendOrderConfirmationJob::dispatch($order);
        UpdateInventoryJob::dispatch($order);
        LogOrderActivityJob::dispatch($order);
        echo "Line 4: All jobs dispatched\n";
        
        echo "Line 5: Preparing response\n";
        return response()->json(['order_id' => $order->id]);
        echo "Line 6: This will never execute\n";
    }
}
```

**Execution Output:**
```
Line 1: Order processing started
Line 2: Order saved to database
Line 3: About to dispatch jobs
Line 4: All jobs dispatched
Line 5: Preparing response

// Background এ (parallel execution):
Job 1: Email confirmation job started
Job 2: Inventory update job started  
Job 3: Logging job started
Job 1: Email sent
Job 3: Activity logged
Job 2: Inventory updated
```

---

## কখন কোনটি ব্যবহার করবেন

### 🎯 Events & Listeners ব্যবহার করুন যখন:

```php
<?php
// ✅ একটি action এর ফলে multiple reactions প্রয়োজন
class UserController extends Controller
{
    public function deleteUser(User $user)
    {
        // User delete করার সাথে সাথে অনেক কিছু করতে হবে
        event(new UserDeleted($user));
        // - Delete user posts
        // - Remove from groups  
        // - Cancel subscriptions
        // - Send goodbye email
        // - Update analytics
        
        return response()->json(['message' => 'User deleted']);
    }
}

// ✅ Business logic এর অংশ হিসেবে
class PaymentController extends Controller
{
    public function processPayment(Request $request)
    {
        $payment = Payment::create($request->validated());
        
        if ($payment->status === 'completed') {
            // Payment success এর সাথে সাথে এগুলো হতেই হবে
            event(new PaymentCompleted($payment));
            // - Update order status
            // - Send receipt
            // - Update user credits
        }
        
        return response()->json(['payment' => $payment]);
    }
}

// ✅ Real-time notifications
class PostController extends Controller
{
    public function createPost(Request $request)
    {
        $post = Post::create($request->validated());
        
        // Followers দের তাৎক্ষণিক notification
        event(new PostCreated($post));
        
        return response()->json(['post' => $post]);
    }
}
```

### 🚀 Jobs & Queues ব্যবহার করুন যখন:

```php
<?php
// ✅ Time-consuming tasks
class ReportController extends Controller
{
    public function generateReport(Request $request)
    {
        // Heavy calculation - background এ করা ভালো
        GenerateMonthlyReportJob::dispatch($request->month, $request->year);
        
        return response()->json(['message' => 'Report generation started']);
    }
}

// ✅ External API calls
class NotificationController extends Controller
{
    public function sendPushNotification(Request $request)
    {
        $users = User::whereIn('id', $request->user_ids)->get();
        
        foreach ($users as $user) {
            // External API call - queue এ করা ভালো
            SendPushNotificationJob::dispatch($user, $request->message);
        }
        
        return response()->json(['message' => 'Notifications queued']);
    }
}

// ✅ Bulk operations
class EmailController extends Controller
{
    public function sendNewsletter(Request $request)
    {
        $subscribers = Subscriber::active()->get();
        
        // হাজার হাজার email - definitely queue
        foreach ($subscribers as $subscriber) {
            SendNewsletterJob::dispatch($subscriber, $request->newsletter_id);
        }
        
        return response()->json(['message' => "{$subscribers->count()} emails queued"]);
    }
}

// ✅ Scheduled tasks
class MaintenanceController extends Controller
{
    public function scheduleCleanup()
    {
        // 1 ঘন্টা পর cleanup করা
        CleanupTempFilesJob::dispatch()->delay(now()->addHour());
        
        // প্রতিদিন রাত 2টায় backup
        DatabaseBackupJob::dispatch()->delay(now()->setTime(2, 0));
        
        return response()->json(['message' => 'Maintenance scheduled']);
    }
}
```

---

## Performance তুলনা

### ⏱️ Response Time Test:

```php
<?php
class PerformanceTestController extends Controller
{
    public function testSynchronousEvents()
    {
        $start = microtime(true);
        
        // 5টি heavy listeners
        event(new HeavyProcessEvent());
        
        $end = microtime(true);
        $duration = ($end - $start) * 1000;
        
        return response()->json([
            'method' => 'Synchronous Events',
            'duration_ms' => $duration,
            'user_wait_time' => $duration . 'ms'
        ]);
    }
    
    public function testAsynchronousJobs()
    {
        $start = microtime(true);
        
        // 5টি heavy jobs
        for ($i = 1; $i <= 5; $i++) {
            HeavyProcessJob::dispatch();
        }
        
        $end = microtime(true);
        $duration = ($end - $start) * 1000;
        
        return response()->json([
            'method' => 'Asynchronous Jobs',
            'duration_ms' => $duration,
            'user_wait_time' => $duration . 'ms'
        ]);
    }
}

// Heavy Listener
class HeavyProcessListener
{
    public function handle(HeavyProcessEvent $event)
    {
        sleep(2); // 2 সেকেন্ড কাজ
    }
}

// Heavy Job  
class HeavyProcessJob implements ShouldQueue
{
    public function handle()
    {
        sleep(2); // 2 সেকেন্ড কাজ
    }
}
```

**Performance Results:**
```json
// Synchronous Events
{
    "method": "Synchronous Events",
    "duration_ms": 10000,
    "user_wait_time": "10000ms"
}

// Asynchronous Jobs
{
    "method": "Asynchronous Jobs", 
    "duration_ms": 50,
    "user_wait_time": "50ms"
}
```

### 📊 Memory Usage:

```php
<?php
class MemoryTestController extends Controller
{
    public function testEventMemory()
    {
        $memoryBefore = memory_get_usage(true);
        
        // 1000টি events
        for ($i = 1; $i <= 1000; $i++) {
            event(new TestEvent($i));
        }
        
        $memoryAfter = memory_get_usage(true);
        $memoryUsed = ($memoryAfter - $memoryBefore) / 1024 / 1024;
        
        return response()->json([
            'method' => 'Events',
            'memory_used_mb' => $memoryUsed
        ]);
    }
    
    public function testJobMemory()
    {
        $memoryBefore = memory_get_usage(true);
        
        // 1000টি jobs
        for ($i = 1; $i <= 1000; $i++) {
            TestJob::dispatch($i);
        }
        
        $memoryAfter = memory_get_usage(true);
        $memoryUsed = ($memoryAfter - $memoryBefore) / 1024 / 1024;
        
        return response()->json([
            'method' => 'Jobs',
            'memory_used_mb' => $memoryUsed
        ]);
    }
}
```

---

## বাস্তব উদাহরণ

### 🛒 E-commerce Order Processing:

```php
<?php
// Events approach - সব কিছু synchronous
class OrderController extends Controller
{
    public function placeOrder(Request $request)
    {
        $order = Order::create($request->validated());
        
        // Event fire - সব listeners একসাথে চলবে
        event(new OrderPlaced($order));
        
        return response()->json(['order' => $order]);
    }
}

// Listeners
class SendOrderConfirmation
{
    public function handle(OrderPlaced $event)
    {
        // Email পাঠানো (2-3 সেকেন্ড)
        Mail::to($event->order->user->email)->send(new OrderConfirmationMail());
    }
}

class UpdateInventory  
{
    public function handle(OrderPlaced $event)
    {
        // Stock update (1 সেকেন্ড)
        foreach ($event->order->items as $item) {
            $item->product->decrement('stock', $item->quantity);
        }
    }
}

class ProcessPayment
{
    public function handle(OrderPlaced $event)
    {
        // Payment gateway call (3-5 সেকেন্ড)
        $paymentGateway = new PaymentGateway();
        $paymentGateway->charge($event->order->total, $event->order->payment_method);
    }
}

// Total response time: 6-9 সেকেন্ড 😰
```

```php
<?php
// Jobs approach - background processing
class OrderController extends Controller
{
    public function placeOrder(Request $request)
    {
        $order = Order::create($request->validated());
        
        // Jobs dispatch - background এ চলবে
        SendOrderConfirmationJob::dispatch($order);
        UpdateInventoryJob::dispatch($order);
        ProcessPaymentJob::dispatch($order);
        
        return response()->json(['order' => $order]);
    }
}

// Jobs
class SendOrderConfirmationJob implements ShouldQueue
{
    public function handle()
    {
        Mail::to($this->order->user->email)->send(new OrderConfirmationMail());
    }
}

class UpdateInventoryJob implements ShouldQueue
{
    public function handle()
    {
        foreach ($this->order->items as $item) {
            $item->product->decrement('stock', $item->quantity);
        }
    }
}

class ProcessPaymentJob implements ShouldQueue
{
    public function handle()
    {
        $paymentGateway = new PaymentGateway();
        $paymentGateway->charge($this->order->total, $this->order->payment_method);
    }
}

// Total response time: 50-100ms 🚀
```

### 📧 Newsletter System:

```php
<?php
// Events approach - সব subscribers এক সাথে
class NewsletterController extends Controller
{
    public function sendNewsletter(Request $request)
    {
        $newsletter = Newsletter::create($request->validated());
        
        // Event fire - সব subscribers এর জন্য email পাঠাবে
        event(new NewsletterCreated($newsletter));
        
        return response()->json(['newsletter' => $newsletter]);
    }
}

class SendToAllSubscribers
{
    public function handle(NewsletterCreated $event)
    {
        $subscribers = Subscriber::active()->get(); // 10,000 subscribers
        
        foreach ($subscribers as $subscriber) {
            Mail::to($subscriber->email)->send(new NewsletterMail($event->newsletter));
        }
        // 10,000 emails = 30-60 মিনিট 😱
    }
}
```

```php
<?php
// Jobs approach - batch processing
class NewsletterController extends Controller
{
    public function sendNewsletter(Request $request)
    {
        $newsletter = Newsletter::create($request->validated());
        
        $subscribers = Subscriber::active()->get();
        
        // প্রতিটি subscriber এর জন্য আলাদা job
        foreach ($subscribers as $subscriber) {
            SendNewsletterToSubscriberJob::dispatch($newsletter, $subscriber);
        }
        
        return response()->json([
            'newsletter' => $newsletter,
            'queued_emails' => $subscribers->count()
        ]);
    }
}

class SendNewsletterToSubscriberJob implements ShouldQueue
{
    public function handle()
    {
        Mail::to($this->subscriber->email)->send(new NewsletterMail($this->newsletter));
    }
}

// Response time: 100-200ms
// Background processing: Parallel execution 🚀
```

### 🔔 Real-time Notifications:

```php
<?php
// Events approach - তাৎক্ষণিক notification
class CommentController extends Controller
{
    public function store(Request $request, Post $post)
    {
        $comment = $post->comments()->create($request->validated());
        
        // Event fire - post author কে তাৎক্ষণিক notification
        event(new CommentAdded($comment));
        
        return response()->json(['comment' => $comment]);
    }
}

class NotifyPostAuthor
{
    public function handle(CommentAdded $event)
    {
        // Real-time notification
        $event->comment->post->user->notify(new NewCommentNotification($event->comment));
        
        // Browser notification
        broadcast(new CommentAddedBroadcast($event->comment));
    }
}

// Perfect for real-time features! ✅
```

```php
<?php
// Jobs approach - delayed notification
class CommentController extends Controller
{
    public function store(Request $request, Post $post)
    {
        $comment = $post->comments()->create($request->validated());
        
        // Job dispatch - notification পরে আসবে
        NotifyPostAuthorJob::dispatch($comment);
        
        return response()->json(['comment' => $comment]);
    }
}

class NotifyPostAuthorJob implements ShouldQueue
{
    public function handle()
    {
        // Delayed notification - real-time এর জন্য ভালো না
        $this->comment->post->user->notify(new NewCommentNotification($this->comment));
    }
}

// Not ideal for real-time! ❌
```

---

## সারসংক্ষেপ

### 🎯 Events & Listeners:
- **কখন**: Business logic এর অংশ, real-time reactions
- **Execution**: Synchronous (default), line by line
- **Performance**: Slower response, higher memory usage
- **Use Cases**: User actions, model events, notifications

### 🚀 Jobs & Queues:
- **কখন**: Heavy tasks, external APIs, bulk operations  
- **Execution**: Asynchronous, background processing
- **Performance**: Faster response, scalable
- **Use Cases**: Email sending, file processing, reports

### 💡 Best Practice:
```php
<?php
// Hybrid approach - Events + Jobs
class OrderController extends Controller
{
    public function placeOrder(Request $request)
    {
        $order = Order::create($request->validated());
        
        // Critical business logic - Event (synchronous)
        event(new OrderPlaced($order)); // Inventory update, order status
        
        // Heavy tasks - Jobs (asynchronous)  
        SendOrderConfirmationJob::dispatch($order);
        ProcessPaymentJob::dispatch($order);
        GenerateInvoiceJob::dispatch($order);
        
        return response()->json(['order' => $order]);
    }
}
```

**Events ব্যবহার করুন** যখন কাজটি business logic এর অংশ এবং তাৎক্ষণিক হতে হবে।
**Jobs ব্যবহার করুন** যখন কাজটি ভারী এবং background এ করা যায়।