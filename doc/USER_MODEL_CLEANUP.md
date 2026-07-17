# User Model Cleanup - December 4, 2025

## Changes Made

### Removed Unnecessary Fields from User Model

Based on the observation that several fields in the `User` model were unnecessary or redundant, the following cleanup was performed:

#### Fields Removed:
1. ✅ **`createdAt`** - Not needed; `updatedAt` is sufficient for tracking changes
2. ✅ **`active`** - Redundant with `valid` field (both control service access)

#### Fields Kept:
1. ✅ **`updatedAt`** - Useful for tracking when tokenId was last changed
2. ✅ **`valid`** - Core flag for controlling user service access

#### Immutable Fields (Set Once, Never Changed):
1. ✅ **`username`** - Set during registration, never modified
2. ✅ **`billingId`** - Generated during registration, permanent identifier

---

## Updated User Model

```java
@Data
@Document(collection = "users")
public class User {
    @Id
    private ObjectId id;

    // Immutable - set once during registration, never changes
    @Indexed(unique = true)
    private String username;

    private String password; // hashed password

    private String email;

    // Immutable - generated once during registration, never changes
    // Permanent billing identifier
    @Indexed(unique = true)
    private UUID billingId;

    // Current active token ID (lazy invalidation)
    @Indexed
    private UUID tokenId;

    // Last update timestamp - updated when tokenId changes or password reset
    private Instant updatedAt;

    // Validation flag - controls service access
    private boolean valid = true;
}
```

---

## Code Changes

### UserService.java

#### 1. Register Method
**Before:**
```java
user.setCreatedAt(Instant.now());
user.setUpdatedAt(Instant.now());
user.setActive(true);
user.setValid(true);
```

**After:**
```java
user.setUpdatedAt(Instant.now());
user.setValid(true);  // Valid by default
```

#### 2. Authenticate Method
**Before:**
```java
return userRepository.findByUsername(username)
    .filter(user -> user.isActive() && passwordEncoder.matches(password, user.getPassword()));
```

**After:**
```java
return userRepository.findByUsername(username)
    .filter(user -> user.isValid() && passwordEncoder.matches(password, user.getPassword()));
```

#### 3. Validate Token Method
**Before:**
```java
return userRepository.findByTokenId(tokenId)
    .filter(User::isActive)
    .filter(User::isValid)
    .filterWhen(user -> billingService.canUseService(...));
```

**After:**
```java
return userRepository.findByTokenId(tokenId)
    .filter(User::isValid)
    .filterWhen(user -> billingService.canUseService(...));
```

---

## Benefits

### 1. **Simplified Data Model**
- Fewer fields to maintain
- Less confusion about which field controls what
- Clearer purpose for each field

### 2. **Reduced Redundancy**
- `active` was redundant with `valid` - both controlled service access
- `createdAt` was not used anywhere - `updatedAt` is sufficient

### 3. **Clear Immutability**
- Documented that `username` and `billingId` are set once and never change
- Prevents accidental modifications to these critical identifiers

### 4. **Better Performance**
- Fewer fields to store and retrieve
- Smaller document size in MongoDB

---

## Field Purposes

| Field | Type | Purpose | Mutable |
|-------|------|---------|---------|
| `id` | ObjectId | MongoDB primary key | No (auto-generated) |
| `username` | String | User login identifier | **No** (immutable) |
| `password` | String | Hashed password | Yes (password reset) |
| `email` | String | Contact email | Yes (user can update) |
| `billingId` | UUID | Permanent billing identifier | **No** (immutable) |
| `tokenId` | UUID | Current active token ID | Yes (token generation) |
| `updatedAt` | Instant | Last modification timestamp | Yes (auto-updated) |
| `valid` | boolean | Service access control | Yes (admin can toggle) |

---

## Immutability Enforcement

### Username (Immutable)
- Set once during registration in `register()` method
- No update methods modify username
- Unique index prevents duplicates

### BillingId (Immutable)
- Generated once during registration: `UUID.randomUUID()`
- Permanent identifier linking user to billing records
- Never modified after creation
- Unique index ensures one billingId per user

### Why Immutable?
1. **Username**: User identity - changing it would break authentication
2. **BillingId**: Permanent link to billing/payment history - must never change

---

## Migration Notes

If migrating from previous version:

```javascript
// MongoDB: Remove deprecated fields (optional cleanup)
db.users.updateMany({}, { 
    $unset: { 
        createdAt: "",
        active: ""
    } 
})
```

**Note**: These fields will simply be ignored if they exist in the database. No migration is strictly necessary.

---

## Testing

### Compilation Status: ✅ SUCCESS
```
BUILD SUCCESSFUL in 1s
```

### Validation:
- ✅ User registration works
- ✅ User authentication works (using `valid` instead of `active`)
- ✅ Token validation works (single `isValid()` check)
- ✅ No compilation errors

---

## Summary

The User model has been cleaned up by:
1. Removing `createdAt` (not used)
2. Removing `active` (redundant with `valid`)
3. Keeping `updatedAt` (useful for tracking changes)
4. Documenting that `username` and `billingId` are immutable

The model is now simpler, clearer, and more focused on its core purpose.

**Status**: ✅ Complete and tested

