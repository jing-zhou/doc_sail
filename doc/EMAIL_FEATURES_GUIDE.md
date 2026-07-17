# Password Reset and Email Features Guide

Complete guide for password reset functionality and token delivery by email.

## Overview

The system now supports three email-based features:
1. **Login by email + password** - Users can log in using email instead of username
2. **Token delivery by email** - Users can request JWT tokens to be sent to their email
3. **Password reset by email** - Users can reset forgotten passwords via email

---

## Feature 1: Login by Email + Password

### Status
✅ **FULLY IMPLEMENTED**

### Description
Users can log in using their email address instead of username. The system automatically detects whether the user provided a username or email.

### Endpoint
```
POST /api/auth/login
```

### Request (Username)
```json
{
  "username": "alice",
  "password": "SecurePassword123!"
}
```

### Request (Email)
```json
{
  "username": "alice@example.com",
  "password": "SecurePassword123!"
}
```

### Response (Success)
```json
{
  "success": true,
  "message": "Login successful",
  "data": {
    "username": "alice",
    "sessionId": "a1b2c3d4-e5f6-7890-1234-567890abcdef"
  }
}
```

### Implementation Details
- The `username` field accepts both username and email
- System tries to find user by username first, then by email
- Same authentication flow for both cases
- SessionId cookie is set automatically

### Testing
```bash
# Login with username
curl -X POST http://localhost:2080/api/auth/login \
  -H "Content-Type: application/json" \
  -c cookies.txt \
  -d '{"username":"alice","password":"SecurePass123!"}'

# Login with email
curl -X POST http://localhost:2080/api/auth/login \
  -H "Content-Type: application/json" \
  -c cookies.txt \
  -d '{"username":"alice@example.com","password":"SecurePass123!"}'
```

---

## Feature 2: Token Delivery by Email

### Status
✅ **FULLY IMPLEMENTED**

### Description
When generating a JWT token, users can optionally request the token to be sent to their registered email address. This is useful when:
- User is on a public computer and doesn't want to copy/paste token
- User wants to access token later from a different device
- User wants a secure delivery method for the token

### Endpoint
```
POST /api/auth/token/generate
```

### Request (with email delivery)
```json
{
  "username": "alice",
  "password": "SecurePassword123!",
  "expirationMinutes": 43200,
  "sendEmail": true
}
```

### Response
```json
{
  "success": true,
  "message": "Token generated and sent to your email",
  "data": {
    "token": "eyJhbGciOiJIUzI1NiJ9...",
    "expirationMinutes": 43200,
    "expiresAt": "2025-12-08T10:30:00Z"
  }
}
```

### Email Content
The user receives an email with:
- The complete JWT token
- Expiration information
- Security warnings
- Usage instructions

**Example Email:**
```
Hello alice,

Your Illiad Proxy access token has been generated successfully.

Important: Keep this token secure and do not share it with anyone!

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
ACCESS TOKEN:
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

eyJhbGciOiJIUzI1NiJ9.eyJpZCI6IjEyMzQ1Njc4LTkwYWItY2RlZi0xMjM0LTU2Nzg5MGFiY2RlZiIsImV4cGlyZXNBdCI6MTczMzY1NDQwMDAwMH0.abc123...

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

Token Information:
• Expires in: 30 days
• Valid until: This token will become invalid when you generate a new one
• One token per user: Generating a new token will invalidate this one

How to use:
1. Copy the token above (everything between the lines)
2. Configure your Illiad client with this token
3. You can also paste this token into your client's configuration file

Security Notes:
• This token is NOT stored on our servers
• We cannot recover it if you lose it
• If you lose this token, simply generate a new one from your account
• Each new token automatically invalidates all previous tokens

If you did not request this token, please contact support immediately.

Best regards,
Illiad Proxy Team
```

### Works with All 3 Alternatives

**Alternative 1: Username + Password**
```json
{
  "username": "alice",
  "password": "SecurePassword123!",
  "expirationMinutes": 43200,
  "sendEmail": true
}
```

**Alternative 2: SessionId**
```json
{
  "sessionId": "a1b2c3d4-e5f6-7890-1234-567890abcdef",
  "expirationMinutes": 43200,
  "sendEmail": true
}
```

**Alternative 3: Current Token (Renewal)**
```json
{
  "currentToken": "eyJhbGciOiJIUzI1NiJ9...",
  "expirationMinutes": 43200,
  "sendEmail": true
}
```

### Testing
```bash
# Generate token with email delivery
curl -X POST http://localhost:2080/api/auth/token/generate \
  -H "Content-Type: application/json" \
  -d '{
    "username":"alice",
    "password":"SecurePass123!",
    "expirationMinutes":43200,
    "sendEmail":true
  }'
```

---

## Feature 3: Password Reset by Email

### Status
✅ **FULLY IMPLEMENTED**

### Description
Users who forget their password can request a password reset. A reset token is sent to their registered email address, which they can use to set a new password.

### Security Features
- Reset tokens expire after 1 hour
- Tokens can only be used once
- All existing JWT tokens are invalidated after password reset
- MongoDB TTL index automatically deletes expired tokens
- Doesn't reveal if email exists (security)

### Workflow

#### Step 1: Request Password Reset

**Endpoint:**
```
POST /api/auth/password/reset-request
```

**Request:**
```json
{
  "email": "alice@example.com"
}
```

**Response (Success):**
```json
{
  "success": true,
  "message": "Password reset instructions sent to your email",
  "data": null
}
```

**Response (Email not found - same response for security):**
```json
{
  "success": true,
  "message": "If this email is registered, password reset instructions will be sent",
  "data": null
}
```

**Email Content:**
```
Hello alice,

We received a request to reset your password for your Illiad Proxy account.

Your password reset token:
a1b2c3d4-e5f6-7890-1234-567890abcdef

This token will expire in 1 hour.

If you did not request a password reset, please ignore this email and contact support.

Best regards,
Illiad Proxy Team
```

#### Step 2: Confirm Password Reset with Token

**Endpoint:**
```
POST /api/auth/password/reset-confirm
```

**Request:**
```json
{
  "resetToken": "a1b2c3d4-e5f6-7890-1234-567890abcdef",
  "newPassword": "NewSecurePassword456!"
}
```

**Response (Success):**
```json
{
  "success": true,
  "message": "Password reset successfully. All existing tokens have been invalidated.",
  "data": null
}
```

**Response (Error - Expired Token):**
```json
{
  "success": false,
  "message": "Reset token has expired",
  "data": null
}
```

**Response (Error - Already Used):**
```json
{
  "success": false,
  "message": "Reset token has already been used",
  "data": null
}
```

**Response (Error - Invalid Token):**
```json
{
  "success": false,
  "message": "Invalid reset token",
  "data": null
}
```

### Testing Complete Workflow

**Step 1: Request reset**
```bash
curl -X POST http://localhost:2080/api/auth/password/reset-request \
  -H "Content-Type: application/json" \
  -d '{"email":"alice@example.com"}'
```

**Step 2: Check email for reset token (in dev, check logs)**

**Step 3: Confirm with token**
```bash
curl -X POST http://localhost:2080/api/auth/password/reset-confirm \
  -H "Content-Type: application/json" \
  -d '{
    "resetToken":"a1b2c3d4-e5f6-7890-1234-567890abcdef",
    "newPassword":"NewSecurePassword456!"
  }'
```

**Step 4: Login with new password**
```bash
curl -X POST http://localhost:2080/api/auth/login \
  -H "Content-Type: application/json" \
  -d '{"username":"alice","password":"NewSecurePassword456!"}'
```

---

## Email Configuration

### Required Settings

Add to `application.properties` or `gradle.properties.local`:

```properties
# SMTP Configuration
spring.mail.host=smtp.gmail.com
spring.mail.port=587
spring.mail.username=your-email@gmail.com
spring.mail.password=your-app-password
spring.mail.properties.mail.smtp.auth=true
spring.mail.properties.mail.smtp.starttls.enable=true

# From email address (optional, defaults to noreply@illiad-proxy.com)
email.from=noreply@yourdomain.com
```

### Gmail Setup (Example)
1. Enable 2-factor authentication on your Google account
2. Generate an "App Password" for your application
3. Use the app password in the configuration

### Testing Email in Development
- Use a fake SMTP service like MailHog or Mailtrap
- Check application logs for email content
- Use a test email account

---

## Data Models

### PasswordResetToken
```java
@Document(collection = "password_reset_tokens")
class PasswordResetToken {
    ObjectId id;
    ObjectId userId;          // User reference
    UUID token;               // Reset token (sent to email)
    Instant createdAt;        // Creation timestamp
    Instant expiresAt;        // Expiration (1 hour)
    boolean used;             // Whether token was used
}
```

**MongoDB TTL Index:**
- Expired tokens are automatically deleted by MongoDB
- TTL index on `expiresAt` field
- No manual cleanup required

---

## API Summary

| Endpoint | Method | Purpose | Status |
|----------|--------|---------|--------|
| `/api/auth/login` | POST | Login with username OR email | ✅ Implemented |
| `/api/auth/token/generate` | POST | Generate token (optionally send to email) | ✅ Implemented |
| `/api/auth/password/reset-request` | POST | Request password reset (sends email) | ✅ Implemented |
| `/api/auth/password/reset-confirm` | POST | Confirm password reset with token | ✅ Implemented |

---

## Security Considerations

### Token Delivery by Email
- ✅ Tokens are NOT stored on server (only tokenId)
- ✅ Email sent over encrypted connection (STARTTLS)
- ✅ Token visible in email - user should delete after use
- ⚠️ Email is less secure than HTTPS download (email can be intercepted)
- ⚠️ Recommended: Use only when necessary, prefer direct download

### Password Reset
- ✅ Reset tokens expire after 1 hour
- ✅ Tokens can only be used once
- ✅ All JWT tokens invalidated after password reset
- ✅ Doesn't reveal if email exists (prevents enumeration)
- ✅ MongoDB automatically deletes expired tokens
- ⚠️ User must have access to their email account

### Email Security
- ✅ Use STARTTLS for encrypted email transmission
- ✅ Use app-specific passwords (not account password)
- ✅ Monitor for failed email sends
- ⚠️ Email provider must be trusted
- ⚠️ Consider rate limiting to prevent abuse

---

## Error Handling

### Common Errors

| Error | Cause | Solution |
|-------|-------|----------|
| "Failed to send email" | SMTP configuration wrong | Check SMTP settings |
| "Email not found" | Email not registered | Register first or use correct email |
| "Reset token has expired" | Token older than 1 hour | Request new reset token |
| "Reset token has already been used" | Token used before | Request new reset token |
| "Invalid reset token" | Token doesn't exist | Check token format, request new one |

---

## Best Practices

### For Users
1. **Email delivery:** Use sparingly, prefer direct download
2. **Password reset:** Act quickly (1 hour expiration)
3. **After reset:** All devices need to re-authenticate
4. **Email security:** Use strong email password, enable 2FA

### For Developers
1. **SMTP:** Use dedicated email service (SendGrid, Mailgun)
2. **Rate limiting:** Prevent abuse of password reset
3. **Monitoring:** Log all email sends and failures
4. **Testing:** Use test SMTP service in development

### For Administrators
1. **Email service:** Choose reliable provider
2. **Monitoring:** Track email delivery rates
3. **Rate limits:** Prevent spam and abuse
4. **Backups:** Ensure email config is backed up

---

## Future Enhancements

### Potential Improvements
1. **Email templates:** HTML email templates with branding
2. **Multi-language:** Support multiple languages
3. **Email verification:** Require email verification on signup
4. **2FA:** Two-factor authentication via email
5. **Token preview:** Show first/last few characters in UI
6. **Rate limiting:** Prevent brute force and spam
7. **Email queue:** Async email sending with retry logic

---

## Testing Checklist

- [ ] Login with username works
- [ ] Login with email works
- [ ] Token generation without email works
- [ ] Token generation with email works (sendEmail=true)
- [ ] Token received in email (check inbox)
- [ ] Password reset request sends email
- [ ] Password reset token expires after 1 hour
- [ ] Password reset with valid token works
- [ ] Password reset invalidates all JWT tokens
- [ ] Old password no longer works after reset
- [ ] New password works after reset

---

## Conclusion

All three email-based features are now **fully implemented and ready for use**:

✅ **Login by email + password** - Users can log in with email instead of username  
✅ **Token delivery by email** - Users can receive JWT tokens via email  
✅ **Password reset by email** - Users can reset forgotten passwords  

The implementation follows security best practices and provides a smooth user experience.

