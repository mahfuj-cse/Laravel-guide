# 7️⃣ Laravel Eloquent ORM - বিস্তারিত বাংলা গাইড

## 📋 সূচিপত্র
- [Eloquent ORM কি?](#eloquent-orm-কি)
- [Models তৈরি ও Configuration](#models-তৈরি-ও-configuration)
- [Mass Assignment](#mass-assignment)
- [Relationships](#relationships)
- [Query Operations](#query-operations)
- [Mutators & Accessors](#mutators--accessors)
- [Model Events](#model-events)
- [Advanced Features](#advanced-features)
- [Best Practices](#best-practices)

---

## Eloquent ORM কি?

**Eloquent ORM** হলো Laravel এর **Object-Relational Mapping** সিস্টেম যা Database Table গুলোকে **PHP Objects** হিসেবে কাজ করতে দেয়।

### 🔥 Eloquent এর সুবিধা:
- ✅ **Intuitive Syntax** - সহজ এবং readable code
- ✅ **Automatic Relationships** - Foreign key relationships
- ✅ **Query Builder Integration** - Powerful query methods
- ✅ **Model Events** - Lifecycle hooks
- ✅ **Mass Assignment Protection** - Security features

### Raw SQL vs Eloquent:
```php
// Raw SQL (কঠিন)
$users = DB::select('SELECT * FROM users WHERE active = ? AND created_at > ?', [1, '2024-01-01']);

// Eloquent (সহজ)
$users = User::where('active', true)->where('created_at', '>', '2024-01-01')->get();
```

---

## Models তৈরি ও Configuration

### ১. Model তৈরি করা:
```bash
# Basic Model
php artisan make:model Post

# Model with Migration
php artisan make:model Post -m

# Model with Migration, Factory, Seeder
php artisan make:model Post -mfs

# Model with Controller
php artisan make:model Post -c

# All together
php artisan make:model Post -a
```

### ২. Basic Model Structure:
```php
<?php
// app/Models/Post.php

namespace App\Models;

use Illuminate\Database\Eloquent\Factories\HasFactory;
use Illuminate\Database\Eloquent\Model;
use Illuminate\Database\Eloquent\SoftDeletes;

class Post extends Model
{
    use HasFactory, SoftDeletes;

    // Table name (optional if follows convention)
    protected $table = 'posts';

    // Primary key (optional if 'id')
    protected $primaryKey = 'id';

    // Primary key type
    protected $keyType = 'int';

    // Auto-incrementing
    public $incrementing = true;

    // Timestamps
    public $timestamps = true;

    // Date format
    protected $dateFormat = 'Y-m-d H:i:s';

    // Connection name
    protected $connection = 'mysql';

    // Fillable attributes (Mass Assignment)
    protected $fillable = [
        'title',
        'content',
        'excerpt',
        'status',
        'user_id',
        'category_id',
        'published_at'
    ];

    // Guarded attributes
    protected $guarded = ['id', 'created_at', 'updated_at'];

    // Hidden attributes (for JSON)
    protected $hidden = ['password', 'remember_token'];

    // Visible attributes (for JSON)
    protected $visible = ['title', 'content', 'status'];

    // Attribute casting
    protected $casts = [
        'published_at' => 'datetime',
        'is_featured' => 'boolean',
        'meta_data' => 'array',
        'settings' => 'json'
    ];

    // Default attribute values
    protected $attributes = [
        'status' => 'draft',
        'is_featured' => false
    ];
}
```

### ৩. Model Conventions:
```php
// Laravel Naming Conventions
Model Name: Post          → Table: posts
Model Name: BlogPost      → Table: blog_posts
Model Name: UserProfile   → Table: user_profiles

// Primary Key: id (auto-increment)
// Timestamps: created_at, updated_at
// Foreign Keys: user_id, category_id
```

---

## Mass Assignment

### ১. Mass Assignment কি?
**Mass Assignment** হলো একসাথে অনেক attributes assign করার পদ্ধতি।

```php
// Mass Assignment
$post = Post::create([
    'title' => 'My Post',
    'content' => 'Post content',
    'user_id' => 1
]);

// Individual Assignment
$post = new Post();
$post->title = 'My Post';
$post->content = 'Post content';
$post->user_id = 1;
$post->save();
```

### ২. Fillable vs Guarded:
```php
class Post extends Model
{
    // ✅ Fillable - Allow these fields
    protected $fillable = [
        'title', 'content', 'user_id', 'status'
    ];

    // ❌ Guarded - Protect these fields
    protected $guarded = [
        'id', 'created_at', 'updated_at'
    ];

    // 🚫 All guarded
    protected $guarded = ['*'];

    // ⚠️ All fillable (dangerous)
    protected $fillable = ['*'];
}
```

### ৩. Mass Assignment Security:
```php
// ❌ Dangerous - User can inject any field
$post = Post::create($request->all());

// ✅ Safe - Only validated fields
$post = Post::create($request->validated());

// ✅ Safe - Specific fields only
$post = Post::create($request->only(['title', 'content']));

// ✅ Safe - With fillable protection
class Post extends Model
{
    protected $fillable = ['title', 'content', 'user_id'];
}
$post = Post::create($request->all()); // Only fillable fields will be used
```

---

## Relationships

### ১. One-to-One (hasOne / belongsTo):
```php
// User Model
class User extends Model
{
    public function profile()
    {
        return $this->hasOne(Profile::class);
    }
}

// Profile Model
class Profile extends Model
{
    protected $fillable = ['user_id', 'bio', 'avatar'];

    public function user()
    {
        return $this->belongsTo(User::class);
    }
}

// Usage
$user = User::find(1);
$profile = $user->profile; // Get user's profile

$profile = Profile::find(1);
$user = $profile->user; // Get profile's user
```

### ২. One-to-Many (hasMany / belongsTo):
```php
// User Model
class User extends Model
{
    public function posts()
    {
        return $this->hasMany(Post::class);
    }
}

// Post Model
class Post extends Model
{
    protected $fillable = ['title', 'content', 'user_id'];

    public function user()
    {
        return $this->belongsTo(User::class);
    }

    public function comments()
    {
        return $this->hasMany(Comment::class);
    }
}

// Comment Model
class Comment extends Model
{
    protected $fillable = ['post_id', 'user_id', 'content'];

    public function post()
    {
        return $this->belongsTo(Post::class);
    }

    public function user()
    {
        return $this->belongsTo(User::class);
    }
}

// Usage
$user = User::find(1);
$posts = $user->posts; // Get all user's posts

$post = Post::find(1);
$author = $post->user; // Get post's author
$comments = $post->comments; // Get post's comments
```

### ৩. Many-to-Many (belongsToMany):
```php
// User Model
class User extends Model
{
    public function roles()
    {
        return $this->belongsToMany(Role::class);
    }

    // With pivot table customization
    public function roles()
    {
        return $this->belongsToMany(Role::class, 'user_roles', 'user_id', 'role_id')
                    ->withPivot('assigned_at', 'assigned_by')
                    ->withTimestamps();
    }
}

// Role Model
class Role extends Model
{
    protected $fillable = ['name', 'description'];

    public function users()
    {
        return $this->belongsToMany(User::class);
    }

    public function permissions()
    {
        return $this->belongsToMany(Permission::class);
    }
}

// Usage
$user = User::find(1);
$roles = $user->roles; // Get user's roles

// Attach/Detach roles
$user->roles()->attach([1, 2, 3]);
$user->roles()->detach([2]);
$user->roles()->sync([1, 3, 4]); // Replace all roles

// With pivot data
$user->roles()->attach(1, ['assigned_at' => now(), 'assigned_by' => auth()->id()]);
```

### ৪. Has-Many-Through:
```php
// Country Model
class Country extends Model
{
    public function posts()
    {
        return $this->hasManyThrough(Post::class, User::class);
    }
}

// User Model
class User extends Model
{
    public function country()
    {
        return $this->belongsTo(Country::class);
    }

    public function posts()
    {
        return $this->hasMany(Post::class);
    }
}

// Usage
$country = Country::find(1);
$posts = $country->posts; // Get all posts from users in this country
```

### ৫. Polymorphic Relationships:
```php
// Comment Model (can belong to Post or Video)
class Comment extends Model
{
    protected $fillable = ['content', 'commentable_id', 'commentable_type'];

    public function commentable()
    {
        return $this->morphTo();
    }
}

// Post Model
class Post extends Model
{
    public function comments()
    {
        return $this->morphMany(Comment::class, 'commentable');
    }
}

// Video Model
class Video extends Model
{
    public function comments()
    {
        return $this->morphMany(Comment::class, 'commentable');
    }
}

// Usage
$post = Post::find(1);
$comments = $post->comments; // Get post comments

$video = Video::find(1);
$comments = $video->comments; // Get video comments

// Create polymorphic comment
$post->comments()->create(['content' => 'Great post!']);
$video->comments()->create(['content' => 'Nice video!']);
```

---

## Query Operations

### ১. Basic CRUD Operations:
```php
// Create
$post = Post::create([
    'title' => 'My Post',
    'content' => 'Post content',
    'user_id' => 1
]);

// Read
$post = Post::find(1); // Find by ID
$post = Post::findOrFail(1); // Find or throw 404
$posts = Post::all(); // Get all
$posts = Post::where('status', 'published')->get(); // With condition

// Update
$post = Post::find(1);
$post->update(['title' => 'Updated Title']);

// Or
Post::where('id', 1)->update(['title' => 'Updated Title']);

// Delete
$post = Post::find(1);
$post->delete();

// Or
Post::destroy(1); // Delete by ID
Post::destroy([1, 2, 3]); // Delete multiple
Post::where('status', 'draft')->delete(); // Conditional delete
```

### ২. Query Builder Methods:
```php
// Where clauses
$posts = Post::where('status', 'published')->get();
$posts = Post::where('views', '>', 100)->get();
$posts = Post::where('title', 'like', '%Laravel%')->get();
$posts = Post::whereIn('status', ['published', 'featured'])->get();
$posts = Post::whereBetween('created_at', ['2024-01-01', '2024-12-31'])->get();
$posts = Post::whereNull('deleted_at')->get();

// Ordering
$posts = Post::orderBy('created_at', 'desc')->get();
$posts = Post::orderBy('title')->orderBy('created_at', 'desc')->get();
$posts = Post::latest()->get(); // Order by created_at desc
$posts = Post::oldest()->get(); // Order by created_at asc

// Limiting
$posts = Post::take(10)->get(); // Limit 10
$posts = Post::skip(20)->take(10)->get(); // Offset 20, Limit 10
$posts = Post::limit(10)->offset(20)->get(); // Same as above

// Grouping
$posts = Post::groupBy('status')->get();
$posts = Post::having('views', '>', 100)->get();

// Aggregates
$count = Post::count();
$max = Post::max('views');
$min = Post::min('views');
$avg = Post::avg('views');
$sum = Post::sum('views');

// Exists
$exists = Post::where('slug', 'my-post')->exists();
$doesntExist = Post::where('slug', 'my-post')->doesntExist();
```

### ৩. Eager Loading (N+1 Problem সমাধান):
```php
// ❌ N+1 Problem
$posts = Post::all();
foreach ($posts as $post) {
    echo $post->user->name; // Each iteration makes a query
}

// ✅ Eager Loading
$posts = Post::with('user')->get();
foreach ($posts as $post) {
    echo $post->user->name; // Only 2 queries total
}

// Multiple relationships
$posts = Post::with(['user', 'comments', 'category'])->get();

// Nested relationships
$posts = Post::with('comments.user')->get();

// Conditional eager loading
$posts = Post::with(['comments' => function ($query) {
    $query->where('approved', true)->orderBy('created_at', 'desc');
}])->get();

// Lazy eager loading
$posts = Post::all();
$posts->load('user', 'comments');
```

### ৪. Scopes:
```php
// Local Scopes
class Post extends Model
{
    public function scopePublished($query)
    {
        return $query->where('status', 'published');
    }

    public function scopeFeatured($query)
    {
        return $query->where('is_featured', true);
    }

    public function scopeByAuthor($query, $authorId)
    {
        return $query->where('user_id', $authorId);
    }

    public function scopeRecent($query, $days = 7)
    {
        return $query->where('created_at', '>=', now()->subDays($days));
    }
}

// Usage
$posts = Post::published()->get();
$posts = Post::published()->featured()->get();
$posts = Post::byAuthor(1)->recent(30)->get();

// Global Scopes
class PublishedScope implements Scope
{
    public function apply(Builder $builder, Model $model)
    {
        $builder->where('status', 'published');
    }
}

// Apply global scope
class Post extends Model
{
    protected static function booted()
    {
        static::addGlobalScope(new PublishedScope);
    }
}

// Remove global scope
$posts = Post::withoutGlobalScope(PublishedScope::class)->get();
```

---

## Mutators & Accessors

### ১. Accessors (Get Attributes):
```php
class User extends Model
{
    // Accessor for full name
    public function getFullNameAttribute()
    {
        return $this->first_name . ' ' . $this->last_name;
    }

    // Accessor for formatted date
    public function getFormattedCreatedAtAttribute()
    {
        return $this->created_at->format('M d, Y');
    }

    // Accessor for avatar URL
    public function getAvatarUrlAttribute()
    {
        return $this->avatar ? asset('storage/' . $this->avatar) : asset('images/default-avatar.png');
    }
}

// Usage
$user = User::find(1);
echo $user->full_name; // John Doe
echo $user->formatted_created_at; // Jan 15, 2024
echo $user->avatar_url; // Full avatar URL
```

### ২. Mutators (Set Attributes):
```php
class User extends Model
{
    // Mutator for password
    public function setPasswordAttribute($value)
    {
        $this->attributes['password'] = bcrypt($value);
    }

    // Mutator for name (capitalize)
    public function setNameAttribute($value)
    {
        $this->attributes['name'] = ucwords(strtolower($value));
    }

    // Mutator for slug
    public function setTitleAttribute($value)
    {
        $this->attributes['title'] = $value;
        $this->attributes['slug'] = Str::slug($value);
    }
}

// Usage
$user = new User();
$user->password = 'secret123'; // Automatically hashed
$user->name = 'john doe'; // Becomes 'John Doe'
$user->title = 'My Blog Post'; // Also sets slug to 'my-blog-post'
```

### ৩. Attribute Casting:
```php
class Post extends Model
{
    protected $casts = [
        'published_at' => 'datetime',
        'is_featured' => 'boolean',
        'meta_data' => 'array',
        'settings' => 'json',
        'price' => 'decimal:2',
        'tags' => 'collection'
    ];
}

// Usage
$post = Post::find(1);
$post->published_at; // Carbon instance
$post->is_featured; // Boolean
$post->meta_data; // Array
$post->settings; // Object/Array from JSON
$post->price; // Decimal with 2 places
$post->tags; // Collection instance
```

---

## Model Events

### ১. Model Events:
```php
class Post extends Model
{
    protected static function booted()
    {
        // Before creating
        static::creating(function ($post) {
            $post->slug = Str::slug($post->title);
            $post->user_id = auth()->id();
        });

        // After creating
        static::created(function ($post) {
            // Send notification
            Notification::send($post->user, new PostCreated($post));
        });

        // Before updating
        static::updating(function ($post) {
            if ($post->isDirty('title')) {
                $post->slug = Str::slug($post->title);
            }
        });

        // After updating
        static::updated(function ($post) {
            // Clear cache
            Cache::forget("post.{$post->id}");
        });

        // Before deleting
        static::deleting(function ($post) {
            // Delete related comments
            $post->comments()->delete();
        });

        // After deleting
        static::deleted(function ($post) {
            // Log deletion
            Log::info("Post deleted: {$post->title}");
        });
    }
}
```

### ২. Observer Pattern:
```bash
# Create Observer
php artisan make:observer PostObserver --model=Post
```

```php
<?php
// app/Observers/PostObserver.php

class PostObserver
{
    public function creating(Post $post)
    {
        $post->slug = Str::slug($post->title);
        $post->user_id = auth()->id();
    }

    public function created(Post $post)
    {
        // Send notifications
        event(new PostCreated($post));
    }

    public function updating(Post $post)
    {
        if ($post->isDirty('title')) {
            $post->slug = Str::slug($post->title);
        }
    }

    public function updated(Post $post)
    {
        Cache::forget("post.{$post->id}");
    }

    public function deleting(Post $post)
    {
        $post->comments()->delete();
    }

    public function deleted(Post $post)
    {
        Log::info("Post deleted: {$post->title}");
    }
}

// Register Observer
// app/Providers/EventServiceProvider.php
class EventServiceProvider extends ServiceProvider
{
    public function boot()
    {
        Post::observe(PostObserver::class);
    }
}
```

---

## Advanced Features

### ১. Soft Deletes:
```php
use Illuminate\Database\Eloquent\SoftDeletes;

class Post extends Model
{
    use SoftDeletes;

    protected $dates = ['deleted_at'];
}

// Usage
$post = Post::find(1);
$post->delete(); // Soft delete (sets deleted_at)

$posts = Post::all(); // Only non-deleted
$posts = Post::withTrashed()->get(); // Include soft deleted
$posts = Post::onlyTrashed()->get(); // Only soft deleted

$post->restore(); // Restore soft deleted
$post->forceDelete(); // Permanent delete
```

### ২. Model Factories:
```php
// database/factories/PostFactory.php
class PostFactory extends Factory
{
    public function definition()
    {
        return [
            'title' => $this->faker->sentence(),
            'content' => $this->faker->paragraphs(3, true),
            'status' => $this->faker->randomElement(['draft', 'published']),
            'user_id' => User::factory(),
        ];
    }

    public function published()
    {
        return $this->state(['status' => 'published']);
    }
}

// Usage
$post = Post::factory()->create();
$posts = Post::factory(10)->create();
$publishedPost = Post::factory()->published()->create();
```

### ৩. Custom Collections:
```php
// app/Collections/PostCollection.php
class PostCollection extends Collection
{
    public function published()
    {
        return $this->filter(function ($post) {
            return $post->status === 'published';
        });
    }

    public function byAuthor($authorId)
    {
        return $this->filter(function ($post) use ($authorId) {
            return $post->user_id === $authorId;
        });
    }
}

// Post Model
class Post extends Model
{
    public function newCollection(array $models = [])
    {
        return new PostCollection($models);
    }
}

// Usage
$posts = Post::all();
$publishedPosts = $posts->published();
$authorPosts = $posts->byAuthor(1);
```

---

## Best Practices

### 🎯 **১. Model Organization:**
```php
class Post extends Model
{
    use HasFactory, SoftDeletes;

    // 1. Constants
    const STATUS_DRAFT = 'draft';
    const STATUS_PUBLISHED = 'published';

    // 2. Properties
    protected $fillable = ['title', 'content', 'status', 'user_id'];
    protected $casts = ['published_at' => 'datetime'];

    // 3. Relationships
    public function user() { return $this->belongsTo(User::class); }
    public function comments() { return $this->hasMany(Comment::class); }

    // 4. Scopes
    public function scopePublished($query) { return $query->where('status', self::STATUS_PUBLISHED); }

    // 5. Accessors
    public function getExcerptAttribute() { return Str::limit($this->content, 100); }

    // 6. Mutators
    public function setTitleAttribute($value) { $this->attributes['title'] = ucfirst($value); }

    // 7. Methods
    public function isPublished() { return $this->status === self::STATUS_PUBLISHED; }
}
```

### 🚀 **২. Performance Tips:**
```php
// ✅ Use eager loading
$posts = Post::with('user', 'comments')->get();

// ✅ Use select to limit columns
$posts = Post::select('id', 'title', 'created_at')->get();

// ✅ Use pagination
$posts = Post::paginate(15);

// ✅ Use chunking for large datasets
Post::chunk(100, function ($posts) {
    foreach ($posts as $post) {
        // Process post
    }
});

// ✅ Use exists() instead of count()
if (Post::where('user_id', 1)->exists()) {
    // User has posts
}
```

### 🛡️ **৩. Security Best Practices:**
```php
// ✅ Always use fillable or guarded
protected $fillable = ['title', 'content', 'user_id'];

// ✅ Validate before mass assignment
$post = Post::create($request->validated());

// ✅ Use route model binding
Route::get('/posts/{post}', function (Post $post) {
    return $post;
});

// ✅ Hide sensitive attributes
protected $hidden = ['password', 'remember_token'];
```

---

## 📚 আরও জানতে:
- [Eloquent ORM](https://laravel.com/docs/eloquent)
- [Eloquent Relationships](https://laravel.com/docs/eloquent-relationships)
- [Eloquent Collections](https://laravel.com/docs/eloquent-collections)