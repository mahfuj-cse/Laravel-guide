# Laravel Repository Pattern - সম্পূর্ণ বাংলা গাইড

## 📋 সূচিপত্র
- [Repository Pattern কি?](#repository-pattern-কি)
- [কেন Repository Pattern ব্যবহার করবেন?](#কেন-repository-pattern-ব্যবহার-করবেন)
- [Advantages & Disadvantages](#advantages--disadvantages)
- [কখন ব্যবহার করবেন?](#কখন-ব্যবহার-করবেন)
- [Implementation Step by Step](#implementation-step-by-step)
- [Call Flow & Architecture](#call-flow--architecture)
- [Advanced Implementation](#advanced-implementation)
- [Best Practices](#best-practices)
- [Testing](#testing)

---

## Repository Pattern কি?

**Repository Pattern** হলো একটি **Design Pattern** যা **Data Access Layer** এবং **Business Logic Layer** এর মধ্যে **abstraction** তৈরি করে। এটি **database operations** কে **centralize** করে এবং **clean architecture** maintain করতে সাহায্য করে।

### 🎯 সহজ ভাষায়:
Repository Pattern হলো একটি **"Data Store"** এর মতো যেখানে:
- সব **database queries** একসাথে থাকে
- **Controller** থেকে **direct database access** remove করা হয়
- **Business logic** এবং **data access** আলাদা করা হয়
- **Testing** এবং **maintenance** সহজ হয়

### 🏗️ Architecture Overview:
```
Client Request → Controller → Service → Repository → Model → Database
                    ↓           ↓          ↓         ↓
                Business    Business   Data      Eloquent
                Logic       Rules      Access    ORM
```

---

## কেন Repository Pattern ব্যবহার করবেন?

### ❌ Without Repository Pattern:
```php
// Controller এ direct database access (Bad Practice)
class UserController extends Controller
{
    public function index()
    {
        $users = User::with('posts')
                    ->where('status', 'active')
                    ->orderBy('created_at', 'desc')
                    ->paginate(10);
        
        return view('users.index', compact('users'));
    }
    
    public function store(Request $request)
    {
        $user = User::create([
            'name' => $request->name,
            'email' => $request->email,
            'password' => bcrypt($request->password),
        ]);
        
        // Send welcome email
        Mail::to($user)->send(new WelcomeEmail($user));
        
        return redirect()->route('users.index');
    }
    
    public function getActiveUsers()
    {
        return User::where('status', 'active')
                  ->where('email_verified_at', '!=', null)
                  ->get();
    }
}
```

**সমস্যাসমূহ:**
- 🚨 **Controller** এ database logic scattered
- 🚨 **Code duplication** (same query multiple places)
- 🚨 **Hard to test** (direct database dependency)
- 🚨 **Tight coupling** between controller and model
- 🚨 **Business logic mixed** with data access

### ✅ With Repository Pattern:
```php
// Clean Controller
class UserController extends Controller
{
    protected $userRepository;
    
    public function __construct(UserRepositoryInterface $userRepository)
    {
        $this->userRepository = $userRepository;
    }
    
    public function index()
    {
        $users = $this->userRepository->getActiveUsersWithPosts();
        return view('users.index', compact('users'));
    }
    
    public function store(Request $request)
    {
        $user = $this->userRepository->create($request->validated());
        
        // Business logic in service
        app(UserService::class)->sendWelcomeEmail($user);
        
        return redirect()->route('users.index');
    }
}

// Repository handles all data access
class UserRepository implements UserRepositoryInterface
{
    public function getActiveUsersWithPosts()
    {
        return User::with('posts')
                  ->where('status', 'active')
                  ->orderBy('created_at', 'desc')
                  ->paginate(10);
    }
    
    public function create(array $data)
    {
        $data['password'] = bcrypt($data['password']);
        return User::create($data);
    }
}
```

---

## Advantages & Disadvantages

### ✅ **Advantages (সুবিধাসমূহ):**

#### 1. **Separation of Concerns**
```php
// Controller: HTTP handling
// Service: Business logic
// Repository: Data access
// Model: Data representation

class UserController extends Controller
{
    // শুধু HTTP request/response handle করে
}

class UserService
{
    // শুধু business logic handle করে
}

class UserRepository
{
    // শুধু database operations handle করে
}
```

#### 2. **Code Reusability**
```php
class UserRepository
{
    public function getActiveUsers()
    {
        return User::where('status', 'active')->get();
    }
}

// Multiple places এ reuse করা যায়
class UserController
{
    public function index()
    {
        $users = $this->userRepository->getActiveUsers();
    }
}

class DashboardController
{
    public function stats()
    {
        $activeUsers = $this->userRepository->getActiveUsers();
    }
}
```

#### 3. **Easy Testing**
```php
// Mock repository for testing
class UserControllerTest extends TestCase
{
    public function test_index_returns_active_users()
    {
        $mockRepo = Mockery::mock(UserRepositoryInterface::class);
        $mockRepo->shouldReceive('getActiveUsers')
                ->once()
                ->andReturn(collect(['user1', 'user2']));
        
        $this->app->instance(UserRepositoryInterface::class, $mockRepo);
        
        $response = $this->get('/users');
        $response->assertStatus(200);
    }
}
```

#### 4. **Database Independence**
```php
// MySQL Repository
class MySqlUserRepository implements UserRepositoryInterface
{
    public function findById($id)
    {
        return User::find($id);
    }
}

// MongoDB Repository
class MongoUserRepository implements UserRepositoryInterface
{
    public function findById($id)
    {
        return $this->collection->findOne(['_id' => $id]);
    }
}

// Same interface, different implementation
```

#### 5. **Centralized Query Logic**
```php
class UserRepository
{
    public function getActiveUsers()
    {
        return User::where('status', 'active')
                  ->where('email_verified_at', '!=', null)
                  ->get();
    }
    
    public function getPremiumUsers()
    {
        return User::where('subscription_type', 'premium')
                  ->where('subscription_expires_at', '>', now())
                  ->get();
    }
    
    // Complex queries centralized
    public function getUsersWithRecentActivity()
    {
        return User::whereHas('posts', function ($query) {
                    $query->where('created_at', '>', now()->subDays(30));
                })
                ->orWhereHas('comments', function ($query) {
                    $query->where('created_at', '>', now()->subDays(30));
                })
                ->get();
    }
}
```

### ❌ **Disadvantages (অসুবিধাসমূহ):**

#### 1. **Over-Engineering**
```php
// Simple CRUD এর জন্য অতিরিক্ত complexity
class UserRepository
{
    public function all()
    {
        return User::all(); // Simple query এর জন্য extra layer
    }
}
```

#### 2. **More Code to Write**
```php
// Interface + Implementation + Service Provider registration
interface UserRepositoryInterface { }
class UserRepository implements UserRepositoryInterface { }
// AppServiceProvider এ binding
```

#### 3. **Learning Curve**
- নতুন developers এর জন্য **additional concepts**
- **Dependency Injection** এবং **Interface** এর knowledge প্রয়োজন

#### 4. **Performance Overhead**
```php
// Extra method calls
Controller → Service → Repository → Model → Database
// Direct access এর চেয়ে slightly slower
```

---

## কখন ব্যবহার করবেন?

### ✅ **ব্যবহার করুন যখন:**

#### 1. **Large Applications**
```php
// Complex business logic
// Multiple developers working
// Long-term maintenance required
```

#### 2. **Complex Database Operations**
```php
class OrderRepository
{
    public function getOrdersWithComplexFilters($filters)
    {
        $query = Order::with(['items', 'customer', 'payments']);
        
        if ($filters['date_range']) {
            $query->whereBetween('created_at', $filters['date_range']);
        }
        
        if ($filters['status']) {
            $query->whereIn('status', $filters['status']);
        }
        
        if ($filters['customer_type']) {
            $query->whereHas('customer', function ($q) use ($filters) {
                $q->where('type', $filters['customer_type']);
            });
        }
        
        return $query->paginate(20);
    }
}
```

#### 3. **Multiple Data Sources**
```php
// Same interface for different data sources
interface UserRepositoryInterface
{
    public function findById($id);
}

class DatabaseUserRepository implements UserRepositoryInterface
{
    public function findById($id)
    {
        return User::find($id);
    }
}

class ApiUserRepository implements UserRepositoryInterface
{
    public function findById($id)
    {
        return Http::get("api/users/{$id}")->json();
    }
}
```

#### 4. **Heavy Testing Requirements**
```php
// Easy to mock and test
class UserService
{
    public function __construct(UserRepositoryInterface $userRepo)
    {
        $this->userRepo = $userRepo;
    }
    
    public function createUser($data)
    {
        // Business logic
        if ($this->userRepo->emailExists($data['email'])) {
            throw new Exception('Email already exists');
        }
        
        return $this->userRepo->create($data);
    }
}
```

### ❌ **ব্যবহার করবেন না যখন:**

#### 1. **Simple CRUD Applications**
```php
// Simple blog, portfolio sites
// Basic user management
// Prototype/MVP development
```

#### 2. **Small Team/Solo Development**
```php
// Rapid development needed
// Simple business logic
// Short-term projects
```

#### 3. **Learning Phase**
```php
// Laravel beginners
// Focus on core concepts first
// Simple applications
```

---

## Implementation Step by Step

### Step 1: Create Repository Interface
```php
<?php
// app/Repositories/Interfaces/UserRepositoryInterface.php

namespace App\Repositories\Interfaces;

interface UserRepositoryInterface
{
    public function all();
    public function find($id);
    public function create(array $data);
    public function update($id, array $data);
    public function delete($id);
    public function getActiveUsers();
    public function findByEmail($email);
}
```

### Step 2: Create Repository Implementation
```php
<?php
// app/Repositories/UserRepository.php

namespace App\Repositories;

use App\Models\User;
use App\Repositories\Interfaces\UserRepositoryInterface;

class UserRepository implements UserRepositoryInterface
{
    protected $model;
    
    public function __construct(User $model)
    {
        $this->model = $model;
    }
    
    public function all()
    {
        return $this->model->all();
    }
    
    public function find($id)
    {
        return $this->model->findOrFail($id);
    }
    
    public function create(array $data)
    {
        return $this->model->create($data);
    }
    
    public function update($id, array $data)
    {
        $user = $this->find($id);
        $user->update($data);
        return $user;
    }
    
    public function delete($id)
    {
        $user = $this->find($id);
        return $user->delete();
    }
    
    public function getActiveUsers()
    {
        return $this->model->where('status', 'active')->get();
    }
    
    public function findByEmail($email)
    {
        return $this->model->where('email', $email)->first();
    }
}
```

### Step 3: Register in Service Provider
```php
<?php
// app/Providers/RepositoryServiceProvider.php

namespace App\Providers;

use Illuminate\Support\ServiceProvider;
use App\Repositories\Interfaces\UserRepositoryInterface;
use App\Repositories\UserRepository;

class RepositoryServiceProvider extends ServiceProvider
{
    public function register()
    {
        $this->app->bind(
            UserRepositoryInterface::class,
            UserRepository::class
        );
    }
    
    public function boot()
    {
        //
    }
}
```

### Step 4: Register Service Provider
```php
// config/app.php
'providers' => [
    // Other providers...
    App\Providers\RepositoryServiceProvider::class,
],
```

### Step 5: Use in Controller
```php
<?php
// app/Http/Controllers/UserController.php

namespace App\Http\Controllers;

use Illuminate\Http\Request;
use App\Repositories\Interfaces\UserRepositoryInterface;

class UserController extends Controller
{
    protected $userRepository;
    
    public function __construct(UserRepositoryInterface $userRepository)
    {
        $this->userRepository = $userRepository;
    }
    
    public function index()
    {
        $users = $this->userRepository->all();
        return view('users.index', compact('users'));
    }
    
    public function show($id)
    {
        $user = $this->userRepository->find($id);
        return view('users.show', compact('user'));
    }
    
    public function store(Request $request)
    {
        $user = $this->userRepository->create($request->validated());
        return redirect()->route('users.index')
                        ->with('success', 'User created successfully');
    }
    
    public function update(Request $request, $id)
    {
        $user = $this->userRepository->update($id, $request->validated());
        return redirect()->route('users.show', $user)
                        ->with('success', 'User updated successfully');
    }
    
    public function destroy($id)
    {
        $this->userRepository->delete($id);
        return redirect()->route('users.index')
                        ->with('success', 'User deleted successfully');
    }
}
```

---

## Call Flow & Architecture

### 📊 Complete Call Flow:

```
1. HTTP Request
   ↓
2. Route
   ↓
3. Controller (Dependency Injection)
   ↓
4. Service (Business Logic) [Optional]
   ↓
5. Repository (Data Access)
   ↓
6. Model (Eloquent ORM)
   ↓
7. Database
   ↓
8. Response (JSON/View)
```

### 🔍 Detailed Flow Example:

#### Request: `GET /users/1`

```php
// 1. Route
Route::get('/users/{id}', [UserController::class, 'show']);

// 2. Controller Constructor (Dependency Injection)
class UserController extends Controller
{
    public function __construct(UserRepositoryInterface $userRepository)
    {
        $this->userRepository = $userRepository; // Laravel resolves this
    }
    
    // 3. Controller Method
    public function show($id)
    {
        $user = $this->userRepository->find($id); // Call repository
        return view('users.show', compact('user'));
    }
}

// 4. Repository Method
class UserRepository implements UserRepositoryInterface
{
    public function find($id)
    {
        return $this->model->findOrFail($id); // Call Eloquent
    }
}

// 5. Eloquent Model
class User extends Model
{
    // Eloquent handles database query
    // SELECT * FROM users WHERE id = ? LIMIT 1
}
```

### 🏗️ Architecture Layers:

#### Layer 1: Presentation Layer (Controller)
```php
class UserController extends Controller
{
    // Handles HTTP requests/responses
    // Validates input
    // Returns views/JSON
    
    public function index()
    {
        $users = $this->userRepository->getActiveUsers();
        return UserResource::collection($users);
    }
}
```

#### Layer 2: Business Logic Layer (Service)
```php
class UserService
{
    // Contains business rules
    // Orchestrates multiple repositories
    // Handles complex operations
    
    public function createUserWithProfile($userData, $profileData)
    {
        DB::transaction(function () use ($userData, $profileData) {
            $user = $this->userRepository->create($userData);
            $this->profileRepository->create($profileData + ['user_id' => $user->id]);
            $this->notificationService->sendWelcomeEmail($user);
        });
    }
}
```

#### Layer 3: Data Access Layer (Repository)
```php
class UserRepository implements UserRepositoryInterface
{
    // Handles database operations
    // Encapsulates query logic
    // Returns models/collections
    
    public function getActiveUsers()
    {
        return $this->model->where('status', 'active')->get();
    }
}
```

#### Layer 4: Data Layer (Model)
```php
class User extends Model
{
    // Represents database table
    // Defines relationships
    // Handles data casting
    
    public function posts()
    {
        return $this->hasMany(Post::class);
    }
}
```

---

## Advanced Implementation

### 1. Base Repository Pattern
```php
<?php
// app/Repositories/BaseRepository.php

namespace App\Repositories;

use Illuminate\Database\Eloquent\Model;

abstract class BaseRepository
{
    protected $model;
    
    public function __construct(Model $model)
    {
        $this->model = $model;
    }
    
    public function all()
    {
        return $this->model->all();
    }
    
    public function find($id)
    {
        return $this->model->findOrFail($id);
    }
    
    public function create(array $data)
    {
        return $this->model->create($data);
    }
    
    public function update($id, array $data)
    {
        $record = $this->find($id);
        $record->update($data);
        return $record;
    }
    
    public function delete($id)
    {
        return $this->find($id)->delete();
    }
    
    public function where($column, $operator, $value)
    {
        return $this->model->where($column, $operator, $value);
    }
    
    public function with($relations)
    {
        return $this->model->with($relations);
    }
}
```

### 2. Extended User Repository
```php
<?php
// app/Repositories/UserRepository.php

namespace App\Repositories;

use App\Models\User;
use App\Repositories\Interfaces\UserRepositoryInterface;

class UserRepository extends BaseRepository implements UserRepositoryInterface
{
    public function __construct(User $model)
    {
        parent::__construct($model);
    }
    
    public function getActiveUsers()
    {
        return $this->model->where('status', 'active')->get();
    }
    
    public function findByEmail($email)
    {
        return $this->model->where('email', $email)->first();
    }
    
    public function getUsersWithPosts()
    {
        return $this->model->with('posts')->get();
    }
    
    public function searchUsers($query)
    {
        return $this->model->where('name', 'like', "%{$query}%")
                          ->orWhere('email', 'like', "%{$query}%")
                          ->get();
    }
    
    public function getUsersByRole($role)
    {
        return $this->model->whereHas('roles', function ($q) use ($role) {
            $q->where('name', $role);
        })->get();
    }
    
    public function getRecentUsers($days = 30)
    {
        return $this->model->where('created_at', '>=', now()->subDays($days))
                          ->orderBy('created_at', 'desc')
                          ->get();
    }
}
```

### 3. Service Layer Integration
```php
<?php
// app/Services/UserService.php

namespace App\Services;

use App\Repositories\Interfaces\UserRepositoryInterface;
use App\Repositories\Interfaces\ProfileRepositoryInterface;
use Illuminate\Support\Facades\DB;
use Illuminate\Support\Facades\Hash;

class UserService
{
    protected $userRepository;
    protected $profileRepository;
    
    public function __construct(
        UserRepositoryInterface $userRepository,
        ProfileRepositoryInterface $profileRepository
    ) {
        $this->userRepository = $userRepository;
        $this->profileRepository = $profileRepository;
    }
    
    public function createUserWithProfile(array $userData, array $profileData)
    {
        return DB::transaction(function () use ($userData, $profileData) {
            // Hash password
            $userData['password'] = Hash::make($userData['password']);
            
            // Create user
            $user = $this->userRepository->create($userData);
            
            // Create profile
            $profileData['user_id'] = $user->id;
            $profile = $this->profileRepository->create($profileData);
            
            // Send welcome email
            $this->sendWelcomeEmail($user);
            
            return $user->load('profile');
        });
    }
    
    public function updateUserProfile($userId, array $userData, array $profileData)
    {
        return DB::transaction(function () use ($userId, $userData, $profileData) {
            $user = $this->userRepository->update($userId, $userData);
            
            if ($user->profile) {
                $this->profileRepository->update($user->profile->id, $profileData);
            } else {
                $profileData['user_id'] = $userId;
                $this->profileRepository->create($profileData);
            }
            
            return $user->load('profile');
        });
    }
    
    public function deactivateUser($userId)
    {
        $user = $this->userRepository->find($userId);
        
        // Business logic
        if ($user->hasActiveSubscription()) {
            throw new \Exception('Cannot deactivate user with active subscription');
        }
        
        return $this->userRepository->update($userId, ['status' => 'inactive']);
    }
    
    private function sendWelcomeEmail($user)
    {
        // Email sending logic
        \Mail::to($user)->send(new \App\Mail\WelcomeEmail($user));
    }
}
```

### 4. Advanced Controller
```php
<?php
// app/Http/Controllers/UserController.php

namespace App\Http\Controllers;

use Illuminate\Http\Request;
use App\Services\UserService;
use App\Repositories\Interfaces\UserRepositoryInterface;
use App\Http\Requests\CreateUserRequest;
use App\Http\Requests\UpdateUserRequest;
use App\Http\Resources\UserResource;

class UserController extends Controller
{
    protected $userRepository;
    protected $userService;
    
    public function __construct(
        UserRepositoryInterface $userRepository,
        UserService $userService
    ) {
        $this->userRepository = $userRepository;
        $this->userService = $userService;
    }
    
    public function index(Request $request)
    {
        if ($request->has('search')) {
            $users = $this->userRepository->searchUsers($request->search);
        } elseif ($request->has('role')) {
            $users = $this->userRepository->getUsersByRole($request->role);
        } else {
            $users = $this->userRepository->getActiveUsers();
        }
        
        return UserResource::collection($users);
    }
    
    public function store(CreateUserRequest $request)
    {
        $userData = $request->only(['name', 'email', 'password']);
        $profileData = $request->only(['phone', 'address', 'bio']);
        
        $user = $this->userService->createUserWithProfile($userData, $profileData);
        
        return new UserResource($user);
    }
    
    public function show($id)
    {
        $user = $this->userRepository->find($id);
        return new UserResource($user);
    }
    
    public function update(UpdateUserRequest $request, $id)
    {
        $userData = $request->only(['name', 'email']);
        $profileData = $request->only(['phone', 'address', 'bio']);
        
        $user = $this->userService->updateUserProfile($id, $userData, $profileData);
        
        return new UserResource($user);
    }
    
    public function destroy($id)
    {
        $this->userService->deactivateUser($id);
        
        return response()->json(['message' => 'User deactivated successfully']);
    }
    
    public function recent()
    {
        $users = $this->userRepository->getRecentUsers();
        return UserResource::collection($users);
    }
}
```

---

## Best Practices

### 1. **Interface Segregation**
```php
// ❌ Fat Interface
interface UserRepositoryInterface
{
    public function all();
    public function find($id);
    public function create(array $data);
    public function getActiveUsers();
    public function getInactiveUsers();
    public function getPremiumUsers();
    public function getAdminUsers();
    // ... 20 more methods
}

// ✅ Segregated Interfaces
interface UserRepositoryInterface
{
    public function all();
    public function find($id);
    public function create(array $data);
    public function update($id, array $data);
    public function delete($id);
}

interface UserQueryRepositoryInterface
{
    public function getActiveUsers();
    public function searchUsers($query);
    public function getUsersByRole($role);
}
```

### 2. **Repository Naming Convention**
```php
// ✅ Good naming
class UserRepository implements UserRepositoryInterface
{
    public function findById($id) { }
    public function findByEmail($email) { }
    public function getActiveUsers() { }
    public function createUser(array $data) { }
    public function updateUser($id, array $data) { }
    public function deleteUser($id) { }
}
```

### 3. **Error Handling**
```php
class UserRepository implements UserRepositoryInterface
{
    public function find($id)
    {
        try {
            return $this->model->findOrFail($id);
        } catch (ModelNotFoundException $e) {
            throw new UserNotFoundException("User with ID {$id} not found");
        }
    }
    
    public function create(array $data)
    {
        try {
            return $this->model->create($data);
        } catch (QueryException $e) {
            if ($e->getCode() === '23000') { // Duplicate entry
                throw new DuplicateUserException('User already exists');
            }
            throw $e;
        }
    }
}
```

### 4. **Caching Integration**
```php
class UserRepository implements UserRepositoryInterface
{
    public function find($id)
    {
        return Cache::remember("user.{$id}", 3600, function () use ($id) {
            return $this->model->findOrFail($id);
        });
    }
    
    public function create(array $data)
    {
        $user = $this->model->create($data);
        
        // Clear related caches
        Cache::forget('users.active');
        Cache::forget('users.count');
        
        return $user;
    }
    
    public function getActiveUsers()
    {
        return Cache::remember('users.active', 1800, function () {
            return $this->model->where('status', 'active')->get();
        });
    }
}
```

### 5. **Query Optimization**
```php
class UserRepository implements UserRepositoryInterface
{
    public function getUsersWithPosts($limit = 10)
    {
        return $this->model->with(['posts' => function ($query) {
                    $query->latest()->limit(5);
                }])
                ->withCount('posts')
                ->limit($limit)
                ->get();
    }
    
    public function searchUsersOptimized($query)
    {
        return $this->model->select(['id', 'name', 'email', 'avatar'])
                          ->where('name', 'like', "%{$query}%")
                          ->orWhere('email', 'like', "%{$query}%")
                          ->limit(20)
                          ->get();
    }
}
```

---

## Testing

### 1. **Repository Testing**
```php
<?php
// tests/Unit/UserRepositoryTest.php

namespace Tests\Unit;

use Tests\TestCase;
use App\Models\User;
use App\Repositories\UserRepository;
use Illuminate\Foundation\Testing\RefreshDatabase;

class UserRepositoryTest extends TestCase
{
    use RefreshDatabase;
    
    protected $userRepository;
    
    protected function setUp(): void
    {
        parent::setUp();
        $this->userRepository = new UserRepository(new User);
    }
    
    public function test_can_create_user()
    {
        $userData = [
            'name' => 'John Doe',
            'email' => 'john@example.com',
            'password' => 'password123'
        ];
        
        $user = $this->userRepository->create($userData);
        
        $this->assertInstanceOf(User::class, $user);
        $this->assertEquals('John Doe', $user->name);
        $this->assertDatabaseHas('users', ['email' => 'john@example.com']);
    }
    
    public function test_can_find_user_by_id()
    {
        $user = User::factory()->create();
        
        $foundUser = $this->userRepository->find($user->id);
        
        $this->assertEquals($user->id, $foundUser->id);
    }
    
    public function test_can_get_active_users()
    {
        User::factory()->count(3)->create(['status' => 'active']);
        User::factory()->count(2)->create(['status' => 'inactive']);
        
        $activeUsers = $this->userRepository->getActiveUsers();
        
        $this->assertCount(3, $activeUsers);
        $activeUsers->each(function ($user) {
            $this->assertEquals('active', $user->status);
        });
    }
}
```

### 2. **Controller Testing with Mocked Repository**
```php
<?php
// tests/Feature/UserControllerTest.php

namespace Tests\Feature;

use Tests\TestCase;
use App\Models\User;
use App\Repositories\Interfaces\UserRepositoryInterface;
use Mockery;

class UserControllerTest extends TestCase
{
    public function test_index_returns_users()
    {
        $users = collect([
            new User(['id' => 1, 'name' => 'John']),
            new User(['id' => 2, 'name' => 'Jane']),
        ]);
        
        $mockRepository = Mockery::mock(UserRepositoryInterface::class);
        $mockRepository->shouldReceive('getActiveUsers')
                      ->once()
                      ->andReturn($users);
        
        $this->app->instance(UserRepositoryInterface::class, $mockRepository);
        
        $response = $this->get('/api/users');
        
        $response->assertStatus(200)
                ->assertJsonCount(2, 'data');
    }
    
    public function test_store_creates_user()
    {
        $userData = [
            'name' => 'John Doe',
            'email' => 'john@example.com',
            'password' => 'password123'
        ];
        
        $user = new User($userData + ['id' => 1]);
        
        $mockRepository = Mockery::mock(UserRepositoryInterface::class);
        $mockRepository->shouldReceive('create')
                      ->once()
                      ->with($userData)
                      ->andReturn($user);
        
        $this->app->instance(UserRepositoryInterface::class, $mockRepository);
        
        $response = $this->postJson('/api/users', $userData);
        
        $response->assertStatus(201)
                ->assertJson(['data' => ['name' => 'John Doe']]);
    }
}
```

### 3. **Service Testing**
```php
<?php
// tests/Unit/UserServiceTest.php

namespace Tests\Unit;

use Tests\TestCase;
use App\Services\UserService;
use App\Repositories\Interfaces\UserRepositoryInterface;
use App\Repositories\Interfaces\ProfileRepositoryInterface;
use Mockery;

class UserServiceTest extends TestCase
{
    public function test_create_user_with_profile()
    {
        $userData = ['name' => 'John', 'email' => 'john@example.com'];
        $profileData = ['phone' => '123456789'];
        
        $user = new \App\Models\User($userData + ['id' => 1]);
        $profile = new \App\Models\Profile($profileData + ['id' => 1, 'user_id' => 1]);
        
        $userRepo = Mockery::mock(UserRepositoryInterface::class);
        $profileRepo = Mockery::mock(ProfileRepositoryInterface::class);
        
        $userRepo->shouldReceive('create')->once()->andReturn($user);
        $profileRepo->shouldReceive('create')->once()->andReturn($profile);
        
        $service = new UserService($userRepo, $profileRepo);
        
        $result = $service->createUserWithProfile($userData, $profileData);
        
        $this->assertInstanceOf(\App\Models\User::class, $result);
    }
}
```

---

## সারসংক্ষেপ

### 🎯 **কখন Repository Pattern ব্যবহার করবেন:**

**✅ ব্যবহার করুন:**
- Large/Complex applications
- Team development
- Heavy testing requirements
- Multiple data sources
- Long-term maintenance

**❌ ব্যবহার করবেন না:**
- Simple CRUD applications
- Rapid prototyping
- Learning phase
- Small projects

### 🏗️ **Implementation Checklist:**

1. ✅ Create Repository Interface
2. ✅ Implement Repository Class
3. ✅ Register in Service Provider
4. ✅ Use Dependency Injection in Controller
5. ✅ Add Service Layer (optional)
6. ✅ Write Tests
7. ✅ Add Error Handling
8. ✅ Optimize Queries

### 📊 **Architecture Benefits:**
- **Separation of Concerns** - Clean code organization
- **Testability** - Easy mocking and testing
- **Maintainability** - Centralized data access
- **Flexibility** - Easy to change data sources
- **Reusability** - Share repository methods

Repository Pattern সঠিকভাবে implement করলে আপনার Laravel application **scalable, maintainable এবং testable** হবে। তবে **over-engineering** এড়িয়ে **project requirements** অনুযায়ী ব্যবহার করুন।