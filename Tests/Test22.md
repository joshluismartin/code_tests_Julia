# Test 22 — JavaScript Basics 1

**Rules:** Don't look at Lesson22.md while answering. You *may* run code with `node -e` to check yourself **after** writing your answer. Then check `Answers22.md`.
**Total:** 36 points

---

## Part A — Multiple choice (1 point each)

**1.** Which keyword should you use by default for a variable in freee-payroll's frontend?
- a) `var`
- b) `let`
- c) `const`
- d) `def`

**2.** What does `undefined` usually mean?
- a) The value is zero
- b) The value was never set / the property doesn't exist
- c) The value is an empty string
- d) The same as `false`

**3.** What does this return? `const f = (x) => { x + 1 };  f(1)`
- a) `2`
- b) `1`
- c) `undefined`
- d) An error

**4.** What is an *arrow function*?
- a) A function written as `(params) => body`
- b) A function that only runs in the browser
- c) A function that points to another file
- d) A Ruby block

**5.** In `import * as API from '@freee-payroll/api';`, what is `API`?
- a) A default export
- b) A single named export called `API`
- c) A namespace object holding everything the module exports
- d) A type-only import

**6.** What does `{ headers }` mean inside an object literal?
- a) An array containing `headers`
- b) `{ headers: headers }` (shorthand property)
- c) A destructuring
- d) A syntax error

**7.** `module.exports = { ... }` is an example of:
- a) ES modules
- b) CommonJS, the older Node module system
- c) TypeScript types
- d) JSON

**8.** Which is the JavaScript equivalent of Ruby's `"Hi #{name}"`?
- a) `'Hi #{name}'`
- b) `"Hi ${name}"`
- c) `` `Hi ${name}` ``
- d) `'Hi ' + #name`

---

## Part B — Fill in the blanks (1 point each blank)

**9.** A variable declared with ______ can be reassigned; one declared with ______ cannot.

**10.** In JS, array indexes start at ______, and reading an index that doesn't exist gives ______.

**11.** The `...` operator used as `{ ...state, isOpen: false }` is called ______. It creates a ______ object instead of changing the original.

**12.** A file can have at most ______ default export(s), but any number of ______ exports.

**13.** `import type { Foo } from './types';` imports only a ______, which disappears when the code is built.

---

## Part C — Fill in the missing code (2 points each)

**14.** Complete the arrow function so it returns the **object** `{ name: n }` in one line:

```js
const makeUser = (n) => ______;
```

**15.** Use destructuring to take `data` out of `response` in one line:

```js
const response = { data: [1, 2, 3], status: 200 };
______ = response;
console.log(data.length); // 3
```

**16.** Complete the import so that `Routes` (default export of `./Routes`) and `run` (default export of `../initializers/run`) are both available:

```ts
import ______ from '../initializers/run';

import ______ from './Routes';
```

---

## Part D — Write the code (3 points each)

**17.** Write an arrow function `fullName` that takes an object with `lastName` and `firstName` (use **parameter destructuring**) and returns them joined with a space using a **template literal**.
`fullName({ lastName: '給与', firstName: '太郎' })` → `'給与 太郎'`

**18.** Given

```js
const state = { isLoading: true, rows: [1, 2] };
```

write one line that creates `nextState`, a copy of `state` where `isLoading` is `false` and `rows` is unchanged, **without modifying `state`**.

---

## Part E — Answer in a sentence or two (2 points each)

**19.** What is the difference between `null` and `undefined`?

**20.** A Ruby developer writes `const total = (a, b) => { a + b };` and is surprised it returns `undefined`. Explain why and give the fix.

---

## Part F — Read real repo code (3 points)

**21.** From `front/packages/@freee-payroll/api/src/TaxOffice.ts:7-12`:

```ts
type IndexResponse = { models: Array<TaxOffice> };

export const index = async (pref: string) => {
  const { data } = await axios.get<IndexResponse>(`/api/p/tax_offices/${pref}`);
  return data.models;
};
```

Ignoring the words `type`, `async`, `await`, `<IndexResponse>` and `: string` (you'll learn them later), answer:
(a) Is `index` a named or default export?
(b) If `pref` is `'13'`, what URL is requested?
(c) What does `const { data } = ...` do, and what does the function return?
