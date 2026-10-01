# Test 28 — React 3: More Hooks and Shared State

**Rules:** Don't look at Lesson28.md while answering. Write your answers under each question. Then check `Answers28.md`.
**Total:** 39 points

---

## Part A — Multiple choice (1 point each)

**1.** What does `React.useMemo(() => compute(a), [a])` do?
- a) Runs `compute` on every render
- b) Caches the result and recomputes only when `a` changes
- c) Saves `a` to the database
- d) Creates a context

**2.** Main reason freee-payroll wraps `fetchSettings` in `React.useCallback`:
- a) To make it run faster
- b) So it keeps the same function reference and doesn't make an effect that depends on it re-run every render
- c) So it can be async
- d) Because ESLint forbids arrow functions

**3.** Changing `ref.current` of a `useRef`:
- a) Triggers a re-render
- b) Does not trigger a re-render
- c) Resets all state
- d) Throws an error

**4.** A *custom hook* is:
- a) A function whose name starts with `use` and that calls other hooks to package reusable stateful logic
- b) A component that returns JSX
- c) A Rails callback
- d) A CSS selector

**5.** What problem does *Context* solve?
- a) Slow API calls
- b) Prop drilling: passing props through many components that don't use them
- c) Type errors
- d) Database migrations

**6.** A *reducer* is:
- a) A function that makes images smaller
- b) A pure function `(state, action) => newState`
- c) A hook that calls the API
- d) A Redux store

**7.** In freee-payroll's Redux "ducks" pattern, one duck file contains:
- a) Only JSX
- b) Action type constants, action creators and the reducer (as the default export) for one slice of state
- c) A Rails model
- d) CSS for one component

**8.** According to `front/README.md`, state that is reused **across multiple features** should consider:
- a) Local `useState` only
- b) React Context
- c) Redux with the ducks pattern
- d) A global `let` variable

---

## Part B — Fill in the blanks (1 point each)

**9.** Two different objects with the same contents are not `===`; React compares deps by ______ equality.

**10.** To read a Context value inside a component, call `React.______(XxxContext)`.

**11.** The component that makes a context value available to its children is called the ______.

**12.** `const [state, ______] = React.useReducer(reducer, initialState);`

**13.** The action shape used by freee-payroll ducks (`{ type, payload, error? }`) is called ______ (three letters).

**14.** freee-payroll's typed version of Redux's `useSelector` is `use______Selector`.

---

## Part C — Fill in the missing code (2 points each)

**15.** Memoize the filtered list.
```tsx
const activeEmployees = React.______(
  () => employees.filter((e) => !e.isRetired),
  [______]
);
```

**16.** Complete the reducer case so it clears `fieldErrors` without mutating `state`.
```tsx
case ACTION_TYPES.FIELD_ERRORS_CLEARED:
  return ______;
```

**17.** Complete the "must be used inside the Provider" hook.
```tsx
const SelectedEmployeeContext = React.createContext<Value | undefined>(______);

export const useSelectedEmployee = (): Value => {
  const context = React.useContext(SelectedEmployeeContext);
  if (______) {
    throw new Error('useSelectedEmployee must be used inside SelectedEmployeeProvider');
  }
  return context;
};
```

**18.** Complete the effect that subscribes to a window event and cleans up.
```tsx
React.useEffect(() => {
  const handleResize = () => setWidth(window.innerWidth);
  window.addEventListener('resize', handleResize);
  return () => {
    ______;
  };
}, []);
```

---

## Part D — Write the code (5 points each)

**19.** Write a Context module `EditingContext.tsx` with:
- type `{ isEditing: boolean; setIsEditing: React.Dispatch<React.SetStateAction<boolean>> }`
- `EditingProvider` that owns `isEditing` with `useState(false)` and wraps `children`
- `useEditing()` hook that returns the context value
(Follow the shape of `ShiftSavingContext.tsx`.)

**20.** Write a duck file `bannerVisibility.ts` in the freee-payroll style:
- action type constants `SHOW = 'BANNER_VISIBILITY/SHOW' as const` and `HIDE = 'BANNER_VISIBILITY/HIDE' as const`
- action creators `show()` and `hide()` returning `FSA<typeof SHOW, null>` / `FSA<typeof HIDE, null>`
- `type Action` as a union using `ReturnType`
- a default-exported reducer with state `boolean` (default `false`) that returns `true` on SHOW, `false` on HIDE, and the unchanged state otherwise.

---

## Part E — Answer in a sentence or two (2 points each)

**21.** When would you choose `useRef` instead of `useState`? Give an example from the lesson.

**22.** Explain what `usePrevious` returns and how it works:
```tsx
const usePrevious = <T>(value: T) => {
  const ref = React.useRef<T | null>(null);
  React.useEffect(() => {
    ref.current = value;
  });
  return ref.current;
};
```

---

## Part F — Read real repo code (3 points)

**23.** From `front/javascripts/home/components/Home/OnboardingTasksPanel.tsx`:
```tsx
  const growthParameters = useTypedSelector(
    ({ companyGrowthParameters }) => companyGrowthParameters
  );
  const notify = useNotification();
  const dispatch = useDispatch();
  // ...
    const action = await CompanyGrowthParametersAction.destroy(
      'showing_home_onboarding_tasks'
    );
    dispatch(action);
```
- a) Where does `growthParameters` come from? (1)
- b) What does `CompanyGrowthParametersAction.destroy(...)` do and return? (1)
- c) What happens after `dispatch(action)`? (1)
