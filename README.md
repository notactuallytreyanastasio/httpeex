# httpeex

A cross-platform HEEx template parser written in [Temper](https://temperlang.dev), compiling to **6 languages**: JavaScript, Python, Java, C#, Lua, and Rust.

## What is HEEx?

HEEx (HTML + EEx) is the templating language used by [Phoenix LiveView](https://github.com/phoenixframework/phoenix_live_view). It extends Elixir's EEx with HTML-aware features like components, slots, and special attributes.

This parser lets you work with HEEx templates in any language—parse them, transform them, render them back to HTML, or build tooling around them.

## Installation

Choose your language:

| Language | Package | Install |
|----------|---------|---------|
| JavaScript | [heex-js](https://github.com/notactuallytreyanastasio/heex-js) | `npm install @httpeex/heex` |
| Python | [heex-py](https://github.com/notactuallytreyanastasio/heex-py) | `pip install heex-parser` |
| Java | [heex-java](https://github.com/notactuallytreyanastasio/heex-java) | Maven: `dev.httpeex:heex-java` |
| C# | [heex-csharp](https://github.com/notactuallytreyanastasio/heex-csharp) | `dotnet add package Heex` |
| Lua | [heex-lua](https://github.com/notactuallytreyanastasio/heex-lua) | `luarocks install heex-parser` |
| Rust | [heex-rs](https://github.com/notactuallytreyanastasio/heex-rs) | `cargo add heex-parser` |

## Features

- **Full HEEx syntax support**: Elements, components, slots, expressions, EEx blocks
- **Component types**: Local (`.button`), remote (`MyApp.Button`), and slots (`:header`)
- **Special attributes**: `:if`, `:for`, `:let`, `:stream`
- **Multiple output formats**: HTML, JSON AST, debug tree
- **Error-tolerant parsing**: Collects multiple errors instead of failing fast

## Quick Example

```javascript
// JavaScript
import { parse, renderHtml } from '@httpeex/heex';

const doc = parse('<div class={@class}><.button>Click</.button></div>');
console.log(renderHtml(doc));
```

```python
# Python
from heex import parse, render_html

doc = parse('<div class={@class}><.button>Click</.button></div>')
print(render_html(doc))
```

## Supported HEEx Syntax

```heex
<!-- Elements -->
<div class="container">content</div>
<input type="text" />

<!-- Components -->
<.button variant="primary">Click me</.button>
<MyApp.Modal title="Hello">Body</:body></MyApp.Modal>

<!-- Slots -->
<.card>
  <:header>Title</:header>
  <:footer>Footer</:footer>
</.card>

<!-- Expressions -->
<span>{@user.name}</span>
<div {@rest}></div>

<!-- Special attributes -->
<li :for={item <- @items}>{item}</li>
<div :if={@show}>Visible</div>

<!-- EEx -->
<%= @value %>
<% code %>
<%# comment %>
```

## Architecture

This project uses [Temper](https://temperlang.dev), a language designed for writing libraries that compile to multiple target languages. The source code in `heex/` compiles to idiomatic code for each supported language.

```
httpeex/                    # This repo - Temper source
├── heex/                   # Parser implementation
│   ├── ast.temper.md       # AST node definitions
│   ├── tokenizer.temper.md # Lexer
│   ├── parser.temper.md    # Parser
│   └── renderer.temper.md  # Output renderers
└── .github/workflows/      # CI/CD
    ├── build.yml           # Build all 6 backends
    └── distribute.yml      # Push to language repos

notactuallytreyanastasio/heex-{js,py,java,csharp,lua,rs}  # Generated repos
```

## Building from Source

Requires Java 21 and the [Temper CLI](https://github.com/temperlang/temper).

```bash
# Clone
git clone https://github.com/notactuallytreyanastasio/httpeex
cd httpeex

# Build all backends
cd heex
temper build -b js -b py -b java -b csharp -b lua -b rust

# Output in temper.out/
ls temper.out/
```

## Contributing

Contributions welcome! The source is in `heex/*.temper.md` files (literate programming style).

1. Fork this repo
2. Make changes to the Temper source
3. Test locally with `temper build`
4. Submit a PR

## License

MIT
