# Lesson 28 — React 3: More Hooks and Shared State

**Time:** ~1 hour (40 min reading + practice, 20 min test)
**Goal:** Read `useMemo`, `useCallback`, `useRef`, custom hooks (`useXxx`), Context providers, `useReducer`, and the Redux "ducks" code in `front/`, and know **which kind of state tool to use when**.
**You need:** Lessons 26–27. A React + TypeScript sandbox (StackBlitz "React TS" or <https://react.dev/learn>).

---

## 1. Why these extra hooks exist

Every re-render runs your component function from the top. That means:
- every `const x = expensive(...)` is **recomputed**,
- every `const handle = () => ...` is a **new function object**,
- every `{ a, b }` object literal is a **new object**.

Usually that's fine. But sometimes you want to:
1. skip recomputing something heavy → **`useMemo`**
2. keep the *same* function between renders (so an effect or child doesn't think it changed) → **`useCallback`**
3. remember a value **without** causing re-renders → **`useRef`**

| Term | Meaning |
|---|---|
| **Memoization** | Caching a result and reusing it until its inputs change. |
| **Referential equality** | Two values are "the same object" (`a === b`). Two different objects with equal contents are **not** `===`. React uses `===`-style comparison for deps. |
| **Stable reference** | A value that stays `===` across renders (e.g. a `useState` setter, a `useCallback` result with unchanged deps). |

```ts
// (illustrative example)
[] === []                 // false — two different arrays
const f = () => 1; f === f // true — the same function
```

---

## 2. `useMemo`: remember a computed value

```tsx
const value = React.useMemo(() => computeSomething(a, b), [a, b]);
```

"Run `computeSomething` on the first render; on later renders, return the cached result unless `a` or `b` changed."

### 🔎 In freee-payroll

`front/javascripts/company_social_insurance_settings/components/InsuranceSettingsIndex.tsx:232-240`

```tsx
  const healthInsuranceSettings = React.useMemo(
    () => healthInsuranceApiSettings.map(toHealthInsuranceSetting),
    [healthInsuranceApiSettings]
  );
  const welfarePensionInsuranceSettings = React.useMemo(
    () =>
      welfarePensionInsuranceApiSettings.map(toWelfarePensionInsuranceSetting),
    [welfarePensionInsuranceApiSettings]
  );
```

The API data (`healthInsuranceApiSettings`) is converted into the shape the screen wants. `useMemo` means: "only redo the `.map` when new API data arrives." Also important: the result is the **same array object** between renders, so any child or effect depending on it doesn't see a "change" every render.

Same file, `:95-105`, memoizes an **object** passed to a modal:

```tsx
  const modalSocialInsuranceSettings = React.useMemo(
    () => ({
      healthInsuranceSettings: toDefaultChangeableSettings(
        healthInsuranceSettings
      ),
      welfarePensionInsuranceSettings: toDefaultChangeableSettings(
        welfarePensionInsuranceSettings
      ),
    }),
    [healthInsuranceSettings, welfarePensionInsuranceSettings]
  );
```

Without `useMemo`, `({ ... })` would be a brand-new object on every render.

---

## 3. `useCallback`: remember a function

```tsx
const handle = React.useCallback((x: number) => { ... }, [deps]);
```

`useCallback(fn, deps)` is the same idea as `useMemo(() => fn, deps)`: you get **the same function object** until `deps` change.

**When does that matter?** When the function is itself a **dependency** of an effect (or of another `useCallback`/`useMemo`). If the function were new every render, the effect would re-run every render — possibly calling the API in an infinite loop.

### 🔎 In freee-payroll

`InsuranceSettingsIndex.tsx:80-83`

```tsx
  const switchTab = React.useCallback((index: number) => {
    const next = TABS[index]?.key ?? 'health_insurance';
    hashHistory.push(`/company_social_insurance_settings?tab=${next}`);
  }, []);
```

`[]` → the function never changes. (`TABS[index]?.key ?? 'health_insurance'` uses `?.` and `??` from Lesson 23: if there's no tab at that index, fall back to health insurance. `hashHistory.push` changes the URL; that's react-router, Lesson 30.)

`front/javascripts/company_social_insurance_settings/hooks/useHealthInsuranceSettings.ts:30-55` — a chain of callbacks:

```tsx
  const handleErrorResponse = React.useCallback(
    (e: unknown) => {
      notify({ message: API.getErrorMessage(e), type: 'error' });
      if (API.isHRError(e)) {
        setFieldErrors(e.response?.data.fields ?? []);
      }
    },
    [notify]
  );

  // 一覧の再取得は保存後などフォーム表示中にも走るため、fieldErrors はクリアしない
  const fetchSettings = React.useCallback(
    async (year: number, month: number) => {
      showLoading();
      try {
        const { data } =
          await API.apiPCompanyHealthInsuranceSettingsYearMonthGet(year, month);
        setSettings(data.company_health_insurance_settings);
      } catch (e) {
        handleErrorResponse(e);
      } finally {
        hideLoading();
      }
    },
    [handleErrorResponse, hideLoading, showLoading]
  );
```

Why so careful? Because the Container calls `fetchSettings` **inside a `useEffect`** and lists it as a dependency (`InsuranceSettingsIndex.tsx:220-230`):

```tsx
  React.useEffect(() => {
    fetchHealthInsuranceSettings(currentYearMonth.year, currentYearMonth.month);
    fetchWelfarePensionInsuranceSettings(
      currentYearMonth.year,
      currentYearMonth.month
    );
  }, [
    currentYearMonth,
    fetchHealthInsuranceSettings,
    fetchWelfarePensionInsuranceSettings,
  ]);
```

If `fetchSettings` were a new function every render, this effect would fetch → set state → re-render → new function → fetch again → forever. `useCallback` keeps it stable, so the effect runs once (or when `currentYearMonth` changes). Also note `API.isHRError(e)` — a **type guard** (Lesson 25) that tells TypeScript `e.response?.data.fields` exists.

> Rule of thumb for reading: `useMemo`/`useCallback` rarely change *what* the code does. They're about *how often* it runs. When reading, you can mentally remove them and understand the logic first.

---

## 4. `useRef`: a box that survives renders without re-rendering

```tsx
const ref = React.useRef(initialValue);
ref.current           // read
ref.current = newVal  // write — does NOT re-render
```

| | `useState` | `useRef` |
|---|---|---|
| Survives re-renders | ✅ | ✅ |
| Changing it re-renders | ✅ | ❌ |
| How to change | `setX(v)` | `ref.current = v` |
| Use for | Anything shown on screen | "Behind the scenes" values: latest input, timers, in-flight flags, DOM elements |

Two uses:
1. **Holding a mutable value** (e.g. "is a request in flight?", "the latest props").
2. **Pointing at a DOM element**: `<input ref={inputRef} />` then `inputRef.current?.focus()`.

### 🔎 In freee-payroll

`front/packages/@freee-payroll/hooks/src/usePrevious.ts:1-11` — a tiny shared hook:

```tsx
import * as React from 'react';

const usePrevious = <T>(value: T) => {
  const ref = React.useRef<T | null>(null);
  React.useEffect(() => {
    ref.current = value;
  });
  return ref.current;
};

export default usePrevious;
```

The effect (no deps → after every render) stores the value *after* the render; the next render returns the stored one → "the value from the previous render." `<T>` makes it generic (Lesson 25).

`front/javascripts/hr_portal/hooks/useAttritionRiskSummary.ts:106-113` — refs to avoid stale data and duplicate requests:

```tsx
  // Latest row/surveyDeliveredAt without forcing the effect to re-run on every
  // referential change. The effect only re-fires when the employee changes.
  const requestInputsRef = useRef({ row, surveyDeliveredAt });
  requestInputsRef.current = { row, surveyDeliveredAt };

  // Tracks the employeeId of an in-flight request so late responses for a
  // previously-selected employee can be discarded.
  const inFlightRef = useRef<number | null>(null);
```

- `requestInputsRef.current` is refreshed on **every render**, so the async code can always read the **latest** `row` without adding `row` to the effect's deps (which would re-run the effect too often). This is the standard fix for a stale closure when you *don't* want a re-run.
- `inFlightRef` remembers which employee's request is in progress; if the user has switched to another employee, the late response is dropped (compare the `isCancelled` flag in Lesson 27).

`front/javascripts/evaluations/components/contextProviders/UnsavedChangesProvider.tsx:37-44` uses `useRef(false)` for "are there unsaved changes?" — the value is only read inside browser event handlers, so it doesn't need to re-render anything.

---

## 5. Custom hooks: your own `useXxx`

| Term | Meaning |
|---|---|
| **Custom hook** | A normal function whose name starts with `use` and which calls other hooks. It packages reusable **stateful logic** (not UI). |

A custom hook returns whatever is useful: a value, an object, functions. Each component that calls it gets its **own** copy of the state.

### 🔎 In freee-payroll

`front/packages/@freee-payroll/hooks/src/useModal.ts:1-15` — shared across features:

```tsx
import * as React from 'react';

// 複数モーダルを管理するための hooks
const useModal = <T extends string>(
  initialModalType: T | undefined = undefined
) => {
  const [modalType, setModalType] = React.useState<T | undefined>(
    initialModalType
  );
  const open = React.useCallback((type: T) => setModalType(type), []);
  const close = React.useCallback(() => setModalType(undefined), []);
  return { open, close, modalType };
};

export default useModal;
```

Usage idea: `const { open, close, modalType } = useModal<'edit' | 'delete'>();` then `<EditModal isOpen={modalType === 'edit'} onClose={close} />`. One state tracks which (if any) modal is open.

Other shared hooks in `front/packages/@freee-payroll/hooks/src/`: `useLoading` (show/hide loading with an in-flight counter), `useTitle`, `usePlanFeatureUsable`, `useHasPermission`, `useMobile`, `usePrevious`, `useTypedSelector`, `useUnsavedChangesWarning`... Check there before writing your own.

**Feature-specific hooks** live in a feature's `hooks/` directory:
- `company_social_insurance_settings/hooks/useHealthInsuranceSettings.ts` — returns `settings`, `fetchSettings`, `changeDefaultSetting`, `isLoading`, `fieldErrors`...: all the API + state logic for one resource, so the Container is short.
- `company_social_insurance_settings/hooks/useKyokaiKenpoRates.ts` — returns `{ isLoading, rates }` (you read its effect in Lesson 27).
- `front/javascripts/migration_projects/hooks/useFetch.ts:18-22` — a **generic** data-fetching hook:

```tsx
export const useFetch = <T>(
  fetcher: (signal: AbortSignal) => Promise<AxiosResponse<T>>,
  key?: string,
  enable: boolean = true
): UseFetchReturn<T> => {
```

It takes any API function, and returns `{ data, error, isLoading, revalidate }` typed with that API's response type `T`. It uses `AbortController` to cancel older requests — a stronger version of `isCancelled`.

Reading tip: when a Container starts with `const { a, b: renamedB } = useSomething();`, open `useSomething` — that's where the real logic is. (`b: renamedB` is renaming during destructuring, Lesson 22 — e.g. `fetchSettings: fetchHealthInsuranceSettings` at `InsuranceSettingsIndex.tsx:206`.)

---

## 6. Context: sharing state without passing props through every level

**Prop drilling** = passing a prop through several components that don't use it, just so a deep child can get it. **Context** solves this.

| Term | Meaning |
|---|---|
| **Context** | A "broadcast channel" for a value. Any component *inside* the Provider can read it directly. |
| **`React.createContext(default)`** | Creates the context object. The default is used only if there's no Provider above. |
| **Provider** | `<XxxContext.Provider value={...}>children</XxxContext.Provider>` — makes `value` available to everything inside. |
| **`useContext(XxxContext)`** | Reads the nearest Provider's value. When the value changes, readers re-render. |

### 🔎 In freee-payroll: the simplest Context

`front/javascripts/shift/components/ShiftSavingContext.tsx:1-30`

```tsx
import * as React from 'react';

type ShiftSavingContextType = {
  isUpdating: boolean;
  setIsUpdating: React.Dispatch<React.SetStateAction<boolean>>;
};

const ShiftSavingContext = React.createContext<ShiftSavingContextType>({
  isUpdating: false,
  setIsUpdating: () => {},
});

export const ShiftSavingProvider = ({
  children,
}: {
  children: React.ReactNode;
}) => {
  const [isUpdating, setIsUpdating] = React.useState<boolean>(false);

  return (
    <ShiftSavingContext.Provider value={{ isUpdating, setIsUpdating }}>
      {children}
    </ShiftSavingContext.Provider>
  );
};

export const useShiftSaving = () => {
  const context = React.useContext(ShiftSavingContext);
  return context;
};
```

The standard 3-part shape you'll see everywhere:
1. **Type + `createContext`** (with a harmless default).
2. **`XxxProvider`** component that owns the state (`useState`) and wraps `children`.
3. **`useXxx()`** hook that calls `useContext` — so other files never import the raw context.

`React.Dispatch<React.SetStateAction<boolean>>` is simply "the type of a `useState` setter for a boolean."

Usage (illustrative example):
```tsx
<ShiftSavingProvider>
  <ShiftTable />      {/* deep inside: const { isUpdating } = useShiftSaving(); */}
  <SaveButton />      {/* const { setIsUpdating } = useShiftSaving(); */}
</ShiftSavingProvider>
```

### 🔎 The "must be inside the Provider" variant

`front/javascripts/product_introduction/components/Context.tsx:15-28`

```tsx
export const ProductIntroductionContext = React.createContext<
  ProductIntroductionContextType | undefined
>(undefined);

export const useProductIntroductionContext =
  (): ProductIntroductionContextType => {
    const context = React.useContext(ProductIntroductionContext);
    if (!context) {
      throw Error(
        'useProductIntroductionContext must be used inside ProductIntroductionContext.Provider'
      );
    }
    return context;
  };
```

The default is `undefined`, and the hook **throws** if no Provider is found. This catches the bug "I forgot to wrap with the Provider" immediately, and the return type is non-`undefined`. The same pattern is in `evaluations/components/contextProviders/UnsavedChangesProvider.tsx:28-30` and `:142-150`.

### 🔎 Cleanup with event listeners

`UnsavedChangesProvider.tsx:81-87`

```tsx
    window.addEventListener('beforeunload', handleBeforeUnload);
    window.addEventListener('popstate', handlePopState);
    return () => {
      window.removeEventListener('beforeunload', handleBeforeUnload);
      window.removeEventListener('popstate', handlePopState);
    };
  }, [setHasUnsavedChangesState]);
```

Subscribe in the effect, unsubscribe in the cleanup. Forgetting the cleanup = **memory leak** (listeners pile up every time the Provider re-mounts).

---

## 7. `useReducer`: state with named actions

When state has several related fields and many ways to change them, scattered `setX` calls get confusing. **`useReducer`** centralizes the rules.

| Term | Meaning |
|---|---|
| **Action** | A plain object describing *what happened*: `{ type: 'FIELD_ERRORS_CLEARED' }`. |
| **Reducer** | A **pure function** `(state, action) => newState`. Never mutates; returns a new object. |
| **`dispatch(action)`** | Sends an action to the reducer. React stores the returned state and re-renders. |
| **Pure function** | Same inputs → same output, no side effects. |

```tsx
const [state, dispatch] = React.useReducer(reducer, initialState);
dispatch({ type: 'SOMETHING_HAPPENED' });
```

### 🔎 In freee-payroll

`front/javascripts/shift/components/ShiftWorkPattern/useShiftWorkPatternEdit.reducer.ts:5-12, 14-18, 46-69` (excerpt)

```tsx
export const ACTION_TYPES = {
  CONFIRM_DIALOG_CLOSED: 'CONFIRM_DIALOG_CLOSED',
  CONFIRM_DIALOG_OPENED: 'CONFIRM_DIALOG_OPENED',
  FIELD_ERRORS_CLEARED: 'FIELD_ERRORS_CLEARED',
  FIELD_ERRORS_FILTERED: 'FIELD_ERRORS_FILTERED',
  HIDE_CHECKBOX_TOGGLED: 'HIDE_CHECKBOX_TOGGLED',
  VALIDATION_FAILED: 'VALIDATION_FAILED',
} as const;

export type State = {
  fieldErrors: Array<HRErrorField>;
  isConfirmDialogOpen: boolean;
  isHideCheckboxChecked: boolean;
};

export function reducer(state: State, action: Action): State {
  switch (action.type) {
    case ACTION_TYPES.VALIDATION_FAILED:
      return { ...state, fieldErrors: action.fieldErrors };
    case ACTION_TYPES.FIELD_ERRORS_CLEARED:
      return { ...state, fieldErrors: [] };
    // ...
    case ACTION_TYPES.CONFIRM_DIALOG_CLOSED:
      return {
        ...state,
        isConfirmDialogOpen: false,
        isHideCheckboxChecked: false,
      };
    // ...
    default:
      return state;
  }
}
```

- `as const` makes the action types literal strings (Lesson 25), and the `Action` type (`:20-38`) is a **discriminated union** on `type` — so in `case VALIDATION_FAILED:` TypeScript knows `action.fieldErrors` exists.
- `{ ...state, fieldErrors: [] }` — copy everything, change one field (never mutate).
- Closing the dialog also resets the checkbox — a rule written **once**, in the reducer.

Used in `useShiftWorkPatternEdit.ts:38` and `:261`:
```tsx
  const [state, dispatch] = React.useReducer(reducer, initialState);
  // ...
        dispatch({ type: ACTION_TYPES.FIELD_ERRORS_CLEARED });
```

---

## 8. Redux and the "ducks" pattern

**Redux** is a separate library: **one global store** for the whole page, updated by reducers and actions — the same ideas as `useReducer`, but shared by every component.

| Term | Meaning |
|---|---|
| **Store** | The single global state object. |
| **Root reducer** | All reducers combined with `combineReducers({ a, b })`; each one owns one slice (key) of the store. |
| **Selector** | A function that picks data from the store: `(s) => s.globalModal`. Read with `useSelector` (freee wrapper: `useTypedSelector`). |
| **`useDispatch`** | Gives you `dispatch` to send actions to the store. |
| **Action creator** | A function that builds an action object: `open(el)` → `{ type: OPEN, payload: el }`. |
| **FSA (Flux Standard Action)** | The action shape used in this repo: `{ type, payload, error?, meta? }`. Global type `FSA<T, P>` in `front/@types/utils.d.ts`. |
| **Ducks pattern** | Put action types, action creators and the reducer for one slice **in one file** (the "duck"), with the reducer as the `default` export. |

### 🔎 A duck: `front/packages/@freee-payroll/ducks/src/globalModal.ts:1-45`

```tsx
const CLOSE = 'GLOBAL_MODAL/CLOSE' as const;
const OPEN = 'GLOBAL_MODAL/OPEN' as const;

export const close = (): FSA<typeof CLOSE, null> => ({
  type: CLOSE,
  payload: null,
});

export const open = (
  modalElement: JSX.Element
): FSA<typeof OPEN, JSX.Element> => ({
  type: OPEN,
  payload: modalElement,
});

type Action = ReturnType<typeof close> | ReturnType<typeof open>;
type State = JSX.Element | undefined | null;

// eslint-disable-next-line default-param-last -- Ignore because of idiom
export default (state: State = null, action: Action): State => {
  if (action.error) {
    return state;
  }

  switch (action.type) {
    case CLOSE: {
      return null;
    }

    case OPEN: {
      return action.payload;
    }

    default:
      return state;
  }
};
```

(Comments removed for brevity.) Type constants → action creators → `Action` union → default-exported reducer. That's a duck.

### 🔎 Wiring and reading

- `front/javascripts/initializers/store.ts:33` — `export const rootReducer = combineReducers({ ... globalModal, ... })`, and `:66` `export type RootReducerType = ReturnType<typeof rootReducer>;`. The store is provided to every page by `initializers/run.tsx` (`<Provider store={store}>`, Lesson 29).
- `front/packages/@freee-payroll/hooks/src/useTypedSelector.ts:6` — `const useTypedSelector: TypedUseSelectorHook<RootReducerType> = useSelector;` — a `useSelector` that knows the store's type.
- `front/packages/@freee-payroll/components/src/GlobalModalContainer.ts:9-12`:
  ```tsx
  const GlobalModalContainer = () => {
    const modalElement = useTypedSelector((s) => s.globalModal);
    return modalElement || null;
  };
  ```
- `front/javascripts/home/components/Home/OnboardingTasksPanel.tsx:625-629` and `:660-663` — read with `useTypedSelector(({ companyGrowthParameters }) => companyGrowthParameters)`, write with `const dispatch = useDispatch();` then `const action = await CompanyGrowthParametersAction.destroy('showing_home_onboarding_tasks'); dispatch(action);`. In this repo, **async action creators call the API and return an FSA**, which you then `dispatch` (see also `holidays/ducks/importHolidays.ts`, `fetchSpecialHolidaySettings`).
- Feature-local ducks also exist: `front/javascripts/holidays/ducks/index.ts` combines three with `combineReducers`.
- Only one file uses Redux Toolkit's `createSlice` (`DEPRECATED_settings/reducers/shalom.ts`) — recognize it, but the repo's norm is hand-written FSA ducks.

`front/packages/@freee-payroll/ducks/README.md` says shared ducks are for data that isn't tied to one screen: global errors, navigation context, and plain serialized company settings (e.g. `CompanyAllowances`) — not screen-specific data like `Employees`.

---

## 9. Which state tool? (freee-payroll rule)

From `front/README.md:21`:

> React の状態管理は local state を基本とすること。同一 feature 内の複数画面をまたぐ状態の管理には React.Context の利用を検討しても良い。複数 feature を跨いで再利用する状態の管理には Redux を ducks pattern で利用することを検討してよい。

| Situation | Use |
|---|---|
| Only this component (and its children via props) | **`useState`** (or `useReducer` if the logic is complex) |
| Shared across several screens/components **within one feature** | **Context** (Provider + `useXxx`) |
| Reused **across features** | **Redux ducks** (`@freee-payroll/ducks`) |
| Behind-the-scenes value that shouldn't re-render | **`useRef`** |
| Reusable stateful logic | **custom hook** |

---

## 10. Practice (15 min)

In a React TS sandbox:

**Practice 1 — useMemo.** Given `employees: {id: number; name: string; isRetired: boolean}[]` in state, compute `activeEmployees` with `useMemo` (filter out retired). Log inside the memo function to see when it re-runs; add an unrelated counter button and confirm the memo doesn't re-run when only the counter changes.

**Practice 2 — Context.** Build `SelectedEmployeeProvider` + `useSelectedEmployee()` (throws if outside the Provider) holding `selectedId` / `setSelectedId`. Use it in two sibling components without passing props.

**Practice 3 — reducer.** Write a reducer for a dialog: state `{ isOpen: boolean; agreed: boolean }`, actions `OPENED`, `CLOSED` (also resets `agreed`), `AGREE_TOGGLED`. Wire it with `useReducer`.

**Practice 4 — custom hook.** Extract Practice 3 into `useAgreementDialog()` returning `{ isOpen, agreed, open, close, toggleAgree }` with `useCallback`s.

<details><summary>Sample solution for 3–4</summary>

```tsx
const ACTION_TYPES = {
  OPENED: 'OPENED',
  CLOSED: 'CLOSED',
  AGREE_TOGGLED: 'AGREE_TOGGLED',
} as const;

type State = { agreed: boolean; isOpen: boolean };
type Action =
  | { type: typeof ACTION_TYPES.OPENED }
  | { type: typeof ACTION_TYPES.CLOSED }
  | { type: typeof ACTION_TYPES.AGREE_TOGGLED };

const initialState: State = { agreed: false, isOpen: false };

function reducer(state: State, action: Action): State {
  switch (action.type) {
    case ACTION_TYPES.OPENED:
      return { ...state, isOpen: true };
    case ACTION_TYPES.CLOSED:
      return { ...state, agreed: false, isOpen: false };
    case ACTION_TYPES.AGREE_TOGGLED:
      return { ...state, agreed: !state.agreed };
    default:
      return state;
  }
}

const useAgreementDialog = () => {
  const [state, dispatch] = React.useReducer(reducer, initialState);
  const open = React.useCallback(() => dispatch({ type: ACTION_TYPES.OPENED }), []);
  const close = React.useCallback(() => dispatch({ type: ACTION_TYPES.CLOSED }), []);
  const toggleAgree = React.useCallback(
    () => dispatch({ type: ACTION_TYPES.AGREE_TOGGLED }),
    []
  );
  return { ...state, close, open, toggleAgree };
};
```
`dispatch` from `useReducer` is stable, so `[]` deps are correct.
</details>

---

## 11. Code Reading Drill (5 min)

Open `front/javascripts/company_social_insurance_settings/components/InsuranceSettingsIndex.tsx` and find:
1. Every `React.useState`, `React.useCallback`, `React.useMemo`, `React.useEffect`. For each, say in one sentence *why* it's there.
2. Which custom hooks the Container calls, and which names are renamed during destructuring.
3. Which props flow from `InsuranceSettingsIndexContainer` down to `InsuranceSettingsIndex`.

---

## 12. Summary

- **`useMemo(fn, deps)`** caches a computed value; **`useCallback(fn, deps)`** caches a function. Both keep **stable references** so effects/children don't re-run needlessly. freee-payroll wraps functions used as effect deps in `useCallback`.
- **`useRef`** is a mutable box (`.current`) that survives renders **without** re-rendering: latest values, in-flight flags, DOM nodes.
- **Custom hooks** (`useXxx`) package stateful logic; shared ones live in `@freee-payroll/hooks`, feature ones in `<feature>/hooks/`.
- **Context** = `createContext` + `XxxProvider` (owns state, wraps `children`) + `useXxx()` (calls `useContext`, often throws if no Provider).
- **`useReducer`**: `dispatch(action)` → pure `reducer(state, action)` → new state. Action types `as const` + discriminated-union `Action`.
- **Redux ducks**: one file per slice with type constants, FSA action creators and a default-exported reducer; combined in `initializers/store.ts`; read with `useTypedSelector`, write with `useDispatch()(await actionCreator())`.
- Choose: **local state → Context (within a feature) → Redux ducks (across features)** (`front/README.md`).

➡️ Now take **Test28.md**.
