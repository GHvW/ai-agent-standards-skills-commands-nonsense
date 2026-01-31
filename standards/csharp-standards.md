
# C# Coding Standards

## Core Philosophy

Write code as a **pragmatic programmer**. Use language features that make code clearer and more maintainable. Prefer patterns where they fit naturally with C#'s language constructs, but don't force them.

## Pure Functions and Methods

### Prefer Pure Functions
- **Methods should take inputs and return outputs**
- Avoid side effects within methods unless necessary (e.g., IO operations, database calls)
- Make the return value of one method be the input to another
- Pure methods are easier to test, reason about, and reuse

```csharp
// GOOD: Pure transformation function (A -> B)
public static TransformedItem ApplyDiscount(Item item) {
    return new TransformedItem(item.Name, item.Price * 0.9);
}

// Use it with Select - just like ReadingExtensions.WithCelsius
var discounted = items.Select(ApplyDiscount).ToList();

// Easy to unit test with just ONE value
[Fact]
public void ApplyDiscount() {
    var item = new Item("Widget", 100.0);
    var result = ApplyDiscount(item);
    Assert.Equal(90.0, result.Price);
}

// BAD: Impure method with side effects
private List<TransformedItem> transformed = new();
public void TransformAndAdd(Item item) {
    var discounted = new TransformedItem(item.Name, item.Price * 0.9);
    this.transformed.Add(discounted); // Modifies state
}

// Using the BAD pattern requires managing state
foreach (var item in items) {
    TransformAndAdd(item); // Repeatedly modifies transformed
}
var result = this.transformed; // Have to read the state afterwards

// Harder to test - need MULTIPLE items to properly test collection management
[Fact]
public void TransformAndAdd() {
    this.transformed.Clear(); // Have to manage state
    var items = new List<Item> {
        new Item("Widget", 100.0),
        new Item("Gadget", 50.0)
    };
    foreach (var item in items) {
        TransformAndAdd(item);
    }
    Assert.Equal(2, this.transformed.Count);
    Assert.Equal(90.0, this.transformed[0].Price);
    Assert.Equal(45.0, this.transformed[1].Price);
}
```

## LINQ Methods

### Use LINQ When It's Clear
- **Use LINQ methods** (`Select`, `Where`, `Aggregate`) when they make code clearer
- These methods work well with pure functions
- Example: `var doubled = numbers.Select(n => n * 2);`

### When NOT to Force These Patterns
- **Don't force collection construction into `Aggregate()`** - use a `foreach` loop instead
- If a for/foreach loop is clearer and more performant, use it
- Loops aren't the devil if you don't abuse them
- Don't create complex nested LINQ queries that hurt readability

### Example: When to Use Loop vs LINQ
```csharp
// BAD: Forcing Aggregate where it doesn't fit
var result = items.Aggregate(
    new { ValidItems = new List<Item>(), InvalidItems = new List<Item>() },
    (acc, item) => {
        if (item.IsValid) {
            acc.ValidItems.Add(item);
        } else {
            acc.InvalidItems.Add(item);
        }
        return acc;
    }
);

// GOOD: Use a foreach loop when building complex collections
var validItems = new List<Item>();
var invalidItems = new List<Item>();
foreach (var item in items) {
    if (item.IsValid) {
        validItems.Add(item);
    } else {
        invalidItems.Add(item);
    }
}
```

## Anti-Patterns to Avoid

### Stateful Objects with Side-Effect Methods
**ABHOR** classes that contain all of the following:
- Have private methods for transformations
- Have public methods for data fetching
- Have public methods for "rendering"
- All of which take no arguments and return nothing
- They only update "state" in an object
- This pattern prevents separation of concerns

```csharp
// BAD: Stateful class with side-effect methods
public class DataManager {
    private List<Data> data;
    private List<TransformedData> transformedData;

    public async Task FetchDataAsync() {
        this.data = await this.httpClient.GetFromJsonAsync<List<Data>>("/api/data");
    }

    public void TransformData() {
        this.transformedData = this.data.Select(/* transform */).ToList();
    }

    public void Render() {
        // Updates some view/state with this.transformedData
    }
}

// GOOD: Separate pure functions/methods
public class DataService {
    private readonly HttpClient httpClient;

    public DataService(HttpClient httpClient) {
        this.httpClient = httpClient;
    }

    public async Task<List<Data>> FetchDataAsync() {
        return await this.httpClient.GetFromJsonAsync<List<Data>>("/api/data");
    }
}

public static class DataExtensions {
    public static TransformedData Transform(this Data data) {
        return new TransformedData( ... );
    }
}

public static class DataRenderer {
    public static string Render(List<TransformedData> data) {
        // Return HTML or other representation
        return /* render data */;
    }
}

// Use it:
var dataService = new DataService(httpClient);
var data = await dataService.FetchDataAsync();
var transformed = data.Select(DataExtensions.Transform).ToList();
var html = DataRenderer.Render(transformed);
```

## Separation of Concerns

### Do:
- Separate data fetching from transformation
- Separate transformation from rendering/persistence
- Keep each method focused on one responsibility
- Pass data between methods rather than sharing state
- Use static methods for pure transformations

### Don't:
- Mix IO with business logic
- Mix rendering with data transformation
- Create god classes that do everything
- Make everything instance methods when static would work

## Modern C# Features to Embrace

### Records for Immutable Data
```csharp
// GOOD: Use records for immutable data structures
public record Reading(
    DateTime Timestamp,
    double Temperature,
    double Humidity,
    double Rainfall
);

// Transform data with pure functions
public static Reading AdjustTemperature(Reading reading, double offset) {
    return reading with { Temperature = reading.Temperature + offset };
}
```

### Pattern Matching
```csharp
// GOOD: Use pattern matching for clear branching logic
public static string GetRecommendation(RainfallAmount amount) =>
    amount switch {
        RainfallAmount.None => "Water your lawn today",
        RainfallAmount.Light => "Monitor soil moisture",
        RainfallAmount.Moderate => "Skip watering for 2 days",
        RainfallAmount.Heavy => "Skip watering for a week",
        _ => throw new ArgumentException("Unknown rainfall amount")
    };
```

### Constructors and Immutability

**Prefer constructors over default initialization** - If a class doesn't need to be used by reflection-based libraries, use constructors to initialize all required properties.

**Prefer transformation methods over in-place mutation** - Instead of mutating objects with setters, create new instances with transformed values.

**If a method only uses public properties, prefer extension methods over instance methods** - This keeps the class focused and makes transformations composable. Extension method static classes can be in the same file as the class they operate on.

```csharp
// GOOD: Constructor with required properties, extension methods for operations
public class WeatherReading {
    public DateTime Timestamp { get; }
    public double Temperature { get; }
    public double Humidity { get; }

    public WeatherReading(DateTime timestamp, double temperature, double humidity) {
        this.Timestamp = timestamp;
        this.Temperature = temperature;
        this.Humidity = humidity;
    }
}

// Extension methods in the same file
public static class WeatherReadingOps {
    public static WeatherReading WithTemperature(this WeatherReading reading, double newTemperature) {
        return new WeatherReading(reading.Timestamp, newTemperature, reading.Humidity);
    }

    public static WeatherReading WithHumidity(this WeatherReading reading, double newHumidity) {
        return new WeatherReading(reading.Timestamp, reading.Temperature, newHumidity);
    }
}

// Use it:
var reading = new WeatherReading(DateTime.Now, 72.0, 50.0);
var adjusted = reading.WithTemperature(75.0).WithHumidity(55.0); // Composable transformations

// BAD: No constructor, properties not initialized
public class WeatherReading {
    public DateTime Timestamp { get; set; }
    public double Temperature { get; set; }
    public double Humidity { get; set; }
}

// Can create invalid instances
var reading = new WeatherReading(); // All properties are default values
```

**EF Core Entities** - For Entity Framework Core entities, provide a constructor with all required non-nullable properties. Properties can have setters (EF Core needs them), but the constructor ensures required fields are always provided.

*Note: This pattern works with recent versions of EF Core (EF Core 5.0+) which support constructor binding.*

```csharp
// GOOD: EF Core entity with constructor for required properties
public class Place {
    public int Id { get; set; }
    public string Name { get; set; }
    public string Code { get; set; }
    public string Location { get; set; }
    public string? Description { get; set; } // Optional field

    // Constructor ensures required fields are provided
    public Place(int id, string name, string code, string location) {
        this.Id = id;
        this.Name = name;
        this.Code = code;
        this.Location = location;
    }
}


// Use it - constructor ensures Name/Code/Location are never null
var station = new Place(1, "Nice Place", "NP01", "Texas");
```

### Specification Pattern for Testable Predicates

Extract predicate logic into testable specification classes. This makes the filtering logic independently testable and more explicit.

```csharp
// GOOD: Specification class for testable predicate logic
public class MinimumHumiditySpec {
    private readonly double minHumidity;

    public MinimumHumiditySpec(double minHumidity) {
        this.minHumidity = minHumidity;
    }

    public bool IsSatisfiedBy(Reading reading) {
        return reading.Humidity >= this.minHumidity;
    }
}

// Extension methods for simple pure transformations
public static class ReadingOps {
    public static Reading WithCelsius(this Reading reading) {
        var celsius = (reading.Temperature - 32) * 5 / 9;
        return reading with { Temperature = celsius };
    }
}

// Use it - the .Where() stays in the consuming code
var humiditySpec = new MinimumHumiditySpec(50.0);
var processedReadings =
    readings
        .Select(ReadingExtensions.WithCelsius)
        .Where(humiditySpec.IsSatisfiedBy)
        .ToList();

// Easy to unit test the specification independently
[Fact]
public void MinimumHumiditySpec_IsSatisfiedBy() {
    var spec = new MinimumHumiditySpec(50.0);
    var reading = new Reading(DateTime.Now, 72.0, 55.0, 0.0);

    Assert.True(spec.IsSatisfiedBy(reading));
}
```

## Repository Pattern (ASP.NET Core Context)

When working with repositories:
- Keep repository methods focused on data access
- Don't put business logic in repositories
- Return data that business logic can transform

```csharp
// GOOD: Repository returns data
public class ReadingRepository {
    public async Task<IReadOnlyCollection<Reading>> GetReadingsByDateRange(
        DateTime start,
        DateTime end
    ) {
        return await this.context
            .Readings
            .Where(r => r.Timestamp >= start && r.Timestamp <= end)
            .ToListAsync();
    }
}

// GOOD: Business logic in separate service/static class
public static class WeatherAnalysis {
    public static WeatherSummary AnalyzeReadings(List<Reading> readings) {
        var avgTemp = readings.Average(r => r.Temperature);
        var totalRainfall = readings.Sum(r => r.Rainfall);
        return new WeatherSummary(avgTemp, totalRainfall);
    }
}
```

## Parse, Don't Validate

**The Problem with Validation**: When you validate data and return a boolean (or throw an exception), the next function after validation doesn't actually know that the data is valid. It just assumes it, or has to check all over again.

**The Solution - Parse into a Valid Type**: By parsing into a class that can only be constructed with valid data, and having the next caller only accept that validated class type, you naturally lead the programmer to correct usage. They don't need to guess if things are correct - they can see it in the type signature. **The only way to get the valid class is to construct it via a "Parse" or "TryCreate" method.**

### Key Pattern Elements:
- **Private constructor** - The valid type can only be created through the parse/create method
- **Static `TryCreate` method** - Returns both the valid object (if successful) and an error message (if failed)
    - Throwing an exception instead of a (success, error) tuple is fine also, but document the Exception in a doc comment
- **Type safety** - If you have an instance of the valid type, you KNOW it's valid

```csharp
// Raw input from user
public class SignUpRequest {
    public string Name { get; set; }
    public string Email { get; set; }
    public string Street { get; set; }
    public string City { get; set; }
}

// GOOD: Parse into a valid type that includes the domain entity
public class ValidSignUpRequest {
    public User User { get; }

    // Private constructor - can only be created via TryCreate
    private ValidSignUpRequest(User user) {
        this.User = user;
    }

    // Static factory method that validates and parses
    public static (ValidSignUpRequest? Data, string? ErrorMessage) TryCreate(SignUpRequest request) {
        if (string.IsNullOrWhiteSpace(request.Name)) {
            return (null, "Name is required and cannot be empty.");
        }

        if (string.IsNullOrWhiteSpace(request.Email)) {
            return (null, "Email is required and cannot be empty.");
        }

        var (addressComponents, errorMsg) =
            AddressComponents.TryCreate(new AddressValidationRequest {
                Street = request.Street,
                City = request.City
            });

        if (addressComponents == null) {
            return (null, errorMsg);
        }

        // Only create if ALL validation passes - construct the domain entity here
        var user = 
            new User(
                request.Name,
                request.Email,
                addressComponents.Street,
                addressComponents.City);

        return (new ValidSignUpRequest(user), null);
    }
}

// Use it - type system enforces validity
public async Task<IActionResult> SignUp(SignUpRequest request) {
    var (validRequest, errorMessage) = ValidSignUpRequest.TryCreate(request);

    if (validRequest == null) {
        return BadRequest(errorMessage);
    }

    // Repository method signature accepts ValidSignUpRequest - type safety all the way down!
    await this.userRepo.SignUpAsync(validRequest);
    return Ok();
}

// Repository method signature tells you: "I only accept VALID sign up requests"
public class UserRepository {
    public async Task SignUpAsync(ValidSignUpRequest validRequest) {
        // No validation needed - the User was constructed via ValidSignUpRequest.TryCreate
        // The type system guarantees it's valid
        await this.dbContext.Users.AddAsync(validRequest.User);
        await this.dbContext.SaveChangesAsync();
    }
}

// BAD: Traditional validation approach
public IActionResult SignUpBad(SignUpRequest request) {
    if (!ValidateSignUp(request)) {
        return BadRequest("Invalid request");
    }

    // Did we validate? What exactly was validated? Have to assume or check again
    ProcessSignUpBad(request);
    return Ok();
}

private bool ValidateSignUp(SignUpRequest request) {
    return !string.IsNullOrWhiteSpace(request.Name)
        && !string.IsNullOrWhiteSpace(request.Email);
}

// This signature doesn't tell us if the request has been validated
private void ProcessSignUpBad(SignUpRequest request) {
    // Is this valid? We assume it is, but the type doesn't tell us
    // Someone might call this method without validating first
    SaveToDatabase(request.Name, request.Email, /* what about address? */);
}
```

### Benefits:
- **Type safety** - Invalid data cannot exist as the valid type
- **Self-documenting** - Method signatures show exactly what's expected
- **Prevents misuse** - Can't accidentally skip validation
- **Composable** - Valid types can build on other valid types (see `AddressComponents.TryCreate`)
- **Single validation point** - Validation logic lives in one place (the `TryCreate` method)

## Component-Style View Models (MVC Context)

### View Models as Functions of Data
```csharp
// GOOD: Static factory methods that transform data into view models
public record WeatherViewModel(
    string StationName,
    double CurrentTemp,
    double TotalRainfall,
    string Recommendation
);

public static class WeatherViewModelFactory {
    public static WeatherViewModel Create(
        Station station,
        List<Reading> readings
    ) {
        var summary = WeatherAnalysis.AnalyzeReadings(readings);
        var recommendation = GetRecommendation(summary.TotalRainfall);

        return new WeatherViewModel(
            station.Name,
            summary.AverageTemperature,
            summary.TotalRainfall,
            recommendation
        );
    }
}

// In controller:
public IActionResult Index() {
    var station = this.stationRepo.GetStation(1);
    var readings = this.readingRepo.GetRecentReadings(station.Id);
    var viewModel = WeatherViewModelFactory.Create(station, readings);
    return View(viewModel);
}
```

## Practical Guidelines

### Immutability
- Use `record` types for immutable data
- Use `readonly` fields
- Don't force immutability where mutability is more practical (e.g., building large collections)

### Async/Await
- Use `async/await` for IO operations
- Don't use `async void` except for event handlers
- Keep async methods pure when possible (same input = same output, ignoring async timing)
- Name async methods with `Async` suffix

### Null Handling
- Enable nullable reference types (`<Nullable>enable</Nullable>`)
- Use null-coalescing operators: `??`, `?.`
- Use pattern matching for null checks

### Functions Over Classes
- Use static classes for pure utility functions
- Use static methods within classes when state isn't needed
- Use instance methods when behavior depends on instance state
- Use classes for data access, services with dependencies

## Summary

Write C# that is:
1. Uses pure methods with clear inputs and outputs
2. Pragmatic - use the right tool for the job
3. Separates concerns (data access, transformation, presentation)
4. Uses modern C# features (records, pattern matching, expression bodies)
5. Component-style view models (view = f(data))
6. Readable and maintainable above all else