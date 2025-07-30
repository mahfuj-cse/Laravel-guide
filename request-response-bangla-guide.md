# 6️⃣ Laravel Request & Response - বিস্তারিত বাংলা গাইড

## 📋 সূচিপত্র
- [Request & Response কি?](#request--response-কি)
- [Request Handling](#request-handling)
- [Form Requests](#form-requests)
- [Validation](#validation)
- [Response Types](#response-types)
- [JSON Responses](#json-responses)
- [File Handling](#file-handling)
- [Advanced Techniques](#advanced-techniques)

---

## Request & Response কি?

**Request** হলো client থেকে server এ আসা **HTTP Request**, আর **Response** হলো server থেকে client এ পাঠানো **HTTP Response**।

### 🔄 Request-Response Cycle:
```
Client → HTTP Request → Laravel → Processing → HTTP Response → Client
```

### 🎯 Laravel এ Request & Response:
- ✅ **Request Object** - Input data, headers, files access
- ✅ **Form Requests** - Validation এবং authorization
- ✅ **Response Object** - Different response types
- ✅ **JSON API** - API development এর জন্য
- ✅ **File Upload/Download** - File handling

---

## Request Handling

### ১. Basic Request Usage:
```php
<?php
// Controller এ Request ব্যবহার

use Illuminate\Http\Request;

class PostController extends Controller
{
    public function store(Request $request)
    {
        // Get all input data
        $data = $request->all();

        // Get specific input
        $title = $request->input('title');
        $content = $request->input('content', 'Default content');

        // Get input with dot notation
        $name = $request->input('user.name');

        // Check if input exists
        if ($request->has('title')) {
            // Title field exists
        }

        // Check if input has value
        if ($request->filled('title')) {
            // Title field has value
        }

        // Get only specific fields
        $data = $request->only(['title', 'content']);

        // Get all except specific fields
        $data = $request->except(['_token', '_method']);

        return response()->json(['data' => $data]);
    }
}
```

### ২. Request Information:
```php
<?php

class RequestInfoController extends Controller
{
    public function info(Request $request)
    {
        // URL Information
        $url = $request->url(); // Without query string
        $fullUrl = $request->fullUrl(); // With query string
        $path = $request->path(); // URI path

        // HTTP Method
        $method = $request->method();
        $isPost = $request->isMethod('post');

        // Headers
        $userAgent = $request->header('User-Agent');
        $allHeaders = $request->headers->all();

        // IP Address
        $ip = $request->ip();
        $ips = $request->ips(); // All IPs (including proxies)

        // Query Parameters
        $search = $request->query('search');
        $allQuery = $request->query();

        // Request Type
        $isAjax = $request->ajax();
        $wantsJson = $request->wantsJson();
        $expectsJson = $request->expectsJson();

        return response()->json([
            'url' => $url,
            'method' => $method,
            'ip' => $ip,
            'user_agent' => $userAgent,
            'is_ajax' => $isAjax
        ]);
    }
}
```

### ৩. Input Validation (Basic):
```php
<?php

class PostController extends Controller
{
    public function store(Request $request)
    {
        // Basic validation
        $request->validate([
            'title' => 'required|max:255',
            'content' => 'required|min:10',
            'category_id' => 'required|exists:categories,id',
            'tags' => 'array',
            'tags.*' => 'string|max:50',
            'published_at' => 'nullable|date|after:now'
        ]);

        // Custom error messages
        $request->validate([
            'title' => 'required|max:255',
            'email' => 'required|email|unique:users'
        ], [
            'title.required' => 'পোস্টের শিরোনাম আবশ্যক',
            'title.max' => 'শিরোনাম সর্বোচ্চ ২৫৫ অক্ষরের হতে পারে',
            'email.unique' => 'এই ইমেইল ঠিকানা ইতিমধ্যে ব্যবহৃত হয়েছে'
        ]);

        $post = Post::create($request->validated());
        
        return redirect()->route('posts.show', $post);
    }
}
```

---

## Form Requests

### ১. Form Request তৈরি করা:
```bash
# Basic Form Request
php artisan make:request StorePostRequest

# Form Request with validation rules
php artisan make:request UpdateUserRequest
```

### ২. Complete Form Request Example:
```php
<?php
// app/Http/Requests/StorePostRequest.php

namespace App\Http\Requests;

use Illuminate\Foundation\Http\FormRequest;
use Illuminate\Validation\Rule;

class StorePostRequest extends FormRequest
{
    /**
     * Determine if the user is authorized to make this request.
     */
    public function authorize(): bool
    {
        // Check if user can create posts
        return auth()->check() && auth()->user()->can('create', Post::class);
    }

    /**
     * Get the validation rules that apply to the request.
     */
    public function rules(): array
    {
        return [
            'title' => 'required|string|max:255|unique:posts,title',
            'content' => 'required|string|min:100',
            'excerpt' => 'nullable|string|max:500',
            'category_id' => 'required|exists:categories,id',
            'tags' => 'nullable|array|max:5',
            'tags.*' => 'string|max:50',
            'featured_image' => 'nullable|image|mimes:jpeg,png,jpg|max:2048',
            'status' => ['required', Rule::in(['draft', 'published', 'scheduled'])],
            'published_at' => 'nullable|date|after:now',
            'meta_title' => 'nullable|string|max:60',
            'meta_description' => 'nullable|string|max:160'
        ];
    }

    /**
     * Get custom error messages for validator errors.
     */
    public function messages(): array
    {
        return [
            'title.required' => 'পোস্টের শিরোনাম আবশ্যক',
            'title.unique' => 'এই শিরোনামের পোস্ট ইতিমধ্যে রয়েছে',
            'content.required' => 'পোস্টের বিষয়বস্তু আবশ্যক',
            'content.min' => 'পোস্টের বিষয়বস্তু কমপক্ষে ১০০ অক্ষরের হতে হবে',
            'category_id.required' => 'ক্যাটেগরি নির্বাচন করুন',
            'category_id.exists' => 'নির্বাচিত ক্যাটেগরি বৈধ নয়',
            'featured_image.image' => 'ফিচার ইমেজ একটি ছবি হতে হবে',
            'featured_image.max' => 'ছবির সাইজ সর্বোচ্চ ২ MB হতে পারে'
        ];
    }

    /**
     * Get custom attributes for validator errors.
     */
    public function attributes(): array
    {
        return [
            'title' => 'শিরোনাম',
            'content' => 'বিষয়বস্তু',
            'category_id' => 'ক্যাটেগরি',
            'featured_image' => 'ফিচার ইমেজ'
        ];
    }

    /**
     * Prepare the data for validation.
     */
    protected function prepareForValidation(): void
    {
        $this->merge([
            'slug' => Str::slug($this->title),
            'user_id' => auth()->id()
        ]);
    }

    /**
     * Configure the validator instance.
     */
    public function withValidator($validator): void
    {
        $validator->after(function ($validator) {
            if ($this->status === 'published' && !$this->filled('published_at')) {
                $this->merge(['published_at' => now()]);
            }

            // Custom validation logic
            if ($this->somethingElseIsInvalid()) {
                $validator->errors()->add('field', 'Something is wrong with this field!');
            }
        });
    }

    /**
     * Handle a failed authorization attempt.
     */
    protected function failedAuthorization(): void
    {
        throw new HttpResponseException(
            response()->json(['message' => 'আপনার পোস্ট তৈরির অনুমতি নেই'], 403)
        );
    }
}
```

### ৩. Form Request ব্যবহার:
```php
<?php

class PostController extends Controller
{
    public function store(StorePostRequest $request)
    {
        // Validation এবং authorization automatically handled
        
        // Get validated data only
        $validatedData = $request->validated();

        // Create post
        $post = Post::create($validatedData);

        // Handle file upload if exists
        if ($request->hasFile('featured_image')) {
            $imagePath = $request->file('featured_image')->store('posts', 'public');
            $post->update(['featured_image' => $imagePath]);
        }

        return redirect()->route('posts.show', $post)
                        ->with('success', 'পোস্ট সফলভাবে তৈরি হয়েছে');
    }

    public function update(UpdatePostRequest $request, Post $post)
    {
        $post->update($request->validated());
        
        return redirect()->route('posts.show', $post)
                        ->with('success', 'পোস্ট আপডেট হয়েছে');
    }
}
```

---

## Validation

### ১. Available Validation Rules:
```php
<?php

// Common validation rules
$rules = [
    // Required
    'name' => 'required',
    'email' => 'required|email',
    'password' => 'required|min:8|confirmed',

    // String validation
    'title' => 'string|max:255',
    'content' => 'string|min:10',

    // Numeric validation
    'age' => 'integer|min:18|max:100',
    'price' => 'numeric|min:0',
    'rating' => 'decimal:1,2', // 1-2 decimal places

    // Date validation
    'birth_date' => 'date|before:today',
    'event_date' => 'date|after:tomorrow',
    'published_at' => 'date_format:Y-m-d H:i:s',

    // File validation
    'avatar' => 'image|mimes:jpeg,png,jpg|max:2048',
    'document' => 'file|mimes:pdf,doc,docx|max:10240',

    // Array validation
    'tags' => 'array|min:1|max:5',
    'tags.*' => 'string|max:50',

    // Database validation
    'email' => 'unique:users,email',
    'category_id' => 'exists:categories,id',

    // Conditional validation
    'phone' => 'required_if:contact_method,phone',
    'address' => 'required_unless:delivery_method,pickup',

    // Custom regex
    'phone' => 'regex:/^01[3-9]\d{8}$/',
    'postal_code' => 'regex:/^\d{4}$/',

    // Boolean
    'terms_accepted' => 'accepted',
    'is_active' => 'boolean',

    // URL validation
    'website' => 'url',
    'profile_url' => 'active_url'
];
```

### ২. Custom Validation Rules:
```php
<?php
// Create custom rule
php artisan make:rule PhoneNumber

// app/Rules/PhoneNumber.php
namespace App\Rules;

use Illuminate\Contracts\Validation\Rule;

class PhoneNumber implements Rule
{
    public function passes($attribute, $value)
    {
        // Bangladesh phone number validation
        return preg_match('/^(\+88)?01[3-9]\d{8}$/', $value);
    }

    public function message()
    {
        return 'ফোন নম্বরটি সঠিক বাংলাদেশী ফরম্যাটে দিন (যেমন: 01712345678)';
    }
}

// Usage
$request->validate([
    'phone' => ['required', new PhoneNumber]
]);
```

### ৩. Conditional Validation:
```php
<?php

class UserController extends Controller
{
    public function store(Request $request)
    {
        $rules = [
            'name' => 'required|string|max:255',
            'email' => 'required|email|unique:users',
            'user_type' => 'required|in:individual,company'
        ];

        // Conditional rules based on user type
        if ($request->user_type === 'company') {
            $rules['company_name'] = 'required|string|max:255';
            $rules['tax_id'] = 'required|string|unique:companies';
        } else {
            $rules['first_name'] = 'required|string|max:255';
            $rules['last_name'] = 'required|string|max:255';
            $rules['date_of_birth'] = 'required|date|before:18 years ago';
        }

        $request->validate($rules);

        // Process user creation
    }
}
```

---

## Response Types

### ১. View Responses:
```php
<?php

class PostController extends Controller
{
    public function index()
    {
        $posts = Post::paginate(10);
        
        // Basic view response
        return view('posts.index', compact('posts'));
    }

    public function show(Post $post)
    {
        // View with data
        return view('posts.show', [
            'post' => $post,
            'comments' => $post->comments()->latest()->get(),
            'relatedPosts' => Post::where('category_id', $post->category_id)
                                 ->where('id', '!=', $post->id)
                                 ->limit(3)
                                 ->get()
        ]);
    }

    public function create()
    {
        $categories = Category::all();
        
        // View with shared data
        return view('posts.create')
                ->with('categories', $categories)
                ->with('pageTitle', 'নতুন পোস্ট তৈরি করুন');
    }
}
```

### ২. Redirect Responses:
```php
<?php

class PostController extends Controller
{
    public function store(Request $request)
    {
        $post = Post::create($request->validated());

        // Basic redirect
        return redirect('/posts');

        // Named route redirect
        return redirect()->route('posts.show', $post);

        // Redirect with flash data
        return redirect()->route('posts.index')
                        ->with('success', 'পোস্ট তৈরি হয়েছে')
                        ->with('post_id', $post->id);

        // Redirect back (previous page)
        return redirect()->back();

        // Redirect with input (for form errors)
        return redirect()->back()->withInput();

        // Redirect with errors
        return redirect()->back()
                        ->withErrors(['title' => 'শিরোনাম আবশ্যক'])
                        ->withInput();
    }

    public function destroy(Post $post)
    {
        $post->delete();

        // Redirect with different message types
        return redirect()->route('posts.index')
                        ->with([
                            'success' => 'পোস্ট মুছে ফেলা হয়েছে',
                            'deleted_post' => $post->title
                        ]);
    }
}
```

### ৩. File Responses:
```php
<?php

class FileController extends Controller
{
    public function download($filename)
    {
        $filePath = storage_path('app/downloads/' . $filename);

        // File download
        return response()->download($filePath);

        // Download with custom name
        return response()->download($filePath, 'custom-name.pdf');

        // Download with headers
        return response()->download($filePath, 'report.pdf', [
            'Content-Type' => 'application/pdf'
        ]);
    }

    public function stream($filename)
    {
        $filePath = storage_path('app/files/' . $filename);

        // Stream file (for viewing in browser)
        return response()->file($filePath);

        // Stream with headers
        return response()->file($filePath, [
            'Content-Type' => 'application/pdf',
            'Content-Disposition' => 'inline; filename="document.pdf"'
        ]);
    }

    public function generatePdf()
    {
        $pdf = PDF::loadView('reports.monthly');

        // Return PDF response
        return $pdf->download('monthly-report.pdf');
        
        // Or stream in browser
        return $pdf->stream('monthly-report.pdf');
    }
}
```

---

## JSON Responses

### ১. Basic JSON Responses:
```php
<?php

class ApiController extends Controller
{
    public function index()
    {
        $posts = Post::all();

        // Basic JSON response
        return response()->json($posts);

        // JSON with status code
        return response()->json($posts, 200);

        // JSON with custom structure
        return response()->json([
            'success' => true,
            'message' => 'Posts retrieved successfully',
            'data' => $posts,
            'meta' => [
                'total' => $posts->count(),
                'timestamp' => now()->toISOString()
            ]
        ]);
    }

    public function store(Request $request)
    {
        $request->validate([
            'title' => 'required|max:255',
            'content' => 'required'
        ]);

        $post = Post::create($request->validated());

        // Success response
        return response()->json([
            'success' => true,
            'message' => 'Post created successfully',
            'data' => $post
        ], 201);
    }

    public function show($id)
    {
        $post = Post::find($id);

        if (!$post) {
            // Error response
            return response()->json([
                'success' => false,
                'message' => 'Post not found',
                'error_code' => 'POST_NOT_FOUND'
            ], 404);
        }

        return response()->json([
            'success' => true,
            'data' => $post
        ]);
    }
}
```

### ২. API Resource Responses:
```bash
# Create API Resource
php artisan make:resource PostResource
php artisan make:resource PostCollection
```

```php
<?php
// app/Http/Resources/PostResource.php

namespace App\Http\Resources;

use Illuminate\Http\Resources\Json\JsonResource;

class PostResource extends JsonResource
{
    public function toArray($request)
    {
        return [
            'id' => $this->id,
            'title' => $this->title,
            'slug' => $this->slug,
            'excerpt' => $this->excerpt,
            'content' => $this->when($request->include_content, $this->content),
            'status' => $this->status,
            'featured_image' => $this->featured_image ? asset('storage/' . $this->featured_image) : null,
            'author' => [
                'id' => $this->user->id,
                'name' => $this->user->name,
                'avatar' => $this->user->avatar_url
            ],
            'category' => [
                'id' => $this->category->id,
                'name' => $this->category->name,
                'slug' => $this->category->slug
            ],
            'tags' => $this->tags->pluck('name'),
            'comments_count' => $this->comments_count,
            'views_count' => $this->views_count,
            'published_at' => $this->published_at?->toISOString(),
            'created_at' => $this->created_at->toISOString(),
            'updated_at' => $this->updated_at->toISOString(),
            
            // Conditional fields
            'edit_url' => $this->when(
                $request->user()?->can('update', $this->resource),
                route('posts.edit', $this->id)
            ),
            
            // Relationships
            'comments' => CommentResource::collection($this->whenLoaded('comments')),
            'related_posts' => PostResource::collection($this->whenLoaded('relatedPosts'))
        ];
    }

    public function with($request)
    {
        return [
            'meta' => [
                'timestamp' => now()->toISOString(),
                'version' => '1.0'
            ]
        ];
    }
}

// Usage in Controller
class ApiPostController extends Controller
{
    public function index()
    {
        $posts = Post::with(['user', 'category', 'tags'])
                    ->withCount('comments')
                    ->paginate(15);

        return PostResource::collection($posts);
    }

    public function show(Post $post)
    {
        $post->load(['user', 'category', 'tags', 'comments.user']);
        
        return new PostResource($post);
    }
}
```

### ৩. Error Handling in API:
```php
<?php

class ApiExceptionHandler
{
    public function render($request, Throwable $exception)
    {
        if ($request->expectsJson()) {
            // Validation errors
            if ($exception instanceof ValidationException) {
                return response()->json([
                    'success' => false,
                    'message' => 'Validation failed',
                    'errors' => $exception->errors()
                ], 422);
            }

            // Model not found
            if ($exception instanceof ModelNotFoundException) {
                return response()->json([
                    'success' => false,
                    'message' => 'Resource not found',
                    'error_code' => 'RESOURCE_NOT_FOUND'
                ], 404);
            }

            // Authentication errors
            if ($exception instanceof AuthenticationException) {
                return response()->json([
                    'success' => false,
                    'message' => 'Unauthenticated',
                    'error_code' => 'UNAUTHENTICATED'
                ], 401);
            }

            // Authorization errors
            if ($exception instanceof AuthorizationException) {
                return response()->json([
                    'success' => false,
                    'message' => 'Unauthorized',
                    'error_code' => 'UNAUTHORIZED'
                ], 403);
            }

            // Generic server error
            return response()->json([
                'success' => false,
                'message' => 'Internal server error',
                'error_code' => 'INTERNAL_ERROR'
            ], 500);
        }

        return parent::render($request, $exception);
    }
}
```

---

## File Handling

### ১. File Upload:
```php
<?php

class FileUploadController extends Controller
{
    public function upload(Request $request)
    {
        $request->validate([
            'file' => 'required|file|max:10240', // 10MB max
            'image' => 'required|image|mimes:jpeg,png,jpg|max:2048', // 2MB max
            'document' => 'required|mimes:pdf,doc,docx|max:5120' // 5MB max
        ]);

        // Basic file upload
        if ($request->hasFile('file')) {
            $file = $request->file('file');
            
            // Store in storage/app/uploads
            $path = $file->store('uploads');
            
            // Store in public disk
            $path = $file->store('uploads', 'public');
            
            // Store with custom name
            $filename = time() . '_' . $file->getClientOriginalName();
            $path = $file->storeAs('uploads', $filename, 'public');
        }

        // Image upload with processing
        if ($request->hasFile('image')) {
            $image = $request->file('image');
            
            // Generate unique filename
            $filename = Str::uuid() . '.' . $image->getClientOriginalExtension();
            
            // Store original
            $path = $image->storeAs('images/original', $filename, 'public');
            
            // Create thumbnail (using Intervention Image)
            $thumbnailPath = 'images/thumbnails/' . $filename;
            Image::make($image)
                ->resize(300, 300, function ($constraint) {
                    $constraint->aspectRatio();
                    $constraint->upsize();
                })
                ->save(storage_path('app/public/' . $thumbnailPath));
        }

        return response()->json([
            'success' => true,
            'message' => 'File uploaded successfully',
            'path' => $path,
            'url' => asset('storage/' . $path)
        ]);
    }

    public function multipleUpload(Request $request)
    {
        $request->validate([
            'files' => 'required|array|max:5',
            'files.*' => 'file|mimes:jpeg,png,jpg,pdf|max:2048'
        ]);

        $uploadedFiles = [];

        foreach ($request->file('files') as $file) {
            $filename = time() . '_' . $file->getClientOriginalName();
            $path = $file->storeAs('uploads', $filename, 'public');
            
            $uploadedFiles[] = [
                'original_name' => $file->getClientOriginalName(),
                'filename' => $filename,
                'path' => $path,
                'url' => asset('storage/' . $path),
                'size' => $file->getSize(),
                'mime_type' => $file->getMimeType()
            ];
        }

        return response()->json([
            'success' => true,
            'message' => count($uploadedFiles) . ' files uploaded successfully',
            'files' => $uploadedFiles
        ]);
    }
}
```

### ২. File Information:
```php
<?php

public function fileInfo(Request $request)
{
    if ($request->hasFile('file')) {
        $file = $request->file('file');
        
        $info = [
            'original_name' => $file->getClientOriginalName(),
            'extension' => $file->getClientOriginalExtension(),
            'mime_type' => $file->getMimeType(),
            'size' => $file->getSize(), // bytes
            'size_human' => $this->formatBytes($file->getSize()),
            'is_valid' => $file->isValid(),
            'temp_path' => $file->getPathname()
        ];
        
        return response()->json($info);
    }
}

private function formatBytes($size, $precision = 2)
{
    $units = ['B', 'KB', 'MB', 'GB', 'TB'];
    
    for ($i = 0; $size > 1024 && $i < count($units) - 1; $i++) {
        $size /= 1024;
    }
    
    return round($size, $precision) . ' ' . $units[$i];
}
```

---

## Advanced Techniques

### ১. Response Macros:
```php
<?php
// app/Providers/ResponseMacroServiceProvider.php

use Illuminate\Support\ServiceProvider;
use Illuminate\Support\Facades\Response;

class ResponseMacroServiceProvider extends ServiceProvider
{
    public function boot()
    {
        // Success response macro
        Response::macro('success', function ($data = null, $message = 'Success', $code = 200) {
            return Response::json([
                'success' => true,
                'message' => $message,
                'data' => $data,
                'timestamp' => now()->toISOString()
            ], $code);
        });

        // Error response macro
        Response::macro('error', function ($message = 'Error', $code = 400, $errors = null) {
            return Response::json([
                'success' => false,
                'message' => $message,
                'errors' => $errors,
                'timestamp' => now()->toISOString()
            ], $code);
        });

        // Paginated response macro
        Response::macro('paginated', function ($data, $message = 'Data retrieved successfully') {
            return Response::json([
                'success' => true,
                'message' => $message,
                'data' => $data->items(),
                'pagination' => [
                    'current_page' => $data->currentPage(),
                    'last_page' => $data->lastPage(),
                    'per_page' => $data->perPage(),
                    'total' => $data->total(),
                    'from' => $data->firstItem(),
                    'to' => $data->lastItem()
                ]
            ]);
        });
    }
}

// Usage
return response()->success($posts, 'Posts retrieved successfully');
return response()->error('Validation failed', 422, $validator->errors());
return response()->paginated($posts);
```

### ২. Custom Request Class:
```php
<?php
// app/Http/Requests/BaseRequest.php

namespace App\Http\Requests;

use Illuminate\Foundation\Http\FormRequest;
use Illuminate\Contracts\Validation\Validator;
use Illuminate\Http\Exceptions\HttpResponseException;

class BaseRequest extends FormRequest
{
    protected function failedValidation(Validator $validator)
    {
        if ($this->expectsJson()) {
            throw new HttpResponseException(
                response()->json([
                    'success' => false,
                    'message' => 'Validation failed',
                    'errors' => $validator->errors()
                ], 422)
            );
        }

        parent::failedValidation($validator);
    }

    protected function failedAuthorization()
    {
        if ($this->expectsJson()) {
            throw new HttpResponseException(
                response()->json([
                    'success' => false,
                    'message' => 'Unauthorized'
                ], 403)
            );
        }

        parent::failedAuthorization();
    }
}
```

### ৩. Request/Response Logging:
```php
<?php
// app/Http/Middleware/LogRequests.php

class LogRequests
{
    public function handle($request, Closure $next)
    {
        $startTime = microtime(true);
        
        // Log request
        Log::info('API Request', [
            'method' => $request->method(),
            'url' => $request->fullUrl(),
            'ip' => $request->ip(),
            'user_agent' => $request->userAgent(),
            'payload' => $request->except(['password', 'password_confirmation'])
        ]);

        $response = $next($request);

        // Log response
        $executionTime = microtime(true) - $startTime;
        
        Log::info('API Response', [
            'status' => $response->getStatusCode(),
            'execution_time' => round($executionTime * 1000, 2) . 'ms',
            'memory_usage' => round(memory_get_peak_usage(true) / 1024 / 1024, 2) . 'MB'
        ]);

        return $response;
    }
}
```

---

## 🎯 Best Practices:

### ✅ **Request Handling:**
- Form Requests ব্যবহার করুন validation এর জন্য
- Input sanitization করুন
- File upload এর জন্য proper validation রাখুন
- Request size limit সেট করুন

### ✅ **Response:**
- Consistent response format ব্যবহার করুন
- Proper HTTP status codes দিন
- API Resources ব্যবহার করুন data transformation এর জন্য
- Error handling properly করুন

### ✅ **Security:**
- CSRF protection enable রাখুন
- File upload validation করুন
- Input validation সবসময় করুন
- Sensitive data response এ include করবেন না

---

## 📚 আরও জানতে:
- [Laravel Requests](https://laravel.com/docs/requests)
- [Laravel Responses](https://laravel.com/docs/responses)
- [Form Request Validation](https://laravel.com/docs/validation#form-request-validation)
- [API Resources](https://laravel.com/docs/eloquent-resources)