# Lesson 24 — TypeScript 1: Types, Unions, Optional Fields, Function Types

**Time:** ~1 hour (40 min reading + practice, 20 min test)
**Goal:** Read TypeScript type annotations: basic types, `type` aliases, object types, optional `?`, `| null`, union and literal types, `&` intersections, `any` vs `unknown`, and function types. After this lesson, a `Props = { ... }` block in any freee-payroll component should read like plain English.
**You need:** Lessons 22–23. Lesson 8 (Sorbet) helps a lot, since it's the same idea for Ruby. Use the [TypeScript Playground](https://www.typescriptlang.org/play) for practice (paste code; errors appear with red underlines).

---

## 1. What TypeScript adds

| Term | Meaning in plain words |
|---|---|
| **Type** | A *kind* of value: "a number", "a string", "an object with an `id` number and a `name` string". |
| **Type annotation** | A label you write after a colon: `let count: number = 0;`. |
| **Type checker** | A program that reads your code *without running it* and reports type mistakes. For TS: **`tsc`** (and in this repo also **`tsgo`**, a new, faster native version). |
| **Compile / transpile** | Convert TS to plain JS. The types are **erased**: the browser never sees them. |
| **Type inference** | TS figures out the type itself: `let n = 1;` → `number`. You don't have to label everything. |
| **Type error** | "You used a value in a way its type doesn't allow," e.g. calling `.toUpperCase()` on a number. |
| **`.ts` / `.tsx`** | TypeScript file / TypeScript + JSX (React) file. |

### Sorbet ↔ TypeScript

| Idea | Sorbet (Ruby, Lesson 8) | TypeScript |
|---|---|---|
| Label a parameter | `sig { params(id: Integer).returns(String) }` | `(id: number): string => ...` |
| Nilable | `T.nilable(String)` | `string \| null` (or `?` for optional, see §5) |
| Array | `T::Array[Integer]` | `number[]` or `Array<number>` |
| Hash / object | `T::Hash[Symbol, String]` | `Record<string, string>` or `{ key: type }` |
| Boolean | `T::Boolean` | `boolean` |
| Anything | `T.untyped` | `any` (banned by lint) / `unknown` |
| Struct | `T::Struct` with `const :id, Integer` | `type X = { id: number }` |
| Strictness | `# typed: strict` | `"strict": true` in `tsconfig.json` |
| Runtime checks? | `sorbet-runtime` **does** check at runtime | **No**: types are erased, so there's no runtime check (use zod for that, Lesson 25) |

🔎 **In freee-payroll:** `tsconfig.json` (repo root) has `"strict": true` and `"noEmit": true` ("only check types; Vite does the actual converting to JS"). CI runs the type check: `.circleci/config.yml:442-443` in the `front_test` job:

```yaml
            yarn run tsc
            yarn run tsgo
```

If your types are wrong, the PR's CI goes red.

---

## 2. Annotations and basic types

```ts
let count: number = 0;
const companyName: string = 'freee';
const isLoading: boolean = false;
const nothing: null = null;
const notSet: undefined = undefined;

const ids: number[] = [1, 2, 3];          // array of numbers
const names: Array<string> = ['A', 'B'];  // same thing, generic style
const locked: ReadonlyArray<number> = [1]; // array you can't push/modify
```

You usually **don't** annotate local variables; inference does it:

```ts
let n = 1;            // TS knows: number
const s = 'text';     // with const, TS infers the exact literal type 'text' (see §6); with let it would be string
n.toUpperCase();      // ❌ error: Property 'toUpperCase' does not exist on type 'number'
```

Annotate **function parameters** (TS can't guess those) and **exported/public shapes** (types for props, API data, state).

### Functions

```ts
const add = (a: number, b: number): number => a + b;
//           ^ params typed        ^ return type

add(1, 2);     // ✅
add('1', 2);   // ❌ Argument of type 'string' is not assignable to parameter of type 'number'.
```

Return type `void` = "returns nothing useful". Return type `Promise<boolean>` = "an async function that eventually gives a boolean".

---

## 3. `type` aliases: naming a shape

A **type alias** gives a name to a type so you can reuse it:

```ts
type Employee = {
  id: number;
  name: string;
  isRetired: boolean;
};

const e: Employee = { id: 1, name: '給与太郎', isRetired: false };   // ✅
const bad: Employee = { id: 1, name: '給与太郎' };                  // ❌ Property 'isRetired' is missing
```

**Object type syntax:** `{ key: type; key2: type; }`. Semicolons (or commas) separate fields.

🔎 **In freee-payroll** `front/javascripts/company_social_insurance_settings/types.ts:5-13`

```ts
export type HealthInsuranceSetting = {
  id: number;
  insurance_kind: HealthInsuranceKind;
  is_default: boolean;
  is_disabled: boolean;
  name: string;
  office_name: string;
  office_symbol: string;
};
```

Notice:
- Field names are `snake_case` here: they mirror the JSON the Rails API sends (Ruby naming). Pure frontend variables are camelCase. Both appear in the repo.
- Fields are **sorted alphabetically**. That's a lint rule (`@typescript-eslint/member-ordering`), not a TS requirement.

### `type` vs `interface`

TypeScript has a second keyword, `interface`, which does almost the same thing:

```ts
interface Employee { id: number; name: string }   // (illustrative example)
```

**freee-payroll always uses `type`.** `.eslintrc.js:146`:

```js
    '@typescript-eslint/consistent-type-definitions': ['error', 'type'],
```

A grep of `front/javascripts` and `front/packages` (excluding generated code) finds ~10,000 `type` declarations and **zero** `interface`s. When you read other projects you'll meet `interface`; treat it as "same idea as `type`".

---

## 4. Props types: the most common place you'll read types

React components receive inputs called **props** (Lesson 26). Their shape is almost always a `type Props = { ... }`.

🔎 `front/javascripts/company_social_insurance_settings/components/common/RateInputRow.tsx:31-39`

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

Read it aloud:
- `companyRate`: a number **or null** (null = not entered). **Required.**
- `fieldErrors?`: **optional**; if given, a read-only array of `HRErrorField` (the `fields` part of the Rails error JSON).
- `onChangeCompanyRate?`: optional; a **function** taking a `number | null` and returning nothing. (The parent passes "what to do when the user edits the value.")
- `rateType`: one of a fixed list of strings (see §6).
- `readOnly`: boolean, required.

Props can also be typed **inline** with destructuring. 🔎 `front/javascripts/employees/components/form/DescriptionListCells/LastName.tsx:4-14`

```tsx
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
```

Left of `}: {` is **destructuring with a default** (`disabled = false`); right is the **type** of the one object parameter. `React.ChangeEventHandler<HTMLInputElement>` is React's ready-made type for "a function that handles a text input's change event".

---

## 5. Optional (`?`) vs nullable (`| null`)

| Written | Meaning | Allowed values |
|---|---|---|
| `note: string` | required string | `'abc'` |
| `note?: string` | **optional**: the key may be missing | `'abc'`, missing, `undefined` |
| `note: string \| null` | required key, value may be **null** | `'abc'`, `null` |
| `note?: string \| null` | may be missing, undefined, or null | all of the above |

🔎 `front/javascripts/hr_portal/types.ts:16-22`

```ts
export type SubActionConfig = {
  description: string;
  id: string;
  name: keyof typeof ATTRITION_RISK_SUB_ACTION_TITLES;
  note?: string | null;
  status: SubActionStatus;
};
```

`note?: string | null`: "a memo that might not be sent at all, or might be sent as `null`." (`keyof typeof` is Lesson 25.)

🔎 `front/javascripts/hr_portal/types.ts:35-45`: many optional fields:

```ts
export type ParentAction = {
  assignees?: Assignee[];
  completedAt: string | null;
  createdAt?: string;
  description: string;
  id: string;
  name: string;
  status?: ParentActionStatus;
  subActions: SubActionConfig[];
  targetEmployee?: TargetEmployee;
};
```

`completedAt: string | null` must be present but is `null` until the action is completed. `createdAt?: string` may be absent.

**Why it matters:** with `strict` on, TS forces you to handle the `null`/`undefined` case before using the value, the #1 source of crashes in JS (`Cannot read properties of undefined`).

```ts
const len = action.createdAt.length;        // ❌ 'action.createdAt' is possibly 'undefined'.
const len = action.createdAt?.length ?? 0;  // ✅
```

---

## 6. Union types and literal types

A **union** `A | B` means "either an A or a B". A **literal type** is a type whose only value is one exact value, like `'draft'`.

Combine them to say "one of these exact strings." This is TypeScript's usual replacement for enums, and it's used everywhere in freee-payroll.

🔎 `front/javascripts/company_social_insurance_settings/types.ts:3`

```ts
export type HealthInsuranceKind = 'kyokai_kenpo' | 'kumiai_kenpo' | 'no_entry';
```

```ts
const k1: HealthInsuranceKind = 'kyokai_kenpo';  // ✅
const k2: HealthInsuranceKind = 'kyokai';        // ❌ Type '"kyokai"' is not assignable to type 'HealthInsuranceKind'.
```

The editor even **autocompletes** the three options. Compare Ruby, where `enumerize :x, in: %i[...]` or a Rails `enum` checks values only at runtime.

🔎 More, `front/javascripts/hr_portal/types.ts:10-14`:

```ts
export type RiskLevel = 'VERY_HIGH' | 'HIGH' | 'POTENTIALLY_HIGH' | 'STABLE';

export type SubActionStatus = 'completed' | 'todo' | 'in_progress';

export type ParentActionStatus = 'in_progress' | 'todo' | 'completed';
```

🔎 A long union written one per line, `front/javascripts/company_social_insurance_settings/components/common/RateInputRow.tsx:12-19`:

```ts
export type RateType =
  | 'adjusted_rate'
  | 'base_rate'
  | 'care_rate'
  | 'child_allowance_contribution_rate'
  | 'child_support_fund_rate'
  | 'specific_rate'
  | 'welfare_pension_insurance_rate';
```

(The leading `|` on the first line is optional; Prettier formats long unions this way.)

Unions aren't only strings: `number | null`, `string | undefined`, `HealthInsuranceSetting | WelfarePensionInsuranceSetting` are all unions.

### Type alias of another type

🔎 `front/javascripts/company_social_insurance_settings/types.ts:26-29`

```ts
export type InsuranceType = 'health_insurance' | 'welfare_pension_insurance';

// 一覧のタブは「どちらの保険を表示するか」そのものなので、同値の union を二重定義せず別名にする
export type InsuranceTab = InsuranceType;
```

The comment says: "the tab list *is* the choice of insurance, so reuse the same union under another name instead of defining it twice."

---

## 7. Intersection types `&`: "all of these"

`A & B` = "has everything from A **and** everything from B." It's how you **extend** a type.

🔎 `front/javascripts/hr_portal/types.ts:59-66`

```ts
export type AttritionRiskItem = SurveyResultItem &
  DummyFields & {
    /** AI対応優先度。サーベイリスクと勤怠リスクから算出する（computeAiPriority）。 */
    aiPriority: RiskLevel;
    aiSubjectiveRisk: RiskLevel;
    /** 勤怠リスク（API由来）。該当従業員の勤怠リスクデータが無い場合は null。 */
    attendanceRisk: RiskLevel | null;
  };
```

"An `AttritionRiskItem` is an API `SurveyResultItem`, plus the `DummyFields`, plus three extra fields." (`/** ... */` is a **doc comment**: editors show it on hover.)

| | `A \| B` (union) | `A & B` (intersection) |
|---|---|---|
| Meaning | either A or B | both A and B combined |
| Plain words | "one of" | "all of" |

---

## 8. `any` vs `unknown`

| Type | Meaning | Allowed in freee-payroll? |
|---|---|---|
| `any` | "Turn off type checking for this value." Anything goes, and mistakes aren't caught. | **Banned** by `.eslintrc.js:236` `'@typescript-eslint/no-explicit-any': 'error'`. You'll see it only with an `eslint-disable` comment explaining why (legacy). |
| `unknown` | "Could be anything, so you must **check** before using it." Safe. | ✅ Preferred for truly unknown data (caught errors, raw JSON). |

```ts
const a: any = 42;
a.toUpperCase();          // no TS error... crashes at runtime 💥

const u: unknown = 42;
u.toUpperCase();          // ❌ TS error: 'u' is of type 'unknown'.
if (typeof u === 'string') {
  u.toUpperCase();        // ✅ inside the check, TS knows it's a string
}
```

🔎 Real `any` with an explanation, `front/javascripts/initializers/run.tsx:22-29`:

```ts
type InitContext =
  | {
      // eslint-disable-next-line @typescript-eslint/no-explicit-any -- Too legacy to add type
      withBackboneRouter: any;
    }
  | {
      withReactRouter: React.ComponentType;
    };
```

"Backbone is too legacy to type, so `any` is allowed here, with a reason." That's the repo's rule for any lint exception: disable **one line**, and say **why** after `--`.

🔎 Real `unknown`, `front/javascripts/hr_portal/hooks/useAttritionRiskSummary.ts:192` (Lesson 23): `.catch((err: unknown) => { ... err instanceof Error ? ... })`.

---

## 9. Function types

A **function type** describes a function's parameters and return value: `(params) => ReturnType`.

```ts
type OnChange = (value: number | null) => void;
type Fetcher = (year: number, month: number) => Promise<void>;
```

🔎 `front/javascripts/company_social_insurance_settings/components/InsuranceSettingsIndex.tsx:32-41`

```ts
type Props = {
  healthInsuranceSettings: ReadonlyArray<HealthInsuranceSetting>;
  isLoading: boolean;
  /** デフォルト設定の保存。成功したら true を返し、呼び出し側でモーダルを閉じる */
  onChangeDefaultSettings: (
    params: ChangeDefaultSettingsParams
  ) => Promise<boolean>;
  tab?: string;
  welfarePensionInsuranceSettings: ReadonlyArray<WelfarePensionInsuranceSetting>;
};
```

`onChangeDefaultSettings` is a function prop: give it `params`, and it returns a Promise that resolves to `true` (saved; the component closes the modal) or `false`.

**Naming convention:** function props start with **`on`** (`onChange`, `onClose`, `onSubmit`), and functions defined inside the component that you pass to them start with **`handle`** (`handleChangeDefaultSettings`).

### `readonly` / `ReadonlyArray`

`ReadonlyArray<T>` (or `readonly T[]`) forbids `push`, `splice`, and similar, which fits the React "don't mutate" rule:

```ts
const xs: ReadonlyArray<number> = [1, 2];
xs.push(3);   // ❌ Property 'push' does not exist on type 'readonly number[]'.
```

---

## 10. Reading a type error

Type errors look scary but have a fixed shape: **"Type X is not assignable to type Y"** means "you gave an X where a Y was required." Example (verified with `tsc --strict`):

```
error TS2322: Type '"kyokai"' is not assignable to type 'HealthInsuranceKind'.
```

Read it right-to-left: "the variable expects `HealthInsuranceKind` (the 3 allowed strings); you gave `"kyokai"`, which is not one of them."

Other common ones:
- `TS2339: Property 'x' does not exist on type 'Y'`: typo, or the field isn't in the type.
- `TS18048: 'x' is possibly 'undefined'`: handle the missing case (`?.`, `??`, or an `if`).
- `TS2345: Argument of type 'string' is not assignable to parameter of type 'number'`: wrong argument type.

---

## 11. Practice (15 min)

Paste each block into the [TypeScript Playground](https://www.typescriptlang.org/play). Predict **which lines error**, then check.

**P1.**
```ts
type Status = 'todo' | 'in_progress' | 'completed';
const a: Status = 'todo';
const b: Status = 'done';
```

**P2.**
```ts
type Memo = { id: number; note?: string | null };
const m1: Memo = { id: 1 };
const m2: Memo = { id: 2, note: null };
const m3: Memo = { note: 'hi' };
```

**P3.**
```ts
type Props = { onSave: (name: string) => void };
const p: Props = { onSave: (name) => console.log(name.toUpperCase()) };
p.onSave(123);
```

**P4. Write it:** a type `Employee` with `id` (number), `displayName` (string), `retireDate` (string or null, required), `departmentName` (optional string).

**P5. Read it:** open `front/javascripts/hr_portal/types.ts` and explain `TargetEmployee` and `Assignee` in one sentence each.

*(Answers: P1 line 3 errors (`'done'` isn't in the union). P2 `m3` errors (missing required `id`); `m1` and `m2` are fine. P3 the last line errors (`123` is not a `string`). Note that inside the arrow, `name` is inferred as `string` from the Props type, so `toUpperCase` is fine. P4 `type Employee = { departmentName?: string; displayName: string; id: number; retireDate: string | null };`. P5 `TargetEmployee` = an employee with `code` (string), `id` (number), `name` (string); `Assignee` = a person responsible, with `id` and `name` strings.)*

---

## 12. Summary

- TypeScript = JS + **types checked before running**; types are erased at build time. CI runs `yarn run tsc` and `yarn run tsgo`.
- Annotate parameters and shared shapes; let inference handle locals.
- `type Name = { field: type; ... }`. **freee-payroll uses `type`, never `interface`** (ESLint rule).
- `field?:` = optional (may be missing); `| null` = may be null. `strict` makes you handle both.
- **Union** `'a' | 'b'` = one of; **intersection** `A & B` = all of. String-literal unions replace enums.
- `any` is banned (except with a reasoned `eslint-disable`); use `unknown` and check it.
- Function types `(x: T) => R`; function props are named `onXxx`; `Promise<T>` for async results; `ReadonlyArray<T>` for arrays you must not mutate.

➡️ Now take **Test24.md**.
