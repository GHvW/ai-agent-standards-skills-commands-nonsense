
# Unit Test Standards

This document defines the unit testing standards for the Brazos Valley Water Smart project.

## Test Naming Conventions

Test method names should describe **what is being tested** and **under what conditions**, but NOT the expected outcome or assertion result.

## Summary

Write unit tests that are:
1. **Clearly named** - Describe the scenario, not the outcome
2. **Simple and focused** - One test, one behavior
3. **Easy to read** - Arrange-Act-Assert structure
4. **Maintainable** - Simple mocks over complex frameworks when possible
5. **Valuable** - Test business logic, not framework behavior