# Testing Instructions

Apply these instructions when creating or modifying tests.

## Frameworks and conventions

1. Follow the style of existing tests in the project.
2. Use JUnit 5, Mockito, and existing project testing utilities.
3. Use `@Mock` or `@MockBean` where appropriate.
4. Do not use `mock()` or `assertThat`.
5. Use `whenever` to configure mock behavior.
6. Use `verify` to check mock interactions.
7. Use `assertAll` when a test contains more than one assertion or verification. Place relevant `verify` calls inside `assertAll`.
8. Use `assertTrue` and `assertFalse` for boolean assertions instead of `assertEquals`.

## Test data

9. Move values reused across multiple test methods into constants in the test class companion object when appropriate.
10. Reuse values from initialized objects instead of duplicating literals. Changing a test value should require modifying it in one place.
11. Use named arguments for Kotlin function calls with more than two parameters, where supported.

## Mockito

12. Never mix argument matchers such as `any()` with raw concrete arguments in the same `whenever` or `verify` invocation. Use consistent argument matching.

## Formatting

13. Keep each `assert`, `verify`, and similar invocation inside `assertAll` on a single line.
14. Avoid excessive line wrapping in methods, constructors, assertions, `verify`, and `whenever`.
15. Do not add redundant comments such as `Preparation`, `Action`, or `Assert`.

## Test design

16. Cover relevant positive, negative, and boundary scenarios.
17. Prefer testing observable behavior through existing public methods rather than private implementation details.
18. Reuse existing test fixtures, builders, and initialization helpers.
19. Do not invent missing DTOs, methods, or dependencies. Inspect the repository first and request missing information if necessary.
20. Do not modify production logic solely to make a test pass without identifying the underlying issue.
21. Run the relevant tests when possible and report the actual results.
