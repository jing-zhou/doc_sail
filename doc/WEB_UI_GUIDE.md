# HeartLink Web UI

Dating-website-style web interface for user authentication and token management.

## Pages

### 1. Landing Page (`/index.html`)

**Purpose:** User registration and login

**Features:**
- Attractive hero section with "Find Your Soulmate" theme
- Modal-based login form
- Modal-based signup form
- Responsive design for mobile and desktop
- Form validation with error messages
- Automatic redirect to dashboard after successful login

**Technologies:**
- Pure HTML/CSS/JavaScript (no frameworks)
- Gradient purple background
- Heart logo and dating-themed UI

### 2. Dashboard (`/dashboard.html`)

**Purpose:** Token generation and management after login

**Features:**
- User welcome message with username
- Token generation with configurable expiration (1 day to 6 months)
- Token display with copy-to-clipboard functionality
- Token download as `.txt` file
- Token expiration date display
- Warning about token invalidation
- Session validation on page load
- Logout functionality
- Informational cards about token usage

**Security:**
- Automatically redirects to landing page if not logged in
- Session verification via HTTP-only cookies
- HTTPS-only cookies (secure flag)

## API Integration

The web UI integrates with the following REST API endpoints:

- `POST /api/auth/register` - User registration
- `POST /api/auth/login` - User login (sets sessionId cookie)
- `GET /api/auth/session` - Session validation
- `POST /api/auth/logout` - Logout (clears session)
- `POST /api/auth/token/generate` - Generate JWT token

## File Structure

```
src/main/resources/static/
├── index.html              # Landing page
├── dashboard.html          # Dashboard page
├── css/
│   ├── landing.css        # Landing page styles
│   └── dashboard.css      # Dashboard page styles
└── js/
    ├── landing.js         # Landing page logic
    └── dashboard.js       # Dashboard page logic
```

## Usage Flow

### New User Registration

1. Visit `/index.html`
2. Click "Sign Up" button
3. Fill in username, email, and password
4. Click "Create Account"
5. Auto-redirected to login modal
6. Login with credentials
7. Redirected to `/dashboard.html`

### Existing User Login

1. Visit `/index.html`
2. Click "Login" button
3. Enter username/email and password
4. Click "Login"
5. Redirected to `/dashboard.html`

### Token Generation

1. Login to dashboard
2. Select token expiration duration
3. Click "Generate Token"
4. Token appears in textarea
5. Copy or download token
6. Use token in client apps as Illiad header

### Token Management

- **Copy:** Click copy button to copy token to clipboard
- **Download:** Click download button to save token as text file
- **Renew:** Click "Generate New Token" to create a new token (invalidates old tokens)
- **Logout:** Click logout to end session

## Design Choices

### Why Dating Website Theme?

1. **Disguise:** Makes the proxy server appear as a legitimate dating website
2. **User-Friendly:** Dating sites have well-established UX patterns
3. **Accessible:** Non-technical users feel comfortable
4. **Professional:** Clean, modern design builds trust

### Why Modals for Login/Signup?

1. **No Page Reload:** Smooth user experience
2. **Focus:** User attention on form
3. **Modern:** Common pattern in web apps
4. **Responsive:** Works well on mobile

### Why Plain JavaScript?

1. **Simplicity:** No build step required
2. **Performance:** Fast page load
3. **Compatibility:** Works in all modern browsers
4. **Maintainability:** Easy to modify

## Customization

### Change Color Scheme

Edit gradients in CSS files:
- Landing page: `linear-gradient(135deg, #667eea 0%, #764ba2 100%)`
- Primary color: `#ff6b6b` (pink/red)

### Change Branding

1. Update logo SVG in HTML files
2. Change site name from "HeartLink"
3. Modify taglines and feature descriptions

### Add More Pages

Create new HTML files in `static/` directory:
- About page
- Help/FAQ page
- Terms of Service
- Privacy Policy

## Browser Support

- Chrome/Edge: Latest 2 versions
- Firefox: Latest 2 versions
- Safari: Latest 2 versions
- Mobile: iOS Safari, Chrome Android

## Security Considerations

1. **HTTPS Required:** All cookies marked as `Secure`
2. **HTTP-Only Cookies:** SessionId not accessible via JavaScript
3. **SameSite Strict:** CSRF protection
4. **No Token Storage:** Tokens not stored server-side
5. **Password Validation:** Minimum 8 characters
6. **Input Sanitization:** All user input validated

## Future Enhancements

- [ ] Password strength indicator
- [ ] Email verification
- [ ] Two-factor authentication
- [ ] Token usage statistics
- [ ] Account settings page
- [ ] Dark mode toggle
- [ ] Multi-language support
- [ ] Profile pictures

