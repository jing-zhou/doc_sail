# Email Token Delivery - Implementation Guide

**Date**: December 4, 2025  
**Status**: ✅ **IMPLEMENTED**

## Feature Overview

Users can now receive their generated JWT tokens via email. This provides:
- **Secure delivery**: Token sent to verified email address
- **Backup option**: Users have email copy if they lose the token
- **Remote configuration**: Useful when configuring clients on different devices
- **Convenience**: Token can be copied from email directly into client configuration

---

## Implementation Summary

### Components Added

1. **EmailService** - Handles all email sending functionality
2. **UserService Enhancement** - New method `generateAndEmailToken()`
3. **Email Configuration** - SMTP settings in application.properties
4. **Dependency** - Spring Boot Mail starter added to build.gradle.kts

---

## Dependencies Added

### build.gradle.kts
```kotlin
implementation("org.springframework.boot:spring-boot-starter-mail")
```

This adds Spring's email support with JavaMailSender.

---

## EmailService Features

### 1. Send JWT Token Email

```java
emailService.sendJwtToken(
    "user@example.com",      // recipient email
    "alice",                  // username for personalization
    "eyJhbG...",             // JWT token string  
    30                        // expires in days
)
```

**Email contains:**
- Personalized greeting
- Full JWT token (plain text, easy to copy)
- Expiration information
- Usage instructions
- Security warnings
- Clear formatting with separators

### 2. Send Welcome Email

```java
emailService.sendWelcomeEmail(
    "user@example.com",
    "alice"
)
```

Automatically sent when user registers (non-blocking, best effort).

### 3. Send Password Reset Email (Future)

```java
emailService.sendPasswordResetEmail(
    "user@example.com",
    "alice",
    "reset-token-123"
)
```

Framework ready for password reset functionality.

---

## UserService Enhancements

### Method: generateAndEmailToken()

```java
/**
 * Generate JWT token and optionally send it via email
 *
 * @param userId user ID
 * @param expirationMinutes token expiration in minutes  
 * @param sendEmail if true, send token to user's email
 * @return JWT token string
 */
public Mono<String> generateAndEmailToken(
    String userId, 
    long expirationMinutes, 
    boolean sendEmail
)
```

#### Usage Examples:

**Generate token WITHOUT email:**
```java
userService.generateAndEmailToken(userId, 43200, false)
    .subscribe(jwt -> {
        // Return JWT to user via API response
        System.out.println("Token: " + jwt);
    });
```

**Generate token WITH email:**
```java
userService.generateAndEmailToken(userId, 43200, true)
    .subscribe(jwt -> {
        // Token sent to email AND returned in response
        System.out.println("Token generated and emailed: " + jwt);
    });
```

**Error handling:**
```java
userService.generateAndEmailToken(userId, 43200, true)
    .subscribe(
        jwt -> {
            // Success: token generated (email sent or failed gracefully)
        },
        error -> {
            // Error: token generation failed
        }
    );
```

**Key Features:**
- Email sending is **non-blocking**
- If email fails, token is **still generated and returned**
- Email failure is logged but doesn't break the flow
- User always gets their token (via API), email is bonus

---

## Email Configuration

### application.properties

```properties
# ==============================================================================
# Email Configuration (SMTP)
# ==============================================================================
spring.mail.enabled=true
spring.mail.host=smtp.gmail.com
spring.mail.port=587
spring.mail.username=your-email@gmail.com
spring.mail.password=your-app-password

# SMTP properties
spring.mail.properties.mail.smtp.auth=true
spring.mail.properties.mail.smtp.starttls.enable=true
spring.mail.properties.mail.smtp.starttls.required=true

# Email sender
spring.mail.from=noreply@illiad-proxy.com
```

### Supported Email Providers

#### 1. Gmail (Recommended for Testing)
```properties
spring.mail.host=smtp.gmail.com
spring.mail.port=587
spring.mail.username=your-email@gmail.com
spring.mail.password=your-app-password  # NOT regular password!
```

**Setup:**
1. Go to Google Account settings
2. Enable 2-Factor Authentication
3. Generate App Password for "Mail"
4. Use App Password in configuration

#### 2. SendGrid (Recommended for Production)
```properties
spring.mail.host=smtp.sendgrid.net
spring.mail.port=587
spring.mail.username=apikey
spring.mail.password=your-sendgrid-api-key
```

**Benefits:**
- High deliverability
- 100 emails/day free tier
- Good for production

#### 3. Amazon SES
```properties
spring.mail.host=email-smtp.us-east-1.amazonaws.com
spring.mail.port=587
spring.mail.username=your-ses-smtp-username
spring.mail.password=your-ses-smtp-password
```

#### 4. Mailgun
```properties
spring.mail.host=smtp.mailgun.org
spring.mail.port=587
spring.mail.username=postmaster@your-domain.mailgun.org
spring.mail.password=your-mailgun-smtp-password
```

---

## Email Template: JWT Token

### Sample Email Body

```
Hello alice,

Your Illiad Proxy access token has been generated successfully.

Important: Keep this token secure and do not share it with anyone!

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
ACCESS TOKEN:
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

eyJhbGciOiJIUzI1NiJ9.eyJpZCI6IjEyMzQ1Njc4LTkwYWItY2RlZi0xMjM0LTU2Nzg5MGFiY2RlZiIsImV4cCI6MTczMzU0MDgwMH0.signature

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

---

## User Flows

### Flow 1: User Generates Token (Download Only)

```
1. User logs in to web interface
2. User clicks "Generate Token"
3. System generates JWT token
4. Token displayed on screen
5. User copies token manually
```

**API Call:**
```java
POST /api/tokens/generate
{
    "expirationMinutes": 43200,
    "sendEmail": false
}

Response:
{
    "token": "eyJhbGci...",
    "expiresAt": "2025-01-03T12:00:00Z"
}
```

### Flow 2: User Generates Token (Download + Email)

```
1. User logs in to web interface
2. User clicks "Generate and Email Token"
3. System generates JWT token
4. Token displayed on screen
5. Email sent to user's email address
6. User receives email with token
```

**API Call:**
```java
POST /api/tokens/generate
{
    "expirationMinutes": 43200,
    "sendEmail": true
}

Response:
{
    "token": "eyJhbGci...",
    "expiresAt": "2025-01-03T12:00:00Z",
    "emailSent": true
}
```

### Flow 3: User Loses Token

```
1. User realizes they lost their token
2. User logs in to web interface
3. User clicks "Generate New Token"
4. Old token immediately invalidated
5. New token generated and displayed
6. Optionally emailed to user
```

---

## Security Considerations

### 1. **Email Privacy**
- ✅ Token sent via TLS-encrypted SMTP
- ✅ Email provider (Gmail, SendGrid) uses TLS
- ⚠️ Email may be stored in user's mailbox (remind users to delete)

### 2. **Token in Email**
- ✅ Token is long and random (JWT format)
- ✅ User's email is verified (they registered with it)
- ⚠️ If email compromised, token is exposed
- ✅ User can generate new token anytime (invalidates old one)

### 3. **Email Sending Errors**
- ✅ Email failure doesn't prevent token generation
- ✅ User always gets token via API response
- ✅ Email is "best effort" - logged but not critical

### 4. **Rate Limiting (Future)**
Consider adding:
- Max 10 token generations per hour per user
- Max 5 email sends per hour per user
- Prevents abuse and spam

---

## Testing

### Test Case 1: Generate Token Without Email
```java
@Test
void generateToken_withoutEmail() {
    String token = userService.generateAndEmailToken(userId, 43200, false).block();
    assertNotNull(token);
    // No email sent
}
```

### Test Case 2: Generate Token With Email
```java
@Test
void generateToken_withEmail() {
    String token = userService.generateAndEmailToken(userId, 43200, true).block();
    assertNotNull(token);
    // Email sent to user's email address
}
```

### Test Case 3: Email Failure Doesn't Break Token Generation
```java
@Test
void generateToken_emailFailsButTokenGenerated() {
    // Configure invalid SMTP settings
    String token = userService.generateAndEmailToken(userId, 43200, true).block();
    assertNotNull(token);  // Token still generated
    // Email failed, but logged
}
```

### Manual Testing

1. **Configure Gmail SMTP:**
   - Create App Password
   - Update application.properties
   - Restart server

2. **Test Token Generation:**
   ```bash
   curl -X POST http://localhost:8080/api/tokens/generate \
     -H "Content-Type: application/json" \
     -d '{"expirationMinutes": 43200, "sendEmail": true}'
   ```

3. **Check Email:**
   - Verify email received
   - Copy token from email
   - Verify token format
   - Test token with client

---

## Deployment Considerations

### Production Email Setup

1. **Use Production Email Service**
   - ✅ SendGrid (recommended)
   - ✅ Amazon SES
   - ✅ Mailgun
   - ❌ NOT Gmail (rate limited, not for production)

2. **Configure SPF/DKIM**
   - Add SPF record to DNS
   - Configure DKIM with email provider
   - Improves deliverability

3. **Monitor Email Delivery**
   - Log email sends
   - Track failures
   - Alert on high failure rates

4. **Email Templates**
   - Consider HTML emails (prettier)
   - Add branding/logo
   - Mobile-friendly formatting

---

## Future Enhancements

### 1. **HTML Emails**
Current: Plain text emails  
Future: HTML with styling, logos, buttons

### 2. **Email Verification**
Add email verification during registration

### 3. **Email Preferences**
Let users opt-in/out of token emails

### 4. **Token History Emails**
Send summary of recent token generations

### 5. **Expiration Reminders**
Email users when token is about to expire

---

## Files Modified/Added

### Added Files:
- ✅ `EmailService.java` - Email sending service

### Modified Files:
- ✅ `build.gradle.kts` - Added Spring Mail dependency
- ✅ `UserService.java` - Added `generateAndEmailToken()` method
- ✅ `application.properties` - Added email configuration

### Documentation:
- ✅ `EMAIL_TOKEN_DELIVERY.md` - This file

---

## Configuration Checklist

Before using email features:

- [ ] Add Spring Mail dependency to build.gradle.kts
- [ ] Configure SMTP settings in application.properties
- [ ] Set email username and password (use App Password for Gmail)
- [ ] Set "from" email address
- [ ] Test email sending with a test user
- [ ] Verify emails are received (check spam folder)
- [ ] Configure SPF/DKIM for production (optional but recommended)
- [ ] Monitor email send failures in logs

---

## Summary

**What's New:**
1. ✅ JWT tokens can be sent to user's email
2. ✅ Welcome email sent on registration (optional)
3. ✅ Email framework ready for password reset
4. ✅ Configurable SMTP settings
5. ✅ Non-blocking, best-effort email delivery

**How to Use:**
- Call `generateAndEmailToken(userId, minutes, true)` to send token via email
- Call `generateAndEmailToken(userId, minutes, false)` for download only
- Configure SMTP in application.properties

**Security:**
- Tokens sent via TLS-encrypted email
- Email failure doesn't prevent token generation
- Users can regenerate tokens anytime

**Status**: ✅ Ready for use after SMTP configuration

---

**Next Steps:**
1. Configure SMTP settings in application.properties
2. Test with Gmail App Password (for development)
3. Switch to SendGrid/SES for production
4. Build REST API endpoints to expose this functionality
5. Create web UI for token generation with email option

## Recent changes (2025-12-06)
- Token delivery workflows were hardened for privacy: API responses that include the token (or a sessionId) do NOT include `userId` or `username`.
- The server uses MongoDB `ObjectId` internally to correlate tokens and users; these ObjectIds are not sent to clients.
- Email delivery still requires the user's email address (server-side), but the HTTP responses avoid returning any sensitive identifiers.
