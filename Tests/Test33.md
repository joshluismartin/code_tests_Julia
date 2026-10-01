# Test 33 — FINAL EXAM (Whole Course)

**Rules:** Closed book for Parts A–D. For Part F you may open the repo files named (reading real code is the skill being tested). Take your time, about 60 minutes. Check `Answers33.md`.
**Total:** 40 points. **Pass = 32+.** If you pass, you can read any part of freee-payroll. 🎓

> First do the **Lesson 33 drills** (their answers are in Answers33.md Part 0). They are not scored.

---

## Part A — Multiple choice: terminology (1 point each)

**1.** In freee-payroll, which condition means a record is **not** deleted?
- a) `deleted_at IS NULL`
- b) `deleted_at = '1000-01-01 00:00:00'` (`DEFAULT_DELETED_AT`)
- c) `is_deleted = false`
- d) The row doesn't exist

**2.** A freee-payroll **Service** class (new style) should expose:
- a) Many public instance methods
- b) Only a public class method `execute`
- c) Only `initialize`
- d) `find_one_by_*` methods

**3.** Which repository method name **violates** the naming policy?
- a) `find_one_by_employee_id`
- b) `find_list_at_pay_year_month`
- c) `find_all`
- d) `find_employees_with_company`

**4.** In Sorbet, `T.nilable(String)` means:
- a) Always a String
- b) A String or `nil`
- c) Any type
- d) An array of strings

**5.** In RSpec, the difference between `let` and `let!` is:
- a) `let!` is lazy, `let` is eager
- b) `let` is evaluated only when first used; `let!` is evaluated before each example
- c) There is no difference
- d) `let!` can only be used in request specs

**6.** In TypeScript, `unknown` differs from `any` because:
- a) `unknown` must be narrowed before you can use it; `any` turns type checking off
- b) They are identical
- c) `unknown` is only for numbers
- d) `any` is safer

**7.** In React, what triggers a component to re-render?
- a) Changing a local variable
- b) Calling a state setter (e.g. `setCount`) with a new value, or receiving new props
- c) Writing `console.log`
- d) Importing a module

**8.** Which is the **recommended** way for new frontend code to call a Private API endpoint?
- a) `fetch('/api/p/...')`
- b) The generated client from `@freee-payroll/api` (e.g. `API.apiPXxxGet`)
- c) jQuery `$.ajax`
- d) Backbone models

---

## Part B — Fill in the blanks (1 point each)

**9.** To run an intentionally all-employee operation, wrap it in `Authz.__________ { }` and write a comment explaining why.

**10.** Heavy read-only queries go to the read replica with `DatabaseSelector.__________ { }`.

**11.** Every controller must include `__________` (or `Omega::Controller::ErrorHandler`).

**12.** In specs, get the seeded company with `__________` instead of `create(:company)`.

**13.** *(2 points, 1 per blank)* The Rails feature-flag call for freee-feature-flag is `Omega::FeatureFlag.__________('flag_name', namespace: :payroll)`; on the frontend, flags live in `$FREEE_DATA.______`.

**14.** New asynchronous jobs go in `app/______/` using ActiveJob (not `app/resque_jobs/`).

---

## Part C — Predict the output (1 point each)

**15.** Ruby:
```ruby
x = nil
p x&.length
y ||= 5
p y
p %w[a b c].each_slice(2).to_a
p [1, 2, 3, 4].select(&:even?).sum
```

**16.** JavaScript/TypeScript:
```ts
const a = [{ id: 1, d: false }, { id: 2, d: true }];
console.log(a.find((s) => s.d)?.id);
console.log([][0]?.name ?? '－');   // imagine the types allow it
console.log(0 || 'x', 0 ?? 'x');
```

---

## Part D — Write code in freee style (4 points each)

**17.** Write a **repository** in the layered architecture that returns the ids of all of a company's employees that are not soft-deleted, **including** employees with no `company_reference_date_rule_id` (締め日支払い日 not set).
Requirements: file path + correct namespace for the `employee_master` domain's infrastructure layer, `class << self`, `extend T::Sig`, a `sig`, a policy-compliant method name, keyword args, go through `Company.find(...).employees` (not `Employee.where`), the right scope to include employees without a CRDR, and `pluck`.

**18.** Write a **React Container + presentational component pair** in TypeScript:
- `EmployeeNameLabel` (default export) takes `Props = { displayName: string }` and renders it inside a Vibes `<Text>`.
- `EmployeeNameLabelContainer` (named export) takes `{ employeeId: number }`, loads the name with `API.apiPEmployeeDisplayNamesIdGet(employeeId)` in `React.useEffect`, keeps it in state, shows an error toast with `useNotification()` + `API.getErrorMessage(e)` on failure, and renders `EmployeeNameLabel`.

---

## Part E — Answer in a sentence or two (2 points each)

**19.** Explain the difference between the **layered architecture** in `app/modules/` and the **Rails-extended** architecture in the rest of `app/`. Where is ActiveRecord allowed in the layered one?

**20.** Why does freee-payroll write the OpenAPI `*.swagger.yaml` **before** the controller? Give two benefits.

**21.** In a request spec, why use `allow(Repo).to receive(:find_one_by_x).and_return(...)` and then `expect(Repo).to have_received(:find_one_by_x).with(...)`, instead of calling the real repository?

---

## Part F — Read real code (3 points each)

**22.** `app/models/employee_time_clock.rb` lines 1–12 and 36–45. Explain: (a) the `default_scope`, (b) why `self.inheritance_column = nil` is needed, (c) what `EmployeeTimeClock.start_date(nil)` returns and why (look at the `if date.present?` inside the lambda).

**23.** `app/modules/payroll/infrastructure/repositories/company_allowance_repository.rb` lines 113–124 (`find_list_by_company_id_for_display_order_update_candidate_ids`). Explain: (a) what it returns, (b) what the feature-flag `unless` does, (c) what must happen to this code when the flag is deleted.

**24.** `front/javascripts/home/components/Home/JoiningEmployeesPanel.tsx` (whole file). Explain: (a) the split between `JoiningEmployeesPanel` and `JoiningEmployeesPanelContainer`, (b) why the async function is wrapped as `(async () => { ... })()` inside `useEffect`, (c) when the panel is rendered at all.
