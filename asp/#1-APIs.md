API applications don't have graphical interfaces, and it delegates the UI to other separate frontend app
data are transmitted between MC (API) and V (frontend) through JSON (or xml)
**API types**:
→ Simple API (no standards)
→ REST API (partial compliance to REST norms)
→ RESTful API (full compliance to REST norms). Most production APIs are "REST APIs" rather than strictly "RESTful" because they make pragmatic trade-offs based on business needs.

### REST Norms

- **Client-Server**: Separation of concerns between client (UI) and server (logic/data).

- **Stateless**: Each request is independent; server stores no client context between requests.

- **Cacheable**: Responses explicitly marked as cacheable or non-cacheable to improve performance.
- **Layered System**: Your request passes through multiple intermediate layers (load balancers, caches, gateways, proxies) before reaching the actual server. The client doesn't know or care how many layers exist—it just sends a request and gets a response. This allows you to add/remove layers without breaking the client.
- **Example**: Client → Load Balancer → Cache → API Gateway → Application Server.
- => The client only sees one endpoint.
- **Uniform Interface**: Standardized way to communicate using consistent conventions (HTTP methods, resource URIs, standard response formats).
- **Code on Demand** (optional): The server can send executable code (like JavaScript) to the client, which the client then executes. This reduces the need for back-and-forth requests by pushing logic to the client side.
- => **Example**: Server sends JavaScript to the browser; the browser executes it locally instead of making another API call.

### ASP.NET Core API Approaches

**Controller-Based**: Organizes endpoints across multiple controller classes. Each controller handles a specific resource or feature. Recommended for large, complex projects requiring clear separation of concerns and maintainability.

**Minimal APIs**: Defines endpoints directly in `Program.cs` with minimal boilerplate. Suitable for small, simple projects with few endpoints. However, putting all logic in `Program.cs` becomes unmaintainable and is considered bad practice as the project grows.

**Best practice**: Use controller-based approach for production applications to maintain organization, scalability, and code clarity.

---

### Minimal APIs:

Define endpoints with logical handlers using lambda expressions or methods directly in `Program.cs`. They provide a lightweight, low-ceremony approach to creating HTTP endpoints without the overhead of traditional controller classes.

**Example:**

```c#
var builder = WebApplicationBuilder.CreateBuilder(args);
var app = builder.Build();

// GET products
app.MapGet("/products", () =>
{
    var products = new[]
    {
        new { id = 1, name = "Laptop" },
        new { id = 2, name = "Phone" },
        new { id = 3, name = "Tablet" }
    };
    return Results.Ok(products);
});

// GET categories
app.MapGet("/categories", () =>
{
    var categories = new[]
    {
        new { id = 1, name = "Electronics" },
        new { id = 2, name = "Clothing" },
        new { id = 3, name = "Books" }
    };
    return Results.Ok(categories);
});

app.Run();
```

### APIs with controllers

**Controller-Based APIs**: Define endpoints through controller classes that handle HTTP requests. Routes direct incoming requests to specific controller actions, which can access and manipulate Models. This approach provides better organization, separation of concerns, and scalability for larger projects.

Example of **program.cs**

```csharp
var builder = WebApplicationBuilder.CreateBuilder(args);

// Register controllers
builder.Services.AddControllers();

var app = builder.Build();

// Map controller routes
app.MapControllers();

app.Run();
```

That's it. Two lines handle everything:

- `AddControllers()` - registers controller services
- `MapControllers()` - maps all controller routes to the pipeline

**Clarifications**:

- **Pipeline**: The sequence of middleware that processes every HTTP request in ASP.NET Core. Think of it as a chain of handlers.
- **MapControllers()**: Takes all the routes defined in your controllers (using `[HttpGet]`, `[HttpPost]`, etc.) and registers them into that pipeline so they can handle incoming requests.

### Why UseRouting is not there in APIs ?:

- **UseRouting in traditional ASP.NET Core (MVC) projects**: Required to enable attribute routing and conventional routing for Views and Controllers. The routing middleware explicitly sets up the routing system before endpoint mapping occurs.
- **MapControllers in API projects (no UseRouting)**: In modern ASP.NET Core (6.0+), `MapControllers()` implicitly includes routing functionality. You don't need an explicit `UseRouting()` call because `MapControllers()` handles both routing and endpoint mapping in one step. The routing is built into the endpoint mapping.

**In essence**:

- **Traditional MVC**: `UseRouting()` → explicit routing setup, then `MapControllers()` → map endpoints
- **API projects**: `MapControllers()` → combines routing + endpoint mapping (no separate `UseRouting()` needed)

Both achieve the same result, but API projects streamline the pipeline since they don't need the complexity of serving Views. The explicit `UseRouting()` in traditional projects is there for historical compatibility and cases where you might need additional middleware between routing and endpoint execution.

### Why UseHsts is Missing in API Projects

**HSTS (HTTP Strict Transport Security)** is a security header that tells browsers to always use HTTPS. It's primarily useful for **browser-based clients** that need this protection.

**API projects don't include UseHsts() because**:

- API clients (mobile apps, desktop apps, other servers) don't interpret HSTS headers—only browsers do
- APIs communicate over HTTPS directly without needing the browser to enforce it
- It's unnecessary overhead for non-browser clients

**Traditional MVC projects include it** because they serve web pages to browsers, which benefit from HSTS protection.

**In summary**: HSTS is a browser security feature. Since APIs serve non-browser clients, it's omitted by default. You can still add it if needed, but it provides no security benefit for API consumption.

---

**MVC Project - Program.cs**:

```csharp
var builder = WebApplicationBuilder.CreateBuilder(args);

builder.Services.AddControllersWithViews();

var app = builder.Build();

if (!app.Environment.IsDevelopment())
{
    app.UseExceptionHandler("/Home/Error");
    app.UseHsts();
}

app.UseHttpsRedirection();
app.UseStaticFiles();
app.UseRouting();
app.UseAuthorization();

app.MapControllerRoute(
    name: "default",
    pattern: "{controller=Home}/{action=Index}/{id?}");

app.Run();
```

**API Project - Program.cs**:

```csharp
var builder = WebApplicationBuilder.CreateBuilder(args);

builder.Services.AddControllers();

var app = builder.Build();

if (!app.Environment.IsDevelopment())
{
    app.UseExceptionHandler("/error");
}

app.UseHttpsRedirection();
app.UseAuthorization();

app.MapControllers();

app.Run();
```

**Key differences**:

- MVC: `AddControllersWithViews()` + `UseStaticFiles()` + `UseHsts()` + `MapControllerRoute()`+`UseRouting()`
- API: `AddControllers()` + `MapControllers()` (simpler, no Views/static files)
  - AddControllersWithViews replaced by AddControllers().
  - MapControllerRoute() replaced by MapControllers() and remove configuration of routes inside MapControllerRoute.

**IMPORTANT**: notice that the **order** of routing and authorization and _middleware_ in general **does matter**.
**Example**:

```csharp
app.UseHttpsRedirection();
app.UseAuthorization();

app.MapControllers();
```

**`MapControllers()` comes last**:

- `MapControllers()` isn't middleware—it's endpoint mapping
- It must be after all middleware because middleware processes requests **before** reaching endpoints
- If `MapControllers()` came first, authorization wouldn't work (it would never execute)

### What is OpenAPI?

OpenAPI is a **standard specification** that describes REST APIs in a machine-readable format (JSON/YAML). It enables automatic API documentation and tooling.

#### `AddOpenApi()` in .NET 9

```csharp
builder.Services.AddOpenApi();
```

**What it does:**

- Registers the built-in OpenAPI document generator
- Replaces the older `AddSwaggerGen()` pattern
- Automatically produces an OpenAPI (JSON) document describing your API endpoints, parameters, and responses

**Result:**

- Auto-generated API documentation
- Swagger UI integration (with `app.MapOpenApi()`)
- Machine-readable API specification for clients and tools

#### `MapOpenApi()`

```csharp
app.MapOpenApi();
```

**Function:**

- Generates and **exposes** the OpenAPI document
- Creates an endpoint (typically `GET /openapi/v1.json`)
- Uses the configuration from `AddOpenApi()`

| Environment     | Status                               |
| --------------- | ------------------------------------ |
| **Development** | ✅ OpenAPI document exposed          |
| **Production**  | ❌ Not exposed (security by default) |

**Access Point:** `https://localhost:7xxx/openapi/v1.json`

Useful but:

- 📄 Complex to read (JSON/YAML format)
- ❌ No built-in testing support

**Solution:** Use external tools like **Postman** to test your API.

This JSON file describes your entire API (endpoints, parameters, responses, schemas).

### Workflow

```
AddOpenApi()        →  [Registers generator]
    ↓
MapOpenApi()        →  [Exposes document endpoint]
    ↓
GET /openapi/v1.json  →  [Returns API description]
```

**Security Note**: Production environments **automatically exclude** the endpoint—no API structure leakage.

**Quick Setup:**

```csharp
builder.Services.AddOpenApi();
app.MapOpenApi();
```

### OpenAPI + Scalar UI - Complete Brief

**Setup**

```bash
dotnet add package Scalar.AspNetCore
```

```csharp
// Program.cs
builder.Services.AddOpenApi();

var app = builder.Build();

if (app.Environment.IsDevelopment())
{
    app.MapOpenApi();                    // Generate OpenAPI JSON
    app.MapScalarApiReference();         // Beautiful UI
}

app.Run();
```

**What You Get**

`AddOpenApi()` : Generates OpenAPI JSON document

`MapOpenApi()` : Exposes JSON at `/openapi/v1.json`

`MapScalarApiReference()` : Beautiful interactive UI for testing

- **UI:** `https://localhost:7xxx/scalar/v1`
- **Raw JSON:** `https://localhost:7xxx/openapi/v1.json`

**Features**

✅ Interactive API documentation  
✅ Test endpoints directly  
✅ Beautiful, modern UI  
✅ Development-only (secure in production)

---

**Key:** Scalar reads the OpenAPI JSON—don't remove `MapOpenApi()`!

### What is Scalar?

Scalar is an interactive API documentation and testing tool that consumes the OpenAPI JSON output from MapOpenApi().

It automatically lists all controllers and their functions in a clear, organized manner. With Scalar, you can test your backend directly without needing Postman or equivalent software.

Scalar should not be considered a replacement for Postman. Postman is more advanced, with one of its main advantages being the creation of automated tests.

Scalar improves collaboration between developers, testers, and clients by providing readable and understandable API documentation along with tools to facilitate interaction with APIs.

### Api Controllers:

When adding a controller, choose API Controller template.

The controller inherits from **ControllerBase** instead of _Controller_. The Controller class contains unused elements like ViewBag, ViewData, TempData, and View() that are unnecessary for APIs.

The controller uses two annotations:

**[ApiController]** allows AddOpenApi() to collect controller information for documentation.
**[Route("api/[controller]")]** defines the routing rule. Access it via: localhost/api/Products

This setup ensures the controller is properly documented in Scalar and follows REST API conventions.

**Example:**

```csharp
[ApiController]
[Route("api/[controller]")]
public class ProductsController : ControllerBase
{
    [HttpGet]
    public IActionResult GetProducts()
    {
        return Ok(new { message = "List of products" });
    }
}
```

### API Controller Routing Requirement

If you attempt to change routing rules in Program.cs using traditional mapping, the code will not work. An API controller must contain [Route] either at the class level or on individual actions.

Reasons _MapControllerRoute_ no longer works:

- **Convention Over Configuration** - Modern .NET prioritizes explicit conventions over implicit assumptions.
- **Resource-Oriented Approach (ROA)** - REST API standards require each resource (controller) to be self-located. Every controller must define its own route independently.

**Example:**

```csharp
[ApiController]
[Route("api/[controller]")]
public class ProductsController : ControllerBase
{
    [HttpGet]
    public IActionResult GetAll() => Ok(new { items = "products" });

    [HttpGet("{id}")]
    public IActionResult GetById(int id) => Ok(new { id });

    [HttpPost]
    public IActionResult Create() => CreatedAtAction(nameof(GetById), new { id = 1 });
}
```

Each action inherits the route from the class. Routes are explicit and self-documenting.

The function is identified by the HTTP verb GET, not by its name. The system detects it based on the [HttpGet] attribute.
In our example above, to access GetAll, you should navigate: **GET** localhost/api/Products
The HTTP verb [HttpGet] is what matters for routing and discovery, not the method name.

**Notice**:
When you run the project, the Scalar interface automatically updates to reflect your API changes. It displays:

- The Products controller
- The Hello function
- The GET HTTP verb

Scalar reads the OpenAPI JSON generated by MapOpenApi() and instantly shows all available endpoints with their methods and parameters.

No manual configuration needed. Changes to controllers appear immediately in the UI.

### HTTP Status Codes in RESTful APIs

- 1xx = Information
- 2xx = Success
- 3xx = Redirection
- 4xx = Client Error
- 5xx = Server Error

### Common codes:

**2xx Success**

- 200 OK - Request succeeded
- 201 Created - Resource created
- 204 No Content - Success, no response body

**3xx Redirection**

- 301 Moved Permanently - Resource relocated

**4xx Client Error**

- 400 Bad Request - Invalid request
- 401 Unauthorized - Authentication required
- 403 Forbidden - Access denied
- 404 Not Found - Resource doesn't exist

**5xx Server Error**

- 500 Internal Server Error - Server failure
- 503 Service Unavailable - Server temporarily down

### HTTP Status Codes in ASP.NET Core

| Code | Function                     | Use Case                |
| ---- | ---------------------------- | ----------------------- |
| 200  | `Ok()`                       | Select/Read             |
| 201  | `CreatedAtAction()`          | Insert/Create           |
| 204  | `NoContent()`                | Delete, Update          |
| 300  | `Redirect()`                 | Redirection             |
| 400  | `BadRequest()`               | Invalid request         |
| 401  | `Unauthorized()`             | Authentication required |
| 403  | `Forbid()`                   | Access denied           |
| 404  | `NotFound()`                 | Resource not found      |
| 405  | `MethodNotAllowed()`         | HTTP method not allowed |
| 406  | `StatusCode(406, "message")` | Not acceptable          |
| 408  | `StatusCode(408, "message")` | Request timeout         |
| 409  | `Conflict()`                 | Resource conflict       |
| 500  | Exception                    | Server error            |
| 504  | `StatusCode(504, "message")` | Gateway timeout         |

### HTTP Verbs in REST APIs

HTTP verbs describe exactly what a consumer wants to do with a resource.

They help API developers protect endpoints and refuse certain requests. For example, a DELETE request can be blocked.

Configuration can go further by granting execution rights for specific verbs to certain users only. For example, only Administrators can execute DELETE operations.

Common HTTP Verbs:

- GET - Retrieve data (read-only, safe)
- POST - Create new resource
- PUT - Replace entire resource
- PATCH - Partial update
- DELETE - Remove resource
- HEAD - Like GET but no response body
- OPTIONS - Describe communication options

**Security Example:**

```csharp
[HttpDelete("{id}")]
[Authorize(Roles = "Admin")]
public IActionResult Delete(int id)
{
    return NoContent();
}
```

Only users with Admin role can execute DELETE requests.

## Safe and Idempotent HTTP Methods

### Safe Methods

A method is safe when it is read-only. The request does not modify server state (no creation, modification, or deletion).

Example: GET is safe.

### Idempotent Methods

A method is idempotent when sending the same request once or multiple times produces the same result.

Example: PUT is idempotent. Calling it 5 times updates the resource to the same state as calling it once.

### Comparison Table

| Method | Idempotent | Safe |
| ------ | ---------- | ---- |
| GET    | Yes        | Yes  |
| PUT    | Yes        | No   |
| POST   | No         | No   |
| DELETE | Yes        | No   |
| PATCH  | No         | No   |

### Why It Matters

If a client is unsure whether a request succeeded, idempotent methods allow safe retries without causing duplicate side effects.

Example: Using PUT to update ensures that retrying the same request multiple times won't create duplicate updates. With POST, retries could cause multiple unintended creations.

---

**Key Difference: PUT vs PATCH**
PUT replaces the entire resource (idempotent). PATCH applies partial updates (not idempotent—repeated calls may have different outcomes).

### Why PATCH is Not Idempotent

PATCH applies partial updates. Multiple identical PATCH requests can produce different results depending on the current resource state.

**Example**

Initial resource:

```json
{ "quantity": 10 }
```

PATCH request body:

```json
{ "quantity": -5 } // Decrement by 5
```

First call: quantity becomes 5
Second call: quantity becomes 0
Third call: quantity becomes -5

Same request, different outcomes each time.

### Comparison with PUT (Idempotent)

PUT request body:

```json
{ "quantity": 5 } // Set to 5
```

First call: quantity = 5
Second call: quantity = 5
Third call: quantity = 5

Same request, same result every time.

**Conclusion:** PATCH modifies based on current state (not idempotent). PUT replaces the entire resource (idempotent).

### HTTP Verbs - Action Mapping

| Action                                     | Verb   | Note                                  |
| ------------------------------------------ | ------ | ------------------------------------- |
| Retrieve list of products                  | GET    | Safe, idempotent                      |
| Retrieve single product                    | GET    | Safe, idempotent                      |
| Retrieve product price                     | GET    | Safe, idempotent                      |
| Delete all products                        | DELETE | Idempotent                            |
| Delete product with ID = 1                 | DELETE | Idempotent                            |
| Delete product with highest ID             | DELETE | Design Error - becomes non-idempotent |
| Add product                                | POST   | Not idempotent                        |
| Modify entire product                      | PUT    | Idempotent                            |
| Modify entire product + 10% price increase | PUT    | Design Error - becomes non-idempotent |
| Modify price only with 10% increase        | PATCH  | Not idempotent                        |

---

### Design Errors

**Delete product with highest ID using DELETE**

- Deleting by "highest ID" changes each time the database state changes
- Violates idempotency principle
- Solution: Use specific ID instead

**PUT with 10% price increase**

- PUT should replace entire resource with exact values, not apply calculations
- Each call modifies the result based on current state
- Violates idempotency principle
- Solution: Use PATCH for incremental updates or provide absolute values in PUT

**Key Rule:** Avoid operations that depend on current state in idempotent methods (PUT, DELETE).

---

## CRUD Operations in API (Look at the API flow file)

It is normal as usual work; create models, DTOs, mappers, repositories...

---

**Api/Products - GET Endpoint Flow**

```
Api/Products - GET
       ↓
[HttpGet] Index
       ↓
Verify if list is empty
       ↙          ↘
   Empty         Not Empty
     ↓              ↓
 NotFound()      Ok(products)
   (404)           (200)
```

**Implementation**

```csharp
[HttpGet]
public IActionResult Index()
{
    if (productList.Count == 0)
    {
        return NotFound();
    }
    return Ok(productList);
}
```

**Response**

| Condition      | Status | Return            |
| -------------- | ------ | ----------------- |
| List is empty  | 404    | `NotFound()`      |
| List has items | 200    | `Ok(productList)` |

The above is just crafted from professor course, but pragmatically speaking, in case the endpoint is found, and the datasource reply with empty rows, you MUST returns 200 instead of NotFound since the endpoint is correct
**Clarifications**:
It must return **200 OK** with an empty array.

```json
{
  "data": []
}
```

**Reason:** The endpoint exists and is working correctly. It found zero results, which is a valid response.

**404 Not Found** means the endpoint itself doesn't exist, not that the query returned no results.

### Api/Products - POST Endpoint Flow **Add Product**

```

api/Products - POST
Body: {Libelle = "Tide", Prix = 12, Qte = 20}
       ↓
[HttpPost] Add
       ↓
Validate ModelState
       ↙              ↘
   Invalid            Valid
     ↓                 ↓
UnprocessableEntity  Check if product exists
   (422)              ↙              ↘
              Found (exists)    Not Found
                  ↓                 ↓
             Conflict(409)    Insert Product
                              ↓
                        CreatedAtAction(201)
```

**Implementation**

```csharp
[HttpPost]
public IActionResult Add(ProductAddDTO dto)
{
    if (!ModelState.IsValid)
    {
        return UnprocessableEntity();
    }

    Product search = products.Where(p => p.Libelle == dto.Libelle).FirstOrDefault();
    if (search != null)
    {
        return Conflict();
    }

    Product model = ProductsMapper.ToModel(dto);
    model.Id = Guid.NewGuid().ToString();
    products.Add(model);

    return CreatedAtAction(nameof(GetById), new { id = model.Id }, model);
}
```

**Response Codes**

| Condition      | Status | Return                  |
| -------------- | ------ | ----------------------- |
| Invalid data   | 422    | `UnprocessableEntity()` |
| Product exists | 409    | `Conflict()`            |
| Success        | 201    | `CreatedAtAction()`     |

### Api/Products/{id} - GET Endpoint Flow **Search by ID**

```
api/Products/{GUID} - GET
       ↓
[HttpGet("{id}")]
GetById(string id)
       ↓
Search for product by ID
       ↙              ↘
   Found           Not Found
     ↓                 ↓
 Ok(product)      NotFound()
   (200)             (404)
```

**Implementation**

```csharp
[HttpGet("{id}")]
public IActionResult GetById(string id)
{
    Product search = products.Where(p => p.Id == id).FirstOrDefault();

    if (search == null)
    {
        return NotFound();
    }

    return Ok(search);
}
```

**Response**

| Condition         | Status | Return        |
| ----------------- | ------ | ------------- |
| Product found     | 200    | `Ok(product)` |
| Product not found | 404    | `NotFound()`  |

**Example**

```
GET /api/products/550e8400-e29b-41d4-a716-446655440000

Response 200:
{
  "id": "550e8400-e29b-41d4-a716-446655440000",
  "libelle": "Tide",
  "prix": 12.50,
  "quantite": 20
}
```

### Api/Products/{id} - PUT Endpoint Flow **Modify Complete Product**

```
api/Products/{GUID} - PUT
Body: {Id=Guid, Libelle="Tide", Prix=12, Qte=20}
       ↓
[HttpPut("{id}")]
Update(string id, ProductUpdateDTO dto)
       ↓
Validate ID match
       ↙              ↘
   Mismatch          Match
     ↓                 ↓
BadRequest()     Search product by ID
   (400)          ↙              ↘
            Found            Not Found
              ↓                 ↓
        Check duplicate      NotFound()
        libelle             (404)
        ↙              ↘
    Exists          Doesn't exist
      ↓                 ↓
 Conflict()       Update product
   (409)               ↓
                  NoContent()
                     (204)
```

**Implementation**

```csharp
[HttpPut("{id}")]
public IActionResult Update(string id, ProductUpdateDTO dto)
{
    // Validate ID match
    if (id != dto.Id)
    {
        return BadRequest();
    }

    // Search product
    Product search = products.Where(p => p.Id == id).FirstOrDefault();
    if (search == null)
    {
        return NotFound();
    }

    // Check duplicate libelle
    Product conflict = products
        .Where(p => p.Id != id && p.Libelle == dto.Libelle)
        .FirstOrDefault();
    if (conflict != null)
    {
        return Conflict();
    }

    // Update product
    Product model = ProductsMapper.ToModel(dto);
    search.Libelle = model.Libelle;
    search.Prix = model.Prix;
    search.Qte = model.Qte;

    return NoContent();
}
```

**Response Codes**

| Condition         | Status | Return         |
| ----------------- | ------ | -------------- |
| ID mismatch       | 400    | `BadRequest()` |
| Product not found | 404    | `NotFound()`   |
| Duplicate libelle | 409    | `Conflict()`   |
| Success           | 204    | `NoContent()`  |

### BadRequest vs UnprocessableEntity - Clarified

**BadRequest (400)**
Malformed JSON syntax - request can't be parsed.

Example: `{invalid json}` or Content-Type mismatch.

**UnprocessableEntity (422)**
Valid JSON but validation fails (required fields missing, wrong type, business rules violated).

Example: Missing required field, negative price, duplicate email.

---

**Rule:**

- **400** = Parser error (invalid JSON)
- **422** = Validation error (ModelState.IsValid == false)

Use **UnprocessableEntity** for all validation failures, including missing required fields.

### Api/Products/{id} - DELETE Endpoint Flow

**Delete Product**

```
api/Products/{GUID} - DELETE
       ↓
[HttpDelete("{id}")]
Delete(string id)
       ↓
Search product by ID
       ↙              ↘
   Found           Not Found
     ↓                 ↓
  Remove          NotFound()
  product           (404)
     ↓
 NoContent()
   (204)
```

**Implementation**

```csharp
[HttpDelete("{id}")]
public IActionResult Delete(string id)
{
    Product search = products.Where(p => p.Id == id).FirstOrDefault();

    if (search == null)
    {
        return NotFound();
    }

    products.Remove(search);
    return NoContent();
}
```

## Response Codes

| Condition         | Status | Return        |
| ----------------- | ------ | ------------- |
| Product not found | 404    | `NotFound()`  |
| Success           | 204    | `NoContent()` |

## Database Context Note

If working with a database, deleting a row associated with another entity may cause a foreign key constraint error.

**Status Code:** Return **409 Conflict** if deletion fails due to database constraints.

```csharp
try
{
    products.Remove(search);
    _context.SaveChanges();
    return NoContent();
}
catch (DbUpdateException)
{
    return Conflict("Cannot delete: product has associated orders");
}
```

### Best Practices - Clean Code & Architecture

**1. Naming Conventions**

Use clear, descriptive names for variables, classes, and functions.

```csharp
// Bad
public class P { }
public void GetD() { }
int x = 10;

// Good
public class Product { }
public IActionResult GetProductById(string id) { }
int productPrice = 10;
```

**2. Follow Conventions**

Adhere to C# naming standards (PascalCase for classes, camelCase for variables).

```csharp
public class ProductController { }
public string productName { get; set; }
public async Task<IActionResult> GetById() { }
```

**3. Single Responsibility Principle (SRP)**

Separate data access logic into a Repository class.

```csharp
// Repository - Only handles database operations
public class ProductRepository
{
    public Product GetById(string id) { }
    public void Add(Product product) { }
    public void Delete(Product product) { }
}

// Service - Contains business logic
public class ProductService
{
    private readonly ProductRepository _repository;

    public ServiceResult Create(ProductDTO dto) { }
}

// Controller - Only handles HTTP requests/responses
public class ProductsController
{
    private readonly ProductService _service;

    [HttpPost]
    public IActionResult Create(ProductDTO dto) { }
}
```

**4. Dependency Injection**

Inject Repository into Service, Service into Controller.

```csharp
// Program.cs
builder.Services.AddScoped<ProductRepository>();
builder.Services.AddScoped<ProductService>();

// Controller
public class ProductsController : ControllerBase
{
    private readonly ProductService _service;

    public ProductsController(ProductService service)
    {
        _service = service;
    }
}
```

**5. Loose Coupling**

Use interfaces to decouple Controller from Service implementation.

```csharp
// Interface
public interface IProductService
{
    ServiceResult Create(ProductDTO dto);
    ServiceResult GetById(string id);
}

// Implementation
public class ProductService : IProductService { }

// Dependency Injection
builder.Services.AddScoped<IProductService, ProductService>();

// Controller uses interface
public class ProductsController : ControllerBase
{
    private readonly IProductService _service;

    public ProductsController(IProductService service)
    {
        _service = service;
    }
}
```

**Architecture Summary**

```
Controller (HTTP)
    ↓ (uses)
Service (Business Logic)
    ↓ (uses)
Repository (Data Access)
    ↓ (accesses)
Database
```

**Benefits:** Maintainability, Testability, Reusability, Scalability

### Consuming API via JavaScript

**Goal:** Create a frontend application to consume the API we built.

**Prerequisites**

- HTML and JavaScript (or TypeScript)
- AJAX calls with JavaScript or jQuery
- Understanding of asynchronous requests

**Basic Example**

Create an HTML page with a button. On click, display the API response message.

```html
<button onclick="getData()">Show</button>
<div class="screen"></div>

<script>
  function getData() {
    var xhr = new XMLHttpRequest();
    xhr.open("GET", "http://localhost:7127/api/products");

    xhr.load = function (response) {
      console.log(response);
    };

    xhr.onerror = function (error) {
      console.log(error);
    };

    xhr.send();
  }
</script>
```

**CORS Error**

When clicking the button, a CORS error appears:

```
GET https://localhost:7127/api/products
❌ CORS Missing Allow Origin
Reason: Cross-Origin Request blocked
Code 200 - [Learn more]
```

**Why?** The frontend (different origin) cannot access the backend without explicit CORS permission.

**Solution:** Enable CORS in your .NET API (Program.cs).

```csharp
// Program.cs
builder.Services.AddCors(options =>
{
    options.AddPolicy("AllowAll", policy =>
    {
        policy.AllowAnyOrigin()
              .AllowAnyMethod()
              .AllowAnyHeader();
    });
});

app.UseCors("AllowAll");
```

**Result:** API response displays successfully after enabling CORS.

### Why Postman Works But Browser Doesn't

Postman bypasses CORS restrictions because it's not a browser—it doesn't enforce CORS policies.

**What is CORS?**: CORS (Cross-Origin Resource Sharing) is a browser security mechanism that blocks requests from different origins (domains).

**The Problem**: Browsers block AJAX requests when:

- API hosted on **Domain A**
- Frontend request from **Domain B**
- Different domains = blocked by default

## Example

```
Frontend: http://localhost:3000
API: http://localhost:7127

Request blocked by browser CORS policy
```

**Why This Security Exists**: Prevents malicious scripts from unauthorized domains accessing your API and user data.

**Solution:** Configure CORS on the server to explicitly allow frontend origins. (see the solution at the top)
