# GitHub Actions Workflow Complete Scenarios - সম্পূর্ণ বাংলা গাইড

## 📋 সূচিপত্র
- [GitHub Actions কি এবং কেন ব্যবহার করবেন](#github-actions-কি-এবং-কেন-ব্যবহার-করবেন)
- [Basic Workflow Structure](#basic-workflow-structure)
- [Laravel CI/CD Complete Scenarios](#laravel-cicd-complete-scenarios)
- [Testing Workflows](#testing-workflows)
- [Deployment Workflows](#deployment-workflows)
- [Security & Code Quality](#security--code-quality)
- [Performance & Optimization](#performance--optimization)
- [Advanced Scenarios](#advanced-scenarios)
- [Troubleshooting](#troubleshooting)

---

## GitHub Actions কি এবং কেন ব্যবহার করবেন

### 🤖 **GitHub Actions Overview**

**GitHub Actions** হলো একটি CI/CD (Continuous Integration/Continuous Deployment) platform যা আপনার code repository এর সাথে directly integrated।

**মূল সুবিধাসমূহ:**
- ✅ **Free** - Public repositories এর জন্য unlimited
- ✅ **Integrated** - GitHub এর সাথে built-in
- ✅ **Flexible** - Custom workflows তৈরি করা যায়
- ✅ **Scalable** - Multiple jobs parallel এ run করা যায়
- ✅ **Secure** - Secrets management built-in

### 🎯 **Laravel Projects এর জন্য Use Cases**

```
1. 🧪 Automated Testing
   - Unit tests, Feature tests
   - Browser tests (Dusk)
   - API testing

2. 🚀 Automated Deployment  
   - Staging deployment
   - Production deployment
   - Multi-server deployment

3. 🔍 Code Quality Checks
   - PHP CS Fixer
   - PHPStan analysis
   - Security scanning

4. 📦 Package Management
   - Composer dependencies
   - NPM packages
   - Asset compilation

5. 🔔 Notifications
   - Slack notifications
   - Email alerts
   - Discord webhooks
```

---

## Basic Workflow Structure

### 📁 **Workflow File Structure**

```yaml
# .github/workflows/ci.yml
name: CI/CD Pipeline                    # Workflow name
on:                                     # Trigger events
  push:
    branches: [main, dev]
  pull_request:
    branches: [main]

jobs:                                   # Jobs definition
  test:                                 # Job name
    runs-on: ubuntu-latest              # Runner OS
    steps:                              # Steps to execute
    - name: Checkout code               # Step name
      uses: actions/checkout@v4         # Action to use
    
    - name: Run tests
      run: php artisan test             # Command to run
```

### 🔧 **Key Components বিস্তারিত**

**1. Triggers (on):**
```yaml
on:
  push:                    # Code push হলে
    branches: [main, dev]  # Specific branches
  pull_request:            # PR create হলে
    branches: [main]
  schedule:                # Scheduled runs
    - cron: '0 2 * * *'    # Daily at 2 AM
  workflow_dispatch:       # Manual trigger
```

**2. Jobs:**
```yaml
jobs:
  test:
    runs-on: ubuntu-latest
    strategy:
      matrix:              # Multiple versions test
        php: [8.1, 8.2, 8.3]
    steps:
      # Steps here
```

**3. Steps:**
```yaml
steps:
- name: Setup PHP
  uses: shivammathur/setup-php@v2
  with:
    php-version: ${{ matrix.php }}
    
- name: Install dependencies
  run: composer install
  
- name: Run tests
  run: php artisan test
```

---

## Laravel CI/CD Complete Scenarios

### 🏗️ **Scenario 1: Complete Laravel CI/CD Pipeline**

```yaml
name: Laravel CI/CD Pipeline

on:
  push:
    branches: [main, dev]
  pull_request:
    branches: [main]

env:
  PHP_VERSION: '8.2'
  NODE_VERSION: '18'

jobs:
  # ==================== TESTING PHASE ====================
  test:
    name: Run Tests
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
        image: redis:7.0
        ports:
          - 6379:6379
        options: --health-cmd="redis-cli ping" --health-interval=10s --health-timeout=5s --health-retries=3

    steps:
    - name: 📥 Checkout Repository
      uses: actions/checkout@v4

    - name: 🐘 Setup PHP
      uses: shivammathur/setup-php@v2
      with:
        php-version: ${{ env.PHP_VERSION }}
        extensions: mbstring, dom, fileinfo, mysql, gd, redis, zip
        coverage: xdebug

    - name: 📦 Cache Composer Dependencies
      uses: actions/cache@v3
      with:
        path: vendor
        key: composer-${{ hashFiles('composer.lock') }}
        restore-keys: composer-

    - name: 🎼 Install Composer Dependencies
      run: composer install --prefer-dist --no-progress --no-suggest

    - name: 📄 Copy Environment File
      run: cp .env.example .env

    - name: 🔑 Generate Application Key
      run: php artisan key:generate

    - name: 🗄️ Run Database Migrations
      run: php artisan migrate --force
      env:
        DB_CONNECTION: mysql
        DB_HOST: 127.0.0.1
        DB_PORT: 3306
        DB_DATABASE: testing
        DB_USERNAME: root
        DB_PASSWORD: password

    - name: 🌱 Seed Database
      run: php artisan db:seed --force

    - name: 🧪 Run PHPUnit Tests
      run: php artisan test --coverage-clover coverage.xml

    - name: 📊 Upload Coverage to Codecov
      uses: codecov/codecov-action@v3
      with:
        file: ./coverage.xml
        flags: unittests
        name: codecov-umbrella

  # ==================== CODE QUALITY PHASE ====================
  code-quality:
    name: Code Quality Checks
    runs-on: ubuntu-latest
    
    steps:
    - name: 📥 Checkout Repository
      uses: actions/checkout@v4

    - name: 🐘 Setup PHP
      uses: shivammathur/setup-php@v2
      with:
        php-version: ${{ env.PHP_VERSION }}
        tools: cs2pr

    - name: 📦 Cache Composer Dependencies
      uses: actions/cache@v3
      with:
        path: vendor
        key: composer-${{ hashFiles('composer.lock') }}

    - name: 🎼 Install Dependencies
      run: composer install --prefer-dist --no-progress

    - name: 🎨 Run PHP CS Fixer
      run: vendor/bin/php-cs-fixer fix --dry-run --format=checkstyle | cs2pr

    - name: 🔍 Run PHPStan Analysis
      run: vendor/bin/phpstan analyse --error-format=checkstyle | cs2pr

    - name: 🛡️ Run Security Check
      run: composer audit

  # ==================== FRONTEND BUILD PHASE ====================
  frontend:
    name: Frontend Build
    runs-on: ubuntu-latest
    
    steps:
    - name: 📥 Checkout Repository
      uses: actions/checkout@v4

    - name: 🟢 Setup Node.js
      uses: actions/setup-node@v4
      with:
        node-version: ${{ env.NODE_VERSION }}
        cache: 'npm'

    - name: 📦 Install NPM Dependencies
      run: npm ci

    - name: 🏗️ Build Assets
      run: npm run build

    - name: 📤 Upload Build Artifacts
      uses: actions/upload-artifact@v3
      with:
        name: frontend-build
        path: public/build/

  # ==================== STAGING DEPLOYMENT ====================
  deploy-staging:
    name: Deploy to Staging
    runs-on: ubuntu-latest
    needs: [test, code-quality, frontend]
    environment: staging
    if: github.ref == 'refs/heads/dev' && github.event_name == 'push'
    
    steps:
    - name: 📥 Checkout Repository
      uses: actions/checkout@v4

    - name: 📤 Download Build Artifacts
      uses: actions/download-artifact@v3
      with:
        name: frontend-build
        path: public/build/

    - name: 🚀 Deploy to Staging Server
      uses: appleboy/ssh-action@v1.0.0
      with:
        host: ${{ secrets.STAGING_HOST }}
        username: ${{ secrets.STAGING_USER }}
        key: ${{ secrets.STAGING_SSH_KEY }}
        script: |
          cd /var/www/staging
          git pull origin dev
          composer install --no-dev --optimize-autoloader
          php artisan migrate --force
          php artisan config:cache
          php artisan route:cache
          php artisan view:cache
          php artisan queue:restart
          sudo systemctl reload nginx

    - name: 🔔 Notify Slack
      uses: 8398a7/action-slack@v3
      with:
        status: ${{ job.status }}
        channel: '#deployments'
        webhook_url: ${{ secrets.SLACK_WEBHOOK }}
      if: always()

  # ==================== PRODUCTION DEPLOYMENT ====================
  deploy-production:
    name: Deploy to Production
    runs-on: ubuntu-latest
    needs: [test, code-quality, frontend]
    environment: production
    if: github.ref == 'refs/heads/main' && github.event_name == 'push'
    
    steps:
    - name: 📥 Checkout Repository
      uses: actions/checkout@v4

    - name: 📤 Download Build Artifacts
      uses: actions/download-artifact@v3
      with:
        name: frontend-build
        path: public/build/

    - name: 🚀 Deploy to Production Server
      uses: appleboy/ssh-action@v1.0.0
      with:
        host: ${{ secrets.PRODUCTION_HOST }}
        username: ${{ secrets.PRODUCTION_USER }}
        key: ${{ secrets.PRODUCTION_SSH_KEY }}
        script: |
          cd /var/www/production
          php artisan down --message="Deployment in progress"
          git pull origin main
          composer install --no-dev --optimize-autoloader
          php artisan migrate --force
          php artisan config:cache
          php artisan route:cache
          php artisan view:cache
          php artisan queue:restart
          php artisan up
          sudo systemctl reload nginx

    - name: 🔔 Notify Team
      uses: 8398a7/action-slack@v3
      with:
        status: ${{ job.status }}
        channel: '#general'
        webhook_url: ${{ secrets.SLACK_WEBHOOK }}
        text: |
          🚀 Production deployment completed!
          Commit: ${{ github.sha }}
          Author: ${{ github.actor }}
      if: success()
```

### 🧪 **Scenario 2: Advanced Testing Workflow**

```yaml
name: Advanced Testing Suite

on:
  push:
    branches: [main, dev]
  pull_request:
    branches: [main]

jobs:
  # ==================== MATRIX TESTING ====================
  test-matrix:
    name: Test PHP ${{ matrix.php }} - Laravel ${{ matrix.laravel }}
    runs-on: ubuntu-latest
    
    strategy:
      fail-fast: false
      matrix:
        php: [8.1, 8.2, 8.3]
        laravel: [10.x, 11.x]
        include:
          - laravel: 10.x
            testbench: 8.x
          - laravel: 11.x
            testbench: 9.x

    services:
      mysql:
        image: mysql:8.0
        env:
          MYSQL_ROOT_PASSWORD: password
          MYSQL_DATABASE: testing
        ports:
          - 3306:3306

    steps:
    - name: 📥 Checkout
      uses: actions/checkout@v4

    - name: 🐘 Setup PHP ${{ matrix.php }}
      uses: shivammathur/setup-php@v2
      with:
        php-version: ${{ matrix.php }}
        extensions: dom, curl, libxml, mbstring, zip, pcntl, pdo, sqlite, pdo_sqlite, bcmath, soap, intl, gd, exif, iconv
        coverage: none

    - name: 🎼 Install Dependencies
      run: |
        composer require "laravel/framework:${{ matrix.laravel }}" "orchestra/testbench:${{ matrix.testbench }}" --no-interaction --no-update
        composer update --prefer-dist --no-interaction

    - name: 📄 Create .env
      run: |
        cp .env.example .env
        php artisan key:generate

    - name: 🗄️ Prepare Database
      run: |
        php artisan migrate --force
        php artisan db:seed --force

    - name: 🧪 Run Tests
      run: php artisan test

  # ==================== BROWSER TESTING ====================
  browser-tests:
    name: Browser Tests (Dusk)
    runs-on: ubuntu-latest
    
    services:
      mysql:
        image: mysql:8.0
        env:
          MYSQL_ROOT_PASSWORD: password
          MYSQL_DATABASE: testing
        ports:
          - 3306:3306

    steps:
    - name: 📥 Checkout
      uses: actions/checkout@v4

    - name: 🐘 Setup PHP
      uses: shivammathur/setup-php@v2
      with:
        php-version: '8.2'
        extensions: dom, curl, libxml, mbstring, zip, pcntl, pdo, sqlite, pdo_sqlite, bcmath, soap, intl, gd, exif, iconv

    - name: 🎼 Install Dependencies
      run: composer install --prefer-dist --no-interaction

    - name: 🟢 Setup Node.js
      uses: actions/setup-node@v4
      with:
        node-version: '18'

    - name: 📦 Install NPM Dependencies
      run: npm ci

    - name: 🏗️ Build Assets
      run: npm run build

    - name: 📄 Prepare Environment
      run: |
        cp .env.dusk.example .env
        php artisan key:generate
        php artisan migrate --force
        php artisan db:seed --force

    - name: 🌐 Start Chrome Driver
      run: ./vendor/laravel/dusk/bin/chromedriver-linux &

    - name: 🚀 Start Laravel Server
      run: php artisan serve --host=127.0.0.1 --port=8000 &

    - name: 🧪 Run Dusk Tests
      run: php artisan dusk

    - name: 📤 Upload Screenshots
      uses: actions/upload-artifact@v3
      if: failure()
      with:
        name: dusk-screenshots
        path: tests/Browser/screenshots

  # ==================== API TESTING ====================
  api-tests:
    name: API Testing
    runs-on: ubuntu-latest
    
    services:
      mysql:
        image: mysql:8.0
        env:
          MYSQL_ROOT_PASSWORD: password
          MYSQL_DATABASE: testing
        ports:
          - 3306:3306

    steps:
    - name: 📥 Checkout
      uses: actions/checkout@v4

    - name: 🐘 Setup PHP
      uses: shivammathur/setup-php@v2
      with:
        php-version: '8.2'

    - name: 🎼 Install Dependencies
      run: composer install --prefer-dist --no-interaction

    - name: 📄 Prepare Environment
      run: |
        cp .env.example .env
        php artisan key:generate
        php artisan migrate --force
        php artisan passport:keys --force

    - name: 🚀 Start Server
      run: php artisan serve --host=127.0.0.1 --port=8000 &

    - name: ⏳ Wait for Server
      run: sleep 5

    - name: 🧪 Run API Tests
      run: php artisan test --testsuite=Feature --filter=Api

    - name: 📊 Generate API Documentation
      run: php artisan scribe:generate

    - name: 📤 Upload API Docs
      uses: actions/upload-artifact@v3
      with:
        name: api-documentation
        path: public/docs/
```

### 🚀 **Scenario 3: Multi-Environment Deployment**

```yaml
name: Multi-Environment Deployment

on:
  push:
    branches: [main, dev, staging]
  workflow_dispatch:
    inputs:
      environment:
        description: 'Environment to deploy'
        required: true
        default: 'staging'
        type: choice
        options:
        - staging
        - production

jobs:
  # ==================== BUILD PHASE ====================
  build:
    name: Build Application
    runs-on: ubuntu-latest
    
    outputs:
      version: ${{ steps.version.outputs.version }}
    
    steps:
    - name: 📥 Checkout
      uses: actions/checkout@v4

    - name: 🐘 Setup PHP
      uses: shivammathur/setup-php@v2
      with:
        php-version: '8.2'

    - name: 🎼 Install Dependencies
      run: composer install --no-dev --optimize-autoloader

    - name: 🟢 Setup Node.js
      uses: actions/setup-node@v4
      with:
        node-version: '18'

    - name: 📦 Install NPM Dependencies
      run: npm ci

    - name: 🏗️ Build Assets
      run: npm run build

    - name: 🏷️ Generate Version
      id: version
      run: echo "version=$(date +'%Y%m%d%H%M%S')-${GITHUB_SHA::7}" >> $GITHUB_OUTPUT

    - name: 📦 Create Deployment Package
      run: |
        tar -czf deployment-${{ steps.version.outputs.version }}.tar.gz \
          --exclude='.git' \
          --exclude='node_modules' \
          --exclude='tests' \
          --exclude='.env*' \
          .

    - name: 📤 Upload Deployment Package
      uses: actions/upload-artifact@v3
      with:
        name: deployment-package
        path: deployment-${{ steps.version.outputs.version }}.tar.gz

  # ==================== STAGING DEPLOYMENT ====================
  deploy-staging:
    name: Deploy to Staging
    runs-on: ubuntu-latest
    needs: build
    environment: staging
    if: |
      (github.ref == 'refs/heads/dev' && github.event_name == 'push') ||
      (github.event_name == 'workflow_dispatch' && github.event.inputs.environment == 'staging')
    
    steps:
    - name: 📤 Download Package
      uses: actions/download-artifact@v3
      with:
        name: deployment-package

    - name: 🚀 Deploy to Staging
      uses: appleboy/ssh-action@v1.0.0
      with:
        host: ${{ secrets.STAGING_HOST }}
        username: ${{ secrets.STAGING_USER }}
        key: ${{ secrets.STAGING_SSH_KEY }}
        script: |
          # Create backup
          cd /var/www
          if [ -d "staging" ]; then
            sudo cp -r staging staging-backup-$(date +%Y%m%d%H%M%S)
          fi
          
          # Deploy new version
          sudo mkdir -p staging-new
          cd staging-new
          
          # Extract deployment package (uploaded separately)
          sudo tar -xzf /tmp/deployment-*.tar.gz
          
          # Setup environment
          sudo cp /var/www/staging/.env .env 2>/dev/null || true
          sudo chown -R www-data:www-data .
          sudo chmod -R 755 .
          
          # Run Laravel commands
          php artisan migrate --force
          php artisan config:cache
          php artisan route:cache
          php artisan view:cache
          
          # Switch to new version
          cd /var/www
          sudo rm -rf staging-old
          sudo mv staging staging-old 2>/dev/null || true
          sudo mv staging-new staging
          
          # Restart services
          sudo systemctl reload nginx
          sudo supervisorctl restart laravel-worker:*

    - name: 🧪 Run Smoke Tests
      run: |
        sleep 30
        curl -f ${{ secrets.STAGING_URL }}/health || exit 1

  # ==================== PRODUCTION DEPLOYMENT ====================
  deploy-production:
    name: Deploy to Production
    runs-on: ubuntu-latest
    needs: build
    environment: production
    if: |
      (github.ref == 'refs/heads/main' && github.event_name == 'push') ||
      (github.event_name == 'workflow_dispatch' && github.event.inputs.environment == 'production')
    
    steps:
    - name: 📤 Download Package
      uses: actions/download-artifact@v3
      with:
        name: deployment-package

    - name: 🚀 Deploy to Production (Blue-Green)
      uses: appleboy/ssh-action@v1.0.0
      with:
        host: ${{ secrets.PRODUCTION_HOST }}
        username: ${{ secrets.PRODUCTION_USER }}
        key: ${{ secrets.PRODUCTION_SSH_KEY }}
        script: |
          # Blue-Green Deployment
          CURRENT_COLOR=$(readlink /var/www/current | grep -o 'blue\|green')
          NEW_COLOR=$([ "$CURRENT_COLOR" = "blue" ] && echo "green" || echo "blue")
          
          echo "Current: $CURRENT_COLOR, Deploying to: $NEW_COLOR"
          
          # Prepare new environment
          sudo rm -rf /var/www/$NEW_COLOR
          sudo mkdir -p /var/www/$NEW_COLOR
          cd /var/www/$NEW_COLOR
          
          # Extract and setup
          sudo tar -xzf /tmp/deployment-*.tar.gz
          sudo cp /var/www/production/.env .env
          sudo chown -R www-data:www-data .
          
          # Laravel setup
          php artisan migrate --force
          php artisan config:cache
          php artisan route:cache
          php artisan view:cache
          
          # Health check
          php artisan serve --host=127.0.0.1 --port=8001 &
          sleep 10
          curl -f http://127.0.0.1:8001/health || exit 1
          pkill -f "artisan serve"
          
          # Switch traffic
          sudo ln -sfn /var/www/$NEW_COLOR /var/www/current
          sudo systemctl reload nginx
          
          # Cleanup old version after 5 minutes
          (sleep 300 && sudo rm -rf /var/www/$CURRENT_COLOR) &

    - name: 🔔 Notify Success
      uses: 8398a7/action-slack@v3
      with:
        status: success
        channel: '#deployments'
        webhook_url: ${{ secrets.SLACK_WEBHOOK }}
        text: |
          🎉 Production deployment successful!
          Version: ${{ needs.build.outputs.version }}
          Commit: ${{ github.sha }}
          Author: ${{ github.actor }}
```

### 🔒 **Scenario 4: Security & Code Quality Pipeline**

```yaml
name: Security & Code Quality

on:
  push:
    branches: [main, dev]
  pull_request:
    branches: [main]
  schedule:
    - cron: '0 2 * * 1'  # Weekly security scan

jobs:
  # ==================== SECURITY SCANNING ====================
  security-scan:
    name: Security Analysis
    runs-on: ubuntu-latest
    
    steps:
    - name: 📥 Checkout
      uses: actions/checkout@v4

    - name: 🐘 Setup PHP
      uses: shivammathur/setup-php@v2
      with:
        php-version: '8.2'

    - name: 🎼 Install Dependencies
      run: composer install --prefer-dist --no-interaction

    - name: 🛡️ Security Audit
      run: composer audit --format=json > security-audit.json

    - name: 🔍 Psalm Security Analysis
      run: |
        composer require --dev psalm/plugin-laravel vimeo/psalm
        vendor/bin/psalm --init
        vendor/bin/psalm --show-info=false

    - name: 🚨 SAST Scan with Semgrep
      uses: returntocorp/semgrep-action@v1
      with:
        config: >-
          p/security-audit
          p/secrets
          p/php

    - name: 📤 Upload Security Results
      uses: actions/upload-artifact@v3
      with:
        name: security-results
        path: |
          security-audit.json
          semgrep-results.json

  # ==================== CODE QUALITY ====================
  code-quality:
    name: Code Quality Analysis
    runs-on: ubuntu-latest
    
    steps:
    - name: 📥 Checkout
      uses: actions/checkout@v4

    - name: 🐘 Setup PHP
      uses: shivammathur/setup-php@v2
      with:
        php-version: '8.2'
        tools: cs2pr

    - name: 🎼 Install Dependencies
      run: composer install --prefer-dist --no-interaction

    - name: 🎨 PHP CS Fixer
      run: |
        vendor/bin/php-cs-fixer fix --dry-run --format=checkstyle | cs2pr
        vendor/bin/php-cs-fixer fix --dry-run --format=json > php-cs-fixer.json

    - name: 🔍 PHPStan Analysis
      run: |
        vendor/bin/phpstan analyse --error-format=checkstyle | cs2pr
        vendor/bin/phpstan analyse --error-format=json > phpstan.json

    - name: 📊 PHP Metrics
      run: |
        composer require --dev phpmetrics/phpmetrics
        vendor/bin/phpmetrics --report-html=metrics app/

    - name: 🧹 PHP Copy/Paste Detector
      run: |
        composer require --dev sebastian/phpcpd
        vendor/bin/phpcpd app/ --log-pmd=phpcpd.xml

    - name: 📈 Code Coverage
      run: |
        php artisan test --coverage-clover=coverage.xml
        composer require --dev phpunit/phpcov
        vendor/bin/phpcov merge --clover=merged-coverage.xml coverage/

    - name: 📤 Upload Quality Reports
      uses: actions/upload-artifact@v3
      with:
        name: quality-reports
        path: |
          php-cs-fixer.json
          phpstan.json
          metrics/
          phpcpd.xml
          coverage.xml

    - name: 💬 Comment PR
      uses: marocchino/sticky-pull-request-comment@v2
      if: github.event_name == 'pull_request'
      with:
        recreate: true
        message: |
          ## 📊 Code Quality Report
          
          ### 🎨 PHP CS Fixer
          ```
          $(cat php-cs-fixer.json | jq -r '.files[] | select(.diff != null) | .name' | head -10)
          ```
          
          ### 🔍 PHPStan
          ```
          $(cat phpstan.json | jq -r '.totals.errors // 0') errors found
          $(cat phpstan.json | jq -r '.totals.file_errors // 0') files with errors
          ```

  # ==================== DEPENDENCY SCANNING ====================
  dependency-scan:
    name: Dependency Analysis
    runs-on: ubuntu-latest
    
    steps:
    - name: 📥 Checkout
      uses: actions/checkout@v4

    - name: 🔍 Composer Dependency Scan
      run: |
        composer install --no-dev
        composer show --format=json > composer-dependencies.json

    - name: 🟢 NPM Audit
      run: |
        npm ci
        npm audit --json > npm-audit.json || true

    - name: 📊 License Check
      run: |
        composer require --dev dominikb/composer-license-checker
        vendor/bin/composer-license-checker check --format=json > licenses.json

    - name: 🚨 Outdated Packages
      run: |
        composer outdated --format=json > outdated-composer.json || true
        npm outdated --json > outdated-npm.json || true

    - name: 📤 Upload Dependency Reports
      uses: actions/upload-artifact@v3
      with:
        name: dependency-reports
        path: |
          composer-dependencies.json
          npm-audit.json
          licenses.json
          outdated-*.json
```

### ⚡ **Scenario 5: Performance & Optimization**

```yaml
name: Performance & Optimization

on:
  push:
    branches: [main]
  pull_request:
    branches: [main]
  schedule:
    - cron: '0 3 * * *'  # Daily performance check

jobs:
  # ==================== PERFORMANCE TESTING ====================
  performance-test:
    name: Performance Testing
    runs-on: ubuntu-latest
    
    services:
      mysql:
        image: mysql:8.0
        env:
          MYSQL_ROOT_PASSWORD: password
          MYSQL_DATABASE: testing
        ports:
          - 3306:3306
      
      redis:
        image: redis:7.0
        ports:
          - 6379:6379

    steps:
    - name: 📥 Checkout
      uses: actions/checkout@v4

    - name: 🐘 Setup PHP
      uses: shivammathur/setup-php@v2
      with:
        php-version: '8.2'
        extensions: mbstring, dom, fileinfo, mysql, gd, redis, zip
        ini-values: memory_limit=512M

    - name: 🎼 Install Dependencies
      run: composer install --no-dev --optimize-autoloader

    - name: 📄 Setup Environment
      run: |
        cp .env.example .env
        php artisan key:generate
        php artisan migrate --force
        php artisan db:seed --force

    - name: 🚀 Start Application
      run: |
        php artisan config:cache
        php artisan route:cache
        php artisan view:cache
        php artisan serve --host=127.0.0.1 --port=8000 &
        sleep 10

    - name: 🔥 Apache Bench Test
      run: |
        sudo apt-get update
        sudo apt-get install -y apache2-utils
        
        # Test homepage
        ab -n 1000 -c 10 -g homepage.dat http://127.0.0.1:8000/ > ab-homepage.txt
        
        # Test API endpoint
        ab -n 500 -c 5 -g api.dat http://127.0.0.1:8000/api/users > ab-api.txt

    - name: 🎯 Artillery Load Test
      run: |
        npm install -g artillery
        
        cat > artillery-config.yml << EOF
        config:
          target: 'http://127.0.0.1:8000'
          phases:
            - duration: 60
              arrivalRate: 10
        scenarios:
          - name: "Homepage"
            requests:
              - get:
                  url: "/"
          - name: "API Test"
            requests:
              - get:
                  url: "/api/users"
        EOF
        
        artillery run artillery-config.yml --output artillery-report.json
        artillery report artillery-report.json --output artillery-report.html

    - name: 📊 Laravel Telescope Performance
      run: |
        composer require laravel/telescope --dev
        php artisan telescope:install
        php artisan migrate
        
        # Generate some requests for analysis
        for i in {1..50}; do
          curl -s http://127.0.0.1:8000/ > /dev/null
          curl -s http://127.0.0.1:8000/api/users > /dev/null
        done

    - name: 📈 Generate Performance Report
      run: |
        cat > performance-summary.md << EOF
        # Performance Test Results
        
        ## Apache Bench Results
        
        ### Homepage Performance
        \`\`\`
        $(grep -E "Requests per second|Time per request|Transfer rate" ab-homepage.txt)
        \`\`\`
        
        ### API Performance  
        \`\`\`
        $(grep -E "Requests per second|Time per request|Transfer rate" ab-api.txt)
        \`\`\`
        
        ## Load Test Summary
        Artillery test completed. See detailed report in artifacts.
        EOF

    - name: 📤 Upload Performance Reports
      uses: actions/upload-artifact@v3
      with:
        name: performance-reports
        path: |
          ab-*.txt
          *.dat
          artillery-report.*
          performance-summary.md

  # ==================== OPTIMIZATION CHECKS ====================
  optimization-check:
    name: Optimization Analysis
    runs-on: ubuntu-latest
    
    steps:
    - name: 📥 Checkout
      uses: actions/checkout@v4

    - name: 🐘 Setup PHP
      uses: shivammathur/setup-php@v2
      with:
        php-version: '8.2'

    - name: 🎼 Install Dependencies
      run: composer install --prefer-dist --no-interaction

    - name: 🔍 Analyze Route Caching
      run: |
        php artisan route:list --json > routes.json
        echo "Total routes: $(cat routes.json | jq length)"
        
        # Check for closure routes (can't be cached)
        php artisan route:list | grep -c "Closure" || echo "No closure routes found"

    - name: 📊 Database Query Analysis
      run: |
        # Install query analyzer
        composer require barryvdh/laravel-debugbar --dev
        
        # Run tests with query logging
        php artisan test --filter=DatabaseTest > query-analysis.txt 2>&1 || true

    - name: 🗜️ Asset Optimization Check
      run: |
        # Check if assets are minified
        if [ -f "public/css/app.css" ]; then
          echo "CSS size: $(wc -c < public/css/app.css) bytes"
        fi
        
        if [ -f "public/js/app.js" ]; then
          echo "JS size: $(wc -c < public/js/app.js) bytes"
        fi

    - name: 🧹 Code Optimization Suggestions
      run: |
        # Check for N+1 queries
        grep -r "foreach.*->.*->" app/ > potential-n-plus-one.txt || echo "No obvious N+1 patterns found"
        
        # Check for missing indexes
        php artisan migrate:status > migration-status.txt

    - name: 📤 Upload Optimization Reports
      uses: actions/upload-artifact@v3
      with:
        name: optimization-reports
        path: |
          routes.json
          query-analysis.txt
          potential-n-plus-one.txt
          migration-status.txt
```

---

## Advanced Scenarios

### 🔄 **Scenario 6: Multi-Branch Strategy**

```yaml
name: Multi-Branch Workflow

on:
  push:
    branches: ['**']
  pull_request:
    branches: [main, dev]

jobs:
  # ==================== BRANCH DETECTION ====================
  detect-changes:
    name: Detect Changes
    runs-on: ubuntu-latest
    outputs:
      backend: ${{ steps.changes.outputs.backend }}
      frontend: ${{ steps.changes.outputs.frontend }}
      docs: ${{ steps.changes.outputs.docs }}
      
    steps:
    - uses: actions/checkout@v4
    - uses: dorny/paths-filter@v2
      id: changes
      with:
        filters: |
          backend:
            - 'app/**'
            - 'config/**'
            - 'database/**'
            - 'routes/**'
            - 'composer.*'
          frontend:
            - 'resources/**'
            - 'public/**'
            - 'package*.json'
            - 'vite.config.js'
          docs:
            - 'docs/**'
            - '*.md'

  # ==================== CONDITIONAL JOBS ====================
  test-backend:
    needs: detect-changes
    if: needs.detect-changes.outputs.backend == 'true'
    runs-on: ubuntu-latest
    steps:
    - name: 🧪 Backend Tests
      run: echo "Running backend tests..."

  test-frontend:
    needs: detect-changes
    if: needs.detect-changes.outputs.frontend == 'true'
    runs-on: ubuntu-latest
    steps:
    - name: 🎨 Frontend Tests
      run: echo "Running frontend tests..."

  # ==================== BRANCH-SPECIFIC DEPLOYMENT ====================
  deploy:
    runs-on: ubuntu-latest
    if: github.event_name == 'push'
    steps:
    - name: 🚀 Deploy based on branch
      run: |
        case ${{ github.ref_name }} in
          main)
            echo "Deploying to production..."
            ;;
          dev)
            echo "Deploying to staging..."
            ;;
          feature/*)
            echo "Deploying to feature environment..."
            ;;
          *)
            echo "No deployment for this branch"
            ;;
        esac
```

### 🐳 **Scenario 7: Docker Integration**

```yaml
name: Docker CI/CD

on:
  push:
    branches: [main, dev]

env:
  REGISTRY: ghcr.io
  IMAGE_NAME: ${{ github.repository }}

jobs:
  # ==================== BUILD DOCKER IMAGE ====================
  build-image:
    name: Build Docker Image
    runs-on: ubuntu-latest
    permissions:
      contents: read
      packages: write
      
    outputs:
      image: ${{ steps.image.outputs.image }}
      
    steps:
    - name: 📥 Checkout
      uses: actions/checkout@v4

    - name: 🐳 Setup Docker Buildx
      uses: docker/setup-buildx-action@v3

    - name: 🔐 Login to Container Registry
      uses: docker/login-action@v3
      with:
        registry: ${{ env.REGISTRY }}
        username: ${{ github.actor }}
        password: ${{ secrets.GITHUB_TOKEN }}

    - name: 🏷️ Extract Metadata
      id: meta
      uses: docker/metadata-action@v5
      with:
        images: ${{ env.REGISTRY }}/${{ env.IMAGE_NAME }}
        tags: |
          type=ref,event=branch
          type=ref,event=pr
          type=sha,prefix={{branch}}-

    - name: 🏗️ Build and Push
      uses: docker/build-push-action@v5
      with:
        context: .
        push: true
        tags: ${{ steps.meta.outputs.tags }}
        labels: ${{ steps.meta.outputs.labels }}
        cache-from: type=gha
        cache-to: type=gha,mode=max

    - name: 📤 Output Image
      id: image
      run: echo "image=${{ env.REGISTRY }}/${{ env.IMAGE_NAME }}:${{ github.ref_name }}-${{ github.sha }}" >> $GITHUB_OUTPUT

  # ==================== SECURITY SCAN ====================
  security-scan:
    name: Security Scan
    runs-on: ubuntu-latest
    needs: build-image
    
    steps:
    - name: 🛡️ Run Trivy Scanner
      uses: aquasecurity/trivy-action@master
      with:
        image-ref: ${{ needs.build-image.outputs.image }}
        format: 'sarif'
        output: 'trivy-results.sarif'

    - name: 📤 Upload Trivy Results
      uses: github/codeql-action/upload-sarif@v2
      with:
        sarif_file: 'trivy-results.sarif'

  # ==================== DEPLOY WITH DOCKER ====================
  deploy:
    name: Deploy Container
    runs-on: ubuntu-latest
    needs: [build-image, security-scan]
    environment: ${{ github.ref_name == 'main' && 'production' || 'staging' }}
    
    steps:
    - name: 🚀 Deploy to Server
      uses: appleboy/ssh-action@v1.0.0
      with:
        host: ${{ secrets.DEPLOY_HOST }}
        username: ${{ secrets.DEPLOY_USER }}
        key: ${{ secrets.DEPLOY_KEY }}
        script: |
          # Pull new image
          docker pull ${{ needs.build-image.outputs.image }}
          
          # Stop old container
          docker stop laravel-app || true
          docker rm laravel-app || true
          
          # Run new container
          docker run -d \
            --name laravel-app \
            --restart unless-stopped \
            -p 80:80 \
            -e APP_ENV=${{ github.ref_name == 'main' && 'production' || 'staging' }} \
            -v /var/www/storage:/var/www/html/storage \
            ${{ needs.build-image.outputs.image }}
          
          # Health check
          sleep 30
          curl -f http://localhost/health || exit 1
```

---

## Troubleshooting

### 🚨 **Common Issues ও Solutions**

**1. Tests Failing:**
```yaml
# Debug steps যোগ করুন
- name: 🐛 Debug Environment
  run: |
    php --version
    composer --version
    php artisan --version
    cat .env
    php artisan config:show database

- name: 🗄️ Check Database Connection
  run: php artisan migrate:status
```

**2. Memory Issues:**
```yaml
# PHP memory limit বাড়ান
- name: 🐘 Setup PHP
  uses: shivammathur/setup-php@v2
  with:
    php-version: '8.2'
    ini-values: memory_limit=512M
```

**3. Timeout Issues:**
```yaml
# Timeout বাড়ান
jobs:
  test:
    timeout-minutes: 30  # Default 360 minutes
```

**4. Cache Issues:**
```yaml
# Cache clear করুন
- name: 🧹 Clear Caches
  run: |
    php artisan config:clear
    php artisan cache:clear
    php artisan route:clear
    php artisan view:clear
```

### 📊 **Monitoring ও Notifications**

```yaml
# Slack notification
- name: 🔔 Notify Slack
  uses: 8398a7/action-slack@v3
  with:
    status: ${{ job.status }}
    channel: '#ci-cd'
    webhook_url: ${{ secrets.SLACK_WEBHOOK }}
  if: always()

# Email notification
- name: 📧 Send Email
  uses: dawidd6/action-send-mail@v3
  with:
    server_address: smtp.gmail.com
    server_port: 587
    username: ${{ secrets.EMAIL_USERNAME }}
    password: ${{ secrets.EMAIL_PASSWORD }}
    subject: "Deployment Status: ${{ job.status }}"
    body: "Deployment to ${{ github.ref_name }} completed with status: ${{ job.status }}"
    to: team@company.com
  if: failure()
```

---

## 🎯 **Best Practices Summary**

### ✅ **Do's:**
- ✅ Use matrix strategy for multiple PHP/Laravel versions
- ✅ Cache dependencies (Composer, NPM)
- ✅ Use environment-specific secrets
- ✅ Implement proper error handling
- ✅ Add health checks after deployment
- ✅ Use artifacts for build outputs
- ✅ Implement blue-green deployment for zero downtime
- ✅ Add security scanning
- ✅ Monitor performance metrics

### ❌ **Don'ts:**
- ❌ Store secrets in code
- ❌ Run tests without proper database setup
- ❌ Deploy without running tests first
- ❌ Ignore security vulnerabilities
- ❌ Skip backup before deployment
- ❌ Use production data in tests
- ❌ Deploy directly to production from feature branches

### 🔧 **Optimization Tips:**
- 🚀 Use `--prefer-dist` for faster Composer installs
- 🚀 Enable OPcache in production
- 🚀 Use Redis for caching and sessions
- 🚀 Implement proper database indexing
- 🚀 Optimize images and assets
- 🚀 Use CDN for static assets
- 🚀 Enable gzip compression

---

এই গাইড Laravel projects এর জন্য comprehensive GitHub Actions workflows প্রদান করে। প্রতিটি scenario real-world use cases এর জন্য optimized এবং production-ready।