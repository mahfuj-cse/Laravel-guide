# Laravel টেস্টিং এবং GitHub Actions - একটি সম্পূর্ণ গাইড

এই গাইডটিতে আমরা Laravel অ্যাপ্লিকেশন টেস্টিং এবং GitHub Actions ব্যবহার করে কিভাবে স্বয়ংক্রিয়ভাবে টেস্ট চালানো যায়, তা বিস্তারিত আলোচনা করবো।

## 📋 সূচিপত্র
- [Laravel টেস্টিং কি?](#laravel-টেস্টিং-কি)
- [GitHub Actions কি?](#github-actions-কি)
- [Workflow ফাইলের বিস্তারিত আলোচনা](#workflow-ফাইলের-বিস্তারিত-আলোচনা)
  - [Workflow Triggers (`on`)](#workflow-triggers-on)
  - Jobs এবং Services
  - Workflow Steps
- Best Practices এবং পরামর্শ

---

## Laravel টেস্টিং কি?

Laravel টেস্টিং হলো আপনার অ্যাপ্লিকেশনের বিভিন্ন অংশ (যেমন, কোড, ফাংশনালিটি, API) সঠিকভাবে কাজ করছে কিনা তা স্বয়ংক্রিয়ভাবে যাচাই করার একটি প্রক্রিয়া। এটি কোডের গুণমান নিশ্চিত করে এবং ভবিষ্যতে কোনো পরিবর্তন আনার ফলে পুরনো কোডে সমস্যা হচ্ছে কিনা (Regression) তা ধরতে সাহায্য করে।

**প্রধানত দুই ধরনের টেস্ট হয়:**
1.  **Unit Test**: ছোট ছোট কোডের অংশ (যেমন, একটি নির্দিষ্ট মেথড) আলাদাভাবে টেস্ট করা হয়।
2.  **Feature Test**: অ্যাপ্লিকেশনের বড় কোনো ফিচার (যেমন, একজন ইউজার রেজিস্ট্রেশন থেকে শুরু করে লগইন পর্যন্ত) টেস্ট করা হয়।

Laravel এ PHPUnit এবং Pest ব্যবহার করে খুব সহজেই টেস্ট লেখা যায়।

**টেস্ট তৈরির কমান্ড:**
```bash
# একটি Feature Test তৈরির জন্য
php artisan make:test UserRegistrationTest

# একটি Unit Test তৈরির জন্য
php artisan make:test UserTest --unit
```

---

## GitHub Actions কি?

**GitHub Actions** হলো একটি **CI/CD (Continuous Integration/Continuous Deployment)** টুল যা GitHub এর সাথে বিল্ট-ইন থাকে। এর মাধ্যমে আপনি আপনার GitHub ریپোজিটরিতে কোড `push` বা `pull request` করার মতো বিভিন্ন ইভেন্টের উপর ভিত্তি করে স্বয়ংক্রিয়ভাবে বিভিন্ন কাজ (যেমন, টেস্টিং, বিল্ড, ডেপ্লয়) চালাতে পারেন।

এই কাজগুলো **workflow** নামক YAML ফایলে লেখা হয়, যা `.github/workflows` ডিরেক্টরিতে থাকে।

---

## Workflow ফাইলের বিস্তারিত আলোচনা

আপনার দেওয়া workflow ফাইলটি একটি চমৎকার উদাহরণ যা শুধুমাত্র পরিবর্তিত ফাইলগুলোর জন্য টেস্ট চালায়। আসুন এর প্রতিটি অংশ বিস্তারিত বুঝি।

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

- **`pull_request`**: যখন `main` বা `test` ব্রাঞ্চে কোনো pull request করা হয়।
- **`push`**: যখন `main` বা `test` ব্রাঞ্চে সরাসরি কোড push করা হয়।
- **`paths`**: এই workflow শুধুমাত্র তখনই চলবে যদি `tests/`, `app/` ফোল্ডারের কোনো ফাইল অথবা `composer.json` / `composer.lock` ফাইল পরিবর্তন হয়। এটি অপ্রয়োজনীয় টেস্ট রান বন্ধ করে রিসোর্স বাঁচায়।

### Jobs এবং Services

- **`jobs`**: একটি workflow-তে এক বা একাধিক job থাকতে পারে। এখানে `test` নামে একটি job আছে।
- **`runs-on: ubuntu-latest`**: এই job-টি GitHub-hosted একটি Ubuntu runner-এ চলবে।
- **`services`**: job চলাকালীন কোনো সার্ভিস (যেমন, ডাটাবেস, Redis) প্রয়োজন হলে এখানে define করা হয়।
  - **`mysql: image: mysql:8.0`**: টেস্টের জন্য একটি MySQL 8.0 ডাটাবেস কন্টেইনার চালু করা হয়েছে।
  - **`env`**: ডাটাবেসের জন্য প্রয়োজনীয় এনভায়রনমেন্ট ভ্যারিয়েবল (root password, database name) সেট করা হয়েছে।
  - **`ports`**: কন্টেইনারের 3306 পোর্টটি runner-এর 3306 পোর্টের সাথে ম্যাপ করা হয়েছে, જેથી `127.0.0.1:3306` দিয়ে কানেক্ট করা যায়।
  - **`options`**: MySQL سروسটি সঠিকভাবে চালু হয়েছে কিনা তা নিশ্চিত করার জন্য হেলথ চেক কমান্ড দেওয়া হয়েছে।

### Workflow Steps

`steps` অংশে job-এর প্রতিটি ধাপ 순서ಾನುসারে লেখা থাকে।

1.  **`uses: actions/checkout@v3`**:
    - এই action-টি আপনার ریپোজিটরির কোড runner-এ checkout (download) করে।

2.  **`name: Setup PHP`**:
    - `shivammathur/setup-php@v2` action ব্যবহার করে PHP 8.1 ইনস্টল করা হয়।
    - `extensions`: Laravel এর জন্য প্রয়োজনীয় PHP এক্সটেনশনগুলো (mbstring, mysql, gd ইত্যাদি) ইনস্টল করা হয়।
    - `coverage: xdebug`: টেস্ট কভারেজ রিপোর্ট তৈরির জন্য Xdebug ইনস্টল করা হয়।

3.  **`name: Install Dependencies`**:
    - `composer install` কমান্ডের মাধ্যমে প্রজেক্টের সব PHP dependency ইনস্টল করা হয়।
    - `-q --no-ansi ...`: এই ফ্ল্যাগগুলো CI পরিবেশে দ্রুত এবং অপ্রয়োজনীয় আউটপুট ছাড়া ইনস্টলেশন নিশ্চিত করে।

4.  **`name: Set Environment Variables`**:
    - `.env.example` ফাইল থেকে `.env` ফাইল তৈরি করা হয় এবং `php artisan key:generate` দিয়ে অ্যাপলিকেশন কী জেনারেট করা হয়। এটি Laravel এর জন্য একটি 필수 ধাপ।
    - **দ্রষ্টব্য:** আপনার workflow-তে `secrets` ব্যবহার করে অনেকগুলো `env` সেট করা হয়েছে। `.env.example` কপি করে `key:generate` করা একটি সহজ এবং প্রচলিত পদ্ধতি।

5.  **`name: Directory Permissions`**:
    - Laravel এর `storage` এবং `bootstrap/cache` ডিরেক্টরিগুলোতে লেখার অনুমতি (permission) দেওয়া হয়।

6.  **`name: Run Database Migrations`**:
    - `php artisan migrate --force` কমান্ডের মাধ্যমে `testing` ডাটাবেসে সব মাইগ্রেশন চালানো হয়।
    - `env` ব্লকে ডাটাবেস কানেকশনের জন্য প্রয়োজনীয় তথ্য (host, port, database, username, password) দেওয়া হয়েছে, যা `services` অংশে define করা MySQL কন্টেইনারের সাথে মিলে যায়।

7.  **`name: Get changed test files`**:
    - এটি একটি দারুণ অপটিমাইজেশন। `tj-actions/changed-files@v39` action-টি ব্যবহার করে শুধুমাত্র সেই টেস্ট ফাইলগুলো (`tests/**/*.php`) খুঁজে বের করা হয় যেগুলো বর্তমান push বা pull request-এ পরিবর্তন করা হয়েছে।
    - এর আউটপুট (`steps.changed-files.outputs.all_changed_files`) পরবর্তী ধাপে ব্যবহার করা হয়।

8.  **`name: Run Tests`**:
    - **Conditional Logic**: প্রথমে চেক করা হয় `any_changed` আউটপুট `true` কিনা।
    - **যদি টেস্ট ফাইল পরিবর্তন হয়**:
      - একটি `for` লুপের মাধ্যমে পরিবর্তিত প্রতিটি টেস্ট ফাইলের জন্য `php artisan test "$file"` কমান্ডটি আলাদাভাবে চালানো হয়।
      - যদি কোনো একটি টেস্ট ফেইল করে, `TEST_FAILED` ভ্যারিয়েবল `1` সেট করা হয় এবং শেষে `exit 1` দিয়ে workflow-টি ফেইল করানো হয়।
      - সব টেস্ট পাস করলে "All tests passed!" 메시지 দেখানো হয়।
    - **যদি কোনো টেস্ট ফাইল পরিবর্তন না হয়**:
      - টেস্ট চালানো স্কিপ করা হয় এবং একটি 메시지 দেখানো হয়। এই পদ্ধতিটি CI রান টাইম بشكل كبير কমিয়ে আনে।

---

## Best Practices এবং পরামর্শ

1.  **`.env` ফাইল ম্যানেজমেন্ট**:
    - আপনার workflow-তে `secrets` থেকে সরাসরি `GITHUB_ENV`-তে ভ্যারিয়েবল সেট করা হয়েছে। এটি কাজ করলেও, `.env.testing` নামে একটি ফাইল ریپোজিটরিতে রেখে CI-তে `.env` হিসেবে কপি করা একটি পরিষ্কার পদ্ধতি।
    - `.env.testing` ফাইলে `DB_DATABASE=testing`, `DB_USERNAME=root`, `DB_PASSWORD=password` এর মতো CI-specific ভ্যারিয়েবলগুলো রাখুন।

2.  **ক্যাশিং (Caching)**:
    - `composer` dependency এবং `npm` dependency (যদি থাকে) ক্যাশ করে রাখলে workflow রান টাইম আরও কমে যাবে। `actions/cache` action-টি এর জন্য ব্যবহার করতে পারেন।

3.  **প্যারালাল টেস্টিং (Parallel Testing)**:
    - বড় প্রজেক্টের ক্ষেত্রে, টেস্টগুলোকে একাধিক job-এ ভাগ করে প্যারালালি চালালে মোট সময় কমে আসে। Laravel 9+ এ প্যারালাল টেস্টিং এর বিল্ট-ইন সাপোর্ট আছে।

4.  **কোড কভারেজ (Code Coverage)**:
    - টেস্ট চালানোর সময় `--coverage` ফ্ল্যাগ ব্যবহার করে কোড কভারেজ রিপোর্ট তৈরি করতে পারেন এবং Codecov বা Coveralls এর মতো سرویسে আপলোড করে কভারেজ ট্র্যাক করতে পারেন।

5.  **সম্পূর্ণ টেস্ট স্যুট চালানো**:
    - শুধুমাত্র পরিবর্তিত ফাইল টেস্ট করা সময় বাঁচায়, কিন্তু মাঝে মাঝে (যেমন, `main` ব্রাঞ্চে merge করার আগে) সম্পূর্ণ টেস্ট স্যুট (`php artisan test`) চালানো উচিত ताकि কোনো Regression না হয়। এর জন্য আলাদা একটি workflow বা job তৈরি করতে পারেন।

এই গাইডটি আপনাকে Laravel টেস্টিং এবং GitHub Actions এর মাধ্যমে একটি শক্তিশালী CI pipeline তৈরি করতে সাহায্য করবে।

