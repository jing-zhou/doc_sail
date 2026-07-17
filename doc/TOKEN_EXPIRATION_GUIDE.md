# Token Expiration - User Choice Guide

## 🎯 Key Principle: **You Decide**

When generating a JWT token, **you choose** when it expires. The system provides selectable options to match your needs.

---

## ⏰ Available Expiration Options

### Full List of Choices

| Duration | Minutes | Days | Best For | Security Level |
|----------|---------|------|----------|----------------|
| **1 Day** | 1,440 | 1 | Testing, temporary access | ⭐⭐⭐⭐⭐ Highest |
| **3 Days** | 4,320 | 3 | Short-term projects | ⭐⭐⭐⭐ Very High |
| **5 Days** | 7,200 | 5 | Work week access | ⭐⭐⭐⭐ Very High |
| **1 Week** | 10,080 | 7 | Weekly rotation | ⭐⭐⭐ High |
| **2 Weeks** | 20,160 | 14 | Bi-weekly rotation | ⭐⭐⭐ High |
| **1 Month** | 43,200 | 30 | **RECOMMENDED** - Monthly rotation | ⭐⭐ Good |
| **3 Months** | 129,600 | 90 | Quarterly rotation | ⭐ Moderate |
| **6 Months** | 259,200 | 180 | **MAXIMUM** - Semi-annual rotation | ⚠️ Lower |

---

## 🎨 How to Choose

### Consider Your Needs:

**Choose SHORTER (1 day - 1 week) if:**
- ✅ Maximum security is critical
- ✅ You don't mind regenerating frequently
- ✅ Testing or temporary access
- ✅ High-risk environment
- ✅ Compliance requirements

**Choose MEDIUM (2 weeks - 1 month) if:**
- ✅ Balance of security and convenience ← **RECOMMENDED**
- ✅ Regular but not too frequent rotation
- ✅ Normal usage patterns
- ✅ Good security practices

**Choose LONGER (3 months - 6 months) if:**
- ✅ Convenience over frequent rotation
- ✅ Stable, long-term access needed
- ✅ Low-risk environment
- ⚠️ Remember: Less frequent rotation = slightly lower security

---

## 💡 Recommendations

### For Most Users: **1 Month (43,200 minutes)**

**Why 1 month is recommended:**
- ✅ Good balance between security and convenience
- ✅ Not too frequent to be annoying
- ✅ Not too long to be risky
- ✅ Easy to remember (monthly calendar)
- ✅ Industry standard for many systems

### Security-Conscious Users: **1 Week (10,080 minutes)**

**For higher security:**
- ✅ More frequent rotation
- ✅ Limited window for compromised tokens
- ✅ Better audit trail
- ✅ Weekly routine (Mondays, Fridays, etc.)

### Convenience-Focused Users: **3 Months (129,600 minutes)**

**For less frequent maintenance:**
- ✅ Quarterly rotation
- ✅ Fewer regenerations needed
- ✅ Good for stable environments
- ⚠️ Remember to mark calendar!

---

## ⚖️ Security vs Convenience Trade-off

```
High Security ←───────────────────────────→ High Convenience
     ↑                                              ↑
  1 day     1 week    1 month   3 months      6 months
  (Daily)  (Weekly)  (Monthly) (Quarterly) (Semi-annual)
```

**The Trade-off:**
- **Shorter expiration** = More secure, more maintenance
- **Longer expiration** = Less secure, less maintenance
- **Sweet spot** = 1 month (recommended)

---

## 🔄 Regeneration Strategy

### Before Expiration

**Best Practice: Regenerate BEFORE expiration**

Example for 1-month token:
```
Day 1:  Generate token (expires Day 30)
Day 25: Regenerate new token (5 days before expiration)
        Old token still works until new one generated
Day 25: New token generated → Old token INVALID immediately
        Update all clients with new token
```

**Why regenerate early?**
- ✅ Avoid service interruption
- ✅ Buffer time if something goes wrong
- ✅ No rush or panic
- ✅ Smooth transition

### Regular Rotation (Even Before Expiration)

**Even better: Regular rotation schedule**

For 3-month token:
```
Option 1: Use full 3 months
  - Generate Jan 1 (expires Apr 1)
  - Regenerate Mar 25

Option 2: Rotate monthly anyway (recommended!)
  - Generate Jan 1 (expires Apr 1)
  - Regenerate Feb 1 (voluntary rotation)
  - Old token invalidated
  - New token expires May 1
```

**Benefits of voluntary rotation:**
- ✅ Higher security (monthly rotation)
- ✅ Flexibility (3-month buffer if needed)
- ✅ Best of both worlds

---

## 🎯 Use Case Examples

### Example 1: Developer Testing
```
Scenario: Testing new client application
Choice: 1 Day (1,440 minutes)
Reason: Temporary, will generate new one tomorrow anyway
```

### Example 2: Personal Use
```
Scenario: Daily proxy usage for browsing
Choice: 1 Month (43,200 minutes)
Reason: Monthly rotation routine, good security
```

### Example 3: Business Environment
```
Scenario: Company proxy access for team
Choice: 1 Week (10,080 minutes)
Reason: Weekly security audits, higher security standard
```

### Example 4: Long-term Project
```
Scenario: 6-month research project
Choice: 6 Months (259,200 minutes)
Reason: Stable access, low regeneration overhead
Note: Consider rotating monthly anyway for security
```

### Example 5: Vacation/Travel
```
Scenario: 2-week trip, need stable access
Choice: 2 Weeks (20,160 minutes)
Reason: Exactly matches trip duration
```

---

## ⚠️ Important Notes

### Cannot Extend Expiration

Once you choose and generate a token:
- ❌ **Cannot extend** the expiration date
- ❌ **Cannot modify** the existing token
- ✅ **Must generate** new token if needed sooner
- ✅ **Can generate** new token anytime (invalidates old)

### Expiration is Enforced

The system **strictly enforces** the expiration:
- ⏰ Token stops working **exactly** at `expiresAt` timestamp
- ⏰ No grace period
- ⏰ No warnings (though you can set your own reminders)
- ⏰ Must generate new token to continue

### You Control Rotation

Even with long expiration:
- ✅ You can generate new token **anytime**
- ✅ Old token invalidated **immediately**
- ✅ Not locked to chosen duration
- ✅ Voluntary rotation encouraged

---

## 📱 Web Interface

When you visit `https://your-domain.com/`:

1. Click "Generate Token" tab
2. Enter your User ID
3. **Select expiration** from dropdown:
   - 1 Day
   - 3 Days
   - 5 Days
   - 1 Week
   - 2 Weeks
   - 1 Month (Recommended) ← Default
   - 3 Months
   - 6 Months (Maximum)
4. Click "Generate JWT Token"
5. Token displayed with your chosen expiration

**Visual Feedback:**
- 💡 Helper text: "You decide when your token expires"
- ✅ Clear labeling of recommended option (1 Month)
- ⏰ Maximum option clearly marked (6 Months)

---

## 🔢 Technical Details

### Implementation

**RerouteHandler.java:**
```java
// Validate expiration time (max 6 months = ~259200 minutes)
// User decides the expiration period from selectable options
long expirationMinutes = request.getExpirationMinutes();
if (expirationMinutes <= 0 || expirationMinutes > 259200) {
    sendJsonResponse(ctx, req, BAD_REQUEST,
        ApiResponse.error("Invalid expiration time. Must be between 1 minute and 6 months"));
    return;
}
```

**HTML Select Options:**
```html
<select id="token-expiration">
    <option value="1440">1 Day</option>
    <option value="4320">3 Days</option>
    <option value="7200">5 Days</option>
    <option value="10080">1 Week</option>
    <option value="20160">2 Weeks</option>
    <option value="43200" selected>1 Month (Recommended)</option>
    <option value="129600">3 Months</option>
    <option value="259200">6 Months (Maximum)</option>
</select>
```

### API Request Example

```bash
curl -X POST "https://your-domain.com/api/auth/token/generate?userId=USER_ID" \
  -H "Content-Type: application/json" \
  -d '{
    "expirationMinutes": 43200
  }'
```

**User chooses:** `43200` (1 month) or any other valid value

---

## 📊 Summary

### Key Points:
✅ **User decides** expiration duration  
✅ **8 options** available (1 day to 6 months)  
✅ **1 month recommended** for balance  
✅ **Cannot extend** once chosen  
✅ **Enforced** strictly by system  
✅ **Can regenerate** anytime before expiration  
✅ **Voluntary rotation** encouraged for security  

### User Empowerment:
- 🎯 You choose what works for you
- 🎯 Flexibility to match your needs
- 🎯 Control over your security/convenience balance
- 🎯 Can always generate new token earlier

**Bottom Line:** The system provides options, **you make the choice** that best fits your situation!

