# Test 30 — React in freee-payroll 2: Data, Routing, Forms & Legacy

**Rules:** Don't look at Lesson30.md while answering. Write your answers under each question. Then check `Answers30.md`.
**Total:** 42 points

---

## Part A — Multiple choice (1 point each)

**1.** Which is the **first choice** for calling a Private API endpoint from React?
- a) `fetch()`
- b) The generated client in `@freee-payroll/api`
- c) `configured-axios`
- d) jQuery `$.ajax`

**2.** When should you use `configured-axios` directly?
- a) Always
- b) For endpoints that have no OpenAPI (swagger) definition
- c) Only in tests
- d) Never; it's deprecated

**3.** What is **CSRF**?
- a) A CSS framework
- b) An attack where another site tricks your logged-in browser into sending requests; prevented with a secret token
- c) A React hook
- d) A kind of database index

**4.** How does the CSRF token get onto requests here?
- a) The developer adds it by hand each time
- b) axios reads the `XSRF-TOKEN` cookie and sends it as the `X-CSRF-TOKEN` header
- c) It's in the URL query string
- d) It's stored in `$FREEE_DATA.ff`

**5.** In `https://example.com/hr_portal#/actions/42`, what does the Rails server receive as the path?
- a) `/hr_portal#/actions/42`
- b) `/actions/42`
- c) `/hr_portal`
- d) `#/actions/42`

**6.** In react-router v3, `<Route path="/actions/:id" ... />` — what is `:id`?
- a) A query string
- b) A path parameter
- c) A CSS id
- d) A feature flag

**7.** According to `front/README.md`, where should state shared across **different features** live?
- a) Local `useState`
- b) React Context
- c) Redux using the ducks pattern
- d) `$FREEE_DATA`

**8.** What does zod's `safeParse` return?
- a) It throws on invalid data
- b) An object with `success: true, data` or `success: false, error`
- c) A Promise
- d) A React component

**9.** A folder named `DEPRECATED_payroll_statements` means:
- a) It must never be edited
- b) It uses Flux and breaks the recommended layout; editing is allowed when needed
- c) It's been deleted from production
- d) It's a test folder

**10.** Which is CoffeeScript?
- a) `const f = () => this.id;`
- b) `url: -> "/api/p/employees/#{@employeeID}"`
- c) `def url; "/api/p/employees/#{@employee_id}"; end`
- d) `<Route path="/" />`

---

## Part B — Fill in the blanks (1 point each blank)

**11.** The generated function for `GET /api/p/employee_display_names/{id}` is named ______.

**12.** After changing a `*.swagger.yaml`, you regenerate the TypeScript client with `make ______`.

**13.** `useLoading()` from `@freee-payroll/hooks` returns `isLoading`, ______ and ______.

**14.** To show an error toast you call `notify({ message: API.______(e), type: '______' })`, where `notify` comes from `______()`.

**15.** freee-payroll uses react-router version ______ with `______History`, so in-app paths appear after the `#` in the URL.

**16.** To navigate from code you write `hashHistory.______('/company_social_insurance_settings?tab=x')`.

---

## Part C — Fill in the missing code (2 points each)

**17.** Complete the API call with loading and error handling:

```tsx
const { hideLoading, isLoading, showLoading } = ______();
const notify = ______();

React.useEffect(() => {
  showLoading();
  (async () => {
    ______ {
      const { data } = await API.apiPCommentsGet();
      setComments(data);
    } ______ (e) {
      notify({ message: API.getErrorMessage(e), type: 'error' });
    } ______ {
      hideLoading();
    }
  })();
}, [hideLoading, notify, showLoading]);
```

**18.** Complete `Routes.tsx` for a feature with a home page at `/` and a detail page at `/items/:id`:

```tsx
import { App } from '@freee-payroll/components';
import * as React from 'react';
import { IndexRoute, Route, Router, ______ } from 'react-router';

const Routes = () => (
  <Router history={______}>
    <Route component={(s) => <App responsive {...s} />} path="/">
      <______ component={ItemsHomePage} />
      <Route component={ItemDetailPageContainer} path="______" />
    </Route>
  </Router>
);

export default Routes;
```

**19.** Rewrite this to follow the `$FREEE_DATA` rule in `front/README.md`:

```tsx
const isNewListEnabled = $FREEE_DATA.ff.new_list_enabled;
...
{isNewListEnabled && <NewList />}
```

---

## Part D — Write the code (3 points each)

**20.** Write a custom hook `useTaxOffices(pref: string)` that:
- keeps `offices` in state (start `[]`),
- uses `useLoading` and `useNotification`,
- in `useEffect` (re-running when `pref` changes) calls `API.TaxOffice.index(pref)` and stores the result,
- shows an error toast with `API.getErrorMessage` on failure,
- ignores the result if the component unmounted (cleanup flag),
- returns `{ isLoading, offices }`.

**21.** Write a Formik-based component field `NoteField` that renders a Vibes `TextField` connected to Formik via `<Field name={name}>`, sets the value with `form.setFieldValue`, and shows `<Message error>` below it when `form.errors[name]` is a string.

---

## Part E — Answer in a sentence or two (2 points each)

**22.** Why does freee-payroll use `hashHistory` (URLs with `#`) so that Rails web routes can stay simple?

**23.** Why should you **not** write `const fooEnabled = $FREEE_DATA.ff.foo_enabled` and then use `fooEnabled`? And what should Storybook do instead of changing `$FREEE_DATA`?

**24.** `hasPermission(...)` hides a button in the UI. Is that enough to protect the feature? Why or why not?

---

## Part F — Read real repo code (4 points)

**25.** `front/javascripts/DEPRECATED_settings/Routes.tsx:825-847`:

```tsx
        {$FREEE_DATA.ff.multiple_social_ins_2nd_enabled ? (
          <>
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

a) What happens when the flag is on and a user visits `#/health_insurance/2025/4`? (1 pt)
b) What is checked before the new route is registered, and where on the server is the same key defined? (1 pt)
c) If the URL is `#/company_social_insurance_settings?tab=welfare_pension_insurance`, what prop value does `InsuranceSettingsIndexContainer` get? (1 pt)
d) This file is in a `DEPRECATED_` folder. Is it wrong that new screens are routed here? Explain in one sentence. (1 pt)
