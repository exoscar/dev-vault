
### 1. at the time of login even the email/username doesn't exist in the db. instead of directly throwing an error. perform a dummy password comparison. 
Sometimes attackers can measure timing differences.
Large systems occasionally perform a fake bcrypt comparison even when user doesn't exist.
like --> if email not found --> 1ms
correct email and invalid password --> 3ms
even though we are passing same error message -> "invalid username and password". by the time difference they might be knowing


Use the same error message "Invalid Username and password" for both password match and isEmailExists. --> Prevents Email enumeration attacks