# Test 29 — React in freee-payroll 1: Structure & UI

**Rules:** Don't look at Lesson29.md while answering. Write your answers under each question. Then check `Answers29.md`.
**Total:** 39 points

---

## Part A — Multiple choice (1 point each)

**1.** What is an *entrypoint*?
- a) The login screen
- b) The first file the bundler starts from to build one JS bundle
- c) The Rails controller for a page
- d) The `<div id="app">` element

**2.** In freee-payroll, a *feature directory* is:
- a) A folder in `app/modules/`
- b) A folder under `front/javascripts/` holding all the code of one product area
- c) Any folder that has a `package.json`
- d) A folder under `front/stylesheets/`

**3.** Why can code write `import * as API from '@freee-payroll/api'`?
- a) It's published on npm
- b) `tsconfig.json` has a `paths` alias
- c) Yarn workspaces link `front/packages/@freee-payroll/*` as named packages
- d) Vite downloads it

**4.** Which file lists all the JS bundles Vite builds?
- a) `package.json`
- b) `front/jsEntries.js`
- c) `config/routes.rb`
- d) `front/README.md`

**5.** Which of these is **forbidden** by ESLint in a file inside `front/javascripts/shift/`?
- a) `import { useLoading } from '@freee-payroll/hooks';`
- b) `import run from '../initializers/run';`
- c) `import { HomePage } from '../hr_portal/components/HomePage';`
- d) `import { Button } from '@c-fo/vibes';`

**6.** What does the `DEPRECATED_` prefix on a feature folder mean?
- a) The folder will be deleted next week; never touch it
- b) The feature uses Flux and breaks the recommended layout; you may still add code if needed
- c) The feature is behind a feature flag
- d) The folder only contains tests

**7.** What is **Vibes** (`@c-fo/vibes`)?
- a) A state-management library
- b) freee's design system / UI component library
- c) A test runner
- d) The Rails view layer

**8.** How should you add styles that Vibes props can't handle (e.g. `:hover`)?
- a) styled-components
- b) CSS Modules
- c) Sass (or the inline `style` prop for simple cases)
- d) A `<style>` tag inside the component

**9.** Which rule does `'react/jsx-sort-props': 'error'` enforce?
- a) Components must be sorted in the file
- b) JSX props must be in alphabetical order
- c) Imports must be sorted
- d) Routes must be sorted

**10.** What does the `run()` function in `initializers/run.tsx` give every feature automatically?
- a) A Rails controller
- b) `NotificationProvider`, the Redux `Provider` with the shared store, and a mount on `#app`
- c) A database connection
- d) Storybook

---

## Part B — Fill in the blanks (1 point each blank)

**11.** The rule from `overview-frontend.md` is: 1 ______ = 1 ______.

**12.** The Rails helper constant that maps `'web/hr_portal'` to `'front/javascripts/hr_portal/index.ts'` is called ______ and lives in `app/helpers/______.rb`.

**13.** Generic, reusable React components should go into the package `@freee-payroll/______`, NOT into `front/javascripts/______/` (maintenance mode).

**14.** For accessibility, `FormControl`'s ______ prop must have the **same value** as the child input's ______ prop.

**15.** `data-test="..."` attributes exist so that automated ______ tests can find elements.

---

## Part C — Fill in the missing code (2 points each)

**16.** Complete a typical feature entry file `front/javascripts/my_feature/index.ts`:

```ts
import run from '________________';

import Routes from './Routes';

run({ ________________: Routes });
```

**17.** Fix this snippet so the label is correctly connected (one input):

```tsx
<FormControl ________="retire_date" label="退職日">
  <DateInput ________="retire_date" onChange={onChange} value={value} />
</FormControl>
```

---

## Part D — Write the code (3 points each)

**18.** Write a file with **both** a presentational component and a container:
- Presentational `PendingCountPanel` takes a prop `count: number` and shows the text `` `${count}件の申請があります` `` inside a Vibes `Text`.
- Default-export the presentational component.
- Named export `PendingCountPanelContainer` that holds `count` in `useState` (start at `0`) and, in `useEffect`, calls `API.apiPPendingRequestsGet()` (pretend it exists) and sets `count` to `response.data.length`. Ignore errors.
- Use `import * as React from 'react'` style like the repo.

**19.** Write a Vibes `TaskDialog` that:
- is open when `isOpen` is true,
- has title `"メモを入力"`, primary button `"保存"`, close button `"キャンセル"`,
- contains a `FormControl` (label `"メモ"`) with a `TextField` (controlled by `memo` / `setMemo`), with correct a11y ids,
- calls `onSave(memo)` on the primary action and `onClose` on request close.
Keep props in alphabetical order.

---

## Part E — Answer in a sentence or two (2 points each)

**20.** Why does freee-payroll forbid importing from one feature directory into another? What should you do instead when two features need the same code?

**21.** Explain the difference between a *presentational* component and a *Container* component, and why the split makes Storybook easier.

**22.** Why is `styled-components` not used here? Name two approved alternatives.

---

## Part F — Read real repo code (4 points)

**23.** This is `front/javascripts/hr_portal/Routes.tsx:18-27`:

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
```

a) Where does `$FREEE_DATA.ff.hr_portal_enabled` come from (which side creates it)? (1 pt)
b) What does a user see when the flag is `false`, for any path? (1 pt)
c) Starting from the browser requesting `/hr_portal`, list in order the files that run until this `Routes` component renders (at least 4 steps). (2 pts)
