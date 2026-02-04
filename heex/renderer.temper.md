# HEEx Renderer

The renderer converts an AST back into various output formats.
This is the third stage of the pipeline.

## Overview

The renderer supports multiple output modes:

1. **HTML String** - Plain HTML output (expressions as placeholders)
2. **Debug String** - AST visualization for development
3. **JSON** - Serialized AST for tooling integration

## Imports

```temper
let {
  Node, Document, Text, Element, Component, ComponentType, Slot,
  Expression, Attribute, StaticAttribute, DynamicAttribute,
  SpreadAttribute, SpecialAttribute, EEx, EExType, EExBlock,
  EExClause, Comment, isVoidElement
} = import("./ast");
```

## HTML Rendering

Render the AST to an HTML string.

```temper
export let renderHtml(doc: Document): String {
  let out = new StringBuilder();
  for (let child of doc.children) {
    renderNode(child, out);
  }
  out.build()
}

let renderNode(node: Node, out: StringBuilder): Void {
  when (node) {
    is Text -> out.append(escapeHtml(node.content));
    is Element -> renderElement(node, out);
    is Component -> renderComponent(node, out);
    is Slot -> renderSlot(node, out);
    is Expression -> renderExpression(node, out);
    is EEx -> renderEEx(node, out);
    is EExBlock -> renderEExBlock(node, out);
    is Comment -> renderComment(node, out);
    else -> void;
  }
}
```

## Element Rendering

```temper
let renderElement(el: Element, out: StringBuilder): Void {
  out.append("<");
  out.append(el.tag);

  // Render attributes
  renderAttributes(el.attributes, out);

  // Self-closing or void element
  if (el.selfClosing || isVoidElement(el.tag)) {
    out.append(" />");
    return;
  }

  out.append(">");

  // Render children
  for (let child of el.children) {
    renderNode(child, out);
  }

  out.append("</");
  out.append(el.tag);
  out.append(">");
}
```

## Component Rendering

```temper
let renderComponent(comp: Component, out: StringBuilder): Void {
  out.append("<");
  out.append(comp.name);

  // Render attributes
  renderAttributes(comp.attributes, out);

  // No children or slots - self-close
  if (comp.children.length == 0 && comp.slots.length == 0) {
    out.append(" />");
    return;
  }

  out.append(">");

  // Render default slot children
  for (let child of comp.children) {
    renderNode(child, out);
  }

  // Render named slots
  for (let slot of comp.slots) {
    renderSlot(slot, out);
  }

  out.append("</");
  out.append(comp.name);
  out.append(">");
}
```

## Slot Rendering

```temper
let renderSlot(slot: Slot, out: StringBuilder): Void {
  out.append("<:");
  out.append(slot.name);

  // Render attributes
  renderAttributes(slot.attributes, out);

  if (slot.children.length == 0) {
    out.append(" />");
    return;
  }

  out.append(">");

  for (let child of slot.children) {
    renderNode(child, out);
  }

  out.append("</:");
  out.append(slot.name);
  out.append(">");
}
```

## Attribute Rendering

```temper
let renderAttributes(attrs: List<Attribute>, out: StringBuilder): Void {
  for (let attr of attrs) {
    out.append(" ");
    when (attr) {
      is StaticAttribute -> do {
        out.append(attr.name);
        out.append("=\"");
        out.append(escapeAttr(attr.value));
        out.append("\"");
      };
      is DynamicAttribute -> do {
        out.append(attr.name);
        out.append("={");
        out.append(attr.expression.code);
        out.append("}");
      };
      is SpreadAttribute -> do {
        out.append("{");
        out.append(attr.expression.code);
        out.append("}");
      };
      is SpecialAttribute -> do {
        out.append(":");
        out.append(attr.kind);
        out.append("={");
        out.append(attr.expression.code);
        out.append("}");
      };
      else -> void;
    }
  }
}
```

## Expression Rendering

```temper
let renderExpression(expr: Expression, out: StringBuilder): Void {
  out.append("{");
  out.append(expr.code);
  out.append("}");
}
```

## EEx Rendering

```temper
let renderEEx(eex: EEx, out: StringBuilder): Void {
  when (eex.eexType) {
    is { kind: "output" } -> do {
      out.append("<%= ");
      out.append(eex.code);
      out.append(" %>");
    };
    is { kind: "comment" } -> do {
      out.append("<%# ");
      out.append(eex.code);
      out.append(" %>");
    };
    else -> do {
      out.append("<% ");
      out.append(eex.code);
      out.append(" %>");
    };
  }
}

let renderEExBlock(block: EExBlock, out: StringBuilder): Void {
  out.append("<%= ");
  out.append(block.blockType);
  out.append(" ");
  out.append(block.expression);
  out.append(" do %>");

  for (let clause of block.clauses) {
    when (clause.clauseType) {
      is "do" -> do {
        for (let child of clause.children) {
          renderNode(child, out);
        }
      };
      is "else" -> do {
        out.append("<% else %>");
        for (let child of clause.children) {
          renderNode(child, out);
        }
      };
      is "end" -> do {
        out.append("<% end %>");
      };
      else -> do {
        out.append("<% ");
        out.append(clause.expression orelse "");
        out.append(" %>");
        for (let child of clause.children) {
          renderNode(child, out);
        }
      };
    }
  }
}
```

## Comment Rendering

```temper
let renderComment(comment: Comment, out: StringBuilder): Void {
  out.append("<!-- ");
  out.append(comment.content);
  out.append(" -->");
}
```

## HTML Escaping

```temper
let escapeHtml(s: String): String {
  let out = new StringBuilder();
  for (let i = 0; i < s.length; ++i) {
    let c = s.charAt(i);
    when (c) {
      is "&" -> out.append("&amp;");
      is "<" -> out.append("&lt;");
      is ">" -> out.append("&gt;");
      else -> out.append(c);
    }
  }
  out.build()
}

let escapeAttr(s: String): String {
  let out = new StringBuilder();
  for (let i = 0; i < s.length; ++i) {
    let c = s.charAt(i);
    when (c) {
      is "&" -> out.append("&amp;");
      is "\"" -> out.append("&quot;");
      is "<" -> out.append("&lt;");
      is ">" -> out.append("&gt;");
      else -> out.append(c);
    }
  }
  out.build()
}
```

## Debug Output

Pretty-print the AST for debugging.

```temper
export let renderDebug(doc: Document): String {
  let out = new StringBuilder();
  out.append("Document\n");
  for (let child of doc.children) {
    renderDebugNode(child, out, "  ");
  }
  out.build()
}

let renderDebugNode(node: Node, out: StringBuilder, indent: String): Void {
  out.append(indent);

  when (node) {
    is Text -> do {
      out.append("Text: \"");
      out.append(node.content.replace("\n", "\\n"));
      out.append("\"\n");
    };
    is Element -> do {
      out.append("Element: <");
      out.append(node.tag);
      out.append(">\n");
      for (let attr of node.attributes) {
        renderDebugAttr(attr, out, indent + "  ");
      }
      for (let child of node.children) {
        renderDebugNode(child, out, indent + "  ");
      }
    };
    is Component -> do {
      out.append("Component: ");
      out.append(node.name);
      out.append("\n");
      for (let attr of node.attributes) {
        renderDebugAttr(attr, out, indent + "  ");
      }
      for (let child of node.children) {
        renderDebugNode(child, out, indent + "  ");
      }
      for (let slot of node.slots) {
        renderDebugNode(slot, out, indent + "  ");
      }
    };
    is Slot -> do {
      out.append("Slot: <:");
      out.append(node.name);
      out.append(">\n");
      for (let child of node.children) {
        renderDebugNode(child, out, indent + "  ");
      }
    };
    is Expression -> do {
      out.append("Expression: {");
      out.append(node.code);
      out.append("}\n");
    };
    is EEx -> do {
      out.append("EEx: ");
      out.append(node.eexType.kind);
      out.append(" \"");
      out.append(node.code);
      out.append("\"\n");
    };
    is EExBlock -> do {
      out.append("EExBlock: ");
      out.append(node.blockType);
      out.append(" ");
      out.append(node.expression);
      out.append("\n");
      for (let clause of node.clauses) {
        out.append(indent + "  Clause: ");
        out.append(clause.clauseType);
        out.append("\n");
        for (let child of clause.children) {
          renderDebugNode(child, out, indent + "    ");
        }
      }
    };
    is Comment -> do {
      out.append("Comment: ");
      out.append(node.content);
      out.append("\n");
    };
    else -> do {
      out.append("Unknown node\n");
    };
  }
}

let renderDebugAttr(attr: Attribute, out: StringBuilder, indent: String): Void {
  out.append(indent);
  out.append("Attr: ");
  when (attr) {
    is StaticAttribute -> do {
      out.append(attr.name);
      out.append("=\"");
      out.append(attr.value);
      out.append("\"");
    };
    is DynamicAttribute -> do {
      out.append(attr.name);
      out.append("={");
      out.append(attr.expression.code);
      out.append("}");
    };
    is SpreadAttribute -> do {
      out.append("{");
      out.append(attr.expression.code);
      out.append("}");
    };
    is SpecialAttribute -> do {
      out.append(":");
      out.append(attr.kind);
      out.append("={");
      out.append(attr.expression.code);
      out.append("}");
    };
    else -> void;
  }
  out.append("\n");
}
```

## JSON Serialization

Serialize the AST to JSON for tooling.

```temper
export let renderJson(doc: Document): String {
  let out = new StringBuilder();
  out.append("{\"type\":\"document\",\"children\":[");
  let first = true;
  for (let child of doc.children) {
    if (!first) {
      out.append(",");
    }
    first = false;
    renderJsonNode(child, out);
  }
  out.append("]}");
  out.build()
}

let renderJsonNode(node: Node, out: StringBuilder): Void {
  when (node) {
    is Text -> do {
      out.append("{\"type\":\"text\",\"content\":");
      jsonString(node.content, out);
      out.append("}");
    };
    is Element -> do {
      out.append("{\"type\":\"element\",\"tag\":");
      jsonString(node.tag, out);
      out.append(",\"attributes\":[");
      let first = true;
      for (let attr of node.attributes) {
        if (!first) { out.append(","); }
        first = false;
        renderJsonAttr(attr, out);
      }
      out.append("],\"children\":[");
      first = true;
      for (let child of node.children) {
        if (!first) { out.append(","); }
        first = false;
        renderJsonNode(child, out);
      }
      out.append("]}");
    };
    is Component -> do {
      out.append("{\"type\":\"component\",\"name\":");
      jsonString(node.name, out);
      out.append(",\"componentType\":\"");
      out.append(node.componentType.kind);
      out.append("\",\"attributes\":[");
      let first = true;
      for (let attr of node.attributes) {
        if (!first) { out.append(","); }
        first = false;
        renderJsonAttr(attr, out);
      }
      out.append("],\"children\":[");
      first = true;
      for (let child of node.children) {
        if (!first) { out.append(","); }
        first = false;
        renderJsonNode(child, out);
      }
      out.append("],\"slots\":[");
      first = true;
      for (let slot of node.slots) {
        if (!first) { out.append(","); }
        first = false;
        renderJsonNode(slot, out);
      }
      out.append("]}");
    };
    is Slot -> do {
      out.append("{\"type\":\"slot\",\"name\":");
      jsonString(node.name, out);
      out.append(",\"children\":[");
      let first = true;
      for (let child of node.children) {
        if (!first) { out.append(","); }
        first = false;
        renderJsonNode(child, out);
      }
      out.append("]}");
    };
    is Expression -> do {
      out.append("{\"type\":\"expression\",\"code\":");
      jsonString(node.code, out);
      out.append("}");
    };
    is EEx -> do {
      out.append("{\"type\":\"eex\",\"eexType\":\"");
      out.append(node.eexType.kind);
      out.append("\",\"code\":");
      jsonString(node.code, out);
      out.append("}");
    };
    is Comment -> do {
      out.append("{\"type\":\"comment\",\"content\":");
      jsonString(node.content, out);
      out.append("}");
    };
    else -> out.append("null");
  }
}

let renderJsonAttr(attr: Attribute, out: StringBuilder): Void {
  when (attr) {
    is StaticAttribute -> do {
      out.append("{\"type\":\"static\",\"name\":");
      jsonString(attr.name, out);
      out.append(",\"value\":");
      jsonString(attr.value, out);
      out.append("}");
    };
    is DynamicAttribute -> do {
      out.append("{\"type\":\"dynamic\",\"name\":");
      jsonString(attr.name, out);
      out.append(",\"expression\":");
      jsonString(attr.expression.code, out);
      out.append("}");
    };
    is SpreadAttribute -> do {
      out.append("{\"type\":\"spread\",\"expression\":");
      jsonString(attr.expression.code, out);
      out.append("}");
    };
    is SpecialAttribute -> do {
      out.append("{\"type\":\"special\",\"kind\":");
      jsonString(attr.kind, out);
      out.append(",\"expression\":");
      jsonString(attr.expression.code, out);
      out.append("}");
    };
    else -> out.append("null");
  }
}

let jsonString(s: String, out: StringBuilder): Void {
  out.append("\"");
  for (let i = 0; i < s.length; ++i) {
    let c = s.charAt(i);
    when (c) {
      is "\"" -> out.append("\\\"");
      is "\\" -> out.append("\\\\");
      is "\n" -> out.append("\\n");
      is "\r" -> out.append("\\r");
      is "\t" -> out.append("\\t");
      else -> out.append(c);
    }
  }
  out.append("\"");
}
```
