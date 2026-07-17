# Email Login Support - Implementation Summary

**Date**: December 4, 2025  
**Status**: ✅ **COMPLETED**

## Feature Overview

Users can now log in with either their **username** or **email address**, along with their password. Both username and email are enforced as unique across all users.

---

## Changes Made

### 1. User Model Updates

#### Added Unique Index to Email Field
```java
@Data
@Document(collection = "users")
public class User {
    // Username - unique, immutable
    @Indexed(unique = true)
    private String username;
    
    // Email - unique, can be used for login, immutable
    @Indexed(unique = true)
    private String email;
    
    // ...other fields
}
```

**Immutability**: Both `username` and `email` are set once during registration and never changed.

---

### 2. UserRepository Updates

#### Added Email Query Methods
```java
@Repository
public interface UserRepository extends ReactiveMongoRepository<User, ObjectId> {
    Mono<User> findByUsername(String username);
    Mono<Boolean> existsByUsername(String username);
    
    // NEW: Email query methods
    Mono<User> findByEmail(String email);
    Mono<Boolean> existsByEmail(String email);
    
    Mono<User> findByBillingId(UUID billingId);
    Mono<User> findByTokenId(UUID tokenId);
}
```

---

### 3. UserService Updates

#### Registration - Validates Both Username and Email Uniqueness
```java
public Mono<User> register(String username, String password, String email) {
    // Check both username and email uniqueness
    return userRepository.existsByUsername(username)
        .flatMap(usernameExists -> {
            if (usernameExists) {
                return Mono.error(new IllegalArgumentException("Username already exists"));
            }
            return userRepository.existsByEmail(email);
        })
        .flatMap(emailExists -> {
            if (emailExists) {
                return Mono.error(new IllegalArgumentException("Email already exists"));
            }
            
            User user = new User();
            user.setUsername(username);  // Immutable
            user.setEmail(email);        // Immutable
            user.setPassword(passwordEncoder.encode(password));
            user.setBillingId(UUID.randomUUID());
            user.setUpdatedAt(Instant.now());
            user.setValid(true);
            
            return userRepository.save(user)
                .flatMap(savedUser ->
                    billingService.createBillingInfo(savedUser.getId().toString(), "free")
                        .thenReturn(savedUser)
                );
        });
}
```

#### Authentication - Supports Username or Email Login
```java
public Mono<User> authenticate(String usernameOrEmail, String password) {
    // Try username first, then email if not found
    return userRepository.findByUsername(usernameOrEmail)
        .switchIfEmpty(userRepository.findByEmail(usernameOrEmail))
        .filter(user -> user.isValid() && passwordEncoder.matches(password, user.getPassword()));
}
```

---

## How It Works

### User Registration Flow

1. **User provides**: username, email, password
2. **System validates**: 
   - Username is unique (check `existsByUsername()`)
   - Email is unique (check `existsByEmail()`)
3. **If validation passes**:
   - Create user with hashed password
   - Set username and email (both immutable)
   - Generate permanent billingId
   - Create billing info
4. **If validation fails**:
   - Return error: "Username already exists" or "Email already exists"

### User Login Flow

1. **User provides**: username/email + password
2. **System tries**:
   - First: Find user by username
   - If not found: Find user by email
3. **If user found**:
   - Check if user is valid
   - Verify password matches
4. **If authentication succeeds**:
   - Return User object
5. **If authentication fails**:
   - Return empty (authentication failed)

---

## Examples

### Example 1: Register User
```java
// User registers with username "alice", email "alice@example.com"
userService.register("alice", "password123", "alice@example.com")
    .subscribe(user -> {
        System.out.println("User registered: " + user.getUsername());
    });
```

**Validation**:
- ✅ Username "alice" must be unique
- ✅ Email "alice@example.com" must be unique

### Example 2: Login with Username
```java
// User logs in with username
userService.authenticate("alice", "password123")
    .subscribe(user -> {
        System.out.println("Logged in: " + user.getUsername());
    });
```

### Example 3: Login with Email
```java
// User logs in with email instead of username
userService.authenticate("alice@example.com", "password123")
    .subscribe(user -> {
        System.out.println("Logged in: " + user.getUsername());
    });
```

### Example 4: Registration Validation Errors
```java
// Try to register with existing username
userService.register("alice", "newpassword", "bob@example.com")
    .subscribe(
        user -> {},
        error -> {
            // Error: "Username already exists"
        }
    );

// Try to register with existing email
userService.register("bob", "password", "alice@example.com")
    .subscribe(
        user -> {},
        error -> {
            // Error: "Email already exists"
        }
    );
```

---

## Uniqueness Enforcement

### Database Level
```java
// MongoDB unique indexes on both fields
@Indexed(unique = true)
private String username;

@Indexed(unique = true)
private String email;
```

**Benefits**:
- Database enforces uniqueness at storage level
- Prevents race conditions
- Fast duplicate checking

### Application Level
```java
// Check existence before creating user
existsByUsername(username)
existsByEmail(email)
```

**Benefits**:
- Clear error messages
- Early validation
- Better user experience

---

## Migration Notes

### For Existing Databases

If you have existing users without unique email indexes:

```javascript
// MongoDB: Add unique index to email field
db.users.createIndex({ email: 1 }, { unique: true })

// Check for duplicate emails before adding index
db.users.aggregate([
  { $group: { _id: "$email", count: { $sum: 1 } } },
  { $match: { count: { $gt: 1 } } }
])

// If duplicates found, clean them up first
```

---

## Security Considerations

### 1. **Immutability of Username and Email**
- Both fields are set once during registration
- Never modified after creation
- Prevents identity confusion

### 2. **Unique Constraints**
- Prevents duplicate accounts
- Enforced at database level
- Application validates before saving

### 3. **Email Privacy**
- Email is used for login but not exposed in tokens
- Only `tokenId` and `billingId` are used in JWT
- Email remains in database only

### 4. **Case Sensitivity**
**Note**: Current implementation is **case-sensitive**.

Examples:
- "Alice@example.com" ≠ "alice@example.com"
- User could register both (not recommended)

**Recommendation for Future**: Consider storing email in lowercase:
```java
user.setEmail(email.toLowerCase());
```

---

## Testing Scenarios

### ✅ Test Case 1: Register with Unique Credentials
- Username: "alice"
- Email: "alice@example.com"
- **Expected**: Success

### ✅ Test Case 2: Register with Duplicate Username
- Username: "alice" (already exists)
- Email: "bob@example.com" (unique)
- **Expected**: Error "Username already exists"

### ✅ Test Case 3: Register with Duplicate Email
- Username: "bob" (unique)
- Email: "alice@example.com" (already exists)
- **Expected**: Error "Email already exists"

### ✅ Test Case 4: Login with Username
- Input: "alice" + password
- **Expected**: Success (find by username)

### ✅ Test Case 5: Login with Email
- Input: "alice@example.com" + password
- **Expected**: Success (find by email)

### ✅ Test Case 6: Login with Invalid Credentials
- Input: "alice" + wrong password
- **Expected**: Empty (authentication failed)

---

## Benefits

### 1. **Improved User Experience**
- Users can use either username or email for login
- More flexible authentication
- Remembering email may be easier than username

### 2. **Better Security**
- Both username and email are unique
- Prevents account confusion
- Clear identity management

### 3. **Data Integrity**
- Unique constraints at database level
- Application-level validation
- Clear error messages

### 4. **Immutability**
- Username and email never change
- Stable user identity
- No confusion in billing or logs

---

## Files Modified

### Models
- ✅ `User.java` - Added unique index to email field

### Repositories
- ✅ `UserRepository.java` - Added `findByEmail()` and `existsByEmail()`

### Services
- ✅ `UserService.java` - Updated `register()` and `authenticate()` methods

### Documentation
- ✅ `EMAIL_LOGIN_SUPPORT.md` - This file

---

## Summary

Users can now:
1. ✅ Register with unique username and email
2. ✅ Login with either username or email
3. ✅ Receive clear error messages for duplicate credentials

System enforces:
1. ✅ Username uniqueness (database + application)
2. ✅ Email uniqueness (database + application)
3. ✅ Immutability of both fields

**Status**: Ready for use. Consider adding email normalization (lowercase) in future update.

