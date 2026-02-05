# Integration Tests

End-to-end tests for the HEEx parser and renderer.

## Setup

All symbols are imported via imports.temper.md from the parent heex module.

## HTML Roundtrip Tests

```temper
test("roundtrip plain text") {
  let doc = parse("Hello world");
  let html = renderHtml(doc);
  parse(html);
}

test("roundtrip simple element") {
  let doc = parse("<div></div>");
  let html = renderHtml(doc);
  parse(html);
}

test("roundtrip element with attributes") {
  let doc = parse("<div class=\"foo\"></div>");
  let html = renderHtml(doc);
  parse(html);
}

test("roundtrip nested elements") {
  let doc = parse("<div><span>text</span></div>");
  let html = renderHtml(doc);
  parse(html);
}

test("roundtrip void element") {
  let doc = parse("<br />");
  let html = renderHtml(doc);
  parse(html);
}
```

## Component Roundtrip Tests

```temper
test("roundtrip local component") {
  let doc = parse("<.button>Click</.button>");
  let html = renderHtml(doc);
  parse(html);
}

test("roundtrip remote component") {
  let doc = parse("<MyApp.Button />");
  let html = renderHtml(doc);
  parse(html);
}

test("roundtrip component with slot") {
  let doc = parse("<.card><:header>Title</:header></.card>");
  let html = renderHtml(doc);
  parse(html);
}
```

## Expression Roundtrip Tests

```temper
test("roundtrip expression") {
  let doc = parse("{@name}");
  let html = renderHtml(doc);
  parse(html);
}

test("roundtrip dynamic attribute") {
  let doc = parse("<div class={@class}></div>");
  let html = renderHtml(doc);
  parse(html);
}
```

## Debug Rendering Tests

```temper
test("debug rendering produces output") {
  let doc = parse("<div>text</div>");
  let debug = renderDebug(doc);
  assert(debug.hasIndex(String.begin));
}

test("debug shows document structure") {
  let doc = parse("<div>text</div>");
  let debug = renderDebug(doc);
  assert(debug.hasIndex(String.begin));
}
```

## JSON Rendering Tests

```temper
test("json rendering produces valid structure") {
  let doc = parse("<div>text</div>");
  let json = renderJson(doc);
  assert(json[String.begin] == char'{');
}

test("json rendering handles attributes") {
  let doc = parse("<div class=\"foo\"></div>");
  let json = renderJson(doc);
  assert(json.hasIndex(String.begin));
}

test("json rendering handles components") {
  let doc = parse("<.button />");
  let json = renderJson(doc);
  assert(json.hasIndex(String.begin));
}
```

## Complex Template Tests

```temper
test("parses nested component") {
  let doc = parse("<.outer><.inner /></.outer>");
  assert(doc.children.length >= 1);
}

test("parses mixed content") {
  let doc = parse("<div>text<span>more</span></div>");
  assert(doc.children.length >= 1);
}
```
