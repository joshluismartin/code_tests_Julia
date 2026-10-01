# Lesson 15 — Private API, OpenAPI, and Error Handling

**Time:** ~1 hour (40 min reading + practice, 20 min test)
**Goal:** Understand how freee-payroll builds a Private API endpoint the "schema-first" way, how errors become JSON, and trace one real endpoint all the way from the React button to the database and back.
**You need:** Lesson 14 (routes, controllers, params). Open `~/Josh_Files/freee-payroll/` in your editor.

---

## 1. New terms

| Term | Meaning in plain words |
|---|---|
| **Private API** | URLs under `/api/p/` that only freee-payroll's **own** frontend calls. |
| **Schema** | A precise description of the *shape* of data: which fields, which types, which are required. |
| **OpenAPI** (a.k.a. **Swagger**) | A standard format (written in YAML) for describing an API: paths, methods, parameters, request and response schemas. "Swagger" is its older name, and freee-payroll files still use it: `*.swagger.yaml`. |
| **YAML** | A text format for data that uses indentation instead of `{ }`. `key: value`, and `- item` for lists. |
| **Schema-driven development** / **schema-first** | Write the schema **before** the code. The schema is the "contract" both sides follow. |
| **Single Source of Truth (SSOT)** | The one place that is considered correct. Here: the schema is the SSOT for what the API does. |
| **Code generation** | A tool writes code for you from the schema. freee-payroll generates the TypeScript API client. |
| **Runtime validation** | While the app runs, each real request/response is checked against the schema. |
| **committee** | The Ruby gem that does this runtime validation. |
| **Middleware** | Code that sits between the web server and your controller and sees every request/response. committee is middleware. |
| **Exception / error** | (Lesson 7) An object `raise`d when something goes wrong. It stops normal flow until something `rescue`s it. |
| **Error monitoring** | A service that collects crashes from production so engineers are alerted. freee uses **Bugsnag**, called through `Omega::Bugcap.notify`. |
| **TypeSpec** (`.tsp`) | A language for writing API schemas more compactly. It compiles to OpenAPI. Used for the **Domain API**. |

---

## 2. The three API "families"

From `docs/knowledge/general/coding-standards/schema-driven-development.md`:

| Family | URL | Combined schema file | Schema written in |
|---|---|---|---|
| **Private API** | `/api/p/` | `app/controllers/api/private/swagger.yaml` | OpenAPI YAML, one file per controller |
| **Domain API** | `/api/internal/` (only the `domain/` controllers) | `app/controllers/api/internal/domain/swagger.yaml` | TypeSpec (`.tsp`), compiled to YAML |
| **Public API** | `/api/v1` | `docs/api/public/swagger.yaml` | (separate process; out of scope) |

As a new developer you'll mostly touch the **Private API**. This lesson focuses on it.

---

## 3. Rule #1: schema first, next to the controller

The team's rules (`app/controllers/api/private/README.md`, translated):

1. **Schema-driven development:** for a new API, write `<controller名>.swagger.yaml` **before** the controller, in the **same folder** as the controller. The placement is checked in CI by `scripts/check-api-schema-implementation-rules.js`. After changing a schema, regenerate the TypeScript client with `make front/packages/@freee-payroll/api/auto-generated`.
2. **New endpoints must be registered** in `config/permission.yaml` (`endpoint_protection`). Don't edit `config/sekisyo.yml` by hand; it's generated.
3. **Tests** are written as `type: :request` specs, but in the `spec/controllers/` folder.

### 3.1 A real schema file

The endpoint we will trace is **"get an employee's display name."** Its controller and schema sit side by side:

```
app/controllers/api/private/employee_display_names_controller.rb
app/controllers/api/private/employee_display_names_controller.swagger.yaml
```

```yaml
# app/controllers/api/private/employee_display_names_controller.swagger.yaml (full)
openapi: 3.0.3
info:
  title: freee-payroll
  version: 1.0.0
paths:
  '/api/p/employee_display_names/{id}':
    get:
      parameters:
        - name: id
          in: path
          required: true
          schema:
            type: integer
            nullable: false
      responses:
        '200':
          description: OK
          content:
            application/json:
              schema:
                type: object
                required:
                  - id
                  - display_name
                properties:
                  id:
                    type: integer
                  display_name:
                    type: string
```

Reading it:

| YAML part | Meaning |
|---|---|
| `paths:` | List of URLs this file describes |
| `'/api/p/employee_display_names/{id}'` | The URL. `{id}` is a path parameter (like `:id` in Rails routes) |
| `get:` | The HTTP method |
| `parameters: - name: id, in: path, required: true, type: integer` | There's a required integer `id` in the path |
| `responses: '200':` | What a successful response looks like |
| `content: application/json: schema:` | The body is JSON with this shape |
| `type: object`, `properties:` | A JSON object with fields `id` (integer) and `display_name` (string) |
| `required: [id, display_name]` | Both fields must always be present |

Other things you'll see in schemas:

| Keyword | Meaning | Example (from `comments_controller.swagger.yaml`) |
|---|---|---|
| `in: query` | A `?name=value` parameter | `- name: url` / `in: query` |
| `requestBody:` | The JSON the client sends (POST/PUT/PATCH) | `$ref: '#/components/schemas/CommentRequestBody'` |
| `components: schemas:` | Named, reusable shapes **inside the same file** | `Comment:`, `CommentRequestBody:` |
| `$ref: '#/components/schemas/X'` | "Use the shape named X" | |
| `nullable: true` | The value may be `null` | `page_name` |
| `enum: [floor, round, ceil]` | Only these values are allowed | `company_daily_prorate_settings_controller.swagger.yaml` |
| `format: date-time` | A string in ISO date-time format | `created_at` |
| `type: array`, `items:` | A list whose elements have this shape | |

### 3.2 Schema rules you must follow (Private API)

From `schema-driven-development.md`:

- ✅ Define **only this controller's** things: path, method, request parameters/body, and the **success** response.
- ❌ **Don't** define error responses in each controller's schema, and don't write custom error handling for them either. Error format and handling are **shared across the whole project** (section 6). Some old files still have error definitions; ignore them.
- ❌ **Don't** `$ref` a schema from **another controller's** swagger file. It creates hidden dependencies between controllers. (Some old files do it; don't copy them.)
- If two controllers need the same resource shape, first ask whether **one endpoint** could serve it. For example, one API returns only IDs and the frontend gets the full objects from the main API. If that's not realistic, define the shape separately in each controller.

---

## 4. From many small files to one big file to TypeScript

```
app/controllers/api/private/**/xxx_controller.swagger.yaml   (hundreds of small files)
        │   make generate-root-swagger      (runs scripts/generate-root-swagger.js)
        ▼
app/controllers/api/private/swagger.yaml    (the combined "root" file: $refs to each small file)
        │   make front/packages/@freee-payroll/api/auto-generated   (runs openapi-generator in Docker)
        ▼
front/packages/@freee-payroll/api/auto-generated/api.ts   (TypeScript functions + types)
```

The combined file just points at each small file:

```yaml
# app/controllers/api/private/swagger.yaml:484-485
  /api/p/employee_display_names/{id}:
    $ref: employee_display_names_controller.swagger.yaml#/paths/~1api~1p~1employee_display_names~1%7Bid%7D
```
(`~1` is how `/` is escaped inside a `$ref` pointer; `%7B`/`%7D` are `{`/`}`.)

The generator names each function **"path + HTTP method" in camelCase**:

| Endpoint | Generated function |
|---|---|
| `GET /api/p/employees` | `apiPEmployeesGet` (example in the docs) |
| `GET /api/p/employee_display_names/{id}` | `apiPEmployeeDisplayNamesIdGet` |
| `POST /api/p/comments` | `apiPCommentsPost` |

It also generates a **type** for the response:

```ts
// front/packages/@freee-payroll/api/auto-generated/api.ts:7741-7754 (comments removed)
export type ApiPEmployeeDisplayNamesIdGet200Response = {
    'id': number;
    'display_name': string;
}
```

```ts
// front/packages/@freee-payroll/api/auto-generated/api.ts:203243-203245
export const apiPEmployeeDisplayNamesIdGet = (id: number, options?: AxiosRequestConfig) => {
    return DefaultApiFp().apiPEmployeeDisplayNamesIdGet(id, options).then((request) => request());
}
```

> 🔑 **Why this matters:** because the TypeScript types come *from the same schema* the server is checked against, the frontend and backend can't silently disagree about field names or types. If someone renames `display_name` in the schema and regenerates, TypeScript errors show every frontend file that needs to change.

---

## 5. Runtime validation with committee

The schema isn't just documentation: the server **checks real traffic** against it. The setup is in `config/application.rb:229-305` (shortened):

```ruby
# config/application.rb:233-244
committee_error_handler = ->(ex, _env) do
  Rails.logger.debug(ex)
  ReachableChecker.execute(message: ex)
end

[
  {
    schema_path: 'app/controllers/api/private/swagger.yaml',
    accept_request_filter: ->(request) { request.path.start_with?('/api/p/') },
    error_handler: committee_error_handler,
    ignore_error: true,
  },
  ...
```
and for the Domain API (`config/application.rb:258-266`) the setting is `strict: true`.

What happens when a request or response **doesn't match** the schema:

| API | Setting | development / test | staging / production |
|---|---|---|---|
| Private (`/api/p/`) | `ignore_error: true` + `ReachableChecker` | **Error** (ReachableChecker raises in dev/test) | Reported to Bugsnag, **request keeps working** |
| Domain | `strict: true` | Error response | Error response (all environments) |

**Why the Private API is lenient in production:** so that an edge-case mismatch that slipped through doesn't break real users. The docs admit this goes against "fail fast", but it was chosen as the lower-risk option. **So in development you must make your controller return exactly what the schema says**, or you'll get errors.

> 💡 Tip: normally the schema is loaded once when Rails starts. Start Rails with `ENABLE_LIVE_RELOAD_PRIVATE_API_SCHEMA=1` and schema edits are picked up without restarting. It's slower, so only use it while editing schemas.

---

## 6. Error handling: how exceptions become JSON

### 6.1 The two main error classes

freee-payroll's own errors live in `lib/hr_errors/`:

| Class | Meaning | Default status | Sent to Bugsnag by default? |
|---|---|---|---|
| `HrErrors::UserError` | The **user** did something wrong or impossible ("that employee doesn't exist", "you can't delete this") | `:bad_request` (400) | **No** (`notify_error: false`) |
| `HrErrors::SystemError` | **We** have a problem (bug, data inconsistency, another service failed) | `:internal_server_error` (500) | **Yes** (`notify_error: true`) |

```ruby
# lib/hr_errors/user_error.rb:12-17 (start of initialize)
def initialize(
  message = DEFAULT_MESSAGE,
  severity: :error,
  notify_error: false,
  status: :bad_request,
  ...
```

Use them like this:

```ruby
raise HrErrors::UserError.new('これ以上、勤務・賃金設定を削除できません')              # 400
raise ::HrErrors::UserError.new('アクセス権限がありません。', status: :forbidden)          # 403
raise ::HrErrors::UserError.new('日次勤怠タグインポート情報が存在しません。', status: :not_found)  # 404
raise ::HrErrors::SystemError.new(original_exception: e)                                # 500, notified
```

The message you pass to `UserError` is **shown to the user**, so write it in polite Japanese.

### 6.2 The flow: `rescue_from` → `ErrorHandleable` → JSON

```
Your code raises an error (anywhere: controller, service, repository, model)
      │
      ▼
ApplicationController: rescue_from StandardError, with: :handle_error     (application_controller.rb:73)
      │
      ▼
ErrorHandleable#handle_error   (app/controllers/concerns/error_handleable.rb:8)
      │  converts ANY error into an HrErrors error:
      │   - HrErrors::*                    → kept as-is
      │   - ActiveRecord::RecordInvalid    → UserError '入力内容に誤りがあります。…' + field errors
      │   - Errors::PayrollError (legacy)  → UserError with the same status
      │   - anything else                  → SystemError (500)
      ▼
super → ApplicationController#handle_error   (application_controller.rb:611)
      │  notifies Bugsnag (Omega::Bugcap.notify) if the error should be notified
      ▼
ErrorHandleable#render_error   (error_handleable.rb:55)
      │  format.json { render json: e.to_json_response, status: e.status }   (line 64)
      ▼
JSON response to the client
```

`to_json_response` gives the **standard error format**:

```ruby
# lib/hr_errors/base.rb:9-11
def to_json_response
  { message: message.presence || DEFAULT_MESSAGE, code: error_code, fields: fields }
end
```

So every error from an `ErrorHandleable` controller looks like:

```json
{ "message": "従業員が見つかりませんでした", "code": null, "fields": [] }
```

(`fields` lists per-field validation errors when there are any; the exact contents vary. This sample is illustrative.)

> 🔑 **Consequence:** in a normal controller you usually **don't write `rescue` at all**. Just `raise HrErrors::UserError` with a good message and status, and the shared machinery does the rest. That's also why the swagger file doesn't define errors.

### 6.3 When controllers *do* rescue

Sometimes a specific low-level error must become a specific user-facing error:

```ruby
# app/controllers/api/private/company_allowances_controller.rb:61-74 (shortened)
def create
  company_allowance =
    ::Payroll::Application::Services::CreateCompanyAllowanceService.execute(
      company_id: current_company.id,
      ...
    )

  render json: build_response(company_allowance), status: :ok
rescue ::Payroll::Infrastructure::Factories::CompanyAllowanceFactory::InvalidCombinationError,
       ::ActiveModel::ValidationError => e
  # ドメインモデルの初期化が持つ検証（calc_type ごとの許容値・範囲）を API 境界で 400 に差し替える。
  raise_invalid_parameter!('手当の設定内容が不正です。…', e.message)
end
```

`rescue` directly inside `def ... end` (without `begin`) covers the whole method. Here two domain errors are turned into a 400.

### 6.4 Legacy error format (just recognize it)

Controllers **without** `ErrorHandleable` fall back to `Api::BaseController#render_error`, which renders `app/views/api/private/shared/error.json.jbuilder` → `{ "messages": [...] }` (plural, an array). New code always uses `ErrorHandleable`.

---

## 7. Other ways to build JSON (recognize them)

| Style | What it looks like | Where you'll see it |
|---|---|---|
| **Plain hash** (most common, modern) | `render json: { round_type: setting.round_type }` | `company_daily_prorate_settings_controller.rb` |
| **Serializer** (ActiveModel::Serializers 0.8) | A class listing `attributes`; can add computed methods | `app/serializers/` |
| **Jbuilder** | A `.json.jbuilder` view file that builds JSON with Ruby | `app/views/api/...` |

**Serializer example:**

```ruby
# app/serializers/comment_serializer.rb:3-24
class CommentSerializer < ActiveModel::Serializer
  attributes :created_at,
             :employee_profile_image_url,
             :user_name,
             :id,
             :company_id,
             :comment,
             :url,
             :user_id,
             :page_name

  def employee_profile_image_url
    options[:authors_by_user_id].dig(object.user_id, :image_url)
  end
  ...
  def created_at
    object.created_at.strftime('%Y-%m-%d %H:%M')
  end
```
- `attributes` = the JSON keys. For each, the serializer calls a method with that name: yours if defined, otherwise the one on `object` (the thing being serialized).
- `options[...]` = extra data passed in when creating it. Used in `comments_controller.rb:15-20`:

```ruby
ActiveModel::ArraySerializer.new(
  comments,
  each_serializer: CommentSerializer,
  authors_by_user_id: authors_by_user_id,
  root: false,
)
```
`ArraySerializer` + `each_serializer:` = "serialize each item of this list with CommentSerializer." `root: false` = don't wrap the output in an extra key.

> ⚠️ `app/CLAUDE.md`: **new references to `EmployeeSerializer` are forbidden** (the RuboCop cop `Hr/UsedEmployeeSerializer`).

**Jbuilder example:**

```ruby
# app/views/api/private/tel_support_requests/initial_values.json.jbuilder (full)
json.res do |json|
  json.accountant_company_external_cid @accountant_company_external_cid
  json.accountant_display_name @accountant_display_name
  json.system_use @system_use
  json.email @email
end
```
The action sets `@instance_variables` and doesn't call `render`. Rails automatically renders the template with the same name (`initial_values`) → `{ "res": { "accountant_company_external_cid": ..., ... } }`.

---

## 8. 🔍 Full trace: "show an employee's display name"

Follow every file. **Open each one in your editor.**

### Step 1: Frontend (TypeScript/React) calls the API

```tsx
// front/javascripts/DEPRECATED_docs/components/Contracts/EditContract.tsx:21
import * as API from '@freee-payroll/api';
// ...:795-798
const loadEmployee = React.useCallback(async (employeeId: number) => {
  const { data } = await API.apiPEmployeeDisplayNamesIdGet(employeeId);
  setEmployee(data);
}, []);
```
You'll learn this syntax in Lessons 22–30. For now: it calls the generated function with an employee ID, **waits** (`await`) for the HTTP response, takes the JSON `data`, and saves it in React state.

### Step 2: HTTP request

```
GET /api/p/employee_display_names/1
```

### Step 3: committee checks the request against the schema (section 5)

`id` must be an integer path parameter. ✓

### Step 4: Route

```ruby
# config/routes.rb:236 (inside namespace :api → scope 'p', module: :private)
resources :employee_display_names, only: [:show]
```
→ `Api::Private::EmployeeDisplayNamesController#show`, `params[:id] = "1"`.

### Step 5: Filters run

`ApplicationController` before_actions: login check (`check_authenticated` in `Api::Private::BaseController`), permissions (`has_allowed_ac?` finds this rule):

```yaml
# config/permission.yaml:1650-1653
  - controller: api/private/employee_display_names
    action: show
    function_keys:
      - employee:contract:write
```

### Step 6: Controller action

```ruby
# app/controllers/api/private/employee_display_names_controller.rb (full)
# typed: true
class Api::Private::EmployeeDisplayNamesController < Api::Private::BaseController
  include ErrorHandleable

  freee_doorkeeper_for :show

  def show
    member =
      ::HrMaster::Infrastructure::Repositories::MemberRepository.find_one_by_employee_id(
        company_id: current_company.id,
        employee_id: show_params[:id].to_i,
      )

    render json: { id: member.employee_id, display_name: member.display_name }
  end

  private

  def show_params
    params.permit(:id)
  end
end
```
- `freee_doorkeeper_for :show` allows this action to also be called with an access token (e.g. from the official mobile app). You can just recognize it.
- The controller **delegates** to a Repository, then shapes the JSON to match the schema **exactly**: `id` and `display_name`.

### Step 7: Repository (in `app/modules/hr_master`, Lessons 17–18)

```ruby
# app/modules/hr_master/infrastructure/repositories/member_repository.rb:44-53
def find_one_by_employee_id(employee_id:, company_id:, include_deleted: false, check_transaction: true)
  assert_transaction_not_open! if check_transaction

  employee = employee_scope(company_id:, include_deleted:).find_by(id: employee_id)
  # 従業員が見つからないのはユーザー起因（存在しない従業員を指定した）なので UserError。
  # employee はあるが member が無い場合はシステム起因（データ不整合）なので、この先で SystemError になる。
  raise HrErrors::UserError.new('従業員が見つかりませんでした') if employee.nil?

  find_member_via_nest_auth(employee:, company_id:)
end
```
- Looks up the `Employee` **within this company** (ActiveRecord, MySQL).
- Not found → `UserError` (the user asked for something that doesn't exist).
- Found → gets the member info from **nest-auth** (another freee microservice) and returns a **domain model** `HrMaster::Domain::Models::Member` (`app/modules/hr_master/domain/models/member.rb`), an immutable object with `employee_id`, `display_name`, etc.

The comment shows the **UserError vs SystemError** judgment: *whose fault is it?*

### Step 8: Response

Success:
```json
{ "id": 1, "display_name": "給与太郎" }
```
committee checks it matches the schema (`id` integer, `display_name` string, both present) ✓. Then React receives it.

Not found: the repository raises `UserError` → `ErrorHandleable` → `{ "message": "従業員が見つかりませんでした", "code": ..., "fields": ... }` with status **400**, the `UserError` default. (The repository doesn't pass `status:`.)

### Step 9: The test

```ruby
# spec/controllers/api/private/employee_display_names_controller_spec.rb:1-22
require 'rails_helper'

describe Api::Private::EmployeeDisplayNamesController, type: :request do
  let(:company) { default_company }
  before { log_in_as_an_admin_user }

  describe '#show' do
    subject { get '/api/p/employee_display_names/1', as: :json }

    before do
      allow(::HrMaster::Infrastructure::Repositories::MemberRepository).to receive(:find_one_by_employee_id).and_return(
        HrMaster::Domain::Models::Member.new(employee_id: 1, display_name: '給与太郎', employee_number: 'EMP001'),
      )
    end

    it do
      is_expected.to eq 200
      expect(JSON.parse(response.body, symbolize_names: true)).to eq({ id: 1, display_name: '給与太郎' })
      ...
```
You'll study specs in Lessons 20–21. Notice: it lives in `spec/controllers/…` but is `type: :request`, exactly as the README requires. It **stubs** (fakes) the repository so the test doesn't need nest-auth. (Its "not found" case stubs a `UserError` with `status: :not_found`, so that test expects 404.)

```
React ─► apiPEmployeeDisplayNamesIdGet(1) ─► GET /api/p/employee_display_names/1
   ─► committee(check request) ─► routes.rb:236 ─► before_actions + permission.yaml
   ─► EmployeeDisplayNamesController#show ─► MemberRepository.find_one_by_employee_id
   ─► Employee (MySQL) + nest-auth ─► Member domain model
   ─► render json: { id, display_name } ─► committee(check response) ─► React setEmployee(data)
```

---

## 9. Checklist: making a new Private API endpoint

1. ✍️ Write `xxx_controller.swagger.yaml` next to where the controller will go (success response only).
2. 🛣 Add the route in `config/routes.rb` inside `scope 'p', module: :private`.
3. 🔐 Add the `controller` + `action` + `function_keys` to `config/permission.yaml`.
4. 🧑‍💻 Write the controller: `< Api::Private::BaseController`, `include ErrorHandleable`, strong params, delegate to a service/repository, `render json:` matching the schema.
5. ⚙️ `make generate-root-swagger` and `make front/packages/@freee-payroll/api/auto-generated`.
6. 🧪 Write a `type: :request` spec in `spec/controllers/api/private/`.
7. 💻 Call it from the frontend with the generated `apiP...` function.

(The repo also has a Claude skill for this: `.claude/skills/gen-private-api/`.)

---

## 10. Practice (15 min)

**Practice 1: read a schema.** Open `app/controllers/api/private/comments_controller.swagger.yaml`. Answer: (a) which query parameter does `GET /api/p/comments` accept? (b) which fields are *required* in `CommentRequestBody`? (c) what status does `DELETE /api/p/comments/{id}` return?

**Practice 2: name the function.** What would the generated function be called for `GET /api/p/company_daily_prorate_setting`? For `PUT` on the same path?

**Practice 3: pick the error.** For each situation, write `UserError` or `SystemError` and a status:
- a) The user tries to delete a comment written by someone else.
- b) Another microservice returned garbage we can't parse.
- c) The requested import ID doesn't exist.

**Practice 4: write a schema.** (illustrative) Write the swagger YAML for `GET /api/p/greetings/{id}` returning `{ "id": integer, "text": string }`, both required.

<details><summary>Practice answers</summary>

1. (a) `url` (query, optional, string). (b) `url` and `comment`. (c) `204`.
2. `apiPCompanyDailyProrateSettingGet` and `apiPCompanyDailyProrateSettingPut` (path + method in camelCase).
3. a) `UserError`, `status: :forbidden` (real code: `comments_controller.rb:52`). b) `SystemError` (500, notified). c) `UserError`, `status: :not_found`.
4.
```yaml
openapi: 3.0.3
info:
  title: freee-payroll
  version: 1.0.0
paths:
  '/api/p/greetings/{id}':
    get:
      parameters:
        - name: id
          in: path
          required: true
          schema:
            type: integer
      responses:
        '200':
          description: OK
          content:
            application/json:
              schema:
                type: object
                required:
                  - id
                  - text
                properties:
                  id:
                    type: integer
                  text:
                    type: string
```
</details>

---

## 11. Summary

- **Private API** = `/api/p/`, called by our own React code. **Schema first**: `xxx_controller.swagger.yaml` next to the controller.
- Only define path, params, body, and **success** response. **No** per-controller errors, **no** `$ref` to other controllers' schemas.
- `make generate-root-swagger` → combined `swagger.yaml`; `make front/packages/@freee-payroll/api/auto-generated` → TypeScript functions named like `apiPEmployeeDisplayNamesIdGet` plus response types.
- **committee** validates real requests/responses: strict in dev/test (via ReachableChecker), lenient in production for `/api/p/`, strict everywhere for the Domain API (TypeSpec).
- Errors: `raise HrErrors::UserError` (user's fault, 400 default, not notified) or `SystemError` (our fault, 500, notified). `ErrorHandleable` turns **any** error into `{ message, code, fields }` JSON.
- JSON is built with a plain hash (modern), serializers, or jbuilder.
- New endpoint checklist: schema → route → permission.yaml → controller → generate → request spec → frontend.

➡️ Now take **Test15.md**.
