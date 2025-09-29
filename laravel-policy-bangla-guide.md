# Laravel Policy - বিস্তারিত বাংলা গাইড

## 📋 সূচিপত্র
- [Policy কি?](#policy-কি)
- [কেন Policy ব্যবহার করবেন?](#কেন-policy-ব্যবহার-করবেন)
- [Policy তৈরি করা](#policy-তৈরি-করা)
- [Policy Methods](#policy-methods)
- [Policy Registration](#policy-registration)
- [Policy ব্যবহার করা](#policy-ব্যবহার-করা)
- [Advanced Features](#advanced-features)
- [Best Practices](#best-practices)

---

## Policy কি?

**Laravel Policy** হলো **Authorization Logic** organize করার একটি **clean এবং structured** উপায়। এটি **specific model** বা **resource** এর জন্য **permission checking** এর সব logic একসাথে রাখে।

### 🎯 সহজ ভাষায়:
Policy হলো একটি **Class** যেখানে আপনি define করেন:
- কে কোন **action** perform করতে পারবে
- কোন **condition** এ permission দেওয়া হবে
- **User** এর **role/permission** check করা

---

## কেন Policy ব্যবহার করবেন?

### ✅ সুবিধাসমূহ:

#### ১. **Clean Code Organization**
```php
// Policy ছাড়া (Controller এ scattered logic)
public function edit(Post $post)
{
    if (auth()->user()->id !== $post->user_id && !auth()->user()->isAdmin()) {
        abort(403);
    }
    // edit logic
}

// Policy দিয়ে (Clean & Organized)
public function edit(Post $post)
{
    $this->authorize('update', $post);
    // edit logic
}
```

#### ২. **Reusable Logic**
```php
// একবার Policy তে define করলে সব জায়গায় use করা যায়
$this->authorize('update', $post);  // Controller এ
@can('update', $post)               // Blade এ
$user->can('update', $post)         // Anywhere
```

#### ৩. **Centralized Permission Management**
```php
// সব permission logic এক জায়গায়
class PostPolicy
{
    public function view($user, $post) { /* logic */ }
    public function create($user) { /* logic */ }
    public function update($user, $post) { /* logic */ }
    public function delete($user, $post) { /* logic */ }
}
```

#### ৪. **Easy Testing**
```php
// Policy methods easily testable
public function test_user_can_update_own_post()
{
    $user = User::factory()->create();
    $post = Post::factory()->create(['user_id' => $user->id]);
    
    $this->assertTrue($user->can('update', $post));
}
```

---

## Policy তৈরি করা

### ১. Artisan Command দিয়ে:
```bash
# Basic Policy
php artisan make:policy PostPolicy

# Model এর সাথে Policy
php artisan make:policy PostPolicy --model=Post
```

### ২. Manual Policy Creation:
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

    /**
     * Determine whether the user can view any models.
     */
    public function viewAny(User $user)
    {
        return true; // সবাই posts দেখতে পারবে
    }

    /**
     * Determine whether the user can view the model.
     */
    public function view(User $user, Post $post)
    {
        return $post->status === 'published' || $user->id === $post->user_id;
    }

    /**
     * Determine whether the user can create models.
     */
    public function create(User $user)
    {
        return $user->email_verified_at !== null;
    }

    /**
     * Determine whether the user can update the model.
     */
    public function update(User $user, Post $post)
    {
        return $user->id === $post->user_id;
    }

    /**
     * Determine whether the user can delete the model.
     */
    public function delete(User $user, Post $post)
    {
        return $user->id === $post->user_id || $user->isAdmin();
    }
}
```

---

## Policy Methods

### ১. Standard CRUD Methods:
```php
class PostPolicy
{
    // GET /posts (index)
    public function viewAny(User $user)
    {
        return true;
    }

    // GET /posts/{post} (show)
    public function view(User $user, Post $post)
    {
        return $post->is_published || $user->id === $post->user_id;
    }

    // GET /posts/create (create form)
    public function create(User $user)
    {
        return $user->hasVerifiedEmail();
    }

    // POST /posts (store)
    public function store(User $user)
    {
        return $user->posts()->count() < 10; // Max 10 posts per user
    }

    // GET /posts/{post}/edit (edit form)
    // PUT/PATCH /posts/{post} (update)
    public function update(User $user, Post $post)
    {
        return $user->id === $post->user_id && $post->status !== 'published';
    }

    // DELETE /posts/{post} (destroy)
    public function delete(User $user, Post $post)
    {
        return $user->id === $post->user_id || $user->hasRole('admin');
    }
}
```

### ২. Custom Methods:
```php
class PostPolicy
{
    // Custom method for publishing
    public function publish(User $user, Post $post)
    {
        return $user->id === $post->user_id && $post->status === 'draft';
    }

    // Custom method for featuring
    public function feature(User $user, Post $post)
    {
        return $user->hasRole('editor') || $user->hasRole('admin');
    }

    // Custom method for commenting
    public function comment(User $user, Post $post)
    {
        return $post->allow_comments && $user->isNot($post->user);
    }
}
```

### ৩. Guest User Handling:
```php
class PostPolicy
{
    // Guest users (nullable User)
    public function view(?User $user, Post $post)
    {
        // Published posts can be viewed by anyone
        if ($post->status === 'published') {
            return true;
        }

        // Draft posts only by owner
        return $user && $user->id === $post->user_id;
    }

    // Only authenticated users can create
    public function create(?User $user)
    {
        return $user !== null;
    }
}
```

---

## Policy Registration

### ১. Auto-Discovery (Laravel 5.8+):
```php
// Laravel automatically discovers policies if they follow naming convention:
// Model: App\Models\Post
// Policy: App\Policies\PostPolicy
```

### ২. Manual Registration:
```php
// app/Providers/AuthServiceProvider.php

use App\Models\Post;
use App\Policies\PostPolicy;

class AuthServiceProvider extends ServiceProvider
{
    protected $policies = [
        Post::class => PostPolicy::class,
        // অন্যান্য model => policy mapping
    ];

    public function boot()
    {
        $this->registerPolicies();
    }
}
```

### ৩. Resource Policy Registration:
```php
// Multiple models with same policy pattern
protected $policies = [
    'App\Models\Post' => 'App\Policies\PostPolicy',
    'App\Models\Comment' => 'App\Policies\CommentPolicy',
    'App\Models\Category' => 'App\Policies\CategoryPolicy',
];
```

---

## Policy ব্যবহার করা

### ১. Controller এ Authorization:
```php
class PostController extends Controller
{
    public function index()
    {
        $this->authorize('viewAny', Post::class);
        return Post::paginate();
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

    // Custom action
    public function publish(Post $post)
    {
        $this->authorize('publish', $post);
        
        $post->update(['status' => 'published']);
        return back()->with('success', 'Post published!');
    }
}
```

### ২. Blade Template এ:
```blade
{{-- posts/index.blade.php --}}
@extends('layouts.app')

@section('content')
<div class="container">
    <div class="row">
        <div class="col-md-12">
            <h1>Posts</h1>
            
            @can('create', App\Models\Post::class)
                <a href="{{ route('posts.create') }}" class="btn btn-primary mb-3">
                    Create New Post
                </a>
            @endcan

            @foreach($posts as $post)
                <div class="card mb-3">
                    <div class="card-body">
                        <h5 class="card-title">{{ $post->title }}</h5>
                        <p class="card-text">{{ Str::limit($post->content, 150) }}</p>
                        
                        <div class="btn-group">
                            @can('view', $post)
                                <a href="{{ route('posts.show', $post) }}" class="btn btn-sm btn-outline-primary">
                                    View
                                </a>
                            @endcan

                            @can('update', $post)
                                <a href="{{ route('posts.edit', $post) }}" class="btn btn-sm btn-outline-secondary">
                                    Edit
                                </a>
                            @endcan

                            @can('publish', $post)
                                <form action="{{ route('posts.publish', $post) }}" method="POST" class="d-inline">
                                    @csrf
                                    @method('PATCH')
                                    <button type="submit" class="btn btn-sm btn-outline-success">
                                        Publish
                                    </button>
                                </form>
                            @endcan

                            @can('delete', $post)
                                <form action="{{ route('posts.destroy', $post) }}" method="POST" class="d-inline">
                                    @csrf
                                    @method('DELETE')
                                    <button type="submit" class="btn btn-sm btn-outline-danger" 
                                            onclick="return confirm('Are you sure?')">
                                        Delete
                                    </button>
                                </form>
                            @endcan
                        </div>
                    </div>
                </div>
            @endforeach
        </div>
    </div>
</div>
@endsection
```

### ৩. Programmatically Check:
```php
// Service class বা অন্য কোথাও
class PostService
{
    public function getUserPosts(User $user)
    {
        $posts = Post::query();

        // User can view all posts if admin
        if ($user->cannot('viewAny', Post::class)) {
            $posts->where('user_id', $user->id);
        }

        return $posts->get();
    }

    public function canUserEditPost(User $user, Post $post)
    {
        return $user->can('update', $post);
    }

    public function getAvailableActions(User $user, Post $post)
    {
        $actions = [];

        if ($user->can('view', $post)) {
            $actions[] = 'view';
        }

        if ($user->can('update', $post)) {
            $actions[] = 'edit';
        }

        if ($user->can('delete', $post)) {
            $actions[] = 'delete';
        }

        if ($user->can('publish', $post)) {
            $actions[] = 'publish';
        }

        return $actions;
    }
}
```

### ৪. API Resource এ:
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
            'created_at' => $this->created_at,
            'updated_at' => $this->updated_at,
            
            // Include permissions
            'permissions' => [
                'can_view' => $request->user()?->can('view', $this->resource) ?? false,
                'can_update' => $request->user()?->can('update', $this->resource) ?? false,
                'can_delete' => $request->user()?->can('delete', $this->resource) ?? false,
                'can_publish' => $request->user()?->can('publish', $this->resource) ?? false,
            ],
        ];
    }
}
```

---

## Advanced Features

### ১. Before Method (Super Admin):
```php
class PostPolicy
{
    /**
     * Perform pre-authorization checks.
     */
    public function before(User $user, $ability)
    {
        // Super admin can do everything
        if ($user->hasRole('super-admin')) {
            return true;
        }

        // Continue with normal policy checks
        return null;
    }

    public function update(User $user, Post $post)
    {
        return $user->id === $post->user_id;
    }
}
```

### ২. Policy with Relationships:
```php
class CommentPolicy
{
    public function create(User $user, Post $post)
    {
        // Can comment if post allows comments and user is not the author
        return $post->allow_comments && $user->isNot($post->user);
    }

    public function update(User $user, Comment $comment)
    {
        // Can edit own comment within 15 minutes
        return $user->id === $comment->user_id 
            && $comment->created_at->diffInMinutes() < 15;
    }

    public function delete(User $user, Comment $comment)
    {
        // Can delete own comment or if you're the post author
        return $user->id === $comment->user_id 
            || $user->id === $comment->post->user_id;
    }
}
```

### ৩. Policy with Complex Logic:
```php
class PostPolicy
{
    public function update(User $user, Post $post)
    {
        // Multiple conditions
        if ($user->id !== $post->user_id) {
            return false;
        }

        // Can't edit published posts unless admin
        if ($post->status === 'published' && !$user->hasRole('admin')) {
            return false;
        }

        // Can't edit if locked
        if ($post->is_locked) {
            return false;
        }

        return true;
    }

    public function publish(User $user, Post $post)
    {
        // Must be owner
        if ($user->id !== $post->user_id) {
            return false;
        }

        // Must be draft
        if ($post->status !== 'draft') {
            return false;
        }

        // Must have minimum content
        if (strlen($post->content) < 100) {
            return false;
        }

        // Must have featured image
        if (!$post->featured_image) {
            return false;
        }

        return true;
    }
}
```

---

## Best Practices

### ১. **Naming Convention**:
```php
// Model: Post -> Policy: PostPolicy
// Model: BlogPost -> Policy: BlogPostPolicy
// Model: UserProfile -> Policy: UserProfilePolicy
```

### ২. **Return Boolean Values**:
```php
// ✅ Good
public function update(User $user, Post $post)
{
    return $user->id === $post->user_id;
}

// ❌ Bad
public function update(User $user, Post $post)
{
    if ($user->id === $post->user_id) {
        return true;
    } else {
        return false;
    }
}
```

### ৩. **Use Descriptive Method Names**:
```php
class PostPolicy
{
    public function view(User $user, Post $post) { }
    public function create(User $user) { }
    public function update(User $user, Post $post) { }
    public function delete(User $user, Post $post) { }
    
    // Custom methods
    public function publish(User $user, Post $post) { }
    public function feature(User $user, Post $post) { }
    public function archive(User $user, Post $post) { }
}
```

### ৪. **Handle Guest Users**:
```php
public function view(?User $user, Post $post)
{
    // Public posts can be viewed by anyone
    if ($post->is_public) {
        return true;
    }

    // Private posts only by authenticated users
    return $user !== null && $user->id === $post->user_id;
}
```

### ৫. **Use Policy in Routes**:
```php
// routes/web.php
Route::middleware(['auth'])->group(function () {
    Route::resource('posts', PostController::class);
    
    // Custom routes with policy
    Route::patch('posts/{post}/publish', [PostController::class, 'publish'])
         ->name('posts.publish')
         ->can('publish', 'post');
         
    Route::patch('posts/{post}/feature', [PostController::class, 'feature'])
         ->name('posts.feature')
         ->can('feature', 'post');
});
```

### ৬. **Testing Policies**:
```php
// tests/Unit/PostPolicyTest.php
class PostPolicyTest extends TestCase
{
    public function test_user_can_view_own_post()
    {
        $user = User::factory()->create();
        $post = Post::factory()->create(['user_id' => $user->id]);

        $this->assertTrue($user->can('view', $post));
    }

    public function test_user_cannot_update_others_post()
    {
        $user = User::factory()->create();
        $otherUser = User::factory()->create();
        $post = Post::factory()->create(['user_id' => $otherUser->id]);

        $this->assertFalse($user->can('update', $post));
    }

    public function test_admin_can_delete_any_post()
    {
        $admin = User::factory()->admin()->create();
        $post = Post::factory()->create();

        $this->assertTrue($admin->can('delete', $post));
    }
}
```

---

## কখন Policy ব্যবহার করবেন?

### ✅ **ব্যবহার করুন যখন:**
- Model-based authorization প্রয়োজন
- Complex permission logic আছে
- Multiple controllers এ same authorization logic
- Clean এবং testable code চান
- Role-based access control implement করতে চান

### ❌ **ব্যবহার করবেন না যখন:**
- Simple boolean check (যেমন: `auth()->check()`)
- One-time authorization logic
- Non-model based permissions
- Very simple applications

---

## সারসংক্ষেপ

Laravel Policy হলো **Authorization Logic** organize করার **best practice**। এটি আপনার code কে **clean, reusable এবং testable** করে তোলে। **Model-based permissions** handle করার জন্য এটি **industry standard** approach।

**মূল বিষয়গুলো:**
- ✅ Clean code organization
- ✅ Reusable authorization logic  
- ✅ Easy testing
- ✅ Centralized permission management
- ✅ Blade template integration
- ✅ API resource support

Policy ব্যবহার করে আপনি **professional এবং maintainable** Laravel applications তৈরি করতে পারবেন।