# Laravel টেস্টিং এবং GitHub Actions - সম্পূর্ণ বাংলা গাইড

এই গাইডটিতে আমরা Laravel অ্যাপ্লিকেশন টেস্টিং এবং GitHub Actions ব্যবহার করে কিভাবে স্বয়ংক্রিয়ভাবে টেস্ট চালানো যায়, তা বিস্তারিত আলোচনা করবো।

## 📋 সূচিপত্র
- [Laravel টেস্টিং কি?](#laravel-টেস্টিং-কি)
- [টেস্টিং এর প্রকারভেদ](#টেস্টিং-এর-প্রকারভেদ)
- [Laravel এ টেস্ট লেখার পদ্ধতি](#laravel-এ-টেস্ট-লেখার-পদ্ধতি)
- [GitHub Actions কি?](#github-actions-কি)
- [CI/CD Pipeline Setup](#cicd-pipeline-setup)
- [Workflow ফাইলের বিস্তারিত আলোচনা](#workflow-ফাইলের-বিস্তারিত-আলোচনা)
- [Advanced Testing Strategies](#advanced-testing-strategies)
- [Performance Optimization](#performance-optimization)
- [Security ও Best Practices](#security-ও-best-practices)
- [Troubleshooting Guide](#troubleshooting-guide)

---

## Laravel টেস্টিং কি?

Laravel টেস্টিং হলো আপনার অ্যাপ্লিকেশনের বিভিন্ন অংশ (যেমন, কোড, ফাংশনালিটি, API) সঠিকভাবে কাজ করছে কিনা তা স্বয়ংক্রিয়ভাবে যাচাই করার একটি প্রক্রিয়া। এটি কোডের গুণমান নিশ্চিত করে এবং ভবিষ্যতে কোনো পরিবর্তন আনার ফলে পুরনো কোডে সমস্যা হচ্ছে কিনা (Regression) তা ধরতে সাহায্য করে।

**প্রধানত দুই ধরনের টেস্ট হয়:**
1.  **Unit Test**: ছোট ছোট কোডের অংশ (যেমন, একটি নির্দিষ্ট মেথড) আলাদাভাবে টেস্ট করা হয়।
2.  **Feature Test**: অ্যাপ্লিকেশনের বড় কোনো ফিচার (যেমন, একজন ইউজার রেজিস্ট্রেশন থেকে শুরু করে লগইন পর্যন্ত) টেস্ট করা হয়।

Laravel এ PHPUnit এবং Pest ব্যবহার করে খুব সহজেই টেস্ট লেখা যায়।

**টেস্ট তৈরির কমান্ড:**
```bash
# একটি Feature Test তৈরির জন্য
php artisan make:test UserRegistrationTest

# একটি Unit Test তৈরির জন্য
php artisan make:test UserTest --unit

# টেস্ট চালানোর কমান্ড
php artisan test

# নির্দিষ্ট টেস্ট চালানো
php artisan test --filter UserRegistrationTest

# Coverage সহ টেস্ট
php artisan test --coverage
```

### 🎯 **টেস্টিং এর সুবিধা:**
- ✅ **Bug Detection:** কোডে সমস্যা তাড়াতাড়ি ধরা পড়ে
- ✅ **Regression Prevention:** নতুন কোড পুরনো ফিচার ভাঙে কিনা জানা যায়
- ✅ **Code Quality:** কোডের মান উন্নত হয়
- ✅ **Documentation:** টেস্ট কোড documentation হিসেবে কাজ করে
- ✅ **Confidence:** Deploy করার সময় আত্মবিশ্বাস বাড়ে

---

## টেস্টিং এর প্রকারভেদ

### 🔬 **Unit Testing**
ছোট ছোট কোডের অংশ (method, class) আলাদাভাবে টেস্ট করা।

**উদাহরণ:**
```php
// tests/Unit/UserTest.php
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
    
    public function test_user_full_name_method()
    {
        $user = new User([
            'first_name' => 'John',
            'last_name' => 'Doe'
        ]);
        
        $this->assertEquals('John Doe', $user->getFullName());
    }
}
```

### 🌐 **Feature Testing**
পুরো ফিচার বা user journey টেস্ট করা।

**উদাহরণ:**
```php
// tests/Feature/UserRegistrationTest.php
class UserRegistrationTest extends TestCase
{
    use RefreshDatabase;
    
    public function test_user_can_register()
    {
        $response = $this->post('/register', [
            'name' => 'John Doe',
            'email' => 'john@example.com',
            'password' => 'password',
            'password_confirmation' => 'password'
        ]);
        
        $response->assertRedirect('/dashboard');
        $this->assertDatabaseHas('users', [
            'email' => 'john@example.com'
        ]);
    }
    
    public function test_user_cannot_register_with_invalid_email()
    {
        $response = $this->post('/register', [
            'name' => 'John Doe',
            'email' => 'invalid-email',
            'password' => 'password'
        ]);
        
        $response->assertSessionHasErrors('email');
        $this->assertDatabaseMissing('users', [
            'email' => 'invalid-email'
        ]);
    }
}
```

### 🔗 **Integration Testing**
বিভিন্ন component একসাথে কাজ করছে কিনা টেস্ট করা।

**উদাহরণ:**
```php
// tests/Feature/OrderProcessingTest.php
class OrderProcessingTest extends TestCase
{
    use RefreshDatabase;
    
    public function test_complete_order_flow()
    {
        // User তৈরি
        $user = User::factory()->create();
        
        // Product তৈরি
        $product = Product::factory()->create(['price' => 100]);
        
        // Order তৈরি
        $response = $this->actingAs($user)->post('/orders', [
            'product_id' => $product->id,
            'quantity' => 2
        ]);
        
        // Order সফল হয়েছে কিনা
        $response->assertStatus(201);
        
        // Database এ order আছে কিনা
        $this->assertDatabaseHas('orders', [
            'user_id' => $user->id,
            'product_id' => $product->id,
            'total_amount' => 200
        ]);
        
        // Email পাঠানো হয়েছে কিনা
        Mail::assertSent(OrderConfirmationMail::class);
    }
}
```

### 🌍 **API Testing**
API endpoints টেস্ট করা।

**উদাহরণ:**
```php
// tests/Feature/Api/PostApiTest.php
class PostApiTest extends TestCase
{
    use RefreshDatabase;
    
    public function test_can_get_all_posts()
    {
        Post::factory()->count(3)->create();
        
        $response = $this->getJson('/api/posts');
        
        $response->assertStatus(200)
                ->assertJsonCount(3, 'data')
                ->assertJsonStructure([
                    'data' => [
                        '*' => ['id', 'title', 'content', 'created_at']
                    ]
                ]);
    }
    
    public function test_can_create_post_with_authentication()
    {
        $user = User::factory()->create();
        
        $response = $this->actingAs($user, 'api')
                        ->postJson('/api/posts', [
                            'title' => 'Test Post',
                            'content' => 'Test Content'
                        ]);
        
        $response->assertStatus(201)
                ->assertJson([
                    'data' => [
                        'title' => 'Test Post',
                        'content' => 'Test Content'
                    ]
                ]);
    }
}
```

---

## Laravel এ টেস্ট লেখার পদ্ধতি

### 🏗️ **Test Structure**

**Arrange-Act-Assert Pattern:**
```php
public function test_user_can_update_profile()
{
    // Arrange - টেস্টের জন্য data setup
    $user = User::factory()->create();
    $this->actingAs($user);
    
    // Act - যে action টেস্ট করবো
    $response = $this->put('/profile', [
        'name' => 'Updated Name',
        'email' => 'updated@example.com'
    ]);
    
    // Assert - expected result check
    $response->assertRedirect('/profile');
    $this->assertDatabaseHas('users', [
        'id' => $user->id,
        'name' => 'Updated Name'
    ]);
}
```

### 🏭 **Factory ব্যবহার**

**Factory তৈরি:**
```bash
php artisan make:factory PostFactory
```

**Factory Definition:**
```php
// database/factories/PostFactory.php
class PostFactory extends Factory
{
    public function definition()
    {
        return [
            'title' => $this->faker->sentence(),
            'content' => $this->faker->paragraphs(3, true),
            'user_id' => User::factory(),
            'published_at' => $this->faker->dateTimeBetween('-1 year', 'now')
        ];
    }
    
    public function published()
    {
        return $this->state([
            'published_at' => now()
        ]);
    }
    
    public function draft()
    {
        return $this->state([
            'published_at' => null
        ]);
    }
}
```

**Factory ব্যবহার:**
```php
// Single post
$post = Post::factory()->create();

// Multiple posts
$posts = Post::factory()->count(5)->create();

// With specific state
$publishedPost = Post::factory()->published()->create();
$draftPost = Post::factory()->draft()->create();

// With relationships
$user = User::factory()->create();
$post = Post::factory()->for($user)->create();
```

### 🗄️ **Database Testing**

**Database Assertions:**
```php
// Database এ data আছে কিনা
$this->assertDatabaseHas('users', ['email' => 'test@example.com']);

// Database এ data নেই কিনা
$this->assertDatabaseMissing('users', ['email' => 'fake@example.com']);

// Database count check
$this->assertDatabaseCount('posts', 5);

// Soft deleted model check
$this->assertSoftDeleted('posts', ['id' => $post->id]);
```

**Database Transactions:**
```php
class UserTest extends TestCase
{
    use RefreshDatabase; // প্রতি টেস্টে database reset
    
    // অথবা
    use DatabaseTransactions; // টেস্ট শেষে rollback
}
```

### 📧 **Mail Testing**

```php
use Illuminate\Support\Facades\Mail;

public function test_welcome_email_is_sent()
{
    Mail::fake();
    
    $user = User::factory()->create();
    
    // Email পাঠানোর action
    event(new UserRegistered($user));
    
    // Email পাঠানো হয়েছে কিনা
    Mail::assertSent(WelcomeEmail::class, function ($mail) use ($user) {
        return $mail->user->id === $user->id;
    });
    
    // নির্দিষ্ট সংখ্যক email পাঠানো হয়েছে কিনা
    Mail::assertSentTimes(WelcomeEmail::class, 1);
}
```

### 🔔 **Notification Testing**

```php
use Illuminate\Support\Facades\Notification;

public function test_user_receives_order_notification()
{
    Notification::fake();
    
    $user = User::factory()->create();
    $order = Order::factory()->for($user)->create();
    
    // Notification পাঠানোর action
    $user->notify(new OrderPlaced($order));
    
    // Notification পাঠানো হয়েছে কিনা
    Notification::assertSentTo($user, OrderPlaced::class);
}
```

### 🎯 **Event Testing**

```php
use Illuminate\Support\Facades\Event;

public function test_user_registration_fires_event()
{
    Event::fake();
    
    $response = $this->post('/register', [
        'name' => 'John Doe',
        'email' => 'john@example.com',
        'password' => 'password'
    ]);
    
    Event::assertDispatched(UserRegistered::class, function ($event) {
        return $event->user->email === 'john@example.com';
    });
}
```

---

## GitHub Actions কি?

**GitHub Actions** হলো একটি **CI/CD (Continuous Integration/Continuous Deployment)** টুল যা GitHub এর সাথে বিল্ট-ইন থাকে। এর মাধ্যমে আপনি আপনার GitHub রিপোজিটরিতে কোড `push` বা `pull request` করার মতো বিভিন্ন ইভেন্টের উপর ভিত্তি করে স্বয়ংক্রিয়ভাবে বিভিন্ন কাজ (যেমন, টেস্টিং, বিল্ড, ডেপ্লয়) চালাতে পারেন।

এই কাজগুলো **workflow** নামক YAML ফাইলে লেখা হয়, যা `.github/workflows` ডিরেক্টরিতে থাকে।

### 🎯 **GitHub Actions এর সুবিধা:**
- ✅ **Automated Testing:** প্রতি commit এ automatic test
- ✅ **Code Quality:** Automated code review
- ✅ **Security Scanning:** Vulnerability detection
- ✅ **Deployment:** Automatic deployment
- ✅ **Integration:** Third-party services integration

---

## CI/CD Pipeline Setup

### 🔄 **Complete CI/CD Workflow**

```yaml
name: Laravel CI/CD Pipeline

on:
  push:
    branches: [ main, develop ]
  pull_request:
    branches: [ main, develop ]

jobs:
  test:
    runs-on: ubuntu-latest
    
    strategy:
      matrix:
        php-version: [8.1, 8.2]
        
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
        image: redis:7.0
        ports:
          - 6379:6379
        options: --health-cmd="redis-cli ping" --health-interval=10s --health-timeout=5s --health-retries=3

    steps:
    - name: Checkout code
      uses: actions/checkout@v4

    - name: Setup PHP
      uses: shivammathur/setup-php@v2
      with:
        php-version: ${{ matrix.php-version }}
        extensions: mbstring, dom, fileinfo, mysql, gd, redis
        coverage: xdebug

    - name: Cache Composer dependencies
      uses: actions/cache@v3
      with:
        path: ~/.composer/cache
        key: ${{ runner.os }}-composer-${{ hashFiles('**/composer.lock') }}
        restore-keys: ${{ runner.os }}-composer-

    - name: Install Composer dependencies
      run: composer install --no-progress --prefer-dist --optimize-autoloader

    - name: Setup Node.js
      uses: actions/setup-node@v3
      with:
        node-version: '18'
        cache: 'npm'

    - name: Install NPM dependencies
      run: npm ci

    - name: Build assets
      run: npm run build

    - name: Copy environment file
      run: cp .env.example .env

    - name: Generate application key
      run: php artisan key:generate

    - name: Set directory permissions
      run: chmod -R 777 storage bootstrap/cache

    - name: Run database migrations
      env:
        DB_CONNECTION: mysql
        DB_HOST: 127.0.0.1
        DB_PORT: 3306
        DB_DATABASE: testing
        DB_USERNAME: root
        DB_PASSWORD: password
        REDIS_HOST: 127.0.0.1
        REDIS_PORT: 6379
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
      run: php artisan test --coverage --min=80

    - name: Upload coverage reports
      uses: codecov/codecov-action@v3
      with:
        file: ./coverage.xml
        flags: unittests
        name: codecov-umbrella

  code-quality:
    runs-on: ubuntu-latest
    
    steps:
    - name: Checkout code
      uses: actions/checkout@v4

    - name: Setup PHP
      uses: shivammathur/setup-php@v2
      with:
        php-version: '8.2'
        extensions: mbstring, dom, fileinfo
        tools: phpstan, php-cs-fixer

    - name: Install dependencies
      run: composer install --no-progress --prefer-dist

    - name: Run PHPStan
      run: vendor/bin/phpstan analyse

    - name: Run PHP CS Fixer
      run: vendor/bin/php-cs-fixer fix --dry-run --diff

  security:
    runs-on: ubuntu-latest
    
    steps:
    - name: Checkout code
      uses: actions/checkout@v4

    - name: Setup PHP
      uses: shivammathur/setup-php@v2
      with:
        php-version: '8.2'

    - name: Install dependencies
      run: composer install --no-progress --prefer-dist

    - name: Run security audit
      run: composer audit

  deploy:
    needs: [test, code-quality, security]
    runs-on: ubuntu-latest
    if: github.ref == 'refs/heads/main' && github.event_name == 'push'
    
    steps:
    - name: Deploy to production
      run: |
        echo "Deploying to production server..."
        # Add your deployment commands here
```

### 🎯 **Matrix Testing**

বিভিন্ন PHP version এ টেস্ট চালানোর জন্য:

```yaml
strategy:
  matrix:
    php-version: [8.1, 8.2, 8.3]
    laravel-version: [10.x, 11.x]
    dependency-version: [prefer-lowest, prefer-stable]
```

### 📊 **Parallel Testing**

```yaml
- name: Run tests in parallel
  run: php artisan test --parallel --processes=4
```

---

## Workflow ফাইলের বিস্তারিত আলোচনা

আপনার দেওয়া workflow ফাইলটি একটি চমৎকার উদাহরণ যা শুধুমাত্র পরিবর্তিত ফাইলগুলোর জন্য টেস্ট চালায়। আসুন এর প্রতিটি অংশ বিস্তারিত বুঝি।

```yaml
name: Run Tests

on:
  pull_request:
    branches: [ main, test ]
    paths:
      - 'tests/**'
      - 'app/**'
      - 'composer.json'
      - 'composer.lock'
  push:
    branches: [ main, test ]
    paths:
      - 'tests/**'
      - 'app/**'
      - 'composer.json'
      - 'composer.lock'

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

    steps:
    - uses: actions/checkout@v3

    - name: Setup PHP
      uses: shivammathur/setup-php@v2
      with:
        php-version: '8.1'
        extensions: mbstring, dom, fileinfo, mysql, gd
        coverage: xdebug

    - name: Install Dependencies
      run: composer install -q --no-ansi --no-interaction --no-scripts --no-progress --prefer-dist

    - name: Set Environment Variables
      run: |
        cp .env.example .env
        php artisan key:generate

    - name: Directory Permissions
      run: chmod -R 777 storage bootstrap/cache

    - name: Run Database Migrations
      env:
        DB_CONNECTION: mysql
        DB_HOST: 127.0.0.1
        DB_PORT: 3306
        DB_DATABASE: testing
        DB_USERNAME: root
        DB_PASSWORD: password
      run: php artisan migrate --force

    - name: Get changed test files
      id: changed-files
      uses: tj-actions/changed-files@v39
      with:
        files: tests/**/*.php

    - name: Run Tests
      env:
        DB_CONNECTION: mysql
        DB_HOST: 127.0.0.1
        DB_PORT: 3306
        DB_DATABASE: testing
        DB_USERNAME: root
        DB_PASSWORD: password
      run: |
        if [ "${{ steps.changed-files.outputs.any_changed }}" == "true" ]; then
          echo "Running tests for changed files:"
          echo "${{ steps.changed-files.outputs.all_changed_files }}"
          TEST_FAILED=0
          for file in ${{ steps.changed-files.outputs.all_changed_files }}; do
            if ! php artisan test "$file"; then
              TEST_FAILED=1
            fi
          done
          if [ $TEST_FAILED -eq 1 ]; then
            echo "❌ Tests failed! PR cannot be merged."
            exit 1
          fi
          echo "✅ All tests passed!"
        else
          echo "No test files changed, skipping tests"
        fi
```

### Workflow Triggers (`on`)

এই অংশটি নির্ধারণ করে কখন workflow টি চলবে।

- **`pull_request`**: যখন `main` বা `test` ব্রাঞ্চে কোনো pull request করা হয়।
- **`push`**: যখন `main` বা `test` ব্রাঞ্চে সরাসরি কোড push করা হয়।
- **`paths`**: এই workflow শুধুমাত্র তখনই চলবে যদি `tests/`, `app/` ফোল্ডারের কোনো ফাইল অথবা `composer.json` / `composer.lock` ফাইল পরিবর্তন হয়। এটি অপ্রয়োজনীয় টেস্ট রান বন্ধ করে রিসোর্স বাঁচায়।

### Jobs এবং Services

- **`jobs`**: একটি workflow-তে এক বা একাধিক job থাকতে পারে। এখানে `test` নামে একটি job আছে।
- **`runs-on: ubuntu-latest`**: এই job-টি GitHub-hosted একটি Ubuntu runner-এ চলবে।
- **`services`**: job চলাকালীন কোনো সার্ভিস (যেমন, ডাটাবেস, Redis) প্রয়োজন হলে এখানে define করা হয়।
  - **`mysql: image: mysql:8.0`**: টেস্টের জন্য একটি MySQL 8.0 ডাটাবেস কন্টেইনার চালু করা হয়েছে।
  - **`env`**: ডাটাবেসের জন্য প্রয়োজনীয় এনভায়রনমেন্ট ভ্যারিয়েবল (root password, database name) সেট করা হয়েছে।
  - **`ports`**: কন্টেইনারের 3306 পোর্টটি runner-এর 3306 পোর্টের সাথে ম্যাপ করা হয়েছে, যাতে `127.0.0.1:3306` দিয়ে কানেক্ট করা যায়।
  - **`options`**: MySQL সার্ভিসটি সঠিকভাবে চালু হয়েছে কিনা তা নিশ্চিত করার জন্য হেলথ চেক কমান্ড দেওয়া হয়েছে।

### Workflow Steps

`steps` অংশে job-এর প্রতিটি ধাপ ক্রমানুসারে লেখা থাকে।

1.  **`uses: actions/checkout@v3`**:
    - এই action-টি আপনার রিপোজিটরির কোড runner-এ checkout (download) করে।

2.  **`name: Setup PHP`**:
    - `shivammathur/setup-php@v2` action ব্যবহার করে PHP 8.1 ইনস্টল করা হয়।
    - `extensions`: Laravel এর জন্য প্রয়োজনীয় PHP এক্সটেনশনগুলো (mbstring, mysql, gd ইত্যাদি) ইনস্টল করা হয়।
    - `coverage: xdebug`: টেস্ট কভারেজ রিপোর্ট তৈরির জন্য Xdebug ইনস্টল করা হয়।

3.  **`name: Install Dependencies`**:
    - `composer install` কমান্ডের মাধ্যমে প্রজেক্টের সব PHP dependency ইনস্টল করা হয়।
    - `-q --no-ansi ...`: এই ফ্ল্যাগগুলো CI পরিবেশে দ্রুত এবং অপ্রয়োজনীয় আউটপুট ছাড়া ইনস্টলেশন নিশ্চিত করে।

4.  **`name: Set Environment Variables`**:
    - `.env.example` ফাইল থেকে `.env` ফাইল তৈরি করা হয় এবং `php artisan key:generate` দিয়ে অ্যাপলিকেশন কী জেনারেট করা হয়। এটি Laravel এর জন্য একটি প্রয়োজনীয় ধাপ।

5.  **`name: Directory Permissions`**:
    - Laravel এর `storage` এবং `bootstrap/cache` ডিরেক্টরিগুলোতে লেখার অনুমতি (permission) দেওয়া হয়।

6.  **`name: Run Database Migrations`**:
    - `php artisan migrate --force` কমান্ডের মাধ্যমে `testing` ডাটাবেসে সব মাইগ্রেশন চালানো হয়।
    - `env` ব্লকে ডাটাবেস কানেকশনের জন্য প্রয়োজনীয় তথ্য (host, port, database, username, password) দেওয়া হয়েছে, যা `services` অংশে define করা MySQL কন্টেইনারের সাথে মিলে যায়।

7.  **`name: Get changed test files`**:
    - এটি একটি দারুণ অপটিমাইজেশন। `tj-actions/changed-files@v39` action-টি ব্যবহার করে শুধুমাত্র সেই টেস্ট ফাইলগুলো (`tests/**/*.php`) খুঁজে বের করা হয় যেগুলো বর্তমান push বা pull request-এ পরিবর্তন করা হয়েছে।
    - এর আউটপুট (`steps.changed-files.outputs.all_changed_files`) পরবর্তী ধাপে ব্যবহার করা হয়।

8.  **`name: Run Tests`**:
    - **Conditional Logic**: প্রথমে চেক করা হয় `any_changed` আউটপুট `true` কিনা।
    - **যদি টেস্ট ফাইল পরিবর্তন হয়**:
      - একটি `for` লুপের মাধ্যমে পরিবর্তিত প্রতিটি টেস্ট ফাইলের জন্য `php artisan test "$file"` কমান্ডটি আলাদাভাবে চালানো হয়।
      - যদি কোনো একটি টেস্ট ফেইল করে, `TEST_FAILED` ভ্যারিয়েবল `1` সেট করা হয় এবং শেষে `exit 1` দিয়ে workflow-টি ফেইল করানো হয়।
      - সব টেস্ট পাস করলে "All tests passed!" মেসেজ দেখানো হয়।
    - **যদি কোনো টেস্ট ফাইল পরিবর্তন না হয়**:
      - টেস্ট চালানো স্কিপ করা হয় এবং একটি মেসেজ দেখানো হয়। এই পদ্ধতিটি CI রান টাইম উল্লেখযোগ্যভাবে কমিয়ে আনে।

---

## Advanced Testing Strategies

### 🧪 **Test-Driven Development (TDD)**

**TDD Cycle:**
1. **Red:** প্রথমে failing test লিখুন
2. **Green:** Test pass করার জন্য minimum code লিখুন
3. **Refactor:** Code improve করুন

**উদাহরণ:**
```php
// Step 1: Red - Failing test
public function test_user_can_calculate_age()
{
    $user = new User(['birth_date' => '1990-01-01']);
    $this->assertEquals(34, $user->getAge()); // This will fail
}

// Step 2: Green - Minimum code
class User extends Model
{
    public function getAge()
    {
        return Carbon::parse($this->birth_date)->age;
    }
}

// Step 3: Refactor - Improve code
class User extends Model
{
    protected $dates = ['birth_date'];
    
    public function getAge(): int
    {
        return $this->birth_date?->age ?? 0;
    }
}
```

### 🎭 **Mocking ও Stubbing**

**External Service Mock:**
```php
use Illuminate\Support\Facades\Http;

public function test_can_fetch_user_data_from_api()
{
    Http::fake([
        'api.example.com/users/*' => Http::response([
            'id' => 1,
            'name' => 'John Doe',
            'email' => 'john@example.com'
        ], 200)
    ]);
    
    $service = new UserApiService();
    $user = $service->fetchUser(1);
    
    $this->assertEquals('John Doe', $user['name']);
    
    Http::assertSent(function ($request) {
        return $request->url() === 'https://api.example.com/users/1';
    });
}
```

**Queue Mock:**
```php
use Illuminate\Support\Facades\Queue;

public function test_job_is_dispatched_on_user_creation()
{
    Queue::fake();
    
    $user = User::factory()->create();
    
    Queue::assertPushed(SendWelcomeEmail::class, function ($job) use ($user) {
        return $job->user->id === $user->id;
    });
}
```

### 🔄 **Custom Assertions**

```php
// tests/TestCase.php
abstract class TestCase extends BaseTestCase
{
    public function assertValidJson($response)
    {
        $this->assertJson($response->getContent());
        $this->assertTrue($response->isSuccessful());
    }
    
    public function assertHasValidationError($response, $field)
    {
        $response->assertStatus(422);
        $response->assertJsonValidationErrors($field);
    }
    
    public function assertUserCan($user, $ability, $model = null)
    {
        $this->assertTrue($user->can($ability, $model));
    }
}
```

---

## Performance Optimization

### ⚡ **Fast Testing Tips**

**1. Database Optimization:**
```php
// Use in-memory SQLite for faster tests
// phpunit.xml
<env name="DB_CONNECTION" value="sqlite"/>
<env name="DB_DATABASE" value=":memory:"/>
```

**2. Selective Testing:**
```bash
# শুধু Unit tests
php artisan test --testsuite=Unit

# শুধু Feature tests
php artisan test --testsuite=Feature

# Specific group
php artisan test --group=api
```

**3. Parallel Execution:**
```bash
# Parallel testing (Laravel 8+)
php artisan test --parallel

# Custom process count
php artisan test --parallel --processes=8
```

**4. Test Caching:**
```yaml
# GitHub Actions cache
- name: Cache test results
  uses: actions/cache@v3
  with:
    path: |
      .phpunit.result.cache
      bootstrap/cache
    key: test-cache-${{ hashFiles('tests/**/*.php') }}
```

### 📈 **Performance Monitoring**

```php
// tests/TestCase.php
protected function setUp(): void
{
    parent::setUp();
    
    if (config('app.env') === 'testing') {
        DB::enableQueryLog();
    }
}

protected function tearDown(): void
{
    if (config('app.env') === 'testing') {
        $queries = DB::getQueryLog();
        if (count($queries) > 10) {
            $this->fail('Too many queries: ' . count($queries));
        }
    }
    
    parent::tearDown();
}
```

---

## Security ও Best Practices

### 🔒 **Security Testing**

**Authentication Testing:**
```php
public function test_unauthenticated_user_cannot_access_dashboard()
{
    $response = $this->get('/dashboard');
    $response->assertRedirect('/login');
}

public function test_user_cannot_access_admin_panel()
{
    $user = User::factory()->create();
    
    $response = $this->actingAs($user)->get('/admin');
    $response->assertStatus(403);
}
```

**Authorization Testing:**
```php
public function test_user_can_only_edit_own_posts()
{
    $user1 = User::factory()->create();
    $user2 = User::factory()->create();
    $post = Post::factory()->for($user2)->create();
    
    $response = $this->actingAs($user1)
                    ->put("/posts/{$post->id}", [
                        'title' => 'Updated Title'
                    ]);
    
    $response->assertStatus(403);
}
```

**CSRF Protection:**
```php
public function test_csrf_protection_is_active()
{
    $response = $this->post('/posts', [
        'title' => 'Test Post'
    ]);
    
    $response->assertStatus(419); // CSRF token mismatch
}
```

### 🛡️ **Input Validation Testing**

```php
public function test_validation_rules()
{
    $testCases = [
        ['email' => 'invalid-email', 'expected_error' => 'email'],
        ['email' => '', 'expected_error' => 'email'],
        ['password' => '123', 'expected_error' => 'password'],
        ['name' => str_repeat('a', 256), 'expected_error' => 'name'],
    ];
    
    foreach ($testCases as $case) {
        $response = $this->post('/register', $case);
        $response->assertSessionHasErrors($case['expected_error']);
    }
}
```

### 📝 **Environment-Specific Testing**

```php
// tests/TestCase.php
protected function setUp(): void
{
    parent::setUp();
    
    // Ensure we're in testing environment
    $this->assertEquals('testing', app()->environment());
    
    // Disable external services in testing
    config(['services.stripe.key' => 'test_key']);
    config(['mail.default' => 'array']);
}
```

---

## Troubleshooting Guide

### 🐛 **Common Issues**

**1. Database Connection Issues:**
```bash
# Check database connection
php artisan tinker
>>> DB::connection()->getPdo()

# Reset database
php artisan migrate:fresh --seed
```

**2. Memory Issues:**
```php
// phpunit.xml
<php>
    <ini name="memory_limit" value="512M"/>
</php>
```

**3. Timeout Issues:**
```php
// Increase timeout for slow tests
public function test_slow_operation()
{
    $this->timeout(300); // 5 minutes
    
    // Your slow test code
}
```

**4. File Permission Issues:**
```bash
# Fix permissions
sudo chown -R $USER:www-data storage
sudo chown -R $USER:www-data bootstrap/cache
chmod -R 775 storage
chmod -R 775 bootstrap/cache
```

### 🔍 **Debugging Tests**

**Debug Output:**
```php
public function test_debug_example()
{
    $user = User::factory()->create();
    
    // Debug output
    dump($user->toArray());
    dd($user->getAttributes());
    
    // Response debugging
    $response = $this->get('/api/users');
    dump($response->getContent());
    
    $this->assertTrue(true);
}
```

**Log Debugging:**
```php
public function test_with_logging()
{
    Log::info('Test started');
    
    $response = $this->post('/api/posts', $data);
    
    Log::info('Response status: ' . $response->status());
    Log::info('Response content: ' . $response->getContent());
    
    $response->assertStatus(201);
}
```

### 📊 **Test Coverage Analysis**

```bash
# Generate coverage report
php artisan test --coverage-html coverage-report

# Coverage with minimum threshold
php artisan test --coverage --min=80

# Coverage for specific directory
php artisan test --coverage-filter app/Services
```

**Coverage Configuration:**
```xml
<!-- phpunit.xml -->
<coverage>
    <include>
        <directory suffix=".php">./app</directory>
    </include>
    <exclude>
        <directory>./app/Console</directory>
        <file>./app/Http/Kernel.php</file>
    </exclude>
</coverage>
```

---

## Best Practices এবং পরামর্শ

1.  **`.env` ফাইল ম্যানেজমেন্ট**:
    - `.env.testing` নামে একটি ফাইল রিপোজিটরিতে রেখে CI-তে `.env` হিসেবে কপি করা একটি পরিষ্কার পদ্ধতি।
    - `.env.testing` ফাইলে `DB_DATABASE=testing`, `DB_USERNAME=root`, `DB_PASSWORD=password` এর মতো CI-specific ভ্যারিয়েবলগুলো রাখুন।

2.  **ক্যাশিং (Caching)**:
    - `composer` dependency এবং `npm` dependency ক্যাশ করে রাখলে workflow রান টাইম আরও কমে যাবে। `actions/cache` action-টি এর জন্য ব্যবহার করতে পারেন।

3.  **প্যারালাল টেস্টিং (Parallel Testing)**:
    - বড় প্রজেক্টের ক্ষেত্রে, টেস্টগুলোকে একাধিক job-এ ভাগ করে প্যারালালি চালালে মোট সময় কমে আসে। Laravel 9+ এ প্যারালাল টেস্টিং এর বিল্ট-ইন সাপোর্ট আছে।

4.  **কোড কভারেজ (Code Coverage)**:
    - টেস্ট চালানোর সময় `--coverage` ফ্ল্যাগ ব্যবহার করে কোড কভারেজ রিপোর্ট তৈরি করতে পারেন এবং Codecov বা Coveralls এর মতো সার্ভিসে আপলোড করে কভারেজ ট্র্যাক করতে পারেন।

5.  **সম্পূর্ণ টেস্ট স্যুট চালানো**:
    - শুধুমাত্র পরিবর্তিত ফাইল টেস্ট করা সময় বাঁচায়, কিন্তু মাঝে মাঝে (যেমন, `main` ব্রাঞ্চে merge করার আগে) সম্পূর্ণ টেস্ট স্যুট (`php artisan test`) চালানো উচিত যাতে কোনো Regression না হয়।

---

## 🎯 Quick Reference

### Daily Commands:
```bash
# Run all tests
php artisan test

# Run with coverage
php artisan test --coverage

# Run specific test
php artisan test --filter UserTest

# Run parallel tests
php artisan test --parallel
```

### GitHub Actions Checklist:
- ✅ Multiple PHP versions testing
- ✅ Database services (MySQL, Redis)
- ✅ Dependency caching
- ✅ Code quality checks
- ✅ Security auditing
- ✅ Coverage reporting
- ✅ Deployment automation

### Best Practices Summary:
1. **Write tests first** (TDD approach)
2. **Use factories** for test data
3. **Mock external services**
4. **Test edge cases** and error conditions
5. **Keep tests fast** and independent
6. **Use descriptive test names**
7. **Maintain high coverage** (80%+)
8. **Regular security audits**

এই সম্পূর্ণ গাইড Laravel testing এবং GitHub Actions এর সব advanced features cover করেছে। এটি follow করে আপনি enterprise-level testing এবং CI/CD pipeline তৈরি করতে পারবেন।