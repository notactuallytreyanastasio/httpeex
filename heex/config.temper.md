# HEEx Parser for Temper

A cross-platform implementation of Phoenix LiveView's HEEx templating language,
written in Temper for compilation to multiple backend languages.

## Package Metadata

```temper
export let name = "heex";
export let version = "0.1.0";
export let description = "HEEx template parser and renderer";
export let license = "MIT";
export let authors = ["httpeex contributors"];
```

## Backend-Specific Configuration

```temper
export let javaGroup = "dev.httpeex";
export let javaName = "heex-java";
export let jsName = "@httpeex/heex";
export let pyName = "heex-py";
export let rustName = "heex-rs";
```

## Module Imports

The library is organized into focused modules that build on each other:

```temper
// Core AST definitions - the data structures representing parsed templates
import("./ast");

// Tokenizer - lexical analysis, converts text to tokens
import("./tokenizer");

// Parser - syntax analysis, converts tokens to AST
import("./parser");

// Renderer - converts AST back to various outputs
import("./renderer");

// Public API surface
import("./exports");

// Test modules
import("./tests/tokenizer");
import("./tests/parser");
import("./tests/integration");
```

## Design Philosophy

### Literate Programming First

This implementation follows Temper's literate programming paradigm. Each module
is a Markdown document with embedded code, allowing the implementation to serve
as both documentation and executable specification.

### Three-Stage Pipeline

Following HEEx's original architecture:

1. **Tokenization**: Raw template → Token stream
2. **Parsing**: Token stream → Abstract Syntax Tree
3. **Rendering**: AST → Output (HTML, code, etc.)

### Error-Tolerant Processing

Like the `temper-regex-parser`, we collect errors rather than failing fast.
This provides better developer experience with multiple error reports.

### Cross-Platform Compatibility

The parser outputs a platform-agnostic AST that can be:
- Rendered to HTML strings
- Compiled to target language code
- Serialized to JSON for tooling
