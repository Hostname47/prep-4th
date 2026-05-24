# JWT Authentication in ASP.NET Core Web APIs — Practical Guide

This comprehensive guide explains **how JWT concepts (encoding, signing, verification, keys)** are applied when building **ASP.NET Core APIs**. It bridges the gap between JWT theory and real-world implementation, with practical code examples and production-ready best practices.

---

## 1. JWT Authentication in ASP.NET Core

ASP.NET Core APIs typically use the **Bearer Token Authentication scheme**, which is part of the OAuth 2.0 standard.

Authentication is handled by the package:

```
Microsoft.AspNetCore.Authentication.JwtBearer
```

The framework automatically:

1. Reads the JWT from the HTTP request's `Authorization` header
2. Decodes the header and payload (Base64URL)
3. Verifies the signature using a configured key
4. Validates claims (issuer, audience, expiration, etc.)
5. Creates a `ClaimsPrincipal` object for the request context
6. Makes the authenticated user accessible via `User` property

If validation fails → request returns **401 Unauthorized**.

---

## 2. Installing Required Package

Install the JWT authentication package:

```bash
dotnet add package Microsoft.AspNetCore.Authentication.JwtBearer
```

This package provides:

- JWT bearer token authentication middleware
- Token validation logic
- Claims principal creation
- Integration with ASP.NET Core's authentication system

---

## 3. Basic Authentication Flow in an API

A typical JWT authentication flow works like this:

```
Client                          API
   |                             |
   |----[POST /login]---------→  | (username/password)
   |                             |
   |  ← [200 + JWT token] ------|
   |                             |
   |  [Store token locally]      |
   |                             |
   |----[GET /api/products]--→   | (with: Authorization: Bearer <JWT>)
   |    (token in header)        |
   |                             | [Verify signature]
   |  ← [200 + data] ------------|
   |                             |
```

**Request example:**

```
GET /api/products HTTP/1.1
Host: api.example.com
Authorization: Bearer eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9...
```

---

## 4. Configuring JWT Authentication

Inside `Program.cs`:

```csharp
using Microsoft.AspNetCore.Authentication.JwtBearer;
using Microsoft.IdentityModel.Tokens;
using System.Text;

var builder = WebApplication.CreateBuilder(args);

// Retrieve secret key from configuration
var secretKey = builder.Configuration["Jwt:Key"];

// Configure authentication
builder.Services.AddAuthentication(options =>
{
    options.DefaultAuthenticateScheme = JwtBearerDefaults.AuthenticationScheme;
    options.DefaultChallengeScheme = JwtBearerDefaults.AuthenticationScheme;
    options.DefaultForbidScheme = JwtBearerDefaults.AuthenticationScheme;
})
.AddJwtBearer(options =>
{
    options.TokenValidationParameters = new TokenValidationParameters
    {
        // Validate the key used to sign the token
        ValidateIssuerSigningKey = true,
        IssuerSigningKey = new SymmetricSecurityKey(
            Encoding.UTF8.GetBytes(secretKey)
        ),

        // Validate the issuer
        ValidateIssuer = true,
        ValidIssuer = builder.Configuration["Jwt:Issuer"],

        // Validate the audience
        ValidateAudience = true,
        ValidAudience = builder.Configuration["Jwt:Audience"],

        // Validate the expiration time
        ValidateLifetime = true,
        ClockSkew = TimeSpan.Zero, // No clock tolerance

        // Optional: validate not before (nbf) claim
        ValidateTokenReplay = false // Set to true if using refresh token rotation
    };

    // Configure events for custom handling
    options.Events = new JwtBearerEvents
    {
        OnAuthenticationFailed = context =>
        {
            if (context.Exception is SecurityTokenExpiredException)
            {
                context.Response.StatusCode = 401;
                context.Response.ContentType = "application/json";
                return context.Response.WriteAsync(
                    "{\"error\":\"Token expired\"}"
                );
            }
            return Task.CompletedTask;
        },
        OnChallenge = context =>
        {
            context.HandleResponse();
            context.Response.StatusCode = 401;
            context.Response.ContentType = "application/json";
            return context.Response.WriteAsync(
                "{\"error\":\"Unauthorized access\"}"
            );
        }
    };
});

// Configure authorization
builder.Services.AddAuthorization();

var app = builder.Build();

// Middleware order is critical
app.UseHttpsRedirection();
app.UseAuthentication();  // Must come before UseAuthorization
app.UseAuthorization();

app.MapControllers();
app.Run();
```

**Key points:**

- `ValidateIssuerSigningKey`: Ensures the token was signed with the expected key
- `ValidateIssuer`: Verifies the token was issued by a trusted source
- `ValidateAudience`: Ensures the token is intended for your API
- `ValidateLifetime`: Checks that the token hasn't expired
- **Middleware order matters**: Authentication must come before Authorization

---

## 5. Where Keys Are Stored

**Critical principle:** Keys are **never** embedded inside the JWT itself.

Instead, they are stored securely in configuration and only used for validation on the server.

### Configuration File Example

**appsettings.json** (development only):

```json
{
  "Jwt": {
    "Key": "your-super-secret-very-long-key-min-256-bits",
    "Issuer": "myapi",
    "Audience": "myapi-users",
    "ExpirationMinutes": 30
  }
}
```

### Production Best Practices

**Never store secrets in appsettings.json for production.** Instead:

#### 1. **Environment Variables**

```bash
export Jwt:Key="your-secret-key"
export Jwt:Issuer="myapi"
export Jwt:Audience="myapi-users"
```

#### 2. **Azure Key Vault** (Recommended for Azure)

```csharp
var keyVaultUrl = new Uri($"https://{keyVaultName}.vault.azure.net");
builder.Configuration.AddAzureKeyVault(
    keyVaultUrl,
    new DefaultAzureCredential()
);
```

#### 3. **AWS Secrets Manager**

```csharp
var secretsManager = new SecretsManagerClient();
var secret = await secretsManager.GetSecretValueAsync(
    new GetSecretValueRequest { SecretId = "jwt-secret" }
);
```

#### 4. **Docker Secrets**

```bash
docker run -e Jwt:Key=$(cat /run/secrets/jwt_key) ...
```

### Key Generation Best Practices

Generate a strong secret key using:

```csharp
// Using System.Security.Cryptography
var key = new byte[32]; // 256 bits
using (var rng = System.Security.Cryptography.RandomNumberGenerator.Create())
{
    rng.GetBytes(key);
}
var secretKey = Convert.ToBase64String(key);
Console.WriteLine(secretKey);
```

Or use OpenSSL:

```bash
openssl rand -base64 32
```

**Requirements:**

- Minimum 256 bits (32 bytes) for HMAC-SHA256
- Should be cryptographically random
- Different keys per environment
- Rotate keys periodically

---

## 6. Creating a JWT Token

When a user successfully authenticates, the API generates and returns a JWT.

### Token Service

```csharp
using System.IdentityModel.Tokens.Jwt;
using System.Security.Claims;
using Microsoft.IdentityModel.Tokens;
using System.Text;

public class JwtService
{
    private readonly IConfiguration _config;
    private readonly ILogger<JwtService> _logger;

    public JwtService(IConfiguration config, ILogger<JwtService> logger)
    {
        _config = config;
        _logger = logger;
    }

    public string GenerateToken(string userId, string email, string role)
    {
        try
        {
            // Define claims (user information encoded in the token)
            var claims = new[]
            {
                // Standard JWT claims
                new Claim(JwtRegisteredClaimNames.Sub, userId),
                new Claim(JwtRegisteredClaimNames.Email, email),
                new Claim(JwtRegisteredClaimNames.Jti, Guid.NewGuid().ToString()),

                // Custom claims
                new Claim(ClaimTypes.Role, role),
                new Claim("preferred_username", email),
                new Claim("created_at", DateTime.UtcNow.ToString("O"))
            };

            // Create signing key
            var secretKey = _config["Jwt:Key"];
            var key = new SymmetricSecurityKey(Encoding.UTF8.GetBytes(secretKey));

            // Create signing credentials
            var credentials = new SigningCredentials(
                key,
                SecurityAlgorithms.HmacSha256 // or HmacSha512 for higher security
            );

            // Create the token
            var token = new JwtSecurityToken(
                issuer: _config["Jwt:Issuer"],
                audience: _config["Jwt:Audience"],
                claims: claims,
                expires: DateTime.UtcNow.AddMinutes(
                    int.Parse(_config["Jwt:ExpirationMinutes"] ?? "30")
                ),
                signingCredentials: credentials
            );

            // Serialize token to string
            var tokenHandler = new JwtSecurityTokenHandler();
            var tokenString = tokenHandler.WriteToken(token);

            _logger.LogInformation($"Token generated for user {userId}");
            return tokenString;
        }
        catch (Exception ex)
        {
            _logger.LogError($"Error generating token: {ex.Message}");
            throw;
        }
    }

    public string GenerateRefreshToken()
    {
        var randomNumber = new byte[32];
        using (var rng = System.Security.Cryptography.RandomNumberGenerator.Create())
        {
            rng.GetBytes(randomNumber);
        }
        return Convert.ToBase64String(randomNumber);
    }
}
```

### Controller Example (Login Endpoint)

```csharp
[ApiController]
[Route("api/[controller]")]
public class AuthController : ControllerBase
{
    private readonly JwtService _jwtService;
    private readonly IUserService _userService;

    public AuthController(JwtService jwtService, IUserService userService)
    {
        _jwtService = jwtService;
        _userService = userService;
    }

    [HttpPost("login")]
    public async Task<IActionResult> Login([FromBody] LoginRequest request)
    {
        // Validate credentials (in real app, check password hash)
        var user = await _userService.AuthenticateAsync(
            request.Email,
            request.Password
        );

        if (user == null)
            return Unauthorized(new { message = "Invalid credentials" });

        // Generate tokens
        var accessToken = _jwtService.GenerateToken(
            user.Id.ToString(),
            user.Email,
            user.Role
        );

        var refreshToken = _jwtService.GenerateRefreshToken();

        // Store refresh token (in database)
        await _userService.SaveRefreshTokenAsync(user.Id, refreshToken);

        return Ok(new
        {
            accessToken,
            refreshToken,
            expiresIn = 1800, // 30 minutes in seconds
            tokenType = "Bearer"
        });
    }
}
```

### Decoded Token Example

After signing, the token consists of three Base64URL-encoded parts:

**Header:**

```json
{
  "alg": "HS256",
  "typ": "JWT"
}
```

**Payload:**

```json
{
  "sub": "123",
  "email": "user@example.com",
  "role": "Admin",
  "iat": 1677000000,
  "exp": 1677001800,
  "iss": "myapi",
  "aud": "myapi-users",
  "jti": "a1b2c3d4..."
}
```

**Signature:**

```
HMACSHA256(
  base64UrlEncode(header) + "." + base64UrlEncode(payload),
  secret_key
)
```

**Final JWT:**

```
eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJzdWIiOiIxMjMiLCJlbWFpbCI6InVzZXJAZXhhbXBsZS5jb20iLCJyb2xlIjoiQWRtaW4ifQ.TJVA95OrM7E2cBab30RMHrHDcEfxjoYZgeFONFh7HgQ
```

---

## 7. What ASP.NET Core Actually Verifies

When a request arrives with a token in the `Authorization` header:

```
Authorization: Bearer eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9...
```

ASP.NET Core's JWT Bearer middleware performs these checks **in order**:

### 1. Format Validation

```csharp
// Extracts "Bearer <token>" from the header
// Ensures it matches the expected format
```

### 2. Decode (No Key Required)

```csharp
// Base64URL decode the header
// Base64URL decode the payload
// Extract claims and metadata
```

### 3. Signature Verification (Critical)

```csharp
// Recompute signature: HMACSHA256(header.payload, secret_key)
// Compare with token's signature
// If mismatch → REJECT (401 Unauthorized)

// This is what prevents tampering:
// If someone changes: "role": "user" → "role": "admin"
// The signature no longer matches, and the token is rejected
```

### 4. Claim Validation

The middleware checks:

| Claim              | Purpose                             | Default                              |
| ------------------ | ----------------------------------- | ------------------------------------ |
| `exp` (expiration) | Token must not be expired           | Checked if `ValidateLifetime = true` |
| `iss` (issuer)     | Token must be from trusted issuer   | Must match `ValidIssuer`             |
| `aud` (audience)   | Token must be intended for this API | Must match `ValidAudience`           |
| `nbf` (not before) | Token not valid before this time    | Optional                             |
| `iat` (issued at)  | Token issue timestamp               | Optional                             |

**Example validation flow:**

```csharp
var validation = new TokenValidationParameters
{
    ValidateLifetime = true,
    ValidateIssuer = true,
    ValidIssuer = "myapi",
    ValidateAudience = true,
    ValidAudience = "myapi-users",
    ClockSkew = TimeSpan.Zero // No tolerance
};

// If current time > exp claim → REJECT
// If iss claim != "myapi" → REJECT
// If aud claim != "myapi-users" → REJECT
```

### 5. Create ClaimsPrincipal

If all checks pass:

```csharp
// Claims are extracted from the JWT
var principal = new ClaimsPrincipal(new ClaimsIdentity(
    claims,
    JwtBearerDefaults.AuthenticationScheme
));

// Available in controller as User property:
var userId = User.FindFirst(ClaimTypes.NameIdentifier)?.Value;
var role = User.FindFirst(ClaimTypes.Role)?.Value;
```

---

## 8. Protecting API Endpoints

Use the `[Authorize]` attribute to require authentication:

```csharp
[ApiController]
[Route("api/[controller]")]
public class ProductsController : ControllerBase
{
    // Public endpoint (no authentication required)
    [HttpGet("featured")]
    public IActionResult GetFeaturedProducts()
    {
        return Ok(new[] { "Product 1", "Product 2" });
    }

    // Protected endpoint (requires valid JWT)
    [Authorize]
    [HttpGet]
    public IActionResult GetAllProducts()
    {
        var userId = User.FindFirst(ClaimTypes.NameIdentifier)?.Value;
        return Ok($"Products for user {userId}");
    }

    // Protected endpoint (specific role required)
    [Authorize(Roles = "Admin")]
    [HttpPost]
    public IActionResult CreateProduct([FromBody] Product product)
    {
        // Only admins can create products
        return CreatedAtAction(nameof(GetAllProducts), product);
    }
}
```

### Global Authorization Policy

Require authentication for all endpoints by default:

```csharp
builder.Services.AddAuthorization(options =>
{
    options.FallbackPolicy = new AuthorizationPolicyBuilder()
        .RequireAuthenticatedUser()
        .Build();
});
```

Then optionally allow public access:

```csharp
[AllowAnonymous]
[HttpGet("public")]
public IActionResult GetPublicInfo()
{
    return Ok("This is public");
}
```

---

## 9. Role-Based Authorization

### Defining Roles in Token

When generating the token, include role claims:

```csharp
var claims = new[]
{
    new Claim(JwtRegisteredClaimNames.Sub, userId),
    new Claim(ClaimTypes.Role, "Admin"),
    new Claim(ClaimTypes.Role, "Moderator"), // Multiple roles allowed
};
```

### Using Roles in Controllers

```csharp
[ApiController]
[Route("api/users")]
public class UsersController : ControllerBase
{
    // Admin only
    [Authorize(Roles = "Admin")]
    [HttpDelete("{id}")]
    public IActionResult DeleteUser(int id)
    {
        return Ok($"User {id} deleted");
    }

    // Admin OR Moderator
    [Authorize(Roles = "Admin,Moderator")]
    [HttpPut("{id}")]
    public IActionResult UpdateUser(int id, [FromBody] UpdateUserRequest request)
    {
        return Ok($"User {id} updated");
    }

    // Any authenticated user
    [Authorize]
    [HttpGet("profile")]
    public IActionResult GetProfile()
    {
        var userId = User.FindFirst(ClaimTypes.NameIdentifier)?.Value;
        return Ok($"Profile of user {userId}");
    }
}
```

### Policy-Based Authorization

For more complex rules:

```csharp
// In Program.cs
builder.Services.AddAuthorization(options =>
{
    options.AddPolicy("AdminOnly", policy =>
        policy.RequireRole("Admin"));

    options.AddPolicy("CanDeleteUsers", policy =>
        policy.RequireAssertion(context =>
            context.User.IsInRole("Admin") ||
            context.User.IsInRole("SuperAdmin")
        ));

    options.AddPolicy("PremiumUser", policy =>
        policy.RequireClaim("subscription_level", "Premium", "Enterprise"));
});

// In controller
[Authorize(Policy = "AdminOnly")]
[HttpDelete("{id}")]
public IActionResult DeleteUser(int id)
{
    return Ok();
}
```

---

## 10. Understanding the Security Model

### What JWT Is **NOT**

❌ **Not encryption** — The payload is Base64URL encoded, not encrypted

- Anyone can decode and read the payload
- Do NOT put passwords, credit cards, or sensitive data in JWT

❌ **Not tamper-proof on the client** — The client has full access to the token

- The client can copy, inspect, and manipulate it
- That's why the server verifies the signature

### What JWT IS

✅ **Cryptographically signed** — The signature proves authenticity

- If the payload is modified, the signature becomes invalid
- The server rejects tampered tokens

✅ **Stateless** — No server-side session storage needed

- All information is in the token
- Scales horizontally across multiple servers

✅ **Self-contained** — The token includes all necessary claims

- No database lookup needed to determine user permissions
- Reduces server load and latency

### Example Attack & Prevention

**Attack scenario:**

```
1. Attacker intercepts token: eyJhbGc...PAYLOAD...SIGNATURE
2. Attacker decodes payload and changes: "role": "user" → "role": "admin"
3. Attacker re-encodes and sends modified token
```

**Prevention:**

```
API receives token and recalculates signature:
- Expected: HMACSHA256(original_header.original_payload, key)
- Received: HMACSHA256(modified_header.modified_payload, key)
- Mismatch detected → Token rejected
```

The attacker cannot recalculate the signature without the secret key (which is server-only).

---

## 11. Implementing Refresh Tokens

Access tokens should be short-lived (15-30 minutes). For long-lived sessions, use refresh tokens.

### Token Generation Flow

```
User Login
    ↓
Generate short-lived Access Token (30 min)
Generate long-lived Refresh Token (7 days)
    ↓
Return both tokens to client
Client stores both (access in memory, refresh in httpOnly cookie)
```

### Refresh Token Service

```csharp
public class RefreshTokenService
{
    private readonly JwtService _jwtService;
    private readonly IUserRepository _userRepository;
    private readonly ILogger<RefreshTokenService> _logger;

    public RefreshTokenService(
        JwtService jwtService,
        IUserRepository userRepository,
        ILogger<RefreshTokenService> logger)
    {
        _jwtService = jwtService;
        _userRepository = userRepository;
        _logger = logger;
    }

    public async Task<TokenResponse> RefreshAccessTokenAsync(string refreshToken)
    {
        // Validate refresh token exists and is not expired
        var storedToken = await _userRepository.GetRefreshTokenAsync(refreshToken);

        if (storedToken == null || storedToken.ExpiresAt < DateTime.UtcNow)
        {
            _logger.LogWarning("Invalid or expired refresh token attempt");
            throw new UnauthorizedAccessException("Invalid refresh token");
        }

        // Generate new access token
        var user = storedToken.User;
        var newAccessToken = _jwtService.GenerateToken(
            user.Id.ToString(),
            user.Email,
            user.Role
        );

        // Optionally rotate refresh token (security best practice)
        var newRefreshToken = _jwtService.GenerateRefreshToken();
        storedToken.Token = newRefreshToken;
        storedToken.ExpiresAt = DateTime.UtcNow.AddDays(7);
        await _userRepository.UpdateRefreshTokenAsync(storedToken);

        return new TokenResponse
        {
            AccessToken = newAccessToken,
            RefreshToken = newRefreshToken,
            ExpiresIn = 1800,
            TokenType = "Bearer"
        };
    }
}
```

### Refresh Token Controller

```csharp
[ApiController]
[Route("api/[controller]")]
public class AuthController : ControllerBase
{
    private readonly RefreshTokenService _refreshTokenService;

    public AuthController(RefreshTokenService refreshTokenService)
    {
        _refreshTokenService = refreshTokenService;
    }

    [HttpPost("refresh")]
    public async Task<IActionResult> RefreshToken([FromBody] RefreshTokenRequest request)
    {
        try
        {
            var response = await _refreshTokenService.RefreshAccessTokenAsync(
                request.RefreshToken
            );
            return Ok(response);
        }
        catch (UnauthorizedAccessException)
        {
            return Unauthorized(new { message = "Invalid refresh token" });
        }
    }

    [HttpPost("logout")]
    [Authorize]
    public async Task<IActionResult> Logout()
    {
        var userId = User.FindFirst(ClaimTypes.NameIdentifier)?.Value;
        // Delete refresh token from database
        await _refreshTokenService.InvalidateRefreshTokenAsync(userId);
        return Ok(new { message = "Logged out successfully" });
    }
}
```

### Database Model for Refresh Tokens

```csharp
public class RefreshToken
{
    public int Id { get; set; }
    public int UserId { get; set; }
    public User User { get; set; }
    public string Token { get; set; } // Long random string
    public DateTime ExpiresAt { get; set; }
    public DateTime CreatedAt { get; set; }
    public string CreatedByIp { get; set; }
    public bool IsRevoked { get; set; }
    public DateTime? RevokedAt { get; set; }
}
```

---

## 12. Token Revocation and Blacklisting

Sometimes you need to invalidate tokens before expiration (e.g., on logout or password change).

### Strategy 1: Token Blacklist (Stateful)

Maintain a list of revoked tokens:

```csharp
public class TokenBlacklistService
{
    private readonly IMemoryCache _cache;
    private readonly IDistributedCache _distributedCache;

    public TokenBlacklistService(IMemoryCache cache, IDistributedCache distributedCache)
    {
        _cache = cache;
        _distributedCache = distributedCache;
    }

    public async Task RevokeTokenAsync(string token, TimeSpan expiresIn)
    {
        // Store in distributed cache (Redis) for multi-server deployments
        await _distributedCache.SetStringAsync(
            $"revoked_token:{token}",
            "revoked",
            new DistributedCacheEntryOptions
            {
                AbsoluteExpirationRelativeToNow = expiresIn
            }
        );
    }

    public async Task<bool> IsTokenRevokedAsync(string token)
    {
        var value = await _distributedCache.GetStringAsync($"revoked_token:{token}");
        return !string.IsNullOrEmpty(value);
    }
}
```

### Strategy 2: JWT ID (jti) + Token Versioning

Include a unique ID in each token and track issued versions:

```csharp
public class JwtService
{
    public string GenerateToken(string userId, string email, string role)
    {
        var jti = Guid.NewGuid().ToString();

        var claims = new[]
        {
            new Claim(JwtRegisteredClaimNames.Sub, userId),
            new Claim(JwtRegisteredClaimNames.Jti, jti), // Unique token ID
            new Claim(ClaimTypes.Role, role),
        };

        // ... rest of token generation
    }
}

// In middleware or custom validation
public class TokenValidationMiddleware
{
    private readonly ITokenRevocationService _revocationService;

    public async Task InvokeAsync(HttpContext context)
    {
        var token = context.Request.Headers["Authorization"]
            .ToString()
            .Replace("Bearer ", "");

        if (!string.IsNullOrEmpty(token))
        {
            var jti = ExtractJtiFromToken(token); // Decode JWT
            if (await _revocationService.IsRevokedAsync(jti))
            {
                context.Response.StatusCode = 401;
                await context.Response.WriteAsync("Token has been revoked");
                return;
            }
        }

        await _next(context);
    }
}
```

### Strategy 3: Token Versioning (Application-Level)

Track user token versions in the database:

```csharp
public class User
{
    public int Id { get; set; }
    public string Email { get; set; }
    public int TokenVersion { get; set; } = 1; // Increment on password change/logout

    public void InvalidateAllTokens()
    {
        TokenVersion++;
    }
}

public class JwtService
{
    public string GenerateToken(string userId, int tokenVersion)
    {
        var claims = new[]
        {
            new Claim(JwtRegisteredClaimNames.Sub, userId),
            new Claim("token_version", tokenVersion.ToString()),
        };

        // ... token generation
    }
}

// In middleware: verify token_version matches user's current TokenVersion
```

---

## 13. Working with Custom Claims

Beyond standard JWT claims, you can add application-specific claims:

### Adding Custom Claims

```csharp
var claims = new[]
{
    // Standard claims
    new Claim(JwtRegisteredClaimNames.Sub, userId),
    new Claim(JwtRegisteredClaimNames.Email, email),

    // Custom claims
    new Claim("department", "Engineering"),
    new Claim("office_location", "New York"),
    new Claim("subscription_level", "Premium"),
    new Claim("permissions", "read,write,delete"), // Comma-separated

    // Array of claims (multiple claims with same type)
    new Claim(ClaimTypes.Role, "Admin"),
    new Claim(ClaimTypes.Role, "Moderator"),
};
```

### Extracting Custom Claims

```csharp
[Authorize]
[HttpGet("my-info")]
public IActionResult GetMyInfo()
{
    var department = User.FindFirst("department")?.Value;
    var subscriptionLevel = User.FindFirst("subscription_level")?.Value;

    // For multiple claims with same type
    var roles = User.FindAll(ClaimTypes.Role).Select(c => c.Value).ToList();

    return Ok(new
    {
        department,
        subscriptionLevel,
        roles
    });
}
```

### Claims in Authorization Policies

```csharp
// In Program.cs
builder.Services.AddAuthorization(options =>
{
    options.AddPolicy("PremiumOnly", policy =>
        policy.RequireClaim("subscription_level", "Premium", "Enterprise"));

    options.AddPolicy("EngineeringTeam", policy =>
        policy.RequireClaim("department", "Engineering"));

    options.AddPolicy("HighPermission", policy =>
        policy.RequireAssertion(context =>
            {
                var permissions = context.User
                    .FindAll("permissions")
                    .Select(c => c.Value)
                    .SelectMany(p => p.Split(','))
                    .ToList();

                return permissions.Contains("delete");
            }
        ));
});

// In controller
[Authorize(Policy = "PremiumOnly")]
[HttpPost("premium-feature")]
public IActionResult UsePremiumFeature()
{
    return Ok("Premium feature accessed");
}
```

---

## 14. Proper Error Handling

### JWT Exception Handling

```csharp
[ApiController]
[Route("api/[controller]")]
public class BaseController : ControllerBase
{
    protected IActionResult HandleAuthError(Exception ex)
    {
        return ex switch
        {
            SecurityTokenExpiredException =>
                Unauthorized(new ErrorResponse
                {
                    Code = "TOKEN_EXPIRED",
                    Message = "Your session has expired. Please login again."
                }),

            SecurityTokenInvalidSignatureException =>
                Unauthorized(new ErrorResponse
                {
                    Code = "INVALID_TOKEN",
                    Message = "The token signature is invalid."
                }),

            SecurityTokenValidationException =>
                Unauthorized(new ErrorResponse
                {
                    Code = "INVALID_TOKEN",
                    Message = "The token validation failed."
                }),

            ArgumentNullException =>
                BadRequest(new ErrorResponse
                {
                    Code = "MISSING_TOKEN",
                    Message = "Authorization token is required."
                }),

            _ => StatusCode(500, new ErrorResponse
            {
                Code = "INTERNAL_ERROR",
                Message = "An unexpected error occurred."
            })
        };
    }
}

public class ErrorResponse
{
    public string Code { get; set; }
    public string Message { get; set; }
    public Dictionary<string, string[]> Errors { get; set; }
}
```

### Custom Exception Handling Middleware

```csharp
public class ErrorHandlingMiddleware
{
    private readonly RequestDelegate _next;
    private readonly ILogger<ErrorHandlingMiddleware> _logger;

    public ErrorHandlingMiddleware(RequestDelegate next, ILogger<ErrorHandlingMiddleware> logger)
    {
        _next = next;
        _logger = logger;
    }

    public async Task InvokeAsync(HttpContext context)
    {
        try
        {
            await _next(context);
        }
        catch (Exception ex)
        {
            _logger.LogError($"Unhandled exception: {ex}");

            context.Response.ContentType = "application/json";

            var response = new ErrorResponse { Code = "INTERNAL_ERROR" };

            if (ex is SecurityTokenException ste)
            {
                context.Response.StatusCode = 401;
                response.Message = "Token validation failed: " + ste.Message;
            }
            else
            {
                context.Response.StatusCode = 500;
                response.Message = "An internal server error occurred.";
            }

            await context.Response.WriteAsJsonAsync(response);
        }
    }
}

// In Program.cs
app.UseMiddleware<ErrorHandlingMiddleware>();
```

---

## 15. Best Practices for ASP.NET Core APIs

### ✅ DO

1. **Use HTTPS exclusively**

   ```csharp
   app.UseHttpsRedirection();
   // All tokens must be transmitted over HTTPS
   ```

2. **Set short expiration times**

   ```csharp
   // 15-30 minutes for access tokens
   expires: DateTime.UtcNow.AddMinutes(30)
   ```

3. **Use strong signing algorithms**

   ```csharp
   // HS512 or RS256 (asymmetric)
   SecurityAlgorithms.HmacSha512
   ```

4. **Implement refresh tokens**

   ```csharp
   // Long-lived refresh tokens for session renewal
   // Store securely on client (httpOnly cookies)
   ```

5. **Store sensitive claims separately**

   ```csharp
   // Include in token: userId, email, role
   // Don't include: password, credit card, SSN
   ```

6. **Use role-based authorization**

   ```csharp
   [Authorize(Roles = "Admin")]
   ```

7. **Validate all claims**

   ```csharp
   ValidateIssuer = true,
   ValidateAudience = true,
   ValidateLifetime = true,
   ValidateIssuerSigningKey = true
   ```

8. **Log authentication events**

   ```csharp
   _logger.LogInformation($"Token generated for user {userId}");
   _logger.LogWarning("Authentication failed for user {email}");
   ```

9. **Use dependency injection for services**

   ```csharp
   builder.Services.AddScoped<JwtService>();
   builder.Services.AddScoped<RefreshTokenService>();
   ```

10. **Implement proper CORS for token requests**
    ```csharp
    builder.Services.AddCors(options =>
    {
        options.AddPolicy("AllowSpecificOrigins", builder =>
            builder.WithOrigins("https://trusted-domain.com")
                   .AllowAnyMethod()
                   .AllowAnyHeader()
                   .AllowCredentials()
        );
    });
    ```

### ❌ DON'T

1. **Don't store sensitive data in JWT**
   - ❌ Password, API keys, credit card numbers
   - ✅ User ID, email, roles

2. **Don't use weak secrets**
   - ❌ "secret" or "123456"
   - ✅ 256-bit cryptographically random strings

3. **Don't hardcode secrets in code**
   - ❌ `var key = "my-secret-key"`
   - ✅ Use configuration/environment variables/key vaults

4. **Don't disable claim validation**
   - ❌ `ValidateLifetime = false`
   - ✅ Validate all relevant claims

5. **Don't use long token expiration**
   - ❌ `AddMinutes(10080)` (7 days)
   - ✅ `AddMinutes(30)` (30 minutes)

6. **Don't transmit tokens in URLs**
   - ❌ `https://api.example.com/data?token=xyz`
   - ✅ Use `Authorization` header: `Bearer xyz`

7. **Don't share JwtService instances improperly**
   - ❌ Static instances
   - ✅ Dependency injection

8. **Don't forget HTTPS**
   - ❌ Using JWT over HTTP
   - ✅ Always use HTTPS in production

9. **Don't skip logout implementation**
   - ❌ JWTs are valid until expiration
   - ✅ Implement token revocation/blacklisting

10. **Don't ignore token validation errors**
    - ❌ Try-catch without logging
    - ✅ Log, monitor, and alert on validation failures

---

## 16. Common Pitfalls to Avoid

### Pitfall 1: Forgetting Middleware Order

❌ Wrong:

```csharp
app.UseAuthorization();
app.UseAuthentication();
```

✅ Correct:

```csharp
app.UseAuthentication();  // Must come first
app.UseAuthorization();
```

### Pitfall 2: Not Validating Claims

❌ Risky:

```csharp
var params = new TokenValidationParameters
{
    ValidateIssuer = false,
    ValidateAudience = false,
    ValidateLifetime = false
};
```

✅ Secure:

```csharp
var params = new TokenValidationParameters
{
    ValidateIssuer = true,
    ValidIssuer = "myapi",
    ValidateAudience = true,
    ValidAudience = "myapi-users",
    ValidateLifetime = true
};
```

### Pitfall 3: Storing Tokens Insecurely on Client

❌ Unsafe:

```javascript
// Storing in localStorage (vulnerable to XSS)
localStorage.setItem("token", jwt);
```

✅ Safer:

```javascript
// Store in memory or httpOnly cookie
// httpOnly prevents JavaScript access (mitigates XSS)
```

### Pitfall 4: Not Handling Token Expiration

❌ Incomplete:

```csharp
// Token expires, but client doesn't know how to refresh
```

✅ Complete:

```csharp
// Return expiresIn with token
// Client refreshes before expiration
// Server provides refresh endpoint
```

### Pitfall 5: Using Same Key Across Environments

❌ Security issue:

```bash
Jwt:Key = "same-key-everywhere" # Dev, staging, prod
```

✅ Secure:

```bash
# Each environment has different key
export Jwt:Key="production-key-xyz" # Production
export Jwt:Key="staging-key-abc"    # Staging
```

### Pitfall 6: Not Logging Security Events

❌ No visibility:

```csharp
// Silent failures
```

✅ Good observability:

```csharp
_logger.LogWarning($"Token validation failed for user {userId}");
_logger.LogError($"Signature verification failed");
_logger.LogInformation($"Token refreshed for user {userId}");
```

---

## 17. Summary

### JWT Authentication Flow

```
1. User provides credentials → POST /api/auth/login
                              ↓
2. Server validates credentials
   Generates access token (short-lived)
   Generates refresh token (long-lived)
                              ↓
3. Client receives tokens
   Stores tokens locally
                              ↓
4. Client makes API request
   Authorization: Bearer <access_token>
                              ↓
5. Middleware intercepts request
   Validates signature ← Uses server secret key
   Validates claims (exp, iss, aud)
   Creates ClaimsPrincipal
                              ↓
6. Controller receives authenticated request
   User property contains claims
   [Authorize] enforces authentication
                              ↓
7. When access token expires
   Client calls POST /api/auth/refresh
   Sends refresh token
                              ↓
8. Server validates refresh token
   Issues new access token
   (Optionally rotates refresh token)
```

### Key Takeaways

| Concept           | Key Point                                                |
| ----------------- | -------------------------------------------------------- |
| **JWT Structure** | Three Base64URL-encoded parts: header.payload.signature  |
| **Signature**     | Cryptographic proof that token wasn't tampered with      |
| **Secret Key**    | Server-only. Never shared. Never in token.               |
| **Payload**       | Readable but not encrypted. Include claims, not secrets. |
| **Validation**    | Signature + claim verification (exp, iss, aud)           |
| **Expiration**    | Short-lived tokens (15-30 min) + refresh tokens          |
| **Stateless**     | No server-side session storage needed                    |
| **Scalable**      | Works across multiple servers without shared state       |
| **Authorization** | Roles and policies based on token claims                 |
| **Security**      | HTTPS + proper key management + claim validation         |

### Implementation Checklist

- [ ] Install `Microsoft.AspNetCore.Authentication.JwtBearer`
- [ ] Configure JWT in `Program.cs`
- [ ] Store keys securely (environment variables/key vault)
- [ ] Create `JwtService` for token generation
- [ ] Create login endpoint
- [ ] Implement refresh token logic
- [ ] Protect endpoints with `[Authorize]`
- [ ] Use role-based authorization
- [ ] Implement error handling
- [ ] Add logging for security events
- [ ] Set up token revocation if needed
- [ ] Test authentication flow end-to-end
- [ ] Enable HTTPS
- [ ] Document API authentication in OpenAPI/Swagger

---

## Additional Resources

- [Microsoft JWT Bearer Documentation](https://learn.microsoft.com/en-us/dotnet/api/microsoft.aspnetcore.authentication.jwtbearer)
- [JWT.io - JWT Debugger & Standards](https://jwt.io)
- [ASP.NET Core Security Documentation](https://learn.microsoft.com/en-us/aspnet/core/security/)
- [OAuth 2.0 Bearer Token RFC 6750](https://tools.ietf.org/html/rfc6750)
- [JWT Security Best Practices](https://tools.ietf.org/html/rfc8949)

---

**Last Updated:** March 2026  
**Version:** 2.0 (Enhanced with Refresh Tokens, Revocation, Custom Claims, and Error Handling)
