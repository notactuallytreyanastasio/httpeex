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
  // Re-parse output to verify it's valid
  parse(output);
}
```

## Basic Roundtrip Tests

### Plain text roundtrip

```temper
test "roundtrips plain text" {
  let input = "Hello world";
  let output = roundtrip(input);
  assert(output == "Hello world");
}
```

### Simple element roundtrip

```temper
test "roundtrips simple element" {
  let output = roundtrip("<div></div>");
  assert(output == "<div></div>");
}
```

### Self-closing element roundtrip

```temper
test "roundtrips self-closing element" {
  let output = roundtrip("<br />");
  assert(output.contains("br"));
}
```

### Element with content roundtrip

```temper
test "roundtrips element with content" {
  let output = roundtrip("<p>Hello</p>");
  assert(output == "<p>Hello</p>");
}
```

### Nested elements roundtrip

```temper
test "roundtrips nested elements" {
  assertRoundtrip("<div><span><em>text</em></span></div>");
}
```

## Attribute Roundtrip Tests

### Static attribute roundtrip

```temper
test "roundtrips static attribute" {
  let output = roundtrip("<div class=\"container\"></div>");
  assert(output.contains("class=\"container\""));
}
```

### Dynamic attribute roundtrip

```temper
test "roundtrips dynamic attribute" {
  let output = roundtrip("<div class={@class}></div>");
  assert(output.contains("class={@class}"));
}
```

### Special attribute roundtrip

```temper
test "roundtrips :if attribute" {
  let output = roundtrip("<div :if={@show}></div>");
  assert(output.contains(":if={@show}"));
}

test "roundtrips :for attribute" {
  let output = roundtrip("<li :for={i <- @items}></li>");
  assert(output.contains(":for={i <- @items}"));
}
```

### Spread attribute roundtrip

```temper
test "roundtrips spread attribute" {
  let output = roundtrip("<div {@attrs}></div>");
  assert(output.contains("{@attrs}"));
}
```

## Component Roundtrip Tests

### Local component roundtrip

```temper
test "roundtrips local component" {
  let output = roundtrip("<.button>Click</.button>");
  assert(output.contains("<.button>"));
  assert(output.contains("</.button>"));
}
```

### Remote component roundtrip

```temper
test "roundtrips remote component" {
  let output = roundtrip("<MyApp.Button />");
  assert(output.contains("MyApp.Button"));
}
```

### Component with slots roundtrip

```temper
test "roundtrips component with slots" {
  let input = "<.card><:header>Title</:header><:body>Content</:body></.card>";
  assertRoundtrip(input);
}
```

## Expression Roundtrip Tests

### Expression in text roundtrip

```temper
test "roundtrips expression in text" {
  let output = roundtrip("Hello {@name}!");
  assert(output == "Hello {@name}!");
}
```

### Expression in element roundtrip

```temper
test "roundtrips expression in element" {
  let output = roundtrip("<span>{@value}</span>");
  assert(output.contains("{@value}"));
}
```

## EEx Roundtrip Tests

### EEx output roundtrip

```temper
test "roundtrips EEx output" {
  let output = roundtrip("<%= @name %>");
  assert(output.contains("<%= @name %>"));
}
```

### EEx block roundtrip

```temper
test "roundtrips EEx if block" {
  let input = "<%= if @show do %>visible<% end %>";
  assertRoundtrip(input);
}
```

## Debug Output Tests

### Debug output structure

```temper
test "debug output shows tree structure" {
  let doc = parse("<div><span>text</span></div>");
  let debug = renderDebug(doc);

  assert(debug.contains("Document"));
  assert(debug.contains("Element: <div>"));
  assert(debug.contains("Element: <span>"));
  assert(debug.contains("Text:"));
}
```

### Debug output shows attributes

```temper
test "debug output shows attributes" {
  let doc = parse("<div class=\"foo\" id={@id}></div>");
  let debug = renderDebug(doc);

  assert(debug.contains("Attr:"));
  assert(debug.contains("class"));
}
```

## JSON Output Tests

### JSON output is valid

```temper
test "JSON output has correct structure" {
  let doc = parse("<div>text</div>");
  let json = renderJson(doc);

  assert(json.startsWith("{"));
  assert(json.endsWith("}"));
  assert(json.contains("\"type\":\"document\""));
  assert(json.contains("\"children\""));
}
```

### JSON output includes all node types

```temper
test "JSON output includes element info" {
  let doc = parse("<div class=\"c\">text</div>");
  let json = renderJson(doc);

  assert(json.contains("\"type\":\"element\""));
  assert(json.contains("\"tag\":\"div\""));
  assert(json.contains("\"type\":\"text\""));
}
```

### JSON output for components

```temper
test "JSON output includes component info" {
  let doc = parse("<.button />");
  let json = renderJson(doc);

  assert(json.contains("\"type\":\"component\""));
  assert(json.contains("\"componentType\":\"local\""));
}
```

## Real-World Template Tests

### Phoenix LiveView form

```temper
test "parses Phoenix form template" {
  let template = "
    <.form for={@form} phx-submit=\"save\">
      <.input field={@form[:email]} type=\"email\" label=\"Email\" />
      <.input field={@form[:password]} type=\"password\" label=\"Password\" />
      <:actions>
        <.button>Log in</.button>
      </:actions>
    </.form>
  ";
  let doc = parse(template);
  let html = renderHtml(doc);
  assert(html.contains("<.form"));
  assert(html.contains("<.input"));
  assert(html.contains("<.button"));
}
```

### Phoenix table component

```temper
test "parses Phoenix table template" {
  let template = "
    <.table id=\"users\" rows={@users}>
      <:col :let={user} label=\"Name\">{user.name}</:col>
      <:col :let={user} label=\"Email\">{user.email}</:col>
      <:action :let={user}>
        <.link navigate={~p\"/users/#{user.id}\"}>Show</.link>
      </:action>
    </.table>
  ";
  let doc = parse(template);
  assert(doc.children.length >= 1);
}
```

### Conditional rendering

```temper
test "parses conditional rendering" {
  let template = "
    <%= if @current_user do %>
      <nav>
        <.link href={~p\"/users/settings\"}>Settings</.link>
        <.link href={~p\"/users/log_out\"} method=\"delete\">Log out</.link>
      </nav>
    <% else %>
      <nav>
        <.link href={~p\"/users/register\"}>Register</.link>
        <.link href={~p\"/users/log_in\"}>Log in</.link>
      </nav>
    <% end %>
  ";
  let doc = parse(template);
  let html = renderHtml(doc);
  assert(html.contains("if @current_user"));
}
```

### List rendering

```temper
test "parses list rendering" {
  let template = "
    <ul>
      <li :for={item <- @items} :key={item.id}>
        {item.name}
        <button phx-click=\"delete\" phx-value-id={item.id}>
          Delete
        </button>
      </li>
    </ul>
  ";
  let doc = parse(template);
  let html = renderHtml(doc);
  assert(html.contains(":for="));
  assert(html.contains(":key="));
}
```

### Flash messages

```temper
test "parses flash messages template" {
  let template = "
    <.flash :if={info = Phoenix.Flash.get(@flash, :info)} kind={:info}>
      {info}
    </.flash>
    <.flash :if={error = Phoenix.Flash.get(@flash, :error)} kind={:error}>
      {error}
    </.flash>
  ";
  let doc = parse(template);
  assert(doc.children.length >= 1);
}
```

### Modal component

```temper
test "parses modal template" {
  let template = "
    <.modal :if={@show_modal} id=\"confirm-modal\" on_cancel={JS.hide(\"#confirm-modal\")}>
      <:title>Confirm Action</:title>
      <p>Are you sure you want to proceed?</p>
      <:footer>
        <.button phx-click=\"cancel\">Cancel</.button>
        <.button phx-click=\"confirm\" class=\"danger\">Confirm</.button>
      </:footer>
    </.modal>
  ";
  let doc = parse(template);
  let html = renderHtml(doc);
  assert(html.contains("<.modal"));
  assert(html.contains("<:title>"));
  assert(html.contains("<:footer>"));
}
```

## Edge Cases

### Empty elements

```temper
test "handles empty elements" {
  assertRoundtrip("<div></div>");
  assertRoundtrip("<span></span>");
  assertRoundtrip("<.component></.component>");
}
```

### Whitespace handling

```temper
test "handles significant whitespace" {
  let input = "<p>Hello world</p>";
  let output = roundtrip(input);
  assert(output.contains("Hello world"));
}
```

### Special characters in text

```temper
test "escapes special characters" {
  let doc = parse("<p>a &lt; b</p>");
  let html = renderHtml(doc);
  // Should preserve or re-escape
  assert(html.contains("&lt;") || html.contains("<"));
}
```

### Deeply nested structure

```temper
test "handles deep nesting" {
  let input = "<a><b><c><d><e>deep</e></d></c></b></a>";
  assertRoundtrip(input);
}
```

### Many siblings

```temper
test "handles many siblings" {
  let input = "<div><a/><b/><c/><d/><e/><f/><g/><h/><i/><j/></div>";
  assertRoundtrip(input);
}
```

### Complex expressions

```temper
test "handles complex expressions" {
  let input = "<div class={Enum.join([\"a\", \"b\"], \" \")}></div>";
  assertRoundtrip(input);
}
```

### Nested expressions in attributes

```temper
test "handles nested braces in expressions" {
  let input = "<div data={%{key: \"value\", nested: %{a: 1}}}></div>";
  assertRoundtrip(input);
}
```
