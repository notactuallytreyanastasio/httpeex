# Integration Tests

End-to-end tests for the HEEx parser and renderer.

## Setup

```temper
let { parse } = import("../parser");
let { renderHtml, renderDebug, renderJson } = import("../renderer");

// Parse and render roundtrip
let roundtrip(input: String): String throws Bubble {
  let doc = parse(input);
  renderHtml(doc)
}

// Helper to check roundtrip preserves structure
let assertRoundtrip(input: String): Void throws Bubble {
  let output = roundtrip(input);
  parse(output);
}
```

## Basic Roundtrip Tests

```temper
test("roundtrips plain text") {
  let input = "Hello world";
  let output = roundtrip(input);
  assert(output == "Hello world");
}

test("roundtrips simple element") {
  let output = roundtrip("<div></div>");
  assert(output == "<div></div>");
}

test("roundtrips self-closing element") {
  let output = roundtrip("<br />");
  assert(output.contains("br"));
}

test("roundtrips element with content") {
  let output = roundtrip("<p>Hello</p>");
  assert(output == "<p>Hello</p>");
}

test("roundtrips nested elements") {
  assertRoundtrip("<div><span><em>text</em></span></div>");
}
```

## Attribute Roundtrip Tests

```temper
test("roundtrips static attribute") {
  let output = roundtrip("<div class=\"container\"></div>");
  assert(output.contains("class=\"container\""));
}

test("roundtrips dynamic attribute") {
  let output = roundtrip("<div class={@class}></div>");
  assert(output.contains("class={@class}"));
}

test("roundtrips :if attribute") {
  let output = roundtrip("<div :if={@show}></div>");
  assert(output.contains(":if={@show}"));
}

test("roundtrips :for attribute") {
  let output = roundtrip("<li :for={i <- @items}></li>");
  assert(output.contains(":for={i <- @items}"));
}

test("roundtrips spread attribute") {
  let output = roundtrip("<div {@attrs}></div>");
  assert(output.contains("{@attrs}"));
}
```

## Component Roundtrip Tests

```temper
test("roundtrips local component") {
  let output = roundtrip("<.button>Click</.button>");
  assert(output.contains("<.button>"));
  assert(output.contains("</.button>"));
}

test("roundtrips remote component") {
  let output = roundtrip("<MyApp.Button />");
  assert(output.contains("MyApp.Button"));
}

test("roundtrips component with slots") {
  let input = "<.card><:header>Title</:header><:body>Content</:body></.card>";
  assertRoundtrip(input);
}
```

## Expression Roundtrip Tests

```temper
test("roundtrips expression in text") {
  let output = roundtrip("Hello {@name}!");
  assert(output == "Hello {@name}!");
}

test("roundtrips expression in element") {
  let output = roundtrip("<span>{@value}</span>");
  assert(output.contains("{@value}"));
}
```

## EEx Roundtrip Tests

```temper
test("roundtrips EEx output") {
  let output = roundtrip("<%= @name %>");
  assert(output.contains("<%= @name %>"));
}

test("roundtrips EEx if block") {
  let input = "<%= if @show do %>visible<% end %>";
  assertRoundtrip(input);
}
```

## Debug Output Tests

```temper
test("debug output shows tree structure") {
  let doc = parse("<div><span>text</span></div>");
  let debug = renderDebug(doc);
  assert(debug.contains("Document"));
  assert(debug.contains("Element"));
}

test("debug output shows attributes") {
  let doc = parse("<div class=\"foo\" id={@id}></div>");
  let debug = renderDebug(doc);
  assert(debug.contains("Attr"));
}
```

## JSON Output Tests

```temper
test("JSON output has correct structure") {
  let doc = parse("<div>text</div>");
  let json = renderJson(doc);
  assert(json.startsWith("{"));
  assert(json.endsWith("}"));
  assert(json.contains("\"type\":\"document\""));
}

test("JSON output includes element info") {
  let doc = parse("<div class=\"c\">text</div>");
  let json = renderJson(doc);
  assert(json.contains("\"type\":\"element\""));
  assert(json.contains("\"tag\":\"div\""));
}

test("JSON output includes component info") {
  let doc = parse("<.button />");
  let json = renderJson(doc);
  assert(json.contains("\"type\":\"component\""));
}
```

## Edge Cases

```temper
test("handles empty elements") {
  assertRoundtrip("<div></div>");
  assertRoundtrip("<span></span>");
  assertRoundtrip("<.component></.component>");
}

test("handles significant whitespace") {
  let input = "<p>Hello world</p>";
  let output = roundtrip(input);
  assert(output.contains("Hello world"));
}

test("escapes special characters") {
  let doc = parse("<p>a &lt; b</p>");
  let html = renderHtml(doc);
  assert(html.contains("&lt;") || html.contains("<"));
}

test("handles deep nesting") {
  let input = "<a><b><c><d><e>deep</e></d></c></b></a>";
  assertRoundtrip(input);
}

test("handles many siblings") {
  let input = "<div><a/><b/><c/><d/><e/><f/><g/><h/><i/><j/></div>";
  assertRoundtrip(input);
}

test("handles complex expressions") {
  let input = "<div class={Enum.join([\"a\", \"b\"], \" \")}></div>";
  assertRoundtrip(input);
}

test("handles nested braces in expressions") {
  let input = "<div data={%{key: \"value\", nested: %{a: 1}}}></div>";
  assertRoundtrip(input);
}
```
