# Test 25 — TypeScript 2

**Rules:** Don't look at Lesson25.md while answering. You may check yourself in the TypeScript Playground **after** writing your answer. Then check `Answers25.md`.
**Total:** 39 points

---

## Part A — Multiple choice (1 point each)

**1.** In `Array<string>`, `string` is the:
- a) type parameter
- b) type argument
- c) constraint
- d) return type

**2.** `Pick<User, 'id'>` produces:
- a) a type with only `id`
- b) a type with everything except `id`
- c) a type with all fields optional
- d) the value of `user.id`

**3.** Which utility type makes every field optional?
- a) `Required<T>`
- b) `Omit<T>`
- c) `Partial<T>`
- d) `Readonly<T>`

**4.** What does `x!` (the non-null assertion) do at runtime?
- a) Throws if x is null
- b) Nothing; it only tells the compiler "trust me, x isn't null/undefined"
- c) Converts x to a boolean
- d) Deep-copies x

**5.** A `.d.ts` file contains:
- a) CSS
- b) only type declarations, no runnable code
- c) compiled JavaScript
- d) test cases

**6.** Where is the type of the global `$FREEE_DATA` declared?
- a) `app/helpers/js_exports_helper.rb`
- b) `front/@types/freee-data.d.ts`
- c) `tsconfig.json`
- d) It has no type (it's `any`)

**7.** `api.ts` under `front/packages/@freee-payroll/api/auto-generated/` is:
- a) hand-written and should be edited for each new endpoint
- b) generated from the OpenAPI (swagger) schema; don't edit it by hand
- c) a test file
- d) a Redux store

**8.** Why use zod when TypeScript already has types?
- a) zod makes the build faster
- b) TS types are erased at runtime; zod validates real data at runtime and can produce the TS type
- c) zod replaces React
- d) zod is required by Rails

**9.** Which is a **discriminated union**?
- a) `string | number`
- b) `{ kind: 'ok'; data: X } | { kind: 'error'; message: string }`
- c) `A & B`
- d) `Record<string, number>`

---

## Part B — Fill in the blanks (1 point each blank)

**10.** A function whose return type is written `value is HRError` is called a type ______.

**11.** `keyof { a: 1; b: 2 }` is the type ______.

**12.** `tsconfig.json` sets `______: true`, which is why the repo writes `TABS[index]?.key` instead of `TABS[index].key`.

**13.** Packages like `@freee-payroll/api` resolve because of Yarn ______, not tsconfig `paths`.

**14.** `z.______<typeof Schema>` gives the TypeScript type described by a zod schema.

**15.** `'GLOBAL_MODAL/CLOSE' as ______` keeps the exact literal type instead of widening to `string`.

---

## Part C — Fill in the missing code (2 points each)

**16.** Make TS require a label for every kind:

```ts
type Kind = 'monthly' | 'daily' | 'hourly';
const KIND_LABEL: ______<Kind, string> = {
  monthly: '月給',
  daily: '日給',
  hourly: '時給',
};
```

**17.** Create `LoadedProps` from `Props`, replacing `employee: Employee | null` with a non-null `employee: Employee`:

```ts
type LoadedProps = ______<Props, 'employee'> ______ { employee: Employee };
```

**18.** Complete the generic function so `last([1, 2, 3])` is typed `number | undefined`:

```ts
const last = ______(xs: ______[]): ______ | undefined => xs[xs.length - 1];
```

---

## Part D — Write the code (3 points each)

**19.** Write a discriminated union type `FetchState<T>` with three variants: loading (no data), error (with `message: string`), ok (with `data: T`). Use `kind` as the discriminant.

**20.** Write a type guard `isString(x: unknown): x is string`, and use it in `const upper = (x: unknown) => ...`, which returns the uppercase string, or `''` if `x` isn't a string.

**21.** Write a zod schema `EmployeeSchema` for `{ id: number; display_name: string }` and the matching TS type `Employee` derived from it.

---

## Part E — Answer in a sentence or two (2 points each)

**22.** Why does the second `return` below fail to compile?

```ts
type State = { kind: 'loading' } | { kind: 'ok'; rows: number[] };
const count = (s: State) => {
  if (s.kind === 'ok') return s.rows.length;
  return s.rows.length;
};
```

**23.** A teammate adds a new feature flag `foo_enabled` in `app/helpers/js_exports_helper.rb` and writes `$FREEE_DATA.ff.foo_enabled` in a `.tsx` file. CI's `yarn run tsc` fails. What did they forget?

---

## Part F — Read real repo code (5 points)

**24.** From `front/packages/@freee-payroll/api/src/errorHandler.ts:12-28` and `:53-63` (abridged):

```ts
export const isHRError = (e: any): e is AxiosError<HRError> => {
  if (!axios.isAxiosError(e)) {
    return false;
  }
  const data = e.response?.data;
  if (typeof data !== 'object') {
    return false;
  }
  if (typeof data.message !== 'string') {
    return false;
  }
  return (
    data.fields == null ||
    (Array.isArray(data.fields) &&
      data.fields.every((field: any) => isHRErrorField(field)))
  );
};

export const getErrorMessage = (e: any) => {
  const defaultMessage = 'エラーが発生しました。 ...';
  if (isHRError(e)) {
    return e.response?.data.message ?? defaultMessage;
  }
  if (isLegacyHRError(e)) {
    return e.response?.data.messages[0] ?? defaultMessage;
  }
  return defaultMessage;
};
```

(a) What does `e is AxiosError<HRError>` tell TypeScript?
(b) List the conditions an error must pass for `isHRError` to return `true`.
(c) Which Rails code produces the JSON shape that `isHRError` recognizes? (Name the concern and the JSON keys.)
(d) What does `?? defaultMessage` protect against?
(e) The file has `any` everywhere, yet `no-explicit-any` is an error in this repo. How is that allowed?
