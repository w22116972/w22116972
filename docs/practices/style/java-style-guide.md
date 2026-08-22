# Google Java Style Guide

> Source: [Google Java Style Guide](https://google.github.io/styleguide/javaguide.html)

This guide defines Google's coding standards for Java source code. It focuses on rules that can be enforced consistently by people or tools.

## Introduction

- A class includes normal classes, records, enums, interfaces, and annotation types.
- A member includes nested classes, fields, methods, and constructors.
- Comments means implementation comments; documentation comments are called Javadoc.
- Examples are illustrative and should not be treated as the only valid formatting.

## Source File Basics

### File Name

A Java source file containing classes uses the case-sensitive name of its single top-level class plus the .java extension.

### Encoding and Characters

- Use UTF-8.
- Use ASCII spaces for horizontal whitespace.
- Do not use tabs for indentation.
- Prefer special escape sequences when they exist.
- Use actual Unicode characters or Unicode escapes based on readability.

## Source File Structure

An ordinary source file contains these sections, in order:

1. License or copyright information, if present.
2. Package declaration.
3. Imports.
4. Exactly one top-level class declaration.

Separate present sections with one blank line.

### Package Declarations

Every source file must have a package declaration except for files such as module-info.java that use a different syntax. Package declarations are not line-wrapped.

### Imports

- Do not use wildcard imports.
- Do not use module imports.
- Do not line-wrap imports.
- Put static imports in one group and non-static imports in a separate group.
- Sort imported names in ASCII order within each group.
- Do not use static imports for nested classes.

### Class Declarations

Each top-level class belongs in its own source file. Keep class members in a logical, explainable order. Keep overloaded methods and constructors together in one contiguous group.

## Formatting

### Braces

Use braces with if, else, for, do, and while, even for a single statement.

Use K&R style for non-empty blocks:

- No line break before the opening brace.
- A line break after the opening brace.
- A line break before the closing brace.
- A line break after the closing brace when it terminates a statement or declaration.

### Indentation

Indent each new block by two spaces. Apply the same indentation to code and comments.

### Statements

Put one statement on each line.

### Column Limit

Limit Java code to 100 characters. Exceptions include package declarations, imports, text blocks, copyable shell commands in comments, and unavoidable long identifiers or URLs.

### Line Wrapping

Prefer breaking at a higher syntactic level. Indent continuation lines by at least four spaces. Keep method names attached to their opening parentheses and commas attached to the preceding token.

### Whitespace

- Use a single blank line between class members when appropriate.
- Use spaces around binary and ternary operators.
- Put a space after commas, semicolons, and casts.
- Put a space before an opening brace.
- Do not use variable-width spaces to preserve horizontal alignment.

### Parentheses

Use grouping parentheses when they improve clarity or prevent reasonable readers from misinterpreting operator precedence.

## Specific Constructs

- Declare one variable per declaration.
- Declare local variables close to first use.
- Do not use C-style array declarations such as String args[]; use String[] args.
- Keep enum formatting consistent with class formatting.
- Use braces and spacing consistently in switch statements and expressions.
- Use annotations in the prescribed order and formatting.

## Naming

- Use UpperCamelCase for classes, interfaces, enums, annotations, and records.
- Use lowerCamelCase for methods, fields, parameters, and local variables.
- Use CONSTANT_CASE for constants.
- Use descriptive names rather than abbreviations.
- Use singular names for enum types and plural names for collections when appropriate.

## Programming Practices

- Mark overridden methods with @Override.
- Avoid catching exceptions without handling them meaningfully.
- Do not ignore exceptions silently.
- Keep deprecated code and APIs clearly annotated.
- Make visibility as narrow as practical.
- Prefer immutable objects where practical.
- Avoid unnecessary inheritance and hidden side effects.
- Keep Javadocs accurate, concise, and useful to callers.

## Formatting Tools

Use [google-java-format](https://github.com/google/google-java-format) where the project supports it.

