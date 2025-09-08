# Laravel Try-Catch & Exception Handling - বাংলা গাইড

## 📋 সূচিপত্র
- [Exception Handling কি এবং কেন প্রয়োজন](#exception-handling-কি-এবং-কেন-প্রয়োজন)
- [Try-Catch Basic Concepts](#try-catch-basic-concepts)
- [Laravel Exception Types](#laravel-exception-types)
- [Production Best Practices](#production-best-practices)
- [Advanced Exception Handling](#advanced-exception-handling)

---

## Exception Handling কি এবং কেন প্রয়োজন

### 🚨 Exception কি?
**Exception** হলো program execution এর সময় যে error বা unexpected situation ঘটে।

**Common Scenarios:**
- Database connection fail
- File not found
- Invalid user input
- API call timeout
- Memory limit exceeded

### কেন Exception Handling প্রয়োজন?
- **User Experience**: User কে proper error message দেখানো
- **Application Stability**: App crash না করে gracefully handle করা
- **Debugging**: Error এর exact location এবং cause জানা
- **Security**: Sensitive information leak না করা
- **Logging**: Error tracking এবং monitoring

```php
// ❌ Without Exception Handling
$user = User::findOrFail($id); // App crashes if user not found

// ✅ With Exception Handling
try {
    $user = User::findOrFail($id);
} catch (ModelNotFoundException $e) {
    return response()->json(['error' => 'User not found'], 404);
}
```

---

## Try-Catch Basic Concepts

### Basic Syntax:
```php
try {
    // Risky code যেখানে exception হতে পারে
    $result = riskyOperation();
} catch (SpecificException $e) {
    // Specific exception handle করা
    handleSpecificError($e);
} catch (Exception $e) {
    // General exception handle করা
    handleGeneralError($e);
} finally {
    // সবসময় execute হবে (optional)
    cleanup();
}
```

### Exception Object Properties:
```php
catch (Exception $e) {
    $e->getMessage()    // Error message
    $e->getCode()       // Error code
    $e->getFile()       // File where error occurred
    $e->getLine()       // Line number
    $e->getTrace()      // Stack trace
}
```

### Multiple Catch Blocks:
```php
try {
    $data = processPayment($amount);
} catch (InsufficientFundsException $e) {
    return response()->json(['error' => 'Insufficient funds'], 400);
} catch (PaymentGatewayException $e) {
    return response()->json(['error' => 'Payment gateway error'], 502);
} catch (Exception $e) {
    return response()->json(['error' => 'Payment failed'], 500);
}
```

---

## Laravel Exception Types

### Database Exceptions:
```php
use Illuminate\Database\QueryException;
use Illuminate\Database\Eloquent\ModelNotFoundException;

try {
    $user = User::findOrFail($id);
    $user->update($data);
} catch (ModelNotFoundException $e) {
    return response()->json(['error' => 'User not found'], 404);
} catch (QueryException $e) {
    if ($e->getCode() === '23000') { // Duplicate entry
        return response()->json(['error' => 'Email already exists'], 409);
    }
    return response()->json(['error' => 'Database error'], 500);
}
```

### Validation Exceptions:
```php
use Illuminate\Validation\ValidationException;

try {
    $validated = $request->validate([
        'email' => 'required|email|unique:users'
    ]);
} catch (ValidationException $e) {
    return response()->json([
        'error' => 'Validation failed',
        'messages' => $e->errors()
    ], 422);
}
```

### HTTP Exceptions:
```php
use Symfony\Component\HttpKernel\Exception\NotFoundHttpException;
use Symfony\Component\HttpKernel\Exception\UnauthorizedHttpException;

try {
    $response = Http::get('https://api.example.com/data');
    if ($response->failed()) {
        throw new Exception('API call failed');
    }
} catch (Exception $e) {
    return response()->json(['error' => 'External service unavailable'], 503);
}
```

---

## Production Best Practices

### 1. Specific Exception Handling:
```php
// ✅ Good: Specific exceptions first
try {
    $payment = processPayment($data);
} catch (InsufficientFundsException $e) {
    // Handle specific case
    return $this->handleInsufficientFunds($e);
} catch (PaymentException $e) {
    // Handle payment related errors
    return $this->handlePaymentError($e);
} catch (Exception $e) {
    // Handle unexpected errors
    return $this->handleUnexpectedError($e);
}
```

### 2. Proper Logging:
```php
try {
    $result = criticalOperation();
} catch (Exception $e) {
    // Log with context
    Log::error('Critical operation failed', [
        'user_id' => auth()->id(),
        'operation' => 'payment_processing',
        'error' => $e->getMessage(),
        'trace' => $e->getTraceAsString()
    ]);
    
    return response()->json(['error' => 'Operation failed'], 500);
}
```

### 3. User-Friendly Messages:
```php
try {
    $user = User::findOrFail($id);
} catch (ModelNotFoundException $e) {
    // Don't expose internal details
    return response()->json([
        'error' => 'The requested user could not be found'
    ], 404);
} catch (Exception $e) {
    // Generic message for unexpected errors
    return response()->json([
        'error' => 'Something went wrong. Please try again later.'
    ], 500);
}
```

### 4. Resource Cleanup:
```php
$file = null;
try {
    $file = fopen('large-file.txt', 'r');
    $data = processFile($file);
} catch (Exception $e) {
    Log::error('File processing failed: ' . $e->getMessage());
    throw $e; // Re-throw if needed
} finally {
    // Always cleanup resources
    if ($file) {
        fclose($file);
    }
}
```

---

## Advanced Exception Handling

### Custom Exceptions:
```php
// Create custom exception
class PaymentFailedException extends Exception
{
    protected $paymentId;
    
    public function __construct($message, $paymentId = null)
    {
        parent::__construct($message);
        $this->paymentId = $paymentId;
    }
    
    public function getPaymentId()
    {
        return $this->paymentId;
    }
}

// Usage
try {
    if (!$payment->process()) {
        throw new PaymentFailedException('Payment processing failed', $payment->id);
    }
} catch (PaymentFailedException $e) {
    Log::error('Payment failed', ['payment_id' => $e->getPaymentId()]);
}
```

### Global Exception Handler:
```php
// app/Exceptions/Handler.php
public function render($request, Throwable $exception)
{
    // API requests
    if ($request->expectsJson()) {
        return $this->handleApiException($request, $exception);
    }
    
    return parent::render($request, $exception);
}

private function handleApiException($request, $exception)
{
    if ($exception instanceof ModelNotFoundException) {
        return response()->json(['error' => 'Resource not found'], 404);
    }
    
    if ($exception instanceof ValidationException) {
        return response()->json([
            'error' => 'Validation failed',
            'messages' => $exception->errors()
        ], 422);
    }
    
    // Log unexpected errors
    Log::error('Unexpected API error', [
        'url' => $request->fullUrl(),
        'method' => $request->method(),
        'error' => $exception->getMessage()
    ]);
    
    return response()->json(['error' => 'Internal server error'], 500);
}
```

### Exception Monitoring:
```php
// Service for exception tracking
class ExceptionTracker
{
    public static function track(Exception $e, array $context = [])
    {
        $data = [
            'message' => $e->getMessage(),
            'file' => $e->getFile(),
            'line' => $e->getLine(),
            'trace' => $e->getTraceAsString(),
            'context' => $context,
            'user_id' => auth()->id(),
            'ip' => request()->ip(),
            'url' => request()->fullUrl(),
            'occurred_at' => now()
        ];
        
        // Store in database
        DB::table('exception_logs')->insert($data);
        
        // Send to external service (Sentry, Bugsnag)
        if (app()->environment('production')) {
            app('sentry')->captureException($e);
        }
    }
}

// Usage
try {
    $result = complexOperation();
} catch (Exception $e) {
    ExceptionTracker::track($e, ['operation' => 'complex_operation']);
    return response()->json(['error' => 'Operation failed'], 500);
}
```

---

## Advantages & Disadvantages

### ✅ Advantages:
- **Graceful Error Handling**: App doesn't crash
- **Better User Experience**: Proper error messages
- **Debugging**: Easy to identify error source
- **Security**: Prevents information leakage
- **Maintainability**: Centralized error handling
- **Monitoring**: Track application health

### ❌ Disadvantages:
- **Performance Overhead**: Try-catch has slight performance cost
- **Code Complexity**: More code to write and maintain
- **Over-catching**: Catching too broad exceptions can hide bugs
- **Resource Usage**: Exception objects consume memory

### When to Use:
```php
// ✅ Use for external dependencies
try {
    $response = Http::timeout(5)->get($apiUrl);
} catch (Exception $e) {
    // Handle API failure
}

// ✅ Use for user input validation
try {
    $validated = $request->validate($rules);
} catch (ValidationException $e) {
    // Handle validation errors
}

// ❌ Don't overuse for simple operations
if ($user->age < 18) {
    // Simple condition, no need for exception
    return response()->json(['error' => 'Age must be 18+'], 400);
}
```

---

## Production Recommendations

### 1. Environment-based Handling:
```php
try {
    $result = riskyOperation();
} catch (Exception $e) {
    if (app()->environment('production')) {
        // Production: Generic message
        Log::error('Operation failed', ['error' => $e->getMessage()]);
        return response()->json(['error' => 'Something went wrong'], 500);
    } else {
        // Development: Detailed error
        return response()->json([
            'error' => $e->getMessage(),
            'file' => $e->getFile(),
            'line' => $e->getLine()
        ], 500);
    }
}
```

### 2. Circuit Breaker Pattern:
```php
class CircuitBreaker
{
    public static function call(callable $operation, $maxFailures = 5)
    {
        $failures = Cache::get('circuit_breaker_failures', 0);
        
        if ($failures >= $maxFailures) {
            throw new Exception('Circuit breaker is open');
        }
        
        try {
            $result = $operation();
            Cache::forget('circuit_breaker_failures');
            return $result;
        } catch (Exception $e) {
            Cache::increment('circuit_breaker_failures');
            throw $e;
        }
    }
}
```

### 3. Retry Mechanism:
```php
function retryOperation(callable $operation, $maxRetries = 3)
{
    $attempt = 0;
    
    while ($attempt < $maxRetries) {
        try {
            return $operation();
        } catch (Exception $e) {
            $attempt++;
            
            if ($attempt >= $maxRetries) {
                throw $e;
            }
            
            // Wait before retry
            sleep(pow(2, $attempt)); // Exponential backoff
        }
    }
}
```

---

## সারসংক্ষেপ

### 🎯 Key Points:
- **Always handle exceptions** in production applications
- **Use specific exceptions** before general ones
- **Log errors properly** with context
- **Don't expose sensitive information** to users
- **Clean up resources** in finally blocks
- **Monitor exceptions** for application health

### 📊 Best Practices:
```php
// Production-ready exception handling
try {
    $result = criticalOperation();
    return response()->json(['data' => $result]);
} catch (SpecificException $e) {
    Log::warning('Specific error occurred', ['context' => $e->getMessage()]);
    return response()->json(['error' => 'Specific error message'], 400);
} catch (Exception $e) {
    Log::error('Unexpected error', [
        'error' => $e->getMessage(),
        'trace' => $e->getTraceAsString()
    ]);
    return response()->json(['error' => 'Something went wrong'], 500);
}
```

Exception handling Laravel application এর stability এবং user experience এর জন্য অত্যন্ত গুরুত্বপূর্ণ।