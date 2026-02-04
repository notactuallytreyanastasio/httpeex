# HEEx Public API

This module defines the public API surface for the HEEx parser library.
It re-exports the core functions and types that consumers should use.

## Imports

```temper
let {
  Node, Document, Text, Element, Component, ComponentType, Slot,
  Expression, Attribute, StaticAttribute, DynamicAttribute,
  SpreadAttribute, SpecialAttribute, EEx, EExType, EExBlock,
  EExClause, Comment, Location, Span,
  isVoidElement, isLocalComponent, isRemoteComponent, isSlot
} = import("./ast");

let { Token, TokenType, Tokenizer, tokenize } = import("./tokenizer");

let { Parser, parse, parseTokens } = import("./parser");

let { renderHtml, renderDebug, renderJson } = import("./renderer");
```

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

Export all AST types for consumers who need to work with the tree directly.

```temper
// Document and nodes
export { Document, Node, Text, Element, Component, Slot, Expression, EEx, EExBlock, Comment };

// Attribute types
export { Attribute, StaticAttribute, DynamicAttribute, SpreadAttribute, SpecialAttribute };

// Supporting types
export { ComponentType, EExType, EExClause, Location, Span };

// Token types (for advanced usage)
export { Token, TokenType };

// Utility functions
export { isVoidElement, isLocalComponent, isRemoteComponent, isSlot };
```

## Error Handling

All parsing functions throw `Bubble` on error. The error message contains
all parse errors with line and column information.

Example usage:

```temper
// In consuming code:
try {
  let doc = parseTemplate("<div>{@name}</div>");
  let html = renderToHtml(doc);
  console.log(html);
} catch (e: Bubble) {
  console.log("Parse errors: " + e.message);
}
```

## Version Information

```temper
export let VERSION = "0.1.0";
export let NAME = "heex";
```
