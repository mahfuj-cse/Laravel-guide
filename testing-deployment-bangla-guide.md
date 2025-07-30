# 2️⃣0️⃣ Laravel Testing & Production Deployment - বিস্তারিত বাংলা গাইড

## 📋 সূচিপত্র
- [Testing কি?](#testing-কি)
- [PHPUnit vs Pest](#phpunit-vs-pest)
- [Unit Testing](#unit-testing)
- [Feature Testing](#feature-testing)
- [HTTP Testing](#http-testing)
- [Database Testing](#database-testing)
- [Mocking](#mocking)
- [GitHub Actions CI/CD](#github-actions-cicd)
- [Production Deployment](#production-deployment)
- [Webhook Integration](#webhook-integration)

---

## Testing কি?

**Testing** হলো **Code Quality** নিশ্চিত করার প্রক্রিয়া। Laravel এ **PHPUnit** এবং **Pest** দুটোই ব্যবহার করা যায়।

### 🎯 Testing এর সুবিধা:
- ✅ **Bug Detection** - সমস্যা আগেই খুঁজে পাওয়া
- ✅ **Code Quality** - Better code structure
- ✅ **Refactoring Safety** - নিরাপদে code পরিবর্তন
- ✅ **Documentation** - Code behavior documentation
- ✅ **CI/CD Integration** - Automated deployment

### Testing Types:
```
Unit Tests → Feature Tests → Integration Tests → E2E Tests
    ↓              ↓               ↓              ↓
  Classes      Controllers    APIs/Services   Full App
```

---

## PHPUnit vs Pest

### ১. PHPUnit (Traditional):
```php
<?php
// tests/Unit/UserTest.php

namespace Tests\Unit;

use PHPUnit\Framework\TestCase;
use App\Models\User;

class UserTest extends TestCase
{
    public function test_user_can_be_created()
    {
        $user = new User([
            'name' => 'John Doe',
            'email' => 'john@example.com'
        ]);

        $this->assertEquals('John Doe', $user->name);
        $this->assertEquals('john@example.com', $user->email);
    }

    public function test_user_full_name()
    {
        $user = new User([
            'first_name' => 'John',
            'last_name' => 'Doe'
        ]);

        $this->assertEquals('John Doe', $user->getFullNameAttribute());
    }
}
```

### ২. Pest (Modern):
```php
<?php
// tests/Unit/UserTest.php

use App\Models\User;

it('can create a user', function () {
    $user = new User([
        'name' => 'John Doe',
        'email' => 'john@example.com'
    ]);

    expect($user->name)->toBe('John Doe');
    expect($user->email)->toBe('john@example.com');
});

it('returns full name', function () {
    $user = new User([
        'first_name' => 'John',
        'last_name' => 'Doe'
    ]);

    expect($user->getFullNameAttribute())->toBe('John Doe');
});

// Install Pest
composer require pestphp/pest --dev
./vendor/bin/pest --init
```

---

## Unit Testing

### ১. Model Testing:
```php
<?php
// tests/Unit/PostTest.php

use App\Models\Post;
use App\Models\User;

it('belongs to a user', function () {
    $user = User::factory()->create();
    $post = Post::factory()->create(['user_id' => $user->id]);

    expect($post->user)->toBeInstanceOf(User::class);
    expect($post->user->id)->toBe($user->id);
});

it('has many comments', function () {
    $post = Post::factory()->create();
    $comments = Comment::factory(3)->create(['post_id' => $post->id]);

    expect($post->comments)->toHaveCount(3);
});

it('can be published', function () {
    $post = Post::factory()->create(['status' => 'draft']);

    $post->publish();

    expect($post->status)->toBe('published');
    expect($post->published_at)->not->toBeNull();
});

it('generates slug from title', function () {
    $post = new Post(['title' => 'My Awesome Post']);

    expect($post->getSlugAttribute())->toBe('my-awesome-post');
});
```

### ২. Service Testing:
```php
<?php
// tests/Unit/PaymentServiceTest.php

use App\Services\PaymentService;
use App\Models\Order;

beforeEach(function () {
    $this->paymentService = new PaymentService();
});

it('processes payment successfully', function () {
    $order = Order::factory()->create(['total' => 100]);

    $result = $this->paymentService->process($order, [
        'card_number' => '4242424242424242',
        'exp_month' => 12,
        'exp_year' => 2025,
        'cvc' => '123'
    ]);

    expect($result)->toBeTrue();
    expect($order->fresh()->status)->toBe('paid');
});

it('handles payment failure', function () {
    $order = Order::factory()->create(['total' => 100]);

    $result = $this->paymentService->process($order, [
        'card_number' => '4000000000000002', // Declined card
        'exp_month' => 12,
        'exp_year' => 2025,
        'cvc' => '123'
    ]);

    expect($result)->toBeFalse();
    expect($order->fresh()->status)->toBe('payment_failed');
});
```

---

## Feature Testing

### ১. Authentication Testing:
```php
<?php
// tests/Feature/AuthTest.php

use App\Models\User;

it('can register a new user', function () {
    $response = $this->post('/register', [
        'name' => 'John Doe',
        'email' => 'john@example.com',
        'password' => 'password123',
        'password_confirmation' => 'password123'
    ]);

    $response->assertRedirect('/dashboard');
    $this->assertDatabaseHas('users', [
        'email' => 'john@example.com'
    ]);
});

it('can login with valid credentials', function () {
    $user = User::factory()->create([
        'email' => 'john@example.com',
        'password' => bcrypt('password123')
    ]);

    $response = $this->post('/login', [
        'email' => 'john@example.com',
        'password' => 'password123'
    ]);

    $response->assertRedirect('/dashboard');
    $this->assertAuthenticatedAs($user);
});

it('cannot login with invalid credentials', function () {
    $response = $this->post('/login', [
        'email' => 'john@example.com',
        'password' => 'wrongpassword'
    ]);

    $response->assertSessionHasErrors(['email']);
    $this->assertGuest();
});
```

### ২. CRUD Testing:
```php
<?php
// tests/Feature/PostTest.php

use App\Models\User;
use App\Models\Post;

it('can create a post', function () {
    $user = User::factory()->create();

    $response = $this->actingAs($user)->post('/posts', [
        'title' => 'My New Post',
        'content' => 'This is the content of my post',
        'status' => 'published'
    ]);

    $response->assertRedirect();
    $this->assertDatabaseHas('posts', [
        'title' => 'My New Post',
        'user_id' => $user->id
    ]);
});

it('can update a post', function () {
    $user = User::factory()->create();
    $post = Post::factory()->create(['user_id' => $user->id]);

    $response = $this->actingAs($user)->put("/posts/{$post->id}", [
        'title' => 'Updated Title',
        'content' => 'Updated content'
    ]);

    $response->assertRedirect();
    expect($post->fresh()->title)->toBe('Updated Title');
});

it('cannot update others post', function () {
    $user1 = User::factory()->create();
    $user2 = User::factory()->create();
    $post = Post::factory()->create(['user_id' => $user1->id]);

    $response = $this->actingAs($user2)->put("/posts/{$post->id}", [
        'title' => 'Hacked Title'
    ]);

    $response->assertStatus(403);
});
```

---

## HTTP Testing

### ১. API Testing:
```php
<?php
// tests/Feature/Api/PostApiTest.php

use App\Models\User;
use App\Models\Post;
use Laravel\Sanctum\Sanctum;

it('can get posts via api', function () {
    Post::factory(5)->create();

    $response = $this->getJson('/api/posts');

    $response->assertStatus(200)
             ->assertJsonStructure([
                 'data' => [
                     '*' => ['id', 'title', 'content', 'created_at']
                 ]
             ]);
});

it('can create post via api with authentication', function () {
    $user = User::factory()->create();
    Sanctum::actingAs($user);

    $response = $this->postJson('/api/posts', [
        'title' => 'API Post',
        'content' => 'Created via API'
    ]);

    $response->assertStatus(201)
             ->assertJson([
                 'data' => [
                     'title' => 'API Post',
                     'content' => 'Created via API'
                 ]
             ]);
});

it('requires authentication for creating posts', function () {
    $response = $this->postJson('/api/posts', [
        'title' => 'Unauthorized Post'
    ]);

    $response->assertStatus(401);
});

it('validates required fields', function () {
    $user = User::factory()->create();
    Sanctum::actingAs($user);

    $response = $this->postJson('/api/posts', []);

    $response->assertStatus(422)
             ->assertJsonValidationErrors(['title', 'content']);
});
```

### ২. File Upload Testing:
```php
<?php
// tests/Feature/FileUploadTest.php

use Illuminate\Http\UploadedFile;
use Illuminate\Support\Facades\Storage;

it('can upload profile image', function () {
    Storage::fake('public');
    $user = User::factory()->create();

    $file = UploadedFile::fake()->image('avatar.jpg', 300, 300);

    $response = $this->actingAs($user)->post('/profile/avatar', [
        'avatar' => $file
    ]);

    $response->assertRedirect();
    Storage::disk('public')->assertExists('avatars/' . $file->hashName());
});

it('validates image file type', function () {
    $user = User::factory()->create();
    $file = UploadedFile::fake()->create('document.pdf', 1000);

    $response = $this->actingAs($user)->post('/profile/avatar', [
        'avatar' => $file
    ]);

    $response->assertSessionHasErrors(['avatar']);
});
```

---

## Database Testing

### ১. Database Transactions:
```php
<?php
// tests/TestCase.php

namespace Tests;

use Illuminate\Foundation\Testing\TestCase as BaseTestCase;
use Illuminate\Foundation\Testing\RefreshDatabase;

abstract class TestCase extends BaseTestCase
{
    use CreatesApplication, RefreshDatabase;

    protected function setUp(): void
    {
        parent::setUp();
        
        // Seed necessary data
        $this->seed([
            RoleSeeder::class,
            PermissionSeeder::class
        ]);
    }
}
```

### ২. Factory Testing:
```php
<?php
// database/factories/PostFactory.php

namespace Database\Factories;

use App\Models\User;
use App\Models\Category;
use Illuminate\Database\Eloquent\Factories\Factory;

class PostFactory extends Factory
{
    public function definition()
    {
        return [
            'title' => $this->faker->sentence(),
            'content' => $this->faker->paragraphs(3, true),
            'status' => $this->faker->randomElement(['draft', 'published']),
            'user_id' => User::factory(),
            'category_id' => Category::factory(),
        ];
    }

    public function published()
    {
        return $this->state(['status' => 'published']);
    }

    public function draft()
    {
        return $this->state(['status' => 'draft']);
    }
}

// Usage in tests
it('can create published posts', function () {
    $posts = Post::factory(5)->published()->create();

    expect($posts)->toHaveCount(5);
    $posts->each(fn($post) => expect($post->status)->toBe('published'));
});
```

---

## Mocking

### ১. Service Mocking:
```php
<?php
// tests/Feature/OrderTest.php

use App\Services\PaymentGateway;
use App\Services\EmailService;
use Mockery;

it('processes order with mocked services', function () {
    // Mock payment gateway
    $paymentMock = Mockery::mock(PaymentGateway::class);
    $paymentMock->shouldReceive('charge')
                ->once()
                ->with(100)
                ->andReturn(['success' => true, 'transaction_id' => 'txn_123']);

    // Mock email service
    $emailMock = Mockery::mock(EmailService::class);
    $emailMock->shouldReceive('sendOrderConfirmation')
              ->once()
              ->andReturn(true);

    // Bind mocks to container
    $this->app->instance(PaymentGateway::class, $paymentMock);
    $this->app->instance(EmailService::class, $emailMock);

    $user = User::factory()->create();
    
    $response = $this->actingAs($user)->post('/orders', [
        'items' => [
            ['product_id' => 1, 'quantity' => 2]
        ],
        'total' => 100
    ]);

    $response->assertStatus(201);
});
```

### ২. External API Mocking:
```php
<?php
// tests/Feature/WeatherTest.php

use Illuminate\Support\Facades\Http;

it('fetches weather data', function () {
    Http::fake([
        'api.weather.com/*' => Http::response([
            'temperature' => 25,
            'condition' => 'sunny'
        ], 200)
    ]);

    $response = $this->get('/weather/dhaka');

    $response->assertStatus(200)
             ->assertJson([
                 'temperature' => 25,
                 'condition' => 'sunny'
             ]);

    Http::assertSent(function ($request) {
        return $request->url() === 'https://api.weather.com/dhaka';
    });
});
```

---

## GitHub Actions CI/CD

### ১. Basic GitHub Actions:
```yaml
# .github/workflows/tests.yml

name: Tests

on:
  push:
    branches: [ main, develop ]
  pull_request:
    branches: [ main ]

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
    - uses: actions/checkout@v3

    - name: Setup PHP
      uses: shivammathur/setup-php@v2
      with:
        php-version: '8.2'
        extensions: mbstring, dom, fileinfo, mysql, redis
        coverage: xdebug

    - name: Copy .env
      run: php -r "file_exists('.env') || copy('.env.example', '.env');"

    - name: Install Dependencies
      run: composer install -q --no-ansi --no-interaction --no-scripts --no-progress --prefer-dist

    - name: Generate key
      run: php artisan key:generate

    - name: Directory Permissions
      run: chmod -R 777 storage bootstrap/cache

    - name: Create Database
      run: |
        mkdir -p database
        touch database/database.sqlite

    - name: Execute tests (Unit and Feature tests) via PHPUnit
      env:
        DB_CONNECTION: mysql
        DB_HOST: 127.0.0.1
        DB_PORT: 3306
        DB_DATABASE: testing
        DB_USERNAME: root
        DB_PASSWORD: password
        REDIS_HOST: 127.0.0.1
        REDIS_PORT: 6379
      run: vendor/bin/phpunit --coverage-text

    - name: Execute tests via Pest
      env:
        DB_CONNECTION: mysql
        DB_HOST: 127.0.0.1
        DB_PORT: 3306
        DB_DATABASE: testing
        DB_USERNAME: root
        DB_PASSWORD: password
      run: vendor/bin/pest --coverage
```

### ২. Advanced CI/CD Pipeline:
```yaml
# .github/workflows/deploy.yml

name: Deploy to Production

on:
  push:
    branches: [ main ]

jobs:
  test:
    runs-on: ubuntu-latest
    
    steps:
    - uses: actions/checkout@v3
    
    - name: Setup PHP
      uses: shivammathur/setup-php@v2
      with:
        php-version: '8.2'
    
    - name: Install Dependencies
      run: composer install --no-dev --optimize-autoloader
    
    - name: Run Tests
      run: vendor/bin/pest
    
    - name: Build Assets
      run: |
        npm install
        npm run build

  deploy:
    needs: test
    runs-on: ubuntu-latest
    if: github.ref == 'refs/heads/main'
    
    steps:
    - uses: actions/checkout@v3
    
    - name: Deploy to Server
      uses: appleboy/ssh-action@v0.1.5
      with:
        host: ${{ secrets.HOST }}
        username: ${{ secrets.USERNAME }}
        key: ${{ secrets.PRIVATE_KEY }}
        script: |
          cd /var/www/html/myapp
          git pull origin main
          composer install --no-dev --optimize-autoloader
          php artisan migrate --force
          php artisan config:cache
          php artisan route:cache
          php artisan view:cache
          php artisan queue:restart
          sudo systemctl reload nginx
```

---

## Production Deployment

### ১. Server Setup Script:
```bash
#!/bin/bash
# deploy.sh

echo "🚀 Starting deployment..."

# Navigate to project directory
cd /var/www/html/myapp

# Maintenance mode
php artisan down --message="Updating application" --retry=60

# Pull latest changes
git pull origin main

# Install/update dependencies
composer install --no-dev --optimize-autoloader --no-interaction

# Clear and cache config
php artisan config:clear
php artisan config:cache

# Clear and cache routes
php artisan route:clear
php artisan route:cache

# Clear and cache views
php artisan view:clear
php artisan view:cache

# Run migrations
php artisan migrate --force

# Restart queue workers
php artisan queue:restart

# Build assets
npm ci --production
npm run build

# Set permissions
chown -R www-data:www-data storage bootstrap/cache
chmod -R 775 storage bootstrap/cache

# Restart services
sudo systemctl reload nginx
sudo systemctl restart php8.2-fpm

# Exit maintenance mode
php artisan up

echo "✅ Deployment completed successfully!"
```

### ২. Zero-downtime Deployment:
```bash
#!/bin/bash
# zero-downtime-deploy.sh

REPO_URL="https://github.com/username/myapp.git"
DEPLOY_PATH="/var/www/html"
APP_NAME="myapp"
TIMESTAMP=$(date +%Y%m%d_%H%M%S)
RELEASE_PATH="$DEPLOY_PATH/releases/$TIMESTAMP"
CURRENT_PATH="$DEPLOY_PATH/current"
SHARED_PATH="$DEPLOY_PATH/shared"

echo "🚀 Starting zero-downtime deployment..."

# Create directories
mkdir -p $DEPLOY_PATH/releases
mkdir -p $SHARED_PATH/storage/logs
mkdir -p $SHARED_PATH/storage/framework/cache
mkdir -p $SHARED_PATH/storage/framework/sessions
mkdir -p $SHARED_PATH/storage/framework/views

# Clone repository
git clone $REPO_URL $RELEASE_PATH
cd $RELEASE_PATH

# Install dependencies
composer install --no-dev --optimize-autoloader --no-interaction

# Link shared directories
ln -nfs $SHARED_PATH/storage $RELEASE_PATH/storage
ln -nfs $SHARED_PATH/.env $RELEASE_PATH/.env

# Build assets
npm ci --production
npm run build

# Cache optimization
php artisan config:cache
php artisan route:cache
php artisan view:cache

# Run migrations
php artisan migrate --force

# Switch to new release
ln -nfs $RELEASE_PATH $CURRENT_PATH

# Restart services
php artisan queue:restart
sudo systemctl reload nginx

# Keep only last 5 releases
cd $DEPLOY_PATH/releases && ls -t | tail -n +6 | xargs rm -rf

echo "✅ Zero-downtime deployment completed!"
```

---

## Webhook Integration

### ১. GitHub Webhook Handler:
```php
<?php
// app/Http/Controllers/WebhookController.php

namespace App\Http\Controllers;

use Illuminate\Http\Request;
use Illuminate\Support\Facades\Log;
use Symfony\Component\Process\Process;

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
        
        // Only deploy on push to main branch
        if ($data['ref'] === 'refs/heads/main') {
            Log::info('GitHub webhook: Push to main branch detected');
            
            // Queue deployment job
            dispatch(new DeploymentJob($data));
            
            return response('Deployment queued', 200);
        }

        return response('No action needed', 200);
    }
}

// app/Jobs/DeploymentJob.php
class DeploymentJob implements ShouldQueue
{
    protected $webhookData;

    public function __construct($webhookData)
    {
        $this->webhookData = $webhookData;
    }

    public function handle()
    {
        Log::info('Starting deployment from webhook');

        $process = new Process(['/var/www/html/deploy.sh']);
        $process->setTimeout(300); // 5 minutes timeout
        
        $process->run(function ($type, $buffer) {
            Log::info('Deployment output: ' . $buffer);
        });

        if ($process->isSuccessful()) {
            Log::info('Deployment completed successfully');
            
            // Send notification
            $this->sendSlackNotification('✅ Deployment successful');
        } else {
            Log::error('Deployment failed: ' . $process->getErrorOutput());
            
            // Send error notification
            $this->sendSlackNotification('❌ Deployment failed');
        }
    }

    private function sendSlackNotification($message)
    {
        // Send to Slack webhook
        Http::post(config('services.slack.webhook_url'), [
            'text' => $message,
            'channel' => '#deployments',
            'username' => 'Deploy Bot'
        ]);
    }
}
```

### ২. Webhook Route Setup:
```php
<?php
// routes/web.php

Route::post('/webhooks/github', [WebhookController::class, 'github'])
     ->middleware('throttle:10,1')
     ->name('webhooks.github');

// Add to VerifyCsrfToken middleware exceptions
// app/Http/Middleware/VerifyCsrfToken.php
protected $except = [
    'webhooks/*',
];
```

### ৩. Deployment Status API:
```php
<?php
// app/Http/Controllers/DeploymentController.php

class DeploymentController extends Controller
{
    public function status()
    {
        $lastDeployment = Cache::get('last_deployment');
        $isDeploying = Cache::get('deployment_in_progress', false);

        return response()->json([
            'status' => $isDeploying ? 'deploying' : 'idle',
            'last_deployment' => $lastDeployment,
            'current_commit' => $this->getCurrentCommit(),
            'uptime' => $this->getUptime()
        ]);
    }

    private function getCurrentCommit()
    {
        $process = new Process(['git', 'rev-parse', 'HEAD']);
        $process->run();
        
        return trim($process->getOutput());
    }

    private function getUptime()
    {
        $process = new Process(['uptime', '-p']);
        $process->run();
        
        return trim($process->getOutput());
    }
}
```

---

## Testing Commands

### ১. Basic Commands:
```bash
# Run all tests
php artisan test

# Run specific test file
php artisan test tests/Feature/PostTest.php

# Run with coverage
php artisan test --coverage

# Run tests in parallel
php artisan test --parallel

# Pest commands
./vendor/bin/pest
./vendor/bin/pest --coverage
./vendor/bin/pest --parallel
```

### ২. Custom Test Commands:
```php
<?php
// app/Console/Commands/RunTestSuite.php

class RunTestSuite extends Command
{
    protected $signature = 'test:suite {--coverage} {--parallel}';
    protected $description = 'Run complete test suite with options';

    public function handle()
    {
        $this->info('🧪 Running Laravel Test Suite...');

        // Prepare database
        $this->call('migrate:fresh', ['--env' => 'testing']);
        $this->call('db:seed', ['--env' => 'testing']);

        // Build command
        $command = ['./vendor/bin/pest'];
        
        if ($this->option('coverage')) {
            $command[] = '--coverage';
        }
        
        if ($this->option('parallel')) {
            $command[] = '--parallel';
        }

        // Run tests
        $process = new Process($command);
        $process->setTimeout(300);
        
        $process->run(function ($type, $buffer) {
            $this->output->write($buffer);
        });

        if ($process->isSuccessful()) {
            $this->info('✅ All tests passed!');
        } else {
            $this->error('❌ Some tests failed!');
            return 1;
        }

        return 0;
    }
}
```

---

## 🎯 Production Best Practices:

### ✅ **Testing Strategy:**
- **Unit tests** for business logic
- **Feature tests** for user workflows  
- **API tests** for endpoints
- **Database tests** for data integrity

### ✅ **CI/CD Pipeline:**
- **Automated testing** on every push
- **Code quality checks** (PHPStan, Psalm)
- **Security scanning** (Snyk, SonarQube)
- **Performance testing** (Load testing)

### ✅ **Deployment:**
- **Zero-downtime deployment**
- **Database migration safety**
- **Rollback strategy**
- **Health checks**

### ✅ **Monitoring:**
- **Application monitoring** (New Relic, Sentry)
- **Server monitoring** (Datadog, Prometheus)
- **Log aggregation** (ELK Stack)
- **Alert notifications** (Slack, Email)

---

## 📚 আরও জানতে:
- [Laravel Testing](https://laravel.com/docs/testing)
- [Pest PHP](https://pestphp.com/)
- [GitHub Actions](https://docs.github.com/en/actions)
- [Laravel Deployment](https://laravel.com/docs/deployment)