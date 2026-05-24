Annotation [[Annotations]]

GlobalExceptionHandler acts as ==Centralized Exception Translation Layer

```
Internal Exceptions
        ↓
Translated Into
        ↓
Stable Client API Responses
```

### @RestControllerAdvice

Globally intercept exceptions thrown from controllers.

```
Controller throws exception
    ↓
GlobalExceptionHandler intercepts
    ↓
Custom structured API response
```


### Important Internal Flow

```
HTTP Request
    ↓
DispatcherServlet
    ↓
Controller
    ↓
Validation / Service / Repository
    ↓
Exception occurs
    ↓
@RestControllerAdvice catches
    ↓
Custom ResponseEntity returned
```


## @ExceptionHandler

```
@ExceptionHandler(MethodArgumentNotValidException.class)
```
This is exception-specific routing.
Spring internally does:
- exception matching
- method resolution
- response serialization


# CONCEPT 3 — Validation Flow

This line connects to a much deeper mechanism:

```
MethodArgumentNotValidException
```

This exception is thrown automatically when:

```
@Valid
```

fails.

Example:

```
public ResponseEntity<?> register(
    @Valid @RequestBody RegisterRequest request
)
```

Flow:

```
JSON Request
    ↓
Jackson deserialization
    ↓
DTO binding
    ↓
Bean Validation
    ↓
Validation fails
    ↓
MethodArgumentNotValidException thrown
```

```e.getBindingResult()```  --> Validation Error Container
This contains:
- validation failures
- rejected fields
- rejected values
- messages

```getFieldErrors()```
Each `FieldError` contains:
- field name
- invalid value
- validation message
- validation code
Example : ``` email -> Invalid email format```


## ResponseEntity

```ResponseEntity<ApiError>```

This represents:
- HTTP status
- headers
- body
You control:
- status codes
- payload
- response metadata
Without `ResponseEntity`,  
Spring uses defaults.
