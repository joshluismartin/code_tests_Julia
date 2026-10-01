# Lesson 29 — React in freee-payroll 1: Structure & UI

**Time:** ~1 hour (40 min reading + practice, 20 min test)
**Goal:** Know where every frontend file lives, how one screen's code gets from `front/` into the browser, and recognize the UI building blocks (Vibes) used on almost every screen.
**You need:** Lessons 22–28 (JavaScript, TypeScript, React). Open `~/Josh_Files/freee-payroll/front/` in your editor.

---

## 1. New words for this lesson

| Term | Meaning in plain words | Example in the repo |
|---|---|---|
| **Feature directory** | One folder under `front/javascripts/` that holds all the code for one area of the product (one "menu item"). | `front/javascripts/holidays/`, `front/javascripts/hr_portal/` |
| **Entrypoint** | The first file the bundler starts from when it builds one JavaScript file for the browser. Everything the screen needs is reached by following `import`s from here. | `front/javascripts/hr_portal/index.ts` |
| **Bundler** | A tool that reads your many `.ts/.tsx` files and joins ("bundles") them into a few `.js` files the browser can load. freee-payroll uses **Vite**. | `vite.config.mts` |
| **Package (internal)** | A shared folder of code with its own `package.json`, which you can import by name like a library. | `@freee-payroll/api`, `@freee-payroll/components` |
| **Yarn workspaces** | A Yarn feature that lets one repo contain many internal packages and import them by name. | `package.json` → `"workspaces"` |
| **Design system** | A company-wide set of ready-made UI components (buttons, inputs, dialogs) so every screen looks the same. freee's is **Vibes** (`@c-fo/vibes`). | `import { TextField } from '@c-fo/vibes';` |
| **Maintenance mode** | "Keep it working, but don't add new things here." | `front/javascripts/react_components/` |
| **Legacy** | Old code still running, written in an older style. | `DEPRECATED_*` folders, CoffeeScript |
| **Accessibility (a11y)** | Making screens usable by everyone, including people using screen readers. "a11y" = a + 11 letters + y. | `FormControl fieldId` rule |
| **Lint rule** | An automatic check (ESLint) that fails CI if code breaks a team rule. | "no cross-feature imports" |
| **i18n** | Internationalization: showing the UI in several languages. "i" + 18 letters + "n". | `useTranslation()` |

---

## 2. The shape of `front/`

Run `ls ~/Josh_Files/freee-payroll/front` and you'll see (verified):

```
front/
├── @types/          ← global TypeScript types (e.g. the type of $FREEE_DATA)
├── CLAUDE.md, README.md   ← frontend rules (READ README.md!)
├── images/, sounds/
├── javascripts/     ← one folder per feature (~60 folders)
├── jsEntries.js     ← list of entrypoints (one per feature bundle)
├── packages/        ← internal shared packages (Yarn workspaces)
└── stylesheets/     ← Sass (.scss) styles
```

| Folder | What goes there | You'll touch it when… |
|---|---|---|
| `front/javascripts/<feature>/` | Screens, components, hooks, types for **one** feature | Building/changing a screen |
| `front/packages/@freee-payroll/*` | Code shared **between** features: `api`, `components`, `constants`, `ducks`, `hooks`, `i18n`, `utils` | Something is needed in 2+ features |
| `front/packages/@legacy/`, `flux`, … | Old CoffeeScript/Backbone and Flux code | Reading old screens |
| `front/@types/` | `.d.ts` global type files (`freee-data.d.ts`, `feature-flags.d.ts`, `utils.d.ts`) | Adding a new `$FREEE_DATA` field or flag |
| `front/stylesheets/root/*.scss` | CSS entrypoints (one per feature) | Styling that Vibes can't do |

> Where are `tsconfig.json`, `vite.config.mts`, `vitest.config.mts`, `.eslintrc.js`? In the **repo root** (`freee-payroll/`), not in `front/`.

### 2.1 Why can I write `import ... from '@freee-payroll/api'`?

Because of **Yarn workspaces**. `package.json:167-170` (repo root):

```json
  "workspaces": [
    "./front/packages/@freee-payroll/*",
    "./front/packages/@legacy/*",
    "./front/packages/chatbot-widget",
```

Every folder in that list has its own `package.json` with a `name`. For example `front/packages/@freee-payroll/api/package.json` says `"name": "@freee-payroll/api"`, `"main": "index.ts"`. Yarn links them so an `import` by name works like importing any npm library.

---

## 3. From a URL to a React screen (the full path)

This is the most important picture of the lesson. Let's follow **`/hr_portal`** (verified).

```
Browser asks:   GET /hr_portal
   │
   ▼
config/routes/web.rb:64            get 'hr_portal', to: 'web/hr_portal#index'
   │
   ▼
Web::HrPortalController#index      renders app/views/web/app.html.erb  →  <div id="app"></div>
   │
   ▼
app/helpers/application_helper.rb:29
   'web/hr_portal' => 'front/javascripts/hr_portal/index.ts'     (JS_FILE_MAP)
   │  → Rails puts a <script> tag for that bundle on the page
   ▼
front/javascripts/hr_portal/index.ts      run({ withReactRouter: Routes })
   │
   ▼
front/javascripts/initializers/run.tsx    createRoot(#app).render(<Providers><Routes/></Providers>)
   │
   ▼
front/javascripts/hr_portal/Routes.tsx    picks the page component for the part after "#"
```

### 3.1 `jsEntries.js`: the list of bundles

`front/jsEntries.js:1-10` (57 lines total):

```js
// ビルド対象の起点となるエントリポイント定義
// MEMO: @keik vite.config.ts に移行

// prettier-ignore
module.exports = {
  'renew-settings': 'front/javascripts/DEPRECATED_settings/index.ts',
  admin2: 'front/javascripts/admin2/index.tsx',
  admin: 'front/javascripts/admin/index.ts',
  advisor_support_requests: 'front/javascripts/DEPRECATED_advisor_support_requests/index.ts',
  application: 'front/javascripts/legacy_application/index.ts',
```

and further down: `front/jsEntries.js:31` `holidays: 'front/javascripts/holidays/index.ts',` and `:33` `hr_portal: 'front/javascripts/hr_portal/index.ts',`.

Vite reads this list: `vite.config.mts:10` `import jsEntries from './front/jsEntries';` and `:26` `...jsEntries,` inside `input`.

> Rule from `docs/knowledge/general/architecture/overview-frontend.md`: **1 feature = 1 entrypoint.** A new feature folder usually means one new line in `jsEntries.js`, one line in `JS_FILE_MAP`, and (if needed) one CSS entry `front/stylesheets/root/<feature>.scss` in `CSS_FILE_MAP` (e.g. `app/helpers/application_helper.rb:66` `'web/holidays' => 'front/stylesheets/root/holidays.scss'`).

If a page is in **no** map, `js_file_path` falls back to `front/javascripts/legacy_application/index.ts` (the old Backbone app) — you learned this in Lesson 16.

### 3.2 The feature entry file

`front/javascripts/hr_portal/index.ts:1-5`:

```ts
import run from '../initializers/run';

import Routes from './Routes';

run({ withReactRouter: Routes });
```

Almost every feature's `index.ts` looks exactly like this (`holidays/index.ts` is identical). `run` is a shared bootstrapper.

### 3.3 `run.tsx`: where React is mounted

`front/javascripts/initializers/run.tsx:22-35` (start):

```tsx
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
      const Router = initContext.withBackboneRouter;
```

Read it with your TypeScript eyes:
- `InitContext` is a **union type**: you pass *either* a Backbone router (old apps) *or* a React router component.
- `'withBackboneRouter' in initContext` is **narrowing with `in`** (Lesson 25).
- `$(() => { ... })` is jQuery for "run when the page has loaded" (legacy, but it's there).

In the React branch (`run.tsx:42-66`) it finds `#app` and renders:

```tsx
createRoot(el!).render(
  <NotificationProvider>
    <Provider store={store}>
      ...
          <GlobalModalContainer />
          <Routes />
      ...
    </Provider>
  </NotificationProvider>
);
```

So every normal feature automatically gets:
- `NotificationProvider` → you can call `useNotification()` anywhere to show a toast message.
- Redux `Provider` with the shared `store` → `useTypedSelector` / `useDispatch` work.
- `GlobalModalContainer` → a place where "global" modals appear.

### 3.4 The exception: mounting directly

`front/javascripts/admin2/index.tsx:1-12`:

```tsx
import { NotificationProvider } from '@freee-payroll/components';
import * as React from 'react';
import { createRoot } from 'react-dom/client';

import Routes from './Routes';

// eslint-disable-next-line @typescript-eslint/no-non-null-assertion -- appは必ず存在する
createRoot(document.querySelector('#app')!).render(
  <NotificationProvider>
    <Routes />
  </NotificationProvider>
);
```

The internal admin tool skips `run` and mounts React itself (no Redux store). Notice the `!` non-null assertion: TypeScript thinks `querySelector` might return `null`; the comment explains why it's safe.

---

## 4. Inside a feature directory

### 4.1 `hr_portal` (a newer feature) — verified tree (partial)

```
front/javascripts/hr_portal/
├── index.ts          ← entrypoint (3 lines)
├── Routes.tsx        ← which component for which "#/..." path
├── types.ts          ← shared TS types for this feature
├── constants.ts
├── hooks/
│   ├── useAgentRpc.ts
│   └── useAttritionRiskSummary.ts
└── components/
    ├── HomePage.tsx
    ├── ActionDetailPage.tsx
    ├── ActionDetailPage/      ← sub-components used only by ActionDetailPage
    ├── AttritionRiskPage.tsx
    ├── AttritionRiskPage/
    └── SubActions/
```

`front/javascripts/hr_portal/Routes.tsx:18-44`:

```tsx
const Routes = () => {
  const isHrPortalEnabled = $FREEE_DATA.ff.hr_portal_enabled;

  if (!isHrPortalEnabled) {
    return (
      <Router history={hashHistory}>
        <Route component={() => <ErrorPage errorType="not_found" />} path="*" />
      </Router>
    );
  }

  return (
    <Router history={hashHistory} onUpdate={onUpdateRoute}>
      <Route
        component={(s) => {
          return <App responsive {...s} />;
        }}
        path="/"
      >
        <IndexRoute component={HomePage} />
        <Route component={AttritionRiskPageContainer} path="/attrition_risk" />
        <Route component={ActionMonitorPageContainer} path="/action_monitor" />
        <Route component={ActionDetailPageContainer} path="/actions/:id" />
      </Route>
    </Router>
  );
};
```

What you can already read:
- A **feature flag** (`$FREEE_DATA.ff.hr_portal_enabled`) hides the whole feature (shows "not found") until it's released.
- `App` (from `@freee-payroll/components`) is the page frame (global navigation etc.).
- The page names end in `...Container` — that's the container/presentational split you'll see next. (Routing details: Lesson 30.)

`front/javascripts/hr_portal/types.ts:10-12` shows the preferred TS style — `type` + string-literal unions:

```ts
export type RiskLevel = 'VERY_HIGH' | 'HIGH' | 'POTENTIALLY_HIGH' | 'STABLE';

export type SubActionStatus = 'completed' | 'todo' | 'in_progress';
```

> `.eslintrc.js:146` `'@typescript-eslint/consistent-type-definitions': ['error', 'type']` — the linter forces `type`, not `interface`.

### 4.2 `company_social_insurance_settings` (the modern exemplar)

This is the best folder to copy from: fully typed, hooks, tests and stories next to each file (verified):

```
front/javascripts/company_social_insurance_settings/
├── types.ts
├── snapshots.test.ts           ← turns all stories into snapshot tests (Lesson 31)
├── hooks/
│   ├── useCompanyProfileFetcher.ts   (+ .test.ts)
│   ├── useHealthInsuranceSettings.ts (+ .test.ts)
│   ├── useInsuranceSettingListControl.ts (+ .test.ts)
│   ├── useKyokaiKenpoRates.ts
│   └── useWelfarePensionInsuranceSettings.ts (+ .test.ts)
└── components/
    ├── InsuranceSettingsIndex.tsx (+ .stories.tsx)
    ├── ChangeDefaultSettingModal.tsx (+ .stories.tsx)
    ├── common/ (RateInputRow.tsx, ApplyYearMonthSection.tsx, … + stories/tests)
    ├── HealthInsuranceSetting/
    └── WelfarePensionInsuranceSetting/
```

It has no entry of its own; its containers are routed from `DEPRECATED_settings/Routes.tsx` (you'll see that in Lesson 30).

### 4.3 The "Presentational + Container" pattern

You'll see this pattern in hundreds of files. One file exports **two** components:

| | Presentational component | Container component |
|---|---|---|
| Export | `export default X` | `export const XContainer` |
| Gets data from | **props only** | hooks, API calls, `$FREEE_DATA` |
| Has state? | Maybe UI state only | Yes: loaded data, loading flags |
| Easy to put in Storybook? | ✅ yes (just pass props) | ❌ needs a server |

Real example — `front/javascripts/home/components/Home/JoiningEmployeesPanel.tsx:9-62`:

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

- The **presentational** part only turns `employeeCount` into UI.
- The **container** checks permission + plan, calls the API in `useEffect`, and stores the result with `useState`.
- `(async () => { ... })()` is an **IIFE** (immediately-invoked function): `useEffect` can't take an `async` function directly, so they create one and call it at once.

---

## 5. The golden rule: no imports across features

From `overview-frontend.md`: *importing across feature directories is forbidden*, to avoid one feature accidentally breaking another.

It's **enforced by ESLint** (`.eslintrc.js:3-21`):

```js
const noRestrictedPathsOption = (() => {
  const basePath = './front/javascripts';
  const nonRestrictedDirs = [
    'flux_infra',
    'initializers',
    'react_components',
    'utils',
  ];
  ...
  const zones = restrictedDirs.map((dir) => ({
    target: `./${dir}/**/*`,
    from: `./!(${[dir, ...nonRestrictedDirs].join('|')})/**/*`,
  }));
```

and `.eslintrc.js:351` `'import/no-restricted-paths': ['error', noRestrictedPathsOption],`.

In words: files in feature `X` may import from `X` itself or from the 4 shared folders (`flux_infra`, `initializers`, `react_components`, `utils`) — **not** from feature `Y`.

```ts
// front/javascripts/hr_portal/components/Foo.tsx   (illustrative example)
import { useLoading } from '@freee-payroll/hooks';             // ✅ shared package
import { HomePage } from './HomePage';                          // ✅ same feature
import run from '../initializers/run';                          // ✅ allowed shared dir
import { AutoCalcSummary } from '../holidays/components/AutoCalcSummary'; // ❌ ESLint error
```

**Need something in two features?** Move it to a package: generic components → `@freee-payroll/components`, hooks → `@freee-payroll/hooks`, helpers → `@freee-payroll/utils`, shared Redux data → `@freee-payroll/ducks`.

---

## 6. Rules from `front/README.md` you must know

(Read the real file: `front/README.md`. Summary in English:)

1. **`$FREEE_DATA`** is a global object filled by Rails. Read it **directly** (details Lesson 30).
2. **Snapshot tests** are generated automatically from Storybook stories — don't write individual snapshots (Lesson 31).
3. **HTTP calls** → use `@freee-payroll/api` (generated from OpenAPI). If no OpenAPI definition exists, use `configured-axios` (Lesson 30).
4. **State**: local state first → React Context for multiple screens in one feature → Redux (ducks) for data shared across features.
5. **`front/javascripts/react_components/` is in maintenance mode** — don't add new code there; put generic components in `@freee-payroll/components`.
6. **`DEPRECATED_` prefix** means "this feature uses Flux and breaks the recommended layout". That's the *only* reason. You may still add code there when needed — "don't overreact" (過剰反応しないこと).
7. **Vibes `FormControl` a11y rule** (see §7.3).

---

## 7. The UI kit: Vibes (`@c-fo/vibes`)

About **2,350** files import from `@c-fo/vibes`. You almost never write raw `<input>` or `<button>`; you use Vibes components.

### 7.1 Common components

| Component | What it is | Typical props |
|---|---|---|
| `Button` | A button | `appearance="primary"|"secondary"|...`, `onClick`, `disabled` |
| `TextField` | One-line text input | `value`, `onChange`, `id`, `label`, `width`, `disabled` |
| `SelectBox` | Drop-down | `options`, `value`, `onChange` |
| `DateInput`, `DecimalInput` | Date / number inputs | `value`, `onChange` |
| `FormControl` | Label + wrapper around an input | `label`, `fieldId`, `required` |
| `TaskDialog` | A modal dialog with OK/Cancel | `isOpen`, `title`, `onPrimaryAction`, `onRequestClose`, `primaryButtonLabel` |
| `MarginBase` | Adds margin around children | `mb`, `mt`, `ml`, `mr` |
| `VStack` / `HStack` | Stack children vertically / horizontally | `gap`, `alignItems` |
| `ColumnBase`, `ContentsBase` | Page/section boxes | `border`, `paddingSize` |
| `HeadlineArea` | Page title area | `pageTitle` |
| `DescriptionList` | Label/value list | `listContents` |
| `Loading` | Spinner overlay | `isLoading`, `coverAll` |
| `Text`, `Paragraph`, `Message` | Text, paragraph, (error) message | `error` |

**Spacing props** like `mb={1}` mean "margin-bottom 1 unit" (units are rem-based; `1.5`, `0.5`, even `-1.5` are used). You'll see `mr`, `ml`, `mt`, `mb`, `ma` on many Vibes components.

### 7.2 A real form inside a dialog

`front/javascripts/employees/components/Employee/DisplayNameInputModal.tsx:61-86`:

```tsx
    <TaskDialog
      closeButtonLabel="キャンセル"
      data-test="表示名入力モーダル"
      isOpen={isOpen}
      onPrimaryAction={() => onSave(trimmed)}
      onRequestClose={onRequestClose}
      primaryButtonLabel="保存"
      suspend={isLoading || trimmed.length === 0}
      title="表示名を入力してください"
    >
      <MarginBase mb={1.5}>
        <FormControl
          fieldId="member_replacement_display_name"
          label="表示名"
          required
        >
          <TextField
            autoComplete="off"
            data-test="表示名入力"
            id="member_replacement_display_name"
            maxLength={DISPLAY_NAME_MAX_LENGTH}
            onChange={(e) => setDisplayName(e.currentTarget.value)}
            value={displayName}
            width="medium"
          />
        </FormControl>
```

Things to notice:
- **Props are in alphabetical order** (`closeButtonLabel`, `data-test`, `isOpen`, …). That's enforced by `.eslintrc.js:336` `'react/jsx-sort-props': 'error'`. Don't be surprised when the linter reorders you!
- `suspend=` disables the primary button while loading or while the text is empty.
- `data-test="..."` attributes are hooks for automated E2E tests to find elements. Keep them when editing.
- It's a **controlled input** (`value` + `onChange` → state) — Lesson 27.

### 7.3 The a11y rule for `FormControl`

From `docs/knowledge/general/coding-standards/frontend-a11y.md`:

- Just giving `FormControl` a `label` does **not** connect the label to the input. A screen reader can't say what the field is.
- **Case 1 (one input):** set `FormControl fieldId="X"` and the child's `id="X"` to the **same value**. Vibes then renders a real `<label htmlFor="X">`.
- **Case 2 (same label twice on a page):** ids must be unique, so give aliases (e.g. `_alt`).
- **Case 3 (several inputs in one FormControl):** don't use `fieldId`; give each child its own `label` prop.

The dialog above follows Case 1: `fieldId="member_replacement_display_name"` = `id="member_replacement_display_name"`. ✅

### 7.4 Other UI libraries you'll see

| Package | Used for |
|---|---|
| `@c-fo/standard-ui` | Page-level layouts (list pages, headers) |
| `@c-fo/table` | Data tables (`Table`, `Column`, `Row`) |
| `@c-fo/icons`, `react-icons` | Icons, e.g. `import { MdCelebration } from 'react-icons/md';` |
| `@freee-payroll/components` | freee-payroll–specific parts: `App`, `ErrorPage`, `NotificationProvider`/`useNotification`, … |

`front/packages/@freee-payroll/components/README.md` says: payroll-specific parts (like `App`, `ErrorPage`) stay here; generic parts (like modals) should eventually be replaced by Vibes.

---

## 8. Styling: how (and how not) to add CSS

| Way | Allowed? | Notes |
|---|---|---|
| Vibes props (`mb`, `width`, `appearance`) | ✅ First choice | Most styling needs are covered |
| Inline `style={{ ... }}` | ✅ OK | e.g. `<div style={{ display: 'flex', marginRight: '1.5rem' }}>` in `company_social_insurance_settings/components/common/RateInputRow.tsx` |
| Sass in `front/stylesheets` | ✅ when you need pseudo-classes (`:hover`) | Built by Vite |
| **styled-components** | ❌ Banned by lint | `.eslintrc.js:289-291`: "Don't use styled-components with import, please use `style` prop or Sass…" |
| CSS Modules | ❌ Not used | overview-frontend.md: hard to plug into Rails assets; Vibes covers most needs |

There is exactly **one** real styled-components use, in `front/packages/@freee-payroll/components/src/SortableItems.tsx:35` (with an `eslint-disable`). Don't copy it.

> The docs even say: if you find yourself hand-writing lots of styles, treat it as a *smell* in the design spec — talk to the designer.

---

## 9. i18n (multiple languages) — rare here

- Almost all UI text is **hard-coded Japanese** directly in JSX (`label="表示名"`).
- `react-i18next` is used in only ~16 files, mostly the **healthcare** feature (e.g. `front/javascripts/healthcare/components/StressCheck/Survey.tsx`), which has English support.
- Setup lives in `front/packages/@freee-payroll/i18n/i18n.ts` with `locales/ja/translation.json` and `locales/en/translation.json`.

Usage pattern:

```tsx
import { useTranslation } from 'react-i18next';
...
const { t } = useTranslation();
...
title={t('title')}
```

(`t('key')` looks up the key in the current language's JSON file.)

---

## 10. 🔎 Practice (15 min)

**Practice 1: trace a screen.** For `/holidays`:
1. Find the Rails route in `config/routes/web.rb` (hint: line 46).
2. Find its `JS_FILE_MAP` line in `app/helpers/application_helper.rb`.
3. Open the entry file. Which function does it call?
4. Open `front/javascripts/holidays/Routes.tsx`. What does line 21 read from `$FREEE_DATA`?

**Practice 2: list the folders.** Run `ls front/javascripts/holidays`. Which subfolders does it have? (Answer: `components/`, `ducks/`, `hooks/` plus files like `Routes.tsx`, `index.ts`, `types.ts`, `constants.tsx`, `utils.ts`, `snapshots.test.ts`.)

**Practice 3: is this import OK?** A file in `front/javascripts/shift/` writes:
- a) `import { useLoading } from '@freee-payroll/hooks';`
- b) `import { HomePage } from '../hr_portal/components/HomePage';`
- c) `import NavigationContextAction from '../flux_infra/actions/NavigationContextAction';`

*(Answers: a ✅, b ❌ cross-feature, c ✅ flux_infra is a non-restricted dir.)*

**Practice 4: spot the a11y bug.** (illustrative example)

```tsx
<FormControl fieldId="employee_num" label="従業員番号">
  <TextField id="emp_num" value={num} onChange={onChange} />
</FormControl>
```

*(Answer: `fieldId` and `id` differ, so the label isn't connected. Make both `employee_num`.)*

**Practice 5: split it.** In `front/javascripts/renew_settings2/components/GroupImportHistories.tsx`, find the presentational component and the container. Which one calls `API.OrganizationGroupImport.index()`? *(The container, `GroupImportHistoriesContainer`.)*

---

## 11. Summary

- `front/javascripts/<feature>/` = one feature; `front/packages/@freee-payroll/*` = shared code (Yarn workspaces).
- URL → Rails route → `web/app.html.erb` (`#app`) → `JS_FILE_MAP` → feature `index.ts` → `run()` → `createRoot` → `Routes`.
- **1 feature = 1 entrypoint** (`front/jsEntries.js`), and **no cross-feature imports** (ESLint `import/no-restricted-paths`).
- Files usually export a **presentational** component (default) and a **Container** (named) that fetches data.
- UI = **Vibes**. Props sorted alphabetically. `FormControl fieldId` must equal the child's `id`.
- Style with Vibes props / inline `style` / Sass. **No styled-components, no CSS Modules.**
- `react_components/` = maintenance mode. `DEPRECATED_` = uses Flux; still OK to touch.
- i18n is rare; UI text is mostly hard-coded Japanese.

➡️ Now take **Test29.md**.
