# Test 23 — JavaScript Basics 2

**Rules:** Don't look at Lesson23.md while answering. You may run code with `node -e` **after** writing your answer. Then check `Answers23.md`.
**Total:** 38 points

---

## Part A — Multiple choice (1 point each)

**1.** Which of these is **truthy** in JavaScript?
- a) `0`
- b) `''`
- c) `[]`
- d) `null`

**2.** What is `0 ?? 10`?
- a) `0`
- b) `10`
- c) `null`
- d) `undefined`

**3.** What is `0 || 10`?
- a) `0`
- b) `10`
- c) `true`
- d) `false`

**4.** `x != null` is true when `x` is:
- a) only not `null`
- b) neither `null` nor `undefined`
- c) truthy
- d) a number

**5.** What is a *Promise*?
- a) A comment that promises to fix the code later
- b) An object representing a value that will be available later (pending → fulfilled/rejected)
- c) A type annotation
- d) A synchronous function

**6.** Which Ruby keyword corresponds to JS `finally`?
- a) `rescue`
- b) `raise`
- c) `ensure`
- d) `begin`

**7.** `xs.find((x) => x.id === 99)` when nothing matches returns:
- a) `null`
- b) `-1`
- c) `false`
- d) `undefined`

**8.** Which JS array method is Ruby's `any?`?
- a) `every`
- b) `some`
- c) `includes`
- d) `filter`

**9.** What does `await` do?
- a) Makes the whole browser wait
- b) Pauses the current `async` function until the Promise settles and gives its value
- c) Creates a new thread
- d) Converts a value to a boolean

---

## Part B — Fill in the blanks (1 point each blank)

**10.** Always compare with ______ (three characters) instead of `==`.

**11.** `user?.name` is JavaScript's version of Ruby's ______.

**12.** An `async` function always returns a ______.

**13.** In a `switch`, each `case` should end with `return` or ______, otherwise execution falls through.

**14.** `Object.______(obj)` turns an object into an array of `[key, value]` pairs; `Object.______(pairs)` turns it back.

---

## Part C — Fill in the missing code (2 points each)

**15.** Return the message, or `'エラー'` if `e.response` or its `data.message` is missing:

```js
const msg = e.response______data.message ______ 'エラー';
```

**16.** Complete to keep only active items and return their names:

```js
const names = items.______((i) => i.is_active).______((i) => i.name);
```

**17.** Complete the error handling:

```js
const load = async () => {
  ______ {
    const { data } = await API.apiPCommentsGet();
    setComments(data);
  } ______ (e) {
    notify({ message: API.getErrorMessage(e), type: 'error' });
  } ______ {
    hideLoading();
  }
};
```

---

## Part D — Write the code (3 points each)

**18.** Write `countErrors`, an arrow function that takes an array of field errors like `[{ name: 'a', message: '...' }, ...]` and a field `name`, and returns **how many** errors have that name. Use `filter` (or `reduce`).
`countErrors([{ name: 'a' }, { name: 'b' }, { name: 'a' }], 'a')` → `2`

**19.** Write an async IIFE that: waits for `fetchCount()` (a function returning a Promise of a number), then `console.log`s `'count: N'`. If it throws, log `'failed'`.

**20.** Write `toLabel(status)` using a **`switch`**: `'fixed'` → `'確定'`, `'draft'` → `'下書き'`, anything else → `'不明'`.

---

## Part E — Answer in a sentence or two (2 points each)

**21.** Why does `{count && <Badge />}` sometimes display a stray `0` in React, and how do you fix it?

**22.** Why do frontends use asynchronous code for API calls instead of simply waiting?

---

## Part F — Read real repo code (4 points)

**23.** From `front/javascripts/home/components/Home/JoiningEmployeesPanel.tsx:36-49`:

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

(a) What is `(async () => { ... })();` called, and why is it written this way?
(b) What HTTP method and URL does `apiPJoiningProceduresEmployeesIncompleteGet` call? (Guess from the name.)
(c) What gets stored with `setEmployeeCount`?
(d) What happens to the user if the request fails?
