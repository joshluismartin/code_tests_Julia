# Lesson 32 — End-to-End Trace: From a Button Click to the Database and Back

**Time:** ~1 hour (40 min reading + practice, 20 min test)
**Goal:** Follow one real request through **every layer** of freee-payroll (React → generated API client → HTTP → Rails middleware → routes → controllers → repository/service → model/other service → JSON → React), and learn a repeatable method to find the code behind **any screen**.
**You need:** Lessons 1–31. The repo open in your editor. Keep a terminal in `~/Josh_Files/freee-payroll/`.

> This lesson has almost no new vocabulary. It **connects** everything you already know.
> Every file path below is real. Open each one as you read. That's the whole point.

---

## 1. The map we'll walk

```
 ┌───────────────────────────── BROWSER ─────────────────────────────┐
 │ ① React component calls   API.apiPEmployeeDisplayNamesIdGet(id)    │
 │ ② generated client builds  GET /api/p/employee_display_names/1     │
 │ ③ axios adds headers + CSRF token (configured-axios)               │
 └───────────────────────────────┬───────────────────────────────────┘
                                 │ HTTP
 ┌───────────────────────────────▼────────────── RAILS ──────────────┐
 │ ④ middleware: committee checks request/response against swagger    │
 │ ⑤ config/routes.rb → Api::Private::EmployeeDisplayNamesController  │
 │ ⑥ before_actions: logged in? allowed by permission.yaml?           │
 │ ⑦ controller#show → MemberRepository.find_one_by_employee_id       │
 │ ⑧ repository → ActiveRecord (MySQL) + nest-auth (gRPC)             │
 │ ⑨ returns HrMaster::Domain::Models::Member (value object)          │
 │ ⑩ render json: { id:, display_name: }                              │
 └───────────────────────────────┬───────────────────────────────────┘
                                 │ JSON
 ┌───────────────────────────────▼───────────────────────────────────┐
 │ ⑪ React: setEmployee(data) → re-render                             │
 └───────────────────────────────────────────────────────────────────┘
```

We'll do this trace twice:
- **Trace A**, a *read*: get an employee's display name.
- **Trace B**, a *write*: change the company's "daily prorate" rounding setting.

---

## 2. Trace A — Reading a display name (GET)

### ① The React caller

`front/javascripts/DEPRECATED_docs/components/Contracts/EditContract.tsx:795-798`

```tsx
  const loadEmployee = React.useCallback(async (employeeId: number) => {
    const { data } = await API.apiPEmployeeDisplayNamesIdGet(employeeId);
    setEmployee(data);
  }, []);
```

Read it line by line:
- `React.useCallback(async (employeeId: number) => {...}, [])`: a memoized async function (Lesson 28). The empty deps array means it is created once.
- `API` is `import * as API from '@freee-payroll/api'` (the Private API client package, Lesson 30).
- `await ...` returns an **axios response**. `{ data }` destructures its body.
- `setEmployee(data)` stores the JSON in state, so React re-renders (Lesson 27).

> The folder is named `DEPRECATED_docs` only because it uses old flux-style layout. That doesn't mean "dead code" (front/README.md: "過剰反応しないこと").

### ② The generated client

The name `apiPEmployeeDisplayNamesIdGet` is a mechanical translation: **path** `/api/p/employee_display_names/{id}` + **method** `GET`, in camelCase (docs/knowledge/general/coding-standards/schema-driven-development.md).

`front/packages/@freee-payroll/api/auto-generated/api.ts:203243-203245`

```ts
export const apiPEmployeeDisplayNamesIdGet = (id: number, options?: AxiosRequestConfig) => {
    return DefaultApiFp().apiPEmployeeDisplayNamesIdGet(id, options).then((request) => request());
}
```

Deeper inside the same file (`api.ts:138216-138220`) is where the URL is built:

```ts
        apiPEmployeeDisplayNamesIdGet: async (id: number, options: AxiosRequestConfig = {}): Promise<RequestArgs> => {
            // verify required parameter 'id' is not null or undefined
            assertParamExists('apiPEmployeeDisplayNamesIdGet', 'id', id)
            const localVarPath = `/api/p/employee_display_names/{id}`
                .replace(`{${"id"}}`, encodeURIComponent(String(id)));
```

The response **type** is also generated (`api.ts:7741-7754`):

```ts
export type ApiPEmployeeDisplayNamesIdGet200Response = {
    'id': number;
    'display_name': string;
}
```
*(comment lines removed for brevity)*

**Where do these come from?** From `app/controllers/api/private/employee_display_names_controller.swagger.yaml`, via `make front/packages/@freee-payroll/api/auto-generated` (Lesson 15). The top of api.ts says `Do not edit the class manually.` You never edit this file by hand. You edit the swagger and regenerate.

### ③ axios adds headers

The generated code imports `globalAxios` from `'axios'` (`api.ts:17`). The package `configured-axios` sets **global defaults** on that same axios object.

`front/packages/configured-axios/index.ts` (whole file, key lines):

```ts
const headers = {
  ...($FREEE_DATA.loginUser?.company_id != null
    ? { 'X-Company-Id': String($FREEE_DATA.loginUser.company_id) }
    : {}),
  ...
};
...
axios.defaults.headers.common = headers;
axios.defaults.xsrfCookieName = 'XSRF-TOKEN';
axios.defaults.xsrfHeaderName = 'X-CSRF-TOKEN';
```
*(slightly compressed; open the file to see the exact formatting)*

So every request carries:
- `X-Company-Id` / `X-User-Id`, read from `$FREEE_DATA`, the global object Rails wrote into the page (Lesson 16).
- The **CSRF token** (Cross-Site Request Forgery protection), copied from the `XSRF-TOKEN` cookie into the `X-CSRF-TOKEN` header. Rails rejects write requests without it.

What goes on the wire:

```
GET /api/p/employee_display_names/1
X-Company-Id: 10
X-CSRF-TOKEN: ...
Cookie: (session cookie)
```

### ④ Middleware: committee checks the schema

Before any controller runs, Rails passes the request through **middleware** (code that wraps every request). `config/application.rb:238-244`:

```ruby
    [
      {
        schema_path: 'app/controllers/api/private/swagger.yaml',
        accept_request_filter: ->(request) { request.path.start_with?('/api/p/') },
        error_handler: committee_error_handler,
        ignore_error: true,
      },
```

- For `/api/p/...` requests, **committee** validates the request params and later the response body against the combined swagger.
- `ignore_error: true` means a mismatch **does not block** the request. `committee_error_handler` (`config/application.rb:233-236`) calls `ReachableChecker.execute(message: ex)`. That **raises in dev/test** (you notice immediately) and **only notifies Bugsnag in production**.

### ⑤ Routing

`config/routes.rb`:

```ruby
8   namespace :api, defaults: { format: :json } do
...
34    scope 'p', module: :private do
...
236      resources :employee_display_names, only: [:show]
```

Put the pieces together:

| Piece | Contributes |
|---|---|
| `namespace :api` | URL `/api/...`, module `Api::` |
| `defaults: { format: :json }` | the request is treated as JSON (matters for error rendering later!) |
| `scope 'p', module: :private` | URL `/p/...`, module `Private::` (the URL says `p`, the code says `private`) |
| `resources :employee_display_names, only: [:show]` | `GET /employee_display_names/:id` → `#show` |

Result: `GET /api/p/employee_display_names/1` → `Api::Private::EmployeeDisplayNamesController#show` with `params[:id] == "1"` (a **String**; URL params are always strings).

### ⑥ before_actions: authenticated? authorized?

The controller inherits: `ApplicationController` → `Api::BaseController` → `Api::Private::BaseController` → our controller.

`app/controllers/application_controller.rb` registers filters that run **before every action**:

```ruby
57  before_action :check_authenticated
...
64  before_action :has_allowed_ac?
...
73  rescue_from StandardError, with: :handle_error
```

1. **`check_authenticated`** is abstract in ApplicationController (it raises `NotImplementedError`). `Api::Private::BaseController` overrides it (`app/controllers/api/private/base_controller.rb:27-30`):
   ```ruby
   def check_authenticated
     # ログイン状態かどうか
     render json: { message: 'タイムアウトしました。...' }, status: :unauthorized unless current_auth_user
   end
   ```
   Not logged in → **401** JSON, and the action never runs. (In Rails, rendering inside a before_action halts the chain.)

2. **`has_allowed_ac?`** (`application_controller.rb:138`) asks `Authz::EndpointProtection.can_access?(controller:, action:)`. That looks up **`config/permission.yaml`**:

   `config/permission.yaml:1650-1653`
   ```yaml
     - controller: api/private/employee_display_names
       action: show
       function_keys:
         - employee:contract:write
   ```
   The user's role must include the permission `employee:contract:write`. Otherwise it raises `HrErrors::UserError.new('アクセス権限がありません。', status: :forbidden)` → **403**. (That's why a new endpoint must be registered in permission.yaml: `app/controllers/api/private/README.md`.)

### ⑦ The controller action

`app/controllers/api/private/employee_display_names_controller.rb` (whole file):

```ruby
 1 # typed: true
 2 class Api::Private::EmployeeDisplayNamesController < Api::Private::BaseController
 3   include ErrorHandleable
 4
 5   freee_doorkeeper_for :show
 6
 7   def show
 8     member =
 9       ::HrMaster::Infrastructure::Repositories::MemberRepository.find_one_by_employee_id(
10         company_id: current_company.id,
11         employee_id: show_params[:id].to_i,
12       )
13
14     render json: { id: member.employee_id, display_name: member.display_name }
15   end
16
17   private
18
19   def show_params
20     params.permit(:id)
21   end
22 end
```

Notice:
- `include ErrorHandleable`: required for every controller (policy spec, `app/README.md`).
- `freee_doorkeeper_for :show` also allows OAuth-token access (used by the official mobile app; see `api/private/base_controller.rb:4-6`).
- `current_company.id`: the tenant always comes from the **session**, never from params. A user can't read another company's data just by changing a number.
- `show_params[:id].to_i`: strong params, then String → Integer, because the repository's `sig` demands `Integer`.
- It calls a **repository** directly, with no service. That's fine for a simple read with no business rule.
- The repository lives in `app/modules/hr_master/...` (layered architecture), while the controller stays in `app/controllers` (HTTP isn't a domain's property, per overview-backend.md).

### ⑧ The repository: DB + gRPC

`app/modules/hr_master/infrastructure/repositories/member_repository.rb:36-53`

```ruby
          sig do
            params(
              employee_id: Integer,
              company_id: Integer,
              include_deleted: T::Boolean,
              check_transaction: T::Boolean,
            ).returns(HrMaster::Domain::Models::Member)
          end
          def find_one_by_employee_id(employee_id:, company_id:, include_deleted: false, check_transaction: true)
            assert_transaction_not_open! if check_transaction

            employee = employee_scope(company_id:, include_deleted:).find_by(id: employee_id)
            # 従業員が見つからないのはユーザー起因（存在しない従業員を指定した）なので UserError。
            # employee はあるが member が無い場合はシステム起因（データ不整合）なので、この先で SystemError になる。
            raise HrErrors::UserError.new('従業員が見つかりませんでした') if employee.nil?

            find_member_via_nest_auth(employee:, company_id:)
          end
```

Everything you learned is here:
- Name `find_one_by_employee_id`: single result (`find_one`) plus filter (`_by_`). It passes `spec/repository_class_policies_spec.rb`.
- Class method, keyword args, Sorbet `sig` with `T::Boolean` defaults.
- `assert_transaction_not_open!` (line 386): this repository calls **another service over the network**, so it must not run inside a DB transaction.
- `employee_scope` (lines 187-193):
  ```ruby
  def employee_scope(company_id:, include_deleted:)
    Company
      .find(company_id)
      .employees
      .unscope_company_reference_date_rule_id
      .then { |employees| include_deleted ? employees.include_deleted : employees }
  end
  ```
  The query goes through `company.employees`, an association and not `Employee.where`, so it respects the Freee/QueryFromActiveRecordClass cop and tenancy. `unscope_company_reference_date_rule_id` removes Employee's second default_scope (Lesson 13). The soft-delete default_scope (`deleted_at = DEFAULT_DELETED_AT`) stays unless `include_deleted`.
- `find_by(id:)` returns `nil` when not found (unlike `find`, which raises). So the code raises `HrErrors::UserError` itself.

Then `find_member_via_nest_auth` (lines 243-270) asks **nest-auth**, the member microservice, over **gRPC**:

```ruby
            return HrMaster::Domain::Models::Member.new(employee_id: employee.id, is_deleted: true) if employee.deleted?

            nest_auth_member =
              if employee.member_id.present?
                req = Proto::NestAuth::GetMemberRequest.new(member_id: employee.member_id)
                begin
                  Omega::GRPC.client(:hr_master).get_member(req, timeout_sec: 5.0).member
                rescue Omega::GRPC::Errors::NotFound
                  nil
                end
              end
```

The display name isn't stored in our `employees` table. It lives in nest-auth. If nest-auth has no member for a live employee, the repository raises `HrErrors::SystemError` (data inconsistency, so a **500** that gets notified).

### ⑨ The domain model

The repository returns `HrMaster::Domain::Models::Member` (`app/modules/hr_master/domain/models/member.rb`): an immutable object with `attr_reader`s, `validates :display_name, presence: true, ...` (line 29) and `validate!` + `freeze` at the end of `initialize` (lines 63-64). It's a plain class that behaves like a value object. Deleted employees get `display_name` `'削除済みの従業員'`.

### ⑩ Rendering JSON and the swagger contract

`render json: { id: member.employee_id, display_name: member.display_name }` gives **200** and `{"id":1,"display_name":"給与太郎"}`.

The contract in `employee_display_names_controller.swagger.yaml` says `required: [id, display_name]`, with `id` an integer and `display_name` a string. On the way out, committee checks the response against it (step ④).

### ⑪ Back in React

`setEmployee(data)` → state changes → the component re-renders with the name. Done. 🎉

### The error path (same request, employee #999 doesn't exist)

```
repository: raise HrErrors::UserError.new('従業員が見つかりませんでした')
   ↓ (exception bubbles up through the controller)
ApplicationController: rescue_from StandardError, with: :handle_error
   ↓
ErrorHandleable#handle_error   (app/controllers/concerns/error_handleable.rb:8)
   case original_error when ::HrErrors::Base → keep as-is
   super(...) → ApplicationController#handle_error → Bugcap notify? (UserError: notify_error false → no)
   ↓
ErrorHandleable#render_error   (error_handleable.rb:55)
   format.json { render json: e.to_json_response, status: e.status }
   ↓
HTTP 400  { "message": "従業員が見つかりませんでした", "code": null, "fields": [] }  (approximately)
   ↓
React: the await throws → catch (e) → API.getErrorMessage(e) → notify({ message, type: 'error' })
```

Two details worth noticing:
- `HrErrors::UserError` defaults to `status: :bad_request` (`lib/hr_errors/user_error.rb:16`), so this is **400** unless the raiser passes `status: :not_found`.
- `to_json_response` is `{ message:, code:, fields: }` (`lib/hr_errors/base.rb:9-11`). On the frontend, `isHRError` / `getErrorMessage` in `front/packages/@freee-payroll/api/src/errorHandler.ts` know exactly this shape. **Backend and frontend agree on one error format.**

### The spec that protects it

`spec/controllers/api/private/employee_display_names_controller_spec.rb` (Lesson 21):

```ruby
describe Api::Private::EmployeeDisplayNamesController, type: :request do
  let(:company) { default_company }
  before { log_in_as_an_admin_user }

  describe '#show' do
    subject { get '/api/p/employee_display_names/1', as: :json }
    ...
    context 'when employee is not found' do
      before do
        allow(::HrMaster::Infrastructure::Repositories::MemberRepository).to receive(
          :find_one_by_employee_id,
        ).and_raise(::HrErrors::UserError.new('従業員が見つかりませんでした。', status: :not_found))
      end

      it do
        is_expected.to eq 404
```

- It's a **request spec** living in `spec/controllers/` (the repo rule).
- It stubs the repository, so no gRPC call happens in the test.
- The error test stubs a UserError **with** `status: :not_found`, so it expects 404. The real repository raises without a status (400). The spec tests the *controller's* handling of whatever the repository raises, not the repository itself. Knowing this difference is exactly the kind of careful reading this lesson is about.

---

## 3. Trace B — Writing a setting (PUT)

**Screen:** 設定 → 日割り計算設定 (daily prorate: how to round when salary is split by days).

### Frontend

`front/javascripts/DEPRECATED_settings/components/CompanyDailyProrate.tsx:191-207`

```tsx
  const onSubmit = async () => {
    setIsLoading(true);
    try {
      const res = await API.apiPCompanyDailyProrateSettingPut({
        round_type: roundType,
      });
      setRoundType(res.data.round_type);
      setIsChanged(false);
      notify({
        message: '日割り計算設定を更新しました。',
        type: 'success',
      });
    } catch (e) {
      notify({ message: API.getErrorMessage(e), type: 'error' });
    }
    setIsLoading(false);
  };
```

Loading flag on, then try the API, then success toast or error toast, then loading flag off. The same file's `useEffect` (lines 177-189) loads the current value with `API.apiPCompanyDailyProrateSettingGet()` on mount. It's the classic Container pattern: `CompanyDailyProrateContainer` (line 167) owns state and API calls, and `CompanyDailyProrate` only renders.

Request: `PUT /api/p/company_daily_prorate_setting` with body `{"round_type":"round"}`.

### Route

`config/routes.rb:424`

```ruby
      resource :company_daily_prorate_setting, only: %i[show update]
```

Singular **`resource`** means there's no `:id` in the URL: one setting per company. `PUT` (or PATCH) → `#update`, and the controller name is still plural: `CompanyDailyProrateSettingsController`.

### Permission

`config/permission.yaml:1441-1444`

```yaml
  - controller: api/private/company_daily_prorate_settings
    action: update
    function_keys:
      - settings:payroll:write
```

The frontend *also* checks the same key before showing the screen (`front/javascripts/DEPRECATED_settings/Routes.tsx:803`: `hasPermission({ functionKeys: ['settings:payroll:write'] })`). **The frontend check is only for UX. The backend check is the real security.**

### Controller

`app/controllers/api/private/company_daily_prorate_settings_controller.rb:12-25`

```ruby
  def update
    permitted_params = params.permit(:round_type)

    # 設定変更時に全従業員の給与再計算が実行されるが、管理対象外の従業員も含めて処理する必要があるため、部門制御を無効化する
    company_daily_prorate_setting =
      Authz.disable_access_control do
        ::Payroll::Application::Services::UpdateCompanyDailyProrateSettingService.execute(
          company: current_company,
          round_type: permitted_params[:round_type],
        )
      end

    render json: build_response(company_daily_prorate_setting: company_daily_prorate_setting), status: :ok
  end
```

`Authz.disable_access_control { }`: a department manager (部門管理者) may only see some employees. Changing a company-wide setting must recalculate **everyone**, so access control is turned off *inside the block only*, and the comment explains why. That's the rule from `app/README.md`: when you intentionally handle all employees, wrap the code and write the reason.

### Application service (use case)

`app/modules/payroll/application/services/update_company_daily_prorate_setting_service.rb:13-34`

```ruby
          def execute(company:, round_type:)
            current_company_daily_prorate_setting =
              ::Payroll::Infrastructure::Repositories::CompanyDailyProrateSettingRepository.find_one_by_company_id(
                company_id: company.id,
              )

            company_daily_prorate_setting =
              ::Payroll::Domain::Models::CompanyDailyProrateSetting.new(company_id: company.id, round_type: round_type)
            if current_company_daily_prorate_setting != company_daily_prorate_setting
              ActiveRecord::Base.transaction do
                ::Payroll::Infrastructure::Repositories::CompanyDailyProrateSettingRepository.save(
                  company_daily_prorate_setting: company_daily_prorate_setting,
                )

                # 賃金計算期間のステータスが未確定の給与明細に対して給与計算を実行する
                cps_ids = company.company_payroll_statements.variable.pluck(:id)
                ::Application::CalcPayrollStatementsService.new(:none, company.id, cps_id: cps_ids).execute
              end
            end

            company_daily_prorate_setting
          end
```

Walk through it:
1. Load the current setting as a **domain model** via the repository.
2. Build the new value object. `CompanyDailyProrateSetting < T::Struct` + `ActsAsValueObject` validates `round_type` is one of `%w[floor round ceil]` on `new` (so a bad value raises `ActiveModel::ValidationError` here, before anything is saved).
3. `!=` compares **by value** (ActsAsValueObject includes `T::Struct::ActsAsComparable`), which is value-object equality from Lesson 17. If nothing changed, it does nothing. That's cheap.
4. `ActiveRecord::Base.transaction do`: save **and** recalculate, all-or-nothing. If recalculation fails, the save rolls back.
5. `pluck(:id)`: ids only, memory-friendly (Lesson 13).
6. `CalcPayrollStatementsService.new(...).execute` is a legacy-style service (instance `execute`, on the policy ignore list). Old and new styles meet in one method. Normal here.
7. It returns the value object, and the controller turns it into `{ round_type: ... }`.

(The file header marks it `@deprecated` because it takes an ActiveRecord `company` instead of `company_id:`. You'll see such notes. They tell you the *preferred* direction.)

### Repository → ActiveRecord

`app/modules/payroll/infrastructure/repositories/company_daily_prorate_setting_repository.rb` `save`:

```ruby
          def save(company_daily_prorate_setting:)
            company = Company.find(company_daily_prorate_setting.company_id)
            current_setting = company.company_daily_prorate_setting
            if current_setting.nil?
              company.create_company_daily_prorate_setting!(round_type: company_daily_prorate_setting.round_type)
            else
              current_setting.update!(round_type: company_daily_prorate_setting.round_type)
            end
          end
```

Only the **infrastructure** layer touches ActiveRecord (`app/models/company_daily_prorate_setting.rb`, with `enum :round_type ... prefix: true` and `validates :round_type, presence: true`). `create_..._setting!` comes from `has_one`. `update!` raises on failure (bang → exception → ErrorHandleable turns `RecordInvalid` into a 400 with field errors).

### The whole Trace B in one picture

```
CompanyDailyProrateContainer.onSubmit
  → API.apiPCompanyDailyProrateSettingPut({ round_type })
  → PUT /api/p/company_daily_prorate_setting
  → routes.rb:424 resource … only: [:show, :update]
  → before_actions (auth, permission.yaml: settings:payroll:write)
  → CompanyDailyProrateSettingsController#update (permit, disable_access_control)
  → Payroll::Application::Services::UpdateCompanyDailyProrateSettingService.execute
      → Repository.find_one_by_company_id → Domain value object
      → compare (!=) → transaction { Repository.save → AR update!; recalc }
  → render json: { round_type: } 200
  → setRoundType, notify success
```

---

## 4. How to find the code for ANY screen

### 4.1 Forward: from a browser URL to the code

Example: you're on **`https://…/holidays#/special_holidays`** (休暇管理 › 特別休暇).

| Step | Look at | You find |
|---|---|---|
| 1. Split the URL | before `#` = **Rails** path `/holidays`; after `#` = **React router** path `/special_holidays` (hashHistory) | |
| 2. Rails route | `grep -n "holidays" config/routes/web.rb` | `config/routes/web.rb:46` `get 'holidays', to: 'web/holidays#index'` |
| 3. Web controller | `app/controllers/web/holidays_controller.rb` | `render template: 'web/app'`, i.e. only `<div id="app"></div>` |
| 4. Which JS bundle? | `JS_FILE_MAP` in `app/helpers/application_helper.rb:27` | `'web/holidays' => 'front/javascripts/holidays/index.ts'` (no match → `legacy_application` fallback, lines 76-86) |
| 5. Entry | `front/javascripts/holidays/index.ts` | `run({ withReactRouter: Routes })` |
| 6. React routes | `front/javascripts/holidays/Routes.tsx` + `constants.tsx:10-14` (`HOLIDAY_ROUTE_MAP.special_holidays = '/special_holidays'`) | `<Route component={CompanySpecialHolidaysContainer} path={HOLIDAY_ROUTE_MAP.special_holidays} />` (when a flag is on) |
| 7. Container | `components/CompanySpecialHolidays.tsx` → `...Container` | the `API.apiP...` calls |
| 8. API → backend | function name → path → `config/routes.rb` → `Api::Private::...Controller` | Trace A steps ⑤–⑩ |

Watch out for **feature flags** and **permissions** in `Routes.tsx`. The same path can render different containers (e.g. `$FREEE_DATA.ff.shinra_uiux_kaizen_paid_holiday_enabled` in `holidays/Routes.tsx`). Check which branch is active for your company.

Another example: **`/settings#/company_daily_prorate`** → `config/routes/web.rb:52` `get 'settings', to: 'web/settings#index'` → `Web::SettingsController` → `JS_FILE_MAP 'web/settings' => 'front/javascripts/DEPRECATED_settings/index.ts'` (line 47) → `DEPRECATED_settings/Routes.tsx:803-812` → `CompanyDailyProrateContainer`. This is Trace B.

### 4.2 Reverse: from an API (or a controller) to the React callers

1. Controller + action → URL → generated name. `PUT /api/p/company_daily_prorate_setting` → `apiPCompanyDailyProrateSettingPut`.
2. Search, excluding the huge generated file:
   ```bash
   grep -rn "apiPCompanyDailyProrateSettingPut" front --include='*.ts' --include='*.tsx' | grep -v auto-generated
   ```
   → `front/javascripts/DEPRECATED_settings/components/CompanyDailyProrate.tsx:194`
3. Also search the raw path, because some code calls `configured-axios` directly (e.g. `axios.get('/api/p/special_holiday_settings')` in `front/javascripts/holidays/ducks/importHolidays.ts`):
   ```bash
   grep -rn "/api/p/company_daily_prorate_setting" front | grep -v auto-generated
   ```
4. Also search the legacy CoffeeScript (`front/packages/@legacy`) for `url:` strings when nothing else matches.

### 4.3 Tools that help

| Tool | Use |
|---|---|
| `bin/rails routes -g employee_display` | Lists matching routes (Prefix, Verb, URI, Controller#Action). Slow (boots Rails). |
| `grep -n "resource" config/routes.rb config/routes/*.rb` | Fast; often enough |
| **find-route skill** (`.claude/skills/find-route/SKILL.md`) | In Claude Code: `/find-route GET /api/p/companies`. The skill: (1) parses method + path, treating real ids like `/employees/2065116/2026/1` as params; (2) **greps routes.rb first** (faster than `rails routes`); (3) uses `bundle exec rails routes -g` only if needed; (4) converts `api/private/companies#index` → `Api::Private::CompaniesController#index` and confirms the class by opening the file. It also finds the swagger file. It notes that `api/p/` is short for `api/private/`. |
| Browser DevTools → **Network** tab | Click the button and watch: method, URL, status, request payload, response JSON. **The fastest way to learn which API a screen calls.** |
| DevTools → Elements → search `data-test="..."` | Components add `data-test` labels (e.g. `data-test="姓"`), so grep that string in `front/` to find the component |
| `permission.yaml` grep by controller | Who is allowed to call it |

---

## 5. Practice (15 min)

**Practice 1: trace a read on your own.** Pick `GET /api/p/comments` (Lesson 14–15).
Write down: (a) the route line in `config/routes.rb`, (b) the controller file + action, (c) the repository method it calls, (d) the permission.yaml function key, (e) the generated TS function name, (f) one frontend caller (grep it).

**Practice 2: forward search.** Open freee人事労務 (staging or local) on any screen. From the URL alone, find the Rails web controller and the `JS_FILE_MAP` entry. Then use the Network tab to find one `/api/p/` call and its controller.

**Practice 3: explain a status.** For Trace A, what status does the browser get if (a) the session expired, (b) the user lacks `employee:contract:write`, (c) employee #999 doesn't exist (real repository), (d) nest-auth has no member for a live employee?

*(Answers for Practice 3: (a) 401 from `check_authenticated`; (b) 403 from `has_allowed_ac?`; (c) 400, UserError default `:bad_request`; (d) 500, SystemError, notified.)*

---

## 6. Summary

- A request passes: **React → generated client → configured-axios → committee → routes → before_actions (auth, permission.yaml) → controller → repository/service → models / gRPC → JSON → React**.
- The **swagger file** is the contract. It generates the TS client *and* is checked at runtime by committee.
- The **tenant** comes from `current_company` (session), never from params.
- Errors become `HrErrors` → ErrorHandleable → `{message, code, fields}` → `API.getErrorMessage` → toast.
- Writes: controller → **application service** → value object compare → **transaction** → repository → ActiveRecord.
- To find a screen's code: URL before `#` → `routes/web.rb` → `JS_FILE_MAP`; after `#` → feature `Routes.tsx` → Container → API name → `routes.rb` → controller. To go in reverse, grep the generated function name (excluding `auto-generated`).

➡️ Now take **Test32.md**.
