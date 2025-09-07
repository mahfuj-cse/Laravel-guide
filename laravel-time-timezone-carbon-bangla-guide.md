# Laravel Time & Timezone Management - বাংলা গাইড

## 📋 সূচিপত্র
- [Time & Timezone কি এবং কেন গুরুত্বপূর্ণ](#time--timezone-কি-এবং-কেন-গুরুত্বপূর্ণ)
- [Laravel এ Time Management](#laravel-এ-time-management)
- [Carbon Library](#carbon-library)
- [UTC vs Local Time](#utc-vs-local-time)
- [Database Time Handling](#database-time-handling)
- [User Timezone Management](#user-timezone-management)
- [Production Best Practices](#production-best-practices)

---

## Time & Timezone কি এবং কেন গুরুত্বপূর্ণ

### 🌍 Timezone কি?
**Timezone** হলো বিশ্বের বিভিন্ন অঞ্চলের সময়ের পার্থক্য। 

- **Bangladesh Standard Time (BST)** = UTC+6 (6 ঘন্টা এগিয়ে)
- **Eastern Standard Time (EST)** = UTC-5 (5 ঘন্টা পিছিয়ে)
- **Greenwich Mean Time (GMT)** = UTC+0 (standard reference)

**একই মুহূর্তে বিভিন্ন দেশে আলাদা সময়:**
- Dhaka: 2024-01-15 14:30:00 BST
- New York: 2024-01-15 03:30:00 EST  
- London: 2024-01-15 08:30:00 GMT

### 🚨 কেন Timezone গুরুত্বপূর্ণ?
- **Global Applications**: বিশ্বব্যাপী users এর জন্য সঠিক সময় দেখানো
- **Meeting Scheduling**: বিভিন্ন timezone এর users এর মধ্যে meeting schedule
- **Data Consistency**: Database এ consistent time storage
- **Legal Compliance**: Financial transactions এর জন্য accurate timestamp

```php
// ❌ Wrong: Timezone ছাড়া
'start_time' => $request->start_time // কোন timezone?

// ✅ Correct: Timezone সহ
'start_time' => Carbon::parse($request->start_time, $request->timezone)->utc()
```

---

## Laravel এ Time Management

### Laravel Time Handling System:

**Laravel এর Default Behavior:**
- Application timezone: `config/app.php` এ set করা
- Database storage: সবসময় UTC তে store হয়
- Auto conversion: Database থেকে পড়ার সময় app timezone এ convert

**Key Concepts:**
- **Server Time**: Server এর system time
- **Application Time**: Laravel config এ set করা timezone
- **Database Time**: সবসময় UTC format এ stored
- **User Time**: Individual user এর preferred timezone

```php
// Laravel এর automatic conversion
protected $dates = ['created_at', 'updated_at'];

// Database: 2024-01-15 08:30:00 (UTC)
// App shows: 2024-01-15 14:30:00 (if timezone is Asia/Dhaka)
```

### Time Layers in Laravel:

**Different Time Sources:**
1. **Server Time**: Operating system এর time
2. **PHP Time**: PHP configuration এর timezone
3. **Laravel Time**: Application config এর timezone
4. **Database Time**: Database server এর time (usually UTC)
5. **User Time**: Individual user এর timezone preference

**Time Flow Example:**
- Server: 2024-01-15 08:30:00 UTC
- Laravel App: 2024-01-15 14:30:00 (Asia/Dhaka)
- User Display: 2024-01-15 03:30:00 (America/New_York)

```php
// Different time sources
'server_time' => date('Y-m-d H:i:s'),
'laravel_now' => now()->format('Y-m-d H:i:s'),
'carbon_utc' => Carbon::now('UTC')->format('Y-m-d H:i:s'),
```

### Laravel Time Flow Process:

**6-Step Time Flow:**
1. **User Input**: User এর local time + timezone
2. **Parse**: Carbon দিয়ে timezone সহ parse করা
3. **Convert to UTC**: Database storage এর জন্য UTC তে convert
4. **Store**: Database এ UTC format এ save
5. **Retrieve**: Database থেকে পড়ে app timezone এ convert
6. **Display**: User এর timezone এ convert করে display

**Example Flow:**
```
1. User Input: 2024-01-15 14:30:00 (Asia/Dhaka)
2. Carbon Parse: 2024-01-15 14:30:00 BST
3. UTC Convert: 2024-01-15 08:30:00 UTC
4. DB Storage: 2024-01-15 08:30:00
5. Retrieved: 2024-01-15 14:30:00 BST
6. User Display: 2024-01-15 14:30:00 BST
```

```php
// Implementation
$carbonTime = Carbon::parse($userInput, $userTimezone);
$utcTime = $carbonTime->utc(); // For database
$displayTime = $retrieved->start_time->setTimezone($userTimezone); // For user
```

---

## Carbon Library

### Carbon কি?
**Carbon** হলো PHP এর DateTime class এর একটি powerful extension। Laravel এ built-in আছে।

### Carbon এর মূল Features:
- **Easy Date Creation**: বিভিন্ন উপায়ে date create করা
- **Timezone Support**: সহজে timezone conversion
- **Date Manipulation**: Add/subtract time periods
- **Formatting**: বিভিন্ন format এ display
- **Comparison**: Date comparison operations

### Basic Usage:
```php
// Current time
$now = Carbon::now();
$utcNow = Carbon::now('UTC');
$dhakaNow = Carbon::now('Asia/Dhaka');

// Parse dates
$date = Carbon::parse('2024-01-15 14:30:00', 'Asia/Dhaka');

// Create specific dates
$specific = Carbon::create(2024, 1, 15, 14, 30, 0, 'Asia/Dhaka');
$today = Carbon::today('Asia/Dhaka');
```

### Carbon Formatting:

**Common Formats:**
- **ISO String**: `toISOString()` → 2024-01-15T08:30:45.000000Z
- **Date Only**: `toDateString()` → 2024-01-15
- **Time Only**: `toTimeString()` → 14:30:45
- **DateTime**: `toDateTimeString()` → 2024-01-15 14:30:45

**Custom Formats:**
```php
$date = Carbon::parse('2024-01-15 14:30:45', 'Asia/Dhaka');

// Custom formatting
$date->format('d M Y, h:i A')  // 15 Jan 2024, 02:30 PM
$date->format('d/m/Y')         // 15/01/2024
$date->format('c')             // 2024-01-15T14:30:45+06:00

// Human readable
$date->diffForHumans()         // 2 hours ago
$date->toDayDateTimeString()   // Mon, Jan 15, 2024 2:30 PM

// Timestamps
$date->timestamp               // 1705308045
```

### Carbon Manipulation:

**Adding/Subtracting Time:**
```php
$date = Carbon::parse('2024-01-15 14:30:00');

// Adding time
$date->copy()->addDays(7)      // 7 দিন যোগ
$date->copy()->addHours(5)     // 5 ঘন্টা যোগ
$date->copy()->addMonths(2)    // 2 মাস যোগ

// Subtracting time
$date->copy()->subDays(3)      // 3 দিন বিয়োগ
$date->copy()->subHours(2)     // 2 ঘন্টা বিয়োগ
```

**Period Start/End:**
```php
// Period boundaries
$date->copy()->startOfDay()    // দিনের শুরু (00:00:00)
$date->copy()->endOfDay()      // দিনের শেষ (23:59:59)
$date->copy()->startOfMonth()  // মাসের শুরু
$date->copy()->endOfMonth()    // মাসের শেষ

// Specific day navigation
$date->copy()->next(Carbon::MONDAY)     // পরবর্তী সোমবার
$date->copy()->previous(Carbon::FRIDAY) // আগের শুক্রবার
```

**Important**: সবসময় `copy()` ব্যবহার করুন original date modify না করার জন্য।

### Carbon Comparisons:

**Basic Comparisons:**
```php
$date1 = Carbon::parse('2024-01-15 14:30:00');
$date2 = Carbon::parse('2024-01-20 10:00:00');

// Comparison methods
$date1->eq($date2)   // Equal
$date1->gt($date2)   // Greater than
$date1->lt($date2)   // Less than
$date1->gte($date2)  // Greater than or equal
$date1->lte($date2)  // Less than or equal
```

**Time Status Checks:**
```php
// Relative to now
$date->isPast()      // অতীতের তারিখ?
$date->isFuture()    // ভবিষ্যতের তারিখ?
$date->isToday()     // আজকের তারিখ?
$date->isYesterday() // গতকাল?
$date->isTomorrow()  // আগামীকাল?

// Day type checks
$date->isMonday()    // সোমবার?
$date->isWeekend()   // সাপ্তাহিক ছুটি?
$date->isWeekday()   // কার্যদিবস?
```

**Date Differences:**
```php
// Calculate differences
$date1->diffInDays($date2)     // দিনের পার্থক্য
$date1->diffInHours($date2)    // ঘন্টার পার্থক্য
$date1->diffForHumans($date2)  // "5 days before"

// Between check
$now->between($date1, $date2)  // দুটি তারিখের মধ্যে?
```

---

## UTC vs Local Time

### UTC কি?
**UTC (Coordinated Universal Time)** = বিশ্বব্যাপী standard time reference

**UTC এর বৈশিষ্ট্য:**
- সব দেশের জন্য একই reference point
- Timezone-independent storage
- Daylight Saving Time এর প্রভাব নেই
- Database consistency বজায় রাখে

**Bangladesh Context:**
- Bangladesh = UTC+6 (6 ঘন্টা এগিয়ে)
- Dhaka 14:30 = UTC 08:30
- কোন Daylight Saving Time নেই

### কেন UTC ব্যবহার করবেন?
- **Consistency**: সব সময় একই format
- **Global Apps**: বিভিন্ন দেশের users
- **No Confusion**: Timezone conversion এর ঝামেলা নেই
- **Database Standard**: Industry best practice

### UTC Storage Pattern:

**Best Practice: Store UTC, Display Local**

**Storage Process:**
1. User input: Local time + timezone
2. Convert to UTC: `Carbon::parse($userTime, $userTimezone)->utc()`
3. Store in database: UTC format
4. Also store: User's original timezone (optional)

**Display Process:**
1. Read from database: UTC time
2. Get user timezone: From profile/header/default
3. Convert to local: `$utcTime->setTimezone($userTimezone)`
4. Display to user: Local format

```php
// Storage
$utcTime = Carbon::parse($userTime, $userTimezone)->utc();
Event::create(['start_time' => $utcTime]);

// Display  
$userTimezone = auth()->user()->timezone ?? 'Asia/Dhaka';
$localTime = $event->start_time->setTimezone($userTimezone);
```

**Benefits:**
- Database consistency
- Easy timezone conversion
- No data corruption
- Global application support

### Multiple Timezone Handling:

**Global Meeting Example:**
একটি UTC time কে বিভিন্ন timezone এ convert করে দেখানো

**Common Timezones:**
- Asia/Dhaka (UTC+6)
- America/New_York (UTC-5/-4)
- Europe/London (UTC+0/+1)
- Asia/Tokyo (UTC+9)
- Australia/Sydney (UTC+10/+11)

```php
// UTC থেকে multiple timezone এ convert
$meetingTime = $meeting->start_time; // UTC from database

$timezones = [
    'Asia/Dhaka' => 'Bangladesh',
    'America/New_York' => 'New York',
    'Europe/London' => 'London',
];

foreach ($timezones as $timezone => $city) {
    $localTime = $meetingTime->copy()->setTimezone($timezone);
    echo "{$city}: " . $localTime->format('Y-m-d H:i:s T');
}
```

**Use Cases:**
- Global team meetings
- International events
- Multi-region applications
- Conference scheduling

---

## Database Time Handling

### Model Timestamp Configuration:

```php
class Event extends Model
{
    // Laravel automatically handles these as Carbon instances
    protected $dates = ['start_time', 'end_time'];
    
    // Or use casts (Laravel 7+)
    protected $casts = [
        'start_time' => 'datetime',
        'end_time' => 'datetime',
    ];
}
```

### Database Best Practices:

**Migration Setup:**
```php
// Always use timestamp columns for time data
$table->timestamp('start_time');
$table->timestamp('end_time');

// Store user's timezone separately if needed
$table->string('timezone', 50)->default('UTC');
```

**Query Examples:**
```php
// Today's events (user timezone aware)
$userTimezone = auth()->user()->timezone ?? 'Asia/Dhaka';
$todayStart = Carbon::today($userTimezone)->utc();
$todayEnd = Carbon::today($userTimezone)->endOfDay()->utc();

$events = Event::whereBetween('start_time', [$todayStart, $todayEnd])->get();
```

---

## User Timezone Management

### User Timezone Storage:

```php
// Migration
Schema::table('users', function (Blueprint $table) {
    $table->string('timezone')->default('Asia/Dhaka');
    $table->boolean('auto_detect_timezone')->default(true);
});
```

### Auto-detect Timezone:

**Frontend JavaScript:**
```javascript
// Get user's timezone
const userTimezone = Intl.DateTimeFormat().resolvedOptions().timeZone;

// Send to Laravel
fetch('/api/set-timezone', {
    method: 'POST',
    body: JSON.stringify({ timezone: userTimezone })
});
```

**Laravel Controller:**
```php
public function setTimezone(Request $request)
{
    auth()->user()->update([
        'timezone' => $request->timezone
    ]);
    
    return response()->json(['message' => 'Timezone updated']);
}
```

### Timezone Middleware:

```php
class SetUserTimezone
{
    public function handle($request, Closure $next)
    {
        $timezone = auth()->user()->timezone ?? 'Asia/Dhaka';
        Carbon::setDefaultTimezone($timezone);
        
        $response = $next($request);
        
        Carbon::setDefaultTimezone(config('app.timezone'));
        return $response;
    }
}
```

---

## Production Best Practices

### Server Configuration:

```bash
# Server timezone should be UTC
sudo timedatectl set-timezone UTC

# MySQL timezone
SET GLOBAL time_zone = '+00:00';

# PHP timezone (php.ini)
date.timezone = "UTC"
```

### Laravel Configuration:

```php
// config/app.php
'timezone' => 'UTC', // Always UTC for production

// .env
APP_TIMEZONE=UTC
DB_TIMEZONE=UTC
```

### Performance Tips:

**Cache Timezone Data:**
```php
// Cache frequently used timezone conversions
$cacheKey = "timezone_offset:{$timezone}";
$offset = Cache::remember($cacheKey, 3600, function () use ($timezone) {
    return Carbon::now($timezone)->format('P');
});
```

**Database Indexing:**
```php
// Add indexes for time-based queries
Schema::table('events', function (Blueprint $table) {
    $table->index(['start_time', 'timezone']);
    $table->index(['user_id', 'start_time']);
});
```

### Error Handling:

```php
public static function safeParseTime($timeString, $timezone = null)
{
    try {
        return Carbon::parse($timeString, $timezone);
    } catch (\Exception $e) {
        \Log::warning("Failed to parse time: {$timeString}");
        return Carbon::parse($timeString, 'UTC'); // Fallback
    }
}
```

### Testing:

```php
// Freeze time for consistent testing
Carbon::setTestNow('2024-01-15 08:30:00');

$event = Event::create([
    'start_time' => Carbon::parse('2024-01-15 14:30:00', 'Asia/Dhaka'),
]);

// Should be stored as UTC
$this->assertEquals('2024-01-15 08:30:00', 
    $event->start_time->utc()->format('Y-m-d H:i:s'));
```

---

## সারসংক্ষেপ

### 🕐 Time Management Best Practices:
- ✅ সবসময় UTC তে database এ store করুন
- ✅ User এর timezone track করুন  
- ✅ Carbon library ব্যবহার করুন
- ✅ Timezone validation implement করুন
- ✅ Server timezone UTC রাখুন

### 📊 Key Commands:
```bash
# Server setup
sudo timedatectl set-timezone UTC

# Laravel
php artisan make:middleware SetUserTimezone
php artisan make:migration add_timezone_to_users
```

### 💡 Essential Code Patterns:
```php
// Storage: Always UTC
$utcTime = Carbon::parse($userTime, $userTimezone)->utc();

// Display: User's timezone  
$displayTime = $utcTime->setTimezone($userTimezone);

// API Response: Multiple formats
return [
    'utc' => $time->utc()->toISOString(),
    'local' => $time->setTimezone($userTimezone)->format('Y-m-d H:i:s'),
    'formatted' => $time->setTimezone($userTimezone)->format('M j, Y g:i A'),
];
```

এই গাইড Laravel এ professional time management system তৈরি করতে সাহায্য করবে।