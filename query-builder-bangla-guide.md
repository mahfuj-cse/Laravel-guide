# 8️⃣ Laravel Query Builder - বিস্তারিত বাংলা গাইড

## 📋 সূচিপত্র
- [Query Builder কি?](#query-builder-কি)
- [Basic Queries](#basic-queries)
- [Where Clauses](#where-clauses)
- [Joins](#joins)
- [Aggregations](#aggregations)
- [Grouping & Ordering](#grouping--ordering)
- [Raw Expressions](#raw-expressions)
- [Advanced Techniques](#advanced-techniques)
- [Performance Tips](#performance-tips)

---

## Query Builder কি?

**Query Builder** হলো Laravel এর **Fluent Interface** যা **SQL queries** তৈরি করার জন্য **PHP methods** ব্যবহার করে।

### 🔥 Query Builder এর সুবিধা:
- ✅ **Fluent Syntax** - Readable এবং chainable methods
- ✅ **Database Agnostic** - সব database এ কাজ করে
- ✅ **SQL Injection Protection** - Automatic parameter binding
- ✅ **Raw SQL Support** - Complex queries এর জন্য
- ✅ **Performance** - Optimized query generation

### Raw SQL vs Query Builder:
```php
// Raw SQL (কঠিন এবং unsafe)
$users = DB::select("SELECT * FROM users WHERE age > ? AND city = ?", [25, 'Dhaka']);

// Query Builder (সহজ এবং safe)
$users = DB::table('users')->where('age', '>', 25)->where('city', 'Dhaka')->get();
```

---

## Basic Queries

### ১. Database Connection:
```php
use Illuminate\Support\Facades\DB;

// Default connection
$users = DB::table('users')->get();

// Specific connection
$users = DB::connection('mysql2')->table('users')->get();
```

### ২. Select Queries:
```php
// Get all records
$users = DB::table('users')->get();

// Get single record
$user = DB::table('users')->where('id', 1)->first();

// Get single value
$email = DB::table('users')->where('id', 1)->value('email');

// Get specific columns
$users = DB::table('users')->select('name', 'email')->get();

// Get distinct records
$cities = DB::table('users')->distinct()->pluck('city');

// Check if record exists
$exists = DB::table('users')->where('email', 'john@example.com')->exists();
```

### ৩. Insert Queries:
```php
// Single insert
DB::table('users')->insert([
    'name' => 'John Doe',
    'email' => 'john@example.com',
    'password' => bcrypt('password')
]);

// Multiple insert
DB::table('users')->insert([
    ['name' => 'John', 'email' => 'john@example.com'],
    ['name' => 'Jane', 'email' => 'jane@example.com'],
    ['name' => 'Bob', 'email' => 'bob@example.com']
]);

// Insert and get ID
$id = DB::table('users')->insertGetId([
    'name' => 'John Doe',
    'email' => 'john@example.com'
]);

// Insert or ignore (MySQL)
DB::table('users')->insertOrIgnore([
    'name' => 'John Doe',
    'email' => 'john@example.com'
]);
```

### ৪. Update Queries:
```php
// Update records
DB::table('users')
    ->where('id', 1)
    ->update(['name' => 'John Smith']);

// Update multiple conditions
DB::table('users')
    ->where('active', false)
    ->where('last_login', '<', now()->subDays(30))
    ->update(['status' => 'inactive']);

// Increment/Decrement
DB::table('posts')->where('id', 1)->increment('views');
DB::table('posts')->where('id', 1)->increment('views', 5);
DB::table('users')->where('id', 1)->decrement('credits', 10);

// Update or insert (upsert)
DB::table('users')->updateOrInsert(
    ['email' => 'john@example.com'],
    ['name' => 'John Doe', 'updated_at' => now()]
);
```

### ৫. Delete Queries:
```php
// Delete records
DB::table('users')->where('id', 1)->delete();

// Delete multiple
DB::table('users')->where('active', false)->delete();

// Truncate table
DB::table('users')->truncate();
```

---

## Where Clauses

### ১. Basic Where Clauses:
```php
// Simple where
$users = DB::table('users')->where('name', 'John')->get();

// Where with operator
$users = DB::table('users')->where('age', '>', 25)->get();

// Multiple where (AND)
$users = DB::table('users')
    ->where('name', 'John')
    ->where('age', '>', 25)
    ->get();

// Where array (AND)
$users = DB::table('users')->where([
    ['name', '=', 'John'],
    ['age', '>', 25]
])->get();
```

### ২. Or Where Clauses:
```php
// Or where
$users = DB::table('users')
    ->where('name', 'John')
    ->orWhere('name', 'Jane')
    ->get();

// Where with closure (grouping)
$users = DB::table('users')
    ->where('name', 'John')
    ->orWhere(function ($query) {
        $query->where('age', '>', 25)
              ->where('city', 'Dhaka');
    })
    ->get();
```

### ৩. Advanced Where Clauses:
```php
// Where In
$users = DB::table('users')->whereIn('id', [1, 2, 3])->get();

// Where Not In
$users = DB::table('users')->whereNotIn('status', ['banned', 'inactive'])->get();

// Where Null
$users = DB::table('users')->whereNull('email_verified_at')->get();

// Where Not Null
$users = DB::table('users')->whereNotNull('email_verified_at')->get();

// Where Between
$users = DB::table('users')->whereBetween('age', [18, 65])->get();

// Where Not Between
$users = DB::table('users')->whereNotBetween('age', [13, 17])->get();

// Where Date
$users = DB::table('users')->whereDate('created_at', '2024-01-15')->get();

// Where Month/Year
$users = DB::table('users')->whereMonth('created_at', 12)->get();
$users = DB::table('users')->whereYear('created_at', 2024)->get();

// Where Time
$users = DB::table('users')->whereTime('created_at', '>', '10:00:00')->get();

// Where Like
$users = DB::table('users')->where('name', 'like', '%john%')->get();

// Where JSON (MySQL/PostgreSQL)
$users = DB::table('users')->where('preferences->language', 'en')->get();
```

### ৪. Conditional Where:
```php
// When condition
$users = DB::table('users')
    ->when($request->has('name'), function ($query) use ($request) {
        return $query->where('name', 'like', '%' . $request->name . '%');
    })
    ->when($request->has('city'), function ($query) use ($request) {
        return $query->where('city', $request->city);
    })
    ->get();

// Unless condition
$users = DB::table('users')
    ->unless($request->show_all, function ($query) {
        return $query->where('active', true);
    })
    ->get();
```

---

## Joins

### ১. Inner Join:
```php
// Basic inner join
$users = DB::table('users')
    ->join('posts', 'users.id', '=', 'posts.user_id')
    ->select('users.*', 'posts.title')
    ->get();

// Multiple joins
$posts = DB::table('posts')
    ->join('users', 'posts.user_id', '=', 'users.id')
    ->join('categories', 'posts.category_id', '=', 'categories.id')
    ->select('posts.*', 'users.name as author', 'categories.name as category')
    ->get();
```

### ২. Left/Right Join:
```php
// Left join
$users = DB::table('users')
    ->leftJoin('posts', 'users.id', '=', 'posts.user_id')
    ->select('users.*', 'posts.title')
    ->get();

// Right join
$posts = DB::table('posts')
    ->rightJoin('users', 'posts.user_id', '=', 'users.id')
    ->select('posts.*', 'users.name')
    ->get();
```

### ৩. Cross Join:
```php
// Cross join
$combinations = DB::table('colors')
    ->crossJoin('sizes')
    ->get();
```

### ৪. Advanced Joins:
```php
// Join with multiple conditions
$users = DB::table('users')
    ->join('posts', function ($join) {
        $join->on('users.id', '=', 'posts.user_id')
             ->where('posts.status', '=', 'published');
    })
    ->get();

// Join with OR condition
$users = DB::table('users')
    ->join('contacts', function ($join) {
        $join->on('users.id', '=', 'contacts.user_id')
             ->orOn('users.id', '=', 'contacts.proxy_user_id');
    })
    ->get();

// Sub-query join
$latestPosts = DB::table('posts')
    ->select('user_id', DB::raw('MAX(created_at) as last_post_created_at'))
    ->groupBy('user_id');

$users = DB::table('users')
    ->joinSub($latestPosts, 'latest_posts', function ($join) {
        $join->on('users.id', '=', 'latest_posts.user_id');
    })
    ->get();
```

---

## Aggregations

### ১. Basic Aggregations:
```php
// Count
$userCount = DB::table('users')->count();
$activeUsers = DB::table('users')->where('active', true)->count();

// Sum
$totalViews = DB::table('posts')->sum('views');
$userTotalViews = DB::table('posts')->where('user_id', 1)->sum('views');

// Average
$avgAge = DB::table('users')->avg('age');
$avgPostViews = DB::table('posts')->avg('views');

// Min/Max
$minAge = DB::table('users')->min('age');
$maxViews = DB::table('posts')->max('views');

// Multiple aggregations
$stats = DB::table('posts')->selectRaw('
    COUNT(*) as total_posts,
    SUM(views) as total_views,
    AVG(views) as avg_views,
    MAX(views) as max_views
')->first();
```

### ২. Conditional Aggregations:
```php
// Count with condition
$publishedPosts = DB::table('posts')
    ->where('status', 'published')
    ->count();

// Sum with grouping
$viewsByUser = DB::table('posts')
    ->select('user_id', DB::raw('SUM(views) as total_views'))
    ->groupBy('user_id')
    ->get();

// Count distinct
$uniqueAuthors = DB::table('posts')->distinct('user_id')->count('user_id');
```

### ৩. Advanced Aggregations:
```php
// Conditional count using CASE
$stats = DB::table('posts')
    ->selectRaw('
        COUNT(*) as total,
        COUNT(CASE WHEN status = "published" THEN 1 END) as published,
        COUNT(CASE WHEN status = "draft" THEN 1 END) as draft
    ')
    ->first();

// Percentage calculation
$userStats = DB::table('users')
    ->selectRaw('
        COUNT(*) as total_users,
        COUNT(CASE WHEN active = 1 THEN 1 END) as active_users,
        ROUND(COUNT(CASE WHEN active = 1 THEN 1 END) * 100.0 / COUNT(*), 2) as active_percentage
    ')
    ->first();
```

---

## Grouping & Ordering

### ১. Group By:
```php
// Basic grouping
$postsByUser = DB::table('posts')
    ->select('user_id', DB::raw('COUNT(*) as post_count'))
    ->groupBy('user_id')
    ->get();

// Multiple grouping
$postsByUserAndStatus = DB::table('posts')
    ->select('user_id', 'status', DB::raw('COUNT(*) as count'))
    ->groupBy('user_id', 'status')
    ->get();

// Group by date
$postsByDate = DB::table('posts')
    ->select(DB::raw('DATE(created_at) as date'), DB::raw('COUNT(*) as count'))
    ->groupBy(DB::raw('DATE(created_at)'))
    ->get();
```

### ২. Having Clause:
```php
// Having with aggregation
$activeUsers = DB::table('posts')
    ->select('user_id', DB::raw('COUNT(*) as post_count'))
    ->groupBy('user_id')
    ->having('post_count', '>', 5)
    ->get();

// Having with multiple conditions
$topAuthors = DB::table('posts')
    ->select('user_id', DB::raw('COUNT(*) as posts'), DB::raw('SUM(views) as total_views'))
    ->groupBy('user_id')
    ->having('posts', '>', 10)
    ->having('total_views', '>', 1000)
    ->get();
```

### ৩. Order By:
```php
// Basic ordering
$users = DB::table('users')->orderBy('name')->get();
$users = DB::table('users')->orderBy('created_at', 'desc')->get();

// Multiple ordering
$posts = DB::table('posts')
    ->orderBy('status')
    ->orderBy('created_at', 'desc')
    ->get();

// Order by raw expression
$posts = DB::table('posts')
    ->orderByRaw('FIELD(status, "featured", "published", "draft")')
    ->get();

// Random ordering
$randomPosts = DB::table('posts')->inRandomOrder()->take(5)->get();

// Latest/Oldest
$latestPosts = DB::table('posts')->latest()->get(); // ORDER BY created_at DESC
$oldestPosts = DB::table('posts')->oldest()->get(); // ORDER BY created_at ASC
```

### ৪. Limit & Offset:
```php
// Limit
$posts = DB::table('posts')->take(10)->get();
$posts = DB::table('posts')->limit(10)->get();

// Offset
$posts = DB::table('posts')->skip(20)->take(10)->get();
$posts = DB::table('posts')->offset(20)->limit(10)->get();

// Pagination
$posts = DB::table('posts')->paginate(15);
$posts = DB::table('posts')->simplePaginate(15);
```

---

## Raw Expressions

### ১. Raw Select:
```php
// Raw select
$users = DB::table('users')
    ->select(DB::raw('COUNT(*) as user_count, status'))
    ->where('status', '<>', 1)
    ->groupBy('status')
    ->get();

// Select raw with bindings
$posts = DB::table('posts')
    ->selectRaw('title, content, SUBSTRING(content, 1, ?) as excerpt', [100])
    ->get();
```

### ২. Raw Where:
```php
// Where raw
$users = DB::table('users')
    ->whereRaw('age > ? AND city = ?', [25, 'Dhaka'])
    ->get();

// Or where raw
$posts = DB::table('posts')
    ->where('status', 'published')
    ->orWhereRaw('views > (SELECT AVG(views) FROM posts)')
    ->get();
```

### ৩. Raw Order By:
```php
// Order by raw
$posts = DB::table('posts')
    ->orderByRaw('CASE WHEN status = "featured" THEN 1 ELSE 2 END')
    ->orderBy('created_at', 'desc')
    ->get();

// Order by custom function
$users = DB::table('users')
    ->orderByRaw('CHAR_LENGTH(name) DESC')
    ->get();
```

### ৪. Raw Expressions in Aggregations:
```php
// Complex aggregation
$stats = DB::table('orders')
    ->selectRaw('
        DATE(created_at) as date,
        COUNT(*) as total_orders,
        SUM(CASE WHEN status = "completed" THEN amount ELSE 0 END) as completed_amount,
        AVG(amount) as avg_order_value
    ')
    ->groupBy(DB::raw('DATE(created_at)'))
    ->get();
```

---

## Advanced Techniques

### ১. Subqueries:
```php
// Subquery in select
$users = DB::table('users')
    ->select('name', 'email')
    ->selectSub(function ($query) {
        $query->from('posts')
              ->whereColumn('user_id', 'users.id')
              ->selectRaw('COUNT(*)');
    }, 'post_count')
    ->get();

// Subquery in where
$users = DB::table('users')
    ->where('id', function ($query) {
        $query->select('user_id')
              ->from('posts')
              ->groupBy('user_id')
              ->havingRaw('COUNT(*) > 10');
    })
    ->get();

// Exists subquery
$users = DB::table('users')
    ->whereExists(function ($query) {
        $query->select(DB::raw(1))
              ->from('posts')
              ->whereColumn('posts.user_id', 'users.id');
    })
    ->get();
```

### ২. Union Queries:
```php
// Union
$first = DB::table('users')->where('active', true);
$users = DB::table('admins')
    ->where('active', true)
    ->union($first)
    ->get();

// Union all
$first = DB::table('posts')->select('title', 'created_at')->where('status', 'published');
$content = DB::table('pages')
    ->select('title', 'created_at')
    ->where('status', 'published')
    ->unionAll($first)
    ->orderBy('created_at', 'desc')
    ->get();
```

### ৩. Window Functions (MySQL 8.0+):
```php
// Row number
$rankedPosts = DB::table('posts')
    ->select('*', DB::raw('ROW_NUMBER() OVER (PARTITION BY user_id ORDER BY views DESC) as rank'))
    ->get();

// Running total
$salesData = DB::table('sales')
    ->select('*', DB::raw('SUM(amount) OVER (ORDER BY date) as running_total'))
    ->orderBy('date')
    ->get();
```

### ৪. Common Table Expressions (CTE):
```php
// Recursive CTE for hierarchical data
$categories = DB::select("
    WITH RECURSIVE category_tree AS (
        SELECT id, name, parent_id, 0 as level
        FROM categories 
        WHERE parent_id IS NULL
        
        UNION ALL
        
        SELECT c.id, c.name, c.parent_id, ct.level + 1
        FROM categories c
        INNER JOIN category_tree ct ON c.parent_id = ct.id
    )
    SELECT * FROM category_tree ORDER BY level, name
");
```

### ৫. Transactions:
```php
// Basic transaction
DB::transaction(function () {
    DB::table('users')->update(['active' => false]);
    DB::table('posts')->where('user_id', 1)->delete();
});

// Manual transaction
DB::beginTransaction();
try {
    DB::table('accounts')->where('id', 1)->decrement('balance', 100);
    DB::table('accounts')->where('id', 2)->increment('balance', 100);
    DB::commit();
} catch (\Exception $e) {
    DB::rollback();
    throw $e;
}
```

---

## Performance Tips

### 🚀 **১. Query Optimization:**
```php
// ✅ Use specific columns instead of *
$users = DB::table('users')->select('id', 'name', 'email')->get();

// ✅ Use indexes for where clauses
$users = DB::table('users')->where('email', 'john@example.com')->first();

// ✅ Use limit for large datasets
$posts = DB::table('posts')->orderBy('created_at', 'desc')->limit(10)->get();

// ✅ Use exists() instead of count() > 0
$hasUsers = DB::table('users')->where('active', true)->exists();
```

### 🎯 **২. Chunking for Large Data:**
```php
// Process large datasets in chunks
DB::table('users')->orderBy('id')->chunk(100, function ($users) {
    foreach ($users as $user) {
        // Process each user
        $this->processUser($user);
    }
});

// Chunk by ID (more efficient)
DB::table('users')->chunkById(100, function ($users) {
    foreach ($users as $user) {
        // Process user
    }
});
```

### 📊 **৩. Query Debugging:**
```php
// Enable query log
DB::enableQueryLog();

// Your queries here
$users = DB::table('users')->where('active', true)->get();

// Get executed queries
$queries = DB::getQueryLog();
dd($queries);

// Debug single query
$users = DB::table('users')->where('active', true)->toSql();
echo $users; // SELECT * FROM users WHERE active = ?
```

### 🔧 **৪. Connection Management:**
```php
// Use read/write connections
$users = DB::connection('read')->table('users')->get();
DB::connection('write')->table('users')->insert($data);

// Connection pooling
DB::purge('mysql'); // Close connection
```

### 📈 **৫. Caching Strategies:**
```php
// Cache expensive queries
$popularPosts = Cache::remember('popular_posts', 3600, function () {
    return DB::table('posts')
        ->select('id', 'title', 'views')
        ->where('status', 'published')
        ->orderBy('views', 'desc')
        ->limit(10)
        ->get();
});

// Cache with tags
Cache::tags(['posts', 'popular'])->put('popular_posts', $posts, 3600);
```

---

## 🎯 Best Practices:

### ✅ **Security:**
- সবসময় parameter binding ব্যবহার করুন
- Raw queries এ user input validate করুন
- SQL injection থেকে বাঁচতে whereRaw এ bindings ব্যবহার করুন

### ✅ **Performance:**
- Specific columns select করুন
- Proper indexing ব্যবহার করুন
- Large datasets এর জন্য chunking করুন
- Query caching implement করুন

### ✅ **Maintainability:**
- Complex queries কে methods এ wrap করুন
- Query scopes ব্যবহার করুন
- Raw SQL minimize করুন

---

## 📚 আরও জানতে:
- [Laravel Query Builder](https://laravel.com/docs/queries)
- [Database Transactions](https://laravel.com/docs/database#database-transactions)
- [Query Builder Performance](https://laravel.com/docs/queries#debugging)