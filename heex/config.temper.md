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

## Imports

Import both the main module and tests (tests are in a separate submodule).

```temper
import(".");
import("./tests");
```

## Module Structure

The library is organized into focused files that are all part of the "heex" module:

- **ast.temper.md** - Core AST definitions (data structures representing parsed templates)
- **tokenizer.temper.md** - Lexical analysis (converts text to tokens)
- **parser.temper.md** - Syntax analysis (converts tokens to AST)
- **renderer.temper.md** - Output generation (converts AST back to various outputs)
- **exports.temper.md** - Public API surface

All files in this directory are automatically combined into a single Temper module.

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
