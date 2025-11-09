# GitHub Actions Workflow - Laravel PHP বিস্তারিত বাংলা গাইড

## 📋 সূচিপত্র
- [Workflow File Structure](#workflow-file-structure)
- [Triggers (on) বিস্তারিত](#triggers-on-বিস্তারিত)
- [Jobs Configuration](#jobs-configuration)
- [Services Setup](#services-setup)
- [Steps বিস্তারিত ব্যাখ্যা](#steps-বিস্তারিত-ব্যাখ্যা)
- [Secrets ব্যবহার](#secrets-ব্যবহার)
- [Changed Files Detection](#changed-files-detection)
- [Error Handling ও Optimization](#error-handling-ও-optimization)

---

## Workflow File Structure

### 📁 **File Location**
```
.github/workflows/test.yml
```

**কেন এই location:**
- GitHub automatically detect করে `.github/workflows/` folder
- `.yml` বা `.yaml` extension হতে হবে
- Multiple workflow files রাখা যায়

### 🏷️ **Workflow Name**
```yaml
name: Run Tests
```

**Purpose:**
- GitHub Actions tab এ এই নাম দেখাবে
- Pull request status check এ display হবে
- Descriptive name দেওয়া best practice

---

## Triggers (on) বিস্তারিত

### 🎯 **Event-based Triggers**

```yaml
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
```

### 📊 **Pull Request Trigger বিশ্লেষণ**

**কখন চলবে:**
- `main` বা `test` branch এ pull request তৈরি হলে
- Pull request update হলে
- Pull request reopen হলে

**Branches Array:**
```yaml
branches: [ main, test ]
# এর মানে:
# - main branch এ PR হলে চলবে
# - test branch এ PR হলে চলবে
# - অন্য branch এ PR হলে চলবে না
```

**Paths Filter:**
```yaml
paths:
  - 'tests/**'      # tests folder এর যেকোনো file
  - 'app/**'        # app folder এর যেকোনো file  
  - 'composer.json' # root এর composer.json
  - 'composer.lock' # root এর composer.lock
```

**কেন এই paths:**
- `tests/**`: Test files change হলে test চালানো দরকার
- `app/**`: Application code change হলে test চালানো দরকার
- `composer.json`: Dependencies change হলে test environment আলাদা হতে পারে
- `composer.lock`: Exact dependency versions change হলে test করা দরকার

**যে files change হলে workflow চলবে না:**
- `README.md`
- `docs/**`
- `.gitignore`
- `public/css/**`
- `public/js/**`

### 🚀 **Push Trigger বিশ্লেষণ**

**কখন চলবে:**
- `main` বা `test` branch এ direct push হলে
- Same path filters apply হবে

**Use Cases:**
- Hotfix push করার পর
- Direct commit করার পর
- Merge করার পর

---

## Jobs Configuration

### 🖥️ **Job Definition**

```yaml
jobs:
  test:                    # Job name
    runs-on: ubuntu-latest # Runner environment
```

**Job Name: `test`**
- এই নামে GitHub Actions tab এ দেখাবে
- অন্য jobs থেকে reference করা যায়
- Descriptive name ব্যবহার করুন

**Runner: `ubuntu-latest`**
- GitHub-hosted runner
- Ubuntu 22.04 LTS (current latest)
- Pre-installed software: PHP, Node.js, Docker, etc.

**Alternative Runners:**
```yaml
runs-on: ubuntu-20.04    # Specific Ubuntu version
runs-on: windows-latest  # Windows runner
runs-on: macos-latest    # macOS runner
runs-on: self-hosted     # Your own runner
```

---

## Services Setup

### 🗄️ **MySQL Service**

```yaml
services:
  mysql:
    image: mysql:8.0
    env:
      MYSQL_ROOT_PASSWORD: password
      MYSQL_DATABASE: testing
    ports:
      - 3306:3306
    options: --health-cmd="mysqladmin ping" --health-interval=10s --health-timeout=5s --health-retries=3
```

### 📊 **MySQL Configuration বিশ্লেষণ**

**Image: `mysql:8.0`**
- Docker Hub থেকে MySQL 8.0 image pull করবে
- Latest stable version
- Production এর সাথে compatible

**Environment Variables:**
```yaml
env:
  MYSQL_ROOT_PASSWORD: password  # Root user password
  MYSQL_DATABASE: testing        # Default database তৈরি হবে
```

**Port Mapping:**
```yaml
ports:
  - 3306:3306
# Container port 3306 → Host port 3306
# Laravel থেকে 127.0.0.1:3306 দিয়ে connect করা যাবে
```

**Health Check Options:**
```yaml
options: --health-cmd="mysqladmin ping" --health-interval=10s --health-timeout=5s --health-retries=3
```

**Health Check বিস্তারিত:**
- `--health-cmd="mysqladmin ping"`: MySQL ready কিনা check করে
- `--health-interval=10s`: প্রতি 10 সেকেন্ডে check করবে
- `--health-timeout=5s`: 5 সেকেন্ড wait করবে response এর জন্য
- `--health-retries=3`: 3 বার fail হলে unhealthy mark করবে

**কেন Health Check দরকার:**
- MySQL fully ready হওয়ার আগে migration চালালে error হবে
- Service dependency ensure করে
- Flaky test prevent করে

---

## Steps বিস্তারিত ব্যাখ্যা

### 1️⃣ **Checkout Code**

```yaml
- uses: actions/checkout@v3
```

**কি করে:**
- Repository এর code runner এ download করে
- Current commit/branch checkout করে
- Working directory তৈরি করে

**Version `@v3`:**
- Stable version
- Security updates পায়
- `@v4` latest available

### 2️⃣ **Setup PHP**

```yaml
- name: Setup PHP
  uses: shivammathur/setup-php@v2
  with:
    php-version: '8.1'
    extensions: mbstring, dom, fileinfo, mysql, gd
    coverage: xdebug
```

**PHP Version: `8.1`**
- Laravel 9/10 compatible
- Production environment match করুন
- String format এ দিতে হবে

**Extensions বিশ্লেষণ:**
```yaml
extensions: mbstring, dom, fileinfo, mysql, gd
```

- `mbstring`: Multi-byte string functions (Laravel core requirement)
- `dom`: XML/HTML parsing (Laravel, testing)
- `fileinfo`: File type detection (file uploads)
- `mysql`: MySQL database connection
- `gd`: Image processing (image manipulation)

**Coverage: `xdebug`**
- Code coverage report তৈরির জন্য
- Performance impact আছে
- শুধু testing environment এ enable করুন

**অন্যান্য useful extensions:**
```yaml
extensions: mbstring, dom, fileinfo, mysql, gd, redis, zip, curl, json
```

### 3️⃣ **Install Dependencies**

```yaml
- name: Install Dependencies
  run: composer install -q --no-ansi --no-interaction --no-scripts --no-progress --prefer-dist
```

**Composer Flags বিশ্লেষণ:**

- `-q` (quiet): Minimal output, faster execution
- `--no-ansi`: ANSI colors disable (CI environment এ cleaner)
- `--no-interaction`: Interactive prompts disable
- `--no-scripts`: Post-install scripts skip (security, speed)
- `--no-progress`: Progress bar disable (cleaner logs)
- `--prefer-dist`: Distribution packages prefer (faster than source)

**Alternative Commands:**
```bash
# Development dependencies সহ
composer install --prefer-dist --no-progress

# Production optimized
composer install --no-dev --optimize-autoloader --prefer-dist

# With caching
composer install --prefer-dist --no-progress --no-suggest
```

### 4️⃣ **Set Environment Variables**

```yaml
- name: Set Environment Variables
  run: |
    echo "APP_KEY=${{ secrets.APP_KEY }}" >> $GITHUB_ENV
    echo "AWS_ACCESS_KEY_ID=${{ secrets.AWS_ACCESS_KEY_ID }}" >> $GITHUB_ENV
    echo "AWS_SECRET_ACCESS_KEY=${{ secrets.AWS_SECRET_ACCESS_KEY }}" >> $GITHUB_ENV
    echo "AWS_DEFAULT_REGION=ap-northeast-1" >> $GITHUB_ENV
    echo "AWS_COGNITO_ACCESS_KEY_ID=${{ secrets.AWS_COGNITO_ACCESS_KEY_ID }}" >> $GITHUB_ENV
    echo "AWS_COGNITO_SECRET_ACCESS_KEY=${{ secrets.AWS_COGNITO_SECRET_ACCESS_KEY }}" >> $GITHUB_ENV
    echo "AWS_COGNITO_REGION=ap-northeast-1" >> $GITHUB_ENV
    echo "AWS_COGNITO_APP_CLIENT_ID=${{ secrets.AWS_COGNITO_APP_CLIENT_ID }}" >> $GITHUB_ENV
    echo "AWS_COGNITO_APP_CLIENT_SECRET=${{ secrets.AWS_COGNITO_APP_CLIENT_SECRET }}" >> $GITHUB_ENV
    echo "AWS_COGNITO_USER_POOL_ID=${{ secrets.AWS_COGNITO_USER_POOL_ID }}" >> $GITHUB_ENV
    echo "TWILIO_SID=${{ secrets.TWILIO_SID }}" >> $GITHUB_ENV
    echo "TWILIO_TOKEN=${{ secrets.TWILIO_TOKEN }}" >> $GITHUB_ENV
    echo "MUX_TOKEN_SECRET=${{ secrets.MUX_TOKEN_SECRET }}" >> $GITHUB_ENV
    echo "SLACK_WEBHOOK_URL=${{ secrets.SLACK_WEBHOOK_URL }}" >> $GITHUB_ENV
```

### 🔐 **Environment Variables বিশ্লেষণ**

**`$GITHUB_ENV` কি:**
- GitHub Actions এর special environment file
- এখানে set করা variables পরবর্তী steps এ available হয়
- Persistent across steps

**Secrets Categories:**

**1. Laravel Core:**
```bash
APP_KEY=${{ secrets.APP_KEY }}
# Laravel application encryption key
# php artisan key:generate দিয়ে তৈরি করা
```

**2. AWS Services:**
```bash
AWS_ACCESS_KEY_ID=${{ secrets.AWS_ACCESS_KEY_ID }}
AWS_SECRET_ACCESS_KEY=${{ secrets.AWS_SECRET_ACCESS_KEY }}
AWS_DEFAULT_REGION=ap-northeast-1
```
- S3, SES, Lambda access এর জন্য
- Region: Tokyo (ap-northeast-1)

**3. AWS Cognito (Authentication):**
```bash
AWS_COGNITO_ACCESS_KEY_ID=${{ secrets.AWS_COGNITO_ACCESS_KEY_ID }}
AWS_COGNITO_SECRET_ACCESS_KEY=${{ secrets.AWS_COGNITO_SECRET_ACCESS_KEY }}
AWS_COGNITO_REGION=ap-northeast-1
AWS_COGNITO_APP_CLIENT_ID=${{ secrets.AWS_COGNITO_APP_CLIENT_ID }}
AWS_COGNITO_APP_CLIENT_SECRET=${{ secrets.AWS_COGNITO_APP_CLIENT_SECRET }}
AWS_COGNITO_USER_POOL_ID=${{ secrets.AWS_COGNITO_USER_POOL_ID }}
```
- User authentication service
- Separate credentials for security

**4. Third-party Services:**
```bash
TWILIO_SID=${{ secrets.TWILIO_SID }}           # SMS service
TWILIO_TOKEN=${{ secrets.TWILIO_TOKEN }}       # SMS authentication
MUX_TOKEN_SECRET=${{ secrets.MUX_TOKEN_SECRET }} # Video streaming
SLACK_WEBHOOK_URL=${{ secrets.SLACK_WEBHOOK_URL }} # Notifications
```

**Alternative Method (.env file):**
```yaml
- name: Create .env file
  run: |
    cp .env.example .env
    echo "APP_KEY=${{ secrets.APP_KEY }}" >> .env
    echo "DB_CONNECTION=mysql" >> .env
    echo "DB_HOST=127.0.0.1" >> .env
    # ... other variables
```

### 5️⃣ **Directory Permissions**

```yaml
- name: Directory Permissions
  run: chmod -R 777 storage bootstrap/cache
```

**কেন দরকার:**
- Laravel এর `storage` folder এ logs, cache, sessions store হয়
- `bootstrap/cache` এ compiled views, config cache থাকে
- Write permission না থাকলে Laravel crash হবে

**Permission `777` বিশ্লেষণ:**
- `7` (owner): read, write, execute
- `7` (group): read, write, execute  
- `7` (others): read, write, execute

**Production এ safer alternative:**
```bash
chmod -R 775 storage bootstrap/cache
chown -R www-data:www-data storage bootstrap/cache
```

**`-R` flag:**
- Recursive: সব subdirectories এবং files এ apply হবে

### 6️⃣ **Run Database Migrations**

```yaml
- name: Run Database Migrations
  env:
    DB_CONNECTION: mysql
    DB_HOST: 127.0.0.1
    DB_PORT: 3306
    DB_DATABASE: testing
    DB_USERNAME: root
    DB_PASSWORD: password
  run: php artisan migrate --force
```

**Environment Variables:**
- `DB_CONNECTION: mysql`: Laravel database driver
- `DB_HOST: 127.0.0.1`: MySQL service host (localhost)
- `DB_PORT: 3306`: MySQL default port
- `DB_DATABASE: testing`: Database name (services এ তৈরি করা)
- `DB_USERNAME: root`: MySQL root user
- `DB_PASSWORD: password`: MySQL root password (services এ set করা)

**`--force` Flag:**
- Production environment এ migration confirmation skip করে
- CI environment এ interactive prompt disable করে
- Automated deployment এর জন্য essential

**Alternative Commands:**
```bash
# Fresh migration (drop all tables)
php artisan migrate:fresh --force

# With seeding
php artisan migrate --seed --force

# Rollback and migrate
php artisan migrate:refresh --force
```

### 7️⃣ **Get Changed Test Files**

```yaml
- name: Get changed test files
  id: changed-files
  uses: tj-actions/changed-files@v39
  with:
    files: tests/**/*.php
```

**Action: `tj-actions/changed-files@v39`**
- Third-party action
- Git diff analyze করে changed files detect করে
- Performance optimization এর জন্য

**`id: changed-files`**
- এই step এর output অন্য steps এ ব্যবহার করা যাবে
- `steps.changed-files.outputs.*` দিয়ে access করা যায়

**`files: tests/**/*.php`**
- শুধু `tests` folder এর `.php` files track করবে
- `**` means recursive subdirectories
- `*.php` means all PHP files

**Available Outputs:**
```yaml
${{ steps.changed-files.outputs.any_changed }}        # true/false
${{ steps.changed-files.outputs.all_changed_files }}  # space-separated list
${{ steps.changed-files.outputs.added_files }}        # newly added files
${{ steps.changed-files.outputs.modified_files }}     # modified files
${{ steps.changed-files.outputs.deleted_files }}      # deleted files
```

### 8️⃣ **Run Tests**

```yaml
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
      for file in ${{ steps.changed-files.outputs.all_changed_files }}; do
        php artisan test "$file"
      done
    else
      echo "No test files changed, skipping tests"
    fi
```

### 🧪 **Test Execution Logic**

**Conditional Execution:**
```bash
if [ "${{ steps.changed-files.outputs.any_changed }}" == "true" ]; then
```
- Changed files আছে কিনা check করে
- Performance optimization: unnecessary tests skip করে

**Loop Through Changed Files:**
```bash
for file in ${{ steps.changed-files.outputs.all_changed_files }}; do
  php artisan test "$file"
done
```
- প্রতিটি changed test file আলাদাভাবে run করে
- Individual file testing
- Specific error reporting

**Alternative: Run All Tests:**
```bash
# সব tests একসাথে চালানো
php artisan test

# Parallel testing
php artisan test --parallel

# With coverage
php artisan test --coverage

# Specific test suite
php artisan test --testsuite=Feature
```

---

## Secrets ব্যবহার

### 🔐 **Secrets Setup Process**

**Step 1: GitHub Repository Settings**
```
Repository → Settings → Security → Secrets and variables → Actions
```

**Step 2: Add Repository Secrets**
```
Secret Name: APP_KEY
Secret Value: base64:your-generated-app-key

Secret Name: AWS_ACCESS_KEY_ID  
Secret Value: AKIAIOSFODNN7EXAMPLE

Secret Name: TWILIO_SID
Secret Value: ACxxxxxxxxxxxxxxxxxxxxxxxxxxxxx
```

### 📊 **Required Secrets List**

**Laravel Core Secrets:**
```
APP_KEY=base64:generated-key-here
```

**AWS Secrets:**
```
AWS_ACCESS_KEY_ID=AKIA...
AWS_SECRET_ACCESS_KEY=wJal...
AWS_COGNITO_ACCESS_KEY_ID=AKIA...
AWS_COGNITO_SECRET_ACCESS_KEY=wJal...
AWS_COGNITO_APP_CLIENT_ID=1234567890abcdef
AWS_COGNITO_APP_CLIENT_SECRET=secret-here
AWS_COGNITO_USER_POOL_ID=ap-northeast-1_aBcDeFgHi
```

**Third-party Secrets:**
```
TWILIO_SID=ACxxxxxxxxxxxxxxxxxxxxxxxxxxxxx
TWILIO_TOKEN=your-auth-token-here
MUX_TOKEN_SECRET=your-mux-secret
SLACK_WEBHOOK_URL=https://hooks.slack.com/services/...
```

### 🛡️ **Secrets Security Best Practices**

**1. Naming Convention:**
```
✅ Good: AWS_ACCESS_KEY_ID, DB_PASSWORD, API_KEY_STRIPE
❌ Bad: key, password, secret, token
```

**2. Environment Separation:**
```
Production: AWS_ACCESS_KEY_ID_PROD
Staging: AWS_ACCESS_KEY_ID_STAGING  
Testing: AWS_ACCESS_KEY_ID_TEST
```

**3. Regular Rotation:**
```
- API Keys: Every 6 months
- Database passwords: Every 3 months
- SSH keys: Every year
```

---

## Changed Files Detection

### 🔍 **Why Changed Files Detection?**

**Performance Benefits:**
- শুধু relevant tests চালায়
- CI time 50-80% কমে যায়
- Resource usage কম হয়
- Faster feedback

**Example Scenario:**
```
Changed files:
- tests/Feature/UserTest.php
- tests/Unit/HelperTest.php

Only these 2 files will be tested instead of all 50+ test files
```

### ⚙️ **Advanced Changed Files Configuration**

**Multiple File Patterns:**
```yaml
- name: Get changed files
  id: changed-files
  uses: tj-actions/changed-files@v39
  with:
    files: |
      tests/**/*.php
      app/**/*.php
    files_ignore: |
      tests/Browser/**
      app/Console/Commands/**
```

**Separate Detection for Different Types:**
```yaml
- name: Get changed test files
  id: changed-tests
  uses: tj-actions/changed-files@v39
  with:
    files: tests/**/*.php

- name: Get changed app files  
  id: changed-app
  uses: tj-actions/changed-files@v39
  with:
    files: app/**/*.php
```

**Conditional Test Execution:**
```yaml
- name: Run Feature Tests
  if: steps.changed-tests.outputs.any_changed == 'true'
  run: php artisan test --testsuite=Feature

- name: Run Unit Tests
  if: steps.changed-app.outputs.any_changed == 'true'  
  run: php artisan test --testsuite=Unit
```

---

## Error Handling ও Optimization

### 🚨 **Common Issues ও Solutions**

**1. MySQL Connection Failed:**
```yaml
# Problem: MySQL service not ready
# Solution: Add health check wait
- name: Wait for MySQL
  run: |
    while ! mysqladmin ping -h 127.0.0.1 -u root -ppassword --silent; do
      echo "Waiting for MySQL..."
      sleep 2
    done
```

**2. Permission Denied Errors:**
```yaml
# Problem: Storage permission issues
# Solution: Fix permissions before migration
- name: Fix Permissions
  run: |
    sudo chown -R $USER:$USER storage bootstrap/cache
    chmod -R 775 storage bootstrap/cache
```

**3. Memory Issues:**
```yaml
# Problem: Composer memory limit
# Solution: Increase PHP memory
- name: Install Dependencies
  run: |
    php -d memory_limit=512M $(which composer) install --prefer-dist --no-progress
```

**4. Test Database Issues:**
```yaml
# Problem: Database not clean between tests
# Solution: Fresh migration
- name: Fresh Migration
  run: php artisan migrate:fresh --force
```

### ⚡ **Performance Optimizations**

**1. Dependency Caching:**
```yaml
- name: Cache Composer dependencies
  uses: actions/cache@v3
  with:
    path: ~/.composer/cache
    key: ${{ runner.os }}-composer-${{ hashFiles('**/composer.lock') }}
    restore-keys: ${{ runner.os }}-composer-

- name: Install Dependencies
  run: composer install --prefer-dist --no-progress
```

**2. Parallel Testing:**
```yaml
- name: Run Tests in Parallel
  run: php artisan test --parallel --processes=4
```

**3. Selective Testing:**
```yaml
- name: Run Only Necessary Tests
  run: |
    if [[ "${{ steps.changed-files.outputs.all_changed_files }}" == *"app/Models"* ]]; then
      php artisan test --testsuite=Unit
    elif [[ "${{ steps.changed-files.outputs.all_changed_files }}" == *"app/Http"* ]]; then
      php artisan test --testsuite=Feature  
    else
      php artisan test --filter="${{ steps.changed-files.outputs.all_changed_files }}"
    fi
```

**4. Database Optimization:**
```yaml
# Use SQLite for faster tests
services:
  # Remove MySQL service
  
steps:
- name: Setup SQLite
  run: |
    echo "DB_CONNECTION=sqlite" >> $GITHUB_ENV
    echo "DB_DATABASE=:memory:" >> $GITHUB_ENV
```

### 📊 **Monitoring ও Reporting**

**1. Test Results Reporting:**
```yaml
- name: Publish Test Results
  uses: dorny/test-reporter@v1
  if: success() || failure()
  with:
    name: PHPUnit Tests
    path: tests/results.xml
    reporter: java-junit
```

**2. Coverage Reporting:**
```yaml
- name: Generate Coverage Report
  run: php artisan test --coverage-clover coverage.xml

- name: Upload Coverage
  uses: codecov/codecov-action@v3
  with:
    file: ./coverage.xml
```

**3. Slack Notifications:**
```yaml
- name: Notify Slack
  if: failure()
  uses: 8398a7/action-slack@v3
  with:
    status: failure
    webhook_url: ${{ secrets.SLACK_WEBHOOK_URL }}
```

---

## 🎯 Quick Reference

### Essential Commands:
```bash
# Local testing
php artisan test
php artisan test --filter=UserTest
php artisan test --coverage

# Database commands  
php artisan migrate:fresh --force
php artisan db:seed --force

# Permission fix
chmod -R 775 storage bootstrap/cache
```

### Workflow Debugging:
```yaml
# Debug step
- name: Debug Info
  run: |
    echo "PHP Version: $(php -v)"
    echo "Composer Version: $(composer --version)"
    echo "Laravel Version: $(php artisan --version)"
    echo "Environment: $(php artisan env)"
    ls -la storage/
```

### Common Secrets:
```
APP_KEY=base64:...
DB_PASSWORD=...
AWS_ACCESS_KEY_ID=...
TWILIO_SID=...
```

এই detailed guide follow করে আপনি GitHub Actions workflow file সম্পূর্ণভাবে বুঝতে পারবেন এবং Laravel project এর জন্য customize করতে পারবেন।