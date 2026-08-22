# Google Technical Writing

This article provides a condensed version of the reference material and is tailored for individuals with busy schedules. To understand the reasons behind each guideline, please refer to the reference above.

The guidelines are categorized into three topics:

1. The choice of words
2. Sentence rewriting
3. The structure of paragraphs

## The Choice of Words

### Choose Strong Verbs

Reduce imprecise, weak, or generic verbs such as the following:

- Forms of `be`: `is`, `are`, `am`, `was`, and `were` (not always bad)
- `occur`
- `happen`

```text
(X) The exception occurs/happens when dividing by zero...
(O) Dividing by zero raises the exception.
```

```text
(X) We are very careful to ensure...
(O) We carefully ensure...
```

```text
(X) When a variable declaration doesn't have a datatype, a compiler error happens.
(O) If you declare a variable but don't specify a datatype, the compiler generates an error message.
```

### Start Numbered List Items with Imperative Verbs

Start numbered list items with imperative verbs:

1. Download the XXX app from Google Play or iTunes.
2. Configure the XXX app's settings.
3. Start the XXX app.

### Prefer Active Voice to Passive Voice

Prefer active voice to make clear who is performing the action.

### Keep Writing Culturally Neutral

Target international audiences and avoid idioms that may be difficult to understand:

```text
(X) Bob's your uncle.
(O) This task is done.
```

```text
(X) XXX is a sticky wicket.
(O) XXX is a challenging problem.
```

## Sentence Rewriting

### Convert Some Long Sentences to Lists

Use:

- Bulleted lists for unordered items
- Numbered lists for ordered items

When you see the conjunction `or` in a long sentence, consider refactoring the sentence into a bulleted list.

```text
(X) To alter the usual flow of a loop, you may use either a break statement (which hops you out of the current loop) or a continue statement (which skips past the remainder of the current iteration of the current loop).
```

```text
(O) To alter the usual flow of a loop, call one of the following statements:
- break, which hops you out of the current loop.
- continue, which skips past the remainder of the current iteration of the current loop.
```

### Minimize Certain Adjectives and Adverbs

Prefer precise, measurable descriptions:

```text
(X) Setting this flag makes the application run screamingly fast.
(O) Setting this flag makes the application run 225–250% faster.
```

### Reduce `there is` and `there are`

```text
(X) There is no creator stack for the main thread.
(O) The main thread does not provide a creator stack.
```

```text
(X) There is a variable called `met_trick` that stores the current accuracy.
(O) The variable `met_trick` stores the current accuracy.
```

```text
(X) There is no guarantee that the updates will be received in sequential order.
(O) Clients might not receive the updates in sequential order.
```

### Reduce Extraneous Words

Prefer concise alternatives:

| Avoid | Prefer |
| --- | --- |
| at this point in time | now |
| is able to | can |
| determine the location of | find |

### Place Conditions Before Instructions

Place conditions before instructions:

```text
(X) Use xxx if ...
(O) If xxx, use ....
```

```text
(X) Click Delete if you want to delete the entire document.
(O) To delete the entire document, click Delete.
```

## The Structure of Paragraphs

### Write an Effective Opening Sentence

The opening sentence should state the theme of the paragraph.

```text
(X) A block of code is any set of contiguous code within the same function. For example, suppose you wrote a block of code that detected whether an input line ended with a period. To evaluate a million input lines, create a loop that runs a million times.
```

```text
(O) A loop runs the same block of code multiple times. For example, suppose you wrote a block of code that detected whether an input line ended with a period. To evaluate a million input lines, create a loop that runs a million times.
```

### Focus Each Paragraph on a Single Topic

Focus each paragraph on a single topic and each sentence on a single idea. Reduce subordinate clauses when they do not extend the main idea but instead branch into a separate idea.

```text
(X) The late 1950s was a key era for programming languages because IBM introduced Fortran in 1957 and John McCarthy introduced Lisp the following year, which gave programmers both an iterative way of solving problems and a recursive way.
```

```text
(O) The late 1950s was a key era for programming languages. IBM introduced Fortran in 1957. John McCarthy invented Lisp the following year. Consequently, by the late 1950s, programmers could solve problems iteratively or recursively.
```

### Answer What, Why, and How

Good paragraphs answer these three questions:

1. What are you trying to tell your reader?
2. Why is it important for the reader to know this?
3. How should the reader use this knowledge? Alternatively, how should the reader know if your point is true?

