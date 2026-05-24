### @RestControllerAdvice [[Annotations]][[Global Exception Handling]]

Globally intercept exceptions thrown from controllers.

```
Controller throws exception
    ↓
GlobalExceptionHandler intercepts
    ↓
Custom structured API response
```


#### Important Internal Flow

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


#### @ExceptionHandler

```
@ExceptionHandler(MethodArgumentNotValidException.class)
```
This is exception-specific routing.
Spring internally does:
- exception matching
- method resolution
- response serialization

