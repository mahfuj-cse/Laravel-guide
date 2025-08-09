# Laravel Forge + GitHub Testing & Deployment - সম্পূর্ণ গাইড

## 📋 সূচিপত্র
- [Environment Setup](#environment-setup)
- [Testing Configuration](#testing-configuration)
- [GitHub Actions Setup](#github-actions-setup)
- [Laravel Forge Configuration](#laravel-forge-configuration)
- [Branch Protection Rules](#branch-protection-rules)
- [Webhook Integration](#webhook-integration)
- [Multiple Server Setup](#multiple-server-setup)
- [Troubleshooting](#troubleshooting)

---

## Environment Setup

### ১. Local Environment (.env.local):
```bash
# .env.local
APP_NAME="My Laravel App"
APP_ENV=local
APP_KEY=base64:your-app-key
APP_DEBUG=true
APP_URL=http://localhost

DB_CONNECTION=mysql
DB_HOST=127.0.0.1
DB_PORT=3306
DB_DATABASE=myapp_local
DB_USERNAME=root
DB_PASSWORD=

CACHE_DRIVER=file
QUEUE_CONNECTION=sync
SESSION_DRIVER=file
```

### ২. Testing Environment (.env.testing):
```bash
# .env.testing
APP_NAME="My Laravel App"
APP_ENV=testing
APP_KEY=base64:your-app-key
APP_DEBUG=true
APP_URL=http://localhost

DB_CONNECTION=sqlite
DB_DATABASE=:memory:

CACHE_DRIVER=array
QUEUE_CONNECTION=sync
SESSION_DRIVER=array
MAIL_MAILER=array

BCRYPT_ROUNDS=4
```

### ৩. Production Environment (.env.production):
```bash
# .env.production (Forge এ set করবেন)
APP_NAME="My Laravel App"
APP_ENV=production
APP_KEY=base64:your-production-key
APP_DEBUG=false
APP_URL=https://yourdomain.com

DB_CONNECTION=mysql
DB_HOST=127.0.0.1
DB_PORT=3306
DB_DATABASE=forge
DB_USERNAME=forge
DB_PASSWORD=your-db-password

CACHE_DRIVER=redis
QUEUE_CONNECTION=redis
SESSION_DRIVER=redis
REDIS_HOST=127.0.0.1
REDIS_PASSWORD=null
REDIS_PORT=6379

MAIL_MAILER=smtp
MAIL_HOST=smtp.mailgun.org
MAIL_PORT=587
MAIL_USERNAME=your-username
MAIL_PASSWORD=your-password
```

---

## Testing Configuration

### ১. PHPUnit Configuration (phpunit.xml):
```xml
<?xml version="1.0" encoding="UTF-8"?>
<phpunit xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
         xsi:noNamespaceSchemaLocation="./vendor/phpunit/phpunit/phpunit.xsd"
         bootstrap="vendor/autoload.php"
         colors="true">
    <testsuites>
        <testsuite name="Unit">
            <directory suffix="Test.php">./tests/Unit</directory>
        </testsuite>
        <testsuite name="Feature">
            <directory suffix="Test.php">./tests/Feature</directory>
        </testsuite>
    </testsuites>
    <coverage processUncoveredFiles="true">
        <include>
            <directory suffix=".php">./app</directory>
        </include>
        <exclude>
            <directory suffix=".php">./app/Console</directory>
            <file>./app/Http/Kernel.php</file>
        </exclude>
    </coverage>
    <php>
        <env name="APP_ENV" value="testing"/>
        <env name="BCRYPT_ROUNDS" value="4"/>
        <env name="CACHE_DRIVER" value="array"/>
        <env name="DB_CONNECTION" value="sqlite"/>
        <env name="DB_DATABASE" value=":memory:"/>
        <env name="MAIL_MAILER" value="array"/>
        <env name="QUEUE_CONNECTION" value="sync"/>
        <env name="SESSION_DRIVER" value="array"/>
        <env name="TELESCOPE_ENABLED" value="false"/>
    </php>
</phpunit>
```

### ২. Sample API Test:
```php
<?php
// tests/Feature/Api/PostApiTest.php

namespace Tests\Feature\Api;

use App\Models\User;
use App\Models\Post;
use Illuminate\Foundation\Testing\RefreshDatabase;
use Laravel\Sanctum\Sanctum;
use Tests\TestCase;

class PostApiTest extends TestCase
{
    use RefreshDatabase;

    protected function setUp(): void
    {
        parent::setUp();
        $this->seed(); // Run necessary seeders
    }

    public function test_can_get_posts_list()
    {
        Post::factory(5)->create();

        $response = $this->getJson('/api/posts');

        $response->assertStatus(200)
                 ->assertJsonStructure([
                     'data' => [
                         '*' => [
                             'id',
                             'title',
                             'content',
                             'created_at',
                             'updated_at'
                         ]
                     ]
                 ]);
    }

    public function test_authenticated_user_can_create_post()
    {
        $user = User::factory()->create();
        Sanctum::actingAs($user);

        $postData = [
            'title' => 'Test Post',
            'content' => 'This is a test post content'
        ];

        $response = $this->postJson('/api/posts', $postData);

        $response->assertStatus(201)
                 ->assertJson([
                     'data' => [
                         'title' => 'Test Post',
                         'content' => 'This is a test post content'
                     ]
                 ]);

        $this->assertDatabaseHas('posts', $postData);
    }

    public function test_unauthenticated_user_cannot_create_post()
    {
        $postData = [
            'title' => 'Test Post',
            'content' => 'This is a test post content'
        ];

        $response = $this->postJson('/api/posts', $postData);

        $response->assertStatus(401);
    }

    public function test_post_creation_validation()
    {
        $user = User::factory()->create();
        Sanctum::actingAs($user);

        $response = $this->postJson('/api/posts', []);

        $response->assertStatus(422)
                 ->assertJsonValidationErrors(['title', 'content']);
    }

    public function test_user_can_update_own_post()
    {
        $user = User::factory()->create();
        $post = Post::factory()->create(['user_id' => $user->id]);
        
        Sanctum::actingAs($user);

        $updateData = [
            'title' => 'Updated Title',
            'content' => 'Updated content'
        ];

        $response = $this->putJson("/api/posts/{$post->id}", $updateData);

        $response->assertStatus(200);
        $this->assertDatabaseHas('posts', $updateData);
    }

    public function test_user_cannot_update_others_post()
    {
        $user1 = User::factory()->create();
        $user2 = User::factory()->create();
        $post = Post::factory()->create(['user_id' => $user1->id]);
        
        Sanctum::actingAs($user2);

        $response = $this->putJson("/api/posts/{$post->id}", [
            'title' => 'Hacked Title'
        ]);

        $response->assertStatus(403);
    }
}
```

---

## GitHub Actions Setup

### ১. Basic GitHub Actions (.github/workflows/tests.yml):
```yaml
name: Laravel Tests

on:
  push:
    branches: [ main, develop, staging ]
  pull_request:
    branches: [ main, develop ]

jobs:
  test:
    runs-on: ubuntu-latest
    
    services:
      mysql:
        image: mysql:8.0
        env:
          MYSQL_ROOT_PASSWORD: password
          MYSQL_DATABASE: testing
        ports:
          - 3306:3306
        options: --health-cmd="mysqladmin ping" --health-interval=10s --health-timeout=5s --health-retries=3

      redis:
        image: redis:alpine
        ports:
          - 6379:6379

    steps:
    - name: Checkout code
      uses: actions/checkout@v4

    - name: Setup PHP
      uses: shivammathur/setup-php@v2
      with:
        php-version: '8.2'
        extensions: mbstring, dom, fileinfo, mysql, redis, gd, zip, bcmath
        coverage: xdebug

    - name: Copy environment file
      run: cp .env.testing .env

    - name: Install Composer dependencies
      run: composer install --no-progress --prefer-dist --optimize-autoloader

    - name: Generate application key
      run: php artisan key:generate

    - name: Set directory permissions
      run: chmod -R 777 storage bootstrap/cache

    - name: Install NPM dependencies
      run: npm ci

    - name: Build assets
      run: npm run build

    - name: Run database migrations
      env:
        DB_CONNECTION: mysql
        DB_HOST: 127.0.0.1
        DB_PORT: 3306
        DB_DATABASE: testing
        DB_USERNAME: root
        DB_PASSWORD: password
      run: php artisan migrate --force

    - name: Seed database
      env:
        DB_CONNECTION: mysql
        DB_HOST: 127.0.0.1
        DB_PORT: 3306
        DB_DATABASE: testing
        DB_USERNAME: root
        DB_PASSWORD: password
      run: php artisan db:seed --force

    - name: Run PHPUnit tests
      env:
        DB_CONNECTION: mysql
        DB_HOST: 127.0.0.1
        DB_PORT: 3306
        DB_DATABASE: testing
        DB_USERNAME: root
        DB_PASSWORD: password
        REDIS_HOST: 127.0.0.1
        REDIS_PORT: 6379
      run: vendor/bin/phpunit --coverage-clover coverage.xml

    - name: Upload coverage to Codecov
      uses: codecov/codecov-action@v3
      with:
        file: ./coverage.xml
        flags: unittests
        name: codecov-umbrella

    - name: Run Laravel Pint (Code Style)
      run: ./vendor/bin/pint --test

    - name: Run PHPStan (Static Analysis)
      run: ./vendor/bin/phpstan analyse --memory-limit=2G
```

---

## Laravel Forge Configuration

### ১. Forge Server Setup:
```bash
# Forge এ Server তৈরি করার সময়:
# 1. Server Provider: DigitalOcean/AWS/Vultr
# 2. Server Size: $20/month minimum for production
# 3. Region: Singapore/Frankfurt (Bangladesh এর কাছে)
# 4. PHP Version: 8.2
# 5. Database: MySQL 8.0
# 6. Additional Software: Redis, Node.js
```

### ২. Site Configuration:
```bash
# Forge Dashboard এ Site তৈরি:
# 1. Root Domain: yourdomain.com
# 2. Project Type: General PHP/Laravel
# 3. Web Directory: /public
# 4. PHP Version: 8.2
```

### ৩. Environment Variables (Forge Dashboard):
```bash
# Forge > Sites > yourdomain.com > Environment

APP_NAME="My Laravel App"
APP_ENV=production
APP_KEY=base64:your-production-key
APP_DEBUG=false
APP_URL=https://yourdomain.com

DB_CONNECTION=mysql
DB_HOST=127.0.0.1
DB_PORT=3306
DB_DATABASE=forge
DB_USERNAME=forge
DB_PASSWORD=your-secure-password

CACHE_DRIVER=redis
QUEUE_CONNECTION=redis
SESSION_DRIVER=redis

MAIL_MAILER=smtp
MAIL_HOST=smtp.mailgun.org
MAIL_PORT=587
MAIL_USERNAME=your-mailgun-username
MAIL_PASSWORD=your-mailgun-password

# GitHub Webhook Secret
GITHUB_WEBHOOK_SECRET=your-webhook-secret
```

### ৪. Deployment Script (Forge):
```bash
# Forge > Sites > yourdomain.com > Deployment Script

cd /home/forge/yourdomain.com

# Enable maintenance mode
$FORGE_PHP artisan down --message="Updating application" --retry=60 || true

# Pull the latest changes
git pull origin $FORGE_SITE_BRANCH

# Install/update composer dependencies
$FORGE_COMPOSER install --no-interaction --prefer-dist --optimize-autoloader --no-dev

# Restart FPM
sudo -S service $FORGE_PHP_FPM reload

# Run database migrations
$FORGE_PHP artisan migrate --force

# Clear caches
$FORGE_PHP artisan config:clear
$FORGE_PHP artisan config:cache

# Clear and cache routes
$FORGE_PHP artisan route:clear
$FORGE_PHP artisan route:cache

# Clear and cache views
$FORGE_PHP artisan view:clear
$FORGE_PHP artisan view:cache

# Install NPM dependencies and build assets
npm ci --production
npm run build

# Restart Queue Workers
$FORGE_PHP artisan queue:restart

# Disable maintenance mode
$FORGE_PHP artisan up
```

---

## Branch Protection Rules

### ১. GitHub Branch Protection Setup:
```bash
# GitHub Repository > Settings > Branches > Add rule

Branch name pattern: main

✅ Require a pull request before merging
  ✅ Require approvals: 1
  ✅ Dismiss stale PR approvals when new commits are pushed
  ✅ Require review from code owners

✅ Require status checks to pass before merging
  ✅ Require branches to be up to date before merging
  Status checks that are required:
    - Laravel Tests
    - test (8.1)
    - test (8.2)

✅ Require conversation resolution before merging
✅ Require signed commits
✅ Include administrators
✅ Restrict pushes that create files larger than 100MB
```

### ২. GitHub Actions Status Check:
```yaml
# .github/workflows/branch-protection.yml
name: Branch Protection

on:
  pull_request:
    branches: [ main, develop ]

jobs:
  tests:
    name: Tests
    runs-on: ubuntu-latest
    
    steps:
    - uses: actions/checkout@v4
    - name: Setup PHP
      uses: shivammathur/setup-php@v2
      with:
        php-version: '8.2'
    
    - name: Install dependencies
      run: composer install
    
    - name: Run tests
      run: vendor/bin/phpunit
    
    - name: Check if tests passed
      run: echo "All tests passed!"

  code-quality:
    name: Code Quality
    runs-on: ubuntu-latest
    
    steps:
    - uses: actions/checkout@v4
    - name: Setup PHP
      uses: shivammathur/setup-php@v2
      with:
        php-version: '8.2'
    
    - name: Install dependencies
      run: composer install
    
    - name: Run PHP CS Fixer
      run: ./vendor/bin/pint --test
    
    - name: Run PHPStan
      run: ./vendor/bin/phpstan analyse
```

---

## Webhook Integration

### ১. GitHub Webhook Setup:
```bash
# GitHub Repository > Settings > Webhooks > Add webhook

Payload URL: https://yourdomain.com/webhooks/github
Content type: application/json
Secret: your-webhook-secret

Which events would you like to trigger this webhook?
✅ Just the push event
```

### ২. Laravel Webhook Handler:
```php
<?php
// app/Http/Controllers/WebhookController.php

namespace App\Http\Controllers;

use Illuminate\Http\Request;
use Illuminate\Support\Facades\Log;
use Illuminate\Support\Facades\Http;

class WebhookController extends Controller
{
    public function github(Request $request)
    {
        // Verify GitHub signature
        $signature = $request->header('X-Hub-Signature-256');
        $payload = $request->getContent();
        $secret = config('services.github.webhook_secret');
        
        $expectedSignature = 'sha256=' . hash_hmac('sha256', $payload, $secret);
        
        if (!hash_equals($signature, $expectedSignature)) {
            Log::warning('Invalid GitHub webhook signature');
            return response('Unauthorized', 401);
        }

        $data = $request->json()->all();
        
        // Log webhook data
        Log::info('GitHub webhook received', [
            'ref' => $data['ref'] ?? null,
            'commits' => count($data['commits'] ?? []),
            'repository' => $data['repository']['name'] ?? null
        ]);

        // Only deploy on push to main branch
        if (($data['ref'] ?? '') === 'refs/heads/main') {
            Log::info('Deploying from main branch push');
            
            // Trigger Forge deployment
            $this->triggerForgeDeployment();
            
            return response('Deployment triggered', 200);
        }

        return response('No deployment needed', 200);
    }

    private function triggerForgeDeployment()
    {
        // Use Forge API
        $forgeToken = config('services.forge.token');
        $serverId = config('services.forge.server_id');
        $siteId = config('services.forge.site_id');

        Http::withToken($forgeToken)
            ->post("https://forge.laravel.com/api/v1/servers/{$serverId}/sites/{$siteId}/deployment/deploy");
    }
}
```

### ৩. Webhook Route:
```php
<?php
// routes/web.php

Route::post('/webhooks/github', [WebhookController::class, 'github'])
     ->middleware('throttle:10,1')
     ->name('webhooks.github');

// Add to CSRF exceptions
// app/Http/Middleware/VerifyCsrfToken.php
protected $except = [
    'webhooks/*',
];
```

---

## Multiple Server Setup

### ১. Staging Server Configuration:
```bash
# Forge এ Staging Server:
# Domain: staging.yourdomain.com
# Branch: develop
# Environment: staging

# Staging Environment Variables:
APP_ENV=staging
APP_DEBUG=true
APP_URL=https://staging.yourdomain.com
DB_DATABASE=staging_db

# Staging Deployment Script:
cd /home/forge/staging.yourdomain.com
git pull origin develop
composer install --optimize-autoloader
php artisan migrate --force
php artisan config:cache
php artisan queue:restart
```

### ২. Production Server Configuration:
```bash
# Forge এ Production Server:
# Domain: yourdomain.com
# Branch: main
# Environment: production

# Production Environment Variables:
APP_ENV=production
APP_DEBUG=false
APP_URL=https://yourdomain.com

# Production Deployment Script (Zero Downtime):
cd /home/forge/yourdomain.com

# Maintenance mode
php artisan down --message="Updating..." --retry=60

# Deploy
git pull origin main
composer install --no-dev --optimize-autoloader
php artisan migrate --force
php artisan config:cache
php artisan route:cache
php artisan view:cache
npm run build
php artisan queue:restart

# Exit maintenance mode
php artisan up
```

---

## 🎯 Complete Setup Checklist:

### ✅ **Local Development:**
- [ ] .env.testing configured
- [ ] Tests written and passing
- [ ] Code style checks (Pint/PHP CS Fixer)
- [ ] Static analysis (PHPStan/Psalm)

### ✅ **GitHub Setup:**
- [ ] Repository created
- [ ] Branch protection rules enabled
- [ ] GitHub Actions workflow configured
- [ ] Secrets added (server credentials)
- [ ] Webhook configured

### ✅ **Laravel Forge:**
- [ ] Server provisioned
- [ ] Site created and configured
- [ ] Environment variables set
- [ ] Deployment script configured
- [ ] SSL certificate installed
- [ ] Database configured

### ✅ **Production Ready:**
- [ ] All tests passing
- [ ] Staging environment working
- [ ] Production deployment successful
- [ ] Monitoring setup (logs, errors)
- [ ] Backup strategy implemented

এই গাইড অনুসরণ করে আপনি একটি সম্পূর্ণ Professional Laravel API Production Setup তৈরি করতে পারবেন যেখানে GitHub এ code push করলে automatically test চলবে এবং test pass হলে Forge এ deploy হবে!