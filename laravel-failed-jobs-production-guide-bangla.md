# Laravel Failed Jobs - Production Level বাংলা গাইড

## 📋 সূচিপত্র
- [Failed Jobs কি এবং কেন হয়](#failed-jobs-কি-এবং-কেন-হয়)
- [Failed Jobs Table Setup](#failed-jobs-table-setup)
- [Job Lifecycle বুঝা](#job-lifecycle-বুঝা)
- [Failed Jobs Monitoring](#failed-jobs-monitoring)
- [Failed Jobs Recovery Strategy](#failed-jobs-recovery-strategy)
- [Production Best Practices](#production-best-practices)
- [Alerting ও Notification](#alerting-ও-notification)
- [Performance Impact](#performance-impact)
- [Troubleshooting Guide](#troubleshooting-guide)

---

## Failed Jobs কি এবং কেন হয়

### 🚨 **Failed Jobs কি?**

Failed jobs হলো সেই background jobs যা execute করার সময় error throw করেছে এবং maximum retry attempts শেষ হয়ে গেছে।

### 🎯 **কেন Jobs Fail হয়?**

**Common Reasons:**
1. **Database Connection Issues** - DB server down/timeout
2. **External API Failures** - Third-party service unavailable
3. **Memory Limit Exceeded** - Large data processing
4. **Timeout Issues** - Job execution time বেশি
5. **Code Errors** - Bugs, undefined variables
6. **File System Issues** - Storage full, permission problems
7. **Network Issues** - Internet connectivity problems
8. **Resource Constraints** - CPU/Memory shortage

### 📊 **Job States Lifecycle:**

```
Pending → Processing → Completed ✅
    ↓
  Failed (after max tries) ❌
    ↓
  Retry → Processing → Completed ✅
```

**State Details:**
- **Pending:** Queue তে waiting
- **Processing:** Worker execute করছে
- **Completed:** Successfully finished
- **Failed:** Max tries exceeded
- **Retry:** Manual/Auto retry initiated

---

## Failed Jobs Table Setup

### 🗄️ **Database Table তৈরি**

```bash
# Failed jobs table migration তৈরি
php artisan queue:failed-table
php artisan migrate
```

**Table Structure:**
```sql
CREATE TABLE failed_jobs (
    id BIGINT PRIMARY KEY AUTO_INCREMENT,
    uuid VARCHAR(255) UNIQUE,
    connection TEXT,
    queue TEXT,
    payload LONGTEXT,
    exception LONGTEXT,
    failed_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);
```

**Column বিস্তারিত:**
- **id:** Unique identifier
- **uuid:** Job UUID (Laravel 8+)
- **connection:** Queue connection (redis/database)
- **queue:** Queue name (default/high/low)
- **payload:** Serialized job data
- **exception:** Error message ও stack trace
- **failed_at:** Failure timestamp

### ⚙️ **Configuration**

**config/queue.php:**
```php
'failed' => [
    'driver' => env('QUEUE_FAILED_DRIVER', 'database'),
    'database' => env('DB_CONNECTION', 'mysql'),
    'table' => 'failed_jobs',
],
```

---

## Job Lifecycle বুঝা

### 🔄 **Job Processing Flow**

**1. Job Creation ও Dispatch:**
```
User Action → Job::dispatch() → Queue Storage → Worker Picks Up
```

**2. Job Execution Attempts:**
```
Try 1 → Fail → Wait → Try 2 → Fail → Wait → Try 3 → Failed Table
```

**3. Retry Logic:**
```
Manual Retry → Remove from Failed → Add to Queue → Process Again
```

### 📋 **Job Configuration Parameters**

**Job Class Properties:**
- **$tries:** Maximum retry attempts (default: 3)
- **$timeout:** Maximum execution time (default: 60s)
- **$backoff:** Delay between retries (seconds/array)
- **$maxExceptions:** Max exceptions before failing
- **$retryUntil:** Retry until specific time

**Example Job Configuration:**
```php
class ProcessPaymentJob implements ShouldQueue
{
    public $tries = 5;           // 5 বার try করবে
    public $timeout = 300;       // 5 মিনিট timeout
    public $backoff = [10, 30, 60]; // Exponential backoff
    public $maxExceptions = 3;   // 3টি exception এর পর fail
    
    public function retryUntil()
    {
        return now()->addHours(2); // 2 ঘন্টা পর্যন্ত retry
    }
}
```

### 🎯 **Worker Behavior**

**Worker Process:**
1. **Pick Job:** Queue থেকে job নেয়
2. **Execute:** Job handle() method চালায়
3. **Success:** Job complete, remove from queue
4. **Failure:** Increment attempts, check max tries
5. **Max Reached:** Move to failed_jobs table
6. **Continue:** Next job process করে

**Worker Commands Impact:**
```bash
# Normal processing
php artisan queue:work --tries=3 --timeout=60

# Aggressive processing (risky)
php artisan queue:work --tries=1 --timeout=30

# Conservative processing (safe)
php artisan queue:work --tries=5 --timeout=300
```

---

## Failed Jobs Monitoring

### 📊 **Monitoring Commands**

**1. Failed Jobs দেখা:**
```bash
# সব failed jobs
php artisan queue:failed

# Output format:
# ID | Connection | Queue | Class | Failed At
# 1  | redis      | default | SendEmailJob | 2024-01-15 10:30:00
```

**2. Specific Failed Job Details:**
```bash
# Detailed view (Laravel 9+)
php artisan queue:failed --id=1

# Shows:
# - Job class name
# - Queue connection
# - Payload data
# - Exception message
# - Stack trace
# - Failed timestamp
```

**3. Failed Jobs Count:**
```bash
# Count total failed jobs
php artisan tinker
>>> DB::table('failed_jobs')->count()
```

### 📈 **Monitoring Metrics**

**Key Metrics to Track:**
1. **Failed Job Count:** Daily/Weekly trends
2. **Failure Rate:** Failed/Total jobs ratio
3. **Common Errors:** Most frequent exceptions
4. **Queue Performance:** Processing time trends
5. **Recovery Rate:** Successfully retried jobs

**Custom Monitoring Command:**
```bash
# Create monitoring command
php artisan make:command MonitorFailedJobs
```

**Monitoring Script Logic:**
```php
class MonitorFailedJobs extends Command
{
    public function handle()
    {
        $failedCount = DB::table('failed_jobs')->count();
        $recentFailed = DB::table('failed_jobs')
            ->where('failed_at', '>', now()->subHour())
            ->count();
            
        $this->info("Total Failed Jobs: {$failedCount}");
        $this->info("Failed in Last Hour: {$recentFailed}");
        
        // Alert if too many failures
        if ($recentFailed > 10) {
            $this->error("HIGH FAILURE RATE DETECTED!");
            // Send notification
        }
    }
}
```

---

## Failed Jobs Recovery Strategy

### 🔄 **Retry Strategies**

**1. Individual Job Retry:**
```bash
# Specific job retry by ID
php artisan queue:retry 5

# Multiple jobs retry
php artisan queue:retry 1,2,3,4,5
```

**2. Bulk Retry:**
```bash
# সব failed jobs retry
php artisan queue:retry all

# Recent failed jobs retry (last 24 hours)
php artisan queue:retry --range=24h
```

**3. Selective Retry:**
```bash
# Specific job class retry
php artisan queue:retry --queue=emails

# By connection
php artisan queue:retry --connection=redis
```

### 🎯 **Smart Recovery Approach**

**Recovery Priority:**
1. **Critical Jobs First:** Payment, security-related
2. **Time-Sensitive:** Notifications, emails
3. **Bulk Operations:** Reports, data processing
4. **Non-Critical:** Cleanup, maintenance

**Recovery Script:**
```bash
#!/bin/bash
# smart-recovery.sh

echo "🔄 Starting Smart Recovery Process..."

# 1. Retry critical payment jobs first
php artisan queue:retry --queue=payments

# 2. Retry time-sensitive notifications
php artisan queue:retry --queue=notifications

# 3. Wait and check system load
sleep 30

# 4. Retry remaining jobs in batches
FAILED_COUNT=$(php artisan queue:failed | wc -l)
if [ $FAILED_COUNT -lt 100 ]; then
    php artisan queue:retry all
else
    echo "⚠️ Too many failed jobs ($FAILED_COUNT). Manual review needed."
fi

echo "✅ Recovery process completed"
```

### 🧹 **Cleanup Strategy**

**1. Old Failed Jobs Cleanup:**
```bash
# Delete failed jobs older than 30 days
php artisan queue:prune-failed --hours=720

# Delete specific failed job
php artisan queue:forget 5
```

**2. Automated Cleanup:**
```php
// In Kernel.php schedule
$schedule->command('queue:prune-failed --hours=168')->weekly(); // 1 week
```

**3. Bulk Cleanup:**
```bash
# Clear all failed jobs (DANGEROUS!)
php artisan queue:flush
```

---

## Production Best Practices

### 🚀 **Job Design Best Practices**

**1. Idempotent Jobs:**
```php
// Job should be safe to run multiple times
class ProcessOrderJob
{
    public function handle()
    {
        $order = Order::find($this->orderId);
        
        // Check if already processed
        if ($order->status === 'processed') {
            return; // Skip if already done
        }
        
        // Process order
        $order->process();
    }
}
```

**2. Proper Error Handling:**
```php
class SendEmailJob
{
    public function handle()
    {
        try {
            Mail::to($this->user)->send(new WelcomeEmail());
        } catch (TransportException $e) {
            // Temporary failure - will retry
            throw $e;
        } catch (InvalidEmailException $e) {
            // Permanent failure - don't retry
            $this->fail($e);
        }
    }
    
    public function failed(\Throwable $exception)
    {
        // Log failure for analysis
        Log::error('Email job failed', [
            'user_id' => $this->user->id,
            'error' => $exception->getMessage()
        ]);
        
        // Notify admin if critical
        if ($this->isCritical) {
            AdminNotification::send($exception);
        }
    }
}
```

**3. Resource Management:**
```php
class ProcessLargeFileJob
{
    public $timeout = 600; // 10 minutes
    public $tries = 3;
    
    public function handle()
    {
        // Process in chunks to avoid memory issues
        $file = Storage::get($this->filePath);
        $chunks = str_split($file, 1024 * 1024); // 1MB chunks
        
        foreach ($chunks as $chunk) {
            $this->processChunk($chunk);
            
            // Free memory periodically
            if (memory_get_usage() > 100 * 1024 * 1024) { // 100MB
                gc_collect_cycles();
            }
        }
    }
}
```

### 📊 **Monitoring Setup**

**1. Failed Job Alerts:**
```php
// Create alert command
class CheckFailedJobs extends Command
{
    public function handle()
    {
        $threshold = 50; // Alert if more than 50 failed jobs
        $count = DB::table('failed_jobs')->count();
        
        if ($count > $threshold) {
            // Send Slack notification
            Http::post(config('services.slack.webhook'), [
                'text' => "🚨 High failed job count: {$count} jobs failed"
            ]);
            
            // Send email to admin
            Mail::to('admin@company.com')->send(new FailedJobAlert($count));
        }
    }
}
```

**2. Scheduled Monitoring:**
```php
// In Kernel.php
$schedule->command('check:failed-jobs')->hourly();
$schedule->command('queue:retry --range=1h')->everyTenMinutes();
```

### 🔧 **Worker Configuration**

**Production Worker Settings:**
```bash
# Supervisor configuration
[program:laravel-worker]
command=php /var/www/artisan queue:work redis --sleep=3 --tries=3 --max-time=3600 --memory=512
numprocs=4
autostart=true
autorestart=true
user=www-data
redirect_stderr=true
stdout_logfile=/var/log/laravel-worker.log
```

**Worker Health Check:**
```bash
#!/bin/bash
# worker-health-check.sh

WORKER_COUNT=$(ps aux | grep "queue:work" | grep -v grep | wc -l)
EXPECTED_WORKERS=4

if [ $WORKER_COUNT -lt $EXPECTED_WORKERS ]; then
    echo "⚠️ Worker count low: $WORKER_COUNT/$EXPECTED_WORKERS"
    sudo supervisorctl restart laravel-worker:*
fi

# Check failed job growth
FAILED_COUNT=$(php artisan queue:failed | wc -l)
if [ $FAILED_COUNT -gt 100 ]; then
    echo "🚨 High failed job count: $FAILED_COUNT"
    # Trigger alert
fi
```

---

## Alerting ও Notification

### 📢 **Alert System Setup**

**1. Slack Integration:**
```php
class FailedJobNotifier
{
    public static function notify($jobClass, $exception, $attempts)
    {
        $message = [
            'text' => '🚨 Job Failed Alert',
            'attachments' => [
                [
                    'color' => 'danger',
                    'fields' => [
                        ['title' => 'Job Class', 'value' => $jobClass, 'short' => true],
                        ['title' => 'Attempts', 'value' => $attempts, 'short' => true],
                        ['title' => 'Error', 'value' => substr($exception, 0, 200), 'short' => false],
                        ['title' => 'Time', 'value' => now()->toDateTimeString(), 'short' => true],
                    ]
                ]
            ]
        ];
        
        Http::post(config('services.slack.webhook'), $message);
    }
}
```

**2. Email Alerts:**
```php
class FailedJobMail extends Mailable
{
    public function build()
    {
        return $this->subject('Failed Job Alert')
                   ->view('emails.failed-job')
                   ->with([
                       'jobClass' => $this->jobClass,
                       'error' => $this->error,
                       'failedAt' => $this->failedAt
                   ]);
    }
}
```

**3. Dashboard Integration:**
```php
// Real-time dashboard data
class QueueStatsController
{
    public function stats()
    {
        return response()->json([
            'failed_jobs' => DB::table('failed_jobs')->count(),
            'pending_jobs' => Redis::llen('queues:default'),
            'processed_today' => Cache::get('jobs_processed_today', 0),
            'failure_rate' => $this->calculateFailureRate(),
        ]);
    }
}
```

### 📊 **Metrics Collection**

**Key Metrics:**
```php
class JobMetrics
{
    public static function recordJobStart($jobClass)
    {
        Redis::incr("jobs:started:{$jobClass}");
        Redis::incr("jobs:started:total");
    }
    
    public static function recordJobSuccess($jobClass, $duration)
    {
        Redis::incr("jobs:completed:{$jobClass}");
        Redis::lpush("jobs:duration:{$jobClass}", $duration);
    }
    
    public static function recordJobFailure($jobClass, $exception)
    {
        Redis::incr("jobs:failed:{$jobClass}");
        Redis::lpush("jobs:errors:{$jobClass}", $exception);
    }
    
    public static function getFailureRate($jobClass = null)
    {
        $key = $jobClass ? "jobs:failed:{$jobClass}" : "jobs:failed:total";
        $failed = Redis::get($key) ?: 0;
        $total = Redis::get("jobs:started:total") ?: 1;
        
        return round(($failed / $total) * 100, 2);
    }
}
```

---

## Performance Impact

### ⚡ **Failed Jobs Table Performance**

**Performance Issues:**
1. **Table Growth:** Failed jobs accumulate over time
2. **Query Slowdown:** Large table affects queries
3. **Storage Usage:** Payload data takes space
4. **Index Fragmentation:** Frequent inserts/deletes

**Optimization Strategies:**

**1. Regular Cleanup:**
```bash
# Daily cleanup of old failed jobs
0 2 * * * php /var/www/artisan queue:prune-failed --hours=168
```

**2. Table Partitioning:**
```sql
-- Partition by month for better performance
ALTER TABLE failed_jobs PARTITION BY RANGE (MONTH(failed_at)) (
    PARTITION p202401 VALUES LESS THAN (2),
    PARTITION p202402 VALUES LESS THAN (3),
    -- Add more partitions as needed
);
```

**3. Archive Strategy:**
```php
class ArchiveFailedJobs extends Command
{
    public function handle()
    {
        $oldJobs = DB::table('failed_jobs')
            ->where('failed_at', '<', now()->subDays(30))
            ->get();
            
        // Archive to separate table
        DB::table('failed_jobs_archive')->insert($oldJobs->toArray());
        
        // Delete from main table
        DB::table('failed_jobs')
            ->where('failed_at', '<', now()->subDays(30))
            ->delete();
            
        $this->info("Archived {$oldJobs->count()} failed jobs");
    }
}
```

### 📈 **Memory Management**

**Worker Memory Issues:**
```bash
# Monitor worker memory
ps aux --sort=-%mem | grep "queue:work"

# Restart workers if memory high
if [ $(ps aux | grep "queue:work" | awk '{sum+=$6} END {print sum}') -gt 1000000 ]; then
    php artisan queue:restart
fi
```

**Memory Optimization:**
```php
class OptimizedJob implements ShouldQueue
{
    public function handle()
    {
        // Process in batches to control memory
        User::chunk(1000, function ($users) {
            foreach ($users as $user) {
                $this->processUser($user);
            }
            
            // Force garbage collection
            gc_collect_cycles();
        });
    }
}
```

---

## Troubleshooting Guide

### 🔍 **Common Issues ও Solutions**

**1. Jobs Failing Immediately:**
```bash
# Check worker logs
tail -f storage/logs/laravel.log

# Check supervisor logs
sudo tail -f /var/log/supervisor/supervisord.log

# Test job manually
php artisan tinker
>>> dispatch(new YourJob($data))
```

**2. High Failure Rate:**
```bash
# Analyze failure patterns
php artisan tinker
>>> DB::table('failed_jobs')->select('exception')->get()->groupBy(function($item) {
    return substr($item->exception, 0, 100);
})->map->count()
```

**3. Memory Issues:**
```bash
# Check memory usage
free -h

# Monitor worker memory
watch -n 5 'ps aux | grep "queue:work" | grep -v grep'

# Restart workers
php artisan queue:restart
```

**4. Database Connection Issues:**
```bash
# Test database connection
php artisan tinker
>>> DB::connection()->getPdo()

# Check connection pool
>>> DB::select('SHOW PROCESSLIST')
```

### 🚨 **Emergency Procedures**

**1. Mass Job Failure:**
```bash
# Stop all workers
sudo supervisorctl stop laravel-worker:*

# Clear problematic jobs
php artisan queue:clear

# Fix underlying issue
# Restart workers
sudo supervisorctl start laravel-worker:*
```

**2. Database Full:**
```bash
# Emergency cleanup
php artisan queue:flush  # Clear all failed jobs
php artisan cache:clear  # Clear cache
php artisan queue:restart # Restart workers
```

**3. System Recovery:**
```bash
#!/bin/bash
# emergency-recovery.sh

echo "🚨 Starting Emergency Recovery..."

# 1. Stop workers
sudo supervisorctl stop laravel-worker:*

# 2. Clear queues
php artisan queue:clear
php artisan cache:clear

# 3. Archive failed jobs
php artisan queue:prune-failed --hours=24

# 4. Restart services
sudo service redis-server restart
sudo service mysql restart

# 5. Start workers
sudo supervisorctl start laravel-worker:*

echo "✅ Emergency recovery completed"
```

---

## 🎯 **Quick Reference**

### Daily Operations:
```bash
# Check failed jobs
php artisan queue:failed

# Retry recent failures
php artisan queue:retry --range=24h

# Monitor worker status
ps aux | grep "queue:work"
```

### Weekly Maintenance:
```bash
# Cleanup old failed jobs
php artisan queue:prune-failed --hours=168

# Analyze failure patterns
php artisan queue:failed | grep -E "SendEmailJob|ProcessPaymentJob"

# Check system resources
free -h && df -h
```

### Emergency Commands:
```bash
# Stop everything
sudo supervisorctl stop laravel-worker:*

# Clear all queues
php artisan queue:clear && php artisan queue:flush

# Restart everything
sudo supervisorctl start laravel-worker:*
```

---

এই গাইড follow করে আপনি production environment এ Laravel failed jobs effectively manage করতে পারবেন। Regular monitoring, proper alerting, এবং strategic recovery approach দিয়ে system reliability maintain করুন।