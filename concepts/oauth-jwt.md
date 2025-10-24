# OAuth2 & JWT Deep Dive

## Overview

Understanding OAuth2 and JWT is essential for implementing secure authentication in modern Android apps.

## OAuth2 Fundamentals

### What is OAuth2?

OAuth2 is an **authorization framework** that enables apps to obtain limited access to user accounts on an HTTP service.

### Authorization vs Authentication

```
Authentication: "Who are you?"
    → Verifying identity
    → Username + Password
    → Biometric
    
Authorization: "What can you do?"
    → Granting access
    → Permissions
    → Scopes
```

## OAuth2 Flow for Mobile

### Authorization Code Flow with PKCE

**PKCE** (Proof Key for Code Exchange) - Recommended for mobile apps

```
1. App generates code_verifier and code_challenge
        ↓
2. App opens browser with authorization request
   URL: /authorize?
        client_id=APP_ID
        &redirect_uri=myapp://callback
        &response_type=code
        &code_challenge=CHALLENGE
        &code_challenge_method=S256
        ↓
3. User logs in via browser
        ↓
4. Server redirects: myapp://callback?code=AUTH_CODE
        ↓
5. App receives auth code via deep link
        ↓
6. App exchanges code for tokens
   POST /token
        code=AUTH_CODE
        &code_verifier=VERIFIER
        &grant_type=authorization_code
        ↓
7. Server responds with tokens
   {
        "access_token": "...",
        "refresh_token": "...",
        "expires_in": 3600
   }
```

### Implementation

```kotlin
class OAuth2Manager(private val context: Context) {
    
    private val clientId = "your_client_id"
    private val redirectUri = "myapp://callback"
    private val authEndpoint = "https://auth.example.com/authorize"
    private val tokenEndpoint = "https://auth.example.com/token"
    
    // Step 1: Generate PKCE values
    fun generateCodeVerifier(): String {
        val bytes = ByteArray(32)
        SecureRandom().nextBytes(bytes)
        return Base64.encodeToString(
            bytes,
            Base64.URL_SAFE or Base64.NO_WRAP or Base64.NO_PADDING
        )
    }
    
    fun generateCodeChallenge(verifier: String): String {
        val bytes = verifier.toByteArray()
        val digest = MessageDigest.getInstance("SHA-256").digest(bytes)
        return Base64.encodeToString(
            digest,
            Base64.URL_SAFE or Base64.NO_WRAP or Base64.NO_PADDING
        )
    }
    
    // Step 2: Build authorization URL
    fun getAuthorizationUrl(codeChallenge: String): String {
        return Uri.parse(authEndpoint)
            .buildUpon()
            .appendQueryParameter("client_id", clientId)
            .appendQueryParameter("redirect_uri", redirectUri)
            .appendQueryParameter("response_type", "code")
            .appendQueryParameter("code_challenge", codeChallenge)
            .appendQueryParameter("code_challenge_method", "S256")
            .appendQueryParameter("scope", "read write")
            .build()
            .toString()
    }
    
    // Step 3: Launch browser
    fun startAuthorization(codeChallenge: String) {
        val intent = Intent(Intent.ACTION_VIEW, Uri.parse(getAuthorizationUrl(codeChallenge)))
        context.startActivity(intent)
    }
    
    // Step 4: Handle callback (in Activity)
    fun handleCallback(uri: Uri): String? {
        return uri.getQueryParameter("code")
    }
    
    // Step 5: Exchange code for tokens
    suspend fun exchangeCodeForTokens(
        code: String,
        codeVerifier: String
    ): TokenResponse {
        val request = FormBody.Builder()
            .add("grant_type", "authorization_code")
            .add("code", code)
            .add("redirect_uri", redirectUri)
            .add("client_id", clientId)
            .add("code_verifier", codeVerifier)
            .build()
        
        val httpRequest = Request.Builder()
            .url(tokenEndpoint)
            .post(request)
            .build()
        
        val response = okHttpClient.newCall(httpRequest).execute()
        return Json.decodeFromString(response.body!!.string())
    }
}

data class TokenResponse(
    @Json(name = "access_token") val accessToken: String,
    @Json(name = "refresh_token") val refreshToken: String,
    @Json(name = "expires_in") val expiresIn: Int,
    @Json(name = "token_type") val tokenType: String = "Bearer"
)
```

### Handle Deep Link

In Activity:
```kotlin
override fun onCreate(savedInstanceState: Bundle?) {
    super.onCreate(savedInstanceState)
    
    intent.data?.let { uri ->
        if (uri.scheme == "myapp" && uri.host == "callback") {
            val code = uri.getQueryParameter("code")
            // Exchange code for tokens
        }
    }
}
```

In `AndroidManifest.xml`:
```xml
<activity android:name=".MainActivity">
    <intent-filter>
        <action android:name="android.intent.action.VIEW" />
        <category android:name="android.intent.category.DEFAULT" />
        <category android:name="android.intent.category.BROWSABLE" />
        <data
            android:scheme="myapp"
            android:host="callback" />
    </intent-filter>
</activity>
```

## JWT (JSON Web Token)

### JWT Structure

```
JWT = Header.Payload.Signature

Header (Base64):
{
    "alg": "HS256",      // Algorithm
    "typ": "JWT"         // Type
}

Payload (Base64):
{
    "sub": "user123",    // Subject (user ID)
    "name": "John Doe",  // Claims
    "iat": 1516239022,   // Issued at
    "exp": 1516242622    // Expiration
}

Signature:
HMACSHA256(
    base64UrlEncode(header) + "." + base64UrlEncode(payload),
    secret
)
```

### Decode JWT (Client Side)

```kotlin
data class JwtPayload(
    val sub: String,     // Subject
    val exp: Long,       // Expiration
    val iat: Long,       // Issued at
    val name: String?    // Custom claims
)

fun decodeJwt(token: String): JwtPayload? {
    return try {
        val parts = token.split(".")
        if (parts.size != 3) return null
        
        val payload = String(
            Base64.decode(parts[1], Base64.URL_SAFE),
            Charsets.UTF_8
        )
        
        Json.decodeFromString<JwtPayload>(payload)
    } catch (e: Exception) {
        null
    }
}

fun isTokenExpired(token: String): Boolean {
    val payload = decodeJwt(token) ?: return true
    val expirationMs = payload.exp * 1000
    return System.currentTimeMillis() > expirationMs
}
```

### Validate JWT

**Important**: Never validate JWT signature on client side! Always validate on server.

Client should only:
- ✅ Check expiration
- ✅ Read claims (user info)
- ❌ Validate signature (server's job)

## Token Refresh Flow

### Refresh Token Strategy

```
Access Token Expires (short-lived: 15min - 1hr)
    ↓
Use Refresh Token to get new Access Token
    ↓
Refresh Token rotated (or reused)
    ↓
Continue using new Access Token
```

### Implementation

```kotlin
class TokenManager(
    private val securePreferences: SecurePreferences,
    private val apiService: AuthApiService
) {
    
    suspend fun getValidAccessToken(): String? {
        val accessToken = securePreferences.getAccessToken() ?: return null
        
        return if (isTokenExpired(accessToken)) {
            refreshAccessToken()
        } else {
            accessToken
        }
    }
    
    private suspend fun refreshAccessToken(): String? {
        val refreshToken = securePreferences.getRefreshToken() ?: return null
        
        return try {
            val response = apiService.refreshToken(
                RefreshTokenRequest(
                    refreshToken = refreshToken,
                    grantType = "refresh_token"
                )
            )
            
            // Save new tokens
            securePreferences.saveTokens(
                accessToken = response.accessToken,
                refreshToken = response.refreshToken ?: refreshToken,
                expiresIn = response.expiresIn
            )
            
            response.accessToken
        } catch (e: Exception) {
            // Refresh failed - user must login again
            securePreferences.clearTokens()
            null
        }
    }
}
```

### Automatic Refresh Interceptor

```kotlin
class TokenAuthenticator(
    private val tokenManager: TokenManager
) : Authenticator {
    
    override fun authenticate(route: Route?, response: Response): Request? {
        // Don't retry if already tried
        if (response.request.header("Authorization") != null &&
            response.priorResponse?.code == 401
        ) {
            return null
        }
        
        // Try to refresh token
        val newToken = runBlocking {
            tokenManager.refreshAccessToken()
        }
        
        return newToken?.let {
            response.request.newBuilder()
                .header("Authorization", "Bearer $it")
                .build()
        }
    }
}

val client = OkHttpClient.Builder()
    .authenticator(TokenAuthenticator(tokenManager))
    .build()
```

## Secure Token Storage

### Never Store Tokens Insecurely

```kotlin
// ❌ Bad - Plain SharedPreferences
sharedPrefs.edit {
    putString("access_token", token)  // Readable by anyone!
}

// ✅ Good - Encrypted storage
val masterKey = MasterKey.Builder(context)
    .setKeyScheme(MasterKey.KeyScheme.AES256_GCM)
    .build()

val encryptedPrefs = EncryptedSharedPreferences.create(
    context,
    "secure_prefs",
    masterKey,
    EncryptedSharedPreferences.PrefKeyEncryptionScheme.AES256_SIV,
    EncryptedSharedPreferences.PrefValueEncryptionScheme.AES256_GCM
)

encryptedPrefs.edit {
    putString("access_token", token)  // Encrypted
}
```

## OAuth2 Scopes

### Understanding Scopes

```
Scopes = Permissions requested from user

Examples:
    "read"              → Read user data
    "write"             → Modify user data
    "profile"           → Access profile info
    "email"             → Access email
    "openid"            → OpenID Connect
```

### Request Scopes

```kotlin
val authUrl = Uri.parse(authEndpoint)
    .buildUpon()
    .appendQueryParameter("scope", "read write profile email")
    .build()
```

### Check Token Scopes

```kotlin
data class JwtPayload(
    val sub: String,
    val exp: Long,
    val scope: String  // "read write profile"
)

fun hasScope(token: String, requiredScope: String): Boolean {
    val payload = decodeJwt(token) ?: return false
    val scopes = payload.scope.split(" ")
    return requiredScope in scopes
}
```

## Best Practices

### 1. Use PKCE

```kotlin
// ✅ Always use PKCE for mobile apps
val codeVerifier = generateCodeVerifier()
val codeChallenge = generateCodeChallenge(codeVerifier)

// Prevents authorization code interception attacks
```

### 2. Secure Token Storage

```kotlin
// ✅ Use EncryptedSharedPreferences
// ✅ Never log tokens
// ✅ Clear tokens on logout
// ✅ Use HTTPS only
```

### 3. Token Expiration

```kotlin
// ✅ Check expiration before use
// ✅ Implement automatic refresh
// ✅ Handle refresh failures (re-login)
```

### 4. Refresh Token Rotation

```kotlin
// Server rotates refresh token on each use
// Old refresh token invalidated
// More secure than reusing refresh token
```

### 5. Handle Edge Cases

```kotlin
suspend fun performAuthenticatedRequest() {
    val token = tokenManager.getValidAccessToken()
    
    if (token == null) {
        // Token refresh failed - need to re-login
        navigateToLogin()
        return
    }
    
    // Proceed with request
}
```

## Common OAuth2 Providers

### Google Sign-In

```kotlin
// Use Google Sign-In library
implementation("com.google.android.gms:play-services-auth:21.2.0")

val gso = GoogleSignInOptions.Builder(GoogleSignInOptions.DEFAULT_SIGN_IN)
    .requestEmail()
    .requestIdToken(getString(R.string.default_web_client_id))
    .build()

val googleSignInClient = GoogleSignIn.getClient(this, gso)
```

### GitHub

```
Authorization URL: https://github.com/login/oauth/authorize
Token URL: https://github.com/login/oauth/access_token
```

### Custom OAuth2 Server

```kotlin
data class OAuth2Config(
    val authorizationEndpoint: String,
    val tokenEndpoint: String,
    val clientId: String,
    val redirectUri: String,
    val scopes: List<String>
)
```

## Security Considerations

### Common Vulnerabilities

#### 1. Token Interception

```kotlin
// ❌ Bad - HTTP
val baseUrl = "http://api.example.com"

// ✅ Good - HTTPS only
val baseUrl = "https://api.example.com"
```

#### 2. Token Leakage

```kotlin
// ❌ Bad - Logged
Timber.d("Token: $accessToken")

// ✅ Good - Never log tokens
Timber.d("Authentication successful")
```

#### 3. No Token Refresh

```kotlin
// ❌ Bad - Use expired tokens
val token = getAccessToken()
apiService.getData(token)  // May be expired

// ✅ Good - Validate before use
val token = getValidAccessToken()  // Refreshes if needed
apiService.getData(token)
```

#### 4. Insecure Storage

```kotlin
// ❌ Bad - Plain storage
prefs.edit { putString("token", accessToken) }

// ✅ Good - Encrypted storage
encryptedPrefs.edit { putString("token", accessToken) }
```

### PKCE Security

**Why PKCE?**

```
Without PKCE (vulnerable):
    Malicious app intercepts authorization code
        ↓
    Exchanges code for tokens
        ↓
    Gets access to user account
    
With PKCE (secure):
    Malicious app intercepts authorization code
        ↓
    Tries to exchange code
        ↓
    ❌ Fails - doesn't have code_verifier
```

## Complete Authentication Flow

### Full Implementation

```kotlin
class AuthenticationManager(
    private val context: Context,
    private val securePreferences: SecurePreferences,
    private val apiService: AuthApiService
) {
    
    private var codeVerifier: String? = null
    
    // Step 1: Start OAuth flow
    fun startOAuthFlow() {
        codeVerifier = generateCodeVerifier()
        val codeChallenge = generateCodeChallenge(codeVerifier!!)
        
        val authUrl = buildAuthorizationUrl(codeChallenge)
        
        val intent = Intent(Intent.ACTION_VIEW, Uri.parse(authUrl))
        context.startActivity(intent)
    }
    
    // Step 2: Handle callback
    suspend fun handleCallback(uri: Uri): Result<Unit> {
        val code = uri.getQueryParameter("code") ?: return Result.failure(
            Exception("No authorization code")
        )
        
        val verifier = codeVerifier ?: return Result.failure(
            Exception("No code verifier")
        )
        
        return exchangeCodeForTokens(code, verifier)
    }
    
    // Step 3: Exchange code for tokens
    private suspend fun exchangeCodeForTokens(
        code: String,
        codeVerifier: String
    ): Result<Unit> {
        return try {
            val response = apiService.getToken(
                grantType = "authorization_code",
                code = code,
                redirectUri = REDIRECT_URI,
                clientId = CLIENT_ID,
                codeVerifier = codeVerifier
            )
            
            securePreferences.saveTokens(
                accessToken = response.accessToken,
                refreshToken = response.refreshToken,
                expiresIn = response.expiresIn
            )
            
            Result.success(Unit)
        } catch (e: Exception) {
            Result.failure(e)
        }
    }
    
    // Get valid token (refresh if needed)
    suspend fun getValidAccessToken(): String? {
        val token = securePreferences.getAccessToken()
        
        return if (token != null && !isTokenExpired(token)) {
            token
        } else {
            refreshAccessToken()
        }
    }
    
    // Refresh access token
    private suspend fun refreshAccessToken(): String? {
        val refreshToken = securePreferences.getRefreshToken() ?: return null
        
        return try {
            val response = apiService.refreshToken(
                grantType = "refresh_token",
                refreshToken = refreshToken,
                clientId = CLIENT_ID
            )
            
            securePreferences.saveTokens(
                accessToken = response.accessToken,
                refreshToken = response.refreshToken ?: refreshToken,
                expiresIn = response.expiresIn
            )
            
            response.accessToken
        } catch (e: Exception) {
            securePreferences.clearTokens()
            null
        }
    }
    
    // Logout
    fun logout() {
        securePreferences.clearTokens()
    }
    
    // Check if logged in
    fun isLoggedIn(): Boolean {
        return securePreferences.getAccessToken() != null
    }
}
```

## JWT Best Practices

### 1. Don't Store Sensitive Data in JWT

```kotlin
// ❌ Bad - Sensitive data in JWT
{
    "sub": "user123",
    "password": "secret123",  // Anyone can decode!
    "ssn": "123-45-6789"
}

// ✅ Good - Only non-sensitive claims
{
    "sub": "user123",
    "name": "John Doe",
    "email": "john@example.com",
    "roles": ["user"]
}
```

### 2. Validate Expiration

```kotlin
fun isTokenValid(token: String): Boolean {
    val payload = decodeJwt(token) ?: return false
    val now = System.currentTimeMillis() / 1000
    return payload.exp > now
}
```

### 3. Use Short-Lived Access Tokens

```
Access Token:  15 min - 1 hour (short-lived)
Refresh Token: 7-90 days (long-lived)
```

### 4. Implement Token Refresh

```kotlin
// Refresh before expiration
suspend fun ensureValidToken(): String? {
    val token = getAccessToken() ?: return null
    val payload = decodeJwt(token) ?: return null
    
    val now = System.currentTimeMillis() / 1000
    val timeUntilExpiry = payload.exp - now
    
    // Refresh if less than 5 minutes remaining
    return if (timeUntilExpiry < 300) {
        refreshAccessToken()
    } else {
        token
    }
}
```

## Common Mistakes

### ❌ Validating JWT Signature on Client

```kotlin
// ❌ Never do this - client can't securely validate
fun validateSignature(token: String): Boolean {
    // Client doesn't have secret key
    // Can't securely validate
}

// ✅ Server validates signature
// Client only checks expiration
```

### ❌ Storing Tokens in Logs

```kotlin
// ❌ Bad
Timber.d("Access token: $token")

// ✅ Good
Timber.d("Successfully authenticated")
```

### ❌ Not Handling Token Expiration

```kotlin
// ❌ Bad
val token = getAccessToken()
apiService.getData(token)  // May fail if expired

// ✅ Good
val token = getValidAccessToken()  // Refreshes if needed
if (token != null) {
    apiService.getData(token)
} else {
    navigateToLogin()
}
```

## Key Takeaways

1. **OAuth2 is for Authorization**
   - Grants access to resources
   - Uses tokens, not passwords
   - Supports scopes

2. **PKCE is Required for Mobile**
   - Prevents code interception
   - No client secret needed
   - Standard for mobile apps

3. **JWT is Self-Contained**
   - Contains claims (user info)
   - Has expiration
   - Can't be validated on client

4. **Tokens Must Be Secured**
   - Encrypted storage
   - HTTPS only
   - Never logged

5. **Implement Refresh Flow**
   - Access tokens expire quickly
   - Refresh tokens last longer
   - Automatic refresh improves UX

6. **Handle Auth Failures**
   - Redirect to login when refresh fails
   - Clear tokens on logout
   - Show appropriate error messages

## Resources

- [OAuth 2.0](https://oauth.net/2/)
- [OAuth 2.0 for Mobile Apps](https://datatracker.ietf.org/doc/html/rfc8252)
- [PKCE (RFC 7636)](https://datatracker.ietf.org/doc/html/rfc7636)
- [JWT.io](https://jwt.io/)
- [JWT Best Practices](https://datatracker.ietf.org/doc/html/rfc8725)
- [Android OAuth2](https://developers.google.com/identity/protocols/oauth2/native-app)

