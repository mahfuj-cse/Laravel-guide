# 1️⃣1️⃣ Laravel Authorization - বিস্তারিত বাংলা গাইড

## 📋 সূচিপত্র
- [Authorization কি?](#authorization-কি)
- [Gates কি?](#gates-কি)
- [Policies কি?](#policies-কি)
- [Blade এ @can Directive](#blade-এ-can-directive)
- [Controller এ authorize()](#controller-এ-authorize)
- [Role-based Authorization](#role-based-authorization)
- [সম্পূর্ণ উদাহরণ](#সম্পূর্ণ-উদাহরণ)

---

## Authorization কি?

**Authorization** হলো **অনুমতি যাচাই** করার প্রক্রিয়া। Authentication এর পর Authorization চেক করে যে User কোন কাজ করতে পারবে।

### Authentication vs Authorization:
- **Authentication** = "তুমি কে?" (Who are you?)
- **Authorization** = "তুমি কি করতে পারো?" (What can you do?)

### Laravel Authorization এর উপাদান:
- ✅ **Gates** - Simple closure-based authorization
- ✅ **Policies** - Class-based authorization for models
- ✅ **@can** - Blade directive for UI
- ✅ **authorize()** - Controller method for checking

---

## Gates কি?

**Gates** হলো **সহজ Authorization Logic** যা Closure দিয়ে তৈরি করা হয়।

### ১. Gates তৈরি করা:
```php
<?php
// app/Providers/AuthServiceProvider.php

namespace App\Providers;

use Illuminate\Foundation\Support\Providers\AuthServiceProvider as ServiceProvider;
use Illuminate\Support\Facades\Gate;
use App\Models\User;
use App\Models\Post;

class AuthServiceProvider extends ServiceProvider
{
    public function boot()
    {
        $this->registerPolicies();

        // Simple Gate
        Gate::define('create-post', function (User $user) {
            return $user->role === 'admin' || $user->role === 'editor';
        });

        // Gate with Model Parameter
        Gate::define('update-post', function (User $user, Post $post) {
            return $user->id === $post->user_id || $user->role === 'admin';
        });

        // Gate with Multiple Parameters
        Gate::define('delete-post', function (User $user, Post $post) {
            // Only admin or post owner can delete
            if ($user->role === 'admin') {
                return true;
            }
            
            return $user->id === $post->user_id && $post->created_at->diffInHours() < 24;
        });

        // Gate for Admin Only
        Gate::define('admin-only', function (User $user) {
            return $user->role === 'admin';
        });

        // Gate with Before Hook (Super Admin)
        Gate::before(function (User $user, $ability) {
            if ($user->role === 'super-admin') {
                return true; // Super admin can do everything
            }
        });
    }
}
```

### ২. Gates ব্যবহার করা:
```php
<?php
// Controller এ Gates ব্যবহার

use Illuminate\Support\Facades\Gate;

class PostController extends Controller
{
    public function create()
    {
        // Gate Check
        if (Gate::denies('create-post')) {
            abort(403, 'You cannot create posts');
        }

        return view('posts.create');
    }

    public function edit(Post $post)
    {
        // Gate with Model
        if (Gate::denies('update-post', $post)) {
            abort(403, 'You cannot edit this post');
        }

        return view('posts.edit', compact('post'));
    }

    public function destroy(Post $post)
    {
        // Multiple ways to check
        if (Gate::allows('delete-post', $post)) {
            $post->delete();
            return redirect()->route('posts.index')->with('success', 'Post deleted');
        }

        return back()->with('error', 'You cannot delete this post');
    }
}
```

### ৩. Gates এর বিভিন্ন Method:
```php
// Check if allowed
if (Gate::allows('create-post')) {
    // User can create post
}

// Check if denied
if (Gate::denies('update-post', $post)) {
    // User cannot update post
}

// Check and get response
$response = Gate::inspect('delete-post', $post);
if ($response->allowed()) {
    // Allowed
} else {
    echo $response->message(); // Error message
}

// Check for any of multiple abilities
if (Gate::any(['update-post', 'delete-post'], $post)) {
    // User can either update or delete
}

// Check for all abilities
if (Gate::check(['create-post', 'admin-only'])) {
    // User has both abilities
}
```

---

## Policies কি?

**Policies** হলো **Model-specific Authorization** যা Class দিয়ে organize করা হয়।

### ১. Policy তৈরি করা:
```bash
# Policy তৈরি
php artisan make:policy PostPolicy

# Model সহ Policy তৈরি
php artisan make:policy PostPolicy --model=Post
```

### ২. Post Policy:
```php
<?php
// app/Policies/PostPolicy.php

namespace App\Policies;

use App\Models\Post;
use App\Models\User;
use Illuminate\Auth\Access\HandlesAuthorization;

class PostPolicy
{
    use HandlesAuthorization;

    // View any posts (index page)
    public function viewAny(User $user)
    {
        return true; // Anyone can view posts list
    }

    // View specific post
    public function view(User $user, Post $post)
    {
        // Can view if published or own post
        return $post->status === 'published' || $user->id === $post->user_id;
    }

    // Create new post
    public function create(User $user)
    {
        return in_array($user->role, ['admin', 'editor', 'author']);
    }

    // Update post
    public function update(User $user, Post $post)
    {
        // Admin can update any post, others only their own
        if ($user->role === 'admin') {
            return true;
        }

        return $user->id === $post->user_id;
    }

    // Delete post
    public function delete(User $user, Post $post)
    {
        // Admin can delete any post
        if ($user->role === 'admin') {
            return true;
        }

        // Author can delete own post within 24 hours
        return $user->id === $post->user_id && 
               $post->created_at->diffInHours() < 24;
    }

    // Restore deleted post
    public function restore(User $user, Post $post)
    {
        return $user->role === 'admin';
    }

    // Permanently delete
    public function forceDelete(User $user, Post $post)
    {
        return $user->role === 'admin';
    }

    // Publish post
    public function publish(User $user, Post $post)
    {
        return in_array($user->role, ['admin', 'editor']);
    }

    // Before method - runs before all other methods
    public function before(User $user, $ability)
    {
        if ($user->role === 'super-admin') {
            return true; // Super admin can do everything
        }
    }
}
```

### ৩. Policy Register করা:
```php
<?php
// app/Providers/AuthServiceProvider.php

class AuthServiceProvider extends ServiceProvider
{
    protected $policies = [
        Post::class => PostPolicy::class,
        Comment::class => CommentPolicy::class,
        User::class => UserPolicy::class,
    ];

    public function boot()
    {
        $this->registerPolicies();
    }
}
```

### ৪. Policy ব্যবহার করা:
```php
<?php
// Controller এ Policy ব্যবহার

class PostController extends Controller
{
    public function index()
    {
        // Check viewAny policy
        $this->authorize('viewAny', Post::class);
        
        $posts = Post::all();
        return view('posts.index', compact('posts'));
    }

    public function show(Post $post)
    {
        // Check view policy
        $this->authorize('view', $post);
        
        return view('posts.show', compact('post'));
    }

    public function create()
    {
        // Check create policy
        $this->authorize('create', Post::class);
        
        return view('posts.create');
    }

    public function store(Request $request)
    {
        $this->authorize('create', Post::class);
        
        $post = Post::create($request->validated());
        return redirect()->route('posts.show', $post);
    }

    public function edit(Post $post)
    {
        $this->authorize('update', $post);
        
        return view('posts.edit', compact('post'));
    }

    public function update(Request $request, Post $post)
    {
        $this->authorize('update', $post);
        
        $post->update($request->validated());
        return redirect()->route('posts.show', $post);
    }

    public function destroy(Post $post)
    {
        $this->authorize('delete', $post);
        
        $post->delete();
        return redirect()->route('posts.index');
    }

    // Custom method
    public function publish(Post $post)
    {
        $this->authorize('publish', $post);
        
        $post->update(['status' => 'published']);
        return back()->with('success', 'Post published');
    }
}
```

---

## Blade এ @can Directive

Blade Template এ Authorization check করার জন্য `@can` directive ব্যবহার করা হয়।

### ১. Basic @can Usage:
```blade
<!-- resources/views/posts/index.blade.php -->
@extends('layouts.app')

@section('content')
<div class="container">
    <div class="d-flex justify-content-between align-items-center mb-4">
        <h1>Posts</h1>
        
        @can('create', App\Models\Post::class)
            <a href="{{ route('posts.create') }}" class="btn btn-primary">Create Post</a>
        @endcan
    </div>

    <div class="row">
        @foreach($posts as $post)
            <div class="col-md-6 mb-4">
                <div class="card">
                    <div class="card-body">
                        <h5 class="card-title">{{ $post->title }}</h5>
                        <p class="card-text">{{ Str::limit($post->content, 100) }}</p>
                        
                        <div class="btn-group">
                            @can('view', $post)
                                <a href="{{ route('posts.show', $post) }}" class="btn btn-sm btn-outline-primary">View</a>
                            @endcan
                            
                            @can('update', $post)
                                <a href="{{ route('posts.edit', $post) }}" class="btn btn-sm btn-outline-secondary">Edit</a>
                            @endcan
                            
                            @can('delete', $post)
                                <form action="{{ route('posts.destroy', $post) }}" method="POST" class="d-inline">
                                    @csrf
                                    @method('DELETE')
                                    <button type="submit" class="btn btn-sm btn-outline-danger" 
                                            onclick="return confirm('Are you sure?')">Delete</button>
                                </form>
                            @endcan
                        </div>
                    </div>
                </div>
            </div>
        @endforeach
    </div>
</div>
@endsection
```

### ২. @cannot এবং @else:
```blade
<!-- Post Show Page -->
@extends('layouts.app')

@section('content')
<div class="container">
    <article>
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
        
        @cannot('view', $post)
            <p>This post is private.</p>
        @endcannot
    </article>

    <!-- Action Buttons -->
    <div class="mt-4">
        @can('update', $post)
            <a href="{{ route('posts.edit', $post) }}" class="btn btn-primary">Edit Post</a>
        @endcan
        
        @can('publish', $post)
            @if($post->status === 'draft')
                <form action="{{ route('posts.publish', $post) }}" method="POST" class="d-inline">
                    @csrf
                    <button type="submit" class="btn btn-success">Publish</button>
                </form>
            @endif
        @endcan
        
        @can('delete', $post)
            <form action="{{ route('posts.destroy', $post) }}" method="POST" class="d-inline">
                @csrf
                @method('DELETE')
                <button type="submit" class="btn btn-danger">Delete</button>
            </form>
        @endcan
    </div>
</div>
@endsection
```

### ৩. Gates এর সাথে @can:
```blade
<!-- Admin Panel -->
@can('admin-only')
    <div class="admin-panel">
        <h3>Admin Controls</h3>
        <a href="{{ route('admin.users') }}" class="btn btn-info">Manage Users</a>
        <a href="{{ route('admin.settings') }}" class="btn btn-warning">Settings</a>
    </div>
@endcan

<!-- Navigation Menu -->
<nav class="navbar">
    <ul class="navbar-nav">
        <li class="nav-item">
            <a href="{{ route('posts.index') }}" class="nav-link">Posts</a>
        </li>
        
        @can('create-post')
            <li class="nav-item">
                <a href="{{ route('posts.create') }}" class="nav-link">Create Post</a>
            </li>
        @endcan
        
        @can('admin-only')
            <li class="nav-item dropdown">
                <a href="#" class="nav-link dropdown-toggle">Admin</a>
                <ul class="dropdown-menu">
                    <li><a href="{{ route('admin.dashboard') }}">Dashboard</a></li>
                    <li><a href="{{ route('admin.users') }}">Users</a></li>
                </ul>
            </li>
        @endcan
    </ul>
</nav>
```

---

## Controller এ authorize()

Controller এ `authorize()` method ব্যবহার করে Authorization check করা হয়।

### ১. Basic authorize() Usage:
```php
<?php

class PostController extends Controller
{
    public function show(Post $post)
    {
        // Simple authorization
        $this->authorize('view', $post);
        
        return view('posts.show', compact('post'));
    }

    public function edit(Post $post)
    {
        // Authorization with custom message
        $this->authorize('update', $post, 'You cannot edit this post');
        
        return view('posts.edit', compact('post'));
    }

    public function store(Request $request)
    {
        // Class-based authorization
        $this->authorize('create', Post::class);
        
        $post = Post::create($request->validated());
        return redirect()->route('posts.show', $post);
    }
}
```

### ২. Manual Authorization Check:
```php
<?php

use Illuminate\Support\Facades\Gate;

class PostController extends Controller
{
    public function update(Request $request, Post $post)
    {
        // Manual check with Gate
        if (Gate::denies('update', $post)) {
            return response()->json(['error' => 'Unauthorized'], 403);
        }

        // Manual check with Policy
        if (!auth()->user()->can('update', $post)) {
            abort(403, 'You cannot update this post');
        }

        $post->update($request->validated());
        return response()->json(['message' => 'Post updated']);
    }

    public function bulkDelete(Request $request)
    {
        $postIds = $request->input('post_ids');
        $posts = Post::whereIn('id', $postIds)->get();

        // Check authorization for each post
        foreach ($posts as $post) {
            if (Gate::denies('delete', $post)) {
                return back()->with('error', "You cannot delete post: {$post->title}");
            }
        }

        // Delete all posts
        foreach ($posts as $post) {
            $post->delete();
        }

        return back()->with('success', 'Posts deleted successfully');
    }
}
```

### ৩. Resource Controller Authorization:
```php
<?php

class PostController extends Controller
{
    public function __construct()
    {
        // Apply authorization to all methods
        $this->authorizeResource(Post::class, 'post');
    }

    // Laravel automatically maps these methods to policy methods:
    // index -> viewAny
    // show -> view  
    // create -> create
    // store -> create
    // edit -> update
    // update -> update
    // destroy -> delete

    public function index()
    {
        // No need to call authorize() - done automatically
        $posts = Post::all();
        return view('posts.index', compact('posts'));
    }

    public function show(Post $post)
    {
        // Authorization already handled
        return view('posts.show', compact('post'));
    }
}
```

---

## Role-based Authorization

### ১. User Model এ Role:
```php
<?php
// app/Models/User.php

class User extends Authenticatable
{
    protected $fillable = [
        'name', 'email', 'password', 'role'
    ];

    // Role checking methods
    public function isAdmin()
    {
        return $this->role === 'admin';
    }

    public function isEditor()
    {
        return $this->role === 'editor';
    }

    public function isAuthor()
    {
        return $this->role === 'author';
    }

    public function hasRole($role)
    {
        return $this->role === $role;
    }

    public function hasAnyRole(array $roles)
    {
        return in_array($this->role, $roles);
    }
}
```

### ২. Role-based Gates:
```php
<?php
// AuthServiceProvider

public function boot()
{
    // Role-based gates
    Gate::define('manage-users', function (User $user) {
        return $user->isAdmin();
    });

    Gate::define('manage-posts', function (User $user) {
        return $user->hasAnyRole(['admin', 'editor']);
    });

    Gate::define('create-content', function (User $user) {
        return $user->hasAnyRole(['admin', 'editor', 'author']);
    });

    // Permission-based gate
    Gate::define('publish-posts', function (User $user) {
        return $user->hasAnyRole(['admin', 'editor']);
    });
}
```

### ৩. Role Middleware:
```php
<?php
// app/Http/Middleware/CheckRole.php

class CheckRole
{
    public function handle($request, Closure $next, ...$roles)
    {
        if (!auth()->check()) {
            return redirect('login');
        }

        $user = auth()->user();
        
        if (!$user->hasAnyRole($roles)) {
            abort(403, 'Insufficient permissions');
        }

        return $next($request);
    }
}

// Register in Kernel.php
protected $routeMiddleware = [
    'role' => \App\Http\Middleware\CheckRole::class,
];

// Usage in routes
Route::middleware(['auth', 'role:admin,editor'])->group(function () {
    Route::get('/admin/posts', [AdminPostController::class, 'index']);
});
```

---

## সম্পূর্ণ উদাহরণ

### ১. Complete Blog Authorization System:
```php
<?php
// Complete PostPolicy

class PostPolicy
{
    public function viewAny(User $user)
    {
        return true;
    }

    public function view(User $user, Post $post)
    {
        if ($post->status === 'published') {
            return true;
        }

        return $user->id === $post->user_id || $user->isAdmin();
    }

    public function create(User $user)
    {
        return $user->hasAnyRole(['admin', 'editor', 'author']);
    }

    public function update(User $user, Post $post)
    {
        if ($user->isAdmin()) {
            return true;
        }

        if ($user->isEditor() && $post->user->hasRole('author')) {
            return true;
        }

        return $user->id === $post->user_id;
    }

    public function delete(User $user, Post $post)
    {
        if ($user->isAdmin()) {
            return true;
        }

        return $user->id === $post->user_id && 
               $post->created_at->diffInHours() < 24;
    }

    public function publish(User $user, Post $post)
    {
        return $user->hasAnyRole(['admin', 'editor']);
    }

    public function feature(User $user, Post $post)
    {
        return $user->isAdmin();
    }
}
```

### ২. Complete Controller with Authorization:
```php
<?php

class PostController extends Controller
{
    public function index()
    {
        $this->authorize('viewAny', Post::class);
        
        $posts = Post::when(!auth()->user()->isAdmin(), function ($query) {
            return $query->where('status', 'published');
        })->paginate(10);

        return view('posts.index', compact('posts'));
    }

    public function show(Post $post)
    {
        $this->authorize('view', $post);
        
        return view('posts.show', compact('post'));
    }

    public function create()
    {
        $this->authorize('create', Post::class);
        
        return view('posts.create');
    }

    public function store(Request $request)
    {
        $this->authorize('create', Post::class);
        
        $post = auth()->user()->posts()->create($request->validated());
        
        return redirect()->route('posts.show', $post);
    }

    public function edit(Post $post)
    {
        $this->authorize('update', $post);
        
        return view('posts.edit', compact('post'));
    }

    public function update(Request $request, Post $post)
    {
        $this->authorize('update', $post);
        
        $post->update($request->validated());
        
        return redirect()->route('posts.show', $post);
    }

    public function destroy(Post $post)
    {
        $this->authorize('delete', $post);
        
        $post->delete();
        
        return redirect()->route('posts.index');
    }

    public function publish(Post $post)
    {
        $this->authorize('publish', $post);
        
        $post->update(['status' => 'published', 'published_at' => now()]);
        
        return back()->with('success', 'Post published successfully');
    }
}
```

---

## 🎯 Best Practices:

### Authorization Tips:
- ✅ **Gates** ব্যবহার করুন simple logic এর জন্য
- ✅ **Policies** ব্যবহার করুন model-specific logic এর জন্য
- ✅ **@can** directive ব্যবহার করুন UI তে
- ✅ **authorize()** method ব্যবহার করুন controller এ
- ✅ **Role-based** এবং **Permission-based** দুটোই implement করুন

### Security Tips:
- ✅ সবসময় Authorization check করুন
- ✅ Frontend এবং Backend দুই জায়গায় check করুন
- ✅ Default deny policy follow করুন
- ✅ Super admin এর জন্য before hook ব্যবহার করুন

---

## 📚 আরও জানতে:
- [Laravel Authorization](https://laravel.com/docs/authorization)
- [Gates Documentation](https://laravel.com/docs/authorization#gates)
- [Policies Documentation](https://laravel.com/docs/authorization#creating-policies)