# Tokenizer Tests

Unit tests for the HEEx tokenizer.

## Setup

```temper
let { Token, TokenType, tokenize } = import("../tokenizer");

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

```temper
test("tokenizes plain text") {
  assertTokenTypes("Hello world", ["text", "eof"]);
  assertTokenValues("Hello world", ["Hello world", ""]);
}

test("preserves whitespace in text") {
  assertTokenTypes("  hello  ", ["text", "eof"]);
}

test("handles empty input") {
  assertTokenTypes("", ["eof"]);
}
```

## HTML Tag Tokenization

```temper
test("tokenizes opening tag") {
  assertTokenTypes("<div>", ["tag_open", "tag_end", "eof"]);
  assertTokenValues("<div>", ["div", ">", ""]);
}

test("tokenizes self-closing tag") {
  assertTokenTypes("<br/>", ["tag_open", "tag_self_close", "eof"]);
  assertTokenValues("<br/>", ["br", "/>"]);
}

test("tokenizes closing tag") {
  assertTokenTypes("</div>", ["tag_close", "eof"]);
  assertTokenValues("</div>", ["div", ""]);
}

test("tokenizes tag with static attribute") {
  assertTokenTypes("<div class=\"container\">", ["tag_open", "attr_name", "attr_equals", "attr_value", "tag_end", "eof"]);
}

test("tokenizes multiple attributes") {
  assertTokenTypes("<div id=\"main\" class=\"wrapper\">", ["tag_open", "attr_name", "attr_equals", "attr_value", "attr_name", "attr_equals", "attr_value", "tag_end", "eof"]);
}

test("tokenizes boolean attribute") {
  assertTokenTypes("<input disabled>", ["tag_open", "attr_name", "tag_end", "eof"]);
}
```

## Component Tokenization

```temper
test("tokenizes local component") {
  assertTokenTypes("<.button>", ["component_open", "tag_end", "eof"]);
  assertTokenValues("<.button>", [".button", ">"]);
}

test("tokenizes local component close") {
  assertTokenTypes("</.button>", ["component_close", "eof"]);
  assertTokenValues("</.button>", [".button", ""]);
}

test("tokenizes remote component") {
  assertTokenTypes("<MyApp.Button>", ["component_open", "tag_end", "eof"]);
  assertTokenValues("<MyApp.Button>", ["MyApp.Button", ">"]);
}

test("tokenizes remote component close") {
  assertTokenTypes("</MyApp.Button>", ["component_close", "eof"]);
}
```

## Slot Tokenization

```temper
test("tokenizes slot open") {
  assertTokenTypes("<:header>", ["slot_open", "tag_end", "eof"]);
  assertTokenValues("<:header>", ["header", ">"]);
}

test("tokenizes slot close") {
  assertTokenTypes("</:header>", ["slot_close", "eof"]);
  assertTokenValues("</:header>", ["header", ""]);
}

test("tokenizes self-closing slot") {
  assertTokenTypes("<:icon />", ["slot_open", "tag_self_close", "eof"]);
}
```

## Expression Tokenization

```temper
test("tokenizes expression") {
  assertTokenTypes("{@name}", ["expr_open", "expr_content", "expr_close", "eof"]);
  assertTokenValues("{@name}", ["{", "@name", "}"]);
}

test("tokenizes expression in text") {
  assertTokenTypes("Hello {@name}!", ["text", "expr_open", "expr_content", "expr_close", "text", "eof"]);
}

test("handles nested braces in expression") {
  assertTokenTypes("{%{a: 1}}", ["expr_open", "expr_content", "expr_close", "eof"]);
  assertTokenValues("{%{a: 1}}", ["{", "%{a: 1}", "}"]);
}

test("handles strings in expression") {
  assertTokenTypes("{\"hello {world}\"}", ["expr_open", "expr_content", "expr_close", "eof"]);
}
```

## Dynamic Attribute Tokenization

```temper
test("tokenizes dynamic attribute") {
  assertTokenTypes("<div class={@class}>", ["tag_open", "attr_name", "attr_equals", "expr_open", "expr_content", "expr_close", "tag_end", "eof"]);
}

test("tokenizes spread attribute") {
  assertTokenTypes("<div {@attrs}>", ["tag_open", "expr_open", "expr_content", "expr_close", "tag_end", "eof"]);
}
```

## Special Attribute Tokenization

```temper
test("tokenizes :if attribute") {
  assertTokenTypes("<div :if={@show}>", ["tag_open", "attr_name", "attr_equals", "expr_open", "expr_content", "expr_close", "tag_end", "eof"]);
  assertTokenValues("<div :if={@show}>", ["div", ":if", "=", "{", "@show", "}", ">"]);
}

test("tokenizes :for attribute") {
  assertTokenTypes("<li :for={item <- @items}>", ["tag_open", "attr_name", "attr_equals", "expr_open", "expr_content", "expr_close", "tag_end", "eof"]);
}

test("tokenizes :let attribute") {
  assertTokenTypes("<:col :let={value}>", ["slot_open", "attr_name", "attr_equals", "expr_open", "expr_content", "expr_close", "tag_end", "eof"]);
}
```

## EEx Tokenization

```temper
test("tokenizes EEx output") {
  assertTokenTypes("<%= @name %>", ["eex_output", "eex_content", "eex_close", "eof"]);
  assertTokenValues("<%= @name %>", ["", "@name", "%>"]);
}

test("tokenizes EEx exec") {
  assertTokenTypes("<% x = 1 %>", ["eex_open", "eex_content", "eex_close", "eof"]);
}

test("tokenizes EEx comment") {
  assertTokenTypes("<%# this is a comment %>", ["eex_comment", "eex_content", "eex_close", "eof"]);
}

test("tokenizes EEx in text") {
  assertTokenTypes("Hello <%= @name %> world", ["text", "eex_output", "eex_content", "eex_close", "text", "eof"]);
}
```

## HTML Comment Tokenization

```temper
test("tokenizes HTML comment") {
  assertTokenTypes("<!-- comment -->", ["comment_open", "comment_content", "comment_close", "eof"]);
  assertTokenValues("<!-- comment -->", ["<!--", " comment ", "-->"]);
}

test("tokenizes multi-line comment") {
  assertTokenTypes("<!--\n  line 1\n  line 2\n-->", ["comment_open", "comment_content", "comment_close", "eof"]);
}
```

## Complex Templates

```temper
test("tokenizes nested structure") {
  let input = "<div><span>{@text}</span></div>";
  assertTokenTypes(input, ["tag_open", "tag_end", "tag_open", "tag_end", "expr_open", "expr_content", "expr_close", "tag_close", "tag_close", "eof"]);
}

test("tokenizes component with slots") {
  let input = "<.card><:header>Title</:header><:body>Content</:body></.card>";
  let tokens = tokenize(input);
  assert(tokens.length > 10);
}
```

## Error Cases

```temper
let shouldBubble(action: fn(): Void throws Bubble): Boolean {
  do {
    action();
    false
  } orelse true
}

test("errors on unterminated tag") {
  assert(shouldBubble { tokenize("<div"); });
}

test("errors on unterminated expression") {
  assert(shouldBubble { tokenize("{@name"); });
}

test("errors on unterminated string in attribute") {
  assert(shouldBubble { tokenize("<div class=\"foo>"); });
}
```
