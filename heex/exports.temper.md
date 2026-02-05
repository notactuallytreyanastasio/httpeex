# HEEx Public API

This module defines the public API surface for the HEEx parser library.
It re-exports the core functions and types that consumers should use.

## Module Dependencies

This file uses types and functions from ast.temper.md, tokenizer.temper.md,
parser.temper.md, and renderer.temper.md.
All files in this directory are automatically combined into the "heex" module.

## Core Parsing Functions

The primary entry points for parsing HEEx templates.

### parse

Parse a HEEx template string into a Document AST.

```temper
export let parseTemplate(input: String): Document throws Bubble {
  parse(input)
}
```

### tokenize

Tokenize a HEEx template string into a list of tokens.
Useful for syntax highlighting or custom processing.

```temper
export let tokenizeTemplate(input: String): List<Token> throws Bubble {
  tokenize(input)
}
```

## Rendering Functions

Convert a Document AST back to various output formats.

### renderToHtml

Render the AST to an HTML string. Expressions are preserved as `{expr}` placeholders.

```temper
export let renderToHtml(doc: Document): String {
  renderHtml(doc)
}
```

### renderToDebug

Render the AST to a debug string showing the tree structure.
Useful for development and debugging.

```temper
export let renderToDebug(doc: Document): String {
  renderDebug(doc)
}
```

### renderToJson

Serialize the AST to JSON format for tooling integration.

```temper
export let renderToJson(doc: Document): String {
  renderJson(doc)
}
```

## Convenience Functions

Combined operations for common use cases.

### parseAndRender

Parse a template and immediately render it back to HTML.
Useful for validating templates or normalizing whitespace.

```temper
export let parseAndRender(input: String): String throws Bubble {
  let doc = parse(input);
  renderHtml(doc)
}
```

### parseAndValidate

Parse a template and return true if valid, throws Bubble with errors if not.

```temper
export let parseAndValidate(input: String): Boolean throws Bubble {
  parse(input);
  true
}
```

## Re-exported Types

Types are exported from their original modules (ast, tokenizer).
Import them directly from those modules for advanced usage.

Note: Direct re-export syntax is not supported. Use the wrapper functions
(parseTemplate, tokenizeTemplate, renderToHtml, etc.) defined above.

## Error Handling

All parsing functions throw `Bubble` on error. The error message contains
all parse errors with line and column information.

Example usage:

```
// In consuming code (pseudocode):
// let doc = parseTemplate("<div>{@name}</div>") orelse bubble();
// let html = renderToHtml(doc);
// console.log(html);
```

## Version Information

```temper
export let VERSION = "0.1.0";
export let NAME = "heex";
```
