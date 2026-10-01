# Lesson 27 — React 2: State and Effects

**Time:** ~1 hour (40 min reading + practice, 20 min test)
**Goal:** Understand how a component *remembers* things (state), how it *reacts to changes* (re-render), and how it *talks to the outside world* (effects), well enough to read any `useState` / `useEffect` in `front/`.
**You need:** Lesson 26. A React + TypeScript sandbox (StackBlitz "React TS" or the editable examples on <https://react.dev/learn>).

---

## 1. Why components need state

Props come from the parent. But some data belongs to the component itself: "is this modal open?", "what has the user typed?", "the list we just loaded from the API". This is **state**.

| Term | Meaning in plain words | Example |
|---|---|---|
| **State** | Data a component remembers between renders, and which **causes a re-render when it changes**. | `displayName` typed into a field |
| **Hook** | A special React function whose name starts with `use`. Hooks let components use React features like state. | `useState`, `useEffect` |
| **`useState`** | The hook that creates one piece of state. Returns `[currentValue, setterFunction]`. | `const [count, setCount] = React.useState(0);` |
| **Setter** | The function returned by `useState` (`setXxx`). Calling it stores a new value **and schedules a re-render**. | `setCount(5)` |
| **Initial value** | The value used on the **first** render only. | `0`, `''`, `[]`, `false` |

```tsx
// (illustrative example)
const Counter = () => {
  const [count, setCount] = React.useState(0);
  return <button onClick={() => setCount(count + 1)}>{count}回</button>;
};
```

What happens when you click:
1. `setCount(1)` stores `1` and tells React "Counter needs to re-render".
2. React calls `Counter()` **again**. This time `useState` returns `1`.
3. The JSX now says `1回`. React diffs and updates only that text.

### 1.1 Why not just a normal variable?

```tsx
// (illustrative example) ❌ does not work
const Counter = () => {
  let count = 0;
  return <button onClick={() => { count += 1; }}>{count}回</button>;
};
```

Two problems: changing `count` doesn't tell React to re-render, and even if it re-rendered, the function runs from the top and `count` would be `0` again. **State survives re-renders; local variables don't.**

### 1.2 State rules

| Rule | Why |
|---|---|
| Never change state directly (`list.push(x)`, `obj.a = 1`). Always call the setter with a **new** value. | React compares old vs new by reference; mutating the same object looks like "nothing changed". |
| For arrays/objects, build a copy: `setList([...list, x])`, `setObj({ ...obj, a: 1 })`. | Spread (`...`) makes a new array/object (Lesson 22). |
| When the new value depends on the old one, you can pass a function: `setCount((prev) => prev + 1)`. | Guarantees you use the latest value, even if several updates happen quickly. |
| Setting state is **not instant**. After `setCount(5)`, the `count` variable in the current render is still the old value. | The new value appears on the *next* render. |
| Give TypeScript a type when the initial value doesn't say enough: `React.useState<GroupImport[]>([])`, `React.useState<number | null>(null)`. | `[]` alone would be `never[]`. |

### 🔎 In freee-payroll

`front/javascripts/renew_settings2/components/GroupImportHistories.tsx:34-35`

```tsx
  const [isLoading, setIsLoading] = React.useState(false);
  const [groupImports, setGroupImports] = React.useState<GroupImport[]>([]);
```

`front/javascripts/company_social_insurance_settings/hooks/useHealthInsuranceSettings.ts:12-21`

```tsx
  const [settings, setSettings] = React.useState<
    ReadonlyArray<API.CompanyHealthInsuranceSetting>
  >([]);
  const [setting, setSetting] =
    React.useState<API.CompanyHealthInsuranceSetting | null>(null);
  const [expiredYear, setExpiredYear] = React.useState<number | null>(null);
  const [expiredMonth, setExpiredMonth] = React.useState<number | null>(null);
  const [fieldErrors, setFieldErrors] = React.useState<
    ReadonlyArray<HRErrorField>
  >([]);
```

Notice the naming convention everywhere: `[thing, setThing]`, booleans named `isXxx` / `hasXxx`. freee-payroll writes `React.useState(...)` (because of `import * as React from 'react'`); some newer files import it by name: `import { useState } from 'react'` then `useState(...)`. Same function.

---

## 2. Controlled inputs

| Term | Meaning |
|---|---|
| **Controlled input** | A text field whose displayed value **comes from state**, and whose `onChange` **updates that state**. React is the "single source of truth" for what's in the box. |
| **Uncontrolled input** | The browser keeps the value itself; you read it later. Rare in freee-payroll. |

### 🔎 In freee-payroll: `DisplayNameInputModal`

`front/javascripts/employees/components/Employee/DisplayNameInputModal.tsx:50` and `:77-85`

```tsx
  const [displayName, setDisplayName] = React.useState('');
  // ...
          <TextField
            autoComplete="off"
            data-test="表示名入力"
            id="member_replacement_display_name"
            maxLength={DISPLAY_NAME_MAX_LENGTH}
            onChange={(e) => setDisplayName(e.currentTarget.value)}
            value={displayName}
            width="medium"
          />
```

The loop:
1. User types "給" → `onChange` fires → `setDisplayName('給')`.
2. Re-render → `value={displayName}` shows "給".
3. Because the value lives in state, the component can use it anywhere: the same file computes `const trimmed = displayName.trim();` and disables the save button with `suspend={isLoading || trimmed.length === 0}` (`:58`, `:68`).

`e.currentTarget.value` = "the current text of the element this handler is attached to."

---

## 3. Lifting state up

When **two components need the same state**, move ("lift") the state to their closest common parent and pass it down as props, plus a setter callback.

```
            Parent  (owns: selectedId, setSelectedId)
           /                        \
  List (props: items, onSelect)   Detail (props: selectedId)
```

```tsx
// (illustrative example)
const Parent = () => {
  const [selectedId, setSelectedId] = React.useState<number | null>(null);
  return (
    <>
      <EmployeeList onSelect={setSelectedId} />
      <EmployeeDetail employeeId={selectedId} />
    </>
  );
};
```

This is exactly what the **Container** in the freee-payroll pattern does: `InsuranceSettingsIndexContainer` owns the data and passes `healthInsuranceSettings`, `isLoading`, `onChangeDefaultSettings` down to `InsuranceSettingsIndex` (`InsuranceSettingsIndex.tsx:203-273`).

freee-payroll's rule (`front/README.md:21`): **use local state by default.** Context and Redux (Lesson 28) are only for state that must be shared across many screens.

---

## 4. Effects: `useEffect`

Rendering must be **pure**: given props and state, just compute JSX. No API calls, no timers, no touching `window`. Anything that reaches *outside* the component is a **side effect**, and it goes in `useEffect`.

| Term | Meaning |
|---|---|
| **Side effect** | Work that affects or depends on the outside world: fetching data, timers, event listeners, changing the page title. |
| **`useEffect(fn, deps)`** | "After React has rendered and updated the screen, run `fn`. Run it again whenever something in `deps` changed." |
| **Dependency array (`deps`)** | The list of values the effect uses. React compares each one with its value from the previous render. |
| **Cleanup function** | A function you **return** from the effect. React calls it before re-running the effect and when the component is removed ("unmounted"). |
| **Mount / unmount** | Mount = the component appears on screen for the first time. Unmount = it's removed. |

### 4.1 The three forms of the dependency array

| Code | When the effect runs |
|---|---|
| `useEffect(fn)` (no array) | After **every** render. Rare. |
| `useEffect(fn, [])` | Once, after the first render (on mount). |
| `useEffect(fn, [a, b])` | After the first render, and again whenever `a` or `b` changed. |

freee-payroll turns on `react-hooks/exhaustive-deps` as an **error** (`.eslintrc.js:331`): ESLint makes you list **every** value from the component that the effect uses. That's why you see long dep arrays with functions like `notify` and `showLoading` in them.

### 4.2 Effect order, step by step

```tsx
// (illustrative example)
React.useEffect(() => {
  console.log('effect: year is', year);
  return () => console.log('cleanup for', year);
}, [year]);
```

- First render with `year=2025` → log `effect: year is 2025`.
- Re-render, `year` unchanged → nothing.
- Re-render with `year=2026` → log `cleanup for 2025`, then `effect: year is 2026`.
- Component removed → log `cleanup for 2026`.

### 🔎 In freee-payroll: resetting state when a prop changes

`DisplayNameInputModal.tsx:52-56`

```tsx
  React.useEffect(() => {
    if (!isOpen) {
      setDisplayName('');
    }
  }, [isOpen]);
```

"Whenever `isOpen` changes, if the modal just closed, empty the text field." So reopening it starts blank.

---

## 5. Loading data from the API

This is the most common effect in the repo. The shape is always:

```tsx
// (illustrative example — the freee-payroll shape)
const [items, setItems] = React.useState<Item[]>([]);
const [isLoading, setIsLoading] = React.useState(false);

React.useEffect(() => {
  (async () => {
    setIsLoading(true);
    try {
      const { data } = await API.apiPItemsGet();
      setItems(data.items);
    } catch (e) {
      notify({ message: API.getErrorMessage(e), type: 'error' });
    }
    setIsLoading(false);
  })();
}, [notify]);
```

Why the strange `(async () => { ... })();`? The function you give `useEffect` must return **nothing or a cleanup function**. An `async` function always returns a Promise, so you can't write `useEffect(async () => ...)`. Instead you define an async arrow function and call it immediately (this is called an **IIFE**, "Immediately Invoked Function Expression").

### 🔎 In freee-payroll: `GroupImportHistoriesContainer`

`front/javascripts/renew_settings2/components/GroupImportHistories.tsx:32-58`

```tsx
export const GroupImportHistoriesContainer = () => {
  useTitle('部門のインポート履歴');
  const [isLoading, setIsLoading] = React.useState(false);
  const [groupImports, setGroupImports] = React.useState<GroupImport[]>([]);

  const notify = useNotification();

  // インポート履歴を取得
  React.useEffect(() => {
    (async () => {
      setIsLoading(true);
      try {
        const groupImports = await API.OrganizationGroupImport.index();
        setGroupImports(groupImports);
      } catch (e) {
        notify({ message: API.getErrorMessage(e), type: 'error' });
      }
      setIsLoading(false);
    })();
  }, [notify]);

  return (
    <Loading coverAll isLoading={isLoading}>
      <GroupImportHistories groupImports={groupImports} />
    </Loading>
  );
};
```

Line by line:
- `useTitle(...)` — a shared custom hook from `@freee-payroll/hooks` that sets the browser tab title.
- `useNotification()` — returns `notify`, which shows a toast message (small pop-up). Used for errors.
- The effect: start loading → call API → save result in state → on error show a toast with `API.getErrorMessage(e)` (which extracts the `message` from the Rails `HrErrors` JSON response, Lesson 15) → stop loading.
- `[notify]` — the only outside value the effect uses. `notify` is stable, so the effect effectively runs once on mount.
- `<Loading coverAll isLoading={isLoading}>` — Vibes spinner overlay while loading.

The first render shows an empty list with a spinner; when data arrives, `setGroupImports` triggers a re-render with the real data. **Always think in two renders: "before data" and "after data".**

---

## 6. Cleanup and race conditions

**Race condition**: two async operations finish in an unexpected order. Example: the user switches from 2025 to 2026 quickly; the 2025 request finishes *after* the 2026 one and overwrites the screen with old data. Or the component is closed before the response arrives, and we call a setter on a component that no longer exists.

The fix: a cleanup that marks the old request as cancelled.

### 🔎 In freee-payroll: the `isCancelled` flag

`front/javascripts/company_social_insurance_settings/hooks/useKyokaiKenpoRates.ts:48-89`

```tsx
  const [rates, setRates] = React.useState<KyokaiKenpoRates>({});
  const { hideLoading, isLoading, showLoading } = useLoading();
  const notify = useNotification();

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
        // eslint-disable-next-line @typescript-eslint/no-unnecessary-condition -- cleanup で書き換わるがリンタは追えない
        if (!isCancelled) {
          setRates(
            Object.fromEntries(
              Object.entries(data.kyokai_kenpo_health_insurance_rates).map(
                ([prefectureCode, rate]) => [
                  prefectureCode,
                  toKyokaiKenpoRate(rate),
                ]
              )
            )
          );
        }
      } catch (e) {
        // eslint-disable-next-line @typescript-eslint/no-unnecessary-condition -- cleanup で書き換わるがリンタは追えない
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

  return { isLoading, rates };
```

How it protects us:
1. Each run of the effect gets **its own** `isCancelled` variable (a *closure*, Lesson 22).
2. If `year` or `month` changes, React runs the **cleanup** of the old effect → the old `isCancelled` becomes `true`.
3. When the old request finally returns, `if (!isCancelled)` is false → its result is **ignored**.
4. `finally { hideLoading(); }` runs whether it succeeded or failed.
5. The comment on `eslint-disable-next-line` says: "the cleanup changes this variable, but the linter can't follow that", so the lint warning is intentionally silenced.
6. The API function name `apiPCompanyHealthInsuranceSettingsKyokaiKenpoRatesYearMonthGet(year, month)` is generated from the Rails route `GET /api/p/company_health_insurance_settings/kyokai_kenpo_rates/{year}/{month}` (Lesson 15).

Other cleanups you'll see: `return () => window.removeEventListener(...)` (Lesson 28), `return () => clearInterval(timer)`.

---

## 7. Stale closures

A **stale closure** is when a function "remembers" an old value from the render in which it was created.

```tsx
// (illustrative example) ❌ bug
const [count, setCount] = React.useState(0);
React.useEffect(() => {
  const timer = setInterval(() => setCount(count + 1), 1000);
  return () => clearInterval(timer);
}, []); // effect only runs once, so it always sees count = 0 → count stays at 1
```

Fixes:
- Use the functional updater: `setCount((prev) => prev + 1)` — doesn't need `count` at all.
- Or list `count` in the deps (the effect re-runs, and the cleanup clears the old timer).

`exhaustive-deps` (enforced in freee-payroll) catches most stale-closure bugs by forcing you to list what you use. When a value must be read "fresh" without re-running the effect, freee-payroll uses `useRef` (Lesson 28).

---

## 8. Rules of Hooks

freee-payroll enforces these with `react-hooks/rules-of-hooks: 'error'` (`.eslintrc.js:332`).

| Rule | ❌ Wrong | ✅ Right |
|---|---|---|
| Only call hooks at the **top level** of a component or custom hook. Not inside `if`, loops, or nested functions. | `if (isOpen) { useState(...) }` | `const [x, setX] = useState(...); if (isOpen) {...}` |
| Only call hooks from **React function components** or **custom hooks** (functions named `useXxx`). | calling `useState` in a normal helper | put it in `useMyThing()` |
| No early `return` **before** a hook. | `if (!data) return null; const [a] = useState();` | call all hooks first, then `return` |

**Why?** React identifies each hook by its **call order**. "The 1st `useState` in this component is `isLoading`, the 2nd is `groupImports`." If a hook is skipped on some renders, the order shifts and React hands the wrong state to the wrong variable.

---

## 9. Practice (15 min)

In a React TS sandbox:

**Practice 1 — controlled input.** Build `NameForm`: a text input whose value is in state; below it show `文字数: N`. Add a "クリア" button that empties it.

**Practice 2 — effect with deps.** Add a `year` state (number) and two buttons "前年"/"翌年". Add an effect that sets `document.title = \`${year}年\`` whenever `year` changes.

**Practice 3 — fake loading.** Write a container that on mount waits 1 second (`await new Promise((r) => setTimeout(r, 1000))`), then sets `employees` to a fixed array, with `isLoading` true meanwhile. Render "読み込み中..." while loading.

**Practice 4 — cancel.** Make the fake load depend on `year` (from Practice 2), and add an `isCancelled` cleanup like `useKyokaiKenpoRates`. Click "翌年" rapidly; confirm only the last year's result is shown.

<details><summary>Sample solution for 3–4</summary>

```tsx
const fakeFetch = (year: number) =>
  new Promise<string[]>((resolve) =>
    setTimeout(() => resolve([`${year}年の給与太郎`]), 1000)
  );

export default function App() {
  const [year, setYear] = React.useState(2026);
  const [employees, setEmployees] = React.useState<string[]>([]);
  const [isLoading, setIsLoading] = React.useState(false);

  React.useEffect(() => {
    let isCancelled = false;
    setIsLoading(true);
    (async () => {
      const result = await fakeFetch(year);
      if (!isCancelled) {
        setEmployees(result);
        setIsLoading(false);
      }
    })();
    return () => {
      isCancelled = true;
    };
  }, [year]);

  return (
    <>
      <button onClick={() => setYear((y) => y + 1)}>翌年</button>
      {isLoading ? <p>読み込み中...</p> : <p>{employees.join(', ')}</p>}
    </>
  );
}
```
</details>

---

## 10. Code Reading Drill (5 min)

Open `front/javascripts/home/components/Home/EmployeeMessagePanel.tsx:36-63` and answer:
1. What state does the container hold?
2. When does its effect run, and when does it actually call the API?
3. What does the destructuring `const { data: { employee_message_collections } } = await ...` do?

(Answers: 1. `messageCollections`, an array of `API.EmployeeMessageCollection`. 2. On mount and whenever `isAvailable` or `setMessageCollections` changes (effectively once); it only calls `API.apiPEmployeeMessageCollectionsRequestingGet()` if `isAvailable` (the logged-in user has an `employee_id` in `$FREEE_DATA`). 3. Takes `response.data.employee_message_collections` out of the Axios response in one step — nested destructuring.)

---

## 11. Summary

- **State** (`const [x, setX] = React.useState(init)`) is memory that survives re-renders; calling `setX` schedules a **re-render**. Never mutate — always pass a new value.
- **Controlled inputs**: `value={state}` + `onChange={(e) => setState(e.currentTarget.value)}`.
- **Lift state up** to the nearest common parent (the Container) when siblings share it. Local state first (freee rule).
- **`useEffect(fn, deps)`** runs side effects after render; **deps** decide when it re-runs; the returned **cleanup** runs before the next run and on unmount.
- The repo's data-loading shape: `useEffect(() => { (async () => { try { await API.apiP...(); setX(...) } catch (e) { notify({ message: API.getErrorMessage(e), type: 'error' }) } })(); }, [...])`, sometimes with an `isCancelled` cleanup.
- **Stale closures**: functions see values from the render they were made in; `exhaustive-deps` (error in this repo) and functional updaters help.
- **Rules of Hooks**: top level only, only in components/custom hooks, same order every render.

➡️ Now take **Test27.md**.
