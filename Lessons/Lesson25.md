# Lesson 25 — TypeScript 2: Generics, Utility Types, Narrowing, Global Types, zod

**Time:** ~1 hour (40 min reading + practice, 20 min test)
**Goal:** Read the "advanced-looking" TypeScript in freee-payroll: generics `<T>`, utility types (`Partial`, `Pick`, `Omit`, `Record`, `ReturnType`, `Awaited`), `keyof`/`typeof`, **narrowing** (type guards, discriminated unions), `as`, `as const`, `!`, `satisfies`, `.d.ts` files, the global `$FREEE_DATA`, where API types come from, and runtime validation with zod.
**You need:** Lesson 24. The [TypeScript Playground](https://www.typescriptlang.org/play).

---

## 1. Generics: types with "blanks"

A **generic** is a type (or function) with a *type parameter*, a blank written `<T>` that gets filled in when used. You've already seen them: `Array<string>` is "Array, with the blank filled by `string`".

| Term | Meaning |
|---|---|
| **Generic** | A reusable type/function that works for many types, with the type given as a parameter. |
| **Type parameter** | The blank, conventionally `T` (also `K` for key, `V` for value, `P` for payload...). |
| **Type argument** | What fills the blank: in `Array<number>`, it's `number`. |
| **Constraint** | `T extends X`: "the blank must be something compatible with X". |

Generics you'll see constantly:

| Generic | Meaning |
|---|---|
| `Array<T>` / `T[]` | array of T |
| `ReadonlyArray<T>` | read-only array of T |
| `Promise<T>` | a future value of type T (what `await` gives you) |
| `Record<K, V>` | object whose keys are K and values are V |
| `React.useState<T>(...)` | a piece of state of type T (Lesson 27) |
| `AxiosResponse<T>` | an HTTP response whose `data` is T |

### Generic types (your own)

🔎 `front/javascripts/migration_projects/hooks/useFetch.ts:4-12`

```ts
type UseFetchState<T> = {
  data: T | undefined;
  error: Error | undefined;
  isLoading: boolean;
};

type UseFetchReturn<T> = UseFetchState<T> & {
  revalidate: () => Promise<T | undefined>;
};
```

"`UseFetchState<Employee[]>` has `data: Employee[] | undefined`; `UseFetchState<Company>` has `data: Company | undefined`." One definition, any data type.

### Generic functions

🔎 Same file, `useFetch.ts:18-22`:

```ts
export const useFetch = <T>(
  fetcher: (signal: AbortSignal) => Promise<AxiosResponse<T>>,
  key?: string,
  enable: boolean = true
): UseFetchReturn<T> => {
```

"Give me a fetcher that returns a response with data of type `T`; I'll give you back state whose `data` is `T`." TS **infers** `T` from the fetcher you pass; you rarely write it.

🔎 Passing a type argument explicitly, `front/packages/@freee-payroll/api/src/TaxOffice.ts:7-11`:

```ts
type IndexResponse = { models: Array<TaxOffice> };

export const index = async (pref: string) => {
  const { data } = await axios.get<IndexResponse>(`/api/p/tax_offices/${pref}`);
  return data.models;
```

`axios.get<IndexResponse>(...)` means "I promise the JSON looks like `IndexResponse`," so `data.models` is typed `TaxOffice[]`. (It's a promise to the compiler, not a runtime check. See zod in §8.)

### Generic class

🔎 `front/javascripts/hr_portal/hooks/useAttritionRiskSummary.ts:61-65` (shortened):

```ts
class BoundedCache<K, V> {
  private readonly store = new Map<K, V>();
  constructor(private readonly _capacity: number) {}

  get(key: K): V | undefined {
```

and at line 87: `new BoundedCache<number, AttritionRiskSummaryState>(CACHE_CAPACITY)`, a cache from employee id (number) to a summary state. (`Map<K, V>` is JS's built-in key/value store; `private readonly` = can only be used inside the class and never reassigned.)

### Constraints

🔎 `front/javascripts/migration_projects/types.ts:111-112`

```ts
const withId = <S extends z.ZodRawShape>(schema: z.ZodObject<S>) =>
  schema.extend({ id: z.string() });
```

`S extends z.ZodRawShape` = "S can be any zod object shape, but must *be* one."

🔎 A generic with **defaults**, the global `FSA` type (`front/@types/utils.d.ts:26`):

```ts
declare type FSA<T extends string, P = undefined, M = never> = (
```

"Flux Standard Action": `T` is the action type string, `P` the payload type (default `undefined`), `M` the meta type. Used in Redux ducks (Lesson 28), e.g. `front/packages/@freee-payroll/ducks/src/globalModal.ts:7`:

```ts
export const close = (): FSA<typeof CLOSE, null> => ({
```

---

## 2. `keyof`, `typeof`, indexed access

| Syntax | Meaning | Example |
|---|---|---|
| `typeof x` (in a type position) | "the type of the *value* x" | `typeof ACTION_TYPES` |
| `keyof T` | union of T's keys | `keyof { a: 1; b: 2 }` → `'a' \| 'b'` |
| `T['field']` | the type of one field of T (**indexed access**) | `Employee['id']` → `number` |

> ⚠️ `typeof` has two meanings: in normal code `typeof x === 'string'` is a *runtime* check (Lesson 22); after `:` or in a `type` it's the *compile-time* "type of".

🔎 `keyof typeof`, `front/javascripts/hr_portal/types.ts:19`:

```ts
  name: keyof typeof ATTRITION_RISK_SUB_ACTION_TITLES;
```

`ATTRITION_RISK_SUB_ACTION_TITLES` is an object (`front/javascripts/hr_portal/constants.ts:144`) like `{ checkTeamOrganizationStatus: 'チームの組織状況を確認', ... }`. `typeof` gets its type; `keyof` takes its keys. So `name` must be one of those keys, and adding a key to the constant automatically allows it here.

🔎 Indexed access, `front/javascripts/company_social_insurance_settings/components/HealthInsuranceSetting/SettingList.tsx:27-28`:

```ts
const toHealthInsuranceKind = (
  insuranceType: API.CompanyHealthInsuranceSetting['insurance_type']
```

"Whatever type the API says the `insurance_type` field has." If the backend schema changes, this follows automatically.

---

## 3. Utility types: building new types from old ones

TypeScript ships helper generics:

| Utility | Meaning | Example |
|---|---|---|
| `Partial<T>` | all fields optional | form state that's filled gradually |
| `Required<T>` | all fields required | |
| `Pick<T, 'a' \| 'b'>` | only these fields | |
| `Omit<T, 'a'>` | all fields except these | |
| `Record<K, V>` | object with keys K, values V | lookup tables |
| `ReturnType<typeof f>` | the return type of function f | |
| `Awaited<P>` | the value inside a Promise | `Awaited<Promise<string>>` → `string` |
| `NonNullable<T>` | T without `null`/`undefined` | |

🔎 **All real, verified lines:**

`front/javascripts/home/components/HomeForOthers.tsx:187`: **Pick**:
```ts
type ContainerProps = Pick<Props, 'navigation'>;
```
"The container only takes the `navigation` prop."

`front/javascripts/hr_portal/components/ActionDetailPage.tsx:53-55`: **Omit** + `&`:
```ts
type LoadedProps = Omit<Props, 'parentAction'> & {
  parentAction: ParentAction;
};
```
In `Props`, `parentAction` is `ParentAction | null` (line 50). `LoadedProps` removes it and re-adds it as **non-null**: "props after the data has loaded."

`front/javascripts/hr_portal/components/AttritionRiskPage.tsx:60`: **Partial**:
```ts
type SearchParamsShape = Partial<AttritionRiskFilterState>;
```
URL search params might contain any subset of the filters.

`front/javascripts/company_social_insurance_settings/types.ts:38-43`: **Record**:
```ts
export const HEALTH_INSURANCE_KIND_LABEL: Record<HealthInsuranceKind, string> =
  {
    kyokai_kenpo: '協会けんぽ',
    kumiai_kenpo: '組合健保',
    no_entry: '未加入',
  };
```
Because the key type is the union `HealthInsuranceKind`, TS **forces every kind to have a label**. Add a 4th kind and forget its label, and you get a compile error. That's why the repo prefers lookup objects like this.

`front/javascripts/company_social_insurance_settings/hooks/useCompanyProfileFetcher.test.ts:35-37`: **Awaited + ReturnType**:
```ts
type FetchCompanyProfileResponse = Awaited<
  ReturnType<typeof API.fetchCompanyProfile>
>;
```
"The type you get after awaiting `API.fetchCompanyProfile()`."

---

## 4. Narrowing: how TS learns a more specific type

A value typed `A | B` can only be used as "things both A and B have"... until you **check**. After the check, TS **narrows** the type inside that branch.

| Check | Narrows by | Example |
|---|---|---|
| `typeof x === 'string'` | primitive kind | `unknown` → `string` |
| `x === null` / `x != null` | null-ness | `string \| null` → `string` |
| truthiness `if (x)` | removes null/undefined/falsy | |
| `x instanceof Error` | class | `unknown` → `Error` |
| `'key' in x` | presence of a property | union of object shapes |
| `x.kind === 'ok'` | a **discriminant** field | discriminated union |
| custom `isX(x)` | a **type guard** function | |

🔎 `in`, `front/javascripts/initializers/run.tsx:22-34`:

```ts
type InitContext =
  | {
      // eslint-disable-next-line @typescript-eslint/no-explicit-any -- Too legacy to add type
      withBackboneRouter: any;
    }
  | {
      withReactRouter: React.ComponentType;
    };

export default (initContext: InitContext) => {
  $(() => {
    bootstrap();
    if ('withBackboneRouter' in initContext) {
```

Inside the `if`, TS knows it's the Backbone shape; in the `else if ('withReactRouter' in initContext)` branch it knows it's the React shape.

### Discriminated unions: the most useful pattern

Give every variant of a union a common field with a **different literal value** (often `kind` or `type`). Checking that field narrows to exactly one variant.

🔎 `front/javascripts/hr_portal/hooks/useAttritionRiskSummary.ts:27-30`

```ts
export type AttritionRiskSummaryState =
  | { kind: 'error'; message: string }
  | { kind: 'loading' }
  | { kind: 'ok'; summary: AttritionRiskAiSummary };
```

```ts
// (illustrative example of using it)
if (state.kind === 'ok') {
  state.summary;   // ✅ TS knows summary exists here
} else if (state.kind === 'error') {
  state.message;   // ✅
}
state.summary;     // ❌ outside a check: Property 'summary' does not exist on type 'AttritionRiskSummaryState'
```

This makes "loading / error / success" states impossible to mix up. It's far safer than three separate variables `isLoading`, `error`, `data` that can contradict each other.

🔎 The reducer's actions are a discriminated union on `type`: `front/javascripts/shift/components/ShiftWorkPattern/useShiftWorkPatternEdit.reducer.ts:20-27`, and in `case ACTION_TYPES.VALIDATION_FAILED:` TS knows `action.fieldErrors` exists (line 49).

### User-defined type guards: `value is Type`

A function returning `x is Y` tells TS: "if I return true, treat x as Y."

🔎 `front/packages/@freee-payroll/api/src/errorHandler.ts:12-28`

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
```

After `if (isHRError(e)) { ... }`, TS lets you use `e.response?.data.message` and `.fields` safely. (This file has a file-level `eslint-disable` for `any` at line 1, with a reason.) `HRError` (`front/packages/freee-types/HRError.ts`) is `{ code?: string | null; fields?: Array<HRErrorField>; message: string }`: exactly the Rails `ErrorHandleable` JSON.

🔎 Another, `useAttritionRiskSummary.ts:32-34`: `const isSummarySection = (value: unknown): value is AttritionRiskAiSummary['work_rhythm'] => {`.

---

## 5. Escape hatches: `as`, `!`, `as const`, `satisfies`

| Syntax | Name | Meaning | Safety |
|---|---|---|---|
| `x as T` | type assertion | "Trust me, x is a T." No check. | ⚠️ can lie |
| `x!` | non-null assertion | "Trust me, x isn't null/undefined." | ⚠️ can lie; lint needs a reason |
| `expr as const` | const assertion | "Make this as specific and read-only as possible." | ✅ safe |
| `expr satisfies T` | satisfies | "Check expr matches T, but keep its precise inferred type." | ✅ safe |

🔎 `!`, `front/javascripts/admin2/index.tsx:7-8`:

```tsx
// eslint-disable-next-line @typescript-eslint/no-non-null-assertion -- appは必ず存在する
createRoot(document.querySelector('#app')!).render(
```

`querySelector` returns `Element | null`. The comment says "`#app` always exists" (Rails renders it), so `!` is justified, with a lint exception and a reason.

🔎 `as const`, `front/packages/@freee-payroll/ducks/src/globalModal.ts:1`:

```ts
const CLOSE = 'GLOBAL_MODAL/CLOSE' as const;
```

Without `as const` inside objects, TS widens strings to plain `string`. With it, the type stays the exact literal `'GLOBAL_MODAL/CLOSE'`, needed for discriminated unions.

🔎 `as const` on an object + `typeof` lookup, `useShiftWorkPatternEdit.reducer.ts:5-12` and `:24`:

```ts
export const ACTION_TYPES = {
  CONFIRM_DIALOG_CLOSED: 'CONFIRM_DIALOG_CLOSED',
  CONFIRM_DIALOG_OPENED: 'CONFIRM_DIALOG_OPENED',
  FIELD_ERRORS_CLEARED: 'FIELD_ERRORS_CLEARED',
  FIELD_ERRORS_FILTERED: 'FIELD_ERRORS_FILTERED',
  HIDE_CHECKBOX_TOGGLED: 'HIDE_CHECKBOX_TOGGLED',
  VALIDATION_FAILED: 'VALIDATION_FAILED',
} as const;
...
      type: typeof ACTION_TYPES.VALIDATION_FAILED;
```

`typeof ACTION_TYPES.VALIDATION_FAILED` = the literal type `'VALIDATION_FAILED'`.

🔎 `satisfies`, `front/javascripts/company_social_insurance_settings/components/HealthInsuranceSetting/SettingList.tsx:90-92`:

```ts
  ] satisfies Array<
    IndexPageListContent<ApiSearchParams, HealthInsuranceSetting>
  >;
```

"Check this column list matches what the list component expects, without losing detail."

### `enum` (rare here)

TS has `enum`, but the repo prefers string-literal unions. Only 4 files (outside generated code) declare one, e.g. `front/javascripts/shift/hooks/useHelpWorkDrawer.ts:8-12`:

```ts
export enum HelpWorkDrawerOpenType {
  none = 'none',
  helpMember = 'helpMember',
  mainDutyOtherFacility = 'mainDutyOtherFacility',
}
```

Used as `HelpWorkDrawerOpenType.helpMember`. Read it; prefer unions when writing.

### `noUncheckedIndexedAccess`: why you see `?.` after `[i]`

`tsconfig.json:13` has `"noUncheckedIndexedAccess": true`: "reading `array[i]` or `obj[key]` might give `undefined`." So the repo writes (`front/javascripts/company_social_insurance_settings/components/InsuranceSettingsIndex.tsx:81`):

```ts
    const next = TABS[index]?.key ?? 'health_insurance';
```

Without `?.`, TS errors: `Object is possibly 'undefined'`.

---

## 6. Declaration files (`.d.ts`) and the global `$FREEE_DATA`

| Term | Meaning |
|---|---|
| **Declaration file (`.d.ts`)** | A file with **only types**, no runnable code. Describes things that exist elsewhere. |
| **Ambient declaration** | `declare ...`: "this exists at runtime; here is its type" (TS doesn't create it). |
| **Global** | Available in every file without importing. |

Rails writes `window.$FREEE_DATA = {...}` into every page (`app/views/layouts/_freee_js_include.html.erb`, Lesson 16). TypeScript can't see Ruby, so the repo **declares** its type.

🔎 `front/@types/freee-data.d.ts:26-30`

```ts
import { FeatureFlags } from './feature-flags';

declare global {
  // eslint-disable-next-line no-var -- Must use `var` to define global variable. @see https://www.typescriptlang.org/docs/handbook/declaration-files/by-example.html#global-variables
  var $FREEE_DATA: {
```

…and at lines 191-192:

```ts
    env: EnvName;
    ff: FeatureFlags;
```

🔎 `front/@types/feature-flags.d.ts:2-4`

```ts
export type FeatureFlags = {
  hr_screen_share_banner: boolean;
  cre_nencho_trial_tel_banner: boolean;
```

So `$FREEE_DATA.ff.hr_portal_enabled` is typed `boolean`, and a typo like `$FREEE_DATA.ff.hr_portl_enabled` is a compile error.

> ⚠️ **Important for your work:** when you add a feature flag on the Rails side (`app/helpers/js_exports_helper.rb` `feature_flags`), you must also add it to `FeatureFlags` in `front/@types/feature-flags.d.ts`. The types and the Ruby hash are kept in sync **by hand**.

Other global types live in `front/@types/`: `utils.d.ts` (the `FSA` type), `backbone.d.ts`, `jquery.d.ts`, `react-router.d.ts`...

---

## 7. Where API types come from (you rarely write them)

Remember schema-driven development (Lesson 15):

```
app/controllers/api/private/*_controller.swagger.yaml   (OpenAPI schema, written by hand)
        │  make generate-root-swagger
        ▼
app/controllers/api/private/swagger.yaml                (combined)
        │  make front/packages/@freee-payroll/api/auto-generated   (openapi-generator)
        ▼
front/packages/@freee-payroll/api/auto-generated/api.ts (functions + types, GENERATED; don't edit)
```

🔎 The swagger for `GET /api/p/employee_display_names/{id}` (Lesson 15) becomes, in `front/packages/@freee-payroll/api/auto-generated/api.ts:7741` (doc comments removed):

```ts
export type ApiPEmployeeDisplayNamesIdGet200Response = {
    'id': number;
    'display_name': string;
}
```

Naming: `ApiP` + path parts + `Id` (the `{id}` parameter) + `Get` + `200Response`. The function is `apiPEmployeeDisplayNamesIdGet(id: number)`.

Import them via the package: `import * as API from '@freee-payroll/api';` then `API.EmployeeMessageCollection[]` (🔎 `front/javascripts/home/components/Home/EmployeeMessagePanel.tsx:9`). Hand-written shared domain types live in `front/packages/freee-types/` (e.g. `HRError`).

**How does `'@freee-payroll/api'` resolve?** Not via `tsconfig` paths (there are none). Yarn **workspaces** (`package.json:167`) link `front/packages/@freee-payroll/*` into `node_modules`, so they import like any installed package.

---

## 8. Runtime validation with zod

TS types vanish at runtime, and `axios.get<T>` is only a *promise*. If the server sends something else, TS can't know. **zod** checks data at runtime *and* produces a TS type from the same definition.

| Term | Meaning |
|---|---|
| **Schema** | A description of valid data, as a value: `z.object({ id: z.number() })`. |
| **`z.infer<typeof Schema>`** | "Give me the TS type this schema describes." |
| **`schema.parse(x)`** | validate; throw if invalid. |
| **`schema.safeParse(x)`** | validate; return `{ success: true, data }` or `{ success: false, error }`. |

🔎 `front/javascripts/migration_projects/types.ts:117-121`

```ts
export const ColumnRefSchema = z.object({
  column_name: z.string(),
  type: z.literal('column'),
});
export type ColumnRef = z.infer<typeof ColumnRefSchema>;
```

`ColumnRef` is automatically `{ column_name: string; type: 'column' }`. One source of truth for both runtime and compile time.

🔎 A discriminated union and an enum in zod (`types.ts:134-151`):

```ts
export const OperandSchema = z.discriminatedUnion('type', [
  ColumnRefSchema,
  LiteralValueSchema,
  CurrentValueSchema,
]);
export type Operand = z.infer<typeof OperandSchema>;

export const ComparisonOperatorSchema = z.enum([
  'eq',
  'neq',
  'lt',
  'lte',
  'gt',
  'gte',
  'matches',
]);
```

🔎 Validating an API response, `front/javascripts/migration_projects/components/MassUpdate/hooks/useFetchMassUpdateMigrationProjects.ts:26-31`:

```ts
  const parsed = z
    .object({
      migration_project: MassUpdateMigrationProjectSchema,
    })
    .optional()
    .safeParse(data);
```

Then `parsed.success ? parsed.data... : undefined`. zod is used in ~16 files (mostly `migration_projects`, `healthcare`, `payroll_verifications`), where data shapes are complex or come from AI/user input.

---

## 9. Practice (15 min)

Use the Playground. Predict first.

**P1. Generics.** What is the type of `first`?
```ts
const head = <T>(xs: T[]): T | undefined => xs[0];
const first = head(['a', 'b']);
```

**P2. Utility types.** Write the resulting types in plain words:
```ts
type User = { id: number; name: string; email: string };
type A = Pick<User, 'id' | 'name'>;
type B = Omit<User, 'email'>;
type C = Partial<User>;
type D = Record<'ja' | 'en', string>;
```

**P3. Narrowing.** Which line errors?
```ts
type State = { kind: 'loading' } | { kind: 'ok'; rows: number[] };
const show = (s: State) => {
  if (s.kind === 'ok') {
    return s.rows.length;
  }
  return s.rows.length;
};
```

**P4. Type guard.** Complete:
```ts
type HRError = { message: string };
const isHRError = (x: unknown): x is HRError =>
  typeof x === 'object' && x !== null && 'message' in x && typeof (x as HRError).message === 'string';
```
Then write `const msg = (e: unknown) => isHRError(e) ? e.message : 'unknown';`. Why is `e.message` OK there?

**P5. Real code.** Open `front/@types/feature-flags.d.ts` and `app/helpers/js_exports_helper.rb` (method `feature_flags`). Pick one flag and find it in both files.

*(Answers: P1 `string | undefined`. P2 A = `{ id: number; name: string }`; B = same as A (all but email); C = every field optional; D = `{ ja: string; en: string }`. P3 the last `return s.rows.length;`: there `s` is narrowed to `{ kind: 'loading' }`, which has no `rows`. P4 because inside the true branch of `isHRError(e)`, TS narrows `e` from `unknown` to `HRError`. P5 e.g. `hrim_position_rank_enabled`, which appears in both.)*

---

## 10. Summary

- **Generics** `<T>` = types with blanks: `Array<T>`, `Promise<T>`, `Record<K, V>`, your own `UseFetchState<T>`, generic functions `useFetch<T>`, constraints `T extends X`.
- `typeof value` (type), `keyof T`, `T['field']` build types from values and other types.
- **Utility types**: `Partial`, `Pick`, `Omit`, `Record`, `ReturnType`, `Awaited`. `Record<Union, string>` makes TS force every case to have a value.
- **Narrowing**: `typeof`, `instanceof`, `in`, null checks, **discriminated unions** (`kind`/`type` field), and **type guards** (`x is T`, e.g. `isHRError`).
- `as` and `!` can lie (the repo needs a reasoned lint disable for `!`); `as const` and `satisfies` are safe. `enum` exists but unions are preferred. `noUncheckedIndexedAccess` makes you write `xs[i]?.x`.
- **`.d.ts`** files declare types for things TS can't see: `$FREEE_DATA` (from Rails) is declared in `front/@types/freee-data.d.ts`; flags in `feature-flags.d.ts`, kept in sync with `js_exports_helper.rb` by hand.
- API types are **generated** from swagger into `@freee-payroll/api/auto-generated` (don't edit). Packages resolve through **Yarn workspaces**.
- **zod** validates at runtime and gives the TS type via `z.infer`.

➡️ Now take **Test25.md**.
