# Lesson 26 — React 1: Components, JSX, Props

**Time:** ~1 hour (40 min reading + practice, 20 min test)
**Goal:** Be able to read any React component in `front/` and say what it shows on screen, what data it receives, and what happens when the user clicks or types.
**You need:** Lessons 22–25 (JavaScript + TypeScript). A browser for practice: open a sandbox at <https://react.dev/learn> (every page has an editable example) or <https://stackblitz.com> → "React TypeScript". Running freee-payroll's own frontend locally needs setup, so we practice in a sandbox and *read* the real repo.

---

## 1. What React is

| Term | Meaning in plain words | Example |
|---|---|---|
| **UI (User Interface)** | What the user sees and clicks: buttons, text, tables. | The freee人事労務 home screen |
| **DOM (Document Object Model)** | The browser's live tree of everything on the page. JavaScript can change it. | `<div>`, `<button>` elements |
| **React** | A JavaScript library that builds the UI from small reusable pieces called **components**. You describe *what the screen should look like*; React figures out *how to update the DOM*. | freee-payroll uses React 18 |
| **Declarative** | You describe the result ("show 3 employees"), not the steps ("create div, append text..."). React code is declarative. | `{count > 0 && <Panel />}` |
| **Component** | A function that returns what should appear on screen. Name starts with a **Capital letter**. | `JoiningEmployeesPanel` |
| **Render** | React calling your component function to find out what to show. | React "renders" `<LastName />` |
| **Re-render** | Calling the component function *again* because something changed (its props or state). | After data loads, the panel re-renders |
| **Virtual DOM** | React keeps a lightweight copy of the UI in memory. On each render it compares the new copy with the old one ("diffing") and only changes the parts of the real DOM that actually differ. This is why re-rendering is cheap. | Only the changed number is updated, not the whole page |

**Mental model:** a component is like a recipe. `UI = component(data)`. Same data in → same UI out. When the data changes, React runs the recipe again and patches the screen.

---

## 2. JSX: HTML-looking code inside TypeScript

**JSX** is a syntax that lets you write HTML-like tags inside JavaScript/TypeScript. Files that contain JSX end in **`.tsx`** (TypeScript + JSX).

```tsx
// (illustrative example)
const Greeting = () => {
  return <h1>こんにちは、給与太郎さん</h1>;
};
```

JSX is not real HTML: a build tool (Vite + TypeScript) turns it into function calls:

```tsx
<h1 className="title">Hello</h1>
// becomes roughly:
React.createElement('h1', { className: 'title' }, 'Hello');
```

That is why many freee-payroll files start with `import * as React from 'react';` even if they never write `React.` — the repo's `tsconfig.json` uses `"jsx": "react"` (the "classic" transform), which turns every tag into `React.createElement(...)`, so `React` must be in scope.

### 2.1 JSX rules you must know

| Rule | Example |
|---|---|
| Return **one** root element. Use a **Fragment** `<>...</>` to group without adding a real DOM element. | `return (<><A /><B /></>);` |
| Every tag must be closed. Tags with no children self-close. | `<TextField />`, `<br />` |
| `{ }` switches back to JavaScript inside JSX. Put any **expression** inside. | `{employeeCount}`, `` {`${n}名`} `` |
| Attributes use camelCase and `className` instead of `class`. | `onClick`, `className`, `htmlFor` |
| Comments inside JSX: `{/* ... */}` | `{/* モーダル */}` |
| Lowercase tag = real HTML element; Capitalized tag = your component. | `<div>` vs `<Button>` |

> **Expression vs statement:** An *expression* produces a value (`a + b`, `x ? 1 : 2`, `list.map(...)`). A *statement* does something (`if`, `for`). Only expressions go inside `{ }` in JSX. That's why JSX uses `? :` and `&&` instead of `if`.

---

## 3. Props: passing data into a component

| Term | Meaning |
|---|---|
| **Props** (properties) | The inputs of a component, passed like HTML attributes. Read-only inside the component. |
| **Props type** | A TypeScript `type` describing which props a component accepts. In freee-payroll this is almost always named `Props`. |
| **Destructuring props** | Pulling fields out of the props object: `const { a, b } = props;` |
| **Default prop value** | A value used when the parent doesn't pass one: `disabled = false`. |
| **children** | Special prop: whatever you put *between* the opening and closing tags. |
| **One-way data flow** | Data goes **down** from parent to child through props. A child never changes its parent's data directly; instead the parent passes a **function** (callback) the child can call. |

```tsx
// (illustrative example)
type Props = {
  name: string;
  age?: number;           // optional prop
};

const EmployeeCard = (props: Props) => {
  const { name, age } = props;
  return <p>{name}{age != null && `（${age}歳）`}</p>;
};

// Using it (the "parent"):
<EmployeeCard name="給与太郎" age={30} />
```

- String props can use quotes: `name="給与太郎"`. Anything else uses braces: `age={30}`, `isOpen={true}` (or just `isOpen`).
- A prop written with no value, like `required`, means `required={true}`. freee-payroll's ESLint rule `react/jsx-boolean-value` enforces this short form.

### 🔎 In freee-payroll: a tiny component with typed, destructured props

`front/javascripts/employees/components/form/DescriptionListCells/LastName.tsx:1-28`

```tsx
import { TextField } from '@c-fo/vibes';
import * as React from 'react';

const LastName = ({
  disabled = false,
  name,
  value,
  onChange,
}: {
  disabled?: boolean;
  name?: string;
  onChange: React.ChangeEventHandler<HTMLInputElement>;
  value: string;
}) => {
  return (
    <TextField
      data-test="姓"
      disabled={disabled}
      label="姓"
      mr={1}
      name={name}
      onChange={onChange}
      value={value}
    />
  );
};

export default LastName;
```

Read it line by line:
- `TextField` comes from **Vibes** (`@c-fo/vibes`), freee's in-house design-system library of ready-made UI parts (Lesson 29).
- The props are destructured **directly in the parameter list** (`({ disabled = false, name, ... })`) and typed inline (`: { disabled?: boolean; ... }`). `disabled = false` is a default value.
- `onChange: React.ChangeEventHandler<HTMLInputElement>` — "a function React calls when the text in an `<input>` changes." `LastName` doesn't decide what to do on change; the **parent** decides and passes the function down. That's one-way data flow.
- `data-test="姓"` is a marker used by freee's automated UI tests to find this element. You'll see `data-test` everywhere.
- Notice the props on `<TextField>` are in **alphabetical order** (`data-test`, `disabled`, `label`, `mr`, `name`, ...). This is enforced by the ESLint rule `react/jsx-sort-props` in `.eslintrc.js`. Don't be surprised when `onChange` appears after `name`.

---

## 4. Rendering lists with `map` and `key`

To show a list, turn an array of data into an array of JSX with `.map()` (Lesson 23):

```tsx
// (illustrative example)
const names = ['給与太郎', '労務花子'];

<ul>
  {names.map((name) => (
    <li key={name}>{name}</li>
  ))}
</ul>
```

| Term | Meaning |
|---|---|
| **key** | A special prop that gives each list item a **stable, unique identity** among its siblings. Usually an `id`. |

**Why keys matter:** when the list changes (item added, removed, reordered), React compares old and new lists. With keys it knows "item `id=5` is still here, just moved" instead of guessing by position. Without good keys, React may reuse the wrong DOM element — typed text can jump to another row, or checkboxes stay checked on the wrong item.

Rules:
- The key must be unique **among siblings** and stable across renders.
- Prefer a real id (`employee.id`). Avoid the array index (`(x, i) => <li key={i}>`) if the list can reorder or items can be removed.
- `key` is not passed to the component as a prop; React uses it internally.
- Forgetting `key` gives a warning in the browser console: *"Each child in a list should have a unique key prop."*

### 🔎 In freee-payroll: `.map` with `key`

`front/javascripts/DEPRECATED_settings/Routes.tsx:870-880` (inside a larger route list)

```tsx
                ].map((path) => (
                  <Route
                    component={() => <ErrorPage errorType="forbidden" />}
                    key={path}
                    path={path}
                  />
                ))}
```

An array of URL paths is turned into `<Route>` elements. Each path string is unique, so the path itself is a good key.

`front/javascripts/company_social_insurance_settings/components/InsuranceSettingsIndex.tsx:43-46` and `:174`

```tsx
const TABS: Array<{ key: InsuranceTab; label: string }> = [
  { key: 'health_insurance', label: '健康保険' },
  { key: 'welfare_pension_insurance', label: '厚生年金保険' },
];
// ...
        tabs={TABS.map((t) => t.label)}
```

Here `.map` produces an array of **strings** (`['健康保険', '厚生年金保険']`) passed as a prop — not JSX, so no `key` is needed. (The `key` field inside `TABS` is just ordinary data, not React's `key`.)

---

## 5. Conditional rendering: `&&` and `? :`

| Pattern | Meaning | Example |
|---|---|---|
| `cond && <X />` | Show `X` only if `cond` is truthy. | `{isAvailable && <Panel />}` |
| `cond ? <A /> : <B />` | Show `A` or `B`. | `{isInvited ? <Cancel /> : <Invite />}` |
| `return null` | Render nothing at all. | `if (!visible) return null;` |

⚠️ **The `0` trap:** `{count && <Panel />}` renders the text **`0`** when `count` is `0`, because `0 && x` evaluates to `0` and React prints numbers. Use a real boolean: `{count > 0 && <Panel />}` or `{!!list.length && ...}`. freee-payroll even turns on the ESLint rule `jsx-falsy/no-falsy-and` (`.eslintrc.js:265`) to catch this. (`null`, `undefined`, `false` and `true` render nothing.)

### 🔎 In freee-payroll: `&&` combined with a ternary

`front/javascripts/employees/components/Employee/MemberInfoCard.tsx:214-245`

```tsx
      {showActionButtons &&
        (isInvited ? (
          <HStack alignItems="center" gap={0.5}>
            <Button
              appearance="secondary"
              data-test="招待解除ボタン"
              disabled={isLoading}
              onClick={onCancelInvitationClick}
            >
              招待解除
            </Button>
            {account.invitationStatus === 'invited' && (
              <Button
                appearance="secondary"
                data-test="招待再送ボタン"
                disabled={isLoading}
                onClick={onResendInvitationClick}
              >
                招待再送
              </Button>
            )}
          </HStack>
        ) : (
          <Button
            appearance={inviteButtonAppearance}
            data-test="招待ボタン"
            disabled={isLoading}
            onClick={onInviteClick}
          >
            招待
          </Button>
        ))}
```

In words:
1. If `showActionButtons` is false → show nothing.
2. Otherwise, if the member `isInvited` → show a row (`HStack` = horizontal stack) with "招待解除" (cancel invitation), plus "招待再送" (resend) **only if** the status is `'invited'`.
3. Otherwise → show one "招待" (invite) button.

`InsuranceSettingsIndex.tsx:176-186` switches between two lists with a ternary:

```tsx
      {activeTab === 'health_insurance' ? (
        <HealthInsuranceSettingList
          isLoading={isLoading}
          items={healthInsuranceSettings}
        />
      ) : (
        <WelfarePensionInsuranceSettingList
          isLoading={isLoading}
          items={welfarePensionInsuranceSettings}
        />
      )}
```

---

## 6. Events: reacting to clicks and typing

| Term | Meaning |
|---|---|
| **Event** | Something the user does: click, type, submit, hover. |
| **Event handler** | A function that runs when the event happens. Props named `onXxx`: `onClick`, `onChange`, `onSubmit`. |
| **Event object (`e`)** | Info about the event, passed to the handler. `e.currentTarget.value` = the input's current text. |
| **Callback prop** | A function passed down as a prop so a child can tell its parent "this happened". |

```tsx
// (illustrative example)
<Button onClick={() => alert('clicked')}>押す</Button>          // ✅ passes a function
<Button onClick={handleSave}>保存</Button>                        // ✅ passes a function
<Button onClick={handleSave()}>保存</Button>                      // ❌ CALLS it during render!
<TextField onChange={(e) => console.log(e.currentTarget.value)} />
```

⚠️ `onClick={handleSave()}` runs `handleSave` immediately while rendering, and passes its *return value* as the handler. Always pass the function itself, or wrap in an arrow function when you need arguments: `onClick={() => onSave(trimmed)}`.

In freee-payroll you'll see callback props named `onXxx` (what the parent passes) and local functions named `handleXxx` (what the component defines). Example from `MemberInfoCard.tsx` above: `onClick={onCancelInvitationClick}` — the card doesn't know *how* to cancel an invitation; its parent does.

---

## 7. The freee-payroll pattern: Presentational component + Container

This is **the most important pattern to recognize in `front/`**. Many files export two components:

| Part | Export | Job |
|---|---|---|
| **Presentational component** | `export default Xxx` | Receives everything via **props** and just draws UI. No API calls. Easy to test and show in Storybook. |
| **Container** | `export const XxxContainer` | Fetches data, holds **state**, handles side effects (hooks, API calls), then renders the presentational component with props. |

### 🔎 In freee-payroll: `JoiningEmployeesPanel`

`front/javascripts/home/components/Home/JoiningEmployeesPanel.tsx:9-26` (presentational)

```tsx
type Props = {
  employeeCount: number;
};

const JoiningEmployeesPanel = (props: Props) => {
  const { employeeCount } = props;

  return (
    <NotificationPanel
      buttonLabel="入社手続き依頼一覧"
      data-test="入社手続き依頼一覧"
      text={`${employeeCount}名が入社手続き中です`}
      url="joining_procedures#/joining_procedure_requests"
    />
  );
};

export default JoiningEmployeesPanel;
```

`JoiningEmployeesPanel.tsx:28-62` (container)

```tsx
export const JoiningEmployeesPanelContainer = () => {
  const [employeeCount, setEmployeeCount] = React.useState(0);
  const isUsable = usePlanFeatureUsable();

  const isAvailable =
    hasPermission({ functionKeys: ['employee:*:*'] }) &&
    isUsable('joining_and_resigning');

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

  return (
    <>
      {isAvailable && employeeCount > 0 && (
        <MarginBase mb={1}>
          <ColumnBase border="default" mb={1} paddingSize="zero">
            <JoiningEmployeesPanel employeeCount={employeeCount} />
          </ColumnBase>
        </MarginBase>
      )}
    </>
  );
};
```

You don't need to understand `useState`/`useEffect` yet (next lesson). For now, see the **shape**:
- The presentational `JoiningEmployeesPanel` only knows "I get a number and show `N名が入社手続き中です`".
- The container checks permission (`hasPermission`) and plan (`isUsable`), calls the API `API.apiPJoiningProceduresEmployeesIncompleteGet()` (generated from the Rails Private API, Lesson 15), stores the count, and shows the panel **only if** `isAvailable && employeeCount > 0` — note `employeeCount > 0`, avoiding the `0` trap.

Same pattern in `front/javascripts/renew_settings2/components/GroupImportHistories.tsx:14-30` (presentational) and `:32-58` (`GroupImportHistoriesContainer`), and `home/components/Home/EmployeeMessagePanel.tsx:12-34` / `:36-63`.

**Why split?** The presentational part can be shown in **Storybook** with fake props (Lesson 31) and Storybook stories are automatically snapshot-tested. The container keeps the messy stuff (API, permissions) in one place.

---

## 8. Components inside components (composition)

Components use other components like LEGO. Data flows down:

```
InsuranceSettingsIndexContainer   (fetches data)
 └─ InsuranceSettingsIndex        (props: settings, isLoading, tab...)
     ├─ Breadcrumbs               (Vibes)
     ├─ UnstableTabBar            (Vibes; tabs={TABS.map(...)})
     ├─ HealthInsuranceSettingList  or  WelfarePensionInsuranceSettingList
     └─ ChangeDefaultSettingModal (isOpen, onClose, onSubmit)
```

`children` lets a component wrap others:

```tsx
// (illustrative example)
const Card = ({ children }: { children: React.ReactNode }) => (
  <div className="card">{children}</div>
);

<Card>
  <p>中身</p>
</Card>
```

`React.ReactNode` = "anything React can render": JSX, strings, numbers, `null`, arrays of these. You'll see `children: React.ReactNode` in the Provider components of Lesson 28.

---

## 9. Practice (15 min)

Open a React + TypeScript sandbox (StackBlitz → "React TS", or any sandbox on react.dev).

**Practice 1 — props.** Write `EmployeeRow` with `Props = { name: string; num: string; isRetired?: boolean }` that renders `<li>{name}（{num}）</li>` and appends ` 退職済み` only when `isRetired` is true.

**Practice 2 — list.** Given
```ts
const employees = [
  { id: 1, name: '給与太郎', num: 'EMP001' },
  { id: 2, name: '労務花子', num: 'EMP002', isRetired: true },
];
```
render a `<ul>` of `EmployeeRow`s with the right `key`.

**Practice 3 — event.** Add a `<button>` to each row that calls a prop `onSelect(id: number)`. In the parent, pass `onSelect={(id) => alert(id)}`.

**Practice 4 — read the repo.** Open `front/javascripts/home/components/Home/EmployeeMessagePanel.tsx`. Point to (a) the Props type, (b) the presentational component, (c) the container, (d) the conditional rendering.

<details><summary>Sample solution for 1–3</summary>

```tsx
type Props = {
  id: number;
  isRetired?: boolean;
  name: string;
  num: string;
  onSelect: (id: number) => void;
};

const EmployeeRow = ({ id, isRetired = false, name, num, onSelect }: Props) => (
  <li>
    {name}（{num}）{isRetired && ' 退職済み'}
    <button onClick={() => onSelect(id)}>選択</button>
  </li>
);

export default function App() {
  const employees = [
    { id: 1, name: '給与太郎', num: 'EMP001' },
    { id: 2, name: '労務花子', num: 'EMP002', isRetired: true },
  ];
  return (
    <ul>
      {employees.map((e) => (
        <EmployeeRow key={e.id} {...e} onSelect={(id) => alert(id)} />
      ))}
    </ul>
  );
}
```
`{...e}` is the spread operator (Lesson 22) — it passes every field of `e` as a prop.
</details>

---

## 10. Code Reading Drill (5 min)

Open `front/javascripts/renew_settings2/components/GroupImportHistories.tsx` and answer out loud:
1. What props does `GroupImportHistories` take?
2. What does `{!!groupImports.length && ...}` do, and why the `!!`?
3. Which export would you put in a Storybook story, and which one talks to the API?

(Answers: 1. `groupImports: GroupImport[]`. 2. Shows the "最新N件..." text only if there is at least one import; `!!` turns the number into a boolean so `0` isn't printed. 3. The default export `GroupImportHistories` for Storybook; `GroupImportHistoriesContainer` calls `API.OrganizationGroupImport.index()`.)

---

## 11. Summary

- A **component** is a Capitalized function returning **JSX**; `.tsx` files hold them.
- **Props** flow **down** (one-way data flow); children talk back through **callback props** (`onXxx`).
- Lists: `array.map(item => <X key={item.id} ... />)`. **Keys** give items stable identity.
- Conditional UI: `cond && <X />` (watch the `0` trap) and `cond ? <A /> : <B />`.
- Events: pass a function (`onClick={handle}`), don't call it (`onClick={handle()}`).
- freee-payroll pattern: **default export = presentational**, **`XxxContainer` = data + hooks**. Props are sorted alphabetically (ESLint), `data-test` marks test targets, UI parts come from Vibes.

➡️ Now take **Test26.md**.
