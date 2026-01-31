# JavaScript Coding Standards

This document defines the JavaScript coding standards for the Brazos Valley Water Smart project.

## Core Philosophy

Write code as a **pragmatic programmer**. Use language features that make code clearer and more maintainable. Prefer patterns where they fit naturally with JavaScript's language constructs, but don't force them.

## Pure Functions

### Prefer Pure Functions
- **Functions should take inputs and return outputs**
- Avoid side effects within functions unless necessary (e.g., IO operations)
- Make the return value of one function be the input to another
- Pure functions are easier to test, reason about, and reuse

```javascript
// GOOD: Pure function
function calculateTotal(items) {
  return items.reduce((sum, item) => sum + item.price, 0);
}

// BAD: Impure function with side effects
let total = 0;
function addToTotal(item) {
  total += item.price; // Modifies external state
}
```

## Array Methods

### Use map/filter/reduce When They're Clear
- **Use `Array.prototype.map()`, `filter()`, and `reduce()`** when they make code clearer
- These methods work well with pure functions
- Example: `const doubled = numbers.map(n => n * 2)`

### When NOT to Force These Patterns
- **Don't force collection construction into a `reduce()`** - use a `for` loop instead
- **Don't wrap IO operations in `.forEach()`** unless you already have a function that fits
- For loops aren't the devil if you don't abuse them
- If a for loop is clearer and more performant, use it

### Example: When to Use For Loop vs Reduce
```javascript
// BAD: Forcing reduce where it doesn't fit
const result = items.reduce((acc, item) => {
  if (item.isValid) {
    acc.validItems.push(item);
  } else {
    acc.invalidItems.push(item);
  }
  return acc;
}, { validItems: [], invalidItems: [] });

// GOOD: Use a for loop when building complex collections
const validItems = [];
const invalidItems = [];
for (const item of items) {
  if (item.isValid) {
    validItems.push(item);
  } else {
    invalidItems.push(item);
  }
}
```

## Anti-Patterns to Avoid

### Stateful Objects with Side-Effect Methods
**ABHOR** objects that have all of the following:
- Have private methods for transformations
- Have public methods for data fetching
- Have public methods for "rendering"
- All of which take no arguments and return nothing
- They only update "state" in an object
- This pattern prevents separation of concerns

```javascript
// BAD: Stateful object with side-effect methods
class DataManager {
  constructor() {
    this.data = null;
    this.transformedData = null;
  }

  async fetchData() {
    this.data = await fetch('/api/data').then(r => r.json());
  }

  transformData() {
    this.transformedData = this.data.map(/* transform */);
  }

  render() {
    document.getElementById('container').innerHTML = /* render this.transformedData */;
  }
}

// GOOD: Separate pure functions
async function fetchUser() {
  const response = await fetch('/api/user/123');
  return response.json();
}

/**
 * @typedef {Object} ApiUser
 * @property {string} first_name - User's first name
 * @property {string} last_name - User's last name
 * @property {string} email - User's email
 * @property {string} created_at - ISO timestamp of account creation
 */

/**
 * @typedef {Object} ViewUser
 * @property {string} fullName - Combined first and last name
 * @property {string} email - User's email
 * @property {string} memberSince - Formatted creation date
 */

/**
 * Transform raw API user data into view-ready format
 * @param {ApiUser} apiUser - Raw user from API
 * @returns {ViewUser} Transformed user data
 */
function transformUser(apiUser) {
  return {
    fullName: `${apiUser.first_name} ${apiUser.last_name}`,
    email: apiUser.email,
    memberSince: new Date(apiUser.created_at).toLocaleDateString()
  };
}

/**
 * Create a user card DOM element
 * @param {ViewUser} user - Transformed user data
 * @returns {HTMLDivElement} The user card element
 */
function UserCard(user) {
  const card = document.createElement("div");
  card.className = "card";
  card.innerHTML = `
    <h3>${user.fullName}</h3>
    <p>${user.email}</p>
    <small>Member since ${user.memberSince}</small>
  `;
  return card;
}

function render(container, data) {
  container.appendChild(data);
}

// Use it:
const apiUser = await fetchUser();
const user = transformUser(apiUser);
const userCard = UserCard(user);
render(document.getElementById('container'), userCard);
```

## Component-Style Architecture

### View as a Function of State
Even without React.js, follow the React philosophy:
- **Pass properties to a function**
- **The function produces an HTML element**
- **View = f(state)**

```javascript
// GOOD: Component-style function
function UserCard({ name, email, avatarUrl }) {
  const card = document.createElement('div');
  card.className = 'card';
  card.innerHTML = `
    <div class="card-image">
      <img src="${avatarUrl}" alt="${name}">
    </div>
    <div class="card-content">
      <h3>${name}</h3>
      <p>${email}</p>
    </div>
  `;
  return card;
}

// Use it:
const userElement = UserCard({
  name: 'John Doe',
  email: 'john@example.com',
  avatarUrl: '/images/john.jpg'
});
container.appendChild(userElement);
```

## DOM Manipulation Patterns

When working with vanilla JavaScript and the DOM, follow these patterns for testability, clarity, and maintainability.

### Component Functions Return HTMLElements

Component functions should work like React components - **pass a properties object, get back an HTMLElement**. This makes testing trivial and usage clear.

```javascript
// GOOD: Component returns HTMLElement
/**
 * @typedef {Object} WeatherCardProps
 * @property {number} temperature - Temperature in Fahrenheit
 * @property {string} condition - Weather condition description
 * @property {string} location - Location name
 */

/**
 * Create a weather card component
 * @param {WeatherCardProps} props - Component properties
 * @returns {HTMLDivElement} Weather card element
 */
function WeatherCard({ temperature, condition, location }) {
  const card = document.createElement('div');
  card.className = 'weather-card';
  card.innerHTML = `
    <h3>${location}</h3>
    <div class="temp">${temperature}°F</div>
    <div class="condition">${condition}</div>
  `;
  return card;
}

// Easy to test - no DOM dependencies
const element = WeatherCard({
  temperature: 72,
  condition: 'Sunny',
  location: 'Bryan'
});
// Assert on element.innerHTML, element.className, etc.

// BAD: Function takes ID, does lookup, creates element, mutates DOM, returns nothing
function renderWeatherCard(containerId, temperature, condition, location) {
  const container = document.getElementById(containerId); // Lookup inside function
  const card = document.createElement('div');
  card.className = 'weather-card';
  card.innerHTML = `
    <h3>${location}</h3>
    <div class="temp">${temperature}°F</div>
    <div class="condition">${condition}</div>
  `;
  container.appendChild(card); // Side effect without return
}

// Usage - unclear what this does without reading the function
renderWeatherCard('weather-container', 72, 'Sunny', 'Bryan');
// Can't test without a real DOM
// Can't reuse the element
// Can't compose with other functions
```

### Pass HTMLElements, Not IDs

When performing DOM side effects, **pass the actual HTMLElement to the function, not an ID string**. This avoids multiple lookups in the same call stack and makes dependencies explicit.

```javascript
// GOOD (Preferred): Create element inline, pass both elements to update function
function replaceContent(container, newElement) {
  container.innerHTML = '';
  container.appendChild(newElement);
}

// Usage - maximally compositional, no intermediate variables
const container = document.getElementById('weather-container');
replaceContent(
  container,
  WeatherCard({ temperature: 72, condition: 'Sunny', location: 'Bryan' })
);

// Also GOOD: Pass container element and data
function updateWeatherDisplay(container, weatherData) {
  const card = WeatherCard(weatherData);
  container.innerHTML = '';
  container.appendChild(card);
}

// Usage - clear that we're working with the container element
const container = document.getElementById('weather-container');
updateWeatherDisplay(container, { temperature: 72, condition: 'Sunny', location: 'Bryan' });

// BAD: Pass ID string - causes repeated lookups
function updateWeatherDisplay(containerId, weatherData) {
  const container = document.getElementById(containerId); // Lookup #1
  const card = WeatherCard(weatherData);
  container.innerHTML = '';

  const containerAgain = document.getElementById(containerId); // Lookup #2 (unnecessary!)
  containerAgain.appendChild(card);
}

// Usage - not clear what element is being manipulated without reading the function
updateWeatherDisplay('weather-container', { temperature: 72, condition: 'Sunny', location: 'Bryan' });
```

### When to Use ID/Class Lookups

**Element lookups by ID or class should happen in the `.cshtml` file** (when possible and practical), not in separate JavaScript files. This makes it easy for developers to see what elements exist on the page without hunting through multiple files.

```javascript
// In Views/Home/Index.cshtml:
@section Scripts {
    <script type="module">
        import { initializeWeatherWidget } from "/js/weather-widget.js";
        import { replaceContent } from "/js/dom-utils.js";

        // Element lookups happen HERE in the view
        const weatherContainer = document.getElementById('weather-container');
        const forecastContainer = document.getElementById('forecast-container');

        // Pass the elements to initialization functions
        initializeWeatherWidget(weatherContainer, {
            city: 'Bryan',
            state: 'TX',
            lat: 30.6744,
            lon: -96.3700
        });

        // Later updates also pass elements
        async function refreshWeather() {
            const data = await fetchWeatherData();

            replaceContent(weatherContainer, WeatherCard(data.current));
            replaceContent(forecastContainer, ForecastCard(data.forecast));
        }

        setInterval(refreshWeather, 600000);
    </script>
}
```

**Benefits of this approach:**
- **Co-location**: Element IDs/classes are right next to the HTML that defines them
- **Discoverability**: Developers can easily see what elements are being manipulated
- **Fewer lookups**: Elements are looked up once and reused
- **Testability**: JavaScript functions accept elements, making them easier to test

**When lookups in JS files are okay:**
- Dynamic element creation where you create the parent element
- Event delegation where you query within a known parent element
- Complex DOM traversal that would clutter the cshtml file

```javascript
// Acceptable: Lookups within a component you created
function WeatherWidget(config) {
  const widget = document.createElement('div');
  widget.className = 'weather-widget';
  widget.innerHTML = `
    <div class="weather-current"></div>
    <div class="weather-forecast"></div>
  `;

  // These elements are internal to the component - lookup is fine
  const currentDiv = widget.querySelector('.weather-current');
  const forecastDiv = widget.querySelector('.weather-forecast');

  updateCurrentWeather(currentDiv, config.current);
  updateForecast(forecastDiv, config.forecast);

  return widget;
}
```

## Separation of Concerns

### Do:
- Separate data fetching from transformation
- Separate transformation from rendering
- Keep each function focused on one responsibility
- Pass data between functions rather than sharing state

### Don't:
- Mix IO with business logic
- Mix rendering with data transformation
- Create god objects that do everything

## Practical Guidelines

### Immutability
- JavaScript collections aren't immutable by default
- Use spread operator for shallow copies: `const newArr = [...oldArr, newItem]`
- Use `Object.freeze()` when you need immutability guarantees
- Don't force immutability where mutability is more practical

### Async/Await
- Prefer `async/await` over raw promises for readability
- Handle errors with try/catch
- Keep async functions pure when possible (same input = same output, ignoring async timing)

### Functions Over Classes
- Prefer functions and closures over classes
- Use classes only when they provide clear benefit (e.g., extending built-in types)

#### Pure Functions Don't Belong as Class Methods

**If a function doesn't use `this`, it should NOT be a class method.** It should be a standalone function.

```javascript
// BAD: Pure function trapped as a method
class WeatherWidget {
  constructor(api) {
    this.api = api;
  }

  // This method doesn't use `this` - it shouldn't be here!
  celsiusToFahrenheit(celsius) {
    return (celsius * 9/5) + 32;
  }

  async updateTemperature() {
    const data = await this.api.fetchWeather(); // Uses this.api
    const tempF = this.celsiusToFahrenheit(data.temperatureC); // Calls pure method
    return tempF;
  }
}

// GOOD: Pure function as standalone export
export function celsiusToFahrenheit(celsius) {
  return (celsius * 9/5) + 32;
}

class WeatherWidget {
  constructor(api) {
    this.api = api;
  }

  async updateTemperature() {
    const data = await this.api.fetchWeather(); // Uses this.api
    const tempF = celsiusToFahrenheit(data.temperatureC); // Calls standalone function
    return tempF;
  }
}
```

**Why this matters:**
- Pure functions trapped in classes are harder to test (you must instantiate the class)
- They can't be imported and reused independently elsewhere in your codebase
- They falsely signal that they depend on instance state
- They create unnecessary coupling

**When methods ARE appropriate:**
- When the function genuinely needs access to instance state (`this.property`)
- When the method modifies instance state
- When the behavior is specific to that class instance

### Documentation with JSDoc
These should only be applied when TypeScript is not being used

- **Use JSDoc comments where they add value** - especially for public APIs, complex functions, and library code
- **Use TypeScript types in JSDoc** when possible (`@template`, `@param {Type}`, `@returns {Type}`)
- **Don't over-document** - clear function names and code are better than redundant comments
- JSDoc is particularly useful for:
  - Functions with generic/template types
  - Functions that transform data (document input and output types)
  - Public APIs that other developers will use
  - Complex business logic that benefits from explanation

```javascript
// GOOD: Define reusable types with @typedef
/**
 * @typedef {Object} ApiUser
 * @property {string} first_name - User's first name
 * @property {string} last_name - User's last name
 * @property {string} email - User's email
 * @property {string} created_at - ISO timestamp of account creation
 */

/**
 * @typedef {Object} ViewUser
 * @property {string} fullName - Combined first and last name
 * @property {string} email - User's email
 * @property {string} memberSince - Formatted creation date
 */

// GOOD: Use typedefs in function signatures
/**
 * Transform raw API user data into view-ready format
 * @param {ApiUser} apiUser - Raw user from API
 * @returns {ViewUser} Transformed user data
 */
function transformUser(apiUser) {
  return {
    fullName: `${apiUser.first_name} ${apiUser.last_name}`,
    email: apiUser.email,
    memberSince: new Date(apiUser.created_at).toLocaleDateString()
  };
}

/**
 * Create a user card DOM element
 * @param {ViewUser} user - Transformed user data
 * @returns {HTMLDivElement} The user card element
 */
function UserCard(user) {
  const card = document.createElement("div");
  card.className = "card";
  card.innerHTML = `
    <h3>${user.fullName}</h3>
    <p>${user.email}</p>
    <small>Member since ${user.memberSince}</small>
  `;
  return card;
}

// DON'T: Over-document obvious code
/**
 * Adds two numbers together
 * @param {number} a - First number
 * @param {number} b - Second number
 * @returns {number} The sum
 */
function add(a, b) {
  return a + b; // This is obvious, JSDoc adds no value
}
```

## Summary

Write JavaScript that is:
1. Uses pure functions with clear inputs and outputs
2. Pragmatic - use the right tool for the job
3. Component-style (view as function of state)
4. Separates concerns (fetch, transform, render)
5. Passes HTMLElements, not ID strings, to functions that manipulate the DOM
6. Performs element lookups in .cshtml files when possible
7. Readable and maintainable above all else
