# 1️⃣2️⃣ Laravel Collections - বিস্তারিত বাংলা গাইড

## 📋 সূচিপত্র
- [Collections কি?](#collections-কি)
- [Collection তৈরি করা](#collection-তৈরি-করা)
- [Basic Methods](#basic-methods)
- [Filtering Methods](#filtering-methods)
- [Transformation Methods](#transformation-methods)
- [Aggregation Methods](#aggregation-methods)
- [Grouping & Sorting](#grouping--sorting)
- [Advanced Techniques](#advanced-techniques)

---

## Collections কি?

**Laravel Collections** হলো **Array এর Wrapper** যা **Powerful Methods** দিয়ে **Data Manipulation** করার সুবিধা দেয়।

### 🔥 Collections এর সুবিধা:
- ✅ **Fluent Interface** - Method chaining
- ✅ **Immutable** - Original data unchanged
- ✅ **Powerful Methods** - 100+ built-in methods
- ✅ **Lazy Evaluation** - Performance optimization
- ✅ **Type Safety** - Better than raw arrays

### Array vs Collection:
```php
// Traditional Array (কঠিন)
$users = [
    ['name' => 'John', 'age' => 25, 'city' => 'Dhaka'],
    ['name' => 'Jane', 'age' => 30, 'city' => 'Chittagong'],
    ['name' => 'Bob', 'age' => 35, 'city' => 'Dhaka']
];

$dhakaUsers = [];
foreach ($users as $user) {
    if ($user['city'] === 'Dhaka') {
        $dhakaUsers[] = $user['name'];
    }
}

// Laravel Collection (সহজ)
$users = collect([
    ['name' => 'John', 'age' => 25, 'city' => 'Dhaka'],
    ['name' => 'Jane', 'age' => 30, 'city' => 'Chittagong'],
    ['name' => 'Bob', 'age' => 35, 'city' => 'Dhaka']
]);

$dhakaUsers = $users->where('city', 'Dhaka')->pluck('name');
```

---

## Collection তৈরি করা

### ১. Collection তৈরির উপায়:
```php
// collect() helper function
$collection = collect([1, 2, 3, 4, 5]);

// Collection class
$collection = new \Illuminate\Support\Collection([1, 2, 3, 4, 5]);

// Empty collection
$collection = collect();

// From array
$array = ['a', 'b', 'c'];
$collection = collect($array);

// From Eloquent
$users = User::all(); // Already a collection
$posts = Post::where('published', true)->get(); // Collection

// From range
$numbers = collect(range(1, 10));

// From string
$chars = collect(str_split('Laravel'));
```

### ২. Collection Types:
```php
// Regular Collection
$collection = collect(['apple', 'banana', 'orange']);

// Eloquent Collection (Model instances)
$users = User::all(); // Illuminate\Database\Eloquent\Collection

// Lazy Collection (Memory efficient)
$lazyCollection = LazyCollection::make(function () {
    yield 1;
    yield 2;
    yield 3;
});
```

---

## Basic Methods

### ১. Basic Operations:
```php
$collection = collect([1, 2, 3, 4, 5]);

// Count
$count = $collection->count(); // 5

// Check if empty
$isEmpty = $collection->isEmpty(); // false
$isNotEmpty = $collection->isNotEmpty(); // true

// First & Last
$first = $collection->first(); // 1
$last = $collection->last(); // 5

// Get by index
$second = $collection->get(1); // 2
$default = $collection->get(10, 'default'); // 'default'

// Check if contains
$contains = $collection->contains(3); // true
$containsCallback = $collection->contains(function ($value) {
    return $value > 3;
}); // true

// To Array
$array = $collection->toArray();

// To JSON
$json = $collection->toJson();
```

### ২. Adding & Removing:
```php
$collection = collect([1, 2, 3]);

// Add items
$newCollection = $collection->push(4); // [1, 2, 3, 4]
$newCollection = $collection->prepend(0); // [0, 1, 2, 3]

// Add with key
$collection = collect(['name' => 'John']);
$newCollection = $collection->put('age', 25); // ['name' => 'John', 'age' => 25]

// Remove items
$collection = collect([1, 2, 3, 4, 5]);
$newCollection = $collection->pop(); // Removes last item
$newCollection = $collection->shift(); // Removes first item
$newCollection = $collection->forget(2); // Removes item at index 2

// Merge collections
$collection1 = collect([1, 2, 3]);
$collection2 = collect([4, 5, 6]);
$merged = $collection1->merge($collection2); // [1, 2, 3, 4, 5, 6]

// Concat (preserves keys)
$concatenated = $collection1->concat($collection2);
```

---

## Filtering Methods

### ১. Where Conditions:
```php
$users = collect([
    ['name' => 'John', 'age' => 25, 'city' => 'Dhaka', 'active' => true],
    ['name' => 'Jane', 'age' => 30, 'city' => 'Chittagong', 'active' => false],
    ['name' => 'Bob', 'age' => 35, 'city' => 'Dhaka', 'active' => true],
    ['name' => 'Alice', 'age' => 28, 'city' => 'Sylhet', 'active' => true]
]);

// Basic where
$dhakaUsers = $users->where('city', 'Dhaka');

// Where with operator
$youngUsers = $users->where('age', '<', 30);
$adultUsers = $users->where('age', '>=', 25);

// Where In
$cities = $users->whereIn('city', ['Dhaka', 'Chittagong']);

// Where Not In
$notDhaka = $users->whereNotIn('city', ['Dhaka']);

// Where Null/Not Null
$withEmail = $users->whereNotNull('email');
$withoutEmail = $users->whereNull('email');

// Where Between
$middleAged = $users->whereBetween('age', [25, 35]);

// Multiple conditions
$activeDhakaUsers = $users->where('city', 'Dhaka')->where('active', true);
```

### ২. Filter Method:
```php
$numbers = collect([1, 2, 3, 4, 5, 6, 7, 8, 9, 10]);

// Filter even numbers
$evenNumbers = $numbers->filter(function ($number) {
    return $number % 2 === 0;
});

// Filter with key
$filtered = $numbers->filter(function ($value, $key) {
    return $key > 2 && $value < 8;
});

// Reject (opposite of filter)
$oddNumbers = $numbers->reject(function ($number) {
    return $number % 2 === 0;
});

// First where condition matches
$firstAdult = $users->first(function ($user) {
    return $user['age'] >= 18;
});

// Take first N items
$firstThree = $numbers->take(3); // [1, 2, 3]

// Take last N items
$lastThree = $numbers->take(-3); // [8, 9, 10]

// Skip N items
$skipTwo = $numbers->skip(2); // [3, 4, 5, 6, 7, 8, 9, 10]
```

### ৩. Unique & Duplicates:
```php
$numbers = collect([1, 2, 2, 3, 3, 3, 4, 5]);

// Remove duplicates
$unique = $numbers->unique(); // [1, 2, 3, 4, 5]

// Unique by key
$users = collect([
    ['name' => 'John', 'city' => 'Dhaka'],
    ['name' => 'Jane', 'city' => 'Dhaka'],
    ['name' => 'Bob', 'city' => 'Chittagong']
]);
$uniqueCities = $users->unique('city');

// Unique by callback
$uniqueByFirstLetter = $users->unique(function ($user) {
    return substr($user['name'], 0, 1);
});

// Get duplicates
$duplicates = $numbers->duplicates();
```

---

## Transformation Methods

### ১. Map & Transform:
```php
$numbers = collect([1, 2, 3, 4, 5]);

// Map - Transform each item
$squared = $numbers->map(function ($number) {
    return $number * $number;
}); // [1, 4, 9, 16, 25]

// Map with key
$withKeys = $numbers->map(function ($value, $key) {
    return ['index' => $key, 'value' => $value, 'squared' => $value * $value];
});

// Transform (modifies original collection)
$numbers->transform(function ($number) {
    return $number * 2;
});

// Map into groups
$users = collect([
    ['name' => 'John', 'age' => 25],
    ['name' => 'Jane', 'age' => 30],
    ['name' => 'Bob', 'age' => 35]
]);

$grouped = $users->mapInto(User::class); // Convert to User objects
```

### ২. Pluck & Extract:
```php
$users = collect([
    ['name' => 'John', 'age' => 25, 'email' => 'john@example.com'],
    ['name' => 'Jane', 'age' => 30, 'email' => 'jane@example.com'],
    ['name' => 'Bob', 'age' => 35, 'email' => 'bob@example.com']
]);

// Pluck single column
$names = $users->pluck('name'); // ['John', 'Jane', 'Bob']

// Pluck with key
$namesByEmail = $users->pluck('name', 'email');
// ['john@example.com' => 'John', 'jane@example.com' => 'Jane', ...]

// Pluck nested values
$posts = collect([
    ['title' => 'Post 1', 'author' => ['name' => 'John', 'email' => 'john@example.com']],
    ['title' => 'Post 2', 'author' => ['name' => 'Jane', 'email' => 'jane@example.com']]
]);
$authorNames = $posts->pluck('author.name'); // ['John', 'Jane']

// Only specific keys
$basicInfo = $users->only(['name', 'email']);

// Except specific keys
$withoutAge = $users->except(['age']);
```

### ৩. Flatten & Collapse:
```php
// Flatten nested arrays
$nested = collect([
    [1, 2, 3],
    [4, 5, 6],
    [7, 8, 9]
]);
$flattened = $nested->flatten(); // [1, 2, 3, 4, 5, 6, 7, 8, 9]

// Flatten with depth
$deepNested = collect([
    [1, [2, [3, 4]]],
    [5, [6, [7, 8]]]
]);
$flattenedOne = $deepNested->flatten(1); // [1, [2, [3, 4]], 5, [6, [7, 8]]]

// Collapse (flatten one level)
$collapsed = $nested->collapse(); // [1, 2, 3, 4, 5, 6, 7, 8, 9]

// Zip collections together
$names = collect(['John', 'Jane', 'Bob']);
$ages = collect([25, 30, 35]);
$zipped = $names->zip($ages); // [['John', 25], ['Jane', 30], ['Bob', 35]]
```

---

## Aggregation Methods

### ১. Mathematical Operations:
```php
$numbers = collect([1, 2, 3, 4, 5]);

// Sum
$sum = $numbers->sum(); // 15

// Sum by key
$products = collect([
    ['name' => 'Laptop', 'price' => 50000],
    ['name' => 'Phone', 'price' => 30000],
    ['name' => 'Tablet', 'price' => 20000]
]);
$totalPrice = $products->sum('price'); // 100000

// Sum with callback
$totalWithTax = $products->sum(function ($product) {
    return $product['price'] * 1.15; // 15% tax
});

// Average
$average = $numbers->avg(); // 3
$avgPrice = $products->avg('price'); // 33333.33

// Min & Max
$min = $numbers->min(); // 1
$max = $numbers->max(); // 5
$cheapest = $products->min('price'); // 20000
$expensive = $products->max('price'); // 50000

// Median
$median = $numbers->median(); // 3

// Mode (most frequent)
$repeated = collect([1, 2, 2, 3, 3, 3, 4]);
$mode = $repeated->mode(); // [3]
```

### ২. Counting & Statistics:
```php
$users = collect([
    ['name' => 'John', 'city' => 'Dhaka', 'age' => 25],
    ['name' => 'Jane', 'city' => 'Dhaka', 'age' => 30],
    ['name' => 'Bob', 'city' => 'Chittagong', 'age' => 35],
    ['name' => 'Alice', 'city' => 'Dhaka', 'age' => 28]
]);

// Count total
$total = $users->count(); // 4

// Count by condition
$dhakaCount = $users->where('city', 'Dhaka')->count(); // 3

// Count by value
$cityCounts = $users->countBy('city');
// ['Dhaka' => 3, 'Chittagong' => 1]

// Count by callback
$ageCounts = $users->countBy(function ($user) {
    return $user['age'] >= 30 ? 'senior' : 'junior';
});
// ['junior' => 2, 'senior' => 2]

// Value counts
$ages = collect([25, 30, 25, 35, 30, 25]);
$valueCounts = $ages->countBy(); // [25 => 3, 30 => 2, 35 => 1]
```

---

## Grouping & Sorting

### ১. Grouping:
```php
$users = collect([
    ['name' => 'John', 'city' => 'Dhaka', 'age' => 25, 'department' => 'IT'],
    ['name' => 'Jane', 'city' => 'Dhaka', 'age' => 30, 'department' => 'HR'],
    ['name' => 'Bob', 'city' => 'Chittagong', 'age' => 35, 'department' => 'IT'],
    ['name' => 'Alice', 'city' => 'Dhaka', 'age' => 28, 'department' => 'Finance']
]);

// Group by key
$byCity = $users->groupBy('city');
// ['Dhaka' => [...], 'Chittagong' => [...]]

// Group by callback
$byAgeGroup = $users->groupBy(function ($user) {
    return $user['age'] >= 30 ? 'senior' : 'junior';
});

// Multiple level grouping
$byCityAndDept = $users->groupBy(['city', 'department']);

// Partition (split into two groups)
[$seniors, $juniors] = $users->partition(function ($user) {
    return $user['age'] >= 30;
});

// Chunk (split into smaller collections)
$chunks = $users->chunk(2); // Groups of 2 users each
```

### ২. Sorting:
```php
$users = collect([
    ['name' => 'John', 'age' => 25, 'salary' => 50000],
    ['name' => 'Jane', 'age' => 30, 'salary' => 60000],
    ['name' => 'Bob', 'age' => 35, 'salary' => 45000],
    ['name' => 'Alice', 'age' => 28, 'salary' => 55000]
]);

// Sort by value
$numbers = collect([3, 1, 4, 1, 5, 9]);
$sorted = $numbers->sort(); // [1, 1, 3, 4, 5, 9]
$sortedDesc = $numbers->sortDesc(); // [9, 5, 4, 3, 1, 1]

// Sort by key
$sortedByAge = $users->sortBy('age');
$sortedByAgeDesc = $users->sortByDesc('age');

// Sort by callback
$sortedByNameLength = $users->sortBy(function ($user) {
    return strlen($user['name']);
});

// Multiple column sorting
$sorted = $users->sortBy([
    ['age', 'asc'],
    ['salary', 'desc']
]);

// Sort keys
$data = collect(['b' => 2, 'a' => 1, 'c' => 3]);
$sortedKeys = $data->sortKeys(); // ['a' => 1, 'b' => 2, 'c' => 3]

// Shuffle
$shuffled = $users->shuffle();

// Reverse
$reversed = $users->reverse();
```

---

## Advanced Techniques

### ১. Reduce & Fold:
```php
$numbers = collect([1, 2, 3, 4, 5]);

// Reduce to single value
$sum = $numbers->reduce(function ($carry, $item) {
    return $carry + $item;
}, 0); // 15

$product = $numbers->reduce(function ($carry, $item) {
    return $carry * $item;
}, 1); // 120

// Complex reduce example
$users = collect([
    ['name' => 'John', 'orders' => 5, 'total' => 1000],
    ['name' => 'Jane', 'orders' => 3, 'total' => 800],
    ['name' => 'Bob', 'orders' => 7, 'total' => 1500]
]);

$summary = $users->reduce(function ($carry, $user) {
    $carry['total_orders'] += $user['orders'];
    $carry['total_amount'] += $user['total'];
    $carry['customers']++;
    return $carry;
}, ['total_orders' => 0, 'total_amount' => 0, 'customers' => 0]);

// Fold (similar to reduce but with different signature)
$concatenated = $numbers->fold('Numbers: ', function ($carry, $item) {
    return $carry . $item . ' ';
}); // "Numbers: 1 2 3 4 5 "
```

### ২. Collection Combinations:
```php
$collection1 = collect([1, 2, 3]);
$collection2 = collect([4, 5, 6]);

// Merge
$merged = $collection1->merge($collection2); // [1, 2, 3, 4, 5, 6]

// Union (keeps original keys)
$union = $collection1->union($collection2);

// Intersect (common values)
$intersect = collect([1, 2, 3, 4])->intersect([2, 3, 4, 5]); // [2, 3, 4]

// Diff (values in first but not in second)
$diff = collect([1, 2, 3, 4])->diff([2, 3]); // [1, 4]

// Symmetric difference
$collection1 = collect([1, 2, 3]);
$collection2 = collect([3, 4, 5]);
$symDiff = $collection1->diff($collection2)->merge($collection2->diff($collection1)); // [1, 2, 4, 5]

// Cross join
$colors = collect(['red', 'blue']);
$sizes = collect(['small', 'large']);
$combinations = $colors->crossJoin($sizes);
// [['red', 'small'], ['red', 'large'], ['blue', 'small'], ['blue', 'large']]
```

### ৩. Lazy Collections:
```php
// Memory efficient for large datasets
$lazyCollection = LazyCollection::make(function () {
    for ($i = 1; $i <= 1000000; $i++) {
        yield $i;
    }
});

// Process in chunks without loading all in memory
$processed = $lazyCollection
    ->filter(function ($number) {
        return $number % 2 === 0;
    })
    ->map(function ($number) {
        return $number * 2;
    })
    ->take(100);

// From file (memory efficient)
$lines = LazyCollection::make(function () {
    $handle = fopen('large-file.txt', 'r');
    while (($line = fgets($handle)) !== false) {
        yield $line;
    }
    fclose($handle);
});

$processedLines = $lines
    ->filter(function ($line) {
        return !empty(trim($line));
    })
    ->map(function ($line) {
        return strtoupper(trim($line));
    });
```

### ৪. Custom Collections:
```php
// Create custom collection class
class UserCollection extends Collection
{
    public function active()
    {
        return $this->filter(function ($user) {
            return $user['active'] === true;
        });
    }

    public function byCity($city)
    {
        return $this->where('city', $city);
    }

    public function averageAge()
    {
        return $this->avg('age');
    }

    public function seniors()
    {
        return $this->filter(function ($user) {
            return $user['age'] >= 30;
        });
    }
}

// Use custom collection
$users = new UserCollection([
    ['name' => 'John', 'age' => 25, 'city' => 'Dhaka', 'active' => true],
    ['name' => 'Jane', 'age' => 30, 'city' => 'Dhaka', 'active' => false],
    ['name' => 'Bob', 'age' => 35, 'city' => 'Chittagong', 'active' => true]
]);

$activeDhakaUsers = $users->active()->byCity('Dhaka');
$avgAge = $users->seniors()->averageAge();
```

### ৫. Real-world Examples:
```php
// E-commerce order processing
$orders = collect([
    ['id' => 1, 'customer' => 'John', 'items' => 3, 'total' => 150, 'status' => 'completed'],
    ['id' => 2, 'customer' => 'Jane', 'items' => 2, 'total' => 80, 'status' => 'pending'],
    ['id' => 3, 'customer' => 'John', 'items' => 1, 'total' => 50, 'status' => 'completed'],
    ['id' => 4, 'customer' => 'Bob', 'items' => 4, 'total' => 200, 'status' => 'completed']
]);

// Customer analytics
$customerStats = $orders
    ->where('status', 'completed')
    ->groupBy('customer')
    ->map(function ($customerOrders, $customer) {
        return [
            'customer' => $customer,
            'total_orders' => $customerOrders->count(),
            'total_spent' => $customerOrders->sum('total'),
            'avg_order_value' => $customerOrders->avg('total'),
            'total_items' => $customerOrders->sum('items')
        ];
    })
    ->sortByDesc('total_spent')
    ->values();

// Sales report
$salesReport = [
    'total_orders' => $orders->where('status', 'completed')->count(),
    'total_revenue' => $orders->where('status', 'completed')->sum('total'),
    'avg_order_value' => $orders->where('status', 'completed')->avg('total'),
    'pending_orders' => $orders->where('status', 'pending')->count(),
    'top_customers' => $customerStats->take(3),
    'revenue_by_month' => $orders
        ->where('status', 'completed')
        ->groupBy(function ($order) {
            return date('Y-m', strtotime($order['created_at'] ?? 'now'));
        })
        ->map(function ($monthOrders) {
            return $monthOrders->sum('total');
        })
];

// Blog post analytics
$posts = collect([
    ['title' => 'Laravel Tips', 'views' => 1500, 'likes' => 120, 'category' => 'Programming'],
    ['title' => 'PHP Best Practices', 'views' => 800, 'likes' => 65, 'category' => 'Programming'],
    ['title' => 'Travel Guide', 'views' => 2000, 'likes' => 200, 'category' => 'Travel'],
    ['title' => 'Cooking Recipe', 'views' => 1200, 'likes' => 90, 'category' => 'Food']
]);

$analytics = [
    'most_viewed' => $posts->sortByDesc('views')->first(),
    'most_liked' => $posts->sortByDesc('likes')->first(),
    'category_performance' => $posts
        ->groupBy('category')
        ->map(function ($categoryPosts) {
            return [
                'total_posts' => $categoryPosts->count(),
                'total_views' => $categoryPosts->sum('views'),
                'total_likes' => $categoryPosts->sum('likes'),
                'avg_engagement' => $categoryPosts->avg(function ($post) {
                    return $post['likes'] / $post['views'] * 100;
                })
            ];
        }),
    'engagement_rate' => $posts->avg(function ($post) {
        return $post['likes'] / $post['views'] * 100;
    })
];
```

---

## 🎯 Performance Tips:

### ✅ **Memory Efficiency:**
```php
// Use Lazy Collections for large datasets
$lazy = LazyCollection::make($generator);

// Chunk large collections
$users->chunk(100)->each(function ($chunk) {
    // Process chunk
});

// Use specific methods instead of generic ones
$users->pluck('name'); // Better than $users->map(fn($u) => $u['name'])
```

### ✅ **Method Chaining:**
```php
// Efficient chaining
$result = $collection
    ->where('active', true)
    ->sortBy('created_at')
    ->take(10)
    ->pluck('name');

// Avoid unnecessary conversions
$collection->toArray(); // Only when needed
```

### ✅ **Best Practices:**
```php
// Use appropriate methods
$collection->isEmpty(); // Better than $collection->count() === 0
$collection->isNotEmpty(); // Better than $collection->count() > 0

// Cache expensive operations
$expensiveResult = Cache::remember('expensive_collection_operation', 3600, function () {
    return $collection->complexOperation();
});
```

---

## 📚 আরও জানতে:
- [Laravel Collections](https://laravel.com/docs/collections)
- [Collection Methods](https://laravel.com/docs/collections#available-methods)
- [Lazy Collections](https://laravel.com/docs/collections#lazy-collections)