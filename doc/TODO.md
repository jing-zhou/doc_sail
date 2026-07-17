# TODO List - Illiad Proxy Server

**Last Updated**: December 4, 2025

---

## High Priority

### 1. ⚠️ Self-Connection Loop Issue (CRITICAL - Design Decision Needed)

**Status**: Identified, Solution Pending  
**Priority**: HIGH  
**Discovered**: December 4, 2025

#### Problem Description

User with valid JWT token wants to generate a new token (legitimate use case for token renewal):
1. Client sends token generation request with Illiad header (valid JWT)
2. `HeaderDecoder` validates JWT → routes to SOCKS5 handler
3. SOCKS5 `CONNECT` request targets server itself (same IP:PORT)
4. Potential infinite loop or connection failure

#### Current Behavior
- SOCKS5 connection attempt to self will likely fail/timeout
- Client receives SOCKS5 FAILURE response
- Not ideal UX for legitimate token renewal scenario

#### Design Constraints
- Cannot modify `HeaderDecoder` (routing decision already made)
- Cannot reject SOCKS5 connection (legitimate use case)
- Must handle in SOCKS5 pipeline after routing decision

#### Possible Solutions to Evaluate

**Option A: Separate API Port (Recommended)**
- Port 443: Proxy traffic only (with Illiad header)
- Port 8080: Management API only (token generation, user management)
- Pros: Clean separation, no self-connection possible
- Cons: Requires client configuration changes, two ports to manage

**Option B: Smart Fallback in V5ConnectHandler**
- Detect self-connection in `V5ConnectHandler`
- Reconstruct original HTTP request
- Reroute to HTTP/HTTPS handler (RerouteHandler)
- Pros: Transparent to client, no configuration changes
- Cons: Complex implementation, "undoing" routing decision

**Option C: Direct Connection Requirement**
- Document that token renewal must NOT go through proxy
- Client configures: proxy for traffic, direct for API
- Pros: Simple, matches typical proxy usage
- Cons: User error prone, poor UX

**Option D: Detect at SOCKS5 Command Level**
- In `V5CommandHandler`, detect self-connection before `CONNECT`
- Switch pipeline to HTTP handlers
- Pros: Early detection, cleaner than Option B
- Cons: Still complex, modifies SOCKS5 flow

#### Recommended Approach
**Phase 1 (Immediate)**: Document proper usage - token renewal should use direct connection, not proxy  
**Phase 2 (Long-term)**: Implement separate API port (Option A) for clean architecture

#### Files Involved
- `src/main/java/com/illiad/server/handler/v5/V5ConnectHandler.java`
- `src/main/java/com/illiad/server/handler/v5/V5CommandHandler.java`
- `src/main/java/com/illiad/server/codec/v5/HeaderDecoder.java` (read-only, no changes)

#### Related Documentation
- `SELF_CONNECTION_LOOP_FIX.md` - Analysis of the issue
- `SELF_CONNECTION_LOOP_SUMMARY.md` - Summary

#### Next Steps
1. Decide on architectural approach (separate port vs smart fallback)
2. If separate port: Design REST API structure
3. If smart fallback: Design reroute mechanism in SOCKS5 handler
4. Update client documentation with recommended usage patterns
5. Implement chosen solution
6. Test token renewal scenarios thoroughly

---

## Medium Priority

### 2. Email Service Dependencies

**Status**: Implemented, Needs Setup  
**Priority**: MEDIUM

#### TODO
- [ ] Run `./gradlew build --refresh-dependencies` to download Spring Mail
- [ ] Configure SMTP settings in `application.properties`
- [ ] Test email sending with Gmail App Password (development)
- [ ] Switch to SendGrid/Amazon SES for production
- [ ] Test welcome emails on registration
- [ ] Test password reset emails
- [ ] Test JWT token delivery via email

#### Files
- `EmailService.java` - Already implemented
- `application.properties` - Needs SMTP configuration

---

### 3. MongoDB TTL Index Setup

**Status**: Documented, Not Created  
**Priority**: MEDIUM

#### TODO
- [ ] Create TTL index for password reset tokens:
```javascript
db.password_reset_tokens.createIndex(
  { "expiresAt": 1 }, 
  { expireAfterSeconds: 0 }
)
```
- [ ] Verify index is working (tokens auto-delete after expiration)
- [ ] Monitor for orphaned tokens

#### Files
- `PasswordResetToken.java` - Model ready
- MongoDB - Index needs manual creation

---

### 4. REST API Endpoints

**Status**: Service Layer Complete, API Layer Pending  
**Priority**: MEDIUM

#### TODO
Build REST API endpoints for:
- [ ] `POST /api/auth/register` - User registration
- [ ] `POST /api/auth/login` - User login
- [ ] `POST /api/tokens/generate` - Generate new JWT token
- [ ] `POST /api/tokens/refresh` - Refresh existing token (with current JWT)
- [ ] `POST /api/password-reset/request` - Request password reset
- [ ] `POST /api/password-reset/reset` - Reset password with token
- [ ] `POST /api/password/change` - Change password (authenticated)
- [ ] `GET /api/user/profile` - Get user profile
- [ ] `GET /api/user/billing` - Get billing info

#### Related
- Links to self-connection loop issue (token generation endpoint)
- May require separate port implementation

---

## Low Priority

### 5. Password Strength Validation

**Status**: Not Implemented  
**Priority**: LOW

#### TODO
- [ ] Add password strength requirements
- [ ] Minimum length validation (e.g., 12 characters)
- [ ] Complexity requirements (uppercase, lowercase, numbers, symbols)
- [ ] Common password blacklist
- [ ] Password strength meter for UI

---

### 6. Rate Limiting

**Status**: Not Implemented  
**Priority**: LOW

#### TODO
- [ ] Password reset requests: max 3 per hour per email
- [ ] Token generation: max 10 per hour per user
- [ ] Login attempts: max 5 failed attempts per hour per IP
- [ ] Email sending: max 10 per hour per user

---

### 7. Email Case Sensitivity

**Status**: Not Implemented  
**Priority**: LOW

#### Current Behavior
- Email is case-sensitive
- "Alice@example.com" ≠ "alice@example.com"
- User could register both

#### TODO
- [ ] Store email in lowercase: `user.setEmail(email.toLowerCase())`
- [ ] Search by lowercase email
- [ ] Migrate existing emails to lowercase

---

### 8. HTML Email Templates

**Status**: Plain Text Only  
**Priority**: LOW

#### TODO
- [ ] Design HTML email templates
- [ ] Add branding/logo
- [ ] Mobile-responsive design
- [ ] Use JavaMailSender with MimeMessage for HTML

---

## Future Enhancements

### 9. Two-Factor Authentication (2FA)

**Status**: Not Planned  
**Priority**: FUTURE

- TOTP-based 2FA for login
- SMS/Email backup codes
- Recovery codes

---

### 10. User Dashboard / Web UI

**Status**: Not Planned  
**Priority**: FUTURE

- User registration UI
- Login UI
- Token generation UI with copy button
- Password reset UI
- Billing/usage dashboard
- Account settings

---

### 11. Billing System Enhancements

**Status**: Basic Structure Exists  
**Priority**: FUTURE

- Payment integration (Stripe, PayPal)
- Subscription plans
- Usage tracking and quotas
- Invoice generation
- Payment history

---

## Completed ✅

### ✅ User Model Cleanup
- Removed `createdAt` and `active` fields
- Documented `username` and `billingId` as immutable
- Completed: December 4, 2025

### ✅ Email Login Support
- Users can login with username OR email
- Both fields enforced as unique
- Completed: December 4, 2025

### ✅ Email Token Delivery
- JWT tokens can be sent via email
- Welcome emails on registration
- Completed: December 4, 2025

### ✅ Password Reset via Email
- Forgot password flow with email tokens
- Change password for authenticated users
- Token-based reset with 1-hour expiration
- Completed: December 4, 2025

### ✅ TokenInfo Refactoring
- Removed TokenInfo model (N:1 relationship bug)
- Implemented lazy invalidation with User.tokenId
- One token per user enforcement
- Completed: December 4, 2025

---

## Notes

- **Self-connection loop issue** is the most critical item requiring architectural decision
- Email and password reset features are complete but need SMTP configuration
- REST API layer is the next major development task
- Most enhancements depend on resolving self-connection issue first (separate API port)

---

**Maintainer**: Review this list monthly and update priorities as needed.

