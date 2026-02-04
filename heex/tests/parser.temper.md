# Parser Tests

Unit tests for the HEEx parser.

## Setup

```temper
let {
  Node, Document, Text, Element, Component, ComponentType, Slot,
  Expression, Attribute, StaticAttribute, DynamicAttribute,
  SpreadAttribute, SpecialAttribute, EEx, EExType, EExBlock,
  EExClause, Comment
} = import("../ast");

let { parse } = import("../parser");

// Helper to get first child of document
let firstChild(doc: Document): Node {
  doc.children[0]
}

// Helper to check node type
let assertNodeType(node: Node, expected: String): Void throws Bubble {
  if (node.nodeType() != expected) {
    throw new Bubble("Expected ${expected}, got ${node.nodeType()}");
  }
}

// Helper to check document child count
let assertChildCount(doc: Document, expected: Int): Void throws Bubble {
  if (doc.children.length != expected) {
    throw new Bubble("Expected ${expected} children, got ${doc.children.length}");
  }
}
```

## Text Parsing

### Plain text

```temper
test "parses plain text" {
  let doc = parse("Hello world");
  assertChildCount(doc, 1);
  let text = firstChild(doc) as Text;
  assert(text.content == "Hello world");
}
```

### Whitespace preservation

```temper
test "preserves whitespace" {
  let doc = parse("  hello  ");
  let text = firstChild(doc) as Text;
  assert(text.content == "  hello  ");
}
```

### Empty document

```temper
test "parses empty document" {
  let doc = parse("");
  assertChildCount(doc, 0);
}
```

## Element Parsing

### Simple element

```temper
test "parses simple element" {
  let doc = parse("<div></div>");
  assertChildCount(doc, 1);
  let el = firstChild(doc) as Element;
  assert(el.tag == "div");
  assert(el.children.length == 0);
  assert(el.selfClosing == false);
}
```

### Self-closing element

```temper
test "parses self-closing element" {
  let doc = parse("<br/>");
  let el = firstChild(doc) as Element;
  assert(el.tag == "br");
  assert(el.selfClosing == true);
}
```

### Void element

```temper
test "parses void element without closing tag" {
  let doc = parse("<input>");
  let el = firstChild(doc) as Element;
  assert(el.tag == "input");
  // Void elements are treated as self-closing
}
```

### Nested elements

```temper
test "parses nested elements" {
  let doc = parse("<div><span></span></div>");
  let div = firstChild(doc) as Element;
  assert(div.tag == "div");
  assert(div.children.length == 1);

  let span = div.children[0] as Element;
  assert(span.tag == "span");
}
```

### Element with text

```temper
test "parses element with text content" {
  let doc = parse("<p>Hello</p>");
  let p = firstChild(doc) as Element;
  assert(p.children.length == 1);

  let text = p.children[0] as Text;
  assert(text.content == "Hello");
}
```

### Deeply nested

```temper
test "parses deeply nested elements" {
  let doc = parse("<div><section><article><p>Text</p></article></section></div>");
  let div = firstChild(doc) as Element;
  let section = div.children[0] as Element;
  let article = section.children[0] as Element;
  let p = article.children[0] as Element;
  let text = p.children[0] as Text;
  assert(text.content == "Text");
}
```

## Attribute Parsing

### Static attribute

```temper
test "parses static attribute" {
  let doc = parse("<div class=\"container\"></div>");
  let el = firstChild(doc) as Element;
  assert(el.attributes.length == 1);

  let attr = el.attributes[0] as StaticAttribute;
  assert(attr.name == "class");
  assert(attr.value == "container");
}
```

### Multiple attributes

```temper
test "parses multiple attributes" {
  let doc = parse("<div id=\"main\" class=\"wrapper\"></div>");
  let el = firstChild(doc) as Element;
  assert(el.attributes.length == 2);
}
```

### Boolean attribute

```temper
test "parses boolean attribute" {
  let doc = parse("<input disabled>");
  let el = firstChild(doc) as Element;
  let attr = el.attributes[0] as StaticAttribute;
  assert(attr.name == "disabled");
  assert(attr.value == "true");
}
```

### Dynamic attribute

```temper
test "parses dynamic attribute" {
  let doc = parse("<div class={@class}></div>");
  let el = firstChild(doc) as Element;

  let attr = el.attributes[0] as DynamicAttribute;
  assert(attr.name == "class");
  assert(attr.expression.code == "@class");
}
```

### Spread attribute

```temper
test "parses spread attribute" {
  let doc = parse("<div {@attrs}></div>");
  let el = firstChild(doc) as Element;

  let attr = el.attributes[0] as SpreadAttribute;
  assert(attr.expression.code == "@attrs");
}
```

### Special attributes

```temper
test "parses :if attribute" {
  let doc = parse("<div :if={@show}></div>");
  let el = firstChild(doc) as Element;

  let attr = el.attributes[0] as SpecialAttribute;
  assert(attr.kind == "if");
  assert(attr.expression.code == "@show");
}

test "parses :for attribute" {
  let doc = parse("<li :for={item <- @items}></li>");
  let el = firstChild(doc) as Element;

  let attr = el.attributes[0] as SpecialAttribute;
  assert(attr.kind == "for");
  assert(attr.expression.code == "item <- @items");
}

test "parses :key attribute" {
  let doc = parse("<li :key={item.id}></li>");
  let el = firstChild(doc) as Element;

  let attr = el.attributes[0] as SpecialAttribute;
  assert(attr.kind == "key");
}
```

## Component Parsing

### Local component

```temper
test "parses local component" {
  let doc = parse("<.button></.button>");
  let comp = firstChild(doc) as Component;
  assert(comp.name == ".button");
  assert(comp.componentType.kind == "local");
}
```

### Self-closing local component

```temper
test "parses self-closing local component" {
  let doc = parse("<.icon />");
  let comp = firstChild(doc) as Component;
  assert(comp.name == ".icon");
}
```

### Remote component

```temper
test "parses remote component" {
  let doc = parse("<MyApp.Button></MyApp.Button>");
  let comp = firstChild(doc) as Component;
  assert(comp.name == "MyApp.Button");
  assert(comp.componentType.kind == "remote");
}
```

### Component with attributes

```temper
test "parses component with attributes" {
  let doc = parse("<.button type=\"submit\" disabled></.button>");
  let comp = firstChild(doc) as Component;
  assert(comp.attributes.length == 2);
}
```

### Component with children

```temper
test "parses component with children" {
  let doc = parse("<.card>Content</.card>");
  let comp = firstChild(doc) as Component;
  assert(comp.children.length == 1);

  let text = comp.children[0] as Text;
  assert(text.content == "Content");
}
```

## Slot Parsing

### Named slot

```temper
test "parses named slot" {
  let doc = parse("<.card><:header>Title</:header></.card>");
  let comp = firstChild(doc) as Component;
  assert(comp.slots.length == 1);

  let slot = comp.slots[0];
  assert(slot.name == "header");
  assert(slot.children.length == 1);
}
```

### Multiple slots

```temper
test "parses multiple slots" {
  let doc = parse("<.card><:header>H</:header><:body>B</:body></.card>");
  let comp = firstChild(doc) as Component;
  assert(comp.slots.length == 2);
}
```

### Slot with :let binding

```temper
test "parses slot with :let binding" {
  let doc = parse("<.table><:col :let={value}>{value}</:col></.table>");
  let comp = firstChild(doc) as Component;
  let slot = comp.slots[0];
  assert(slot.letBinding == "value");
}
```

### Self-closing slot

```temper
test "parses self-closing slot" {
  let doc = parse("<.tabs><:tab label=\"One\" /></.tabs>");
  let comp = firstChild(doc) as Component;
  assert(comp.slots.length == 1);
}
```

### Mixed children and slots

```temper
test "parses component with both children and slots" {
  let doc = parse("<.modal>Default content<:footer>Buttons</:footer></.modal>");
  let comp = firstChild(doc) as Component;
  assert(comp.children.length == 1); // Default content
  assert(comp.slots.length == 1);    // Footer slot
}
```

## Expression Parsing

### Text interpolation

```temper
test "parses expression in text" {
  let doc = parse("Hello {@name}!");
  assertChildCount(doc, 3);

  let expr = doc.children[1] as Expression;
  assert(expr.code == "@name");
}
```

### Expression in element

```temper
test "parses expression in element" {
  let doc = parse("<span>{@value}</span>");
  let span = firstChild(doc) as Element;
  let expr = span.children[0] as Expression;
  assert(expr.code == "@value");
}
```

## EEx Parsing

### Output expression

```temper
test "parses EEx output" {
  let doc = parse("<%= @name %>");
  let eex = firstChild(doc) as EEx;
  assert(eex.eexType.kind == "output");
  assert(eex.code == "@name");
}
```

### Exec expression

```temper
test "parses EEx exec" {
  let doc = parse("<% x = 1 %>");
  let eex = firstChild(doc) as EEx;
  assert(eex.eexType.kind == "exec");
}
```

### EEx comment

```temper
test "parses EEx comment" {
  let doc = parse("<%# comment %>");
  let eex = firstChild(doc) as EEx;
  assert(eex.eexType.kind == "comment");
}
```

### EEx if block

```temper
test "parses EEx if block" {
  let doc = parse("<%= if @show do %>visible<% end %>");
  let block = firstChild(doc) as EExBlock;
  assert(block.blockType == "if");
  assert(block.clauses.length >= 1);
}
```

### EEx if/else block

```temper
test "parses EEx if/else block" {
  let doc = parse("<%= if @show do %>yes<% else %>no<% end %>");
  let block = firstChild(doc) as EExBlock;
  assert(block.blockType == "if");
  // Should have do, else, end clauses
}
```

### EEx for block

```temper
test "parses EEx for block" {
  let doc = parse("<%= for item <- @items do %><li>{item}</li><% end %>");
  let block = firstChild(doc) as EExBlock;
  assert(block.blockType == "for");
}
```

## Comment Parsing

### HTML comment

```temper
test "parses HTML comment" {
  let doc = parse("<!-- comment -->");
  let comment = firstChild(doc) as Comment;
  assert(comment.content == " comment ");
}
```

### Comment between elements

```temper
test "parses comment between elements" {
  let doc = parse("<div></div><!-- separator --><span></span>");
  assertChildCount(doc, 3);
  assertNodeType(doc.children[1], "comment");
}
```

## Error Handling

### Mismatched tags

```temper
test "errors on mismatched tags" {
  try {
    parse("<div></span>");
    throw new Bubble("Should have thrown");
  } catch (e: Bubble) {
    assert(e.message.contains("Mismatched") || e.message.contains("closing"));
  }
}
```

### Unclosed tag

```temper
test "errors on unclosed tag" {
  try {
    parse("<div><span></div>");
    throw new Bubble("Should have thrown");
  } catch (e: Bubble) {
    assert(e.message.contains("closing") || e.message.contains("Mismatched"));
  }
}
```

### Invalid component close

```temper
test "errors on mismatched component close" {
  try {
    parse("<.button></.other>");
    throw new Bubble("Should have thrown");
  } catch (e: Bubble) {
    assert(e.message.contains("Mismatched") || e.message.contains("component"));
  }
}
```

## Complex Documents

### Full page structure

```temper
test "parses complex document" {
  let html = "
    <div class=\"container\">
      <.header title={@title}>
        <:nav>
          <a href=\"/\">Home</a>
        </:nav>
      </.header>
      <%= if @user do %>
        <p>Welcome, {@user.name}!</p>
      <% else %>
        <.login_form />
      <% end %>
    </div>
  ";
  let doc = parse(html);
  // Should parse without error
  assert(doc.children.length >= 1);
}
```
