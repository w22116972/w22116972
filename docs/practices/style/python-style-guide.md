# Google Python Style Guide

> Source: [Google Python Style Guide](https://google.github.io/styleguide/pyguide.html)

This guide describes recommended practices for readable Python programs at Google. Use automated formatting and linting where possible, while keeping readability as the primary goal.

## Language Rules

### Lint

Run pylint over Python code and suppress warnings only when they are inappropriate. Add an explanation when the reason is not obvious.

### Imports

- Import packages and modules rather than individual types, classes, or functions.
- Prefer full package names.
- Avoid relative imports.
- Use aliases only for standard abbreviations or to resolve a clear naming conflict.
- Keep imports explicit and easy to identify.

### Packages

Import modules using their full package path. Do not rely on the directory containing the main binary being present in sys.path.

### Exceptions

- Prefer built-in exception classes when appropriate.
- Do not use assert for application logic or precondition validation.
- Do not catch all exceptions unless re-raising them or intentionally creating an isolation point.
- Keep try blocks small.
- Use finally for cleanup.
- Custom exception names should end in Error and inherit from an existing exception class.

### Mutable Global State

Avoid mutable global state. If it is necessary:

- Keep it internal with a leading underscore.
- Expose access through functions or class methods.
- Explain why the design requires it.
- Use uppercase names for module-level constants.

### Nested Functions and Classes

Nested functions and classes are acceptable when they close over a local value other than self or cls. Do not nest code merely to hide it from users; use a leading underscore at module scope instead.

### Comprehensions and Generator Expressions

Use comprehensions and generator expressions for simple cases. Optimize for readability rather than conciseness.

### Iterators and Operators

Prefer default iterators and operators for lists, dictionaries, and files. Do not mutate a container while iterating over it.

### Generators

Use generators when they simplify stateful iteration or avoid creating a complete list in memory. Use “Yields:” rather than “Returns:” in generator docstrings and clean up expensive resources explicitly.

### Lambda Functions

Use lambdas for short, simple expressions. Prefer a regular function when the expression is longer than approximately 60–80 characters or spans multiple lines.

### Conditional Expressions

Use conditional expressions only for simple cases. Use a complete if statement when the expression is complex.

### Default Arguments

Do not use mutable objects as default argument values:

    def foo(items=None):
        if items is None:
            items = []

### Properties

Use properties only when access remains cheap, straightforward, and unsurprising. Do not hide expensive work or significant side effects behind normal attribute access.

### Threading

Do not rely on the atomicity of built-in types. Use synchronization primitives when correctness depends on concurrent access.

### Power Features

Avoid metaclasses and other advanced features unless necessary. Use them only when the benefits clearly outweigh the complexity.

### Modern Python

Use modern Python features when they improve clarity and are supported by the project. Keep from __future__ imports at the beginning of the file after the module docstring and comments.

### Type-Annotated Code

Use type annotations to improve readability and static analysis. Keep annotations accurate, simple, and consistent with the surrounding code.

## Style Rules

### Semicolons

Do not use semicolons to put multiple statements on one line.

### Line Length

Limit lines to 80 characters. Exceptions include long URLs, comments containing long commands, and unavoidable long strings.

### Indentation and Whitespace

- Use four spaces per indentation level.
- Do not use tabs.
- Use two blank lines around top-level definitions.
- Use one blank line around method definitions.
- Avoid trailing whitespace.
- Put one space around binary operators and after commas.
- Do not use extra spaces inside parentheses, brackets, or braces.

### Comments and Docstrings

Write complete sentences in comments and docstrings. Explain intent and non-obvious behavior rather than repeating what the code already says.

Use docstrings for public modules, functions, classes, and methods. Document arguments, return values, and raised exceptions when useful.

### Strings

Use the project's established quote convention consistently. Use triple-quoted strings for multiline text and docstrings.

For logging, use the logging framework's formatting arguments instead of interpolating values before the call.

### Files and Stateful Resources

Use context managers for files, sockets, and other stateful resources.

### Naming

- module_name, package_name, function_name, method_name, and local_variable
- ClassName
- CONSTANT_NAME
- _internal_name
- Avoid single-character names except for conventional local variables.
- Avoid names that shadow built-ins or use confusing abbreviations.

### Main

Put executable program logic behind a main() function and use the standard __name__ == '__main__' guard.

### Function Length

Keep functions focused. If a function is difficult to understand or test, consider extracting smaller functions.

### Type Annotations

Use annotations consistently. Prefer explicit, useful types, and avoid suppressing type-checking errors without a clear reason.

## Parting Words

Write Python code for the next reader. Consistency, clarity, and maintainability are more valuable than cleverness.

