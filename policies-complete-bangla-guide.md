# Laravel Policies - সম্পূর্ণ বিস্তারিত বাংলা গাইড

## 📋 সূচিপত্র
- [Policies কি এবং কেন?](#policies-কি-এবং-কেন)
- [কখন Policies ব্যবহার করবেন?](#কখন-policies-ব্যবহার-করবেন)
- [Gates vs Policies - পার্থক্য](#gates-vs-policies---পার্থক্য)
- [Policies তৈরি ও Structure](#policies-তৈরি-ও-structure)
- [Production Best Practices](#production-best-practices)
- [Real-world Examples](#real-world-examples)
- [Advanced Techniques](#advanced-techniques)
- [Testing Policies](#testing-policies)

---

## Policies কি এবং কেন?

### 🎯 Policies কি?
**Policy** হলো **Model-specific Authorization Logic** এর জন্য **Organized Class**। এটি একটি নির্দিষ্ট Model এর সাথে সম্পর্কিত সব Authorization Rules একসাথে রাখে।

### 🔥 কেন Policies ব্যবহার করবেন?

#### ✅ **Organization & Structure:**
```php
// ❌ Gates এ সব একসাথে (Messy)
Gate::define('view-post', function ($user, $post) { /* logic */ });
Gate::define('update-post', function ($user, $post) { /* logic */ });
Gate::define('delete-post', function ($user, $post) { /* logic */ });
Gate::define('publish-post', function ($user, $post) { /* logic */ });

// ✅ Policy এ Organized
class PostPolicy
{
    public function view(User $user, Post $post) { /* logic */ }
    public function update(User $user, Post $post) { /* logic */ }
    public function delete(User $user, Post $post) { /* logic */ }
    public function publish(User $user, Post $post) { /* logic */ }
}
```

#### ✅ **Auto-Discovery:**
```php
// Laravel automatically discovers policies
$this->authorize('update', $post); // Automatically calls PostPolicy::update()
```

#### ✅ **Resource Controller Integration:**
```php
// Automatic authorization for all resource methods
class PostController extends Controller
{
    public function __construct()
    {
        $this->authorizeResource(Post::class, 'post');
    }
    // Laravel automatically maps: index->viewAny, show->view, etc.
}
```

#### ✅ **Testability:**
```php
// Easy to test individual policy methods
public function test_user_can_update_own_post()
{
    $this->assertTrue($user->can('update', $post));
}
```

---

## কখন Policies ব্যবহার করবেন?

### 🟢 **Policies ব্যবহার করুন যখন:**

#### ১. **Model-specific Authorization:**
```php
// ✅ Perfect for Policies
- User can edit their own Post
- Admin can delete any Comment
- Author can publish their Article
- Owner can manage their Company
```

#### ২. **Complex Business Logic:**
```php
class PostPolicy
{
    public function update(User $user, Post $post)
    {
        // Complex business rules
        if ($user->isAdmin()) return true;
        
        if ($user->id === $post->user_id) {
            // Author can edit within 24 hours
            return $post->created_at->diffInHours() < 24;
        }
        
        if ($user->isEditor()) {
            // Editor can edit published posts
            return $post->status === 'published';
        }
        
        return false;
    }
}
```

#### ৩. **Resource Controllers:**
```php
// Perfect match with Resource Controllers
Route::resource('posts', PostController::class);
// Automatically maps to PostPolicy methods
```

#### ৪. **Multiple Related Actions:**
```php
class DocumentPolicy
{
    public function view(User $user, Document $doc) { /* */ }
    public function download(User $user, Document $doc) { /* */ }
    public function share(User $user, Document $doc) { /* */ }
    public function approve(User $user, Document $doc) { /* */ }
    public function archive(User $user, Document $doc) { /* */ }
}
```

### 🔴 **Gates ব্যবহার করুন যখন:**

#### ১. **Simple Global Permissions:**
```php
// ✅ Better as Gates
Gate::define('access-admin-panel', function ($user) {
    return $user->isAdmin();
});

Gate::define('create-post', function ($user) {
    return $user->hasRole(['admin', 'editor', 'author']);
});
```

#### ২. **Non-Model Related:**
```php
// ✅ Gates for system-wide permissions
Gate::define('manage-settings', function ($user) {
    return $user->isSuperAdmin();
});
```

---

## Gates vs Policies - পার্থক্য

| বিষয় | Gates | Policies |
|-------|-------|----------|
| **উদ্দেশ্য** | Simple, global permissions | Model-specific authorization |
| **Structure** | Closure-based | Class-based |
| **Organization** | All in AuthServiceProvider | Separate class files |
| **Auto-discovery** | Manual definition | Automatic |
| **Resource Integration** | Manual | Automatic |
| **Complexity** | Simple logic | Complex business rules |
| **Testing** | Harder to test | Easy to test |
| **Reusability** | Limited | High |

### 📊 **কখন কোনটা ব্যবহার করবেন:**

```php
// 🟢 Gates - Perfect for:
Gate::define('admin-only', fn($user) => $user->isAdmin());
Gate::define('beta-feature', fn($user) => $user->beta_access);
Gate::define('maintenance-mode', fn($user) => $user->isSuperAdmin());

// 🟢 Policies - Perfect for:
PostPolicy::class     // Post CRUD operations
UserPolicy::class     // User management
OrderPolicy::class    // Order processing
CompanyPolicy::class  // Company management
```

---

## Policies তৈরি ও Structure

### ১. **Policy তৈরি করা:**
```bash
# Basic Policy
php artisan make:policy PostPolicy

# Policy with Model
php artisan make:policy PostPolicy --model=Post

# Policy with all resource methods
php artisan make:policy PostPolicy --model=Post --resource
```

### ২. **Standard Policy Structure:**
```php
<?php
// app/Policies/PostPolicy.php

namespace App\Policies;

use App\Models\Post;
use App\Models\User;
use Illuminate\Auth\Access\HandlesAuthorization;
use Illuminate\Auth\Access\Response;

class PostPolicy
{
    use HandlesAuthorization;

    /**
     * Determine if the user can view any posts.
     */
    public function viewAny(User $user): bool
    {
        return true; // Anyone can view posts list
    }

    /**
     * Determine if the user can view the post.
     */
    public function view(?User $user, Post $post): bool
    {
        // Public posts can be viewed by anyone
        if ($post->status === 'published') {
            return true;
        }

        // Private posts only by owner
        return $user && $user->id === $post->user_id;
    }

    /**
     * Determine if the user can create posts.
     */
    public function create(User $user): bool
    {
        return $user->hasAnyRole(['admin', 'editor', 'author']);
    }

    /**
     * Determine if the user can update the post.
     */
    public function update(User $user, Post $post): Response|bool
    {
        // Admin can update any post
        if ($user->isAdmin()) {
            return true;
        }

        // Author can update own post within time limit
        if ($user->id === $post->user_id) {
            if ($post->created_at->diffInHours() > 24) {
                return Response::deny('You can only edit posts within 24 hours.');
            }
            return true;
        }

        // Editor can update published posts
        if ($user->isEditor() && $post->status === 'published') {
            return true;
        }

        return Response::deny('You do not have permission to edit this post.');
    }

    /**
     * Determine if the user can delete the post.
     */
    public function delete(User $user, Post $post): bool
    {
        // Admin can delete any post
        if ($user->isAdmin()) {
            return true;
        }

        // Author can delete own unpublished post
        return $user->id === $post->user_id && $post->status !== 'published';
    }

    /**
     * Determine if the user can restore the post.
     */
    public function restore(User $user, Post $post): bool
    {
        return $user->isAdmin();
    }

    /**
     * Determine if the user can permanently delete the post.
     */
    public function forceDelete(User $user, Post $post): bool
    {
        return $user->isSuperAdmin();
    }

    /**
     * Custom method - Publish post
     */
    public function publish(User $user, Post $post): bool
    {
        return $user->hasAnyRole(['admin', 'editor']);
    }

    /**
     * Custom method - Feature post
     */
    public function feature(User $user, Post $post): bool
    {
        return $user->isAdmin() && $post->status === 'published';
    }

    /**
     * Before method - runs before all other methods
     */
    public function before(User $user, string $ability): ?bool
    {
        // Super admin can do everything
        if ($user->isSuperAdmin()) {
            return true;
        }

        // Return null to continue with normal authorization
        return null;
    }
}
```

### ৩. **Policy Registration:**
```php
<?php
// app/Providers/AuthServiceProvider.php

namespace App\Providers;

use App\Models\Post;
use App\Models\Comment;
use App\Models\User;
use App\Models\Company;
use App\Policies\PostPolicy;
use App\Policies\CommentPolicy;
use App\Policies\UserPolicy;
use App\Policies\CompanyPolicy;
use Illuminate\Foundation\Support\Providers\AuthServiceProvider as ServiceProvider;

class AuthServiceProvider extends ServiceProvider
{
    /**
     * The policy mappings for the application.
     */
    protected $policies = [
        Post::class => PostPolicy::class,
        Comment::class => CommentPolicy::class,
        User::class => UserPolicy::class,
        Company::class => CompanyPolicy::class,
    ];

    /**
     * Register any authentication / authorization services.
     */
    public function boot(): void
    {
        $this->registerPolicies();

        // Additional gates can be defined here
    }
}
```

---

## Production Best Practices

### 🏗️ **১. Folder Structure:**
```
app/
├── Policies/
│   ├── BasePolicy.php          # Common policy logic
│   ├── PostPolicy.php
│   ├── CommentPolicy.php
│   ├── UserPolicy.php
│   ├── Admin/
│   │   ├── AdminUserPolicy.php
│   │   └── SystemPolicy.php
│   └── Api/
│       ├── ApiPostPolicy.php
│       └── ApiUserPolicy.php
```

### 🎯 **২. Base Policy Pattern:**
```php
<?php
// app/Policies/BasePolicy.php

namespace App\Policies;

use App\Models\User;
use Illuminate\Auth\Access\HandlesAuthorization;

abstract class BasePolicy
{
    use HandlesAuthorization;

    /**
     * Common before method for all policies
     */
    public function before(User $user, string $ability): ?bool
    {
        // Super admin bypass
        if ($user->isSuperAdmin()) {
            return true;
        }

        // Banned users cannot do anything
        if ($user->isBanned()) {
            return false;
        }

        return null;
    }

    /**
     * Check if user has any of the given roles
     */
    protected function hasAnyRole(User $user, array $roles): bool
    {
        return $user->hasAnyRole($roles);
    }

    /**
     * Check if user owns the resource
     */
    protected function isOwner(User $user, $resource): bool
    {
        return $user->id === $resource->user_id;
    }

    /**
     * Check if resource is within time limit for editing
     */
    protected function withinTimeLimit($resource, int $hours = 24): bool
    {
        return $resource->created_at->diffInHours() <= $hours;
    }
}
```

### 🔧 **৩. Extended Policy Example:**
```php
<?php
// app/Policies/PostPolicy.php

namespace App\Policies;

use App\Models\Post;
use App\Models\User;
use Illuminate\Auth\Access\Response;

class PostPolicy extends BasePolicy
{
    public function viewAny(User $user): bool
    {
        return true;
    }

    public function view(?User $user, Post $post): bool
    {
        if ($post->isPublished()) {
            return true;
        }

        return $user && $this->isOwner($user, $post);
    }

    public function create(User $user): bool
    {
        return $this->hasAnyRole($user, ['admin', 'editor', 'author']);
    }

    public function update(User $user, Post $post): Response|bool
    {
        if ($user->isAdmin()) {
            return true;
        }

        if ($this->isOwner($user, $post)) {
            if (!$this->withinTimeLimit($post, 24)) {
                return Response::deny('Posts can only be edited within 24 hours.');
            }
            return true;
        }

        if ($user->isEditor() && $post->isPublished()) {
            return true;
        }

        return Response::deny('Unauthorized to update this post.');
    }

    public function delete(User $user, Post $post): bool
    {
        if ($user->isAdmin()) {
            return true;
        }

        return $this->isOwner($user, $post) && !$post->isPublished();
    }

    public function publish(User $user, Post $post): Response|bool
    {
        if (!$this->hasAnyRole($user, ['admin', 'editor'])) {
            return Response::deny('Only admins and editors can publish posts.');
        }

        if ($post->isPublished()) {
            return Response::deny('Post is already published.');
        }

        return true;
    }

    public function unpublish(User $user, Post $post): bool
    {
        return $user->isAdmin() && $post->isPublished();
    }

    public function feature(User $user, Post $post): Response|bool
    {
        if (!$user->isAdmin()) {
            return Response::deny('Only admins can feature posts.');
        }

        if (!$post->isPublished()) {
            return Response::deny('Only published posts can be featured.');
        }

        return true;
    }
}
```

### 🚀 **৪. Controller Integration:**
```php
<?php
// app/Http/Controllers/PostController.php

namespace App\Http\Controllers;

use App\Models\Post;
use App\Http\Requests\StorePostRequest;
use App\Http\Requests\UpdatePostRequest;
use Illuminate\Http\Request;

class PostController extends Controller
{
    public function __construct()
    {
        // Automatic authorization for all resource methods
        $this->authorizeResource(Post::class, 'post');
    }

    public function index()
    {
        // Authorization already handled by authorizeResource
        $posts = Post::published()->paginate(10);
        return view('posts.index', compact('posts'));
    }

    public function show(Post $post)
    {
        // Authorization already handled
        return view('posts.show', compact('post'));
    }

    public function create()
    {
        // Authorization already handled
        return view('posts.create');
    }

    public function store(StorePostRequest $request)
    {
        // Authorization already handled
        $post = auth()->user()->posts()->create($request->validated());
        return redirect()->route('posts.show', $post);
    }

    public function edit(Post $post)
    {
        // Authorization already handled
        return view('posts.edit', compact('post'));
    }

    public function update(UpdatePostRequest $request, Post $post)
    {
        // Authorization already handled
        $post->update($request->validated());
        return redirect()->route('posts.show', $post);
    }

    public function destroy(Post $post)
    {
        // Authorization already handled
        $post->delete();
        return redirect()->route('posts.index');
    }

    // Custom actions need manual authorization
    public function publish(Post $post)
    {
        $this->authorize('publish', $post);
        
        $post->update(['status' => 'published', 'published_at' => now()]);
        return back()->with('success', 'Post published successfully');
    }

    public function feature(Post $post)
    {
        $this->authorize('feature', $post);
        
        $post->update(['featured' => true]);
        return back()->with('success', 'Post featured successfully');
    }
}
```

### 🎨 **৫. Blade Integration:**
```blade
<!-- resources/views/posts/show.blade.php -->
@extends('layouts.app')

@section('content')
<div class="container">
    <article class="post">
        <h1>{{ $post->title }}</h1>
        
        @can('view', $post)
            <div class="post-content">
                {!! $post->content !!}
            </div>
        @else
            <div class="alert alert-warning">
                You don't have permission to view this post.
            </div>
        @endcan

        <div class="post-meta">
            <p>By {{ $post->author->name }} on {{ $post->created_at->format('M d, Y') }}</p>
            
            @if($post->status === 'draft')
                <span class="badge badge-warning">Draft</span>
            @endif
            
            @if($post->featured)
                <span class="badge badge-success">Featured</span>
            @endif
        </div>

        <!-- Action Buttons -->
        <div class="post-actions mt-4">
            @can('update', $post)
                <a href="{{ route('posts.edit', $post) }}" class="btn btn-primary">
                    <i class="fas fa-edit"></i> Edit
                </a>
            @endcan

            @can('delete', $post)
                <form action="{{ route('posts.destroy', $post) }}" method="POST" class="d-inline">
                    @csrf
                    @method('DELETE')
                    <button type="submit" class="btn btn-danger" 
                            onclick="return confirm('Are you sure?')">
                        <i class="fas fa-trash"></i> Delete
                    </button>
                </form>
            @endcan

            @can('publish', $post)
                @if($post->status === 'draft')
                    <form action="{{ route('posts.publish', $post) }}" method="POST" class="d-inline">
                        @csrf
                        <button type="submit" class="btn btn-success">
                            <i class="fas fa-globe"></i> Publish
                        </button>
                    </form>
                @endif
            @endcan

            @can('feature', $post)
                @if(!$post->featured)
                    <form action="{{ route('posts.feature', $post) }}" method="POST" class="d-inline">
                        @csrf
                        <button type="submit" class="btn btn-warning">
                            <i class="fas fa-star"></i> Feature
                        </button>
                    </form>
                @endif
            @endcan
        </div>
    </article>
</div>
@endsection
```

---

## Real-world Examples

### 🏢 **১. Company Management System:**
```php
<?php
// app/Policies/CompanyPolicy.php

class CompanyPolicy extends BasePolicy
{
    public function viewAny(User $user): bool
    {
        return $user->hasAnyRole(['admin', 'manager']);
    }

    public function view(User $user, Company $company): bool
    {
        // Admin can view any company
        if ($user->isAdmin()) return true;
        
        // User can view their own company
        return $user->company_id === $company->id;
    }

    public function create(User $user): bool
    {
        return $user->isAdmin();
    }

    public function update(User $user, Company $company): Response|bool
    {
        if ($user->isAdmin()) return true;
        
        // Company owner can update
        if ($user->id === $company->owner_id) return true;
        
        // Manager can update basic info only
        if ($user->isManager() && $user->company_id === $company->id) {
            return Response::allow('Limited update access');
        }
        
        return Response::deny('Unauthorized to update company');
    }

    public function delete(User $user, Company $company): Response|bool
    {
        if (!$user->isAdmin()) {
            return Response::deny('Only admins can delete companies');
        }
        
        if ($company->hasActiveSubscription()) {
            return Response::deny('Cannot delete company with active subscription');
        }
        
        return true;
    }

    public function manageUsers(User $user, Company $company): bool
    {
        return $user->isAdmin() || 
               ($user->id === $company->owner_id) ||
               ($user->isManager() && $user->company_id === $company->id);
    }

    public function viewFinancials(User $user, Company $company): bool
    {
        return $user->isAdmin() || 
               ($user->id === $company->owner_id) ||
               ($user->hasRole('accountant') && $user->company_id === $company->id);
    }
}
```

### 📝 **২. Document Management System:**
```php
<?php
// app/Policies/DocumentPolicy.php

class DocumentPolicy extends BasePolicy
{
    public function view(User $user, Document $document): Response|bool
    {
        // Public documents
        if ($document->visibility === 'public') return true;
        
        // Owner can always view
        if ($this->isOwner($user, $document)) return true;
        
        // Shared documents
        if ($document->isSharedWith($user)) return true;
        
        // Department access
        if ($document->visibility === 'department' && 
            $user->department_id === $document->department_id) {
            return true;
        }
        
        return Response::deny('Document not accessible');
    }

    public function download(User $user, Document $document): Response|bool
    {
        if (!$this->view($user, $document)) {
            return Response::deny('Cannot download inaccessible document');
        }
        
        // Check download permissions
        if ($document->download_restricted && !$this->isOwner($user, $document)) {
            return Response::deny('Download restricted by owner');
        }
        
        return true;
    }

    public function share(User $user, Document $document): bool
    {
        return $this->isOwner($user, $document) || $user->isAdmin();
    }

    public function approve(User $user, Document $document): Response|bool
    {
        if ($document->status !== 'pending') {
            return Response::deny('Document is not pending approval');
        }
        
        return $user->hasAnyRole(['admin', 'manager', 'supervisor']);
    }
}
```

### 🛒 **৩. E-commerce Order System:**
```php
<?php
// app/Policies/OrderPolicy.php

class OrderPolicy extends BasePolicy
{
    public function view(User $user, Order $order): bool
    {
        // Customer can view their own orders
        if ($user->id === $order->customer_id) return true;
        
        // Staff can view orders
        return $user->hasAnyRole(['admin', 'staff', 'manager']);
    }

    public function update(User $user, Order $order): Response|bool
    {
        // Cannot update completed orders
        if ($order->status === 'completed') {
            return Response::deny('Completed orders cannot be modified');
        }
        
        // Customer can update pending orders
        if ($user->id === $order->customer_id && $order->status === 'pending') {
            return true;
        }
        
        // Staff can update processing orders
        if ($user->hasRole('staff') && $order->status === 'processing') {
            return true;
        }
        
        return $user->isAdmin();
    }

    public function cancel(User $user, Order $order): Response|bool
    {
        if ($order->status === 'shipped') {
            return Response::deny('Cannot cancel shipped orders');
        }
        
        // Customer can cancel within 1 hour
        if ($user->id === $order->customer_id) {
            if ($order->created_at->diffInHours() > 1) {
                return Response::deny('Orders can only be cancelled within 1 hour');
            }
            return true;
        }
        
        return $user->hasAnyRole(['admin', 'manager']);
    }

    public function refund(User $user, Order $order): Response|bool
    {
        if ($order->status !== 'completed') {
            return Response::deny('Only completed orders can be refunded');
        }
        
        if (!$user->hasAnyRole(['admin', 'manager'])) {
            return Response::deny('Only managers can process refunds');
        }
        
        // Check refund time limit (30 days)
        if ($order->completed_at->diffInDays() > 30) {
            return Response::deny('Refund period has expired');
        }
        
        return true;
    }
}
```

---

## Advanced Techniques

### 🔧 **১. Policy with Response Messages:**
```php
<?php

use Illuminate\Auth\Access\Response;

class PostPolicy extends BasePolicy
{
    public function update(User $user, Post $post): Response
    {
        if ($user->isAdmin()) {
            return Response::allow();
        }

        if ($user->id !== $post->user_id) {
            return Response::deny('You can only edit your own posts.');
        }

        if ($post->status === 'published') {
            return Response::deny('Published posts cannot be edited.');
        }

        if ($post->created_at->diffInHours() > 24) {
            return Response::deny('Posts can only be edited within 24 hours of creation.');
        }

        return Response::allow();
    }
}

// Controller এ response handle করা
public function edit(Post $post)
{
    $response = Gate::inspect('update', $post);
    
    if ($response->denied()) {
        return back()->with('error', $response->message());
    }
    
    return view('posts.edit', compact('post'));
}
```

### 🎛️ **২. Policy with Context:**
```php
<?php

class PostPolicy extends BasePolicy
{
    public function update(User $user, Post $post, array $context = []): Response|bool
    {
        // Admin bypass
        if ($user->isAdmin()) return true;
        
        // Check if it's an API request
        if ($context['via'] === 'api') {
            // Stricter rules for API
            return $this->isOwner($user, $post) && 
                   $this->withinTimeLimit($post, 1); // 1 hour for API
        }
        
        // Web interface - more lenient
        return $this->isOwner($user, $post) && 
               $this->withinTimeLimit($post, 24); // 24 hours for web
    }
}

// Usage with context
$this->authorize('update', [$post, ['via' => 'api']]);
```

### 🔄 **৩. Policy Inheritance:**
```php
<?php
// Base Content Policy
abstract class ContentPolicy extends BasePolicy
{
    protected function canModifyContent(User $user, $content): bool
    {
        if ($user->isAdmin()) return true;
        if ($this->isOwner($user, $content)) return true;
        if ($user->isEditor() && $content->status === 'published') return true;
        
        return false;
    }
    
    protected function canDeleteContent(User $user, $content): bool
    {
        if ($user->isAdmin()) return true;
        
        return $this->isOwner($user, $content) && 
               $content->status !== 'published';
    }
}

// Post Policy extends Content Policy
class PostPolicy extends ContentPolicy
{
    public function update(User $user, Post $post): bool
    {
        return $this->canModifyContent($user, $post);
    }
    
    public function delete(User $user, Post $post): bool
    {
        return $this->canDeleteContent($user, $post);
    }
}

// Article Policy extends Content Policy
class ArticlePolicy extends ContentPolicy
{
    public function update(User $user, Article $article): bool
    {
        return $this->canModifyContent($user, $article);
    }
    
    public function delete(User $user, Article $article): bool
    {
        return $this->canDeleteContent($user, $article);
    }
}
```

### 🎯 **৪. Dynamic Policy Loading:**
```php
<?php
// app/Providers/AuthServiceProvider.php

class AuthServiceProvider extends ServiceProvider
{
    public function boot(): void
    {
        $this->registerPolicies();
        
        // Dynamic policy registration
        $this->registerDynamicPolicies();
    }
    
    protected function registerDynamicPolicies(): void
    {
        // Auto-register policies based on models
        $modelPath = app_path('Models');
        $policyPath = app_path('Policies');
        
        if (is_dir($modelPath) && is_dir($policyPath)) {
            $models = glob($modelPath . '/*.php');
            
            foreach ($models as $modelFile) {
                $modelName = basename($modelFile, '.php');
                $modelClass = "App\\Models\\{$modelName}";
                $policyClass = "App\\Policies\\{$modelName}Policy";
                
                if (class_exists($modelClass) && class_exists($policyClass)) {
                    Gate::policy($modelClass, $policyClass);
                }
            }
        }
    }
}
```

---

## Testing Policies

### 🧪 **১. Basic Policy Testing:**
```php
<?php
// tests/Unit/Policies/PostPolicyTest.php

namespace Tests\Unit\Policies;

use App\Models\Post;
use App\Models\User;
use App\Policies\PostPolicy;
use Tests\TestCase;
use Illuminate\Foundation\Testing\RefreshDatabase;

class PostPolicyTest extends TestCase
{
    use RefreshDatabase;

    protected PostPolicy $policy;

    protected function setUp(): void
    {
        parent::setUp();
        $this->policy = new PostPolicy();
    }

    public function test_admin_can_view_any_post()
    {
        $admin = User::factory()->admin()->create();
        $post = Post::factory()->create();

        $this->assertTrue($this->policy->view($admin, $post));
    }

    public function test_user_can_view_own_post()
    {
        $user = User::factory()->create();
        $post = Post::factory()->create(['user_id' => $user->id]);

        $this->assertTrue($this->policy->view($user, $post));
    }

    public function test_user_cannot_view_others_draft_post()
    {
        $user = User::factory()->create();
        $otherUser = User::factory()->create();
        $post = Post::factory()->draft()->create(['user_id' => $otherUser->id]);

        $this->assertFalse($this->policy->view($user, $post));
    }

    public function test_user_can_update_own_post_within_time_limit()
    {
        $user = User::factory()->create();
        $post = Post::factory()->create([
            'user_id' => $user->id,
            'created_at' => now()->subHours(12) // 12 hours ago
        ]);

        $this->assertTrue($this->policy->update($user, $post));
    }

    public function test_user_cannot_update_own_post_after_time_limit()
    {
        $user = User::factory()->create();
        $post = Post::factory()->create([
            'user_id' => $user->id,
            'created_at' => now()->subHours(25) // 25 hours ago
        ]);

        $response = $this->policy->update($user, $post);
        $this->assertFalse($response);
    }

    public function test_editor_can_update_published_post()
    {
        $editor = User::factory()->editor()->create();
        $post = Post::factory()->published()->create();

        $this->assertTrue($this->policy->update($editor, $post));
    }
}
```

### 🎯 **২. Feature Testing with Policies:**
```php
<?php
// tests/Feature/PostAuthorizationTest.php

namespace Tests\Feature;

use App\Models\Post;
use App\Models\User;
use Tests\TestCase;
use Illuminate\Foundation\Testing\RefreshDatabase;

class PostAuthorizationTest extends TestCase
{
    use RefreshDatabase;

    public function test_user_can_edit_own_post()
    {
        $user = User::factory()->create();
        $post = Post::factory()->create(['user_id' => $user->id]);

        $response = $this->actingAs($user)
            ->get(route('posts.edit', $post));

        $response->assertStatus(200);
    }

    public function test_user_cannot_edit_others_post()
    {
        $user = User::factory()->create();
        $otherUser = User::factory()->create();
        $post = Post::factory()->create(['user_id' => $otherUser->id]);

        $response = $this->actingAs($user)
            ->get(route('posts.edit', $post));

        $response->assertStatus(403);
    }

    public function test_admin_can_delete_any_post()
    {
        $admin = User::factory()->admin()->create();
        $post = Post::factory()->create();

        $response = $this->actingAs($admin)
            ->delete(route('posts.destroy', $post));

        $response->assertRedirect(route('posts.index'));
        $this->assertSoftDeleted($post);
    }

    public function test_user_cannot_publish_post()
    {
        $user = User::factory()->create();
        $post = Post::factory()->create(['user_id' => $user->id]);

        $response = $this->actingAs($user)
            ->post(route('posts.publish', $post));

        $response->assertStatus(403);
    }

    public function test_editor_can_publish_post()
    {
        $editor = User::factory()->editor()->create();
        $post = Post::factory()->draft()->create();

        $response = $this->actingAs($editor)
            ->post(route('posts.publish', $post));

        $response->assertRedirect();
        $this->assertEquals('published', $post->fresh()->status);
    }
}
```

---

## 🎯 Production Checklist:

### ✅ **Security:**
- [ ] All sensitive operations have policy checks
- [ ] Before hooks handle super admin access
- [ ] Response messages don't leak sensitive info
- [ ] Time-based restrictions are properly implemented

### ✅ **Performance:**
- [ ] Policies don't make unnecessary database queries
- [ ] Eager loading is used where needed
- [ ] Caching is implemented for expensive checks

### ✅ **Maintainability:**
- [ ] Base policy class for common logic
- [ ] Clear method names and documentation
- [ ] Consistent return types (bool vs Response)
- [ ] Proper error messages

### ✅ **Testing:**
- [ ] Unit tests for all policy methods
- [ ] Feature tests for authorization flows
- [ ] Edge cases are covered
- [ ] Performance tests for complex policies

---

## 📚 আরও জানতে:
- [Laravel Authorization](https://laravel.com/docs/authorization)
- [Policy Classes](https://laravel.com/docs/authorization#creating-policies)
- [Authorization Responses](https://laravel.com/docs/authorization#authorization-responses)