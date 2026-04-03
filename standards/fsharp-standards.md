# F# Coding Standards

## Modules
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


## Type signatures
Use angle bracket style rather than OCaml style for types:
`Option<Person>` rather than `Person option`
`list<Person>` rather than `Person list`
etc.

For generics, use capital letters istead of lowercase letters
```fsharp
let myfunc (a: 'A) = // ...

// instead of
let myfunc (a: 'a) = // ...
```


If you use newlines for formatting an F# function format it like this:
```fsharp
let myfunc
    (a: A')
    (b: B')
    (c: C')
 : D' =
    // implementaiton ....
```
most F# examples will show the return type with the same spacing as the arguments. I do not like that. Until the F# parser doesn't allow it, a new line with a space followed by the `:` and return type followed by the `=` is what I like to have.