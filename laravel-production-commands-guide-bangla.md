# Laravel Production Commands - Essential বাংলা গাইড

## 📋 সূচিপত্র
- [Cache Commands](#cache-commands)
- [Config Commands](#config-commands)
- [Route Commands](#route-commands)
- [Queue Commands](#queue-commands)
- [View Commands](#view-commands)
- [Optimization Commands](#optimization-commands)
- [Deployment Commands](#deployment-commands)

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

## Queue Commands

### ⚡ **Queue Management:**

```bash
# Start queue worker
php artisan queue:work

# Queue with specific connection
php artisan queue:work redis

# Queue with timeout
php artisan queue:work --timeout=60
```

**Production Impact:**
- ✅ Background job processing
- ❌ Memory leaks if not monitored
- ⚠️ Needs process monitoring (Supervisor)

```bash
# Restart queue workers
php artisan queue:restart
```
**কখন ব্যবহার করবেন:**
- Code deployment এর পর
- Queue worker memory issues

```bash
# Clear failed jobs
php artisan queue:flush
```

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

**Critical Commands for Production:**
1. `php artisan optimize` - Before going live
2. `php artisan queue:restart` - After deployments  
3. `php artisan cache:clear` - When cache issues
4. `php artisan migrate --force` - Database updates