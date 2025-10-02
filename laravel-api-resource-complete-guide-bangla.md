# Laravel API Resource - সম্পূর্ণ বাংলা গাইড

## 📋 সূচিপত্র
- [API Resource কি?](#api-resource-কি)
- [Data Flow কিভাবে কাজ করে](#data-flow-কিভাবে-কাজ-করে)
- [Basic API Resource](#basic-api-resource)
- [Resource Collection](#resource-collection)
- [Conditional Loading](#conditional-loading)
- [Relationship Loading](#relationship-loading)
- [Performance Optimization](#performance-optimization)
- [Advanced Techniques](#advanced-techniques)
- [Best Practices](#best-practices)

---

## API Resource কি?

**Laravel API Resource** হলো একটি **transformation layer** যা **Eloquent models** কে **JSON response** এ convert করে। এটি **data presentation** এবং **API response structure** control করার জন্য ব্যবহৃত হয়।

### 🎯 সহজ ভাষায়:
- **Model Data** → **API Resource** → **JSON Response**
- Database থেকে আসা raw data কে **formatted JSON** এ রূপান্তর
- **Consistent API structure** maintain করা
- **Sensitive data** hide করা এবং **additional data** add করা

### 🔄 Without vs With API Resource:

```php
// ❌ Without API Resource (Direct Model Return)
class UserController extends Controller
{
    public function index()
    {
        $users = User::all();
        return response()->json($users); // Raw model data
    }
}

// Response: Exposes all model attributes including sensitive data
{
    "id": 1,
    "name": "John Doe",
    "email": "john@example.com",
    "password": "$2y$10$...", // 🚨 Sensitive data exposed
    "remember_token": "abc123...",
    "created_at": "2024-01-15T10:30:00.000000Z",
    "updated_at": "2024-01-15T10:30:00.000000Z"
}
```

```php
// ✅ With API Resource (Controlled Response)
class UserController extends Controller
{
    public function index()
    {
        $users = User::all();
        return UserResource::collection($users); // Formatted response
    }
}

// Response: Clean, controlled structure
{
    "id": 1,
    "name": "John Doe",
    "email": "john@example.com",
    "joined_date": "January 15, 2024",
    "is_active": true
}
```

---

## Data Flow কিভাবে কাজ করে

### 📊 Complete Data Flow Diagram:

```
Database → Model → Controller → API Resource → JSON Response → Client
    ↓         ↓         ↓            ↓              ↓           ↓
  Raw Data  Eloquent  Query      Transform      Format     Frontend
            Object    Results     Data          Response    Display
```

### 🔍 Step by Step Data Flow:

#### Step 1: Database Query (Controller)
```php
class UserController extends Controller
{
    public function index()
    {
        // 1. Database query with relationships
        $users = User::with(['posts', 'profile'])
                    ->withCount('posts')
                    ->get();
        
        // 2. Pass to API Resource
        return UserResource::collection($users);
    }
    
    public function show(User $user)
    {
        // 1. Load specific relationships
        $user->load(['posts.comments', 'profile']);
        
        // 2. Pass single model to resource
        return new UserResource($user);
    }
}
```

#### Step 2: API Resource Transformation
```php
class UserResource extends JsonResource
{
    public function toArray($request)
    {
        // 3. Transform model data
        return [
            'id' => $this->id,
            'name' => $this->name,
            'email' => $this->email,
            
            // 4. Add computed fields
            'full_name' => $this->first_name . ' ' . $this->last_name,
            'avatar_url' => $this->avatar ? asset($this->avatar) : null,
            
            // 5. Format dates
            'joined_date' => $this->created_at->format('F j, Y'),
            'last_active' => $this->last_login_at?->diffForHumans(),
            
            // 6. Conditional data loading
            'posts_count' => $this->when(isset($this->posts_count), $this->posts_count),
            'posts' => PostResource::collection($this->whenLoaded('posts')),
            'profile' => new ProfileResource($this->whenLoaded('profile')),
        ];
    }
}
```

#### Step 3: JSON Response Generation
```php
// Laravel automatically converts resource to JSON
{
    "data": [
        {
            "id": 1,
            "name": "John Doe",
            "email": "john@example.com",
            "full_name": "John Doe",
            "avatar_url": "https://example.com/avatars/john.jpg",
            "joined_date": "January 15, 2024",
            "last_active": "2 hours ago",
            "posts_count": 5,
            "posts": [...],
            "profile": {...}
        }
    ]
}
```

---

## Basic API Resource

### 1. Creating API Resource:
```bash
# Single resource
php artisan make:resource UserResource

# Resource collection
php artisan make:resource UserCollection

# Both resource and collection
php artisan make:resource User --collection
```

### 2. Basic Resource Structure:
```php
<?php
// app/Http/Resources/UserResource.php

namespace App\Http\Resources;

use Illuminate\Http\Resources\Json\JsonResource;

class UserResource extends JsonResource
{
    /**
     * Transform the resource into an array.
     */
    public function toArray($request)
    {
        return [
            // Basic fields
            'id' => $this->id,
            'name' => $this->name,
            'email' => $this->email,
            
            // Formatted fields
            'created_at' => $this->created_at->toDateTimeString(),
            'updated_at' => $this->updated_at->toDateTimeString(),
        ];
    }
}
```

### 3. Using in Controller:
```php
class UserController extends Controller
{
    // Single resource
    public function show(User $user)
    {
        return new UserResource($user);
    }
    
    // Resource collection
    public function index()
    {
        $users = User::paginate(10);
        return UserResource::collection($users);
    }
}
```

---

## Resource Collection

### 1. Custom Collection Class:
```php
<?php
// app/Http/Resources/UserCollection.php

namespace App\Http\Resources;

use Illuminate\Http\Resources\Json\ResourceCollection;

class UserCollection extends ResourceCollection
{
    /**
     * Transform the resource collection into an array.
     */
    public function toArray($request)
    {
        return [
            'data' => $this->collection,
            'meta' => [
                'total_users' => $this->collection->count(),
                'active_users' => $this->collection->where('is_active', true)->count(),
                'generated_at' => now()->toDateTimeString(),
            ],
            'links' => [
                'self' => request()->url(),
            ],
        ];
    }
}
```

### 2. Collection with Additional Data:
```php
class UserCollection extends ResourceCollection
{
    public function toArray($request)
    {
        return [
            'users' => UserResource::collection($this->collection),
            'statistics' => [
                'total' => $this->collection->count(),
                'verified' => $this->collection->whereNotNull('email_verified_at')->count(),
                'premium' => $this->collection->where('is_premium', true)->count(),
            ],
            'filters_applied' => $request->only(['search', 'status', 'role']),
        ];
    }
    
    public function with($request)
    {
        return [
            'version' => '1.0',
            'api_documentation' => 'https://api.example.com/docs',
        ];
    }
}
```

### 3. Pagination with Collections:
```php
class UserController extends Controller
{
    public function index(Request $request)
    {
        $users = User::when($request->search, function ($query, $search) {
                    $query->where('name', 'like', "%{$search}%");
                })
                ->when($request->status, function ($query, $status) {
                    $query->where('status', $status);
                })
                ->paginate(15);
        
        return new UserCollection($users);
    }
}

// Response includes pagination meta
{
    "data": [...],
    "links": {
        "first": "http://example.com/users?page=1",
        "last": "http://example.com/users?page=10",
        "prev": null,
        "next": "http://example.com/users?page=2"
    },
    "meta": {
        "current_page": 1,
        "from": 1,
        "last_page": 10,
        "per_page": 15,
        "to": 15,
        "total": 150
    }
}
```

---

## Conditional Loading

### 1. `when()` Method:
```php
class UserResource extends JsonResource
{
    public function toArray($request)
    {
        return [
            'id' => $this->id,
            'name' => $this->name,
            'email' => $this->email,
            
            // Conditional fields
            'email' => $this->when($request->user()->isAdmin(), $this->email),
            'phone' => $this->when($this->phone, $this->phone),
            'bio' => $this->when($this->bio, $this->bio),
            
            // Conditional with default value
            'avatar' => $this->when($this->avatar, $this->avatar, 'default-avatar.png'),
            
            // Conditional with callback
            'permissions' => $this->when($request->user()->can('view-permissions'), function () {
                return $this->permissions->pluck('name');
            }),
        ];
    }
}
```

### 2. `whenLoaded()` for Relationships:
```php
class UserResource extends JsonResource
{
    public function toArray($request)
    {
        return [
            'id' => $this->id,
            'name' => $this->name,
            'email' => $this->email,
            
            // Only include if relationship is loaded
            'posts' => PostResource::collection($this->whenLoaded('posts')),
            'profile' => new ProfileResource($this->whenLoaded('profile')),
            'roles' => RoleResource::collection($this->whenLoaded('roles')),
            
            // Conditional relationship with callback
            'recent_posts' => $this->whenLoaded('posts', function () {
                return PostResource::collection(
                    $this->posts->sortByDesc('created_at')->take(3)
                );
            }),
        ];
    }
}
```

### 3. `whenCounted()` for Count Relationships:
```php
class UserResource extends JsonResource
{
    public function toArray($request)
    {
        return [
            'id' => $this->id,
            'name' => $this->name,
            
            // Only include if count is loaded
            'posts_count' => $this->whenCounted('posts'),
            'comments_count' => $this->whenCounted('comments'),
            'followers_count' => $this->whenCounted('followers'),
            
            // Conditional count with default
            'likes_count' => $this->whenCounted('likes', 0),
        ];
    }
}
```

### 4. `whenPivotLoaded()` for Pivot Data:
```php
class UserResource extends JsonResource
{
    public function toArray($request)
    {
        return [
            'id' => $this->id,
            'name' => $this->name,
            
            // Pivot data from many-to-many relationships
            'role_assigned_at' => $this->whenPivotLoaded('role_user', function () {
                return $this->pivot->created_at;
            }),
            
            'team_role' => $this->whenPivotLoaded('team_user', function () {
                return $this->pivot->role;
            }),
        ];
    }
}
```

---

## Relationship Loading

### 1. Nested Resources:
```php
class PostResource extends JsonResource
{
    public function toArray($request)
    {
        return [
            'id' => $this->id,
            'title' => $this->title,
            'content' => $this->content,
            'status' => $this->status,
            
            // Nested resources
            'author' => new UserResource($this->whenLoaded('user')),
            'category' => new CategoryResource($this->whenLoaded('category')),
            'tags' => TagResource::collection($this->whenLoaded('tags')),
            'comments' => CommentResource::collection($this->whenLoaded('comments')),
            
            // Counts
            'views_count' => $this->views_count ?? 0,
            'likes_count' => $this->whenCounted('likes'),
            'comments_count' => $this->whenCounted('comments'),
        ];
    }
}
```

### 2. Controller with Proper Loading:
```php
class PostController extends Controller
{
    public function index(Request $request)
    {
        $posts = Post::with(['user', 'category'])
                    ->withCount(['likes', 'comments'])
                    ->when($request->include_tags, function ($query) {
                        $query->with('tags');
                    })
                    ->paginate(10);
        
        return PostResource::collection($posts);
    }
    
    public function show(Post $post)
    {
        // Load all necessary relationships
        $post->load([
            'user.profile',
            'category',
            'tags',
            'comments.user',
            'comments' => function ($query) {
                $query->latest()->limit(10);
            }
        ]);
        
        // Load counts
        $post->loadCount(['likes', 'views', 'shares']);
        
        return new PostResource($post);
    }
}
```

### 3. Dynamic Relationship Loading:
```php
class UserController extends Controller
{
    public function show(User $user, Request $request)
    {
        // Dynamic loading based on request parameters
        $includes = [];
        
        if ($request->has('include')) {
            $requestedIncludes = explode(',', $request->include);
            
            $allowedIncludes = ['posts', 'profile', 'roles', 'permissions'];
            $includes = array_intersect($requestedIncludes, $allowedIncludes);
        }
        
        if (!empty($includes)) {
            $user->load($includes);
        }
        
        // Dynamic counts
        if ($request->has('count')) {
            $requestedCounts = explode(',', $request->count);
            $allowedCounts = ['posts', 'comments', 'followers', 'following'];
            $counts = array_intersect($requestedCounts, $allowedCounts);
            
            if (!empty($counts)) {
                $user->loadCount($counts);
            }
        }
        
        return new UserResource($user);
    }
}

// Usage: GET /users/1?include=posts,profile&count=posts,followers
```

---

## Performance Optimization

### 1. Avoiding N+1 Problems:
```php
// ❌ Bad: N+1 Problem
class UserController extends Controller
{
    public function index()
    {
        $users = User::all(); // 1 query
        return UserResource::collection($users);
    }
}

class UserResource extends JsonResource
{
    public function toArray($request)
    {
        return [
            'id' => $this->id,
            'name' => $this->name,
            'posts_count' => $this->posts->count(), // N queries
            'latest_post' => $this->posts->latest()->first(), // N queries
        ];
    }
}
```

```php
// ✅ Good: Proper Eager Loading
class UserController extends Controller
{
    public function index()
    {
        $users = User::withCount('posts')
                    ->with(['posts' => function ($query) {
                        $query->latest()->limit(1);
                    }])
                    ->get();
        
        return UserResource::collection($users);
    }
}

class UserResource extends JsonResource
{
    public function toArray($request)
    {
        return [
            'id' => $this->id,
            'name' => $this->name,
            'posts_count' => $this->posts_count, // No extra query
            'latest_post' => new PostResource($this->posts->first()), // No extra query
        ];
    }
}
```

### 2. Conditional Resource Loading:
```php
class UserResource extends JsonResource
{
    public function toArray($request)
    {
        return [
            'id' => $this->id,
            'name' => $this->name,
            'email' => $this->email,
            
            // Only load heavy relationships when requested
            'posts' => $this->when(
                $request->has('include_posts'),
                function () {
                    return PostResource::collection($this->whenLoaded('posts'));
                }
            ),
            
            // Lightweight data always included
            'posts_count' => $this->whenCounted('posts'),
            'is_active' => $this->is_active,
        ];
    }
}
```

### 3. Resource Caching:
```php
class UserResource extends JsonResource
{
    public function toArray($request)
    {
        // Cache expensive computations
        $cacheKey = "user_resource_{$this->id}_{$this->updated_at->timestamp}";
        
        return Cache::remember($cacheKey, 3600, function () {
            return [
                'id' => $this->id,
                'name' => $this->name,
                'email' => $this->email,
                'avatar_url' => $this->getAvatarUrl(), // Expensive operation
                'statistics' => $this->calculateStatistics(), // Heavy computation
            ];
        });
    }
    
    private function getAvatarUrl()
    {
        // Expensive image processing or external API call
        return $this->avatar ? asset($this->avatar) : $this->generateGravatar();
    }
    
    private function calculateStatistics()
    {
        // Heavy database calculations
        return [
            'total_posts' => $this->posts()->count(),
            'total_likes' => $this->posts()->withCount('likes')->get()->sum('likes_count'),
            'engagement_rate' => $this->calculateEngagementRate(),
        ];
    }
}
```

---

## Advanced Techniques

### 1. Resource Inheritance:
```php
// Base resource
abstract class BaseResource extends JsonResource
{
    protected function formatDate($date)
    {
        return $date ? $date->format('Y-m-d H:i:s') : null;
    }
    
    protected function formatCurrency($amount)
    {
        return number_format($amount, 2);
    }
    
    protected function getPermissions($request)
    {
        return [
            'can_edit' => $request->user()?->can('update', $this->resource),
            'can_delete' => $request->user()?->can('delete', $this->resource),
        ];
    }
}

// Specific resource extending base
class UserResource extends BaseResource
{
    public function toArray($request)
    {
        return [
            'id' => $this->id,
            'name' => $this->name,
            'created_at' => $this->formatDate($this->created_at),
            'permissions' => $this->getPermissions($request),
        ];
    }
}
```

### 2. Resource Wrapping:
```php
// Disable default wrapping
class AppServiceProvider extends ServiceProvider
{
    public function boot()
    {
        JsonResource::withoutWrapping();
    }
}

// Custom wrapping
class UserResource extends JsonResource
{
    public static $wrap = 'user';
    
    public function toArray($request)
    {
        return [
            'id' => $this->id,
            'name' => $this->name,
        ];
    }
}

// Response: { "user": { "id": 1, "name": "John" } }
```

### 3. Resource Middleware:
```php
class UserResource extends JsonResource
{
    public function toArray($request)
    {
        $data = [
            'id' => $this->id,
            'name' => $this->name,
            'email' => $this->email,
        ];
        
        // Apply middleware-like transformations
        $data = $this->applyUserPermissions($data, $request);
        $data = $this->applyLocalization($data, $request);
        $data = $this->applyVersioning($data, $request);
        
        return $data;
    }
    
    private function applyUserPermissions($data, $request)
    {
        if (!$request->user()?->isAdmin()) {
            unset($data['email']);
        }
        
        return $data;
    }
    
    private function applyLocalization($data, $request)
    {
        $locale = $request->header('Accept-Language', 'en');
        
        if ($locale === 'bn') {
            $data['name'] = $this->name_bn ?? $data['name'];
        }
        
        return $data;
    }
    
    private function applyVersioning($data, $request)
    {
        $version = $request->header('API-Version', 'v1');
        
        if ($version === 'v2') {
            $data['full_name'] = $data['name'];
            unset($data['name']);
        }
        
        return $data;
    }
}
```

### 4. Resource Events:
```php
class UserResource extends JsonResource
{
    public function toArray($request)
    {
        // Fire event before transformation
        event(new ResourceTransforming($this->resource, $request));
        
        $data = [
            'id' => $this->id,
            'name' => $this->name,
            'email' => $this->email,
        ];
        
        // Fire event after transformation
        event(new ResourceTransformed($data, $this->resource, $request));
        
        return $data;
    }
}

// Event listeners
class LogResourceAccess
{
    public function handle(ResourceTransformed $event)
    {
        Log::info('Resource accessed', [
            'resource_type' => get_class($event->resource),
            'resource_id' => $event->resource->id,
            'user_id' => $event->request->user()?->id,
            'ip' => $event->request->ip(),
        ]);
    }
}
```

---

## Best Practices

### 1. **Consistent Structure:**
```php
class UserResource extends JsonResource
{
    public function toArray($request)
    {
        return [
            // Always include ID first
            'id' => $this->id,
            
            // Basic attributes
            'name' => $this->name,
            'email' => $this->email,
            'status' => $this->status,
            
            // Computed attributes
            'full_name' => $this->first_name . ' ' . $this->last_name,
            'avatar_url' => $this->avatar ? asset($this->avatar) : null,
            
            // Formatted dates
            'created_at' => $this->created_at->toISOString(),
            'updated_at' => $this->updated_at->toISOString(),
            
            // Relationships (conditional)
            'posts' => PostResource::collection($this->whenLoaded('posts')),
            'profile' => new ProfileResource($this->whenLoaded('profile')),
            
            // Counts
            'posts_count' => $this->whenCounted('posts'),
            
            // Permissions (if applicable)
            'permissions' => $this->when($request->user(), [
                'can_edit' => $request->user()?->can('update', $this->resource),
                'can_delete' => $request->user()?->can('delete', $this->resource),
            ]),
        ];
    }
}
```

### 2. **Error Handling:**
```php
class UserResource extends JsonResource
{
    public function toArray($request)
    {
        try {
            return [
                'id' => $this->id,
                'name' => $this->name,
                'email' => $this->email,
                'avatar_url' => $this->getAvatarUrl(),
                'posts' => PostResource::collection($this->whenLoaded('posts')),
            ];
        } catch (\Exception $e) {
            // Log error but don't break the response
            Log::error('Error in UserResource transformation', [
                'user_id' => $this->id,
                'error' => $e->getMessage(),
            ]);
            
            // Return minimal safe data
            return [
                'id' => $this->id,
                'name' => $this->name ?? 'Unknown User',
                'error' => 'Data transformation error',
            ];
        }
    }
    
    private function getAvatarUrl()
    {
        try {
            return $this->avatar ? asset($this->avatar) : $this->generateGravatar();
        } catch (\Exception $e) {
            return asset('images/default-avatar.png');
        }
    }
}
```

### 3. **Testing Resources:**
```php
// tests/Feature/UserResourceTest.php
class UserResourceTest extends TestCase
{
    public function test_user_resource_structure()
    {
        $user = User::factory()->create();
        
        $resource = new UserResource($user);
        $response = $resource->toArray(request());
        
        $this->assertArrayHasKey('id', $response);
        $this->assertArrayHasKey('name', $response);
        $this->assertArrayHasKey('email', $response);
        $this->assertArrayNotHasKey('password', $response);
        
        $this->assertEquals($user->id, $response['id']);
        $this->assertEquals($user->name, $response['name']);
    }
    
    public function test_user_resource_with_relationships()
    {
        $user = User::factory()->has(Post::factory()->count(3))->create();
        $user->load('posts');
        
        $resource = new UserResource($user);
        $response = $resource->toArray(request());
        
        $this->assertArrayHasKey('posts', $response);
        $this->assertCount(3, $response['posts']);
    }
    
    public function test_user_resource_conditional_loading()
    {
        $user = User::factory()->create();
        
        // Without loading posts
        $resource = new UserResource($user);
        $response = $resource->toArray(request());
        $this->assertArrayNotHasKey('posts', $response);
        
        // With loading posts
        $user->load('posts');
        $resource = new UserResource($user);
        $response = $resource->toArray(request());
        $this->assertArrayHasKey('posts', $response);
    }
}
```

### 4. **Documentation:**
```php
/**
 * User API Resource
 * 
 * Transforms User model for API responses
 * 
 * @property int $id User ID
 * @property string $name User full name
 * @property string $email User email address
 * @property string $avatar_url User avatar URL
 * @property Carbon $created_at Account creation date
 * @property Carbon $updated_at Last update date
 * 
 * Conditional fields:
 * @property PostResource[] $posts User posts (when loaded)
 * @property ProfileResource $profile User profile (when loaded)
 * @property int $posts_count Number of posts (when counted)
 * 
 * Example usage:
 * ```php
 * $user = User::with('posts')->withCount('posts')->find(1);
 * return new UserResource($user);
 * ```
 */
class UserResource extends JsonResource
{
    // Implementation...
}
```

---

## সারসংক্ষেপ

### 🎯 মূল বিষয়সমূহ:

**API Resource এর কাজ:**
- ✅ Model data কে JSON এ transform করা
- ✅ Sensitive data hide করা
- ✅ Consistent API structure maintain করা
- ✅ Conditional data loading
- ✅ Performance optimization

**Data Flow:**
1. **Controller** → Database query + relationship loading
2. **API Resource** → Data transformation + formatting
3. **JSON Response** → Client consumption

**Key Methods:**
- `when()` - Conditional fields
- `whenLoaded()` - Relationship loading
- `whenCounted()` - Count relationships
- `collection()` - Multiple resources

**Performance Tips:**
- ✅ Proper eager loading in controller
- ✅ Use `whenLoaded()` for relationships
- ✅ Use `withCount()` instead of `count()`
- ✅ Cache expensive computations
- ✅ Avoid N+1 problems

**Best Practices:**
- 🎯 Consistent resource structure
- 🔒 Hide sensitive data
- 📊 Include permissions when needed
- 🧪 Write comprehensive tests
- 📝 Document resource structure

Laravel API Resource সঠিকভাবে ব্যবহার করে আপনি **clean, consistent এবং performant** API তৈরি করতে পারবেন যা **frontend applications** এর জন্য **optimal data structure** প্রদান করে।