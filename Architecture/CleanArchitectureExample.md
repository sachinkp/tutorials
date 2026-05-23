# Clean Architecture

## NoSQL - Simple
This solution demonstrates a .NET Core 9/10 clean architecture implementation where the Domain Layer remains pure (no dependencies on Cosmos DB or API contracts) and the Infrastructure Layer encapsulates the Cosmos DB SDK, preventing leakage.
We achieve separation using:
1. Domain Entities (Core)
2. DTOs/Contracts (API layer)
3. Repository Interfaces (Application layer)
4. Repository Implementation (Infrastructure layer)

1. Domain Layer (Core)Contains business logic and pure entities. 
No Cosmos DB attributes here.csharp// Domain/Entities/Product.cs
```
namespace Domain.Entities;

public class Product
{
    // Domain knows nothing about Cosmos "id"
    public Guid Id { get; private set; }
    public string Name { get; private set; }
    public decimal Price { get; private set; }
    public string Category { get; private set; } // Used as Partition Key

    public Product(Guid id, string name, decimal price, string category)
    {
        Id = id;
        Name = name;
        Price = price;
        Category = category;
    }
}

```

2. Application Layer (Interfaces & DTOs)
Defines how the application interacts with data without knowing the database type.csharp// Application/Interfaces/IProductRepository.cs
```
namespace Application.Interfaces;

public interface IProductRepository
{
    Task CreateAsync(Product product);
    Task<Product?> GetByIdAsync(Guid id, string category);
}

// Application/DTOs/CreateProductRequest.cs
namespace Application.DTOs;
public record CreateProductRequest(string Name, decimal Price, string Category);

// Application/DTOs/ProductResponse.cs
namespace Application.DTOs;
public record ProductResponse(Guid Id, string Name, decimal Price);
```

3. Infrastructure Layer (Cosmos DB)
Implements interfaces and maps domain entities to database-specific models if needed (or maps to JSON).csharp// Infrastructure/Persistence/CosmosProductEntity.cs

```
using Newtonsoft.Json;

namespace Infrastructure.Persistence;

// This is the model actually saved to Cosmos
internal class CosmosProductEntity
{
    [JsonProperty("id")] // Maps Guid to Cosmos 'id'
    public string Id { get; set; }
    public string Name { get; set; }
    public decimal Price { get; set; }
    
    [JsonProperty("category")] // Explicit Partition Key
    public string Category { get; set; }
}

// Infrastructure/Persistence/ProductRepository.cs
using Use code with caution.csharpMicrosoft.Azure.CosmosUse code with caution.csharp;

public class ProductRepository : IProductRepository
{
    private readonly Container _container;

    public ProductRepository(CosmosClient client, string dbName, string containerName)
    {
        _container = client.GetContainer(dbName, containerName);
    }

    public async Task CreateAsync(Product product)
    {
        // Map Domain -> Infrastructure Model
        var cosmosEntity = new CosmosProductEntity 
        { 
            Id = product.Id.ToString(), 
            Name = product.Name,
            Price = product.Price,
            Category = product.Category
        };

        await _container.CreateItemAsync(cosmosEntity, new PartitionKey(cosmosEntity.Category));
    }

    public async Task<Product?> GetByIdAsync(Guid id, string category)
    {
        try
        {
            ItemResponse<CosmosProductEntity> response = 
                await _container.ReadItemAsync<CosmosProductEntity>(
                    id.ToString(), 
                    new PartitionKey(category));
            
            // Map Infrastructure Model -> Domain
            return new Product(Guid.Parse(response.Resource.Id), 
                response.Resource.Name, response.Resource.Price, response.Resource.Category);
        }
        catch (CosmosException ex) when (ex.StatusCode == System.Net.HttpStatusCode.NotFound)
        {
            return null;
        }
    }
}
```

4. API Layer (Presentation)
Handles HTTP requests and maps DTOs to Domain objects.csharp// API/Controllers/ProductsController.cs

```
[ApiController]
[Route("api/[controller]")]
public class ProductsController : ControllerBase
{
    private readonly IProductRepository _repository;

    public ProductsController(IProductRepository repository)
    {
        _repository = repository;
    }

    [HttpPost]
    public async Task<IActionResult> Create(CreateProductRequest request)
    {
        var product = new Product(Guid.NewGuid(), request.Name, request.Price, request.Category);
        await _repository.CreateAsync(product);
        
        return CreatedAtAction(nameof(GetById), 
            new { id = product.Id, category = product.Category }, 
            new ProductResponse(product.Id, product.Name, product.Price));
    }

    [HttpGet("{category}/{id}")]
    public async Task<IActionResult> GetById(string category, Guid id)
    {
        var product = await _repository.GetByIdAsync(id, category);
        if (product == null) return NotFound();
        
        return Ok(new ProductResponse(product.Id, product.Name, product.Price));
    }
}
```
Key Clean Architecture Benefits in this Example
Domain Purity: Domain.Entities.Product has no [JsonProperty] or Microsoft.Azure.Cosmos namespaces.
Infrastructure Encapsulation: CosmosProductEntity is internal, hidden from API and Domain.
Separation of Contract: API receives CreateProductRequest (DTO), not the domain entity.
Partitioning Awareness: The repository handles partitioning in the infrastructure layer, not the application logic.

## NoSQL- Command & Query
This solution implements a clean architecture with strict separation of concerns, ensuring domain models do not leak into the API or Infrastructure layers. It uses CQRS (Command Query Responsibility Segregation), with Azure Cosmos DB as the NoSQL database.

1. Project Structure (Clean Architecture)
Domain: Core logic, entities (no dependencies).
Application: Use cases (Commands/Queries), interfaces, 3rd party abstraction.
Infrastructure: Cosmos DB repo, 3rd party API implementation.
API: Controllers, DTOs (API Contract).

2. Domain Layer (The Core)
No database attributes (e.g., [JsonProperty]) here. 
The domain defines what a "Product" is, not how it is stored.csharp// Domain/Entities/Product.cs
```
public class Product
{
    public Guid Id { get; private set; }
    public string Name { get; private set; }
    public decimal Price { get; private set; }
    public string PartnerId { get; private set; }

    public Product(Guid id, string name, decimal price, string partnerId)
    {
        Id = id;
        Name = name;
        Price = price;
        PartnerId = partnerId;
    }
    // Domain logic/validation
    public void UpdatePrice(decimal newPrice) { /* ... */ }
}

// Domain/Interfaces/IProductRepository.cs
public interface IProductRepository
{
    Task<Product> GetByIdAsync(Guid id);
    Task AddAsync(Product product);
}

// Domain/Interfaces/IExternalPricingService.cs
public interface IExternalPricingService
{
    Task<decimal> GetValidatedPriceAsync(string partnerId);
    Task<bool> ReportWriteAsync(Guid productId);
}
```

3. Application Layer (Use Cases)
Uses MediatR for CQRS. Handles business rules by calling the third-party service interface.csharp// Application/Products/Commands/CreateProductCommand.cs

```
public record CreateProductCommand(string Name, decimal Price, string PartnerId) : IRequest<Guid>;

public class CreateProductHandler : IRequestHandler<CreateProductCommand, Guid>
{
    private readonly IProductRepository _repo;
    private readonly IExternalPricingService _pricingService;

    public CreateProductHandler(IProductRepository repo, IExternalPricingService pricingService)
    {
        _repo = repo;
        _pricingService = pricingService;
    }

    public async Task<Guid> Handle(CreateProductCommand request, CancellationToken ct)
    {
        // BUSINESS RULE VALIDATION via Third Party
        var validatedPrice = await _pricingService.GetValidatedPriceAsync(request.PartnerId);
        
        var product = new Product(Guid.NewGuid(), request.Name, validatedPrice, request.PartnerId);
        
        await _repo.AddAsync(product);
        
        // Write to 3rd party
        await _pricingService.ReportWriteAsync(product.Id);
        
        return product.Id;
    }
}
```

4. Infrastructure Layer (Cosmos DB)
Implements repository and 3rd party client. Domain models are mapped to Data Transfer Objects (DTOs) for Cosmos.csharp// Infrastructure/Persistence/ProductDto.cs

```
// This is the model used for COSMOS DB serialization
public class ProductDto
{
    [JsonProperty("id")] // No Domain Leakage: Attributes live here only
    public string Id { get; set; }
    public string Name { get; set; }
    public decimal Price { get; set; }
    public string PartnerId { get; set; }
}

// Infrastructure/Persistence/ProductRepository.cs
public class ProductRepository : IProductRepository
{
    private readonly Container _container;
    public ProductRepository(CosmosClient client) { /* Init container */ }

    public async Task AddAsync(Product product)
    {
        // MAP Domain -> DTO
        var dto = new ProductDto { Id = product.Id.ToString(), /*...*/ };
        await _container.CreateItemAsync(dto);
    }
}

// Infrastructure/ExternalServices/ExternalPricingService.cs
public class ExternalPricingService : IExternalPricingService
{
    private readonly HttpClient _httpClient;
    public async Task<decimal> GetValidatedPriceAsync(string partnerId) { /* Call Api */ return 100; }
    public async Task<bool> ReportWriteAsync(Guid productId) { /* Call Api */ return true; }
}
```

5. API Layer (Contracts)
API contracts are strictly separated from domain entities using Request/Response records.csharp// API/Controllers/ProductsController.cs
[ApiController]

```
[Route("api/[controller]")]
public class ProductsController : ControllerBase
{
    private readonly IMediator _mediator;

    public ProductsController(IMediator mediator) => _mediator = mediator;

    [HttpPost]
    public async Task<ActionResult<Guid>> Create([FromBody] CreateProductRequest request)
    {
        // MAP API Contract -> Command
        var command = new CreateProductCommand(request.Name, request.Price, request.PartnerId);
        var result = await _mediator.Send(command);
        return Ok(result);
    }
}

// API/Models/CreateProductRequest.cs
public record CreateProductRequest(string Name, decimal Price, string PartnerId);
```

Key Takeaways for 
1. No Domain LeakageInfrastructure maps Product (Domain) to ProductDto (Cosmos).
2. API maps CreateProductRequest (Contract) to CreateProductCommand 
3. (Application).Domain contains only business logic and is agnostic to JSON or NoSQL attributes.


## NoSQL - Application Service (Preferred Approach)
### 1. API Layer: Contract Validation
We use a separate validator class to keep the ProductRequest DTO clean. This handles the "10+ properties" without cluttering the controller.
```csharp
// API Layer: The Contract
public record ProductRequest(string Name, decimal Price, string Category, string Description);

// API Layer: The Guard
public class ProductRequestValidator : AbstractValidator<ProductRequest>
{
    public ProductRequestValidator()
    {
        RuleFor(x => x.Name).NotEmpty().MaximumLength(100);
        RuleFor(x => x.Price).InclusiveBetween(0.1m, 100000m);
        RuleFor(x => x.Category).NotEmpty();
        // Add rules for the remaining 10+ properties...
    }
}

```
### 2. Domain Layer: Entity Invariants
The Domain Entity shouldn't just be a "bag of getters/setters." We use a **Factory Method** (or a constructor with logic) to ensure a Product is valid the moment it is born.
```csharp
// Domain Layer: The Truth
public class Product
{
    public string Id { get; private set; }
    public string Name { get; private set; }
    public decimal Price { get; private set; }

    // Private constructor forces usage of the Factory Method
    private Product() { }

    public static Product Create(string name, decimal price)
    {
        // Domain Rule: Logic that is true regardless of the API format
        if (string.IsNullOrWhiteSpace(name)) 
            throw new DomainException("Product name is required for the business.");
            
        if (price > 1000000) 
            throw new DomainException("Price exceeds internal corporate limits.");

        return new Product 
        { 
            Id = Guid.NewGuid().ToString(), 
            Name = name, 
            Price = price 
        };
    }
}

```
### 3. Application Layer: The Execution logic
The service doesn't need to validate the strings/decimals again—it trusts that if it has a Product object, the data is structurally sound. It focuses on the **Third-Party validation**.
```csharp
// Application Layer: The Process
public class ProductService : IProductService
{
    private readonly IProductRepository _repository;
    private readonly IThirdPartyService _thirdParty;

    public ProductService(IProductRepository repository, IThirdPartyService thirdParty)
    {
        _repository = repository;
        _thirdParty = thirdParty;
    }

    public async Task ProcessProductAsync(Product product)
    {
        // 1. Business Validation (Contextual/External)
        var validation = await _thirdParty.ValidateWithExternalSystemAsync(product);
        if (!validation.IsValid)
            throw new BusinessValidationException(validation.Reason);

        // 2. Local Write (Cosmos)
        await _repository.UpsertAsync(product);

        // 3. Remote Write (Third Party)
        await _thirdParty.RegisterWithExternalSystemAsync(product);
    }
}

```
### 4. API Controller: Putting it together
The controller orchestrates the transition from "External Contract" to "Internal Entity."
```csharp
[HttpPost]
public async Task<IActionResult> Create([FromBody] ProductRequest request)
{
    // 1. API Contract Validation (FluentValidation)
    var validator = new ProductRequestValidator();
    var validationResult = await validator.ValidateAsync(request);
    
    if (!validationResult.IsValid)
        return BadRequest(validationResult.Errors);

    try 
    {
        // 2. Map to Domain & Run Domain Invariants
        var product = Product.Create(request.Name, request.Price);

        // 3. Pass to Application Service
        await _productService.ProcessProductAsync(product);
        
        return Ok(new { id = product.Id });
    }
    catch (DomainException ex)
    {
        return UnprocessableEntity(ex.Message);
    }
}

```
### Why this is robust:
 * **Layer 1 (API):** Catches "garbage" data (empty strings, nulls) using FluentValidation.
 * **Layer 2 (Domain):** Catches "impossible" data (prices that break business logic) inside the Entity.
 * **Layer 3 (Application):** Catches "unauthorized" data (price rejected by external API) via the Service.
Since you're managing 10+ properties, would you like to see how to implement a **Global Exception Filter** so the controller doesn't need all those try-catch blocks?


## Anemic Domain Model

Strictly speaking, if the Product entity only has getters and setters (even with a factory method), it is **Anemic**. If it contains the rules that govern its own existence, it is **Rich**.
Here is the breakdown of why the current approach is hovering on the edge, and how to make it a truly **Rich Domain Model**.
### The "Anemic" Version (What we have so far)
In an anemic model, the Product is just a data container. All the "thinking" happens in the ProductService.
 * **Symptoms:** The Entity has no methods other than getters/setters.
 * **Problem:** The ProductService becomes a "God Class" that contains all the logic, while the Domain is just a hollow shell. If you add a second service (e.g., ImportService), you end up duplicating the logic.
### The "Rich" Version (True Domain Modeling)
In a Rich Domain Model, the **Entity** is responsible for its own state transitions. The **Service** only orchestrates external dependencies (DB and APIs).
#### 1. Rich Domain Entity
Notice we move the business logic *inside* the class and make the properties private set.
```csharp
// Domain Layer
public class Product
{
    public string Id { get; private set; }
    public string Name { get; private set; }
    public decimal Price { get; private set; }
    public bool IsValidatedExternally { get; private set; }

    // Logic: Only the Entity knows how to "Apply" a validation result
    public void ApplyExternalValidation(ValidationResult result)
    {
        if (!result.IsValid)
            throw new DomainException($"Business Rule Violation: {result.Reason}");

        this.IsValidatedExternally = true;
    }

    // Logic: Changing price isn't just setting a value; it's a business action
    public void AdjustPrice(decimal newPrice)
    {
        if (newPrice > this.Price * 2) 
            throw new DomainException("Price increases cannot exceed 100% in one step.");
        
        this.Price = newPrice;
        this.IsValidatedExternally = false; // Reset validation if data changes
    }
}

```
#### 2. The Orchestrator (Still Clean)
The Service remains lean. It doesn't "decide" if the product is valid; it asks the Third-Party for data and then tells the **Entity** to process that data.
```csharp
// Application Layer
public async Task ProcessProductAsync(Product product)
{
    // Service gets the context...
    var result = await _thirdParty.ValidateWithExternalSystemAsync(product);
    
    // ...but the Entity makes the decision
    product.ApplyExternalValidation(result);

    await _repository.UpsertAsync(product);
}

```
### Comparison: Anemic vs. Rich
| Feature | Anemic Model | Rich Model |
|---|---|---|
| **Logic Location** | Everything is in ProductService. | Logic is inside Product. |
| **Entity State** | Public setters (anyone can change anything). | Private setters (state changed via methods). |
| **Service Role** | "Manager" who does all the work. | "Coordinator" who connects the Entity to the DB. |
| **Testing** | Must mock services to test simple rules. | Can unit test rules directly on the Entity. |
### Is it "Anemic" to map in the Controller?
Mapping the ProductRequest to a Product entity in the controller is **not** anemic—it's just a mapping strategy. However, if that Product entity has no methods to protect its 10+ properties, the *design* is anemic.
### When to stay Anemic
If your application is 100% **CRUD** (Create, Read, Update, Delete) with no complex rules, a **Rich Domain Model** can be overkill. If your "business logic" is just saving what the user sent, an anemic model is perfectly fine.
But, since you mentioned **business rule validation** and **third-party writes**, you likely have enough complexity to justify a **Rich Model** to prevent your Service layer from becoming a tangled mess of if/else statements.
Would you like to see how to handle the **10+ properties** in a Rich Model without having a constructor with 15 arguments?

## Clean - EF & SQL
Transitioning from Cosmos DB to **Postgres** with **Entity Framework Core (EF Core)** requires a shift from a "Document" mindset to a "Relational" one. In Clean Architecture, the Domain remains largely the same, but the **Infrastructure** layer now utilizes DbContext and **Configurations**.
We will also address the "10+ properties" issue using a **Rich Domain Model** and the **Fluent API** to ensure no domain leakage.
### 1. Domain Layer (Rich Entity)
To avoid a constructor with 15 arguments, we use a **Static Factory Method**. We keep setters private to prevent the model from becoming anemic.
```csharp
// Domain/Entities/Product.cs
public class Product
{
    public Guid Id { get; private set; }
    public string Name { get; private set; }
    public decimal Price { get; private set; }
    public string Category { get; private set; }
    // ... imagine 10+ other properties here
    public DateTime CreatedAt { get; private set; }

    private Product() { } // Required for EF Core

    public static Product Create(string name, decimal price, string category)
    {
        if (price <= 0) throw new DomainException("Price must be positive.");
        
        return new Product
        {
            Id = Guid.NewGuid(),
            Name = name,
            Price = price,
            Category = category,
            CreatedAt = DateTime.UtcNow
        };
    }
}

```
### 2. Infrastructure Layer (EF Core & Postgres)
This is where the Postgres-specific logic lives. We use IEntityTypeConfiguration to map our Domain Entity to the database schema without adding attributes (like [Key]) to the Domain class.
```csharp
// Infrastructure/Persistence/ApplicationDbContext.cs
public class ApplicationDbContext : DbContext
{
    public ApplicationDbContext(DbContextOptions<ApplicationDbContext> options) : base(options) { }
    
    public DbSet<Product> Products { get; set; }

    protected override void OnModelCreating(ModelBuilder modelBuilder)
    {
        // Apply configurations from this assembly
        modelBuilder.ApplyConfigurationsFromAssembly(typeof(ApplicationDbContext).Assembly);
    }
}

// Infrastructure/Persistence/Configurations/ProductConfiguration.cs
public class ProductConfiguration : IEntityTypeConfiguration<Product>
{
    public void Configure(EntityTypeBuilder<Product> builder)
    {
        builder.ToTable("products");
        builder.HasKey(x => x.Id);
        
        builder.Property(x => x.Name).IsRequired().HasMaxLength(200);
        builder.Property(x => x.Price).HasPrecision(18, 2);
        
        // Map private properties if necessary
        builder.Property(x => x.CreatedAt).HasDefaultValueSql("CURRENT_TIMESTAMP");
    }
}

```
#### The Postgres Repository Implementation
```csharp
// Infrastructure/Persistence/Repositories/ProductRepository.cs
public class ProductRepository : IProductRepository
{
    private readonly ApplicationDbContext _context;

    public ProductRepository(ApplicationDbContext context) => _context = context;

    public async Task AddAsync(Product product)
    {
        await _context.Products.AddAsync(product);
        await _context.SaveChangesAsync();
    }

    public async Task<Product?> GetByIdAsync(Guid id) => 
        await _context.Products.FindAsync(id);
}

```
### 3. Application Layer (Orchestration)
The service handles the third-party validation and the Postgres write.
```csharp
public class ProductService : IProductService
{
    private readonly IProductRepository _repository;
    private readonly IThirdPartyService _thirdParty;

    public ProductService(IProductRepository repository, IThirdPartyService thirdParty)
    {
        _repository = repository;
        _thirdParty = thirdParty;
    }

    public async Task CreateProductAsync(Product product)
    {
        // 1. External Validation
        var validation = await _thirdParty.ValidateAsync(product);
        if (!validation.IsValid) throw new BusinessException(validation.Message);

        // 2. Postgres Write via EF Core
        await _repository.AddAsync(product);

        // 3. External Write
        await _thirdParty.NotifyCreationAsync(product);
    }
}

```
### 4. API Layer (The Entry Point)
We use the mapping strategy discussed earlier to convert the 10+ property DTO into the Rich Domain Entity.
```csharp
[HttpPost]
public async Task<IActionResult> Create(ProductRequest request)
{
    // 1. Validate the Contract (FluentValidation)
    // 2. Map Contract -> Domain Entity
    var product = Product.Create(
        request.Name, 
        request.Price, 
        request.Category
        // ... pass other properties
    );

    // 3. Call Service
    await _productService.CreateProductAsync(product);

    return Ok(new { product.Id });
}

```
### Key Differences with Postgres/EF Core:
 * **Tracking:** EF Core tracks changes to entities. In a more complex "Update" scenario, you wouldn't even need a Update method in the repository; you'd just fetch the entity, call a method on it (e.g., product.AdjustPrice(10)), and call SaveChangesAsync().
 * **Migrations:** You will use EF Core migrations to manage the Postgres schema, keeping the database in sync with your ProductConfiguration.
 * **Transactions:** EF Core handles transactions automatically. If AddAsync and NotifyCreationAsync both need to succeed or fail together, you can wrap them in an IDbContextTransaction.


## Orchstration 
When you have multiple application services working together to complete a single business process, you should introduce an **Orchestration Layer**.
In Clean Architecture, this stays within the **Application Layer**.
### Where it Lives
The Orchestration Layer is essentially a "High-Level Service" that sits inside the **Application Project**. It coordinates multiple lower-level services to ensure a business process (a "Use Case") is completed correctly.
### Why Use an Orchestrator?
If ProductService starts calling InventoryService, and InventoryService calls NotificationService, you create a **"Big Ball of Mud"** where services are tightly coupled.
Instead, the **Orchestrator** acts as the brain:
 1. **Service A** (Product) does its job.
 2. **Service B** (Inventory) does its job.
 3. **The Orchestrator** manages the flow and handles what happens if one fails.
### Code Example: The "Create Product" Orchestrator
Imagine a scenario where creating a product requires:
 1. Saving to **Postgres**.
 2. Checking **External Validation** (3rd Party API).
 3. Updating **Search Index** (Another service).
#### 1. The Orchestrator (Application Layer)
This service doesn't contain "Business Rules" (those are in the Entities); it contains "Process Logic."
```csharp
// Application/Services/ProductOrchestrator.cs
public class ProductOrchestrator : IProductOrchestrator
{
    private readonly IProductService _productService;
    private readonly IInventoryService _inventoryService;
    private readonly INotificationService _notificationService;
    private readonly IUnitOfWork _unitOfWork;

    public ProductOrchestrator(
        IProductService productService, 
        IInventoryService inventoryService,
        INotificationService notificationService,
        IUnitOfWork unitOfWork)
    {
        _productService = productService;
        _inventoryService = inventoryService;
        _notificationService = notificationService;
        _unitOfWork = unitOfWork;
    }

    public async Task ExecuteCreateProcessAsync(Product product)
    {
        // 1. Start a Transaction (Postgres)
        using var transaction = await _unitOfWork.BeginTransactionAsync();

        try
        {
            // 2. Call Service A
            await _productService.CreateProductAsync(product);

            // 3. Call Service B (e.g., reserve initial stock)
            await _inventoryService.InitializeStockAsync(product.Id);

            // 4. Commit DB changes
            await _unitOfWork.CommitAsync();

            // 5. Call Service C (External Notification - happens after DB commit)
            await _notificationService.NotifyStockManagersAsync(product);
        }
        catch (Exception)
        {
            await _unitOfWork.RollbackAsync();
            throw;
        }
    }
}

```
### Implementation Guidelines
#### 1. Keep Services Specialized
The ProductService should only care about Product entities and the ProductRepository. It should **never** know that the InventoryService exists.
#### 2. The Unit of Work Pattern
When using Postgres/EF Core, the Orchestrator often uses a **Unit of Work**. This ensures that if the InventoryService fails, the Product created by the ProductService is rolled back in the database.
#### 3. Transactional Integrity
 * **Local Transactions:** For multiple tables in Postgres, use DbContext.Database.BeginTransaction().
 * **Distributed Transactions:** If the services are different microservices with different databases, you would move from a simple Orchestrator to the **Saga Pattern** (using a tool like MassTransit).
#### 4. Controller Interaction
The Controller now calls the **Orchestrator**, not the individual services.
```csharp
[HttpPost]
public async Task<IActionResult> Create(ProductRequest request)
{
    var product = request.ToDomain(); // Map to Rich Entity
    
    // Controller talks to the "Brain"
    await _productOrchestrator.ExecuteCreateProcessAsync(product);
    
    return Ok();
}

```
### Summary of Responsibility
| Component | Responsibility |
|---|---|
| **Domain Entity** | "Can I do this?" (Internal rules). |
| **Application Service** | "How do I do this for one entity?" (CRUD/Single Logic). |
| **Orchestrator** | "In what order do we do things across multiple services?" (Workflow). |
Do you have a specific "failure scenario" in mind (like what happens if the 3rd party API is down but the Postgres write succeeded) that we should address with a compensation logic?



