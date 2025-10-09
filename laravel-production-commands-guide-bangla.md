# Laravel Production Commands - Essential বাংলা গাইড

## 📋 সূচিপত্র
- [Queue Commands বিস্তারিত](#queue-commands-বিস্তারিত)
- [Job Scheduling Commands](#job-scheduling-commands)
- [Kernel Commands](#kernel-commands)
- [Queue Worker Management](#queue-worker-management)
- [Cache Commands](#cache-commands)
- [Config Commands](#config-commands)
- [Route Commands](#route-commands)
- [View Commands](#view-commands)
- [Optimization Commands](#optimization-commands)
- [Deployment Commands](#deployment-commands)
- [Production Issues & Solutions](#production-issues--solutions)

---

## Queue Commands বিস্তারিত

### 🔄 **Queue Worker Commands:**

```bash
# Basic queue worker start
php artisan queue:work
```
**কি করে:**
- Background এ jobs process করে
- Database/Redis থেকে jobs নেয়
- Memory তে worker চালু রাখে

**কখন ব্যবহার:**
- Development environment এ
- Quick testing এর জন্য

**সমস্যা হতে পারে:**
- Memory leak হতে পারে
- Code change detect করে না
- Process crash হলে restart হয় না

```bash
# Specific connection দিয়ে
php artisan queue:work redis
php artisan queue:work database
php artisan queue:work sqs
```

```bash
# Timeout সহ
php artisan queue:work --timeout=300
```
**কেন দরকার:**
- Long running jobs এর জন্য
- Default 60 seconds timeout
- Job hang হলে kill করে

```bash
# Memory limit সহ
php artisan queue:work --memory=512
```
**কি করে:**
- 512MB memory use করার পর worker restart
- Memory leak prevent করে

```bash
# Specific queue process
php artisan queue:work --queue=high,default,low
```
**Priority order:**
- প্রথমে high queue
- তারপর default
- শেষে low queue

### 🔄 **Queue Management Commands:**

```bash
# Queue restart (সবচেয়ে গুরুত্বপূর্ণ)
php artisan queue:restart
```
**কি করে:**
- সব running workers কে gracefully stop করে
- Current job complete হওয়ার পর stop
- New code changes load করে

**কখন ব্যবহার:**
- প্রতিটি deployment এর পর (অবশ্যই)
- Code changes এর পর
- Worker memory issues হলে

```bash
# Failed jobs দেখা
php artisan queue:failed
```
**কি দেখায়:**
- Failed job ID
- Job class name
- Error message
- Failed time

```bash
# Failed job retry
php artisan queue:retry 5
php artisan queue:retry all
```

```bash
# Failed jobs clear
php artisan queue:flush
```
**সাবধান:** সব failed jobs permanently delete হয়ে যাবে

```bash
# Queue table clear
php artisan queue:clear
```
**কি করে:**
- Pending jobs delete করে
- Running jobs থাকলে সমস্যা হতে পারে

### 📊 **Queue Monitoring:**

```bash
# Queue status check
php artisan queue:monitor redis:default,redis:high --max=100
```
**কি করে:**
- Queue length monitor করে
- 100+ jobs হলে alert

```bash
# Queue work with stats
php artisan queue:work --verbose
```
**কি দেখায়:**
- Processing job details
- Memory usage
- Execution time

---

## Job Scheduling Commands

### ⏰ **Cron Job Setup:**

```bash
# Cron entry (server এ add করতে হবে)
* * * * * cd /path-to-your-project && php artisan schedule:run >> /dev/null 2>&1
```
**কি করে:**
- প্রতি মিনিটে Laravel scheduler check করে
- Due scheduled tasks run করে

### 📅 **Schedule Commands:**

```bash
# Scheduled tasks list
php artisan schedule:list
```
**কি দেখায়:**
- সব scheduled commands
- Next run time
- Command description

```bash
# Schedule run manually
php artisan schedule:run
```
**কখন ব্যবহার:**
- Testing scheduled tasks
- Manual trigger করতে

```bash
# Specific schedule test
php artisan schedule:test
```

```bash
# Schedule work (development)
php artisan schedule:work
```
**কি করে:**
- Development এ cron এর মত কাজ করে
- প্রতি মিনিটে check করে

---

## Kernel Commands

### 🔧 **Custom Artisan Commands:**

```bash
# Command create
php artisan make:command SendEmails
```

```bash
# Command list
php artisan list
```
**কি দেখায়:**
- সব available commands
- Custom commands
- Built-in commands

```bash
# Command help
php artisan help queue:work
```

### ⚙️ **Kernel Configuration:**

**app/Console/Kernel.php এ:**
- Commands register করা হয়
- Schedule define করা হয়
- Command signatures set করা হয়

**গুরুত্বপূর্ণ methods:**
- `schedule()` - Cron jobs define
- `commands()` - Custom commands load

---

## Queue Worker Management

### 🔄 **Supervisor Configuration:**

```ini
[program:laravel-worker]
process_name=%(program_name)s_%(process_num)02d
command=php /var/www/html/artisan queue:work redis --sleep=3 --tries=3 --max-time=3600
autostart=true
autorestart=true
stopasgroup=true
killasgroup=true
user=www-data
numprocs=8
redirect_stderr=true
stdout_logfile=/var/www/html/storage/logs/worker.log
stopwaitsecs=3600
```

**Supervisor Commands:**
```bash
# Supervisor restart
sudo supervisorctl reread
sudo supervisorctl update
sudo supervisorctl restart laravel-worker:*
```

### 🚨 **Production Worker Issues:**

**Memory Issues:**
- Worker memory বাড়তে থাকে
- `--memory=512` limit set করুন
- Regular restart করুন

**Code Changes না নেওয়া:**
- `queue:restart` করতে ভুলে যাওয়া
- Supervisor restart করা

**Job Timeout:**
- Long running jobs
- `--timeout` increase করুন
- Job এ timeout handle করুন

**Failed Jobs বেড়ে যাওয়া:**
- Database connection issues
- Memory limit exceed
- Code errors

---

## Cache Commands

### 🚀 **Cache Clear & Optimize:**

```bash
# Cache clear করা
php artisan cache:clear
```
**কখন ব্যবহার করবেন:** 
- Production deployment এর পর
- Cache corruption হলে
- Memory issues হলে

**Impact:** 
- ✅ Frees memory
- ❌ Next requests will be slower
- ❌ Database load increase

```bash
# View cache clear
php artisan view:clear
```
**কখন ব্যবহার করবেন:**
- Blade template changes এর পর
- View compilation errors

```bash
# Route cache
php artisan route:cache
```
**কখন ব্যবহার করবেন:**
- Production deployment
- Route changes এর পর

**Impact:**
- ✅ 50x faster route resolution
- ❌ Closure routes won't work

```bash
# Config cache
php artisan config:cache
```
**কখন ব্যবহার করবেন:**
- Production deployment
- Config changes এর পর

**Impact:**
- ✅ Faster config loading
- ❌ env() function won't work in code

---

## Queue Commands (Summary)

### ⚡ **Quick Reference:**

```bash
# Essential queue commands
php artisan queue:work --memory=512 --timeout=300
php artisan queue:restart  # After every deployment
php artisan queue:failed   # Check failed jobs
php artisan queue:retry all # Retry failed jobs
```

**Production Impact:**
- ✅ Background job processing
- ❌ Memory leaks if not monitored
- ⚠️ Needs Supervisor for auto-restart
- 🚨 Must restart after code changes

---

## Optimization Commands

### 🔧 **Production Optimization:**

```bash
# Complete optimization
php artisan optimize
```
**এটি চালায়:**
- `config:cache`
- `route:cache` 
- `view:cache`

```bash
# Clear all optimizations
php artisan optimize:clear
```

**Production Deployment Script:**
```bash
#!/bin/bash
# deploy.sh

# Pull latest code
git pull origin main

# Install dependencies
composer install --no-dev --optimize-autoloader

# Clear caches
php artisan cache:clear
php artisan config:clear
php artisan route:clear
php artisan view:clear

# Optimize for production
php artisan config:cache
php artisan route:cache
php artisan view:cache

# Run migrations
php artisan migrate --force

# Restart queue workers
php artisan queue:restart

# Restart PHP-FPM
sudo service php8.2-fpm restart
```

---

## Server Impact & Monitoring

### ⚠️ **Memory Impact:**

```bash
# Check memory usage
free -h

# Monitor Laravel processes
ps aux | grep php

# Check queue worker memory
ps aux | grep "queue:work"
```

### 📊 **Performance Monitoring:**

```bash
# Enable query logging
php artisan tinker
>>> DB::enableQueryLog();
>>> // Run your operations
>>> dd(DB::getQueryLog());
```

---

## Production Issues & Solutions

### 🚨 **Common Queue Issues:**

**1. Jobs না চলা:**
```bash
# Check করুন
php artisan queue:failed
ps aux | grep "queue:work"

# Solution
php artisan queue:restart
sudo supervisorctl restart laravel-worker:*
```

**2. Memory Leak:**
```bash
# Check memory
free -h
ps aux | grep php | grep queue

# Solution
php artisan queue:work --memory=512
```

**3. Jobs Stuck:**
```bash
# Clear stuck jobs
php artisan queue:clear
php artisan queue:restart
```

**4. Schedule না চলা:**
```bash
# Check cron
crontab -l

# Test schedule
php artisan schedule:run
php artisan schedule:list
```

### 📋 **Production Deployment Checklist:**

```bash
#!/bin/bash
# Complete deployment script

# 1. Maintenance mode
php artisan down

# 2. Pull code
git pull origin main

# 3. Dependencies
composer install --no-dev --optimize-autoloader
npm ci && npm run build

# 4. Clear caches
php artisan cache:clear
php artisan config:clear
php artisan route:clear
php artisan view:clear

# 5. Database
php artisan migrate --force

# 6. Optimize
php artisan config:cache
php artisan route:cache
php artisan view:cache

# 7. Queue restart (গুরুত্বপূর্ণ)
php artisan queue:restart

# 8. Services restart
sudo supervisorctl restart laravel-worker:*
sudo service php8.2-fpm restart
sudo service nginx restart

# 9. Maintenance mode off
php artisan up

# 10. Health check
curl -f http://yoursite.com/health || exit 1
```

### 🔍 **Monitoring Commands:**

```bash
# System monitoring
top -p $(pgrep -d',' php)
htop

# Laravel logs
tail -f storage/logs/laravel.log

# Queue monitoring
watch -n 5 'php artisan queue:failed | wc -l'

# Database connections
php artisan tinker
>>> DB::select('SHOW PROCESSLIST');
```

**Critical Commands for Production:**
1. `php artisan queue:restart` - প্রতি deployment এ (সবচেয়ে গুরুত্বপূর্ণ)
2. `php artisan optimize` - Before going live
3. `php artisan cache:clear` - Cache issues হলে
4. `php artisan migrate --force` - Database updates
5. `php artisan schedule:list` - Cron jobs verify
6. `php artisan queue:failed` - Failed jobs check