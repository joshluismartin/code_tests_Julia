# Test 32 — End-to-End Trace

**Rules:** Don't look at Lesson32.md while answering. You MAY open the repo for Part F (that's realistic). Then check `Answers32.md`.
**Total:** 33 points

---

## Part A — Multiple choice (1 point each)

**1.** In `GET /api/p/employee_display_names/1`, which Rails route pieces produce the controller name `Api::Private::EmployeeDisplayNamesController`?
- a) `namespace :api` + `scope 'p', module: :private` + `resources :employee_display_names`
- b) `draw :web` + `resources :employee_display_names`
- c) `namespace :private` + `get 'p'`
- d) The generated TypeScript client decides it

**2.** What does `configured-axios` add to requests?
- a) The swagger schema
- b) `X-Company-Id` / `X-User-Id` headers and the CSRF token header
- c) The user's password
- d) Nothing; it only logs requests

**3.** For `/api/p/...`, committee is configured with `ignore_error: true`. What happens in production when a response doesn't match the swagger?
- a) The request fails with 500
- b) The response is still sent; the mismatch is reported through ReachableChecker (Bugsnag)
- c) The swagger file is updated automatically
- d) The frontend crashes

**4.** Which file decides whether a logged-in user's role may call `api/private/employee_display_names#show`?
- a) `config/routes.rb`
- b) `config/permission.yaml`
- c) `front/@types/freee-data.d.ts`
- d) `app/views/web/app.html.erb`

**5.** `MemberRepository.find_one_by_employee_id` raises `HrErrors::UserError.new('従業員が見つかりませんでした')` with no `status:`. What HTTP status does the client receive?
- a) 404
- b) 400
- c) 500
- d) 200

**6.** Why does `MemberRepository` call `assert_transaction_not_open!`?
- a) Transactions are forbidden in freee-payroll
- b) It makes a gRPC call to nest-auth, which should not happen inside a DB transaction
- c) To make the query faster
- d) Because it writes to the database

**7.** In the URL `/holidays#/special_holidays`, which part is handled by **React Router** (hashHistory)?
- a) `/holidays`
- b) `/special_holidays`
- c) Both
- d) Neither

**8.** In `UpdateCompanyDailyProrateSettingService`, why is `Authz.disable_access_control { }` used in the controller?
- a) To skip login
- b) A company-wide setting change must recalculate payroll for all employees, including ones outside a department manager's scope
- c) To make the request faster
- d) To bypass permission.yaml

---

## Part B — Fill in the blanks (1 point each)

**9.** The generated function for `PUT /api/p/company_daily_prorate_setting` is named `__________`.

**10.** The combined Private API schema that committee reads is `app/controllers/api/private/__________`.

**11.** A Rails web page for a React screen renders the template `'web/app'`, which contains only `<div id="______"></div>`.

**12.** The Ruby hash that maps `'web/holidays'` to `'front/javascripts/holidays/index.ts'` is `__________` in `app/helpers/application_helper.rb`.

**13.** When no mapping is found for a web controller, the JS bundle falls back to `front/javascripts/__________/index.ts`.

**14.** The JSON body produced by `HrErrors::Base#to_json_response` has the three keys `message`, `______` and `______`.

---

## Part C — Put in order (4 points)

**15.** Order these steps of Trace B (write flow) from first to last:

- A. `ActiveRecord::Base.transaction do` → repository `save` → `update!`
- B. `API.apiPCompanyDailyProrateSettingPut({ round_type })`
- C. `has_allowed_ac?` checks `settings:payroll:write`
- D. `render json: { round_type: ... }`
- E. `UpdateCompanyDailyProrateSettingService.execute(company:, round_type:)`
- F. `params.permit(:round_type)`
- G. `setRoundType(res.data.round_type)` + success toast
- H. value object `!=` comparison

---

## Part D — Write it (3 points each)

**16.** Write the terminal command(s) to find **every frontend caller** of `GET /api/p/employee_display_names/{id}`, excluding the generated client.

**17.** A new endpoint `GET /api/p/employee_nicknames/:id` is needed. List, in order, the **files you must create or edit** (backend + frontend) following freee-payroll's rules. (At least 6 items.)

---

## Part E — Answer in a sentence or two (2 points each)

**18.** The frontend already hides the 日割り計算設定 screen with `hasPermission({ functionKeys: ['settings:payroll:write'] })`. Why is the `permission.yaml` entry still necessary?

**19.** Why does the controller use `current_company.id` instead of accepting a `company_id` from the request params?

**20.** The request spec stubs the repository to raise `UserError(..., status: :not_found)` and expects 404, but the real repository raises without a status. Is the spec wrong? Explain what the spec is really testing.

---

## Part F — Read real code (3 points)

**21.** Open `front/javascripts/DEPRECATED_settings/components/CompanyDailyProrate.tsx` lines 177–189. Explain (a) when this code runs, (b) which endpoint it calls, and (c) what happens on success and on failure.
