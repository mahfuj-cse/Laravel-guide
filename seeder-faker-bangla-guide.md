# Laravel Seeder এবং Faker - বিস্তারিত গাইড

## 📋 সূচিপত্র
- [Seeder কি?](#seeder-কি)
- [Faker কি?](#faker-কি)
- [Seeder vs Faker - পার্থক্য](#seeder-vs-faker---পার্থক্য)
- [Factory কি?](#factory-কি)
- [বাস্তব উদাহরণ](#বাস্তব-উদাহরণ)

---

## Seeder কি?

**Seeder** হলো Laravel এর একটি ফিচার যা ডাটাবেসে **নির্দিষ্ট ডেটা** ইনসার্ট করার জন্য ব্যবহৃত হয়।

### Seeder এর বৈশিষ্ট্য:
- ✅ **স্থির ডেটা** (Fixed Data) ইনসার্ট করে
- ✅ **প্রোডাকশন** এবং **টেস্টিং** দুই ক্ষেত্রেই ব্যবহার হয়
- ✅ **Admin User**, **Default Settings** এর মতো জরুরি ডেটা যোগ করে

### Seeder তৈরি করা:
```bash
php artisan make:seeder UserSeeder
```

### Seeder এর উদাহরণ:
```php
<?php

namespace Database\Seeders;

use Illuminate\Database\Seeder;
use App\Models\User;
use Illuminate\Support\Facades\Hash;

class UserSeeder extends Seeder
{
    public function run()
    {
        // নির্দিষ্ট Admin User তৈরি
        User::create([
            'name' => 'Admin User',
            'email' => 'admin@example.com',
            'password' => Hash::make('password123'),
            'role' => 'admin'
        ]);

        // নির্দিষ্ট Test User তৈরি
        User::create([
            'name' => 'Test User',
            'email' => 'test@example.com',
            'password' => Hash::make('password123'),
            'role' => 'user'
        ]);
    }
}
```

---

## Faker কি?

**Faker** হলো একটি PHP লাইব্রেরি যা **র‍্যান্ডম/নকল ডেটা** তৈরি করে।

### Faker এর বৈশিষ্ট্য:
- ✅ **র‍্যান্ডম ডেটা** জেনারেট করে
- ✅ **টেস্টিং** এর জন্য ব্যবহার হয়
- ✅ **বাস্তবসম্মত** কিন্তু **নকল** ডেটা তৈরি করে

### Faker এর উদাহরণ:
```php
<?php

namespace Database\Seeders;

use Illuminate\Database\Seeder;
use App\Models\User;
use Faker\Factory as Faker;

class FakeUserSeeder extends Seeder
{
    public function run()
    {
        $faker = Faker::create();

        // 50টি র‍্যান্ডম User তৈরি
        for ($i = 0; $i < 50; $i++) {
            User::create([
                'name' => $faker->name,           // র‍্যান্ডম নাম
                'email' => $faker->email,         // র‍্যান্ডম ইমেইল
                'password' => bcrypt('password'), // একই পাসওয়ার্ড
                'phone' => $faker->phoneNumber,   // র‍্যান্ডম ফোন
                'address' => $faker->address,     // র‍্যান্ডম ঠিকানা
            ]);
        }
    }
}
```

---

## Seeder vs Faker - পার্থক্য

| বিষয় | Seeder | Faker |
|-------|--------|-------|
| **উদ্দেশ্য** | ডাটাবেসে ডেটা ইনসার্ট করা | র‍্যান্ডম ডেটা জেনারেট করা |
| **ডেটার ধরন** | নির্দিষ্ট/স্থির ডেটা | র‍্যান্ডম/নকল ডেটা |
| **ব্যবহার** | প্রোডাকশন + টেস্টিং | শুধু টেস্টিং |
| **উদাহরণ** | Admin User, Settings | Test Users, Sample Posts |
| **কমান্ড** | `php artisan make:seeder` | Faker লাইব্রেরি ব্যবহার |

### সহজ ভাষায়:
- **Seeder** = ডেটা ভরার পদ্ধতি 🗂️
- **Faker** = নকল ডেটা তৈরির টুল 🎭

---

## Factory কি?

**Factory** হলো Model এর জন্য **টেমপ্লেট** যা Faker ব্যবহার করে র‍্যান্ডম ডেটা তৈরি করে।

### Factory তৈরি করা:
```bash
php artisan make:factory PostFactory
```

### Factory এর উদাহরণ:
```php
<?php

namespace Database\Factories;

use Illuminate\Database\Eloquent\Factories\Factory;
use App\Models\User;

class PostFactory extends Factory
{
    public function definition()
    {
        return [
            'title' => $this->faker->sentence(6),        // র‍্যান্ডম টাইটেল
            'content' => $this->faker->paragraphs(3, true), // র‍্যান্ডম কন্টেন্ট
            'user_id' => User::factory(),                // র‍্যান্ডম User
            'views' => $this->faker->numberBetween(0, 1000), // র‍্যান্ডম ভিউ
            'published' => $this->faker->boolean(70),    // 70% সম্ভাবনায় true
        ];
    }
}
```

### Factory ব্যবহার:
```php
// Seeder এ Factory ব্যবহার
class PostSeeder extends Seeder
{
    public function run()
    {
        // 100টি র‍্যান্ডম Post তৈরি
        \App\Models\Post::factory(100)->create();
    }
}
```

---

## বাস্তব উদাহরণ

### ১. নির্দিষ্ট ডেটার জন্য Seeder:
```php
class RoleSeeder extends Seeder
{
    public function run()
    {
        // এই ডেটা সবসময় একই থাকবে
        $roles = ['admin', 'editor', 'user'];
        
        foreach ($roles as $role) {
            Role::create(['name' => $role]);
        }
    }
}
```

### ২. টেস্ট ডেটার জন্য Faker + Factory:
```php
class TestDataSeeder extends Seeder
{
    public function run()
    {
        // 50টি র‍্যান্ডম User
        User::factory(50)->create();
        
        // প্রতি User এর জন্য 1-5টি Post
        User::all()->each(function ($user) {
            Post::factory(rand(1, 5))->create([
                'user_id' => $user->id
            ]);
        });
    }
}
```

### ৩. DatabaseSeeder এ সব একসাথে:
```php
class DatabaseSeeder extends Seeder
{
    public function run()
    {
        // প্রথমে নির্দিষ্ট ডেটা
        $this->call([
            RoleSeeder::class,      // স্থির ডেটা
            UserSeeder::class,      // Admin User
        ]);
        
        // তারপর টেস্ট ডেটা (শুধু development এ)
        if (app()->environment('local')) {
            $this->call([
                TestDataSeeder::class,  // র‍্যান্ডম ডেটা
            ]);
        }
    }
}
```

### ৪. Seeder চালানো:
```bash
# সব Seeder চালানো
php artisan db:seed

# নির্দিষ্ট Seeder চালানো
php artisan db:seed --class=UserSeeder

# Migration + Seeder একসাথে
php artisan migrate:fresh --seed
```

---

## 🎯 মূল কথা:

### কখন Seeder ব্যবহার করবেন:
- ✅ Admin User তৈরি করতে
- ✅ Default Settings যোগ করতে
- ✅ জরুরি Master Data ইনসার্ট করতে

### কখন Faker ব্যবহার করবেন:
- ✅ টেস্টিং এর জন্য Sample Data তৈরি করতে
- ✅ UI/UX টেস্ট করতে
- ✅ Performance টেস্ট করতে

### সবচেয়ে ভালো Practice:
```php
// Real Data = Seeder
// Test Data = Seeder + Factory + Faker
```

---

## 📚 আরও জানতে:
- [Laravel Seeding Documentation](https://laravel.com/docs/seeding)
- [Faker Documentation](https://fakerphp.github.io/)
- [Model Factories](https://laravel.com/docs/eloquent-factories)