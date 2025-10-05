# Database Normalization, Indexing, Relationships & Transactions - সম্পূর্ণ বাংলা গাইড

## 📋 সূচিপত্র
- [Database Normalization](#database-normalization)
- [Database Indexing](#database-indexing)
- [Database Relationships](#database-relationships)
- [Database Transactions](#database-transactions)
- [Real-world Examples](#real-world-examples)
- [Best Practices](#best-practices)

---

## Database Normalization

### 🎯 **Normalization কি?**
**Database Normalization** হলো **data redundancy** কমানো এবং **data integrity** বজায় রাখার একটি process। এটি **duplicate data** eliminate করে এবং **storage space** save করে।

### 📚 **সহজ উদাহরণ - স্কুল ডাটাবেস:**

#### ❌ **Un-normalized Table (0NF):**
```sql
-- students_courses table (Bad Design)
CREATE TABLE students_courses (
    id INT PRIMARY KEY,
    student_name VARCHAR(100),
    student_email VARCHAR(100),
    student_phone VARCHAR(20),
    student_address TEXT,
    course_name VARCHAR(100),
    course_code VARCHAR(10),
    teacher_name VARCHAR(100),
    teacher_email VARCHAR(100),
    grade CHAR(2)
);

-- Sample Data
INSERT INTO students_courses VALUES
(1, 'রহিম', 'rahim@email.com', '01711111111', 'ঢাকা', 'গণিত', 'MATH101', 'করিম স্যার', 'karim@school.com', 'A+'),
(2, 'রহিম', 'rahim@email.com', '01711111111', 'ঢাকা', 'পদার্থবিজ্ঞান', 'PHY101', 'সালাম স্যার', 'salam@school.com', 'A'),
(3, 'করিম', 'karim@email.com', '01722222222', 'চট্টগ্রাম', 'গণিত', 'MATH101', 'করিম স্যার', 'karim@school.com', 'B+');
```

**সমস্যাসমূহ:**
- 🚨 **Data Redundancy**: রহিমের তথ্য দুইবার stored
- 🚨 **Update Anomaly**: রহিমের ফোন নম্বর change করলে multiple places এ update করতে হবে
- 🚨 **Insert Anomaly**: নতুন teacher add করতে হলে student থাকতে হবে
- 🚨 **Delete Anomaly**: student delete করলে teacher এর তথ্যও চলে যাবে

### 🔄 **1st Normal Form (1NF):**

**Rule:** প্রতিটি column এ **atomic values** থাকতে হবে (no repeating groups)

```sql
-- ❌ Violates 1NF
CREATE TABLE students (
    id INT PRIMARY KEY,
    name VARCHAR(100),
    subjects VARCHAR(200) -- 'গণিত, পদার্থবিজ্ঞান, রসায়ন' (Multiple values)
);

-- ✅ Follows 1NF
CREATE TABLE students (
    id INT PRIMARY KEY,
    name VARCHAR(100),
    email VARCHAR(100),
    phone VARCHAR(20)
);

CREATE TABLE student_subjects (
    id INT PRIMARY KEY,
    student_id INT,
    subject VARCHAR(100),
    FOREIGN KEY (student_id) REFERENCES students(id)
);
```

### 🔄 **2nd Normal Form (2NF):**

**Rule:** 1NF + **No partial dependency** (non-key attributes should depend on entire primary key)

```sql
-- ❌ Violates 2NF
CREATE TABLE order_items (
    order_id INT,
    product_id INT,
    product_name VARCHAR(100), -- Depends only on product_id, not on (order_id, product_id)
    product_price DECIMAL(10,2), -- Depends only on product_id
    quantity INT,
    PRIMARY KEY (order_id, product_id)
);

-- ✅ Follows 2NF
CREATE TABLE orders (
    id INT PRIMARY KEY,
    customer_id INT,
    order_date DATE
);

CREATE TABLE products (
    id INT PRIMARY KEY,
    name VARCHAR(100),
    price DECIMAL(10,2)
);

CREATE TABLE order_items (
    order_id INT,
    product_id INT,
    quantity INT,
    PRIMARY KEY (order_id, product_id),
    FOREIGN KEY (order_id) REFERENCES orders(id),
    FOREIGN KEY (product_id) REFERENCES products(id)
);
```

### 🔄 **3rd Normal Form (3NF):**

**Rule:** 2NF + **No transitive dependency** (non-key attributes should not depend on other non-key attributes)

```sql
-- ❌ Violates 3NF
CREATE TABLE employees (
    id INT PRIMARY KEY,
    name VARCHAR(100),
    department_id INT,
    department_name VARCHAR(100), -- Depends on department_id (transitive dependency)
    department_location VARCHAR(100) -- Depends on department_id
);

-- ✅ Follows 3NF
CREATE TABLE departments (
    id INT PRIMARY KEY,
    name VARCHAR(100),
    location VARCHAR(100)
);

CREATE TABLE employees (
    id INT PRIMARY KEY,
    name VARCHAR(100),
    department_id INT,
    FOREIGN KEY (department_id) REFERENCES departments(id)
);
```

### 🏪 **Real Example - E-commerce Database:**

#### **Before Normalization:**
```sql
-- ❌ Bad Design
CREATE TABLE orders_bad (
    order_id INT PRIMARY KEY,
    customer_name VARCHAR(100),
    customer_email VARCHAR(100),
    customer_phone VARCHAR(20),
    customer_address TEXT,
    product_names TEXT, -- 'Laptop, Mouse, Keyboard'
    product_prices TEXT, -- '50000, 1500, 2000'
    quantities TEXT, -- '1, 2, 1'
    total_amount DECIMAL(10,2)
);
```

#### **After Normalization:**
```sql
-- ✅ Good Design (3NF)
CREATE TABLE customers (
    id INT PRIMARY KEY,
    name VARCHAR(100),
    email VARCHAR(100) UNIQUE,
    phone VARCHAR(20),
    address TEXT
);

CREATE TABLE products (
    id INT PRIMARY KEY,
    name VARCHAR(100),
    price DECIMAL(10,2),
    stock_quantity INT
);

CREATE TABLE orders (
    id INT PRIMARY KEY,
    customer_id INT,
    order_date TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    total_amount DECIMAL(10,2),
    FOREIGN KEY (customer_id) REFERENCES customers(id)
);

CREATE TABLE order_items (
    id INT PRIMARY KEY,
    order_id INT,
    product_id INT,
    quantity INT,
    unit_price DECIMAL(10,2),
    FOREIGN KEY (order_id) REFERENCES orders(id),
    FOREIGN KEY (product_id) REFERENCES products(id)
);
```

---

## Database Indexing

### 🚀 **Indexing কি?**
**Database Index** হলো একটি **data structure** যা **query performance** improve করে। এটি **book এর index** এর মতো কাজ করে।

### 📖 **সহজ উদাহরণ - বই এর Index:**
```
বই এর পৃষ্ঠা: 1000 পৃষ্ঠা
Index ছাড়া: পুরো বই পড়ে 'Laravel' খুঁজতে হবে (1000 পৃষ্ঠা)
Index সহ: Index দেখে সরাসরি পৃষ্ঠা 245 এ যাওয়া (1 সেকেন্ড)
```

### 🔍 **Index Types:**

#### **1. Primary Index (Clustered):**
```sql
-- Primary key automatically creates clustered index
CREATE TABLE users (
    id INT PRIMARY KEY, -- Clustered index
    name VARCHAR(100),
    email VARCHAR(100)
);

-- Data physically stored in order of id
-- Row 1: id=1, name='আহমেদ', email='ahmed@email.com'
-- Row 2: id=2, name='রহিম', email='rahim@email.com'
-- Row 3: id=3, name='করিম', email='karim@email.com'
```

#### **2. Secondary Index (Non-clustered):**
```sql
-- Create index on frequently searched columns
CREATE INDEX idx_users_email ON users(email);
CREATE INDEX idx_users_name ON users(name);

-- Query performance improvement
SELECT * FROM users WHERE email = 'rahim@email.com'; -- Fast with index
SELECT * FROM users WHERE name = 'রহিম'; -- Fast with index
SELECT * FROM users WHERE phone = '01711111111'; -- Slow without index
```

#### **3. Composite Index:**
```sql
-- Index on multiple columns
CREATE INDEX idx_orders_customer_date ON orders(customer_id, order_date);

-- Efficient for queries like:
SELECT * FROM orders WHERE customer_id = 123 AND order_date = '2024-01-15';
SELECT * FROM orders WHERE customer_id = 123; -- Also efficient

-- Not efficient for:
SELECT * FROM orders WHERE order_date = '2024-01-15'; -- Only uses partial index
```

#### **4. Unique Index:**
```sql
-- Ensures uniqueness and improves performance
CREATE UNIQUE INDEX idx_users_email_unique ON users(email);

-- Prevents duplicate emails
INSERT INTO users (name, email) VALUES ('রহিম', 'rahim@email.com'); -- OK
INSERT INTO users (name, email) VALUES ('করিম', 'rahim@email.com'); -- Error: Duplicate
```

### 📊 **Performance Example:**

```sql
-- Table with 1 million users
CREATE TABLE users (
    id INT PRIMARY KEY,
    name VARCHAR(100),
    email VARCHAR(100),
    phone VARCHAR(20),
    city VARCHAR(50),
    created_at TIMESTAMP
);

-- Without Index
SELECT * FROM users WHERE email = 'john@example.com';
-- Execution time: 2.5 seconds (Full table scan)

-- With Index
CREATE INDEX idx_users_email ON users(email);
SELECT * FROM users WHERE email = 'john@example.com';
-- Execution time: 0.001 seconds (Index seek)
```

### ⚡ **Laravel Indexing:**

```php
// Migration with indexes
Schema::create('posts', function (Blueprint $table) {
    $table->id();
    $table->string('title');
    $table->text('content');
    $table->unsignedBigInteger('user_id');
    $table->string('status')->default('draft');
    $table->timestamp('published_at')->nullable();
    $table->timestamps();
    
    // Indexes
    $table->index('user_id'); // Single column index
    $table->index(['status', 'published_at']); // Composite index
    $table->unique('slug'); // Unique index
    
    // Foreign key with index
    $table->foreign('user_id')->references('id')->on('users');
});

// Add index to existing table
Schema::table('posts', function (Blueprint $table) {
    $table->index('title');
});
```

---

## Database Relationships

### 🔗 **Relationship Types:**

#### **1. One-to-One (1:1):**
```sql
-- Example: User এবং Profile
CREATE TABLE users (
    id INT PRIMARY KEY,
    name VARCHAR(100),
    email VARCHAR(100)
);

CREATE TABLE profiles (
    id INT PRIMARY KEY,
    user_id INT UNIQUE, -- UNIQUE ensures 1:1
    bio TEXT,
    avatar VARCHAR(255),
    FOREIGN KEY (user_id) REFERENCES users(id)
);
```

```php
// Laravel Eloquent
class User extends Model
{
    public function profile()
    {
        return $this->hasOne(Profile::class);
    }
}

class Profile extends Model
{
    public function user()
    {
        return $this->belongsTo(User::class);
    }
}

// Usage
$user = User::with('profile')->find(1);
echo $user->profile->bio;
```

#### **2. One-to-Many (1:N):**
```sql
-- Example: User এবং Posts
CREATE TABLE users (
    id INT PRIMARY KEY,
    name VARCHAR(100)
);

CREATE TABLE posts (
    id INT PRIMARY KEY,
    user_id INT,
    title VARCHAR(200),
    content TEXT,
    FOREIGN KEY (user_id) REFERENCES users(id)
);
```

```php
// Laravel Eloquent
class User extends Model
{
    public function posts()
    {
        return $this->hasMany(Post::class);
    }
}

class Post extends Model
{
    public function user()
    {
        return $this->belongsTo(User::class);
    }
}

// Usage
$user = User::with('posts')->find(1);
foreach ($user->posts as $post) {
    echo $post->title;
}
```

#### **3. Many-to-Many (M:N):**
```sql
-- Example: Students এবং Courses
CREATE TABLE students (
    id INT PRIMARY KEY,
    name VARCHAR(100)
);

CREATE TABLE courses (
    id INT PRIMARY KEY,
    name VARCHAR(100),
    code VARCHAR(10)
);

-- Pivot table
CREATE TABLE student_course (
    id INT PRIMARY KEY,
    student_id INT,
    course_id INT,
    enrolled_at TIMESTAMP,
    grade CHAR(2),
    FOREIGN KEY (student_id) REFERENCES students(id),
    FOREIGN KEY (course_id) REFERENCES courses(id),
    UNIQUE KEY unique_enrollment (student_id, course_id)
);
```

```php
// Laravel Eloquent
class Student extends Model
{
    public function courses()
    {
        return $this->belongsToMany(Course::class)
                   ->withPivot(['enrolled_at', 'grade'])
                   ->withTimestamps();
    }
}

class Course extends Model
{
    public function students()
    {
        return $this->belongsToMany(Student::class)
                   ->withPivot(['enrolled_at', 'grade']);
    }
}

// Usage
$student = Student::with('courses')->find(1);
foreach ($student->courses as $course) {
    echo $course->name . ' - Grade: ' . $course->pivot->grade;
}
```

### 🏫 **Real Example - School Management:**

```sql
-- Complete school database with relationships
CREATE TABLE schools (
    id INT PRIMARY KEY,
    name VARCHAR(100),
    address TEXT
);

CREATE TABLE classes (
    id INT PRIMARY KEY,
    school_id INT,
    name VARCHAR(50), -- 'Class 10A'
    room_number VARCHAR(10),
    FOREIGN KEY (school_id) REFERENCES schools(id)
);

CREATE TABLE teachers (
    id INT PRIMARY KEY,
    school_id INT,
    name VARCHAR(100),
    subject VARCHAR(50),
    FOREIGN KEY (school_id) REFERENCES schools(id)
);

CREATE TABLE students (
    id INT PRIMARY KEY,
    class_id INT,
    name VARCHAR(100),
    roll_number INT,
    FOREIGN KEY (class_id) REFERENCES classes(id)
);

CREATE TABLE subjects (
    id INT PRIMARY KEY,
    name VARCHAR(100),
    code VARCHAR(10)
);

-- Many-to-many: Teachers can teach multiple subjects
CREATE TABLE teacher_subjects (
    teacher_id INT,
    subject_id INT,
    PRIMARY KEY (teacher_id, subject_id),
    FOREIGN KEY (teacher_id) REFERENCES teachers(id),
    FOREIGN KEY (subject_id) REFERENCES subjects(id)
);

-- Many-to-many: Students can take multiple subjects
CREATE TABLE student_subjects (
    student_id INT,
    subject_id INT,
    teacher_id INT,
    marks DECIMAL(5,2),
    PRIMARY KEY (student_id, subject_id),
    FOREIGN KEY (student_id) REFERENCES students(id),
    FOREIGN KEY (subject_id) REFERENCES subjects(id),
    FOREIGN KEY (teacher_id) REFERENCES teachers(id)
);
```

---

## Database Transactions

### 💳 **Transaction কি?**
**Database Transaction** হলো **একাধিক database operations** এর একটি **logical unit** যা **সম্পূর্ণভাবে execute** হবে অথবা **একেবারেই হবে না**।

### 🏦 **সহজ উদাহরণ - ব্যাংক Transfer:**

```sql
-- ❌ Without Transaction (Dangerous)
UPDATE accounts SET balance = balance - 1000 WHERE account_number = 'ACC001'; -- রহিমের account থেকে কাটা
-- যদি এখানে error হয় বা power cut হয়?
UPDATE accounts SET balance = balance + 1000 WHERE account_number = 'ACC002'; -- করিমের account এ যোগ

-- সমস্যা: রহিমের টাকা কেটে গেছে কিন্তু করিমের account এ যোগ হয়নি!
```

```sql
-- ✅ With Transaction (Safe)
START TRANSACTION;

UPDATE accounts SET balance = balance - 1000 WHERE account_number = 'ACC001';
UPDATE accounts SET balance = balance + 1000 WHERE account_number = 'ACC002';

-- Check if both operations successful
IF (@@ERROR = 0)
    COMMIT; -- Save changes
ELSE
    ROLLBACK; -- Cancel all changes
END IF;
```

### 🔒 **ACID Properties:**

#### **A - Atomicity (অবিভাজ্যতা):**
```php
// Laravel Transaction Example
DB::transaction(function () {
    // Either all operations succeed or all fail
    $order = Order::create([
        'customer_id' => 1,
        'total' => 5000
    ]);
    
    OrderItem::create([
        'order_id' => $order->id,
        'product_id' => 1,
        'quantity' => 2
    ]);
    
    Product::where('id', 1)->decrement('stock', 2);
    
    // If any operation fails, all will be rolled back
});
```

#### **C - Consistency (সামঞ্জস্য):**
```sql
-- Database constraints ensure consistency
CREATE TABLE accounts (
    id INT PRIMARY KEY,
    account_number VARCHAR(20) UNIQUE,
    balance DECIMAL(15,2) CHECK (balance >= 0), -- Cannot be negative
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

-- Transaction maintains consistency
START TRANSACTION;
UPDATE accounts SET balance = balance - 1000 WHERE id = 1;
UPDATE accounts SET balance = balance + 1000 WHERE id = 2;
-- Total money in system remains same (consistent)
COMMIT;
```

#### **I - Isolation (বিচ্ছিন্নতা):**
```php
// Concurrent transactions don't interfere
// Transaction 1
DB::transaction(function () {
    $product = Product::find(1);
    if ($product->stock >= 5) {
        $product->decrement('stock', 5);
        Order::create(['product_id' => 1, 'quantity' => 5]);
    }
});

// Transaction 2 (running simultaneously)
DB::transaction(function () {
    $product = Product::find(1);
    if ($product->stock >= 3) {
        $product->decrement('stock', 3);
        Order::create(['product_id' => 1, 'quantity' => 3]);
    }
});
```

#### **D - Durability (স্থায়িত্ব):**
```sql
-- Once committed, data is permanent
START TRANSACTION;
INSERT INTO orders (customer_id, total) VALUES (1, 5000);
COMMIT; -- Data is now permanently saved

-- Even if system crashes after COMMIT, data will be there
```

### 🛒 **Real Example - E-commerce Order:**

```php
class OrderService
{
    public function createOrder($orderData)
    {
        return DB::transaction(function () use ($orderData) {
            // 1. Create order
            $order = Order::create([
                'customer_id' => $orderData['customer_id'],
                'total' => $orderData['total'],
                'status' => 'pending'
            ]);
            
            // 2. Create order items and update stock
            foreach ($orderData['items'] as $item) {
                // Check stock availability
                $product = Product::lockForUpdate()->find($item['product_id']);
                
                if ($product->stock < $item['quantity']) {
                    throw new InsufficientStockException();
                }
                
                // Create order item
                OrderItem::create([
                    'order_id' => $order->id,
                    'product_id' => $item['product_id'],
                    'quantity' => $item['quantity'],
                    'price' => $product->price
                ]);
                
                // Update stock
                $product->decrement('stock', $item['quantity']);
            }
            
            // 3. Process payment
            $payment = Payment::create([
                'order_id' => $order->id,
                'amount' => $orderData['total'],
                'method' => $orderData['payment_method']
            ]);
            
            // 4. Update customer points
            $customer = Customer::find($orderData['customer_id']);
            $customer->increment('points', $orderData['total'] * 0.01);
            
            return $order;
            
            // If any step fails, everything will be rolled back
        });
    }
}
```

### 🔄 **Transaction Isolation Levels:**

```php
// Read Uncommitted (Lowest isolation)
DB::transaction(function () {
    // Can read uncommitted changes from other transactions
    // Problem: Dirty reads possible
}, 1); // ISOLATION_READ_UNCOMMITTED

// Read Committed (Default in most databases)
DB::transaction(function () {
    // Can only read committed changes
    // Problem: Non-repeatable reads possible
}, 2); // ISOLATION_READ_COMMITTED

// Repeatable Read
DB::transaction(function () {
    // Same query returns same results within transaction
    // Problem: Phantom reads possible
}, 4); // ISOLATION_REPEATABLE_READ

// Serializable (Highest isolation)
DB::transaction(function () {
    // Complete isolation, transactions run as if serial
    // Problem: Performance impact
}, 8); // ISOLATION_SERIALIZABLE
```

---

## Real-world Examples

### 🏪 **Complete E-commerce Database Design:**

```sql
-- Normalized, indexed, with proper relationships
CREATE TABLE categories (
    id INT PRIMARY KEY AUTO_INCREMENT,
    name VARCHAR(100) NOT NULL,
    slug VARCHAR(100) UNIQUE NOT NULL,
    parent_id INT NULL,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    INDEX idx_categories_parent (parent_id),
    INDEX idx_categories_slug (slug),
    FOREIGN KEY (parent_id) REFERENCES categories(id)
);

CREATE TABLE products (
    id INT PRIMARY KEY AUTO_INCREMENT,
    category_id INT NOT NULL,
    name VARCHAR(200) NOT NULL,
    slug VARCHAR(200) UNIQUE NOT NULL,
    description TEXT,
    price DECIMAL(10,2) NOT NULL,
    stock_quantity INT NOT NULL DEFAULT 0,
    is_active BOOLEAN DEFAULT TRUE,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP,
    
    INDEX idx_products_category (category_id),
    INDEX idx_products_slug (slug),
    INDEX idx_products_active_price (is_active, price),
    INDEX idx_products_stock (stock_quantity),
    FOREIGN KEY (category_id) REFERENCES categories(id)
);

CREATE TABLE customers (
    id INT PRIMARY KEY AUTO_INCREMENT,
    email VARCHAR(100) UNIQUE NOT NULL,
    password VARCHAR(255) NOT NULL,
    first_name VARCHAR(50) NOT NULL,
    last_name VARCHAR(50) NOT NULL,
    phone VARCHAR(20),
    date_of_birth DATE,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    
    UNIQUE INDEX idx_customers_email (email),
    INDEX idx_customers_name (first_name, last_name)
);

CREATE TABLE orders (
    id INT PRIMARY KEY AUTO_INCREMENT,
    customer_id INT NOT NULL,
    order_number VARCHAR(20) UNIQUE NOT NULL,
    status ENUM('pending', 'processing', 'shipped', 'delivered', 'cancelled') DEFAULT 'pending',
    subtotal DECIMAL(10,2) NOT NULL,
    tax_amount DECIMAL(10,2) NOT NULL DEFAULT 0,
    shipping_amount DECIMAL(10,2) NOT NULL DEFAULT 0,
    total_amount DECIMAL(10,2) NOT NULL,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    
    INDEX idx_orders_customer (customer_id),
    INDEX idx_orders_status (status),
    INDEX idx_orders_date (created_at),
    INDEX idx_orders_number (order_number),
    FOREIGN KEY (customer_id) REFERENCES customers(id)
);

CREATE TABLE order_items (
    id INT PRIMARY KEY AUTO_INCREMENT,
    order_id INT NOT NULL,
    product_id INT NOT NULL,
    quantity INT NOT NULL,
    unit_price DECIMAL(10,2) NOT NULL,
    total_price DECIMAL(10,2) NOT NULL,
    
    INDEX idx_order_items_order (order_id),
    INDEX idx_order_items_product (product_id),
    FOREIGN KEY (order_id) REFERENCES orders(id) ON DELETE CASCADE,
    FOREIGN KEY (product_id) REFERENCES products(id)
);
```

### 💰 **Transaction Example - Order Processing:**

```php
class OrderProcessor
{
    public function processOrder($customerId, $items, $paymentData)
    {
        return DB::transaction(function () use ($customerId, $items, $paymentData) {
            
            // 1. Validate customer
            $customer = Customer::findOrFail($customerId);
            
            // 2. Calculate totals
            $subtotal = 0;
            $orderItems = [];
            
            foreach ($items as $item) {
                $product = Product::lockForUpdate()->findOrFail($item['product_id']);
                
                // Check stock
                if ($product->stock_quantity < $item['quantity']) {
                    throw new Exception("Insufficient stock for {$product->name}");
                }
                
                $itemTotal = $product->price * $item['quantity'];
                $subtotal += $itemTotal;
                
                $orderItems[] = [
                    'product_id' => $product->id,
                    'quantity' => $item['quantity'],
                    'unit_price' => $product->price,
                    'total_price' => $itemTotal
                ];
            }
            
            $taxAmount = $subtotal * 0.1; // 10% tax
            $shippingAmount = $subtotal > 1000 ? 0 : 100; // Free shipping over 1000
            $totalAmount = $subtotal + $taxAmount + $shippingAmount;
            
            // 3. Create order
            $order = Order::create([
                'customer_id' => $customerId,
                'order_number' => 'ORD-' . time() . '-' . rand(1000, 9999),
                'subtotal' => $subtotal,
                'tax_amount' => $taxAmount,
                'shipping_amount' => $shippingAmount,
                'total_amount' => $totalAmount,
                'status' => 'pending'
            ]);
            
            // 4. Create order items and update stock
            foreach ($orderItems as $item) {
                OrderItem::create(array_merge($item, ['order_id' => $order->id]));
                
                Product::where('id', $item['product_id'])
                       ->decrement('stock_quantity', $item['quantity']);
            }
            
            // 5. Process payment
            $payment = $this->processPayment($order, $paymentData);
            
            if (!$payment->successful) {
                throw new Exception('Payment failed');
            }
            
            // 6. Update order status
            $order->update(['status' => 'processing']);
            
            // 7. Send confirmation email
            Mail::to($customer->email)->send(new OrderConfirmation($order));
            
            return $order;
            
        }); // End transaction - if any step fails, everything rolls back
    }
    
    private function processPayment($order, $paymentData)
    {
        // Payment processing logic
        return (object) ['successful' => true];
    }
}
```

---

## Best Practices

### ✅ **Normalization Best Practices:**

1. **Start with 3NF** for most applications
2. **Denormalize only when necessary** for performance
3. **Use foreign keys** to maintain referential integrity
4. **Avoid over-normalization** that leads to complex joins

### ✅ **Indexing Best Practices:**

1. **Index frequently queried columns**
2. **Use composite indexes** for multi-column queries
3. **Don't over-index** - impacts INSERT/UPDATE performance
4. **Monitor query performance** and add indexes as needed

### ✅ **Relationship Best Practices:**

1. **Use appropriate relationship types**
2. **Always define foreign key constraints**
3. **Use cascade options carefully**
4. **Consider performance impact** of deep relationships

### ✅ **Transaction Best Practices:**

1. **Keep transactions short** to avoid locks
2. **Handle exceptions properly** with try-catch
3. **Use appropriate isolation levels**
4. **Avoid nested transactions** when possible

### 📊 **Performance Monitoring:**

```sql
-- Check slow queries
SHOW PROCESSLIST;

-- Analyze query performance
EXPLAIN SELECT * FROM products WHERE category_id = 1 AND price > 1000;

-- Check index usage
SHOW INDEX FROM products;

-- Monitor transaction locks
SELECT * FROM information_schema.INNODB_LOCKS;
```

এই concepts গুলো master করলে আপনি **efficient, scalable এবং maintainable** database design করতে পারবেন।