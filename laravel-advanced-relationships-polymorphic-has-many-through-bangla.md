# Laravel Advanced Relationships - Polymorphic & Has-Many-Through - সম্পূর্ণ বাংলা গাইড

## 📋 সূচিপত্র
- [Polymorphic Relationships](#polymorphic-relationships)
- [Has-Many-Through Relationships](#has-many-through-relationships)
- [Advanced Polymorphic Patterns](#advanced-polymorphic-patterns)
- [Real-world Examples](#real-world-examples)
- [Performance Considerations](#performance-considerations)
- [Best Practices](#best-practices)

---

## Polymorphic Relationships

### 🎭 **Polymorphic Relationship কি?**
**Polymorphic Relationship** হলো এমন একটি relationship যেখানে **একটি model** **multiple different models** এর সাথে **same relationship** maintain করতে পারে।

### 📚 **সহজ উদাহরণ - Comment System:**

#### **Traditional Approach (❌ Not Scalable):**
```php
// ❌ Separate comment tables for each model
CREATE TABLE post_comments (
    id INT PRIMARY KEY,
    post_id INT,
    content TEXT,
    user_id INT
);

CREATE TABLE video_comments (
    id INT PRIMARY KEY,
    video_id INT,
    content TEXT,
    user_id INT
);

CREATE TABLE photo_comments (
    id INT PRIMARY KEY,
    photo_id INT,
    content TEXT,
    user_id INT
);

// Problem: Code duplication, maintenance nightmare
```

#### **Polymorphic Approach (✅ Scalable):**
```php
// ✅ Single comments table for all models
CREATE TABLE comments (
    id INT PRIMARY KEY,
    content TEXT,
    user_id INT,
    commentable_id INT,     -- Foreign key to any model
    commentable_type VARCHAR(255), -- Model class name
    created_at TIMESTAMP,
    updated_at TIMESTAMP
);

// Sample data
INSERT INTO comments VALUES
(1, 'Great post!', 1, 5, 'App\\Models\\Post', NOW(), NOW()),
(2, 'Nice video!', 2, 3, 'App\\Models\\Video', NOW(), NOW()),
(3, 'Beautiful photo!', 1, 8, 'App\\Models\\Photo', NOW(), NOW());
```

### 🏗️ **Polymorphic Implementation:**

#### **1. One-to-Many Polymorphic:**
```php
// Comment Model
class Comment extends Model
{
    protected $fillable = ['content', 'user_id', 'commentable_id', 'commentable_type'];
    
    // Polymorphic relationship
    public function commentable()
    {
        return $this->morphTo();
    }
    
    public function user()
    {
        return $this->belongsTo(User::class);
    }
}

// Post Model
class Post extends Model
{
    protected $fillable = ['title', 'content', 'user_id'];
    
    public function comments()
    {
        return $this->morphMany(Comment::class, 'commentable');
    }
}

// Video Model
class Video extends Model
{
    protected $fillable = ['title', 'url', 'user_id'];
    
    public function comments()
    {
        return $this->morphMany(Comment::class, 'commentable');
    }
}

// Photo Model
class Photo extends Model
{
    protected $fillable = ['title', 'url', 'user_id'];
    
    public function comments()
    {
        return $this->morphMany(Comment::class, 'commentable');
    }
}
```

#### **Usage Examples:**
```php
// Create comments for different models
$post = Post::find(1);
$post->comments()->create([
    'content' => 'Great post!',
    'user_id' => auth()->id()
]);

$video = Video::find(1);
$video->comments()->create([
    'content' => 'Amazing video!',
    'user_id' => auth()->id()
]);

// Retrieve comments
$post = Post::with('comments.user')->find(1);
foreach ($post->comments as $comment) {
    echo $comment->user->name . ': ' . $comment->content;
}

// Get the parent model from comment
$comment = Comment::find(1);
$parentModel = $comment->commentable; // Could be Post, Video, or Photo

if ($parentModel instanceof Post) {
    echo "Comment on post: " . $parentModel->title;
} elseif ($parentModel instanceof Video) {
    echo "Comment on video: " . $parentModel->title;
}
```

### 🏷️ **Real Example - Tagging System:**

```php
// Tag Model
class Tag extends Model
{
    protected $fillable = ['name', 'slug'];
    
    // Polymorphic many-to-many
    public function posts()
    {
        return $this->morphedByMany(Post::class, 'taggable');
    }
    
    public function videos()
    {
        return $this->morphedByMany(Video::class, 'taggable');
    }
    
    public function photos()
    {
        return $this->morphedByMany(Photo::class, 'taggable');
    }
}

// Post Model
class Post extends Model
{
    public function tags()
    {
        return $this->morphToMany(Tag::class, 'taggable');
    }
}

// Video Model
class Video extends Model
{
    public function tags()
    {
        return $this->morphToMany(Tag::class, 'taggable');
    }
}

// Migration for polymorphic many-to-many
Schema::create('taggables', function (Blueprint $table) {
    $table->id();
    $table->unsignedBigInteger('tag_id');
    $table->unsignedBigInteger('taggable_id');
    $table->string('taggable_type');
    $table->timestamps();
    
    $table->index(['taggable_id', 'taggable_type']);
});

// Usage
$post = Post::find(1);
$post->tags()->attach([1, 2, 3]); // Attach tags to post

$video = Video::find(1);
$video->tags()->attach([2, 4, 5]); // Attach tags to video

// Get all posts with specific tag
$tag = Tag::find(1);
$posts = $tag->posts; // All posts with this tag
$videos = $tag->videos; // All videos with this tag
```

### 💾 **Advanced Example - File Attachments:**

```php
// Attachment Model
class Attachment extends Model
{
    protected $fillable = [
        'filename', 'original_name', 'mime_type', 'size',
        'attachable_id', 'attachable_type'
    ];
    
    public function attachable()
    {
        return $this->morphTo();
    }
    
    // Helper methods
    public function getUrlAttribute()
    {
        return asset('storage/attachments/' . $this->filename);
    }
    
    public function isImage()
    {
        return str_starts_with($this->mime_type, 'image/');
    }
    
    public function getHumanSizeAttribute()
    {
        $bytes = $this->size;
        $units = ['B', 'KB', 'MB', 'GB'];
        
        for ($i = 0; $bytes > 1024; $i++) {
            $bytes /= 1024;
        }
        
        return round($bytes, 2) . ' ' . $units[$i];
    }
}

// Models that can have attachments
class Post extends Model
{
    public function attachments()
    {
        return $this->morphMany(Attachment::class, 'attachable');
    }
    
    public function images()
    {
        return $this->morphMany(Attachment::class, 'attachable')
                   ->where('mime_type', 'like', 'image/%');
    }
}

class Message extends Model
{
    public function attachments()
    {
        return $this->morphMany(Attachment::class, 'attachable');
    }
}

class Product extends Model
{
    public function attachments()
    {
        return $this->morphMany(Attachment::class, 'attachable');
    }
    
    public function gallery()
    {
        return $this->morphMany(Attachment::class, 'attachable')
                   ->where('mime_type', 'like', 'image/%')
                   ->orderBy('created_at');
    }
}

// Usage
$post = Post::find(1);
$post->attachments()->create([
    'filename' => 'unique-filename.jpg',
    'original_name' => 'my-photo.jpg',
    'mime_type' => 'image/jpeg',
    'size' => 1024000
]);

// Get all images for a post
$images = $post->images;
foreach ($images as $image) {
    echo '<img src="' . $image->url . '" alt="' . $image->original_name . '">';
}
```

---

## Has-Many-Through Relationships

### 🔗 **Has-Many-Through কি?**
**Has-Many-Through** relationship হলো **intermediate model** এর মাধ্যমে **distant relationship** establish করা।

### 🌍 **সহজ উদাহরণ - Country → Users → Posts:**

```php
// Database Structure
countries (id, name)
users (id, name, country_id)
posts (id, title, content, user_id)

// Relationship Chain: Country → Users → Posts
```

#### **Models:**
```php
// Country Model
class Country extends Model
{
    protected $fillable = ['name', 'code'];
    
    // Direct relationship
    public function users()
    {
        return $this->hasMany(User::class);
    }
    
    // Has-Many-Through relationship
    public function posts()
    {
        return $this->hasManyThrough(
            Post::class,    // Final model
            User::class,    // Intermediate model
            'country_id',   // Foreign key on users table
            'user_id',      // Foreign key on posts table
            'id',           // Local key on countries table
            'id'            // Local key on users table
        );
    }
    
    // Get all comments through users and posts
    public function comments()
    {
        return $this->hasManyThrough(
            Comment::class,
            User::class,
            'country_id', // Foreign key on users table
            'user_id',    // Foreign key on comments table
            'id',         // Local key on countries table
            'id'          // Local key on users table
        );
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

// Post Model
class Post extends Model
{
    public function user()
    {
        return $this->belongsTo(User::class);
    }
    
    public function country()
    {
        // Accessing country through user
        return $this->hasOneThrough(
            Country::class,
            User::class,
            'id',         // Foreign key on users table
            'id',         // Foreign key on countries table
            'user_id',    // Local key on posts table
            'country_id'  // Local key on users table
        );
    }
}
```

#### **Usage Examples:**
```php
// Get all posts from a specific country
$country = Country::find(1); // Bangladesh
$posts = $country->posts; // All posts by Bangladeshi users

echo "Posts from {$country->name}: " . $posts->count();

// Get posts with user information
$country = Country::with('posts.user')->find(1);
foreach ($country->posts as $post) {
    echo $post->title . ' by ' . $post->user->name;
}

// Get country statistics
$country = Country::withCount(['users', 'posts'])->find(1);
echo "Users: {$country->users_count}, Posts: {$country->posts_count}";
```

### 🏢 **Complex Example - Company → Departments → Employees → Projects:**

```php
// Database Structure
companies (id, name)
departments (id, name, company_id)
employees (id, name, department_id)
projects (id, name, employee_id)

// Company Model
class Company extends Model
{
    public function departments()
    {
        return $this->hasMany(Department::class);
    }
    
    // Employees through departments
    public function employees()
    {
        return $this->hasManyThrough(
            Employee::class,
            Department::class
        );
    }
    
    // Projects through departments and employees
    public function projects()
    {
        return $this->hasManyThrough(
            Project::class,
            Employee::class,
            'department_id', // Foreign key on employees table
            'employee_id',   // Foreign key on projects table
            'id',            // Local key on companies table
            'id'             // Local key on employees table
        )->join('departments', 'employees.department_id', '=', 'departments.id')
         ->where('departments.company_id', $this->id);
    }
}

// Department Model
class Department extends Model
{
    public function company()
    {
        return $this->belongsTo(Company::class);
    }
    
    public function employees()
    {
        return $this->hasMany(Employee::class);
    }
    
    public function projects()
    {
        return $this->hasManyThrough(
            Project::class,
            Employee::class
        );
    }
}

// Employee Model
class Employee extends Model
{
    public function department()
    {
        return $this->belongsTo(Department::class);
    }
    
    public function company()
    {
        return $this->hasOneThrough(
            Company::class,
            Department::class,
            'id',            // Foreign key on departments table
            'id',            // Foreign key on companies table
            'department_id', // Local key on employees table
            'company_id'     // Local key on departments table
        );
    }
    
    public function projects()
    {
        return $this->hasMany(Project::class);
    }
}

// Usage
$company = Company::find(1);

// Get all employees across all departments
$employees = $company->employees;

// Get all projects across all departments and employees
$projects = $company->projects;

// Get company statistics
$company = Company::withCount(['departments', 'employees', 'projects'])->find(1);
echo "Departments: {$company->departments_count}";
echo "Employees: {$company->employees_count}";
echo "Projects: {$company->projects_count}";
```

---

## Advanced Polymorphic Patterns

### 🎨 **Polymorphic Many-to-Many with Pivot Data:**

```php
// Activity Log System
class Activity extends Model
{
    protected $fillable = ['name', 'description'];
    
    public function posts()
    {
        return $this->morphedByMany(Post::class, 'activitable')
                   ->withPivot(['performed_at', 'metadata'])
                   ->withTimestamps();
    }
    
    public function users()
    {
        return $this->morphedByMany(User::class, 'activitable')
                   ->withPivot(['performed_at', 'metadata']);
    }
}

class Post extends Model
{
    public function activities()
    {
        return $this->morphToMany(Activity::class, 'activitable')
                   ->withPivot(['performed_at', 'metadata'])
                   ->withTimestamps();
    }
}

// Migration
Schema::create('activitables', function (Blueprint $table) {
    $table->id();
    $table->unsignedBigInteger('activity_id');
    $table->unsignedBigInteger('activitable_id');
    $table->string('activitable_type');
    $table->timestamp('performed_at');
    $table->json('metadata')->nullable();
    $table->timestamps();
});

// Usage
$post = Post::find(1);
$activity = Activity::find(1); // 'viewed'

$post->activities()->attach($activity->id, [
    'performed_at' => now(),
    'metadata' => json_encode(['ip' => request()->ip(), 'user_agent' => request()->userAgent()])
]);

// Retrieve with pivot data
$post = Post::with('activities')->find(1);
foreach ($post->activities as $activity) {
    echo $activity->name . ' at ' . $activity->pivot->performed_at;
    echo ' Metadata: ' . $activity->pivot->metadata;
}
```

### 🔄 **Self-Referencing Polymorphic:**

```php
// Notification System
class Notification extends Model
{
    protected $fillable = ['type', 'title', 'message', 'notifiable_id', 'notifiable_type'];
    
    public function notifiable()
    {
        return $this->morphTo();
    }
    
    // Notifications can have related notifications
    public function related()
    {
        return $this->morphMany(Notification::class, 'notifiable');
    }
}

class User extends Model
{
    public function notifications()
    {
        return $this->morphMany(Notification::class, 'notifiable');
    }
}

class Order extends Model
{
    public function notifications()
    {
        return $this->morphMany(Notification::class, 'notifiable');
    }
}

// Usage
$user = User::find(1);
$notification = $user->notifications()->create([
    'type' => 'order_shipped',
    'title' => 'Order Shipped',
    'message' => 'Your order has been shipped'
]);

// Create follow-up notification
$notification->related()->create([
    'type' => 'delivery_update',
    'title' => 'Delivery Update',
    'message' => 'Your package is out for delivery'
]);
```

---

## Real-world Examples

### 🛒 **E-commerce System:**

```php
// Product Review System (Polymorphic)
class Review extends Model
{
    protected $fillable = ['rating', 'comment', 'user_id', 'reviewable_id', 'reviewable_type'];
    
    public function reviewable()
    {
        return $this->morphTo();
    }
    
    public function user()
    {
        return $this->belongsTo(User::class);
    }
}

class Product extends Model
{
    public function reviews()
    {
        return $this->morphMany(Review::class, 'reviewable');
    }
    
    public function averageRating()
    {
        return $this->reviews()->avg('rating');
    }
}

class Service extends Model
{
    public function reviews()
    {
        return $this->morphMany(Review::class, 'reviewable');
    }
}

// Category → Products → Orders (Has-Many-Through)
class Category extends Model
{
    public function products()
    {
        return $this->hasMany(Product::class);
    }
    
    // Get all orders for products in this category
    public function orders()
    {
        return $this->hasManyThrough(
            Order::class,
            Product::class,
            'category_id',
            'product_id',
            'id',
            'id'
        )->join('order_items', 'orders.id', '=', 'order_items.order_id');
    }
    
    // Get total sales for this category
    public function totalSales()
    {
        return $this->orders()
                   ->join('order_items', 'orders.id', '=', 'order_items.order_id')
                   ->where('order_items.product_id', 'products.id')
                   ->sum('order_items.total');
    }
}

// Usage
$category = Category::find(1); // Electronics
$totalSales = $category->totalSales();
$orderCount = $category->orders()->count();

echo "Category: {$category->name}";
echo "Total Sales: \${$totalSales}";
echo "Total Orders: {$orderCount}";
```

### 📱 **Social Media Platform:**

```php
// Likeable System (Polymorphic)
class Like extends Model
{
    protected $fillable = ['user_id', 'likeable_id', 'likeable_type'];
    
    public function likeable()
    {
        return $this->morphTo();
    }
    
    public function user()
    {
        return $this->belongsTo(User::class);
    }
}

class Post extends Model
{
    public function likes()
    {
        return $this->morphMany(Like::class, 'likeable');
    }
    
    public function isLikedBy(User $user)
    {
        return $this->likes()->where('user_id', $user->id)->exists();
    }
}

class Comment extends Model
{
    public function likes()
    {
        return $this->morphMany(Like::class, 'likeable');
    }
}

// Group → Users → Posts (Has-Many-Through)
class Group extends Model
{
    public function users()
    {
        return $this->belongsToMany(User::class, 'group_members');
    }
    
    public function posts()
    {
        return $this->hasManyThrough(
            Post::class,
            User::class,
            'group_id',
            'user_id',
            'id',
            'id'
        )->join('group_members', 'users.id', '=', 'group_members.user_id')
         ->where('group_members.group_id', $this->id);
    }
}

// Usage
$group = Group::find(1);
$posts = $group->posts()->latest()->take(10)->get();

foreach ($posts as $post) {
    echo $post->title . ' - Likes: ' . $post->likes()->count();
}
```

---

## Performance Considerations

### ⚡ **Eager Loading:**

```php
// ❌ N+1 Problem
$posts = Post::all();
foreach ($posts as $post) {
    echo $post->comments->count(); // N+1 queries
}

// ✅ Eager Loading
$posts = Post::with('comments')->get();
foreach ($posts as $post) {
    echo $post->comments->count(); // 2 queries total
}

// ✅ Polymorphic Eager Loading
$comments = Comment::with('commentable')->get();
foreach ($comments as $comment) {
    echo get_class($comment->commentable) . ': ' . $comment->commentable->title;
}

// ✅ Constrained Eager Loading
$posts = Post::with(['comments' => function ($query) {
    $query->where('approved', true)->latest();
}])->get();
```

### 🔍 **Indexing for Polymorphic:**

```php
// Migration with proper indexes
Schema::create('comments', function (Blueprint $table) {
    $table->id();
    $table->text('content');
    $table->unsignedBigInteger('user_id');
    $table->unsignedBigInteger('commentable_id');
    $table->string('commentable_type');
    $table->timestamps();
    
    // Composite index for polymorphic relationship
    $table->index(['commentable_id', 'commentable_type']);
    $table->index('user_id');
});
```

### 📊 **Query Optimization:**

```php
// ✅ Efficient polymorphic queries
$posts = Post::whereHas('comments', function ($query) {
    $query->where('approved', true);
})->withCount(['comments' => function ($query) {
    $query->where('approved', true);
}])->get();

// ✅ Efficient has-many-through with conditions
$country = Country::withCount([
    'posts' => function ($query) {
        $query->where('published', true);
    }
])->find(1);
```

---

## Best Practices

### ✅ **When to Use Polymorphic:**

1. **Multiple models need same relationship**
   - Comments on Posts, Videos, Photos
   - Likes on Posts, Comments, Reviews
   - File attachments on Messages, Posts, Products

2. **Avoid code duplication**
   - Same functionality across different models
   - Consistent behavior patterns

3. **Flexible system design**
   - Easy to add new models to existing relationships
   - Scalable architecture

### ✅ **When to Use Has-Many-Through:**

1. **Accessing distant relationships**
   - Country → Users → Posts
   - Company → Departments → Employees

2. **Aggregating data across relationships**
   - Total sales by category through products
   - User activity across different levels

3. **Reporting and analytics**
   - Cross-table statistics
   - Hierarchical data analysis

### ⚠️ **Common Pitfalls:**

```php
// ❌ Don't use polymorphic for simple relationships
class User extends Model
{
    // Bad: Only User has Profile
    public function profile()
    {
        return $this->morphOne(Profile::class, 'profilable');
    }
}

// ✅ Use simple relationship instead
class User extends Model
{
    public function profile()
    {
        return $this->hasOne(Profile::class);
    }
}

// ❌ Don't over-complicate has-many-through
// If you need complex joins, consider using Query Builder

// ✅ Keep relationships logical and maintainable
```

### 🎯 **Decision Matrix:**

| Scenario | Use Polymorphic | Use Has-Many-Through | Use Simple Relationship |
|----------|----------------|---------------------|------------------------|
| Comments on multiple models | ✅ | ❌ | ❌ |
| Country → Users → Posts | ❌ | ✅ | ❌ |
| User → Profile | ❌ | ❌ | ✅ |
| Tags on multiple models | ✅ | ❌ | ❌ |
| Company → Employees | ❌ | ❌ | ✅ |

Advanced relationships সঠিকভাবে ব্যবহার করলে আপনার Laravel application **flexible, scalable এবং maintainable** হবে।