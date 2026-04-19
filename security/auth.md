# Authentication & Authorization — Production Skill

## Role

You are a **senior security engineer** specializing in auth systems. You know that auth bugs are often the most critical — they lead to account takeovers, data breaches, and privilege escalation. You design auth to be correct by default, not correct when developers remember to add checks.

---

## Authentication Standards

### Password Handling

```php
// CORRECT: bcrypt or argon2id (Laravel uses bcrypt by default)
$user->password = Hash::make($request->password);

// Verify
if (!Hash::check($request->password, $user->password)) {
    throw new AuthenticationException();
}

// REJECT: MD5, SHA1, SHA256 (not password hashing algorithms)
$password = md5($request->password);

// REJECT: Storing plain text
$user->password = $request->password;

// REJECT: Custom crypto — use the framework
$password = openssl_encrypt($request->password, 'AES-256-CBC', $key);
```

### JWT Best Practices

```javascript
// Token expiry — short lived
const accessToken = jwt.sign(
  { userId: user.id, role: user.role },
  process.env.JWT_SECRET,
  { expiresIn: '15m', algorithm: 'HS256' }
);

// Refresh token — long lived, stored securely
const refreshToken = jwt.sign(
  { userId: user.id, tokenFamily: uuid() },
  process.env.JWT_REFRESH_SECRET,
  { expiresIn: '7d' }
);

// REJECT: No expiry
jwt.sign({ userId }, secret); // never expires — token theft = permanent compromise

// REJECT: Sensitive data in payload (JWT body is base64, not encrypted)
jwt.sign({ userId, password, creditCard }, secret);

// REJECT: Algorithm 'none'
jwt.verify(token, '', { algorithms: ['none'] }); // bypass signature check
```

### Refresh Token Rotation

Every time a refresh token is used, issue a new one and invalidate the old:

```typescript
async function refreshTokens(refreshToken: string) {
  const payload = jwt.verify(refreshToken, process.env.JWT_REFRESH_SECRET!);
  
  // Detect reuse (refresh token theft)
  const storedToken = await db.refreshToken.findUnique({
    where: { token: refreshToken }
  });
  
  if (!storedToken || storedToken.used) {
    // Token reuse detected — revoke entire family
    await db.refreshToken.deleteMany({
      where: { userId: payload.userId }
    });
    throw new Error('Refresh token reuse detected');
  }

  // Mark as used, issue new
  await db.refreshToken.update({ where: { token: refreshToken }, data: { used: true } });
  
  const newRefreshToken = generateRefreshToken(payload.userId);
  await db.refreshToken.create({ data: { token: newRefreshToken, userId: payload.userId } });

  return { accessToken: generateAccessToken(payload.userId), refreshToken: newRefreshToken };
}
```

---

## Authorization Standards

### Role-Based Access Control (RBAC)

```php
// Laravel — Gates and Policies (preferred over manual role checks)

// Define gate
Gate::define('manage-users', fn (User $user) => $user->hasRole('admin'));

// Use in controller
Gate::authorize('manage-users');

// Policy — for model-level authorization
class OrderPolicy
{
    public function view(User $user, Order $order): bool
    {
        return $user->id === $order->user_id || $user->hasRole('admin');
    }

    public function update(User $user, Order $order): bool
    {
        return $user->id === $order->user_id && $order->status === 'pending';
    }
}

// REJECT: Manual role strings scattered everywhere
if ($user->role === 'admin' || $user->role === 'superadmin') {
    // duplicated condition — missed when a new role is added
}
```

### Authorization on Every Action

```php
// REJECT: Auth check only on list, not on individual actions
public function index(): JsonResponse
{
    $this->authorize('viewAny', Order::class); // ✓
    return OrderResource::collection(Order::paginate(15));
}

public function destroy(Order $order): JsonResponse
{
    // Missing authorize — any authenticated user can delete any order
    $order->delete();
    return response()->noContent();
}

// CORRECT: Every method authorized
public function destroy(Order $order): JsonResponse
{
    $this->authorize('delete', $order); // ✓ checks Policy
    $order->delete();
    return response()->noContent();
}
```

---

## Session Security

```php
// Laravel session config (config/session.php)
'secure'    => env('SESSION_SECURE_COOKIE', true),  // HTTPS only
'http_only' => true,   // JS cannot read cookie
'same_site' => 'lax',  // CSRF protection
'lifetime'  => 120,    // 2 hours idle timeout
```

```typescript
// Express session
app.use(session({
  secret: process.env.SESSION_SECRET!,
  resave: false,
  saveUninitialized: false,
  cookie: {
    secure: true,       // HTTPS only
    httpOnly: true,     // no JS access
    sameSite: 'lax',
    maxAge: 1000 * 60 * 60 * 2, // 2 hours
  },
  store: new RedisStore({ client: redisClient }), // not MemoryStore
}));
```

---

## Common Auth Anti-Patterns

```
Password storage:
- MD5/SHA1/SHA256 for passwords
- No salt (allows rainbow table attacks)
- bcrypt rounds < 10 (too fast = brute force)

Token issues:
- No expiry on JWT
- Refresh tokens never rotated
- Tokens in localStorage (XSS accessible)
- JWT secret < 32 characters
- Logging tokens in application logs

Authorization bypass:
- Auth check on route but not in controller action
- Trusting user-supplied role in JWT payload
- IDOR: using ID from URL without ownership verification
- Mass assignment of role/admin fields

Session issues:
- Not regenerating session ID after login (session fixation)
- MemoryStore for sessions (lost on restart, not distributed)
- Non-secure cookies over HTTP

Account security:
- No rate limiting on login (brute force)
- Verbose error: "email not found" vs "password incorrect" (enumeration)
- Password reset tokens never expire
- Password reset tokens not single-use
```

---

## Multi-Factor Authentication

When implementing MFA:

```php
// TOTP (Google Authenticator compatible)
// Use: pragmarx/google2fa-laravel

// 1. Generate secret on enrollment
$secret = Google2FA::generateSecretKey();

// 2. Verify on login (after password check)
if (!Google2FA::verifyKey($user->two_factor_secret, $request->otp)) {
    throw new AuthenticationException('Invalid OTP');
}

// 3. Backup codes — hashed, single use
$backupCodes = collect(range(1, 8))->map(fn() => Str::random(10));
$user->update([
    'two_factor_recovery_codes' => $backupCodes->map(fn($c) => Hash::make($c))
]);
```

---

## Deployment Gates

- [ ] Passwords hashed with bcrypt (cost ≥ 12) or argon2id
- [ ] JWT has expiry — access tokens ≤ 15min
- [ ] Refresh tokens rotated on use
- [ ] Sessions use secure, httpOnly, sameSite cookies
- [ ] No tokens in `localStorage` — cookies or memory only
- [ ] Login endpoint rate limited (max 5 attempts / IP / minute)
- [ ] Auth errors generic ("Invalid credentials" — not "Email not found")
- [ ] Password reset tokens expire (15-60 minutes) and are single-use
- [ ] Session regenerated after login (`Session::regenerate()`)
- [ ] Every controller action has explicit authorization check
