# Test 24 — TypeScript 1

**Rules:** Don't look at Lesson24.md while answering. You may check yourself in the TypeScript Playground **after** writing your answer. Then check `Answers24.md`.
**Total:** 37 points

---

## Part A — Multiple choice (1 point each)

**1.** What happens to TypeScript types when the code runs in the browser?
- a) They are checked again at runtime
- b) They are erased; the browser only runs plain JavaScript
- c) They are converted into Ruby
- d) They slow the page down

**2.** Which keyword does freee-payroll use to define object shapes?
- a) `interface`
- b) `class`
- c) `type`
- d) `struct`

**3.** `note?: string` means:
- a) `note` must be a string and cannot be missing
- b) `note` may be missing (undefined), otherwise it's a string
- c) `note` may be `null` but not missing
- d) `note` is a boolean

**4.** `'todo' | 'in_progress' | 'completed'` is a:
- a) intersection type
- b) union of string literal types
- c) enum
- d) array type

**5.** What does `A & B` mean?
- a) A or B
- b) A without B
- c) A value that has all properties of A and all properties of B
- d) Compare A and B

**6.** Why is `unknown` preferred over `any`?
- a) It's faster
- b) `unknown` forces you to check the value before using it, while `any` disables checking
- c) `any` doesn't exist in TypeScript
- d) `unknown` can only hold strings

**7.** Which CI commands type-check the frontend in freee-payroll?
- a) `bundle exec srb typecheck`
- b) `yarn run tsc` and `yarn run tsgo`
- c) `yarn build`
- d) `bin/rspec`

**8.** In Sorbet you'd write `T.nilable(String)`. The closest TypeScript is:
- a) `string?`
- b) `string | null`
- c) `String!`
- d) `Nullable<String>`

---

## Part B — Fill in the blanks (1 point each blank)

**9.** TypeScript figuring out a type without an annotation is called type ______.

**10.** A type that allows only one exact value, like `'draft'`, is called a ______ type.

**11.** The type of a function that takes a `number | null` and returns nothing is `(value: number | null) => ______`.

**12.** An array that can't be pushed to is typed `______<number>`.

**13.** By freee-payroll convention, function props are named starting with ______, and the functions you pass to them start with ______.

---

## Part C — Fill in the missing code (2 points each)

**14.** Complete so `status` can only be `'fixed'` or `'draft'`:

```ts
type PayrollStatus = ______;
```

**15.** Complete the type so `retireDate` is **required** but may be `null`, and `memo` is **optional**:

```ts
type Employee = {
  id: number;
  memo______: string;
  retireDate: ______;
};
```

**16.** Annotate the parameters and return type:

```ts
const formatYen = (amount______, withMark______)______ =>
  withMark ? `¥${amount}` : String(amount);
```
(`amount` is a number, `withMark` is a boolean, result is a string.)

---

## Part D — Write the code (3 points each)

**17.** Write a `type Props` for a component that receives:
- `title` (string, required)
- `count` (number or null, required)
- `isOpen` (boolean, optional)
- `onClose` (function with no arguments, returns nothing, required)

Remember the repo's lint rule for field order.

**18.** Write a type `WithTimestamps` = `{ createdAt: string; updatedAt: string }` and then `type Comment` which is a `{ id: number; body: string }` **intersected** with `WithTimestamps`.

**19.** Write a function `labelOf(kind: HealthInsuranceKind): string` for `type HealthInsuranceKind = 'kyokai_kenpo' | 'kumiai_kenpo' | 'no_entry'` returning `'協会けんぽ'`, `'組合健保'`, `'未加入'`.

---

## Part E — Answer in a sentence or two (2 points each)

**20.** Which lines produce type errors, and why?

```ts
type Status = 'todo' | 'completed';
const s1: Status = 'todo';
const s2: Status = 'Todo';
let count: number = 0;
count = '1';
```

**21.** You see `// eslint-disable-next-line @typescript-eslint/no-explicit-any -- Too legacy to add type` above a line. What does this tell you about the repo's rules?

---

## Part F — Read real repo code (4 points)

**22.** From `front/javascripts/company_social_insurance_settings/components/common/RateInputRow.tsx:31-39`:

```ts
type Props = {
  companyRate: number | null;
  employeeRate: number | null;
  fieldErrors?: ReadonlyArray<HRErrorField>;
  onChangeCompanyRate?: (value: number | null) => void;
  onChangeEmployeeRate?: (value: number | null) => void;
  rateType: RateType;
  readOnly: boolean;
};
```

(a) Which props must a parent always pass?
(b) What does `number | null` mean for `companyRate`, in business terms?
(c) Describe `onChangeCompanyRate` in plain words.
(d) Why might `fieldErrors` be a `ReadonlyArray` instead of a normal array?
