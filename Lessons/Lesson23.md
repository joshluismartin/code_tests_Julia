# Lesson 23 — JavaScript Basics 2: Logic, Array Methods, Async Code

**Time:** ~1 hour (40 min reading + practice, 20 min test)
**Goal:** Read the logic you'll see in every frontend file: comparisons, truthy/falsy, `&&` `||` `??` `?.`, the ternary, `switch`, array methods (`map`, `filter`, `find`, `some`, `reduce`...), and **asynchronous code** (`Promise`, `async`/`await`, `try`/`catch`), which is how the frontend talks to the Rails API.
**You need:** Lesson 22. A terminal for `node -e`. The repo open at `~/Josh_Files/freee-payroll/front/`.

---

## 1. Comparing values: `===` (always use this)

| Operator | Meaning | Example |
|---|---|---|
| `===` | strictly equal (same type and value) | `1 === 1` → `true`, `1 === '1'` → `false` |
| `!==` | strictly not equal | `'a' !== 'b'` → `true` |
| `==` / `!=` | "loose" equal: converts types first (confusing) | `1 == '1'` → `true` 😱 |
| `<`, `>`, `<=`, `>=` | comparisons | `count > 0` |

**Rule:** always use `===` / `!==`. There is **one accepted exception** you'll see in freee-payroll:

```js
x != null   // true when x is neither null NOR undefined
```

`!= null` is a handy shortcut for "is this set to something?", because loose `!=` treats `null` and `undefined` as equal.

🔎 `front/packages/configured-axios/index.ts:4`

```ts
  ...($FREEE_DATA.loginUser?.company_id != null
```

"If `company_id` exists (not null, not undefined)..."

---

## 2. Truthy and falsy (different from Ruby!)

In an `if` or with `&&`/`||`, JS treats some values as "false-like" (**falsy**):

| Falsy in JS | Falsy in Ruby |
|---|---|
| `false`, `null`, `undefined`, **`0`**, **`''`** (empty string), **`NaN`** | only `false` and `nil` |

Everything else is **truthy** (including `[]` and `{}`).

> ⚠️ Ruby trap: in Ruby `0` and `''` are truthy. In JS they are **falsy**. `if (employeeCount)` is false when the count is `0`.

### `!!value`: convert anything to a real boolean

`!` means "not". `!!` = "not not" = "convert to `true`/`false`".

🔎 `front/javascripts/home/components/Home/EmployeeMessagePanel.tsx:37`

```ts
  const isAvailable: boolean = !!$FREEE_DATA.userInfo.user?.employee_id;
```

"`isAvailable` is `true` if the user has an employee id, else `false`."

🔎 `front/javascripts/renew_settings2/components/GroupImportHistories.tsx:20` uses `!!` before `.length` because `0` would otherwise be *rendered as the text "0"* in React (a classic bug):

```tsx
        {!!groupImports.length &&
```

---

## 3. Logical operators: `&&`, `||`, `??`, `?.`, `!`

| Operator | Name | Meaning | Ruby equivalent |
|---|---|---|---|
| `a && b` | AND | if `a` is falsy → `a`; else → `b` | `a && b` |
| `a \|\| b` | OR | if `a` is truthy → `a`; else → `b` | `a \|\| b` |
| `a ?? b` | **nullish coalescing** | if `a` is `null`/`undefined` → `b`; else → `a` | `a.nil? ? b : a` (roughly `a \|\| b`) |
| `a?.b` | **optional chaining** | if `a` is `null`/`undefined` → `undefined`; else `a.b` | `a&.b` |
| `!a` | NOT | opposite boolean | `!a` |

### `||` vs `??`: an important difference

```js
0 || 10       // 10  ← 0 is falsy, so || skips it
0 ?? 10       // 0   ← 0 is not null/undefined, so ?? keeps it
'' || 'N/A'   // 'N/A'
'' ?? 'N/A'   // ''
null ?? 'N/A' // 'N/A'
```

Use `??` for "default only when missing"; `0` and `''` are real values you want to keep. freee-payroll mostly uses `??` for defaults.

### `?.` stops errors on missing things

```js
const user = null;
user.name;        // ❌ TypeError: Cannot read properties of null
user?.name;       // undefined ✅
user?.name ?? '名無し'  // '名無し'
```

`?.` also works on calls and indexes: `obj.method?.()`, `list?.[0]`.

🔎 Both together, `front/packages/@freee-payroll/api/src/errorHandler.ts:53-63`:

```ts
export const getErrorMessage = (e: any) => {
  const defaultMessage =
    'エラーが発生しました。 再読み込みをしても解消しない場合は、サポートセンターまでご連絡ください。';
  if (isHRError(e)) {
    return e.response?.data.message ?? defaultMessage;
  }
  if (isLegacyHRError(e)) {
    return e.response?.data.messages[0] ?? defaultMessage;
  }
  return defaultMessage;
};
```

Read it as: "If the error is our standard format, return its `message` (if the response exists), else the default message. If it's the legacy format, return the first of `messages`..." Connect this to Lesson 15: the Rails `ErrorHandleable` returns `{ message, code, fields }`, the "HRError" shape; old controllers return `{ messages: [...] }`, the "legacy" shape. **This function is the frontend half of that contract.**

### `&&` as "if" (you'll see this in React)

```js
isAvailable && doSomething();   // only calls doSomething if isAvailable is truthy
```

In React JSX: `{isLoading && <Spinner />}` means "show the spinner only while loading" (Lesson 26).

---

## 4. Making decisions: `if`, ternary, `switch`

```js
if (count > 10) {
  console.log('many');
} else if (count > 0) {
  console.log('some');
} else {
  console.log('none');
}
```

- Parentheses around the condition are **required**. Braces `{ }` mark the block (Ruby uses `end`).
- Early return is common: `if (!isAvailable) { return; }` (Ruby: `return unless available`).

### Ternary: `condition ? valueIfTrue : valueIfFalse`

Same as Ruby. Very common because JSX can't contain `if` statements.

🔎 `front/javascripts/company_social_insurance_settings/components/InsuranceSettingsIndex.tsx:48-51` (seen in Lesson 22):

```ts
const toActiveTab = (tab?: string): InsuranceTab =>
  tab === 'welfare_pension_insurance'
    ? 'welfare_pension_insurance'
    : 'health_insurance';
```

### `switch`: choose by value (Ruby's `case/when`)

🔎 `front/javascripts/shift/components/ShiftWorkPattern/useShiftWorkPatternEdit.reducer.ts:46-51`

```ts
export function reducer(state: State, action: Action): State {
  switch (action.type) {
    case ACTION_TYPES.VALIDATION_FAILED:
      return { ...state, fieldErrors: action.fieldErrors };
    case ACTION_TYPES.FIELD_ERRORS_CLEARED:
      return { ...state, fieldErrors: [] };
```

- `switch (x)` compares `x` with each `case` using `===`.
- Each case normally ends with `return` or `break`; otherwise execution "falls through" into the next case. (`tsconfig.json` has `noFallthroughCasesInSwitch: true` to catch that mistake.)
- `default:` is the "else".

---

## 5. Array methods (JS's version of Ruby's Enumerable)

These take a **callback** (a function), just as Ruby methods take a block.

| JS | Ruby | What it returns |
|---|---|---|
| `xs.map((x) => x * 2)` | `xs.map { \|x\| x * 2 }` | new array, same length |
| `xs.filter((x) => x > 1)` | `xs.select { ... }` | new array with items that pass |
| `xs.find((x) => x.id === 3)` | `xs.find { ... }` | first matching item or **`undefined`** |
| `xs.findIndex((x) => ...)` | `xs.index { ... }` | index, or **`-1`** if none |
| `xs.some((x) => ...)` | `xs.any? { ... }` | `true` / `false` |
| `xs.every((x) => ...)` | `xs.all? { ... }` | `true` / `false` |
| `xs.includes(3)` | `xs.include?(3)` | `true` / `false` |
| `xs.reduce((acc, x) => acc + x, 0)` | `xs.reduce(0) { \|acc, x\| acc + x }` | one accumulated value |
| `xs.forEach((x) => ...)` | `xs.each { ... }` | nothing (`undefined`) |
| `xs.join(', ')` | `xs.join(', ')` | string |
| `xs.slice(0, 3)` | `xs.first(3)` | new array (first 3) |
| `Object.keys(obj)` / `Object.values(obj)` | `h.keys` / `h.values` | arrays |
| `Object.entries(obj)` | `h.to_a` | array of `[key, value]` pairs |
| `Object.fromEntries(pairs)` | `pairs.to_h` | object |

The callback also receives the index: `xs.map((x, i) => ...)`.

🔎 **Real examples** (open each and find the line):

`front/javascripts/company_social_insurance_settings/components/ChangeDefaultSettingModal.tsx:27`: `filter`:
```ts
) => settings.filter((s) => !s.is_disabled);
```
"Keep only settings that are not disabled."

`front/javascripts/company_social_insurance_settings/components/common/RateInputRow.tsx:56`: `some`:
```ts
    fieldErrors.some((fieldError) => fieldError.name === fieldName);
```
"Is there any error for this field?"

`front/javascripts/company_social_insurance_settings/components/InsuranceSettingsIndex.tsx:65`: `find` + `?.`:
```ts
): number | undefined => items.find((s) => s.is_default)?.id;
```
"Find the default setting; if found, take its `id`; if not found, `undefined`."

`vite.config.mts:31` (repo root): `reduce` + spread + a computed key `[entry]`:
```ts
          .reduce((acc, entry) => ({ ...acc, [entry]: entry }), {}),
```
"Start with `{}`; for each file path `entry`, add a key equal to the path." `['a','b']` → `{ a: 'a', b: 'b' }`. (`[entry]:` in an object literal means "use the *value* of `entry` as the key".)

`front/javascripts/company_social_insurance_settings/hooks/useKyokaiKenpoRates.ts:65-72`: `Object.entries` → `map` → `Object.fromEntries`, the JS way to "transform every value of an object":
```ts
            Object.fromEntries(
              Object.entries(data.kyokai_kenpo_health_insurance_rates).map(
                ([prefectureCode, rate]) => [
                  prefectureCode,
                  toKyokaiKenpoRate(rate),
                ]
              )
            )
```
Ruby equivalent: `rates.transform_values { |rate| to_kyokai_kenpo_rate(rate) }`. Note `([prefectureCode, rate]) =>`: **array destructuring in the parameter** unpacks each `[key, value]` pair.

---

## 6. Asynchronous code: waiting for the server

### 6.1 Why "async"?

Asking the Rails server for data takes time (maybe 200 ms). JS in the browser runs on **one thread**: if it simply *waited*, the whole page would freeze. Instead JS says "start the request, and tell me when it's done."

| Term | Meaning |
|---|---|
| **Synchronous** | Each line finishes before the next starts. |
| **Asynchronous (async)** | Start something now, get the result later, and keep doing other work meanwhile. |
| **Promise** | An object representing "a value that will arrive later". It is *pending*, then either **fulfilled** (success, with a value) or **rejected** (failure, with an error). |
| **`await`** | "Pause *this function* until the Promise finishes, then give me its value." Only allowed inside an `async` function. |
| **`async` function** | A function marked `async`; it can use `await`, and it **always returns a Promise**. |
| **`.then(fn)` / `.catch(fn)`** | Older way: "when fulfilled, run fn" / "when rejected, run fn". |
| **Exception / error** | Something went wrong; it is **thrown** (Ruby: `raise`). |
| **`try` / `catch` / `finally`** | Ruby's `begin` / `rescue` / `ensure`. |

### 6.2 Same thing, two styles

```js
// .then style
fetchData()
  .then((result) => console.log(result))
  .catch((error) => console.error(error));

// async/await style (used in freee-payroll)
const load = async () => {
  try {
    const result = await fetchData();
    console.log(result);
  } catch (error) {
    console.error(error);
  } finally {
    console.log('done either way');
  }
};
```

| Ruby | JS |
|---|---|
| `begin` | `try {` |
| `rescue => e` | `} catch (e) {` |
| `ensure` | `} finally {` |
| `raise 'x'` | `throw new Error('x');` |

Try it:

```bash
node -e "const wait = (ms) => new Promise((resolve) => setTimeout(() => resolve('ok'), ms)); (async () => { console.log('start'); const v = await wait(300); console.log(v); })(); console.log('meanwhile')"
# start
# meanwhile
# ok
```

`meanwhile` prints *before* `ok`: the program kept going while waiting. That's asynchrony.

### 6.3 The async IIFE pattern

**IIFE** = "Immediately Invoked Function Expression": define a function and call it right away: `(async () => { ... })();`. You'll see this inside React's `useEffect`, because the effect function itself can't be `async` (Lesson 27).

🔎 `front/javascripts/home/components/Home/JoiningEmployeesPanel.tsx:36-49`

```tsx
  React.useEffect(() => {
    (async () => {
      if (!isAvailable) {
        return;
      }
      try {
        const employeesResponse =
          await API.apiPJoiningProceduresEmployeesIncompleteGet();
        setEmployeeCount(employeesResponse.data.length);
      } catch (e) {
        // noop
      }
    })();
  }, [isAvailable]);
```

Line by line:
1. `(async () => { ... })();`: make an async function and call it immediately.
2. `if (!isAvailable) { return; }`: early exit.
3. `await API.apiP...Get()`: call the Private API `GET /api/p/joining_procedures/employees/incomplete` (generated function; the name comes from the URL) and wait.
4. `employeesResponse.data.length`: the JSON body is an array; count it.
5. `catch (e) { // noop }`: "no operation": if it fails, silently show nothing.

🔎 A fuller version with `finally`, `front/javascripts/company_social_insurance_settings/hooks/useKyokaiKenpoRates.ts:52-87` (shortened):

```ts
  React.useEffect(() => {
    let isCancelled = false;
    showLoading();
    (async () => {
      try {
        const { data } =
          await API.apiPCompanyHealthInsuranceSettingsKyokaiKenpoRatesYearMonthGet(
            year,
            month
          );
        if (!isCancelled) {
          setRates(/* ...transform data... */);
        }
      } catch (e) {
        if (!isCancelled) {
          notify({ message: API.getErrorMessage(e), type: 'error' });
        }
      } finally {
        hideLoading();
      }
    })();
    return () => {
      isCancelled = true;
    };
  }, [hideLoading, month, notify, showLoading, year]);
```

The standard freee-payroll recipe for loading data:
- `showLoading()` before, `hideLoading()` in `finally` (always runs, success or failure).
- On error: `notify({ message: API.getErrorMessage(e), type: 'error' })`, which shows the server's error message in a toast.
- `isCancelled` protects against "the screen was closed before the response came back" (Lesson 27 explains the `return () => {...}` cleanup).

### 6.4 `catch` with an unknown error

In TypeScript, a caught error has type `unknown` (it could be anything). Check it before using `.message`:

🔎 `front/javascripts/hr_portal/hooks/useAttritionRiskSummary.ts:192-195`

```ts
    fetchSummary().catch((err: unknown) => {
      const message = err instanceof Error ? err.message : '不明なエラー';
      finalize({ kind: 'error', message });
    });
```

`err instanceof Error` means "is `err` an Error object?" (Ruby: `err.is_a?(StandardError)`). Note this uses the `.catch` style on a Promise returned by an async function. Both styles appear in the repo.

---

## 7. The browser: DOM, console, debugger

| Term | Meaning |
|---|---|
| **DOM** (Document Object Model) | The browser's live tree of the page's HTML elements, which JS can read and change. |
| **`document.querySelector('#app')`** | "Find the first element matching this CSS selector" (`#app` = element with `id="app"`). |
| **DevTools** | The browser's developer panel (F12 / ⌥⌘I): Console, Network (see every `/api/p/...` request!), Elements, Sources. |
| **`console.log(x)`** | Print for debugging (remove before committing; ESLint may complain). |
| **`debugger;`** | A statement that pauses execution in DevTools when it's open (like Ruby's `binding.b`). |

🔎 `front/javascripts/admin2/index.tsx:8`

```tsx
createRoot(document.querySelector('#app')!).render(
```

"Find `<div id="app">` (rendered by Rails' `app/views/web/app.html.erb`, Lesson 16) and draw the React app inside it." (The `!` is TypeScript for "trust me, this isn't null", Lesson 25.)

> 💡 **Tip:** open freee人事労務 locally, DevTools → **Network** tab → filter "api/p". Click around and watch the requests and JSON responses. It's the best way to connect the frontend code with the Rails controllers.

---

## 8. Practice (15 min)

Predict each output, **then** run it.

**P1.** Truthy/falsy:
```bash
node -e "console.log(!!0, !!'', !![], !!'0', 0 || 5, 0 ?? 5, null ?? 5)"
```

**P2.** Optional chaining:
```bash
node -e "const u = { profile: null }; console.log(u.profile?.name, u.profile?.name ?? 'none')"
```

**P3.** Array methods:
```bash
node -e "const s = [{id:1,is_default:false,is_disabled:true},{id:2,is_default:true,is_disabled:false}]; console.log(s.filter((x) => !x.is_disabled).map((x) => x.id), s.find((x) => x.is_default)?.id, s.some((x) => x.id === 3))"
```

**P4.** reduce into an object:
```bash
node -e "console.log(['a.css','b.css'].reduce((acc, e) => ({ ...acc, [e]: e }), {}))"
```

**P5.** Async order:
```bash
node -e "(async () => { console.log(1); await null; console.log(3); })(); console.log(2)"
```

**P6.** try/catch/finally:
```bash
node -e "const f = async () => { try { throw new Error('boom'); } catch (e) { console.log('caught', e.message); } finally { console.log('finally'); } }; f()"
```

*(Answers: P1 `false false true true 5 0 5`. P2 `undefined none`. P3 `[ 2 ] 2 false`. P4 `{ 'a.css': 'a.css', 'b.css': 'b.css' }`. P5 `1 2 3` on separate lines: code after `await` runs later. P6 `caught boom` then `finally`.)*

---

## 9. Summary

- Use `===` / `!==`. Exception: `x != null` means "not null and not undefined".
- Falsy in JS: `false, null, undefined, 0, '', NaN`. `!!x` converts to a boolean.
- `||` replaces any falsy value; **`??`** replaces only `null`/`undefined`. **`?.`** = Ruby's `&.`.
- Ternary `c ? a : b` and `switch/case` (like Ruby `case/when`).
- Array methods: `map`, `filter` (= `select`), `find`, `some` (= `any?`), `every`, `reduce`, `includes`, `Object.entries`/`fromEntries`.
- **Async:** a Promise is a future value. `async` functions can `await`. Handle errors with `try/catch/finally`. The repo's loading recipe: `showLoading` → `await API.apiP...()` → `catch` → `notify({ message: API.getErrorMessage(e), type: 'error' })` → `finally hideLoading()`.
- `document.querySelector('#app')` finds the element Rails rendered; DevTools' Network tab shows every API call.

➡️ Now take **Test23.md**.
