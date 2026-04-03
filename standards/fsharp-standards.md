# F# Coding Standards

When creating new types and modules, where possible, declare the type outside the module and then declare a module for the functions that work on that type.

Example:
```fsharp
namespace Example

// declare the type outside the module
type Person = { First: string; Last: string }

// declare a module for functions that work on the type
module Person =

    // constructor
    let create first last =
        { First = first; Last = last }

    // method that works on the type
    let fullName { First = first; Last = last } =
        first + " " + last
```