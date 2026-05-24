# A guide to JWT in ASP.NET and APIs

This guide is directed by Mouad Nassri under MIT license.

## Install Jwt

dotnet add package System.IdentityModel.Tokens.Jwt
dotnet add package Microsoft.AspNetCore.Authentication.JwtBearer

## Then generate a private key

Run this first to generate a secret to be used as private key for Jwt

```sh
# Run this in linux or Git bash
openssl rand -base64 64
```

## Initialize user secrets (adds UserSecretsId to .csproj)

dotnet user-secrets init

## Store the key

dotnet user-secrets set "JwtOptions:Secret" "ko6NfhtU36UC+qazLvFqBc7nwKMYWfnMAKVnjhDnUbYJWtd3Q+DYm1bdGbddXLSPWnhdUWQirCFEWx6EqN08bw=="
dotnet user-secrets set "JwtOptions:Issuer" "mouad-jwt-app"
dotnet user-secrets set "JwtOptions:Audience" "mouad-consumer"
dotnet user-secrets set "JwtOptions:ExpiryInMinutes" "60"

## Add Placeholder in appsettings.json

```json
{
  // ...
  "JwtOptions": {
    "Secret": "",
    "Issuer": "",
    "Audience": "",
    "ExpiryInMinutes": 15,
    "RefreshExpireDays": 7
  }
}
```

Empty strings here — real values come from User Secrets / env vars

This approach is preffered as mentioned in slide 19 of Adil's course for security purposes.

## Now let's create an options class

The ASP.NET Core options extension pattern allows you to bind configuration values to **strongly typed C#** classes and register them with the dependency injection container, promoting type safety and separation of concerns.

Now go to nugget and install Microsoft Options Extension

Then let's create an option class to reflect our Jwt options in appsettings file.

```c#
// Create this class inside Options folder

public class JwtOptions
{
    public const string SectionName = "JwtOptions";
    public string Secret { get; set; } = string.Empty;
    public string Issuer { get; set; } = string.Empty;
    public string Audience { get; set; } = string.Empty;
    public int ExpiryInMinutes { get; set; } = 60;
    public int RefreshExpireDays { get; set; } = 60;
}
```

## Then let's define our Jwt service

```c#

// Interface

namespace todo_with_ef_and_jwt.Services.Jwt
{
    public interface IJwtService
    {
        public (string, string) GenerateJwtAccessAndrefreshTokens(string userId, string email, IList<string> roles);
    }
}

// Service class

using Microsoft.Extensions.Options;
using Microsoft.IdentityModel.Tokens;
using System.IdentityModel.Tokens.Jwt;
using System.Security.Claims;
using System.Security.Cryptography;
using System.Text;
using todo_with_ef_and_jwt.Options;

namespace todo_with_ef_and_jwt.Services.Jwt
{
    public class JwtService: IJwtService
    {
        JwtOptions jwtOptions;

        public JwtService(IOptions<JwtOptions> options)
        {
            jwtOptions = options.Value;
        }

        public (string, string) GenerateJwtAccessAndrefreshTokens(string userId, string email, IList<string> roles)
        {
            var key = new SymmetricSecurityKey(
                Encoding.UTF8.GetBytes(jwtOptions.Secret));

            var credentials = new SigningCredentials(
                key, SecurityAlgorithms.HmacSha256);

            var claims = new List<Claim>
            {
                new(JwtRegisteredClaimNames.Sub, userId),
                new(JwtRegisteredClaimNames.Email, email),
                new(JwtRegisteredClaimNames.Jti, Guid.NewGuid().ToString()),
            };

            claims.AddRange(roles.Select(r => new Claim(ClaimTypes.Role, r)));

            var token = new JwtSecurityToken(
                issuer: jwtOptions.Issuer,
                audience: jwtOptions.Audience,
                claims: claims,
                expires: DateTime.UtcNow.AddMinutes(jwtOptions.ExpiryInMinutes),
                signingCredentials: credentials
            );

            return (new JwtSecurityTokenHandler().WriteToken(token), GenerateRefreshToken());
        }

        private string GenerateRefreshToken()
        {
            var randomNumber = new byte[64]; // 64 bytes results in a 128-character base64 string
            using var rng = RandomNumberGenerator.Create();
            rng.GetBytes(randomNumber);
            return Convert.ToBase64String(randomNumber);
        }
    }
}

```

Then let's configure Program.cs:

```c#
// ========= Add those before app.build()

// 1. Bind JwtOptions
builder.Services.Configure<JwtOptions>(
    builder.Configuration.GetSection(JwtOptions.SectionName));

// 2. Register JWT service
builder.Services.AddScoped<IJwtService, JwtService>();

// 3. Configure Authentication
var jwtOptions = builder.Configuration
    .GetSection(JwtOptions.SectionName)
    .Get<JwtOptions>()!;

builder.Services.AddAuthentication(options =>
{
    options.DefaultAuthenticateScheme = JwtBearerDefaults.AuthenticationScheme;
    options.DefaultChallengeScheme = JwtBearerDefaults.AuthenticationScheme;
})
.AddJwtBearer(options =>
{
    options.TokenValidationParameters = new TokenValidationParameters
    {
        ValidateIssuerSigningKey = true,
        IssuerSigningKey = new SymmetricSecurityKey(
            Encoding.UTF8.GetBytes(jwtOptions.Secret)),
        ValidateIssuer = true,
        ValidIssuer = jwtOptions.Issuer,
        ValidateAudience = true,
        ValidAudience = jwtOptions.Audience,
        ValidateLifetime = true,
        ClockSkew = TimeSpan.Zero
    };
});

// Then add this before useAuthorization() please
app.UseAuthentication();

```

builder.Services.AddAuthentication is responsible for JWT reading and validating your tokens when you want to access a protected route.

## Routes to be implemented

auth/login
auth/register
auth/refresh
auth/change-password
auth/me

(Please look at todo-with-ef-and-jwt project to see code remaining code)

## Important notes

In methods, always whe you use await, the controller/service method should be declared as async.

Also when using **await** to wait for repo to get an entity, when it's done it return a real entity not Task<Entity> so make sure your method has Entity return type and not Task<Entity> because your method may has Task returned instead of the real object. In controllers, when you declare a method action as async, it forces you to have then result type to be wrapped insode Task<> because the method itself is async.

IPasswordHasher<T>.HashPassword(...) does NOT produce the same hash every time for the same password ! be careful when checking submitted password because the HashPassword will not help you check that the submitted password match the hashed one in db, It uses a random salt, so each call generates a different hash even if the password is identical.

So this comparison will almost always fail:

originalPassword != submittedPasswordHashed

You must use:

passwordHasher.VerifyHashedPassword(...)

```c#
public async Task ChangePassword(ChangePasswordDto dto)
{
    var userId = new Guid(this.httpContextAccessor.HttpContext.User
        .FindFirst(ClaimTypes.NameIdentifier)?.Value);

    User user = await dbContext.Users.FindAsync(userId);

    if (user == null)
    {
        throw new UserNotFound("User not found");
    }


    var result = passwordHasher.VerifyHashedPassword(
        user,
        user.HashedPassword,
        dto.CurrentPassword
    );

    if (result != PasswordVerificationResult.Success)
    {
        throw new InvalidePassword("Invalid password");
    }

    user.HashedPassword = passwordHasher.HashPassword(user, dto.Password);

    await dbContext.SaveChangesAsync();
}
```
