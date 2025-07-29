# 9️⃣ Laravel Migrations & Seeders - বিস্তারিত বাংলা গাইড

## 📋 সূচিপত্র
- [Migration কি?](#migration-কি)
- [Migration তৈরি ও ব্যবহার](#migration-তৈরি-ও-ব্যবহার)
- [Schema Building](#schema-building)
- [Seeder কি?](#seeder-কি)
- [Database Seeding](#database-seeding)
- [Factory কি?](#factory-কি)
- [সম্পূর্ণ উদাহরণ](#সম্পূর্ণ-উদাহরণ)

---

## Migration কি?

**Migration** হলো ডাটাবেস স্কিমার **Version Control System**। এটি দিয়ে আপনি:
- ✅ টেবিল তৈরি/মুছে ফেলতে পারেন
- ✅ কলাম যোগ/বাদ দিতে পারেন  
- ✅ ইনডেক্স তৈরি করতে পারেন
- ✅ টিম মেম্বারদের সাথে ডাটাবেস স্ট্রাকচার শেয়ার করতে পারেন

### Migration এর সুবিধা:
- 🔄 **Rollback** করা যায়
- 👥 **Team Collaboration** সহজ
- 🏗️ **Database Structure** ট্র্যাক করা যায়

---

## Migration তৈরি ও ব্যবহার

### ১. Migration তৈরি করা:
```bash
# নতুন টেবিল তৈরির জন্য
php artisan make:migration create_posts_table

# বিদ্যমান টেবিল পরিবর্তনের জন্য
php artisan make:migration add_status_to_posts_table --table=posts

# কলাম যোগ করার জন্য
php artisan make:migration add_image_to_posts_table
```

### ২. নতুন টেবিল তৈরি:
```php
<?php
// database/migrations/2024_01_01_000000_create_posts_table.php

use Illuminate\Database\Migrations\Migration;
use Illuminate\Database\Schema\Blueprint;
use Illuminate\Support\Facades\Schema;

return new class extends Migration
{
    public function up()
    {
        Schema::create('posts', function (Blueprint $table) {
            $table->id();                           // Primary Key
            $table->string('title');                // VARCHAR(255)
            $table->text('content');                // TEXT
            $table->string('slug')->unique();       // Unique Slug
            $table->enum('status', ['draft', 'published'])->default('draft');
            $table->integer('views')->default(0);   // View Count
            $table->foreignId('user_id')->constrained()->onDelete('cascade');
            $table->timestamps();                   // created_at, updated_at
        });
    }

    public function down()
    {
        Schema::dropIfExists('posts');
    }
};
```

### ৩. বিদ্যমান টেবিলে কলাম যোগ:
```php
<?php
// database/migrations/2024_01_02_000000_add_image_to_posts_table.php

return new class extends Migration
{
    public function up()
    {
        Schema::table('posts', function (Blueprint $table) {
            $table->string('featured_image')->nullable()->after('content');
            $table->boolean('is_featured')->default(false)->after('status');
            $table->json('meta_data')->nullable();
        });
    }

    public function down()
    {
        Schema::table('posts', function (Blueprint $table) {
            $table->dropColumn(['featured_image', 'is_featured', 'meta_data']);
        });
    }
};
```

### ৪. Migration চালানো:
```bash
# সব Migration চালানো
php artisan migrate

# নির্দিষ্ট Migration চালানো
php artisan migrate --path=/database/migrations/2024_01_01_000000_create_posts_table.php

# Migration Rollback
php artisan migrate:rollback

# সব Migration Reset করে আবার চালানো
php artisan migrate:fresh

# Migration Status দেখা
php artisan migrate:status
```

---

## Schema Building

### কলামের ধরন (Column Types):
```php
Schema::create('users', function (Blueprint $table) {
    // সংখ্যা
    $table->id();                           // BIGINT UNSIGNED AUTO_INCREMENT
    $table->integer('age');                 // INT
    $table->bigInteger('phone');            // BIGINT
    $table->decimal('price', 8, 2);         // DECIMAL(8,2)
    $table->float('rating', 3, 1);          // FLOAT(3,1)
    
    // টেক্সট
    $table->string('name', 100);            // VARCHAR(100)
    $table->text('description');            // TEXT
    $table->longText('content');            // LONGTEXT
    $table->char('code', 10);               // CHAR(10)
    
    // তারিখ ও সময়
    $table->date('birth_date');             // DATE
    $table->time('start_time');             // TIME
    $table->datetime('event_datetime');     // DATETIME
    $table->timestamp('created_at');        // TIMESTAMP
    $table->timestamps();                   // created_at + updated_at
    
    // বুলিয়ান
    $table->boolean('is_active');           // BOOLEAN
    
    // JSON
    $table->json('settings');               // JSON
    
    // ENUM
    $table->enum('status', ['active', 'inactive']);
});
```

### কলাম Modifiers:
```php
$table->string('email')->nullable();           // NULL হতে পারে
$table->string('username')->unique();          // Unique
$table->integer('sort_order')->default(0);     // Default Value
$table->string('title')->after('name');        // নির্দিষ্ট কলামের পরে
$table->string('description')->comment('Post description'); // Comment
```

### Index তৈরি:
```php
// Primary Key
$table->primary('id');

// Unique Index
$table->unique('email');
$table->unique(['first_name', 'last_name'], 'unique_name');

// Regular Index
$table->index('user_id');
$table->index(['user_id', 'created_at'], 'user_created_index');

// Foreign Key
$table->foreignId('user_id')->constrained();
$table->foreignId('category_id')->constrained('categories')->onDelete('cascade');
```

---

## Seeder কি?

**Seeder** হলো ডাটাবেসে **প্রাথমিক ডেটা** ভরার পদ্ধতি।

### Seeder এর ব্যবহার:
- ✅ **Admin User** তৈরি
- ✅ **Default Settings** যোগ করা
- ✅ **Master Data** (Country, Category) ইনসার্ট
- ✅ **Test Data** তৈরি

### ১. Seeder তৈরি:
```bash
php artisan make:seeder UserSeeder
php artisan make:seeder PostSeeder
php artisan make:seeder CategorySeeder
```

### ২. Basic Seeder:
```php
<?php
// database/seeders/UserSeeder.php

namespace Database\Seeders;

use Illuminate\Database\Seeder;
use App\Models\User;
use Illuminate\Support\Facades\Hash;

class UserSeeder extends Seeder
{
    public function run()
    {
        // Admin User তৈরি
        User::create([
            'name' => 'Admin User',
            'email' => 'admin@example.com',
            'password' => Hash::make('password123'),
            'role' => 'admin',
            'email_verified_at' => now(),
        ]);

        // Regular Users
        $users = [
            ['name' => 'John Doe', 'email' => 'john@example.com'],
            ['name' => 'Jane Smith', 'email' => 'jane@example.com'],
            ['name' => 'Bob Wilson', 'email' => 'bob@example.com'],
        ];

        foreach ($users as $userData) {
            User::create([
                'name' => $userData['name'],
                'email' => $userData['email'],
                'password' => Hash::make('password123'),
                'role' => 'user',
                'email_verified_at' => now(),
            ]);
        }
    }
}
```

### ৩. Category Seeder:
```php
<?php
// database/seeders/CategorySeeder.php

class CategorySeeder extends Seeder
{
    public function run()
    {
        $categories = [
            'Technology',
            'Health',
            'Education',
            'Travel',
            'Food',
            'Sports'
        ];

        foreach ($categories as $category) {
            \App\Models\Category::create([
                'name' => $category,
                'slug' => \Str::slug($category),
                'description' => "This is {$category} category",
            ]);
        }
    }
}
```

---

## Database Seeding

### ১. DatabaseSeeder এ সব একসাথে:
```php
<?php
// database/seeders/DatabaseSeeder.php

namespace Database\Seeders;

use Illuminate\Database\Seeder;

class DatabaseSeeder extends Seeder
{
    public function run()
    {
        // প্রথমে Master Data
        $this->call([
            CategorySeeder::class,
            UserSeeder::class,
        ]);

        // তারপর Dependent Data
        $this->call([
            PostSeeder::class,
            CommentSeeder::class,
        ]);

        // Development Environment এ Test Data
        if (app()->environment('local')) {
            $this->call([
                TestDataSeeder::class,
            ]);
        }
    }
}
```

### ২. Seeder চালানো:
```bash
# সব Seeder চালানো
php artisan db:seed

# নির্দিষ্ট Seeder চালানো
php artisan db:seed --class=UserSeeder

# Migration + Seeder একসাথে
php artisan migrate:fresh --seed

# Production এ Seeder চালানো
php artisan db:seed --force
```

---

## Factory কি?

**Factory** হলো Model এর জন্য **টেমপ্লেট** যা **Faker** ব্যবহার করে র‍্যান্ডম ডেটা তৈরি করে।

### ১. Factory তৈরি:
```bash
php artisan make:factory PostFactory
php artisan make:factory CommentFactory
```

### ২. Post Factory:
```php
<?php
// database/factories/PostFactory.php

namespace Database\Factories;

use Illuminate\Database\Eloquent\Factories\Factory;
use App\Models\User;
use App\Models\Category;

class PostFactory extends Factory
{
    public function definition()
    {
        $title = $this->faker->sentence(6);
        
        return [
            'title' => $title,
            'slug' => \Str::slug($title),
            'content' => $this->faker->paragraphs(5, true),
            'excerpt' => $this->faker->paragraph(2),
            'featured_image' => $this->faker->imageUrl(800, 600, 'posts'),
            'status' => $this->faker->randomElement(['draft', 'published']),
            'views' => $this->faker->numberBetween(0, 10000),
            'is_featured' => $this->faker->boolean(20), // 20% সম্ভাবনা
            'user_id' => User::factory(),
            'category_id' => Category::factory(),
            'published_at' => $this->faker->dateTimeBetween('-1 year', 'now'),
        ];
    }

    // State Methods
    public function published()
    {
        return $this->state(function (array $attributes) {
            return [
                'status' => 'published',
                'published_at' => now(),
            ];
        });
    }

    public function draft()
    {
        return $this->state(function (array $attributes) {
            return [
                'status' => 'draft',
                'published_at' => null,
            ];
        });
    }
}
```

### ৩. Factory ব্যবহার:
```php
<?php
// database/seeders/PostSeeder.php

class PostSeeder extends Seeder
{
    public function run()
    {
        // 50টি র‍্যান্ডম Post
        \App\Models\Post::factory(50)->create();

        // 20টি Published Post
        \App\Models\Post::factory(20)->published()->create();

        // 10টি Draft Post
        \App\Models\Post::factory(10)->draft()->create();

        // নির্দিষ্ট User এর জন্য Post
        $user = \App\Models\User::find(1);
        \App\Models\Post::factory(5)->create([
            'user_id' => $user->id,
        ]);
    }
}
```

---

## সম্পূর্ণ উদাহরণ

### ১. Blog System এর Migration:
```php
// Users Table
Schema::create('users', function (Blueprint $table) {
    $table->id();
    $table->string('name');
    $table->string('email')->unique();
    $table->string('password');
    $table->enum('role', ['admin', 'editor', 'user'])->default('user');
    $table->timestamp('email_verified_at')->nullable();
    $table->timestamps();
});

// Categories Table
Schema::create('categories', function (Blueprint $table) {
    $table->id();
    $table->string('name');
    $table->string('slug')->unique();
    $table->text('description')->nullable();
    $table->timestamps();
});

// Posts Table
Schema::create('posts', function (Blueprint $table) {
    $table->id();
    $table->string('title');
    $table->string('slug')->unique();
    $table->text('excerpt')->nullable();
    $table->longText('content');
    $table->string('featured_image')->nullable();
    $table->enum('status', ['draft', 'published'])->default('draft');
    $table->integer('views')->default(0);
    $table->boolean('is_featured')->default(false);
    $table->timestamp('published_at')->nullable();
    $table->foreignId('user_id')->constrained()->onDelete('cascade');
    $table->foreignId('category_id')->constrained()->onDelete('cascade');
    $table->timestamps();
});
```

### ২. Complete Seeder:
```php
<?php
// database/seeders/BlogSeeder.php

class BlogSeeder extends Seeder
{
    public function run()
    {
        // Admin User
        $admin = User::create([
            'name' => 'Admin',
            'email' => 'admin@blog.com',
            'password' => Hash::make('password'),
            'role' => 'admin',
        ]);

        // Categories
        $categories = ['Tech', 'Health', 'Travel', 'Food'];
        foreach ($categories as $cat) {
            Category::create([
                'name' => $cat,
                'slug' => Str::slug($cat),
                'description' => "All about {$cat}",
            ]);
        }

        // Random Users
        User::factory(10)->create();

        // Posts for each category
        Category::all()->each(function ($category) {
            Post::factory(5)->create([
                'category_id' => $category->id,
                'status' => 'published',
            ]);
        });
    }
}
```

### ৩. চালানোর কমান্ড:
```bash
# Migration চালানো
php artisan migrate

# Seeder চালানো
php artisan db:seed --class=BlogSeeder

# সব একসাথে
php artisan migrate:fresh --seed
```

---

## 🎯 Best Practices:

### Migration এর জন্য:
- ✅ সবসময় `down()` method লিখুন
- ✅ Foreign Key Constraint ব্যবহার করুন
- ✅ Index যোগ করুন যেখানে প্রয়োজন
- ✅ Column নাম meaningful রাখুন

### Seeder এর জন্য:
- ✅ Production Data আলাদা রাখুন
- ✅ Test Data Environment অনুযায়ী চালান
- ✅ Factory ব্যবহার করে র‍্যান্ডম ডেটা তৈরি করুন
- ✅ DatabaseSeeder এ সব organize করুন

---

## 📚 আরও জানতে:
- [Laravel Migrations](https://laravel.com/docs/migrations)
- [Database Seeding](https://laravel.com/docs/seeding)
- [Model Factories](https://laravel.com/docs/eloquent-factories)