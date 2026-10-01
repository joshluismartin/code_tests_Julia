# Lesson 31 — Frontend Testing & Tooling

**Time:** ~1 hour (40 min reading + practice, 20 min test)
**Goal:** Read and write frontend tests (Vitest + Testing Library), write Storybook stories (which double as snapshot tests), and know what every frontend CI check does.
**You need:** Lessons 20–21 (the *idea* of tests from RSpec), Lessons 26–30 (React). Open the repo in your editor.

---

## 1. New words for this lesson

| Term | Meaning in plain words | Example |
|---|---|---|
| **Vitest** | The JavaScript/TypeScript test runner (like RSpec for the frontend). | `yarn run vitest run` |
| **Test file** | A file ending in `.test.ts` / `.test.tsx` next to the code it tests. | `ApplyYearMonthSection.test.tsx` |
| **Assertion / matcher** | A check: "I expect X to equal Y". | `expect(x).toBe(1)` |
| **Mock** | A fake replacement for a real function/module, so tests don't call the real server. | `vi.fn()`, `vi.mock(...)` |
| **Testing Library** | Helpers to render a component into a fake page and find elements "like a user would" (by text, by role). | `render`, `screen.getByText` |
| **DOM** | The tree of HTML elements the browser builds. Tests use a fake DOM. | `document.querySelectorAll` |
| **happy-dom** | A fake browser DOM that runs inside Node, so tests don't need a real browser. | `vitest.config.mts` |
| **Storybook** | A web app that shows each component in isolation, in different states ("stories"). | `yarn storybook` |
| **Story** | One named state of a component (e.g. `Default`, `ReadOnly`). | `export const ReadOnly = () => ...` |
| **CSF** | *Component Story Format*: stories written as a default export + named exports. | `RateInputRow.stories.tsx` |
| **Snapshot test** | Saves the rendered HTML of a story to a file; later runs fail if the HTML changes unexpectedly. | `__snapshots__/*.snapshot` |
| **Linter** | Finds bad patterns (ESLint). | `yarn run lint` |
| **Formatter** | Rewrites code layout automatically (Prettier). | `npx prettier -c .` |
| **Type check** | Runs the TypeScript compiler only to find type errors (no output files). | `yarn run tsc` |
| **Dependency cruiser** | Checks rules about which files may import which (e.g. no circular imports). | `yarn run depcruise front` |

---

## 2. What CI runs for the frontend

Open `.circleci/config.yml`. The `front_test` job (starts at line 426) runs (verified, lines 436-445):

```yaml
      - run:
          name: Run test commands for frontend
          command: |
            export NODE_OPTIONS="--max-old-space-size=6144"
            yarn run lint
            yarn run depcruise front
            yarn run tsc
            yarn run tsgo
            export NODE_OPTIONS="--max-old-space-size=2048"
            yarn run vitest run --silent true
```

and the `prettier` job runs `npx prettier -c .` (line 423).

| Command | What it checks | Config |
|---|---|---|
| `yarn run lint` | ESLint rules (the `package.json` script `eslint --cache ... front`) | `.eslintrc.js` |
| `yarn run depcruise front` | Import rules (no circular imports, no dev-only packages in production code, no unused files) | `.dependency-cruiser.js` |
| `yarn run tsc` | TypeScript type check (`noEmit: true` in `tsconfig.json`) | `tsconfig.json` |
| `yarn run tsgo` | Same type check with the new native TypeScript compiler (`@typescript/native-preview` in `package.json`) | `tsconfig.json` |
| `yarn run vitest run` | All frontend tests, including story snapshots | `vitest.config.mts` |
| `npx prettier -c .` | Formatting (`-c` = check only) | `.prettierrc.js` → `require('prettier-config-freee')` |

> `yarn run <name>` runs a `package.json` script **or** an installed tool's command with that name. `tsc`, `vitest`, `depcruise` are tool commands; `lint` is a script.

**Run things locally** (from the repo root):

```bash
yarn run lint                                  # ESLint
yarn run tsc                                   # type check
yarn run vitest run front/javascripts/company_social_insurance_settings   # tests in one folder
yarn run vitest run path/to/File.test.tsx      # one test file
npx prettier -w path/to/File.tsx               # fix formatting of one file (-w = write)
yarn storybook                                 # open Storybook at http://localhost:6006
```

(`yarn storybook` = `storybook dev -p 6006` in `package.json`.)

### 2.1 The dependency-cruiser rules (plain words)

`.dependency-cruiser.js` (verified):
- `no-circular-at-runtime`: A imports B imports A → error.
- `not-to-dev-dep`: production code under `front/` must not import dev-only packages (tests/stories are allowed to).
- `no-unused`: files not reachable from any feature `index.ts(x)` are flagged (dead code).

---

## 3. Vitest basics

### 3.1 Configuration

`vitest.config.mts` (repo root, lines 6-10):

```ts
export default defineConfig({
  root: path.dirname(url.fileURLToPath(import.meta.url)),
  test: {
    environment: 'happy-dom',
    setupFiles: ['.vitest/setup.ts'],
```

- `environment: 'happy-dom'` → tests get a fake `window`/`document`.
- `setupFiles` → `.vitest/setup.ts` runs before tests. It sets a default `window.$FREEE_DATA` (with e.g. `ff: {}`) so code that reads the global doesn't crash.

### 3.2 The vocabulary (compare with RSpec)

| RSpec (Ruby) | Vitest (TS) | Meaning |
|---|---|---|
| `describe 'X' do` | `describe('X', () => { ... })` | Group of tests |
| `context '...'` | `describe('...')` (nested) | Sub-group |
| `it '...' do` | `it('...', () => { ... })` (or `test`) | One test |
| `before do` | `beforeEach(() => ...)` | Run before each test |
| `after do` | `afterEach(() => ...)` | Run after each test |
| `expect(x).to eq(1)` | `expect(x).toBe(1)` | Same value (`===`) |
| `expect(h).to eq({...})` | `expect(h).toEqual({...})` | Deep equality for objects/arrays |
| `double` / `allow(...).to receive` | `vi.fn()` / `vi.mock()` | Fakes |
| `have_received(:x).with(1)` | `expect(fn).toHaveBeenCalledWith(1)` | Was the fake called with these args? |
| — | `toHaveBeenCalledTimes(1)`, `toContain('x')` | Call count / contains |

**`toBe` vs `toEqual`:** `toBe` checks it's the *same thing* (`===`); two different objects with the same contents are **not** `toBe` each other. Use `toEqual` for objects/arrays.

```ts
// (illustrative example)
expect(1 + 1).toBe(2);                     // ✅
expect({ a: 1 }).toBe({ a: 1 });           // ❌ fails: different objects
expect({ a: 1 }).toEqual({ a: 1 });        // ✅
```

### 3.3 A pure-function test with `$FREEE_DATA` flags

`front/javascripts/shift/components/EmployeeActivation.test.tsx:1-35` (verified):

```tsx
import { afterEach, beforeEach, describe, expect, it } from 'vitest';

import '../../../../.storybook/globalProps';

import { buildEmployeeCountChangeLinkProps } from './EmployeeActivation';

const GOOGLE_FORM_URL =
  'https://docs.google.com/forms/d/e/1FAIpQLSdnWoBJdm5tJo5OCIAYW8FcqtTn_UHXlkMTrNNQl6xJuMrlYQ/viewform';

describe('buildEmployeeCountChangeLinkProps', () => {
  const originalFlag = $FREEE_DATA.ff.hr_shift_self_purchase;
  const originalSubscriptionNumber =
    $FREEE_DATA.applied_plans.zuora_subscr_no.hr_shift_employee;

  afterEach(() => {
    $FREEE_DATA.ff.hr_shift_self_purchase = originalFlag;
    $FREEE_DATA.applied_plans.zuora_subscr_no.hr_shift_employee =
      originalSubscriptionNumber;
  });

  describe('feature flag hr_shift_self_purchase が false のとき', () => {
    beforeEach(() => {
      $FREEE_DATA.ff.hr_shift_self_purchase = false;
    });

    it('AIシフト契約番号の有無によらず、既存の Google Forms へのリンクを返す', () => {
      $FREEE_DATA.applied_plans.zuora_subscr_no.hr_shift_employee = '12345';

      expect(buildEmployeeCountChangeLinkProps()).toEqual({
        linkTitle: '従業員数の変更',
        linkUrl: GOOGLE_FORM_URL,
        linkTarget: '_blank',
      });
    });
  });
```

What to learn:
- `import '../../../../.storybook/globalProps';` — an import **only for its side effect**: it fills `window.$FREEE_DATA` with realistic fake data (`.storybook/globalProps.ts:5` `window.$FREEE_DATA = { pref: [...] ...`). Many tests do this.
- **Save → change → restore:** the test saves the original flag, sets it in `beforeEach`, and restores it in `afterEach`, so other tests aren't affected. (Tests may mutate `$FREEE_DATA` like this; Storybook stories must not.)
- It tests both flag states in separate `describe` blocks — same idea as "separate new/old contexts for feature flags" in `feature-flag.md`.

---

## 4. Testing components with Testing Library

### 4.1 The tools

| Function | From | Does |
|---|---|---|
| `render(<X />)` | `@testing-library/react` | Renders a component into the fake DOM |
| `screen.getByText('...')` | same | Finds an element by its visible text (throws if 0 or >1 found) |
| `screen.getByRole('button', { name: '...' })` | same | Finds by accessibility role + name (the best way — like a screen reader) |
| `fireEvent.click(el)` | same | Simulates a click |
| `cleanup` | same | Removes rendered components after a test |
| `renderHook(() => useX())` | same | Runs a hook inside a tiny test component; `result.current` is its return value |
| `act(async () => ...)` | same | Wraps actions that update state so React finishes updating before you assert |
| `vi.fn()` | `vitest` | A fake function that records its calls |

### 4.2 A real component test

`front/javascripts/company_social_insurance_settings/components/common/ApplyYearMonthSection.test.tsx` (verified; lines 1-28 and 44-56):

```tsx
import { cleanup, fireEvent, render, screen } from '@testing-library/react';
import * as React from 'react';
import { afterEach, describe, expect, it, vi } from 'vitest';

import '../../../../../.storybook/globalProps';

import ApplyYearMonthSection, {
  centeredScrollTop,
} from './ApplyYearMonthSection';

// movableRange (globalProps): 2013-12 〜 2017-12 + 2ヶ月延長のため、
// 2018-02 まで選択肢が存在する。範囲の中間である 2016-06 を使う
const renderSection = (
  year: number,
  month: number,
  onChange: (year: number, month: number) => void = vi.fn()
) =>
  render(
    <ApplyYearMonthSection
      expiredMonth={null}
      expiredYear={null}
      month={month}
      onChange={onChange}
      year={year}
    />
  );

afterEach(cleanup);
...
  it('選択肢を選ぶと onChange がその年月で呼ばれる', () => {
    const onChange = vi.fn();
    renderSection(2016, 6, onChange);

    fireEvent.click(screen.getByText('2016年6月'));

    // getByRole の name は既定で完全一致のため、…衝突しない
    fireEvent.click(screen.getByRole('button', { name: '2016年7月' }));

    expect(onChange).toHaveBeenCalledWith(2016, 7);
  });
```

Read it as a story:
1. A helper `renderSection` renders the component; the `onChange` prop defaults to a fake `vi.fn()`.
2. The test clicks the current value (opens the dropdown), then clicks the "2016年7月" button.
3. It asserts the component called `onChange(2016, 7)`.

This is **testing behavior, not implementation**: it acts like a user (click text/buttons) and checks the visible result or the callback — never internal state. (Same spirit as `spec/CLAUDE.md`: don't transcribe implementation into tests.)

### 4.3 Testing a hook with mocks

`front/javascripts/company_social_insurance_settings/hooks/useCompanyProfileFetcher.test.ts:1-20` (verified):

```ts
import * as API from '@freee-payroll/api';
import { act, renderHook } from '@testing-library/react';
import { AuthCompany } from 'freee-types';
import { type Mock, beforeEach, describe, expect, it, vi } from 'vitest';

import { useCompanyProfileFetcher } from './useCompanyProfileFetcher';

vi.mock('@freee-payroll/api', () => ({
  fetchCompanyProfile: vi.fn(),
  getErrorMessage: vi.fn(),
}));

const mockNotification = vi.fn();
vi.mock('@freee-payroll/components', () => ({
  useNotification: () => mockNotification,
}));

const showLoading = vi.fn();
const hideLoading = vi.fn();
vi.mock('@freee-payroll/hooks', () => ({
```

and in a test (lines ~70-86):

```ts
      mockFetchCompanyProfile.mockResolvedValueOnce(
        buildFetchResponse(buildAuthCompany())
      );

      const { result } = renderHook(() =>
        useCompanyProfileFetcher({ onApply })
      );

      await act(async () => {
        await result.current.fetch();
      });

      expect(onApply).toHaveBeenCalledTimes(1);
```

- `vi.mock('@freee-payroll/api', () => ({...}))` **replaces the whole module** in this test file: no real HTTP happens.
- `mockResolvedValueOnce(x)` = "the next call returns a Promise that resolves to x" (like RSpec `and_return` for async).
- `renderHook` runs the hook; `result.current.fetch()` calls a function the hook returned.
- `act(async () => ...)` waits for the state updates.

---

## 5. Storybook stories

### 5.1 CSF format

`front/javascripts/company_social_insurance_settings/components/common/RateInputRow.stories.tsx:1-28` (verified):

```tsx
import { action } from '@storybook/addon-actions';
import * as React from 'react';

import RateInputRow from './RateInputRow';

export default {
  component: RateInputRow,
};

export const Default = () => (
  <RateInputRow
    companyRate={5}
    employeeRate={5}
    onChangeCompanyRate={action('onChangeCompanyRate')}
    onChangeEmployeeRate={action('onChangeEmployeeRate')}
    rateType="base_rate"
    readOnly={false}
  />
);

export const ReadOnly = () => (
  <RateInputRow
    companyRate={9.15}
    employeeRate={9.15}
    rateType="welfare_pension_insurance_rate"
    readOnly
  />
);
```

- `export default { component: X }` — tells Storybook which component this file is about.
- Each **named export** = one story (one state). Names in PascalCase: `Default`, `ReadOnly`, `WithFieldErrors`.
- `action('name')` — a fake callback that logs "onChangeCompanyRate was called with …" in the Storybook UI.
- Stories render the **presentational** component with plain props — this is why the Presentational/Container split (Lesson 29) matters.
- `storiesOf(...)` (old API) is **banned by ESLint** — write CSF.
- Storybook finds stories via `.storybook/main.ts`: `stories: ['../front/**/*.stories.tsx']`.
- Remember the `$FREEE_DATA` rule: **never** mutate `$FREEE_DATA` in a story; pass values as props.

### 5.2 Stories = snapshot tests (automatically)

`front/javascripts/company_social_insurance_settings/snapshots.test.ts:1-8`:

```ts
import { runSnapshotTests } from 'test-helper';
import { describe } from 'vitest';

import '../../../.storybook/globalProps';

describe('Snapshots', () => {
  runSnapshotTests(__dirname);
});
```

`runSnapshotTests` (`front/packages/test-helper/index.tsx:20`) finds every `**/*.stories.tsx` under the folder, renders each story, and compares the HTML with a file in a `__snapshots__/` folder next to the story, e.g.
`front/javascripts/company_social_insurance_settings/components/common/__snapshots__/ApplyYearMonthSection.stories-Default.snapshot`.

Rules from `front/README.md`:
- **Don't write individual snapshot tests** — add a story instead; it becomes a snapshot automatically.
- When you intentionally change UI, the snapshot diff shows up in your PR. Review it (e.g. the a11y doc uses the snapshot diff `<span>` → `<label htmlFor>` as proof its fix worked), then update the snapshot files (Vitest's update flag: `yarn run vitest run -u <path>`).

### 5.3 The testing pyramid here

| What | Tool | Where |
|---|---|---|
| Visual states of components | Storybook stories → snapshots | `*.stories.tsx` + `snapshots.test.ts` per feature |
| Logic in pure functions/hooks | Vitest (+ `renderHook`) | `*.test.ts` |
| User interactions on a component | Vitest + Testing Library | `*.test.tsx` |
| Server API behavior | RSpec request specs | `spec/controllers/**` (Lesson 21) |

---

## 6. ESLint rules you'll bump into (collected)

All in `.eslintrc.js` (verified):

| Rule | Line | What it wants |
|---|---|---|
| `@typescript-eslint/consistent-type-definitions: ['error', 'type']` | 146 | Use `type`, not `interface` |
| `no-restricted-imports` (styled-components, `storiesOf`, Vibes `Container`) | ~272-291 | Don't import these |
| `react-hooks/rules-of-hooks` | ~332 | Hooks only at top level of components/hooks |
| `react/jsx-sort-props` | 336 | Alphabetical JSX props |
| `import/no-restricted-paths` | 351 | No cross-feature imports |
| `import/order` (alphabetize, newlines between groups) | ~353 | Import order: packages, then blank line, then relative imports |

Prettier (`prettier-config-freee`) handles quotes, semicolons, wrapping — just run it; don't argue with it.

Occasionally you'll see `// eslint-disable-next-line <rule> -- <reason>` (e.g. `run.tsx` and `admin2/index.tsx` for the `!` assertion). The team writes a **reason after `--`** when disabling a rule.

---

## 7. 🔎 Practice (15 min)

**Practice 1: CI mapping.** For each problem, which CI command fails?
a) A file in `hr_portal` imports from `holidays`. b) A variable typed `number` receives a `string`. c) Code uses double spaces instead of Prettier's style. d) A circular import. e) A test's `expect` fails.
*(a `yarn run lint`; b `yarn run tsc`/`tsgo`; c `npx prettier -c .`; d `yarn run depcruise front`; e `yarn run vitest run`)*

**Practice 2: find the snapshots.** Run
`ls front/javascripts/company_social_insurance_settings/components/common/__snapshots__`
How many snapshot files exist for `ApplyYearMonthSection`? What are the story names? (Look at the file names: `...stories-Default.snapshot`, `-Disabled`, `-Interactive`.)

**Practice 3: toBe or toEqual?** Which matcher for: a) `sum(1, 2)` returns `3`; b) `buildProps()` returns `{ a: 1 }`; c) `['x', 'y']`.
*(a toBe; b toEqual; c toEqual)*

**Practice 4: write a story (on paper).** For `JoiningEmployeesPanel` (Lesson 29; props `{ employeeCount: number }`), write a CSF file with stories `One` (1 employee) and `Many` (25). *(Answer: `export default { component: JoiningEmployeesPanel };` + `export const One = () => <JoiningEmployeesPanel employeeCount={1} />;` + `export const Many = ...`.)*

**Practice 5: run one test (if your local env is set up).** `yarn run vitest run front/javascripts/shift/components/EmployeeActivation.test.tsx`

---

## 8. Summary

- Frontend CI (`front_test`): `lint` → `depcruise front` → `tsc` → `tsgo` → `vitest run`; plus `prettier -c .`.
- **Vitest**: `describe` / `it` / `beforeEach` / `afterEach` / `expect(...).toBe|toEqual|toHaveBeenCalledWith`; fakes with `vi.fn()` / `vi.mock()`; happy-dom fake browser; `.vitest/setup.ts` presets `$FREEE_DATA`.
- **Testing Library**: `render`, `screen.getByText` / `getByRole`, `fireEvent.click`, `renderHook`, `act`. Test behavior, not internals.
- Tests often `import '.storybook/globalProps'` for realistic `$FREEE_DATA`; if a test changes a flag, restore it in `afterEach`.
- **Storybook (CSF)**: `export default { component }` + one named export per state; `action()` for callbacks; no `storiesOf`; no mutating `$FREEE_DATA`.
- Each feature's `snapshots.test.ts` turns **all stories into snapshot tests** — don't write separate snapshot tests.

➡️ Now take **Test31.md**.
