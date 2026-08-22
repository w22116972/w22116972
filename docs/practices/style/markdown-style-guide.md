# Markdown Style Guide

> Source: [Google Markdown style guide](https://google.github.io/styleguide/docguide/style.html)

Much of what makes Markdown refreshing is the ability to write plain text and get great formatted output. Keep Markdown simple and consistent with the rest of the documentation corpus.

The goals are:

1. Keep source text readable and portable.
2. Keep the Markdown corpus maintainable over time and across teams.
3. Keep the syntax simple and easy to remember.

## Minimum Viable Documentation

A small set of fresh and accurate docs is better than a sprawling collection of documentation in various states of disrepair.

- Identify what you really need, such as release docs, API docs, or testing guidelines.
- Delete obsolete content frequently and in small batches.
- Keep documentation up to date with the same care as tests.

## Better Is Better Than Best

Documentation reviews should encourage improvement while allowing authors to iterate quickly.

As a reviewer:

1. Approve immediately when reasonable and trust that comments will be fixed appropriately.
2. Suggest an alternative instead of leaving a vague comment.
3. For substantial changes, create a follow-up change yourself.
4. Block submission only when the change actually makes the documentation worse.

As an author:

1. Avoid spending cycles on trivial arguments.
2. Accept reasonable changes early and continue improving the document.
3. Use the “Better/Best Rule” when deciding whether a change is good enough to submit.

## Capitalization

Use the original names of products, tools, and binaries, preserving their capitalization.

```markdown
# Markdown style guide

`Markdown` is a simple platform for internal engineering documentation.
```

## Document Layout

A document generally benefits from this structure:

```markdown
# Document title

Short introduction.

[TOC]

## Topic

Content.

## See also

- https://link-to-more-info
```

Follow these conventions:

1. Start with one H1 heading, ideally matching the filename.
2. Optionally place the author below the title.
3. Add a short introduction of one to three sentences.
4. Add `[TOC]` after the introduction when the hosting system supports it.
5. Start the remaining headings at H2.
6. Put miscellaneous links under a final `## See also` heading.

## Table of Contents

Use a `[TOC]` directive unless all content fits above the fold on a laptop.

Place `[TOC]` after the introduction and before the first H2 heading. Its position matters for screen readers and keyboard navigation even when the rendered page displays the table of contents elsewhere.

## Character Line Limit

Use an 80-character line limit for Markdown content. This keeps documentation compatible with code-oriented tooling and makes source reviews easier.

Exceptions include:

- Links
- Tables
- Headings
- Code blocks

Wrap text before and after long links where possible.

## Trailing Whitespace

Do not use trailing whitespace. Use a trailing backslash sparingly when a line break is required, but prefer a blank line to create a new paragraph.

## Headings

### Use ATX-Style Headings

Use `#`, `##`, and deeper headings. Avoid underlined headings with `=` or `-`, which are harder to maintain and make heading levels ambiguous.

### Use Unique, Complete Names

Use unique and descriptive names for headings, including subsection headings. Clear headings create intuitive anchor links.

Prefer:

```markdown
## Foo
### Foo summary
### Foo example

## Bar
### Bar summary
### Bar example
```

### Add Spacing

Add a space after the heading markers and blank lines before and after headings.

### Use a Single H1

Use one H1 heading as the document title. All subsequent headings should be H2 or deeper.

### Capitalize Titles and Headers Consistently

Follow the capitalization guidance from the [Google Developer Documentation Style Guide](https://developers.google.com/style).

## Lists

### Use Lazy Numbering for Long Lists

For long or frequently changing lists, use lazy numbering:

```markdown
1.  Foo.
1.  Bar.
    1.  Foofoo.
    1.  Barbar.
1.  Baz.
```

For short, stable lists, fully numbered items are easier to read in source.

### Use Consistent Nested List Spacing

Use a four-space indent for nested numbered and bulleted lists. For small, non-nested, single-line lists, one space after the marker is sufficient.

## Code

### Inline Code

Use backticks for short code quotations, field names, file names, and other literal values:

```markdown
Run `really_cool_script.sh arg`.

Update the `README.md` file.
```

Use inline code for generic file types and specific file names when referring to them literally.

### Use Code Spans for Escaping

Wrap fake paths, example URLs, shell variables, and other text that should not be processed as Markdown in backticks.

### Code Blocks

Use fenced code blocks for code longer than one line:

```python
def foo(bar):
    return bar
```

Always declare the language when possible so syntax highlighting does not need to guess.

Prefer fenced code blocks to indented code blocks because fences make boundaries clear, support language declarations, and are easier to search.

When a command spans multiple lines, escape the newline so readers can copy and paste it:

```shell
$ bazel run :target -- --flag --foo=longvalue \
  --bar=anotherlongvalue
```

Indent code blocks correctly when nesting them inside lists.

## Links

Keep long links out of the surrounding prose where possible.

### Use Explicit Paths

Use explicit paths for links within Markdown:

```markdown
[Other page](/path/to/other/markdown/page.md)
```

### Avoid Cross-Directory Relative Paths

Relative links are safe within the same directory:

```markdown
[Other page](other-page.md)
```

For links to another directory, prefer an explicit path instead of a chain of `../` segments.

### Use Informative Link Titles

Write link text that explains the destination. Avoid link text such as “here,” “link,” or a duplicated URL.

Prefer:

```markdown
See the [Markdown guide](markdown.md) for more information.
```

### Use Reference Links When Helpful

Use reference links when a long destination would make the surrounding Markdown difficult to read or when the same destination appears multiple times.

Define references near their first use, generally before the next heading. References used across multiple sections should be defined at the end of the document.

## Images

Use images sparingly and prefer simple screenshots when an image communicates more clearly than text.

- Add meaningful alternative text.
- Use an image when showing something is easier than describing it.
- Remember that some readers cannot see images and need the surrounding text or alternative text.

## Tables

Use tables for uniform data distributed across two dimensions and for comparisons that readers need to scan quickly.

Prefer lists and subheadings when:

- Many cells are empty.
- The number of columns is large relative to the number of rows.
- Cells contain long prose.
- The information does not need two-dimensional comparison.

Keep table cells concise and use reference links when URLs would make the table difficult to read.

## Prefer Markdown to HTML

Prefer standard Markdown syntax and avoid HTML hacks. Markdown is more readable, portable, and compatible with documentation tools.

If Markdown seems insufficient, reconsider whether the formatting is necessary. Large tables are an exception when Markdown tables are the clearest available format.

## See Also

- [Google Developer Documentation Style Guide](https://developers.google.com/style)
- [Google Markdown style guide](https://google.github.io/styleguide/docguide/style.html)

