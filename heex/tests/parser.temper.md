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

let firstChild(doc: Document): Node {
  doc.children[0]
}

let assertNodeType(node: Node, expected: String): Void throws Bubble {
  if (node.nodeType() != expected) {
    throw new Bubble("Expected ${expected}, got ${node.nodeType()}");
  }
}

let assertChildCount(doc: Document, expected: Int): Void throws Bubble {
  if (doc.children.length != expected) {
    throw new Bubble("Expected ${expected} children, got ${doc.children.length}");
  }
}
```

## Text Parsing

```temper
test("parses plain text") {
  let doc = parse("Hello world");
  assertChildCount(doc, 1);
  let text = firstChild(doc) as Text;
  assert(text.content == "Hello world");
}

test("preserves whitespace") {
  let doc = parse("  hello  ");
  let text = firstChild(doc) as Text;
  assert(text.content == "  hello  ");
}

test("parses empty document") {
  let doc = parse("");
  assertChildCount(doc, 0);
}
```

## Element Parsing

```temper
test("parses simple element") {
  let doc = parse("<div></div>");
  assertChildCount(doc, 1);
  let el = firstChild(doc) as Element;
  assert(el.tag == "div");
  assert(el.children.length == 0);
  assert(el.selfClosing == false);
}

test("parses self-closing element") {
  let doc = parse("<br/>");
  let el = firstChild(doc) as Element;
  assert(el.tag == "br");
  assert(el.selfClosing == true);
}

test("parses void element without closing tag") {
  let doc = parse("<input>");
  let el = firstChild(doc) as Element;
  assert(el.tag == "input");
}

test("parses nested elements") {
  let doc = parse("<div><span></span></div>");
  let div = firstChild(doc) as Element;
  assert(div.tag == "div");
  assert(div.children.length == 1);
  let span = div.children[0] as Element;
  assert(span.tag == "span");
}

test("parses element with text content") {
  let doc = parse("<p>Hello</p>");
  let p = firstChild(doc) as Element;
  assert(p.children.length == 1);
  let text = p.children[0] as Text;
  assert(text.content == "Hello");
}

test("parses deeply nested elements") {
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

```temper
test("parses static attribute") {
  let doc = parse("<div class=\"container\"></div>");
  let el = firstChild(doc) as Element;
  assert(el.attributes.length == 1);
  let attr = el.attributes[0] as StaticAttribute;
  assert(attr.name == "class");
  assert(attr.value == "container");
}

test("parses multiple attributes") {
  let doc = parse("<div id=\"main\" class=\"wrapper\"></div>");
  let el = firstChild(doc) as Element;
  assert(el.attributes.length == 2);
}

test("parses boolean attribute") {
  let doc = parse("<input disabled>");
  let el = firstChild(doc) as Element;
  let attr = el.attributes[0] as StaticAttribute;
  assert(attr.name == "disabled");
  assert(attr.value == "true");
}

test("parses dynamic attribute") {
  let doc = parse("<div class={@class}></div>");
  let el = firstChild(doc) as Element;
  let attr = el.attributes[0] as DynamicAttribute;
  assert(attr.name == "class");
  assert(attr.expression.code == "@class");
}

test("parses spread attribute") {
  let doc = parse("<div {@attrs}></div>");
  let el = firstChild(doc) as Element;
  let attr = el.attributes[0] as SpreadAttribute;
  assert(attr.expression.code == "@attrs");
}

test("parses :if attribute") {
  let doc = parse("<div :if={@show}></div>");
  let el = firstChild(doc) as Element;
  let attr = el.attributes[0] as SpecialAttribute;
  assert(attr.kind == "if");
  assert(attr.expression.code == "@show");
}

test("parses :for attribute") {
  let doc = parse("<li :for={item <- @items}></li>");
  let el = firstChild(doc) as Element;
  let attr = el.attributes[0] as SpecialAttribute;
  assert(attr.kind == "for");
  assert(attr.expression.code == "item <- @items");
}

test("parses :key attribute") {
  let doc = parse("<li :key={item.id}></li>");
  let el = firstChild(doc) as Element;
  let attr = el.attributes[0] as SpecialAttribute;
  assert(attr.kind == "key");
}
```

## Component Parsing

```temper
test("parses local component") {
  let doc = parse("<.button></.button>");
  let comp = firstChild(doc) as Component;
  assert(comp.name == ".button");
  assert(comp.componentType.kind == "local");
}

test("parses self-closing local component") {
  let doc = parse("<.icon />");
  let comp = firstChild(doc) as Component;
  assert(comp.name == ".icon");
}

test("parses remote component") {
  let doc = parse("<MyApp.Button></MyApp.Button>");
  let comp = firstChild(doc) as Component;
  assert(comp.name == "MyApp.Button");
  assert(comp.componentType.kind == "remote");
}

test("parses component with attributes") {
  let doc = parse("<.button type=\"submit\" disabled></.button>");
  let comp = firstChild(doc) as Component;
  assert(comp.attributes.length == 2);
}

test("parses component with children") {
  let doc = parse("<.card>Content</.card>");
  let comp = firstChild(doc) as Component;
  assert(comp.children.length == 1);
  let text = comp.children[0] as Text;
  assert(text.content == "Content");
}
```

## Slot Parsing

```temper
test("parses named slot") {
  let doc = parse("<.card><:header>Title</:header></.card>");
  let comp = firstChild(doc) as Component;
  assert(comp.slots.length == 1);
  let slot = comp.slots[0];
  assert(slot.name == "header");
  assert(slot.children.length == 1);
}

test("parses multiple slots") {
  let doc = parse("<.card><:header>H</:header><:body>B</:body></.card>");
  let comp = firstChild(doc) as Component;
  assert(comp.slots.length == 2);
}

test("parses slot with :let binding") {
  let doc = parse("<.table><:col :let={value}>{value}</:col></.table>");
  let comp = firstChild(doc) as Component;
  let slot = comp.slots[0];
  assert(slot.letBinding == "value");
}

test("parses self-closing slot") {
  let doc = parse("<.tabs><:tab label=\"One\" /></.tabs>");
  let comp = firstChild(doc) as Component;
  assert(comp.slots.length == 1);
}

test("parses component with both children and slots") {
  let doc = parse("<.modal>Default content<:footer>Buttons</:footer></.modal>");
  let comp = firstChild(doc) as Component;
  assert(comp.children.length == 1);
  assert(comp.slots.length == 1);
}
```

## Expression Parsing

```temper
test("parses expression in text") {
  let doc = parse("Hello {@name}!");
  assertChildCount(doc, 3);
  let expr = doc.children[1] as Expression;
  assert(expr.code == "@name");
}

test("parses expression in element") {
  let doc = parse("<span>{@value}</span>");
  let span = firstChild(doc) as Element;
  let expr = span.children[0] as Expression;
  assert(expr.code == "@value");
}
```

## EEx Parsing

```temper
test("parses EEx output") {
  let doc = parse("<%= @name %>");
  let eex = firstChild(doc) as EEx;
  assert(eex.eexType.kind == "output");
  assert(eex.code == "@name");
}

test("parses EEx exec") {
  let doc = parse("<% x = 1 %>");
  let eex = firstChild(doc) as EEx;
  assert(eex.eexType.kind == "exec");
}

test("parses EEx comment") {
  let doc = parse("<%# comment %>");
  let eex = firstChild(doc) as EEx;
  assert(eex.eexType.kind == "comment");
}

test("parses EEx if block") {
  let doc = parse("<%= if @show do %>visible<% end %>");
  let block = firstChild(doc) as EExBlock;
  assert(block.blockType == "if");
  assert(block.clauses.length >= 1);
}

test("parses EEx if/else block") {
  let doc = parse("<%= if @show do %>yes<% else %>no<% end %>");
  let block = firstChild(doc) as EExBlock;
  assert(block.blockType == "if");
}

test("parses EEx for block") {
  let doc = parse("<%= for item <- @items do %><li>{item}</li><% end %>");
  let block = firstChild(doc) as EExBlock;
  assert(block.blockType == "for");
}
```

## Comment Parsing

```temper
test("parses HTML comment") {
  let doc = parse("<!-- comment -->");
  let comment = firstChild(doc) as Comment;
  assert(comment.content == " comment ");
}

test("parses comment between elements") {
  let doc = parse("<div></div><!-- separator --><span></span>");
  assertChildCount(doc, 3);
  assertNodeType(doc.children[1], "comment");
}
```

## Error Handling

```temper
let shouldBubble(action: fn(): Void throws Bubble): Boolean {
  do {
    action();
    false
  } orelse true
}

test("errors on mismatched tags") {
  assert(shouldBubble { parse("<div></span>"); });
}

test("errors on unclosed tag") {
  assert(shouldBubble { parse("<div><span></div>"); });
}

test("errors on mismatched component close") {
  assert(shouldBubble { parse("<.button></.other>"); });
}
```

## Complex Documents

```temper
test("parses complex document") {
  let html = "<div class=\"container\"><.header title={@title}><:nav><a href=\"/\">Home</a></:nav></.header></div>";
  let doc = parse(html);
  assert(doc.children.length >= 1);
}
```
