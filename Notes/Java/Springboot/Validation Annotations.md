## Validation Annotations [[Annotations]] 

> Every validation annotation has a message attribute

### @NotNull
value cannot be `null`
`@NotNull` does NOT care about content.
```
@NotNull
private String name;
```
### @NotBlank
Stronger validation for Strings.
Means:
- not null
- not empty
- not whitespace only
```
@NotBlank
private String username;
```
### Difference: `@NotNull` vs `@NotBlank`

|Value|`@NotNull`|`@NotBlank`|
|---|---|---|
|`null`|❌|❌|
|`""`|✅|❌|
|`" "`|✅|❌|
|`"abc"`|✅|✅|

### @Email
value must be an valid email address.
it doesn't check for blank,null and empty values.

```
@NotBlank(message = "Email is required")
@Email(message = "Invalid Email Format")
private String email;
```

### @Size(min = ,max= ,message= )
it Specifies the length of the datatype. it works for string, collections and arrays

### @Pattern( regex="",message="")
it checks with the pattern and validates it.




## Custom Annotation
```
@Target(ElementType.FIELD)
@Retention(RetentionPolicy.RUNTIME)
@Constraint(validatedBy = PasswordValidator.class)
public @interface ValidPassword {

    String message() default "Invalid password";

    Class<?>[] groups() default {};

    Class<? extends Payload>[] payload() default {};
}
```

This will create the custom annotation
```
public @interface ValidPassword
```
`@Target(ElementType.FIELD)` -- >This defines where annotation can be used
`@Retention(RetentionPolicy.RUNTIME)` --> Keep this annotation available while the app is running.

> Validation frameworks like Jakarta Validation inspect annotations at runtime using reflection.

`@Constraint(validatedBy = PasswordValidator.class)` 
When `@ValidPassword` is used,  
run logic from `PasswordValidator`.

`String message()` --> Default error message.

```
package org.devsync.spring.common.validators;  
  
import jakarta.validation.ConstraintValidator;  
import jakarta.validation.ConstraintValidatorContext;  
  
public class PasswordValidator implements ConstraintValidator<ValidPassword,String> {  
  
    @Override  
    public boolean isValid(String password,  
                           ConstraintValidatorContext context) {  
        if (password == null) {  
            return false;  
        }  
        return password.matches(  
                "^(?=.*[A-Z])(?=.*[a-z])(?=.*\\d)(?=.*[@#$%^&+=!]).{8,20}$"  
        );  
    }  
}
```

### @Documented
Marks a custom annotation so it appears in generated documentation (like Javadocs).
Only on annotations.
#### Common Usage
Usually added to:
- custom validation annotations
- framework annotations
- reusable annotations
#### Example
```
@Documented
@Target(ElementType.FIELD)
@Retention(RetentionPolicy.RUNTIME)
public @interface ValidPassword {
}
```
#### What It Affects
- documentation visibility
- reflection/documentation tools
#### Common Meta-Annotations Used With It

|Annotation|Purpose|
|---|---|
|`@Target`|where annotation can be used|
|`@Retention`|annotation lifecycle|
|`@Documented`|include in generated docs|
|`@Inherited`|inherited by subclasses|