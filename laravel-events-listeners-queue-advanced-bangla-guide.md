# Laravel Events, Listeners ও Advanced Queue Management - সম্পূর্ণ বাংলা গাইড

## 📋 সূচিপত্র
- [Events ও Listeners কি?](#events-ও-listeners-কি)
- [Event-Listener Setup](#event-listener-setup)
- [Queueable Events ও Listeners](#queueable-events-ও-listeners)
- [Queue Priority Management](#queue-priority-management)
- [Failed Job Handling](#failed-job-handling)
- [Production Queue Monitoring](#production-queue-monitoring)
- [Job Table Management](#job-table-management)
- [Advanced Queue Strategies](#advanced-queue-strategies)
- [Performance Optimization](#performance-optimization)

---

## Events ও Listeners কি?

### 🎯 **Event-Driven Architecture**

**Events** হলো application এ ঘটে যাওয়া specific actions বা occurrences। **Listeners** হলো সেই events এর response এ execute হওয়া code।

### 🔥 **কেন ব্যবহার করবেন?**

- ✅ **Decoupled Code:** Components আলাদাভাবে কাজ করে
- ✅ **Scalability:** Multiple listeners একই event শুনতে পারে
- ✅ **Maintainability:** Code organization ভালো হয়
- ✅ **Testability:** Individual components test করা সহজ
- ✅ **Flexibility:** Runtime এ listeners add/remove করা যায়

### 📊 **Real-world Examples:**

```php
// User registration হলে যা যা করতে হয়:
UserRegistered::class => [
    SendWelcomeEmail::class,      // Welcome email পাঠানো
    CreateUserProfile::class,     // Profile তৈরি করা
    AssignDefaultRole::class,     // Default role assign করা
    LogUserActivity::class,       // Activity log করা
    NotifyAdmins::class,         // Admin দের জানানো
]
```

---

## Event-Listener Setup

### 🎪 **Event তৈরি করা**

```bash
# Event class তৈরি
php artisan make:event UserRegistered
php artisan make:event OrderPlaced
php artisan make:event PaymentProcessed
```

**Event Class Structure:**
```php
<?php
// app/Events/UserRegistered.php

namespace App\Events;

use App\Models\User;
use Illuminate\Broadcasting\InteractsWithSockets;
use Illuminate\Foundation\Events\Dispatchable;
use Illuminate\Queue\SerializesModels;

class UserRegistered
{
    use Dispatchable, InteractsWithSockets, SerializesModels;

    public $user;
    public $registrationData;

    public function __construct(User $user, array $registrationData = [])
    {
        $this->user = $user;
        $this->registrationData = $registrationData;
    }

    /**
     * Broadcasting এর জন্য (optional)
     */
    public function broadcastOn()
    {
        return new PrivateChannel('users.' . $this->user->id);
    }
}
```

### 🎧 **Listener তৈরি করা**

```bash
# Listener class তৈরি
php artisan make:listener SendWelcomeEmail --event=UserRegistered
php artisan make:listener CreateUserProfile --event=UserRegistered
php artisan make:listener NotifyAdmins --event=UserRegistered
```

**Listener Class Structure:**
```php
<?php
// app/Listeners/SendWelcomeEmail.php

namespace App\Listeners;

use App\Events\UserRegistered;
use App\Mail\WelcomeEmail;
use Illuminate\Contracts\Queue\ShouldQueue;
use Illuminate\Queue\InteractsWithQueue;
use Illuminate\Support\Facades\Mail;

class SendWelcomeEmail implements ShouldQueue
{
    use InteractsWithQueue;

    public $tries = 3;           // 3 বার try করবে
    public $timeout = 120;       // 2 মিনিট timeout
    public $backoff = [10, 30];  // Retry delay

    public function handle(UserRegistered $event)
    {
        try {
            Mail::to($event->user->email)
                ->send(new WelcomeEmail($event->user));
                
            \Log::info('Welcome email sent', [
                'user_id' => $event->user->id,
                'email' => $event->user->email
            ]);
            
        } catch (\Exception $e) {
            \Log::error('Welcome email failed', [
                'user_id' => $event->user->id,
                'error' => $e->getMessage()
            ]);
            
            // Re-throw to trigger retry
            throw $e;
        }
    }

    /**
     * Job fail হলে কি করবে
     */
    public function failed(UserRegistered $event, \Throwable $exception)
    {
        \Log::error('Welcome email permanently failed', [
            'user_id' => $event->user->id,
            'error' => $exception->getMessage(),
            'attempts' => $this->attempts()
        ]);

        // Admin notification
        Mail::to('admin@company.com')->send(
            new FailedJobNotification($event, $exception)
        );
    }
}
```

### 📝 **Event-Listener Registration**

**EventServiceProvider এ register করা:**
```php
<?php
// app/Providers/EventServiceProvider.php

namespace App\Providers;

use App\Events\UserRegistered;
use App\Events\OrderPlaced;
use App\Events\PaymentProcessed;
use App\Listeners\SendWelcomeEmail;
use App\Listeners\CreateUserProfile;
use App\Listeners\NotifyAdmins;
use App\Listeners\ProcessOrderItems;
use App\Listeners\UpdateInventory;
use App\Listeners\SendPaymentReceipt;
use Illuminate\Foundation\Support\Providers\EventServiceProvider as ServiceProvider;

class EventServiceProvider extends ServiceProvider
{
    protected $listen = [
        // User Registration Events
        UserRegistered::class => [
            SendWelcomeEmail::class,
            CreateUserProfile::class,
            NotifyAdmins::class,
        ],

        // Order Events
        OrderPlaced::class => [
            ProcessOrderItems::class,
            UpdateInventory::class,
            SendOrderConfirmation::class,
        ],

        // Payment Events
        PaymentProcessed::class => [
            SendPaymentReceipt::class,
            UpdateOrderStatus::class,
            TriggerShipping::class,
        ],
    ];

    public function boot()
    {
        parent::boot();

        // Dynamic event listeners
        Event::listen('user.*', function ($eventName, array $data) {
            \Log::info("User event fired: {$eventName}", $data);
        });
    }
}
```

### 🚀 **Event Dispatch করা**

**Controller থেকে:**
```php
<?php
// app/Http/Controllers/AuthController.php

class AuthController extends Controller
{
    public function register(Request $request)
    {
        $validated = $request->validate([
            'name' => 'required|string|max:255',
            'email' => 'required|email|unique:users',
            'password' => 'required|min:8|confirmed',
        ]);

        $user = User::create([
            'name' => $validated['name'],
            'email' => $validated['email'],
            'password' => Hash::make($validated['password']),
        ]);

        // Event dispatch করা
        UserRegistered::dispatch($user, $validated);

        return response()->json([
            'message' => 'User registered successfully',
            'user' => $user
        ], 201);
    }
}
```

**Model Events থেকে:**
```php
<?php
// app/Models/User.php

class User extends Authenticatable
{
    protected static function booted()
    {
        static::created(function ($user) {
            UserRegistered::dispatch($user);
        });

        static::updated(function ($user) {
            if ($user->wasChanged('email')) {
                EmailChanged::dispatch($user, $user->getOriginal('email'));
            }
        });

        static::deleted(function ($user) {
            UserDeleted::dispatch($user);
        });
    }
}
```

---

## Queueable Events ও Listeners

### 🔄 **ShouldQueue Interface**

**Queueable Listener:**
```php
<?php

namespace App\Listeners;

use Illuminate\Contracts\Queue\ShouldQueue;
use Illuminate\Queue\InteractsWithQueue;

class ProcessLargeDataset implements ShouldQueue
{
    use InteractsWithQueue;

    public $tries = 5;
    public $timeout = 600;        // 10 minutes
    public $maxExceptions = 3;
    public $backoff = [60, 300, 900]; // 1min, 5min, 15min

    public function handle($event)
    {
        // Heavy processing
        $this->processData($event->data);
    }

    /**
     * Determine the time at which the job should timeout
     */
    public function retryUntil()
    {
        return now()->addHours(2);
    }

    /**
     * Calculate the number of seconds to wait before retrying
     */
    public function backoff()
    {
        return [60, 300, 900];
    }
}
```

### 🎯 **Queue Connection Specification**

```php
class SendWelcomeEmail implements ShouldQueue
{
    use InteractsWithQueue;

    public $connection = 'redis';    // Specific connection
    public $queue = 'emails';        // Specific queue name
    public $delay = 30;              // 30 seconds delay

    public function handle(UserRegistered $event)
    {
        // Email sending logic
    }
}
```

### 📊 **Conditional Queueing**

```php
class NotifyAdmins implements ShouldQueue
{
    use InteractsWithQueue;

    public function handle(UserRegistered $event)
    {
        // Skip queue for VIP users
        if ($event->user->is_vip) {
            $this->delete(); // Remove from queue
            $this->handleImmediately($event);
            return;
        }

        // Normal processing
        $this->sendNotification($event);
    }

    /**
     * Determine if the job should be queued
     */
    public function shouldQueue($event)
    {
        return !$event->user->is_vip;
    }
}
```

---

## Queue Priority Management

### 🏆 **Priority Queues Setup**

**Queue Configuration:**
```php
// config/queue.php

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

// Multiple queue names
'queues' => [
    'critical' => 'critical',    // Payment, security
    'high' => 'high',           // User notifications
    'default' => 'default',     // Normal operations
    'low' => 'low',            // Reports, cleanup
    'bulk' => 'bulk',          // Mass operations
]
```

### 🎯 **Priority-based Listeners**

```php
// Critical Priority - Payment Processing
class ProcessPayment implements ShouldQueue
{
    use InteractsWithQueue;

    public $connection = 'redis';
    public $queue = 'critical';
    public $tries = 5;
    public $timeout = 300;

    public function handle(PaymentRequested $event)
    {
        // Payment processing logic
    }
}

// High Priority - User Notifications
class SendUrgentNotification implements ShouldQueue
{
    use InteractsWithQueue;

    public $connection = 'redis';
    public $queue = 'high';
    public $tries = 3;
    public $timeout = 120;

    public function handle(UrgentAlert $event)
    {
        // Urgent notification logic
    }
}

// Low Priority - Analytics
class UpdateAnalytics implements ShouldQueue
{
    use InteractsWithQueue;

    public $connection = 'redis';
    public $queue = 'low';
    public $tries = 2;
    public $timeout = 600;

    public function handle(UserActivity $event)
    {
        // Analytics processing
    }
}
```

### ⚡ **Worker Commands with Priority**

```bash
# High priority worker (critical, high, default)
php artisan queue:work redis --queue=critical,high,default --sleep=1 --tries=3

# Normal worker (default, low)
php artisan queue:work redis --queue=default,low --sleep=3 --tries=3

# Bulk processing worker (bulk only)
php artisan queue:work redis --queue=bulk --sleep=5 --tries=2 --timeout=1800

# Multiple workers with different priorities
php artisan queue:work redis --queue=critical --sleep=0 --tries=5 &
php artisan queue:work redis --queue=high,default --sleep=2 --tries=3 &
php artisan queue:work redis --queue=low,bulk --sleep=5 --tries=2 &
```

### 🔧 **Supervisor Configuration**

```ini
# /etc/supervisor/conf.d/laravel-workers.conf

# Critical Queue Worker
[program:laravel-critical-worker]
process_name=%(program_name)s_%(process_num)02d
command=php /var/www/artisan queue:work redis --queue=critical --sleep=0 --tries=5 --max-time=3600
autostart=true
autorestart=true
stopasgroup=true
killasgroup=true
user=www-data
numprocs=2
redirect_stderr=true
stdout_logfile=/var/www/storage/logs/critical-worker.log
stopwaitsecs=3600

# High Priority Worker
[program:laravel-high-worker]
process_name=%(program_name)s_%(process_num)02d
command=php /var/www/artisan queue:work redis --queue=high,default --sleep=2 --tries=3 --max-time=3600
autostart=true
autorestart=true
stopasgroup=true
killasgroup=true
user=www-data
numprocs=4
redirect_stderr=true
stdout_logfile=/var/www/storage/logs/high-worker.log
stopwaitsecs=3600

# Low Priority Worker
[program:laravel-low-worker]
process_name=%(program_name)s_%(process_num)02d
command=php /var/www/artisan queue:work redis --queue=low,bulk --sleep=5 --tries=2 --max-time=7200
autostart=true
autorestart=true
stopasgroup=true
killasgroup=true
user=www-data
numprocs=2
redirect_stderr=true
stdout_logfile=/var/www/storage/logs/low-worker.log
stopwaitsecs=3600
```

---

## Failed Job Handling

### 🚨 **Failed Job Detection ও Recovery**

**Failed Job Table Setup:**
```bash
# Failed jobs table তৈরি
php artisan queue:failed-table
php artisan migrate
```

**Failed Job Monitoring:**
```php
<?php
// app/Console/Commands/MonitorFailedJobs.php

namespace App\Console\Commands;

use Illuminate\Console\Command;
use Illuminate\Support\Facades\DB;
use Illuminate\Support\Facades\Mail;
use App\Mail\FailedJobAlert;

class MonitorFailedJobs extends Command
{
    protected $signature = 'queue:monitor-failed';
    protected $description = 'Monitor and alert on failed jobs';

    public function handle()
    {
        $failedJobs = DB::table('failed_jobs')
            ->where('failed_at', '>', now()->subHour())
            ->get();

        if ($failedJobs->count() > 10) {
            $this->error("High failure rate detected: {$failedJobs->count()} jobs failed in last hour");
            
            // Send alert email
            Mail::to('admin@company.com')->send(
                new FailedJobAlert($failedJobs)
            );
        }

        // Analyze failure patterns
        $failuresByClass = $failedJobs->groupBy(function ($job) {
            $payload = json_decode($job->payload, true);
            return $payload['displayName'] ?? 'Unknown';
        });

        foreach ($failuresByClass as $class => $jobs) {
            if ($jobs->count() > 5) {
                $this->warn("Class {$class} has {$jobs->count()} failures");
            }
        }

        return 0;
    }
}
```

### 🔄 **Automatic Retry Strategies**

**Smart Retry Logic:**
```php
<?php
// app/Console/Commands/RetryFailedJobs.php

namespace App\Console\Commands;

use Illuminate\Console\Command;
use Illuminate\Support\Facades\DB;
use Illuminate\Support\Facades\Artisan;

class RetryFailedJobs extends Command
{
    protected $signature = 'queue:smart-retry {--class=} {--hours=24}';
    protected $description = 'Smart retry of failed jobs based on failure patterns';

    public function handle()
    {
        $hours = $this->option('hours');
        $class = $this->option('class');

        $query = DB::table('failed_jobs')
            ->where('failed_at', '>', now()->subHours($hours));

        if ($class) {
            $query->where('payload', 'like', "%{$class}%");
        }

        $failedJobs = $query->get();

        foreach ($failedJobs as $job) {
            $payload = json_decode($job->payload, true);
            $exception = $job->exception;

            // Skip if it's a permanent failure
            if ($this->isPermanentFailure($exception)) {
                $this->info("Skipping permanent failure: Job ID {$job->id}");
                continue;
            }

            // Retry with exponential backoff
            $retryDelay = $this->calculateRetryDelay($job);
            
            if ($retryDelay > 0) {
                $this->info("Retrying job {$job->id} with {$retryDelay}s delay");
                Artisan::call('queue:retry', ['id' => $job->id]);
            }
        }

        return 0;
    }

    private function isPermanentFailure($exception)
    {
        $permanentErrors = [
            'ValidationException',
            'ModelNotFoundException',
            'AuthenticationException',
            'InvalidArgumentException'
        ];

        foreach ($permanentErrors as $error) {
            if (strpos($exception, $error) !== false) {
                return true;
            }
        }

        return false;
    }

    private function calculateRetryDelay($job)
    {
        // Exponential backoff based on failure time
        $failedAt = \Carbon\Carbon::parse($job->failed_at);
        $hoursSinceFailed = $failedAt->diffInHours(now());

        if ($hoursSinceFailed < 1) return 0;      // Too recent
        if ($hoursSinceFailed < 6) return 300;    // 5 minutes
        if ($hoursSinceFailed < 24) return 1800;  // 30 minutes
        if ($hoursSinceFailed < 72) return 3600;  // 1 hour
        
        return 0; // Too old, don't retry
    }
}
```

### 📊 **Job ID দিয়ে Specific Job Handle করা**

**Job Details Retrieval:**
```php
<?php
// app/Console/Commands/JobDetails.php

namespace App\Console\Commands;

use Illuminate\Console\Command;
use Illuminate\Support\Facades\DB;

class JobDetails extends Command
{
    protected $signature = 'queue:job-details {id}';
    protected $description = 'Get detailed information about a specific job';

    public function handle()
    {
        $jobId = $this->argument('id');

        // Check in failed_jobs table
        $failedJob = DB::table('failed_jobs')->where('id', $jobId)->first();
        
        if ($failedJob) {
            $this->displayFailedJobDetails($failedJob);
            return 0;
        }

        // Check in jobs table (pending jobs)
        $pendingJob = DB::table('jobs')->where('id', $jobId)->first();
        
        if ($pendingJob) {
            $this->displayPendingJobDetails($pendingJob);
            return 0;
        }

        $this->error("Job with ID {$jobId} not found");
        return 1;
    }

    private function displayFailedJobDetails($job)
    {
        $payload = json_decode($job->payload, true);
        
        $this->info("=== Failed Job Details ===");
        $this->line("ID: {$job->id}");
        $this->line("UUID: {$job->uuid}");
        $this->line("Connection: {$job->connection}");
        $this->line("Queue: {$job->queue}");
        $this->line("Class: " . ($payload['displayName'] ?? 'Unknown'));
        $this->line("Failed At: {$job->failed_at}");
        
        $this->info("\n=== Job Data ===");
        $this->line(json_encode($payload['data'] ?? [], JSON_PRETTY_PRINT));
        
        $this->error("\n=== Exception ===");
        $this->line($job->exception);

        // Retry options
        if ($this->confirm('Do you want to retry this job?')) {
            \Artisan::call('queue:retry', ['id' => $job->id]);
            $this->info('Job queued for retry');
        }
    }

    private function displayPendingJobDetails($job)
    {
        $payload = json_decode($job->payload, true);
        
        $this->info("=== Pending Job Details ===");
        $this->line("ID: {$job->id}");
        $this->line("Queue: {$job->queue}");
        $this->line("Class: " . ($payload['displayName'] ?? 'Unknown'));
        $this->line("Attempts: {$job->attempts}");
        $this->line("Available At: " . date('Y-m-d H:i:s', $job->available_at));
        $this->line("Created At: " . date('Y-m-d H:i:s', $job->created_at));
        
        $this->info("\n=== Job Data ===");
        $this->line(json_encode($payload['data'] ?? [], JSON_PRETTY_PRINT));

        // Management options
        if ($this->confirm('Do you want to delete this job?')) {
            DB::table('jobs')->where('id', $job->id)->delete();
            $this->info('Job deleted');
        }
    }
}
```

---

## Production Queue Monitoring

### 📊 **Real-time Queue Dashboard**

**Queue Stats Controller:**
```php
<?php
// app/Http/Controllers/QueueMonitorController.php

namespace App\Http\Controllers;

use Illuminate\Http\Controller;
use Illuminate\Support\Facades\DB;
use Illuminate\Support\Facades\Redis;

class QueueMonitorController extends Controller
{
    public function dashboard()
    {
        $stats = [
            'pending_jobs' => $this->getPendingJobs(),
            'failed_jobs' => $this->getFailedJobs(),
            'processed_today' => $this->getProcessedToday(),
            'queue_lengths' => $this->getQueueLengths(),
            'worker_status' => $this->getWorkerStatus(),
            'failure_rate' => $this->getFailureRate(),
        ];

        return response()->json($stats);
    }

    private function getPendingJobs()
    {
        return DB::table('jobs')->count();
    }

    private function getFailedJobs()
    {
        return [
            'total' => DB::table('failed_jobs')->count(),
            'last_24h' => DB::table('failed_jobs')
                ->where('failed_at', '>', now()->subDay())
                ->count(),
            'last_hour' => DB::table('failed_jobs')
                ->where('failed_at', '>', now()->subHour())
                ->count(),
        ];
    }

    private function getProcessedToday()
    {
        // This requires custom tracking
        return \Cache::get('jobs_processed_today', 0);
    }

    private function getQueueLengths()
    {
        $queues = ['critical', 'high', 'default', 'low', 'bulk'];
        $lengths = [];

        foreach ($queues as $queue) {
            $lengths[$queue] = Redis::llen("queues:{$queue}");
        }

        return $lengths;
    }

    private function getWorkerStatus()
    {
        // Check if workers are running
        $workers = [];
        $processes = shell_exec('ps aux | grep "queue:work" | grep -v grep');
        
        if ($processes) {
            $lines = explode("\n", trim($processes));
            foreach ($lines as $line) {
                if (preg_match('/--queue=([^\s]+)/', $line, $matches)) {
                    $queue = $matches[1];
                    $workers[$queue] = ($workers[$queue] ?? 0) + 1;
                }
            }
        }

        return $workers;
    }

    private function getFailureRate()
    {
        $total = \Cache::get('jobs_total_today', 1);
        $failed = DB::table('failed_jobs')
            ->whereDate('failed_at', today())
            ->count();

        return round(($failed / $total) * 100, 2);
    }
}
```

### 🔔 **Alert System**

**Queue Alert Service:**
```php
<?php
// app/Services/QueueAlertService.php

namespace App\Services;

use Illuminate\Support\Facades\DB;
use Illuminate\Support\Facades\Mail;
use Illuminate\Support\Facades\Http;
use App\Mail\QueueAlert;

class QueueAlertService
{
    private $thresholds = [
        'high_failure_rate' => 10,      // 10% failure rate
        'queue_backup' => 100,          // 100+ jobs in queue
        'no_workers' => true,           // No workers running
        'old_failed_jobs' => 50,        // 50+ failed jobs
    ];

    public function checkAlerts()
    {
        $alerts = [];

        // Check failure rate
        if ($this->getFailureRate() > $this->thresholds['high_failure_rate']) {
            $alerts[] = [
                'type' => 'high_failure_rate',
                'message' => 'High job failure rate detected',
                'severity' => 'critical'
            ];
        }

        // Check queue backup
        $queueLengths = $this->getQueueLengths();
        foreach ($queueLengths as $queue => $length) {
            if ($length > $this->thresholds['queue_backup']) {
                $alerts[] = [
                    'type' => 'queue_backup',
                    'message' => "Queue '{$queue}' has {$length} pending jobs",
                    'severity' => 'warning'
                ];
            }
        }

        // Check worker status
        if (empty($this->getActiveWorkers())) {
            $alerts[] = [
                'type' => 'no_workers',
                'message' => 'No queue workers are running',
                'severity' => 'critical'
            ];
        }

        // Check old failed jobs
        $oldFailedJobs = DB::table('failed_jobs')
            ->where('failed_at', '<', now()->subDays(7))
            ->count();

        if ($oldFailedJobs > $this->thresholds['old_failed_jobs']) {
            $alerts[] = [
                'type' => 'old_failed_jobs',
                'message' => "{$oldFailedJobs} failed jobs older than 7 days",
                'severity' => 'info'
            ];
        }

        // Send alerts
        if (!empty($alerts)) {
            $this->sendAlerts($alerts);
        }

        return $alerts;
    }

    private function sendAlerts($alerts)
    {
        // Email alerts
        Mail::to('admin@company.com')->send(new QueueAlert($alerts));

        // Slack alerts for critical issues
        $criticalAlerts = array_filter($alerts, function($alert) {
            return $alert['severity'] === 'critical';
        });

        if (!empty($criticalAlerts)) {
            $this->sendSlackAlert($criticalAlerts);
        }
    }

    private function sendSlackAlert($alerts)
    {
        $message = "🚨 Critical Queue Issues:\n";
        foreach ($alerts as $alert) {
            $message .= "• " . $alert['message'] . "\n";
        }

        Http::post(config('services.slack.webhook_url'), [
            'text' => $message,
            'channel' => '#alerts',
            'username' => 'Queue Monitor'
        ]);
    }

    private function getFailureRate()
    {
        $total = \Cache::get('jobs_total_today', 1);
        $failed = DB::table('failed_jobs')
            ->whereDate('failed_at', today())
            ->count();

        return ($failed / $total) * 100;
    }

    private function getQueueLengths()
    {
        // Implementation from previous example
        return [];
    }

    private function getActiveWorkers()
    {
        // Implementation from previous example
        return [];
    }
}
```

### 📈 **Performance Metrics Tracking**

**Job Metrics Middleware:**
```php
<?php
// app/Jobs/Middleware/TrackJobMetrics.php

namespace App\Jobs\Middleware;

use Illuminate\Support\Facades\Redis;
use Illuminate\Support\Facades\Cache;

class TrackJobMetrics
{
    public function handle($job, $next)
    {
        $startTime = microtime(true);
        $jobClass = get_class($job);
        $date = now()->format('Y-m-d');

        // Increment job start counter
        Redis::incr("jobs:started:{$jobClass}:{$date}");
        Redis::incr("jobs:started:total:{$date}");

        try {
            $result = $next($job);

            // Track successful completion
            $duration = microtime(true) - $startTime;
            Redis::incr("jobs:completed:{$jobClass}:{$date}");
            Redis::lpush("jobs:duration:{$jobClass}:{$date}", $duration);

            // Keep only last 100 durations
            Redis::ltrim("jobs:duration:{$jobClass}:{$date}", 0, 99);

            return $result;

        } catch (\Exception $e) {
            // Track failure
            Redis::incr("jobs:failed:{$jobClass}:{$date}");
            Redis::lpush("jobs:errors:{$jobClass}:{$date}", $e->getMessage());

            throw $e;
        }
    }
}
```

**Job Performance Analysis:**
```php
<?php
// app/Console/Commands/AnalyzeJobPerformance.php

namespace App\Console\Commands;

use Illuminate\Console\Command;
use Illuminate\Support\Facades\Redis;

class AnalyzeJobPerformance extends Command
{
    protected $signature = 'queue:analyze-performance {--days=7}';
    protected $description = 'Analyze job performance metrics';

    public function handle()
    {
        $days = $this->option('days');
        $analysis = [];

        for ($i = 0; $i < $days; $i++) {
            $date = now()->subDays($i)->format('Y-m-d');
            $analysis[$date] = $this->analyzeDayPerformance($date);
        }

        $this->displayAnalysis($analysis);
        return 0;
    }

    private function analyzeDayPerformance($date)
    {
        $jobClasses = $this->getJobClasses($date);
        $dayAnalysis = [];

        foreach ($jobClasses as $jobClass) {
            $started = Redis::get("jobs:started:{$jobClass}:{$date}") ?: 0;
            $completed = Redis::get("jobs:completed:{$jobClass}:{$date}") ?: 0;
            $failed = Redis::get("jobs:failed:{$jobClass}:{$date}") ?: 0;

            $durations = Redis::lrange("jobs:duration:{$jobClass}:{$date}", 0, -1);
            $avgDuration = !empty($durations) ? array_sum($durations) / count($durations) : 0;

            $dayAnalysis[$jobClass] = [
                'started' => $started,
                'completed' => $completed,
                'failed' => $failed,
                'success_rate' => $started > 0 ? ($completed / $started) * 100 : 0,
                'avg_duration' => round($avgDuration, 2),
            ];
        }

        return $dayAnalysis;
    }

    private function displayAnalysis($analysis)
    {
        foreach ($analysis as $date => $dayData) {
            $this->info("\n=== {$date} ===");
            
            $table = [];
            foreach ($dayData as $jobClass => $metrics) {
                $table[] = [
                    'Job Class' => class_basename($jobClass),
                    'Started' => $metrics['started'],
                    'Completed' => $metrics['completed'],
                    'Failed' => $metrics['failed'],
                    'Success Rate' => round($metrics['success_rate'], 1) . '%',
                    'Avg Duration' => $metrics['avg_duration'] . 's',
                ];
            }

            $this->table([
                'Job Class', 'Started', 'Completed', 'Failed', 'Success Rate', 'Avg Duration'
            ], $table);
        }
    }

    private function getJobClasses($date)
    {
        $keys = Redis::keys("jobs:started:*:{$date}");
        $classes = [];

        foreach ($keys as $key) {
            if (preg_match('/jobs:started:(.+):\d{4}-\d{2}-\d{2}$/', $key, $matches)) {
                $classes[] = $matches[1];
            }
        }

        return array_unique($classes);
    }
}
```

---

## Job Table Management

### 🗄️ **Database Optimization**

**Job Table Cleanup:**
```php
<?php
// app/Console/Commands/CleanupJobTables.php

namespace App\Console\Commands;

use Illuminate\Console\Command;
use Illuminate\Support\Facades\DB;

class CleanupJobTables extends Command
{
    protected $signature = 'queue:cleanup {--days=30} {--dry-run}';
    protected $description = 'Cleanup old job records';

    public function handle()
    {
        $days = $this->option('days');
        $dryRun = $this->option('dry-run');

        $this->info("Cleaning up job records older than {$days} days...");

        // Cleanup failed jobs
        $failedJobsQuery = DB::table('failed_jobs')
            ->where('failed_at', '<', now()->subDays($days));

        $failedCount = $failedJobsQuery->count();
        
        if ($dryRun) {
            $this->info("Would delete {$failedCount} failed jobs");
        } else {
            $deleted = $failedJobsQuery->delete();
            $this->info("Deleted {$deleted} failed jobs");
        }

        // Archive old failed jobs before deletion
        if (!$dryRun && $failedCount > 0) {
            $this->archiveFailedJobs($days);
        }

        // Cleanup job batches (if using job batching)
        if (Schema::hasTable('job_batches')) {
            $batchesQuery = DB::table('job_batches')
                ->where('created_at', '<', now()->subDays($days));

            $batchCount = $batchesQuery->count();
            
            if ($dryRun) {
                $this->info("Would delete {$batchCount} job batches");
            } else {
                $deleted = $batchesQuery->delete();
                $this->info("Deleted {$deleted} job batches");
            }
        }

        return 0;
    }

    private function archiveFailedJobs($days)
    {
        $archiveTable = 'failed_jobs_archive';
        
        // Create archive table if not exists
        if (!Schema::hasTable($archiveTable)) {
            Schema::create($archiveTable, function (Blueprint $table) {
                $table->id();
                $table->string('uuid')->unique();
                $table->text('connection');
                $table->text('queue');
                $table->longText('payload');
                $table->longText('exception');
                $table->timestamp('failed_at')->useCurrent();
                $table->timestamp('archived_at')->useCurrent();
            });
        }

        // Move old failed jobs to archive
        DB::statement("
            INSERT INTO {$archiveTable} (uuid, connection, queue, payload, exception, failed_at, archived_at)
            SELECT uuid, connection, queue, payload, exception, failed_at, NOW()
            FROM failed_jobs 
            WHERE failed_at < ?
        ", [now()->subDays($days)]);

        $this->info("Archived old failed jobs to {$archiveTable}");
    }
}
```

### 📊 **Job Table Indexing**

**Performance Indexes:**
```php
<?php
// database/migrations/add_job_table_indexes.php

use Illuminate\Database\Migrations\Migration;
use Illuminate\Database\Schema\Blueprint;
use Illuminate\Support\Facades\Schema;

class AddJobTableIndexes extends Migration
{
    public function up()
    {
        Schema::table('jobs', function (Blueprint $table) {
            $table->index(['queue', 'available_at'], 'jobs_queue_available_at_index');
            $table->index(['available_at'], 'jobs_available_at_index');
            $table->index(['created_at'], 'jobs_created_at_index');
        });

        Schema::table('failed_jobs', function (Blueprint $table) {
            $table->index(['failed_at'], 'failed_jobs_failed_at_index');
            $table->index(['queue'], 'failed_jobs_queue_index');
        });
    }

    public function down()
    {
        Schema::table('jobs', function (Blueprint $table) {
            $table->dropIndex('jobs_queue_available_at_index');
            $table->dropIndex('jobs_available_at_index');
            $table->dropIndex('jobs_created_at_index');
        });

        Schema::table('failed_jobs', function (Blueprint $table) {
            $table->dropIndex('failed_jobs_failed_at_index');
            $table->dropIndex('failed_jobs_queue_index');
        });
    }
}
```

---

## Advanced Queue Strategies

### 🎯 **Job Batching**

**Batch Job Implementation:**
```php
<?php
// app/Jobs/ProcessUsersBatch.php

namespace App\Jobs;

use Illuminate\Bus\Batchable;
use Illuminate\Bus\Queueable;
use Illuminate\Contracts\Queue\ShouldQueue;
use Illuminate\Foundation\Bus\Dispatchable;
use Illuminate\Queue\InteractsWithQueue;
use Illuminate\Queue\SerializesModels;

class ProcessUsersBatch implements ShouldQueue
{
    use Batchable, Dispatchable, InteractsWithQueue, Queueable, SerializesModels;

    public $users;

    public function __construct($users)
    {
        $this->users = $users;
    }

    public function handle()
    {
        // Skip if batch is cancelled
        if ($this->batch()->cancelled()) {
            return;
        }

        foreach ($this->users as $user) {
            // Process individual user
            $this->processUser($user);
        }
    }

    private function processUser($user)
    {
        // User processing logic
        sleep(1); // Simulate processing time
    }
}
```

**Batch Dispatch:**
```php
<?php
// app/Console/Commands/ProcessAllUsers.php

use Illuminate\Support\Facades\Bus;
use App\Jobs\ProcessUsersBatch;

class ProcessAllUsers extends Command
{
    public function handle()
    {
        $users = User::all();
        $chunks = $users->chunk(100); // Process 100 users per job

        $jobs = [];
        foreach ($chunks as $chunk) {
            $jobs[] = new ProcessUsersBatch($chunk);
        }

        $batch = Bus::batch($jobs)
            ->then(function (Batch $batch) {
                // All jobs completed successfully
                \Log::info('All users processed successfully');
            })
            ->catch(function (Batch $batch, \Throwable $e) {
                // First batch job failure detected
                \Log::error('Batch processing failed: ' . $e->getMessage());
            })
            ->finally(function (Batch $batch) {
                // The batch has finished executing
                \Log::info('Batch processing completed');
            })
            ->dispatch();

        $this->info("Batch dispatched with ID: {$batch->id}");
        return 0;
    }
}
```

### 🔄 **Job Chaining**

**Sequential Job Processing:**
```php
<?php

use Illuminate\Support\Facades\Bus;
use App\Jobs\ProcessPayment;
use App\Jobs\SendReceipt;
use App\Jobs\UpdateInventory;
use App\Jobs\NotifyShipping;

// Chain jobs to run sequentially
Bus::chain([
    new ProcessPayment($order),
    new SendReceipt($order),
    new UpdateInventory($order),
    new NotifyShipping($order),
])->dispatch();

// Chain with error handling
Bus::chain([
    new ProcessPayment($order),
    new SendReceipt($order),
])->catch(function (\Throwable $e) {
    // Handle chain failure
    \Log::error('Payment chain failed: ' . $e->getMessage());
})->dispatch();
```

### ⚡ **Rate Limited Jobs**

**Rate Limited Processing:**
```php
<?php
// app/Jobs/CallExternalAPI.php

namespace App\Jobs;

use Illuminate\Contracts\Queue\ShouldQueue;
use Illuminate\Queue\Middleware\RateLimited;

class CallExternalAPI implements ShouldQueue
{
    public function middleware()
    {
        return [
            new RateLimited('api-calls'),
        ];
    }

    public function handle()
    {
        // API call logic
    }
}
```

**Rate Limiter Configuration:**
```php
<?php
// app/Providers/AppServiceProvider.php

use Illuminate\Cache\RateLimiting\Limit;
use Illuminate\Support\Facades\RateLimiter;

public function boot()
{
    RateLimiter::for('api-calls', function ($job) {
        return Limit::perMinute(60); // 60 calls per minute
    });

    RateLimiter::for('email-sending', function ($job) {
        return Limit::perMinute(100)->by($job->user->id);
    });
}
```

---

## Performance Optimization

### ⚡ **Queue Performance Tips**

**1. Connection Optimization:**
```php
// config/queue.php
'redis' => [
    'driver' => 'redis',
    'connection' => 'default',
    'queue' => env('REDIS_QUEUE', 'default'),
    'retry_after' => 90,
    'block_for' => 5,           // Block for 5 seconds when no jobs
    'after_commit' => false,    // Don't wait for DB transaction
],
```

**2. Worker Optimization:**
```bash
# Optimized worker command
php artisan queue:work redis \
    --sleep=3 \
    --tries=3 \
    --max-time=3600 \
    --memory=512 \
    --timeout=60
```

**3. Job Optimization:**
```php
class OptimizedJob implements ShouldQueue
{
    use Dispatchable, InteractsWithQueue, Queueable, SerializesModels;

    // Optimize serialization
    public function __serialize()
    {
        return [
            'user_id' => $this->user->id, // Store ID instead of model
        ];
    }

    public function __unserialize($data)
    {
        $this->user = User::find($data['user_id']); // Reload model
    }

    public function handle()
    {
        // Efficient processing
        DB::transaction(function () {
            // Batch operations
        });
    }
}
```

### 📊 **Monitoring Commands Schedule**

```php
// app/Console/Kernel.php

protected function schedule(Schedule $schedule)
{
    // Monitor failed jobs every 5 minutes
    $schedule->command('queue:monitor-failed')
             ->everyFiveMinutes();

    // Check queue alerts every 10 minutes
    $schedule->call(function () {
        app(QueueAlertService::class)->checkAlerts();
    })->everyTenMinutes();

    // Cleanup old jobs daily
    $schedule->command('queue:cleanup --days=30')
             ->daily();

    // Analyze performance weekly
    $schedule->command('queue:analyze-performance --days=7')
             ->weekly();

    // Restart workers daily (prevent memory leaks)
    $schedule->command('queue:restart')
             ->daily();
}
```

---

## 🎯 Quick Reference

### Essential Commands:
```bash
# Worker management
php artisan queue:work redis --queue=critical,high,default
php artisan queue:restart

# Monitoring
php artisan queue:failed
php artisan queue:monitor redis:default --max=100
php artisan queue:retry all

# Job management by ID
php artisan queue:job-details 123
php artisan queue:retry 123
php artisan queue:forget 123
```

### Production Checklist:
```
✅ Supervisor configuration for auto-restart
✅ Priority queues setup (critical, high, default, low)
✅ Failed job monitoring and alerts
✅ Regular cleanup of old jobs
✅ Performance metrics tracking
✅ Rate limiting for external APIs
✅ Job batching for bulk operations
✅ Proper error handling and logging
```

এই comprehensive guide follow করে আপনি Laravel এ professional-level event-driven architecture এবং robust queue management system implement করতে পারবেন।