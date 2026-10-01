# Lesson 30 — React in freee-payroll 2: Data, Routing, Forms & Legacy

**Time:** ~1 hour (40 min reading + practice, 20 min test)
**Goal:** Read and write the code that (1) talks to the Rails API, (2) switches screens with react-router, (3) reads `$FREEE_DATA`, (4) handles forms — and (5) recognize legacy code so it doesn't scare you.
**You need:** Lesson 29, and Lessons 15–16 (Private API, `$FREEE_DATA`). Open the repo in your editor.

---

## 1. New words for this lesson

| Term | Meaning in plain words | Example |
|---|---|---|
| **HTTP client** | A library that sends HTTP requests from JavaScript. freee-payroll uses **axios**. | `axios.get('/api/p/...')` |
| **Axios instance** | An axios object pre-configured with default headers/options. | `configured-axios` |
| **Generated client** | TypeScript functions *automatically written* by a tool (openapi-generator) from the OpenAPI (swagger) files. You don't edit them. | `apiPEmployeeDisplayNamesIdGet` |
| **Header** | Extra key/value info sent with a request (not in the URL or body). | `X-Company-Id: 123` |
| **Cookie** | A small piece of data the browser stores and sends back to the same site automatically. | `XSRF-TOKEN` |
| **CSRF** | *Cross-Site Request Forgery*: an attack where an evil site tricks your logged-in browser into sending a request to freee. A **CSRF token** is a secret only freee's real pages know; the server rejects requests without it. | `X-CSRF-TOKEN` header |
| **Client-side routing** | Changing which component is shown **without** asking the server for a new page. | react-router |
| **Hash history** | Routing that uses the part of the URL after `#`. Changing it never reloads the page. | `/hr_portal#/attrition_risk` |
| **Path parameter** | A variable part of a route path. | `/actions/:id` |
| **Query string** | `?key=value` after the path. | `?tab=welfare_pension_insurance` |
| **Global state / store** | Data kept in one place that many components can read (Redux). | `useTypedSelector` |
| **Duck** | A Redux file that contains the action types, action creators **and** the reducer for one piece of data together. | `@freee-payroll/ducks/src/globalModal.ts` |
| **Schema validation** | Checking at runtime that data has the shape you expect. | zod `safeParse` |
| **Legacy** | Old code still running in older technology. | Backbone, CoffeeScript, Flux |

---

## 2. Calling the API

### 2.1 The `@freee-payroll/api` package

`front/packages/@freee-payroll/api/index.ts` contains **two kinds** of API code:

1. **Hand-written wrappers** in `src/` that use `configured-axios` — exported as namespaces, e.g. `export * as AuthCompany from './src/AuthCompany';` (index.ts around line 49), and helpers like `getErrorMessage` from `./src/errorHandler`.
2. **The generated client**: `front/packages/@freee-payroll/api/index.ts:85`

```ts
export * from './auto-generated';
```

`front/README.md` rule: **use `@freee-payroll/api` (the generated client) for HTTP calls.** Only when an endpoint has **no OpenAPI definition**, use `configured-axios` directly.

### 2.2 How a generated function is born (full stack)

You saw this pipeline in Lesson 15. Here's the frontend end of it:

```
app/controllers/api/private/employee_display_names_controller.swagger.yaml
   paths: '/api/p/employee_display_names/{id}':  get:
        │
        │  make generate-root-swagger                          (merge all *.swagger.yaml)
        │  make front/packages/@freee-payroll/api/auto-generated  (run openapi-generator)
        ▼
front/packages/@freee-payroll/api/auto-generated/api.ts
   apiPEmployeeDisplayNamesIdGet(id: number, options?)   ← generated, typed
        │
        ▼
your component:  await API.apiPEmployeeDisplayNamesIdGet(employeeId)
```

**Naming rule:** path + HTTP method, in camelCase.

| Endpoint | Generated function |
|---|---|
| `GET /api/p/employees` | `apiPEmployeesGet` |
| `GET /api/p/employee_display_names/{id}` | `apiPEmployeeDisplayNamesIdGet` |
| `GET /api/p/company_health_insurance_settings/{year}/{month}` | `apiPCompanyHealthInsuranceSettingsYearMonthGet` |
| `POST /api/p/comments` | `apiPCommentsPost` |

The response type is also generated, so `data.display_name` is type-checked against the swagger file. If the Rails team changes the schema and you regenerate, TypeScript tells you everything that broke. 🎉

**Real call site** — `front/javascripts/DEPRECATED_docs/components/Contracts/EditContract.tsx:795-798`:

```tsx
  const loadEmployee = React.useCallback(async (employeeId: number) => {
    const { data } = await API.apiPEmployeeDisplayNamesIdGet(employeeId);
    setEmployee(data);
  }, []);
```

- The function returns a Promise of an axios response; `{ data }` destructures the JSON body.
- (Import at the top of that file, line 21: `import * as API from '@freee-payroll/api';` — the repo's usual style.)

### 2.3 `configured-axios`: headers and CSRF

`front/packages/configured-axios/index.ts:1-26`:

```ts
import axios from 'axios';

const headers = {
  ...($FREEE_DATA.loginUser?.company_id != null
    ? {
        'X-Company-Id': String($FREEE_DATA.loginUser.company_id),
      }
    : {}),
  ...($FREEE_DATA.loginUser?.user_id != null
    ? {
        'X-User-Id': String($FREEE_DATA.loginUser.user_id),
      }
    : {}),
};
export const axiosOptions = {
  headers,
  xsrfCookieName: 'XSRF-TOKEN',
  xsrfHeaderName: 'X-CSRF-TOKEN',
};

// cookie から取得した CSRFトークンをデフォルトで header に添付する axios インスタンスを作る
export default axios.create(axiosOptions);

axios.defaults.headers.common = headers;
axios.defaults.xsrfCookieName = 'XSRF-TOKEN';
axios.defaults.xsrfHeaderName = 'X-CSRF-TOKEN';
```

Line by line:
- `...(cond ? { key: value } : {})` — **conditional spread**: add the header only if the value exists.
- `X-Company-Id` / `X-User-Id`: tell the server which company/user the page was loaded for. The server checks they match the session (leak check — Lesson 15's `CompanyIdLeakError`).
- CSRF: Rails sets a cookie `XSRF-TOKEN` (`after_action :set_csrf_cookie` in `ApplicationController`). axios reads it and sends it back as header `X-CSRF-TOKEN`. An evil site can't read freee's cookies, so it can't send that header → its forged request is rejected.
- The last three lines set **global axios defaults**, so the generated client (which uses the global axios) gets the same headers.

Hand-written wrapper example — `front/packages/@freee-payroll/api/src/TaxOffice.ts:1-12`:

```ts
import axios from 'configured-axios';
import { TaxOffice } from 'freee-types';

/**
 * 税務署一覧を取得する API
 */
type IndexResponse = { models: Array<TaxOffice> };

export const index = async (pref: string) => {
  const { data } = await axios.get<IndexResponse>(`/api/p/tax_offices/${pref}`);
  return data.models;
};
```

`axios.get<IndexResponse>` uses a **generic** (Lesson 25) to say what JSON comes back.

### 2.4 Loading and errors — the standard recipe

`front/javascripts/company_social_insurance_settings/hooks/useKyokaiKenpoRates.ts:45-90` (verified; trimmed a little inside `setRates`):

```ts
export const useKyokaiKenpoRates = (options: Options) => {
  const { month, year } = options;

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
        if (!isCancelled) {
          setRates( /* convert data.kyokai_kenpo_health_insurance_rates */ );
        }
      } catch (e) {
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
};
```

The pieces, all from shared packages:

| Piece | From | What it does |
|---|---|---|
| `useLoading()` | `@freee-payroll/hooks` | Counter-based loading flag: `showLoading()` +1, `hideLoading()` −1, `isLoading = count > 0` (so parallel requests work) — `front/packages/@freee-payroll/hooks/src/useLoading.ts` |
| `useNotification()` | `@freee-payroll/components` | Returns `notify({ message, type })` to show a toast. Works because `run()` wraps the app in `NotificationProvider`. |
| `API.getErrorMessage(e)` | `@freee-payroll/api` | Reads the server's error message (or a default "エラーが発生しました…") |
| `isCancelled` + cleanup | plain React | If the component unmounts (or year/month change) before the response arrives, don't set state on stale data. |
| `try / catch / finally` | JS | `finally` runs whether success or error → loading always ends. |

### 2.5 Error shapes: `getErrorMessage` and `isHRError`

Rails' `ErrorHandleable` returns `{ message, code, fields }` (Lesson 15). The frontend type for that is in `front/packages/freee-types/HRError.ts`:

```ts
export type HRError = {
  code?: string | null;
  fields?: Array<HRErrorField>;
  message: string;
};
```

`front/packages/@freee-payroll/api/src/errorHandler.ts:53-63`:

```ts
export const getErrorMessage = (e: any) => {
  const defaultMessage =
    'エラーが発生しました。 再読み込みをしても解消しない場合は、サポートセンターまでご連絡ください。';
  if (isHRError(e)) {
    return e.response?.data.message ?? defaultMessage;
  }
  if (isLegacyHRError(e)) {
    return e.response?.data.messages[0] ?? defaultMessage;
  }
  return defaultMessage;
};
```

- `isHRError` (same file, line 12) is a **type guard**: `(e: any): e is AxiosError<HRError>`. After `if (API.isHRError(e))`, TypeScript knows `e.response?.data.fields` exists.
- `isLegacyHRError` handles the **old** format `{ messages: [...] }` from controllers that don't include `ErrorHandleable`.
- `fields` carries per-field errors (e.g. "this rate must be 0–99.99999"), which forms show next to inputs (see `fieldErrors` prop in `RateInputRow.tsx`).

---

## 3. Where should state live?

Rule from `front/README.md`:

```
Only this component needs it?          → local state  (useState / useReducer)
Several screens in the SAME feature?   → React Context (createContext / useContext)
Several DIFFERENT features share it?   → Redux, ducks pattern (@freee-payroll/ducks)
```

Examples in the repo:
- Local: almost everything (`useState` appears in ~1,400 files).
- Context: `front/javascripts/shift/components/ShiftSavingContext.tsx` (a Provider + `useShiftSaving` hook).
- Redux ducks: `front/packages/@freee-payroll/ducks/src/globalModal.ts` — hand-written action creators + reducer:

```ts
const CLOSE = 'GLOBAL_MODAL/CLOSE' as const;
const OPEN = 'GLOBAL_MODAL/OPEN' as const;
...
export default (state: State = null, action: Action): State => {
  ...
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

Read it with `useTypedSelector` (`@freee-payroll/hooks`), e.g. `front/packages/@freee-payroll/components/src/GlobalModalContainer.ts`: `useTypedSelector((s) => s.globalModal)`. Change it with `useDispatch()` from `react-redux`.

> Note: Redux Toolkit's `createSlice` appears in only one (DEPRECATED) file. Ducks here are hand-written "FSA" (Flux Standard Action: `{ type, payload, error? }`) objects — the global `FSA` type is in `front/@types/utils.d.ts`.

---

## 4. Routing with react-router **v3**

freee-payroll uses **react-router 3.0.5** (old API — tutorials online for v6 look different!).

### 4.1 Two levels of routing

```
Rails decides the PAGE (bundle)           React decides the SCREEN inside it
/hr_portal         ──► hr_portal bundle ──► #/                  → HomePage
                                         ──► #/attrition_risk   → AttritionRiskPageContainer
                                         ──► #/actions/42       → ActionDetailPageContainer (id = "42")
```

freee-payroll uses **`hashHistory`**: the in-app path lives after `#`. The browser **never sends the part after `#` to the server**, so Rails only ever sees `/hr_portal` and always returns the same `app.html.erb`. That's why most Rails web routes are one simple `get`. Some features use browser-style paths instead, and Rails then needs a catch-all like `config/routes/web.rb:44`:

```ruby
get 'attendances(/*path)', to: 'web/attendances#index'
```

(`(/*path)` = "and anything after it" → same page.)

### 4.2 The pieces (verified from repo files)

| Piece | Meaning | Where seen |
|---|---|---|
| `<Router history={hashHistory}>` | Root; use hash URLs | `hr_portal/Routes.tsx:30` |
| `onUpdate={fn}` | Called on every route change | `hr_portal/Routes.tsx:30`, `holidays/Routes.tsx:39` |
| `<Route path="/" component={...}>` with children | Layout route; children render inside it | `hr_portal/Routes.tsx:31-41` |
| `<IndexRoute component={HomePage} />` | What to show at exactly `/` | `hr_portal/Routes.tsx:37` |
| `path="/actions/:id"` | Path param `id` (available as `props.params.id`) | `hr_portal/Routes.tsx:40` |
| `<IndexRedirect to=... />` | At `/`, redirect elsewhere | `holidays/Routes.tsx:46` |
| `<Redirect from="old" to="new" />` | Old URL → new URL | `DEPRECATED_settings/Routes.tsx:828` |
| `path="*"` | Anything else (e.g. not-found page) | `hr_portal/Routes.tsx:24` |
| `props.location.query.tab` | Read `?tab=...` | `DEPRECATED_settings/Routes.tsx:843` |
| `hashHistory.push('/x?tab=y')` | Navigate from code | `company_social_insurance_settings/components/InsuranceSettingsIndex.tsx:82` |

A real, rich example — `front/javascripts/DEPRECATED_settings/Routes.tsx:825-847`:

```tsx
        {$FREEE_DATA.ff.multiple_social_ins_2nd_enabled ? (
          <>
            {/* 新画面は年月を持たないため、旧 URL は年月を捨てて一覧へ寄せる */}
            <Redirect
              from="health_insurance"
              to="company_social_insurance_settings"
            />
            <Redirect
              from="health_insurance/:year/:month"
              to="company_social_insurance_settings"
            />
            {hasPermission({ functionKeys: ['settings:payroll:write'] }) ? (
              <>
                <Route
                  component={(props: {
                    location: { query: { tab?: string } };
                  }) => (
                    <InsuranceSettingsIndexContainer
                      tab={props.location.query.tab}
                    />
                  )}
                  path="company_social_insurance_settings"
                />
```

Read it:
1. A **feature flag** chooses between the new screens and the old ones (ternary in JSX).
2. Old URLs are **redirected** to the new list page.
3. **`hasPermission`** (from `@freee-payroll/utils`) checks the user's permission keys (the same function keys as `config/permission.yaml`) — without permission, other routes show `ErrorPage errorType="forbidden"`.
4. The route reads `?tab=` from `location.query` and passes it as a prop to the Container.

> This is a `DEPRECATED_` folder, yet it hosts the newest screens' routes. Remember: `DEPRECATED_` ≠ "don't touch".

---

## 5. `$FREEE_DATA`: the global from Rails

Recap (Lesson 16): Rails renders `window.$FREEE_DATA = {...}` into every page. Its TypeScript type is declared globally in `front/@types/freee-data.d.ts` (`declare global { var $FREEE_DATA: { ... } }`), and flag names in `front/@types/feature-flags.d.ts` (`export type FeatureFlags = { ... }`), wired as `ff: FeatureFlags`.

Common fields:

| Field | Meaning |
|---|---|
| `$FREEE_DATA.ff.xxx` | Feature flags (release flags) |
| `$FREEE_DATA.plan_features.xxx` | Can this company's plan use feature xxx? (e.g. `holidays/Routes.tsx:21`) |
| `$FREEE_DATA.loginUser` | Current company/user ids (used by configured-axios) |
| `$FREEE_DATA.userInfo.user` | Info about the logged-in user (e.g. `employee_id`) |
| `$FREEE_DATA.applied_plans` | Plan/contract details |

### 5.1 The rules (from `front/README.md`)

1. **Reference it directly.** Global variables are usually bad, but here it's intentional "core knowledge".
2. **Don't copy a flag into a local variable** like `const fooEnable = $FREEE_DATA.ff.foo_enabled`. Seeing `$FREEE_DATA.ff.` at the usage site tells the reader instantly "this is a feature flag"; a local variable makes them hunt for where it came from.
3. **Exception:** to keep a pure function independent, or in Storybook, pass the value **as a prop/argument**. **Never overwrite `$FREEE_DATA` inside a Storybook story** — it would leak into other stories.

Good (verified) — `front/javascripts/home/components/HomeForOthers.tsx:107-108`:

```tsx
      {!$FREEE_DATA.ff.remove_healthcare_task_cards &&
        !$FREEE_DATA.ff.healthcare_sc_emp_view_enabled &&
```

⚠️ **Real code isn't always perfect.** `front/javascripts/hr_portal/Routes.tsx:19` does exactly what the rule discourages:

```tsx
  const isHrPortalEnabled = $FREEE_DATA.ff.hr_portal_enabled;
```

It still works — the rule is about readability. When *you* write new code, follow the rule; when reading, don't be confused by exceptions.

### 5.2 Permission and plan helpers

- `hasPermission({ functionKeys: ['employee:*:*'] })` — from `@freee-payroll/utils` (implemented in `src/policyDecisionPoint`). Also `useHasPermission` hook.
- `usePlanFeatureUsable()` — from `@freee-payroll/hooks`; returns `isUsable('joining_and_resigning')` style checker (see `home/components/Home/JoiningEmployeesPanel.tsx:30-34` in Lesson 29).

These only **hide UI**. The **server** still enforces permissions (`permission.yaml`, `has_allowed_ac?`). Never rely on the frontend for security.

---

## 6. Forms

### 6.1 Controlled inputs (most common)

From Lesson 29: `value={displayName}` + `onChange={(e) => setDisplayName(e.currentTarget.value)}` in `DisplayNameInputModal.tsx`. For simple forms, just `useState`.

### 6.2 Formik (135 files; version 1.5.8)

Formik manages form values, submit, and errors for bigger forms. It uses the **render-prop** style: you pass a function as children.

`front/javascripts/home/components/Home/TimeClockModals/EditTimeClockModal.tsx:108-121`:

```tsx
    <Formik
      enableReinitialize
      initialValues={initialValues}
      onSubmit={handleSubmit}
    >
      {({ handleSubmit, values }) => {
        return (
          <TaskDialog
            closeButtonLabel="キャンセル"
            isOpen
            onPrimaryAction={() => handleSubmit()}
            onRequestClose={onRequestClose}
            primaryButtonLabel="保存"
            title="打刻修正"
          >
```

- `initialValues` = starting form values (`{ time_clocks: currentTimeClocks }`, line 85).
- The children function receives Formik helpers (`handleSubmit`, `values`, …).
- `enableReinitialize`: if `initialValues` change, reset the form.

A reusable field — `front/javascripts/approval_requests/components/ApprovalRequestFields/DateField.tsx:12-37`:

```tsx
export const DateField = ({ id, name, onChange, required }: DateFieldProps) => {
  return (
    <Field name={name}>
      {({ field, form }: FieldProps) => (
        <VStack gap={0.5}>
          <DateInput
            error={!!form.errors[name]}
            id={id}
            mr={1}
            onChange={(date) => {
              form.setFieldValue(field.name, date);
              onChange && onChange(date);
            }}
            required={required}
            value={field.value}
          />
          {typeof form.errors[name] === 'string' && (
            <Message error ml={0.25}>
              {form.errors[name]}
            </Message>
          )}
        </VStack>
      )}
    </Field>
  );
};
```

- `<Field name="...">` connects to one value in Formik's state. `field.value` = current value; `form.setFieldValue(name, v)` = change it; `form.errors[name]` = its error.
- `typeof form.errors[name] === 'string' && (...)` — **narrowing** + conditional rendering: show the message only if there's a string error.
- `onChange && onChange(date)` — call the optional callback only if it was given.

### 6.3 zod: checking data at runtime

TypeScript types disappear at runtime; if the server sends unexpected JSON, TS can't catch it. **zod** (16 files) checks shape at runtime.

`front/javascripts/migration_projects/components/MassUpdate/hooks/useFetchMassUpdateMigrationProjects.ts:26-31`:

```ts
  const parsed = z
    .object({
      migration_project: MassUpdateMigrationProjectSchema,
    })
    .optional()
    .safeParse(data);
```

and later (`:48-50`):

```ts
    data: parsed.success
      ? (parsed.data?.migration_project ?? undefined)
      : undefined,
```

- `z.object({...})` describes the expected shape; `.safeParse(data)` returns `{ success: true, data }` or `{ success: false, error }` (never throws).
- Types can be derived from schemas: `export type ColumnRef = z.infer<typeof ColumnRefSchema>;` (`front/javascripts/migration_projects/types.ts`).

---

## 7. Recognizing legacy code (don't copy, don't panic)

| You see… | It is… | Status |
|---|---|---|
| `.coffee` files, `->`, `@foo` | **CoffeeScript** (old JS dialect) + **Backbone.js** | In `front/packages/@legacy/` (62 `.coffee` files). Old screens. |
| `run({ withBackboneRouter: ... })` | Legacy Backbone app boot | `front/javascripts/legacy_application/index.ts` |
| `front/javascripts/flux_infra/` | **Flux** data flow (older than Redux) | README: maintenance mode, "新規実装は追加しないでください" |
| `DEPRECATED_<feature>/` | Feature that uses Flux | OK to edit when needed |
| `front/javascripts/react_components/` | Old shared components | Maintenance mode; new generic ones → `@freee-payroll/components` |
| `$(...)` jQuery | Old DOM library | Still used in `run.tsx` for "page ready" |

**CoffeeScript Backbone model** — `front/packages/@legacy/app/models/employee_yearend_adjustment_evidence_files.coffee:1-19`:

```coffee
PayrollModel = require('../payroll/payroll_model')

module.exports = class EmployeeYearendAdjustmentEvidenceFiles extends PayrollModel

  employeeID: null
  id: null

  # @override
  initialize: (data, options = {}) ->
    super arguments...
    @employeeID = options.employeeID
    @id = options.id
    @year = options.year

  url: ->
    if @id
      "/api/p/employees/#{@employeeID}/yearend_adjustments/#{@year}/evidence_files/#{@id}"
    else
      "/api/p/employees/#{@employeeID}/yearend_adjustments/#{@year}/evidence_files"
```

Decode: `->` = function, `@id` = `this.id`, `"#{x}"` = string interpolation (like Ruby!), indentation instead of `{}`. It's still calling the same `/api/p/...` Private API.

**Legacy boot** — `front/javascripts/legacy_application/index.ts:1-7`:

```ts
import ApplicationOldRouter from '@legacy/app/routers';

import run from '../initializers/run';

run({
  withBackboneRouter: ApplicationOldRouter,
});
```

**Flux still used by new routes** — `hr_portal/Routes.tsx:5` imports `NavigationContextAction` from `../flux_infra/actions/NavigationContextAction`, and calls it in `onUpdateRoute` (lines 12-16) to tell the global navigation which menu is active. `flux_infra/actions/NavigationContextAction.ts:1` starts with `import { dispatch } from '../dispatcher/app_dispatcher';`. It's legal to import because `flux_infra` is a non-restricted dir.

---

## 8. 🔎 Practice (15 min)

**Practice 1: name the function.** What is the generated function name for:
a) `GET /api/p/comments`  b) `DELETE /api/p/comments/{id}`  c) `PUT /api/p/company_daily_prorate_setting`
*(a `apiPCommentsGet`, b `apiPCommentsIdDelete`, c `apiPCompanyDailyProrateSettingPut` — check by searching `auto-generated/api.ts` for the name!)*

**Practice 2: grep it.** Run
`grep -n "export const apiPEmployeeDisplayNamesIdGet" front/packages/@freee-payroll/api/auto-generated/api.ts`
and look at its parameter types.

**Practice 3: read the URL.** For `https://p.secure.freee.co.jp/settings#/company_social_insurance_settings?tab=welfare_pension_insurance`:
- What does Rails see? (`/settings`)
- Which route in `DEPRECATED_settings/Routes.tsx` matches? (`company_social_insurance_settings`)
- What is `props.location.query.tab`? (`'welfare_pension_insurance'`)

**Practice 4: where does the state go?**
a) Whether a dropdown is open → ? b) A "saving…" flag shared by several components in the shift feature → ? c) Company allowance settings used by payroll and attendance features → ?
*(a local state, b Context — see `ShiftSavingContext.tsx`, c Redux ducks — `@freee-payroll/ducks/src/companyAllowances.ts`)*

**Practice 5: decode CoffeeScript.** What does `@employeeID = options.employeeID` mean in TypeScript? *(`this.employeeID = options.employeeID;`)*

---

## 9. Summary

- Call APIs with **`@freee-payroll/api`** (generated from swagger: name = path + method). Use **`configured-axios`** only for endpoints without OpenAPI.
- configured-axios adds `X-Company-Id` / `X-User-Id` and the **CSRF** token (`XSRF-TOKEN` cookie → `X-CSRF-TOKEN` header).
- Standard recipe: `useLoading` + `try/catch/finally` + `notify({ message: API.getErrorMessage(e), type: 'error' })` + cleanup flag.
- State: **local → Context → Redux ducks**.
- **react-router v3 + `hashHistory`**: `Router`, `Route`, `IndexRoute`, `Redirect`, `:id`, `location.query`, `hashHistory.push`. The `#` part never reaches Rails.
- `$FREEE_DATA`: read **directly** (`$FREEE_DATA.ff.x`), don't alias; pass as props in Storybook; never mutate in stories. Frontend checks (`hasPermission`) only hide UI.
- Forms: controlled inputs, **Formik** (`<Formik>`, `<Field>` render props), **zod** `safeParse` for runtime checks.
- Legacy: CoffeeScript/Backbone (`@legacy`), Flux (`flux_infra`), `DEPRECATED_*`, `react_components/` — read them, don't extend them (except DEPRECATED_ when needed).

➡️ Now take **Test30.md**.
