# TypeScript Coding Standards

All JavaScript Coding Standards (found in [javascript-standards](./javascript-standards.md)) apply to TypeScript unless explicitely mentioned.

## Philosophy

Write TypeScript types that make it so the JavaScript you would normally write will typecheck properly


## Use interfaces or types to represent data objects

```typescript
// GOOD
interface Vec2 {
    x: number;
    y: number;
}

type Vec2 = {
    x: number;
    y: number
}

// BAD
class Vec2 {
    x: number;
    y: number;

    constructor(x: number, y: number) {
        this.x = x;
        this.y = y;
    }
}
```

Using interfaces and types makes using object literals (a common JS pattern) easy.

Given this JavaScript:
```javascript
function distance(from: Vec2, to: Vec2): number {
    return Math.sqrt(
        Math.pow(to.x - from.x, 2) + Math.pow(to.y - from.y, 2)
    );
}
```

You can easily convert it to TypeScript by just adding types to the function signature.
```typescript
function distance(from: Vec2, to: Vec2): number {
    return Math.sqrt(
        Math.pow(to.x - from.x, 2) + Math.pow(to.y - from.y, 2)
    );
}
```
If you use classes to define data objects, you have to change your traditional JavaScript to look like this:
```typescript
function scale(by: number, vec: Vec2): Vec2 {
    return new Vec2(  // new initialization required now
        x: vec.x * by, 
        y: vec.y * by 
    );
}
```

Note: since TS is structurally typed, any class with an `x` and `y` would satisfy `Vec2` if `Vec2` is defined as a type or interface:
```
class Location {
    x: number;
    y: number;
    address

    constructor(x: number, y: number) {
        this.x = x;
        this.y = y;
    }
}
```

Using classes is fine when you need methods, but if you're just creating a data object, make it an `interface` or `type`