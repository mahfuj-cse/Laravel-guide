# 1️⃣5️⃣ Laravel Queues & Jobs - বিস্তারিত বাংলা গাইড

## 📋 সূচিপত্র
- [Queues & Jobs কি?](#queues--jobs-কি)
- [Queue Configuration](#queue-configuration)
- [Jobs তৈরি ও ব্যবহার](#jobs-তৈরি-ও-ব্যবহার)
- [Queue Drivers](#queue-drivers)
- [Job Processing](#job-processing)
- [Failed Jobs](#failed-jobs)
- [Advanced Features](#advanced-features)
- [সম্পূর্ণ উদাহরণ](#সম্পূর্ণ-উদাহরণ)

---

## Queues & Jobs কি?

**Queue** হলো **কাজের তালিকা** যেখানে **Jobs** গুলো সারিবদ্ধভাবে থাকে। **Job** হলো **একটি নির্দিষ্ট কাজ** যা **Background এ** চলে।

### কেন Queues ব্যবহার করবেন?
- ✅ **Slow Operations** background এ চালানো (Email, File Upload)
- ✅ **User Experience** উন্নত করা (Fast Response)
- ✅ **Server Load** কমানো
- ✅ **Scalability** বৃদ্ধি করা

### Queue vs Synchronous:
```php
// Synchronous (ধীর)
public function register(Request $request)
{
    $user = User::create($request->all());
    Mail::to($user)->send(new WelcomeEmail($user)); // 2-3 সেকেন্ড অপেক্ষা
    return response()->json(['message' => 'User registered']);
}

// Asynchronous with Queue (দ্রুত)
public function register(Request $request)
{
    $user = User::create($request->all());
    SendWelcomeEmail::dispatch($user); // তাৎক্ষণিক
    return response()->json(['message' => 'User registered']);
}
```

---

## Queue Configuration

### ১. Queue Driver Setup:
```php
<?php
// config/queue.php

return [
    'default' => env('QUEUE_CONNECTION', 'sync'),

    'connections' => [
        'sync' => [
            'driver' => 'sync',
        ],

        'database' => [
            'driver' => 'database',
            'table' => 'jobs',
            'queue' => 'default',
            'retry_after' => 90,
            'after_commit' => false,
        ],

        'redis' => [
            'driver' => 'redis',
            'connection' => 'default',
            'queue' => env('REDIS_QUEUE', 'default'),
            'retry_after' => 90,
            'block_for' => null,
            'after_commit' => false,
        ],

        'beanstalkd' => [
            'driver' => 'beanstalkd',
            'host' => 'localhost',
            'queue' => 'default',
            'retry_after' => 90,
            'block_for' => 0,
            'after_commit' => false,
        ],
    ],
];
```

### ২. Environment Configuration:
```bash
# .env file

# Database Queue
QUEUE_CONNECTION=database

# Redis Queue
QUEUE_CONNECTION=redis
REDIS_HOST=127.0.0.1
REDIS_PASSWORD=null
REDIS_PORT=6379

# Beanstalkd Queue
QUEUE_CONNECTION=beanstalkd
```

### ৩. Database Queue Setup:
```bash
# Jobs table তৈরি
php artisan queue:table
php artisan migrate

# Failed jobs table তৈরি
php artisan queue:failed-table
php artisan migrate
```

---

## Jobs তৈরি ও ব্যবহার

### ১. Basic Job তৈরি:
```bash
# Job তৈরি
php artisan make:job SendWelcomeEmail
php artisan make:job ProcessPayment
php artisan make:job GenerateReport
```

### ২. Simple Job Example:
```php
<?php
// app/Jobs/SendWelcomeEmail.php

namespace App\Jobs;

use Illuminate\Bus\Queueable;
use Illuminate\Contracts\Queue\ShouldQueue;
use Illuminate\Foundation\Bus\Dispatchable;
use Illuminate\Queue\InteractsWithQueue;
use Illuminate\Queue\SerializesModels;
use Illuminate\Support\Facades\Mail;
use App\Models\User;
use App\Mail\WelcomeEmail;

class SendWelcomeEmail implements ShouldQueue
{
    use Dispatchable, InteractsWithQueue, Queueable, SerializesModels;

    protected $user;

    public function __construct(User $user)
    {
        $this->user = $user;
    }

    public function handle()
    {
        // Email পাঠানোর কাজ
        Mail::to($this->user->email)->send(new WelcomeEmail($this->user));
        
        // Log করা
        \Log::info("Welcome email sent to: " . $this->user->email);
    }
}
```

### ৩. Job Dispatch করা:
```php
<?php
// Controller এ Job dispatch

use App\Jobs\SendWelcomeEmail;

class UserController extends Controller
{
    public function register(Request $request)
    {
        $user = User::create($request->validated());

        // Job dispatch করা
        SendWelcomeEmail::dispatch($user);

        // অথবা
        dispatch(new SendWelcomeEmail($user));

        return response()->json([
            'message' => 'User registered successfully',
            'user' => $user
        ]);
    }

    public function sendBulkEmails()
    {
        $users = User::where('email_verified_at', null)->get();

        foreach ($users as $user) {
            // প্রতিটি user এর জন্য আলাদা job
            SendWelcomeEmail::dispatch($user);
        }

        return response()->json([
            'message' => "{$users->count()} emails queued"
        ]);
    }
}
```

### ৪. Job with Delay:
```php
<?php

class ReminderController extends Controller
{
    public function sendReminder(User $user)
    {
        // 1 ঘন্টা পর email পাঠানো
        SendReminderEmail::dispatch($user)->delay(now()->addHour());

        // 3 দিন পর email পাঠানো
        SendFollowUpEmail::dispatch($user)->delay(now()->addDays(3));

        return response()->json(['message' => 'Reminder scheduled']);
    }
}
```

---

## Queue Drivers

### ১. Database Driver:
```php
<?php
// Database queue - সবচেয়ে সহজ

// Setup
QUEUE_CONNECTION=database

// Migration
php artisan queue:table
php artisan migrate

// Job dispatch
SendEmail::dispatch($user);

// Worker চালানো
php artisan queue:work
```

### ২. Redis Driver:
```php
<?php
// Redis queue - দ্রুত এবং scalable

// Setup
QUEUE_CONNECTION=redis

// Redis install করতে হবে
composer require predis/predis

// Job dispatch
SendEmail::dispatch($user)->onQueue('emails');

// Specific queue worker
php artisan queue:work redis --queue=emails,default
```

### ৩. Beanstalkd Driver:
```php
<?php
// Beanstalkd queue - high performance

// Setup
QUEUE_CONNECTION=beanstalkd

// Beanstalkd install করতে হবে
composer require pda/pheanstalk

// Job dispatch
SendEmail::dispatch($user)->onQueue('high-priority');

// Worker চালানো
php artisan queue:work beanstalkd
```

---

## Job Processing

### ১. Queue Worker চালানো:
```bash
# Basic worker
php artisan queue:work

# Specific connection
php artisan queue:work redis

# Specific queue
php artisan queue:work --queue=high,default

# Memory limit সহ
php artisan queue:work --memory=512

# Timeout সহ
php artisan queue:work --timeout=60

# Max jobs সহ
php artisan queue:work --max-jobs=1000

# Max time সহ
php artisan queue:work --max-time=3600
```

### ২. Job Batching:
```php
<?php
// app/Jobs/ProcessCsvImport.php

use Illuminate\Bus\Batchable;

class ProcessCsvImport implements ShouldQueue
{
    use Dispatchable, InteractsWithQueue, Queueable, SerializesModels, Batchable;

    protected $csvRow;

    public function __construct($csvRow)
    {
        $this->csvRow = $csvRow;
    }

    public function handle()
    {
        // Batch cancelled check
        if ($this->batch()->cancelled()) {
            return;
        }

        // CSV row process করা
        User::create([
            'name' => $this->csvRow['name'],
            'email' => $this->csvRow['email'],
        ]);
    }
}

// Controller এ batch dispatch
use Illuminate\Bus\Batch;
use Illuminate\Support\Facades\Bus;

public function importCsv(Request $request)
{
    $csvData = collect($request->csv_data);
    
    $jobs = $csvData->map(function ($row) {
        return new ProcessCsvImport($row);
    });

    $batch = Bus::batch($jobs)
        ->then(function (Batch $batch) {
            // All jobs completed successfully
            \Log::info('CSV import completed');
        })
        ->catch(function (Batch $batch, Throwable $e) {
            // First batch job failure
            \Log::error('CSV import failed: ' . $e->getMessage());
        })
        ->finally(function (Batch $batch) {
            // Batch finished executing
            \Log::info('CSV import batch finished');
        })
        ->dispatch();

    return response()->json([
        'batch_id' => $batch->id,
        'message' => 'Import started'
    ]);
}
```

### ৩. Job Chaining:
```php
<?php
// Jobs একের পর এক চালানো

use Illuminate\Support\Facades\Bus;

public function processOrder(Order $order)
{
    Bus::chain([
        new ProcessPayment($order),
        new SendOrderConfirmation($order),
        new UpdateInventory($order),
        new SendShippingNotification($order),
    ])->dispatch();

    return response()->json(['message' => 'Order processing started']);
}
```

---

## Failed Jobs

### ১. Failed Job Handling:
```php
<?php
// Job এ failed method যোগ করা

class SendWelcomeEmail implements ShouldQueue
{
    use Dispatchable, InteractsWithQueue, Queueable, SerializesModels;

    public $tries = 3; // 3 বার চেষ্টা করবে
    public $maxExceptions = 2; // 2টি exception এর পর fail
    public $timeout = 120; // 2 মিনিট timeout

    public function handle()
    {
        // Email পাঠানোর কাজ
        Mail::to($this->user->email)->send(new WelcomeEmail($this->user));
    }

    public function failed(Throwable $exception)
    {
        // Job fail হলে এই method চলবে
        \Log::error('Welcome email failed for user: ' . $this->user->id);
        \Log::error('Error: ' . $exception->getMessage());

        // Admin কে notification পাঠানো
        AdminNotification::dispatch('Email job failed', $exception->getMessage());
    }
}
```

### ২. Failed Jobs Commands:
```bash
# Failed jobs দেখা
php artisan queue:failed

# Failed job retry করা
php artisan queue:retry 1

# সব failed jobs retry করা
php artisan queue:retry all

# Failed job delete করা
php artisan queue:forget 1

# সব failed jobs clear করা
php artisan queue:flush
```

### ৩. Job Retry Configuration:
```php
<?php

class ProcessPayment implements ShouldQueue
{
    public $tries = 5; // 5 বার চেষ্টা
    public $maxExceptions = 3; // 3টি exception
    public $backoff = [1, 5, 10]; // Retry delay (seconds)
    public $timeout = 300; // 5 মিনিট timeout

    public function retryUntil()
    {
        // 1 দিন পর্যন্ত retry করবে
        return now()->addDay();
    }

    public function handle()
    {
        // Payment processing logic
        if (!$this->processPayment()) {
            throw new \Exception('Payment processing failed');
        }
    }
}
```

---

## Advanced Features

### ১. Job Middleware:
```php
<?php
// app/Jobs/Middleware/RateLimited.php

class RateLimited
{
    public function handle($job, $next)
    {
        Redis::throttle('key')
            ->block(0)->allow(10)->every(60)
            ->then(function () use ($job, $next) {
                $next($job);
            }, function () use ($job) {
                $job->release(10); // 10 সেকেন্ড পর retry
            });
    }
}

// Job এ middleware ব্যবহার
class SendEmail implements ShouldQueue
{
    public function middleware()
    {
        return [new RateLimited];
    }

    public function handle()
    {
        // Email sending logic
    }
}
```

### ২. Unique Jobs:
```php
<?php
// Duplicate job prevent করা

use Illuminate\Contracts\Queue\ShouldBeUnique;

class ProcessUserData implements ShouldQueue, ShouldBeUnique
{
    protected $user;

    public function __construct(User $user)
    {
        $this->user = $user;
    }

    public function uniqueId()
    {
        return $this->user->id;
    }

    public function handle()
    {
        // User data processing
    }
}
```

### ৩. Job Events:
```php
<?php
// app/Providers/EventServiceProvider.php

use Illuminate\Queue\Events\JobProcessed;
use Illuminate\Queue\Events\JobFailed;

class EventServiceProvider extends ServiceProvider
{
    public function boot()
    {
        Queue::after(function (JobProcessed $event) {
            // Job completed
            \Log::info('Job completed: ' . $event->job->resolveName());
        });

        Queue::failing(function (JobFailed $event) {
            // Job failed
            \Log::error('Job failed: ' . $event->job->resolveName());
        });
    }
}
```

---

## সম্পূর্ণ উদাহরণ

### ১. E-commerce Order Processing:
```php
<?php
// Complete order processing system

// 1. Payment Processing Job
class ProcessPayment implements ShouldQueue
{
    use Dispatchable, InteractsWithQueue, Queueable, SerializesModels;

    protected $order;
    public $tries = 3;
    public $timeout = 300;

    public function __construct(Order $order)
    {
        $this->order = $order;
    }

    public function handle()
    {
        // Payment gateway integration
        $paymentResult = PaymentGateway::charge([
            'amount' => $this->order->total,
            'currency' => 'USD',
            'source' => $this->order->payment_token,
        ]);

        if ($paymentResult->success) {
            $this->order->update(['status' => 'paid']);
            
            // Chain next jobs
            SendOrderConfirmation::dispatch($this->order);
            UpdateInventory::dispatch($this->order);
        } else {
            throw new PaymentException('Payment failed: ' . $paymentResult->error);
        }
    }

    public function failed(Throwable $exception)
    {
        $this->order->update(['status' => 'payment_failed']);
        SendPaymentFailedNotification::dispatch($this->order, $exception->getMessage());
    }
}

// 2. Order Confirmation Job
class SendOrderConfirmation implements ShouldQueue
{
    protected $order;

    public function __construct(Order $order)
    {
        $this->order = $order;
    }

    public function handle()
    {
        Mail::to($this->order->user->email)
            ->send(new OrderConfirmationMail($this->order));

        // SMS notification
        if ($this->order->user->phone) {
            SMS::send($this->order->user->phone, 
                "Your order #{$this->order->id} has been confirmed!");
        }
    }
}

// 3. Inventory Update Job
class UpdateInventory implements ShouldQueue
{
    protected $order;

    public function __construct(Order $order)
    {
        $this->order = $order;
    }

    public function handle()
    {
        foreach ($this->order->items as $item) {
            $product = Product::find($item->product_id);
            $product->decrement('stock', $item->quantity);

            // Low stock alert
            if ($product->stock < 10) {
                LowStockAlert::dispatch($product);
            }
        }
    }
}

// Controller এ order processing
class OrderController extends Controller
{
    public function store(Request $request)
    {
        $order = Order::create($request->validated());

        // Start order processing chain
        Bus::chain([
            new ProcessPayment($order),
            new SendOrderConfirmation($order),
            new UpdateInventory($order),
            function () use ($order) {
                // Final step - mark as processing
                $order->update(['status' => 'processing']);
            }
        ])->catch(function (Throwable $e) use ($order) {
            // Handle chain failure
            $order->update(['status' => 'failed']);
            \Log::error('Order processing failed: ' . $e->getMessage());
        })->dispatch();

        return response()->json([
            'message' => 'Order placed successfully',
            'order_id' => $order->id
        ]);
    }
}
```

### ২. Bulk Email Campaign:
```php
<?php
// Bulk email system

class SendCampaignEmail implements ShouldQueue, ShouldBeUnique
{
    use Dispatchable, InteractsWithQueue, Queueable, SerializesModels, Batchable;

    protected $campaign;
    protected $user;
    public $tries = 3;

    public function __construct(Campaign $campaign, User $user)
    {
        $this->campaign = $campaign;
        $this->user = $user;
    }

    public function uniqueId()
    {
        return $this->campaign->id . '-' . $this->user->id;
    }

    public function handle()
    {
        if ($this->batch()->cancelled()) {
            return;
        }

        // Check if user unsubscribed
        if ($this->user->unsubscribed) {
            return;
        }

        Mail::to($this->user->email)
            ->send(new CampaignEmail($this->campaign, $this->user));

        // Track email sent
        CampaignStat::create([
            'campaign_id' => $this->campaign->id,
            'user_id' => $this->user->id,
            'status' => 'sent',
            'sent_at' => now(),
        ]);
    }
}

// Campaign Controller
class CampaignController extends Controller
{
    public function send(Campaign $campaign)
    {
        $users = User::where('subscribed', true)->get();
        
        $jobs = $users->map(function ($user) use ($campaign) {
            return new SendCampaignEmail($campaign, $user);
        });

        $batch = Bus::batch($jobs)
            ->then(function (Batch $batch) use ($campaign) {
                $campaign->update(['status' => 'completed']);
                \Log::info("Campaign {$campaign->id} completed successfully");
            })
            ->catch(function (Batch $batch, Throwable $e) use ($campaign) {
                $campaign->update(['status' => 'failed']);
                \Log::error("Campaign {$campaign->id} failed: " . $e->getMessage());
            })
            ->name("Campaign: {$campaign->name}")
            ->onQueue('emails')
            ->dispatch();

        return response()->json([
            'message' => 'Campaign started',
            'batch_id' => $batch->id,
            'total_emails' => $users->count()
        ]);
    }
}
```

### ৩. Queue Monitoring:
```php
<?php
// Queue monitoring system

class QueueMonitorController extends Controller
{
    public function stats()
    {
        $stats = [
            'pending_jobs' => DB::table('jobs')->count(),
            'failed_jobs' => DB::table('failed_jobs')->count(),
            'processed_jobs' => Cache::get('processed_jobs_count', 0),
        ];

        return response()->json($stats);
    }

    public function retryAllFailed()
    {
        Artisan::call('queue:retry all');
        
        return response()->json([
            'message' => 'All failed jobs have been retried'
        ]);
    }
}

// Queue monitoring middleware
class QueueStatsMiddleware
{
    public function handle($job, $next)
    {
        $startTime = microtime(true);
        
        $next($job);
        
        $executionTime = microtime(true) - $startTime;
        
        // Log job stats
        \Log::info('Job executed', [
            'job' => $job->resolveName(),
            'execution_time' => $executionTime,
            'memory_usage' => memory_get_peak_usage(true),
        ]);
        
        // Increment processed jobs counter
        Cache::increment('processed_jobs_count');
    }
}
```

---

## 🎯 Best Practices:

### Performance Tips:
- ✅ **Redis** ব্যবহার করুন high-traffic এর জন্য
- ✅ **Job batching** ব্যবহার করুন bulk operations এর জন্য
- ✅ **Queue priorities** সেট করুন
- ✅ **Worker memory limits** সেট করুন
- ✅ **Supervisor** ব্যবহার করুন production এ

### Security Tips:
- ✅ **Job data** validate করুন
- ✅ **Sensitive data** serialize করবেন না
- ✅ **Rate limiting** implement করুন
- ✅ **Failed jobs** monitor করুন

### Monitoring Tips:
- ✅ **Queue size** monitor করুন
- ✅ **Job execution time** track করুন
- ✅ **Failed jobs** alert setup করুন
- ✅ **Worker health** check করুন

---

## 📚 আরও জানতে:
- [Laravel Queues](https://laravel.com/docs/queues)
- [Job Batching](https://laravel.com/docs/queues#job-batching)
- [Queue Workers](https://laravel.com/docs/queues#running-the-queue-worker)