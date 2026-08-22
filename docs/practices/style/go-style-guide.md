# Google Go Style Guide

> Source: [Google Go Style Guide](https://google.github.io/styleguide/go/)

The Go Style Guide and its companion documents describe readable and idiomatic Go. The guidance is intended to reduce guesswork, unify review terminology, and help teams write maintainable code.

## Documents

- **Style Guide**: The normative and canonical foundation for Go style.
- **Style Decisions**: Detailed decisions and their reasoning; normative but not canonical.
- **Best Practices**: Patterns that solve common problems; useful but not normative.

## Goals

The documents aim to:

- Agree on principles for weighing alternative styles.
- Codify settled Go style.
- Provide canonical examples of Go idioms.
- Explain the trade-offs behind style decisions.
- Reduce surprises in readability reviews.
- Help reviewers use consistent terminology.

They do not aim to:

- Enumerate every comment that may occur in a readability review.
- List every rule developers must memorize.
- Replace judgment when using language features.
- Justify large-scale rewrites solely to eliminate style differences.

## Core Principles

### Consistency

Uniformity has value, even when individual preferences differ. Make style improvements when useful, but avoid unnecessary churn in existing code. Apply current practices to new code and address nearby issues over time.

### Readability

Prefer code that is easy for another Go developer to understand. The best choice is often the most familiar and idiomatic one, not the shortest possible expression.

### Formatting

Use gofmt and keep formatting consistent with the project. Automated formatting reduces subjective arguments and lets reviews focus on behavior.

### Canonical

A canonical rule is prescriptive and intended to remain stable. It describes a standard that old and new code should follow.

### Normative

A normative rule establishes consistency for reviewers. Normative guidance may change as the language, libraries, and evidence evolve.

### Idiomatic

An idiomatic pattern is common, familiar, and easy to recognize. Prefer it when it serves the purpose well.

## Review Guidance

Style issues are subjective and involve trade-offs. Follow the style guide even when you would personally choose another style, but do not nit-pick every historical violation or create unrelated formatting churn.

The guide assumes familiarity with [Effective Go](https://go.dev/doc/effective_go).

## Additional References

- [Go Language Specification](https://go.dev/ref/spec)
- [Go FAQ](https://go.dev/doc/faq)
- [Go Memory Model](https://go.dev/ref/mem)
- [Go Data Structures](https://research.swtch.com/godata)
- [Go Interfaces](https://research.swtch.com/interfaces)
- [Go Proverbs](https://go-proverbs.github.io/)

