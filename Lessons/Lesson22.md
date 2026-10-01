# Lesson 22 — JavaScript Basics 1: Values, Functions, Objects, Modules

**Time:** ~1 hour (40 min reading + practice, 20 min test)
**Goal:** Read basic JavaScript: variables, values, functions (especially **arrow functions**), objects, arrays, destructuring, spread, and `import`/`export`. These are the building blocks of every `.ts`/`.tsx` file in `front/`.
**You need:** A terminal (for `node -e`) and the repo open at `~/Josh_Files/freee-payroll/front/`.

> Why JavaScript before TypeScript? **TypeScript = JavaScript + types.** Every TypeScript file is JavaScript with extra type labels. If you can read the JavaScript part, the types are "just labels" on top (Lessons 24–25).

---

## 1. What is JavaScript?

| Term | Meaning in plain words |
|---|---|
| **JavaScript (JS)** | The programming language that browsers understand. Every interactive web page uses it. |
| **Node.js** | A program that runs JavaScript *outside* the browser (in your terminal). freee-payroll uses Node for building, linting, and testing the frontend. |
| **TypeScript (TS)** | JavaScript with **types** added. It is converted ("compiled") to plain JavaScript before the browser runs it. |
| **ECMAScript / ES2021** | The official name/version of the JS standard. `tsconfig.json` says `"target": "es2021"`: "produce JS that follows the 2021 standard." |
| **Statement** | One instruction, usually ending with `;` (semicolons are used everywhere in this repo because Prettier adds them). |
| **Expression** | A piece of code that produces a value, e.g. `1 + 2`, `name.length`. |
| **Console** | Where `console.log(...)` prints. In Node: your terminal. In the browser: DevTools → Console. |

Try it right now:

```bash
node -e "console.log('Hello from JS', 1 + 2)"
# → Hello from JS 3
```

### Ruby ↔ JavaScript cheat sheet (you already know Ruby!)

| Idea | Ruby | JavaScript |
|---|---|---|
| Print | `puts 'hi'` | `console.log('hi')` |
| "Nothing" | `nil` | `null` **and** `undefined` (two kinds!) |
| Variable | `name = 'Taro'` | `const name = 'Taro';` or `let name = 'Taro';` |
| Naming | `snake_case` | `camelCase` (e.g. `employeeCount`) |
| String interpolation | `"Hi #{name}"` | `` `Hi ${name}` `` (backticks!) |
| Method / function | `def add(a, b) a + b end` | `const add = (a, b) => a + b;` |
| Return | last line is returned automatically | must write `return` (except one-line arrow functions) |
| Hash | `{ id: 1 }` | Object `{ id: 1 }` |
| Symbol `:id` | `:id` | no everyday equivalent; plain strings `'id'` are used |
| Array | `[1, 2, 3]` | `[1, 2, 3]` |
| Block | `list.map { \|x\| x * 2 }` | `list.map((x) => x * 2)` |
| Safe navigation | `user&.name` | `user?.name` |
| Comment | `# comment` | `// comment` or `/* comment */` |
| Load other files | `require` / Rails autoload | `import ... from '...'` |

---

## 2. Variables: `const` and `let`

| Keyword | Meaning | When used in freee-payroll |
|---|---|---|
| `const` | A variable that **can't be reassigned** | Almost always (the default) |
| `let` | A variable that **can be reassigned** | Only when the value must change later |
| `var` | Old-style variable (confusing scoping rules) | Basically never; you'll see it only in `declare global { var $FREEE_DATA ... }` (Lesson 25) |

```js
const companyName = 'freee';
companyName = 'other';      // ❌ TypeError: Assignment to constant variable.

let count = 0;
count = count + 1;          // ✅ fine
```

> ⚠️ `const` means *the variable can't point to something else*. It does **not** freeze the object inside. `const user = { name: 'A' }; user.name = 'B';` works. (Compare Ruby's `freeze`, which really freezes.)

🔎 **In freee-payroll** `front/javascripts/company_social_insurance_settings/hooks/useKyokaiKenpoRates.ts:53`:

```ts
    let isCancelled = false;
```

This is `let` because a few lines later (line 85) the code does `isCancelled = true;`. Everything else in that file is `const`.

**Naming:** variables and functions use `camelCase` (`employeeCount`, `isLoading`); true constants use `UPPER_SNAKE_CASE`, e.g. `front/javascripts/hr_portal/hooks/useAttritionRiskSummary.ts:10`:

```ts
const CACHE_CAPACITY = 32;
```

Booleans are usually named `isXxx` / `hasXxx` / `canXxx` (`isLoading`, `hasError`, `canSubmit`), like Ruby's `?` methods.

---

## 3. Values (primitive types)

| Type | Examples | Notes |
|---|---|---|
| **number** | `42`, `3.14`, `-1`, `100_000` | One number type for integers *and* decimals (Ruby has Integer and Float). |
| **string** | `'text'`, `"text"`, `` `text` `` | The repo uses **single quotes** (Prettier config), and backticks when interpolating. |
| **boolean** | `true`, `false` | |
| **null** | `null` | "Deliberately empty / no value." |
| **undefined** | `undefined` | "Not set at all." A missing object property or a function with no `return` gives `undefined`. |

Check a value's type with `typeof`:

```bash
node -e "console.log(typeof 1, typeof 'a', typeof true, typeof undefined, typeof null)"
# → number string boolean undefined object
```

(`typeof null` is `'object'`. This is a famous historical bug in JS; just remember it.)

### `null` vs `undefined`

```js
const user = { name: 'Taro', department: null };
user.department;   // null       ← the key exists, value is "nothing on purpose"
user.email;        // undefined  ← the key doesn't exist at all
```

In Ruby both would be `nil`. In JS you need to think about both. The repo's API types often say `string | null` ("a string, or null"). You'll see this again in Lesson 24.

### Template literals (string interpolation)

Backtick strings can contain `${expression}`:

🔎 `front/javascripts/company_social_insurance_settings/components/common/RateInputRow.tsx:53`

```ts
  const employeeFieldName = `${rateType}_employee`;
```

If `rateType` is `'base_rate'`, the result is `'base_rate_employee'`.

🔎 `front/packages/@freee-payroll/api/src/TaxOffice.ts:10`, a URL built from a variable:

```ts
  const { data } = await axios.get<IndexResponse>(`/api/p/tax_offices/${pref}`);
```

---

## 4. Functions

A **function** is a named, reusable piece of code (like a Ruby method). JS has several ways to write one:

```js
// (a) function declaration
function add(a, b) {
  return a + b;
}

// (b) arrow function: the style used almost everywhere in freee-payroll
const add = (a, b) => {
  return a + b;
};

// (c) arrow function with implicit return: no { } and no `return`
const add = (a, b) => a + b;
```

| Term | Meaning |
|---|---|
| **Parameter** | The name in the definition: `a`, `b`. |
| **Argument** | The actual value passed when calling: `add(1, 2)`: `1`, `2`. |
| **Return value** | What the function gives back. |
| **Arrow function** | The `(...) => ...` syntax. Reads as "given these inputs, produce this." |
| **Implicit return** | An arrow function *without* `{ }` returns its expression automatically. |
| **Default parameter** | `(x = 10) => ...`: uses 10 if the caller passes nothing. |
| **Callback** | A function passed to another function to be called later, e.g. `list.map((x) => x * 2)`. Same role as a Ruby block. |

> ⚠️ **Biggest Ruby → JS trap:** with `{ }` you MUST write `return`. Ruby returns the last line automatically; JS does not.
>
> ```js
> const double = (x) => { x * 2 };   // returns undefined!
> const double = (x) => x * 2;       // returns x * 2 ✅
> ```

> ⚠️ **Returning an object from a one-line arrow:** wrap it in `( )`, otherwise `{` is read as a function body:
> ```js
> const makeUser = (name) => ({ name: name });
> ```

🔎 **In freee-payroll** `front/javascripts/company_social_insurance_settings/components/InsuranceSettingsIndex.tsx:48-51`, an arrow function with implicit return and a ternary (`? :`, Lesson 23):

```ts
const toActiveTab = (tab?: string): InsuranceTab =>
  tab === 'welfare_pension_insurance'
    ? 'welfare_pension_insurance'
    : 'health_insurance';
```

Ignore `?: string` and `: InsuranceTab` for now: those are TypeScript type labels. Strip them and it's plain JS: `const toActiveTab = (tab) => tab === '...' ? '...' : '...'`.

🔎 A default parameter, `front/javascripts/migration_projects/hooks/useFetch.ts:21`:

```ts
  enable: boolean = true
```

"If the caller doesn't pass `enable`, it's `true`." (Same as Ruby `def x(enable: true)`.)

### Calling functions and methods

```js
add(1, 2);                // call a function
'freee'.toUpperCase();    // call a method on a value → 'FREEE'
[1, 2, 3].includes(2);    // → true
```

Parentheses are **required** in JS, even with no arguments: `list.length` is a property (no parens), `name.trim()` is a method call. Ruby lets you drop parentheses; JS doesn't.

---

## 5. Objects and arrays

### Objects (like Ruby hashes)

```js
const employee = {
  id: 1,
  displayName: '給与太郎',
  'X-Company-Id': '10',          // keys with special characters need quotes
  department: { name: '開発部' }, // nested object
};

employee.id;                 // 1           (dot access)
employee['X-Company-Id'];    // '10'        (bracket access, needed for special keys)
employee.department.name;    // '開発部'
employee.email;              // undefined   (no such key: no error!)
employee.age = 30;           // add/change a property
```

| Term | Meaning |
|---|---|
| **Object** | A collection of *key: value* pairs. (JSON is based on this syntax.) |
| **Property** | One key/value pair; `employee.id` "reads the id property". |
| **Key** | The name. Unlike Ruby, you write `{ id: 1 }` and read `obj.id`; there are no symbols. |
| **Shorthand property** | `{ headers }` is the same as `{ headers: headers }` when a variable has the same name as the key. |

🔎 Shorthand property, `front/packages/configured-axios/index.ts:15-19`:

```ts
export const axiosOptions = {
  headers,
  xsrfCookieName: 'XSRF-TOKEN',
  xsrfHeaderName: 'X-CSRF-TOKEN',
};
```

`headers,` means `headers: headers,` (a variable named `headers` defined earlier in the same file).

### Arrays

```js
const names = ['A', 'B', 'C'];
names[0];          // 'A'   (index starts at 0, like Ruby)
names[5];          // undefined (Ruby: nil)
names.length;      // 3     (property, no parentheses)
names.push('D');   // add to the end (Ruby: <<)
```

🔎 `front/javascripts/company_social_insurance_settings/components/InsuranceSettingsIndex.tsx:43-46`, an **array of objects**, a very common shape:

```ts
const TABS: Array<{ key: InsuranceTab; label: string }> = [
  { key: 'health_insurance', label: '健康保険' },
  { key: 'welfare_pension_insurance', label: '厚生年金保険' },
];
```

Without the type label: `const TABS = [ {...}, {...} ];`, two tabs, each with a `key` and a `label`.

---

## 6. Destructuring: unpacking objects and arrays

**Destructuring** = pulling values out of an object/array into variables in one line. It's *everywhere* in freee-payroll, so learn to read it fluently.

### Object destructuring

```js
const options = { month: 9, year: 2026 };

// long way
const month = options.month;
const year = options.year;

// destructuring: same result
const { month, year } = options;
```

🔎 `front/javascripts/company_social_insurance_settings/hooks/useKyokaiKenpoRates.ts:46`

```ts
  const { month, year } = options;
```

🔎 `front/packages/@freee-payroll/api/src/TaxOffice.ts:10`: the API call returns a *response object*; we only want its `data` property:

```ts
  const { data } = await axios.get<IndexResponse>(`/api/p/tax_offices/${pref}`);
```

### Renaming and defaults

```js
const { row: currentRow } = obj;       // take obj.row, call it currentRow
const { fieldErrors = [] } = props;    // if props.fieldErrors is undefined, use []
```

🔎 Renaming, `front/javascripts/hr_portal/hooks/useAttritionRiskSummary.ts:137-138`:

```ts
      const { row: currentRow, surveyDeliveredAt: currentDeliveredAt } =
        requestInputsRef.current;
```

🔎 Default value, `front/javascripts/company_social_insurance_settings/components/common/RateInputRow.tsx:45` (inside a larger `const { ... } = props;`):

```ts
    fieldErrors = [],
```

### Array destructuring

```js
const [first, second] = ['A', 'B'];   // first = 'A', second = 'B'
```

🔎 You'll see this in *every* React component (Lesson 27), `useKyokaiKenpoRates.ts:48`:

```ts
  const [rates, setRates] = React.useState<KyokaiKenpoRates>({});
```

`useState` returns a two-item array `[currentValue, functionToChangeIt]`, and destructuring names them.

### Destructuring function parameters

```js
const greet = ({ name, age }) => `${name} (${age})`;
greet({ name: 'Taro', age: 30 });   // 'Taro (30)'
```

This is how React components read their inputs ("props"). Same idea as Ruby keyword arguments `def greet(name:, age:)`.

---

## 7. Spread `...`

The **spread** operator `...` "pours out" the contents of an object or array into a new one.

```js
const base = { a: 1, b: 2 };
const copy = { ...base };             // { a: 1, b: 2 }  (a new object)
const changed = { ...base, b: 99 };   // { a: 1, b: 99 } (later keys win)

const xs = [1, 2];
const ys = [...xs, 3];                // [1, 2, 3]
```

Why does the repo love this? React requires **not mutating** (changing in place) existing data; you create a *new* object with the change. Spread makes that one line.

🔎 `front/javascripts/shift/components/ShiftWorkPattern/useShiftWorkPatternEdit.reducer.ts:51`

```ts
      return { ...state, fieldErrors: [] };
```

"Return a copy of `state` where `fieldErrors` is replaced by an empty array."

🔎 A harder real example, `front/packages/configured-axios/index.ts:3-14`:

```ts
const headers = {
  ...($FREEE_DATA.loginUser?.company_id != null
    ? {
        'X-Company-Id': String($FREEE_DATA.loginUser.company_id),
      }
    : {}),
  ...($FREEE_DATA.loginUser?.user_id != null
    ? {
        'X-User-Id': String($FREEE_DATA.loginUser.user_id),
      }
    : {}),
};
```

How to read it:
- "If the logged-in user's `company_id` exists, spread `{ 'X-Company-Id': '<id>' }` in; otherwise spread `{}` (nothing)."
- Same for `user_id`.
- Result: an object with 0, 1, or 2 header keys.
- `String(...)` converts a number to a string. `?.` and `!= null` are explained in Lesson 23.

These headers get attached to every API request so the Rails server knows which company/user is calling.

---

## 8. Modules: `import` and `export`

A **module** is one file. Files share code by **exporting** things and **importing** them elsewhere. (Rails autoloads classes by name; JS makes you import explicitly at the top of every file.)

### Named exports vs default export

```js
// math.ts
export const add = (a, b) => a + b;        // named export
export const sub = (a, b) => a - b;        // named export
const multiply = (a, b) => a * b;
export default multiply;                   // default export (max one per file)

// other.ts
import multiply, { add, sub } from './math';   // default first, then { named }
```

| Term | Meaning |
|---|---|
| **Named export** | Exported with its name; imported with `{ name }` and the name must match. |
| **Default export** | "The main thing" of a file; importer can call it anything. |
| **Namespace import** | `import * as API from '...'`: everything the file exports, grouped under one name. |
| **Type-only import** | `import type { X } from '...'`: imports only a TypeScript type (erased at build). |
| **Relative path** | `'./Routes'`, `'../types'`: a file near this one. `.` = this folder, `..` = parent folder. No file extension needed. |
| **Package path** | `'react'`, `'@c-fo/vibes'`, `'@freee-payroll/api'`: an installed package (or internal workspace package, Lesson 29). |

🔎 A typical component file header, `front/javascripts/home/components/Home/JoiningEmployeesPanel.tsx:1-7`:

```tsx
import { ColumnBase, MarginBase } from '@c-fo/vibes';
import * as API from '@freee-payroll/api';
import { usePlanFeatureUsable } from '@freee-payroll/hooks';
import { hasPermission } from '@freee-payroll/utils';
import * as React from 'react';

import NotificationPanel from './NotificationPanel';
```

- Lines 1, 3, 4: **named imports** from packages.
- Lines 2, 5: **namespace imports** (`API.apiPJoiningProceduresEmployeesIncompleteGet()` is used later).
- Line 7: a **default import** from a sibling file.
- Order: packages first (alphabetical), blank line, then local files. That's enforced by the ESLint rule `'import/order'` in `.eslintrc.js:353` (with `alphabetize` and `'newlines-between': 'always'`).

🔎 The same file exports two things (`JoiningEmployeesPanel.tsx:26` and `:28`):

```tsx
export default JoiningEmployeesPanel;

export const JoiningEmployeesPanelContainer = () => {
```

This "default = display component, named `...Container` = data-loading component" pattern is very common in the repo (Lesson 29).

🔎 `import type`, `front/javascripts/hr_portal/types.ts:1-5`:

```ts
import type {
  EmployeeAttendanceRiskResponseAttendanceRisk,
  SurveyResultItem,
  SurveyRisk,
} from '@freee-payroll/api';
```

### The older style: CommonJS (`module.exports` / `require`)

Before `import`/`export` existed, Node used **CommonJS**. You'll still see it in config files. 🔎 `front/jsEntries.js:5-8`:

```js
module.exports = {
  'renew-settings': 'front/javascripts/DEPRECATED_settings/index.ts',
  admin2: 'front/javascripts/admin2/index.tsx',
  admin: 'front/javascripts/admin/index.ts',
```

`module.exports = X` is the old version of `export default X`, and `require('x')` is the old version of `import`. (The legacy CoffeeScript code uses `require` too.) Read it; don't write it in new `front/javascripts` code.

---

## 9. Practice (15 min)

Run each with `node -e "..."` (use double quotes outside, single quotes inside), or paste into a file `practice.js` and run `node practice.js`.

**P1. Variables and template literals.** Predict, then run:
```bash
node -e "const name = 'Taro'; let n = 1; n = n + 1; console.log(\`\${name} has \${n} tasks\`)"
```

**P2. The `return` trap.** Predict the two outputs:
```bash
node -e "const a = (x) => { x * 2 }; const b = (x) => x * 2; console.log(a(5), b(5))"
```

**P3. Destructuring.** Predict:
```bash
node -e "const res = { data: { models: [1, 2] }, status: 200 }; const { data } = res; const { models } = data; console.log(models.length)"
```

**P4. Spread.** Predict:
```bash
node -e "const state = { fieldErrors: ['x'], isOpen: true }; const next = { ...state, fieldErrors: [] }; console.log(state.fieldErrors.length, next.fieldErrors.length, next.isOpen)"
```

**P5. Read real code.** Open `front/javascripts/home/components/Home/EmployeeMessagePanel.tsx` and find: (a) a default or named export, (b) an object destructuring, (c) a template literal.

*(Answers: P1 `Taro has 2 tasks`. P2 `undefined 10`. P3 `2`. P4 `1 0 true`: the original `state` was not changed. P5 (a) `export const EmployeeMessagePanel` line 12 and `export const EmployeeMessagePanelContainer` line 36; (b) `const { messageCollections } = props;` line 13; (c) `` title={`入社予定者が${messageCollections.length}人います。…`} `` line 27.)*

---

## 10. Summary

- Use **`const`** by default, **`let`** only if you reassign. Names are `camelCase`.
- Values: number, string, boolean, **`null`** (empty on purpose), **`undefined`** (not set).
- Backticks `` `...${x}...` `` interpolate.
- **Arrow functions** `(a) => a * 2` are the norm. With `{ }` you must write **`return`**.
- Objects `{ key: value }`, arrays `[...]`. Missing keys give `undefined`, not an error.
- **Destructuring** `const { data } = res;`, `const [value, setValue] = ...;` unpacks.
- **Spread** `{ ...obj, key: newValue }` makes a *changed copy* (no mutation).
- Files share code with **`import` / `export`** (named vs default, `import * as API`, `import type`). `module.exports`/`require` is the older CommonJS style.

➡️ Now take **Test22.md**.
