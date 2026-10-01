# Test 27 — React 2: State and Effects

**Rules:** Don't look at Lesson27.md while answering. Write your answers under each question. Then check `Answers27.md`.
**Total:** 37 points

---

## Part A — Multiple choice (1 point each)

**1.** What is *state* in React?
- a) Data passed from the parent that the child cannot change
- b) Data a component remembers between renders; changing it with the setter triggers a re-render
- c) A row in the database
- d) A CSS class

**2.** What does `React.useState(0)` return?
- a) The number `0`
- b) A Promise
- c) An array `[currentValue, setterFunction]`
- d) An object `{ value, set }`

**3.** What is a *side effect*?
- a) A bug
- b) Work that reaches outside the component: API calls, timers, event listeners, changing the page title
- c) A CSS animation
- d) A prop with a default value

**4.** `React.useEffect(fn, [])` runs `fn`:
- a) After every render
- b) Never
- c) Once, after the first render (on mount)
- d) Only when the user clicks

**5.** When does React run the *cleanup function* returned from an effect?
- a) Never
- b) Before the effect runs again, and when the component is unmounted
- c) Only when the page is reloaded
- d) Before the first render

**6.** What is a *controlled input*?
- a) An input disabled by an admin
- b) An input whose value comes from state and whose `onChange` updates that state
- c) An input validated on the server
- d) An input inside a `<form>`

**7.** Which of these breaks the *Rules of Hooks*?
- a) Calling `useState` at the top of a component
- b) Calling `useEffect` inside `if (isOpen) { ... }`
- c) Calling a custom hook `useLoading()` at the top of another custom hook
- d) Calling two `useState`s in a row

**8.** What is a *stale closure*?
- a) A function that remembers an old value from the render in which it was created
- b) A closed modal
- c) An expired session
- d) An unused import

---

## Part B — Fill in the blanks (1 point each)

**9.** "Moving shared state to the closest common parent and passing it down as props" is called ______ state up.

**10.** freee-payroll enforces the ESLint rule `react-hooks/______-deps`, which requires listing every value an effect uses in its dependency array.

**11.** You can't pass an `async` function directly to `useEffect` because an async function always returns a ______, but the effect must return nothing or a cleanup function.

**12.** In `setCount((prev) => prev + 1)`, passing a function to the setter is called a ______ updater and always uses the latest value.

**13.** In freee-payroll, API errors in a container are usually shown with `notify({ message: API.______(e), type: 'error' })`.

---

## Part C — Fill in the missing code (2 points each)

**14.** Make this a controlled input.
```tsx
const [memo, setMemo] = React.useState('');
<TextField
  onChange={(e) => ______}
  value={______}
/>
```

**15.** Add a new employee to the `employees` state array **without mutating** it.
```tsx
const [employees, setEmployees] = React.useState<Employee[]>([]);
const add = (e: Employee) => {
  setEmployees(______);
};
```

**16.** Complete the effect so it refetches when `year` changes and ignores out-of-date responses.
```tsx
React.useEffect(() => {
  let isCancelled = false;
  (async () => {
    const { data } = await API.apiPHolidaysYearGet(year);
    if (______) {
      setHolidays(data.holidays);
    }
  })();
  return () => {
    ______;
  };
}, [______]);
```

**17.** Fix the Rules-of-Hooks bug:
```tsx
const Panel = ({ isVisible }: Props) => {
  if (!isVisible) {
    return null;
  }
  const [count, setCount] = React.useState(0);
  return <p>{count}</p>;
};
```

---

## Part D — Write the code (5 points)

**18.** Write `CommentsContainer`:
- state `comments` (type `Comment[]`, start empty) and `isLoading` (boolean)
- on mount, set loading, call `await API.apiPCommentsGet()` and store `data` into `comments`; on error call `notify({ message: API.getErrorMessage(e), type: 'error' })`; always stop loading
- get `notify` from `useNotification()`
- render `<Loading isLoading={isLoading}><CommentList comments={comments} /></Loading>`
Follow the freee-payroll shape (IIFE inside `useEffect`, correct deps).

---

## Part E — Answer in a sentence or two (2 points each)

**19.** Why can't a normal `let` variable replace `useState` for a counter?

**20.** Explain the race condition that the `isCancelled` flag in `useKyokaiKenpoRates` prevents.

**21.** Why does React require hooks to be called in the same order on every render?

---

## Part F — Read real repo code (5 points)

**22.** From `front/javascripts/renew_settings2/components/GroupImportHistories.tsx:32-58`:
```tsx
export const GroupImportHistoriesContainer = () => {
  useTitle('部門のインポート履歴');
  const [isLoading, setIsLoading] = React.useState(false);
  const [groupImports, setGroupImports] = React.useState<GroupImport[]>([]);

  const notify = useNotification();

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
- a) What does the user see on the very first render? (1)
- b) When does the effect run? (1)
- c) What happens if the API call fails? (1)
- d) Why is `<GroupImport[]>` written after `useState`? (1)
- e) Name the component that receives the data as props and say why it's separate. (1)
