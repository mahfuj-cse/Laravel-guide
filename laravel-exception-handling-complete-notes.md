# Laravel Exception Handling - Complete Notes

## 📚 Exception Handling কি?

**Exception** = Program চলার সময় যে error/problem হয়

### কেন Exception Handling করবেন?
1. **App Crash রোধ করা** - Error হলে app বন্ধ না হওয়া
2. **User Experience** - User কে proper message দেখানো
3. **Debugging** - Error এর exact location জানা
4. **Security** - Sensitive info hide করা
5. **Logging** - Error track করা

---

## 🔥 সবচেয়ে বেশি ব্যবহৃত Exception Types

### 1. ModelNotFoundException (সবচেয়ে Common)
```php
// ❌ Without handling - App crashes
$user = User::findOrFail($id);

// ✅ With handling
try {
    $user = User::findOrFail($id);
} catch (ModelNotFoundException $e) {
    return response()->json(['error' => 'User not found'], 404);
}
```

### 2. ValidationException (Form/API Validation)
```php
try {
    $request->validate([
        'email' => 'required|email',
        'password' => 'required|min:8'
    ]);
} catch (ValidationException $e) {
    return response()->json([
        'error' => 'Validation failed',
        'messages' => $e->errors()
    ], 422);
}
```

### 3. QueryException (Database Errors)
```php
try {
    User::create($data);
} catch (QueryException $e) {
    if ($e->getCode() === '23000') { // Duplicate entry
        return response()->json(['error' => 'Email already exists'], 409);
    }
    return response()->json(['error' => 'Database error'], 500);
}
```

### 4. AuthenticationException (Login/Auth)
```php
try {
    if (!Auth::attempt($credentials)) {
        throw new AuthenticationException('Invalid credentials');
    }
} catch (AuthenticationException $e) {
    return response()->json(['error' => 'Login failed'], 401);
}
```

---

## 🎯 Basic Try-Catch Structure

### Simple Try-Catch
```php
try {
    // Risky code
    $result = riskyOperation();
} catch (Exception $e) {
    // Handle error
    return response()->json(['error' => 'Something went wrong'], 500);
}
```

### Multiple Catch (Most Important Pattern)
```php
try {
    $user = User::findOrFail($id);
    $user->update($data);
} catch (ModelNotFoundException $e) {
    // Specific error first
    return response()->json(['error' => 'User not found'], 404);
} catch (QueryException $e) {
    // Database error
    return response()->json(['error' => 'Update failed'], 500);
} catch (Exception $e) {
    // General error last
    return response()->json(['error' => 'Unexpected error'], 500);
}
```

### Try-Catch-Finally
```php
$file = null;
try {
    $file = fopen('data.txt', 'r');
    $content = fread($file, 1024);
} catch (Exception $e) {
    Log::error('File error: ' . $e->getMessage());
} finally {
    // Always runs
    if ($file) {
        fclose($file);
    }
}
```

---

## 🚀 Production Ready Patterns

### 1. Controller Level Exception Handling
```php
class UserController extends Controller
{
    public function show($id)
    {
        try {
            $user = User::with('profile')->findOrFail($id);
            return response()->json($user);
        } catch (ModelNotFoundException $e) {
            return $this->errorResponse('User not found', 404);
        } catch (Exception $e) {
            Log::error('User fetch failed', ['id' => $id, 'error' => $e->getMessage()]);
            return $this->errorResponse('Failed to fetch user', 500);
        }
    }
    
    private function errorResponse($message, $code)
    {
        return response()->json(['error' => $message], $code);
    }
}
```

### 2. Service Layer Exception Handling
```php
class PaymentService
{
    public function processPayment($amount, $cardToken)
    {
        try {
            // Payment gateway call
            $response = $this->gateway->charge($amount, $cardToken);
            
            if (!$response->success) {
                throw new PaymentFailedException($response->message);
            }
            
            return $response;
        } catch (PaymentFailedException $e) {
            Log::warning('Payment failed', ['amount' => $amount, 'error' => $e->getMessage()]);
            throw $e; // Re-throw for controller to handle
        } catch (Exception $e) {
            Log::error('Payment system error', ['error' => $e->getMessage()]);
            throw new PaymentSystemException('Payment system unavailable');
        }
    }
}
```

### 3. API Resource Exception Handling
```php
class ApiController extends Controller
{
    public function store(Request $request)
    {
        try {
            $validated = $request->validate([
                'name' => 'required|string',
                'email' => 'required|email|unique:users'
            ]);
            
            $user = User::create($validated);
            
            return response()->json([
                'success' => true,
                'data' => $user
            ], 201);
            
        } catch (ValidationException $e) {
            return response()->json([
                'success' => false,
                'error' => 'Validation failed',
                'messages' => $e->errors()
            ], 422);
        } catch (QueryException $e) {
            return response()->json([
                'success' => false,
                'error' => 'Failed to create user'
            ], 500);
        }
    }
}
```

---

## 🛠️ Exception Object Properties (জানা জরুরি)

```php
catch (Exception $e) {
    $e->getMessage()        // Error message
    $e->getCode()          // Error code (0, 404, 500 etc)
    $e->getFile()          // File path where error occurred
    $e->getLine()          // Line number
    $e->getTrace()         // Full stack trace (array)
    $e->getTraceAsString() // Stack trace as string
    $e->getPrevious()      // Previous exception (if chained)
}
```

### Practical Usage:
```php
catch (Exception $e) {
    Log::error('Critical error occurred', [
        'message' => $e->getMessage(),
        'file' => $e->getFile(),
        'line' => $e->getLine(),
        'user_id' => auth()->id(),
        'url' => request()->fullUrl()
    ]);
}
```

---

## 🎨 Custom Exceptions (Advanced)

### Creating Custom Exception
```php
// app/Exceptions/PaymentFailedException.php
class PaymentFailedException extends Exception
{
    protected $paymentId;
    
    public function __construct($message, $paymentId = null, $code = 0)
    {
        parent::__construct($message, $code);
        $this->paymentId = $paymentId;
    }
    
    public function getPaymentId()
    {
        return $this->paymentId;
    }
}
```

### Using Custom Exception
```php
try {
    $payment = Payment::create($data);
    
    if (!$payment->process()) {
        throw new PaymentFailedException('Payment processing failed', $payment->id);
    }
} catch (PaymentFailedException $e) {
    Log::error('Payment failed', [
        'payment_id' => $e->getPaymentId(),
        'message' => $e->getMessage()
    ]);
    
    return response()->json(['error' => 'Payment failed'], 400);
}
```

---

## 🌐 Global Exception Handler

### app/Exceptions/Handler.php (Most Important File)
```php
class Handler extends ExceptionHandler
{
    public function render($request, Throwable $exception)
    {
        // API requests
        if ($request->expectsJson()) {
            return $this->handleApiException($exception);
        }
        
        return parent::render($request, $exception);
    }
    
    private function handleApiException($exception)
    {
        // Model not found
        if ($exception instanceof ModelNotFoundException) {
            return response()->json(['error' => 'Resource not found'], 404);
        }
        
        // Validation error
        if ($exception instanceof ValidationException) {
            return response()->json([
                'error' => 'Validation failed',
                'messages' => $exception->errors()
            ], 422);
        }
        
        // Authentication error
        if ($exception instanceof AuthenticationException) {
            return response()->json(['error' => 'Unauthenticated'], 401);
        }
        
        // Database error
        if ($exception instanceof QueryException) {
            Log::error('Database error', ['error' => $exception->getMessage()]);
            return response()->json(['error' => 'Database error occurred'], 500);
        }
        
        // General error
        Log::error('Unexpected error', [
            'message' => $exception->getMessage(),
            'file' => $exception->getFile(),
            'line' => $exception->getLine()
        ]);
        
        return response()->json(['error' => 'Internal server error'], 500);
    }
}
```

---

## 📊 Logging Best Practices

### Proper Error Logging
```php
try {
    $result = criticalOperation();
} catch (Exception $e) {
    // Environment-based logging
    if (app()->environment('production')) {
        // Production: Less details
        Log::error('Operation failed', [
            'user_id' => auth()->id(),
            'operation' => 'critical_operation',
            'message' => $e->getMessage()
        ]);
    } else {
        // Development: Full details
        Log::error('Operation failed', [
            'message' => $e->getMessage(),
            'file' => $e->getFile(),
            'line' => $e->getLine(),
            'trace' => $e->getTraceAsString()
        ]);
    }
    
    return response()->json(['error' => 'Operation failed'], 500);
}
```

### Log Levels
```php
Log::emergency('System is unusable');
Log::alert('Action must be taken immediately');
Log::critical('Critical conditions');
Log::error('Runtime errors');        // Most used
Log::warning('Exceptional occurrences'); // Most used
Log::notice('Normal but significant events');
Log::info('Interesting events');     // Most used
Log::debug('Detailed debug information');
```

---

## ⚡ Performance Considerations

### Do's and Don'ts
```php
// ✅ Good: Specific exceptions
try {
    $user = User::findOrFail($id);
} catch (ModelNotFoundException $e) {
    // Handle specific case
}

// ❌ Bad: Too broad exception catching
try {
    $user = User::findOrFail($id);
} catch (Exception $e) {
    // Catches everything, might hide bugs
}

// ✅ Good: Use exceptions for exceptional cases
if ($user->age < 18) {
    return response()->json(['error' => 'Age must be 18+'], 400);
}

// ❌ Bad: Using exceptions for normal flow control
try {
    if ($user->age < 18) {
        throw new Exception('Age must be 18+');
    }
} catch (Exception $e) {
    return response()->json(['error' => $e->getMessage()], 400);
}
```

---

## 🔄 Retry Pattern (Advanced)

```php
function retryOperation(callable $operation, $maxRetries = 3, $delay = 1)
{
    $attempt = 0;
    
    while ($attempt < $maxRetries) {
        try {
            return $operation();
        } catch (Exception $e) {
            $attempt++;
            
            if ($attempt >= $maxRetries) {
                throw $e; // Final attempt failed
            }
            
            Log::warning("Operation failed, retrying... Attempt: {$attempt}", [
                'error' => $e->getMessage()
            ]);
            
            sleep($delay * $attempt); // Exponential backoff
        }
    }
}

// Usage
try {
    $result = retryOperation(function() {
        return Http::timeout(5)->get('https://api.example.com/data');
    });
} catch (Exception $e) {
    Log::error('API call failed after retries', ['error' => $e->getMessage()]);
    return response()->json(['error' => 'Service unavailable'], 503);
}
```

---

## 📋 Quick Reference Checklist

### ✅ Production Checklist:
1. **Always catch specific exceptions first**
2. **Log errors with proper context**
3. **Don't expose sensitive information to users**
4. **Use environment-based error messages**
5. **Clean up resources in finally blocks**
6. **Implement global exception handler**
7. **Monitor exception rates**
8. **Use custom exceptions for business logic**

### 🚨 Common Mistakes to Avoid:
1. **Empty catch blocks** - `catch (Exception $e) {}`
2. **Catching Exception too broadly**
3. **Not logging errors**
4. **Exposing stack traces to users**
5. **Using exceptions for normal flow control**

---

## 💡 Real-World Example (Complete)

```php
class OrderController extends Controller
{
    public function store(Request $request)
    {
        DB::beginTransaction();
        
        try {
            // Validate input
            $validated = $request->validate([
                'product_id' => 'required|exists:products,id',
                'quantity' => 'required|integer|min:1'
            ]);
            
            // Check stock
            $product = Product::findOrFail($validated['product_id']);
            if ($product->stock < $validated['quantity']) {
                throw new InsufficientStockException('Not enough stock available');
            }
            
            // Create order
            $order = Order::create([
                'user_id' => auth()->id(),
                'product_id' => $validated['product_id'],
                'quantity' => $validated['quantity'],
                'total' => $product->price * $validated['quantity']
            ]);
            
            // Update stock
            $product->decrement('stock', $validated['quantity']);
            
            // Process payment
            $paymentResult = app(PaymentService::class)->processPayment(
                $order->total, 
                $request->payment_token
            );
            
            $order->update(['payment_id' => $paymentResult->id]);
            
            DB::commit();
            
            return response()->json([
                'success' => true,
                'order' => $order,
                'message' => 'Order placed successfully'
            ], 201);
            
        } catch (ValidationException $e) {
            DB::rollBack();
            return response()->json([
                'success' => false,
                'error' => 'Validation failed',
                'messages' => $e->errors()
            ], 422);
            
        } catch (ModelNotFoundException $e) {
            DB::rollBack();
            return response()->json([
                'success' => false,
                'error' => 'Product not found'
            ], 404);
            
        } catch (InsufficientStockException $e) {
            DB::rollBack();
            return response()->json([
                'success' => false,
                'error' => $e->getMessage()
            ], 400);
            
        } catch (PaymentFailedException $e) {
            DB::rollBack();
            Log::warning('Payment failed for order', [
                'user_id' => auth()->id(),
                'product_id' => $validated['product_id'] ?? null,
                'error' => $e->getMessage()
            ]);
            return response()->json([
                'success' => false,
                'error' => 'Payment processing failed'
            ], 400);
            
        } catch (Exception $e) {
            DB::rollBack();
            Log::error('Order creation failed', [
                'user_id' => auth()->id(),
                'error' => $e->getMessage(),
                'trace' => $e->getTraceAsString()
            ]);
            return response()->json([
                'success' => false,
                'error' => 'Failed to create order'
            ], 500);
        }
    }
}
```

এই notes গুলো Laravel exception handling এর 90% use case cover করে। Practice করলেই expert হয়ে যাবেন!