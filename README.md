
# Unit Testing vs Integration Testing - The Complete Guide

## Table of Contents
1. [Core Concepts](#1---core-concepts)
2. [Unit Testing in Detail](#2---unit-testing-in-detail)
3. [Integration Testing in Detail](#3---integration-testing-in-detail)
4. [Side-by-Side Comparison](#4---side-by-side-comparison)
5. [The Same Feature, Two Different Tests](#5---the-same-feature-two-different-tests)
6. [The Test Pyramid](#6---the-test-pyramid)
7. [When to Use Which?](#7---when-to-use-which)
8. [Key Concepts You Need to Know](#8---key-concepts-you-need-to-know)
9. [Common Mistakes](#9---common-mistakes)

---

## 1 - Core Concepts

### Unit Testing
> **Test ONE thing in complete isolation from everything else.**

Imagine you have a watch. A Unit Test is taking **one single gear** out of the watch and testing it alone:
- Does it rotate correctly?
- Are the teeth shaped right?
- You don't care about the other gears. You're testing this gear **in complete isolation**.

### Integration Testing
> **Test that multiple parts work together correctly.**

Same watch analogy. An Integration Test is assembling **a group of gears together** and checking:
- Do they rotate in sync?
- When the first gear turns, does the second one respond correctly?
- Here you're testing the **communication and integration** between parts.

---

## 2 - Unit Testing in Detail

### Key Characteristics

| Characteristic | Description |
|---------------|-------------|
| **Isolation** | Each test runs alone, no dependency on DB, API, or file system |
| **Speed** | Extremely fast (milliseconds) because there are no I/O operations |
| **Repeatability** | Same result every time, no external factors affecting it |
| **Scope** | One method or one class |

### How Do We Isolate Code? (Mocking)

The core idea: if your code talks to a Database, you DON'T actually talk to the Database.
Instead, you create a **Mock** - a **fake version** of the Database that behaves exactly how you want.

```
┌──────────────────────────────────────────┐
│          Unit Test Environment            │
│                                          │
│  ┌──────────────┐    ┌────────────────┐  │
│  │  OrderService │───>│  Mock Repo     │  │
│  │  (real code)  │    │  (FAKE)        │  │
│  │               │    │                │  │
│  │               │    │  Mock Payment  │  │
│  │               │    │  (FAKE)        │  │
│  └──────────────┘    └────────────────┘  │
│                                          │
│  ✗ No real Database                      │
│  ✗ No Network calls                     │
│  ✗ No File System (usually)             │
│  ✓ Everything lives in Memory           │
└──────────────────────────────────────────┘
```

### Example: Unit Test for an Order Service

```csharp
[Fact]
public void CalculateTotal_WithDiscountCode_AppliesDiscount()
{
    // ===== Arrange (Setup) =====

    // 1. Create a Mock for the repository - a fake version of the DB layer
    var mockOrderRepo = new Mock<IOrderRepository>();
    var mockDiscountService = new Mock<IDiscountService>();

    // 2. Define the fake behavior
    mockDiscountService.Setup(d => d.GetDiscount("SAVE20"))
                       .Returns(0.20m);  // Returns 20% without hitting any real service

    // 3. Create the real service with fake dependencies
    var orderService = new OrderService(mockOrderRepo.Object, mockDiscountService.Object);

    var order = new Order
    {
        Items = new List<OrderItem>
        {
            new() { Name = "Laptop", Price = 1000m, Quantity = 1 },
            new() { Name = "Mouse",  Price = 50m,   Quantity = 2 },
        }
    };

    // ===== Act (Execute) =====
    var total = orderService.CalculateTotal(order, discountCode: "SAVE20");

    // ===== Assert (Verify) =====
    Assert.Equal(880m, total);  // (1000 + 100) * 0.80 = 880
}
```

### Another Example: Unit Test with Verify

```csharp
[Fact]
public async Task PlaceOrder_ValidOrder_SavesAndSendsEmail()
{
    // Arrange
    var mockRepo = new Mock<IOrderRepository>();
    var mockEmailService = new Mock<IEmailService>();
    mockRepo.Setup(r => r.SaveAsync(It.IsAny<Order>())).ReturnsAsync(true);

    var service = new OrderService(mockRepo.Object, mockEmailService.Object);
    var order = new Order { CustomerEmail = "john@example.com", Total = 500m };

    // Act
    await service.PlaceOrderAsync(order);

    // Assert - Verify that the right methods were called
    mockRepo.Verify(r => r.SaveAsync(It.IsAny<Order>()), Times.Once,
        "Order should be saved to the repository exactly once");

    mockEmailService.Verify(e => e.SendConfirmation("john@example.com", It.IsAny<string>()),
        Times.Once, "Confirmation email should be sent to the customer");
}
```

### What's Happening Step by Step?

```
Step 1: Create Mocks for all external dependencies
        ↓
        We NEVER talk to a real Database or Email server

Step 2: Define fake behavior ("when X is called, return Y")
        ↓
        When the code asks for a discount, the Mock returns 20%
        No real service is involved

Step 3: Run the REAL code (OrderService)
        ↓
        The business logic runs normally, but talks to Mocks

Step 4: Verify the result OR verify that certain methods were called
        ↓
        Did it calculate the right total?
        Did it call Save? Did it send the email?
```

---

## 3 - Integration Testing in Detail

### Key Characteristics

| Characteristic | Description |
|---------------|-------------|
| **Integration** | Tests multiple parts together (code + DB + API) |
| **Speed** | Slower because it talks to real systems |
| **Realism** | Closer to production, catches issues Unit Tests will miss |
| **Scope** | Multiple components working together |

### Architecture

```
┌───────────────────────────────────────────────┐
│        Integration Test Environment            │
│                                               │
│  ┌──────────────┐    ┌────────────────────┐   │
│  │  OrderService │───>│  Real Repository   │   │
│  │  (real code)  │    │  (REAL)            │   │
│  │               │    │                    │   │
│  │               │    │  Real DbContext    │   │
│  └──────────────┘    │  (REAL)            │   │
│                       └────────┬───────────┘   │
│                                │               │
│                       ┌────────▼───────────┐   │
│                       │   Real Database     │   │
│                       │   (SQL Server /     │   │
│                       │    PostgreSQL /     │   │
│                       │    In-Memory DB)    │   │
│                       └────────────────────┘   │
│                                               │
│  ✓ Real Database (or realistic substitute)    │
│  ✓ Real Network calls                        │
│  ✓ Real ORM (Entity Framework, Dapper, etc.) │
│  ✓ Real SQL Queries                          │
└───────────────────────────────────────────────┘
```

### Example: Integration Test with a Real Database

```csharp
public class OrderServiceIntegrationTests : IDisposable
{
    // ===== Real objects, NOT Mocks =====
    private readonly AppDbContext _db;
    private readonly OrderRepository _repo;
    private readonly OrderService _service;

    public OrderServiceIntegrationTests()
    {
        // Set up a real database connection (could be a test DB or in-memory)
        var options = new DbContextOptionsBuilder<AppDbContext>()
            .UseSqlServer("Server=localhost;Database=TestDb;Trusted_Connection=true;")
            .Options;

        _db = new AppDbContext(options);
        _db.Database.EnsureCreated();  // Create tables if they don't exist

        _repo = new OrderRepository(_db);
        _service = new OrderService(_repo);
    }

    [Fact]
    public async Task PlaceOrder_ValidOrder_PersistsToDatabase()
    {
        // Arrange
        var order = new Order
        {
            CustomerEmail = "john@example.com",
            Items = new List<OrderItem>
            {
                new() { Name = "Laptop", Price = 1000m, Quantity = 1 }
            }
        };

        // Act - This actually saves to the real database
        await _service.PlaceOrderAsync(order);

        // Assert - Query the real database to verify the record exists
        var savedOrder = await _db.Orders
            .Include(o => o.Items)
            .FirstOrDefaultAsync(o => o.CustomerEmail == "john@example.com");

        Assert.NotNull(savedOrder);
        Assert.Single(savedOrder.Items);
        Assert.Equal("Laptop", savedOrder.Items[0].Name);
        Assert.Equal(1000m, savedOrder.Items[0].Price);
    }

    public void Dispose()
    {
        _db.Database.EnsureDeleted();  // Clean up the test database
        _db?.Dispose();
    }
}
```

### Example: API Integration Test

```csharp
// Boots the ENTIRE ASP.NET Core application in memory
// and sends real HTTP requests to it
public class OrdersApiTests : IClassFixture<WebApplicationFactory<Program>>
{
    private readonly HttpClient _client;

    public OrdersApiTests(WebApplicationFactory<Program> factory)
    {
        _client = factory.CreateClient();
    }

    [Fact]
    public async Task GetOrders_ReturnsSuccessStatusCode()
    {
        // Sends a real GET request to the API
        var response = await _client.GetAsync("/api/orders");

        // Verifies the response is 200 OK
        response.EnsureSuccessStatusCode();
    }

    [Fact]
    public async Task CreateOrder_ReturnsCreatedWithLocation()
    {
        // Arrange
        var newOrder = new { CustomerEmail = "test@example.com", Total = 99.99 };
        var content = new StringContent(
            JsonSerializer.Serialize(newOrder),
            Encoding.UTF8,
            "application/json");

        // Act - Sends a real POST request
        var response = await _client.PostAsync("/api/orders", content);

        // Assert
        Assert.Equal(HttpStatusCode.Created, response.StatusCode);
        Assert.NotNull(response.Headers.Location);
    }

    [Fact]
    public async Task GetOrder_InvalidId_Returns404()
    {
        var response = await _client.GetAsync("/api/orders/99999");

        Assert.Equal(HttpStatusCode.NotFound, response.StatusCode);
    }
}
```

---

## 4 - Side-by-Side Comparison

```
┌──────────────────┬────────────────────────┬────────────────────────┐
│    Criteria       │    Unit Testing        │  Integration Testing   │
├──────────────────┼────────────────────────┼────────────────────────┤
│ What it tests     │ Single method/class    │ Multiple parts together│
│                  │                        │                        │
│ Speed            │ Very fast (ms)         │ Slower (seconds)       │
│                  │                        │                        │
│ Dependencies     │ All Mocked             │ Real (DB, API, etc.)   │
│                  │                        │                        │
│ Setup complexity │ Simple                 │ Complex (config, DB)   │
│                  │                        │                        │
│ Maintenance      │ Easy                   │ Harder                 │
│                  │                        │                        │
│ Confidence level │ Individual code works  │ System works as whole  │
│                  │                        │                        │
│ What it catches  │ Logic errors           │ Communication errors   │
│                  │                        │ between components     │
│                  │                        │                        │
│ Needs DB?        │ No                     │ Yes                    │
│                  │                        │                        │
│ Needs Network?   │ No                     │ Usually yes            │
│                  │                        │                        │
│ Pattern          │ AAA                    │ AAA + Setup/Teardown   │
│                  │ (Arrange-Act-Assert)   │ + Cleanup              │
│                  │                        │                        │
│ Run frequency    │ Every commit           │ Before deploy / in CI  │
│                  │                        │                        │
│ On failure tells │ "This method is broken"│ "Something between     │
│ you...           │ (pinpointed)           │ components is broken"  │
│                  │                        │ (broader)              │
│                  │                        │                        │
│ Cost to write    │ Low                    │ Medium to High         │
│                  │                        │                        │
│ Flakiness        │ Rare                   │ More common            │
│                  │ (deterministic)        │ (network, timing, etc.)│
└──────────────────┴────────────────────────┴────────────────────────┘
```

---

## 5 - The Same Feature, Two Different Tests

### Scenario: A `UserService` that registers new users

Here's the service we're testing:

```csharp
public class UserService
{
    private readonly IUserRepository _repo;
    private readonly IEmailService _email;

    public async Task<User> RegisterAsync(string name, string email)
    {
        if (string.IsNullOrEmpty(email)) throw new ArgumentException("Email required");

        var existing = await _repo.GetByEmailAsync(email);
        if (existing != null) throw new DuplicateUserException();

        var user = new User { Name = name, Email = email, CreatedAt = DateTime.UtcNow };
        await _repo.SaveAsync(user);
        await _email.SendWelcomeEmail(email);
        return user;
    }
}
```

### The UNIT Test asks: "Does the logic work?"

```csharp
public class UserServiceUnitTests
{
    [Fact]
    public async Task Register_DuplicateEmail_ThrowsException()
    {
        // Arrange - Everything is fake
        var mockRepo = new Mock<IUserRepository>();
        var mockEmail = new Mock<IEmailService>();

        // The fake repo says "yes, this email already exists"
        mockRepo.Setup(r => r.GetByEmailAsync("john@test.com"))
                .ReturnsAsync(new User { Email = "john@test.com" });

        var service = new UserService(mockRepo.Object, mockEmail.Object);

        // Act & Assert - Should throw because email is taken
        await Assert.ThrowsAsync<DuplicateUserException>(
            () => service.RegisterAsync("John", "john@test.com"));

        // Verify that Save was NEVER called (we rejected the duplicate)
        mockRepo.Verify(r => r.SaveAsync(It.IsAny<User>()), Times.Never);

        // Verify that no welcome email was sent
        mockEmail.Verify(e => e.SendWelcomeEmail(It.IsAny<string>()), Times.Never);
    }

    [Fact]
    public async Task Register_EmptyEmail_ThrowsArgumentException()
    {
        var mockRepo = new Mock<IUserRepository>();
        var mockEmail = new Mock<IEmailService>();
        var service = new UserService(mockRepo.Object, mockEmail.Object);

        await Assert.ThrowsAsync<ArgumentException>(
            () => service.RegisterAsync("John", ""));
    }

    [Fact]
    public async Task Register_ValidUser_SavesAndSendsEmail()
    {
        var mockRepo = new Mock<IUserRepository>();
        var mockEmail = new Mock<IEmailService>();

        mockRepo.Setup(r => r.GetByEmailAsync("new@test.com"))
                .ReturnsAsync((User?)null);  // No existing user

        var service = new UserService(mockRepo.Object, mockEmail.Object);

        var user = await service.RegisterAsync("Jane", "new@test.com");

        Assert.Equal("Jane", user.Name);
        mockRepo.Verify(r => r.SaveAsync(It.IsAny<User>()), Times.Once);
        mockEmail.Verify(e => e.SendWelcomeEmail("new@test.com"), Times.Once);
    }
}
```

### The INTEGRATION Test asks: "Does it actually work end-to-end?"

```csharp
public class UserServiceIntegrationTests : IClassFixture<WebApplicationFactory<Program>>
{
    private readonly HttpClient _client;

    public UserServiceIntegrationTests(WebApplicationFactory<Program> factory)
    {
        _client = factory.CreateClient();
    }

    [Fact]
    public async Task Register_ValidUser_ReturnsCreatedAndExistsInDatabase()
    {
        // Arrange
        var request = new { Name = "Jane", Email = "jane@test.com" };
        var content = new StringContent(
            JsonSerializer.Serialize(request), Encoding.UTF8, "application/json");

        // Act - Sends a REAL HTTP POST through the full pipeline
        //       Controller → Service → Repository → Database
        var response = await _client.PostAsync("/api/users/register", content);

        // Assert - The API responded correctly
        Assert.Equal(HttpStatusCode.Created, response.StatusCode);

        // Assert - The user actually exists in the database
        var getResponse = await _client.GetAsync("/api/users?email=jane@test.com");
        var body = await getResponse.Content.ReadAsStringAsync();
        var user = JsonSerializer.Deserialize<User>(body);

        Assert.NotNull(user);
        Assert.Equal("Jane", user.Name);
    }

    [Fact]
    public async Task Register_DuplicateEmail_Returns409Conflict()
    {
        // Register the first user
        var request = new { Name = "John", Email = "john@test.com" };
        var content = new StringContent(
            JsonSerializer.Serialize(request), Encoding.UTF8, "application/json");
        await _client.PostAsync("/api/users/register", content);

        // Try to register with the same email
        var duplicate = new StringContent(
            JsonSerializer.Serialize(request), Encoding.UTF8, "application/json");
        var response = await _client.PostAsync("/api/users/register", duplicate);

        // Should return 409 Conflict
        Assert.Equal(HttpStatusCode.Conflict, response.StatusCode);
    }
}
```

### What Each Test Catches

```
                    Unit Test catches:              Integration Test catches:
                    ─────────────────               ─────────────────────────
Duplicate check ──→ ✓ Logic is correct              ✓ DB unique constraint works
                                                    ✓ Error maps to 409 status

Email validation ─→ ✓ Throws ArgumentException      ✓ API returns 400 Bad Request
                                                    ✓ Model validation works

Save user ────────→ ✓ Save() is called once          ✓ Record persists in real DB
                                                    ✓ EF mapping is correct
                                                    ✓ Column types match

Send email ───────→ ✓ SendWelcome() is called        ✓ Email service connection works
                    ✓ Called with correct email       ✓ No timeout issues

Full flow ────────→ ✗ Can't test real pipeline        ✓ Controller → Service → Repo → DB
                                                    ✓ DI container wiring is correct
                                                    ✓ Middleware runs correctly
```

### The Car Analogy

```
Unit Test        = Testing that the ENGINE runs on its own
Integration Test = Testing that the entire CAR drives

The engine might work perfectly on its own     → Unit Test PASSES
But when you put it in the car, maybe:
  - The wires aren't connected right           → Integration Test FAILS
  - The fuel line doesn't fit                  → Integration Test FAILS
  - The transmission doesn't mesh              → Integration Test FAILS

Real-world example:
  - Unit Test passes: Mock returns "save successful"
  - Integration Test fails: DB column is VARCHAR(50) but data is 60 chars
  - You'd NEVER catch that with a Unit Test
```

---

## 6 - The Test Pyramid

```
                    ╱╲
                   ╱  ╲
                  ╱ E2E╲            ← Fewest (slow, expensive, brittle)
                 ╱ Tests ╲           Selenium / Playwright
                ╱──────────╲         Full user journey in a real browser
               ╱            ╲
              ╱ Integration   ╲     ← Medium count
             ╱   Tests         ╲     API + DB + Services together
            ╱                   ╲    Real HTTP calls, real database
           ╱─────────────────────╲
          ╱                       ╲
         ╱     Unit Tests          ╲  ← Most tests (fast, cheap, stable)
        ╱                           ╲  Individual methods with Mocks
       ╱─────────────────────────────╲  Pure logic, edge cases
      ╱───────────────────────────────╲

      Fast    ←──────────────────→ Slow
      Cheap   ←──────────────────→ Expensive
      Many    ←──────────────────→ Few
      Isolated ←─────────────────→ Integrated
      Stable  ←──────────────────→ Flaky
```

### The Rule of Thumb:
- **70%** Unit Tests (the foundation - fast feedback loop)
- **20%** Integration Tests (the middle - verify components work together)
- **10%** E2E Tests (the top - full user journeys, only critical paths)

### Why a Pyramid and Not an Inverted Pyramid?

```
    ANTI-PATTERN (Ice Cream Cone)          CORRECT (Pyramid)

         ╱──────────╲                           ╱╲
        ╱  Lots of   ╲                         ╱  ╲
       ╱   Manual     ╲                       ╱ E2E╲
      ╱    Testing     ╲                     ╱──────╲
     ╱──────────────────╲                   ╱ Integ. ╲
    ╱   Lots of E2E      ╲                 ╱──────────╲
   ╱──────────────────────╲               ╱   Unit     ╲
  ╱    Some Integration    ╲             ╱──────────────╲
  ──────────────────────────
  ╱ Few/No Unit Tests      ╲

  Slow, expensive, flaky        Fast, cheap, reliable
```

---

## 7 - When to Use Which?

### Use Unit Tests when:
- You have complex business logic (calculations, conditions, transformations)
- You want to verify a method behaves correctly with many different inputs
- You want fast tests that run on every commit (< 1 second for hundreds of tests)
- You're doing TDD (Test Driven Development)
- You're testing edge cases (null, empty, negative numbers, boundary values)
- You want to verify that certain methods are called (or NOT called)

### Use Integration Tests when:
- You want to verify that database operations actually work (mapping, constraints)
- You want to test a full API endpoint (request → response)
- You need to verify configuration, connection strings, and environment setup
- You want to catch compatibility issues between different layers
- You want to test the full request pipeline (middleware, auth, validation, etc.)
- You want to verify that dependency injection is wired correctly

### Decision Flowchart

```
What am I testing?
│
├── Pure business logic (math, validation, transformations)?
│   └── Unit Test
│
├── Does it touch a database?
│   └── Integration Test
│
├── Does it call an external API or service?
│   └── Integration Test
│
├── Am I testing an API endpoint end-to-end?
│   └── Integration Test
│
├── Am I testing how multiple classes work together?
│   └── Integration Test
│
├── Am I testing edge cases of a single method?
│   └── Unit Test
│
└── Am I testing that a method calls another method correctly?
    └── Unit Test (with Mocks + Verify)
```

---

## 8 - Key Concepts You Need to Know

### AAA Pattern (Arrange - Act - Assert)
Every test follows the same structure:
```csharp
[Fact]
public void MyTest()
{
    // Arrange - Prepare everything you need
    var service = new PriceCalculator();

    // Act - Execute the operation you want to test
    var total = service.Calculate(price: 100, taxRate: 0.15m);

    // Assert - Verify the result
    Assert.Equal(115m, total);
}
```

### Mocking (using Moq library)
```csharp
// Create a fake version of an Interface
var mock = new Mock<IOrderRepository>();

// Define what happens when someone calls a specific method
mock.Setup(r => r.GetByIdAsync(42))
    .ReturnsAsync(new Order { Id = 42, Total = 99.99m });

// Use the mock in your service
var service = new OrderService(mock.Object);

// Verify that a specific method was called
mock.Verify(r => r.SaveAsync(It.IsAny<Order>()), Times.Once);
```

### Key Mocking Concepts

| Concept | What It Does | Example |
|---------|-------------|---------|
| `Mock<T>` | Creates a fake implementation of interface T | `new Mock<IRepository>()` |
| `.Setup()` | Defines behavior: "when X is called, return Y" | `.Setup(r => r.Get(1)).Returns(item)` |
| `.Returns()` | Specifies the return value | `.Returns(new List<Order>())` |
| `.ReturnsAsync()` | Same but for async methods | `.ReturnsAsync(order)` |
| `.Throws<T>()` | Makes the method throw an exception | `.Throws<TimeoutException>()` |
| `.Verify()` | Asserts that a method was called | `.Verify(r => r.Save(), Times.Once)` |
| `It.IsAny<T>()` | Matches any argument of type T | `It.IsAny<Order>()` |
| `It.Is<T>(predicate)` | Matches argument that satisfies a condition | `It.Is<int>(x => x > 0)` |
| `Times.Once` | Asserts exactly one call | Used inside `.Verify()` |
| `Times.Never` | Asserts zero calls | Used inside `.Verify()` |
| `Times.Exactly(n)` | Asserts exactly n calls | `Times.Exactly(3)` |

### Fixtures (Shared Setup)
```csharp
// IClassFixture creates the factory ONCE and shares it across all tests in the class
// This avoids the cost of booting the app for every single test
public class OrdersApiTests : IClassFixture<WebApplicationFactory<Program>>
{
    private readonly HttpClient _client;

    public OrdersApiTests(WebApplicationFactory<Program> factory)
    {
        _client = factory.CreateClient();
    }
}
```

### IDisposable (Cleanup)
```csharp
// In Integration Tests you MUST clean up after yourself
public class DatabaseTests : IDisposable
{
    private readonly AppDbContext _db;

    public void Dispose()
    {
        _db.Database.EnsureDeleted();  // Remove test database
        _db?.Dispose();                // Close the connection
    }
}
```

**Why?** Real database connections are limited resources. If you don't dispose them,
you'll run out of connections and tests will start failing randomly.

### WebApplicationFactory (API Integration Testing)
```csharp
// Boots your ENTIRE ASP.NET Core application in memory
// No need to run the app separately - it's all in-process
public class CustomWebApplicationFactory : WebApplicationFactory<Program>
{
    protected override void ConfigureWebHost(IWebHostBuilder builder)
    {
        builder.ConfigureServices(services =>
        {
            // Swap the real database with an in-memory one for testing
            services.RemoveAll<DbContextOptions<AppDbContext>>();
            services.AddDbContext<AppDbContext>(options =>
                options.UseInMemoryDatabase("TestDb"));
        });
    }
}
```

---

## 9 - Common Mistakes

### Mistake 1: Testing implementation details in Unit Tests

```csharp
// BAD - Testing HOW it works (brittle, breaks on refactor)
[Fact]
public void Calculate_CallsMultiplyInternally()
{
    // This test breaks if you refactor the math, even if the result is correct
}

// GOOD - Testing WHAT it produces (resilient, tests behavior)
[Fact]
public void Calculate_ReturnsCorrectTotal()
{
    var result = calculator.Calculate(100, 0.15m);
    Assert.Equal(115m, result);
}
```

### Mistake 2: Unit Tests that secretly depend on external systems

```csharp
// BAD - This is NOT a unit test, it hits the real database
[Fact]
public void GetUser_ReturnsUser()
{
    var repo = new UserRepository(realConnectionString);  // Real DB!
    var user = repo.GetById(1);
    Assert.NotNull(user);
}

// GOOD - True unit test with mocked dependency
[Fact]
public void GetUser_ReturnsUser()
{
    var mock = new Mock<IUserRepository>();
    mock.Setup(r => r.GetById(1)).Returns(new User { Id = 1 });
    // Now test the SERVICE that uses this repo, not the repo itself
}
```

### Mistake 3: Integration Tests without cleanup

```csharp
// BAD - Leaves data behind, causes other tests to fail
[Fact]
public void CreateOrder_SavesToDB()
{
    _db.Orders.Add(new Order { Id = 1 });
    _db.SaveChanges();
    // Test passes, but the data stays in the DB forever
}

// GOOD - Cleans up after itself
[Fact]
public void CreateOrder_SavesToDB()
{
    try
    {
        _db.Orders.Add(new Order { Id = 1 });
        _db.SaveChanges();
        Assert.NotNull(_db.Orders.Find(1));
    }
    finally
    {
        _db.Orders.Remove(_db.Orders.Find(1));
        _db.SaveChanges();
    }
}
// Or better: use IDisposable to reset the DB after each test
```

### Mistake 4: No tests at all (the worst mistake)

```
"I'll add tests later" = "I'll never add tests"

Start with the Test Pyramid:
1. Write Unit Tests for your business logic FIRST
2. Add Integration Tests for your critical paths
3. Add E2E Tests only for the most important user journeys

Even 5 well-written tests are better than 0 tests.
```

---

## Quick Summary

```
┌─────────────────────────────────────────────────────────────┐
│                                                             │
│  Unit Test:                                                 │
│    "Does this piece of code do what it's supposed to?"      │
│    → Tests logic in complete isolation                      │
│    → Fast, no external dependencies                        │
│    → Uses Mocks                                            │
│    → Runs in milliseconds                                  │
│                                                             │
│  Integration Test:                                          │
│    "Do the parts work together correctly?"                  │
│    → Tests the communication between components             │
│    → Slower, needs real DB/API                              │
│    → Uses real systems                                     │
│    → Runs in seconds                                       │
│                                                             │
│  Both are important and complement each other!              │
│  Unit Tests        = The foundation (many and fast)         │
│  Integration Tests = The safety net (fewer but deeper)      │
│                                                             │
│  A Unit Test passing does NOT mean the system works.        │
│  An Integration Test passing gives you REAL confidence.     │
│  You need BOTH.                                             │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```
