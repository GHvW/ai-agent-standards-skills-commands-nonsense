# Entity Framework Core Standards

## Entity Construction

### Parameterless Constructors

Parameterless constructors are **not required** in later versions of Entity Framework Core. EF Core supports constructor binding and can materialize entities using constructors with parameters.

**Do not add parameterless constructors** if the version of EF Core being used does not require them.

❌ **Incorrect**:
```csharp
public class Customer
{
    public int Id { get; set; }
    public string Name { get; set; }

    public Customer(int id, string name)
    {
        this.Id = id;
        this.Name = name;
    }

    // NOT NEEDED
    private Customer() { }
}
```

✅ **Correct**:
```csharp
public class Customer
{
    public int Id { get; set; }
    public string Name { get; set; }

    public Customer(int id, string name)
    {
        this.Id = id;
        this.Name = name;
    }
}
```