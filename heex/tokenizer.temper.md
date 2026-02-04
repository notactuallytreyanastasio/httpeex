# HEEx Tokenizer

The tokenizer converts raw template text into a stream of tokens.
This is the first stage of the parsing pipeline.

## Token Types

HEEx has several distinct token types representing different syntactic elements.

```temper
let { Location, Span } = import("./ast");

// All possible token types in HEEx
export class TokenType {
  // Structural
  public static let Text = new TokenType("text");
  public static let Whitespace = new TokenType("whitespace");

  // Tags
  public static let TagOpen = new TokenType("tag_open");         // <div
  public static let TagClose = new TokenType("tag_close");       // </div>
  public static let TagSelfClose = new TokenType("tag_self_close"); // />
  public static let TagEnd = new TokenType("tag_end");           // >

  // Components
  public static let ComponentOpen = new TokenType("component_open");   // <.name or <Module.name
  public static let ComponentClose = new TokenType("component_close"); // </.name> or </Module.name>

  // Slots
  public static let SlotOpen = new TokenType("slot_open");       // <:name
  public static let SlotClose = new TokenType("slot_close");     // </:name>

  // Attributes
  public static let AttrName = new TokenType("attr_name");
  public static let AttrEquals = new TokenType("attr_equals");   // =
  public static let AttrValue = new TokenType("attr_value");     // "value" or 'value'

  // Expressions
  public static let ExprOpen = new TokenType("expr_open");       // {
  public static let ExprClose = new TokenType("expr_close");     // }
  public static let ExprContent = new TokenType("expr_content"); // content inside {}

  // EEx
  public static let EExOpen = new TokenType("eex_open");         // <%
  public static let EExOutput = new TokenType("eex_output");     // <%=
  public static let EExComment = new TokenType("eex_comment");   // <%#
  public static let EExClose = new TokenType("eex_close");       // %>
  public static let EExContent = new TokenType("eex_content");   // content inside <% %>

  // HTML Comments
  public static let CommentOpen = new TokenType("comment_open"); // <!--
  public static let CommentClose = new TokenType("comment_close"); // -->
  public static let CommentContent = new TokenType("comment_content");

  // Special
  public static let EOF = new TokenType("eof");

  public var kind: String;

  public constructor(k: String) {
    kind = k;
  }

  public toString(): String {
    kind
  }
}
```

## Token Structure

Each token carries its type, value, and source location.

```temper
export class Token(
  public tokenType: TokenType,
  public value: String,
  public span: Span,
) {
  public toString(): String {
    "${tokenType.kind}(\"${value}\")"
  }
}
```

## Tokenizer State

The tokenizer maintains state as it processes the input.

```temper
export class Tokenizer(
  public chars: String,
  public var pos: Int,
  public var line: Int,
  public var column: Int,
) {
  // Error collection for non-fatal errors
  public var errors: ListBuilder<String> = new ListBuilder<String>();

  // Token output
  public var tokens: ListBuilder<Token> = new ListBuilder<Token>();

  // Constructor with defaults
  public constructor(input: String) {
    chars = input;
    pos = 0;
    line = 1;
    column = 1;
  }

  // Check if we've consumed all input
  public isDone(): Boolean {
    pos >= chars.length
  }

  // Peek at current character without consuming
  public peek(): String? {
    if (isDone()) {
      null
    } else {
      chars.charAt(pos)
    }
  }

  // Peek ahead by n characters
  public peekAhead(n: Int): String? {
    if (pos + n >= chars.length) {
      null
    } else {
      chars.charAt(pos + n)
    }
  }

  // Check if input matches a string at current position
  public matches(s: String): Boolean {
    if (pos + s.length > chars.length) {
      false
    } else {
      chars.substring(pos, pos + s.length) == s
    }
  }

  // Consume and return current character
  public advance(): String {
    let c = chars.charAt(pos);
    pos = pos + 1;
    if (c == "\n") {
      line = line + 1;
      column = 1;
    } else {
      column = column + 1;
    }
    c
  }

  // Consume n characters and return them
  public advanceBy(n: Int): String {
    let start = pos;
    for (let i = 0; i < n; ++i) {
      advance();
    }
    chars.substring(start, pos)
  }

  // Get current location
  public location(): Location {
    new Location(line, column, pos)
  }

  // Record an error
  public error(msg: String): Void {
    errors.add("${line}:${column}: ${msg}");
  }

  // Add a token
  public addToken(tokenType: TokenType, value: String, start: Location): Void {
    let span = new Span(start, location());
    tokens.add(new Token(tokenType, value, span));
  }
}
```

## Character Classification

Helper functions for classifying characters.

```temper
export let isWhitespace(c: String): Boolean {
  c == " " || c == "\t" || c == "\n" || c == "\r"
}

export let isAlpha(c: String): Boolean {
  (c >= "a" && c <= "z") || (c >= "A" && c <= "Z")
}

export let isDigit(c: String): Boolean {
  c >= "0" && c <= "9"
}

export let isAlphaNumeric(c: String): Boolean {
  isAlpha(c) || isDigit(c)
}

export let isNameChar(c: String): Boolean {
  isAlphaNumeric(c) || c == "_" || c == "-" || c == "."
}

export let isNameStart(c: String): Boolean {
  isAlpha(c) || c == "_"
}
```

## Main Tokenization Loop

The primary entry point for tokenization.

```temper
export let tokenize(input: String): List<Token> throws Bubble {
  let t = new Tokenizer(input);
  tokenizeAll(t);

  if (t.errors.length > 0) {
    throw new Bubble(t.errors.build().join("\n"));
  }

  t.tokens.build()
}

let tokenizeAll(t: Tokenizer): Void {
  while (!t.isDone()) {
    tokenizeNext(t);
  }
  // Add EOF token
  t.addToken(TokenType.EOF, "", t.location());
}

let tokenizeNext(t: Tokenizer): Void {
  let c = t.peek();

  if (c == null) {
    return;
  }

  // Check for EEx expressions first (before < check)
  if (t.matches("<%")) {
    tokenizeEEx(t);
    return;
  }

  // Check for HTML comments
  if (t.matches("<!--")) {
    tokenizeComment(t);
    return;
  }

  // Check for tag start
  if (c == "<") {
    tokenizeTag(t);
    return;
  }

  // Check for expression start
  if (c == "{") {
    tokenizeExpression(t);
    return;
  }

  // Otherwise, it's text content
  tokenizeText(t);
}
```

## Text Tokenization

Consume text until we hit a special character.

```temper
let tokenizeText(t: Tokenizer): Void {
  let start = t.location();
  let text = new StringBuilder();

  while (!t.isDone()) {
    let c = t.peek();

    // Stop at special characters
    if (c == "<" || c == "{") {
      break;
    }

    text.append(t.advance());
  }

  let value = text.build();
  if (value.length > 0) {
    t.addToken(TokenType.Text, value, start);
  }
}
```

## Tag Tokenization

Handle HTML tags, components, and slots.

```temper
let tokenizeTag(t: Tokenizer): Void {
  let start = t.location();

  // Consume <
  t.advance();

  // Check for closing tag
  if (t.peek() == "/") {
    t.advance();
    tokenizeClosingTag(t, start);
    return;
  }

  // Check for slot (:name)
  if (t.peek() == ":") {
    t.advance();
    tokenizeSlotOpen(t, start);
    return;
  }

  // Check for local component (.name)
  if (t.peek() == ".") {
    t.advance();
    tokenizeComponentOpen(t, start, true);
    return;
  }

  // Read tag/component name
  let name = readName(t);

  if (name.length == 0) {
    t.error("Expected tag name after <");
    return;
  }

  // Check if it's a remote component (starts with uppercase)
  if (isRemoteComponent(name)) {
    t.addToken(TokenType.ComponentOpen, name, start);
  } else {
    t.addToken(TokenType.TagOpen, name, start);
  }

  // Tokenize attributes
  tokenizeAttributes(t);

  // Check for self-closing or end
  skipWhitespace(t);
  if (t.matches("/>")) {
    t.advanceBy(2);
    t.addToken(TokenType.TagSelfClose, "/>", t.location());
  } else if (t.peek() == ">") {
    t.advance();
    t.addToken(TokenType.TagEnd, ">", t.location());
  } else {
    t.error("Expected > or /> to close tag");
  }
}

let tokenizeClosingTag(t: Tokenizer, start: Location): Void {
  // Check for slot close (</:name>)
  if (t.peek() == ":") {
    t.advance();
    let name = readName(t);
    skipWhitespace(t);
    if (t.peek() == ">") {
      t.advance();
    }
    t.addToken(TokenType.SlotClose, name, start);
    return;
  }

  // Check for local component close (</.name>)
  if (t.peek() == ".") {
    t.advance();
    let name = readName(t);
    skipWhitespace(t);
    if (t.peek() == ">") {
      t.advance();
    }
    t.addToken(TokenType.ComponentClose, "." + name, start);
    return;
  }

  // Regular tag or remote component close
  let name = readName(t);
  skipWhitespace(t);
  if (t.peek() == ">") {
    t.advance();
  }

  if (isRemoteComponent(name)) {
    t.addToken(TokenType.ComponentClose, name, start);
  } else {
    t.addToken(TokenType.TagClose, name, start);
  }
}

let tokenizeSlotOpen(t: Tokenizer, start: Location): Void {
  let name = readName(t);
  t.addToken(TokenType.SlotOpen, name, start);
  tokenizeAttributes(t);

  skipWhitespace(t);
  if (t.matches("/>")) {
    t.advanceBy(2);
    t.addToken(TokenType.TagSelfClose, "/>", t.location());
  } else if (t.peek() == ">") {
    t.advance();
    t.addToken(TokenType.TagEnd, ">", t.location());
  }
}

let tokenizeComponentOpen(t: Tokenizer, start: Location, isLocal: Boolean): Void {
  let name = readName(t);
  let fullName = if (isLocal) { "." + name } else { name };
  t.addToken(TokenType.ComponentOpen, fullName, start);
  tokenizeAttributes(t);

  skipWhitespace(t);
  if (t.matches("/>")) {
    t.advanceBy(2);
    t.addToken(TokenType.TagSelfClose, "/>", t.location());
  } else if (t.peek() == ">") {
    t.advance();
    t.addToken(TokenType.TagEnd, ">", t.location());
  }
}
```

## Attribute Tokenization

Handle static and dynamic attributes.

```temper
let tokenizeAttributes(t: Tokenizer): Void {
  while (!t.isDone()) {
    skipWhitespace(t);

    let c = t.peek();

    // End of attributes
    if (c == ">" || c == "/" || c == null) {
      break;
    }

    // Check for spread attribute {@attrs}
    if (c == "{") {
      tokenizeSpreadAttribute(t);
      continue;
    }

    // Check for special attribute (:if, :for, :key)
    if (c == ":") {
      tokenizeSpecialAttribute(t);
      continue;
    }

    // Regular attribute name
    let start = t.location();
    let name = readName(t);

    if (name.length == 0) {
      t.error("Expected attribute name");
      t.advance(); // Skip bad character
      continue;
    }

    t.addToken(TokenType.AttrName, name, start);

    skipWhitespace(t);

    // Check for =
    if (t.peek() == "=") {
      t.advance();
      t.addToken(TokenType.AttrEquals, "=", t.location());

      skipWhitespace(t);

      // Attribute value
      tokenizeAttributeValue(t);
    }
  }
}

let tokenizeAttributeValue(t: Tokenizer): Void {
  let start = t.location();
  let c = t.peek();

  // Dynamic value {expression}
  if (c == "{") {
    tokenizeExpression(t);
    return;
  }

  // Quoted string value
  if (c == "\"" || c == "'") {
    let quote = t.advance();
    let value = new StringBuilder();

    while (!t.isDone() && t.peek() != quote) {
      value.append(t.advance());
    }

    if (t.peek() == quote) {
      t.advance();
    } else {
      t.error("Unterminated string");
    }

    t.addToken(TokenType.AttrValue, value.build(), start);
    return;
  }

  // Unquoted value (until whitespace or >)
  let value = new StringBuilder();
  while (!t.isDone()) {
    let ch = t.peek();
    if (isWhitespace(ch) || ch == ">" || ch == "/") {
      break;
    }
    value.append(t.advance());
  }

  if (value.length > 0) {
    t.addToken(TokenType.AttrValue, value.build(), start);
  }
}

let tokenizeSpreadAttribute(t: Tokenizer): Void {
  tokenizeExpression(t);
}

let tokenizeSpecialAttribute(t: Tokenizer): Void {
  let start = t.location();
  t.advance(); // Skip :

  let name = readName(t);
  t.addToken(TokenType.AttrName, ":" + name, start);

  skipWhitespace(t);

  if (t.peek() == "=") {
    t.advance();
    t.addToken(TokenType.AttrEquals, "=", t.location());
    skipWhitespace(t);
    tokenizeAttributeValue(t);
  }
}
```

## Expression Tokenization

Handle {expression} interpolation.

```temper
let tokenizeExpression(t: Tokenizer): Void {
  let start = t.location();
  t.advance(); // Skip {

  t.addToken(TokenType.ExprOpen, "{", start);

  // Read until matching }
  let content = new StringBuilder();
  let depth = 1;

  while (!t.isDone() && depth > 0) {
    let c = t.peek();

    if (c == "{") {
      depth = depth + 1;
      content.append(t.advance());
    } else if (c == "}") {
      depth = depth - 1;
      if (depth > 0) {
        content.append(t.advance());
      }
    } else if (c == "\"" || c == "'") {
      // Skip string contents
      let quote = c;
      content.append(t.advance());
      while (!t.isDone() && t.peek() != quote) {
        if (t.peek() == "\\") {
          content.append(t.advance());
        }
        content.append(t.advance());
      }
      if (!t.isDone()) {
        content.append(t.advance());
      }
    } else {
      content.append(t.advance());
    }
  }

  t.addToken(TokenType.ExprContent, content.build(), start);

  if (t.peek() == "}") {
    t.advance();
    t.addToken(TokenType.ExprClose, "}", t.location());
  } else {
    t.error("Unterminated expression");
  }
}
```

## EEx Tokenization

Handle <% %> and <%= %> expressions.

```temper
let tokenizeEEx(t: Tokenizer): Void {
  let start = t.location();

  t.advanceBy(2); // Skip <%

  // Determine EEx type
  let eexType = TokenType.EExOpen;
  if (t.peek() == "=") {
    t.advance();
    eexType = TokenType.EExOutput;
  } else if (t.peek() == "#") {
    t.advance();
    eexType = TokenType.EExComment;
  }

  t.addToken(eexType, "", start);

  // Read content until %>
  let content = new StringBuilder();

  while (!t.isDone() && !t.matches("%>")) {
    content.append(t.advance());
  }

  t.addToken(TokenType.EExContent, content.build().trim(), start);

  if (t.matches("%>")) {
    t.advanceBy(2);
    t.addToken(TokenType.EExClose, "%>", t.location());
  } else {
    t.error("Unterminated EEx expression");
  }
}
```

## Comment Tokenization

Handle <!-- --> HTML comments.

```temper
let tokenizeComment(t: Tokenizer): Void {
  let start = t.location();

  t.advanceBy(4); // Skip <!--
  t.addToken(TokenType.CommentOpen, "<!--", start);

  let content = new StringBuilder();

  while (!t.isDone() && !t.matches("-->")) {
    content.append(t.advance());
  }

  t.addToken(TokenType.CommentContent, content.build(), start);

  if (t.matches("-->")) {
    t.advanceBy(3);
    t.addToken(TokenType.CommentClose, "-->", t.location());
  } else {
    t.error("Unterminated comment");
  }
}
```

## Helper Functions

```temper
let readName(t: Tokenizer): String {
  let name = new StringBuilder();

  // First character must be valid name start
  let c = t.peek();
  if (c != null && isNameStart(c)) {
    name.append(t.advance());

    // Rest can be name chars
    while (!t.isDone()) {
      c = t.peek();
      if (c != null && isNameChar(c)) {
        name.append(t.advance());
      } else {
        break;
      }
    }
  }

  name.build()
}

let skipWhitespace(t: Tokenizer): Void {
  while (!t.isDone()) {
    let c = t.peek();
    if (c != null && isWhitespace(c)) {
      t.advance();
    } else {
      break;
    }
  }
}

// Re-export helper from ast
let isRemoteComponent(name: String): Boolean {
  let first = name.charAt(0);
  first >= "A" && first <= "Z"
}
```
