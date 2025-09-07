# Laravel Time & Timezone Management - সম্পূর্ণ বাংলা গাইড

## 📋 সূচিপত্র
- [Time & Timezone কি এবং কেন গুরুত্বপূর্ণ](#time--timezone-কি-এবং-কেন-গুরুত্বপূর্ণ)
- [Laravel এ Time Management](#laravel-এ-time-management)
- [Carbon Library](#carbon-library)
- [UTC vs Local Time](#utc-vs-local-time)
- [Timezone Configuration](#timezone-configuration)
- [Database Time Handling](#database-time-handling)
- [User Timezone Management](#user-timezone-management)
- [API Time Response](#api-time-response)
- [Production Best Practices](#production-best-practices)

---

## Time & Timezone কি এবং কেন গুরুত্বপূর্ণ

### 🌍 Timezone কি?
```php
<?php
// বিশ্বের বিভিন্ন অঞ্চলের সময়ের পার্থক্য

// Bangladesh Standard Time (BST) = UTC+6
$dhaka = new DateTime('2024-01-15 14:30:00', new DateTimeZone('Asia/Dhaka'));

// Eastern Standard Time (EST) = UTC-5  
$newYork = new DateTime('2024-01-15 03:30:00', new DateTimeZone('America/New_York'));

// Greenwich Mean Time (GMT) = UTC+0
$london = new DateTime('2024-01-15 08:30:00', new DateTimeZone('Europe/London'));

// সবগুলো একই সময়কে represent করে!
echo "Dhaka: " . $dhaka->format('Y-m-d H:i:s T') . "\n";     // 2024-01-15 14:30:00 BST
echo "New York: " . $newYork->format('Y-m-d H:i:s T') . "\n"; // 2024-01-15 03:30:00 EST  
echo "London: " . $london->format('Y-m-d H:i:s T') . "\n";   // 2024-01-15 08:30:00 GMT
```

### 🚨 কেন Timezone গুরুত্বপূর্ণ?
```php
<?php
// ❌ Wrong approach - timezone ছাড়া
class EventController extends Controller
{
    public function createEvent(Request $request)
    {
        Event::create([
            'title' => $request->title,
            'start_time' => $request->start_time, // "2024-01-15 14:30:00" - কোন timezone?
        ]);
    }
}

// ✅ Correct approach - timezone সহ
class EventController extends Controller
{
    public function createEvent(Request $request)
    {
        Event::create([
            'title' => $request->title,
            'start_time' => Carbon::parse($request->start_time, $request->timezone)->utc(),
            'timezone' => $request->timezone,
        ]);
    }
}
```

---

## Laravel এ Time Management

### ১. Laravel এর Default Time Handling:

```php
<?php
// Laravel কিভাবে time handle করে

// config/app.php
'timezone' => 'UTC', // Laravel এর default timezone

// Laravel automatically converts:
// 1. Database থেকে data পড়ার সময় UTC থেকে app timezone এ convert
// 2. Database এ save করার সময় app timezone থেকে UTC তে convert

// Example
class User extends Model
{
    protected $dates = ['created_at', 'updated_at', 'email_verified_at'];
    
    // Laravel automatically handles these conversions:
    public function getCreatedAtAttribute($value)
    {
        // Database: 2024-01-15 08:30:00 (UTC)
        // Returns: 2024-01-15 14:30:00 (if app timezone is Asia/Dhaka)
        return $this->asDateTime($value);
    }
}
```

### ২. Server Time vs Application Time:

```php
<?php
class TimeDebugController extends Controller
{
    public function showTimeInfo()
    {
        return response()->json([
            // Server level times
            'server_timezone' => date_default_timezone_get(),
            'server_time' => date('Y-m-d H:i:s'),
            'server_timestamp' => time(),
            
            // PHP level times  
            'php_timezone' => ini_get('date.timezone'),
            'php_time' => (new DateTime())->format('Y-m-d H:i:s'),
            
            // Laravel level times
            'laravel_timezone' => config('app.timezone'),
            'laravel_now' => now()->format('Y-m-d H:i:s'),
            'laravel_utc_now' => now()->utc()->format('Y-m-d H:i:s'),
            
            // Carbon times
            'carbon_now' => Carbon::now()->format('Y-m-d H:i:s'),
            'carbon_utc' => Carbon::now('UTC')->format('Y-m-d H:i:s'),
            'carbon_dhaka' => Carbon::now('Asia/Dhaka')->format('Y-m-d H:i:s'),
            
            // Database time
            'database_time' => DB::select('SELECT NOW() as current_time')[0]->current_time,
        ]);
    }
}

// Output example:
{
    "server_timezone": "UTC",
    "server_time": "2024-01-15 08:30:00",
    "php_timezone": "UTC", 
    "php_time": "2024-01-15 08:30:00",
    "laravel_timezone": "Asia/Dhaka",
    "laravel_now": "2024-01-15 14:30:00",
    "laravel_utc_now": "2024-01-15 08:30:00",
    "carbon_now": "2024-01-15 14:30:00",
    "carbon_utc": "2024-01-15 08:30:00", 
    "carbon_dhaka": "2024-01-15 14:30:00",
    "database_time": "2024-01-15 08:30:00"
}
```

### ৩. Laravel Time Flow:

```php
<?php
// Laravel এ time কিভাবে flow করে

class TimeFlowExample extends Controller
{
    public function demonstrateFlow()
    {
        echo "=== Laravel Time Flow ===\n";
        
        // Step 1: User input (with timezone)
        $userInput = "2024-01-15 14:30:00"; // User এর local time
        $userTimezone = "Asia/Dhaka";
        echo "1. User Input: {$userInput} ({$userTimezone})\n";
        
        // Step 2: Convert to Carbon with timezone
        $carbonTime = Carbon::parse($userInput, $userTimezone);
        echo "2. Carbon Parse: " . $carbonTime->format('Y-m-d H:i:s T') . "\n";
        
        // Step 3: Convert to UTC for database storage
        $utcTime = $carbonTime->utc();
        echo "3. UTC for DB: " . $utcTime->format('Y-m-d H:i:s T') . "\n";
        
        // Step 4: Save to database (Laravel auto-converts)
        $event = new Event();
        $event->start_time = $carbonTime; // Laravel saves as UTC
        echo "4. Saved to DB: " . $event->start_time->utc()->format('Y-m-d H:i:s') . "\n";
        
        // Step 5: Retrieve from database (Laravel auto-converts to app timezone)
        $retrieved = Event::find(1);
        echo "5. Retrieved: " . $retrieved->start_time->format('Y-m-d H:i:s T') . "\n";
        
        // Step 6: Convert to user's timezone for display
        $displayTime = $retrieved->start_time->setTimezone($userTimezone);
        echo "6. Display to User: " . $displayTime->format('Y-m-d H:i:s T') . "\n";
    }
}

// Output:
// 1. User Input: 2024-01-15 14:30:00 (Asia/Dhaka)
// 2. Carbon Parse: 2024-01-15 14:30:00 BST
// 3. UTC for DB: 2024-01-15 08:30:00 UTC
// 4. Saved to DB: 2024-01-15 08:30:00
// 5. Retrieved: 2024-01-15 14:30:00 BST (if app timezone is Asia/Dhaka)
// 6. Display to User: 2024-01-15 14:30:00 BST
```

---

## Carbon Library

### ১. Carbon Basics:

```php
<?php
use Carbon\Carbon;

class CarbonBasicsController extends Controller
{
    public function carbonExamples()
    {
        // Current time
        $now = Carbon::now();
        $utcNow = Carbon::now('UTC');
        $dhakaNow = Carbon::now('Asia/Dhaka');
        
        // Parsing dates
        $parsed1 = Carbon::parse('2024-01-15 14:30:00');
        $parsed2 = Carbon::parse('2024-01-15 14:30:00', 'Asia/Dhaka');
        $parsed3 = Carbon::createFromFormat('d/m/Y H:i', '15/01/2024 14:30');
        
        // Creating specific dates
        $specific = Carbon::create(2024, 1, 15, 14, 30, 0, 'Asia/Dhaka');
        $today = Carbon::today('Asia/Dhaka');
        $tomorrow = Carbon::tomorrow('Asia/Dhaka');
        $yesterday = Carbon::yesterday('Asia/Dhaka');
        
        return response()->json([
            'current_times' => [
                'now' => $now->toISOString(),
                'utc_now' => $utcNow->toISOString(),
                'dhaka_now' => $dhakaNow->toISOString(),
            ],
            'parsed_dates' => [
                'parsed1' => $parsed1->toISOString(),
                'parsed2' => $parsed2->toISOString(), 
                'parsed3' => $parsed3->toISOString(),
            ],
            'specific_dates' => [
                'specific' => $specific->toISOString(),
                'today' => $today->toDateString(),
                'tomorrow' => $tomorrow->toDateString(),
                'yesterday' => $yesterday->toDateString(),
            ]
        ]);
    }
}
```

### ২. Carbon Formatting:

```php
<?php
class CarbonFormattingController extends Controller
{
    public function formatExamples()
    {
        $date = Carbon::parse('2024-01-15 14:30:45', 'Asia/Dhaka');
        
        return response()->json([
            // Basic formats
            'iso_string' => $date->toISOString(),           // 2024-01-15T08:30:45.000000Z
            'date_string' => $date->toDateString(),         // 2024-01-15
            'time_string' => $date->toTimeString(),         // 14:30:45
            'datetime_string' => $date->toDateTimeString(), // 2024-01-15 14:30:45
            
            // Custom formats
            'custom_format' => $date->format('d M Y, h:i A'), // 15 Jan 2024, 02:30 PM
            'bangla_format' => $date->format('d/m/Y'),        // 15/01/2024
            'api_format' => $date->format('c'),               // 2024-01-15T14:30:45+06:00
            
            // Human readable
            'human_diff' => $date->diffForHumans(),          // 2 hours ago
            'human_readable' => $date->toDayDateTimeString(), // Mon, Jan 15, 2024 2:30 PM
            
            // Timestamps
            'timestamp' => $date->timestamp,                  // 1705308045
            'milliseconds' => $date->getPreciseTimestamp(3), // 1705308045000
            
            // Different timezones
            'utc' => $date->utc()->toISOString(),
            'new_york' => $date->setTimezone('America/New_York')->format('Y-m-d H:i:s T'),
            'london' => $date->setTimezone('Europe/London')->format('Y-m-d H:i:s T'),
        ]);
    }
}
```

### ৩. Carbon Manipulation:

```php
<?php
class CarbonManipulationController extends Controller
{
    public function manipulationExamples()
    {
        $date = Carbon::parse('2024-01-15 14:30:00', 'Asia/Dhaka');
        
        return response()->json([
            'original' => $date->toDateTimeString(),
            
            // Adding time
            'add_days' => $date->copy()->addDays(7)->toDateTimeString(),
            'add_hours' => $date->copy()->addHours(5)->toDateTimeString(),
            'add_minutes' => $date->copy()->addMinutes(30)->toDateTimeString(),
            'add_months' => $date->copy()->addMonths(2)->toDateTimeString(),
            
            // Subtracting time
            'sub_days' => $date->copy()->subDays(3)->toDateTimeString(),
            'sub_hours' => $date->copy()->subHours(2)->toDateTimeString(),
            
            // Start/End of periods
            'start_of_day' => $date->copy()->startOfDay()->toDateTimeString(),
            'end_of_day' => $date->copy()->endOfDay()->toDateTimeString(),
            'start_of_month' => $date->copy()->startOfMonth()->toDateTimeString(),
            'end_of_month' => $date->copy()->endOfMonth()->toDateTimeString(),
            'start_of_week' => $date->copy()->startOfWeek()->toDateTimeString(),
            'end_of_week' => $date->copy()->endOfWeek()->toDateTimeString(),
            
            // Specific modifications
            'next_monday' => $date->copy()->next(Carbon::MONDAY)->toDateTimeString(),
            'previous_friday' => $date->copy()->previous(Carbon::FRIDAY)->toDateTimeString(),
            'set_time' => $date->copy()->setTime(9, 0, 0)->toDateTimeString(),
            'set_date' => $date->copy()->setDate(2024, 12, 25)->toDateTimeString(),
        ]);
    }
}
```

### ৪. Carbon Comparisons:

```php
<?php
class CarbonComparisonController extends Controller
{
    public function comparisonExamples()
    {
        $date1 = Carbon::parse('2024-01-15 14:30:00');
        $date2 = Carbon::parse('2024-01-20 10:00:00');
        $now = Carbon::now();
        
        return response()->json([
            // Basic comparisons
            'is_equal' => $date1->eq($date2),           // false
            'is_not_equal' => $date1->ne($date2),       // true
            'is_greater' => $date1->gt($date2),         // false
            'is_less' => $date1->lt($date2),            // true
            'is_greater_equal' => $date1->gte($date2),  // false
            'is_less_equal' => $date1->lte($date2),     // true
            
            // Time checks
            'is_past' => $date1->isPast(),              // true/false
            'is_future' => $date1->isFuture(),          // true/false
            'is_today' => $date1->isToday(),            // true/false
            'is_yesterday' => $date1->isYesterday(),    // true/false
            'is_tomorrow' => $date1->isTomorrow(),      // true/false
            
            // Day checks
            'is_monday' => $date1->isMonday(),          // true/false
            'is_weekend' => $date1->isWeekend(),        // true/false
            'is_weekday' => $date1->isWeekday(),        // true/false
            
            // Date differences
            'diff_in_days' => $date1->diffInDays($date2),       // 5
            'diff_in_hours' => $date1->diffInHours($date2),     // 115
            'diff_in_minutes' => $date1->diffInMinutes($date2), // 6930
            'diff_for_humans' => $date1->diffForHumans($date2), // "5 days before"
            
            // Between checks
            'is_between' => $now->between($date1, $date2),      // true/false
        ]);
    }
}
```

---

## UTC vs Local Time

### ১. UTC কি এবং কেন ব্যবহার করি:

```php
<?php
// UTC (Coordinated Universal Time) = বিশ্বব্যাপী standard time

class UTCExplanationController extends Controller
{
    public function explainUTC()
    {
        $localTime = Carbon::now('Asia/Dhaka');  // Local time
        $utcTime = Carbon::now('UTC');           // UTC time
        
        return response()->json([
            'explanation' => [
                'utc_definition' => 'UTC is the primary time standard by which the world regulates clocks and time',
                'why_use_utc' => 'Consistent time reference across all timezones',
                'bangladesh_offset' => 'Bangladesh is UTC+6 (6 hours ahead of UTC)',
            ],
            
            'current_times' => [
                'dhaka_local' => $localTime->format('Y-m-d H:i:s T'),
                'utc_time' => $utcTime->format('Y-m-d H:i:s T'),
                'offset_hours' => $localTime->offsetHours,
                'timezone_name' => $localTime->timezoneName,
            ],
            
            'conversion_example' => [
                'local_to_utc' => $localTime->utc()->format('Y-m-d H:i:s T'),
                'utc_to_local' => $utcTime->setTimezone('Asia/Dhaka')->format('Y-m-d H:i:s T'),
            ],
            
            'database_storage' => [
                'recommended' => 'Always store in UTC',
                'reason' => 'Timezone-independent storage',
                'display' => 'Convert to user timezone when displaying',
            ]
        ]);
    }
}
```

### ২. UTC Storage Pattern:

```php
<?php
// Best practice: Store UTC, Display Local

class EventController extends Controller
{
    public function store(Request $request)
    {
        // Input: User's local time + timezone
        $userTime = $request->start_time;     // "2024-01-15 14:30:00"
        $userTimezone = $request->timezone;   // "Asia/Dhaka"
        
        // Convert to UTC for storage
        $utcTime = Carbon::parse($userTime, $userTimezone)->utc();
        
        $event = Event::create([
            'title' => $request->title,
            'start_time' => $utcTime,           // Stored as UTC
            'user_timezone' => $userTimezone,   // Store user's timezone
        ]);
        
        return response()->json([
            'event' => $event,
            'stored_utc' => $utcTime->toISOString(),
            'user_local' => $utcTime->setTimezone($userTimezone)->format('Y-m-d H:i:s T'),
        ]);
    }
    
    public function show(Event $event, Request $request)
    {
        // Get user's preferred timezone (from request, user profile, or default)
        $userTimezone = $request->header('X-Timezone') 
                       ?? auth()->user()->timezone 
                       ?? 'Asia/Dhaka';
        
        // Convert UTC time to user's timezone
        $localTime = $event->start_time->setTimezone($userTimezone);
        
        return response()->json([
            'event' => $event,
            'times' => [
                'utc' => $event->start_time->utc()->toISOString(),
                'local' => $localTime->format('Y-m-d H:i:s T'),
                'user_timezone' => $userTimezone,
            ]
        ]);
    }
}
```

### ৩. Multiple Timezone Handling:

```php
<?php
class MultiTimezoneController extends Controller
{
    public function showGlobalMeeting(Meeting $meeting)
    {
        $meetingTime = $meeting->start_time; // UTC time from database
        
        // Convert to different timezones
        $timezones = [
            'Asia/Dhaka' => 'Bangladesh',
            'America/New_York' => 'New York',
            'Europe/London' => 'London', 
            'Asia/Tokyo' => 'Tokyo',
            'Australia/Sydney' => 'Sydney',
            'America/Los_Angeles' => 'Los Angeles',
        ];
        
        $globalTimes = [];
        foreach ($timezones as $timezone => $city) {
            $localTime = $meetingTime->copy()->setTimezone($timezone);
            $globalTimes[] = [
                'city' => $city,
                'timezone' => $timezone,
                'time' => $localTime->format('Y-m-d H:i:s'),
                'offset' => $localTime->format('T (P)'),
                'is_next_day' => $localTime->format('Y-m-d') !== $meetingTime->utc()->format('Y-m-d'),
            ];
        }
        
        return response()->json([
            'meeting' => $meeting,
            'utc_time' => $meetingTime->utc()->toISOString(),
            'global_times' => $globalTimes,
        ]);
    }
}
```

---

## Timezone Configuration

### ১. Application Timezone Setup:

```php
<?php
// config/app.php
return [
    // Application timezone (affects Carbon::now(), Model timestamps)
    'timezone' => env('APP_TIMEZONE', 'UTC'),
    
    // Available timezones for users
    'available_timezones' => [
        'Asia/Dhaka' => 'Bangladesh Standard Time (BST)',
        'Asia/Kolkata' => 'India Standard Time (IST)', 
        'America/New_York' => 'Eastern Standard Time (EST)',
        'Europe/London' => 'Greenwich Mean Time (GMT)',
        'Asia/Dubai' => 'Gulf Standard Time (GST)',
        'Asia/Singapore' => 'Singapore Standard Time (SST)',
    ],
];

// .env
APP_TIMEZONE=Asia/Dhaka
DB_TIMEZONE=UTC
```

### ২. Database Timezone Configuration:

```php
<?php
// config/database.php
'mysql' => [
    'driver' => 'mysql',
    'host' => env('DB_HOST', '127.0.0.1'),
    'port' => env('DB_PORT', '3306'),
    'database' => env('DB_DATABASE', 'forge'),
    'username' => env('DB_USERNAME', 'forge'),
    'password' => env('DB_PASSWORD', ''),
    'charset' => 'utf8mb4',
    'collation' => 'utf8mb4_unicode_ci',
    'prefix' => '',
    'prefix_indexes' => true,
    'strict' => true,
    'engine' => null,
    'options' => extension_loaded('pdo_mysql') ? array_filter([
        PDO::MYSQL_ATTR_SSL_CA => env('MYSQL_ATTR_SSL_CA'),
    ]) : [],
    // Force UTC timezone for database connections
    'timezone' => '+00:00',
],

// Database connection timezone setup
class DatabaseServiceProvider extends ServiceProvider
{
    public function boot()
    {
        // Set database timezone to UTC
        DB::statement("SET time_zone = '+00:00'");
        
        // Or set in connection
        config(['database.connections.mysql.timezone' => '+00:00']);
    }
}
```

### ৩. User Timezone Management:

```php
<?php
// Migration for user timezone
Schema::table('users', function (Blueprint $table) {
    $table->string('timezone')->default('Asia/Dhaka');
    $table->boolean('auto_detect_timezone')->default(true);
});

// User Model
class User extends Authenticatable
{
    protected $fillable = ['name', 'email', 'password', 'timezone'];
    
    public function getLocalTime($utcTime)
    {
        return Carbon::parse($utcTime)->setTimezone($this->timezone);
    }
    
    public function convertToUTC($localTime)
    {
        return Carbon::parse($localTime, $this->timezone)->utc();
    }
    
    // Accessor for formatted timezone
    public function getTimezoneDisplayAttribute()
    {
        $timezones = config('app.available_timezones');
        return $timezones[$this->timezone] ?? $this->timezone;
    }
}

// Timezone Controller
class TimezoneController extends Controller
{
    public function updateTimezone(Request $request)
    {
        $request->validate([
            'timezone' => 'required|string|in:' . implode(',', array_keys(config('app.available_timezones')))
        ]);
        
        auth()->user()->update([
            'timezone' => $request->timezone
        ]);
        
        return response()->json([
            'message' => 'Timezone updated successfully',
            'timezone' => $request->timezone,
            'current_time' => Carbon::now($request->timezone)->format('Y-m-d H:i:s T')
        ]);
    }
    
    public function getAvailableTimezones()
    {
        $timezones = [];
        foreach (config('app.available_timezones') as $timezone => $display) {
            $now = Carbon::now($timezone);
            $timezones[] = [
                'timezone' => $timezone,
                'display_name' => $display,
                'current_time' => $now->format('H:i'),
                'offset' => $now->format('P'),
                'offset_hours' => $now->offsetHours,
            ];
        }
        
        return response()->json(['timezones' => $timezones]);
    }
}
```

---

## Database Time Handling

### ১. Model Timestamp Handling:

```php
<?php
class Event extends Model
{
    protected $fillable = ['title', 'start_time', 'end_time', 'timezone'];
    
    // Laravel automatically handles these as Carbon instances
    protected $dates = ['start_time', 'end_time'];
    
    // Or use casts (Laravel 7+)
    protected $casts = [
        'start_time' => 'datetime',
        'end_time' => 'datetime',
    ];
    
    // Custom accessor to get time in user's timezone
    public function getStartTimeLocalAttribute()
    {
        $userTimezone = auth()->user()->timezone ?? 'Asia/Dhaka';
        return $this->start_time->setTimezone($userTimezone);
    }
    
    // Custom mutator to ensure UTC storage
    public function setStartTimeAttribute($value)
    {
        // If value comes with timezone info, convert to UTC
        if (is_string($value)) {
            $this->attributes['start_time'] = Carbon::parse($value)->utc();
        } else {
            $this->attributes['start_time'] = $value;
        }
    }
    
    // Scope for timezone-aware queries
    public function scopeInTimezone($query, $timezone)
    {
        return $query->selectRaw("
            *,
            CONVERT_TZ(start_time, '+00:00', ?) as start_time_local,
            CONVERT_TZ(end_time, '+00:00', ?) as end_time_local
        ", [$this->getTimezoneOffset($timezone), $this->getTimezoneOffset($timezone)]);
    }
    
    private function getTimezoneOffset($timezone)
    {
        return Carbon::now($timezone)->format('P');
    }
}
```

### ২. Database Queries with Timezone:

```php
<?php
class EventQueryController extends Controller
{
    public function getTodaysEvents(Request $request)
    {
        $userTimezone = $request->header('X-Timezone') ?? auth()->user()->timezone ?? 'Asia/Dhaka';
        
        // Method 1: Convert in PHP
        $todayStart = Carbon::today($userTimezone)->utc();
        $todayEnd = Carbon::today($userTimezone)->endOfDay()->utc();
        
        $events = Event::whereBetween('start_time', [$todayStart, $todayEnd])
                      ->get()
                      ->map(function ($event) use ($userTimezone) {
                          $event->start_time_local = $event->start_time->setTimezone($userTimezone);
                          return $event;
                      });
        
        // Method 2: Convert in Database (MySQL)
        $eventsDb = Event::selectRaw("
            *,
            CONVERT_TZ(start_time, '+00:00', ?) as start_time_local,
            DATE(CONVERT_TZ(start_time, '+00:00', ?)) as event_date_local
        ", [
            Carbon::now($userTimezone)->format('P'),
            Carbon::now($userTimezone)->format('P')
        ])
        ->whereRaw("DATE(CONVERT_TZ(start_time, '+00:00', ?)) = ?", [
            Carbon::now($userTimezone)->format('P'),
            Carbon::today($userTimezone)->format('Y-m-d')
        ])
        ->get();
        
        return response()->json([
            'user_timezone' => $userTimezone,
            'today_date' => Carbon::today($userTimezone)->format('Y-m-d'),
            'events_php' => $events,
            'events_db' => $eventsDb,
        ]);
    }
    
    public function getEventsInDateRange(Request $request)
    {
        $request->validate([
            'start_date' => 'required|date',
            'end_date' => 'required|date|after_or_equal:start_date',
            'timezone' => 'required|string'
        ]);
        
        // Convert user's date range to UTC for database query
        $startUTC = Carbon::parse($request->start_date, $request->timezone)
                          ->startOfDay()
                          ->utc();
        
        $endUTC = Carbon::parse($request->end_date, $request->timezone)
                        ->endOfDay()
                        ->utc();
        
        $events = Event::whereBetween('start_time', [$startUTC, $endUTC])
                      ->orderBy('start_time')
                      ->get()
                      ->groupBy(function ($event) use ($request) {
                          return $event->start_time
                                      ->setTimezone($request->timezone)
                                      ->format('Y-m-d');
                      });
        
        return response()->json([
            'date_range' => [
                'start' => $request->start_date,
                'end' => $request->end_date,
                'timezone' => $request->timezone,
            ],
            'utc_range' => [
                'start' => $startUTC->toISOString(),
                'end' => $endUTC->toISOString(),
            ],
            'events_by_date' => $events,
        ]);
    }
}
```

### ৩. Migration with Timezone Considerations:

```php
<?php
// Migration for timezone-aware tables
class CreateEventsTable extends Migration
{
    public function up()
    {
        Schema::create('events', function (Blueprint $table) {
            $table->id();
            $table->string('title');
            $table->text('description')->nullable();
            
            // Store all times in UTC
            $table->timestamp('start_time');
            $table->timestamp('end_time');
            
            // Store original timezone for reference
            $table->string('timezone', 50)->default('UTC');
            
            // User who created the event
            $table->foreignId('user_id')->constrained();
            
            // Laravel timestamps (also UTC)
            $table->timestamps();
            
            // Indexes for timezone queries
            $table->index(['start_time', 'timezone']);
            $table->index(['user_id', 'start_time']);
        });
    }
}

// Seeder with timezone data
class EventSeeder extends Seeder
{
    public function run()
    {
        $timezones = ['Asia/Dhaka', 'America/New_York', 'Europe/London'];
        
        foreach ($timezones as $timezone) {
            Event::create([
                'title' => "Meeting in {$timezone}",
                'start_time' => Carbon::parse('2024-01-15 14:30:00', $timezone)->utc(),
                'end_time' => Carbon::parse('2024-01-15 15:30:00', $timezone)->utc(),
                'timezone' => $timezone,
                'user_id' => 1,
            ]);
        }
    }
}
```

---

## User Timezone Management

### ১. Auto-detect User Timezone:

```php
<?php
// Frontend JavaScript to detect timezone
/*
<script>
// Get user's timezone
const userTimezone = Intl.DateTimeFormat().resolvedOptions().timeZone;

// Send to Laravel
fetch('/api/set-timezone', {
    method: 'POST',
    headers: {
        'Content-Type': 'application/json',
        'X-CSRF-TOKEN': document.querySelector('meta[name="csrf-token"]').content
    },
    body: JSON.stringify({
        timezone: userTimezone,
        offset: new Date().getTimezoneOffset()
    })
});
</script>
*/

class TimezoneDetectionController extends Controller
{
    public function setTimezone(Request $request)
    {
        $request->validate([
            'timezone' => 'required|string',
            'offset' => 'required|integer'
        ]);
        
        // Validate timezone
        if (!in_array($request->timezone, timezone_identifiers_list())) {
            return response()->json(['error' => 'Invalid timezone'], 400);
        }
        
        // Update user timezone
        if (auth()->check()) {
            auth()->user()->update([
                'timezone' => $request->timezone,
                'timezone_offset' => $request->offset,
                'timezone_detected_at' => now(),
            ]);
        } else {
            // Store in session for guest users
            session([
                'user_timezone' => $request->timezone,
                'timezone_offset' => $request->offset,
            ]);
        }
        
        return response()->json([
            'message' => 'Timezone set successfully',
            'timezone' => $request->timezone,
            'current_time' => Carbon::now($request->timezone)->format('Y-m-d H:i:s T'),
        ]);
    }
    
    public function getTimezoneInfo(Request $request)
    {
        $timezone = $this->getUserTimezone($request);
        $now = Carbon::now($timezone);
        
        return response()->json([
            'timezone' => $timezone,
            'current_time' => $now->format('Y-m-d H:i:s T'),
            'utc_offset' => $now->format('P'),
            'offset_hours' => $now->offsetHours,
            'is_dst' => $now->dst,
            'timezone_name' => $now->timezoneName,
        ]);
    }
    
    private function getUserTimezone($request)
    {
        // Priority: Header > User Profile > Session > Default
        return $request->header('X-Timezone')
               ?? auth()->user()->timezone ?? null
               ?? session('user_timezone')
               ?? config('app.timezone');
    }
}
```

### ২. Timezone Middleware:

```php
<?php
// app/Http/Middleware/SetUserTimezone.php
namespace App\Http\Middleware;

use Closure;
use Illuminate\Http\Request;
use Carbon\Carbon;

class SetUserTimezone
{
    public function handle(Request $request, Closure $next)
    {
        // Get user's timezone
        $timezone = $this->getUserTimezone($request);
        
        // Set Carbon default timezone for this request
        Carbon::setDefaultTimezone($timezone);
        
        // Add timezone to request for easy access
        $request->merge(['user_timezone' => $timezone]);
        
        $response = $next($request);
        
        // Reset to app default after request
        Carbon::setDefaultTimezone(config('app.timezone'));
        
        // Add timezone header to response
        $response->headers->set('X-User-Timezone', $timezone);
        
        return $response;
    }
    
    private function getUserTimezone($request)
    {
        return $request->header('X-Timezone')
               ?? auth()->user()->timezone ?? null
               ?? session('user_timezone')
               ?? 'Asia/Dhaka';
    }
}

// Register middleware
// app/Http/Kernel.php
protected $middlewareGroups = [
    'api' => [
        \App\Http\Middleware\SetUserTimezone::class,
    ],
];
```

### ৩. Timezone Helper Service:

```php
<?php
// app/Services/TimezoneService.php
namespace App\Services;

use Carbon\Carbon;
use Illuminate\Support\Facades\Cache;

class TimezoneService
{
    public function convertToUserTimezone($utcTime, $userTimezone = null)
    {
        $timezone = $userTimezone ?? $this->getUserTimezone();
        return Carbon::parse($utcTime)->setTimezone($timezone);
    }
    
    public function convertToUTC($localTime, $userTimezone = null)
    {
        $timezone = $userTimezone ?? $this->getUserTimezone();
        return Carbon::parse($localTime, $timezone)->utc();
    }
    
    public function getUserTimezone()
    {
        return auth()->user()->timezone ?? 
               session('user_timezone') ?? 
               'Asia/Dhaka';
    }
    
    public function getTimezoneList()
    {
        return Cache::remember('timezone_list', 3600, function () {
            $timezones = [];
            
            foreach (timezone_identifiers_list() as $timezone) {
                try {
                    $carbon = Carbon::now($timezone);
                    $timezones[] = [
                        'timezone' => $timezone,
                        'display_name' => $this->getTimezoneDisplayName($timezone),
                        'offset' => $carbon->format('P'),
                        'offset_hours' => $carbon->offsetHours,
                        'current_time' => $carbon->format('H:i'),
                    ];
                } catch (\Exception $e) {
                    continue;
                }
            }
            
            // Sort by offset
            usort($timezones, function ($a, $b) {
                return $a['offset_hours'] <=> $b['offset_hours'];
            });
            
            return $timezones;
        });
    }
    
    private function getTimezoneDisplayName($timezone)
    {
        $displayNames = [
            'Asia/Dhaka' => 'Bangladesh Standard Time',
            'Asia/Kolkata' => 'India Standard Time',
            'America/New_York' => 'Eastern Time',
            'Europe/London' => 'Greenwich Mean Time',
            'Asia/Dubai' => 'Gulf Standard Time',
        ];
        
        return $displayNames[$timezone] ?? str_replace(['_', '/'], [' ', ' / '], $timezone);
    }
    
    public function formatForUser($utcTime, $format = 'Y-m-d H:i:s', $userTimezone = null)
    {
        return $this->convertToUserTimezone($utcTime, $userTimezone)->format($format);
    }
    
    public function getBusinessHours($timezone = null)
    {
        $timezone = $timezone ?? $this->getUserTimezone();
        $now = Carbon::now($timezone);
        
        return [
            'timezone' => $timezone,
            'current_time' => $now->format('H:i'),
            'is_business_hours' => $now->hour >= 9 && $now->hour < 18 && $now->isWeekday(),
            'business_start' => '09:00',
            'business_end' => '18:00',
            'is_weekend' => $now->isWeekend(),
        ];
    }
}
```

---

## API Time Response

### ১. Consistent API Time Format:

```php
<?php
// app/Http/Resources/EventResource.php
namespace App\Http\Resources;

use Illuminate\Http\Resources\Json\JsonResource;

class EventResource extends JsonResource
{
    public function toArray($request)
    {
        $userTimezone = $request->user_timezone ?? 'Asia/Dhaka';
        
        return [
            'id' => $this->id,
            'title' => $this->title,
            'description' => $this->description,
            
            // Multiple time formats for flexibility
            'times' => [
                'utc' => [
                    'start' => $this->start_time->utc()->toISOString(),
                    'end' => $this->end_time->utc()->toISOString(),
                ],
                'local' => [
                    'start' => $this->start_time->setTimezone($userTimezone)->toISOString(),
                    'end' => $this->end_time->setTimezone($userTimezone)->toISOString(),
                ],
                'formatted' => [
                    'start' => $this->start_time->setTimezone($userTimezone)->format('M j, Y g:i A'),
                    'end' => $this->end_time->setTimezone($userTimezone)->format('M j, Y g:i A'),
                ],
                'timestamps' => [
                    'start' => $this->start_time->timestamp,
                    'end' => $this->end_time->timestamp,
                ],
            ],
            
            'timezone_info' => [
                'user_timezone' => $userTimezone,
                'original_timezone' => $this->timezone,
                'offset' => $this->start_time->setTimezone($userTimezone)->format('P'),
            ],
            
            'created_at' => $this->created_at->toISOString(),
            'updated_at' => $this->updated_at->toISOString(),
        ];
    }
}
```

### ২. API Response Transformer:

```php
<?php
// app/Http/Controllers/ApiBaseController.php
namespace App\Http\Controllers;

use Carbon\Carbon;

class ApiBaseController extends Controller
{
    protected function transformTimeResponse($data, $userTimezone = null)
    {
        $timezone = $userTimezone ?? request()->user_timezone ?? 'Asia/Dhaka';
        
        if (is_array($data)) {
            return array_map(function ($item) use ($timezone) {
                return $this->transformTimeResponse($item, $timezone);
            }, $data);
        }
        
        if (is_object($data) && method_exists($data, 'toArray')) {
            $array = $data->toArray();
            return $this->transformTimeResponse($array, $timezone);
        }
        
        return $data;
    }
    
    protected function timeResponse($utcTime, $userTimezone = null)
    {
        $timezone = $userTimezone ?? request()->user_timezone ?? 'Asia/Dhaka';
        $carbon = Carbon::parse($utcTime);
        
        return [
            'utc' => $carbon->utc()->toISOString(),
            'local' => $carbon->setTimezone($timezone)->toISOString(),
            'formatted' => $carbon->setTimezone($timezone)->format('M j, Y g:i A'),
            'timestamp' => $carbon->timestamp,
            'timezone' => $timezone,
        ];
    }
}

// Usage in controllers
class EventApiController extends ApiBaseController
{
    public function index(Request $request)
    {
        $events = Event::paginate(10);
        
        return response()->json([
            'events' => EventResource::collection($events),
            'meta' => [
                'current_time' => $this->timeResponse(now()),
                'user_timezone' => $request->user_timezone,
            ]
        ]);
    }
}
```

### ৩. Time-aware API Filters:

```php
<?php
class EventFilterController extends Controller
{
    public function filter(Request $request)
    {
        $request->validate([
            'date_from' => 'nullable|date',
            'date_to' => 'nullable|date|after_or_equal:date_from',
            'timezone' => 'nullable|string',
            'time_filter' => 'nullable|in:today,tomorrow,this_week,next_week,this_month',
        ]);
        
        $userTimezone = $request->timezone ?? $request->user_timezone ?? 'Asia/Dhaka';
        $query = Event::query();
        
        // Handle predefined time filters
        if ($request->time_filter) {
            [$startUTC, $endUTC] = $this->getTimeFilterRange($request->time_filter, $userTimezone);
            $query->whereBetween('start_time', [$startUTC, $endUTC]);
        }
        
        // Handle custom date range
        if ($request->date_from) {
            $startUTC = Carbon::parse($request->date_from, $userTimezone)->startOfDay()->utc();
            $query->where('start_time', '>=', $startUTC);
        }
        
        if ($request->date_to) {
            $endUTC = Carbon::parse($request->date_to, $userTimezone)->endOfDay()->utc();
            $query->where('start_time', '<=', $endUTC);
        }
        
        $events = $query->orderBy('start_time')->get();
        
        return response()->json([
            'events' => EventResource::collection($events),
            'filter_info' => [
                'timezone' => $userTimezone,
                'date_from' => $request->date_from,
                'date_to' => $request->date_to,
                'time_filter' => $request->time_filter,
            ]
        ]);
    }
    
    private function getTimeFilterRange($filter, $timezone)
    {
        $now = Carbon::now($timezone);
        
        switch ($filter) {
            case 'today':
                return [
                    $now->copy()->startOfDay()->utc(),
                    $now->copy()->endOfDay()->utc()
                ];
            
            case 'tomorrow':
                return [
                    $now->copy()->addDay()->startOfDay()->utc(),
                    $now->copy()->addDay()->endOfDay()->utc()
                ];
            
            case 'this_week':
                return [
                    $now->copy()->startOfWeek()->utc(),
                    $now->copy()->endOfWeek()->utc()
                ];
            
            case 'next_week':
                return [
                    $now->copy()->addWeek()->startOfWeek()->utc(),
                    $now->copy()->addWeek()->endOfWeek()->utc()
                ];
            
            case 'this_month':
                return [
                    $now->copy()->startOfMonth()->utc(),
                    $now->copy()->endOfMonth()->utc()
                ];
            
            default:
                return [null, null];
        }
    }
}
```

---

## Production Best Practices

### ১. Server Configuration:

```bash
# Server timezone should be UTC
sudo timedatectl set-timezone UTC

# MySQL timezone
mysql -u root -p
SET GLOBAL time_zone = '+00:00';
SET time_zone = '+00:00';

# PHP timezone (php.ini)
date.timezone = "UTC"

# Laravel .env
APP_TIMEZONE=UTC
DB_TIMEZONE=UTC
```

### ২. Performance Optimization:

```php
<?php
// Cache timezone data
class OptimizedTimezoneService
{
    public function getTimezoneOffset($timezone)
    {
        return Cache::remember("timezone_offset:{$timezone}", 3600, function () use ($timezone) {
            return Carbon::now($timezone)->format('P');
        });
    }
    
    public function convertBulkTimes($utcTimes, $userTimezone)
    {
        // Batch convert for better performance
        $offset = $this->getTimezoneOffset($userTimezone);
        
        return collect($utcTimes)->map(function ($utcTime) use ($userTimezone) {
            return Carbon::parse($utcTime)->setTimezone($userTimezone);
        });
    }
}

// Database indexes for timezone queries
Schema::table('events', function (Blueprint $table) {
    $table->index(['start_time', 'timezone']);
    $table->index(['user_id', 'start_time']);
});

// Eager loading with timezone conversion
class Event extends Model
{
    public function scopeWithUserTimezone($query, $timezone)
    {
        return $query->selectRaw("
            *,
            CONVERT_TZ(start_time, '+00:00', ?) as start_time_local
        ", [$this->getTimezoneOffset($timezone)]);
    }
}
```

### ৩. Error Handling:

```php
<?php
class TimezoneErrorHandler
{
    public static function safeParseTime($timeString, $timezone = null)
    {
        try {
            return Carbon::parse($timeString, $timezone);
        } catch (\Exception $e) {
            \Log::warning("Failed to parse time: {$timeString} with timezone: {$timezone}", [
                'error' => $e->getMessage()
            ]);
            
            // Fallback to UTC
            return Carbon::parse($timeString, 'UTC');
        }
    }
    
    public static function validateTimezone($timezone)
    {
        if (!in_array($timezone, timezone_identifiers_list())) {
            throw new \InvalidArgumentException("Invalid timezone: {$timezone}");
        }
        
        return true;
    }
}

// Global exception handler for timezone errors
class Handler extends ExceptionHandler
{
    public function render($request, Throwable $exception)
    {
        if ($exception instanceof \InvalidArgumentException && 
            str_contains($exception->getMessage(), 'timezone')) {
            
            return response()->json([
                'error' => 'Invalid timezone',
                'message' => 'Please provide a valid timezone identifier',
                'available_timezones' => array_keys(config('app.available_timezones'))
            ], 400);
        }
        
        return parent::render($request, $exception);
    }
}
```

### ৪. Testing Time Functions:

```php
<?php
// tests/Feature/TimezoneTest.php
namespace Tests\Feature;

use Tests\TestCase;
use Carbon\Carbon;

class TimezoneTest extends TestCase
{
    public function test_utc_storage()
    {
        // Freeze time for consistent testing
        Carbon::setTestNow('2024-01-15 08:30:00');
        
        $event = Event::create([
            'title' => 'Test Event',
            'start_time' => Carbon::parse('2024-01-15 14:30:00', 'Asia/Dhaka'),
        ]);
        
        // Should be stored as UTC
        $this->assertEquals('2024-01-15 08:30:00', $event->start_time->utc()->format('Y-m-d H:i:s'));
    }
    
    public function test_timezone_conversion()
    {
        $utcTime = Carbon::parse('2024-01-15 08:30:00', 'UTC');
        
        $dhakaTime = $utcTime->copy()->setTimezone('Asia/Dhaka');
        $newYorkTime = $utcTime->copy()->setTimezone('America/New_York');
        
        $this->assertEquals('14:30:00', $dhakaTime->format('H:i:s'));
        $this->assertEquals('03:30:00', $newYorkTime->format('H:i:s'));
    }
    
    public function test_api_timezone_header()
    {
        $response = $this->withHeaders([
            'X-Timezone' => 'Asia/Dhaka'
        ])->get('/api/events');
        
        $response->assertStatus(200)
                ->assertHeader('X-User-Timezone', 'Asia/Dhaka');
    }
}
```

---

## সারসংক্ষেপ

### 🕐 Time Management Best Practices:
- ✅ সবসময় UTC তে database এ store করুন
- ✅ User এর timezone track করুন
- ✅ API response এ multiple time format দিন
- ✅ Carbon library ব্যবহার করুন
- ✅ Timezone validation implement করুন

### 📊 Key Points:
```php
// Storage: Always UTC
$event->start_time = Carbon::parse($userTime, $userTimezone)->utc();

// Display: User's timezone  
$displayTime = $event->start_time->setTimezone($userTimezone);

// API: Multiple formats
return [
    'utc' => $time->utc()->toISOString(),
    'local' => $time->setTimezone($userTimezone)->toISOString(),
    'formatted' => $time->setTimezone($userTimezone)->format('M j, Y g:i A'),
];
```

এই গাইড Laravel এ professional time management system তৈরি করতে সাহায্য করবে।