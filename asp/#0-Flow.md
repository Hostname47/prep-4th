# ASP.NET Flow

In this guide, i want to document the complete flow of .net from request to response by including all parts involved in the process, like controllers, dtos, services..etc.

## When a request hits your API

HTTP Request
↓
Routing (find controller/action)
↓
Model Binding (map JSON → DTO)
↓
Validation (Data Annotations on DTO)
↓
Controller (business entry point)
↓
Service Layer (business logic)
↓
Repository/Data Access (EF Core)
↓
Database
↓
Response (DTO → JSON)

--

Think of ASP.NET Core API like this:

Controller = Entry point
DTO = Contract (input/output)
Service = Brain (logic)
DbContext = Database gateway

--

## Recommended Project Structure

/Controllers
/DTOs
/Models (Domain/Entities)
/Services
/Repositories (optional if using EF directly)
/Data (DbContext)
/Mappings (AutoMapper)
/Validators (optional FluentValidation)

## Let's practice with auth microservice

First i created all the folders above emptym just to make things clear.

Then the first thing to begin with is, creating models (classes) like User, roles...

### 1. Users model

```c#
    public class User
    {
        public required Guid Id { get; set; }
        public required string FullName { get; set; }
        public required string Email { get; set; }
        public required string NormalizedEmail { get; set; }
        public required string PasswordHash { get; set; }
        public DateTime CreatedAt { get; set; } = DateTime.UtcNow;
    }
```

Please note that we use required to force dto to have those fields as a validation step, and not relying on putting valiation only in DTOs because we need validation in models as well in cases where we map or use entities directly like when mapping database records into models or when developer use model directly.

Also, notice Normalized email, this is used to make emails normalized in case client use upper and lowe cases in git email.

### 2. Create DTOs

```c#
// Register user dto
public class UserRegisterDto
{
    [Required]
    [MinLength(2)]
    [MaxLength(100)]
    public required string FullName { get; set; }
    [Required]
    [EmailAddress]
    public required string Email { get; set; }
    [Required]
    [RegularExpression(@"^(?=.*[a-z])(?=.*[A-Z])(?=.*\d)(?=.*[#$^+=!*()@%&]).{8,}$",
    ErrorMessage = "Password must be at least 8 characters long and contain at least one uppercase letter, one lowercase letter, one digit, and one special character.")]
    public required string Password { get; set; }
    [Required]
    [Compare("Password", ErrorMessage = "Password and Confirmation Password must match.")]
    public required string ConfirmPassword { get; set; }
}

public class UserLoginDto
{
    [Required]
    [EmailAddress]
    [MaxLength(320)]
    public required string Email { get; set; }
    [Required]
    [MaxLength(120)]
    public required string Password { get; set; }
}

public class UserDto
{
    public required Guid Id { get; set; }
    public required string FullName { get; set; }
    public required string Email { get; set; }
    public DateTime CreatedAt { get; set; }
    public required string Token { get; set; }
}
```

Please always make sure to have validation on your dtos like MinLength, MaxLength, EmailAddress

Please note that in APIs, DataType(DataType.<DataType>) is useless

Make sure to always use MaxLength with all your strings to prevent **Large Payload Attacks**

[Required] in a response DTO? Technically not necessary because you control the values you return
[Required] is mainly for input validation

#### Key takeaway

Input DTOs → validated, protect API
Output DTOs → sanitized, hide sensitive info

### 3. Create Mapper

```c#
public class UserMapper
{
    public static User RegisterDtoToUser(UserRegisterDto dto, IPasswordHasher<User> passwordHasher)
    {
        return new User {
            Id = Guid.NewGuid(),
            FullName = dto.FullName,
            Email = dto.Email,
            PasswordHash = passwordHasher.HashPassword(null!, dto.Password),
            NormalizedEmail = dto.Email.ToUpper(),
            CreatedAt = DateTime.UtcNow,
        };
    }

    public static UserDto UserToUserDto(User userm string? Token = null)
    {
        return new UserDto
        {
            Id = user.Id,
            FullName = user.FullName,
            Email = user.Email,
            CreatedAt = user.CreatedAt,
            Token = Token,
        };
    }
}
```

### 3. Create Repository interface, and Repository class

**REMEMBER**: Repositories should deal with models, not DTOs

UserRegisterDto and UserLoginDto are input objects from the controller
Repository should only care about the User model
Service layer is the one that converts DTO → User

Please also note that the repository should only get user by email, not verifying the user email and password hashed, you can get the user by email, then in service, you can hash the submitted password and conpare it with the hashed password returned from the repository:

Service layer will call GetUserByEmail(dto.Email)
Service layer will then verify the password using IPasswordHasher<User>
Repository never touches passwords or login logic

```c#
public interface IUserRepository
{
    User AddUser(User user);
    User? GetUserByEmail(string email);
}
```

Now before going to implementing Repository, go to sql server and create a database called authms and create users table

```sql
CREATE TABLE [dbo].[Users](
    [Id] UNIQUEIDENTIFIER NOT NULL PRIMARY KEY,
    [FullName] NVARCHAR(100) NOT NULL,
    [Email] NVARCHAR(320) NOT NULL,
    [NormalizedEmail] NVARCHAR(320) NOT NULL,
    [PasswordHash] NVARCHAR(255) NOT NULL,
    [CreatedAt] DATETIME2 NOT NULL
);
```

Then add connection string to appsettings file:

```c#
"ConnectionStrings": {
    "DefaultConnection": "Server=NASSRI;Database=authms;Trusted_Connection=True;MultipleActiveResultSets=true"
}
```

Then go to Nuget Package Manager and install **Microsoft.Data.SqlClient** package.

Now, each time we want to do something with database, we need to have IConfiguration injected, then use it to get the connection string then we pass that connection string to SqlConnection to create a database connection instance. This is how normally things should be done, but we can create a helper class that does most of this job and we can then easily use it to get the connection string and refactor:

#### Without helper class, here is how you should do it

```c#
using Microsoft.Data.SqlClient;

public class UserService
{
    private readonly IConfiguration _configuration;

    public UserService(IConfiguration configuration)
    {
        _configuration = configuration;
    }

    public void AddUser(string username)
    {
        string connectionString = _configuration.GetConnectionString("DefaultConnection");

        using SqlConnection conn = new SqlConnection(connectionString);
        string sql = "INSERT INTO Users (Username) VALUES (@username)";

        using SqlCommand cmd = new SqlCommand(sql, conn);
        cmd.Parameters.AddWithValue("@username", username);

        conn.Open();
        cmd.ExecuteNonQuery();
    }
}
```

Problem: **You repeat** GetConnectionString, SqlConnection, Open() everywhere.

#### With helper class

```c#
public class UserService
{
    private readonly DatabaseConnectionFactory _dbFactory;

    public UserService(DatabaseConnectionFactory dbFactory)
    {
        _dbFactory = dbFactory;
    }

    public void AddUser(string username)
    {
        using SqlConnection conn = _dbFactory.CreateConnection();
        string sql = "INSERT INTO Users (Username) VALUES (@username)";

        using SqlCommand cmd = new SqlCommand(sql, conn);
        cmd.Parameters.AddWithValue("@username", username);

        conn.Open();
        cmd.ExecuteNonQuery();
    }
}
```

Now that we have this helpoer let's go to implement the repository:

```c#
    public class UserRepository : IUserRepository
    {
        DatabaseConnectionFactory databaseFactory;

        public UserRepository(DatabaseConnectionFactory databaseFactory)
        {
            this.databaseFactory = databaseFactory;
        }

        public User AddUser(User user)
        {
            using SqlConnection conn = databaseFactory.CreateConnection();
            string query = "INSERT INTO Users (FullName, Email, NormalizedEmail, PasswordHash, CreatedAt) OUTPUT INSERTED.Id VALUES (@FullName, @Email, @NormalizedEmail, @PasswordHash, @CreatedAt)";

            using SqlCommand cmd = new SqlCommand(query, conn);
            cmd.Parameters.Add("@FullName", SqlDbType.NVarChar, 100).Value = user.FullName;
            cmd.Parameters.Add("@Email", SqlDbType.NVarChar, 320).Value = user.Email;
            cmd.Parameters.Add("@NormalizedEmail", SqlDbType.NVarChar, 320).Value = user.NormalizedEmail;
            cmd.Parameters.Add("@PasswordHash", SqlDbType.NVarChar, 255).Value = user.PasswordHash;
            cmd.Parameters.Add("@CreatedAt", SqlDbType.DateTime2).Value = user.CreatedAt;
            cmd.CommandType = CommandType.Text;

            conn.Open();

            object result = cmd.ExecuteScalar();
            if (result == null)
                throw new Exception("Failed to insert user.");

            user.Id = (Guid)result;

            return user;
        }

        public User? GetUserByEmail(string email)
        {
            using SqlConnection conn = databaseFactory.CreateConnection();

            using SqlCommand command = new SqlCommand("SELECT * FROM Users WHERE Email = @Email", conn);

            command.Parameters.Add("@Email", SqlDbType.NVarChar, 320).Value = email;

            conn.Open();

            using (SqlDataReader reader = command.ExecuteReader(CommandBehavior.SingleRow))
            {
                if (reader.Read())
                {
                    Guid Id = reader.GetGuid(reader.GetOrdinal("Id"));
                    string Fullname = reader.GetString(reader.GetOrdinal("FullName"));
                    string Email = reader.GetString(reader.GetOrdinal("Email"));
                    string NormalizedEmail = reader.GetString(reader.GetOrdinal("NormalizedEmail"));
                    string PasswordHash = reader.GetString(reader.GetOrdinal("PasswordHash"));
                    DateTime CreatedAt = reader.GetDateTime(reader.GetOrdinal("CreatedAt"));

                    return new User()
                    {
                        Id = Id,
                        Email = Email,
                        FullName = Fullname,
                        NormalizedEmail = NormalizedEmail,
                        PasswordHash = PasswordHash,
                        CreatedAt = CreatedAt
                    };
                }
            }

            return null;
        }
    }
```

Now that our repository is ready to be used, Let's define it in our container in **program.cs**:

```c#
builder.Services.AddScoped<IUserRepository, UserRepository>();
```

### 4. Create Auth Service
