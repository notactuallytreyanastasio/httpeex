# HEEx Parser

The parser transforms a token stream into an Abstract Syntax Tree.
This is the second stage of the parsing pipeline.

## Overview

The parser is a **recursive descent parser** that builds a tree structure
from the flat token stream. It handles:

- Nested HTML elements
- Component declarations (local and remote)
- Named slots
- EEx control flow blocks
- Attribute processing

## Imports

```temper
let {
  Node, Document, Text, Element, Component, ComponentType, Slot,
  Expression, Attribute, StaticAttribute, DynamicAttribute,
  SpreadAttribute, SpecialAttribute, EEx, EExType, EExBlock,
  EExClause, Comment, Location, Span, isVoidElement
} = import("./ast");

let {
  Token, TokenType, tokenize
} = import("./tokenizer");
```

## Parser State

```temper
export class Parser(
  public tokens: List<Token>,
  public var pos: Int,
) {
  // Error collection
  public var errors: ListBuilder<String> = new ListBuilder<String>();

  constructor(tokenList: List<Token>) {
    tokens = tokenList;
    pos = 0;
  }

  // Check if we've consumed all tokens
  public isDone(): Boolean {
    pos >= tokens.length || current().tokenType == TokenType.EOF
  }

  // Get current token
  public current(): Token {
    if (pos >= tokens.length) {
      // Return EOF token
      tokens[tokens.length - 1]
    } else {
      tokens[pos]
    }
  }

  // Peek at current token type
  public peek(): TokenType {
    current().tokenType
  }

  // Peek at next token type
  public peekNext(): TokenType {
    if (pos + 1 >= tokens.length) {
      TokenType.EOF
    } else {
      tokens[pos + 1].tokenType
    }
  }

  // Check if current token matches type
  public check(t: TokenType): Boolean {
    peek().kind == t.kind
  }

  // Consume current token and advance
  public advance(): Token {
    let tok = current();
    pos = pos + 1;
    tok
  }

  // Consume token of expected type, or error
  public expect(t: TokenType): Token? {
    if (check(t)) {
      advance()
    } else {
      error("Expected ${t.kind}, got ${peek().kind}");
      null
    }
  }

  // Record an error with location
  public error(msg: String): Void {
    let tok = current();
    errors.add("${tok.span.start}: ${msg}");
  }
}
```

## Main Parse Function

Entry point for parsing a HEEx template.

```temper
export let parse(input: String): Document throws Bubble {
  let tokenList = tokenize(input);
  parseTokens(tokenList)
}

export let parseTokens(tokenList: List<Token>): Document throws Bubble {
  let p = new Parser(tokenList);
  let children = parseChildren(p, null);

  if (p.errors.length > 0) {
    throw new Bubble(p.errors.build().join("\n"));
  }

  new Document(children, null)
}
```

## Parsing Children

Parse a sequence of child nodes until we hit a closing tag or EOF.

```temper
let parseChildren(p: Parser, closingTag: String?): List<Node> {
  let children = new ListBuilder<Node>();

  while (!p.isDone()) {
    // Check for closing tag
    if (closingTag != null && isClosingTag(p, closingTag)) {
      break;
    }

    // Check for slot close
    if (p.check(TokenType.SlotClose)) {
      break;
    }

    // Check for component close
    if (p.check(TokenType.ComponentClose)) {
      break;
    }

    // Check for tag close
    if (p.check(TokenType.TagClose)) {
      break;
    }

    let node = parseNode(p);
    if (node != null) {
      children.add(node);
    }
  }

  children.build()
}

let isClosingTag(p: Parser, tagName: String): Boolean {
  if (p.check(TokenType.TagClose)) {
    p.current().value == tagName
  } else if (p.check(TokenType.ComponentClose)) {
    p.current().value == tagName
  } else if (p.check(TokenType.SlotClose)) {
    p.current().value == tagName
  } else {
    false
  }
}
```

## Parsing Individual Nodes

Dispatch to the appropriate parser based on token type.

```temper
let parseNode(p: Parser): Node? {
  let t = p.peek();

  when (t) {
    is { kind: "text" } -> parseText(p);
    is { kind: "tag_open" } -> parseElement(p);
    is { kind: "component_open" } -> parseComponent(p);
    is { kind: "slot_open" } -> parseSlot(p);
    is { kind: "expr_open" } -> parseExpression(p);
    is { kind: "eex_open" } -> parseEEx(p, EExType.Exec);
    is { kind: "eex_output" } -> parseEEx(p, EExType.Output);
    is { kind: "eex_comment" } -> parseEEx(p, EExType.Comment);
    is { kind: "comment_open" } -> parseComment(p);
    else -> do {
      // Skip unexpected token
      p.error("Unexpected token: ${t.kind}");
      p.advance();
      null
    };
  }
}
```

## Text Parsing

```temper
let parseText(p: Parser): Node {
  let tok = p.advance();
  new Text(tok.value, tok.span)
}
```

## Element Parsing

Parse an HTML element with attributes and children.

```temper
let parseElement(p: Parser): Node {
  let startTok = p.advance(); // Consume TagOpen
  let tagName = startTok.value;

  // Parse attributes
  let attrs = parseAttributes(p);

  // Check for self-closing
  if (p.check(TokenType.TagSelfClose)) {
    p.advance();
    return new Element(tagName, attrs, [], true, startTok.span);
  }

  // Expect >
  p.expect(TokenType.TagEnd);

  // Void elements don't have children
  if (isVoidElement(tagName)) {
    return new Element(tagName, attrs, [], true, startTok.span);
  }

  // Parse children until closing tag
  let children = parseChildren(p, tagName);

  // Expect closing tag
  if (p.check(TokenType.TagClose)) {
    let closeTok = p.advance();
    if (closeTok.value != tagName) {
      p.error("Mismatched closing tag: expected </${tagName}>, got </${closeTok.value}>");
    }
  } else {
    p.error("Expected closing tag </${tagName}>");
  }

  new Element(tagName, attrs, children, false, startTok.span)
}
```

## Component Parsing

Parse local (.name) or remote (Module.name) components.

```temper
let parseComponent(p: Parser): Node {
  let startTok = p.advance(); // Consume ComponentOpen
  let name = startTok.value;

  // Determine component type
  let compType = if (name.startsWith(".")) {
    ComponentType.Local
  } else {
    ComponentType.Remote
  };

  // Parse attributes
  let attrs = parseAttributes(p);

  // Check for self-closing
  if (p.check(TokenType.TagSelfClose)) {
    p.advance();
    return new Component(compType, name, attrs, [], [], startTok.span);
  }

  // Expect >
  p.expect(TokenType.TagEnd);

  // Parse children and slots
  let { children, slots } = parseComponentBody(p, name);

  // Expect closing tag
  if (p.check(TokenType.ComponentClose)) {
    let closeTok = p.advance();
    if (closeTok.value != name) {
      p.error("Mismatched component close: expected </${name}>, got </${closeTok.value}>");
    }
  } else {
    p.error("Expected closing tag for component ${name}");
  }

  new Component(compType, name, attrs, children, slots, startTok.span)
}

// Result type for component body parsing
class ComponentBodyResult(
  public children: List<Node>,
  public slots: List<Slot>,
) {}

let parseComponentBody(p: Parser, componentName: String): ComponentBodyResult {
  let children = new ListBuilder<Node>();
  let slots = new ListBuilder<Slot>();

  while (!p.isDone()) {
    // Check for component close
    if (p.check(TokenType.ComponentClose)) {
      break;
    }

    // Check for slot
    if (p.check(TokenType.SlotOpen)) {
      slots.add(parseSlot(p) as Slot);
      continue;
    }

    let node = parseNode(p);
    if (node != null) {
      children.add(node);
    }
  }

  new ComponentBodyResult(children.build(), slots.build())
}
```

## Slot Parsing

Parse named slots (<:name>...</:name>).

```temper
let parseSlot(p: Parser): Node {
  let startTok = p.advance(); // Consume SlotOpen
  let name = startTok.value;

  // Parse attributes (including :let)
  let attrs = parseAttributes(p);

  // Extract :let binding if present
  let letBinding: String? = null;
  for (let attr of attrs) {
    when (attr) {
      is SpecialAttribute -> do {
        if (attr.kind == "let") {
          letBinding = attr.expression.code;
        }
      };
      else -> void;
    }
  }

  // Check for self-closing
  if (p.check(TokenType.TagSelfClose)) {
    p.advance();
    return new Slot(name, attrs, [], letBinding, startTok.span);
  }

  // Expect >
  p.expect(TokenType.TagEnd);

  // Parse children
  let children = parseChildren(p, null);

  // Expect slot close
  if (p.check(TokenType.SlotClose)) {
    let closeTok = p.advance();
    if (closeTok.value != name) {
      p.error("Mismatched slot close: expected </:${name}>, got </:${closeTok.value}>");
    }
  } else {
    p.error("Expected closing tag for slot ${name}");
  }

  new Slot(name, attrs, children, letBinding, startTok.span)
}
```

## Attribute Parsing

Parse element/component attributes.

```temper
let parseAttributes(p: Parser): List<Attribute> {
  let attrs = new ListBuilder<Attribute>();

  while (!p.isDone()) {
    // Stop at tag end
    if (p.check(TokenType.TagEnd) || p.check(TokenType.TagSelfClose)) {
      break;
    }

    // Spread attribute
    if (p.check(TokenType.ExprOpen)) {
      let expr = parseExpression(p) as Expression;
      attrs.add(new SpreadAttribute(expr, expr.span));
      continue;
    }

    // Attribute name
    if (p.check(TokenType.AttrName)) {
      let nameTok = p.advance();
      let name = nameTok.value;

      // Check for special attribute (:if, :for, :key, :let)
      let isSpecial = name.startsWith(":");

      // Check for =
      if (p.check(TokenType.AttrEquals)) {
        p.advance(); // Consume =

        // Parse value
        if (p.check(TokenType.ExprOpen)) {
          let expr = parseExpression(p) as Expression;
          if (isSpecial) {
            attrs.add(new SpecialAttribute(name.substring(1), expr, nameTok.span));
          } else {
            attrs.add(new DynamicAttribute(name, expr, nameTok.span));
          }
        } else if (p.check(TokenType.AttrValue)) {
          let valTok = p.advance();
          attrs.add(new StaticAttribute(name, valTok.value, nameTok.span));
        } else {
          p.error("Expected attribute value");
        }
      } else {
        // Boolean attribute (no value)
        attrs.add(new StaticAttribute(name, "true", nameTok.span));
      }
    } else {
      // Unexpected token in attributes
      break;
    }
  }

  attrs.build()
}
```

## Expression Parsing

Parse {expression} interpolation.

```temper
let parseExpression(p: Parser): Node {
  let startTok = p.advance(); // Consume ExprOpen

  let code = "";
  if (p.check(TokenType.ExprContent)) {
    code = p.advance().value;
  }

  p.expect(TokenType.ExprClose);

  new Expression(code, startTok.span)
}
```

## EEx Parsing

Parse <% %> and <%= %> expressions.

```temper
let parseEEx(p: Parser, eexType: EExType): Node {
  let startTok = p.advance(); // Consume EExOpen/EExOutput/EExComment

  let code = "";
  if (p.check(TokenType.EExContent)) {
    code = p.advance().value;
  }

  p.expect(TokenType.EExClose);

  // Check if this is a block start (if, case, for, etc.)
  if (isBlockStart(code)) {
    return parseEExBlock(p, code, startTok.span);
  }

  new EEx(eexType, code, startTok.span)
}

let isBlockStart(code: String): Boolean {
  let trimmed = code.trim();
  trimmed.startsWith("if ") ||
  trimmed.startsWith("case ") ||
  trimmed.startsWith("cond ") ||
  trimmed.startsWith("for ") ||
  trimmed.startsWith("unless ")
}

let parseEExBlock(p: Parser, startCode: String, span: Span?): Node {
  // Extract block type and expression
  let parts = startCode.trim().split(" ", 2);
  let blockType = parts[0];
  let expression = if (parts.length > 1) { parts[1] } else { "" };

  // Remove trailing "do" if present
  if (expression.endsWith(" do")) {
    expression = expression.substring(0, expression.length - 3);
  }

  let clauses = new ListBuilder<EExClause>();

  // Parse do clause
  let doChildren = parseEExBlockBody(p, blockType);
  clauses.add(new EExClause("do", null, doChildren, span));

  // Handle additional clauses (else, end, etc.)
  while (!p.isDone()) {
    if (p.check(TokenType.EExOpen) || p.check(TokenType.EExOutput)) {
      let tok = p.current();
      // Peek at content
      if (isBlockClause(p)) {
        p.advance(); // Consume EEx open
        let clauseCode = "";
        if (p.check(TokenType.EExContent)) {
          clauseCode = p.advance().value.trim();
        }
        p.expect(TokenType.EExClose);

        if (clauseCode == "end") {
          clauses.add(new EExClause("end", null, [], span));
          break;
        } else if (clauseCode == "else") {
          let elseChildren = parseEExBlockBody(p, blockType);
          clauses.add(new EExClause("else", null, elseChildren, span));
        } else {
          // Case clause or other
          let clauseChildren = parseEExBlockBody(p, blockType);
          clauses.add(new EExClause("->", clauseCode, clauseChildren, span));
        }
      } else {
        break;
      }
    } else {
      break;
    }
  }

  new EExBlock(blockType, expression, clauses.build(), span)
}

let isBlockClause(p: Parser): Boolean {
  // Look ahead to see if content is a block clause
  if (p.pos + 1 < p.tokens.length) {
    let nextTok = p.tokens[p.pos + 1];
    if (nextTok.tokenType.kind == "eex_content") {
      let code = nextTok.value.trim();
      return code == "end" || code == "else" || code.contains("->");
    }
  }
  false
}

let parseEExBlockBody(p: Parser, blockType: String): List<Node> {
  let children = new ListBuilder<Node>();

  while (!p.isDone()) {
    // Check for block clause
    if (p.check(TokenType.EExOpen) || p.check(TokenType.EExOutput)) {
      if (isBlockClause(p)) {
        break;
      }
    }

    let node = parseNode(p);
    if (node != null) {
      children.add(node);
    }
  }

  children.build()
}
```

## Comment Parsing

```temper
let parseComment(p: Parser): Node {
  let startTok = p.advance(); // Consume CommentOpen

  let content = "";
  if (p.check(TokenType.CommentContent)) {
    content = p.advance().value;
  }

  p.expect(TokenType.CommentClose);

  new Comment(content, startTok.span)
}
```
