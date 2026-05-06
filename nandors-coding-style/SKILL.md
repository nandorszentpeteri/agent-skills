---
name: nandors-coding-style
description: >-
  Functional coding conventions: factory functions over classes, immutability,
  pure functions, currying, arrow functions, one-liners. Load proactively
  whenever writing, modifying, reviewing, or refactoring code — applies to every
  code change automatically.
---

# Nandor's Coding Style

A functional approach to writing code, inspired by Eric Elliott's "Composing Software" and functional programming paradigms. The core belief: **composition over inheritance**, **functions over objects**, **immutability over mutation**.

## Language-Specific Guides

Apply the core rules below in all code. When working in a specific language, also apply its guide:

- **TypeScript / JavaScript**: See `typescript.md`

## Core Rules

### Functions Over Classes

Prefer functions, modules, and factory functions. Avoid classes, constructors, `new`, and `this`.

```javascript
// Do
const createUser = (name, email) => ({
  name,
  email,
  greet: () => `Hello, ${name}`,
});

// Don't
class User {
  constructor(name, email) {
    this.name = name;
    this.email = email;
  }
  greet() {
    return `Hello, ${this.name}`;
  }
}
```

### Arrow / Lambda Functions

Prefer arrow or lambda syntax over named function declarations.

```javascript
// Do
const add = (a, b) => a + b;

const processItems = (items) => items.filter((item) => item.active).map((item) => item.name);

// Don't
function add(a, b) {
  return a + b;
}
```

### Immutability

Never mutate arguments or shared state. Create new copies using spread, destructuring, or language-native immutable patterns.

```javascript
// Do
const updateUser = (user, name) => ({ ...user, name });

const addItem = (list, item) => [...list, item];

// Don't
const updateUser = (user, name) => {
  user.name = name;
  return user;
};
```

### Pure Functions

Same input = same output, no side effects. Isolate side effects at the boundaries of your system.

```javascript
// Do
const calculateTotal = (items) => items.reduce((sum, item) => sum + item.price, 0);

// Don't
let runningTotal = 0;
const addToTotal = (item) => {
  runningTotal += item.price;
  return runningTotal;
};
```

### Currying and Partial Application

Use currying or partial application to share scope and create specialized functions.

```javascript
// Do
const log = (level) => (message) => console.log(`[${level}] ${message}`);

const warn = log('WARN');
const error = log('ERROR');

const fetchFrom = (baseUrl) => (path) => fetch(`${baseUrl}${path}`);

// Don't
const log = (level, message) => console.log(`[${level}] ${message}`);
```

### One-Liners

Keep functions concise. If it fits clearly on one line, write it as one.

```javascript
const isActive = (user) => user.status === 'active';
const getName = (user) => user.name;
const toUpper = (str) => str.toUpperCase();
const head = (arr) => arr[0];
const prop = (key) => (obj) => obj[key];
```

### Import Separation

Group 3rd-party/global imports first, then a blank line, then local/project imports. Within each group, keep imports in **alphabetical order** unless a language addendum overrides grouping (see **React import order** in `typescript.md`: **context → hooks → rest** within the local block comes first; **then** alphabetical order **within** each of those subgroups).

```javascript
import dayjs from 'dayjs';
import { useState, useEffect } from 'react';

import { AppLeaf } from 'types/common/leaf';
import { formatLeaves } from 'modules/leaves';
```

## Testing

Unit testing is essential. Test atomic values — one behavior per test.

- Write pure functions first; they are the easiest to test without mocks
- Each test should verify a single behavior or outcome
- Keep test setup minimal and isolated
- Prefer simple assertions over complex mock chains
- Use fake timers and minimal mocks where side effects are unavoidable

```javascript
// Do — atomic, isolated tests
it('should calculate total for items', () => {
  const items = [{ price: 10 }, { price: 20 }];
  expect(calculateTotal(items)).toBe(30);
});

it('should return 0 for empty list', () => {
  expect(calculateTotal([])).toBe(0);
});

// Don't — testing multiple behaviors in one test
it('should work correctly', () => {
  expect(calculateTotal([{ price: 10 }])).toBe(10);
  expect(calculateTotal([])).toBe(0);
  expect(calculateTotal([{ price: 5 }, { price: 5 }])).toBe(10);
});
```

## Anti-Patterns

Avoid the following:

- **Classes, `new`, `this`** — use factory functions and modules instead
- **Deep inheritance chains** — compose behavior through functions
- **Mutable shared state** — create new copies, isolate side effects
- **`function` declarations** — use arrow/lambda functions
- **Over-mocking in tests** — write pure functions that don't need mocks
