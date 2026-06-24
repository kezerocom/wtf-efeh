# UFEH Documentation (wtf-ufeh)

## What is UFEH

UFEH is an extension of HTML that adds reactivity, componentization, and server‑side logic without requiring bundlers or additional configuration. Processing happens on the server and the result is delivered fully rendered to the client.

## Installation and usage

The `wtf-ufeh` package relies on Node.js and is installed globally via npm or pnpm:

```bash
npm install -g wtf-ufeh
# or
pnpm add -g wtf-ufeh
```

After installation the `wtf-ufeh` command is available. The main commands are:

- `wtf-ufeh serve <directory>` – starts the production server.
- `wtf-ufeh hot serve <directory>` – starts a development server with hot reload.

## How it works

The engine recursively scans the provided directory, respecting `.gitignore` rules to skip folders like `node_modules`. Every `.html` file is served as static content, except when it contains exactly one root element – either `<component>` or `<route>`. In that case the engine processes the file, executes its server‑side logic, and delivers the rendered page.

## File structure

A UFEH file is an `.html` document with a single root element: `<component>` or `<route>`. Both share the same internal structure.

### Component

Defines a reusable block that can be imported by other components or routes. The `name` attribute is an arbitrary unique identifier; no naming pattern is enforced – consistency during import is all that matters.

```html
<component name="my.Component">
  <imports> ... </imports>
  <properties> ... </properties>
  <template> ... </template>
  <methods> ... </methods>
  <effects> ... </effects>
</component>
```

### Route

Defines a page reachable by a URL. The `path` attribute can be a string or a JSON array of patterns, with dynamic segments in `{ }`. Routes never receive external properties.

```html
<route path="/">
  <!-- same internal sections as component -->
</route>
```

## Sections

### `<imports>`

Declares other components to be used.

```html
<imports>
  <import name="ui.Button" as="Button" />
</imports>
```

- `name` – the component name as defined in its `component name`. The engine locates the corresponding file during scanning.
- `as` – alias used in the template.

### `<properties>`

Declares all reactive data of the component as `<attribute>` elements. A type must always be provided.

```html
<properties>
  <attribute name="counter" type="number" value="0" />
  <attribute name="title" type="string" required="true" />
  <attribute name="color" type="string" value="'blue'" required="false" />
  <attribute name="onClick" type="(id: number): void" required="false" />
</properties>
```

- `type` – the attribute type (primitive, array, inline object, or function signature).
- `value` – initial value; for `required="false"` it serves as default.
- `required`:
  - absent → internal attribute, not settable from outside.
  - `"true"` → mandatory when using the component.
  - `"false"` → optional; if not provided, the value from `value` is used.
- In routes, `required` is ignored.

### `<template>`

Contains HTML markup with dynamic interpolation using `@{expression}`.

```html
<template>
  <div>
    <h1>@{title}</h1>
    <p>@{counter}</p>
    <button onClick="@{increment(2)}">+2</button>
  </div>
</template>
```

Any JavaScript expression can be written inside `@{}`, including function calls and arrow functions.

### `<methods>`

Groups functions that manipulate attributes. The body is plain JavaScript.

```html
<methods>
  <method name="increment" params="delta: number">
    counter += delta;
  </method>
</methods>
```

- `params` – optional, declares parameters with the syntax `name: type; ...`.
- Special method `$` (routes only): runs on the server before rendering. It receives `context: HttpContext`.
  ```html
  <method name="$" params="context: HttpContext">
    if (!context.request.user) {
      context.redirect('/login');
    }
  </method>
  ```
- Any method of any component can also accept `HttpContext` as a parameter; the engine injects the current context.

### `<effects>`

Grouped side effects.

```html
<effects>
  <effect>
    console.log('mounted');
  </effect>
  <effect proprieties="counter">
    console.log('counter changed:', counter);
  </effect>
  <effect proprieties="['counter', 'title']">
    console.log('something changed');
  </effect>
</effects>
```

- Without `proprieties`: runs once when the component mounts.
- `proprieties`: accepts a string (single dependency) or a JSON array with multiple names. The effect triggers whenever any of the dependencies change.
- Timers, listeners, and subscriptions are automatically cleaned up when the component unmounts.

### Control flow

Conditionals and loops use dedicated tags.

```html
<if condition="@{loggedIn}">
  <p>Welcome</p>
<else>
  <p>Please log in</p>
</else>
</if>

<ul>
  <for each="item" in="@{items}">
    <li key="@{item.id}">@{item.name}</li>
  </for>
</ul>
```

### `<slot />`

Enables content projection. In the component definition, `<slot />` is replaced by whatever content is placed inside the component tag when used.

```html
<!-- Card.html -->
<component name="ui.Card">
  <template>
    <div class="card"><slot /></div>
  </template>
</component>

<!-- Usage -->
<Card>
  <p>Card content</p>
</Card>
```

## Type system

Supported types in `attribute` and `params`:

- Primitives: `string`, `number`, `boolean`, `any`, `void`
- Arrays: `string[]`, `User[]`
- Inline objects: `{ name: string; age: number }` (separators `;` or `,`)
- Functions: `(param: type, ...): returnType` (e.g., `(email: string, password: string): void`)

## Routing and server‑side logic

URL parameters (e.g., `/{id}`) are captured and exposed as attributes in the template. For routes with multiple patterns and optional parameters, declare an attribute with a default value:

```html
<route path="['/{lang}', '/']">
  <properties>
    <attribute name="lang" type="string" value="'en'" />
  </properties>
</route>
```

The `$` method is the route guard. Through `HttpContext` you can:

- Redirect: `context.redirect(url, statusCode)`
- Return an error: `context.error(statusCode, message)`
- Access the request (`request`) and manipulate the response (`response`)

## Tailwind CSS

If a `tailwindcss.css` file exists at the project root, the engine automatically generates `/build/styles.css`. Include it in the `<head>`:

```html
<link rel="stylesheet" href="/build/styles.css">
```

## Docker

Example production container:

```dockerfile
FROM node:20-alpine
RUN npm install -g wtf-ufeh
WORKDIR /app
COPY . .
EXPOSE 3000
CMD ["wtf-ufeh", "serve", "."]
```

Build and run:

```bash
docker build -t my-site .
docker run -p 3000:3000 my-site
```

## Complete example

**ui/Button.html**

```html
<component name="ui.Button">
  <properties>
    <attribute name="onClick" type="(e: Event): void" required="false" />
  </properties>
  <template>
    <button onClick="@{onClick}"><slot /></button>
  </template>
</component>
```

**index.html**

```html
<route path="/">
  <imports>
    <import name="ui.Button" as="Button" />
  </imports>
  <properties>
    <attribute name="counter" type="number" value="0" />
  </properties>
  <template>
    <html>
      <head>
        <link rel="stylesheet" href="/build/styles.css">
      </head>
      <body class="p-4">
        <h1 class="text-xl">Counter: @{counter}</h1>
        <Button onClick="@{increment}" class="bg-blue-500 text-white px-4 py-2 rounded">
          +1
        </Button>
      </body>
    </html>
  </template>
  <methods>
    <method name="increment">counter += 1;</method>
  </methods>
</route>
```

Run `wtf-ufeh serve .` and open `http://localhost:3000`.
