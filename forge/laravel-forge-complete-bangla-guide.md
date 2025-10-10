# Laravel Forge - সম্পূর্ণ বাংলা গাইড (নতুনদের জন্য)

## 📋 সূচিপত্র
- [Laravel Forge কি এবং কেন ব্যবহার করবেন](#laravel-forge-কি-এবং-কেন-ব্যবহার-করবেন)
- [প্রথম সেটআপ - ধাপে ধাপে](#প্রথম-সেটআপ---ধাপে-ধাপে)
- [Forge Dashboard - সব ট্যাবের বিস্তারিত](#forge-dashboard---সব-ট্যাবের-বিস্তারিত)
- [Deploy Script - প্রোডাকশন টেমপ্লেট](#deploy-script---প্রোডাকশন-টেমপ্লেট)
- [Queue ও Worker Management](#queue-ও-worker-management)
- [SSL ও Domain Setup](#ssl-ও-domain-setup)
- [সাধারণ সমস্যা ও সমাধান](#সাধারণ-সমস্যা-ও-সমাধান)
- [Production Best Practices](#production-best-practices)

---

## Laravel Forge কি এবং কেন ব্যবহার করবেন

### 🚀 **Laravel Forge কি?**

Laravel Forge হলো একটি **server provisioning এবং deployment tool** যা আপনার Laravel অ্যাপ্লিকেশনকে production server এ deploy করার সব জটিল কাজ সহজ করে দেয়।

### 🎯 **কেন ব্যবহার করবেন?**

**ম্যানুয়াল সেটআপ ছাড়াই পাবেন:**
- ✅ Nginx web server configuration
- ✅ PHP-FPM optimization
- ✅ MySQL/PostgreSQL database setup
- ✅ Redis caching
- ✅ SSL certificate (Let's Encrypt)
- ✅ Queue worker management
- ✅ Scheduled jobs (cron)
- ✅ Firewall configuration
- ✅ Auto deployment from Git

**সময় বাঁচায়:**
- Manual server setup: 4-6 ঘন্টা
- Forge দিয়ে: 10-15 মিনিট

### 💰 **খরচ কেমন?**

- **Forge subscription:** $12/মাস (unlimited servers)
- **Server cost:** VPS provider অনুযায়ী (DigitalOcean: $5-20/মাস)

### 🏢 **কোন প্রজেক্টের জন্য উপযুক্ত?**

- ✅ Small to medium Laravel projects
- ✅ Client projects (quick deployment)
- ✅ Startup MVPs
- ✅ Personal projects
- ❌ Enterprise level (Kubernetes/Docker better)

---

## প্রথম সেটআপ - ধাপে ধাপে

### 1️⃣ **Forge Account ও Server তৈরি**

```bash
# Step 1: Forge.laravel.com এ account তৈরি করুন
# Step 2: VPS Provider connect করুন (DigitalOcean/AWS/Vultr)
```

**Server তৈরি:**
1. Forge Dashboard → **"Create Server"**
2. **Provider:** DigitalOcean/AWS/Vultr select
3. **Region:** Singapore/Frankfurt (Bangladesh এর কাছে)
4. **Size:** $10/month (2GB RAM) minimum
5. **PHP Version:** 8.2 (latest stable)
6. **Database:** MySQL 8.0
7. **Server Name:** `my-app-production`

**⏱️ সময় লাগবে:** 5-10 মিনিট

### 2️⃣ **Site তৈরি করা**

Server ready হলে:

1. Server page → **"New Site"**
2. **Domain:** `yourdomain.com` বা `subdomain.yourdomain.com`
3. **Project Type:** Laravel
4. **Web Directory:** `/public` (auto selected)

### 3️⃣ **Git Repository যুক্ত করা**

1. Site → **"Repository"** tab
2. **Provider:** GitHub/GitLab/Bitbucket
3. **Repository:** `username/repository-name`
4. **Branch:** `main` বা `master`
5. **Deploy Key** automatically add হবে

### 4️⃣ **Environment Variables সেট করা**

1. Site → **"Environment"** tab
2. আপনার local `.env` file এর content copy করুন
3. **Production values** দিয়ে replace করুন:

```env
APP_NAME="My App"
APP_ENV=production
APP_KEY=base64:xxxxx  # php artisan key:generate
APP_DEBUG=false
APP_URL=https://yourdomain.com

DB_CONNECTION=mysql
DB_HOST=127.0.0.1
DB_PORT=3306
DB_DATABASE=forge
DB_USERNAME=forge
DB_PASSWORD=xxxxx  # Forge auto generate করবে

CACHE_DRIVER=redis
QUEUE_CONNECTION=redis
SESSION_DRIVER=redis

REDIS_HOST=127.0.0.1
REDIS_PASSWORD=null
REDIS_PORT=6379
```

### 5️⃣ **প্রথম Deploy**

1. Site → **"Deployments"** tab
2. **Deploy Script** check করুন (নিচে template আছে)
3. **"Deploy Now"** button click
4. **Deployment log** দেখুন

---

## Forge Dashboard - সব ট্যাবের বিস্তারিত

### 📊 **Overview Tab**

**কি দেখাবে:**
- Server status (CPU, RAM, Disk usage)
- PHP/Nginx/MySQL service status
- Recent deployments
- Quick deploy button
- Site health metrics

**কখন ব্যবহার:**
- Daily monitoring
- Quick deploy
- Server health check

### 🚀 **Deployments Tab**

**Deploy Script Management:**
```bash
# Default script যা edit করতে পারেন
cd $FORGE_SITE_PATH
git pull origin $FORGE_SITE_BRANCH

$FORGE_COMPOSER install --no-dev --no-interaction --prefer-dist --optimize-autoloader

if [ -f artisan ]; then
    $FORGE_PHP artisan migrate --force
    $FORGE_PHP artisan config:cache
    $FORGE_PHP artisan route:cache
    $FORGE_PHP artisan view:cache
    $FORGE_PHP artisan queue:restart
fi
```

**Features:**
- **Quick Deploy:** Git push এ auto deploy
- **Deploy Hooks:** Before/After deploy scripts
- **Deployment History:** Previous deployments
- **Rollback:** Previous version এ ফিরে যাওয়া

### ⚙️ **Processes Tab**

**Queue Workers:**
```bash
# Worker command example
php artisan queue:work redis --sleep=3 --tries=3 --max-time=3600 --memory=512
```

**কি করতে পারবেন:**
- Queue worker start/stop/restart
- Horizon setup (Redis queue management)
- Custom daemon processes
- Worker logs monitoring

**Production Settings:**
- **Processes:** 2-4 workers (traffic অনুযায়ী)
- **Memory Limit:** 512MB
- **Max Time:** 3600 seconds
- **Sleep:** 3 seconds
- **Tries:** 3 attempts

### 💻 **Commands Tab**

**Saved Commands:**
```bash
# Useful commands to save
php artisan storage:link
php artisan config:clear && php artisan config:cache
php artisan migrate:status
php artisan queue:failed
composer dump-autoload
npm run build
```

**কখন ব্যবহার:**
- One-time maintenance tasks
- Debug commands
- Manual cache clearing
- Database operations

### 🌐 **Network Tab**

**Firewall Rules:**
```bash
# Default open ports
22    # SSH
80    # HTTP
443   # HTTPS
3306  # MySQL (only from specific IPs)
```

**Custom Rules:**
- WebSocket port (6001) for broadcasting
- Custom API ports
- Database access from specific IPs
- Block malicious IPs

### 📊 **Observe Tab**

**Log Files:**
- **Nginx Access:** Traffic logs
- **Nginx Error:** Server errors
- **PHP-FPM:** PHP process errors
- **Laravel:** Application logs
- **Queue Worker:** Job processing logs

**Monitoring:**
- CPU usage alerts
- Memory usage alerts
- Disk space alerts
- Failed deployment alerts

### 🌍 **Domains Tab**

**Domain Management:**
```bash
# Multiple domains for same site
yourdomain.com
www.yourdomain.com
api.yourdomain.com
```

**SSL Certificates:**
- **Let's Encrypt:** Free SSL (auto-renewal)
- **Custom SSL:** Upload your own certificate
- **Wildcard SSL:** `*.yourdomain.com`

**Redirects:**
- www → non-www
- HTTP → HTTPS
- Old domain → New domain

### ⚙️ **Settings Tab**

**PHP Configuration:**
- PHP version change
- PHP.ini settings
- Memory limits
- Upload limits
- Execution time

**Site Settings:**
- Web directory change
- User permissions
- Timezone settings
- Maintenance mode

---

## Deploy Script - প্রোডাকশন টেমপ্লেট

### 🚀 **Complete Production Deploy Script**

```bash
#!/usr/bin/env bash
set -e

echo "🚀 Starting deployment..."

# Variables
SITE_PATH=$FORGE_SITE_PATH
BRANCH=$FORGE_SITE_BRANCH
PHP=$FORGE_PHP
COMPOSER=$FORGE_COMPOSER

# Navigate to site directory
cd $SITE_PATH

# 1. Maintenance Mode
echo "🔧 Enabling maintenance mode..."
$PHP artisan down --retry=60 --secret="forge-deploy-secret" || true

# 2. Pull latest code
echo "📥 Pulling latest code from $BRANCH..."
git pull origin $BRANCH

# 3. Install/Update dependencies
echo "📦 Installing Composer dependencies..."
$COMPOSER install --no-dev --no-interaction --prefer-dist --optimize-autoloader

# 4. Install Node dependencies (if needed)
if [ -f "package.json" ]; then
    echo "📦 Installing Node dependencies..."
    npm ci --only=production
    echo "🏗️ Building assets..."
    npm run build
fi

# 5. Clear caches
echo "🧹 Clearing caches..."
$PHP artisan cache:clear
$PHP artisan config:clear
$PHP artisan route:clear
$PHP artisan view:clear

# 6. Run migrations
echo "🗄️ Running database migrations..."
$PHP artisan migrate --force

# 7. Cache optimization
echo "⚡ Optimizing for production..."
$PHP artisan config:cache
$PHP artisan route:cache
$PHP artisan view:cache

# 8. Storage link (if needed)
if [ ! -L "public/storage" ]; then
    echo "🔗 Creating storage link..."
    $PHP artisan storage:link
fi

# 9. Restart queue workers
echo "🔄 Restarting queue workers..."
$PHP artisan queue:restart

# 10. Restart Horizon (if using)
# $PHP artisan horizon:terminate

# 11. Clear OPcache (if enabled)
if command -v php-fpm8.2 &> /dev/null; then
    echo "🧹 Clearing OPcache..."
    sudo service php8.2-fpm reload
fi

# 12. Disable maintenance mode
echo "✅ Disabling maintenance mode..."
$PHP artisan up

echo "🎉 Deployment completed successfully!"

# 13. Health check
echo "🏥 Running health check..."
curl -f $FORGE_SITE_URL/health || echo "⚠️ Health check failed"

echo "📊 Deployment summary:"
echo "- Branch: $BRANCH"
echo "- Commit: $(git rev-parse --short HEAD)"
echo "- Time: $(date)"
```

### 🔧 **Deploy Hooks**

**Before Deploy Hook:**
```bash
# Backup database before major updates
mysqldump -u forge -p$DB_PASSWORD forge > /home/forge/backups/$(date +%Y%m%d_%H%M%S).sql

# Notify team
curl -X POST "https://hooks.slack.com/services/YOUR/SLACK/WEBHOOK" \
  -H 'Content-type: application/json' \
  --data '{"text":"🚀 Starting deployment..."}'
```

**After Deploy Hook:**
```bash
# Warm up cache
curl -s $FORGE_SITE_URL > /dev/null

# Notify team
curl -X POST "https://hooks.slack.com/services/YOUR/SLACK/WEBHOOK" \
  -H 'Content-type: application/json' \
  --data '{"text":"✅ Deployment completed!"}'
```

---

## Queue ও Worker Management

### 🔄 **Queue Worker Setup**

**Basic Worker Configuration:**
```bash
# Command
php artisan queue:work redis

# Production Command
php artisan queue:work redis --sleep=3 --tries=3 --max-time=3600 --memory=512
```

**Parameters বিস্তারিত:**
- `--sleep=3`: Job না থাকলে 3 সেকেন্ড wait
- `--tries=3`: Failed হলে 3 বার try
- `--max-time=3600`: 1 ঘন্টা পর worker restart
- `--memory=512`: 512MB memory limit

### 🎯 **Priority Queue Setup**

```bash
# Multiple queue with priority
php artisan queue:work redis --queue=high,default,low --sleep=1 --tries=3
```

**Queue Priority:**
1. **high**: Critical jobs (email, notifications)
2. **default**: Normal jobs (image processing)
3. **low**: Background jobs (reports, cleanup)

### 🌟 **Horizon Setup (Advanced)**

**Installation:**
```bash
composer require laravel/horizon
php artisan horizon:install
php artisan horizon:publish
```

**Forge Process Setup:**
```bash
# Command
php artisan horizon

# Auto-restart on code changes
php artisan horizon:terminate
```

**Benefits:**
- Beautiful dashboard
- Job metrics
- Failed job management
- Real-time monitoring

### 📊 **Queue Monitoring**

**Useful Commands:**
```bash
# Check failed jobs
php artisan queue:failed

# Retry all failed jobs
php artisan queue:retry all

# Clear failed jobs
php artisan queue:flush

# Monitor queue length
php artisan queue:monitor redis:default --max=100
```

---

## SSL ও Domain Setup

### 🔒 **Let's Encrypt SSL**

**Setup Steps:**
1. Domain → **"SSL Certificates"**
2. **Certificate Type:** Let's Encrypt
3. **Domains:** 
   - `yourdomain.com`
   - `www.yourdomain.com`
4. **"Obtain Certificate"** click
5. **Auto-renewal** enabled by default

**Verification:**
```bash
# Check SSL status
curl -I https://yourdomain.com

# Check certificate expiry
openssl s_client -connect yourdomain.com:443 -servername yourdomain.com 2>/dev/null | openssl x509 -noout -dates
```

### 🌐 **Multiple Domain Setup**

**Scenario 1: Same App, Multiple Domains**
```bash
# Primary: yourdomain.com
# Aliases: www.yourdomain.com, old-domain.com
```

**Scenario 2: Subdomain for API**
```bash
# Main site: yourdomain.com
# API: api.yourdomain.com (separate site)
```

### 🔄 **Domain Redirects**

**Nginx Configuration (auto-generated):**
```nginx
# HTTP to HTTPS redirect
server {
    listen 80;
    server_name yourdomain.com www.yourdomain.com;
    return 301 https://yourdomain.com$request_uri;
}

# www to non-www redirect
server {
    listen 443 ssl;
    server_name www.yourdomain.com;
    return 301 https://yourdomain.com$request_uri;
}
```

---

## সাধারণ সমস্যা ও সমাধান

### 🚨 **Deploy Issues**

**1. Deploy Success কিন্তু 500 Error:**

```bash
# Check করুন
1. .env file সঠিক আছে কিনা
2. APP_KEY generate হয়েছে কিনা
3. Database connection ঠিক আছে কিনা
4. Storage permission ঠিক আছে কিনা

# Solutions
php artisan key:generate --force
php artisan config:cache
chown -R forge:forge storage bootstrap/cache
chmod -R 775 storage bootstrap/cache
```

**2. Composer Install Failed:**

```bash
# Memory issue
echo 'memory_limit = 512M' >> /etc/php/8.2/cli/php.ini

# Permission issue
chown -R forge:forge /home/forge/.composer
```

**3. Asset Build Failed:**

```bash
# Node version issue
curl -fsSL https://deb.nodesource.com/setup_18.x | sudo -E bash -
sudo apt-get install -y nodejs

# Clear npm cache
npm cache clean --force
rm -rf node_modules package-lock.json
npm install
```

### 🔄 **Queue Issues**

**1. Jobs না চলা:**

```bash
# Check worker status
ps aux | grep "queue:work"

# Check failed jobs
php artisan queue:failed

# Restart workers
php artisan queue:restart

# Check Redis connection
redis-cli ping
```

**2. Memory Leak:**

```bash
# Check memory usage
free -h
ps aux --sort=-%mem | head

# Solution: Add memory limit
php artisan queue:work --memory=512
```

**3. Jobs Stuck:**

```bash
# Clear stuck jobs
php artisan queue:clear

# Restart Redis
sudo service redis-server restart

# Restart workers
php artisan queue:restart
```

### 🌐 **SSL Issues**

**1. SSL Certificate Failed:**

```bash
# Check domain DNS
dig yourdomain.com

# Check domain pointing to server
ping yourdomain.com

# Manual certificate request
sudo certbot --nginx -d yourdomain.com -d www.yourdomain.com
```

**2. Mixed Content Warnings:**

```bash
# Force HTTPS in AppServiceProvider
if (app()->environment('production')) {
    URL::forceScheme('https');
}

# Update .env
APP_URL=https://yourdomain.com
```

### 📊 **Performance Issues**

**1. Slow Response Time:**

```bash
# Enable OPcache
echo 'opcache.enable=1' >> /etc/php/8.2/fpm/php.ini
echo 'opcache.memory_consumption=128' >> /etc/php/8.2/fpm/php.ini

# Optimize Composer autoloader
composer dump-autoload --optimize

# Cache everything
php artisan config:cache
php artisan route:cache
php artisan view:cache
```

**2. High Memory Usage:**

```bash
# Check processes
htop

# Optimize PHP-FPM
# Edit /etc/php/8.2/fpm/pool.d/www.conf
pm.max_children = 10
pm.start_servers = 2
pm.min_spare_servers = 1
pm.max_spare_servers = 3
```

---

## Production Best Practices

### 🔒 **Security Best Practices**

**1. Server Security:**
```bash
# Change default SSH port
sudo nano /etc/ssh/sshd_config
# Port 2222

# Disable root login
PermitRootLogin no

# Use SSH keys only
PasswordAuthentication no
```

**2. Application Security:**
```bash
# Environment variables
APP_DEBUG=false
APP_ENV=production

# Database security
DB_HOST=127.0.0.1  # localhost only
```

**3. Firewall Rules:**
```bash
# Only necessary ports
22 (SSH) - Specific IPs only
80 (HTTP) - All
443 (HTTPS) - All
3306 (MySQL) - Localhost only
```

### ⚡ **Performance Optimization**

**1. Caching Strategy:**
```bash
# Application cache
CACHE_DRIVER=redis

# Session cache
SESSION_DRIVER=redis

# Queue cache
QUEUE_CONNECTION=redis

# View cache (production only)
php artisan view:cache
```

**2. Database Optimization:**
```bash
# Query optimization
php artisan optimize:clear
php artisan config:cache

# Database indexing
Schema::table('posts', function (Blueprint $table) {
    $table->index(['user_id', 'created_at']);
});
```

**3. Asset Optimization:**
```bash
# Vite build optimization
npm run build

# Image optimization
composer require intervention/image
```

### 📊 **Monitoring Setup**

**1. Application Monitoring:**
```bash
# Laravel Telescope (development only)
composer require laravel/telescope --dev

# Laravel Horizon (queue monitoring)
composer require laravel/horizon
```

**2. Server Monitoring:**
```bash
# Basic monitoring script
#!/bin/bash
# monitor.sh

# Check disk space
df -h | grep -E "/$|/home" | awk '{print $5}' | sed 's/%//' | while read usage; do
    if [ $usage -gt 80 ]; then
        echo "Disk usage is above 80%: $usage%"
    fi
done

# Check memory usage
free | grep Mem | awk '{printf("Memory usage: %.2f%%\n", $3/$2 * 100.0)}'

# Check load average
uptime
```

**3. Log Monitoring:**
```bash
# Laravel logs
tail -f storage/logs/laravel.log

# Nginx logs
tail -f /var/log/nginx/access.log
tail -f /var/log/nginx/error.log

# PHP-FPM logs
tail -f /var/log/php8.2-fpm.log
```

### 🔄 **Backup Strategy**

**1. Database Backup:**
```bash
#!/bin/bash
# backup.sh

DATE=$(date +%Y%m%d_%H%M%S)
BACKUP_DIR="/home/forge/backups"
DB_NAME="forge"

# Create backup directory
mkdir -p $BACKUP_DIR

# Database backup
mysqldump -u forge -p$DB_PASSWORD $DB_NAME > $BACKUP_DIR/db_$DATE.sql

# Compress backup
gzip $BACKUP_DIR/db_$DATE.sql

# Keep only last 7 days
find $BACKUP_DIR -name "db_*.sql.gz" -mtime +7 -delete

echo "Backup completed: db_$DATE.sql.gz"
```

**2. File Backup:**
```bash
# Storage backup
tar -czf /home/forge/backups/storage_$(date +%Y%m%d).tar.gz storage/app/public

# Code backup (optional)
tar -czf /home/forge/backups/code_$(date +%Y%m%d).tar.gz --exclude=node_modules --exclude=vendor .
```

**3. Automated Backup (Cron):**
```bash
# Add to crontab
0 2 * * * /home/forge/backup.sh >> /home/forge/backup.log 2>&1
```

### 📈 **Scaling Considerations**

**1. Vertical Scaling (Upgrade Server):**
- 1GB RAM → 2GB RAM
- 1 CPU → 2 CPU
- 25GB SSD → 50GB SSD

**2. Horizontal Scaling (Multiple Servers):**
- Load balancer
- Database server separation
- Redis server separation
- CDN for static assets

**3. Queue Scaling:**
```bash
# Multiple workers
php artisan queue:work redis --queue=high --processes=2
php artisan queue:work redis --queue=default --processes=4
php artisan queue:work redis --queue=low --processes=1
```

---

## 🎯 **Quick Reference Commands**

### Daily Operations:
```bash
# Deploy
git push origin main  # Auto deploy if enabled

# Check logs
tail -f storage/logs/laravel.log

# Check queue
php artisan queue:failed

# Clear cache
php artisan cache:clear
```

### Weekly Maintenance:
```bash
# Update dependencies
composer update
npm update

# Check disk space
df -h

# Check failed jobs
php artisan queue:failed

# Backup database
mysqldump -u forge -p forge > backup_$(date +%Y%m%d).sql
```

### Emergency Commands:
```bash
# Maintenance mode
php artisan down

# Emergency deploy rollback
git reset --hard HEAD~1
php artisan up

# Restart all services
sudo service nginx restart
sudo service php8.2-fpm restart
sudo service mysql restart
sudo service redis-server restart
```

---

এই গাইড follow করে আপনি Laravel Forge দিয়ে professional level এ আপনার Laravel application deploy এবং manage করতে পারবেন। কোন specific সমস্যা হলে Forge এর documentation এবং Laravel community forum check করুন।