# Tokenizer Tests

Unit tests for the HEEx tokenizer.

## Setup

```temper
let { Token, TokenType, tokenize } = import("../tokenizer");

// Test helper to check token types
let assertTokenTypes(input: String, expected: List<String>): Void throws Bubble {
  let tokens = tokenize(input);
  let actual = tokens.map { t => t.tokenType.kind };

  if (actual.length != expected.length) {
    throw new Bubble("Expected ${expected.length} tokens, got ${actual.length}: ${actual}");
  }

  for (let i = 0; i < expected.length; ++i) {
    if (actual[i] != expected[i]) {
      throw new Bubble("Token ${i}: expected ${expected[i]}, got ${actual[i]}");
    }
  }
}

// Test helper to check token values
let assertTokenValues(input: String, expected: List<String>): Void throws Bubble {
  let tokens = tokenize(input);
  let actual = tokens.map { t => t.value };

  for (let i = 0; i < expected.length && i < actual.length; ++i) {
    if (actual[i] != expected[i]) {
      throw new Bubble("Token ${i}: expected value '${expected[i]}', got '${actual[i]}'");
    }
  }
}
```

## Text Tokenization

### Plain text

```temper
test "tokenizes plain text" {
  assertTokenTypes("Hello world", ["text", "eof"]);
  assertTokenValues("Hello world", ["Hello world", ""]);
}
```

### Text with whitespace

```temper
test "preserves whitespace in text" {
  assertTokenTypes("  hello  \n  world  ", ["text", "eof"]);
  assertTokenValues("  hello  \n  world  ", ["  hello  \n  world  ", ""]);
}
```

### Empty input

```temper
test "handles empty input" {
  assertTokenTypes("", ["eof"]);
}
```

## HTML Tag Tokenization

### Simple opening tag

```temper
test "tokenizes opening tag" {
  assertTokenTypes("<div>", ["tag_open", "tag_end", "eof"]);
  assertTokenValues("<div>", ["div", ">", ""]);
}
```

### Self-closing tag

```temper
test "tokenizes self-closing tag" {
  assertTokenTypes("<br/>", ["tag_open", "tag_self_close", "eof"]);
  assertTokenValues("<br/>", ["br", "/>"]);
}
```

### Closing tag

```temper
test "tokenizes closing tag" {
  assertTokenTypes("</div>", ["tag_close", "eof"]);
  assertTokenValues("</div>", ["div", ""]);
}
```

### Tag with attributes

```temper
test "tokenizes tag with static attribute" {
  assertTokenTypes("<div class=\"container\">", [
    "tag_open", "attr_name", "attr_equals", "attr_value", "tag_end", "eof"
  ]);
}
```

### Tag with multiple attributes

```temper
test "tokenizes multiple attributes" {
  assertTokenTypes("<div id=\"main\" class=\"wrapper\">", [
    "tag_open",
    "attr_name", "attr_equals", "attr_value",
    "attr_name", "attr_equals", "attr_value",
    "tag_end", "eof"
  ]);
}
```

### Boolean attribute

```temper
test "tokenizes boolean attribute" {
  assertTokenTypes("<input disabled>", [
    "tag_open", "attr_name", "tag_end", "eof"
  ]);
}
```

## Component Tokenization

### Local component

```temper
test "tokenizes local component" {
  assertTokenTypes("<.button>", ["component_open", "tag_end", "eof"]);
  assertTokenValues("<.button>", [".button", ">"]);
}
```

### Local component closing

```temper
test "tokenizes local component close" {
  assertTokenTypes("</.button>", ["component_close", "eof"]);
  assertTokenValues("</.button>", [".button", ""]);
}
```

### Remote component

```temper
test "tokenizes remote component" {
  assertTokenTypes("<MyApp.Button>", ["component_open", "tag_end", "eof"]);
  assertTokenValues("<MyApp.Button>", ["MyApp.Button", ">"]);
}
```

### Remote component closing

```temper
test "tokenizes remote component close" {
  assertTokenTypes("</MyApp.Button>", ["component_close", "eof"]);
}
```

## Slot Tokenization

### Slot opening

```temper
test "tokenizes slot open" {
  assertTokenTypes("<:header>", ["slot_open", "tag_end", "eof"]);
  assertTokenValues("<:header>", ["header", ">"]);
}
```

### Slot closing

```temper
test "tokenizes slot close" {
  assertTokenTypes("</:header>", ["slot_close", "eof"]);
  assertTokenValues("</:header>", ["header", ""]);
}
```

### Self-closing slot

```temper
test "tokenizes self-closing slot" {
  assertTokenTypes("<:icon />", ["slot_open", "tag_self_close", "eof"]);
}
```

## Expression Tokenization

### Simple expression

```temper
test "tokenizes expression" {
  assertTokenTypes("{@name}", ["expr_open", "expr_content", "expr_close", "eof"]);
  assertTokenValues("{@name}", ["{", "@name", "}"]);
}
```

### Expression in text

```temper
test "tokenizes expression in text" {
  assertTokenTypes("Hello {@name}!", [
    "text", "expr_open", "expr_content", "expr_close", "text", "eof"
  ]);
}
```

### Nested braces in expression

```temper
test "handles nested braces in expression" {
  assertTokenTypes("{%{a: 1}}", ["expr_open", "expr_content", "expr_close", "eof"]);
  assertTokenValues("{%{a: 1}}", ["{", "%{a: 1}", "}"]);
}
```

### Expression with strings

```temper
test "handles strings in expression" {
  assertTokenTypes("{\"hello {world}\"}", ["expr_open", "expr_content", "expr_close", "eof"]);
}
```

## Dynamic Attribute Tokenization

### Dynamic attribute value

```temper
test "tokenizes dynamic attribute" {
  assertTokenTypes("<div class={@class}>", [
    "tag_open", "attr_name", "attr_equals",
    "expr_open", "expr_content", "expr_close",
    "tag_end", "eof"
  ]);
}
```

### Spread attribute

```temper
test "tokenizes spread attribute" {
  assertTokenTypes("<div {@attrs}>", [
    "tag_open",
    "expr_open", "expr_content", "expr_close",
    "tag_end", "eof"
  ]);
}
```

## Special Attribute Tokenization

### :if attribute

```temper
test "tokenizes :if attribute" {
  assertTokenTypes("<div :if={@show}>", [
    "tag_open", "attr_name", "attr_equals",
    "expr_open", "expr_content", "expr_close",
    "tag_end", "eof"
  ]);
  assertTokenValues("<div :if={@show}>", ["div", ":if", "=", "{", "@show", "}", ">"]);
}
```

### :for attribute

```temper
test "tokenizes :for attribute" {
  assertTokenTypes("<li :for={item <- @items}>", [
    "tag_open", "attr_name", "attr_equals",
    "expr_open", "expr_content", "expr_close",
    "tag_end", "eof"
  ]);
}
```

### :let attribute

```temper
test "tokenizes :let attribute" {
  assertTokenTypes("<:col :let={value}>", [
    "slot_open", "attr_name", "attr_equals",
    "expr_open", "expr_content", "expr_close",
    "tag_end", "eof"
  ]);
}
```

## EEx Tokenization

### Output expression

```temper
test "tokenizes EEx output" {
  assertTokenTypes("<%= @name %>", [
    "eex_output", "eex_content", "eex_close", "eof"
  ]);
  assertTokenValues("<%= @name %>", ["", "@name", "%>"]);
}
```

### Exec expression

```temper
test "tokenizes EEx exec" {
  assertTokenTypes("<% x = 1 %>", [
    "eex_open", "eex_content", "eex_close", "eof"
  ]);
}
```

### Comment

```temper
test "tokenizes EEx comment" {
  assertTokenTypes("<%# this is a comment %>", [
    "eex_comment", "eex_content", "eex_close", "eof"
  ]);
}
```

### EEx in text

```temper
test "tokenizes EEx in text" {
  assertTokenTypes("Hello <%= @name %> world", [
    "text", "eex_output", "eex_content", "eex_close", "text", "eof"
  ]);
}
```

## HTML Comment Tokenization

### Simple comment

```temper
test "tokenizes HTML comment" {
  assertTokenTypes("<!-- comment -->", [
    "comment_open", "comment_content", "comment_close", "eof"
  ]);
  assertTokenValues("<!-- comment -->", ["<!--", " comment ", "-->"]);
}
```

### Multi-line comment

```temper
test "tokenizes multi-line comment" {
  assertTokenTypes("<!--\n  line 1\n  line 2\n-->", [
    "comment_open", "comment_content", "comment_close", "eof"
  ]);
}
```

## Complex Templates

### Nested structure

```temper
test "tokenizes nested structure" {
  let input = "<div><span>{@text}</span></div>";
  assertTokenTypes(input, [
    "tag_open", "tag_end",
    "tag_open", "tag_end",
    "expr_open", "expr_content", "expr_close",
    "tag_close",
    "tag_close",
    "eof"
  ]);
}
```

### Component with slots

```temper
test "tokenizes component with slots" {
  let input = "<.card><:header>Title</:header><:body>Content</:body></.card>";
  let tokens = tokenize(input);
  // Should have component open/close, two slot open/close pairs, text
  assert(tokens.length > 10);
}
```

## Error Cases

### Unterminated tag

```temper
test "errors on unterminated tag" {
  try {
    tokenize("<div");
    throw new Bubble("Should have thrown");
  } catch (e: Bubble) {
    assert(e.message.contains("Expected") || e.message.contains("tag"));
  }
}
```

### Unterminated expression

```temper
test "errors on unterminated expression" {
  try {
    tokenize("{@name");
    throw new Bubble("Should have thrown");
  } catch (e: Bubble) {
    assert(e.message.contains("expression") || e.message.contains("Unterminated"));
  }
}
```

### Unterminated string

```temper
test "errors on unterminated string in attribute" {
  try {
    tokenize("<div class=\"foo>");
    throw new Bubble("Should have thrown");
  } catch (e: Bubble) {
    assert(e.message.contains("string") || e.message.contains("Unterminated"));
  }
}
```
