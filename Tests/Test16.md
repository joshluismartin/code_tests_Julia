# Test 16 — Views, Helpers, and How Rails Hands Off to React

**Rules:** Don't look at Lesson16.md. Write your answers under each question, then check `Answers16.md`.
**Total:** 40 points

---

## Part A — Multiple choice (1 point each)

**1.** Which ERB tag runs Ruby **and prints** the result?
- a) `<% %>`
- b) `<%= %>`
- c) `<%# %>`
- d) `<%- -%>`

**2.** A *partial* is:
- a) A half-finished migration
- b) A small reusable template whose filename starts with `_`
- c) A JavaScript file
- d) A database view

**3.** In a layout, `<%= yield %>` means:
- a) Stop rendering
- b) Insert the current page's own view here
- c) Load JavaScript
- d) Print the user's name

**4.** A *helper* is:
- a) A method you can call from views, usually defined in `app/helpers/`
- b) A test
- c) A background job
- d) A database table

**5.** What does `app/views/web/app.html.erb` contain?
- a) The full HTML of the holidays screen
- b) Only `<div id="app"></div>` (plus a comment)
- c) All the CSS
- d) The login form

**6.** What is the fallback JS entrypoint when a web controller isn't in `JS_FILE_MAP` (and isn't admin)?
- a) `front/javascripts/home/index.ts`
- b) `front/javascripts/legacy_application/index.ts`
- c) Nothing is loaded
- d) An error page

**7.** `window.$FREEE_DATA` is:
- a) Fetched by React from `/api/p/freee_data`
- b) A global JS object written into the page's HTML by Rails when rendering the layout
- c) Stored in the browser's cookies
- d) A MySQL table

**8.** Which feature-flag mechanism is the **first choice** according to the team docs?
- a) Environment variables
- b) sysparam
- c) freee-feature-flag (`Omega::FeatureFlag.active?`)
- d) Comments in code

**9.** Which feature-flag mechanism reflects a change **instantly** (because it's read from the DB)?
- a) freee-feature-flag
- b) sysparam (`current_company.xxx_enabled?`)
- c) Environment variable
- d) None

**10.** *SPA* stands for:
- a) Server Page Application
- b) Single-Page Application
- c) Simple Program API
- d) Styled Page Asset

---

## Part B — Fill in the blanks (1 point each)

**11.** `$FREEE_DATA` is created in the partial `app/views/layouts/______.html.erb`.

**12.** The hash mapping controllers to JS entrypoints is `______` in `app/helpers/______.rb`.

**13.** Feature flags reach React under the key `$FREEE_DATA.______`.

**14.** The helper method that builds that key is `______` in `app/helpers/js_exports_helper.rb`.

**15.** The CI job that runs specs with all release flags turned on is `______`.

**16.** To create a new sysparam release flag, run `rails generate ______ HogeHogeEnabled "Activate HogeHoge"`.

---

## Part C — Fill in the missing code (2 points each)

**17.** Print a partial and a variable in ERB:

```erb
<______ render 'layouts/footer' %>
<p><______ @title %></p>
```

**18.** Read a freee-feature-flag named `new_export_enabled` in the payroll namespace:

```ruby
if Omega::FeatureFlag.______('new_export_enabled', namespace: ______)
  ...
end
```

**19.** Complete `js_file_path`'s lookup order:

```ruby
inherent_file_path =
  JS_FILE_MAP["#{params[:controller]}______#{params[:action]}"] || JS_FILE_MAP["#{params[______]}"]
```

**20.** In React, following the team rule, check the flag `nencho_plan_enabled`:

```tsx
if (______) {
  // show the 年末調整 plan banner
}
```

---

## Part D — Write the code (3 points each)

**21.** (illustrative) A new page `GET /reports2` should show a React app from `front/javascripts/reports2/index.ts`. Write: (a) the route line in `config/routes/web.rb`, (b) the controller, (c) the line to add to `JS_FILE_MAP`.

**22.** (illustrative) A service computes a CSV row limit. When the sysparam flag `bigger_csv_limit_enabled?` is on for the company, the limit is 10,000; otherwise 3,000. Only this one value differs. Write the branch the way the team guidance recommends.

---

## Part E — Answer in a sentence or two (2 points each)

**23.** Why do so many routes in `config/routes/web.rb` end with `(/*path)`?

**24.** `front/CLAUDE.md` says not to write `const fooEnabled = $FREEE_DATA.ff.foo_enabled`. Why?

**25.** Why should you avoid checking a sysparam flag (`current_company.xxx_enabled?`) inside a loop over thousands of employees?

---

## Part F — Read real code (4 points)

**26.** Explain what happens, step by step, when a logged-in user opens `https://…/holidays` in the browser, until they see the screen. Mention: the route file, the controller and what it renders, the layout parts (`$FREEE_DATA`, JS tag, `yield`), `JS_FILE_MAP`, and what the entrypoint does. (1 point per two correct steps, up to 4.)
